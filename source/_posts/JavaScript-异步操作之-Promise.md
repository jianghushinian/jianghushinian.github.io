---
title: JavaScript 异步操作之 Promise
date: 2019-08-24 19:16:43
tags:
- JavaScript
- 异步编程
categories:
- JavaScript
---

由于 JavaScript 通常是由单线程来执行代码，所以在编写 JavaScript 代码时经常需要使用异步操作来提高程序性能。一般来说异步执行在 JavaScript 中使用 `回调函数` 的形式来实现。不过近年来由于社区的推动，`Promise` 已经成为 JavaScript 异步编程的一个标准，使用 `Promise` 进行异步编程，代码的可维护性将有很大提升，尤其是使用 `Promise` 取代多层 `回调函数` 嵌套的问题。

<!-- more -->

### `Promise` 简介

下面是构造一个最简单的 `Promise` 代码示例：

```javascript
// 构造 Promise
const promise = new Promise(function (resolve, reject) {
    // 异步操作代码
    if (异步操作成功) {
        resolve(this.response);
    } else {
        reject(this.status);
    }
});

// promise 状态改变时的回调
promise.then(function (res) {
    console.log(res);  // 执行成功回调
}, function (error) {
    console.log(error);  // 执行失败回调
})
```

`Promise` 是 JavaScript 提供的一个对象，可以通过 `new` 关键字构造这个对象。`Promise` 构造函数接收一个函数作为参数，该函数接收另外两个函数作为参数，分别是 `resolve`、`reject`，这两个函数由 JavaScript 引擎自身提供，无需我们手动创建。

`Promise` 对象有三种状态，分别是：`pending`(进行中)、`fulfilled`(已成功)、`rejected`(已失败)。有两种状态改变，一种是从 `pending` 变为 `fulfilled`，另一种是从 `pending` 变为 `rejected`，一旦状态改变，状态就会凝固，无法被再次更改。而 `resolve` 的作用就是将 `Promise` 对象的状态从 `pending` 变为 `fulfilled`，`reject` 的作用是将 `Promise` 对象的状态从 `pending` 变为 `rejected`。

通常我们在传递给 `Promise` 构造函数的函数顶部编写一些异步代码，例如一个 `AJAX` 请求，然后当异步代码执行成功的时候，调用 `resolve`，`resolve` 是一个函数，它能够接收参数，所以我们调用它时可以将异步代码的执行结果传递给它，此时 `resolve` 函数有两个作用，一是改变 `Promise` 的状态为 `fulfilled`，二是它能够将传递进去的参数传递到 `promise` 实例的 `then` 方法的第一个回调函数中，待后续操作使用。异步代码执行失败时调用 `reject` 函数，此时 `reject` 函数同样有两个作用，一是改变 `Promise` 的状态为 `rejected`，二是将传递进去的参数传递到 `promise` 实例的 `then` 方法的第二个回调函数中。

当 `Promise` 状态一旦改变，我们就可以通过 `Promise` 对象的实例（`promise` 变量）的 `then` 方法获取到异步操作的结果。`then` 方法接收两个回调函数作为参数，第一个回调函数会在 `Promise` 状态变为 `fulfilled` 的时候自动调用，第二个回调函数会在 `Promise` 状态变为 `rejected` 的时候自动调用。这两个回调函数分别接收一个参数，第一个回调函数接收到的 `res` 参数实际上就是我们在 `Promise` 内部异步代码执行成功时传递给 `resolve` 的参数，第二个回调函数接收到的 `error` 参数是在 `Promise` 内部异步代码执行失败时传递给 `reject` 的参数。

以上就是 `Promise` 对象的执行流程，接下来我们就用一个示例来展示 `Promise` 在实际编写代码中如何应用。

先来看一段用 `回调函数` 的写法发送 `AJAX` 请求的代码示例：

```javascript
function handleResponse() {
    if (this.readyState === 4) {
        if (this.status === 200) {
            console.log(this.response);
        } else {
            console.log(this.status);
        }
    }
}

const request = new XMLHttpRequest();
request.open('GET', 'http://httpbin.org/get');
request.onreadystatechange = handleResponse;
request.send();
```

在编写 Javascript 时这种代码非常常见，那么如何用 `Promise` 的写法来编写上面这段代码呢？请看下面的示例：

