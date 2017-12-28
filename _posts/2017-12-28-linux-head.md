---
layout:     post
title:      "Linux命令每日学之tail"
subtitle:   ""
date:       2017-12-28 12:00:00
author:     "Tango"
header-img: "img/in-post/post-2017-10-11/post-bg-universe.jpg"
catalog: true
tags:   
    - Linux
    - tail
    - Linux命令
---

**本文介绍几个文本操作指令**


### 1. tail：从文本末尾查看文本

#### 常用例子

`tail -n 6  file :`显示后6行  
`tail -f file :` 动态显示(不断刷新,即是文本增加时刷新显示)
`tail -n 6 -f file:`查看日志常用组合

### 2.head：从文本开头查看文本


#### 常用例子

`head -c 6 file：`显示6个字节数
`head -n 6 file：`显示6个行数


### 3.more：显示文本

#### 常用例子

`more -n file` 从n行开始显示


### 4.Cat：显示文本

`cat -n file` 显示行号







 
