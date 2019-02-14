## 内存交换
由于计算机的物理内存是有限的, 而进程对内存的使用是不确定的, 所以物理内存总有用完的可能性. 那么当系统的物理内存不足时, Linux内核使用什么方案来避免申请不到物理内存这个问题呢?

相对于内存来说, 磁盘的容量是非常大的, 所以Linux内核实现了一个叫 `内存交换` 的功能 -- 把某些进程的一些暂时用不到的内存页保存到磁盘中, 然后把物理内存页分配给更紧急的用户使用, 当进程用到时再从磁盘读回到内存中即可. 有了 `内存交换` 功能, 系统可使用的内存就可以远远大于物理内存的容量.

### LRU算法
`内存交换` 过程首先是找到一个合适的用户进程内存管理结构，然后把进程占用的内存页交换到磁盘中，并断开虚拟内存与物理内存的映射，最后释放进程占用的内存页。由于涉及到IO操作，所以这是一个比较耗时的过程。如果被交换出去的内存页刚好又被访问了，这时又需要从磁盘中把内存页的数据交换到内存中。所以，在这种情况下不单不能解决内存紧缺的问题，而且增加了系统的负荷。

为了解决这个问题，Linux内核使用了一种称为 `LRU (Least Recently Used)` 的算法, 下面介绍一下 `LRU算法` 的大体过程.

`LRU` 的中文翻译是 `最近最少使用`, 顾名思义就是一段时间内没有被使用, 那么Linux内核怎么知道哪些内存页面最近没有被使用呢? 最简单的方法就是把内存页放进一个队列里, 如果内存页被访问了, 就把内存页移动到链表的头部, 这样没被访问的内存页在一段时间后便会移动到队列的尾部, 而释放内存页时从链表的尾部开始. 著名的缓存服务器 `memcached` 就是使用这种 `LRU算法`.

Linux内核也使用了类似的算法, 但相对要复杂一些. Linux内核维护着三个队列: 活跃队列, 非活跃脏队列和非活跃干净队列. 为什么Linux需要维护三个队列, 而不是使用一个队列呢? 这是因为Linux希望内存页交换过程慢慢进行, Linux内核有个内核线程 `kswapd` 会定时检查系统的空闲内存页是否紧缺, 如果系统的空闲内存页紧缺时时, 就会选择一些用户进程把其占用的内存页添加到活跃链表中并断开进程与此内存页的映射关系. 随着时间的推移, 如果内存页没有被访问, 那么就会被移动到非活跃脏链表. 非活跃脏链表中的内存页是需要被交换到磁盘的, 当系统中空闲内存页紧缺时就会从非活跃脏链表的尾部开始把内存页刷新到磁盘中, 然后移动到非活跃干净链表中, 非活跃干净链表中的内存页是可以立刻分配给进程使用的. 各个链表之间的移动如下图:

![lru links](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/memory_lru.jpg)

如果在这个过程中, 内存页又被访问了, 那么Linux内核会把内存页移动到活跃链表中, 并且建立内存映射关系, 这样就不需要从磁盘中读取内存页的内容.

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
`do_try_to_free_pages()` 函数第一步先判断系统中的空闲物理内存页是否短缺, 或者非活跃脏页面的数量大于空闲物理内存页和非活跃干净页面的总和, 其中一个条件满足了, 就调用 `page_launder()` 函数把非活跃脏链表中的页面刷到磁盘中, 然后移动到非活跃干净链表中. 接下来如果内存还是紧缺的话, 那么就调用 `shrink_dcache_memory()`, `shrink_icache_memory()` 和 `refill_inactive()` 函数继续释放内存.

