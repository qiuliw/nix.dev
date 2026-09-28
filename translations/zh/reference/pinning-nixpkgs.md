(ref-pinning-nixpkgs)=

# 固定 Nixpkgs

指定远程 Nix 表达式（例如 Nixpkgs 提供的表达式）有多种方式：

- [`$NIX_PATH` 环境变量](https://nix.dev/manual/nix/stable/command-ref/env-common.html#env-NIX_PATH)
- 大多数命令（如 `nix-build`、`nix-shell` 等）的 [`-I` 选项](https://nix.dev/manual/nix/stable/command-ref/opt-common.html#opt-I)
- Nix 表达式中的 [`fetchurl`](https://nix.dev/manual/nix/stable/language/builtins.html#builtins-fetchurl)、[`fetchTarball`](https://nix.dev/manual/nix/stable/language/builtins.html#builtins-fetchTarball)、[`fetchGit`](https://nix.dev/manual/nix/stable/language/builtins.html#builtins-fetchGit) 或 [Nixpkgs fetchers](https://nixos.org/manual/nixpkgs/stable/#chap-pkgs-fetchers)

## 可能的 URL 值

- 本地文件路径：

  ```
  ./path/to/expression.nix
  ```

  使用 `./.` 表示表达式位于当前目录下的 `default.nix` 文件中。

- 固定到特定提交：

  ```
  https://github.com/NixOS/nixpkgs/archive/eabc38219184cc3e04a974fe31857d8e0eac098d.tar.gz
  ```

- 使用最新的 channel 版本（表示所有测试均已通过）：

  ```
  http://nixos.org/channels/nixos-22.11/nixexprs.tar.xz
  ```

- channel 的简写语法：

  ```
  channel:nixos-22.11
  ```

- 使用 GitHub 托管的最新 channel 版本：

  ```
  https://github.com/NixOS/nixpkgs/archive/nixos-22.11.tar.gz
  ```

- 使用 release 分支上的最新提交，但尚未经过测试：

  ```
  https://github.com/NixOS/nixpkgs/archive/release-21.11.tar.gz
  ```

## 示例

- ```shell-session
  $ nix-build -I ~/dev
  ```

- ```shell-session
  $ nix-build -I nixpkgs=http://nixos.org/channels/nixos-22.11/nixexprs.tar.xz
  ```

- ```shell-session
  $ nix-build -I nixpkgs=channel:nixos-22.11
  ```

- ```shell-session
  $ NIX_PATH=nixpkgs=http://nixos.org/channels/nixos-22.11/nixexprs.tar.xz nix-build
  ```

- ```shell-session
  $ NIX_PATH=nixpkgs=channel:nixos-22.11 nix-build
  ```

- 在 Nix 语言中：

  ```nix
  let
    pkgs = import (fetchTarball "https://github.com/NixOS/nixpkgs/archive/nixos-22.11.tar.gz") {};
  in pkgs.stdenv.mkDerivation { ... }
  ```

## 查找特定提交与发行版

[status.nixos.org](https://status.nixos.org/) 提供：

- 每个发行版已通过测试的最新提交——在固定到特定提交时使用
- 活跃的 release channel 列表——在跟踪最新 channel 版本时使用

完整的 channel 列表见 [nixos.org/channels](https://nixos.org/channels)。

:::{tip}
关于 Nixpkgs 与 NixOS 发行版的更多信息：[](channel-branches)
:::
