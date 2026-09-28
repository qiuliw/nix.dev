(custom-binary-cache)=
# 配置 Nix 使用自定义二进制缓存

可通过 [`substituters`](https://nix.dev/manual/nix/latest/command-ref/conf-file.html#conf-substituters) 与 [`trusted-public-keys`](https://nix.dev/manual/nix/latest/command-ref/conf-file.html#conf-trusted-public-keys) 设置配置 Nix 使用二进制缓存，可单独使用，也可与 cache.nixos.org 一并使用。

:::{warning}
Nix 会接受任何用与已配置公钥对应的私钥签名的请求 store 对象。
因此，获得这些私钥的访问权限即可将任意文件替换进你的 Nix store。
这包括可能以提升权限运行或自动运行的可执行文件！

只添加你无条件信任的公钥。
:::

例如，给定位于 `https://example.org`、公钥为 `My56...Q==%` 的二进制缓存，以及 `default.nix` 中的某个 derivation，可通过[将设置作为命令行标志](https://nix.dev/manual/nix/latest/command-ref/conf-file#command-line-flags)传入，使 Nix 仅一次性独占使用该缓存：

```shell-session
$ nix-build --substituters https://example.org --trusted-public-keys example.org:My56...Q==%
```

要永久配置优先于公共缓存尝试自定义缓存，请将其以较低 `priority` 值作为 `extra-substiters` 添加到 [Nix 配置文件](https://nix.dev/manual/nix/latest/command-ref/conf-file#configuration-file)：

```shell-session
$ echo "extra-substituters = https://example.org?priority=30" >> /etc/nix/nix.conf
$ echo "extra-trusted-public-keys = example.org:My56...Q==%" >> /etc/nix/nix.conf
```

要始终只使用自定义缓存：

```shell-session
$ echo "substituters = https://example.org" >> /etc/nix/nix.conf
$ echo "trusted-public-keys = example.org:My56...Q==%" >> /etc/nix/nix.conf
```

::::{admonition} NixOS
在 NixOS 上，通过 [`nix.settings`](https://search.nixos.org/options?show=nix.settings) 选项配置 Nix：

```nix
{ ... }: {
  nix.settings = {
    substituters = [ "https://example.org?priority=30" ];
    trusted-public-keys = [ "example.org:My56...Q==%" ];
  };
}
```
::::

## 下一步

- 跟随教程[搭建你自己的 HTTP 二进制缓存](setup-http-binary-cache)

:::{tip}
使用[远程构建机器](distributed-build-setup-tutorial)作为首选二进制缓存，以减少外部流量。
:::
