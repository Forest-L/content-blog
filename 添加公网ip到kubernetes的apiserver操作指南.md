---
title: 添加公网ip到kubernetes的apiserver操作指南
date: 2020-06-14 00:28:19
tags: 
  - kubernetes
  - apiserver
  - 外网ip
  - certSANs

top: 100
---
通常情况下，我们的kubernetes集群是内网环境，如果希望通过本地访问这个集群，怎么办呢？大家想到的是Kubeadm在初始化的时候会为管理员生成一个 Kubeconfig文件，把它下载下来 是不是就可以？事实证明这样不行， 因为这个集群是内网集群，Kubeconfig文件 中APIServer的地址是内网ip。解决方案很简单，把公网ip签到证书里面就可以，其中有apiServerCertSANs这个选项，只要把公网IP写到这里，再启动这个集群的时候，这个公网ip就会签到证书里。
<!--more-->
## 1. 环境信息
* 安装方式：kubeadm
* 内网IP：192.168.0.8
* 外网IP：139.198.19.37
* 证书目录：/etc/kubernetes/pki
* kubeadm配置文件目录：/etc/kubernetes/kubeadm-config.yaml

## 2. 查看apiserver的证书包含的ip,进入到证书目录执行
`cd /etc/kubernetes/pki`
`openssl x509 -in apiserver.crt -noout -text`

```
openssl x509 -in apiserver.crt -noout -text
Certificate:
    Data:
        ................
        Validity
            Not Before: Jun  5 02:26:44 2020 GMT
            Not After : Jun  5 02:26:44 2021 GMT
        ..................
        X509v3 extensions:
            ..........
            X509v3 Subject Alternative Name:
                IP Address:192.168.0.8
    .......
```

## 3. 添加公网IP到apiserver
绑定的公网ip为 139.198.19.37 ，确保公网ip的防火墙已经打开6443端口
* 3.1 登录到主节点，进入 /etc/kubernetes/目录下
* 3.2 修改kubeadm-config.yaml，找到 ClusterConfiguration 中的 	certSANs (如无，在 apiServer 下添加这一配置)，如下。添加刚才绑定的 139.198.19.37 到 certSANs 下，保存文件。
```
apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
etcd:
  external:
    endpoints:
    - https://192.168.0.8:2379
apiServer:
  extraArgs:
    authorization-mode: Node,RBAC
  timeoutForControlPlane: 4m0s
  certSANs:
    - 192.168.0.8
    - 139.198.19.37
```
* 3.3 执行如下命令更新 apiserver.crt apiserver.key
注意需要把之前apiserver.crt apiserver.key做备份,进入到pki目录下，执行如下指令做备份：
`mv apiserver.crt apiserver.crt-bak`
`mv apiserver.key apiserver.key-bak`
备份完之后，回到/etc/kubernetes目录下，执行公网ip添加到apiserver操作指令为：
kubeadm init phase certs apiserver --config kubeadm-config.yaml
```
kubeadm init phase certs apiserver --config kubeadm-config.yaml
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [192.168.0.8  139.198.19.37]
```
* 3.4 再次查看apiserver中证书包含的ip，指令如下,看的公网ip则操作成功。
openssl x509 -in pki/apiserver.crt -noout -text | grep 139.198.19.37
```
openssl x509 -in pki/apiserver.crt -noout -text | grep 139.198.19.37
                IP Address:192.168.0.8, IP Address:139.198.19.37
```
* 3.5 重启kube-apiserver
如果是高可用集群，直接杀死当前节点的kube-apiserver进程，等待kubelet拉起kube-apiserver即可。需要在三个节点执行步骤1到步骤4，逐一更新。
如果是非高可用集群，杀死kube-apiserver可能会导致服务有中断，需要在业务低峰的时候操作。
进入/etc/kubernetes/manifests目录下，mv kube-apiserver.yaml文件至别的位置，然后又移回来即可
* 3.6 修改kubeconfig中的server ip地址为 139.198.19.37，保存之后就可以直接通过公网访问kubernetes集群
`kubectl --kubeconfig config config view`
`kubectl --kubeconfig config get node`





