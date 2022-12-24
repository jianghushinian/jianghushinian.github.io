---
title: Golang 常见设计模式之装饰模式
date: 2022-02-28 08:57:42
tags:
- Golang
categories:
- Golang
---

熟悉 Python 的同学想必对装饰模式都不会太陌生，Python 从语法上原生支持装饰器，大大提高了装饰模式在 Python 中的应用。而在 Go 语言中，虽然装饰模式没有像 Python 中应用那么广泛，但也有其用武之地，这篇文章我们就来一起看下装饰模式在 Go 语言中的应用。

<!-- more -->

### 简单装饰器

首先我们编写一个简单的 `hello` 函数：

```go
package main

import "fmt"

func hello() {
	fmt.Println("Hello World!")
}

func main() {
	hello()
}
```

上面代码执行后输出 `Hello World!`。

现在我们想在打印 `Hello World!` 前后各加一行日志，最直接的实现方式如下：

```go
package main

import "fmt"

func hello() {
	fmt.Println("before")
	fmt.Println("Hello World!")
	fmt.Println("after")
}

func main() {
	hello()
}
```

代码执行后输出：

```
before
Hello World!
after
```

更好的实现方式是单独编写一个 `logger` 函数，专门用来打印日志：

```go
package main

import "fmt"

func logger(f func()) func() {
	return func() {
		fmt.Println("before")
		f()
		fmt.Println("after")
	}
}

func hello() {
	fmt.Println("Hello World!")
}

func main() {
	hello := logger(hello)
	hello()
}
```

`logger` 函数接收一个函数，并且返回一个函数，而且参数和返回值的函数签名同 `hello` 函数一样。我们将原来调用 `hello()` 的地方改成：

```go
hello := logger(hello)
hello()
```

就实现了 `logger` 函数对 `hello` 函数的包装，执行后的打印结果仍为：

```
before
Hello World!
after
```

这样我们就以一种更加优雅的方式，实现了给 `hello` 函数增加日志的功能，因为这个 `logger` 函数不仅可以用于 `hello`，还可以用于其他任何与 `hello` 函数有着同样签名的函数。

`logger` 函数也就是我们在 Python 中经常使用的装饰器，如果按照 Python 中装饰器的写法，我们可以这样做：

```go
package main

import "fmt"

func logger(f func()) func() {
	return func() {
		fmt.Println("before")
		f()
		fmt.Println("after")
	}
}

// 给 hello 函数打上 logger 装饰器
@logger
func hello() {
	fmt.Println("Hello World!")
}

func main() {
	// hello 函数调用方式不变
	hello()
}
```

但很遗憾，上面的程序无法通过编译，Go 语言目前还没有像 Python 语言一样从语法层面提供对装饰器语法糖的支持。

### 装饰器实现中间件

尽管 Go 语言中装饰器的写法不如 Python 语言精简，但有一个场景的确非常适用，那就是 Web 开发场景中的中间件组件。

如果你用过 `Gin` Web 框架，那么应该很容易能看懂如下代码：

```go
package main

import "github.com/gin-gonic/gin"

func main() {
	r := gin.New()

	// 使用中间件
	r.Use(gin.Logger(), gin.Recovery())

	r.GET("/ping", func(c *gin.Context) {
		c.JSON(200, gin.H{
			"message": "pong",
		})
	})
	_ = r.Run(":8888")
}
```

在 `Gin` 框架中可以通过 `r.Use(middlewares...)` 的方式给路由增加非常多的中间件，这样我们就能够很方便的拦截路由处理函数，并在其前后分别做一些处理逻辑。如上面示例中使用 `gin.Logger()` 增加日志，使用 `gin.Recovery()` 来处理 `panic` 异常。

`Gin` 框架的中间件正是使用装饰模式来实现的，我们可以借用 Go 语言自带的 `http` 库来简单模拟下：

```go
package main

import (
	"fmt"
	"net/http"
)

func loggerMiddleware(f http.HandlerFunc) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		fmt.Println("before")
		f(w, r)
		fmt.Println("after")
	}
}

func authMiddleware(f http.HandlerFunc) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		if token := r.Header.Get("token"); token != "fake_token" {
			_, _ = w.Write([]byte("unauthorized\n"))
			return
		}
		f(w, r)
	}
}

func handleHello(w http.ResponseWriter, r *http.Request) {
	fmt.Println("handle hello")
	_, _ = w.Write([]byte("Hello World!\n"))
}

func main() {
	http.HandleFunc("/hello", authMiddleware(loggerMiddleware(handleHello)))
	fmt.Println(http.ListenAndServe(":8888", nil))
}
```

