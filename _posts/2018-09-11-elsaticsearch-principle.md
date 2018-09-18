---
layout:     post
title:      "Elasticsearch原理分析"
subtitle:   ""
date:       2018-07-05 12:00:00
author:     "Tango"
header-img: "img/post-bg-universe.jpg"
catalog: true
tags:   
    - Elasticsearch
    - 系统架构
---

ElasticSearch是一个基于Lucence的搜索服务器。它提供了一个分布式多用户能力的全文搜索引擎，基于RESTful web接口。
##ES核心概念
###系统概念
####节点node
节点可以理解为一个es服务器，每个节点能够独立对外提供es服务。多个节点组网则构成集群。
####集群cluster
代表一个集群，集群中有多个节点，其中有一个为主节点，这个主节点是可以通过选举产生的，主从节点是对于集群内部来说的。es的一个概念就是去中心化，字面上理解就是无中心节点，这是对于集群外部来说的，因为从外部来看es集群，在逻辑上是个整体，你与任何一个节点的通信和与整个es集群通信是等价的。
####分片shards
代表索引分片，es可以把一个完整的索引分成多个分片，这样的好处是可以把一个大的索引拆分成多个，分布到不同的节点上。构成分布式搜索。分片的数量只能在索引创建前指定，并且索引创建后不能更改。
####副本replicas
代表索引副本，es可以设置多个索引的副本，副本的作用一是提高系统的容错性，当某个节点某个分片损坏或丢失时可以从副本中恢复。二是提高es的查询效率，es会自动对搜索请求进行负载均衡。

上述关系可以表示为如下关系:集群-节点-索引-分片-副本依次包含。

```
{  
  "cluster": {  
    "node": {  
      "index0": {  
        "shard0": {
          "replicas0": 0,
          "replicas1": 1
        },
        "shard1": {
          "replicas0": 0,
          "replicas1": 1
        },
        "shard2": {
          "replicas0": 0,
          "replicas1": 1
        }
      }
    }
  }
}
```

###数据概念

|ES|mysql|备注|  
|--|--|--|  
|索引Index|数据库||  
|类型Type|数据表||  
|文档Document|记录||  

### ES物理文件结构

```
└── nodes
    └── 0
        ├── indices
        │   └── user
        │       ├── 0
        │       │   ├── index
        │       │   │   ├── segments_2
        │       │   │   ├── segments.gen
        │       │   │   └── write.lock
        │       │   ├── _state
        │       │   │   └── state-0.st
        │       │   └── translog
        │       │       └── translog-1536809664493
        │       ├── 1
        │       │   ├── index
        │       │   │   ├── segments_2
        │       │   │   ├── segments.gen
        │       │   │   └── write.lock
        │       │   ├── _state
        │       │   │   └── state-0.st
        │       │   └── translog
        │       │       └── translog-1536809664502
        │       ├── 2
        │       │   ├── index
        │       │   │   ├── _0.cfe
        │       │   │   ├── _0.cfs
        │       │   │   ├── _0.si
        │       │   │   ├── segments_3
        │       │   │   ├── segments.gen
        │       │   │   └── write.lock
        │       │   ├── _state
        │       │   │   └── state-0.st
        │       │   └── translog
        │       │       └── translog-1536809664503
        │       ├── 3
        │       │   ├── index
        │       │   │   ├── _1.cfe
        │       │   │   ├── _1.cfs
        │       │   │   ├── _1.si
        │       │   │   ├── segments_4
        │       │   │   ├── segments.gen
        │       │   │   └── write.lock
        │       │   ├── _state
        │       │   │   └── state-0.st
        │       │   └── translog
        │       │       └── translog-1536809664503
        │       ├── 4
        │       │   ├── index
        │       │   │   ├── _2.cfe
        │       │   │   ├── _2.cfs
        │       │   │   ├── _2.si
        │       │   │   ├── segments_4
        │       │   │   ├── segments.gen
        │       │   │   └── write.lock
        │       │   ├── _state
        │       │   │   └── state-0.st
        │       │   └── translog
        │       │       └── translog-1536809664598
        │       └── _state
        │           └── state-1.st
        ├── node.lock
        └── _state
            └── global-3.st
```
##ES系统架构
![](/img/in-post/post-inpost-es.png)
如图描述三个节点组成一个集群，每个节点保有三个索引。每个索引分片数为3，每个分片副本数为1.  
说明：  
1.ES集群是去中心化结构，即无主节点，各个节点地位平等，都可以对外服务。图中描述NODE0节点为master节点，其实际含义更像是是协调者：维护整个集群的节点状态以及元数据的更新。  
2.replicas一般是分配到与shard不同的节点上，以保证数据的可靠性，图中为描述方便，将replicas与shards定义到同一个节点。  
3.shard命名: shards01表示index0的分片1，shards00即是index0的分片0.  

