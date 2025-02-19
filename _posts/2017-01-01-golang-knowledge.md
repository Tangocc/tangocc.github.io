---
layout:     post
title:      "基础知识之Golang"
subtitle:   ""
date:       2017-12-16 12:00:00
author:     "Tango"
header-img: "img/post-bg-universe.jpg"
catalog: true
tags:   
    - golang
    - 基础知识
---  


# 基础知识

## 语言

### Go语言
 
  - 内存管理方式
     - mHeadp->mCentral->mSpan->mObject
     - https://zhuanlan.zhihu.com/p/59125443
  - 协程调度模型GPM：
     - https://studygolang.com/articles/20991
     - https://segmentfault.com/a/1190000016824319
  - channel原理 https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-channel/
  - sync包
  - go泛型
  - strings包
  - sync.Map原理
  - map扩容机制：
     - https://zhuanlan.zhihu.com/p/66676224
  - go垃圾回收
     - https://zhuanlan.zhihu.com/p/82921000
     - https://zhuanlan.zhihu.com/p/74853110
  - context包
     - https://zhuanlan.zhihu.com/p/88915174
     - 作用：提供一种函数/协程 数据传递/通信、超时、取消、消息通知等机制
     - 基础类型： context.TODO()/context.Background()
     - 接口：Deadline/Done/Err/Value
     - WithCancel/WithTimeout/WithValue/WithDeadline
  - go语言新特性：
     - https://studygolang.com/articles/26529?fr=sidebar
  - go逃逸分析：
          - 在编译原理中，分析指针动态范围的方法称之为逃逸分析。通俗来讲，当一个对象的指针被多个方法或线程引用时，我们称这个指针发生了逃逸。
更简单来说，逃逸分析决定一个变量是分配在堆上还是分配在栈上。
       - 局部变量作为指针返回
       - 对象指针被多个子程序
       - 切片或者map存储指针
  - new 和 make区别
   - new: 创建
   - make: 创建及初始化, slice/map/channel

### php语言
 
  - 字符串操作函数
   - strlen
   - explode
   - implode
   - substr
   - trim
   - 字符串遍历
   
- 数组函数：
  - 二维数组
  - array_fill ( int $start_index , int $num , mixed $value ) : array
  - array_pop ( array &$array ) : mixed
  - array_push ( array &$array , mixed $value1 [, mixed $... ] ) : int
  - ksort ( array &$array [, int $sort_flags = SORT_REGULAR ] ) : bool
  - usort ( array &$array , callable $value_compare_func ) : bool
  - array_shift ( array &$array ) : mixed
  - array_unshift()
  
- array实现原理
- php请求处理流程
  - https://baijiahao.baidu.com/s?id=1637483210158521104&wfr=spider&for=pc
- php如何实现弱类型
- php生命周期
   - https://segmentfault.com/a/1190000013321594
- 内存管理
   - https://segmentfault.com/a/1190000014764790
    
     
### 数据结构与算法

#### 数据结构

 - 树
     - 前序/中序/后序遍历(递归/非递归)
     - 树高度
     - 深度优先遍历
     - 广度优先遍历
 - 图
     - 最短路径
     - 最小生成树
 - 排序算法
     - 快排
     - 归并排序
     - 冒泡排序
     - 堆排序
 - 查找
     - 二分查找
     - KMP算法
#### 算法
 - 最大回文子串
 - 最大递增序列

### 计算机操作系统
  - linux系统
    - top / netstat / ps / lsof
    - awk / sed 
    - 文件系统原理
  - CPU管理
  - 内存管理
  - 进程管理
  - 存储器管理
  - 设备管理


### 计算机网络
 - ISO七层模型
 - TCP/IP
     - 滑动窗口
     - 拥塞控制
     - 三次握手四次挥手
     - TCP报文结构
     - 粘包
     - tcp/udp区别
 - Https
     - Http报文结构
     - 常用状态码
     - cookie/session
     - https原理
     - http2特性: https://www.jianshu.com/p/67c541a421f9
     - 双向流、消息头压缩、单 TCP 的多路复用、服务端推送等特性
 - 网络安全
     - 拒绝服务 DDos 
     - SQL注入
     - dns劫持 
     
### 数据库
 - 数据库整体架构
 - binlog、redolog原理
 - ACID实现原理
 - 分库分表、分区
 - 事务隔离级别
 - 索引原理
 - MVCC原理
 - MySQL行锁 - 索引上加锁
 - mysql行锁机制
     - https://juejin.im/post/6844903654328107015 

### redis
 - 缓存与数据一致性
 - AoF和RDB的区别
 - redis key值过期策略和淘汰策略
 - redis分布式锁注意事项
 - watch原理与事务机制及乐观锁
 - 发布订阅模式
    - https://redisbook.readthedocs.io/en/latest/feature/pubsub.html
 - 主从同步原理：
   - https://www.jianshu.com/p/5147a6825322
 - psync原理
   - https://zhuanlan.zhihu.com/p/150960398
 - 遍历key值: SCAN命令
 - 哨兵和集群模式的区别：
   - 哨兵模式解决高可用故障发现与故障转移。集群则是实现分区扩展内存
 - redis新特性
   - 多线程读取+解析命令 
   - disque数据类型实现队列
 - Redis选主原理：
   - Gossip：接收不到主节点的下线 

### 消息队列
  - Kafka
     - https://baijiahao.baidu.com/s?id=1655359291726496245&wfr=spider&for=pc
     - https://www.cnblogs.com/frankdeng/p/9310684.html
  

### 设计题
  - LRU缓存算法
  - 单例模式
  - **线程池**
  - 两个线程交替打印字符串
  - 连接池
  - 如何实现一个延迟队列
      - redis zset结构
      - Redis过期删除回调
  - 限流器
      - 计数器
      - 令牌桶
      - 漏斗
  
  
### 设计模式
  - 职责链模式
  - 命令模式
  - 桥接模式
  - 单例模式
  - 简单工厂
  

### 容器化技术
 
 - docker原理
     - namespace
     - cgroup
 - k8s组件及其作用
    - master
        - api-server
        - controller-manager
        - scheduler
   - node
      - kubelet
      - kube-proxy
   - etcd
   
  - k8s资源类型:pod/service/deployment/replicaController/Replica Set
  - k8s网络模式:
  
  ### 微服务
  - 什么是微服务
  - 微服务分拆原则
  
  ### 分布式
  - CAP理论
     - https://zhuanlan.zhihu.com/p/50990721
     - https://www.zhihu.com/question/54105974
  - BASE理论
      - Basically Available
      - Soft State
      - Eventually constistent
  - 两阶段提交
  - TCC
     - https://www.cnblogs.com/jajian/p/10014145.html
  - 领域驱动设计
  - 分布式数据一致性
  - Raft算法：http://thesecretlivesofdata.com/raft/
      - 三种角色： Leader /Follower / Candidate
      - 两个阶段：选主、日志同步


参考文档：
https://github.com/xingshaocheng/architect-awesome/blob/master/README.md#%E4%BA%8B%E5%8A%A1-acid-%E7%89%B9%E6%80%A7
