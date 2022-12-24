---
title: 用 Python 撸一个 Web 服务器-第2章：Hello World
date: 2020-07-17 16:36:09
tags: 
- Python
- Web
categories: 
- Web
---

### 从一个 Hello World 程序说起

要编写 Web 服务器，需要用到一个 Python 内置库 `socket`。Socket 是一个比较抽象的概念，中文叫套接字，它代表一个网络连接。两台计算机之间要进行通讯，大概分为三个步骤：建立连接，传输数据，关闭连接。而 `socket` 库为我们提供了这个能力。

<!-- more -->

按照国际惯例，我们将通过编写一个 Hello World 程序来开始 Web 服务器的学习 。

首先要创建一个基于 `TCP` 的 `socket` 对象：

```python
# 导入 socket
import socket

# 创建 socket 对象
sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
```

`socket.socket()` 方法用来创建一个 `socket` 对象。同时我们给它传递了两个参数：`socket.AF_INET` 表示使用IPv4 协议，`socket.SOCK_STREAM` 表示这是一个基于 `TCP` 的 `socket` 对象。这两个参数也是默认参数，都可以不传。

HTTP 协议是基于请求 —— 响应模型的，请求只可以是客户端发起的，服务器进行响应。服务器并不具备主动发起请求的能力，但是它需要被动的等待客户端的请求。所以现在有了 `socket` 对象以后我们接下来要做的就是监听客户端的请求：

```python
# 绑定 IP 和端口
sock.bind(('127.0.0.1', 8000))
# 开始监听
sock.listen(5)
```

`socket` 对象的 `bind` 方法用来绑定监听的 IP 地址和端口，它接收一个由 IP 和端口组成的 `tuple` 作为参数，`127.0.0.1` 代表本机 IP，只有运行在本机上的浏览器才能连接。端口号允许范围在 `0~65535` 之间，但是小于 `1024` 的端口号需要管理员权限才可使用。`sock.listen(5)` 用来开启监听，等待连接的最大数量指定为 `5`。

开启监听以后，就可以等待接收客户端的请求了：

```python
client, addr = sock.accept()
```

`sock.accept()` 会阻塞程序，等待客户端的连接，一旦有客户端连接上来，它会分别返回客户端连接对象和客户端的地址。

与客户端建立好连接后，接下来就是接收客户端发来的请求数据：

```python
data = b''
while True:
    chunk = client.recv(1024)
    data += chunk
    if len(chunk) < 1024:
        break
```

接收客户端请求数据需要调用客户端连接对象的 `recv` 方法，参数为每一次接收的数据长度。`socket` 通讯过程中的数据都为 Python 的 `bytes` 类型。这里每次接收 1024 个字节，等待数据全部接收完成退出循环。

接收到客户端发来的数据后，就需要对数据进行处理，然后返回响应给客户端的浏览器：

```python
# 打印从客户端接收的数据
print(f'data: {data}')
# 给客户端发送响应数据
client.sendall(b'HTTP/1.1 200 OK\r\nContent-Type: text/html\r\n\r\n<h1>Hello World</h1>')
```

为了简单起见，在接收到客户端发来的数据后直接进行打印，并没有做进一步的解析处理。接着就是服务器给客户端发送响应数据。发送的数据同样为 `bytes` 类型。数据按照 HTTP 协议的规范进行组装，首先是状态行 `HTTP/1.1 200 OK`，紧跟是着一个换行符 `\r\n`，然后通过响应头 `Content-Type: text/html` 指定响应结果为 HTML 类型，接下来是两个连续的 `\r\n\r\n`，注意因为在响应头和响应报文之间隔着一个空行，所以才会出现两个连续的 `\r\n\r\n`，最后就是响应体部分 `<h1>Hello World</h1>`。

在发送完响应数据后，我们需要关闭客户端连接对象和服务端 `socket` 对象：

```python
# 关闭客户端连接对象
client.close()
# 关闭 socket 对象
sock.close()
```

至此，一个 `Hello World` 服务器程序编写完成，下面是完整代码：

