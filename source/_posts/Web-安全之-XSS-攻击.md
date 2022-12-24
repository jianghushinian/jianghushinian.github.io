---
title: Web 安全之 XSS 攻击
date: 2021-01-01 09:28:54
tags:
- Web
- 安全
categories:
- Web
---

XSS 攻击中文为跨站脚本攻击，全拼为 Cross-Site Scripting，其缩写本应为 CSS，但这个名称已经被作为 Cascading Style Sheets(层叠样式表) 的缩写，为了避免冲突，就将 Cross(交叉) 缩写成了看起来像一个交叉形状的字母 X。

<!-- more -->

### 攻击原理

XSS 攻击是一种网站应用程序的安全漏洞攻击，是代码注入的一种。它允许恶意用户将代码注入到网页上，其他用户在观看网页时就会受到影响。攻击成功后，攻击者可能得到更高的权限，如窃取网站个人主页信息、Cookie 信息等。

### 攻击分类

根据攻击来源的不同，XSS 攻击一般可以分为三类：存储型、反射型（非持久型） 和 DOM 型。

- 存储型攻击步骤：

	1. 攻击者将恶意代码通过表单等方式提交至目标网站数据库中。
	2. 用户访问目标网站，目标网站后端程序从数据库中提取出恶意代码并返回给浏览器。
	3. 浏览器解析并渲染响应结果的过程中执行了恶意代码。
	4. 恶意代码此时便可以执行窃取用户数据等非法操作。

- 反射型攻击步骤：

	1. 攻击者将恶意代码嵌入至 URL 中并诱导用户点击。
	2. 用户点击恶意 URL，目标网站后端程序从 URL 中提取出恶意代码并返回给浏览器。
	3. 浏览器解析并渲染响应结果的过程中执行了恶意代码。
	4. 恶意代码此时便可以执行窃取用户数据等非法操作。

- DOM 型攻击步骤：

	1. 攻击者将恶意代码嵌入至 URL 中并诱导用户点击。
	2. 用户点击恶意 URL，目标网站前端程序从 URL 中提取出恶意代码。
	3. 浏览器在渲染过程中执行了恶意代码。
	4. 恶意代码此时便可以执行窃取用户数据等非法操作。

存储型攻击和反射型攻击恶意代码都是经由目标网站后端程序返回，存储型攻击恶意代码被存储到了数据库中，反射型攻击恶意代码通常存在于 URL 中，因此也称作非持久型攻击。

DOM 型攻击恶意代码通常也存在于 URL 中，与存储型攻击和反射型攻击区别在于恶意代码不经过后端程序，完全由浏览器端触发。

可以发现 XSS 攻击的本质就是浏览器执行了非法的恶意代码，从而出现用户信息被窃取等非法操作。

### 攻击示例

这里我用 `Flask` 编写一个简单的 Web 应用来作为反射型 XSS 攻击的示例。执行以下代码之前你需要使用 `pip install flask` 的方式安装 `Flask`。

```python
from flask import Flask, request, make_response

app = Flask(__name__)


@app.route('/hello')
def hello():
    name = request.args.get('name')
    response = make_response(f'<h1>Hello {name}!</h1>')
    response.set_cookie('csrftoken', '0yz8xHMCQEfUwuUBIfiOmrre1XV2GyFAYgwdscY4XxpK9BLBuT34nSP1wFhK3GFD')
    return response


if __name__ == '__main__':
    app.run()
```

这个 Web 应用中只包含一个 `hello` 视图函数，它返回一个简单的欢迎页面，并设置了一条 `Cookie` 信息到用户浏览器中。

启动 `Flask` 应用，使用浏览器访问 `http://127.0.0.1:5000/hello?name=xiaoming` 即可看到 `Hello xiaoming!`。

<img src="hello.png" width="100%" style="max-width: 500px">

目前来看，我们的 Web 程序似乎能够正常工作，接下来我将演示如何通过传入恶意代码来实现 XSS 攻击。

这次我们使用浏览器访问 `http://127.0.0.1:5000/hello?name=<script>alert(document.cookie);</script>` 就可以通过 XSS 攻击的方式获取到用户浏览器中的 `Cookie` 信息。

<img src="xss.png" width="100%" style="max-width: 500px">

既然 `alert` 可以执行，并且成功获取到了 `Cookie` 信息，那么就意味着任意 `JavaScript` 代码都可以被执行，这是相当危险的。比如通过 `AJAX` 请求将获取到的 `Cookie` 信息发送至攻击者的服务器，此时攻击者就可以使用窃取到的用户 `Cookie` 信息来做一些非法操作。

