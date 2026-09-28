(ad-hoc-envs)=

# 临时 shell 环境

```{contributors}
:authors: domenkozar
:editors: fricklerhandwerk
```

在 Nix shell 环境中，你可以立刻使用任何用 Nix 打包的程序，而无需永久安装。

你也可以把启动这种 shell 的命令分享给别人，它会在所有 Linux 发行版、WSL 以及 macOS 上工作[^1]。

[^1]: 并非所有包都同时支持 Linux 与 macOS。尤其是图形程序的支持可能有所不同。

## 创建 shell 环境

一旦你 {ref}`安装了 Nix <install-nix>`，就可以用它创建包含你想用程序的新 *shell 环境*。

在本节中，你将运行两个可能尚未安装的冷门程序：`cowsay` 和 `lolcat`：

```shell-session
$ cowsay no can do
The program ‘cowsay’ is currently not installed.

$ echo no chance | lolcat
The program ‘lolcat’ is currently not installed.
```

使用带 `-p`（`--packages`）选项的 `nix-shell`，声明我们需要 `cowsay` 和 `lolcat` 这两个包：

:::{note}
第一次为这些包调用 `nix-shell` 时，下载全部依赖可能需要一段时间。
:::

```shell-session
$ nix-shell -p cowsay lolcat
these 3 derivations will be built:
  /nix/store/zx1j8gchgwzfjn7sr4r8yxb7a0afkjdg-builder.pl.drv
  /nix/store/h9sbaa2k8ivnihw2czhl5b58k0f7fsfh-lolcat-100.0.1.drv
  ...

[nix-shell:~]$
```

在 Nix shell 中，你可以使用这些包提供的程序：

```shell-session
[nix-shell:~]$ cowsay Hello, Nix! | lolcat
```

输入 `exit` 或按 `CTRL-D` 退出 shell 后，这些程序就不再可用。

```shell-session
[nix-shell:~]$ exit
exit

$ cowsay no more
The program ‘cowsay’ is currently not installed.

$ echo all gone | lolcat
The program ‘lolcat’ is currently not installed.
```

## 一次性运行程序

你还可以更快：直接运行任意程序：

```console
$ nix-shell -p cowsay --run "cowsay Nix"
```

如果命令只是程序名本身，则不需要引号：

```console
$ nix-shell -p hello --run hello
```

## 搜索包

shell 环境里可以放什么？
只要你能想到的，通常就有对应的 Nix 包。

