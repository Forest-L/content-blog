---
title: NetApp 存储在 KubeSphere 上的实践
date: 2019-09-23 01:08:54
tags: k8s tools
top: 100
---

**NetApp**是向目前的数据密集型企业提供统一存储解决方案的居世界最前列的公司，其 Data ONTAP是全球首屈一指的存储操作系统。NetApp 的存储解决方案涵盖了专业化的硬件、软件和服务，为开放网络环境提供了无缝的存储管理。
**Ontap**数据管理软件支持高速闪存、低成本旋转介质和基于云的对象存储等存储配置，为通过块或文件访问协议读写数据的应用程序提供统一存储。
**Trident**是一个由NetApp维护的完全支持的开源项目。以帮助您满足容器化应用程序的复杂**持久性**需求。
**KubeSphere** 是一款开源项目，在目前主流容器调度平台 Kubernetes 之上构建的企业级分布式多租户**容器管理平台**，提供简单易用的操作界面以及向导式操作方式，在降低用户使用容器调度平台学习成本的同时，极大降低开发、测试、运维的日常工作的复杂度。
<!--more-->
### 1、整体方案
在VMware Workstation环境下安装ONTAP;ONTAP系统上创建SVM(Storage Virtual Machine)且对接nfs协议；在已有k8s环境下部署Trident,Trident将使用ONTAP系统上提供的信息（svm、managementLIF和dataLIF）作为后端来提供卷；在已创建的k8s和StorageClass卷下部署kubesphere。
### 2、版本信息
Ontap: 9.5
Trident: v19.07
k8s: 1.15
kubesphere: 2.0.2
### 3、步骤
主要描述ontap搭建及配置、Trident搭建和配置和kubesphere搭建及配置等方面。
#### 3.1 ontap搭建及配置
在VMware Workstation上Simulate_ONTAP_9.5P6_Installation_and_Setup_Guide运行，ontap启动之后，按下面操作配置，其中以cluster base license、feature licenses for the non-ESX build配置证书、e0c、ip address：192.168.*.20、netmask:255.255.255.0、集群名:cluster1、密码等信息。
https://IP address,以上设置的iP地址，用户名和密码：
![netapp.png](http://ww1.sinaimg.cn/large/006bbiLEgy1g6t9q3s4kkj30yf0l8qsj.jpg)

#### 3.2 Trident搭建及配置
* 下载安装包trident-installer-19.07.0.tar.gz，解压进入trident-installer目录，执行trident安装指令:
`./tridentctl install -n trident`
* 结合ontap的提供的参数创建第一个后端vi backend.json。
```
{
    "version": 1,
    "storageDriverName": "ontap-nas",
    "backendName": "customBackendName",
    "managementLIF": "10.0.0.1",
    "dataLIF": "10.0.0.2",
    "svm": "trident_svm",
    "username": "cluster-admin",
    "password": "password"
}
```
生成后端卷`./tridentctl -n trident create backend -f backend.json`
* 创建StorageClass,vi storage-class-ontapnas.yaml
```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ontapnasudp
provisioner: netapp.io/trident
mountOptions: ["rw", "nfsvers=3", "proto=udp"]
parameters:
  backendType: "ontap-nas"
```
创建StorageClass指令`kubectl create -f storage-class-ontapnas.yaml`

#### 3.3 kubesphere的安装及配置
* 在 Kubernetes 集群中创建名为 kubesphere-system 和 kubesphere-monitoring-system 的 namespace。

```
$ cat <<EOF | kubectl create -f -
---
apiVersion: v1
kind: Namespace
metadata:
    name: kubesphere-system
---
apiVersion: v1
kind: Namespace
metadata:
    name: kubesphere-monitoring-system
EOF
```

* 创建 Kubernetes 集群 CA 证书的 Secret。

```
$ kubectl -n kubesphere-system create secret generic kubesphere-ca  \
--from-file=ca.crt=/etc/kubernetes/pki/ca.crt  \
--from-file=ca.key=/etc/kubernetes/pki/ca.key
```

* 若 etcd 已经配置过证书，则参考如下创建（以下命令适用于 Kubeadm 创建的 Kubernetes 集群环境）：

```
$ kubectl -n kubesphere-monitoring-system create secret generic kube-etcd-client-certs  \
--from-file=etcd-client-ca.crt=/etc/kubernetes/pki/etcd/ca.crt  \
--from-file=etcd-client.crt=/etc/kubernetes/pki/etcd/healthcheck-client.crt  \
--from-file=etcd-client.key=/etc/kubernetes/pki/etcd/healthcheck-client.key
```

* 修改kubesphere.yaml中存储的设置参数和对应的参数即可
`kubectl apply -f kubesphere.yaml`

* 访问 KubeSphere UI 界面。

![kubesphere](https://pek3b.qingstor.com/kubesphere-docs/png/20190912002602.png)

#### 参考文档
http://docs.netapp.com/ontap-9/index.jsp?lang=zh_CN
https://netapp-trident.readthedocs.io/en/stable-v19.07/introduction.html
https://github.com/kubesphere/ks-installer/blob/master/README_zh.md









