---
title: Python 异常处理机制
date: 2019-09-16 20:07:24
tags:
- Python
categories:
- Python
---

Python 解释器在执行代码时，如果发生错误，解释器会立即抛出异常并终止程序的运行。异常处理，实际上就是一种错误处理机制。

<!-- more -->

### 异常的常见场景

程序产生异常的原因有很多种，举例来说，在 Python 中做除法运算时，`0` 是不能做除数的，否则将会抛出 `ZeroDivisionError` 异常。

```python
>>> 1 / 0
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
ZeroDivisionError: division by zero
```

在 Python 控制台输入 `1 / 0` 并回车执行，解释器将立即抛出 `ZeroDivisionError` 异常，由异常提示信息`division by zero` 可知，产生异常的原因是 `0` 作为了除数。

还有很多其他容易产生异常的场景，如在发送网络请求时，有可能会遇到连接错误、超时等；访问系统资源时，有可能会遇到权限不足、资源不存在等。

在 Python 中，每一种异常都用一个类来定义，上面的 `ZeroDivisionError` 实际上就是一个异常类。Python 内置了很多种异常类，接下来就介绍下 Python 中的异常类。

### Python 异常类及继承关系

Python 提供了一个基础异常类 `BaseException`，所有其他的异常都继承于此类。以下列出了所有内置异常类及继承关系。

```
BaseException
 +-- SystemExit
 +-- KeyboardInterrupt
 +-- GeneratorExit
 +-- Exception
      +-- StopIteration
      +-- StopAsyncIteration
      +-- ArithmeticError
      |    +-- FloatingPointError
      |    +-- OverflowError
      |    +-- ZeroDivisionError
      +-- AssertionError
      +-- AttributeError
      +-- BufferError
      +-- EOFError
      +-- ImportError
      |    +-- ModuleNotFoundError
      +-- LookupError
      |    +-- IndexError
      |    +-- KeyError
      +-- MemoryError
      +-- NameError
      |    +-- UnboundLocalError
      +-- OSError
      |    +-- BlockingIOError
      |    +-- ChildProcessError
      |    +-- ConnectionError
      |    |    +-- BrokenPipeError
      |    |    +-- ConnectionAbortedError
      |    |    +-- ConnectionRefusedError
      |    |    +-- ConnectionResetError
      |    +-- FileExistsError
      |    +-- FileNotFoundError
      |    +-- InterruptedError
      |    +-- IsADirectoryError
      |    +-- NotADirectoryError
      |    +-- PermissionError
      |    +-- ProcessLookupError
      |    +-- TimeoutError
      +-- ReferenceError
      +-- RuntimeError
      |    +-- NotImplementedError
      |    +-- RecursionError
      +-- SyntaxError
      |    +-- IndentationError
      |         +-- TabError
      +-- SystemError
      +-- TypeError
      +-- ValueError
      |    +-- UnicodeError
      |         +-- UnicodeDecodeError
      |         +-- UnicodeEncodeError
      |         +-- UnicodeTranslateError
      +-- Warning
           +-- DeprecationWarning
           +-- PendingDeprecationWarning
           +-- RuntimeWarning
           +-- SyntaxWarning
           +-- UserWarning
           +-- FutureWarning
           +-- ImportWarning
           +-- UnicodeWarning
           +-- BytesWarning
           +-- ResourceWarning
```

乍一看，几十种异常类感觉有点头大，其实大可不必担心。这么多异常我们可以概括的将其分为两大类，系统类异常和非系统类异常。系统类异常包括 `SystemExit`、`KeyboardInterrupt`、`GeneratorExit`，非系统类异常就是 `Exception` 及其所有子类。可以看到，只有这 4 个异常类是直接继承自异常基类 `BaseException`，所有其他异常类都继承自 `Exception`。

实际上，我们真正在编码过程中，常见的异常类并不多，如读取文件时可能抛出的 `FileNotFoundError` 异常表示文件未找到，根据下标获取列表元素时可能抛出的 `IndexError` 异常表示索引越界，以及上面示例介绍的除 0 异常 `ZeroDivisionError` 等。你并不需要把这么多种异常都背下来，你只需在遇到异常时查看文档即可。而比较常见的异常类，遇到过几次你就会了然于心了，下次写代码的时候你就能够预判哪些地方可能会出现什么样异常。

### 处理异常

上面介绍了 Python 中异常的继承关系，以及一些常见的异常场景。接下来我们就要学习如何处理异常，因为程序一旦产生异常，如果不进行处理，程序就会自动终止运行。做好异常处理才能够使程序在遇到异常的时候不至于终止而继续运行下去。

一个比较完整的异常处理代码示例如下：

