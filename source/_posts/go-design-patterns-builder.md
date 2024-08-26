---
title: 'Builder 模式在 Go 语言中的应用'
date: 2024-08-26 19:56:20
tags:
- Go
- 设计模式
categories:
- Go
---

Builder 模式是一种创建型模式，即用来创建对象。

Builder 模式，中文翻译不太统一，有时候被翻译为建造者模式或构建者模式，有时候也被翻译为生成器模式。为了不给读者造成困扰，我还是直接叫它 Builder 模式好了。

[《设计模式：可复用面向对象软件的基础》](https://book.douban.com/subject/34262305/) 一书中对 Builder 模式的意图阐明如下：

> 将一个复杂对象的构建与它的表示分离，使得同样的构建过程可以创建不同的表示。

<!-- more -->

### 经典 Builder 模式

假设我们要建造一个大 `House`，其结构体定义如下：

```go
// House 代表一个由多个部分构建的复杂对象
type House struct {
	// 地基
	Foundation string
	// 墙壁
	Walls string
	// 屋顶
	Roof string
}
```

通常我们会定义如下构造函数：

```go
// NewHouse 一个普通的 House 构造函数
func NewHouse(foundation, walls, roof string) *House {
	return &House{
		Foundation: foundation,
		Walls:      walls,
		Roof:       roof,
	}
}
```

可以使用构造函数来创建 `House`：

```go
house := NewHouse("Concrete Foundation", "Wooden Walls", "Shingle Roof")
fmt.Printf("%+v\n", house)
```

现在，如果我们再为这个 `House` 增加一些属性的话，`NewHouse` 的参数列表就会变得很长。并且，如果有些参数是带有默认值的，有些参数是必传的，`NewHouse` 用起来就会比较别扭。

此时，有经验的读者应该会想到使用 Options 模式。没错，这是一个在 Go 语言中非常流行的设计模式，能够很好的解决可选参数问题。

> NOTE:
> 如果你对 Go 的 Options（选项）模式不太熟悉，可以参考我的另一篇文章 [《Go 常见设计模式之选项模式》](https://jianghushinian.cn/2021/12/12/Golang-常见设计模式之选项模式/)。

不过，咱们今天不讲如何使用 Options 模式来解决此类问题，今天来看看如何使用 Builder 模式来解决这个问题。

要使用 Builder 模式，首先我们就要定义一个 `Builder` 接口：

```go
// Builder 用于构建 House 对象的接口
type Builder interface {
	BuildFoundation()
	BuildWalls()
	BuildRoof()
	GetResult() *House
}
```

`Builder` 接口声明了构建 `House` 对象都有哪些行为。

其包含 4 个方法，`BuildFoundation` 用来构建地基，`BuildWalls` 用来构建墙壁，`BuildRoof` 用来构建屋顶，最后还有一个 `GetResult` 方法用来获取构建完成的 `House` 对象。

这是 Builder 模式的惯用法，定义若干个 `BuildXxx` 方法用来构建对象的属性，最后再定义一个 `GetXxx` 方法用来获取被构建的对象。

我们再定义一个 `ConcreteBuilder` 结构体，用来实现 `Builder` 接口：

```go
// ConcreteBuilder Builder 接口的具体实现，用于构建具体的 House
type ConcreteBuilder struct {
	house *House
}

// NewConcreteBuilder 创建一个新的 ConcreteBuilder 实例
func NewConcreteBuilder() *ConcreteBuilder {
	return &ConcreteBuilder{house: &House{}}
}

// BuildFoundation 构建地基
func (b *ConcreteBuilder) BuildFoundation() {
	b.house.Foundation = "Concrete Foundation"
}

// BuildWalls 构建墙壁
func (b *ConcreteBuilder) BuildWalls() {
	b.house.Walls = "Wooden Walls"
}

// BuildRoof 构建屋顶
func (b *ConcreteBuilder) BuildRoof() {
	b.house.Roof = "Shingle Roof"
}

// GetResult 返回构建完成的 House 对象
func (b *ConcreteBuilder) GetResult() *House {
	return b.house
}
```

在典型的 Builder 设计模式中，我们还差一个 `Director` 对象，定义如下：

```go
// Director 用于控制构建过程的指挥者
type Director struct {
	builder Builder
}

// NewDirector 创建一个新的 Director 实例
func NewDirector(builder Builder) *Director {
	return &Director{builder: builder}
}

// Construct 构建 House 的方法
func (d *Director) Construct() {
	d.builder.BuildFoundation()
	d.builder.BuildWalls()
	d.builder.BuildRoof()
}
```

`Director` 是一个用于控制构建过程的「指挥者」，它依赖一个 `Builder` 接口类型的对象，传入不同的 `Builder`，可以实现不同的行为，这也是依赖注入的思想。

通过 `Director` 对象，我们可以控制构建 `House` 的整个过程，`Construct` 方法就是用来干这个的。

使用 Builder 模式创建一个大 `House` 流程如下：

```go
// 创建具体的 Builder
builder := NewConcreteBuilder()

// 创建 Director 并传入具体的 Builder
director := NewDirector(builder)

// 通过 Director 控制构建过程
director.Construct()

// 获取构建的最终产品
house := builder.GetResult()

// 输出构建好的 House
fmt.Printf("%+v\n", house)
```

这就是 Builder 模式的经典写法。

**这里涉及几个主要角色：**

- `Builder` 接口是一个用于构建 `House` 对象的接口，它抽象了构建 `House` 各个属性的方法，以及一个获取构建完成的 `House` 对象的方法。

- `ConcreteBuilder` 结构体是 `Builder` 接口的具体实现，其内部保存了 `House` 结构体。

如果你不嫌麻烦，有人也会这么定义它：

```go
type ConcreteBuilder struct {
	// 地基
	Foundation string
	// 墙壁
	Walls string
	// 屋顶
	Roof string
}
```

此时 `GetResult` 方法应该怎么写就留给你去思考了。

- `Director` 结构体是一个指挥者，主要功能就是负责执行生成步骤，即它的核心功能在于 `Construct` 方法的定义。我们可以根据需要来决定传入不同的 `Builder` 对象。

可以发现，在标准的 Builder 设计模式中，`Builder` 的方法通常是没有参数的，每个方法负责一步固定的构建过程。这种设计的好处是过程非常清晰，步骤明确，但它的灵活性较低。

我称这种写法为经典的 Builder 设计模式。

在这种情况下，每个步骤的实现通常是固定的，比如所有使用 `ConcreteBuilder` 构造的的 `House` 都使用同样的地基、墙壁和屋顶材料。

要想创建一个具有不同属性的 `House`，我们就需要实现另外一个新的 `ConcreteBuilder1`。

为了增加灵活性，可以在 `Builder` 的方法中添加参数，以允许在构建过程中设置不同的属性值。

具体如何实现你可以先思考下，我们在接下来的示例中会进行讲解。

### 简化版 Builder 模式

经典版本的 Builder 模式代码看起来有点“死板”，因为这是我模仿 Java 版本“抄”过来的。

我们接下来介绍下 Go 语言特色版本 Builder 模式该如何编写。

这次我们以构造一个 `Car` 为例，定义如下：

```go
// Car 代表一个汽车对象
type Car struct {
	// 品牌
	Brand string
	// 型号
	Model string
	// 颜色
	Color string
	// 发动机类型
	Engine string
}
```

定义 `CarBuilder` 结构体作为用来构建 `Car` 的 `Builder` 对象：

```go
// CarBuilder 用于构建 Car 对象的构建器
type CarBuilder struct {
	car Car
}

// NewCarBuilder 创建一个新的 CarBuilder 实例
func NewCarBuilder() *CarBuilder {
	return &CarBuilder{car: Car{}}
}
```

接下来我们就可以为其定义构建方法了：

```go
// SetBrand 设置汽车的品牌
func (b *CarBuilder) SetBrand(brand string) *CarBuilder {
	b.car.Brand = brand
	return b
}

// SetModel 设置汽车的型号
func (b *CarBuilder) SetModel(model string) *CarBuilder {
	b.car.Model = model
	return b
}

// SetColor 设置汽车的颜色
func (b *CarBuilder) SetColor(color string) *CarBuilder {
	b.car.Color = color
	return b
}

// SetEngine 设置汽车的发动机类型
func (b *CarBuilder) SetEngine(engine string) *CarBuilder {
	b.car.Engine = engine
	return b
}

// Build 构建并返回最终的 Car 对象
func (b *CarBuilder) Build() Car {
	return b.car
}
```

这里的 `SetXxx` 方法用于设置属性，对标的是前文构建 `House` 示例中的 `BuildXxx` 方法。并且 `SetXxx` 系列方法都支持传入参数，在构建过程中可以更加灵活的设置不同的属性值。这样仅需要定义一个 `CarBuilder` 对象，不必再定义 `CarBuilder2`、`CarBuilder3` ... 了。

> NOTE:
> 这里之所以换了一种命名方式，采用 `SetXxx`，是为了给你演示两种不同的命名风格，这两种写法都比较常见，看到其他人写的代码时你不要疑惑。

`Build` 方法则对标前文示例中的 `GetResult` 方法，作用相同。

并且，这个版本的 Builder 模式省略了 `Builder` 接口的定义。

> NOTE:
> 当然有时候为了方便编写测试代码，我们还是需要定义接口的。这里只是演示 Builder 模式在 Go 语言中的最小化实现。

最后，我们还去掉了 `Director` 这个角色。这可以让代码更加简洁，使用起来也更加灵活。

至此，Go 语言简化版的 Builder 模式就定义完成了。

用法如下：

```go
// 使用 CarBuilder 构建一个 Car 对象
car := NewCarBuilder().
	SetBrand("Tesla").
	SetModel("Model S").
	SetColor("Red").
	SetEngine("Electric").
	Build()

fmt.Printf("Car: %+v\n", car)
```

需要提醒的是，**我们不应该对构建方法 `SetXxx` 的调用顺序做任何假设**，所以这样使用也是可以的：

```go
car := NewCarBuilder().
	SetEngine("Electric").
	SetModel("Model S").
	SetBrand("Tesla").
	SetColor("Red").
	Build()

fmt.Printf("Car: %+v\n", car)
```

我们只需要保证 `Build` 方法在最后调用即可。

我们还可以在 `Build` 方法内部做统一的属性值校验。所有通过 `SetXxx` 设置的属性，都可以在 `Build` 方法中进行校验。

因为我们对用户调用了几个 `SetXxx` 方法是无法预知的，所以也就不建议在 `SetXxx` 方法中参数校验。假如用户少调用了一个方法，那么定义在 `SetXxx` 方法中的校验就不会生效。所以我们应该在 `Build` 方法内部做统一的属性值校验。

其实这也能暴露 Builder 模式的一个问题，就是将构建过程（属性赋值操作）分为多个步骤以后，我们构建的对象可能不完整（用户可能少调用了哪个 `SetXxx`），所以一定要定义一个 `Build` 方法来获取最终的构建产物（可以校验属性值），这个方法是不可省略的。

讲解完了 Go 语言中我们如何定义和使用 Builder 模式，接下来我再介绍一个实用场景，来加深你对 Builder 模式的理解。

### 实用场景

我们就以一个 Web Server 为例，演示下在中间件场景中如何使用 Builder 模式。

假设我们有这样一个使用 Gin 框架编写的 HTTP Server 程序：

```go
package main

import (
	"net/http"

	"github.com/gin-gonic/gin"
)

func main() {
	r := gin.Default()

	r.GET("/admin", func(c *gin.Context) {
		c.JSON(http.StatusOK, gin.H{
			"message": "Welcome Admin!",
		})
	})

	r.GET("/user", func(c *gin.Context) {
		c.JSON(http.StatusOK, gin.H{
			"message": "Welcome User!",
		})
	})

	r.Run(":8000")
}
```

示例程序有两个接口，分别是 `/admin` 和 `/user`，它们都接收 GET 请求并返回一段字符串内容。

现在我们想为这两个接口增加 RBAC 权限控制，`/admin` 接口只有角色为 `admin` 的用户才可以访问，`/user` 接口则角色为 `admin` 和 `user` 的用户都可以访问。

这个需求显然可以使用 Gin 的中间件来实现。

我们最容易想到的方式，就是定义一个用于控制 RBAC 权限的中间件函数：

```go
func RBACMiddleware(allowedRoles []string) gin.HandlerFunc {
	return func(c *gin.Context) {
		userRole := c.GetHeader("Role") // 从请求头中获取用户角色
		for _, role := range allowedRoles {
			if role == userRole {
				c.Next()
				return
			}
		}
		c.AbortWithStatus(http.StatusForbidden)
	}
}
```

代码非常简单，我就不过多解释了，`RBACMiddleware` 中间件可以这样使用：

```go
// 为 /admin 路由设置只有管理员角色才能访问
adminMiddleware := RBACMiddleware([]string{"admin"})

r.GET("/admin", adminMiddleware, func(c *gin.Context) {
	c.JSON(http.StatusOK, gin.H{
		"message": "Welcome Admin!",
	})
})

// 为 /user 路由设置普通用户和管理员角色都能访问
userMiddleware := RBACMiddleware([]string{"admin", "user"})

r.GET("/user", userMiddleware, func(c *gin.Context) {
	c.JSON(http.StatusOK, gin.H{
		"message": "Welcome User!",
	})
})
```

现在启动这个 HTTP Server，来测试一下我们的中间件效果：

```bash
# 使用 admin 角色访问 /admin 接口
$ curl -i -H "Role: admin" "http://localhost:8000/admin"
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
Date: Mon, 26 Aug 2024 06:33:07 GMT
Content-Length: 28

{"message":"Welcome Admin!"}

# 使用 user 角色访问 /admin 接口
$ curl -i -H "Role: user" "http://localhost:8000/admin"
HTTP/1.1 403 Forbidden
Date: Mon, 26 Aug 2024 06:33:03 GMT
Content-Length: 0

# 使用 admin 角色访问 /user 接口
$ curl -i -H "Role: admin" "http://localhost:8000/user"
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
Date: Mon, 26 Aug 2024 06:32:53 GMT
Content-Length: 27

{"message":"Welcome User!"}

# 使用 user 角色访问 /user 接口
$ curl -i -H "Role: user" "http://localhost:8000/user"
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
Date: Mon, 26 Aug 2024 06:32:59 GMT
Content-Length: 27

{"message":"Welcome User!"}
```

根据客户端的请求日志来看，我们的目的实现了。除了使用 `user` 角色访问 `/admin` 接口时，接口返回 `403` 状态码，没有返回响应体，其他请求都能得到正常响应。

我们再来看下如何使用 Builder 模式来实现 `RBACMiddleware` 中间件：

```go
// RBACMiddleware RBAC 中间件结构体
type RBACMiddleware struct {
	allowedRoles []string
}

// Middleware 返回一个 Gin 中间件函数
func (r *RBACMiddleware) Middleware() gin.HandlerFunc {
	return func(c *gin.Context) {
		userRole := c.GetHeader("Role") // 从请求头中获取用户角色
		for _, role := range r.allowedRoles {
			if role == userRole {
				c.Next()
				return
			}
		}
		c.AbortWithStatus(http.StatusForbidden)
	}
}

// RBACMiddlewareBuilder 用于构建 RBACMiddleware 的构建器
type RBACMiddlewareBuilder struct {
	rbacMiddleware *RBACMiddleware
}

// NewRBACMiddlewareBuilder 创建一个新的 RBACMiddlewareBuilder 实例
func NewRBACMiddlewareBuilder() *RBACMiddlewareBuilder {
	return &RBACMiddlewareBuilder{
		rbacMiddleware: &RBACMiddleware{},
	}
}

// AllowRole 添加允许访问的角色
func (b *RBACMiddlewareBuilder) AllowRole(role string) *RBACMiddlewareBuilder {
	b.rbacMiddleware.allowedRoles = append(b.rbacMiddleware.allowedRoles, role)
	return b
}

// Build 返回构建完成的 RBACMiddleware
func (b *RBACMiddlewareBuilder) Build() *RBACMiddleware {
	return b.rbacMiddleware
}
```

有了前文对 Builder 模式的讲解，这段代码就很好理解了。

> NOTE:
> 为了增强代码的语义，这里的 `AllowRole` 方法命名并没有定义为 `SetXxx` 或 `BuildXxx`，但作用相同。

使用 Builder 模式实现的 RBAC 中间件用法如下：

```go
// 为 /admin 路由设置只有管理员角色才能访问
adminMiddleware := NewRBACMiddlewareBuilder().
	AllowRole("admin").
	Build().
	Middleware()

r.GET("/admin", adminMiddleware, func(c *gin.Context) {
	c.JSON(http.StatusOK, gin.H{
		"message": "Welcome Admin!",
	})
})

// 为 /user 路由设置普通用户和管理员角色都能访问
userMiddleware := NewRBACMiddlewareBuilder().
	AllowRole("admin").
	AllowRole("user").
	Build().
	Middleware()

r.GET("/user", userMiddleware, func(c *gin.Context) {
	c.JSON(http.StatusOK, gin.H{
		"message": "Welcome User!",
	})
})
```

你可以再次尝试使用 `curl` 命令来测试我们的中间件效果，我就不再进行演示了。

这就是一个典型的 Builder 模式在 Go 语言中的使用场景。

也许你会觉得这个示例程序中第一种中间件实现 `func RBACMiddleware(allowedRoles []string) gin.HandlerFunc` 更加简单易用。

没错，这不是错觉。但是，我想要说的是，如果在版本迭代的过程中，`RBACMiddleware` 函数需要增加参数。那么 `RBACMiddleware` 的所有调用方，就都需要修改。因为函数签名改变了，即使新增的参数可能并不是一个必须参数。

这就增加了代码的维护成本。解决方案就是 Builder 模式。

使用 Builder 模式来实现中间件，如果需要新增参数，我们只需要为 `RBACMiddlewareBuilder` 增加一个 `SetXxx` 方法即可。调用方可以根据需要决定是否调用 `SetXxx` 方法，完全不影响现有调用方的代码。

所以，其实这个场景下 Builder 模式并不是一种不得不用的方式，而是一种可以更为优雅的解决未来需求变更的方式。这就为我们留足了“后门”，即使刚开始代码设计的不够合理，未来我们也可以使用对现有调用方无感知的方式更新我们的代码。

这其实也可以作为套路代码，以后 Gin 框架的中间件代码都可以参考这样设计。

不过，且慢！其实我平常还会使用一种比这个更加“简陋”的写法来实现中间件：

```go
// RBACMiddlewareBuilder RBAC 中间件结构体
type RBACMiddlewareBuilder struct {
	allowedRoles []string
}

// NewRBACMiddlewareBuilder 创建一个新的 RBACMiddlewareBuilder 实例
func NewRBACMiddlewareBuilder() *RBACMiddlewareBuilder {
	return &RBACMiddlewareBuilder{}
}

// Build 返回一个 Gin 中间件函数
func (b *RBACMiddlewareBuilder) Build() gin.HandlerFunc {
	return func(c *gin.Context) {
		userRole := c.GetHeader("Role") // 从请求头中获取用户角色
		for _, role := range b.allowedRoles {
			if role == userRole {
				c.Next()
				return
			}
		}
		c.AbortWithStatus(http.StatusForbidden)
	}
}

// AllowRole 添加允许访问的角色
func (b *RBACMiddlewareBuilder) AllowRole(role string) *RBACMiddlewareBuilder {
	b.allowedRoles = append(b.allowedRoles, role)
	return b
}
```

用法如下：

```go
adminMiddleware := NewRBACMiddlewareBuilder().
	AllowRole("admin").
	Build()
```

所以，其实我们不必纠结于设计模式的具体定义。对于 Builder 模式，在 Go 语言的实践中，我们完全可以视情况而定舍弃如 `Builder` 接口、`Director` 指挥者这样的角色，以最小化的代码，来实现 Builder 模式。

此外，其实使用 Options 模式也可以实现这个 `RBACMiddleware`。Builder 模式与 Options 模式最大的区别就是 Options 模式不能链式调用，不过却可以支持任意数量的参数。你可以自行尝试实现一下，对比下与 Builder 模式的区别。不过，如果你懒得实现，也可以参考 [我实现的版本](https://github.com/jianghushinian/blog-go-example/tree/main/design-patterns/builder/middleware/middleware/options)。



### 总结

Builder 模式是一种创建型模式，可以用来创建对象。

Builder 模式中有几个角色，分别是 `Builder` 接口，`ConcreteBuilder` 具体实现，以及 `Director` 指挥者。对这几个角色了解清楚，你就能理解什么是 Builder 模式。

不过，在生产实践中，我们也不要过于“学院派”，可以按照自己的理解来实现 Builder 模式。毕竟设计模式是拿来用的，而不是让我们死记硬背应付考试的。

在实用场景介绍中，我讲解了如何使用 Builder 模式来实现一个 RBAC 权限控制中间件。

实现 Builder 模式时，需要注意的一点是，不能假设用户对 `SetXxx` 方法的调用顺序，所以代码实现上，一定不能依赖这些 `SetXxx` 方法的调用顺序。

你认为适配器模式还有哪些应用场景，可以一起交流学习。

本文示例源码我都放在了 [GitHub](https://github.com/jianghushinian/blog-go-example/tree/main/design-patterns/builder) 中，欢迎点击查看。

希望此文能对你有所启发。

**延伸阅读**

- 《设计模式：可复用面向对象软件的基础》：https://book.douban.com/subject/34262305/
- Go 常见设计模式之选项模式：https://jianghushinian.cn/2021/12/12/Golang-常见设计模式之选项模式/
- Go 常见设计模式之装饰模式：https://jianghushinian.cn/2022/02/28/Golang-常见设计模式之装饰模式/
- Go 常见设计模式之单例模式：https://jianghushinian.cn/2022/03/04/Golang-常见设计模式之单例模式/
- 适配器模式在 Go 语言中的应用：https://jianghushinian.cn/2024/08/04/go-design-patterns-adapter/
- 本文 GitHub 示例代码：https://github.com/jianghushinian/blog-go-example/tree/main/design-patterns/builder

**联系我**

- 公众号：[Go编程世界](https://jianghushinian.cn/about)
- 微信：jianghushinian
- 邮箱：[jianghushinian007@outlook.com](mailto:jianghushinian007@outlook.com)
- 博客：https://jianghushinian.cn
