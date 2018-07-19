---
layout:     post
title:      "NGINX系列之HTTP模块开发"
subtitle:   ""
date:       2018-03-20 12:00:00
author:     "Tango"
header-img: "img/post-bg-universe.jpg"
catalog: true
tags:   
    - nginx
    - 系统架构
---

上文介绍nginx请求11个阶段处理，本文将动手实操开发并注册一个HTTP模块，在实现`ngx_http_hello_world_module `模块的过程中，详细介绍其实现步骤。

## 模块组成介绍
### 模块功能
功能相对简单，从`Hello World`开始，通过实现扩展HTTP模块`ngx_http_hello_world_module `，实现在响应头中增加一个字段`X-Hello-World:Tango`功能。
### 模块开发步骤
1.编写模块基本结构。包括模块的定义，模块上下文结构，模块的配置结构等。
2.实现handler的挂载函数，根据模块的需求选择正确的挂载方式。
3.编写模块核心功能实现函数handler处理函数。
4.编写编译的文件config
5.将实现的模块添加到nginx
6.修改配置文件，验证可行性。

```
#include <ngx_config.h>
#include <ngx_core.h>
#include <ngx_http.h>

static ngx_int_t ngx_http_hello_world_handler(ngx_http_request_t *r);
static ngx_int_t ngx_http_hello_world_init(ngx_conf_t *cf);


static ngx_int_t ngx_http_hello_world_handler(ngx_http_request_t *r)
{

	printf("%s\n", "-------ngx_http_hello_world_handler--------");
	ngx_int_t rc = ngx_http_discard_request_body(r);
    if (rc != NGX_OK) {
        return rc;
    }
    h = ngx_list_push(&r->headers_out.headers );
    if(h == NULL){
    	return NGX_ERROR;
    }
    ngx_str_set(&h->key,"X-Hello-World");
    ngx_str_set(&h->value,"Tango");
    h->hash = 1;
  
    return NG_DECLINED;
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

static char* ngx_http_hello_world_set(ngx_conf_t *cf, ngx_command_t *cmd, void *conf){
	printf("---------ngx_http_hello_world_set--------");
 //    ngx_http_handler_pt        *h;
	// ngx_http_core_main_conf_t  *cmcf;

	// cmcf = ngx_http_conf_get_module_main_conf(cf, ngx_http_core_module);

	// h = ngx_array_push(&cmcf->phases[NGX_HTTP_CONTENT_PHASE].handlers);
	// if (h == NULL) {
	//         return NGX_CONF_ERROR;
	// }

	// *h = ngx_http_hello_world_handler;

	return NGX_CONF_OK;

}

static ngx_command_t  ngx_http_hello_world_commands[] = {

    { ngx_string("hello_world_header"),
      NGX_HTTP_LOC_CONF|NGX_CONF_NOARGS,
      ngx_http_hello_world_set,
      NGX_HTTP_LOC_CONF_OFFSET,
      0,
      NULL },
      ngx_null_command
};


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


参考文章 [handler模块(100%)](http://tengine.taobao.org/book/chapter_03.html)