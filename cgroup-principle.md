## CGroup 实现原理

前面我们介绍了 `CGroup` 的使用与基本概念，接下来将通过分析源码来介绍 `CGroup` 的实现原理。在分析源码前，我们先介绍几个重要的数据结构，因为 `CGroup` 就是通过这几个数据结构来控制进程组对各种资源的使用。

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
