---
title: 'Go 语言中你不知道的 io.Discard 妙用'
date: 2025-04-18 16:28:01
tags:
- Go
categories:
- Go
---

在 Go 语言中，`io.Discard` 是一个实现了 `io.Writer` 接口的**特殊变量**，用于丢弃所有写入的数据。 `io.Discard` 在 Go 1.15 及之前版本中是放在 `io/ioutil` 包中实现的。而在 Go 1.16 版本，得以正式转正，被实现在 `io` 包中。本文我们来一起学习下 `io.Discard` 的实现及使用场景。

<!-- more -->

### 使用示例

假如我们有一个 Web Server 程序，它提供了 HTTP Get 请求的健康检查接口 `/healthz`，调用后返回 HTTP 200 状态码和类似 `{"timestamp":"2025-04-17 23:42:15"}` 格式的 Body 数据。

其实大多数情况下，我们想检查这个 Web Server 是否健康，只需检查 `/healthz` 接口响应的状态码为 200 即可，而不必关心返回内容。

如果使用 Go 程序来做健康检查，那么就可以使用 `io.Discard` 丢弃响应体内容，实现代码如下：

```go
func healthCheck() {
	// 发送健康检查请求
	resp, err := http.Get("http://127.0.0.1:5555/healthz")
	if err != nil {
		panic(fmt.Sprintf("请求失败: %v", err))
	}
	defer resp.Body.Close() // 确保连接回收

	// 丢弃响应体
	_, _ = io.Copy(io.Discard, resp.Body)

	// 状态码校验
	if resp.StatusCode != http.StatusOK {
		panic(fmt.Sprintf("非预期状态码: %d", resp.StatusCode))
	}

	fmt.Println("健康检查通过")
}
```

这里使用了 `io.Copy(io.Discard, resp.Body)` 将响应体内容复制到 `io.Discard` 进行丢弃。你可能会问，不丢弃 `resp.Body` 会有什么后果吗？

如果不丢弃 `resp.Body`，那么 `defer resp.Body.Close()` 还是能够确保 `resp.Body` 被正确关闭，不会导致资源泄露。

不过，如果我们主动清空 `resp.Body` 的内容，则会带来一个额外的好处，会使当前这个 HTTP 请求的底层 TCP 连接能够被复用，从而更加有效的利用资源。

如下是 `http.Get(url)` 这个函数调用流程中涉及的核心代码：

```go
func Get(url string) (resp *Response, err error) {
	return DefaultClient.Get(url)
}

func (c *Client) Get(url string) (resp *Response, err error) {
	req, err := NewRequest("GET", url, nil)
	if err != nil {
		return nil, err
	}
	return c.Do(req)
}

func (c *Client) Do(req *Request) (*Response, error) {
	return c.do(req)
}

func (c *Client) do(req *Request) (retres *Response, reterr error) {
	...

			// Close the previous response's body. But
			// read at least some of the body so if it's
			// small the underlying TCP connection will be
			// re-used. No need to check for errors: if it
			// fails, the Transport won't reuse it anyway.
			const maxBodySlurpSize = 2 << 10
			if resp.ContentLength == -1 || resp.ContentLength <= maxBodySlurpSize {
				io.CopyN(io.Discard, resp.Body, maxBodySlurpSize)
			}
			resp.Body.Close()
    ...
}
```

请求调用流程是：`http.Get(url)` => ` DefaultClient.Get(url)` => `c.Do(req)` => `c.do(req)`。

在 `c.do` 方法中，有这么一段注释值得我们注意：

```go
// Close the previous response's body. But
// 关闭前一个响应的 body。但
// read at least some of the body so if it's
// 至少读取部分响应体数据，这样当响应体较小时
// small the underlying TCP connection will be
// 底层 TCP 连接会被复用。
// re-used. No need to check for errors: if it
// 无需检查错误：若操作失败，
// fails, the Transport won't reuse it anyway.
// Transport 也不会复用该连接。
```

也就是说当我们调用 `http.Get(url)` 发起 HTTP 请求后，如果操作失败或未读取 Body，那么 Transport 是不会复用 TCP 连接的。如果读取 Body 后再关闭连接，那么 Transport 就会复用底层  TCP  连接。

