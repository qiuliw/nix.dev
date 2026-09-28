(flakes-definition)=
# Flakes

```{contributors}
:authors: kiara
```

## 什么是 flakes？

Flakes 提供了一个名为 `flake.nix` 的入口文件，旨在共享 Nix 代码。
它们让使用相同版本构建程序变得容易。

`flake.nix` 是一个以[标准结构][standard structure]声明输入与输出的文件。

> 注意：[实验性功能][Experimental]，需要 [Nix 2.4]。

该文件可以如下所示：

```nix
{
  description = "My example flake";

  inputs = {
    nixpkgs.url = "github:nixos/nixpkgs?ref=nixos-unstable";
  };

  outputs = { self, nixpkgs }: {
    packages.x86_64-linux = {
      default = self.packages.x86_64-linux.hello;
      hello = nixpkgs.legacyPackages.x86_64-linux.hello;
    };
  };
}
```

[`outputs`] 包含多种[内置类型][built-in types]，但也可以[扩展][extended]。
你可以在 [wiki] 上找到这些类型的概览。

[`inputs`] 让你声明依赖项。

一旦你运行 [`nix` command]，Nix 会创建 [`flake.lock`] 来固定依赖项。

如果这些依赖项本身也有 `inputs`，Nix 会检查*它们的*锁文件以确定要使用的版本。
使用相同版本有助于确保程序按预期工作，但你也可以覆盖这些版本。

[`nix` command] 默认原生与 flakes 集成。

```bash
nix build github:NixOS/nixpkgs#hello
```

你可以传入指向本地（例如 `.`）或远程（例如 `github:NixOS/nixpkgs`）项目目录的[引用][references]。
更多细节请参阅 [`nix` command] 的参考文档。

指向 flakes 的别名存储在[注册表][registry]中。
可以通过[命令行][command-line]或 {term}`NixOS` 选项 [`nix.registry`] 进行扩展。

[^subset]: Flakes 默认使用纯模式，使构建与主机环境隔离。
这也称为密封求值（hermetic evaluation），并阻止求值（非网络的）[impure] 函数。
Flake 的 `inputs` 和元数据字段不能是任意的 Nix 表达式。
这是为了防止复杂的、可能永不终止的计算。
`outputs` 字段的函数参数必须显式指定：它不支持 [η 归约][eta-reduction]。

NixOS 手册进一步解释了[基于 flake 的安装][flake-based installs]。

[Experimental]: https://nix.dev/manual/nix/stable/development/experimental-features#xp-feature-flakes
[Nix 2.4]: https://nix.dev/manual/nix/stable/release-notes/rl-2.4.html#highlights
[standard structure]: https://nix.dev/manual/nix/stable/command-ref/new-cli/nix3-flake.html#flake-format
[`nix` command]: https://nix.dev/manual/nix/stable/command-ref/new-cli/nix.html
[references]: https://nix.dev/manual/nix/stable/command-ref/new-cli/nix3-flake#flake-references
[`outputs`]: https://wiki.nixos.org/wiki/Flakes#Output_schema
[built-in types]: https://github.com/NixOS/nix/blob/38c755f168b7c38cd4687aacf5d7e59f049658d3/src/nix/flake.cc#L594-L769
[extended]: https://github.com/NixOS/nix/blob/38c755f168b7c38cd4687aacf5d7e59f049658d3/src/nix/flake.cc#L772-L776
[wiki]: https://wiki.nixos.org/wiki/Flakes#Output_schema
[`inputs`]: https://nix.dev/manual/nix/stable/command-ref/new-cli/nix3-flake.html#flake-inputs
[`flake.lock`]: https://nix.dev/manual/nix/stable/command-ref/new-cli/nix3-flake.html#lock-files
[impurities]: https://nix.dev/manual/nix/stable/tutorials/nix-language.html#impurities
[registry]: https://github.com/NixOS/flake-registry
[command-line]: https://nix.dev/manual/nix/2.28/command-ref/new-cli/nix3-registry.html
[`nix.registry`]: https://search.nixos.org/options?channel=unstable&show=nix.registry&query=registry
[eta-reduction]: https://wiki.haskell.org/Eta_conversion
[flake-based installs]: https://nixos.org/manual/nixos/stable/#sec-installation-manual-installing

## 我应该在项目中使用 flakes 吗？

Flakes 是一种仍有未解决问题的实验性扩展格式。
其功能通常也可以不通过 flakes 实现。

