---
title: k8s之helm部署及用法
date: 2019-07-29 20:58:28
tags: k8s tools
top: 100
---
# k8s之helm部署及用法
Helm是Kubernetes的一个包管理工具，用来简化Kubernetes应用的部署和管理。可以把Helm比作CentOS的yum工具。所以可以把该包在不同环境下部署起来,前提需要部署k8s环境。
<!--more--> 
## 1. helm部署
Helm由两部分组成，客户端helm和服务端tiller。
* tiller运行在Kubernetes集群上，管理chart安装的release
* helm是一个命令行工具，可在本地运行，一般运行在CI/CD Server上。一般我们用helm操作

### 1.1 客户端helm和服务端tiller安装，以192.168.11.20为例
下载地址：https://github.com/helm/helm/releases
这里可以下载的是helm v2.14.1，解压缩后将可执行文件helm拷贝到/usr/local/bin下。这样客户端helm就在这台机器上安装完成了。
通过`helm version`显示客户端安装好了，但是服务端没有好.
```
helm version
Client: &version.Version{SemVer:"v2.14.1", GitCommit:"5270352a09c7e8b6e8c9593002a73535276507c0", GitTreeState:"clean"}
Error: Get http://localhost:8080/api/v1/namespaces/kube-system/pods?labelSelector=app%3Dhelm%2Cname%3Dtiller: dial tcp [::1]:8080: connect: connection refused
```
为了安装服务端tiller，还需要在这台机器上配置好kubectl工具和kubeconfig文件，确保kubectl工具可以在这台机器上访问apiserver且正常使用。
`kubectl get cs`
使用helm在k8s上部署tiller：
`helm init --service-account tiller --skip-refresh`
<font color=#DC143C >说明:</font>
如果网络原因不能访问gcr.io，可以通过helm init –service-account tiller –tiller-image <your-docker-registry>/tiller:2.7.2 –skip-refresh使用私有镜像仓库中的tiller镜像。ps:lilinlinlin/tiller:2.7.2
tiller默认被部署在k8s集群中的kube-system这个namespace下。
`kubectl get pod -n kube-system -l app=helm`
再次helm version可以打印客户端和服务端的版本：
```
helm version
Client: &version.Version{SemVer:"v2.7.2", GitCommit:"8
Server: &version.Version{SemVer:"v2.7.2",
```

## 2. kubernetes RBAC配置
因为我们将tiller部署在Kubernetes 1.8上，Kubernetes APIServer开启了RBAC访问控制，所以我们需要创建tiller使用的service account: tiller并分配合适的角色给它。
这里简单起见直接分配cluster-admin这个集群内置的ClusterRole给它。创建`vi helm-rbac.yaml`文件：

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tiller
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: tiller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: tiller
    namespace: kube-system
```

`kubectl create -f helm-rbac.yaml`
## 3.添加国内helm源
`helm repo add stable http://mirror.azure.cn/kubernetes/charts/`
更新chart repo: `helm repo update`
## 4. helm的基本使用
下面我们开始尝试创建一个chart，这个chart用来部署一个简单的服务
```
helm create hello-test
Creating hello-test

tree hello-test/
hello-test/
├── charts
├── Chart.yaml
├── templates
│   ├── deployment.yaml
│   ├── _helpers.tpl
│   ├── ingress.yaml
│   ├── NOTES.txt
│   └── service.yaml
└── values.yaml
```
* charts目录中是本chart依赖的chart，当前是空的
* Chart.yaml这个yaml文件用于描述Chart的基本信息，如名称版本等
* templates是Kubernetes manifest文件模板目录，模板使用chart配置的值生成Kubernetes manifest文件。
* templates/NOTES.txt 纯文本文件，可在其中填写chart的使用说明
* value.yaml 是chart配置的默认值
在 values.yaml 中，可以看到，默认创建的是一个 Nginx 应用。为了方便外网访问测试，将 values.yaml 中 service 的属性修改为:
```
service:
  type: NodePort
  port: 30080
```
### 4.1 部署应用
`helm install ./hello-test`
但实际上可以这样部署为,**.tgz为chart包，**.yaml类似与values.yaml把变量文件定义出来。
`helm upgrade --install <name> **.tgz **.yaml --namespace <namespace-name>`
### 4.2 查看部署应用
`helm list`
### 4.3 删除部署应用
`helm delete <name>`
### 4.4 打包chart
`helm package <name>`

## 参考：
https://www.kubernetes.org.cn/3435.html