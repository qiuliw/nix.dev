# 常见问题

## Nix 这个名字的由来是什么？

> The name *Nix* is derived from the Dutch word *niks*, meaning *nothing*;
> build actions do not see anything that has not been explicitly declared as an input.
>
> &mdash; <cite>[Nix: A Safe and Policy-Free System for Software Deployment](https://edolstra.github.io/pubs/nspfssd-lisa2004-final.pdf), LISA XVIII, 2004</cite>

Nix 的标志灵感来自 [Haskell 标志的一个构思](https://wiki.haskell.org/File:Sgf-logo-blue.png)，以及 [*nix* 在拉丁语中意为 *雪*](https://nix-dev.science.uu.narkive.com/VDaaP1BY/nix-logo)。

## 什么是 flakes？

参见 [](flakes-definition)。

(channel-branches)=
## 我应该使用哪个频道分支？

Nixpkgs 和 NixOS 都有稳定版与滚动版发布。

这些发布以称为「频道分支」（channel branches）的变体形式分发：
用于发布的 Git 分支，同时也会被转换为 Nix 频道。

:::{tip}
关于频道的更多信息，请查阅 Nix 参考手册中的 [`nix-channel`](https://nix.dev/manual/nix/2.22/command-ref/nix-channel) 条目；关于 Nixpkgs 的分支策略，请参阅 [Nixpkgs 贡献指南](https://github.com/NixOS/nixpkgs/blob/master/CONTRIBUTING.md#branch-conventions)。
:::

### 稳定版

稳定版仅会收到用于修复缺陷或安全漏洞的保守更新；除此之外不会更改软件包版本。
每六个月会发布一个新的稳定版。

- 在 Linux（包括 NixOS 和 WSL）上，使用 [`nixos-*`](https://github.com/NixOS/nixpkgs/branches/all?query=nixos-)。

  这些分支指向大多数 Linux 软件包已预先构建、并可从二进制缓存获取的提交。
  此外，这些提交已通过完整的 NixOS 测试套件。

- 在 macOS/Darwin 上，使用 [`nixpkgs-*-darwin`](https://github.com/NixOS/nixpkgs/branches/all?query=nixpkgs-)

  这些分支指向大多数 Darwin 软件包已预先构建、并可从二进制缓存获取的提交。

- 在其他任何平台上，使用上述哪一个都可以。

  Hydra 不会为其他平台预先构建任何二进制文件。

所有这些「频道分支」都跟随对应的 [`release-*`](https://github.com/NixOS/nixpkgs/branches/all?query=release-) 分支。

:::{admonition} 示例
`nixos-23.05` 和 `nixpkgs-23.05-darwin` 都基于 `release-23.05`。
:::

### 滚动版

滚动版跟随 [`master`](https://github.com/NixOS/nixpkgs/branches/all?query=master)，即主要开发分支。

- 在 Linux（包括 NixOS 和 WSL）上，使用 [`nixos-unstable`](https://github.com/NixOS/nixpkgs/branches/all?query=nixos-unstable)。
- 在其他任何平台上，使用 [`nixpkgs-unstable`](https://github.com/NixOS/nixpkgs/branches/all?query=nixpkgs-unstable)。

[`*-small`](https://github.com/NixOS/nixpkgs/branches/all?query=-small) 频道分支只通过了较小的测试套件，因此相对于其基础分支更新更及时，但稳定性保证更少。

## 沙箱构建中是否还存在不纯因素？

是的。包括：

- CPU 架构——人们正大力避免编译生成本机指令，转而使用硬编码的受支持指令。
- 系统的当前时间/日期。
- 用于构建的文件系统（另见 [`TMPDIR`](https://nix.dev/manual/nix/stable/command-ref/env-common.html#env-TMPDIR)）。
- Linux 内核参数，例如：
  - [IPv6 能力](https://github.com/NixOS/nix/issues/5615)。
  - binfmt 解释器，例如用 [`boot.binfmt.emulatedSystems`](https://search.nixos.org/options?show=boot.binfmt.emulatedSystems) 配置的解释器。
- 构建系统的时序行为——在某些情况下，并行 Make 构建可能无法获得正确的输入。
- 随机值的插入，例如来自 `/dev/random` 或 `/dev/urandom`。
- Nix 版本之间的差异。例如，新的 Nix 版本可能引入新的环境变量。像 `env > $out` 这样的语句，Nix 并不承诺在将来仍会产生相同的输出。
