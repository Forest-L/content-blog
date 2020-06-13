---
title: kubesphere2.1-HA环境，一个master或者两个master宕机恢复
date: 2019-11-13 14:38:51
tags: 
  - k8s tools
  - etcd

top: 100
---
# kubesphere2.1-HA环境，一个master或者两个master宕机恢复
kubesphere2.1版本提供了master节点的高可用功能，为生产环境保驾护航。然而很多事情都有意外，当其中一个master或者两个master卡住了，或者重启都不能自动恢复的情况下，那么怎么恢复呢，以下分一个master宕机和两个master宕机的恢复方法。
<!--more-->

### 验证环境信息
```
os: centos7.5
master1: 192.168.11.6
master2: 192.168.11.8
master3: 192.168.11.13
node1: 192.168.11.14
lb: 192.168.11.253
nfs服务端: 192.168.11.14
```
## 一个master宕机的模拟及恢复方法
* 正常的环境：nodes都running，etcd服务都正常。

```
kubectl get nodes
NAME      STATUS   ROLES    AGE   VERSION
master1   Ready    master   19m   v1.15.5
master2   Ready    master   16m   v1.15.5
master3   Ready    master   16m   v1.15.5
node1     Ready    worker   14m   v1.15.5

export ETCDCTL_API=3

etcdctl --cacert=/etc/ssl/etcd/ssl/ca.pem --cert=/etc/ssl/etcd/ssl/node-master1.pem --key=/etc/ssl/etcd/ssl/node-master1-key.pem --endpoints=192.168.11.6:2379,192.168.11.8:2379,192.168.11.13:2379 endpoint status
192.168.11.6:2379, b3487da7c562b17, 3.2.18, 3.8 MB, true, 5, 4434
192.168.11.8:2379, 3d04d74d80b7f07b, 3.2.18, 3.8 MB, false, 5, 4434
192.168.11.13:2379, 3ad323c042cc4c05, 3.2.18, 3.8 MB, false, 5, 4434
```

* 一个master宕机情况，把master2重置，看nodes和etcd情况。还需在界面创建一些带有存储的pod用例。

```
kubectl get nodes   
NAME      STATUS     ROLES    AGE   VERSION
master1   Ready      master   57m   v1.15.5
master2   NotReady   master   55m   v1.15.5
master3   Ready      master   55m   v1.15.5
node1     Ready      worker   52m   v1.15.5

etcdctl --cacert=/etc/ssl/etcd/ssl/ca.pem --cert=/etc/ssl/etcd/ssl/node-master1.pem --key=/etc/ssl/etcd/ssl/node-master1-key.pem --endpoints=192.168.11.6:2379,192.168.11.8:2379,192.168.11.13:2379 endpoint status
Failed to get the status of endpoint 192.168.11.8:2379 (context deadline exceeded)
192.168.11.6:2379, b3487da7c562b17, 3.2.18, 11 MB, true, 5, 14953
192.168.11.13:2379, 3ad323c042cc4c05, 3.2.18, 11 MB, false, 5, 14972
```
* 恢复方法：修改脚本中hosts.ini文件，需要注意顺序
在[all]组里面，master2行整体放在master3下面，[kube-master]和[etcd]组中master2放在master3下面,然后在另一个终端机器上重新执行安装脚本。以下恢复情况，etcd正常，nodes也正常，业务数据存在且业务pod没有中断正常使用。

```
kubectl get nodes
NAME      STATUS     ROLES    AGE   VERSION
master1   Ready      master   83m   v1.15.5
master2   Ready      master   80m   v1.15.5
master3   Ready      master   80m   v1.15.5
node1     Ready      worker   78m   v1.15.5

etcdctl --cacert=/etc/ssl/etcd/ssl/ca.pem --cert=/etc/ssl/etcd/ssl/node-master1.pem --key=/etc/ssl/etcd/ssl/node-master1-key.pem --endpoints=192.168.11.6:2379,192.168.11.8:2379,192.168.11.13:2379 endpoint status
192.168.11.6:2379, b3487da7c562b17, 3.2.18, 11 MB, true, 11, 20292
192.168.11.8:2379, 3d04d74d80b7f07b, 3.2.18, 11 MB, false, 11, 20297
192.168.11.13:2379, 3ad323c042cc4c05, 3.2.18, 11 MB, false, 11, 20298

docker ps | grep tomcat
6863620b07cf        882487b8be1d                                     "catalina.sh run"        17 minutes ago       Up 17 minutes                           k8s_tomcat-2jl160_tomcat-v1-6c7745494-k64w5_demo_5b406841-df9b-4d5c-b018-998c400a2c3e_1
```
## 两个master宕机的模拟及恢复方法
* 两个master宕机情况，把master2和master3重置，看nodes和etcd情况，nodes不正常，etcd两个不正常。