下面我们先来分析一下 `page_launder()` 这个函数, 由于这个函数很长, 所以我们分段来解释:
```cpp
int page_launder(int gfp_mask, int sync)
{
    int launder_loop, maxscan, cleaned_pages, maxlaunder;
    int can_get_io_locks;
    struct list_head * page_lru;
    struct page * page;

    can_get_io_locks = gfp_mask & __GFP_IO; // 是否需要进行写盘操作

    launder_loop = 0;
    maxlaunder = 0;
    cleaned_pages = 0;

dirty_page_rescan:
    spin_lock(&pagemap_lru_lock);
    maxscan = nr_inactive_dirty_pages;
    // 从非活跃脏链表的后面开始扫描
    while ((page_lru = inactive_dirty_list.prev) != &inactive_dirty_list &&
                maxscan-- > 0) {
        page = list_entry(page_lru, struct page, lru);
    ...
```
上面的代码首先把 `pagemap_lru_lock` 上锁, 然后从尾部开始遍历非活跃脏链表.
```cpp
        // 如果满足以下的任意一个条件, 都表示内存页在使用中, 把他移动到活跃链表
        if (PageTestandClearReferenced(page) ||             // 如果设置了 PG_referenced 标志
                page->age > 0 ||                            // 如果age大于0, 表示页面被访问过
                (!page->buffers && page_count(page) > 1) || // 如果页面被其他进程映射
                page_ramdisk(page)) {                       // 如果用于内存磁盘的页面
            del_page_from_inactive_dirty_list(page);
            add_page_to_active_list(page);
            continue;
        }
```
上面代码判断内存页是否能需要重新移动到活跃链表中, 依据有: 
* 内存页是否设置了 `PG_referenced` 标志; 
* 内存页的age字段是否大于0 (age字段是内存页的生命周期); 
* 内存页是否还有映射关系; 
* 内存页是否用于内存磁盘. 

如果满足上面其中一个条件, 都需要重新把内存页移动到活跃页面中.
```cpp
        if (PageDirty(page)) { // 如果页面是脏的, 那么应该把页面写到磁盘中
            int (*writepage)(struct page *) = page->mapping->a_ops->writepage;
            int result;

            if (!writepage)
                goto page_active;

            /* First time through? Move it to the back of the list */
            if (!launder_loop) { // 第一次只把页面移动到链表的头部, 这是为了先处理已经干净的页面
                list_del(page_lru);
                list_add(page_lru, &inactive_dirty_list);
                UnlockPage(page);
                continue;
            }

            /* OK, do a physical asynchronous write to swap.  */
            ClearPageDirty(page);
            page_cache_get(page);
            spin_unlock(&pagemap_lru_lock);

            result = writepage(page);
            page_cache_release(page);

            /* And re-start the thing.. */
            spin_lock(&pagemap_lru_lock);
            if (result != 1)
                continue;
            /* writepage refused to do anything */
            set_page_dirty(page);
            goto page_active;
        }
```
上面的代码首先判断内存页是否脏的(是否设置了 `PG_dirty` 标志), 如果是, 那么就需要把内存页刷新到磁盘中. 这里有个要主要的地方是, 当 `launder_loop` 变量为0时只是把内存页移动到非活跃脏链表的头部. 当 `launder_loop` 变量为1时才会把内存页刷新到磁盘中. 为什么要这样做呢? 这是因为Linux内核希望第一次扫描先把非活跃脏链表中的干净内存页移动到非活跃干净链表中, 第二次扫描才把脏的内存页刷新到磁盘中. 后面的代码会对 `launder_loop` 变量进行修改. 而且我们发现, 把脏页面刷新到磁盘后, 并没有立刻把内存页移动到非活跃干净链表中, 而是简单的清除了 `PG_dirty` 标志.
```cpp
        if (page->buffers) { // 涉及文件系统部分, 先略过
            ...
        } else if (page->mapping && !PageDirty(page)) { // 内存页是干净的, 移动到非活跃干净链表
            del_page_from_inactive_dirty_list(page);
            add_page_to_inactive_clean_list(page);
            UnlockPage(page);
            cleaned_pages++;
        } else {
page_active:
            del_page_from_inactive_dirty_list(page);
            add_page_to_active_list(page);
            UnlockPage(page);
        }
```
上面的代码比较简单, 如果内存页已经是干净的, 那么久移动到非活跃干净链表中.
```cpp
    if (can_get_io_locks && !launder_loop && free_shortage()) {
        launder_loop = 1;
        /* If we cleaned pages, never do synchronous IO. */
        if (cleaned_pages)
            sync = 0;
        /* We only do a few "out of order" flushes. */
        maxlaunder = MAX_LAUNDER;
        /* Kflushd takes care of the rest. */
        wakeup_bdflush(0);
        goto dirty_page_rescan;
    }

    /* Return the number of pages moved to the inactive_clean list. */
    return cleaned_pages;
}
```
