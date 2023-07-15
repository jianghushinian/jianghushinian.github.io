---
title: 在 Go 中如何编写测试代码
date: 2023-07-09 20:55:55
tags:
- Go
- Test
categories:
- Go
---

在程序开发过程中，测试是非常重要的一环，甚至有一种开发模式叫 TDD（测试驱动开发），先编写测试，再编写功能代码，通过测试来推动整个开发的进行，可见测试在开发中的重要程度。

为此，Go 语言提供了 `testing` 框架来方便我们编写测试，本文将向大家介绍在 Go 中如何编写测试代码。

<!-- more -->

### 测试分类

在 Go 中，编写的测试用例可以分为四类：

- 单元测试：测试函数名称以 `Test` 开头，如 `TestXxx`、`Test_Xxx`，用来对程序的最小单元进行测试，如函数、方法等。

- 基准测试：也叫性能测试，测试函数名称以 `Benchmark` 开头，用来测量程序的性能指标。

- 示例测试：测试函数名称以 `Example` 开头，可以用来测试程序的标准输出内容。

- 模糊测试：也叫随机测试，测试函数名称以 `Fuzz` 开头，是一种基于随机输入的自动化测试技术，适合用来测试处理用户输入的代码，在 Go 1.18 中被引入。

这四类测试用例都有各自的适用场景，其中单元测试最为常用，你一定要掌握。

以上这些测试用例都可以使用 `go test` 命令来执行。

### 测试规范

编写测试代码并不需要我们学习新的 Go 语法，但有些测试规范还是需要遵守的。

Go 在提供 `testing` 测试框架时，就规定了很多测试规范，用来约束我们编写测试的方式，这有助于项目的工程化。

#### 测试文件命名规范

首先，测试文件命名必须以 `_test.go` 结尾，否则将被测试框架忽略。比如我们的 Go 代码文件名为 `hello.go`，则测试文件可以命名为 `hello_test.go`。

只有以 `_test.go` 结尾的测试文件，才能使用 `go test` 命令执行。

在构建 Go 程序时，`go build` 命令会忽略以 `_test.go` 结尾的测试文件。

#### 测试包命名规范

测试用例除了根据使用场景可以分为四类，还可以根据代码和测试用例是否在一个包中，分为白盒测试和黑盒测试。

- 白盒测试：将测试代码和被测代码放在同一个包中，也就是二者包名相同，这些测试用例属于白盒测试。比如 Go 代码文件名为 `hello.go`，包名为 `hello`，测试文件可以命名为 `hello_test.go`，并且必须与 `hello.go` 放在同一个目录下，包名也必须为 `hello`。白盒测试的测试用例可以使用和测试当前包中所有标识符（变量、函数等），包括未导出的标识符。

- 黑盒测试：将测试代码和被测代码放在不同的包中，即包名不同，这些测试用例属于黑盒测试。比如 Go 代码文件名为 `hello.go`，包名为 `hello`，测试文件同样可以命名为 `hello_test.go`，与 `hello.go` 放在同一个目录下，但包名不能再叫 `hello`，应该命名为 `hello_test`，`hello_test.go` 文件也可以放在专门的 `test` 目录下，此时可以随意命名包名。黑盒测试的测试用例仅能够使用和测试被测代码包中可导出的标识符，因为二者已经不再属于同一个包，这遵循 Go 语法规范。

根据二者各自特点，在开发时我们应该多编写白盒测试，这样才能提升代码测试覆盖率。

#### 测试用例命名规范

在 Go 中我们使用测试函数来编写测试用例，根据单元测试、基准测试、示例测试、模糊测试四种不同类型的测试分类，测试函数必须以 `Test`、`Benchmark`、`Example`、`Fuzz` 其中一种开头。

测试函数签名示例如下：

```go
func TestXxx(*testing.T)
func BenchmarkXxx(*testing.B)
func ExampleXxx()
func FuzzXxx(*testing.F)
```

测试函数不能有返回值，其中单元测试、基准测试和模糊测试都接收一个参数，由 `testing` 框架提供，示例测试则不需要传递参数。

其中 `Xxx` 一般是被测试的函数名称，首字母必须大写。如果是以 `Test_Xxx` 方式命名测试函数，则 `Xxx` 首字母大小写均可。

#### 测试变量命名规范

对于测试变量的命名，`testing` 框架没有有强制约束，但社区中也形成了一些规范。

比如，函数签名中的参数变量定义如下：

```go
func TestXxx(t *testing.T)
func BenchmarkXxx(b *testing.B)
func FuzzXxx(f *testing.F)
```

单元测试、基准测试和模糊测试参数变量即为参数类型 `*testing.<T>` 的小写形式。

在编写测试代码时，有一个最常见的场景，就是比较被测函数的实际输出和测试函数中的预期输出是否相等，通常可以使用 `got/want` 或 `actual/expected` 来命名变量：

```go
if got != want {
	t.Errorf("Xxx(x) = %s; want %s", got, want)
}
```

或：

```go
if actual != expected {
	t.Errorf("Xxx(x) = %s; expected %s", actual, expected)
}
```

读者可以根据喜好和团队中的开发规范选择其中一种变量命名。

此外，在单元测试中我们还会经常编写一种叫表格测试的测试用例，写法如下：

