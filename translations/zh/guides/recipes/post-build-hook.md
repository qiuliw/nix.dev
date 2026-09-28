(post-build-hooks)=
# 设置构建后钩子

```{contributors}
:authors: grahamc
```

本指南展示如何使用 Nix [`post-build-hook`](https://nix.dev/manual/nix/2.22/command-ref/conf-file#conf-post-build-hook) 配置选项，将构建结果自动上传到 [S3 兼容的二进制缓存](https://nix.dev/manual/nix/2.22/store/types/s3-binary-cache-store)。

(implementation-caveats)=
## 实现注意事项

这是一个简单且可用的示例，但并不适合所有用例。

构建后钩子程序在每次执行的构建之后运行，并阻塞构建循环。
如果钩子程序失败，构建循环会退出。

具体而言，当网络连接缓慢或不稳定时，此实现会使 Nix 变慢或不可用。
更高级的实现可能会将 store 路径传递给用户提供的守护进程或队列，以便在构建循环之外处理这些 store 路径。

## 先决条件

本教程假定你已[配置好 S3 兼容的二进制缓存](https://nix.dev/manual/nix/2.22/store/types/s3-binary-cache-store#authenticated-writes-to-your-s3-compatible-binary-cache)，并且 `root` 用户的默认 AWS 配置文件可以上传到该存储桶。

## 设置签名密钥

使用 [`nix-store --generate-binary-cache-key`](https://nix.dev/manual/nix/2.22/command-ref/nix-store/generate-binary-cache-key) 创建一对加密密钥。
你将用私钥签名路径，并分发公钥以验证路径的真实性。

```console
$ nix-store --generate-binary-cache-key example-nix-cache-1 /etc/nix/key.private /etc/nix/key.public
$ cat /etc/nix/key.public
example-nix-cache-1:1/cKDz3QCCOmwcztD2eV6Coggp6rqc9DGjWv7C0G+rM=
```

在任何将访问该存储桶的机器上 [](custom-binary-cache)。
例如，在 `nix.conf` 中将缓存 URL 添加到 [`substituters`](https://nix.dev/manual/nix/2.22/command-ref/conf-file#conf-substituters)，并将公钥添加到 [`trusted-public-keys`](https://nix.dev/manual/nix/2.22/command-ref/conf-file#conf-trusted-public-keys)：

```
substituters = https://cache.nixos.org/ s3://example-nix-cache
trusted-public-keys = cache.nixos.org-1:6NCHdD59X431o0gWypbMrAURkbJ16ZPMQFGspcDShjY= example-nix-cache-1:1/cKDz3QCCOmwcztD2eV6Coggp6rqc9DGjWv7C0G+rM=
```

为缓存进行构建的机器必须使用私钥对 derivation 签名。
包含你刚生成的私钥的文件路径必须添加到这些机器的 [`secret-key-files`](https://nix.dev/manual/nix/2.22/command-ref/conf-file#conf-secret-key-files) 设置中：

```
secret-key-files = /etc/nix/key.private
```

## 实现构建钩子

将以下脚本写入 `/etc/nix/upload-to-cache.sh`：

```bash
#!/bin/sh
set -eu
set -f # disable globbing
export IFS=' '
echo "Uploading paths" $OUT_PATHS
exec nix copy --to "s3://example-nix-cache" $OUT_PATHS
```

`$OUT_PATHS` 变量是以空格分隔的 Nix store 路径列表。
在本例中，我们期望并希望 shell 进行分词，使每个输出路径成为 `nix store sign` 的独立参数。
Nix 保证这些路径不包含任何空格，但 store 路径可能包含 glob 字符。
`set -f` 禁用 shell 中的 globbing。

确保 `root` 用户可以执行该钩子程序：

```console
# chmod +x /etc/nix/upload-to-cache.sh
```

## 更新 Nix 配置

在本地机器上将 [`post-build-hook`](https://nix.dev/manual/nix/2.22/command-ref/conf-file#conf-post-build-hook) 配置选项设为运行该钩子：

```
post-build-hook = /etc/nix/upload-to-cache.sh
```

然后在所有相关机器上重启 `nix-daemon`，例如：

```
pkill nix-daemon
```

## 测试

构建任意 derivation，例如：

```console
$ nix-build -E '(import <nixpkgs> {}).writeText "example" (builtins.toString builtins.currentTime)'
this derivation will be built:
  /nix/store/s4pnfbkalzy5qz57qs6yybna8wylkig6-example.drv
building '/nix/store/s4pnfbkalzy5qz57qs6yybna8wylkig6-example.drv'...
running post-build-hook '/home/grahamc/projects/github.com/NixOS/nix/post-hook.sh'...
post-build-hook: Signing paths /nix/store/ibcyipq5gf91838ldx40mjsp0b8w9n18-example
post-build-hook: Uploading paths /nix/store/ibcyipq5gf91838ldx40mjsp0b8w9n18-example
/nix/store/ibcyipq5gf91838ldx40mjsp0b8w9n18-example
```

要检查钩子是否生效，从 store 中删除该路径，并尝试从二进制缓存中替换它：

```console
$ rm ./result
$ nix-store --delete /nix/store/ibcyipq5gf91838ldx40mjsp0b8w9n18-example
$ nix-store --realise /nix/store/ibcyipq5gf91838ldx40mjsp0b8w9n18-example
copying path '/nix/store/m8bmqwrch6l3h8s0k3d673xpmipcdpsa-example from 's3://example-nix-cache'...
warning: you did not specify '--add-root'; the result might be removed by the garbage collector
/nix/store/m8bmqwrch6l3h8s0k3d673xpmipcdpsa-example
```

## 结论

你已配置 Nix，将每次本地构建自动签名并上传到远程 S3 兼容的二进制缓存。

在部署到生产环境之前，请务必考虑[实现注意事项](#implementation-caveats)。