```
kubectl get nodes
Unable to connect to the server: EOF
[root@master1 ~]# 
[root@master1 ~]# 
[root@master1 ~]# etcdctl --cacert=/etc/ssl/etcd/ssl/ca.pem --cert=/etc/ssl/etcd/ssl/node-master1.pem --key=/etc/ssl/etcd/ssl/node-master1-key.pem --endpoints=192.168.11.6:2379,192.168.11.8:2379,192.168.11.13:2379 endpoint status
Failed to get the status of endpoint 192.168.11.8:2379 (context deadline exceeded)
Failed to get the status of endpoint 192.168.11.13:2379 (context deadline exceeded)
192.168.11.6:2379, b3487da7c562b17, 3.2.18, 11 MB, false, 12, 27944
```
* 恢复方法：
1、同样需要在host.ini文件修改master的顺序，由于我们重置2和3，所有此处不用修改顺序；
2、在已解压的安装介质目录下，进入k8s/roles/kubernetes/preinstall/tasks/main.yml文件，用#注释如下内容，重跑安装脚本，作用说明：在重置的master2和master3机器上安装docker和etcd，但整个集群还需修复。。
```
#- import_tasks: 0020-verify-settings.yml
#  when:
#    - not dns_late
#  tags:
#    - asserts
```
3、在master2机器上，临时修复master2节点的etcd服务，执行如下指令：
```
停止etcd：systemctl stop etcd
备份etcd数据：mv /var/lib/etcd /var/lib/etcd-bak
从master1的/var/backups/kube_etcd/备份目录下拷贝最近时间的snapshot.db至master2机器上
在master2先转为版本3指令令：export ETCDCTL_API=3
在master2恢复指令：etcdctl snapshot restore /root/snapshot.db    --endpoints=192.168.11.8:2379    --cert=/etc/ssl/etcd/ssl/node-master2.pem    --key=/etc/ssl/etcd/ssl/node-master2-key.pem    --cacert=/etc/ssl/etcd/ssl/ca.pem --data-dir=/var/lib/etcd
重启etcd: systemctl restart etcd
```
4、修改host.ini文件master2和master3顺序，把[all][kube-master][etcd]三个组的master3放在最后面，再次跑安装脚本。其中的host.ini文件为

```
[all]
master1 ansible_connection=local  ip=192.168.11.6
master2  ansible_host=192.168.11.8  ip=192.168.11.8  ansible_ssh_pass=
master3  ansible_host=192.168.11.13  ip=192.168.11.13  ansible_ssh_pass=
node1    ansible_host=192.168.11.14  ip=192.168.11.14  ansible_ssh_pass=

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
5、由第四步只是临时修复master2的etcd，并不完全修复，先全部修复作如下处理：
```
第四步执行过程中，z在“wait for etcd up”会报错，先ctrl +c 终止脚本；
在master2机器上，停止etcd服务：systemctl stop etcd
在master2机器上，删除etcd数据：rm -rf /var/lib/etcd
再次修改host.ini文件，master2和master3顺序，把[all][kube-master][etcd]三个组的master2放在最后面，再次跑安装脚本即可。
```
## 验证结果
* 集群中的nodes、etcd和带有存储数据的业务pod情况：
* 集群中的nodes、etcd和带有存储数据的业务pod情况,nodes恢复正常，etcd服务也回复正常，带有存储的pod一直运行，集群恢复成功：

```
[root@master1 ~]# kubectl get nodes
NAME      STATUS   ROLES    AGE     VERSION
master1   Ready    master   4h56m   v1.15.5
master2   Ready    master   4h53m   v1.15.5
master3   Ready    master   4h53m   v1.15.5
node1     Ready    worker   4h51m   v1.15.5
[root@master1 ~]# 
[root@master1 ~]# 
[root@master1 ~]# 
[root@master1 ~]# etcdctl --cacert=/etc/ssl/etcd/ssl/ca.pem --cert=/etc/ssl/etcd/ssl/node-master1.pem --key=/etc/ssl/etcd/ssl/node-master1-key.pem --endpoints=192.168.11.6:2379,192.168.11.8:2379,192.168.11.13:2379 endpoint status
192.168.11.6:2379, b3487da7c562b17, 3.2.18, 12 MB, false, 1372, 32116
192.168.11.8:2379, 3d04d74d80b7f07b, 3.2.18, 12 MB, true, 1372, 32116
192.168.11.13:2379, 3ad323c042cc4c05, 3.2.18, 12 MB, false, 1372, 32116

docker ps | grep tomcat
6863620b07cf        882487b8be1d                                     "catalina.sh run"        4 hours ago         Up 4 hours                              k8s_tomcat-2jl160_tomcat-v1-6c7745494-k64w5_demo_5b406841-df9b-4d5c-b018-998c400a2c3e_1
```
