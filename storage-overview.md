

- [存储概念](#存储概念)
  - [物理存储 HDD与SDD](#物理存储-hdd与sdd)
  - [存储类型](#存储类型)
  - [IO基础](#io基础)
    - [IOPS](#iops)
    - [吞吐量(Throughput)](#吞吐量throughput)
    - [延迟(Latency)](#延迟latency)
    - [实际问题](#实际问题)
  - [存储不同部署方式](#存储不同部署方式)
    - [本地存储](#本地存储)
    - [网络存储](#网络存储)
      - [网络块存储](#网络块存储)
    - [网络文件存储](#网络文件存储)
    - [分布式存储](#分布式存储)
      - [Object对象](#object对象)
      - [File文件](#file文件)
      - [Block块](#block块)
  - [其他](#其他)
    - [RAID](#raid)
    - [纠删码](#纠删码)
    - [副本](#副本)
    - [快照](#快照)
    - [备份](#备份)
    - [归档](#归档)


## 存储概念

### 物理存储 HDD与SDD
||HDD|SDD|
|:---:|:---:|:---:|
|存储的基本原理|靠磁性物质存储数据，通过最小磁性单元磁畴磁化后的正反方向转化成电信号代表1/0，无磁状态为空|靠存储电子存储数据，隧道效应，通过加电压使得电子穿过绝缘体，困在浮栅中，无电子为1，有电子为0|
|数据组织方式|磁畴组成磁单元，多个磁单元组成扇区，扇区Sector是一个圆弧，最小的寻址单位，最早是512B, 现在是4KB。写数据的时候，优先在同一个柱面，也就是多个磁盘同一位置的扇区进行写数据，考虑到磁头大小和位置，柱面物理上可能不是绝对的垂直，不同的碟片上可能有偏移|由多个晶体管组成一个page，最小一般为16KB，多个page组成一个block，为最小擦除单位，然后组成die, plane|
|寻址方式|依靠磁头在磁盘的不同同心圆即磁道切换来寻址。|依靠互相垂直位线、字线加不同伏值的电压来寻址，最小单位是页Page，一般是16KB。由于和主流文件系统的最小单位4KB不同，ssd会复用page，可能将不同的逻辑块写入同一个page中，因此ssd维护一个逻辑块到物理地址的寻址映射表。(LBA - Logic Block Addressing, FBA - Fixed Block  Architecture)|
|物理接口|SATA, SAS|M.2(SATA, NVMe/PCIe)|
|读写语义|SCSI|SCSI/NVMe|

### 存储类型
||块存储|文件存储|对象存储|
|:---:|:----:|:----:|:----:|
|数据访问|通过逻辑块地址(LBA)按块读写，上层自己组织数据|通过路径/文件名访问，由文件系统组织目录、权限、元数据|通过ID/Key访问，对象连通元数据一起存储|
|本地接口/协议|SCSI/NVMe设备语义|本地文件系统如xfs,ntfs,ext4|-|
|远程协议|SAN(SCSI/NVMe over FC/Ethernet)|NFS, CIFS/SMB|S3, Swift|
|应用场景|数据库、虚拟机磁盘等IO密集型|共享目录等通用|图片视频，备份、归档等非结构化数据，读多写少|
|数据结构及寻址方式|块号/偏移|文件系统的目录树+inode/元数据|Bucket+Key，底层结合哈希、索引、元数据服务实现|
|特点|低延迟，随机读写性强，但不直接管理文件|用户友好|易扩展，适合分布式、高并发，但延时高|

### IO基础

#### IOPS
每秒访问数，一般用4K, 8K小数据块模拟随机存储，HDD的IOPS为100 ~ 500, SDD的IOPS可以达到1K ~ 500K, 云存储采用多层内存缓存+sdd落地, IOPS可以达到更快。
目前数据中心存储采用以太网较多，传统企业使用的SAN网络较少，主要原因是价格高，FC兼容性差，无论是以太网还是FC，网络延迟对IOPS的影响都较小。
随机读写次数较多的，例如数据库，需要首先衡量存储设备的IOPS

#### 吞吐量(Throughput)
是指每秒数据的传输量，常用MB/s, GB/s单位。大文件，例如4K视频的读写，需要首先衡量存储设备的Throughput。

#### 延迟(Latency)
存储设备的相应速度，从收到请求到返回数据的时间，Latency对IOPS的影响较大，对小文件的随机读写，影响比较大。
HDD, 常用ms毫秒为单位, SSD，常用μs微秒为单位

#### 实际问题
Q: 假设你要为一个同时运行在线交易系统和视频点播服务的服务器选择存储，在线交易需要快速响应每笔订单，视频点播需要流畅播放高清视频，你会怎么平衡 IOPS、吞吐量和延迟？

A: 在线交易系统需求是低延迟高IOPS，在线视频网站的要求是低IOPS，高吞吐量。
在设计这个解决方案的时候，可以考虑将SSD主要用于交易系统，HDD主要用于视频点播服务，使用LVM+dm-cache实现。
将两个设备分别创建两个PV，加入VG中，创建交易系统卷的时候，指定物理卷为SSD,使用其中一部分块空间；创建视频点播服务卷的时候，将SSD物理卷中剩余的块空间作为cache pool，和HDD一起创建卷。
这样既满足了交易系统的高IOPS，也能满足视频点播服务的大容量存储，增加了SSD缓存cache，也提高了性能。

``` shell
pvcreate /dev/sdb  # SSD
pvcreate /dev/sdc  # HDD
vgcreate storage_vg /dev/sdb /dev/sdc

lvcreate -L 50G -n transaction_lv storage_vg /dev/sdb

lvcreate -L 50G -n ssd_cache_lv storage_vg /dev/sdb

lvcreate -l 100%FREE -n video_hdd_lv storage_vg /dev/sdc

lvconvert --type cache-pool --cachemode writethrough storage_vg/ssd_cache_lv

lvconvert --cache-pool storage_vg/ssd_cache_lv storage_vg/video_hdd_lv
```

### 存储不同部署方式
本地存储、网络存储、分布式存储

#### 本地存储
通过本机总线直连本地存储设备，延迟低、性能高，共享能力弱。
```
Application
        -> VFS
        -> File System(ext4, xfs)
        -> Block Layer
        -> Device Driver
        -> Disks

注: 有些应用可绕过文件系统直接使用块设备
Application(e.g. DB)
        -> Block Layer
        -> Device Driver
        -> Disks
```

#### 网络存储
通过网络(FC, Ethernet)连接存储设备，主机设备与存储设备分离，便于集中管理与共享。读写延迟、性能受到网络架构影响，传统网络存储通常以纵向扩展(scale up)为主。FC SAN需要专用网络，性能高，延迟低但价格昂贵。
##### 网络块存储
通过FC/iSCSI/NVMe-oF等方式，把远程的存储呈现为主机上的块设备。
主机通过NIC/HBA连接存储，需要一个主动的发现过程：触发后，主机侧的initiator发现远端的target/LUN, OS创建一个本地块存储设备(例如/dev/sdx)，应用对该设备的读写，通过initiator与远程的存储进行通信，完成真正的数据读写。这个连接存储设备的网卡，在操作系统中一直存在，也需要进行配置管理。
```
Application
        -> VFS
        -> File System
        -> Block Layer
        -> SAN/iSCSI/NVME-oF initiator
        -> NIC / HBA
    ~~~ Network ~~~
        -> Storage Array Controller
        -> Disks
```


#### 网络文件存储
通过以太网来连接的存储服务器，通过SMB/NFS协议提供文件服务，通过挂载将文件服务导出的目录挂载在本地目录进行读写。
```
Application
        -> VFS
        -> SMB/NFS Client
    ~~~~~~ Network ~~~~~~
        -> NAS Server
        -> Export/Share
        -> File System
        -> Disks

```

#### 分布式存储
通过软件方式，将多台机器上的存储资源组织成存储池，提供对象、块、文件等接口，具有横向扩展以及容错能力，系统复杂度较高。

##### Object对象
```
Application
        -> Network(S3/HTTP API)
        -> Object Storage Cluster
        +--> Storage Device on Node 1
        +--> Storage Device on Node 2
        +--> Storage Device on Node 3
```
##### File文件
```
Application
        -> VFS
        -> Distributed FS Client / SMB / NFS
    ~~~~~~ Network ~~~~~~
        -> Distributed File Storage Cluster
        +--> Storage Device on Node 1
        +--> Storage Device on Node 2
        +--> Storage Device on Node 3
```
##### Block块
```
Application
        -> VFS
        -> File System
        -> Block Layer
        -> Distributed Block Client / iSCSI / NVMe-oF Initiator
    ~~~~~~ Network ~~~~~~
        -> Distributed Block Storage Cluster / Network Protocol (iSCSI/NVMe-oF)
        +--> Storage Device on Node 1
        +--> Storage Device on Node 2
        +--> Storage Device on Node 3
```

### 其他
#### RAID
RAID全称Redundat Array of Independent Disks, 独立磁盘冗余阵列。是通过多块磁盘不同的组织形式，提高磁盘的读写效率以及可靠性的技术。
|名称|原理|特点|最少需要磁盘数|使用场景|
|:---:|:----:|:----:|:----:|:----:|
|RAID0|条带化，将不同数据分散写到不同磁盘上|提升读写效率，没有容错性，一块磁盘数据丢失就造成数据丢失|2|临时数据|
|RAID1|镜像，将数据同时写到不同磁盘上，提升数据可靠性|提升数据可靠性，有一块磁盘损坏数据依然可用；容量利用率低，50%|2|关键小容量数据，如系统盘|
|RAID2|将数据**按位**分散到不同磁盘，使用汉明码纠错机制，额外专用磁盘存储纠错码|实现复杂|3|实际场景几乎不使用|
|RAID3|将数据**按字节**分散到不同磁盘，额外专用磁盘存储纠错码|一块磁盘损坏数据依然可用，专用数据校验盘可能成为瓶颈|3|实际场景很少使用|
|RAID4|将数据**按块**分散到不同磁盘，额外专用磁盘存储纠错码|一块磁盘损坏数据依然可用，专用数据校验盘可能成为瓶颈|3|实际场景很少使用|
|RAID5|将数据**按块**分散到不同磁盘，校验信息也分散到不同磁盘上|一块磁盘损坏数据依然可用，读写性能、数据可靠性较均衡|3|通用文件存储|
|RAID6|将数据**按块**分散到不同磁盘，配备**两份独立校验信息**，校验信息也分散到不同磁盘上|读写性能较RAID5稍差，容错能力更强，两块磁盘损坏的情况下数据依然可用|4|通用文件存储|
#### 纠删码

#### 副本
#### 快照
#### 备份
#### 归档