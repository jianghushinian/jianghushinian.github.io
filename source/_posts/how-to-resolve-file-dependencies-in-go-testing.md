---
title: 在 Go 语言单元测试中如何解决文件依赖问题
date: 2023-07-19 20:07:57
tags:
- Go
- Test
categories:
- Go
---

现如今的 Web 应用程序往往采用 RESTful API 接口形式对外提供服务，后端接口直接向前端返回 HTML 文件的情况越来越少，所以在程序中操作文件的场景也变少了。不过有些时候还是需要对文件进行操作，比如某个 API 接口需要返回应用程序的 ChangeLog，那么这个接口就可以通过读取项目的 `CHANGELOG.md` 文件内容，将其发送给前端。

在编写单元测试时，文件就成了被测试代码的外部依赖，本文就来讲解下测试过程中如何解决文件外部依赖问题。

<!-- more -->

### 获取 ChangeLog 程序示例

假设我们有一个函数，可以读取项目的 ChangeLog 信息并返回。

程序代码实现如下：

```go
package main

import (
	"io"
	"os"
)

var (
	version        = "dev"
	commit         = "none"
	builtGoVersion = "unknown"
	changeLogPath  = "CHANGELOG.md"
)

type ChangeLogSpec struct {
	Version        string
	Commit         string
	BuiltGoVersion string
	ChangeLog      string
}

func GetChangeLog() (ChangeLogSpec, error) {
	data, err := os.ReadFile(changeLogPath)
	if err != nil {
		return ChangeLogSpec{}, err
	}

	return ChangeLogSpec{
		Version:        version,
		Commit:         commit,
		BuiltGoVersion: builtGoVersion,
		ChangeLog:      string(data),
	}, nil
}
```

`GetChangeLog` 函数实现比较简单，首先从 `changeLogPath` 文件路径中读取 ChangeLog 内容，然后结合程序版本号、COMMIT 信息、Go 版本号一起组装成 `ChangeLogSpec` 结构体，并返回。

### 使用临时文件测试

现在，我们要对 `GetChangeLog` 函数进行单元测试。

可以发现，`GetChangeLog` 函数内部依赖了 `changeLogPath` 文件路径，然后从中读取内容。所以，在编写测试时，我们要考虑 `changeLogPath` 文件如何指定。

我们最先想到的就是指定 `changeLogPath` 文件的真实路径。但是，这可能会存在问题，比如本地环境和 CI 环境下 `changeLogPath` 文件路径不同，那么在编写测试代码时，就要考虑根据不同的测试环境执行不同逻辑。所以，这种方式不应该成为首选方案。

不过，我们可以换种思路，Go 语言提供了 `os.CreateTemp` 方法，可以创建一个临时文件。那么，我们就可以考虑在测试函数开始时创建一个临时文件来保存 ChangeLog，然后为 `changeLogPath` 变量赋值为临时文件路径，测试代码执行完成后删除临时文件，这样就能够解决单元测试中依赖外部文件的问题。

按照这个思路，编写的单元测试代码如下：

```go
func TestGetChangeLog(t *testing.T) {
	// 创建临时文件
	// 第一个参数传 ""，表示在操作系统的临时目录下创建该文件
	// 文件文件名会以第二个参数作为前缀，剩余的部分会自动生成，以确保并发调用时生成的文件名不重复
	f, err := os.CreateTemp("", "TEST_CHANGELOG")
	assert.NoError(t, err)
	defer func() {
		_ = f.Close()
		// 尽管操作系统会在某个时间自动清理临时文件，但主动清理是创建者的责任
		_ = os.RemoveAll(f.Name())
	}()

	changeLogPath = f.Name()

	data := `
# Changelog
All notable changes to this project will be documented in this file.
`
	_, err = f.WriteString(data)
	assert.NoError(t, err)

	expected := ChangeLogSpec{
		Version:        "v0.1.1",
		Commit:         "1",
		BuiltGoVersion: "1.20.1",
		ChangeLog: `
# Changelog
All notable changes to this project will be documented in this file.
`,
	}

	actual, err := GetChangeLog()
	assert.NoError(t, err)
	assert.Equal(t, expected, actual)
}
```

