(reading-nix-language)=


# Nix 语言基础

```{contributors}
:authors: fricklerhandwerk
:editors: infinisil
```

Nix 语言旨在方便地创建与组合 *derivation*——精确描述如何用已有文件的内容派生出新文件。
它是一门领域特定、纯函数式、惰性求值、动态类型的编程语言。

:::{admonition} Nix 语言的主要用途
:class: note


- {term}`Nixpkgs`

  世界上规模最大、更新最及时的软件发行版，用 Nix 语言编写。

- {term}`NixOS`

  基于 Nix 与 Nixpkgs、可完全声明式配置的 Linux 发行版。

  其底层模块化配置系统用 Nix 语言编写，并使用 Nixpkgs 中的包。
  它提供的操作系统环境与服务也用 Nix 语言配置。

:::

你可能很快就会遇到看起来非常复杂的 Nix 语言表达式。
与任何编程语言一样，所需的 Nix 语言代码量与要解决的问题复杂度密切相关，也反映了对问题及其解法的理解程度。
构建软件本身就很复杂，而 Nix 既 *暴露* 也 *允许管理* 这种复杂性——借助 Nix 语言。

不过，Nix 语言本身只有本教程将介绍的少数基本概念，它们可以任意组合。
看起来复杂的部分往往不来自语言本身，而来自其用法。

## 概览

本教程介绍如何 **阅读 Nix 语言**，以便跟进其他教程与示例。

在实践中 **使用 Nix 语言** 涉及多方面：

- 语言：语法与语义
- 库：`builtins` 与 `pkgs.lib`
- 开发者工具：测试、调试、代码检查、格式化……
- 通用构建机制：`stdenv.mkDerivation`、构建辅助函数……
- 组合与配置机制：`override`、`overrideAttrs`、overlays、`callPackage`……
- 生态特定的打包机制：`buildGoModule`、`buildPythonApplication`……
- NixOS 模块系统：`config`、`option`……

本教程只覆盖最重要的语言特性，简要讨论库，并在结尾指向其他组件的参考资料与资源。

### 你将学到什么？

本教程应能帮助你阅读典型的 Nix 语言代码并理解其结构。
目标是突出 Nix 语言与你可能熟悉的语言之间的差异。

因此，本教程展示 Nix 语言中最常见、最具辨识度的模式：

- [命名与访问值](names-values)
- 声明与调用 [函数](functions)
- [内置函数与库函数](libraries)
- 用于获取构建输入的 [不纯操作](impurities)
- 描述构建任务的 [Derivations](derivations)

:::{important}
本教程 *不会* 详细解释全部 Nix 语言特性，也 *不会* 深入具体语法规则。
例如，本教程会跳过诸如 `if ... then ... else ...` 这类常见构造。

完整语言参考见 [Nix 手册][manual-language]。
:::

[manual-language]: https://nix.dev/manual/nix/stable/language/index.html

### 你需要什么？

- 熟悉软件开发
- 熟悉 Unix shell，以便阅读命令行示例 <!-- TODO: link to yet-to-be instructions on "how to read command line examples" -->
- 已 {ref}`安装 Nix <install-nix>`，以便运行示例

### 需要多长时间？

- 没有函数式编程经验：2 小时
- 熟悉函数式编程：1 小时
- 精通函数式编程：30 分钟

请运行所有示例。
动手试验，验证你的假设，检验所学。
若希望确认自己完全理解示例，请阅读详细说明。

### 如何运行示例？

- 一段 Nix 语言代码是一个 *Nix 表达式*。
- 对 Nix 表达式求值会得到一个 *Nix 值*。
- *Nix 文件*（扩展名 `.nix`）的内容是一个 Nix 表达式。

:::{note}
*求值* 指根据语言规则把表达式变换成值。
:::

本教程包含许多 Nix 表达式示例。
每个示例后都附有预期的求值结果。

下面的示例是一个将两个数相加的 Nix 表达式：

```{code-block} nix
:class: expression
1 + 2
```

```{code-block}
:class: value
3
```

#### 交互式求值

使用 [`nix repl`] 交互式地对 Nix 表达式求值（在命令行中输入）：

```shell-session
$ nix repl
Welcome to Nix 2.13.3. Type :? for help.

nix-repl> 1 + 2
3
```

:::{note}
Nix 语言使用惰性求值，`nix repl` 默认只在需要时计算值。

为了清晰，部分示例会展示完全求值后的数据结构。
若你的输出与示例不符，可尝试在输入表达式前加上 `:p`。

示例：

```shell-session
nix-repl> { a.b.c = 1; }
{ a = { ... }; }

nix-repl> :p { a.b.c = 1; }
{ a = { b = { c = 1; }; }; }
```

输入 `:q` 退出 [`nix repl`]。

:::

[`nix repl`]: https://nix.dev/manual/nix/stable/command-ref/new-cli/nix3-repl.html

#### 对 Nix 文件求值

使用 [`nix-instantiate --eval`][nix-instantiate] 对 Nix 文件中的表达式求值。

```shell-session
$ echo 1 + 2 > file.nix
$ nix-instantiate --eval file.nix
3
```

:::{dropdown} 详细说明

第一条命令把 `1 + 2` 写入当前目录的 `file.nix`。
此时 `file.nix` 的内容是 `1 + 2`，可用下列命令确认：

```shell-session
$ cat file.nix
1 + 2
```

第二条命令对 `file.nix` 运行带 `--eval` 选项的 `nix-instantiate`，读取文件并对其中的 Nix 表达式求值。
结果值会打印为输出。

`--eval` 是必需的，以便只对文件求值而不做其他事。
若省略 `--eval`，`nix-instantiate` 期望给定文件中的表达式求值为一种称为 *derivation* 的特殊值，本教程末尾 [](derivations) 会介绍。

:::

:::{note}
若未指定文件名，`nix-instantiate --eval` 会尝试读取 `default.nix`。

```shell-session
$ echo 1 + 2 > default.nix
$ nix-instantiate --eval
3
```
:::

:::{note}
Nix 语言使用惰性求值，`nix-instantiate` 默认只在需要时计算值。

为了清晰，部分示例会展示完全求值后的数据结构。
若你的输出与示例不符，可尝试给 `nix-instantiate` 加上 `--strict` 选项。

示例：

```shell-session
$ echo "{ a.b.c = 1; }" > file.nix
$ nix-instantiate --eval file.nix
{ a = <CODE>; }
```

