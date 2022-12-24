---
title: JavaScript 立即执行函数
date: 2020-01-18 13:56:31
tags:
- JavaScript
categories:
- JavaScript
---

立即执行函数，顾名思义，即创建后被立刻执行的函数。在实际工作中，立即执行函数并不是必须用到的语法，不过使用立即执行函数有个好处是能够简化代码。

<!-- more -->

### 立即执行函数语法

在 JavaScript 中一个普通的函数定义语法如下：

```javascript
function foo() {
    console.log(123);
}
```

如果要调用这个函数，那么我们会执行：

```javascript
foo();
```

如果把它改成立即执行函数的写法是这样的：

```javascript
(function foo() {
    console.log(123);
}());
```

或者这样的：

```javascript
(function foo() {
    console.log(123);
})();
```

以上两个立即执行函数相当于 `函数声明` + `函数调用`。一个普通的函数声明需要使用 `function` 关键字，函数调用需要使用函数名加一个圆括号（`foo()`）。我们知道函数名实际上保存了函数的引用，所以 `foo` 变量保存了 `foo` 函数体的引用，因此理论上立即执行函数的写法可以是这样：

```javascript
function foo() {
    console.log(123);
}();
```

但是如果这样写，JavaScript 解释器会报 `SyntaxError` 语法错误。所以就需要用一个括号把整个函数括起来再进行调用。用来调用函数的括号可以写在把函数括起来的括号的里面或者外面，两种写法没有任何区别，选择一个你更能接收的写法即可。


### 立即执行函数特点

立即执行函数和普通函数几乎没什么差别，既能接收参数、也能有返回值，但它们之间还是有一个非常不一样的地方，就是立即执行函数执行完后就立刻被销毁。

我们上面定义了一个 `foo` 函数，如果想多次执行，那么只需要多次调用 `foo()` 即可：

```javascript
function foo() {
    console.log(123);
}

foo();
foo();
foo();
```

但是，如果是立即执行函数，多次调用 `foo()` 将得到异常。

```javascript
(function foo() {
    console.log(123);
})();

foo();  // ReferenceError: foo is not defined
```

可以看到在第二次调用 `foo()` 函数时，会得到 `foo is not defined`。即 `foo` 函数实际上已经被销毁了。所以我们在定义立即执行函数的时候，通常不需要指定函数名，定义一个匿名函数即可：

```javascript
(function () {
    console.log(123);
})();
```


### 立即执行函数使用场景

立即执行函数执行一次就被销毁，所以如果我们定义了一个函数只被执行一次的时候就可以将其换成立即执行函数。

立即执行函数比较常见的两个使用场景是：

1. 数据初始化
2. 解决闭包中引用循环变量的问题

对于数据初始化比较好理解，如想要计算 `num` 的值：

```javascript
var num = (function (a, b) {
    return a + b;
})(1, 2);
```

而对于如何解决闭包中引用循环变量的问题，我们需要看一个稍微复杂一点的例子：

```javascript
function foo() {
    var arr = [];
    for (var i = 0; i < 3; i++) {
        arr.push(function bar() {
            return i * i;
        });
    }
    return arr;
}

var result = foo();
result[0]();  // 9
result[1]();  // 9
result[2]();  // 9
```

以上示例代码中，闭包中的 `bar` 函数引用了其外部的循环变量 `i`，我们本来期待的结果是 `0`、`1`、`4`，但实际上最终结果都为 `9`。

出现此问题的原因是 `for` 循环不具备独立的块级作用域导致的，此时就可以使用立即执行函数来解决这个问题。

```javascript
function foo() {
    var arr = [];
    for (var i = 0; i < 3; i++) {
        arr.push((function (j) {
            return function bar() {
                return j * j;
            }
        })(i));
    }
    return arr;
}

var result = foo();
result[0]();  // 0
result[1]();  // 1
result[2]();  // 4
```

我们在 `bar` 函数的外部，定义了一个立即执行函数，这样，在每次循环时，当前循环变量 `i` 的值都被传递到了函数内部的 `j` 变量，所以能得到我们想要的结果。

其实还有一种更简单的解决方法，直接将声明循环变量 `i` 的关键字 `var` 改为 `let` 也能同样达到效果，读者可以自行尝试。


### 立即执行函数探索，谨慎尝试

我在上面举了一个使用立即执行函数初始化数据的例子：

```javascript
var num = (function (a, b) {
    return a + b;
})(1, 2);
```

其实这种写法不使用括号把函数包起来也是可以的：

```javascript
var num = function (a, b) {
    return a + b;
}(1, 2);
```

同样能得到正确的结果，并且 JavaScript 解释器并没有报错。

实际上，在执行赋值操作的时候，上面的代码已经变成了一个表达式，而 JavaScript 解释器认为 `函数表达式` 是可以直接被执行的。

所以下面的代码也能够被正确执行：

```javascript
+function () {
    console.log(123);
}();

-function () {
    console.log(123);
}();
```

注意这里的 `+`、`-` 代表正、负，并不是加、减。JavaScript 有相当多的这种比较鬼畜的写法，并不建议大家在实际工作中使用，仅供娱乐。
