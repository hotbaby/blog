---
title: Python Functools
date: 2017-03-29 21:53:49
tags: [Python]
toc: true
comments: true
---



#### `functools.wraps()`

`wraps(wrapped[,assigned][,updated])`
常用装饰器函数中返回的`wrapper()`函数，解决了被装饰函数__name__、__doc__等签名丢失问题。

```
>>> import functools
>>> def decorator(f):
	@functools.wrap(f)
	def wrapper(*args, **kwargs):
		return f(*args, **kwargs)
	return wrapper
```

#### `functools.partial()`

`partial(func[,*args][,**kwargs])`
是一个装饰器函数，也是一个闭包，返回一个可调用对象，freeze一些参数。

实现原理

```
def partial(func, *args, **keywords):
    def newfunc(*fargs, **fkeywords):
        newkeywords = keywords.copy()
        newkeywords.update(fkeywords)
        return func(*(args + fargs), **newkeywords)
    newfunc.func = func
    newfunc.args = args
    newfunc.keywords = keywords
    return newfunc
```

## References

- <https://docs.python.org/2/library/functions.html>
- <https://docs.python.org/2/library/functools.html>
- <https://docs.python.org/2/library/itertools.html>