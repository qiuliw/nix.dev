(file-sets-tutorial)=
# 处理本地文件

```{contributors}
:authors: infinisil
```

要在 Nix derivation 中构建本地项目，源文件必须对其 [`builder` 可执行文件](https://nix.dev/manual/nix/stable/language/derivations#attr-builder) 可访问。
默认情况下，`builder` 运行在[隔离环境](https://nix.dev/manual/nix/stable/command-ref/conf-file.html#conf-sandbox)中，该环境只允许从 Nix store 读取。
Nix 语言内置了将本地文件复制到 store 并暴露所得 store 路径的功能。

不过，直接使用这些功能可能很棘手：

- 将路径强制转换为字符串，例如广泛使用的 `src = ./.` 模式，
  会使 derivation 依赖于当前目录的名称。
  此外，它总会将整个目录加入 store，包括不需要的文件，
  这些文件一变就会导致不必要的重新构建。

- [`builtins.path`](https://nix.dev/manual/nix/stable/language/builtins.html#builtins-path) 函数
  （等价于 [`lib.sources.cleanSourceWith`](https://nixos.org/manual/nixpkgs/stable/#function-library-lib.sources.cleanSourceWith)）
  可以解决这些问题。
  然而，用 `filter` 函数接口表达所需的路径选择往往很难。

在本教程中，你将学习如何使用 Nixpkgs [`lib.fileset` 库](https://nixos.org/manual/nixpkgs/stable/#sec-functions-library-fileset) 在 derivation 中处理本地文件。
它抽象了内置功能，并提供更安全、更便捷的接口。

## 文件集

_文件集_（file set）是一种表示本地文件集合的数据类型。
可以用该库的各种函数创建、组合和操作文件集。

你可以用 [`nix repl`](https://nix.dev/manual/nix/stable/command-ref/new-cli/nix3-repl) 探索和了解该库：

```shell-session
$ nix repl -f channel:nixos-23.11
...
nix-repl> fs = lib.fileset
```

[`trace`](https://nixos.org/manual/nixpkgs/stable/#function-library-lib.fileset.trace) 函数会漂亮地打印给定文件集中包含的文件：

```shell-session
nix-repl> fs.trace ./. null
trace: /home/user (all files in directory)
null
```

所有期望以文件集作为参数的函数也可以接受[路径](https://nix.dev/manual/nix/stable/language/values#type-path)。
此类路径参数会[隐式转换为文件集](https://nixos.org/manual/nixpkgs/stable/#sec-fileset-path-coercion)，包含给定路径下的_所有_文件。
在上一次 trace 中，这由 `(all files in directory)` 标明。

:::{tip}
`trace` 函数会漂亮地打印其第一个参数，并返回第二个参数。
但在 `nix repl` 中你通常只需要漂亮打印，因此可以省略第二个参数：

```shell-session
nix-repl> fs.trace ./.
trace: /home/user (all files in directory)
«lambda @ /nix/store/1czr278x24s3bl6qdnifpvm5z03wfi2p-nixpkgs-src/lib/fileset/default.nix:555:8»
```
:::

尽管文件集在概念上包含本地文件，除非显式请求，这些文件*绝不会*被加入 Nix store。
因此你不必过于担心意外将密钥复制到全局可读的 store 中。

在本例中，虽然你漂亮地打印了主目录，但没有复制任何文件。
这与诸如 `"${./.}"` 这样的路径到字符串强制转换形成对比，
后者会在求值时将整个目录复制到 Nix store。

:::{warning}
在使用 [`flakes` 和 `nix-command` 实验性功能](https://nix.dev/manual/nix/stable/command-ref/new-cli/nix3-flake)时，
除非它是 Git 仓库，否则 Flake 内的本地目录总会被*完整*复制到 Nix store。
:::

这种隐式强制转换对文件同样有效：

```shell-session
$ touch some-file
```

```shell-session
nix-repl> fs.trace ./some-file
trace: /home/user
trace: - some-file (regular)
```

除了包含的文件外，这还会打印其[文件类型](https://nix.dev/manual/nix/stable/language/builtins.html#builtins-readFileType)。


## 示例项目

为了进一步试验该库，先做一个示例项目。
创建新目录，进入其中，并用 `npins` 固定 Nixpkgs 依赖：

```shell-session
$ mkdir fileset
$ cd fileset
$ nix-shell -p npins --run "npins init --bare; npins add github nixos nixpkgs --branch nixos-23.11"
```

然后创建内容如下的 `default.nix` 文件：

```{code-block} nix
:caption: default.nix
{
  system ? builtins.currentSystem,
  sources ? import ./npins,
}:
let
  pkgs = import sources.nixpkgs {
    config = { };
    overlays = [ ];
    inherit system;
  };
in
pkgs.callPackage ./package.nix { }
```

再添加两个要处理的源文件：

```shell-session
$ echo hello > hello.txt
$ echo world > world.txt
```

## 将文件加入 Nix store

给定文件集中的文件可用 [`toSource`](https://nixos.org/manual/nixpkgs/stable/#function-library-lib.fileset.toSource) 加入 Nix store。
该函数的参数需要 `root` 属性，以确定要将哪个源目录复制到 store。
结果中只包含 `fileset` 属性中的文件。

如下定义 `package.nix`：

```{code-block} nix
:caption: package.nix
{ stdenv, lib }:
let
  fs = lib.fileset;
  sourceFiles = ./hello.txt;
in

fs.trace sourceFiles

stdenv.mkDerivation {
  name = "fileset";
  src = fs.toSource {
    root = ./.;
    fileset = sourceFiles;
  };
  postInstall = ''
    mkdir $out
    cp -v hello.txt $out
  '';
}
```

对 `fs.trace` 的调用会打印将用作 derivation 输入的文件集。

试着构建它：

:::{note}
首次获取 Nixpkgs 会花费一段时间。
:::

```
$ nix-build
trace: /home/user/fileset
trace: - hello.txt (regular)
this derivation will be built:
  /nix/store/3ci6avmjaijx5g8jhb218i183xi7bi2n-fileset.drv
...
'hello.txt' -> '/nix/store/sa4g6h13v0zbpfw6pzva860kp5aks44n-fileset/hello.txt'
...
/nix/store/sa4g6h13v0zbpfw6pzva860kp5aks44n-fileset
```

但文件集库的真正优势在于它以不同方式组合文件集的能力。

## 差集

为了将 `hello.txt` 和 `world.txt` 都复制到输出中，再次把整个项目目录作为源加入：

```{code-block} diff
:caption: package.nix
 { stdenv, lib }:
 let
   fs = lib.fileset;
-  sourceFiles = ./hello.txt;
+  sourceFiles = ./.;
 in

 fs.trace sourceFiles

 stdenv.mkDerivation {
   name = "fileset";
   src = fs.toSource {
     root = ./.;
     fileset = sourceFiles;
   };
   postInstall = ''
     mkdir $out
-    cp -v hello.txt $out
+    cp -v {hello,world}.txt $out
   '';
 }
```

这会按预期工作：

```shell-session
$ nix-build
trace: /home/user/fileset (all files in directory)
this derivation will be built:
  /nix/store/fsihp8872vv9ngbkc7si5jcbigs81727-fileset.drv
...
'hello.txt' -> '/nix/store/wmsxfgbylagmf033nkazr3qfc96y7mwk-fileset/hello.txt'
'world.txt' -> '/nix/store/wmsxfgbylagmf033nkazr3qfc96y7mwk-fileset/world.txt'
...
/nix/store/wmsxfgbylagmf033nkazr3qfc96y7mwk-fileset
```

然而，如果再次运行 `nix-build`，输出路径会不同！

```shell-session
$ nix-build
trace: /home/user/fileset (all files in directory)
this derivation will be built:
  /nix/store/nlh7ismrf27xsnl3m20vfz6rvwlbbbca-fileset.drv
...
'hello.txt' -> '/nix/store/xknflcvjaa8dj6a6vkg629zmcrgz10rh-fileset/hello.txt'
'world.txt' -> '/nix/store/xknflcvjaa8dj6a6vkg629zmcrgz10rh-fileset/world.txt'
...
/nix/store/xknflcvjaa8dj6a6vkg629zmcrgz10rh-fileset
```

问题在于：`nix-build` 默认会在工作目录中创建指向刚生成的 store 路径的 `result` 符号链接：

```
$ ls -l result
result -> /nix/store/xknflcvjaa8dj6a6vkg629zmcrgz10rh-fileset
```

由于 `src` 引用整个目录，而该目录的内容在 `nix-build` 成功后会改变，Nix 每次都得重新开始。

:::{note}
不使用文件集库时也会发生这种情况，例如直接设置 `src = ./.;`。
:::

[`difference`](https://nixos.org/manual/nixpkgs/stable/#function-library-lib.fileset.difference) 函数从一个文件集中减去另一个。
结果是一个新文件集，包含第一个参数中所有不在第二个参数中的文件。

用它过滤掉 `./result`，修改 `sourceFiles` 的定义：

```{code-block} diff
:caption: package.nix
 { stdenv, lib }:
 let
   fs = lib.fileset;
-  sourceFiles = ./.;
+  sourceFiles = fs.difference ./. ./result;
 in
```

构建时，文件集库会标明从该目录取用了哪些文件：

```shell-session
$ nix-build
trace: /home/user/fileset
trace: - package.nix (regular)
trace: - default.nix (regular)
trace: - hello.txt (regular)
trace: - npins (all files in directory)
trace: - world.txt (regular)
this derivation will be built:
  /nix/store/zr19bv51085zz005yk7pw4s9sglmafvn-fileset.drv
...
'hello.txt' -> '/nix/store/vhyhk6ij39gjapqavz1j1x3zbiy3qc1a-fileset/hello.txt'
'world.txt' -> '/nix/store/vhyhk6ij39gjapqavz1j1x3zbiy3qc1a-fileset/world.txt'
...
/nix/store/vhyhk6ij39gjapqavz1j1x3zbiy3qc1a-fileset
```

再次尝试构建会复用已有的 store 路径：

```
$ nix-build
trace: /home/user/fileset
trace: - package.nix (regular)
trace: - default.nix (regular)
trace: - hello.txt (regular)
trace: - npins (all files in directory)
trace: - world.txt (regular)
/nix/store/vhyhk6ij39gjapqavz1j1x3zbiy3qc1a-fileset
```

## 缺失的文件

不过，删除 `./result` 符号链接会带来新问题：

```shell-session
$ rm result
$ nix-build
error: lib.fileset.difference: Second argument (negative set)
  (/home/user/fileset/result) is a path that does not exist.
  To create a file set from a path that may not exist, use `lib.fileset.maybeMissing`.
```

按错误信息中的说明，使用 [`maybeMissing`](https://nixos.org/manual/nixpkgs/stable/#function-library-lib.fileset.maybeMissing) 从可能不存在的路径创建文件集（若不存在则文件集为空）：

```{code-block} diff
:caption: package.nix
 { stdenv, lib }:
 let
   fs = lib.fileset;
-  sourceFiles = fs.difference ./. ./result;
+  sourceFiles = fs.difference ./. (fs.maybeMissing ./result);
 in
```

现在可以工作了，由于 `./result` 不存在，会使用整个目录：

```
$ nix-build
trace: /home/user/fileset (all files in directory)
this derivation will be built:
  /nix/store/zr19bv51085zz005yk7pw4s9sglmafvn-fileset.drv
...
/nix/store/vhyhk6ij39gjapqavz1j1x3zbiy3qc1a-fileset
```

再次构建会产生不同的 trace，但输出路径相同：

```
$ nix-build
trace: /home/user/fileset
trace: - package.nix (regular)
trace: - default.nix (regular)
trace: - hello.txt (regular)
trace: - npins (all files in directory)
trace: - world.txt (regular)
/nix/store/vhyhk6ij39gjapqavz1j1x3zbiy3qc1a-fileset
```

## 并集（显式排除文件）

仍有一个问题：
更改_任意_已包含文件都会导致 derivation 重新构建，即使它并不依赖那些文件。

在 `package.nix` 末尾追加一个空行：

```shell-session
$ echo >> package.nix
```

Nix 又会从头开始：

```shell-session
$ nix-build
trace: /home/user/fileset
trace: - default.nix (regular)
trace: - npins (all files in directory)
trace: - package.nix (regular)
trace: - string.txt (regular)
this derivation will be built:
  /nix/store/zmgpqlpfz2jq0w9rdacsnpx8ni4n77cn-filesets.drv
...
/nix/store/6pffjljjy3c7kla60nljk3fad4q4kkzn-filesets
```

一种修复方式是使用 [`unions`](https://nixos.org/manual/nixpkgs/stable/#function-library-lib.fileset.unions)。

创建一个包含要排除文件并集的文件集（`fs.unions [ ... ]`），再从完整目录（`./.`）中减去它（`difference`）：

```{code-block} nix
:caption: package.nix
  sourceFiles =
    fs.difference
      ./.
      (fs.unions [
        (fs.maybeMissing ./result)
        ./default.nix
        ./package.nix
        ./npins
      ]);
```

这会按预期工作：

```
$ nix-build
trace: /home/user/fileset
trace: - hello.txt (regular)
trace: - world.txt (regular)
this derivation will be built:
  /nix/store/gr2hw3gdjc28fmv0as1ikpj7lya4r51f-fileset.drv
...
/nix/store/ckn40y7hgqphhbhyrq64h9r6rvdh973r-fileset
```

现在更改任何已排除的文件不一定会触发新构建：

```
$ echo >> package.nix
```

```
$ nix-build
trace: /home/user/fileset
trace: - hello.txt (regular)
trace: - world.txt (regular)
/nix/store/ckn40y7hgqphhbhyrq64h9r6rvdh973r-fileset
```

## 过滤

[`fileFilter`](https://nixos.org/manual/nixpkgs/stable/#function-library-lib.fileset.fileFilter) 函数允许过滤文件集，使每个包含的文件都满足给定条件。

用它选择所有名称以 `.nix` 结尾的文件：

```{code-block} diff
:caption: package.nix
   sourceFiles =
     fs.difference
       ./.
       (fs.unions [
         (fs.maybeMissing ./result)
-        ./default.nix
-        ./package.nix
+        (fs.fileFilter (file: file.hasExt "nix") ./.)
         ./npins
       ]);
```

即使我们添加一个新的 `.nix` 文件，结果也不会改变。

```shell-session
$ nix-build
trace: /home/user/fileset
trace: - hello.txt (regular)
trace: - world.txt (regular)
/nix/store/ckn40y7hgqphhbhyrq64h9r6rvdh973r-fileset
```

值得注意的是，使用 `difference ./.` 的方式是显式选择要_排除_的文件，这意味着源目录中新增的文件默认会被包含。
取决于你的项目，这可能比下一节的替代方案更合适。

## 并集（显式包含文件）

与上一种做法相对，`unions` 也可用于只选择要_包含_的文件。
这意味着当前目录中新增的文件默认会被忽略。

创建一些额外文件：

```shell-session
$ mkdir src
$ touch build.sh src/select.{c,h}
```

然后显式地只从要包含的文件创建文件集：

```{code-block} nix
:caption: package.nix
{ stdenv, lib }:
let
  fs = lib.fileset;
  sourceFiles = fs.unions [
    ./hello.txt
    ./world.txt
    ./build.sh
    (fs.fileFilter
      (file: file.hasExt "c" || file.hasExt "h")
      ./src
    )
  ];
in

fs.trace sourceFiles

stdenv.mkDerivation {
  name = "fileset";
  src = fs.toSource {
    root = ./.;
    fileset = sourceFiles;
  };
  postInstall = ''
    cp -vr . $out
  '';
}
```

`postInstall` 脚本被简化了，依赖源文件已被适当预先过滤：

```shell-session
$ nix-build
trace: /home/user/fileset
trace: - build.sh (regular)
trace: - hello.txt (regular)
trace: - src (all files in directory)
trace: - world.txt (regular)
this derivation will be built:
  /nix/store/sjzkn07d6a4qfp60p6dc64pzvmmdafff-fileset.drv
...
'.' -> '/nix/store/zl4n1g6is4cmsqf02dci5b2h5zd0ia4r-fileset'
'./build.sh' -> '/nix/store/zl4n1g6is4cmsqf02dci5b2h5zd0ia4r-fileset/build.sh'
'./hello.txt' -> '/nix/store/zl4n1g6is4cmsqf02dci5b2h5zd0ia4r-fileset/hello.txt'
'./world.txt' -> '/nix/store/zl4n1g6is4cmsqf02dci5b2h5zd0ia4r-fileset/world.txt'
'./src' -> '/nix/store/zl4n1g6is4cmsqf02dci5b2h5zd0ia4r-fileset/src'
'./src/select.c' -> '/nix/store/zl4n1g6is4cmsqf02dci5b2h5zd0ia4r-fileset/src/select.c'
'./src/select.h' -> '/nix/store/zl4n1g6is4cmsqf02dci5b2h5zd0ia4r-fileset/src/select.h'
...
/nix/store/zl4n1g6is4cmsqf02dci5b2h5zd0ia4r-fileset
```

即使新增文件，也只会使用指定的文件：

```shell-session
$ touch src/select.o README.md

$ nix-build
trace: - build.sh (regular)
trace: - hello.txt (regular)
trace: - src
trace:   - select.c (regular)
trace:   - select.h (regular)
trace: - world.txt (regular)
/nix/store/zl4n1g6is4cmsqf02dci5b2h5zd0ia4r-fileset
```

## 匹配 Git 跟踪的文件

如果目录属于 Git 仓库，将其传给 [`gitTracked`](https://nixos.org/manual/nixpkgs/stable/#function-library-lib.fileset.gitTracked) 会得到一个只包含 Git 所跟踪文件的文件集。

创建本地 Git 仓库，并将除 `src/select.o` 和 `./result` 外的所有文件加入其中：

```shell-session
$ git init
Initialized empty Git repository in /home/user/fileset/.git/
$ git add -A
$ git reset src/select.o result
```

用 `gitTracked` 复用这一文件选择：

```{code-block} nix
:caption: package.nix
  sourceFiles = fs.gitTracked ./.;
```

再次构建：

```shell-session
$ nix-build
warning: Git tree '/home/user/fileset' is dirty
trace: /home/vg/src/nix.dev/fileset
trace: - README.md (regular)
trace: - package.nix (regular)
trace: - build.sh (regular)
trace: - default.nix (regular)
trace: - hello.txt (regular)
trace: - npins (all files in directory)
trace: - src
trace:   - select.c (regular)
trace:   - select.h (regular)
trace: - world.txt (regular)
this derivation will be built:
  /nix/store/p9aw3fl5xcjbgg9yagykywvskzgrmk5y-fileset.drv
...
/nix/store/cw4bza1r27iimzrdbfl4yn5xr36d6k5l-fileset
```

不过这包含得太多了，因为并非所有这些文件都是按原先意图构建 derivation 所需要的。

:::{note}
在使用 [`flakes` 和 `nix-command` 实验性功能](https://nix.dev/manual/nix/stable/command-ref/new-cli/nix3-flake)时，
不需要此函数，因为 `nix build` 默认只允许访问 Git 跟踪的文件。
不过，为了给稳定版 Nix 提供相同的开发者体验，仍建议使用此函数。
:::

## 交集

这时就轮到 `intersection` 出场了。
它允许创建一个只包含同时存在于_两个_给定文件集中的文件的文件集。

选择既被 Git 跟踪*又*与构建相关的所有文件：

```{code-block} nix
:caption: package.nix
  sourceFiles =
    fs.intersection
      (fs.gitTracked ./.)
      (fs.unions [
        ./hello.txt
        ./world.txt
        ./build.sh
        ./src
      ]);
```

这会得到与另一种做法相同的输出，因此会复用之前的构建结果：

```shell-session
$ nix-build
warning: Git tree '/home/user/fileset' is dirty
trace: - build.sh (regular)
trace: - hello.txt (regular)
trace: - src
trace:   - select.c (regular)
trace:   - select.h (regular)
trace: - world.txt (regular)
/nix/store/zl4n1g6is4cmsqf02dci5b2h5zd0ia4r-fileset
```

## 小结

我们展示了如何使用所有基本文件集函数的一些示例。
对于更复杂的用例，可以按需组合它们。

完整列表与更多细节，见 [`lib.fileset` 参考文档](https://nixos.org/manual/nixpkgs/stable/#sec-functions-library-fileset)。
