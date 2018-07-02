---
title: Django中间件
date: 2017-03-30 21:59:10
tags: [Django, Python]
toc: true
comments: true
---

中间件是django处理request/response钩子的框架。它是一个用来修改输入、输出的轻量级的插件系统。 从另外角度上讲，中间件也是一种特殊的“装饰器”，装饰所有的视图函数。

## 版本说明

django中间件新版本与旧版本不兼容， 本文档是基于django 1.10编写。

**新版本中间件**

```python
class NewMiddlewareCls(object):
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        response = self.get_response(request)
        return response
```

**旧版本中间件**

```python
class OldMiddlewarCls(object):
    def __init__(self): pass
```

## 框架

### 中间件示例

```python
from django.utils.depreaction import MiddlewareMixin

class ExampleMiddleware(MiddlewareMixin):
    def process_request(self, request):
        return None

    def process_exceptions(self, request, exception):
        return None # or return HttpResponse()

    def process_response(self, request, response):
        return response
```

### 加载中间件

`WSGIHandler`在初始化时加载中配置文件中的中间件

```python
class BaseHandler(object):
    ...
    def load_middleware(self):
        if settings.MIDDLEWARE is None:
            ...
        else:
            handler = convert_exception_to_response(self._get_response)
            for middleware_path in reversed(settings.MIDDLEWARE):
                middleware = import_string(middleware_path)
                try:
                    mw_instance = middleware(handler)
                except MiddlewareNotUsed as exc:
                    if settings.DEBUG:
                        if six.text_type(exc):
                            logger.debug('MiddlewareNotUsed(%r): %s', middleware_path, exc)
                        else:
                            logger.debug('MiddlewareNotUsed: %r', middleware_path)
                    continue

                if mw_instance is None:
                    raise ImproperlyConfigured(
                        'Middleware factory %s returned None.' % middleware_path
                    )

                if hasattr(mw_instance, 'process_view'):
                    self._view_middleware.insert(0, mw_instance.process_view)
                if hasattr(mw_instance, 'process_template_response'):
                    self._template_response_middleware.append(mw_instance.process_template_response)
                if hasattr(mw_instance, 'process_exception'):
                    self._exception_middleware.append(mw_instance.process_exception)

                handler = convert_exception_to_response(mw_instance)
class WSGIHandler(base.BaseHandler):
    request_class = WSGIRequest

    def __init__(self, *args, **kwargs):
        super(WSGIHandler, self).__init__(*args, **kwargs)
        self.load_middleware()
```

### 运行中间件

收到客户端发起的一个请求，调用所有注册的中间件。

```python
class WSGIHandler(base.BaseHandler):
    ...
    def __call__(self, environ, start_response):
        ...
        response = self.get_response(request)
        ...
    ...


class BaseHandler(object):
    ...
    def get_response(self, request):
        """Return an HttpResponse object for the given HttpRequest."""
        # Setup default url resolver for this thread
        set_urlconf(settings.ROOT_URLCONF)

        response = self._middleware_chain(request)

        # This block is only needed for legacy MIDDLEWARE_CLASSES; if
        # MIDDLEWARE is used, self._response_middleware will be empty.
        try:
            # Apply response middleware, regardless of the response
            for middleware_method in self._response_middleware:
                response = middleware_method(request, response)
                # Complain if the response middleware returned None (a common error).
                if response is None:
                    raise ValueError(
                        "%s.process_response didn't return an "
                        "HttpResponse object. It returned None instead."
                        % (middleware_method.__self__.__class__.__name__))
        except Exception:  # Any exception should be gathered and handled
            signals.got_request_exception.send(sender=self.__class__, request=request)
            response = self.handle_uncaught_exception(request, get_resolver(get_urlconf()), sys.exc_info())

        response._closable_objects.append(request)

        # If the exception handler returns a TemplateResponse that has not
        # been rendered, force it to be rendered.
        if not getattr(response, 'is_rendered', True) and callable(getattr(response, 'render', None)):
            response = response.render()

        if response.status_code == 404:
            logger.warning(
                'Not Found: %s', request.path,
                extra={'status_code': 404, 'request': request},
            )

        return response
```

### 中间件chain

所有的中间件通过`MiddlewareMixin`链接起来，形成`middleware_chain`,参考*设计模式 chain of responsibility职责链*。

```python
class MiddlewareMixin(object):
    def __init__(self, get_response=None):
        self.get_response = get_response
        super(MiddlewareMixin, self).__init__()

    def __call__(self, request):
        response = None
        if hasattr(self, 'process_request'):
            response = self.process_request(request)
        if not response:
            response = self.get_response(request)
        if hasattr(self, 'process_response'):
            response = self.process_response(request, response)
        return response
```

新版本middleware处理时序：

[![img](http://blog.mengyangyang.org/images/pasted-1.png)](http://blog.mengyangyang.org/images/pasted-1.png)

旧版版本middleware处理时序：
[![img](https://docs.djangoproject.com/en/1.9/_images/middleware.svg)](https://docs.djangoproject.com/en/1.9/_images/middleware.svg)

### 其他

如果`process_request`返回结果为不`None`, 则不再迭代调用下一个层中间件(或视图函数)。

中间件处理分为几个阶段：

- process_request
- process_view
- process_exception
- process_template_response
- process_exception
- process_response

## 参考

- [django middleware framework](https://docs.djangoproject.com/en/1.10/topics/http/middleware/)
- [DEP5 improved middleware](https://github.com/django/deps/blob/master/final/0005-improved-middleware.rst)
- *设计模式-可复用面向对象基础*