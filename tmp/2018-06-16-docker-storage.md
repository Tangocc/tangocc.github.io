---
layout:     post
title:      "Docker容器之存储引擎"
subtitle:   ""
date:       2018-06-20 12:00:00
author:     "Tango"
header-img: "img/post-bg-universe.jpg"
catalog: true
tags:   
    - Docker
    - 容器
    - 存储引擎
---


#Docker镜像存储结构

镜像分层


镜像分层好处


读取文件


写入文件


镜像在机器上存储所占大小


# Docker存储引擎
---

docker支持可插拔式存储引擎，目前来看，没有一种适用于各种场景的万能存储引擎，用户根据自己实际场景选择不同的存储引擎，此外，Docker只能运行一个存储引擎，所有容器的damon进程使用同样的存储引擎创建。

docker支持的存储引擎有：

- AUFS
- devicemap
- overlay
- btrfs
- VFS
- ZFS



## 存储引擎选择

没有万能的存储引擎，官方根据经验给予的各种存储引擎的特点

![](/img/in-post/docker-storage.png)