:::{tip}
在 [search.nixos.org](https://search.nixos.org/packages) 输入你想运行的程序名，即可找到提供它的包。
:::

在下面的例子中，请找出这些程序对应的包名：

- `git`
- `nvim`
- `npm`

在搜索结果中，每一项会显示包名，详情里会列出可用程序。[^2]

[^2]: 包名不等于程序名。许多包提供多个程序，或者作为库根本不提供程序。即便某个包恰好只提供一个程序，包名与程序名也不一定相同。

(run-any-program)=
## 运行任意程序组合

有了包名之后，就可以启动包含该包的 shell。
`-p`（`--packages`）参数可以接受多个包名。

启动一个提供 `git`、`nvim` 和 `npm` 的 Nix shell。
同样，第一次调用可能需要较长时间下载依赖。

```shell-session
$ nix-shell -p git neovim nodejs
these 9 derivations will be built:
  /nix/store/7gz8jyn99kw4k74bgm4qp6z487l5ap06-packdir-start.drv
  /nix/store/d6fkgxc3b04m85wrhg6j0l5y0ray82l7-packdir-opt.drv
  /nix/store/da6njv7r0zzc2n54n2j54g2a5sbi4a5i-manifest.vim.drv
  /nix/store/zs4jb2ybr4rcyzwq0dagg9rlhlc368h6-builder.pl.drv
  /nix/store/g8sl2xnsshfrz9f39ki94k8p15vp3xd7-vim-pack-dir.drv
  /nix/store/jmxkg8b1psk52awsvfziy9nq6dwmxmjp-luajit-2.1.0-2022-10-04-env.drv
  /nix/store/kn83q8yk6ds74zgyklrjhvv5wkv5wmch-python3-3.10.9-env.drv
  /nix/store/m445wn3vizcgg7syna2cdkkws3kk1gq8-neovim-ruby-env.drv
  /nix/store/r2wa882mw99c311a4my7hcis9lq3kp3v-neovim-0.8.1.drv
these 151 paths will be fetched (186.43 MiB download, 1018.20 MiB unpacked):
  /nix/store/046zxlxhq4srm3ggafkymx794bn1jksc-bzip2-1.0.8
  /nix/store/0p1jxcb7b4p8jhhlf8qnjc4cqwy89460-unibilium-2.1.1
  /nix/store/0q4fpnqmg8liqraj7zidylcyd062f6z0-perl5.36.0-URI-5.05
  ...

[nix-shell:~]$
```

(check-package-version)=
### 检查包版本

检查你使用的是否是 Nix 提供的这些程序的特定版本——即便机器上原本已经安装过它们。

```shell-session
[nix-shell:~]$ which git
/nix/store/3cdi52xh6lk3h1fb51jkxs3p561p37wg-git-2.38.3/bin/git

[nix-shell:~]$ git --version
git version 2.38.3

[nix-shell:~]$ which nvim
/nix/store/ynskzgkf07lmrrs3cl2kzr9ah487lwab-neovim-0.8.1/bin/nvim

[nix-shell:~]$ nvim --version | head -1
NVIM v0.8.1

[nix-shell:~]$ which npm
/nix/store/q12w83z0i5pi1y0m6am7qmw1r73228sh-nodejs-18.12.1/bin/npm

[nix-shell:~]$ npm --version
8.19.2
```

## 嵌套 shell 会话

如果临时还需要另一个程序，可以再开一层嵌套的 Nix shell。
指定包所提供的程序会被加入当前环境。

```shell-session
[nix-shell:~]$ nix-shell -p python3
this path will be fetched (11.42 MiB download, 62.64 MiB unpacked):
  /nix/store/pwy30a7siqrkki9r7xd1lksyv9fg4l1r-python3-3.10.11
copying path '/nix/store/pwy30a7siqrkki9r7xd1lksyv9fg4l1r-python3-3.10.11' from 'https://cache.nixos.org'...

[nix-shell:~]$ python --version
Python 3.10.11
```

像往常一样退出 shell，即可回到上一层环境。

(towards-reproducibility)=
## 迈向可复现

这些 shell 环境非常方便，但目前的例子还不能称为可复现。
在另一台机器上运行同样的命令，可能拉取到不同版本的包，这取决于那边何时安装了 Nix。

可复现是什么意思？
完全可复现的例子，无论何时何地运行，都会得到完全相同的结果。
提供的环境每次都一模一样。

下面的例子创建了一个完全可复现的环境。
你可以在任何地方、任何时间运行它，得到完全相同版本的 `git`。

```shell-session
$ nix-shell -p git --run "git --version" --pure -I nixpkgs=https://github.com/NixOS/nixpkgs/tarball/2a601aafdc5605a5133a2ca506a34a3a73377247
...
git version 2.33.1
```

这里有三件事：

1. `--run` 在 Nix 创建的环境中执行给定的 [Bash 命令](https://www.gnu.org/software/bash/manual/bash.html#Shell-Commands)，完成后退出。

   当你想快速运行机器上未安装的程序时，可以配合 `nix-shell` 使用。

2. `--pure` 在运行 shell 时丢弃系统中已设置的大多数环境变量。

   这意味着该 shell 内只有 Nix 提供的 `git` 可用。
   这对示例中的简单一行命令很有用。
   但在开发时，你通常仍希望编辑器和其他工具可用。
   因此我们建议开发环境省略 `--pure`，只在需要额外隔离时再加。

3. `-I` 决定包声明的来源。

   这里我们提供了 [`nixpkgs` 的一个特定 Git 修订](https://github.com/NixOS/nixpkgs/tree/2a601aafdc5605a5133a2ca506a34a3a73377247)，从而明确该集合中将使用哪些版本的包。


## 参考

- [Nix 手册：`nix-shell`](https://nix.dev/manual/nix/stable/command-ref/nix-shell)（或运行 `man nix-shell`）
- [Nix 手册：`-I` 选项](https://nix.dev/manual/nix/stable/command-ref/opt-common.html#opt-I)

## 下一步

- {ref}`reproducible-scripts` – 用 Nix 编写可复现脚本
- {ref}`reading-nix-language` – 学习阅读用于声明包与配置的 Nix 语言
- {ref}`declarative-reproducible-envs` – 用声明式配置文件创建可复现 shell 环境
- {ref}`pinning-nixpkgs` – 学习指定包源精确版本的不同方式

如果暂时不打算继续试用 Nix，可以运行下面的命令，清理示例下载的不同版本程序占用的磁盘空间：

```shell-session
$ nix-collect-garbage
```
