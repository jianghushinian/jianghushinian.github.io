---
title: 在 Go 语言单元测试中如何解决 MySQL 存储依赖问题
date: 2023-07-16 15:53:33
tags:
- Go
- Test
categories:
- Go
---

在编写单元测试的过程中，如果被测试代码有外部依赖，为了便于测试，我们就要想办法来解决这些外部依赖问题。在做 Web 开发时，MySQL 存储就是一个非常常见的外部依赖，本文就来探讨在 Go 语言中编写单元测试时，如何解决 MySQL 存储依赖。

<!-- more -->

### HTTP 服务程序示例

假设我们有一个 HTTP 服务程序对外提供服务，代码如下：

> main.go

```go
package main

import (
	"encoding/json"
	"fmt"
	"gorm.io/gorm"
	"io"
	"net/http"
	"strconv"

	"github.com/julienschmidt/httprouter"

	"github.com/jianghushinian/blog-go-example/test/mysql/store"
)

func NewUserHandler(db *gorm.DB) *UserHandler {
	return &UserHandler{
		store: store.NewUserStore(db),
	}
}

type UserHandler struct {
	store store.UserStore
}

func (h *UserHandler) CreateUser(w http.ResponseWriter, r *http.Request, _ httprouter.Params) {
	...
}

func (h *UserHandler) GetUser(w http.ResponseWriter, r *http.Request, ps httprouter.Params) {
	...
}

func setupRouter(handler *UserHandler) *httprouter.Router {
	router := httprouter.New()
	router.POST("/users", handler.CreateUser)
	router.GET("/users/:id", handler.GetUser)
	return router
}

func main() {
	mysqlDB, _ := store.NewMySQLDB("localhost", "3306", "user", "password", "test")
	handler := NewUserHandler(mysqlDB)
	router := setupRouter(handler)

	_ = http.ListenAndServe(":8000", router)
}
```

这个服务监听 `8000` 端口，分别提供了两个 HTTP 接口：

`POST /users` 用来创建用户。

`GET /users/:id` 用来获取指定 ID 对应的用户信息。

`UserHandler` 是一个结构体，它依赖外部存储接口 `store.UserStore`，这个接口定义如下：

> store/store.go

```go
package store

import "gorm.io/gorm"

type UserStore interface {
	Create(user *User) error
	Get(id int) (*User, error)
}

func NewUserStore(db *gorm.DB) UserStore {
	return &userStore{db}
}

type userStore struct {
	db *gorm.DB
}

func (s *userStore) Create(user *User) error {
	return s.db.Create(user).Error
}

func (s *userStore) Get(id int) (*User, error) {
	var user User
	err := s.db.First(&user, id).Error
	return &user, err
}
```

`store.UserStore` 定义了两个方法，分别用来创建、获取用户信息。

`User` 模型定义如下：

> store/model.go

```go
type User struct {
	ID   int    `gorm:"id"`
	Name string `gorm:"name"`
}
```

`store.userStore` 结构体则实现了 `store.UserStore` 接口。

