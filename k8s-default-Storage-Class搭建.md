---
title: k8s default Storage Class搭建
categories: k8s tools
date: 2019-07-23 22:45:48
tags: k8s tools
top: 100
---
# k8s default Storage Class搭建
在k8s中，StorageClass为动态存储，存储大小设置不确定，对存储并发要求高和读写速度要求高等方面有很大优势；pv为静态存储，存储大小要确定。而default Storage Class的作用为pvc文件没有标识任何和storageclass相关联的信息，但通过annotations属性关联起来。
<!--more--> 
![storageClass](http://ww1.sinaimg.cn/large/006bbiLEgy1g5a6p0p8fjj30ng0e43za.jpg "storageClass")
## 1. 创建storageClass
要使用 StorageClass，我们就得安装对应的自动配置程序，比如我们这里存储后端使用的是 nfs，那么我们就需要使用到一个 nfs-client 的自动配置程序，我们也叫它 Provisioner，这个程序使用我们已经配置好的 nfs 服务器，来自动创建持久卷，也就是自动帮我们创建 PV。
nfs服务器参考博客：
https://lilinlinlin.github.io/2019/07/23/Linux-nfs-%E6%90%AD%E5%BB%BA/
### 1.1 nfs-client-provisioner
前提：
* Kubernetes 1.9+
* Existing NFS Share
* helm

#### 1.1.1 安装nfs-client-provisioner指令：
`helm install --name nfs-client --set nfs.server=192.168.0.9 --set nfs.path=/nfsdatas stable/nfs-client-provisioner`
如果安装报错，显示没有该helm的stable，在机器上添加helm 源
`helm repo add stable http://mirror.azure.cn/kubernetes/charts/`
#### 1.1.2 卸载指令：
`helm delete nfs-client`
#### 1.1.3 验证storageClass是否存在
在相应安装nfs-client-provisioner机器上执行：`kubectl get sc`即可，如下所示：
```
[root@i-cj1r8a8m ~]# kubectl get sc
NAME              PROVISIONER                                   AGE
nfs-client    cluster.local/my-release-nfs-client-provisioner   1h
```
## 2. DefaultStorageClass
在定义StorageClass时，可以在Annotation中添加一个键值对：storageclass.kubernetes.io/is-default-class: true，那么此StorageClass就变成默认的StorageClass了。
### 2.1 第一种方法
在这个PVC对象中添加一个声明StorageClass对象的标识，这里我们可以利用一个annotations属性来标识，如下
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvctest
  annotations:
    volume.beta.kubernetes.io/storage-class: "nfs-client"
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 10Mi
```
### 2.2 第二种方法
用 kubectl patch 命令来更新：
`kubectl patch storageclass nfs-client -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'`
最后结果中包含default为：
```
[root@i-cj1r8a8m ~]# kubectl get sc
NAME                 PROVISIONER                                   AGE
nfs-client (default)cluster.local/my-release-nfs-client-provisioner 2h
```