```go
func TestXxx(t *testing.T) {
	tests := []struct {
		name string
		arg  float64
		want float64
	}{
	...
	}
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			if got := Xxx(tt.arg); got != tt.want {
				t.Errorf("Xxx(%f) = %v, want %v", tt.arg, got, tt.want)
			}
		})
	}
}
```

其中 `tests` 代表多个测试用例，循环时以 `tt` 作为循环变量（`tt` 可以避免与单元测试函数的参数变量 `t` 命名冲突）。

表格测试还有另一个版本：

```go
func TestXxx(t *testing.T) {
	cases := []struct {
		name string
		arg  float64
		want float64
	}{
	...
	}
	for _, cc := range cases {
		t.Run(cc.name, func(t *testing.T) {
			if got := Xxx(cc.arg); got != cc.want {
				t.Errorf("Xxx(%f) = %v, want %v", cc.arg, got, cc.want)
			}
		})
	}
}
```

现在 `cases` 代表多个测试用例，循环时以 `cc` 作为循环变量（`cc` 可以避免与常见的 `context` 缩写 `c` 命名冲突）。

编写测试代码的常见规范我们就先讲解到这里，更多规范将在下文讲解对应示例时再进行详细说明。

### 单元测试

单元测试是我们最常编写的测试用例，所以先来学习下如何编写单元测试。

首先，我们准备一个 `Abs` 函数作为被测试的代码，存放于 `abs.go` 文件中，其包名为 `abs`，代码如下：

```go
package abs

import "math"

func Abs(x float64) float64 {
	return math.Abs(x)
}
```

#### 白盒测试

现在为 `Abs` 编写一个白盒测试函数，在存放 `abs.go` 文件的同一目录下，新建 `abs_test.go` 文件，包名同样定义为 `abs`，编写测试代码如下：

```go
package abs

import "testing"

func TestAbs(t *testing.T) {
	got := Abs(-1)
	if got != 1 {
		t.Errorf("Abs(-1) = %f; want 1", got)
	}
}
```

单元测试函数 `TestAbs` 代码非常简单，先调用了 `Abs(-1)` 函数，并将得到的返回结果 `got` 与 `1` 做相等性比较，如果不相等，则说明测试没有通过，使用 `t.Errorf` 打印错误信息。

参数 `*testing.T` 是一个结构体指针，提供了如下几个方法用于错误报告：

- `t.Log/t.Logf`：打印正常日志信息，类似 `fmt.Print`。

- `t.Error/t.Errorf`：打印测试失败时的错误信息，不影响当前测试函数内后续代码的继续执行。

- `t.Fatal/t.Fatalf`：打印测试失败时的错误信息，并终止当前测试函数执行。

在测试函数所在目录下使用 `go test` 命令执行测试代码：

```bash
$ go test
PASS
ok      github.com/jianghushinian/blog-go-example/test/getting-started/abs      0.139s
```

`go test` 会自动查找当前目录下所有以 `_test.go` 结尾来命名的测试文件，并执行其内部编写的全部测试函数。

输出 `PASS` 表示测试通过，`github.com/jianghushinian/blog-go-example/test/getting-started/abs` 是程序的 `module` 名称。

`go test` 命令还支持使用 `-v` 标志输出更多信息：

```bash
$ go test -v
=== RUN   TestAbs
--- PASS: TestAbs (0.00s)
PASS
ok      github.com/jianghushinian/blog-go-example/test/getting-started/abs      0.437s
```

如果我们不小心将单元测试的函数名错误的写成 `Testabs`，即 `abs` 没有大写开头：

```go
func Testabs(t *testing.T) {
	got := Abs(-1)
	if got != 1 {
		t.Errorf("Abs(-1) = %f; want 1", got)
	}
}
```

则测试函数不会被 `go test` 命令执行。

通常情况下，我们不会对一个函数只做一种输入参数的测试，为了提高测试覆盖率，我们可能还需要多测试几种参数的用例，比如测试下 `Abs(2)`、`Abs(3)` 等是否正确。

这时，可以像如下这样编写测试函数：

```go
func TestAbs(t *testing.T) {
	got := Abs(-1)
	if got != 1 {
		t.Errorf("Abs(-1) = %f; want 1", got)
	}

	got = Abs(2)
	if got != 2 {
		t.Errorf("Abs(2) = %f; want 2", got)
	}
}
```

但这样的代码显然过于“平铺直叙”，不够优雅。

在这种更加复杂的情况下，我们可以使用「表格测试」，代码如下：

```go
func TestAbs_TableDriven(t *testing.T) {
	tests := []struct {
		name string
		x    float64
		want float64
	}{
		{
			name: "positive",
			x:    2,
			want: 2,
		},
		{
			name: "negative",
			x:    -3,
			want: 3,
		},
	}
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			if got := Abs(tt.x); got != tt.want {
				t.Errorf("Abs(%f) = %v, want %v", tt.x, got, tt.want)
			}
		})
	}
}
```

为了便于与之前编写的测试函数 `TestAbs` 区分，我为当前测试函数命名为 `TestAbs_TableDriven`，代表这是一个表格驱动的测试。

在测试函数内部，首先定义了一个匿名结构体切片，用来保存多个测试用例。

