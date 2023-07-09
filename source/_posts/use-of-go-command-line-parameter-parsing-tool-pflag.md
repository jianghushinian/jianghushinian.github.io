---
title: Go 命令行参数解析工具 pflag 使用
date: 2023-03-27 20:53:37
tags:
- Go
- 命令行
categories:
- Go
---

在使用 Go 进行开发的过程中，命令行参数解析是我们经常遇到的需求。尽管 Go 标准库提供了 flag 包用于实现命令行参数解析，但只能满足基本需要，不支持高级特性。于是 Go 社区中出现了一个叫 pflag 的第三方包，功能更加全面且足够强大。在本文中，我们将学习并掌握如何使用 pflag。

<!-- more -->

### 特点

pflag 作为 Go 内置 flag 包的替代品，具有如下特点：

- 实现了 POSIX/GNU 风格的 --flags。

- pflag 与[《The GNU C Library》](https://www.gnu.org/software/libc/manual/html_node/index.html) 中「25.1.1 程序参数语法约定」章节中 [POSIX 建议语法](https://www.gnu.org/software/libc/manual/html_node/Argument-Syntax.html)兼容。

- 兼容 Go 标准库中的 flag 包。如果直接使用 flag 包定义的全局 `FlagSet` 对象 `CommandLine`，则完全兼容；否则当你手动实例化了 `FlagSet` 对象，这时就需要为每个标志设置一个简短标志（`Shorthand`）。

### 使用

#### 基本用法

我们可以像使用 Go 标准库中的 flag 包一样使用 pflag。

```go
package main

import (
	"fmt"

	"github.com/spf13/pflag"
)

type host struct {
	value string
}

func (h *host) String() string {
	return h.value
}

func (h *host) Set(v string) error {
	h.value = v
	return nil
}

func (h *host) Type() string {
	return "host"
}

func main() {
	var ip *int = pflag.Int("ip", 1234, "help message for ip")

	var port int
	pflag.IntVar(&port, "port", 8080, "help message for port")

	var h host
	pflag.Var(&h, "host", "help message for host")

	// 解析命令行参数
	pflag.Parse()

	fmt.Printf("ip: %d\n", *ip)
	fmt.Printf("port: %d\n", port)
	fmt.Printf("host: %+v\n", h)

	fmt.Printf("NFlag: %v\n", pflag.NFlag()) // 返回已设置的命令行标志个数
	fmt.Printf("NArg: %v\n", pflag.NArg())   // 返回处理完标志后剩余的参数个数
	fmt.Printf("Args: %v\n", pflag.Args())   // 返回处理完标志后剩余的参数列表
	fmt.Printf("Arg(1): %v\n", pflag.Arg(1)) // 返回处理完标志后剩余的参数列表中第 i 项
}
```

以上示例演示的 pflag 用法跟 flag 包用法一致，可以做到二者无缝替换。

示例分别使用 `pflag.Int()`、`pflag.IntVar()`、`pflag.Var()` 三种不同方式来声明标志。其中 `ip` 和 `port` 都是 `int` 类型标志，`host` 标志则为自定义的 `host` 类型，它实现了 `pflag.Value` 接口，通过实现接口类型，标志能够支持任意类型，增加灵活性。

通过 `--help/-h` 参数查看命令行程序使用帮助：

```bash
$ go run main.go --help                      
Usage of ./main:
      --host host   help message for host
      --ip int      help message for ip (default 1234)
      --port int    help message for port (default 8080)
pflag: help requested
```

可以发现，帮助信息中的标志位置是经过重新排序的，并不是标志定义的顺序。

与 flag 包不同的是，pflag 包参数定界符是两个 `-`，而不是一个 `-`，在 pflag 中 `--` 和 `-` 具有不同含义，这点稍后会进行介绍。

`ip` 标志的默认参数为 `1234`，`port` 标志的默认参数为 `8080`。

> 注意：在有些终端下执行程序退出后，还会多打印一行 `exit status 2`，这并不意味着程序没有正常退出，而是因为 `--help` 意图就是用来查看使用帮助，所以程序在打印使用帮助信息后，主动调用 `os.Exit(2)` 退出了。

通过如下方式使用命令行程序：

```bash
$ go run main.go --ip 1 x y --host localhost a b 
ip: 1
port: 8080
host: {value:localhost}
NFlag: 2
NArg: 4
Args: [x y a b]
Arg(1): y
```

`ip` 标志的默认值已被命令行参数 `1` 所覆盖，由于没有传递 `port` 标志，所以打印结果为默认值 `8080`，`host` 标志的值也能够被正常打印。

还有 4 个非选项参数数 `x`、`y`、`a`、`b` 也都被 pflag 识别并记录了下来。这点比 flag 要强大，在 flag 包中，非选项参数数只能写在所有命令行参数最后，`x`、`y` 出现在这里程序是会报错的。

#### 进阶用法

除了像 flag 包一样的用法，pflag 还支持一些独有的用法，以下是用法示例。

```go
package main

import (
	"fmt"
	"os"

	"github.com/spf13/pflag"
)

type host struct {
	value string
}

func (h *host) String() string {
	return h.value
}

func (h *host) Set(v string) error {
	h.value = v
	return nil
}

func (h *host) Type() string {
	return "host"
}

func main() {
	flagset := pflag.NewFlagSet("test", pflag.ExitOnError)

	var ip = flagset.IntP("ip", "i", 1234, "help message for ip")

	var boolVar bool
	flagset.BoolVarP(&boolVar, "boolVar", "b", true, "help message for boolVar")

	var h host
	flagset.VarP(&h, "host", "H", "help message for host")

	flagset.SortFlags = false

	flagset.Parse(os.Args[1:])

	fmt.Printf("ip: %d\n", *ip)
	fmt.Printf("boolVar: %t\n", boolVar)
	fmt.Printf("host: %+v\n", h)

	i, err := flagset.GetInt("ip")
	fmt.Printf("i: %d, err: %v\n", i, err)
}
```

首先我们通过 `pflag.NewFlagSet` 自定义了 `FlagSet` 对象 `flagset`，之后的标志定义和解析都通过 `flagset` 来完成。

前文示例中 `pflag.Int()` 这种用法，实际上使用的是全局 `FlagSet` 对象 `CommandLine`，`CommandLine` 定义如下：

```go
var CommandLine = NewFlagSet(os.Args[0], ExitOnError)
```

现在同样使用三种不同方式来声明标志，分别为 `flagset.IntP()`、`flagset.BoolVarP()`、`flagset.VarP()`。不难发现，这三个方法的命名结尾都多了一个 `P`，它们的能力也得以升级，三个方法都多了一个 `shorthand string` 参数（`flagset.IntP` 的第 2 个参数，`flagset.BoolVarP` 和 `flagset.VarP` 的第 3 个参数）用来设置简短标志。

从声明标志的方法名中我们能够总结出一些规律：

- `pflag.<Type>` 类方法名会将标志参数值存储在指针中并返回。

- `pflag.<Type>Var` 类方法名中包含 `Var` 关键字的，会将标志参数值绑定到第一个指针类型的参数。

- `pflag.<Type>P`、`pflag.<Type>VarP` 类方法名以 `P` 结尾的，支持简短标志。

一个完整标志在命令行传参时使用的分界符为 `--`，而一个简短标志的分界符则为 `-`。

`flagset.SortFlags = false` 作用是禁止打印帮助信息时对标志进行重排序。

示例最后，使用 `flagset.GetInt()` 获取参数的值。

通过 `--help/-h` 参数查看命令行程序使用帮助：

```bash
$ go run main.go --help
Usage of test:
  -i, --ip int      help message for ip (default 1234)
  -b, --boolVar     help message for boolVar (default true)
  -H, --host host   help message for host
pflag: help requested
```

这次的帮助信息中，标志顺序没有被改变，就是声明的顺序。

每一个标志都会对应一个简短标志，如 `-b` 和 `--boolVar` 是等价的，可以更加方便的设置参数。

指定如下命令行参数运行示例：

```bash
$ go run main.go --ip 1 -H localhost --boolVar=false
ip: 1
boolVar: false
host: {value:localhost}
i: 1, err: <nil>
```

通过 `--ip 1` 使用完整标志指定 `ip` 参数值。

通过 `-H localhost` 使用简短标志指定 `host` 参数值。

布尔类型的标志指定参数 `--boolVar=false` 需要使用等号 `=` 而非空格。

#### 命令行标志语法

命令行标志遵循如下语法：

|语法|说明|
|---|---|
|`--flag`|适用于 bool 类型标志，或具有 `NoOptDefVal` 属性的标志。|
|`--flag x`|适用于非 bool 类型标志，或没有 `NoOptDefVal` 属性的标志。|
|`--flag=x`|适用于 bool 类型标志。|
|`-n 1234`/`-n=1234`/`-n1234`|简短标志，非 bool 类型且没有 `NoOptDefVal` 属性，三者等价。|

标志解析在终止符 `--` 之后停止。

整数标志接受 1234、0664、0x1234，并且可能为负数。

布尔标志接受 1, 0, t, f, true, false, TRUE, FALSE, True, False。

`Duration` 标志接受任何对 `time.ParseDuration` 有效的输入。

#### 标志名 Normalize

借助 `pflag.NormalizedName` 我们能够给标志起一个或多个别名、规范化标志名等。

```go
package main

import (
	"fmt"
	"os"
	"strings"

	"github.com/spf13/pflag"
)

func normalizeFunc(f *pflag.FlagSet, name string) pflag.NormalizedName {
	// alias
	switch name {
	case "old-flag-name":
		name = "new-flag-name"
		break
	}

	// --my-flag == --my_flag == --my.flag
	from := []string{"-", "_"}
	to := "."
	for _, sep := range from {
		name = strings.Replace(name, sep, to, -1)
	}
	return pflag.NormalizedName(name)
}

func main() {
	flagset := pflag.NewFlagSet("test", pflag.ExitOnError)

	var ip = flagset.IntP("new-flag-name", "i", 1234, "help message for new-flag-name")
	var myFlag = flagset.IntP("my-flag", "m", 1234, "help message for my-flag")

	flagset.SetNormalizeFunc(normalizeFunc)
	flagset.Parse(os.Args[1:])

	fmt.Printf("ip: %d\n", *ip)
	fmt.Printf("myFlag: %d\n", *myFlag)
}
```

要使用 `pflag.NormalizedName`，我们需要创建一个函数 `normalizeFunc`，然后将其通过 `flagset.SetNormalizeFunc(normalizeFunc)` 注入到 `flagset` 使其生效。

在 `normalizeFunc` 函数中，我们给 `new-flag-name` 标志起了一个别名 `old-flag-name`。

另外，还对标志名进行了规范化处理，带有 `-` 和 `_` 分割符的标志名，会统一规范化成以 `.` 作为分隔符的标志名。

使用示例如下：

```bash
$ go run pflag.go --old-flag-name 2 --my-flag 200
ip: 2
myFlag: 200

$ go run pflag.go --new-flag-name 3 --my_flag 300
ip: 3
myFlag: 300
```

#### NoOptDefVal

`NoOptDefVal` 是 `no option default values` 的简写。

创建标志后，可以为标志设置 `NoOptDefVal` 属性，如果标志具有 `NoOptDefVal` 属性并且在命令行上设置了标志而没有参数选项，则标志将设置为 `NoOptDefVal` 指定的值。

如下示例：

```go
var ip = flag.IntP("flagname", "f", 1234, "help message")
flag.Lookup("flagname").NoOptDefVal = "4321"
```

不同参数结果如下：

|命令行参数|结果值|
|---|---|
|--flagname=1357|ip=1357|
|--flagname|ip=4321|
|[nothing]|ip=1234|

#### 弃用/隐藏标志

使用 `flags.MarkDeprecated` 可以弃用一个标志，使用 `flags.MarkShorthandDeprecated` 可以弃用一个简短标志，使用 `flags.MarkHidden` 可以隐藏一个标志。

```go
package main

import (
	"fmt"
	"os"

	"github.com/spf13/pflag"
)

func main() {
	flags := pflag.NewFlagSet("test", pflag.ExitOnError)

	var ip = flags.IntP("ip", "i", 1234, "help message for ip")

	var boolVar bool
	flags.BoolVarP(&boolVar, "boolVar", "b", true, "help message for boolVar")

	var h string
	flags.StringVarP(&h, "host", "H", "127.0.0.1", "help message for host")

	// 弃用标志
	flags.MarkDeprecated("ip", "deprecated")
	flags.MarkShorthandDeprecated("boolVar", "please use --boolVar only")

	// 隐藏标志
	flags.MarkHidden("host")

	flags.Parse(os.Args[1:])

	fmt.Printf("ip: %d\n", *ip)
	fmt.Printf("boolVar: %t\n", boolVar)
	fmt.Printf("host: %+v\n", h)
}
```

查看使用帮助：

```bash
$ go run main.go -h                                 
Usage of test:
      --boolVar   help message for boolVar (default true)
pflag: help requested
```

从打印结果可以发现，弃用标志 `ip` 时，其对应的简短标志 `i` 也会跟着被弃用；弃用 `boolVar` 所对应的简短标志 `b` 时，`boolVar` 标志会被保留；`host` 标志则完全被隐藏。

指定如下命令行参数运行示例：

```bash
$ go run main.go --ip 1 --boolVar=false -H localhost
Flag --ip has been deprecated, deprecated
ip: 1
boolVar: false
host: localhost
```

打印信息中会提示用户 `ip` 标志已经弃用，不过使用 `--ip 1` 指定的参数值依然能够生效。

隐藏的 `host` 标志使用 `-H localhost` 指定参数值同样能够生效。

指定如下命令行参数运行示例：

```bash
$ go run main.go -i 1 -b=false --host localhost
Flag --ip has been deprecated, deprecated
Flag shorthand -b has been deprecated, please use --boolVar only
ip: 1
boolVar: false
host: localhost
```

打印信息中增加了一条简短标志 `-b` 已被弃用的提示，指定参数值依然生效。

对于弃用的 `ip` 标志，使用简短标志形式传惨 `-i 1` 同样生效。

#### 支持 flag 类型

由于 pflag 对 flag 包兼容，所以可以在一个程序中混用二者：

```go
package main

import (
	"flag"
	"fmt"

	"github.com/spf13/pflag"
)

func main() {
	var ip *int = pflag.Int("ip", 1234, "help message for ip")
	var port *int = flag.Int("port", 80, "help message for port")

	pflag.CommandLine.AddGoFlagSet(flag.CommandLine)
	pflag.Parse()

	fmt.Printf("ip: %d\n", *ip)
	fmt.Printf("port: %d\n", *port)
}
```

其中，`ip` 标志是使用 `pflag.Int()` 声明的，`port` 标志则是使用 `flag.Int()` 声明的。只需要通过 `AddGoFlagSet` 方法将 `flag.CommandLine` 注册到 pflag 中，那么 pflag 就可以使用 flag 中声明的标志集合了。

运行示例结果如下：

```bash
$ go run main.go --ip 10 --port 8000
ip: 10
port: 8000
```

### 总结

本文主要介绍了 Go第三方标志包 pflag 的特点及用法。

首先介绍了 pflag 的基本使用方法，包括声明标志、解析命令行参数、获取标志值等。接着介绍了 pflag 的进阶用法，例如自定义 `FlagSet`、使用 `pflag.<Type>P` 方法来支持简短标志。之后又对命令行标志语法进行了讲解，对于布尔值、非布尔值和简短标志，都有各自不同的语法。我们还讲解了如何借助 `pflag.NormalizedName` 给标志起一个或多个别名、规范化标志名。然后介绍了 `NoOptDefVal` 的作用和如何弃用/隐藏标志。最后通过示例演示了如何在一个程序中混用 flag 和 pflag。

> 彩蛋：不知道你有没有发现，示例中的 `ip` 标志的名称其实代表的是 `int pointer` 而非 `Internet Protocol Address`。`ip` 标志源自官方示例，不过我顺势而为又声明了 `port`、`host` 标志，算是一个程序中的谐音梗 :)。

**参考**

> pflag 源码: https://github.com/spf13/pflag
> pflag 文档: https://pkg.go.dev/github.com/spf13/pflag
> 程序参数语法约定: https://www.gnu.org/software/libc/manual/html_node/Argument-Syntax.html
