---
layout:     post
title:      "NGINX系列之HTTP模块开发"
subtitle:   ""
date:       2018-07-05 12:00:00
author:     "Tango"
header-img: "img/post-bg-universe.jpg"
catalog: true
tags:   
    - nginx
    - 系统架构
---

**上文介绍nginx请求11个阶段处理，本文将动手实操开发并注册一个HTTP模块，在实现`ngx_http_hello_world_module `模块的过程中，详细介绍其实现步骤。**

## 模块组成介绍
### 模块定义
- 模块名称
> `ngx_http_hello_world_module`

- 模块功能
> 功能相对简单，从`Hello World`开始，通过实现扩展HTTP模块`ngx_http_hello_world_module `，实现在响应头中增加一个字段`X-Hello-World:XXX`功能。
- 模块使用
> 修改nginx.conf文件，在location块中添加命令 `X-Hello-World ` 其中`X-Hello-World`是命令，`XXX`是值。


### 模块开发步骤
1.编写模块基本结构。包括模块的定义，模块上下文结构，模块的配置结构等。  
2.实现handler的挂载函数，根据模块的需求选择正确的挂载方式。  
3.编写模块核心功能实现函数handler处理函数。  
4.编写编译的文件config  
5.将实现的模块添加到nginx  
6.修改配置文件，验证可行性。  

### 模块开发
#### 定义模块命令结构
首先，`ngx_command_s `结构体定义在`src/core/ngx_conf_file.h`文件中，其结构如下所示:  

```
struct ngx_command_s {
    ngx_str_t             name;
    ngx_uint_t            type;
    char               *(*set)(ngx_conf_t *cf, ngx_command_t *cmd, void *conf);
    ngx_uint_t            conf;
    ngx_uint_t            offset;
    void                  *post;
};

```

- name  
 配置指令的名称

- type  
配置指令的类型。  
nginx提供了很多预定义的属性值（一些宏定义），通过逻辑或运算符可组合在一起，形成对这个配置指令的详细的说明。  
下面列出可在这里使用的预定义属性值及说明。  
`NGX_CONF_NOARGS`：配置指令不接受任何参数。  
`NGX_CONF_TAKE1`：配置指令接受1个参数。  
`NGX_CONF_TAKE2`：配置指令接受2个参数。  
`NGX_CONF_TAKE3`：配置指令接受3个参数。  
`NGX_CONF_TAKE4`：配置指令接受4个参数。  
`NGX_CONF_TAKE5`：配置指令接受5个参数。  
`NGX_CONF_TAKE6`：配置指令接受6个参数。  
`NGX_CONF_TAKE7`：配置指令接受7个参数。  

可以组合多个属性，比如一个指令即可以不填参数，也可以接受1个或者2个参数。那么就是 `NGX_CONF_NOARGS | NGX_CONF_TAKE1 |NGX_CONF_TAKE2`。如果写上面三个属性在一起，你觉得麻烦，那么没有关系，nginx提供了一些定义，使用起来更简洁。

`NGX_CONF_TAKE12`：配置指令接受1个或者2个参数。
`NGX_CONF_TAKE13`：配置指令接受1个或者3个参数。
`NGX_CONF_TAKE23`：配置指令接受2个或者3个参数。
`NGX_CONF_TAKE123`：配置指令接受1个或者2个或者3参数。
`NGX_CONF_TAKE1234`：配置指令接受1个或者2个或者3个或者4个参数。
`NGX_CONF_1MORE`：配置指令接受至少一个参数。
`NGX_CONF_2MORE`：配置指令接受至少两个参数。
`NGX_CONF_MULTI`: 配置指令可以接受多个参数，即个数不定。
`NGX_CONF_BLOCK`：配置指令可以接受的值是一个配置信息块。也就是一对大括号括起来的内容。里面可以再包括很多的配置指令。比如常见的server指令就是这个属性的。

`NGX_CONF_FLAG`：配置指令可以接受的值是”on”或者”off”，最终会被转成bool值。
`NGX_CONF_ANY`：配置指令可以接受的任意的参数值。一个或者多个，或者”on”或者”off”，或者是配置块。

