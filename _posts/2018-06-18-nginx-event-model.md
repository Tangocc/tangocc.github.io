---
layout:     post
title:      "NGINX系列之事件模型"
subtitle:   ""
date:       2018-06-18 12:00:00
author:     "Tango"
header-img: "img/post-bg-universe.jpg"
catalog: true
tags:   
    - nginx
    - 内存池
---

nginx在完成进程的创建后，主进程进入信号处理的循环中，不参与事件处理；worker进程则进入事件处理过程。nginx任何操作，包括定时任务、连接、读写等都可以定义为事件，事件具有特点是被动特性，即发生才处理，避免因为轮训状态而导致的时间消耗。本章将介绍nginx的事件处理模型。

### 伪代码

为了方便理解，首先给出工作进程执行函数的伪代码：
```
void ngx_worker_process_cycle(){

   nginx_worker_process_init();
   
   while(1){
      
      ngx_worker_signal_handler();
      
      /*以下为ngx_process_events_and_timers函数*/
      ngx_event_find_timer();
      ngx_event_process_timer();
      //判断是否获得accept_mutex锁，决定是否执行accept事件。
      if(ngx_trylock_accept_mutex()){
         ngx_process_accept_events();
         ngx_unlock_accept_mutex();
      }
      ngx_process_read_events();
      ngx_process_write_events();
      ngx_event_expire_timers();
      /*以上为ngx_process_events_and_timers函数*/
   }
}


```
工作进程完成初始化以后，进入时间处理模块，首先判断是否有来自主进程信号(退出？重启？)并进行相应处理，然后依次处理accept事件、读事件、写事件。先处理accept事件是为了尽快释放

上一章节中提出nginx如何实现定时任务？如何确保定时的准确性？
nginx将定时任务存储在RBTree红黑树中，每次处理任务之前从红黑树中取出距离目前最近的一次定时任务，假设时间为delta




### ngx_process_events_and_timers

上章分析中，当进程创建后，子进程进入核心事件处理函数`ngx_process_events_and_timers `,在该函数中进行定时器以及读写事件的处理：
首先，解析是否有定时器任务，如果没有则将定时时间设为∞，如果有定时任务

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
        timer = ngx_event_find_timer();
        flags = NGX_UPDATE_TIME;

#if (NGX_WIN32)

        /* handle signals from master in case of network inactivity*/
        if (timer == NGX_TIMER_INFINITE || timer > 500) {
            timer = 500;
        }

#endif
    }

    if (ngx_use_accept_mutex) {
        if (ngx_accept_disabled > 0) {
            ngx_accept_disabled--;

        } else {
            if (ngx_trylock_accept_mutex(cycle) == NGX_ERROR) {
                return;
            }

            if (ngx_accept_mutex_held) {
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

    delta = ngx_current_msec;

    (void) ngx_process_events(cycle, timer, flags);

    delta = ngx_current_msec - delta;

    ngx_event_process_posted(cycle, &ngx_posted_accept_events);

    if (ngx_accept_mutex_held) {
        ngx_shmtx_unlock(&ngx_accept_mutex);
    }

    if (delta) {
        ngx_event_expire_timers();
    }

    ngx_event_process_posted(cycle, &ngx_posted_events);
}

```



```

void
ngx_event_process_posted(ngx_cycle_t *cycle, ngx_queue_t *posted)
{
    ngx_queue_t  *q;
    ngx_event_t  *ev;

    while (!ngx_queue_empty(posted)) {

        q = ngx_queue_head(posted);
        ev = ngx_queue_data(q, ngx_event_t, queue);

        ngx_log_debug1(NGX_LOG_DEBUG_EVENT, cycle->log, 0,
                      "posted event %p", ev);

        ngx_delete_posted_event(ev);

        ev->handler(ev);
    }
}


```



```

ngx_msec_t
ngx_event_find_timer(void)
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


void
ngx_event_expire_timers(void)
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

        ngx_log_debug2(NGX_LOG_DEBUG_EVENT, ev->log, 0,
                       "event timer del: %d: %M",
                       ngx_event_ident(ev->data), ev->timer.key);

        ngx_rbtree_delete(&ngx_event_timer_rbtree, &ev->timer);

#if (NGX_DEBUG)
        ev->timer.left = NULL;
        ev->timer.right = NULL;
        ev->timer.parent = NULL;
#endif

        ev->timer_set = 0;

        ev->timedout = 1;

        ev->handler(ev);
    }
}

```