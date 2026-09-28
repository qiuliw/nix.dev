# 常见问题

## Nix

### 如何自动格式化 Nix 语言代码？

[`nixfmt`](https://github.com/NixOS/nixfmt) 是 {term}`Nix language` 代码的官方格式化工具。
请参阅其源码仓库获取安装说明。

`nixfmt` 被[用于格式化](https://github.com/NixOS/nixpkgs/blob/master/ci/default.nix) {term}`Nixpkgs` 中的所有代码。

### 如何在 Nix 语言中在路径与字符串之间转换？

参见 Nix 参考手册中的[字符串插值](https://nix.dev/manual/nix/2.19/language/string-interpolation)以及[作用于路径与字符串的运算符](https://nix.dev/manual/nix/2.19/language/operators#string-concatenation)。

### 如何构建某个软件包的反向依赖？

```shell-session
$ nix-shell -p nixpkgs-review --run "nixpkgs-review wip"
```

### 如何用 Nix 管理 \$HOME 中的点文件（dotfiles）？

参见 <https://github.com/nix-community/home-manager>

### 构建自定义软件包的推荐流程是什么？

请阅读 [](packaging-tutorial)。

### 如何使用 Nixpkgs 仓库的克隆来更新或编写新软件包？

请阅读 [](packaging-tutorial) 以及 [Nixpkgs 贡献指南](https://github.com/NixOS/nixpkgs/blob/master/CONTRIBUTING.md)。

## NixOS

### 如何运行非 Nix 可执行文件？

NixOS 开箱即无法运行面向通用 Linux 环境的动态链接可执行文件。
这是因为，按设计，它没有全局库路径，也不遵循[文件系统层次标准](https://refspecs.linuxfoundation.org/FHS_3.0/fhs/index.html)（FHS）。

有几种方式可以解决这种环境期望上的不匹配：

- 若 {term}`Nixpkgs` 中已有对应软件包，请使用该版本。
  可在 <https://search.nixos.org/packages> 搜索可用软件包。

- 为该程序编写 Nix 表达式，在你自己的配置中打包。

  有多种做法：
  - 从源码构建。

    许多开源程序在编译时对文件放置位置高度灵活。
    入门介绍见 [](packaging-tutorial)。
  - 使用 [`autoPatchelfHook`](https://nixos.org/manual/nixpkgs/stable/#setup-hook-autopatchelfhook) 修改程序的 [ELF 头](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format)，以包含库路径。

    在无法从源码构建时采用此方法。
  - 使用 [`buildFHSEnv`](https://nixos.org/manual/nixpkgs/stable/#sec-fhs-environments) 将程序包装在类 FHS 环境中运行。

    这是最后手段，但有时仍有必要，例如程序会下载并运行其他可执行文件时。

- 使用 [`nix-ld`](https://github.com/Mic92/nix-ld) 创建一个仅适用于未打包程序的库路径。
  将其添加到你的 `configuration.nix`：

  ```nix
    programs.nix-ld.enable = true;
    programs.nix-ld.libraries = with pkgs; [
      # Add any missing dynamic libraries for unpackaged programs
      # here, NOT in environment.systemPackages
    ];
  ```

  然后运行 `nixos-rebuild switch`，并注销再重新登录，以使新的环境变量生效。
  （仅在启用 `nix-ld` 时需要这样做；所含库的变更在重建后立即生效。）

  :::{note}
  `nix-ld` 在 `x86_64` 机器上对 32 位可执行文件无效。
  :::

- 使用 [`steam-run`](https://nixos.org/manual/nixpkgs/stable/#sec-steam-run)，在为 Steam 软件包准备的类 FHS 环境中运行你的程序：

  ```shell-session
  $ nix-shell -p steam-run --run "steam-run <command>"
  ```

### 如何构建自己的 ISO？

参见 <http://nixos.org/nixos/manual/index.html#sec-building-image>

### 如何连接到 NixOS 测试中的任意机器？

应用以下补丁：

```diff
diff --git a/nixos/lib/test-driver/test-driver.pl b/nixos/lib/test-driver/test-driver.pl
index 8ad0d67..838fbdd 100644
--- a/nixos/lib/test-driver/test-driver.pl
+++ b/nixos/lib/test-driver/test-driver.pl
@@ -34,7 +34,7 @@ foreach my $vlan (split / /, $ENV{VLANS} || "") {
     if ($pid == 0) {
         dup2(fileno($pty->slave), 0);
         dup2(fileno($stdoutW), 1);
-        exec "vde_switch -s $socket" or _exit(1);
+        exec "vde_switch -tap tap0 -s $socket" or _exit(1);
     }
     close $stdoutW;
     print $pty "version\n";
```

然后 vde_switch 网络应可在本地访问。

### 如何在已有 Linux 安装中引导安装 NixOS？

有若干工具可用：

- <https://github.com/nix-community/nixos-anywhere>
- <https://github.com/jeaye/nixos-in-place>
- <https://github.com/elitak/nixos-infect>
- <https://github.com/cleverca22/nix-tests/tree/master/kexec>