```python
# server.py

import socket


def main():
    # 创建 socket 对象
    sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    # 允许端口复用
    sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    # 绑定 IP 和端口
    sock.bind(('127.0.0.1', 8000))
    # 开始监听
    sock.listen(5)

    # 等待客户端请求
    client, addr = sock.accept()
    print(f'client type: {type(client)}\naddr: {addr}')

    # 接收客户端发来的数据
    data = b''
    while True:
        chunk = client.recv(1024)
        data += chunk
        if len(chunk) < 1024:
            break

    # 打印从客户端接收的数据
    print(f'data: {data}')
    # 给客户端发送响应数据
    client.sendall(b'HTTP/1.1 200 OK\r\nContent-Type: text/html\r\n\r\n<h1>Hello World</h1>')

    # 关闭客户端连接对象
    client.close()
    # 关闭 socket 对象
    sock.close()


if __name__ == '__main__':
    main()
```

将以上代码写入到 `server.py` 文件中。然后在终端中使用 Python 运行此文件：`python3 server.py`。

![运行 Hello World 程序](Hello_World_Server.png)

打开浏览器，地址栏输入 `http://127.0.0.1:8000`，你将得到如下结果：

![Hello World](Hello_World.png)

`Hello World`！浏览器成功渲染出了服务器的响应结果。

回到终端可以查看打印出来的客户端请求信息：

![客户端请求信息](Hello_World_Client_Request_Data.png)

可以发现，客户端连接对象实际上也是一个 `socket` 对象，客户端 IP 地址为 `127.0.0.1` 端口为 `50510`。最后是客户端请求数据，只有请求行和请求头，由于没有请求体，所以最后以两个连续的 `\r\n\r\n` 结束。

细心的读者可能已经发现在最后给出的完整的 Hello World 程序代码中，在创建 `socket` 对象后有一行：

```python
sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
```

前面并没有介绍这行代码的作用，实际上它的作用是允许端口复用。如果不写这行代码，那么在程序运行完成后需要马上重启程序时，由于上次的端口还在占用，会导致程序抛出异常，端口需要在间隔一段时间后才会被释放允许使用。加上这行代码就不会出现此问题，方便调试。

以上，我们实现了一个简单的能够返回 Hello World 的服务器程序。


### 让服务器永久运行

上面实现的 Hello World 服务器程序运行一次就退出了。通常来说，服务器端的程序是永久运行的程序。因为你不知道客户端什么时候发送请求，所以就需要服务器端一直处在监听状态。这样才能保证任何时候客户端发送请求都能被服务器端接收到。

```python
# server_forever.py

import socket


def main():
    # 创建 socket 对象
    sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    # 允许端口复用
    sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    # 绑定 IP 和端口
    sock.bind(('127.0.0.1', 8000))
    # 开始监听
    sock.listen(5)

    while True:
        # 等待客户端请求
        client, addr = sock.accept()
        print(f'client type: {type(client)}\naddr: {addr}')

        # 接收客户端发来的数据
        data = b''
        while True:
            chunk = client.recv(1024)
            data += chunk
            if len(chunk) < 1024:
                break

        # 打印从客户端接收的数据
        print(f'data: {data}')
        # 给客户端发送响应数据
        client.sendall(b'HTTP/1.1 200 OK\r\nContent-Type: text/html\r\n\r\n<h1>Hello World</h1>')

        # 关闭客户端连接对象
        client.close()


if __name__ == '__main__':
    main()
```

上面的程序中加入了一个 `while True` 无限循环，在处理完一个客户端连接对象以后程序马上执行到下一次循环，开始等待新的客户端连接，这样就实现了服务器程序永久运行。并且删除了 `main` 函数最后一行 `sock.close()` 代码，因为既然要让程序永久运行下去，那么也就不需要关闭服务器端 `socket` 连接了。

将以上代码保存到 `server_forever.py` 文件中，同样在命令行终端使用 Python 运行此程序，浏览器多刷新几次页面，依然能够正常加载 `Hello World`。

不过，此时如果在终端查看打印信息，会发现每次刷新浏览器时，浏览器并不是一次只发送一个请求，而是两个请求。

