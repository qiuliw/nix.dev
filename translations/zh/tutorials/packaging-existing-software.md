---
myst:
  html_meta:
    "description lang=zh": "用 Nix 打包现有软件"
    "keywords": "Nix, packaging"
---

(packaging-tutorial)=
# 用 Nix 打包现有软件

```{contributors}
:authors: proofconstruction
:editors: fricklerhandwerk
```

Nix 的主要用途之一，是解决打包软件时常见的困难，例如如何声明并获取依赖。

从长远看，Nix 能缓解这类问题。
但在*初次*用 Nix 打包现有软件时，经常会遇到看似难以理解的错误。

## 简介

在本教程中，你将创建第一个 [Nix 推导式（derivation）](https://nix.dev/manual/nix/stable/language/derivations) 来打包 C/C++ 软件。
你会利用 [Nixpkgs 标准环境](https://nixos.org/manual/nixpkgs/stable/#part-stdenv)（`stdenv`），它能自动完成大量工作。

### 你会学到什么？

教程从 `hello` 开始——一个只需 `stdenv` 已提供依赖的 "hello world" 实现。
接下来，你会构建带有自身依赖的更复杂包，并由此用到更多推导式特性。

你将遇到并处理 Nix 错误信息、构建失败以及其他各种问题，并在此过程中培养迭代式调试技巧。

### 你需要什么？

- 熟悉 Unix shell 与纯文本编辑器
- 你应对[阅读 Nix 语言](reading-nix-language)有一定把握。如有需要，请先回去完成该教程。

### 需要多长时间？

仔细完成所有步骤大约需要 60 分钟。

## 你的第一个包

:::{note}
<!--
TODO: link to the Nix manual glossary entry once it's in a released build:
https://hydra.nixos.org/job/nix/master/build.x86_64-linux/latest/download/manual/glossary.html#package
-->
_包（package）_ 是一个定义较松散的概念，既可以指一组文件及其他数据，也可以指在其生成之前表示该集合的 {term}`Nix expression`。
Nixpkgs 中的包有约定俗成的结构，使其能在搜索中被发现，并能与其他包一起组合进环境。

就本教程而言，"包" 是一个会求值为推导式的 Nix 语言函数。
它能让你或其他人产出可供实际使用的产物——这也正是 "用 Nix 打包现有软件" 的结果。
:::

首先，看这个推导式骨架：

```nix
{ stdenv }:

stdenv.mkDerivation {	}
```

这是一个函数：接收包含 `stdenv` 的属性集，并产出一个推导式（目前什么也不做）。

### 包函数

GNU Hello 是 "hello world" 程序的一个实现，其源代码可[从 GNU Project 的 FTP 服务器](https://ftp.gnu.org/gnu/hello/)获取。

首先，在传给 `mkDerivation` 的集合中添加 `pname` 属性。
每个包都需要名称和版本；没有它们时，Nix 会抛出 `error: derivation name missing`。

```diff

stdenv.mkDerivation {
+ pname = "hello";
+ version = "2.12.1";

```

接下来，声明对最新版 `hello` 的依赖，并指示 Nix 使用 `fetchzip` 下载[源代码归档](https://ftp.gnu.org/gnu/hello/hello-2.12.1.tar.gz)。

:::{note}
`fetchzip` 能获取的[归档类型](https://nixos.org/manual/nixpkgs/stable/#fetchurl)不止 zip！
:::

哈希只有在归档下载并解压之后才能知道。
若提供给 `fetchzip` 的哈希不正确，Nix 会报错。
先把 `hash` 属性设为空字符串，再用报错信息确定正确哈希：

```nix
# hello.nix
{
  stdenv,
  fetchzip,
}:

stdenv.mkDerivation {
  pname = "hello";
  version = "2.12.1";

  src = fetchzip {
    url = "https://ftp.gnu.org/gnu/hello/hello-2.12.1.tar.gz";
    sha256 = "";
  };
}
```

将此文件保存为 `hello.nix`，运行 `nix-build`，观察你的第一次构建失败：

```console
$ nix-build hello.nix
error: cannot evaluate a function that has an argument without a value ('stdenv')
       Nix attempted to evaluate a function as a top level expression; in
       this case it must have its arguments supplied either by default
       values, or passed explicitly with '--arg' or '--argstr'. See
       https://nix.dev/manual/nix/stable/language/constructs.html#functions.

       at /home/nix-user/hello.nix:3:3:

            2| {
            3|   stdenv,
             |   ^
            4|   fetchzip,
```

问题：`hello.nix` 中的表达式是一个*函数*，只有传入正确的*参数*时才会产生预期输出。

### 用 `nix-build` 构建

`stdenv` 来自 [`nixpkgs`](https://github.com/NixOS/nixpkgs/)，必须用另一段 Nix 表达式导入后，再作为参数传给该推导式。

推荐做法是在与 `hello.nix` 同一目录下创建 `default.nix`，内容如下：

```nix
# default.nix
let
  nixpkgs = fetchTarball "https://github.com/NixOS/nixpkgs/tarball/nixos-24.05";
  pkgs = import nixpkgs { config = {}; overlays = []; };
in
{
  hello = pkgs.callPackage ./hello.nix { };
}
```

这样你就可以运行 `nix-build -A hello` 来实现 `hello.nix` 中的推导式，用法与 Nixpkgs 中当前的惯例类似。

:::{note}
若函数参数属性集中要求的属性与 `pkgs` 中的属性匹配，`callPackage` 会自动把这些属性传给该函数。
在本例中，`callPackage` 会向 `hello.nix` 中定义的函数提供 `stdenv` 和 `fetchzip`。

教程 [](./callpackage.md) 详细介绍了其工作原理。
:::

现在用新参数运行 `nix-build` 命令：

```console
$ nix-build -A hello
error: hash mismatch in fixed-output derivation '/nix/store/pd2kiyfa0c06giparlhd1k31bvllypbb-source.drv':
         specified: sha256-AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=
            got:    sha256-1kJjhtlsAkpNB7f6tZEs+dbKd8z7KoNHyDHEJ0tmhnc=
error: 1 dependencies of derivation '/nix/store/b4mjwlv73nmiqgkdabsdjc4zq9gnma1l-hello-2.12.1.drv' failed to build
```

### 查找文件哈希
如预期，错误的文件哈希导致了错误，而 Nix 友好地提供了正确哈希。
在 `hello.nix` 中，用正确哈希替换空字符串：

```nix
# hello.nix
{
  stdenv,
  fetchzip,
}:

stdenv.mkDerivation {
  pname = "hello";
  version = "2.12.1";

  src = fetchzip {
    url = "https://ftp.gnu.org/gnu/hello/hello-2.12.1.tar.gz";
    sha256 = "sha256-1kJjhtlsAkpNB7f6tZEs+dbKd8z7KoNHyDHEJ0tmhnc=";
  };
}
```

再次运行上一条命令：

```console
$ nix-build -A hello
this derivation will be built:
  /nix/store/rbq37s3r76rr77c7d8x8px7z04kw2mk7-hello.drv
building '/nix/store/rbq37s3r76rr77c7d8x8px7z04kw2mk7-hello.drv'...
...
configuring
...
configure: creating ./config.status
config.status: creating Makefile
...
building
... <many more lines omitted>
```
推导式构建成功。

控制台输出显示调用了 `configure`，它生成了随后用于构建项目的 `Makefile`。
此例中无需编写任何构建指令，因为 `stdenv` 构建系统基于 [GNU Autoconf](https://www.gnu.org/software/autoconf/)，能自动检测项目目录结构。

### 构建结果
检查工作目录中的结果：

```console
$ ls
default.nix hello.nix  result
```

这个 `result` 是指向 Nix store 中已构建二进制文件位置的[符号链接](https://en.wikipedia.org/wiki/Symbolic_link)；你可以调用 `./result/bin/hello` 来执行该程序：

```console
$ ./result/bin/hello
Hello, world!
```

恭喜，你已成功用 Nix 打包了第一个程序！

接下来，你将打包另一款软件，其依赖超出 `stdenv`，会带来新挑战，并要求你使用更多 `mkDerivation` 特性。

## 带依赖的包

现在添加第二个稍复杂的程序：[`icat`](https://github.com/atextor/icat)（在终端中渲染图像）。

修改上一节的 `default.nix`，为 `icat` 添加新属性：

```nix
# default.nix
let
  nixpkgs = fetchTarball "https://github.com/NixOS/nixpkgs/tarball/nixos-24.05";
  pkgs = import nixpkgs { config = {}; overlays = []; };
in
{
  hello = pkgs.callPackage ./hello.nix { };
  icat = pkgs.callPackage ./icat.nix { };
}
```

将 `hello.nix` 复制为新文件 `icat.nix`，并更新其中的 `pname` 和 `version` 属性：

```nix
# icat.nix
{
  stdenv,
  fetchzip,
}:

stdenv.mkDerivation {
  pname = "icat";
  version = "v0.5";

  src = fetchzip {
    # ...
  };
}
```

现在下载源代码。
`icat` 的上游仓库托管在 [GitHub](https://github.com/atextor/icat) 上，因此应替换之前的[源码获取器](https://nixos.org/manual/nixpkgs/stable/#chap-pkgs-fetchers)。
这次改用 [`fetchFromGitHub`](https://nixos.org/manual/nixpkgs/stable/#fetchfromgithub) 而非 `fetchzip`，相应地更新函数的参数属性集：

```nix
# icat.nix
{
  stdenv,
  fetchFromGitHub,
}:

stdenv.mkDerivation {
  pname = "icat";
  version = "v0.5";

  src = fetchFromGitHub {
    # ...
  };
}
```

### 从 GitHub 获取源码
`fetchzip` 需要 `url` 和 `sha256` 参数，而 [`fetchFromGitHub`](https://nixos.org/manual/nixpkgs/stable/#fetchfromgithub) 需要更多参数。

源码 URL 是 `https://github.com/atextor/icat`，由此已能得到前两个参数：
- `owner`：控制该仓库的账户名

  ```
  owner = "atextor";
  ```
- `repo`：要获取的仓库名

  ```
  repo = "icat";
  ``````

前往项目的 [Tags 页面](https://github.com/atextor/icat/tags)，找到合适的 [Git 修订（revision）](https://git-scm.com/docs/revisions)（`rev`），例如与你要获取的发行版对应的 Git 提交哈希或标签（如 `v1.0`）。

本例中，最新发行标签是 `v0.5`。

与 `hello` 示例一样，也必须提供哈希。
这次不用空字符串再让 `nix-build` 在错误中报告正确哈希，你可以先用 `nix-prefetch-url` 命令直接获取正确哈希。

你需要的是 tarball *内容* 的 SHA256 哈希（而非 tarball 文件本身的哈希）。
因此传入 `--unpack` 和 `--type sha256` 参数：

```console
$ nix-prefetch-url --unpack https://github.com/atextor/icat/archive/refs/tags/v0.5.tar.gz --type sha256
path is '/nix/store/p8jl1jlqxcsc7ryiazbpm7c1mqb6848b-v0.5.tar.gz'
0wyy2ksxp95vnh71ybj1bbmqd5ggp13x3mk37pzr99ljs9awy8ka
```

为 `fetchFromGitHub` 设置正确哈希：

```nix
# icat.nix
{
  stdenv,
  fetchFromGitHub,
}:

stdenv.mkDerivation {
  pname = "icat";
  version = "v0.5";

  src = fetchFromGitHub {
    owner = "atextor";
    repo = "icat";
    rev = "v0.5";
    sha256 = "0wyy2ksxp95vnh71ybj1bbmqd5ggp13x3mk37pzr99ljs9awy8ka";
  };
}
```

### 缺失的依赖

仅对新建的 `icat` 属性运行 `nix-build`，会报告一个全新问题：

```console
$ nix-build -A icat
these 2 derivations will be built:
  /nix/store/86q9x927hsyyzfr4lcqirmsbimysi6mb-source.drv
  /nix/store/l5wz9inkvkf0qhl8kpl39vpg2xfm2qpy-icat.drv
...
error: builder for '/nix/store/l5wz9inkvkf0qhl8kpl39vpg2xfm2qpy-icat.drv' failed with exit code 2;
       last 10 log lines:
       >                  from /nix/store/hkj250rjsvxcbr31fr1v81cv88cdfp4l-glibc-2.37-8-dev/include/stdio.h:27,
       >                  from icat.c:31:
       > /nix/store/hkj250rjsvxcbr31fr1v81cv88cdfp4l-glibc-2.37-8-dev/include/features.h:195:3: warning: #warning "_BSD_SOURCE and _SVID_SOURCE are deprecated, use _DEFAULT_SOURCE" [8;;https://gcc.gnu.org/onlinedocs/gcc/Warning-Options.html#index-Wcpp-Wcpp8;;]
       >   195 | # warning "_BSD_SOURCE and _SVID_SOURCE are deprecated, use _DEFAULT_SOURCE"
       >       |   ^~~~~~~
       > icat.c:39:10: fatal error: Imlib2.h: No such file or directory
       >    39 | #include <Imlib2.h>
       >       |          ^~~~~~~~~~
       > compilation terminated.
       > make: *** [Makefile:16: icat.o] Error 1
       For full logs, run 'nix log /nix/store/l5wz9inkvkf0qhl8kpl39vpg2xfm2qpy-icat.drv'.
```

编译器错误。
`icat` 源码已从 GitHub 拉取，Nix 试图构建找到的内容，但因缺少依赖——`imlib2` 头文件——而编译失败。

若你在 [search.nixos.org 上搜索 `imlib2`](https://search.nixos.org/packages?query=imlib2)，会发现 `imlib2` 已在 Nixpkgs 中。

通过在 `icat.nix` 函数参数中加入 `imlib2`，将该包加入构建环境。
再把参数值 `imlib2` 加入 `stdenv.mkDerivation` 的 `buildInputs` 列表：

```nix
# icat.nix
{
  stdenv,
  fetchFromGitHub,
  imlib2,
}:

stdenv.mkDerivation {
  pname = "icat";
  version = "v0.5";

  src = fetchFromGitHub {
    owner = "atextor";
    repo = "icat";
    rev = "v0.5";
    sha256 = "0wyy2ksxp95vnh71ybj1bbmqd5ggp13x3mk37pzr99ljs9awy8ka";
  };

  buildInputs = [ imlib2 ];
}
```

再次运行 `nix-build -A icat`，你会遇到另一个错误，但这次编译推进得更远：

```console
$ nix-build -A icat
this derivation will be built:
  /nix/store/bw2d4rp2k1l5rg49hds199ma2mz36x47-icat.drv
...
error: builder for '/nix/store/bw2d4rp2k1l5rg49hds199ma2mz36x47-icat.drv' failed with exit code 2;
       last 10 log lines:
       >                  from icat.c:31:
       > /nix/store/hkj250rjsvxcbr31fr1v81cv88cdfp4l-glibc-2.37-8-dev/include/features.h:195:3: warning: #warning "_BSD_SOURCE and _SVID_SOURCE are deprecated, use _DEFAULT_SOURCE" [8;;https://gcc.gnu.org/onlinedocs/gcc/Warning-Options.html#index-Wcpp-Wcpp8;;]
       >   195 | # warning "_BSD_SOURCE and _SVID_SOURCE are deprecated, use _DEFAULT_SOURCE"
       >       |   ^~~~~~~
       > In file included from icat.c:39:
       > /nix/store/4fvrh0sjc8sbkbqda7dfsh7q0gxmnh9p-imlib2-1.11.1-dev/include/Imlib2.h:45:10: fatal error: X11/Xlib.h: No such file or directory
       >    45 | #include <X11/Xlib.h>
       >       |          ^~~~~~~~~~~~
       > compilation terminated.
       > make: *** [Makefile:16: icat.o] Error 1
       For full logs, run 'nix log /nix/store/bw2d4rp2k1l5rg49hds199ma2mz36x47-icat.drv'.
```

你可以看到一些应由上游代码修正的警告。
但对本教程而言，重要的是 `fatal error: X11/Xlib.h: No such file or directory`：又缺少一个依赖。

## 查找包

确定依赖应从何处获取目前仍有些麻烦，因为包名并不总是与库名或程序名对应。

你需要来自 `X11` C 包的 `Xlib.h` 头文件。
对应的 Nixpkgs 推导式是 `libX11`，位于 `xorg` 包集中。
有多种方式可以弄清这一点：

### `search.nixos.org`

:::{tip}
最简单的查找方式是在 search.nixos.org/packages。
:::

可惜在本例中，[搜索 `x11`](https://search.nixos.org/packages?query=x11) 会产生太多无关结果，因为 X11 无处不在。
左侧栏有包集列表，[选择 `xorg`](https://search.nixos.org/packages?buckets={%22package_attr_set%22%3A[%22xorg%22]%2C%22package_license_set%22%3A[]%2C%22package_maintainers_set%22%3A[]%2C%22package_platforms%22%3A[]}&query=x11) 会看到有希望的结果。

若其他方法都失败，熟悉在 [Nixpkgs 源代码](https://github.com/nixos/nixpkgs) 中按关键字搜索会很有帮助。

### 本地代码搜索

要在源码中查找名称赋值，搜索 `"<keyword> ="`。
例如，这些是 GitHub 上 [`"x11 = "`](https://github.com/search?q=repo%3ANixOS%2Fnixpkgs+%22x11+%3D%22&type=code) 或 [`"libx11 ="`](https://github.com/search?q=repo%3ANixOS%2Fnixpkgs+%22libx11+%3D%22&type=code) 的搜索结果。

或者克隆 [Nixpkgs 仓库](https://github.com/nixos/nixpkgs) 并在本地搜索代码。

启动一个提供所需工具的 shell——版本控制用 `git`，代码搜索用 `rg`（由 [`ripgrep` 包](https://search.nixos.org/packages?show=ripgrep) 提供）：
```console
$ nix-shell -p git ripgrep
[nix-shell:~]$
```

Nixpkgs 仓库非常大。
只克隆最新修订，以免等待完整克隆太久：

```console
[nix-shell:~]$ git clone https://github.com/NixOS/nixpkgs --depth 1
...
[nix-shell:~]$ cd nixpkgs/
```

为缩小结果范围，只搜索存放所有包配方的 `pkgs` 子目录：

```console
[nix-shell:~]$ rg "x11 =" pkgs
pkgs/tools/X11/primus/default.nix
21:  primus = if useNvidia then primusLib_ else primusLib_.override { nvidia_x11 = null; };
22:  primus_i686 = if useNvidia then primusLib_i686_ else primusLib_i686_.override { nvidia_x11 = null; };

pkgs/applications/graphics/imv/default.nix
38:    x11 = [ libGLU xorg.libxcb xorg.libX11 ];

pkgs/tools/X11/primus/lib.nix
14:    if nvidia_x11 == null then libGL

pkgs/top-level/linux-kernels.nix
573:    ati_drivers_x11 = throw "ati drivers are no longer supported by any kernel >=4.1"; # added 2021-05-18;
... <a lot more results>
```

由于 `rg` 默认区分大小写，
加上 `-i` 以确保不会遗漏：

```
[nix-shell:~]$ rg -i "libx11 =" pkgs
pkgs/applications/version-management/monotone-viz/graphviz-2.0.nix
55:    ++ lib.optional (libX11 == null) "--without-x";

pkgs/top-level/all-packages.nix
14191:    libX11 = xorg.libX11;

pkgs/servers/x11/xorg/default.nix
1119:  libX11 = callPackage ({ stdenv, pkg-config, fetchurl, xorgproto, libpthreadstubs, libxcb, xtrans, testers }: stdenv.mkDerivation (finalAttrs: {

pkgs/servers/x11/xorg/overrides.nix
147:  libX11 = super.libX11.overrideAttrs (attrs: {
```

### 本地推导式搜索

要在命令行搜索推导式，使用 [`nix-index`](https://github.com/nix-community/nix-index) 提供的 `nix-locate`。

### 将包集添加为依赖

把 `xorg` 加入推导式的输入属性集，并在 `buildInputs` 中使用 `xorg.libX11`：

```nix
# icat.nix
{
  stdenv,
  fetchFromGitHub,
  imlib2,
  xorg,
}:

stdenv.mkDerivation {
  pname = "icat";
  version = "v0.5";

  src = fetchFromGitHub {
    owner = "atextor";
    repo = "icat";
    rev = "v0.5";
    sha256 = "0wyy2ksxp95vnh71ybj1bbmqd5ggp13x3mk37pzr99ljs9awy8ka";
  };

  buildInputs = [ imlib2 xorg.libX11 ];
}
```

:::{note}
因为 Nix 语言是惰性求值的，只访问 `xorg.libX11` 意味着 `xorg` 属性集的其余内容永远不会被处理。
:::

## 修复构建失败

再次运行上一条命令：

```console
$ nix-build -A icat
this derivation will be built:
  /nix/store/x1d79ld8jxqdla5zw2b47d2sl87mf56k-icat.drv
...
error: builder for '/nix/store/x1d79ld8jxqdla5zw2b47d2sl87mf56k-icat.drv' failed with exit code 2;
       last 10 log lines:
       >   195 | # warning "_BSD_SOURCE and _SVID_SOURCE are deprecated, use _DEFAULT_SOURCE"
       >       |   ^~~~~~~
       > icat.c: In function 'main':
       > icat.c:319:33: warning: ignoring return value of 'write' declared with attribute 'warn_unused_result' [8;;https://gcc.gnu.org/onlinedocs/gcc/Warning-Options.html#index-Wunused-result-Wunused-result8;;]
       >   319 |                                 write(tempfile, &buf, 1);
       >       |                                 ^~~~~~~~~~~~~~~~~~~~~~~~
       > gcc -o icat icat.o -lImlib2
       > installing
       > install flags: SHELL=/nix/store/8fv91097mbh5049i9rglc73dx6kjg3qk-bash-5.2-p15/bin/bash install
       > make: *** No rule to make target 'install'.  Stop.
       For full logs, run 'nix log /nix/store/x1d79ld8jxqdla5zw2b47d2sl87mf56k-icat.drv'.
```

缺失依赖的错误已解决，但现在有另一个问题：`make: *** No rule to make target 'install'.  Stop.`

### `installPhase`
`stdenv` 正在自动处理 `icat` 附带的 `Makefile`。
控制台输出显示 `configure` 和 `make` 执行无误，因此 `icat` 二进制已成功编译。

失败发生在 `stdenv` 尝试运行 `make install` 时。
项目附带的 `Makefile` 恰好缺少 `install` 目标。
`icat` 仓库中的 `README` 只提到用 `make` 构建该工具，安装步骤留给用户自行处理。

要在推导式中加入此步骤，使用 [`installPhase` 属性](https://nixos.org/manual/nixpkgs/stable/#ssec-install-phase)。
它包含一组命令字符串，执行这些命令以完成安装。

因为 `make` 已成功完成，`icat` 可执行文件位于构建目录中。
你只需把它从那里复制到输出目录。

在 Nix 中，输出目录存储在 `$out` 变量里。
该变量可在推导式的 [`builder` 执行环境](https://nix.dev/manual/nix/2.19/language/derivations#builder-execution)中访问。
在 `$out` 目录下创建 `bin` 目录，并将 `icat` 二进制复制到那里：

```nix
# icat.nix
{
  stdenv,
  fetchFromGitHub,
  imlib2,
  xorg,
}:

stdenv.mkDerivation {
  pname = "icat";
  version = "v0.5";

  src = fetchFromGitHub {
    owner = "atextor";
    repo = "icat";
    rev = "v0.5";
    sha256 = "0wyy2ksxp95vnh71ybj1bbmqd5ggp13x3mk37pzr99ljs9awy8ka";
  };

  buildInputs = [ imlib2 xorg.libX11 ];

  installPhase = ''
    mkdir -p $out/bin
    cp icat $out/bin
  '';
}
```

### 阶段与钩子

Nixpkgs 的 `stdenv.mkDerivation` 推导式划分为多个[阶段（phases）](https://nixos.org/manual/nixpkgs/stable/#sec-stdenv-phases)。
每个阶段旨在控制构建过程的某个方面。

此前你看到 `stdenv.mkDerivation` 期望项目的 `Makefile` 有 `install` 目标，没有时就会失败。
为修复这一点，你定义了自定义的 `installPhase`，其中包含将 `icat` 二进制复制到正确输出位置（即安装它）的指令。
在那之前，`stdenv.mkDerivation` 自动为 `icat` 包确定了 `buildPhase` 信息。

在推导式实现（realisation）期间，每个推导式阶段中都可能执行若干 shell 函数（在 Nixpkgs 中称为 "钩子/hooks"）。
钩子会设置变量、source 文件、创建目录等。

它们与各阶段对应，并在该阶段执行前后运行。
它们为构建期间的常见操作修改构建环境。

即使你没有直接使用这些钩子，也应在你定义的推导式阶段中调用它们。
这便于日后轻松[覆盖](https://nixos.org/manual/nixpkgs/stable/#chap-overrides)推导式的特定部分。
同时也能保持代码整洁、更易阅读。

调整你的 `installPhase` 以调用相应钩子：

```nix
# icat.nix

# ...

  installPhase = ''
    runHook preInstall
    mkdir -p $out/bin
    cp icat $out/bin
    runHook postInstall
  '';

# ...

```

## 成功的构建

再次运行 `nix-build -A icat` 命令，终于会按你期望的方式、可重复地完成构建。
在本地目录运行 `ls`，会看到指向 Nix store 位置的 `result` 符号链接：

```console
$ ls
default.nix hello.nix icat.nix result
```

`result/bin/icat` 就是之前构建的可执行文件。成功！

运行 `nix-build`（不指定属性）会一次性构建所有属性。
第一个（`hello`）出现在 `result/bin/` 下，第二个（`icat`）出现在 `result-2/bin/` 下。
添加更多属性会产生额外的 `result-n` 符号链接。

## 参考

- [Nixpkgs Manual - Standard Environment](https://nixos.org/manual/nixpkgs/unstable/#part-stdenv)

## 下一步

- [](callpackage-tutorial)
- [](sharing-dependencies)
- [](automatic-direnv)
- [](python-dev-environment)
- [将你自己的新包添加到 Nixpkgs](https://github.com/NixOS/nixpkgs/blob/master/CONTRIBUTING.md)
  - [](../contributing/how-to-contribute.md)
  - [](../contributing/how-to-get-help.md)
