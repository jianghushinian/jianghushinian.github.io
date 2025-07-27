---
title: 'Go 项目中的 doc.go 文件是干嘛的？'
date: 2025-07-27 09:41:41
tags:
- Go
categories:
- Go
---

作为广大 Gopher 中的一员，你一定在 Go 项目中写过或者见过一个叫 `doc.go` 的文件，不知道你是否好奇这个文件到底是干嘛的，它有哪些作用？本文就来介绍一下 `doc.go` 这个文件。

<!-- more -->

### doc.go 文件作用

简单来讲，`doc.go` 文件在 Go 语言项目中主要用于**集中管理包的文档**，是提升代码可读性和自动化生成文档的关键工具，其设计逻辑与 Go 的文档工具链（如 `godoc`/`godoc`）深度契合。

Go 在源码中就大量应用了 `doc.go` 文件，比如 Go `fmt` 源码中就有一个 `doc.go` 文件：

> [https://github.com/golang/go/blob/master/src/fmt/doc.go](https://github.com/golang/go/blob/master/src/fmt/doc.go)
>

```go
// Copyright 2009 The Go Authors. All rights reserved.
// Use of this source code is governed by a BSD-style
// license that can be found in the LICENSE file.

/*
Package fmt implements formatted I/O with functions analogous
to C's printf and scanf.  The format 'verbs' are derived from C's but
are simpler.

# Printing

The verbs:

General:

	%v	the value in a default format
		when printing structs, the plus flag (%+v) adds field names
	%#v	a Go-syntax representation of the value
		(floating-point infinities and NaNs print as ±Inf and NaN)
	%T	a Go-syntax representation of the type of the value
	%%	a literal percent sign; consumes no value

...
*/
package fmt
```

可以看到，这个文件内容主要包含三个部分：

+ 首先是在 `//` 注释中声明了版权信息（Copyright）。
+ 接着在 `/**/` 注释中写明了 `fmt` 包的使用文档，由于内容过多，这里进行了省略。
+ 最后是 `fmt` 包的声明。

其他再无内容。

所以根据文件内容，也印证了 `doc.go` 文件的作用就是用来编写包文档的。

不过，`doc.go` 文件的作用不止如此。

现在，我们可以打开 `fmt` 包的官方文档网址 [https://pkg.go.dev/fmt](https://pkg.go.dev/fmt) 来看一下。

截图如下：

![fmt](go-dev-fmt.png)

也可以打开镜像站网址 [https://golang.google.cn/pkg/fmt/](https://golang.google.cn/pkg/fmt/) 。

文档截图如下：

![fmt](golang-google-fmt.png)

> NOTE:
>
>  [https://golang.google.cn](https://golang.google.cn) 是 Go 在 2018 年专门为中国大陆提供访问的 Go 官方文档网站，你可以在官方博客「[Hello, 中国!](https://go.dev/blog/hello-china)」中看到说明。
>

可以发现，无论是 [https://go.dev/](https://go.dev/) ，还是 [https://golang.google.cn](https://golang.google.cn) 网站，`doc.go` 文件注释内容都会完整的展示在这里。即我们在 `doc.go` 文件中编写的文档内容，在使用 Go 工具链生成网页版文档时，就会展示在网页中。

### doc.go 实践

我们可以编写一个小的 demo 实例程序，来演示一下 `doc.go` 文件如何使用。

首先，因为 `doc.go` 用于统一生成包级文档，所以通常会被放置于包根目录下，其包声明前的注释会被 `godoc` 或 `go doc` 提取为包的概要说明。

> NOTE:
>
> `godoc` 和 `go doc` 都是用来查看 Go 项目文档的，我之后会专门写一篇文章进行讲解。
>

例如：

```go
// Package calculator provides math operations for financial calculations.
package calculator
```

我们要编写的 demo 程序是一个能够实现四则运算的包，项目目录结构如下：

```bash
$ tree calculator                           
calculator
├── calculator.go
└── doc.go

1 directory, 2 files
```

`calculator/doc.go` 文件完整内容如下：

```go
// Package calculator provides math operations for financial calculations.
//
// 本包实现安全的四则运算，包含加减乘除功能。
// 所有函数均针对整型设计，适用于基础计算场景。
//
// 使用示例：
//
//	sum := calculator.Add(5, 3)          // 8
//	diff := calculator.Subtract(5, 3)    // 2
//	product := calculator.Multiply(5, 3) // 15
//	quotient := calculator.Divide(6, 3)  // 2
//
// 注意事项：
//  1. 除法函数 Divide() 在除数为 0 时返回 0
//  2. 整数除法会截断小数部分（如 5/2=2）
package calculator // import "govanityurls.jianghushinian.cn/blog-go-example/doc/docgo/calculator"

```

`calculator/calculator.go` 用于实现包的具体代码逻辑，文件内容如下：

```go
package calculator

// Add 返回两个整数的和
func Add(a, b int) int {
	return a + b
}

// Subtract 返回两个整数的差（a - b）
func Subtract(a, b int) int {
	return a - b
}

// Multiply 返回两个整数的乘积
func Multiply(a, b int) int {
	return a * b
}

// Divide 返回两个整数的商（a / b）
func Divide(a, b int) int {
	if b == 0 {
		return 0 // 简单处理除零错误
	}
	return a / b
}

```

现在，我们在终端中通过 `go doc` 命令可以直接查看此包的文档：

```bash
$  go doc calculator
package calculator // import "govanityurls.jianghushinian.cn/blog-go-example/doc/docgo/calculator"

Package calculator provides math operations for financial calculations.

本包实现安全的四则运算，包含加减乘除功能。 所有函数均针对整型设计，适用于基础计算场景。

使用示例：

    sum := calculator.Add(5, 3)          // 8
    diff := calculator.Subtract(5, 3)    // 2
    product := calculator.Multiply(5, 3) // 15
    quotient := calculator.Divide(6, 3)  // 2

注意事项：
 1. 除法函数 Divide() 在除数为 0 时返回 0
 2. 整数除法会截断小数部分（如 5/2=2）

func Add(a, b int) int
func Divide(a, b int) int
func Multiply(a, b int) int
func Subtract(a, b int) int

```

输出的文档正是我们在 `calculator/doc.go` 文件中编写的内容。

你还可以通过如下方式查看某个具体函数的文档：

```bash
$ go doc calculator.Add
package calculator // import "govanityurls.jianghushinian.cn/blog-go-example/doc/docgo/calculator"

func Add(a, b int) int
    Add 返回两个整数的和（a + b）

```

此外，我们可以通过 `godoc` 命令启动一个本地 Web 服务，用来在网页中查看 `calculator` 包文档。  
示例如下：

```bash
$ godoc -http=:8080
using module mode; GOMOD=/Users/jianghushinian/go/blog-go-example/doc/docgo/go.mod

```

执行 `godoc` 命令后，程序会阻塞在这里。浏览器打开 [http://localhost:8080/pkg/govanityurls.jianghushinian.cn/blog-go-example/doc/docgo/calculator/](http://localhost:8080/pkg/govanityurls.jianghushinian.cn/blog-go-example/doc/docgo/calculator/) 即可查看文档：

> NOTE:
>
> 可以使用 `go install golang.org/x/tools/cmd/godoc@latest` 命令安装 `godoc`。
>

![calculator](calculator.png)

### 社区实践

如果你比较细心，应该已经发现了我在示例程序的 `doc.go` 文件中 `calculator` 包声明后面加了如下注释：

```go
package calculator // import "govanityurls.jianghushinian.cn/blog-go-example/doc/docgo/calculator"
```

这个注释并不是随便加的，这其实是一种被叫做 Canonical import paths（权威导入路径）的技术，来实现定制包的导入路径，你可以在这 [https://go.dev/doc/go1.4#canonicalimports](https://go.dev/doc/go1.4#canonicalimports) 查看其详细说明。

这个特性在 Go 1.4 版本被加入，简单来解释，Go 包的导入路径通常是其在代码托管平台上的完整路径，比如我将 `calculator` 包托管在 GitHub 中（[https://github.com/jianghushinian/blog-go-example/tree/main/doc/docgo/calculator](https://github.com/jianghushinian/blog-go-example/tree/main/doc/docgo/calculator)），那么它的导入路径就是 `github.com/jianghushinian/blog-go-example/doc/docgo/calculator`。但你应该也见过很多特例，比如 [GORM](https://github.com/go-gorm/gorm) 托管在 GitHub 中，但其导入路径是 `gorm.io/gorm`；Uber 的 [zap](https://github.com/uber-go/zap) 也托管在 GitHub 中，但其导入路径是 `go.uber.org/zap`。当你要实现这种功能时，就可以使用权威导入路径注释了。

所以，你能在 Kubernetes 项目中也能看到类似的实践：

> [https://github.com/kubernetes/kubernetes/blob/v1.32.0/staging/src/k8s.io/api/apps/v1/doc.go](https://github.com/kubernetes/kubernetes/blob/v1.32.0/staging/src/k8s.io/api/apps/v1/doc.go)
>

```go
/*
Copyright 2017 The Kubernetes Authors.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
*/

// +k8s:deepcopy-gen=package
// +k8s:protobuf-gen=package
// +k8s:openapi-gen=true
// +k8s:prerelease-lifecycle-gen=true

package v1 // import "k8s.io/api/apps/v1"
```

这是一个非常具有代表性的 `doc.go` 文件。

首先依然是版权声明 (Copyright)，接着是一段代码生成指令（`// +k8s:` 注释），最后在 `package v1` 后面还有一个权威导入路径注释 `import "k8s.io/api/apps/v1"`。

这里，我们又学习到一种 `doc.go` 的使用场景，就是可以在这个文件中编写一些代码生成指令，用于控制整个包的代码生成。

`// +k8s:` 是 Kubernetes 进行代码生成的 Annotation 风格注释，通过此类注释，Kubernetes 工具可以直接生成对应代码，并且，将其写在 `doc.go` 文件中，则表示生成的是这个包全局的代码。如果想要控制更细粒度的代码生成，则可以写在包具体的文件中，比如我们的 `calculator.go` 中。

不过，实际上在 Kubernetes 1.33 版本代码中，`doc.go` 文件已经移除了权威导入路径这个注释。

如下：

> [https://github.com/kubernetes/kubernetes/blob/v1.33.0/staging/src/k8s.io/api/apps/v1/doc.go](https://github.com/kubernetes/kubernetes/blob/v1.33.0/staging/src/k8s.io/api/apps/v1/doc.go)
>

```go
/*
Copyright 2017 The Kubernetes Authors.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
*/

// +k8s:deepcopy-gen=package
// +k8s:protobuf-gen=package
// +k8s:openapi-gen=true
// +k8s:prerelease-lifecycle-gen=true

package v1
```

其实，自从 Go 在 1.11 版本开始支持了 Go Modules，实际上权威导入路径这个特性已经没什么用了，我们可以直接在 `go.mod` 文件中定义包名称，而无需考虑其代码托管平台，例如：

```go
module govanityurls.jianghushinian.cn/blog-go-example/doc/docgo

go 1.24.1
```

这样，你就直接可以通过将其作为包的导入路径了。

> NOTE:
>
> 当然，你可能需要为这个包的域名（govanityurls.jianghushinian.cn）搭建一个 Go 包的导航服务器，可以参考 Go Vanity URLs 项目：[https://github.com/GoogleCloudPlatform/govanityurls](https://github.com/GoogleCloudPlatform/govanityurls) 。
>

### 总结

本文对 `doc.go` 这个广大 Gopher 习以为常的文件作用进行了讲解说明，并演示了它是如何使用的，希望能解决一部分人的好奇或者困惑。

你还知道 `doc.go` 有哪些用途，欢迎留言区讨论交流。

本文示例源码我都放在了 [GitHub](https://github.com/jianghushinian/blog-go-example/tree/main/doc/docgo) 中，欢迎点击查看。

希望此文能对你有所启发。

**延伸阅读**

+ Go fmt doc.go：[https://github.com/golang/go/blob/master/src/fmt/doc.go](https://github.com/golang/go/blob/master/src/fmt/doc.go)
+ Kubernetes 1.32.0 doc.go：[https://github.com/kubernetes/kubernetes/blob/v1.32.0/staging/src/k8s.io/api/apps/v1/doc.go](https://github.com/kubernetes/kubernetes/blob/v1.32.0/staging/src/k8s.io/api/apps/v1/doc.go)
+ Kubernetes 1.33.0 doc.go：[https://github.com/kubernetes/kubernetes/blob/v1.33.0/staging/src/k8s.io/api/apps/v1/doc.go](https://github.com/kubernetes/kubernetes/blob/v1.33.0/staging/src/k8s.io/api/apps/v1/doc.go)
+ Go 1.4 Canonical import paths：[https://go.dev/doc/go1.4#canonicalimports](https://go.dev/doc/go1.4#canonicalimports)
+ Go Vanity URLs：[https://github.com/GoogleCloudPlatform/govanityurls](https://github.com/GoogleCloudPlatform/govanityurls)
+ 本文 GitHub 示例代码：[https://github.com/jianghushinian/blog-go-example/tree/main/doc/docgo](https://github.com/jianghushinian/blog-go-example/tree/main/doc/docgo)
+ 本文永久地址：[https://jianghushinian.cn/2025/07/27/doc-go/](https://jianghushinian.cn/2025/07/27/doc-go/)

**联系我**

+ 公众号：[Go编程世界](https://jianghushinian.cn/about)
+ 微信：jianghushinian
+ 邮箱：[jianghushinian007@outlook.com](mailto:jianghushinian007@outlook.com)
+ 博客：[https://jianghushinian.cn](https://jianghushinian.cn)
+ GitHub：[https://github.com/jianghushinian](https://github.com/jianghushinian)
