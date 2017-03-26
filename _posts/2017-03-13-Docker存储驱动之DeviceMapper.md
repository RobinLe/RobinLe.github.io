---
layout: post
title: Docker存储驱动DeviceMapper
category: docker
tags: storage
---

上篇介绍了Docker存储驱动aufs以及AUFS文件系统，本篇聊聊`devicemapper`存储驱动以及**Device Mapper**技术，**Device Mapper**较之**AUFS**工作相对底层。

<br/>

##### Device Mapper是一种Linux块设备映射技术框架
**Device Mapper**不同于**AUFS**、**ext4**、**NFS**等，因为它并不是一个文件系统（File System），而是Linux内核映射块设备的一种技术框架。提供的一种从逻辑设备（虚拟设备）到物理设备的映射框架机制，在该机制下，用户可以很方便的根据自己的需要制定实现存储资源的管理策略。

当前比较流行的 Linux 下的逻辑卷管理器如 LVM2（Linux Volume Manager 2 version)、EVMS(Enterprise Volume Management System)、dmraid(Device Mapper Raid Tool)等都是基于该机制实现的。

值得一提的是**Device Mapper**工作在块级别（block），并不工作在文件级别（file）。**Device Mapper**自Linux 2.6.9后编入Linux内核，所有基于Linux内核2.6.9以后的发行版都内置**Device Mapper**。

*起初Docker仅在Ubuntu和Debian上使用AUFS运行，在Docker火起来后，另一个Linux大户Redhat家族也想使用Docker，但是由于AUFS并未编入Linux内核，也并未写入Redhat家族内核，所以包括RedHat和CentOS均无法使用Docker。RedHat开发人员曾想把AUFS写入内核以支持Docker，但最终还是选用的Device mapper作为后台存储技术，而Docker方面重新设计了Docker引擎以支持多种驱动，并开发devicemapper存储驱动支持Device Mapper作为后台存储。*

<br/>

##### Device Mapper主要分为用户空间部分和内核空间部分
用户空间相关部分主要负责配置具体的策略和控制逻辑，比如逻辑设备和哪些物理设备建立映射，怎么建立这些映射关系等，包含`device mapper`库和`dmsetup`工具。对用户空间创建删除`device mapper`设备的操作进行封装。

