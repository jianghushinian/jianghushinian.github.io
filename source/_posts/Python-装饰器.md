---
title: Python 装饰器
date: 2019-08-16 21:33:31
tags:
- Python
- Pythonic
categories:
- Python
---
Python 中一切皆对象，函数也是对象。函数可以赋值给一个变量，函数可以当作参数传递个另一个函数，函数可以通过 return 语句返回函数。而装饰器就是一个能够接收函数并返回函数的函数。这话乍听起来有点绕，但装饰器本质上就是一个函数。
既然要学习装饰器，首先就要知道它用于什么场景，装饰器通过面向切面编程来增强代码的健壮性，比如：记录日志，处理缓存，权限校验等。接下来我们就一步一步的学习 Python 中装饰器的用法。

先来看一个简单的函数定义，函数只有一个功能，打印 Hello World：

```python
def hello():
    print('Hello World!')
```

现在新的需求来了，要在原有的函数执行前加入日志记录功能，于是就有了下面这段代码：

```python
def hello():
    print('run hello')
    print('Hello World!')
```

现在上面的问题解决了，只需要增加一行代码就能搞定。但问题是，实际工作场景下，我们可能需要修改的并不只是一个 `hello` 函数，有可能是 10 个、20 个函数同时需要增加日志功能。这个时候问题就来了，我们不太可能挨个函数依次复制这一行代码，况且那个时候有可能增加的不只是一行代码，可能上百行。并且这样就会造成出现大量的重复代码，当代码出现过多重复，你就要小心了，它很容易引起意想不到的 bug，并且难以排查及维护。
一个很容易想到的方法是定义一个专门打印日志的函数 `log`，然后在每个函数中都调用一下 `log` 函数：

```Python
def log():
    print('run hello')

def hello():
    log()
    print('Hello World!')
```

这样做还是需要修 `hello` 函数内部的代码，不是说不能这样做，但这样做显然违反了 `开闭原则` 思想 —— 对已实现的功能代码封闭，对扩展开放。虽然这句话通常用在面向对象编程思想中，但函数式编程同样适用。
我们可以考虑用高阶函数的方式来解决这个问题，还是定义一个 `log` 函数，但这次它接收一个函数作为参数，这个函数内部先执行打印日志的功能，在 `log` 函数最后调用传递进来的函数：

```python
def log(func):
    print('run hello')
    func()

def hello():
    print('Hello World!')

log(hello)
```

上面的代码就利用了函数可以当作参数传给另一个函数的特性，解决了需要修改原来函数内部代码的问题。这样做虽然功能上实现了，并且没有破坏原有函数内部的逻辑，但是却破坏了函数调用方的代码逻辑。也就是说，在原来代码中所有调用 `hello` 函数的语句不得不从 `hello()` 改为 `log(hello)`，这样做似乎更麻烦了些。

### 简单装饰器

那么，现在就是该引出 `装饰器` 这个概念的时候了，`装饰器` 非常擅长用 `Pythonic` 的方式解决这类问题。
来看一个最简单的装饰器的写法：

```python
def log(func):
    def wrapper():
        print('run hello')
        func()
    return wrapper

def hello():
    print('Hello World!')

hello = log(hello)
hello()
```

这段代码充分体现了前面所介绍的函数的特性，函数可以赋值给一个变量，函数可以当作参数传递个另一个函数，函数可以通过 return 语句返回函数。现在的 `log` 函数就是一个 `装饰器`。
首先定义一个 `log` 函数，它接收一个函数作为参数，并且它的内部又定义了一个 `wrapper` 函数，`wrapper` 函数在打印日志以后，调用了传递进来的 `func` 函数（也就是`hello`函数），在 `log` 函数的最后返回这个内部定义的函数。
在示例代码的最底部，我们将 `hello` 函数当作参数传递给 `log` 函数，并将其返回结果又赋值给变量 `hello`，此时的 `hello` 变量所指向的其实已经不是原来的 `hello` 函数，而是 `log` 装饰器返回的内部函数 `wrapper`。
现在调用方无需修改调用方式，仍然使用 `hello()` 的方式去调用 `hello` 函数，但它的功能已经增强了，会自动在执行 `print('Hello World!')` 逻辑之前加上打印日志的功能。
上面的代码我们从功能上实现了 `装饰器` 的效果。但实际上，Python 在语法层面上直接支持了装饰器模式。仅需要一个 `@` 符号就能让上面的代码更加可读，且易于维护。