结合这段注释和代码，可以发现，其实 `http.Get(url)` 内部也会使用 `io.CopyN(io.Discard, resp.Body, maxBodySlurpSize)` 这种形式丢弃响应体，从而复用 TCP 连接。

看来 `io.Discard` 还是一个比较有用的工具。

### 源码解读

你是否会好奇 `io.Discard` 是如何实现的呢？

其实它的实现非常简单，接下来，我们就从源码的角度来更加深入的学习 `io.Discard`。

#### Go 1.15 及之前版本

我在前文中说过 `io.Discard` 在 Go 1.15 及之前版本中，是存放在 `io/ioutil` 包的，其代码如下：

> [https://github.com/golang/go/blob/go1.15.15/src/io/ioutil/ioutil.go#L158](https://github.com/golang/go/blob/go1.15.15/src/io/ioutil/ioutil.go#L158)
>

```go
type devNull int

// devNull implements ReaderFrom as an optimization so io.Copy to
// ioutil.Discard can avoid doing unnecessary work.
var _ io.ReaderFrom = devNull(0)

// Write 实现 io.Writer 接口
func (devNull) Write(p []byte) (int, error) {
	return len(p), nil
}

// WriteString 实现 io.StringWriter 接口
func (devNull) WriteString(s string) (int, error) {
	return len(s), nil
}

var blackHolePool = sync.Pool{
	New: func() interface{} {
		b := make([]byte, 8192)
		return &b
	},
}

// ReadFrom 实现 io.ReadFrom 接口
func (devNull) ReadFrom(r io.Reader) (n int64, err error) {
	bufp := blackHolePool.Get().(*[]byte)
	readSize := 0
	for {
		readSize, err = r.Read(*bufp)
		n += int64(readSize)
		if err != nil {
			blackHolePool.Put(bufp)
			if err == io.EOF {
				return n, nil
			}
			return
		}
	}
}

// Discard is an io.Writer on which all Write calls succeed
// without doing anything.
var Discard io.Writer = devNull(0)
```

通过 `var Discard io.Writer = devNull(0)` 这行代码我们知道，`io.Discard` 是一个变量，并且实现了 `io.Writer` 接口，其类型为 `devNull`，而 `devNull` 的底层类型是 `int` 类型。

从 Go 1.15 版本中 `io.Discard` 的类型名称可以看出，其对标的正是 Linux/Unix 系统中的 `/dev/null`。在 Linux/Unix 系统中，`/dev/null` 是一个“黑洞”，用于实现静默丢弃数据。它是一个特殊的设备文件，向它写入的任何数据都会被系统所丢弃，读取它时则立即返回文件结束（EOF）。

如下是一个在 Makefile 命令中使用 `/dev/null` 的示例。

> [https://github.com/onexstack/miniblog/blob/master/scripts/make-rules/golang.mk#L32](https://github.com/onexstack/miniblog/blob/master/scripts/make-rules/golang.mk#L32)
>

```makefile
go.build.verify:
	@if ! which go &>/dev/null; then echo "Cannot found go compile tool. Please install go tool first."; exit 1; fi
```

这里使用 `&>/dev/null` 将 `which go` 执行的输出进行丢弃。

当执行 `make go.build.verify` 时，如果在系统中未安装 Go，则会报错并以状态码 1 退出，输出 `Cannot found go compile tool. Please install go tool first.`，如果安装了 Go，则正常退出。

> NOTE:
>
> 此 Makefile 代码片段来源于 onexstack 技术栈中的开源项目 [miniblog](https://github.com/onexstack/miniblog/tree/master)，miniblog 是一个小而美的 Go 实战项目，入门但不简单。
>
> 你可以在这里 [https://t.zsxq.com/1979HJ45x](https://t.zsxq.com/1979HJ45x) 看到 miniblog 项目的教程。
>

Go 语言中 `io.Discard` 的行为正是模拟了 `/dev/null` 的特性。

`io.Discard` 实现的 `io.Writer` 接口非常简单，在 `Write` 方法中什么也没做，直接返回了 `len(p), nil`，即写入数据长度和无任何错误。`io.Discard` 就是以这么朴素的方式实现了数据的丢弃。

`io.Discard` 实现的 `WriteString` 方法同样如此，此方法用来实现 `io.StringWriter` 接口。

此外，`io.Discard` 还实现了 `io.ReadFrom` 接口，实现这个接口的方法内部逻辑不再只有一行代码，不过我们先不急着研究它，放在下一小结讲解 Go 1.16 及之后版本的 `io.Discard` 源码部分。

#### Go 1.16 及之后版本

在 Go 1.16 及之后版本中 `io.Discard` 的实现迁移到了 `io` 中，其代码如下：

> [https://github.com/golang/go/blob/go1.24.0/src/io/io.go#L639](https://github.com/golang/go/blob/go1.24.0/src/io/io.go#L639)
>

```go
// Discard 是一个 [io.Writer] 接口的实现，所有写入调用都会成功但不会执行任何实际操作
var Discard Writer = discard{}

// discard 是一个空结构体实现
type discard struct{}

// discard 实现了 io.ReaderFrom 接口作为性能优化手段，使得通过 io.Copy 向 io.Discard 拷贝数据时能避免冗余操作
var _ ReaderFrom = discard{}

// Write 实现 io.Writer 接口
func (discard) Write(p []byte) (int, error) {
	return len(p), nil
}

// WriteString 实现 io.StringWriter 接口
func (discard) WriteString(s string) (int, error) {
	return len(s), nil
}

// 使用 sync.Pool 构建缓冲池
var blackHolePool = sync.Pool{
	New: func() any {
		b := make([]byte, 8192) // 8KB 缓冲池
		return &b               // 返回指针对象
	},
}

// ReadFrom 实现 io.ReadFrom 接口
func (discard) ReadFrom(r Reader) (n int64, err error) {
	bufp := blackHolePool.Get().(*[]byte) // 从池中获取缓冲区
	readSize := 0
	for {
		readSize, err = r.Read(*bufp) // 读取数据到缓冲区
		n += int64(readSize)          // 累加丢弃的字节数
		if err != nil {
			blackHolePool.Put(bufp) // 用后归还缓冲池
			if err == EOF {         // 正常结束
				return n, nil
			}
			return // 异常出错
		}
	}
}
```

`io.Discard` 的主要逻辑并没有改变，有趣的是，它从之前的 `devNull` 类型改成了 `discard` 类型。`devNull` 底层是 `int` 类型，而 `discard` 则是空结构体。至于为什么要改成空结构体，你可以在我的文章《[Go 中空结构体惯用法，我帮你总结全了！](https://mp.weixin.qq.com/s/Qi0RrRyHLx8Q4SmhQeb-uQ)》中找到答案。

在这里，我详细解释下 `ReadFrom` 方法的实现。

首先，这里使用 `sync.Pool` 构建了一个缓冲池 `blackHolePool`，`sync.Pool` 的 `New` 方法返回一个 8KB 大小的缓冲对象，当我们调用 `blackHolePool.Get()` 时，就可以从缓冲池中获取对象 `&b`。所以 `bufp` 是通过 `sync.Pool` 分配的 8KB 固定长度切片（`make([]byte, 8192)`）。通过 `sync.Pool` 复用 8KB 切片，避免频繁内存分配，能够降低 Go GC 的压力。

> 注意：
>
> 在使用 `sync.Pool` 时，New 属性所对应的构造函数应该返回指针类型对象 `&b` 这在 `sync.Pool` 官方示例中也有提到 [https://pkg.go.dev/sync@go1.24.0#example-Pool](https://pkg.go.dev/sync@go1.24.0#example-Pool)
>

接着，在 `for` 循环中，每次调用 `r.Read(*bufp)` 时，数据会从 `bufp` 切片的起始位置开始填充（这会覆盖前一次循环中缓冲区的内容）。这里会累加丢弃的字节数，用于最终返回。

对于 `bufp` 对象，用后需要归还到缓冲池中，以便下次使用，所以需要调用 `sync.Pool` 对象的 `Put` 方法进行归还。

如果 `r.Read` 返回 `EOF` 则终止循环，返回累计丢弃的字节数 `n`。如果返回其他类型错误，则直接返回 `err` 和已经读取的字节数。

`io.Discard` 之所以要实现 `ReadFrom` 方法，其实是为了支持与 `io.Copy` 协同工作。

当使用 `io.Copy(io.Discard, src)` 丢弃数据时，`io.Copy` 会优先调用 `ReadFrom` 方法。

`io.Copy` 实现源码如下：

> [https://github.com/golang/go/blob/go1.24.0/src/io/io.go#L387](https://github.com/golang/go/blob/go1.24.0/src/io/io.go#L387)
>

```go
func Copy(dst Writer, src Reader) (written int64, err error) {
	return copyBuffer(dst, src, nil)
}

func copyBuffer(dst Writer, src Reader, buf []byte) (written int64, err error) {
	// If the reader has a WriteTo method, use it to do the copy.
	// Avoids an allocation and a copy.
	if wt, ok := src.(WriterTo); ok {
		return wt.WriteTo(dst)
	}
	// Similarly, if the writer has a ReadFrom method, use it to do the copy.
	if rf, ok := dst.(ReaderFrom); ok {
		return rf.ReadFrom(src)
	}
	if buf == nil {
		size := 32 * 1024
		if l, ok := src.(*LimitedReader); ok && int64(size) > l.N {
			if l.N < 1 {
				size = 1
			} else {
				size = int(l.N)
			}
		}
		buf = make([]byte, size)
	}
	for {
		nr, er := src.Read(buf)
		if nr > 0 {
			nw, ew := dst.Write(buf[0:nr])
			if nw < 0 || nr < nw {
				nw = 0
				if ew == nil {
					ew = errInvalidWrite
				}
			}
			written += int64(nw)
			if ew != nil {
				err = ew
				break
			}
			if nr != nw {
				err = ErrShortWrite
				break
			}
		}
		if er != nil {
			if er != EOF {
				err = er
			}
			break
		}
	}
	return written, err
}
```

`io.Copy` 的核心实现逻辑遵循 “优先调用高效接口 => 动态适配缓冲区 => 循环读写兜底” 的分层策略。其核心步骤如下：

1. 如果 `src` 对象实现了 `WriterTo` 接口（如 `*os.File`、`*bytes.Buffer` 等），则直接调用 `wt.WriteTo(dst)` 将数据写入 `dst`，这样可以利用系统底层的零拷贝机制。
2. 如果 `dst` 对象实现了 `ReaderFrom` 接口（如 `*os.File`、`*net.TCPConn` 等），则直接调用 `rf.ReadFrom(src)` 从 `src` 读取内容到 `dst`，让目标对象自主控制数据读取逻辑。
3. 默认缓冲区大小设为  32KB，如果 `src` 对象是 `*LimitedReader` 类型，则动态调整缓冲区大小，避免内存浪费。
4. 最终实现循环读写逻辑，调用 `src.Read(buf)` 从 `src` 读取数据到 `buf`，然后调用 `dst.Write(buf[0:nr])` 从 `buf` 将数据写入 `dst`。

而 `io.Copy` 内部之所以考虑了这么多种场景，正是为了灵活性和性能优化所考虑的，比如 `*os.File.WriterTo` 方法实现了零拷贝，这样可以将性能优化到极致。

`io.Copy` 设计思想可以总结为下表：

| 层级 | 优化目标 | 技术手段 | 性能 |
| --- | --- | --- | --- |
| 接口优先 | 零拷贝、减少内存操作 | 检查是否实现 `WriterTo`/`ReaderFrom` 接口 | 高 |
| 缓冲区动态适配 | 平衡内存与系统调用开销 | 按需分配缓冲区大小 | 中 |
| 循环读写兜底 | 通用性、错误处理完备性 | 默认缓冲区大小，循环读写 | 低 |


那么 `io.Discard` 实现 `ReadFrom` 方法的目的其实也就不言自明了，核心目的是通过优化数据读取和丢弃的流程，提升性能并减少资源消耗。

### 实践案例

现在我们学习了 `io.Discard` 源码，可以继续来探索一下，`io.Discard` 更加真实的实践案例。

#### Go 源码中的应用

首先我们来看一下 `io.Discard` 在 Go 源码中如何应用？

我们知道，Go 语言在 `os` 包中，为我们提供了 `os.Stat` 函数，用于获取文件或目录元数据信息，其作用与 Linux/Unix 系统调用 `stat` 类似，但设计上更符合 Go 语言的接口规范。

如下是 `os.Stat` 函数使用示例：

```go
func filesize(name string) (int64, error) {
	fi, err := os.Stat(name)
	if err != nil {
		if os.IsNotExist(err) {
			return 0, errors.New("文件不存在")
		}
		return 0, fmt.Errorf("读取文件失败: %w", err)
	}
	return fi.Size(), nil
}
```

调用 `fi = os.Stat(name)` 可以通过文件路径 `name` 获取文件系统对象的元数据对象 `fi`，然后通过 `fi.Size()` 可以得到文件大小。

`os.Stat` 用法非常简单，通常 Go 内置包的函数或方法都会有完整的单元测试，那么 `os.Stat` 如何 Go 源码是如何测试其正确性的呢？测试代码如下：

> [https://github.com/golang/go/blob/go1.24.0/src/os/os_test.go#L180](https://github.com/golang/go/blob/go1.24.0/src/os/os_test.go#L180)
>

```go
func TestStat(t *testing.T) {
	t.Parallel()

	path := sfdir + "/" + sfname
	dir, err := Stat(path)
	if err != nil {
		t.Fatal("stat failed:", err)
	}
	if !equal(sfname, dir.Name()) {
		t.Error("name should be ", sfname, "; is", dir.Name())
	}
	filesize := size(path, t)
	if dir.Size() != filesize {
		t.Error("size should be", filesize, "; is", dir.Size())
	}
}
```

这个单元测试用例中调用了 `filesize := size(path, t)` 来获取文件大小，然后与 `dir.Size()` 进行比较，以此来判断通过 `os.Stat` 获取的文件大小是否正确。

而这个 `size` 函数实现如下：

> [https://github.com/golang/go/blob/go1.24.0/src/os/os_test.go#L133](https://github.com/golang/go/blob/go1.24.0/src/os/os_test.go#L133)
>

```go
func size(name string, t *testing.T) int64 {
	file, err := Open(name)
	if err != nil {
		t.Fatal("open failed:", err)
	}
	defer func() {
		if err := file.Close(); err != nil {
			t.Error(err)
		}
	}()
	n, err := io.Copy(io.Discard, file)
	if err != nil {
		t.Fatal(err)
	}
	return n
}
```

可以发现，`size` 函数使用 `n, err := io.Copy(io.Discard, file)` 来丢弃 `file` 中的内容，并将返回值 `n` 作为文件大小。

这便是 `io.Discard` 包在 Go 源码中的应用。

#### Kubernetes 源码中的应用

接着我们再来看一下 `io.Discard` 在 Kubernetes 源码中如何应用？

我们知道，Kubernetes 在 Pod 的生命周期中为容器提供了 `postStart` 启动钩子和 `preStop` 停止钩子。比如我们可以像如下方式，在容器创建后，将当前 Pod 运行的服务上报到注册中心：

```yaml
lifecycle:
  postStart:
    httpGet:
      path: /register?ip=${POD_IP}&port=8080
      port: 8848  # 注册中心端口
      host: service.default.svc.cluster.local
      scheme: HTTP
```

我们可以更进一步，从源码的角度看一下 `postStart` 内部的执行逻辑。

为了方便，我们可以从单元测试开始读起，在 kubelet 源码中找到 `TestRunHandlerHttpsFailureFallback` 测试函数：

> [https://github.com/kubernetes/kubernetes/blob/v1.31.0/pkg/kubelet/lifecycle/handlers_test.go#L727](https://github.com/kubernetes/kubernetes/blob/v1.31.0/pkg/kubelet/lifecycle/handlers_test.go#L727)
>

```go
func TestRunHandlerHttpsFailureFallback(t *testing.T) {
    ...
    handlerRunner := NewHandlerRunner(srv.Client(), &fakeContainerCommandRunner{}, fakePodStatusProvider, recorder).(*handlerRunner)
	...
	container := v1.Container{
		Name: containerName,
		Lifecycle: &v1.Lifecycle{
			PostStart: &v1.LifecycleHandler{
				HTTPGet: &v1.HTTPGetAction{
					// set the scheme to https to ensure it falls back to HTTP.
					Scheme: "https",
					Host:   "127.0.0.1",
					Port:   intstr.FromString(port),
					Path:   "bar",
					HTTPHeaders: []v1.HTTPHeader{
						{
							Name:  "Authorization",
							Value: "secret",
						},
					},
				},
			},
		},
	}
	pod := v1.Pod{}
	...
	pod.Spec.Containers = []v1.Container{container}
	msg, err := handlerRunner.Run(ctx, containerID, &pod, &container, container.Lifecycle.PostStart)
	...
}
```

在这里，首先通过 `NewHandlerRunner` 函数构造了一个用来处理容器生命周期函数的 `Handler` 对象 `handlerRunner`。

接下来创建的 `container` 中实现了 `postStart` 钩子，通过 HTTP Get 方式，访问 `https://127.0.0.1:port/bar` 地址。在 Pod 准备就绪后，调用 `handlerRunner.Run` 方法来处理容器生命周期函数。

`Run` 方法实现如下：

> [https://github.com/kubernetes/kubernetes/blob/v1.31.0/pkg/kubelet/lifecycle/handlers.go#L70](https://github.com/kubernetes/kubernetes/blob/v1.31.0/pkg/kubelet/lifecycle/handlers.go#L70)
>

```go
func (hr *handlerRunner) Run(ctx context.Context, containerID kubecontainer.ContainerID, pod *v1.Pod, container *v1.Container, handler *v1.LifecycleHandler) (string, error) {
	switch {
	case handler.Exec != nil:
		...
	case handler.HTTPGet != nil:
		err := hr.runHTTPHandler(ctx, pod, container, handler, hr.eventRecorder)
		var msg string
		if err != nil {
			msg = fmt.Sprintf("HTTP lifecycle hook (%s) for Container %q in Pod %q failed - error: %v", handler.HTTPGet.Path, container.Name, format.Pod(pod), err)
			klog.V(1).ErrorS(err, "HTTP lifecycle hook for Container in Pod failed", "path", handler.HTTPGet.Path, "containerName", container.Name, "pod", klog.KObj(pod))
		}
		return msg, err
	case handler.Sleep != nil:
		...
	default:
		...
	}
}
```

可以看到，如果是  HTTP Get 类型的钩子，会调用 `hr.runHTTPHandler` 进一步处理。

`runHTTPHandler` 方法实现如下：

> [https://github.com/kubernetes/kubernetes/blob/v1.31.0/pkg/kubelet/lifecycle/handlers.go#L143](https://github.com/kubernetes/kubernetes/blob/v1.31.0/pkg/kubelet/lifecycle/handlers.go#L143)
>

```go
func (hr *handlerRunner) runHTTPHandler(ctx context.Context, pod *v1.Pod, container *v1.Container, handler *v1.LifecycleHandler, eventRecorder record.EventRecorder) error {
	...
	req, err := httpprobe.NewRequestForHTTPGetAction(handler.HTTPGet, container, podIP, "lifecycle")
	if err != nil {
		return err
	}
	resp, err := hr.httpDoer.Do(req)
	discardHTTPRespBody(resp)
	...
}
```

这里通过 `httpprobe.NewRequestForHTTPGetAction` 构造一个 `HTTP` 请求对象，然后交给 `hr.httpDoer.Do(req)` 去执行，接下来重点来了，这里会调用 `discardHTTPRespBody(resp)` 函数。根据这个函数的命名，我们也能大概猜测到它是用来干什么的。

`discardHTTPRespBody` 函数实现如下：

> [https://github.com/kubernetes/kubernetes/blob/v1.31.0/pkg/kubelet/lifecycle/handlers.go#L199](https://github.com/kubernetes/kubernetes/blob/v1.31.0/pkg/kubelet/lifecycle/handlers.go#L199)
>

```go
func discardHTTPRespBody(resp *http.Response) {
	if resp == nil {
		return
	}

	// Ensure the response body is fully read and closed
	// before we reconnect, so that we reuse the same TCP
	// connection.
	defer resp.Body.Close()

	if resp.ContentLength <= maxRespBodyLength {
		io.Copy(io.Discard, &io.LimitedReader{R: resp.Body, N: maxRespBodyLength})
	}
}
```

可以发现，这里与我在文章开头说演示的示例思想完全相同，在 `defer` 语句中关闭 `resp.Body`，并主动使用 `io.Copy(io.Discard, ...)` 来丢弃 `resp.Body`。

并且注释写的也很明确，确保读完响应体内容并关闭，能够实现复用 TCP 连接。

这便是 `io.Discard` 包在 Kubernetes 源码中的应用。

### 总结

`io.Discard` 虽然是 Go 提供的一个很小的功能点，但其有自己的妙用所在。`io.Discard` 对标的是 Linux/Unix 系统中的 `/dev/null` 文件，我们可以使用它来实现静默丢弃数据。

我带你分别阅读了 `io.Discard` 在 Go 1.15 及之前版本以及在 Go 1.16 及之后版本中的实现。即使是这样一个小功能点，Go 在升级时也做了优化，使用空结构体替代了 `int` 的实现。

我还举了两个实践案例，来带你体会 `io.Discard` 的真实使用场景。

Go 中的 `os` 包在单元测试中，利用 `io.Discard` 来计算文件大小。我们在日常的开发中也可以思考一下在哪些单元测试场景中可以使用 `io.Discard`。

Kubernetes 在 kubelet 中使用 `io.Discard` 来丢弃 HTTP 响应体，以此达到复用底层 TCP 连接的目的。

你还知道 `io.Discard` 有哪些用法，欢迎告诉我咱们一起交流讨论。

本文示例源码我都放在了 [GitHub](https://github.com/jianghushinian/blog-go-example/tree/main/io/discard) 中，欢迎点击查看。

希望此文能对你有所启发。

**延伸阅读**

+ Go 旧版 io.Discard 源码实现：[https://github.com/golang/go/blob/go1.15.15/src/io/ioutil/ioutil.go#L158](https://github.com/golang/go/blob/go1.15.15/src/io/ioutil/ioutil.go#L158)
+ Go 新版 io.Discard 源码实现：[https://github.com/golang/go/blob/go1.24.0/src/io/io.go#L387](https://github.com/golang/go/blob/go1.24.0/src/io/io.go#L387)
+ Go os 包 io.Discard 使用示例：[https://github.com/golang/go/blob/go1.24.0/src/os/os_test.go#L133](https://github.com/golang/go/blob/go1.24.0/src/os/os_test.go#L133)
+ Kubernetes io.Discard 使用示例：[https://github.com/kubernetes/kubernetes/blob/v1.31.0/pkg/kubelet/lifecycle/handlers.go#L199](https://github.com/kubernetes/kubernetes/blob/v1.31.0/pkg/kubelet/lifecycle/handlers.go#L199)
+ sync.Pool 使用文档：[https://pkg.go.dev/sync@go1.24.0#example-Pool](https://pkg.go.dev/sync@go1.24.0#example-Pool)
+ onexstack miniblog 项目：[https://github.com/onexstack/miniblog](https://github.com/onexstack/miniblog)
+ 本文 GitHub 示例代码：[https://github.com/jianghushinian/blog-go-example/tree/main/io/discard](https://github.com/jianghushinian/blog-go-example/tree/main/io/discard)
+ 本文永久地址：[https://jianghushinian.cn/2025/04/18/io-discard](https://jianghushinian.cn/2025/04/18/io-discard)

**联系我**

+ 公众号：[Go编程世界](https://jianghushinian.cn/about)
+ 微信：jianghushinian
+ 邮箱：[jianghushinian007@outlook.com](mailto:jianghushinian007@outlook.com)
+ 博客：[https://jianghushinian.cn](https://jianghushinian.cn)
+ GitHub：[https://github.com/jianghushinian](https://github.com/jianghushinian)