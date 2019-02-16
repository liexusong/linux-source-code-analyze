## 进程调度
在Linux内核中通常有几十或者上百个进程在运行, 但个人电脑的CPU一般也只有双核或者四核, CPU的一个核在某一时刻只能运行一个进程, 所以有四个核的CPU只能同时运行4个进程, 那么Linux内核怎么可以运行比CPU核数多的进程呢? 这里就涉及到一个叫 `进程运行时间片` 的概念.

`进程运行时间片` 是让每个进程在CPU中运行一段指定的时间(时间片), 当某一个进程的时间片用完后, 由Linux内核切换到其他时间片还没用完的进程运行. 进程管理结构 `task_struct` 中有个保存着时间片的字段 `counter`, 如下:
```cpp
struct task_struct {
    ...
    long counter;
    ...
};
```

### 时钟中断
有了时间片的概念后, 进程就不能为所欲为的占用CPU了. 但这里有个问题, 就是进程的时间片不会自己减少的, 那么应该由谁来将进程的时间片减少呢? 答案就是 `时钟中断` 程序(`中断处理` 在后面会介绍, 所以这里不会对 `中断处理` 作详细的介绍).

`时钟中断` 是指每隔一段相同的时间, 都会发出一个中断信号(由8253芯片触发), CPU接受到中断信号后触发内核中相应的中断处理程序. 当 `时钟中断` 发生时会调用 `timer_interrupt()` 函数来处理中断, `timer_interrupt()` 函数源码如下:
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