如果你需要运行已经使用 flakes 的现有软件，或想为其开发做贡献，可以放心使用它们。
如果你想自己编写 Nix 代码，也可以考虑我们关于[依赖管理][dependency management]的指南。[^flake-inputs]
这份概览可以帮助你从 flakes 中获得所需能力，同时保持兼容性。

[dependency management]: https://nix.dev/guides/recipes/dependency-management.html

[^flake-inputs]: 仅提供 flake 入口点的 Nix 仓库可以使用 [`flake-inputs`] 导入。

### 可发现性

使用 flakes 的第一步是添加一个指定 `outputs` 的 `flake.nix` 文件。

优点：

- 使用来自其他 flake 项目的代码。
- Nix 会检查 `flake.nix` 的结构是否有效。

缺点：

- Flakes 没有[参数][parameters]。
  这意味着 `flake.nix` 及其最终用户必须显式说明所用的 [`system`]。
  像 [`flake-utils`] 这类工具可以让这件事更容易。
- 作为实验性功能，flakes 仍可能发生变化。

[parameters]: https://github.com/NixOS/nix/issues/2861
[`system`]: https://github.com/NixOS/nix/issues/3843
[`flake-utils`]: https://github.com/numtide/flake-utils

替代方案：

- 将 flakes 用作现有 Nix 代码上的薄包装。
  这样，代码可以两种方式使用。
- 使用 Nix 模块：flake 用户可以用 `flake = false;` 导入它们。
- 自 NixOS 26.05 起，可以从单个 [`system.nix` entrypoint]加载一个或多个 NixOS 配置。

[`import`]: https://nix.dev/tutorials/nix-language#import
[`system.nix` entrypoint]: https://nixos.org/manual/nixos/stable/release-notes.html#sec-release-26.05-highlights

### 运行命令

Flakes 通过 Nix 的 [v3 `nix` 命令行界面][v3 `nix` command line interface]使用。
它可以通过像 `.` 或 `github:NixOS/nixpkgs` 这样的引用构建或运行程序。

你可以通过添加以下选项为单条命令启用它：

```
 --experimental-features 'nix-command flakes'
````

或在 NixOS 或 Home Manager 配置中永久启用：

```
nix.settings.experimental-features = [ "nix-command" "flakes" ];
```

要构建 `flake.nix` 中 `packages.x86_64-linux.default` 的 derivation，运行：

```bash
nix build .#packages.x86_64-linux.default
```

你可以将其缩写为 `nix build .#default`，或直接使用 `nix build`。

`nix run` 运行 `outputs.apps` 中的程序。
`nix run .#default` 运行 `outputs.apps.default`。
仅使用 `nix run` 也会运行该程序。

例如，要从 {term}`Nixpkgs` 运行 `hello` 软件包：

```bash
nix run nixpkgs#hello -- --greeting "hello from flakes"
```

这会使用你的注册表别名 `nixpkgs` 所设定的版本。

要从 {term}`Nixpkgs` 的 `nixpkgs-unstable` 分支运行 `hello` 软件包：

```bash
nix run github:NixOS/nixpkgs/nixpkgs-unstable#hello
```

[v3 `nix` command line interface]: https://nix.dev/manual/nix/stable/command-ref/new-cli/nix.html

优点：

- Flakes 会缓存构建结果，以便之后相同构建节省时间。
  例如，如果你在持续集成中运行未更改的构建，这可以节省时间。
- Flakes 推动让程序易于运行，也包括从远程仓库运行。
- Flakes 默认以[纯模式][pure mode]运行。
  这推动一种更可能使程序可复现的编写风格[^reproducible]。
- 对于使用 [Git] 的项目，flakes 只构建已跟踪的文件。
  这有助于防止重新构建。

[pure mode]: https://nix.dev/manual/nix/stable/tutorials/nix-language.html#impurities

[^reproducible]: 即使在纯模式下，可复现性[实际上也无法保证][not actually guaranteed]。

[not actually guaranteed]: https://discourse.nixos.org/t/nix-flakes-explained-what-they-solve-why-they-matter-and-the-future/72302/7

缺点：

- 构建会将整个 flake 目录复制到 Nix store。
  这会缓存它们，但对于像 {term}`Nixpkgs` 这样的大型仓库可能会更[慢][slower]。
- 实现对于 [flakes] 和 [v3 CLI] 仍存在问题。
- 文件必须先被暂存（staged），flakes 才能看到它们。

