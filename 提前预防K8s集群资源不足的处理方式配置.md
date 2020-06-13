---
title: 提前预防K8s集群资源不足的处理方式配置
date: 2020-04-20 14:22:39
tags:
  - k8s tools
  - shell-operator
  - kubelet

top: 100
---
# 提前预防K8s集群资源不足的处理方式配置

在管理集群的时候我们常常会遇到资源不足的情况，在这种情况下我们要保证整个集群可用，并且尽可能减少应用的损失。根据该问题提出以下两种方案：一种为优化kubelet参数，另一种为脚本化诊断处理。<!--more-->
## 概念解释
* CPU 的使用时间是可压缩的，换句话说它本身无状态，申请资源很快，也能快速正常回收。
* 内存（memory）大小是不可压缩的，因为它是有状态的（内存里面保存的数据），申请资源很慢（需要计算和分配内存块的空间），并且回收可能失败（被占用的内存一般不可回收）。

## 优化kubelet参数
优化kubelet参数通过k8s资源表示、节点资源配置及kubelet参数设置、应用优先级和资源动态调整这几个方面来介绍。k8s资源表示为yaml文件中如何添加requests和limites参数。节点资源配置及kubelet参数设置描述为一个node上面资源配置情况，从而来优化kubelet参数。应用优先级描述为当资源不足时，优先保留那些pod不被驱逐。资源动态调整描述为运算能力的增减，如：HPA 、VPA和Cluster Auto Scaler。

### k8s资源表示
* 在k8s中，资源表示配置字段是 spec.containers[].resource.limits/request.cpu/memory。yaml格式如下：
```
spec:
  template:
    ...
    spec:
      containers:
        ...
        resources:
          limits:
            cpu: "1"
            memory: 1000Mi
          requests:
            cpu: 20m
            memory: 100Mi
```

### 资源动态调整
动态调整的思路：应用的实际流量会不断变化，因此使用率也是不断变化的，为了应对应用流量的变化，我们应用能够自动调整应用的资源。比如在线商品应用在促销的时候访问量会增加，我们应该自动增加 pod 运算能力来应对；当促销结束后，有需要自动降低 pod 的运算能力防止浪费。
运算能力的增减有两种方式：改变单个 pod 的资源，已经增减 pod 的数量。这两种方式对应了 kubernetes 的 HPA 和 VPA和Cluster Auto Scaler。
* HPA: 横向 pod 自动扩展的思路是这样的：kubernetes 会运行一个 controller，周期性地监听 pod 的资源使用情况，当高于设定的阈值时，会自动增加 pod 的数量；当低于某个阈值时，会自动减少 pod 的数量。自然，这里的阈值以及 pod 的上限和下限的数量都是需要用户配置的。
* VPA: VPA 调整的是单个 pod 的 request 值（包括 CPU 和 memory）VPA 包括三个组件：
（1）Recommander：消费 metrics server 或者其他监控组件的数据，然后计算 pod 的资源推荐值
（2）Updater：找到被 vpa 接管的 pod 中和计算出来的推荐值差距过大的，对其做 update 操作（目前是 evict，新建的 pod 在下面 admission controller 中会使用推荐的资源值作为 request）
（3）Admission Controller：新建的 pod 会经过该 Admission Controller，如果 pod 是被 vpa 接管的，会使用 recommander 计算出来的推荐值
* CLuster Auto Scaler：能够根据整个集群的资源使用情况来增减节点。Cluster Auto Scaler 就是监控这个集群因为资源不足而 pending 的 pod，根据用户配置的阈值调用公有云的接口来申请创建机器或者销毁机器。

### 节点资源配置及kubelet参数设置
节点资源的配置一般分为 2 种：
* 资源预留：为系统进程和 k8s 进程预留资源
* pod 驱逐：节点资源到达一定使用量，开始驱逐 pod

|Node Capacity
|:-------------:
|kube-reserved
|system-reserved
|eviction-threshold
|Allocatable

