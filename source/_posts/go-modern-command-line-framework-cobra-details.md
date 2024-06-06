---
title: Go 语言现代命令行框架 Cobra 详解
date: 2023-05-08 23:12:19
tags:
- Go
- 命令行
categories:
- Go
---

Cobra 是一个 Go 语言开发的命令行（CLI）框架，它提供了简洁、灵活且强大的方式来创建命令行程序。它包含一个用于创建命令行程序的库（Cobra 库），以及一个用于快速生成基于 Cobra 库的命令行程序工具（Cobra 命令）。Cobra 是由 Go 团队成员 [spf13](https://spf13.com/) 为 [Hugo](https://gohugo.io/) 项目创建的，并已被许多流行的 Go 项目所采用，如 Kubernetes、Helm、Docker (distribution)、Etcd 等。

<!-- more -->

### 概念

Cobra 建立在命令、参数和标志这三个结构之上。要使用 Cobra 编写一个命令行程序，需要明确这三个概念。

- 命令（COMMAND）：命令表示要执行的操作。

- 参数（ARG）：是命令的参数，一般用来表示操作的对象。

- 标志（FLAG）：是命令的修饰，可以调整操作的行为。

一个好的命令行程序在使用时读起来像句子，用户会自然的理解并知道如何使用该程序。

要编写一个好的命令行程序，需要遵循的模式是 `APPNAME VERB NOUN --ADJECTIVE` 或 `APPNAME COMMAND ARG --FLAG`。

在这里 `VERB` 代表动词，`NOUN` 代表名词，`ADJECTIVE` 代表形容词。

以下是一个现实世界中好的命令行程序的例子：

```bash
$ hugo server --port=1313
```

以上示例中，`server` 是一个命令（子命令），`port` 是一个标志（`1313` 是标志的参数，但不是命令的参数 ARG）。

下面是一个 `git` 命令的例子：

```bash
$ git clone URL --bare
```

以上示例中，`clone` 是一个命令（子命令），`URL` 是命令的参数，`bare` 是标志。

### 快速开始

要使用 Cobra 创建命令行程序，需要先通过如下命令进行安装：

```bash
$ go get -u github.com/spf13/cobra/cobra
```

安装好后，就可以像其他 Go 语言库一样导入 Cobra 包并使用了。

```go
import "github.com/spf13/cobra"
```

#### 创建一个命令

假设我们要创建的命令行程序叫作 `hugo`，可以编写如下代码创建一个命令：

> `hugo/cmd/root.go`

```go
var rootCmd = &cobra.Command{
  Use:   "hugo",
  Short: "Hugo is a very fast static site generator",
  Long: `A Fast and Flexible Static Site Generator built with
                love by spf13 and friends in Go.
                Complete documentation is available at https://gohugo.io`,
  Run: func(cmd *cobra.Command, args []string) {
    fmt.Println("run hugo...")
  },
}

func Execute() {
  if err := rootCmd.Execute(); err != nil {
    fmt.Println(err)
    os.Exit(1)
  }
}
```

`cobra.Command` 是一个结构体，代表一个命令，其各个属性含义如下：

`Use` 是命令的名称。

`Short` 代表当前命令的简短描述。

`Long` 表示当前命令的完整描述。

`Run` 属性是一个函数，当执行命令时会调用此函数。

`rootCmd.Execute()` 是命令的执行入口，其内部会解析 `os.Args[1:]` 参数列表（默认情况下是这样，也可以通过 `Command.SetArgs` 方法设置参数），然后遍历命令树，为命令找到合适的匹配项和对应的标志。

#### 创建 `main.go`

按照编写 Go 程序的惯例，我们要为 `hugo` 程序编写一个 `main.go` 文件，作为程序的启动入口。

> `hugo/main.go`

```go
package main

import (
	"hugo/cmd"
)

func main() {
	cmd.Execute()
}
```

`main.go` 代码实现非常简单，只在 `main` 函数中调用了 `cmd.Execute()` 函数，来执行命令。

#### 编译并运行命令

现在，我们就可以编译并运行这个命令行程序了。

```bash
# 编译
$ go build -o hugo
# 执行
$ ./hugo
run hugo...
```

> 笔记：示例代码里没有打印 `Run` 函数的 `args` 参数内容，你可以自行打印看看结果（提示：`args` 为命令行参数列表）。

以上我们编译并执行了 `hugo` 程序，输出内容正是 `cobra.Command` 结构体中 `Run` 函数内部代码的执行结果。

我们还可以使用 `--help` 查看这个命令行程序的使用帮助。

```bash
$ ./hugo --help
A Fast and Flexible Static Site Generator built with
                love by spf13 and friends in Go.
                Complete documentation is available at https://gohugo.io

Usage:
  hugo [flags]

Flags:
  -h, --help   help for hugo
```

这里打印了 `cobra.Command` 结构体中 `Long` 属性的内容，如果 `Long` 属性不存在，则打印 `Short` 属性内容。

`hugo` 命令用法为 `hugo [flags]`，如 `hugo --help`。

这个命令行程序自动支持了 `-h/--help` 标志。

以上就是使用 Cobra 编写一个命令行程序最常见的套路，这也是 Cobra 推荐写法。

当前项目目录结构如下：

```bash
$ tree hugo     
hugo
├── cmd
│   └── root.go
├── go.mod
├── go.sum
└── main.go
```

Cobra 程序目录结构基本如此，`main.go` 作为命令行程序的入口，不要写过多的业务逻辑，所有命令都应该放在 `cmd/` 目录下，以后不管编写多么复杂的命令行程序都可以这么来设计。

### 添加子命令

与定义 `rootCmd` 一样，我们可以使用 `cobra.Command` 定义其他命令，并通过 `rootCmd.AddCommand()` 方法将其添加为 `rootCmd` 的一个子命令。

> `hugo/cmd/version.go`

```go
var versionCmd = &cobra.Command{
	Use:   "version",
	Short: "Print the version number of Hugo",
	Long:  `All software has versions. This is Hugo's`,
	Run: func(cmd *cobra.Command, args []string) {
		fmt.Println("Hugo Static Site Generator v0.9 -- HEAD")
	},
}