最后要说明的是，无论如何，nginx的配置指令的参数个数不可以超过NGX_CONF_MAX_ARGS个。目前这个值被定义为8，也就是不能超过8个参数值。

下面介绍一组说明配置指令可以出现的位置的属性。

`NGX_DIRECT_CONF`：可以出现在配置文件中最外层。例如已经提供的配置指令daemon，master_process等。
`NGX_MAIN_CONF`: http、mail、events、error_log等。
`NGX_ANY_CONF`: 该配置指令可以出现在任意配置级别上。
对于我们编写的大多数模块而言，都是在处理http相关的事情，也就是所谓的都是NGX_HTTP_MODULE，对于这样类型的模块，其配置可能出现的位置也是分为直接出现在http里面，以及其他位置。

`NGX_HTTP_MAIN_CONF`: 可以直接出现在http配置指令里。  
`NGX_HTTP_SRV_CONF`: 可以出现在http里面的server配置指令里。  
`NGX_HTTP_LOC_CONF`: 可以出现在http server块里面的location配置指令里。  
`NGX_HTTP_UPS_CONF`: 可以出现在http里面的upstream配置指令里。  
`NGX_HTTP_SIF_CONF`: 可以出现在http里面的server配置指令里的if语句所在的block中。  
`NGX_HTTP_LMT_CONF`: 可以出现在http里面的limit_except指令的block中。  
`NGX_HTTP_LIF_CONF`: 可以出现在http server块里面的location配置指令里的if语句所在的block中。

- set  
命令参数函数指针。当nginx在解析配置的时候，如果遇到这个配置指令，将会把读取到的值传递给这个函数进行分解处理。
先看该函数的返回值，处理成功时，返回NGX_OK，否则返回NGX_CONF_ERROR或者是一个自定义的错误信息的字符串。

再看一下这个函数被调用的时候，传入的三个参数。

`cf`: 该参数里面保存从配置文件读取到的原始字符串以及相关的一些信息。特别注意的是这个参数的args字段是一个ngx_str_t类型的数组，该数组的首个元素是这个配置指令本身，第二个元素是指令的第一个参数，第三个元素是第二个参数，依次类推。  
  
`cmd`: 这个配置指令对应的ngx_command_t结构。  

`conf`: 就是定义的存储这个配置值的结构体，自定义。例如下面定义的结构体`ngx_http_hello_world_header_loc_conf_t `,用户在处理的时候可以使用类型转换，转换成自己知道的类型，再进行字段的赋值。  
  
为了更加方便的实现对配置指令参数的读取，nginx已经默认提供了对一些标准类型的参数进行读取的函数，可以直接赋值给set字段使用。下面来看一下这些已经实现的set类型函数。

`ngx_conf_set_flag_slot`： 读取NGX_CONF_FLAG类型的参数。
`ngx_conf_set_str_slot`:读取字符串类型的参数。
`ngx_conf_set_str_array_slot`: 读取字符串数组类型的参数。
`ngx_conf_set_keyval_slot`： 读取键值对类型的参数。
`ngx_conf_set_num_slot`: 读取整数类型(有符号整数ngx_int_t)的参数。
`ngx_conf_set_size_slot`:读取size_t类型的参数，也就是无符号数。
`ngx_conf_set_off_slot`: 读取off_t类型的参数。
`ngx_conf_set_msec_slot`: 读取毫秒值类型的参数。
`ngx_conf_set_sec_slot`: 读取秒值类型的参数。
`ngx_conf_set_bufs_slot`： 读取的参数值是2个，一个是buf的个数，一个是buf的大小。例如： output_buffers 1 128k;
`ngx_conf_set_enum_slot`: 读取枚举类型的参数，将其转换成整数`ngx_uint_t`类型。
`ngx_conf_set_bitmask_slot`: 读取参数的值，并将这些参数的值以bit位的形式存储。例如：HttpDavModule模块的dav_methods指令。  

- conf  

该字段被NGX_HTTP_MODULE类型模块所用 (我们编写的基本上都是NGX_HTTP_MOUDLE，只有一些nginx核心模块是非NGX_HTTP_MODULE)，该字段指定当前配置项存储的内存位置。实际上是使用哪个内存池的问题。    

