# crushmap 範例

###### tags: `Ceph`, `crush`

## 簡介說明
crushmap 建好-> crush rule 建好 -> 指定 rule to crushmap -> 建 ecpool 並指定 crush map -> 建 複本 的 pool 並指定crush map

## ceph 名詞解釋:

### ceph osd pool default size = 2
* 在 replicated pool 的話是幾個 replica
* 在 erasure coded pool 的話是 m 有幾個
* Resilience: You can set how many OSD are allowed to fail without losing data. For replicated pools, it is the desired number of copies/replicas of an object. A typical configuration stores an object and one additional copy (i.e., size = 2), but you can determine the number of copies/replicas. For erasure coded pools, it is the number of coding chunks (i.e. m=2 in the erasure code profile)

### bucket weights 
* weight 值是用來設定 osd 硬碟間相對的大小, 官網(https://docs.ceph.com/en/latest/rados/operations/crush-map-edits/) 建議用 1.0 代表 1TB : recommend using 1.00 as the relative weight for a 1TB storage device

## 前置作業
### 每台裝置上都需安裝：
* OS
Ubuntu Server 16.04
Kernel 4.12.14
* Package
Ceph 12.2.8 (Luminous)
OpenSSH
### Setup VM using Vagrant
### Add repository and install Ceph
```
$ sudo wget -q -O- 'https://download.ceph.com/keys/release.asc' | sudo apt-key add -
$ sudo apt-add-repository 'deb https://download.ceph.com/debian-nautilus/ xenial main'
$ apt-get update
$ apt-get install ceph ceph-common
```
## Ceph Monitor 初始安裝
### 產生 FSID
* FSID
請使用 $ uuidgen 產生
Output 會是一個標準UUID格式字串 (8-4-4-4-12)
e.g => 723fb218-0afa-4c85-a751-71a4ad9921a9
* 注意
FSID每個Ceph cluster只能有一個唯一的UUID
無法中途改變此FSID

### 產生 ceph.conf
* 一個標準 INI 格式的Config File
* Config File包含有：FSID、Monitor member & IP、CephX and other Configurations
* ceph.conf 範例如下
* modify fsid
* modify mon host, public network, cluster network -> vm’s ip

```
vim /etc/ceph/ceph.conf
```

* ceph.conf:
```
# Version 12.2.5
[global]
fsid = cb976184-a9a6-4b73-9167-3521327db945
mon initial members = mon01
mon host = 10.73.7.81
public network = 10.73.7.0/24
cluster network = 10.73.7.0/24

#Clusters require authentication by default.
auth cluster required = cephx
auth service required = cephx
auth client required = cephx

mon max pg per osd = 1000
mon allow pool delete = true
osd crush update on start = false
```

### 製作 Monitor Keyring
* 使用 Ceph command 製作一把keyring
  * Keyring Name => mon.
  * Keyring的 Capacity為
    * mon allow *
* 執行 $ ceph-authtool 命令

```
$ ceph-authtool --create-keyring /tmp/ceph.mon.keyring --gen-key -n mon. --cap mon 'allow *'
```

### 製作 Admin Keyring

* 使用 Ceph command 製作一把keyring
  * Keyring Name => client.admin
  * Keyring的 Capacity為
    * mon allow *
    * osd allow *
    * mds allow *
    * mgr allow *
* 執行 $ ceph-authtool 命令
```
$ ceph-authtool --create-keyring /etc/ceph/ceph.client.admin.keyring --gen-key -n client.admin --set-uid=0 --cap mon 'allow *' --cap osd 'allow *' --cap mds 'allow *' --cap mgr 'allow *'
```
* 把 Monitor keyring 匯入 Admin Keyring
執行 $ ceph-authtool 命令:
```
$ ceph-authtool /tmp/ceph.mon.keyring --import-keyring /etc/ceph/ceph.client.admin.keyring
```

### 導出bootstrap-osd keyring
* 使用 Ceph command 製作一把keyring
* 執行 $ ceph-authtool 命令
```
# ceph-authtool --create-keyring /var/lib/ceph/bootstrap-osd/ceph.keyring --gen-key -n client.bootstrap-osd --cap mon 'profile bootstrap-osd' --cap mgr 'allow r'
```

* 把 Monitor keyring 匯入 bootstrap-osd keyring
* 執行 $ ceph-authtool 命令
```
# ceph-authtool /tmp/ceph.mon.keyring --import-keyring /var/lib/ceph/bootstrap-osd/ceph.keyring
```

### 製作 monmap
* 把Ceph cluster 所以 monitor 資訊添加進 monmap 的檔案中
* 第一次製作 monmap時，需要有 FSID
* 每個 monitor 要包含 member name, IP and Port
* 執行 $ monmaptool 命令 (modify ip)
```
$ monmaptool --create --add mon01 192.168.35.151:6789 --fsid `ceph-conf --lookup fsid` /tmp/monmap
```

* 使用 $monmaptool 命令查看製作出來的monmap是否正確
```
$ monmaptool --print /tmp/monmap
```

### 變更檔案擁有者
* 變更 ceph.conf 和 ceph.client.admin.keyring的擁有者
```
$ chown -R ceph:ceph /etc/ceph
```

* 變更 Ceph MON01的擁有者
```
$ chown -R ceph:ceph /var/lib/ceph/mon
$ chown -R ceph:ceph /tmp/monmap
$ chown -R ceph:ceph /tmp/ceph.mon.keyring
```

### 初始化Monitor Daemon
* 建立Ceph monitor 所需要目錄與檔案
* 以Ceph User 的角色執行 $ ceph-mon : 
```
$ ceph-mon --mkfs --id mon01 --monmap /tmp/monmap --keyring /tmp/ceph.mon.keyring
```
* 檢查是否有執行成功，查看Ceph monitor下是否有 keyring 和 store.db兩個檔案
```
$ ls /var/lib/ceph/mon/ceph-mon01/
$ chown ceph:ceph -R /var/lib/ceph/mon
```

### 設定系統Services
* 需要 Enable / Start Ceph monitor
* 執行 $ systemctl :
```
$ systemctl enable ceph-mon@mon01.service
$ systemctl start ceph-mon@mon01.service
```

* 檢查是否Daemon 是否有正常請動，執行 $ ps :
```
$ ps aux | grep ceph
```

* 沒有啟動Ceph monitor Daemon 請查閱 Troubleshooting (TBST) 文件

## Ceph MGR 初始化
### 創建 Keyring
* 建立 MGR01的 Keyring
```
$ ceph auth get-or-create mgr.mgr01 mon 'allow profile mgr' osd 'allow *' mds 'allow *'
```

* 導出 mgr01的 keyring
```
$ ceph auth get-or-create mgr.mgr01 -o keyring
```

### 創建 mgr01工作目錄
* 創建 mgr01工作目錄
```
$ mkdir -p /var/lib/ceph/mgr/ceph-mgr01
```

* 搬移 keyring至工作目錄
```
$ mv keyring /var/lib/ceph/mgr/ceph-mgr01
```

* 變更 ACL
```
$ chown -R ceph:ceph /var/lib/ceph/mgr/ceph-mgr01
```

### 設定系統Services
* 需要 Enable / Start Ceph MGR
* 執行 $ systemctl :
```
$ systemctl enable ceph-mgr@mgr01.service
$ systemctl start ceph-mgr@mgr01.service
```
* 檢查是否Daemon 是否有正常請動，執行 $ ps :
```
$ ps aux | grep ceph
```
* 沒有啟動Ceph MGR Daemon 請查閱 Troubleshooting (TBST) 文件

## Ceph OSD 初始化
### 導出bootstrap-osd keyring
* 檢索 admin key
```
ceph-create-keys --id mon01
```

* 檢視 bootstrap-osd 的 keyring 
```
$ cat /var/lib/ceph/bootstrap-osd/ceph.keyring
```
* 導出的Keyring的檔案名稱必定為 ceph.keyring
* 變更檔案擁有者
```
$ chown -R ceph:ceph /var/lib/ceph/bootstrap-osd
```

### 取得 OSD & WAL/DB Disk
執行 $ frisk -l 查找 OSD Disk 和 WAL/DB Disk
```
$ fdisk -l
```

假設 OSD Disk為 /dev/sdb 


### 切割 WAL/DB Size
* WAL/DB是從 Bluestore Backend時新加入的功能， Bluestore配後由RockDB 是Key/Value Database是用來儲存 Metadate.
* RockDB 需要有 WAL和 DB兩個元件才能運作.
* WAL 全名為 Write ahead Logging主要功能是所有修改在提交之前都要先寫入 WAL文件中。
  * 通常大小介於 512MB~2GB
  * 空間越大在 Flush到硬碟中，所需要 RAM就會越大
* DB為 RocksDB儲存位置
  * 通常大小要  >10GB
* WAL/DB 盡量放在 High Speed Device上
  * e.g. SSD or NVME

### 執行 Disk Create
* 執行 $ ceph-volume 切格 Partitions 和配置WAL/DB Partition
ceph-volume lvm create --bluestore --data <block> [--block.wal <wal> --block.db <db>]
```
$ ceph-volume lvm create --bluestore --data /dev/sdb
```

* 使用 $ fdisk -l 檢查 切割狀態
  * /dev/sdc 的 TYPE 要是 Ceph OSD

### 變更檔案擁有者
* 變更 Ceph OSD的擁有者
```
$ chown -R ceph:ceph /var/lib/ceph/osd/ceph-0
```

* 檢視CRUSH map
```
$ ceph osd tree
```


### 設定系統Services
* 需要 Enable / Start Ceph OSD
* 執行 $ systemctl :
```
$ systemctl enable ceph-osd@0.service
$ systemctl start ceph-osd@0.service
```

* 檢查是否Daemon 是否有正常請動，執行 $ ps :
```
$ ps aux | grep ceph
```
* 沒有啟動Ceph OSD Daemon 請查閱 Troubleshooting (TBST) 文件


## 設定Ceph CRUSH Map
* 搬移 osd 到指定host、root上
```
$ ceph osd crush add osd.0 1.0 root=cacheDefault host=SSD_HOST_01
$ ceph osd crush add osd.1 1.0 root=cacheDefault host=SSD_HOST_02
$ ceph osd crush add osd.2 1.0 root=cacheDefault host=SSD_HOST_03 
$ ceph osd crush add osd.3 1.0 root=default host=HDD_HOST_01 
$ ceph osd crush add osd.4 1.0 root=default host=HDD_HOST_02 
$ ceph osd crush add osd.5 1.0 root=default host=HDD_HOST_03
```
* 設定 osd class
```
$ ceph osd crush set-device-class ssd osd.0 osd.1 osd.2 
$ ceph osd crush set-device-class hdd osd.3 osd.4 osd.5
```
* 設定 crush rule
```
$ ceph osd crush rule ls
$ ceph osd crush rule create-replicated ssd cacheDefault host ssd
$ ceph osd crush rule create-replicated hdd default host hdd
```

### 建立 Erasure Code crush rule
* ceph osd crush rule create-erasure {rulename} {profilename}

* 下列範例先建立 k=3, m=2, failure domain= rack 的 profile 接著用來建立 erasure code crush rule
```
$ ceph osd erasure-code-profile set myprofile \
   k=3 \
   m=2 \
   crush-failure-domain=rack
$ ceph osd crush rule create-erasure ecrule myprofile
```


### 建立 replicated crush rule
* ceph osd crush rule create-replicated &lt;rule-name> &lt;root> &lt;failure-domain> &lt;class>

* 下列範例建立了名稱是 "replicated_rule", root 是 default, failure domain 是 host 的 replicated crush rule:
```
ceph osd crush rule create-replicated replicated_rule default host ssd
```

### 建 ecpool 並指定 crush rule
* To create an erasure coded pool, execute:
ceph osd pool create {pool-name} [{pg-num} [{pgp-num}]]  erasure [erasure-code-profile] [crush-rule-name] [expected_num_objects]
* 下列指令裡 12 是 placement group 的個數. 且使用預設的 erasure code profile 和 crush rule:
```
$ ceph osd pool create ecpool 12 12 erasure
```
* 下列指令則會建立 k=3 m=2 且 failure domain 是 rack 的 erasure coded pool: 
```
$ ceph osd pool create ecpool erasure myprofile ecrule
```

### 建 複本的 pool 並指定 crush rule
* To create a replicated pool, execute:
ceph osd pool create pool_name pg_num pgp_num replicated crush_ruleset_name expected_num_objects

* 下列例子會建立名稱是 "repool" 的兩個副本的 replicated pool, 其中 crush rule 是設定使用前面範例建立的 "replicated_rule"
```
$ ceph osd pool create repool 100 100 replicated_rule ssd
$ ceph osd pool set repool size 3
```