func init() {
	rootCmd.AddCommand(versionCmd)
}
```

现在重新编译并运行命令行程序。

```bash
$ go build -o hugo
$ ./hugo version                       
Hugo Static Site Generator v0.9 -- HEAD
```

可以发现 `version` 命令已经被加入进来了。

再次查看帮助信息：

```bash
$ ./hugo -h
A Fast and Flexible Static Site Generator built with
                love by spf13 and friends in Go.
                Complete documentation is available at https://gohugo.io

Usage:
  hugo [flags]
  hugo [command]

Available Commands:
  completion  Generate the autocompletion script for the specified shell
  help        Help about any command
  version     Print the version number of Hugo

Flags:
  -h, --help   help for hugo

Use "hugo [command] --help" for more information about a command.
```

这次的帮助信息更为丰富，除了可以使用 `hugo [flags]` 语法，由于子命令的加入，又多了一个 `hugo [command]` 语法可以使用，如 `hugo version`。

现在有三个可用命令：

`completion` 可以为指定的 Shell 生成自动补全脚本，将在 [Shell 补全](#Shell-补全) 小节进行讲解。

`help` 用来查看帮助，同 `-h/--help` 类似，可以使用 `hugo help command` 语法查看 `command` 命令的帮助信息。

`version` 为新添加的子命令。

查看子命令帮助信息：

```bash
$ ./hugo help version
All software has versions. This is Hugo's

Usage:
  hugo version [flags]

Flags:
  -h, --help   help for version
```

### 使用命令行标志

Cobra 完美适配 [pflag](https://github.com/spf13/pflag)，结合 pflag 可以更灵活的使用标志功能。

> 提示：对 pflag 不熟悉的读者可以参考我的另一篇文章[《Go 命令行参数解析工具 pflag 使用》](https://jianghushinian.cn/2023/03/27/use-of-go-command-line-parameter-parsing-tool-pflag/)。

#### 持久标志

如果一个标志是`持久的`，则意味着该标志将可用于它所分配的命令以及该命令下的所有子命令。

对于全局标志，可以定义在根命令 `rootCmd` 上。

```go
var Verbose bool
rootCmd.PersistentFlags().BoolVarP(&Verbose, "verbose", "v", false, "verbose output")
```

#### 本地标志

标志也可以是`本地的`，这意味着它只适用于该指定命令。

```go
var Source string
rootCmd.Flags().StringVarP(&Source, "source", "s", "", "Source directory to read from")
```

#### 父命令的本地标志

默认情况下，Cobra 仅解析目标命令上的本地标志，忽略父命令上的本地标志。通过在父命令上启用 `Command.TraverseChildren` 属性，Cobra 将在执行目标命令之前解析每个命令的本地标志。

```go
var rootCmd = &cobra.Command{
	Use:   "hugo",
	TraverseChildren: true,
}
```

> 提示：如果你不理解，没关系，继续往下看，稍后会有示例代码演示讲解。

#### 必选标志

默认情况下，标志是可选的。我们可以将其标记为必选，如果没有提供，则会报错。

```go
var Region string
rootCmd.Flags().StringVarP(&Region, "region", "r", "", "AWS region (required)")
rootCmd.MarkFlagRequired("region")
```

定义好以上几个标志后，为了展示效果，我们对 `rootCmd.Run` 方法做些修改，分别打印 `Verbose`、`Source`、`Region` 几个变量。

```go
var rootCmd = &cobra.Command{
	Use:   "hugo",
	Run: func(cmd *cobra.Command, args []string) {
		fmt.Println("run hugo...")
		fmt.Printf("Verbose: %v\n", Verbose)
		fmt.Printf("Source: %v\n", Source)
		fmt.Printf("Region: %v\n", Region)
	},
}
```

另外，为了测试启用 `Command.TraverseChildren` 的效果，我又添加了一个 `print` 子命令。

> `hugo/cmd/print.go`

```go
var printCmd = &cobra.Command{
	Use: "print [OPTIONS] [COMMANDS]",
	Run: func(cmd *cobra.Command, args []string) {
		fmt.Println("run print...")
		fmt.Printf("printFlag: %v\n", printFlag)
		fmt.Printf("Source: %v\n", Source)
	},
}

func init() {
	rootCmd.AddCommand(printCmd)

	// 本地标志
	printCmd.Flags().StringVarP(&printFlag, "flag", "f", "", "print flag for local")
}
```

现在，我们重新编译并运行 `hugo`，来对上面添加的这几个标志进行测试。

```bash
$ go build -o hugo
$ ./hugo -h                    
A Fast and Flexible Static Site Generator built with
                love by spf13 and friends in Go.
                Complete documentation is available at https://gohugo.io

Usage:
  hugo [flags]
  hugo [command]

Available Commands:
  completion  Generate the autocompletion script for the specified shell
  help        Help about any command
  print       
  version     Print the version number of Hugo

Flags:
  -h, --help            help for hugo
  -r, --region string   AWS region (required)
  -s, --source string   Source directory to read from
  -v, --verbose         verbose output