因为http模块对所有http模块所要保存的配置信息，划分了main, server和location三个地方进行存储，每个地方都有一个内存池用来分配存储这些信息的内存。
  
这里可能的值为 `NGX_HTTP_MAIN_CONF_OFFSET`、`NGX_HTTP_SRV_CONF_OFFSET`或`NGX_HTTP_LOC_CONF_OFFSET`。当然也可以直接置为0，就是NGX_HTTP_MAIN_CONF_OFFSET。

- post

该字段存储一个指针。可以指向任何一个在读取配置过程中需要的数据，以便于进行配置读取的处理。大多数时候，都不需要，所以简单地设为0即可。

根据上面的介绍，我们定义自己的命令结构如下：

```
//命令值的存储结构体
typedef struct
{
    ngx_str_t header_value; //存储命令的值
    
} ngx_http_hello_world_header_loc_conf_t;

//命令结构体
static ngx_command_t  ngx_http_hello_world_commands[] = {

    { ngx_string("hello_world_header"),
      NGX_HTTP_LOC_CONF|NGX_CONF_TAKE1, //命令出现在loc块内，且具有一个值
      ngx_http_hello_world_set,
      NGX_HTTP_LOC_CONF_OFFSET,
      offset(ngx_http_hello_world_header_loc_conf_t, header_value),//存储到ngx_http_hello_world_header_loc_conf_t结构体的header_value位置
      NULL },
      ngx_null_command //标志命令结束
};

static char* ngx_http_hello_world_set(ngx_conf_t *cf, ngx_command_t *cmd, void *conf){
	    
	    printf("---------ngx_http_hello_world_set--------");
       ngx_http_hello_world_header_loc_conf_t* local_conf;

       local_conf = conf;
       char* rv = ngx_conf_set_str_slot(cf, cmd, conf);

        return rv;

}

 
```
#### 定义模块上下文结构
模块上下文结构实际上是提供一组回调函数指针，这些函数有在创建存储配置信息的对象的函数，也有在创建前和创建后会调用的函数。这些函数都将被nginx在合适的时间进行调用。
其结构如下:

```
typedef struct  {
    ngx_int_t   (*preconfiguration)(ngx_conf_t *cf),                /* preconfiguration */
    ngx_int_t   (*postconfiguration)(ngx_conf_t *cf),               /* postconfiguration */

    void       *(*create_main_conf)(ngx_conf_t *cf),                /* create main configuration */
    char       *(*init_main_conf)(ngx_conf_t *cf, void *conf),      /* init main configuration */

    void       *(*create_srv_conf)(ngx_conf_t *cf),                 /* create server configuration */
    char       *(*merge_srv_conf)(ngx_conf_t *cf, void *prev, void *conf), /* merge server configuration */

    void       *(*create_loc_conf)(ngx_conf_t *cf),    /* create location configuration */
    char       *(*merge_loc_conf)(ngx_conf_t *cf, void *prev, void *conf)   /* merge location configuration */
}ngx_http_module_t;

```
- preconfiguration
 在创建和读取该模块的配置信息之前被调用。

- postconfiguration
 在创建和读取该模块的配置信息之后被调用。

- create_main_conf
 调用该函数创建本模块位于http block的配置信息存储结构。该函数成功的时候，返回创建的配置对象。失败的话，返回NULL。

- init_main_conf
 调用该函数初始化本模块位于http block的配置信息存储结构。该函数成功的时候，返回NGX_CONF_OK。失败的话，返回NGX_CONF_ERROR或错误字符串。

- create_srv_conf:
 调用该函数创建本模块位于http server block的配置信息存储结构，每个server block会创建一个。该函数成功的时候，返回创建的配置对象。失败的话，返回NULL。

- merge_srv_conf:	
 因为有些配置指令既可以出现在http block，也可以出现在http server block中。那么遇到这种情况，每个server都会有自己存储结构来存储该server的配置，但是在这种情况下http block中的配置与server block中的配置信息发生冲突的时候，就需要调用此函数进行合并，该函数并非必须提供，当预计到绝对不会发生需要合并的情况的时候，就无需提供。当然为了安全起见还是建议提供。该函数执行成功的时候，返回NGX_CONF_OK。失败的话，返回NGX_CONF_ERROR或错误字符串。

