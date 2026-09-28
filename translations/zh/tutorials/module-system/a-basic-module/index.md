# 一个基础模块

```{contributors}
:authors: djacu
:editors: fricklerhandwerk
```

什么是模块？

* 模块是一个接受属性集并返回属性集的函数。
* 它可以声明 options，说明最终结果中允许哪些属性。
* 它可以为自身或其他模块声明的 options 定义值。
* 由模块系统求值时，它会根据声明与定义生成一个属性集。

最简单的模块是一个接受任意属性并返回空属性集的函数：

```{code-block} nix
:caption: options.nix
{ ... }:
{
}
```

要定义任何值，模块系统必须先知道哪些值是允许的。
这通过声明 *options* 来完成，它们指定哪些属性可以被设置并在其他地方使用。

## 声明 options

Options 在顶层 `options` 属性下用 [`lib.mkOption`](https://nixos.org/manual/nixpkgs/stable/#function-library-lib.options.mkOption) 声明。

```{literalinclude} options.nix
:language: nix
:caption: options.nix
```

:::{note}
`lib` 参数由模块系统自动传入。
这使得 [Nixpkgs 库函数](https://nixos.org/manual/nixpkgs/stable/#chap-functions) 可在每个模块的函数体中使用。

省略号 `...` 是必要的，因为模块系统可以向模块传入任意参数。

:::

传给 `lib.mkOption` 的参数中，属性 `type` 指定该 option 的哪些值是合法的。
[`lib.types`](https://nixos.org/manual/nixos/stable/#sec-option-types-basic) 下有多种可用类型。

这里我们声明了一个类型为 `str` 的 option `name`：
模块系统在定义值时期望得到一个字符串。

既然已经声明了 option，接下来自然要给它赋值。

## 定义值

Options 在顶层 `config` 属性下设置或 *定义*：

```{literalinclude} config.nix
:language: nix
:caption: config.nix
```

在 option 声明中，我们创建了一个字符串类型的 option `name`。
在 option 定义中，我们把同一个 option 设为一个字符串。

Option 声明与 option 定义不必在同一文件中。
哪些模块会参与生成最终属性集，在设置模块系统求值时指定。

## 求值模块

模块由 Nixpkgs 库中的 [`lib.evalModules`](https://nixos.org/manual/nixpkgs/stable/#module-system-lib-evalModules) 求值。
它接受一个属性集作为参数，其中 `modules` 属性是要合并并求值的模块列表。

`evalModules` 的输出包含所有已求值模块的信息，最终值出现在属性 `config` 中。

```{literalinclude} default.nix
:language: nix
:caption: default.nix
```

下面是一个辅助脚本，用 [`nix-instantiate --eval`](https://nix.dev/manual/nix/stable/command-ref/nix-instantiate) 解析并求值我们的 `default.nix` 文件，并以 JSON 打印输出：

```{literalinclude} eval.bash
:language: bash
:caption: eval.bash
```

只要每个定义都有对应的声明，求值就会成功。
如果存在未声明的 option 定义，或定义的值类型错误，模块系统会抛出错误。

运行脚本（`./eval.bash`）应显示与我们配置一致的输出：

```{code-block}
{
  "name": "Boaty McBoatface"
}
```
