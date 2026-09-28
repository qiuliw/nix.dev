(install-nix)=

# 安装 Nix

要求：
 - 安装前，你可能需要先安装 `xz-utils`（或同类工具），以便解压下方脚本将下载的 Nix 二进制包（`.tar.xz`）。

:::::{tab-set}

::::{tab-item} Linux

通过推荐的 [多用户安装] 安装 Nix：

```shell-session
$ curl -L https://nixos.org/nix/install | sh -s -- --daemon
```

在 Arch Linux 上，也可以 [通过 `pacman` 安装 Nix](https://wiki.archlinux.org/title/Nix#Installation)。

在 Fedora 上，可以 [通过 `dnf` 安装 Nix](https://src.fedoraproject.org/rpms/nix)。

::::

::::{tab-item} macOS

通过推荐的 [多用户安装] 安装 Nix：

```shell-session
$ curl -L https://nixos.org/nix/install | sh
```

:::{important}
**升级到 macOS 15 Sequoia**

如果你最近升级到 macOS 15 Sequoia，并在运行 Nix 命令时遇到如下错误：
```console
error: the user '_nixbld1' in the group 'nixbld' does not exist
```
请参考 GitHub 议题 [NixOS/nix#10892](https://github.com/NixOS/nix/issues/10892)，按说明修复安装，无需重装。
:::

::::

::::{tab-item} Windows (WSL2)

通过推荐的 [单用户安装] 安装 Nix：

```shell-session
$ curl -L https://nixos.org/nix/install | sh -s -- --no-daemon
```

如果你已启用 [systemd 支持]，则通过推荐的 [多用户安装] 安装 Nix：

```shell-session
$ curl -L https://nixos.org/nix/install | sh -s -- --daemon
```

[systemd 支持]: https://learn.microsoft.com/en-us/windows/wsl/wsl-config#systemd-support

::::

::::{tab-item} Docker

启动带有 Nix 的 Docker shell：

```shell-session
$ docker run -it nixos/nix
```

或者启动 Docker shell，并挂载一个 `workdir` 目录：

```shell-session
$ mkdir workdir
$ docker run -it -v $(pwd)/workdir:/workdir nixos/nix
```

上面的 `workdir` 示例也可以用来开始折腾 Nixpkgs：

```shell-session
$ git clone git@github.com:NixOS/nixpkgs
$ docker run -it -v $(pwd)/nixpkgs:/nixpkgs nixos/nix
bash-5.1# nix-build -I nixpkgs=/nixpkgs -A hello
bash-5.1# find ./result # 该符号链接指向构建出的包
```

::::

:::::

## 验证安装

打开**一个新的终端**，输入：

```shell-session
$ nix --version
nix (Nix) 2.11.0
```

[多用户安装]: https://nix.dev/manual/nix/stable/installation/multi-user.html
[单用户安装]: https://nix.dev/manual/nix/stable/installation/single-user.html