我们首先通过 `os.CreateTemp("", "TEST_CHANGELOG")` 创建了一个临时文件，然后将 `data` 内容写入临时文件作为 ChangeLog，再然后将临时文件名称 `f.Name()` 赋值给 `changeLogPath`，之后就可以调用 `GetChangeLog` 函数进行测试了。

对于程序版本号、COMMIT 信息、Go 版本号这几个变量，因为都是全局变量，所以也属于外部依赖。

对于全局变量的依赖，我们可以在 `init` 函数中对其进行初始化，这样就相当于在测试环境中固定了这几个变量的值，便于测试。

```go
func init() {
	version = "v0.1.1"
	commit = "1"
	builtGoVersion = "1.20.1"
}
```

> 笔记：你也可以在 `TestMain` 函数中对其进行初始化。

使用 `go test` 来执行测试函数：

```bash
$ go test -v -run="TestGetChangeLog$"     
=== RUN   TestGetChangeLog
--- PASS: TestGetChangeLog (0.00s)
PASS
ok      github.com/jianghushinian/blog-go-example/test/file     0.562s
```

测试通过。

### 使用 Go embed 测试

以上我们介绍了使用临时文件的方式来解决被测试函数依赖外部文件的问题。

不过我们在测试中提供的 ChangeLog 内容不多：

```go
	data := `
# Changelog
All notable changes to this project will be documented in this file.
`
```

为了让单元测试更加可靠，你也许想测试 ChangeLog 内容比较多的情况下，`GetChangeLog` 函数能否正常工作。

我们可以编写一个真实的 `CHANGELOG.md` 文件，存放于 `testdata/CHANGELOG.md` 路径下：

```markdown
# Kubernetes v0.1.1

## 主要特性和改进

- 添加了一些新的主要特性和改进。

## 重要变更

- 这里列出了对现有功能的重要变更。

## API 变更

- 在 API 中进行的重要变更和更新。

## 已知问题

- 列出了已知的问题和限制。

## Bug 修复

- 修复了以下已知 Bug。

## 改进和优化

- 对现有功能进行了改进和优化。

## 安全性更新

- 列出了安全性方面的更新和修复。

## 已弃用功能

- 列出了已被弃用的功能。

## 警告和提醒

- 列出了需要注意的警告和提醒事项。

## 社区贡献者

- 致谢并列出了为此版本做出贡献的社区成员。

更详细的信息可以查阅 Kubernetes 官方文档和发布说明。
```

此时，我们可以使用 Go 提供的 embed 技术来将文件内容嵌入到 Go 变量中。

#### embed []byte

embed 可以实现在 Go 程序编译时，直接将文件内容嵌入到 Go 变量。embed 目前支持嵌入两种基础类型的变量，分别是 `[]byte` 和 `strings`。嵌入这两种类型变量方式相同，本小节就像大家演示下如何通过将文件嵌入 `[]byte` 变量的方式来编写 `GetChangeLog` 函数的单元测试。

为 `GetChangeLog` 函数编写的单元测试代码如下：

```go
package main

import (
	_ "embed"
	"os"
	"testing"

	"github.com/stretchr/testify/assert"
)

//go:embed testdata/CHANGELOG.md
var changelog []byte

func TestGetChangeLog_by_embed(t *testing.T) {
	f, err := os.CreateTemp("", "TEST_CHANGELOG")
	assert.NoError(t, err)
	defer func() {
		_ = f.Close()
		_ = os.RemoveAll(f.Name())
	}()

	changeLogPath = f.Name()

	_, err = f.Write(changelog)
	assert.NoError(t, err)

	expected := ChangeLogSpec{
		Version:        "v0.1.1",
		Commit:         "1",
		BuiltGoVersion: "1.20.1",
		ChangeLog:      string(changelog),
	}

	actual, err := GetChangeLog()
	assert.NoError(t, err)
	assert.Equal(t, expected, actual)
}
```

单元测试中，我们最需要关注的是这行代码：

```go
//go:embed testdata/CHANGELOG.md
var changelog []byte
```

`//go:embed` 是一个指令注释，用来标记嵌入指令，注意冒号 `:` 前后没有空格，`testdata/CHANGELOG.md` 指明要嵌入的文件。

在嵌入指令下方，紧挨着我们定义了变量 `var changelog []byte` 用来接收被嵌入文件的内容。

