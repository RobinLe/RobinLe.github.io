---
layout: post
title: Docker存储驱动AUFS
category: docker
tags: storage
---


Docker存储驱动之AUFS，关于Docker的Image（镜像）与Container（容器）的存储以及存储驱动之AUFS

<br/>

#### Docker存储驱动简介
Docker内置多种存储驱动，每种存储驱动都是基于Linux文件系统（Linux FS）或者卷管理（Volume Manager）技术。一般来说，Docker存储驱动的名称与文件系统（存储技术）同名，见下表对应关系

|    文件系统/存储    |      存储驱动名称      |
| :-----------: | :--------------: |
|   OverlayFS   | overlay/overlay2 |
|     AUFS      |       aufs       |
|     Btrfs     |      btrfs       |
| Device Mapper |   devicemapper   |
|      VFS      |       vfs        |
|      ZFS      |       zfs        |

<br/>

#### AUFS简介
AUFS，起初名为`AnotherUnionFileSystem`，是一种UnionFS，V2版本后更名为`advanced multi-layered unification fileystem`即高级多层统一文件系统。

AUFS是一位名为岡岛纯二郎的日本人于2006年基于UnionFS开发的，目的也是为了提高其可靠性和性能，也在AUFS上实现了一些新的概念比如写分支平衡等（writeable branch balancing）。

> 纯二郎先生曾多次提交aufs到linux主干，但一直被拒绝，可纯二郎依旧日以继夜的修改代码，并不断提交，可一直没有被merge，而纯二郎最终也放弃提交代码到linux主干，从而aufs也一直没有进入linux内核。关于为何linux不要aufs，主要是aufs代码太糟糕，“dense，unreadable，and uncommented”，乱，可读性差，没注释...。

*Tips：虽然AUFS没有进入Liunx内核，但是AUFS默认是写入Ubuntu内核的，Docker早期仅支持Ubuntu便是这个原因*  

<br/>

#### AUFS核心概念
将多个目录合并成一个虚拟文件系统，成员目录称为虚拟文件系统的一个分支（**branch**）。

例如，把`/tmp`，`/var`，`/opt`三个目录联合挂着到`/aufs`目录下，则`/aufs`目录可见`/tmp`，`/var`，`/opt`目录下的所有文件。而每个成员目录，则称为虚拟文件系统的一个**branch**。

