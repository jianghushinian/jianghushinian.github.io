---
title: 'Go 语言中 YAML to JSON 踩坑笔记'
date: 2025-08-03 21:39:54
tags:
- Go
categories:
- Go
---

最近在搬砖的过程中遇到了一个在 Go 代码中 YAML 转 JSON 引发报错的小问题，随手记录一下。场景是这样的，我实现了一个功能，支持用户上传 YAML/JSON 格式的文档，为了方便，我会将文档全部转为 JSON 格式再来统一处理。平时经常操作 YAML，也经常操作 JSON，但二者互转的场景不怎么会遇到，所以因为一时疏忽，就遇到了问题。

<!-- more -->

### 使用 yaml.v3 包处理 YAML

首先，假设我们有如下一段 YAML 文档：

```yaml
object:
  a: 1
  1: 2
  "1": 3
  key: value
  array:
  - null_value: 
  - boolean: true
  - integer: 1
```

这个 YAML 文档是一个对象结构，并且对象内部还包含了一个数组。细心观察，你会发现对象中有两个 `key` 比较近似，一个是数字类型的 `1`，另一个是字符串类型的 `"1"`，这点需要注意。

在 Go 语言中可以使用 `yaml.v3` 来处理 YAML 文档，使用内置的 `encoding/json` 来处理 JSON 内容。我们来编写一段 Go 代码将上面这个 YAML 文档转换成 JSON 对象。

实现如下：

```go
package main

import (
	"encoding/json"
	"fmt"
	"log"

	"github.com/icza/dyno" // 用于递归转换 map[interface{}] => map[string]
	"gopkg.in/yaml.v3"     // YAML 解析库
)

func main() {
	yamlExample := `
object:
  a: 1
  1: 2
  "1": 3
  key: value
  array:
  - null_value: 
  - boolean: true
  - integer: 1
`

	// 解析 YAML
	var data interface{}
	if err := yaml.Unmarshal([]byte(yamlExample), &data); err != nil {
		log.Fatalf("YAML parse error: %v", err)
	}
	fmt.Printf("Type: %T\nValue: %#v\n", data, data)

	fmt.Println("--------------------------")

	// 关键：递归转换 map 键类型（interface{} → string）
	convertedData := dyno.ConvertMapI2MapS(data)

	// 转换为 JSON
	jsonData, err := json.Marshal(convertedData)
	if err != nil {
		log.Fatalf("JSON convert error: %v", err)
	}
	fmt.Printf("Type: %T\nValue: %s\n", jsonData, jsonData)
}
```

这段代码用于将 YAML 格式的文档解析并转换成 JSON 格式。

执行示例代码，得到输出如下：

```bash
$ go run main.go
2025/07/31 23:22:49 YAML parse error: yaml: unmarshal errors:
  line 5: mapping key "1" already defined at line 4
exit status 1
```

根据输出结果可以发现，代码直接就报错了。

错误信息提示 `"1"` 这个 `key` 之前已经存在。此时，我们可以尝试注释掉 `"1": 3` 这个键值对：

```go
	yamlExample := `
object:
  a: 1
  1: 2
  # "1": 3
  key: value
  array:
  - null_value: 
  - boolean: true
  - integer: 1
`
```

然后再次执行示例代码，得到输出如下：

```bash
$ go run main.go              
Type: map[string]interface {}
Value: map[string]interface {}{"object":map[interface {}]interface {}{"a":1, "array":[]interface {}{map[string]interface {}{"null_value":interface {}(nil)}, map[string]interface {}{"boolean":true}, map[string]interface {}{"integer":1}}, "key":"value", 1:2}}
--------------------------
Type: []uint8
Value: {"object":{"1":2,"a":1,"array":[{"null_value":null},{"boolean":true},{"integer":1}],"key":"value"}}
```

这一次，我们成功的将 YAML 转换换成了 JSON。

由此可见，在 Go 语言中，使用 `yaml.v3` 包解析 YAML 时，`1` 和 `"1"` 会被认为是对象的同一个 `key`，会引发冲突。

此外，你应该也已经发现，示例代码中，通过 `yaml.Unmarshal` 反序列化出来的的 `data` 对象并没有直接交给 `json.Marshal` 去处理，而是中间经过了 `dyno.ConvertMapI2MapS(data)` 的转换，之后再进行 JSON 序列化操作。

如果我们去掉 `dyno.ConvertMapI2MapS(data)` 这行代码，写成这样：

