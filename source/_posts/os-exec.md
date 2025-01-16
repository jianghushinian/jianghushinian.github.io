---
title: 'Go os/exec 极速入门'
date: 2025-01-16 23:02:20
tags:
- Go
categories:
- Go
---

`os/exec` 是 Go 提供的内置包，可以用来执行外部命令或程序。比如，我们的主机上安装了 `redis-server` 二进制文件，那么就可以使用 `os/exec` 在 Go 程序中启动 `redis-server` 提供服务。当然，我们也可以使用 `os/exec` 执行 `ls`、`pwd` 等操作系统内置命令。本文不求内容多么深入，旨在带大家极速入门 `os/exec` 的常规使用。

<!-- more -->

### 极速入门

如下是 [os/exec](https://pkg.go.dev/os/exec@go1.23.0#pkg-index) 包实现的 exported 结构体和方法：

```go
func LookPath(file string) (string, error)
type Cmd
    func Command(name string, arg ...string) *Cmd
    func CommandContext(ctx context.Context, name string, arg ...string) *Cmd
    func (c *Cmd) CombinedOutput() ([]byte, error)
    func (c *Cmd) Environ() []string
    func (c *Cmd) Output() ([]byte, error)
    func (c *Cmd) Run() error
    func (c *Cmd) Start() error
    func (c *Cmd) StderrPipe() (io.ReadCloser, error)
    func (c *Cmd) StdinPipe() (io.WriteCloser, error)
    func (c *Cmd) StdoutPipe() (io.ReadCloser, error)
    func (c *Cmd) String() string
    func (c *Cmd) Wait() error
```
`Cmd` 结构体表示一个准备或正在执行的外部命令。调用函数 `Command` 或 `CommandContext` 可以构造一个 `*Cmd` 对象。调用 `Run`、`Start`、`Output`、`CombinedOutput` 方法可以运行 `*Cmd` 对象所代表的命令。 调用 `Environ` 方法可以获取命令执行时的环境变量。调用 `StdinPipe`、`StdoutPipe`、`StderrPipe` 方法用于获取管道对象。调用 `Wait` 方法可以阻塞等待命令执行完成。调用 `String` 方法返回命令的字符串形式。`LookPath `函数用于搜索可执行文件。

#### 运行一个命令

使用最简单的方式运行一个命令：

```go
package main

import (
	"log"
	"os/exec"
)

func main() {
	// 创建一个命令
	cmd := exec.Command("echo", "Hello, World!")

	// 执行命令并等待命令完成
	err := cmd.Run() // 执行后控制台不会有任何输出
	if err != nil {
		log.Fatalf("Command failed: %v", err)
	}
}
```

`exec.Command` 函数用于创建一个命令，函数第一个参数是命令的名称，后面跟一个不定常参数作为这个命令的参数，最终会传递给这个命令。

`*Cmd.Run` 方法会阻塞等待命令执行完成，默认情况下命令执行后控制台不会有任何输出：

```bash
# 执行程序
$ go run main.go
# 执行完成后没有任何输出
```

我们也可以在后台运行一个命令：

```go
func main() {
	cmd := exec.Command("sleep", "3")

	// 执行命令（非阻塞，不会等待命令执行完成）
	if err := cmd.Start(); err != nil {
		log.Fatalf("Command start failed: %v", err)
		return
	}

	fmt.Println("Command running in the background...")

	// 阻塞等待命令完成
	if err := cmd.Wait(); err != nil {
		log.Fatalf("Command wait failed: %v", err)
		return
	}

	log.Println("Command finished")
}
```

实际上 `Run` 方法就等于 `Start` + `Wait` 方法，如下是 `Run` 方法源码的实现：

```go
func (c *Cmd) Run() error {
	if err := c.Start(); err != nil {
		return err
	}
	return c.Wait()
}
```

#### 创建带有 context 的命令

`os/exec` 还提供了一个 `exec.CommandContext` 构造函数可以创建一个带有 `context` 的命令。那么我们就可以利用 `context` 的特性来控制命令的执行了。

```go
func main() {
	ctx, cancel := context.WithTimeout(context.Background(), 100*time.Millisecond)
	defer cancel()

	cmd := exec.CommandContext(ctx, "sleep", "5")

	if err := cmd.Run(); err != nil {
		log.Fatalf("Command failed: %v\n", err) // signal: killed
	}
}
```

执行示例代码，得到输出如下：

```bash
$ go run main.go
2025/01/14 23:54:20 Command failed: signal: killed
exit status 1
```

当命令执行超时会收到 `killed` 信号自动取消。

#### 获取命令的输出

无论是调用 `*Cmd.Run` 还是 `*Cmd.Start` 方法，默认情况下执行命令后控制台不会得到任何输出。

我们可以使用 `*Cmd.Output` 方法来执行命令，以此来获取命令的标准输出：

```go
func main() {
	// 创建一个命令
	cmd := exec.Command("echo", "Hello, World!")

	// 执行命令，并获取命令的输出，Output 内部会调用 Run 方法
	output, err := cmd.Output()
	if err != nil {
		log.Fatalf("Command failed: %v", err)
	}

	fmt.Println(string(output)) // Hello, World!
}
```

执行示例代码，得到输出如下：

```bash
$ go run main.go
Hello, World!
```

#### 获取组合的标准输出和错误输出

`*Cmd.CombinedOutput` 方法能够在运行命令后，返回其组合的标准输出和标准错误输出：

```go
func main() {
	// 使用一个命令，既产生标准输出，也产生标准错误输出
	cmd := exec.Command("sh", "-c", "echo 'This is stdout'; echo 'This is stderr' >&2")

	// 获取 标准输出 + 标准错误输出 组合内容
	output, err := cmd.CombinedOutput()
	if err != nil {
		log.Fatalf("Command execution failed: %v", err)
	}

	// 打印组合输出
	fmt.Printf("Combined Output:\n%s", string(output))
}
```

执行示例代码，得到输出如下：

```bash
$ go run main.go
Combined Output:
This is stdout
This is stderr
```

#### 设置标准输出和错误输出

我们可以利用 `*Cmd` 对象的 `Stdout` 和 `Stderr` 属性，重定向标准输出和标准错误输出到当前进程：

```go
func main() {
	cmd := exec.Command("ls", "-l")

	// 设置标准输出和标准错误输出到当前进程，执行后可以在控制台看到命令执行的输出
	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stderr

	if err := cmd.Run(); err != nil {
		log.Fatalf("Command failed: %v", err)
	}
}
```

这样，使用 `*Cmd.Run` 执行命令后控制台就能看到命令执行的输出了。

执行示例代码，得到输出如下：

```bash
$ go run main.go
total 4824
-rw-r--r--  1 jianghushinian  staff       12 Jan  4 10:37 demo.log
drwxr-xr-x  3 jianghushinian  staff       96 Jan 13 09:41 examples
-rwxr-xr-x  1 jianghushinian  staff  2453778 Jan  1 15:09 main
-rw-r--r--  1 jianghushinian  staff     6179 Jan 15 09:13 main.go
```

#### 使用标准输入传递数据

我们可以使用 `grep` 命令接收 `stdin` 的数据，<font style="color:rgb(77, 77, 77);">然后在其中搜索包含指定模式的文本行：</font>

```go
func main() {
	cmd := exec.Command("grep", "hello")

	// 通过标准输入传递数据给命令
	cmd.Stdin = bytes.NewBufferString("hello world!\nhi there\n")

	// 获取标准输出
	output, err := cmd.Output()
	if err != nil {
		log.Fatalf("Command failed: %v", err)
		return
	}

	fmt.Println(string(output)) // hello world!
}
```

可以将一个 `io.Reader` 对象赋值给 `*Cmd.Stdin` 属性，来实现将数据通过 `stdin` 传递给外部命令。

执行示例代码，得到输出如下：

```bash
$ go run main.go
hello world!
```

再比如，我们还可以将打开的文件描述符传给 `*Cmd.Stdin` 属性：

```go
func main() {
	file, err := os.Open("demo.log") // 打开一个文件
	if err != nil {
		log.Fatalf("Open file failed: %v\n", err)
		return
	}
	defer file.Close()

	cmd := exec.Command("cat")
	cmd.Stdin = file       // 将文件作为 cat 的标准输入
	cmd.Stdout = os.Stdout // 获取标准输出

	if err := cmd.Run(); err != nil {
		log.Fatalf("Command failed: %v", err)
	}
}
```

只要是 `io.Reader` 对象即可。

#### 设置和使用环境变量

`*Cmd` 的 `Environ` 方法可以获取环境变量，`Env` 属性则可以设置环境变量：

```go
func main() {
	cmd := exec.Command("printenv", "ENV_VAR")

	log.Printf("ENV: %+v\n", cmd.Environ())

	// 设置环境变量
	cmd.Env = append(cmd.Environ(), "ENV_VAR=HelloWorld")

	log.Printf("ENV: %+v\n", cmd.Environ())

	// 获取输出
	output, err := cmd.Output()
	if err != nil {
		log.Fatalf("Command failed: %v", err)
	}

	fmt.Println(string(output)) // HelloWorld
}
```

这段代码输出结果与执行环境相关，我就不演示执行结果了，你可以自行尝试。

不过最终的 `output` 输出结果一定是 `HelloWorld`。

#### 使用管道

`os/exec` 支持管道功能，`*Cmd` 对象提供的 `StdinPipe`、`StdoutPipe`、`StderrPipe` 三个方法用于获取管道对象。故名思义，三者分别对应标准输入、标准输出、标准错误输出的管道对象。

<font style="color:rgb(0, 0, 0);">使用示例如下：</font>

```go
func main() {
	// 命令中使用了管道
	cmdEcho := exec.Command("echo", "hello world\nhi there")

	outPipe, err := cmdEcho.StdoutPipe()
	if err != nil {
		log.Fatalf("Command failed: %v", err)
	}

	// 注意，这里不能使用 Run 方法阻塞等待，应该使用非阻塞的 Start 方法
	if err := cmdEcho.Start(); err != nil {
		log.Fatalf("Command failed: %v", err)
	}

	cmdGrep := exec.Command("grep", "hello")
	cmdGrep.Stdin = outPipe
	output, err := cmdGrep.Output()
	if err != nil {
		log.Fatalf("Command failed: %v", err)
	}

	fmt.Println(string(output)) // hello world
}
```

首先创建一个用于执行 `echo` 命令的 `*Cmd` 对象 `cmdEcho`，并调用它的 `StdoutPipe` 方法获得标准输出管道对象 `outPipe`；然后调用 `Start` 方法非阻塞的方式执行 `echo` 命令；接着创建一个用于执行 `grep` 命令的 `*Cmd` 对象 `cmdGrep`，将 `cmdEcho` 的标准输出管道对象赋值给 `cmdGrep.Stdin` 作为标准输入，这样，两个命令就通过管道串联起来了；最终通过 `cmdGrep.Output` 方法拿到 `cmdGrep` 命令的标准输出。

执行示例代码，得到输出如下：

```bash
$ go run main.go
hello world
```

#### 使用 `bash -c` 执行复杂命令

如果你不想使用 `os/exec` 提供的管道功能，那么在命令中直接使用管道符 `|`，也可以实现同样功能。

不过此时就需要使用 `sh -c` 或者 `bash -c` 等 Shell 命令来解析执行更复杂的命令了：

```go
func main() {
	// 命令中使用了管道
	cmd := exec.Command("bash", "-c", "echo 'hello world\nhi there' | grep hello")

	output, err := cmd.Output()
	if err != nil {
		log.Fatalf("Command failed: %v", err)
	}

	fmt.Println(string(output)) // hello world
}
```

这段代码中的管道功能同样生效。

#### 指定工作目录

可以通过指定 `*Cmd` 对象的的 `Dir` 属性来指定工作目录：

```go
func main() {
	cmd := exec.Command("cat", "demo.log")
	cmd.Stdout = os.Stdout // 获取标准输出
	cmd.Stderr = os.Stderr // 获取错误输出

	// cmd.Dir = "/tmp" // 指定绝对目录
	cmd.Dir = "." // 指定相对目录

	if err := cmd.Run(); err != nil {
		log.Fatalf("Command failed: %v", err)
	}
}
```

#### 捕获退出状态

上面讲解了很多执行命令相关操作，但其实还有一个很重要的点没有讲到，就是如何捕获外部命令执行后的退出状态码：

```go
func main() {
	// 查看一个不存在的目录
	cmd := exec.Command("ls", "/nonexistent")

	// 运行命令
	err := cmd.Run()

	// 检查退出状态
	var exitError *exec.ExitError
	if errors.As(err, &exitError) {
		log.Fatalf("Process PID: %d exit code: %d", exitError.Pid(), exitError.ExitCode()) // 打印 pid 和退出码
	}
}
```

这里执行 `ls` 命令来查看一个不存在的目录 `/nonexistent`，程序退出状态码必然不为 `0`。

执行示例代码，得到输出如下：

```bash
$ go run main.go
2025/01/15 23:31:44 Process PID: 78328 exit code: 1
exit status 1
```

#### 搜索可执行文件

最后要介绍的函数就只剩一个 `LookPath` 了，它用来搜索可执行文件。

搜索一个存在的命令：

```go
func main() {
	path, err := exec.LookPath("ls")
	if err != nil {
		log.Fatal("installing ls is in your future")
	}
	fmt.Printf("ls is available at %s\n", path)
}
```

执行示例代码，得到输出如下：

```bash
 $ go run main.go
ls is available at /bin/ls
```

搜索一个不存在的命令：

```go
func main() {
	path, err := exec.LookPath("lsx")
	if err != nil {
		log.Fatal(err)
	}
	fmt.Printf("ls is available at %s\n", path)
}
```

执行示例代码，得到输出如下：

```bash
$ go run main.go
2025/01/15 23:37:45 exec: "lsx": executable file not found in $PATH
exit status 1
```

### 功能练习

介绍完了 `os/exec` 常用的方法和函数，我们现在来做一个小练习，使用 `os/exec` 来执行外部命令 `ls -l /var/log/*.log`。

示例如下：

```go
func main() {
	cmd := exec.Command("ls", "-l", "/var/log/*.log")

	output, err := cmd.CombinedOutput() // 获取标准输出和错误输出
	if err != nil {
		log.Fatalf("Command failed: %v", err)
	}

	fmt.Println(string(output))
}
```

执行示例代码，得到输出如下：

```bash
$ go run main.go
2025/01/16 09:15:52 Command failed: exit status 1
exit status 1
```

执行报错了，这里的错误码为 `1`，但错误信息并不明确。

这个报错其实是因为 `os/exec` 默认不支持通配符参数导致的，`exec.Command` 不支持直接在参数中使用 Shell 通配符（如 `*`），因为它不会通过 Shell 来解析命令，而是直接调用底层的程序。

要解决这个问题，可以通过显式调用 Shell（例如 `bash` 或 `sh`），让 Shell 来解析通配符。

比如使用 `bash -c` 执行通配符命令 `ls -l /var/log/*.log`：

```go
func main() {
    // 使用 bash -c 来解析通配符
    cmd := exec.Command("bash", "-c", "ls -l /var/log/*.log")

    output, err := cmd.CombinedOutput() // 获取标准输出和错误输出
    if err != nil {
        log.Fatalf("Command failed: %v", err)
    }

    fmt.Println(string(output))
}
```

执行示例代码，得到输出如下：

```bash
$ go run main.go
-rw-r--r--  1 root  wheel         0 Oct  7 21:20 /var/log/alf.log
-rw-r--r--  1 root  wheel     11936 Jan 13 11:36 /var/log/fsck_apfs.log
-rw-r--r--  1 root  wheel       334 Jan 13 11:36 /var/log/fsck_apfs_error.log
-rw-r--r--  1 root  wheel     19506 Jan 11 18:04 /var/log/fsck_hfs.log
-rw-r--r--@ 1 root  wheel  21015342 Jan 16 09:02 /var/log/install.log
-rw-r--r--  1 root  wheel      1502 Nov  5 09:44 /var/log/shutdown_monitor.log
-rw-r-----@ 1 root  admin      3779 Jan 16 08:59 /var/log/system.log
-rw-r-----  1 root  admin    187332 Jan 16 09:05 /var/log/wifi.log
```

此外，我们还可以用 Go 标准库提供的 `filepath.Glob` 来手动解析通配符：

```go
func main() {
	// 匹配通配符路径
	files, err := filepath.Glob("/var/log/*.log")
	if err != nil {
		log.Fatalf("Glob failed: %v", err)
	}
	if len(files) == 0 {
		log.Println("No matching files found")
		return
	}

	// 将匹配到的文件传给 ls 命令
	args := append([]string{"-l"}, files...)
	cmd := exec.Command("ls", args...)

	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stderr

	if err := cmd.Run(); err != nil {
		log.Fatalf("Command failed: %v", err)
	}
}
```

`filepath.Glob` 函数会返回模式匹配的文件名列表，如果不匹配则返回 `nil`。这样，我们就可以先解析文件名列表，再交给 `exec.Command` 来执行 `ls` 命令了。

### 项目实战

虽然我们现在入门了 `os/exec` 包，不过还没在实际项目中使用过，本小节就来看下 `os/exec` 在真实项目中的应用。

我在「[超简单！用 Go 启动 Redis 实例](https://jianghushinian.cn/2025/01/08/go-tempredis/)」一文中<font style="color:rgb(33, 33, 33);">介绍了一款可以用</font>来启动 `redis-server` 的开源库 `github.com/stvp/tempredis`。现在，我来带你深入 `tempredis` 的源码探究其实现原理，也让你体会下 `os/exec` 的实际应用。

`tempredis` 源码实现非常简单，核心源码只有两个文件 [tempredis.go](https://github.com/stvp/tempredis/blob/master/tempredis.go) 和 [config.go](https://github.com/stvp/tempredis/blob/master/config.go)。

`config.go` 源码如下：

> [https://github.com/stvp/tempredis/blob/master/config.go](https://github.com/stvp/tempredis/blob/master/config.go)
>

```go
// Config is a key-value map of Redis config settings.
type Config map[string]string

func (c Config) Socket() string {
	return c["unixsocket"]
}
```

这里定义了一个 `Config` 结构体用于保存配置信息，实际上就是一个 `map`。`Socket` 方法返回 `unixsocket` 这个 `key` 所对应的 `value`。

接下来我们来分析 `tempredis.go` 源码：

> [https://github.com/stvp/tempredis/blob/master/tempredis.go](https://github.com/stvp/tempredis/blob/master/tempredis.go)
>

```go
// Server encapsulates the configuration, starting, and stopping of a single
// redis-server process that is reachable via a local Unix socket.
type Server struct {
	dir       string
	config    Config
	cmd       *exec.Cmd
	stdout    io.Reader
	stdoutBuf bytes.Buffer
	stderr    io.Reader
}
```

首先，这里定义了一个结构体 `Server` 用于表示 `redis-server` 服务器对象。可以发现，其内部包含了 `*exec.Cmd` 类型属性 `cmd`，这是 `Server` 结构体的核心所在。

接下来看下 `Start` 函数的定义：

```go
// Start initiates a new redis-server process configured with the given
// configuration. redis-server will listen on a temporary local Unix socket. An
// error is returned if redis-server is unable to successfully start for any
// reason.
func Start(config Config) (server *Server, err error) {
	if config == nil {
		config = Config{}
	}

	dir, err := ioutil.TempDir(os.TempDir(), "tempredis")
	if err != nil {
		return nil, err
	}

	if _, ok := config["unixsocket"]; !ok {
		config["unixsocket"] = fmt.Sprintf("%s/%s", dir, "redis.sock")
	}
	if _, ok := config["port"]; !ok {
		config["port"] = "0"
	}

	server = &Server{
		dir:    dir,
		config: config,
	}
	err = server.start()
	if err != nil {
		return server, err
	}
	err = server.ready()
	if err != nil {
		return server, err
	}
	return server, nil
}
```

`Start` 函数用于根据给定的配置 `Config` 创建并启动 `Server`。

实例化的 `server` 对象就是上面介绍的 `Server` 结构体，这里通过 `server.start()` 来启动 Redis 服务，并通过 `server.ready()` 来阻塞判断服务是否启动成功。

`start` 方法定义如下：

```go
func (s *Server) start() (err error) {
	if s.cmd != nil {
		return fmt.Errorf("redis-server has already been started")
	}

	s.cmd = exec.Command("redis-server", "-")

	stdin, _ := s.cmd.StdinPipe()
	s.stdout, _ = s.cmd.StdoutPipe()
	s.stderr, _ = s.cmd.StderrPipe()

	err = s.cmd.Start()
	if err == nil {
		err = writeConfig(s.config, stdin)
	}

	return err
}
```

可以看到，这里构造的核心命令就是 `exec.Command("redis-server", "-")`，并将其赋值给 `s.cmd` 属性；然后分别记录了几个管道对象；接着调用 `s.cmd.Start()` 来执行命令；并通过 `writeConfig(s.config, stdin)` 将配置信息传递给标准输入供 `redis-server -` 命令读取使用。

`writeConfig` 函数实现如下：

```go
func writeConfig(config Config, w io.WriteCloser) (err error) {
	for key, value := range config {
		if value == "" {
			value = "\"\""
		}
		_, err = fmt.Fprintf(w, "%s %s\n", key, value)
		if err != nil {
			return err
		}
	}
	return w.Close()
}
```

接着我们再来看下 `ready` 方法的实现：

```go
func (s *Server) ready() (err error) {
	c := make(chan error, 1)
	go func() {
		// Block until Redis is ready to accept connections.
		c <- s.waitFor()
	}()

	select {
	case err := <-c:
		return err
	case <-time.After(1 * time.Second):
		return fmt.Errorf("timed out awaiting startup")
	}
}
```

这里启动一个新的 goroutine 调用 `s.waitFor()` 来阻塞等待 Redis 服务准备就绪，并将结果通过 `channel` 传递给主 goroutine，主 goroutine 会等待 Redis 服务准备就绪或超时退出。

`waitFor` 方法实现如下：

```go
var (
	// ready is the string redis-server prints to stdout after starting
	// successfully.
	ready = []string{
		"The server is now ready to accept connections",
		"Ready to accept connections",
	}
)

// waitFor blocks until redis-server is ready
func (s *Server) waitFor() (err error) {
	var line string

	scanner := bufio.NewScanner(s.stdout)
	for scanner.Scan() {
		line = scanner.Text()
		fmt.Fprintf(&s.stdoutBuf, "%s\n", line)
		for _, s := range ready {
			if strings.Contains(line, s) {
				return nil
			}
		}
	}
	err = scanner.Err()
	if err == nil {
		err = io.EOF
	}
	return err
}
```

这里的源码实现比较有意思，就是通过读取执行 `redis-server` 命令后，捕获到的标准输出中的字符串内容，来判断是否匹配 `ready` 变量中的字符串，以此来识别 `redis-server` 是否启动成功。在 Redis 7.2 版本之前，启动成功会在控制台输出 `The server is now ready to accept connections`，在 Redis 7.2 版本及以后，启动成功会在控制台输出 `Ready to accept connections`。

就是这样，有些我们以为可能需要写很多代码才能支持的功能，源码实现竟然如此朴素。

`Server` 对象也实现了 `Socket` 方法，不过仅仅是 `Config.Socket` 方法的代理：

```go
// Socket returns the full path to the local redis-server Unix socket.
func (s *Server) Socket() string {
	return s.config.Socket()
}
```

还记得在 `Server.start` 方法中会将管道对象记录到 `Server` 对象中吗？

```go
s.stdout, _ = s.cmd.StdoutPipe()
s.stderr, _ = s.cmd.StderrPipe()
```

如下就是二者被使用的地方：

```go
// Stdout blocks until redis-server returns and then returns the full stdout
// output.
func (s *Server) Stdout() string {
	io.Copy(&s.stdoutBuf, s.stdout)
	return s.stdoutBuf.String()
}

// Stderr blocks until redis-server returns and then returns the full stdout
// output.
func (s *Server) Stderr() string {
	bytes, _ := ioutil.ReadAll(s.stderr)
	return string(bytes)
}
```

这两个方法分别用来阻塞等待返回 `redis-server` 执行后的标准输出和标准错误输出。

它们二者在实现上有所差别，因为 `*Server.waitFor` 中其实已经消费了 `stdout`，所以会将其暂存到 `*Server.stdoutBuf` 属性中。

最后用于关闭 `redis-server` 的两个方法源码实现如下：

```go
// Term gracefully shuts down redis-server. It returns an error if redis-server
// fails to terminate.
func (s *Server) Term() (err error) {
	return s.signalAndCleanup(syscall.SIGTERM)
}

// Kill forcefully shuts down redis-server. It returns an error if redis-server
// fails to die.
func (s *Server) Kill() (err error) {
	return s.signalAndCleanup(syscall.SIGKILL)
}

func (s *Server) signalAndCleanup(sig syscall.Signal) error {
	s.cmd.Process.Signal(sig)
	_, err := s.cmd.Process.Wait()
	os.RemoveAll(s.dir)
	return err
}
```

这里通过 `s.cmd.Process.Signal(sig)` 为进程发送信号，`*Cmd.Process` 属性是 `*os.Process` 类型，实际上 `Cmd` 对象是底层类型 `*os.Process` 和 `*os.ProcessState` 的封装。我们暂且不必深究更底层的实现，只需要知道 `*Cmd.Process` 代表了命令执行的进行对象，调用 `Signal` 方法可以为进程发送信号，调用 `Wait` 方法会阻塞等待进程执行完成。

至此，`tempredis` 包的源码就都讲解完成了。

### 总结

`os/exec` 是 Go 语言内置包，用于运行一个外部命令。我向你介绍了它的常用方法，算是带你一起入个门。并且我们还一起阅读了第三方包 tempredis 的源码，以此来学习在工程实践中如何使用 `os/exec`。

此外，你还需要注意，调用 `*Cmd` 的 `Run`、`Start`、`Output`、`CombinedOutput` 方法执行外部命令以后，这个对象就无法重复使用了，即一个命令不能被执行多次，否则将得到 `exec: already started` 报错信息。

对 `os/exec` 的入门讲解就到这里，如果你还想进行更深入的探索，可以参考官方文档及其源码。

本文示例源码我都放在了 [GitHub](https://github.com/jianghushinian/blog-go-example/tree/main/os/exec) 中，欢迎点击查看。

希望此文能对你有所启发。

**延伸阅读**

+ os/exec Documentation：[https://pkg.go.dev/os/exec@go1.23.0](https://pkg.go.dev/os/exec@go1.23.0)
+ tempredis Documentation：[https://pkg.go.dev/github.com/stvp/tempredis](https://pkg.go.dev/github.com/stvp/tempredis)
+ tempredis GitHub 源码：[https://github.com/stvp/tempredis](https://github.com/stvp/tempredis)
+ <font style="color:rgb(31, 35, 40);">Distributed mutual exclusion lock using Redis for Go：</font>[https://github.com/go-redsync/redsync](https://github.com/go-redsync/redsync)
+ Install Redis：[https://redis.io/docs/latest/operate/oss_and_stack/install/install-redis/](https://redis.io/docs/latest/operate/oss_and_stack/install/install-redis/)
+ 本文 GitHub 示例代码：[https://github.com/jianghushinian/blog-go-example/tree/main/os/exec](https://github.com/jianghushinian/blog-go-example/tree/main/os/exec)

**联系我**

+ 公众号：[Go编程世界](https://jianghushinian.cn/about)
+ 微信：jianghushinian
+ 邮箱：[jianghushinian007@outlook.com](mailto:jianghushinian007@outlook.com)
+ 博客：[https://jianghushinian.cn](https://jianghushinian.cn)
+ GitHub：[https://github.com/jianghushinian](https://github.com/jianghushinian)
