---
title: Python 闭包
date: 2019-10-21 20:32:23
tags:
- Python
categories:
- Python
---

`闭包`（closure）作为一个不太容易理解的概念出现在很多主流编程语言中，Python 中很多高级实现都离不开 `闭包`，`装饰器` 就是使用 `闭包` 的典型例子。

<!-- more -->

### 作用域

要学习 `闭包`，先了解 `作用域` 的概念。

`作用域` 是程序运行时变量的存在范围。常见 `作用域` 有 `全局作用域` 和 `局部作用域`，定义在`全局作用域`中的变量，在程序运行过程中的任何地方都可以访问到；而定义在函数内部的变量只有在函数内部才能够访问，函数内部的作用域就是`局部作用域`，为了便于理解，我这里称它为`函数作用域`。

`全局作用域`不可以读取 `函数作用域` 中的局部变量：

```python
def foo():
    num = 100

print(num)  # NameError: name 'num' is not defined
```

`函数作用域` 可以向上读取 `全局作用域` 中的全局变量：

```python
num = 100

def foo():
    print(num)  # 100

foo()
```

### 闭包

在 Python 中，函数是 `一等公民`，意思是说你可以像看待 `int`、`dict` 等常见的数据类型一样去看待它。其实函数不仅仅可以定义在 `全局作用域` 中，函数也可以定义在另一个 `函数作用域` 中。

```python
def foo():
    num = 100

    def bar():
        print(num)  # 100

    bar()

foo()
```

以上代码中，在 `foo` 函数内部又定义了一个 `bar` 函数。`bar` 函数可以向上读取 `foo` 函数内定义的任何变量。`bar` 函数读取变量 `num` 的顺序是：先在 `bar` 函数自己的 `函数作用域` 内查找，如果找不到就向上查找 `foo` 函数的 `作用域`，如果还是找不到，最后再向上查找 `全局作用域`。

以上代码稍作修改，就能成为一个 `闭包` 函数：

```python
def foo():
    num = 100

    def bar():
        print(num)  # 100

    return bar

result = foo()
result()
```

可以看到，只需要将之前的示例代码中 `bar()` 这句函数调用，改为 `return bar`，然后在调用 `foo` 函数时用一个变量 `result` 接收 `foo` 函数的返回值，最后再调用 `result` 这个变量，就能得到同样的结果。这样其实间接的的实现了在 `全局作用域` 中 `读取函数作用域` 中的变量。这就是 `闭包` 的威力。

像这样，函数嵌套函数，内部函数引用外部函数 `作用域` 中的局部变量，外部函数将内部函数的引用当作返回值返回，此时，我们就可以将内部函数和它引用的外部函数 `作用域` 中的局部变量统称为 `闭包`。

### 闭包的应用

假如需要实现一个计数器，我们很容易想到使用 `类` 来实现：

```python
class Counter(object):
    def __init__(self):
        self.num = 0

    def __call__(self):
        self.num += 1
        return self.num

counter = Counter()
print(counter())  # 1
print(counter())  # 2
print(counter())  # 3
```

由于 `类` 可以保存数据并且操作数据，所以很轻松就能够使用 `类` 来实现计数器。

而函数本身没法在每次调用时保存数据，所以无法实现一个计数器的功能。但当我们有了 `闭包` 函数，就能够用 `函数` 的形式来实现计数器了。

```python
def make_counter():
    num = 0

    def counter():
        nonlocal num
        num += 1
        return num

    return counter

counter = make_counter()
print(counter())  # 1
print(counter())  # 2
print(counter())  # 3
```

以上，我们就用 `闭包函数` 实现了一个计数器。

可以看到内部 `counter` 函数有一条语句 `nonlocal num`，关键字 `nonlocal` 的作用可以对应 `global` 关键字来理解。当我们在 `函数作用域` 内部修改 `全局作用域` 中的 `不可变类型` 变量时，我们会使用 `global` 关键字来标明一个变量是全局变量，同样的 `nonlocal` 关键字的作用是标明 `num` 是一个 `闭包` 中的变量，它有一个专业的术语叫做 `自由变量`。通常来说，一个函数执行完成以后，其内部的变量会随之被销毁，而 `自由变量` `num` 并不会被立即销毁，它随同 `counter` 函数一起组成了 `闭包`。

### 闭包的陷阱

下面是一个使用 `闭包` 时常见的误区：

```python
def func():
    li = []
    for i in range(3):
        def f():
            return i

        li.append(f)
    return li

li = func()
for f in li:
    print(f())
   
# 运行结果
# 2
# 2
# 2
```

我们期待的打印结果依次为 `0`、`1`、`2`，但实际上运行以上代码，得到的结果确是 `2`、`2`、`2`。

这是因为，内部函数 `f` 和循环变量 `i` 形成了 `闭包`。`func` 函数内部共产生了 3 个 `f` 函数，它们依次被追加到 `li` 列表中，但是并没有立即被执行，而其内部引用了 `自由变量` `i`，在 3 次 `for` 循环执行完成后，`i` 的值已经变成了 `2`，等到在函数外部接收到 `li` 再一次循环执行每个 `f` 函数时，所打印的 `i` 的值自然就都为 `2` 了。

要解决这个问题，需要想办法在每次循环时，让内部的 `闭包` 函数 `f` 绑定住当前的循环变量，而不使其跟随 `i` 的变动而改变：

```python
def func():
    li = []
    for i in range(3):
        def wrap(j):
            def f():
                return j

            return f

        li.append(wrap(i))
    return li

li = func()
for f in li:
    print(f())

# 运行结果
# 0
# 1
# 2
```

在这里，我在内部函数 `f` 的外部又嵌套了一层函数 `wrap`，在执行 `li.append(wrap(i))` 语句时，将循环变量 `i` 的值传递给 `wrap` 函数，这样 `wrap` 函数执行完成后，函数 `f` 和变量 `j` 就组成了 `闭包`。每次 `for` 循环时，都将循环变量 `i` 当作参数传递给 `wrap` 函数，由于每产生的 `wrap` 函数相互独立并且绑定到函数参数上的变量值不会改变，这样就能保证每次循环过程中在 `wrap` 函数内部保存的变量 `j` 互相独立，所以最终能够得到预期的结果。
