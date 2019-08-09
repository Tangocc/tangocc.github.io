---
layout:     post
title:      "MySql系列之源码浅析"
subtitle:   ""
date:       2018-10-11 12:00:00
author:     "Tango"
header-img: "img/post-bg-universe.jpg"
catalog: true
tags:   
    - MySql
    - 数据库 
---


> 源码才是王道。  
>

### 1.MySQL源码

---
#### **1. 主函数sql/mysqld.cc中，代码如下：**

```
//标准入口函数
int main(int argc, char **argv) 
{
    //调用mysys/My_init.c->my_init()，初始化mysql内部的系统库
    MY_INIT(argv[0]);
    //初始化日志功能
    logger.init_base(); 
    //读取配置文件信息
    load_defaults(conf_file_name, groups, &argc, &argv);
    //检测启动时的用户选项 
    user_info = check_user(mysqld_user);
    //设置以该用户运行
    set_user(mysqld_user, user_info);
    //初始化内部的一些组件，如table_cache, query_cache等。
    init_server_components();
    //初始化网络模块，创建socket监听
    network_init();
    // 创建pid文件
    start_signal_handler();
     // 初始化mysql中的status变量
    init_status_vars();
    //创建manager线程
    start_handle_manager();
    //主要处理函数，处理新的连接并创建新的线程处理
    handle_connections_sockets();
}
```
#### **2.监听连接: sql/mysqld.cc - handle_connections_sockets:**

```
pthread_handler_t handle_connections_sockets(void *arg __attribute__((unused))) {
    // unix_socket在network_init中被打开
    FD_SET(unix_sock,&clientFDs); 
    // abort_loop是全局变量，在某些情况下被置为1表示要退出。
    while (!abort_loop) { 
        // 需要监听的socket
        readFDs=clientFDs; 
        // select异步监听
        select((int) max_used_connection,&readFDs,0,0,0); 
         // 接受请求
        new_sock = accept(sock, my_reinterpret_cast(struct sockaddr *)(&cAddr),&length);
        // 创建mysqld任务线程描述符，它封装了一个客户端连接请求的所有信息
        thd= new THD; 
        // 网络操作抽象层
        vio_tmp=vio_new(new_sock, VIO_TYPE_SOCKET, VIO_LOCALHOST); 
        // 初始化任务线程描述符的网络操作
        my_net_init(&thd->net,vio_tmp)); 
        // 创建任务线程
        create_new_thread(thd); 
    }
}

```

#### **3. 创建连接 sql/mysqld.cc  create_new_thread/create_thread_to_handle_connection:**

```
static void create_new_thread(THD *thd) {
    NET *net=&thd->net;
    // 看看当前连接数是不是超过了系统配置允许的最大值，如果是就断开连接。
    if (connection_count >= max_connections + 1 || abort_loop) { 
        close_connection(thd, ER_CON_COUNT_ERROR, 1);
        delete thd;
    }
    ++connection_count;
    // 将新连接加入到thread_scheduler的连接队列中。
    thread_scheduler.add_connection(thd); 
}
```

#### **4. 线程调度器thread_scheduler - create_thread_to_handle_connection**

```
void create_thread_to_handle_connection(THD *thd) {
    //看当前工作线程缓存(thread_cache)中有否空余的线程
    if (cached_thread_count > wake_thread) { 
      thread_cache.append(thd);
       // 有的话则唤醒一个线程来用
      pthread_cond_signal(&COND_thread_cache);
   } else {
      threads.append(thd);
      //没有可用空闲线程则创建一个新的线程
      pthread_create(&thd->real_id,&connection_attrib,   handle_one_connection,   (void*) thd))); 
   }
}

```

#### **5.handle_one_connection**
```
pthread_handler_t handle_one_connection(void *arg) {
    // 初始化线程预处理操作
    thread_scheduler.init_new_connection_thread(); 
    //载入一些Session级变量
    setup_connection_thread_globals(thd); 
    for (;;) { 
        //初始化LEX词法解析器
        lex_start(thd); 
        // 进行连接身份验证
        login_connection(thd); 
        // 初始化线程Status,即show status看到的
        prepare_new_connection_state(thd); 
        // 处理命令
        do_command(thd); 
        //没事做了关闭连接,丢入线程池
        end_connection(thd); 
    }
}

```