这是一个简单的 Web Server 程序，其监听 `8888` 端口，当访问 `/hello` 路由时会进入 `handleHello` 函数逻辑。

我们分别使用 `loggerMiddleware`、`authMiddleware` 函数对 `handleHello` 进行了包装，使其支持打印访问日志和认证校验功能。假如我们还有其他中间件拦截功能需要加入，就可以这么无限包装下去。

启动这个 Server 来验证下装饰器：

![request](request.png)

![server_log](server_log.png)

可以对结果进行简单分析，第一次请求 `/hello` 接口时，由于没有携带认证 `token`，收到了 `unauthorized` 响应；第二次请求时携带了 `token`，得到响应 `Hello World!`，并且后台程序打印如下日志：

```
before
handle hello
after
```

说明中间件执行顺序是先由外向内进入，再由内向外返回。而这种一层一层包装处理逻辑的模型有一个非常形象的名字，叫作洋葱模型。

![onion](onion.png)

这个名字可以说非常贴切了。但我们用洋葱模型实现的中间件，相比于 `Gin` 框架的中间件写法上还差点意思，这种一层层包裹函数的写法不如 `Gin` 框架提供的 `r.Use(middlewares...)` 写法直观。

如果你去看 `Gin` 框架源码，就会发现它的中间件和 `handler` 处理函数实际上会被一起聚合到路由节点的 `handlers` 属性中，而这个 `handlers` 属性其实就是一个 `HandlerFunc` 类型切片。对应到咱们用 `http` 标准库实现的 Web Server 中，就是满足 `func(ResponseWriter, *Request)` 类型的 `handler` 切片。当路由接口被调用时，`Gin` 框架就会依次执行 `handlers` 切片中的所有函数，整个中间件的调用流程就像一条流水线一样依次调用，再依次返回。而这种思想也有一个形象的名字，叫作流水线（Pipeline）。

![pipeline](pipeline.png)

接下来我们要做的就是将 `handleHello` 和两个中间件 `loggerMiddleware`、`authMiddleware` 聚合到一起，同样形成一个 Pipeline。

```go
package main

import (
	"fmt"
	"net/http"
)

func authMiddleware(f http.HandlerFunc) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		if token := r.Header.Get("token"); token != "fake_token" {
			_, _ = w.Write([]byte("unauthorized\n"))
			return
		}
		f(w, r)
	}
}

func loggerMiddleware(f http.HandlerFunc) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		fmt.Println("before")
		f(w, r)
		fmt.Println("after")
	}
}

type handler func(http.HandlerFunc) http.HandlerFunc

// 聚合 handler 和 middleware
func pipelineHandlers(h http.HandlerFunc, hs ...handler) http.HandlerFunc {
	for i := range hs {
		h = hs[i](h)
	}
	return h
}

func handleHello(w http.ResponseWriter, r *http.Request) {
	fmt.Println("handle hello")
	_, _ = w.Write([]byte("Hello World!\n"))
}

func main() {
	http.HandleFunc("/hello", pipelineHandlers(handleHello, loggerMiddleware, authMiddleware))
	fmt.Println(http.ListenAndServe(":8888", nil))
}
```

我们借用 `pipelineHandlers` 函数将 `handler` 和 `middleware` 聚合到一起，实现了让这个简单的 Web Server 中间件用法跟 `Gin` 框架用法相似的效果。

再次启动 Server 进行验证：

![request](pipeline_request.png)

![server_log](pipeline_server_log.png)

改造成功，跟之前使用洋葱模型写法的结果如出一辙。

### 总结

在简单介绍了 Go 语言中如何实现装饰模式后，我们通过对一个 Web Server 程序中间件的讲解，学习了装饰模式在 Go 语言中的应用。因为 Go 语言是静态类型语言，不像 Python 那般灵活，所以在实现上要多费一点力气，并且 Go 语言实现的装饰器由于有类型上的限制，也不如 Python 装饰器那般通用，但仍有用武之地。

尽管我们最终实现的 `pipelineHandlers` 不如 `Gin` 框架中间件强大，比如不能延迟调用，通过 `c.Next()` 控制中间件调用流等，但通过这个简单的示例，相信对你接下来深入学习 `Gin` 框架会有所帮助。

