# 故障排除

本页汇总了使用 Nix 时可能遇到的问题及其解决提示。

## 二进制缓存宕机或无法访问时怎么办？

向 Nix 命令传入 [`--option substitute false`](https://nix.dev/manual/nix/stable/command-ref/conf-file#conf-substitute)。

## 如何强制 Nix 重新检查二进制缓存中是否存在某内容？

Nix 会跟踪二进制缓存中有哪些内容，以免每次命令都去查询。
这也包括否定答案，即某个给定的 store 路径无法被替换（substitute）的情况。

传入 [`--narinfo-cache-negative-ttl`](https://nix.dev/manual/nix/stable/command-ref/conf-file.html#conf-narinfo-cache-negative-ttl) 选项，以秒为单位设置缓存超时。

## 如何修复：`error: querying path in database: database disk image is malformed`

这是一个[已知问题](https://github.com/NixOS/nix/issues/1353)。
尝试：

```shell-session
$ sqlite3 /nix/var/nix/db/db.sqlite "pragma integrity_check"
```

这将打印出[数据库](https://nix.dev/manual/nix/stable/glossary#gloss-nix-database)中的错误。
若错误是由于缺失引用导致的，以下方法可能有效：

```shell-session
$ mv /nix/var/nix/db/db.sqlite /nix/var/nix/db/db.sqlite-bkp
$ sqlite3 /nix/var/nix/db/db.sqlite-bkp ".dump" | sqlite3 /nix/var/nix/db/db.sqlite
```

## 如何修复：`error: current Nix store schema is version 10, but I only support 7`

这是一个[已知问题](https://github.com/NixOS/nix/issues/1251)。

这意味着使用新版本的 Nix 升级了[数据库](https://nix.dev/manual/nix/stable/glossary#gloss-nix-database)的 SQLite schema，之后你又尝试使用旧版本的 Nix。

解决方法是导出数据库，并用旧版 Nix 重新导入数据：

```shell-session
$ /path/to/nix/unstable/bin/nix-store --dump-db > /tmp/db.dump
$ mv /nix/var/nix/db /nix/var/nix/db.toonew
$ mkdir /nix/var/nix/db
$ nix-store --load-db < /tmp/db.dump
```

## 如何修复：`writing to file: Connection reset by peer`

这可能意味着你试图将过大的文件或目录导入 [Nix store](https://nix.dev/manual/nix/stable/glossary#gloss-store)，或者机器资源不足，例如磁盘空间或内存。

尝试减小要导入的目录大小，或运行[垃圾回收](https://nix.dev/manual/nix/stable/command-ref/nix-collect-garbage)。

## macOS 更新破坏了 Nix 安装

这是一个[已知问题](https://github.com/NixOS/nix/issues/3616)。
[Nix 安装程序](https://nix.dev/manual/nix/latest/installation/installing-binary)会修改 `/etc/zshrc`。
当 macOS 更新时，通常会再次覆盖 `/etc/zshrc`。

作为变通方法，将以下代码片段添加到 `/etc/zshrc` 末尾并重启 shell：

```bash
if [ -e '/nix/var/nix/profiles/default/etc/profile.d/nix-daemon.sh' ]; then
  . '/nix/var/nix/profiles/default/etc/profile.d/nix-daemon.sh'
fi
```
