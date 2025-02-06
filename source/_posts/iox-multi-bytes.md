---
title: '在 Go 中如何将 [][]byte 转为 io.Reader ？'
date: 2025-02-07 00:30:31
tags:
- Go
categories:
- Go
---

起因：在春节前的某一天，我在 [ekit](https://github.com/ecodeclub/ekit) 项目的交流群里看到大明老师发了这样一条消息：

> 各位大佬，问个小问题，有咩有谁用过 [][]byte 转为 io.Reader 的东西？我以前搞过一次，但是我忘了是我手搓了一个实现，还是用的开源的，还是SDK 自带的
>

<!-- more -->

并且大明老师还为此开了一个 [issue](https://github.com/ecodeclub/ekit/issues/271)。

看到这条消息，我想起了我在对 Go 还不太熟悉时，曾写过一个 `io.MultiReader` 的实现（当时写完了我才知道原来 Go 中自带了 `io.MultiReader`），想必应该有相似之处，于是就尝试写了一个出来。

不过当我写完时发现已经有人提交了代码，于是我就没把它当回事，也没有写测试代码进行测试，就放一边了。春节假期闲来无事，我忽然想起来这件事，就看了下对应的 [pr](https://github.com/ecodeclub/ekit/pull/272)，发现提交 pr 的作者和我的实现思路不太一样。不过，虽然这个功能很小，既然我也实现了，就补齐下单元测试，发出来供参考，顺便写（水）一篇文章 :)。

### 思路设计

首先我设计了如下结构体：

```go
type MultiBytes struct {
	data  [][]byte // 存储数据的嵌套切片
	index int      // 当前读/写到的外层切片索引，data[index]
	pos   int      // 当前读/写到的切片所处理到的位置下标，data[index][pos]
}
```

有了这个结构体，那么就可以设计 `MultiBytes` 的整体实现思路了。

首先，我们需要一个构造函数 `NewMultiBytes` 来创建 `MultiBytes` 对象。其次，则要实现 `io.Reader` 接口。最后，我们也可以顺便实现一下 `io.Write` 接口。

`MultiBytes` 支持的函数和方法设计如下：

```go
type MultiBytes
    func NewMultiBytes(data [][]byte) *MultiBytes
    func (b *MultiBytes) Read(p []byte) (int, error)
    func (b *MultiBytes) Write(p []byte) (int, error)
```

基于此，我为每个方法画了一个流程图，你可以参考下：

![](multi-bytes.png)

流程图中包含了每个方法内部的主体逻辑。

### 代码实现

既然有了结构体和方法签名，那么就可以依次实现所有方法了。

首先是构造函数 `NewMultiBytes` 的实现：

> [https://github.com/jianghushinian/blog-go-example/blob/main/iox/multi_bytes.go](https://github.com/jianghushinian/blog-go-example/blob/main/iox/multi_bytes.go)
>

```go
// NewMultiBytes 构造一个 MultiBytes
func NewMultiBytes(data [][]byte) *MultiBytes {
	return &MultiBytes{
		data: data,
	}
}
```

这没什么好说的，就是根据给定的 `data` 初始化了一个 `*MultiBytes` 对象，`index` 和 `pod` 都为默认值 0。

接着是 `Read` 方法的实现：

```go
// Read 实现 io.Reader 接口，从 data 中读取数据到 p
func (b *MultiBytes) Read(p []byte) (int, error) {
	// 如果 p 是空的，直接返回
	if len(p) == 0 {
		return 0, nil
	}

	// 所有数据都已读完
	if b.index >= len(b.data) {
		return 0, io.EOF
	}

	n := 0 // 记录已读取的字节数

	for n < len(p) {
		// 如果当前切片已经读完，则切换到下一个切片
		if b.pos >= len(b.data[b.index]) {
			b.index++
			b.pos = 0
			// 如果所有切片都已读完，退出循环
			if b.index >= len(b.data) {
				break
			}
		}

		// 从当前切片读取数据
		bytes := b.data[b.index]
		cnt := copy(p[n:], bytes[b.pos:])
		b.pos += cnt
		n += cnt
	}

	// 未读取到数据且已经读到结尾
	if n == 0 {
		return 0, io.EOF
	}

	return n, nil
}
```

`Read` 方法就是按照流程图中的整体脉络实现的。需要强调的一点是，程序最后还有一个 `if n == 0` 的判断，如果成立，返回 `io.EOF`。这是为了处理 `data` 中嵌套的内部切片为空的情况，比如当 `data` 值为 `[][]byte{[]byte{}}` 这种情况时，程序就会走到这个分支。

然后是 `Write` 方法的实现：

```go
// Write 实现 io.Writer 接口，将数据追加到 data 中
func (b *MultiBytes) Write(p []byte) (int, error) {
	// 如果 p 是空的，直接返回
	if len(p) == 0 {
		return 0, nil
	}

	// 创建副本以避免外部修改影响数据
	clone := make([]byte, len(p))
	copy(clone, p)
	b.data = append(b.data, clone)
	return len(p), nil
}
```

值得注意的是，在 `Write` 方法实现中，对 `p` 进行了拷贝，生成新的副本，目的是防止用户在调用 `Write(p)` 以后，随意修改 `p` 的值而影响 `MultiBytes` 对象内部的 `data`。

最后，如果你不嫌麻烦，还可以增加如下两行代码，以检查 `MultiBytes` 是否实现了  `io.Reader` 和 `io.Write` 接口：

```go
var _ io.Reader = (*MultiBytes)(nil)
var _ io.Writer = (*MultiBytes)(nil)
```

至此，能够将 `[][]byte` 转为 `io.Reader` 的 `MultiBytes` 实现完成。

我们可以简单测试一下效果。

示例代码：

> [https://github.com/jianghushinian/blog-go-example/blob/main/iox/examples/multi_bytes.go](https://github.com/jianghushinian/blog-go-example/blob/main/iox/examples/multi_bytes.go)
>

```go
package main

import (
	"fmt"

	"github.com/jianghushinian/blog-go-example/iox"
)

func main() {
	mb := iox.NewMultiBytes([][]byte{[]byte("Hello, World!\n")})
	_, _ = mb.Write([]byte("你好，世界！"))
	p := make([]byte, 32)
	_, _ = mb.Read(p)
	fmt.Println(string(p))
}
```

执行示例代码，得到输出如下：

```bash
$ go run examples/multi_bytes.go
Hello, World!
你好，世界！
```

### 总结

本文带大家实现了一个能够将 `[][]byte` 转为 `io.Reader` 的 `MultiBytes`，代码逻辑并不复杂，不过一些细节还是需要注意。

你还可以点击[这里](https://github.com/jianghushinian/blog-go-example/blob/main/iox/multi_bytes_test.go)查看更多的单元测试，如果你在使用过程中，发现任何 bug，欢迎交流。

本文示例源码我都放在了 [GitHub](https://github.com/jianghushinian/blog-go-example/blob/main/iox/multi_bytes.go) 中，欢迎点击查看。

希望此文能对你有所启发。

**延伸阅读**

+ ekit issue：https://github.com/ecodeclub/ekit/issues/271
+ 本文 GitHub 示例代码：https://github.com/jianghushinian/blog-go-example/blob/main/iox/multi_bytes.go
+ 本文 GitHub 测试代码：https://github.com/jianghushinian/blog-go-example/blob/main/iox/multi_bytes_test.go

**联系我**

+ 公众号：[Go编程世界](https://jianghushinian.cn/about)
+ 微信：jianghushinian
+ 邮箱：[jianghushinian007@outlook.com](mailto:jianghushinian007@outlook.com)
+ 博客：[https://jianghushinian.cn](https://jianghushinian.cn)
+ GitHub：[https://github.com/jianghushinian](https://github.com/jianghushinian)
