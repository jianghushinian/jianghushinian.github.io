---
title: 用 Python 撸一个 Web 服务器-第3章：使用 MVC 构建程序
date: 2020-07-17 16:52:44
tags: 
- Python
- Web
categories: 
- Web
---

### Todo List 程序介绍

我们将要编写的 Todo List 程序包含四个页面，分别是注册页面、登录页面、首页、编辑页面。以下分别为四个页面的截图。

<!-- more -->

注册页面：

![注册](Register.png)

登录页面：

![登录](Login.png)

首页：

![首页](Index.png)

编辑页面：

![编辑](Edit.png)

程序页面非常简洁，甚至有些 Low。但这足够我们学习开发 Web 服务器程序原理，页面样式的问题并不是我们本次学习的重点，所以读者不必纠结于此。

Todo List 程序功能大概分为两个部分，一部分是 todo 管理，包含增删改查基础功能；另一部分是用户管理，包含注册和登录功能。


### 初识 MVC

介绍了 Todo List 程序的页面和功能，接下来我们就要思考如何设计程序。

以客户端通过浏览器向服务器发送一个获取应用首页的请求为例，来分析下服务器在收到这个请求后都需要做哪些事情：

1. 首先服务器需要对请求数据进行解析，发现客户端是要获取应用首页。

2. 然后找到代表首页的 HTML 文件，读取 HTML 文件中的内容。

3. 最后将 HTML 内容组装成符合 HTTP 规范的数据进行返回。

这是一个较为理想的情况，因为 HTML 页面内容是固定的，我们不需要对其进行其他处理，直接返回给浏览器即可。通常我们管这种页面叫静态页面。

但实际情况中，Todo List 程序首页内容并不是一成不变的，而是动态变化的。首页 HTML 文件中只定义基础结构，具体的 todo 数据需要动态填充进去。所以一个更加完整的服务器处理请求的过程应该像下面这样：

1. 首先服务器需要对请求数据进行解析，发现客户端是要获取应用首页。

2. 然后从数据库中读取 todo 数据。

3. 接着找到代表首页的 HTML 文件，读取 HTML 文件中的内容。

4. 再将 todo 数据动态添加到 HTML 内容中。

5. 最后将处理好的 HTML 内容组装成符合 HTTP 规范的数据进行返回。

现在已经知道了服务器处理请求的完整过程，我们就可以设计服务器程序了。试想一下，如果 Todo List 程序都像 Hello World 程序一样把代码都写在一个 Python 文件中也不是不可以。但这样的代码显然不具备良好的扩展性和可维护性。那么更好的设计模式是什么呢？

其实对于 Web 服务器程序的设计，业界早已达成了一个普遍的共识，那就是 `MVC` 模式：

M（Model）：模型，用来存储和处理 Web 应用数据。

V（View）：视图，格式化显示 Web 应用页面。

C（Controller）：控制器，负责基础逻辑，如从模型层读取数据并将数据填充到视图层，然后返回响应。

通过 `MVC` 的分层结构，能够让 Web 应用设计更加清晰，可以很容易的构建可扩展、易维护的代码。模型层，说直白些其实就是用来读写数据库的 Python 代码，新增 todo 的时候，可以通过模型层的代码将数据保存到数据库中，访问首页时需要展示所有已保存的 todo，这时可以通过模型层的代码从数据库中读取所有 todo。视图层，可以将其简单的理解为 HTML 模板文件的集合。控制器起到粘合的作用，它将从模型层读取过来的数据填充到视图层并返回给浏览器，或者将浏览器通过 HTML 页面提交过来的数据解析出来再通过模型层写入数据库中。

我画了一个示例图，来帮助你理解 `MVC` 模式。图中标注了浏览器发起一个请求到获得响应，中间经历的完整过程。 

![MVC](MVC.png)

还是以客户端请求 Todo List 程序首页为例，一个完整的请求过程如下：

1. 浏览器发起请求。

2. 请求到达服务器后首先进入控制器，然后控制器从模型获取 todo 数据。

