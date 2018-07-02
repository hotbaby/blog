---
title: Nginx autoindex乱码
date: 2017-05-09 22:13:20
tags: [Nginx]
toc: true
comments: true
---

使用nginx_autoindex_moudle检索本地文件时，如果文件名字包含中文，则会出现乱码.

nginx配置文件如下

```nginx
location /static/ {
alias $static/;        
autoindex on;
autoindex_exact_size on;
autoindex_localtime on;
}
```

检查发现nginx auto_index 生成的html文件header未包含**charset**

nginx 源码剖析

`src/http/modules/ngx_http_autoindex_module.c`

```c
switch (format) {

case NGX_HTTP_AUTOINDEX_JSON:
    ngx_str_set(&r->headers_out.content_type, "application/json");
    break;

case NGX_HTTP_AUTOINDEX_JSONP:
    ngx_str_set(&r->headers_out.content_type, "application/javascript");
    break;

case NGX_HTTP_AUTOINDEX_XML:
    ngx_str_set(&r->headers_out.content_type, "text/xml");
    ngx_str_set(&r->headers_out.charset, "utf-8");
    break;

default: /* NGX_HTTP_AUTOINDEX_HTML */
    ngx_str_set(&r->headers_out.content_type, "text/html");
    break;
}
```

确认**autoindex_format**为html时，确实未增加charset的http头部信息。

修改源代码

```c
default: /* NGX_HTTP_AUTOINDEX_HTML */
    ngx_str_set(&r->headers_out.content_type, "text/html");
    ngx_str_set(&r->headers_out.charset, "utf-8");
    break;
```

重新编译nginx

```shell
$ ./configure --prefix=/opt/nginx --with-http_ssl_module
$ make; make install
```

重启nginx

强制刷新页面， OK, 问题解决

*An official read-only mirror of http://hg.nginx.org/nginx/ which is updated hourly. Pull requests on GitHub cannot be accepted and will be automatically closed. The proper way to submit changes to nginx is via the nginx development mailing list, see http://nginx.org/en/docs/contributing_changes.html http://nginx.org/*

nginx 不支持github pull request. Fuck

## References

- [nginx github repo](https://github.com/nginx/nginx)
- [nginx autoindex module document](https://nginx.org/en/docs/http/ngx_http_autoindex_module.html)