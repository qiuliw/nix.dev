(python-dev-environment)=
# 搭建 Python 开发环境

本示例中，你将把使用 [Flask](https://flask.palletsprojects.com) Web 框架构建 Python Web 应用作为练习。
要充分利用本示例，你应熟悉[定义声明式 shell 环境](declarative-reproducible-envs)。

创建名为 `myapp.py` 的新文件并添加以下代码：

```{code-block} python myapp.py
#!/usr/bin/env python

from flask import Flask

app = Flask(__name__)

@app.route("/")
def hello():
    return {
        "message": "Hello, Nix!"
    }

def run():
    app.run(host="0.0.0.0", port=5000)

if __name__ == "__main__":
    run()
```

这是一个简单的 Flask 应用，它提供包含消息 `"Hello, Nix!"` 的 JSON 文档。

创建新文件 `shell.nix` 以声明开发环境：

```{code-block} nix shell.nix
{ pkgs ? import (fetchTarball "https://github.com/NixOS/nixpkgs/tarball/nixos-23.11") {} }:

pkgs.mkShellNoCC {
  packages = with pkgs; [
    (python3.withPackages (ps: [ ps.flask ]))
    curl
    jq
  ];
}
```

这描述了一个 shell 环境，其中包含通过 [`python3.withPackages`](https://nixos.org/manual/nixpkgs/stable/#python.withpackages-function) 包含 `flask` 包的 `python3` 实例。
它还包含用于执行 Web 请求的工具 [`curl`]，以及用于解析和格式化 JSON 文档的工具 [`jq`]。

[`curl`]: https://search.nixos.org/packages?show=curl
[`jq`]: https://search.nixos.org/packages?show=jq

这两者都不是 Python 包。
如果你采用 Python 的 [virtualenv](https://virtualenv.pypa.io/en/latest/)，则无法在不额外手动操作的情况下将这些工具加入开发环境。

运行 `nix-shell` 进入你刚声明的环境：

```shell-session
$ nix-shell
these 2 derivations will be built:
  /nix/store/5yvz7zf8yzck6r9z4f1br9sh71vqkimk-builder.pl.drv
  /nix/store/aihgjkf856dbpjjqalgrdmxyyd8a5j2m-python3-3.9.13-env.drv
these 93 paths will be fetched (109.50 MiB download, 468.52 MiB unpacked):
  /nix/store/0xxjx37fcy2nl3yz6igmv4mag2a7giq6-glibc-2.33-123
  /nix/store/138azk9hs5a2yp3zzx6iy1vdwi9q26wv-hook
...

[nix-shell:~]$
```

在该 shell 环境中启动 Web 应用：

```shell-session
[nix-shell:~]$ python ./myapp.py
 * Serving Flask app 'myapp'
 * Debug mode: off
WARNING: This is a development server. Do not use it in a production deployment. Use a production WSGI server instead.
 * Running on all addresses (0.0.0.0)
 * Running on http://127.0.0.1:5000
 * Running on http://192.168.1.100:5000
Press CTRL+C to quit
```

你现在拥有一个正在运行的 Python Web 应用。
试一试吧。

打开新终端以启动另一个 shell 环境会话，并按以下命令操作：

```shell-session
$ nix-shell

[nix-shell:~]$ curl 127.0.0.1:5000
{"message":"Hello, Nix!"}

[nix-shell:~]$ curl 127.0.0.1:5000 | jq '.message'
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100    26  100    26    0     0  13785      0 --:--:-- --:--:-- --:--:-- 26000
"Hello, Nix!"
```

如所示，你可以使用 `curl` 和 `jq` 测试正在运行的 Web 应用，无需任何手动安装。

你可以将我们创建的文件提交到版本控制并与他人共享。
只要他人已[安装 Nix](install-nix)，就可以使用相同的 shell 环境。

## 下一步

- [](packaging-tutorial)
- [](file-sets-tutorial)
- [](automatic-direnv)
- [](./dependency-management.md)