3. 模型操作数据库，查询 todo 数据。

4. 数据库返回 todo 数据。

5. 模型将从数据库中查询的 todo 数据返回给控制器，控制器暂时将数据保存在内存中。

6. 控制器从视图中获取首页 HTML 模板。

7. 控制器将从模型查出来的 todo 数据填充到首页 HTML 模板中，并组装成符合 HTTP 规范的数据。

8. 服务器返回响应。

其实 `MVC` 是一个宏观上的分层，具体细节部分还需要根据我们设计程序的粒度来进行处理，比如有些逻辑既可以写在控制器层，也可以写在模型层，甚至我们还可以在 `MVC` 的基础上扩展更多的分层。这些都需要结合具体的业务逻辑来决定。


### 构建 Todo List 程序

学习了 `MVC` 模式，我们就可以根据 `MVC` 模式来试着构建 Todo List 程序了。

Todo List 程序分两部分：todo 管理、用户管理。在项目初期，我们肯定不会考虑的太过全面。所以可以先不考虑用户管理功能部分的实现，先只考虑如何实现 todo 管理功能。

Todo List 程序目录结构设计如下：

```
todo_list
├── server.py
└── todo
    ├── __init__.py
    ├── config.py
    ├── controllers.py
    ├── db
    │   └── todo.json
    ├── models.py
    ├── templates
    │   ├── edit.html
    │   └── index.html
    └── utils.py
```

这里以 `todo_list/` 作为程序的根目录，根目录下包含 `server.py` 文件和 `todo/` 目录。其中 `server.py` 主要功能就是作为一个 `Web Server` 来接收请求和返回响应，它是 Todo List 程序的入口和出口。而 `todo/` 目录则是 Todo List 程序处理业务逻辑的核心。

`todo/` 目录下的 `__init__.py` 将 `todo/` 文件夹标记为一个 Python 包。 `config.py` 用于存储一些项目的基础配置。`utils.py` 是一个工具集，里面可以定义一些供其他模块调用的类和方法。`db/` 目录作为 Todo List 程序存储数据的目录，`db/todo.json` 用来存储所有的 todo 内容。剩下还有两个 `.py` 文件和一个目录没有介绍，相信你已经猜到了 `models.py`、`templates/`、`controllers.py` 分别对应了 `MVC` 模式中的模型、视图、控制器。`models.py` 中编写操作 todo 数据的代码，`templates/` 目录用来存放 HTML 模板文件，`templates/index.html` 是首页，`templates/edit.html` 是编辑页面，`controllers.py` 编写负责程序控制的基础逻辑代码。

我们对项目的目录结构有了一个概览，这里我要强调一下 `db/` 目录的作用。我们在开发整个 Todo List 程序的过程中都不会使用实际的数据库程序，项目中所有需要存储的数据都保存在  `db/` 目录下的文件中。在开发 Web 程序时，需要用到数据库的目的就是为了存储数据，对于 Todo List 程序来说使用文件同样能满足需求，同时能够照顾到对数据库不了解的读者。

### Todo List 首页开发

Todo List 程序目录结构构建完成后就可以动手开发程序了，我们可以从一个请求经历的过程来着手。

一个请求发送到服务器，首先服务器需要有一个能够接收请求的入口，在程序根目录 `todo_list/` 下的 `server.py` 就是这个入口。`server.py` 文件代码如下：

```python
# todo_list/server.py

import socket
import threading

from todo.config import HOST, PORT, BUFFER_SIZE


def process_connection(client):
    """处理客户端请求"""
    # 接收请求报文数据
    request_bytes = b''
    while True:
        chunk = client.recv(BUFFER_SIZE)
        request_bytes += chunk
        if len(chunk) < BUFFER_SIZE:
            break

    # 请求报文
    request_message = request_bytes.decode('utf-8')
    print(f'request_message: {request_message}')

    # TODO: 解析请求
    # TODO: 返回响应
    
    # 关闭连接
    client.close()


def main():
    """入口函数"""
    with socket.socket() as s:
        s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
        s.bind((HOST, PORT))
        s.listen(5)
        print(f'running on http://{HOST}:{PORT}')

        while True:
            client, address = s.accept()
            print(f'client address: {address}')
            # 创建新的线程来处理客户端连接
            t = threading.Thread(target=process_connection, args=(client,))
            t.start()
```

