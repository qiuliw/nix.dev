(how-to-contribute)=
# 如何贡献

Nix 生态由众多志愿者与少数受薪开发者共同开发，维护着世界上最大的开源软件发行版之一。
要让它持续可用、保持更新，并不断改进，离不开你的支持。

本指南说明如何为 {term}`Nix`、{term}`Nixpkgs` 或 {term}`NixOS` 做贡献。
假定你已对基础概念与工作流较为熟悉，这些内容在[入门教程系列](tutorials)中有概述。
最重要的方面包括：[Nix 语言](reading-nix-language)、Nixpkgs 中用于[构建软件的 derivation](packaging-tutorial) 的各种机制、[模块系统](module-system-tutorial)，以及 [NixOS 集成测试](integration-testing-vms)。

:::{important}
如果你无法贡献时间，可考虑[通过 Open Collective 向 NixOS Foundation 捐款](https://opencollective.com/nixos)。

目前重点是[资助线下活动](https://github.com/NixOS/foundation/issues?q=is%3Aissue%20label%3Afunding-request%20)，以分享知识并壮大精通 Nix 的开发者社区。
若预算充足，就有可能为关键基础设施与代码的持续维护和开发付费——这类要求很高的工作，我们无法指望志愿者无限期承担。
:::

## 入门

先阅读[参考文档](reference)以及与你关心内容相关的代码，再提出有根据的问题。

[加入我们的社区交流平台](https://nixos.org/community)，与其他用户和开发者取得联系。
如果你对某个特定主题感兴趣，可以了解并考虑参与我们的[社区团队](https://nixos.org/community/#governance-teams)。

所有源代码与文档都在 [GitHub](https://github.com/NixOS) 上，提出变更需要 GitHub 账号。
技术讨论发生在 issue 与 pull request 的评论中。

:::{tip}
如果你是 Nix 新手，建议先从[贡献文档](./documentation/index.md)开始。

这是我们最需要帮助、也最容易上手的地方。

文档与贡献指南往往不完整或过时——尽管我们希望并非如此。
我们正在改进。
你可以在遇到贡献流程问题时立即加以解决，从而改善每个人的处境。
这可能会拖慢你解决最初关切的进度。
但会让任何人在未来更容易做出有意义的贡献。
从长远看，也会带来更好的代码与文档。
:::

## 报告问题

:::{note}
如需询问关于代码或如何操作的一般性问题，请使用我们的[社区交流平台](https://nixos.org/community)

要陈述技术问题并提出解决方案，请在 GitHub 上开 issue，并在问题解决或不再成立时关闭它们。
:::

我们只能修复已知的问题，因此请报告你遇到的任何问题。

- 报告 {term}`Nix` 相关问题（包括 [Nix 参考手册](https://nix.dev/manual/nix/stable)）请至 <https://github.com/NixOS/nix/issues>。

- 报告 {term}`Nixpkgs` 或 {term}`NixOS` 相关问题（包括软件包、配置模块、[Nixpkgs 手册](https://nixos.org/manual/nixpkgs/stable) 与 [NixOS 手册](https://nixos.org/manual/nixos/stable)）请至 <https://github.com/NixOS/nixpkgs/issues>。

请确认尚无已有针对你问题的未关闭 issue。
请遵循 issue 模板并填写所有要求的信息。

请特别注意提供最小且易于理解的示例来复现你所面临的问题。
你还应展示自己尝试解决问题时的发现。
这会大大提高问题最终得到解决的可能性，且出于多个重要原因：

- 可复现的示例简洁且无歧义。

  这有助于对 issue 进行分类、理解问题、找到根因并制定解决方案。
  你的初步研究还能进一步帮助维护者进行分析。

- 它让任何人都能判断该 issue 是否仍然相关。

  Issue 可能长时间无人处理。
  即便过了数月或数年，决定如何处理它们也需要检查底层问题是否仍然存在或已解决。
  这必须易于完成：这样任何人都能协助分类，并通知维护者关闭或重新排定优先级。

- 该示例可在解决问题时用于回归测试。

:::{tip}
理想情况下，你还应提出或勾勒一个解决方案。
完美的 issue 实际上就是一个 pull request：它直接解决问题，并用测试确保问题不会再次出现。
:::

:::{important}
请仅在你愿意且有能力自行实现时，才开 issue 请求新功能（例如软件包、模块、命令等）。
这样该 issue 可用于评估用户兴趣、判断该功能是否适合项目，并讨论实现策略。
:::

## 为 Nix 做贡献

{term}`Nix` 是生态的基石，主要用 C++ 编写。

若想协助开发，请查看 [GitHub 上 Nix 仓库中的贡献指南](https://github.com/NixOS/nix/blob/master/CONTRIBUTING.md)。

(contribute-nixpkgs)=
## 为 Nixpkgs 做贡献

:::{tip}
如需口头介绍，可观看 NixCon 2024 演讲 [Becoming a Nixpkgs Contributor](https://www.youtube.com/watch?v=eijTOBBbCv4)。
:::

{term}`Nixpkgs` 是一个大型软件项目，涵盖多个开发领域。
你可以在 [Nixpkgs issue 追踪器][nixpkgs issues] 中寻找可改进的地方。

[nixpkgs issues]: https://github.com/NixOS/nixpkgs/issues?q=is%3Aopen+is%3Aissue+-label%3A%226.topic%3A+nixos%22+-label%3A%226.topic%3A+module+system%22+-label%3A%226.+topic%3A+nixos-container%22+sort%3Areactions-%2B1-desc

若想提供帮助，请先阅读 [GitHub 上 Nixpkgs 仓库中的贡献指南](https://github.com/NixOS/nixpkgs/blob/master/CONTRIBUTING.md)，以了解代码与贡献流程概览。
另有[针对特定编程语言的说明](https://nixos.org/manual/nixpkgs/unstable/#chap-language-support)用于添加软件包。

## 为 NixOS 做贡献

{term}`NixOS` 是一个共同开发的 Linux 发行版，可通过声明式编程接口以高度灵活的方式方便地配置。
模块与默认配置的代码位于 [`nixpkgs` GitHub 仓库的 `nixos` 目录](https://github.com/NixOS/nixpkgs/tree/master/nixos)中。

请参阅 [NixOS 手册的开发章节](https://nixos.org/manual/nixos/stable/index.html#ch-development)开始改进。
针对 NixOS 的贡献者文档仍然不足，但 [Nixpkgs 贡献](contribute-nixpkgs)的大多数约定同样适用。
非常欢迎帮助改进这部分文档。

如果你是新贡献者，可查看[标记为 `good-first-bug` 的 issue](https://github.com/NixOS/nixpkgs/issues?q=is%3Aopen+label%3A%223.skill%3A+good-first-bug%22+label%3A%226.topic%3A+nixos%22)。
如果你已经比较熟悉，参与[热门 issue][nixos issues] 会深受其他 NixOS 用户的欢迎。

[nixos issues]: https://github.com/NixOS/nixpkgs/issues?q=is%3Aopen+is%3Aissue+label%3A%226.topic%3A+nixos%22+sort%3Areactions-%2B1-desc

# 如何获得帮助

如果你已准备好 pull request 并需要帮助推进，请参阅 [contributing-how-to-get-help](https://nix.dev/contributing/how-to-get-help#contributing-how-to-get-help) 了解更多信息。
