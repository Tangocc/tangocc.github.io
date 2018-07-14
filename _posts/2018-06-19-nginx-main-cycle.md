---
layout:     post
title:      "NGINX系列之MAIN函数"
subtitle:   ""
date:       2018-03-20 12:00:00
author:     "Tango"
header-img: "img/post-bg-universe.jpg"
catalog: true
tags:   
    - nginx
    - 系统架构
    - 主流程
---

近期在系统学习Nginx相关源码，针对nginx进程模型、事件处理模型、配置以及扩展开发等诸多方面希望能够沉淀一些东西，故针对上述方面整理一系列博客，既然是源码分析，那就首先从main函数开始吧。

### 核心数据结构

- 全局变量cycle数据结构  
`ngx_cycle_s `变量是nginx中贯穿始终的全局变量，其存储在系统运行过程中的所有信息，包括配置文件信息、模块信息、客户端连接、读写事件处理函数等信息。其结构如下所示： 

```
struct ngx_cycle_s {
    void                  ****conf_ctx;
    ngx_pool_t               *pool;
    ngx_log_t                *log;
    ngx_log_t                 new_log;
    ngx_uint_t                log_use_stderr;  /* unsigned  log_use_stderr:1; */

    ngx_connection_t        **files;
    ngx_connection_t         *free_connections;
    ngx_uint_t                free_connection_n;

    ngx_module_t            **modules;
    ngx_uint_t                modules_n;
    ngx_uint_t                modules_used;    /* unsigned  modules_used:1; */

    ngx_queue_t               reusable_connections_queue;
    ngx_uint_t                reusable_connections_n;

    ngx_array_t               listening;
    ngx_array_t               paths;

    ngx_array_t               config_dump;
    ngx_rbtree_t              config_dump_rbtree;
    ngx_rbtree_node_t         config_dump_sentinel;

    ngx_list_t                open_files;
    ngx_list_t                shared_memory;

    ngx_uint_t                connection_n;
    ngx_uint_t                files_n;

    ngx_connection_t         *connections;
    ngx_event_t              *read_events;
    ngx_event_t              *write_events;

    ngx_cycle_t              *old_cycle;

    ngx_str_t                 conf_file;
    ngx_str_t                 conf_param;
    ngx_str_t                 conf_prefix;
    ngx_str_t                 prefix;
    ngx_str_t                 lock_file;
    ngx_str_t                 hostname;
};
```
- Main核心配置数据结构

```
//核心配置
typedef struct {
    ngx_flag_t                daemon;
    ngx_flag_t                master;

    ngx_msec_t                timer_resolution;
    ngx_msec_t                shutdown_timeout;

    ngx_int_t                 worker_processes; 
    ngx_int_t                 debug_points;

    ngx_int_t                 rlimit_nofile;
    off_t                     rlimit_core;

    int                       priority;

    ngx_uint_t                cpu_affinity_auto;
    ngx_uint_t                cpu_affinity_n;
    ngx_cpuset_t             *cpu_affinity;

    char                     *username;
    ngx_uid_t                 user;
    ngx_gid_t                 group;

    ngx_str_t                 working_directory;
    ngx_str_t                 lock_file;

    ngx_str_t                 pid;
    ngx_str_t                 oldpid;

    ngx_array_t               env;
    char                    **environment;

    ngx_uint_t                transparent;  /* unsigned  transparent:1; */
} ngx_core_conf_t;
```

- 模块数据结构  

```
//模块数据结构
struct ngx_module_s {
    ngx_uint_t            ctx_index;
    ngx_uint_t            index;

    char                 *name;

    ngx_uint_t            spare0;
    ngx_uint_t            spare1;

    ngx_uint_t            version;
    const char           *signature;

    void                 *ctx;
    ngx_command_t        *commands;
    ngx_uint_t            type;

    ngx_int_t           (*init_master)(ngx_log_t *log);

    ngx_int_t           (*init_module)(ngx_cycle_t *cycle);

    ngx_int_t           (*init_process)(ngx_cycle_t *cycle);
    ngx_int_t           (*init_thread)(ngx_cycle_t *cycle);
    void                (*exit_thread)(ngx_cycle_t *cycle);
    void                (*exit_process)(ngx_cycle_t *cycle);

    void                (*exit_master)(ngx_cycle_t *cycle);

    uintptr_t             spare_hook0;
    uintptr_t             spare_hook1;
    uintptr_t             spare_hook2;
    uintptr_t             spare_hook3;
    uintptr_t             spare_hook4;
    uintptr_t             spare_hook5;
    uintptr_t             spare_hook6;
    uintptr_t             spare_hook7;
};


typedef struct {
    ngx_str_t             name;
    void               *(*create_conf)(ngx_cycle_t *cycle);
    char               *(*init_conf)(ngx_cycle_t *cycle, void *conf);
} ngx_core_module_t;

```