[slower]: https://github.com/NixOS/nix/issues/3121
[flakes]: https://github.com/NixOS/nix/issues?q=is%3Aissue+is%3Aopen+label%3Aflakes+sort%3Areactions-%2B1-desc
[v3 CLI]: https://github.com/NixOS/nix/issues?q=is%3Aissue+is%3Aopen+label%3Anew-cli+sort%3Areactions-%2B1-desc

替代方案：

- 普通 Nix 文件可以与 [v2 commands]（如 [`nix-build`]、[`nix-shell`]）一起使用，或与 v3 命令的 [`--file` flag]或 `-f` 一起使用。
- 在 {term}`NixOS` 中，你可以通过在 NixOS 配置中设置 [`nixpkgs.flake.source = pkgs.path;`]，让 v3 命令的 `nixpkgs` 指向软件包集合 `pkgs`。
  另见[管理依赖][managing dependencies]。
- 使用实验性功能 [`fetch-tree`] 中的 [`builtins.fetchTree`]，可以为非 flake 入口点模拟[^emulated] [`nix run`]。

[Git]: https://git-scm.com/
[v2 CLI]: https://nix.dev/manual/nix/stable/command-ref/main-commands
[`--file` flag]: https://nix.dev/manual/nix/stable/command-ref/new-cli/nix3-build.html#options-that-change-the-interpretation-of-installables
[`nix-build`]: https://nix.dev/manual/nix/stable/command-ref/nix-build.html
[`nix-shell`]: https://nix.dev/manual/nix/stable/command-ref/nix-shell.html
[`nixpkgs.flake.source = pkgs.path;`]: https://search.nixos.org/options?channel=unstable&show=nixpkgs.flake.source&query=nixpkgs.flake.source
[managing dependencies]: https://nix.dev/guides/recipes/dependency-management#managing-nixos-configurations
[`builtins.fetchTree`]: https://noogle.dev/f/builtins/fetchTree
[`fetch-tree`]: https://nix.dev/manual/nix/stable/development/experimental-features#xp-feature-fetch-tree
[`nix run`]: https://nix.dev/manual/nix/stable/command-ref/new-cli/nix3-run.html

[^emulated]: 对于非 flake 项目，`nix run github:NixOS/nixpkgs#hello` 可能看起来像 `nix-shell -p '(import (builtins.fetchTree "github:NixOS/nixpkgs").outPath { }).hello' --run 'hello'`。
可以使用 Bash 别名定义一个沿用 `nix run` 语法的即用命令 `nix-run`，例如 `alias nix-run='run() { $(nix-instantiate --raw --impure --eval --expr "(import <nixpkgs> {}).lib.getExe (import (builtins.fetchTree \"$(cut -d "#" -f 1 <<< "$1")\").outPath { }).$(cut -d "#" -f 2 <<< "$1")"); }; run'`。

### 依赖管理

Flakes 的 `inputs` 属性可以管理依赖项。
默认情况下，这会隐式处理递归依赖。
如果某个库被多次使用，这可能给出同一库的不同版本。
如果你愿意，可以使用 `follows` 语句覆盖它们：

```
{
  inputs = {
    nixpkgs.url = "github:nixos/nixpkgs?ref=nixos-unstable";
    home-manager = {
      url = "github:nix-community/home-manager";
      inputs.nixpkgs.follows = "nixpkgs";
    };
  };
}
```

这样，Home Manager 的 inputs 会复用你选择的 `nixpkgs`。

优点：

- 便于复现已发布的软件，并跟随它们所用的版本。
- 你可以覆盖递归 inputs。

缺点：

- Flakes 默认跟随依赖项 `flake.lock` 中的版本，因此如果你不用 `follows` 覆盖它们，你可能会得到：
  - 同一依赖项的多个版本。
  - 过时的依赖项（如果其版本未积极更新）。
- 依赖项会被急切地获取，加载你可能并不使用的依赖项。
- 如果你自己不使用 flakes，则由 flake inputs 管理的依赖项很难覆盖。
- 如果你的 flake 被用作库，你需要为所有递归 inputs 添加 `follows` 语句。
  否则下游使用者无法对你的间接 inputs 添加他们的 `follows`。

替代方案：

- 用 [`npins`] 处理依赖项。
- 使用 [fetchers] 或 [`builtins.fetchTree`] 等函数内联处理依赖项。
- 使用 `inputs.<name>.flake = false;`

[fetchers]: https://nixos.org/manual/nixpkgs/stable/#chap-pkgs-fetchers
[`flake-inputs`]: https://github.com/fricklerhandwerk/flake-inputs
[`npins`]: https://nix.dev/guides/recipes/dependency-management.html

