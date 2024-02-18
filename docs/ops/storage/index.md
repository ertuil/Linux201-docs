---
icon: material/harddisk
---

# 存储系统

!!! warning "本文初稿已完成，但可能仍需大幅度修改"

服务器需要使用的存储方案与个人计算机的差异较大，例如：

- 服务器提供了大量的盘位，不同服务器的盘位使用的接口、磁盘的尺寸可能不同。
- 服务器一般提供 RAID 卡，可以在硬件层面实现 RAID 功能，大部分 RAID 卡也允许管理员设置直通到操作系统中，由操作系统实现 RAID 功能。
- 根据工作负载的不同，可能需要考虑不同文件系统的差异，并且选择合适的文件系统。
- 由于磁盘数量多，磁盘故障的概率也会增加，因此需要能够及时发现故障，并采取相应的措施。
- …………

本部分会对运维需要了解的基础知识进行介绍。

## 磁盘 {#disks}

### 磁盘类型简介 {#disk-type}

在服务器上最常见的为机械硬盘（HDD）和固态硬盘（SSD）。机械硬盘使用磁盘片和机械臂进行数据读写，固态硬盘使用闪存芯片进行数据读写。
除此以外，还可能有一些特殊的存储设备，例如磁带、Intel Optane 等，这里不做详细介绍。

一般来讲，单块机械硬盘的性能取决于 RPM（转速）。在估算时可以认为顺序读写带宽在 100 MB/s 左右，4K 随机读写带宽在 1 MB/s 左右，IOPS[^1] 在 100 左右。
而单块固态硬盘的顺序读写带宽在 500 MB/s (SATA) 或 1500 MB/s (NVMe) 以上，4K 随机读写带宽在 50 MB/s 左右，IOPS 在 10000 左右。
对于企业级硬盘，这些性能指标可能会更高。

硬盘的使用寿命的一项重要参数是 MTBF（Mean Time Between Failures，平均故障间隔时间）。
由于机械硬盘包含运动的部件，机械硬盘的 MTBF 一般会比固态硬盘低，大致在 10 万小时到 100 万小时。
尽管看起来很长，但是由于实际使用中的振动、温度、读写次数等因素，硬盘的实际寿命可能会远远低于 MTBF。
而对于 SSD 来说，一项更加重要的参数是 TBW（Total Bytes Written，总写入字节数），它表示了闪存芯片可以承受的总写入量，一般在几十到数千 TB；
读取操作对 SSD 的损耗可以忽略不计。

对具体的硬盘型号，建议阅读厂商的文档（例如 datasheet 等），以获取准确的信息。

### 磁盘规格与尺寸 {#disk-size}

关于磁盘规格，在服务器安装时我们主要关心以下几点：

- 磁盘的尺寸（2.5 英寸、3.5 英寸）
    - 部分手册会将 2.5 英寸的磁盘称为 SFF（Small Form Factor），3.5 英寸的磁盘称为 LFF（Large Form Factor）。
- 磁盘的接口与协议（SAS、SATA、NVMe、M.2、U.2）
- 是否与服务器配置兼容：
    - 部分服务器会有硬盘白名单，只有白名单中的硬盘才能识别。
    - 使用的硬件 RAID 方案可能会对硬盘接口有要求，例如 SATA 和 SAS 接口的硬盘不能混用。

!!! warning "仔细阅读并确认厂商文档"

    一个现实中发生过的例子是：将错误的硬盘托架安装至服务器盘位，导致托架卡住无法取出，最后费了近半个小时，甚至用上了螺丝刀作为杠杆，才将其松动，取出硬盘。

磁盘需要带上托架才能安装到服务器中。托架的主要作用是固定住磁盘，并且方便安装和取出。托架的尺寸与磁盘的尺寸有关，3.5 英寸磁盘的托架一般需要安装转接板才能安装 2.5 英寸的磁盘，但也有一些托架预留了 2.5 英寸磁盘的螺丝孔位。同时，托架也一般有 LED 灯，在磁盘故障时会亮黄灯或红灯，也一般能够使用服务器的管理工具控制灯闪烁来定位磁盘。

