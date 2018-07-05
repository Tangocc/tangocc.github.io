---
layout:     post
title:      "Linux命令每日学之find"
subtitle:   ""
date:       2017-12-22 12:00:00
author:     "Tango"
header-img: "img/post-bg-universe.jpg"
catalog: true
tags:   
    - Linux
    - find
    - Linux命令
---

**find命令是在指定目录下查找文件或者子目录。区别与grep的是 grep 是在文件中查找字符。如果不指定参数，find默认查找当前目录下文件和子目录。**

### 命令格式

`
find [参数] [目录] [文件名]                       
`
### 命令功能
在指定目录查找满足条件的文件或者子目录。
### 命令参数

`-name <文件名称>: 查找名称为指定名称的文件。               ` 
 
`-iname <文件名称>: 查找名称为指定名称的文件（忽略大小写）。               ` 

`-Btime <时间(天为单位)>: 查找匹配字符串的行数`  
  
`-Bmin <时间(分钟为单位)>: 只输出匹配的部分                  `   
 
`-amin <时间(小时为单位)>: 最近一次存取(access)时间与查询时间在给定时间范围之内`

`-cmin <时间(小时为单位)>: 最近一次改变(change)时间与查询时间在给定时间范围之内`

`-delete : 删除查找到的文件或者文件目录`  

`-exec <命令> : 查找到的文件或者文件目录后执行指定的操作`

`-regex <模式串> : 查找满足指定模式串的文件或者文件夹`

### 命令实例
>测试的文件名:
>test


#### 1. -name 

```
tango @ TangodeMacBook-Pro in ~/Desktop on git:master x [19:42:23] 
$ find . -name Postman.app
./Postman.app

```
#### 2.-exec 查找指定文件并复制。
```
# tango @ TangodeMacBook-Pro in ~/Desktop/cc on git:master x [20:01:29] 
$ ls
test
# tango @ TangodeMacBook-Pro in ~/Desktop/cc on git:master x [20:04:57] C:1
$ find test -exec cp {} ./test2 \;

# tango @ TangodeMacBook-Pro in ~/Desktop/cc on git:master x [20:07:04] 
$ ls
test  test2

# tango @ TangodeMacBook-Pro in ~/Desktop/cc on git:master x [20:07:10] 
$ 

```
#### 3.-size 

```
根据文件大小进行匹配 find . -type f -size 文件大小单元

```  

#### 4.-atime 
```
搜索超过七天内被访问过的所有文件 find . -type f -atime +7
```

#### 5.-name

```
在/home目录下查找以.txt结尾的文件名 find /home -name "*.txt"

```

#### 6.-regex

```
基于正则表达式匹配文件路径 find . -regex ".*\(\.txt\|\.pdf\)$"

```
