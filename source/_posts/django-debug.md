---
title: Django调试
date: 2017-03-30 22:01:27
tags: [Django]
---

django默认情况会将所有的错误信息以HTML形式返回给前端，这样导致在运行nose的单元测试用例出现错误时，无法看到详细的错误栈帧信息，给程序的debug带来一定的困扰。

如何debug django异常栈帧？

方法一:

配置`settings.py`,将django的所有的debug信息，错误栈帧信息输出到终端

```python
LOGGING = {
    'version': 1,
    'disable_existing_loggers': False,
    'handlers': {
        'console': {
            'class': 'logging.StreamHandler',
        },
    },
    'loggers': {
        'django': {
            'handlers': ['console'],
            'level': os.getenv('DJANGO_LOG_LEVEL', 'INFO'),
        },
    },
}
```

连接信号`got_request_exception`

```python
from django.core.signals import got_request_exception

def exc_cb(sender, **kwargs):
    import traceback
    traceback.print_exc()

got_request_exception.connect(exc_cb)
```

方法二： 通过middleware，在`process_exception`中增加exception的调试信息，并将此middleware增加到`settings.py`的`MIDDLEWARE_CLASSES`中。

```python
import traceback

class LogMiddleware(object):
    def process_exception(self, request, exception):
        traceback.print_exc()
```

**注: 在中间件process_exception方法中打印异常栈帧，只能debug view函数的异常。 如果异常发生在中间件中，无法打印异常信息。**

**参考**

- [django logging](https://docs.djangoproject.com/en/1.11/topics/logging/)
- [django signals](https://docs.djangoproject.com/en/1.11/topics/signals/)
- [logging](https://docs.python.org/2/library/logging.html)
- [logging.config](https://docs.python.org/2/library/logging.config.html)