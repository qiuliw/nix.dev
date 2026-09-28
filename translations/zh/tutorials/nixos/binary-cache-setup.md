---
myst:
  html_meta:
    "description lang=en": "Setting up a binary cache for store objects"
    "keywords": "Nix, caching"
---

(setup-http-binary-cache)=
# 设置 HTTP 二进制缓存

```{contributors}
:authors: tfc
:editors: fricklerhandwerk
```

二进制缓存存储预构建的 [Nix store 对象](https://nix.dev/manual/nix/latest/store/store-object)，并通过网络提供给其他机器。
任何带有 Nix store 的机器都可以成为其他机器的二进制缓存。

## 简介

在本教程中，你将在一台 NixOS 机器上设置 Nix 二进制缓存，通过 HTTP 或 HTTPS 提供 store 对象。

### 你将学到什么？

你将学会如何：
- 为缓存设置签名密钥
- 在提供缓存的 NixOS 机器上启用正确的服务
- 检查设置是否按预期工作

### 你需要什么？

- 本地机器上可用的 [Nix 安装](<install-nix>)

- 对一台用作缓存的 NixOS 机器的 SSH 访问权限

  如果你刚接触 NixOS，请了解[模块系统](module-system-tutorial)，并用 [](nixos-vms) 配置你的第一台系统。

- （可选）公网 IP 和 DNS 域名

  如果你不自行托管，请查看 NixOS Wiki 上的 [NixOS-friendly hosters](https://wiki.nixos.org/wiki/NixOS_friendly_hosters)。
  按照 [](provisioning-remote-machines-tutorial) 教程部署你的 NixOS 配置。

对于局域网中的缓存，我们假定：
- 主机名为 `cache`（请替换为你的主机名，或使用 IP 地址）
- 主机通过 HTTP 在 80 端口提供 store 对象（这是默认值）

对于可公开访问的缓存，我们假定：
- 域名为 `cache.example.com`（请替换为你的域名）
- 主机通过 HTTPS 在 443 端口提供 store 对象（这是默认值）

### 需要多久？

- 25 分钟

## 设置服务

对于托管缓存的 NixOS 机器，在 `binary-cache.nix` 中创建新的配置模块：

```{code-block} nix
{ config, ... }:

{
  services.nix-serve = {
    enable = true;
    secretKeyFile = "/var/secrets/cache-private-key.pem";
  };

  services.nginx = {
    enable = true;
    recommendedProxySettings = true;
    virtualHosts.cache = {
      locations."/".proxyPass = "http://${config.services.nix-serve.bindAddress}:${toString config.services.nix-serve.port}";
    };
  };

  networking.firewall.allowedTCPPorts = [
    config.services.nginx.defaultHTTPListenPort
  ];
}
```

[`services.nix-serve`] 下的选项用于配置二进制缓存服务。

`nix-serve` 不支持 IPv6 或 SSL/HTTPS。
[`services.nginx`] 选项用于设置代理，该代理支持 IPv6，并处理发往主机名 `cache` 的请求。

[`services.nix-serve`]: https://search.nixos.org/options?query=services.nix-serve
[`services.nginx`]: https://search.nixos.org/options?query=services.nginx

:::{important}
本教程末尾有一个[可选的 HTTPS 小节](https-binary-cache)。
:::

将新的 NixOS 模块添加到现有机器配置中：

```{code-block} nix
{ config, ... }:

{
  imports = [
    ./binary-cache.nix
  ];

  # ...
}
```

从本地机器部署新配置：

```shell-session
nixos-rebuild switch --no-flake --target-host root@cache
```

:::{note}
由于尚无私钥文件，二进制缓存守护进程会报告错误。
:::

## 生成签名密钥对

你需要一对私钥和公钥，以确保缓存中的 store 对象是可信的。

<!-- TODO: link to the remote builds tutorial for the case where store objects are signed after building them -->

要为二进制缓存生成密钥对，请将示例主机名 `cache.example.com` 替换为你的主机名：

```shell-session
nix-store --generate-binary-cache-key cache.example.com cache-private-key.pem cache-public-key.pem
```

`cache-private-key.pem` 将由二进制缓存守护进程在提供二进制文件时用于签名。
将其复制到托管缓存的机器上 `services.nix-serve.secretKeyFile` 所配置的位置：

```shell-session
scp cache-private-key.pem root@cache:/var/secrets/cache-private-key.pem
```

到目前为止，由于缺少私钥文件，二进制缓存守护进程一直在重启循环中。
检查它现在是否正常工作：

```shell-session
ssh root@cache systemctl status nix-serve.service
```

:::{important}
在本地机器上使用 `cache-public-key.pem` [](custom-binary-cache)。
:::

## 测试可用性

以下步骤检查一切是否正确设置，并可能有助于排查问题。

### 检查基本可用性

通过查询缓存，测试二进制缓存、反向代理和防火墙规则是否按预期工作：

```shell-session
$ curl http://cache/nix-cache-info
StoreDir: /nix/store
WantMassQuery: 1
Priority: 30
```

### 检查 store 对象签名

要测试 store 对象是否正确签名，请检查示例 derivation 的元数据。
在二进制缓存主机上，构建 `hello` 软件包并从缓存获取 `.narinfo` 文件：

```shell-session
$ hash=$(nix-build '<nixpkgs>' -A pkgs.hello | awk -F '/' '{print $4}' | awk -F '-' '{print $1}')
$ curl "http://cache/$hash.narinfo" | grep "Sig: "
...
Sig: cache.example.org:GyBFzocLAeLEFd0hr2noK84VzPUw0ArCNYEnrm1YXakdsC5FkO2Bkj2JH8Xjou+wxeXMjFKa0YP2AML7nBWsAg==
```

确保输出包含此前缀为 `Sig:` 的行，并显示你生成的公钥。

(https-binary-cache)=
### 通过 HTTPS 提供二进制缓存

如果二进制缓存可公开访问，可以用 [Let's Encrypt](https://letsencrypt.org/) SSL 证书强制使用 HTTPS。
像这样编辑你的 `binary-cache.nix`，并确保将示例 URL 和邮箱地址替换为你的：

```{code-block} diff
   services.nginx = {
     enable = true;
     recommendedProxySettings = true;
-    virtualHosts.cache = {
+    virtualHosts."cache.example.com" = {
+      enableACME = true;
+      forceSSL = true;
       locations."/".proxyPass = "http://${config.services.nix-serve.bindAddress}:${toString config.services.nix-serve.port}";
     };
   };

+   security.acme = {
+     acceptTerms = true;
+     certs = {
+       "cache.example.com".email = "you@example.com";
+     };
+   };

   networking.firewall.allowedTCPPorts = [
     config.services.nginx.defaultHTTPListenPort
+    config.services.nginx.defaultSSLListenPort
   ];
```

重建系统以部署这些变更：

```shell-session
nixos-rebuild switch --no-flake --target-host root@cache.example.com
```

## 下一步

如果你的二进制缓存已经是一台[远程构建机](https://nix.dev/manual/nix/latest/advanced-topics/distributed-builds)，它将提供其 Nix store 中的所有 store 对象。

- 使用二进制缓存的主机名和生成的公钥 [](custom-binary-cache)
- 用 [](post-build-hooks) 将 store 对象上传到二进制缓存
- [](distributed-build-setup-tutorial)

为节省存储空间，请参考以下 NixOS 配置属性：

- [`nix.gc`](https://search.nixos.org/options?query=nix.gc)：自动垃圾回收选项
- [`nix.optimise`](https://search.nixos.org/options?query=nix.optimise)：定期优化 Nix store 的选项

## 替代方案

- [`nix-serve-ng`](https://github.com/aristanetworks/nix-serve-ng)：用 Haskell 编写的 `nix-serve` 即插即用替代品

- [SSH Store](https://nix.dev/manual/nix/latest/store/types/ssh-store)、[Experimental SSH Store](https://nix.dev/manual/nix/latest/store/types/experimental-ssh-store) 以及 [S3 Binary Cache Store](https://nix.dev/manual/nix/latest/store/types/s3-binary-cache-store) 也可用于提供缓存。
  有许多提供 S3 兼容存储的商业服务商，例如：
  - Amazon S3
  - Tigris
  - Cloudflare R2

- [attic](https://github.com/zhaofengli/attic)：基于 S3 兼容存储提供商的 Nix 二进制缓存服务器

- [Cachix](https://www.cachix.org)：作为服务的 Nix 二进制缓存

## 参考

- [Nix Manual on HTTP Binary Cache Store](https://nix.dev/manual/nix/latest/store/types/http-binary-cache-store)
- [`services.nix-serve` 模块选项][`services.nix-serve`]
- [`services.nginx` 模块选项][`services.nginx`]
