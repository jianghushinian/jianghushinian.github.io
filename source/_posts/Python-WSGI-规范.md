---
title: Python WSGI 规范
date: 2021-04-07 08:39:06
tags:
- Python
categories:
- Python
---

作为 Python Web 开发者来说，在开发程序阶段一般是不会接触到 WSGI 这个名词的，但当程序开发完成，考虑上线部署的时候，WSGI 规范是一个绕不开的话题，本文将介绍何为 WSGI。

<!--more-->

WSGI 全拼 Web Server Gateway Interface，是为 Python 语言定义的 Web 服务器和 Web 应用程序（或框架）之间的一种通用编程接口。翻译成白话就是说 WSGI 是一个协议，就像 HTTP 协议定义了客户端和服务端数据传输的规范，WSGI 协议定义了 Web 服务器和 Web 应用程序之间协同工作的规范。

### Python Web 应用部署方案

`Flask` 或 `Django` 等 Web 框架都提供了内置的 Web Server，本地开发阶段可以使用 `flask run` 或 `python manage.py runserver` 来分别启动 `Flask` 或 `Django` 内置的 Server。

在生产环境部署应用时，通常不会使用框架内置的 Server，而是使用 `Gunicorn` 或 `uWSGI` 来部署，以获得更好的性能。部署过 Python Web 应用的同学应该对如下部署架构有所了解，左侧是浏览器，右侧是服务器。在服务器内部，首先通过 `Nginx` 来监听 `80/443` 端口，当接收到来自客户端的请求时，`Nginx` 会将请求转发到监听 `5000` 端口的 `Gunicorn/uWSGI` Server，接着请求会通过 WSGI 协议被传递到 `Flask/Django` 框架，在框架内部处理请求逻辑后，会将响应信息按照原路返回。

![Python Web Server Deploy](Python_Web_Server_Deploy.png)

你可能会问，`Nginx` 性能很高，为什么不将应用直接部署到 `Nginx` 上，而是中间通过 `Gunicorn/uWSGI` 做一层转发呢？因为 `Nginx` 没有遵循 WSGI 规范，并不能像 `Gunicorn/uWSGI` 这样很容易的与 `Flask/Django` 框架结合起来。

### WSGI 规范

根据 Python Web 应用部署架构，我们知道了 WSGI 所处的位置，接下来看下 WSGI 规范具体定义了哪些内容。

如同 HTTP 协议有一个客户端和一个服务端，WSGI 协议有一个 Application 端和一个 Server 端，其中 Application 就是指 `Flask`、`Django` 这些 Web 框架，而 Server 就是指 `Gunicorn`、`uWSGI` 等 Web 服务器。

WSGI 协议规定 Application 端需要实现成一个可调用对象（函数、类等），其接口如下：

```python
def simple_app(environ, start_response):
    status = '200 OK'
    response_headers = [('Content-type', 'text/plain')]
    start_response(status, response_headers)
    return ['Hello world!\n']
```

`simple_app` 就是一个最简单的 Application，它需要接收两个参数，`environ` 是一个 `dict`，其中保存了所有 HTTP 请求相关的信息，由 Server 端提供，`start_response` 是一个可调用对象，同样由 Server 端提供，`simple_app`内部需要调用一次 `start_response`，并将 `状态码` 和 `响应头` 当作参数传递给它，`simple_app` 最终会返回一个可迭代对象作为 HTTP Body  内容返回给客户端。

我们已经知道了 Application 端接口，接下来看下一个符合 WSGI 协议的 Server 端实现：

```python
import os


def wsgi_server(application):
    environ = dict(os.environ.items())

    def start_response(status, response_headers):
        print(f'status: {status}')
        print(f'response_headers: {response_headers}')

    result = application(environ, start_response)
    for data in result:
        print(f'response_body: {data}')
```

示例中 Server 端同样使用函数来实现，`wsgi_server` 接收一个 `application` 作为参数，在其内部构造了 `environ` 和 `start_response` 两个对象，这里使用环境变量信息来模拟 HTTP 请求信息构造  `environ` 字典，`start_response` 同样被定义为一个函数，供 `application` 在内部对其进行调用，`wsgi_server` 函数最后会调用 `application` 并对其进行打印。

现在有了 Application 端和 Server 端，我们可以来测试一下这个简单的 WSGI 程序示例。只需要将 `simple_app` 作为参数传递给 `wsgi_server` 并调用 `wsgi_server` 即可：

```python
wsgi_server(simple_app)
```

执行以上代码，将得到如下打印：

```
status: 200 OK
response_headers: [('Content-type', 'text/plain')]
response_body: Hello world!
```

以上，我们分别实现了符合 WSGI 规范的 Application 端和 Server 端，虽然程序看起来比较简陋，但不论多么复杂的 Python Web 框架和 Server 都同样遵循此规范。

### WSGI 实际应用

学习了 WSGI 规范，我们可以来验证下平时使用的 Python Web 框架是否真的遵循此规范，这里以 `Flask` 框架源码为例，可以在 `https://github.com/pallets/flask/blob/master/src/flask/app.py` 查看 `Flask` 的定义：

```python
class Flask(Scaffold):
    ...

    def __call__(self, environ, start_response):
        """The WSGI server calls the Flask application object as the
        WSGI application. This calls :meth:`wsgi_app`, which can be
        wrapped to apply middleware.
        """
        return self.wsgi_app(environ, start_response)
```

`Flask` 类内部通过实现 `__call__` 方法，使得 `Flask` 实例对象成为一个可调用对象，其接口实现同样符合 WSGI Application 规范。

### 参考

> https://www.python.org/dev/peps/pep-0333/