程序编译后，变量 `changelog` 的值就是 `testdata/CHANGELOG.md` 文件中的内容了。

注意，文件开头的 `import` 中要导入 `embed`，嵌入指令才可以使用。

之后的单元测试代码改动就比较小了，仅用 `changelog` 变量替换了原来代码中的 `data` 变量。

使用 `go test` 来执行测试函数：

```bash
$ go test -v -run="TestGetChangeLog_by_embed"
=== RUN   TestGetChangeLog_by_embed
--- PASS: TestGetChangeLog_by_embed (0.00s)
PASS
ok      github.com/jianghushinian/blog-go-example/test/file     0.365s
```

单元测试仍能通过。

#### embed fs.FS

Go embed 技术不仅能够嵌入文件到基础类型变量，还能直接将文件嵌入为一个文件系统。

为了演示这一强大的功能，我们修改下 `GetChangeLog` 函数代码，让其接收一个 `io.Reader` 类型的参数，然后从这个参数中读取 ChangeLog 内容，而不再是通过读取指定路径下的 ChangeLog 内容。

修改后程序代码如下：

```go
func GetChangeLogByIOReader(reader io.Reader) (ChangeLogSpec, error) {
	data, err := io.ReadAll(reader)
	if err != nil {
		return ChangeLogSpec{}, err
	}

	return ChangeLogSpec{
		Version:        version,
		Commit:         commit,
		BuiltGoVersion: builtGoVersion,
		ChangeLog:      string(data),
	}, nil
}
```

如下是为新的 `GetChangeLogByIOReader` 函数编写的单元测试代码：

```go
//go:embed testdata/CHANGELOG.md
var fs embed.FS

func TestGetChangeLogByIOReader(t *testing.T) {
	f, err := fs.Open("testdata/CHANGELOG.md")
	assert.NoError(t, err)

	data, err := io.ReadAll(f)
	assert.NoError(t, err)

	// 将数据的读取位置重置到开头
	_, err = f.(io.ReadSeeker).Seek(0, 0)
	assert.NoError(t, err)

	expected := ChangeLogSpec{
		Version:        "v0.1.1",
		Commit:         "1",
		BuiltGoVersion: "1.20.1",
		ChangeLog:      string(data),
	}

	actual, err := GetChangeLogByIOReader(f)
	assert.NoError(t, err)
	assert.Equal(t, expected, actual)
}
```

我们同样使用 `//go:embed testdata/CHANGELOG.md` 来指定嵌入的文件，不过，这次定义的变量 `var fs embed.FS` 是一个文件系统，里面包含了被嵌入的文件。

在测试代码中，使用 `fs.Open("testdata/CHANGELOG.md")` 打开文件内容，得到 `fs.File` 类型对象，之后就可以像其他 Go 文件对象一样操作它。

使用 `go test` 来执行测试函数：

```bash
$ go test -v -run="TestGetChangeLogByIOReader"                         
=== RUN   TestGetChangeLogByIOReader
--- PASS: TestGetChangeLogByIOReader (0.00s)
PASS
ok      github.com/jianghushinian/blog-go-example/test/file     0.135s
```

测试通过。

### 总结

本文向大家介绍了在 Go 中编写单元测试时，如何解决文件外部依赖的问题。

Go 语言提供了 `os.CreateTemp` 方法，可以创建一个临时文件，我们可以利用这个方法来解决文件外部依赖。

此外，Go 语言还提供了 embed 技术，能够在程序编译时直接将文件内容嵌入到 Go 变量中。这项技术虽然不是为单元测试而生的，但我们可以借此来解决文件外部依赖问题。本文为大家演示了如何将文件嵌入到 `[]byte` 和文件系统，两种方案用法差异不大，可以根据需求和喜好进行选择。

本文完整代码示例我放在了 [GitHub](https://github.com/jianghushinian/blog-go-example/tree/main/test/file) 上，欢迎点击查看。

希望此文能对你有所帮助。

**联系我**

- 微信：jianghushinian
- 邮箱：[jianghushinian007@outlook.com](mailto:jianghushinian007@outlook.com)
- 博客地址：https://jianghushinian.cn

**参考**

- Go embed：https://pkg.go.dev/embed
