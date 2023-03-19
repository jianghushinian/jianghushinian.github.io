---
title: Go 第三方 log 库之 logrus 使用
date: 2023-03-15 20:51:51
tags:
- Golang
- Log
categories:
- Golang
---

Logrus 是目前 GitHub 上 Star 数量最多的 Go 日志库。尽管目前 Logrus 处于维护模式，不再引入新功能，但这并不意味着它已经死了。Logrus 仍将继续维护，以确保安全性、错误修复和提高性能。作为 Go 社区中最受欢迎的日志库之一，Logrus 最大的贡献是推动了 Golang 社区广泛使用结构化的日志记录。著名的 Docker 项目就在使用 Logrus 记录日志，这进一步证明了其在实际应用中的可靠性和实用性。

<!-- more -->

### 特点

Logrus 具有如下特点：

- 与 Go log 标准库 API 完全兼容，这意味着任何使用 log 标准库的代码都可以将日志库无缝切换到 Logrus。

- 支持七种日志级别：`Trace`、`Debug`、`Info`、`Warn`、`Error`、`Fatal`、`Panic`。

- 支持结构化日志记录（key-value 形式，容易被程序解析，如 JSON 格式），通过 Filed 机制进行结构化的日志记录。

- 支持自定义日志格式，内置两种格式 `JSONFormatter`（JSON 格式） 和 `TextFormatter`（文本格式），并允许用户通过实现 `Formatter` 接口来自定义日志格式。

- 支持可扩展的 Hooks 机制，可以为不同级别的日志添加 Hooks 将日志记录到不同位置，例如将 `Error`、`Fatal` 和 `Panic` 级别的错误日志发送到 logstash、kafka 等。

- 支持在控制台输出带有不同颜色的日志。

- 并发安全。

### 使用

#### 替代 Go log 标准库