Use "hugo [command] --help" for more information about a command.
```

以上帮助信息清晰明了，我就不过多解释了。

执行 `hugo` 命令：

```bash
$ ./hugo -r test-region
run hugo...
Verbose: false
Source: 
Region: test-region
```

现在 `-r/--region` 为必选标志，不传将会得到 `Error: required flag(s) "region" not set` 报错。

执行 `print` 子命令：

```bash
$ ./hugo print -f test-flag
run print...
printFlag: test-flag
Source: 
```

以上执行结果可以发现，父命令的标志 `Source` 内容为空。

现在使用如下命令执行 `print` 子命令：

```bash
$ ./hugo -s test-source print -f test-flag
run print...
printFlag: test-flag
Source: test-source
```

在 `print` 子命令前，我们指定了 `-s test-source` 标志，`-s/--source` 是父命令 `hugo` 的标志，也能够被正确解析，这就是启用 `Command.TraverseChildren` 的效果。

如果我们将 `rootCmd` 的 `TraverseChildren` 属性置为 `false`，则会得到 `Error: unknown shorthand flag: 's' in -s` 报错。

```bash
# 指定 rootCmd.TraverseChildren = false 后，重新编译程序
$ go build -o hugo
# 执行同样的命令，现在会得到报错
$ ./hugo -s test-source print -f test-flag
Error: unknown shorthand flag: 's' in -s
Usage:
  hugo print [OPTIONS] [COMMANDS] [flags]

Flags:
  -f, --flag string   print flag for local
  -h, --help          help for print

Global Flags:
  -v, --verbose   verbose output

unknown shorthand flag: 's' in -s
```

### 处理配置

除了将命令行标志的值绑定到变量，我们也可以将标志绑定到 [Viper](https://github.com/spf13/viper)，这样就可以使用 `viper.Get()` 来获取标志的值了。

```go
var author string

func init() {
  rootCmd.PersistentFlags().StringVar(&author, "author", "YOUR NAME", "Author name for copyright attribution")
  viper.BindPFlag("author", rootCmd.PersistentFlags().Lookup("author"))
}
```

> 提示：对 Viper 不熟悉的读者可以参考我的另一篇文章[《在 Go 中如何使用 Viper 来管理配置》](https://jianghushinian.cn/2023/04/25/how-to-use-viper-for-configuration-management-in-go/)。

另外，我们可以使用 `cobra.OnInitialize()` 来初始化配置文件。

```go
var cfgFile string

func init() {
	cobra.OnInitialize(initConfig)
	rootCmd.Flags().StringVarP(&cfgFile, "config", "c", "", "config file")
}

func initConfig() {
	if cfgFile != "" {
		viper.SetConfigFile(cfgFile)
	} else {
		home, err := homedir.Dir()
		if err != nil {
			fmt.Println(err)
			os.Exit(1)
		}

		viper.AddConfigPath(home)
		viper.SetConfigName(".cobra")
	}

	if err := viper.ReadInConfig(); err != nil {
		fmt.Println("Can't read config:", err)
		os.Exit(1)
	}
}
```

传递给 `cobra.OnInitialize()` 的函数 `initConfig` 函数将在调用命令的 `Execute` 方法时运行。

为了展示使用 Cobra 处理配置的效果，需要修改 `rootCmd.Run` 函数的打印代码：

```go
var rootCmd = &cobra.Command{
	Use:   "hugo",
	Run: func(cmd *cobra.Command, args []string) {
		fmt.Println("run hugo...")
		fmt.Printf("Verbose: %v\n", Verbose)
		fmt.Printf("Source: %v\n", Source)
		fmt.Printf("Region: %v\n", Region)
		fmt.Printf("Author: %v\n", viper.Get("author"))
		fmt.Printf("Config: %v\n", viper.AllSettings())
	},
}
```

提供 `config.yaml` 配置文件内容如下：

```yaml
username: jianghushinian
password: 123456
server:
  ip: 127.0.0.1
  port: 8080
