---
title: 在 Python 中如何使用并发模型编程
date: 2023-05-14 18:00:18
tags:
- Python
- Pythonic
categories:
- Python
---

关于什么是并发模型，我在这里引用 Go 语言联合创造者 Rob Pike 的一段话：

> 并发是指一次处理多件事。并行是指一次做多件事。二者不同，但是有联系。一个关于结构，一个关于执行。并发用于制定方案，用来解决可能（但未必）并行的问题。

<!-- more -->

在不涉及并发概念的情况下，一个单进程单线程的程序执行情况可能是这样的：调用一个函数，发出调用的代码开始等待函数执行完成，直到函数返回结果，如果函数抛出异常，则可以把调用函数的代码放到 `try/except` 语句块中，来捕获和处理异常。

但是，当涉及到并发时，情况就没这么简单了。在启用多线程（或多进程）后，你无法在一个线程（或进程）中知道另一个线程（或进程）被调用的函数何时执行完成，也无法轻松得知函数调用结果或捕获异常。只能采用某种通知的方式，来进行线程（或进程）间通信，这可能是一个信号，也可能是一个消息队列等。

本文主要讲解如何让 Python 能够同时处理多个任务，即如何使用并发模型编程。

### 目标

我们将要实现一个旋转指针程序，启动一个程序，阻塞 3 秒钟（模拟耗时任务），在这期间，终端展示字符动画，让用户知道程序仍在执行，并没有停止，3 秒后程序打印耗时任务的计算结果并退出。

实现好的程序效果如下：

![效果展示](spinner-thread.gif)

这有点像下载进度条，因为打印旋转指针和耗时任务是“同时”进行的，这种场景只能通过并发模型来实现。

我们将分别使用多线程、多进程以及协程来实现这个程序，以此来演示 Python 的的并发模型用法。

### 多线程版

第一版旋转指针程序使用 Python 多线程来编写，首先，我们需要定义两个函数 `spin`、`slow` 分别用来实现旋转指针和模拟耗时任务（比如从网上下载一个文件）。

```python
import itertools
import time
from threading import Thread, Event


def spin(msg: str, done: Event) -> None:
    for char in itertools.cycle(r'\|/-'):
        status = f'\r{char} {msg}'
        print(status, end='', flush=True)
        if done.wait(0.1):
            break
    blanks = ' ' * len(status)
    print(f'\r{blanks}\r', end='')


def slow() -> int:
    time.sleep(3)
    return 42
```

`threading` 模块提供多线程支持，`Thread` 实例用来管理一个新的线程，`Event` 可以用来进行线程间通信。

`spin` 函数将作为一个任务在单独的线程中执行，它接收两个参数 `msg`、`done`，传递进来的 `msg` 将会跟随旋转指针一起被打印，`done` 参数类型为 `threading.Event`，用来实现多个线程间的通信，以此来同步任务状态。

`itertools.cycle(r'\|/-')` 是一个无限迭代器，一次产出一个字符，不停的迭代。比如用 `for` 遍历 `itertools.cycle('123')`，将得到无限迭代的数据 `123123123...`。这里 `\|/-` 字符不停迭代并被打印，就会产生旋转指针的效果。

打印的 `status` 字符串第一个字符为 `\r`，可以实现将光标移动到行首，这是一个使用文本在控制台实现动画的小技巧。

接下来的 `done.wait(0.1)` 是这个函数的关键代码，它是主线程与执行当前函数的子线程之间通信的桥梁。`done.wait` 方法签名为 `Event.wait(self, timeout=None)`，该方法等待 `timeout` 指定的时间后返回 `False`，我们在这里指定为 0.1 秒。如果在其他线程中使用 `Event.set()` 设置了这个事件，则当前线程该方法将立即返回 `True`，此时 `for` 循环就会被 `break` 掉。

`spin` 函数在退出前，还会打印几个空格来实现清空当前行打印内容的效果，并且最终还将光标移动到行首。

`slow` 函数使用 `time.sleep(3)` 暂停 3 秒，模拟耗时操作，这个函数将像我们往常编写的单线程代码一样在主线程中执行。

接下来我们要编写多线程代码来分别调用 `spin` 和 `slow` 两个函数，完成这个旋转指针程序。

