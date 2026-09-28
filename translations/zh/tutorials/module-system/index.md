(module-system-tutorial)=
# 模块系统

Nixpkgs 与 NixOS 的许多能力都来自模块系统。

模块系统是一个 Nix 语言库，它能让你：
- 用许多独立的 Nix 表达式声明同一个属性集。
- 对该属性集中的值施加类型约束。
- 在不同 Nix 表达式中为同一属性定义值，并按其类型自动合并。

这些 Nix 表达式称为模块，必须具有特定结构。

在本教程系列中你将学习：
- 什么是模块，以及如何创建模块。
- 什么是 options，以及如何声明它们。
- 如何表达模块之间的依赖关系。

## 你需要什么？

- 熟悉数据类型与通用编程概念
- 一份 {ref}`Nix 安装 <install-nix>` 以便运行示例
- 具备阅读与编写 {ref}`Nix 语言 <reading-nix-language>` 的中级能力

## 需要多久？

这是一篇很长的教程。
请至少预留 3 小时。

```{toctree}
:maxdepth: 1
:caption: 课程
:numbered:
a-basic-module/index.md
deep-dive.md
```
