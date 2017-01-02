---
layout: post
title: Kubernetes二进制部署
category: k8s
tags: deploy
---



本文介绍如何部署Kubernetes，由于基础设施环境多样，本文部署情况仅仅针对**裸机或虚拟机**环境，暂不支持公有云如GCE，AWS，DigitOcean环境，也不支持直接与虚拟化系统如ESXi进行部署。



### 部署方式

Kubernetes部署方式分为两种，二进制方式或容器化方式

1. 二进制：直接部署所有组件到主机
2. 容器化：所有组件在容器内方式运行




------

### 二进制部署Kubernetes

**一、依赖**

- docker-engine 1.12
- ubuntu14.04

**二、下载安装包**

下载[k8s_install_binaries](\\192.168.3.36\lan_share_robin\k8s_install_binaries.tar.gz)到**master**机器，解压安装包

![](file:///./images/11.png)

进入`cluster`目录，编辑`setup.sh`脚本文件

![](https://robinle.github.io/images/12.png)

![](file:///./images/13.png)

需要根据具体部署的集群信息更新`setup.sh`脚本文件

* **nodes**：所有节点信息`username@ip`，以空格分开，第一个默认为**master**节点
* **roles**：`a代表master`，`i代表node`，`ai代表master node`，定义**nodes**参数所有节点的角色信息
* **NUM_NODES**：节点数量，集群共有多少节点
* **SERVICE_CLUSTER_IP_RANGE**：定义`service IP`范围，给定一个**`私有IP`**范围，确保不要和`本地私有IP `**冲突**
* **FLANNEL_NET**：定义`flannel overlay`网络地址，确保不要与**SERVICE_CLUSTER_IP_RANGE**冲突。



赋予`setup.sh`执行权限，并执行`setup.sh`脚本

**重要：如果机器没有`ssh`秘钥，则生成秘钥文件，可以不输入直接回车自动生成**

![](file:///./images/14.png)

安装过程中，需要输入多次**master**节点和**node**节点的密码信息，执行结束会得到如下提示

![](file:///./images/15.png)



**master**节点查看节点信息，验证集群状态

```
root@k8s-master:/home/ubuntu/k8s_install_binaries/cluster# kubectl get nodes
NAME           STATUS    AGE
192.168.3.44   Ready     5m
192.168.3.48   Ready     4m
```

![](file:///./images/16.png)

```
root@k8s-master:/home/ubuntu/k8s_install_binaries/cluster# ps -ef
```

![](file:///./images/17.png)
