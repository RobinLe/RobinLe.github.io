---
layout: post
title: Kubernetes容器化部署
---

> 2016-12-29    robin Lee    miaomiao3312763@qq.com



本文介绍如何部署Kubernetes，由于基础设施环境多样，本文部署情况仅仅针对**裸机或虚拟机**环境，暂不支持公有云如GCE，AWS，DigitOcean环境，也不支持直接与虚拟化系统如ESXi进行部署。



### 部署方式

Kubernetes部署方式分为两种，二进制方式或容器化方式

1. 二进制：直接部署所有组件到主机
2. 容器化：所有组件在容器内方式运行




---

### 容器部署Kubernetes

**一、依赖**

* docker-engine 1.12
* ubuntu16.04



**二、下载安装包**

下载[k8s_install_images](\\192.168.3.36\lan_share_robin\k8s_install_images.tar.gz)到**master**和**slave**机器，解压安装包，赋予`setup.sh`可执行权限

![](file:///./images/1.png)

```
root@k8s-master:/home/ubuntu# tar xzf k8s_install_images.tar.gz 
root@k8s-master:/home/ubuntu# ls
k8s_install_images  k8s_install_images.tar.gz
root@k8s-master:/home/ubuntu# cd k8s_install_images
root@k8s-master:/home/ubuntu/k8s_install_images# ls
16.04-xenial  flannel.yaml  images  setup.sh
root@k8s-master:/home/ubuntu/k8s_install_images# chmod +x setup.sh 
root@k8s-master:/home/ubuntu/k8s_install_images# 
```

![file:///2.png](file:///./images/2.png)



执行`./setup.sh`，`setup.sh`会自动安装**kubectl**，**kubernetes-cni**，**kubelet**，**kubeadm**组件

![](file:///./images/3.png)

并且会自动导入**docker**镜像

![](file:///./images/4.png)

使用`docker images`查看**master**机器k8s组件镜像信息

![](file:///./images/5.png)



使用**kubeadm**初始化**master**节点，需要定义pod私有地址范围（不要和主机私有地址范围冲突）

```
root@k8s-master:/home/ubuntu/# kubeadm init --pod-network-cidr=10.244.0.0/16
```

![](file:///./images/6.png)

**重要：记录`kubeadm join --token=`信息，因为slave需要该条指令加入集群**

创建**flannel**网络容器

```
root@k8s-master:/home/ubuntu/k8s_install_images# kubectl apply -f flannel.yaml
```

![](file:///./images/7.png)

使用**kubectl**查看kubernetes组件运行情况

```
kubectl get pods --all-namespaces
```

![](file:///./images/8.png)



向**kubernetes**集群添加**slave**节点

下载k8s_install_images安装包，解压，执行`setup.sh`同上，安装完**kubeadm**，**kubectl**，**kubelet**等组件后执行**master**节点初始化后输出的`kubeadm join`指令

```
kubeadm join --token=5ac10b.060d697903aae36d 192.168.3.48
```

![](file:///./images/9.png)



在**master**节点查看集群情况

![](file:///./images/10.png)