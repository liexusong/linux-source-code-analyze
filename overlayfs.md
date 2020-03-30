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

#### `OverlayFS` 文件系统注册

要让 Linux 提供 `OverlayFS` 文件系统的功能，首先需要对 `OverlayFS` 文件系统进行注册。注册过程通过 `ovl_init()` 函数实现，代码如下：
```cpp
static struct file_system_type ovl_fs_type = {
    .owner      = THIS_MODULE,
    .name       = "overlay",
    .mount      = ovl_mount,
    .kill_sb    = kill_anon_super,
};

static int __init ovl_init(void)
{
    return register_filesystem(&ovl_fs_type);
}
```

`ovl_init()` 函数通过调用 `register_filesystem()` 函数向系统注册 `OverlayFS` 文件系统，`register_filesystem()` 函数需要提供一个类型为 `file_system_type` 结构的参数，`file_system_type` 结构的 `mount` 字段指定当挂载文件系统时进行的相应操作例程，而 `name` 字段表示文件系统的名字。

注册完 `OverlayFS` 文件系统后就可以进行挂载。我们抛弃 `虚拟文件系统` 挂载的一般过程，只针对 `OverlayFS` 文件系统的特殊处理。

当挂载 `OverlayFS` 文件系统时会触发调用 `ovl_mount()` 函数，`ovl_mount()` 函数代码如下：
```cpp
static struct dentry *ovl_mount(struct file_system_type *fs_type,
    int flags, const char *dev_name, void *raw_data)
{
    return mount_nodev(fs_type, flags, raw_data, ovl_fill_super);
}
```

从上面的代码可以看出，`ovl_mount()` 函数调用了 `mount_nodev()` 函数进行挂载。`mount_nodev()` 函数会处理挂载过程的通用逻辑（如创建文件系统超级块），然后调用 `ovl_fill_super()` 函数对超级块进行填充。

下面我们来分析一下 `ovl_fill_super()` 函数的实现，在分析 `ovl_fill_super()` 函数前先介绍一下 `ovl_fs` 和 `ovl_entry` 这两个结构，因为 `ovl_fill_super()` 函数使用了这两个结构。


__`ovl_fs` 结构定义如下：__
```cpp
struct ovl_config {
    char *lowerdir;
    char *upperdir;
    char *workdir;
};

struct ovl_fs {
    struct vfsmount *upper_mnt;
    struct vfsmount *lower_mnt;
    struct dentry *workdir;
    long lower_namelen;
    struct ovl_config config;
};
```
下面介绍一下 `ovl_fs` 结构的各个字段：
1. `upper_mnt`：upper目录的挂载点。
2. `lower_mnt`：lower目录的挂载点。
3. `workdir`：保存work目录的dentry对象。
4. `lower_namelen`：lower目录的名字长度。
5. `config`：用于保存挂载文件系统时传入的参数（如lower目录路径、upper目录路径和work目录路径）。
