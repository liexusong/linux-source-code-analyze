## 内存交换
由于计算机的物理内存是有限的, 而进程对内存的使用是不确定的, 所以物理内存总有用完的可能性. 那么当系统的物理内存不足时, Linux内核使用什么方案来避免申请不到物理内存这个问题呢?

相对于内存来说, 磁盘的容量是非常大的, 所以Linux内核实现了一个叫 `内存交换` 的功能 -- 把某些进程的一些暂时用不到的内存页保存到磁盘中, 然后把物理内存页分配给更紧急的用户使用, 当进程用到时再从磁盘读回到内存中即可. 有了 `内存交换` 功能, 系统可使用的内存就可以远远大于物理内存的容量.

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
`kswapd` 内核线程由一个无限循环组成, 首先通过 `inactive_shortage()` 和 `free_shortage()` 函数判断系统的非活跃页面和空闲物理内存页是否短缺, 如果短缺的话, 那么就调用 `do_try_to_free_pages()` 函数试图释放一些物理内存页.
