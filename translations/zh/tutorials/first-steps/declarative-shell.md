---
myst:
  html_meta:
    "keywords": "tutorial, declarative, shell, environment, developer, nix, nixpkgs"
---

(declarative-reproducible-envs)=
# 用 `shell.nix` 声明可复现的 shell 环境

```{contributors}
:authors: domenkozar, zmitchell
:editors: fricklerhandwerk
```
## 概览

声明式 shell 环境可以让你：

- 在环境激活时自动运行 bash 命令
- 自动设置环境变量
- 把环境定义纳入版本控制，并在其他机器上复现

### 你将学到什么？

在 {ref}`ad-hoc-envs` 教程中，你学会了用 `nix-shell -p` 命令式地创建 shell 环境。
这很适合快速使用工具而不永久安装。
你也学会了用某个 Git 提交作为参数，按特定 Nixpkgs 修订执行该命令，以复现之前用过的同一环境。

本教程将介绍如何用 {term}`Nix file` 中的声明式配置创建可复现的 shell 环境。
这个文件可以分享给任何人，在不同机器上复现相同环境。

### 需要多久？

30 分钟

### 你需要什么？

- 熟悉 Unix shell
- 对 [Nix 语言](reading-nix-language) 有初步了解

## 进入临时 shell

假设我们想要一个可以使用 `cowsay` 和 `lolcat` 的环境。
最简单的方式是用 `nix-shell -p` 命令：

```
$ nix-shell -p cowsay lolcat
```

这个命令能用，但有不少缺点：
- 每次进入 shell 都要输入 `-p cowsay lolcat`。
- （从易用性上）很难进一步定制 shell 环境。

更好的做法是用 `shell.nix` 文件来创建 shell 环境。

## 一个基础的 `shell.nix` 文件

创建名为 `shell.nix` 的文件，内容如下：

{lineno-start=1}
```nix
let
  nixpkgs = fetchTarball "https://github.com/NixOS/nixpkgs/tarball/nixos-24.05";
  pkgs = import nixpkgs { config = {}; overlays = []; };
in

pkgs.mkShellNoCC {
  packages = with pkgs; [
    cowsay
    lolcat
  ];
}
```

::::{dropdown} 详细说明
我们使用[固定到某个发行分支的 Nixpkgs 版本](<ref-pinning-nixpkgs>)。
如果你学过 [](ad-hoc-envs) 教程，又不想重新下载全部依赖，请指定与 [](towards-reproducibility) 一节中完全相同的修订：

{lineno-start=1 emphasize-lines="2"}
```nix
let
  nixpkgs = fetchTarball "https://github.com/NixOS/nixpkgs/tarball/2a601aafdc5605a5133a2ca506a34a3a73377247";
  pkgs = import nixpkgs { config = {}; overlays = []; };
in
```

我们显式设置 `config` 和 `overlays`，以免被[全局配置](https://nixos.org/manual/nixpkgs/stable/#chap-packageconfig)意外覆盖。

`mkShellNoCC` 是一个接受属性集作为参数的函数。
这里我们给它一个 `packages` 属性，其值为来自 `pkgs` 属性集的两项组成的列表。

:::{Dropdown} 关于 `mkShell` 的旁注

`nix-shell` 和 `mkShell` 最初是为了构造包含[调试包构建所需工具](https://nixos.org/manual/nixpkgs/stable/#sec-tools-of-stdenv)（如 Make 或 GCC）的 shell 环境。
后来它才被广泛用于为其他目的创建临时环境。
`mkShellNoCC` 会生成这样的环境，但不包含编译器工具链。

你可能会看到一些 `mkShell` 或 `mkShellNoCC` 的例子，把包加到 `buildInputs` 或 `nativeBuildInputs` 属性中。
`mkShellNoCC` 是 [`mkDerivation` 的包装](https://nixos.org/manual/nixpkgs/stable/#sec-pkgs-mkShell)，因此它接受与 `mkDerivation` 相同的参数，例如 `buildInputs` 或 `nativeBuildInputs`。
传给 `mkShellNoCC` 的 `packages` 属性参数只是 `nativeBuildInputs` 的别名。
:::
::::

在与 `shell.nix` 相同的目录中运行 `nix-shell` 进入环境：

:::{note}
第一次对该文件调用 `nix-shell` 时，下载全部依赖可能需要一段时间。
:::

```console
$ nix-shell
[nix-shell]$ cowsay hello | lolcat
```

`nix-shell` 默认会在当前目录查找名为 `shell.nix` 的文件，并根据其中的 Nix 表达式构建 shell 环境。
`packages` 属性中定义的包会出现在 `$PATH` 中。

## 环境变量

你可能希望在进入 shell 环境时自动导出某些环境变量。

设置 `GREETING`，使其可在 shell 环境中使用：

```diff
 let
   nixpkgs = fetchTarball "https://github.com/NixOS/nixpkgs/tarball/nixos-24.05";
   pkgs = import nixpkgs { config = {}; overlays = []; };
 in

 pkgs.mkShellNoCC {
   packages = with pkgs; [
     cowsay
     lolcat
   ];

+  GREETING = "Hello, Nix!";
 }
```

传给 `mkShellNoCC` 的任意属性名，只要不是保留名，且其值可被强制转换为字符串，最终都会成为环境变量。

试一试。输入 `exit` 或按 `Ctrl`+`D` 退出 shell，再用 `nix-shell` 重新进入。

```console
[nix-shell]$ echo $GREETING
```

:::{warning}
有些变量无法按上述方式设置。

例如，大多数 shell 的提示符格式由 `PS1` 环境变量控制，但 `nix-shell` 默认已经设置了它，并会忽略参数中的 `PS1` 属性。

如果需要覆盖这些受保护的环境变量，请使用下一节描述的 `shellHook` 属性。
:::

## 启动命令

你可能希望在进入交互式 shell 环境之前先运行一些 shell 命令。
这些命令可以写在传给 `mkShellNoCC` 的 `shellHook` 属性中。

设置 `shellHook`，输出彩色问候：

```diff
 let
   nixpkgs = fetchTarball "https://github.com/NixOS/nixpkgs/tarball/nixos-24.05";
   pkgs = import nixpkgs { config = {}; overlays = []; };
 in

 pkgs.mkShellNoCC {
   packages = with pkgs; [
     cowsay
     lolcat
   ];

   GREETING = "Hello, Nix!";
+
+  shellHook = ''
+    echo $GREETING | cowsay | lolcat
+  '';
 }
```

再试一次。输入 `exit` 或按 `Ctrl`+`D` 退出，再用 `nix-shell` 重新进入，观察效果。

## 参考

- [`mkShell` 文档](https://nixos.org/manual/nixpkgs/stable/#sec-pkgs-mkShell)
- Nixpkgs [shell 函数与工具](https://nixos.org/manual/nixpkgs/stable/#ssec-stdenv-functions) 文档
- [`nix-shell` 文档](https://nix.dev/manual/nix/stable/command-ref/nix-shell)

## 下一步

- [](reading-nix-language)
- [](automatic-direnv)
- [](../../guides/recipes/sharing-dependencies.md)
- [](../../guides/recipes/dependency-management.md)
