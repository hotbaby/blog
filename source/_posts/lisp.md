---
title:Lisp
date: 2017-06-01 22:21:04
tags: [Lisp]
---

Lisp语言从诞生的时候就包含9种思想.其中一些我们今天已经习以为常,另一些则刚刚在其他高级语言中出现,至今还有2种是Lisp独有的.按照大众的接受程度,这9种思想依次如下排列:

1. 条件结构.现在大家都觉得这是理所当然的,但是Fortran I就没有这个结构,它只有底层机器的goto结构.
2. 函数也是一种数据类型.在Lisp语言中,函数与整数或字符串一样,也属于数据类型的一种.它有自己的字面表示形式(literal representation),能狗存储在变量中,也能当作参数传递.一种数据类型应该有的功能,它都有.
3. 递归.Lisp是第一个支持递归函数的高级语言.
4. 变量的动态类型.在Lisp语言中,所有变量实际上都是指针,所指向的值有类型之分,而变量本身没有.复制变量就是相当于复制指针,而不是复制它们指向的数据.
5. 垃圾回收机制.
6. 程序由表达式组成.Lisp程序是一些表达树的集合,每个表达式都返回一个值.这与Fortran和大多数后来的语言都截然不同,他们的程序都由表达式和语句组成.区分表达式与语句在Fortran I中是自然的,因为它不支持语句嵌套.所以,如果你需要用数学式子计算一个值,那就只有表达式返回这个值,没有其他语法结构可用,否则就无法处理这个值.后来,新的编程语言支持块结构,这种限制当然就不存在了.但是为时已晚,表达式和语句的区分已经根深蒂固.它从Fortran扩散到它们两者的后继语言.
7. 符号类型.符号实际上是一种指针,指向存储在散列表中字符串.所以,比较两个符号是否相等,只要看它们的指针是否一样就可以了,不用逐个字符比较.
8. 代码使用符号和常量组成的树形表示法.
9. 无论什么时候,整个语言都是可用的.Lisp并不真正区分读取,编译期和运行期.你可以在读取期编译或运行代码,也可以在编译期读取和运行代码,还可以在运行期读取或编译代码.在读取期运行代码,使得用户可以重新调整Lisp的语法,在编译期运行代码,则是Lisp宏的工作基础,在运行期编译代码,使得Lisp可以在Emacs这样的程序中充当扩展语言(extension language),在运行期读取代码,使得程序之间可以用S表达式通信,近来XML格式的出现使得这个概念被重新”发明”出来了.

Lisp语言刚出现的时候,这些思想与其他编程语言大相径庭,后者的设计思想主要由50年代后期的硬件决定.随着时间流逝,流行的编程语言不断更新换代,语言设计思想逐渐向Lisp靠拢.思想(1)到思想(5)已经被广泛接受,思想(6)开始在主流编程语言中出现,思想(7)在Python语言中有所实现,不过似乎没有专用的语法.

思想(8)可能是最有意思的一点.它与思想(9)只是由于偶然的原因成为Lisp语言的一部分，因为它们不属于麦卡锡的原始构想，是由拉塞尔自行添加的．它们从此使得Lisp语言看上去很古怪，但也成为了这种语言最独一无二的特点．说Lisp语法古怪不是因为它的语法很古怪，而是因为它根本就没有语法，程序直接以解析树(parse tree)的形式表达出来．在其他语言中，这种形式只是经过解析在后台产生，但是Lisp直接采用它作为表达式形式．它由列表构成，而列表则是Lisp的基本数据结构．

用一种语言自己的数据结构来表达该语言是非常强大的功能．思想(8)和思想(9)，意味着你可以写出一种能够自己编程的程序．