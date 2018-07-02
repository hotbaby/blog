---
title: Python PyPI
date: 2017-06-08 22:22:06
tags: [Python, PyPI]
toc: true
---

PyPI(Python Package Index)是Python软件仓库。

pip是Python包管理工具，默认使用`pypi.python.org`作为PyPI的镜像。

pip安装Python软件包经常会出现”链接pypi.python.org失败”。为了优化包的管理，考虑替换`pypi.python.org`,转而使用国内的PyPI镜像。

**PyPI mirror list**

| Mirror          | Location                           |
| --------------- | ---------------------------------- |
| pypi.python.org | San Francisco, California US       |
| pypi.douban.com | Beijing, Beijing CN                |
| pypi.fcio.net   | Oberhausen, Nordrhein-Westfalen DE |

**Linux(Debian) 替换PyPI镜像**

`touch ~/.pip/pip.conf`

```
[global]
index-url = https://pypi.douban.com/simple
format = columns
```

**参考**

- <https://www.pypi-mirrors.org/>
- <https://zhuanlan.zhihu.com/p/21863043>

