---
layout:     post
title:      "NGINX系列之进程模型"
subtitle:   ""
date:       2018-03-20 12:00:00
author:     "Tango"
header-img: "img/post-bg-universe.jpg"
catalog: true
tags:   
    - nginx
    - 系统架构
    - 进程模型
---

在上一节中分析了nginx主流程，在`mian`函数中完成服务器的配置文件解析以及模块初始化工作后，根据系统设置进入单进程或者多进程模式，本文将分析nginx进程模型。

### 进程模型简介

在web服务中，随着用户基数增长，技术演进的趋势是提高系统的并发性和稳定性，一种方式是通过扩展机器的个数实现负载均衡，通过多台机器的量变引起质变，提高系统的并发性，显然这种方式资源利用率较低，成本较高；第二种方式则是演进web服务系统架构，提高单台机器的并发性，例如异步、多线程技术，通过单台机器的并发性提高提升系统整体性能。nginx的高性能与其架构设计密不可分，其主要来源于两个方面：

- 异步非阻塞
  关于异步非阻塞不做过多介绍，参考文章 [5种网络编程模型](https://blog.csdn.net/u013291818/article/details/61428269) 。nginx底层采用epoll事件处理机制，提高单个进程的并发性。
- 多进程单线程
  单线程避免资源争夺的竞争以及上下文切换带来的损耗，多进程提高“机器”个数，提高系统的整体并发性。

从上文中可知，nginx分为单进程模式和多进程模式，单进程模式常常在开发环境调试时候使用，在对外服务时nginx多以多进程方式工作。多进程工作方式中为方便进程的统一管理，系统中分为一个master进程和多个work进程，master进程主要负责信号处理以及work进程的管理，包括接收外界信号、向worker进程发送信号，监控worker进程的运行状态等，不直接对外提供web服务;worker进程则主要对外提供web服务，各个work进程之间相互隔离且相互平等，从而避免进程之间的资源竞争导致的性能损耗，worker进程数目可以设置，一般设置为机器cpu核数(原因是降低进程之间上下文切换带来的损耗)。其进程模型可以用下图表示：  

![](/img/in-post/post-2018-07-03-nginx-process-model.png)

### 流程伪代码

#### master主进程
   主进程启动以后首先初始化系统相应的信号量标志位，然后根据配置参数(子进程个数、最大连接数等)通过fork复制创建工作进程，worker进程与master进程具有相同的上下文环境，接下来主进程和工作进程进入不同的循环，主进程保存子进程返回的pid写入文件，接着主进程进入信号处理的循环，监听系统接收到的(例如nginx -reload)信号并进行相关的处理。
master进程有如下几种信号:  

- TERM INT 快速停止
- QUIT 从容关闭
- HUP 平滑重启

master进程流程伪代码如下：

```
void ngx_master_process_cycle(ngx_cycle_t *cycle){


	// 1.初始化信号位
    ngx_init_singal_flag();
    // 2.加载配置参数
    ngx_load_conf();
    // 3.fork创建worker进程
    ngx_start_worker_processes();
    // 4.启动缓存管理工作进程
    ngx_start_cache_manager_processes();
    //进入信号处理循环
    while(1){
    
    switch(singal){
	    case delay:
	          ngx_ delay_singal_handler();
	          break;
	    case quit:
	          ngx_quit_singal_handler();
	          break;
		case terminate:
	          ngx_terminate_singal_handler();
	          break;
		case reconfigure:
	          ngx_reconfigure_singal_handler();
	          break;
		.....    
    }
       
}

```

#### worker工作进程
  通过主进程fork复制出worker进程，worker进程环境变量(监听socket、文件描述符等)都一样，因此各个worker进程完成等同，在相同的socket端口监听，请求到来时，每个worker工作进程都会监听到，但最终只会有一个worker进程会接受并处理，（此处涉及到惊群现象,将在下篇文章介绍)。创建工作进程以后，工作进程进入循环，首先处理退出信号，然后进入事件处理过程ngx_process_events_and_timers(cycle)，在进程处理函数中，首先处理定时任务，然后处理读取任务再处理写任务。这里有三个小问题：  

