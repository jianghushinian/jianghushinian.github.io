---
title: JavaScript 异步操作之回调函数
date: 2020-12-31 08:45:50
tags:
- JavaScript
- 异步编程
categories:
- JavaScript
---

本文试图尝试站在初学 `异步` 编程的角度来解释什么是 `回调函数`。

<!-- more -->

### 同步和异步

在介绍 `回调函数` 之前，先来看两个概念 `同步` 和 `异步`。

`同步` 行为通常指代码从上到下一行一行的顺序执行，后面的代码总是在前面的代码执行完成以后才会执行。

`同步` 操作的例子如下：

```javascript
let a, b;
function foo() {
    a = 1;
}
foo();
b = a + 1;
console.log(b); //2
```

由于代码顺序执行，先调用函数 `foo();`，后执行 `b = a + 1;`，所以在执行 `b = a + 1;` 时，`a` 的值一定已经被确定为 `1` 了，可以立即使用，变量 `b` 最终结果为 `2`。

`异步` 行为则指代码并非按照顺序执行，后面的代码不一定总是在前面的代码执行完成以后才会执行。

`异步` 操作的例子如下：

```javascript
let a, b;
function foo() {
    a = 1;
}
setTimeout(foo, 1000); //1 秒以后再调用 foo()
b = a + 1;
console.log(b); //NaN
```

这里没有显式的调用 `foo()`，而是将函数名 `foo` 传递给 `setTimeout`，JavaScript 运行时在 1 秒以后会自动调用 `foo()`。

首先代码依然顺序执行，当执行到 `setTimeout(foo, 1000);` 时，JavaScript 主线程发现这是一个将要异步执行的任务，就会将 `foo()` 放入 `任务队列` 然后继续执行下面的同步代码 `b = a + 1;`，当 `b = a + 1;` 执行完毕，所有同步代码都被执行完成，此时 JavaScript 主线程再去 `任务队列` 中取出需要执行的任务来执行，也就是 1 秒后执行 `foo()`。

因为 `b = a + 1;` 先于 `foo()` 执行，所以这段异步操作执行后，变量 `b` 最终结果为 `NaN`。

### 回调函数

介绍了 `同步` 和 `异步` 的概念，我们再来看下何为 `回调函数`。

在 `异步` 操作示例中，`setTimeout(foo, 1000);` 这句代码中的 `foo` 就可以被称作 `回调函数`。所谓 `回调函数`，就是被主线程放入到 `任务队列` 中的代码，这段代码通常以函数为单位，并且等到所有 `同步` 代码执行完成以后才会被执行。

所以上面的 `异步` 操作示例程序如果想得到与 `同步` 操作一样的结果，就得改成这样：

```javascript
let a, b;
function foo() {
    a = 1;
    b = a + 1;
    console.log(b); //2
}
setTimeout(foo, 1000); //1 秒以后再调用 foo()
```

因为计算 `b` 的值时依赖 `a` 的值，而 `a = 1;` 是在 `回调函数` 中执行的，也就是说所有 `同步` 代码执行完成以后才会执行 `回调函数` 中的 `异步` 代码，所以 `b = a + 1;` 也要移动到 `回调函数` 中。`回调函数` 中的代码也是顺序执行的，所以 `b = a + 1;` 语句要放在 `a = 1;` 语句之后。这样就能使变量 `b` 最终结果为 `2`。

### 回调函数应用

在 JavaScript 编程中，`AJAX` 操作可以说是非常常见的 `异步` 操作场景了。如果你不了解 `AJAX` 的概念，那么我强烈建议你学习一下。

以下是一个使用 `jQuery` 发送 `AJAX` 请求的示例：

```javascript
function handleSuccess(res) {
    console.log(res);
}

function handleError(xhr) {
    console.log(xhr.status);
}

$.ajax({
    type: 'GET',
    url: 'http://localhost:3000/',
    success: handleSuccess,
    error: handleError
});
```

其中 `success` 注册请求成功的 `回调函数` `handleSuccess`，`error` 注册请求失败的 `回调函数` `handleError`。

当 `AJAX` 请求成功并得到响应，JavaScript 运行时会自动调用 `handleSuccess`，请求失败则自动调用 `handleError`。

这里使用 `回调函数` 的目的，就是为了 `异步` 处理请求结果。因为发送 `AJAX` 请求是一个比较耗时的操作，如果 `同步` 处理，则会大大降低 JavaScript 运行效率。当 `AJAX` 请求发出以后，我们并不知道何时获得响应，所以采用 `异步` 编程的方式，将处理响应的代码注册为 `回调函数`。

可以发现，`回调函数` 和 `异步` 密不可分，`回调函数` 的概念只会出现在 `异步` 编程中，`同步` 编程是没有必要存在 `回调函数` 的。

### P.S.

文中对 JavaScript `异步` 执行机制描述并不准确，要想理解 JavaScript `异步` 执行机制则必须要学习 `Event Loop`。本文并不想引入太多复杂的概念，只想让初学 `异步` 编程的人能够快速理解什么是 `回调函数`。

如果你是一个 JavaScript 开发者，那么本文到此就结束了。如果你恰巧也是一个 Python 开发者，你还可以继续看下去。

对于写过 Python 爬虫的人来说，一定听说过 Scrapy 这个非常有名的爬虫框架，你可能在 Scrapy 项目的 spider 文件中见到过如下类似的代码：

```python
class DoubanSpider(scrapy.Spider):
    name = 'douban'
    allowed_domains = ['movie.douban.com']

    def start_requests(self):
        for i in range(0, 10):
            url = f'https://movie.douban.com/top250?start={i * 25}'
            yield scrapy.Request(url=url, callback=self.parse)
```

其中 `callback=self.parse` 就是将 `self.parse` 注册为 `回调函数`。因为 Scrapy 是一个 `异步` 爬虫框架，当 `scrapy.Request` 发出请求以后，程序并不会阻塞，而当请求成功并得到响应时，Scrapy 会自动调用我们注册的 `回调函数` `self.parse`，这与 `jQuery` 的 `AJAX` 如出一辙。

在用 Python 编程时，大部分时间都在写 `同步` 代码，所以很多初学者在初次接触 `异步` 编程时会感到不适应。当你在学习一门编程语言中陷入困惑，也许可以在其他编程语言中找到灵感。