我在[深入探究 Go log 标准库](https://jianghushinian.cn/2023/03/11/dive-into-the-go-log-standard-library/)一文中举过一个使用 Go log 标准库的简单[示例](https://jianghushinian.cn/2023/03/11/dive-into-the-go-log-standard-library/#%E4%BD%BF%E7%94%A8)，现在可以将其无缝切换到 Logrus，只需要把 `import "log"` 改成 `import log "github.com/sirupsen/logrus"` 即可实现：

```go
package main

// 替代 import "log"
import log "github.com/sirupsen/logrus"

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

执行以上代码，得到如下输出：

![日志输出](logrus.png)

虽然输出格式与使用 Go log 标准库表现略有不同，但程序执行并不会报错，说明二者完全兼容。

另外，根据截图可以发现不同级别的日志输出颜色也不相同。

#### 基本使用

Logrus 基本使用方式如下：

```go
package main

import (
	"os"

	"github.com/sirupsen/logrus"
)

func main() {
	logrus.Debug("Debug msg") // 不会输出，因为默认日志级别为 logrus.InfoLevel
	logrus.Debugf("%s msg", "Debug")
	logrus.Debugln("Debugln msg")
	logrus.Info("Info msg") // 会被输出
	logrus.Infof("%s msg", "Infof")
	logrus.Infoln("Infoln msg")
	logrus.SetFormatter(&logrus.TextFormatter{ // 输出格式 logfmt 风格
		DisableColors: true,
		FullTimestamp: true,
	})
	logrus.Warn("Warn msg")

	// 设置项
	logrus.SetOutput(os.Stdout)                  // 设置日志输出位置，一个实现 io.Writer 接口的对象，默认值为 os.Stderr
	logrus.SetLevel(logrus.TraceLevel)           // 设置日志级别，默认值为 logrus.InfoLevel
	logrus.SetFormatter(&logrus.JSONFormatter{}) // 设置日志输出格式，默认值为 logrus.TextFormatter
	logrus.SetReportCaller(true)                 // 设置日志是否记录被调用的位置，默认值为 false

	// 日志级别由低到高
	logrus.Trace("Trace msg")
	logrus.Debug("Debug msg")
	logrus.Info("Info msg")
	logrus.Warn("Warn msg")
	logrus.Warning("Warning msg") // 与 Warn 等价，内部会调用 Warn
	logrus.Error("Error msg")
	// logrus.Fatal("Fatal msg") // 输出日志后会调用 os.Exit(1)
	// logrus.Panic("Panic msg") // 输出日志后会调用 panic()

	// 日志字段
	logrus.WithFields(logrus.Fields{
		"animal": "walrus",
		"size":   10,
	}).Info("Info msg")
	logrus.WithField("omg", true).WithField("number", 122).Error("Error msg")
}
```

Logrus 默认日志级别为 `Info`，所以默认情况下 `logrus.Debug` 不会被输出。

Logrus 提供了 `JSONFormatter` 和 `TextFormatter` 来分别实现 JSON 和 Text 格式的日志输出，它们都实现了 `Formatter` 接口。除此之外，这里还有一个第三方实现的 `Formatter` [列表](https://github.com/sirupsen/logrus#formatters)可供选择，如果这些依然无法满足你的需求，则可以自己实现 `Formatter` 接口对象定制日志格式。

默认输出格式为 `TextFormatter`，可以通过修改 `TextFormatter` 的 `DisableColors` 属性为 `true` 来关闭控制台日志颜色，如果同时设置 `FullTimestamp` 属性为 `true`，则可以输出 [logfmt](https://pkg.go.dev/github.com/kr/logfmt) 风格的日志，除了记录日志消息内容，还会包含记录日志的时间和日志级别。

> 笔记：如果你对 `logfmt` 格式不熟悉，可以参考[此文](https://brandur.org/logfmt)进行了解。

Logrus 还提供了 `SetXxx` 来修改属性，其中 `SetReportCaller` 可以用来开启记录日志在文件中所在的位置（`file` 字段）和方法名（`func` 字段）。

Logrus 鼓励用户通过日志字段记录结构化日志，可以使用 `WithFields` 和 `WithField` 两种形式，并且可以链式调用 `logrus.WithField("omg", true).WithField("number", 122).Error("Error msg")`。这有别于 `logrus.Fatalf("Failed to send event %s to topic %s with key %d")` 这种纯文本形式，结构化日志有利于工具提取并分析日志。

执行以上代码，得到如下输出：

![日志输出](basic-use.png)

根据日志输出可以发现，前三行输出 `Info` 级别日志，并且带有颜色。

第四行使用 `TextFormatter` 输出了 logfmt 风格日志，并且从这行开始不再带有颜色，默认输出了 `time`、`level`、`msg` 三个字段。

从第五行开始，因为使用了 `JSONFormatter`，所以日志输出为 JSON 格式，并且还增加了 `file`、`func` 两个字段，分别记录了输出日志所在的文件和函数名，这是 `SetReportCaller(true)` 的作用。

#### 自定义 Logger

除了通过 `logrus.Info("Info msg")` 这种开箱即用的方式使用 Logrus 默认的 `Logger`，我们还可以自定义 `Logger`。

```go
package main

import (
	"io"
	"os"

	"github.com/sirupsen/logrus"
)

func main() {
	// 自定义 Logger
	log := logrus.New()

	// 同时写到多个输出
	w1 := os.Stdout
	w2, _ := os.OpenFile("demo.log", os.O_WRONLY|os.O_CREATE, 0644)
	log.SetOutput(io.MultiWriter(w1, w2)) // io.MultiWriter 返回一个 io.Writer 对象

	// Logger 对象的属性基本都是导出字段，可以通过 SetXxx 方法修改，也可以直接赋值修改
	log.SetFormatter(&logrus.JSONFormatter{})
	log.Formatter = &logrus.TextFormatter{
		DisableColors: true, // 关闭控制台颜色输出
		FieldMap: logrus.FieldMap{ // 允许用户自定义默认字段的键名
			logrus.FieldKeyMsg: "message",
		},
	}

	// 默认字段
	logger := log.WithFields(logrus.Fields{
		"omg":    true,
		"number": 122,
	})
	logger.Warn("Warn msg")
	logger.Error("Error msg")

	// 还可以继续添加日志字段
	logger.WithFields(logrus.Fields{
		"animal": "walrus",
		"size":   10,
	}).Info("Info msg")
}
```

可以通过 `logrus.New()` 创建新的 `Logger`，默认使用的 `std` `Logger` 对象也是这么创建的。`logrus.Info("Info msg")` 调用的是包级别的导出函数，但在 `Info` 函数内部会调用 `std.Info()` 方法，这个设计思路跟内置 log 标准库很像。

我们可以通过使用 `io.MultiWriter(w1, w2, ...)` 将日志同时输出到多个位置，`io.MultiWriter` 返回一个 `io.Writer` 对象。

用户可以通过 `Formatter` 中的 `FieldMap` 字段来对默认字段的键名进行重命名，如日志输出中默认有一个 `msg` 字段记录日志信息，这里将其修改为 `message`。

`FieldMap` 支持重命名的默认字段如下：

```go
const (
	defaultTimestampFormat = time.RFC3339
	FieldKeyMsg            = "msg"
	FieldKeyLevel          = "level"
	FieldKeyTime           = "time"
	FieldKeyLogrusError    = "logrus_error"
	FieldKeyFunc           = "func"
	FieldKeyFile           = "file"
)
```

另外，由于 `WithFields` 支持链式调用，那么我们就可以利用这个特点来创建一个带有默认字段的 `logger` 对象：

```go
logger := log.WithFields(logrus.Fields{
    "omg":    true,
    "number": 122,
})
```

接下来，只要使用 `logger.Xxx` 来记录日志，则每条日志都会带有默认字段。

执行以上代码，控制台和日志文件 `demo.log` 得到如下输出：

```log
time="2023-03-15T22:01:48+08:00" level=warning message="Warn msg" number=122 omg=true
time="2023-03-15T22:01:48+08:00" level=error message="Error msg" number=122 omg=true
time="2023-03-15T22:01:48+08:00" level=info message="Info msg" animal=walrus number=122 omg=true size=10
```

#### Hooks

Logrus 最令人心动的两个功能，一个是结构化日志，另一个就是 Hooks 了。

Hooks 为 Logrus 提供了极大的灵活性，通过 Hooks 可以实现各种扩展功能。比如可以通过 Hooks 实现：`Error` 以上级别日志发送邮件通知、重要日志告警、日志切割、程序优雅退出等，非常实用。

Logrus 提供了 `Hook` 接口，只要我们实现了这个接口，并将其注册到 Logrus 中，就可以使用 Hooks 的强大能力了。`Hook` 接口定义如下：

```go
type Hook interface {
	Levels() []Level
	Fire(*Entry) error
}
```

`Levels` 方法返回一个日志级别切片，Logrus 记录的日志级别如果存在于切片中，则会触发 Hooks，即调用 `Fire` 方法。

以下代码是 Hooks 功能使用示例：

```go
package main

import (
	"fmt"

	"github.com/orandin/lumberjackrus"
	"github.com/sirupsen/logrus"
)

type emailHook struct{}

func (hook *emailHook) Levels() []logrus.Level {
	return logrus.AllLevels // 所有日志级别都会执行 Fire 方法
}

func (hook *emailHook) Fire(entry *logrus.Entry) error {
	// 修改日志内容
	entry.Data["app"] = "email"
	// 发送邮件
	msg, _ := entry.String()
	fakeSendEmail(msg)
	return nil
}

func fakeSendEmail(msg string) {
	fmt.Printf("fakeSendEmail: %s", msg)
}

func newRotateHook() logrus.Hook {
	hook, _ := lumberjackrus.NewHook(
		&lumberjackrus.LogFile{ // 通用日志配置
			Filename:   "general.log",
			MaxSize:    100,
			MaxBackups: 1,
			MaxAge:     1,
			Compress:   false,
			LocalTime:  false,
		},
		logrus.InfoLevel,
		&logrus.TextFormatter{DisableColors: true},
		&lumberjackrus.LogFileOpts{ // 针对不同日志级别的配置
			logrus.TraceLevel: &lumberjackrus.LogFile{
				Filename: "trace.log",
			},
			logrus.ErrorLevel: &lumberjackrus.LogFile{
				Filename:   "error.log",
				MaxSize:    10,    // 日志文件在轮转之前的最大大小，默认 100 MB
				MaxBackups: 10,    // 保留旧日志文件的最大数量
				MaxAge:     10,    // 保留旧日志文件的最大天数
				Compress:   true,  // 是否使用 gzip 对日志文件进行压缩归档
				LocalTime:  false, // 是否使用本地时间，默认 UTC 时间
			},
		},
	)
	return hook
}

func main() {
	logrus.Trace("Trace msg") // not be written
	logrus.Warn("Warn msg")   // written in general.log
	logrus.Error("Error msg") // written in error.log
}

func init() {
	logrus.SetFormatter(&logrus.JSONFormatter{})
	logrus.SetLevel(logrus.InfoLevel) // default

	logrus.AddHook(&emailHook{})
	logrus.AddHook(newRotateHook())
}
```

这里实现了两个 Hooks，分别用来提供发送邮件和日志切割功能。

`emailHook` 实现了一个简单的发送邮件 `Hook`，其 `Levels` 方法返回 `logrus.AllLevels`，代表所有日志级别。`Fire` 方法接收一个 `*logrus.Entry` 参数，它包含了一条日志相关的所有信息，日志内容保存在 `entry.Data` Map 中，通过 `entry.String()` 能够获取日志的字符串格式内容。这里使用 `fakeSendEmail` 来模拟发送邮件。

`newRotateHook()` 利用第三方库 [lumberjackrus](https://github.com/orandin/lumberjackrus) 实现了日志切割和归档功能，并且能够将不同级别的日志输出到不同文件。`lumberjackrus` 是专门为 Logrus 而打造的文件日志 Hooks，其官方介绍为 `local filesystem hook for Logrus`。

执行以上代码，控制台得到如下输出：

```log
fakeSendEmail: {"app":"email","level":"warning","msg":"Warn msg","time":"2023-03-15T23:00:48+08:00"}
{"app":"email","level":"warning","msg":"Warn msg","time":"2023-03-15T23:00:48+08:00"}
fakeSendEmail: {"app":"email","level":"error","msg":"Error msg","time":"2023-03-15T23:00:48+08:00"}
{"app":"email","level":"error","msg":"Error msg","time":"2023-03-15T23:00:48+08:00"}
```

由于 `emailHook` 支持所有日志级别，所以每次输出日志，都会发送邮件，并且在 `fakeSendEmail` 中打印的消息内容会附加一个 `"app":"email"` 键值对。

`general.log` 日志文件得到如下输出：

```log
time="2023-03-15T23:00:48+08:00" level=warning msg="Warn msg" app=email
```

`error.log` 日志文件得到如下输出：

```log
time="2023-03-15T23:00:48+08:00" level=error msg="Error msg" app=email
```

通过 `lumberjackrus` 的能力，将不同级别日志分别输出到不同文件，并且如果日志达到 10MB 还会得到一个文件名类似 `error-2023-03-15T15-03-24.279.log.gz` 的压缩文件，这是由于为 `lumberjackrus.LogFile` 结构体的属性 `Compress` 设置为 `true` 导致的，将旧的日志文件归档可以节省日志文件所占用的磁盘空间。

可以发现，通过 Hooks 的强大能力，可以极大的拓宽 Logrus 的能力边界。如果你想自己实现一个日志库，同样建议支持 Hooks 功能。

> 笔记：
> Logrus 内置 Hooks 列表：https://github.com/sirupsen/logrus/tree/master/hooks
> 第三方开发的 Hooks 列表：https://github.com/sirupsen/logrus/wiki/Hooks

#### 处理不同环境

Logrus 并没有像 [zap](https://github.com/uber-go/zap) 那样提供现成的 API 来支持在不同的环境下使用（Production 和 Development），如果你想在生产和测试环境使用不同的格式输出日志，则需要通过代码判断在不同环境设置不同的 `Formatter` 来实现。示例如下：

```go
import (
  log "github.com/sirupsen/logrus"
)

func init() {
  if Environment == "production" {
    log.SetFormatter(&log.JSONFormatter{})
  } else {
    log.SetFormatter(&log.TextFormatter{})
  }
}
```

#### 测试

如果你的单元测试程序中需要测试日志内容，Logrus 提供了 `test.NewNullLogger` 日志记录器，它只会记录日志，不输出任何内容。使用示例如下：

```go
import(
  "testing"

  "github.com/sirupsen/logrus"
  "github.com/sirupsen/logrus/hooks/test"
  "github.com/stretchr/testify/assert"
)

func TestSomething(t*testing.T){
  logger, hook := test.NewNullLogger()
  logger.Error("Helloerror")

  assert.Equal(t, 1, len(hook.Entries))
  assert.Equal(t, logrus.ErrorLevel, hook.LastEntry().Level)
  assert.Equal(t, "Helloerror", hook.LastEntry().Message)

  hook.Reset()
  assert.Nil(t, hook.LastEntry())
}
```

### 总结

本文介绍了 Logrus 的基本特点，以及如何使用。

Logrus 完全兼容 log 标准库，所以可以实现无缝替换。其 API 设计思路跟 log 标准库的风格也有很多相似之处，都提供了一个默认的 `std` 日志对象达到开箱即用的效果。Logrus 最实用的两个功能，一个是支持结构化日志，一个是支持 Hooks 机制，这极大的提升了可用性和灵活性，也使得 Logrus 成为最受欢迎的 Go 日志库。

**参考**

> Logrus: https://github.com/sirupsen/logrus
> lumberjackrus: https://github.com/orandin/lumberjackrus