- 为什么要采用这种处理顺序？
- 定时任务是如何保证基本准确定时的?
- 事件处理怎么保证一个请求只被一个进程处理的?

上述问题将在下一篇事件处理模型中介绍。
流程伪代码如下：

```
static void ngx_worker_process_cycle(ngx_cycle_t *cycle, void *data)
{
    // 1.初始化worker进程
    ngx_worker_process_init(cycle, worker);

    while(1){	
		 // 1.处理退出信息
        if (ngx_exiting) {
            if (ngx_event_no_timers_left() == NGX_OK) {
                 ngx_worker_process_exit(cycle);
            }
        }
        // 2.事件处理 将在下一章节介绍
        ngx_process_events_and_timers(cycle);
		 // 3.
        if (ngx_terminate) {
            ngx_worker_process_exit(cycle);
        }

        if (ngx_quit) {
            if (!ngx_exiting) {
                ngx_exiting = 1;
                ngx_set_shutdown_timer(cycle);
                ngx_close_listening_sockets(cycle);
                ngx_close_idle_connections(cycle);
            }
        }

        if (ngx_reopen) {
           ngx_reopen_files(cycle, -1);
        }
    }
}

```


### 精简源码分析

#### 多进程模式`ngx_master_process_cycle`
- 文件位置：/src/os/unix/ngx_process_cycle.c

```
void ngx_master_process_cycle(ngx_cycle_t *cycle){
    char              *title;
    u_char            *p;
    size_t             size;
    ngx_int_t          i;
    ngx_uint_t         n, sigio;
    sigset_t           set;
    struct itimerval   itv;
    ngx_uint_t         live;
    ngx_msec_t         delay;
    ngx_listening_t   *ls;
    ngx_core_conf_t   *ccf;

    // 1.配置相应信号标志位
    sigemptyset(&set);
    sigaddset(&set, SIGCHLD);
    sigaddset(&set, SIGALRM);
    sigaddset(&set, SIGIO);
    sigaddset(&set, SIGINT);
    sigaddset(&set, ngx_signal_value(NGX_RECONFIGURE_SIGNAL));
    sigaddset(&set, ngx_signal_value(NGX_REOPEN_SIGNAL));
    sigaddset(&set, ngx_signal_value(NGX_NOACCEPT_SIGNAL));
    sigaddset(&set, ngx_signal_value(NGX_TERMINATE_SIGNAL));
    sigaddset(&set, ngx_signal_value(NGX_SHUTDOWN_SIGNAL));
    sigaddset(&set, ngx_signal_value(NGX_CHANGEBIN_SIGNAL));
    // 2.配置参数
    ccf = (ngx_core_conf_t *) ngx_get_conf(cycle->conf_ctx, ngx_core_module);

    // 3.启动worker进程
    ngx_start_worker_processes(cycle, ccf->worker_processes,
                               NGX_PROCESS_RESPAWN);
    // 4.缓存管理进程
    ngx_start_cache_manager_processes(cycle, 0);

    // 5.master进程进入循环 处理信号
    for ( ;; ) {
        if (delay) {
        
        }

         if (ngx_reap) {
            ngx_reap = 0;
            live = ngx_reap_children(cycle);
        }

        if (!live && (ngx_terminate || ngx_quit)) {
            ngx_master_process_exit(cycle);
        }

        if (ngx_terminate) {
           ngx_signal_worker_processes(cycle,ngx_signal_value(NGX_TERMINATE_SIGNAL));
            }
            continue;
        }

        if (ngx_quit) {
            ngx_signal_worker_processes(cycle,ngx_signal_value(NGX_SHUTDOWN_SIGNAL));
        }

        if (ngx_reconfigure) {
            if (ngx_new_binary) {
                ngx_start_worker_processes(cycle, ccf->worker_processes,NGX_PROCESS_RESPAWN);
                ngx_start_cache_manager_processes(cycle, 0);
                ngx_noaccepting = 0;
                continue;
            }

            cycle = ngx_init_cycle(cycle);
            
            ccf = (ngx_core_conf_t *) ngx_get_conf(cycle->conf_ctx,
                                                   ngx_core_module);
            ngx_start_worker_processes(cycle, ccf->worker_processes,
                                       NGX_PROCESS_JUST_RESPAWN);
            ngx_start_cache_manager_processes(cycle, 1);

            /* allow new processes to start */
            ngx_msleep(100);

            live = 1;
            ngx_signal_worker_processes(cycle,ngx_signal_value(NGX_SHUTDOWN_SIGNAL));
        }

        if (ngx_restart) {
            ngx_start_worker_processes(cycle, ccf->worker_processes,NGX_PROCESS_RESPAWN);
            ngx_start_cache_manager_processes(cycle, 0);
                   }

        if (ngx_reopen) {
            ngx_reopen = 0;
            ngx_reopen_files(cycle, ccf->user);
            ngx_signal_worker_processes(cycle,ngx_signal_value(NGX_REOPEN_SIGNAL));
        }

        if (ngx_change_binary) {
            ngx_change_binary = 0;
            ngx_new_binary = ngx_exec_new_binary(cycle, ngx_argv);
        }

        if (ngx_noaccept) {
            ngx_noaccept = 0;
            ngx_noaccepting = 1;
            ngx_signal_worker_processes(cycle,ngx_signal_value(NGX_SHUTDOWN_SIGNAL));
        }
    }
}
```
#### 启动worker进程`ngx_start_worker_processes`

