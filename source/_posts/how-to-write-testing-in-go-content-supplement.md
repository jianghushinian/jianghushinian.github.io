---
title: 在 Go 中如何编写测试代码内容补充
date: 2023-07-23 11:37:55
tags:
- Go
- Test
categories:
- Go
---

最近一直在写 Go 语言测试相关的文章，从[《在 Go 中如何编写测试代码》](https://jianghushinian.cn/2023/07/09/how-to-write-testing-in-go/)开始，已经更新了七篇文章。本来打算测试系列文章就此告一段落，近期不再写相关内容了。但是重读一遍这个系列的文章，发现还是有一些遗漏的知识点，和之前文章中提到过但没有深入讲解的内容。本文就作为这个系列文章的一个补充，讲解下我认为在 Go 测试中还有哪些值得一写的内容。

<!-- more -->

### 准备

本文所有测试用例都是基于 `Abs` 函数编写的，`Abs` 函数定义如下：

```go
func Abs(x int) int {
	fmt.Printf(">>> call abs(%d)\n", x)
	if x < 0 {
		return -x
	}
	return x
}
```

### TestMain

有些时候，我们可能需要在测试前执行一些准备工作，测试后执行一些清理工作。比如测试前启动一个测试用的 HTTP Server，测试后关闭这个 Server。`TestMain` 函数就是用来干这个的，它相当于测试中的 `main` 函数。

`TestMain` 函数参数为 `*testing.M` 类型，测试开始时会最先被执行，在 `TestMain` 函数中可以调用 `(*testing.M).Run()`，这会执行全部的测试用例。利用这个特性，我们可以在调用 `(*testing.M).Run()` 之前执行测试准备工作，在调用 `(*testing.M).Run()` 之后执行清理工作。

为 `Abs` 函数编写单元测试代码如下：

```go
func TestAbs(t *testing.T) {
	want := 1
	if got := Abs(-1); got != want {
		t.Fatalf("Abs() = %v, want %v", got, want)
	}
}

func setup() {
	fmt.Println("> setup completed")
}

func teardown() {
	fmt.Println("> teardown completed")
}

func TestMain(m *testing.M) {
	setup()
	code := m.Run()
	teardown()
	os.Exit(code)
}
```

`setup()` 可以用来执行准备工作，`teardown()` 用来执行清理工作，`m.Run()` 执行全部测试用例后会返回程序退出码，可以在程序退出时传递给 `os.Exit()` 函数。

使用 `go test` 来执行测试函数：

```bash
$ go test -v
> setup completed
=== RUN   TestAbs
>>> call abs(-1)
--- PASS: TestAbs (0.00s)
PASS
> teardown completed
ok      github.com/jianghushinian/blog-go-example/test/supplement       0.369s
```

执行结果符合预期。

### Setup/Teardown

`TestMain` 函数是全局粒度的，有些时候，我们想要单独为某一个测试用例实现准备和清理函数，可以这样做：

```go
func TestAbs(t *testing.T) {
	teardownTest := setupTest(t)
	defer teardownTest(t)
	want := 1
	if got := Abs(-1); got != want {
		t.Fatalf("Abs() = %v, want %v", got, want)
	}
}

// testing.TB is the interface common to T, B, and F.
func setupTest(tb testing.TB) func(tb testing.TB) {
	fmt.Println(">> setup Test")

	return func(tb testing.TB) {
		fmt.Println(">> teardown Test")
	}
}
```

定义 `setupTest` 函数为某个测试用例提供准备工作，它接收参数为 `testing.TB` 接口，testing 框架中的 `*testing.<T|B|F>` 都实现了这个接口，所以 `setupTest` 函数在单元测试、基准测试、模糊测试中都可以使用。`setupTest` 函数返回的函数可以用来执行清理工作。

使用 `go test` 来执行测试函数：

```bash
$ go test -v
> setup completed
=== RUN   TestAbs
>> setup Test
>>> call abs(-1)
>> teardown Test
--- PASS: TestAbs (0.00s)
PASS
> teardown completed
ok      github.com/jianghushinian/blog-go-example/test/supplement       0.518s
```

### 表格测试

如果我们想为一个函数编写多个测试用例，则可以使用表格测试（`table-driven tests`）。

表格测试将所有的测试用例保存在结构体切片中，然后在 `for` 循环中依次执行每个测试用例，实现代码如下：

```go
func TestAbsWithTable(t *testing.T) {
	type args struct {
		x int
	}
	tests := []struct {
		name string
		args args
		want int
	}{
		{
			name: "positive",
			args: args{x: 1},
			want: 1,
		},
		{
			name: "negative",
			args: args{x: -1},
			want: 1,
		},
	}
	for _, tt := range tests {
		teardownTest := setupTest(t)
		defer teardownTest(t)
		if got := Abs(tt.args.x); got != tt.want {
			t.Fatalf("Abs() = %v, want %v", got, tt.want)
		}
	}
}
```

使用 `go test` 来执行测试函数：

```bash
$ go test -v
> setup completed
=== RUN   TestAbsWithTable
>> setup Test
>>> call abs(1)
>> setup Test
>>> call abs(-1)
>> teardown Test
>> teardown Test
--- PASS: TestAbsWithTable (0.00s)
PASS
> teardown completed
ok      github.com/jianghushinian/blog-go-example/test/supplement       0.550s
```

根据执行结果可以发现，每个测试用例是顺序执行的。但是存在一个问题，虽然每个测试用例的准备函数 `setupTest(t)` 是在当前轮次循环中进行调用的，可清理函数 `teardownTest(t)` 却是在 `for` 循环执行完成后，才会被依次执行。这是 Go 语言在 `for` 循环中使用 `defer` 语句天然存在的问题，如果你不知道如何解决，可以参考我的另一篇文章[《在 Go 中如何实现类似 Python 中的 with 上下文管理器》](https://jianghushinian.cn/2023/06/23/how-to-implement-a-context-manager-similar-to-python-s-with-in-go/)。

现在我们尝试将第一个测试用例故意改错，将 `want` 值修改为 `2`：

```go
{
	name: "positive",
	args: args{x: 1},
	want: 2,
}
```

再次使用 `go test` 来执行测试函数：

```bash
$ go test -v -run="TestAbsWithTable"
> setup completed
=== RUN   TestAbsWithTable
>> setup Test
>>> call abs(1)
    main_test.go:42: Abs() = 1, want 2
>> teardown Test
--- FAIL: TestAbsWithTable (0.00s)
FAIL
> teardown completed
exit status 1
FAIL    github.com/jianghushinian/blog-go-example/test/supplement       0.541s
```

可以发现，第一个测试用例执行报错了，并且测试直接终止，没有继续执行第二个测试用例。

这个表现与我们直接在多个函数中编写的测试用例有所不同，如果我们像下面这样定义两个测试用例：

```go
func TestAbs1(t *testing.T) {
	teardownTest := setupTest(t)
	defer teardownTest(t)
	want := 2
	if got := Abs(-1); got != want {
		t.Fatalf("Abs() = %v, want %v", got, want)
	}
}

func TestAbs2(t *testing.T) {
	teardownTest := setupTest(t)
	defer teardownTest(t)
	want := 1
	if got := Abs(-1); got != want {
		t.Fatalf("Abs() = %v, want %v", got, want)
	}
}
```

那么当使用 `go test` 执行这两个测试用例时，`TestAbs1` 同样会失败退出，但这并不会影响 `TestAbs2` 测试用例的执行。

要解决这个问题，就该 `Subtests` 登场了。

### Subtests

Subtests 被译为子测试，子测试可以解决我们在表格测试中遇到的所有问题。

Subtests 用法很简单，仅需要将我们原来在表格测试时 `for` 循环中执行的代码，迁移到 `(*testing.T).Run()` 函数中来执行即可，实现如下：

```go
func TestAbsWithTableAndSubtests(t *testing.T) {
	type args struct {
		x int
	}
	tests := []struct {
		name string
		args args
		want int
	}{
		{
			name: "positive",
			args: args{x: 1},
			want: 2,
		},
		{
			name: "negative",
			args: args{x: -1},
			want: 1,
		},
	}
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			teardownTest := setupTest(t)
			defer teardownTest(t)
			if got := Abs(tt.args.x); got != tt.want {
				t.Fatalf("Abs() = %v, want %v", got, tt.want)
			}
		})
	}
}
```

我们将原来的这段代码：

```go
teardownTest := setupTest(t)
defer teardownTest(t)
if got := Abs(tt.args.x); got != tt.want {
	t.Fatalf("Abs() = %v, want %v", got, tt.want)
}
```

迁移到了 `t.Run()` 中：

```go
t.Run(tt.name, func(t *testing.T) {
	teardownTest := setupTest(t)
	defer teardownTest(t)
	if got := Abs(tt.args.x); got != tt.want {
		t.Fatalf("Abs() = %v, want %v", got, tt.want)
	}
})
```

`t.Run()` 第一个参数用来记录测试用例名称，第二个参数是一个匿名的函数，参数为 `*testing.T`，可以将表格测试中 `for` 内的代码全部放在这个匿名函数中来执行。

使用 `go test` 来执行测试函数：

```bash
$ go test -v -run="TestAbsWithTableAndSubtests"
> setup completed
=== RUN   TestAbsWithTableAndSubtests
=== RUN   TestAbsWithTableAndSubtests/positive
>> setup Test
>>> call abs(1)
    main_test.go:72: Abs() = 1, want 2
>> teardown Test
=== RUN   TestAbsWithTableAndSubtests/negative
>> setup Test
>>> call abs(-1)
>> teardown Test
--- FAIL: TestAbsWithTableAndSubtests (0.00s)
    --- FAIL: TestAbsWithTableAndSubtests/positive (0.00s)
    --- PASS: TestAbsWithTableAndSubtests/negative (0.00s)
FAIL
> teardown completed
exit status 1
FAIL    github.com/jianghushinian/blog-go-example/test/supplement       0.524s
```

可以发现，使用 Subtests 后，即使第一个测试用例执行退出了，也不会影响第二个测试用例的执行。并且，这也避免了在 `for` 循环中直接使用 `defer` 语句，`teardownTest(t)` 函数的执行时机也正常了。

此外，我们还可以发现，Subtests 是有层级关系的，并且每一个测试用例成功和失败都会被单独标记：

```bash
--- FAIL: TestAbsWithTableAndSubtests (0.00s)
    --- FAIL: TestAbsWithTableAndSubtests/positive (0.00s)
    --- PASS: TestAbsWithTableAndSubtests/negative (0.00s)
```

根据这几行日志，我们能够发现 `TestAbsWithTableAndSubtests` 测试执行失败了，它包含了两个子测试，其中 `TestAbsWithTableAndSubtests/positive` 子测试执行失败，而 `TestAbsWithTableAndSubtests/negative` 子测试执行通过。子测试的名称是测试函数名 + `/` + 传递给 `t.Run()` 的第一个参数 `tt.name`。

我们可以单独指定需要执行的子测试，这也是普通的表格测试无法做到的。

使用 `go test` 执行子测试：

```bash
go test -v -run="TestAbsWithTableAndSubtests/negative"
> setup completed
=== RUN   TestAbsWithTableAndSubtests
=== RUN   TestAbsWithTableAndSubtests/negative
>> setup Test
>>> call abs(-1)
>> teardown Test
--- PASS: TestAbsWithTableAndSubtests (0.00s)
    --- PASS: TestAbsWithTableAndSubtests/negative (0.00s)
PASS
> teardown completed
ok      github.com/jianghushinian/blog-go-example/test/supplement       0.569s
```

不过，Subtests 也存在缺点，就是不支持并发执行。

如果想让其支持并行，可以使用 `t.Parallel()` 将其标记为可并行执行：

```go
func TestAbs(t *testing.T) {
    for _, tt := range tests {
        t.Run(tt.Name, func(t *testing.T) {
            t.Parallel()
            ...
        })
    }
}
```

Subtests 的常见用法我们就介绍完了。

根据上面的测试结果，我们要牢记，表格测试一定要与子测试一起配合使用。

### 自动生成测试代码

通过前文的示例讲解，我们不难发现，其实表格测试是有套路的，表格测试基本框架如下：

```go
func TestAbs(t *testing.T) {
	type args struct {
		x int
	}
	tests := []struct {
		name string
		args args
		want int
	}{
		// TODO: Add test cases.
	}
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			if got := Abs(tt.args.x); got != tt.want {
				t.Errorf("Abs() = %v, want %v", got, tt.want)
			}
		})
	}
}
```

既然基本测试框架不会变化，那么我们就可以使用程序来自动生成单元测试模板代码。

`gotests` 就提供了这样的功能，可以使用如下命令进行安装：

```bash
$ go get -u github.com/cweill/gotests/...
```

使用如下命令为所有代码生成测试：

```bash
$ gotests -all -w .
```

`-all` 表示为所有代码都生成测试。

`-w` 表示将输出写入指定文件，而不是标准输出。后面的 `.` 参数代表当前目录，`gotests` 会在当前目录下创建 `xxx_test.go` 文件并写入生成的测试代码模板。

如果只为 `Abs` 函数生成测试用例，可以使用 `-only` 参数：

```bash
$ gotests -only Abs -w .
Generated TestAbs
```

输出 `Generated TestAbs` 表示测试代码模板已经生成。

如下输出则表示没有生成测试，可能是测试函数已经存在。

```bash
No tests generated for .
```

`-only` 标志接收正则参数，可以指定为 `Abs`、`Add` 两个函数生成测试：

```bash
$ gotests -only "Abs|Add" -w .
```

`-excl` 标志与 `-only` 标志作用相反，为指定的函数 `Abs` 以外的其他函数生成测试：

```bash
$ gotests -excl Abs -w .
```

`gotests` 更多功能可以使用 `gotests --help` 进行查看。

### 何时编写测试

我花了好几篇文章来讲解如何编写测试代码，但还没讲解过应该在何时编写测试代码，现在就来简单聊聊这个话题。

根据开发周期来看，编写测试的时机有三个：

1. 编写代码之前，先编写测试代码，即测试驱动开发 —— TDD。

2. 编写代码过程中，每实现一个小功能（函数、方法等），就为这个小功能编写测试代码，相当于同步进行。

3. 整个项目前期先不写测试，等项目上线稳定后，再统一为项目编写测试代码。

在这里，我最推荐第二种做法。

在我了解的国内开发团队，很少有使用 TDD 模式来编写测试代码的（也可能是我见识比较少），这跟市场环境有关。不过 TDD 开发模式在很多外企非常流行，比如 Thoughtworks 就在采用 TDD 开发模式，所以 Thoughtworks 也培养了一批大师级的 TDD 信奉者。

至于第三种做法，根据我的经验，基本上是项目周期非常短，着急上线，但最后的结果大概率是没有动力再为项目编写测试代码的。

### 何时执行测试

我们再来聊聊何时执行测试代码。

首先，我们每写一个单元测试，都要立即执行，验证测试程序和被测程序的正确性，这个过程中可能要多次修改和执行单元测试。

其次，在每个 feature 完成后，要执行当前项目的全部测试代码，以此来验证开发当前的功能是否对其他功能组件产生影响，这也被称作回归测试。

接着，代码会被 push 到代码仓库，此时应该在 CI 环境完整的执行一次全部测试代码，以此来保证代码被合并到主分支前是没有问题的。不过，这个过程中，可能有些测试没必要在 CI 环境执行，那么可以使用 `(*testing.T).Skip` 跳过这些单元测试。

最终被合并到主分支的代码，一定是测试全部通过的。

### 总结

本文中我们介绍了 `TestMain` 的用法，以及利用 `TestMain` 来实现全局的 `setup/teardown` 函数。接着又教大家如何实现单个测试用例级别的 `setup/teardown` 函数。

我们还学习了表格测试和子测试的用法，搞清楚了为什么要用子测试以及它能解决哪些问题。我们应该牢记，表格测试一定要与子测试一起配合使用。

最后，我又和大家聊了编写单元测试的时机以及执行单元测试的时机。

至此，关于在 Go 中如何编写测试相关的文章就告一段落了。

本文完整代码示例我放在了 [GitHub](https://github.com/jianghushinian/blog-go-example/tree/main/test/supplement) 上，欢迎点击查看。

希望此文能对你有所帮助。

### P.S.

本文完结后，我写的这一系列关于测试的文章，应该可以覆盖到我们平时工作和开发中超过 90% 的测试场景。虽然可能还会有一些遗漏的内容没有讲解，但既然被遗漏，说明不太常见，所以这个系列的文章暂时不会再继续深究下去，很长一段时间内我应该不会再写这个主题的文章了。

如果你在编写测试代码方面有一些心得想和我讨论分享，欢迎你联系我。

**联系我**

- 微信：jianghushinian
- 邮箱：[jianghushinian007@outlook.com](mailto:jianghushinian007@outlook.com)
- 博客地址：https://jianghushinian.cn

**参考**

- testing 文档：https://pkg.go.dev/testing@go1.20.1
- Using Setup and Teardown in Golang's Tests：https://www.sobyte.net/post/2022-07/go-setup-and-teardown/
- gotests 源码仓库：https://github.com/cweill/gotests
