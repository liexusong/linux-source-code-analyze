## CGroup 源码分析

`CGroup` 全称 `Control Group` 中文意思为 `控制组`，用于控制（限制）进程对系统各种资源的使用，比如 `CPU`、`内存`、`网络` 和 `磁盘I/O` 等资源的限制，著名的容器引擎 `Docker` 就是使用 `CGroup` 来对容器进行资源限制。

### CGroup 使用

本文主要以 `内存子系统（memory subsys）` 作为例子来阐述 `CGroup` 的原理，所以这里先介绍怎么通过 `内存子系统` 来限制进程对内存的使用。

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