```

现在重新编译并运行 `hugo` 命令：

```bash
# 编译
$ go build -o hugo
# 执行
$ ./hugo -r test-region --author jianghushinian -c ./config.yaml 
run hugo...
Verbose: false
Source: 
Region: test-region
Author: jianghushinian
Config: map[author:jianghushinian password:123456 server:map[ip:127.0.0.1 port:8080] username:jianghushinian]
```

> 笔记：Cobra 同时支持 pflag 和 Viper 两个库，实际上这三个库出自同一作者 [spf13](https://spf13.com/)。

### 参数验证

在执行命令行程序时，我们可能需要对命令参数进行合法性验证，`cobra.Command` 的 `Args` 属性提供了此功能。

`Args` 属性类型为一个函数：`func(cmd *Command, args []string) error`，可以用来验证参数。

Cobra 内置了以下验证函数：

- `NoArgs`：如果存在任何命令参数，该命令将报错。

- `ArbitraryArgs`：该命令将接受任意参数。

- `OnlyValidArgs`：如果有任何命令参数不在 `Command` 的 `ValidArgs` 字段中，该命令将报错。

- `MinimumNArgs(int)`：如果没有至少 N 个命令参数，该命令将报错。

- `MaximumNArgs(int)`：如果有超过 N 个命令参数，该命令将报错。

- `ExactArgs(int)`：如果命令参数个数不为 N，该命令将报错。

- `ExactValidArgs(int)`：如果命令参数个数不为 N，或者有任何命令参数不在 `Command` 的 `ValidArgs` 字段中，该命令将报错。

- `RangeArgs(min, max)`：如果命令参数的数量不在预期的最小数量 `min` 和最大数量 `max` 之间，该命令将报错。

内置验证函数用法如下：

```go
var versionCmd = &cobra.Command{
	Use:   "version",
	Run: func(cmd *cobra.Command, args []string) {
		fmt.Println("Hugo Static Site Generator v0.9 -- HEAD")
	},
	Args: cobra.MaximumNArgs(2), // 使用内置的验证函数，位置参数多于 2 个则报错
}
```

重新编译并运行 `hugo` 命令：

```bash
# 编译
$ go build -o hugo
# 两个命令参数满足验证函数的要求
$ ./hugo version a b  
Hugo Static Site Generator v0.9 -- HEAD
# 超过两个参数则报错
$ ./hugo version a b c
Error: accepts at most 2 arg(s), received 3
```

当然，我们也可以自定义验证函数：

```go
var printCmd = &cobra.Command{
	Use: "print [OPTIONS] [COMMANDS]",
	Run: func(cmd *cobra.Command, args []string) {
		fmt.Println("run print...")
		// 命令行位置参数列表：例如执行 `hugo print a b c d` 将得到 [a b c d]
		fmt.Printf("args: %v\n", args)
	},
	// 使用自定义验证函数
	Args: func(cmd *cobra.Command, args []string) error {
		if len(args) < 1 {
			return errors.New("requires at least one arg")
		}
		if len(args) > 4 {
			return errors.New("the number of args cannot exceed 4")
		}
		if args[0] != "a" {
			return errors.New("first argument must be 'a'")
		}
		return nil
	},
}
```

重新编译并运行 `hugo` 命令：

```bash
# 编译
$ go build -o hugo
# 4 个参数满足条件
$ ./hugo print a b c d
run print...
args: [a b c d]
# 没有参数则报错
$ ./hugo print  
Error: requires at least one arg
# 第一个参数不满足验证函数逻辑，也会报错
$ ./hugo print x      
Error: first argument must be 'a'
```

### Hooks

在执行 `Run` 函数前后，我们可以执行一些钩子函数，其作用和执行顺序如下：

1. `PersistentPreRun`：在 `PreRun` 函数执行之前执行，对此命令的子命令同样生效。

2. `PreRun`：在 `Run` 函数执行之前执行。

3. `Run`：执行命令时调用的函数，用来编写命令的业务逻辑。

4. `PostRun`：在 `Run` 函数执行之后执行。

5. `PersistentPostRun`：在 `PostRun` 函数执行之后执行，对此命令的子命令同样生效。

修改 `rootCmd` 如下：

```go
var rootCmd = &cobra.Command{
	Use:   "hugo",
	PersistentPreRun: func(cmd *cobra.Command, args []string) {
		fmt.Println("hugo PersistentPreRun")
	},
	PreRun: func(cmd *cobra.Command, args []string) {
		fmt.Println("hugo PreRun")
	},
	Run: func(cmd *cobra.Command, args []string) {
		fmt.Println("run hugo...")
	},
	PostRun: func(cmd *cobra.Command, args []string) {
		fmt.Println("hugo PostRun")
	},
	PersistentPostRun: func(cmd *cobra.Command, args []string) {
		fmt.Println("hugo PersistentPostRun")
	},
}
```

重新编译并运行 `hugo` 命令：

```bash
# 编译
$ go build -o hugo
# 执行
$ ./hugo
hugo PersistentPreRun
hugo PreRun
run hugo...
hugo PostRun
hugo PersistentPostRun
```

输出顺序符合预期。

其中 `PersistentPreRun`、`PersistentPostRun` 两个函数对子命令同样生效。

```bash
$ ./hugo version 
hugo PersistentPreRun
Hugo Static Site Generator v0.9 -- HEAD
hugo PersistentPostRun
```

以上几个函数都有对应的 `<Hooks>E` 版本，`E` 表示 `Error`，即函数执行出错将会返回 `Error`，执行顺序不变：

1. `PersistentPreRunE`

2. `PreRunE`

3. `RunE`

4. `PostRunE`

5. `PersistentPostRunE`

如果定义了 `<Hooks>E` 函数，则 `<Hooks>` 函数不会执行。比如同时定义了 `Run` 和 `RunE`，则只会执行 `RunE`，不会执行 `Run`，其他 `Hooks` 函数同理。

```go
var rootCmd = &cobra.Command{
	Use:   "hugo",
	PersistentPreRun: func(cmd *cobra.Command, args []string) {
		fmt.Println("hugo PersistentPreRun")
	},
	PersistentPreRunE: func(cmd *cobra.Command, args []string) error {
		fmt.Println("hugo PersistentPreRunE")
		return nil
	},
	PreRun: func(cmd *cobra.Command, args []string) {
		fmt.Println("hugo PreRun")
	},
	PreRunE: func(cmd *cobra.Command, args []string) error {
		fmt.Println("hugo PreRunE")
		return errors.New("PreRunE err")
	},
	Run: func(cmd *cobra.Command, args []string) {
		fmt.Println("run hugo...")
	},
	PostRun: func(cmd *cobra.Command, args []string) {
		fmt.Println("hugo PostRun")
	},
	PersistentPostRun: func(cmd *cobra.Command, args []string) {
		fmt.Println("hugo PersistentPostRun")
	},
}
```

重新编译并运行 `hugo` 命令：

```bash
# 编译
$ go build -o hugo
# 执行
$ ./hugo          
hugo PersistentPreRunE
hugo PreRunE
Error: PreRunE err
Usage:
  hugo [flags]
  hugo [command]

Available Commands:
  completion  Generate the autocompletion script for the specified shell
  help        Help about any command
  print       
  version     Print the version number of Hugo

Flags:
      --author string   Author name for copyright attribution (default "YOUR NAME")
  -c, --config string   config file
  -h, --help            help for hugo
  -r, --region string   AWS region (required)
  -s, --source string   Source directory to read from
  -v, --verbose         verbose output

Use "hugo [command] --help" for more information about a command.