- create_loc_conf:
 调用该函数创建本模块位于location block的配置信息存储结构。每个在配置中指明的location创建一个。该函数执行成功，返回创建的配置对象。失败的话，返回NULL。

- merge_loc_conf:	
 与merge_srv_conf类似，这个也是进行配置值合并的地方。该函数成功的时候，返回NGX_CONF_OK。失败的话，返回NGX_CONF_ERROR或错误字符串。

Nginx里面的配置信息都是上下一层层的嵌套的，对于具体某个location的话，对于同一个配置，如果当前层次没有定义，那么就使用上层的配置，否则使用当前层次的配置。

这些配置信息一般默认都应该设为一个未初始化的值，针对这个需求，Nginx定义了一系列的宏定义来代表各种配置所对应数据类型的未初始化值，如下：

又因为对于配置项的合并，逻辑都类似，也就是前面已经说过的，如果在本层次已经配置了，也就是配置项的值已经被读取进来了（那么这些配置项的值就不会等于上面已经定义的那些UNSET的值），就使用本层次的值作为定义合并的结果，否则，使用上层的值，如果上层的值也是这些UNSET类的值，那就赋值为默认值，否则就使用上层的值作为合并的结果。对于这样类似的操作，Nginx定义了一些宏操作来做这些事情，我们来看其中一个的定义。

```
#define ngx_conf_merge_uint_value(conf, prev, default)                       \
    if (conf == NGX_CONF_UNSET_UINT) {                                       \
        conf = (prev == NGX_CONF_UNSET_UINT) ? default : prev;               \
    }

```

显而易见，这个逻辑确实比较简单，所以其它的宏定义也类似，我们就列具其中的一部分吧。

```
ngx_conf_merge_value
ngx_conf_merge_ptr_value
ngx_conf_merge_uint_value
ngx_conf_merge_msec_value
ngx_conf_merge_sec_value

```

上述参数定义介绍完毕以后，我们定义我们的模块上下文如下:

```
static ngx_http_module_t  ngx_http_hello_world_module_ctx = {
    NULL,                                  /* preconfiguration */
    ngx_http_hello_world_init,              /* postconfiguration */
    NULL,                                  /* create main configuration */
    NULL,                                  /* init main configuration */
    NULL,                                  /* create server configuration */
    NULL,                                  /* merge server configuration */
    NULL,       /* create location configuration */
    NULL         /* merge location configuration */
};

//进行模块的挂载，下文详细介绍
static ngx_int_t ngx_http_hello_world_init(ngx_conf_t *cf){
	printf("---------ngx_http_hello_world_init--------");
    ngx_http_handler_pt        *h;
	ngx_http_core_main_conf_t  *cmcf;

	cmcf = ngx_http_conf_get_module_main_conf(cf,ngx_http_core_module);

	h = ngx_array_push(&cmcf->phases[NGX_HTTP_CONTENT_PHASE].handlers);
	if (h == NULL) {
	        return NGX_ERROR;
	}

	*h = ngx_http_hello_world_handler;

	return NGX_OK;

}

```
#### 定义模块
在完成上述两部分定义以后，ngx_module_t类型的变量来说明这个模块本身的信息。其结构如下:
```
ngx_module_t  ngx_http_hello_world_module = {
    NGX_MODULE_V1,
    &ngx_http_hello_world_module_ctx,       /* module context */
    ngx_http_hello_world_commands,          /* module directives */
    NGX_HTTP_MODULE,                       /* module type */
    NULL,                                  /* init master */
    NULL,                                  /* init module */
    NULL,                                  /* init process */
    NULL,                                  /* init thread */
    NULL,                                  /* exit thread */
    NULL,                                  /* exit process */
    NULL,                                  /* exit master */
    NGX_MODULE_V1_PADDING
};
```
#### 定义模块处理函数

在完成模块相关结构定义以后，必须提供一个真正的处理函数handler，其命名形式为

`typedef ngx_int_t (*ngx_http_handler_pt)(ngx_http_request_t *r);`  

这个函数负责对来自客户端请求的真正处理。这个函数的处理，既可以选择自己直接生成内容，也可以选择拒绝处理，由后续的handler去进行处理，或者是选择丢给后续的filter进行处理。  

