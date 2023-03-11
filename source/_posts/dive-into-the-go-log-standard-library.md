---
title: 深入探究 Go log 标准库
date: 2023-03-11 07:56:26
tags:
- Golang
- Log
categories:
- Golang
---

Go 语言标准库中的 log 包设计简洁明了，易于上手，可以轻松记录程序运行时的信息、调试错误以及跟踪代码执行过程中的问题等。使用 log 包无需繁琐的配置即可直接使用。本文旨在深入探究 log 包的使用和原理，帮助读者更好地了解和掌握它。

<!-- more -->

### 使用

先来看一个 log 包的使用示例：

```go
package main

import "log"

func main() {
	log.Print("Print")
	log.Printf("Printf: %s", "print")
	log.Println("Println")

	log.Fatal("Fatal")
	log.Fatalf("Fatalf: %s", "fatal")
	log.Fatalln("Fatalln")

	log.Panic("Panic")
	log.Panicf("Panicf: %s", "panic")
	log.Panicln("Panicln")
}
```

假设以上代码存放在 `main.go` 中，通过 `go run main.go` 执行代码将会得到如下输出：

```bash
$ go run main.go
2023/03/08 22:33:22 Print
2023/03/08 22:33:22 Printf: print
2023/03/08 22:33:22 Println
2023/03/08 22:33:22 Fatal
exit status 1
```

以上示例代码中使用 log 包提供的 9 个函数分别对日志进行输出，最终得到 4 条打印日志。我们来分析下每个日志函数的作用，来看看为什么出现这样的结果。

log 包提供了 3 类共计 9 种方法来输出日志内容。

|函数名|作用|使用示例|
|---|---|---|
|Print|打印日志|log.Print("Print")|
|Printf|打印格式化日志|log.Printf("Printf: %s", "print")|
|Println|打印日志并换行|log.Println("Println")|
|Panic|打印日志后执行 panic(s)（s 为日志内容）|log.Panic("Panic")|
|Panicf|打印格式化日志后执行 panic(s)|log.Panicf("Panicf: %s", "panic")|
|Panicln|打印日志并换行后执行 panic(s)|log.Panicln("Panicln")|
|Fatal|打印日志后执行 os.Exit(1)|log.Fatal("Fatal")|
|Fatalf|打印格式化日志后执行 os.Exit(1)|log.Fatalf("Fatalf: %s", "fatal")|
|Fatalln|打印日志并换行后执行 os.Exit(1)|log.Panicln("Panicln")|

根据以上表格说明，我们可以知道，log 包在执行 `log.Fatal("Fatal")` 时，程序打印完日志就通过 `os.Exit(1)` 退出了。这也就可以解释上面的示例程序，为什么打印了 9 次日志，却只输出了 4 条日志，并且最后程序退出码为 1 了。

以上是 log 包最基本的使用方式，如果我们想对日志输出做一些定制，可以使用 `log.New` 创建一个自定义 `logger`：

```go
logger := log.New(os.Stdout, "[Debug] - ", log.Lshortfile)
```

`log.New` 函数接收三个参数，分别用来指定：日志输出位置（一个 `io.Writer` 对象）、日志前缀（字符串，每次打印日志都会跟随输出）、日志属性（定义好的常量，稍后会详细讲解）。

使用示例：

```go
package main

import (
	"log"
	"os"
)

func main() {
	logger := log.New(os.Stdout, "[Debug] - ", log.Lshortfile)
	logger.Println("custom logger")
}
```

示例输出：

```log
[Debug] - main.go:10: custom logger
```

以上示例中，指定日志输出到 `os.Stdout`，即标准输出；日志前缀 `[Debug] - ` 会自动被加入到每行日志的行首；这条日志没有打印当前时间，而是打印了文件名和行号，这是 `log.Lshortfile` 日志属性的作用。

日志属性可选项如下：

|属性|说明|
|---|---|
|Ldate|当前时区的日期，格式：2009/01/23|
|Ltime|当前时区的时间，格式：01:23:23|
|Lmicroseconds|当前时区的时间，格式：01:23:23.123123，精确到微妙|
|Llongfile|全文件名和行号，格式：/a/b/c/d.go:23|
|Lshortfile|当前文件名和行号，格式：d.go:23，会覆盖 Llongfile|
|LUTC|使用 UTC 而非本地时区，推荐日志全部使用 UTC 时间|
|Lmsgprefix|将 `prefix` 内容从行首移动到日志内容前面|
|LstdFlags|标准 logger 对象的初始值（等于：`Ldate\|Ltime`）|

