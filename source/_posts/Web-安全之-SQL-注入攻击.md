---
title: Web 安全之 SQL 注入攻击
date: 2020-01-29 09:25:19
tags:
- Web
- 安全
categories:
- Web
---

SQL 注入攻击是一种非常普遍的攻击方式，也是最危险的 Web 程序安全风险之一。所以学习并防范 SQL 注入攻击是非常必要的。

<!-- more -->

### 攻击原理

在编写 Web 程序时，如果后端代码不对前端用户输入数据做安全性检查，直接将数据当作参数拼接到  SQL 语句中，然后执行 SQL 语句，恶意用户就可以通过传入包含 SQL 关键字的参数来做一些相当危险的操作，如获取敏感数据、删除数据等，以此来达到攻击目的。

### 攻击示例

这里我用一个验证用户登录的例子来作为 SQL 注入攻击的示例。

首先执行如下 Python 代码来生成演示数据。为简单起见，程序中使用 `SQLite` 数据库来进行演示，以下代码创建了一张 `user` 表，包含三个字段分别为 `id`、`username`、`password`，并插入了一条数据，用户名密码都为 `test`。

```python
# demo_db.py

import sqlite3

conn = sqlite3.connect('test.db')
cursor = conn.cursor()

# 创建 user 表并插入数据
cursor.execute('create table user (id int(11) primary key, username varchar(20), password varchar(20))')
cursor.execute('insert into user(id, username, password) values (1, "test", "test")')

cursor.close()
conn.commit()
conn.close()
```

接下来使用 `Flask` 编写一个简单的 Web 应用。执行以下代码之前你需要使用 `pip install flask` 的方式安装 `Flask`。

```python
# demo_sql_inject.py

import sqlite3

from flask import Flask, request

app = Flask(__name__)


@app.route('/login', methods=['GET', 'POST'])
def login():
    # 如果为 POST 请求，则处理登录逻辑
    if request.method == 'POST':
        # 解析前端页面通过 form 表单传递过来的用户名、密码
        form = request.form
        username = form.get('username')
        password = form.get('password')
        print(f'username: {username}, password: {password}')

        # 连接数据库
        conn = sqlite3.connect('test.db')
        cursor = conn.cursor()

        # 根据用户名、密码拼接 SQL 语句
        sql = f"select * from user where username='{username}' and password='{password}'"
        print(f'sql: {sql}')
        # 执行 SQL 语句
        cursor.execute(sql)

        # 如果根据用户输入的用户名、密码能够查询出数据，则登录成功，否则登录失败
        result = cursor.fetchall()
        print(f'result: {result}')
        if result:
            return '登录成功'
        else:
            return '登录失败'
    # GET 请求，则返回登录表单
    else:
        return """
        <form method="post">
            <input name="username" placeholder="username">
            <input name="password" placeholder="password">
            <button>登录</button>
        </form>
        """


if __name__ == '__main__':
    app.run()
```

这个 Web 应用中只包含一个 `login` 视图函数，它分别接收两个请求方法 `GET`、`POST`。如果为 `GET` 请求，返回登录页面，如果为 `POST` 请求，解析前端页面通过 `form` 表单传递过来的用户名、密码，然后根据用户名、密码拼接 SQL 语句，接下来执行这条 SQL 语句来查询 `user` 表中的数据，能够查询出数据，则代表用户输入的用户名和密码正确，登录成功，否则登录失败。

启动 `Flask` 应用，浏览器访问登录页面 `http://127.0.0.1:5000/login`。

<img src="login.png" width="100%" style="max-width: 500px">

首先输入正确的用户名 `test` 和密码 `test`，点击登录按钮，将得到 `登录成功` 响应。

<img src="input_right_info.png" width="100%" style="max-width: 500px">

<img src="login_success.png" width="100%" style="max-width: 500px">

```
# 控制台打印信息
username: test, password: test
sql: select * from user where username='test' and password='test'
result: [(1, 'test', 'test')]
```

然后测试输入错误的用户名或密码，如输入 `123` 作为密码，点击登录按钮，将得到 `登录失败` 响应。

<img src="input_error_info.png" width="100%" style="max-width: 500px">

<img src="login_fail.png" width="100%" style="max-width: 500px">

```
# 控制台打印信息
username: test, password: 123
sql: select * from user where username='test' and password='123'
result: []
```

目前来看，我们的 Web 程序似乎能够正常工作，接下来我将演示如何通过传入 SQL 关键字来实现注入攻击。

这次在用户名的输入框输入 `demo' or 1=1; --`，密码输入框输入 `123`，点击登录按钮，你会惊奇的发现我们使用错误的用户名和密码竟然得到了 `登录成功` 响应。

<img src="input_sql_inject_script_info.png" width="100%" style="max-width: 500px">

