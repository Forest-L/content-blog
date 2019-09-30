---
title: ubuntu快速安装wecenter
date: 2019-08-13 23:23:54
tags: Linux tools
top: 100
---
# ubuntu快速安装wecenter
以镜像的形式安装wecenter，简化了nginx和php环境的单独安装。WeCenter（wecenter.com）是一款建立知识社区的开源程序（免费版），专注于企业和行业社区内容的整理、归类、检索和分享，是知识化问答社区的首选软件。后台使用PHP开发，MVC架构，前端使用Bootstrap框架。
<!--more-->
## 准备
mysql需要搭建，参考：
https://lilinlinlin.github.io/2019/08/13/docker%E9%83%A8%E7%BD%B2mysql5-7/#more
镜像：wecenter/wecenter:3.3.2

## wecenter安装
`docker run --name wecenter1 -p 8081:80  -d wecenter/wecenter:3.3.2`
通过外网ip加端口访问,链接为：
`eip:8081/install`

## 安装成功，浏览器打开之后
![](http://ww1.sinaimg.cn/large/006bbiLEgy1g5ygifdatkj30xv0q8abj.jpg)
* 点击下一步，进入配置系统，正确填入数据库的主机、账号、密码和数据库的名称（默认wecenter），点开始安装。
* 上步成功之后，管理员配置，用户名admin，密码自己定义。

## 参考文档：
http://help.websoft9.com/lamp-guide/installation/wecenter/install.html