### 主函数 main()

- 文件位置：`/src/nginx.c `

```
int ngx_cdecl main(int argc, char *const *argv)
{
    ngx_buf_t        *b;
    ngx_log_t        *log;
    ngx_uint_t        i;
    ngx_cycle_t      *cycle, init_cycle;
    ngx_conf_dump_t  *cd;
    ngx_core_conf_t  *ccf;

    ngx_debug_init();

    // 1.获取命令行参数
    ngx_get_options(argc, argv)
    // 2.初始化本地时钟
    ngx_time_init();
    ngx_pid = ngx_getpid();
    ngx_parent = ngx_getppid();

    // 3.初始化日志
    log = ngx_log_init(ngx_prefix);
    
    // 4.初始化全局变量ngx_cycle_t   
    ngx_memzero(&init_cycle, sizeof(ngx_cycle_t));
    init_cycle.log = log;
    ngx_cycle = &init_cycle;
    init_cycle.pool = ngx_create_pool(1024, log);

    // 5.保存命令行参数到全局变量中
    ngx_save_argv(&init_cycle, argc, argv);
    // 6.处理命令行参数
    ngx_process_options(&init_cycle);
    // 7.初始化系统相关参数
    ngx_os_init(log);

    // 8.预初始化模块
    ngx_preinit_modules();
    
    // 9.
    cycle = ngx_init_cycle(&init_cycle);
    ngx_os_status(cycle->log);
    ngx_cycle = cycle;
    // 9.获取核心模块配置参数
    ccf = (ngx_core_conf_t *) ngx_get_conf(cycle->conf_ctx, ngx_core_module);
    // 10.创建pid存储文件
    ngx_create_pidfile(&ccf->pid, cycle->log);
    ngx_log_redirect_stderr(cycle);
    // 11.单进程模式运行
    if (ngx_process == NGX_PROCESS_SINGLE) {
        ngx_single_process_cycle(cycle);
    } else {
    // 12.多进程模式运行
        ngx_master_process_cycle(cycle);
    }

    return 0;
}

```


### 预初始化模块 `ngx_preinit_modules()`
- 文件位置 `/src/ngx_module.c`
- 主要工作：配置各个模块的顺序索引

```
ngx_int_t ngx_preinit_modules(void)
{
    ngx_uint_t  i;
	//ngx_modules是全局变量，定义在nginx_modules.h文件中
    for (i = 0; ngx_modules[i]; i++) {
        ngx_modules[i]->index = i;
        ngx_modules[i]->name = ngx_module_names[i];
    }

    ngx_modules_n = i;
    ngx_max_module = ngx_modules_n + NGX_MAX_DYNAMIC_MODULES;

    return NGX_OK;
}

```

`ngx_preinit_modules()`函数主要完成模块到序号初始化，其数组`ngx_modules`是何时产生的呢？  

- `ngx_modules`包含所有nginx定义的模块,其初始化是由执行configure命令前定义.
- 新增模块或者减少模块可以在configure命令执行前 auto/modules文件里面修改。

### `ngx_init_cycle`
- 文件位置 `/src/ngx_module.c`
- 主要工作：初始化系统时钟、主机名、模块配置文件、各种后续数据结构，形成全局变量cycle