```python
try:
    ...  # 可能产生异常的代码
except Exception as e:
    ...  # 异常处理代码
else:
    ...  # 没有产生异常时执行的代码
finally:
    ...  # 不管是否产生异常，都会执行的代码
```

其中，`try` 语句块内编写的是一些有可能产生异常的代码块，一旦产生异常，就会被 `except` 关键字所捕获，`except` 关键字后面跟着的是要捕获的异常类型，`Exception` 代表捕获所有非系统类异常，`as e` 是给捕获的异常起一个别名 `e`，这个别名可以是任意符合 Python 变量命名规范的标识符，有了这个别名，就可以在 `except` 代码块中使用这个别名了，而如果 `try` 代码块中执行的代码没有产生异常，程序就会跳过`except` 代码块直接执行 `else` 代码块中的逻辑，无论是否产生异常 `finally` 代码块中的代码最终一定会被执行。

我们还是拿前面 `1 / 0` 的示例来通过实际代码观察下异常代码执行逻辑：

```python
try:
    print('try 代码块开始执行')
    1 / 0  # 会产生异常的代码
    print('try 代码块执行完毕')
except Exception as e:
    print('打印异常信息:', e)
else:
    print('没有产生异常')
finally:
    print('finally')
```

代码执行结果如下：

```
try 代码块开始执行
打印异常信息: division by zero
finally
```

可以看到，由于 `1 / 0` 会产生异常，所以 `print('try 代码块执行完毕')` 这句代码并没有执行，产生异常后直接执行了 `except` 代码块中的代码。由于 `else` 代码块只有在不产生异常的情况下才会被执行，所以 `else` 语句块中的代码也没有执行，而 `finally` 语句块中的代码最终被执行了。

我们再来对比下不产生异常时代码的执行逻辑：

```python
try:
    print('try 代码块开始执行')
    1 / 1  # 不会产生异常
    print('try 代码块执行完毕')
except Exception as e:
    print('打印异常信息:', e)
else:
    print('没有产生异常')
finally:
    print('finally')
```

代码执行结果如下：

```
try 代码块开始执行
try 代码块执行完毕
没有产生异常
finally
```

两段代码做一下对比，就很容易看出异常处理的执行逻辑了。

事实上，上面异常处理的代码示例中 `else` 和 `finally` 是可以被省略的。

```python
try:
    print('try 代码块开始执行')
    1 / 0
    print('try 代码块执行完毕')
except Exception as e:
    print('打印异常信息:', e)
```

这种写法也是我们编码过程中用的最多的写法。

上面说到 `except` 关键字能够捕获到异常，但其实是否执行 `except` 代码块是由跟在后面的 `Exception` 异常类决定的。`Exception` 能够捕获其所有子类及自身的异常，以上代码之所以能够捕获异常，是因为 `1 / 0` 产生的 `ZeroDivisionError` 类实际上是 `Exception` 类的子类。这里我们明确知道会产生 `ZeroDivisionError` 异常，所以更通常的写法是将 `except Exception as e:` 这句代码换成 `except ZeroDivisionError as e:`。

```python
try:
    print('try 代码块开始执行')
    1 / 0
    print('try 代码块执行完毕')
except ZeroDivisionError as e:
    print('打印异常信息:', e)
```

有时候一段代码有可能产生多种异常，如 `Web` 开发中的登录接口，往往需要前端通过 `AJAX` 将用户输入的 `username`（用户名）、`password`（密码）等信息组成 `JSON` 数据传递到后台，后台可能会先将 `JSON` 数据解析成字典，然后通过字典的 `key` 也就是 `username` 和 `password` 来获取用户输入的数据，最终完成登录操作。在这个过程中，解析 `JSON` 有可能产生异常，从解析后的字典中获取数据也有可能因为 `key` 不存在而产生异常。这个时候就需要 `except` 能够同时捕获多种异常，示例代码如下：

```python
import json

try:
    data = request.body
    json_data = json.loads(data)  # 解析 JSON 有可能抛出 json.JSONDecodeError 异常
    username = json_data['username']  # 如果 key 不存在则抛出 KeyError 异常
    password = json_data['password']
    ...
except (json.JSONDecodeError, KeyError) as e:
    print('打印异常信息:', e)
```

如果想用 `except` 同时能够捕获多种异常，可以将多个异常类组成一个元组跟在 `except` 后面。这样只要抛出的异常在元组内，`except` 代码块中的代码就会被执行。

其实捕获多种异常还有另一种写法：

```python
import json

try:
    data = '{}'
    json_data = json.loads(data)
    name = json_data['name']
except json.JSONDecodeError as e:
    print('JSONDecodeError 异常:', e)
except KeyError as e:
    print('KeyError 异常:', e)
except Exception as e:
    print('其他异常:', e)
```

