## 进程调度
在Linux内核中通常有几十或者上百个进程在运行, 但个人电脑的CPU一般也只有双核或者四核, CPU的一个核在某一时刻只能运行一个进程, 所以有四个核的CPU只能同时运行4个进程, 那么Linux内核怎么可以运行比CPU核数多的进程呢? 这里就涉及到一个叫 `进程运行时间片` 的概念.

`进程运行时间片` 是让每个进程在CPU中运行一段指定的时间(时间片), 当某一个进程的时间片用完后, 由Linux内核切换到其他时间片还没用完的进程运行. 进程管理结构 `task_struct` 中有个保存着时间片的字段 `counter`, 如下:
```cpp
struct task_struct {
    ...
    volatile long need_resched;
    ...
    long counter;
    ...
};
```

### 时钟中断
有了时间片的概念后, 进程就不能为所欲为的占用CPU了. 但这里有个问题, 就是进程的时间片不会自己减少的, 那么应该由谁来将进程的时间片减少呢? 答案就是 `时钟中断` 程序(`中断处理` 在后面会介绍, 所以这里不会对 `中断处理` 作详细的介绍).

`时钟中断` 是指每隔一段相同的时间, 都会发出一个中断信号(称为一个tick, 由8253芯片触发), CPU接受到中断信号后触发内核中相应的中断处理程序. 当 `时钟中断` 发生时会调用 `timer_interrupt()` 函数来处理中断, `timer_interrupt()` 函数源码如下:
```cpp
static void timer_interrupt(int irq, void *dev_id, struct pt_regs *regs)
{
    int count;

    write_lock(&xtime_lock);

    ...
    do_timer_interrupt(irq, NULL, regs);

    write_unlock(&xtime_lock);
}

static inline void do_timer_interrupt(int irq, void *dev_id, struct pt_regs *regs)
{
    ...
    do_timer(regs);
    ...
    if ((time_status & STA_UNSYNC) == 0 &&
        xtime.tv_sec > last_rtc_update + 660 &&
        xtime.tv_usec >= 500000 - ((unsigned) tick) / 2 &&
        xtime.tv_usec <= 500000 + ((unsigned) tick) / 2) {
        if (set_rtc_mmss(xtime.tv_sec) == 0)
            last_rtc_update = xtime.tv_sec;
        else
            last_rtc_update = xtime.tv_sec - 600;
    }
    ...
}
```
从上面的代码可以看到, `timer_interrupt()` 函数会调用 `do_timer_interrupt()` 函数, 而 `do_timer_interrupt()` 函数最终会调用 `do_timer()`, `do_timer()` 函数是时钟中断处理的主要逻辑, 源码如下:
```cpp
void do_timer(struct pt_regs *regs)
{
    (*(unsigned long *)&jiffies)++;
#ifndef CONFIG_SMP
    /* SMP process accounting uses the local APIC timer */
    update_process_times(user_mode(regs));
#endif
    mark_bh(TIMER_BH);
    if (TQ_ACTIVE(tq_timer))
        mark_bh(TQUEUE_BH);
}
```
`do_timer()` 函数主要调用 `update_process_times()` 函数更新进程的时间片, 代码如下:
```cpp
void update_process_times(int user_tick)
{
    struct task_struct *p = current;
    ...
    if (p->pid) {
        if (--p->counter <= 0) {
            p->counter = 0;
            p->need_resched = 1;
        }
        ...
    }
    ...
}
```
从上面的代码可以看出, 每次时钟中断发生都会将当前进程的时间片减一, 当时间片用完后会设置进程的 `need_resched` 字段为1(表示需要调用当前进程).

这里有个问题, 就是时钟中断只是把进程的 `need_resched` 字段设置为1而已, 并没有对进程进行调度啊, 那么什么时候才会对进程进行调度呢? 答案是从内核态返回到用户态的时候.

从内核态返回到用户态有几个时机: 
1. 中断处理完成后返回. 
2. 异常处理完成后返回. 
3. 系统调用完成后返回.

譬如, 当用户进程调用系统调用返回时, 会调用以下的汇编代码:
```asm
ENTRY(ret_from_sys_call)
    ...
ret_with_reschedule:
    cmpl $0,need_resched(%ebx)  // 判断当前进程的 need_resched 字段是否为1
    jne reschedule              // 如果是, 就跳到reschedule处执行
    cmpl $0,sigpending(%ebx)
    jne signal_return
restore_all:
    RESTORE_ALL                 // 返回到用户空间

reschedule:
    call SYMBOL_NAME(schedule)  // 调用 schedule() 函数进行进程的调度
    jmp ret_from_sys_call
```
由于是汇编写的, 所以有点难懂, 所以这里我大概说说这段代码的流程:
1. 首先判断当前进程的 `need_resched` 字段是否为1.
2. 如果进程的 `need_resched` 为1, 那么就调用 `schedule()` 函数进行进程的调度.
3. 调用完 `schedule()` 函数后, 继续返回到 `ret_from_sys_call` 处执行.