这些属性都是预定义好的常量，不能修改，可以通过 `|` 运算符组合使用（如：`log.Ldate|log.Ltime|log.Lshortfile`）。

使用 `log.New` 函数创建 `logger` 对象以后，依然可以通过 `logger` 对象的方法修改其属性值：

|方法|作用|
|---|---|
|SetOutput|设置日志输出位置|
|SetPrefix|设置日志输出前缀|
|SetFlags|设置日志属性|

现在我们来看一个更加完整的使用示例：

```go
package main

import (
	"log"
	"os"
)

func main() {
	// 准备日志文件
	logFile, _ := os.Create("demo.log")
	defer func() { _ = logFile.Close() }()

	// 初始化日志对象
	logger := log.New(logFile, "[Debug] - ", log.Lshortfile|log.Lmsgprefix)
	logger.Print("Print")
	logger.Println("Println")

	// 修改日志配置
	logger.SetOutput(os.Stdout)
	logger.SetPrefix("[Info] - ")
	logger.SetFlags(log.Ldate|log.Ltime|log.LUTC)
	logger.Print("Print")
	logger.Println("Println")
}
```

执行以上代码，得到 `demo.log` 日志内容如下：

```log
main.go:15: [Debug] - Print
main.go:16: [Debug] - Println
```

控制台输出内容如下：

```log
[Info] - 2023/03/11 01:24:56 Print
[Info] - 2023/03/11 01:24:56 Println
```

可以发现，在 `demo.log` 日志内容中，因为指定了 `log.Lmsgprefix` 属性，所以日志前缀 `[Debug] - ` 被移动到了日志内容前面，而非行首。

因为后续通过 `logger.SetXXX` 对 `logger` 对象的属性进行了动态修改，所以最后两条日志输出到系统的标准输出。

以上，基本涵盖了 log 包的所有常用功能。接下来我们就通过走读源码的方式来更深入的了解 log 包了。

### 源码

