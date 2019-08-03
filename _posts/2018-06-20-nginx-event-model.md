---
layout:     post
title:      "NGINX系列之事件模型"
subtitle:   ""
date:       2018-07-08 12:00:00
author:     "Tango"
header-img: "img/post-bg-universe.jpg"
catalog: true
tags:   
    - nginx
    - 事件模型
---

nginx在完成进程的创建后，主进程进入信号处理的循环中，不参与事件处理；worker进程则进入事件处理过程。nginx任何操作，包括定时任务、连接、读写等都可以定义为事件，事件具有的特点是被动特性，即发生才处理，避免因为轮训状态而导致的时间消耗。本章将介绍nginx的事件处理模型。

## 事件模型简介
### 定时器任务
上一章节中提到nginx如何实现定时任务？如何确保定时任务的准确性？  
>nginx将定时任务存储在RBTree红黑树中，每次处理任务之前从红黑树中取出距离当前最近的一次定时任务，假设时间为delta。
>  
>epoll模型在执行epoll_wait函数时候，可以设定等待时间，通过将delta传入该函数，使得epoll事件处理在delta时间后退出等待，从而可以处理定时任务。

### 惊群现象
惊群现象是一种比拟的说法，意思是在鸟群中扔一颗石子，所有的鸟儿都会被惊吓而飞起。
在nginx中，master进程**首先**初始化监听套接字(比如8080)后，通过fork创建worker进程，**所有的worker进程具有相同的进程上下文**，也就是说所有的worker进程都会在**同一个套接字(8080)**监听，当某一个连接到来时候，所有的worker进程**都会**被唤醒(接收到事件发生的信号)，但是最终只会有**一个进程会真正处理该请求**，其他进程又会再次进入“睡眠状态”。这带来的问题就是进程的切换带来**资源的消耗**。

**nginx是如何处理这种问题的呢？**  
nginx通过共享内存的方式创建一把互斥锁(accept_mutex)，同一时刻只能有一个worker进程持有该accept_mutex锁，worker进程在处理accept事件之前首先试着去获取accept_mutex,只有在获得accept_mutex锁的进程接受accept事件，在接受并处理完accept事件后即释放accept_mutex锁，各个worker进程再次进入竞争锁的过程，这样就避免多个进程同时唤醒处理同一个accept事件。其代码如下:  

```
  // 判断是否使用accept_mutex锁
    if (ngx_use_accept_mutex) {
        if (ngx_accept_disabled > 0) {
            ngx_accept_disabled--;
        } else {
            
            if (ngx_trylock_accept_mutex(cycle) == NGX_ERROR) {
                return;
            }
			  // 是否获得accept_mutex锁
            if (ngx_accept_mutex_held) {
              // 设置相应标志位
                flags |= NGX_POST_EVENTS;
            } else {
                if (timer == NGX_TIMER_INFINITE
                    || timer > ngx_accept_mutex_delay)
                {
                    timer = ngx_accept_mutex_delay;
                }
            }
        }
    }

```
该功能的实现要通过配置文件中accept_mutex on/off开关设置，默认是on。此外，accept_mutex同时可以实现负载均衡。


### 事件模型伪代码

为了方便理解，首先给出工作进程执行函数的伪代码：  

```
void ngx_worker_process_cycle(){

   // 1.worker进程初始化
   nginx_worker_process_init();
   
   while(1){
      // 2.处理主进程发送的信号
      ngx_worker_signal_handler();
      
      /*以下为ngx_process_events_and_timers函数*/
      // 3.查找是否有到时的定时任务
      ngx_event_find_timer();
      
      // 4.判断是否获得accept_mutex锁，决定是否执行accept事件。
      if(ngx_trylock_accept_mutex()){
         // 持有accept_mutex锁则设置接受连接标志位
         flags |= NGX_POST_EVENTS;
      }
      // 5.释放锁
      ngx_release_accept_mutex();
      // 6.处理读事件
      ngx_process_read_events();
      // 7.处理写事件
      ngx_process_write_events();
      // 8.处理定时任务
      ngx_event_expire_timers();
      /*以上为ngx_process_events_and_timers函数*/
   }
}


```
工作进程完成初始化以后，进入事件处理模块i：

①首先判断是否有来自主进程信号(退出？重启？)并进行相应处理；

②然后从定时器任务红黑树中取出距离当前时间最近的定时任务，将时间作为epoll_wait等待时间；

③判断是否开启accept_mutex锁，如果开启则试着获取accept_mutex锁，如果获取到则置位NGX_POST_EVENTS，标识接受accept事件；

