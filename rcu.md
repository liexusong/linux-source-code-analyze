## RCU原理与实现

Linux内核有多种锁机制，比如 `自旋锁`、`信号量` 和 `读写锁` 等。不同的场景使用不同的锁，如在读多写少的场景可以使用读写锁，而在锁粒度比较小的场景可以使用自旋锁。

本文主要介绍一种比较有趣的锁，名为：`RCU`，`RCU` 是 `Read Copy Update` 这几个单词的缩写，中文翻译是 `读 复制 更新`，顾名思义这个锁只需要三个步骤就能完成：__1) 读__、 __2) 复制__、 __3) 更新__。但是往往现实并不是那么美好的，这个锁机制要比这个名字复杂很多。

我们先来介绍一下 `RCU` 的使用场景，`RCU` 的特点是：多个 `reader（读者）` 可以同时读取共享的数据，而 `updater（更新者）` 更新共享的数据时需要复制一份，然后对副本进行修改，修改完把原来的共享数据替换成新的副本，而对旧数据的销毁（释放）等待到所有读者都不再引用旧数据时进行。

`RCU` 与 `读写锁` 一样可以支持多个读者同时访问临界区，并且比 `读写锁` 更为轻量，性能更好。

### RCU 原理

分析下面代码存在的问题（例子参考：《深入理解并行编程》）：
```cpp
struct foo {
    int a;
    char b;
    long c;
 };

DEFINE_SPINLOCK(foo_mutex);

struct foo *gbl_foo;

void foo_read(void)
{
    foo *fp = gbl_foo;
    if (fp != NULL)
        dosomething(fp->a, fp->b, fp->c);
}

void foo_update(foo* new_fp)
{
    spin_lock(&foo_mutex);
    foo *old_fp = gbl_foo;
    gbl_foo = new_fp;
    spin_unlock(&foo_mutex);
    free(old_fp);
}
```
假如有线程A和线程B同时执行 `foo_read()`，而另线程C执行 `foo_update()`，那么会出现以下情况：
1) 线程A和线程B同时读取到旧的 `gbl_foo` 的指针。
2) 线程A和线程B同时读取到新的 `gbl_foo` 的指针。
3) 线程A和线程B有一个读取到新的 `gbl_foo` 的指针，另外一个读取到旧的 `gbl_foo` 的指针。

如果线程A或线程B在读取旧的 `gbl_foo` 数据还没完成时，线程C释放了旧的 `gbl_foo` 指针，那么将会导致程序奔溃。

为了解决这个问题，`RCU` 提出 `宽限期` 的概念。

