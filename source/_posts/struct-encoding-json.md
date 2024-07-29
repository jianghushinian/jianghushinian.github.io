---
title: '在 Go 中如何使用反射实现简易版 encoding/json'
date: 2024-07-28 09:37:41
tags:
- Go
- JSON
categories:
- Go
---

在使用 Go 语言开发过程中，我们经常需要实现结构体到 JSON 字符串的序列化（Marshalling）或 JSON 字符串到结构体的反序列化（Unmarshalling）操作。Go 为我们提供了 `encoding/json` 库可以很方便的实现这一需求。

在本文中，我们将探索如何使用 Go 的反射机制自己来实现一个简易版的 `encoding/json` 库。这个过程不仅能帮助我们理解序列化和反序列化的基本原理，还能提供一种实用的反射使用方法，加深我们对反射的理解。

通过本文的学习，我们将实现一个能够将结构体转和 JSON 字符串互相转换的包。

<!-- more -->

### encoding/json

我们先来回顾下在 Go 中如何使用 `encoding/json` 库实现结构体转和 JSON 字符串互转。

示例代码如下：

```go
package main

import (
	"encoding/json"
	"fmt"
)

type User struct {
	Name  string `json:"name"`
	Age   int    `json:"age"`
	Email string
}

func main() {
	{
		user := User{
			Name:  "江湖十年",
			Age:   20,
			Email: "jianghushinian007@outlook.com",
		}

		jsonData, err := json.Marshal(user)
		if err != nil {
			fmt.Println("Error marshal to JSON:", err)
			return
		}

		fmt.Printf("JSON data: %s\n", jsonData)
	}

	{
		jsonData := `{"name": "江湖十年", "age": 20, "Email": "jianghushinian007@outlook.com"}`

		var user User
		err := json.Unmarshal([]byte(jsonData), &user)
		if err != nil {
			fmt.Println("Error unmarshal from JSON:", err)
			return
		}

		fmt.Printf("User struct: %+v\n", user)
	}
}
```

示例程序中定义了一个 `User` 结构体，结构体包含三个字段，`Name`、`Age` 和 `Email`。

`encoding/json` 会根据结构体字段上的 JSON `Tag`（标签）进行序列化和反序列化。序列化时，JSON `Tag` 会作为 JSON 字符串的 `key`，字段值作为 JSON 字符串的 `value`。反序列化时，JSON 字符串的 `key` 所对应的值会被映射到具有同样 JSON `Tag` 的结构体字段上。

`Name` 字段的 JSON `Tag` 是 `name`，则对应的 JSON 字符串 `key` 为 `name`；`Age` 字段的 JSON `Tag` 是 `age`，则对应的 JSON 字符串 `key` 为 `age`；`Email` 字段没有 JSON `Tag`，则默认会使用 `Email` 作为对应的 JSON 字符串 `key`。

执行示例代码，得到如下输出：

```bash
$ go run main.go
JSON data: {"name":"江湖十年","age":20,"Email":"jianghushinian007@outlook.com"}
User struct: {Name:江湖十年 Age:20 Email:jianghushinian007@outlook.com}
```

### reflect 简介

`reflect` 是 Go 语言为我们提供的**反射**库，用于在**运行时检查类型并操作对象**。它是实现动态编程和元编程的基础，使程序能够在**运行时获取类型信息并进行相应的操作**。

有如下示例代码：

```go
package main

import (
	"fmt"
	"reflect"
)

type User struct {
	Name  string `json:"name"`
	Age   int    `json:"age"`
	Email string
}

func main() {
	// 内置类型
	{
		age := 20

		val := reflect.ValueOf(age)
		typ := reflect.TypeOf(age)
		fmt.Println(val, typ)

	// 自定义结构体类型
	{
		user := User{
			Name:  "江湖十年",
			Age:   20,
			Email: "jianghushinian007@outlook.com",
		}

		val := reflect.ValueOf(user)
		typ := reflect.TypeOf(user)
		fmt.Println(val, typ)
	}
}
```

执行示例代码，得到如下输出：

```bash
$ go run main.go
20 int
{江湖十年 20 jianghushinian007@outlook.com} main.User
```

`reflect` 最常用的两个方法分别是 `reflect.ValueOf` 和 `reflect.TypeOf`，它们分别返回 `reflect.Value` 和 `reflect.Type` 类型。这两个方法可以应用于任何类型对象（`any`）。

- `reflect.Value`：表示一个 Go 值，它提供了一些方法，**可以获取值的详细信息，也可以操作值**，例如获取值的类型、设置值等。

