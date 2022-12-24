---
title: Python 中的猴子补丁
date: 2020-05-16 21:24:05
tags: 
- Python
categories:
- Python
---

猴子补丁是从 `Monkey Patch` 翻译过来的，最初接触时觉得这个名字非常奇怪，于是就搜索了下其名字的由来，大致有两种说法：

<!-- more -->

1. 猴子补丁这个叫法起源于 `Zope` 框架，开发者在修复 `Zope` 的 `Bug` 的时候，经常在程序后面追加更新部分。最开始大家把它叫 `Guerrilla Patch(杂牌军补丁)`，位置不固定，哪里需要就出现在哪里。在英语里 `Guerrilla` 和 `Gorilla` 读音相似，有人就将其叫为 `Gorilla Patch`，`Gorilla` 译为大猩猩，叫着叫着，又有人把其叫成了 `Monkey Patch` 猴子补丁。
2. 另一种说法是在英文里 `monkeying about` 译为顽皮的，而猴子补丁的用法会将代码弄乱，也比较 “顽皮”，故此就有了 `Monkey Patch` 的叫法。

### 什么是猴子补丁

简单介绍了猴子补丁这个词的由来，那么什么是猴子补丁呢？所谓猴子补丁就是一种运行时的属性替换。

直白了说，就是在程序运行的过程中，为内存中的对象替换属性、方法，而不是通过修改源码的方式替换对象的属性、方法等。

### 猴子补丁示例

Python 在 3.5 版本之前，只对协程提供了基本支持，不过有个第三方库 `gevent` 通过大量使用猴子补丁的方式提供了更加完善的协程支持。

`gevent` 使用猴子补丁方法如下：

```python
import socket
from gevent import monkey

print(f'Python socket: {socket.socket}')

monkey.patch_socket()  # 使用 gevent 的猴子补丁
print(f'Gevent socket: {socket.socket}')
```

输出结果：

```
Python socket: <class 'socket.socket'>
Gevent socket: <class 'gevent._socket3.socket'>
```

`gevent` 使用猴子补丁非常简单，只需要一行代码 `monkey.patch_socket()` 就能实现。通过程序执行的结果可以发现，使用猴子补丁后，Python 内置的 `socket` 对象被替换为了 `gevent` 的 `socket` 对象。这就是猴子补丁的威力。

### 自己实现猴子补丁

看了 `gevent` 中使用猴子补丁的使用示例，对猴子补丁有了一个初步的概念。接下来我们就来自己实现一个猴子补丁。

在 Python 的 `multiprocessing` 内置库中，有一个 `cpu_count` 方法可以获取当前计算机的 CPU 核心数，我们可以通过用猴子补丁的方式来修改 `cpu_count` 这个方法。

```python
import multiprocessing

# 获取计算机 CPU 核心数
print(multiprocessing.cpu_count())


def cpu_count():
    return 10


multiprocessing.cpu_count = cpu_count  # 使用猴子补丁方式
print(multiprocessing.cpu_count())
```

输出结果：

```
4
10
```

我电脑的 CPU 核心数为 4，所以第一次输出结果即为 4。而通过 `multiprocessing.cpu_count = cpu_count` 这行代码将 `multiprocessing` 的 `cpu_count` 方法替换成自己实现的 `cpu_count` 方法后，得到的结果则为 10。 

这里就通过猴子补丁的方式实现了运行时的方法替换。这就是猴子补丁的原理，在未对源码以任何形式进行修改的情况下，只通过 `打补丁` 的方式替换掉 `multiprocessing `原有的 `cpu_count` 方法，以后每次调用 `multiprocessing.cpu_count()` 执行的都是自定义的 `cpu_count` 方法，也就是说返回结果将永远为 10。

### 总结

猴子补丁灵活且强大，但却是把双刃剑。维基百科上说，猴子补丁是一种很脏的编程技巧。它固然强大，但如果被滥用，将造成代码难以维护。