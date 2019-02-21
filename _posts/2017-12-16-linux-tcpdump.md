---
layout:     post
title:      "Linux命令每日学之Tcpdump"
subtitle:   ""
date:       2017-12-16 12:00:00
author:     "Tango"
header-img: "img/post-bg-universe.jpg"
catalog: true
tags:   
    - Linux
    - Tcpdump
    - Linux命令
---


网络数据包截获分析工具。支持针对网络层、协议、主机、网络或端口的过滤。并提供and、or、not等逻辑语句帮助去除无用的信息。

> tcpdump - dump traffic on a network


### 不指定任何参数

监听第一块网卡上经过的数据包。主机上可能有不止一块网卡，所以经常需要指定网卡。

> tcpdump

### 监听特定网卡

> tcpdump -i en0

### 监听特定主机

监听本机跟主机182.254.38.55之间往来的通信包。

备注：出、入的包都会被监听。

> tcpdump host 182.254.38.55


### 特定来源

> tcpdump src host hostname

###特定目标地址

> tcpdump dst host hostname

如果不指定src跟dst，那么来源 或者目标 是hostname的通信都会被监听

> tcpdump host hostname

###特定端口

>tcpdump port 3000

###监听TCP/UDP

服务器上不同服务分别用了TCP、UDP作为传输层，假如只想监听TCP的数据包

>tcpdump tcp

###来源主机+端口+TCP

监听来自主机123.207.116.169在端口22上的TCP数据包

>tcpdump tcp port 22 and src host 123.207.116.169

### 监听特定主机之间的通信

>tcpdump ip host 210.27.48.1 and 210.27.48.2

210.27.48.1除了和210.27.48.2之外的主机之间的通信

> tcpdump ip host 210.27.48.1 and ! 210.27.48.2

稍微详细点的例子

tcpdump tcp -i eth1 -t -s 0 -c 100 and dst port ! 22 and src net 192.168.1.0/24 -w ./target.cap

(1)tcp: ip icmp arp rarp 和 tcp、udp、icmp这些选项等都要放到第一个参数的位置，用来过滤数据报的类型 (2)-i eth1 : 只抓经过接口eth1的包 (3)-t : 不显示时间戳 (4)-s 0 : 抓取数据包时默认抓取长度为68字节。加上-S 0 后可以抓到完整的数据包 (5)-c 100 : 只抓取100个数据包 (6)dst port ! 22 : 不抓取目标端口是22的数据包 (7)src net 192.168.1.0/24 : 数据包的源网络地址为192.168.1.0/24 (8)-w ./target.cap : 保存成cap文件，方便用ethereal(即wireshark)分析


###限制抓包的数量

如下，抓到1000个包后，自动退出

>tcpdump -c 1000

### 保存到本地

tcpdump默认会将输出写到缓冲区，只有缓冲区内容达到一定的大小，或者tcpdump退出时，才会将输出写到本地磁盘

>tcpdump -n -vvv -c 1000 -w /tmp/tcpdump_save.cap

也可以加上-U强制立即写到本地磁盘（一般不建议，性能相对较差）