- `reflect.Type`：表示一个 Go 类型，它提供了一些方法，**可以获取类型的详细信息**，例如类型的名称（`Name`）、种类（`Kind`，基本类型、结构体、切片等）。

接下来对 `reflect.Value` 和 `reflect.Type` 类型的常用方法进行介绍，以如下实例化 `User` 结构体指针作为被操作对象：

```go
// 实例化 User 结构体指针
user := &User{
	Name:  "江湖十年",
	Age:   20,
	Email: "jianghushinian007@outlook.com",
}
```

#### reflect.Value 常用方法

`reflect.Value` 提供了 `Kind` 方法可以获取对应的类型`类别`：

```go
// 注意这里传递的是指针类型
kind := reflect.ValueOf(user).Kind()
fmt.Println(kind)
kind = reflect.ValueOf(*user).Kind()
fmt.Println(kind)
kind = reflect.ValueOf(user).Elem().Kind()
fmt.Println(kind)
```

这段示例代码将得到如下输出：

```bash
ptr
struct
struct
```

这里 `Kind` 方法返回的是 `User` 的底层类型 `struct`，以及 `ptr` 类型，`ptr` 代表指针类型。

值得注意的是，如果传递给 `reflect.ValueOf` 的是指针类型（`user`），需要使用 `Elem` 方法获取指针指向的值；如果传递给 `reflect.ValueOf` 的是值类型（`*user`），则可以直接得到值。

使用指针类型的好处是可以使用 `reflect.Value` 提供的 `Set<Type>` 方法直接修改 `user` 字段的值，稍后讲解。

`reflect.Value` 同样提供了 `Type` 方法，可以得到 `reflect.Type`：

```go
// 以下二者等价
tpy := reflect.ValueOf(user).Type()
fmt.Println(tpy)
tpy1 := reflect.TypeOf(user)
fmt.Println(tpy1)
fmt.Println(reflect.DeepEqual(tpy, tpy1))
```

这与 `reflect.TypeOf` 等价。

这段示例代码将得到如下输出：

```bash
*main.User
*main.User
true
```

我们有多种方式可以获取结构体值字段：

```go
nameField := reflect.ValueOf(user).Elem().FieldByName("Name")
ageField := reflect.ValueOf(user).Elem().FieldByIndex([]int{1})
emailField := reflect.ValueOf(user).Elem().Field(2)
```

- `FieldByName` 方法可以通过字段名获取结构体字段。

- `FieldByIndex` 方法通过索引切片获取结构体字段。

- `Field` 方法通过索引获取结构体字段。

实际上 `FieldByIndex` 方法内部调用的也是 `Field` 方法。这里的索引是结构体字段按照顺序排序所在位置，即 `Name` 字段索引为 0，`Age` 字段索引为 1，`Email` 字段索引为 2。

我们可以使用 `NumField` 获取结构体字段总个数：

```go
numField := reflect.ValueOf(*user).NumField()
fmt.Println(numField)
```

拿到结构体字段对象后，可以根据其具体类型获取对应值：

```go
fmt.Println(nameField.String())
fmt.Println(ageField.Int())
fmt.Println(emailField.String())
```

以上示例代码将得到如下输出：

```bash
3
江湖十年
20
jianghushinian007@outlook.com
```

因为我们传递给 `reflect.ValueOf` 函数的是 `User` 结构体指针，所以可以使用 `reflect.Value` 提供的 `Set<Type>` 方法设置结构体字段的值：

```go
nameField.SetString("jianghushinian")             // 设置 Name 字段的值
ageField.SetInt(18)                               // 设置 Age 字段的值
emailField.SetString("jianghushinian007@163.com") // 设置 Email 字段的值
```

现在打印 `user` 对象：

```go
fmt.Println(user)
```

得到输出：

```bash
&{jianghushinian 18 jianghushinian007@163.com}
```

如果我们传递给 `reflect.ValueOf` 函数的不是 `User` 结构体指针，而是结构体对象：

```go
nameField := reflect.ValueOf(*user).FieldByName("Name")
```

现在去设置字段值：

```go
nameField.SetString("jianghushinian")
```

程序会直接 `panic`：

```bash
panic: reflect: reflect.Value.SetString using unaddressable value
```

此外，我们还可以总结一个规律，使用指针时，就需要通过 `Elem` 方法获取指针指向的值，不使用指针就不需要调用 `Elem` 方法。

#### reflect.Type 常用方法

现在我们再来简单介绍下 `reflect.Type` 的几个常用方法。

