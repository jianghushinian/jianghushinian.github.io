---
title: 用 Python 撸一个 Web 服务器-第4章：动态渲染数据
date: 2020-07-17 21:17:42
tags: 
- Python
- Web
categories: 
- Web
---

上一章中为了尽快让 Todo List 程序跑起来，并没有完全按照 `MVC` 模式编写程序。这一章就让我们一起实现一个完整的 `MVC` 模式 Todo List 程序首页。

<!-- more -->

### 使用模型操作数据

我们来分析下请求 Todo List 程序首页时，模型层需要做哪些事情。当一个请求到达首页视图函数 `index` 时，它需要做两件事情，首先调用模型层获取全部的 todo 数据，然后将 todo 数据动态填充到 `index.html` 模板中。

调用模型层获取全部的 todo 数据，只需要在模型层编写读取 `todo/db/todo.json` 文件数据的代码即可。在这之前，我们需要先确定 todo 在文件中存储的格式。

Todo List 程序中 todo 需要存储的数据只有一个，就是 todo 的内容。所以我们可以将 todo 以如下格式存储到 `todo/db/todo.json` 文件：

```json
// todo_list/todo/db/todo.json

[
    {
        "id": 1,
        "content": "hello world"
    },
    {
        "id": 2,
        "content": "你好，世界！"
    }
]
```

这是一个标准的 `JSON` 格式，每一个对象代表了一条 todo，`content` 字段即为 todo 内容，`id` 作为每条数据的索引不会展示在页面中，方便我们对数据进行排序、快速查找等操作。 

为了简化程序，我将数据存储在 `JSON` 文件中而不是数据库中。存储到文件的格式多种多样，但 `JSON` 格式是一种非常流行且友好的数据格式，在 Python 中也能够很方便的对 `JSON` 格式的文件进行读写操作。

注意：

1. `JSON` 文件不支持注释，所以如果你打算直接从上面示例中复制数据到 `todo.json` 文件时，需要去掉顶部文件名注释。
2. 如果 `todo/db/todo.json` 文件内容为空，使用 Python 读取时会抛出 `JSONDecodeError` 异常，起码要保证其内部有一个空数组 `[]` 存在，才能正常读取。

确定了 `todo/db/todo.json` 文件数据格式，就可以编写在模型层读取 todo 数据的代码了：

```python
# todo_list/todo/models.py

import os
import json

from todo.config import BASE_DIR


class Todo(object):
    """
    Todo 模型类
    """

    def __init__(self, **kwargs):
        self.id = kwargs.get('id')
        self.content = kwargs.get('content', '')

    @classmethod
    def _db_path(cls):
        """获取存储 todo 数据文件的绝对路径"""
        # 返回 'todo_list/todo/db/todo.json' 文件的绝对路径
        path = os.path.join(BASE_DIR, 'db/todo.json')
        return path

    @classmethod
    def _load_db(cls):
        """加载 JSON 文件中所有 todo 数据"""
        path = cls._db_path()
        with open(path, 'r', encoding='utf-8') as f:
            return json.load(f)

    @classmethod
    def all(cls, sort=False, reverse=False):
        """获取全部 todo"""
        # 这一步用来将所有从 JSON 文件中读取的 todo 数据转换为 Todo 实例化对象，方便后续操作
        todo_list = [cls(**todo_dict) for todo_dict in cls._load_db()]
        # 对数据按照 id 进行排序
        if sort:
            todo_list = sorted(todo_list, key=lambda x: x.id, reverse=reverse)
        return todo_list
```

定义 `Todo` 模型类来操作 todo 数据。`Todo` 模型类的 `all` 方法用来读取全部的 todo 数据，在其内部将所有从 `JSON` 文件中读取的 todo 数据转换为 `Todo` 实例化对象并组装成 `list` 返回。`all` 方法还可以对数据进行排序，排序操作实际上转发给了 Python 内置的 `sorted` 函数来完成。

有了全部的 todo 数据，下一步操作就是将 todo 数据动态填充到 `todo/templates/index.html` 模板中。

### 使用模板引擎渲染 HTML

上一章实现的 Todo List 程序返回的首页数据都是固定写死在 `todo/templates/index.html` 代码中的。现在需要动态填充 todo 内容，我们需要学习一个新的概念叫作 `模板渲染`。

