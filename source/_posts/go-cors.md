---
title: '2024 都要过完了，我不允许你在 Go 中还不会解决 CORS 跨域问题'
date: 2024-11-18 23:03:15
tags:
- Go
categories:
- Go
---

大家好，我是[江湖十年](https://jianghushinian.cn/about/)，今天来简单聊聊我们在 Web 开发中绕不开的一个话题——跨域。

关于跨域以及如何解决跨域，算是老生常谈的问题了。虽然这个问题很常见且在网上一搜就能找到大把解决方案。但我在工作中还是会经常发现有人不理解或无法解释清楚到底什么是跨域，本文就来一次性解释清楚跨域问题，让你知其然并能知其所以然。

<!-- more -->

### 什么是跨域

要理解什么是跨域，以及为什么会出现跨域问题，就得从浏览器的同源策略来说起。

[同源策略](https://zh.wikipedia.org/wiki/%E5%90%8C%E6%BA%90%E7%AD%96%E7%95%A5)是浏览器的安全基石，由 Netscape 公司在 1995 年引入浏览器，目前所有主流浏览器都遵守这一策略。

简单来说，同源策略是指，在 A 网站设置的 Cookie，在 B 网站无法使用，因为它们**不同源**。

如何区分两个网页是否同源，有三项判别标准：

+ 协议（<font style="color:rgb(37, 41, 51);">protocol</font>）
+ 主机（<font style="color:rgb(37, 41, 51);">host</font>）
+ 端口（port）

只有这三者**全部相同**的两个网页，才被认为是**同源**。否则，为**不同源**。

我们有一个 URL 长这样：[https://jianghushinian.cn/2024/11/18/go-cors/](https://jianghushinian.cn/2024/11/18/go-cors/)。其中，协议为 `https`，主机（或者叫域名）为 `jianghushinian.cn`，端口为 `443`，因为 `443` 是 `https` 协议默认端口号，所以可以省略。

下表列举出了一些 URL 与 [https://jianghushinian.cn/2024/11/18/go-cors/](https://jianghushinian.cn/2024/11/18/go-cors/) 是否属于同源：

| <font style="color:rgb(37, 41, 51);">URL</font> | <font style="color:rgb(37, 41, 51);">是否同源</font> | <font style="color:rgb(37, 41, 51);">原因</font> |
| --- | --- | --- |
| [https://jianghushinian.cn/2024/11/11/sync-once/](https://jianghushinian.cn/2024/11/11/sync-once/) | <font style="color:rgb(37, 41, 51);">是</font> | <font style="color:rgb(37, 41, 51);">协议、域名、端口相同，只有路径不同</font> |
| [http://jianghushinian.cn/2024/11/18/go-cors/](http://jianghushinian.cn/2024/11/18/go-cors/) | <font style="color:rgb(37, 41, 51);">否</font> | <font style="color:rgb(37, 41, 51);">协议不同（HTTPS 和 HTTP）</font> |
| [https://www.jianghushinian.cn/2024/11/18/go-cors/](https://www.jianghushinian.cn/2024/11/18/go-cors/) | <font style="color:rgb(37, 41, 51);">否</font> | <font style="color:rgb(37, 41, 51);">域名不同（jianghushinian.cn 和 www.jianghushinian.cn）</font> |
| [https://www.google.com/](https://www.google.com/) | <font style="color:rgb(37, 41, 51);">否</font> | <font style="color:rgb(37, 41, 51);">域名不同（jianghushinian.cn 和  www.google.com）</font> |
| [https://jianghushinian.cn:8443/2024/11/18/go-cors/](https://jianghushinian.cn:8443/2024/11/18/go-cors/) | <font style="color:rgb(37, 41, 51);">否</font> | <font style="color:rgb(37, 41, 51);">端口不同（443 和 8443）</font> |


现在，我们知道了什么是浏览器的同源策略。那么它和跨域有什么关系呢？**其实，我们常说的跨域，就是不同源。**

在不同源的情况下，除了不能跨域访问 Cookie，浏览器还存在其他限制。

同源策略对跨源交互主要有以下几种限制：

+ **DOM 访问：**一个源中的 JavaScript 脚本无法访问另一个源的页面中的 DOM 元素。
+ **Cookie 和 Storage：**不同源的网页无法访问彼此的 Cookie、LocalStorage、SessionStorage 和 IndexDB。
+ **AJAX 请求：**`XMLHttpRequest` 或 `Fetch` 只能向同源的 URL 发送请求。跨源请求会被阻止，除非服务器启用了 CORS（跨域资源共享）。
+ **`<iframe>` 间访问：**如果一个页面通过 `<iframe>` 嵌套了另一个页面，默认情况下，两个页面的 JavaScript 不能操作对方的 DOM，也无法进行通信。

以上几种交互，跟后端有关的，就是 AJAX 请求了。

> NOTE:
> 其他几种跨源交互如何解决通信问题，属于前端领域，如果你感兴趣，可以参考阮一峰老师的文章[《浏览器同源政策及其规避方法》](https://www.ruanyifeng.com/blog/2016/04/same-origin-policy.html)。

在同源策略的限制下，跨源的 AJAX 请求会被浏览器阻止，导致前端 JavaScript 发出的请求无法得到成功响应。

你一定见过这个令人头大的经典报错：

![跨域](cors.png)

之所以产生这个报错，是因为现在基本上都采用前后端分离架构，前端和后端分开部署。<font style="color:rgb(37, 41, 51);">这就很容易导致前后端域名不一致，会触发浏览器的同源策略限制，导致请求失败。</font>

<font style="color:rgb(37, 41, 51);">有多种方案可以解决跨域问题，如 Nginx 反向代理、JSONP、WebSocket 和 CORS。</font>在编写 Go 程序时，要通过服务端来解决这个问题，就需要引出本文的重点内容 CORS 了。

> NOTE:
>
> 其他几种解决跨域问题的方案不是本文重点，且都比 CORS 概念更加简单，我就不过多介绍了，感兴趣的读者可以自行学习。

### 跨域资源共享（CORS）

CORS 是跨域资源共享（<font style="color:rgb(27, 27, 27);">Cross-Origin Resource Sharing</font>）的缩写，它是 W3C 标准推出用来解决 AJAX 跨域请求的方案。同源策略默认阻止跨域获取资源，但是 CORS 给了 Web 服务器这样的权限，即服务器可以决定是否允许跨源请求访问到它的资源。

CORS 需要浏览器和后端服务器同时支持，所以 CORS 可以理解为一个“协议”。只要通信双方都支持此协议，那么就能够发送跨域请求。所有主流的浏览器都已支持 CORS 功能，并且对用户来说是无感知的，所以通过 CORS 实现跨域，主要工作量在后端。

CORS 标准新增了一组 HTTP 头字段（请求头和响应头），允许服务器端通过 HTTP 响应头声明**哪些源站**通过**浏览器**有权限访问**哪些资源**。而在浏览器侧，CORS 请求分为两类，简单请求和非简单请求。对于那些可能对服务器数据产生**副作用**的 HTTP 请求（非简单请求），浏览器必须先使用 OPTIONS 方法发起一个预检请求（Preflight Request），从而获知服务端是否允许该跨源请求。服务器确认允许之后，浏览器才会发起实际的 HTTP 请求。此外，在预检请求的返回中，服务器端也可以通过 HTTP 头字段告知客户端，是否允许携带 Cookie 等[认证信息](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Authentication)。

上面的描述比较抽象，不是那么好理解。现在我们来具体说明一下在 CORS 中，什么是简单请求，什么是非简单请求。

#### 简单请求

<font style="color:rgb(27, 27, 27);">如果浏览器在发起请求时，请求内容同时</font>满足所有下述条件<font style="color:rgb(27, 27, 27);">，则为简单请求：</font>

+ 请求方法为下列方法之一：
    - HEAD
    - GET
    - POST
+ 请求头不能超出 CORS 规定的安全 HTTP 头字段集合：
    - [Accept](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Accept)
    - [Accept-Language](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Accept-Language)
    - [Content-Language](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Content-Language)
    - [Content-Type](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Content-Type)<font style="color:rgb(27, 27, 27);">（需要注意额外的 MIME 类型限制）</font>
    - [Range](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Range)<font style="color:rgb(27, 27, 27);">（只允许简单的范围标头值，如 `bytes=256-` 或 `bytes=127-255`）
+ Content-Type 请求头所指定的 MIME 类型值仅限于下列三者之一：
    - text/plain
    - multipart/form-data
    - application/x-www-form-urlencoded
+ 如果请求是使用 `XMLHttpRequest` 对象发出的，在返回的 `XMLHttpRequest.upload` 对象属性上没有注册任何事件监听器（也就是说，给定一个 XMLHttpRequest 实例 `xhr`，没有调用 `xhr.upload.addEventListener()`，以监听该上传请求）。
+ 请求中没有使用 `ReadableStream` 对象。

> NOTE:
> 如果你了解前端的 `<form>` 表单，你会发现，允许的 `Content-Type` 跟 `<form>` 表单所支持的 [MIME 类型](https://developer.mozilla.org/zh-CN/docs/Web/HTML/Element/form)可选值如出一辙。这其实就是为了让 AJAX 发送跨域请求时的表现对齐 `<form>` 表单，因为表单是可以直接发送跨域请求的。
>

<font style="color:rgb(27, 27, 27);">知道了什么是简单请求，我们来分析下简单请求基本流程：</font>

![简单请求](simple-request.png)

CORS 简单请求看起来跟普通请求区别不大，只不过浏览器会自动在 HTTP 请求头中加上 `Origin` 字段，用来说明本次请求来自哪个源，`Origin` 字段的值完整包含了判断跨域请求的三项标准：协议、域名、端口号。

一个简单请求的 HTTP 头信息示例如下：

```http
GET /resources/public-data/ HTTP/1.1
Accept: */*
Accept-Encoding: gzip, deflate, br, zstd
Accept-Language: zh-CN,zh;q=0.9,en;q=0.8
Cache-Control: no-cache
Connection: keep-alive
Host: foo.example
Origin: https://foo.example
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/130.0.0.0 Safari/537.36
```

其中请求头 `Origin: https://foo.example` 表明了请求源。

服务端响应的 HTTP 头信息示例如下：

```http
HTTP/1.1 200 OK
Access-Control-Allow-Origin: *
Content-Type: application/json; charset=utf-8
Content-Length: 44
Date: Sun, 17 Nov 2024 02:45:52 GMT
```

<font style="color:rgb(27, 27, 27);">响应头中的 </font>`Access-Control-Allow-Origin: *`<font style="color:rgb(27, 27, 27);"> 表明该资源可以被</font>**任意**跨域的<font style="color:rgb(27, 27, 27);">外源访问。</font>

<font style="color:rgb(27, 27, 27);">所以，使用 </font>`Origin`<font style="color:rgb(27, 27, 27);"> 和 </font>`Access-Control-Allow-Origin`<font style="color:rgb(27, 27, 27);"> 这一对 HTTP 头就能完成最简单的访问控制，突破同源策略的限制。</font>

#### 非简单请求

<font style="color:rgb(27, 27, 27);">与</font>简单请求<font style="color:rgb(27, 27, 27);">不同，非简单请求要求浏览器在发送正式请求之前，必须先使用 </font>OPTIONS<font style="color:rgb(27, 27, 27);"> 方法发起一个预检请求到服务器，以获知服务器是否允许该正式请求。</font>

<font style="color:rgb(27, 27, 27);">非简单请求基本流程如下：</font>

![非简单请求](options-request.png)

上图中，`Preflight request` 就是预检请求，`Main request` 就是正式请求。只有预检请求通过，浏览器才会继续发送正式请求。

一个预检请求的 HTTP 头信息示例如下：

```http
OPTIONS /doc HTTP/1.1
Accept: */*
Accept-Encoding: gzip, deflate, br, zstd
Accept-Language: zh-CN,zh;q=0.9,en;q=0.8
Connection: keep-alive
Host: foo.example
Origin: https://foo.example
Access-Control-Request-Method: POST
Access-Control-Request-Headers: X-PINGOTHER, Content-Type
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/130.0.0.0 Safari/537.36
```

其中请求头 `Origin: https://foo.example` 同简单请求一样，表明了请求源。

此外，还有两个附加的 HTTP 头字段：`Access-Control-Request-Method: POST` 用来告诉服务器，接下来的正式请求将使用 POST 方法。`Access-Control-Request-Headers: X-PINGOTHER, Content-Type` 用来告诉服务器，<font style="color:rgb(27, 27, 27);">正式请求将携带两个自定义请求头字段 </font>`X-PINGOTHER`<font style="color:rgb(27, 27, 27);"> 和 </font>`Content-Type`<font style="color:rgb(27, 27, 27);">。</font>

如果你细心观察，还会发现预检请求的 URL 和正式请求的 URL 其实是相同的。

预检请求的服务端响应的 HTTP 头信息示例如下：

```http
HTTP/1.1 204 No Content
Access-Control-Allow-Origin: https://foo.example
Access-Control-Allow-Credentials: true
Access-Control-Allow-Headers: X-PINGOTHER,Content-Type
Access-Control-Allow-Methods: POST,GET,OPTIONS
Vary: Origin
Date: Sun, 17 Nov 2024 03:42:16 GMT
```

<font style="color:rgb(27, 27, 27);">响应头中的 </font>`Access-Control-Allow-Origin: https://foo.example`<font style="color:rgb(27, 27, 27);"> 表明该资源可以被</font> `https://foo.example` 这个源<font style="color:rgb(27, 27, 27);">访问。</font>`Access-Control-Allow-Credentials: true`<font style="color:rgb(27, 27, 27);"> 表示允许浏览器携带 Cookie 等认证信息。当允许携带 Cookie 时，</font>`Access-Control-Allow-Origin`<font style="color:rgb(27, 27, 27);"> 值不能为通配符 </font>`*`<font style="color:rgb(27, 27, 27);">，必须指定明确的域名。</font>

剩下两个 CORS 相关的 HTTP 头字段很好理解，`Access-Control-Allow-Headers: X-PINGOTHER,Content-Type` 表明了服务端允许的附加**请求头**。`Access-Control-Allow-Methods: POST,GET,OPTIONS` 表明了服务端允许的**请求方法**。

预检请求完成后，浏览器将发送正式请求：

```http
POST /doc HTTP/1.1
Accept: */*
Accept-Encoding: gzip, deflate, br, zstd
Accept-Language: zh-CN,zh;q=0.9,en;q=0.8
Cache-Control: no-cache
Connection: keep-alive
Content-Length: 15
Content-Type: application/json
Host: foo.example
Origin: https://foo.example
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/130.0.0.0 Safari/537.36
```

服务端响应的 HTTP 头信息示例如下：

```http
HTTP/1.1 200 OK
Access-Control-Allow-Origin: https://foo.example
Access-Control-Allow-Credentials: true
Content-Type: application/json; charset=utf-8
Vary: Origin
Date: Sun, 17 Nov 2024 03:42:16 GMT
Content-Length: 68
```

至此，一个非简单请求的 CORS 流程就完成了。

#### CORS 规范定义的 HTTP 头字段

我们对 CORS 中的简单请求和非简单请求有了较为清晰的认识。现在，我们再来梳理下 CORS 规范中定义的所有 HTTP 头字段。

请求头字段如下：

+ [Origin](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Origin)<font style="color:rgb(27, 27, 27);">：表明获取资源的请求是从哪个源发起的。此请求头与服务器端的 </font>[Access-Control-Allow-Origin](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Access-Control-Allow-Origin)<font style="color:rgb(27, 27, 27);"> 响应头对应。</font>

<font style="color:rgb(27, 27, 27);">预检请求头字段如下：</font>

+ [Access-Control-Request-Headers](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Access-Control-Request-Headers)<font style="color:rgb(27, 27, 27);">：用于发起预检请求，告知服务器正式请求会携带哪些 HTTP 请求头（通过 </font>[setRequestHeader()](https://developer.mozilla.org/zh-CN/docs/Web/API/XMLHttpRequest/setRequestHeader)<font style="color:rgb(27, 27, 27);"> 等设置的）。此请求头与服务器端的 </font>[Access-Control-Allow-Headers](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Access-Control-Allow-Headers)<font style="color:rgb(27, 27, 27);"> 响应头对应。</font>
+ [Access-Control-Request-Method](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Access-Control-Request-Method)<font style="color:rgb(27, 27, 27);">：用于发起预检请求，告知服务器正式请求会使用哪一种 </font><font style="color:rgb(27, 27, 27);">HTTP 请求方法</font><font style="color:rgb(27, 27, 27);">。此请求头与服务器端的 </font>[Access-Control-Allow-Methods](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Access-Control-Allow-Methods)<font style="color:rgb(27, 27, 27);"> 响应头对应。</font>

响应头字段如下：

+ [Access-Control-Allow-Origin](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Access-Control-Allow-Origin)<font style="color:rgb(27, 27, 27);">：必须字段，</font>告知浏览器允许该源访问资源，可以是单一的源，也可以是通配符 `*`。如果浏览器需要携带认证信息，则不能使用 `*`。<font style="color:rgb(27, 27, 27);">此响应头是服务器端对浏览器端 </font>[Origin](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Origin)<font style="color:rgb(27, 27, 27);"> 请求头的响应。</font>
+ [Access-Control-Allow-Headers](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Access-Control-Allow-Headers)<font style="color:rgb(27, 27, 27);">：可选字段，用于</font>预检请求<font style="color:rgb(27, 27, 27);">的响应，指明了正式请求中允许浏览器携带的 HTTP 请求头字段。如果浏览器需要携带认证信息，则不能使用 </font>`*`<font style="color:rgb(27, 27, 27);">。此响应头是服务器端对浏览器端 </font>[Access-Control-Request-Headers](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Access-Control-Request-Headers)<font style="color:rgb(27, 27, 27);"> 请求头的响应。</font>
+ [Access-Control-Allow-Methods](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Access-Control-Allow-Methods)<font style="color:rgb(27, 27, 27);">：可选字段，用于</font>预检请求<font style="color:rgb(27, 27, 27);">的响应，指明了正式请求中允许的 HTTP 请求方法。如果浏览器需要携带认证信息，则不能使用 </font>`*`<font style="color:rgb(27, 27, 27);">。此响应头是服务器端对浏览器端 </font>[Access-Control-Request-Method](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Access-Control-Request-Method)<font style="color:rgb(27, 27, 27);"> 请求头的响应。</font>
+ [Access-Control-Expose-Headers](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Access-Control-Expose-Headers)<font style="color:rgb(27, 27, 27);">：可选字段，列出允许的响应头字段，供浏览器的 JavaScript 代码（如 </font>[getResponseHeader()](https://developer.mozilla.org/zh-CN/docs/Web/API/XMLHttpRequest/getResponseHeader)<font style="color:rgb(27, 27, 27);">）获取。浏览器发起 </font>CORS 请求时，`XMLHttpRequest`对象的`getResponseHeader()`方法，或 `fetch` 响应对象 `Response` 的 `headers.get` 方法，只能拿到 6 个基本的响应头字段：`Cache-Control`、`Content-Language`、`Content-Type`、`Expires`、`Last-Modified`、`Pragma`。如果想访问其他响应头字段，就需要在 `Access-Control-Expose-Headers`<font style="color:rgb(27, 27, 27);"> </font>中指定。
+ [Access-Control-Allow-Credentials](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Access-Control-Allow-Credentials)<font style="color:rgb(27, 27, 27);">：可选字段，该字段指定了当浏览器的 </font>`credentials`<font style="color:rgb(27, 27, 27);"> 设置为 </font>`true`<font style="color:rgb(27, 27, 27);"> 时，是否允许浏览器 JavaScript 脚本读取响应体 </font>`response`<font style="color:rgb(27, 27, 27);"> 中的内容。同时，它为 </font>`true`<font style="color:rgb(27, 27, 27);">，表明允许浏览器携带 Cookie。</font>
+ [Access-Control-Max-Age](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Access-Control-Max-Age)<font style="color:rgb(27, 27, 27);">：可选字段，指定了预检请求的结果能够被缓存多久。</font>

<font style="color:rgb(27, 27, 27);">其他相关头字段如下：</font>

+ [Timing-Allow-Origin](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Timing-Allow-Origin)：主要用于控制 Web 应用程序是否能够访问资源的性能信息，例如通过 [ResourceTimingAPI](https://developer.mozilla.org/en-US/docs/Web/API/Resource_Timing_API) 获取有关加载资源的详细时间数据。
+ [Vary](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Vary)：服务器声明缓存策略，可以根据 `Origin` 头动态生成响应。

<font style="color:rgb(27, 27, 27);">以上就是 CORS 规范中支持的全部 HTTP 头字段了。</font>

<font style="color:rgb(27, 27, 27);">注意，CORS 规范所定义的请求头，都无需手动设置，当使用 AJAX 发起跨域请求时，浏览器会自行设置。所以我在前文说 CORS 请求是对前端用户无感知的。</font>

### 在 Go 中解决跨域问题

学完了 CORS 的概念，终于到实践环节了。

接下来我就以两个例子，分别演示下 CORS 中的简单请求和非简单请求，以及在 Go 程序中如何解决跨域问题。

#### 简单请求

这是一个简单请求的前端代码实现：

> https://github.com/jianghushinian/blog-go-example/blob/main/cors/index.html

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>CORS Demo</title>
</head>
<body>
<h1>CORS 简单请求演示</h1>
<button id="fetchData">发送跨域请求</button>
<pre id="output"></pre>

<script>
    document.getElementById('fetchData').addEventListener('click', () => {
        fetch('http://localhost:8000/data') // 不同源（端口不同）
            .then(response => {
                if (!response.ok) {
                    throw new Error(`HTTP error! Status: ${response.status}`);
                }
                return response.json();
            })
            .then(data => {
                document.getElementById('output').textContent = JSON.stringify(data, null, 2);
            })
            .catch(error => {
                document.getElementById('output').textContent = `Error: ${error.message}`;
            });
    });
</script>
</body>
</html>
```

如果你不熟悉前端也没关系，你只需要知道，这个代码会在页面中展示一个按钮，点击按钮后，浏览器会通过 AJAX 发送一个 GET 请求到地址 `http://localhost:8000/data`。

如下示例代码是使用 Gin 框架开启的后端服务器：

> https://github.com/jianghushinian/blog-go-example/blob/main/cors/main.go

```go
package main

import (
	"github.com/gin-gonic/gin"
)

func main() {
	r := gin.Default()

	// 路由处理：返回简单的 JSON 数据
	r.GET("/data", func(c *gin.Context) {
		c.JSON(200, gin.H{
			"message": "这是未配置 CORS 的响应",
		})
	})

	// 监听在 8000 端口
	r.Run(":8000")
}
```

代码很简单，无需解释，我们直接启动它：

```bash
$ go run main.go
```

通过 GoLand IDE 打开前端网页，点击 `发送跨域请求` 按钮：

![简单请求跨域](cors-simple.png)

我们得到了经典的跨域请求报错。

这是因为前端和后端**不同源**，前端网页的 URL 根路径是 `http://localhost:63342`，后端服务的 URL 根路径是 `http://localhost:8000`，显然二者端口不一样，所以触发了浏览器跨域限制。

现在我们重新编写一版服务端代码的实现：

> https://github.com/jianghushinian/blog-go-example/blob/main/cors/cors/main.go

```go
package main

import "github.com/gin-gonic/gin"

// NOTE: 在 Handler Func 中处理简单请求
func main() {
	r := gin.Default()

	// 路由处理：返回简单的 JSON 数据
	r.GET("/data", func(c *gin.Context) {
		// 设置 CORS 响应头
		c.Header("Access-Control-Allow-Origin", "*")

		c.JSON(200, gin.H{
			"message": "这是配置了 CORS 的响应",
		})
	})

	// 监听在 8000 端口
	r.Run(":8000")
}
```

这里重点代码仅有一行 `c.Header("Access-Control-Allow-Origin", "*")`，不必过多解释，我想你已经明白了这意味着什么。

启动示例代码：

```bash
$ go run cors/main.go
```

我们刷新下页面，重新点击 `发送跨域请求` 按钮：

![CORS 简单请求](cors-simple-success.png)

这一次，得到了成功响应。

并且从截图中可以看到，服务端设置允许跨域的响应头信息 `Access-Control-Allow-Origin: *`。

简单请求的 CORS 支持就是这么简单，实际上在服务端仅需要修改一行 Go 代码。

#### 非简单请求

我们再来看下非简单请求的实践示例。

非简单请求的前端代码实现：

> https://github.com/jianghushinian/blog-go-example/blob/main/cors/index-options.html

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>CORS Non-Simple Request Demo</title>
</head>
<body>
<h1>CORS 非简单请求演示</h1>
<button id="fetchData">发送跨域请求</button>
<pre id="output"></pre>

<script>
    document.getElementById('fetchData').addEventListener('click', () => {
        fetch('http://localhost:8000/data', {
            method: 'POST', // 非简单请求，使用 POST
            headers: {
                'Content-Type': 'application/json', // 非简单请求的 Content-Type
                'X-Custom-Header': 'CustomValue'    // 自定义请求头
            },
            body: JSON.stringify({name: '江湖十年'})  // 带有请求体
        })
            .then(response => {
                if (!response.ok) {
                    throw new Error(`HTTP error! Status: ${response.status}`);
                }
                // console.log(`X-jwt-token: ${response.headers.get("X-jwt-token")}`)
                return response.json();
            })
            .then(data => {
                document.getElementById('output').textContent = JSON.stringify(data, null, 2);
            })
            .catch(error => {
                document.getElementById('output').textContent = `Error: ${error.message}`;
            });
    });
</script>
</body>
</html>
```

这里同样在页面中展示一个按钮，不过现在点击按钮后，浏览器会通过 AJAX 发送一个 POST 请求到地址 `http://localhost:8000/data`。并且，`Content-Type` 所指定的 MIME 类型值也不是简单请求所支持的类型。所以这是一个非简单请求。此外，还设置了自定义请求头 `X-Custom-Header`，以及请求体 `{name: '江湖十年'}`。

我们前面实现的 Go 服务端程序仅支持简单请求，所以也需要修改。

现在我们重新编写一版支持预检请求的服务端代码的实现：

> https://github.com/jianghushinian/blog-go-example/blob/main/cors/cors/main.go

```go
package main

import "github.com/gin-gonic/gin"

// NOTE: 使用 CORS 中间件处理非简单请求
func main() {
	r := gin.Default()

	// 使用 CORS 中间件
	r.Use(Cors)

	// 路由处理：返回简单的 JSON 数据
	r.POST("/data", func(c *gin.Context) {
		var requestData map[string]interface{}
		if err := c.BindJSON(&requestData); err != nil {
			c.JSON(400, gin.H{"error": "Invalid JSON"})
			return
		}
		c.JSON(200, gin.H{
			"message":     "非简单请求已成功",
			"requestData": requestData,
		})
	})

	// 监听在 8000 端口
	r.Run(":8000")
}

func Cors(c *gin.Context) {
	// 如果没有 Origin 请求头则说明不是 CORS 请求
	if c.Request.Header.Get("Origin") == "" {
		return
	}

	// 允许的 CORS 请求源
	c.Header("Access-Control-Allow-Origin", "*")

	// 处理预请求
	if c.Request.Method == "OPTIONS" {
		c.Header("Access-Control-Allow-Headers", "Content-Type,X-Custom-Header")
		c.Header("Access-Control-Allow-Methods", "GET,POST,PUT,PATCH,DELETE,OPTIONS")
		c.Header("Content-Type", "application/json")
		c.AbortWithStatus(204)
	}

	// 处理正式请求
	c.Next()
}
```

这里编写了一个 `Cors` 中间件，来支持服务端程序处理 CORS 请求。我在前文提到，预检请求的 URL 与正式请求的 URL 一致，所以预检请求的 URL 理论上可以为任意路径，所以非常适合在中间件中实现。

代码中写了注释，所以很容易看懂。首先根据是否存在 `Origin` 请求头来判断当前请求是否为 CORS 请求，如果是则继续处理。接着设置了允许任意跨域的<font style="color:rgb(27, 27, 27);">外源访问。然后对预请求进行处理，</font>我们可以发现，当 `Access-Control-Allow-Headers` 和 `Access-Control-Allow-Methods` 支持多个时，可以使用逗号分隔的列表来表示，这在 HTTP 头字段中算是惯用法了。最后预检请求通过，使用 `c.AbortWithStatus(204)` 设置响应状态码为 `204`，因为响应中没有响应体，所以相比于 `200` 响应码，使用 `204` 响应码更为合适。

启动服务端示例代码：

```bash
$ go run cors/main.go
```

打开非简单请求演示页面，点击 `发送跨域请求` 按钮：

![CORS 非简单请求](cors-options.png)

可以看到浏览器为我们发送了两个请求，第一个请求类型为 `preflight`，即预检请求。预检请求成功后，发送第二个请求类型为 `fetch`，因为我们的 AJAX 正式请求是使用 `fetch` 库来实现的。

预检请求的请求头如下：

```http
OPTIONS /data HTTP/1.1
Accept: */*
Accept-Encoding: gzip, deflate, br, zstd
Accept-Language: zh-CN,zh;q=0.9,en;q=0.8
Access-Control-Request-Headers: content-type,x-custom-header
Access-Control-Request-Method: POST
Cache-Control: no-cache
Connection: keep-alive
Host: localhost:8000
Origin: http://localhost:63342
Pragma: no-cache
Referer: http://localhost:63342/
Sec-Fetch-Dest: empty
Sec-Fetch-Mode: cors
Sec-Fetch-Site: same-site
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/130.0.0.0 Safari/537.36
```

响应头如下：

```http
HTTP/1.1 204 No Content
Access-Control-Allow-Headers: Content-Type,X-Custom-Header
Access-Control-Allow-Methods: GET,POST,PUT,PATCH,DELETE,OPTIONS
Access-Control-Allow-Origin: *
Content-Type: application/json
Date: Mon, 18 Nov 2024 00:23:00 GMT
```

现在我们就掌握了在 Go 中如何解决 CORS 跨域问题。

接下来你可能想要优化下我们实现的 `Cors` 中间件，不过且慢，其实在 Gin 框架中我们有现成的工具可以用。

#### Gin 框架中如何解决跨域问题

在 Gin 框架生态的 [gin-contrib](https://github.com/gin-contrib) 列表中有一个 [cors](https://github.com/gin-contrib/cors) 库就是专门为 Gin 提供的接口跨域请求的 CORS 中间件。

使用示例如下：

> https://github.com/jianghushinian/blog-go-example/blob/main/cors/cors/main.go

```go
package main

import (
	"strings"
	"time"

	"github.com/gin-contrib/cors"
	"github.com/gin-gonic/gin"
)

// NOTE: 使用 Gin 的 CORS 中间件处理非简单请求
func main() {
	r := gin.Default()

	// 使用 CORS 中间件配置非简单请求
	r.Use(cors.New(cors.Config{
		// AllowOrigins: []string{"http://localhost:63342"},          // 允许的前端域名
		AllowMethods:     []string{"GET", "POST", "OPTIONS"},          // 允许的 HTTP 方法
		AllowHeaders:     []string{"Content-Type", "X-Custom-Header"}, // 允许的请求头
		ExposeHeaders:    []string{"X-jwt-token"},                     // 允许暴露给 JavaScript 脚本的响应头
		AllowCredentials: true,                                        // 是否允许凭证（Cookies）
		AllowOriginFunc: func(origin string) bool { // 使用函数设置允许的前端域名
			if strings.HasPrefix(origin, "http://localhost") {
				return true
			}
			return strings.HasSuffix(origin, "jianghushinian.cn")
		},
		MaxAge: 1 * time.Hour,
	}))

	// 路由处理：返回简单的 JSON 数据
	r.POST("/data", func(c *gin.Context) {
		var requestData map[string]interface{}
		if err := c.BindJSON(&requestData); err != nil {
			c.JSON(400, gin.H{"error": "Invalid JSON"})
			return
		}
		c.Header("X-jwt-token", "fake-token")
		c.JSON(200, gin.H{
			"message":     "非简单请求已成功",
			"requestData": requestData,
		})
	})

	// 监听在 8000 端口
	r.Run(":8000")
}
```

这个中间件最方便也是我最喜欢的地方，就是它在设置 `Access-Control-Allow-Origin` 响应头时， 支持使用函数 `AllowOriginFunc`。这样，就允许我们在函数中编写逻辑，很容易就能写出同时适配开发和生产环境的 CORS 逻辑。而不必在 `AllowOrigins` 列表中列出全部可能出现的情况。所有支持设置多个值的响应头，都以 `[]string` 类型来实现。用起来还是非常简单方便的。

这个示例与我们实现的 `Cors` 中间件本质上原理是相同的，我就不跑起来演示了，你可以自行尝试。

#### K8s 如何解决跨域问题

最后我们再来看下 K8s 源码中是如何解决 CORS 跨域问题的。

相关部分源码如下：

> [https://github.com/kubernetes/kubernetes/blob/v1.31.0/staging/src/k8s.io/apiserver/pkg/server/filters/cors.go#L36](https://github.com/kubernetes/kubernetes/blob/v1.31.0/staging/src/k8s.io/apiserver/pkg/server/filters/cors.go#L36)

```go
func WithCORS(handler http.Handler, allowedOriginPatterns []string, allowedMethods []string, allowedHeaders []string, exposedHeaders []string, allowCredentials string) http.Handler {
	if len(allowedOriginPatterns) == 0 {
		return handler
	}
	allowedOriginPatternsREs := allowedOriginRegexps(allowedOriginPatterns)

	// Set defaults for methods and headers if nothing was passed
	if allowedMethods == nil {
		allowedMethods = []string{"POST", "GET", "OPTIONS", "PUT", "DELETE", "PATCH"}
	}
	allowMethodsResponseHeader := strings.Join(allowedMethods, ", ")

	if allowedHeaders == nil {
		allowedHeaders = []string{"Content-Type", "Content-Length", "Accept-Encoding", "X-CSRF-Token", "Authorization", "X-Requested-With", "If-Modified-Since"}
	}
	allowHeadersResponseHeader := strings.Join(allowedHeaders, ", ")

	if exposedHeaders == nil {
		exposedHeaders = []string{"Date"}
	}
	exposeHeadersResponseHeader := strings.Join(exposedHeaders, ", ")

	return http.HandlerFunc(func(w http.ResponseWriter, req *http.Request) {
		origin := req.Header.Get("Origin")
		if origin == "" {
			handler.ServeHTTP(w, req)
			return
		}
		if !isOriginAllowed(origin, allowedOriginPatternsREs) {
			handler.ServeHTTP(w, req)
			return
		}

		w.Header().Set("Access-Control-Allow-Origin", origin)
		w.Header().Set("Access-Control-Allow-Methods", allowMethodsResponseHeader)
		w.Header().Set("Access-Control-Allow-Headers", allowHeadersResponseHeader)
		w.Header().Set("Access-Control-Expose-Headers", exposeHeadersResponseHeader)
		w.Header().Set("Access-Control-Allow-Credentials", allowCredentials)

		// Stop here if its a preflight OPTIONS request
		if req.Method == "OPTIONS" {
			w.WriteHeader(http.StatusNoContent)
			return
		}

		// Dispatch to the next handler
		handler.ServeHTTP(w, req)
	})
}

func isOriginAllowed(originHeader string, allowedOriginPatternsREs []*regexp.Regexp) bool {
	for _, re := range allowedOriginPatternsREs {
		if re.MatchString(originHeader) {
			return true
		}
	}
	return false
}

func allowedOriginRegexps(allowedOrigins []string) []*regexp.Regexp {
	res, err := compileRegexps(allowedOrigins)
	if err != nil {
		klog.Fatalf("Invalid CORS allowed origin, --cors-allowed-origins flag was set to %v - %v", strings.Join(allowedOrigins, ","), err)
	}
	return res
}

func compileRegexps(regexpStrings []string) ([]*regexp.Regexp, error) {
	regexps := []*regexp.Regexp{}
	for _, regexpStr := range regexpStrings {
		r, err := regexp.Compile(regexpStr)
		if err != nil {
			return []*regexp.Regexp{}, err
		}
		regexps = append(regexps, r)
	}
	return regexps, nil
}
```

可以看到，`WithCORS` 函数的第一个参数是 `http.Handler`，即控制器函数。后面几个参数都是用来设置 CORS 响应头的，不过也能发现它不支持 `Access-Control-Max-Age` 响应头的设置。

值得注意的一点是，`WithCORS` 的实现中 `allowedOriginPatterns` 是支持正则匹配的。这跟 Gin 框架中 `cors` 库支持函数匹配目的一样，都是为了方便我们处理 `Access-Control-Allow-Origin` 响应头。所以，CORS 中哪个响应头最重要，也就不言自明了。

### 总结

我们从浏览器的同源策略开始，讲解了跨域问题的起因，以及如何使用 CORS 来解决跨域问题。

并且，在 Go 中进行了代码演示。我还讲解了在 Gin 框架和 K8s 中是如何解决 CORS 问题的。如果你使用其他 Web 框架，应该很容易就能找到对应的解决方案。如果没有，我们已经掌握了 CORS 原理，那自己实现一个也很简单，并且我们也可以参考 Gin `cors` 中间件或  K8s `WithCORS` 的实现。

记住，跨域是浏览器的同源策略导致的。也就是说，只有在浏览器中才会出现跨域问题。如果你使用 `curl`、`Postman` 等，或者很多编程语言的请求库，如 Python 的 `urllib`、`requests`、Go 的 `net/http` 等，根本就不会有跨域问题。

这也就是为什么使用 Nginx 反向代理能解决跨域问题。还有两个本文提及但没有深入讲解的 CORS 解决方案：`JSONP` 是一个较为“古老”的技术了，现在几乎见不到了，我工作多年也仅在刚入行的时候用过一次，并且它只能发送 GET 请求，已经不推荐使用了。另外，使用 WebSocket 协议时无需考虑浏览器跨域问题，WebSocket 协议不实行同源策略。

针对后端的 CORS 解放方案，本文从原理到实践都讲解清楚了，希望下一次再遇到跨域问题，我们不要只会复制粘贴解决问题，遇到复杂情况，也会知道如何排查和处理。

本文示例源码我都放在了 [GitHub](https://github.com/jianghushinian/blog-go-example/tree/main/cors) 中，欢迎点击查看。

希望此文能对你有所启发。

**延伸阅读**

+ 同源策略：[https://zh.wikipedia.org/wiki/同源策略](https://zh.wikipedia.org/wiki/同源策略)
+ 浏览器的同源策略：[https://developer.mozilla.org/zh-CN/docs/Web/Security/Same-origin_policy](https://developer.mozilla.org/zh-CN/docs/Web/Security/Same-origin_policy)
+ 跨源资源共享：[https://developer.mozilla.org/zh-CN/docs/Glossary/CORS](https://developer.mozilla.org/zh-CN/docs/Glossary/CORS)
+ 跨源资源共享（CORS）：https://developer.mozilla.org/zh-CN/docs/Web/HTTP/CORS
+ 列入 CORS 白名单的响应标头：https://developer.mozilla.org/zh-CN/docs/Glossary/CORS-safelisted_response_header
+ 浏览器同源政策及其规避方法：[https://www.ruanyifeng.com/blog/2016/04/same-origin-policy.html](https://www.ruanyifeng.com/blog/2016/04/same-origin-policy.html)
+ 跨域资源共享 CORS 详解：[https://www.ruanyifeng.com/blog/2016/04/cors.html](https://www.ruanyifeng.com/blog/2016/04/cors.html)
+ CORS gin's middleware：https://github.com/gin-contrib/cors
+ K8s WithCORS：https://github.com/kubernetes/kubernetes/blob/v1.31.0/staging/src/k8s.io/apiserver/pkg/server/filters/cors.go#L36
+ 本文 GitHub 示例代码：[https://github.com/jianghushinian/blog-go-example/tree/main/cors](https://github.com/jianghushinian/blog-go-example/tree/main/cors)

**联系我**

+ 公众号：[Go编程世界](https://jianghushinian.cn/about)
+ 微信：jianghushinian
+ 邮箱：[jianghushinian007@outlook.com](mailto:jianghushinian007@outlook.com)
+ 博客：[https://jianghushinian.cn](https://jianghushinian.cn)
