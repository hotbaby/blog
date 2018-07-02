---
title: Django缓存
date: 2017-04-06 22:04:15
tags: [Python, Django, 缓存]
toc: true
comments: true
---

每次用户请求一个页面，Web服务器都要进行很多计算，查询数据库，合成模板，处理业务逻辑等，再将页面返回给用户．后续的相同资源的请求，服务器都需要重复这些计算．

django提供了缓存机制，每次将资源的响应的副本存储指定的位置，下次用户再发起相同的请求时，服务器不再需要进行类似的计算，直接将上次响应的副本返回给用户．这样既减少服务器的负载，又降低用户的请求时延，提高了用户体验．

## HTTP缓存介绍

TODO

## 缓存框架

缓存是django的一个核心组件，提供缓存服务。

### 缓存实现

[![img](https://github.com/hotbaby/cache/blob/master/django_cache/resources/django-cache.png?raw=true)](https://github.com/hotbaby/cache/blob/master/django_cache/resources/django-cache.png?raw=true)

**config cache**

```
settings.CACHES = {
    'default': {  
        'BACKEND':'django.core.cache.backends.memcached.MemcachedCache',
        'LOCATION': '127.0.0.1:11211',
    },
}
```

以memcached为例介绍缓存的配置，`BACKEND`是缓存实例实现类，`LOCATION`是缓存实例的位置．

#### CacheHandler

`CacheHandler`管理缓存实例的访问。

实现原理： 根据`settings.CACHES`中配置（运行时）创建缓存实例，重载`__getitem__`特殊方法管理缓存实例的访问，通过线程局部变量`thread.local()`保证对于`settings.CACHES`中每个缓存在每个线程中只有一个实例．

`django.core.cache.caches` 是`CacheHandler`的一个实例，`django.core.cache.cache`是`caches['default']`的代理．

#### 缓存backends

采用模板方法的设计模式，通过抽象基类`BaseCache`声明了一套缓存操作接口，而将接口实现延迟到具体的子类中。如图所示：
[![img](https://github.com/hotbaby/cache/blob/master/django_cache/resources/django-cache-backends.png?raw=true)](https://github.com/hotbaby/cache/blob/master/django_cache/resources/django-cache-backends.png?raw=true)

`BaseCache` 是一个抽象类，定义了缓存通用的操作接口和参数默认值．

其中重要的参数：

- `key` 生成机制
- `timeout` 超时时间
- `max_entries` 最大条目数
- `cull_frequency` 更新频率

`MemcachedCache`实现抽象类中声明的接口，通过`_cache`实现数据的设置和获取操作．

```python
@property
def _cache(self):
    """
    Implements transparent thread-safe access to a memcached client.
    """
    if getattr(self, '_client', None) is None:
        self._client = self._lib.Client(self._servers)

    return self._client
```

`LocMemCache`实现了线程安全本地内存缓存．

实现原理：`_cache`一个字典对象用于缓存内存对象，`_expire_info`一个字典对象用于缓存内存对象的过期时间，`_lock`是django内部实现一个读写锁保证线程安全．在存储和获取内存对象时，通过`pickle`进行对象的序列化和反序列化．

`FileBasedCache`实现了基于文件的缓存机制．

实现原理： 缓存对象，将过期时间和内存对象通过`pickle`序列化后写入本地文件系统．获取对象，检查是否需要删除就的对象，反序列化，检查是否过期，返回对象．

### 缓存中间件

如果使能缓存中间件，每个django的页面都会被缓存．

缓存中间件的工作原理（参考实现代码）：

- 只有状态码为200的，方法为HEAD,GET请求的响应被缓存
- 检测缓存中是否已缓存该请求的响应对象
- 如果命中，返回原始响应对象的一个浅拷贝（shallow copy）
- 如果未命中，继续处理view函数
- 根据请求的header决定是否需要缓存
- 设置响应的ETag, Last-Modified, Expires, Cache-Control HTTP header.

配置缓存中间件

```
settings.MIDDLEWARE = [
    'django.middleware.cache.UpdateCacheMiddleware',
    ...
    'django.middleware.cache.FetchFromCacheMiddleware',
]
```

**注意：**

在response阶段，中间件的处理顺序是bottom-top, `UPdateCacheMiddleware`必须最后被执行，因此放在靠前的位置．在request阶段，中间件的处理顺序是top-bottom, `FetchFromCacheMiddleware`必须最后被执行，因此放在靠后的位置．
[![img](https://github.com/hotbaby/cache/blob/master/django_cache/resources/django-middleware-ordering.png?raw=true)](https://github.com/hotbaby/cache/blob/master/django_cache/resources/django-middleware-ordering.png?raw=true)

#### `FetchFromCacheMiddleware`

```python
  def process_request(self, request):
      """
      Checks whether the page is already cached and returns the cached
      version if available.
      """
        # 只缓存HEAD, GET的请求
        if request.method not in ('GET', 'HEAD'):
          request._cache_update_cache = False
          return None  # Don't bother checking the cache.

# 获取GET方法的cache_key,如果不存在，则设置_cache_update_cache标志位为True，需要更新缓存
        # try and get the cached GET response
      cache_key = get_cache_key(request, self.key_prefix, 'GET', cache=self.cache)
      if cache_key is None:
          request._cache_update_cache = True
          return None  # No cache information available, need to rebuild.
      response = self.cache.get(cache_key)
        # 如果cache为命中，而且请求的方法为HEAD,则获取请求方法为HEAD的cache_key
        # if it wasn't found and we are looking for a HEAD, try looking just for that
      if response is None and request.method == 'HEAD':
          cache_key = get_cache_key(request, self.key_prefix, 'HEAD', cache=self.cache)
          response = self.cache.get(cache_key)

# 缓存都未命中，设置_cache_update_cache标志位为True,调用view函数，并更新缓存
        if response is None:
          request._cache_update_cache = True
          return None  # No cache information available, need to rebuild.

# 缓存命中，设置_cache_update_cahe标志为False, 不调用view函数，不更新缓存
        # hit, return cached response
      request._cache_update_cache = False
      return response
```

#### `UpdateCacheMiddleware`

```python
  def _should_update_cache(self, request, response):
      return hasattr(request, '_cache_update_cache') and request._cache_update_cache

  def process_response(self, request, response):
      """Sets the cache, if needed."""
        # 不缓存，直接返回Response
        if not self._should_update_cache(request, response):
          # We don't need to update the cache, just return.
          return response

# 如果是流数据或状态码不为200, 不缓存
        if response.streaming or response.status_code != 200:
          return response

# 如果是私有数据，不缓存
        # Don't cache responses that set a user-specific (and maybe security
      # sensitive) cookie in response to a cookie-less request.
      if not request.COOKIES and response.cookies and has_vary_header(response, 'Cookie'):
          return response

      # Try to get the timeout from the "max-age" section of the "Cache-
      # Control" header before reverting to using the default cache_timeout
      # length.
      timeout = get_max_age(response)
      if timeout is None:
          timeout = self.cache_timeout
      elif timeout == 0:
          # max-age was set to 0, don't bother caching.
          return response
      
        # 设置缓存的HTTP头部信息
        patch_response_headers(response, timeout)
        # 缓存响应
        if timeout:
          cache_key = learn_cache_key(request, response, timeout, self.key_prefix, cache=self.cache)
          if hasattr(response, 'render') and callable(response.render):
              response.add_post_render_callback(
                  lambda r: self.cache.set(cache_key, r, timeout)
              )
          else:
              self.cache.set(cache_key, response, timeout)
      return response
```

### 缓存装饰器

缓存装饰器用于控制view是否缓存.

#### nerver_cache

```python
def never_cache(view_func):
    @wraps(view_func, assigned=available_attrs(view_func))
    def _wrapped_view_func(request, *args, **kwargs):
        response = view_func(request, *args, **kwargs)
        add_never_cache_headers(response)
        return response
    return _wrapped_view_func
```

view装饰器，在view函数处理完成后，patch缓存相关的HTTP头，控制该相应不被缓存。

相关的HTTP headers:

- Last-Modifed: current_datetime
- Expires: current_datetime
- Cache-Control: max-age=0

#### cache_control

view装饰器，在view函数处理完成后，patch`Cache-Control`HTTP头。

#### cache_page

view装饰器，用于缓存页面。该装饰器是对cache中间件`CacheMiddleware`的封装。

## 参考

- [django’s cache framework](https://docs.djangoproject.com/en/1.11/topics/cache/#the-low-level-cache-api)
- [django redis](http://django-redis-chs.readthedocs.io/zh_CN/latest/)
- [rfc7234 Hypertext Transfer Protocol (HTTP/1.1): Caching](https://tools.ietf.org/html/rfc7234)
- [rfc5861 HTTP Cache-Control Extensions for Stale Content](http://blog.mengyangyang.org/2017/04/06/djaong-cache/HTTP%20Cache-Control%20Extensions%20for%20Stale%20Content)
- [django core cache code](https://github.com/django/django/tree/master/django/core/cache)
- [python pickle library](https://docs.python.org/2/library/pickle.html)
- [python threading.local](https://docs.python.org/2/library/threading.html#threading.local)
- [django middleware ordering](https://docs.djangoproject.com/en/1.10/ref/middleware/#middleware-ordering)