```go
package main

import (
	"encoding/json"
	"fmt"
	"log"

	"gopkg.in/yaml.v3" // YAML 解析库
)

func main() {
	yamlExample := `
object:
  a: 1
  1: 2
  # "1": 3
  key: value
  array:
  - null_value: 
  - boolean: true
  - integer: 1
`

	// 解析 YAML
	var data interface{}
	if err := yaml.Unmarshal([]byte(yamlExample), &data); err != nil {
		log.Fatalf("YAML parse error: %v", err)
	}
	fmt.Printf("Type: %T\nValue: %#v\n", data, data)

	fmt.Println("--------------------------")

	// 转换为 JSON
	jsonData, err := json.Marshal(data)
	if err != nil {
		log.Fatalf("JSON convert error: %v", err)
	}
	fmt.Printf("Type: %T\nValue: %s\n", jsonData, jsonData)
}
```

执行示例代码，得到输出如下：

```bash
$ go run main.go
Type: map[string]interface {}
Value: map[string]interface {}{"object":map[interface {}]interface {}{"a":1, "array":[]interface {}{map[string]interface {}{"null_value":interface {}(nil)}, map[string]interface {}{"boolean":true}, map[string]interface {}{"integer":1}}, "key":"value", 1:2}}
--------------------------
2025/07/31 23:34:26 JSON convert error: json: unsupported type: map[interface {}]interface {}
exit status 1
```

这一次，程序同样会报错。

不过错误信息发生了变化，这次不再是 `yaml.Unmarshal` 报错，而是 `json.Marshal` 出现报错。错误信息提示 `json.Marshal` 不支持 `map[interface {}]interface {}` 类型对象的序列化操作。

我们可以整理一下 `yaml.Unmarshal `反序列化得到的对象，这样更加清晰：

```go
map[string]interface{}{
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
```

可以看到，这个对象内部嵌套了 `map[interface {}]interface {}` 类型。

因为 `json.Marshal` 接收的对象 `key` 必须是 `string` 类型，而 Go 语言中 `map` 的 `key` 只要是可比较的类型即可（这就包括了可比较的 `interface{}` 类型）。而代码中 `dyno.ConvertMapI2MapS` 函数的作用，正是将 `map[interface {}]interface {}` 类型，转换成 `map[string]interface {}` 类型。

`dyno.ConvertMapI2MapS` 函数源码如下：

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

这个函数代码还是比较清晰的，就是递归处理 `map[interface{}]interface{}` 类型，将其转换成目标类型 `map[string]interface {}`。

