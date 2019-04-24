---
layout:     post
title:      "Device Map文件系统修改docker容器磁盘大小"
subtitle:   ""
date:       2019-03-19 12:00:00
author:     "Tango"
header-img: "img/post-bg-universe.jpg"
catalog: true
tags:   
    - docker
---
 最近在运维docker集群时，多次出现容器磁盘空间(/home)目录使用率过高问题，定位过程中发现/home目录默认限制是10G，显然无法满足线上的环境要求，因此需要在不停服条件下对容器/home磁盘空间进行扩容。

## 背景知识

要真正理解我们要做的事情，首先来了解 Device Mapper 插件的工作原理。

它是基于 Device Mapper 的“精简目标”的特性。它实际上是目标块设备的快照，之所以被称为“精简”是因为它允许精简配置。精简配置意味着你有一个（希望很大）可用存储块的池，接着你可以从那个池中创建任意大小的块设备（虚拟磁盘，如有需要）；在你实际读写后，这些存储块将会被标记为已使用（或者从池中拿走）。

这意味着你是可以超额使用这个池，比如在一个 100GB 的池里面创建几千个 10GB 的卷，甚至可能是一个 100TB 的卷在一个 1GB 的池里面。只要你的**实际读写的块的容量不大于池**的大小，你怎么做都 OK 。

除此之外，精简目标的方式是可以做快照的。这表明无论何时，你都可以创建一个存在的卷的浅拷贝。在用户看来，就像你有两个一样的卷，它们可以独立地各自修改。即使你做了一个完整的拷贝，除了在时间上它是瞬间发生的（即使是很大的卷），它们不会两次重复使用存储。额外的存储只有当其中任何一卷有变化的时候才会发生，然后精简目标会从池里面分配一个存储快。

从本质上来看，“精简目标”实际上使用了两个存储设备：一个（大）的是存储块池自己，还有一个小的存储了一些元数据。这些元数据中包括了卷、快照、以及每个卷的块或者快照同存储池中块的映射信息。

## 具体操作

### 1.获取容器完整CONTAINER ID 

> docker inspect 84ff4017f426   
> 
> [
    {
        "Id": "84ff4017f426753d8fabe1f9852923019d5d250f6b17fe6e09cebd9a94d6244b",
        "Created": "2019-03-13T03:19:53.005659631Z",
        "Path": "/bin/bash",
        "Args": [],
        "State": {
            "Status": "running",
            "Running": true,
            "Paused": false,
            "Restarting": false,
            "OOMKilled": false,
            "Dead": false,
            "Pid": 18204,
            "ExitCode": 0,
            "Error": "",
            "StartedAt": "2019-03-13T03:19:53.634989982Z",
            "FinishedAt": "0001-01-01T00:00:00Z"
        },
        "Image": "sha256:f4b98b05d54e1bce0f07fa4517c83e48c33e0e7085fe6322a4c5c7cda9da83e7",
        "ResolvConfPath": "/mnt/data/docker/containers/84ff4017f426753d8fabe1f9852923019d5d250f6b17fe6e09cebd9a94d6244b/resolv.conf",
        "HostnamePath": "/mnt/data/docker/containers/84ff4017f426753d8fabe1f9852923019d5d250f6b17fe6e09cebd9a94d6244b/hostname",
        "HostsPath": "/mnt/data/docker/containers/84ff4017f426753d8fabe1f9852923019d5d250f6b17fe6e09cebd9a94d6244b/hosts",
        "LogPath": "/mnt/data/docker/containers/84ff4017f426753d8fabe1f9852923019d5d250f6b17fe6e09cebd9a94d6244b/84ff4017f426753d8fabe1f9852923019d5d250f6b17fe6e09cebd9a94d6244b-json.log",
        "Name": "/kickass_almeida",
        "RestartCount": 0,
        "Driver": "devicemapper",
        "MountLabel": "",
        "ProcessLabel": "",
        "AppArmorProfile": "",
        "ExecIDs": null,
        "HostConfig": {
            "Binds": null,
            "ContainerIDFile": "",
            "LogConfig": {
                "Type": "json-file",
                "Config": {}
            },
            "NetworkMode": "default",
            "PortBindings": {
                "8088/tcp": [
                    {
                        "HostIp": "",
                        "HostPort": "8088"
                    }
                ]
            },
            "RestartPolicy": {
                "Name": "no",
                "MaximumRetryCount": 0
            },
            "VolumeDriver": "",
            "VolumesFrom": null,
            "CapAdd": null,
            "CapDrop": null,
            "Dns": [],
            "DnsOptions": [],
            "DnsSearch": [],
            "ExtraHosts": null,
            "GroupAdd": null,
            "IpcMode": "",
            "Links": null,
            "OomScoreAdj": 0,
            "PidMode": "",
            "Privileged": false,
            "PublishAllPorts": false,
            "ReadonlyRootfs": false,
            "SecurityOpt": null,
            "UTSMode": "",
            "ShmSize": 67108864,
            "ConsoleSize": [
                0,
                0
            ],
            "Isolation": "",
            "CpuShares": 0,
            "CgroupParent": "",
            "BlkioWeight": 0,
            "BlkioWeightDevice": null,
            "BlkioDeviceReadBps": null,
            "BlkioDeviceWriteBps": null,
            "BlkioDeviceReadIOps": null,
            "BlkioDeviceWriteIOps": null,
            "CpuPeriod": 0,
            "CpuQuota": 0,
            "CpusetCpus": "",
            "CpusetMems": "",
            "Devices": [],
            "KernelMemory": 0,
            "Memory": 0,
            "MemoryReservation": 0,
            "MemorySwap": 0,
            "MemorySwappiness": -1,
            "OomKillDisable": false,
            "PidsLimit": 0,
            "Ulimits": null
        },
        "GraphDriver": {
            "Name": "devicemapper",
            "Data": {
                "DeviceId": "763",
                "DeviceName": "docker-252:16-126353413-cc6af4274aa46e529820615e05dd20b04cc6e9f2bc7f892202e9329c0b1b969f",
                "DeviceSize": "10737418240"
            }
        },
        "Mounts": [],
        "Config": {
            "Hostname": "84ff4017f426",
            "Domainname": "",
            "User": "root",
            "AttachStdin": true,
            "AttachStdout": true,
            "AttachStderr": true,
            "ExposedPorts": {
                "22/tcp": {},
                "8080/tcp": {},
                "8088/tcp": {}
            },
            "Tty": true,
            "OpenStdin": true,
            "StdinOnce": true,
            "Env": [
                "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
                "DOCKER_FIX="
            ],
            "Cmd": [
                "/bin/bash"
            ],
            "Image": "10.14.12.104:5000/edu-base-image:v3",
            "Volumes": {},
            "WorkingDir": "",
            "Entrypoint": null,
            "OnBuild": null,
            "Labels": {},
            "StopSignal": "SIGTERM"
        },
        "NetworkSettings": {
            "Bridge": "",
            "SandboxID": "79d13981bd7b067e24e3c561360b64af8fb5252c913d7ed5500df4592e03bae6",
            "HairpinMode": false,
            "LinkLocalIPv6Address": "",
            "LinkLocalIPv6PrefixLen": 0,
            "Ports": {
                "22/tcp": null,
                "8080/tcp": null,
                "8088/tcp": [
                    {
                        "HostIp": "0.0.0.0",
                        "HostPort": "8088"
                    }
                ]
            },
            "SandboxKey": "/var/run/docker/netns/79d13981bd7b",
            "SecondaryIPAddresses": null,
            "SecondaryIPv6Addresses": null,
            "EndpointID": "9a1fd28b93e2ae06548f124a2c596b835045abc5fa0e23d40242c455527bdc7d",
            "Gateway": "172.17.6.1",
            "GlobalIPv6Address": "",
            "GlobalIPv6PrefixLen": 0,
            "IPAddress": "172.17.6.4",
            "IPPrefixLen": 24,
            "IPv6Gateway": "",
            "MacAddress": "02:42:ac:11:06:04",
            "Networks": {
                "bridge": {
                    "IPAMConfig": null,
                    "Links": null,
                    "Aliases": null,
                    "NetworkID": "28a3ca6566223478fe532f9a5b2b582acc87980c888e4f9575b149912c1debf0",
                    "EndpointID": "9a1fd28b93e2ae06548f124a2c596b835045abc5fa0e23d40242c455527bdc7d",
                    "Gateway": "172.17.6.1",
                    "IPAddress": "172.17.6.4",
                    "IPPrefixLen": 24,
                    "IPv6Gateway": "",
                    "GlobalIPv6Address": "",
                    "GlobalIPv6PrefixLen": 0,
                    "MacAddress": "02:42:ac:11:06:04"
                }
            }
        }
    }
]

