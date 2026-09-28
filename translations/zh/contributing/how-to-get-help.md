(contributing-how-to-get-help)=
# 如何获得帮助

如果你在某次贡献中需要协助，有几个地方可以寻求帮助。

## 如何找到维护者

为提高效率并增加成功几率，应先尝试联系具备更具体知识的个人或团队：

- 如果你的贡献针对 Nixpkgs 中的某个软件包，请在其
  [`maintainers`](https://nixos.org/manual/nixpkgs/stable/#var-meta-maintainers)
  属性中查找维护者。
- 检查是否有团队负责相关子系统：
  - 在 [NixOS 网站](https://nixos.org/community/#governance-teams)上。
  - 在 [Nixpkgs 维护者团队列表](https://github.com/NixOS/nixpkgs/blob/master/maintainers/team-list.nix)中。
  - 在 [Nixpkgs](https://github.com/NixOS/nixpkgs/blob/master/ci/OWNERS) 或
    [Nix](https://github.com/NixOS/nix/blob/master/.github/CODEOWNERS) 的 `CODEOWNERS` 文件中。
- 对你需要帮助的文件检查 [`git blame`](https://git-scm.com/docs/git-blame) 或 [`git log`](https://www.git-scm.com/docs/git-log) 的输出。
  记下提交相关代码的人的电子邮件地址。

## 应使用哪些交流渠道

找到要联系的人后，可通过[社区交流平台](https://nixos.org/community)之一与他们联系：

- [GitHub](https://github.com/nixos)

  所有源代码都维护在 GitHub 上。
  这里适合讨论实现细节。

  在 issue 评论或 pull request 描述中，[提及](https://docs.github.com/en/get-started/writing-on-github/getting-started-with-writing-and-formatting-on-github/basic-writing-and-formatting-syntax#mentioning-people-and-teams)在 [`maintainers-list.nix` 文件](https://github.com/NixOS/nixpkgs/blob/master/maintainers/maintainer-list.nix)中找到的 GitHub 用户名。

- [Discourse](https://discourse.nixos.org)

  Discourse 用于公告、协调与开放性问题。

  可尝试用 [`maintainers-list.nix` 文件][maintainers-list]中找到的 GitHub 用户名来提及或直接联系特定用户。
  请注意，有些人在 Discourse 上使用不同的用户名。

- [Matrix]

  Matrix 用于短暂、及时的交流以及私信。

  要联系维护者，请使用其在 [`maintainers-list.nix` 文件][maintainers-list]中找到的 Matrix 句柄。
  若某位维护者没有 Matrix 句柄，可尝试搜索其 GitHub 用户名，因为大多数人倾向于跨渠道使用相同用户名。

  维护者团队有时会有自己的公开 Matrix 房间。

- 电子邮件

  使用通过 `git log` 找到的电子邮件地址。

- 会议与活动

  查看[官方 NixOS 日历](https://calendar.google.com/calendar/u/0/embed?src=b9o52fobqjak8oq8lfkhg3t0qg@group.calendar.google.com)与 [Discourse 社区日历](https://discourse.nixos.org/t/community-calendar/18589)，了解实时或线下活动。
  部分社区团队会定期开会并发布会议纪要。

## 其他渠道

如果找不到能帮助你贡献的特定用户或团队，可以求助于更广泛的社区，使用以下官方交流渠道之一：

- [NixOS Matrix space][matrix] 中与你问题相关的房间。
- Discourse 上的 [*Help* 分类](https://discourse.nixos.org/c/learn/9)。
- Matrix 上的通用 [`#nix`](https://matrix.to/#/#nix:nixos.org) 房间。

[matrix]: https://matrix.to/#/#community:nixos.org
[maintainers-list]: https://github.com/NixOS/nixpkgs/blob/master/maintainers/maintainer-list.nix
