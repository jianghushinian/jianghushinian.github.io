---
title: Flask 路由处理 URL 路径末尾带斜线或不带斜线机制
date: 2020-01-27 09:58:50
tags:
- Python
- Flask
categories:
- Python
---

在浏览 Web 网站时，如果你细心观察会发现有些网站的 URL 网址末尾加不加斜线都能正常访问同一个页面。本篇文章就带大家来看看 Flask 中如何处理这种情况。

<!-- more -->

### 使用多个路由装饰器来绑定同一个视图函数

在 Flask 中通常来说每个请求由一个视图函数来处理，视图函数和请求 URL 的绑定是通过 `@app.route()` 装饰器实现的，使用 Flask 实现的一个最简单的 Hello World 程序如下：

```python
from flask import Flask

app = Flask(__name__)


@app.route('/hello')
def hello():
    return 'Hello World!'


if __name__ == '__main__':
    app.run()
```

浏览器地址栏访问 `http://127.0.0.1:5000/hello` 即可得到 `Hello World!` 响应。

![http://127.0.0.1:5000/hello](hello.png)

此时访问末尾带斜线的 URL 地址 `http://127.0.0.1:5000/hello/` 则无法获得正确响应。

![http://127.0.0.1:5000/hello/](hello_not_found.png)

要想使程序能够同时支持 URL 路径末尾带斜线或不带斜线，只需要在 `hello` 视图函数上再增加一个路由装饰器即可：

```python
from flask import Flask

app = Flask(__name__)


@app.route('/hello/')
@app.route('/hello')
def hello():
    return 'Hello World!'


if __name__ == '__main__':
    app.run()
```

启动程序，再次访问 `http://127.0.0.1:5000/hello/` 同样能得到 `Hello World!` 响应。

![http://127.0.0.1:5000/hello/](hello_line.png)

### 使用 Flask 自带的机制来支持 URL 路径末尾带斜线或不带斜线

我们使用了两个路由装饰器解决了 URL 路径末尾带斜线或不带斜线都能正确得到响应的问题。但这样做对搜索引擎来说，同一个页面会被索引两次，对 SEO 是不利的。

实际上，Flask 本身是支持 URL 路径末尾带斜线或不带斜线都能访问同一个页面的。我们只需要使用 `@app.route('/hello/')` 一个路由装饰器即可：

```python
from flask import Flask

app = Flask(__name__)


@app.route('/hello/')
def hello():
    return 'Hello World!'


if __name__ == '__main__':
    app.run()
```

启动程序，访问 `http://127.0.0.1:5000/hello/` 可得到 `Hello World!` 响应。

![http://127.0.0.1:5000/hello/](hello_line1.png)

访问末尾不带斜线的 URL 路径 `http://127.0.0.1:5000/hello` 同样能得到 `Hello World!` 响应。

![http://127.0.0.1:5000/hello](hello1.png)

当我们去掉了 `@app.route('/hello')` 装饰器，只使用一个 `@app.route('/hello/')` 装饰器就能够同时支持 URL 路径末尾带斜线或不带斜线。

在 Chrome 开发者工具 Network 选项卡可以发现，实际上 Flask 实现这种机制的原理就是采用重定向。如果访问末尾不带斜线的 URL 路径 `http://127.0.0.1:5000/hello`，Flask 会自动返回 308 状态码，重定向到带斜线的 URL 路径 `http://127.0.0.1:5000/hello/`，以此来实现只需要绑定一个路由 `@app.route('/hello/')` 即可同时支持 URL 路径末尾带斜线或不带斜线。

Flask 以一种优雅的实现，即同时支持 URL 路径末尾带斜线或不带斜线两种路径，又不影响搜索引擎的 SEO，可谓一举两得，值得借鉴。