---
layout:     post
title:      "Docker容器之网络模式"
subtitle:   ""
date:       2018-03-20 12:00:00
author:     "Tango"
header-img: "img/post-bg-universe.jpg"
catalog: true
tags:   
    - Docker
    - 容器
    - 网络模式
---

## 四种网络模式简介


docker容器提供四种网络模式，可以在启动容器时通过`--net=[模式名]`进行指定：

- host模式

host模式即在创建容器时不进行network namespace隔离，容器与宿主机保持在同一个network namespace中，宿主机IP地址即是容器IP地址，但是其文件系统、进程树等namespace仍然与宿主机环境保持隔离。


比如我们在机器10.10.26.36机器上以`docker run --net=host -it nginx /bin/bash`命令启动一个nginx服务，进入到容器中通过`hostname -i `查看容器IP地址

```
## 宿主机
[root@bjyz-bcc-arch /bin]# hostname -i
10.10.26.36

##容器内
[root@bjyz-bcc-arch /bin]# docker exec -it a733519f6888 /bin/bash
root@bjyz-bcc-arch:/# hostname -i
10.14.29.151

```


- bridge模式

桥接模式是docker容器默认的网络模式，这个模式下，docker容器拥有自己的network namespace,其网络与宿主机保持隔离，其具有不同的ip地址和路由信息。
其原理是：

docker damon在启动时会创建一个虚拟网桥docker0，类似于二层交换机。容器启动时会创建veth pair虚拟设备，其中 eth0 挂载在容器内，eth1则挂接在docker0网桥中，这样容器就加入一个二层网络，docker0成为网络的虚拟网关。接下来docker会在私有网段中选择一个与宿主机网络没有冲突的IP地址段，如172.17.0.0/16中选择一个未被占用的ip地址作为容器的IP。


同样的，我们在机器`10.10.26.36`机器上以`docker run --net=bridge -it nginx /bin/bash`命令启动一个nginx服务，进入到容器中通过`hostname -i `查看容器IP地址

```
## 宿主机
[root@bjyz-bcc-arch /bin]# hostname -i
10.10.26.36

[root@bjyz-bcc-arch /bin]# docker run --net=bridge -it nginx /bin/bash
f11484a2147970589bb6ffcfecd3886134768dfe89c1945e5f69e7a57f4b5d6b

##容器内
root@f11484a21479:/# hostname -i
172.17.0.3

```
容器分配的地址为`172.17.0.3`

- container模式

container模式则是指新创建的容器与其他已创建的容器共享同一个network namespace。

同样的，我们在机器`10.10.26.36`机器上以上一个容器f11484a2147为共享网络容器创建新的容器，`docker run --net=container:f11484a2147 -it nginx /bin/bash`命令启动一个nginx服务，进入到容器中通过`hostname -i `查看容器IP地址

```
## 宿主机
[root@bjyz-bcc-arch /bin]# hostname -i
10.10.26.36

[root@bjyz-bcc-arch /bin]# docker run --net=container:f11484a2147 -it nginx /bin/bash
66d2ffad9509c84935c7f4b48234d45ef658bbb8f5e5b91027550ba7c48a043a

##容器内
root@f11484a21479:/# hostname -i
172.17.0.3

```
我们发现进入容器后新创建的容器主机名为`f11484a21479 `,ip地址为`172.17.0.3`

- none模式

在这种模式下，docker容器拥有独立的network namespace，但是并不为docker容器进行任何网络配置。也就是说，这个docker容器没有网卡、ip、路由等信息。需要我们手动为docker容器添加网卡、配置IP等。

同样的，我们在机器`10.10.26.36`机器上以`docker run --net=none -it nginx /bin/bash`命令启动一个nginx服务，进入到容器中通过`hostname -i `查看容器IP地址

```
## 宿主机
[root@bjyz-bcc-arch /bin]# hostname -i
10.10.26.36

[root@bjyz-bcc-arch /bin]# docker run --net= none -it nginx /bin/bash
e59639daa28e7359277ca2addae4d31cdb43681b5050454a52a99cfcf9d66c09

##容器内
root@e59639daa28e:/# hostname -i
hostname: Temporary failure in name resolution

```
我们发现进入容器后新创建的容器无法获得容器IP地址。



另外，我们可以通过`docker inspect [容器ID]`方式查看已启动容器的网络模式呢？

```
[root@bjyz-bcc-arch /bin]# docker inspect e59639daa28e7359277ca2addae4d31cdb43681b5050454a52a99cfcf9d66c09

[
    {
        "Id": "e59639daa28e7359277ca2addae4d31cdb43681b5050454a52a99cfcf9d66c09",
        "Created": "2019-07-11T03:19:30.597584671Z",
        "Path": "/bin/bash",
        "Args": [],
        "State": {
            "Status": "running",
            "Running": true,
            "Paused": false,
            "Restarting": false,
            "OOMKilled": false,
            "Dead": false,
            "Pid": 16864,
            "ExitCode": 0,
            "Error": "",
            "StartedAt": "2019-07-11T03:19:30.801413501Z",
            "FinishedAt": "0001-01-01T00:00:00Z"
        },
               "Networks": {
                "none": {
                    "IPAMConfig": null,
                    "Links": null,
                    "Aliases": null,
                    "NetworkID": "591a23ccbdf4b633cdd0e127dbcaafa11a6772038e18b30140271ac62e5beb2d",
                    "EndpointID": "4c94b85695799946c244d36a7c2b4ca154bbeaa373b0237edfad3ab68c7afb17",
                    "Gateway": "",
                    "IPAddress": "",
                    "IPPrefixLen": 0,
                    "IPv6Gateway": "",
                    "GlobalIPv6Address": "",
                    "GlobalIPv6PrefixLen": 0,
                    "MacAddress": ""
                }
            }
        }
    }
]

```
 
## bridge模式原理

无论host模式还是bridge模式，再或者container模式其底层都是通过IPTABLE规则进行路由转发实现，
关于iptable原理不再赘述，下面以bridge模式介绍网络实现原理。

![](/img/in-post/docker-bridge.png)

桥接模式结构图
1.docker damon启动时会创建docker0虚拟网桥，并在私有网段中选取一段网段(如，172.17.0.1/16)构成一段子网,并通过veth pair链接到物理网考echo0,此时docker0网桥构成二层虚拟网络

2.docker容器启动时，会在docker0子网中选取一个未使用的IP地址作为container的IP地址，并同时通过veth pair链接到 docker0子网中。

3.通过设置iptable规则，通过NAT技术进行路由规则配置。



下面通过容器A请求容器B的过程解释容器如何通过IPtable路由实现网络通信。


docker容器请求数据过程


iptable规则
