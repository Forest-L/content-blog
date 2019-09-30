---
title: netapp联合k8s在kubesphere应用
date: 2019-09-09 20:48:00
tags: k8s tools
top: 100
---
# netapp联合k8s在kubesphere应用
#### 配置说明
* k8s: 1.13+
* ontap: 9.5
* trident: v19.07
* kubesphere: 2.0.2

#### 前期准备
需要准备材料在以下链接上，链接：https://pan.baidu.com/s/1q3KugGrz-XWJzqhgD7Ze9g 
提取码：rhyw 
<!--more-->
包括ontap的安装说明，Workstation上ontap9.5模拟器的ova，ontap使用文档，对接各种存储的协议证书，

#### 1. ontap的安装
本次测试的环境是安装在VMware Workstation，具体参考链接上这个文档Simulate_ONTAP_9.5P6_Installation_and_Setup_Guide，大致流程为：
* window机器的资源配置
* 开启VT
* 为模拟ONTAP配置VMware Workstation
* 在VMware Workstation上配置网络适配器，选择bridge网络
* 开启模拟ONTAP

#### 2. 模拟ontap的配置
以上ontap启动之后，按下面操作配置，其中以cluster base license、feature licenses for the non-ESX build配置证书、e0c、ip address：192.168.*.20、netmask:255.255.255.0、集群名:cluster1、密码等信息。
* 2.1 Press Ctrl-C for Boot 菜单消息显示时，按 Ctrl-C

```
md1.uzip: 39168 x 16384 blocks
md2.uzip: 5760 x 16384 blocks
*******************************
* *
* Press Ctrl-C for Boot Menu. *
* *
*******************************
^C
Boot Menu will be available.
```

* 2.2 选择4配置

```
Please choose one of the following:
(1) Normal Boot.
(2) Boot without /etc/rc.
(3) Change password.
(4) Clean configuration and initialize all disks.
(5) Maintenance mode boot.
(6) Update flash from backup config.
(7) Install new software first.
(8) Reboot node.
Selection (1-8)? 4
```

* 2.3 确认reset 和 确定

```
Zero disks, reset config and install a new file system?: y
This will erase all the data on the disks, are you sure?: y
```

* 2.4 创建集群,填写参数

```
Enter the cluster management interface port [e0d]: e0c
Enter the cluster management interface IP address: 192.168.x.20
Enter the cluster management interface netmask: 255.255.255.0
Enter the cluster management interface default gateway: <Enter>
A cluster management interface on port e0c with IP address 192.168.x.
20 has been created.
You can use this address to connect to and manager the cluster.
Do you want to create a new cluster or join an existing cluster?
{create}:
create
Enter the cluster name: cluster1
login: admin
Password: <password you defined>
Enter the cluster base license key:SMKQROWJNQYQSDAAAAAAAAAAAAAA
```

#### 3. 登录ontap界面
参考链接中的m_SL10537_gui_nas_basic_concepts_v2.1.0文档，这里需要配置的信息为，对接各个存储的协议证书、子网的设置、聚合的创建、svm创建和导出策略的配置。

* https://IP address,以上设置的iP地址，用户名和密码：
![netapp.png](http://ww1.sinaimg.cn/large/006bbiLEgy1g6t9q3s4kkj30yf0l8qsj.jpg)

* 对接各个存储的协议证书
登录平台之后，配置--》许可证--》添加对应的证书，显示为绿色的勾就添加正确。
* 子网的设置
登录平台，网络--》子网--》创建
* 聚合的创建
登录平台，存储--》聚合和磁盘--》聚合--》创建
* svm的创建，此处需要选择对应的存储协议、 存储中的聚合和权限、管理(LIF)和数据（LIF）等信息。
登录平台，存储--》SVM--》创建
* 导出策略的设置，svm创建之后，点击svm设置--》导出策略，在规则索引下添加客户端规范0.0.0.0/0，协议和权限。

#### 4. trident安装部署
介质在链接中，包括所需要的镜像和trident安装包和想要的配置文件。
* 如果多台机器情形，需要在每台机器上执行`docker load -i trident.tar`
* 解压trident安装包，`tar -xf $BASE_FOLDER/trident-installer-19.07.0.tar.gz`
* 进入trident-installer目录，执行trident安装指令：`./tridentctl install -n trident`
* 检查是否安装成功`kubectl get pod -n trident`
* 创建并验证第一个后端,注意backend.json填写正确的ontap参数，
`./tridentctl -n trident create backend -f backend.json`
* 验证后端是否生成：`./tridentctl -n trident get backend`
* 创建storage class：`kubectl create -f sample-input/storage-class-basic.yaml`

#### 5. kubesphere安装部署
参考官方部署文档为：https://github.com/kubesphere/ks-installer



