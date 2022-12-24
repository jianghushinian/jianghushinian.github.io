---
title: 用 Python 撸一个 Web 服务器-第5章：处理静态资源
date: 2020-07-17 21:21:43
tags: 
- Python
- Web
categories: 
- Web
---

### 处理静态文件

由于我们实现的模板引擎不支持直接将 CSS 嵌入在 HTML 中的写法，所以要将 CSS 独立出来。在 `todo/` 目录下新建 `static/` 目录，专门用来存储 CSS、JavaScript、图片等静态文件，在 `static/` 目录下新建 `css/` 目录用来存储 CSS 样式。我们把之前在 `todo/templates/index.html` HTML 页面中写的 CSS 移动到 `todo/static/css/` 目录下的 `style.css` 文件中：

<!-- more -->

```css
/* todo_list/todo/static/css/style.css */

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
```

接下来在首页 HTML 中引入 `style.css`：

```html
<!-- todo_list/todo/templates/index.html -->

<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>Todo List</title>
    <!-- 引入外部 CSS 样式 -->
    <link rel="stylesheet" href="/static/css/style.css">
</head>
<body>
<h1 class="container">Todo List</h1>
<div class="container">
    <ul>
        {% for todo in todo_list %}
            <li>
                <div>{{ todo.content }}</div>
            </li>
        {% endfor %}
    </ul>
</div>
</body>
</html>
```

浏览器在访问 Todo List 程序首页得到 `index.html` 源码后，发现 HTML 顶部的 `link` 标签，浏览器会自动再次发送一个请求到 `link` 标签 `href` 属性所指定的网址来获取 CSS 样式。所以我们还要编写一个视图函数来处理 `/static/css/style.css` 路径的请求，返回 `style.css` 内容。

```python
# todo_list/todo/controllers.py

import os

from todo.config import BASE_DIR
from todo.utils import Response, render_template


# 此处省略了 index 视图函数
...

def static(request):
    """读取静态资源视图函数"""
    # 能够处理的静态资源类型
    content_type = {
        'css': 'text/css',
        '.js': 'text/javascript',
        'png': 'image/png',  # 文件需要返回二进制
        'jpg': 'image/jpeg',
    }

    # 请求路径格式：/static/css/style.css
    # 根据请求路径中的文件名后缀判断静态文件类型，指定响应首部字段 Content-Type
    headers = {
        'Content-Type': content_type.get(request.path[-3:], 'text/plain'),
    }

    # 获取静态资源绝对路径
    path = request.path.lstrip('/')
    file_path = os.path.join(BASE_DIR, path)

    # 读取静态资源内容，构造响应对象并返回
    with open(file_path, 'r') as f:
        body = f.read()
    return Response(body, headers=headers)


routes = {
    ...
    '/static': (static, ['GET']),
}
```

编写 `static` 视图函数来专门处理静态文件，它不只能够处理 CSS 类型文件，还支持 JavaScript 文件和图片。实际上静态资源的访问路径被我故意设计为以 `/static` 开头加上静态文件类型 `css` 再加上文件名 `style.css` 的形式，这样只需要编写一个 `static` 视图函数就能够处理多种类型的静态文件，而不需给 CSS、JavaScript 等静态资源都单独编写一个视图函数。服务器在收到客户端发过来的请求时，判断请求路径以 `/static` 开头即可断定客户端需要请求静态资源，然后将请求交由 `static` 视图函数来处理。

假如程序需要用到 JavaScript，只需要在 `todo/static/` 目录下新建 `js/index.js` 文件，当收到请求路径为 `/static/js/index.js` 的客户端请求时，`static` 视图函数就能够正确处理响应。

`static` 视图函数根据不同的资源类型，会指定对应的响应首部字段。浏览器根据响应首部字段 `Content-Type` 的值来判断响应报文类型，`Content-Type: text/html` 指明响应报文类型为  HTML，`Content-Type: text/css` 指明响应报文类型为 CSS。