PreRunE err
```

可以发现，虽然同时定义了 `PersistentPreRun`、`PersistentPreRunE` 两个钩子函数，但只有 `PersistentPreRunE` 会被执行。

在执行 `PreRunE` 时返回了一个错误 `PreRunE err`，程序会终止运行并打印错误信息。

如果子命令定义了自己的 `Persistent*Run` 函数，则不会继承父命令的 `Persistent*Run` 函数。

```go
var versionCmd = &cobra.Command{
	Use:   "version",
	PersistentPreRun: func(cmd *cobra.Command, args []string) {
		fmt.Println("version PersistentPreRun")
	},
	PreRun: func(cmd *cobra.Command, args []string) {
		fmt.Println("version PreRun")
	},
	Run: func(cmd *cobra.Command, args []string) {
		fmt.Println("Hugo Static Site Generator v0.9 -- HEAD")
	},
}
```

重新编译并运行 `hugo` 命令：

```bash
# 编译
$ go build -o hugo
# 执行子命令
$ ./hugo version  
version PersistentPreRun
version PreRun
Hugo Static Site Generator v0.9 -- HEAD
hugo PersistentPostRun
```

### 定义自己的 Help 命令

如果你对 Cobra 自动生成的帮助命令不满意，我们可以自定义帮助命令或模板。

```go
cmd.SetHelpCommand(cmd *Command)
cmd.SetHelpFunc(f func(*Command, []string))
cmd.SetHelpTemplate(s string)
```

Cobra 提供了三个方法来实现自定义帮助命令，后两者也适用于任何子命令。

默认情况下，我们可以使用 `hugo help command` 语法查看子命令的帮助信息，也可以使用 `hugo command -h/--help` 查看。

使用 `help` 命令查看帮助信息：

```bash
$ ./hugo help version
hugo PersistentPreRunE
All software has versions. This is Hugo's

Usage:
  hugo version [flags]

Flags:
  -h, --help   help for version

Global Flags:
      --author string   Author name for copyright attribution (default "YOUR NAME")
  -v, --verbose         verbose output
hugo PersistentPostRun
```

使用 `-h/--help` 查看帮助信息：

```bash
$ ./hugo version -h  
All software has versions. This is Hugo's

Usage:
  hugo version [flags]

Flags:
  -h, --help   help for version

Global Flags:
      --author string   Author name for copyright attribution (default "YOUR NAME")
  -v, --verbose         verbose output
```

二者唯一的区别是，使用 `help` 命令查看帮助信息时会执行钩子函数。

我们可以使用 `rootCmd.SetHelpCommand` 来控制 `help` 命令输出，使用 `rootCmd.SetHelpFunc` 来控制 `-h/--help` 输出。

```go
rootCmd.SetHelpCommand(&cobra.Command{
	Use:    "help",
	Short:  "Custom help command",
	Hidden: true,
	Run: func(cmd *cobra.Command, args []string) {
		fmt.Println("Custom help command")
	},
})
rootCmd.SetHelpFunc(func(command *cobra.Command, strings []string) {
	fmt.Println(strings)
})
```

重新编译并运行 `hugo` 命令：

```bash
# 编译
$ go build -o hugo
# 使用 `help` 命令查看帮助信息
$ ./hugo help version
hugo PersistentPreRunE
Custom help command
hugo PersistentPostRun
# 使用 `-h` 查看根命令帮助信息
$ ./hugo -h
[-h]
# 使用 `-h` 查看 version 命令帮助信息
$ ./hugo version -h  
[version -h]
```

可以发现，使用 `help` 命令查看帮助信息输出结果是 `rootCmd.SetHelpCommand` 中 `Run` 函数的执行输出。使用 `-h` 查看帮助信息输出结果是 `rootCmd.SetHelpFunc` 函数的执行输出，`strings` 代表的是命令行标志和参数列表。

现在我们再来测试下 `rootCmd.SetHelpTemplate` 的作用，它用来设置帮助信息模板，支持标准的 Go Template 语法，自定义模板如下：

```go
rootCmd.SetHelpTemplate(`Custom Help Template:
Usage:
	{{.UseLine}}
Description:
	{{.Short}}
Commands:
{{- range .Commands}}
	{{.Name}}: {{.Short}}
{{- end}}
`)
```

> 注意：为了单独测试 `cmd.SetHelpTemplate(s string)`，我已将上面 `rootCmd.SetHelpCommand` 和 `rootCmd.SetHelpFunc` 部分代码注释掉了。

重新编译并运行 `hugo` 命令：

```bash
# 编译
$ go build -o hugo
# 查看帮助
$ ./hugo -h            
Custom Help Template:
Usage:
        hugo [flags]
Description:
        Hugo is a very fast static site generator
Commands:
        completion: Generate the autocompletion script for the specified shell
        help: Help about any command
        print: 
        version: Print the version number of Hugo
# 查看子命令帮助
$ ./hugo help version
hugo PersistentPreRunE
Custom Help Template:
Usage:
        hugo version [flags]
Description:
        Print the version number of Hugo
Commands:
hugo PersistentPostRun
```

可以发现，无论使用 `help` 命令查看帮助信息，还是使用 `-h` 查看帮助信息，其输出内容都遵循我们自定义的模版格式。

### 定义自己的 Usage Message

当用户提供无效标志或无效命令时，Cobra 通过向用户显示 `Usage` 来提示用户如何正确的使用命令。

例如，当用户输入无效的标志 `--demo` 时，将得到如下输出：

```bash
$ ./hugo --demo
Error: unknown flag: --demo
Usage:
  hugo [flags]
  hugo [command]

Available Commands:
  completion  Generate the autocompletion script for the specified shell
  help        Help about any command
  print       
  version     Print the version number of Hugo

Flags:
      --author string   Author name for copyright attribution (default "YOUR NAME")
  -c, --config string   config file
  -h, --help            help for hugo
  -s, --source string   Source directory to read from
  -v, --verbose         verbose output

Use "hugo [command] --help" for more information about a command.

