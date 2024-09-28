---
title: '厌倦了黑底白字？用 Go 给终端点颜色瞧瞧！'
date: 2024-09-27 19:45:30
tag:
- Go
- Terminal
categories:
- Go
---

如果你每天都在使用终端，想必无法忍受终端永远都是黑白两种配色。如果你不知道终端中各种花哨的颜色是如何输出的，那么本文就来帮你解答。

而如果你恰巧在使用 Go 语言，那么你将在一分钟内学会使用 Go 语言在终端中输出彩色字符。

<!-- more -->

![Colors in terminal](colors-in-terminal.jpg)

废话不多说，我们开始吧。

### 给终端加点颜色

Go 社区中有一个叫 [fatih/color](https://github.com/fatih/color) 的包，可以非常方便的为输出的字符添加颜色。并且这个库用法非常简单，我来带你极速上手。

示例代码如下：

```go
package main

import (
	"github.com/fatih/color"
)

func main() {
	color.Cyan("Prints text in cyan.")
	color.Blue("Prints %s in blue.", "text")
	color.Red("Prints text in red.")
	color.Magenta("And many others...")
}
```

执行示例代码，得到以下输出：

![Standard colors](standard-colors.png)

没错，就是这么简单！

导入第三方包 `fatih/color`，使用 `color.Blue` 输出蓝色，然后使用 `color.Red` 输出红色。

现在你已经学会了使用 Go 语言在终端中输出彩色字符 :)。

当然，`fatih/color` 还有更多玩法。它不仅能改变输出字符颜色，还能**同时**给字符设置一些其他属性，比如字体加粗、下划线等。

示例代码如下：

```go
// 创建一个 color 对象，输出效果：蓝绿色 + 下划线
c := color.New(color.FgCyan).Add(color.Underline)
c.Println("Prints cyan text with an underline.") // 注意 Println 输出自动加换行

// 输出效果：蓝绿色 + 加粗
d := color.New(color.FgCyan, color.Bold)
d.Printf("This prints bold cyan %s\n", "too!.") // 注意 Printf 需要手动加换行

// 输出效果：红色 + 白色背景
red := color.New(color.FgRed)
whiteBackground := red.Add(color.BgWhite)
whiteBackground.Println("Red text with white background.")
```

执行示例代码，得到以下输出：

![Mix and reuse colors](mix-and-reuse-colors.png)

我们也可以将内容输出到任何 `io.Writer` 对象：

```go
f, _ := os.OpenFile("output.log", os.O_APPEND|os.O_CREATE|os.O_WRONLY, 0644)

color.New(color.FgBlue).Fprintln(f, "blue color!")

blue := color.New(color.FgBlue)
blue.Fprint(f, "This will print text in blue.\n")
```

这里将 `color` 对象的输出内容直接写入文件。

执行示例代码，得到以下输出：

![Use your own output](use-your-own-output.png)

可以使用 `vim` 打开 `output.log` 查看文件内容如下：

![output.log](use-your-own-output-content.png)

> NOTE:
> 至于文件内容为什么长这样，稍后分析。

`fatih/color` 还支持混合输出普通字符和带颜色的字符：

```go
// 输出效果：普通文本 + 黄色 warning + 红色 error
yellow := color.New(color.FgYellow).SprintFunc()
red := color.New(color.FgRed).SprintFunc()
fmt.Printf("This is a %s and this is %s.\n", yellow("warning"), red("error"))

// 模拟输出日志内容
fmt.Println(color.GreenString("Info:"), "a info log message")
fmt.Printf("%v: %v\n", color.RedString("Warn"), "a warning log message")
```

执行示例代码，得到以下输出：

![Insert into noncolor strings](insert-into-noncolor-strings.png)

最后，我们可以在不修改原来的 `fmt.PrintX` 代码的情况下，改变其输出内容属性：

```go
// 修改标准输出
color.Set(color.FgYellow)
fmt.Println("Existing text will now be in yellow")
fmt.Printf("This one %s\n", "too")
color.Unset() // 恢复设置
fmt.Println("This is normal text")

func() {
	// 在函数中使用
	color.Set(color.FgMagenta, color.Bold)
	defer color.Unset()

	fmt.Println("All text will now be bold magenta.")
}()
fmt.Println("This is normal text too")
```

