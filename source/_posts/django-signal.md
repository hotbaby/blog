---
title: Django信号
date: 2017-04-20 22:07:38
tags: [Django, Python]
toc: true
comments: true
---

信号实现了一个复杂系统中子系统之间的解耦，一个子系统的状态发生改变时，通过信号同步（或通知）其他依赖于该系统的系统更新状态，实现了状态的一致性。

不同于flask，直接使用blinker，django内部实现了信号处理机制。

**实现原理**

定义信号

```python
def __init__(self, providing_args=None, use_caching=False):
    """
    Create a new signal.

    providing_args
        A list of the arguments this signal can pass along in a send() call.
    """
    self.receivers = []
    if providing_args is None:
        providing_args = []
    self.providing_args = set(providing_args)
    self.lock = threading.Lock()
    self.use_caching = use_caching
    # For convenience we create empty caches even if they are not used.
    # A note about caching: if use_caching is defined, then for each
    # distinct sender we cache the receivers that sender has in
    # 'sender_receivers_cache'. The cache is cleaned when .connect() or
    # .disconnect() is called and populated on send().
    self.sender_receivers_cache = weakref.WeakKeyDictionary() if use_caching else {}
    self._dead_receivers = False
```

连接信号

```python
def connect(self, receiver, sender=None, weak=True, dispatch_uid=None):
   from django.conf import settings

    # If DEBUG is on, check that we got a good receiver
    if settings.configured and settings.DEBUG:
        assert callable(receiver), "Signal receivers must be callable."

        # Check for **kwargs
        if not func_accepts_kwargs(receiver):
            raise ValueError("Signal receivers must accept keyword arguments (**kwargs).")

    if dispatch_uid:
        lookup_key = (dispatch_uid, _make_id(sender))
    else:
        lookup_key = (_make_id(receiver), _make_id(sender))

    if weak:
        ref = weakref.ref
        receiver_object = receiver
        # Check for bound methods
        if hasattr(receiver, '__self__') and hasattr(receiver, '__func__'):
            ref = WeakMethod
            receiver_object = receiver.__self__
        if sys.version_info >= (3, 4):
            receiver = ref(receiver)
            weakref.finalize(receiver_object, self._remove_receiver)
        else:
            receiver = ref(receiver, self._remove_receiver)

    with self.lock:
        self._clear_dead_receivers()
        for r_key, _ in self.receivers:
            if r_key == lookup_key:
                break
        else:
            self.receivers.append((lookup_key, receiver))
        self.sender_receivers_cache.clear()
```

发送信号

```python
def send(self, sender, **named):
    responses = []
    if not self.receivers or self.sender_receivers_cache.get(sender) is NO_RECEIVERS:
        return responses

    for receiver in self._live_receivers(sender):
        response = receiver(signal=self, sender=sender, **named)
        responses.append((receiver, response))
    return responses
```

断开连接

```python
def disconnect(self, receiver=None, sender=None, weak=True, dispatch_uid=None):
    if dispatch_uid:
        lookup_key = (dispatch_uid, _make_id(sender))
    else:
        lookup_key = (_make_id(receiver), _make_id(sender))

    disconnected = False
    with self.lock:
        self._clear_dead_receivers()
        for index in range(len(self.receivers)):
            (r_key, _) = self.receivers[index]
            if r_key == lookup_key:
                disconnected = True
                del self.receivers[index]
                break
        self.sender_receivers_cache.clear()
    return disconnected
```

信号发生时序图

TODO

weakref

为了防止调用被释放了对象，信号内部保持对receiver调用对象的弱引用，每次发送信号之前，检查该对象是否存在，如果不存在，则标记`_dead_receivers`为True,等待清楚。

thread lock

使用线程锁保证线程安全

**观察者设计模式**

定义对象间一对多的依赖关系，当一个对象的状体发生改变时，所有依赖于它的对象都得到通知并被自动更新。

将一系统分割成一系列相互协作的类有一个副作用：需要维护对象间的一致性。我们不希望为了维持一致性而使各类聚合，这样就降低了可重用性。

observer模式描述了如何建立这种关系。这一模式的关键对象是目标（subject）和观察者（observer）.一个目标可以有任意数量的观察者。一旦目标的状态发生改变，所有的观察者都得到通知。作为对这个通知的响应，每个观察者都将查询目标以使其状态与目标的状态同步。

这种交互也成为发布-订阅（publish-subscribe）.目标是通知的发布者，可以有任意数目的观察者订阅并接收通知。

`/django/dispatch/dispatcher.py`

**References**

- [python2.7 weakref](https://docs.python.org/2/library/weakref.html)
- [python3.4 weakref](https://docs.python.org/3.4/library/weakref.html)
- *Design Patterns - Elements of Reusable Object-Oriented Software Observer*