### schedule()函数
现在我们来分析一下 `schedule()` 这个函数, 由于这个函数比较长, 所以我们分段来分析这个函数:
```cpp
asmlinkage void schedule(void)
{
    struct schedule_data * sched_data;
    struct task_struct *prev, *next, *p;
    struct list_head *tmp;
    int this_cpu, c;

    ...
    prev = current;
    ...

    spin_lock_irq(&runqueue_lock);

    if (prev->policy == SCHED_RR)
        goto move_rr_last;
move_rr_back:

    switch (prev->state) {
        case TASK_INTERRUPTIBLE:
            if (signal_pending(prev)) {
                prev->state = TASK_RUNNING;
                break;
            }
        default:
            del_from_runqueue(prev);
        case TASK_RUNNING:
    }
    prev->need_resched = 0;
```
上面的代码首先判断当前进程是否可中断休眠状态, 并且接受到信号, 如果是就唤醒当前进程. 如果当前进程是休眠状态, 那么就把当前进程从运行队列中删除. 接着把当前进程的 `need_resched` 字段设置为0.

```cpp
repeat_schedule:
    next = idle_task(this_cpu);
    c = -1000;
    if (prev->state == TASK_RUNNING)
        goto still_running;

still_running_back:
    list_for_each(tmp, &runqueue_head) {
        p = list_entry(tmp, struct task_struct, run_list);
        if (can_schedule(p, this_cpu)) {
            int weight = goodness(p, this_cpu, prev->active_mm);
            if (weight > c)
                c = weight, next = p;
        }
    }
```
这段代码是便利运行队列中的所有进程, 然后通过调用 `goodness()` 函数来计算每个进程的运行优先级, 值越大就越先被运行, 找到的进程会被保存到 `next` 变量中. 我们来看看 `goodness()` 的计算过程:
```cpp
static inline int goodness(struct task_struct * p, int this_cpu, struct mm_struct *this_mm)
{
    int weight;

    weight = -1;
    if (p->policy & SCHED_YIELD)
        goto out;

    if (p->policy == SCHED_OTHER) { // 普通进程
        weight = p->counter;
        if (!weight)
            goto out;

        if (p->mm == this_mm || !p->mm)
            weight += 1;
        weight += 20 - p->nice;
        goto out;
    }

    weight = 1000 + p->rt_priority;
out:
    return weight;
}
```
计算过程很简单, 首先进程在Linux内核中分为实时进程和普通进程, 普通进程的计算方法就是:

    进程时间片 + 20 - 进程的友好值

而实时进程的计算方法是:

    1000 + 实时进程的优先级

我们继续来分析 `schedule()` 函数的余下部分:
```cpp
    prepare_to_switch();
    {
        // 切换进程的内存空间
        struct mm_struct *mm = next->mm;
        struct mm_struct *oldmm = prev->active_mm;
        if (!mm) {
            if (next->active_mm) BUG();
            next->active_mm = oldmm;
            atomic_inc(&oldmm->mm_count);
            enter_lazy_tlb(oldmm, next, this_cpu);
        } else {
            if (next->active_mm != mm) BUG();
            switch_mm(oldmm, mm, next, this_cpu);
        }

        if (!prev->mm) {
            prev->active_mm = NULL;
            mmdrop(oldmm);
        }
    }

    switch_to(prev, next, prev);
    __schedule_tail(prev);
```
找到合适的进程后, 接下来就是进行调度工作了. 调度工作首先调用 `switch_mm()` 函数来把旧进程的内存空间切换到新进程的内存空间, 切换内存空间主要是通过把 `cr3` 寄存器的值设置为新进程页目录的地址. 接着调用 `switch_to()` 函数进行进程的切换, 我们来看看 `switch_to()` 函数的实现:
```cpp
#define switch_to(prev,next,last) do {                          \
    asm volatile("pushl %%esi\n\t"                              \
             "pushl %%edi\n\t"                                  \
             "pushl %%ebp\n\t"                                  \
             "movl %%esp,%0\n\t"    /* save ESP */              \
             "movl %3,%%esp\n\t"    /* restore ESP */           \
             "movl $1f,%1\n\t"      /* save EIP */              \
             "pushl %4\n\t"         /* restore EIP */           \
             "jmp __switch_to\n"                                \
             "1:\t"                                             \
             "popl %%ebp\n\t"                                   \
             "popl %%edi\n\t"                                   \
             "popl %%esi\n\t"                                   \
             :"=m" (prev->thread.esp),"=m" (prev->thread.eip),  \
              "=b" (last)                                       \
             :"m" (next->thread.esp),"m" (next->thread.eip),    \
              "a" (prev), "d" (next),                           \
              "b" (prev));                                      \
} while (0)
```
又是一段难懂的汇编, 而且是比汇编更难懂的GCC嵌入汇编. 为了让大家不陷入痛苦之中, 这里主要介绍一下这段代码的作用. 在 `进程管理` 一节中, 我们介绍过进程管理结构 `task_struct` 是放置在内核栈的底部的, 所以要切换进程只需要切换内核栈即可. 这里正是通过这个方法来切换进程的, 我们看到的 `movl %3, %%esp` 这行代码就是切换到新进程的内核栈.

当调用完 `schedule()` 函数后, 现在通过 `get_current()` 函数获取到的当前进程就是我们刚才切换的新进程了, 至此进程切换完成.
