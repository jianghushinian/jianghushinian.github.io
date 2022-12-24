---
title: Python 中的类方法和静态方法
date: 2021-10-13 12:24:14
tags:
- Python
categories:
- Python
---

Python 中常见的方法分三种：实例方法、类方法和静态方法。实例方法最为常见，也最容易理解，而另外两种方法对于很多 Python 编程新手来说就不那么容易理解了，也比较容易用错。本文将对类方法和静态方法进行讲解，希望能够加深读者对这两种方法的理解。

<!-- more -->

### 实例方法

首先我们来简单回顾下 Python 中实例方法的定义：

```python
class Hello(object):
    def say_hello(self, name):
        print("hello", name)
```

如果要调用 `Hello` 类中定义的 `say_hello` 实例方法，需要先实例化一个实例对象 `Hello()`，再进行调用：

```python
h = Hello()
h.say_hello("world")
# > hello world
```

### 类方法

回顾了实例方法，我们再来看下类方法如何使用。

类方法定义如下：

```python
class Hello(object):
    @classmethod
    def print_class_name(cls):
        print(cls.__name__)
```

对比实例方法，我们可以发现，类方法在定义上与实例方法有两处不同：首先在方法上面多了一个装饰器 `@classmethod` 标识这个方法为类方法，而不是实例方法；然后是方法参数由 `self` 变成了 `cls` 表示这个方法接收到的是一个类，而不是实例对象。

类方法的调用无需对类进行实例化，可以直接通过类来调用：

```python
Hello.print_class_name()
# > Hello
```

通过 `cls.__name__` 的方式我们打印了 `Hello` 类的类名。类方法中 `cls` 等价于 `Hello`，就像实例方法中 `self` 等价于实例对象 `Hello()` 一样。同 `self` 一样，`cls` 也是约定俗成的变量名，我们也可以不写成 `cls` 而写成任何我们喜欢的变量名。

类方法其实也可以通过实例对象进行调用：

```python
h = Hello()
h.print_class_name()
# > Hello
```

得到的结果是一样的，其实当我们用实例对象调用类方法的时候，Python 解释器会自动帮我们获取到实例对象所属的类，然后将其当作参数传给类方法。

现在我们了解了类方法的语法，那么什么时候需要用到类方法呢？

类方法最常用的两个场景一个是类的继承，另一个是构造方法。

#### 类的继承中使用类方法

首先来看如何在类继承中使用 `cls` 来做子类的名称绑定。

```python
class Animal(object):
    def __init__(self, name):
        self.name = name

    @classmethod
    def shout(cls):
        print(f"{cls.__name__} shout")


class Cat(Animal):
    pass


class Dog(Animal):
    pass


cat = Cat("cat")
cat.shout()
# > Cat shout

dog = Dog("dog")
dog.shout()
# > Dog shout
```

这里我们定义一个动物类 `Animal`，其有一个 `shout` 方法用来打印哪种动物在叫。`Cat` 和 `Dog` 类都继承自 `Animal` 类，当通过 `Cat` 类或 `Dog` 类的实例对象调用 `shout` 类方法时，就会分别打印出 `Cat shout` 和 `Dog shout`。这样我们就实现了只需在父类中定义一个类方法，便可以在子类调用父类方法的时候根据类型的不同自动获取子类的类型名称。如果不使用类方法来定义，我们可能会写出如下代码：

```python
class Animal(object):
    def __init__(self, name):
        self.name = name

    def shout(self):
        print("Animal shout")


class Cat(Animal):
    def shout(self):
        print("Cat shout")


class Dog(Animal):
    def shout(self):
        print("Dog shout")


cat = Cat("cat")
cat.shout()
# > Cat shout

dog = Dog("dog")
dog.shout()
# > Dog shout
```

虽然同样能实现功能，但这种实现大大增加了工作量，一是子类需要重写父类的方法，二是将类名 `Cat`、`Dog` 都进行了硬编码，这种写法极不推荐。

