---
myst:
  html_meta:
    "description lang=en": "Building and running Docker images"
    "keywords": "Docker, containers, Nix, reproducible, build, tutorial"
---

(nixos-docker-images)=
# 构建并运行 Docker 镜像

```{contributors}
:authors: domenkozar
```

[Docker](https://www.docker.com/) 是一套用于构建、管理和部署容器的工具与服务。

许多云平台提供基于 Docker 的容器托管。
在构建可复现软件时，创建 Docker 容器是一项常见任务。
在本教程中，你将学习如何使用 Nix 构建 Docker 容器。

## 前置条件

你需要同时安装 Nix 和 [Docker](https://docs.docker.com/get-docker/)。
Docker 可在 `nixpkgs` 中获取，这也是在 NixOS 上安装它的首选方式。
不过，如果你使用其他 Linux 发行版或 macOS，也可以使用操作系统自带的 Docker 安装方式。

## 构建你的第一个容器

[Nixpkgs](https://github.com/NixOS/nixpkgs) 提供了 `dockerTools` 用于创建 Docker 镜像：

```nix
{ pkgs ? import <nixpkgs> { }
, pkgsLinux ? import <nixpkgs> { system = "x86_64-linux"; }
}:

pkgs.dockerTools.buildImage {
  name = "hello-docker";
  config = {
    Cmd = [ "${pkgsLinux.hello}/bin/hello" ];
  };
}
```

:::{note}
如果你在运行 **macOS** 或任何非 `x86_64-linux` 的平台，你需要：

- [设置远程构建机](distributed-build-setup-tutorial) 以在 Linux 上构建
- [交叉编译到 Linux](cross-compilation)，将 `pkgsLinux.hello` 替换为 `pkgs.pkgsCross.musl64.hello`
:::

我们调用 `dockerTools.buildImage` 并传入一些参数：

- 镜像的 `name`
- `config`，其中包括镜像启动后应在容器内运行的命令 `Cmd`。
  这里我们引用 `nixpkgs` 中的 GNU hello 软件包，并在容器中运行其实可执行文件。

将内容保存为 `hello-docker.nix` 并构建：

```shell-session
$ nix-build hello-docker.nix
these derivations will be built:
  /nix/store/qpgdp0qpd8ddi1ld72w02zkmm7n87b92-docker-layer-hello-docker.drv
  /nix/store/m4xyfyviwbi38sfplq3xx54j6k7mccfb-runtime-deps.drv
  /nix/store/v0bvy9qxa79izc7s03fhpq5nqs2h4sr5-docker-image-hello-docker.tar.gz.drv
warning: unknown setting 'experimental-features'
building '/nix/store/qpgdp0qpd8ddi1ld72w02zkmm7n87b92-docker-layer-hello-docker.drv'...
No contents to add to layer.
Packing layer...
Computing layer checksum...
Finished building layer 'hello-docker'
building '/nix/store/m4xyfyviwbi38sfplq3xx54j6k7mccfb-runtime-deps.drv'...
building '/nix/store/v0bvy9qxa79izc7s03fhpq5nqs2h4sr5-docker-image-hello-docker.tar.gz.drv'...
Adding layer...
tar: Removing leading `/' from member names
Adding meta...
Cooking the image...
Finished.
/nix/store/y74sb4nrhxr975xs7h83izgm8z75x5fc-docker-image-hello-docker.tar.gz
```

镜像标签（`y74sb4nrhxr975xs7h83izgm8z75x5fc`）对应 Nix 构建哈希，确保 Docker 镜像与我们的 Nix 构建一致。
输出最后一行中的 store 路径引用了该 Docker 镜像。

## 运行容器

要使用该容器，从 `nix-build` 默认创建的 `result` 符号链接将镜像加载到 Docker 的镜像仓库：

```shell-session
$ docker load < result
Loaded image: hello-docker:y74sb4nrhxr975xs7h83izgm8z75x5fc
```

你也可以使用 store 路径来加载镜像，从而不必依赖 `result` 是否存在：

```shell-session
$ docker load < /nix/store/y74sb4nrhxr975xs7h83izgm8z75x5fc-docker-image-hello-docker.tar.gz
Loaded image: hello-docker:y74sb4nrhxr975xs7h83izgm8z75x5fc
```

更方便的是，你可以在一条命令中完成全部操作。
这种方式的优点是：若有任何更改，`nix-build` 会重新构建镜像，并将新的 store 路径传给 `docker load`：

```shell-session
$ docker load < $(nix-build hello-docker.nix)
Loaded image: hello-docker:y74sb4nrhxr975xs7h83izgm8z75x5fc
```

现在镜像已加载到 Docker，你可以运行它：

```shell-session
$ docker run -t hello-docker:y74sb4nrhxr975xs7h83izgm8z75x5fc
Hello, world!
```

## 使用 Docker 镜像

本教程不包含 Docker 镜像使用的一般介绍。
[官方 Docker 文档](https://docs.docker.com/) 是更好的学习资源。

注意，当你用 Nix 构建 Docker 镜像时，通常不会编写 `Dockerfile`，因为 Nix 在 Docker 生态中替代了 Dockerfile 的功能。
尽管如此，了解 Dockerfile 的结构仍然有助于理解 Nix 如何替代其中各项功能。
另一方面，是否使用 Docker CLI、Docker Compose、Docker Swarm 或 Docker Hub，仍取决于你的具体用例。

## 下一步

- 关于如何使用 `dockerTools` 的更多细节，请参阅[参考文档](https://nixos.org/nixpkgs/manual/#sec-pkgs-dockerTools)。
- 你可以浏览更多[用 Nix 构建的 Docker 镜像示例](https://github.com/NixOS/nixpkgs/blob/master/pkgs/build-support/docker/examples.nix)。
- 看看 [Arion](https://docs.hercules-ci.com/arion/)，一个对 Nix 有一流支持的 `docker-compose` 封装。
- 在 {ref}`使用 GitHub Actions 的 CI <github-actions>` 上构建 docker 镜像。