`name` 是一个字符串，可以是任何句子，用来标记当前测试用例所测试的场景，这样代码维护者通过 `name` 字段就能够知道当前用例所测试的场景，作用相当于代码注释。

`x` 作为 `Abs` 函数的入参，其类型等同于 `Abs` 函数的参数，如果被测试函数有多个参数，这里也可以使用一个结构体来保存。

`want` 记录当前测试用例的期望值。

在 `for` 循环中，我们可以使用 `*testing.T` 提供的 `t.Run` 方法执行测试用例，这和直接编写的 `TestXxx` 测试函数没什么本质区别。

现在使用 `go test` 命令执行测试代码：

```bash
go test -v
=== RUN   TestAbs
--- PASS: TestAbs (0.00s)
=== RUN   TestAbs_TableDriven
=== RUN   TestAbs_TableDriven/positive
=== RUN   TestAbs_TableDriven/negative
--- PASS: TestAbs_TableDriven (0.00s)
    --- PASS: TestAbs_TableDriven/positive (0.00s)
    --- PASS: TestAbs_TableDriven/negative (0.00s)
PASS
ok      github.com/jianghushinian/blog-go-example/test/getting-started/abs      0.145s
```

可以发现，表格测试的输出信息更加丰富，能够分别打印出表格中的每一个测试用例，并且使用缩进来展示层级关系。

现在我们故意将其中的一个测试用例改错：

```go
{
	name: "negative",
	x:    -3,
	want: 33,
}
```

再次使用 `go test` 命令执行测试代码看下如何输出：

```bash
go test -v
=== RUN   TestAbs
--- PASS: TestAbs (0.00s)
=== RUN   TestAbs_TableDriven
=== RUN   TestAbs_TableDriven/positive
=== RUN   TestAbs_TableDriven/negative
    abs_test.go:36: Abs(-3.000000) = 3, want 33
--- FAIL: TestAbs_TableDriven (0.00s)
    --- PASS: TestAbs_TableDriven/positive (0.00s)
    --- FAIL: TestAbs_TableDriven/negative (0.00s)
FAIL
exit status 1
FAIL    github.com/jianghushinian/blog-go-example/test/getting-started/abs      0.515s
```

根据打印结果，我们很容易能够发现是 `TestAbs_TableDriven` 测试函数中 `negative` 这个测试用例执行失败了。

有些场景下，我们可能想要跳过某些测试用例，可以使用 `(*testing.T).Skip` 方法来实现：

```go
func TestAbs_Skip(t *testing.T) {
	// CI 环境跳过当前测试
	if os.Getenv("CI") != "" {
		t.Skip("it's too slow, skip when running in CI")
	}

	t.Log(t.Skipped())

	got := Abs(-2)
	if got != 2 {
		t.Errorf("Abs(-2) = %f; want 2", got)
	}
}
```

假如 `TestAbs_Skip` 是一个非常耗时的测试用例，我们就可以使用 `t.Skip` 在 CI 环境下跳过此测试。

`t.Skipped()` 返回当前测试用例是否被跳过。

使用 `go test` 命令执行测试：

```bash
$ CI=1 go test -v -run="TestAbs_Skip"
=== RUN   TestAbs_Skip
    abs_test.go:46: it's too slow, skip when running in CI
--- SKIP: TestAbs_Skip (0.00s)
PASS
ok      github.com/jianghushinian/blog-go-example/test/getting-started/abs      0.103s
```

这次我们使用 `-run` 参数来指定想要执行的测试用例，`-run` 参数的值支持正则。

并且指定了环境变量 `CI=1`。

从打印结果来看，`TestAbs_Skip` 测试用例的确被跳过了，所以 `t.Log(t.Skipped())` 没有被执行到。

默认情况下，测试用例是从上到下按照顺序执行的，不过，我们可以使用 `(*testint.T).Parallel` 来标记一个测试函数支持并发执行：

```go
func TestAbs_Parallel(t *testing.T) {
	t.Log("Parallel before")
	// 标记当前测试支持并行
	t.Parallel()
	t.Log("Parallel after")

	got := Abs(2)
	if got != 2 {
		t.Errorf("Abs(2) = %f; want 2", got)
	}
}
```

只有一个测试函数支持并发执行意义不大，我们可以将 `TestAbs` 测试函数也修改为支持并发执行：

```go
func TestAbs(t *testing.T) {
	t.Parallel()
	got := Abs(-1)
	if got != 1 {
		t.Errorf("Abs(-1) = %f; want 1", got)
	}
}
```

现在，使用 `go test` 命令来测试下并发执行测试用例：

```bash
$ go test -v -run=".*Parallel.*|^TestAbs$"
=== RUN   TestAbs
=== PAUSE TestAbs
=== RUN   TestAbs_Parallel
    abs_test.go:59: Parallel before
=== PAUSE TestAbs_Parallel
=== CONT  TestAbs
--- PASS: TestAbs (0.00s)
=== CONT  TestAbs_Parallel
    abs_test.go:62: Parallel after
--- PASS: TestAbs_Parallel (0.00s)
PASS
ok      github.com/jianghushinian/blog-go-example/test/getting-started/abs      0.200s
```

这里我们只执行了 `TestAbs_Parallel`、`TestAbs` 这两个测试函数。

