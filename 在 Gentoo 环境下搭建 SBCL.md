title: 在 Gentoo 环境下搭建 SBCL
date: 2014-01-12
---

**何谓 SBCL** 粗率的说， SBCL （全名为 Steel Bank Common Lisp 是目前较为活跃的 Common Lisp 编译器和运行环境。可扩展与多语言覆盖等功能受到了广大 Lisp 爱好者的青睐。

目前笨人在学习 Lisp 语言，也来玩个。但是我是用 Gentoo Linux 安装所以就…

<!-- more -->

首先用 Gentoo 的 `portage` 保管利器检查看看
```bash
sudo emerge --ask sbcl
```

但是笨人之前遇到类似 SBCL 不能找到我们的 core file。查了查网上，这时只要… touch 一个文件 /etc/env.d/50sbcl 然后打下：

```bash
SBCL_HOME=/usr/lib/sbcl
SBCL_SOURCE_ROOT=/usr/lib/sbcl/src
```

然后 update 下环境，清下我们的缓存。

```bash
sudo env-update
```

安装完后，如果没出现问题就行了。

```bash
$ sbcl
```

安装完后，如果没出现问题就行了。

```bash
sbcl
```

## 问世间 Quicklisp 为何物？
Quicklisp 是最流行的 Common-Lisp 函数库管理器，可以 *比较* 方便地安装所要的函数库。

安装也是非常的傻瓜化：

```bash
cd ~ && mkdir quicklisp && cd quicklisp
curl -O http://beta.quicklisp.org/quicklisp.lisp
```

下载完后我们 load 它进我们的 SBCL
```bash
sbcl --load ~/quicklisp/quicklisp.lisp
```

下载完后我们 load 它进我们的 SBCL 里面。

依次将以下的命令打下
```lisp
(quicklisp-quickstart:install)
(ql:add-to-init-file)
```

第一行是要求 SBCL 安装我们的 quicklisp。而第二行是把 quicklisp 写入我们的 SBCL config 文件里，每次启动 SBCL 时自动启动 quicklisp。

这样就安装完了啦～我们可以随时在终端上阅读 manual ：

```bash
man sbcl
```

> :-) happy hacking !
