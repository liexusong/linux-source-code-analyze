# in_interrupt() 原理

`in_interrupt()` 宏主要用于判断当前执行上下文是否处于中断上下文中，其定义如下：

```c

#define in_interrupt()  irq_count()
#define irq_count()     (preempt_count() & (HARDIRQ_MASK | SOFTIRQ_MASK))
#define preempt_count() (current_thread_info()->preempt_count)

```

所以最终展开如下：

```c
#define in_interrupt()  ((current_thread_info()->preempt_count) & (HARDIRQ_MASK | SOFTIRQ_MASK))
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




