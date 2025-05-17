---
title: '一行命令统计代码行数'
date: 2025-05-17 17:50:24
tags:
- 工具
categories:
- 工具
---

我在网上冲浪时，在 GitHub 上发现了一个感兴趣的开源项目 [OneX](https://github.com/onexstack/onex) ，我将其下载到本地，现在我该如何知道这个项目的体量呢？一个非常直观的指标是看这个项目有多少行代码。

<!-- more -->

我们可以使用如下命令，来统计 OneX 项目代码行数：

```bash
$ cd onex
$ find . -name "*.go" -print | xargs wc -l | tail -n 1
  155784 total
```

截止目前，这个项目共计 155784 行 Go 代码。

本文完。

### scc

开个玩笑，接下来我将介绍一款强大的统计代码行数的工具 [scc](https://github.com/boyter/scc) 。

scc 是 Sloc、Cloc 和 Code 三个单词的首字母缩写，scc 是一款用 Go 编写的速度飞快且准确的代码统计工具，并且支持复杂度计算和成本估计。

可以通过如下命令来安装 scc：

```bash
$ go install github.com/boyter/scc/v3@latest
```

如果你是 Mac 电脑，也可以使用 Homebrew 来安装 scc：

```bash
$ brew install scc
```

使用 scc 统计 OneX 项目代码行数：

![scc](scc.png)

scc 使用非常简单，只需要进入到项目目录，然后执行 `scc` 命令，即可实现统计。

scc 支持多种输出格式，如 `tabular`（表格）、`json`、`csv`、`html`、`sql` 等，默认以表格的形式在命令行输出统计结果，其输出字段依次为：语言、文件数、总行数、空白行、注释行、代码行、复杂度。

以下是针对 `scc` 代码统计结果的简单分析：

**一、核心指标解读**

1. 语言分布与代码量
    - Go 语言主导：占比 76.5%（155,784/203,648 行），文件数 1,220 个，复杂度 11,382，是项目核心开发语言。
    - 非代码文件占比：
        * YAML/Markdown：占 9.3%（(12,390+6,510)/203,648 行），多为配置文件和文档，说明此项目文档齐全。
        * Shell/Dockerfile/Makefile：占 7.9%（(13,346+778+1,920)/203,648 行），反映项目实现了容器化且自动化程度高。
    - 代码总量超过 14+ 万行，说明这是一个大而全的超级项目。
2. 代码质量指标
    - 注释率：全局注释占比 16.2%（32,933/203,648 行），Go 语言注释率 15.3%（23,859/155,784 行），符合健康标准（通常建议 10-20%）。
    - 复杂度分析：
        * Go 复杂度：平均每个文件 9.3 行（11,382/1,220 行）复杂代码。
        * Shell 复杂度：平均每个文件 9.6 行（789/82 行）复杂代码，说明可能存在较为复杂的嵌套逻辑。

**二、项目规模估算**

1. COCOMO 模型结果
    - 成本估算：
        * 开发成本：$5,124,210。
        * 时间估算：25.59 个月。
        * 需要人数：需 17.79 人团队。

![cost](cost.png)

当然，关于成本估算，我们可以通过 `--avg-wage` 参数修改人员的年平均工资（默认为 `$56286`）；还可以通过参数 `--cocomo-project-type` 修改模型（默认为 `organic`），可以选择 `organic, semi-detached, embedded, "custom,1,1,1,1"` 其中的一项。

2. 实际代码效率
    - 千行代码成本：$25.16（总成本 / 总代码行数），反映代码质量较高或团队效率较好。

以上便是 scc 输出结果的基本解读。

scc 支持非常多的可选参数，你可以通过命令 `scc -h` 获得帮助。这里简单介绍几个常用选项：

```bash
$ scc ./src # 统计指定路径
$ scc -l # 查看支持的语言和扩展名
$ scc -i go,java  # 仅统计 Go 和 Java 文件
$ scc -f json # JSON 格式输出
$ scc -f html > report.html # 生成可视化 HTML 报告
$ scc --format-multi="html:report.html,json:data.json" # 同时输出多种格式，并写入文件
$ scc --avg-wage=60000  # 自定义平均工资（默认 $56,286 美元）
```

此外，如果你想在统计 Go 代码时忽略以 `*_test.go` 结尾的测试文件，则可以结合 `git` 来时先。

首先在 `.gitignore` 文件中加入如下配置项：

```git
**/*_test.go
```

然后，再次执行 `scc` 命令即可在统计时忽略所有以 `*_test.go` 结尾的测试文件。

因为 `scc` 默认就会读取 `.gitignore` 中的配置项，如果你想忽略 `.gitignore` 文件的作用，则可以执行如下命令：

```bash
# 指定参数 --no-gitignore 忽略 .gitignore 文件产生的副作用
$ scc --no-gitignore
```

### cloc

除了 scc 工具，其实还有一个老牌的更加流行的工具 cloc 也可以用来统计代码。cloc 使用 Perl 语言编写，以下是使用 cloc 工具统计 OneX 项目代码行数的输出结果：

![cloc](cloc.png)

可以发现，与 scc 输出结果略有差异，不过整体来说都比较准确。cloc 最大的问题是慢，从速度角度来说，确实 scc 速度更快，体验更好。针对 OneX 项目代码统计，scc 几乎瞬间完成，而 cloc 则需要 40+ 秒才统计完成。

### 总结

scc 是一款用 Go 编写的速度飞快且准确的代码统计工具，并且支持复杂度计算和成本估计。通过使用 scc，一行命令我们就可以统计代码行数。

以下是我用 Hunyuan 大模型帮我生成的 scc 与其他主流代码统计工具的对比表格：

| 工具 | 语言 | 核心功能 | 优势 | 局限性 |
| --- | --- | --- | --- | --- |
| scc | Go | 代码行数、复杂度、COCOMO 估算 | 速度最快（百万行级处理） | 功能相对单一 |
| cloc | Perl | 代码行数、注释、空白行 | 支持更多非代码文件（如二进制） | 统计速度较慢 |
| tokei | Rust | 语言分类统计 | 支持 200+ 语言，内存占用低 | 无复杂度分析功能 |


仅供参考。

希望此文能对你有所启发。

**延伸阅读**

+ OneX GitHub 源码：[https://github.com/onexstack/onex](https://github.com/onexstack/onex)
+ scc GitHub 源码：[https://github.com/boyter/scc](https://github.com/boyter/scc)
+ cloc GitHub 源码：[https://github.com/AlDanial/cloc](https://github.com/AlDanial/cloc)
+ tokie GitHub 源码：[https://github.com/XAMPPRocky/tokei](https://github.com/XAMPPRocky/tokei)
+ 构造性成本模型（COCOMO）：[https://zh.wikipedia.org/wiki/构造性成本模型](https://zh.wikipedia.org/wiki/%E6%9E%84%E9%80%A0%E6%80%A7%E6%88%90%E6%9C%AC%E6%A8%A1%E5%9E%8B)
+ 本文永久地址：[https://jianghushinian.cn/2025/05/17/scc/](https://jianghushinian.cn/2025/05/17/scc/)

**联系我**

+ 公众号：[Go编程世界](https://jianghushinian.cn/about)
+ 微信：jianghushinian
+ 邮箱：[jianghushinian007@outlook.com](mailto:jianghushinian007@outlook.com)
+ 博客：[https://jianghushinian.cn](https://jianghushinian.cn)
+ GitHub：[https://github.com/jianghushinian](https://github.com/jianghushinian)
