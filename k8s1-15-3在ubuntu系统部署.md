---
title: k8s1.15.3在ubuntu系统部署
date: 2019-09-02 11:58:58
tags: k8s tools
top: 100
---
# k8s1.15.3在ubuntu系统部署
国内环境下，k8s1.15.3在ubuntu系统部署，相关的镜像以及docker的deb包和k8s核心组件的deb包在以下百度链接下。
[k8s介质](https://pan.baidu.com/s/1FqfkBiRfa03xaKbmKnqI2w)
提取码：05ef
<!--more-->
#### 配置
2核4G
k8s：v1.15.3
ubuntu:18.04
### 1. 前期准备
* 关闭ufw防火墙,Ubuntu默认未启用,无需设置。
`sudo ufw disable`

* 禁用SELINUX （ubuntu19.04默认不安装）
`sudo setenforce 0`

* 开启数据包转发,修改/etc/sysctl.conf，开启ipv4转发
`net.ipv4.ip_forward=1 注释取消`

* 防火墙修改FORWARD链默认策略
`sudo iptables -P FORWARD ACCEPT`

* 禁用swap
`sudo swapoff -a`

* 配置iptables参数，使得流经网桥的流量也经过iptables/netfilter防火墙

```
sudo tee /etc/sysctl.d/k8s.conf <<-'EOF'
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF

sudo sysctl --system
```

### 2.docker安装
* 安装deb包,通过dpkg指令
```
dpkg -i containerd.io_1.2.5-1_amd64.deb && \
dpkg -i docker-ce-cli_18.09.5~3-0~ubuntu-bionic_amd64.deb && \
dpkg -i docker-ce_18.09.5~3-0~ubuntu-bionic_amd64.deb
```
* docker使用加速器（阿里云加速器）
```
tee /etc/docker/daemon.json <<- 'EOF'
{
"registry-mirrors": ["https://5xcgs6ii.mirror.aliyuncs.com"]
}
EOF
```
* 设置docker开机自启动
`sudo systemctl enable docker && sudo systemctl start docker`

### 3.安装kubeadm、kubelet、kubectl
* 通过dpkg -i 来安装k8s核心组件，指令如下
```
dpkg -i cri-tools_1.13.0-00_amd64.deb  kubernetes-cni_0.7.5-00_amd64.deb socat_1.7.3.2-2ubuntu2_amd64.deb conntrack_1%3a1.4.4+snapshot20161117-6ubuntu2_amd64.deb kubelet_1.15.3-00_amd64.deb  kubectl_1.15.3-00_amd64.deb kubeadm_1.15.3-00_amd64.deb
```
* 设置开机自启动
`sudo systemctl enable kubelet && sudo systemctl start kubelet`

### 4.kubeadm init初始化集群
* 先要将需要的镜像解压
`docker load -i k8s1153.tar`
* 查看Kubernetes需要哪些镜像
`kubeadm config images list --kubernetes-version=v1.15.3`
* 注意apiserver-advertise-address要换成本机的IP
```
sudo kubeadm init --apiserver-advertise-address=192.168.11.21 --pod-network-cidr=172.16.0.0/16 --service-cidr=10.233.0.0/16 --kubernetes-version=v1.15.3
```
* 创建kubectl使用的kubeconfig文件
`mkdir -p $HOME/.kube`
`sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config`
`sudo chown $(id -u):$(id -g) $HOME/.kube/config`

* 创建flannel的pod，命令如下
以下两个文件在百度下链接下。
`kubectl create -f kube-flannel.yml`
`kubectl apply -f weave-net.yml`

### 5.检查集群及重新添加节点
* 检查node是否ready
`kubectl get nodes`
* 检查pod是否running
`kubectl get pod --all-namespaces -o wide`
* 添加节点，如果忘记token了，可以在master上面执行如下指令获取
`kubeadm token list`
* 添加节点，需要提前在新的机器上安装kubelet等服务及需要把相关的镜像拷贝过去解压。最后通过如下指令添加：
`kubeadm join –token=4fccd2.b0e0f8918bd95d3e 192.168.11.21:6443`

##### 参考文档
[参考](https://blog.csdn.net/qq_42346414/article/details/89949380)
