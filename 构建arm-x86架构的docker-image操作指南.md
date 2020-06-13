---
title: 构建arm/x86架构的docker image操作指南
date: 2020-06-13 20:41:08
tags: 
  - docker
  - builder
  - arm

top: 100
---
# 构建arm/x86架构的docker image操作指南
由于arm环境越来越受欢迎，镜像不单单满足x86结构的docker镜像，还需要arm操作系统的镜像，以下说明在x86机器上如何build一个arm结构的镜像，使用buildx指令来同时构建arm/x86结构的镜像。
<!--more-->
## 1.	启动一台ubuntu的机器，并安装docker 19.03
在测试过程中发现 Centos7.5 有下面的问题，这里我们直接绕过
[issue](https://github.com/multiarch/qemu-user-static/issues/38)
docker安装参考[docker安装](https://mirrors.tuna.tsinghua.edu.cn/help/docker-ce/)
## 2.	运行下列命令安装并测试qemu

* 查看机器的架构
```
uname -m
x86_64
```

* 正常测试docker启动一个arm镜像容器
```
docker run --rm -t arm64v8/ubuntu uname -m
standard_init_linux.go:211: exec user process caused "exec format error"
```

* 添加特权模式安装qemu，且启动一个arm镜像容器
```
docker run --rm --privileged multiarch/qemu-user-static --reset -p yes
docker run --rm -t arm64v8/ubuntu uname -m
aarch64
```


## 3.	  启用docker  buildx 命令
docker buildx 为跨平台构建 docker 镜像所使用的命令。目前为实验特性，可以设置dokcer cli的配置，将实验特性开启。

将下面配置添加到CLI配置文件当中~/.docker/config.json
```
{
  "experimental": "enabled"
}
```


## 4.	创建新的builder实例（默认的docker实例不支持镜像导出）
`docker buildx create --name ks-all`
`docker buildx use ks-all`
`docker buildx inspect --bootstrap`

执行下面命令可以看到 builder 已经创建好，并且支持多种平台的构建。
```
docker buildx ls
NAME/NODE DRIVER/ENDPOINT             STATUS  PLATFORMS
ks-all *  docker-container
  ks-all0 unix:///var/run/docker.sock running linux/amd64, linux/arm64, linux/riscv64, linux/ppc64le, linux/s390x, linux/386, linux/arm/v7, linux/arm/v6
default   docker
  default default                     running linux/amd64, linux/386
```

## 5.	执行构建命令（以ks-installer为例）
在 ks-installer目录下执行命令可以构建 arm64与amd64的镜像，并自动推送到镜像仓库中。
`docker buildx build -f /root/ks-installer/Dockerfile --output=type=registry --platform linux/arm64  -t lilinlinlin/ks-installer:2.1.0-arm64 .`

(需要注意现在 ks-installer 的 Dockerfile中 go build 命令带有 GOOS GOARCH等，这些要删除)

构建成功之后，可以在dockerhub下图当中可以看到是支持两种arch
```
DIGEST                       OS/ARCH                          COMPRESSED SIZE
97dd2142cac6                 linux/amd64                       111.13 MB
ce366ad696cb                 linux/arm64                       111.13 MB
```

## 6.	 构建并保存为tar 文件
* 可以参考 buildx 的官方文档
https://github.com/docker/buildx#-o---outputpath-typetypekeyvalue

`docker buildx build  --output=type=docker,dest=/root/ks-installer.tar --platform 
linux/arm64 -t lilinlinlin/ks-installer:2.1.0-arm64 ./pkg/db/`

构建tar包时需要注意output的类型需要是docker，而不是tar


## 更多参考
https://github.com/docker/buildx
https://github.com/multiarch/qemu-user-static
https://community.arm.com/developer/tools-software/tools/b/tools-software-ides-blog/posts/getting-started-with-docker-for-arm-on-linux