可以发现，两个函数都不是一次性执行完成的，日志中 `PAUSE` 表示暂停当前函数的执行，`CONT` 表示恢复当前函数执行。

有时候，我们测试的并不是一个函数，而是一个方法，比如我们想要测试 `Animal` 结构体的 `shout` 方法：

```go
package animal

type Animal struct {
	Name string
}

func (a Animal) shout() string {
	if a.Name == "dog" {
		return "旺！"
	}
	if a.Name == "cat" {
		return "喵～"
	}
	return "吼～"
}
```

那么，测试函数可以命名为 `TestAnimal_shout`，如下是我们针对 `Dog` 和 `Cat` 两种不同的 `Animal` 对象编写的测试代码：

```go
package animal

import (
	"testing"
)

func TestAnimalDog_shout(t *testing.T) {
	dog := Animal{Name: "dog"}
	got := dog.shout()
	want := "旺！"
	if got != want {
		t.Errorf("got %s; want %s", got, want)
	}
}

func TestAnimalCat_shout(t *testing.T) {
	cat := Animal{Name: "cat"}
	got := cat.shout()
	want := "喵～"
	if got != want {
		t.Errorf("got %s; want %s", got, want)
	}
}
```

#### 黑盒测试

讲完了白盒测试，我们再来演示下如何编写黑盒测试。

要为 `Abs` 编写黑盒测试非常简单，我们只需要将 `TestAbs` 移动到新的包中即可。

```go
package abs_test

import (
	"testing"

	"github.com/jianghushinian/blog-go-example/test/getting-started/abs"
)

func TestAbs(t *testing.T) {
	got := abs.Abs(-1)
	if got != 1 {
		t.Errorf("Abs(-1) = %f; want 1", got)
	}
}
```

因为黑盒测试的函数 `TestAbs` 与 `Abs` 不在同一个包中，所以需要先使用 `import` 导入 `abs` 包，之后才能使用 `abs.Abs` 函数。

至此，常见的单元测试场景我们就介绍完了。

接下来，我们一起来看如何编写基准测试。

### 基准测试

基准测试也叫性能测试，顾名思义，是为了度量程序的性能。

为 `Abs` 编写的基准测试代码如下：

```go
func BenchmarkAbs(b *testing.B) {
	for i := 0; i < b.N; i++ {
		Abs(-1)
	}
}
```

基准测试同样放在 `abs_test.go` 文件中，以 `Benchmark` 开头，参数不再是 `*testing.T`，而是 `*testing.B`，在测试函数中，我们循环了 `b.N` 次调用 `Abs(-1)`，`b.N` 的值是一个动态值，我们无需操心，`testing` 框架会为其分配合理的值，以使测试函数运行足够多的次数，可以准确的计时。

默认情况下，`go test` 命令并不会运行基准测试，需要指定 `-bench` 参数：

```bash
$ go test -bench="." 
goos: darwin
goarch: arm64
pkg: github.com/jianghushinian/blog-go-example/test/getting-started/abs
BenchmarkAbs-8          1000000000               0.5096 ns/op
PASS
ok      github.com/jianghushinian/blog-go-example/test/getting-started/abs      0.674s
```

`-bench` 参数同样接收一个正则，`.` 匹配所有基准测试。

我们需要重点关注的是这行结果：

```bash
BenchmarkAbs-8          1000000000               0.5096 ns/op
```

`BenchmarkAbs-8` 中，`BenchmarkAbs` 是测试函数名，`8` 是 `GOMAXPROCS` 的值，即参与执行的 CPU 核心数。

`1000000000` 表示测试执行了这么多次。

`0.5096 ns/op` 表示每次循环平均消耗的纳秒数。

如果还想查看基准测试的内存统计情况，则可以指定 `-benchmem` 参数：

```bash
$ go test -bench="BenchmarkAbs$" -benchmem           
goos: darwin
goarch: arm64
pkg: github.com/jianghushinian/blog-go-example/test/getting-started/abs
BenchmarkAbs-8          1000000000               0.5097 ns/op          0 B/op          0 allocs/op
PASS
ok      github.com/jianghushinian/blog-go-example/test/getting-started/abs      0.681s
```

现在，`BenchmarkAbs-8` 这行得到了更多输出：

```bash
BenchmarkAbs-8          1000000000               0.5097 ns/op          0 B/op          0 allocs/op
```

`0 B/op` 表示每次执行测试代码分配了多少字节内存。

`0 allocs/op` 表示每次执行测试代码分配了多少次内存。

此外，在执行 `go test` 命令时，我们可以使用 `-benchtime=Ns` 参数指定基准测试函数执行时间为 `N` 秒：

```bash
$ go test -bench="BenchmarkAbs$" -benchtime=0.1s 
goos: darwin
goarch: arm64
pkg: github.com/jianghushinian/blog-go-example/test/getting-started/abs
BenchmarkAbs-8          210435709                0.5096 ns/op
PASS
ok      github.com/jianghushinian/blog-go-example/test/getting-started/abs      0.600s
```

`-benchtime` 参数值为 `time.Duration` 类型支持的时间格式。

`-benchtime` 参数还有一个特殊语法 `-benchtime=Nx` 参数，可以指定基准测试函数执行次数为 `N` 次：

