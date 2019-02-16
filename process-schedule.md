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

但最终也会调用以下的汇编代码:
```gas
ENTRY(ret_from_sys_call)
#ifdef CONFIG_SMP
    movl processor(%ebx),%eax
    shll $CONFIG_X86_L1_CACHE_SHIFT,%eax
    movl SYMBOL_NAME(irq_stat)(,%eax),%ecx
    testl SYMBOL_NAME(irq_stat)+4(,%eax),%ecx
#else
    movl SYMBOL_NAME(irq_stat),%ecx
    testl SYMBOL_NAME(irq_stat)+4,%ecx
#endif
    jne   handle_softirq
    
ret_with_reschedule:
    cmpl $0,need_resched(%ebx)
    jne reschedule
    cmpl $0,sigpending(%ebx)
    jne signal_return
restore_all:
    RESTORE_ALL
```
