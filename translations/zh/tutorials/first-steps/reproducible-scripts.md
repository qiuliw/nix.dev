(reproducible-scripts)=

# 可复现的解释型脚本

```{contributors}
:authors: rapenne-s
:editors: fricklerhandwerk
```

在本教程中，你将学习如何用 Nix 创建并运行可复现的解释型脚本，也就是常说的 [shebang] 脚本。

## 要求

- 可用的 {ref}`Nix 安装 <install-nix>`
- 熟悉 [Bash]

## 一个依赖并不简单的简单脚本

下面的脚本会获取某个 URL 的 XML 内容，转换成 JSON，并格式化以便阅读：

```bash
#! /bin/bash

curl https://github.com/NixOS/nixpkgs/releases.atom | xml2json | jq .
```

它需要 `curl`、`xml2json` 和 `jq` 这些程序，还需要 `bash` 解释器。
如果运行脚本的系统缺少其中任一依赖，脚本会部分失败或完全失败。

借助 Nix，我们可以显式声明所有依赖，并生成一个脚本：只要机器支持 Nix，以及从 Nixpkgs 取得的所需包，它就能始终运行。

## 脚本

[shebang] 决定用哪个程序来运行解释型脚本。

[Bash]: https://www.gnu.org/software/bash/
[shebang]: https://en.wikipedia.org/wiki/Shebang_(Unix)

我们将使用 shebang 行 `#!/usr/bin/env nix-shell`。

[`env`] 是大多数现代类 Unix 操作系统在路径 `/usr/bin/env` 上提供的程序。
它接受一个命令名作为参数，并在环境变量 `$PATH` 列出的目录中运行找到的第一个同名可执行文件。

[`env`]: https://pubs.opengroup.org/onlinepubs/9699919799/utilities/env.html

我们把 [`nix-shell` 用作 shebang 解释器]。
对本用例相关的参数如下：

- `-i` 指定用哪个程序解释文件的其余部分
- `--pure` 在运行脚本时排除大多数环境变量
- `-p` 列出解释器环境中应存在的包
- `-I` 显式设置包的 [搜索路径]

更多选项细节见 [`nix-shell` 参考文档](https://nix.dev/manual/nix/stable/command-ref/nix-shell.html#options)。

[`nix-shell` 用作 shebang 解释器]: https://nix.dev/manual/nix/stable/command-ref/nix-shell.html#use-as-a--interpreter
[搜索路径]: https://nix.dev/manual/nix/stable/command-ref/opt-common.html#opt-I

创建名为 `nixpkgs-releases.sh` 的文件，内容如下：

```shell
#!/usr/bin/env nix-shell
#! nix-shell -i bash --pure
#! nix-shell -p bash cacert curl jq python3Packages.xmljson
#! nix-shell -I nixpkgs=https://github.com/NixOS/nixpkgs/archive/2a601aafdc5605a5133a2ca506a34a3a73377247.tar.gz

curl https://github.com/NixOS/nixpkgs/releases.atom | xml2json | jq .
```

第一行是标准 shebang。
后面的 shebang 行是 Nix 特有的写法：

- 通过 `-i` 选项，指定用 `bash` 解释文件的其余部分。

- 这里启用了 `--pure`，以避免脚本隐式使用运行机器上可能已存在的程序。

- `-p` 选项列出脚本运行所需的包。

  命令 `xml2json` 由包 `python3Packages.xmljson` 提供，而 `bash`、`jq` 和 `curl` 由同名包提供。
  为使 SSL 认证正常工作，还必须有 `cacert`。

  :::{tip}
  使用 [search.nixos.org](https://search.nixos.org/packages) 查找提供你所需程序的包。
  :::

- `-I` 的参数指向 Nixpkgs 仓库的某个特定 Git 提交。

  这确保脚本无论在哪里运行，都会使用完全相同的包版本。

给脚本加上可执行权限：

 ```console
 chmod +x nixpkgs-releases.sh
 ```

运行脚本：

```console
./nixpkgs-releases.sh
```

## 下一步

- {ref}`reading-nix-language`：学习用于声明包与配置的 Nix 语言。
- {ref}`declarative-reproducible-envs`：用声明式配置文件创建可复现 shell 环境。
- [垃圾回收](https://nix.dev/manual/nix/stable/package-management/garbage-collection.html) – 释放通过 Nix 提供的程序所占用的存储空间
