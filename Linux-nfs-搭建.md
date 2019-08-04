---
title: Linux nfs 搭建
categories: Linux tools
date: 2019-07-23 16:36:28
tags: Linux tools
top: 100
---
# Linux 下NFS 服务器的搭建及配置

nfs是网络存储文件系统，客户端通过网络访问不同主机上磁盘的数据，用于unix系统。
<!--more-->
## 演示nfs机器信息，分为服务端和客户端介绍
服务端：192.168.0.9，客户端：192.168.0.10
## 1.1服务端安装（192.168.0.9）
<font color=#DC143C >centos:</font>
`sudo yum install nfs-utils -y`

<font color=#DC143C >ubuntu16.04:</font>
`sudo apt install nfs-kernel-server`
## 1.2服务端配置
<font color=#DC143C >centos:</font>
rpcbind服务的开机自启和启动：
`sudo systemctl enable rpcbind;sudo systemctl restart rpcbind`
nfs服务的开机自启和启动：
`sudo systemctl enable nfs;sudo systemctl restart nfs`

<font color=#DC143C >ubuntu16.04:</font>
`sudo systemctl enable nfs-kernel-server;sudo systemctl restart nfs-kernel-server`
## 1.3配置共享目录
目录为服务端目录，后续存储的数据在该目录下
`sudo mkdir -p /nfsdatas`
`sudo chmod 755 /nfsdatas`
<font color=#DC143C >重要:</font> 根据上面创建的目录，相应配置导出目录
`sudo vi /etc/exports`
添加如下配置再保存，重启nfs服务即可：
/nfsdatas/ 192.168.0.0/24(rw,sync,no_root_squash,no_all_squash)
1. /nfsdatas:共享目录位置。
2. 192.168.0.0/24：客户端IP范围，*代表所有。
3. rw：权限设置，可读可写。
4. sync：同步共享目录。
5. no_root_squash: 可以使用root授权。
6. no_all_squash: 可以使用普通用户授权

`systemctl restart nfs`

## 2.1客户端安装（192.168.0.10）
<font color=#DC143C >centos:</font>
`sudo yum install nfs-utils -y`

<font color=#DC143C >ubuntu16.04:</font>
`sudo apt install nfs-common`
## 2.2客户端开机自启和启动即可
<font color=#DC143C >centos:</font>
`sudo systemctl enable rpcbind`
`sudo systemctl start rpcbind`

<font color=#DC143C >ubuntu16.04:</font>
`sudo systemctl enable nfs-common;sudo systemctl restart nfs-common`
## 2.3客户端验证及测试
检查服务端的共享目录：
`showmount -e 192.168.0.9`
客户端创建目录
`sudo mkdir -p /tmp/nfsdata`
挂载指令：
`sudo mount -t nfs 192.168.0.9:/nfsdatas /tmp/nfsdata`
然后进入/tmp/nfsdata目录下，新建文件
`sudo touch test`
之后在nfs服务端192.168.0.9的/nfsdatas目录查看是否有test文件