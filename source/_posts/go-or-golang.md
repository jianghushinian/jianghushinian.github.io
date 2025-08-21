---
title: 'Go 还是 Golang？可能你一直都搞错了！'
date: 2025-08-21 08:48:24
tags:
- Go
categories:
- Go
---

大家好，我是江湖十年。

今天来聊一个对于广大 Gopher 来说既熟悉又陌生的话题，Go 语言的名字到底是叫 Go 还是叫 Golang？这是一个很容易被忽视或者不被开发者所重视的问题，也许你从未考虑过这个问题，但我认为这其实是一个比较严肃的话题。

我们先从 Go 语言的诞生开始讲起。

<!-- more -->

### Go 语言的诞生

2007年 9 月 20 日，一个周四的下午，Google 程序员 [Rob Pike](https://en.wikipedia.org/wiki/Rob_Pike) 由于无法忍受 C++ 的缓慢编译过程，便与 [Robert Griesemer](https://en.wikipedia.org/wiki/Robert_Griesemer) 和 [Ken Thompson](https://en.wikipedia.org/wiki/Ken_Thompson) 开始讨论设计一门新的编程语言。你可以在维基百科中浏览这三位顶级程序员大佬的事迹，而这次讨论便是 Go 语言诞生的起点。

在这次沟通完的第二天，这三位大佬又聚在一起进行了新一轮的讨论，第三天 Robert Griesemer 便发出一封主题为 “prog lang discussion” 的邮件，记录了他们三位的讨论结果。自此，Rob Pike 想要设计一门新的编程语言的宏图开始有了可追溯的源头。

在之后的日子里，随着 Go 语言的流行，Go 社区中也广泛流传着这张经典图片：

![go author](go-author.png)

以上三位大佬就是 Go语言之父，从左到右这三位作者分别是 Robert Griesemer、Rob Pike 和 Ken Thompson。

### Go 名字由来

在 Go 语言发布的十周年纪念日，Rob Pike 在自己的博客中发表了「[Go: Ten years and climbing](https://commandcenter.blogspot.com/2017/09/go-ten-years-and-climbing.html)」一文，来纪念这个日子。

在此文中，Rob Pike 提到，2007 年 9 月 25 日，他在第一封有关语言设计的邮件中写道：

```plain
  Subject: Re: prog lang discussion 
  From: Rob 'Commander' Pike 
  Date: Tue, Sep 25, 2007 at 3:12 PM
  To: Robert Griesemer, Ken Thompson
  
  i had a couple of thoughts on the drive home.
  
  1. name
    
  'go'. you can invent reasons for this name but it has nice properties.
  it's short, easy to type. tools: goc, gol, goa. if there's an interactive
  debugger/interpreter it could just be called 'go'. the suffix is .go
  ...
```

这封邮件正是用来回复主题为 “prog lang discussion” 的邮件。

根据这封邮件可知，`go` 这个名字由此诞生。之所以起这个名字，并没有什么特殊的含义，仅仅是 Rob Pike 认为 `go` 足够简洁，很容易打出来，如果以后有其他相关工具，名字可以叫 `goc`、`gol`、`goa`，简洁优雅。

这便是 `go` 这个名字的由来。

### Golang 的由来

Go 项目在 2009 年 11 月 10 日被正式开源，项目代码被托管在 [code.google.com](https://code.google.com/) 网站中，而 Go 语言最初的官方网址为 [golang.com](http://golang.com/) 。

因为官方网址中有 `golang` 字样，这也是导致 `golang` 这个叫法流行起来的最直接原因。而之所以 Go 没有使用 [go.com](http://go.com/) 作为官网，是因为这个域名在迪斯尼那里，并不在 Google 手里。

而现在你访问 [golang.com](http://golang.com/) 域名，则会被重定向到 [go.dev](https://go.dev/)，这是 Go 语言的最新官网。那为什么 Go 不一开始就启用这个域名呢？因为在 Go 发布时，还没有 `.dev` 域名 :）。

所以，Golang 这个叫法能够流行起来，全拜 Go 官网所赐。但其实 Go 官方从来没有将这门编程语言叫做 `golang`，在 Go 官方口径中，这门语言自始至终都叫 `go`。

甚至 Go 官方在 [FAQ](https://go.dev/doc/faq) 中也针对 [Is the language called Go or Golang?](https://go.dev/doc/faq#go_or_golang) 这个问题进行了解答：

```plain
The language is called Go. The “golang” moniker arose because the web site was originally golang.org. (There was no .dev domain then.) Many use the golang name, though, and it is handy as a label. For instance, the social media tag for the language is “#golang”. The language’s name is just plain Go, regardless.

A side note: Although the official logo has two capital letters, the language name is written Go, not GO.
```

这门语言叫做 Go。之所以有“golang”这个称呼，是因为最初的网站是 golang.org。（当时还没有.dev 域名。）尽管很多人使用 golang 这个名字，但它作为标签非常方便。例如，这门语言的社交媒体标签就是“#golang”。无论怎样，这门语言的名字就是 Go。

此外，这里还强调了，尽管 Go 语言的官方  [logo](https://go.dev/blog/go-brand) 有两个大写字母，但这门语言名称仍然写作 `Go`，而不是 `GO`。

![go logo](go-logo.png)

### 维基百科也会出错？

如果你去查看[维基百科](https://zh.wikipedia.org/wiki/Go)中对 Go 语言的定义，你会发现维基百科竟然也将 Golang 作为 Go 语言的合法称呼。

![go wiki](go-wiki.png)

这里引用 `[3]` 的链接 [https://techcrunch.com/2009/11/10/google-go-language/](https://techcrunch.com/2009/11/10/google-go-language/) 我点击进去看了下，文中从来没有提到过 `golang` 这一说法，仅贴了 Go 语言当时的官网地址 [golang.com](http://golang.com/) 。

这就比较有意思了，Go 官方强调这门语言叫做 `go`，不叫 `golang`，但显然 Golang 已经深入人心，互联网的记忆无法被抹除，看来这一局面自始至终都无法改变。

### 总结

现在，我们知道了，Go 语言的正确写法是 `Go/go`，而 `GO/Golang/golang` 都是错误的写法。

一个有趣的现象，在主流的招聘平台中，你搜索 Go 语言，出来的岗位大多写着的都是 Golang 开发，而非 Go 开发。不过，好在我查阅了很多 Go 语言相关的计算机书籍，发现它们都统一使用了 Go 这个名字来作为书籍的名称，从这点开来出版社确实更加严谨。

此外，如果你注意过鼎鼎大名的主流编程语言排行榜 [TIOBE](https://www.tiobe.com/tiobe-index/)，可以发现，榜单中只有 Erlang(如果它算主流编程语言的话 :)) 的名字带有 `lang` 后缀，其他编程语言都没有这个后缀，这是一个不起眼的小细节。

但是，这篇文章并不是要纠正你什么，甚至我自己的公众号「Go编程世界」的简介里，也有 Golang 字样。相比于 Go，Golang 这个写法其实对 SEO 更加友好，当大家都使用错误的叫法，那么它最终可能就会变成正确答案😄。

本文讲解了一个对广大 Gopher 来说既无用有又用的知识，知道 Go 官方叫什么，对我们写 Go 代码或者与人交流，并不能改变什么，所以这是一个无用的知识；但是，如果你对编程有着敬畏之心，我想这是一个严肃的问题，这代表了作为一名 Gopher 的专业性。当别人问起你所从事领域的一个不起眼的小问题，你能不含糊其辞，而是非常准确的回答上来，这就叫做专业。

希望此文能对你有所启发。

**延伸阅读**

+ Go 官网：[http://golang.com/](http://golang.com/)
+ Go 官网：[https://go.dev/](https://go.dev/)
+ Rob Pike Wiki：[https://en.wikipedia.org/wiki/Rob_Pike](https://en.wikipedia.org/wiki/Rob_Pike)
+ Robert Griesemer：[https://en.wikipedia.org/wiki/Robert_Griesemer](https://en.wikipedia.org/wiki/Robert_Griesemer)
+ Ken Thompson Wiki：[https://en.wikipedia.org/wiki/Ken_Thompson](https://en.wikipedia.org/wiki/Ken_Thompson)
+ Go: Ten years and climbing：[https://commandcenter.blogspot.com/2017/09/go-ten-years-and-climbing.html](https://commandcenter.blogspot.com/2017/09/go-ten-years-and-climbing.html)
+ Go FAQ Is the language called Go or Golang?：[https://go.dev/doc/faq#go_or_golang](https://go.dev/doc/faq#go_or_golang)
+ Go's New Brand：[https://go.dev/blog/go-brand](https://go.dev/blog/go-brand)
+ Go Wiki：[https://zh.wikipedia.org/wiki/Go](https://zh.wikipedia.org/wiki/Go)
+ Google's Go: A New Programming Language That's Python Meets C++：[https://techcrunch.com/2009/11/10/google-go-language/](https://techcrunch.com/2009/11/10/google-go-language/)
+ TIOBE：[https://www.tiobe.com/tiobe-index/](https://www.tiobe.com/tiobe-index/)
+ 本文永久地址：[https://jianghushinian.cn/2025/08/21/go-or-golang](https://jianghushinian.cn/2025/08/21/go-or-golang)

**联系我**

+ 公众号：[Go编程世界](https://jianghushinian.cn/about)
+ 微信：jianghushinian
+ 邮箱：[jianghushinian007@outlook.com](mailto:jianghushinian007@outlook.com)
+ 博客：[https://jianghushinian.cn](https://jianghushinian.cn)
+ GitHub：[https://github.com/jianghushinian](https://github.com/jianghushinian)
