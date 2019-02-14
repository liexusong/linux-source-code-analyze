## 内存交换
由于计算机的物理内存是有限的, 而进程对内存的使用是不确定的, 所以物理内存总有用完的可能性. 那么当系统的物理内存不足时, Linux内核使用什么方案来避免申请不到物理内存这个问题呢?

相对于内存来说, 磁盘的容量是非常大的, 所以Linux内核实现了一个叫 `内存交换` 的功能 -- 把某些进程的一些暂时用不到的内存页保存到磁盘中, 然后把物理内存页分配给更紧急的用户使用, 当进程用到时再从磁盘读回到内存中即可. 有了 `内存交换` 功能, 系统可使用的内存就可以远远大于物理内存的容量.

### LRU算法
`内存交换` 过程首先是找到一个合适的用户进程内存管理结构，然后把进程占用的内存页交换到磁盘中，并断开虚拟内存与物理内存的映射，最后释放进程占用的内存页。由于涉及到IO操作，所以这是一个比较耗时的过程。如果被交换出去的内存页刚好又被访问了，这时又需要从磁盘中把内存页的数据交换到内存中。所以，在这种情况下不单不能解决内存紧缺的问题，而且增加了系统的负荷。

为了解决这个问题，Linux内核使用了一种称为 `LRU (Least Recently Used)` 的算法, 下面介绍一下 `LRU算法` 的大体过程.

`LRU` 的中文翻译是 `最近最少使用`, 顾名思义就是一段时间内没有被使用, 那么Linux内核怎么知道哪些内存页面最近没有被使用呢? 最简单的方法就是把内存页放进一个队列里, 如果内存页被访问了, 就把内存页移动到链表的头部, 这样没被访问的内存页在一段时间后便会移动到队列的尾部, 而释放内存页时从链表的尾部开始. 著名的缓存服务器 `memcached` 就是使用这种 `LRU算法`.

### kswapd内核线程
在Linux系统启动时会调用 `kswapd_init()` 函数, 代码如下:
```cpp
static int __init kswapd_init(void)
{
    printk("Starting kswapd v1.8\n");
    swap_setup();
    kernel_thread(kswapd, NULL, CLONE_FS | CLONE_FILES | CLONE_SIGNAL);
    kernel_thread(kreclaimd, NULL, CLONE_FS | CLONE_FILES | CLONE_SIGNAL);
    return 0;
}
```
可以看到, `kswapd_init()` 函数会创建 `kswapd` 和 `kreclaimd` 两个内核线程, 这两个内核线程负责在系统物理内存紧缺时释放一些物理内存页, 从而使系统的可用内存达到一个平衡. 下面我们重点来分析 `kswapd` 这个内核线程, `kswapd()` 的源码如下:
```cpp
int kswapd(void *unused)
{
    struct task_struct *tsk = current;

    tsk->session = 1;
    tsk->pgrp = 1;
    strcpy(tsk->comm, "kswapd");
    sigfillset(&tsk->blocked);
    kswapd_task = tsk;

    tsk->flags |= PF_MEMALLOC;

    for (;;) {
        static int recalc = 0;

        if (inactive_shortage() || free_shortage()) {
            int wait = 0;
            /* Do we need to do some synchronous flushing? */
            if (waitqueue_active(&kswapd_done))
                wait = 1;
            do_try_to_free_pages(GFP_KSWAPD, wait);
        }

        refill_inactive_scan(6, 0);

        if (time_after(jiffies, recalc + HZ)) {
            recalc = jiffies;
            recalculate_vm_stats();
        }

        wake_up_all(&kswapd_done);
        run_task_queue(&tq_disk);

        if (!free_shortage() || !inactive_shortage()) {
            interruptible_sleep_on_timeout(&kswapd_wait, HZ);
        } else if (out_of_memory()) {
            oom_kill();
        }
    }
}
```
`kswapd` 内核线程由一个无限循环组成, 首先通过 `inactive_shortage()` 和 `free_shortage()` 函数判断系统的非活跃页面和空闲物理内存页是否短缺, 如果短缺的话, 那么就调用 `do_try_to_free_pages()` 函数试图释放一些物理内存页. 然后通过调用 `refill_inactive_scan()` 函数把一些活跃链表中的内存页移动到非活跃脏链表中. 最后, 如果空闲物理内存页或者非活跃内存页不短缺, 那么就让 `kswapd` 内核线程休眠一秒.

接下来我们分析一下 `do_try_to_free_pages()` 函数做了一些什么工作, 代码如下:
```cpp
static int do_try_to_free_pages(unsigned int gfp_mask, int user)
{
    int ret = 0;

    if (free_shortage() || nr_inactive_dirty_pages > nr_free_pages() + nr_inactive_clean_pages())
        ret += page_launder(gfp_mask, user);

    if (free_shortage() || inactive_shortage()) {
        shrink_dcache_memory(6, gfp_mask);
        shrink_icache_memory(6, gfp_mask);
        ret += refill_inactive(gfp_mask, user);
    } else {
        kmem_cache_reap(gfp_mask);
        ret = 1;
    }

    return ret;
}
```
`do_try_to_free_pages()` 函数第一步先判断系统中的空闲物理内存页是否短缺, 或者非活跃脏页面的数量大于空闲物理内存页和非活跃干净页面的总和, 其中一个条件满足了, 就调用 `page_launder()` 函数把非活跃脏链表中的页面刷到磁盘中, 然后移动到非活跃干净链表中.
