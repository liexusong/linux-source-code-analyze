## `OverlayFS` 源码分析

`Docker` 底层有三驾马车，`Namespace`、`CGroup` 和 `UnionFS（联合文件系统）`。前面我们介绍过 `Namespace` 和 `CGroup`，接下来将会介绍 `UnionFS` 的实现原理。

> `UnionFS（联合文件系统）` 是一种分层、轻量级并且高性能的文件系统，它支持对文件系统的修改作为一次提交来一层层的叠加，同时可以将不同目录挂载到同一个虚拟文件系统下。

`UnionFS` 是 `Docker` 镜像的基础，镜像可以通过分层来进行继承，基于基础镜像（没有父镜像），可以制作各种具体的应用镜像。由于 Linux 下有多种的 `UnionFS` （如 `AUFS`、`OverlayFS` 和 `Btrfs` 等），所以我们以实现相对简单的 `OverlayFS` 作为分析对象。

### `OverlayFS` 使用

我们先来看看 `OverlayFS` 基本原理（图片来源于网络）：

![overlayfs-map](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/overlayfs-map.png)

从上图可知，`OverlayFS` 文件系统主要有三个角色，`lowerdir`、`upperdir` 和 `merged`。`lowerdir` 是只读层，用户不能修改这个层的文件；`upperdir` 是可读写层，用户能够修改这个层的文件；而 `merged` 是合并层，把 `lowerdir` 层和 `upperdir` 层的文件合并展示。

使用 `OverlayFS` 前需要进行挂载操作，挂载 `OverlayFS` 文件系统的基本命令如下：

```bash
$ mount -t overlay overlay -o lowerdir=lower1:lower2,upperdir=upper,workdir=work merged
```

参数 `-t` 表示挂载的文件系统类型，这里设置为 `overlay` 表示文件系统类型为 `OverlayFS`，而参数 `-o` 指定的是 `lowerdir`、`upperdir` 和 `workdir`，最后的 `merged` 目录就是最终的挂载点目录。下面说明一下 `-o` 参数几个目录的作用：

1. `lowerdir`：指定用户需要挂载的lower层目录，指定多个目录可以使用 `:` 来分隔（最大支持500层）。
2. `upperdir`：指定用户需要挂载的upper层目录。
3. `workdir`：指定文件系统的工作基础目录，挂载后内容会被清空，且在使用过程中其内容用户不可见。

### `OverlayFS` 实现原理

下面我们开始分析 `OverlayFS` 的实现原理。

`OverlayFS` 文件系统的作用是合并 `upper` 目录和 `lower` 目录的中的内容，如果 `upper` 目录与 `lower` 目录同时存在同一文件或目录，那么 `OverlayFS` 文件系统怎么处理呢？

1. 如果 `upper` 和 `lower` 目录下同时存在同一文件，那么按 `upper` 目录的文件为准。比如 `upper` 与 `lower` 目录下同时存在文件 `a.txt`，那么按 `upper` 目录的 `a.txt` 文件为准。
2. 如果 `upper` 和 `lower` 目录下同时存在同一目录，那么把 `upper` 目录与 `lower` 目录的内容合并起来。比如 `upper` 与 `lower` 目录下同时存在目录 `test`，那么把 `upper` 目录下的 `test` 目录中的内容与 `lower` 目录下的 `test` 目录中的内容合并起来。

为了简单起见，本文使用的是 `Linux 3.18.3` 版本，此版本的 `OverlayFS` 文件系统只支持一层的 `lower` 目录，所以简化了多层 `lower` 合并的逻辑。

#### `OverlayFS` 文件系统挂载

前面介绍过挂载 `OverlayFS` 文件系统的命令，挂载 `OverlayFS` 文件系统会触发系统调用 `sys_mount()`，而 `sys_mount()` 会执行 `虚拟文件系统` 的通用挂载过程，如申请和初始化 `超级块对象(super block)`（可参考：[虚拟文件系统](https://github.com/liexusong/linux-source-code-analyze/blob/master/virtual_file_system.md)）。然后调用具体文件系统的 `fill_super()` 接口来填充 `超级块对象`，对于 `OverlayFS` 文件系统而言，最终会调用 `ovl_fill_super()` 函数来填充 `超级块对象`。

