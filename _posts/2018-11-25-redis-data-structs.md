---
layout:     post
title:      "REDIS系列之底层数据结构"
subtitle:   ""
date:       2017-12-01 12:00:00
author:     "Tango"
header-img: "img/post-bg-universe.jpg"
catalog: true
tags:   
    - REDIS
    - KEY-VALUE
    - 缓存
---

Redis是一款优秀的**key-value**数据库，其中存储的键值对都是有对象(Object)组成，可以存储**字符串对象、哈希对象、列表对象、集合对象、有序集合对象**;由于C语言中没有相关对象的实现，Redis自身扩种底层的数据结构实现上述对象的存储，本文将对REDIS数据库底层的数据结构进行介绍。
 
---

## 1. SDS字符串

Redis并没有直接采用C语言中字符串表示(以空字符结尾的字符数据，一下简称C字符串)，而是扩展一种名为简单动态字符串`(Simple Dynamic String, SDS)`的数据结构，并将SDS作为Redis默认字符串表示。其主要出于**安全性、效率性**考虑。SDS数据结构如下:  

```C  
typdef struct sdshdr{

  //记录buff数组中以使用字节的数量  
  //相当于sds字符串的长度  
  int len;  
  
  //buff中卫使用字节的数量
  int free;
  
  //字节数组用于保存字符串
  char *buff;
  
} SDS;

```
#### 效率性
C字符串以‘\0’结尾，获取字符串长度需要遍历数组，其时间复杂度O(N),伪代码如下:  
  
```
  int cnt = 0;  
  while( *buff++ != '\0'){
     cnt++;
  }
  return cnt;
```
对于SDS字符串来说,`sdshr->len`即是字符串长度，时间复杂度为`O(1)`.
字符串数据结构是在程序中经常会使用到的数据结构,Redis把字符串长度时间复杂度从O(N)降到O(1),这确保获取字符串长度的工作不会成为Redis的性能瓶颈。

#### 安全性

除了获取字符串的复杂度高之外，C字符串不记录自身长度带来另一个问题是容器造成缓冲区溢出。举个例子,<string.h>/strcat函数可以将src字符串中的内容拼接到dest字符串的末尾:
 `char *strcat(char *dest,char *src) `  
 假如在在内存中有两个C字符串`s1="hello"`和`s2="redis"`,如图所示:
   
![](/img/in-post/redis/redis-sds.png)

如果程序执行`stcat(s1," world");`将s1的内容修改为`“hello world”`,那么在strcat函数执行之后,s1数据将溢出，s2的数据将被破坏。  
与C字符串不同，SDS自动空间分配策略防止发生缓冲区溢出的可能性;当SDS API需要对SDS进行修改时，API首先检查SDS空间是否满足修改的要求,如果不满足的话，API将进行空间扩展至执行修改所需大小,然后执行拼接,伪代码如下：  

```
  strcat(SDS dest,SDS src){
  
   if(src->len > dest->free){
     //重新分配空间，再复制
   }else{
     //复制字符串
   }
  }

```
此外，当SDS的API对一个SDS进行修改，发生缓冲区扩充时候，redis采用空间预分配机制，即程序不仅分配SDS修改所必须要的空间，还会分配额外的未使用空间。具体策略分为以下两个方面:    
   
-  如果修改后的SDS的长度(len属性)**小于1M**，则程序在分配缓冲区时，回额外分配len长度**同样大小的未使用空间**(free属性)。  
-  如果修改后的SDS的**长度大于1M**,则程序在分配缓冲区时，会**额外分配1M未使用的空间**(free属性)。  

#### 二进制安全性
C字符串的长度以'\0'结尾，并且除了结尾字符可以是'\0'，字符串里面不能包含空字符，否则程序最先被程序读入的空字符会被误认为是字符串的结尾，这限制C字符串之鞥保存文本数据，不能保存图像、音频、压缩文件这样的二进制文件。但是**Redis采用len属性标志字符串的结尾**，这使得Redis可以存储多种类型数据，是二进制安全。