`reflect.Type` 同样提供了如下几个方法，与 `reflect.Value` 对应：

```go
nameField, _ := reflect.TypeOf(user).Elem().FieldByName("Name")
ageField := reflect.TypeOf(user).Elem().FieldByIndex([]int{1})
emailField := reflect.TypeOf(user).Elem().Field(2)
```

我们来输出下这几个对象的值：

```go
fmt.Printf("%+v\n", nameField)
fmt.Printf("%+v\n", ageField)
fmt.Printf("%+v\n", emailField)
```

得到如下输出：

```bash
{Name:Name PkgPath: Type:string Tag:json:"name" Offset:0 Index:[0] Anonymous:false}
{Name:Age PkgPath: Type:int Tag:json:"age" Offset:16 Index:[1] Anonymous:false}
{Name:Email PkgPath: Type:string Tag: Offset:24 Index:[2] Anonymous:false}
```

这里打印了结构体每个字段的信息。

- `Name` 对应字段名。

- `PkgPath` 是包路径。

- `Type` 是结构体字段类型。

- `Tag` 即为字段标签。

- `Offset` 是字段偏移量。如果你不清楚什么是字段偏移量，可以参考我写的另一篇文章[《Go 语言中的结构体内存对齐你了解吗？》](https://jianghushinian.cn/2024/07/07/do-you-understand-the-memory-alignment-of-structs-in-the-go/)。

- `Index` 是字段索引位置。

- `Anonymous` 表示是否为匿名字段。比如如下结构体：

```go
type User struct {
	Name  string `json:"name"`
	Age   int    `json:"age"`
	Email string
	string
}
```

这个结构体定义中，最后一个字段就是匿名字段。

现在我们想获取结构体字段 JSON `Tag`，可以这样做：

```go
tag := nameField.Tag
fmt.Printf("%+v\n", tag)
fmt.Printf("%+v\n", tag.Get("json"))
```

将得到如下输出：

```bash
json:"name"
name
```

`reflect` 基础语法就讲解到这里，更多使用方法需要我们在以后的的实践中去探索。

### 使用 reflect 实现 encoding/json

接下来就看看，如何使用 `reflect` 自己实现一个简易版本的 `encoding/json`。

示例程序目录结构如下：

```bash
$ tree                    
.
├── encoding
│   └── json
│       ├── decode.go
│       └── encode.go
├── go.mod
└── main.go
```

- `encoding/json/encode.go` 用于实现序列化功能。

- `encoding/json/decode.go` 用于实现反序列化功能。

- `main.go` 用来验证这个简易版的 `encoding/json` 功能。

#### 序列化

首先是实现序列化的代码：

```go
package json

import (
	"fmt"
	"reflect"
	"strconv"
	"strings"
)

// Marshal 序列化
func Marshal(v any) (string, error) {
	// 拿到对象 v 的 reflect.Value 和 reflect.Type
	val := reflect.ValueOf(v)
	if val.Kind() != reflect.Struct {
		return "", fmt.Errorf("only structs are supported")
	}
	typ := val.Type()

	// 用来保存 JSON 字符串
	jsonBuilder := strings.Builder{}

	// NOTE: 三步走拼接 JSON 字符串

	// 1. JSON 左花括号
	jsonBuilder.WriteString("{")

	// 2. key/value
	for i := 0; i < val.NumField(); i++ {
		fieldVal := val.Field(i)
		fieldType := typ.Field(i)

		// 获取 JSON 标签
		tag := fieldType.Tag.Get("json")
		if tag == "" {
			tag = fieldType.Name
		}

		jsonBuilder.WriteString(`"` + tag + `":`)

		// 根据字段类型转换，仅支持 string/int
		switch fieldVal.Kind() {
		case reflect.String:
			jsonBuilder.WriteString(`"` + fieldVal.String() + `"`)
		case reflect.Int:
			jsonBuilder.WriteString(strconv.FormatInt(fieldVal.Int(), 10))
		default:
			return "", fmt.Errorf("unsupported field type: %s", fieldVal.Kind())
		}

		if i < val.NumField()-1 {
			jsonBuilder.WriteString(",")
		}
	}

	// 3. JSON 右花括号
	jsonBuilder.WriteString("}")

	return jsonBuilder.String(), nil
}
```

这段代码中没有新的 `reflect` 语法，我们都在前文中介绍了，这里捋一下代码逻辑。

所谓序列化操作，就是 Go 结构体转 JSON 字符串的操作。

这里函数名参考 `encoding/json` 同样被定义为 `Marshal`。

首先我们拿到对象 `v` 的 `reflect.Value` 和 `reflect.Type`，待后续使用。

接着使用 `strings.Builder` 构造了一个用来保存 JSON 字符串信息的对象 `jsonBuilder`。

构造 JSON 字符串分三步走：

1. 先写入 JSON 左花括号 `{` 内容到 `jsonBuilder`。

2. 根据结构体字段和值，构造 JSON 字符串的键值对 `key/value` 并写入 `jsonBuilder`。

3. 最后写入 JSON 右花括号 `}` 内容到 `jsonBuilder`。

函数最终返回 `jsonBuilder.String()` 即为 JSON 字符串。

这里面主要逻辑都在步骤 2 中。

首先会遍历结构体每个字段，并使用如下方式获取每个字段对应的 JSON `Tag`：

```go
tag := fieldType.Tag.Get("json")
if tag == "" {
	tag = fieldType.Name
}
```

当 JSON `Tag` 不存在，则默认使用结构体字段名作为 JSON 字符串的 `key`，比如 `User.Email` 字段。

将 JSON `key` 和 `:` 写入 `jsonBuilder`：

```go
jsonBuilder.WriteString(`"` + tag + `":`)
```

然后根据结构体字段类型转换成对应的 JSON 数据类型，写入 `jsonBuilder`：

```go
switch fieldVal.Kind() {
case reflect.String:
	jsonBuilder.WriteString(`"` + fieldVal.String() + `"`)
case reflect.Int:
	jsonBuilder.WriteString(strconv.FormatInt(fieldVal.Int(), 10))
default:
	return "", fmt.Errorf("unsupported field type: %s", fieldVal.Kind())
}
```

每次循环末尾，判断是否为结构体最后一个字段，如果不是，则写入分隔符 `,`：

```go
if i < val.NumField()-1 {
	jsonBuilder.WriteString(",")
}
```

至此，序列化代码逻辑大功告成。

我们可以使用如下示例代码进行测试：

```go
import simplejson "github.com/jianghushinian/blog-go-example/struct/encoding-json/encoding/json"

...

user := User{
	Name:  "江湖十年",
	Age:   20,
	Email: "jianghushinian007@outlook.com",
}

jsonData, err := simplejson.Marshal(user)
if err != nil {
	fmt.Println("Error marshal to JSON:", err)
	return
}

fmt.Printf("JSON data: %s\n", jsonData)
```

执行示例代码，得到如下输出：

```bash
$ go run main.go
JSON data: {"name":"江湖十年","age":20,"Email":"jianghushinian007@outlook.com"}
```

没有任何问题，与原生的 `encoding/json` 中的 `Marshal` 方法表现一致。

#### 反序列化

接下来是实现反序列化的代码：

```go
package json

import (
	"errors"
	"fmt"
	"reflect"
	"strconv"
	"strings"
)

// Unmarshal 反序列化
func Unmarshal(data []byte, v interface{}) error {
	parsedData, err := parseJSON(string(data))
	if err != nil {
		return err
	}

	val := reflect.ValueOf(v).Elem()
	typ := val.Type()

	for i := 0; i < val.NumField(); i++ {
		fieldVal := val.Field(i)
		fieldType := typ.Field(i)

		// 获取 JSON 标签
		tag := fieldType.Tag.Get("json")
		if tag == "" {
			tag = fieldType.Name
		}

		// 从解析的数据中获取值
		if value, ok := parsedData[tag]; ok {
			switch fieldVal.Kind() {
			case reflect.String:
				fieldVal.SetString(value)
			case reflect.Int:
				intValue, err := strconv.Atoi(value)
				if err != nil {
					return err
				}
				fieldVal.SetInt(int64(intValue))
			default:
				return fmt.Errorf("unsupported field type: %s", fieldVal.Kind())
			}
		}
	}

	return nil
}
```

这段代码中同样没有新的 `reflect` 语法。

所谓反序列化操作，就是 JSON 字符串转 Go 结构体的操作。

这里函数名参考 `encoding/json` 同样被定义为 `Unmarshal`，并且函数签名也保持一致。

反序列化操作首先使用 `parseJSON` 函数解析传递进来的 JSON 数据，得到 `parsedData`。

`parsedData` 类型为 `map[string]string`，`map` 的 `key` 为 JSON 字符串中的 `key`，`map` 的 `value` 即为 JSON 字符串中的 `value`。

接下来核心逻辑是遍历结构体每个字段，并获取字段对应的 JSON `Tag`：

```go
tag := fieldType.Tag.Get("json")
if tag == "" {
	tag = fieldType.Name
}
```

当 JSON `Tag` 不存在，则默认使用结构体字段名作为 JSON 字符串的 `key`，比如 `User.Email` 字段。

然后根据 JSON `Tag` 从解析后的 `parsedData` 数据中获取 `key/value`：

```go
if value, ok := parsedData[tag]; ok {
	switch fieldVal.Kind() {
	case reflect.String:
		fieldVal.SetString(value)
	case reflect.Int:
		intValue, err := strconv.Atoi(value)
		if err != nil {
			return err
		}
		fieldVal.SetInt(int64(intValue))
	default:
		return fmt.Errorf("unsupported field type: %s", fieldVal.Kind())
	}
}
```

这里根据结构体字段的类型，将 `parsedData` 中对应的字符串 `value` 转换成对应类型。并使用 `reflect.Value` 提供的 `SetString` 和 `SetInt` 方法设置字段的值。

现在，我们唯一没有讲解的逻辑就只剩下 `parseJSON` 函数了。

`parseJSON` 函数定义如下：

```go
// 简易版 JSON 解析器，仅支持 string/int 且不考虑嵌套
func parseJSON(data string) (map[string]string, error) {
	result := make(map[string]string)

	data = strings.TrimSpace(data)
	if len(data) < 2 || data[0] != '{' || data[len(data)-1] != '}' {
		return nil, errors.New("invalid JSON")
	}

	data = data[1 : len(data)-1]
	parts := strings.Split(data, ",")
	for _, part := range parts {
		kv := strings.SplitN(part, ":", 2)
		if len(kv) != 2 {
			return nil, errors.New("invalid JSON")
		}

		k := strings.Trim(strings.TrimSpace(kv[0]), `"`)
		v := strings.Trim(strings.TrimSpace(kv[1]), `"`)

		result[k] = v
	}

	return result, nil
}
```

`parseJSON` 实现了一个简易版本的 JSON 字符串解析器，能够将 JSON 字符串的 `key/value` 解析出来，并保存到 `map[string]string` 中。

我们可以使用如下示例代码进行测试 `Unmarshal` 代码逻辑是否正确：

```go
import simplejson "github.com/jianghushinian/blog-go-example/struct/encoding-json/encoding/json"