④处理accept事件并放入放入队列中，立即放弃锁，避免其他进程的等待；

⑤处理读事件

⑥处理读事件

⑦处理定时器任务并移除失效的定时任务。


下面详细分析事件处理模型代码。


## 事件模型源码分析

### 入口函数`ngx_process_events_and_timers`

```

void
ngx_process_events_and_timers(ngx_cycle_t *cycle)
{
    ngx_uint_t  flags;
    ngx_msec_t  timer, delta;

    if (ngx_timer_resolution) {
        timer = NGX_TIMER_INFINITE;
        flags = 0;

    } else {
        //这里从红黑树中找出距离当前时间定时任务最短的时间
        timer = ngx_event_find_timer();
        flags = NGX_UPDATE_TIME;

#if (NGX_WIN32)

        //针对win系统，避免长时间阻塞而无法响应信号，如果定时的时间大于500ms，置为500ms
        if (timer == NGX_TIMER_INFINITE || timer > 500) {
            timer = 500;
        }

#endif
    }

    // 判断是否使用accept_mutex锁
    if (ngx_use_accept_mutex) {
        // 进行负载均衡
        if (ngx_accept_disabled > 0) {
            ngx_accept_disabled--;

        } else {
            //试着去获取accept_mutex锁
            if (ngx_trylock_accept_mutex(cycle) == NGX_ERROR) {
                return;
            }
            //如果获得则设置accept事件标志位
            if (ngx_accept_mutex_held) {
                flags |= NGX_POST_EVENTS;

            } else {
                //否则 不设置标识位 计算下次获取锁的时间
                if (timer == NGX_TIMER_INFINITE
                    || timer > ngx_accept_mutex_delay)
                {
                    timer = ngx_accept_mutex_delay;
                }
            }
        }
    }
    //记录当前时间
    delta = ngx_current_msec;

    // 这里是进行事件处理 对应epoll模块ngx_epoll_process_events
    (void) ngx_process_events(cycle, timer, flags);

    // 统计此次时间处理消耗的时间
    delta = ngx_current_msec - delta;

    //在事件处理模块中将accept事件放入队列中，现在进行处理
    ngx_event_process_posted(cycle, &ngx_posted_accept_events);

    // 如果持有accept_mutex锁，则尽快释放锁。
    if (ngx_accept_mutex_held) {
        ngx_shmtx_unlock(&ngx_accept_mutex);
    }

    if (delta) {
        // 红黑树中到时的定时任务并移除
        ngx_event_expire_timers();
    }
    //处理读写事件
    ngx_event_process_posted(cycle, &ngx_posted_events);
}

```


### 事件处理函数 `ngx_event_process_posted`

流程比较简单，从队列中获取一个事件,然后执行相应的handler方法，直到队列中不存在事件。

```
void ngx_event_process_posted(ngx_cycle_t *cycle, ngx_queue_t *posted)
{
    ngx_queue_t  *q;
    ngx_event_t  *ev;

    while (!ngx_queue_empty(posted)) {

        q = ngx_queue_head(posted);
        ev = ngx_queue_data(q, ngx_event_t, queue);
        ngx_delete_posted_event(ev);
        ev->handler(ev);
    }
}


```

### 红黑树获取距离最近的定时时间`ngx_event_find_timer`

```
ngx_msec_t ngx_event_find_timer(void)
{
    ngx_msec_int_t      timer;
    ngx_rbtree_node_t  *node, *root, *sentinel;

    if (ngx_event_timer_rbtree.root == &ngx_event_timer_sentinel) {
        return NGX_TIMER_INFINITE;
    }

    root = ngx_event_timer_rbtree.root;
    sentinel = ngx_event_timer_rbtree.sentinel;

    node = ngx_rbtree_min(root, sentinel);

    timer = (ngx_msec_int_t) (node->key - ngx_current_msec);

    return (ngx_msec_t) (timer > 0 ? timer : 0);
}
```

###  执行定时任务
遍历红黑树，执行到期的定时任务并移除。

```
void ngx_event_expire_timers(void)
{
    ngx_event_t        *ev;
    ngx_rbtree_node_t  *node, *root, *sentinel;

    sentinel = ngx_event_timer_rbtree.sentinel;

    for ( ;; ) {
        root = ngx_event_timer_rbtree.root;

        if (root == sentinel) {
            return;
        }

        node = ngx_rbtree_min(root, sentinel);

        /* node->key > ngx_current_msec */

        if ((ngx_msec_int_t) (node->key - ngx_current_msec) > 0) {
            return;
        }

        ev = (ngx_event_t *) ((char *) node - offsetof(ngx_event_t, timer));

        ngx_rbtree_delete(&ngx_event_timer_rbtree, &ev->timer);

        ev->timer_set = 0;

        ev->timedout = 1;

        ev->handler(ev);
    }
}

```