```shell-session
$ echo "{ a.b.c = 1; }" > file.nix
$ nix-instantiate --eval --strict file.nix
{ a = { b = { c = 1; }; }; }
```

:::

[nix-instantiate]: https://nix.dev/manual/nix/stable/command-ref/nix-instantiate.html

### 关于空白

空白用于在需要时分隔 [词法记号]。
除此之外空白无关紧要。

[词法记号]: https://en.wikipedia.org/wiki/Lexical_analysis#Lexical_token_and_lexical_tokenization

换行、缩进与额外空格是为了便于阅读。

下面两段等价：

```{code-block} nix
:class: expression
let
 x = 1;
 y = 2;
in x + y
```

```{code-block}
:class: value
3
```

```{code-block} nix
:class: expression
let x=1;y=2;in x+y
```

```{code-block}
:class: value
3
```

(names-values)=
## 名称与值

Nix 语言中的值可以是原始数据类型、列表、属性集与函数。

原始数据类型与列表的示例会出现在 [属性集](attrset) 的上下文中。
本节稍后会介绍字符串的特殊功能：[字符串插值](string-interpolation)、[文件系统路径](file-system-paths)，以及 [缩进字符串](indented-strings)。
[函数](functions) 单独介绍。

[属性集](attrset) 与 [`let` 表达式](let) 用于给值赋名。
赋值用单个等号（`=`）表示。

在 Nix 语言代码中遇到等号（`=`）时：
- 左侧是被赋的名称。
- 右侧是值，以分号（`;`）结束。

(attrset)=
### 属性集 `{ ... }`

属性集是一组名-值对，其中名称必须唯一。

下面的示例展示了全部原始数据类型、列表与属性集。

:::{note}
若你熟悉 JSON，可以把 Nix 语言想象成 *带有函数的 JSON*。

*不含函数* 的 Nix 语言数据类型与 JSON 中的对应物工作方式相同，外观也很相似。
:::


::::{grid} 2

:::{grid-item} **Nix**
```nix
{
  string = "hello";
  integer = 1;
  float = 3.141;
  bool = true;
  null = null;
  list = [ 1 "two" false ];
  attribute-set = {
    a = "hello";
    b = 2;
    c = 2.718;
    d = false;
  }; # comments are supported
}
```
:::

:::{grid-item} **JSON**
```json
{
  "string": "hello",
  "integer": 1,
  "float": 3.141,
  "bool": true,
  "null": null,
  "list": [1, "two", false],
  "object": {
    "a": "hello",
    "b": 1,
    "c": 2.718,
    "d": false
  }
}
```
:::

::::

