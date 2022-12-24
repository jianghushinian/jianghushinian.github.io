---
title: Python 上下文管理器实现
date: 2019-09-18 22:33:35
tags:
- Python
- Pythonic
categories:
- Python
---

在 Web 开发中，我们经常需要对资源进行操作，如从数据库中读取数据、日志文件的写入等。这些都是常见的资源操作，而资源是有限的，所以当我们对资源操作完成以后就需要对其进行释放。如果资源没有得到释放，当资源占用数达到了操作系统的限定数，就会导致程序崩溃。

<!-- more -->

在文件操作过程中，为了保证文件操作完成后关闭文件资源，我们可能会写出这样的代码：

```python
try:
    f = open('demo.txt', 'w')
    f.write('hello')
finally:
    f.close()
```

这是一个典型的资源操作的例子，不过 Python 提供了 `with` 语句可以对以上代码进行简化：

```python
with open('demo.txt', 'w') as f:
    f.write('hello')
```

可以看到，用 `with` 重写过的代码中并没有显式的调用 `f.close()` 方法来关闭文件，但当 `with` 代码块执行完以后，文件还是会被正常关闭的。即使在调用 `f.write()` 时产生异常，文件同样会被关闭。

这就是 `with` 语句的强大之处，它不仅可以简化我们的代码，并且能够自动释放资源。其实这就是所谓的 `上下文管理器`。能够自动分配和释放资源正是上下文管理器最强大的地方。

`open` 函数能够通过 `with` 语句完成自动化的上下文管理，实际上，我们同样可以实现自定义的 `上下文管理器`。Python 为我们实现自定义 `上下文管理器` 提供了一个统一的接口——`上下文管理协议`。只要我们按照 Python 的规定实现了 `上下文管理协议`，就能够实现自定义的 `上下文管理器`。

### 基于类实现上下文管理器

我们知道，在 Python 中有很多双下划线开头、双下划线结尾的方法，我们通常称作 `魔法方法`。基于类的上下文管理器就是通过 `__enter__`、`__exit__` 两个 `魔法方法` 来实现的。

下面通过一个自定义的文件上下文管理器类来介绍上下文管理器的实现：

```python
class MyFileManager(object):
    def __init__(self, filename, mode):
        print('call __init__')
        self.filename = filename
        self.mode = mode
        self.file = None

    def __enter__(self):
        print('call __enter__')
        self.file = open(self.filename, self.mode)
        return self.file

    def __exit__(self, exc_type, exc_val, exc_tb):
        print('call __exit__')
        if self.file:
            self.file.close()

with MyFileManager('demo.txt', 'w') as f:
    print('write file')
    f.write('test')
```

代码执行结果如下：

```
call __init__
call __enter__
write file
call __exit__
```

可以看到，只需要十几行的代码，我们就实现了一个上下文管理器。而它的效果和 `with open` 并无两样。由代码执行结果我们可以分析出代码的执行顺序。首先在 `with` 语句后面实例化 `MyFileManager` 类，实例化过程中会调用类的 `__init__` 方法，接着自动调用了 `__enter__` 方法并拿到返回值，即文件对象，`as f` 就是将 `__enter__` 方法返回的文件对象赋值给 `f` 变量，然后执行 `with` 代码块中的代码对文件进行写入，当程序将要退出 `with` 代码块时，就会自动调用 `MyFileManager`  类的 `__exit__` 方法。

以上就是自定义 `上下文管理器` 的执行逻辑。要实现 `上下文管理器`，`__enter__`、`__exit__` 两个方法是必须实现的。其中 `__enter__` 方法用来获取资源，所以它将在 `with` 语句代码块开始执行之前被调用，`__exit__` 方法用来释放资源，它将在 `with` 语句代码块执行结束将要退出时被调用。

需要额外注意的是 `__exit__` 方法的参数，相比 `__enter__` 方法它多了三个参数，这三个参数的作用分别是：

```
exc_type: 异常类型
exc_val: 异常的值
exc_tb:异常栈
```

当执行 `with` 语句块中的代码，程序有可能会抛出异常，而 `__exit__` 方法的这三个参数，就会包含 `with` 语句块中产生的异常信息。所以 `__exit__` 方法还为我们提供了另外一个功能，即可以在此方法内部对异常进行处理，由开发者来决定异常的处理方式。

```python
class MyFileManager(object):
    def __init__(self, filename, mode):
        print('call __init__')
        self.filename = filename
        self.mode = mode
        self.file = None

    def __enter__(self):
        print('call __enter__')
        self.file = open(self.filename, self.mode)
        return self.file

    def __exit__(self, exc_type, exc_val, exc_tb):
        print('call __exit__')
        if self.file:
            self.file.close()
        if exc_type:
            print(f'exc_type: {exc_type}')
            print(f'exc_val: {exc_val}')
            print(f'exc_tb: {exc_tb}')
            return True

with MyFileManager('demo.txt', 'w') as f:
    raise Exception('write error')
```

代码执行结果如下：

```
call __init__
call __enter__
call __exit__
exc_type: <class 'Exception'>
exc_val: write error
exc_tb: <traceback object at 0x036D08A0>
```

可以看到，在 `with` 代码块中抛出的异常已经被 `__exit__` 方法捕获了。需要注意的是，在 `__exit__` 方法内部的最后还返回了 `True`，这代表异常已经处理，而不需要继续向上抛出，如果没有返回 `True`，那么异常将继续向上抛出，最终被 Python 解释器捕获到，继而终止程序。

### 基于生成器实现上下文管理器

Python 还提供了另一种基于生成器来实现 `上下文管理器` 的方法。示例代码如下：

```python
from contextlib import contextmanager

@contextmanager
def my_file_manager(filename, mode):
    print('open file')
    f = open(filename, mode)
    yield f
    print('close file')
    f.close()

with my_file_manager('demo.txt', 'w') as f:
    print('write file')
    f.write('demo')
```

代码执行结果如下：

```
open file
write file
close file
```

基于生成器的 `上下文管理器` 可以让我们不必通过类来实现，只需要一个打上 `@contextmanager` 装饰器的生成器函数即可。相比类实现的方式代码更加简洁，不过理解上来说没有类实现的方式来的清晰。但这种风格写起来更加 `Pythonic`。

其实 `上下文管理器` 的本质是能够在我们需要执行的一段代码之前和之后分别执行一段逻辑。这刚好能够利用生成器函数的特性，在 `yield` 关键字之前执行预处理工作，然后代码执行到 `yield` 处时让出函数的控制权，这时候再执行我们需要执行的一段代码逻辑，最后 `with` 语句帮我们把执行控制权交回到生成器函数内部，执行一些清理工作。

最后，一个小知识点，一个 `with` 语句实际上可以同时处理多个上下文管理器。

```python
with open('a.txt', 'w') as a, open('b.txt', 'w') as b:
    a.write('hello')
    b.write('world')
```
