---
title: 用 Python 撸一个 Web 服务器-第7章：重构——更好的组织代码
date: 2020-07-17 21:30:51
tags: 
- Python
- Web
categories: 
- Web
---

通过前几章的学习，我们完成了 Todo List 程序的 todo 管理部分，实现了对 todo 的增、删、改、查基本操作，这也是几乎所有 Web 程序都具备的功能。我们当然可以按照目前的思路继续来实现用户管理部分，在 `models.py` 中编写用户相关的模型，在 `templates/` 目录下新建用户相关 HTML，在 `controllers.py` 中编写用户相关的视图函数。但是，随着新功能的加入，把不同功能的代码都写在相同的文件中必然会引起代码的混乱。为实现易维护、易扩展的代码，我们需要对项目的目录结构进行重构。

<!-- more -->

### 项目重构

目前为止，我们实现的 Todo List 程序目录结构如下:

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
    ├── static
    │   ├── css
    │   │   └── style.css
    │   └── favicon.ico
    ├── templates
    │   ├── edit.html
    │   └── index.html
    └── utils.py
```

重构后的目录结构如下:

```
todo_list
├── server.py
└── todo
    ├── __init__.py
    ├── config.py
    ├── controllers
    │   ├── __init__.py
    │   ├── static.py
    │   └── todo.py
    ├── db
    │   └── todo.json
    ├── models
    │   ├── __init__.py
    │   └── todo.py
    ├── static
    │   ├── css
    │   │   └── style.css
    │   └── favicon.ico
    ├── templates
    │   └── todo
    │       ├── edit.html
    │       └── index.html
    └── utils
        ├── __init__.py
        ├── http.py
        └── templating.py
```

首先，将原来的 `controllers.py` 文件换成了 `controllers/` 包，在 `controllers/` 目录下将视图函数按照功能分别放到不同的文件中，并在 `controllers/__init__.py` 中将这些视图函数汇集到一起。将读取静态资源的视图函数 `static` 和读取网页 `ICO` 图标的视图函数 `favicon` 都放到  `controllers/static.py` 中，将 todo 相关的视图函数都放到 `controllers/todo.py` 中。

同样的，将 `models.py` 文件换成 `models/` 包，将原来的 `Todo` 模型类放到 `models/todo.py` 中。不过这里不只是简单的将原来的 `Todo` 模型代码迁移过来，还对其进行了重构，抽象出一个模型基类 `Model` 将其放到 `models/__init__.py` 中，然后 `Todo` 继承自 `Model` 模型基类。这样做的好处是等我们编写用户模型时，查找、保存等方法就不需要在用户模型中再写一遍了，只需要让用户模型也继承 `Model` 模型基类即可。

```python
# todo_list/todo/models/__init__.py

import os
import json

from todo.config import BASE_DIR


