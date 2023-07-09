---
title: 在 Go 中如何实现类似 Python 中的 with 上下文管理器
date: 2023-06-23 08:53:10
tags:
- Go
categories:
- Go
---

熟悉 Python 的同学应该知道 Python 中的上下文管理器非常好用，在对数据库进行读写、访问文件等操作时，上下文管理器能够确保资源在使用后得到释放。在 Go 中是否也能实现上下文管理器呢？这便是本文所要探讨的话题。

<!-- more -->

### Python 上下文管理器

以操作文件为例，为了保证操作文件完成后资源能被正确关闭，在 Python 中我们可以编写出如下代码：

```python
try:
    f = open('foo.txt', 'r')
    print(f.readlines())
finally:
    f.close()
```

不过这种写法显然不够 Pythonic，Python 在语法层面提供了 `with` 语句实现上下文管理，用法如下：

```python
with open('foo.txt', 'r') as f:
    print(f.readlines())
```

这段使用 `with` 语句实现的代码，才更符合 Python 哲学。

如果你对 Python `with` 语法不熟悉，可以参阅我的文章[《Python 上下文管理器实现》](https://jianghushinian.cn/2019/09/18/Python-上下文管理器实现/)。

### Go 中资源释放问题

我们知道，在 Go 语言中访问数据库、文件等资源时，可以使用 `defer` 语句完成资源释放操作。

如下定义一个 `ReadFile` 函数用来读取文件：

```go
func ReadFile(paths []string) error {
	for _, path := range paths {
		file, err := os.Open(path)
		if err != nil {
			return err
		}
		defer file.Close()

		content, err := io.ReadAll(file)
		if err != nil {
			return err
		}
		fmt.Printf("%s content: %s\n", file.Name(), content)
	}
	return nil
}
```

这个函数使用循环遍历传进来的文件路径列表，依次打开文件并输出文件内容。

为了保证即使在遇到错误时，资源也能够被释放，我们往往会使用 `defer file.Close()` 来关闭文件。

不过，这段代码其实是存在问题的，我们知道 `defer` 的调用实际上并不会立即执行，而是等到函数退出时才会执行。

所以，代码中的 `defer` 调用并不会在本轮循环中处理完当前文件时被执行，而是直到所有循环执行完成，函数退出时才会执行。

我们可以对以上示例稍作修改，来验证下这个问题：

```go
func ReadFile(paths []string) error {
	for _, path := range paths {
		file, err := os.Open(path)
		if err != nil {
			return err
		}
		defer func() {
			file.Close()
			fmt.Printf("close %s\n", file.Name())
		}()

		content, err := io.ReadAll(file)
		if err != nil {
			return err
		}
		fmt.Printf("%s content: %s\n", file.Name(), content)
	}
	return nil
}
```

我们将原来的 `defer` 语句改成：

```go
defer func() {
    file.Close()
    fmt.Printf("close %s\n", file.Name())
}()
```

以此来显示 `defer` 调用时机。

针对以上示例，我们使用如下代码来调用：

```go
func main() {
	err := ReadFile([]string{"foo.txt", "bar.txt"})
	fmt.Printf("ReadFile err: %v\n", err)
}
```

> 注意：`foo.txt`、`bar.txt` 两个文件我已经提前准备好了，`foo.txt` 文件内容为 `foo`，`bar.txt` 文件内容为 `bar`。

执行以上示例，得到如下输出：

```bash
$ go run main.go
foo.txt content: foo
bar.txt content: bar
close bar.txt
close foo.txt
ReadFile err: <nil>
```

根据输出内容可以验证，`defer` 语句的调用，的确在 `for` 循环退出以后才开始执行。

如果打开资源过多，而没有及时关闭，势必会造成资源的浪费，甚至因此而意外终止程序。

所以切记，不要在循环中使用 `defer`。

我们可以使用匿名函数来解决这个问题：

```go
func ReadFile(paths []string) error {
	for _, path := range paths {
		file, err := os.Open(path)
		if err != nil {
			return err
		}
		err = func() error {
			defer func() {
				file.Close()
				fmt.Printf("close %s\n", file.Name())
			}()

			content, err := io.ReadAll(file)
			if err != nil {
				return err
			}
			fmt.Printf("%s content: %s\n", file.Name(), content)
			return nil
		}()
		if err != nil {
			return err
		}
	}
	return nil
}
```

现在，将 `defer` 语句放入到一个立即执行的匿名函数中，就可以解决问题了。

执行以上示例，得到如下输出：

```go
$ go run main.go
foo.txt content: foo
close foo.txt
bar.txt content: bar
close bar.txt
ReadFile err: <nil>
```

可以发现，现在 `defer` 语句不再是等到 `for` 循环退出才会执行，而是在匿名函数退出时即可执行。

这样，就达到了在本轮循环中尽早释放不再使用的文件资源的目的。

此外，为了代码的可读性，我们可以将匿名函数提取出来，单独封装一个函数：

```go
func ReadFile(paths []string) error {
	for _, path := range paths {
		file, err := os.Open(path)
		if err != nil {
			return err
		}
		err = processFile(file)
		if err != nil {
			return err
		}
	}
	return nil
}

func processFile(file *os.File) error {
	defer func() {
		file.Close()
		fmt.Printf("close %s\n", file.Name())
	}()

	content, err := io.ReadAll(file)
	if err != nil {
		return err
	}
	fmt.Printf("%s content: %s\n", file.Name(), content)
	return nil
}
```

`processFile` 函数专门用来处理打开的文件，`ReadFile` 函数可读性也得到了提高。

执行以上示例，得到如下输出：

```bash
go run main.go
foo.txt content: foo
close foo.txt
bar.txt content: bar
close bar.txt
ReadFile err: <nil>
```

这个输出符合预期。

以上我们介绍了两种方式，能够解决 `defer` 语句延迟调用的问题。

### 在 Go 中实现上下文管理器

最近为了写[《Go 语言中 database/sql 是如何设计的》](https://jianghushinian.cn/2023/06/21/how-is-database-sql-designed-in-go/)一文，我阅读了下 `database/sql` 的源码。在这个过程中，`*sql.DB.queryDC` 方法中一小段代码激起了我的兴趣：

```go
func (db *DB) queryDC(ctx, txctx context.Context, dc *driverConn, releaseConn func(error), query string, args []any) (*Rows, error) {
	...
	if ok {
		var nvdargs []driver.NamedValue
		var rowsi driver.Rows
		var err error
		withLock(dc, func() {
			nvdargs, err = driverArgsConnLocked(dc.ci, nil, args)
			if err != nil {
				return
			}
			rowsi, err = ctxDriverQuery(ctx, queryerCtx, queryer, query, nvdargs)
		})
		...
	}
	...
}
```

在 `*sql.DB.queryDC` 方法中有一个 `withLock` 函数的调用，`withLock` 函数定义如下：

```go
func withLock(lk sync.Locker, fn func()) {
	lk.Lock()
	defer lk.Unlock()
	fn()
}
```

当看到 `withLock` 函数定义时，我瞬间就想到了 Python 中的 `with` 上下文管理器。

`withLock` 接收一个 `sync.Locker` 接口，定义如下：

```go
type Locker interface {
	Lock()
	Unlock()
}
```

它只有两个方法，加锁和释放锁。

`withLock` 能够用于所有实现 `sync.Locker` 接口的对象，在执行 `fn()` 前加锁，执行之后释放锁。

这与 Python 的上下文管理器功能如出一辙，就是这么一个只有三行的小函数，实现却相当精妙，真可谓短小精悍。

于是，参考 `withLock` 函数实现，解决 `for` 循环中`defer` 语句延迟调用的问题，就有了第三种解法。

我们可以模仿 `withLock` 实现一个 `WithClose` 函数：

```go
func WithClose(closer io.Closer, fn func()) {
	defer func() {
		closer.Close()
		fmt.Printf("close %s\n", closer.(*os.File).Name())
	}()
	fn()
}
```

`WithClose` 接收一个 `io.Closer` 接口，定义如下：

```go
type Closer interface {
	Close() error
}
```

我们可以在执行 `fn()` 函数之前，使用 `defer` 语句来调用 `io.Closer` 的 `Close` 方法释放资源。

现在，我们可以在 `ReadFile` 函数中使用这个小函数了：

```go
func ReadFile(paths []string) error {
	for _, path := range paths {
		file, err := os.Open(path)
		if err != nil {
			return err
		}
		WithClose(file, func() {
			var content []byte
			content, err = io.ReadAll(file)
			if err != nil {
				return
			}
			fmt.Printf("%s content: %s\n", file.Name(), content)
		})
		if err != nil {
			return err
		}
	}
	return nil
}
```

这个用法同 `*sql.DB.queryDC` 中调用 `withLock` 函数一样，并且因为闭包的存在，我们可以拿到 `WithClose` 内部执行的 `fn()` 函数所产生的错误对象。

执行以上示例，得到如下输出：

```bash
$ go run main.go
foo.txt content: foo
close foo.txt
bar.txt content: bar
close bar.txt
ReadFile err: <nil>
```

这个输出依然符合预期。

我们可以测试下遇到错误的情况，修改 `main` 函数，调用 `ReadFile` 时最后传入一个不存在的文件 `baz.txt`：

```go
func main() {
	err := ReadFile([]string{"foo.txt", "bar.txt", "baz.txt"})
	fmt.Printf("ReadFile err: %v\n", err)
}
```

执行以上示例，得到如下输出：

```bash
$ go run main.go
foo.txt content: foo
close foo.txt
bar.txt content: bar
close bar.txt
ReadFile err: open baz.txt: no such file or directory
```

遇到错误能够被正常捕获。

现在，我们就在 Go 中实现类了似 Python 中的 `with` 上下文管理器，为解决 `for` 循环中`defer` 语句延迟调用的问题提供了新思路。

### 总结

本文灵感来自于 `database/sql` 源码中的一小段代码，为大家讲解了如何在 Go 中实现类似 Python 中的 `with` 上下文管理器。

切记，不要在循环中使用 `defer`。为了解决这个问题，我们可以使用匿名函数、函数封装以及 `WithClose` 三种方案。

希望此文能对你有所帮助。

### P.S.

`database/sql` 源码中的这一小段代码，找回了我在开始用 Go 作为主力语言后，很久没有在编程语言语法层面上体会过快感。相较于我最近写的几篇长篇大论型文章，本文显得微不足道，但我还是很乐于为这一小段代码写一篇文章分享出来，毕竟这久违的感觉又回来了。

从把 Go 作为主力编程语言开始，写代码的思路都是“平铺直叙”，很少思考怎么写出更加优雅且有趣的代码。尽管我也分享过几篇 Go 编程模式的文章，但相较于用 Python 作为主力编程语言时，还是少了很多“花哨”的小技巧在里面，更多的是遵循套路的样板代码。

尽管 Go 语言的哲学更适合工程化，但 Go 代码写多了，有时不免会略感乏味，怀念 Python 的灵活。我无意于讨论哪种编程语言的好坏，只是，愿在编程的道路上，你我都能找到属于自己的乐趣所在。

**联系我**

- 微信：jianghushinian
- 邮箱：[jianghushinian007@outlook.com](mailto:jianghushinian007@outlook.com)
- 博客地址：https://jianghushinian.cn/

**参考**

- database/sql 源码：https://github.com/golang/go/tree/go1.20.1/src/database/sql
- Go 语言中 database/sql 是如何设计的：https://jianghushinian.cn/2023/06/21/how-is-database-sql-designed-in-go/
- Python 上下文管理器实现：https://jianghushinian.cn/2019/09/18/Python-上下文管理器实现/
