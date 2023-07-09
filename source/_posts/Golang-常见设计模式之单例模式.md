---
title: Go 常见设计模式之单例模式
date: 2022-03-04 20:07:42
tags:
- Go
- 设计模式
categories:
- Go
---

单例模式是设计模式中最简单的一种模式，单例模式能够确保无论对象被实例化多少次，全局都只有一个实例存在。根据单例模式的特性，我们可以将其应用到全局唯一性配置、数据库连接对象、文件访问对象等。Go 语言有多种方式可以实现单例模式，我们今天就来一起学习下。

<!-- more -->

### 饿汉式

饿汉式实现单例模式非常简单，直接看代码：

```go
package singleton

type singleton struct{}

var instance = &singleton{}

func GetSingleton() *singleton {
	return instance
}
```

`singleton` 包在被导入时会自动初始化 `instance` 实例，使用时通过调用 `singleton.GetSingleton()` 函数即可获得 `singleton` 这个结构体的单例对象。

由于单例对象是在包加载时立即被创建出来，所以也就有了这个形象的名称叫作饿汉式。与之对应的另一种实现方式叫作懒汉式，当实例被第一次使用时才会被创建。

需要注意的是，尽管饿汉式实现单例模式如此简单，但大多数情况下仍不被推荐使用，因为如果单例实例化时初始化内容过多，可能造成程序加载用时较长。

### 懒汉式

接下来我们再来看下如何通过懒汉式实现单例模式：

```go
package singleton

type singleton struct{}

var instance *singleton

func GetSingleton() *singleton {
	if instance == nil {
		instance = &singleton{}
	}
	return instance
}
```

相较于饿汉式的实现，我们把实例化 `singleton` 结构体部分的代码移到了 `GetSingleton()` 函数内部。这样一来，就将对象实例化的步骤延迟到了 `GetSingleton()` 被第一次调用时。

通过 `instance == nil` 的判断来实现单例并不十分可靠，当有多个 `goroutine` 同时调用 `GetSingleton()` 时无法保证并发安全。

### 支持并发的单例

如果你用 Go 语言写过并发编程，那么应该可以很快想到解决懒汉式单例模式并发安全问题的方案：

```go
package singleton

import "sync"

type singleton struct{}

var instance *singleton

var mu sync.Mutex

func GetSingleton() *singleton {
	mu.Lock()
	defer mu.Unlock()

	if instance == nil {
		instance = &singleton{}
	}
	return instance
}
```

我们对代码的主要修改就是在 `GetSingleton()` 函数最开始加了如下两行代码：

```go
mu.Lock()
defer mu.Unlock()
```

通过加锁的机制，就可以保证这个实现单例模式的函数是并发安全的。

不过这样也有些问题，因为用了锁机制，每次调用 `GetSingleton()` 时程序都会进行加锁、解锁的步骤，这样会导致程序性能的下降。

### 双重锁定

加锁导致程序性能下降，但我们又不得不用锁来保证程序的并发安全，于是有人想出了双重锁定（`Double-Check Locking`）的方案：

```go
package singleton

import "sync"

type singleton struct{}

var instance *singleton

var mu sync.Mutex

func GetSingleton() *singleton {
	if instance == nil {
		mu.Lock()
		defer mu.Unlock()

		if instance == nil {
			instance = &singleton{}
		}
	}
	return instance
}
```

可以看到，所谓的双重锁定实际上就是在程序加锁前又加了一层 `instance == nil` 判断，这样就兼顾了性能和安全两个方面。

不过这段代码看起来有些奇怪，既然外层已经判断了 `instance == nil`，加锁后却又进行了第二次 `instance == nil`  判断。其实外层的 `instance == nil` 判断是为了提高程序的执行效率，因为如果 `instance` 已经存在，则无需进入 `if` 逻辑，程序直接返回 `instance` 即可。这样就免去了原来每次调用 `GetSingleton()` 都上锁的操作，将加锁的粒度更加精细化。而内层的 `instance == nil`  判断则是考虑了并发安全，在极端情况下，多个 `goroutine` 同时走到了加锁这一步，内层判断就起到作用了。

### Gopher 惯用方案

虽然我们通过双重锁定机制兼顾和性能和并发安全，但代码有些丑陋，不符合广大 Gopher 的期待。好在 Go 语言在 `sync` 包中提供了 `Once` 机制能够帮助我们写出更加优雅的代码：

```go
package singleton

import "sync"

type singleton struct{}

var instance *singleton

var once sync.Once

func GetSingleton() *singleton {
	once.Do(func() {
		instance = &singleton{}
	})
	return instance
}
```

`Once` 是一个结构体，在执行 `Do` 方法的内部通过 `atomic` 操作和加锁机制来保证并发安全，且 `once.Do` 能够保证多个 `goroutine` 同时执行时 `&singleton{}` 只被创建一次。

其实 `Once` 并不神秘，其内部实现跟上面使用的双重锁定机制非常类似，只不过把 `instance == nil` 换成了 `atomic` 操作，感兴趣的同学可以查看下其对应源码。

### 总结

以上就是 Go 语言中实现单例模式的几种常用套路，经过对比可以得出结论，最推荐的方式是使用 `once.Do` 来实现，`sync.Once` 包帮我们隐藏了部分细节，却可以让代码可读性得到很大提升。

