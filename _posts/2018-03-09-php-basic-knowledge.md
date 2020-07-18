---
layout:     post
title:      "面试之PHP"
subtitle:   ""
date:       2017-12-16 12:00:00
author:     "Tango"
header-img: "img/post-bg-universe.jpg"
catalog: true
tags:   
    - 面试
    - php
---  

## PHP弱类型实现原理

php作为一种解释型脚本语言，简单易上手。其变量类型无需定义，运行时决定变量类型。
比如：

```
<?php
$i = 1;
$i ='1';
$i = array(1,2);
?>

```

本文一探究竟php如何实现弱类型。
答案：核心原理是C语言union联合体。

- 核心数据结构zval

```

typedef struct _zval_struct zval;  
   
struct _zval_struct {  
    /* Variable information */  
    zvalue_value value;     /* value */  
    zend_uint refcount__gc;  
    zend_uchar type;    /* active type */  
    zend_uchar is_ref__gc;  
};  
   
typedef union _zvalue_value {  
    long lval;  /* long value */  
    double dval;    /* double value */  
    struct {  
        char *val;  
        int len;  
    } str;  
    HashTable *ht;  /* hash table value */  
    zend_object_value obj;  
} zvalue_value;
```

对于Zend Engine引擎来说，所有的数据类型在内存中都是一种数据类型，即是`zval`。
其核心的字段含义：

- is_ref__gc:是否是引用类型
- refcount__gc:引用计数
- type: 数据类型
- value: 数据内容

其中对于value数据结构`_zvalue_value `是union类型结构。`type `字段标识这个变量的数据类型枚举值：

```
#define IS_UNDEF                    0
#define IS_NULL                     1
#define IS_FALSE                    2
#define IS_TRUE                     3
#define IS_LONG                     4
#define IS_DOUBLE                   5
#define IS_STRING                   6
#define IS_ARRAY                    7
#define IS_OBJECT                   8
#define IS_RESOURCE                 9
#define IS_REFERENCE                10

```
举例来说：

```
$a = '1';  
//此时zval.type = IS_STRING,那么zval.value就去取str.  
$a = array();  
//此时zval.type = IS_ARRAY,那么zval.value就去取ht.
```

## PHP数组原理
- PHP5数组原理

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200713204018272.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTMyOTE4MTg=,size_16,color_FFFFFF,t_70)

- PHP7数组原理
![在这里插入图片描述](https://img-blog.csdnimg.cn/2020071320401851.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTMyOTE4MTg=,size_16,color_FFFFFF,t_70)
