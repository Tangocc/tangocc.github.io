---
layout:     post
title:      "Linux命令每日学之Ping"
subtitle:   ""
date:       2017-12-16 12:00:00
author:     "Tango"
header-img: "img/in-post/post-2017-10-11/post-bg-universe.jpg"
catalog: true
tags:   
    - Linux
    - Ping
    - Linux命令
---

**Ping命令是常用的linux命令常用于测试两台机器之间是否网络畅通，其原理是通过ICMP报文实现。
**

> 注意:linux下的ping和windows下的ping稍有区别,linux下ping不会自动终止,需要按ctrl+c终止或者用参数-c指定要求完成的回应次数。
>   


### 命令格式

`
ping [参数] [主机名/IP]
`
### 命令功能
ping 命令每秒发送一个数据报并且为每个接收到的响应打印一行输出。ping 命令计算信号往返时间和(信息)包丢失情况的统计信息，并且在完成之后显示一个简要总结。ping 命令在程序超时或当接收到 SIGINT 信号时结束。Host 参数或者是一个有效的主机名或者是因特网地址。

### 命令参数

`-c: 在发送指定数目的包后停止`  
`-s: 指定发送的数据字节数，预设值是56，加上8字节的ICMP头，一共是64ICMP数据字节`
`-t: 设置存活数值TTL的大小`  
`-v: 详细显示指令的执行过程`
### 命令实例

#### 1.Ping通的情况
```
# tango @ TangodeMacBook-Pro in ~ on git:master x 
$ ping -c 10 10.94.214.21
PING 10.94.214.21 (10.94.214.21): 56 data bytes
64 bytes from 10.94.214.21: icmp_seq=0 ttl=54 time=2.813 ms
64 bytes from 10.94.214.21: icmp_seq=1 ttl=54 time=5.212 ms
64 bytes from 10.94.214.21: icmp_seq=2 ttl=54 time=2.259 ms
64 bytes from 10.94.214.21: icmp_seq=3 ttl=54 time=2.104 ms
64 bytes from 10.94.214.21: icmp_seq=4 ttl=54 time=2.456 ms
64 bytes from 10.94.214.21: icmp_seq=5 ttl=54 time=2.305 ms
64 bytes from 10.94.214.21: icmp_seq=6 ttl=54 time=2.126 ms
64 bytes from 10.94.214.21: icmp_seq=7 ttl=54 time=2.255 ms
64 bytes from 10.94.214.21: icmp_seq=8 ttl=54 time=2.205 ms
64 bytes from 10.94.214.21: icmp_seq=9 ttl=54 time=3.796 ms

--- 10.94.214.21 ping statistics ---
10 packets transmitted, 10 packets received, 0.0% packet loss
round-trip min/avg/max/stddev = 2.104/2.753/5.212/0.950 ms


```
#### 2.Ping不通的情况
```
# tango @ TangodeMacBook-Pro in ~ on git:master x [20:58:55] 
$ ping -c 10 10.94.214.222
PING 10.94.214.222 (10.94.214.222): 56 data bytes
Request timeout for icmp_seq 0
Request timeout for icmp_seq 1
Request timeout for icmp_seq 2
Request timeout for icmp_seq 3
Request timeout for icmp_seq 4
Request timeout for icmp_seq 5
Request timeout for icmp_seq 6
Request timeout for icmp_seq 7
Request timeout for icmp_seq 8

--- 10.94.214.222 ping statistics ---
10 packets transmitted, 0 packets received, 100.0% packet loss
```
