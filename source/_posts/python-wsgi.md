---
title: Python WSGI
date: 2018-05-17 22:17:37
tags: [Python, WSGI]
---

WSGI(Web Server Gateway Interface) Web服务网关接口。

WSGI的目的替代CGI。CGI进程（类似Python解释器）针对每个请求进行创建，完成请求后退出。如果应程序接收树钱个请求，创建大量的语言解释器进程就会很快导致服务器宕机。

**其目标在Web服务器与Web框架层之间提供一个通用的API标准，减少之间互操作性，并形成统一的调用方式。**

## WSGI 应用

根据WSGI的定义，其应用是可调用的对象，其参数固定为两个：

- 含有服务器环境变量的字典
- 可调用的对象， 该对象使用HTTP状态码和会返回客户端的HTTP头来初始化响应

```python
def simple_wsgi_app(environment, start_response):
    status = '200 OK'
    headers = ['Content-Type': 'text/plain']
    start_response(status, headers)
    return ['Hello world!']
```

**environment** 包含一些环境变量，如HTTP_HOST, HTTP_USER, HTTP_AGENT, SERVER_PROTOCOL等。‘

**start_response()**是一个可调用的对象，必须在应用执行，生成最终发送回客户端的响应

werkzeug中start_response定义：

```python
def start_response(status, response_headers, exc_info=None):
    if exc_info:
        try:
            if headers_sent:
                reraise(*exc_info)
        finally:
            exc_info = None
    elif headers_set:
        raise AssertionError('Headers already set')
    headers_set[:] = [status, response_headers]
    return write
```

## WSGI服务器

```python
import StringIO

def run_wsgi_app(app, environment):
    body = StringIO.StringIO()

    def start_response(status, headers):
        body.write('Status: %s\r\n' % status)
        for header in headers:
            body.write('%s: %s\r\n' % header)
        return body.write

    iterable = app(environment, start_response)
    try:
        body.write('\r\n%s\r\n' % '\r\n'.join(line for line in iterable))
    finally:
        if hasattr(iterable, 'close') and callable(iterable.close):
            iterable.close()
```

## Flask WSGI 应用和服务实现

**flask WSGI app**

```python
# Flask:wsgi_app

def wsgi_app(self, environ, start_response):
    """The actual WSGI application.  This is not implemented in
    `__call__` so that middlewares can be applied without losing a
    reference to the class.  So instead of doing this::

        app = MyMiddleware(app)

    It's a better idea to do this instead::

        app.wsgi_app = MyMiddleware(app.wsgi_app)

    Then you still have the original application object around and
    can continue to call methods on it.

    .. versionchanged:: 0.7
       The behavior of the before and after request callbacks was changed
       under error conditions and a new callback was added that will
       always execute at the end of the request, independent on if an
       error occurred or not.  See :ref:`callbacks-and-errors`.

    :param environ: a WSGI environment
    :param start_response: a callable accepting a status code,
                           a list of headers and an optional
                           exception context to start the response
    """
    ctx = self.request_context(environ)
    ctx.push()
    error = None
    try:
        try:
            response = self.full_dispatch_request()
        except Exception as e:
            error = e
            response = self.handle_exception(e)
        return response(environ, start_response)
    finally:
        if self.should_ignore_error(error):
            error = None
        ctx.auto_pop(error)

# Flask:__call__

def __call__(self, environ, start_response):
    """Shortcut for :attr:`wsgi_app`."""
    return self.wsgi_app(environ, start_response)

# BaseResponse:__call__

def __call__(self, environ, start_response):
    """Process this response as WSGI application.

    :param environ: the WSGI environment.
    :param start_response: the response callable provided by the WSGI
                           server.
    :return: an application iterator
    """
    app_iter, status, headers = self.get_wsgi_response(environ)
    start_response(status, headers)
    return app_iter
```

**flask WSGI Server**

```python
def run_wsgi(self):
    if self.headers.get('Expect', '').lower().strip() == '100-continue':
        self.wfile.write(b'HTTP/1.1 100 Continue\r\n\r\n')

    self.environ = environ = self.make_environ()
    headers_set = []
    headers_sent = []

    def write(data):
        assert headers_set, 'write() before start_response'
        if not headers_sent:
            status, response_headers = headers_sent[:] = headers_set
            try:
                code, msg = status.split(None, 1)
            except ValueError:
                code, msg = status, ""
            self.send_response(int(code), msg)
            header_keys = set()
            for key, value in response_headers:
                self.send_header(key, value)
                key = key.lower()
                header_keys.add(key)
            if 'content-length' not in header_keys:
                self.close_connection = True
                self.send_header('Connection', 'close')
            if 'server' not in header_keys:
                self.send_header('Server', self.version_string())
            if 'date' not in header_keys:
                self.send_header('Date', self.date_time_string())
            self.end_headers()

        assert isinstance(data, bytes), 'applications must write bytes'
        self.wfile.write(data)
        self.wfile.flush()

    def start_response(status, response_headers, exc_info=None):
        if exc_info:
            try:
                if headers_sent:
                    reraise(*exc_info)
            finally:
                exc_info = None
        elif headers_set:
            raise AssertionError('Headers already set')
        headers_set[:] = [status, response_headers]
        return write

    def execute(app):
        application_iter = app(environ, start_response)
        try:
            for data in application_iter:
                write(data)
            if not headers_sent:
                write(b'')
        finally:
            if hasattr(application_iter, 'close'):
                application_iter.close()
            application_iter = None

    try:
        execute(self.server.app)
    except (socket.error, socket.timeout) as e:
        self.connection_dropped(e, environ)
    except Exception:
        if self.server.passthrough_errors:
            raise
        from werkzeug.debug.tbtools import get_current_traceback
        traceback = get_current_traceback(ignore_system_exceptions=True)
        try:
            # if we haven't yet sent the headers but they are set
            # we roll back to be able to set them again.
            if not headers_sent:
                del headers_set[:]
            execute(InternalServerError())
        except Exception:
            pass
        self.server.log('error', 'Error on request:\n%s',
                        traceback.plaintext)
```

## References

- 《Python核心编程》
- <https://docs.python.org/2/howto/webservers.html#step-back-wsgi>