<!-- TODO: 一张托架的照片 -->

### 磁盘接口与协议 {#disk-interface}

机械硬盘使用 SATA（Serial ATA）或 SAS（Serial Attached SCSI）接口连接，一些固态硬盘也会使用 SATA 接口。SATA 是个人计算机上最常见的硬盘连接方式，而 SAS 的接口带宽更高（虽然机械硬盘实际的传输速度通常无法跑满 SATA 的带宽），支持更多功能，并且向下兼容 SATA，因此在服务器上更常见。

SATA 与 SAS 的详细对比可参考[英文 Wikipedia 中 SAS 的 "Comparison with SATA" 一节](https://en.wikipedia.org/wiki/Serial_Attached_SCSI#Comparison_with_SATA)。

随着固态硬盘的普及，PCIe 接口的固态硬盘也越来越多见。PCIe 接口的固态硬盘通常使用 NVMe 协议，因此也被称为 NVMe SSD。NVMe SSD 常见的接口形态有 U.2、M.2 和 AIC（PCIe Add-in Card，即扩展卡）。

- M.2 接口的尺寸最小，在个人计算机上也更常见，甚至可以放入硬盘盒中作为小巧轻便的移动硬盘使用。

    - 常见的 M.2 接口有 B-key 和 M-key 两种，主流的 NVMe SSD 通常使用 M-key 接口，而 SATA SSD 通常使用 B-key 或 B+M（两个缺口）。一个 M.2 接口是否支持 NVMe 或 SATA 协议取决于主板或控制器，因此具体情况需要参考产品的说明书。
        - 作为参考，2023 年以来市面上已经见不到只支持 SATA 而不支持 NVme 的 M.2 接口了，尽管 M.2 SATA 的 SSD 仍然有卖。
    - 除了接口的形状，M.2 接口的长宽也有一系列选项，例如 2230、2242、2280、22110 等，即宽度为 22 mm，长度分别为 30、42、80、110 mm 等。个人电脑一般采用 2280 的尺寸，而服务器（和一些高端台式机主板）上可能会使用 22110 的尺寸。这些尺寸不影响接口的电气特性，只是为了适应不同的空间和散热需求。
  
- U.2 接口形状与 SAS 类似，但是**不兼容 SAS**（毕竟底层协议都不一样），且 2.5 英寸和 15 mm 以上厚度的外形相比 M.2 也具有更好的散热能力，是服务器上的常见形态。
- AIC 就是一块 PCIe 扩展卡，可以插入 PCIe 插槽中使用。

??? example "图片：M.2 SSD"

    <figure markdown="span">
      ![M.2 2280 SSD](https://upload.wikimedia.org/wikipedia/commons/thumb/2/2b/Intel_512G_M2_Solid_State_Drive.jpg/500px-Intel_512G_M2_Solid_State_Drive.jpg)
      <figcaption>M.2 2280 SSD</figcaption>
    </figure>

??? example "图片：U.2 SSD"

    <figure markdown="span">
      ![U.2 SSD](https://upload.wikimedia.org/wikipedia/commons/thumb/e/e2/OCZ_Z6300_NVMe_flash_SSD%2C_U.2_%28SFF-8639%29_form-factor.jpg/620px-OCZ_Z6300_NVMe_flash_SSD%2C_U.2_%28SFF-8639%29_form-factor.jpg)
      <figcaption>U.2 SSD</figcaption>
    </figure>

??? example "图片：AIC SSD"

    <figure markdown="span">
      ![PCIe AIC SSD](https://m.media-amazon.com/images/I/61jyO1d8v1L.jpg)
      <figcaption>PCIe 插卡式 SSD</figcaption>
    </figure>

### S.M.A.R.T. {#smart}

磁盘的 S.M.A.R.T.（Self-Monitoring, Analysis and Reporting Technology）功能可以记录磁盘的运行状态，并且在磁盘出现故障前提供预警。S.M.A.R.T. 可以记录磁盘的温度、读写错误率、磁盘旋转速度、磁盘的寿命等信息。

Linux 系统上可以安装 `smartmontools` 包来查看磁盘的 S.M.A.R.T. 信息。使用以下命令查看可用的磁盘与其 S.M.A.R.T 状态：

```console
$ sudo smartctl --scan
/dev/nvme0 -d nvme # /dev/nvme0, NVMe device
$ sudo smartctl -a /dev/nvme0
（内容省略）
```

在服务器上，如果使用硬件 RAID，使用 `smartctl` 需要添加额外的参数来从 RAID 控制器获取真实的磁盘信息，例如下面的例子：

```console
$ sudo smartctl --scan
/dev/sda -d scsi # /dev/sda, SCSI device
/dev/sdb -d scsi # /dev/sdb, SCSI device
/dev/sdc -d scsi # /dev/sdc, SCSI device
/dev/sdd -d scsi # /dev/sdd, SCSI device
/dev/bus/4 -d megaraid,8 # /dev/bus/4 [megaraid_disk_08], SCSI device
/dev/bus/4 -d megaraid,9 # /dev/bus/4 [megaraid_disk_09], SCSI device
/dev/bus/4 -d megaraid,10 # /dev/bus/4 [megaraid_disk_10], SCSI device
/dev/bus/4 -d megaraid,11 # /dev/bus/4 [megaraid_disk_11], SCSI device
/dev/bus/4 -d megaraid,12 # /dev/bus/4 [megaraid_disk_12], SCSI device
/dev/bus/4 -d megaraid,13 # /dev/bus/4 [megaraid_disk_13], SCSI device
/dev/bus/4 -d megaraid,14 # /dev/bus/4 [megaraid_disk_14], SCSI device
/dev/bus/4 -d megaraid,15 # /dev/bus/4 [megaraid_disk_15], SCSI device
/dev/bus/0 -d megaraid,8 # /dev/bus/0 [megaraid_disk_08], SCSI device
/dev/bus/0 -d megaraid,9 # /dev/bus/0 [megaraid_disk_09], SCSI device
/dev/bus/0 -d megaraid,10 # /dev/bus/0 [megaraid_disk_10], SCSI device
/dev/bus/0 -d megaraid,11 # /dev/bus/0 [megaraid_disk_11], SCSI device
/dev/bus/0 -d megaraid,12 # /dev/bus/0 [megaraid_disk_12], SCSI device
/dev/bus/0 -d megaraid,13 # /dev/bus/0 [megaraid_disk_13], SCSI device
$ sudo smartctl -a /dev/sdd  # 直接查询只能看到没有意义的控制器信息
smartctl 7.2 2020-12-30 r5155 [x86_64-linux-5.10.0-21-amd64] (local build)
Copyright (C) 2002-20, Bruce Allen, Christian Franke, www.smartmontools.org

=== START OF INFORMATION SECTION ===
Vendor:               AVAGO
Product:              MR9361-8i
Revision:             4.68
Compliance:           SPC-3
User Capacity:        1,919,816,826,880 bytes [1.91 TB]
Logical block size:   512 bytes
Physical block size:  4096 bytes
Logical Unit id:      0x600605b00f17786026223e2d33c1767b
Serial number:        007b76c1332d3e22266078170fb00506
Device type:          disk
Local Time is:        Sun Feb 11 18:40:45 2024 CST
SMART support is:     Unavailable - device lacks SMART capability.

=== START OF READ SMART DATA SECTION ===
Current Drive Temperature:     0 C
Drive Trip Temperature:        0 C

Error Counter logging not supported

Device does not support Self Test logging
$ sudo smartctl -a /dev/bus/4 -d megaraid,8  # 添加参数可以看到真实的磁盘信息
（内容省略）
```

有关 S.M.A.R.T. 指标解释与监控的内容将会在 [LVM 与 RAID](./lvm-raid.md) 中介绍。

### Trim (Discard/Unmap)

SSD 的闪存存储的特点是：不支持任意的随机写，修改数据只能通过清空区块之后重新写入来实现。并且区块能够经受的写入次数是有限的。
SSD 中的固件会进行区块管理，以将写入带来的磨损分散到所有区块中。但是，固件并不清楚文件系统的情况，因此在文件系统中删除某个文件之后，
SSD 固件会仍然认为对应的区块存储了数据，不能释放。Trim 操作由操作系统发出，告诉固件哪些区块可以释放，以提升性能，延长 SSD 使用寿命。一些特殊的存储设备也会支持 trim 操作，例如虚拟机磁盘（`virtio-scsi`）、部分企业级的 SAN 等。

??? note "关注存储的可用空间比例"

    不建议将存储的可用空间全部或接近全部耗尽，这是因为：

    - 机械硬盘：可用空间不足时，文件系统为了存储数据，会不得不产生大量磁盘碎片，而机械硬盘的随机读写性能很差；
    - 固态硬盘：可用空间不足会导致没有足够的空区块改写内容，因此可能不得不大量重复擦写已有的区块，加速磨损。

一般来说，确保 `fstrim.timer` 处于启用状态即可。一些文件系统也支持调整 trim/discard 参数（立即 discard 或周期性 discard，
一般推荐后者）。

## RAID

RAID（Redundant Array of Inexpensive Disks）是一种将多个磁盘组合在一起实现数据冗余和性能提升的技术。不同的磁盘组合方式称为“RAID 级别（RAID Level）”，常见的有 RAID 0、RAID 1、RAID 5、RAID 6、RAID 10 等。

RAID 0

:   也称作条带化（Striped），将数据分块存储在多个磁盘上，可以充分利用所有容量，获得叠加的顺序读写性能（但随机读写性能一般），但没有冗余，任何一块磁盘损坏都会导致整个阵列的数据丢失，适合需要高性能读写但不需要数据安全性的场景。

RAID 1

:   也称作镜像（Mirrored），将数据完全复制到多个磁盘上，提供了绝对冗余，整个阵列中只需要有一块盘存活，数据就不会丢失。代价是整个阵列的容量等单块磁盘的容量，空间利用率低下，适合需要高可靠性<s>而且不缺钱</s>的场景。同时由于每块盘上的数据完全一致，RAID 1 的读取性能可以叠加（甚至包括随机读取），但写入性能不会提升。

RAID 5

:   将数据和**一份**校验信息分块存储在多个磁盘上，可以允许阵列中任何一块磁盘损坏，兼顾冗余性和容量利用率。重建期间的性能会严重下降，并且一旦在重建完成前又坏了一块盘，那么你就寄了。

RAID 6

:   将数据和**两份**校验信息分块存储在多个磁盘上，比 RAID 5 多了一份校验信息，可以容纳两块磁盘损坏，适合大容量或者磁盘较多的阵列。尽管允许两块盘损坏，但我们仍然建议在第一块盘损坏后立即更换并重建，不要等到更危险的时候。

RAID 10, 50, 60

:   将不同级别的 RAID 组合在一起，兼顾性能和冗余，各取所长，对于 10 块盘以上的阵列是更加常见的选择。例如 RAID 10 = RAID 1 + RAID 0，通常将每两块盘组成 RAID 1，再将这些 RAID 1 的组合拼成一个大 RAID 0。

!!! danger "磁盘阵列不是备份"

    RAID 不是备份，它可以实现在某块磁盘故障时保证系统继续运行，但是不能在数据误删除、自然灾害、人为破坏等情况下保护数据。

    [![磁盘阵列不是备份](images/raid-is-not-backup.png)](https://mirrors.tuna.tsinghua.edu.cn/tuna/tunight/2023-03-26-disk-array/slides.pdf)

    *图片来自[金枪鱼之夜：实验物理垃圾佬之乐——PB 级磁盘阵列演进](https://tuna.moe/event/2023/disk-array/)*

### RAID 等级比较

| 等级    | 容量         | 冗余                           | 读写性能                                         | 适用场景                     |
| ------- | ------------ | ------------------------------ | ------------------------------------------------ | ---------------------------- |
| RAID 0  | 全部叠加     | 无，挂一块盘就寄了             | 顺序读写性能高，随机读写性能一般（略好于单块盘） | 临时数据、缓存               |
| RAID 1  | 单块盘       | 最高，只要有一块盘存活就行     | 叠加的读性能，但是只有单块盘的写性能             | 重要数据                     |
| RAID 5  | N-1 块盘     | 可以坏一块盘                   | 顺序读写性能高，随机读写性能差；重建期间**很差** | 兼顾容量和安全性             |
| RAID 6  | N-2 块盘     | 可以坏两块盘                   | 顺序读写性能高，随机读写性能差；重建期间**更差** | 比 RAID 5 更稳一点           |
| RAID 10 | 每组 RAID 1 容量叠加    | 每组 RAID 1 内只需要存活一块盘 | 顺序和随机性能都不错，并且重建期间还凑合         | 兼顾性能和安全性             |
| RAID 60 | （自行计算） | 每组 RAID 6 内可以坏两块盘     | 顺序读写性能不错，并且重建期间<s>更凑合了</s>    | 盘很多，并且兼顾容量和安全性 |

RAID 4 和 RAID 50 在这里不作讨论，因为它们没人用。

!!! question "思考题"

    1. 为什么 RAID 5 和 RAID 6 的重建期间性能会很差？
    2. 为什么将大量的磁盘组合成单个 RAID 6 不是一个好主意？
    3. 为什么对大容量磁盘使用 RAID 5 是一个坏主意？
    4. 为什么说 RAID 5/6 有很高的写惩罚（Write Penalty）？

### RAID 实现方式

RAID 可以在硬件层面实现，也可以在操作系统层面通过软件实现。

硬件：

- LSI MegaRAID 系列是最常见的硬件 RAID 卡
- HPE Smart Array 系列
- 一些专用的存储服务器，如 HPE MSA、Dell EMC PowerVault 等存储网络（SAN）设备
- Intel RST 和 Intel VMD，是个人计算机上常见的硬件 RAID 方案<s>（但是非常难用）</s>

Windows：

- 较早的“动态磁盘”功能支持 RAID 0、1、5 等级别，并且是在分区层面实现的，因此同一个硬盘组上可以同时存在多个采用不同 RAID 级别的卷（文件系统）
- 较新的 Windows 开始支持 Storage Spaces，可以实现更多 RAID 级别，以及“镜像加速”、“自动热迁移”等功能，但是比动态磁盘更加难用，重建也更复杂

Linux：

- mdadm 是 Linux 上最常见的软件 RAID 实现方式之一
- Linux 常用的逻辑卷管理工具 LVM 也支持 RAID 功能
- 一部分文件系统，如 ZFS 和 Btrfs，也支持 RAID 功能

服务器一般都配备了硬件 RAID 卡，如果需要使用软件 RAID，需要设置 RAID 卡或对应磁盘到 HBA 模式（也叫 JBOD/Non-RAID/直通 模式）。
一部分低端 RAID 卡不提供相关功能的支持，尽管可以通过为每块磁盘创建单独的 RAID 0 阵列实现类似的功能，但是并不推荐。

## 文件系统

在 Linux 上，ext4 是最常见的文件系统。但是对于某些需求，ext4 可能无法满足，例如：

- 需要支持存储大量文件，数量甚至可能超过 ext4 的限制
- 需要支持快照、透明压缩、数据校验等高级功能

在[「分区与文件系统」部分](./filesystem.md)会对文件系统的选择进行介绍。

[^1]: IOPS 为每秒的 I/O 操作数，可以简单认为是命令操作延迟的倒数。