`server.py` 程序几乎就是之前实现的多线程版 Hello World 服务器程序照搬过来的。为了程序代码更加清晰，这里将服务器的 IP 地址、端口、接收请求的缓冲区大小定义为变量写在了配置文件 `todo/config.py` 中，所以需要在 `server.py` 文件顶部从配置文件中导入 `HOST` 、`PORT` 、`BUFFER_SIZE`。在 `main` 函数中，实例化 `socket` 对象部分的代码也有所改变，这里采用了 `with` 语句来实例化 `socket` 对象，这样能够保证任何情况下退出程序时 `socket` 都能够被正确关闭。此处 `with` 语句的用法可以类比文件操作时的  `with` 语句。处理客户端连接请求的 `process_connection` 函数内部基本逻辑没有改变，其中有两个 `TODO` 注释表示解析请求和返回响应的功能暂未实现。

从 `server.py` 入口程序接收到客户端的请求以后，需要解析请求报文，并根据解析出来的请求报文来决定如何处理请求并返回响应。所以接下来我们需要编写解析请求的代码。

不过在这之前，我先给出 `todo/config.py` 配置文件的代码，毕竟之后还会用到：

```python
# todo_list/todo/config.py

import os

# todo/ 目录绝对路径
BASE_DIR = os.path.dirname(os.path.abspath(__file__))

# IP
HOST = '127.0.0.1'
# 端口
PORT = 8000

# 缓冲大小
BUFFER_SIZE = 1024
```

配置文件中除了包含前面介绍过的表示 IP 地址、端口、接收请求的缓冲区大小的几个变量，还有一个 `BASE_DIR` 变量用来表示 `todo/` 目录的绝对路径，方便在程序中获取项目路径。

现在来看下如何解析请求，我们可以定义一个 `Request` 类用来专门解析请求报文，代码写在 `todo/utils.py` 文件中：

```python
# todo_list/todo/utils.py

class Request(object):
    """请求类"""

    def __init__(self, request_message):
        method, path, headers = self.parse_data(request_message)
        self.method = method  # 请求方法 GET、POST
        self.path = path  # 请求路径 /index
        self.headers = headers  # 请求头 {'Host': '127.0.0.1:8000'}

    def parse_data(self, data):
        """解析请求报文数据"""
        # 用请求报文中的第一个 '\r\n\r\n' 做分割，将得到请求头和请求体
        # 请求体暂时用不到先不处理
        header, body = data.split('\r\n\r\n', 1)
        method, path, headers = self._parse_header(header)
        return method, path, headers

    def _parse_header(self, data):
        """解析请求头"""
        # 拆分请求行和请求首部
        request_line, request_header = data.split('\r\n', 1)

        # 请求行拆包 'GET /index HTTP/1.1' -> ['GET', '/index', 'HTTP/1.1']
        # 因为 HTTP 版本号没什么用，所以用一个下划线 _ 变量来接收
        method, path, _ = request_line.split()
        
        # 解析请求首部所有的键值对，组装成字典
        headers = {}
        for header in request_header.split('\r\n'):
            k, v = header.split(': ', 1)
            headers[k] = v
        
        return method, path, headers
```

 `Request` 类的初始化方法 `__init__` 接收请求报文字符串作为参数。在其内部调用 `parse_data` 方法将请求报文字符串解析成我们需要的结构化数据。

解析完请求报文，我们需要根据请求报文信息来判断如何返回响应。基础逻辑判断部分的代码可以写在 `todo/controllers.py` 中：

