---
title: 'Go 官方推荐的 Go 项目文件组织方式'
date: 2025-06-21 11:36:40
tags:
- Go
categories:
- Go
---

刚开始接触 Go 的开发者大概都会遇到一个问题：我该如何组织我的 Go 项目？这种问题当然没有标准答案，不过 Go 官方下场，给了广大 Gopher 一个推荐模板。本文就来带大家一起来学习一下 Go 官方对于 Go 项目布局的指导原则。

本文以 Go 官方博客「[Organizing a Go module](https://go.dev/doc/modules/layout)」为基石进行讲解。

<!-- more -->

### 基础 Go 包

对于最基础的 Go 包，我们可以按照如下方式组织：

```bash
project-root-directory/
  go.mod
  modname.go
  modname_test.go
```

所有代码都位于项目的根目录下，这个项目由一个模块（module）组成，而这个模块又由一个包（package）组成。

假设这个项目会上传到 GitHub 仓库 `github.com/someuser/modname` 中，那么 `go.mod` 文件中的第一行（module line）应写为：

```go
module github.com/someuser/modname
```

`modname.go` 文件中的代码使用如下命令声明一个 `modname` 包：

```go
package modname

// ... package code here
```

在其他 Go 项目中，可以通过如下方式导入这个 Go 包：

```go
import "github.com/someuser/modname"
```

此外，一个 Go 包其实可以被拆分成多个文件：

```bash
project-root-directory/
  go.mod
  modname.go
  modname_test.go
  auth.go
  auth_test.go
  hash.go
  hash_test.go
```

这里所有 Go 文件都属于同一个包，即都会声明为 `package modname`。

> NOTE:
>
> 不知道你有没有注意，示例中的项目名称 `project-root-directory` 使用短横线来连接多个单词，而 Go 文件名则使用单词直接连写的形式 `modname.go`（而非 `mod_name.go`），这也是 Go 的一贯风格，虽然没有强制约定，但看多了社区中优秀的 Go 代码，你就会有所体会。
>
> 关于这点更加深入的探讨，你可以阅读我曾写过文章《[Go 项目文件命名规范是什么？](https://mp.weixin.qq.com/s/reVRrXYexMB6c8RChFW2Lg)》来进行学习。
>

### 可执行的 Go 程序

Go 可执行程序（或命令行程序）一般根据复杂程度来构建目录，最小的 Go 程序可以仅定义一个包含 `main` 函数的 Go 文件，如果功能较多，可以将其拆分为多个文件：

```bash
project-root-directory/
  go.mod
  auth.go
  auth_test.go
  client.go
  main.go
```

`main.go` 中存放了 `main` 函数，不过这只是一个约定俗成的做法，你可以将其放到任何合法的 `xxx.go` 中。

当拆分为多个文件，如果 `main.go` 引用了其他文件中的代码，这个时候就不能简单的使用 `go run main.go` 来执行 Go 程序了，可以按照如下方式执行：

```bash
$ go run .
```

并且，用户可以使用以下命令将 Go 程序安装到自己的主机上：

```bash
$ go install github.com/someuser/modname@latest
```

### 更复杂的 Go 程序

如果一个 Go 包或者 Go 可执行程序更加复杂，则可以按照如下方式组织目录结构：

```bash
project-root-directory/
  internal/
    auth/
      auth.go
      auth_test.go
    hash/
      hash.go
      hash_test.go
  go.mod
  modname.go
  modname_test.go
```

这里还演示了使用 `modname.go` 作为程序的入口，而非命名为 `main.go`。

`internal/` 目录下的所有包都不会被公开，其他程序无法引用。这是 Go 语言层面限制，强制的，不仅仅是约定俗成。这样，我们就可以随时随地的重构 `internal/` 目录中的代码，而不必担心影响其他程序。

当然，`modname.go` 中可以导入 `internal/` 目录下的包，因为它们本来就是一个程序。在 `modname.go` 中导入 `auth` 包：

```go
import "github.com/someuser/modname/internal/auth"
```

### 包含多个包的 Go 程序

我们知道，一个 Go 模块（module）可以由多个包（package）组成，而每个包都可以有自己的子目录。

我们可以组织成一个层次结构更深的 Go 程序：

```bash
project-root-directory/
  go.mod
  modname.go
  modname_test.go
  auth/
    auth.go
    auth_test.go
    token/
      token.go
      token_test.go
  hash/
    hash.go
  internal/
    trace/
      trace.go
```

这个目录看起来更符合一个真实的 Go 包。

假设 `go.mod` 中声明 `module` 如下：

```go
module github.com/someuser/modname
```

而 `modname.go` 位于项目根目录，声明包为 `modname`，用户在使用时则可以按照如下方式导入此包：

```go
import "github.com/someuser/modname"
```

也可以按照如下方式导入子包：

```go
import "github.com/someuser/modname/auth"
import "github.com/someuser/modname/auth/token"
import "github.com/someuser/modname/hash"
```

注意：`internal/trace` 无法被外部程序导入。

### 包含多个可执行的 Go 程序

有时候，我们可能会在一个 Git 仓库中包含多个可执行的 Go 程序，可以按照如下方式组织目录：

```bash
project-root-directory/
  go.mod
  internal/
    ... shared internal packages
  prog1/
    main.go
  prog2/
    main.go
```

`prog1/` 和 `prog2/` 中分别有自己的 `main.go` 文件，即两个独立的可执行 Go 程序。

`prog1/` 和 `prog2/` 可以共享根目录中的其他包或子包。

我们可以按照如下方式分别安装二者到主机上：

```bash
$ go install github.com/someuser/modname/prog1@latest
$ go install github.com/someuser/modname/prog2@latest
```

其实，针对一个 Git 仓库中包含多个可执行的 Go 程序的情况，我们还有更好的做法来组织目录：

```bash
project-root-directory/
  go.mod
  modname.go
  modname_test.go
  auth/
    auth.go
    auth_test.go
  internal/
    ... internal packages
  cmd/
    prog1/
      main.go
    prog2/
      main.go
```

将可执行文件都放到 `cmd/` 目录下也是约定俗成的做法。下次，当你见到 `cmd/` 目录，就应该想到这个目录是干什么的。

现在可执行程序的安装命令如下：

```bash
$ go install github.com/someuser/modname/cmd/prog1@latest
$ go install github.com/someuser/modname/cmd/prog2@latest
```

### 服务器项目

其实，我们在用 Go 编写程序时，大多数是用来实现 Go Web Server 程序，这也是 Go 最常见的使用场景，Go 是实现服务器的通用语言选择。

比如 Kubernetes 项目，其最核心的组件也是 Server 程序。

通常，我们可以按照如下方式来组织 Go 服务器项目：

```bash
project-root-directory/
  go.mod
  internal/
    auth/
      ...
    metrics/
      ...
    model/
      ...
  cmd/
    api-server/
      main.go
    metrics-analyzer/
      main.go
    ...
  ... the project's other directories with non-Go code
```

如果你有过 Go Web 开发经验，是不是很熟悉这个目录结构？

这已经到了广大 Gopher 最熟悉的领域，接下来，我就不过多讲解了，你可以继续阅读我写的另一篇文章《[如何设计一个优秀的 Go Web 项目目录结构](https://mp.weixin.qq.com/s/ewAu_JqK0rZPcNMQq0JuIg)》，里面介绍了开源项目 [https://github.com/golang-standards/project-layout](https://github.com/golang-standards/project-layout) 对 Go Web 项目组织的最佳实践。

### 总结

我们学习了 Go 官方推荐的项目布局的指导原则，现在我们对于 Go 包和 Go 可执行程序的目录组织方式，都有了较为清晰的认知。

此外，我们还顺便学习了如何安装 Go 可以执行程序，可以通过 `go install xxx` 来安装。

对于广大 Gopher，接触最多的应该是 Go Web 服务器的开发。对于 Go Web 项目，不仅 Go 官方给出了指导原则，我们还可以参考 [project-layout](https://github.com/golang-standards/project-layout) 项目学习一个大型的 Go 项目是如何布局的。

希望此文能对你有所启发。

**延伸阅读**

+ Organizing a Go module：[https://go.dev/doc/modules/layout](https://go.dev/doc/modules/layout)
+ Standard Go Project Layout：[https://github.com/golang-standards/project-layout](https://github.com/golang-standards/project-layout)
+ Tutorial: Create a Go module：[https://go.dev/doc/tutorial/create-module](https://go.dev/doc/tutorial/create-module)
+ Managing module source：[https://go.dev/doc/modules/managing-source](https://go.dev/doc/modules/managing-source)
+ Internal Directories：[https://pkg.go.dev/cmd/go#hdr-Internal_Directories](https://pkg.go.dev/cmd/go#hdr-Internal_Directories)
+ 如何设计一个优秀的 Go Web 项目目录结构：[https://mp.weixin.qq.com/s/ewAu_JqK0rZPcNMQq0JuIg](https://mp.weixin.qq.com/s/ewAu_JqK0rZPcNMQq0JuIg)
+ Go 项目文件命名规范是什么？：[https://mp.weixin.qq.com/s/reVRrXYexMB6c8RChFW2Lg](https://mp.weixin.qq.com/s/reVRrXYexMB6c8RChFW2Lg)
+ 本文永久地址：[https://jianghushinian.cn/2025/06/21/go-modules-layout](https://jianghushinian.cn/2025/06/21/go-modules-layout)

**联系我**

+ 公众号：[Go编程世界](https://jianghushinian.cn/about)
+ 微信：jianghushinian
+ 邮箱：[jianghushinian007@outlook.com](mailto:jianghushinian007@outlook.com)
+ 博客：[https://jianghushinian.cn](https://jianghushinian.cn)
+ GitHub：[https://github.com/jianghushinian](https://github.com/jianghushinian)