`宽限期` 是指线程引用旧数据结束前的一段时间，如下图（图片来源：[RCU原理分析](https://www.cnblogs.com/chaozhu/p/6265740.html)）：

![rcu-grace-period](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/rcu-grace-period.png)

如上图所示，线程1、线程2和线程5在删除（替换）旧数据前已经在使用旧数据，所以必须等待它们不再引用旧数据时才能对旧数据进行销毁，这个等待的时间就是 `宽限期`。由于线程3、线程4和线程6使用的是新数据（已经被替换成新的指针），所以不需要等到它们。

> 由于 `RCU` 的读者需要禁止抢占，所以对于 `RCU` 来说，`宽限期` 是所有CPU都进行一次用户态调度的时间。

上面的这段话是什么意思？

禁止抢占代表CPU不能调度到其他线程，CPU只能等待当前线程离开临界区（不再引用旧数据）才能进行调度。也就是说，如果CPU进行了一次调度，说明线程已经不再引用旧数据。如果所有CPU都进行了一次调度，就说明已经没有线程引用旧数据，那么就可以对旧数据进行销毁。

### RCU 使用

> 本文使用的是Linux2.6.0版本的内核。

#### RCU 读者

要做Linux内核中使用 `RCU`，读者需要使用 `rcu_read_lock()` 来对临界区进行 “上锁”，本质上 `rcu_read_lock()` 就是禁止CPU进行抢占，如下代码：
```cpp
#define rcu_read_lock()     preempt_disable()  // 禁止抢占
```

当不再引用数据时，需要使用 `rcu_read_unlock()` 对临界区进行 “解锁”，本质上 `rcu_read_unlock()` 就是开启抢占，如下代码：
```cpp
#define rcu_read_unlock()   preempt_enable()  // 开启抢占
```

例子如下：
```cpp
void foo_read(void)
{
    rcu_read_lock(); // 上锁

    foo *fp = gbl_foo;
    if (fp != NULL)
        dosomething(fp->a, fp->b, fp->c);

    rcu_read_unlock(); // 解锁
}
```

#### RCU 更新者

对于更新者，有两种方式：
1. 调用 `call_rcu()` 异步销毁，由内核自动触发，非阻塞。
2. 调用 `synchronize_kernel()` 同步销毁，等待宽限期过后，阻塞。

例子如下：
```cpp
void foo_update(foo* new_fp)
{
    spin_lock(&foo_mutex);
    foo *old_fp = gbl_foo;
    gbl_foo = new_fp;
    spin_unlock(&foo_mutex);

    synchronize_kernel(); // 等待宽限期过后

    free(old_fp);
}
```

### RCU 实现

在介绍 `RCU` 实现前，先要介绍两个重要的数据结构：`rcu_ctrlblk` 和 `rcu_data`。`rcu_ctrlblk` 结构用于记录当前系统宽限期批次信息，而 `rcu_data` 结构用于记录每个CPU的调度次数与需要延迟执行的函数列表。

#### rcu_ctrlblk 结构
```cpp
struct rcu_ctrlblk {
    spinlock_t  mutex;
    long        curbatch;
    long        maxbatch;
    cpumask_t   rcu_cpu_mask;
};
```
rcu_ctrlblk 结构各个字段的作用：
1. `mutex`：由于 `rcu_ctrlblk` 结构是全局变量，所需通过这个锁来进行同步。
2. `curbatch`：当前批次数（`RCU` 的实现把每个宽限期当成是一个批次）。
3. `maxbatch`：系统最大批次数，如果 `maxbatch` 大于 `curbatch` 说明还有没有完成的批次。
4. `rcu_cpu_mask`：当前批次还没有进行调度的CPU列表，因为前面说过，必须所有CPU进行一次调度宽限期才能算结束。

#### rcu_data 结构
```cpp
struct rcu_data {
    long              qsctr;         /* User-mode/idle loop etc. */
    long              last_qsctr;    /* value of qsctr at beginning */
                                     /* of rcu grace period */
    long              batch;         /* Batch # for current RCU batch */
    struct list_head  nxtlist;
    struct list_head  curlist;
};
```
每个CPU都有一个 `rcu_data` 结构，其各个字段的作用如下：
1. `qsctr`：当前CPU调度的次数。
2. `last_qsctr`：用于记录宽限期开始时的调度次数，如果 `qsctr` 比它大，说明当前CPU的宽限期已经结束。
3. `batch`：用于记录当前CPU的批次数。
4. `nxtlist`：下一次批次要执行的函数列表。
5. `curlist`：当前批次要执行的函数列表。

#### 时钟中断

每次时钟中断都会触发调用 `scheduler_tick()` 函数，而 `scheduler_tick()` 函数会调用 `rcu_check_callbacks()` 函数，调用链：`scheduler_tick() -> rcu_check_callbacks()`。`rcu_check_callbacks()` 函数实现如下：
```cpp
void rcu_check_callbacks(int cpu, int user)
{
    if (user || (idle_cpu(cpu) && !in_softirq() && hardirq_count() <= (1 << HARDIRQ_SHIFT))) {
        RCU_qsctr(cpu)++;
    }
    tasklet_schedule(&RCU_tasklet(cpu)); // 这里会调用 rcu_process_callbacks() 函数
}
```
这个函数主要做两件事情：
1. 判断当前进程是否处于用户态，如果是就对当前CPU的 `rcu_data` 结构的 `qsctr` 字段进行加一操作。前面说过，如果进程处于用户态，代表内核已经推出了临界区，也就是说不再引用旧数据。
2. 调用 `rcu_process_callbacks()` 函数继续进行其他工作。

我们解析来分析 `rcu_process_callbacks()` 函数：
```cpp
static void rcu_process_callbacks(unsigned long unused)
{
    int cpu = smp_processor_id(); // 获取CPU ID
    LIST_HEAD(list);

    // 1. 如果CPU的当前批次有要执行的函数列表, 并且全局批次数大于CPU的当前批次数,
    // 那么把当前批次要执行的函数列表移动到list列表中, 并且清空当前批次要执行的函数列表。
    if (!list_empty(&RCU_curlist(cpu)) &&
        rcu_batch_after(rcu_ctrlblk.curbatch, RCU_batch(cpu))) {
        list_splice(&RCU_curlist(cpu), &list);
        INIT_LIST_HEAD(&RCU_curlist(cpu));
    }

    // 2. 如果CPU下一个批次要执行的函数列表不为空, 并且当前批次要执行的函数列表为空,
    // 那么把下一个批次要执行的函数列表移动到当前批次要执行的函数列表, 并且清空下一个批次要执行的函数列表
    local_irq_disable();
    if (!list_empty(&RCU_nxtlist(cpu)) && list_empty(&RCU_curlist(cpu))) {
        list_splice(&RCU_nxtlist(cpu), &RCU_curlist(cpu));
        INIT_LIST_HEAD(&RCU_nxtlist(cpu));
        local_irq_enable();

        /*
         * start the next batch of callbacks
         */
        spin_lock(&rcu_ctrlblk.mutex);
        RCU_batch(cpu) = rcu_ctrlblk.curbatch + 1; // 把CPU当前批次设置为全局批次数加一
        rcu_start_batch(RCU_batch(cpu)); // 开始新的批次周期(宽限期)
        spin_unlock(&rcu_ctrlblk.mutex);
    } else {
        local_irq_enable();
    }

    // 调用 rcu_check_quiescent_state() 函数检测所有CPU是否都退出临界区（宽限期结束），如果是则对全局批次数进行加一操作。
    rcu_check_quiescent_state();

    // 执行CPU当前批次函数列表的函数
    if (!list_empty(&list))
        rcu_do_batch(&list);
}
```
`rcu_process_callbacks()` 函数主要完成4件事情：
1. 如果CPU的当前批次（`rcu_data` 结构的 `curlist` 字段）有要执行的函数列表（一般都是销毁旧数据的函数），并且全局批次数大于CPU的当前批次数。那么把列表移动到 `list` 中，并且清空当前批次的函数列表。
2. 如果CPU的下一次批次（`rcu_data` 结构的 `nxtlist` 字段）有要执行的函数列表，并且当前批次要执行的函数列表为空。那么把其移动到当前批次的函数列表，并清空下一次批次的函数列表。接着把CPU的当前批次数设置为全局批次数加一，然后调用 `rcu_start_batch()` 函数开始一个新的批次周期（宽限期）。
3. 调用 `rcu_check_quiescent_state()` 函数检测所有CPU是否都退出临界区（宽限期结束），如果是则对全局批次数进行加一操作。
4. 如果CPU当前批次执行的函数列表不为空，那么就执行函数列表中的函数。

从上面的代码可知，每个CPU的当前批次要执行的函数列表必须等待全局批次数大于当前CPU的批次数才能被执行。全局批次数由 `rcu_check_quiescent_state()` 函数推动，我们来看看 `rcu_check_quiescent_state()` 函数的实现：
```cpp
static void rcu_check_quiescent_state(void)
{
    int cpu = smp_processor_id();

    // 如果当前CPU不在rcu_cpu_mask列表中，表示当前CPU已经经历了一次调用，所有不需要再执行下去
    if (!cpu_isset(cpu, rcu_ctrlblk.rcu_cpu_mask))
        return;

    if (RCU_last_qsctr(cpu) == RCU_QSCTR_INVALID) { // 宽限期开始记录点
        RCU_last_qsctr(cpu) = RCU_qsctr(cpu);
        return;
    }
    if (RCU_qsctr(cpu) == RCU_last_qsctr(cpu)) // 还没有被调度过，直接返回
        return;

    spin_lock(&rcu_ctrlblk.mutex);
    if (!cpu_isset(cpu, rcu_ctrlblk.rcu_cpu_mask))
        goto out_unlock;

    // 当前CPU已经被调度了，清空其所在rcu_cpu_mask列表的标志位
    cpu_clear(cpu, rcu_ctrlblk.rcu_cpu_mask);
    RCU_last_qsctr(cpu) = RCU_QSCTR_INVALID;

    // 如果还有CPU没有经历一次调度，直接返回
    if (!cpus_empty(rcu_ctrlblk.rcu_cpu_mask))
        goto out_unlock;

    // 执行到这里表示所有CPU都执行了一次调度
    rcu_ctrlblk.curbatch++; // 全局批次数加一
    rcu_start_batch(rcu_ctrlblk.maxbatch); // 开始下一轮批次

out_unlock:
    spin_unlock(&rcu_ctrlblk.mutex);
}
```
`rcu_check_quiescent_state()` 函数主要做以下几件事情：
1. 判断当前CPU是否已经经历一次调度，如果是，就把其从 `rcu_ctrlblk.rcu_cpu_mask` 列表中清除。
2. 如果所有CPU都经历了一次调度，那么就对全局批次数进行加一操作。
3. 开始下一轮的批次周期。

从上面的分析可以看出，推动 `RCU` 对旧数据进行销毁的动力是时钟中断。

`call_rcu()` 函数会把销毁函数添加到当前CPU的 `nxtlist` 函数列表中，代码如下：
```cpp
void call_rcu(struct rcu_head *head, void (*func)(void *arg), void *arg)
{
    int cpu;
    unsigned long flags;

    head->func = func;
    head->arg = arg;
    local_irq_save(flags);
    cpu = smp_processor_id();
    list_add_tail(&head->list, &RCU_nxtlist(cpu)); // 添加到CPU的nxtlist列表中
    local_irq_restore(flags);
}
```
随着时钟中断的推动，`nxtlist` 函数列表会移动到 `curlist` 函数列表中，然后会在适当的时机执行 `curlist` 函数列表中的函数。
