---
title: HTTP缓存
date: 2017-07-29 22:32:15
tags: [HTTP]
toc: true
---

缓存是某个资源(或文档)的副本。对于私有缓存，可以服务同一个客户端的对同一个资源多次的请求，比如浏览器缓存。对于共享缓存(或代理缓存)，可以服务多个客户端对于同一个资源的请求，比如CDN。为什么需要缓存，缓存有哪些优点：

- 减少网络延迟，更快的加载内容，提高用户体验
- 减少冗余数据传输，降低网络负载
- 降低服务器负载，服务器可以更快的处理请求

## 不同类型的缓存

- 私有缓存 - 服务于单个用户
- 共享缓存 -存储响应被很多用户复用

其他缓存，gateway cache, CDN, reverse proxy cache 和　部署在服务器上的负载均衡器，以获得更好的可靠性和性能．

[![HTTP Cache Type](http://www.processon.com/chart_image/597c3deee4b06e43d2d663b5.png)](http://www.processon.com/chart_image/597c3deee4b06e43d2d663b5.png)HTTP Cache Type

**私有浏览器缓存(Private browser cache)**

私有缓存服务于单个用户．

### 共享代理缓存(Shared proxy caches)

共享缓存服务于多个用户．比如ISP或者企业可以配置一个web代理，作为本地网络基础架构，服务于多个用户．热点资源可以被多个用户复用，减少网络流量和延迟．　

常用的缓存条目的形式：

- 成功的检索请求结果 - 200 OK响应，比如HTML 文档，图片或者文件
- 永久重定向 - 301(Moved Permanently)响应
- 错误响应 - 404(Not Found)结果页
- 未完成的结果 - 206(Partial Content)响应
- 不仅仅是GET请求的响应 - 定义一个合适的cache key

## 缓存处理步骤

对于一个HTTP GET请求报文，基本缓存处理包含7个步骤：

1. 接收-缓存从网络中读取请求报文
2. 解析-缓存对报文进行解析，提取HTTP首部信息
3. 查询-缓存查询是否命中，如果没有，则从源服务器获取，并缓存到本地
4. 新鲜度检测-检查副本是否新鲜，如果不新鲜，则与源服务器进行验证
5. 响应-缓存用新的首部和缓存主体创建响应报文
6. 发送-缓存通过网络将响应发送给客户端
7. 日志-缓存记录这次请求的日志

缓存处理流程图：
[![HTTP_CACHE_GET_FLOW_CHART](http://www.processon.com/chart_image/597ca20ee4b06e43d2d675ec.png)](http://www.processon.com/chart_image/597ca20ee4b06e43d2d675ec.png)HTTP_CACHE_GET_FLOW_CHART

## 缓存控制

服务器可以通过HTTP定义文档过期之前可以将其缓存多长时间。

- Cache-Control: no-store
- Cache-Control: no-cache
- Cache-Control: must-revalidate
- Cache-Control: max-age
- Expires

no-store与no-cache

HTTP/1.1提供了几种限制对象缓存方式。 no-store和no-cache首部可以防止缓存未经证实的已缓存对象：

```
Pragma: no-cache
Cache-Control: no-store
Cache-Control: no-cache
```

标识为`no-store`的响应回禁止缓存对响应进行复制。标识为`no-cache`的响应可以在本地缓存中，只是在与服务器进行新鲜度再验证之前，缓存不能提供给客户端使用。

`Cache-Control: max-age=3600`表示收到服务器响应文档处于新鲜状态的秒数。max-age=0表示不缓存文档。

`Expires: Fri, 05 Jul 2017, 12:00:00 GMT`表示文档绝对过期时间。**不推荐使用Expires**，HTTP设计者后来任务，由于服务器之间的时间不同步或不正确，会导致文档新鲜度计算错误。

如果源服务器希望缓存严格遵守过期时间，可以在加`Cache-Control: must-revalidate`的HTTP首部。`Cache-Control: must-revalidate`响应告诉缓存，在事先没有跟源服务器再验证之前，不能提供这个对象的过期副本。缓存仍然可以提供新鲜的副本。如果缓存进行must-revalidate新鲜度是，源服务器不可用，缓存必须返回一条Gateway Timeout从错误。

## 缓存命中

缓存命中、未命中和再验证：
[![HTTP Cache Hit](http://www.processon.com/chart_image/597c0011e4b06e43d2d653b8.png)](http://www.processon.com/chart_image/597c0011e4b06e43d2d653b8.png)HTTP Cache Hit

缓存命中率

由缓存缓存提供服务的请求所占的比例成为称为缓存命中率(cache hit rate)。命中率在0到1之间，0表示缓存全部未命中，1表示缓存全部命中。缓存服务提供者希望缓存的命中是100%，而实际的缓存命中率与缓存大小，缓存内容变化，请求者兴趣相似度等因素相关。

## 缓存新鲜度

### 文档过期

就像牛奶过期一样，文档也有过期时间。

```
HTTP/1.1 200 OK
Content-Type: text/plain
Cache-Control: max-age=484200
```

```
HTTP/1.1 200 OK
Content-Type: text/plain
Expires: Fri, 05 Jul 2017, 12:00:00 GMT
```

`Cache-Control: max-age=484200`是一个相对过期时间，max-age定义了文档的最大使用期，从第一次生成文档到文档不再新鲜为止，以秒为单位。

`Expires:Fri, 05 Jul 2017, 12:00:00 GMT`是一个绝对过期时间，如果过期时间已经过了，则文档不再新鲜。该首部要**时钟同步**。

### 服务器再验证

缓存文档过期并不意味着该副本与服务器文档不一致，只是意味着要与源服务器进行再验证。

- 如果再验证内容发生了变化，缓存获取新的副本，替换过期副本
- 如果再验证内容没有发生变化，缓存只需要获取新的首部，对缓存的副本的首部进行更新

条件再验证HTTP首部：

| Header           | Description                                                  |
| ---------------- | ------------------------------------------------------------ |
| If-Modify-Since: | 如果从指定日期之后文档被修改过了，就执行该请求。可以与Last-Modified服务器响应首部配合使用，只有在内容被修改后，才去获取新的内容 |
| If-None-Match:   | 服务器可以为文档提供特殊的标签，而不是将其与最近修改时间相匹配，这些标签就像序列号一样 |

If-Modify-Since: Date再验证:

- 如果自指定日期后，文档被修改了，If-Modify-Since条件为真，源服务器会返回成功的响应，包含新的过期首部和新文档实体
- 如果自指定日期后，文档未被修改，If-Modify-Since条件为假，源服务器会返回一个304 Not Modified的响应，不包含文档实体内容

[![img](http://www.processon.com/chart_image/597c3916e4b06e43d2d6618c.png)](http://www.processon.com/chart_image/597c3916e4b06e43d2d6618c.png)

If-None-Match: Tags

有些情况下，仅使用最后修改时间是不够的。

- 文档被周期性的重写，最后修改时间发生变化，而内容未改变
- 服务器无法判定最后修改时间

HTTP允许用户对实体打标签，进行标识。

[![img](http://www.processon.com/chart_image/597c40b1e4b06b35d2fa3f2e.png)](http://www.processon.com/chart_image/597c40b1e4b06b35d2fa3f2e.png)

什么时候使用最后修改时间和标签验证？

如果服务器返回了ETag首部，客户端必须使用标签验证。如果服务器只返回了Last-Modified首部，客户端可以使用最后修改时间验证。

### 新鲜度计算算法

为了分辨文档是否新鲜，需要计算两个值，文档的使用期(age)和文档的新鲜生存期(freshness lifetime)。如果文档使用期小于文档新鲜生存期，则文档是新鲜的。

```
is_fresh_enough = True if age < freshness_lifetime
```

#### 使用期

使用期包含了网络传输时间和文档在缓存的停留时间。

```
apparent_time = time_got_response - date_header_value
corrent_apparent_time = max(0, apparent_time)
age_when_document_arrived_at_our_cache = corrent_apparent_time

how_long_copy_has_been_in_our_cache = current_time - got_response_time

age = age_when_document_arrived_at_our_cache + how_long_copy_has_been_in_our_cache
```

基于Date首部计算apparent使用期

apparen时间：

apparent时间等于获得响应时间减去服务器发送文档时间：

```
apparent_time = time_got_response - date_header_value
```

为了防止由于服务器时间不同步导致apparent_time为负，进行时间修正：

```
apparent_time = max(0, apparent_time)
age_when_document_arrived_at_our_cache = apparent_time
```

对网络时延对补偿：

**如果文档在网络或服务器中阻塞了很长时间，相对使用期的计算可能会极大的低估文档使用期。缓存知道文档请求时间，以及文档到达时间。HTTP/1.1会在这些网络延迟上加上整个网络时延，一遍对其进行保守校正。这个从缓存到服务器到缓存高估了服务器到缓存延迟，它是保守的。如果出错，只会使文档看起来比实际使用期要老，并会引发不必要的验证。**

```
response_delay_estimate = time_got_response - time_issued_request
age_when_document_arrived_at_our_cache = apparent_time + response_delay_estimate
```

> Note:该时延补偿会导致最后计算文档使用期大于实际的文档使用期。apparent_time是包含网络时延的，对网络时延补偿是否必要？在服务器负载较高，对服务器的处理时间进行补偿倒是很有必要。

缓存停留时间：

```
how_long_copy_has_been_in_our_cache = current_time - got_response_time
```

[![img](http://www.processon.com/chart_image/597c52a3e4b06e43d2d66a4e.png)](http://www.processon.com/chart_image/597c52a3e4b06e43d2d66a4e.png)

#### 新鲜生存期

```
def calculate_server_freshness_limit(**kwargs):
	heuristic = False
	server_freshness_limit = default_cache_min_lifetime
    if max_age_value_set:
    	server_freshness_limit = max_age_value_set
    elif expires_value_set:
   		server_freshness_limit = expires_value_set - date_value
    elif last_modified_value_set:
    	time_since_last_modify = max(0, date_value - last_modified_value)
        server_freshness_limit = int(time_since_last_modify*lm_factor)
        heuristic = True
    else:
    	server_freshness_limit = default_cache_min_lifetime
        heuristic = True

	if heuristic:
    	if server_freshness_limit > default_cache_max_lifetime:
        	server_freshness_limit = default_cache_max_lifetime
        if server_freshness_limit < default_cache_min_lifetime:
        	server_freshness_limit = default_cache_min_lifetime
            
	return server_freshness_limit
```

LM-factor算法计算新鲜周期

- 如果已缓存文档最后一次修改发生在很久以前，它可能是一份稳定的文档，不会突然发生变化，因此将其汲取保存在缓存中比较安全
- 如果已缓存的文档最近被修改过，就说明它很可能会频繁发生变化，因此在与服务器再验证之前，只应该将其缓存很短一段时间

```
time_since_modify = max(0, date_value - server_last_modified)
server_freshness_limit = time_since_modify * lm_factor
```

[![img](http://www.processon.com/chart_image/597c9f3be4b06e43d2d6757c.png)](http://www.processon.com/chart_image/597c9f3be4b06e43d2d6757c.png)

## 其他

缓存相关的HTTP头部：

| Header          | Description                    |
| --------------- | ------------------------------ |
| Cache-Control   | 缓存控制                       |
| Expires         | 过期绝对时间                   |
| If-Modify-Since | 从某个时间开始文档是否发生改变 |
| If-None-Match   | 文档的标签是否发生改变         |
| Last-Modified   | 最后修改时间                   |
| ETag            | 文档标签                       |

## 参考

- <https://developer.mozilla.org/en-US/docs/Web/HTTP/Caching>
- <https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Cache-Control>
- <https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Expires>
- <https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Last-Modified>
- [rfc7234 Hypertext Transfer Protocol (HTTP/1.1): Caching](https://tools.ietf.org/html/rfc7234)
- [rfc5861 HTTP Cache-Control Extensions for Stale Content](http://blog.mengyangyang.org/2017/07/29/http-cache/HTTP%20Cache-Control%20Extensions%20for%20Stale%20Content)
- [Caching Tutorial](https://www.mnot.net/cache_docs/)
- [redbot](https://redbot.org/), a tool to check your cache-related HTTP headers.
- [Hypertext Transfer Protocol – HTTP/1.1](https://www.ietf.org/rfc/rfc2616.txt)