* Node Capacity：Node的所有硬件资源
* kube-reserved：给kube组件预留的资源：kubelet,kube-proxy以及docker等
* system-reserved：给system进程预留的资源
* eviction-threshold：kubelet eviction的阈值设定
* Allocatable：真正scheduler调度Pod时的参考值（保证Node上所有Pods的request resource不超过Allocatable）

allocatable的值即对应 describe node 时看到的allocatable容量，pod 调度的上限
```
计算公式：节点上可配置值 = 总量 - 预留值 - 驱逐阈值

Allocatable = Capacity - Reserved(kube+system) - Eviction Threshold
```
以上配置均在kubelet 中添加，涉及的参数有：
```
--kube-reserved=cpu=200m,memory=250Mi \
--system-reserved=cpu=200m,memory=250Mi \
--eviction-hard=memory.available<5%,nodefs.available<10%,imagefs.available<10% \
--eviction-soft=memory.available<10%,nodefs.available<15%,imagefs.available<15% \
--eviction-soft-grace-period=memory.available=2m,nodefs.available=2m,imagefs.available=2m \
--eviction-max-pod-grace-period=120 \
--eviction-pressure-transition-period=30s \
--eviction-minimum-reclaim=memory.available=0Mi,nodefs.available=500Mi,imagefs.available=500Mi
```
以上配置均为百分比，举例：

以2核4GB内存40GB磁盘空间的配置为例，Allocatable是1.6 CPU，3.3Gi 内存，25Gi磁盘。当pod的总内存消耗大于3.3Gi或者磁盘消耗大于25Gi时，会根据相应策略驱逐pod。
Allocatable = 4Gi - 250Mi -250Mi - 4Gi*5% = 3.3Gi

（1） 配置 k8s组件预留资源的大小，CPU、Mem
```
指定为k8s系统组件（kubelet、kube-proxy、dockerd等）预留的资源量，
如：--kube-reserved=cpu=1,memory=2Gi,ephemeral-storage=1Gi。
这里的kube-reserved只为非pod形式启动的kube组件预留资源，假如组件要是以static pod（kubeadm）形式启动的，那并不在这个kube-reserved管理并限制的cgroup中，而是在kubepod这个cgroup中。
（ephemeral storage需要kubelet开启feature-gates，预留的是临时存储空间（log，EmptyDir），生产环境建议先不使用）
ephemeral-storage是kubernetes1.8开始引入的一个资源限制的对象，kubernetes 1.10版本中kubelet默认已经打开的了,到目前1.11还是beta阶段，主要是用于对本地临时存储使用空间大小的限制，如对pod的empty dir、/var/lib/kubelet、日志、容器可读写层的使用大小的限制。
```
（2）配置 系统守护进程预留资源的大小（预留的值需要根据机器上容器的密度做一个合理的值）
```
含义：为系统守护进程(sshd, udev等)预留的资源量，
如：--system-reserved=cpu=500m,memory=1Gi,ephemeral-storage=1Gi。
注意，除了考虑为系统进程预留的量之外，还应该为kernel和用户登录会话预留一些内存。
```
（3）配置 驱逐pod的硬阈值
```
含义：设置进行pod驱逐的阈值，这个参数只支持内存和磁盘。
通过--eviction-hard标志预留一些内存后，当节点上的可用内存降至保留值以下时，
kubelet 将会对pod进行驱逐。
配置：--eviction-hard=memory.available<5%,nodefs.available<10%,imagefs.available<10%
```
（4）配置 驱逐pod的软阈值
```
--eviction-soft=memory.available<10%,nodefs.available<15%,imagefs.available<15%
```
（5）定义达到软阈值之后，持续时间超过多久才进行驱逐
```
--eviction-soft-grace-period=memory.available=2m,nodefs.available=2m,imagefs.available=2m
```
（6）驱逐pod前最大等待时间=min(pod.Spec.TerminationGracePeriodSeconds, eviction-max-pod-grace-period)，单位为秒
```
--eviction-max-pod-grace-period=120
```
（7）至少回收的资源量
```
--eviction-minimum-reclaim=memory.available=0Mi,nodefs.available=500Mi,imagefs.available=500Mi
```
（8）防止波动,kubelet 多久才上报节点的状态
```
--eviction-pressure-transition-period=30s
```

