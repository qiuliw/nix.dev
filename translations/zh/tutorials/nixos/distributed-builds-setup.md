---
myst:
  html_meta:
    "description lang=en": "Setting up distributed builds"
    "keywords": "Nix, builds, distribution, scaling"
---

(distributed-build-setup-tutorial)=
# 设置分布式构建

```{contributors}
:authors: tfc
:editors: fricklerhandwerk
```

Nix 可以通过将工作同时分摊到多台计算机上来加速构建。

## 简介

在本教程中，你将设置一台独立的构建机器，并配置本地机器将构建任务卸载到它上面。

### 你将学到什么？

你将学习如何
- 为从本地机器到远程构建器的远程构建访问创建新用户
- 以可持续的方式配置远程构建器
- 测试远程构建器的连通性与认证
- 配置本地机器以自动分发构建任务

### 你需要什么？

- 熟悉 [Nix 语言](reading-nix-language)
- 熟悉 [](module-system-tutorial)

- 一台*本地机器*（示例主机名：`localmachine`）

  这台计算机[已安装 Nix](install-nix)，负责将构建任务分发给其他机器。

- 一台*远程机器*（示例主机名：`remotemachine`）

  一台运行 NixOS 的计算机，接受来自*本地机器*的构建任务。
  请按 [](provisioning-remote-machines-tutorial) 设置远程 NixOS 系统。

### 需要多长时间？

- 25 分钟

## 创建 SSH 密钥对

*本地机器*上的 Nix 守护进程以 `root` 用户运行，需要*私钥*文件以向远程机器认证自身。
*远程机器*需要*公钥*以识别*本地机器*。

在*本地机器*上，以 `root` 身份运行以下命令创建 SSH 密钥对：

```shell-session
# ssh-keygen -f /root/.ssh/remotebuild
```

:::{note}
密钥对文件的名称与位置可以自由选择。
:::

(set-up-remote-builder)=
## 设置远程构建器

在*远程机器*的 NixOS 配置目录中，创建文件 `remote-builder.nix`：

```{code-block} nix
{
  users.users.remotebuild = {
    isSystemUser = true;
    group = "remotebuild";
    useDefaultShell = true;

    openssh.authorizedKeys.keyFiles = [ ./remotebuild.pub ];
  };

  users.groups.remotebuild = {};

  nix.settings.trusted-users = [ "remotebuild" ];
}
```

将文件 `remotebuild.pub` 复制到该目录。

该配置模块创建一个没有主目录的新用户 `remotebuild`。
*本地机器*上的 `root` 用户将能使用先前生成的 SSH 密钥通过 SSH 登录远程构建器。

将新的 NixOS 模块加入*远程机器*的现有配置：

```{code-block} nix
{
  imports = [
    ./remote-builder.nix
  ];

  # ...
}
```

以 root 身份激活新配置：

```shell-session
nixos-rebuild switch --no-flake --target-host root@remotemachine
```

### 测试认证

确认 SSH 连接与认证可用。
在*本地机器*上以 `root` 身份运行：

```shell-session
# ssh remotebuild@remotemachine -i /root/.ssh/remotebuild "echo hello"
hello
```

如果能看到 `hello` 消息，说明认证可用。

这次测试登录也会将远程构建器的主机密钥添加到本地机器的 `/root/.ssh/known_hosts` 文件中。
之后的登录不会再被主机密钥检查打断。

(set-up-distributed-builds)=
## 设置分布式构建

:::{note}
如果你的*本地机器*运行的是 NixOS，请跳过本节，并[通过模块选项配置 Nix](distributed-builds-config-nixos)。
:::

