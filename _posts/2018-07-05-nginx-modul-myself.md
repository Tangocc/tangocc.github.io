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

上文介绍nginx请求11个阶段处理，本文将动手实操开发并注册一个HTTP模块。
实现功能：增加响应头``

## 模块组成介绍




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
    // size_t                          n;
    // uint32_t                        hash;
    // ngx_str_t                       key;
    // ngx_uint_t                      i;
    // ngx_slab_pool_t                *shpool;
    // ngx_rbtree_node_t              *node;
    // ngx_pool_cleanup_t             *cln;
    // ngx_http_limit_conn_ctx_t      *ctx;
    // ngx_http_limit_conn_node_t     *lc;
    // ngx_http_limit_conn_conf_t     *lccf;
    // ngx_http_limit_conn_limit_t    *limits;
    // ngx_http_limit_conn_cleanup_t  *lccln;
     ngx_table_elt_t  *h;

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

	cmcf = ngx_http_conf_get_module_main_conf(cf, ngx_http_core_module);

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