---
layout: post
title: 容器部署kubernetes
category: k8s
tags: deploy
---


本文介绍在使用容器方式部署kubernetes，在centos7主机上，基于docker v1.12.5部署kubernetes v1.5.0。仅部署kubernetes必要组件，各个组件均封装成容器部署运行。



### 测试环境

|     HOST      |   OS    |   Dependence   |  Role  |
| :-----------: | :-----: | :------------: | :----: |
| 192.168.3.100 | CentOS7 | Docker v1.12.5 | master |
| 192.168.3.101 | CentOS7 | Docker v1.12.5 |  node  |



### Kubernetes必要组件

#### master

|        Component        | Version |  Format   |
| :---------------------: | :-----: | :-------: |
|          etcd           | v0.3.15 | container |
|     kube-apiserver      | v1.5.0  | container |
| kube-controller-manager | v1.5.0  | container |
|      kube-schduler      | v1.5.0  | container |

#### node

| Component  | Version |  Format   |
| :--------: | :-----: | :-------: |
|  kubelet   | v1.5.0  |  binary   |
| kube-proxy | v1.5.0  | container |



#### etcd容器获取及启动

获取官方官方镜像

```
[root@k8s-master ~]# docker pull quay.io/coreos/etcd:v3.0.15
```

从**Docerfile**构建

```shell
# description:     etcd v0.3.15-linux-amd64
FROM alpine:3.4

ADD etcd /usr/local/bin/
ADD etcdctl /usr/local/bin/
RUN mkdir -p /var/etcd/
RUN mkdir -p /var/lib/etcd/

EXPOSE 2379 2380

# Define default command.
CMD ["/usr/local/bin/etcd"]
```

启动**etcd**容器，监听2379，2380，4001端口

```
[root@k8s-master ~]# docker run -p 4001:4001 -p 2380:2380 -p 2379:2379 --name k8s-etcd k8s/etcd:v3.0.15 etcd\
 -name etcd0 \
 -advertise-client-urls http://192.168.3.100:2379,http://192.168.3.100:4001 \
 -listen-client-urls http://0.0.0.0:2379,http://0.0.0.0:4001 \
 -initial-advertise-peer-urls http://192.168.3.100:2380 \
 -listen-peer-urls http://0.0.0.0:2380 \
 -initial-cluster-token etcd-cluster-1 \
 -initial-cluster etcd0=http://192.168.3.100:2380 \
 -initial-cluster-state new
```

![](https://raw.githubusercontent.com/RobinLe/RobinLe.github.io/master/_posts/images/28.png)



#### k8s组件容器获取及启动

下载官方编译后的二进制文件包(https://storage.googleapis.com/kubernetes-release/release/v1.5.0/kubernetes-server-linux-amd64.tar.gz)，解压后在`kubernetes/server/bin`目录下拥有各个组件二进制文件和镜像包

```
[root@k8s-master bin]# pwd
/root/kubernetes/server/bin
[root@k8s-master bin]# ll -h
total 1.3G
-rwxr-x---. 1 root root 145M Dec 13 07:45 hyperkube
-rwxr-x---. 1 root root  91M Dec 13 07:45 kubeadm
-rwxr-x---. 1 root root 118M Dec 13 07:45 kube-apiserver
-rw-r-----. 1 root root   33 Dec 13 07:45 kube-apiserver.docker_tag
-rw-r-----. 1 root root 119M Dec 13 07:45 kube-apiserver.tar
-rwxr-x---. 1 root root  97M Dec 13 07:45 kube-controller-manager
-rw-r-----. 1 root root   33 Dec 13 07:45 kube-controller-manager.docker_tag
-rw-r-----. 1 root root  98M Dec 13 07:45 kube-controller-manager.tar
-rwxr-x---. 1 root root  49M Dec 13 07:46 kubectl
-rwxr-x---. 1 root root 6.6M Dec 13 07:45 kube-discovery
-rwxr-x---. 1 root root  44M Dec 13 07:45 kube-dns
-rwxr-x---. 1 root root  46M Dec 13 07:46 kubefed
-rwxr-x---. 1 root root 103M Dec 13 07:45 kubelet
-rwxr-x---. 1 root root  44M Dec 13 07:45 kube-proxy
-rw-r-----. 1 root root   33 Dec 13 07:46 kube-proxy.docker_tag
-rw-r-----. 1 root root 174M Dec 13 07:46 kube-proxy.tar
-rwxr-x---. 1 root root  51M Dec 13 07:45 kube-scheduler
-rw-r-----. 1 root root   33 Dec 13 07:46 kube-scheduler.docker_tag
-rw-r-----. 1 root root  52M Dec 13 07:46 kube-scheduler.tar
```

![](https://raw.githubusercontent.com/RobinLe/RobinLe.github.io/master/_posts/images/27.png)

直接`docker load`镜像

```
[root@k8s-master bin]# docker load -i kube-apiserver.tar
[root@k8s-master bin]# docker load -i kube-controller-manager.tar
[root@k8s-master bin]# docker load -i kube-scheduler.tar
```

或者使用**Dockerfile**构建镜像

```
# description:      kube-apiserver, kubernetes-1.5.0
FROM busybox:1.25
ADD kube-apiserver /usr/local/bin/

---
# description:      kube-controller-manager, kubernetes-1.5.0
FROM busybox:1.25
ADD kube-controller-manager /usr/local/bin/

---
# description:      kube-scheduler, kubernetes-1.5.0
FROM busybox:1.25
ADD kube-scheduler /usr/local/bin/
```

![](https://raw.githubusercontent.com/RobinLe/RobinLe.github.io/master/_posts/images/30.png)



#### kube-apiserver启动

```
[root@k8s-master ~]# docker run -p 8080:8080 --name k8s-apiserver k8s/kube-apiserver:v1.5.0 kube-apiserver \
 --insecure-bind-address=0.0.0.0 \
 --insecure-port=8080 \
 --etcd-servers=http://172.17.0.2:2379 \
 --logtostderr=true \
 --service-cluster-ip-range=10.10.3.0/24 \
 --advertise-address=192.168.3.100
```

![](https://raw.githubusercontent.com/RobinLe/RobinLe.github.io/master/_posts/images/29.png)

#### kube-controller-manager启动

```
[root@k8s-master ~]# docker run --name k8s-controller k8s/kube-controller-manager:v1.5.0 kube-controller-manager \
 --logtostderr=true \
 --master=172.17.0.3:8080
```

![](https://raw.githubusercontent.com/RobinLe/RobinLe.github.io/master/_posts/images/31.png)

#### kube-schduler启动

```
[root@k8s-master ~]# docker run --name k8s-scheduler k8s/kube-scheduler:v1.5.0 kube-scheduler \
 --logtostderr=true \
 --master=172.17.0.3:8080
```

![](https://raw.githubusercontent.com/RobinLe/RobinLe.github.io/master/_posts/images/32.png)
