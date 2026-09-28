---
myst:
  html_meta:
    "description lang=en": "Installing NixOS on a Raspberry Pi"
    "keywords": "Raspberry Pi, rpi, NixOS, installation, image, tutorial"
---


# 在树莓派上安装 NixOS

```{contributors}
:authors: domenkozar
:editors: proofconstruction
```

本教程假定你使用的是 [树莓派 4 Model B（4GB RAM）](https://www.raspberrypi.org/products/raspberry-pi-4-model-b/)。

开始本教程之前，请确认你已备齐
[全部所需硬件](https://projects.raspberrypi.org/en/projects/raspberry-pi-setting-up/1)：

- HDMI 线缆/转接器。
- 8GB 及以上容量的 SD 卡。
- SD 读卡器（若你的机器没有 SD 卡槽）。
- 树莓派电源线。
- USB 键盘。

:::{note}
本教程针对树莓派 4B 编写。使用此前受支持的型号（如 3B 或 3B+）也可行，但需要对本教程做一些修改。
:::

## 从 NixOS 实况镜像启动

:::{note}
从 USB 启动可能需要升级 EEPROM 固件。本教程从 SD 卡启动，以避免此类问题。
:::

要在另一台已安装 Nix 的设备上准备 AArch64 镜像，请运行以下命令：

```shell-session
$ nix-shell -p wget zstd

[nix-shell:~]$ wget https://hydra.nixos.org/build/226381178/download/1/nixos-sd-image-23.11pre500597.0fbe93c5a7c-aarch64-linux.img.zst
[nix-shell:~]$ unzstd -d nixos-sd-image-23.11pre500597.0fbe93c5a7c-aarch64-linux.img.zst
[nix-shell:~]$ dmesg --follow
```

:::{note}
你可以从 [Hydra](https://hydra.nixos.org/job/nixos/trunk-combined/nixos.sd_image.aarch64-linux) 下载较新的镜像：
点击最近一次成功构建（标有绿色对勾），然后复制构建产物镜像的链接。
:::

:::{note}
如果你所在的系统上可以使用 [Etcher](https://www.balena.io/etcher/) 这类软件，用它将镜像写入 SD 卡可能更方便。
:::

终端应会持续打印内核消息。

插入 SD 卡后，终端应会打印它被分配到的设备名，例如 `/dev/sdX`。

按 <kbd>Ctrl</kbd>+<kbd>C</kbd> 停止 `dmesg --follow`。

将 NixOS 复制到 SD 卡：在以下命令中把 `sdX` 替换为你的设备名：

```console
[nix-shell:~]$ sudo dd if=nixos-sd-image-23.11pre500597.0fbe93c5a7c-aarch64-linux.img of=/dev/sdX bs=4096 conv=fsync status=progress
```

该命令结束后，**将 SD 卡插入树莓派并通电开机**。

你应该会看到一个全新的 shell。

如果镜像无法启动，值得先[更新固件](https://www.raspberrypi.com/documentation/computers/raspberry-pi.html#bootloader_update_stable)，然后再试一次启动该镜像。

## 接入互联网

运行 `sudo -i` 以获取 root shell，供本教程剩余步骤使用。

此时你需要联网。如果可以使用以太网线，接上后跳到下一节。

如果要连接 Wi‑Fi，运行 `ip link` 查找无线网卡接口名。寻找以 `wl` 开头的接口（常见为 `wlan0`）。

将 `SSID` 和 `passphrase` 替换为你的 Wi‑Fi 凭据并运行：
```shell-session
# wpa_supplicant -B -i wlan0 -c <(wpa_passphrase 'SSID' 'passphrase') &
```

等待 `wpa_supplicant` 建立连接。然后为网卡分配静态 IP 以启用互联网访问。将 `192.168.1.X` 替换为你网络中可用的 IP 地址，将 `192.168.1.1` 替换为路由器的网关地址：
```shell-session
# ip addr add 192.168.1.X/24 dev wlan0
# ip route add default via 192.168.1.1
```

现在可以下载并运行 `dhcpcd` 以获取正式的 IP 地址：
```shell-session
# nix-shell -p dhcpcd
# dhcpcd wlan0
```

几秒后，通过运行以下命令验证连接：
```shell-session
# host nixos.org
```

如果 DNS 解析成功，说明已能上网，可以继续下一节。

如果打错了字，运行 `pkill wpa_supplicant` 后重来。

## 更新固件

为获取厂商的更新与缺陷修复，我们先更新树莓派固件：

```shell-session
# nix-shell -p raspberrypi-eeprom
# mount /dev/disk/by-label/FIRMWARE /mnt
# BOOTFS=/mnt FIRMWARE_RELEASE_STATUS=stable rpi-eeprom-update -d -a
```

## 安装并配置 NixOS

接下来用我们自己的配置安装 NixOS，这里创建 `guest` 用户并启用 SSH 守护进程。

在下面的 `let` 绑定中，将 `SSID` 和 `SSIDpassword` 的值改为你之前使用的 `SSID` 和 `passphrase`：

```nix
{ config, pkgs, lib, ... }:

let
  user = "guest";
  password = "guest";
  SSID = "mywifi";
  SSIDpassword = "mypassword";
  interface = "wlan0";
  hostname = "myhostname";
in {

  boot = {
    kernelPackages = pkgs.linuxKernel.packages.linux_rpi4;
    initrd.availableKernelModules = [ "xhci_pci" "usbhid" "usb_storage" ];
    loader = {
      grub.enable = false;
      generic-extlinux-compatible.enable = true;
    };
  };

  fileSystems = {
    "/" = {
      device = "/dev/disk/by-label/NIXOS_SD";
      fsType = "ext4";
      options = [ "noatime" ];
    };
  };

  networking = {
    hostName = hostname;
    wireless = {
      enable = true;
      networks."${SSID}".psk = SSIDpassword;
      interfaces = [ interface ];
    };
  };

  environment.systemPackages = with pkgs; [ vim ];

  services.openssh.enable = true;

  users = {
    mutableUsers = false;
    users."${user}" = {
      isNormalUser = true;
      password = password;
      extraGroups = [ "wheel" ];
    };
  };

  hardware.enableRedistributableFirmware = true;
  system.stateVersion = "23.11";
}
```

为节省输入整份配置的时间，可以下载它：

```shell-session
# curl -L https://tinyurl.com/tutorial-nixos-install-rpi4 > /etc/nixos/configuration.nix
```

:::{note}
写入 NixOS 配置中的凭据，会在该配置构建时以明文形式存储在 `/nix/store` 中。

如果你**不**希望发生这种情况，可以在控制台手动输入凭据，或使用社区提供的加密密钥管理方案之一。
:::

由于 `nixos-sd-image` 的设计方式，此时 NixOS 实际上*已经安装好了*，因此我们只需用新配置执行 `nixos-rebuild`：

```shell-session
# nixos-rebuild boot
# reboot
```

如果系统未能启动，在启动菜单中选择最旧的配置以回到实况镜像，然后重新开始。

## 进行更改

系统已成功启动，恭喜。

要进一步修改配置，可[搜索 NixOS 选项](https://search.nixos.org/options)，
编辑 `/etc/nixos/configuration.nix`，然后更新系统：

```shell-session
$ sudo -i
# nixos-rebuild switch
```

## 下一步

- 系统可用后，可尝试用 `nixos-rebuild switch --upgrade` 升级，以安装更新的软件包版本；若有问题，再重启回滚到旧配置。
- 要启用硬件加速以获得更好的图形桌面体验，将 [`nixos-hardware`](https://github.com/nixos/nixos-hardware) 模块加入配置：

  ```nix
  imports = [
    "${fetchTarball "https://github.com/NixOS/nixos-hardware/tarball/master"}/raspberry-pi/4"
  ];
  ```

  我们建议固定对 `nixos-hardware` 的引用：[](ref-pinning-nixpkgs)

- 要调整影响硬件的引导加载项选项，[参见 `config.txt` 选项](https://www.raspberrypi.org/documentation/configuration/config-txt/)。可通过运行 `mount /dev/disk/by-label/FIRMWARE /mnt` 并打开 `/mnt/config.txt` 来修改这些选项。