:::{note}
- [属性集语法](https://nix.dev/manual/nix/stable/language/syntax#attrs-literal)：属性名通常不需要引号
- [列表语法](https://nix.dev/manual/nix/stable/language/syntax#list-literal)：列表元素以空白分隔
:::



(rec-attrset)=
#### 递归属性集 `rec { ... }`

有时你会看到属性集声明前带有 `rec`。
这允许在集合内部访问其属性。

示例：

```{code-block} nix
:class: expression
rec {
  one = 1;
  two = one + 1;
  three = two + 1;
}
```

```{code-block}
:class: value
{ one = 1; three = 3; two = 2; }
```

:::{note}
属性集中的元素可以按任意顺序声明，求值时会排序。
:::

反例：

```{code-block} nix
:class: expression
{
  one = 1;
  two = one + 1;
  three = two + 1;
}
```

```{code-block}
:class: value
error: undefined variable 'one'

       at «string»:3:9:

            2|   one = 1;
            3|   two = one + 1;
             |         ^
            4|   three = two + 1;
```

(let)=
### `let ... in ...`

也称为「`let` 表达式」或「`let` 绑定」

`let` 表达式允许给值赋名以便重复使用。

示例：

```{code-block} nix
:class: expression
let
  a = 1;
in
a + a
```

```{code-block}
:class: value
2
```

:::{dropdown} 详细说明

赋值写在关键字 `let` 与 `in` 之间。
本例中我们赋值 `a = 1`。

`in` 之后是这些赋值生效的表达式，也就是可以使用已赋名称的地方。
本例中表达式是 `a + a`，其中 `a` 指的是 `a = 1`。

用所赋的值替换名称后，`a + a` 求值为 `2`。

:::

名称可以按任意顺序赋值，赋值（`=`）右侧的表达式可以引用其他已赋的名称。

示例：

```{code-block} nix
:class: expression
let
  b = a + 1;
  a = 1;
in
a + b
```

```{code-block}
:class: value
3
```

:::::{dropdown} 详细说明

赋值写在关键字 `let` 与 `in` 之间。
本例中我们赋值 `a = 1` 与 `b = a + 1`。

赋值顺序无关紧要。
因此下面把赋值顺序颠倒的示例是等价的：


```{code-block} nix
:class: expression
let
  a = 1;
  b = a + 1;
in
a + b
```

```{code-block}
:class: value
3
```

注意 `b = a + 1` 中的 `a` 指的是 `a = 1`。

`in` 之后是这些赋值生效的表达式。
本例中表达式是 `a + b`，其中 `a` 指 `a = 1`，`b` 指 `b = a + 1`。

用所赋的值替换名称后，`a + b` 求值为 `3`。

这与 [递归属性集](rec-attrset) 类似：
两者中赋值顺序都无关紧要，左侧的名称都可以用在赋值（`=`）右侧的表达式中。

示例：

::::{grid} 2

:::{grid-item} `let ... in ...`

```{code-block} nix
:class: expression
let
  b = a + 1;
  c = a + b;
  a = 1;
in {  c = c; a = a; b = b; }
```

```{code-block}
:class: value
{ a = 1; b = 2; c = 3; }
```

:::

:::{grid-item} `rec { ... }`

```{code-block} nix
:class: expression
rec {
  b = a + 1;
  c = a + b;
  a = 1;
}
```

```{code-block}
:class: value
{ a = 1; b = 2; c = 3; }
```

:::

::::

区别在于：递归属性集求值为 [属性集](attrset)，而 `in` 关键字之后可以跟任意表达式。

下面的示例中我们用 `let` 表达式构造一个列表：

```{code-block} nix
:class: expression
let
  b = a + 1;
  c = a + b;
  a = 1;
in [ a b c ]
```

```{code-block}
:class: value
[ 1 2 3 ]
```

:::::

只有 `let` 表达式自身内部的表达式才能访问新声明的名称。
这些绑定具有局部作用域。

反例：

```{code-block} nix
:class: expression
{
  a = let x = 1; in x;
  b = x;
}
```

```{code-block}
:class: value
error: undefined variable 'x'

       at «string»:3:7:

            2|   a = let x = 1; in x;
            3|   b = x;
             |       ^
            4| }
```

<!-- TODO: exercise - use let to reuse a value in an attribute set -->

### 属性访问

集合中的属性用点（`.`）与属性名访问。

示例：

```{code-block} nix
:class: expression
let
  attrset = { x = 1; };
in
attrset.x
```

```{code-block}
:class: value
1
```

访问嵌套属性的方式相同。

示例：

```{code-block} nix
:class: expression
let
  attrset = { a = { b = { c = 1; }; }; };
in
attrset.a.b.c
```

```{code-block}
:class: value
1
```

点（`.`）记法也可用于赋值属性。

示例：

```{code-block} nix
:class: expression
{ a.b.c = 1; }
```

```{code-block}
:class: value
{ a = { b = { c = 1; }; }; }
```

(with)=
### `with ...; ...`

`with` 表达式允许访问属性而无需反复引用其属性集。

示例：

```{code-block} nix
:class: expression
let
  a = {
    x = 1;
    y = 2;
    z = 3;
  };
in
with a; [ x y z ]
```

```{code-block}
:class: value
[ 1 2 3 ]
```

表达式

```{code-block} nix
with a; [ x y z ]
```

等价于

```{code-block} nix
[ a.x a.y a.z ]
```

通过 `with` 引入的属性只在分号（`;`）之后的表达式作用域内有效。

反例：

```{code-block} nix
:class: expression
let
  a = {
    x = 1;
    y = 2;
    z = 3;
  };
in
{
  b = with a; [ x y z ];
  c = x;
}
```

```{code-block}
:class: value
error: undefined variable 'x'

       at «string»:10:7:

            9|   b = with a; [ x y z ];
           10|   c = x;
             |       ^
           11| }
```

(inherit)=
### `inherit ...`

`inherit` 是把现有作用域中某个名称的值赋给嵌套作用域中同名属性的简写。
它便于避免多次重复同一名称。

示例：

```{code-block} nix
:class: expression
let
  x = 1;
  y = 2;
in
{
  inherit x y;
}
```

```{code-block}
:class: value
{ x = 1; y = 2; }
```

片段

```{code-block} nix
inherit x y;
```
等价于

```{code-block} nix
x = x; y = y;
```

(inherit-from-attrset)=
### `inherit (...) ...`

也可以通过把属性集名称写在括号中，从特定属性集 `inherit` 名称。

示例：

```{code-block} nix
:class: expression
let
  a = { x = 1; y = 2; };
in
{
  inherit (a) x y;
}
```

```{code-block}
:class: value
{ x = 1; y = 2; }
```

片段

```{code-block} nix
inherit (a) x y;
```

等价于

```{code-block} nix
x = a.x; y = a.y;
```

`inherit` 也可用于 `let` 表达式内部。

示例：

```{code-block} nix
:class: expression
let
  a = { x = 1; y = 2; };
  inherit (a) x y;
in [ x y ]
```

```{code-block}
:class: value
[ 1 2 ]
```

:::{dropdown} 详细说明

虽然本例有些刻意，但在更复杂的代码中，你经常会看到嵌套的 [`let` 表达式](let) 从外层作用域复用名称。

这里我们用属性集 `a = { x = 1; y = 2; }` 作为有意义的 inherit 来源。
`let` 表达式用 `( )` 从 `a` inherit `x` 与 `y`，等价于写作：

```{code-block} nix
let
  x = a.x;
  y = a.y;
in
```

新的内层作用域现在包含 `x` 与 `y`，它们用在列表 `[ x y ]` 中。

:::

(string-interpolation)=
### 字符串插值 `${ ... }`

以前称为「antiquotation」。

Nix 表达式的值可以用美元符号与花括号（`${ }`）插入到字符串中。

示例：

```{code-block} nix
:class: expression
let
  name = "Nix";
in
"hello ${name}"
```

```{code-block}
:class: value
"hello Nix"
```

只允许字符串，或可以表示为字符串的值。

反例：

```{code-block} nix
:class: expression
let
  x = 1;
in
"${x} + ${x} = ${x + x}"
```

```{code-block}
:class: value
error: cannot coerce an integer to a string

       at «string»:4:2:

            3| in
            4| "${x} + ${x} = ${x + x}"
             |  ^
            5|
```

插值表达式可以任意嵌套。

（这会变得很难读。实践中请避免。）

示例：

```{code-block} nix
:class: expression
let
  a = "no";
in
"${a + " ${a + " ${a}"}"}"
```

```{code-block}
:class: value
"no no no"
```

:::{dropdown} 详细说明
任何值可以表示为字符串的 Nix 表达式都可以用在 `${ }` 中。

上面表达式中的 `+` 是 [字符串连接运算符](https://nix.dev/manual/nix/latest/language/operators#string-concatenation)，它接受两个字符串并产生一个新字符串。

示例中的表达式故意写得很绕，以说明任意嵌套的字符串插值是可行的，但往往很难阅读。

它表示一个字符串，其中插值的是：把 `a` 的值与另一个以空格开头并再含一段插值字符串的字符串相连接。
那第二段插值字符串又是把 `a` 的值与另一个以空格开头并再含 `a` 之插值的字符串相连接的结果。

示例：
```{code-block} nix
:class: expression
let
  a = "one";
  b = "two";
in
"${a + b}"
```

```{code-block}
:class: value
"onetwo"
```

内置函数在 [后面的章节](libraries) 讨论。
:::

:::{warning}
你可能会遇到在已赋名称前使用美元符号（`$`）、但没有花括号（`{ }`）的字符串：

这些 *不是* 插值字符串，通常表示 shell 脚本中的变量。

此时使用外围 Nix 表达式中的名称只是巧合。

示例：

```{code-block} nix
:class: expression
let
  out = "Nix";
in
"echo ${out} > $out"
```

```{code-block}
:class: value
"echo Nix > $out"
```
:::

<!-- TODO: link to escaping rules -->

(indented-strings)=
### 缩进字符串

也称为「多行字符串」。

Nix 语言为具有共同缩进的多行字符串提供了便捷语法。

缩进字符串用 *双单引号*（`'' ''`）表示。

示例：

```{code-block} nix
:class: expression
''
multi
line
string
''
```

```{code-block}
:class: value
"multi\nline\nstring\n"
```

结果会去掉等量的前置空白。

示例：

```{code-block} nix
:class: expression
''
  one
   two
    three
''
```

```{code-block}
:class: value
"one\n two\n  three\n"
```

:::{note}
缩进字符串也支持 [字符串插值](string-interpolation)。
细节见 [Nix 语言字符串字面量文档](https://nix.dev/manual/nix/2.24/language/syntax#string-literal)。
:::

(file-system-paths)=
### 文件系统路径

Nix 语言为文件系统路径提供了便捷语法。

绝对路径总是以斜杠（`/`）开头。

示例：

```{code-block} nix
:class: expression
/absolute/path
```

```{code-block}
:class: value
/absolute/path
```

相对路径至少包含一个斜杠（`/`），但不以斜杠开头。
它们求值为相对于包含该表达式的文件的路径。

下列示例假定包含它的 Nix 文件位于 `/current/directory`（或 `nix repl` 在 `/current/directory` 中运行）。

示例：


```{code-block} nix
:class: expression
./relative
```

```{code-block}
:class: value
/current/directory/relative
```

示例：

```{code-block} nix
:class: expression
relative/path
```

```{code-block}
:class: value
/current/directory/relative/path
```

一个点（`.`）表示给定路径中的当前目录。

你经常会看到下面的表达式，它指定某个 Nix 文件所在的目录。

示例：

```{code-block} nix
:class: expression
./.
```

```{code-block}
:class: value
/current/directory
```

:::{dropdown} 详细说明

由于相对路径必须包含斜杠（`/`）但不能以斜杠开头，而点（`.`）表示不改变目录，组合 `./.` 就把当前目录指定为相对路径。

:::

两个点（`..`）表示父目录。

示例：

```{code-block} nix
:class: expression
../.
```

```{code-block}
:class: value
/current
```
:::{note}
路径可以用在插值表达式中——这是一种 [不纯操作](impurities)，在 [后面的章节](path-impurities) 详细介绍。
:::

(lookup-path-tutorial)=
#### 查找路径

也称为「尖括号语法」。

示例：

```{code-block} nix
:class: expression
<nixpkgs>
```

```{code-block}
:class: value
/nix/var/nix/profiles/per-user/root/channels/nixpkgs
```

[查找路径](https://nix.dev/manual/nix/2.22/language/constructs/lookup-path) 的值是一个文件系统路径，取决于 [`builtins.nixPath`](https://nix.dev/manual/nix/2.22/language/builtin-constants#builtins-nixPath) 的值。

实践中，`<nixpkgs>` 指向某个版本的 {term}`Nixpkgs` 的文件系统路径。

例如，`<nixpkgs/lib>` 指向该文件系统路径下的 `lib` 子目录：

```{code-block} nix
:class: expression
<nixpkgs/lib>
```

```{code-block}
:class: value
/nix/var/nix/profiles/per-user/root/channels/nixpkgs/lib
```

虽然你会遇到许多这样的示例，但生产代码中请 [避免查找路径](search-path)，因为它们是不可复现的 [不纯操作](impurities)。

[NIX_PATH]: https://nix.dev/manual/nix/stable/command-ref/env-common.html?highlight=nix_path#env-NIX_PATH
[nixpkgs]: https://github.com/NixOS/nixpkgs
[manual-primitives]: https://nix.dev/manual/nix/stable/language/values.html#primitives

(functions)=
## 函数

函数在 Nix 语言中无处不在，值得特别关注。

函数总是恰好接受一个参数。
参数与函数体以冒号（`:`）分隔。

在 Nix 语言代码中遇到冒号（`:`）时：
- 左侧是函数参数
- 右侧是函数体。

函数参数是除 [属性集](attrset) 与 [`let` 表达式](let) 之外，给值赋名的第三种方式。
值得注意的是，值事先未知：名称是占位符，在 [调用函数](calling-functions) 时再填入。

Nix 语言中的函数声明可以有不同形式。
下面分别说明，这里先给出概览：

- 单个参数

  ```{code-block} nix
  x: x + 1
  ```

  - 通过嵌套实现多个参数

    ```{code-block} nix
    x: y: x + y
    ```

- 属性集参数

  ```{code-block} nix
  { a, b }: a + b
  ```

  - 带默认属性

    ```{code-block} nix
    { a, b ? 0 }: a + b
    ```

  - 允许额外属性

    ```{code-block} nix
    { a, b, ...}: a + b
    ```

- 具名属性集参数

  ```{code-block} nix
  args@{ a, b, ... }: a + b + args.c
  ```

  或

  ```{code-block} nix
  { a, b, ... }@args: a + b + args.c
  ```

Nix 语言中的函数没有名称。
它们是匿名的，这种函数称为 *lambda*。[^lambda]

[^lambda]: 术语 *lambda* 是 [lambda 演算](https://en.wikipedia.org/wiki/Lambda_calculus) 中 [lambda 抽象](https://en.wikipedia.org/wiki/Lambda_calculus#lambdaAbstr) 的简写。

示例：

```{code-block} nix
:class: expression
x: x + 1
```

```{code-block} nix
:class: value
<LAMBDA>
```

`<LAMBDA>` 表示结果值是一个匿名函数。

与其他值一样，函数可以赋给一个名称。

示例：

```{code-block} nix
:class: expression
let
  f = x: x + 1;
in f
```

```{code-block} nix
:class: value
<LAMBDA>
```

(calling-functions)=
### 调用函数

也称为「函数应用」。

用参数调用函数意味着把参数写在函数后面。

示例：

```{code-block} nix
:class: expression
let
  f = x: x + 1;
in f 1
```

```{code-block}
:class: value
2
```

示例：

```{code-block} nix
:class: expression
let
  f = x: x.a;
in
f { a = 1; }
```

```{code-block}
:class: value
1
```

上面的示例对字面属性集调用 `f`。
也可以按名称传递参数。

示例：

```{code-block} nix
:class: expression
let
  f = x: x.a;
  v = { a = 1; };
in
f v
```

```{code-block}
:class: value
1
```

由于函数与参数以空白分隔，有时需要括号（`( )`）才能得到预期结果。

示例：

```{code-block} nix
:class: expression
(x: x + 1) 1
```

```{code-block}
:class: value
2
```

:::{dropdown} 详细说明

该表达式把匿名函数 `x: x + 1` 应用到参数 `1`。
函数必须写在括号中，以便与参数区分开。

:::

示例：

列表元素也以空白分隔，因此下列两者不同：

```{code-block} nix
:class: expression
let
 f = x: x + 1;
 a = 1;
in [ (f a) ]
```

```{code-block} nix
:class: value
[ 2 ]
```

```{code-block} nix
:class: expression
let
 f = x: x + 1;
 a = 1;
in [ f a ]
```

```{code-block}
:class: value
[ <LAMBDA> 1 ]
```

第一个示例读作：把 `f` 应用到 `a`，并把结果放入列表。
结果列表有一个元素。

第二个示例读作：把 `f` 与 `a` 放入列表。
结果列表有两个元素。

#### 多个参数

也称为「[柯里化] 函数」。

Nix 函数恰好接受一个参数。
多个参数可以通过嵌套函数来处理。

这种嵌套函数可以像接受多个参数的函数一样使用，同时提供额外的灵活性。

[柯里化]: https://en.wikipedia.org/wiki/Currying

示例：

```{code-block} nix
:class: expression
x: y: x + y
```

```{code-block}
:class: value
<LAMBDA>
```

上面的函数等价于

```{code-block} nix
:class: expression
x: (y: x + y)
```

```{code-block}
:class: value
<LAMBDA>
```

该函数接受一个参数，并返回另一个函数 `y: x + y`，其中 `x` 已设为该参数的值。

示例：

```{code-block} nix
:class: expression
let
  f = x: y: x + y;
in
f 1
```

```{code-block}
:class: value
<LAMBDA>
```

把 `f 1` 得到的函数再应用到另一个参数，就会得到内层函数体 `x + y`（`x` 为 `1`，`y` 为另一参数），此时可以完全求值。

```{code-block} nix
:class: expression
let
  f = x: y: x + y;
in
f 1 2
```

```{code-block}
:class: value
3
```

<!-- TODO: exercise - assign the lambda a name and do something with it -->

### 属性集参数

也称为「关键字参数」或「解构」。

Nix 函数可以声明为要求以具有特定结构的属性集作为参数。

写法是：用逗号（`,`）分隔期望的属性名，并用花括号（`{ }`）括起来。

示例：

```{code-block} nix
:class: expression
{a, b}: a + b
```

```{code-block} nix
:class: value
<LAMBDA>
```

参数定义了该集合必须恰好包含的属性。
遗漏或传入额外属性都是错误。

示例：

```{code-block} nix
:class: expression
let
  f = {a, b}: a + b;
in
f { a = 1; b = 2; }
```

```{code-block} nix
:class: value
3
```

反例：

```{code-block} nix
:class: expression
let
  f = {a, b}: a + b;
in
f { a = 1; b = 2; c = 3; }
```

```{code-block}
:class: value
error: 'f' at (string):2:7 called with unexpected argument 'c'

       at «string»:4:1:

            3| in
            4| f { a = 1; b = 2; c = 3; }
             | ^
            5|
```

<!-- TODO: not the same as x: x.a + x.b (!!!!) -->

#### 默认值

也称为「默认参数」。

解构参数可以为属性提供默认值。

写法是：用问号（`?`）分隔属性名与其默认值。

若属性有默认值，则参数中不必提供该属性。

示例：

```{code-block} nix
:class: expression
let
  f = {a, b ? 0}: a + b;
in
f { a = 1; }
```

```{code-block}
:class: value
1
```

示例：

```{code-block} nix
:class: expression
let
  f = {a ? 0, b ? 0}: a + b;
in
f { } # empty attribute set
```

```{code-block}
:class: value
0
```

#### 额外属性

用省略号（`...`）允许额外属性：

```{code-block} nix
{a, b, ...}: a + b
```

与前一个反例不同，传入包含额外属性的参数不是错误。

示例：

```{code-block} nix
:class: expression
let
  f = {a, b, ...}: a + b;
in
f { a = 1; b = 2; c = 3; }
```

```{code-block}
:class: value
3
```

### 具名属性集参数

也称为「@ 模式」「@ 语法」或「at 语法」。

可以为属性集参数命名，以便作为整体访问。

写法是：在属性集参数前或后加上名称，用 at 符号（`@`）分隔。

示例：

```{code-block} nix
:class: expression
{a, b, ...}@args: a + b + args.c
```

```{code-block}
:class: value
<LAMBDA>
```

或

```{code-block} nix
:class: expression
args@{a, b, ...}: a + b + args.c
```

```{code-block}
:class: value
<LAMBDA>
```

示例：

```{code-block} nix
:class: expression
let
  f = {a, b, ...}@args: a + b + args.c;
in
f { a = 1; b = 2; c = 3; }
```

```{code-block} nix
:class: value
6
```

(libraries)=
## 函数库

除了 [内置运算符][operators]（`+`、`==`、`&&` 等）之外，还有两个广泛使用的库，它们 *合在一起* 可视为 Nix 语言的标准库。
要理解并浏览 Nix 语言代码，你需要了解两者。

<!-- TODO: find a place for operators -->

请浏览它们，熟悉可用内容。

[operators]: https://nix.dev/manual/nix/stable/language/operators.html

(builtins)=
### `builtins`

也称为「原始操作」或「primops」。

Nix 自带许多内置于语言中的函数。
它们作为 Nix 语言解释器的一部分，用 C++ 实现。

:::{note}
Nix 手册列出了全部 [内置函数][nix-builtins]，并说明如何使用它们。
:::

这些函数可通过 `builtins` 常量访问。

示例：

```{code-block} nix
:class: expression
builtins.toString
```

```{code-block}
:class: value
<PRIMOP>
```

[nix-builtins]: https://nix.dev/manual/nix/stable/language/builtins.html

(reading-nix-language-import)=
#### `import`

大多数内置函数只能通过 `builtins` 访问。
一个显著例外是 `import`，它在顶层也可用。

`import` 接受指向 Nix 文件的路径，读取该文件以对其中的 Nix 表达式求值，并返回结果值。
若路径指向目录，则改用该目录中的 `default.nix`。

示例：

```shell-session
$ echo 1 + 2 > file.nix
```

```{code-block} nix
:class: expression
import ./file.nix
```

```{code-block}
:class: value
3
```

:::{dropdown} 详细说明

前面的 shell 命令把内容 `1 + 2` 写入当前目录的 `file.nix`。

上面的 Nix 表达式把该文件称为 `./file.nix`。
`import` 读取文件并对其中的 Nix 表达式求值。

若文件系统路径不存在则报错。

读取 `file.nix` 后，该 Nix 表达式等价于文件内容：

```{code-block} nix
:class: expression
1 + 2
```

```{code-block}
:class: value
3
```
:::

由于 Nix 文件可以包含任意 Nix 表达式，被 `import` 的函数可以立即应用到参数。

每当你在 `import` 调用后发现额外记号，返回值就是一个函数。
其后的内容都是该函数的参数。

示例：

```shell-session
$ echo "x: x + 1" > file.nix
```

```{code-block} nix
:class: expression
import ./file.nix 1
```

```{code-block}
:class: value
2
```

::::{dropdown} 详细说明

前面的 shell 命令把内容 `x: x + 1` 写入当前目录的 `file.nix`。

上面的 Nix 表达式把该文件称为 `./file.nix`。
`import ./file.nix` 读取文件并对其中的 Nix 表达式求值。

若文件系统路径不存在则报错。

读取文件后，Nix 表达式 `import ./file.nix` 等价于文件内容：

```{code-block} nix
:class: expression
(x: x + 1) 1
```

```{code-block}
:class: value
2
```

这把函数 `x: x + 1` 应用到参数 `1`，因此求值为 `2`。

:::{note}
需要括号来区分函数声明与函数应用。
:::

::::

(pkgs-lib)=
### `pkgs.lib`

[`nixpkgs`][nixpkgs] 仓库包含一个名为 [`lib`][nixpkgs-lib] 的属性集，提供大量有用函数。
它们用 Nix 语言实现，而 [`builtins`](builtins) 则是语言本身的一部分。

:::{note}
Nixpkgs 手册列出了全部 [Nixpkgs 库函数][nixpkgs-functions]。
:::

[nixpkgs-functions]: https://nixos.org/manual/nixpkgs/stable/#sec-functions-library
[nixpkgs-lib]: https://github.com/NixOS/nixpkgs/blob/master/lib/default.nix

这些函数通常通过 `pkgs.lib` 访问，按惯例把 Nixpkgs 属性集命名为 `pkgs`。

示例：

```{code-block} nix
:class: expression
let
  pkgs = import <nixpkgs> {};
in
pkgs.lib.strings.toUpper "lookup paths considered harmful"
```

```{code-block}
:class: value
LOOKUP PATHS CONSIDERED HARMFUL
```

:::{dropdown} 详细说明

这是一个更复杂的示例，但到现在你应该已经熟悉其中的所有组成部分。

名称 `pkgs` 被声明为从某个文件 `import` 的表达式。
该文件的路径由查找路径 `<nixpkgs>` 的值决定，而后者又由求值时 `$NIX_PATH` 环境变量决定。
由于该表达式恰好是一个函数，它需要一个参数才能求值；本例中传入空属性集 `{}` 就足够了。

现在 `pkgs` 在 `let ... in ...` 的作用域中，可以访问其属性。
从 Nixpkgs 手册可知，[`lib.strings.toUpper`] 下存在一个函数。

[`lib.strings.toUpper`]: https://nixos.org/manual/nixpkgs/stable/#function-library-lib.strings.toUpper

为简洁起见，本例用查找路径获取 *某个版本* 的 Nixpkgs。
函数 `toUpper` 足够简单，我们可以预期它对不同版本的 Nixpkgs 不会产生不同结果。
然而，更复杂的软件很可能受此问题影响。
一个完全可复现的示例因此会像这样：

```{code-block} nix
:class: expression
let
  nixpkgs = fetchTarball "https://github.com/NixOS/nixpkgs/archive/06278c77b5d162e62df170fec307e83f1812d94b.tar.gz";
  pkgs = import nixpkgs {};
in
pkgs.lib.strings.toUpper "always pin your sources"
```

```{code-block}
:class: value
ALWAYS PIN YOUR SOURCES
```

细节见 [](pinning-nixpkgs)。

你还会经常看到 `pkgs` 作为参数传给函数。
按惯例可以假定它指 Nixpkgs 属性集，该集合有 `lib` 属性：

```{code-block} nix
:class: expression
{ pkgs, ... }:
pkgs.lib.strings.removePrefix "no " "no true scotsman"
```

```{code-block}
:class: value
<LAMBDA>
```

要让该函数产生结果，可以把它写入文件（例如 `file.nix`），并通过 `nix-instantiate` 传入参数：

```shell-session
$ nix-instantiate --eval file.nix --arg pkgs 'import <nixpkgs> {}'
"true scotsman"
```

在 NixOS 配置中，以及在 Nixpkgs 内部，你也经常会看到直接传入 `lib`。
此时可以假定该 `lib` 等价于仅有 `pkgs` 时的 `pkgs.lib`。

示例：

```{code-block} nix
:class: expression
{ lib, ... }:
let
  to-be = true;
in
lib.trivial.or to-be (! to-be)
```

```{code-block}
:class: value
<LAMBDA>
```

要让该函数产生结果，可以把它写入文件（例如 `file.nix`），并通过 `nix-instantiate` 传入参数：

```shell-session
$ nix-instantiate --eval file.nix --arg lib '(import <nixpkgs> {}).lib'
true
```

有时 `pkgs` 与 `lib` 都会作为参数传入。
此时可以假定 `pkgs.lib` 与 `lib` 等价。
这样做是为了避免反复写 `pkgs.lib`，以提高可读性。

示例：

```{code-block} nix
{ pkgs, lib, ... }:
# ... multiple uses of `pkgs`
# ... multiple uses of `lib`
```

:::

由于历史原因，`pkgs.lib` 中的一些函数与同名的 [`builtins`](builtins) 等价。

(impurities)=
## 不纯操作

到目前为止，本教程只覆盖了 *纯表达式*：
声明数据并用函数变换它。

实践中，描述 derivation——Nix 语言的核心特性，使对文件系统进行函数式编程成为可能——需要观察外部世界。
[Derivations](derivations) 将在本教程稍后讨论。

这里相关的 Nix 语言不纯操作只有一种：
从文件系统读取文件作为 *构建输入*。

Derivations 引用构建输入以描述如何派生新文件。
运行时，derivation 只能访问显式声明的构建输入。

在 Nix 语言中，指定构建输入的唯一方式是显式地使用：

- 文件系统路径
- 专用函数

Nix 与 Nix 语言通过内容哈希引用文件。若文件内容事先未知，在表达式求值期间读取文件就不可避免。

:::{note}
Nix 还支持其他类型的不纯表达式，例如 [查找路径](search-path) 或常量 [`builtins.currentSystem`](https://nix.dev/manual/nix/stable/language/builtin-constants.html#builtins-currentSystem)。
这里不详细介绍它们，因为它们不影响 Nix 语言的基本工作方式，且由于其破坏可复现性而不被鼓励使用。
:::

(path-impurities)=
### 路径

每当文件系统路径用在 [字符串插值](string-interpolation) 中，该文件的内容会作为副作用复制到文件系统中一个特殊位置——*Nix store*。

求值后的字符串于是包含赋给该文件的 Nix store 路径。

<!-- TODO: link to explanation of the Nix store -->

示例：

```shell-session
$ echo 123 > data
```

```{code-block} nix
:class: expression
"${./data}"
```

```{code-block}
:class: value
"/nix/store/h1qj5h5n05b5dl5q4nldrqq8mdg7dhqk-data"
```

:::{dropdown} 详细说明

前面的 shell 命令把字符 `123` 写入当前目录的 `data` 文件。

上面的 Nix 表达式把该文件称为 `./data`，并把文件系统路径转换为 [插值字符串](string-interpolation) `${ ... }`。

此类插值表达式必须求值为可以表示为字符串的值。
文件系统路径就是这样的值，其字符串表示是对应的 Nix store 路径：

```{code-block}
/nix/store/<hash>-<name>
```

Nix store 路径通过取文件内容的哈希（`<hash>`）并与文件名（`<name>`）组合得到。
作为求值的副作用，文件被复制到 Nix store 目录 `/nix/store`。
若文件系统路径不存在则报错。

:::

对目录也会发生同样的事：整个目录（包括嵌套的文件与目录）被复制到 Nix store，求值后的字符串成为该目录的 Nix store 路径。

### Fetchers

用作构建输入的文件不必来自文件系统。

Nix 语言提供内置的不纯函数，以便在求值期间通过网络获取文件：

- [`builtins.fetchurl`](https://nix.dev/manual/nix/stable/language/builtins.html#builtins-fetchurl)
- [`builtins.fetchTarball`](https://nix.dev/manual/nix/stable/language/builtins.html#builtins-fetchTarball)
- [`builtins.fetchGit`](https://nix.dev/manual/nix/stable/language/builtins.html#builtins-fetchGit)
- [`builtins.fetchClosure`](https://nix.dev/manual/nix/stable/language/builtins.html#builtins-fetchClosure)

这些函数求值为 Nix store 中的文件系统路径。

示例：

```{code-block} nix
:class: expression
builtins.fetchurl "https://github.com/NixOS/nix/archive/7c3ab5751568a0bc63430b33a5169c5e4784a0ff.tar.gz"
```

```{code-block}
:class: value
"/nix/store/7dhgs330clj36384akg86140fqkgh8zf-7c3ab5751568a0bc63430b33a5169c5e4784a0ff.tar.gz"
```

其中一些还提供额外便利，例如自动解压归档。

示例：

```{code-block} nix
:class: expression
builtins.fetchTarball "https://github.com/NixOS/nix/archive/7c3ab5751568a0bc63430b33a5169c5e4784a0ff.tar.gz"
```

```{code-block}
:class: value
"/nix/store/d59llm96vgis5fy231x6m7nrijs0ww36-source"
```

:::{note}
Nixpkgs 手册中关于 [Fetchers][nixpkgs-fetchers] 的章节列出了更多通过网络获取文件的库函数。
:::

若网络请求失败则报错。

[nixpkgs-fetchers]: https://nixos.org/manual/nixpkgs/stable/#chap-pkgs-fetchers

(derivations)=
## Derivations


Derivations 是 Nix 与 Nix 语言的核心：
- Nix 语言用于描述 derivations。
- Nix 运行 derivations 以产生 *构建结果*。
- 构建结果又可以作为其他 derivations 的输入。

用于声明 derivation 的 Nix 语言原语是内置的不纯函数 `derivation`。

它通常被 Nixpkgs 的构建机制 `stdenv.mkDerivation` 包装，后者隐藏了非平凡构建过程中的大量复杂性。

:::{note}
实践中你可能永远不会直接遇到 `derivation`。
:::

每当你遇到 `mkDerivation`，它都表示 Nix 最终会 *构建* 的东西。

示例：[使用 `mkDerivation` 的包](mkDerivation-example)

`derivation`（以及 `mkDerivation`）的求值结果是一个具有特定结构与特殊属性的 [属性集](attrset)：
它可以用于 [字符串插值](string-interpolation)，此时求值为其构建结果的 Nix store 路径。

示例：

```{code-block} nix
:class: expression
let
  pkgs = import <nixpkgs> {};
in "${pkgs.nix}"
```

```{code-block}
:class: value
"/nix/store/sv2srrjddrp2isghmrla8s6lazbzmikd-nix-2.11.0"
```

:::{note}
你的输出可能不同。
它可能产生不同的哈希，甚至不同的包版本。

Derivation 的输出路径完全由其输入决定，本例中输入来自 *某个* 版本的 Nixpkgs。

因此，除了仅用于说明的示例外，请 [避免查找路径](search-path) 以确保可预期的结果。
:::

:::{dropdown} 详细说明

该示例从查找路径 `<nixpkgs>` import Nix 表达式，并把得到的函数应用到空属性集 `{}`。
其输出被赋名为 `pkgs`。

用 [字符串插值](string-interpolation) 把属性 `pkgs.nix` 转换为字符串是允许的，因为 `pkgs.nix` 是一个 derivation。
也就是说，最终 `pkgs.nix` 归结为对 `derivation` 的调用。

结果字符串是该 derivation 的构建结果将存放的文件系统路径。

Derivations 的内部机制还有更深层次，但此刻知道此类表达式求值为 Nix store 路径就足够了。

:::

对 derivations 做字符串插值，用于在声明新 derivations 时将其构建结果作为文件系统路径引用。

这允许用 Nix 语言构造任意复杂的 derivations 组合。

## 完整示例

到目前为止，示例都是对 Nix 语言构造的人为说明。

你现在应该能够阅读简单包与配置的 Nix 语言代码，并对下列实际示例给出类似的解释。

::: {note}
下列练习的目标不是理解代码的含义或工作方式，而是理解它在函数、属性集及其他 Nix 语言数据类型方面的结构。
:::

### Shell 环境

```{code-block} nix
{ pkgs ? import <nixpkgs> {} }:
let
  message = "hello world";
in
pkgs.mkShellNoCC {
  packages = with pkgs; [ cowsay ];
  shellHook = ''
    cowsay ${message}
  '';
}
```

该示例声明一个 shell 环境（初始化时运行 `shellHook`）。

说明：

- 该表达式是一个以属性集为参数的函数。
- 若参数有属性 `pkgs`，则在函数体中使用它。
  否则，默认从查找路径 `<nixpkgs>` 找到的文件中 import Nix 表达式（本例中它是一个函数），用空属性集调用该函数，并使用结果值。
- 名称 `message` 绑定到字符串值 `"hello world"`。
- `pkgs` 集合的属性 `mkShellNoCC` 是一个函数，被传入一个属性集作为参数。
  其返回值也是外层函数的结果。
- 传给 `mkShellNoCC` 的属性集有属性 `packages`（设为含一个元素的列表：来自 `pkgs` 的 `cowsay` 属性）与 `shellHook`（设为缩进字符串）。
- 缩进字符串包含一个插值表达式，它会展开 `message` 的值，得到 `"hello world"`。


### NixOS 配置

```{code-block} nix
{ config, pkgs, ... }: {

  imports = [ ./hardware-configuration.nix ];

  environment.systemPackages = with pkgs; [ git ];

  # ...

}
```

该示例是（部分）NixOS 配置。

说明：

- 该表达式是一个以属性集为参数的函数。
  它返回一个属性集。
- 参数至少必须有属性 `config` 与 `pkgs`，也可以有更多属性。
- 返回的属性集包含属性 `imports` 与 `environment`。
- `imports` 是一个含一个元素的列表：指向该 Nix 文件旁一个名为 `hardware-configuration.nix` 的文件的路径。

  :::{note}
  `imports` 不是不纯内置的 `import`，而是一个普通属性名！
  :::
- `environment` 本身是一个属性集，其中有一个属性 `systemPackages`，它将求值为含一个元素的列表：来自 `pkgs` 集合的 `git` 属性。
- 参数 `config` 没有（在示例中显示）被使用。

(mkDerivation-example)=
### 包

```{code-block} nix
{ lib, stdenv, fetchurl }:

stdenv.mkDerivation rec {

  pname = "hello";

  version = "2.12";

  src = fetchurl {
    url = "mirror://gnu/${pname}/${pname}-${version}.tar.gz";
    sha256 = "1ayhp9v4m4rdhjmnl2bq3cibrbqqkgjbl3s7yk2nhlh8vj3ay16g";
  };

  meta = with lib; {
    license = licenses.gpl3Plus;
  };

}
```

该示例是来自 Nixpkgs 的（简化）包声明。

说明：

- 该表达式是一个函数，其参数属性集必须恰好有属性 `lib`、`stdenv` 与 `fetchurl`。
- 它返回对函数 `mkDerivation` 求值的结果，`mkDerivation` 是 `stdenv` 的属性，应用于一个递归集合。
- 传给 `mkDerivation` 的递归集合在函数 `fetchurl` 的参数中使用了自身的 `pname` 与 `version` 属性。
  `fetchurl` 本身来自外层函数的参数。
- 属性 `meta` 本身是一个属性集，其中 `license` 属性的值是赋给嵌套属性 `lib.licenses.gpl3Plus` 的值。

## 参考资料

- [Nix 手册：Nix 语言][manual-language]
- [Nix 手册：字符串插值][manual-string-interpolation]
- [Nix 手册：运算符][operators]
- [Nix 手册：内置函数][nix-builtins]
- [Nix 手册：`nix repl`][`nix repl`]
- [Nixpkgs 手册：函数参考][nixpkgs-functions]
- [Nixpkgs 手册：Fetchers][nixpkgs-fetchers]

[manual-string-interpolation]: https://nix.dev/manual/nix/stable/language/string-interpolation.html

## 下一步

### 动手实践

- [](declarative-reproducible-envs) – 从 Nix 文件创建可复现的 shell 环境
- [](./packaging-existing-software.md) – 让更多软件可通过 Nix 使用


若你想暂时离开 Nix 学习，可以用下列命令从 Nix store 中移除未使用的构建结果：

```console
$ nix-collect-garbage
```

### 深入学习

若你完成了这些示例，会注意到阅读 Nix 语言能揭示代码结构，但不一定能说明代码实际意味着什么。

往往无法仅从手头代码确定：
- 某个具名值或函数参数的数据类型。
- 被调用函数接受何种数据类型的参数。
- 给定属性集中存在哪些属性。

示例：

```{code-block} nix
{ x, y, z }: (x y) z.a
```

你如何知道……
- `x` 会是一个在给定参数后返回函数的函数？
- 给定 `x` 是函数，`y` 会是适合传给 `x` 的参数？
- 给定 `(x y)` 是函数，`z.a` 会是适合传给 `(x y)` 的参数？
- `z` 到底会是一个属性集？
- 给定 `z` 是属性集，它会有属性 `a`？
- `y` 与 `z.a` 会是什么数据类型？
- 最终结果的数据类型是什么？

而该函数的调用者又如何知道它需要一个带有属性 `x`、`y`、`z` 的属性集？

回答这类问题需要了解给定表达式预期使用的上下文。

Nix 生态与代码风格由约定驱动。
你在 Nix 语言代码中遇到的大多数名称来自 Nixpkgs：

- [Nix Pills][nix-pills] - 从第一性原理详细解释 derivations 以及 Nixpkgs 如何构建

Nixpkgs 提供广泛使用的通用构建机制：

- [`stdenv`][stdenv] - 最重要的是 `mkDerivation`
- [构建辅助函数][build-helpers] - 用于创建 derivations，包括 shell 脚本与单个文件

来自 Nixpkgs 的包可通过多种机制修改：

- [overrides] – 特别是 `override` 与 `overrideAttrs`，用于修改单个包
- [overlays] – 用于产生带有个别修改包的自定义 Nixpkgs 变体

不同语言生态与框架对纳入 Nixpkgs 有不同要求：

- [语言与框架][language-support] 列出了 Nixpkgs 提供的工具，用于用 Nix 构建特定语言或框架的包。

NixOS Linux 发行版使用有其自身约定的 [模块化配置系统](module-system-tutorial)。

[nix-pills]: https://nixos.org/guides/nix-pills/
[stdenv]: https://nixos.org/manual/nixpkgs/stable/#chap-stdenv
[build-helpers]: https://nixos.org/manual/nixpkgs/stable/#part-builders
[overlays]: https://nixos.org/manual/nixpkgs/stable/#chap-overlays
[overrides]: https://nixos.org/manual/nixpkgs/stable/#chap-overrides
[language-support]: https://nixos.org/manual/nixpkgs/stable/#chap-language-support
[nixos-modules]: https://nixos.org/manual/nixos/stable/index.html#sec-writing-modules
