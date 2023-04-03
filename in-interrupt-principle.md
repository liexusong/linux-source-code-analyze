# in_interrupt() 原理

`in_interrupt()` 宏主要用于判断当前执行上下文是否处于中断上下文中，其定义如下：

```c

#define in_interrupt()  irq_count()
#define irq_count()     (preempt_count() & (0x0FFF0000 | 0x0000FF00))
#define preempt_count() (current_thread_info()->preempt_count)

```

所以最终展开如下：

```c
#define in_interrupt()  ((current_thread_info()->preempt_count) & (0x0FFFFF00))
```

其中 `current_thread_info()` 函数会返回一个类型为 `struct thread_info` 的指针，此指针指向当前线程的信息，`struct thread_info` 结构定义如下：

```c
struct thread_info {
    struct task_struct *task; /* main task structure */
    ...
    unsigned long flags;      /* low level flags */
    unsigned long status;     /* thread-synchronous flags */
    __u32 cpu;                /* current CPU */
    int preempt_count;        /* 0 => preemptable, <0 => BUG */
    ...
};
```

`struct thread_info` 结构中的 `preempt_count` 字段有 3 个用途：

* 记录当前执行上下文是否处于硬中断上下文中。
* 记录当前执行上下文是否处于软中断上下文中。
* 记录当前执行上下文是否禁止内核抢占。

由于 `preempt_count` 字段的类型为 `整型`，所以内核将其分为3个部分，如下图所示：

```text

         hardirq   softirq  preempt
      /          \/       \/       \
+====+============+========+========+
|....|............|........|........|
+====+============+========+========+
31                                  0

```

* 内核抢占计数器占用 `preempt_count` 字段的 `0 ~ 7` 位。
* 软中断中断计数器占用 `preempt_count` 字段的 `8 ~ 15` 位。
* 硬中断中断计数器占用 `preempt_count` 字段的 `16 ~ 27` 位。

当外部设备发生中断时，内核会调用 `irq_enter()` 宏来标记进入硬中断上下文，`irq_enter()` 宏展开后如下所示：

```c
#define irq_enter() \
    do { current_thread_info()->preempt_count += (1UL << 16); } while (0)
```

从上面代码可以看出，`irq_enter()` 宏就是增加硬中断计数器的值。
