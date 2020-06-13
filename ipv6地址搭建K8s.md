---
title: ipv6地址搭建K8s
date: 2020-01-14 20:16:09
tags:
  - k8s install
  - ipv6
  - kubeadm
top: 100
---
# ipv6地址搭建K8s

### 环境
centos: 7.7
k8s: v1.16.0
<!--more-->
### 提前准备
* 修改主机名
`hostnamectl set-hostname node1`
* 添加ipv6地址及主机名
`vi /etc/hosts`
* 添加操作系统的ipv6的参数，且使参数生效`sysctl -p`
```
vi /etc/sysctl.conf
net.ipv6.conf.all.disable_ipv6 = 0
net.ipv6.conf.default.disable_ipv6 = 0
net.ipv6.conf.lo.disable_ipv6 = 0
net.ipv6.conf.all.forwarding=1
```
* 开启ipv6,添加如下内容
```
vi /etc/sysconfig/network
NETWORKING_IPV6=yes
```

* 开启网卡的ipv6,添加如下内容，最后执行reboot生效。
```
vi /etc/sysconfig/network-scripts/ifcfg-eth0
IPV6INIT=yes
IPV6_AUTOCONF=yes
```
* 关闭防火墙
```
systemctl stop firewalld
systemctl disable firewalld
setenforce 0
vi /etc/selinux/config
SELINUX=disabled
```
* 关闭虚拟内存,添加如下内容，最后通过执行sysctl -p /etc/sysctl.d/k8s.conf生效。
```
swapoff -a
vi /etc/sysctl.d/k8s.conf 添加下面一行：
vm.swappiness=0
```
### 安装docker
```
yum install -y yum-utils device-mapper-persistent-data lvm2
yum install wget -y
wget -O /etc/yum.repos.d/docker-ce.repo https://download.docker.com/linux/centos/docker-ce.repo
sudo sed -i 's+download.docker.com+mirrors.tuna.tsinghua.edu.cn/docker-ce+' /etc/yum.repos.d/docker-ce.repo
yum makecache fast
yum install docker-ce -y
systemctl enable docker;systemctl restart docker
docker的配置vi /etc/docker/daemon.json
{
"insecure-registry":["0.0.0.0/0"],
"ipv6": true,
"fixed-cidr-v6": "2001:db8:1::/64",
"host":["unix:///var/run/docker.sock","tcp://:::2375"],
"log-level":"debug"
}
systemctl restart docker
echo "1" >/proc/sys/net/bridge/bridge-nf-call-iptables
```
### 安装kubectl、kubeadm和kubelet插件

```
添加k8s下载源
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
        http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

指定版本的安装
yum install kubelet-1.16.0 kubeadm-1.16.0 kubectl-1.16.0 -y
systemctl enable kubelet && sudo systemctl start kubelet
```
### 初始化的准备
* 查看安装过程需要哪些镜像
`kubeadm config images list --kubernetes-version=v1.16.0`
* 通过脚本下载所需的镜像。
```
vi images.sh 
#!/bin/bash
images=(kube-proxy:v1.16.0 kube-scheduler:v1.16.0 kube-controller-manager:v1.16.0 kube-apiserver:v1.16.0 etcd:3.3.15-0 pause:3.1 coredns:1.6.2)
for imageName in ${images[@]} ; do
docker pull gcr.azk8s.cn/google-containers/$imageName
docker tag gcr.azk8s.cn/google-containers/$imageName k8s.gcr.io/$imageName
docker rmi gcr.azk8s.cn/google-containers/$imageName
done
```
* 执行如下指令下载：`chmod +x images.sh && ./images.sh`

* 拷贝kubeadm.yaml文件，需要注意advertiseAddress参数为本机ipv6地址

```
vi kubeadm.yaml 
apiVersion: kubeadm.k8s.io/v1beta2
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: "2402:e7c0:0:a00:ffff:ffff:fffe:fffb"
  bindPort: 6443
nodeRegistration:
  taints:
  - effect: PreferNoSchedule
    key: node-role.kubernetes.io/master
---
apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
kubernetesVersion: v1.16.0
networking:
  podSubnet: 1100::/52
  serviceSubnet: fd00:4000::/112
```

* 执行如下安装指令：`kubeadm init --config=kubeadm.yaml`
* 如果使1.16.0之前版本需要安装指令后面添加如下参数执行：`--ignore-preflight-errors=HTTPProxy`
* 如果以下安装有问题，需要重置，先执行`kubeadm reset`,再执行以上`kubeadm init`指令，安装成功之后，需要做如下操作：

```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

kubectl taint node node1 node-role.kubernetes.io/master-
```

### 安装网络插件，如calico
* 下载calico.yaml文件

```
curl https://docs.projectcalico.org/v3.11/manifests/calico.yaml -O
需要修改及添加的内容为,总共三处：
 "ipam": {
     "type": "calico-ipam",
     "assign_ipv4": "false",
     "assign_ipv6": "true",
     "ipv4_pools": ["172.16.0.0/16", "default-ipv4-ippool"],
     "ipv6_pools": ["1100::/52", "default-ipv6-ippool"]
  },

- name: CALICO_IPV4POOL_CIDR
  value: "172.16.0.0/16"
- name: IP6
  value: "autodetect"
- name: CALICO_IPV6POOL_CIDR
  value: "1100::/52"

# Disable IPv6 on Kubernetes.
- name: FELIX_IPV6SUPPORT
  value: "true"
```
* calico的镜像,可以提前下载
calico/cni:v3.11.1
calico/pod2daemon-flexvol:v3.11.1
calico/node:v3.11.1
calico/kube-controllers:v3.11.1
* 执行calico，`kubectl apply -f calico.yaml`

### 验证：
`kubectl get pod --all-namespaces -o wide`
`kubectl get nodes -o wide`

* 部署tomcat应用验证

```
kubectl run tomcat  --image=tomcat:8.0  --port=8080
kubectl get pod
kubectl expose deployment tomcat  --port=8080 --target-port=8080 --type=NodePort
# kubectl get svc
NAME         TYPE        CLUSTER-IP        EXTERNAL-IP   PORT(S)          AGE
kubernetes   ClusterIP   fd00:4000::1      <none>        443/TCP          33m
tomcat       NodePort    fd00:4000::bf3e   <none>        8080:30693/TCP   22m

curl -6g [2402:e7c0:0:a00:ffff:ffff:fffe:fffb]:32012
```
#### 参考
https://www.kubernetes.org.cn/5173.html