r是http请求。里面包含请求所有的信息，这里不详细说明了，可以参考别的章节的介绍。 该函数处理成功返回NGX_OK，处理发生错误返回NGX_ERROR，拒绝处理（留给后续的handler进行处理）返回NGX_DECLINE。 返回NGX_OK也就代表给客户端的响应已经生成好了，否则返回NGX_ERROR就发生错误了。
我们的模块的功能是在响应报文中增加报文头`X-Hello-World:Tango`，因此通过定义handler函数实现该功能。

```
static ngx_int_t ngx_http_hello_world_handler(ngx_http_request_t *r)
{

	printf("%s\n", "-------ngx_http_hello_world_handler--------");
	//不关心包体内容，因此丢弃包体内容
	ngx_int_t rc = ngx_http_discard_request_body(r);
    if (rc != NGX_OK) {
        return rc;
    }
    //增加一个头部信息
    h = ngx_list_push(&r->headers_out.headers );
    if(h == NULL){
    	return NGX_ERROR;
    }
    ngx_str_set(&h->key,"X-Hello-World");
    ngx_str_set(&h->value,"Tango");
    h->hash = 1;
  
    return NG_DECLINED;
}

```


#### 模块处理函数挂载

上一篇章中介绍HTTP请求处理分为11个阶段处理，我们在完成相应模块开发以后，需要将相应的handler函数挂载到相应阶段。其11个阶段分别为:

| 阶段 | 描述 | 备注 |  
| ---- | ---- | ----- |  
|NGX_HTTP_POST_READ_PHASE|读取请求内容阶段||
|NGX_HTTP_SERVER_REWRITE_PHASE|Server请求地址重写阶段|
|NGX_HTTP_FIND_CONFIG_PHASE|配置查找阶段|不可挂载
|NGX_HTTP_REWRITE_PHASE|Location请求地址重写阶段|
|NGX_HTTP_POST_REWRITE_PHASE|请求地址重写提交阶段|不可挂载
|NGX_HTTP_PREACCESS_PHASE|访问权限检查准备阶段|
NGX_HTTP_ACCESS_PHASE|访问权限检查阶段|
NGX_HTTP_POST_ACCESS_PHASE|访问权限检查提交阶段|不可挂载
NGX_HTTP_TRY_FILES_PHASE|配置项try_files处理阶段|不可挂载
NGX_HTTP_CONTENT_PHASE|内容产生阶段
NGX_HTTP_LOG_PHASE|日志模块处理阶段

表格中备注不可挂载的阶段一般不做模块挂载处理。一般情况下，我们自定义的模块，大多数是挂载在NGX_HTTP_CONTENT_PHASE阶段的。挂载的动作一般是在模块上下文调用的postconfiguration函数中。
挂载方式分为按需挂载和按处理阶段挂载两种方式。
##### 按处理阶段挂载

```
static ngx_int_t ngx_http_hello_world_init(ngx_conf_t *cf){
	printf("---------ngx_http_hello_world_init--------");
    ngx_http_handler_pt        *h;
	ngx_http_core_main_conf_t  *cmcf;

	cmcf = ngx_http_conf_get_module_main_conf(cf,ngx_http_core_module);

	h = ngx_array_push(&cmcf->phases[NGX_HTTP_CONTENT_PHASE].handlers);
	if (h == NULL) {
	        return NGX_ERROR;
	}

	*h = ngx_http_hello_world_handler;

	return NGX_OK;
}
```
##### 按需挂载
以这种方式挂载的handler也被称为 content handler。

当一个请求进来以后，nginx从NGX_HTTP_POST_READ_PHASE阶段开始依次执行每个阶段中所有handler。执行到 NGX_HTTP_CONTENT_PHASE阶段的时候，如果这个location有一个对应的content handler模块，那么就去执行这个content handler模块真正的处理函数。否则继续依次执行NGX_HTTP_CONTENT_PHASE阶段中所有content phase handlers，直到某个函数处理返回NGX_OK或者NGX_ERROR。

换句话说，当某个location处理到NGX_HTTP_CONTENT_PHASE阶段时，如果有content handler模块，那么NGX_HTTP_CONTENT_PHASE挂载的所有content phase handlers都不会被执行了。

