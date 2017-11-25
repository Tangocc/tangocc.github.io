
 Redis是一款优秀的key-value数据库，其中存储的键值对都是有对象(Object)组成，可以存储字符串对象、哈希对象、列表对象、集合对象、有序集合对象;由于C语言中没有相关对象的实现，Redis自身扩种底层的数据结构实现上述对象的存储，本文将对REDIS数据库底层的数据结构进行介绍。
 
---

## 1. SDS字符串

首先看看SDS数据结构:  
  
```C  
typdef sdshdr{

  //记录buff数组中以使用字节的数量  
  //相当于sds字符串的长度  
  int len;  
  //buff中卫使用字节的数量
  int free;
  //字节数组用于保存字符串
  char *buff;
} SDS;

```

Redis并没有直接采用C语言中字符串表示(以空字符结尾的字符数据，一下简称C字符串)，而是扩展一种名为简单动态字符串(Simple Dynamic String, SDS)的数据结构，并将SDS作为Redis默认字符串表示。其主要出于安全性、效率性考虑。

#### 效率性
C字符串以‘\0’结尾，获取字符串长度需要遍历数组，其时间复杂度O(N),伪代码如下:  

```
  int cnt = 0;  
  while( *buff++ != '\0'){
     cnt++;
  }
  return cnt;
```
对于SDS字符串来说，’sdshr->len‘即是字符串长度，时间复杂度为O(1).
字符串数据结构是在程序中经常会使用到的数据结构,Redis把字符串长度时间复杂度从O(N)降到O(1),这确保获取字符串长度的工作不会成为Redis的性能瓶颈。

#### 安全性

除了获取字符串的复杂度高之外，C字符串不记录自身长度带来另一个问题是容器造成缓冲区溢出。举个例子,<string.h>/strcat函数可以将src字符串中的内容拼接到dest字符串的末尾:
 `char *strcat(char *dest,char *src) `  
 假如在在内存中有两个C字符串s1="hello"和s2="redis",如图所示  
   
|'h'|'e'|'l'|'l'|'o'|'\0'|'r'|'e'|'d'|'i'|'s'|'\0'|  
|--:|:--:|:--:|:--:|:--:|:--:|:--:|:--:|:--:|:--:|:--:|:--:|:--:|
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
   
- 如果修改后的SDS的长度(len属性)小于1M，则程序在分配缓冲区时，回额外分配len长度同样大小的未使用空间(free属性)。  
-  如果修改后的SDS的长度大于1M,则程序在分配缓冲区时，会额外分配1M未使用的空间(free属性)。  

#### 二进制安全性
C字符串的长度以'\0'结尾，并且除了结尾字符可以是'\0'，字符串里面不能包含空字符，否则程序最先被程序读入的空字符会被误认为是字符串的结尾，这限制C字符串之鞥保存文本数据，不能保存图像、音频、压缩文件这样的二进制文件。但是Redis采用len属性标志字符串的结尾，这使得redsi可以存储多种类型数据，是二进制安全。

### 总结
Redis只会使用C字符串作为字面量，大多数情况下Redis使用SDS座位字符串的表示。
比起C字符串，SDS具有如下优点:  
  
- 常熟复杂度获取字符串长度  
- 避免缓冲区溢出  
- 预分配机制减少修改字符串时内存冲分配次数。
- 二进制安全  

---

## 2.链表
首先看看其数据结构:

```
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
下图是一个list链表结构示意图：
[](/img/redis-list-struct.png)

### 总结  

- 链表是一个双端链表，每个链表节点有一个listNode结构来表示  
- 每个链表使用list结构来表示，这个结构带有头指针、尾指针以及长度等信息  
- redis链表可以保存各种不同类型的值  
- 广泛应用于实现redis各种功能


## 3.字典
字典又叫关联数组、映射(map),是一种保存键值对的抽象数据结构。字典在Redis中应用相当广泛，器数据库就是使用字典座位底层实现，对数据库的增删改查操作也是构建在对字典的操作之上。
其数据结构如下所示：

```
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

### 哈希表扩容


## 总结
- 字典被广泛应用于实现Redis各种功能  
- Redis中的字典使用hash表作为底层实现，每个字典带有两个哈希表，一个平时使用，另一个仅在进行rehash时使用  
- 哈希表采用开链方式解决键冲突  
- 在对哈希表进行扩展操作时，程序将现有的键值对rehash到新哈希表中，并且这个过程并不是一次性完成的，而是渐进式完成。


## 4.跳跃表

## 5.压缩列表

## 6.整数集合









