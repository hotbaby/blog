---
title: Python Itertools
date: 2018-07-01 21:57:10
tags: [Python]
---

#### itertools.imap()

`imap(func, *iterables)` 和iter和map混合体。

工作原理

```python
>>> def myimap(func, *iterables):
	iterables = map(iter,iterables)
	while True:
		args = [next(it) for it in iterables]
		yield func(*args)

		
>>> [item for item in myimap(pow, (2,2), (3,3))]
[8, 8]
```

#### itertools.chain()

`chain(*iterables)` 返回一个迭代器，该迭代器依次返回可迭代对象中没一个元素。

工作原理

```python
>>> def mychain(*iterables):
	for iter_ in iterables:
		for item in iter_:
			yield item
			
>>> [item for item in mychain('abc', 'def')]
['a', 'b', 'c', 'd', 'e', 'f']
```

## References

- <https://docs.python.org/2/library/functions.html>
- <https://docs.python.org/2/library/functools.html>
- <https://docs.python.org/2/library/itertools.html>