但是使用这个方法挂载上去的handler有一个特点是必须在NGX_HTTP_CONTENT_PHASE阶段才能执行到。如果你想自己的handler在更早的阶段执行，那就不要使用这种挂载方式。

那么在什么情况会使用这种方式来挂载呢？一般情况下，某个模块对某个location进行了处理以后，发现符合自己处理的逻辑，而且也没有必要再调用NGX_HTTP_CONTENT_PHASE阶段的其它handler进行处理的时候，就动态挂载上这个handler。此种方式可以在定义`ngx_http_hello_world_commands`中set函数进行挂载。

```
static char * ngx_http_hello_world_set(ngx_conf_t *cf, ngx_command_t *cmd, void *conf){
	  printf("---------ngx_http_hello_world_init--------");
     ngx_http_handler_pt        *h;
	  ngx_http_core_loc_conf_t  *clcf;

     clcf = ngx_http_conf_get_module_loc_conf(cf, ngx_http_core_module);
     clcf->handler =ngx_http_hello_world_handler;

     return NGX_CONF_OK;
}

```


#### 创建模块文件
在完成上述函数定义以后，即完成一个模块的开发，则将上述函数组织成一个模块文件。
在`nginx/src/`目录下创建目录`ext`,在ext目录下以模块名称创建文件`ngx_http_hello_world_module.c `
将上述函数定义保存到文件中，其完整结构如下:

```
#include <ngx_config.h>
#include <ngx_core.h>
#include <ngx_http.h>

static ngx_int_t ngx_http_hello_world_handler(ngx_http_request_t *r);
static ngx_int_t ngx_http_hello_world_init(ngx_conf_t *cf);
ngx_module_t  ngx_http_hello_world_module;

typedef struct {
  
  ngx_str_t header_value;
  
} ngx_http_hello_world_header_loc_conf_t;

static ngx_int_t ngx_http_hello_world_handler(ngx_http_request_t *r){

	printf("%s\n", "-------ngx_http_hello_world_handler--------");
  
  ngx_http_hello_world_header_loc_conf_t* local_conf;
  ngx_table_elt_t  *h;

  local_conf = ngx_http_get_module_loc_conf(r,ngx_http_hello_world_module);
  ngx_str_t header_value = local_conf->header_value;

	ngx_int_t rc = ngx_http_discard_request_body(r);
  if (rc != NGX_OK) {
        return rc;
  }

  h = ngx_list_push(&r->headers_out.headers );
  if(h == NULL){
    	return NGX_ERROR;
  }
  ngx_str_set(&h->key,"X-Hello-World");
  ngx_str_set(&h->value,header_value.data);
  h->hash = 1;
  
  return NGX_DECLINED;
}


static ngx_int_t ngx_http_hello_world_init(ngx_conf_t *cf){
	
  printf("---------ngx_http_hello_world_init--------");
  ngx_http_handler_pt        *h;
	ngx_http_core_main_conf_t  *cmcf;

	cmcf = ngx_http_conf_get_module_main_conf(cf,ngx_http_core_module);
	h = ngx_array_push(&cmcf->phases[NGX_HTTP_CONTENT_PHASE].handlers);
	if (h == NULL) {
	    return NGX_ERROR;
	}
	*h = ngx_http_hello_world_handler;

	return NGX_OK;

}

static void *ngx_http_hello_world_create_loc_conf(ngx_conf_t *cf){

  ngx_http_hello_world_header_loc_conf_t* local_conf = NULL;
  local_conf = ngx_pcalloc(cf->pool, sizeof(ngx_http_hello_world_header_loc_conf_t));
  if (local_conf == NULL)
  {
          return NULL;
  }
  ngx_str_null(&local_conf->header_value);

  return local_conf;
}

static char* ngx_http_hello_world_set(ngx_conf_t *cf, ngx_command_t *cmd, void *conf){
	 
   printf("---------ngx_http_hello_world_set--------");
   //ngx_http_hello_world_header_loc_conf_t* local_conf;
   //local_conf = conf;
   char* rv = ngx_conf_set_str_slot(cf, cmd, conf);
   return rv;

}

static ngx_command_t  ngx_http_hello_world_commands[] = {

  { ngx_string("hello_world_header"),
    NGX_HTTP_LOC_CONF|NGX_CONF_TAKE1,
    ngx_http_hello_world_set,
    NGX_HTTP_LOC_CONF_OFFSET,
    offsetof(ngx_http_hello_world_header_loc_conf_t,header_value),
    NULL },
    ngx_null_command
};


static ngx_http_module_t  ngx_http_hello_world_module_ctx = {
  NULL,                                  /* preconfiguration */
  ngx_http_hello_world_init,             /* postconfiguration */
  NULL,                                  /* create main configuration */
  NULL,                                  /* init main configuration */
  NULL,                                  /* create server configuration */
  NULL,                                  /* merge server configuration */
  ngx_http_hello_world_create_loc_conf,  /* create location configuration */
  NULL                                   /* merge location configuration */
};

ngx_module_t  ngx_http_hello_world_module = {
    NGX_MODULE_V1,
    &ngx_http_hello_world_module_ctx,       /* module context */
    ngx_http_hello_world_commands,          /* module directives */
    NGX_HTTP_MODULE,                       /* module type */
    NULL,                                  /* init master */
    NULL,                                  /* init module */
    NULL,                                  /* init process */
    NULL,                                  /* init thread */
    NULL,                                  /* exit thread */
    NULL,                                  /* exit process */
    NULL,                                  /* exit master */
    NGX_MODULE_V1_PADDING
};

```

