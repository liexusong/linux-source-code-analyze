# Linux源码分析

## 目录

* 进程管理
    * [进程管理](https://github.com/liexusong/linux-source-code-analyze/blob/master/process-management.md)
    * [进程调度](https://github.com/liexusong/linux-source-code-analyze/blob/master/process-schedule.md)
* 同步机制
    * [并发同步](https://github.com/liexusong/linux-source-code-analyze/blob/master/concurrency-synchronize.md)
    * [等待队列](https://github.com/liexusong/linux-source-code-analyze/blob/master/waitqueue.md)
    * [顺序锁](https://github.com/liexusong/linux-source-code-analyze/blob/master/seqlock.md)
* 内存管理
    * [物理内存管理](https://github.com/liexusong/linux-source-code-analyze/blob/master/physical-memory-managemen.md)
    * [伙伴分配算法](https://github.com/liexusong/linux-source-code-analyze/blob/master/physical-memory-buddy-system.md)
    * [Slab分配算法](https://github.com/liexusong/linux-source-code-analyze/blob/master/physical-memory-slab-algorithm.md)
    * [虚拟内存管理](https://github.com/liexusong/linux-source-code-analyze/blob/master/virtual_memory_address_manager.md)
    * [mmap完全剖析](https://github.com/liexusong/linux-source-code-analyze/blob/master/memory_mmap.md)
    * [内存交换](https://github.com/liexusong/linux-source-code-analyze/blob/master/memory_swap.md)
    * [vmalloc原理与实现](https://github.com/liexusong/linux-source-code-analyze/blob/master/vmalloc-memory-implements.md)
    * [写时复制](https://github.com/liexusong/linux-source-code-analyze/blob/master/copy-on-write.md)
    * [零拷贝技术](https://github.com/liexusong/linux-source-code-analyze/blob/master/zero-copy.md)
    * [虚拟内存空间管理](https://github.com/liexusong/linux-source-code-analyze/blob/master/process-virtual-memory-manage.md)
* 中断机制
    * [硬件相关](https://github.com/liexusong/linux-source-code-analyze/blob/master/interrupt_hardware.md)
    * [中断处理](https://github.com/liexusong/linux-source-code-analyze/blob/master/interrupt_softward.md)
    * [系统调用](https://github.com/liexusong/linux-source-code-analyze/blob/master/syscall.md)
* 文件系统
    * [虚拟文件系统](https://github.com/liexusong/linux-source-code-analyze/blob/master/virtual_file_system.md)
    * [MINIX文件系统](https://github.com/liexusong/linux-source-code-analyze/blob/master/minix_file_system.md)
    * [通用块层](https://github.com/liexusong/linux-source-code-analyze/blob/master/filesystem-generic-block-layer.md)
    * [直接I/O](https://github.com/liexusong/linux-source-code-analyze/blob/master/direct-io.md)
    * [原生异步I/O](https://github.com/liexusong/linux-source-code-analyze/blob/master/native-aio.md)
    * [inotify源码分析](https://github.com/liexusong/linux-source-code-analyze/blob/master/inotify-source-code-analysis.md)
* 进程间通信
    * [信号处理机制](https://github.com/liexusong/linux-source-code-analyze/blob/master/signal.md)
    * [共享内存](https://github.com/liexusong/linux-source-code-analyze/blob/master/ipc-shm.md)
* 网络
    * [Socket接口](https://github.com/liexusong/linux-source-code-analyze/blob/master/socket_interface.md)
    * [Unix Domain Socket](https://github.com/liexusong/linux-source-code-analyze/blob/master/unix-domain-sockets.md)
    * [TUN/TAP设备原理与实现](https://github.com/liexusong/linux-source-code-analyze/blob/master/tun-tap-principle.md)
    * [LVS原理与实现 - 原理篇](https://github.com/liexusong/linux-source-code-analyze/blob/master/lvs-principle-and-source-analysis-part1.md)
    * [LVS原理与实现 - 实现篇](https://github.com/liexusong/linux-source-code-analyze/blob/master/lvs-principle-and-source-analysis-part2.md)
    * [ARP协议与邻居子系统剖析](https://github.com/liexusong/linux-source-code-analyze/blob/master/arp-neighbour.md)
    * [IP协议源码分析](https://github.com/liexusong/linux-source-code-analyze/blob/master/ip-source-code.md)
    * [UDP协议源码分析](https://github.com/liexusong/linux-source-code-analyze/blob/master/udp-source-code.md)
    * [TCP源码分析 - 三次握手之 connect 过程](https://github.com/liexusong/linux-source-code-analyze/blob/master/tcp-three-way-handshake-connect.md)
    * [Linux网桥工作原理与实现](https://github.com/liexusong/linux-source-code-analyze/blob/master/net_bridge.md)
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
    * [完全公平调度算法](https://github.com/liexusong/linux-source-code-analyze/blob/master/cfs-scheduler.md)
    * [HugePages原理与使用](https://github.com/liexusong/linux-source-code-analyze/blob/master/hugepage.md)
    * [HugePages实现剖析](https://github.com/liexusong/linux-source-code-analyze/blob/master/hugepages-source-code-analysis.md)
    * [什么是iowait](https://github.com/liexusong/linux-source-code-analyze/blob/master/iowait.md)
    
## 其他版本Linux

### 1、Linux-3.x

### 2、Linux-4.x
* eBPF
    * [eBPF源码分析 - kprobe模块](https://github.com/liexusong/linux-source-code-analyze/blob/master/eBPF.md)

### 3、Linux-5.x
* 文件系统与I/O
   * io_uring

### 我们的公众号

![qrcode](https://image-static.segmentfault.com/376/558/3765589661-607fef350658b_fix732)



