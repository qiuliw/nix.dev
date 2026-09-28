---
myst:
  html_meta:
    "description lang=en": "Provisioning remote machines"
    "keywords": "Nix, deployment, remote, provisioning, nixos-anywhere, disko, partitioning, installation"
---

(provisioning-remote-machines-tutorial)=
# 通过 SSH 配置远程机器

```{contributors}
:authors: tfc
:editors: fricklerhandwerk
```

可以使用 [`nixos-anywhere`] 和 [`disko`]，在正在运行的系统上将任意 Linux 安装替换为 NixOS 配置。

[`nixos-anywhere`]: https://nix-community.github.io/nixos-anywhere/
[`disko`]: https://github.com/nix-community/disko

## 简介

在本教程中，你将把一份 NixOS 配置部署到一台正在运行的计算机上。

### 你将学到什么？

你将学习如何
- 用声明式磁盘布局与 SSH 访问指定一份最小 NixOS 配置
- 检查配置是否有效
- 在远程机器上部署并更新 NixOS 配置

### 你需要什么？

- 熟悉 [Nix 语言](reading-nix-language)
- 熟悉 [](module-system-tutorial)

为成功进行无人值守安装，请确保*目标机器*满足：

- 它是一台运行 Linux 的 QEMU 虚拟机
  - 支持 [`kexec`](https://en.wikipedia.org/wiki/Kexec)
  - 指令集架构（ISA）为 `x86-64` 或 `aarch64`
  - 至少有 1 GB RAM

  也可以是从 USB 启动的实况系统，例如 [NixOS 安装程序](https://nixos.org/download/#download-nixos-accordion)。

- IP 地址通过 DHCP 自动配置
- 你可以通过 SSH 登录
  - 使用公钥认证（推荐）或密码
  - 以用户 `root` 或其他具有 `sudo` 权限的用户身份

*本地机器*只需要已可用的 [Nix 安装](install-nix)。

本教程中将*目标机器*称为 `target-machine`。
请将其替换为实际的主机名或 IP 地址。

## 准备环境

创建新的项目目录并用 shell 进入该目录：

```shell-session
mkdir remote
cd remote
```

[指定依赖](dependency-management-npins)：`nixpkgs`、`disko` 和 `nixos-anywhere`：

```shell-session
$ nix-shell -p npins
[nix-shell:remote]$ npins init
[nix-shell:remote]$ npins add github nix-community disko
[nix-shell:remote]$ npins add github nix-community nixos-anywhere
```

创建新文件 `shell.nix`，使用固定的依赖提供所需全部工具：

```{code-block} nix
let
  sources = import ./npins;
  pkgs = import sources.nixpkgs {};
in

pkgs.mkShell {
  nativeBuildInputs = with pkgs; [
    npins
    nixos-anywhere
    nixos-rebuild
  ];
  shellHook = ''
    export NIX_PATH="nixpkgs=${sources.nixpkgs}:nixos-config=$PWD/configuration.nix"
  '';
}
```

现在退出临时环境并进入新指定的环境：

```shell-session
[nix-shell:remote]$ exit
$ nix-shell
```

该 shell 环境已准备好，可使用定义明确的 Nixpkgs 版本以及 `nixos-anywhere` 和 `nixos-rebuild`。

:::{important}
请在此环境中运行后续所有命令。
:::

## 创建 NixOS 配置

新的 NixOS 配置将由通用系统配置与磁盘布局规范组成。

本示例中的磁盘布局描述单块磁盘，带有[主引导记录](https://en.wikipedia.org/wiki/Master_boot_record)（MBR）和 [EFI 系统分区](https://en.wikipedia.org/wiki/EFI_system_partition)（ESP），以及占用全部剩余空间的根文件系统。
它可在 EFI 与 BIOS 系统上同时工作。

创建新文件 `single-disk-layout.nix`，写入磁盘布局规范：

{lineno-start=1}
```nix
{ ... }:

{
  disko.devices.disk.main = {
    type = "disk";
    content = {
      type = "gpt";
      partitions = {
        MBR = {
          priority = 0;
          size = "1M";
          type = "EF02";
        };
        ESP = {
          priority = 1;
          size = "500M";
          type = "EF00";
          content = {
            type = "filesystem";
            format = "vfat";
            mountpoint = "/boot";
          };
        };
        root = {
          priority = 2;
          size = "100%";
          content = {
            type = "filesystem";
            format = "ext4";
            mountpoint = "/";
          };
        };
      };
    };
  };
}
```

创建文件 `configuration.nix`，导入磁盘布局定义并指定要格式化的磁盘：

:::{tip}
如果你不知道目标磁盘的设备标识符，可在*目标机器*上用 `lsblk` 列出全部设备：

```shell-session
$ ssh target-machine lsblk
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sda      8:0    0   256G  0 disk
├─sda1   8:1    0 248.5G  0 part /nix/store
│                                /
└─sda2   8:2    0   7.5G  0 part [SWAP]
sr0     11:0    1  1024M  0 rom
```

在本示例中，磁盘名为 `sda`。
块设备路径则为 `/dev/sda`。
请记下该值以供后用。
:::

{lineno-start=1}
```nix
{ modulesPath, ... }:

let
  diskDevice = "/dev/sda";
  sources = import ./npins;
in
{
  imports = [
    (modulesPath + "/profiles/qemu-guest.nix")
    (sources.disko + "/module.nix")
    ./single-disk-layout.nix
  ];

  disko.devices.disk.main.device = diskDevice;

  boot.loader.grub = {
    devices = [ diskDevice ];
    efiSupport = true;
    efiInstallAsRemovable = true;
  };

  services.openssh.enable = true;

  users.users.root.openssh.authorizedKeys.keys = [
    "<your SSH key here>"
  ];

  system.stateVersion = "24.11";
}
```

:::{important}
将 `/dev/sda` 替换为你的磁盘块设备路径。

将 `<your SSH key here>` 字符串替换为你希望用于日后以用户 `root` 登录的 SSH 公钥。
:::

:::{dropdown} 详细说明

`let` 块中的 `diskDevice` 变量定义磁盘块设备的路径：

{lineno-start=3 emphasize-lines="2"}
```nix
let
  diskDevice = "/dev/sda";
  sources = import ./npins;
in
```

它用于设置磁盘布局规范所描述的分区与格式化目标。
它也用于引导加载程序配置，使其在传统 BIOS 与 UEFI 系统上均可引导：

{lineno-start=14 emphasize-lines="1,4"}
```nix
  disko.devices.disk.main.device = diskDevice;

  boot.loader.grub = {
    devices = [ diskDevice ];
    efiSupport = true;
    efiInstallAsRemovable = true;
  };
```

`qemu-guest.nix` 模块使该系统兼容在 QEMU 虚拟机中运行：

{lineno-start=8 emphasize-lines="2"}
```nix
  imports = [
    (modulesPath + "/profiles/qemu-guest.nix")
    (sources.disko + "/module.nix")
    ./single-disk-layout.nix
  ];
```

根据磁盘布局规范，`disko` 库会生成分区脚本，以及在启动时相应地挂载分区的那部分 NixOS 配置。
第一行导入该库，第二行应用磁盘布局：

{lineno-start=8 emphasize-lines="3,4"}
```nix
  imports = [
    (modulesPath + "/profiles/qemu-guest.nix")
    (sources.disko + "/module.nix")
    ./single-disk-layout.nix
  ];
```
:::

## 测试磁盘布局

检查磁盘布局是否有效：

```shell-session
nix-build -E "((import <nixpkgs> {}).nixos [ ./configuration.nix ]).installTest"
```

该命令通过构建 `disko` 模块提供的 `installTest` 属性中的 derivation，在虚拟机中运行完整安装过程。

## 部署系统

要部署系统，构建配置及对应的磁盘格式化脚本，并使用结果运行 `nixos-anywhere`：

:::{important}
将 `target-host` 替换为你*目标机器*的主机名或 IP 地址。
:::

```shell-session
toplevel=$(nixos-rebuild build --no-flake)
diskoScript=$(nix-build -E "((import <nixpkgs> {}).nixos [ ./configuration.nix ]).diskoScript")
nixos-anywhere --store-paths "$diskoScript" "$toplevel" root@target-host
```

:::{note}
如果你没有使用公钥认证：
将环境变量 `SSH_PASS` 设为你的密码，然后在 `nixos-anywhere` 命令末尾追加 `--env-password` 标志。
:::

`nixos-anywhere` 现在会登录目标系统，分区、格式化并挂载磁盘，然后安装 NixOS 配置。
随后它会重启系统。

## 更新系统

要更新系统，运行 `npins` 并重新部署配置：

```shell-session
npins update nixpkgs
nixos-rebuild switch --no-flake --target-host root@target-host
```

不再需要 `nixos-anywhere`，除非你想更改磁盘布局。


# 下一步

- [](binary-cache-setup)
- [](post-build-hooks)

## 参考

- [`nixos-anywhere` 项目页面][`nixos-anywhere`]
- [`disko` 项目仓库][`disko`]
- [磁盘布局示例合集](https://github.com/nix-community/disko/tree/master/example)
