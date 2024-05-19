---
title: 'xgo: 一款新鲜出炉的 Go 代码测试利器'
date: 2024-05-19 16:14:55
tags:
- Go
- Test
categories:
- Go
---

我曾经写过一篇文章[《测试代码终极解决方案 Monkey Patching》](https://jianghushinian.cn/2023/07/22/the-ultimate-solution-to-test-code-monkey-patching/)，里面介绍了 Go 语言中的猴子补丁方案。如今，时隔数月我又发现了一款新的工具可以实现 Monkey Patching，本文将带大家一起尝鲜下这款新的测试工具表现如何。

<!-- more -->

### 简介

简单一句话介绍 `xgo`：它是一款强大的的 Go 测试工具集，功能包括 Trap、Mock、Trace、增量覆盖率。

当然，开发中最常用的还是 Mock 功能，也是本文讲解的重点（不要慌，其他功能也会介绍）。

以下是 `xgo` 支持的所有平台：

|| x86 | x86_64 (amd64) | arm64 | any other Arch... |
| :----: | :----: | :----: | :----: | :----: |
| Linux  | Y | Y | Y | Y |
| Windows | Y | Y | Y | Y |
| macOS | Y | Y	| Y	| Y |
| any other OS... |	Y | Y | Y | Y |

可以发现，`xgo` 支持所有 `go` 语言支持的 OS 和 Arch，即它是跨平台的。

跨平台这一点是最吸引我的地方，也是能让 `xgo` 脱颖而出的关键。

此外，`xgo` 还是并发安全的 Monkey Patching 方案，这点也是有别于其他方案的一个亮点。

本文就以测试一个 HTTP 服务程序来演示 `xgo` 的基本使用。

### HTTP 服务程序示例

假设我们有一个 HTTP 服务程序对外提供用户服务，代码如下：

```go
package main

import (
	"encoding/json"
	"fmt"
	"io"
	"net/http"
	"strconv"

	"github.com/julienschmidt/httprouter"
	"gorm.io/driver/mysql"
	"gorm.io/gorm"
)

type User struct {
	ID   int
	Name string
}

func NewMySQLDB(host, port, user, pass, dbname string) (*gorm.DB, error) {
	dsn := fmt.Sprintf("%s:%s@tcp(%s:%s)/%s?charset=utf8mb4&parseTime=True&loc=Local",
		user, pass, host, port, dbname)
	return gorm.Open(mysql.Open(dsn), &gorm.Config{})
}

func NewUserHandler(store *gorm.DB) *UserHandler {
	return &UserHandler{store: store}
}

type UserHandler struct {
	store *gorm.DB
}

func (h *UserHandler) CreateUser(w http.ResponseWriter, r *http.Request, _ httprouter.Params) {
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
		w.WriteHeader(http.StatusBadRequest)
		_, _ = fmt.Fprintf(w, `{"msg":"%s"}`, err.Error())
		return
	}

	if err := h.store.Create(&u).Error; err != nil {
		w.WriteHeader(http.StatusInternalServerError)
		_, _ = fmt.Fprintf(w, `{"msg":"%s"}`, err.Error())
		return
	}
	w.WriteHeader(http.StatusCreated)
}

func (h *UserHandler) GetUser(w http.ResponseWriter, r *http.Request, ps httprouter.Params) {
	id := ps[0].Value
	uid, _ := strconv.Atoi(id)

	w.Header().Set("Content-Type", "application/json")
	var u User
	if err := h.store.First(&u, uid).Error; err != nil {
		w.WriteHeader(http.StatusInternalServerError)
		_, _ = fmt.Fprintf(w, `{"msg":"%s"}`, err.Error())
		return
	}
	_, _ = fmt.Fprintf(w, `{"id":%d,"name":"%s"}`, u.ID, u.Name)
}

func setupRouter(handler *UserHandler) *httprouter.Router {
	router := httprouter.New()
	router.POST("/users", handler.CreateUser)
	router.GET("/users/:id", handler.GetUser)
	return router
}

func main() {
	mysqlDB, _ := NewMySQLDB("localhost", "3306", "user", "password", "test")
	handler := NewUserHandler(mysqlDB)
	router := setupRouter(handler)
	_ = http.ListenAndServe(":8000", router)
}
```

这是一个简单的 Web Server 程序，服务监听 `8000` 端口，提供了两个接口：

`POST /users` 用来创建用户。

`GET /users/:id` 用来查询指定 ID 对应的用户信息。

代码逻辑比较简单，我就不详细讲解了。

为了保证业务的正确性，我们应该对 `(*UserHandler).CreateUser` 和 `(*UserHandler).GetUser` 这两个 Handler 方法进行单元测试。

### 使用 xgo 进行单元测试

#### 安装

`xgo` 使用前必须通过 `go install` 命令进行安装：

```bash
$ go install github.com/xhd2015/xgo/cmd/xgo@latest
$ xgo version
1.0.35
```

#### 编写测试代码

以 `(*UserHandler).CreateUser` 方法为例演习下如何编写测试代码。

我们先来分析下这个方法的依赖项：

首先`UserHandler` 这个结构体本身有一个 `store` 属性，依赖了 `*gorm.DB` 对象。

其次，`CreateUser` 方法还接收三个参数，它们都属于 HTTP 网络相关的外部依赖，你可以在我的另一篇文章[《在 Go 语言单元测试中如何解决 HTTP 网络依赖问题》](https://jianghushinian.cn/2023/07/15/how-to-resolve-http-dependencies-in-go-testing/)中找到解决方案，就不在本文中进行讲解了。

所以，我们应该要想办法解决 `*gorm.DB` 这个外部依赖。

由于我们编写代码时，没有为支持单元测试而专门使用接口来进行解耦，导致 `UserHandler` 结构体直接依赖了 `*gorm.DB` 结构体对象，无法使用 `gomock` 工具对依赖项进行 Mock。

在不改变代码的前提下，我们可以使用 `xgo` 提供的 Monkey Patching 技术为依赖对象 `*gorm.DB` 打上猴子补丁，以此来解决测试代码中难以调用 `h.store.First(&u, uid).Error` 方法问题。

要使用 `xgo` 编写测试，需要引入 `xgo` 提供的 `runtime` 包，所以先使用 `go get` 命令将其添加到 `go.mod` 依赖项： 

```bash
$ go get github.com/xhd2015/xgo/runtime@latest
```

使用 `xgo` 为 `(*UserHandler).CreateUser` 方法编写的测试代码如下：

```go
package main

import (
	"net/http/httptest"
	"strings"
	"testing"

	"github.com/stretchr/testify/assert"
	"github.com/xhd2015/xgo/runtime/mock"
	"gorm.io/gorm"
)

func TestUserHandler_CreateUser(t *testing.T) {
	mysqlDB := &gorm.DB{}
	handler := NewUserHandler(mysqlDB)
	router := setupRouter(handler)

	// 为 mysqlDB 打上猴子补丁，替换其 Create 方法
	mock.Patch(mysqlDB.Create, func(value interface{}) (tx *gorm.DB) {
		expected := &User{
			Name: "user1",
		}
		actual := value.(*User)
		assert.Equal(t, expected, actual)
		return mysqlDB
	})

	w := httptest.NewRecorder()
	req := httptest.NewRequest("POST", "/users", strings.NewReader(`{"name": "user1"}`))
	router.ServeHTTP(w, req)

	// 断言成功响应
	assert.Equal(t, 201, w.Code)
	assert.Equal(t, "application/json", w.Header().Get("Content-Type"))
	assert.Equal(t, "", w.Body.String())
}
```

我们使用 `xgo` 提供的 `mock.Patch` 方法，为 `mysqlDB` 对象的 `Create` 方法打了一个猴子补丁，然后使用匿名函数来实现这个 `Create` 方法，并且，在匿名函数的内部还对 `Create` 方法接收到的参数进行了验证。

没错，`xgo` 使用起来就是这么简单，这也体现了猴子补丁的强大，它能原地修改 `mysqlDB.Create` 方法的实现。

这样，在执行测试代码时，测试方法将不再执行 `mysqlDB.Create` 原方法内部逻辑，而会被替换为调用在此定义的匿名函数逻辑。

要执行测试，我们不能像原来一样使用 `go test` 来执行测试函数，需要将 `go` 命令替换为 `xgo` 命令：

```bash
$ xgo test -v -run TestUserHandler_CreateUser          
=== RUN   TestUserHandler_CreateUser
--- PASS: TestUserHandler_CreateUser (0.00s)
PASS
ok      github.com/jianghushinian/blog-go-example/test/xgo      0.524s
```

测试通过。

`xgo` 用法跟普通的 `go test` 用法完全相同，这也大大简化了我们切换命令的心智负担，几乎零成本切换。

> NOTE: 如果直接使用 `go test -v -run TestUserHandler_CreateUser` 执行测试将得到报错，读者可自行测试。

接下来我们再为 `(*UserHandler).GetUser` 方法编写如下测试代码：

```go
func TestUserHandler_GetUser(t *testing.T) {
	mysqlDB := &gorm.DB{}
	handler := NewUserHandler(mysqlDB)
	router := setupRouter(handler)

	// 为 mysqlDB 打上猴子补丁，替换其 First 方法
	mock.Patch(mysqlDB.First, func(dest interface{}, conds ...interface{}) (tx *gorm.DB) {
		assert.Equal(t, dest, &User{})
		assert.Equal(t, len(conds), 1)
		assert.Equal(t, conds[0], 1)

		u := dest.(*User)
		u.ID = 1
		u.Name = "user1"
		return mysqlDB
	})

	w := httptest.NewRecorder()
	req := httptest.NewRequest("GET", "/users/1", nil)
	router.ServeHTTP(w, req)

	assert.Equal(t, 200, w.Code)
	assert.Equal(t, "application/json", w.Header().Get("Content-Type"))
	assert.Equal(t, `{"id":1,"name":"user1"}`, w.Body.String())
}
```

与之前的套路如出一辙，使用 `xgo` 执行测试：

```bash
$ xgo test -v -run TestUserHandler_GetUser          
=== RUN   TestUserHandler_GetUser
--- PASS: TestUserHandler_GetUser (0.00s)
PASS
ok      github.com/jianghushinian/blog-go-example/test/xgo      0.424s
```

测试通过。

现在，你也许会问，这种使用 `mock.Patch` 打过猴子补丁的测试代码需要使用 `xgo` 才能执行，那没有用到 `mock.Patch` 的普通测试代码能不能也用 `xgo` 执行呢？答案是肯定的。

比如我们随意写一个没什么意义的 demo 测试：

```go
func TestDemo(t *testing.T) {
	t.Log("---------- TestDemo ----------")
}
```

使用 `xgo` 执行测试代码：

```bash
$ xgo test -v -run TestDemo               
=== RUN   TestDemo
    main_test.go:65: ---------- TestDemo ----------
--- PASS: TestDemo (0.00s)
PASS
ok      github.com/jianghushinian/blog-go-example/test/xgo      0.219s
```

测试通过。

使用 `xgo` 一次执行全部测试代码：

```bash
$ xgo test -v               
=== RUN   TestUserHandler_CreateUser
--- PASS: TestUserHandler_CreateUser (0.00s)
=== RUN   TestUserHandler_GetUser
--- PASS: TestUserHandler_GetUser (0.00s)
=== RUN   TestDemo
    main_test.go:65: ---------- TestDemo ----------
--- PASS: TestDemo (0.00s)
PASS
ok      github.com/jianghushinian/blog-go-example/test/xgo      0.175s
```

测试通过。

这样我们就统一了执行测试代码的方式，所有测试都可以使用 `xgo` 来执行。理论上，我们只需要将现有项目执行 `go test` 的地方，替换成 `xgo test` 即可兼容所有测试代码，这大大降低了引入 `xgo` 的迁移成本。

### xgo 其他功能

前文提到，`xgo` 核心功能包括 Trap、Mock、Trace、增量覆盖率。

其实我们上面介绍的 `mock.Patch` 即为 Mock 功能，不过除了这个 API，`xgo` 还提供了另外一个 Mock API `mock.Mock`，实际上这两个方法底层调用的是同一个函数，用法也类似，我就不进行演示了，感兴趣的读者可以深入源码进行研究。

接下来我将依次介绍下 Trap、Trace、增量覆盖率这几个功能。

#### Trap

Trap 是 `xgo` 的核心，也是 Mock、Trace 功能的基础，它可以对 Go 函数进行拦截。

以下是一个官方文档中使用 Trap 的例子：

```go
package main

import (
	"context"
	"fmt"

	"github.com/xhd2015/xgo/runtime/core"
	"github.com/xhd2015/xgo/runtime/trap"
)

func init() {
	trap.AddInterceptor(&trap.Interceptor{
		Pre: func(ctx context.Context, f *core.FuncInfo, args core.Object, results core.Object) (interface{}, error) {
			if f.Name == "A" {
				fmt.Printf("trap A\n")
				return nil, nil
			}
			if f.Name == "B" {
				fmt.Printf("abort B\n")
				return nil, trap.ErrAbort
			}
			return nil, nil
		},
	})
}

func main() {
	A()
	B()
}

func A() {
	fmt.Printf("A\n")
}

func B() {
	fmt.Printf("B\n")
}
```

使用 `go` 命令执行代码：

```bash
$ go run main.go
A
B
```

代码正常执行。

如果改为使用 `xgo` 执行代码；

```bash
$ xgo run main.go
trap A
A
abort B
```

可以发现，`xgo` 改变了代码执行结果，这就是 Trap 的强大之处，`xgo` 拦截了原有代码的逻辑，进而执行拦截器内部的逻辑。

不过这种用法并不太多，我们更多的场景还是使用更上层的 Mock 功能来编写测试代码。

#### Trace

Trace 功能可以将 Go 程序执行过程可视化，在一定程度上可以替代 Debug 工具，方便我们以可视化的方式进行代码调试。

要想使用 Trace 功能，也很简单，仅需要在使用 `xgo` 执行测试代码时加上 `--strace` 标志：

```bash
$ xgo test -v -run TestDemo --strace
=== RUN   TestDemo
    main_test.go:65: ---------- TestDemo ----------
--- PASS: TestDemo (0.00s)
PASS
ok      github.com/jianghushinian/blog-go-example/test/xgo      0.162s
```

执行以上命令会在当前目录生成一个 `TestDemo.json` 文件，文件中即为可视化所需报告数据。

接下来执行如下命令即可开启 Trace 可视化服务：

```bash
$ xgo tool trace TestDemo.json                     
Server listen at http://localhost:7070
```

此时会自动打开浏览器显示类似如下页面：

![Trace Demo](trace.png)

左侧列表可视化的展示了堆栈跟踪信息，每项前面如果是蓝色表示被调用函数正常返回，红色表示返回错误。如果你使用 VSCode 开发代码的话，点击 VSCode 图标还会自动定位到 VSCode 中函数定义的位置，方便排查问题。

遗憾的是，经过笔者实测目前此功能还不够稳定，存在影响使用的 BUG，甚至经常超时无法生成 Trace 文件。

#### 增量覆盖率

我们要介绍的最后一个功能是增量覆盖率。

`go test` 本身支持测试覆盖率，不过 `xgo` 更近一步，它可以根据 `Git` 变更，计算出增量测试覆盖率，极大方便了代码 review 的过程。

为了查看变更代码的增量覆盖率，我们对 `GetUser` 方法代码进行了如下修改：

![git diff 命令输出](git-diff.png)

使用 `xgo` 命令输出测试覆盖率文件：

```bash
$ xgo test -v -coverpkg . -coverprofile cover.out
=== RUN   TestUserHandler_CreateUser
--- PASS: TestUserHandler_CreateUser (0.00s)
=== RUN   TestUserHandler_GetUser
true...
--- PASS: TestUserHandler_GetUser (0.00s)
=== RUN   TestDemo
    main_test.go:65: ---------- TestDemo ----------
--- PASS: TestDemo (0.00s)
PASS
coverage: 54.8% of statements in .
ok  	github.com/jianghushinian/blog-go-example/test/xgo	0.962s
```

> NOTE: 由于此功能基于 Git，所以如果代码不在 Git 仓库，则执行命令会报错。并且笔者实测，如果一个 Git 仓库存在多个项目情况下，执行命令也会报错。

得到测试覆盖率文件 `cover.out` 后，执行以下命令启动一个本地 Server 来展示测试覆盖率：

```bash
$ xgo tool coverage serve cover.out
```

执行命令后，`xgo` 会自动开启浏览器并访问 `http://localhost:8000` 地址：

![Incremental Coverage](incremental-coverage.png)

默认展示的就是增量代码测试覆盖率。蓝色表示已覆盖，黄色表示未覆盖，展示结果符合预期。

我们也可以切换成查看全局代码测试覆盖率：

![Full Coverage](full-coverage.png)

以上就是 `xgo` 对增量测试覆盖率的支持，还是能够比较方便查看增量代码测试覆盖率的。

### 总结

`xgo` 作为一款 Monkey Patching 解决方案的工具，其支持 Trap、Mock、Trace、增量覆盖率几个功能，方便我们用来编写单元测试。

Trap 是 `xgo` 的核心，虽然不太常用，但上层的 Mock 和 Trace 都是基于 Trap 实现的。

Mock 是我们用的最多的功能，其可以实现跨平台的 Monkey Patching 解决方案。

Trace 功能可以方便我们以可视化的形式对代码进行 Debug。

而增量覆盖率则可以方便我们在 review 代码时可视化的感知到增量代码的测试情况。

总结下 `xgo` 目前的优点和不足：

**优点：**

- 相比于其他 Monkey Patching 解决方案，`xgo` 的跨平台支持最好，这也是我认为 `xgo` 最大的优势。
- 并发安全，多个协程下 Mock 完全隔离。
- 相比于 `gomonkey` 等，`xgo` 在 `Patch` 后不需要进行 `Reset` 操作对猴子补丁进行恢复。
- 使用方便，仅需要将 `go` 命令替换成 `xgo` 即可。

**不足：**

- 虽然使用时仅需将 `go` 命令替换成 `xgo` 即可，但这也意味着需要单独安装 `xgo` 才行，原来的 `go test` 命令执行会报错，不够友好。
- 不能随意升级 Go 版本，至少目前笔者测试结果是这样，我在使用 `go1.22.0` 时有些命令会报错，换成 `go1.21.0` 则没有问题。
- Trace 功能速度比较慢，且存在 BUG。
- 没有提供 `--help` 命令，较为遗憾。

以上优点和不足，都是我个人基于当前版本测试下来的主观使用体验，希望 `xgo` 能够尽快发展起来，补齐短板。

如果，你想了解 `xgo` 的诞生以及实现方案，前几天 `xgo` 作者在 [Go 夜读](https://talkgo.org/)分享了[Go 夜读第 151 期：xgo: 基于编译期代码重写实现 Mock 和 Trace](https://talkgo.org/t/topic/5514)。

本文完整代码示例我放在了 [GitHub](https://github.com/jianghushinian/blog-go-example/tree/main/test/xgo) 上，欢迎点击查看。

希望此文能对你有所帮助。

### P.S.

本来预计这段时间不会再写 Go 测试相关文章了，因为之前写的测试相关文章已经覆盖了大部分日常编写单元测试的场景。不过前段时间 `xgo` 作者联系到我，跟我分享了 `xgo` 项目，解决了 `gomonkey` 项目的兼容性和并发问题。

因为深入研究 Go 语言 Monkey Patching 解决方案这个方向的人很少，所以我对这个项目还是比较感兴趣的，于是花时间体验了下，便有了此文。

`xgo` 项目给在 Go 语言中单元测试带来了新的可能性，是目前我体验过的最方便也是兼容性最好的 Monkey Patching 方案。

诚然，`xgo` 项目还不够成熟，它还非常年轻，刚开源出来不久，但是它开了个好头，期待给它足够的时间，能够成长为 Go 社区里 Monkey Patch 解决方案中最为流行的项目之一。

**参考**

- xgo 源码：https://github.com/xhd2015/xgo
- xgo: 在go中使用-toolexec实现猴子补丁：https://blog.xhd2015.xyz/zh/posts/xgo-monkey-patching-in-go-using-toolexec/
- xgo trace: 一个强大的Go堆栈可视化工具：https://blog.xhd2015.xyz/zh/posts/xgo-trace_a-powerful-visualization-tool-in-go/
- Go 夜读第 151 期：xgo: 基于编译期代码重写实现 Mock 和 Trace：https://talkgo.org/t/topic/5514
- 本文 GitHub 示例代码：https://github.com/jianghushinian/blog-go-example/tree/main/test/xgo

**联系我**

- 公众号：Go编程世界
- 微信：jianghushinian
- 邮箱：[jianghushinian007@outlook.com](mailto:jianghushinian007@outlook.com)
- 博客：https://jianghushinian.cn