### 应用优先级
当资源不足时，配置了如上驱逐参数，pod之间的驱逐顺序是怎样的呢？以下描述设置不同优先级来确保集群中核心的组件不被驱逐还正常运行，OOM 的优先级如下,pod oom 值越低，也就越不容易被系统杀死。：
```
BestEffort Pod > Burstable Pod > 其它进程（内核init进程等） > Guaranteed Pod > kubelet/docker 等 > sshd 等进程
```
kubernetes 把 pod 分成了三个 QoS 等级，而其中和limits和requests参数有关：
* Guaranteed：oom优先级最低，可以考虑数据库应用或者一些重要的业务应用。除非 pods 使用超过了它们的 limits，或者节点的内存压力很大而且没有 QoS 更低的 pod，否则不会被杀死。
* Burstable：这种类型的 pod 可以多于自己请求的资源（上限有 limit 指定，如果 limit 没有配置，则可以使用主机的任意可用资源），但是重要性认为比较低，可以是一般性的应用或者批处理任务。
* Best Effort：oom优先级最高，集群不知道 pod 的资源请求情况，调度不考虑资源，可以运行到任意节点上（从资源角度来说），可以是一些临时性的不重要应用。pod 可以使用节点上任何可用资源，但在资源不足时也会被优先杀死。
Pod 的 requests 和 limits 是如何对应到这三个 QoS 等级上的，可以用下面一张表格概括：

|request是否配置  | limits是否配置  |  两者的关系 | Qos  | 说明  |
| :------------: | :------------: | :------------: | :------------: | :------------: |
|是|是|requests=limits|Guaranteed|所有容器的cpu和memory都必须配置相同的requests和limits  |
|是|是|request<limit|Burstable  |只要有容器配置了cpu或者memory的request和limits就行   |
|是|否||Burstable|只要有容器配置了cpu或者memory的request就行   |
|否|是||Guaranteed/Burstable|如果配置了limits，k8s会自动把对应资源的request设置和limits一样。如果所有容器所有资源都配置limits，那就是Guaranteed;如果只有部分配置了limits，就是Burstable |
|否|否||Best Effort|所有的容器都没有配置资源requests或limits   |

说明：
* request和limits相同，可以参考资源动态调整中的VPA设置合理值。
* 如果只配置了limits，没有配置request，k8s会把request值和limits值一样。
* 如果只配置了request，没有配置limits，该pod共享node上可用的资源，实际上很反对这样设置。

### 总结
动态地资源调整通过 kubelet 驱逐程序进行的，但需要和应用优先级配合使用才能达到很好的效果，否则可能驱逐集群中核心组件。

## 脚本化诊断处理
什么叫脚本化诊断处理呢？它的含义为：当集群中的某台机器资源（一般指memory）用到85%-90%时，脚本自动检查到且该节点为不可调度。缺点为：背离了资源动态调整中CLuster Auto Scaler特点。
* 集群中每台机器都可以执行kubectl指令：
如果没有设置，可将master机器上的$HOME/.kube/config文件拷贝到node机器上。
* 可以通过[shell-operator](https://github.com/Forest-L/shell-operator)自动诊断机器资源且做cordon操作处理
* 脚本中关键说明
（1）获取本地IP：ip a | grep 'state UP' -A2| grep inet | grep -v inet6 | grep -v 127 | sed 's/^[ \t]*//g' | cut -d ' ' -f2 | cut -d '/' -f1
（2）获取本地ip对应的node名：kubectl get nodes -o  wide | grep "本地ip" | awk '{print $1}'
（3）不可调度：kubectl cordon node <node名>
（4）获取总内存： free -m | awk 'NR==2{print $2}'
（5）获取使用内存： free -m | awk 'NR==2{print $3}'

## 参考文档
https://kubernetes.io/zh/docs/tasks/administer-cluster/out-of-resource/
https://cizixs.com/2018/06/25/kubernetes-resource-management/
https://segmentfault.com/a/1190000021402192
























