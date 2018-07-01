---
title: Django异常处理
date: 2017-12-06 16:41:04
tags: Django
---

基于django技术栈实现的WEB应用，生产环境中都会关闭DEBUG选项，Django默认只会输出很少的错误信息，不利于开发人员快速定位、解决问题。为了解决此问题，考虑定制Django默认的错误处理，还原错误现场，配合错误日志、邮件报警快速发现、解决BUG。

> 本文档不包含错误日志、邮件报警等内容

## Django异常内容调用栈帧

### WSGI处理函数

`django.core.handlers.base.py`

```python
class BaseHandler(object):
    # Changes that are always applied to a response (in this order).
    ... ...
    
    def get_response(self, request):
        "Returns an HttpResponse object for the given HttpRequest"
        # Setup default url resolver for this thread, this code is outside
        # the try/except so we don't get a spurious "unbound local
        # variable" exception in the event an exception is raised before
        # resolver is set
        # 初始化路由解析器
        urlconf = settings.ROOT_URLCONF
        urlresolvers.set_urlconf(urlconf)
        resolver = urlresolvers.RegexURLResolver(r'^/', urlconf)
        
        try:
            response = None
            ... ...
        except:  # Handle everything else.
            # Get the exception info now, in case another exception is thrown later.
            signals.got_request_exception.send(sender=self.__class__, request=request)
            # 调用处理未捕获的异常方法
            response = self.handle_uncaught_exception(request, resolver, sys.exc_info())
       		... ...
	# 处理未捕获的异常
	def handle_uncaught_exception(self, request, resolver, exc_info):
        """
        Processing for any otherwise uncaught exceptions (those that will
        generate HTTP 500 responses). Can be overridden by subclasses who want
        customised 500 handling.
        Be *very* careful when overriding this because the error could be
        caused by anything, so assuming something like the database is always
        available would be an error.
        """
        if settings.DEBUG_PROPAGATE_EXCEPTIONS:
            raise
        logger.error('Internal Server Error: %s', request.path,
            exc_info=exc_info,
            extra={
                'status_code': 500,
                'request': request
            }
        )
        if settings.DEBUG:
            return debug.technical_500_response(request, *exc_info)
        # If Http500 handler is not installed, re-raise last exception
        if resolver.urlconf_module is None:
            six.reraise(*exc_info)
        # Return an HttpResponse that displays a friendly error message.
        # 查找注册的错误处理函数
        callback, param_dict = resolver.resolve_error_handler(500)
        # 调用错误处理函数
        return callback(request, **param_dict)
class WSGIHandler(base.BaseHandler):
      def __call__(self, environ, start_response):
    	... ...
        response = self.get_response(request)
```

### URL路由解析

`django.core.urlresolvers.py`

```python
class RegexURLPattern(LocaleRegexProvider):
    def __init__(self, regex, callback, default_args=None, name=None):
        LocaleRegexProvider.__init__(self, regex)
        # callback is either a string like 'foo.views.news.stories.story_detail'
        # which represents the path to a module and a view function name, or a
        # callable object (view).
        if callable(callback):
            self._callback = callback
        else:
            self._callback = None
            self._callback_str = callback
        self.default_args = default_args or {}
        self.name = name
        
    # 解析错误处理函数
    def resolve_error_handler(self, view_type):
        callback = getattr(self.urlconf_module, 'handler%s' % view_type, None)
        if not callback:
            # No handler specified in file; use default
            # Lazy import, since django.urls imports this file
            from django.conf import urls
            callback = getattr(urls, 'handler%s' % view_type)
        return get_callable(callback), {}
```

### 注册Django默认错误处理函数

`django.confs.urls.__init__.py`

```python
handler400 = 'django.views.defaults.bad_request'
handler403 = 'django.views.defaults.permission_denied'
handler404 = 'django.views.defaults.page_not_found'
handler500 = 'django.views.defaults.server_error'
```

### 定义Django默认错误处理函数

`django.views.defaults.py`

```python
@requires_csrf_token
def server_error(request, template_name='500.html'):
    """
    500 error handler.
    Templates: :template:`500.html`
    Context: None
    """
    try:
        template = loader.get_template(template_name)
    except TemplateDoesNotExist:
        return http.HttpResponseServerError('<h1>Server Error (500)</h1>', content_type='text/html')
    return http.HttpResponseServerError(template.render())
```

## 定制Django500异常处理

### 定制`WSGIHandler`

`project/wsgi.py`

```python
class CustomWSGIHandler(WSGIHandler):
    """
    定制WSGIHandler
    """
    def handle_uncaught_exception(self, request, resolver, exc_info):
        """
        重载夫类异常处理函数
        """
        _logger = logging.getLogger('app.wsgi')
        if settings.DEBUG_PROPAGATE_EXCEPTIONS:
            raise
        _logger.error('Internal Server Error: %s', request.path,
            exc_info=exc_info,
            extra={
                'status_code': 500,
                'request': request
            }
        )
        # 不再返回500的异常HTML报文
        # if settings.DEBUG:
        #     return debug.technical_500_response(request, *exc_info)
        # If Http500 handler is not installed, re-raise last exception
        if resolver.urlconf_module is None:
            six.reraise(*exc_info)
        # Return an HttpResponse that displays a friendly error message.
        callback, param_dict = resolver.resolve_error_handler(500)
        return callback(request, **param_dict)
def get_wsgi_application():
    django.setup()
    return CustomWSGIHandler()
```

### 定制500错误处理函数

`project/error_handlers.py`

```python
@requires_csrf_token
def server_error(request, template_name='500.html'):
    """
    定制500异常函数
    """
    _logger.error('server_error.')
    return http.HttpResponseServerError('Server Error')
```

### 注册500错误处理函数

`protject/urls.py`

```python
handler500 = 'error_handlers.server_error'
```

### 注册`ROOT_URLCONF`

`project/settings.py`

```python
ROOT_URLCONF = 'project.urls'
```

## 参考

- [Django Writing views](https://docs.djangoproject.com/en/1.11/topics/http/views/)