### 总结
Redis只会使用C字符串作为字面量，大多数情况下Redis使用SDS座位字符串的表示。
比起C字符串，SDS具有如下**优点**:  
  
- 常熟复杂度获取字符串长度  
- 避免缓冲区溢出  
- 预分配机制减少修改字符串时内存冲分配次数。
- 二进制安全  


## 2.链表
链表作为一种高效的数据存储结构内嵌在许多高级语言中，但是C语言没有list结构,因此Redis定义一种链表结构,其数据结构如下:

```C
//链表节点
typedef struct {

   //前置节点
   struct listNode *prev;
   
   //后置节点
   struct *listNode *next;
   
   //节点值
   void *value;
   
} listNode;

//链表
typedef struct {

   //头指针
   listNode *head;
  
   //尾指针
   listNode *tail;
  
   //长度
   unsigned long len;
  
   //节点值复制函数
   void *(*dup)(void *ptr);
  
   //节点值释放函数
   void *(*free)(void *ptr);
  
   //节点值对比函数
   int (*match)(void *ptr,void *key);
}
```
list结构为链表提供了表头指针head、表尾指针tail以及链表长度计数器le，而dup、free\match函数则是用于实现多态链表所需的类型特定函数。
下图是一个list链表结构示意图:
![](/img/in-post/redis/redis-list-struct.png)

### 总结  

- 链表是一个双端链表，每个链表节点有一个`listNode`结构来表示  
- 每个链表使用`list`结构来表示，这个结构带有头指针、尾指针以及长度等信息  
- Redis链表可以保存各种不同类型的值  
- 广泛应用于实现redis各种功能


## 3.字典
字典又叫关联数组、映射(map),是一种保存键值对的抽象数据结构。字典在Redis中应用相当广泛，器数据库就是使用字典座位底层实现，对数据库的增删改查操作也是构建在对字典的操作之上。
其数据结构如下所示：

```C
typedef struct dictht {

   // 哈希表数组
   dictEntry **table;
   
   //哈希表大小
   unsigned long size;
   
   //哈希表大小掩码，用于计算索引值
   unsigned long sizemask;
   
   //该hash表已有节点的数量
   unsigned long used;
   
} dictht;

typedef struct dictEntry {
    //键
    void *key;
    
    //值
    union {
      void *val;
      uint64_tu64;
      int64_ts64;
    } v;
    
    //指向下个哈希表节点，形成链表
    struct dictEntry *next;
   
} dictEntry;

typedef struct dict{

    //类型特定函数
    dictType *type;
    
    //私有数据
    void *pricdata;
    
    //hash表
    dictht ht[2];
    
    //rehash索引
    int rehashidx;

} dict;
```
如图是一个例子:
![](/img/in-post/redis/redis-map.jpg)

#### 哈希表扩容
哈希表**扩容步骤**如下:  

- 计算扩容后的内存,分配相应存储空间赋值给`ht[1]`
- 将`ht[0]`中元素根据hash算法**全部**重新映射到`ht[1]`空间中
- 回收原来`h[0]`内存空间
- 将`ht[1]`赋值给`ht[0]`

哈希表中元素个数可能很多,如果**一次性**进行rehash将会**耗费大量时间**，**阻塞**服务端对客户端服务进程，因此哈希表采用**一种渐进式hash算法**,即是将rehash动作**分多次、渐进式**地完成。其步骤如下:  