```python
# todo_list/todo/controllers.py

from todo.utils import render_template


def index():
    """首页视图函数"""
    return render_template('index.html')
```

定义在控制器层的函数也叫视图函数，因为它们通常返回视图层的 HTML 内容。`index` 视图函数用来处理请求首页的逻辑，它返回 `render_template` 函数的调用结果，`render_template` 函数的作用是将 HTML 内容读取成字符串并返回，其定义如下：

```python
# todo_list/todo/utils.py

import os

from todo.config import BASE_DIR


def render_template(template):
    """读取 HTML 内容"""
    # 读取 'todo_list/todo/templates' 目录下的 HTML 文件内容
    template_dir = os.path.join(BASE_DIR, 'templates')
    path = os.path.join(template_dir, template)
    with open(path, 'r', encoding='utf-8') as f:
        html = f.read()
    return html
```

在 `todo/controllers.py` 文件底部还定义了一个 `routes` 字典，字典的键为请求路径，值为一个元组，元组的第一个元素作为处理请求的函数，第二个元素是一个列表，里面定义处理请求的函数所允许的请求方法。`index` 视图函数能够同时匹配两个路径：`/`、`/index`，因为这两个路径通常都代表首页。

```python
# todo_list/todo/controllers.py

routes = {
    '/': (index, ['GET']),
    '/index': (index, ['GET']),
}
```

读取出 HTML 内容以后，我们就可以构造响应报文并返回给浏览器了。在 `utils.py` 文件下，编写一个 `Response` 类用来构造响应：

```python
# todo_list/todo/utils.py

class Response(object):
    """响应类"""

    # 根据状态码获取原因短语
    reason_phrase = {
        200: 'OK',
        405: 'METHOD NOT ALLOWED',
    }

    def __init__(self, body, headers=None, status=200):
        # 默认响应首部字段，指定响应内容的类型为 HTML
        _headers = {
            'Content-Type': 'text/html; charset=utf-8',
        }

        if headers is not None:
            _headers.update(headers)
        self.headers = _headers  # 响应头
        self.body = body  # 响应体
        self.status = status  # 状态码

    def __bytes__(self):
        """构造响应报文"""
        # 状态行 'HTTP/1.1 200 OK\r\n'
        header = f'HTTP/1.1 {self.status} {self.reason_phrase.get(self.status, "")}\r\n'
        # 响应首部
        header += ''.join(f'{k}: {v}\r\n' for k, v in self.headers.items())
        # 空行
        blank_line = '\r\n'
        # 响应体
        body = self.body

        response_message = header + blank_line + body
        return response_message.encode('utf-8')
```

`Response` 类的初始化方法 `__init__` 接收三个参数，分别为响应体、响应首部字段、状态码。其中响应体为 `str` 类型，首页的响应体实际上就是 `index.html` 文件内容。响应首部字段为 `dict` 类型，在构造响应报文时，所有的响应首部字段最终按照 HTTP 规范拼接到一起作为响应首部。状态码为数值类型，目前只考虑了状态码为 `200` 正常响应和 `405` 请求方法不被允许。

需要注意的是，`Response` 类定义了 `__bytes__` 魔法方法作为构造响应报文的方法。当使用 Python 内置的 `bytes` 方法转换 `Response` 实例对象时（`bytes(Response())`），会自动调用 `__bytes__` 魔法方法。

从解析请求到构造响应报文的代码现在已经基本编写完成。接下来我们将整个处理请求的流程串联起来，回到 `server.py` 文件，继续完善代码：

