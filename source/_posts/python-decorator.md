---
title: Python装饰器
date: 2017-03-28 19:29:29
tags: [Python]
---

装饰器是一种设计模型（结构型模式），可以动态给一个对象添加一些额外的职责，而不用修改该对象的任何code。

比如，我们要给一个API增加权限认证，可以通过`auth_decorator()`装饰这个API，而不必修改每个API的代码；debug一个函数的耗时，可以实现一个`time_decorator()`装饰要这些函数，而不用修改这些函数的内部实现；给个`TextView()`增加滚动条的装饰器`ScrollDecorator()`；Flask使用`route()`装饰器进行路由的注册等等。

装饰器优点：

- 比静态继承更灵活。与静态继承相比，装饰器可以灵活向对象添加额外的责任
- 避免在层次结构高层的类有特多的特征。 可以定义一个简单的类，通过装饰器给他逐渐添加功能。

## 基本装饰器

Python从语法本身就支持装饰器，Python装饰器是一个可调用对象（比如，函数、类），接受一个函数对象作为输入，返回另外一个函数对象。

最简单的Python函数装饰器：

```python
>>> def decorator(f):
	def wrapper(*args, **kwargs):
		print('before call %s' % f.__name__)
		result = f(*args, **kwargs)
		print('after call %s' % f.__name__)
		return result
	return wrapper

>>> @decorator
def func(*args, **kwargs):
	print('call func')

	
>>> func()
before call func
call func
after call func
```

### 函数装饰器

函数装饰器，接受一个函数f作为输入，返回另外一个函数对象。

```python
'''
function decorator
'''
def time_cost_decorator(f):
    def wrapper(*args, **kwargs):
        start_time = time.time()
        rv = f(*args, **kwargs)
        end_time = time.time()
        delta = end_time - start_time
        print('time cost: %ds' %  delta)
        return rv
    return wrapper

@time_cost_decorator
def sleep_10s_func(*args, **kwargs):
    time.sleep(10)

>>> sleep_10s_func()
time cost: 10s
```

使用类实现函数装饰器，在对象初始化时接受一个函数f作为输入，在模拟调用`__call__`特殊方法中调用f函数

```python
# class decorator
class TimeCostDecorator(object):
    def __init__(self, f):
        self._f = f

    def __call__(self, *args, **kwargs):
        start_time = time.time()
        ret = self._f(*args, **kwargs)
        end_time = time.time()
        delta = end_time - start_time
        print('time cost %ds' % delta)
        return ret


@TimeCostDecorator
def sleep_5s_func():
    time.sleep(5)
```

### 类装饰器

类装饰器的与函数装饰器语法非常类似。类装饰器接受cls作为输入，返回另外一个cls。

类装饰器可以用来管理管理类，类实例的创建。

函数实现类装饰器

```python
>>> def class_decorator(cls):
	class ClassWrapper(object):
		def __init__(self, *args, **kwargs):
			self._ins = cls(*args, **kwargs)
		def __getattr__(self, name):
			print('call ClassWapper.__getattr__ func')
			return getattr(self._ins, name)
	return ClassWrapper

>>> @class_decorator
class Foo(object):
	def func(self):
		print('call Foo.func')

		
>>> Foo().func()
call ClassWapper.__getattr__ func
call Foo.func
```

类实现类装饰器

```python
class ClassDecorator(object):
	def __init__(self, cls):
		self._cls = cls
	def __call__(self, *args, **kwargs):
		self._ins = self._cls(*args, **kwargs)
		return self
	def __getattr__(self, name):
		print('call ClassDecorator.__getattr__')
		return getattr(self._ins, name)

	
>>> @ClassDecorator
class Foo(object):
	def func(self):
		print('call Foo.func func')

		
>>> Foo().func()
call ClassDecorator.__getattr__
call Foo.func func
>>>
```

## 装饰器进阶

### 带参数装饰器

不仅被装饰的函数可以携带参数，装饰器函数也可以携带参数。

Decorator params imply three levels of callables: a callable to accept decorator arguments, which return a callable to serve a callable to serve as decorator, which return a callbale to handle calls to the origin function or class. Each of the three levels may be a function or class and may retain state in the form of scopes or class attributes.

函数实现的带参数的函数装饰器：

以最近写的权限验证为例，描述带参数的装饰器实现

```
>>> def perm_required(perm, **options):
	def decorator(f):
		def wrapper(*args, **kwargs):
			return f(*args, **kwargs)
		return wrapper
	return decoraor
```

类实现的带参数的装饰器：

TODO

函数实现的带参数类装饰器：

TODO

类实现的带参数类装饰器：

TODO

### 函数签名

`functools.wraps()` 保留被封装函数的签名，如`__module__`, `__name__`, `__doc__`等

以下函数装饰器，被装饰的函数的签名被覆盖

```python
>>> def decorator(f):
	def wrapper(*args, **kwargs):
		return f(*args, **kwargs)
	return wrapper

>>> @decorator
def func(): pass

>>> func.__name__
'wrapper'
>>>
```

调用`functools.wraps()`之后

```python
>>> def decorator(f):
	@functools.wraps(f)
	def wrapper(*args, **kwargs):
		return f(*args, **kwargs)
	return wrapper

>>> 
>>> @decorator
def func(): pass

>>> func.__name__
'func'
```

`wraps()`函数的定义

```python
WRAPPER_ASSIGNMENTS = ('__module__', '__name__', '__doc__')
WRAPPER_UPDATES = ('__dict__')
def wraps(wrapped, assign=WRAPPER_ASSIGNMENTS, updated=WRAPPER_UPDATES):pass
```

### 嵌套装饰器

有时一个装饰器不能满足需求，这时，我们可以添加多个装饰器(decorator nesting)。

以下两个装饰器函数分别实现如下功能，`scroll_decorator()`添加滚动条，`border_decorator()`添加边框

```python
>>> def border_decorator(f):
	def wrapper(*args, **kwargs):
		print('border wrapper')
		return f(*args, **kwargs)
	return wrapper

>>> def scroll_decorator(f):
	def wrapper(*args, **kwargs):
		print('scroll decorator')
		return f(*args, **kwargs)
	return wrapper

>>> 
>>> @border_decorator
@scroll_decorator
def decorated_func(*args, **kwargs):
	print('call decorated func')

	
>>> 
>>> decorated_func()
border wrapper
scroll decorator
call decorated func
>>>
```

以上装饰器从语法与以下等价

```python
>>> def decorated_func(*args, **kwargs):
	print('decorated func')

>>> f = border_decorator(scroll_decorator(decorated_func))
>>> f()
border wrapper
scroll decorator
decorated func
```

## 装饰器的其他特性

### 装饰器与闭包

TODO

### 装饰器与描述符

TODO

## 装饰器使用场景

### 统计API的调用时间

TODO

### 单例设计模式

TODO

### 跟踪方法调用

TODO

### 其他用例

- Flask 路由管理
- Django model 事务管理
- Tornado 异步框架

## 参考

- <https://docs.python.org/2/library/functools.html#module-functools>
- <http://stackoverflow.com/questions/5929107/python-decorators-with-parameters>
- *Design Patterns - Elements of Resuable Object-Oriented Software*
- *Learning Python 5th Edition*