<img src="login_success1.png" width="100%" style="max-width: 500px">

```
# 控制台打印信息
username: demo' or 1=1; --, password: 123
sql: select * from user where username='demo' or 1=1; --' and password='123'
result: [(1, 'test', 'test')]
```

此时我们已经通过 SQL 注入攻击的方式绕过了后台系统的验证，前端用户在不知道用户名和密码的情况下实现了登录。

由控制台打印信息可以看到，实际上我们最终拼装的 SQL 语句为 `select * from user where username='demo' or 1=1; --' and password='123'`，在 SQL 中 `;` 用来结束一条语句，`--` 用来注释后面的语句，所以这条 SQL 语句真正被执行的有效部分为 `select * from user where username='demo' or 1=1; `，而这条语句的条件 `or 1=1` 永远成立，所以只要 `user` 表中有数据，总会查出结果，也就能够返回 `登录成功` 响应。

这仅是在不知道用户名密码的情况下实现了登录，实际上还可能出现更糟糕的情况，如果代码中执行 SQL 语句不使用 `cursor.execute(sql)` 方法（`execute` 方法只支持执行单条 SQL 语句），而是使用 `cursor.executescript(sql)` 方法（`executescript` 方法支持执行一段 SQL 语句），并且用户猜到我们的用户表名为 `user`，那么用户将可以在登录页面的 `username` 输入框输入 `demo' or 1=1; drop table user; --` 来删除整张 `user` 表。

由此可见，SQL 注入是一个相当危险的 Web 安全漏洞攻击。

### 防范方法

通过演示示例我们可以发现，SQL 注入攻击方式主要是通过用户输入 SQL 关键字来实现，并且如果用户想删除表数据则需要知道表名，所以防范 SQL 注入主要通过定义复杂表名和对用户输入的 SQL 关键字进行转义。

防范 SQL 注入攻击方法有很多，我这里主要介绍两种比较常见并且开发成本相对较低的解决方案：

1. 可以使用参数化查询的方式执行 SQL 语句，从而避免手动拼接有安全风险的 SQL 语句。我们可以将上面演示的 Web 程序中拼接 SQL 语句的部分代码改为使用参数查询。

```python
# demo_sql_inject.py

import sqlite3

from flask import Flask, request

app = Flask(__name__)


@app.route('/login', methods=['GET', 'POST'])
def login():
    # 如果为 POST 请求，则处理登录逻辑
    if request.method == 'POST':
        # 解析前端页面通过 form 表单传递过来的用户名、密码
        form = request.form
        username = form.get('username')
        password = form.get('password')
        print(f'username: {username}, password: {password}')

        # 连接数据库
        conn = sqlite3.connect('test.db')
        cursor = conn.cursor()

        # 构建参数化查询语句
        sql = "select * from user where username=? and password=?"
        print(f'sql: {sql}')
        # 执行 SQL 语句，将用户名、密码组成元组传入 execute 方法的第二个参数
        cursor.execute(sql, (username, password))

        # 如果根据用户输入的用户名、密码能够查询出数据，则登录成功，否则登录失败
        result = cursor.fetchall()
        print(f'result: {result}')
        if result:
            return '登录成功'
        else:
            return '登录失败'
    # GET 请求，则返回登录表单
    else:
        return """
        <form method="post">
            <input name="username" placeholder="username">
            <input name="password" placeholder="password">
            <button>登录</button>
        </form>
        """


if __name__ == '__main__':
    app.run()
```

可以看到，程序其实只修改了两行代码，修改后的程序不再通过字符串的方式拼接 SQL，而是首先构建参数化查询语句 `select * from user where username=? and password=?`，使用 `?` 对参数进行占位，然后在执行 SQL 时 `cursor.execute(sql, (username, password))` 将用户名和密码组装成元组传入。

再次测试在用户名的输入框输入 `demo' or 1=1; --`，密码输入框输入 `123`，点击登录按钮，将会得到 `登录失败` 响应。

<img src="input_sql_inject_script_info1.png" width="100%" style="max-width: 500px">

<img src="login_fail1.png" width="100%" style="max-width: 500px">

使用 `sqlite3` 驱动自带的参数化查询方式能够有效的避免 SQL 注入攻击，其原理就是在执行 `cursor.execute(sql, (username, password))` 时，`sqlite3` 内部会自动对 SQL 关键字进行转义。

2. 可以使用如 `SQLAlchemy` 等 ORM 来解决 SQL 注入的问题，主流 ORM 框架都是经过市场检验的，对常见的 SQL 注入攻击都有比较有效的解决方案。

当然，所谓安全都是相对的，并没有绝对的安全，对敏感数据加密，做好数据库备份都是非常必要的工作。