```javascript
const promise = new Promise(function (resolve, reject) {
    function handleResponse() {
        if (this.readyState === 4) {
            if (this.status === 200) {
                resolve(this.response);  // 请求成功时调用
            } else {
                reject(this.status);  // 请求失败时调用
            }
        }
    }
    const request = new XMLHttpRequest();
    request.open('GET', 'http://httpbin.org/get');
    request.onreadystatechange = handleResponse;
    request.send();
})

promise.then(function (res) {
    console.log(res);  // 成功回调
}, function (error) {
    console.log(error);  // 失败回调
})
```

以上就是通过 `Promise` 来处理 `AJAX` 的写法。乍一看，觉得代码是不是变多了，也更麻烦了？事实上，的确如此，如果仅仅从发送一个简单的 `AJAX` 请求代码来看我们确实没必要采用 `Promise` 的写法，这样做只会让代码看起来更加复杂。

### `Promise` 能够解决什么问题

那么 `Promise` 的适用场景到底是什么，能够解决什么问题呢？

1. `Promise` 非常适合解决 `回调地狱` 问题。
2. `Promise` 解决了回调函数中不能够使用 `return` 的问题。

我们来看看 `Promise` 是如何解决这两个问题的。假设我们需要调用一个获取用户发表的博客列表的接口，而这个接口需要登录才能够调用。也就是说我们调用接口的顺序必须是：先调用登录认证接口进行登录，然后再去调用获取用户博客列表的接口。

如果使用 `jQuery` 发送 `AJAX` 请求的 `回调函数` 写法，我们有可能会写出这样的代码：

```javascript
// 发送第一个 AJAX 请求进行登录
$.ajax({
    type: 'GET',
    url: 'http://localhost:3000/login',
    success: function (res) {
        // 登录成功回调函数内部，发送第二个 AJAX 请求获取数据
        $.ajax({
            type: 'GET',
            url: 'http://localhost:3000/blog/list',
            success: function (res) {
                console.log(res);
            },
            error: function (xhr) {
                console.log(xhr.status);
            }
        });
    },
    error: function (xhr) {
        console.log(xhr.status);
    }
});
```

可以看到，使用 `回调函数` 形式写出的代码是嵌套形式的。也就是说，每多一次请求，我们就要在 `success` 回调函数内部再写一层嵌套。如果嵌套层次过多，那么就会出现 JavaScript 令人头疼的问题 —— `回调地狱`。嵌套层次越多，代码就越难以理解，如果出现异常，那么调试将会是一个非常痛苦的过程。

我们尝试着用 `Promise` 来解决上面的问题：

```javascript
const getResponse = function (url) {
    const promise = new Promise(function (resolve, reject) {
        $.ajax({
            type: 'GET',
            url: url,
            success: function (res) {
                resolve(res);
            },
            error: function (xhr) {
                reject(xhr);
            }
        });
    });
    return promise;
}

getResponse('http://localhost:3000/login').then(function (res) {
    getResponse('http://localhost:3000/blog/list').then(function (res) {
        console.log(res);
    }, function (xhr) {
        console.log(xhr.status);
    })
}, function (xhr) {
    console.log(xhr.status);
});
```

以上是使用 `Promise` 进行重构的代码，但仔细观察你会发现：上面的代码依然在用 `回调` 的思想来写 `Promise` 代码。在第一次调用 `getResponse` 函数请求 `http://localhost:3000/login` 成功后，代码执行 `then` 方法，在 `then` 方法的内部，再一次调用了 `getResponse` 函数，这一次请求 `http://localhost:3000/blog/list` 地址，成功后执行下一个 `then` 方法，在 `then` 方法内部就能够成功获取用户博客列表的接口返回结果。

其实上面这段代码同样存在 `回调地狱` 的问题，因为如果请求次数过多，我们同样需要在一层层的嵌套函数中调用 `getResponse`。这无疑没有摆脱 `回调地狱`。

来看正确使用 `Promise` 对象解决 `回调地狱` 的示例：

```javascript
const getResponse = function (url) {
    const promise = new Promise(function (resolve, reject) {
        $.ajax({
            type: 'GET',
            url: url,
            success: function (res) {
                resolve(res);
            },
            error: function (xhr) {
                reject(xhr.status);
            }
        });
    });
    return promise;
}

getResponse('http://localhost:3000/login').then(function (res) {
    return getResponse('http://localhost:3000/blog/list');
}, function (xhr) {
    console.log(xhr.status);
}).then(function (res) {
    console.log(res);
}, function (xhr) {
    console.log(xhr.status);
});
```

在 `Promise` 接口中，`then` 方法会返回一个新的 `Promise` 实例，所以我们能够采用链式调用的写法，`then` 方法后面再接另一个 `then` 方法，理论上可以无限的接下去。

