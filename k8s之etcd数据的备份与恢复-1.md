---
title: k8s之etcd数据的备份与恢复
date: 2019-10-09 08:41:37
tags:
  - k8s tools
  - k8s etcd
top: 100
---
# k8s之etcd备份与恢复
etcd是一款开源的分布式一致性键值存储。目前有版本为V3以上，但是它的API又有v2和v3之分，以至于操作指令也不一样。
查看etcd版本`etcdctl --version`
<!--more-->
<font color=#DC143C >说 明 : </font>
若使用 v3 备份数据时存在 v2 的数据则不影响恢复

若使用 v2 备份数据时存在 v3 的数据则恢复失败

### 1、对于API2备份与恢复方法
* snap: 存放快照数据,etcd防止WAL文件过多而设置的快照，存储etcd数据状态。
* wal: 存放预写式日志,最大的作用是记录了整个数据变化的全部历程。在etcd中，所有数据的修改在提交前，都要先写入到WAL中。

* 备份指令：
`etcdctl backup --data-dir /home/etcd/ --backup-dir /home/etcd_backup`

* 恢复指令：
`etcd -data-dir=/home/etcd_backup/ -force-new-cluster`

恢复时会覆盖 snapshot 的元数据(member ID 和 cluster ID)，所以需要启动一个新的集群。

### 2、对于API3备份与恢复方法
在使用 API 3 时需要使用环境变量 ETCDCTL_API 明确指定。
`export ETCDCTL_API=3`

#### 2.1备份数据
```
etcdctl --endpoints localhost:2379 \
   --cert=/etc/ssl/etcd/ssl/node-master.pem \
   --key=/etc/ssl/etcd/ssl/node-master-key.pem \
   --cacert=/etc/ssl/etcd/ssl/ca.pem \snapshot save \
   /var/backups/kube_etcd/snapshot.db
```

<font color=#DC143C >说 明 : </font>
* /var/backups/kube_etcd这个目录是根宿主机的/var/lib/etcd目录相映射的，所以备份在这个目录在对应的宿主机上也是能看见的。

* 这些证书对应文件可以直接在etcd容器内通过ps aux|more看见
其中–cert-file对应–cert，–key对应–key-file –cacert对应–trusted-ca-file

#### 2.2恢复数据
* 停止三台master节点的kube-apiserver，指令为：
`mv /etc/kubernetes/manifests/kube-apiserver.yaml /etc/kubernetes/kube-apiserver.yaml`

* 在三个master节点停止 etcd 服务,指令为：
`systemctl stop etcd`

* 在三个master节点转移并备份当前 etcd 集群数据,指令为：
`mv /var/lib/etcd /var/lib/etcd.bak`

* 将最新的etcd备份数据恢复至三个master节点，其中master_ip为不同master主机的IP
指令为：
```
export ETCDCTL_API=3 和
etcdctl snapshot restore /var/backups/kube_etcd/etcd-******/snapshot.db \
   --endpoints=master_ip:2379 \
   --cert=/etc/ssl/etcd/ssl/node-master.pem \
   --key=/etc/ssl/etcd/ssl/node-master-key.pem \
   --cacert=/etc/ssl/etcd/ssl/ca.pem \
   --data-dir=/var/lib/etcd
```

* 执行 etcd 恢复命令,指令为：
`systemctl restart etcd`

* 重启 kube-apiserver,指令为：
`mv /etc/kubernetes/kube-apiserver.yaml /etc/kubernetes/manifests/kube-apiserver.yaml `

* 检查是否正常,指令为：
`kubectl get pod  --all-namespaces`

* 检查etcd集群状态及成员指令为：
`etcdctl --endpoints=https://192.168.0.91:2379 --cacert=/etc/ssl/etcd/ssl/ca.pem --cert=/etc/ssl/etcd/ssl/node-ks-allinone.pem --key=/etc/ssl/etcd/ssl/node-ks-allinone-key.pem member list`
