---
myst:
  html_meta:
    "description lang=en": "Continuous Integration with GitHub Actions and a binary cache"
    "keywords": "CI, Continuous Integration, GitHub Actions, Binary Cache, Nix"
---

(github-actions)=

# 使用 GitHub Actions 的持续集成

```{contributors}
:authors: domenkozar
```

将 [GitHub Actions](https://github.com/features/actions) 设置为提交与拉取请求的持续集成（CI）工作流。

借助 Nix，CI 可通过二进制缓存为每个项目的每个分支构建并缓存开发者环境。

构建时间是关键的 CI 指标。Cachix（见下文）是最直接的缓存方案。

## 使用 Cachix 缓存构建

使用 [Cachix](https://cachix.org/)，你不必浪费时间重复构建同一个 derivation，还可以与所有开发者共享已构建的 derivation。

每次任务结束后，刚构建的 derivation 会被推送到你的二进制缓存。

每次任务开始前，待构建的 derivation 会先从你的二进制缓存中替换（若存在）。

### 1. 创建你的第一个二进制缓存

建议按团队使用不同的二进制缓存，取决于谁对其拥有写/读权限。

在[创建二进制缓存](https://app.cachix.org/cache)页面填写表单。

在新创建的二进制缓存上，按照 **Push binaries** 选项卡中的说明操作。

### 2. 设置密钥

在你的 GitHub 仓库或组织（可跨所有仓库使用）上：

1. 点击 `Settings`。
2. 点击 `Secrets and variables`，并在下拉列表中点击 `Actions`
3. 点击 `New repository secret`。
4. 添加你先前生成的密钥（`CACHIX_SIGNING_KEY` 和/或 `CACHIX_AUTH_TOKEN`）。

### 3. 设置 GitHub Actions

创建 `.github/workflows/test.yml`，内容如下：

```yaml
name: "Test"
on:
  pull_request:
  push:
jobs:
  tests:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - uses: cachix/install-nix-action@v25
      with:
        nix_path: nixpkgs=channel:nixos-unstable
    - uses: cachix/cachix-action@v14
      with:
        name: mycache
        # If you chose signing key for write access
        signingKey: '${{ secrets.CACHIX_SIGNING_KEY }}'
        # If you chose API tokens for write access OR if you have a private cache
        authToken: '${{ secrets.CACHIX_AUTH_TOKEN }}'
    - run: nix-build
    - run: nix-shell --run "echo OK"
```

一旦提交并推送到你的 GitHub 仓库，
你应能在提交和 PR 上看到状态检查出现。

## 下一步

- 参见 [GitHub Actions 工作流语法](https://docs.github.com/en/actions/reference/workflow-syntax-for-github-actions)
- 要快速搭建 Nix 项目，请阅读
  [Getting started Nix template](https://github.com/nix-dot-dev/getting-started-nix-template)。

[github-actions-caching-limits]: https://docs.github.com/en/actions/using-workflows/caching-dependencies-to-speed-up-workflows