```python
def log(func):
    def wrapper():
        print('run hello')
        func()
    return wrapper

@log
def hello():
    print('Hello World!')

hello()
```

`@` 符号是 Python 在语法层面上提供的语法糖，但它本质上完全等价于 `hello = log(hello)`。
以上就是一个最精简的符合 `Pythonic` 的 `装饰器`，无论你以后遇到多么复杂的装饰器，请记住，它最终的本质实际上就是一个函数，只不过利用了一些 Python 中的函数特性使其能够处理更复杂的业务场景。

### 被装饰的函数带有参数、返回值的装饰器

实际工作场景，我们写的函数往往都很复杂，想要写一个通用性更强的装饰器，还需要做一些细节部分的工作。不过你已经了解了装饰器的本质，剩下的例子理解起来并不会很费力，你只需要在特定的场景使用特定功能的装饰器就可以了。

```python
def log(func):
    def wrapper(*args, **kwargs):
        print('run hello')
        return func(*args, **kwargs)
    return wrapper

@log
def hello(name):
    print('Hello World!')
    return f'I am {name}.'

result = hello('xiaoming')
print(result)
```

`*args, **kwargs` 这两个不定长参数，就很好的解决了装饰器通用性的问题，使得装饰器在装饰任何函数的时候，参数都可以原样的传入到原函数内部。`wrapper` 函数最后调用 `func` 函数的前面加上了 `return` 语句，它的作用就是将原函数的 `return` 结果返回给调用方。

### 保持被装饰函数的元信息的装饰器

`log` 装饰器内部的 `wrapper` 函数打印日志的代码 `print('run hello')` 是固定的字符串，假如我们想要让其可以根据函数名自动更改打印结果，如 `print(f'run {函数名}.')` 这样的形式。
每个函数都有一个 `__name__` 属性，能够返回其函数名：

```python
def hello(name):
    print('Hello World!')

print(hello.__name__)  # hello
```

但问题是现在使用了 `log` 装饰器以后，原来的 `hello` 函数已经指向 `wrapper` 函数了，所以如果你测试就会发现，被装饰过的 `hello` 函数 `__name__` 属性已经变成了 `wrapper`，这显然不是我们想要的结果。
我们可以通过 `wrapper.__name__ = func.__name__` 一行语句解决这个问题，不过我们还有更好的办法。Python 内置了一个装饰器 `functools.wraps` 就能够帮我们解决这个问题。

```python
from functools import wraps

def log(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        print(f'run {func.__name__}')
        return func(*args, **kwargs)
    return wrapper

@log
def hello(name):
    print('Hello World!')
    return f'I am {name}.'

print(hello('xiaoming'))
print(hello.__name__)
```

### 装饰器自身带有参数

也许你想控制 `log` 装饰器的日志级别，那么给装饰器传参是一个很容易想到的办法，下面来看一下需要接收参数的装饰器的例子：

```python
from functools import wraps

def log(level):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            if level == 'warn':
                print(f'run {func.__name__}')
            elif level == 'info':
                pass
            return func(*args, **kwargs)
        return wrapper
    return decorator

@log('warn')
def hello(name):
    print('Hello World!')
    return f'I am {name}.'

result = hello('xiaoming')
print(result)
```