#### **6.执行语句 sql/sql_parse.cc - do_command函数**

```
bool do_command(THD *thd) {
    NET *net= &thd->net;
    //读取客户端发送的报文
    packet_length = my_net_read(net);
    packet = (char*) net->read_pos;
    // 解析客户端传过来的命令类型
    command = (enum enum_server_command) (uchar) packet[0]; 
    // 根据命令类型进行分发
    dispatch_command(command, thd, packet+1, (uint) (packet_length-1));
}

```

####  **7.指令分发 sql/sql_parse.cc定义dispatch_command**

```
bool dispatch_command(enum enum_server_command command, THD *thd, char* packet, uint packet_length) {
    NET *net = &thd->net;
    thd->command = command; 
    switch (command) { //判断命令类型
        case COM_INIT_DB: ...;
        case COM_TABLE_DUMP: ...;
        case COM_CHANGE_USER: ...;
        ...
        case COM_QUERY: //如果是Query
            alloc_query(thd, packet, packet_length); //从网络数据包中读取Query并存入thd->query
            mysql_parse(thd, thd->query, thd->query_length, &end_of_stmt); //送去解析
    }
}

```


#### **8.sql/sql_parse.cc mysql_parse函数负责解析SQL**

```
void mysql_parse(THD *thd, const char *inBuf, uint length, const char ** found_semicolon) {
    //初始化线程解析描述符
    lex_start(thd); 
    // 看query cache中有否命中，有就直接返回结果，否则进行查找
    if (query_cache_send_result_to_client(thd, (char*) inBuf, length) <= 0) { 
        Parser_state parser_state(thd, inBuf, length);  
        // 解析SQL语句 
        parse_sql(thd, & parser_state, NULL); 
        // 执行语句
        mysql_execute_command(thd); 
    }
}

```

#### **9.执行命令 mysql_execute_command**

```

int mysql_execute_command(THD *thd) {
    // 解析过后的SQL语句的语法结构
    LEX  *lex= thd->lex;  
    // 该语句要访问的表的列表
    TABLE_LIST *all_tables = lex->query_tables;   
    switch (lex->sql_command) {
        ...
        case SQLCOM_INSERT://insert语句
            insert_precheck(thd, all_tables);
            mysql_insert(thd, all_tables, lex->field_list, lex->many_values, lex->update_list, lex->value_list, lex->duplicates, lex->ignore);
            break; ... 
        case SQLCOM_SELECT: // select 语句
             // 检查用户对数据表的访问权限
            check_table_access(thd, lex->exchange ? SELECT_ACL | FILE_ACL :  SELECT_ACL,  all_tables, UINT_MAX, FALSE);   
            // 执行select语句
            execute_sqlcom_select(thd, all_tables);     
            break;
    }
}

```
#### **10.接下来sql/sql_insert.cc中mysql_insert函数**
```
bool mysql_insert(THD *thd,
                   TABLE_LIST *table_list,      // 该INSERT要用到的表
                   List<Item> &fields,             // 使用的项
                   ....) {
     // 这里的锁只是防止表结构修改
    open_and_lock_tables(thd, table_list); 
    mysql_prepare_insert(...);
    //里面还有trigger，错误，view之类的杂七杂八的东西，我们都忽略
    foreach value in values_list {
        write_record(...);
    }
} 
```

#### **11.接着看真正写数据的函数write_record (在sql/sql_insert.cc),精简代码如下:**
```
int write_record(THD *thd, TABLE *table,COPY_INFO *info) {  // 写数据记录
    //如果是REPLACE或UPDATE则替换数据
    if (info->handle_duplicates == DUP_REPLACE || info->handle_duplicates == DUP_UPDATE) { 
        table->file->ha_write_row(table->record[0]);
        table->file->ha_update_row(table->record[1], table->record[0]));
    } else {
        table->file->ha_write_row(table->record[0]);
    }
}
//存储引擎抽象的Handler API !
int handler::ha_write_row(uchar *buf) { 
    write_row(buf);   // 调用具体的实现
    binlog_log_row(table, 0, buf, log_func));  // 写binlog
}

```

---
### 2.请求数据流


![](/img/in-post/mysql/post-mysql-data-stream.jpg)