从上面输出可以看出容器完整ID为`84ff4017f426753d8fabe1f9852923019d5d250f6b17fe6e09cebd9a94d6244b `
device ID 为`docker-252:16-126353413-cc6af4274aa46e529820615e05dd20b04cc6e9f2bc7f892202e9329c0b1b969f`

### 2.查看/dev/mapper

那里应该有一个对应容器文件系统的符号链接，以 docker-X:Y-Z- 开头：

>ls -l /dev/mapper/docker-252:16-126353413-cc6af4274aa46e529820615e05dd20b04cc6e9f2bc7f892202e9329c0b1b969f
>
>lrwxrwxrwx 1 root root 7 Mar 19 20:34 docker-252:16-126353413-cc6af4274aa46e529820615e05dd20b04cc6e9f2bc7f892202e9329c0b1b969f -> ../dm-5

### 3.查看当前卷信息

> dmsetup table docker-252:16-126353413-cc6af4274aa46e529820615e05dd20b04cc6e9f2bc7f892202e9329c0b1b969f
> 
> 0 88080384 thin 253:1 763
> 
输出参数中 `88080384 = 42*1024*1024*1024/512 `为设备的大小，表示有多少个 512－bytes 的扇区. 这个值略高于 10GB 的大小。

后期修改容器空间大小即是修改该值

###  4.修改卷大小

我们会改变**第二个数字**，要非常小心保持其他的值不变。你的卷可能不是 7 ，所以要使用正确的值!
>cat 0 28080389 thin 253:1 763 | dmsetup load docker-252:16-126353413-cc6af4274aa46e529820615e05dd20b04cc6e9f2bc7f892202e9329c0b1b969f


### 5.激活新表
> dmsetup resume docker-252:16-126353413-cc6af4274aa46e529820615e05dd20b04cc6e9f2bc7f892202e9329c0b1b969f
> 

### 6.登录到容器内查看

登录到容器内查看容器大小即修改为我们设置的大小。

## 完整脚本





> 参考文章 https://blog.csdn.net/xiangxizhishi/article/details/75208714