客户端发送查询请求`CURL -XGET  es-search-dns.com/index0/type/_search?query=xxx`，其执行过程分为两步：  

- query阶段。  

>1.集群中各个节点维护索引分片存储的节点位置，假设客户端请求到NODE1节点，NODE0节点根据自身维护的索引-分片位置关系，将该请求广播到index0几点各个分片中去(图中即NODE0、NODE2)  
2.每个shard是一个完成的Lucene搜索引擎，shards会在本地执行查询请求后会生成一个命中文档的优先级队列。  
3，每个shard返回docId和所有参与排序字段的值例如_score到优先级队列里面  
4.shard将查询的结果返回给NODE1节点，NODE1节点将每个shards返回的数据合并到全局的排序列表。  
5.进入fetch阶段。  

- fetch阶段。
  
>1.fetch阶段获取到符合该搜索条件的docID,因此需要获取到完成的document记录。  
>2.NODE1节点标识了那些document需要被拉取出来，并发送一个批量的mutil get请求到相关的shard上。  
>3.每个shard加载相关document，如果需要他们将会被返回到NODE1节点上  
>4.NODE1节点拉取所有需要的document后，则将数据返回给客户端。  
>5.至此，完成依次查询请求。  

## ES索引存储原理
###不变性
写到磁盘的倒序索引是不变的：自从写到磁盘就再也不变。 这会有很多好处：

- 不需要添加锁。不存在写操作，因此不存在多线程更改数据。
- 提高读性能。一旦索引被内核的文件系统做了Cache，绝大多数的读操作会直接从内存而不需要经过磁盘。
提升其他缓存（例如fiter cache）的性能。其他的缓存在该索引的生命周期内保持有效，减少磁盘I/O和计算消耗。
当然，索引的不变性也有缺点。如果你想让新修改过的文档可以被搜索到，你必须重新构建整个索引。这在一个index可以容纳的数据量和一个索引可以更新的频率上都是一个限制。

如何在不丢失不变形的好处下让倒序索引可以更改？答案是：使用不只一个的索引。 新添额外的索引来反映新的更改来替代重写所有倒序索引的方案。 Lucene引进了per-segment搜索的概念。一个segment是一个完整的倒序索引的子集，所以现在index在Lucene中的含义就是一个segments的集合，每个segment都包含一些提交点(commit point)。

###segment工作流程
>1.新的文档在内存中组织。  
>2.每隔一段时间，buffer将会被提交： 生成一个新的segment(一个额外的新的倒序索引)并被写到磁盘，同时一个新的提交点(commit point)被写入磁盘，包含新的segment的名称。 磁盘fsync，所有在内核文件系统中的数据等待被写入到磁盘，来保障它们被物理写入。  
3.新的segment被打开，使它包含的文档可以被索引。  
4.内存中的buffer将被清理，准备接收新的文档。

当一个新的请求来时，会遍历所有的segments。词条分析程序会聚合所有的segments来保障每个文档和词条相关性的准确。通过这种方式，新的文档轻量的可以被添加到对应的索引中。

