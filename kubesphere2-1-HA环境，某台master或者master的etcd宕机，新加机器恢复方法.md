---
title: kubesphere2.1-HA环境，某台master或者master的etcd宕机，新加机器恢复方法
date: 2019-11-18 22:05:42
tags: 
  - k8s tools
  - etcd

top: 100
---
kubesphere2.1版本提供了master节点的高可用功能，为生产环境保驾护航。然而在生产环境中，为了业务更正常运行，当以下两种情形发生时，告诉大家怎么恢复。第一种情形是：其中某台master机器的etcd服务不能正常提供服务，而需要在另外一台机器上部署一个etcd服务加入到现etcd集群中；第二种情形是：其中某台master宕机，需要在另外一台机器上部署master，并加入到现master集群中。
<!--more-->
### 环境信息
```
os: centos7.5
master1: 192.168.11.6
master2: 192.168.11.16
master3: 192.168.11.13
node1: 192.168.11.14
lb: 192.168.11.253
nfs服务端: 192.168.11.14
新加机器master2: 192.168.11.8
安装介质机器：192.168.11.6
```
## 一个master宕机，在另外一台服务器上恢复方法
假设为master2（192.168.11.16）宕机了，现以新机器192.168.11.8作为恢复master2的机器。以下操作都在master1机器上执行。
1、在安装介质机器即master1机器上面，进入/etc/ssl/etcd/ssl目录下，将含有master2相关联的文件证书删掉（那个master有问题就出来那个）。
`rm -rf admin-master2-key.pem admin-master2.pem member-master2-key.pem member-master2.pem node-master2-key.pem node-master2.pem`
2、将etcd集群中master2的节点移除。
```
第一步先转为etcd3版本：export ETCDCTL_API=3
查看etcd集群的成员：
etcdctl --cacert=/etc/ssl/etcd/ssl/ca.pem --cert=/etc/ssl/etcd/ssl/node-master1.pem --key=/etc/ssl/etcd/ssl/node-master1-key.pem member list
b3487da7c562b17, started, etcd1, https://192.168.11.6:2380, https://192.168.11.6:2379
3ad323c042cc4c05, started, etcd2, https://192.168.11.13:2380, https://192.168.11.13:2379
52a4779150442b41, started, etcd3, https://192.168.11.16:2380, https://192.168.11.16:2379
第三步：移除master2节点，如192.168.11.16
etcdctl member remove 52a4779150442b41 --cacert=/etc/ssl/etcd/ssl/ca.pem --cert=/etc/ssl/etcd/ssl/node-master1.pem  --key=/etc/ssl/etcd/ssl/node-master1-key.pem
```

3、新打开master1机器窗口进入解压包目录下修改conf/hosts.ini,一定要新打开窗口（由于上一步操作使master1机器为etcd3版本指令）。
master2的IP由原来的192.168.11.16改成192.168.11.8；在[all]组里面，master2行整体放在master3下面，[kube-master]和[etcd]组中master2放在master3下面,然后执行安装脚本即可。
4、验证结果：
用`kubectl get nodes -o wide`指令看master2IP是否替换；
看etcd集群中是否包含新的master2IP，指令为：
```
1、开启etcd3版本：export ETCDCTL_API=3
2、etcdctl --cacert=/etc/ssl/etcd/ssl/ca.pem --cert=/etc/ssl/etcd/ssl/node-master1.pem --key=/etc/ssl/etcd/ssl/node-master1-key.pem --endpoints=192.168.11.6:2379,192.168.11.8:2379,192.168.11.13:2379 endpoint status
```

## 一个master中etcd服务不正常，在另外一台服务器上恢复etcd方法
假设为master2（192.168.11.16）宕机了，现以新机器192.168.11.8作为恢复master2的机器。以下操作都在master1机器上执行。
1、在安装介质机器即master1机器上面，进入/etc/ssl/etcd/ssl目录下，将含有master2相关联的文件证书删掉（那个master有问题就出来那个）。
`rm -rf admin-master2-key.pem admin-master2.pem member-master2-key.pem member-master2.pem node-master2-key.pem node-master2.pem`
2、将etcd集群中master2的节点移除。
```
第一步先转为etcd3版本：export ETCDCTL_API=3
查看etcd集群的成员：
etcdctl --cacert=/etc/ssl/etcd/ssl/ca.pem --cert=/etc/ssl/etcd/ssl/node-master1.pem --key=/etc/ssl/etcd/ssl/node-master1-key.pem member list
b3487da7c562b17, started, etcd1, https://192.168.11.6:2380, https://192.168.11.6:2379
3ad323c042cc4c05, started, etcd2, https://192.168.11.13:2380, https://192.168.11.13:2379
52a4779150442b41, started, etcd3, https://192.168.11.16:2380, https://192.168.11.16:2379
第三步：移除master2节点，如192.168.11.16
etcdctl member remove 52a4779150442b41 --cacert=/etc/ssl/etcd/ssl/ca.pem --cert=/etc/ssl/etcd/ssl/node-master1.pem  --key=/etc/ssl/etcd/ssl/node-master1-key.pem
```

3、新打开master1机器窗口进入解压包目录下修改conf/hosts.ini,一定要新打开窗口（由于上一步操作使master1机器为etcd3版本指令）。
master2的IP由原来的192.168.11.16改成192.168.11.8；在[all]组里面，master2行整体放在master3下面，[kube-master]和[etcd]组中master2放在master3下面。
4、进入解压包，scripts目录下，编辑install.sh脚本，用#将如下内容注释掉：
`ansible-playbook -i $BASE_FOLDER/../k8s/inventory/my_cluster/hosts.ini $BASE_FOLDER/../kubesphere/kubesphere.yml -b`
5、进入解压包，k8s目录下，编辑cluster.yml文件，用#将以下开头的内容至结尾都注释掉
```
- hosts: k8s-cluster
  any_errors_fatal: "{{ any_errors_fatal | default(true) }}"
  roles:
    - { role: kubespray-defaults}
    - { role: kubernetes/node, tags: node }
  environment: "{{ proxy_env }}"
```

6、重新到脚本目录，执行install.sh脚本即可。
7、验证结果：
看etcd集群中是否包含新的master2IP，指令为：
```
1、开启etcd3版本：export ETCDCTL_API=3
2、etcdctl --cacert=/etc/ssl/etcd/ssl/ca.pem --cert=/etc/ssl/etcd/ssl/node-master1.pem --key=/etc/ssl/etcd/ssl/node-master1-key.pem member list
```

