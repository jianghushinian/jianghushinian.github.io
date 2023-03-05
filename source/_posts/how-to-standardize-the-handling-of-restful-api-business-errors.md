---
title: 如何规范 RESTful API 的业务错误处理
date: 2023-03-04 17:49:41
tags:
- Golang
- Web
categories:
- Web
---

现如今，主流的 Web API 都采用 RESTful 设计风格，对于接口返回的 HTTP 状态码和响应内容都有统一的规范。针对接口错误响应，一般都会返回一个 Code（错误码）和 Message（错误消息内容），通常错误码 Code 用来定位一个唯一的错误，错误消息 Message 用来展示错误信息。

本文就来详细介绍下，如何将 RESTful API 的错误处理进行规范化。

<!-- more -->

### 错误码

#### 为什么需要业务错误码

虽然 RESTful API 能够通过 HTTP 状态码来标记一个请求的成功或失败，但 HTTP 状态码作为一个通用的标准，并不能很好的表达业务错误。

比如一个 500 的错误响应，可能是由后端数据库连接异常引起的、也可能由内部代码逻辑错误引起，这些都无法通过 HTTP 状态码感知到，如果程序出现错误，不方便开发人员 Debug。

因此我们有必要设计一套用来标识业务错误的错误码，这有别于 HTTP 状态码，是跟系统具体业务息息相关的。

#### 错误码功能

在设计错误码之前，我们需要明确下错误码应该具备哪些属性，以满足业务需要。

- 错误码必须是唯一的。只有错误码是唯一的才方便在程序出错时快速定位问题，不然程序出错，返回错误码不唯一，想要根据错误码排查问题，就要针对这一错误码所表示的错误列表进行逐一排查。

- 错误码需要是可阅读的。意思是说，通过错误码，我们就能快速定位到是系统的哪个组件出现了错误，并且知道错误的类型，不然也谈不上叫「业务错误码」了。一个清晰可读的错误码在微服务系统中定位问题尤其有效。

- 通过错误码能够方便知道 HTTP 状态码。这一点往往容易被人忽略，不过我比较推荐这种做法，因为在 Review 代码时，通过返回错误码，就能很容易知道接口返回 HTTP 状态码，这不仅方便理解代码，更方便错误的统一处理。

#### 错误码设计

##### 错误码调研

错误码的设计我们可以参考业内使用量比较大的开放 API 设计，比较有代表性的是阿里云和新浪网的开放 API。

如以下是一个阿里云 ECS 接口错误的返回：

```json
{
	"RequestId": "5E571499-13C5-55E3-9EA6-DEFA0DBC85E4",
	"HostId": "ecs-cn-hangzhou.aliyuncs.com",
	"Code": "InvalidOperation.NotSupportedEndpoint",
	"Message": "The specified endpoint can't operate this region. Please use API DescribeRegions to get the appropriate endpoint, or upgrade your SDK to latest version.",
	"Recommend": "https://next.api.aliyun.com/troubleshoot?q=InvalidOperation.NotSupportedEndpoint&product=Ecs"
}
```

可以发现，Code 和 Message 都为字符串类型，并且还有 RequestId（当前请求唯一标识）、HostId（Host 唯一标识）、Recommend（错误诊断地址），可以说这个错误信息非常全面了。

再来看下新浪网开放 API 错误返回结果的设计：

```json
{
	"request": "/statuses/home_timeline.json",
	"error_code": "20502",
	"error": "Need you follow uid."
}
```

相比阿里云，新浪网的错误返回更简洁一些。其中 request 为请求路径，error_code 即为错误码 Code，error 则表示错误信息 Message。

错误代码 `20502` 说明如下：

|2|05|02|
|:-|:-|:-|
|服务级错误（1为系统级错误）|服务模块代码|具体错误代码|

新浪网的错误码为数字类型的字符串，相比阿里云的错误码要简短不少，并且对程序更加友好，也是我个人更推荐的设计。

##### 业务错误码