unknown flag: --demo
```

首先程序会报错 `Error: unknown flag: --demo`，报错后会显示 `Usage` 信息。

这个输出格式默认与 `help` 信息一样，我们也可以进行自定义。Cobra 提供了如下两个方法，来控制输出，具体效果我就不演示了，留给读者自行探索。

```go
cmd.SetUsageFunc(f func(*Command) error)
cmd.SetUsageTemplate(s string)
```

### 未知命令建议

在我们使用 `git` 命令时，有一个非常好用的功能，能够对用户输错的未知命令智能提示。

示例如下：

```bash
$ git statu
git: 'statu' is not a git command. See 'git --help'.

The most similar commands are
	status
	stage
	stash
```

当我们输入一个不存在的命令 `statu` 时，`git` 会提示命令不存在，并且给出几个最相似命令的建议。

这个功能非常实用，幸运的是，Cobra 自带了此功能。

如下，当我们输入一个不存在的命令 `vers` 时，`hugo` 会自动给出建议命令 `version`：

```bash
$ ./hugo vers      
Error: unknown command "vers" for "hugo"

Did you mean this?
        version

Run 'hugo --help' for usage.
unknown command "vers" for "hugo"

Did you mean this?
        version
```

> 注意⚠️：根据我的实测，要想让此功能生效，`Command.TraverseChildren` 属性要置为 `false`。

如果你想彻底关闭此功能，可以使用如下设置：

```go
command.DisableSuggestions = true
```

或者使用如下设置调整字符串匹配的最小距离：

```go
command.SuggestionsMinimumDistance = 1
```

`SuggestionsMinimumDistance` 是一个正整数，表示输错的命令与正确的命令最多有几个不匹配的字符（最小距离），才会给出建议。如当值为 `1` 时，用户输入 `hugo versiox` 会给出建议，而如果用户输入 `hugo versixx` 时，则不会给出建议，因为已经有两个字母不匹配 `version` 了。

### Shell 补全

前文在讲[添加子命令](#添加子命令)小节时，我们见到过 `completion` 子命令，可以为指定的 Shell 生成自动补全脚本，现在我们就来讲解它的用法。

直接执行 `hugo completion` 命令，我们可以查看它支持的几种 Shell 类型 `bash`、`fish`、`powershell`、`zsh`。

```bash
$ ./hugo completion  
Generate the autocompletion script for hugo for the specified shell.
See each sub-command's help for details on how to use the generated script.

Usage:
  hugo completion [command]

Available Commands:
  bash        Generate the autocompletion script for bash
  fish        Generate the autocompletion script for fish
  powershell  Generate the autocompletion script for powershell
  zsh         Generate the autocompletion script for zsh

Flags:
  -h, --help   help for completion

Global Flags:
      --author string   Author name for copyright attribution (default "YOUR NAME")
  -v, --verbose         verbose output

Use "hugo completion [command] --help" for more information about a command.
```

要想知道自己正在使用的 Shell 类型，可以使用如下命令：

```zsh
$ echo $0   
/bin/zsh
```

可以发现，我使用的是 `zsh`，所以我就以 `zsh` 为例，来演示下 `completion` 命令补全用法。

使用 `-h/--help` 我们可以查看使用说明：

```bash
$ ./hugo completion zsh -h
Generate the autocompletion script for the zsh shell.

If shell completion is not already enabled in your environment you will need
to enable it.  You can execute the following once:

        echo "autoload -U compinit; compinit" >> ~/.zshrc

To load completions in your current shell session:

        source <(hugo completion zsh)

To load completions for every new session, execute once:

#### Linux:

        hugo completion zsh > "${fpath[1]}/_hugo"

#### macOS:

        hugo completion zsh > $(brew --prefix)/share/zsh/site-functions/_hugo

You will need to start a new shell for this setup to take effect.

Usage:
  hugo completion zsh [flags]

Flags:
  -h, --help              help for zsh
      --no-descriptions   disable completion descriptions

Global Flags:
      --author string   Author name for copyright attribution (default "YOUR NAME")
  -v, --verbose         verbose output
```

根据帮助信息，如果为当前会话提供命令行补全功能，可以使用 `source <(hugo completion zsh)` 命令来实现。

如果要让命令行补全功能永久生效，Cobra 则非常贴心的为 Linux 和 macOS 提供了不同命令。

你可以根据提示选择自己喜欢的方式来实现命令行补全功能。

我这里只实现为当前会话提供命令行补全功能为例进行演示：

```bash
# 首先在项目根目录下，安装 hugo 命令行程序，安装后软件存放在 $GOPATH/bin 目录下
$ go install .
# 添加命令行补全功能
$ source <(hugo completion zsh)
# 现在命令行补全已经生效，只需要输入一个 `v`，然后按下键盘上的 `Tab` 键，命令将自动补全为 `version`
$ hugo v
# 命令已被自动补全
$ hugo version 
version PersistentPreRun
version PreRun
Hugo Static Site Generator v0.9 -- HEAD
```

其实将命令 `source <(hugo completion zsh)` 添加到 `~/.zshrc` 文件中，也能实现每次进入 `zsh` 后自动加载 `hugo` 的命令行补全功能。

> 注意：在执行 `source <(hugo completion zsh)` 前需要将 `rootCmd` 中的钩子函数内部的 `fmt.Println` 代码全部注释掉，不然打印内容会被当作命令来执行，将会得到 `Error: unknown command "PersistentPreRunE" for "hugo"` 类似报错信息，虽然命令行补全功能依然能够生效，但「没有消息才是最好的消息」。

### 生成文档

Cobra 支持生成 `Markdown`、`ReStructured Text`、`Man Page` 三种格式文档。

这里以生成 `Markdown` 格式文档为例，来演示下 Cobra 这一强大功能。

我们可以定义一个标志 `md-docs` 来决定是否生成文档：

> `hugo/cmd/root.go`

```go
var MarkdownDocs bool

func init() {
	rootCmd.Flags().BoolVarP(&MarkdownDocs, "md-docs", "m", false, "gen Markdown docs")
	...
}

