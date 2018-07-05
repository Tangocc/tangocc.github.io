---
layout:     post
title:      "NGINX系列之内存池"
subtitle:   ""
date:       2018-06-18 12:00:00
author:     "Tango"
header-img: "img/in-post/post-2017-10-11/post-bg-universe.jpg"
catalog: true
tags:   
    - nginx
    - 内存池
---


## Nginx内存池 `ngx_pool_t`
Nginx作为高性能到web服务器，自然需要满足高效的内存使用率和分配效率。
内存池满足上述两个要求，主要优点：

- 统一内存管理，避免内存碎片化，提高系统到使用率（Nginx做内存对齐处理，牺牲一定到使用率换来寻址速率）
- 避免多次向系统申请内存(涉及内核态和用户态转换)，提高内存到分配效率；
- 内存统一分配和销毁，避免内存泄露。

---

### 数据结构 `ngx_pool_t`
其数据结构如下所示：

```
typedef struct ngx_pool_large_s  ngx_pool_large_t;
typedef struct ngx_pool_s        ngx_pool_t;  
//回调函数
struct ngx_pool_cleanup_s {
    ngx_pool_cleanup_pt   handler;
    void                 *data;
    ngx_pool_cleanup_t   *next;
};

//大内存块
struct ngx_pool_large_s {
    ngx_pool_large_t     *next;//下一数据块地址
    void                 *alloc;//当前数据块地址
};

//内存块(内存池中小内存分配位置)
typedef struct {
    u_char               *last;
    u_char               *end;
    ngx_pool_t           *next;
    ngx_uint_t            failed;
} ngx_pool_data_t;

//内存池结构
struct ngx_pool_s {
    ngx_pool_data_t       d;//内存池的数据区，即真正分配内存的区域
    size_t                max;//每次可分配最大内存，用于判定是走大内存分配还是小内存分配逻辑
    ngx_pool_t           *current;//指向当前内存块
    ngx_chain_t          *chain;//缓冲区链表
    ngx_pool_large_t     *large;//内存块大于max内存块的链表
    ngx_pool_cleanup_t   *cleanup;//销毁内存池回调函数
    ngx_log_t            *log;//日志
};

```
其内存结构图如图所示：  

![](/img/in-post/post-2018-06-18-nginx-pool.png)

### 核心函数
---


#### 创建内存池 `ngx_create_pool(size_t size, ngx_log_t * log)`
```

/*
 * NGX_MAX_ALLOC_FROM_POOL should be (ngx_pagesize - 1), i.e. 4095 on x86.
 * On Windows NT it decreases a number of locked pages in a kernel.
 */
#define NGX_MAX_ALLOC_FROM_POOL  (ngx_pagesize - 1)
#define NGX_DEFAULT_POOL_SIZE    (16 * 1024)
#define NGX_POOL_ALIGNMENT       16
/**
 * 创建一个内存池
 *  * 1.调用系统内存分配函数，分配一块内存，并做内存对齐
 *  * 2.指针赋值
 *  * 3.设定一次可分配内存大小
 */
ngx_pool_t * ngx_create_pool(size_t size, ngx_log_t * log)
{
    ngx_pool_t * p;
    //调用系统内存分配函数，分配一块内存，且做对齐处理
    p = ngx_memalign(NGX_POOL_ALIGNMENT, size, log);
    if (p == NULL) {
        return NULL;
    }
    //指针赋值
    p->d . last = (u_char *) p + sizeof(ngx_pool_t);
    p->d . end = (u_char *) p + size;
    p->d . next = NULL;
    p->d . failed = 0;

    size = size - sizeof(ngx_pool_t);
    //NGX_MAX_ALLOC_FROM_POOL = ngx_pagesize - 1( 内存页的大小)
    //即是一次内存分配最大值是内存页的大小
    p->max = (size < NGX_MAX_ALLOC_FROM_POOL) ? size : NGX_MAX_ALLOC_FROM_POOL;

    p->current = p;
    p->chain = NULL;
    p->large = NULL;
    p->cleanup = NULL;
    p->log = log;

    return p;
}
```