以 `root` 身份向 [Nix 配置文件](https://nix.dev/manual/nix/2.23/command-ref/conf-file) 追加内容，配置 Nix 使用远程构建器：

```
# cat << EOF >> /etc/nix/nix.conf
builders = ssh-ng://remotebuild@remotebuilder $(nix-instantiate --eval -E builtins.currentSystem) /root/.ssh/remotemachine - - nixos-test,big-parallel,kvm
builders-use-substitutes = true
```

::::{dropdown} 详细说明
第一行通过指定以下内容将远程机器注册为远程构建器：
- 协议、用户与主机名
- *本地机器*的[系统类型](https://nix.dev/manual/nix/2.23/command-ref/conf-file#conf-system)

  这会将该系统类型的任务委托给*远程机器*。

- SSH 密钥的位置
- 一份[受支持的系统特性](https://nix.dev/manual/nix/2.23/command-ref/conf-file#conf-system-features)列表

  必须指定这一特定列表，才能将编译器构建以及运行 [NixOS VM 测试](integration-testing-vms) 委托给远程机器。

详情见 [`builders` 设置的参考文档](https://nix.dev/manual/nix/2.23/command-ref/conf-file#conf-builders)。

第二行指示所有远程构建器从各自的二进制缓存获取依赖，而不是从*本地机器*获取。
这假定远程构建器的互联网连接至少与本地机器一样快。
::::

要激活该配置，重启 Nix 守护进程：

:::::{tab-set}
::::{tab-item} Linux
在使用 `systemd` 的 Linux 上，以 `root` 身份运行：

```shell-session
# systemctl restart nix-daemon.service
```
::::

::::{tab-item} macOS
在 macOS 上，以 `root` 身份运行：

```shell-session
# sudo launchctl stop org.nixos.nix-daemon
# sudo launchctl start org.nixos.nix-daemon
```
::::
:::::


(distributed-builds-config-nixos)=
:::::{admonition} NixOS

如果你的*本地机器*运行的是 NixOS，在其配置目录中创建文件 `distributed-builds.nix`：

```{code-block} nix
{ pkgs, ... }:
{
  nix.distributedBuilds = true;
  nix.settings.builders-use-substitutes = true;

  nix.buildMachines = [
    {
      hostName = "remotebuilder";
      sshUser = "remotebuild";
      sshKey = "/root/.ssh/remotebuild";
      system = pkgs.stdenv.hostPlatform.system;
      supportedFeatures = [ "nixos-test" "big-parallel" "kvm" ];
    }
  ];
}
```

::::{dropdown} 详细说明
该配置模块启用分布式构建并添加远程构建器，指定：
- SSH 主机名与用户名
- SSH 密钥的位置
- *本地机器*的哪种[系统类型](https://nix.dev/manual/nix/2.23/command-ref/conf-file#conf-system)

  这会将该系统类型的任务委托给*远程机器*。

- 一份[受支持的系统特性](https://nix.dev/manual/nix/2.23/command-ref/conf-file#conf-system-features)列表

  必须指定这一特定列表，才能将编译器构建以及运行 [NixOS VM 测试](integration-testing-vms) 委托给远程机器。

详情见 [NixOS 关于 `nix.buildMachines` 的选项文档](https://search.nixos.org/options?query=nix.buildMachines)。

`builders-use-substitutes` 指示所有远程构建器从各自的二进制缓存获取依赖，而不是从*本地机器*获取。
这假定远程构建器的互联网连接至少与本地机器一样快。
::::

将新的 NixOS 模块加入现有机器配置：

```{code-block} nix
{
  imports = [
    ./distributed-builds.nix
  ];

  # ...
}
```

以 `root` 身份激活新配置：

```shell-session
# nixos-rebuild switch
```
:::::

## 测试分布式构建

尝试在*本地机器*上构建一个新的 derivation：

```shell-session
$ nix-build --max-jobs 0 -E "$(cat << EOF
(import <nixpkgs> {}).writeText "test" "$(date)"
EOF
)"
this derivation will be built:
  /nix/store/9csjdxv6ir8ccnjl6ijs36izswjgchn0-test.drv
building '/nix/store/9csjdxv6ir8ccnjl6ijs36izswjgchn0-test.drv' on 'ssh://remotebuilder'...
copying 0 paths...
copying 1 paths...
copying path '/nix/store/hvj5vyg4723nly1qh5a8daifbi1yisb3-test' from 'ssh://remotebuilder'...
/nix/store/hvj5vyg4723nly1qh5a8daifbi1yisb3-test
```

由于结果 derivation 依赖于当前系统时间，每次调用都会变化，因此永远不会在本地缓存中。
[`--max-jobs 0` 命令行参数](https://nix.dev/manual/nix/2.23/command-ref/conf-file#conf-max-jobs)强制 Nix 在远程构建器上构建它。

最后一行输出包含输出路径，表明构建分发按预期工作。

## 优化远程构建器配置

为最大化并行度、启用自动垃圾回收，并防止 Nix 构建占用全部内存，向 `remote-builder.nix` 配置模块添加以下行：

```{code-block} diff
 {
   users.users.remotebuild = {
     isNormalUser = true;
     createHome = false;
     group = "remotebuild";

     openssh.authorizedKeys.keyFiles = [ ./remotebuild.pub ];
   };

   users.groups.remotebuild = {};

-  nix.settings.trusted-users = [ "remotebuild" ];
+  nix = {
+    nrBuildUsers = 64;
+    settings = {
+      trusted-users = [ "remotebuild" ];
+
+      min-free = 10 * 1024 * 1024;
+      max-free = 200 * 1024 * 1024;

+      max-jobs = "auto";
+      cores = 0;
+    };
+  };

+  systemd.services.nix-daemon.serviceConfig = {
+    MemoryAccounting = true;
+    MemoryMax = "90%";
+    OOMScoreAdjust = 500;
+  };
}
```

:::{tip}
关于 [`nix.settings`](https://search.nixos.org/options?show=nix.settings) 中可用选项的详情，请参阅 [Nix 参考手册](https://nix.dev/manual/nix/latest/command-ref/conf-file)。
:::

远程构建器可能具有不同的性能特征。
对每个 `nix.buildMachines` 项，为各台不同的远程构建器正确设置 `maxJobs`、`speedFactor` 和 `supportedFeatures` 属性。
这有助于*本地机器*上的 Nix 以最优方式分发构建。

:::{tip}
详情见 [NixOS 关于 `nix.buildMachines` 的选项文档](https://search.nixos.org/options?query=nix.buildMachines)。
:::

将 `nix.buildMachines.*.publicHostKey` 字段设为各远程构建器的公钥主机密钥，以防中间人场景破坏构建分发。

## 下一步

- 在每台远程构建器上 [](custom-binary-cache)
- [](post-build-hooks) 以将 store 对象上传到二进制缓存

要设置多台构建器，对每台远程构建器重复 [](set-up-remote-builder) 一节中的说明。
将所有新的远程构建器添加到 [](set-up-distributed-builds) 一节所示的 `nix.buildMachines` 属性中。

:::{tip}
将远程构建机器配置为[托管二进制缓存](setup-http-binary-cache)，并将其用作[首选二进制缓存](custom-binary-cache)，以减少外部流量。
:::

## 替代方案

- [nixbuild.net](https://nixbuild.net) - 作为服务提供的 Nix 远程构建器
- [Hercules CI](https://hercules-ci.com/) - 带自动构建分发的持续集成
- [garnix](https://garnix.io/) - 带构建分发的托管持续集成

## 参考

- [Nix 参考手册：分布式构建设置](https://nix.dev/manual/nix/latest/command-ref/conf-file#conf-builders)