...

jsonData := `{"name": "江湖十年", "age": 20, "Email": "jianghushinian007@outlook.com"}`

var user User
err := simplejson.Unmarshal([]byte(jsonData), &user)
if err != nil {
	fmt.Println("Error unmarshal from JSON:", err)
	return
}

fmt.Printf("User struct: %+v\n", user)
```

执行示例代码，得到如下输出：

```bash
$ go run main.go
User struct: {Name:江湖十年 Age:20 Email:jianghushinian007@outlook.com}
```

没有任何问题，与原生的 `encoding/json` 中的 `Unmarshal` 方法表现一致。

### 总结

`reflect` 是 Go 语言为我们提供的反射库，用于在运行时检查类型并操作对象。

`reflect` 最常用的两个方法分别是 `reflect.ValueOf` 和 `reflect.TypeOf`，调用这两个方法分别可以得到 `reflect.Value` 和 `reflect.Type` 类型。

有了这两个类型及其方法，我们可以获取任意一个 Go 对象的类型信息、值的详细信息和操作值，可见反射之强大。

本文使用 `reflect` 反射包实现了一个简易版本的 `encoding/json`。

虽然是简易版本，很多 `case` 和异常都没有考虑，但这足够我们学习 `encoding/json` 原理了，并且这也是一个很好的 `reflect` 实践应用。

不过话虽如此，对于何时使用反射，我的观点是：反射固然强大，有了它我们的代码足够灵活，但是过度使用反射会让代码变得复杂且混乱。所以非必要，尽量不要使用反射。

本文示例源码我都放在了 [GitHub](https://github.com/jianghushinian/blog-go-example/tree/main/struct/encoding-json) 中，欢迎点击查看。

希望此文能对你有所启发。

**延伸阅读**

- reflect@go1.22.5 Documentation：https://pkg.go.dev/reflect@go1.22.5
- encoding/json@go1.22.5 Documentation：https://pkg.go.dev/encoding/json@go1.22.5
- 本文 GitHub 示例代码：https://github.com/jianghushinian/blog-go-example/tree/main/struct/encoding-json

**联系我**

- 公众号：[Go编程世界](https://jianghushinian.cn/about)
- 微信：jianghushinian
- 邮箱：[jianghushinian007@outlook.com](mailto:jianghushinian007@outlook.com)
- 博客：https://jianghushinian.cn
