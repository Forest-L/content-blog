---
title: Kubesphere基于Velero做集群的迁移
date: 2019-11-21 15:07:17
tags: 
  - k8s tools
  - etcd

top: 100
---
#Kubesphere基于Velero做集群的迁移
使用Velero 快速完成云原生应用迁移至备份集群中。<!--more-->
### 环境信息
集群A（生产）：
master：192.168.11.6、192.168.11.13、192.168.11.16
lb：192.168.11.252
node：192.168.11.22
nfs：192.168.11.14
集群B（容灾）：
master：192.168.11.8、192.168.11.10、192.168.11.17
lb：192.168.11.253
node：192.168.11.18
nfs：192.168.11.14
### Velero安装部署
集群A和集群B都需要安装velero，安装过程参考官方文档[velero安装](https://velero.io/docs/v1.2.0/contributions/minio/),大致流程为：
```
1、安装velero客户端安装包，A和B集群都需要。
1.1、wget https://github.com/vmware-tanzu/velero/releases/download/v1.0.0/velero-v1.0.0-linux-amd64.tar.gz
1.2、解压安装包，且将velero拷贝至/usr/local/bin目录下。
2、安装velero服务端，A和B集群都需要。
2.1、本地创建密钥文件，vi credentials-velero
[default]
aws_access_key_id = minio
aws_secret_access_key = minio123
2.2、集群B环境，运下载和运行00-minio-deployment.yaml文件，且需要将其中的ClusterIP改成NodePort，添加nodePort: 31860，集群A环境不需要执行这步。
kubectl apply -f examples/minio/00-minio-deployment.yaml
2.3、集群B环境，启动服务端,需要在密钥文件同级目录下执行：
velero install \
    --provider aws \
    --bucket velero \
    --secret-file ./credentials-velero \
    --use-volume-snapshots=false \
    --backup-location-config region=minio,s3ForcePathStyle="true",s3Url=http://minio.velero.svc:9000 \
	--plugins velero/velero-plugin-for-aws:v1.0.0
	
2.4、集群A环境，启动服务端，注意：需要在集群A中获取velero的外部curl：
2.4.1、集群A中，kubectl get svc -n velero,获取9000映射的端口，如：9000:31860，根据情况而定
2.4.2、启动指令：
velero install \
    --provider aws \
    --bucket velero \
    --secret-file ./credentials-velero \
    --use-volume-snapshots=false \
    --backup-location-config region=minio,s3ForcePathStyle="true",s3Url=http://192.168.11.8:31860 \
	--plugins velero/velero-plugin-for-aws:v1.0.0

```
### 集群A数据的备份及集群B恢复
具体备份指令，定时备份，参考官方文档[备份指令](https://velero.io/docs/v1.2.0/contributions/minio/)
在集群A中模拟了带有持久化的有状态和无状态的应用，备份维度以namespace为基准，为test。
备份指令为：velero backup create test-backup --include-namespaces test
集群A所有的机器关机且在机器B恢复验证：
```
[root@master1 velero-v1.2.0-linux-amd64]# velero restore create --from-backup test-backup
Restore request "test-backup-20191121141336" submitted successfully.
Run `velero restore describe test-backup-20191121141336` or `velero restore logs test-backup-20191121141336` for more details.
[root@master1 velero-v1.2.0-linux-amd64]# kubectl get ns
NAME                           STATUS   AGE
default                        Active   43h
demo                           Active   42h
kube-node-lease                Active   43h
kube-public                    Active   43h
kube-system                    Active   43h
kubesphere-controls-system     Active   43h
kubesphere-monitoring-system   Active   43h
kubesphere-system              Active   43h
openpitrix-system              Active   23h
test                           Active   12s
velero                         Active   24m
[root@master1 velero-v1.2.0-linux-amd64]# kubectl get pod -n test
NAME                             READY   STATUS    RESTARTS   AGE
mysql-v1-0                       1/1     Running   0          22s
tomcattest-v1-554c8875cd-26fz4   1/1     Running   0          22s
tomcattest-v1-554c8875cd-cmm2z   1/1     Running   0          22s
tomcattest-v1-554c8875cd-dc7mr   1/1     Running   0          22s
tomcattest-v1-554c8875cd-fcgn4   1/1     Running   0          22s
tomcattest-v1-554c8875cd-wqb4t   1/1     Running   0          22s
wordpress-v1-65d58448f8-g5bh8    1/1     Running   0          22s
```