内核中主要提供完成这些用户空间策略所需要的机制，负责具体过滤和重定向 IO 请求。通过不同的驱动插件，转发IO请求至目的设备上。附上**Device Mapper**架构图。
![img](https://raw.githubusercontent.com/RobinLe/RobinLe.github.io/master/_posts/images/2017031310.gif)

<br/>

#### Device Mapper技术分析
**Device Mapper**作为Linux块设备映射技术框架，向外部提供逻辑设备。包含三个重要概念，映射设备（mapped device），映射表（map table），目标设备（target device）。

映射设备即对外提供的逻辑设备，映射设备向下寻找必须找到支撑的目标设备，映射表存储映射设备和目标设备的映射关系。目标设备可以是映射设备或者物理设备，如果目标设备是一块映射设备，则属于嵌套，理论上可以无限迭代下去。

简而言之，**Device Mapper**对外提供一个虚拟设备供使用，而这块虚拟设备可以通过映射表找到相应的地址，该地址可以指向一块物理设备，也可以指向一个虚拟设备。
![img](https://raw.githubusercontent.com/RobinLe/RobinLe.github.io/master/_posts/images/2017031311.png)

映射表，是由用户空间创建，传递到内核空间。映射驱动在内核空间是插件，已实现的映射驱动（target driver） 插件包括软 raid、软加密、逻辑卷条带、多路径（mulitpath）、镜像（mirror）、快照（snapshot）等。

Device Mapper中的IO流处理，从虚拟设备（逻辑设备）根据映射表并指定特定的映射驱动转发到目标设备上。

<br/>

#### Docker中的Device Mapper核心技术
**Docker**的`devicemapper`驱动有三个核心概念，`copy on-write（写复制）`，`thin-provisioning（瘦供给）`。`snapshot（快照）`，首先简单介绍一下这三种技术。

CoW（copy on write）写复制，一些文件系统提供的写时复制策略，**AUFS**文中已介绍，不赘述。`devicemapper`支持在块级别（block）写复制。

Snapshot（快照技术），**SNIA定义**：关于指定数据集合的一个完全可用拷贝，该拷贝包括相应数据在某个时间点（拷贝开始的时间点）的映像。快照可以是其所表示的数据的一个副本，也可以是数据的一个复制品。而从具体的技术细节来讲，快照是指向保存在存储设备中的数据的引用标记或指针。

接下来的档案修改或任何新增、删除动作，均不会覆写原本数据所在的磁盘区块，而是将修改部分写入其它可用的磁盘区块中。device mapper映射驱动支持snapshot技术。

Thin-provisioning（瘦供给），直译为精简配置，《容器与容器云》（浙大SEL实验室著）同译为精简配置，但个人认为瘦供给更易理解，瘦供给是动态分配，需要多少分配多少，区别于传统分配固定空间从而造成的资源浪费，附下图
![img](https://raw.githubusercontent.com/RobinLe/RobinLe.github.io/master/_posts/images/2017031312.jpg)

<br/>

#### Docker中devicemapper驱动存储
1. 首先创建一个瘦资源池（thin pool，使用瘦供给技术，存储动态分配），瘦资源池使用2块设备，一个用于存储数据，一个用于存储元数据（虚拟设备和目标设备的映射表）
2. 基于该瘦资源池创建基础设备（Device Mapper中的逻辑设备）
3. 所有的镜像数据均基于该基础设备创建快照，容器层则基于镜像层创建快照。

![img](https://raw.githubusercontent.com/RobinLe/RobinLe.github.io/master/_posts/images/2017031313.png)
*Tips：Docker在创建瘦资源池（thin pool）的时候，默认使用loop-lvm模式，该模式使用稀疏文件构建资源池。另一种模式是direct-lvm模式，该模式使用块设备创建资源池。docker模式采用loop-lvm是因为不需要额外配置，而在生产环境中建议使用direct-lvm模式，该模式需要手动配置数据设备从而构建瘦资源池。*

<br/>

##### 使用devicemapper的读写操作
`devicemapper`使用`snapshot`快照技术，每层镜像都是基于上一个镜像的快照构建。所有读取数据操作，如果数据在低层快照中，则会重定向到低层快照数据地址并取出数据。写数据会向瘦资源池（thin pool）请求空间写数据，由于`devicemapper`工作在block层，空间分配以块为单位，docker中默认块大小为64KB。

镜像的复用即通过`snapshot`快照技术实现，有别于aufs等存储驱动。所有的新增和删除都是不覆盖原本数据块，而是使用其他可用的新的数据块进行操作。

<br/>

#### 总结
**存储空间节省：**由于工作在block级，每一个block 64KB，尤其是在更新大文件的时候，不需要复制整个文件，只需要更新文件有修改所在的block，有效节省存储空间。

**存取速度：**`devicemapper`逻辑设备映射目标设备，目标设备构建于SSD等高速存储上，能获得更好的速度。

**兼容性好：**由于**Device Mapper**从Linux 2.6以后就写入内核，所以尽管发行版不同，但都能使用`devicemapper`驱动。

**内存资源浪费：**启动N个相同的容器会加载N份相同文件到内存，势必造成主机资源浪费，不建议在PaaS或者资源密集场合使用。

*Tips：关于资源浪费，因为devicemapper工作在block，很难实现pagecache*

<br/>

#### 参考资料
1. [https://docs.docker.com/engine/userguide/storagedriver/device-mapper-driver/#device-mapper-and-docker-performance](https://docs.docker.com/engine/userguide/storagedriver/device-mapper-driver/#device-mapper-and-docker-performance)
2. [https://www.ibm.com/developerworks/cn/linux/l-devmapper/](https://www.ibm.com/developerworks/cn/linux/l-devmapper/)
3. [http://coolshell.cn/articles/17200.html](http://coolshell.cn/articles/17200.html)
4. [https://www.kernel.org/doc/Documentation/device-mapper/](https://www.kernel.org/doc/Documentation/device-mapper/)
5. [http://hustcat.github.io/overlayfs-intro/](http://hustcat.github.io/overlayfs-intro/)
6. [https://en.wikipedia.org/wiki/Device_mapper](https://en.wikipedia.org/wiki/Device_mapper)

