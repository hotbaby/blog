---
title: Nginx重试机制导致流水号重复
comments: true
toc: true
date: 2019-04-16 16:51:34
tags: Nginx
---

### 问题

调用银行E租通`EZ26`接口时，报错[流水号重复]错误。

<!-- more -->

### 分析

环境描述

- 操作系统: Ubuntu 14.04.5
- Nginx: 1.4.6
- Python: 2.7.6
- Django: 1.8.18
- Python Requests: 2.21.0



大致分析思路：确认流水号的生成机制是否存在重复的可能性，如果没有，分析业务、Nginx的日志，检查是否存在重试机制。



1、确认流水号的生成机制？

流水号生成算法：时间戳(到毫秒) + 10位随机数。因此，流水号生成机制上，不存在流水号重复的可能性。

2、网络请求链路分析？

![](https://ws1.sinaimg.cn/large/006tNc79ly1g23h6t2yenj311r0jf3yy.jpg)

3、确认Python的`requests`包内部是否有重试机制？

分析业务Web App的日志，没有发现重复流水号的请求记录。检查`request`代码内部没有默认重试机制。

4、确认前置机转发请求是否有重试机制？

经确认，前置机转发没有重试机制。

5、分析Nginx内部是否有重试机制？

分析Nginx访问、错误日志，发现有很多请求的响应时间是60，正好是Nginx转发的超时时间， `ngx_read_timeout`默认为60s。怀疑Nginx内部有超时重试机制。

Nginx访问日志：

```nginx
{"@timestamp":"2019-04-12T16:13:47+08:00","host":"47.93.151.131","clientip":"123.57.117.130","size":176,"http_host":"www.jxfangguanjia.com","request":"POST /ccbapi/ccb/sdk/online/EZ26 HTTP/1.1","url":"/ccb/sdk/online/EZ26","xff":"47.93.151.131","referer":"-","agent":"python-requests/2.21.0","status":"504","upstreamhost":"39.108.162.242:22116","responsetime":60.060,"upstreamtime":"60.060"}
{"@timestamp":"2019-04-12T16:13:48+08:00","host":"47.93.151.131","clientip":"123.57.117.130","size":532,"http_host":"www.jxfangguanjia.com","request":"POST /ccbapi/ccb/sdk/online/EZ26 HTTP/1.1","url":"/ccb/sdk/online/EZ26","xff":"47.93.151.131","referer":"-","agent":"python-requests/2.21.0","status":"200","upstreamhost":"39.108.162.242:22116","responsetime":0.226,"upstreamtime":"0.226"}
```

进一步分析[Nginx proxy文档](<http://nginx.org/en/docs/http/ngx_http_proxy_module.html>)：

```nginx
Syntax:	proxy_read_timeout time;
Default:	
proxy_read_timeout 60s;
Context:	http, server, location

Defines a timeout for reading a response from the proxied server. The timeout is set only between two successive read operations, not for the transmission of the whole response. If the proxied server does not transmit anything within this time, the connection is closed.
```

意思是`proxy_read_timeout`超时的默认行为是，该请求的链接会关闭，没有重试。

**进入死胡同，进一步分析Nginx的proxy模块文档，发现Nginx升级到1.7.5版本，增加`proxy_next_upstream_tries`和`proxy_next_upstream_timeout`两个指令，怀疑是不是当前的Nginx的版本是不是比较低，内部默认是有重试机制的。**

查看Nginx的版本为`1.4.6`，更进一步分析Nginx 1.4版本的源代码，验证这种可能性。

Nginx源代码分析

下载[Nginx源代码](<https://github.com/nginx/nginx>)，并checkout到`branches/stable-1.4`分支，分析Nginx的upstream和proxy模块代码。

Nginx proxy模块`proxy_read_timeout`指令注册

`nginx/src/http/modules/ngx_http_proxy_module.c`

```c
static ngx_command_t  ngx_http_proxy_commands[] = {
			{ ngx_string("proxy_read_timeout"),
      NGX_HTTP_MAIN_CONF|NGX_HTTP_SRV_CONF|NGX_HTTP_LOC_CONF|NGX_CONF_TAKE1,
      ngx_conf_set_msec_slot,
      NGX_HTTP_LOC_CONF_OFFSET,
      offsetof(ngx_http_proxy_loc_conf_t, upstream.read_timeout),
      NULL }
}
```

`nginx/src/http/ngx_http_upstream.c`

```c
static void
ngx_http_upstream_connect(ngx_http_request_t *r, ngx_http_upstream_t *u)
{
	  c = u->peer.connection;
    c->write->handler = ngx_http_upstream_handler;
    c->read->handler = ngx_http_upstream_handler;	/* 注册upstream read handler，read为ngx事件*/
  
    u->read_event_handler = ngx_http_upstream_process_header; /* 注册read_event_handler */
}

static void
ngx_http_upstream_send_request(ngx_http_request_t *r, ngx_http_upstream_t *u)
{
 ngx_add_timer(c->read, u->conf->read_timeout);  /* 注册定时器 */
}

static void
ngx_http_upstream_handler(ngx_event_t *ev)
{
    ngx_connection_t     *c;
    ngx_http_request_t   *r;
    ngx_http_log_ctx_t   *ctx;
    ngx_http_upstream_t  *u;

    c = ev->data;
    r = c->data;

    u = r->upstream;
    c = r->connection;

    ctx = c->log->data;
    ctx->current_request = r;

    ngx_log_debug2(NGX_LOG_DEBUG_HTTP, c->log, 0,
                   "http upstream request: \"%V?%V\"", &r->uri, &r->args);

    if (ev->write) {
        u->write_event_handler(r, u);

    } else {
        u->read_event_handler(r, u);	/* 超时调用read hanlder */
    }

    ngx_http_run_posted_requests(c);
}

static void
ngx_http_upstream_process_header(ngx_http_request_t *r, ngx_http_upstream_t *u)
{
    ssize_t            n;
    ngx_int_t          rc;
    ngx_connection_t  *c;

    c = u->peer.connection;

    ngx_log_debug0(NGX_LOG_DEBUG_HTTP, c->log, 0,
                   "http upstream process header");

    c->log->action = "reading response header from upstream";

  	/* 链接read超时，则进行重试*/
    if (c->read->timedout) {
        ngx_http_upstream_next(r, u, NGX_HTTP_UPSTREAM_FT_TIMEOUT);
        return;
    }
}
```

分析Nginx的`upstream`模块，发现在上游服务器超时后，会自动进行重试。因此，建行服务器收到相同流水号的请求，直接返回[流水号重复]的错误。而`proxy` 模块 `proxy_read_time`指令默认值微60s。

另外，在当前版本中，没有找到可以控制重试机制的指令，需要升级Nginx版本。而根据文档描述，在1.7.5以及更高版本中，已经包含Nginx upstream的重试机制的指令，而且默认情况下，重试机制处于关闭状态。

### 解决

升级Nginx版本，要求版本号大于1.7.5

### 延伸

从这个BUG想到的：

* 不要急于下结论，首先要排除自己业务出错的可能性，再去与其他业务部门沟通、分析问题的原因，同时也不能完全寄希望于别人身上，他人只负责提供意见或灵感辅助解决问题，自己要持续跟踪、分析，并最终解决问你题。
* 不要局限于一个节点，了解整个业务端流程、网络拓扑，逐层拆解，分析可能出现问题节点。
* 分析日志，从日志寻找解决问题的线索。
* 查看开源软件文档、代码，从中分析导致问题的根本原因，找到解决问题的最终解决方案。

### 参考

- [Nginx源代码](<https://github.com/nginx/nginx>)
- [Nginx document proxy module](<http://nginx.org/en/docs/http/ngx_http_proxy_module.html>)