func GenDocs() {
	if MarkdownDocs {
		if err := doc.GenMarkdownTree(rootCmd, "./docs/md"); err != nil {
			fmt.Println(err)
			os.Exit(1)
		}
	}
}
```

在 `main.go` 中调用 `GenDocs()` 函数。

```go
func main() {
	cmd.Execute()
	cmd.GenDocs()
}
```

现在，重新编译并运行 `hugo` 即可生成文档：

```bash
# 编译
$ go build -o hugo
# 生成文档
$ ./hugo --md-docs
# 查看生成的文档
$ tree docs/md       
docs/md
├── hugo.md
├── hugo_completion.md
├── hugo_completion_bash.md
├── hugo_completion_fish.md
├── hugo_completion_powershell.md
├── hugo_completion_zsh.md
├── hugo_print.md
└── hugo_version.md
```

可以发现，Cobra 不仅为 `hugo` 命令生成了文档，并且还生成了子命令的文档以及命令行补全的文档。

### 使用 Cobra 命令创建项目

文章读到这里，我们可以发现，其实 Cobra 项目是遵循一定套路的，目录结构、文件、模板代码都比较固定。

此时，脚手架工具就派上用场了。Cobra 提供了 `cobra-cli` 命令行工具，可以通过命令的方式快速创建一个命令行项目。

安装：

```bash
$ go install github.com/spf13/cobra-cli@latest
```

查看使用帮助：

```bash
$ cobra-cli -h       
Cobra is a CLI library for Go that empowers applications.
This application is a tool to generate the needed files
to quickly create a Cobra application.

Usage:
  cobra-cli [command]

Available Commands:
  add         Add a command to a Cobra Application
  completion  Generate the autocompletion script for the specified shell
  help        Help about any command
  init        Initialize a Cobra Application

Flags:
  -a, --author string    author name for copyright attribution (default "YOUR NAME")
      --config string    config file (default is $HOME/.cobra.yaml)
  -h, --help             help for cobra-cli
  -l, --license string   name of license for the project
      --viper            use Viper for configuration

Use "cobra-cli [command] --help" for more information about a command.
```

可以发现，`cobra-cli` 脚手架工具仅提供了少量命令和标志，所以上手难度不大。

#### 初始化模块

要使用 `cobra-cli` 生成一个项目，首先要手动创建项目根目录并使用 `go mod` 命令进行初始化。

假设我们要编写的命令行程序叫作 `cog`，模块初始化过程如下：

```bash
# 创建项目目录
$ mkdir cog
# 进入项目目录
$ cd cog
# 初始化模块
$ go mod init github.com/jianghushinian/blog-go-example/cobra/getting-started/cog
```

#### 初始化命令行程序

有了初始化好的 Go 项目，我们就可以初始化命令行程序了。

```bash
# 初始化程序
$ cobra-cli init
Your Cobra application is ready at
# 查看生成的项目目录结构
$ tree .
.
├── LICENSE
├── cmd
│   └── root.go
├── go.mod
├── go.sum
└── main.go

2 directories, 5 files
# 执行命令行程序
$ go run main.go                                                                 
A longer description that spans multiple lines and likely contains
examples and usage of using your application. For example:

Cobra is a CLI library for Go that empowers applications.
This application is a tool to generate the needed files
to quickly create a Cobra application.
```

使用 `cobra-cli` 初始化程序非常方便，只需要一个简单的 `init` 命令即可完成。

目录结构跟我们手动编写的程序相同，只不过多了一个 `LICENSE` 文件，用来存放项目的开源许可证。

通过 `go run main.go` 执行这个命令行程序，即可打印 `rootCmd.Run` 的输出结果。

使用脚手架自动生成的 `cog/main.go` 文件内容如下：

```go
/*
Copyright © 2023 NAME HERE <EMAIL ADDRESS>

*/
package main

import "github.com/jianghushinian/blog-go-example/cobra/getting-started/cog/cmd"

func main() {
	cmd.Execute()
}
```

自动生成的 `cog/cmd/root.go` 文件内容如下：

```go
/*
Copyright © 2023 NAME HERE <EMAIL ADDRESS>

*/
package cmd

import (
	"os"

	"github.com/spf13/cobra"
)



// rootCmd represents the base command when called without any subcommands
var rootCmd = &cobra.Command{
	Use:   "cog",
	Short: "A brief description of your application",
	Long: `A longer description that spans multiple lines and likely contains
examples and usage of using your application. For example:

Cobra is a CLI library for Go that empowers applications.
This application is a tool to generate the needed files
to quickly create a Cobra application.`,
	// Uncomment the following line if your bare application
	// has an action associated with it:
	// Run: func(cmd *cobra.Command, args []string) { },
}

// Execute adds all child commands to the root command and sets flags appropriately.
// This is called by main.main(). It only needs to happen once to the rootCmd.
func Execute() {
	err := rootCmd.Execute()
	if err != nil {
		os.Exit(1)
	}
}

func init() {
	// Here you will define your flags and configuration settings.
	// Cobra supports persistent flags, which, if defined here,
	// will be global for your application.

	// rootCmd.PersistentFlags().StringVar(&cfgFile, "config", "", "config file (default is $HOME/.cog.yaml)")

	// Cobra also supports local flags, which will only run
	// when this action is called directly.
	rootCmd.Flags().BoolP("toggle", "t", false, "Help message for toggle")
}
```

以上两个文件跟我们手动编写的代码没什么两样，套路完全相同，唯一不同的是每个文件头部都会多出来一个 `Copyright` 头信息，用来标记代码的 `LICENSE`。

#### 可选标志

`cobra-cli` 提供了如下三个标志分别用来设置项目的作者、许可证类型、是否使用 Viper 管理配置。

```bash
$ cobra-cli init --author "jianghushinian" --license mit --viper
Your Cobra application is ready at
```

以上命令我们指定可选标志后对项目进行了重新初始化。

现在 `LICENSE` 文件内容不再为空，而是 `MIT` 协议。

```LICENSE
The MIT License (MIT)