和之前的装饰器相比，带参数的装饰器又多了一层函数嵌套，实际上效果是这样的 `hello = log('warn')(hello)`，首先调用 `log('warn')` 返回的是内部 `decorator` 函数，接着就相当于 `hello = decorator(hello)`，实际上到这一步就和不带参数的装饰器一样了。

### 装饰器即支持带参数又支持不带参数

有时候可能会遇到更加变态的需求，需要装饰器传不传参数都能够使用，解决方式有多种，我这里给出一个比较简单容易理解的实现。

```python
from functools import wraps

def log(level):
    if callable(level):
        @wraps(level)
        def wrapper1(*args, **kwargs):
            print(f'run {level.__name__}')
            return level(*args, **kwargs)
        return wrapper1
    else:
        def decorator(func):
            @wraps(func)
            def wrapper2(*args, **kwargs):
                if level == 'warn':
                    print(f'run {func.__name__}')
                elif level == 'info':
                    pass
                return func(*args, **kwargs)
            return wrapper2
    return decorator

@log('warn')
def hello(name):
    print('Hello World!')
    return f'I am {name}.'

@log
def world():
    print('world')

print(hello('xiaoming'))
world()
```

`callable` 可以判断传递进来的参数是否可调用，不过需要注意，`callable` 只支持 `Python3.2` 及以上版本，你可以查看[官方文档](https://docs.python.org/3/library/functions.html?highlight=callable#callable)获取详细信息。

### 类装饰器

相比函数装饰器，类装饰更灵活，也更强大。在 Python 类中可以定义 `__call__` 方法，使其在无需实例化的情况下自身可以被调用，而此时就会执行 `__call__` 内部的代码。

```python
class Log(object):
    def __init__(self, func):
        self._func = func

    def __call__(self):
        print('before')
        self._func()
        print('after')

@Log
def hello():
    print('hello world!')

hello()
```

### 装饰器装饰顺序

一个函数其实可以同时被多个装饰器所装饰，那么多个装饰器的装饰顺序是怎样的呢？下面我们就来探索一下。

```python
def a(func):
    def wrapper():
        print('a before')
        func()
        print('a after')
    return wrapper

def b(func):
    def wrapper():
        print('b before')
        func()
        print('b after')
    return wrapper

def c(func):
    def wrapper():
        print('c before')
        func()
        print('c after')
    return wrapper

@a
@b
@c
def hello():
    print('Hello World!')

hello()
```

以上代码运行结果：

```
a before
b before
c before
Hello World!
c after
b after
a after
```

多装饰的语法等效于 `hello = a(b(c(hello)))`。根据打印结果不难发现这段代码的执行顺序。如果你了解过 Node.js 的 `Koa2` 框架的中间件机制，那么你一定不会陌生以上代码的执行顺序，实际上 Python 装饰器同样遵循 `洋葱模型`。多装饰器的代码执行顺序就像剥洋葱一样，先由外到内进入，然后再由内到外。
给大家留一个思考题：最终的 `hello.__name__` 指向哪一个装饰器内部的 `wrapper` 函数呢？

### 装饰器实战

理解了装饰器，我们就要用起来，文章开头有提到装饰器的用途，下面我们来看一个实际场景下使用装饰器的例子。
`Flask` 是 Python Web 生态中非常流行的一个微框架，你可以到 [GitHub](https://github.com/pallets/flask)上查看其源码。下面就是一个用 `Flask` 编写的最小 Web 应用。
在这里 `@app.route("/")` 装饰器的作用就是将根路由 `/`发送过来的请求绑定到处理函数 `hello` 上面来进行处理。这样当我们启动 `Flask` Web Server 以后，在浏览器地址访问 `http://127.0.0.1:5000/` 就能够获得返回结果 `Hello, World!`。

```python
from flask import Flask

app = Flask(__name__)

@app.route("/")
def hello():
    return "Hello, World!"
```

当然，更多的装饰器使用场景还是需要你自己亲自动手去探索发现。
