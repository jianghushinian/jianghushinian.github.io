---
title: 深入探究 Go flag 标准库
date: 2023-03-26 11:01:06
tags:
- Go
- 命令行
categories:
- Go
---

在使用 Go 进行开发的过程中，命令行参数解析是我们经常遇到的需求。而 flag 包正是一个用于实现命令行参数解析的 Go 标准库。在本文中，我们将深入探讨 flag 标准库的实现原理和使用技巧，以帮助读者更好地理解和掌握该库的使用方法。

<!-- more -->

### 使用

#### 示例

flag 基本使用示例代码如下：

```go
package main

import (
	"flag"
	"fmt"
)

type flagVal struct {
	val string
}

func (v *flagVal) String() string {
	return v.val
}

func (v *flagVal) Set(s string) error {
	v.val = s
	return nil
}

func main() {
	// 1. 使用 flag.Type() 返回  *int 类型命令行参数
	var nFlag = flag.Int("n", 1234, "help message for flag n")

	// 2. 使用 flag.TypeVar() 绑定命令行参数到 int 类型变量
	var flagvar int
	flag.IntVar(&flagvar, "flagvar", 1234, "help message for flagvar")

	// 3. 使用 flag.Var() 绑定命令行参数到实现了 flag.Value 接口的自定义类型变量
	val := flagVal{}
	flag.Var(&val, "val", "help message for val")

	// 解析命令行参数
	flag.Parse()

	fmt.Printf("nFlag: %d\n", *nFlag)
	fmt.Printf("flagvar: %d\n", flagvar)
	fmt.Printf("val: %+v\n", val)

	fmt.Printf("NFlag: %v\n", flag.NFlag()) // 返回已设置的命令行标志个数
	fmt.Printf("NArg: %v\n", flag.NArg())   // 返回处理完标志后剩余的参数个数
	fmt.Printf("Args: %v\n", flag.Args())   // 返回处理完标志后剩余的参数列表
	fmt.Printf("Arg(1): %v\n", flag.Arg(1)) // 返回处理完标志后剩余的参数列表中第 i 项
}
```

可以通过指定 `--help/-h` 参数来查看这个命令行程序的使用帮助：

```bash
$ go run main.go -h                      
Usage of ./main:
  -flagvar int
        help message for flagvar (default 1234)
  -n int
        help message for flag n (default 1234)
  -val value
        help message for val
```

这个程序接收三个命令行参数：

- `int` 类型的 `-flagvar`，默认值为 `2134`。

- `int` 类型的 `-n`，默认值为 `2134`。

- `value` 类型的 `-val`，无默认值。

我们可以将 `-flagvar`、`-n`、`-val` 称作 `flag`，即「标志」，这也是 Go 内置命令行参数解析库被命名为 flag 的原因，见名知意。

这三个参数在示例代码中，分别使用了三种不同形式来指定：

- `flag.Type()`:

`-n` 标志是使用 `var nFlag = flag.Int("n", 1234, "help message for flag n")` 来指定的。

`flag.Int` 函数签名如下：

```go
func Int(name string, value int, usage string) *int
```

`flag.Int` 函数接收三个参数，分别是标志名称、标志默认参数值、标志使用帮助信息。函数最终还会返回一个 `*int` 类型的值，表示用户在执行命令行程序时为这个标志指定的参数。

除了使用 `flag.Int` 来设置 `int` 类型标志，flag 还支持其他多种类型，如使用 `flag.String` 来设置 `string` 类型标志。

- `flag.TypeVar()`:

`-flagvar` 标志是使用 `flag.IntVar(&flagvar, "flagvar", 1234, "help message for flagvar")` 来指定的。

`flag.IntVar` 函数签名如下：

```go
func IntVar(p *int, name string, value int, usage string)
```

与 `flag.Int` 不同的是，`flag.IntVar` 函数取消了返回值，而是会将用户传递的命令行参数绑定到第一个参数 `p *int`。

除了使用 `flag.IntVar` 来绑定 `int` 类型参数到标志，flag 还提供其他多个函数来支持绑定不同类型参数到标志，如使用 `flag.StringVar` 来绑定 `string` 类型标志。

- `flag.Var()`:

`-val` 标志是使用 `flag.Var(&val, "val", "help message for val")` 来指定的。

`flag.Var` 函数签名如下：

```go
func Var(value Value, name string, usage string)
```

`flag.Var` 函数接收三个参数，后两个参数分别是标志名称、标志使用帮助信息。而用户传递的命令行参数将被绑定到第一个参数 `value`。

