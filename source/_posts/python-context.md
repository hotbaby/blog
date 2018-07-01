---
title: Python上下文管理器
date: 2017-07-09 22:26:25
tags: [Python]
---

上下管理器是一个对象，定义了执行`with`语句时需要创建的上下文。context manager的`__enter__()`和`__exit__()`方法分别在进入、退出`with`语句时被调用。

`object.__enter__(self)`

`object.__exit__(self, exc_type, exc_value, traceback)`

## with statement

```python
with_stmt ::=  "with" with_item ("," with_item)* ":" suite
with_item ::=  expression ["as" target]
```

**with**语句执行数据流：

1. 评估上下文表达式是否包含上下文管理器
2. 加载上下文管理器的`__exit__`方法
3. 调用上下文管理的`__enter__`方法
4. 如果target包含在with语句中，将`__enter__`方法的返回值赋给target
5. 执行suite
6. 调用`__exit__()`方法

```python
mgr = (EXPR)
exit = type(mgr).__exit__  # Not calling it yet
value = type(mgr).__enter__(mgr)
exc = True
try:
    VAR = value  # Only if "as VAR" is present
    BLOCK
except:
    # The exceptional case is handled here
    exc = False
    if not exit(mgr, *sys.exc_info()):
    		raise
    # The exception is swallowed if exit() returns true
finally:
    # The normal and non-local-goto cases are handled here
    if exc:
        exit(mgr, None, None, None)
```

## contextlib

contextlib提供了快速定义支持上下文管理器的函数对象。

定义一个生成器函数，contextmanager装饰后就变成一个支持上下文管理器的函数对象。yield之前语句子在代码块之前被执行，yield之后语句在代码执行完之后被执行。

```python
>>> from contextlib import contextmanager
>>> 
>>> @contextmanager
... def tag(name):
...     print('<%s>' % name)
...     yield
...     print('</%s>' % name)
... 
>>> with tag('h1'):
...     print('hotbaby')
... 
<h1>
hotbaby
</h1>
```

## context decorator

contextmanager是一个函数装饰器，装饰只包含一个yield语句的生成器函数，返回一个支持上下文管理器的函数对象。

```python
def contextmanager(func):
    @wraps(func)
    def helper(*args, **kwds):
        return GeneratorContextManager(func(*args, **kwds))
    return helper
```

> Note: 被装饰的生成器函数变成生成器作为参数传递到**GeneratorContextManager**对象中。

生成器上下文管理器

```python
class GeneratorContextManager(object):
    """Helper for @contextmanager decorator."""

    def __init__(self, gen):
        self.gen = gen

    def __enter__(self):
        try:
            return self.gen.next()
        except StopIteration:
            raise RuntimeError("generator didn't yield")

    def __exit__(self, type, value, traceback):
        if type is None:
            try:
                self.gen.next()
            except StopIteration:
                return
            else:
                raise RuntimeError("generator didn't stop")
        else:
            if value is None:
                # Need to force instantiation so we can reliably
                # tell if we get the same exception back
                value = type()
            try:
                self.gen.throw(type, value, traceback)
                raise RuntimeError("generator didn't stop after throw()")
            except StopIteration, exc:
                # Suppress the exception *unless* it's the same exception that
                # was passed to throw().  This prevents a StopIteration
                # raised inside the "with" statement from being suppressed
                return exc is not value
            except:
                # only re-raise if it's *not* the exception that was
                # passed to throw(), because __exit__() must not raise
                # an exception unless __exit__() itself failed.  But throw()
                # has to raise the exception to signal propagation, so this
                # fixes the impedance mismatch between the throw() protocol
                # and the __exit__() protocol.
                #
                if sys.exc_info()[1] is not value:
                    raise
```

## references

- <http://hotbaby.org/python/2/reference/datamodel.html#with-statement-context-managers>
- <http://hotbaby.org/python/2/reference/compound_stmts.html#the-with-statement>
- <http://hotbaby.org/python/2/library/contextlib.html>
- <https://hg.python.org/cpython/file/2.7/Lib/contextlib.py>
- <https://www.python.org/dev/peps/pep-0343/>