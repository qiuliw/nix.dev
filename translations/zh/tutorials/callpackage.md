---
date: 2022-09-08
myst:
  html_meta:
    "keywords": "tutorial, callPackage, override, package, customise, parameters, nix, nixpkgs"
---

(callpackage-tutorial)=
# 使用 `callPackage` 进行包参数与覆盖

```{contributors}
:authors: NobbZ
:editors: mmesch, fricklerhandwerk
```

Nix 自带一门用于创建软件包与配置的专用编程语言：Nix 语言。
它被用来构建 Nix 软件包集合，即 {term}`Nixpkgs`。

作为纯函数式语言，Nix 语言允许声明自定义函数，以抽象常见模式。
Nixpkgs 中最突出的模式之一，就是对软件包配方进行参数化。

## 概览

Nixpkgs 本身也是一个体量可观的软件项目，多年来形成了自己的编码约定与惯用法。
它[确立了一种约定](https://github.com/nixos/nixpkgs/commit/d17f0f9cbca38fabb71624f069cd4c0d6feace92)：通过名为 [`callPackage`](https://github.com/NixOS/nixpkgs/commit/fd268b4852d39c18e604c584dd49a611dc795a9b) 的函数，将参数化软件包与自动填充的设置组合在一起。
本教程将说明如何使用它，以及它为何有益。

### 你将学到什么？

- 使用 `callPackage` 调用遵循 Nixpkgs 约定的软件包配方
- 覆盖软件包参数
- 创建相互依赖的软件包集合

### 你需要什么？

- 熟悉 [Nix 语言](reading-nix-language)
- 具备[打包已有软件](packaging-tutorial)的初步经验

### 需要多久？

- 45 分钟

## 自动函数调用

创建一个新文件 `hello.nix`，它可以是 Nixpkgs 中常见的典型软件包配方：
一个接受属性集的函数，属性对应于顶层软件包集合中的 derivation，并返回一个 derivation。

```{code-block} nix
:caption: hello.nix
{ writeShellScriptBin }:
writeShellScriptBin "hello" ''
  echo "Hello, world!"
''
```

:::{dropdown} 详细说明
`hello.nix` 声明了一个函数，其参数是一个包含单个元素 `writeShellScriptBin` 的属性集。
[`writeShellScriptBin`](https://nixos.org/manual/nixpkgs/unstable/#trivial-builder-writeShellScriptBin) 是 Nixpkgs 中恰好存在的一个函数，属于[构建辅助函数](https://nixos.org/manual/nixpkgs/unstable/#part-builders)，会返回一个 derivation。
在本例中，该 derivation 的输出包含一个位于 `$out/bin/hello` 的可执行 shell 脚本，运行时会打印 "Hello world"。
:::

现在创建文件 `default.nix`，内容如下：

```{code-block} nix
:caption: default.nix
let
  pkgs = import <nixpkgs> { };
in
pkgs.callPackage ./hello.nix { }
```

实现 `default.nix` 中的 derivation，并运行生成的可执行文件：

```shell-session
$ nix-build
$ ./result/bin/hello
Hello, world!
```

当求值 `hello.nix` 中的函数时，参数 `writeShellScriptBin` 会被自动填充。
对于函数参数中的每个属性，如果 `pkgs` 属性集中存在同名属性，`callPackage` 就会将其传入。

在如此简单的设置中，为软件包额外创建 `hello.nix` 文件似乎有些繁琐。
我们这样做，是因为这正是 Nixpkgs 的组织方式：
每个软件包配方都是一个声明函数的文件。
该函数以软件包的依赖作为参数。

## 参数化构建

修改 `default.nix`，使其产生一个由 derivation 组成的属性集，其中属性 `hello` 包含原来的 derivation：

```{code-block} nix
:caption: default.nix
let
  pkgs = import <nixpkgs> { };
in
{
  hello = pkgs.callPackage ./hello.nix { };
}
```

通过 [`-A` / `--attr` 选项](https://nix.dev/manual/nix/2.19/command-ref/nix-build#opt-attr) 访问并构建属性 `hello` 时，结果与之前相同：

```shell-session
$ nix-build -A hello
$ ./result/bin/hello
Hello, world!
```

同时修改 `hello.nix`，增加一个默认值为 `"world"` 的额外参数 `audience`：

```{code-block} nix
:caption: hello.nix
{
  writeShellScriptBin,
  audience ? "world",
}:
writeShellScriptBin "hello" ''
  echo "Hello, ${audience}!"
''
```

这同样不会改变结果。

当修改 `default.nix` 以利用这个新参数时，事情就变得更有意思了。
在传给 `callPackage` 的第二个参数中传入 `audience`：

```{code-block} diff
:caption: default.nix
 let
   pkgs = import <nixpkgs> { };
 in
 {
-  hello = pkgs.callPackage ./hello.nix { };
+  hello = pkgs.callPackage ./hello.nix { audience = "people"; };
 }
```

该属性会被传递给 `hello.nix` 中定义的函数的参数：
同样的语法也可以用来显式设置自动发现的参数（例如 `writeShellScriptBin`），但在这里没有必要。

试一下：

```shell-session
$ nix-build -A hello
$ ./result/bin/hello
Hello, people!
```

这种模式在 Nixpkgs 中被广泛使用：
例如，表示 Go 程序的函数通常有一个参数 `buildGoModule`。
常见的做法是写出类似 `callPackage ./go-program.nix { buildGoModule = buildGo116Module; }` 的表达式，以更改默认的 Go 编译器版本。
因此，Nixpkgs 不仅仅是一个庞大的预配置软件包库，更是一组函数——软件包*配方*——可以即时定制软件包乃至整个生态系统（例如「所有使用我的自定义解释器的 Python 包」），而无需复制代码。

# 覆盖

`callPackage` 还提供了更多便利：允许通过返回的 derivation 上的 `override` 函数，在*事后*定制参数。

在 `default.nix` 中添加第三个属性 `hello-folks`，并将其设为对 `hello.override` 的调用，并传入 `audience` 的新值：

```{code-block} diff
:caption: default.nix
 let
   pkgs = import <nixpkgs> { };
 in
-{
+rec {
   hello = pkgs.callPackage ./hello.nix { audience = "people"; };
+  hello-folks = hello.override { audience = "folks"; };
 }
```

:::{note}
得到的属性集现在是递归的（通过关键字 `rec`）。
也就是说，属性的值可以引用同一属性集中的名称。
:::

`override` 会把 `audience` 传给 `hello.nix` 中的原始函数——它会*覆盖*在最初生成 derivation `hello` 的 `callPackage` 中传入的相应参数。
其余参数保持不变。
这尤其有用，在那些提供大量选项以定制软件包的包中经常可以看到。

构建 `hello-folks` 属性并运行生成的可执行文件，将再次得到该脚本的一个新版本：

```shell-session
$ nix-build -A hello-folks
$ ./result/bin/hello
Hello, folks!
```

一个真实世界的例子是 [`neovim`](https://search.nixos.org/packages?show=neovim) 软件包配方，它有诸如 `extraLuaPackages`、`extraPythonPackages` 或 `withRuby` 等可覆盖参数。
目前这些参数只能通过阅读源代码来发现；可在 [search.nixos.org/packages](https://search.nixos.org/packages) 上跟随指向 📦 Source 的链接找到源码。

## 相互依赖的软件包集合

你实际上可以创建自己的 `callPackage` 版本！
当软件包集合中的配方彼此依赖时，这会很方便。

:::{note}
以下示例不展示被「调用」的文件，因为理解原理并不需要它们。
:::

考虑下面这个由 derivation 组成的递归属性集：

```{code-block} nix
:caption: default.nix
let
  pkgs = import <nixpkgs> { };
in
rec {
  a = pkgs.callPackage ./a.nix { };
  b = pkgs.callPackage ./b.nix { inherit a; };
  c = pkgs.callPackage ./c.nix { inherit b; };
  d = pkgs.callPackage ./d.nix { };
  e = pkgs.callPackage ./e.nix { inherit c d; };
}
```

:::{note}
这里，`inherit a;` 等价于 `a = a;`。
:::

先前声明的 derivation 通过 `callPackage` 作为参数传给其他 derivation。

在这种情况下，你必须记住：对每个软件包，在对应的 {term}`Nix file` 中手动指定所有不在 Nixpkgs 中的所需参数。
如果 `./b.nix` 需要参数 `a`，但没有 `pkgs.a`，函数调用就会报错。
这很快就会变得相当繁琐，尤其是在较大的软件包集合中。

使用 `lib.callPackageWith`，可以基于某个属性集创建你自己的 `callPackage`。

```{code-block} nix
:caption: default.nix
let
  pkgs = import <nixpkgs> { };
  callPackage = pkgs.lib.callPackageWith (pkgs // packages);
  packages = {
    a = callPackage ./a.nix { };
    b = callPackage ./b.nix { };
    c = callPackage ./c.nix { };
    d = callPackage ./d.nix { };
    e = callPackage ./e.nix { };
  };
in
packages
```

这需要一些解释。

首先注意，我们不再使用递归属性集，而是在 `let` 绑定中赋值要操作的名称。
它与递归集合有相同的性质：
等号（`=`）左侧的名称可以用在右侧的表达式中。
因此，当我们用 `//` 运算符把 `packages` 的内容与已有的属性集 `pkgs` 合并时，可以引用 `packages`。

你的自定义 `callPackages` 现在会把 `pkgs` *以及* `packages` 中的所有属性提供给被调用的软件包函数（`packages` 中的同名属性优先），并且 `packages` 会随着每次调用递归地构建起来。

最后一点可能会让人头晕。
这种构造之所以可行，是因为 Nix 语言是惰性求值的。
也就是说，只有在真正需要时才会计算值。
这使得可以在尚未完全定义 `packages` 的情况下把它传来传去。

每个软件包的依赖现在在这一层是隐式的（在各个软件包文件中仍然是显式的），而 `callPackage` 会*自动地*解析它们。
这让你不必手动处理依赖，并避免可能只在漫长构建过程后期才暴露的配置错误。

当然，这个小例子用原来的形式仍然可以管理。
而隐式递归的变体可能会让不熟悉惰性求值的软件开发者看不清结构，对他们来说反而比之前更难读。
但在大型构造中，这一好处真正显现出来：那时是代码量本身会掩盖结构，而手动修改会变得繁琐且容易出错。

## 小结

使用 `callPackage` 不仅遵循 Nixpkgs 约定（让有经验的 Nix 用户更容易跟进你的代码），还免费带来一些好处：

1. 参数化构建
2. 可覆盖的构建
3. 相互依赖软件包集合的简洁实现

## 参考资料

- [Nixpkgs 手册：`callPackageWith`](https://nixos.org/manual/nixpkgs/stable/#function-library-lib.customisation.callPackageWith)

## 下一步

- [](file-sets-tutorial) - 学习用 Nix 打包自己的项目
- [](module-system-deep-dive) - 学习驾驭 NixOS 背后的函数式编程魔法
