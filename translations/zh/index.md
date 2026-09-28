---
myst:
  html_meta:
    "description lang=zh": "使用 Nix 完成实际工作的官方文档。"
    "keywords": "Nix, Nixpkgs, NixOS, Linux, 构建系统, 部署, 打包, 声明式, 可复现, 不可变, 软件, 开发者"
    "property=og:locale": "zh_CN"
---


# 欢迎来到 nix.dev

nix.dev 是 Nix 生态官方文档的家园。
由 [Nix 文档团队](https://nixos.org/community/teams/documentation) 维护。

本中文站为社区翻译版本；尚未翻译的页面会暂时显示英文原文。

如果你是初学者，请先 {ref}`安装 Nix <install-nix>`，然后从教程系列开始学习！

::::{grid} 2
:::{grid-item-card} 教程
:link: tutorials
:link-type: ref
:text-align: center

循序渐进的入门课程
:::

:::{grid-item-card} 指南
:link: guides
:link-type: ref
:text-align: center

完成具体任务的操作指南
:::
::::

::::{grid} 2
:::{grid-item-card} 参考
:link: reference
:link-type: ref
:text-align: center

详细的技术说明合集
:::

:::{grid-item-card} 概念
:link: concepts
:link-type: ref
:text-align: center

Nix 生态的历史与思想解读
:::
::::

## 用 Nix 可以做什么？

借助 Nix 生态，你可以：

- {ref}`可复现的开发环境 <ad-hoc-envs>`。
- 通过 URL 轻松安装软件。
- 在多台计算机之间轻松迁移软件环境。
- {ref}`用声明式方式描述 Linux 机器 <deploy-nixos-using-terraform>`。
- {ref}`用虚拟机做可复现的集成测试 <integration-testing-vms>`。
- 避免与已安装软件发生版本冲突。
- 从源代码安装软件。
- {ref}`通过二进制缓存实现透明的构建缓存 <github-actions>`。
- 对软件可审计性有强支持。
- {ref}`一流的交叉编译支持 <cross-compilation>`。
- 远程构建。
- 远程部署。
- 原子升级与回滚。


## Nix 适合谁？

Nix 适合那些既希望计算机长期、可重复地严格按预期工作，又熟悉命令行与纯文本编辑器的人。

你不必是职业软件开发者，也不需要正式的信息学学历，就能从 Nix 中获益良多。
不过，复杂软件项目经验和一些信息学知识，有助于理解它为什么有用、如何工作，以及如何高效使用并[做出改进](how-to-contribute)。

如果你属于以下角色，用上 Nix 之后往往很难再回头：

- 全栈或后端开发者
- 测试工程师
- 嵌入式系统开发者
- DevOps 工程师
- 系统管理员
- 数据科学家
- 自然科学研究者
- 工科学生
- 开源软件爱好者


```{toctree}
:hidden:

install-nix.md
tutorials/index.md
guides/index.md
reference/index.md
concepts/index.md
contributing/index.md
acknowledgements/index.md
```