Copyright © 2023 jianghushinian

Permission is hereby granted...
```

并且 Go 文件 `Copyright` 头信息中作者信息也会被补全。

```go
/*
Copyright © 2023 jianghushinian

...
*/
```

> 笔记：`cobra-cli` 命令内置开源许可证支持 `GPLv2`、`GPLv3`、`LGPL`、`AGPL`、`MIT`、`2-Clause BSD` 或 `3-Clause BSD`。也可以参考[官方文档](https://github.com/spf13/cobra-cli/blob/main/README.md#configuring-the-cobra-generator)来指定自定义许可证。

> 提示：如果你对开源许可证不熟悉，可以参考我的另一篇文章[《开源协议简介》](https://jianghushinian.cn/2023/01/15/open-source-license-introduction/)。

#### 添加命令

我们可以使用 `add` 命令为程序添加新的命令，并且 `add` 命令同样支持可选标志。

```bash
$ cobra-cli add serve
$ cobra-cli add config
$ cobra-cli add create -p 'configCmd' --author "jianghushinian" --license mit --viper 
```

这里分别添加了三个命令 `serve`、`config`、`create`，前两者都是 `rootCmd` 的子命令，`create` 命令则通过 `-p 'configCmd'` 参数指定为 `config` 的子命令。

> 注意⚠️：使用 `-p 'configCmd'` 标志指定当前命令的父命令时，`configCmd` 必须是小驼峰命名法，因为 `cobra-cli` 为 `config` 生成的命令代码自动命名为 `configCmd`，而不是 `config_cmd` 或其他形式，这符合 Go 语言变量命名规范。

现在命令行程序目录结构如下：

```bash
$ tree .
.
├── LICENSE
├── cmd
│   ├── config.go
│   ├── create.go
│   ├── root.go
│   └── serve.go
├── go.mod
├── go.sum
└── main.go

2 directories, 8 files
```

可以使用如下命令执行子命令：

```bash
$ go run main.go config create
create called
```

其他新添加的命令同理。

#### 使用配置取代标志

如果你不想每次生成或添加命令时都指定选项参数，则可以定义 `~/.cobra.yaml` 文件来保存配置信息：

```yaml
author: jianghushinian <jianghushinian007@outlook.com>
year: 2023
license: MIT
useViper: true
```

再次使用 `init` 命令初始化程序：

```bash
$ cobra-cli init                                                                      
Using config file: /Users/jianghushinian/.cobra.yaml
```

会提示使用了 `~/.cobra.yaml` 配置文件。

现在 `LICENSE` 文件内容格式如下：

```LICENSE
The MIT License (MIT)

Copyright © 2023 jianghushinian <jianghushinian007@outlook.com>

...
```

Go 文件 `Copyright` 头信息也会包含日期、用户名、用户邮箱。

```go
/*
Copyright © 2023 jianghushinian <jianghushinian007@outlook.com>

...
*/
```

如果你不想把配置保存在 `~/.cobra.yaml` 中，`cobra-cli` 还提供了 `--config` 标志来指定任意目录下的配置文件。

至此，`cobra-cli` 的功能我们就都讲解完成了，还是非常方便实用的。

### 总结

在我们日常开发中，编写命令行程序是必不可少，很多开源软件都具备强大的命令行工具，如 K8s、Docker、Git 等。

一款复杂的命令行程序通常有上百种使用组合，所以如何组织和编写出好用的命令行程序是很考验开发者功底的，而 Cobra 则为我们开发命令行程序提供了足够的便利。这也是为什么我将其称为命令行框架，而不仅仅是一个 Go 第三方库。

Cobra 功能非常强大，要使用它来编写命令行程序首先要明白三个概念：命令、参数和标志。

Cobra 不仅支持子命令，还能够完美兼容 pflag 和 Viper 包，因为这三个包都是同一个作者开发的。关于标志，Cobra 支持持久标志、本地标志以及将标志标记为必选。Cobra 可以将标志绑定到 Viper，方便使用 `viper.Get()` 来获取标志的值。对于命令行参数，Cobra 提供了不少验证函数，我们也可以自定义验证函数。

Cobra 还提供了几个 Hooks 函数 `PersistentPreRun`、`PreRun`、`PostRun`、`PersistentPostRun`，可以分别在执行 `Run` 前后来处理一段逻辑。

如果觉得 Cobra 提供的默认帮助信息不能满足需求，我们还可以定义自己的 Help 命令和 Usage Message，非常灵活。

Cobra 还支持未知命令的智能提示功能以及 Shell 自动补全功能，此外，它还支持自动生成 `Markdown`、`ReStructured Text`、`Man Page` 三种格式的文档。这对命令行工具的使用者来说非常友好，还能极大减少开发者的工作量。

最后，Cobra 的命令行工具 `cobra-cli` 进一步提高了编写命令行程序的效率，非常推荐使用。

本文完整代码示例我放在了 [GitHub](https://github.com/jianghushinian/blog-go-example/tree/main/cobra/getting-started) 上，欢迎点击查看。

希望此文能对你有所帮助。

**联系我**

- 微信：jianghushinian
- 邮箱：[jianghushinian007@outlook.com](mailto:jianghushinian007@outlook.com)
- 博客地址：https://jianghushinian.cn/

**参考**

- Cobra 官网：https://cobra.dev/
- Cobra 源码：https://github.com/spf13/cobra
- Cobra 文档：https://pkg.go.dev/github.com/spf13/cobra
- Cobra-CLI 文档：https://github.com/spf13/cobra-cli/blob/main/README.md
- 本文示例代码：https://github.com/jianghushinian/blog-go-example/tree/main/cobra/getting-started
