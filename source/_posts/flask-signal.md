---
title: Flask信号
date: 2017-05-14 22:15:46
tags: [Python, Flask]
toc: true
comments: true
---

Flask signals 默认没有自己实现signal，而是使用blink进行信号的定义，连接，分发。

## Flask singal

flask未实现自己信号处理,而是使用blink.

```python
signals_available = False
try:
    from blinker import Namespace
    signals_available = True
except ImportError:
    class Namespace(object):
        def signal(self, name, doc=None):
            return _FakeSignal(name, doc)
```

flask 支持的信号

```python
_signals = Namespace()

# Core signals.  For usage examples grep the source code or consult
# the API documentation in docs/api.rst as well as docs/signals.rst
template_rendered = _signals.signal('template-rendered')
before_render_template = _signals.signal('before-render-template')
request_started = _signals.signal('request-started')
request_finished = _signals.signal('request-finished')
request_tearing_down = _signals.signal('request-tearing-down')
got_request_exception = _signals.signal('got-request-exception')
appcontext_tearing_down = _signals.signal('appcontext-tearing-down')
appcontext_pushed = _signals.signal('appcontext-pushed')
appcontext_popped = _signals.signal('appcontext-popped')
message_flashed = _signals.signal('message-flashed')
```

## Blinker signal

blinker支持的特性:

- a global registry of named signals
- anonymous signals
- custom name registries
- permanently or temporarily connected receivers
- automically disconnected receivers via weak referencing
- sending arbirary data payloads
- collecting return values from signal receivers
- thread safety

### Blinker signal sample

TODO

### Blinker signal realization

TODO

注册

弱引用

线程安全

## Problems

connecter weak reference

```python
def register_signal_handlers(app):
   import logging
   from flask.signals import request_finished
   _logger = logging.getLogger('api.debug')
   
   def log_response(sender, response, **options):
       print(response)
   request_finished.connect(log_response, app)
```

以上代码，不能按原意正确的运行。因为log_response是局部作用域函数，在函数调用完成后，该作用域会消失，因此不能正确的调用`log_response`函数。

## References

- <https://pythonhosted.org/blinker/>