## `OverlayFS` 源码分析

`Docker` 底层有三驾马车，`Namespace`、`CGroup` 和 `UnionFS（联合文件系统）`。前面我们介绍过 `Namespace` 和 `CGroup`，接下来将会介绍 `UnionFS` 的实现原理。

> `UnionFS（联合文件系统）` 是一种分层、轻量级并且高性能的文件系统，它支持对文件系统的修改作为一次提交来一层层的叠加，同时可以将不同目录挂载到同一个虚拟文件系统下。

`UnionFS` 是 `Docker` 镜像的基础，镜像可以通过分层来进行继承，基于基础镜像（没有父镜像），可以制作各种具体的应用镜像。由于 Linux 下有多种的 `UnionFS` （如 `AUFS`、`OverlayFS` 和 `Btrfs` 等），所以我们以实现相对简单的 `OverlayFS` 作为分析对象。

### `OverlayFS` 使用

我们先来看看 `OverlayFS` 基本原理图：

![overlayfs-map]()

使用 `OverlayFS` 前需要进行挂载操作，挂载 `OverlayFS` 文件系统的基本命令如下：

```bash
$ mount -t overlay overlay -o lowerdir=lower1:lower2:lower3,upperdir=upper,workdir=work merged
```

参数 `-t` 表示挂载的文件系统类型，这里设置为 `overlay` 表示文件系统类型为 `OverlayFS`，而参数 `-o` 指定的是 `lowerdir`、`upperdir` 和 `workdir`，最后的 `merged` 目录就是最终的挂载点目录。下面说明一下 `-o` 参数几个目录的作用：

1. `lowerdir`：指定用户需要挂载的lower层目录（支持多lower，最大支持500层）。
2. `upperdir`：指定用户需要挂载的upper层目录。
3. `workdir`：指定文件系统的工作基础目录，挂载后内容会被清空，且在使用过程中其内容用户不可见。

