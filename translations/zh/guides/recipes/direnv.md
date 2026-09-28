(automatic-direnv)=
# 使用 `direnv` 自动激活环境

不必为每个项目手动激活环境，你可以在每次进入项目目录或其中的 `shell.nix` 发生变化时，重新加载[声明式 shell](declarative-reproducible-envs)。

1. [使 nix-direnv 可用](https://github.com/nix-community/nix-direnv)
2. [将其挂接到你的 shell](https://direnv.net/docs/hook.html)

例如，编写内容如下的 `shell.nix`：

```nix
let
  nixpkgs = fetchTarball "https://github.com/NixOS/nixpkgs/tarball/nixos-23.11";
  pkgs = import nixpkgs { config = {}; overlays = []; };
in

pkgs.mkShellNoCC {
  packages = with pkgs; [
    hello
  ];
}
```

在项目的顶级目录中运行：

```shell-session
$ echo "use nix" > .envrc && direnv allow
```

下次启动终端并进入项目的顶级目录时，`direnv` 会自动启动 `shell.nix` 中定义的 shell。

```shell-session
$ cd myproject
$ which hello
/nix/store/1gxz5nfzfnhyxjdyzi04r86sh61y4i00-hello-2.12.1/bin/hello
```

`direnv` 也会检查 `shell.nix` 文件的变更。

进行如下添加：

```diff
 let
   nixpkgs = fetchTarball "https://github.com/NixOS/nixpkgs/tarball/nixos-23.11";
   pkgs = import nixpkgs { config = {}; overlays = []; };
 in

 pkgs.mkShellNoCC {
   packages = with pkgs; [
     hello
   ];
+
+  shellHook = ''
+    hello
+  '';
 }
```

在第一次交互之后（运行任意命令或按 `Enter`），运行中的环境应会自行重新加载。

```shell-session
Hello, world!
```
