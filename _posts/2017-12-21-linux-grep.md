---
layout:     post
title:      "Linux命令每日学之grep"
subtitle:   ""
date:       2017-12-21 12:00:00
author:     "Tango"
header-img: "img/in-post/post-2017-10-11/post-bg-universe.jpg"
catalog: true
tags:   
    - Linux
    - grep
    - Linux命令
---

grep (global search regular expression(RE) and print out the line，全面搜索正则表达式并把行打印出来),倾向于理解为抓取、获取。grep命令是强大的正则表达式文本搜索命令,其常常配合文本相关命令使用。


### 命令格式

`
grep [参数] [正则表达式] [文件名]                       
`
### 命令功能
根据参数查找匹配模式的文本行。

### 命令参数

`-v: 反向查找。查找除了匹配的行以外的行，即是查找不匹配的行。               ` 
 
`-E: 匹配正则表达式。               ` 

`-c: 查找匹配字符串的行数`  
  
`-o 只输出匹配的部分                  `   
 
`-i 忽略大小写匹配字符串`

`-r 递归查找文件目录`

`-n 在显示符合范本样式的那一列之前，标示出该列的编号。`

### 命令实例

> 用于测试的文件： test.txt  
> 
> Hello World!  
> 
>I am grep!  
>
>I am 20 years old.  
>
>my phone is 122222222.



#### 1. -v 查找不包含数字的行

```
# tango @ TangodeMacBook-Pro in ~/Desktop on git:master x [20:40:54] 
$ cat test | grep -vE "[0-9]+"
Hello World!
I am grep!


```
#### 2.-E 查找包含数字的行。
```
# tango @ TangodeMacBook-Pro in ~/Desktop on git:master x [20:41:04] 
$ cat test | grep -E "[0-9]+"
I am 20 years old.
my phone is 122222222.
```
#### 3.-o 只输出匹配的部分

```
# tango @ TangodeMacBook-Pro in ~/Desktop on git:master x [20:42:08] 
$ cat test | grep -oE "[0-9]+"
20
122222222

```  

#### 4.-c 查找匹配字符串的行数
```
# tango @ TangodeMacBook-Pro in ~/Desktop on git:master x [20:45:17] C:1
$ cat test | grep -cE "[0-9]+"
2
```

#### 5.-i 忽略大小写匹配字符串

```

# tango @ TangodeMacBook-Pro in ~/Desktop on git:master x [20:47:26] C:2
$ grep "HELLO" test    

# tango @ TangodeMacBook-Pro in ~/Desktop on git:master x [20:47:31] C:1
$ grep -i "HELLO" test
Hello World!

```

#### 6.-r 递归查找文件目录

```

```

#### 7. -n 在显示符合范本样式的那一列之前，标示出该列的编号。

```
# tango @ TangodeMacBook-Pro in ~/Desktop on git:master x [20:50:33] C:130
$ cat test |grep -nE "[0-9]+"
3:I am 20 years old.
4:my phone is 122222222.

```