![img](https://raw.githubusercontent.com/RobinLe/RobinLe.github.io/master/_posts/images/2017031301.png)

每个branch可以指定`readwrite/whiteout-able/readonly`权限，**只读（ro），读写（rw），写隐藏（wo）**。一般情况下，aufs只有最上层的branch具有读写权限，其余branch均为只读权限。只读branch只能逻辑上修改。

例如，把`/tmp`目录和`/var`目录联合挂载到`/aufs`目录下，`/var`目录挂载为顶层branch具有**rw**权限，而`/tmp`目录挂载为下层branch，具有**ro**权限。如果把`/tmp`目录`/var`目录`/opt`目录联合挂载到`/aufs`目录下，只有最顶层`/opt` branch具有**rw**权限，下层branch均为**ro**权限

![img](https://raw.githubusercontent.com/RobinLe/RobinLe.github.io/master/_posts/images/2017031306.png)

*Tips：mount -t aufs -o br=/tmp:/var:/opt none /aufs*

*该指令联合挂载`/tmp`，`/var`，`/opt`至挂载点`/aufs`，按照从左到右的顺序`/tmp`位于顶层branch，而`/opt`位于最底层branch，所有的branch按照顺序共同构成`/aufs`栈*

AUFS每层branch可以动态的增加删除，每增加一层，下层默认置为**ro**，最上一层为**rw**。删除branch是在aufs挂载点移除，并未删除挂载目录。

<br/>

#### Docker中的AUFS
AUFS是Docker使用的第一个存储驱动，长时间以来，一直很稳定。虽然目前Docker支持多种存储驱动，而在Ubuntu中默认使用aufs存储驱动，可以使用docker info指令进行查看。对于Docker而言，AUFS有很多优秀的特点，比如**快速启动容器**，**高效存储利用率**，**高效内存利用率**等。

**Docker镜像和AUFS branch对应关系**

Docker镜像（image）是由一个或多个AUFS branch组成，并且所有的branch均为只读权限。简单来说，AUFS所有**ro** branch按照一定顺序堆积构成docker image镜像。

在运行容器的时候，创建一个AUFS branch位于image层之上，具有**rw**权限，并把这些branch联合挂载到一个挂载点下。这就是Docker能够一个镜像运行多个容器的原理所在。

例如，image有3个目录位于`/var/lib/docker/aufs/diff/`文件夹内，当基于该image创建容器时，创建一个容器运行目录，同样位于`/var/lib/docker/aufs/diff/`目录下，并使用aufs联合挂载image目录和container目录到`/var/lib/docker/aufs/mnt/`目录下。

每一个目录在aufs内都是一层branch，只有顶层容器branch可读写，下层image branch均只读。创建多个容器时，只需创建多个容器运行目录，使用aufs把容器运行目录挂载在image目录之上，即实现一个image运行多个container。

![img](https://raw.githubusercontent.com/RobinLe/RobinLe.github.io/master/_posts/images/2017031307.png)

如上图中基于同一个image运行两个容器，低层image一致，每启动一个新容器，便会新建一个目录作为aufs branch并与image branch进行联合挂载。

<br/>

**容器文件读写与删除**

当容器需要修改一个文件，而该文件位于低层branch时，顶层branch会直接复制低层branch的文件至顶层再进行修改，而低层的文件不变，这种方式即是**CoW**技术（写复制），AUFS默认支持**Cow**技术。

当容器删除一个低层branch文件时，只是在顶层branch对该文件进行重命名并隐藏，实际并未删除文件，只是不可见，这种方式即AUFS的**whiteout**（写隐藏）。

下图所示，容器层所见**file1**文件为镜像层文件，当需要修改**file1**时，会从镜像层把文件复制到容器层，然后进行修改，从而保证镜像层数据的完整性和复用性。

![img](https://raw.githubusercontent.com/RobinLe/RobinLe.github.io/master/_posts/images/2017031308.png)

下图所示，当需要删除**file1**时，由于**file1**是镜像层文件，容器层会创建一个`.wh`前置的隐藏文件，从而实现对**file1**的隐藏，实际并未删除**file1**，从而保证镜像层数据的完整性和复用性。

![img](https://raw.githubusercontent.com/RobinLe/RobinLe.github.io/master/_posts/images/2017031309.png)

<br/>

**Docker本地存储目录**

Docker镜像存在在`/var/lib/docker/aufs/diff/`目录下。`/var/lib/docker/aufs/layers/`目录存储image的元信息，主要是image的每一层layer组织顺序（AUFS branch堆积顺序）。

Docker容器挂载点位于`/var/lib/docker/aufs/mnt/<container-id>`目录下，该目录即为AUFS联合挂载点。

*Tips：早期镜像的每一层数据存储在`/var/lib/docker/aufs/diff/`目录下，镜像ID与目录名同名，从docker v1.10版本之后镜像ID不再与`/var/lib/docker/aufs/diff/`目录下的目录同名，而是采用一个安全的hash值作为镜像存储文件夹名称。*

<br/>

#### 总结
采用 aufs作为 docker 的容器存储驱动，使用AUFS技术，能够提供如下好处

**节省存储空间**：多个容器可以共享 基础镜像（base image）存储

**快速部署：**如果要部署多个容器，基础镜像（base image）可以避免多次拷贝

**内存更省：**因为多个容器共享base image，以及OS的disk缓存机制，多个容器中的进程命中缓存内容的几率大大增加

**允许在不更改基础镜像的同时修改其目录中的文件：**所有写操作都发生在最上层的writeable层中，这样可以大大增加base image能共享的文件内容。

<br/>

#### 参考文献
1. [https://docs.docker.com/engine/userguide/storagedriver/aufs-driver/#related-information](https://docs.docker.com/engine/userguide/storagedriver/aufs-driver/#related-information)  
2. http://aufs.sourceforge.net/  
3. [https://en.wikipedia.org/wiki/Aufs ](https://en.wikipedia.org/wiki/Aufs)  
4. 于烨，李斌，刘思尧Docker 技术的移植性分析研究。软件，2015,（7）