由于需要指定响应首部字段，所以 `static` 视图函数返回的并不是响应体，而是直接构造了一个 `Response` 对象。因此之前编写的 `make_response` 函数逻辑相应的也需要做些修改：

```python
# todo_list/server.py

def make_response(request, headers=None):
    """构造响应报文"""
    # 默认状态码为 200
    status = 200
    # 处理静态资源请求
    if request.path.startswith('/static'):
        route, methods = routes.get('/static')
    else:
        route, methods = routes.get(request.path)

    # 如果请求方法不被允许返回 405 状态码
    if request.method not in methods:
        status = 405
        data = 'Method Not Allowed'
    else:
        # 请求首页时 route 实际上就是我们在 controllers.py 中定义的 index 视图函数
        data = route(request)

    # 如果返回结果为 Response 对象，直接获取响应报文
    if isinstance(data, Response):
        response_bytes = bytes(data)
    else:
        # 返回结果为字符串，需要先构造 Response 对象，然后再获取响应报文
        response = Response(data, headers=headers, status=status)
        response_bytes = bytes(response)

    print(f'response_bytes: {response_bytes}')
    return response_bytes
```

修改后的 `make_response` 函数会判断请求路径如果以是 `/static` 开头则将请求交给 `static` 视图函数来处理。同时还修改了视图函数的调用接口 `data = route(request)`，每次调用视图函数时需要将 `request` 对象传递进去，所以记得修改 `index` 视图函数让其能够接收 `request` 参数。 `make_response` 函数最后还对视图函数返回结果 `data` 做了类型检查，如果返回的是 `Response` 对象，则可以直接获取响应报文，否则需要先构造 `Response` 对象，然后再获取响应报文。

终端中进入项目根目录 `todo_list/` 下，使用 Python 运行 `server.py` 文件，就能看到带有 CSS 样式的 Todo List 程序首页了：

![Todo List 首页](Todo_List_Index_Use_CSS.png)

现在 Todo List 程序已经有了处理静态资源的能力，接下来再给 Todo List 程序添加一个网页图标。

网页图标格式通常为 `.ico` 文件，将其放到 `todo/static/` 目录下。然后在控制器中编写一个 `favicon` 视图函数用来处理 `ICO` 图标：

```python
# todo_list/todo/controllers.py

...

def favicon(request):
    """读取网页 ICO 图标"""
    headers = {
        'Content-Type': 'image/x-icon',
    }
    path = f'static/{request.path.lstrip("/")}'
    file_path = os.path.join(BASE_DIR, path)
    # ICO 图标文件需要读取为二进制
    with open(file_path, 'rb') as f:
        body = f.read()
    return Response(body, headers=headers)


routes = {
    ...
    '/favicon.ico': (favicon, ['GET']),
}
```

当请求路径为 `/favicon.ico` 时匹配到 `favicon` 视图函数进行处理。因为 `ICO` 图标文件需要以二进制格式来读取，并且不能够直接转换为字符串类型，所以在构造 `Response` 对象时传入的响应体不再是 `str` 类型，而是 `bytes` 类型。因此还要修改下 `Response` 类，使其能够接收 `bytes` 类型：

```python
# todo_list/todo/utils.py

class Response(object):
    """响应类"""

    ...

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

        # body 支持 str 或 bytes 类型
        if isinstance(body, str):
            body = body.encode('utf-8')
        response_message = (header + blank_line).encode('utf-8') + body
        return response_message
```

只需要修改 `Response` 类的 `__bytes__` 方法即可，根据 `body` 类型来决定如何构造响应报文。

现在重启 Todo List 程序，浏览器中刷新首页，你会看到浏览器标签页已经显示出 `ICO` 图标了：

![Todo List 图标](Todo_List_ICO.png)

> 本章源码：[chapter5](https://github.com/jianghushinian/todo-list/tree/master/chapter5)
