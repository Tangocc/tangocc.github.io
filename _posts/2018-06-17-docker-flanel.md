---
layout:     post
title:      "Docker容器之Flanel原理"
subtitle:   ""
date:       2018-03-20 12:00:00
author:     "Tango"
header-img: "img/post-bg-universe.jpg"
catalog: true
tags:   
    - Docker
    - 容器
---

根据官网的描述，flannel是一个专为kubernetes定制的三层网络解决方案，主要用于解决容器的跨主机通信问题。

## 概况


首先，flannel利用Kubernetes API或者etcd用于存储整个集群的网络配置，其中最主要的内容为设置集群的网络地址空间。例如，设定整个集群内所有容器的IP都取自网段“10.1.0.0/16”。

接着，flannel在每个主机中运行flanneld作为agent，它会为所在主机从集群的网络地址空间中，获取一个小的网段subnet，本主机内所有容器的IP地址都将从中分配。

然后，flanneld再将本主机获取的subnet以及用于主机间通信的Public IP，同样通过kubernetes API或者etcd存储起来。

最后，flannel利用各种backend mechanism，例如udp，vxlan等等，跨主机转发容器间的网络流量，完成容器间的跨主机通信。

 
## 容器间的跨主机通信如何运行

![](/img/in-post/post-flanel.png)

如上图所示，
集群范围内的网络地址空间为10.1.0.0/16，Machine A获取的subnet为10.1.15.0/24，且其中的两个容器IP分别为10.1.15.2/24和10.1.15.3/24，两者都在10.1.15.0/24这一子网范围内，对于下方的Machine B同理。

**如果上方Machine A中IP地址为10.1.15.2/24的容器要与下方Machine B中IP地址为10.1.16.2/24的容器进行通信，封包是如何进行转发的??**

从上文可知，每个主机的flanneld会将自己与所获取subnet的关联信息存入etcd中，例如，subnet 10.1.15.0/24所在主机可通过IP 192.168.0.100访问，subnet 10.1.16.0/24可通过IP 192.168.0.200访问。反之，每台主机上的flanneld通过监听etcd，也能够知道其他的subnet与哪些主机相关联。

如上图，Machine A上的flanneld通过监听etcd已经知道subnet 10.1.16.0/24所在的主机可以通过Public 192.168.0.200访问，而且熟悉docker桥接模式的同学肯定知道，目的地址为10.1.16.2/24的封包一旦到达Machine B，就能通过cni0网桥转发到相应的pod，从而达到跨宿主机通信的目的。

**因此，flanneld只要想办法将封包从Machine A转发到Machine B就OK了。**

而上文中的backend就是用于完成这一任务。不过，达到这个目的的方法是多种多样的，所以我们也就有了很多种backend。

在这里我们举例介绍的是最简单的一种方式hostgw : 因为Machine A和Machine B处于同一个子网内，它们原本就能直接互相访问。因此最简单的方法是：在Machine A中的容器要访问Machine B的容器时，我们可以将Machine B看成是网关，当有封包的目的地址在subnet 10.1.16.0/24范围内时，就将其直接转发至B即可。

而这通过图中那条红色标记的路由就能完成，对于Machine B同理可得。由此，在满足仍有subnet可以分配的条件下，我们可以将上述方法扩展到任意数目位于同一子网内的主机。而任意主机如果想要访问主机X中subnet为S的容器，只要在本主机上添加一条目的地址为R，网关为X的路由即可。

