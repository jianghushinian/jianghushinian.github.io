---
title: 测试代码终极解决方案 Monkey Patching
date: 2023-07-22 15:57:26
tags:
- Go
- Test
categories:
- Go
---

前面几篇文章，我讲解了在 Go 语言中如何编写测试代码，因为有时候我们编写的代码难以测试，我又写了[一篇文章](http://localhost:4000/2023/07/22/how-to-write-testable-code-in-go/)专门讲解在 Go 语言中如何编写出可测试的代码。

但有些时候，我们可能需要维护早期编写的“烂代码”，这些代码不方便测试，可维护阶段需要修改代码，为了验证代码功能正常，我们又不得不补充测试。针对这种情况，本文将向大家介绍一种测试代码的终极解决方案 —— Monkey Patching。

<!-- more -->

### 简介

Monkey Patching 翻译过来叫猴子补丁，如果你写过 Python、JavaScript 等动态语言代码，想必对猴子补丁不会太陌生。如果你对猴子补丁不太了解，可以看下我的另一篇文章[《Python 中的猴子补丁》](https://jianghushinian.cn/2020/05/16/Python-%E4%B8%AD%E7%9A%84%E7%8C%B4%E5%AD%90%E8%A1%A5%E4%B8%81/)。

如果你对在 Go 这种静态编程语言中，如何实现 Monkey Patching 比较感兴趣，可以看下[这篇文章](https://bou.ke/blog/monkey-patching-in-go/)。

### HTTP 服务程序示例

假设我们有一个 HTTP 服务程序对外提供服务，代码如下：

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

`GET /users/:id` 用来获取指定 ID 对应的用户信息。

为了保证业务的正确性，我们应该对 `(*UserHandler).CreateUser` 和 `(*UserHandler).GetUser` 这两个 Handler 进行单元测试。

### 使用 Monkey Patching 编写测试

这里以 `(*UserHandler).CreateUser` 为例进行讲解如何使用 Monkey Patching 编写测试。

先来分析下这个方法的依赖项：

首先 `UserHandler` 这个结构体本身有一个 `store` 属性，依赖了 `*gorm.DB` 对象。

其次，`CreateUser` 方法还接收三个参数，它们都属于 HTTP 网络相关的外部依赖，你可以在我的另一篇文章[《在 Go 语言单元测试中如何解决 HTTP 网络依赖问题》](https://jianghushinian.cn/2023/07/15/how-to-resolve-http-dependencies-in-go-testing/)中找到解决方案，就不在本文中进行讲解了。

所以，我们应该要想办法解决 `*gorm.DB` 这个外部依赖。

由于我们编写代码时，没有考虑如何编写测试，所以就没有使用接口来进行解耦，导致 `UserHandler` 结构体直接依赖了 `*gorm.DB` 结构体对象。

在不改变代码的前提下，我们可以使用 Monkey Patching 技术为依赖对象 `*gorm.DB` 打上猴子补丁，以此来解决测试代码中难以调用 `h.store.First(&u, uid).Error` 方法问题。

我们可以使用 `gomonkey` 来实现 Monkey Patching，使用如下命令安装：

```bash
$ go get github.com/agiledragon/gomonkey/v2
```

使用 `gomonkey` 为 `(*UserHandler).CreateUser` 方法编写的测试代码如下：

```go
func TestUserHandler_CreateUser(t *testing.T) {
	mysqlDB := &gorm.DB{}
	handler := NewUserHandler(mysqlDB)
	router := setupRouter(handler)

	// 为 mysqlDB 打上猴子补丁，替换其 Create 方法
	patches := gomonkey.ApplyMethod(reflect.TypeOf(mysqlDB), "Create",
		func(in *gorm.DB, value interface{}) (tx *gorm.DB) {
			expected := &User{
				Name: "user1",
			}
			actual := value.(*User)
			assert.Equal(t, expected, actual)
			return in
		})
	// 测试执行完成后将猴子补丁复原
	defer patches.Reset()

	w := httptest.NewRecorder()
	req := httptest.NewRequest("POST", "/users", strings.NewReader(`{"name": "user1"}`))
	router.ServeHTTP(w, req)

	assert.Equal(t, 201, w.Code)
	assert.Equal(t, "application/json", w.Header().Get("Content-Type"))
	assert.Equal(t, "", w.Body.String())
}
```

首先我们直接使用 `&gorm.DB{}` 创建了一个 `*gorm.DB` 对象，注意这里并没有通过 `NewMySQLDB` 方法来打开一个真正的数据库连接，这仅仅是一个空对象。

然后将其传递给 `NewUserHandler` 来完成构造 `*UserHandler` 对象的正常流程。

接下来，我们要重点关注的是如下这部分代码：

```go
// 为 mysqlDB 打上猴子补丁，替换其 Create 方法
patches := gomonkey.ApplyMethod(reflect.TypeOf(mysqlDB), "Create",
    func(in *gorm.DB, value interface{}) (tx *gorm.DB) {
        expected := &User{
            Name: "user1",
        }
        actual := value.(*User)
        assert.Equal(t, expected, actual)
        return in
    })
// 测试执行完成后将猴子补丁复原
defer patches.Reset()
```

我们使用 `gomonkey` 库的 `ApplyMethod` 方法，为 `mysqlDB` 对象的 `Create` 方法打了一个猴子补丁，然后使用匿名函数来实现这个 `Create` 方法，并且，在匿名函数的内部还对 `Create` 方法接收到的参数进行了验证。

`gomonkey.ApplyMethod` 方法返回一个 `*gomonkey.Patches` 对象，使用 `defer` 语句延迟调用 `patches.Reset()`，可以在测试执行完成后将被 Monkey Patching 的对象进行还原。

这就是猴子补丁的强大，它能原地修改 `mysqlDB.Create` 方法的实现。

使用 `go test` 来执行测试函数：

```bash
GOARCH=amd64 go test -gcflags=all=-l -p 1 -v
=== RUN   TestUserHandler_CreateUser
--- PASS: TestUserHandler_CreateUser (0.02s)
PASS
ok      github.com/jianghushinian/blog-go-example/test/monkeypatching   0.675s
```

测试通过。

注意，在执行测试时，我指定了 `GOARCH=amd64` 环境变量。这是因为我的主机是 Apple M2 芯片的 ARM 平台，如果你是 X86 平台则无需指定此环境变量。

此外，我们还为 `go test` 命令指定了两个特殊参数：

`-gcflags=all=-l` 参数是用来关闭 Go 语言[内联优化](https://zh.wikipedia.org/wiki/%E5%86%85%E8%81%94%E5%B1%95%E5%BC%80)的。默认情况下，Go 在构建代码时会进行内联优化，但是 `gomonkey` 并不支持这一功能，这与其实现原理有关。

`-p 1` 参数可以将执行测试的代码并发数置为 1。这是由于 `gomonkey` 不是并发安全的，这同样与其实现原理有关。

虽然执行测试代码时需要多传递两个参数，但 `gomonkey` 为我们提供的便利性远大于这点小麻烦。

### 总结

本文介绍了一种编写测试代码的终极解决方案 Monkey Patching，使用这项技术，可以在不手动修改程序代码的情况下，来完成对某个对象的原地替换。

`gomonkey` 库非常强大，它不仅能够为结构体的方法打上猴子补丁，它还支持为一个函数、一个全局变量、一个函数变量等打上猴子补丁，更多方法可以参考[这篇文章](https://www.jianshu.com/p/633b55d73ddd)。

不过使用 `gomonkey` 也有很多缺点，它不支持 Go 语言的内联优化，也不支持并发的执行测试代码，并且对于 ARM 平台支持不够完善。

所以，我们应该视情况来考虑是否要使用 `gomonkey`。

本文完整代码示例我放在了 [GitHub](https://github.com/jianghushinian/blog-go-example/tree/main/test/monkeypatching) 上，欢迎点击查看。

希望此文能对你有所帮助。

### P.S.

其实，对于是否要写下这篇文章我是很犹豫的，因为我不推荐在 Go 中使用 Monkey Patching 技术，引入 Monkey Patching 就意味着代码里存在“坏味道”。但是，有些时候，我们工作中总要跟“烂代码”做斗争，当重构代码代价大于收益时，我们还是要有一种方案来解决难以编写测试代码的问题，Monkey Patching 就是我们编写测试的终极解决方案。

`gomonkey` 库是一位国人开发的，其思想起源于 [monkey](https://github.com/bouk/monkey) 项目。`monkey` 库的作者虽然创造了在 Go 语言中实现 Monkey Patching 的技术，但是他却不推荐使用 `monkey`，`monkey` 在创建之初就存在争议，可以在 [Hacker News](https://news.ycombinator.com/item?id=9290917)上看到当时的讨论。并且，作者最终将 `monkey` 库的许可证设为了不允许他人使用，可以参考[这篇文章](https://esoteric.codes/blog/bouk-monkey-satirical-code-used-by-people-who-dont-get-the-joke)，有趣的是，文章结尾作者推荐了 `gomonkey` 项目。

最后，还是要提醒大家，不到万不得已，不推荐使用猴子补丁解决问题。

**联系我**

- 微信：jianghushinian
- 邮箱：[jianghushinian007@outlook.com](mailto:jianghushinian007@outlook.com)
- 博客地址：https://jianghushinian.cn/

**参考**

- Monkey Patching in Go：https://bou.ke/blog/monkey-patching-in-go/
- gomonkey：https://github.com/agiledragon/gomonkey
- gomonkey 1.0 正式发布！：https://www.jianshu.com/p/633b55d73ddd
- 你该刷新gomonkey的惯用法了：https://www.jianshu.com/p/25d49af216b7
