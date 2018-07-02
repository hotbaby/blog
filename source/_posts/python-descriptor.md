---
title: Python描述符
date: 2017-07-10 22:28:07
tags: [Python]
toc: true
comments: true
---

如果一个对象定义了以下任意方法，这个对象就是一个描述符。给描述符下个定义，描述符就是绑定了行为属性的对象。

`object.__get__(self, instance, owner)`

`object.__set__(self, instance, value)`

`object.__delete__(self, instance)`

属性访问的默认行为就是从一个对象字典中获取、设置和删除属性。比如，`a.x`首先会搜索`a.__dict__['x']`，其次`type(a).__dict__['x']`，最后所有`type(a)`的元类。**如果要查找的值一个包含描述器方法的对象，Python会用调用描述器方法代替默认行为。**

> Note:只有new-style class会调用描述符的对象的方法。

描述符是一个强大的通用协议。Python内建的property, staticmethod, classmethod, super背后的实现机制都是描述符协议。

## Descriptor Protocol

`object.__get__(self, ins, _type=None)`

`object.__set__(self, ins, value)`

`object.__del__(self, ins)`

如果一个对象包含上面任意一个方法，就可以被看作是一个描述符。如果一个对象定义了`__get__`和`__set__`两个方法，该对象可以被看作一个数据描述符，如果一个对象只定义了`__get__`，该对象就是non-data描述符。

**数据描述符与非数据描述符的区别在于，描述符与对象实例entry调用优先级。如果一个实例的字典有一个entry和数据描述符的名字相同，数据描述符的调用的优先级高。如果一个实例有一个entry和非数据描述符的名字相同个，实例entry的调用的优先级高。**

## Invoking Descriptors

`obj.d`查找obj的字典是否包含d，如果d定义了`__get__`方法，`d.__get__(obj, type(obj))`就会被调用。

```python
def __getattribute__(self, key):
    "Emulate type_getattro() in Objects/typeobject.c"
    v = object.__getattribute__(self, key)
    if hasattr(v, '__get__'):
        return v.__get__(None, self)
    return v
```

super()返回的对象有一个定制化的`__getattribute__`方法，用于调用描述符。`super(B, self).m`先会搜索`obj.__class__.__mro__`查找基类A，如果是一个数据描述符，则会调用`A.__dict__['m'].__get__(obj, B)`，如果是一个非数据描述符，返回结果不会改变。

## Built-in Descriptors

### Property

```python
class C(object):
    def getx(self): return self.__x
    def setx(self, value): self.__x = value
    def delx(self): del self.__x
    x = property(getx, setx, delx, "I'm the 'x' property.")


class Property(object):
    "Emulate PyProperty_Type() in Objects/descrobject.c"

    def __init__(self, fget=None, fset=None, fdel=None, doc=None):
        self.fget = fget
        self.fset = fset
        self.fdel = fdel
        if doc is None and fget is not None:
            doc = fget.__doc__
        self.__doc__ = doc

    def __get__(self, obj, objtype=None):
        if obj is None:
            return self
        if self.fget is None:
            raise AttributeError("unreadable attribute")
        return self.fget(obj)

    def __set__(self, obj, value):
        if self.fset is None:
            raise AttributeError("can't set attribute")
        self.fset(obj, value)

    def __delete__(self, obj):
        if self.fdel is None:
            raise AttributeError("can't delete attribute")
        self.fdel(obj)

    def getter(self, fget):
        return type(self)(fget, self.fset, self.fdel, self.__doc__)

    def setter(self, fset):
        return type(self)(self.fget, fset, self.fdel, self.__doc__)

    def deleter(self, fdel):
        return type(self)(self.fget, self.fset, fdel, self.__doc__)
```

### Staticmethod

```python
class StaticMethod(object):
    "Emulate PyStaticMethod_Type() in Objects/funcobject.c"

    def __init__(self, f):
        self.f = f

    def __get__(self, obj, objtype=None):
        return self.f
```

### Classmethod

```python
class ClassMethod(object):
    "Emulate PyClassMethod_Type() in Objects/funcobject.c"

    def __init__(self, f):
        self.f = f

    def __get__(self, obj, klass=None):
        if klass is None:
            klass = type(obj)
        def newfunc(*args):
            return self.f(klass, *args)
        return newfunc
```

## References

- <https://docs.python.org/2/reference/datamodel.html#descriptors>
- <https://docs.python.org/2/howto/descriptor.html>