`store.userStore` 结构体又依赖了 [GORM](https://github.com/go-gorm/gorm) 库的 `*gorm.DB` 类型，表示一个数据库连接对象。

我们可以使用 `NewMySQLDB` 建立数据库连接得到 `*gorm.DB` 对象：

> store/mysql.go

```go
func NewMySQLDB(host, port, user, pass, dbname string) (*gorm.DB, error) {
	dsn := fmt.Sprintf("%s:%s@tcp(%s:%s)/%s?charset=utf8mb4&parseTime=True&loc=Local",
		user, pass, host, port, dbname)
	return gorm.Open(mysql.Open(dsn), &gorm.Config{})
}
```

至此，这个 HTTP 服务程序整体逻辑就基本介绍完了。

其目录结构如下：

```bash
$ tree
.
├── go.mod
├── go.sum
├── main.go
└── store
    ├── model.go
    ├── mysql.go
    └── store.go
```

为了保证业务的正确性，我们应该对 `(*UserHandler).CreateUser` 和 `(*UserHandler).GetUser` 这两个 Handler 进行单元测试。

这两个 Handler 定义如下：

```go
func (h *UserHandler) CreateUser(w http.ResponseWriter, r *http.Request, _ httprouter.Params) {
	w.Header().Set("Content-Type", "application/json")

	body, err := io.ReadAll(r.Body)
	if err != nil {
		w.WriteHeader(http.StatusBadRequest)
		_, _ = fmt.Fprintf(w, `{"msg":"%s"}`, err.Error())
		return
	}
	defer func() { _ = r.Body.Close() }()

	u := store.User{}
	if err := json.Unmarshal(body, &u); err != nil {
		w.WriteHeader(http.StatusInternalServerError)
		_, _ = fmt.Fprintf(w, `{"msg":"%s"}`, err.Error())
		return
	}

	if err := h.store.Create(&u); err != nil {
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
	u, err := h.store.Get(uid)
	if err != nil {
		w.WriteHeader(http.StatusInternalServerError)
		_, _ = fmt.Fprintf(w, `{"msg":"%s"}`, err.Error())
		return
	}
	_, _ = fmt.Fprintf(w, `{"id":%d,"name":"%s"}`, u.ID, u.Name)
}
```

不过，由于文章篇幅所限，我这里仅以测试 `(*UserHandler).GetUser` 方法为例，演示如何在测试过程中解决 MySQL 依赖问题，对 `(*UserHandler).CreateUser` 方法的测试就当做作业留给你自己来完成了（当然，你也可以到我的 [GitHub](https://github.com/jianghushinian/blog-go-example/tree/main/test/mysql) 上查看我的实现）。

### Fake 测试

我们要为 `(*UserHandler).GetUser` 方法编写单元测试，首先就要分析下这个方法的外部依赖。

```go
func (h *UserHandler) GetUser(w http.ResponseWriter, r *http.Request, ps httprouter.Params) {
	id := ps[0].Value
	uid, _ := strconv.Atoi(id)

	w.Header().Set("Content-Type", "application/json")
	u, err := h.store.Get(uid)
	if err != nil {
		w.WriteHeader(http.StatusInternalServerError)
		_, _ = fmt.Fprintf(w, `{"msg":"%s"}`, err.Error())
		return
	}
	_, _ = fmt.Fprintf(w, `{"id":%d,"name":"%s"}`, u.ID, u.Name)
}
```

`UserHandler` 结构本身依赖了 `store.UserStore`，这是一个接口，定义了创建和获取用户信息的两个方法。

我们使用实现了 `store.UserStore` 接口的 `store.userStore` 结构体来初始化 `UserHandler`：

```go
func NewUserHandler(db *gorm.DB) *UserHandler {
	return &UserHandler{
		store: store.NewUserStore(db),
	}
}

func NewUserStore(db *gorm.DB) UserStore {
	return &userStore{db}
}
```

`store.userStore` 结构体会使用 GORM 来完成对 MySQL 数据库的操作。所以，我们分析出 `GetUser` 方法的第一个外部依赖实际上就是 MySQL 存储。

`GetUser` 方法还接收三个参数，它们都属于 HTTP 网络相关的外部依赖，你可以在我的另一篇文章[《在 Go 语言单元测试中如何解决 HTTP 网络依赖问题》](https://jianghushinian.cn/2023/07/15/how-to-resolve-http-dependencies-in-go-testing/)中找到解决方案，就不在本文中进行讲解了。

所以，我们现在重点要关注的就只有一个问题，如何解决 MySQL 存储依赖。

我们来整理下 MySQL 外部依赖的程序调用链：

![MySQL 外部依赖](user-handler.png)

可以发现，`store.UserStore` 接口是 `UserHandler` 和 `store.userStore` 结构体建立连接的桥梁，我们可以将它作为突破口，实现一个 Fake object，来替换 `store.userStore` 结构体。

![Fake object](fake-user-store.png)

所谓 Fake object，其实就是我们同样要定义一个结构体，并实现 `Create` 和 `Get` 两个方法，以此来实现 `store.UserStore` 接口。

```go
type fakeUserStore struct{}

func (f *fakeUserStore) Create(user *store.User) error {
	return nil
}

func (f *fakeUserStore) Get(id int) (*store.User, error) {
	return &store.User{ID: id, Name: "test"}, nil
}
```

与 `store.userStore` 结构体不同，`fakeUserStore` 并不依赖 `*gorm.DB`，也就不涉及 MySQL 数据库操作了，这样就解决了 MySQL 外部存储依赖。

`(*fakeUserStore).Create` 方法没做任何操作，直接返回 `nil`，`(*fakeUserStore).Get` 方法则根据传进来的 `id` 返回固定的 `User` 信息。这也是 Fake object 的特点，为真实对象实现一个简化版本。

这样，我们在编写测试代码时，只需要取代 `store.userStore` 结构体，使用 `fakeUserStore` 来实例化 `UserHandler`，就可以避免与 MySQL 数据库打交道了。

```go
handler := &UserHandler{store: &fakeUserStore{}}
```

为 `(*UserHandler).GetUser` 方法编写的单元测试完整代码如下：

```go
func TestUserHandler_GetUser_by_fake(t *testing.T) {
	handler := &UserHandler{store: &fakeUserStore{}}
	router := setupRouter(handler)

	w := httptest.NewRecorder()
	req := httptest.NewRequest("GET", "/users/1", nil)
	router.ServeHTTP(w, req)

	assert.Equal(t, 200, w.Code)
	assert.Equal(t, "application/json", w.Header().Get("Content-Type"))
	assert.Equal(t, `{"id":1,"name":"test"}`, w.Body.String())
}
```

现在被测试的 `(*UserHandler).GetUser` 方法中通过 `h.store.Get(uid)` 从数据库中获取用户信息时，就不用再去查询 MySQL 了，而是由 `(*fakeUserStore).Get` 方法直接返回 Fake 数据。

使用 `go test` 来执行测试函数：

```bash
$ go test -v -run="TestUserHandler_GetUser_by_fake"
=== RUN   TestUserHandler_GetUser_by_fake
--- PASS: TestUserHandler_GetUser_by_fake (0.00s)
PASS
ok      github.com/jianghushinian/blog-go-example/test/mysql    0.465s
```

测试通过。

可以发现，使用 Fake 测试来解决 MySQL 外部依赖还是比较简单的，我们仅需要参考 `store.userStore` 实现一个简化版本的 `fakeUserStore`，然后在测试过程中，使用简化版本的 `fakeUserStore` 对象替换掉 `store.userStore` 即可。

### Mock 测试

前文中，我们使用 `fakeUserStore` 来替换 `store.userStore`，以此来接口 MySQL 依赖问题。

不过，这种使用 Fake object 来解决外部依赖的方式存在两个较为常见的弊端：

一个是使用 Fake object 需要手动编写大量代码，这里的 `store.UserStore` 接口仅定义了两个方法还好，但一个线上的复杂业务，可能有几十个接口，每个接口又有几十个方法，此时如果还是手动来编写这些代码，需要消耗大量时间。

另一个是 Fake object 返回结果比较固定，如果想测试其他情况，比如查询的 `User` 不存在，需要报错的情况，就得在 `(*fakeUserStore).Get` 方法中编写更多的逻辑，这增加了实现 Fake object 的复杂度。

那么有没有一种替代方案，来弥补 Fake object 的这两个弊端呢？

答案是使用 Mock 测试。

Mock 和 Fake 类似，本质上都是使用一个对象，去替代另一个对象。Fake 测试是实现了一个真实对象（`store.userStore`）的简化版本（`fakeUserStore`），Mock 测试则是使用模拟对象来断言真实对象被调用时的输入符合预期，然后通过模拟对象返回指定输出。

在 Go 中，我们可以使用 [gomock](https://github.com/uber/mock) 来实现 Mock 测试。

`gomock` 项目起源于 Google 的 [golang/mock](https://github.com/golang/mock/) 仓库。不幸的是，谷歌不再维护这个项目了。幸运的是，这个项目由 Uber [fork](https://github.com/uber/mock) 了一份，并继续维护。

`gomock` 包含两个部分：`gomock` 包和 `mockgen` 命令行工具。`gomock` 包用来完成对被 Mock 对象的生命周期管理，`mockgen` 工具则用来自动生成 Mock 代码。

可以通过如下方式来安装 `gomock` 包和 `mockgen` 工具：

```bash
$ go get go.uber.org/mock/gomock@latest
$ go install go.uber.org/mock/mockgen@latest
```

> 注意：在项目根目录下通过 `go get` 命令获取 `gomock` 包后，不要急着执行 `go mod tidy`，因为现在 `gomock` 包属于 `indirect` 依赖，还没有被使用。当通过 `mockgen` 工具生成了 Mock 代码以后，再来执行 `go mod tidy`，`go.mod` 文件中才不会丢失 `gomock` 依赖。

要想使用 `gomock` 来模拟 `store.UserStore` 接口的实现，我们先要使用 `mockgen` 工具来生成 Mock 代码：

```bash
 $ mockgen -source store/store.go -destination store/mocks/gomock.go -package mocks
```

`-source` 参数指明需要 Mock 的接口文件路径，即 `store.UserStore` 接口所在文件。

`-destination` 参数指明生成的 Mock 文件路径。

`-package` 参数指明生成的 Mock 文件包名。

在项目根目录下执行 `mockgen` 命令，即可生成 Mock 文件：

```go
// Code generated by MockGen. DO NOT EDIT.
// Source: store/store.go

// Package mocks is a generated GoMock package.
package mocks

import (
	reflect "reflect"

	store "github.com/jianghushinian/blog-go-example/test/mysql/store"
	gomock "go.uber.org/mock/gomock"
)

// MockUserStore is a mock of UserStore interface.
type MockUserStore struct {
	ctrl     *gomock.Controller
	recorder *MockUserStoreMockRecorder
}

// MockUserStoreMockRecorder is the mock recorder for MockUserStore.
type MockUserStoreMockRecorder struct {
	mock *MockUserStore
}

// NewMockUserStore creates a new mock instance.
func NewMockUserStore(ctrl *gomock.Controller) *MockUserStore {
	mock := &MockUserStore{ctrl: ctrl}
	mock.recorder = &MockUserStoreMockRecorder{mock}
	return mock
}

// EXPECT returns an object that allows the caller to indicate expected use.
func (m *MockUserStore) EXPECT() *MockUserStoreMockRecorder {
	return m.recorder
}

// Create mocks base method.
func (m *MockUserStore) Create(user *store.User) error {
	m.ctrl.T.Helper()
	ret := m.ctrl.Call(m, "Create", user)
	ret0, _ := ret[0].(error)
	return ret0
}

// Create indicates an expected call of Create.
func (mr *MockUserStoreMockRecorder) Create(user interface{}) *gomock.Call {
	mr.mock.ctrl.T.Helper()
	return mr.mock.ctrl.RecordCallWithMethodType(mr.mock, "Create", reflect.TypeOf((*MockUserStore)(nil).Create), user)
}

// Get mocks base method.
func (m *MockUserStore) Get(id int) (*store.User, error) {
	m.ctrl.T.Helper()
	ret := m.ctrl.Call(m, "Get", id)
	ret0, _ := ret[0].(*store.User)
	ret1, _ := ret[1].(error)
	return ret0, ret1
}

// Get indicates an expected call of Get.
func (mr *MockUserStoreMockRecorder) Get(id interface{}) *gomock.Call {
	mr.mock.ctrl.T.Helper()
	return mr.mock.ctrl.RecordCallWithMethodType(mr.mock, "Get", reflect.TypeOf((*MockUserStore)(nil).Get), id)
}
```

> 提示：生成的 mocks 包代码你无需全部看懂，仅知道它大概生成了什么内容，如何使用即可。

可以发现，`mockgen` 为我们生成了 `mocks.MockUserStore` 结构体，并且实现了 `Create`、`Get` 两个方法，即实现了 `store.UserStore` 接口。

现在，我们就可以使用生成的 Mock 对象来编写单元测试代码了：

```go
func TestUserHandler_GetUser_by_mock(t *testing.T) {
	ctrl := gomock.NewController(t)
	// 断言 mockUserStore.Get 方法会被调用
	defer ctrl.Finish()

	mockUserStore := mocks.NewMockUserStore(ctrl)
	mockUserStore.EXPECT().Get(2).Return(&store.User{
		ID:   2,
		Name: "user2",
	}, nil)

	handler := &UserHandler{store: mockUserStore}
	router := setupRouter(handler)

	w := httptest.NewRecorder()
	req := httptest.NewRequest("GET", "/users/2", nil)
	router.ServeHTTP(w, req)

	assert.Equal(t, 200, w.Code)
	assert.Equal(t, "application/json", w.Header().Get("Content-Type"))
	assert.Equal(t, `{"id":2,"name":"user2"}`, w.Body.String())
}
```

`gomock.NewController(t)` 用来创建一个 Mock 控制器，该对象可以控制整个 Mock 生命周期。

`ctrl.Finish()` 用来断言 Mock 对象使用 `EXPECT()` 方法设置的期待执行方法会被调用，一般使用 `defer` 语句来调用，防止最后忘记。不过，如果你使用的 Go 版本大于 1.14，则可以不必显式调用 `ctrl.Finish()`。

`mocks.NewMockUserStore(ctrl)` 使用 Mock 控制器创建了 `*mocks.MockUserStore` 对象，有了它，我们就可以模拟调用 `store.UserStore` 接口对应方法的逻辑了：

```go
mockUserStore.EXPECT().Get(2).Return(&store.User{
    ID:   2,
    Name: "user2",
}, nil)
```

`mockUserStore` 对象就相当于我们前文中实现的 `fakeUserStore`。

Mock 对象的 `EXPECT()` 方法用来设置预期被调用的方法，以及被调用方法所期望的输入，它支持链式调用，`.Get(2)` 表示期望在测试中调用 Mock 对象 `mockUserStore` 的 `Get` 方法时，输入参数是 `2`，`Return` 方法用来设置输出，即返回值内容。

这就相当于，我们实现了 `fakeUserStore` 的 `Get` 方法。

我们可以使用 `mockUserStore` 来实例化 `UserHandler` 对象。

在 `req` 请求中，我们设置请求的用户 ID 值为 `2`，即 `mockUserStore` 对象断言中的参数，二者参数匹配，Mock 对象才能生效。

单元测试最后，断言了返回结果为 `{"id":2,"name":"user2"}`，即 `mockUserStore` 对象期望的返回结果。

现在我们就可以测试 `(*UserHandler).GetUser` 方法了。

使用 `go test` 来执行测试函数：

```bash
$ go test -v -run="TestUserHandler_GetUser_by_mock"
=== RUN   TestUserHandler_GetUser_by_mock
--- PASS: TestUserHandler_GetUser_by_mock (0.00s)
PASS
ok      github.com/jianghushinian/blog-go-example/test/mysql    0.220s
```

测试通过。

使用 Mock 测试来解决 MySQL 外部依赖问题，我们无需手动编写 Mock 对象的代码，可以使用 `mockgen` 工具为我们自动生成，简化了 Fake 测试中编写 `fakeUserStore` 的过程。

并且，如果想要测试其他情况，仅需要再次使用 Mock 对象的 `EXPECT()` 方法来设置 `Get` 方法的期望输入和输出即可。

比如设置预期查询 ID 为 `3` 的用户信息时，返回 `user not found` 错误：

```go
mockUserStore.EXPECT().Get(3).Return(nil, errors.New("user not found"))
```

Mock 测试更方便我们测试不同业务场景。

### gomock 更多用法

`gomock` 还有一些使用技巧值得分享。

#### mockgen

前文中，我们使用 `mockgen` 通过指定源码文件形式生成了 Mock 代码：

```bash
$ mockgen -source store/store.go -destination store/mocks/gomock.go -package mocks
```

`mockgen` 工具还支持通过反射模式来生成 Mock 代码：

```bash
$ mockgen -package mocks -destination store/mocks/gomock.go github.com/jianghushinian/blog-go-example/test/mysql/store UserStore
```

命令最后的两个参数分别代表需要生成 Mock 代码的包的导入路径和逗号分隔的接口列表。

执行以上命令同样能够成功生成 Mock 代码。

此外，我们还可以将 `mockgen` 命令写到 Go 文件中，然后使用 Go `generate` 工具来生成 Mock 代码：

> store/generate.go

```go
package store

//go:generate mockgen -package mocks -destination ./mocks/gomock.go . UserStore
```

这次我们的 `mockgen` 命令又有所不同，包的导入路径仅为一个 `.`，表示当前目录，这也是被支持的。

这时候，我们只需要在项目根目录下执行 `go generate ./...` 命令即可生成 Mock 代码。`./...` 表示查找项目下全部文件，`go generate` 会自动找到带有 `//go:generate` 注释的命令并执行。

如果我们有多个源码文件要生成 Mock 代码，`go generate` 方式就非常合适，仅需要在 Go 文件中分多行依次写出 `mockgen` 命令即可使用一条命令一次全部生成。


#### gomock

前文中，我们使用了 Mock 对象 `mockUserStore` 的 `EXPECT()` 方法来设置 `Get` 方法所期待的输入和输出。

```go
mockUserStore.EXPECT().Get(2).Return(&store.User{
    ID:   2,
    Name: "user2",
}, nil)
```

有时候，`EXPECT()` 所作用的方法可能存在多个参数，且有些参数不容易模拟，比如最常见的 `context.Context` 参数，针对这些情况，`gomock` 提供了更多的参数匹配方法：

`gomock.Any()` 表示匹配任意参数，适合参数模拟困难的情况。

`gomock.Eq(x)` 表示匹配与 `x` 相等的参数。

`gomock.Not(x)` 表示匹配与 `x` 不想等的参数。

`gomock.Nil()` 表示匹配 `nil` 参数。

`gomock.Len(i)` 表示匹配长度为 `i` 的参数。

`gomock.All(ms)` 表示传入的所有参数都想等才能匹配。

以上这些参数匹配方法都可以像如下这样使用：

```go
mockUserStore.EXPECT().Get(gomock.Eq(2)).Return(&store.User{
    ID:   2,
    Name: "user2",
}, nil)
```

此外，我们可以约束 `EXPECT()` 所作用方法的执行次数：

```go
.Return(xxx).Times(2) // 预期方法会被调用 2 次
.Return(xxx).MaxTimes(2) // 预期方法最多执行 2 次
.Return(xxx).MinTimes(2) // 预期方法至少执行 2 次
.Return(xxx).AnyTimes() // 预期方法执行任意次都能匹配
```

还可以约束 `EXPECT()` 所作用方法的执行顺序：

```go
.Return(xxx).After(preReq) // 当前预期方法在 preReq 预期方法执行完成之后执行
```

以上便是我认为 `gomock` 中比较常用的功能讲解，更多功能可参考[官方文档](https://pkg.go.dev/go.uber.org/mock/gomock)。

### 总结

本文向大家介绍了在 Go 中编写单元测试时，如何解决 MySQL 外部依赖的问题。

我们分别使用了 Fake object 和 Mock 两种方式，来替换原有的外部依赖。

Web 服务的代码不是随意设计的，有意将 `UserHandler` 依赖的类型设为 `store.UserStore` 接口，而不是 `store.userStore` 结构体，是为了解耦。通过使用接口，解决了 `UserHandler` 与 `store.userStore` 结构体强绑定的问题，这就给我们使用 `fakeUserStore` 或 `mockUserStore` 来替代 `store.userStore` 创造了机会。

可以发现，本文介绍的两种方法其实不仅能够用于解决 MySQL 外部依赖问题。任何使用接口编写的代码，在测试时都可以使用这两种方式来替换依赖。这就是 Go 面向接口编程的好处。

本文完整代码示例我放在了 [GitHub](https://github.com/jianghushinian/blog-go-example/tree/main/test/mysql) 上，欢迎点击查看。

希望此文能对你有所帮助。

**联系我**

- 微信：jianghushinian
- 邮箱：[jianghushinian007@outlook.com](mailto:jianghushinian007@outlook.com)
- 博客地址：https://jianghushinian.cn

**参考**

- gomock 源码：https://github.com/golang/mock/
- Uber gomock 源码：https://github.com/uber/mock
- Uber gomock 文档：https://pkg.go.dev/go.uber.org/mock/gomock
