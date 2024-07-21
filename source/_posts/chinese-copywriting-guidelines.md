---
title: '中文文案排版指北'
date: 2024-07-20 22:50:45
tags:
- 写作
- 开发规范
categories:
- 规范
---

> 「有研究显示，打字的时候不喜欢在中文和英文之间加空格的人，感情路都走得很辛苦，有七成的比例会在 34 岁的时候跟自己不爱的人结婚，而其余三成的人最后只能把遗产留给自己的猫。毕竟爱情跟书写都需要适时地留白。
>
> 与大家共勉之。」——[vinta/paranoid-auto-spacing](https://github.com/vinta/pangu.js)

记得我刚学编程那会，在网上看教程学习，经常看到有人自诩说自己有代码洁癖。但其实看别人写的代码久了，发现并不是那么一回事。我发现很多人代码虽然写的还算规范，但代码注释却写的很随意，排版也不统一，什么风格都有。可代码注释，也是代码的一部分不是吗？

<!-- more -->

前段时间我写过一篇关于[《Go 项目文件命名规范是什么？》](https://jianghushinian.cn/2024/05/25/what-is-the-naming-convention-for-go-project-files/)的文章，反响还不错，所以决定今天来写一写关于代码注释、`README.md` 等中文文案排版这个事。

### 中文文案排版指北

我们知道，任何主流编程语言，都有代码规范，无论是官方规范还是社区规范。比如 Python 界大名鼎鼎的 [PEP 8 – Style Guide for Python Code](https://peps.python.org/pep-0008/)，Go 语言中的 [Google Go Style](https://google.github.io/styleguide/go/) 等。

有些代码规范中会写明代码注释相关的规范或风格，但由于众所周知的原因，这些规范的语境大都是英文。而我们自己在开发过程中，很多项目的代码注释部分都会选择写中文注释。此外，我们在编写项目 `README.md` 文档，或者 `Release Notes` 时，也经常需要写中文。

此时，我们就需要一个关于中文文案的排版规范，来约束文案风格。而程序员写的代码注释、`README.md` 等内容中，一个最大的特点就是**中英文混杂**，比如你正在看的此文。

所以，我们需要一个对**中英文混杂**情况重点考虑的文案排版规范。幸运的是，GitHub 中有一个叫 [chinese-copywriting-guidelines](https://github.com/sparanoid/chinese-copywriting-guidelines) 的项目就是我们要找的规范，而该项目 `README.md` 中的标题即为本文的标题「中文文案排版指北」。该项目旨在：**统一中文文案、排版的相关用法，降低团队成员之间的沟通成本，增强网站气质**。

接下来我们就一起来看一下，该项目对中文文案排版做了哪些约束或建议。

#### 空格

首先我们最应该考虑的就是空格问题。

##### 中英文之间需要增加空格

1. 在 LeanCloud 上，数据存储是围绕 `AVObject` 进行的。

2. 在LeanCloud上，数据存储是围绕`AVObject`进行的。

3. 在 LeanCloud上，数据存储是围绕`AVObject` 进行的。

以上 3 句话你认为哪一句看起来最舒服？

如果你的答案是都一样，那么你可能对文字排版不那么敏感，或者也许你不是本文的目标读者。

在我看来，第 1 句话是看起来最为规范和舒适的。因为它在中英文之间都加了空格，这样文字看起来就不会那么紧凑，比较有美感。而第 2、3 两句的写法，一个句子还不算明显，如果是一大段文本，就会显得非常凌乱。

不过凡事总有例外，比如像「豆瓣FM」这种**产品名词或者专有名词等**，我们还是应该按照官方所定义的格式书写，不然就失去了原有语义。

> NOTE: 顺便打个广告，我的公众号叫「[Go编程世界](https://jianghushinian.cn/about/)」，中英文之间同样没有空格。
> 当时起名字时，其实我是写了空格的，但是为了照顾到需要手动输入文字搜索的同学，和参考了市面上大多数产品名词，我还是决定改成了没有空格的版本。

除了要考虑中英文之间是否需要增加空格，还有一些场景也要考虑空格问题，下面来一一介绍。

##### 中文与数字之间需要增加空格

正确：

> 今天出去买菜花了 5000 元。

错误：

> 今天出去买菜花了 5000元。
>
> 今天出去买菜花了5000元。

##### 数字与单位之间需要增加空格

正确：

> 我家的光纤入屋宽带有 10 Gbps，SSD 一共有 20 TB。

错误：

> 我家的光纤入屋宽带有 10Gbps，SSD 一共有 20TB。

例外：度数／百分比与数字之间不需要增加空格：

正确：

> 角度为 90° 的角，就是直角。
>
> 新 MacBook Pro 有 15% 的 CPU 性能提升。

错误：

> 角度为 90 ° 的角，就是直角。
>
> 新 MacBook Pro 有 15 % 的 CPU 性能提升。

##### 全角标点与其他字符之间不加空格

正确：

> 刚刚买了一部 iPhone，好开心！

错误：

> 刚刚买了一部 iPhone ，好开心！
>
> 刚刚买了一部 iPhone， 好开心！

#### 标点符号

关于标点符号，也有很多考究。

##### 不重复使用标点符号

虽然中国大陆的标点符号用法允许重复使用标点符号，但是这么做会破坏句子的美观性。

正确：

> 德国队竟然战胜了巴西队！
>
> 她竟然对你说「喵」？！

错误：

> 德国队竟然战胜了巴西队！！
>
> 德国队竟然战胜了巴西队！！！！！！！！
>
> 她竟然对你说「喵」？？！！
>
> 她竟然对你说「喵」？！？！？？！！

> NOTE: 这一点，我个人持保留意见。因为有时候确实重复使用多个标点能够表示强调，甚至重要的事情还要说三遍呢，不是吗？😄。

##### 全角和半角

不明白什么是全角（全形）与半角（半形）符号？请查看维基百科条目『[全角和半角](https://zh.wikipedia.org/wiki/%E5%85%A8%E5%BD%A2%E5%92%8C%E5%8D%8A%E5%BD%A2)』。

###### 使用全角中文标点

正确：

> 嗨！你知道嘛？今天前台的小妹跟我说「喵」了哎！
>
> 核磁共振成像（NMRI）是什么原理都不知道？JFGI！

错误：

> 嗨! 你知道嘛? 今天前台的小妹跟我说 "喵" 了哎！
>
> 嗨!你知道嘛?今天前台的小妹跟我说"喵"了哎！
>
> 核磁共振成像 (NMRI) 是什么原理都不知道? JFGI!
>
> 核磁共振成像(NMRI)是什么原理都不知道?JFGI!

例外：中文句子内夹有英文书籍名、报刊名时，不应借用中文书名号，应以英文斜体表示。

###### 数字使用半角字符

正确：

> 这个蛋糕只卖 1000 元。

错误：

> 这个蛋糕只卖 １０００ 元。

例外：在设计稿、宣传海报中如出现极少量数字的情形时，为方便文字对齐，是可以使用全角数字的。

###### 遇到完整的英文整句、特殊名词，其内容使用半角标点

正确：

> 乔布斯那句话是怎么说的？「Stay hungry, stay foolish.」
>
> 推荐你阅读 *Hackers & Painters: Big Ideas from the Computer Age*，非常地有趣。

错误：

> 乔布斯那句话是怎么说的？「Stay hungry，stay foolish。」
>
> 推荐你阅读《Hackers＆Painters：Big Ideas from the Computer Age》，非常的有趣。

> NOTE: 这一点，我个人同样持保留意见。因为我认为《Hackers＆Painters: Big Ideas from the Computer Age》写法其实还挺直观的，书名号能够直接表达出这是一本书，而非博客、期刊等。这里你还要注意，`Hackers＆Painters:` 中的 `:` 是半角符号。

#### 名词

对于专有名词，是需要特别强调的。因为这代表了一个程序员的**专业性**。

比如，一个简历上专业技能写着 `HTML、CSS、JavaScript` 的前端开发，和一个写着 `Html、Css、JS` 的前端开发，凭直觉你认为谁更专业呢？

##### 专有名词使用正确的大小写

大小写相关用法原属于英文书写范畴，不属于本 wiki 讨论内容，在这里只对部分易错用法进行简述。

正确：

> 使用 GitHub 登录
>
> 我们的客户有 GitHub、Foursquare、Microsoft Corporation、Google、Facebook, Inc.。

错误：

> 使用 github 登录
>
> 使用 GITHUB 登录
>
> 使用 Github 登录
>
> 使用 gitHub 登录
>
> 使用 gｲんĤЦ8 登录
>
> 我们的客户有 github、foursquare、microsoft corporation、google、facebook, inc.。
>
> 我们的客户有 GITHUB、FOURSQUARE、MICROSOFT CORPORATION、GOOGLE、FACEBOOK, INC.。
>
> 我们的客户有 Github、FourSquare、MicroSoft Corporation、Google、FaceBook, Inc.。
>
> 我们的客户有 gitHub、fourSquare、microSoft Corporation、google、faceBook, Inc.。
>
> 我们的客户有 gｲんĤЦ8、ｷouЯƧquﾑгє、๓เςг๏ร๏Ŧt ς๏гק๏гคtเ๏ภn、900913、ƒ4ᄃëв๏๏к, IПᄃ.。

##### 不要使用不地道的缩写

正确：

> 我们需要一位熟悉 TypeScript、HTML5，至少理解一种框架（如 React、Next.js）的前端开发者。

错误：

> 我们需要一位熟悉 Ts、h5，至少理解一种框架（如 RJS、nextjs）的 FED。

#### 争议

有规范，就会有争议，毕竟萝卜青菜各有所爱。

以下用法略带有个人色彩，即：无论是否遵循下述规则，从语法的角度来讲都是**正确**的。

##### 链接之间增加空格

用法：

> 请 [提交一个 issue](#) 并分配给相关同事。
>
> 访问我们网站的最新动态，请 [点击这里](#) 进行订阅！

对比用法：

> 请[提交一个 issue](#)并分配给相关同事。
>
> 访问我们网站的最新动态，请[点击这里](#)进行订阅！

##### 简体中文使用直角引号

用法：

> 「老师，『有条不紊』的『紊』是什么意思？」

对比用法：

> “老师，‘有条不紊’的‘紊’是什么意思？”

以上，就是 [vinta/paranoid-auto-spacing](https://github.com/vinta/pangu.js) 项目给出的指导性意见，其中有些我持保留意见的地方都加了 `NOTE` 说明。

### 参考项目

如果你平时善于观察，其实很多项目或文章，都采用了类似「中文文案排版指北」中的规范。

#### 开源项目文档

比如著名的 [Kubernetes](https://github.com/kubernetes/kubernetes) 项目[官方文档](https://kubernetes.io/zh-cn/)中文版，随便拿出[一段内容](https://kubernetes.io/zh-cn/docs/tasks/configure-pod-container/assign-memory-resource/)：

> 要为容器指定内存请求，请在容器资源清单中包含 `resources: requests` 字段。 同理，要指定内存限制，请包含 `resources: limits`。
>
> 在本练习中，你将创建一个拥有一个容器的 Pod。 容器将会请求 100 MiB 内存，并且内存会被限制在 200 MiB 以内。 这是 Pod 的配置文件：

就非常规范且美观。

不过，如果你足够细心，其实也会发现文档中存在一点小瑕疵。这段文字在每个句号 `。` 之后都会存在一个空格，比如 `...一个容器的 Pod。 容器将会...`。实际上中文语境下，句号后面是不应该存在空格的。

不过这无伤大雅，这份文档已经非常规范了，且整个文档都非常统一。

#### 代码注释

许多中文开源项目的注释也值得参考。

比如我写的 [Todo List](https://github.com/jianghushinian/todo-list) 项目，旨在带大家用 Python 撸一个 Web 服务器。

虽然项目有点简陋，但代码中的每一行中文注释，我都是按照规范写的。

比如如下代码：

```python
def make_response(request, headers=None):
    """构造响应报文

    Args:
        request: 请求对象
        headers: 响应头

    Returns:
        返回响应报文
    """
    # 默认状态码为 200
    status = 200
    # 处理静态资源请求
    if request.path.startswith('/static'):
        route, methods = routes.get('/static')
    else:
        try:
            route, methods = routes.get(request.path)
        except TypeError:
            # 返回给用户 404 页面
            return bytes(errors[404]())

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

    logger(f'response_bytes: {response_bytes}')
    return response_bytes
```

> NOTE: 这里自卖自夸一下😄，你也可以参考中文社区中其他开源项目。如果有比较好的推荐，也欢迎联系我加入到本篇文章的「延伸阅读」部分。

#### 文章

我们平时学习编程过程中，都会看很多文章。而文章排版是否规范，其实很大程度上决定了我们是否要继续阅读下去。

这一点我认为做的最好的就是[极客时间](https://time.geekbang.org/)的专栏文章了。

比如[这段内容](https://time.geekbang.org/column/article/39972)：

> 很多大公司，比如 BAT、Google、Facebook，面试的时候都喜欢考算法、让人现场写代码。有些人虽然技术不错，但每次去面试都会“跪”在算法上，很是可惜。那你有没有想过，为什么这些大公司都喜欢考算法呢？

随便拿出一篇，都是赏心悦目，这一点不得不夸夸极客时间的编辑。

> NOTE: 非广告，我是发自内心觉得极客时间专栏在同类产品中写的最为规范。

#### 书籍

不知道你有没有注意，对于中文文案中混杂英文的部分，有时候我们会用 `` ` `` 把英文围起来，就会表现出 `xxx` 格式；有时候却不会围起来，只会表现出 xxx 格式，比如：

> 在 LeanCloud 上，数据存储是围绕 `AVObject` 进行的。

在我看来是否使用 `` ` `` 围起来，需要视情况而定。比如上面这段文字，`AVObject` 是我们关注的重点，使用 `` ` `` 包裹起来，这有点类似「专有名词」。

再比如：

> Go 语言中的 `slog` 日志库。

看得多了，你就有感觉了，知道什么时候需要加 `` ` ``，什么时候不需要。

关于这一点可以参考[阮一峰](https://ruanyifeng.com/)老师写的[《ES6 入门教程》](https://es6.ruanyifeng.com/)一书，比如[这段内容](https://es6.ruanyifeng.com/#docs/let)：

> ES6 新增了`let`命令，用来声明变量。它的用法类似于`var`，但是所声明的变量，只在`let`命令所在的代码块内有效。

不过，这本书也有一些小瑕疵，就是阮老师在一些中英文之间没有加空格。

看来，没有绝对遵循规范的完美文档😄。

因为只要是人为书写的内容，即使再优雅的规范，也会可能写错，或者有人不遵守。

那么解决这个问题的方案，我想聪明的你应该已经想到了。没错，使用格式化工具，就像 `Gofmt` 那样。

### 工具

#### CLI 工具

这里介绍一款和「中文文案排版指北」语法规范配套 CLI 工具 [autocorrect](https://github.com/huacnlee/autocorrect)。

安装方式非常简单：

```bash
# 安装
$ curl -sSL https://git.io/JcGER | sh
# 查看版本
$ autocorrect -V          
AutoCorrect 2.11.1
```

`autocorrect` 用法也非常简单，有如下文本内容：

```bash
$ cat test.txt
你好Hello世界
```

使用 `autocorrect` 工具对文本进行格式化：

```bash
$ autocorrect test.txt 
你好 Hello 世界
```

使用 `--lint` 标志能过查看文案中到底哪行内容排版存在问题：

```bash
$ autocorrect --lint test.txt              
.
test.txt:1:1
-你好Hello世界
+你好 Hello 世界


Error: 1, Warning: 0
AutoCorrect spend time: 35.137ms
```

如果你想修改原文内容只需要加上 `--fix` 标志：

```bash
$ autocorrect --fix test.txt
AutoCorrect spend time: 26.905ms
$ cat test.txt                                       
你好 Hello 世界
```

原文内容已被修改。

不过，有一种情况比较特殊，就是前文提到的专有名词。

现在 `test.txt` 内容如下：

```bash
$ cat test.txt                                                                                                          
你好Hello世界
Go编程世界
```

如果直接使用 `autocorrect` 工具对文本进行格式化：

```bash
$ autocorrect test.txt
你好 Hello 世界
Go 编程世界
```

「Go编程世界」也被空格拆开了，这显然不是我们想要的。

好在 `autocorrect` 提供了配置文件，我们可以将「Go编程世界」加入“字典”中。这样 `autocorrect` 就能认识「Go编程世界」这个专有名词了。

在当前目录下新建 `.autocorrectrc` 配置文件：

```yaml
rules:
textRules:
  "Go编程世界": 0
```

现在再次使用 `autocorrect` 工具对文本进行格式化：

```bash
$  autocorrect test.txt
你好 Hello 世界
Go编程世界
```

问题解决。

#### 编程语言库

除了命令行工具，其实所有主流编程语言都有对应的文案格式化工具或第三方库，你可以在这个[工具列表](https://github.com/sparanoid/chinese-copywriting-guidelines/blob/master/README.zh-Hans.md#%E5%B7%A5%E5%85%B7)中找到各种编程语言对应的代码库。

拿 Go 语言工具 [autocorrect-go](https://github.com/longbridgeapp/autocorrect) 来进行演示。

使用方式如下：

```bash
$ go get github.com/longbridgeapp/autocorrect
```

```go
package main

import (
	"fmt"

	"github.com/longbridgeapp/autocorrect"
)

func main() {
	fmt.Println(autocorrect.Format("长桥LongBridge App下载"))
	// => "长桥 LongBridge App 下载"

	fmt.Println(autocorrect.Format("Ruby 2.7版本第1次发布"))
	// => "Ruby 2.7 版本第 1 次发布"

	fmt.Println(autocorrect.Format("于3月10日开始"))
	// => "于 3 月 10 日开始"

	fmt.Println(autocorrect.Format("包装日期为2013年3月10日"))
	// => "包装日期为 2013 年 3 月 10 日"

	fmt.Println(autocorrect.Format("生产环境中使用Go"))
	// => "生产环境中使用 Go"

	fmt.Println(autocorrect.Format("本番環境でGoを使用する"))
	// => "本番環境で Go を使用する"

	fmt.Println(autocorrect.Format("프로덕션환경에서Go사용"))
	// => "프로덕션환경에서 Go 사용"

	fmt.Println(autocorrect.Format("需要符号?自动转换全角字符、数字:我们将在１６：３２分出发去ＣＢＤ中心."))
	// => "需要符号？自动转换全角字符、数字：我们将在 16:32 分出发去 CBD 中心。"
}
```

执行示例代码，将得到如下输出：

```bash
$ go run main.go
长桥 LongBridge App 下载
Ruby 2.7 版本第 1 次发布
于 3 月 10 日开始
包装日期为 2013 年 3 月 10 日
生产环境中使用 Go
本番環境で Go を使用する
프로덕션환경에서 Go 사용
需要符号？自动转换全角字符、数字：我们将在 16:32 分出发去 CBD 中心。
```

可以发现，`autocorrect-go` 不只支持中文，日文和韩文也能够支持。

#### IDE 插件

如果你使用 GoLand IDE，那么恭喜你，有一款现成的插件 [AutoCorrect](https://plugins.jetbrains.com/plugin/20244-autocorrect) 可以供你使用。

![AutoCorrect Plugins](autocorrect.png)

安装好后，在编辑器界面点击「右键」就可以看到格式化工具了：

![AutoCorrect](AutoCorrect.jpg)

你也可以为其设置快捷键，方便使用。

### 总结

无规矩不成方圆，本文专门讲解了「中文文案排版指北」。

[chinese-copywriting-guidelines](https://github.com/sparanoid/chinese-copywriting-guidelines) 项目非常具有参考价值。它约束了中文文案排版格式，并且重点考虑了**中英文混杂**。

在编写文案时，我们最先要考虑的问题就是：中英文之间需要增加空格。其次还要考虑标点符号、专有名词等。

当然，没有一个规范能考虑所有情况，所以遇到规范没有提及的“灰色地带”或有争议的地方，我们可以大胆的制定自己的规范，只要自己和团队人能够共同遵守的规范，就是好规范。

不过只要是人为书写文案，就一定存在不规范的情况，这点毋庸置疑。解决方案就是可以使用各种 `Lint` 工具，帮我们格式化文案。

本文示例源码我都放在了 [GitHub](https://github.com/jianghushinian/blog-go-example/tree/main/autocorrect) 中，欢迎点击查看。

希望此文能对你有所启发。

### P.S.

本文无意冒犯于谁，规范是人制定的，每个人都可以遵守自己的规范。如果你不喜欢中英文之间写空格，那也没什么不好。每个人都有自己的权利，只不过我更喜欢这样做。

**延伸阅读**

- 中文文案排版指北：https://github.com/sparanoid/chinese-copywriting-guidelines
- 中文文案排版指北衍生项目：https://github.com/mzlogin/chinese-copywriting-guidelines
- CLI 版 autocorrect：https://github.com/huacnlee/autocorrect
- Go 语言版 autocorrect：https://github.com/longbridgeapp/autocorrect
- JetBrains IDE 插件：https://plugins.jetbrains.com/plugin/20244-autocorrect
- Kubernetes 文档：https://kubernetes.io/zh-cn/docs/home/
- Todo List：https://github.com/jianghushinian/todo-list
- 极客时间专栏：https://time.geekbang.org/resource
- ES6 入门教程：https://es6.ruanyifeng.com/
- Go 项目文件命名规范是什么？：https://jianghushinian.cn/2024/05/25/what-is-the-naming-convention-for-go-project-files/
- 本文 GitHub 示例代码：https://github.com/jianghushinian/blog-go-example/tree/main/autocorrect

**联系我**

- 公众号：[Go编程世界](https://jianghushinian.cn/about)
- 微信：jianghushinian
- 邮箱：[jianghushinian007@outlook.com](mailto:jianghushinian007@outlook.com)
- 博客：https://jianghushinian.cn