####删除和更新
segments是不变的，所以文档不能从旧的segments中删除，也不能在旧的segments中更新来映射一个新的文档版本。取之的是，每一个提交点都会包含一个.del文件，列举了哪一个segmen的哪一个文档已经被删除了。 当一个文档被”删除”了，它仅仅是在.del文件里被标记了一下。被”删除”的文档依旧可以被索引到，但是它将会在最终结果返回时被移除掉。

文档的更新同理：当文档更新时，旧版本的文档将会被标记为删除，新版本的文档在新的segment中建立索引。也许新旧版本的文档都会本检索到，但是旧版本的文档会在最终结果返回时被移除。

####实时索引
在上述的per-segment搜索的机制下，新的文档会在分钟级内被索引，但是还不够快。 瓶颈在磁盘。将新的segment提交到磁盘需要fsync来保障物理写入。但是fsync是很耗时的。它不能在每次文档更新时就被调用，否则性能会很低。 现在需要一种轻便的方式能使新的文档可以被索引，这就意味着不能使用fsync来保障。 在ES和物理磁盘之间是内核的文件系统缓存。之前的描述中,在内存中索引的文档会被写入到一个新的segment。但是现在我们将segment首先写入到内核的文件系统缓存，这个过程很轻量，然后再flush到磁盘，这个过程很耗时。但是一旦一个segment文件在内核的缓存中，它可以被打开被读取。

####更新持久化
不使用fsync将数据flush到磁盘，我们不能保障在断电后或者进程死掉后数据不丢失。ES是可靠的，它可以保障数据被持久化到磁盘。一个完全的提交会将segments写入到磁盘，并且写一个提交点，列出所有已知的segments。当ES启动或者重新打开一个index时，它会利用这个提交点来决定哪些segments属于当前的shard。 如果在提交点时，文档被修改会怎么样？

translog日志提供了一个所有还未被flush到磁盘的操作的持久化记录。当ES启动的时候，它会使用最新的commit point从磁盘恢复所有已有的segments，然后将重现所有在translog里面的操作来添加更新，这些更新发生在最新的一次commit的记录之后还未被fsync。

translog日志也可以用来提供实时的CRUD。当你试图通过文档ID来读取、更新、删除一个文档时，它会首先检查translog日志看看有没有最新的更新，然后再从响应的segment中获得文档。这意味着它每次都会对最新版本的文档做操作，并且是实时的。

####Segment合并
通过每隔一秒的自动刷新机制会创建一个新的segment，用不了多久就会有很多的segment。segment会消耗系统的文件句柄，内存，CPU时钟。最重要的是，每一次请求都会依次检查所有的segment。segment越多，检索就会越慢。

ES通过在后台merge这些segment的方式解决这个问题。小的segment merge到大的，大的merge到更大的。。。

这个过程也是那些被”删除”的文档真正被清除出文件系统的过程，因为被标记为删除的文档不会被拷贝到大的segment中。

##ES工作原理
###Lucene 工作流程
####如何创建索引
- 假设搜索的文档  

>文件一：Students should be allowed to Go out with their friends, but not allowed to drink beer.  
>文件二：My friend Jerry went to school to see his students but found them drunk which is not allowed.

- 将原文档传给分次组件(Tokenizer)

分词组件(Tokenizer)会做以下几件事情( 此过程称为Tokenize) ：

>1.将文档分成一个一个单独的单词。  
2.去除标点符号。  
3.去除停词(Stop word) 。  
4.所谓停词(Stop word)就是一种语言中最普通的一些单词，由于没有特别的意义，因而大多数情况下不能成为搜索的关键词，因而创建索引时，这种词会被去掉而减少索引的大小。经过分词(Tokenizer) 后得到的结果称为词元(Token) 。   
在我们的例子中，便得到以下词元(Token)： 
“Students”，“allowed”，“go”，“their”，“friends”，“allowed”，“drink”，“beer”，“My”，“friend”，“Jerry”，“went”，“school”，“see”，