```python
def supervisor() -> int:
    done = Event()
    spinner = Thread(target=spin, args=('thinking!', done))
    print(f'spinner object: {spinner}')
    spinner.start()
    result = slow()
    done.set()
    spinner.join()
    return result


def main() -> None:
    result = supervisor()
    print(f'Answer: {result}')


if __name__ == '__main__':
    main()
```

`supervisor` 函数中，首先实例化了一个 `Event` 对象，用于多线程通信。

接着，又实例化了一个 `Thread` 对象，用来管理子线程，`target` 参数接收一个函数 `spin`，这个函数将在一个独立的子线程中执行，`args` 参数接收一个元组，在子线程中调用 `spin` 函数时，元组的各个参数将被原样传递给 `spin` 函数。

`Thread` 对象必须要显式的调用 `start` 方法才能启动，所以代码执行到 `spinner.start()` 时，子线程才会真正开始执行。子线程只会执行 `spin` 函数，至于下方的代码与子线程无关，都是主线程要执行的代码。

子线程的执行对主线程执行不会产生影响，主线程代码会继续往下运行，主线程调用 `slow()` 时会被耗时任务所阻塞。此时，子线程内部代码执行不受影响，所以子线程会不停的打印旋转指针。

等待 3 秒结束后，主线程中 `slow()` 函数返回结果，主线程调用 `done.set()` 将 `Event` 对象设置为 `True`。此时，子线程 `spin` 函数内部 `done.wait(0.1)` 会立即返回 `True`，随即 `for`循环终止，`spin` 执行完成后子线程也就退出了。

主线程不受子线程退出影响，会接着往下执行，调用 `spinner.join()` 是为了等待子线程结束，主线程会阻塞在这里，保证子线程结束后才会往下执行。显然，子线程在执行完 `spin` 函数就结束了，所以主线程代码会继续往下执行。

`supervisor` 函数最终返回 `slow` 方法的返回值 `result`。

入口函数 `main` 打印 `result` 值后，主线程也退出了，程序终止。

以上，就是多线程版旋转指针程序的全部逻辑了。

我们来测试下这个程序执行效果：

![spinner-thread](spinner-thread.gif)

多线程对象 `spinner` 输出结果为 `<Thread(Thread-1, initial)>`，其中 `Thread-1` 是线程名称，`initial` 是线程状态，表示当前线程刚初始化完成，尚未启动。

### 多进程版

Python 提供了 `multiprocessing` 来支持多进程，这个模块的 API 基本模仿了多线程的 `threading` 模块，所以有了前文的基础，多进程代码也非常容易看懂。

同多线程一样，`multiprocessing` 包也为多进程通信提供了 `Event` 对象。不同的是，`threading.Event` 是一个类，`multiprocessing.Event` 是一个函数，它返回一个 `synchronize.Event` 类实例。所以 `spin` 函数签名需要进行如下修改：

```python
from multiprocessing import Process, Event
from multiprocessing import synchronize


def spin(msg: str, done: synchronize.Event) -> None:
    ...
```

`spin` 函数内部代码无需调整，只需要修改参数 `done` 的类型注解即可。所以不难发现 `synchronize.Event` 同样支持 `Event.wait(self, timeout=None)` 方法。

多进程版本的 `supervisor` 函数也要稍作修改：

```python
def supervisor() -> int:
    done = Event()
    spinner = Process(target=spin, args=('thinking!', done))
    print(f'spinner object: {spinner}')
    spinner.start()
    result = slow()
    done.set()
    spinner.join()
    return result
```

虽然 `multiprocessing.Event` 和 `threading.Event` 类型不同，但二者用法和作用则完全相同。

这里使用 `Process` 实例化一个进程对象，`Process` 用法和 `Thread` 用法同样如出一辙。

只需要对代码做少量的改动，我们就将程序从多线程迁移到了多进程。这一点，Python 做的非常友好，掌握了多线程编程，基本上就掌握了多进程编程，我们只需要在适当的时候，使用不同的模块即可。

下面是多进程版本旋转指针程序测试效果：

![spinner-process](spinner-process.gif)

多进程对象 `spinner` 输出结果为 `<Process name='Process-1' parent=94367 initial>`，进程名称为 `Process-1`，`parent` 代表父进程 ID 为 `94367`（即主进程 ID），`initial` 是进程状态，表示当前进程刚初始化完成，尚未启动。

### 协程版

最后我们将使用协程实现旋转指针程序，这一版本代码改动会比较大。