执行示例代码，得到以下输出：

![Plug into existing code](plug-into-existing-code.png)

`fatih/color` 更多用法，可以参考官方 [README.md](https://github.com/fatih/color)。

现在我们对 `fatih/color` 进行一个小实战，用 [Cobra](github.com/spf13/cobra) 写一个命令行 demo 程序 `lscolor`。它模仿 Linux 的 `ls` 命令，对目录或文件输出不同颜色。

代码如下：

```go
package main

import (
	"fmt"
	"os"

	"github.com/fatih/color"
	"github.com/spf13/cobra"
)

var rootCmd = &cobra.Command{
	Use:   "lscolor",
	Short: "lscolor lists files and directories with colors",
	Args:  cobra.MaximumNArgs(1), // 允许零个或一个参数
	Run:   run,
}

func run(cmd *cobra.Command, args []string) {
	dir := "." // 默认使用当前目录
	if len(args) > 0 {
		dir = args[0] // 如果提供了参数，则使用该参数
	}

	// 获取当前目录的文件和子目录
	entries, err := os.ReadDir(dir)
	if err != nil {
		color.Red("error reading directory: %s", err)
		return
	}

	// 遍历并输出每个条目
	for _, entry := range entries {
		if entry.IsDir() {
			color.Cyan(entry.Name()) // 目录用蓝绿色
		} else {
			color.White(entry.Name()) // 文件用白色
		}
	}
}

func main() {
	if err := rootCmd.Execute(); err != nil {
		fmt.Println(err)
		os.Exit(1)
	}
}
```

> NOTE:
> 如果你对 [Cobra](github.com/spf13/cobra) 不熟悉，可以参考我的另一篇文章：[《Go 语言现代命令行框架 Cobra 详解》](https://jianghushinian.cn/2023/05/08/go-modern-command-line-framework-cobra-details/)。

执行示例代码，得到以下输出：

![lscolor](lscolor.png)

现在，你已经掌握了使用 `fatih/color` 在终端输出彩色字符，如果你对终端输出彩色字符的原理感兴趣，不妨接着往下看。

### 终端输出彩色字符原理解析

我们知道，键盘中的每一个键都在 [ASCII](https://zh.wikipedia.org/wiki/ASCII) 表中。比如字母 `A` 在 ASCII 表中对应的数字为 `65`，字母 `B` 在 ASCII 表中对应的数字为 `66`。像这些字符，叫「可显示字符」，输出到终端是可以被我们看见的。

ASCII 表中还定义了一些不可显示的字符，它们是 ASCII 表中 `0~31` 以及 `127` 这些数字，也叫「控制字符」。

比如经典的「回车」和「换行」字符，`CR`（Carriage Return） 代表回车，`LF`（Line Feed）代表换行。因为它们无法显示，通常在文件传输等场景，需要对其进行转义，所以它们都有对应的[转义字符](https://zh.wikipedia.org/wiki/转义字符)。`CR` 的转义字符是 `\r`，`LF` 转义字符是 `\n`。`CR` 在 ASCII 表中对应的数字为 `13`，`LF` 在 ASCII 表中对应的数字为 `10`。

控制字符很好用，可以控制终端的各种属性。但现有的控制字符还是太少了，有些需求无法满足，比如给终端输出的文字加上颜色。因此就有了可以使用[转义序列](https://zh.wikipedia.org/wiki/转义序列)来控制终端属性的方法。

以转义字符开头的字符序列被叫做转义序列，所以转义序列通常由转义字符后跟普通字符序列组成。转义字符 + 几个普通字符组成的转义序列就能作为一个控制字符。

终端在收到转义符时，会把其后面的紧挨着几个字符当作指令来解析，这样就能实现相应的控制。此外，我们还需要一个表示转义序列结束的标志，转义序列结束后，可以紧挨着普通的文本字符。终端在识别出有效的转义序列结束后，会执行控制命令，随后打印所接收到的普通字符。

铺垫了这么多，是时候来讲解终端是如何输出彩色字符的了。

一个控制终端输出颜色的转义序列长这样：`\e[34;4m`。

其中，`\e` 很显然是一个转义字符，表示 `ESC` 键。`ESC` 在 ASCII 表中对应的数字为 `27`，十六进制是 `0x1B`，八进制是 `033`，所以 `\e`、`\0x1B`、`\033` 写法都是等价的。

`ESC` + `[` 表示控制序列导入器（Control Sequence Introducer），简称 `CSI`，这是由 [ANSI转义序列](https://zh.wikipedia.org/wiki/ANSI转义序列) 规定的。所以 `\e[` 就表示这是一个控制序列，告诉终端控制序列开始了，接下来几个字符不要当作普通字符输出，要作为指令来解析。

`34;4` 都是普通字符，但是在这里，它们是指令。`;` 是一个分隔符，用来分隔多个指令，`34` 代表红色，`4` 代表下划线。这个指令的完整格式是 `[<PREFIX>];[<COLOR>];[<TEXT DECORATION>]`，即 `前缀;颜色;文本装饰`，每个指令之间使用 `;` 分隔，每个指令都可以省略，显然这里省略了 `PREFIX` 指令。

最后的字符 `m` 想必你已经猜到了，没错，它就是控制序列结束的标志。

所以 `\e[34;4m` 就表示：输出蓝色并且带有下划线。

我们可以执行 `echo -e '\033[34;4mHello'` 命令尝试一下，看看结果：

> NOTE:
> `echo` 的 `-e` 标志表示开启转义，否则只会当作普通文本解析。

![Blue](blue.png)

指令生效了。细心的你一定发现了问题，就是接下来的所有输出都变成了蓝色 + 下划线，如何还原呢？

我们可以使用 `\e[0m` 指令，它表示重置所有属性。

![Reset](reset.png)

现在，回过头去看看我们使用 `fatih/color` 写的示例代码，你是不是能大概猜测到它是怎么实现的。

还记得 `output.log` 文件内容吗？

![output.log](use-your-own-output-content.png)

在 [ANSI转义序列](https://zh.wikipedia.org/wiki/ANSI转义序列)中，控制指令 `ESC + [` 一般会被显示为 `^[[`。所以，这个文本内容也就一目了然了。

`fatih/color` 正是使用了 `\e[` 控制序列来实现在终端中输出彩色字符的。

好了，原理部分就讲解到这里，虽然概念比较多，但其实就是一个简单的指令罢了。

### 总结

现在你不仅学会使用 Go 语言在终端中输出带有颜色的字符，还明白了其实现原理。

无需我多做什么总结了，如果你有兴趣了解终端都支持输出哪些颜色，可以去看看 [ANSI转义序列](https://zh.wikipedia.org/wiki/ANSI转义序列)。

本文示例源码我都放在了 [GitHub](https://github.com/jianghushinian/blog-go-example/tree/main/terminal/colors) 中，欢迎点击查看。

希望此文能对你有所启发。

**延伸阅读**

- fatih/color 源码：https://github.com/fatih/color
- 字符编码：https://zh.wikipedia.org/wiki/字符编码
- ASCII：https://zh.wikipedia.org/wiki/ASCII
- 控制字符：https://zh.wikipedia.org/wiki/控制字符
- 转义字符：https://zh.wikipedia.org/wiki/转义字符
- 转义序列：https://zh.wikipedia.org/wiki/转义序列
- ANSI转义序列：https://zh.wikipedia.org/wiki/ANSI转义序列
- Colors In Terminal：https://jafrog.com/2013/11/23/colors-in-terminal.html
- Go 语言现代命令行框架 Cobra 详解：https://jianghushinian.cn/2023/05/08/go-modern-command-line-framework-cobra-details
- 本文 GitHub 示例代码：https://github.com/jianghushinian/blog-go-example/tree/main/terminal/colors

**联系我**

- 公众号：[Go编程世界](https://jianghushinian.cn/about)
- 微信：jianghushinian
- 邮箱：[jianghushinian007@outlook.com](mailto:jianghushinian007@outlook.com)
- 博客：https://jianghushinian.cn
