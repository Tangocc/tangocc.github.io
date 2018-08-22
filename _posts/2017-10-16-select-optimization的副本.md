---
layout:     post
title:      "MySql系列之Innodb存储引擎"
subtitle:   ""
date:       2017-10-16 12:00:00
author:     "Tango"
header-img: "img/post-bg-universe.jpg"
catalog: true
tags:   
    - MySql数据库 
    - innodb
    - 存储引擎
---


插件化存储引擎是MySQL特点，用户可以根据自己的需求使用不同的存储引擎，甚至通过抽象的API接口实现自己的存储引擎。innodb存储引擎支持行级锁以及事务特性，也是多种场合使用较多的存储引擎，本文将介绍innodb存储引擎。

### 存储引擎简介

mysql系统来说，存储引擎是真正实现数据的存储与读取操作的对象，数据库实例通过抽象API接口与存储引擎交互，存储引擎自定义实现数据的物理、逻辑组织形式以及索引、事务等特性。
抽象存储引擎API接口是通过抽象类handler来实现，handler类提供诸如打开/关闭table、扫表、查询Key数据、写记录、删除记录等基础操作方法。每一个存储引擎通过继承handler类，实现以上提到的方法，在方法里面实现对底层存储引擎的读写接口的转调。从5.0版本开始，为了避免在初始化、事务提交、保存事务点、回滚操作时需要操作one-table实例，引入了handlerton结构让存储引擎提供了发生上面提到操作时的钩子操作。


### 存储引擎架构
![](/img/in-post/mysql/post-innodb-memeory-disk.jpeg)
![](/img/in-post/mysql/post-innodb-engine-struct.jpeg)
图片转自 [](https://blog.csdn.net/nayanminxing/article/details/51151455)

### 物理组织结构


### 逻辑组织结构

### 日志文件















