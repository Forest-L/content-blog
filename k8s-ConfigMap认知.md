---
title: k8s ConfigMap认知
categories: k8s tools
date: 2019-07-25 23:17:17
tags: k8s tools
top: 100
---
# k8s configMap 认知
许多应用程序会从配置文件、命令行参数或环境变量中读取配置信息，而ConfigMap作用是保存配置信息，格式为键值对，可以单独一个key/value使用，也可以多个key/value构成的文件使用。数据不包含敏感信息的字符串。ConfigMap必须在Pod引用它之前创建;Pod只能使用同一个命名空间内的ConfigMap。
<!--more--> 
## 1 常见configMap创建方式

### 1.1 从key-value字符串创建ConfigMap

`kubectl create configmap config1 --from-literal=config.test=good`

### 1.2 从目录创建,/root/configmap/test1和/root/configmap/test2 中

```
vi test1
a:a1
vi test2
b:b1
```

` kubectl create configmap special-config --from-file=/root/configmap/ `

### 1.3 通过pod形式创建,最常用的方式configtest.yaml内容如下：
```
kind: ConfigMap
apiVersion: v1
metadata:
  name: configtest
  namespace: default
data:
  test.property.1: a1
  test.property.2: b2
  test.property.file: |-
    property.1=a1
    property.2=b2
    property.3=c3
```
`kubectl create -f configtest.yaml`

## 2. 常见查看configmap方式
以上面所示的第三种方式创建的configmap为例，名为：configtest。configmap可以简称cm。
### 2.1 "-o json"格式，
`kubectl get cm configtest -o json`

### 2.2 go模板的格式
```
-o go-template='{{.data}}'格式
kubectl get configmap configtest -o go-template='{{.data}}'
```
## 3. 常见使用configmap场景
通过多种方式在Pod中使用，比如设置环境变量、设置容器命令行参数、在Volume中创建配置文件等。
<font color=#DC143C >说 明 : </font>
镜像以：buysbox镜像为例，如果能连国外的网，选择gcr.io/google_containers/busybox，不能连国外的网但能国内的外网使用：busybox:1.28.4这个镜像。
下面所有command里面都加了sleep,方便大家进容器查看配置是否起作用。
### 3.1 configmap先创建好,以下创建两种类型。
`kubectl create configmap special-config --from-literal=special.how=very --from-literal=special.type=charm`
`kubectl create configmap env-config --from-literal=log_level=INFO`
### 3.2 环境参数，注意busybox镜像选择及sleep时间,test-pod.yaml。
```
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  containers:
    - name: test-container
      image: gcr.io/google_containers/busybox
      command: [ "/bin/sh", "-c", "sleep 60 ;env" ]
      env:
        - name: SPECIAL_LEVEL_KEY
          valueFrom:
            configMapKeyRef:
              name: special-config
              key: special.how
        - name: SPECIAL_TYPE_KEY
          valueFrom:
            configMapKeyRef:
              name: special-config
              key: special.type
      envFrom:
        - configMapRef:
            name: env-config
  restartPolicy: Never
```
`kubectl create -f test-pod.yaml`
进入对应的容器里，输入ENV则会包含如下结果：
```
SPECIAL_LEVEL_KEY=very
SPECIAL_TYPE_KEY=charm
log_level=INFO
```
### 3.3 命令行参数，注意busybox镜像选择及sleep时间,dapi-test-pod.yaml。
```
apiVersion: v1
kind: Pod
metadata:
  name: dapi-test-pod
spec:
  containers:
    - name: test-container
      image: gcr.io/google_containers/busybox
      command: [ "/bin/sh", "-c", "sleep 60 ;echo $(SPECIAL_LEVEL_KEY) $(SPECIAL_TYPE_KEY)" ]
      env:
        - name: SPECIAL_LEVEL_KEY
          valueFrom:
            configMapKeyRef:
              name: special-config
              key: special.how
        - name: SPECIAL_TYPE_KEY
          valueFrom:
            configMapKeyRef:
              name: special-config
              key: special.type
  restartPolicy: Never
```
`kubectl create -f dapi-test-pod.yaml`
进入容器中，执行`echo $(SPECIAL_LEVEL_KEY) $(SPECIAL_TYPE_KEY)`指令得到结果：
very charm
### 3.4 volume将ConfigMap作为文件或目录直接挂载，注意busybox镜像选择及sleep时间,当存在同名文件时，直接覆盖掉，vol-test-pod.yaml。
```
apiVersion: v1
kind: Pod
metadata:
  name: vol-test-pod
spec:
  containers:
    - name: test-container
      image: gcr.io/google_containers/busybox
      command: [ "/bin/sh", "-c", "sleep 60 ;cat /etc/config/special.how" ]
      volumeMounts:
      - name: config-volume
        mountPath: /etc/config
  volumes:
    - name: config-volume
      configMap:
        name: special-config
  restartPolicy: Never
```

`kubectl create -f vol-test-pod.yaml`
进入容器，cat的结果：very
