---
title: 在 Go 语言单元测试中如何解决 HTTP 网络依赖问题
date: 2023-07-15 20:57:35
tags:
- Go
- Test
categories:
- Go
---

在开发 Web 应用程序时，确保 HTTP 功能的正确性是至关重要的。然而，由于 Web 应用程序通常涉及到与外部依赖的交互，编写 HTTP 请求和响应的有效测试变得具有挑战性。在进行单元测试时，我们必须思考如何解决被测程序的外部依赖问题。

因此，在 Go 语言中，我们需要找到一种可靠的方法来测试 HTTP 请求和响应。本文将探讨在 Go 中进行 HTTP 应用测试时，如何解决应用程序的依赖问题，以确保我们能够编写出可靠的测试用例。

<!-- more -->

### HTTP Server 测试

首先，我们来看下，站在 HTTP Server 端的角度，如何编写应用程序的测试代码。

假设我们有一个 HTTP Server 对外提供服务，代码如下：

```go
package main

import (
	"encoding/json"
	"fmt"
	"io"
	"net/http"
	"strconv"

	"github.com/julienschmidt/httprouter"
)

type User struct {
	ID   int    `json:"id"`
	Name string `json:"name"`
}

var users = []User{
	{ID: 1, Name: "user1"},
}

func CreateUserHandler(w http.ResponseWriter, r *http.Request, _ httprouter.Params) {
	...
}

func GetUserHandler(w http.ResponseWriter, r *http.Request, ps httprouter.Params) {
	...
}

func setupRouter() *httprouter.Router {
	router := httprouter.New()
	router.POST("/users", CreateUserHandler)
	router.GET("/users/:id", GetUserHandler)
	return router
}

func main() {
	router := setupRouter()
	_ = http.ListenAndServe(":8000", router)
}
```

这个服务监听 `8000` 端口，分别提供了两个 HTTP 接口：

`POST /users` 用来创建用户。

`GET /users/:id` 用来获取指定 ID 对应的用户信息。

为了保证业务的正确性，我们需要对 `CreateUserHandler` 和 `GetUserHandler` 这两个 Handler 进行单元测试。

我们先来看下用于创建用户的 `CreateUserHandler` 函数是如何定义的：

```go
func CreateUserHandler(w http.ResponseWriter, r *http.Request, _ httprouter.Params) {
	w.Header().Set("Content-Type", "application/json")

	body, err := io.ReadAll(r.Body)
	if err != nil {
		w.WriteHeader(http.StatusBadRequest)
		_, _ = fmt.Fprintf(w, `{"msg":"%s"}`, err.Error())
		return
	}
	defer func() { _ = r.Body.Close() }()

	u := User{}
	if err := json.Unmarshal(body, &u); err != nil {
		w.WriteHeader(http.StatusInternalServerError)
		_, _ = fmt.Fprintf(w, `{"msg":"%s"}`, err.Error())
		return
	}
	u.ID = users[len(users)-1].ID + 1
	users = append(users, u)

	w.WriteHeader(http.StatusCreated)
}
```

在这个 Handler 中，首先写入响应头 `Content-Type: application/json`，表示创建用户的响应内容为 JSON 格式。

接着从请求体 `r.Body` 中读取客户端提交的用户信息。

如果读取请求体失败，则写入响应状态码 `400`，表示客户端提交的用户信息有误，并返回 JSON 错误响应。

接着，使用 `json.Unmarshal` 对请求体进行 JSON 解码，将数据填入 `User` 结构体中。

如果 JSON 解码失败，则写入响应状态码 `500`，表示服务端出现了错误，并返回 JSON 错误响应。

最终，将新创建的用户信息保存到 `users` 切片中，并写入响应状态码 `201`，表示用户创建成功。注意，根据 RESTful 规范，这里并不需要返回响应体。

下面，我们来分析下如何对这个 Handler 函数编写单元测试代码。

首先，我们思考下 `CreateUserHandler` 这个函数都有哪些外部依赖？

从函数参数来看，我们需要一个用来表示 HTTP 响应的 `http.ResponseWriter`，一个用来表示 HTTP 请求的 `*http.Request`，以及一个用来记录 HTTP 请求路由参数的 `httprouter.Params`。

在函数内部，则依赖了全局变量 `users`。

知道了这些外部依赖，那么，我们如何编写单元测试才能解决这些外部依赖呢？

