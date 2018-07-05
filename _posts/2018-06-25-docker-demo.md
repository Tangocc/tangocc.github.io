---
layout:     post
title:      "Docker容器之简单应用"
subtitle:   ""
date:       2018-03-20 12:00:00
author:     "Tango"
header-img: "img/post-bg-universe.jpg"
catalog: true
tags:   
    - Docker
    - 容器
---

#遇见Docker容器
---

学校期间接触到容器的概念，但一直未曾真正实践，看到名字就感觉望而生畏，在实习阶段真正接触到Docker容器的使用，惊叹于这一神来之笔！！太有用了！！  

相信程序员经常遇到如下场景：

```
测试员小A：Monkey，程序在我机器上运行不了....
程序员:在我机器上正常啊....
.......无限撕逼中.......
.......无限找BUG中....

```
上述场景，尤其C++ Monkey应该是习以为常,由于链接库、依赖等等版本不兼容问题，经常出现`在我机器上运行好好的`的声音。咦？？有没有一种方式，我在提交代码时候，可以将我开发环境的链接库、依赖等等一并提交，保持开发和测试环境各个库的版本一致呢？

```
Docker：我可以!!！ I can!!! 私はできる!!!! 할 수 있다,...
```
当然上述应用是Docker容器应用场景的一种。

## 初识Docker

### 简介
Docker是一个开源的应用容器引擎，翻译成中文为码头工人。  

![](/img/in-post/post-2018-06-25-docker.jpg)  

如图，Docker容器形如集装箱，其实软件、环境、依赖包的集合，容器和容器之间相互隔离。

### 核心概念

- 镜像(Image)  
  镜像即是副本，是容器的静态文件存储形式，通过命令`docker run`运行镜像即得到运行的docker容器，其定义类似windows ghost。  
  通过同一个镜像可以创建完全相同的docker容器。   
  镜像存储在仓库中，代码交付(程序及运行环境)即是镜像的交付。
- 容器  
  镜像通过命令运行后即是容器实例。镜像与容器关系类比与代码与程序。代码是程序的静态形式，代码执行即获得程序，程序是代码的动态形式。
- 仓库  
  仓库即是存储镜像的地方，类似于git仓库，分为本地仓库和远程仓库，通过命令`docker pull`和`docker push`可以拉取或者推送镜像文件。 


## 初试Docker
### 目标
**基于Linux系统，Docker内编写shell脚本文件，Docker容器启动后执行该脚本文件在控制台输入`hello world!`。并将上述制作成镜像。**

### 步骤
#### 1.安装Docker环境
此处不再叙述，首先在Linux系统部署docker环境，通过`yum install docker`.
安装完成通过`docker version`验证是否安装成功。显示如下则表示成功

```
Client:
 Version:      1.10.3
 API version:  1.22
 Go version:   go1.5.3
 Git commit:   20f81dd
 Built:        Thu Mar 10 21:49:11 2016
 OS/Arch:      linux/amd64

Server:
 Version:      1.10.3
 API version:  1.22
 Go version:   go1.5.3
 Git commit:   20f81dd
 Built:        Thu Mar 10 21:49:11 2016
 OS/Arch:      linux/amd64
``` 
#### 2.拉取标准镜像
首先拉取标准镜像，本文基于Linux环境部署tomcat服务器，因此拉取centos标准镜像。命令：  

>docker pull centos

#### 3.运行该镜像获得docker容器实例

- 查看下载的docker 镜像ID

> docker images  
	REPOSITORY                      TAG            IMAGE ID            CREATED             SIZE  
	centos                        latest         49f7960eb7e4        2 weeks ago         199.7 MB

- 运行镜像获取Docker容器实例


> docker run -i -t -d 49f7960eb7e4 /bin/bash
 

运行上述命令(49f7960eb7e4是IMAGE ID，按照自己机器的ID输入，不可照抄)进入运行并进入docker实例中。

 - `-i `以交互方式运行
 - `-t` docker容器运行后进入终端命令行
 - `-d`以后台进程方式运行docker容器，这样在退出容器后其容器并没有退出。

#### 4.进入docker容器
在运行上述命令后，控制台输入一串字符串`00d0ccf4aebaff1246ced1197264bb15fb80125bcac6156c754772f61bd16d01`,容器已经启动且在后台运行，
通过命令
`docker ps`可以查看到刚刚运行的容器  
> docker ps
CONTAINER ID      IMAGE        COMMAND        CREATED              STATUS              PORTS                                        NAMES  
00d0ccf4aeba     49f7960eb7e4   "/bin/bash"    About a minute ago   Up About a minute                                              

通过命令`docker exec`进入docker容器中

> docker exec -it 00d0ccf4aeba /bin/bash

上述命令中`00d0ccf4aeba `是`docker ps`命令输出的`container ID`，运行上述命令以后则进入到熟悉的linux命令行界面，下面就可以像操作linux系统一样操作。   

#### 5.编写脚本文件
/目录下编写脚本文件,定义为`startup.sh`
> \#!/bin/bash  
echo "Hello World !"  
echo "This is first docker."

增加可执行权限  
> chmod a+x startup.sh

#### 6.保存制作的Docker镜像
至此，完成docker镜像制作的准备工作。  
通过命令`exit`退出该docker容器,回到物理机终端。  
通过`docker ps`查看刚刚的docker容器。
>注：因为在启动docker容器时候执行`docker run -itd`通过`-d`参数使容器后台运行，所以退出时其docker容器仍然在后台运行，如果运行`docker run`没有加`-d`参数，当退出时则是真正的退出，上述所做的修改将全部消失。

#### 7.制作镜像
> docker commit 00d0ccf4aeba my-image:v1
其中`00d0ccf4aeba `是container ID
上述指令语义是将容器00d0ccf4aeba重新打包为名称为my-image，标签为v1的新镜像

#### 8.运行新的镜像
再次执行`docker images`会查看到刚刚制作的镜像`my-image:v1`。
执行命令`docker run`运行新镜像

> docker run -itd 2b6238c926e8 /bin/bash /startup.sh    

命令行输出如下:  
> Hello World !  
This is first docker.

**至此，完成Docker镜像的制作。将该镜像`docker push`推到仓库中，然后再任何一台机器通过docker pull即可运行该容器。**



## 常用指令

- 运行镜像

> docker run -i [image-id ]

- 运行镜像+脚本

> docker run -i -t [image-name] [script-name]

- 后台运行镜像+脚本

> docker run -i -t [image-name] [script-name]

- 显示运行的Docker容器

> docker ps 

- 进入容器内

> docker exec -it [docker-id] /bin/bash

- 宿主机器与docker之间复制文件

> docker cp host_path containerID:container_path