结合市面上这些优秀的开放 API 错误码设计，以及我在实际开发中的工作总结，我设计的错误码规则如下：

- 业务错误码由 8 位纯数字组成，类型为 `int`。

- 业务错误码示例格式：`40001002`。

- 错误码说明：

|1-3 位|4-5 位|6-8 位|
|:-|:-|:-|
|400|01|002|
|HTTP 状态码|组件编号|组件内部错误码|

错误码设计为纯数字主要是为了程序中使用起来更加方便，比如根据错误码计算 HTTP 状态码，只需要通过简单的数学取模计算就能做到。

使用两位数字来标记不同组件，最多能表示 99 个组件，即使项目全部采用微服务开发，一般来说也是足够用的。

最后三位代表组件内部错误码，最多能表示 1000 个错误。其实通常来说一个组件内部是用不上这么多错误的，如果组件较小，完全可以设计成两位数字。

另外，有些厂商中还会设计一些公共的错误码，可以称为「全局错误码」，这些错误码在各组件间通用，以此来减少定义重复错误。在我们的错误码设计中，可以将组件编号为 `00` 的标记为全局错误码，其他组件编号从 `01` 开始。

##### 错误格式

有了错误码，还需要定义错误响应格式，设计一个标准的 API 错误响应格式如下：

```json
{
	"code": 50000000,
	"message": "系统错误",
	"reference": "https://github.com/jianghushinian/gokit/tree/main/errors"
}
```

`code` 即为错误码，`message` 为错误信息，`reference` 则是错误文档地址，用来告知用户如何解决这个错误，对标的是阿里云错误响应中的 `Recommend` 字段。

##### 错误码实现

因为每一个错误码和错误信息以及错误文档地址都是一一对应的，所以我们需要一个对象来保存这些信息，在 Go 中可以使用结构体。

可以设计如下结构体：

```go
type apiCode struct {
	code int
	msg  string
	ref  string
}
```

这是一个私有结构体，外部项目要想使用，则需要一个构造函数：

```go
func NewAPICode(code int, message string, reference ...string) APICoder {
	ref := ""
	if len(reference) > 0 {
		ref = reference[0]
	}
	return &apiCode{
		code: code,
		msg:  message,
		ref:  ref,
	}
}
```

其中 `reference` 被设计为可变参数，如果不传则默认为空。

`NewAPICode` 返回值 `APICoder` 是一个接口，这在 Go 中是一种惯用做法。通过接口可以解耦，方便依赖 `apiCode` 的代码编写测试，用户可以对 APICoder 进行 Mock；另一方面，我们稍后会为 `apiCode` 实现对应的错误包，使用接口来表示错误码可以方便用户定义自己的 `apiCode` 类型。

为了便于使用，`apiCode` 提供了如下几个能力：

```go
func (a *apiCode) Code() int {
	return a.code
}

func (a *apiCode) Message() string {
	return a.msg
}

func (a *apiCode) Reference() string {
	return a.ref
}

func (a *apiCode) HTTPStatus() int {
	v := a.Code()
	for v >= 1000 {
		v /= 10
	}
	return v
}
```

至此 `APICoder` 接口接口的定义也就有了：

```go
type APICoder interface {
	Code() int
	Message() string
	Reference() string
	HTTPStatus() int
}
```

`apiCode` 则实现了 `APICoder` 接口。

现在我们可以通过如下方式创建错误码结构体对象：

```go
var (
	CodeBadRequest   = NewAPICode(40001001, "请求不合法")
	CodeUnknownError = NewAPICode(50001001, "系统错误", "https://github.com/jianghushinian/gokit/tree/main/errors")
)
```

### 错误包

设计好了错误码，并不能直接使用，我们还需要一个与之配套的错误包来简化错误码的使用。

#### 错误包功能

- 错误包要能够完美支持上面设计的错误码。所以需要使用 `APICoder` 来构造错误对象。

- 错误包应该能够查看原始错误原因。这就需要实现 `Unwrap` 方法，`Wrap/Unwrap` 方法是在 Go 1.13 中被加入进 `errors` 包的，目的是能够处理嵌套错误。