- dict数据结构中`rehashidx`标志 rehash的位置。
- 为`ht[1]`分配内存空间，让字典同时持有`ht[0]`和`ht[1]`两个hash表。
- 在rehash期间，每次对字典进行添加、删除、查找、或者更新操作时候程序处理执行指定操作意外，还会顺带将ht[0]哈希表在`rehashidx`索引上的所有键值对rehash到`ht[1]`,当rehash完成之后，程序将rehashidx属性的值+1
- 随着字典操作的不断执行，最终`ht[0]`内的所有键值对都会被rehash到`ht[1]`,这时将`rehashidx`置为-1,表示rehash完成。
- 将`ht[1]`赋值给`ht[0]`,回收`ht[0]`空间。

 
### 总结
- 字典被广泛应用于实现Redis各种功能  
- Redis中的字典**使用hash表作为底层实现**，每个字典带有**两个哈希表**，一个平时使用，另一个仅在进行rehash时使用  
- 哈希表采用**开链方式**解决键冲突  
- 在对**哈希表进行扩展操作**时，程序将现有的键值对rehash到新哈希表中，并且这个过程并不是一次性完成的，而是**渐进式完成**。



## 4.压缩列表
压缩列表`ziplist`则是列表键和哈希键的底层实现,当存储的元素较少或者长度较短时候，为节约内存，则使用连续的存储空间存储元素。压缩列表是为了**提高内存的使用率**，是由一系列特殊编码的连续内存块组成的顺序型数据结构。一个压缩列表尅包含任意多个节点，每个节点可以保存字节数组或者整数值。
其数据结构如下：

```
typedef struct ziplist{

    //记录整个压缩列表占用的字节数
    uint32_t zlbytes;
    
    //记录压缩列表的尾节点距离起始节点字节数
    uint32_t zltail;
    
    //记录压缩列表包含节点数量
    uint16_t zllen;
    
    //存储的元素
    zlentry extryX;
    
    //压缩列表结尾特殊标记
    uint8_t zlend;

} ziplist;

typedef struct zlentry {

    //前一个节点字节长度
    uint previous_entry_length;
 
    //编码方式
    uint encoding;
 
    //字节数组
    uint8_t contents[];

}

```

如下图是一个例子：
![](/img/in-post/redis/redis-ziplist.jpg)

### 总结
- 压缩列表是一种**提高内存使用效率**的**顺序型**数据结构
- 压缩列表可以包含多个节点，每个节点保存一个字节数组或者整数
- 压缩列表被用作列表**键和哈希键的底层实现之一**。


## 5.整数集合
整数集合`intset`是Redis底层存储整数值的集合抽象数据结构，可以保存`int16_t` 、`int32_t`、`int64_t`的整数值，并且保证**集合中不会出现重复元素**。其数据结构如下:  

```C
typedef struct intset{

    //编码方式
    uint32_t encoding;
    
    //数组长度，即是集合元素个数
    uint32_t length;
    
    //保存整数
    int8_t contents[];
    
} intset;
``` 
  
- contents是底层缓冲区实现，保存数据本身,其数据按照大小顺序排列。
- encoding编码方式，标志数据是16位、32位还是64位编码方式。
- length表示contents中存储整数的个数。   
 
下图是一个例子:
![](/img/in-post/redis/redis-intset.jpg)  

当向集合中添加新元素，如果现有编码长度小于新元素的长度，就需要进行集合编码升级，升级分为3步:  

 - 根据新编码方式,计算底层数组的长度,为新的数组分配内存空间。
 - 将原有数组中元素进行扩充编码方式，并放在心数组中合适位置。
 - 维护新数组的有序性,并将contents指向新数组,回收旧数组。  
 

### 总结
- 整数集合是集合键的底层实现之一
- 其底层实现为数组，数组以有序、无重复的方式保存几个元素
- 整数集合支持自动升级编码方式
- 整数集合不支持降级操作

---

**本文以上主要介绍Redis中常用的5中底层数据结构，其作为五大对象Object的底层实现基本数据结构,理解其原理很容易理解Object实现原理以及常用命令的实现原理。接下来文章将介绍五大对象`string`、`set`、`sortedset`、`hash`、`list`的实现原理。**  


> **参考文献**《Redis设计与实现(第二版)》