#### 内存池申请内存 `ngx_palloc(ngx_pool_t * pool, size_t size)`

```

/**
 * 内存池中申请一块内存
 *   * 如果申请内存小于定义的max则从内存池中分配内存
 *   * 否则分配大块内存挂载在large指针上
 */
void * ngx_palloc(ngx_pool_t * pool, size_t size)
{
#if !(NGX_DEBUG_PALLOC)
    if (size <= pool->max) {
    return ngx_palloc_small(pool, size, 1);
}
#endif

    return ngx_palloc_large(pool, size);
}
```

#### 申请小内存 `ngx_palloc_small(ngx_pool_t * pool, size_t size, ngx_uint_t align)`
```
/**
 * 分配小块内存(小于max参数指定大小)
 *  * 1.循环遍历内存块ngx_pool_data_t链表
 *  * 2.如果当前内存块大小大于申请到内存大小，则移动相应指针，返回分配的内存指针
 *  * 3.否则，移动到下一个内存块
 *  * 4.如果所有内存块都满足不了申请的内存，则重新申请一块内存块挂载到链表上，并返回分配的内存地址
 */

static ngx_inline void * ngx_palloc_small(ngx_pool_t * pool, size_t size, ngx_uint_t align)
{
    u_char * m;
    ngx_pool_t * p;

    p = pool->current;
   //1.循环遍历内存块ngx_pool_data_t链表
    do {
        m = p->d . last;

        if (align) {
            m = ngx_align_ptr(m, NGX_ALIGNMENT);
        }
        //2.如果当前内存块大小大于申请到内存大小，则移动相应指针，返回分配的内存指针
        if ((size_t) (p->d . end - m) >= size) {
            p->d . last = m + size;

            return m;
        }
        //3.否则，移动到下一个内存块
        p = p->d . next;

    } while (p);
    //4.如果所有内存块都满足不了申请的内存，则重新申请一块内存块挂载到链表上，并返回分配的内存地址
    return ngx_palloc_block(pool, size);
}

```

#### 申请内存块  `ngx_palloc_block(ngx_pool_t * pool, size_t size)`

```
/**
 * 分配内存块ngx_pool_t
 */
static void * ngx_palloc_block(ngx_pool_t * pool, size_t size)
{
    u_char * m;
    size_t       psize;
    ngx_pool_t * p, *new;

    psize = (size_t) (pool->d . end - (u_char *) pool);

    m = ngx_memalign(NGX_POOL_ALIGNMENT, psize, pool->log);
    if (m == NULL) {
        return NULL;
    }

    new = (ngx_pool_t *) m;

    new->d . end = m + psize;
    new->d . next = NULL;
    new->d . failed = 0;

    m += sizeof(ngx_pool_data_t);
    m = ngx_align_ptr(m, NGX_ALIGNMENT);
    new->d . last = m + size;
    //调整各个内存块分配失败次数，如果失败次数大于4，则此块内存将不使用
    for (p = pool->current; p->d . next; p = p->d . next) {
    if (p->d . failed++ > 4) {
        pool->current = p->d . next;
        }
    }

    p->d . next = new;

    return m;
}


```
#### 申请大内存块 `ngx_palloc_large(ngx_pool_t * pool, size_t size)`
```
/**
 * 分配大内存块
 */
static void * ngx_palloc_large(ngx_pool_t * pool, size_t size)
{
    void * p;
    ngx_uint_t         n;
    ngx_pool_large_t * large;

    p = ngx_alloc(size, pool->log);
    if (p == NULL) {
        return NULL;
    }

    n = 0;

    for (large = pool->large; large; large = large->next) {
    if (large->alloc == NULL) {
        large->alloc = p;
            return p;
        }

        if (n++ > 3) {
            break;
        }
    }

    large = ngx_palloc_small(pool, sizeof(ngx_pool_large_t), 1);
    if (large == NULL) {
        ngx_free(p);
        return NULL;
    }

    large->alloc = p;
    large->next = pool->large;
    pool->large = large;

    return p;
}


```





