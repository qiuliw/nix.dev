(dependency-management-npins)=
# 使用 `npins` 自动管理远程源

Nix 语言描述 Nix 所管理文件之间的依赖关系。
Nix 表达式本身也可以依赖远程源，并且有多种方式指定其来源，如 [](pinning-nixpkgs) 所示。

若要围绕远程源处理获得更多自动化，请在项目中设置 [`npins`](https://github.com/andir/npins/)：

```shell-session
$ nix-shell -p npins --run "npins init --bare; npins add github nixos nixpkgs --branch nixos-23.11"
```

该命令会获取 Nixpkgs 23.11 发布分支的最新修订。
它会在当前目录生成 `npins/sources.json`，其中包含对所得修订的固定引用。
同时还会创建 `npins/default.nix`，将这些依赖作为属性集暴露出来。

将生成的 `npins/default.nix` 作为 `default.nix` 中函数参数的默认值导入，并用它引用 Nixpkgs 源目录：

```nix
{
  sources ? import ./npins,
  system ? builtins.currentSystem,
  pkgs ? import sources.nixpkgs { inherit system; config = {}; overlays = []; },
}:
{
  package = pkgs.hello;
}
```

`nix-build` 会以空属性集 `{}`，或通过 [`--arg`](https://nix.dev/manual/nix/stable/command-ref/nix-build#opt-arg) 或 [`--argstr`](https://nix.dev/manual/nix/stable/command-ref/nix-build#opt-argstr) 传入的属性，调用顶层函数。
此模式允许以编程方式[覆盖远程源](overriding-sources-npins)。

将 `npins` 加入项目的开发环境，以便随时可用：

```diff
 {
   sources ? import ./npins,
   system ? builtins.currentSystem,
   pkgs ? import sources.nixpkgs { inherit system; config = {}; overlays = []; },
 }:
-{
+rec {
   package = pkgs.hello;
+  shell = pkgs.mkShellNoCC {
+    inputsFrom = [ package ];
+    packages = with pkgs; [
+      npins
+    ];
+  };
 }
```

再添加一个 `shell.nix`，以便更方便地进入该环境：

```nix
(import ./. {}).shell
```

细节见 [](./sharing-dependencies)；注意此处必须向导入的表达式传入空属性集，因为 `default.nix` 现在包含一个函数。

(overriding-sources-npins)=
## 覆盖源

作为示例，我们将用较旧版本的 Nixpkgs 使用先前创建的表达式。

进入开发环境，创建新目录，并用不同版本的 Nixpkgs 设置 npins：

```shell-session
$ nix-shell
[nix-shell]$ mkdir old
[nix-shell]$ cd old
[nix-shell]$ npins init --bare
[nix-shell]$ npins add github nixos nixpkgs --branch nixos-21.11
```

在新目录中创建文件 `default.nix`，用刚创建的 `sources` 导入原先的表达式。

```nix
import ../default.nix { sources = import ./npins; }
```

这将构建出不同版本：

```shell-session
$ nix-build -A build
$ ./result/bin/hello --version | head -1
hello (GNU Hello) 2.10
```

也可以在命令行上覆盖源：

```shell-session
nix-build .. -A build --arg sources 'import ./npins'
```

## 从 `niv` 迁移

本指南的先前版本曾推荐使用 [`niv`](https://github.com/nmattia/niv/)，这是一个用 Haskell 编写的类似固定管理器。

如果你的项目使用 `niv`，可以将远程源定义导入到 `npins`：

```shell-session
npins import-niv
```

:::{warning}
所有导入的条目都会被更新，因此它们不一定会指向与之前相同的提交。
:::

## 管理 NixOS 配置

NixOS 默认使用 channel 来定位 `nixpkgs`。
你可以改为从 `system.nix` 入口点（自 26.05 起可用）用 `npins` 固定版本：

```nix
let
  sources = import ./npins;
in
import "${sources.nixpkgs}/nixos" {
  configuration = ./configuration.nix;
}
```

在 NixOS 26.05 的 `system.nix` 之前，使用：

```bash
sudo NIX_PATH="nixos-config=configuration.nix:nixpkgs=$(nix-instantiate --raw --eval npins -A nixpkgs.outPath)" nixos-rebuild switch
```

如果将 npins 与 `system.nix` 一起使用，请在配置中禁用 channel：

```nix
# configuration.nix
{
  # ...
  nix.channel.enable = false;
}
```

要使此类固定依赖在使用 NixOS 配置时可作为[查找路径](https://nix.dev/tutorials/nix-language.html#lookup-paths)（例如 `<nixpkgs>`）使用，可以采用：

```nix
# configuration.nix
{ lib, ... }:
let
  sources = import ./npins;
in
{
  # ...
  nix.nixPath = lib.mapAttrsToList (k: v: "${k}=${v}") sources;
}
```

要使用 [v3 命令行]并运行通过 [flake] 暴露软件包的依赖中的程序，
例如 `nix run nixpkgs#hello`，
你可以启用 flakes，并将固定项添加到 flake registry，例如：

```nix
# configuration.nix
{ lib, ... }:
let
  sources = import ./npins;
in
{
  # ...
  experimental-features = "nix-command flakes";
  nix.registry = lib.mapAttrs (_: path: {
    to = {
      type = "path";
      inherit path;
    };
  }) sources;
```

[v3 命令行]: https://nix.dev/manual/nix/stable/command-ref/new-cli/nix.html
[flake]: https://nix.dev/concepts/flakes

## 下一步

- 查看内置帮助以获取更多信息：

  ```shell-session
  npins --help
  ```

- 关于指定远程源的不同方式的更多细节与示例，见 [](pinning-nixpkgs)。
