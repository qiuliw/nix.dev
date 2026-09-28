(nixos-vms)=

# NixOS 虚拟机

```{contributors}
:authors: olafklingt, domenkozar
:editors: fricklerhandwerk
```

NixOS 最重要的特性之一，是能够以声明方式配置整个系统，包括要安装的软件包、要运行的服务，以及其他设置与选项。

NixOS 配置可用于通过虚拟机测试和使用 NixOS，相比完整的“裸机”安装，这是更轻量的选择。

## 你将学到什么？

本教程介绍如何创建 NixOS 虚拟机。
虚拟机是实验或调试 NixOS 配置的实用工具。

## 你需要什么？

- 一台支持虚拟化的 Linux 系统
- （可选）用于运行图形虚拟机的图形环境
- 可用的 [Nix 安装](https://nix.dev/install-nix)
- [Nix 语言](reading-nix-language)的基础知识

:::{important}
NixOS 配置是遵循 [NixOS 模块](https://nixos.org/manual/nixos/stable/index.html#sec-writing-modules)约定的 Nix 语言函数。
要深入了解模块系统，请参阅 [](module-system-deep-dive) 教程。
:::

## 从默认 NixOS 配置开始

:::{note}
本教程从第一性原理逐步构建你的 `configuration.nix`，并解释每一步。
如果你愿意，也可以直接跳到[示例配置](sample-nixos-config)一节。
:::

我们从一份最小的 `configuration.nix` 开始：

```nix
{ config, pkgs, ... }:

{
  boot.loader.systemd-boot.enable = true;
  boot.loader.efi.canTouchEfiVariables = true;

  system.stateVersion = "24.05";
}
```

为了能够登录，向返回的属性集中添加以下行：

```nix
  users.users.alice = {
    isNormalUser = true;
    extraGroups = [ "wheel" ];
  };
```

此外，你还需要为该用户指定密码。
仅为演示目的，通过向用户配置添加 `initialPassword` 选项来指定一个不安全的明文密码：

```nix
   initialPassword = "test";
```

我们添加两个轻量程序作为示例：

```nix
  environment.systemPackages = with pkgs; [
    cowsay
    lolcat
  ];
```

:::{warning}
除非你清楚自己在做什么，否则不要在本示例之外使用明文密码。更安全的替代方案见 [`initialHashedPassword`](https://nixos.org/manual/nixos/stable/options.html#opt-users.extraUsers._name_.initialHashedPassword) 或 [`ssh.authorizedKeys`](https://nixos.org/manual/nixos/stable/options.html#opt-users.extraUsers._name_.openssh.authorizedKeys.keys)。
:::

(sample-nixos-config)=
### 示例配置

完整的 `configuration.nix` 文件如下所示：

```nix
{ config, pkgs, ... }:
{
  boot.loader.systemd-boot.enable = true;
  boot.loader.efi.canTouchEfiVariables = true;

  users.users.alice = {
    isNormalUser = true;
    extraGroups = [ "wheel" ]; # Enable ‘sudo’ for the user.
    initialPassword = "test";
  };

  environment.systemPackages = with pkgs; [
    cowsay
    lolcat
  ];

  system.stateVersion = "24.05";
}
```

## 从 NixOS 配置创建基于 QEMU 的虚拟机

使用 `nix-build` 命令创建 NixOS 虚拟机：

```shell-session
$ nix-build '<nixpkgs/nixos>' -A vm -I nixpkgs=channel:nixos-24.05 -I nixos-config=./configuration.nix
```

该命令从 NixOS 的 `nixos-24.05` 发行版构建属性 `vm`，并使用相对路径中指定的 NixOS 配置。

::::{dropdown} 详细说明

- [`nix-build`](https://nix.dev/manual/nix/stable/command-ref/nix-build.html) 的位置参数是要构建的 derivation 的路径。
  该路径可从[求值为 derivation 的 Nix 表达式](derivations)获得。

  虚拟机构建辅助定义在 NixOS 中，而 NixOS 是 [`nixpkgs` 仓库](https://github.com/NixOS/nixpkgs)的一部分。
  因此我们使用[查找路径](lookup-path-tutorial) `<nixpkgs/nixos>`。

- [`-A` 选项](https://nix.dev/manual/nix/stable/command-ref/opt-common.html#opt-attr)指定要从所提供的 Nix 表达式 `<nixpkgs/nixos>` 中选取的属性。

  要构建虚拟机，我们选择在 [`nixos/default.nix`](https://github.com/NixOS/nixpkgs/blob/7c164f4bea71d74d98780ab7be4f9105630a2eba/nixos/default.nix#L19) 中定义的 `vm` 属性。

- [`-I` 选项](https://nix.dev/manual/nix/stable/command-ref/opt-common.html#opt-I)会向搜索路径前置条目。

  这里我们将 `nixpkgs` 设为指向[特定版本的 Nixpkgs](ref-pinning-nixpkgs)，并将 `nix-config` 设为当前目录中的 `configuration.nix` 文件。
::::

## 运行虚拟机

上一条命令在工作目录中创建了名为 `result` 的链接。
它指向包含该虚拟机的目录。

```shell-session
$ ls -R ./result
result:
bin  system

result/bin:
run-nixos-vm
```

运行虚拟机：

```shell-session
$ QEMU_KERNEL_PARAMS=console=ttyS0 ./result/bin/run-nixos-vm -nographic; reset
```

由于使用了 `-nographic`，该命令会在当前终端中运行 QEMU。
`console=ttyS0` 还会显示引导过程，最终停在控制台登录界面。

以用户 `alice`、密码 `test` 登录。
检查程序是否确实按配置可用：

```shell-session
$ cowsay hello | lolcat
```

通过关机退出虚拟机：

```shell-session
$ sudo poweroff
```

:::{note}
如果你忘记将用户加入 `wheel`，或没有设置密码，可从另一个终端停止虚拟机：

```shell-session
$ sudo pkill qemu
```
:::

运行虚拟机会在当前目录创建 `nixos.qcow2` 文件。
该磁盘镜像文件包含虚拟机的动态状态。
它可能干扰调试，因为它会保留先前运行的状态，例如用户密码。

更改配置时请删除该文件：

```shell-session
$ rm nixos.qcow2
```

## 在图形 VM 上运行 GNOME

要创建带图形用户界面的虚拟机，向配置添加以下行：

```nix
  # Enable the X11 windowing system.
  services.xserver.enable = true;

  # Enable the GNOME Desktop Environment.
  services.xserver.displayManager.gdm.enable = true;
  services.xserver.desktopManager.gnome.enable = true;
```

这三行分别启用 X11、GDM 显示管理器（以便登录）以及 Gnome 桌面管理器。

:::{tip}

你也可以使用 `installation-cd-graphical-gnome.nix` 模块从头生成配置文件：

```shell-session
nix-shell -I nixpkgs=channel:nixos-24.05 -p "$(cat <<EOF
  let
    pkgs = import <nixpkgs> { config = {}; overlays = []; };
    iso-config = pkgs.path + /nixos/modules/installer/cd-dvd/installation-cd-graphical-gnome.nix;
    nixos = pkgs.nixos iso-config;
  in nixos.config.system.build.nixos-generate-config
EOF
)"
```

```shell-session
$ nixos-generate-config --dir ./
```

::::

完整的 `configuration.nix` 文件如下所示：

```nix
{ config, pkgs, ... }:
{
  boot.loader.systemd-boot.enable = true;
  boot.loader.efi.canTouchEfiVariables = true;

  services.xserver.enable = true;

  services.xserver.displayManager.gdm.enable = true;
  services.xserver.desktopManager.gnome.enable = true;

  users.users.alice = {
    isNormalUser = true;
    extraGroups = [ "wheel" ];
    initialPassword = "test";
  };

  system.stateVersion = "24.05";
}
```

要获得图形输出，不带特殊选项运行虚拟机：

```shell-session
$ nix-build '<nixpkgs/nixos>' -A vm -I nixpkgs=channel:nixos-24.05 -I nixos-config=./configuration.nix
$ ./result/bin/run-nixos-vm
```

## 在 VM 上以 Sway 作为 Wayland 合成器运行

要切换到 Wayland 合成器，禁用 `services.xserver.desktopManager.gnome` 并启用 `programs.sway`：

```{code-block} diff
:caption: configuration.nix
-  services.xserver.desktopManager.gnome.enable = true;
+  programs.sway.enable = true;
```

:::{note}
在虚拟机中运行 Wayland 合成器可能因 QEMU 使用的显示驱动而出现问题。
你需要从可用驱动中选择与 Sway 兼容的一种。
选项见 [QEMU 用户文档](https://www.qemu.org/docs/master/system/qemu-manpage.html)。
一种可能是 `virtio-vga` 驱动：

```shell-session
$ ./result/bin/run-nixos-vm -device virtio-vga
```

也可以在配置文件中添加传给 QEMU 的参数：

```nix
{ config, pkgs, ... }:
{
  boot.loader.systemd-boot.enable = true;
  boot.loader.efi.canTouchEfiVariables = true;

  services.xserver.enable = true;

  services.xserver.displayManager.gdm.enable = true;
  programs.sway.enable = true;

  imports = [ <nixpkgs/nixos/modules/virtualisation/qemu-vm.nix> ];
  virtualisation.qemu.options = [
    "-device virtio-vga"
  ];

  users.users.alice = {
    isNormalUser = true;
    extraGroups = [ "wheel" ];
    initialPassword = "test";
  };

  system.stateVersion = "24.05";
}
```

:::

NixOS 手册中有关于 [X11](https://nixos.org/manual/nixos/stable/#sec-x11) 和 [Wayland](https://nixos.org/manual/nixos/stable/#sec-wayland) 的章节，列出了其他窗口管理器。

## 参考

- [NixOS 手册：NixOS 配置](https://nixos.org/manual/nixos/stable/index.html#ch-configuration)。
- [NixOS 手册：模块](https://nixos.org/manual/nixos/stable/index.html#sec-writing-modules)。
- [NixOS 手册选项参考](https://nixos.org/manual/nixos/stable/options.html)。
- [NixOS 手册：更改配置](https://nixos.org/manual/nixos/stable/#sec-changing-config)。
- [NixOS 源码：`tools.nix` 中的 `configuration template`](https://github.com/NixOS/nixpkgs/blob/4e0525a8cdb370d31c1e1ba2641ad2a91fded57d/nixos/modules/installer/tools/tools.nix#L122-L226)。
- [NixOS 源码：`default.nix` 中的 `vm` 属性](https://github.com/NixOS/nixpkgs/blob/master/nixos/default.nix)。
- [Nix 手册：`nix-build`](https://nix.dev/manual/nix/stable/command-ref/nix-build.html)。
- [Nix 手册：通用命令行选项](https://nix.dev/manual/nix/stable/command-ref/opt-common.html)。
- [QEMU 用户文档](https://www.qemu.org/docs/master/system/qemu-manpage.html)，了解更多运行时选项
- [NixOS 选项搜索：`virtualisation.qemu`](https://search.nixos.org/options?query=virtualisation.qemu)，用于声明式虚拟机配置

## 下一步

- [](module-system-deep-dive)
- [](integration-testing-vms)
- [](bootable-iso-image)
