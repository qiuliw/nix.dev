---
myst:
  html_meta:
    "description lang=en": "Cross compilation tutorial using Nix"
    "keywords": "Nix, cross compilation, cross-compile, Nix"
---


(cross-compilation)=

# 交叉编译

Nixpkgs 提供了用于为不同系统类型交叉编译软件的工具。

## 你需要什么？


- 使用 C 编译器的经验
- [Nix 语言](<reading-nix-language>)的基础知识

## 平台

编译代码时，需要区分 **构建平台（build platform）**——可执行文件在此*构建*，以及 **宿主平台（host platform）**——编译后的可执行文件在此*运行*。[^id3]

**本地编译（Native compilation）** 是这两个平台相同的特殊情况。
**交叉编译（Cross compilation）** 则是这两个平台不同的一般情况。

当宿主平台资源有限（例如 CPU）或不便于开发访问时，就需要进行交叉编译。

Nixpkgs 对交叉编译有经过充分测试的支持。

[^id3]: 交叉编译平台的术语在不同构建系统之间有所不同。
    我们选择遵循
    [autoconf 术语](https://www.gnu.org/software/autoconf/manual/autoconf-2.69/html_node/Hosts-and-Cross_002dCompilation.html)。

## 什么是目标平台？

还有第三个平台概念，称为 **目标平台（target platform）**。

目标平台适用于你想要构建编译器二进制文件的情况。
在这种情况下，你会在 *构建平台* 上构建编译器，在 *宿主平台* 上运行它来编译代码，并在 *目标平台* 上运行最终的可执行文件。

由于这种情况很少需要，本教程假设目标平台与宿主平台相同。

## 确定宿主平台配置

构建平台由 Nix 在配置阶段自动确定。

宿主平台最好通过在宿主平台上运行以下命令来确定：

```shell-session
$ $(nix-build '<nixpkgs>' -I nixpkgs=channel:nixos-23.11 -A gnu-config)/config.guess
aarch64-unknown-linux-gnu
```

如果无法这样做（例如宿主平台不便于开发访问），则需要按以下模板手动构造平台配置：

```
<cpu>-<vendor>-<os>-<abi>
```

这种字符串表示法因历史原因用于 `nixpkgs`。

请注意，`<vendor>` 通常是 `unknown`，而 `<abi>` 是可选的。
平台也没有唯一标识符，例如 `unknown` 和 `pc` 可以互换（这也是该脚本名为 `config.guess` 的原因）。

如果无法安装 Nix，请设法在宿主平台上可运行的操作系统中运行 `config.guess`（通常随 autoconf 软件包提供）。

其他常见的平台配置示例：

- aarch64-apple-darwin14
- aarch64-pc-linux-gnu
- x86_64-w64-mingw32
- aarch64-apple-ios

:::{note}
macOS/Darwin 是一个特例，因为并非整个操作系统都是开源的。
只能在 `aarch64-darwin` 和 `x86_64-darwin` 之间进行交叉编译。
`aarch64-darwin` 支持最近才添加，因此交叉编译几乎未经测试。
:::

## 用 Nix 选择宿主平台

`nixpkgs` 自带一组用于交叉编译的预定义宿主平台，称为 `pkgsCross`。

可以在 `nix repl` 中探索它们：

:::{note}
[从 Nix 2.19 开始](https://nix.dev/manual/nix/latest/release-notes/rl-2.19)，`nix repl` 需要 `-f` / `--file` 标志：
```shell-session
$ nix repl -f '<nixpkgs>' -I nixpkgs=channel:nixos-23.11
```
:::

```shell-session
$ nix repl '<nixpkgs>' -I nixpkgs=channel:nixos-23.11
Welcome to Nix 2.18.1. Type :? for help.

Loading '<nixpkgs>'...
Added 14200 variables.

nix-repl> pkgsCross.<TAB>
pkgsCross.aarch64-android             pkgsCross.musl-power
pkgsCross.aarch64-android-prebuilt    pkgsCross.musl32
pkgsCross.aarch64-darwin              pkgsCross.musl64
pkgsCross.aarch64-embedded            pkgsCross.muslpi
pkgsCross.aarch64-multiplatform       pkgsCross.or1k
pkgsCross.aarch64-multiplatform-musl  pkgsCross.pogoplug4
pkgsCross.aarch64be-embedded          pkgsCross.powernv
pkgsCross.amd64-netbsd                pkgsCross.ppc-embedded
pkgsCross.arm-embedded                pkgsCross.ppc64
pkgsCross.armhf-embedded              pkgsCross.ppc64-musl
pkgsCross.armv7a-android-prebuilt     pkgsCross.ppcle-embedded
pkgsCross.armv7l-hf-multiplatform     pkgsCross.raspberryPi
pkgsCross.avr                         pkgsCross.remarkable1
pkgsCross.ben-nanonote                pkgsCross.remarkable2
pkgsCross.fuloongminipc               pkgsCross.riscv32
pkgsCross.ghcjs                       pkgsCross.riscv32-embedded
pkgsCross.gnu32                       pkgsCross.riscv64
pkgsCross.gnu64                       pkgsCross.riscv64-embedded
pkgsCross.i686-embedded               pkgsCross.scaleway-c1
pkgsCross.iphone32                    pkgsCross.sheevaplug
pkgsCross.iphone32-simulator          pkgsCross.vc4
pkgsCross.iphone64                    pkgsCross.wasi32
pkgsCross.iphone64-simulator          pkgsCross.x86_64-embedded
pkgsCross.mingw32                     pkgsCross.x86_64-netbsd
pkgsCross.mingwW64                    pkgsCross.x86_64-netbsd-llvm
pkgsCross.mmix                        pkgsCross.x86_64-unknown-redox
pkgsCross.msp430
```

这些交叉编译软件包的属性名是随着时间推移较为随意地选定的。
它们通常与对应的平台配置字符串不匹配。

你可以从 `pkgsCross.<platform>.stdenv.hostPlatform.config` 获取平台字符串：

```shell-session
nix-repl> pkgsCross.aarch64-multiplatform.stdenv.hostPlatform.config
"aarch64-unknown-linux-gnu"
```

如果你所需的宿主平台尚未定义，请[向上游贡献它](https://github.com/NixOS/nixpkgs/blob/master/lib/systems/examples.nix)。

## 指定宿主平台

设置交叉编译的机制如下：

1. 获取构建平台配置，并将其应用到当前软件包集（按惯例称为 `pkgs`）。

   在 `pkgs = import <nixpkgs> {}` 中，构建平台被隐含为当前系统。
   这会生成一个构建环境 `pkgs.stdenv`，其中包含在构建平台上编译所需的所有依赖。

2. 将适当的宿主平台配置应用到 `pkgsCross` 中的所有软件包。

   获取 `pkgs.pkgsCross.<host>.hello` 将生成在构建平台上编译、可在 `<host>` 平台上运行的软件包 `hello`。

有多种等价方式可以访问面向宿主平台的软件包。

1. 在构建平台环境中显式选取宿主平台软件包：

   ```nix
   let
     nixpkgs = fetchTarball "https://github.com/NixOS/nixpkgs/tarball/release-23.11";
     pkgs = import nixpkgs {};
   in
   pkgs.pkgsCross.aarch64-multiplatform.hello
   ```

2. 在导入 `nixpkgs` 时将宿主平台传给 `crossSystem`。
   这会配置 `nixpkgs`，使其所有软件包都为宿主平台构建：

   ```nix
   let
     nixpkgs = fetchTarball "https://github.com/NixOS/nixpkgs/tarball/release-23.11";
     pkgs = import nixpkgs { crossSystem = { config = "aarch64-unknown-linux-gnu"; }; };
   in
   pkgs.hello
   ```

   等价地，你可以将宿主平台作为参数传给 `nix-build`：

   ```sh
   $ nix-build '<nixpkgs>' -I nixpkgs=channel:nixos-23.11 \
     --arg crossSystem '{ config = "aarch64-unknown-linux-gnu"; }' \
     -A hello
   ```

## 第一次交叉编译

要交叉编译像 [hello](https://www.gnu.org/software/hello/) 这样的软件包，选择平台属性——在我们的例子中是 `aarch64-multiplatform`——并运行：

```shell-session
$ nix-build '<nixpkgs>' -I nixpkgs=channel:nixos-23.11 \
  -A pkgsCross.aarch64-multiplatform.hello
...
/nix/store/1dx87l5rav8679lqigf9xxkb7wvh2m4k-hello-aarch64-unknown-linux-gnu-2.12.1
```

:::{note}
软件包在 store 路径中的哈希会随着 channel 的更新而变化。
:::

[搜索软件包](https://search.nixos.org/packages)属性名，以找到你想要构建的那个。

## 实际交叉编译一个 Hello World 示例

以下示例将一个 Hello World 程序交叉编译为静态可执行文件，目标平台为 `armv6l-unknown-linux-gnueabihf` 和 `x86_64-w64-mingw32`（Windows），并使用[模拟器](https://en.wikipedia.org/wiki/Emulator)运行生成的可执行文件。

假设我们有一个 `cross-compile.nix`：

```nix
let
  nixpkgs = fetchTarball "https://github.com/NixOS/nixpkgs/tarball/release-23.11";
  pkgs = import nixpkgs {};

  # Create a C program that prints Hello World
  helloWorld = pkgs.writeText "hello.c" ''
    #include <stdio.h>

    int main (void)
    {
      printf ("Hello, world!\n");
      return 0;
    }
  '';

  # A function that takes host platform packages
  crossCompileFor = hostPkgs:
    # Run a simple command with the compiler available
    hostPkgs.runCommandCC "hello-world-cross-test" {} ''
      # Wine requires home directory
      HOME=$PWD

      # Compile our example using the compiler specific to our host platform
      $CC ${helloWorld} -o hello

      # Run the compiled program using user mode emulation (Qemu/Wine)
      # buildPackages is passed so that emulation is built for the build platform
      ${hostPkgs.stdenv.hostPlatform.emulator hostPkgs.buildPackages} hello > $out

      # print to stdout
      cat $out
    '';
in {
  # Statically compile our example using the two platform hosts
  rpi = crossCompileFor pkgs.pkgsCross.raspberryPi;
  windows = crossCompileFor pkgs.pkgsCross.mingwW64;
}
```

如果我们构建这个示例并打印两个生成的 derivation，应该分别为每个看到 "Hello, world!"：

```shell-session
$ cat $(nix-build cross-compile.nix)
Hello, world!
Hello, world!
```

## 带交叉编译器的开发环境

在 {ref}`声明式可复现环境教程 <declarative-reproducible-envs>` 中，我们了解了 Nix 如何帮助我们为项目提供工具和系统库。

也可以提供一个配置了 **使用 musl 交叉编译为静态二进制文件** 的编译器的环境。

假设我们有一个 `shell.nix`：

```nix
let
  nixpkgs = fetchTarball "https://github.com/NixOS/nixpkgs/tarball/release-23.11";
  pkgs = (import nixpkgs {}).pkgsCross.aarch64-multiplatform;
in

# callPackage is needed due to https://github.com/NixOS/nixpkgs/pull/126844
pkgs.pkgsStatic.callPackage ({ mkShell, zlib, pkg-config, file }: mkShell {
  # these tools run on the build platform, but are configured to target the host platform
  nativeBuildInputs = [ pkg-config file ];
  # libraries needed for the host platform
  buildInputs = [ zlib ];
}) {}
```

以及 `hello.c`：

```{code-block} c hello.c
#include <stdio.h>

int main (void)
{
  printf ("Hello, world!\n");
  return 0;
}
```

我们可以对其进行交叉编译：

```shell-session
$ nix-shell --run '$CC hello.c -o hello' shell.nix
```

并确认它是 aarch64：

```shell-session
$ nix-shell --run 'file hello' shell.nix
hello: ELF 64-bit LSB executable, ARM aarch64, version 1 (SYSV), statically linked, with debug_info, not stripped
```

## 下一步

- [官方二进制缓存](https://cache.nixos.org)中可供交叉编译软件包使用的二进制文件数量有限，因此为节省重新编译的时间，请配置 {ref}`你自己的二进制缓存以及使用 GitHub Actions 的 CI <github-actions>`。

- 虽然 Nixpkgs 中许多编译器支持交叉编译，但并非全部都支持。

  此外，支持交叉编译并非简单的工作，由于需要测试的组合众多，某些软件包可能无法构建。

  [关于 Nix 中交叉编译实现的详细说明](https://nixos.org/manual/nixpkgs/stable/#chap-cross)可以帮助修复这些问题。

- Nix 社区有一个[专门的 Matrix 房间](https://matrix.to/#/#cross-compiling:nixos.org)可提供交叉编译方面的帮助。
