---
title: centos系统k8s-1.16版本安装
date: 2019-09-22 21:48:14
tags: k8s tools
top: 100
---
# centos系统k8s-1.16版本安装
k8s1.16版本相对之前版本变化不小，亮点和升级参看[v1.16说明](http://k8smeetup.com/article/N1lqGc0i8v)。相关联的镜像和v1.16二进制包上传至百度云上，链接如下[k8s1.16介质，ftq5](https://pan.baidu.com/s/19khl0Hn5ZnZ8TvbO5HZVww)
<!--more--> 
## 1.准备
### 1.1系统准备
需要将主机ip和主机名放在每台机器的`vi /etc/hosts`下
`192.168.11.21 i-fahx5c7k`
`192.168.11.22 i-ouaaujhz`
如果各个主机启用了防火墙，需要开启各个组件所需要的端口，可以查看https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/
这里各个节点禁用防火墙
`systemctl stop firewalld`
`systemctl disable firewalld`
禁用selinux
`setenforce 0`
vi /etc/selinux/config
SELINUX=disabled
创建vi /etc/sysctl.d/k8s.conf文件，添加如下内容：
`net.bridge.bridge-nf-call-ip6tables = 1`
`net.bridge.bridge-nf-call-iptables = 1`
`net.ipv4.ip_forward = 1`
执行命令使修改生效
`modprobe br_netfilter`
`sysctl -p /etc/sysctl.d/k8s.conf`
### 1.2 kube-proxy开启ipvs的前置条件
在所有的节点上执行如下脚本：
```
cat > /etc/sysconfig/modules/ipvs.modules <<EOF
#!/bin/bash
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack_ipv4
EOF
chmod 755 /etc/sysconfig/modules/ipvs.modules && bash /etc/sysconfig/modules/ipvs.modules && lsmod | grep -e ip_vs -e nf_conntrack_ipv4

各个节点需要安装 ipset ipvsadm
yum install ipset ipvsadm -y
```
### 1.3 docker安装
安装docker的yum源,国内寻找清华源
`yum install wget -y`
`yum install -y yum-utils device-mapper-persistent-data lvm2`
`wget -O /etc/yum.repos.d/docker-ce.repo https://download.docker.com/linux/centos/docker-ce.repo`
`sed -i 's+download.docker.com+mirrors.tuna.tsinghua.edu.cn/docker-ce+' /etc/yum.repos.d/docker-ce.repo`
`yum makecache fast`
`yum install docker-ce -y`
	重启docker：`systemctl enable docker;systemctl restart docker`
修改docker cgroup driver为systemd
创建或修改`vi /etc/docker/daemon.json`：
```
{
  "exec-opts": ["native.cgroupdriver=systemd"]
}
```
重启docker:`systemctl restart docker`


## 2. 使用kubeadm部署kubernetes
### 2.1 安装kubeadm和kubelet
下面在各节点安装kubeadm和kubelet和kubectl，请在百度云上面下载rpm包，在k8s116目录下执行如下指令：
`yum install -y cri-tools-1.13.0-0.x86_64.rpm cni-0.7.5-0.x86_64.rpm kubelet-1.16.0-0.x86_64.rpm kubectl-1.16.0-0.x86_64.rpm kubeadm-1.16.0-0.x86_64.rpm `
k8s 1.8开始要求关闭系统的swap,不关闭，kubelet将无法启动，执行指令方法：
`swapoff -a`
修改 /etc/fstab 文件，注释掉 SWAP 的自动挂载，使用free -m确认swap已经关闭。 swappiness参数调整，修改`vi /etc/sysctl.d/k8s.conf`添加下面一行：
`vm.swappiness=0`
执行`sysctl -p /etc/sysctl.d/k8s.conf`使修改生效。
因为这里本次测试主机上还运行其他服务，关闭swap可能会对其他服务产生影响，所以这里修改kubelet的配置去掉这个限制。
修改`vi /etc/sysconfig/kubelet`，加入：`KUBELET_EXTRA_ARGS=--fail-swap-on=false`
### 2.2 使用kubeadm init初始化集群
在各节点开机启动kubelet服务：`systemctl enable kubelet`
使用`kubeadm config print init-defaults`可以打印集群初始化默认的使用的配置:
```
apiVersion: kubeadm.k8s.io/v1beta2
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: abcdef.0123456789abcdef
  ttl: 24h0m0s
  usages:
  - signing
  - authentication
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 1.2.3.4
  bindPort: 6443
nodeRegistration:
  criSocket: /var/run/dockershim.sock
  name: node1
  taints:
  - effect: NoSchedule
    key: node-role.kubernetes.io/master
---
apiServer:
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta2
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controllerManager: {}
dns:
  type: CoreDNS
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: k8s.gcr.io
kind: ClusterConfiguration
kubernetesVersion: v1.14.0
networking:
  dnsDomain: cluster.local
  serviceSubnet: 10.96.0.0/12
scheduler: {}
```
从默认的配置中可以看到，可以使用imageRepository定制在集群初始化时拉取k8s所需镜像的地址。advertiseAddress的ip替换本机，基于默认配置定制出本次使用kubeadm初始化集群所需的配置文件且在11.21机器上`vi kubeadm.yaml`：
```
apiVersion: kubeadm.k8s.io/v1beta2
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 192.168.11.21
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
  podSubnet: 10.244.0.0/16
```
<font color=#DC143C >说 明 : </font>
使用kubeadm默认配置初始化的集群，会在master节点打上node-role.kubernetes.io/master:NoSchedule的污点，阻止master节点接受调度运行工作负载。这里测试环境只有两个节点，所以将这个taint修改为node-role.kubernetes.io/master:PreferNoSchedule。

在开始初始化集群之前，kubeadm config images pull查看需要哪些镜像,需要从百度云上面下载tar包镜像下来，tar通过scp分别传到各个节点执行docker load -i解压，
镜像列表:
```
k8s.gcr.io/kube-apiserver            v1.16.0             b305571ca60a        42 hours ago        217MB
k8s.gcr.io/kube-proxy                v1.16.0             c21b0c7400f9        42 hours ago        86.1MB
k8s.gcr.io/kube-controller-manager   v1.16.0             06a629a7e51c        42 hours ago        163MB
k8s.gcr.io/kube-scheduler            v1.16.0             301ddc62b80b        42 hours ago        87.3MB
k8s.gcr.io/etcd                      3.3.15-0            b2756210eeab        2 weeks ago         247MB
k8s.gcr.io/coredns                   1.6.2               bf261d157914        5 weeks ago         44.1MB
gcr.io/kubernetes-helm/tiller        v2.14.1             ac22eb1f780e        3 months ago        94.2MB
quay.io/coreos/flannel               v0.11.0-amd64       ff281650a721        7 months ago        52.6MB
k8s.gcr.io/pause                     3.1                 da86e6ba6ca1        21 months ago       742kB
radial/busyboxplus                   curl                71fa7369f437        5 years ago         4.23MB
```
接下来使用kubeadm初始化集群，选择node1作为Master Node，在node1上执行下面的命令：`kubeadm init --config kubeadm.yaml --ignore-preflight-errors=Swap`

`kubeadm join 192.168.11.21:6443 --token k6s0kn.nyao1ulwwmlm2org \
    --discovery-token-ca-cert-hash sha256:b721d25356668f921c90bf5f554836cb346b827a62b78797d0bcbcba24214725 `
```
其中关键步骤：
* [kubelet-start] 生成kubelet的配置文件”/var/lib/kubelet/config.yaml”
* [certs]生成相关的各种证书
* [kubeconfig]生成相关的kubeconfig文件
* [control-plane]使用/etc/kubernetes/manifests目录中的yaml文件创建apiserver、controller-manager、scheduler的静态pod
* [bootstraptoken]生成token记录下来，后边使用kubeadm join往集群中添加节点时会用到
* 下面的命令是配置常规用户如何使用kubectl访问集群：
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
* 最后给出了将节点加入集群的命令kubeadm join 192.168.99.11:6443 –token 4qcl2f.gtl3h8e5kjltuo0r \ –discovery-token-ca-cert-hash sha256:7ed5404175cc0bf18dbfe53f19d4a35b1e3d40c19b10924275868ebf2a3bbe6e
```
需要在11.21机器上执行：
`mkdir -p $HOME/.kube`
`sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config`
`sudo chown $(id -u):$(id -g) $HOME/.kube/config`
查看集群状态，确认组件都处于healthy状态：
`kubectl get cs`
集群初始化如果遇到问题，可以使用下面的命令进行清理：
`kubeadm reset`
### 2.3 安装Pod Network
接下来安装flannel network add-on：
`mkdir -p ~/k8s/`
`cd ~/k8s`
`curl -O https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml`
`kubectl apply -f  kube-flannel.yml`
这里注意kube-flannel.yml这个文件里的flannel的镜像是0.11.0，quay.io/coreos/flannel:v0.11.0-amd64
注意需要在vi /var/lib/kubelet/kubeadm-flags.env文件配置中去掉--network-plugin=cni,然后重启kubelet,
```
systemctl daemon-reload
systemctl restart kubelet
```
如果Node有多个网卡的话，参考https://github.com/kubernetes/kubernetes/issues/39701，
目前需要在kube-flannel.yml中使用–iface参数指定集群主机内网网卡的名称，否则可能会出现dns无法解析。需要将kube-flannel.yml下载到本地，flanneld启动参数加上–iface=<iface-name>
```
containers:
      - name: kube-flannel
        image: quay.io/coreos/flannel:v0.11.0-amd64
        command:
        - /opt/bin/flanneld
        args:
        - --ip-masq
        - --kube-subnet-mgr
        - --iface=eth1
......
```
使用`kubectl get pods –-all-namespaces -o wide`确保所有的Pod都处于Running状态。
```
kube-flannel.yml
[root@i-fahx5c7k k8s]# kubectl get pods --all-namespaces -o wide
NAMESPACE     NAME                                 READY   STATUS    RESTARTS   AGE     IP              NODE         NOMINATED NODE   READINESS GATES
kube-system   coredns-5c98db65d4-nbb4w             1/1     Running   0          6m29s   10.244.0.2      i-fahx5c7k   <none>           <none>
kube-system   coredns-5c98db65d4-wtm58             1/1     Running   0          6m29s   10.244.0.3      i-fahx5c7k   <none>           <none>
kube-system   etcd-i-fahx5c7k                      1/1     Running   0          5m26s   192.168.11.21   i-fahx5c7k   <none>           <none>
kube-system   kube-apiserver-i-fahx5c7k            1/1     Running   0          5m37s   192.168.11.21   i-fahx5c7k   <none>           <none>
kube-system   kube-controller-manager-i-fahx5c7k   1/1     Running   0          5m45s   192.168.11.21   i-fahx5c7k   <none>           <none>
kube-system   kube-flannel-ds-amd64-bqswg          1/1     Running   0          58s     192.168.11.21   i-fahx5c7k   <none>           <none>
kube-system   kube-proxy-zhzxj                     1/1     Running   0          6m29s   192.168.11.21   i-fahx5c7k   <none>           <none>
kube-system   kube-scheduler-i-fahx5c7k            1/1     Running   0          5m20s   192.168.11.21   i-fahx5c7k   <none>           <none>
```
### 2.4 测试集群DNS是否可用
`kubectl run curl --image=radial/busyboxplus:curl -it`
进入后执行`nslookup kubernetes.default`确认解析正常:
```
[ root@curl-6bf6db5c4f-f8jjn:/ ]$ nslookup kubernetes.default
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      kubernetes.default
Address 1: 10.96.0.1 kubernetes.default.svc.cluster.local
```
### 2.5 向Kubernetes集群中添加Node节点
下面将11.22这个主机添加到Kubernetes集群中，在11.22机器上执行:
`kubeadm join 192.168.11.21:6443 --token k6s0kn.nyao1ulwwmlm2org \
    --discovery-token-ca-cert-hash sha256:b721d25356668f921c90bf5f554836cb346b827a62b78797d0bcbcba24214725`
11.22加入集群很是顺利，下面在master节点上执行命令查看集群中的节点：
```
[root@i-fahx5c7k k8s]# kubectl get nodes
NAME         STATUS   ROLES    AGE   VERSION
i-fahx5c7k   Ready    master   13m   v1.15.1
i-ouaaujhz   Ready    <none>   50s   v1.15.1
```
#### 2.5.1 如何从集群中移除Node
如果需要从集群中移除11.22这个Node执行下面的命令：
在master节点上执行：
`kubectl drain i-ouaaujhz --delete-local-data --force --ignore-daemonsets`

`kubectl delete node i-ouaaujhz`
在11.22上执行：
`kubeadm reset`
### kube-proxy开启ipvs
修改ConfigMap的kube-system/kube-proxy中的config.conf，mode: “ipvs”
`kubectl edit cm kube-proxy -n kube-system`
之后重启各个节点上的kube-proxy pod：
`kubectl get pod -n kube-system | grep kube-proxy | awk '{system("kubectl delete pod "$1" -n kube-system")}'`
日志查看：`kubectl logs kube-proxy-62ntf  -n kube-system`出现ipvs即开启。
## 3.Kubernetes常用组件部署
使用Helm这个Kubernetes的包管理器，这里也将使用Helm安装Kubernetes的常用组件。
### 3.1 Helm的安装

Helm由客户端命helm令行工具和服务端tiller组成，Helm的安装十分简单。 下载helm命令行工具到master节点node1的/usr/local/bin下，这里下载的2.14.1版本：

`curl -O https://get.helm.sh/helm-v2.14.1-linux-amd64.tar.gz`
`tar -zxvf helm-v2.14.1-linux-amd64.tar.gz`
`cd linux-amd64/`
`cp helm /usr/local/bin/`
为了安装服务端tiller，还需要在这台机器上配置好kubectl工具和kubeconfig文件，确保kubectl工具可以在这台机器上访问apiserver且正常使用。 这里的11.21节点已经配置好了kubectl。
`helm init --output yaml > tiller.yaml`
更新 tiller.yaml 两处：apiVersion 版本;增加选择器
```
apiVersion: apps/v1
kind: Deployment
...
spec:
  replicas: 1
  strategy: {}
  selector:
    matchLabels:
      app: helm
      name: tiller
```
创建：`kubectl create -f tiller.yaml`
因为Kubernetes APIServer开启了RBAC访问控制，所以需要创建tiller使用的service account: tiller并分配合适的角色给它。这里简单起见直接分配cluster-admin这个集群内置的ClusterRole给它。
```
kubectl create serviceaccount --namespace kube-system tiller
kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
kubectl patch deploy --namespace kube-system tiller-deploy -p '{"spec":{"template":{"spec":{"serviceAccount":"tiller"}}}}'
```
检查helm是否安装成功`helm list`

### 安装k8s1.16脚本的安装
解压包，然后执行脚本install-k8s.sh
`tar -xzvf k8s116.tar.gz`
`./install-k8s.sh` 



#### 参考文档
[kubeadm安装](https://kubernetes.io/zh/docs/)
[kubeadm创建集群](https://kubernetes.io/zh/docs/setup/independent/create-cluster-kubeadm/)