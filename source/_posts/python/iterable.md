---
layout: post
title: "python核心 - 迭代器"
date: 2015-12-01 20:02:42 +0800
toc: false
categories: python
tags: [python核心]
---

迭代(Iteration)指的是去获取元素的一种方式，一个接一个。当你显式或隐式的使用循环来遍历某个元素集的时候，那就是迭代。

在Python里面，可迭代对象(iterable)和迭代器(iterator)有着特殊的含义。

一个`iterable`指的就是一个对象：它要么定义了一个返回迭代器(iterator)的`__iter__`方法，
要么定义了一个可下标访问元素的`__getitem__`方法，下标从0开始，并且下标不合法的时候抛出`IndexError`异常。
也就是说就算一个对象没有定义`__iter__`方法，但是定义了`__getitem__`方法的话也可以，python里面的`str`对象就是个很好的例子。<!--more-->

而`iterator`定义就比较简单，它就是一个定义了`next`(Python 2)或者`__next__`(Python 3)方法的对象。
这个对象会记住迭代操作的状态，每次调用`__next__`方法时，会将状态值更新指向下一个迭代值。

当你使用一个`for`循环或者`map`，或着一个列表推导，那么会先通过iter()获取相应的迭代器，然后每次循环自动通过`next`方法调用这个迭代器(iterator)，从中获取每一个元素，从而完成迭代过程。

在一个`iterable`对象上执行`iter`会返回一个`iterator`对象, 比如`iter(obj)`

下面一个例子可以非常清晰的解释清楚：
```python
>>> s = 'cat'      # s is an ITERABLE
                   # s is a str object that is immutable
                   # s has no state
                   # s has a __getitem__() method

>>> t = iter(s)    # t is an ITERATOR
                   # t has state (it starts by pointing at the "c")
                   # t has a next() method and an __iter__() method

>>> next(t)        # the next() function returns the next value and advances the state
'c'
>>> next(t)        # the next() function returns the next value and advances
'a'
>>> next(t)        # the next() function returns the next value and advances
't'
>>> next(t)        # next() raises StopIteration to signal that iteration is complete
Traceback (most recent call last):
...
StopIteration

>>> iter(t) is t   # the iterator is self-iterable
```

可以使用isinstance()判断一个对象是否是Iterable对象：
```python
>>> from collections import Iterable
>>> isinstance([], Iterable)
True
>>> isinstance({}, Iterable)
True
>>> isinstance('abc', Iterable)
True
>>> isinstance((x for x in range(10)), Iterable)
True
>>> isinstance(100, Iterable)
False
```

可以使用isinstance()判断一个对象是否是Iterator对象：
```python
>>> from collections import Iterator
>>> isinstance((x for x in range(10)), Iterator)
True
>>> isinstance([], Iterator)
False
>>> isinstance({}, Iterator)
False
>>> isinstance('abc', Iterator)
False
```

生成器都是Iterator对象，但list、dict、str虽然是Iterable，却不是Iterator。

把list、dict、str等Iterable变成Iterator可以使用iter()函数：
```python
>>> isinstance(iter([]), Iterator)
True
>>> isinstance(iter('abc'), Iterator)
True
```
你可能会问，为什么list、dict、str等数据类型不是Iterator？

这是因为Python的Iterator对象表示的是一个数据流，Iterator对象可以被next()函数调用并不断返回下一个数据，直到没有数据时抛出StopIteration错误。
可以把这个数据流看做是一个有序序列，但我们却不能提前知道序列的长度，只能不断通过next()函数实现按需计算下一个数据，所以Iterator的计算是惰性的，只有在需要返回下一个数据时它才会计算。

Iterator甚至可以表示一个无限大的数据流，例如全体自然数。而使用list是永远不可能存储全体自然数的。

**小结**

凡是可作用于for循环的对象都是Iterable类型；

凡是可作用于next()函数的对象都是Iterator类型，它们表示一个惰性计算的序列；

集合数据类型如list、dict、str等是Iterable但不是Iterator，不过可以通过iter()函数获得一个Iterator对象。

Python的for循环本质上就是通过不断调用next()函数实现的，例如：
```python
for x in [1, 2, 3, 4, 5]:
    pass
```

实际上完全等价于：
```python
# 首先获得Iterator对象:
it = iter([1, 2, 3, 4, 5])
# 循环:
while True:
    try:
        # 获得下一个值:
        x = next(it)
    except StopIteration:
        # 遇到StopIteration就退出循环
        break
```