```
static void
ngx_start_worker_processes(ngx_cycle_t *cycle, ngx_int_t n, ngx_int_t type)
{
    ngx_int_t      i;
    ngx_channel_t  ch;

    for (i = 0; i < n; i++) {
        ngx_spawn_process(cycle, ngx_worker_process_cycle,(void *) (intptr_t) i, "worker process", type);
        ngx_pass_open_channel(cycle, &ch);
    }
}
```
#### 启动缓存进程`ngx_start_cache_manager_processes `

```
static void ngx_start_cache_manager_processes(ngx_cycle_t *cycle, ngx_uint_t respawn){
    ngx_uint_t       i, manager, loader;
    ngx_path_t     **path;
    ngx_channel_t    ch;

    manager = 0;
    loader = 0;

    path = ngx_cycle->paths.elts;
    for (i = 0; i < ngx_cycle->paths.nelts; i++) {

        if (path[i]->manager) {
            manager = 1;
        }

        if (path[i]->loader) {
            loader = 1;
        }
    }

    if (manager == 0) {
        return;
    }

    ngx_spawn_process(cycle, ngx_cache_manager_process_cycle,
                      &ngx_cache_manager_ctx, "cache manager process",
                      respawn ? NGX_PROCESS_JUST_RESPAWN : NGX_PROCESS_RESPAWN);
    ngx_spawn_process(cycle, ngx_cache_manager_process_cycle,
                      &ngx_cache_loader_ctx, "cache loader process",
                      respawn ? NGX_PROCESS_JUST_SPAWN : NGX_PROCESS_NORESPAWN);
    ngx_pass_open_channel(cycle, &ch);
}

```

#### 单进程`ngx_single_process_cycle`

```
void ngx_single_process_cycle(ngx_cycle_t *cycle){
    ngx_uint_t  i;

    ngx_set_environment(cycle, NULL) 
    for (i = 0; cycle->modules[i]; i++) {
        if (cycle->modules[i]->init_process) {
            if (cycle->modules[i]->init_process(cycle) == NGX_ERROR) {
                /* fatal */
                exit(2);
            }
        }
    }

    for ( ;; ) {
        
        ngx_process_events_and_timers(cycle);
        if (ngx_terminate || ngx_quit) {

            for (i = 0; cycle->modules[i]; i++) {
                if (cycle->modules[i]->exit_process) {
                    cycle->modules[i]->exit_process(cycle);
                }
            }

            ngx_master_process_exit(cycle);
        }

        if (ngx_reconfigure) {
            ngx_reconfigure = 0;
                    cycle = ngx_init_cycle(cycle);
            if (cycle == NULL) {
                cycle = (ngx_cycle_t *) ngx_cycle;
                continue;
            }

            ngx_cycle = cycle;
        }

        if (ngx_reopen) {
            ngx_reopen = 0;
            ngx_log_error(NGX_LOG_NOTICE, cycle->log, 0, "reopening logs");
            ngx_reopen_files(cycle, (ngx_uid_t) -1);
        }
    }
}

```

