---
title: '在 Go 中为什么推荐使用空结构体作为 Context 的 key'
date: 2025-06-08 17:19:25
tags:
- Go
categories:
- Go
---

我曾在《[Go 中空结构体惯用法，我帮你总结全了！](https://mp.weixin.qq.com/s/Qi0RrRyHLx8Q4SmhQeb-uQ)》一文中介绍过空结构体的多种用法，本文再来补充一种惯用法：将空结构体作为 Context 的 `key` 来进行安全传值。

<!-- more -->

> NOTE:
>
> 如果你对 Go 语言中的 Context 不够熟悉，可以阅读我的另一篇文章《[Go 并发控制：context 源码解读](https://mp.weixin.qq.com/s/wPXzISELBc33vjvxPvQY5w)》。
>

### 使用 Context 进行传值

我们知道 Context 主要有两种用法，控制链路和安全传值。

在此我来演示下如何使用 Context 进行安全传值：

```go
package main

import (
	"context"
	"fmt"
)

const requestIdKey = "request-id"

func main() {
	ctx := context.Background()

	// NOTE: 通过 context 传递 request id 信息
	// 设置值
	ctx = context.WithValue(ctx, requestIdKey, "req-123")
	// 获取值
	fmt.Printf("request-id: %s\n", ctx.Value(requestIdKey))
}
```

通过 Context 进行传值的方式非常简单，`context.WithValue(ctx, key, value)` 函数可以为一个已存在的 `ctx` 对象，附加一个键值对（注意：这里的 `key` 和 `value` 都是 `any` 类型）。然后，可以使用 `ctx.Value(key)` 来获取 `key` 对应的 `value`。

这里我们使用字符串 `request-id` 作为 `key`，值为 `req-123`。

执行示例代码，得到输出如下：

```bash
$ go run main.go                                                            
request-id: req-123
```

题外话，之所以说 Context 可以进行安全传值，是因为它的源码实现是并发安全的，你可以在《[Go 并发控制：context 源码解读](https://mp.weixin.qq.com/s/wPXzISELBc33vjvxPvQY5w)》中学习其实现原理。

但是，从代码编写者的角度来说，通过 Context 传值并不都是“安全”的，咱们接着往下看。

### key 冲突问题

既然 Context 的 `key` 可以是任意类型，那么固然也可以是任意值。我们在写代码的时候，经常会用一些如 `i`、`info`、`data` 等作为变量，那么我们也很有可能使用 `data` 这样类似的字符串值作为 Context 的 `key`：

```go
package main

import (
	"context"
	"fmt"
)

// NOTE: 这个 key 非常容易冲突
const dataKey = "data"

func main() {
	ctx := context.Background()

	ctx = context.WithValue(ctx, dataKey, "some data")

	fmt.Printf("data: %s\n", ctx.Value(dataKey))

	userDataKey := "data" // 与 dataKey 值相同
	ctx = context.WithValue(ctx, userDataKey, "user data")

    fmt.Printf("user data: %s\n", ctx.Value(userDataKey))

    // 再次查看 dataKey 的值
	fmt.Printf("data: %s\n", ctx.Value(dataKey))
}
```

在这个示例中，`dataKey` 的值为 `data`，我们将其作为 `key` 存入 Context。稍后，又将 `userDataKey` 变量作为 Context 的 `key` 存入 Context。最终，`ctx.Value(dataKey)` 返回的值是什么呢？

执行示例代码，得到输出如下：

```bash
$ data: some data
user data: user data
data: user data
```

结果很明显，虽然 `dataKey` 和 `userDataKey` 这两个 `key` 的变量名不一样，但是它们的值同为 `data`，最终导致 `ctx.Value(dataKey)` 返回的结果与 `ctx.Value(userDataKey)` 返回结果相同。

即 `dataKey` 的值已经被 `userDataKey` 所覆盖。所以我才说，通过 Context 传值并不都是“安全”的，因为你的键值对可能会被覆盖。

### 解决 key 冲突问题

如何有效避免 Context 传值是 `key` 冲突的问题呢？

最简单的方案，就是为 `key` 定义一个具有业务属性的前缀，比如用户相关的数据 `key` 为 `user-data`，文章相关的数据 `key` 为 `post-data`：

```go
package main

import (
	"context"
	"fmt"
)

// NOTE: 为了避免 key 冲突，我们通常可以为 key 定义一个业务属性的前缀
const (
	userDataKey = "user-data"
	postDataKey = "post-data"
)

func main() {
	ctx := context.Background()

	ctx = context.WithValue(ctx, userDataKey, "user data")

	fmt.Printf("user-data: %s\n", ctx.Value(userDataKey))

	ctx = context.WithValue(ctx, postDataKey, "post data")

	fmt.Printf("post-data: %s\n", ctx.Value(postDataKey))
}
```

执行示例代码，得到输出如下：

```bash
$ go run main.go
user-data: user data
post-data: post data
```

这样不同业务的 `key` 就不会互相干扰了。

你也许还想到了更好的方式，比如将所有 `key` 定义为常量，并统一放在一个叫 `constant` 的包中，这也是一种很常见的解决问题的方式。

但我个人极其不推荐这种做法，虽然表面上看将所有常量统一放在一个包中，集中管理，更方便维护。但当常量一多，这个包简直是灾难。Go 更推崇将代码中的变量、常量等定义在其使用的地方，而不是统一放在一个文件中管理，为自己增加心智负担。

其实我们还有更加优雅的解决办法，是时候让空结构体登场了。

### 使用空结构体作为 key

Context 的 `key` 可以是任意类型，那么空结构体就是绝佳方案。

我们可以使用空结构体定义一个自定义类型，然后作为 Context 的 `key`：

```go
package main

import (
	"context"
	"fmt"
)

// NOTE: 使用空结构体作为 context key
type emptyKey struct{}
type anotherEmpty struct{}

func main() {
	ctx := context.Background()

	ctx = context.WithValue(ctx, emptyKey{}, "empty struct data")

	fmt.Printf("empty data: %s\n", ctx.Value(emptyKey{}))

	ctx = context.WithValue(ctx, anotherEmpty{}, "another empty struct data")

	fmt.Printf("another empty data: %s\n", ctx.Value(anotherEmpty{}))

	// 再次查看 emptyKey 对应的 value
	fmt.Printf("empty data: %s\n", ctx.Value(emptyKey{}))
}
```

执行示例代码，得到输出如下：

```bash
$ go run main.go
empty data: empty struct data
another empty data: another empty struct data
empty data: empty struct data
```

这一次，没有出现覆盖情况。

### 空结构体作为 key 的错误用法

现在，我们来换一种用法，不再自定义新的类型，而是直接将空结构体变量作为 Context 的 `key`，示例如下：

```go
package main

import (
	"context"
	"fmt"
)

// NOTE: 空结构体作为 context key 的错误用法

func main() {
	ctx := context.Background()

	key1 := struct{}{}
	ctx = context.WithValue(ctx, key1, "data1")
	fmt.Printf("key1 data: %s\n", ctx.Value(key1))

	key2 := struct{}{}
	ctx = context.WithValue(ctx, key2, "data2")
	fmt.Printf("key2 data: %s\n", ctx.Value(key2))

	// 再次查看 key1 对应的 value
	fmt.Printf("key1 data: %s\n", ctx.Value(key1))
}
```

执行示例代码，得到输出如下：

```bash
$ go run main.go
key1 data: data1
key2 data: data2
key1 data: data2
```

可以发现，这次又出现了 `key2` 键值对覆盖 `key1` 键值对的情况。

所以，你有没有注意到，使用空结构体作为 Context 的 `key`，最关键的步骤，其实是要基于空结构体定义一个**新的类型**。我们使用这个**新类型的实例对象**作为 `key`，而不是直接使用空**结构体变量**作为 `key`，**这二者是有本质区别的**。

这是两个不同的类型：

```go
type emptyKey struct{}
type anotherEmpty struct{}
```

它们相同点只不过是二者都没有属性和方法，都是一个空的结构体。

而 `emptyKey{}` 和 `anotherEmpty{}` 一定**不相等**，因为它们是**不同的类型**。

这是两个空结构体变量：

```go
key1 := struct{}{}
key2 := struct{}{}
```

显然，`key1` **等于** `key2`，因为它们的**值相等**，并且**类型也相同**。

有很多人看到将空结构体作为 Context 的 `key` 时，第一想法是担心冲突，以为空结构体作为 `key` 时只能保存一个键值对，设置多个键值对时，后面的空结构体 `key` 会覆盖之前的空结构体 `key`。

实则不然，每一个 `key` 都是一个新的类型，而非常量（`const`），这一点很重要，非常容易让人误解。

很多文章或教程并没有强调这一点，所以导致很多人看了教程以后，觉得这个特性没什么用，或产生困惑。其实这也是编程的魅力所在，编程是一门非常注重实操的学科，看一遍和写一遍完全是不同概念，这决定了你对某项技术理解程度。

### 使用自定义类型作为 key

既然我们在使用空结构体作为 Context 的 `key` 时，是定义了一个新的类型，那么我们是否也可以使用其他自定义类型作为 Context 的 `key` 呢？

答案是肯定的，因为 Context 的 `key` 本身就是 `any` 类型。

使用基于 `string` 自定义类型，作为 Context 的 `key` 示例如下：

```go
package main

import (
	"context"
	"fmt"
)

// NOTE: 基于 string 自定义类型，作为 context key
type key1 string
type key2 string

func main() {
	ctx := context.Background()

	ctx = context.WithValue(ctx, key1(""), "data1")
	fmt.Printf("key1 data: %s\n", ctx.Value(key1("")))

	ctx = context.WithValue(ctx, key2(""), "data2")
	fmt.Printf("key2 data: %s\n", ctx.Value(key2("")))

	// 再次查看 key1 对应的 value
	fmt.Printf("key1 data: %s\n", ctx.Value(key1("")))
}
```

那么你认为这段代码执行结果如何呢？这就交给你自行去测试了。

### 项目实战

上面介绍了几种可以作为 Context 的 `key` 进行传值的做法，有推荐做法，也有踩坑做法。

接下来我们一起看下真实的企业级项目 [OneX](https://github.com/onexstack/onex) 中是如何定义和使用 Context 进行安全传值的。

#### 使用空结构体作为 key 在 Context 中传递事务

OneX 中巧妙的使用了 Context 来传递 GORM 的事务对象 `tx`，其实现如下：

> [https://github.com/onexstack/onex/blob/feature/onex-v2/internal/usercenter/store/store.go#L40](https://github.com/onexstack/onex/blob/feature/onex-v2/internal/usercenter/store/store.go#L40)
>

```go
// transactionKey is the key used to store transaction context in context.Context.
type transactionKey struct{}

// NewStore initializes a singleton instance of type IStore.
// It ensures that the datastore is only created once using sync.Once.
func NewStore(db *gorm.DB) *datastore {
	// Initialize the singleton datastore instance only once.
	once.Do(func() {
		S = &datastore{db}
	})

	return S
}

// DB filters the database instance based on the input conditions (wheres).
// If no conditions are provided, the function returns the database instance
// from the context (transaction instance or core database instance).
func (store *datastore) DB(ctx context.Context, wheres ...where.Where) *gorm.DB {
	db := store.core
	// Attempt to retrieve the transaction instance from the context.
	if tx, ok := ctx.Value(transactionKey{}).(*gorm.DB); ok {
		db = tx
	}

	// Apply each provided 'where' condition to the query.
	for _, whr := range wheres {
		db = whr.Where(db)
	}
	return db
}

// TX starts a new transaction instance.
// nolint: fatcontext
func (store *datastore) TX(ctx context.Context, fn func(ctx context.Context) error) error {
	return store.core.WithContext(ctx).Transaction(
		func(tx *gorm.DB) error {
			ctx = context.WithValue(ctx, transactionKey{}, tx)
			return fn(ctx)
		},
	)
}
```

OneX 的 `store` 层用来操作数据库进行 CRUD，你可以阅读令飞老师的《[简洁架构设计：如何设计一个合理的软件架构？](https://mp.weixin.qq.com/s/sYT4IhDoklGWZQ9btSB-qw)》这篇文章来了解 OneX 的架构设计。

在 `store` 的源码中定义了空结构体类型 `transactionKey` 作为 Context 的 `key`，在调用 `store.TX` 方法时，方法内部会将事务对象 `tx` 作为 `value` 保存到 Context 中。

在调用 `store.DB` 对数据库进行操作时，就会优先判断当前是否处于事务中，如果 `ctx.Value(transactionKey{})` 有值，则说明当前事务正在进行，使用 `tx` 对象继续操作，否则说明是一个简单的数据库操作，直接返回 `db` 对象。

使用示例如下：

> [https://github.com/onexstack/onex/blob/feature/onex-v2/internal/usercenter/biz/v1/user/user.go#L69](https://github.com/onexstack/onex/blob/feature/onex-v2/internal/usercenter/biz/v1/user/user.go#L69)
>

```go
// Create implements the Create method of the UserBiz.
func (b *userBiz) Create(ctx context.Context, rq *v1.CreateUserRequest) (*v1.CreateUserResponse, error) {
	var userM model.UserM
	_ = core.Copy(&userM, rq) // Copy request data to the User model.

	// Start a transaction for creating the user and secret.
	err := b.store.TX(ctx, func(ctx context.Context) error {
		// Attempt to create the user in the data store.
		if err := b.store.User().Create(ctx, &userM); err != nil {
			// Handle duplicate entry error for username.
			match, _ := regexp.MatchString("Duplicate entry '.*' for key 'username'", err.Error())
			if match {
				return v1.ErrorUserAlreadyExists("user %q already exists", userM.Username)
			}
			return v1.ErrorUserCreateFailed("create user failed: %s", err.Error())
		}

		// Create a secret for the newly created user.
		secretM := &model.SecretM{
			UserID:      userM.UserID,
			Name:        "generated",
			Expires:     0,
			Description: "automatically generated when user is created",
		}
		if err := b.store.Secret().Create(ctx, secretM); err != nil {
			return v1.ErrorSecretCreateFailed("create secret failed: %s", err.Error())
		}

		return nil
	})
	if err != nil {
		return nil, err // Return any error from the transaction.
	}

	return &v1.CreateUserResponse{UserID: userM.UserID}, nil
}
```

当创建用户对象 `user` 时，会同步创建一条 `secret` 记录，此时就用到了事务。

`b.store.TX` 用来开启事务，`b.store.User().Create()` 创建 `user` 对象时，`Create()` 方法内部，其实会返回 `tx` 对象，对于`b.store.Secret().Create()` 的调用同理，这样就完成了事务操作。

OneX 实际上实现了一个泛型版本的 `Create()` 方法：

> [https://github.com/onexstack/onex/blob/feature/onex-v2/staging/src/github.com/onexstack/onexstack/pkg/store/store.go#L63](https://github.com/onexstack/onex/blob/feature/onex-v2/staging/src/github.com/onexstack/onexstack/pkg/store/store.go#L63)
>

```go
// Create inserts a new object into the database.
func (s *Store[T]) Create(ctx context.Context, obj *T) error {
	if err := s.db(ctx).Create(obj).Error; err != nil {
		s.logger.Error(ctx, err, "Failed to insert object into database", "object", obj)
		return err
	}
	return nil
}

// db retrieves the database instance and applies the provided where conditions.
func (s *Store[T]) db(ctx context.Context, wheres ...where.Where) *gorm.DB {
	dbInstance := s.storage.DB(ctx)
	for _, whr := range wheres {
		if whr != nil {
			dbInstance = whr.Where(dbInstance)
		}
	}
	return dbInstance
}
```

无论是调用 `b.store.User().Create()`，还是调用 `b.store.Secret().Create()`，其实最终都会调用此方法，而 `s.db(ctx)` 内部又调用了 `s.storage.DB(ctx)`，这里的 `DB` 方法，其实就是 `*datastor.DB`。

至此，在 OneX 中使用空结构体作为 `key` 在 Context 中传递事务的主体脉络就理清了。如果你对 OneX 整体架构不够清晰，可能对这部分的讲解比较困惑，那么可以看看其源码，这是一个非常优秀的开源项目。

#### contextx 包

此外，OneX 项目还专门抽象出一个 `contextx` 包，用来定义公共的 Context 操作。

比如可以使用 Context 传递 `userID`：

> [https://github.com/onexstack/onex/blob/feature/onex-v2/internal/pkg/contextx/contextx.go](https://github.com/onexstack/onex/blob/feature/onex-v2/internal/pkg/contextx/contextx.go)
>

```go
type (
	userKey        struct{}
	...
)

// WithUserID put userID into context.
func WithUserID(ctx context.Context, userID string) context.Context {
	return context.WithValue(ctx, userKey{}, userID)
}

// UserID extract userID from context.
func UserID(ctx context.Context) string {
	userID, _ := ctx.Value(userKey{}).(string)
	return userID
}
...
```

`contextx` 包中有很多类似实现，你可以查看源码学习更多使用技巧。

### 总结

本文讲解了在 Go 中使用空结构体作为 Context 的 `key` 进行安全传值的小技巧。虽然这是一个不太起眼的小技巧，并且面试中也不会被问到，但正是这些微小的细节，决定了你写的代码最终质量。如果你想写出优秀的项目，那么每一个细节都值得深入思考，找到更优解。

使用 Context 对象传值时，其 `key` 可以是任意类型，那么为什么使用空结构体是更好的选择呢？因为空结构体不占内存空间，并且满足了唯一性的要求，所以空结构体是最优解。

并且，我们还可以像 `contextx` 包那样，将常用的传值操作放在一起，对外只暴露 `Set/Get` 两个方法，而将 Context 的 `key` 定义为**未导出（unexported）类型**，那么就不可能出现 `key` 冲突的情况。

当然，我们应该尽量避免使用 Context 来传递值，只在必要时使用。显式胜于隐式，当隐式代码变多，也将是灾难。

本文示例源码我都放在了 [GitHub](https://github.com/jianghushinian/blog-go-example/tree/main/struct/empty/context-key) 中，欢迎点击查看。

希望此文能对你有所启发。

**延伸阅读**

+ Go 中空结构体惯用法，我帮你总结全了！：[https://mp.weixin.qq.com/s/Qi0RrRyHLx8Q4SmhQeb-uQ](https://mp.weixin.qq.com/s/Qi0RrRyHLx8Q4SmhQeb-uQ)
+ Go 并发控制：context 源码解读：[https://mp.weixin.qq.com/s/wPXzISELBc33vjvxPvQY5w](https://mp.weixin.qq.com/s/wPXzISELBc33vjvxPvQY5w)
+ Golang Tip #37: Using Unexported Empty Struct as Context Key：[https://x.com/func25/status/1763897057511899516](https://x.com/func25/status/1763897057511899516)
+ OneX GitHub 源码：[https://github.com/onexstack/onex](https://github.com/onexstack/onex)
+ 简洁架构设计：如何设计一个合理的软件架构？：[https://mp.weixin.qq.com/s/sYT4IhDoklGWZQ9btSB-qw](https://mp.weixin.qq.com/s/sYT4IhDoklGWZQ9btSB-qw)
+ 本文 GitHub 示例代码：[https://github.com/jianghushinian/blog-go-example/tree/main/struct/empty/context-key](https://github.com/jianghushinian/blog-go-example/tree/main/struct/empty/context-key)
+ 本文永久地址：[https://jianghushinian.cn/2025/06/08/empty-struct-as-key-of-ctx/](https://jianghushinian.cn/2025/06/08/empty-struct-as-key-of-ctx/)

**联系我**

+ 公众号：[Go编程世界](https://jianghushinian.cn/about)
+ 微信：jianghushinian
+ 邮箱：[jianghushinian007@outlook.com](mailto:jianghushinian007@outlook.com)
+ 博客：[https://jianghushinian.cn](https://jianghushinian.cn)
+ GitHub：[https://github.com/jianghushinian](https://github.com/jianghushinian)