class Model(object):
    """
    Model 模型类
    """

    @classmethod
    def _db_path(cls):
        """获取存储模型对象数据的文件的绝对路径"""
        class_name = cls.__name__
        file_name = f'{class_name.lower()}.json'
        path = os.path.join(BASE_DIR, 'db', file_name)
        return path

    @classmethod
    def _load_db(cls):
        """加载 JSON 文件中所有模型对象数据"""
        path = cls._db_path()
        with open(path, 'r', encoding='utf-8') as f:
            return json.load(f)

    @classmethod
    def _save_db(cls, data):
        """将模型对象数据保存到 JSON 文件"""
        path = cls._db_path()
        with open(path, 'w', encoding='utf-8') as f:
            json.dump(data, f, ensure_ascii=False, indent=4)

    @classmethod
    def all(cls, sort=False, reverse=False):
        """查询全部模型对象"""
        # 这一步用来将所有从 JSON 文件中读取的 model 数据转换为 Model 实例化对象，方便后续操作
        models = [cls(**model) for model in cls._load_db()]
        # 对数据按照 id 排序
        if sort:
            models = sorted(models, key=lambda x: x.id, reverse=reverse)
        return models

    @classmethod
    def find_by(cls, limit=-1, ensure_one=False, sort=False, reverse=False, **kwargs):
        """根据传入条件查询模型对象"""
        result = []
        models = [model.__dict__ for model in cls.all(sort=sort, reverse=reverse)]
        for model in models:
            # 根据关键字参数查询 model
            for k, v in kwargs.items():
                if model.get(k) != v:
                    break
            else:
                result.append(cls(**model))

        # 查询给定条数的数据
        if 0 < limit < len(result):
            result = result[:limit]
        # 查询结果集中的第一条数据
        if ensure_one:
            result = result[0] if len(result) > 0 else None

        return result

    @classmethod
    def get(cls, id):
        """通过 id 查询模型对象"""
        result = cls.find_by(id=id, ensure_one=True)
        return result

    def save(self):
        """保存模型对象"""
        # 查找出除 self 以外所有 model
        # model.__dict__ 是保存了所有实例属性的字典
        models = [model.__dict__ for model in self.all(sort=True) if model.id != self.id]

        # 自增 id
        if self.id is None:
            # 如果 model_list 大于 0 说明不是第一条 model，取最后一条 model 的 id 加 1
            if len(models) > 0:
                self.id = models[-1]['id'] + 1
            # 否则说明是第一条 model，id 为 1
            else:
                self.id = 1

        # 将当前 model 追加到 model_list
        models.append(self.__dict__)
        # 将所有 model 保存到文件
        self._save_db(models)

    def delete(self):
        """删除模型对象"""
        model_list = [model.__dict__ for model in self.all() if model.id != self.id]
        self._save_db(model_list)
```

```python
# todo_list/todo/models/todo.py

from . import Model


class Todo(Model):
    """
    Todo 模型类
    """

    def __init__(self, **kwargs):
        self.id = kwargs.get('id')
        self.content = kwargs.get('content', '')
```

存放 HTML 模板的 `templates/` 目录中又增加了一层目录结构，`todo/` 目录用来存放 todo 相关 HTML 模板。

原来的工具集 `utils.py` 文件也换成了一个 Python 包，根据工具代码的不同类型将其分别放入不同文件。请求类 `Request`、响应类 `Response` 和重定向函数 `redirect` 都放到 `utils/http.py` 中。模板引擎类 `Template`、渲染模板函数 `render_template` 放到 `utils/templating.py` 中。

至此，项目目录结构重构完成。在这里大部分是采用将原来的单文件改成 Python 包的形式，这样能更好的组织代码结构。

在编写用户管理功能之前，我们先来介绍下 Web 开发过程中两个重要的部分，日志和测试。日志和测试是保证生产环境项目稳定运行的重要保障，日志可以记录程序的异常信息和对程序的运行状况进行监控、分析等，测试则能够有效降低生产环境中程序出现 `BUG` 的概率。

### 日志

Todo List 程序之前记录日志的方式是通过 `print` 函数来实现的，在前几章的代码中可以找到很多 `print` 语句。不过 `print` 函数默认将结果输出到屏幕，而生产环境中通常需要将日志输出到文件中保存下来，方便后续对日志进行分析。我们可以通过给 `print` 函数指定 `file` 参数（一个文件对象）将其输出内容写入文件。

在 `utils/` 目录下新建 `logging.py` 用来编写日志记录函数:

```python
# todo_list/utils/logging.py

import os
import datetime

from todo.config import BASE_DIR

path = os.path.join(BASE_DIR, 'logs/todo.log')


def logger(*args, **kwargs):
    """记录日志"""
    now = datetime.datetime.strftime(datetime.datetime.now(), '%Y-%m-%d %H:%M:%S')
    with open(path, 'a') as f:
        # 将日志输出到屏幕，方便调试，上线时可关掉
        print(now, '-', *args, **kwargs)
        # 将日志输出到文件
        print(now, '-', *args, **kwargs, file=f)
