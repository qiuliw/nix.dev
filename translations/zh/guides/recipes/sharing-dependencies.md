(sharing-dependencies)=
# 开发 shell 中的依赖

在 [`default.nix` 中打包软件](packaging-tutorial)时，你会希望有一个 [`shell.nix` 中的开发环境](declarative-reproducible-envs)，以便用 `nix-shell` 方便地进入，或[用 `direnv` 自动进入](./direnv)。

如何将 `default.nix` 中软件包的依赖与 `shell.nix` 中的开发环境共享？

## 摘要

使用 [`pkgs.mkShellNoCC` 的 `inputsFrom` 属性](https://nixos.org/manual/nixpkgs/stable/#sec-pkgs-mkShell-attributes)：

```nix
# default.nix
let
  pkgs = import <nixpkgs> {};
  myPackage = pkgs.callPackage ./package.nix {};
in
{
  inherit myPackage;
  shell = pkgs.mkShellNoCC {
    inputsFrom = [ myPackage ];
  };
}
```

在 `shell.nix` 中导入 `shell` 属性：

```nix
# shell.nix
(import ./.).shell
```

## 完整示例

假设你的软件包定义在 `package.nix` 中：

```nix
# package.nix
{ cowsay, runCommand }:
runCommand "cowsay-output" { buildInputs = [ cowsay ]; } ''
  cowsay Hello, Nix! > $out
''
```

在此示例中，`cowsay` 通过 `buildInputs` 声明为构建时依赖。

再假设你的项目定义在 `default.nix` 中：

```nix
# default.nix
let
  nixpkgs = fetchTarball "https://github.com/NixOS/nixpkgs/tarball/nixos-23.11";
  pkgs = import nixpkgs { config = {}; overlays = []; };
in
{
  myPackage = pkgs.callPackage ./package.nix {};
}
```

向 `default.nix` 添加一个指定环境的属性：


```diff
 let
   nixpkgs = fetchTarball "https://github.com/NixOS/nixpkgs/tarball/nixos-23.11";
   pkgs = import nixpkgs { config = {}; overlays = []; };
 in
 {
   myPackage = pkgs.callPackage ./package.nix {};
+  shell = pkgs.mkShellNoCC {
+  };
 }
```

将 `myPackage` 属性移入 `let` 绑定以便复用。
然后用 [`inputsFrom`](https://nixos.org/manual/nixpkgs/stable/#sec-pkgs-mkShell-attributes) 将该软件包的依赖纳入环境：

```diff
 let
   nixpkgs = fetchTarball "https://github.com/NixOS/nixpkgs/tarball/nixos-23.11";
   pkgs = import nixpkgs { config = {}; overlays = []; };
+  myPackage = pkgs.callPackage ./package.nix {};
 in
 {
-  myPackage = pkgs.callPackage ./package.nix {};
+  inherit myPackage;
   shell = pkgs.mkShellNoCC {
+    inputsFrom = [ myPackage ];
   };
 }
```

最后，在 `shell.nix` 中导入 `shell` 属性：

```nix
# shell.nix
(import ./.).shell
```

检查开发环境，其中包含构建时依赖 `cowsay`：

```console
$ nix-shell --pure
[nix-shell]$ cowsay shell.nix
```

## 下一步

- [](pinning-nixpkgs)
- [](./direnv)
- [](python-dev-environment)
- [](packaging-tutorial)