由于几种不同类型 XSS 攻击的本质是一样的，所以其他类型 XSS 攻击不再进行演示，读者可以自行尝试。

### 防范方法

防范 XSS 攻击最主要的手段就是在前端渲染 `HTML` 之前对将要渲染的内容进行转义操作。转义后的内容在浏览器中将会被看作文本，而不是可执行的代码。

修改 `hello` 视图函数，不再通过拼接字符串的形式构造响应内容，而是使用 `Jinja2` 模板对内容进行渲染。

```python
from flask import Flask, request, make_response, render_template

app = Flask(__name__)


@app.route('/hello')
def hello():
    name = request.args.get('name')
    response = make_response(render_template('index.html', name=name))
    response.set_cookie('csrftoken', '0yz8xHMCQEfUwuUBIfiOmrre1XV2GyFAYgwdscY4XxpK9BLBuT34nSP1wFhK3GFD')
    return response


if __name__ == '__main__':
    app.run()
```

`index.html` 模板内容如下：

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>Hello</title>
</head>
<body>
<h1>Hello {{ name }}!</h1>
</body>
</html>
```

再次使用浏览器访问 `http://127.0.0.1:5000/hello?name=<script>alert(document.cookie);</script>` 将不会看到 `alert` 弹框。

<img src="xss_fail.png" width="100%" style="max-width: 500px">

此时查看网页源码将会看到 `<script>alert(document.cookie);</script>` 被 `Jinja2` 模板转义成了 `&lt;script&gt;alert(document.cookie);&lt;/script&gt;`。

<img src="source.png" width="100%" style="max-width: 500px">

可以发现 `<script>` 标签中的 `<` 被转义成了 `&lt;`，`>` 被转义成了 `&gt;`。这样，浏览器就不会把用户输入的内容作为代码进行解析了，而是当作普通文本进行渲染，从而避免了 XSS 攻击。

但是转义并不能完全过滤 XSS 攻击。将以上演示代码再次进行修改：

```python
from flask import Flask, request, make_response, render_template

app = Flask(__name__)


@app.route('/hello')
def hello():
    name = request.args.get('name')
    url = 'javascript:alert(document.cookie)'  # 假如这是一个用户提交的 URL
    response = make_response(render_template('index.html', name=name, url=url))
    response.set_cookie('csrftoken', '0yz8xHMCQEfUwuUBIfiOmrre1XV2GyFAYgwdscY4XxpK9BLBuT34nSP1wFhK3GFD')
    return response


if __name__ == '__main__':
    app.run()
```

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>Hello</title>
</head>
<body>
<h1>Hello {{ name }}!</h1>
<a href="{{ url }}">这是一个用户提交的 URL</a>
</body>
</html>
```

在 `index.html` 中添加了一个 `a` 标签作为跳转链接，假设 `javascript:alert(document.cookie)` 是用户提交的非法 `URL`，如果我们仅仅是对内容进行转义，则无法过滤此 XSS 攻击。当用户点击页面中的 `a` 标签时依然会看到 `alert` 弹框。

<img src="a_xss.png" width="100%" style="max-width: 500px">

所以针对 `a` 标签的 `href` 属性，我们还要做特殊处理，判断 `URL` 是否以 `javascript:` 开头，过滤掉非法 `URL`。

虽然 XSS 攻击常见类型只有三种，但可以被攻击的漏洞却非常多。如 `HTML` 中的内嵌文本、`a`标签的 `href` 属性、`img` 标签的 `src` 属性、标签的 `onclick` 等内联事件。一切渲染用户输入数据的地方都有可能存在漏洞，所以开发人员一定要对用户输入的文本进行合适的过滤操作。

不要试图在前端页面用户提交数据的地方对内容进行过滤，因为攻击者可能不通过前端页面进行数据提交。所以要在浏览器渲染代码之前对内容进行过滤。

如果你对 `Jinja2` 比较熟悉，可能用过 `safe` 过滤器，当你了解了 XSS 攻击以后请一定要慎用，确保数据源安全的情况下才可以将不转义的内容原样返回给浏览器。对于 `Vue` 的 `v-html`、`React` 的 `dangerouslySetInnerHTML` 功能同理。

当然，所谓安全都是相对的，并没有绝对的安全，对于 XSS 攻击的防范，开发者应该保持警惕，做到尽量降低安全风险。
