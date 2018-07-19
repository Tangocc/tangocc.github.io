---
layout:     post
title:      "NGINX系列之扩展开发"
subtitle:   ""
date:       2018-03-20 12:00:00
author:     "Tango"
header-img: "img/post-bg-universe.jpg"
catalog: true
tags:   
    - nginx
    - 系统架构
---

上文介绍nginx 11个阶段处理，本文则实现并注册一个自己定义的HTTP模块。

## 模块组成介绍
任意打开文件夹`/nginx/src/http/modules`下一个文件可以看到其代码结构为：

```

static ngx_command_t  ngx_http_access_commands[] = {

    { ngx_string("allow"),
      NGX_HTTP_MAIN_CONF|NGX_HTTP_SRV_CONF|NGX_HTTP_LOC_CONF|NGX_HTTP_LMT_CONF
                        |NGX_CONF_TAKE1,
      ngx_http_access_rule,
      NGX_HTTP_LOC_CONF_OFFSET,
      0,
      NULL },
      ngx_null_command
};



static ngx_http_module_t  ngx_http_access_module_ctx = {
    NULL,                                  /* preconfiguration */
    NULL,                  /* postconfiguration */

    NULL,                                  /* create main configuration */
    NULL,                                  /* init main configuration */

    NULL,                                  /* create server configuration */
    NULL,                                  /* merge server configuration */

    ngx_http_access_create_loc_conf,       /* create location configuration */
    ngx_http_access_merge_loc_conf         /* merge location configuration */
};


ngx_module_t  ngx_http_access_module = {
    NGX_MODULE_V1,
    &ngx_http_access_module_ctx,           /* module context */
    ngx_http_access_commands,              /* module directives */
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


static ngx_int_t
ngx_http_access_handler(ngx_http_request_t *r)
{
    return NGX_DECLINED;
}

```

