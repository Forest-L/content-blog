---
title: centos、ubuntu和pip离线依赖包的制作方法及解压方法
date: 2019-10-10 17:19:43
tags:
---

# centos、ubuntu和pip离线

## 1、pip安装离线本地包,pip版本（19.2.3）
* 导出本地已有的依赖包,需要创建一个空的requirements.txt文件。
`pip freeze > requirements.txt`
* 下载到某个目录下（提前创建目录/packs），指定pip源。
`pip download -r requirements.txt -d /packs -i http://mirrors.aliyun.com/pypi/simple/ --trusted-host mirrors.aliyun.com`
* 安装requirements.txt依赖,可以通过pip --help获取相关指令。
```
在线安装：
pip install -r requirements.txt
离线安装：
将/packs目录下的包拷贝到离线环境的机器某目录上（/packs），
pip install --no-index --find-links="/packs" -r requirements.txt
```

## 2、centos安装离线本地包
* 制作离线包
```
新的机器上需要在/etc/yum.conf下打开缓存，keepcache=1;
新建目录存放rpm包，如mkdir -p /root/centos-repo;
安装单个工具，yum install -y iotop --downloaddir=/root/centos-repo;
创建本地源，createrepo /root/centos-repo;
制作iso包：mkisofs -r -o /root/centos-7.5-amd64.iso /root/centos-repo/
```

## 3、ubuntu安装离线本地包
* 制作离线包
```
创建存放目录：mkdir -p /home/ubuntu/packs；
安装软件包dpkg-dev:apt-get install dpkg-dev
拷贝dep包至存放目录：sudo cp -r /var/cache/apt/archives/* /home/ubuntu/packs；
进入packs目录下，生成包的依赖信息：dpkg-scanpackages packs /dev/null |gzip > packs/Packages.gz
制作iso包：mkisofs -r -o /home/ubuntu/ubuntu-16.04.5-amd64.iso /home/ubunut/packs
```

* 解压离线包,如Ubuntu16.04.5
```
创建目录：mkdir  -p /kubeinstaller/apt_repo/16.04.5/iso
挂载至/etc/fstab下在iso包目录下：sh -c "echo 'ubuntu-16.04.5-server-amd64.iso /kubeinstaller/apt_repo/16.04.5/iso iso9660 loop 0  0' >> /etc/fstab"
备份之前源：mv -f /etc/apt/sources.list /etc/apt/sources.list-bak
添加新的源：sh -c "echo 'deb [trusted=yes]  file:///kubeinstaller/apt_repo/16.04.5/iso/  /' > /etc/apt/sources.list"
挂载生效：mount -a

# clean the process using apt or dpkg
    apt_process=`ps -aux | grep -E 'apt|dpkg' | grep -v 'grep' | awk '{print $2}'`
    for process in ${apt_process}
    do
        kill -9 $process
    done
    # remove the apt lock
    sudo rm -f /var/lib/apt/lists/lock
```

#### 参考文献
pip: https://www.cnblogs.com/zengchunyun/p/9344664.html
centos: https://blog.csdn.net/hao_rh/article/details/73275071
ubuntu: https://www.cnblogs.com/gzxbkk/p/7809296.html
