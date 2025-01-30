---
title: 用 Go 语言实现刘谦 2024 春晚魔术，还原尼格买提汗流浃背的尴尬瞬间!
date: 2024-02-10 19:40:46
tags:
- Go
- 魔术
categories:
- Go
---

龙年春晚如期而至，想必大家都看（~~没~~）完（~~眼~~）了（~~看~~）吧，今年春晚最搞笑节目，当属刘谦的魔术：《守岁共此时》，不过笑点不是节目本身，而是本场魔术的 ~~托儿~~（主持人）尼格买提。

小尼：已经开始流汗了 😅。

本文将带大家一起用 Go 语言来还原下整个魔术的过程。

<!-- more -->

在开始写代码前，翻车画面必须置顶 🤣：

<div style="text-align: center"><img src="0.png" width="100%" style="max-width: 400px;"></div>

### 魔术：《守岁共此时》

这是一个公式魔术，即有着固定的套路，从数学角度来看是一个「约瑟夫问题」（严格证明可以在网上搜到，如果你感兴趣可以去看下）。

不过，咱们今天不讲枯燥的数学公式，固定套路是程序的强项，今天就用 Go 语言实现这个魔术的套路。

接下来，咱们直奔主题，回忆每一个步骤的同时，用程序将其实现。

#### 步骤

##### 一、 准备 4 张扑克牌，随意打乱，完成洗牌。

![步骤一](1.png)

代码实现如下：

```go
// 我们使用 `int` 类型的切片 `pokers` 来存储扑克牌
var pokers []int = []int{2, 7, 6, 5}

// 1. 洗牌，随意打乱
rand.NewSource(time.Now().UnixNano())
rand.Shuffle(len(pokers), func(i, j int) {
	pokers[i], pokers[j] = pokers[j], pokers[i]
})
fmt.Printf("1. 洗牌后的牌：%v\n", pokers)
```

我们以截图中刘谦手里持有的 4 张扑克牌为例，用一个 `int` 类型的 `slice` 存储这些牌。

在 Go 中可以使用 `rand.Shuffle` 方法来打乱 `slice` 的顺序。

执行程序，得到洗牌后的结果如下：

```bash
[6 7 5 2]
```

##### 二、 对折，然后撕开，接着叠在一起，相当于扑克牌变成了两份，double 了，4 张变 8 张。

![步骤二](2.png)

这一步相当于 `pokers` 内容 * 2，代码实现如下：

```go
pokers = append(pokers, pokers...)
fmt.Printf("2. 对折后的牌：%v\n", pokers)
```

复制后的 `pokers` 如下：

```bash
[6 7 5 2 6 7 5 2]
```

##### 三、 问问自己名字有几个字，就从最上面拿出对应个数的牌放到底部。

![步骤三](3.png)

例如「刘谦」名字有 2 个字，即将 `slice` 前 2 个元素取出，并放到 `slice` 最后，代码实现如下：

```go
pokers = append(pokers[2:], pokers[:2]...)
fmt.Printf("3. 问问自己名字有几个字，就从最上面拿出对应个数的牌放到底部：%v\n", pokers)
```

得到新 `pokers`：

```bash
[5 2 6 7 5 2 6 7]
```

##### 四、 拿起最上面的 3 张牌，插入中间任意位置。

![步骤四](4.png)

代码实现如下：

```go
pokers = append(pokers[3:7], pokers[0], pokers[1], pokers[2], pokers[7])
fmt.Printf("4. 拿起最上面的 3 张牌，插入中间任意位置: %v\n", pokers)
```

这里将 `pokers` 前 3 个元素取出，并插入最后一个元素之前，得到新 `pokers`：

```bash
[7 5 2 6 5 2 6 7]
```

##### 五、 拿出最上面的 1 张牌，藏于秘密的地方，比如屁股下。

![步骤五](5.png)

代码实现如下：

```go
top := pokers[0]
pokers = pokers[1:]
fmt.Printf("5. 拿出最上面的 1 张牌： %d, %v\n", top, pokers)
```
这里使用 top 变量暂存最上面的 1 张牌：

```bash
7, [5 2 6 5 2 6 7]
```

此时得到 `top` 值为 7，`pokers` 还剩 7 个元素。

##### 六、 如果你是南方人，从上面拿起 1 张牌；如果你是北方人，则从上面拿起 2 张牌；假如我们不确定自己是南方人还是北方人，那就干脆拿起 3 张牌，然后插入中间任意位置。

![步骤六](6.png)

这一步其实同第四步操作一样，假设我们是北方人，代码实现如下：

```go
pokers = append(pokers[2:6], pokers[0], pokers[1], pokers[6])
fmt.Printf("6. 从上面拿出 2 张牌: %v\n", pokers)
```

得到新 `pokers`：

```bash
[6 5 2 6 5 2 7]
```

##### 七、 如果你是男生，从上面拿起 1 张牌；如果你是女生，则从上面拿起 2 张牌，撒到空中（扔掉）。

![步骤七](7.png)

假设是男生，代码实现如下：

