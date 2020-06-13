---
title: tcpdump抓包实战教程
date: 2020-01-09 15:27:50
tags:
  - tcpdump
  - 网络工具
top: 100
---
# tcpdump抓包实战教程
做 web 开发，接口对接过程中，分析 http 请求报文数据包格式是否正确，定位问题，省去无用的甩锅过程，再比如抓取 tcp/udp 报文，分析 tcp 连接过程中的三次握手和四次挥手。windows使用wireshark工具，Linux使用的是tcpdump工具，也可以生成.pcap文件在wireshark图形化工具上分析。<!--more-->

### 命令行
* -i 选择网卡接口
* -n： 不解析主机名
* -nn：不解析端口
* port 80： 抓取80端口上面的数据
* tcp： 抓取tcp的包
* udp：抓取udp的包
* -w： 保存成pcap文件
* dst：目的ip
* src：源ip
* -c：<数据包数目>
* -s0表示可按包长显示完整的包

### tcp标志位
* SYN，显示为S，同步标志位，用于建立会话连接，同步序列号；
* ACK，显示为.，确认标志位，对已接收的数据包进行确认；
* FIN，显示为F，完成标志位，表示我已经没有数据要发送了，即将关闭连接；
* RESET，显示为R，重置标志位，用于连接复位、拒绝错误和非法的数据包；
* PUSH，显示为P，推送标志位，表示该数据包被对方接收后应立即交给上层应用，而不在缓冲区排队；
* URGENT，显示为U，紧急标志位，表示数据包的紧急指针域有效，用来保证连接不被阻断，并督促中间设备尽

### nginx连接问题。
* 抓取nginx(端口30880)交互的包：我们知道nginx交互其实是tcp协议，因此使用如下命令
`tcpdump -i eth0 tcp and port 30880 -n -nn -C 20 -W 50 -s 0`
* tcp建立连接：先在服务的机器上执行如下命令，接着把multinode.ks.dev.chenshaowen.com:30880的url在浏览器访问即可。以下标志位含义可以参考上面描述。

```
[root@master ~]#tcpdump -i eth0 tcp and port 30880 -n -nn -C 20 -W 50 -s 0 
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes
10:44:36.391603 IP 139.198.254.12.61476 > 192.168.12.2.30880: Flags [S], seq 3950767038, win 64240, options [mss 1360,nop,wscale 8,nop,nop,sackOK], length 0
10:44:36.391864 IP 192.168.12.2.30880 > 139.198.254.12.61476: Flags [S.], seq 1197638836, ack 3950767039, win 29200, options [mss 1460,nop,nop,sackOK,nop,wscale 7], length 0
10:44:36.447960 IP 139.198.254.12.61478 > 192.168.12.2.30880: Flags [.], ack 1, win 515, length 0
10:44:36.448242 IP 139.198.254.12.61476 > 192.168.12.2.30880: Flags [.], ack 1, win 515, length 0
10:44:36.592856 IP 192.168.12.2.30880 > 139.198.254.12.61476: Flags [P.], seq 5250:6642, ack 1511, win 252, length 1392
10:44:36.647585 IP 139.198.254.12.61476 > 192.168.12.2.30880: Flags [.], ack 6642, win 515, length 0
10:44:41.592442 IP 192.168.12.2.30880 > 139.198.254.12.61476: Flags [F.], seq 6642, ack 1511, win 252, length 0
```
#### 简单分析
* 当url一打开，tcpdump就有数据显示。看到的S标志位，建立会话连接；接着看到.标志位，对接受包进行确认；然后看到P标志位，表示数据包被对方接受上交给上层应用；最后看到F标志位，表示完成，没有数据要发送。

### 非服务端tcpdump客户端的数据
* 一般情况下，我们都是在服务端使用tcpdump工具抓包的；如果需要在非服务端使用tcpdump抓包可以通过网络流量镜像方式，使服务端的流量到目标地址上。

### 某服务不正常tcpdump测试结果
* 抓取某不正常的服务(端口30881)交互的包：我们知道nginx交互其实是tcp协议，因此使用如下命令
`tcpdump -i eth0 tcp and port 30881 -n -nn -C 20 -W 50 -s 0`
* tcp建立连接：先在服务的机器上执行如下命令，接着把multinode.ks.dev.chenshaowen.com:30881的url在浏览器访问即可。以下标志位含义可以参考上面描述。

```
[root@master ~]#tcpdump -i eth0 tcp and port 30881 -n -nn -C 20 -W 50 -s 0 
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes
10:56:00.569543 IP 139.198.254.12.61608 > 192.168.12.2.30881: Flags [S], seq 1702724216, win 64240, options [mss 1360,nop,wscale 8,nop,nop,sackOK], length 0
10:56:00.569723 IP 192.168.12.2.30881 > 139.198.254.12.61608: Flags [R.], seq 0, ack 1702724217, win 0, length 0
10:56:00.571570 IP 139.198.254.12.61609 > 192.168.12.2.30881: Flags [S], seq 97188145, win 64240, options [mss 1360,nop,wscale 8,nop,nop,sackOK], length 0
10:56:00.571637 IP 192.168.12.2.30881 > 139.198.254.12.61609: Flags [R.], seq 0, ack 97188146, win 0, length 0
```
#### 简单分析
* 当url一打开，一开始出现S标志位，表示建立会话连接；接着出现R标志位，表示重置，用于连接复位，拒绝错误和非法的数据包；最后有出现S标志位，再次建立会话连接，一直S与R交替出现，说明该服务不正常。

### 参考文章
https://dreamgoing.github.io/tcpdump%E5%AE%9E%E6%88%98.html
https://klionsec.github.io/2017/01/31/tcpdump-sniffer-pass/#menu
https://hubinwei.me/2018/07/25/tcpdump%E6%8A%93%E5%8C%85%E7%BB%83%E4%B9%A0/