最直接的办法，就是启动这个 Web Server，然后在单元测试代码中对 `POST /users` 接口发送一个 HTTP 请求，之后判断程序的 HTTP 响应结果以及 `users` 变量中的数据，来验证 `CreateUserHandler` 函数的正确性。

但这种做法显然超出了单元测试的范畴，更像是在做集成测试。单元测试的一个主要特征就是要隔离外部依赖，使用`测试替身`来替换依赖。

所以，我们应该想办法来制作`测试替身`。

我们先从最简单的 `users` 变量开始，想办法在测试过程中替换掉 `users`。

`users` 仅是一个切片变量，用来保存用户数据，我们可以编写一个函数，将其内容替换成测试数据，代码如下：

```go
func setupTestUser() func() {
	defaultUsers := users
	users = []User{
		{ID: 1, Name: "test-user1"},
	}
	return func() {
		users = defaultUsers
	}
}
```

`setupTestUser` 函数内部为全局变量 `users` 进行了重新赋值，并返回一个匿名函数，这个匿名函数可以将 `users` 变量值恢复。

在测试期间可以这样使用：

```go
func TestCreateUserHandler(t *testing.T) {
	cleanup := setupTestUser()
	defer cleanup()
	...
}
```

在测试最开始时调用 `setupTestUser` 来初始化测试数据，使用 `defer` 语句实现测试函数退出时恢复 `users` 数据。

接下来，我们需要构造一个表示 HTTP 响应的 `http.ResponseWriter`。

幸运的是，这并不需要费多少力气，Go 语言官方早就想到了这个诉求，为我们提供了 `net/http/httptest` 标准库，这个库实现了一些专门用来进行网络测试的实用工具。

构造一个测试用的 HTTP 响应对象仅需一行代码就能完成：

```go
w := httptest.NewRecorder()
```

得到的 `w` 变量实现了 `http.ResponseWriter` 接口，可以直接传递给 Handler 函数。

要想构造一个表示 HTTP 请求的 `*http.Request` 对象，同样非常简单：

```go
body := strings.NewReader(`{"name": "user2"}`)
req := httptest.NewRequest("POST", "/users", body)
```

使用 `httptest.NewRequest` 创建的 `req` 变量正是 `*http.Request` 类型，它包含了请求方法、路径、请求体。

现在，我们只差一个用来记录 HTTP 请求路由参数的 `httprouter.Params` 类型对象没有构造了。

`httprouter.Params` 是由 `httprouter` 这个第三方包提供的，`httprouter` 是一个高性能的 HTTP 路由，兼容 `net/http` 标准库。

它提供了 `(*httprouter.Router).ServeHTTP` 方法，可以调用请求对应的 Handler 函数。即可以根据请求对象 `*http.Request`，自动调用 `CreateUserHandler` 函数。

在调用 Handler 函数时，`httprouter` 会解析请求中的路由参数保存在 `httprouter.Params` 对象中并传给 Handler，所以这个对象无需我们手动构造。

现在，单元测试函数的逻辑就清晰了：

```go
func TestCreateUserHandler(t *testing.T) {
	cleanup := setupTestUser()
	defer cleanup()

	w := httptest.NewRecorder()

	body := strings.NewReader(`{"name": "user2"}`)
	req := httptest.NewRequest("POST", "/users", body)

	router := setupRouter()
	router.ServeHTTP(w, req)
}
```

根据前文的讲解，我们构造了单元测试所需的依赖项。

`setupRouter()` 返回 `*httprouter.Router` 对象，当代码执行到 `router.ServeHTTP(w, req)` 时，就会根据传递的 `req` 参数，自动调用与之匹配的 Handler，即被测试函数 `CreateUserHandler`。

接下来，我们要做的就是判断 `CreateUserHandler` 函数执行后的结果是否正确。

完整单元测试代码如下：

```go
package main

import (
	"encoding/json"
	"net/http/httptest"
	"strings"
	"testing"

	"github.com/stretchr/testify/assert"
)

func TestCreateUserHandler(t *testing.T) {
	cleanup := setupTestUser()
	defer cleanup()

	w := httptest.NewRecorder()

	body := strings.NewReader(`{"name": "user2"}`)
	req := httptest.NewRequest("POST", "/users", body)

	router := setupRouter()
	router.ServeHTTP(w, req)

	assert.Equal(t, 201, w.Code)
	assert.Equal(t, "application/json", w.Header().Get("Content-Type"))
	assert.Equal(t, "", w.Body.String())

	assert.Equal(t, 2, len(users))
	u2, _ := json.Marshal(users[1])
	assert.Equal(t, `{"id":2,"name":"user2"}`, string(u2))
}
```