```go
pokers = pokers[1:]
fmt.Printf("7. 如果你是男生，从上面拿起 1 张牌；如果你是女生，则从上面拿起 2 张牌，撒到空中（扔掉）：%v\n", pokers)
```

得到新 `pokers`：

```bash
[5 2 6 5 2 7]
```

##### 八、 魔法时刻，在遥远的魔术的历史上，流传了一个七字真言「见证奇迹的时刻」，可以带给我们幸福。现在，我们每念一个字，从上面拿一张放到最底部，即需要完成 7 次同样的操作。

![步骤八](8.png)

我们可以用 `for loop` 来完成重复操作，代码实现如下：

```go
for range []string{"见", "证", "奇", "迹", "的", "时", "刻"} {
	pokers = append(pokers[1:], pokers[0])
}
fmt.Printf("8. 见证奇迹的时刻：%v\n", pokers)
```

得到新 `pokers`：

```bash
[2 6 5 2 7 5]
```

##### 九、 最后一个环节，叫「好运留下来，烦恼丢出去」，在念到「好运留下来」时，从上面拿起 1 张牌放入底部；在念到「烦恼丢出去」时，从上面拿起 1 张牌扔掉，女生需要完成 4 次同样的操作，男生需要完成 5 次同样的操作。

![步骤九 - 好运留下来](9-1.png)

![步骤九 - 烦恼丢出去](9-2.png)

因为第七步中我们选择了男生，所以现在要完成 5 次同样的操作，同样适用 `for loop` 来完成，代码实现如下：

```go
for range []int{1, 2, 3, 4, 5} {
	// 好运留下来
	pokers = append(pokers[1:], pokers[0])
	// 烦恼丢出去
	pokers = pokers[1:]
}
fmt.Printf("9. 好运留下来，烦恼丢出去：%v\n", pokers)
```

最终得到的 `pokers`：

```bash
[7]
```

切片 `pokers` 中仅存一个元素，即手中只剩一张牌。

##### 见证奇迹

接下来就是见证奇迹的时刻，拿出藏于屁股下的牌和手里唯一剩下的一张牌对比，正是同一张牌：

![见证奇迹](10.png)

打印 `top` 值和 `pokers` 中仅存的元素。

```go
fmt.Printf("见证奇迹：%d == %d", top, pokers[0])
```

结果毫无悬念：

```bash
见证奇迹：7 == 7
```

程序完整日志输出如下：

```bash
1. 洗牌后的牌：[6 7 5 2]
2. 对折后的牌：[6 7 5 2 6 7 5 2]
3. 问问自己名字有几个字，就从最上面拿出对应个数的牌放到底部：[5 2 6 7 5 2 6 7]
4. 拿起最上面的 3 张牌，插入中间任意位置: [7 5 2 6 5 2 6 7]
5. 拿出最上面的 1 张牌：7, [5 2 6 5 2 6 7]
6. 从上面拿出 2 张牌: [6 5 2 6 5 2 7]
7. 如果你是男生，从上面拿起 1 张牌；如果你是女生，则从上面拿起 2 张牌，撒到空中（扔掉）：[5 2 6 5 2 7]
8. 见证奇迹的时刻：[2 6 5 2 7 5]
9. 好运留下来，烦恼丢出去：[7]
见证奇迹：7 == 7
```

#### 还原「尼格买提」神操作

在此必须回顾一下尼格买提尴尬瞬间，反复尴尬 🤣：

<div style="text-align: center"><img src="0.png" width="100%" style="max-width: 400px;"></div>

仔细观看回放就能发现，其实小尼错在了第六步，小尼是北方人，所以应该从上面拿起 2 张牌，插入中间任意位置，但由于操作失误，错将拿起的第 2 张牌插入到最底部。

神操作在此：

<div style="text-align: center"><img src="6-1.png" width="100%" style="max-width: 400px;"></div>

即第六步代码原本应该如下：

```go
pokers = append(pokers[2:6], pokers[0], pokers[1], pokers[6])
```

到小尼这里变成了：

```go
pokers = append(pokers[2:6], pokers[0], pokers[6], pokers[1])
```

读者可以自行修改代码后尝试运行下，我就不展示结果了，怕尴尬 😅，哈哈哈。

### 彩蛋

不知道大家有没有注意到，在魔术表演结束时，观众席镜头左下角那个手里拿着 AJ 的姐姐！

![观众席](a-j-1.png)

放大特写：

<div style="text-align: center"><img src="a-j.png" width="100%" style="max-width: 400px;"></div>

AJ 姐姐满脸洋溢着尴尬的笑容 🤣。

### 总结

没有总结。

本文仅供大家消遣娱乐，顺便学点编程 😄。

本文完整代码示例我放在了 [GitHub](https://github.com/jianghushinian/blog-go-example/tree/main/2024-spring-festival-gala-magic/main.go) 上，欢迎点击查看。

**联系我**

- 微信：jianghushinian
- 邮箱：[jianghushinian007@outlook.com](mailto:jianghushinian007@outlook.com)
- 博客地址：https://jianghushinian.cn/