Python 在 3.5 版本提供了 `async`、`await` 关键字（可以参考 [PEP 492](https://peps.python.org/pep-0492/)），开始原生支持了协程，我们不再需要编写难懂的 `yeild from` 来使用生成器实现协程功能了。

Python 协程通常在单线程的事件循环中运行。协程是一个可以挂起自身并在以后恢复的“函数”，`async` 用来定义协程，一个协程必须显式的使用 `await` 关键字主动让出控制权，另一个协程才有机会在主事件循环的调度下并发的执行。

协程版本旋转指针程序需要对 `spin` 和 `slow` 两个函数做如下修改：

```python
import asyncio
import itertools


async def spin(msg: str) -> None:
    for char in itertools.cycle(r'\|/-'):
        status = f'\r{char} {msg}'
        print(status, end='', flush=True)
        try:
            await asyncio.sleep(0.1)
        except asyncio.CancelledError:
            break
    blanks = ' ' * len(status)
    print(f'\r{blanks}\r', end='')


async def slow() -> int:
    await asyncio.sleep(3)
    return 42
```

首先我们使用 `async def` 将 `spin` 定义为一个协程，让其不再是一个常规的函数。

`spin` 协程取消了第二个参数，因为 Python 没有为协程提供 `Event` 对象来进行通信，我们需要采用其他招式。

在原来使用 `Event` 通信的地方替换成了由 `try/except` 包裹的 `await asyncio.sleep(0.1)` 语句块代码。这段代码块有如下三个作用：

1. `await asyncio.sleep(0.1)` 的作用类似 `time.sleep`，可以让程序暂停 0.1 秒。不同的是，使用 `await asyncio.sleep` 暂停时不阻塞其他协程。

2. 因为这里加入了 `await` 关键字，代码执行到这里时，当前协程会主动让出控制权，不再继续往下执行，由事件循环来调度其他协程执行。

3. 如果在控制当前协程的 `Task` 实例中调用 `cancel` 方法（有关 `Task` 的内容稍后会进行讲解），`await asyncio.sleep(0.1)` 会抛出 `CancelledError` 异常，这里使用 `try/except` 捕获异常后退出循环。这样，我们就在多个协程间利用异常机制完成了通信，而不必借助于额外的 `Event` 对象。

`slow` 函数也被改造为一个协程，其内部原来编写的阻塞代码 `time.sleep(3)` 被替换为了 `await asyncio.sleep(3)`。

可以发现，其实协程与普通的函数在定义上差别不大，只不过多了两个关键字 `async` 和 `await`。但二者在执行方式上大有不同，普通函数在使用 `()` 运算符调用时（即 `spin()`）会立刻执行，而协程在使用 `spin()` 时只会创建一个协程对象，不会执行。

要执行上面两个协程对象，我们还要对 `supervisor` 和 `main` 函数进行改造：

```python
async def supervisor() -> int:
    spinner = asyncio.create_task(spin('thinking!'))
    print(f'spinner object: {spinner}')
    result = await slow()
    spinner.cancel()
    return result


def main() -> None:
    result = asyncio.run(supervisor())
    print(f'Answer: {result}')


if __name__ == '__main__':
    main()
```

`supervisor` 函数同样被修改为协程，`spin('thinking!')` 并不会像函数一样立即执行，只会创建一个协程对象，将它传递给 `asyncio.create_task`，我们可以得到一个 `asyncio.Task` 对象，这个 `Task` 对象包装了协程对象并调度其执行，它还提供控制和查询协程对象运行状态的方法。

使用 `await` 关键字来调用 `slow` 协程，这将阻塞 `supervisor` 程序（但会让出控制权，使其他协程得以执行），直到 `slow` 返回，返回结果赋值给 `result` 变量。

接着调用了 `spinner.cancel()`，`Task.cancel` 方法调用后，将立即在 `Task` 所包装的协程对象即 `spin` 协程中抛出 `CancelledError` 异常，`spin` 中需要使用 `try/except` 捕获 `await asyncio.sleep(0.1)` 抛出的异常，这样，就实现了不同协程之间通过异常进行通信。

`main` 是唯一的普通函数，没有被改造为协程。`main` 函数中的 `asyncio.run` 是整个协程的启动入口，`asyncio.run` 函数启动事件循环，驱动 `supervisor()` 协程运行，最终也将启动其他协程。

在以上示例代码中，我们可以总结出运行协程的 3 种方式：

- `asyncio.run(coroutine())`：在一个常规函数中调用，是协程启动入口，将开启协程的事件循环，调用后保持阻塞，直至拿到 `coroutine()` 的返回结果。

- `asyncio.create_task(coroutine())`：在协程中调用，接收另一个协程对象并调度其最终执行，返回的 `Task` 对象是对协程对象的包装，并且提供控制和查询协程对象运行状态的方法。

- `await coroutine()`：在协程中调用，`await` 关键字主动让出执行控制权，终止当前协程执行，直至拿到 `coroutine()` 的返回结果。同时这也是一个表达式，返回结果即为 `coroutine()` 返回值。

下面是协程版本旋转指针程序测试效果：

![spinner-async](spinner-async.gif)

在协程版本中，`spinner` 是一个 `Task` 对象，其字符串表示形式为 `<Task pending name='Task-2' coro=<spin() running at /Users/jianghushinian/spin/spinner_async.py:8>>`。

根据以上示例代码，我们可以总结出 Python 协程的最大特点：一处异步，处处异步。在协程中任何耗时操作都会减慢事件循环，由于事件循环是单线程管理的，所以这会影响其他所有协程。在编写协程代码，要格外小心，不要写出同步阻塞的代码。好在如今 Python 已经从语法层面原生支持协程，比使用生成器实现协程的年代要好多了。

给你留个小作业：尝试将 `slow` 协程中 `await asyncio.sleep(3)` 替换成普通的 `time.sleep(3)` 观察下效果并思考为什么。

### 总结

本文我们分别使用了多线程、多进程以及协程这三种不同的并发模型实现了旋转指针程序。三者比较起来，多线程、多进程在语法上差别不大，协程则大为不同，理解起来也更加困难。

由 `supervisor` 中打印的 `spinner` 对象结果可以看出，线程对象使用 `Thread` 来表示，进程对象使用 `Process` 来表示，协程对象则使用 `Task` 来表示。协程的定位是用户态线程，相比传统意义上的线程更加轻量，所以叫作 `Task` 也合理，代表同一个线程下的不同任务。

线程和进程无法在外部终止，即主线程和主进程无法终止由子线程或子进程来执行的 `spin` 函数，只能通过 `Event` 来进行通信，然后由 `spin` 函数内部自己终止。协程则可以通过任务实例方法 `Task.cancel()` 进行通信，`spin` 协程中捕获 `CancelledError` 异常后终止自身代码。

记住，使用协程的代码只有一个执行流，就如同单线程代码，同样只有一个执行流，只不过单线程代码执行流永远是从上到下，而协程的执行流则由事件循环来控制。

多线程和多进程模型是抢占式的，由操作系统进行调度执行。用户通常需要控制的是不要让多个线程（进程）同时操作同一个数据，常使用互斥锁来解决这一问题。而协程只有一个控制循环，协程的控制权在我们自己手里，我们决定什么时候切换其他任务来执行。所以在编写协程代码时，要时刻注意不要写出同步阻塞代码。

本文示例全部出自《流畅的Python（第2版）》一书，这本书的每一小节都值得拿出来单独探讨，我之所以挑选这一主题进行分享，是因为刚好这个主题内容在《流畅的Python（第2版）》中有所升级，如果你阅读过第一版，本文能够很好的展示第二版更新了什么。并发模型是一个宏大的话题，本文只作为入门指引，想要更深入学习，推荐阅读《流畅的Python（第2版）》。

最后，我要强烈推荐一下刘志军博主的知识星球「[ChatGPT研究社](https://t.zsxq.com/10CSFP9Wk)」，军哥还是公众号「Python之禅」的出品人，目前星球已经有近 500 人加入，专栏文章更新近 30 篇，星球不限于分享 ChatGPT 内容，还有不定期抽奖。同时，我也会不定期在星球分享 AIGC 相关内容。

![ChatGPT研究社](chatgpt.jpeg)

本文完整代码示例我放在了 [GitHub](https://github.com/jianghushinian/blog-python-example/tree/main/spin) 上，欢迎点击查看。

希望此文能对你有所帮助。

**联系我**

- 微信：jianghushinian
- 邮箱：[jianghushinian007@outlook.com](mailto:jianghushinian007@outlook.com)
- 博客地址：https://jianghushinian.cn/

**参考**

- 《流畅的Python（第2版）》
- 本文示例代码：https://github.com/jianghushinian/blog-python-example/tree/main/spin