这里引入了第三方包 [testify](https://github.com/stretchr/testify) 用来进行断言操作，`assert.Equal` 能够判断两个对象是否相等，这可以简化代码，不再需要使用 `if` 来判断了。更多关于 `testify` 包的使用，可以查看[官方文档](https://pkg.go.dev/github.com/stretchr/testify)。

我们首先断言了响应状态码是否为 `201`。

接着又断言了响应头的 `Content-Type` 字段是否为 `application/json`。

然后判断了响应内容是否为空。

最后，通过 `users` 中的值来判断用户信息是否保存正确。

使用 `go test` 来执行测试函数：

```bash
$ go test -v -run="TestCreateUserHandler" . 
=== RUN   TestCreateUserHandler
--- PASS: TestCreateUserHandler (0.00s)
PASS
ok      github.com/jianghushinian/blog-go-example/test/http/server      0.544s
```

测试通过。

至此，我们成功为 `CreateUserHandler` 函数编写了一个单元测试。

不过，这个单元测试仅覆盖了正常逻辑，`CreateUserHandler` 方法返回 `400` 和 `500` 两种状态码的逻辑没有被测试覆盖，这两种场景就留做作业你自己来完成吧。

接下来，我们再为获取用户信息的函数 `GetUserHandler` 编写一个单元测试。

先来看下 `GetUserHandler` 函数的定义：

```go
func GetUserHandler(w http.ResponseWriter, r *http.Request, ps httprouter.Params) {
	userID, _ := strconv.Atoi(ps[0].Value)
	w.Header().Set("Content-Type", "application/json")

	for _, u := range users {
		if u.ID == userID {
			user, _ := json.Marshal(u)
			_, _ = w.Write(user)
			return
		}
	}

	w.WriteHeader(http.StatusNotFound)
	_, _ = w.Write([]byte(`{"msg":"notfound"}`))
}
```

获取用户信息的逻辑，相对简单一点。

首先从 HTTP 请求的路径参数中获取用户 ID。

然后判断这个 ID 对应的用户信息是否存在，如果存在就返回用户信息。

不存在，则写入 `404` 状态码，并返回 `notfound` 信息。

有了前文对 `CreateUserHandler` 函数编写测试的经验，想必如何对 `GetUserHandler` 函数进行测试你已经轻车熟路了。

以下是我为其编写的测试代码：

```go
func TestGetUserHandler(t *testing.T) {
	cleanup := setupTestUser()
	defer cleanup()

	type want struct {
		code int
		body string
	}
	tests := []struct {
		name string
		args int
		want want
	}{
		{
			name: "get test-user1",
			args: 1,
			want: want{
				code: 200,
				body: `{"id":1,"name":"test-user1"}`,
			},
		},
		{
			name: "get user not found",
			args: 2,
			want: want{
				code: 404,
				body: `{"msg":"notfound"}`,
			},
		},
	}

	router := setupRouter()
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			req := httptest.NewRequest("GET", fmt.Sprintf("/users/%d", tt.args), nil)

			w := httptest.NewRecorder()
			router.ServeHTTP(w, req)

			assert.Equal(t, tt.want.code, w.Code)
			assert.Equal(t, tt.want.body, w.Body.String())
		})
	}
}
```

获取用户信息的单元测试代码，在测试执行开始，同样使用 `setupTestUser` 函数来初始化测试数据，并使用 `defer` 来完成数据恢复。

这次为了提高测试覆盖率，我对 `GetUserHandler` 函数的正常响应以及返回 `404` 状态码的异常响应场景都进行了测试。

这里使用了表格测试，不了解表格测试的读者，可以查看我的另一篇文章[《在 Go 中如何编写测试代码》](https://jianghushinian.cn/2023/07/09/how-to-write-testing-in-go/)。

除了使用表格测试的形式，其他测试逻辑与 `CreateUserHandler` 的单元测试逻辑基本相同，我就不过多介绍了。

使用 `go test` 来执行测试函数：

```bash
$ go test -v -run="TestGetUserHandler" .
=== RUN   TestGetUserHandler
=== RUN   TestGetUserHandler/get_test-user1
=== RUN   TestGetUserHandler/get_user_not_found
--- PASS: TestGetUserHandler (0.00s)
    --- PASS: TestGetUserHandler/get_test-user1 (0.00s)
    --- PASS: TestGetUserHandler/get_user_not_found (0.00s)
PASS
ok      github.com/jianghushinian/blog-go-example/test/http/server      0.516s
```

表格测试的两个用例都通过了测试。

### HTTP Client 测试

接下来，我们来看下，站在 HTTP Client 端的角度，如何编写应用程序的测试代码。

假设我们有一个进程监控程序，能够检测某个进程是否正在执行，如果进程退出，就发送一条消息通知到飞书群。

代码如下：

```go
package main

import (
	"bytes"
	"encoding/json"
	"fmt"
	"log"
	"net/http"
	"os"
	"strconv"
	"syscall"
	"time"
)

func monitor(pid int) (*Result, error) {
	for {
		// 检查进程是否存在
		err := syscall.Kill(pid, 0)
		if err != nil {
			log.Printf("Process %d exited\n", pid)
			webhook := os.Getenv("WEBHOOK")
			return sendFeishu(fmt.Sprintf("Process %d exited", pid), webhook)
		}

		log.Printf("Process %d is running\n", pid)
		time.Sleep(1 * time.Second)
	}
}

func main() {
	if len(os.Args) != 2 {
		log.Println("Usage: ./monitor <pid>")
		return
	}

	pid, err := strconv.Atoi(os.Args[1])
	if err != nil {
		log.Printf("Invalid pid: %s\n", os.Args[1])
		return
	}

	result, err := monitor(pid)
	if err != nil {
		log.Fatal(err)
	}
	log.Println(result)
}
```

这个程序可以通过 `./monitor <pid>` 形式启动。

`monitor` 函数内部有一个循环，会根据传递进来的进程 PID 不断的来检测对应进程是否存在。

如果不存在，则说明进程已经停止，然后调用 `sendFeishu` 函数发送消息通知到指定的飞书 `webhook` 地址。

`monitor` 函数会将 `sendFeishu` 函数的返回结果原样返回。

`sendFeishu` 函数实现如下：

```go
type Message struct {
	Content struct {
		Text string `json:"text"`
	} `json:"content"`
	MsgType string `json:"msg_type"`
}

type Result struct {
	StatusCode    int    `json:"StatusCode"`
	StatusMessage string `json:"StatusMessage"`
	Code          int    `json:"code"`
	Data          any    `json:"data"`
	Msg           string `json:"msg"`
}

func sendFeishu(content, webhook string) (*Result, error) {
	msg := Message{
		Content: struct {
			Text string `json:"text"`
		}{
			Text: content,
		},
		MsgType: "text",
	}

	body, _ := json.Marshal(msg)
	resp, err := http.Post(webhook, "application/json", bytes.NewReader(body))
	if err != nil {
		return nil, err
	}
	defer func() { _ = resp.Body.Close() }()

	result := new(Result)
	if err := json.NewDecoder(resp.Body).Decode(result); err != nil {
		return nil, err
	}
	if result.Code != 0 {
		return nil, fmt.Errorf("code: %d, error: %s", result.Code, result.Msg)
	}

	return result, nil
}
```

`sendFeishu` 函数能够将传递进来的消息发送到指定的 `webhook` 地址。

至于内部具体逻辑，我们并不需要关心，只当作第三方包来使用即可，仅需要知道它最终会返回 `*Result` 对象。

现在我们需要对 `monitor` 函数进行测试。

我们同样需要先分析下 `monitor` 函数的外部依赖是什么。

首先 `monitor` 函数的参数 `pid` 是一个 `int` 类型，不难构造。

`monitor` 函数内部调用了 `sendFeishu` 函数，并且将 `sendFeishu` 的返回结果原样返回，所以 `sendFeishu` 函数是一个外部依赖。

另外，传递个给 `sendFeishu` 函数的 `webhook` 地址是从环境变量中获取的，这也算是一个外部依赖。

所以要测试 `monitor` 函数，我们需要使用`测试替身`来解决这两个外部依赖项。

对于环境变量的依赖很好解决，Go 提供了 `os.Setenv` 可以在程序中动态设置环境变量的值。

对于另一个依赖项 `sendFeishu` 函数，它又依赖了 `webhook` 地址所对应的 HTTP Server。

所以我们需要解决 HTTP Server 的依赖问题。

针对 HTTP Server，Go 标准库 `net/http/httptest` 同样提供了对应工具。

我们可以使用 `httptest.NewServer` 创建一个测试用的 HTTP Server：

```go
func newTestServer() *httptest.Server {
	return httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		w.Header().Set("Content-Type", "application/json")

		switch r.RequestURI {
		case "/success":
			_, _ = fmt.Fprintf(w, `{"StatusCode":0,"StatusMessage":"success","code":0,"data":{},"msg":"success"}`)
		case "/error":
			_, _ = fmt.Fprintf(w, `{"code":19001,"data":{},"msg":"param invalid: incoming webhook access token invalid"}`)
		}
	}))
}
```

`newTestServer` 函数返回一个用于测试的 HTTP Server 对象。

在 `newTestServer` 函数内部，定义了两个路由 `/success` 和 `/error`，分别来处理成功响应和失败响应两种情况。

与前文介绍的 `setupTestUser` 函数一样，我们需要在测试程序开始执行时准备测试数据，即启动这个测试用的 HTTP Server，在测试程序执行完成后清理数据，即关闭 HTTP Server。

不过，这次我们不再使用 `setupTestUser` 函数结合 `defer cleanup()` 的方式，而是换种方式来实现：

```go
var ts *httptest.Server

func TestMain(m *testing.M) {
	ts = newTestServer()
	m.Run()
	ts.Close()
}
```

首先我们定义了一个全局变量 `ts`，用来保存测试用的 HTTP Server 对象。

然后在 `TestMain` 函数中调用 `newTestServer` 函数为 `ts` 变量赋值。

接下来执行 `m.Run()` 方法。

最终调用 `ts.Close()` 关闭 HTTP Server。

`TestMain` 函数名不是随意取的，而是 Go 单元测试中的一个约定名称，它相当于 `main` 函数，在使用 `go test` 命令执行所有测试用例前，会优先执行 `TestMain` 函数。

在 `TestMain` 函数中调用 `m.Run()`，`(*testing.M).Run()` 方法会执行全部的测试用例。

当所有测试用例执行完成后，代码才会执行到 `ts.Close()`。

所以，相较于 `setupTestUser` 函数在每个测试函数内部都要调用一次的用法，`TestMain` 函数更加省力。不过这也决定了二者适用场景不同。`TestMain` 函数粒度更大，作用于全部测试用例，`setupTestUser` 函数只作用于单个测试函数。

现在，我们已经解决了 `monitor` 函数的依赖项问题。

为其编写的单元测试如下：

```go
func Test_monitor(t *testing.T) {
	type args struct {
		pid     int
		webhook string
	}
	tests := []struct {
		name    string
		args    args
		want    *Result
		wantErr error
	}{
		{
			name: "process exited and send feishu success",
			args: args{
				pid:     10000000,
				webhook: ts.URL + "/success",
			},
			want: &Result{
				StatusCode:    0,
				StatusMessage: "success",
				Code:          0,
				Data:          make(map[string]interface{}),
				Msg:           "success",
			},
		},
		{
			name: "process exited and send feishu error",
			args: args{
				pid:     20000000,
				webhook: ts.URL + "/error",
			},
			wantErr: errors.New("code: 19001, error: param invalid: incoming webhook access token invalid"),
		},
	}
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			_ = os.Setenv("WEBHOOK", tt.args.webhook)

			got, err := monitor(tt.args.pid)
			if err != nil {
				if tt.wantErr == nil || err.Error() != tt.wantErr.Error() {
					t.Errorf("monitor() error = %v, wantErr %v", err, tt.wantErr)
					return
				}
			}

			if !reflect.DeepEqual(got, tt.want) {
				t.Errorf("monitor() got = %v, want %v", got, tt.want)
			}
		})
	}
}
```

这里同样采用表格测试的方式，有两个测试用例，一个用于测试被检测程序退出后发送飞书消息成功的情况，一个用于测试被检测程序退出后发送飞书消息失败的情况。

测试用例中 `pid` 被设置为很大的值，已经超过了 Linux 系统允许的最大 `pid` 值，所以检测结果一定是程序已经退出。

由于被检测程序不退出的情况，`monitor` 函数会一直循环检测，逻辑比较简单，就没有对这个逻辑编写测试用例。

使用 `go test` 来执行测试函数：

```bash
$ go test -v -run="^Test_monitor$" .
=== RUN   Test_monitor
=== RUN   Test_monitor/process_exited_and_send_feishu_success
2023/07/15 13:27:46 Process 10000000 exited
=== RUN   Test_monitor/process_exited_and_send_feishu_error
2023/07/15 13:27:46 Process 20000000 exited
--- PASS: Test_monitor (0.00s)
    --- PASS: Test_monitor/process_exited_and_send_feishu_success (0.00s)
    --- PASS: Test_monitor/process_exited_and_send_feishu_error (0.00s)
PASS
ok      github.com/jianghushinian/blog-go-example/test/http/client      0.166s
```

测试通过。

以上，我们通过 `net/http/httptest` 提供的测试工具，在本地启动了一个测试 HTTP Server，来解决被测试代码依赖外部 HTTP 服务的问题。

有时候，我们不想真正的在本地启动一个 HTTP Server，或者无法做到这一点。

那么，我们还有另一种方案来解决这个问题，可以使用 `gock` 来模拟 HTTP 服务。

`gock` 是 Go 社区中的一个第三方包，虽然不在本地启动一个 HTTP Server，但是它能够拦截所有被 mock 的 HTTP 请求。所以，我们能够利用 `gock` 拦截 `sendFeishu` 函数发送给 `webhook` 地址的请求，然后返回 mock 数据。这样，就可以使用 mock 的方式来解决依赖外部 HTTP 服务的问题。

使用 `gock` 编写的单元测试代码如下：

```go
package main

import (
	"os"
	"testing"

	"github.com/h2non/gock"
	"github.com/stretchr/testify/assert"
)

func Test_monitor_by_gock(t *testing.T) {
	defer gock.Off() // Flush pending mocks after test execution

	gock.New("http://localhost:8080").
		Post("/webhook").
		Reply(200).
		JSON(map[string]interface{}{
			"StatusCode":    0,
			"StatusMessage": "success",
			"Code":          0,
			"Data":          make(map[string]interface{}),
			"Msg":           "success",
		})

	_ = os.Setenv("WEBHOOK", "http://localhost:8080/webhook")
	got, err := monitor(30000000)
	assert.NoError(t, err)
	assert.Equal(t, &Result{
		StatusCode:    0,
		StatusMessage: "success",
		Code:          0,
		Data:          make(map[string]interface{}),
		Msg:           "success",
	}, got)

	assert.True(t, gock.IsDone())
}
```

首先，在测试函数的开始，使用 `defer` 延迟调用 `gock.Off()`，可以保证在测试完成后刷新挂起的 mock，即还原被 mock 对象的初始状态。

然后，我们使用 `gock.New()` 对 `http://localhost:8080` 这个 URL 进行 mock，这样 `gock` 会拦截测试过程中所有发送到这个地址的 HTTP 请求。

`gock.New()` 支持链式调用，`.Post("/webhook")` 表示拦截对 `/webhook` 这个 URL 的 POST 请求。

`.Reply(200)` 表示针对这个请求，返回 `200` 状态码。

`.JSON(...)` 即为返回的 JSON 格式响应内容。

接着，我们将 `webhook` 地址设置为 `http://localhost:8080/webhook`，这样，在调用 `sendFeishu` 函数时发送的请求就会被拦截，并返回上一步中的 `.JSON(...)` 内容。

之后就是调用 `monitor` 函数，并断言测试结果是否正确。

最后，调用 `assert.True(t, gock.IsDone())` 来验证已经没有挂起的 mock 了。

使用 `go test` 来执行测试函数：

```bash
$ go test -v -run="^Test_monitor_by_gock$" .
=== RUN   Test_monitor_by_gock
2023/07/15 13:28:22 Process 30000000 exited
--- PASS: Test_monitor_by_gock (0.00s)
PASS
ok      github.com/jianghushinian/blog-go-example/test/http/client      0.574s
```

单元测试执行通过。

### 总结

本文向大家介绍了在 Go 中编写单元测试时，如何解决 HTTP 外部依赖的问题。

我们分别站在 HTTP 服务端和 HTTP 客户端两个角度，使用 `net/http/httptest` 标准库和 `gock` 第三方库来实现`测试替身`解决 HTTP 外部依赖。

并且分别介绍了使用 `setupTestUser` + `defer cleanup()` 以及 `TestMain` 两种形式，来做测试准备和清理工作。二者作用于不同粒度，需要根据测试需要进行选择。

本文完整代码示例我放在了 [GitHub](https://github.com/jianghushinian/blog-go-example/tree/main/test/http) 上，欢迎点击查看。

希望此文能对你有所帮助。

**联系我**

- 微信：jianghushinian
- 邮箱：[jianghushinian007@outlook.com](mailto:jianghushinian007@outlook.com)
- 博客地址：https://jianghushinian.cn

**参考**

- Go testing 文档：https://pkg.go.dev/net/http/httptest
- Testify 源码：https://github.com/stretchr/testify
- gock 源码：https://github.com/h2non/gock
