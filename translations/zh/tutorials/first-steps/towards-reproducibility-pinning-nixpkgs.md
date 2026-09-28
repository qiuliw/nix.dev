(pinning-nixpkgs)=

# 迈向可复现：固定 Nixpkgs

在各种 Nix 示例中，你经常会看到：

```nix
{ pkgs ? import <nixpkgs> {} }:

...
```

:::{note}
`<nixpkgs>` 指向某个 {term}`Nixpkgs` 修订在文件系统中的路径。
关于[查找路径](lookup-path-tutorial)的更多信息，见 [](reading-nix-language)。
:::

这是一种**方便**的做法，可以快速演示 Nix 表达式，并通过导入 Nix 包让它先跑起来。

然而，<ref-search-path>**这样得到的 Nix 表达式并不是完全可复现的**。

## 在 Nix 表达式内用 URL 固定包

要创建**完全可复现**的 Nix 表达式，可以固定 Nixpkgs 的精确版本。

最简单的方式是按相关 Git 提交哈希拉取所需 Nixpkgs 版本的 tarball：

```nix
{ pkgs ? import (fetchTarball "https://github.com/NixOS/nixpkgs/archive/06278c77b5d162e62df170fec307e83f1812d94b.tar.gz") {}
}:

...
```

可以通过 [status.nixos.org](https://status.nixos.org/) 选择提交，
该站点列出了所有发行版以及通过全部测试的最新提交。

选择提交时，建议遵循以下之一：

- 使用特定版本（如 `nixos-21.05`）跟随**最新稳定版 NixOS**，**或者**
- 通过 `nixos-unstable` 跟随最新的**不稳定发行版**。

## 下一步

- 关于固定 `nixpkgs` 的更多示例与细节，见 {ref}`ref-pinning-nixpkgs`。
- [](dependency-management-npins)
