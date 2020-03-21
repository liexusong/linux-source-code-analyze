## CGroup 实现原理

前面我们介绍了 `CGroup` 的使用与基本概念，接下来将通过分析源码（本文使用的 Linux2.6.25 版本）来介绍 `CGroup` 的实现原理。在分析源码前，我们先介绍几个重要的数据结构，因为 `CGroup` 就是通过这几个数据结构来控制进程组对各种资源的使用。

### `cgroup` 结构体

前面介绍过，`cgroup` 是用来控制进程组对各种资源的使用，而在内核中，`cgroup` 是通过 `cgroup` 结构体来描述的，我们来看看其定义：

```cpp
struct cgroup {
    unsigned long flags;        /* "unsigned long" so bitops work */
    atomic_t count;
    struct list_head sibling;   /* my parent's children */
    struct list_head children;  /* my children */
    struct cgroup *parent;      /* my parent */
    struct dentry *dentry;      /* cgroup fs entry */
    struct cgroup_subsys_state *subsys[CGROUP_SUBSYS_COUNT];
    struct cgroupfs_root *root;
    struct cgroup *top_cgroup;
    struct list_head css_sets;
    struct list_head release_list;
};
```

下面我们来介绍一下 `cgroup` 结构体各个字段的用途：
1. `flags`: 用于标识当前 `cgroup` 的状态。
2. `count`: 引用计数器，表示有多少个进程在使用这个 `cgroup`。
3. `sibling、children、parent`: 由于 `cgroup` 是通过 `层级` 来进行管理的，这三个字段就把同一个 `层级` 的所有 `cgroup` 连接成一棵树。`parent` 指向当前 `cgroup` 的父节点，`sibling` 连接着所有兄弟节点，而 `children` 连接着当前 `cgroup` 的所有子节点。
4. `dentry`: 由于 `cgroup` 是通过 `虚拟文件系统` 来进行管理的，在介绍 `cgroup` 使用时说过，可以把 `cgroup` 当成是 `层级` 中的一个目录，所以 `dentry` 字段就是用来描述这个目录的。
5. `subsys`: 前面说过，`子系统` 能够附加到 `层级`，而附加到 `层级` 的 `子系统` 都有其限制进程组使用资源的算法和统计数据。所以 `subsys` 字段就是提供给各个 `子系统` 存放其限制进程组使用资源的统计数据。我们可以看到 `subsys` 字段是一个数组，而数组中的每一个元素都代表了一个 `子系统` 相关的统计数据。从实现来看，`cgroup` 只是把多个进程组织成控制进程组，而真正限制资源使用的是各个 `子系统`。
6. `root`: 用于保存 `层级` 的一些数据，比如：`层级` 的根节点，附加到 `层级` 的 `子系统` 列表（因为一个 `层级` 可以附加多个 `子系统`），还有这个 `层级` 有多少个 `cgroup` 节点等。
7. `top_cgroup`: `层级` 的根节点（根cgroup）。