`value` 是 `flag.Value` 类型，而 `flag.Value` 是一个接口，定义如下：

```go
type Value interface {
	String() string
	Set(string) error
}
```

我们可以自定义类型，只要实现了 `flag.Value` 接口，都可以传递给 `flag.Var`，这极大的增加了 flag 包的灵活性。

定义完三个标志，我们还需要使用 `flag.Parse()` 来解析命令行参数，只有解析成功以后，才会将户传递的命令行参数值绑定到对应的标志变量中。之后就可以使用 `nFlag`、`flagvar`、`val` 的变量值了。

在 `main` 函数底部，使用 `flag.NFlag()`、`flag.NArg()`、`flag.Args()`、`flag.Arg(1)` 几个函数获取并展示了命令行参数相关信息。

现在我们尝试给这个命令行程序传递几个参数并执行它，看下输出结果：

```basn
$ go run main.go -n 100 -val test a b c d
nFlag: 100
flagvar: 1234
val: {val:test}
NFlag: 2
NArg: 4
Args: [a b c d]
Arg(1): b
```

我们通过 `-n 100` 为 `-n` 标志指定了参数值 `100`，最终会被赋值给 `nFlag` 变量。

由于没有指定 `flagvar` 标志的参数值，所以 `flagvar` 变量会被赋予默认值 `1234`。

接着，我们又通过 `-val test` 为 `-val` 标志指定了参数值 `test`，最终赋值给了自定义的 `flagVal` 结构体的 `val` 字段。

因为只设置了 `-n` 和 `-val` 两个标志的参数值，所以函数 `flag.NFlag()` 返回结果为 2。

`a b c d` 四个参数由于没有被定义，所以 `flag.NArg()` 返回结果为 4。

`flag.Args()` 返回的切片中存储了 `a b c d` 四个参数。

`flag.Arg(1)` 返回切片中下标为 `1` 位置的参数，即 `b`。

#### 标志类型

在上面的示例中，我们展示了 `int` 类型和自定义的 `flag.Value` 的使用，flag 包支持的所有标志类型汇总如下：

|参数类型|合法值|
|---|---|
|bool|strconv.ParseBool 能够解析的有效值，接受：1, 0, t, f, T, F, true, false, TRUE, FALSE, True, False。|
|time.Duration|time.ParseDuration 能够解析的有效值，如："300ms", "-1.5h" or "2h45m"，合法单位："ns", "us" (or "µs"), "ms", "s", "m", "h"。|
|float64|合法的浮点数类型。|
|int/int64/uint/uint64|合法的整数类型，如：1234, 0664, 0x1234，也可以是负数。|
|string|合法的字符串类型。|
|flag.Value|实现了该接口的类型。|

除了支持几种 Go 默认的原生类型外，如果我们想实现其他类型标志的定义，都可以通过 `flag.Value` 接口类型来完成。其实 flag 包内部对于 `bool`、`int` 等所有类型的定义，都实现了 `flag.Value` 接口，在稍后讲解源码过程中将会有所体现。

#### 标志语法

命令行标志支持多种语法：

|<div style="min-width: 40pt">语法</div>|说明|
|---|---|
|-flag|bool 类型标志可以使用，表示参数值为 true。|
|--flag|支持两个 `-` 字符，与 -flag 等价。|
|-flag=x|所有类型通用，为标志 flag 传递参数值 x。|
|-flag x|作用等价于 -flag=x，但是仅限非 bool 类型标志使用，假如这样使用 `cmd -x *`，其中 * 是 Unix shell 通配符，如果存在名为 0、false 等文件，则参数值结果会发生变化。|

flag 解析参数时会在第一个非标志参数之前（单独的一个 `-` 字符也是非标志参数）或终止符 `--` 之后停止。

### 源码解读

