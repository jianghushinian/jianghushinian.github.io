---
title: 开源协议简介
date: 2023-01-15 10:09:11
tags: 开源
---

开源协议是每一个想要做开源软件的开发者都需要了解的，即使你不想做开源软件，那么当你使用他人开源的软件时也需要了解一些开源协议相关的内容，这样能够尽量避免一些不必要的麻烦。

<!-- more -->

### 什么是开源

开源即开放源代码，是 [OSI (Open Source Initiative)](https://opensource.org/) 这个组织提出来的。而被开源的软件，我们通常称为开源软件。你可能还见到过 `Free Software` 一词，它代表 `自由软件` 而非 `免费软件`，是开源软件的前身。

开源软件 = 开放源代码 + 开源协议，一份没有添加开源协议的开源代码，并不是真正的开源软件，也就不能随意使用。

### 开源许可证

开源协议是指开源软件所携带的一份声明协议，这份协议也叫开源许可证。开源许可证声明了开源协议的内容，规定了**原作者和使用者的权利以及义务**。

开源许可证是具有法律效力的，并且需要得到 OSI 这个组织的认证，目前 OSI 共计认证了 [110+](https://opensource.org/licenses/category) 个开源许可证，这些被认证的开源许可证都必须遵循 [OSD (Open Source Definition)](https://opensource.org/osd) 规则。

虽然开源许可证非常多，但常用的就那么几种，下面就主要介绍下几种常见开源许可证。

开源许可证分成两大类：宽松式许可证（Permissive Licenses）、反版权许可证（Copyleft Licenses）。

#### 宽松式许可证（Permissive Licenses）

顾名思义，这类开源许可证比较宽松，限制更少。常见宽松开源许可证有：

- [BSD (2-Clause)](https://opensource.org/licenses/BSD-2-Clause) (Berkeley Software Distribution，伯克利软件发行版)

    源代码或二进制形式的重新分发，必须保留原始的许可证声明。

- [BSD (3-Clause)](https://opensource.org/licenses/BSD-3-Clause)

    在 BSD(2-Clause) 基础上增加了一条，禁止使用原始作者的名字为衍生软件进行促销。

    Go 语言就在使用 BSD (3-Clause) 开源许可证。

- [MIT](https://opensource.org/licenses/MIT) (Massachusetts Institute of Technology，麻省理工学院许可证)

    免费授予任何人该软件及相关文档的权限，包括但不限于使用、复制、修改、合并、发表、分发、再授权、出售软件的副本。分发软件时，必须保留原始的许可证声明。

    MIT 是最为宽松的开源许可证，所以这也使得它成为最流行的开源许可证，如目前在前端领域非常有名的 Vue.js 就在使用它。

- [Apache-2.0](https://opensource.org/licenses/Apache-2.0) (Apache 软件基金会发布的许可证)

    Apache 许可证内容非常多，不过可以简单的总结几点：

    分发软件时，必须保留原始的许可证声明。

    所有修改过的文件，必须加以说明告知用户此文件已被更改。

    没有修改过的文件，不得修改许可证。

    云原生领域著名软件 Kubernetes 使用的正是 Apache-2.0 开源许可证。

#### 反版权许可证（Copyleft Licenses）

Copyleft 一词由 理查德·斯托曼 发明，表示 Copyright (版权) 的反义词。Copyright 表示不经许可，用户无权复制，商业软件开发人员通过 Copyright 剥夺了用户的自由。Copyleft 则表示不经许可，用户有权复制，Copyleft 使用版权来给予用户自由。

Copyleft 类的许可证要比 Permissive 许可证限制更多。常见 Copyleft 开源许可证有：

- [GPL](https://opensource.org/licenses/gpl-license) (GNU General Public License)

    GPL 有两个版本，GPL-2.0 和 GPL-3，同 BSD 一样，更高版本会带来更多的限制。GPL 协议内容也非常多，我们最需要关注的一点是：使用了 GPL 协议的开源软件，其衍生软件如果需要分发，就必须开源并且同样要使用此协议。
    
    由于这条规定的存在，有人甚至把 GPL 协议称为 "GPL 病毒"，因为它具有跟病毒一样的传染性。不过 GPL 仍然是非常流行的开源许可证，比如大名鼎鼎的 Linux 就采用了 GPL 协议。

    GPL 是流行开源许可证中最为严格的，所以对于使用开源软件所衍化的商业化软件就不够友好了。

- [LGPL](https://opensource.org/licenses/lgpl-license) (GNU Library General Public License)

    算是 GPL 的一个变种，主要为类库使用而设计的开源协议。

    商用软件如果采用类库方式引用使用了 LGPL 协议的开源软件，则可以不用开源。

    如果是修改或衍生软件需要分发，则必须开源并且同样要使用此协议。这点与 GPL 协议一样。

- [MPL-2.0](https://opensource.org/licenses/MPL-2.0) (Mozilla Public License 2.0，Mozilla 基金会发布的许可证)

    MPL 融合了 BSD 开源许可证 和 GPL 开源许可证 的特性，力争在专有软件和开源软件开发者之间寻求平衡。是比 BSD 更严格，比 GPL 更宽松的开源许可证。

    MPL 允许新增的独立代码文件闭源，但在 MPL 授权下的代码文件必须保持 MPL 授权且开源。这使得 MPL 既不像 MIT 和 BSD 那样允许派生作品完全转化为闭源，也不像 GPL 那样要求所有的派生作品，包括新的组件在内，必须全部保持使用 GPL。

    Mozilla 自家的 Firefox 浏览器就使用此开源许可证。

另外值得一提的是，在 OSI 官网公布的开源许可证列表中，有一个叫「[木兰（Mulan PSL2）](https://opensource.org/licenses/MulanPSL-2.0)」的开源许可证，它是中国本土唯一获得 OSI 认可的开源许可证。Mulan PSL2 以中英文双语表述，中英文版本具有同等法律效力。如果中英文版本存在任何冲突不一致，以中文版为准。

「木兰」并不是一个许可证，而是一系列许可证，它包含木兰宽松许可证、木兰公共许可证、木兰开放作品许可协议。其中木兰宽松许可证第 2 版（Mulan PSL2）在 2020 年 2 月 14 日通过 OSI 批准。

如果你想使用一个中文的开源许可证，那么 Mulan PSL2 目前是你唯一的选择。

#### 使用开源许可证

以 MIT 为例，我们来学习下如何在自己的开源项目中使用开源许可证：

1. 首先我们需要在自己的开源项目根目录下创建一个叫 `LICENSE` 的文本文件，注意文件名不包含任何后缀。

2. 然后去到 OSI 官网找到 [MIT](https://opensource.org/licenses/MIT) 开源许可证模板，内容如下：

```text
Copyright <YEAR> <COPYRIGHT HOLDER>

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
```

3. 将开源许可证模板内容复制到 `LICENSE` 文本文件中，并将第一行 `Copyright` 后面的 `<YEAR>` 替换为当前年份，将 `<COPYRIGHT HOLDER>` 替换为自己的名字。

4. 当开放项目源代码时，将此文件一同开放出去即可。如果你使用 GitHub 开放源码，则只需要将此 `LICENSE` 加入到 git 管理即可。

我开源的 [todo-list](https://github.com/jianghushinian/todo-list/blob/master/LICENSE) 项目就是使用 MIT 协议，你可以作为参考。

如果你是在 GitHub 上新建开源项目，在创建项目界面，有一个 `Choose a license` 按钮可以很方便的选择一款开源协议，并且 GitHub 会自动替换许可证模板中的年份、作者等信息。

![GitHub Choose a license](Create-a-New-Repository.png)

另外，我们在开放源代码时，其实可以不使用 OSI 认证的开源许可证，而是选择自己写一份许可证，用来声明版权。这同样是具有法律效力的，不过这份许可证就不能叫做开源许可证了。

### 如何选择开源许可证

乌克兰程序员 Paul Bagwell 画了[一张图](https://web.archive.org/web/20110503183702/http://pbagwl.com/post/5078147450/description-of-popular-software-licenses)在网上很是流行，阮一峰老师将其翻译成了[中文](https://www.ruanyifeng.com/blog/2011/05/how_to_choose_free_software_licenses.html)，我自己也画了一份中文版本：

![choose a licence](choose-licence.png)

> 注：
> 1. 关于什么是许可证兼容性可以参考：<a href="开源许可证兼容性指南.docx" target="_blank">《开源许可证兼容性指南》</a>
> 2. 原图中 `GPL V3` 和 `GPL v2` 的主要区别是是否禁止 DRM(Digital Rights Management)，但根据我查到的[资料](https://www.gnu.org/licenses/gpl-faq.html#DRMProhibited)，`GPL V3` 并不禁止 DRM，如果有人破解了 DRM，那么破解者有权利分发他的软件，而不受 DMCA(Digital Millennium Copyright ACT) 或类似法律的威胁。

从上图中可以看出，大体上左边的许可证比较严格，右边的许可证较为宽松。此图虽然不够严谨，胜在方便理解。在开源自己的项目时，可以根据此图快速选择出适合自己的开源许可证。

### 知识共享许可证

有时候，我们想要开源的并不是一款软件，而是一套开源的教程或者书籍等，此时严格来讲并不能使用上面所介绍的开源许可证。

在 [OSD 第 2 条](https://opensource.org/osd)中有规定：开源软件是必须要包含源代码的。

![](source-code.png)

也就是说，教程或者书籍等没有源代码，并不能作为开源软件，也就不能使用开源许可证。

此类项目想要开源，应该使用「知识共享许可证」（creative commons licenses），通常也叫 CC 许可证。

CC 许可证由 [Creative Commons 基金会](https://creativecommons.org/)提出，虽然没有得到 OSI 的认可，但他仍具有法律效力，并且应用广泛。

上面提到的「木兰开放作品许可协议」就是对标知识共享许可证的。同木兰许可证类似，知识共享许可证也是一系列许可证，目前最新的知识共享许可证为 4.0 版本，常见的许可证有 6 种：

- CC BY 4.0 (Attribution 4.0 International，署名 4.0 国际)

- CC BY-SA 4.0 (Attribution-ShareAlike 4.0 International，署名-相同方式共享 4.0 国际)

- CC BY-ND 4.0 (Attribution-NoDerivatives 4.0 International，署名-禁止演绎 4.0 国际)

- CC BY-NC 4.0 (Attribution-NonCommercial 4.0 International，署名-非商业性使用 4.0 国际)

- CC BY-NC-SA 4.0 (Attribution-NonCommercial-ShareAlike 4.0 International，署名-非商业性使用-相同方式共享 4.0 国际)

- CC BY-NC-ND 4.0 (Attribution-NonCommercial-NoDerivatives 4.0 International，署名-非商业性使用-禁止演绎 4.0 国际)

可以发现，CC 许可证命名方式就是它的权利简拼组合。以下是对其中出现的几个名词的解释：

署名：必须给出原作者的署名，提供指向本许可协议的链接，同时标明是否对原始作品作了修改。

非商业性使用：您不得将本作品用于商业目的。

相同方式共享：在任何媒介以任何形式复制、发行本作品时必须采用相同的许可证。

禁止演绎：禁止修改、转换或以本作品为基础进行创作。

非商业性使用：不得用于盈利性目的。

之所以每个许可证后面都带有国际两个字，是因为这系列许可证发布了不同的地域版，不过国际版更为通用。

需要注意 CC 系列许可证一旦发布，就不可收回，只要你遵守许可协议条款，许可人就无法收回你的这些权利。

如需使用 CC 许可证，可以参考我的博客[首页底部](https://jianghushinian.cn/)位置使用示例。

<div style="text-align: center"><img src="cc.png" width="100%" style="max-width: 400px;"></div>

### 开源案例

介绍完了开源协议，我们再来看一个开源案例：

#### 中国首例因违反 GPL 协议致侵害计算机软件著作权纠纷

2021-06-30 在中国裁判文书网上公布了一则民事判决书，标题为：「济宁市罗盒网络科技有限公司诉被告福建风灵创景科技有限公司(以下简称福建风灵公司)、被告北京风灵创景科技有限公司(以下简称北京风灵公司)、被告深圳市腾讯计算机系统有限公司(以下简称腾讯公司)侵害计算机软件著作权纠纷一审民事判决书」。案件概况如下：

原告济宁市罗盒网络科技有限公司独立开「罗盒（VirtualApp）」从 2016 年 7 月 8 日的版本开始引入开源协议，起初为 LGPL3.0 协议，从 2016 年 8 月 12 日开始更换为 GPL3.0 协议。2017 年 10 月 29 日开始删除适用 GPL3.0 协议的表述，但英文介绍中仍保留`openplatform` 的表述。

2018 年 9 月，原告调查发现名为「点心桌面」的软件使用了 VirtualApp 的代码，将两个软件源代码进行分析比对，两者间 421 个可比代码中有 308 个代码具有实质相似性，有 27 个代码具有高度相似性，有 78 个代码具有一般相似性。因此，被诉侵权软件与涉案软件构成实质相似。

经查，被诉「点心桌面」中使用了原告采用 GPL3.0 协议发布的 VirtualApp，被告对此亦予以确认。

原告申请赔偿 2000 万，最终，法院酌情确定赔偿数额为 50 万元。原告为制止本案侵权行为所支出的合理费用，计算在赔偿损失数额范围之内。

更多细节可以<a href="宁市罗盒网络科技有限公司诉被告福建风灵创景科技有限公司以下简称福建风灵公司被告北京风灵创景科技有限....docx" target="_blank">点击下载</a>查看。

该案例给开源软件使用者敲响一记警钟，使用开源软件一定要查看并遵循开源许可证。

### 总结

本文带大家一起认识了什么是开源协议，并且还对常用开源协议进行了分析，以及如何使用开源协议。同时讲解了针对书籍等开源作品使用的知识共享许可协议和使用方式。最终分享了一个开源软件纠纷案例，以说明了解开源协议的重要性。

此文仅为作者本人学习并整理的开源协议相关知识，即不够全面，也不够严谨，不能作为法律依据。希望你能通过本篇文章认识并重视开源协议，学习和书写本篇文章时间有限，难免出现表达不够准确或错误的地方，欢迎批评指正。

最后，想提醒大家，身为一名开发者，掌握开源协议是有必要的。不过开源协议的内容非常多且专业，想要完全了解也是一项繁重的工作，毕竟这不是我们的专业领域，如果遇到无法确定的问题，可以寻求身边的专业法务帮忙。

**参考**

> https://opensource.org/osd
> https://opensource.org/licenses/category
> https://zh.m.wikipedia.org/zh-hans/开源软件
> https://zh.wikipedia.org/zh-cn/自由软件
> https://zh.m.wikipedia.org/wiki/Mozilla公共许可证
> https://jxself.org/translations/gpl-3.zh.shtml
> https://web.archive.org/web/20110503183702/http://pbagwl.com/post/5078147450/description-of-popular-software-licenses
> https://www.ruanyifeng.com/blog/2011/05/how_to_choose_free_software_licenses.html
> https://www.ruanyifeng.com/blog/2017/10/open-source-license-tutorial.html
> https://www.ruanyifeng.com/blog/2008/04/creative_commons_licenses.html
> https://opensource.com/article/17/9/open-source-licensing
> https://creativecommons.org/licenses/by-nc-sa/4.0/deed.zh
> https://license.coscl.org.cn/
> https://choosealicense.rustwiki.org/
> https://wenshu.court.gov.cn/
> https://www.oschina.net/news/159435