首先我们编写的 HTML 页面不再是完全使用 HTML 的标签来编写，而需要使用一些占位变量来替换需要动态填充的部分，这样编写出来的 HTML 页面通常称为模板。将 HTML 模板读取到内存中，使用真实的 todo 数据来替换掉占位变量而获得最终将要返回的字符串数据，这个过程称为渲染。能够实现读取 HTML 中的占位变量并正确替换为真实值的代码称为模板引擎。

Todo List 程序首页主体部分代码如下：

```html
<h1 class="container">Todo List</h1>
<div class="container">
    <ul>
        <li>
            <div>Hello World</div>
        </li>
        <li>
            <div>你好，世界！</div>
        </li>
    </ul>
</div>
```

其中每一个 `li` 标签代表一条 todo，显然 todo 的条数是不确定的，所以每一个 `li` 标签都需要动态生成。根据这段 HTML 代码，可以编写出如下模板：

```html
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
```

这段模板代码中只保留了一对 `li` 标签，它被嵌套在 `for` 循环中，`for` 语句块从 `{% for todo in todo_list %}` 开始，到 `{% endfor %}`  结束。`todo_list` 变量是在模板渲染阶段传进来的由所有 todo 对象组成的 `list`，`list` 中有多少个元素就会渲染多少个 `li` 标签。`for` 循环内部使用了循环变量 `todo`，`{{ todo.content }}` 表示获取 `todo` 变量的 `content` 属性，这与 Python 中获取对象的属性语法相同。

了解了模板语法，我们还需要有一个能够读懂模板语法的模板引擎。Todo List 程序的 HTML 模板只会用到 `for` 循环和模板变量这两种语法，所以我们将要实现的模板引擎只需要能够解析这两种语法即可。

```python
# todo_list/todo/utils.py

class Template(object):
    """模板引擎"""

    def __init__(self, text, context):
        # 保存最终结果
        self.result = []
        # 保存从 HTML 中解析出来的 for 语句代码片段
        self.for_snippet = []
        # 上下文变量
        self.context = context
        # 使用正则匹配出所有的 for 语句、模板变量
        self.snippets = re.split('({{.*?}}|{%.*?%})', text, flags=re.DOTALL)
        # 标记是否为 for 语句代码段
        is_for_snippet = False

        # 遍历所有匹配出来的代码片段
        for snippet in self.snippets:
            # 解析模板变量
            if snippet.startswith('{{'):
                if is_for_snippet is False:
                    # 去掉花括号和空格，获取变量名
                    var = snippet[2:-2].strip()
                    # 获取变量的值
                    snippet = self._get_var_value(var)
            # 解析 for 语句
            elif snippet.startswith('{%'):
                # for 语句开始代码片段 -> {% for todo in todo_list %}
                if 'in' in snippet:
                    is_for_snippet = True
                    self.result.append('{}')
                # for 语句结束代码片段 -> {% endfor %}
                else:
                    is_for_snippet = False
                    snippet = ''

            if is_for_snippet:
                # 如果是 for 语句代码段，需要进行二次处理，暂时保存到 for 语句片段列表中
                self.for_snippet.append(snippet)
            else:
                # 如果是模板变量，直接将变量值追加到结果列表中
                self.result.append(snippet)

    def _get_var_value(self, var):
        """根据变量名获取变量的值"""
        # 如果 '.' 不在变量名中，直接在上下文变量中获取变量的值
        if '.' not in var:
            value = self.context.get(var)
        # '.' 在变量名中（对象.属性），说明是要获取对象的属性
        else:
            obj, attr = var.split('.')
            value = getattr(self.context.get(obj), attr)

        # 保证返回的变量值为字符串
        if not isinstance(value, str):
            value = str(value)
        return value

    def _parse_for_snippet(self):
        """解析 for 语句片段代码"""
        # 保存 for 语句片段解析结果
        result = []
        if self.for_snippet:
            # 解析 for 语句开始代码片段
            # '{% for todo in todo_list %}' -> ['for', 'todo', 'in', 'todo_list']
            words = self.for_snippet[0][2:-2].strip().split()
            # 从上下文变量中获取 for 语句中的可迭代对象
            iter_obj = self.context.get(words[-1])
            # 遍历可迭代对象
            for i in iter_obj:
                # 遍历 for 语句片段的代码块
                for snippet in self.for_snippet[1:]:
                    # 解析模板变量
                    if snippet.startswith('{{'):
                        # 去掉花括号和空格，获取变量名
                        var = snippet[2:-2].strip()
                        # 如果 '.' 不在变量名中，直接将循环变量 i 赋值给 snippet
                        if '.' not in var:
                            snippet = i
                        # '.' 在变量名中（对象.属性），说明是要获取对象的属性
                        else:
                            obj, attr = var.split('.')
                            # 将对象的属性值赋值给 snippet
                            snippet = getattr(i, attr)
                    # 保证变量值为字符串
                    if not isinstance(snippet, str):
                        snippet = str(snippet)
                    # 将解析出来的循环变量结果追加到 for 语句片段解析结果列表中
                    result.append(snippet)
        return result

    def render(self):
        """渲染"""
        # 获取 for 语句片段解析结果
        for_result = self._parse_for_snippet()
        # 将渲染结果组装成字符串并返回
        return ''.join(self.result).format(''.join(for_result))


def render_template(template, **context):
    """渲染模板"""
    # 读取 'todo_list/todo/templates' 目录下的 HTML 文件内容
    template_dir = os.path.join(BASE_DIR, 'templates')
    path = os.path.join(template_dir, template)

    with open(path, 'r', encoding='utf-8') as f:
        # 将从 HTML 中读取的内容传递给模板引擎
        t = Template(f.read(), context)

    # 调用模板引擎的渲染方法，实现模板渲染
    return t.render()
```

