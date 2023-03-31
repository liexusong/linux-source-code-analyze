# in_interrupt() 原理

`in_interrupt()` 宏主要用于判断当前执行上下文是否处于中断上下文中，其定义最终展开如下：

```c

#define in_interrupt() irq_count()
#define irq_count() (preempt_count() & (HARDIRQ_MASK | SOFTIRQ_MASK))
#define preempt_count() (current_thread_info()->preempt_count)

```