- 错误包应该能够支持对内对外展示不同信息。这就需要实现 `Format` 方法，根据需要可以将错误格式化成不同输出。

- 错误包应该能够支持展示堆栈信息。这对 Debug 来说相当重要，也是 Go 自带的 `errors` 包不足的地方。

- 为了方便在日志中记录结构化错误信息，错误包还要能够支持 JSON 序列化。这需要实现 `MarshalJSON/UnmarshalJSON` 两个方法。

#### 错误包设计

一个错误对象结构体设计如下：

```go
type apiError struct {
	coder APICoder
	cause error
	*stack
}
```

其中 `coder` 用来保存实现了 `APICoder` 接口的对象，`cause` 用来记录错误原因，`stack` 用来展示错误堆栈。

错误对象的构造函数如下：

```go
var WrapC = NewAPIError

func NewAPIError(coder APICoder, cause ...error) error {
	var c error
	if len(cause) > 0 {
		c = cause[0]
	}
	return &apiError{
		coder: coder,
		cause: c,
		stack: callers(),
	}
}
```

`NewAPIError` 通过 `APICoder` 来创建错误对象，第二个参数为一个可选的错误原因。

其实构造一个错误对象也就是对一个错误进行 `Wrap` 的过程，所以我还为构造函数 `NewAPIError` 定义了一个别名 `WrapC`，表示使用错误码将一个错误包装成一个新的错误。

一个错误对象必须要实现 `Error` 方法：

```go
func (a *apiError) Error() string {
	return fmt.Sprintf("[%d] - %s", a.coder.Code(), a.coder.Message())
}
```

默认情况下，获取到的错误内容只包含错误码 Code 和错误信息 Message。

为了方便获取被包装错误的原始错误，还要实现 `Unwrap` 方法：

```go
func (a *apiError) Unwrap() error {
	return a.cause
}
```

为了能在打印或写入日志时展示不同信息，则要实现 `Format` 方法：

```go
func (a *apiError) Format(s fmt.State, verb rune) {
	switch verb {
	case 'v':
		if s.Flag('+') {
			str := a.Error()
			if a.Unwrap() != nil {
				str += " " + a.Unwrap().Error()
			}
			_, _ = io.WriteString(s, str)
			a.stack.Format(s, verb)
			return
		}
		if s.Flag('#') {
			cause := ""
			if a.Unwrap() != nil {
				cause = a.Unwrap().Error()
			}
			data, _ := json.Marshal(errorMessage{
				Code:      a.coder.Code(),
				Message:   a.coder.Message(),
				Reference: a.coder.Reference(),
				Cause:     cause,
				Stack:     fmt.Sprintf("%+v", a.stack),
			})
			_, _ = io.WriteString(s, string(data))
			return
		}
		fallthrough
	case 's':
		_, _ = io.WriteString(s, a.Error())
	case 'q':
		_, _ = fmt.Fprintf(s, "%q", a.Error())
	}
}
```

`Format` 方法能够支持在使用 `fmt.Printf("%s", apiError)` 格式化输出时打印定制化的信息。

`Format` 方法支持的不同格式输出如下：

|格式占位符|输出信息|
|:------:|:------|
|%s|错误码、错误信息|
|%v|错误码、错误信息，与 %s 等价|
|%+v|错误码、错误信息、错误原因、错误堆栈|
|%#v|JSON 格式的 错误码、错误信息、错误文档地址、错误原因、错误堆栈|
|%q|在 错误码、错误信息 外层增加了一个双引号|

这些错误格式基本上能满足所有业务开发中的需求了，如果还有其他格式需要，则可以在此基础上进一步开发 `Format` 方法。

用来进行 JSON 序列化和反序列化的 `MarshalJSON/UnmarshalJSON` 方法实现如下：