```bash
$ go test -bench="BenchmarkAbs$" -benchtime=10x
goos: darwin
goarch: arm64
pkg: github.com/jianghushinian/blog-go-example/test/getting-started/abs
BenchmarkAbs-8                10                20.90 ns/op
PASS
ok      github.com/jianghushinian/blog-go-example/test/getting-started/abs      0.391s
```

有时候，我们在编写基准测试时，被测函数可能需要一些准备数据，而这些准备数据的时间不应该算做被测试函数的耗时。

此时，可以使用 `(*testing.B).ResetTimer` 重置计时：

```go
func BenchmarkAbsResetTimer(b *testing.B) {
	time.Sleep(100 * time.Millisecond) // 模拟耗时的准备工作
	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		Abs(-1)
	}
}
```

这样，在调用 `b.ResetTimer()` 之前的耗时操作将不被记入测试结果的耗时中。

还有一种方法，也可以跳过准备工作的计时，即先使用 `(*testing.B).StopTimer` 停止计时，耗时的准备工作完成后再使用 `(*testing.B).StartTimer` 恢复计时：

```go
func BenchmarkAbsStopTimerStartTimer(b *testing.B) {
	b.StopTimer()
	time.Sleep(100 * time.Millisecond) // 模拟耗时的准备工作
	b.StartTimer()
	for i := 0; i < b.N; i++ {
		Abs(-1)
	}
}
```

默认情况下，基准测试 `for` 循环中的代码是串行执行的，如果想要并行执行，可以将被测试代码的调用放在 `(*testing.B).RunParallel` 中：

```go
func BenchmarkAbsParallel(b *testing.B) {
	b.RunParallel(func(pb *testing.PB) {
		for pb.Next() {
			Abs(-1)
		}
	})
}
```

我们还可以使用 `(*testing.B).SetParallelism` 控制并发协程数：

```go
func BenchmarkAbsParallel(b *testing.B) {
	b.SetParallelism(2) // 设置并发 Goroutines 数量为 2 * GOMAXPROCS
	b.RunParallel(func(pb *testing.PB) {
		for pb.Next() {
			Abs(-1)
		}
	})
}
```

在使用 `go test` 命令执行基准测试时，可以指定 `-cpu` 参数来设置 `GOMAXPROCS`。

要想了解 `go test` 支持的更多参数，可以使用 `go help testflag` 命令进行查看。

### 示例测试

示例测试以 `Example` 开头，无参数和返回值，通常存放在 `example_test.go` 文件中。

 约定一个包、函数 F、类型 T、方法 M 的示例测试命名如下：

```go
func Example() { ... } // 整个包的示例测试
func ExampleF() { ... } // 函数 F 的示例测试
func ExampleT() { ... } // 类型 T 的示例测试
func ExampleT_M() { ... } // 类型 T 的 M 方法的示例测试
```

一个包、函数、类型、方法如果存在多个示例测试，可以通过在名称后面附加一个不同的后缀来命名示例测试函数，后缀必须以小写字母开头，如下：

```go
func Example_suffix() { ... }
func ExampleF_suffix() { ... }
func ExampleT_suffix() { ... }
func ExampleT_M_suffix() { ... }
```

以下是一个为 `Abs` 函数编写的示例测试：

```go
func ExampleAbs() {
	fmt.Println(Abs(-1))
	fmt.Println(Abs(2))
	// Output:
	// 1
	// 2
}
```

示例测试函数末尾需要使用 `// Output:` 注释，来标记被测试函数的标准输出内容。

这里分别使用 `fmt.Println(Abs(-1))`、`fmt.Println(Abs(2))` 调用了两次 `Abs` 函数，所以会得到两个输出。

示例测试会拦截测试过程中的标准输出，并与 `// Output:` 注释之后的内容做对比，如果相等，则测试通过。

`go test` 默认情况下，不会执行示例测试，可以通过 `-run` 指定示例测试函数：

```bash
$ go test -v -run "ExampleAbs$"           
=== RUN   ExampleAbs
--- PASS: ExampleAbs (0.00s)
PASS
ok      github.com/jianghushinian/blog-go-example/test/getting-started/abs      2.050s
```

我们还可以使用 `// Unordered Output:` 注释，来标记被测试函数的标准输出内容。

这将忽略被测试函数的输出顺序：

```go
func ExampleAbs_unordered() {
	fmt.Println(Abs(2))
	fmt.Println(Abs(-1))
	// Unordered Output:
	// 1
	// 2
}
```

以上这个示例测试函数中，无论是先调用 `Abs(2)` 还是先调用 `Abs(-1)`，测试函数都能通过。

没有输出注释 `// Output:` 或 `// Unordered Output:`的示例测试函数只会被编译，但不会执行。

此外，示例测试还有一个非常有用功能，它能被 `godoc` 或 `pkgsite` 工具所识别，将示例函数的代码提取后作为被测试函数文档的一部分。

> 注意：旧版本的 Go 自带了 `godoc` 工具，能够在本地启动一个 Web 服务器，对本地安装的 Go 包提供文档服务，不过现在官方已经不维护 `godoc` 了，所以不再推荐使用。Go 1.15 以后虽然集成了 `go doc` 工具，但是无法启动 Web 服务，比较适合命令行中查看 Go 包的文档。现在，Go 官方比较推荐使用的工具是 `pkgsite`，能够启动 Web 服务，并且它与 Go 在线文档站点长得一样。