#### 工作进程循环`ngx_worker_process_cycle `
```
static void ngx_worker_process_cycle(ngx_cycle_t *cycle, void *data)
{
     ngx_worker_process_init(cycle, worker);

    for ( ;; ) {

        if (ngx_exiting) {
            if (ngx_event_no_timers_left() == NGX_OK) {
                ngx_log_error(NGX_LOG_NOTICE, cycle->log, 0, "exiting");
                ngx_worker_process_exit(cycle);
            }
        }
        
        ngx_process_events_and_timers(cycle);

        if (ngx_terminate) {
            ngx_log_error(NGX_LOG_NOTICE, cycle->log, 0, "exiting");
            ngx_worker_process_exit(cycle);
        }

        if (ngx_quit) {
            ngx_quit = 0;
            ngx_setproctitle("worker process is shutting down");

            if (!ngx_exiting) {
                ngx_exiting = 1;
                ngx_set_shutdown_timer(cycle);
                ngx_close_listening_sockets(cycle);
                ngx_close_idle_connections(cycle);
            }
        }

        if (ngx_reopen) {
            ngx_reopen = 0;
            ngx_log_error(NGX_LOG_NOTICE, cycle->log, 0, "reopening logs");
            ngx_reopen_files(cycle, -1);
        }
    }
}


```
#### master进程fork子进程`ngx_spawn_process `

```
ngx_pid_t ngx_spawn_process(ngx_cycle_t *cycle, ngx_spawn_proc_pt proc, void *data,char *name, ngx_int_t respawn)
{
    //创建子进程
    pid = fork();

    switch (pid) {
    case -1:
        ngx_close_channel(ngx_processes[s].channel, cycle->log);
        return NGX_INVALID_PID;
    case 0: //worker进程
        ngx_parent = ngx_pid;
        ngx_pid = ngx_getpid();
        proc(cycle, data);
        break;
    default:
        break;
    }

    ngx_processes[s].pid = pid;
    ngx_processes[s].exited = 0;

    ngx_processes[s].proc = proc;
    ngx_processes[s].data = data;
    ngx_processes[s].name = name;
    ngx_processes[s].exiting = 0;

    switch (respawn) {

    case NGX_PROCESS_NORESPAWN:
        ngx_processes[s].respawn = 0;
        ngx_processes[s].just_spawn = 0;
        ngx_processes[s].detached = 0;
        break;

    case NGX_PROCESS_JUST_SPAWN:
        ngx_processes[s].respawn = 0;
        ngx_processes[s].just_spawn = 1;
        ngx_processes[s].detached = 0;
        break;

    case NGX_PROCESS_RESPAWN:
        ngx_processes[s].respawn = 1;
        ngx_processes[s].just_spawn = 0;
        ngx_processes[s].detached = 0;
        break;

    case NGX_PROCESS_JUST_RESPAWN:
        ngx_processes[s].respawn = 1;
        ngx_processes[s].just_spawn = 1;
        ngx_processes[s].detached = 0;
        break;

    case NGX_PROCESS_DETACHED:
        ngx_processes[s].respawn = 0;
        ngx_processes[s].just_spawn = 0;
        ngx_processes[s].detached = 1;
        break;
    }

    if (s == ngx_last_process) {
        ngx_last_process++;
    }

    return pid;
}

```

在完成工作进程和主进程创建工作以后，系统进入事件处理阶段，其功能实现主要在函数`ngx_process_events_and_timers `中，文章介绍事件处理模型。