> 注意：本文以 [Go 1.19.4 源码](https://github.com/golang/go/tree/go1.19.4/src/flag)为例，其他版本可能存在差异。

熟悉了 flag 包的基本使用，接下来我们就要深入到 flag 的源码，来探究其内部是如何实现。

阅读 flag 包的源码，我们可以从使用 flag 包的流程来入手。

#### 定义标志

在 `main` 函数中，我们首先通过如下代码定义了一个标志 `-n`。

```go
var nFlag = flag.Int("n", 1234, "help message for flag n")
```

`flag.Int` 函数定义如下：

```go
func Int(name string, value int, usage string) *int {
	return CommandLine.Int(name, value, usage)
}
```

可以发现，`flag.Int` 函数调用并返回了 `CommandLine` 对象的 `Int` 方法，并将参数原样传递进去。

来看看 `CommandLine` 是个什么：

```go
var CommandLine = NewFlagSet(os.Args[0], ExitOnError)

func NewFlagSet(name string, errorHandling ErrorHandling) *FlagSet {
	f := &FlagSet{
		name:          name,
		errorHandling: errorHandling,
	}
	f.Usage = f.defaultUsage
	return f
}
```

`CommandLine` 是使用 `NewFlagSet` 创建的 `FlagSet` 结构体指针，在构造 `FlagSet` 对象时，需要两个参数 `os.Args[0]` 和 `ExitOnError`。

我们知道 `os.Args` 存储了程序执行时指定的所有命令行参数，`os.Args[0]` 就是当前命令行程序的名称，`ExitOnError` 是一个常量，用来标记在出现 `error` 时应该如何做，`ExitOnError` 表示在遇到 `error` 时退出程序。

来看下 `FlagSet` 是如何定义：

```go
type FlagSet struct {
	Usage func()

	name          string
	parsed        bool
	actual        map[string]*Flag
	formal        map[string]*Flag
	args          []string // arguments after flags
	errorHandling ErrorHandling
	output        io.Writer // nil means stderr; use Output() accessor
}
```

`Usage` 字段是一个函数，根据名字大概能够猜测出，这个函数会在指定 `--help/-h` 参数查看命令行程序使用帮助时被调用。

`parsed` 用来标记是否调用过 `flag.Parse()`。

`actual` 和 `formal` 分别用来存储从命令行解析的标志参数和在程序中指定的默认标志参数。它们都使用 `map` 来存储 `Flag` 类型的指针，`FlagSet` 可以看作是 `Flag` 结构体的「集合」。

`args` 用来保存处理完标志后剩余的参数列表。

`errorHandling` 标记在出现 `error` 时应该如何做。

`output` 用来设置输出位置，这可以改变 `--help/-h` 时展示帮助信息的输出位置。

现在来看下 `Flag` 的定义：

```go
type Flag struct {
	Name     string // 标志名称
	Usage    string // 帮助信息
	Value    Value  // 标志所对应的命令行参数值
	DefValue string // 用来记录字符串类型的默认值，它不会被改变
}
```

`Flag` 用来记录一个命令行参数，里面存储了一个标志所有信息。

可以说 `Flag` 和 `FlagSet` 两个结构体就是 flag 包的核心，所有功能都是围绕这两个结构体设计的。

标志所对应的命令行参数值为 `flag.Value` 接口类型，在前文中已经见过了，定义如下：

```go
type Value interface {
	String() string
	Set(string) error
}
```

之所以使用接口，是为了能够存储任何类型的值，除了 flag 包默认支持的内置类型，用户也可以定义自己的类型，只要实现了 `Value` 接口即可。

如我们在前文示例程序中定义的 `flagVal` 类型。

现在 `CommandLine` 的定义以及内部实现我们都看过了，是时候回过头来看一看 `CommandLine` 对象的 `Int` 方法了：

```go
func (f *FlagSet) Int(name string, value int, usage string) *int {
	p := new(int)
	f.IntVar(p, name, value, usage)
	return p
}
```

`Int` 方法内部调用了 `f.IntVar()` 方法，定义如下： 

```go
func (f *FlagSet) IntVar(p *int, name string, value int, usage string) {
	f.Var(newIntValue(value, p), name, usage)
}
```

`IntVar` 方法又调用了 `f.Var()` 方法。

`Var` 方法第一个参数为 `newIntValue(value, p)`，我们来看看 `newIntValue` 函数是如何定义的：

```go
type intValue int

func newIntValue(val int, p *int) *intValue {
	*p = val
	return (*intValue)(p)
}

func (i *intValue) Set(s string) error {
	v, err := strconv.ParseInt(s, 0, strconv.IntSize)
	if err != nil {
		err = numError(err)
	}
	*i = intValue(v)
	return err
}

func (i *intValue) Get() any { return int(*i) }

func (i *intValue) String() string { return strconv.Itoa(int(*i)) }

```

`newIntValue` 是一个构造函数，用来创建一个 `intValue` 类型的指针，`intValue` 底层类型实际上是 `int`。

定义 `intValue` 类型的目的就是为了实现 `flag.Value` 接口。

再来看下 `Var` 方法如何定义：

```go
func (f *FlagSet) Var(value Value, name string, usage string) {
	// Flag must not begin "-" or contain "=".
	if strings.HasPrefix(name, "-") {
		panic(f.sprintf("flag %q begins with -", name))
	} else if strings.Contains(name, "=") {
		panic(f.sprintf("flag %q contains =", name))
	}

	// Remember the default value as a string; it won't change.
	flag := &Flag{name, usage, value, value.String()}
	_, alreadythere := f.formal[name]
	if alreadythere {
		var msg string
		if f.name == "" {
			msg = f.sprintf("flag redefined: %s", name)
		} else {
			msg = f.sprintf("%s flag redefined: %s", f.name, name)
		}
		panic(msg) // Happens only if flags are declared with identical names
	}
	if f.formal == nil {
		f.formal = make(map[string]*Flag)
	}
	f.formal[name] = flag
}
```

`name` 参数即为标志名，在 `Var` 方法内部，首先对标志名的合法性进行了校验，不能以 `-` 开头且不包含 `=`。

接着，根据参数创建了一个 `Flag` 类型，并且校验了标志是否被重复定义。

最后将 `Flag` 保存在 `formal` 属性中。

到这里，整个函数调用关系就结束了，我们来梳理一下代码执行流程：

`flag.Int` -> `CommandLine.Int` -> `CommandLine.IntVar` -> `CommandLine.Var`。

经过这个调用过程，我们就得到了一个 `Flag` 对象，其名称为 `n`、默认参数值为 `1234`、值的类型为 `intValue`、帮助信息为 `help message for flag n`。并将这个 `Flag` 对象保存在了 `CommandLine` 这个类型为 `FlagSet` 的结构体指针对象的 `formal` 属性中。

我们在示例程序中还使用了另外两种方式定义标志。

使用 `flag.IntVar(&flagvar, "flagvar", 1234, "help message for flagvar")` 定义标志 `-flagvar`。

`flag.IntVar` 定义如下：

```go
func IntVar(p *int, name string, value int, usage string) {
	CommandLine.Var(newIntValue(value, p), name, usage)
}
```

可以发现，`flag.IntVar` 函数内部没有调用 `CommandLine.Int` 和 `CommandLine.IntVar` 的过程，而是直接调用 `CommandLine.Var`。

另外，我们还使用 `flag.Var(&val, "val", "help message for val")` 定义了 `-val` 标志。

`flag.Var` 定义如下：

```go
func Var(value Value, name string, usage string) {
	CommandLine.Var(value, name, usage)
}
```

`flag.Var` 函数内部同样直接调用了 `CommandLine.Var`，并且由于参数 `value` 已经是 `Value` 接口类型，可以无需调用 `newIntValue` 这类构造函数将 Go 内置类型转为 `Value` 类型，直接传递参数即可。

#### 解析标志参数

命令行参数定义完成了，终于到了解析部分，可以使用 `flag.Parse()` 解析命令行参数。

`flag.Parse` 函数代码如下：

```go
func Parse() {
	CommandLine.Parse(os.Args[1:])
}
```

内部同样是调用 `CommandLine` 对象对应的方法，并且将除程序名称以外的命令行参数都传递到 `Parse` 方法中，`Parse` 方法定义如下：

```go
func (f *FlagSet) Parse(arguments []string) error {
	f.parsed = true
	f.args = arguments
	for {
		seen, err := f.parseOne()
		if seen {
			continue
		}
		if err == nil {
			break
		}
		switch f.errorHandling {
		case ContinueOnError:
			return err
		case ExitOnError:
			if err == ErrHelp {
				os.Exit(0)
			}
			os.Exit(2)
		case PanicOnError:
			panic(err)
		}
	}
	return nil
}
```

首先将 `f.parsed` 标记为 `true`，在调用 `f.Parsed()` 方法时会被返回：

```go
func (f *FlagSet) Parsed() bool {
	return f.parsed
}
```

接着又将 `arguments` 保存在 `f.args` 属性中。

然后就是循环解析命令行参数的过程，每调用一次 `f.parseOne()` 解析一个标志，直到解析完成或遇到 `error` 退出程序。

`parseOne` 方法实现如下：

```go
func (f *FlagSet) parseOne() (bool, error) {
	if len(f.args) == 0 {
		return false, nil
	}
	s := f.args[0]
	if len(s) < 2 || s[0] != '-' {
		return false, nil
	}
	numMinuses := 1
	if s[1] == '-' {
		numMinuses++
		if len(s) == 2 { // "--" terminates the flags
			f.args = f.args[1:]
			return false, nil
		}
	}
	name := s[numMinuses:]
	if len(name) == 0 || name[0] == '-' || name[0] == '=' {
		return false, f.failf("bad flag syntax: %s", s)
	}

	// it's a flag. does it have an argument?
	f.args = f.args[1:]
	hasValue := false
	value := ""
	for i := 1; i < len(name); i++ { // equals cannot be first
		if name[i] == '=' {
			value = name[i+1:]
			hasValue = true
			name = name[0:i]
			break
		}
	}
	m := f.formal
	flag, alreadythere := m[name] // BUG
	if !alreadythere {
		if name == "help" || name == "h" { // special case for nice help message.
			f.usage()
			return false, ErrHelp
		}
		return false, f.failf("flag provided but not defined: -%s", name)
	}

	if fv, ok := flag.Value.(boolFlag); ok && fv.IsBoolFlag() { // special case: doesn't need an arg
		if hasValue {
			if err := fv.Set(value); err != nil {
				return false, f.failf("invalid boolean value %q for -%s: %v", value, name, err)
			}
		} else {
			if err := fv.Set("true"); err != nil {
				return false, f.failf("invalid boolean flag %s: %v", name, err)
			}
		}
	} else {
		// It must have a value, which might be the next argument.
		if !hasValue && len(f.args) > 0 {
			// value is the next arg
			hasValue = true
			value, f.args = f.args[0], f.args[1:]
		}
		if !hasValue {
			return false, f.failf("flag needs an argument: -%s", name)
		}
		if err := flag.Value.Set(value); err != nil {
			return false, f.failf("invalid value %q for flag -%s: %v", value, name, err)
		}
	}
	if f.actual == nil {
		f.actual = make(map[string]*Flag)
	}
	f.actual[name] = flag
	return true, nil
}
```

`parseOne` 代码稍微多一点，不过整体脉络还是比较清晰的。

首先对 `f.args` 参数进行了校验，接着提取标志前导符号 `-` 的个数放到 `numMinuses` 变量中，然后取出标志名并对标志语法做了检查。

接下来取出参数 `value`，并且判断标志名是否为 `-help/-h`，如果是则说明用户只想打印程序使用帮助信息，打印后 `parseOne` 会返回 `ErrHelp`，上层的调用者 `f.Parse` 就会捕获到 `ErrHelp`，然后调用 `os.Exit(0)` 直接退出程序。

其中 `f.usage()` 实现了打印帮助信息的功能，内部具体实现这里就不讲解了，因为基本上是内容排版的实现，不是核心功能，感兴趣可以自己尝试看一看。

最后就是根据参数值是否为 `bool` 类型分别进行参数绑定，将参数设置到对应的标志变量中，并将标志保存到 `f.actual` 中。

以上步骤都执行完成后，在执行 `fmt.Printf("nFlag: %d\n", *nFlag)` 时，就能够获取到 `nFlag` 被赋予的参数值了。

至此，flag 包源码的整体脉络都已经清晰了。

#### 其他代码

在我们的示例代码最后，还打印了 `NFlag()`、`NArg()`、`Args()`、`Arg(1)` 几个函数的结果。

这几个函数实现非常简单，代码如下：

```go
func NFlag() int { return len(CommandLine.actual) }
func NArg() int { return len(CommandLine.args) }
func Args() []string { return CommandLine.args }
func Arg(i int) string {
	return CommandLine.Arg(i)
}
```

由于代码过于简单，我就不进行解释了，相信通过上面的讲解，这几个函数的作用你也能做到一目了然。

flag 包还有一些其他类型，如 `stringValue`、`float64Value`，这些类型实现思路都是一样的，也不再一一讲解。

最后，flag 包其他附属的函数实现，不是主要脉络，留给读者自行查看学习。

### 总结

在开发命令行程序时，Go 标准库中的 flag 包是一个不错的选择。

本文先对 flag 包的基本使用进行了演示，接着又对源码进行了深度剖析。

flag 包支持三种方式定义标志，`flag.Parse()` 能够对命令行参数进行解析，解析成功后，就可以在代码中使用参数值了。

希望本文能对读者有所帮助。

**参考**

> Go flag 源码：https://github.com/golang/go/tree/go1.19.4/src/flag
> Go flag 文档：https://pkg.go.dev/flag