```

接着在 `todo_list/` 目录下新建一个 `logs/` 目录用来存放日志。最后将代码中所有 `print` 语句全部换成 `logger` 即可。这样日志能够同时输出到屏幕和文件，如果在生产环境则可以只输出到文件。


### 测试

测试在程序开发过程中占有举足轻重的地位，但很多团队和开发者却对其视而不见，以各种理由忽略测试。尤其是越小的团队越以开发周期短、时间紧为由省略开发程序的测试过程。但我一直认为这是得不偿失的做法，短期内可能加快了程序开发的进度，但长远来看，后期投入的开发维护精力、成本等将会大大增加。并且生产环境一旦出现严重漏洞，将带来不可挽回的损失。

尽管 Todo List 程序非常微小，但还是要对其加入测试。程序测试方法有很多，如单元测试、功能测试、集成测试等。这里着重介绍下单元测试，单元测试是指对软件中的最小可测试单元进行检查和验证。其中所谓的最小可测单元可以是一个函数、一个类等。

在 `todo_list/` 目录下新建 `tests/` 目录用来存放所有测试文件，测试代码根据被测代码类型的不同分别放到不同文件中，如测试视图函数的代码全部放到名为 `test_controllers.py` 的文件中，测试模型的代码全部放到名为 `test_models.py` 的文件中。一个约定俗成的做法是让所有的测试文件名都以 `test_` 开头。

接下来我以测试首页视图函数和新增 todo 视图函数为例，讲解测试代码的编写：

```python
# todo_list/tests/test_controllers.py

import os
import sys

sys.path.insert(0, os.path.dirname(os.path.dirname(os.path.abspath(__file__))))

from todo.utils.http import Request
from todo.controllers import routes
from todo.models.todo import Todo


def test_index():
    """测试首页"""
    request_message = 'GET / HTTP/1.1\r\nHost: 127.0.0.1:8000\r\n\r\n'
    request = Request(request_message)
    route, method = routes.get(request.path)
    r = route(request)

    assert b'Todo List' in bytes(r, encoding='utf-8')
    assert b'/new' in bytes(r, encoding='utf-8')


def test_new():
    """测试新增 todo"""
    # 生成随机 todo 内容
    content = uuid.uuid4().hex
    request_message = f'POST /new HTTP/1.1\r\nHost: 127.0.0.1:8000\r\n\r\ncontent={content}'
    request = Request(request_message)
    route, method = routes.get(request.path)
    r = route(request)
    t = Todo.find_by(content=content, ensure_one=True)
    t.delete()

    assert b'302 FOUND' in bytes(r)
    assert b'/index' in bytes(r)
    assert t.content == content


def main():
    test_index()
    test_new()


if __name__ == '__main__':
    main()
```

 `test_index` 函数用来测试首页视图函数，为了简化测试代码，测试函数中并没有通过请求 Web Server 来获取响应。首先将请求消息报文 `request_message` 传递给 `Resquest` 类构造了一个请求对象，然后根据请求路径 `request.path` 获取处理该请求的视图函数，接着调用视图函数来获取响应报文。这样做的好处是不需要编写发起请求的客户端程序，但测试覆盖率肯定会有所下降。这是一个选择性的问题，需要考虑时间成本、投入产出比等。测试函数的最后通过断言语句，来断言响应报文中必然包含的内容。

`test_new` 函数用来测试新增 todo 视图函数，大体逻辑与 `test_index` 差不多。在生成测试 todo 内容时使用了 `UUID`，目的是为了生成足够随机的字符串避免与 `db/todo.json` 中已存在的 todo 内存重复，这样通过 `Todo.find_by()` 方法查找 todo 时能够确保查询结果正确。还需要注意的一点是在新增 todo 成功后又将其删除了，这样做的目的是为了让测试代码不对原有数据产生影响。理论上，测试代码每次执行的结果都应该相同，并且不应该破坏程序原有的数据。

使用 Python 运行测试文件 `python3 test_controllers.py`，如果测试代码执行完成后没有任何输出，就说明全部测试通过。测试代码遵循 Linux 设计哲学，没有消息就是最好的消息。如果测试代码执行过程中抛出 `AssertionError` 异常，则说明测试未通过，要么是被测代码有问题，要么是测试代码本身有问题。

由于篇幅所限，对 Todo List 程序的测试部分讲解就到这里，其他部分的测试代码可以访问本章节源码进行查看。

> 本章源码：[chapter7](https://github.com/jianghushinian/todo-list/tree/master/chapter7)
