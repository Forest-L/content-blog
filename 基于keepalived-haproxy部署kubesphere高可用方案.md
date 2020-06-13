---
title: 基于keepalived+haproxy部署kubesphere高可用方案
date: 2019-12-12 00:51:56
tags:
  - haproxy
  - keepalived
top: 100
---
# 基于keepalived+haproxy部署kubesphere高可用方案
通过keepalived + haproxy实现的，其中keepalived提供一个VIP，通过VIP关联所有的Master节点；然后haproxy提供端口转发功能。由于VIP还是存在Master的机器上的，默认配置API Server的端口是6443，所以我们需要将另外一个端口关联到这个VIP上，一般用8443。<!--more-->
#### 环境信息：
```
青云机器操作系统：centos7.5
master1:192.168.0.10
master2:192.168.0.11
master3:192.168.0.12
node:192.168.0.6
vip:192.168.0.200
```
#### 提前说明及遇到过坑

* keepalived提供VIP时，需提前规划的vip不能ping通。
* 腾讯云服务器上不提供由keepalived方式产生VIP，需要提前在云平台界面HAVIP上创建，创建出的vip再在keepalived.conf中配置。（特别注意这点）。
* 通过keepalived服务，vip只能在其中的某一个master中看到，如果ip a方式在每个master都看到，说明keepalived有问题。
* keepalived+haproxy正常安装之后，检查node节点可以和vip通信。
* common.yaml配置文件需要填写正确的ip和转发的端口。
* hosts.ini文件master需要添加master1、master2和master3。

### keepalived安装和配置，三台master机器都要安装。
* 安装keepalived, `yum install -y keepalived`
* /etc/keepalived/keepalived.conf文件下修改配置,需要修改自己场景的vIP,填写正确服务器的网卡名如：eth0。

```
global_defs {
    router_id lb-backup
}

vrrp_instance VI-kube-master {
    state MASTER
    priority 110
    dont_track_primary
    interface eth0
    virtual_router_id 90
    advert_int 3
    virtual_ipaddress {
        192.168.0.200
    }
}
```
重启keepalived服务及断电自启：`systemctl enable keepalived;systemctl restart keepalived`
### haproxy安装和配置，三台master机器都要安装。
* 安装haproyx，`yum install -y haproxy`
* /etc/haproxy/haproxy.cfg 文件下修改配置，server服务端分别为master的IP值。

```
global
        log /dev/log    local0
        log /dev/log    local1 notice
        chroot /var/lib/haproxy
        #stats socket /run/haproxy/admin.sock mode 660 level admin
        stats timeout 30s
        user haproxy
        group haproxy
        daemon
        nbproc 1

defaults
        log     global
        timeout connect 5000
        timeout client  50000
        timeout server  50000

listen kube-master
        bind 0.0.0.0:8443
        mode tcp
        option tcplog
        balance roundrobin
        server master1 192.168.0.10:6443  check inter 10000 fall 2 rise 2 weight 1
        server master2 192.168.0.11:6443  check inter 10000 fall 2 rise 2 weight 1
        server master3 192.168.0.12:6443  check inter 10000 fall 2 rise 2 weight 1
```
重启haproxy服务及断电自启：`systemctl enable haproxy;systemctl restart haproxy`
### hosts.ini配置和common.yaml配置。
* hosts.ini配置实例如下：

```
[all]
master1 ansible_connection=local  ip=192.168.0.10
master2  ansible_host=192.168.0.11  ip=192.168.0.11  ansible_ssh_pass=****
master3  ansible_host=192.168.0.12  ip=192.168.0.12  ansible_ssh_pass=****
node1  ansible_host=192.168.0.6  ip=192.168.0.6  ansible_ssh_pass=****

[kube-master]
master1
master2
master3

[kube-node]
node1


[etcd]
master1
master2
master3

[k8s-cluster:children]
kube-node
kube-master 
```
* common.yaml的lb配置，注意此处填写VIP，及转发的端口8443。

```
# apiserver_loadbalancer_domain_name: "lb.kubesphere.local"
loadbalancer_apiserver:
  address: 192.168.0.200
  port: 8443
```

### 安装kubesphere及结果
kubesphere-all-v2.1.0/scripts目录下，执行./install.sh，选择2+yes即可，然后等待脚本的安装。
node正常的结果：
```
kubectl get nodes -o wide
NAME      STATUS   ROLES    AGE   VERSION   INTERNAL-IP    EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION          CONTAINER-RUNTIME
master1   Ready    master   95m   v1.15.5   192.168.0.10   <none>        CentOS Linux 7 (Core)   3.10.0-862.el7.x86_64   docker://18.9.7
master2   Ready    master   88m   v1.15.5   192.168.0.11   <none>        CentOS Linux 7 (Core)   3.10.0-862.el7.x86_64   docker://18.9.7
master3   Ready    master   88m   v1.15.5   192.168.0.12   <none>        CentOS Linux 7 (Core)   3.10.0-862.el7.x86_64   docker://18.9.7
node1     Ready    worker   86m   v1.15.5   192.168.0.6    <none>        CentOS Linux 7 (Core)   3.10.0-862.el7.x86_64   docker://18.9.7
```
apiserver中vip生效的结果
```
telnet 192.168.0.200 8443
Trying 192.168.0.200...
Connected to 192.168.0.200.
Escape character is '^]'.
^CConnection closed by foreign host.
```






