`Template` 类就是我们为 Todo List 程序实现的模板引擎。模板引擎的代码有些复杂，我写了比较详细的注释来帮助你理解。模板渲染的大概过程如下：

首先实例化 `Template` 对象，`Template` 对象的初始化方法 `__init__` 需要传递两个参数，分别是 HTML 字符串和保存了模板所需变量的 `dict`，在初始化时会解析出 HTML 中所有的 `for` 语句和模板变量，模板变量直接被替换为对应的值，`for` 语句代码段则被暂存起来，等到需要真正渲染模板时，调用模板引擎实例对象的 `render` 方法，完成 `for` 语句的解析和值替换，最终将渲染结果组装成字符串并返回。

`render_template` 函数的代码也做了相应的调整，它的功能不再只是读取 HTML 内容，而是需要在内部调用模板引擎获取渲染结果。

对于基础薄弱的读者来说可能模板引擎部分的代码不太好理解，那么暂时先不必深究，你只需要知道模板引擎干了什么，明白它的原理无非是将 HTML 字符串中的模板语法全部找出来，然后根据语法规则将其替换成真正的变量值，最后渲染成正确的 HTML。本质上还是字符串的拼接，就像 Python 字符串的 `format` 方法一样，它能够找到字符串中的花括号 `{}`，然后替换成传递给它的参数值。


### MVC 模式的 Todo List 程序首页

我们已经介绍了使用模型操作数据和使用模板引擎渲染 HTML，现在就可以用动态渲染的 HTML 首页替换之前的静态首页了。

修改首页 `todo/templates/index.html` 的 HTML 代码为一个模板：

```html
<!-- todo_list/todo/templates/index.html -->

<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>Todo List</title>
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

这里我暂时去掉了 HTML 顶部的 CSS 样式，因为我们的模板引擎不支持这种直接将 CSS 嵌入在 HTML 中的写法，之后我会介绍如何通过 `link` 标签来引入外部样式。

我们还要对 `index` 视图函数做些修改，在视图函数内部调用 `Todo` 模型的 `all` 方法来获取所有 todo，然后传递给模板引擎对 HTML 进行渲染，得到最终结果。修改后的代码如下：

```python
# todo_list/todo/controllers.py

from todo.utils import render_template
from todo.models import Todo


def index():
    """首页视图函数"""
    # 倒序排序，最近添加的 todo 排在前面
    todo_list = Todo.all(sort=True, reverse=True)
    context = {
        'todo_list': todo_list,
    }
    return render_template('index.html', **context)
```

在终端中进入项目根目录 `todo_list/` 下，使用 Python 运行 `server.py` 文件，将得到经过动态渲染的 Todo List 程序首页：

![Todo List 首页](Todo_List_Index_Use_Template.png)

现在 Todo List 程序首页已经是动态渲染的了，下一章我们就来解决样式问题。

> 本章源码：[chapter4](https://github.com/jianghushinian/todo-list/tree/master/chapter4)
