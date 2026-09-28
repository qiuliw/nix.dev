# 最佳实践

```{contributors}
:authors: domenkozar
:editors: fricklerhandwerk, infinisil
```

## URL

Nix 语言语法支持裸 URL，因此可以写 `https://example.com` 而不是 `"https://example.com"`

[RFC 45](https://github.com/NixOS/rfcs/pull/45) 已获接受，以弃用不加引号的 URL，并提供了
若干论证，说明该特性弊大于利。

:::{tip}
始终为 URL 加引号。
:::

(rec-expression)=
## 递归属性集 `rec { ... }`

`rec` 允许你在同一属性集内引用名称。

示例：

```{code-block} nix
:class: expression
rec {
  a = 1;
  b = a + 2;
}
```

```{code-block}
:class: value
{ a = 1; b = 3; }
```

一个常见陷阱是：在遮蔽名称时引入难以调试的错误 `infinite recursion`。
最简单的例子是：

```{code-block} nix
let a = 1; in rec { a = a; }
```

:::{tip}
避免使用 `rec`。改用 `let ... in`。

示例：

```{code-block} nix
:class: expression
let
  a = 1;
in {
  a = a;
  b = a + 2;
}
```
:::

:::{tip}
自引用可通过显式命名属性集来实现：

```{code-block} nix
:class: expression
let
  argset = {
    a = 1;
    b = argset.a + 2;
  };
in
  argset
```
:::

## `with` 作用域

在实际代码中仍常见如下表达式：

```{code-block} nix
:class: expression
with (import <nixpkgs> {});

# ... lots of code
```

这会将被导入表达式的所有属性引入当前表达式的作用域。

这种方式存在问题：

- 静态分析无法对该代码进行推理，因为必须实际求值该文件才能知道哪些名称在作用域中。
- 当使用多个 `with` 时，名称来自何处就不再清晰。
- `with` 的作用域规则并不直观，详见此 [Nix issue](https://github.com/NixOS/nix/issues/490)。

:::{tip}
不要在 Nix 文件顶部使用 `with`。
在 `let` 表达式中显式赋名。

示例：

```{code-block} nix
:class: expression
let
  pkgs = import <nixpkgs> {};
  inherit (pkgs) curl jq;
in

# ...
```
:::

较小的作用域通常问题较少，但因作用域规则仍可能带来意外。

:::{tip}
若想完全避免 `with`，可尝试将如下形式的表达式

```{code-block} nix
:class: expression
buildInputs = with pkgs; [ curl jq ];
```

替换为：

```{code-block} nix
:class: expression
buildInputs = builtins.attrValues {
  inherit (pkgs) curl jq;
};
```
:::

(search-path)=
## `<...>` 查找路径

你经常会遇到引用 `<nixpkgs>` 的 Nix 语言代码示例。

`<...>` 是一种特殊语法，[于 2011 年引入]，用于方便地访问环境变量 [`$NIX_PATH`] 中的值。

[于 2011 年引入]: https://github.com/NixOS/nix/commit/1ecc97b6bdb27e56d832ca48cdafd3dbb5185a04
[`$NIX_PATH`]: https://nix.dev/manual/nix/stable/command-ref/env-common.html#env-NIX_PATH

这意味着查找路径的值取决于外部系统状态。
使用查找路径时，同一 Nix 表达式可能产生不同结果。

在大多数情况下，安装 Nix 时 `$NIX_PATH` 会被设置为最新 channel，因此很可能因机器而异。

:::{note}
[Channels](https://nix.dev/manual/nix/stable/command-ref/nix-channel.html) 是一种引用远程 Nix 表达式并获取其最新版本的机制。
:::

所订阅 channel 的状态外在于依赖它的 Nix 表达式。
它不易跨机器移植。
这可能限制可复现性。

例如，两台不同机器上的开发者很可能让 `<nixpkgs>` 指向 {term}`Nixpkgs` 仓库的不同修订。
构建可能对一人成功、对另一人失败，从而造成困惑。

:::{tip}
使用 [](pinning-nixpkgs) 中展示的技术显式声明依赖。

除最小示例外，不要使用查找路径。
:::

有些工具期望查找路径已设置。在这种情况下：

::::{tip}
在版本控制下的中心位置将 `$NIX_PATH` 设为已知值。

:::{admonition} NixOS
在 NixOS 上，可用 [`nix.nixPath`](https://search.nixos.org/options?show=nix.nixPath) 选项永久设置 `$NIX_PATH`。
:::
::::

(nixpkgs-config)=
## 可复现的 Nixpkgs 配置

为快速获取软件包用于演示，我们使用如下简洁模式：

```nix
import <nixpkgs> {}
```

然而，即使按 [](pinning-nixpkgs) 所示替换了 `<nixpkgs>`，结果仍可能不完全可复现。
这是因为出于历史原因，[Nixpkgs 顶层表达式]默认会不纯地从文件系统读取以获取配置参数。
已填充相应文件的系统可能得到不同结果。

[Nixpkgs 顶层表达式]: https://github.com/NixOS/nixpkgs/blob/master/default.nix

这是一个众所周知的问题，无法在不破坏现有配置的情况下解决。

:::{tip}
导入 Nixpkgs 时显式设置 [`config`](https://nixos.org/manual/nixpkgs/stable/#chap-packageconfig) 与 [`overlays`](https://nixos.org/manual/nixpkgs/stable/#chap-overlays)：


```nix
import <nixpkgs> { config = {}; overlays = []; }
```
:::

我们在教程中就是这样做的，以确保示例行为与预期完全一致。
在最小示例中我们会跳过，以减少干扰。

## 更新嵌套属性集

[属性集更新运算符](https://nix.dev/manual/nix/stable/language/operators.html#update)会合并两个属性集。

示例：

```{code-block} nix
:class: expression
{ a = 1; b = 2; } // { b = 3; c = 4; }
```

```{code-block} nix
:class: value
{ a = 1; b = 3; c = 4; }
```

然而，右侧的名称优先，且更新是浅层的。

示例：

```{code-block} nix
:class: expression
{ a = { b = 1; }; } // { a = { c = 3; }; }
```

```{code-block} nix
:class: value
{ a = { c = 3; }; }
```

这里，键 `b` 被完全移除，因为整个 `a` 的值被替换了。

:::{tip}
使用 Nixpkgs 函数 [`pkgs.lib.recursiveUpdate`](https://nixos.org/manual/nixpkgs/stable/#function-library-lib.attrsets.recursiveUpdate)：

```{code-block} nix
:class: expression
let pkgs = import <nixpkgs> {}; in
pkgs.lib.recursiveUpdate { a = { b = 1; }; } { a = { c = 3;}; }
```

```{code-block} nix
:class: value
{ a = { b = 1; c = 3; }; }
```
:::

## 可复现的源路径

```{code-block} nix
:class: expression
let pkgs = import <nixpkgs> {}; in

pkgs.stdenv.mkDerivation {
  name = "foo";
  src = ./.;
}
```

若包含该表达式的 Nix 文件位于 `/home/myuser/myproject`，则 `src` 的 store 路径将是 `/nix/store/<hash>-myproject`。

问题在于，此时你的构建不再可复现，因为它依赖于父目录名。
这无法在源码中声明，并会引入不纯度。

若有人在名称不同的目录中构建该项目，他们会为 `src` 及其依赖项得到不同的 store 路径。
这可能成为不必要重建的原因。

:::{tip}
使用 [`builtins.path`](https://nix.dev/manual/nix/stable/language/builtins.html#builtins-path)，并将 `name` 属性设为固定值。

这将根据 `name` 而不是工作目录派生 store 路径的符号名：

```{code-block} nix
:class: expression
let pkgs = import <nixpkgs> {}; in

pkgs.stdenv.mkDerivation {
  name = "foo";
  src = builtins.path { path = ./.; name = "myproject"; };
}
```
:::
