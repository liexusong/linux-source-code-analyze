## CGroup 介绍

`CGroup` 全称 `Control Group` 中文意思为 `控制组`，用于控制（限制）进程对系统各种资源的使用，比如 `CPU`、`内存`、`网络` 和 `磁盘I/O` 等资源的限制，著名的容器引擎 `Docker` 就是使用 `CGroup` 来对容器进行资源限制。

### CGroup 使用

本文主要以 `内存子系统（memory subsystem）` 作为例子来阐述 `CGroup` 的原理，所以这里先介绍怎么通过 `内存子系统` 来限制进程对内存的使用。

> `子系统` 是 `CGroup` 用于控制某种资源（如内存或者CPU等）使用的逻辑或者算法

`CGroup` 使用了 `虚拟文件系统` 来进行管理限制的资源信息和被限制的进程列表等，例如要创建一个限制内存使用的 `CGroup` 可以使用下面命令：
```bash
$ mount -t cgroup -o memory memory /sys/fs/cgroup/memory
```
上面的命令用于创建内存子系统的根 `CGroup`，如果系统已经存在可以跳过。然后我们使用下面命令在这个目录下面创建一个新的目录 `test`，
```bash
$ mkdir /sys/fs/cgroup/memory/test
```
这样就在内存子系统的根 `CGroup` 下创建了一个子 `CGroup`，我们可以通过 `ls` 目录来查看这个目录下有哪些文件：
```bash
$ ls /sys/fs/cgroup/memory/test
cgroup.clone_children       memory.kmem.max_usage_in_bytes      memory.limit_in_bytes            memory.numa_stat            memory.use_hierarchy
cgroup.event_control        memory.kmem.slabinfo                memory.max_usage_in_bytes        memory.oom_control          notify_on_release
cgroup.procs                memory.kmem.tcp.failcnt             memory.memsw.failcnt             memory.pressure_level       tasks
memory.failcnt              memory.kmem.tcp.limit_in_bytes      memory.memsw.limit_in_bytes      memory.soft_limit_in_bytes
memory.force_empty          memory.kmem.tcp.max_usage_in_bytes  memory.memsw.max_usage_in_bytes  memory.stat
memory.kmem.failcnt         memory.kmem.tcp.usage_in_bytes      memory.memsw.usage_in_bytes      memory.swappiness
memory.kmem.limit_in_bytes  memory.kmem.usage_in_bytes          memory.move_charge_at_immigrate  memory.usage_in_bytes
```
可以看到在目录下有很多文件，每个文件都是 `CGroup` 用于控制进程组的资源使用。我们可以向 `memory.limit_in_bytes` 文件写入限制进程（进程组）使用的内存大小，单位为字节(bytes)。例如可以使用以下命令写入限制使用的内存大小为 `1MB`：
```bash
$ echo 1048576 > /sys/fs/cgroup/memory/test/memory.limit_in_bytes
```
然后我们可以通过以下命令把要限制的进程加入到 `CGroup` 中：
```bash
$ echo task_pid > /sys/fs/cgroup/memory/test/tasks
```
上面的 `task_pid` 为进程的 `PID`，把进程PID添加到 `tasks` 文件后，进程对内存的使用就受到此 `CGroup` 的限制。

### CGroup 基本概念

在介绍 `CGroup` 原理前，先介绍一下 `CGroup` 几个相关的概念，因为要理解 `CGroup` 就必须要理解他们：

* `任务（task）`。任务指的是系统的一个进程，如上面介绍的 `tasks` 文件中的进程；

* `控制组（control group）`。控制组就是受相同资源限制的一组进程。`CGroup` 中的资源控制都是以控制组为单位实现。一个进程可以加入到某个控制组，也从一个进程组迁移到另一个控制组。一个进程组的进程可以使用 `CGroup` 以控制组为单位分配的资源，同时受到 `CGroup` 以控制组为单位设定的限制；

* `层级（hierarchy）`。由于控制组是以目录形式存在的，所以控制组可以组织成层级的形式，即一棵控制组组成的树。控制组树上的子节点控制组是父节点控制组的孩子，继承父控制组的特定的属性；

* `子系统（subsystem）`。一个子系统就是一个资源控制器，比如 `CPU子系统` 就是控制 CPU 时间分配的一个控制器。子系统必须附加（attach）到一个层级上才能起作用，一个子系统附加到某个层级以后，这个层级上的所有控制组都受到这个子系统的控制。

他们之间的关系如下图：

![cgroup-base](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/cgroup-base.jpg)

我们可以把 `层级` 中的一个目录当成是一个 `CGroup`，那么目录里面的文件就是这个 `CGroup` 用于控制进程组使用各种资源的信息（比如 `tasks` 文件用于保存这个 `CGroup` 控制的进程组所有的进程PID，而 `memory.limit_in_bytes` 文件用于描述这个 `CGroup` 能够使用的内存字节数）。

而附加在 `层级` 上的 `子系统` 表示这个 `层级` 中的 `CGroup` 可以控制哪些资源，每当向 `层级` 附加 `子系统` 时，`层级` 中的所有 `CGroup` 都会产生很多与 `子系统` 资源控制相关的文件。

### CGroup 操作规则

使用 `CGroup` 时，必须按照 `CGroup` 一些操作规则来进行操作，否则会出错。下面介绍一下关于 `CGroup` 的一些操作规则：

1. 一个 `层级` 可以附加多个 `子系统`，如下图：

![cgroup-rule1](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/cgroup-rule1.jpeg)

2. 一个已经被挂载的 `子系统` 只能被再次挂载在一个空的 `层级` 上，不能挂载到已经挂载了其他 `子系统` 的 `层级`，如下图：

![cgroup-rule2](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/cgroup-rule2.jpeg)

3. 每个 `任务` 只能在同一个 `层级` 的唯一一个 `CGroup` 里，并且可以在多个不同层级的 `CGroup` 中，如下图：

![cgroup-rule3](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/cgroup-rule3.jpeg)

4. 子进程在被 `fork` 出时自动继承父进程所在 `CGroup`，但是 `fork` 之后就可以按需调整到其他 `CGroup`，如下图：

![cgroup-rule4](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/cgroup-rule4.jpeg)

关于 `CGroup` 的介绍和使用就到这里，接下来我们来分析一下内核是怎么实现 `CGroup` 的。