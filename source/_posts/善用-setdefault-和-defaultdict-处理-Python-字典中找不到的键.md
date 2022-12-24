---
title: 善用 setdefault 和 defaultdict 处理 Python 字典中找不到的键
date: 2019-09-22 15:04:37
tags:
- Python
- Pythonic
categories:
- Python
---

当以 `dict[key]` 的形式获取字典中某个键的值的时候，如果 `key` 不在字典中，我们将得到一个 `KeyError` 异常。由于处理异常需要写额外的代码，所以我们往往更多的使用 `dict.get(key, defalut)` 的形式来获取字典中某个键的值。这种情况下如果 `key` 不在这个 `dict` 中，也会返回一个指定的默认值 `default`，而不至于抛出异常。

<!-- more -->

### 使用 `setdefault`

在实际工作中，我们从数据库中查询的数据有可能是下面这种形式。

```python
data = [('one', 1), ('one', 2), ('two', 1), ('two', 2,), ('three', 1)]
```

现在需要将其转换为字典，并根据列表中每个元组的第一个元素当作键来进行分组，将所有键相同的值存放到列表中，结果如下。

```python
d = {
    'one': [1, 2],
    'two': [1, 2],
    'three': [1]
}
```

将列表转换为字典的过程，我们有可能写出这样的代码：

```python
d = {}

for key, value in data:
    value_list = d.get(key, [])  # 如果 key 不存在，返回一个空列表
    value_list.append(value)
    d[key] = value_list
```

这段代码逻辑比较简单，在 `for` 循环中，每次获取到对应的 `key` 的值，如果不存在就初始化一个空列表，然后将新的值追加到列表中，最后重新赋值给字典对应的 `key`。

这样虽然能够实现功能，但在 Python 中实现它我们还有更好的办法，即使用字典的 `setdefault` 方法为不存在的 `key` 赋予默认值。

```python
d = {}

for key, value in data:
    d.setdefault(key, []).append(value)
```

如你所见，之前用三行代码才能解决的问题，使用 `setdefault` 后只需要一行代码即可搞定。

事实上，使用 `setdefault` 不止在于代码更加的简洁，它还能提高程序执行效率。当调用 `d.setdefault(key, [])` 时，如果在 `d` 中查询不到 `key`，`setdefault` 会直接创建一个空列表并赋值给 `d[key]`，最后返回这个新创建的空列表，所以能够在后续直接调用列表的 `append` 方法对其进行更新操作。这个过程只会在 `d` 中查询一次 `key` 这个键，而之前的做法中则需要查询两次。

### 使用 `defaultdict`

`defaultdict` 是 Python 标准库 `collections` 下提供的一个能够以 `Pythonic` 的方式解决上面问题的另一种实现方案。

`defaultdict` 跟普通的 `dict` 用法几乎一样，只不过需要在其构造方法中提供一个可调用对象作为参数。当在字典中查询某个 `key` 不存在时，这个可调用对象的返回结果即为这个 `key` 的默认值。

以下是使用 `defaultdict` 来解决上面问题的代码示例：

```python
from collections import defaultdict

d = defaultdict(list)  # 用 list 的构造方法来创建默认值

for key, value in data:
    d[key].append(value)
```

注意：在实例化 `defaultdict` 时，如果没有传入参数，字典仍能够被正常创建，但当遇到不存在的 `key` 时就会抛出 `KeyError` 异常。
