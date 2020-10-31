# Linux源码分析
基于 Linux-2.4.0, 源码: [linux-2.4.0](https://github.com/liexusong/linux-2.4.0)

## 目录

* 进程管理
    * [进程管理](https://github.com/liexusong/linux-source-code-analyze/blob/master/process-management.md)
    * [进程调度](https://github.com/liexusong/linux-source-code-analyze/blob/master/process-schedule.md)
    * [并发同步](https://github.com/liexusong/linux-source-code-analyze/blob/master/concurrency-synchronize.md)
    * [等待队列](https://github.com/liexusong/linux-source-code-analyze/blob/master/waitqueue.md)
* 内存管理
    * [物理内存管理](https://github.com/liexusong/linux-source-code-analyze/blob/master/physical-memory-managemen-1.md)
    * [伙伴分配算法](https://github.com/liexusong/linux-source-code-analyze/blob/master/physical-memory-managemen-2.md)
    * [Slab分配算法](https://github.com/liexusong/linux-source-code-analyze/blob/master/physical-memory-managemen-3.md)
    * [虚拟内存管理](https://github.com/liexusong/linux-source-code-analyze/blob/master/virtual_memory_address_manager.md)
    * [mmap完全剖析](https://github.com/liexusong/linux-source-code-analyze/blob/master/memory_mmap.md)
    * [内存交换](https://github.com/liexusong/linux-source-code-analyze/blob/master/memory_swap.md)
    * [vmalloc原理与实现](https://github.com/liexusong/linux-source-code-analyze/blob/master/vmalloc-memory-implements.md)
* 中断机制
    * [硬件相关](https://github.com/liexusong/linux-source-code-analyze/blob/master/interrupt_hardware.md)
    * [中断处理](https://github.com/liexusong/linux-source-code-analyze/blob/master/interrupt_softward.md)
* 系统调用
    * [系统调用](https://github.com/liexusong/linux-source-code-analyze/blob/master/syscall.md)
* 文件系统
    * [虚拟文件系统](https://github.com/liexusong/linux-source-code-analyze/blob/master/virtual_file_system.md)
    * [MINIX文件系统](https://github.com/liexusong/linux-source-code-analyze/blob/master/minix_file_system.md)
* 进程间通信
    * [信号处理机制](https://github.com/liexusong/linux-source-code-analyze/blob/master/signal.md)
    * [共享内存](https://github.com/liexusong/linux-source-code-analyze/blob/master/ipc-shm.md)
* 网络
    * [Socket接口](https://github.com/liexusong/linux-source-code-analyze/blob/master/socket_interface.md)
    * [Unix Domain Socket](https://github.com/liexusong/linux-source-code-analyze/blob/master/unix-domain-sockets.md)
* 其他
    * [定时器实现](https://github.com/liexusong/linux-source-code-analyze/blob/master/kernel-timer.md)
    * [多路复用I/O](https://github.com/liexusong/linux-source-code-analyze/blob/master/multiplexing-io.md)
    * [GDB原理之ptrace](https://github.com/liexusong/linux-source-code-analyze/blob/master/ptrace.md)
* 容器相关
    * [docker实现原理之 - namespace](https://github.com/liexusong/linux-source-code-analyze/blob/master/namespace.md)
    * [docker实现原理之 - CGroup介绍](https://github.com/liexusong/linux-source-code-analyze/blob/master/cgroup.md)
    * [docker实现原理之 - CGroup实现原理](https://github.com/liexusong/linux-source-code-analyze/blob/master/cgroup-principle.md)
    * [docker实现原理之 - OverlayFS实现原理](https://github.com/liexusong/linux-source-code-analyze/blob/master/overlayfs.md)
* 2.6+内核分析
    * [Epoll原理与实现](https://github.com/liexusong/linux-source-code-analyze/blob/master/epoll-principle.md)
    * [RCU原理与实现](https://github.com/liexusong/linux-source-code-analyze/blob/master/rcu.md)
    * [O(1)调度算法](https://github.com/liexusong/linux-source-code-analyze/blob/master/process-schedule-o1.md)