这里以 `pkgsite` 工具为例展示下示例测试函数生成的文档效果。

首先，安装 `pkgsite`：

```bash
$ go install golang.org/x/pkgsite/cmd/pkgsite@latest
```

然后，在 `abs.go` 目录下执行 `pkgsite` 即可启动文档服务：

```bash
$ pkgsite
```

现在访问 `http://localhost:8080` 即可进入文档服务首页：

![doc](go-doc.png)

点击模块名 `github.com/jianghushinian/blog-go-example/test/getting-started` 即可找到 `Abs` 函数位置，在 `Abs` 函数下方，标题 `Example`、`Example (Unordered)` 下就是通过示例测试生成的示例文档：

![example](example.png)

> 注意：这里需要额外提及的一点是，我们查看文档的本地包模块名称（`module`）应该带 `.`，也就是一般使用域名作为包名的一部分，否则启动 `pkgsite` 后将会报错，无法查看本地包的文档，具体原因可以查看这个 [issue](https://github.com/golang/go/issues/32819)。

### 模糊测试

模糊测试在 Go 1.18 中被引入，模糊测试（`fuzz testing`）又叫随机测试，是一种基于随机输入的自动化测试技术。

模糊测试比较适合用于发现处理用户输入的代码中存在的问题。

关于模糊测试的编写方式，有一张图广泛流传：

![fuzzing](fuzzing.png)

模糊测试同样需要放在 `_test.go` 文件中，并且以 `Fuzz` 开头，参数为 `*testing.F`。

上图中，`f.Add(5, "hello")` 是在为模糊测试提供初始的种子语料，其实就是被测试函数接收的合法参数，后续的模糊测试过程中，会根据这个种子语料，生成更多的模糊测试参数。这有点类似我们生成随机数时需要传递一个随机种子。虽然调用 `f.Add` 方法不是必须的，但提供合法的种子语料有利于更早发现被测试函数的问题。

`f.Fuzz` 是模糊测试的主体逻辑，它接收一个函数，函数的第一个参数为 `*testing.T`，之后是被测函数接收的参数，称为 `Fuzzing arguments`。

`Fuzzing arguments` 参数是 `testing` 框架随机生成的，所以叫随机测试，这些随机生成的参数将依次传递给 `Foo` 函数。

调用 `Foo` 函数和判断测试结果是否正确的代码，就跟我们编写的普通单元测试一样了。

可以发现，模糊测试相较于单元测试，多了一个自动生成测试参数的过程。

不过，`Fuzzing arguments` 支持的参数类型有限，仅支持如下几种类型：

- string, []byte
- int, int8, int16, int32/rune, int64
- uint, uint8/byte, uint16, uint32, uint64
- float32, float64
- bool

此外，编写模糊测试时，`Fuzz target` 不要依赖全局状态，因为模糊测试会并行执行。

为了演示如何编写模糊测试，我编写了一个 `Hello` 函数：

```go
package hello

import "errors"

var (
	ErrEmptyName   = errors.New("empty name")
	ErrTooLongName = errors.New("too long name")
)

func Hello(name string) (string, error) {
	if name == "" {
		return "", ErrEmptyName
	}
	if len(name) > 10 {
		return "", ErrTooLongName
	}
	return "Hello " + name, nil
}
```

将 `Hello` 函数放在 `hello.go` 文件中。

在 `Hello` 函数内部，对 `name` 参数进行了校验，不能为空，且长度不能超过 10。

为 `Hello` 函数编写模糊测试代码如下：

```go
func FuzzHello(f *testing.F) {
	f.Add("Foo")
	f.Fuzz(func(t *testing.T, name string) {
		_, err := Hello(name)
		if err != nil {
			if errors.Is(err, ErrEmptyName) || errors.Is(err, ErrTooLongName) {
				return
			}
			t.Errorf("unexpected error: %s, name: %s", err, name)
		}
	})
}
```

模糊测试代码放在 `hello_fuzz_test.go` 中，包名同样为 `hello`。

在 `f.Fuzz` 中调用了 `Hello` 函数，并判断返回的 `err` 是否符合预期，如果不符合预期，则表示测试失败。

`go test` 命令默认情况下同样不会执行模糊测试，我们需要指定 `-fuzz` 参数：

```bash
$ go test -fuzz="FuzzHello"
fuzz: elapsed: 0s, gathering baseline coverage: 0/1 completed
fuzz: elapsed: 0s, gathering baseline coverage: 1/1 completed, now fuzzing with 8 workers
fuzz: elapsed: 3s, execs: 521335 (173703/sec), new interesting: 2 (total: 3)
fuzz: elapsed: 6s, execs: 947014 (141945/sec), new interesting: 2 (total: 3)
fuzz: elapsed: 9s, execs: 1391822 (148228/sec), new interesting: 2 (total: 3)
fuzz: elapsed: 12s, execs: 1838008 (148764/sec), new interesting: 2 (total: 3)
fuzz: elapsed: 15s, execs: 2266978 (143002/sec), new interesting: 2 (total: 3)
^Cfuzz: elapsed: 15s, execs: 2308214 (139431/sec), new interesting: 2 (total: 3)
PASS
ok      github.com/jianghushinian/blog-go-example/test/getting-started/hello    17.131s
```

以上测试执行过程中，我使用 `^C` 终止了测试。模糊测试默认情况下会一直执行下去，直至遇到 crash 终止。

通过以上示例，我们可以发现，模糊测试之所以强大，就是因为其会一直执行，不断生成测试参数，以覆盖更多的情况和边界条件。

也正因为如此，模糊测试通常不建议在 CI 中执行。

不过，我们可以使用 `-fuzztime` 限制模糊测试执行的时间：

```bash
go test -fuzz="FuzzHello" -fuzztime 10s
fuzz: elapsed: 0s, gathering baseline coverage: 0/3 completed
fuzz: elapsed: 0s, gathering baseline coverage: 3/3 completed, now fuzzing with 8 workers
fuzz: elapsed: 3s, execs: 470505 (156828/sec), new interesting: 0 (total: 3)
fuzz: elapsed: 6s, execs: 948821 (159392/sec), new interesting: 0 (total: 3)
fuzz: elapsed: 9s, execs: 1423720 (158326/sec), new interesting: 0 (total: 3)
fuzz: elapsed: 10s, execs: 1573524 (139348/sec), new interesting: 0 (total: 3)
PASS
ok      github.com/jianghushinian/blog-go-example/test/getting-started/hello    11.820s
```

这次，我没有按 `^C` 键终止测试，而是 10 秒过后，模糊测试自动终止。

现在，我们修改下 `Hello` 函数，使其返回一个未知的错误：

```go
func Hello(name string) (string, error) {
	if name == "" {
		return "", ErrEmptyName
	}
	if len(name) > 10 {
		return "", ErrTooLongName
	}
	if name == "Bob" {
		return "", errors.New("not allowed")
	}
	return "Hello " + name, nil
}
```

当 `name` 值为 `Bob` 时，`Hello` 函数将返回一个未知错误，模拟 BUG 场景。

再次使用 `go test` 命令执行模糊测试：

```bash
$ go test -fuzz="FuzzHello"              
fuzz: elapsed: 0s, gathering baseline coverage: 0/3 completed
fuzz: elapsed: 0s, gathering baseline coverage: 3/3 completed, now fuzzing with 8 workers
fuzz: minimizing 30-byte failing input file
fuzz: elapsed: 1s, minimizing
--- FAIL: FuzzHello (1.17s)
    --- FAIL: FuzzHello (0.00s)
        hello_fuzz_test.go:18: unexpected error: not allowed, name: Bob
    
    Failing input written to testdata/fuzz/FuzzHello/19f92ff5a07664a0
    To re-run:
    go test -run=FuzzHello/19f92ff5a07664a0
FAIL
exit status 1
FAIL    github.com/jianghushinian/blog-go-example/test/getting-started/hello    1.626s
```

可以发现，这次模糊测试失败了。

根据测试结果，是在执行 `FuzzHello/19f92ff5a07664a0` 时失败的，`19f92ff5a07664a0` 是模糊测试生成的文件，位于 `testdata/fuzz/FuzzHello/19f92ff5a07664a0`。

使用 `tree` 命令查看 `19f92ff5a07664a0` 位置：

```bash
 tree hello
hello
├── hello.go
├── hello_fuzz_test.go
└── testdata
    └── fuzz
        └── FuzzHello
            └── 19f92ff5a07664a0
```

`testdata` 目录及目录下所有内容都是模糊测试自动生成的。

`19f92ff5a07664a0` 文件内容如下：

```txt
go test fuzz v1
string("Bob")
```

文件第一行 `go test fuzz v1` 是模糊测试要求的文件头，用于标识这是一个种子语料文件，并且使用的编解码器的版本为 `v1`。

第二行就是种子语料，是一个 Go 代码片段，即 `string` 类型的 `Bob` 参数。正是这个参数，引发了错误。

至此，我们使用模糊测试发现了 `Hello` 函数中隐藏的 BUG，这在黑盒测试中尤其有效，我们无需查看 `Hello` 函数内部代码，为每个边界条件编写测试用例，模糊测试会自动生成大量的随机参数，检测程序的异常。

### 测试覆盖率

`go test` 命令支持使用 `-cover` 标志查看测试覆盖率：

```bash
$ go test -cover ./... 
?       github.com/jianghushinian/blog-go-example/test/getting-started  [no test files]
ok      github.com/jianghushinian/blog-go-example/test/getting-started/abs      2.334s  coverage: 100.0% of statements
ok      github.com/jianghushinian/blog-go-example/test/getting-started/animal   1.924s  coverage: 100.0% of statements
ok      github.com/jianghushinian/blog-go-example/test/getting-started/hello    2.738s  coverage: 60.0% of statements
ok      github.com/jianghushinian/blog-go-example/test/getting-started/test/abs 3.136s  coverage: [no statements]
```

> 注意：根据我的实际测试结果来看，测试覆盖率默认仅包含单元测试、示例测试和模糊测试（模糊测试仅执行 `f.Add` 添加的种子参数测试），基准测试并不会被统计。要想将基准测试纳入覆盖率统计，需要增加 `-bench` 参数。你可以增加 `-v` 函数查看更详细信息。

此外，`go test` 命令还支持使用 `-coverprofile` 参数生成覆盖率 `profile` 文件：

```bash
$ go test -coverprofile=coverage.out ./...
?       github.com/jianghushinian/blog-go-example/test/getting-started  [no test files]
ok      github.com/jianghushinian/blog-go-example/test/getting-started/abs      3.101s  coverage: 100.0% of statements
ok      github.com/jianghushinian/blog-go-example/test/getting-started/animal   3.495s  coverage: 100.0% of statements
ok      github.com/jianghushinian/blog-go-example/test/getting-started/hello    3.910s  coverage: 60.0% of statements
ok      github.com/jianghushinian/blog-go-example/test/getting-started/test/abs 4.312s  coverage: [no statements]
```

命令执行后，将在当前目录生成一个 `coverage.out` 文件，内容如下：

```bash
cat coverage.out   
mode: set
github.com/jianghushinian/blog-go-example/test/getting-started/abs/abs.go:5.29,7.2 1 1
github.com/jianghushinian/blog-go-example/test/getting-started/animal/animal.go:7.32,8.21 1 1
github.com/jianghushinian/blog-go-example/test/getting-started/animal/animal.go:8.21,10.3 1 1
github.com/jianghushinian/blog-go-example/test/getting-started/animal/animal.go:11.2,11.21 1 1
github.com/jianghushinian/blog-go-example/test/getting-started/animal/animal.go:11.21,13.3 1 1
github.com/jianghushinian/blog-go-example/test/getting-started/animal/animal.go:14.2,14.17 1 1
github.com/jianghushinian/blog-go-example/test/getting-started/hello/hello.go:10.41,11.16 1 1
github.com/jianghushinian/blog-go-example/test/getting-started/hello/hello.go:11.16,13.3 1 0
github.com/jianghushinian/blog-go-example/test/getting-started/hello/hello.go:14.2,14.20 1 1
github.com/jianghushinian/blog-go-example/test/getting-started/hello/hello.go:14.20,16.3 1 0
github.com/jianghushinian/blog-go-example/test/getting-started/hello/hello.go:17.2,17.29 1 1
```

有了 `coverage.out` 文件，我们可以直接使用 `go tool cover` 命令查看测试覆盖率：

```bash
$ go tool cover -func=coverage.out                       
github.com/jianghushinian/blog-go-example/test/getting-started/abs/abs.go:5:            Abs             100.0%
github.com/jianghushinian/blog-go-example/test/getting-started/animal/animal.go:7:      shout           100.0%
github.com/jianghushinian/blog-go-example/test/getting-started/hello/hello.go:10:       Hello           60.0%
total:                                                                                  (statements)    81.8%
```

以上方式，实现了在命令行查看程序测试覆盖率。

我们还可以通过 `go tool cover` 命令以可视化的方式查看测试覆盖率：

```bash
$ go tool cover -html=coverage.out -o=coverage.html
```

执行命令后，会在当前目录下生成 `coverage.html` 文件，使用浏览器打开内容如下：

![coverage](coverage.png)

在页面顶部左侧，可以切换查看不同的测试文件和对应测试覆盖率。

灰色代码表示未被跟踪 `not tracked`。

红色部分表示未被测试的代码 `not covered`。

绿色部分表示已经被测试覆盖的代码 `covered`。

这样，我们就可以更加直观的查看和分析代码测试覆盖率了。

### 总结

本文向大家介绍了 Go 中编写各种测试代码的方式。

Go 支持单元测试、基准测试、示例测试以及模糊测试四种测试方法。

单元测试是我们最常使用的测试方法，如果被测代码需要编写多个测试用例，可以使用表格测试。

基准测试能够测量程序的性能指标，默认情况下 `go test` 不会执行基准测试，需要指定 `-bench regexp` 参数才可以执行。

示例测试可以测试程序的标准输出内容，并且能够配合 `pkgsite` 工具，在查看本地包文档时作为被测函数文档的一部分。

模糊测试是 Go 1.18 版本引入的，是一种基于随机输入的自动化测试技术，非常强大，适合用于发现处理用户输入的代码中存在的问题。

根据测试代码与被测代码是否在同一个包中，测试又可以分为白盒测试和黑盒测试，我们应该尽量编写白盒测试。

可以使用 `go test -cover` 查看测试覆盖率，我们可以将测试覆盖率基线集成到 CI 中，来保证单元测试覆盖率。

`go test` 命令支持的更多参数可以通过 `go help testflag` 命令查看。

由于篇幅所限，本文仅算做是 Go 单元测试的基础入门，更多单元测试在实战场景中的应用，我会在后续文章中进行讲解，敬请期待。

本文完整代码示例我放在了 [GitHub](https://github.com/jianghushinian/blog-go-example/tree/main/test/getting-started) 上，欢迎点击查看。

希望此文能对你有所帮助。

**联系我**

- 微信：jianghushinian
- 邮箱：[jianghushinian007@outlook.com](mailto:jianghushinian007@outlook.com)
- 博客地址：https://jianghushinian.cn/

**参考**

- Go testing 文档：https://pkg.go.dev/testing@go1.20.1
- Go Fuzzing：https://go.dev/security/fuzz/
- Go 1.18新特性前瞻：原生支持Fuzzing测试：https://tonybai.com/2021/12/01/first-class-fuzzing-in-go-1-18/
