---
title: Python元类
date: 2017-07-18 22:29:23
tags: [Python]
---

*[Metaclasses] are deeper magic than 99% of users should ever worry about. If you wonder whether you need them, you don’t (the people who actually need them know with certainty that they need them, and don’t need an explanation about why).* The Python core developer Time Peters said.

元类是创建类的类。新风格(new-style)类是type类的实例。元类是type类的派生类，通过重载type类的`__new__`和`__init__`方法，重新定义类创建协议，来实现类创建的定制化。

## 元类模型

(对象)实例是通过类创建，类是通过type类创建，元类是type类的派生类。

- 类型(自定义)是通过type类或type派生元类创建
- 元类是type类的派生类
- 用户自定义类是type类的实例
- 用户自定义类可以生成自己的实例

### 类声明协议

Python解释器在类声明语句结束时，调用type类型创建，`class = type(classname, superclasses, attributedict)`。

```python
type.__new__(meta, classname, superclasses, attributedict)
type.__init__(cls, classname, superclasses, attributedict)
```

## 声明元类

Py3与Py2声明元类的方式不一样：

Py3声明元类

```python
>>> class Metaclass(type):
...     def __new__(meta, classname, superclasses, attributedict):
...         return type(classname, superclasses, attributedict)
...     def __init__(cls, classname, superclasses, attributedict):
...         pass
... 
>>> class Dummy(object, metaclass=Metaclass):
...     pass
...
```

Py2声明元类

```python
>>> class Metaclass(type):
...     def __new__(meta, classname, superclasses, attributedict):
...         return type(classname, superclasses, attributedict)
...     def __init__(cls, classname, superclasses, attributedict):
...         pass
... 
>>> class Dummy(object):
...     __metaclass__ = Metaclass
...
```

## 继承和实例

- 元类继承于type类
- 元类的声明可以被派生类继承
- 元类的属性不能被类的实例继承
- 元类的属性可以被类获取

**元类继承于type类**

元类重载type类的`__new__`和`__init__`方法，定制类的创建和初始化。

**元类的声明可以被派生类继承**

**元类的属性不能被类实例继承**

类是元类的实例，元类的行为可以被类访问，当类不能被类的实例访问。

**元类的属性可以被类获取**

### 继承

**Python继承算法**

1. 对于一个实例，先搜索这个实例，再搜索实例的类，再搜索超类
   a. 先搜索实例的`__dict__`
   b. 再搜索该实例的类的`__mro__`对应类的`__dict__`
2. 对于一个类，先搜索类，再搜索超类，再搜索元类
   a. 根据`__mro__`搜索类的`__dict__`
   b. 再搜索元类的`__dict__`
3. 规则1和2中，再b阶段中，数据描述的优先级高
4. 规则1和2中，对于内置的运算符，跳过a阶段

**数据描述符继承算法**

对于定义了`__set__`拦截赋值的描述符就是数据描述符。

```python
>>> class D(object):
...     def __get__(self, ins, _type):
...             print('call D.__get__')
...     def __set__(self, ins, value):
...             print('call D.__set__')
... 
>>> 
>>> class Dummy(object):
...     d = D()
... 
>>> ins = Dummy()
>>> ins.d
call D.__get__
>>> ins.d = 'spam'
call D.__set__
```

未定义`__set__`的描述符

```python
>>> class D(object):
...     def __get__(self, ins, value):
...             print('call D.__get__')
... 
>>> class Dummy(object):
...     d = D()
... 
>>> ins = Dummy()
>>> ins.d
call D.__get__
>>> ins.d = 'spam'
>>> ins.d
'spam
```

Python 的继承算法

1. 对于实例I，先搜索实例，再搜索类，再搜索超类
   a. 根据类的`__mro__`搜索超类的`__dict__`
   b. If 如果在a阶段发现了数据描述，调用该数据描述，完成后退出
   c. Else 返回该实例`__dict__`中的值
   d. Else 调用非数据描述符，并放回结果
2. 对于类C，搜索类，再搜索超类，再搜索元类
   a. 搜索类的`__mro__`依次搜索类的`__dict__`
   b. If 如果在a阶段发现了数据描述符，调用该数据描述符，完成后退出
   c. Else 返回该类`__dict__`中值g
   d. Else 调用非数据描述符，返回结果

> Note, 数据描述符的优先级 > 普通属性 > 非数据描述符

## 元类与类装饰器

TODO

## 示例

django ORM

ripozo API

## 参考

- *Learning Python 5th Edition*