我的开源项目 `Todo List 教程` 中 [`models`](https://github.com/jianghushinian/todo-list/tree/master/chapter8/todo_list/todo/models) 部分就使用了这种思想来实现模型代码的复用，感兴趣的同学可以了深入解一下。

#### 使用类方法实现构造方法

我们定义一个 `Poerson` 类，它的 `__init__` 方法接收两个参数 `first` 和 `last` 分别用来接收用户的 `first_name` 和 `last_name`，它还有一个 `name` 方法用来获取用户的全名。

```python
class Person(object):
    def __init__(self, first, last):
        self.first_name = first
        self.last_name = last

    def name(self):
        return self.first_name + self.last_name

p = Person("江湖", "十年")
print(p.name())
# > 江湖十年
```

假设我们现在有一个爬虫程序，能够在网络上的某个网站中爬取到很多的用户名，这些用户名可能是 `江湖-十年`、`江湖_十年` 或者 `江湖十年` 这三种形式。每一个用户名我们都需要实例化一个 `Person` 对象，由于用户名有三种形式，我们可以考虑使用一个函数来处理不同用户名，然后将处理后的用户名传给 `Person` 类来生成 `Person` 对象，可以写出如下代码：

```python
def generate_person(name):
    if "-" in name:
        first, last = name.split("-")
    elif "_" in name:
        first, last = name.split("_")
    else:
        first, last = name, ""
    return Person(first, last)
  
p1 = generate_person("江湖-十年")
p2 = generate_person("江湖_十年")
p3 = generate_person("江湖十年")
print(p1.name())
# > 江湖十年
print(p2.name())
# > 江湖十年
print(p3.name())
# > 江湖十年
```

这样，不管我们爬取到的用户名是哪种形式，都可以通过 `generate_person` 函数先预处理一下获取到 `first_name` 和 `last_name`，然后实例化 `Person` 对象进行返回。

其实有一种更好的方式对用户名进行预处理，那就是使用类方法：

```python
class Person(object):
    def __init__(self, first, last):
        self.first_name = first
        self.last_name = last

    def name(self):
        return self.first_name + self.last_name

    @classmethod
    def new(cls, name):
        if "-" in name:
            first, last = name.split("-")
        elif "_" in name:
            first, last = name.split("_")
        else:
            first, last = name, ""
        return cls(first, last)


p1 = Person.new("江湖-十年")
p2 = Person.new("江湖_十年")
p3 = Person.new("江湖十年")
print(p1.name())
# > 江湖十年
print(p2.name())
# > 江湖十年
print(p3.name())
# > 江湖十年
```

我们将 `generate_person` 函数移动到 `Person` 类内部作为一个类方法，并将其重命名为 `new`，最后通过 `return` 一个 `cls` 实例来返回 `Person` 实例对象。

熟悉 `Java` 等静态语言的同学应该很容易想到，这个类方法 `new` 实际上就是其他编程语言中的 `构造函数`。所以我们也就通过类方法的形式，在 Python 中实现了构造方法（函数）。

相比于使用 `generate_person` 函数生成 `Person` 实例对象，使用 `new` 构造方法的好处是代码封装性更强，也更容易理解。构造函数本就应该属于类的一部分，而不是单独写一个函数来实现。

爬虫框架 `Scrapy` 的 `Item Pipeline` 中有一个叫 [`from_crawler`](https://docs.scrapy.org/en/latest/topics/item-pipeline.html#from_crawler) 的类方法，实际上就是 `Scrapy` 提供给我们的一个构造方法。

### 静态方法

接下来我们一起看下静态方法在 Python 中的应用。

静态方法定义如下：

```python
from datetime import datetime

class Hello(object):
    def __init__(self, name):
        self.name = name

    def say_hello(self, create_time):
        print(f"hello {self.name} - {self.process_datetime(create_time)}.")

    @staticmethod
    def process_datetime(dt):
        return dt.strftime("%Y-%m-%d %H:%S:%M")

h = Hello("tim")
h.say_hello(datetime.now())
# > hello tim - 2099-01-07 13:14:15.
```

`Hello` 类的 `process_datetime` 方法即为一个静态方法，可以看到静态方法的定义跟一个普通函数区别不大，只是在上面多了一个 `@staticmethod` 装饰器。这个函数作用是对日期时间类型做格式化操作，返回我们想要的 `str` 类型的日期时间，在调用 `say_hello` 时可以通过参数传递进来一个 `create_time`，我们想把它打印到 `hello name` 问候语句的后面，于是调用了 `process_datetime` 静态方法来对日期时间进行格式化处理。

这个示例代码比较简单，所以优势不是非常明显，但假设我们的 `process_datetime` 方法有大量逻辑要处理，这样单独提取出来一个静态方法，会比把所有逻辑写在 `say_hello` 方法内部更加合理，因为这样的话代码可读性会更高，并且可复用。

对比来看，相较于类方法来说，静态方法更加简单，使用场景也更少。

`Django` 框架源码中有多处用到了 `staticmethod`，比如在 `models` 中 `RegisterLookupMixin` 类下有一个 [`merge_dicts`](https://github.com/django/django/blob/main/django/db/models/query_utils.py#L165) 方法作用就使用了静态方法将多个 `dict` 进行合并操作，这个操作不涉及类属性或实例属性的引用，所以使用静态方法比较合适。

以上，我介绍了 Python 中类方法和静态方法的定义和使用，希望你在写代码的时候能够更加得心应手，我只对常见用法做了总结，更多用法还是需要你在实践中更加深入的体会。同时我也在每个示例讲解的下面贴上了开源项目中对不同方法的使用，通过阅读优秀开源项目的源码，也能够加深我们对“最佳实践”的理解。

