(module-system-deep-dive)=
# 模块系统深入讲解

或者说：*用模块包裹整个世界*

```{contributors}
:authors: infinisil
:editors: fricklerhandwerk, proofconstruction
```

在本教程中，你将跟随一次详尽的演示，学习如何用 Nix 模块包装现有 API。

## 概览

本教程跟进 [@infinisil](https://github.com/infinisil) 为 [Summer of Nix](https://github.com/ngi-nix/summer-of-nix) 2021 参与者所作的[关于模块的演讲](https://infinisil.com/modules.mp4)（[源码](https://github.com/tweag/summer-of-nix-modules)）。

一边播放视频一边阅读本教程，可能有助于更好地跟踪你将处理的代码变更。

### 你将学到什么？

你将编写模块来与 [Google Maps API](https://developers.google.com/maps/documentation/maps-static) 交互，声明表示地图几何、位置图钉等内容的模块 options。

在教程过程中，你会先写一些*不正确*的配置，借此讨论由此产生的错误信息以及如何解决，尤其是在讨论类型检查时。

### 你需要什么？

本练习将使用两个辅助脚本。
将 {download}`map.sh <files/map.sh>` 和 {download}`geocode.sh <files/geocode.sh>` 下载到你的工作目录。

:::{warning}
要运行本教程中的示例，你需要在 `$XDG_DATA_HOME/google-api/key` 中放入 [Google API 密钥](https://developers.google.com/maps/documentation/maps-static/start#before-you-begin)。
:::


## 空模块

将以下内容写入名为 `default.nix` 的文件：

```{code-block} nix
:caption: default.nix
{ ... }:
{

}
```

## 声明 options

我们需要一些辅助函数，它们来自 [Nixpkgs 库](https://github.com/NixOS/nixpkgs/tree/master/lib)，由模块系统作为 `lib` 传入：

```{code-block} diff
:caption: default.nix
- { ... }:
+ { lib, ... }:
{

}
```

使用 [`lib.mkOption`](https://nixos.org/manual/nixpkgs/stable/#function-library-lib.options.mkOption)，声明 `scripts.output` option，类型为 `lines`：

```{code-block} diff
:caption: default.nix
 { lib, ... }: {

+ options = {
+   scripts.output = lib.mkOption {
+     type = lib.types.lines;
+   };
+ };

 }
```

`lines` 类型意味着唯一合法的值是字符串，并且多个定义应以换行符连接。

:::{note}
Option 的名称与属性路径是任意的。
这里我们使用 `scripts`，因为稍后还会添加另一个脚本，并将这个命名为 `output`，因为它将输出最终的地图。
:::

## 求值模块

编写新文件 `eval.nix`，调用 [`lib.evalModules`](https://nixos.org/manual/nixpkgs/unstable/#module-system-lib-evalModules) 并对 `default.nix` 中的模块求值：

```{code-block} nix
:caption: eval.nix
let
  nixpkgs = fetchTarball "https://github.com/NixOS/nixpkgs/tarball/nixos-23.11";
  pkgs = import nixpkgs { config = {}; overlays = []; };
in
pkgs.lib.evalModules {
  modules = [
    ./default.nix
  ];
}
```

运行以下命令：

:::{warning}
这将导致错误。
:::

```console
nix-instantiate --eval eval.nix -A config.scripts.output
```

:::{dropdown} 详细说明
[`nix-instantiate --eval`](https://nix.dev/manual/nix/stable/command-ref/nix-instantiate) 会解析并求值指定路径上的 Nix 文件，并打印结果。
`evalModules` 生成一个属性集，最终配置值出现在 `config` 属性中。
因此我们在 [属性路径](https://nix.dev/manual/nix/stable/language/operators#attribute-selection) `config.scripts.output` 处求值 `eval.nix` 中的 Nix 表达式。
:::

错误信息表明 `scripts.output` option 被使用但尚未定义：在访问它之前必须为该 option 设置一个值。
你将在接下来的步骤中完成这件事。

## 类型检查

如前所述，`lines` 类型只允许字符串值。

:::{warning}
在本节中，你将设置一个非法值并遇到类型错误。
:::

如果你尝试给该 option 赋一个整数，会发生什么？

向 `default.nix` 添加以下行：

```{code-block} diff
:caption: default.nix
 { lib, ... }: {

  options = {
    scripts.output = lib.mkOption {
      type = lib.types.lines;
    };
  };

+ config = {
+   scripts.output = 42;
+ };
 }
```

现在尝试执行之前的命令，见证你的第一个模块错误：

```console
$ nix-instantiate --eval eval.nix -A config.scripts.output
error:
...
       error: A definition for option `scripts.output' is not of type `strings concatenated with "\n"'. Definition values:
       - In `/home/nix-user/default.nix': 42
```

定义 `scripts.output = 42;` 导致了类型错误：整数不是用换行符连接的字符串。

为了让这个模块通过类型检查并成功求值 `scripts.output` option，你现在要给 `scripts.output` 赋一个字符串。

在本例中，你将赋一个 shell 命令，它在当前目录运行 {download}`map <files/map.sh>` 脚本。
该脚本进而调用 Google Maps Static API 生成世界地图。
输出被传给 [`feh`](https://feh.finalrewind.org/)（一个极简图片查看器）进行显示。

更新 `default.nix`，将 `scripts.output` 的值改为以下字符串：

```{code-block} diff
:caption: default.nix
   config = {
-    scripts.output = 42;
+    scripts.output = ''
+      ./map.sh size=640x640 scale=2 | feh -
+    '';
   };
```

## 插曲：可复现的脚本

那个简单的命令在你的系统上很可能无法按预期工作，因为可能缺少所需依赖（curl 和 feh）。
我们可以通过用 `pkgs.writeShellApplication` 打包原始的 {download}`map <files/map.sh>` 脚本来解决这个问题。

首先，通过添加一个设置 `config._module.args` 的模块，在模块求值中提供 `pkgs` 参数：

```{code-block} diff
:caption: eval.nix
 pkgs.lib.evalModules {
   modules = [
+    ({ config, ... }: { config._module.args = { inherit pkgs; }; })
     ./default.nix
   ];
 }
```

:::{note}
该机制目前仅在[模块系统代码中有文档](https://github.com/NixOS/nixpkgs/blob/master/lib/modules.nix#L140-L182)，且该文档不完整且已过时。
:::

然后将 `default.nix` 改为以下内容：

```{code-block} nix
:caption: default.nix
{ pkgs, lib, ... }: {

  options = {
    scripts.output = lib.mkOption {
      type = lib.types.package;
    };
  };

  config = {
    scripts.output = pkgs.writeShellApplication {
      name = "map";
      runtimeInputs = with pkgs; [ curl feh ];
      text = ''
        ${./map.sh} size=640x640 scale=2 | feh -
      '';
    };
  };
}
```

这将访问先前添加的 `pkgs` 参数以便使用依赖，并将当前目录中的 `map` 文件复制到 Nix store，使包装后的脚本也能使用它；包装后的脚本本身也会存放在 Nix store 中。

用以下命令运行脚本：

```console
nix-build eval.nix -A config.scripts.output
./result/bin/map
```

为了更快地迭代，打开一个新终端并设置 [`entr`](https://github.com/eradman/entr)，以便在当前目录中任何源文件变更时重新运行脚本：

```console
nix-shell -p entr findutils bash --run \
  "ls *.nix | \
   entr -rs ' \
     nix-build eval.nix -A config.scripts.output --no-out-link \
     | xargs printf -- \"%s/bin/map\" \
     | xargs bash \
   ' \
  "
```

该命令执行以下操作：
- 列出所有 `.nix` 文件
- 让 `entr` 监视它们的变更。每次变更时用 `-r` 终止已调用的命令。
- 每次变更时：
    - 按上面的方式运行 `nix-build`，但不添加 `./result` 符号链接
    - 取得结果 store 路径并追加 `/bin/map`
    - 运行以该方式构造出的路径上的可执行文件

## 声明更多 options

我们不会直接设置所有脚本参数，而是通过模块系统来完成。
这不仅会通过类型检查增加一些安全性，还将允许构建抽象来管理不断增长的复杂度与变化的需求。

让我们先引入另一个 option `requestParams`，它将表示向 Google Maps API 发出的请求参数。

其类型将是 `listOf <elementType>`，即由同一类型元素组成的列表。

这次你不会用 `lines`，而是希望列表元素的类型为 `str`，一种通用的字符串类型。

`str` 与 `lines` 的区别在于它们的合并行为：
模块 option 类型不仅检查合法值，还规定同一 option 的多个定义应如何合并为一个。
- 对于 `lines`，多个定义通过用换行符连接来合并。
- 对于 `str`，不允许有多个定义。这里不成问题，因为无法多次定义同一个列表元素。

对 `default.nix` 文件做如下添加：

```{code-block} diff
:caption: default.nix
     scripts.output = lib.mkOption {
       type = lib.types.package;
     };
+
+    requestParams = lib.mkOption {
+      type = lib.types.listOf lib.types.str;
+    };
   };

  config = {
    scripts.output = pkgs.writeShellApplication {
      name = "map";
      runtimeInputs = with pkgs; [ curl feh ];
      text = ''
        ${./map.sh} size=640x640 scale=2 | feh -
      '';
    };
+
+    requestParams = [
+      "size=640x640"
+      "scale=2"
+    ];
   };
 }
```

## Options 之间的依赖

给定的模块通常声明一个产生结果、供其他地方使用的 option，在本例中是 `scripts.output`。

Options 可以依赖其他 options，从而可以构建更有用的抽象。

这里，我们希望 `scripts.output` option 使用 `requestParams` 的值作为 `./map` 脚本的参数。

### 访问 option 值

要使 option 值对模块可用，声明该模块的函数的参数必须包含 `config` 属性。

更新 `default.nix` 以添加 `config` 属性：

```{code-block} diff
:caption: default.nix
-{ pkgs, lib, ... }: {
+{ pkgs, lib, config, ... }: {
```

当设置 options 的模块被求值时，结果值可通过 `config` 下对应的属性名访问。

:::{note}
Option 值不能直接从同一模块中访问。

模块系统会对它收到的所有模块求值，其中任何一个都可以定义某个 option 的值。
当多个模块设置同一 option 时会发生什么，由该 option 的类型决定。
:::

:::{warning}
`config` *参数*与 `config` *属性* **不是**同一回事：
- `config` *参数*保存模块系统惰性求值的结果，该结果会纳入传给 `evalModules` 的所有模块及其 `imports`。
- 模块的 `config` *属性*将该特定模块的 option 值暴露给模块系统以供求值。
:::

现在对 `default.nix` 做如下修改：

```{code-block} diff
:caption: default.nix
   config = {
     scripts.output = pkgs.writeShellApplication {
       name = "map";
       runtimeInputs = with pkgs; [ curl feh ];
       text = ''
-        ${./map.sh} size=640x640 scale=2 | feh -
+        ${./map.sh} ${lib.concatStringsSep " "
+          config.requestParams} | feh -
       '';
```

这里，`config.requestParams` 属性的值由模块系统根据同一文件中的定义填充。

:::{note}
Nix 语言中的惰性求值允许模块系统把一个值放到传给定义该值的模块的 `config` 参数中。
:::

然后用 `lib.concatStringsSep " "` 将 `config.requestParams` 值中的每个列表元素连接成单个字符串，各 `requestParams` 列表元素以空格分隔。

该结果表示要传给 `./map` 脚本的命令行参数列表。

## 条件定义
有时，你希望 option 值是可选的。当不一定需要为某个 option 定义值时，这会很有用，如下例所示。

你将定义一个新 option `map.zoom`，用于控制地图的缩放级别。如果未传入对应参数，Google Maps API 会推断缩放级别；这种情况可以用 `nullOr <type>` 表示，它表示类型为 `<type>` 的值或 `null`。这*并不*自动意味着当 option 未定义时其值为 `null`——我们仍然需要定义一个默认值。

像下面这样，在顶层 `options` 声明中添加带有 `zoom` option 的 `map` 属性集：

```{code-block} diff
:caption: default.nix
     requestParams = lib.mkOption {
       type = lib.types.listOf lib.types.str;
     };
+
+    map = {
+      zoom = lib.mkOption {
+        type = lib.types.nullOr lib.types.int;
+        default = null;
+      };
+    };
   };
```

要利用这一点，使用 `mkIf <condition> <definition>` 函数，它仅在条件求值为 `true` 时添加该定义。
对 `config` 块中的 `requestParams` 列表做如下添加：

```{code-block} diff
:caption: default.nix
     requestParams = [
       "size=640x640"
       "scale=2"
+      (lib.mkIf (config.map.zoom != null)
+        "zoom=${toString config.map.zoom}")
     ];
   };
```

仅当 `config.map.zoom` 的值不是 `null` 时，才会向脚本调用添加 `zoom` 参数。

## 默认值

假设在我们的应用中，我们希望默认行为有所不同：将缩放级别设为 `10`，从而必须显式启用自动缩放。

这可以通过 [`mkOption`](https://github.com/NixOS/nixpkgs/blob/master/lib/options.nix) 的 `default` 参数完成。
若声明该默认值的 option 未被另行指定，就会使用该值。

修改相应的行：

```{code-block} diff
:caption: default.nix
     map = {
       zoom = lib.mkOption {
         type = lib.types.nullOr lib.types.int;
-        default = null;
+        default = 10;
       };
     };
   };
```

## 包装 shell 命令

你现在已经声明了控制地图尺寸与缩放级别的 options，但尚未提供指定地图中心位置的方式。

现在添加 `center` option，可以用你自己的位置作为默认值：

```{code-block} diff
:caption: default.nix
         type = lib.types.nullOr lib.types.int;
         default = 10;
       };
+
+      center = lib.mkOption {
+        type = lib.types.nullOr lib.types.str;
+        default = "switzerland";
+      };
     };
   };
```

要实现该行为，你将使用 {download}`geocode <files/geocode.sh>` 工具，它把位置名称转换为坐标。
有多种方式可以让新包可用，但作为练习，你将把它作为模块系统中的一个 option 添加。

首先，添加一个新 option 来容纳该包：


```{code-block} diff
:caption: default.nix
   options = {
     scripts.output = lib.mkOption {
       type = lib.types.package;
     };
+
+    scripts.geocode = lib.mkOption {
+      type = lib.types.package;
+    };
```

然后为该 option 定义值：通过在 `writeShellApplication` 中包装对原始脚本的调用，使其可复现：

```{code-block} diff
:caption: default.nix
   config = {
+    scripts.geocode = pkgs.writeShellApplication {
+      name = "geocode";
+      runtimeInputs = with pkgs; [ curl jq ];
+      text = ''exec ${./geocode.sh} "$@"'';
+    };
+
     scripts.output = pkgs.writeShellApplication {
       name = "map";
       runtimeInputs = with pkgs; [ curl feh ];
```

现在向 `requestParams` 列表再添加一个 `mkIf` 调用，通过 `config.scripts.geocode` 访问包装后的包，并运行其中的可执行文件 `/bin/geocode`：

```{code-block} diff
:caption: default.nix
       "scale=2"
       (lib.mkIf (config.map.zoom != null)
         "zoom=${toString config.map.zoom}")
+      (lib.mkIf (config.map.center != null)
+        "center=\"$(${config.scripts.geocode}/bin/geocode ${
+          lib.escapeShellArg config.map.center
+        })\"")
     ];
   };
```

这次你使用了 `escapeShellArg`，将 `config.map.center` 值作为命令行参数传给 `geocode`，再把结果字符串插值回设置 `center` 值的 `requestParams` 字符串中。

在 Nix 模块中包装 shell 命令执行是一种有助于控制系统变更的技术，因为它使用更符合人体工程学的属性与值接口，而不是手动处理转义的各种细节。

## 拆分模块

[模块模式](https://nixos.org/manual/nixos/stable/#sec-writing-modules)包含 `imports` 属性，它允许引入更多模块，例如将大型配置拆分为多个文件。

特别是，这允许你将 option 声明与它们在配置中的使用位置分开。

创建一个新模块 `marker.nix`，你可以在其中声明用于在地图上定义位置图钉及其他标记的 options：

```{code-block} diff
:caption: marker.nix
{ lib, config, ... }: {

}
```

在 `default.nix` 中用 `imports` 属性引用这个新文件：

```{code-block} diff
:caption: default.nix
 { pkgs, lib, config, ... }: {

+  imports = [
+    ./marker.nix
+  ];
+
```

## `submodule` 类型

我们想在地图上设置多个标记。
标记是一种有多个字段的复杂类型。

这正是模块系统类型系统中最有用的类型之一发挥作用之处：`submodule`。
该类型允许你定义带有自己 options 的嵌套模块。

这里，你将定义一个新的 `map.markers` option，其类型是子模块列表，每个子模块都有嵌套的 `location` 类型，从而可以定义地图上的标记列表。

每次对标记的赋值都会在顶层 `config` 求值期间进行类型检查。

对 `marker.nix` 做如下修改：

```{code-block} diff
:caption: marker.nix
-{ lib, config, ... }: {
+{ lib, config, ... }:
+let
+  markerType = lib.types.submodule {
+    options = {
+      location = lib.mkOption {
+        type = lib.types.nullOr lib.types.str;
+        default = null;
+      };
+    };
+  };
+in {
+
+  options = {
+    map.markers = lib.mkOption {
+      type = lib.types.listOf markerType;
+    };
+  };
```

## 在其他模块中定义 options

由于模块系统组合 option 定义的方式，你可以自由地将值赋给在其他模块中定义的 options。

在本例中，你将使用 `map.markers` option 生成新元素并添加到 `requestParams` 列表，使你声明的标记出现在返回的地图上——但这些是从 `marker.nix` 中声明的模块完成的。

要实现该行为，向 `marker.nix` 添加以下 `config` 块：

```{code-block} diff
:caption: marker.nix
+  config = {
+
+    map.markers = [
+      { location = "new york"; }
+    ];
+
+    requestParams = let
+      paramForMarker =
+        builtins.map (marker: "$(${config.scripts.geocode}/bin/geocode ${
+          lib.escapeShellArg marker.location})") config.map.markers;
+    in [ "markers=\"${lib.concatStringsSep "|" paramForMarker}\"" ];
+  };
```

:::{warning}
为避免与 `map` option 设置以及最终的 `config.map` 配置值混淆，这里我们显式使用 `builtins.map` 作为 `map` 函数。
:::


这里，你再次使用了 `escapeShellArg` 和字符串插值来生成 Nix 字符串，这次生成的是用管道符分隔的地理编码位置属性列表。

`requestParams` 值也被设为结果字符串列表，得益于 `list` 类型的默认合并行为，它会被追加到 `default.nix` 中定义的 `requestParams` 列表。

定义多个标记时，为地图确定合适的中心或缩放级别可能很难；让 API 替你完成会更容易。

为此，在 `marker.nix` 中、`requestParams` 声明上方做如下添加：

```{code-block} diff
:caption: marker.nix
+    map.center = lib.mkIf
+      (lib.length config.map.markers >= 1)
+      null;
+
+    map.zoom = lib.mkIf
+      (lib.length config.map.markers >= 2)
+      null;
+
     requestParams = let
       paramForMarker = marker:
         let
```

在这种情况下，当未传入中心或缩放级别时，Google Maps API 的默认行为是选取所有给定标记的几何中心，并设置适合一次查看所有标记的缩放级别。

## 嵌套子模块

接下来，我们希望允许多个具名用户各自定义一个标记列表。

为此你将添加一个类型为 `lib.types.attrsOf <subtype>` 的 `users` option，它允许你将 `users` 定义为一个属性集，其值的类型为 `<subtype>`。

这里，该子类型将是另一个子模块，允许声明出发标记，适合向 API 查询行程推荐路线。

这将再次使用 `markerType` 子模块，形成嵌套的子模块结构。

为了将 `users` 中的标记定义传播到 `map.markers` option，做如下修改。

在 `let` 块中：

```{code-block} diff
:caption: marker.nix
+  userType = lib.types.submodule {
+    options = {
+      departure = lib.mkOption {
+        type = markerType;
+        default = {};
+      };
+    };
+  };
+
 in {
```

这为用户定义了一个子模块类型，带有类型为 `markerType` 的 `departure` option。

在 `options` 块中、`map.markers` 上方：

```{code-block} diff
:caption: marker.nix
+    users = lib.mkOption {
+      type = lib.types.attrsOf userType;
+    };
```

这允许在任何导入 `marker.nix` 的子模块的 `config` 中添加 `users` 属性集，其中每个属性都是上一步声明的 `userType` 类型。

在 `config` 块中、`map.center` 上方：

```{code-block} diff
:caption: marker.nix
   config = {

-    map.markers = [
-      { location = "new york"; }
-    ];
+    map.markers = lib.filter
+      (marker: marker.location != null)
+      (lib.concatMap (user: [
+        user.departure
+      ]) (lib.attrValues config.users));

     map.center = lib.mkIf
       (lib.length config.map.markers >= 1)
```

这会从 `config` 参数中所有用户的 `departure` 标记取出，若其 `location` 属性不是 `null`，则将它们添加到 `map.markers`。

`config.users` 属性集被传给 `attrValues`，它返回集合中各属性值的列表（这里是你定义的 `config.users` 集合），并按字母顺序排序（这是 Nix 语言中属性名的存储方式）。

回到 `default.nix`，结果 `map.markers` option 值仍由 `requestParams` 访问，而 `requestParams` 又用于生成最终调用 Google Maps API 的脚本参数。

以这种方式定义 options，使你可以设置多个 `users.<name>.departure.location` 值，并生成具有合适缩放与中心、且图钉对应*所有* `users` 的 `departure.location` 值集合的地图。

在 2021 年的 Summer of Nix 中，这构成了交互式多人地图演示的基础。

## `strMatching` 类型

既然地图可以渲染多个标记，是时候添加一些样式定制了。

为了区分标记，向 `markerType` 子模块再添加一个 option，以允许为每个标记图钉添加标签。

API 文档指出[这些标签必须是大写字母或数字](https://developers.google.com/maps/documentation/maps-static/start#MarkerStyles)。

你可以用 `strMatching "<regex>"` 类型实现这一点，其中 `<regex>` 是一个正则表达式，将接受任何匹配的值，在本例中为大写字母或数字。

在 `let` 块中：

```{code-block} diff
:caption: marker.nix
         type = lib.types.nullOr lib.types.str;
         default = null;
       };
+
+      style.label = lib.mkOption {
+        type = lib.types.nullOr
+          (lib.types.strMatching "[A-Z0-9]");
+        default = null;
+      };
     };
   };
```

同样，`types.nullOr` 允许 `null` 值，默认值已设为 `null`。

在 `paramForMarker` 函数中：

```{code-block} diff
:caption: marker.nix
     requestParams = let
-      paramForMarker =
-        builtins.map (marker: "$(${config.scripts.geocode}/bin/geocode ${
-         lib.escapeShellArg marker.location})") config.map.markers;
-    in [ "markers=\"${lib.concatStringsSep "|" paramForMarker}\"" ];
+      paramForMarker = marker:
+        let
+          attributes =
+            lib.optional (marker.style.label != null)
+            "label:${marker.style.label}"
+            ++ [
+              "$(${config.scripts.geocode}/bin/geocode ${
+                lib.escapeShellArg marker.location
+              })"
+            ];
+        in "markers=\"${lib.concatStringsSep "|" attributes}\"";
+      in
+        builtins.map paramForMarker config.map.markers;
```

注意我们现在通过将 `label` 与 `location` 属性连接在一起，为每个用户创建一个唯一的 `marker`，并将它们赋给 `requestParams`。
仅当设置了 `marker.style.label` 时，每个 `marker` 的标签才会传播到 CLI 参数。

## 函数作为子模块参数

目前，如果未显式设置标签，就不会显示任何标签。
但由于每个 `users` 属性都有名称，我们可以用它作为自动值。

这个 `firstUpperAlnum` 函数允许你取得用户名的第一个字符，并具有传给 `departure.style.label` 的正确类型：

```{code-block} diff
:caption: marker.nix
{ lib, config, ... }:
 let
+  # Returns the uppercased first letter
+  # or number of a string
+  firstUpperAlnum = str:
+    lib.mapNullable lib.head
+    (builtins.match "[^A-Z0-9]*([A-Z0-9]).*"
+    (lib.toUpper str));

   markerType = lib.types.submodule {
     options = {
```

通过把 `lib.types.submodule` 的参数变成函数，你可以在其中访问参数。

子模块自动可用的一个特殊参数是 `name`，在 `attrsOf` 中使用时，它给出该子模块所定义在的属性名：

```{code-block} diff
:caption: marker.nix
-  userType = lib.types.submodule {
+  userType = lib.types.submodule ({ name, ... }: {
     options = {
       departure = lib.mkOption {
         type = markerType;
         default = {};
       };
     };
-  };
```

在这种情况下，你无法轻易从标记子模块的 `label` option 访问该名称，否则本可以在那里设置 `default` 值。

相反，你可以在 `user` 子模块的 `config` 部分设置默认值，如下所示：

```{code-block} diff
:caption: marker.nix
+
+    config = {
+      departure.style.label = lib.mkDefault
+        (firstUpperAlnum name);
+    };
+  });

 in {

```

:::{note}
模块 options 有一个*优先级*，以整数表示，它决定将 option 设为特定值时的优先次序。
合并值时，数值最小的优先级胜出。

`lib.mkDefault` 修饰符将其参数值的优先级设为 1000，即最低优先权。

这确保为同一 option 设置的其他值会优先生效。
:::

## `either` 与 `enum` 类型

为了更好的视觉对比，有办法改变标记的*颜色*会很有帮助。

这里你将为此使用两个新的类型函数：
- `either <this> <that>`，接受两个类型作为参数，并允许其中任意一个
- `enum [ <allowed values> ]`，接受一个允许值列表，并允许其中任意一个

在 `let` 块中，添加以下 `colorType` option，它可以保存包含某些给定颜色名称或 RGB 值的字符串，并加入这个新的复合类型：

```{code-block} diff
:caption: marker.nix
     ...
     (builtins.match "[^A-Z0-9]*([A-Z0-9]).*"
     (lib.toUpper str));

+  # Either a color name or `0xRRGGBB`
+  colorType = lib.types.either
+    (lib.types.strMatching "0x[0-9A-F]{6}")
+    (lib.types.enum [
+      "black" "brown" "green" "purple" "yellow"
+      "blue" "gray" "orange" "red" "white" ]);
+
   markerType = lib.types.submodule {
     options = {
       location = lib.mkOption {
```

这允许匹配 24 位十六进制数或等于指定颜色名称之一的字符串。

在 `let` 块底部，添加 `style.color` option 并指定默认值：

```{code-block} diff
:caption: marker.nix
           (lib.types.strMatching "[A-Z0-9]");
         default = null;
       };
+
+      style.color = lib.mkOption {
+        type = colorType;
+        default = "red";
+      };
     };
   };
```

现在向 `paramForMarker` 列表添加一个条目，以使用这个新 option：

```{code-block} diff
:caption: marker.nix
               (marker.style.label != null)
               "label:${marker.style.label}"
             ++ [
+              "color:${marker.style.color}"
               "$(${config.scripts.geocode}/bin/geocode ${
                 lib.escapeShellArg marker.location
               })"
```

如果你设置了许多不同的标记，能够单独更改它们的大小会很有帮助。

向 `marker.nix` 添加一个新的 `style.size` option，允许你从预定义大小集合中选择：

```{code-block} diff
:caption: marker.nix
         type = colorType;
         default = "red";
       };
+
+      style.size = lib.mkOption {
+        type = lib.types.enum
+          [ "tiny" "small" "medium" "large" ];
+        default = "medium";
+      };
     };
   };
```

现在在 `paramForMarker` 中为大小参数添加映射，选择合适的字符串传给 API：

```{code-block} diff
:caption: marker.nix
     requestParams = let
       paramForMarker = marker:
         let
+          size = {
+            tiny = "tiny";
+            small = "small";
+            medium = "mid";
+            large = null;
+          }.${marker.style.size};
+
```

最后，向 `attributes` 字符串再添加一个 `lib.optional` 调用，以使用所选大小：

```{code-block} diff
:caption: marker.nix
           attributes =
             lib.optional
               (marker.style.label != null)
               "label:${marker.style.label}"
+            ++ lib.optional
+              (size != null)
+              "size:${size}"
             ++ [
               "color:${marker.style.color}"
               "$(${config.scripts.geocode}/bin/geocode ${
```

## `pathType` 子模块

到目前为止，你已经创建了声明*出发*标记的 option，以及若干配置标记视觉呈现的 options。

现在我们想计算并显示从用户位置到某个目的地的路线。

下一节定义的新 option 将允许你设置*到达*标记，它与出发标记一起，使你能用下面定义的新模块在地图上绘制*路径*。

首先，创建新文件 `path.nix`，内容如下：

```{code-block} nix
:caption: path.nix
{ lib, config, ... }:
let
  pathType = lib.types.submodule {
    options = {
      locations = lib.mkOption {
        type = lib.types.listOf lib.types.str;
      };
    };
  };
in
{
  options = {
    map.paths = lib.mkOption {
      type = lib.types.listOf pathType;
    };
  };
  config = {
    requestParams =
      let
        attrForLocation = loc:
          "$(${config.scripts.geocode}/bin/geocode ${lib.escapeShellArg loc})";
        paramForPath = path:
          let
            attributes =
              builtins.map attrForLocation path.locations;
          in
          ''path="${lib.concatStringsSep "|" attributes}"'';
      in
      builtins.map paramForPath config.map.paths;
  };
}
```

`path.nix` 模块声明了一个 option，用于在我们的 `map` 上定义路径列表，每条路径是地理位置字符串的列表。


在 `config` 属性中，我们通过用适当转换后的坐标设置 `requestParams` option 值来扩充 API 调用，这些值将与其他地方设置的请求参数连接在一起。

现在从你的 `marker.nix` 模块导入这个新的 `path.nix` 模块：

```{code-block} diff
:caption: marker.nix
 in {

+  imports = [
+    ./path.nix
+  ];
+
   options = {

     users = lib.mkOption {
```

将 `departure` option 声明复制为 `marker.nix` 中的新 `arrival` option，以完成初始路径实现：

```{code-block} diff
:caption: marker.nix
         type = markerType;
         default = {};
       };
+
+      arrival = lib.mkOption {
+        type = markerType;
+        default = {};
+      };
     };
```

接下来，向 `config` 块添加 `arrival.style.label` 属性，与 `departure.style.label` 镜像对应：

```{code-block} diff
:caption: marker.nix
     config = {
       departure.style.label = lib.mkDefault
         (firstUpperAlnum name);
+      arrival.style.label = lib.mkDefault
+        (firstUpperAlnum name);
     };
   });
```

最后，更新传给 `map.markers` 中 `concatMap` 的函数的返回列表，以同时包含每个用户的 `arrival` 标记：

```{code-block} diff
:caption: marker.nix
     map.markers = lib.filter
       (marker: marker.location != null)
       (lib.concatMap (user: [
-        user.departure
+        user.departure user.arrival
       ]) (lib.attrValues config.users));

     map.center = lib.mkIf
```

现在你有了在地图上定义路径的基础，用于连接成对的出发与到达点。

在路径模块中，定义连接每个用户出发与到达位置的路径：

```{code-block} diff
:caption: path.nix
   config = {
+
+    map.paths = builtins.map (user: {
+      locations = [
+        user.departure.location
+        user.arrival.location
+      ];
+    }) (lib.filter (user:
+      user.departure.location != null
+      && user.arrival.location != null
+    ) (lib.attrValues config.users));
+
     requestParams = let
       attrForLocation = loc:
         "$(geocode ${lib.escapeShellArg loc})";
```

新的 `map.paths` 属性包含为所有用户定义的全部有效路径的列表。

仅当该用户设置了 `departure` 与 `arrival` 属性时，路径才有效。

## 整数值上的 `between` 约束

用户们发话了，他们要求能用 `weight` option 自定义路径样式。

和之前一样，你现在要为路径样式声明一个新的子模块。

虽然你也可以直接声明 `style.weight` option，但在本例中你应使用子模块，以便稍后能复用路径样式类型。

向 `path.nix` 的 `let` 块添加 `pathStyleType` 子模块 option：
```{code-block} diff
:caption: path.nix
 { lib, config, ... }:
 let
+
+  pathStyleType = lib.types.submodule {
+    options = {
+      weight = lib.mkOption {
+        type = lib.types.ints.between 1 20;
+        default = 5;
+      };
+    };
+  };
+
   pathType = lib.types.submodule {
```

:::{note}
`ints.between <lower> <upper>` 类型允许给定（含端点）范围内的整数。
:::

路径粗细默认为 5，但可以设为 1 到 20 范围内的任意整数，更高的粗细会在地图上产生更粗的路径。

现在向文件更下方的 `options` 集合添加 `style` option：

```{code-block} diff
:caption: path.nix
     options = {
       locations = lib.mkOption {
         type = lib.types.listOf lib.types.str;
       };
+
+      style = lib.mkOption {
+        type = pathStyleType;
+        default = {};
+      };
     };

   };
```

最后，更新 `paramForPath` 中的 `attributes` 列表：

```{code-block} diff
:caption: path.nix
       paramForPath = path:
         let
           attributes =
-            builtins.map attrForLocation path.locations;
+            [
+              "weight:${toString path.style.weight}"
+            ]
+            ++ builtins.map attrForLocation path.locations;
         in "path=\"${lib.concatStringsSep "|" attributes}\"";
```

## `pathStyle` 子模块

用户仍然无法真正自定义路径样式。
为每个用户引入一个新的 `pathStyle` option。

模块系统允许你多次为同一个 option 声明值；若类型允许这样做，它会负责将各声明的值合并在一起。

这使得可以在 `marker.nix` 模块中有 `users` option 的定义，同时在 `path.nix` 中也有 `users` 定义：

```{code-block} diff
:caption: path.nix
 in {
   options = {
+
+    users = lib.mkOption {
+      type = lib.types.attrsOf (lib.types.submodule {
+        options.pathStyle = lib.mkOption {
+          type = pathStyleType;
+          default = {};
+        };
+      });
+    };
+
     map.paths = lib.mkOption {
       type = lib.types.listOf pathType;
     };
```

然后在处理每个用户路径的 `map.paths` 中添加一行使用 `user.pathStyle` option：

```{code-block} diff
:caption: path.nix
         user.departure.location
         user.arrival.location
       ];
+      style = user.pathStyle;
     }) (lib.filter (user:
       user.departure.location != null
       && user.arrival.location != null
```

## 路径样式：颜色

与标记一样，路径也应有可自定义的颜色。

你可以用目前已经遇到过的类型来完成这件事。

向 `path.nix` 添加新的 `colorType` 块，指定允许的颜色名称以及 RGB/RGBA 十六进制值：

```{code-block} diff
:caption: path.nix
 { lib, config, ... }:
 let

+  # Either a color name, `0xRRGGBB` or `0xRRGGBBAA`
+  colorType = lib.types.either
+    (lib.types.strMatching "0x[0-9A-F]{6}([0-9A-F]{2})?")
+    (lib.types.enum [
+      "black" "brown" "green" "purple" "yellow"
+      "blue" "gray" "orange" "red" "white"
+    ]);
+
   pathStyleType = lib.types.submodule {
```

在 `weight` option 下，添加新的 `color` option 以使用新的 `colorType` 值：

```{code-block} diff
:caption: path.nix
         type = lib.types.ints.between 1 20;
         default = 5;
       };
+
+      color = lib.mkOption {
+        type = colorType;
+        default = "blue";
+      };
     };
   };
```

最后，向 `attributes` 列表添加一行使用 `color` option：

```{code-block} diff
:caption: path.nix
           attributes =
             [
               "weight:${toString path.style.weight}"
+              "color:${path.style.color}"
             ]
             ++ map attrForLocation path.locations;
         in "path=${
```

## 进一步的样式

既然你已经走到这一步，为了进一步改善渲染地图的美观度，再添加一个样式 option，允许将路径绘制为*测地线*，即地球上两点之间最短的“直线距离”。

由于该功能可以打开或关闭，你可以用 `bool` 类型来做，其值可以是 `true` 或 `false`。

现在对 `path.nix` 做如下修改：

```{code-block} diff
:caption: path.nix
         type = colorType;
         default = "blue";
       };
+
+      geodesic = lib.mkOption {
+        type = lib.types.bool;
+        default = false;
+      };
     };
   };
```

务必在 `attributes` 列表中也添加一行以使用该值，使 option 值包含在 API 调用中：

```{code-block} diff
:caption: path.nix
             [
               "weight:${toString path.style.weight}"
               "color:${path.style.color}"
+              "geodesic:${lib.boolToString path.style.geodesic}"
             ]
             ++ map attrForLocation path.locations;
         in "path=${
```

## 总结

在本教程中，你学会了如何编写自定义 Nix 模块，借助 Nixpkgs `lib` 中的若干新工具函数，将外部服务置于声明式控制之下。

你在多个文件中定义了几个模块，每个都带有利用模块系统类型检查的独立子模块。

这些模块以声明式方式暴露了外部 API 的功能。

你现在可以用 Nix 征服世界了。