```C

ngx_cycle_t * ngx_init_cycle(ngx_cycle_t *old_cycle)
{
    void                *rv;
    char               **senv;
    ngx_uint_t           i, n;
    ngx_log_t           *log;
    ngx_time_t          *tp;
    ngx_conf_t           conf;
    ngx_pool_t          *pool;
    ngx_cycle_t         *cycle, **old;
    ngx_shm_zone_t      *shm_zone, *oshm_zone;
    ngx_list_part_t     *part, *opart;
    ngx_open_file_t     *file;
    ngx_listening_t     *ls, *nls;
    ngx_core_conf_t     *ccf, *old_ccf;
    ngx_core_module_t   *module;
    char                 hostname[NGX_MAXHOSTNAMELEN];

	 // 1.更新时钟和时区
    ngx_timezone_update();
    ngx_time_update();


    log = old_cycle->log;
    pool = ngx_create_pool(NGX_CYCLE_POOL_SIZE, log);
    pool->log = log;

    cycle = ngx_pcalloc(pool, sizeof(ngx_cycle_t));
    cycle->pool = pool;
    cycle->log = log;
    cycle->old_cycle = old_cycle;
    // 2.初始化相关数据结构
    ngx_array_init(&cycle->paths, pool, n, sizeof(ngx_path_t *);
    ngx_memzero(cycle->paths.elts, n * sizeof(ngx_path_t *);
    ngx_array_init(&cycle->config_dump, pool, 1, sizeof(ngx_conf_dump_t);
    ngx_rbtree_init(&cycle->config_dump_rbtree, &cycle->config_dump_sentinel,ngx_str_rbtree_insert_value);
    ngx_list_init(&cycle->open_files, pool, n, sizeof(ngx_open_file_t);
    ngx_list_init(&cycle->shared_memory, pool, n, sizeof(ngx_shm_zone_t);
    ngx_array_init(&cycle->listening, pool, n, sizeof(ngx_listening_t);
    ngx_memzero(cycle->listening.elts, n * sizeof(ngx_listening_t));
    ngx_queue_init(&cycle->reusable_connections_queue);
    cycle->conf_ctx = ngx_pcalloc(pool, ngx_max_module * sizeof(void *));

	// 3.获取主机名
    gethostname(hostname, NGX_MAXHOSTNAMELEN) ;
    hostname[NGX_MAXHOSTNAMELEN - 1] = '\0';
    cycle->hostname.len = ngx_strlen(hostname);
    cycle->hostname.data = ngx_pnalloc(pool, cycle->hostname.len);
    // 4.创建modules数组
    ngx_cycle_modules(cycle);

    // 5.创建核心模块到conf
    for (i = 0; cycle->modules[i]; i++) {
        if (cycle->modules[i]->type != NGX_CORE_MODULE) {
            continue;
        }
        module = cycle->modules[i]->ctx;
        if (module->create_conf) {
            rv = module->create_conf(cycle);
           cycle->conf_ctx[cycle->modules[i]->index] = rv;
        }
    }


    ngx_memzero(&conf, sizeof(ngx_conf_t));
    conf.ctx = cycle->conf_ctx;
    conf.cycle = cycle;
    conf.pool = pool;
    conf.log = log;
    conf.module_type = NGX_CORE_MODULE;
    conf.cmd_type = NGX_MAIN_CONF;
    ngx_conf_param(&conf);     
    ngx_conf_parse(&conf, &cycle->conf_file)
    
    // 6.调用核心模块的init_conf回调函数 
     for (i = 0; cycle->modules[i]; i++) {
        if (cycle->modules[i]->type != NGX_CORE_MODULE) {
            continue;
        }
        module = cycle->modules[i]->ctx;
        if (module->init_conf) {
            module->init_conf(cycle,cycle->conf_ctx[cycle->modules[i]->index])
        }
    }

    // 7.单进程模式则直接返回
    if (ngx_process == NGX_PROCESS_SIGNALLER) {
        return cycle;
    }
    // 8.多进程模式则创建pid文件
    ccf = (ngx_core_conf_t *) ngx_get_conf(cycle->conf_ctx, ngx_core_module);
    ngx_create_pidfile(&ccf->pid, log);
    ngx_open_listening_sockets(cycle) 
    
    ngx_init_modules(cycle);

    return cycle;

  }

```
### `ngx_cycle_modules`
- 文件位置：
- 主要工作：生成modules数组

```
ngx_int_t ngx_cycle_modules(ngx_cycle_t *cycle)
{
    /*
     * create a list of modules to be used for this cycle,
     * copy static modules to it
     */

    cycle->modules = ngx_pcalloc(cycle->pool, (ngx_max_module + 1)
                                              * sizeof(ngx_module_t *));
    if (cycle->modules == NULL) {
        return NGX_ERROR;
    }

    ngx_memcpy(cycle->modules, ngx_modules,
               ngx_modules_n * sizeof(ngx_module_t *));

    cycle->modules_n = ngx_modules_n;

    return NGX_OK;
}

```


### `ngx_init_modules `

- 文件位置
- 主要工作：回调各个模块初始化函数

```
ngx_int_t ngx_init_modules(ngx_cycle_t *cycle)
{
    ngx_uint_t  i;

    for (i = 0; cycle->modules[i]; i++) {
        if (cycle->modules[i]->init_module) {
            if (cycle->modules[i]->init_module(cycle) != NGX_OK) {
                return NGX_ERROR;
            }
        }
    }

    return NGX_OK;
}

```
在完成模块创建和初始化工作后，系统进入`ngx_single_process_cycle `和`ngx_master_process_cycle `函数，创建主进程和工作进程，其具体的进程模型将在文章[NGINX系列之进程模型](https://tangocc.github.io/2018/03/20/nginx-process-model/)中介绍。