#### 编写config文件
对于开发一个模块，我们是需要把这个模块的C代码组织到一个目录里，同时需要编写一个config文件。这个config文件的内容就是告诉nginx的编译脚本，该如何进行编译。在nginx目录(configure命令目录)下创建config文件
```
ngx_addon_name=ngx_http_hello_world_module
HTTP_MODULES="$HTTP_MODULES ngx_http_hello_world_module"
NGX_ADDON_SRCS="$NGX_ADDON_SRCS $ngx_addon_dir/ngx_http_hello_world_module.c"
```
#### 编译与安装
```
./configure --with-debug --add-module=/home/iknow/nginx/nginx-1.15.1/src/ext/
make
make install
```
执行上述命令进行nginx源码编译与安装,其中xxx为nginx源码的绝对路径。

#### 运行nginx

**修改配置文件**  
修改`/usr/local/nginx/conf/nginx.conf`文件
在location模块下增加定义的命令:

```
   server {
        listen       80;
        server_name  localhost;

        #charset koi8-r;

        #access_log  logs/host.access.log  main;

        location / {
            ##添加的命令
            hello_world_header Tango; 添加的命令
            root   html;
            index  index.html index.htm;
        }

        #error_page  404              /404.html;

        # redirect server error pages to the static page /50x.html
        #
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }

        # proxy the PHP scripts to Apache listening on 127.0.0.1:80
        #
        #location ~ \.php$ {
        #    proxy_pass   http://127.0.0.1;
        #}

        # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
        #
        #location ~ \.php$ {
        #    root           html;
        #    fastcgi_pass   127.0.0.1:9000;
        #    fastcgi_index  index.php;
        #    fastcgi_param  SCRIPT_FILENAME  /scripts$fastcgi_script_name;
        #    include        fastcgi_params;
        #}

        # deny access to .htaccess files, if Apache's document root
        # concurs with nginx's one
        #
        #location ~ /\.ht {
        #    deny  all;
        #}
    }

```
**启动nginx**

```
/usr/local/nginx/sbin/nginx
```

#### 验证
执行`curl -i localhost:80/`
得到响应:

```
HTTP/1.1 200 OK
Server: nginx/1.15.1
Date: Sat, 21 Jul 2018 13:33:21 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Thu, 19 Jul 2018 05:18:08 GMT
Connection: keep-alive
X-Hello-World: Tango
X-Hello-World: Tango
ETag: "5b501f10-264"
Accept-Ranges: bytes

<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>

```
至此，完成一个HTTP模块开发。

至此，完成一个HTTP模块开发。

至此，完成一个HTTP模块开发。

**参考文章** [handler模块(100%)](http://tengine.taobao.org/book/chapter_03.html)