那么，为什么 `yaml.Unmarshal` 返回的对象中会包含 `map[interface{}]interface{}` 类型，而不能直接全部都用 `map[string]interface {}` 类型呢？其实在 `yaml.v3` 官方代码仓库 [issues/137](https://github.com/go-yaml/yaml/issues/137) 中还真有人问过作者，作者的回复是：

> Unfortunately not.. Unlike json, yaml can take keys of arbitrary types.
>

即，不能这么做，因为 YAML 规范是允许**任意类型值**作为对象 `key` 的，所以这也是为什么在我提供的 YAML 示例文档中 `1: 2` 和 `"1": 3` 同时存在是合法的。

所以，你在使用 Go 语言操作 YAML 转 JSON 时，保险起见，一定要加上 `dyno.ConvertMapI2MapS` 函数，来避免直接将 `map[interface{}]interface{}` 类型传递给 `json.Marshal`，以免踩坑。

其实，这个问题不止我一个人遇到，你可以直接在 [issues](https://github.com/go-yaml/yaml/issues?q=json%3A%20unsupported%20type%3A%20map%5Binterface%20%7B%7D%5Dinterface%20%7B%7D) 中搜到这个问题：

![json unsupported type](json-unsupported-type.png)

[gopkg.in/yaml.v3](https://github.com/go-yaml/yaml) 包是由 Canonical 公司（Ubuntu 背后的公司）为支持其项目 Juju 开发的社区库，是 Go 社区目前使用最广泛的 YAML 解析库。但遗憾的是，2025 年 4 月 2 日，此项目已被作者 Gustavo Niemeyer 标记为不再维护。

为此，我又特意测试了另一个款处理 YAML 的 Go 包，我们一起来看下效果。

### 使用 go-yaml 包处理 YAML

`go-yaml` 包使用示例如下：

```go
package main

import (
	"encoding/json"
	"fmt"
	"log"

	"github.com/goccy/go-yaml" // YAML 解析库
)

func main() {
	yamlExample := `
object:
  a: 1
  1: 2
  "1": 3
  key: value
  array:
  - null_value: 
  - boolean: true
  - integer: 1
`

	// 解析 YAML
	var data interface{}
	if err := yaml.Unmarshal([]byte(yamlExample), &data); err != nil {
		log.Fatalf("YAML parse error: %v", err)
	}
	fmt.Printf("Type: %T\nValue: %#v\n", data, data)

	fmt.Println("--------------------------")

	// 转换为 JSON
	jsonData, err := json.Marshal(data)
	if err != nil {
		log.Fatalf("JSON convert error: %v", err)
	}
	fmt.Printf("Type: %T\nValue: %s\n", jsonData, jsonData)
}
```

可以发现，这个包用法与 `yaml.v3` 如出一辙，我们仅需要修改 `import` 语句，就可以直接切换到 `go-yaml` 包。

执行示例代码，得到输出如下：

```bash
$ go run goccy-go-yaml/main.go
2025/07/31 23:25:15 YAML parse error: [5:3] mapping key "1" already defined at [4:3]
   2 | object:
   3 |   a: 1
   4 |   1: 2
>  5 |   "1": 3
         ^
   6 |   key: value
   7 |   array:
   8 |   - null_value:
exit status 1
```

与 `yaml.v3` 包一样，`go-yaml` 包也无法处理 YAML 文档中同时包含 `1: 2` 和 `"1": 3` 两个键值对的问题。不过这里的报错信息更加清晰一些。

现在，我们同样注释掉 `"1": 3` 这个键值对：

```go
	yamlExample := `
object:
  a: 1
  1: 2
  # "1": 3
  key: value
  array:
  - null_value: 
  - boolean: true
  - integer: 1
`
```

再次执行示例代码，得到输出如下：

```bash
$ go run goccy-go-yaml/main.go 
Type: map[string]interface {}
Value: map[string]interface {}{"object":map[string]interface {}{"1":0x2, "a":0x1, "array":[]interface {}{map[string]interface {}{"null_value":interface {}(nil)}, map[string]interface {}{"boolean":true}, map[string]interface {}{"integer":0x1}}, "key":"value"}}
--------------------------
Type: []uint8
Value: {"object":{"1":2,"a":1,"array":[{"null_value":null},{"boolean":true},{"integer":1}],"key":"value"}}
```

我们成功的将 YAML 转换换成了 JSON。

可以发现，`go-yaml` 包与 `yaml.v3` 包用法一致，使用 `go-yaml` 包的好处是 `yaml.v3` + `dyno.ConvertMapI2MapS` 才能完成的事情，`go-yaml` 一个包就能够搞定。

到这里，如何在 Go 语言中操作 YAML to JSON 就讲完了。不过文章还没完，我再用 Python 处理一下 YAML to JSON 的场景，来解释一下，为什么我会在编写 Go 代码的时候疏忽了这个问题。

### Python 中处理 YAML to JSON

我们现在使用 Python 来实现 YAML to JSON，同样的还是操作前文中的 YAML 文档。

示例代码如下：

```python
import json
import yaml

yaml_example = """
object:
  a: 1
  1: 2
  "1": 3
  key: value
  array:
  - null_value:
  - boolean: true
  - integer: 1
"""


def main():
    y = yaml.load(yaml_example, Loader=yaml.SafeLoader)
    print(type(y), y)

    print("--------------------------")

    j = json.dumps(y)
    print(type(j), j)

    print("--------------------------")

    d = json.loads(j)
    print(type(d), d)

if __name__ == '__main__':
    main()
```

执行示例代码，得到输出如下：

```bash
$ python main.py
<class 'dict'> {'object': {'a': 1, 1: 2, '1': 3, 'key': 'value', 'array': [{'null_value': None}, {'boolean': True}, {'integer': 1}]}}
--------------------------
<class 'str'> {"object": {"a": 1, "1": 2, "1": 3, "key": "value", "array": [{"null_value": null}, {"boolean": true}, {"integer": 1}]}}
--------------------------
<class 'dict'> {'object': {'a': 1, '1': 3, 'key': 'value', 'array': [{'null_value': None}, {'boolean': True}, {'integer': 1}]}}
```

可以发现，Python 完美的处理了 YAML 文档中同时包含 `1: 2` 和 `"1": 3` 两个键值对的场景。Python 中的 `dict` 同样支持 `1` 和 `"1"` 同时作为对象的 `key`。

并且，还有一个有意思的现象，`json.dumps(y)` 序列化得到的 JSON 字符串中，同时出现了 `"1": 2`、`"1": 3` 两个键值对，然后使用这个 JSON 字符串，再次进行反序列化操作 `json.loads(j)`，得到新的 `dict` 对象中，只保留了一个`'1': 3` 键值对，并没有 `1: 2` 键值对。

这是因为，JSON 规范本身就规定 `key` 必须是字符串类型，所以 `1` 和 `"1"` 最终变成了同一个 `key`，出现了覆盖现象。

所以你看，同样的 YAML 文档，在 Python 就没有问题，在 Go 中缺遇到了问题。这也是我非常喜欢 Python 的一点，它足够灵活。顺便说句题外话，由于 Python 语言非常灵活的特点，很多人误以为 Python 是弱类型语言，事实并非如此，Python 是**动态强类型**语言，而 Go 是**静态强类型**语言。

### 原因分析

我们演示并讲解了 YAML to JSON 报错的问题，其实归根到底是 YAML 和 JSON 格式的问题。YAML 是 JSON 的超集，任何合法的 JSON 文档，都应该是合法的 YAML。但是，反之则不行，并不是所有合法的 YAML 都是合法的 JSON。

YAML 允许数字作为对象的 `key`，这点与 Python 的 `dict` 完美结合，但是与 Go 却存在冲突，Go 更加严格。所以才会出现，同样的 YAML 文档，在 Python 中处理不会遇到问题，在 Go 中处理却得到报错。

以上这些都是我们通过编程进行试验得出的结果。

问题虽然解决了，不过我还想在文章中扩展一些关于 YAML 和 JSON 规范的知识。我们一起来看一下更具权威性的规范。

### YAML 规范

YAML 规范官方文档为 [https://yaml.org/](https://yaml.org/) ，如果你感兴趣可以点击进去详细阅读。

这里我为你展示一个我认为比较有意思的点，如下：

> A question mark and space (“`? `”) indicate a complex [mapping](https://yaml.org/spec/1.2.2/#mapping) [key](https://yaml.org/spec/1.2.2/#nodes). Within a [block collection](https://yaml.org/spec/1.2.2/#block-collection-styles), [key/value pairs](https://yaml.org/spec/1.2.2/#mapping) can start immediately following the [dash](https://yaml.org/spec/1.2.2/#block-sequences), [colon](https://yaml.org/spec/1.2.2/#flow-mappings) or [question mark](https://yaml.org/spec/1.2.2/#flow-mappings).
>

这段内容引用自 [https://yaml.org/spec/1.2.2/#22-structures](https://yaml.org/spec/1.2.2/#22-structures) 。意思是说问号和空格（“ `? `”）在 YAML 中可以用来表示一个复杂的 `mapping` 键。我举两个例子你就懂了。

第一个例子：

```yaml
? - Detroit Tigers
  - Chicago cubs
: - 2001-07-23
```

这是一个合法的 YAML 文档。

`? ` 标识它后面的内容是对象的 `key`，这个例子中对象的 `key` 如下：

```yaml
- Detroit Tigers
- Chicago cubs
```

`: ` 标识它后面的内容是对象的 `value`：

```yaml
- 2001-07-23
```

第二个例子：

```yaml
? [ New York Yankees,
    Atlanta Braves ]
: [ 2001-07-02, 2001-08-12,
    2001-08-14 ]
```

有了前一个示例的解释，这个示例相信你已经能够看懂了。它的 `key` 和 `value` 都是数组类型。

不过，很遗憾，这段复杂的 YAML 文档 Go 和 Python 都不支持。

不过在 JavaScript 中可以得到支持，你可以使用如下代码解析复杂 YAML：

```javascript
const yaml = require('js-yaml');

const yamlExample = `
? - Detroit Tigers
  - Chicago cubs
: - 2001-07-23

? [ New York Yankees,
    Atlanta Braves ]
: [ 2001-07-02, 2001-08-12,
    2001-08-14 ]
`;
try {
    const data = yaml.load(yamlExample);
    console.log(data);
} catch (e) {
    console.error(e);
}
```

得到的解析结果为：

```javascript
{
  'Detroit Tigers,Chicago cubs': [ 2001-07-23T00:00:00.000Z ],
  'New York Yankees,Atlanta Braves': [
    2001-07-02T00:00:00.000Z,
    2001-08-12T00:00:00.000Z,
    2001-08-14T00:00:00.000Z
  ]
}
```

可以发现，JavaScript 也是通过一种变通的方式，将对象的 `key` 全部转换成字符串来进行处理。

### JSON 规范

JSON 全称为 JavaScript Object Notation，即 JavaScript 对象表示法，是由美国程序员[道格拉斯·克罗克福特](https://zh.wikipedia.org/wiki/%E9%81%93%E6%A0%BC%E6%8B%89%E6%96%AF%C2%B7%E5%85%8B%E7%BE%85%E5%85%8B%E7%A6%8F%E7%89%B9)构想和设计的一种轻量级[资料交换格式](https://zh.wikipedia.org/wiki/%E6%95%B0%E6%8D%AE%E4%BA%A4%E6%8D%A2)。它是 JavaScript 的子集，也是市面上所有主流编程语言进行资料交换的事实标准。

[维基百科](https://zh.wikipedia.org/wiki/JSON)中对于 JSON 对象的描述如下：

> 对象：若干无序的“键-值对”(key-value pairs)，其中键只能是字符串[[2]](https://zh.wikipedia.org/wiki/JSON#cite_note-2)。建议但不强制要求对象中的键是独一无二的。对象以花括号`{}`包裹。多个键-值对之间使用逗号`,`分隔。键与值之间用冒号`:`分隔。
>

这其中，最值得注意的一句话是「**键只能是字符串**」，这一点需谨记。

### 总结

本文通过一个示例讲解了我在 Go 中进行 YAML to JSON 时所遇到的问题，并演示了如何解决问题。

此外，我还简单介绍了 YAML 和 JSON 规范，让你知其然并能知其所以然。这也是我写此篇文章的目的，这个问题很小，甚至有些不值一提，几分钟就能搞定，但它背后折射出的是我当时在写代码过程中对官方标准规范的忽视，还是倡导大家多阅读官方文档。

在我编程刚入门的时候，其实是恐惧看一些官方文档或规范的，觉得他们都写得晦涩难懂，但往往很多细节都隐藏在官方标准的规范里，这是我们绕不过去的门槛，没有捷径可以走。

最后，我还想强调一点，单元测试很重要，充分测试你的代码，有助于减少 Bug 的产生，虽然无法避免 :)。

好了，本文讲解就到这里，更多关于 YAML/JSON 规范的内容你可以在官网查看学习。

本文示例源码我都放在了 [GitHub](https://github.com/jianghushinian/blog-go-example/tree/main/yaml) 中，欢迎点击查看。

希望此文能对你有所启发。

**延伸阅读**

+ YAML Wiki：[https://zh.wikipedia.org/wiki/YAML](https://zh.wikipedia.org/wiki/YAML)
+ YAML 官网：[https://yaml.org/](https://yaml.org/)
+ YAML Specification：[https://github.com/yaml/yaml-spec](https://github.com/yaml/yaml-spec)
+ YAML Lint：[https://www.yamllint.com/](https://www.yamllint.com/)
+ JS-YAML demo. YAML JavaScript parser.：[https://nodeca.github.io/js-yaml/](https://nodeca.github.io/js-yaml/)
+ What is a complex mapping key in YAML?：[https://stackoverflow.com/questions/33987316/what-is-a-complex-mapping-key-in-yaml](https://stackoverflow.com/questions/33987316/what-is-a-complex-mapping-key-in-yaml)
+ JSON Wiki：[https://zh.wikipedia.org/wiki/JSON](https://zh.wikipedia.org/wiki/JSON)
+ 介绍 JSON：[https://www.json.org/json-zh.html](https://www.json.org/json-zh.html)
+ yaml.v3 GitHub 源码：[https://github.com/go-yaml/yaml](https://github.com/go-yaml/yaml)
+ go-yaml GitHub 源码：[https://github.com/goccy/go-yaml](https://github.com/goccy/go-yaml)
+ 本文 GitHub 示例代码：[https://github.com/jianghushinian/blog-go-example/tree/main/yaml](https://github.com/jianghushinian/blog-go-example/tree/main/yaml)
+ 本文永久地址：[https://jianghushinian.cn/2025/08/03/go-yaml-to-json](https://jianghushinian.cn/2025/08/03/go-yaml-to-json)

**联系我**

+ 公众号：[Go编程世界](https://jianghushinian.cn/about)
+ 微信：jianghushinian
+ 邮箱：[jianghushinian007@outlook.com](mailto:jianghushinian007@outlook.com)
+ 博客：[https://jianghushinian.cn](https://jianghushinian.cn)
+ GitHub：[https://github.com/jianghushinian](https://github.com/jianghushinian)