```go
func (a *apiError) MarshalJSON() ([]byte, error) {
	return json.Marshal(&errorMessage{
		Code:      a.coder.Code(),
		Message:   a.coder.Message(),
		Reference: a.coder.Reference(),
	})
}

func (a *apiError) UnmarshalJSON(data []byte) error {
	e := &errorMessage{}
	if err := json.Unmarshal(data, e); err != nil {
		return err
	}
	a.coder = NewAPICode(e.Code, e.Message, e.Reference)
	return nil
}

type errorMessage struct {
	Code      int    `json:"code"`
	Message   string `json:"message"`
	Reference string `json:"reference,omitempty"`
	Cause     string `json:"cause,omitempty"`
	Stack     string `json:"stack,omitempty"`
}
```

为了不对外部暴露敏感信息，对外的 HTTP API 只会返回 `Code`、`Message`、`Reference`（可选）三个字段，对内需要额外展示错误原因以及错误堆栈。所以 `errorMessage` 中 `Reference`、`Cause`、`Stack` 字段都带有 `omitempty` 属性，这样在 `MarshalJSON` 时只会序列化 `Code`、`Message`、`Reference` 这三个字段。

至此，我们就实现了错误包的设计。

### 错误码及错误包的使用

#### 使用示例

通过上面的讲解，我们了解了错误码和错误包的设计规范，接下来看看如何使用它们。这里以错误码及错误包在 [Gin](https://github.com/gin-gonic/gin) 中的使用为例进行讲解。

使用 Gin 构建一个简单的 Web Server 如下：

```go
package main

import (
	"errors"
	"fmt"
	"strconv"

	"github.com/gin-gonic/gin"

	apierr "github.com/jianghushinian/gokit/errors"
)

var (
	ErrAccountNotFound = errors.New("account not found")
	ErrDatabase        = errors.New("database error")
)

var (
	CodeBadRequest   = NewAPICode(40001001, "请求不合法")
	CodeNotFound     = NewAPICode(40401001, "资源未找到")
	CodeUnknownError = NewAPICode(50001001, "系统错误", "https://github.com/jianghushinian/gokit/tree/main/errors")
)

type Account struct {
	ID   int    `json:"id"`
	Name string `json:"name"`
}

func AccountOne(id int) (*Account, error) {
	for _, v := range accounts {
		if id == v.ID {
			return &v, nil
		}
	}

	// 模拟返回数据库错误
	if id == 500 {
		return nil, ErrDatabase
	}
	return nil, ErrAccountNotFound
}

var accounts = []Account{
	{ID: 1, Name: "account_1"},
	{ID: 2, Name: "account_2"},
	{ID: 3, Name: "account_3"},
}

func ShowAccount(c *gin.Context) {
	id := c.Param("id")
	aid, err := strconv.Atoi(id)
	if err != nil {
		// 将 errors 包装成 APIError 返回
		ResponseError(c, apierr.WrapC(CodeBadRequest, err))
		return
	}

	account, err := AccountOne(aid)
	if err != nil {
		switch {
		case errors.Is(err, ErrAccountNotFound):
			err = apierr.NewAPIError(CodeNotFound, err)
		case errors.Is(err, ErrDatabase):
			err = apierr.NewAPIError(CodeUnknownError, fmt.Errorf("account %d: %w", aid, err))
		}
		ResponseError(c, err)
		return
	}
	ResponseOK(c, account)
}

func main() {
	r := gin.Default()

	r.GET("/accounts/:id", ShowAccount)

	if err := r.Run(":8080"); err != nil {
		panic(err)
	}
}
```

在这个 Web Server 中定义了一个 `ShowAccount` 函数，用来处理获取账号逻辑，在 `ShowAccount` 内部程序执行成功返回 `ResponseOK(c, account)`，失败则返回 `ResponseError(c, err)`。

在处理返回失败的响应时，都会通过 `apierr.WrapC` 或 `apierr.NewAPIError` 将底层函数返回的初始错误进行一层包装，根据错误级别，包装成不同的错误码进行返回。

其中 `ResponseOK` 和 `ResponseError` 定义如下：

```go
func ResponseOK(c *gin.Context, spec interface{}) {
	if spec == nil {
		c.Status(http.StatusNoContent)
		return
	}
	c.JSON(http.StatusOK, spec)
}

func ResponseError(c *gin.Context, err error) {
	log(err)
	e := apierr.ParseCoder(err)
	httpStatus := e.HTTPStatus()
	if httpStatus >= 500 {
		// send error msg to email/feishu/sentry...
		go fakeSendErrorEmail(err)
	}
	c.AbortWithStatusJSON(httpStatus, err)
}

// log 打印错误日志，输出堆栈
func log(err error) {
	fmt.Println("========== log start ==========")
	fmt.Printf("%+v\n", err)
	fmt.Println("========== log end ==========")
}

// fakeSendErrorEmail 模拟将错误信息发送到邮件，JSON 格式
func fakeSendErrorEmail(err error) {
	fmt.Println("========== error start ==========")
	fmt.Printf("%#v\n", err)
	fmt.Println("========== error end ==========")
}
```

`ResponseOK` 其实就是 Gin 框架的正常返回，`ResponseError` 则专门用来处理并返回 API 错误。

在 `ResponseError` 中首先通过 `log(err)` 来记录错误日志，在其内部使用 `fmt.Printf("%+v\n", err)` 进行打印。

之后我们还对 HTTP 状态码进行了判断，大于 500 的错误将会发送邮件通知，这里使用 `fmt.Printf("%#v\n", err)` 进行模拟。

其中 `apierr.ParseCoder(err)` 能够从一个错误对象中获取到实现了 `APICoder` 的错误码对象，实现如下：

```go
func ParseCoder(err error) APICoder {
	for {
		if e, ok := err.(interface {
			Coder() APICoder
		}); ok {
			return e.Coder()
		}
		if errors.Unwrap(err) == nil {
			return CodeUnknownError
		}
		err = errors.Unwrap(err)
	}
}
```

这样，我们就能够通过一个简单的 Web Server 示例程序来演示如何使用错误码和错误包了。

可以通过 `go run main.go` 启动这个 Web Server。

先来看下在这个 Web Server 中一个正常的返回结果是什么样，使用 cURL 来发送一个请求：`curl http://localhost:8080/accounts/1`。

客户端得到如下响应结果：

```json
{
	"id": 1,
	"name": "account_1"
}
```

服务端打印正常的请求日志：

![Server Log](web-server.png)

再来测试下请求一个不存在的账号：`curl http://localhost:8080/accounts/12`。

客户端得到如下响应结果：

```json
{
	"code": 40401001,
	"message": "资源未找到"
}
```

返回结果中没有 `reference` 字段，是因为对于 `reference` 为空的情况，在 JSON 序列化过程中会被隐藏。

服务端打印的错误日志如下：

```text
========== log start ==========
[40401001] - 资源未找到 account not found
main.ShowAccount
        /app/errors/examples/main.go:56
github.com/gin-gonic/gin.(*Context).Next
        /go/pkg/mod/github.com/gin-gonic/gin@v1.9.0/context.go:174
github.com/gin-gonic/gin.CustomRecoveryWithWriter.func1
        /go/pkg/mod/github.com/gin-gonic/gin@v1.9.0/recovery.go:102
github.com/gin-gonic/gin.(*Context).Next
        /go/pkg/mod/github.com/gin-gonic/gin@v1.9.0/context.go:174
github.com/gin-gonic/gin.LoggerWithConfig.func1
        /go/pkg/mod/github.com/gin-gonic/gin@v1.9.0/logger.go:240
github.com/gin-gonic/gin.(*Context).Next
        /go/pkg/mod/github.com/gin-gonic/gin@v1.9.0/context.go:174
github.com/gin-gonic/gin.(*Engine).handleHTTPRequest
        /go/pkg/mod/github.com/gin-gonic/gin@v1.9.0/gin.go:620
github.com/gin-gonic/gin.(*Engine).ServeHTTP
        /go/pkg/mod/github.com/gin-gonic/gin@v1.9.0/gin.go:576
net/http.serverHandler.ServeHTTP
        /usr/local/go/src/net/http/server.go:2947
net/http.(*conn).serve
        /usr/local/go/src/net/http/server.go:1991
runtime.goexit
        /usr/local/go/src/runtime/asm_arm64.s:1165
========== log end ==========
```

可以发现，错误日志中不仅打印了错误码（[40401001]）和错误信息（资源未找到），还打印了错误原因（account not found）以及下面的错误堆栈。

如此清晰的错误日志得益于我们实现的 `Format` 函数的强大功能。

现在再来触发一个 HTTP 状态码为 500 的错误响应：`curl http://localhost:8080/accounts/500`。

客户端得到如下响应结果：

```json
{
	"code": 50001001,
	"message": "系统错误",
	"reference": "https://github.com/jianghushinian/gokit/tree/main/errors"
}
```

这次得到一个带有 `reference` 字段的完整错误响应。

服务端打印的错误日志如下：

```text
========== log start ==========
[50001001] - 系统错误 account 500: database error
main.ShowAccount
        /app/errors/examples/main.go:58
github.com/gin-gonic/gin.(*Context).Next
        /go/pkg/mod/github.com/gin-gonic/gin@v1.9.0/context.go:174
github.com/gin-gonic/gin.CustomRecoveryWithWriter.func1
        /go/pkg/mod/github.com/gin-gonic/gin@v1.9.0/recovery.go:102
github.com/gin-gonic/gin.(*Context).Next
        /go/pkg/mod/github.com/gin-gonic/gin@v1.9.0/context.go:174
github.com/gin-gonic/gin.LoggerWithConfig.func1
        /go/pkg/mod/github.com/gin-gonic/gin@v1.9.0/logger.go:240
github.com/gin-gonic/gin.(*Context).Next
        /go/pkg/mod/github.com/gin-gonic/gin@v1.9.0/context.go:174
github.com/gin-gonic/gin.(*Engine).handleHTTPRequest
        /go/pkg/mod/github.com/gin-gonic/gin@v1.9.0/gin.go:620
github.com/gin-gonic/gin.(*Engine).ServeHTTP
        /go/pkg/mod/github.com/gin-gonic/gin@v1.9.0/gin.go:576
net/http.serverHandler.ServeHTTP
        /usr/local/go/src/net/http/server.go:2947
net/http.(*conn).serve
        /usr/local/go/src/net/http/server.go:1991
runtime.goexit
        /usr/local/go/src/runtime/asm_arm64.s:1165
========== log end ==========
[GIN] 2023/03/05 - 02:02:28 | 500 |     426.292µs |       127.0.0.1 | GET      "/accounts/500"
========== error start ==========
{"code":50001001,"message":"系统错误","reference":"https://github.com/jianghushinian/gokit/tree/main/errors","cause":"account 500: database error","stack":"\nmain.ShowAccount\n\t/app/errors/examples/main.go:58\ngithub.com/gin-gonic/gin.(*Context).Next\n\t/go/pkg/mod/github.com/gin-gonic/gin@v1.9.0/context.go:174\ngithub.com/gin-gonic/gin.CustomRecoveryWithWriter.func1\n\t/go/pkg/mod/github.com/gin-gonic/gin@v1.9.0/recovery.go:102\ngithub.com/gin-gonic/gin.(*Context).Next\n\t/go/pkg/mod/github.com/gin-gonic/gin@v1.9.0/context.go:174\ngithub.com/gin-gonic/gin.LoggerWithConfig.func1\n\t/go/pkg/mod/github.com/gin-gonic/gin@v1.9.0/logger.go:240\ngithub.com/gin-gonic/gin.(*Context).Next\n\t/go/pkg/mod/github.com/gin-gonic/gin@v1.9.0/context.go:174\ngithub.com/gin-gonic/gin.(*Engine).handleHTTPRequest\n\t/go/pkg/mod/github.com/gin-gonic/gin@v1.9.0/gin.go:620\ngithub.com/gin-gonic/gin.(*Engine).ServeHTTP\n\t/go/pkg/mod/github.com/gin-gonic/gin@v1.9.0/gin.go:576\nnet/http.serverHandler.ServeHTTP\n\t/usr/local/go/src/net/http/server.go:2947\nnet/http.(*conn).serve\n\t/usr/local/go/src/net/http/server.go:1991\nruntime.goexit\n\t/usr/local/go/src/runtime/asm_arm64.s:1165"}
========== error end ==========
```

这一次除了 `log` 函数打印的日志，还能看到 `fakeSendErrorEmail` 函数打印的日志，正是一个 JSON 格式的结构化日志。

以上便是我们设计的错误码及错误包在实际开发场景中的应用。

#### 使用建议

根据我的经验，总结了一些错误码及错误包的使用建议，现在将其分享给你。

##### 使用尽量少的 HTTP 状态码

HTTP 状态码大概分为 5 大类，分别是 1XX、2XX、3XX、4XX、5XX。根据我的实际工作经验，我们并不会使用全部的状态码，最常用的状态码不超过 10 个。

所以即使我们设计的业务错误码支持携带 HTTP 状态码，但也不推荐使用过多的 HTTP 状态码，以免加重前端工作量。

推荐在错误码中使用的 HTTP 状态码如下：

- 400: 请求不合法
- 401: 认证失败
- 403: 授权失败
- 404: 资源未找到
- 500: 系统错误

其中 4XX 代表客户端错误，而如果是服务端错误，则统一使用 500 状态码，具体错误原因可以通过业务错误码定位。

##### 使用中间件来记录错误日志

由于我们设计的错误包支持 `Unwrap` 操作，所以建议出现错误时的处理流程如下：

1. 最底层代码遇到错误时通过 `errors.New/fmt.Errorf` 来创建一个错误对象，然后将错误返回（可选择性的记录一条日志）。

```go
func Query(id int) (obj, error) {
    // do something
    return nil, fmt.Errorf("%d not found", id)
}
```

2. 中间过程中处理函数遇到下层函数返回的错误，不做任何额外处理，直接将其向上层返回。

```go
if err != nil {
    return err
}
```

3. 在处理用户请求的 Handler 函数中（如 `ShowAccount`）通过 `apierr.WrapC` 将错误包装成一个 `APIError` 返回。

```go
if err != nil {
    return apierr.WrapC(CodeNotFound, err)
}
```

4. 最上层代码通过在框架层面实现的中间件（如实现一个 after hook middleware）来统一处理错误，打印完整错误日志、发送邮件提醒等，并将安全的错误信息返回给前端。如我们实现的 `ResponseError` 函数功能。

### 总结

本篇文章讲解了如何设计一个规范的错误码以及与之配套的错误包。

我参考了一些开源的 API 错误码设计方案，并结合我自己的实际工作经验，给出了我认为比较合理的错误码设计方案。

同时也针对这个错误码方案，设计了一个配套的错误包，来简化使用过程，并给出了我的一些使用建议。

错误包中记录错误堆栈部分的代码参考了 [pkg/errors](https://github.com/pkg/errors) 包实现，感兴趣的同学可以点击进去进行进一步学习。

本文完整代码实现我放在了 [Github](https://github.com/jianghushinian/gokit/tree/main/errors) 上，供你参考使用。

希望本文对你有所启发，如果你有更好的错误码设计方案，欢迎一起交流讨论。

**参考**

> 阿里云 ECS 错误码：https://next.api.aliyun.com/document/Ecs/2014-05-26/errorCode
> 腾讯云服务器错误码：https://cloud.tencent.com/document/api/213/30435
> 新浪错误码：https://open.weibo.com/wiki/Error_code
> pkg/errors：https://github.com/pkg/errors
> 错误包实现：https://github.com/jianghushinian/gokit/tree/main/errors
