(bootable-iso-image)=
# 构建可引导的 ISO 镜像

```{contributors}
:authors: domenkozar
```

:::{note}
如果需要为不同平台构建镜像，请参阅 [Cross compiling](https://github.com/nix-community/nixos-generators#user-content-cross-compiling)。
:::

你可能会发现官方安装镜像缺少某些硬件支持。

解决办法是创建 `myimage.nix`，让其基于最小安装 ISO 并指向最新内核：

```nix
{ pkgs, modulesPath, lib, ... }: {
  imports = [
    "${modulesPath}/installer/cd-dvd/installation-cd-minimal.nix"
  ];

  # use the latest Linux kernel
  boot.kernelPackages = pkgs.linuxPackages_latest;

  # Needed for https://github.com/NixOS/nixpkgs/issues/58959
  boot.supportedFilesystems = lib.mkForce [ "btrfs" "reiserfs" "vfat" "f2fs" "xfs" "ntfs" "cifs" ];
}
```

使用上述配置生成 ISO：

```shell-session
$ NIX_PATH=nixpkgs=https://github.com/NixOS/nixpkgs/archive/74e2faf5965a12e8fa5cff799b1b19c6cd26b0e3.tar.gz nix-shell -p nixos-generators --run "nixos-generate --format iso --configuration ./myimage.nix -o result"
```

将新镜像复制到 U 盘，把 `sdX` 替换为你的设备名：

```shell-session
$ dd if=result/iso/*.iso of=/dev/sdX status=progress
$ sync
```

## 下一步

- 查看此 [生成器支持的格式列表](https://github.com/nix-community/nixos-generators#user-content-supported-formats)，以找到你的云提供商或虚拟化技术。
- 查看 [创建 NixOS live CD 的替代指南](https://wiki.nixos.org/wiki/Creating_a_NixOS_live_CD)
