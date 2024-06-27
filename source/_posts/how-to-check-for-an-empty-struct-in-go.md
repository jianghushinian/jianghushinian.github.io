---
title: 在 Go 中如何检查结构体是否为空
date: 2024-06-27 09:17:58
tags:
- Go
- 翻译
categories:
- Go
---

本文翻译自：[How to Check for an Empty Struct in Go](https://freshman.tech/snippets/go/check-empty-struct/)。

本文概述了几种在 Go 中判断结构体是否为空的方法，适用于具有**可比较字段**和**不可比较字段**的结构体。Go 中的空结构体是指所有字段均设置为对应字段零值的结构体。

<!-- more -->

### 使用零值字面量进行检查

对于仅包含[可比较字段](https://go.dev/ref/spec#Comparison_operators)的结构体，只需要将结构体实例与其**零值字面量**进行比较：

```go
package main

import (
	"fmt"
)

type Person struct {
	name  string
	age   int
	email int
}

func main() {
	var p1 Person

	p2 := Person{
		name: "John",
		age:  45,
	}

	fmt.Println(p1 == Person{})
	fmt.Println(p2 == Person{})
}
```

```bash
true
false
```

确保在 `if` 语句中使用括号括住结构体字面量，以避免出现解析问题：

```go
if p1 == (Person{}) {

}
```

对于指向结构的指针，请确保在比较之前取消引用：

```go
p3 := &Person{}

if *p3 == (Person{}) {

}
```

### 使用 reflect.DeepEqual()

对于具有**不可比较字段（slices, maps, functions）**的结构，可以使用 `reflect.DeepEqual()` 进行比较：

```go
package main

import (
	"fmt"
	"reflect"
)

type Person struct {
	name            string
	age             int
	email           int
	favouriteColors []string // non-comparable field
}

func main() {
	var p1 Person

	p2 := Person{
		name:            "John",
		age:             45,
		favouriteColors: []string{"red", "green"},
	}

	fmt.Println(reflect.DeepEqual(p1, Person{}))
	fmt.Println(reflect.DeepEqual(p2, Person{}))
}
```

```bash
true
false
```

这个 `DeepEqual()` 方法实际上适用于任何结构比较，而不仅仅是检查结构体是否为空。

### 使用 reflect.Value.IsZero()

该方法在 `Go 1.13` 中引入，`reflect.Value.IsZero()` 提供了另一种检查空结构体的方法：

```go
package main

import (
	"fmt"
	"reflect"
)

type Person struct {
	name            string
	age             int
	email           int
	favouriteColors []string // non-comparable field
}

func main() {
	var p1 Person

	p2 := Person{
		name:            "John",
		age:             45,
		favouriteColors: []string{"red", "green"},
	}

	fmt.Println(reflect.ValueOf(p1).IsZero())
	fmt.Println(reflect.ValueOf(p2).IsZero())
}
```

```bash
true
false
```

### 总结

这些技巧提供了在 Go 语言中识别空结构体的可靠方法，适用于不同的字段类型和结构体特性。如果你有更多的建议，请在下面的评论区提出。

感谢阅读，编码愉快！

**延伸阅读**

- 原文地址: https://freshman.tech/snippets/go/check-empty-struct/
- Comparison operators: https://go.dev/ref/spec#Comparison_operators

**联系我**

- 公众号：[Go编程世界](https://jianghushinian.cn/about)
- 微信：jianghushinian
- 邮箱：[jianghushinian007@outlook.com](mailto:jianghushinian007@outlook.com)
- 博客：https://jianghushinian.cn