```python
# todo_list/server.py

import socket
import threading

from todo.config import HOST, PORT, BUFFER_SIZE
from todo.utils import Request, Response
from todo.controllers import routes


def process_connection(client):
    """处理客户端请求"""
    # 接收请求报文数据
    request_bytes = b''
    while True:
        chunk = client.recv(BUFFER_SIZE)
        request_bytes += chunk
        if len(chunk) < BUFFER_SIZE:
            break

    # 请求报文
    request_message = request_bytes.decode('utf-8')
    print(f'request_message: {request_message}')

    # 解析请求报文，构造请求对象
    request = Request(request_message)
    # 根据请求对象构造响应报文
    response_bytes = make_response(request)
    # 返回响应
    client.sendall(response_bytes)

    # 关闭连接
    client.close()


def make_response(request, headers=None):
    """构造响应报文"""
    # 默认状态码为 200
    status = 200
    # 获取匹配当前请求路径的处理函数和函数所接收的请求方法
    # request.path 等于 '/' 或 '/index' 时，routes.get(request.path) 将返回 (index, ['GET'])
    route, methods = routes.get(request.path)

    # 如果请求方法不被允许，返回 405 状态码
    if request.method not in methods:
        status = 405
        data = 'Method Not Allowed'
    else:
        # 请求首页时 route 实际上就是我们在 controllers.py 中定义的 index 视图函数
        data = route()

    # 获取响应报文
    response = Response(data, headers=headers, status=status)
    response_bytes = bytes(response)
    print(f'response_bytes: {response_bytes}')

    return response_bytes


def main():
    """入口函数"""
    with socket.socket() as s:
        s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
        s.bind((HOST, PORT))
        s.listen(5)
        print(f'running on http://{HOST}:{PORT}')

        while True:
            client, address = s.accept()
            print(f'client address: {address}')
            # 创建新的线程来处理客户端连接
            t = threading.Thread(target=process_connection, args=(client,))
            t.start()


if __name__ == '__main__':
    main()
```

首先完成之前未写完的 `process_connection` 函数。将原来标记 `TODO` 注释的地方替换成了如下代码：

```python
# 解析请求报文，构造请求对象
request = Request(request_message)
# 根据请求对象构造响应报文
response_bytes = make_response(request)
# 返回响应
client.sendall(response_bytes)
```

新增了一个 `make_response` 函数，方便用来根据请求对象构造响应报文。函数中我写了比较详细的注释，你可以根据注释内容读懂代码逻辑。

最后给出首页 `todo/templates/index.html` 的 HTML 代码：

```html
<!--todo_list/todo/templates/index.html-->

<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>Todo List</title>
    <style>
        * {
            margin: 0;
            padding: 0;
        }

        ul {
            list-style: none;
        }

        a {
            text-decoration: none;
            outline: none;
            color: #000000;
        }

        h1 {
            margin: 20px auto;
        }

        .container {
            display: flex;
            justify-content: center;
            align-items: center;
        }

        .container ul {
            width: 100%;
            max-width: 600px;
        }

        .container ul li {
            height: 40px;
            line-height: 40px;
            margin-bottom: 4px;
            padding: 0 6px;
            display: flex;
            justify-content: space-between;
            background-color: #d2d2d2;
        }
    </style>
</head>
<body>
<h1 class="container">Todo List</h1>
<div class="container">
    <ul>
        <li>
            <div>Hello World</div>
        </li>
    </ul>
</div>
</body>
</html>
```

HTML 代码比较简单，其中顶部写了一些基础的 CSS 样式，都很容易看懂，这里不再讲解。

接下来在终端中，进入 `todo_list/`目录下，使用 Python 运行 `server.py` 文件，看到如下打印结果说明程序已经正常启动：

![运行 Todo List 服务器](Run_Server.png)

打开浏览器，地址栏输入 `http://127.0.0.1:8000/` 或者 `http://127.0.0.1:8000/index`，你将看到 Todo List 程序首页：

![Todo List 首页](Todo_List_Index.png)

至此，Todo List 程序首页初步完成。不过我想很多读者看到这里会产生疑惑，说好的 `MVC` 呢，目前为止我们并没有编写一行模型层的代码，并且首页的 HTML 内容也不是动态填充的。没错，为了能够尽快让 Todo List 程序跑起来，我有意的避开了这两个问题，下一章我们再来解决这两个问题。

> 本章源码：[chapter3](https://github.com/jianghushinian/todo-list/tree/master/chapter3)