注意第一个 `then` 方法的内部，在调用 `getResponse('http://localhost:3000/blog/list')` 的前面加上了 `return`。正是这个 `return` 的存在，才使得不使用层层嵌套的写法成为可能，得以使代码展平。`Promise` 实例后面的 `then` 方法会依次执行，前一次 `then` 方法内部的 `return` 结果，会当作参数传入下一个 `then` 方法。如果使用回调函数的写法，因为其内部没有办法 `return`，所以你只能无限的增加嵌套来解决问题。

这段代码才能体现出 `Promise` 对象的真正威力，它能够把层层嵌套的代码全部展平。每多一次请求，就在后面再增加一个 `then`方法的调用，这样写出来的代码明显要直观很多，你不需要再写出多层嵌套的代码。

### `Promise` 的 `then` 方法和 `catch` 方法

通过前面的介绍我们知道 `Promise` 的 `then` 方法接收两个回调函数作为参数。实际上，它的第二个参数并不是必填的，如果你不关心异常，那么就可以只写第一个回调函数。

`Promise` 还提供了 `catch` 方法专门用来接收异常，写法如下：

```javascript
getResponse('http://localhost:3000/login').then(function (res) {
    return getResponse('http://localhost:3000/blog/list');
}).catch(function (xhr) {
    console.log(xhr.status);
});
```

用上面这种写法来处理异常更符合直觉，看起来也更加清晰，不用在 `then` 方法内部传入第二个回调函数去处理异常，而是单独在 `catch` 方法内部进行处理。事实上 `.catch(callback)` 方法等价于 `.then(undefined, callback)`。

### `Promise` 的 `all` 方法和 `race` 方法

`Promise` 的 `all` 方法可以将多个 `Promise` 实例，合并为一个 `Promise` 实例。

这有什么用？假设我们需要等待多个异步操作都执行完才能执行下一步逻辑，此时就是使用 `all` 方法的绝佳时机。这种情况使用 `回调函数` 的形式不太好解决，而使用 `Promise` 就非常简单。

我们通过请求三次 `http://httpbin.org/get` 来模拟需要等待的三个异步操作，写出的代码如下：

```javascript
const p1 = getResponse('http://httpbin.org/get');
const p2 = getResponse('http://httpbin.org/get');
const p3 = getResponse('http://httpbin.org/get');

const p = Promise.all([p1, p2, p3]);
p.then(res => {
    console.log(res);
}).catch(err => {
    console.log(err);
});
```
`all` 方法接收一个数组作为参数，数组每一项都是一个 `Promise` 对象，`Promise` 实例 `p` 就是合并后的对象，`p1`、`p2`、`p3` 的状态决定了 `p` 的最终状态。

只有 `p1`、`p2`、`p3` 的状态同时都变为 `fulfilled` 时，`p` 的状态才会变为 `fulfilled`。只要 `p1`、`p2`、`p3` 的状态有一个变为 `rejected`，`p` 的状态就会变为 `rejected`。当 `p` 的状态变为 `fulfilled`时，`p1`、`p2`、`p3` 的返回结果会组成一个列表，自动传递给 `p` 的 `then` 方法的回调函数。并且这个列表的顺序由传入 `all` 方法的数组中 `p1`、`p2`、`p3` 的顺序决定。

当 `p1`、`p2`、`p3` 任意一个状态变为 `rejected` 时，只有其没有自己的 `catch` 方法时才会调用 `all` 方法后面的 `catch` 方法。否则只会调用其自己的 `catch` 方法。

至于 `race` 方法的用法与 `all` 方法完全相同，其区别是：只要 `p1`、`p2`、`p3` 的状态有一个变为 `fulfilled` 时，`p` 的状态就会变为 `fulfilled`。这也是 `race` 这个单词所代表的含义 `竞争`，只要 `p1`、`p2`、`p3` 其中有一个对象率先改变了状态，那么 `p` 的状态也就随之改变并且凝固。其他两个对象内部的代码还是会执行，但其结果已经没有用了。

由于使用 `race` 的代码只需要将 `Promise.all([p1, p2, p3])` 换成 `Promise.race([p1, p2, p3])` 即可，所以这里不再给出代码示例。

以上就是关于 `Promise` 使用方法的介绍，事实上 `Promise` 不止提供了这些接口，不过其他接口很少会用到，所以这里不再过多介绍，等你熟悉了它的用法后可以再进一步去学习。