- 将得到的词元(Token)传给语言处理组件(Linguistic Processor)

语言处理组件(linguistic processor)主要是对得到的词元(Token)做一些同语言相关的处理。 
对于英语，语言处理组件(Linguistic Processor) 一般做以下几点：

>1.变为小写(Lowercase) 。  
2.将单词缩减为词根形式，如“cars ”到“car ”等。这种操作称为：stemming 。  
3.将单词转变为词根形式，如“drove ”到“drive ”等。这种操作称为：lemmatization 。  
在我们的例子中，经过语言处理，得到的词(Term)如下： 
“student”，“allow”，“go”，“their”，“friend”，“allow”，“drink”，“beer”，“my”，“friend”，“jerry”，“go”，“school”，“see”，“his”，“student”，“find”，“them

- 将得到的词(Term)传给索引组件(Indexer)

索引 组件(Indexer)主要做以下几件事情：

>1.利用得到的词(Term)创建一个字典。  
2.对字典按字母顺序进行排序。  
3.合并相同的词(Term) 成为文档倒排(Posting List) 链表。  
到此为止索引已经建好。  

#### 如何检索数据

- 用户输入查询语句
- 对查询语句进行词法分析，语法分析，及语言处理

>词法分析主要用来识别单词和关键字。
语法分析主要是根据查询语句的语法规则来形成一棵语法树。
语言处理同索引过程中的语言处理几乎相同。 
如learned变成learn,tables 变成table等。

- 搜索索引，得到符合语法树的文档。

此步骤有分几小步：
>1.首先，在反向索引表中，分别找出包含lucene，learn，hadoop的文档链表。   
2.其次，对包含lucene，learn的链表进行合并操作，得到既包含lucene又包含learn的文档链表。   
3.然后，将此链表与hadoop的文档链表进行差操作，去除包含hadoop的文档，从而得到既包含lucene又包含learn而且不包含hadoop的文档链表。   
此文档链表就是我们要找的文档。  
根据得到的文档和查询语句的相关性，对结果进行排序 
>>首先，一个文档有很多词(Term)组成 
其次对于文档之间的关系，不同的Term重要性不同 
找出词(Term) 对文档的重要性的过程称为计算词的权重(Term weight) 的过程。 
判断词(Term) 之间的关系从而得到文档相关性的过程应用一种叫做向量空间模型的算法

计算权重的过程
影响一个词(Term)在一篇文档中的重要性主要有两个因素：

Term Frequency (tf)：即此Term在此文档中出现了多少次。tf 越大说明越重要。
Document Frequency (df)：即有多少文档包含次Term。df 越大说明越不重要。

这仅仅只term weight计算公式的简单典型实现。实现全文检索系统的人会有自己的实现，Lucene就与此稍有不同。

判断Term之间的关系从而得到文档相关性的过程，也即向量空间模型的算法(VSM)
我们把文档看作一系列词(Term)，每一个词(Term)都有一个权重(Term weight)，不同的词(Term)根据自己在文档中的权重来影响文档相关性的打分计算。

索引过程：  
1) 有一系列被索引文件  
2) 被索引文件经过语法分析和语言处理形成一系列词(Term) 。  
3) 经过索引创建形成词典和反向索引表。
4) 通过索引存储将索引写入硬盘。  
搜索过程：  
a) 用户输入查询语句。  
b) 对查询语句经过语法分析和语言分析得到一系列词(Term) 。  
c) 通过语法分析得到一个查询树。  
d) 通过索引存储将索引读入到内存。  
e) 利用查询树搜索索引，从而得到每个词(Term) 的文档链表，对文档链表进行交，差，并得到结果文档。  
f) 将搜索到的结果文档对查询的相关性进行排序。  
g) 返回查询结果给用户。  

