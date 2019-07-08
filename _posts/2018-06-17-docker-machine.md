---
layout:     post
title:      "Docker容器之核心原理"
subtitle:   ""
date:       2018-03-20 12:00:00
author:     "Tango"
header-img: "img/post-bg-universe.jpg"
catalog: true
tags:   
    - Docker
    - 容器
---

# Docker容器核心原理


在上一节中已经介绍docker的使用以及常用命令，本章则介绍docker实现的核心原理，在开始之前提出几个问题：

- docker如何实现容器之间的隔离？容器内的主机名和宿主机的主机名一样的么？为什么？

- 宿主机中进程ID号和容器内的进程ID号会产生冲突么？

- 修改容器的hosts文件会对宿主机的hosts文件产生影响么？

- 如果保证或者限制容器内的cpu、meme等资源的使用情况？

对于上述问题的答案都是**否**，那么衍生出两个问题：

**为什么呢？docker是如何实现的呢？**

在接下来将针对上述两个问题进行介绍。


## Docker容器的本质


**简单理解，Docker容器就是运行于宿主机器上通过namespace技术进行资源隔离，cgroup技术实现资源限制的一个/或多个进程。换句话说，容器是宿主机上的一个进程。**

比如：
我们在容器中启动nginx进程，则在容器中通过`ps -ef`会发现有一个nginx进程

```
# ps -ef

UID        PID  PPID  C STIME TTY          TIME CMD
root         1     0  0 Jul05 ?        00:00:54 /usr/bin/python /usr/bin/supervisord
work        95     1  0 Jul05 ?        00:00:00 nginx: master process /home/work/odp
work        96    95  0 Jul05 ?        00:00:05 nginx: worker process
work        97    95  0 Jul05 ?        00:00:10 nginx: worker process
work        98    95  0 Jul05 ?        00:00:12 nginx: worker process
work        99    95  0 Jul05 ?        00:00:14 nginx: worker process
work       100    95  0 Jul05 ?        00:00:13 nginx: worker process
work       101    95  0 Jul05 ?        00:00:15 nginx: worker process
work       102    95  0 Jul05 ?        00:00:13 nginx: worker process
work       103    95  0 Jul05 ?        00:00:14 nginx: worker process
root     15541     0  1 09:59 ?        00:00:00 /bin/bash
root     15552 15541  0 09:59 ?        00:00:00 ps -ef
```

同样，我们再宿主机上执行`ps -ef`也会发现上面的进程信息

```

# ps -ef

UID        PID  PPID  C STIME TTY          TIME CMD
root         1     0  0 Jul05 ?        00:00:54 /usr/bin/python /usr/bin/supervisord
work        95     1  0 Jul05 ?        00:00:00 nginx: master process /home/work/odp
work        96    95  0 Jul05 ?        00:00:05 nginx: worker process
work        97    95  0 Jul05 ?        00:00:10 nginx: worker process
work        98    95  0 Jul05 ?        00:00:12 nginx: worker process
work        99    95  0 Jul05 ?        00:00:14 nginx: worker process
work       100    95  0 Jul05 ?        00:00:13 nginx: worker process
work       101    95  0 Jul05 ?        00:00:15 nginx: worker process
work       102    95  0 Jul05 ?        00:00:13 nginx: worker process
work       103    95  0 Jul05 ?        00:00:14 nginx: worker process
root     15541     0  1 09:59 ?        00:00:00 /bin/bash
root     15552 15541  0 09:59 ?        00:00:00 ps -ef

```


既然同属于一个宿主机，那么**多个容器之间是如何实现隔离**的呢？

其原理就是通过linux内核提供的**namespace**技术和**cgroup**技术实现。


## Namespace

### 原理

之前介绍，实现一个隔离的容器环境，要实现独立的网络、独立的进程树、独立的主机名、独立的进程通信方式、独立的文件系统、独立的用户及用户组，针对上述的6项隔离，Linux内核提供6种namespace隔离的系统调用

其核心的实现参数是进程创建函数

```
clone(int *(child_dun)(void *),void * child_stack,int flags,void * arg);

```
- child_fun即是子进程的主函数
- child_stack：子进程堆栈空间
- flags：创建进程的标志位
- args:子进程的参数

**在同一个namespace下的进程可以彼此感知彼此的变化，而对外界进程一无所知。**

clone()实际是对底层系统调用fork()的一种更通用的实现方式，通过flags参数20多种标志位的组合控制进程复制过程中的方方面面，其中涉及6项隔离的控制位为：

|namespace|标志位 | 功能 | 
|----|---|---|---|
|IPC|CLONE_NEWIPC| 隔离信号量、消息队列和共享内存| 
|Mount|CLONE_NEWNS |隔离文件系统  |   
|Network|CLONE_NEWNET |   隔离网络设备、网络栈、端口 | 
|PID|CLONE_NEWPID| 隔离进程树 |  
|User|CLONE_NEWUSER|  隔离用户和用户组  | 
|UTS|CLONE_NEWUTS| 隔离主机名与域名| 

**如何查看进程的namespace?**

从3.8版本的内核开始，在`/proc/[pid]/ns`目录下可以看到指向不同namepspace号的文件，形如：

```
# ls -al
total 0
dr-x--x--x 2 root root 0 Jul  5 12:03 .
dr-xr-xr-x 9 root root 0 Jul  5 12:01 ..
lrwxrwxrwx 1 root root 0 Jul  5 12:03 ipc -> ipc:[4026532183]
lrwxrwxrwx 1 root root 0 Jul  5 12:03 mnt -> mnt:[4026532190]
lrwxrwxrwx 1 root root 0 Jul  5 12:03 net -> net:[4026531956]
lrwxrwxrwx 1 root root 0 Jul  5 12:03 pid -> pid:[4026532191]
lrwxrwxrwx 1 root root 0 Jul  5 12:03 user -> user:[4026531837]
lrwxrwxrwx 1 root root 0 Jul  5 12:03 uts -> uts:[4026531838]

```
**如果两个进程指向的namespace编号相同，就说明它们再同一个namespace下，否则便在不同的namespace里**

namespace中所有的进程都结束，则该namespace自动销毁，(如果该软链文件被打开，则namespace会一直存在)

### 实例

**下面以创建PID隔离为例介绍docker对于namespace的使用**


```
int childMain(void * args){
 print("in child process!");
 return 1;
}


int main(void * args){
   int childPid = clone(childMain,childStack_STACK_SIZE,CLONE_NEWPID|SIGCHLD,NULL);
}
```

上述代码即实现了PID隔离，细心的读者可能会执行上述代码验证通过ps-ef命令验证是否真正实现PID隔离，但是事与愿违，在子进程中仍然会看到父进程中的进程，这是应为ps命令是从/proc挂载目录中读取进程信息，而我们此时并没有实现Mount隔离。

---

## Cgroup

上面已经介绍，Docker通过Linux内核namespace实现一个相对隔离的shell环境，这一节将介绍Docker 如何通过cgroup实现资源的限制。

cgroup 即是Control Group，顾明思义就是把任务放到一个组里面统一加以控制。