这种写法的好处是，你可以在不同的捕获异常代码块中单独去写针对特定异常的处理逻辑。如果首先抛出了 `json.JSONDecodeError` 异常，那么之后的 `except` 就不会被执行了，这有点像 `if ... else ...` 的逻辑。可以看到代码的最后还加上了 `Exception` 异常类的捕获，这也是推荐的做法，因为假使程序没能如预期的抛出 `json.JSONDecodeError` 或者 `KeyError` 异常，最后还是会有一个兜底的 `Exception` 异常捕获，这样就不至于因为未知的异常而终止程序。

异常捕获还可以有更简洁的写法：

```python
try:
    print('try 代码块开始执行')
    1 / 0
    print('try 代码块执行完毕')
except:
    print('出现了异常')
```

这样写能够捕获任何异常，不过，通常并不建议这样做，因为在 `except` 代码块中你并不知道到底产生了哪种异常，而实际工作中，你往往需要在出现异常的地方记录日志，就需要知道是产生了哪种异常。

还有一种 `try ... finally` 的形式的写法也是可以的。通常用于不关心程序是否报错，但程序占用的资源无论如何最终都需要释放的时候。

```python
try:
    f = open('a.txt', 'w')
    # do something
finally:
    f.close()
```

其实我们使用的 `with open` 语句底层原理大概就是这样实现的。

### 自定义异常

如果你观察的比较仔细，就会发现，上面的例子中出现的 `JSONDecodeError` 异常并不在文章开头所列的 Python 内置异常类中。事实上，它是 `json` 模块中定义的一个异常类。根据这个启发，你应该能猜到，其实 Python 是允许我们定义自己的异常类的。

下面就是一个自定义异常类的示例：

```python
class MyException(Exception):
    pass

try:
    print(1)
    raise MyException('自定义异常')
    print(2)
except MyException as e:
    print(e)
```

代码执行结果如下：

```
1
自定义异常
```

自定义异常类也非常简单，我们只需要定一个类继承 `Exception` 类即可，你甚至不需要写其他任何代码。

实际上，自定义异常类可以继承任何 Python 中内置的异常类，但更推荐的继承类就是 `Exception`，虽然 `BaseException` 才是所有异常类的顶层类，并且能捕获更多的异常。但它会将 `KeyboardInterrupt` 异常捕获到，`KeyboardInterrupt` 异常通常由 `Ctrl + C` 来触发，而触发 `Ctrl + C` 时往往是我们需要主动退出程序，应该让 Python 解释器正常抛出这个异常才对。

### 异常处理时可能会遇到的问题

在异常处理过程中，你可能会遇到一些不太常见的问题。

假如在一个函数中处理异常，如果 `try` 代码块中有 `return` 语句，那么 `finally` 语句是否还会执行呢？

```python
count = 1

def foo():
    global count
    try:
        print('try')
        return count
    except Exception as e:
        print('except')
        print(e)
    finally:
        print('finally')
        count += 1

result = foo()
print(result)
print(count)
```

代码执行结果如下：

```
try
finally
1
2
```

由代码执行结果可以看出，`return` 语句实际上是在 `finally` 语句执行之前执行的，不过，`return` 的结果是要等到执行完 `finally` 代码块内的语句之后才返回的。所以在函数外部打印的 `result` 结果为 `1`，`count` 结果最终变为了 `2`。

再看一个关于变量名和 `Exception` 别名出现同名的情况下，会产生什么结果。

```python
e = 0

print(e)  # 0
try:
    1 / 0
except Exception as e:
    print('捕获异常:', e)  # 捕获异常: division by zero

print(e)  # NameError: name 'e' is not defined
```

代码执行结果如下：

```
0
捕获异常: division by zero
Traceback (most recent call last):
  File "demo.py", line 9, in <module>
    print(e)
NameError: name 'e' is not defined
```

我们在最开始定义了一个变量 `e = 0`，并且在 `except Exception as e:` 时同样将 `Exception` 的别名定义成了 `e`。最终的现象是，在 `except` 代码块中可以使用这个变量，而 `except` 代码块执行完成以后，再次打印 `e` 变量就抛出了 `NameError` 异常，异常信息提示 `e` 没有定义。

实际上，出现这个问题的原因 [Python 官方文档](https://docs.python.org/3/reference/compound_stmts.html#the-try-statement) 已有说明。

以上处理异常的代码其实相当于：

```python
e = 0

print(e)
try:
    1 / 0
except Exception as e:
    try:
        print('捕获异常:', e)
    finally:
        del e

print(e)
```

在每次执行了 `except` 代码块的最后，Python 解释器都会自动删除掉异常类的别名，所以在 `except` 代码块的外面再次打印 `e` 时，抛出了 `NameError` 异常。
