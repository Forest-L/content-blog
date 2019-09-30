---
title: docker部署mysql5.7
date: 2019-08-13 21:54:47
tags: Linux tools
top: 100
---
# docker部署mysql5.7
越来越多服务容器化，下面以mysql5.7版本为例容器化部署。按三种方式：最简单、配置文件映射和数据映射来展开。
<!--more-->
## 准备条件
docker官方镜像：mysql:5.7
个人docker账号：lilinlinlin/mysql:5.7
## 1. 最简单的部署mysql
`docker run -p 3306:3306 -e MYSQL_ROOT_PASSWORD=rootroot -d lilinlinlin/mysql:5.7`
Navicat 测试连接 用户名:root 密码:rootroot 端口:3306

## 2. 数据映射到本机部署mysql
先在本机新建一个目录，`mkdir -p /var/lib/mysql`
然后run起来,-v前面是宿主机的目录，--restart always表示重启容器
`docker run -p 3306:3306 --restart always -v /var/lib/mysql:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=mysqlroot -d 镜像ID`

## 3. 配置文件映射到本机部署mysql
在本机/etc下新建vi my.cnf配置文件，如果有的话先删除原来的，再加入创建新的
```
[mysql]
#设置mysql客户端默认字符集
default-character-set=utf8
socket=/var/lib/mysql/mysql.sock

[mysqld]
#mysql5.7以后的不兼容问题处理
sql_mode=NO_ENGINE_SUBSTITUTION,STRICT_TRANS_TABLES
datadir=/var/lib/mysql
socket=/var/lib/mysql/mysql.sock
# Disabling symbolic-links is recommended to prevent assorted security risks
symbolic-links=0
# Settings user and group are ignored when systemd is used.
# If you need to run mysqld under a different user or group,
# customize your systemd unit file for mariadb according to the
# instructions in http://fedoraproject.org/wiki/Systemd
#允许最大连接数
max_connections=200
#服务端使用的字符集默认为8比特编码的latin1字符集
character-set-server=utf8
#创建新表时将使用的默认存储引擎
default-storage-engine=INNODB
lower_case_table_names=1
max_allowed_packet=16M 
#设置时区
default-time_zone='+8:00'
[mysqld_safe]
log-error=/var/log/mariadb/mariadb.log
pid-file=/var/run/mariadb/mariadb.pid

#
# include all files from the config directory
#
!includedir /etc/mysql/conf.d/
!includedir /etc/mysql/mysql.conf.d/
```
运行的指令：--privileged=true 获取临时的selinux的权限。
`docker run -p 3306:3306 --privileged=true -v /etc/my.cnf:/etc/mysql/mysql.conf.d/mysqld.cnf -e MYSQL_ROOT_PASSWORD=mysql密码 -d 镜像ID`

## 查看容器启动情况
`docker ps -a|grep mysql`