![客户端请求信息](Hello_World_Client_Request_Data2.png)

打开 Chrome 控制台查看 `Network`，果然浏览器发送了两个请求。

![Chrome 请求](Hello_World_Chrome_Request_Data.png)

第一个请求路径为 `/`，根据浏览器请求及响应记录来看是符合预期的。

![Hello World 请求头及响应头信息](Hello_World_Request_Headers_And_Response_Headers.png)

![Hello World 响应信息](Hello_World_Response.png)

第二个请求路径为 `/favicon.ico`，这个请求的响应结果同样为 `<h1>Hello World</h1>`。

![网站图标请求头及响应头信息](Favicon_Request_Headers_And_Response_Headers.png)

![网站图标响应信息](Favicon_Response.png)

实际上，这个请求是 Chrome 浏览器自主发起的，是为了获取网站图标用的。当在浏览器中打开京东网站首页时，浏览器标签栏就会加载出京东网站的图标。

![京东网站图标](JD_Favicon.png)

我们自己编写的 Hello World 服务器由于没有返回正确的图标文件，而是返回了一个 `<h1>Hello World</h1>` 字符串，所以浏览器并不能将其识别为图标。最终在 `Hello World` 页面标签栏也就不会有像京东网站类似的图标了。这个问题目前来说我们并不需要关心，在之后实现 Todo List 程序时再来解决。

有些读者可能会疑惑为什么 Hello World 服务器返回的是一个不完整的 HTML 页面，只是一个带有 `h1` 标签的字符串 `<h1>Hello World</h1>`，浏览器就能够正常渲染页面，并对 `Hello World` 做加粗处理。这其实是 Chrome 浏览器的容错机制，如果检测到 HTML 标签不全，那么它会自动补全缺少的标签。以达到更好的渲染效果。

现在如果要结束服务器程序，只需要在程序运行终端按组合键 `Ctrl + C` 即可。


### 让服务器同时支持多个客户端连接

我们现在实现的 Hello World 服务器程序由于是单线程的，所以服务器一次只能处理一个请求。但我们使用的京东等网站实际上同时会有很多客户端在连接的，如果一次只能处理一个请求，那么客户端体验将非常差。

为了让我们的程序也能支持同时处理多个客户端连接，需要将其改成多线程版本。

```python
# threading_server_forever.py

import socket
import threading


def process_connection(client):
    """处理客户端连接"""
    # 接收客户端发来的数据
    data = b''
    while True:
        chunk = client.recv(1024)
        data += chunk
        if len(chunk) < 1024:
            break

    # 打印从客户端接收的数据
    print(f'data: {data}')
    # 给客户端发送响应数据
    client.sendall(b'HTTP/1.1 200 OK\r\nContent-Type: text/html\r\n\r\n<h1>Hello World</h1>')

    # 关闭客户端连接对象
    client.close()


def main():
    # 创建 socket 对象
    sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    # 允许端口复用
    sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    # 绑定 IP 和端口
    sock.bind(('127.0.0.1', 8000))
    # 开始监听
    sock.listen(5)

    while True:
        # 等待客户端请求
        client, addr = sock.accept()
        print(f'client type: {type(client)}\naddr: {addr}')

        # 创建新的线程来处理客户端连接
        t = threading.Thread(target=process_connection, args=(client,))
        t.start()


if __name__ == '__main__':
    main()
```

改成多线程版本以后，服务器每接收到一个客户端连接，就将其交给一个新的子线程来处理，主线程继续执行到下一轮循环等待新的客户端连接。这样，就实现了让服务器同时支持多个客户端连接。

本章通过编写一个 Hello World 程序学习了 Web 服务器的开发 。如果你是编程新手，对 `socket` 编程理解起来还是略有困难，那么你可以类比 Python 的文件操作来进行对比学习。文件处理通常也是三个步骤：打开文件、读写数据、关闭文件。通过这样利用已有知识来类比学习新技术也是一个不错的方法。

> 本章源码：[chapter2](https://github.com/jianghushinian/todo-list/tree/master/chapter2)