> 注意：本文以 [Go 1.19.4 源码](https://github.com/golang/go/tree/go1.19.4/src/log)为例，其他版本可能存在差异。

Go 标准库的 log 包代码量非常少，算上注释也才 400+ 行，非常适合初学者阅读学习。

在上面介绍的第一个示例中，我们使用 log 包提供的 9 个公开函数对日志进行输出，并通过表格的形式分别介绍了函数的作用和使用示例，那么现在我们就来看看这几个函数是如何定义的：

```go
func Print(v ...any) {
	if atomic.LoadInt32(&std.isDiscard) != 0 {
		return
	}
	std.Output(2, fmt.Sprint(v...))
}

func Printf(format string, v ...any) {
	if atomic.LoadInt32(&std.isDiscard) != 0 {
		return
	}
	std.Output(2, fmt.Sprintf(format, v...))
}

func Println(v ...any) {
	if atomic.LoadInt32(&std.isDiscard) != 0 {
		return
	}
	std.Output(2, fmt.Sprintln(v...))
}

func Fatal(v ...any) {
	std.Output(2, fmt.Sprint(v...))
	os.Exit(1)
}

func Fatalf(format string, v ...any) {
	std.Output(2, fmt.Sprintf(format, v...))
	os.Exit(1)
}

func Fatalln(v ...any) {
	std.Output(2, fmt.Sprintln(v...))
	os.Exit(1)
}

func Panic(v ...any) {
	s := fmt.Sprint(v...)
	std.Output(2, s)
	panic(s)
}

func Panicf(format string, v ...any) {
	s := fmt.Sprintf(format, v...)
	std.Output(2, s)
	panic(s)
}

func Panicln(v ...any) {
	s := fmt.Sprintln(v...)
	std.Output(2, s)
	panic(s)
}
```

可以发现，这些函数代码主逻辑基本一致，都是通过 `std.Output` 输出日志。不同的是，`PrintX` 输出日志后程序就执行结束了；`Fatal` 输出日志后会执行 `os.Exit(1)`；而 `Panic` 输出日志后会执行 `panic(s)`。

那么接下来就是要搞清楚这个 `std` 对象是什么，以及它的 `Output` 方法是如何定义的。

我们先来看下 `std` 是什么：

```go
var std = New(os.Stderr, "", LstdFlags)

func New(out io.Writer, prefix string, flag int) *Logger {
	l := &Logger{out: out, prefix: prefix, flag: flag}
	if out == io.Discard {
		l.isDiscard = 1
	}
	return l
}
```

可以看到，`std` 其实就是使用 `New` 创建的一个 `Logger` 对象，日志输出到标准错误输出，日志前缀为空，日志属性为 `LstdFlags`。

这跟我们上面讲的自定义日志对象 `logger := log.New(os.Stdout, "[Debug] - ", log.Lshortfile)` 方式如出一辙。也就是说，当我们通过 `log.Print("Print")` 打印日志时，其实使用的是 log 包内部已经定义好的 `Logger` 对象。

`Logger` 定义如下：

```go
type Logger struct {
	mu        sync.Mutex // 锁，保证并发情况下对其属性操作是原子性的
	prefix    string     // 日志前缀，即 Lmsgprefix 参数值
	flag      int        // 日志属性，用来控制日志输出格式
	out       io.Writer  // 日志输出位置，实现了 io.Writer 接口即可，如 文件、os.Stderr
	buf       []byte     // 存储日志输出内容
	isDiscard int32      // 当 out = io.Discard 是，此值为 1
}
```

其中，`flag` 和 `isDiscard` 这两个属性有必要进一步解释下。

首先是 `flag` 用来记录日志属性，其合法值如下：

```go
const (
	Ldate         = 1 << iota     // the date in the local time zone: 2009/01/23
	Ltime                         // the time in the local time zone: 01:23:23
	Lmicroseconds                 // microsecond resolution: 01:23:23.123123.  assumes Ltime.
	Llongfile                     // full file name and line number: /a/b/c/d.go:23
	Lshortfile                    // final file name element and line number: d.go:23. overrides Llongfile
	LUTC                          // if Ldate or Ltime is set, use UTC rather than the local time zone
	Lmsgprefix                    // move the "prefix" from the beginning of the line to before the message
	LstdFlags     = Ldate | Ltime // initial values for the standard logger
)
```

具体含义我就不再一一解释了，前文的表格已经写的很详细了。

值得注意的是，这里在定义常量时，巧妙的使用了左移运算符 `1 << iota`，使得常量的值呈现 1、2、4、8... 这样的递增效果。其实是为了位运算方便，通过对属性进行位运算，来决定输出内容，其本质上跟基于位运算的权限管理是一样的。所以在使用 `log.New` 新建 `Logger` 对象时可以支持 `log.Ldate|log.Ltime|log.Lshortfile` 这种形式设置多个属性。

`std` 对象的属性初始值 `LstdFlags` 也是在这里定义的。

其次还有一个属性 `isDiscard`，是用来丢弃日志的。在上面介绍 `PrintX` 函数定义时，在输出日志前有一个 `if atomic.LoadInt32(&std.isDiscard) != 0` 的判断，如果结果为真，则直接 `return` 不记录日志。

在 Go 标准库的 `io` 包里，有一个 `io.Discard` 对象，`io.Discard` 实现了 `io.Writer`，它执行 Write 操作后不会产生任何实际的效果，是一个用于丢弃数据的对象。比如有时候我们不在意数据内容，但可能存在数据不读出来就无法关闭连接的情况，这时候就可以使用  `io.Copy(io.Discard, io.Reader)` 将数据写入 `io.Discard` 实现丢弃数据的效果。

使用 `New` 创建 `Logger` 对象时，如果 `out == io.Discard` 则 `l.isDiscard` 的值会被置为 `1`，所以使用 `PrintX` 函数记录的日志将会被丢弃，而 `isDiscard` 属性之所以是 `int32` 类型而不是 `bool`，是因为方便原子操作。

现在，我们终于可以来看 `std.Output` 的实现了：

```go
func (l *Logger) Output(calldepth int, s string) error {
	now := time.Now() // 获取当前时间
	var file string
	var line int
	// 加锁，保证并发安全
	l.mu.Lock()
	defer l.mu.Unlock()
	// 通过位运算来判断是否需要获取文件名和行号
	if l.flag&(Lshortfile|Llongfile) != 0 {
		// 运行 runtime.Caller 获取文件名和行号比较耗时，所以先释放锁
		l.mu.Unlock()
		var ok bool
		_, file, line, ok = runtime.Caller(calldepth)
		if !ok {
			file = "???"
			line = 0
		}
		// 获取到文件行号后再次加锁，保证下面代码并发安全
		l.mu.Lock()
	}
	// 清空上次缓存的内容
	l.buf = l.buf[:0]
	// 格式化日志头信息（如：日期时间、文件名和行号、前缀）并写入 buf
	l.formatHeader(&l.buf, now, file, line)
	// 追加日志内容到 buf
	l.buf = append(l.buf, s...)
	// 保证输出日志以 \n 结尾
	if len(s) == 0 || s[len(s)-1] != '\n' {
		l.buf = append(l.buf, '\n')
	}
	// 调用 Logger 对象的 out 属性的 Write 方法输出日志
	_, err := l.out.Write(l.buf)
	return err
}
```

`Output` 方法代码并不多，基本逻辑也比较清晰，首先根据日志属性来决定是否需要获取文件名和行号，因为调用 `runtime.Caller` 是一个耗时操作，开销比较大，为了增加并发性，暂时将锁释放，获取到文件名和行号后再重新加锁。

接下来就是准备日志输出内容了，首先清空 `buf` 中保留的上次日志信息，然后通过 `formatHeader` 方法格式化日志头信息，接着把日志内容也追加到 `buf` 中，在这之后有一个保证输出日志以 `\n` 结尾的逻辑，来保证输出的日志都是单独一行的。不知道你有没有注意到，在前文的 log 包使用示例中，我们使用 `Print` 和 `Println` 两个方法时，最终日志输出效果并无差别，使用 `Print` 打印日志也会换行，其实就是这里的逻辑决定的。

最后，通过调用 `l.out.Write` 方法，将 `buf` 内容输出。

我们来看下用来格式化日志头信息的 `formatHeader` 方法是如何定义：

```go
func (l *Logger) formatHeader(buf *[]byte, t time.Time, file string, line int) {
	// 如果没有设置 Lmsgprefix 属性，将日志前缀内容设置到行首
	if l.flag&Lmsgprefix == 0 {
		*buf = append(*buf, l.prefix...)
	}
	// 判断是否设置了日期时间相关的属性
	if l.flag&(Ldate|Ltime|Lmicroseconds) != 0 {
		// 是否设置 UTC 时间
		if l.flag&LUTC != 0 {
			t = t.UTC()
		}
		// 是否设置日期
		if l.flag&Ldate != 0 {
			year, month, day := t.Date()
			itoa(buf, year, 4)
			*buf = append(*buf, '/')
			itoa(buf, int(month), 2)
			*buf = append(*buf, '/')
			itoa(buf, day, 2)
			*buf = append(*buf, ' ')
		}
		// 是否设置时间
		if l.flag&(Ltime|Lmicroseconds) != 0 {
			hour, min, sec := t.Clock()
			itoa(buf, hour, 2)
			*buf = append(*buf, ':')
			itoa(buf, min, 2)
			*buf = append(*buf, ':')
			itoa(buf, sec, 2)
			if l.flag&Lmicroseconds != 0 {
				*buf = append(*buf, '.')
				itoa(buf, t.Nanosecond()/1e3, 6)
			}
			*buf = append(*buf, ' ')
		}
	}
	// 是否设置文件名和行号
	if l.flag&(Lshortfile|Llongfile) != 0 {
		if l.flag&Lshortfile != 0 {
			short := file
			for i := len(file) - 1; i > 0; i-- {
				if file[i] == '/' {
					short = file[i+1:]
					break
				}
			}
			file = short
		}
		*buf = append(*buf, file...)
		*buf = append(*buf, ':')
		itoa(buf, line, -1)
		*buf = append(*buf, ": "...)
	}
	// 如果设置了 Lmsgprefix 属性，将日志前缀内容放到日志头信息最后，也就是紧挨着日志内容前面
	if l.flag&Lmsgprefix != 0 {
		*buf = append(*buf, l.prefix...)
	}
}
```

`formatHeader` 方法是 log 包里面代码量最多的一个方法，主要通过按位与（&）来计算是否设置了某个日志属性，然后根据日志属性来格式化头信息。

其中多次调用 `itoa` 函数，`itoa` 顾名思义，将 `int` 转换成 `ASCII` 码，`itoa` 定义如下：

```go
func itoa(buf *[]byte, i int, wid int) {
	// Assemble decimal in reverse order.
	var b [20]byte
	bp := len(b) - 1
	for i >= 10 || wid > 1 {
		wid--
		q := i / 10
		b[bp] = byte('0' + i - q*10)
		bp--
		i = q
	}
	// i < 10
	b[bp] = byte('0' + i)
	*buf = append(*buf, b[bp:]...)
}
```

这个函数短小精悍，它接收三个参数，`buf` 用来保存转换后的内容，`i` 就是带转换的值，比如 `year`、`month` 等，`wid` 表示转换后 `ASCII` 码字符串宽度，如果传进来的 `i` 宽度不够，则前面补零。比如传入 `itoa(&b, 12, 3)`，最终输出字符串为 `012`。

至此，我们已经理清了 `log.Print("Print")` 是如何打印一条日志的，其函数调用流程如下：

![调用流程](flow.png)

上面我们讲解了使用 log 包中默认的 `std` 这个 `Logger` 对象打印日志的调用流程。当我们使用自定义的 `Logger` 对象（`logger := log.New(os.Stdout, "[Debug] - ", log.Lshortfile)`）来打印日志时，调用的 `loggger.Print` 是一个方法，而不是 `log.Print` 这个包级别的函数，所以其实 `Logger` 结构体也实现了 9 种输出日志方法： 

```go
func (l *Logger) Print(v ...any) {
	if atomic.LoadInt32(&l.isDiscard) != 0 {
		return
	}
	l.Output(2, fmt.Sprint(v...))
}

func (l *Logger) Printf(format string, v ...any) {
	if atomic.LoadInt32(&l.isDiscard) != 0 {
		return
	}
	l.Output(2, fmt.Sprintf(format, v...))
}

func (l *Logger) Println(v ...any) {
	if atomic.LoadInt32(&l.isDiscard) != 0 {
		return
	}
	l.Output(2, fmt.Sprintln(v...))
}

func (l *Logger) Fatal(v ...any) {
	l.Output(2, fmt.Sprint(v...))
	os.Exit(1)
}

func (l *Logger) Fatalf(format string, v ...any) {
	l.Output(2, fmt.Sprintf(format, v...))
	os.Exit(1)
}

func (l *Logger) Fatalln(v ...any) {
	l.Output(2, fmt.Sprintln(v...))
	os.Exit(1)
}

func (l *Logger) Panic(v ...any) {
	s := fmt.Sprint(v...)
	l.Output(2, s)
	panic(s)
}

func (l *Logger) Panicf(format string, v ...any) {
	s := fmt.Sprintf(format, v...)
	l.Output(2, s)
	panic(s)
}

func (l *Logger) Panicln(v ...any) {
	s := fmt.Sprintln(v...)
	l.Output(2, s)
	panic(s)
}
```

这 9 个方法跟 log 包级别的函数一一对应，用于自定义 `Logger` 对象。

有一个值得注意的点，在这些方法内部，调用 `l.Output(2, s)` 时，第一个参数 `calldepth` 传递的是 2，这跟 `runtime.Caller(calldepth)` 函数内部实现有关，`runtime.Caller` 函数签名如下：

```go
func Caller(skip int) (pc uintptr, file string, line int, ok bool)
```

`runtime.Caller` 返回当前 Goroutine 的栈上的函数调用信息（程序计数器、文件信息、行号），其参数 `skip` 表示当前向上层的栈帧数，0 代表当前函数，也就是调用 `runtime.Caller` 的函数，1 代表上一层调用者，以此类推。

因为函数调用链为 `main.go -> log.Print -> std.Output -> runtime.Caller`，所以 `skip` 值即为 2：

- 0: 表示 `std.Output` 中调用 `runtime.Caller` 所在的源码文件和行号。
- 1: 表示 `log.Print` 中调用 `std.Output` 所在的源码文件和行号。
- 2: 表示 `main.go` 中调用 `log.Print` 所在的源码文件和行号。

这样当代码出现问题时，就能根据日志中记录的函数调用栈来找到报错的源码位置了。

另外，前文介绍过三个设置 `Logger` 对象属性的方法，分别是 `SetOutput`、`SetPrefix`、`SetFlags`，其实这三个方法各自还有与之对应的获取相应属性的方法，定义如下：

```go
func (l *Logger) Flags() int {
	l.mu.Lock()
	defer l.mu.Unlock()
	return l.flag
}

func (l *Logger) SetFlags(flag int) {
	l.mu.Lock()
	defer l.mu.Unlock()
	l.flag = flag
}

func (l *Logger) Prefix() string {
	l.mu.Lock()
	defer l.mu.Unlock()
	return l.prefix
}

func (l *Logger) SetPrefix(prefix string) {
	l.mu.Lock()
	defer l.mu.Unlock()
	l.prefix = prefix
}

func (l *Logger) Writer() io.Writer {
	l.mu.Lock()
	defer l.mu.Unlock()
	return l.out
}

func (l *Logger) SetOutput(w io.Writer) {
	l.mu.Lock()
	defer l.mu.Unlock()
	l.out = w
	isDiscard := int32(0)
	if w == io.Discard {
		isDiscard = 1
	}
	atomic.StoreInt32(&l.isDiscard, isDiscard)
}
```

其实就是针对每个私有属性，定义了 `getter`、`setter` 方法，并且每个方法内部为了保证并发安全，都进行了加锁操作。

当然，log 包级别的函数，也少不了这几个功能：

```go
func Default() *Logger { return std }

func SetOutput(w io.Writer) {
	std.SetOutput(w)
}

func Flags() int {
	return std.Flags()
}

func SetFlags(flag int) {
	std.SetFlags(flag)
}

func Prefix() string {
	return std.Prefix()
}

func SetPrefix(prefix string) {
	std.SetPrefix(prefix)
}

func Writer() io.Writer {
	return std.Writer()
}

func Output(calldepth int, s string) error {
	return std.Output(calldepth+1, s) // +1 for this frame.
}
```

至此，log 包的全部代码我们就一起走读完成了。

总结一下：log 包主要设计了 `Logger` 对象和其方法，并且为了开箱即用，在包级别又对外提供了默认的 `Logger` 对象 `std` 和一些包级别的对外函数。`Logger` 对象的方法，和包级别的函数基本上是一一对应的，签名一样，这样就大大降低了使用难度。

### 使用建议

关于 log 包的使用，我还有几条建议分享给你：

- log 默认不支持 `Debug`、`Info`、`Warn` 等更细粒度级别的日志输出方法，不过我们可以通过创建多个 `Logger` 对象的方式来实现，只需要给每个 `Logger` 对象指定不同的日志前缀即可：

```go
loggerDebug = log.New(os.Stdout, "[Debug]", log.LstdFlags)
loggerInfo = log.New(os.Stdout, "[Info]", log.LstdFlags)
loggerWarn = log.New(os.Stdout, "[Warn]", log.LstdFlags)
loggerError = log.New(os.Stdout, "[Error]", log.LstdFlags)
```

- 仅在 `main.go` 文件中使用 `log.Panic`、`log.Fatal`，下层程序遇到错误时先记录日志，然后将错误向上一层层抛出，直到调用栈顶层才决定要不要使用 `log.Panic`、`log.Fatal`。

- log 包作为 Go 标准库，仅支持日志的基本功能，不支持记录结构化日志、日志切割、Hook 等高级功能，所以更适合小型项目使用，比如一个单文件的脚本。对于中大型项目，则推荐使用一些主流的第三方日志库，如 [logrus](https://github.com/sirupsen/logrus)、[zap](https://github.com/uber-go/zap)、[glog](https://github.com/golang/glog) 等。

- 另外，如果你对 Go 标准日志库有所期待，Go 官方还打造了另一个日志库 [slog](https://github.com/golang/exp/tree/master/slog) 现已进入实验阶段，如果项目发展顺利，将可能成为 log 包的替代品。

### 总结

本文我与读者一起深入探究了 Go log 标准库，首先向大家介绍了 log 包如何使用，接着又带领大家一起走读了 log 包的源码，最后我也给出了一些自己对 log 包的使用建议。

**参考**

> Go log 源码：https://github.com/golang/go/tree/go1.19.4/src/log
