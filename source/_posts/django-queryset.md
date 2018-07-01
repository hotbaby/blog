---
title: Django QuerySet
date: 2017-10-23 16:41:04
tags: Django
---

### 字段查找过滤

| 操作符        | 含义                                                         |
| ------------- | ------------------------------------------------------------ |
| __exact       | 精确等于 like ‘aaa’                                          |
| __iexact      | 精确等于 忽略大小写 ilike ‘aaa’                              |
| __contains    | 包含 like ‘%aaa%’                                            |
| __icontains   | 包含 忽略大小写 ilike ‘%aaa%’，但是对于sqlite来说，contains的作用效果等同于icontains。 |
| __gt          | 大于                                                         |
| __gte         | 大于等于                                                     |
| __lt          | 小于                                                         |
| __lte         | 小于等于                                                     |
| __in          | 存在于一个list范围内                                         |
| __startswith  | 以…开头                                                      |
| __istartswith | 以…开头 忽略大小写                                           |
| __endswith    | 以…结尾                                                      |
| __iendswith   | 以…结尾，忽略大小写                                          |
| __range       | 在…范围内                                                    |
| __year        | 日期字段的年份                                               |
| __month       | 日期字段的月份                                               |
| __day         | 日期字段的日                                                 |

## Reference

* [Django QuerySet API Reference](https://docs.djangoproject.com/en/1.7/ref/models/querysets/)