### epoll模型

```
static ngx_int_t ngx_epoll_process_events(ngx_cycle_t *cycle, ngx_msec_t timer, ngx_uint_t flags)
{
    int                events;
    uint32_t           revents;
    ngx_int_t          instance, i;
    ngx_uint_t         level;
    ngx_err_t          err;
    ngx_event_t       *rev, *wev;
    ngx_queue_t       *queue;
    ngx_connection_t  *c;

    /* NGX_TIMER_INFINITE == INFTIM */

    ngx_log_debug1(NGX_LOG_DEBUG_EVENT, cycle->log, 0,
                   "epoll timer: %M", timer);

    events = epoll_wait(ep, event_list, (int) nevents, timer);

    err = (events == -1) ? ngx_errno : 0;

    if (flags & NGX_UPDATE_TIME || ngx_event_timer_alarm) {
        ngx_time_update();
    }

    if (err) {
        if (err == NGX_EINTR) {

            if (ngx_event_timer_alarm) {
                ngx_event_timer_alarm = 0;
                return NGX_OK;
            }

            level = NGX_LOG_INFO;

        } else {
            level = NGX_LOG_ALERT;
        }

        ngx_log_error(level, cycle->log, err, "epoll_wait() failed");
        return NGX_ERROR;
    }

    if (events == 0) {
        if (timer != NGX_TIMER_INFINITE) {
            return NGX_OK;
        }

        ngx_log_error(NGX_LOG_ALERT, cycle->log, 0,
                      "epoll_wait() returned no events without timeout");
        return NGX_ERROR;
    }

    for (i = 0; i < events; i++) {
        c = event_list[i].data.ptr;

        instance = (uintptr_t) c & 1;
        c = (ngx_connection_t *) ((uintptr_t) c & (uintptr_t) ~1);

        rev = c->read;

        if (c->fd == -1 || rev->instance != instance) {

            /*
             * the stale event from a file descriptor
             * that was just closed in this iteration
             */

            ngx_log_debug1(NGX_LOG_DEBUG_EVENT, cycle->log, 0,
                           "epoll: stale event %p", c);
            continue;
        }

        revents = event_list[i].events;

        ngx_log_debug3(NGX_LOG_DEBUG_EVENT, cycle->log, 0,
                       "epoll: fd:%d ev:%04XD d:%p",
                       c->fd, revents, event_list[i].data.ptr);

        if (revents & (EPOLLERR|EPOLLHUP)) {
            ngx_log_debug2(NGX_LOG_DEBUG_EVENT, cycle->log, 0,
                           "epoll_wait() error on fd:%d ev:%04XD",
                           c->fd, revents);

            /*
             * if the error events were returned, add EPOLLIN and EPOLLOUT
             * to handle the events at least in one active handler
             */

            revents |= EPOLLIN|EPOLLOUT;
        }

#if 0
        if (revents & ~(EPOLLIN|EPOLLOUT|EPOLLERR|EPOLLHUP)) {
            ngx_log_error(NGX_LOG_ALERT, cycle->log, 0,
                          "strange epoll_wait() events fd:%d ev:%04XD",
                          c->fd, revents);
        }
#endif

        if ((revents & EPOLLIN) && rev->active) {

#if (NGX_HAVE_EPOLLRDHUP)
            if (revents & EPOLLRDHUP) {
                rev->pending_eof = 1;
            }

            rev->available = 1;
#endif

            rev->ready = 1;

            if (flags & NGX_POST_EVENTS) {
                queue = rev->accept ? &ngx_posted_accept_events
                                    : &ngx_posted_events;

                ngx_post_event(rev, queue);

            } else {
                rev->handler(rev);
            }
        }

        wev = c->write;

        if ((revents & EPOLLOUT) && wev->active) {

            if (c->fd == -1 || wev->instance != instance) {

                /*
                 * the stale event from a file descriptor
                 * that was just closed in this iteration
                 */

                ngx_log_debug1(NGX_LOG_DEBUG_EVENT, cycle->log, 0,
                               "epoll: stale event %p", c);
                continue;
            }

            wev->ready = 1;
#if (NGX_THREADS)
            wev->complete = 1;
#endif

            if (flags & NGX_POST_EVENTS) {
                ngx_post_event(wev, &ngx_posted_events);

            } else {
                wev->handler(wev);
            }
        }
    }

    return NGX_OK;
}

```


