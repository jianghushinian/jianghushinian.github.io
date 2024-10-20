---
title: '一行命令为项目文件添加开源协议头'
date: 2024-10-20 10:13:28
tags:
- Go
- License
- 开源
categories:
- Go
---

今天给大家介绍一款可以为项目文件添加开源协议头信息的命令行工具 [addlicense](https://github.com/superproj/addlicense)。

如果一个现有的项目，想要开源，免不了要为项目中的文件增加开源协议头信息。虽然很多 IDE 都可以为新创建的文件自动增加头信息，但修改已有的文件还是要麻烦些。好在我们有 `addlicense` 工具可以使用，一行命令就能搞定。并且 `addlicense` 是用 Go 语言开发的，本文不仅教你如何使用，还会对其源码进行分析讲解。

<!-- more -->

### 安装

使用如下命令安装 `addlicense`：

```bash
$ go install github.com/superproj/addlicense
```

使用 `-h/--help` 查看帮助信息：

```bash
$ addlicense -h
Usage: addlicense [flags] pattern [pattern ...]

The program ensures source code files have copyright license headers
by scanning directory patterns recursively.

It modifies all source files in place and avoids adding a license header
to any file that already has one.

The pattern argument can be provided multiple times, and may also refer
to single files.

Flags:

      --check                check only mode: verify presence of license headers and exit with non-zero code if missing
  -h, --help                 show this help message
  -c, --holder string        copyright holder (default "Google LLC")
  -l, --license string       license type: apache, bsd, mit, mpl (default "apache")
  -f, --licensef string      custom license file (no default)
      --skip-dirs strings    regexps of directories to skip
      --skip-files strings   regexps of files to skip
  -v, --verbose              verbose mode: print the name of the files that are modified
  -y, --year string          copyright year(s) (default "2024")
```

参数说明：

+ `--check` 只检查文件是否存在 License，执行后会打印所有不包含 License 版权头信息的文件名。
+ `-h/--help` 查看 `addlicense` 使用帮助信息，我们已经使用过了。
+ `-c/--holder` 指定 License 的版权所有者。
+ `-l/--license` 指定 License 的协议类型，目前内置支持了 `Apache 2.0`、`BSD`、`MIT` 和 `MPL 2.0` 协议。
+ `-f/--licensef` 指定自定义的 License 头文件。
+ `--skip-dirs` 跳过指定的目录。
+ `--skip-files` 跳过指定的文件。
+ `-v/--verbose` 打印被更改的文件名。
+ `-y/--year` 指定 License 的版权起始年份。

### 使用

准备实验的目录如下：

```bash
$ tree data -a
data
├── a
│   ├── main.go
│   └── main_test.go
├── b
│   └── c
│       └── keep
├── c
│   └── main.py
├── d.go
└── d_test.go

5 directories, 6 files
```

#### 使用内置 License

检查 `data` 目录下的所有文件是否存在 License 头信息：

```bash
$ addlicense --check data
data/a/main_test.go
data/d_test.go
data/d.go
data/c/main.py
data/a/main.go
```

输出了没有 License 头信息的文件。可以发现，这里自动跳过了没有后缀名的文件 `keep`。

> NOTE:
> 因为 `addlicense`是并发操作多个目录，所以每次执行打印结果顺序可能不同。

为缺失 License 头信息的文件添加 License 头信息：

```bash
$ addlicense -v -l mit -c 江湖十年 --skip-dirs=c data
data/a/main_test.go added license
data/a/main.go added license
data/d.go added license
data/d_test.go added license
```

输出了所有本次命令增加了 License 头信息的文件。

`data/a/main.go` 效果如下：

```go
// Copyright (c) 2024 江湖十年
//
// Permission is hereby granted, free of charge, to any person obtaining a copy of
// this software and associated documentation files (the "Software"), to deal in
// the Software without restriction, including without limitation the rights to
// use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of
// the Software, and to permit persons to whom the Software is furnished to do so,
// subject to the following conditions:
//
// The above copyright notice and this permission notice shall be included in all
// copies or substantial portions of the Software.
//
// THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
// IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS
// FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR
// COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER
// IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN
// CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.

package main

import "fmt"

...
```

#### 指定自定义 License

我们也可以指定自定义的 License 文件 `boilerplate.txt` 内容如下：

```plain
Copyright 2024 jianghushinian <jianghushinian007@outlook.com>. All rights reserved.
Use of this source code is governed by a MIT style
license that can be found in the LICENSE file. The original repo for
this file is https://github.com/jianghushinian/blog-go-example.
```

为缺失 License 头信息的文件添加 License 头信息：

```bash
$ addlicense -v -f ./boilerplate.txt --skip-dirs=^a$ --skip-files=d.go,d_test.go data
data/c/main.py added license
```

> NOTE:
> 注意这次的命令使用了正则 `--skip-dirs=^a$` 来跳过目录 `a`，没有直接使用 `--skip-dirs=a` 是因为如果这样做会跳过整个 `data` 目录，不再进一步遍历子目录。稍后阅读完 `addlicense` 源码就知道为什么会这样了。
>

`data/c/main.py` 效果如下：

```go
# Copyright 2024 jianghushinian <jianghushinian007@outlook.com>. All rights reserved.
# Use of this source code is governed by a MIT style
# license that can be found in the LICENSE file. The original repo for
# this file is https://github.com/jianghushinian/blog-go-example.

def main():
    print("Hello Python")
...
```

### 源码解读

我们学会了 `addlicense` 命令行工具如何使用，接下来可以深入其源码，来看看它是如何实现的。这样在使用过程中如果出现任何问题，也方便排查。

`addlicense` 项目很小，项目源文件如下：

```bash
$ tree addlicense                        
addlicense
├── Makefile
├── README.md
├── boilerplate.txt
├── go.mod
├── go.sum
└── main.go

1 directory, 6 files
```

`addlicense` 的代码逻辑，其实只有一个 `main.go` 文件，我们来对其代码进行逐行分析。

打开 `main.go` 文件，首先映入眼帘的就是 License 头信息：

```go
// Copyright 2020 Lingfei Kong <colin404@foxmail.com>. All rights reserved.
// Use of this source code is governed by a MIT style
// license that can be found in the LICENSE file.

// This program ensures source code files have copyright license headers.
// See usage with "addlicense -h".
package main

import (
	"bufio"
	"bytes"
	"errors"
	"fmt"
	"html/template"
	"io/ioutil"
	"os"
	"path/filepath"
	"regexp"
	"strings"
	"time"
	"unicode"

	"github.com/spf13/pflag"
	"golang.org/x/sync/errgroup"
)
```

License 头信息下面就是正常的 Go 包声明和导入信息。

接下来是几个常量的定义：

```go
const helpText = `Usage: addlicense [flags] pattern [pattern ...]

The program ensures source code files have copyright license headers
by scanning directory patterns recursively.

It modifies all source files in place and avoids adding a license header
to any file that already has one.

The pattern argument can be provided multiple times, and may also refer
to single files.

Flags:
`

const tmplApache = `Copyright {{.Year}} {{.Holder}}

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

     http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.`

const tmplBSD = `Copyright (c) {{.Year}} {{.Holder}} All rights reserved.
Use of this source code is governed by a BSD-style
license that can be found in the LICENSE file.`

const tmplMIT = `Copyright (c) {{.Year}} {{.Holder}}

Permission is hereby granted, free of charge, to any person obtaining a copy of
this software and associated documentation files (the "Software"), to deal in
the Software without restriction, including without limitation the rights to
use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of
the Software, and to permit persons to whom the Software is furnished to do so,
subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS
FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR
COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER
IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN
CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.`

const tmplMPL = `This Source Code Form is subject to the terms of the Mozilla Public
License, v. 2.0. If a copy of the MPL was not distributed with this
file, You can obtain one at https://mozilla.org/MPL/2.0/.`

```

常量 `helpText` 就是使用 `-h/--help` 打印帮助信息最上面的内容，回去看看是不是能对应上。

剩下的几个常量就是内置支持的 License 头信息了，分别是 `Apache`、`BSD`、`MIT`、`MPL` 协议。看到每个 License 头信息中的 `{ {.Year} } { {.Holder} }` 就知道，这是 Go template 的模板语法。

然后，我们能看到定义的所有命令行标志都在这里了：

```go
var (
	holder    = pflag.StringP("holder", "c", "Google LLC", "copyright holder")
	license   = pflag.StringP("license", "l", "apache", "license type: apache, bsd, mit, mpl")
	licensef  = pflag.StringP("licensef", "f", "", "custom license file (no default)")
	year      = pflag.StringP("year", "y", fmt.Sprint(time.Now().Year()), "copyright year(s)")
	verbose   = pflag.BoolP("verbose", "v", false, "verbose mode: print the name of the files that are modified")
	checkonly = pflag.BoolP(
		"check",
		"",
		false,
		"check only mode: verify presence of license headers and exit with non-zero code if missing",
	)
	skipDirs  = pflag.StringSliceP("skip-dirs", "", nil, "regexps of directories to skip")
	skipFiles = pflag.StringSliceP("skip-files", "", nil, "regexps of files to skip")
	help      = pflag.BoolP("help", "h", false, "show this help message")
)
```

这里使用了 `pflag` 库来定义所有命令行标志，每个标志的作用已经在前文讲解过了，忘记的读者可以翻上去回顾一下。

可以发现 `--skip-dirs` 和 `--skip-files` 两个标志都是 `slice` 类型，传入格式为 `a,b,c`。

> NOTE:
> 如果你不太熟悉 `pflag` 库，可以参考我的另一篇文章[《Go 命令行参数解析工具 pflag 使用》](https://jianghushinian.cn/2023/03/27/use-of-go-command-line-parameter-parsing-tool-pflag/)。

下面就进入主逻辑 `main` 函数了：

```go
func main() {
	pflag.Usage = usage
	pflag.Parse()

	if *help {
		pflag.Usage()
		os.Exit(1)
	}

	if pflag.NArg() == 0 {
		pflag.Usage()
		os.Exit(1)
	}

	if len(*skipDirs) != 0 {
		ps, err := getPatterns(*skipDirs)
		if err != nil {
			fmt.Println(err.Error())
			os.Exit(1)
		}

		patterns.dirs = ps
	}

	if len(*skipFiles) != 0 {
		ps, err := getPatterns(*skipFiles)
		if err != nil {
			fmt.Println(err.Error())
			os.Exit(1)
		}

		patterns.files = ps
	}

	data := &copyrightData{
		Year:   *year,
		Holder: *holder,
	}

	var t *template.Template
	if *licensef != "" {
		d, err := ioutil.ReadFile(*licensef)
		if err != nil {
			fmt.Printf("license file: %v\n", err)
			os.Exit(1)
		}
		t, err = template.New("").Parse(string(d))
		if err != nil {
			fmt.Printf("license file: %v\n", err)
			os.Exit(1)
		}
	} else {
		t = licenseTemplate[*license]
		if t == nil {
			fmt.Printf("unknown license: %s\n", *license)
			os.Exit(1)
		}
	}

	// process at most 1000 files in parallel
	ch := make(chan *file, 1000)
	done := make(chan struct{})
	go func() {
		var wg errgroup.Group
		for f := range ch {
			f := f // https://golang.org/doc/faq#closures_and_goroutines
			wg.Go(func() error {
				// nolint: nestif
				if *checkonly {
					// Check if file extension is known
					lic, err := licenseHeader(f.path, t, data)
					if err != nil {
						fmt.Printf("%s: %v\n", f.path, err)

						return err
					}
					if lic == nil { // Unknown fileExtension
						return nil
					}
					// Check if file has a license
					isMissingLicenseHeader, err := fileHasLicense(f.path)
					if err != nil {
						fmt.Printf("%s: %v\n", f.path, err)

						return err
					}
					if isMissingLicenseHeader {
						fmt.Printf("%s\n", f.path)

						return errors.New("missing license header")
					}
				} else {
					modified, err := addLicense(f.path, f.mode, t, data)
					if err != nil {
						fmt.Printf("%s: %v\n", f.path, err)

						return err
					}
					if *verbose && modified {
						fmt.Printf("%s added license\n", f.path)
					}
				}

				return nil
			})
		}
		err := wg.Wait()
		close(done)
		if err != nil {
			os.Exit(1)
		}
	}()

	for _, d := range pflag.Args() {
		walk(ch, d)
	}
	close(ch)
	<-done
}
```

这里逻辑很长，咱们一点点来拆解阅读。

首先是对命令行标志的处理：

```go
pflag.Usage = usage
pflag.Parse()

if *help {
    pflag.Usage()
    os.Exit(1)
}

if pflag.NArg() == 0 {
    pflag.Usage()
    os.Exit(1)
}
```

`pflag.Usage` 字段是一个函数，用来打印使用帮助信息，变量 `usage` 定义如下：

```go
var (
	...
	usage           = func() {
		fmt.Println(helpText)
		pflag.PrintDefaults()
	}
)
```

`if *help` 就是对 `-h/--help` 标志进行判断，如果用户输入了此标志，就打印帮助信息，并直接退出程序。

`pflag.NArg()` 返回处理完标志后剩余的参数个数，用来指定需要处理的目录。如果用户没传，同样打印帮助信息并退出。

如果执行 `addlicense -v -l mit -c 江湖十年 a b c` 命令，`pflag.NArg()` 会返回 `a`、`b`、`c` 三个目录。我们至少要传一个搜索路径，不然 `addlicense` 会不知道去找哪些文件。

你可能想，这里也可以设置为默认查找当前目录，即默认目录为 `.`。但是我个人不推荐这种设计，因为 `addlicense` 会修改文件，最好还是用户明确传了哪个目录，再去操作。不然假如用户不小心在家目录下执行了这个命令，所有文件都被改了。

显然，在这个场景中，显式胜于隐式。

接下来是对 `--skip-dirs` 和 `--skip-files` 两个命令行标志的处理：

```go
if len(*skipDirs) != 0 {
    ps, err := getPatterns(*skipDirs)
    if err != nil {
        fmt.Println(err.Error())
        os.Exit(1)
    }

    patterns.dirs = ps
}

if len(*skipFiles) != 0 {
    ps, err := getPatterns(*skipFiles)
    if err != nil {
        fmt.Println(err.Error())
        os.Exit(1)
    }

    patterns.files = ps
}
```

跳过的目录和文件都通过 `getPatterns` 函数来转换成正则表达式，并赋值给 `patterns` 对象。

`patterns` 和 `getPatterns` 定义如下：

```go
var patterns = struct {
	dirs  []*regexp.Regexp
	files []*regexp.Regexp
}{}

func getPatterns(patterns []string) ([]*regexp.Regexp, error) {
	patternsRe := make([]*regexp.Regexp, 0, len(patterns))
	for _, p := range patterns {
		patternRe, err := regexp.Compile(p)
		if err != nil {
			fmt.Printf("can't compile regexp %q\n", p)

			return nil, fmt.Errorf("compile regexp failed, %w", err)
		}
		patternsRe = append(patternsRe, patternRe)
	}

	return patternsRe, nil
}
```

接着又构建了一个 `copyrightData` 对象：

```go
data := &copyrightData{
    Year:   *year,
    Holder: *holder,
}
```

其中 `holder` 是通过 `-c/--holder` 传入的，`year` 是通过 `-y--year` 传入的，`year`不传默认值就是当前年份。

`data` 变量稍后将用于渲染模板。

而接下来就是构造模版逻辑：

```go
var t *template.Template
if *licensef != "" {
    d, err := ioutil.ReadFile(*licensef)
    if err != nil {
        fmt.Printf("license file: %v\n", err)
        os.Exit(1)
    }
    t, err = template.New("").Parse(string(d))
    if err != nil {
        fmt.Printf("license file: %v\n", err)
        os.Exit(1)
    }
} else {
    t = licenseTemplate[*license]
    if t == nil {
        fmt.Printf("unknown license: %s\n", *license)
        os.Exit(1)
    }
}
```

`if *licensef != ""` 表示如果用户使用`-f/--licensef` 指定了自定义的 License 头文件，则进入此逻辑，读取其中内容作为模板。

否则，使用默认支持的版权内容作为模板。`licenseTemplate` 是一个全局变量，并在 `init`中被初始化：

```go
var (
	licenseTemplate = make(map[string]*template.Template)
	...
)

func init() {
	licenseTemplate["apache"] = template.Must(template.New("").Parse(tmplApache))
	licenseTemplate["mit"] = template.Must(template.New("").Parse(tmplMIT))
	licenseTemplate["bsd"] = template.Must(template.New("").Parse(tmplBSD))
	licenseTemplate["mpl"] = template.Must(template.New("").Parse(tmplMPL))
}
```

无论哪个分支，只要报错，就会调用 `os.Exit(1)` 退出。

接下来就是程序的核心逻辑了：

```go
// process at most 1000 files in parallel
ch := make(chan *file, 1000)
done := make(chan struct{})
go func() {
    var wg errgroup.Group
    for f := range ch {
        f := f // https://golang.org/doc/faq#closures_and_goroutines
        wg.Go(func() error {
            // nolint: nestif
            if *checkonly {
                // Check if file extension is known
                lic, err := licenseHeader(f.path, t, data)
                if err != nil {
                    fmt.Printf("%s: %v\n", f.path, err)

                    return err
                }
                if lic == nil { // Unknown fileExtension
                    return nil
                }
                // Check if file has a license
                isMissingLicenseHeader, err := fileHasLicense(f.path)
                if err != nil {
                    fmt.Printf("%s: %v\n", f.path, err)

                    return err
                }
                if isMissingLicenseHeader {
                    fmt.Printf("%s\n", f.path)

                    return errors.New("missing license header")
                }
            } else {
                modified, err := addLicense(f.path, f.mode, t, data)
                if err != nil {
                    fmt.Printf("%s: %v\n", f.path, err)

                    return err
                }
                if *verbose && modified {
                    fmt.Printf("%s added license\n", f.path)
                }
            }

            return nil
        })
    }
    err := wg.Wait()
    close(done)
    if err != nil {
        os.Exit(1)
    }
}()

for _, d := range pflag.Args() {
    walk(ch, d)
}
close(ch)
<-done
```

这段代码乍一看挺多，其实理清思路还是比较容易理解的。

我们先理清这段代码的整体脉络：

```go
// process at most 1000 files in parallel
ch := make(chan *file, 1000)
done := make(chan struct{})
go func() {
    var wg errgroup.Group
    for f := range ch {
        wg.Go(func() error {
            ...
            return nil
        })
    }
    err := wg.Wait()
    close(done)
    if err != nil {
        os.Exit(1)
    }
}()

for _, d := range pflag.Args() {
    walk(ch, d)
}
close(ch)
<-done
```

这段代码设计还是比较精妙的，主 `goroutine` 与子 `goroutine` 通过 `ch` 和 `done` 进行协作。这也是典型的生产者消费者模型。

`ch := make(chan *file, 1000)` 创建了一个带缓冲的通道，缓冲大小为 1000，即最大并发为 1000。它用于将遍历到的文件（通过 `walk` 函数找到的文件）发送到消费者 `goroutine` 中。

`done := make(chan struct{})` 创建了一个无缓冲的通道，用于通知主 `goroutine` 所有并发任务（检查或修改文件）已经完成。

生产者 `goroutine` 遍历 `pflag.Args()` 的返回值并调用 `walk(ch, d)` 来将生产的数据传入 `ch`。`pflag.Args()` 调用会返回处理完标志后剩余的参数列表，类型为 `[]string`，即传进来的目录或文件。前面提到的 `pflag.NArg()` 返回几，`pflag.Args()` 返回的切片中就有几个值。

当生产者中的 `walk` 函数完成遍历所有目录并发送所有文件后，主 `goroutine` 会调用 `close(ch)` 关闭 `ch` 通道，通知接收方没有更多的文件需要处理。然后调用 `<-done` 阻塞，等待消费者 `goroutine` 发送过来的完成信号。

消费者 `goroutine` 中，`for f := range ch { ... }` 循环从 `ch` 通道接收文件（`*file` 类型），并为每个文件启动一个新的 `goroutine`（通过 `errgroup` 的 `Go` 方法管理并发任务）。如果你对 `errgroup` 不熟悉，可以参考后文附录部分对 `errgroup` 的讲解，了解其用法后再回过来接着分析代码。当 `ch` 通道被关闭，`for` 循环也就结束了。`wg.Wait()` 会等待所有消费 `goroutine` 处理完成并返回。然后调用 `close(done)` 关闭 `done` 通道。最后根据是否有 `goroutine` 返回 `error` 来决定是否调用 `os.Exit(1)` 进行异常退出。

当消费者 `goroutine` 关闭 `done` 通道时，生产者 `<-done` 会立即收到完成信号，由于这是 `main` 函数的最后一行代码，`<-done` 返回也就意味着整个程序执行完成并退出。

两个 `goroutine` 协同工作的主要逻辑已经解释清楚，我们就来分别看下二者的具体逻辑实现。

生产者 `goroutine` 主要逻辑都在 `walk` 函数中：

```go
func walk(ch chan<- *file, start string) {
	_ = filepath.Walk(start, func(path string, fi os.FileInfo, err error) error {
		if err != nil {
			fmt.Printf("%s error: %v\n", path, err)

			return nil
		}
		if fi.IsDir() {
			for _, pattern := range patterns.dirs {
				if pattern.MatchString(fi.Name()) {
					return filepath.SkipDir
				}
			}

			return nil
		}

		for _, pattern := range patterns.files {
			if pattern.MatchString(fi.Name()) {
				return nil
			}
		}

		ch <- &file{path, fi.Mode()}

		return nil
	})
}
```

`walk`接收两个参数 `ch` 通道以及遍历的起始目录 `start`。

其中 `ch` 通道中的 `file` 类型定义如下：

```go
type file struct {
	path string
	mode os.FileMode
}
```

`path`表示文件路径，`mode` 表示文件操作模式。

`walk`函数内部使用 `filepath.Walk` 来从 `start` 开始递归的遍历目录，并对其进行处理。如果你对 `filepath.Walk` 不熟悉，可以参考后文附录部分对 `filepath.Walk` 的讲解，了解其用法后再回过来接着分析代码。

这里处理逻辑也很简单，就是通过正则匹配，来过滤用户通过 `--skip-dirs` 和 `--skip-files` 两个标志传进来需要跳过的目录和文件。然后将需要处理的文件传递给 `ch` 通道等待消费者去处理。

> NOTE:
>
> 现在你知道为什么前文示例中的命令使用了正则 `--skip-dirs=^a$` 来跳过目录 `a`，而没有直接使用 `--skip-dirs=a` 了吗？对字符串 `a` 做 `pattern.MatchString` 会匹配到 `data`，所以程序才会跳过整个 `data` 目录，不再进一步遍历子目录。
>

当 `*file` 对象被传入 `ch` 通道，消费者就要开始工作了。

消费 `goroutine` 中主逻辑分两种情况：

1. 用户执行命令时输入了 `--check` 标志，只检查文件是否存在 License。
2. 需要添加 License 头信息的逻辑。

我们一个一个来看。

1. 用户执行命令时输入了 `--check` 标志，只检查文件是否存在 License，处理逻辑如下：

```go
if *checkonly {
    // Check if file extension is known
    lic, err := licenseHeader(f.path, t, data)
    if err != nil {
        fmt.Printf("%s: %v\n", f.path, err)

        return err
    }
    if lic == nil { // Unknown fileExtension
        return nil
    }
    // Check if file has a license
    isMissingLicenseHeader, err := fileHasLicense(f.path)
    if err != nil {
        fmt.Printf("%s: %v\n", f.path, err)

        return err
    }
    if isMissingLicenseHeader {
        fmt.Printf("%s\n", f.path)

        return errors.New("missing license header")
    }
}
```

首先调用 `licenseHeader` 函数来检查文件扩展名是否支持，它接收三个参数，分别是文件路径、License 模板、和 `data`，还记得 `data` 的内容吗？包含 `holder` 和 `year`，用来渲染模板。

`licenseHeader` 函数实现如下：

```go
func licenseHeader(path string, tmpl *template.Template, data *copyrightData) ([]byte, error) {
	var lic []byte
	var err error
	switch fileExtension(path) {
	default:
		return nil, nil
	case ".c", ".h":
		lic, err = prefix(tmpl, data, "/*", " * ", " */")
	case ".js", ".mjs", ".cjs", ".jsx", ".tsx", ".css", ".tf", ".ts":
		lic, err = prefix(tmpl, data, "/**", " * ", " */")
	case ".cc",
		".cpp",
		".cs",
		".go",
		".hh",
		".hpp",
		".java",
		".m",
		".mm",
		".proto",
		".rs",
		".scala",
		".swift",
		".dart",
		".groovy",
		".kt",
		".kts":
		lic, err = prefix(tmpl, data, "", "// ", "")
	case ".py", ".sh", ".yaml", ".yml", ".dockerfile", "dockerfile", ".rb", "gemfile":
		lic, err = prefix(tmpl, data, "", "# ", "")
	case ".el", ".lisp":
		lic, err = prefix(tmpl, data, "", ";; ", "")
	case ".erl":
		lic, err = prefix(tmpl, data, "", "% ", "")
	case ".hs", ".sql":
		lic, err = prefix(tmpl, data, "", "-- ", "")
	case ".html", ".xml", ".vue":
		lic, err = prefix(tmpl, data, "<!--", " ", "-->")
	case ".php":
		lic, err = prefix(tmpl, data, "", "// ", "")
	case ".ml", ".mli", ".mll", ".mly":
		lic, err = prefix(tmpl, data, "(**", "   ", "*)")
	}

	return lic, err
}
```

里面逻辑看起来 `case` 比较多，不过主要是为了支持各种编程语言的文件。

函数 `fileExtension` 用来获取文件扩展名：

```go
func fileExtension(name string) string {
	if v := filepath.Ext(name); v != "" {
		return strings.ToLower(v)
	}

	return strings.ToLower(filepath.Base(name))
}
```

然后根据不同的文件扩展名调用 `prefix` 函数渲染模板。

`prefix` 函数定义如下：

```go
func prefix(t *template.Template, d *copyrightData, top, mid, bot string) ([]byte, error) {
	var buf bytes.Buffer
	if err := t.Execute(&buf, d); err != nil {
		return nil, fmt.Errorf("render template failed, err: %w", err)
	}
	var out bytes.Buffer
	if top != "" {
		fmt.Fprintln(&out, top)
	}
	s := bufio.NewScanner(&buf)
	for s.Scan() {
		fmt.Fprintln(&out, strings.TrimRightFunc(mid+s.Text(), unicode.IsSpace))
	}
	if bot != "" {
		fmt.Fprintln(&out, bot)
	}
	fmt.Fprintln(&out)

	return out.Bytes(), nil
}
```

`prefix` 函数会根据不同编程语言的注释风格生成版权声明头信息。它需要传入 License 模板、版权信息（年份、作者）、开头、中间、结尾标识符。

所以我们调用 `lic, err := licenseHeader(f.path, t, data)`，最终得到的 `lic` 实际上内容根据文件类型是渲染后的 License 信息。

比如同一个 License 头信息，在不同编程语言文件中都要写成对应的注释形式，所以要入乡随俗。

在 Go 文件中 License 头信息长这样：

```go
// Copyright 2024 jianghushinian <jianghushinian007@outlook.com>. All rights reserved.
// Use of this source code is governed by a MIT style
// license that can be found in the LICENSE file. The original repo for
// this file is https://github.com/jianghushinian/blog-go-example.
```

在 Python 文件中 License 头信息则要长这样：

```python
# Copyright 2024 jianghushinian <jianghushinian007@outlook.com>. All rights reserved.
# Use of this source code is governed by a MIT style
# license that can be found in the LICENSE file. The original repo for
# this file is https://github.com/jianghushinian/blog-go-example.
```

接下来判断如果没拿到结果，说明是不支持的文件扩展名，直接返回不做进一步处理，逻辑如下：

```go
if lic == nil { // Unknown fileExtension
    return nil
}
```

之后调用 `fileHasLicense`检查文件是否包含授权头信息。`fileHasLicense` 函数实现如下：

```go
func fileHasLicense(path string) (bool, error) {
	b, err := ioutil.ReadFile(path)
	if err != nil {
		return false, err
	}

	if hasLicense(b) {
		return false, nil
	}

	return true, nil
}

func hasLicense(b []byte) bool {
	n := 1000
	if len(b) < 1000 {
		n = len(b)
	}

	return bytes.Contains(bytes.ToLower(b[:n]), []byte("copyright")) ||
		bytes.Contains(bytes.ToLower(b[:n]), []byte("mozilla public"))
}
```

这里实现比较简单，就是读取文件内容，然后判断前 1000 个字符中是否包含 `copyright` 或 `mozilla public` 关键字。

`fileHasLicense` 函数返回后，如果其返回值为 `true`，则说明文件中不包含 License 头信息，直接返回一个 `error`：

```go
if isMissingLicenseHeader {
    fmt.Printf("%s\n", f.path)

    return errors.New("missing license header")
}
```

这里返回的 `error` 会被 `err := wg.Wait()` 拿到，最终调用 `os.Exit(1)` 异常退出。

2. 处理需要添加 License 头信息的逻辑如下：

```go
else {
    modified, err := addLicense(f.path, f.mode, t, data)
    if err != nil {
        fmt.Printf("%s: %v\n", f.path, err)

        return err
    }
    if *verbose && modified {
        fmt.Printf("%s added license\n", f.path)
    }
}
```

这里调用 `addLicense` 函数为指定文件插入 License 头信息。

`addLicense` 函数实现如下：

```go
func addLicense(path string, fmode os.FileMode, tmpl *template.Template, data *copyrightData) (bool, error) {
	var lic []byte
	var err error
	lic, err = licenseHeader(path, tmpl, data)
	if err != nil || lic == nil {
		return false, err
	}

	b, err := ioutil.ReadFile(path)
	if err != nil {
		return false, err
	}
	if hasLicense(b) {
		return false, nil
	}

	line := hashBang(b)
	if len(line) > 0 {
		b = b[len(line):]
		if line[len(line)-1] != '\n' {
			line = append(line, '\n')
		}
		line = append(line, '\n')
		lic = append(line, lic...)
	}
	b = append(lic, b...)

	return true, ioutil.WriteFile(path, b, fmode)
}
```

首先这里也调用了 `licenseHeader` 来判断文件扩展名是否被支持，并渲染 License 模板。

然后调用 `hasLicense` 来判断文件是否已经存在 License 头信息。

如果文件不存在 License 头信息，接下来的逻辑就是正式准备写入 License 头信息了。

接下来这段逻辑分两种情况，首先调用 `hashBang` 函数用来判断文件是否存在 [Shebang](https://zh.wikipedia.org/wiki/Shebang) 行，如果有 `Shebang` 行，则源文件内容为 `Shebang` 行 + 代码，新内容为 `Shebang` 行 + License 头信息 + 代码。如果没有 `Shebang` 行存在，则源文件内容只包含代码，新内容为 License 头信息 + 代码。

`hashBang` 函数内容如下：

```go
var head = []string{
	"#!",                       // shell script
	"<?xml",                    // XML declaratioon
	"<!doctype",                // HTML doctype
	"# encoding:",              // Ruby encoding
	"# frozen_string_literal:", // Ruby interpreter instruction
	"<?php",                    // PHP opening tag
}

func hashBang(b []byte) []byte {
	line := make([]byte, 0, len(b))
	for _, c := range b {
		line = append(line, c)
		if c == '\n' {
			break
		}
	}
	first := strings.ToLower(string(line))
	for _, h := range head {
		if strings.HasPrefix(first, h) {
			return line
		}
	}

	return nil
}
```

最后这段逻辑就简单了：

```go
if *verbose && modified {
    fmt.Printf("%s added license\n", f.path)
}
```

这里用来处理 `-v/--verbose` 参数。

至此，`addlicense` 所有源码就都解读完成了。

### 总结

本文介绍可一行命令为项目文件添加开源协议头的工具 `addlicense`，并且还对其源码进行了逐行解读，让你能够知其然，也能知其所以然。

不过 `addlicense` 工具能力还比较有限，比如不支持跳过 `a/b/c` 这种嵌套目录，再比如 `hashBang` 函数支持有限，不支持 Python3 的 `# -*- coding:utf-8 -*-` 等。

如果感兴趣，你可以一起投入到项目建设中来，为这个工具提供更强大的能力，欢迎提交 [PR](https://github.com/superproj/addlicense/pulls)。

本文示例源码我都放在了 [GitHub](https://github.com/jianghushinian/blog-go-example/tree/main/tools/addlicense) 中，欢迎点击查看。

希望此文能对你有所启发。

### 附录

#### filepath.Walk

`filepath.Walk` 是 Go 标准库中的一个函数，用来递归遍历文件系统中的目录和文件。它可以遍历指定目录下的所有文件和子目录，并对每个文件或目录执行用户提供的回调函数。

##### 基本语法

`Walk` 函数签名如下：

```go
func Walk(root string, fn WalkFunc) error
```

`Walk` 接收两个参数：

+ `root`：需要递归遍历的起始目录路径。
+ `fn`：每次遍历到一个文件或目录时调用的回调函数。

`Walk` 遍历以 `root` 为根的文件树，并为树中的每个文件或目录（包括 `root`）调用 `fn` 函数。

`WalkFunc` 函数签名如下：

```go
type WalkFunc func(path string, info fs.FileInfo, err error) error
```

`WalkFunc` 接收三个参数：

+ `path`：当前文件或目录的路径。
+ `info`：当前文件或目录的 `fs.FileInfo`，这里包含了文件的元信息，如是否为目录、文件大小等。
+ `err`：错误信息，如权限问题。

该函数返回的错误结果会控制 `Walk` 是否继续执行。如果函数返回特殊值 `filepath.SkipDir`，则 `Walk` 会跳过当前目录（如果 `path` 是目录跳过当前目录，否则跳过 `path` 的父目录）但继续遍历其他内容。如果函数返回特殊值 `filepath.SkipAll`，则 `Walk` 将跳过所有剩余的文件和目录。否则，如果函数返回非 `nil` 错误，则 `Walk` 将完全停止并返回该错误。

##### 使用示例

现在我们准备如下用来测试的目录：

```bash
$ tree data -a
data
├── .git
├── a
│   ├── main.go
│   └── main_test.go
├── b
│   └── c
│       └── keep
├── d.go
└── d_test.go

5 directories, 5 files
```

我们来使用 `Walk` 遍历 `data` 目录，并且输出每个文件或目录的路径。此外，需要跳过名为 `.git` 的目录和以 `test.go` 结尾的 Go 测试文件。

示例代码如下：

```go
package main

import (
	"fmt"
	"os"
	"path/filepath"
	"strings"
)

func main() {
	// 定义起始目录
	root := "./data"

	// 调用 Walk 函数遍历目录
	err := filepath.Walk(root, func(path string, info os.FileInfo, err error) error {
		if err != nil {
			// 如果发生错误，则输出错误并继续遍历
			fmt.Printf("Error accessing path %s: %v\n", path, err)
			return nil
		}

		// 跳过名为 `.git` 的目录
		if info.IsDir() && info.Name() == ".git" {
			fmt.Printf("Skipping directory: %s\n", path)
			return filepath.SkipDir
		}

		// 跳过 Go 测试文件
		if !info.IsDir() && strings.HasSuffix(info.Name(), "test.go") {
			fmt.Println("Skipping file:", path)
			return nil
		}

		// 输出每个文件或目录的路径
		fmt.Println(path)
		return nil
	})

	if err != nil {
		fmt.Printf("Error walking the path %v\n", err)
	}
}
```

通过 `info.IsDir()` 可以判断是否为目录，`info.Name()` 可以获取文件或目录名。

使用 `strings.HasSuffix()` 函数可以过滤出 `.go` 文件。

执行示例代码，得到输出如下：

```bash
$ go run main.go 
./data
Skipping directory: data/.git
data/a
data/a/main.go
Skipping file: data/a/main_test.go
data/b
data/b/c
data/b/c/keep
data/d.go
Skipping file: data/d_test.go
```

#### errgroup

`errgroup` 是 [Go 官方库 x](https://go.dev/wiki/X-Repositories) 中提供的一个非常实用的工具，用于并发执行多个 goroutine，并且方便的处理错误。

##### 使用场景

1. **并发处理多个任务**：当需要并发执行多个任务时，`errgroup` 有助于管理这些任务。
2. **收集错误**：`errgroup` 会在任何一个 goroutine 出现错误时收集并返回这个错误，避免手动处理 goroutine 的错误。
3. **等待所有 goroutine 完成**：`errgroup` 提供了一个简便的方法等待所有并发的 goroutine 完成。

##### 基本使用

`errgroup` 基本使用套路如下：

1. 导入 `errgroup` 包。
2. 创建一个 `errgroup.Group` 实例。
3. 使用 `Group.Go` 方法启动多个并发任务。
4. 使用 `Group.Wait` 方法等待所有 goroutine 完成或有一个返回错误。

##### 使用示例

我们有 10 个并发任务用 `errgroup` 来管理，示例代码如下：

```go
package main

import (
	"errors"
	"fmt"

	"golang.org/x/sync/errgroup"
)

func main() {
	var g errgroup.Group
	for i := 0; i < 10; i++ {
		i := i
		g.Go(func() error {
			if i == 3 {
				return errors.New("task 3 failed")
			}
			if i == 5 {
				return errors.New("task 5 failed")
			}

			// 其他任务继续运行
			fmt.Printf("run task %d\n", i)

			return nil // 正常返回 nil 表示成功
		})
	}
	if err := g.Wait(); err != nil {
		fmt.Printf("Error: %v\n", err)
	}
}
```

代码解析：

1. `var g errgroup.Group`: 创建了一个 `errgroup.Group` 对象，它用于管理多个 goroutine 并跟踪它们的状态。
2. `g.Go(func() error {...})`: 每次调用 `g.Go`，都会启动一个新的 goroutine，传入的匿名函数是任务的执行内容。`Go` 方法会记录这个任务的返回值（`error` 类型）。
3. **并发执行任务**：在 `g.Go` 内部执行的 `func() error` 都会并发执行。
4. `g.Wait()`: `g.Wait` 会等待所有的 goroutine 执行完成。如果所有任务都执行成功，它会返回 `nil`，否则，无论有几个 goroutine 执行失败，它会**返回第一个出现的错误**。示例中第 3 个任务和第 5 个任务出错，**其他的 8 个任务不会受到影响，它们依然会继续运行并完成**。

执行示例代码，得到输出如下：

```bash
$ go run main.go 
run task 9
run task 4
run task 2
run task 6
run task 7
run task 1
run task 8
run task 0
Error: task 3 failed
```

由于任务是并发执行，所以多次执行输出结果顺序可能不同。

并且，输出错误可能是 `Error: task 3 failed`，也有可能是 `Error: task 5 failed`。

这里还有一个更加真实的改编自 `errgroup` [官方文档](https://pkg.go.dev/golang.org/x/sync@v0.8.0/errgroup#example-Group-JustErrors)的示例，用来并发请求多个 URL 并输出响应状态码。

你可以再来感受下 `errgroup` 的用法，代码如下：

```go
package main

import (
	"fmt"
	"net/http"
	"sync"

	"golang.org/x/sync/errgroup"
)

func main() {
	g := new(errgroup.Group)
	var urls = []string{
		"http://www.golang.org/",
		"http://www.google.com/",
		"http://www.somestupidname.com/", // 这是一个错误的 URL，会导致任务失败
	}

	// 创建一个 map 来保存结果
	var result sync.Map

	// 启动多个 goroutine，并发处理多个 URL
	for _, url := range urls {
		// NOTE: 注意这里的 url 需要传递给闭包函数，避免闭包共享变量问题，自 Go 1.22 开始无需考虑此问题
		url := url // https://golang.org/doc/faq#closures_and_goroutines

		// 启动一个 goroutine 来获取 URL
		g.Go(func() error {
			resp, err := http.Get(url)
			if err != nil {
				return err // 发生错误，返回该错误
			}
			defer resp.Body.Close()

			// 保存每个 URL 的响应状态码
			result.Store(url, resp.Status)
			return nil
		})
	}

	// 等待所有 goroutine 完成
	if err := g.Wait(); err != nil {
		// 如果有任何一个 goroutine 返回了错误，这里会得到该错误
		fmt.Println("Error: ", err)
	}

	// 所有 goroutine 都执行完成，遍历并打印成功的结果
	result.Range(func(key, value any) bool {
		fmt.Printf("%s: %s\n", key, value)
		return true
	})
}
```

你可能注意到示例代码中有一句 `url := url`，这是由于在 Go 1.22 以前，由于 `for` 循环声明的变量只会被创建一次，并在每次迭代时更新。所以为了避免多个 `goroutine` 中拿到相同的 `url` 值，而进行的拷贝操作。

在 Go 1.22 中，循环的每次迭代都会创建新的变量，以避免意外的共享错误。这在 Go 1.22 [Release Notes](https://go.dev/doc/go1.22#language) 中有说明。

执行示例代码，得到输出如下：

```bash
$ go run main.go
Error:  Get "http://www.somestupidname.com/": dial tcp: lookup www.somestupidname.com: no such host
http://www.google.com/: 200 OK
http://www.golang.org/: 200 OK
```

**延伸阅读**

+ 开源协议简介：[https://jianghushinian.cn/2023/01/15/open-source-license-introduction/](https://jianghushinian.cn/2023/01/15/open-source-license-introduction/)
+ Go 命令行参数解析工具 pflag 使用：[https://jianghushinian.cn/2023/03/27/use-of-go-command-line-parameter-parsing-tool-pflag/](https://jianghushinian.cn/2023/03/27/use-of-go-command-line-parameter-parsing-tool-pflag/)
+ filepath.Walk 文档：[https://pkg.go.dev/path/filepath@go1.23.1#Walk](https://pkg.go.dev/path/filepath@go1.23.1#Walk)
+ errgroup 文档：[https://pkg.go.dev/golang.org/x/sync@v0.8.0/errgroup](https://pkg.go.dev/golang.org/x/sync@v0.8.0/errgroup)
+ Go FAQ: What happens with closures running as goroutines：[https://go.dev/doc/faq#closures_and_goroutines](https://go.dev/doc/faq#closures_and_goroutines)
+ Go 1.22 Release Notes：[https://go.dev/doc/go1.22#language](https://go.dev/doc/go1.22#language)
+ Go Wiki: X-Repositories：[https://go.dev/wiki/X-Repositories](https://go.dev/wiki/X-Repositories)
+ Shebang：[https://zh.wikipedia.org/wiki/Shebang](https://zh.wikipedia.org/wiki/Shebang)
+ addlicense 源码：[https://github.com/superproj/addlicense](https://github.com/superproj/addlicense)
+ 本文 GitHub 示例代码：[https://github.com/jianghushinian/blog-go-example/tree/main/tools/addlicense](https://github.com/jianghushinian/blog-go-example/tree/main/tools/addlicense)

**联系我**

+ 公众号：[Go编程世界](https://jianghushinian.cn/about)
+ 微信：jianghushinian
+ 邮箱：[jianghushinian007@outlook.com](mailto:jianghushinian007@outlook.com)
+ 博客：[https://jianghushinian.cn](https://jianghushinian.cn)
