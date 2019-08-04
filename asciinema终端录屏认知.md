---
title: asciinema终端录屏认知
date: 2019-07-31 21:46:33
tags: Linux tools
top: 100
---
# 1. asciinema 终端录屏
asciinema是一个在终端下非常棒的录制分享软件，基于文本的录屏工具，对终端输入输出进行捕捉， 然后以文本的形式来记录和回放！对多种系统都支持。
<!--more-->
## 2. asciinema安装
各种安装方法如连接，以下按centos为例进行：
https://asciinema.org/docs/installation
`yum install -y epel-release `
`yum install -y asciinema`
检查是否成功，看asciinema版本
`asciinema --version`
### 3. 指令及常见用法
```
[root@i-cvswezkk ~]# asciinema --help
usage: asciinema [-h] [--version] {rec,play,upload,auth} ...

Record and share your terminal sessions, the right way.

positional arguments:
  {rec,play,upload,auth}
    rec         Record terminal session                                 # 记录终端会话
    play        Replay terminal session                                 # 播放重播终端会话
    upload      Upload locally saved terminal session to asciinema.org  #上传本地保存的终端会话到asciinema.org
    auth                Manage recordings on asciinema.org account      # 管理asciinema.org帐户上的记录

optional arguments:
  -h, --help            show this help message and exit
  --version             show program's version number and exit
```
### 3.1 各个指令具体含义：
.cast和.json文件是一样的，注册asciinema就需要邮箱即可，然后邮箱认证即可，可能收到邮箱的时间长点。
记录终端并将其上传到asciinema.org
`asciinema rec`
将终端记录到本地文件
`asciinema rec demo.cast`
记录终端并将其上传到asciinema.org，指定标题："my aslog1"
`asciinema rec -t "my aslog1"`
将终端记录到本地文件，将空闲时间限制到最大2.5秒
`asciinema rec -i 2.5 demo.cast`
从本地文件重放终端记录
`asciinema play demo.cast`
asciinema还提供了一个可以管理asciinema个人账户所拥有的会话文件的功能, 命令为
`asciinema auth`
执行命令"asciinema auth"命令后, 会返回一个网络地址, 点击这个地址就会打开asciinema个人账号注册界面
本地修改记录文件
`vi demo.cast`
本地修改记录文件再重新上传至asciinema.org，
```
# asciinema upload /root/260132.json
View the recording at:

    https://asciinema.org/a/M1K9rrUHl5D0q3TOVDhtRcJIn
```
## 4.浏览器访问效果
双击对应的一个录屏，有share  download settings，且settings可以设置参数
![asciinema-demo效果图](http://ww1.sinaimg.cn/large/006bbiLEgy1g5jeey8ujlj30xe0m83yk.jpg "asciinema-demo效果图")