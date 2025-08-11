---
title: '在 Go 中使用 dyno 包处理动态对象'
date: 2025-08-11 09:29:35
tags:
- Go
categories:
- Go
---

我在《[Go 语言中 YAML to JSON 踩坑笔记](https://mp.weixin.qq.com/s/rGKEgsDl0Wrl0m7W8i2soA)》一文中提到了使用 `dyno` 包来解决 `json.Marshal` 时遇到不支持的 `map[interface{}]interface{}` 类型报错的问题。本文就来通过源码的形式为大家详解一下 `dyno` 包的原理。

<!-- more -->

### dyno 包简介

首先，我们要搞清楚 `dyno` 包是用来干什么的：`dyno` 包的主要目标是方便我们操作动态对象，当我们在序列化/反序列化 JSON 或 YAML 文档时，通常选择使用 `map[string]interface{}` 或 `map[interface{}]interface{}` 类型来表示对象，使用  `[]interface{}` 来表示数组，而 `dyno` 包支持在任意嵌套深度和任意组合中处理这些混合类型的结构。

`dyno` 包实现的所有函数如下：

![dyno](dyno.jpg)

可以发现，`dyno` 包实现了对嵌套的 `map/slice` 类型对象的 CRUD 操作，并且可以通过 `ConvertMapI2MapS` 函数将 `map` 的 `interface{}` 类型的 `key` 转换成 `string` 类型。

接下来，我们将对 `dyno` 包的源码进行讲解。

### dyno 包源码解析

#### Get

我们先来从 `Get` 函数讲起，`Get` 函数用法如下：

```go
package main

import (
	"fmt"

	"github.com/icza/dyno"
)

func main() {
	y := map[string]interface{}{
		"object": map[interface{}]interface{}{
			"a": 1,
			"array": []interface{}{
				map[string]interface{}{
					"null_value": interface{}(nil),
				},
				map[string]interface{}{
					"boolean": true,
				},
				map[string]interface{}{
					"integer": 1,
				},
			},
			"key": "value",
			1:     2,
		},
	}

	// 按路径获取值
	// 混合使用字符串键（字典）和整型索引（切片）
	get, err := dyno.Get(y, "object", "array", 2, "integer")
	if err != nil {
		panic(err)
	}
	fmt.Printf("%T, %#v\n", get, get)
}
```

我们有一个嵌套的 `map` 类型对象 `y`，现在我们想拿到 `y["object"]["array"][2]["integer"]` 的值，如果手撸代码，这将是“灾难”（这一点相比于 Python 确实不够优雅）。

`dyno` 包 `Get` 函数就是来帮我们做这件事情的，只需要将 `key` 或索引依次传递给 `Get` 函数即可：

```go
get, err := dyno.Get(y, "object", "array", 2, "integer")
```

执行示例代码，得到输出如下：

```bash
$ go run main.go
int, 1
```

我们仅用了一行代码，就获取到了 `map` 中嵌套多层的对象的值。

而这背后的 `Get` 的函数到底是如何实现的呢？我们一起来看一下。

`dyno` 包 `Get` 函数源码如下：

```go
// Get 从动态结构（如 map[string]interface{} 或 []interface{}）中按路径获取值
// 如果 path 参数为空，则直接返回 v
func Get(v interface{}, path ...interface{}) (interface{}, error) {
	// 遍历路径参数 path
	for i, el := range path {
		switch node := v.(type) { // 断言 v 的类型，注意每一轮循环中这个 v 都是新的值
		// 情况 1：处理键为 string 的 map
		case map[string]interface{}:
			key, ok := el.(string) // 路径元素必须是 string
			if !ok {
				return nil, fmt.Errorf("expected string path element, got: %T (path element idx: %d)", el, i)
			}
			v, ok = node[key] // 从 map 中获取值
			if !ok {
				return nil, fmt.Errorf("missing key: %s (path element idx: %d)", key, i)
			}

		// 情况 2：处理键为任意类型的 map
		case map[interface{}]interface{}:
			var ok bool
			v, ok = node[el] // 直接使用路径元素作为键，来获取值
			if !ok {
				return nil, fmt.Errorf("missing key: %v (path element idx: %d)", el, i)
			}

		// 情况 3：处理切片
		case []interface{}:
			idx, ok := el.(int) // 路径元素必须是 int
			if !ok {
				return nil, fmt.Errorf("expected int path element, got: %T (path element idx: %d)", el, i)
			}
			if idx < 0 || idx >= len(node) { // 索引越界检查
				return nil, fmt.Errorf("index out of range: %d (path element idx: %d)", idx, i)
			}
			v = node[idx] // 从切片中获取值

		// 情况 4：不支持的类型
		default:
			return nil, fmt.Errorf("expected map or slice node, got: %T (path element idx: %d)", node, i)
		}
	}

	// 返回最终 v 的值
	return v, nil
}
```

这里源码整体结构还是比较清晰的，首先是一个 `for` 循环，用来遍历 `path`（即我们传进来的 `"object", "array", 2, "integer"` 这些参数）。接着在 `for` 循环内部，根据 `v`（即我们传进来的 `y` 对象）的类型，分 4 种情况来获取元素。

+ 如果 `v` 的类型为 `map[string]interface{}`，那么 `path` 参数就必须是 `string` 类型，这样才能在 `map` 中获取 `path` 这个 `key` 对应的值。
+ 如果 `v` 的类型为 `map[interface{}]interface{}`，那么可以直接使用路径元素作为 `key`，来从 `map` 中获取值。
+ 如果 `v` 的类型为 `[]interface{}`，那么 `path` 参数就必须是 `int` 类型，以此来从 `slice` 中获取值。
+ 最后，如果 `v` 的类型不满足上述三者之一，则为不支持的类型，直接返回错误。

`Get` 函数最终返回 `v` 的值。

`Get` 函数源码还是比较好理解的，而与 `Get` 函数近似的 [GetInt](https://godoc.org/github.com/icza/dyno#GetInt), [GetFloat64](https://pkg.go.dev/github.com/icza/dyno#GetFloat64)，[GetString](https://godoc.org/github.com/icza/dyno#GetString), [GetSlice](https://godoc.org/github.com/icza/dyno#GetSlice), [GetMapI](https://godoc.org/github.com/icza/dyno#GetMapI), [GetMapS](https://godoc.org/github.com/icza/dyno#GetMapS) 这几函数个，其实都是对 `Get` 函数的包装，我们以 `GetInt` 函数源码来举例，其他几个函数无需讲解你就都明白了。

`GetInt` 函数源码如下：

```go
// GetInt returns an int value denoted by the path.
//
// If path is empty or nil, v is returned as an int.
func GetInt(v interface{}, path ...interface{}) (int, error) {
	v, err := Get(v, path...)
	if err != nil {
		return 0, err
	}
	i, ok := v.(int)
	if !ok {
		return 0, fmt.Errorf("expected int value, got: %T", v)
	}
	return i, nil
}
```

根据源码可以发现，`GetInt` 内部调用了 `Get` 函数来获取值，只不过之后对值 `v` 进行了类型断言，判断其是否为 `Int` 类型，其他并无特殊功能。

所以我们只需要学习一个 `Get` 函数，然后记住其他所有 `Get<Type>` 函数都是基于 `Get` 函数来实现的即可。

此外，`Get<Type>` 类型的函数还有另外 3 个：[GetInteger](https://godoc.org/github.com/icza/dyno#GetInteger), [GetFloating](https://godoc.org/github.com/icza/dyno#GetFloating), [GetBoolean](https://godoc.org/github.com/icza/dyno#GetBoolean) 。

这 3 个函数源码实现也比较类似，我们以 `GetInteger` 为例为你讲解，其源码如下：

```go
// GetInteger returns an int64 value denoted by the path.
//
// This function accepts many different types and converts them to int64, namely:
//
//	-integer types (int, int8, int16, int32, int64, uint, uint8, uint16, uint32, uint64)
//	 (which implies the aliases byte and rune too)
//	-floating point types (float64, float32)
//	-string (fmt.Sscan() will be used for parsing)
//	-any type with an Int64() (int64, error) method (e.g. json.Number)
//
// If path is empty or nil, v is returned as an int64.
func GetInteger(v interface{}, path ...interface{}) (int64, error) {
	v, err := Get(v, path...)
	if err != nil {
		return 0, err
	}

	switch i := v.(type) {
	case int64:
		return i, nil
	case int:
		return int64(i), nil
	case int32:
		return int64(i), nil
	case int16:
		return int64(i), nil
	case int8:
		return int64(i), nil
	case uint:
		return int64(i), nil
	case uint64:
		return int64(i), nil
	case uint32:
		return int64(i), nil
	case uint16:
		return int64(i), nil
	case uint8:
		return int64(i), nil
	case float64:
		return int64(i), nil
	case float32:
		return int64(i), nil
	case string:
		var n int64
		_, err := fmt.Sscan(i, &n)
		return n, err
	case interface {
		Int64() (int64, error)
	}:
		return i.Int64()
	default:
		return 0, fmt.Errorf("expected some form of integer number, got: %T", v)
	}
}
```

`GetInteger` 内部依然调用了 `Get` 函数来获取值，而这一次，其内部会将所有数字类型全部转换成 `int64` 类型来返回。

看到这里，另外两个函数 [GetFloating](https://godoc.org/github.com/icza/dyno#GetFloating), [GetBoolean](https://godoc.org/github.com/icza/dyno#GetBoolean) 也就不必我过多讲解，都是同样的套路，你一看源码便知。

最后，`Get` 类函数还有一个特例，叫 `SGet`。这个函数专门用来从嵌套的 `map[string]interface{}` 结构中，通过纯字符串路径（不支持切片索引）获取值。

`SGet` 函数源码如下：

```go
// SGet 从嵌套的 map[string]interface{} 结构中，通过纯字符串路径（不支持切片索引）获取值
// 如果 path 参数为空，则直接返回 m
func SGet(m map[string]interface{}, path ...string) (interface{}, error) {
	if len(path) == 0 {
		return m, nil
	}

	lastIdx := len(path) - 1
	var value interface{}
	var ok bool

	// 遍历 path
	for i, key := range path {
		// 1. 检查当前键是否存在
		if value, ok = m[key]; !ok {
			return nil, fmt.Errorf("missing key: %s (path element idx: %d)", key, i)
		}
		// 2. 若是最后一个路径元素，直接推出循环
		if i == lastIdx {
			break
		}
		// 3. 检查中间节点是否为 map[string]interface{}
		m2, ok := value.(map[string]interface{})
		if !ok {
			return nil, fmt.Errorf("expected map with string keys node, got: %T (path element idx: %d)", value, i)
		}
		// 4. 更新当前节点，继续深入下一层
		m = m2
	}

	return value, nil
}
```

注意，这里函数签名与 `Get` 不同，它接收的值是 `map[string]interface{}` 类型，相应的 `path` 也必须是 `string` 类型。

`SGet` 函数仅处理单一的类型，所以源码实现上比 `Get` 函数简洁不少。

#### Set

接下来，我们再来看下 `Set` 函数是如何使用和实现的。

`Set` 函数用于通过路径在动态结构（嵌套的 map 或 slice）中修改指定位置元素的值。

使用示例如下：

```go
package main

import (
	"fmt"

	"github.com/icza/dyno"
)

func main() {

	m := map[string]interface{}{
		"user": map[string]interface{}{
			"name": "江湖十年",
			"address": map[string]interface{}{
				"city": "Beijing",
				"zip":  10115,
			},
		},
	}

	// 按路径设置值
	err := dyno.Set(m, "Hangzhou", "user", "address", "city")
	if err != nil {
		panic(err)
	}

	// 按路径获取值
	get, err := dyno.SGet(m, "user", "address", "city")
	if err != nil {
		panic(err)
	}
	fmt.Printf("%T, %#v\n", get, get)
}
```

执行示例代码，得到输出如下：

```bash
$ go run main.go
string, "Hangzhou"
```

`Set` 函数源码实现如下：

```go
// Set 通过路径在动态结构（嵌套的 map 或 slice）中修改指定位置元素的值
func Set(v interface{}, value interface{}, path ...interface{}) error {
	// 1. 路径非空校验
	if len(path) == 0 {
		return fmt.Errorf("path cannot be empty")
	}

	// 2. 分离末位元素
	i := len(path) - 1 // The last index
	if len(path) > 1 {
		var err error
		v, err = Get(v, path[:i]...) // 末位元素的值
		if err != nil {
			return err
		}
	}

	el := path[i] // 末位元素的键或索引

	// 3. 根据末位元素值的类型进行赋值
	switch node := v.(type) {
	case map[string]interface{}: // 情况 1：处理键为 string 的 map
		key, ok := el.(string)
		if !ok {
			return fmt.Errorf("expected string path element, got: %T (path element idx: %d)", el, i)
		}
		node[key] = value

	case map[interface{}]interface{}: // 情况 2：处理键为任意类型的 map
		node[el] = value

	case []interface{}: // 情况 3：处理切片
		idx, ok := el.(int)
		if !ok {
			return fmt.Errorf("expected int path element, got: %T (path element idx: %d)", el, i)
		}
		if idx < 0 || idx >= len(node) {
			return fmt.Errorf("index out of range: %d (path element idx: %d)", idx, i)
		}
		node[idx] = value

	default: // 情况 4：不支持的类型
		return fmt.Errorf("expected map or slice node, got: %T (path element idx: %d)", node, i)
	}

	return nil
}
```

整体上看，`Set` 函数与 `Get` 函数要考虑的几种情况相同，因为它们都支持的是同一种类型。并且 `Set` 函数也需要遍历 `path` 从外向内逐层获取对象的值，拿到嵌套在最内层对象的值，然后对其进行修改。所以 `Set` 函数先做的操作就是通过调用 `Get` 函数拿到末位元素的值，然后根据末位元素值的类型对其进行修改。

此外，修改类操作还包含一个 `SSet` 函数，`SSet` 针对的是 `map[string]interface{}` 类型的 `map` 进行赋值，这正好与 `SGet` 函数相对应。`SSet` 源码我就不解读了，就留作作业交给你自行去探索了。

#### Append

`Append` 函数用来为某个对象追加一个值，它最终作用于 `slice` 类型。

`Append` 函数源码如下：

```go
// Append 在动态结构（如嵌套的 map 或 slice）中，向路径指向的切片末尾追加元素
//
// The slice denoted by path must already exist.
//
// Path cannot be empty or nil, else an error is returned.
func Append(v interface{}, value interface{}, path ...interface{}) error {
	// 1. 路径非空校验
	if len(path) == 0 {
		return fmt.Errorf("path cannot be empty")
	}

	// 2. 获取路径指向的节点（必须是切片类型）
	node, err := Get(v, path...)
	if err != nil {
		return err
	}

	// 3. 类型断言：验证是否为切片
	s, ok := node.([]interface{})
	if !ok {
		return fmt.Errorf("expected slice node, got: %T (path element idx: %d)", node, len(path)-1)
	}

	// 4. 追加元素并更新原切片
	return Set(v, append(s, value), path...)
}
```

根据源码，可以发现 `Append` 函数实际上是 `Get` + `Set` 操作。与 `Set` 函数类似，它首先获取路径指向的节点，然后再更新节点对象。

而 `AppendMore` 函数与 `Append` 函数唯一的区别是，它能一次追加多个值。

其函数签名如下：

```go
func AppendMore(v interface{}, values []interface{}, path ...interface{}) error
```

你能想到它的源码是如何实现的吗？这个函数源码我就不贴的，你自己一看便知。

#### Delete

我们再来讲讲删除操作，`dyno` 包提供的 `Delete` 可以根据给定的 `path`，从 `map` 中删除指定的键值对，或从 `slice` 中删除指定元素。

`Delete` 函数源码如下：

```go
// Delete 根据给定的 path，从 map 中删除指定的键值对，或从 slice 中删除指定元素
func Delete(v interface{}, key interface{}, path ...interface{}) error {
	// 1. 路径校验：若 v 是切片，路径不能为空
	if len(path) == 0 {
		if _, ok := v.([]interface{}); ok {
			return fmt.Errorf("path cannot be empty if v is a slice")
		}
	}

	// 2. 获取路径末位节点
	node, err := Get(v, path...)
	if err != nil {
		return err
	}

	// 3. 根据末位节点类型删除
	switch node2 := node.(type) {
	case map[string]interface{}: // 情况 1：处理键为 string 的 map
		skey, ok := key.(string)
		if !ok {
			return fmt.Errorf("expected string key, got: %T", key)
		}
		delete(node2, skey) // 删除 string 类型的键

	case map[interface{}]interface{}: // 情况 2：处理键为任意类型的 map
		delete(node2, key) // 直接删除键

	case []interface{}: // 情况 3：处理切片
		idx, ok := key.(int)
		if !ok {
			return fmt.Errorf("expected int key, got: %T", key)
		}
		if idx < 0 || idx >= len(node2) {
			return fmt.Errorf("index out of range: %d", idx)
		}
		// 删除元素：移位 + 截断
		copy(node2[idx:], node2[idx+1:]) // 后续元素前移
		// Clear the emptied element:
		node2[len(node2)-1] = nil // 清空尾部引用（防内存泄漏）
		// Must set the new slice value:
		return Set(v, node2[:len(node2)-1], path...) // 将新的节点赋值给 v

	default: // 情况 4：不支持的类型
		return fmt.Errorf("expected map or slice node, got: %T (path element idx: %d)", node, len(path)-1)
	}

	return nil
}
```

源码讲解到这里，你可能已经注意到了，无论是 `Get` 或是 `Set`，还是这里的 `Delete`，它们需要考虑的情况基本相同，只不过操作不同。

`Delete` 函数针对每种情况，来实现对应的删除操作。如果节点类型为 `map[string]interface{}` 或 `map[interface{}]interface{}`，那么可以直接调用 `delete` 函数对其进行删除，如果节点类型为` []interface{}`，则需要在删除元素后注意清空对象的引用，防止占用资源。

#### ConvertMapI2MapS

最后，当我们要通过 `json.Marshal` 序列化对象时，如果遇到不支持的 `map[interface{}]interface{}` 类型，那么 `ConvertMapI2MapS` 函数就派上用场了。

`ConvertMapI2MapS` 函数源码如下：

```go
// ConvertMapI2MapS walks the given dynamic object recursively, and
// converts maps with interface{} key type to maps with string key type.
// This function comes handy if you want to marshal a dynamic object into
// JSON where maps with interface{} key type are not allowed.
//
// Recursion is implemented into values of the following types:
//   -map[interface{}]interface{}
//   -map[string]interface{}
//   -[]interface{}
//
// When converting map[interface{}]interface{} to map[string]interface{},
// fmt.Sprint() with default formatting is used to convert the key to a string key.
func ConvertMapI2MapS(v interface{}) interface{} {
	switch x := v.(type) {
	case map[interface{}]interface{}: // 目标转换类型，需要把 key 为 interface{} 类型的转换成 string
		m := map[string]interface{}{}
		for k, v2 := range x {
			switch k2 := k.(type) {
			case string: // 如果 key 已经是 string 类型，则直接使用
				m[k2] = ConvertMapI2MapS(v2)
			default: // 如果 key 是其他类型则需要转换成 string
				m[fmt.Sprint(k)] = ConvertMapI2MapS(v2)
			}
		}
		v = m

	case []interface{}: // 递归处理数组元素
		for i, v2 := range x {
			x[i] = ConvertMapI2MapS(v2)
		}

	case map[string]interface{}: // key 已经是 string，仅递归处理 value
		for k, v2 := range x {
			x[k] = ConvertMapI2MapS(v2)
		}
	}

	return v
}
```

这个函数代码还是比较清晰的，就是递归处理 `map[interface{}]interface{}` 类型，将其转换成目标类型 `map[string]interface{}`。

你可以在我的另一篇文章《[Go 语言中 YAML to JSON 踩坑笔记](https://mp.weixin.qq.com/s/rGKEgsDl0Wrl0m7W8i2soA)》中找到 `ConvertMapI2MapS` 函数的应用。

### 适用场景与优势

跟我前文我对 `dyno` 源码的讲解，想必你已经看出来 `dyno` 包的应用场景了。

`dyno` 包主要用于支持动态数据解析，方便我们的操作。我们可以使用 `dyno` 包来处理深度嵌套的 YAML/JSON 结构，无需预定义静态类型。并且结合 `dyno.Set`/`dyno.Append` 还支持动态修改结构。

此外，通过源码我们可以发现，`dyno` 包的源码实现完全不依赖反射（reflection），所以性能不错。

`dyno` 包有个非常实用的功能，就是在序列化时，用来兼容 `encoding/json`（不支持 `map[interface{}]interface{}`），使用 `dyno.ConvertMapI2MapS`转换键类型（得到 `map[string]interface{}`）。

### 总结

本文通过对 `dyno` 包源码的讲解，我们学习了如何方便的在 Go 中处理动态数据解析，并且我们还学会了其实现原理。

`dyno` 包完全不依赖反射，所以非常推荐使用。它可以很方便的帮我们解决在处理 YAML/JSON 文档时遇到的类型问题。

本文示例源码我都放在了 [GitHub](https://github.com/jianghushinian/blog-go-example/tree/main/yaml/dyno) 中，欢迎点击查看。

希望此文能对你有所启发。

**延伸阅读**

+ dyno 包文档：[https://pkg.go.dev/github.com/icza/dyno](https://pkg.go.dev/github.com/icza/dyno)
+ dyno GitHub 源码：[https://github.com/icza/dyno](https://github.com/icza/dyno)
+ Go 语言中 YAML to JSON 踩坑笔记：[https://mp.weixin.qq.com/s/rGKEgsDl0Wrl0m7W8i2soA](https://mp.weixin.qq.com/s/rGKEgsDl0Wrl0m7W8i2soA)
+ 本文 GitHub 示例代码：[https://github.com/jianghushinian/blog-go-example/tree/main/yaml/dyno](https://github.com/jianghushinian/blog-go-example/tree/main/yaml/dyno)
+ 本文永久地址：[https://jianghushinian.cn/2025/08/11/go-dyno/](https://jianghushinian.cn/2025/08/11/go-dyno/)

**联系我**

+ 公众号：[Go编程世界](https://jianghushinian.cn/about)
+ 微信：jianghushinian
+ 邮箱：[jianghushinian007@outlook.com](mailto:jianghushinian007@outlook.com)
+ 博客：[https://jianghushinian.cn](https://jianghushinian.cn)
+ GitHub：[https://github.com/jianghushinian](https://github.com/jianghushinian)

加我微信备注「加群」，拉你进 Go 语言学习交流群。
