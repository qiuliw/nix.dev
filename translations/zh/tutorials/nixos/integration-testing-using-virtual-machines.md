(integration-testing-vms)=

# 使用 NixOS 虚拟机进行集成测试

```{contributors}
:authors: olafklingt, domenkozar
:editors: fricklerhandwerk
```

## 你将学到什么？

本教程介绍 Nixpkgs 中用于测试 NixOS 配置的功能。
它还展示如何设置涉及多台机器的分布式测试场景。

## 你需要什么？

- Linux 上可用的 [Nix 安装](<install-nix>)，或 [NixOS](https://nixos.org/manual/nixos/stable/index.html#sec-installation)
- [Nix 语言](<reading-nix-language>)的基础知识
- [NixOS 配置](<nixos-vms>)的基础知识

## 简介

Nixpkgs 提供一个[测试环境](https://nixos.org/manual/nixos/stable/index.html#sec-nixos-tests)，用于自动化分布式系统的集成测试。
它允许基于一组声明式 NixOS 配置定义测试，并使用 Python shell，以 [QEMU](https://www.qemu.org/) 为后端与这些配置交互。
这类测试被广泛用于确保 NixOS 按预期工作，因此通常称为 [NixOS 测试](https://nixos.org/manual/nixos/stable/index.html#sec-nixos-tests)。
它们可以在 NixOS 之外编写并启动，适用于任何 Linux 机器[^darwin]。

[^darwin]: [在 macOS 上运行 NixOS VM 测试](https://github.com/NixOS/nixpkgs/issues/108984)的支持也已实现，但[目前尚无文档](https://github.com/NixOS/nixpkgs/issues/254552)。

Nix 的设计特性使集成测试可复现，这使它们在持续集成（CI）流水线中很有价值。

## `testers.runNixOSTest` 函数

NixOS VM 测试使用 `testers.runNixOSTest` 函数定义。
NixOS VM 测试的模式如下：

```nix
let
  nixpkgs = fetchTarball "https://github.com/NixOS/nixpkgs/tarball/nixos-23.11";
  pkgs = import nixpkgs { config = {}; overlays = []; };
in

pkgs.testers.runNixOSTest {
  name = "test-name";
  nodes = {
    machine1 = { config, pkgs, ... }: {
      # ...
    };
    machine2 = { config, pkgs, ... }: {
      # ...
    };
  };
  testScript = { nodes, ... }: ''
    # ...
  '';
}
```

函数 `testers.runNixOSTest` 接受一个[模块](https://nixos.org/manual/nixos/stable/#sec-writing-modules)以指定[测试选项](https://nixos.org/manual/nixos/stable/index.html#sec-test-options-reference)。
由于该模块只设置配置值，可以使用缩写的模块记法。

必须设置以下配置值：

- [`name`](https://nixos.org/manual/nixos/stable/index.html#test-opt-name) 定义测试的名称。

- [`nodes`](https://nixos.org/manual/nixos/stable/index.html#test-opt-nodes) 包含一组命名的配置，因为测试脚本可能涉及不止一台虚拟机。
  每台虚拟机都由一份 NixOS 配置创建。

- [`testScript`](https://nixos.org/manual/nixos/stable/index.html#test-opt-testScript) 定义 Python 测试脚本，可以是字面字符串，也可以是接受 `nodes` 属性的函数。
  该 Python 测试脚本可通过 `nodes` 中使用的名称访问虚拟机。
  它在虚拟机中拥有超级用户权限。
  在 Python 脚本中，每台虚拟机都可通过 `machine` 对象访问。
  NixOS 提供在这些配置上运行测试的[方法](https://nixos.org/manual/nixos/stable/index.html#ssec-machine-objects)。

测试框架会自动启动虚拟机并运行 Python 脚本。

## 最小示例

作为对默认配置的最小测试，我们将检查用户 `root` 和 `alice` 是否都能运行 Firefox。
我们将从头构建该示例。

1. 使用[固定版本的 Nixpkgs](ref-pinning-nixpkgs)，并[显式设置配置选项与 overlays](nixpkgs-config)，以避免它们被全局配置无意覆盖：

   ```nix
   let
     nixpkgs = fetchTarball "https://github.com/NixOS/nixpkgs/tarball/nixos-23.11";
     pkgs = import nixpkgs { config = {}; overlays = []; };
   in

   pkgs.testers.runNixOSTest {
     # ...
   }
   ```

1. 为测试贴上描述性名称：

   ```nix
   name = "minimal-test";
   ```

1. 由于本示例只使用一台虚拟机，我们将指定的节点简单地称为 `machine`。
   该名称是任意的，可以自由选择。
   作为配置，你使用默认配置中的相关部分，[我们在上一教程中用过](<nixos-vms>)：

   ```nix
   nodes.machine = { config, pkgs, ... }: {
     users.users.alice = {
       isNormalUser = true;
       extraGroups = [ "wheel" ];
       packages = with pkgs; [
         firefox
         tree
       ];
     };

     system.stateVersion = "23.11";
   };
   ```

1. 这是测试脚本：

   ```python
   machine.wait_for_unit("default.target")
   machine.succeed("su -- alice -c 'which firefox'")
   machine.fail("su -- root -c 'which firefox'")
   ```

   该 Python 脚本引用 `machine`，这是在 `nodes` 属性集中为虚拟机配置所选用的名称。

   脚本等待 systemd 到达 `default.target`。
   它使用 `su` 命令在用户之间切换，并使用 `which` 命令检查用户是否能访问 `firefox`。
   它期望命令 `which firefox` 对用户 `alice` 成功，对 `root` 失败。

   该脚本将作为 `testScript` 属性的值。

完整的 `minimal-test.nix` 文件内容如下：

```nix
let
  nixpkgs = fetchTarball "https://github.com/NixOS/nixpkgs/tarball/nixos-23.11";
  pkgs = import nixpkgs { config = {}; overlays = []; };
in

pkgs.testers.runNixOSTest {
  name = "minimal-test";

  nodes.machine = { config, pkgs, ... }: {

    users.users.alice = {
      isNormalUser = true;
      extraGroups = [ "wheel" ];
      packages = with pkgs; [
        firefox
        tree
      ];
    };

    system.stateVersion = "23.11";
  };

  testScript = ''
    machine.wait_for_unit("default.target")
    machine.succeed("su -- alice -c 'which firefox'")
    machine.fail("su -- root -c 'which firefox'")
  '';
}
```

## 运行测试

要设置所有机器并运行测试脚本：

```shell-session
$ nix-build minimal-test.nix
```

    ...
    test script finished in 10.96s
    cleaning up
    killing machine (pid 10)
    (0.00 seconds)
    /nix/store/bx7z3imvxxpwkkza10vb23czhw7873w2-vm-test-run-minimal-test

## 在虚拟机中使用交互式 Python shell

开发测试或出现问题时，交互式地调试测试，或访问某台机器的终端会很有用。

要启动带测试框架的交互式 Python 会话：

```shell-session
$ $(nix-build -A driverInteractive minimal-test.nix)/bin/nixos-test-driver
```

在这里你可以运行任何测试操作。
使用 `test_script()` 函数执行来自 `minimal-test.nix` 的 `testScript` 属性。

如果虚拟机尚未启动，测试环境会在首次对 `machine` 对象调用方法时负责启动它。

但你也可以手动触发虚拟机启动：

```shell-session
>>> machine.start()
```
针对特定节点，

或

```shell-session
>>> start_all()
```
针对所有节点。

你可以使用以下方式进入虚拟机上的交互式 shell：

```shell-session
>>> machine.shell_interact()
```

并运行如下 shell 命令：

```shell-session
uname -a
```

    Linux server 5.10.37 #1-NixOS SMP Fri May 14 07:50:46 UTC 2021 x86_64 GNU/Linux


::::{dropdown} 重新运行已成功的测试

<!-- FIXME: this should be a separate recipe that can be linked to, as it's a bit of knowledge one will need now and again. -->

因为测试结果保存在 Nix store 中，成功的测试会被缓存。
这意味着只要测试设置（节点配置与测试脚本）在语义上保持不变，Nix 就不会再次运行该测试。
因此，要再次运行测试，需要删除结果。

如果尝试通过符号链接删除结果，你会得到以下错误：

```shell-session
nix-store --delete ./result
```

    finding garbage collector roots...
    0 store paths deleted, 0.00 MiB freed
    error: Cannot delete path '/nix/store/4klj06bsilkqkn6h2sia8dcsi72wbcfl-vm-test-run-unnamed' since it is still alive. To find out why, use: nix-store --query --roots

相反，先删除符号链接，然后再删除缓存的结果：

```shell-session
rm ./result
nix-store --delete /nix/store/4klj06bsilkqkn6h2sia8dcsi72wbcfl-vm-test-run-unnamed
```

也可以用一条命令完成：

```shell-session
result=$(readlink -f ./result) rm ./result && nix-store --delete $result
```
::::

## 涉及多台虚拟机的测试

测试可以涉及多台虚拟机，例如用于测试客户端–服务器通信。

以下示例设置包括：
- 一台名为 `server` 的虚拟机，运行带默认配置的 [nginx](https://nginx.org/en/)。
- 一台名为 `client` 的虚拟机，提供 `curl` 以发起 HTTP 请求。
- 一段在 `client` 与 `server` 之间编排测试逻辑的 `testScript`。

完整的 `client-server-test.nix` 文件内容如下：

```{code-block}
let
  nixpkgs = fetchTarball "https://github.com/NixOS/nixpkgs/tarball/nixos-23.11";
  pkgs = import nixpkgs { config = {}; overlays = []; };
in

pkgs.testers.runNixOSTest {
  name = "client-server-test";

  nodes.server = { pkgs, ... }: {
    networking = {
      firewall = {
        allowedTCPPorts = [ 80 ];
      };
    };
    services.nginx = {
      enable = true;
      virtualHosts."server" = {};
    };
  };

  nodes.client = { pkgs, ... }: {
    environment.systemPackages = with pkgs; [
      curl
    ];
  };

  testScript = ''
    server.wait_for_unit("default.target")
    client.wait_for_unit("default.target")
    client.succeed("curl http://server/ | grep -o \"Welcome to nginx!\"")
  '';
}
```

测试脚本执行以下步骤：
1) 启动服务器并等待其就绪。
1) 启动客户端并等待其就绪。
1) 在客户端上运行 `curl`，并用 `grep` 检查期望的返回字符串。
   测试根据返回值通过或失败。

运行测试：

```shell-session
$ nix-build client-server-test.nix
```

## 关于 NixOS 测试的补充信息

- 在 CI 上运行集成测试需要硬件加速，许多 CI 并不支持。

  要在 [GitHub Actions](<github-actions>) 中运行集成测试，参见[如何禁用硬件加速](https://github.com/cachix/install-nix-action#how-do-i-run-nixos-tests)。

- NixOS 自带大量可用作教学示例的测试。

  不错的灵感来源是 [Matrix 与 IRC 桥接](https://github.com/NixOS/nixpkgs/blob/master/nixos/tests/matrix/appservice-irc.nix)。

<!-- TODO: move examples from https://wiki.nixos.org/wiki/NixOS_Testing_library to the NixOS manual and troubleshooting tips to nix.dev -->

## 下一步

- [](module-system-deep-dive)
- [](bootable-iso-image)
- [](nixos-docker-images)