### 仅使用 Flake 的 Nix

由于 flakes（在很大程度上[^subset]）将 Nix 用作内部语言，你甚至可以把所有 Nix 代码放在 flake 文件中。
在社区中，这种编码风格称为 dendritic pattern。
使用 flakes 时，借助库 [`flake-parts`](https://github.com/hercules-ci/flake-parts)，这种模式会更容易，
它让你把代码分散到不同的类 flake 文件中。

优点：

- 有助于从任何此类 Nix 代码中使用 flakes 的模式（schema）和 inputs。

缺点：

- 使得在不使用 flakes 的情况下访问这些代码更加困难。

替代方案：

- 将 flakes 用作现有 Nix 代码上的薄包装，这样代码可以两种方式使用。
- 使用库 [`flake-compat`]，将 flake 的默认软件包或 shell 暴露给非 flake 用户。
- 在模块之间显式传递所需参数，或使用 NixOS 的 [`specialArgs`]。

[`flake-compat`]: https://github.com/NixOS/flake-compat
[`specialArgs`]: https://nixos.org/manual/nixos/unstable/options#opt-_module.args

### 历史

- 构想
  - Flakes 在 [RFC 49] 中提出，并在一篇[博文][blog post]中介绍。
- 设计
  - flakes 提案因试图一次解决太多问题、且处于错误的抽象层而受到批评。
  - 设计仍有[各种问题][various problems]，包括版本控制、可组合性、交叉编译以及与 nixpkgs 的紧耦合等。
  - 围绕 `nix` CLI 仍有许多[未解决的设计问题][open design questions]。
- 实现
  - 实现仍存在[问题][problems with the implementation]。
- 流程
  - 尽管对设计仍有未解决的关切，
    实现在 RFC 尚未被接受的情况下就被合并了（实际上在合并时被撤回），
    引发了关于正当流程的质疑。
  - RFC 在没有给出结束实验时间表的情况下被关闭。
  - Flakes 已被许多项目依赖，使得在不破坏许多人代码的情况下迭代其设计更加困难。
- 社区
  - 该设计并未被社区的所有部分接受，例如 {term}`Nixpkgs` 并未在其内部工具中使用它。
    因此出现了围绕 flakes 的分支方案，
    例如 Determinate Systems 公司（它提供围绕 flakes 的专有功能）单方面宣布该功能已稳定，
    而社区驱动的 Nix 分支 Lix 则将[功能集收敛][consolidated the featureset]到事实上的「v1」。
    这种分支有可能最终破坏最初推动 flakes 实验的统一接口承诺。

[RFC 49]: https://github.com/NixOS/rfcs/pull/49
[blog post]: https://tweag.io/blog/2020-05-25-flakes/
[various problems]: https://wiki.lix.systems/books/lix-contributors/page/flakes-feature-freeze#bkmrk-design-issues-of-fla
[open design questions]: https://github.com/NixOS/nix/issues?q=is%3Aissue+is%3Aopen+label%3Anew-cli+sort%3Areactions-%2B1-desc
[problems with the implementation]: https://github.com/NixOS/nix/issues?q=is%3Aissue+is%3Aopen+label%3Aflakes+sort%3Areactions-%2B1-desc
[consolidated the featureset]: https://wiki.lix.systems/books/lix-contributors/page/flake-stabilisation-proposal

## 延伸阅读

- [wiki 文章](https://wiki.nixos.org/wiki/Flakes)
- [Flakes aren't real and cannot hurt you: a guide to using Nix flakes the non-flake way](https://jade.fyi/blog/flakes-arent-real/)（Jade Lovelace，2024 年 1 月）
- [Nix Flakes is an experiment that did too much at once...](https://samuel.dionne-riel.com/blog/2023/09/06/flakes-is-an-experiment-that-did-too-much-at-once.html)（[评论](https://discourse.nixos.org/t/nix-flakes-is-an-experiment-that-did-too-much-at-once/32707)）（Samuel Dionne-Riel，2023 年 9 月）
- [Experimental does not mean unstable](https://determinate.systems/posts/experimental-does-not-mean-unstable)（[评论](https://discourse.nixos.org/t/experimental-does-not-mean-unstable-detsyss-perspective-on-nix-flakes/32703)）（Graham Christensen，2023 年 9 月）
- [The Nix Hour: comparing flakes to traditional Nix](https://www.youtube.com/watch?v=atmoYyBAhF4)（Silvan Mosberger，2022 年 11 月）
