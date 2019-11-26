## 中断处理 - 上半部

由于 `APIC中断控制器` 有点小复杂，所以本文主要通过 `8259A中断控制器` 来介绍Linux对中断的处理过程。

前面说过，`8259A中断控制器` 由两片 8259A 风格的外部芯片以 `级联` 的方式连接在一起，每个芯片可处理多达 8 个不同的 IRQ（中断请求），所以可用 IRQ 线的个数达到 15 个。如下图：

![8259A](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/8259A.png)

在内核中每条IRQ线由结构体 `irq_desc_t` 来描述，`irq_desc_t` 定义如下：
```c
typedef struct {
    unsigned int status;        /* IRQ status */
    hw_irq_controller *handler;
    struct irqaction *action;   /* IRQ action list */
    unsigned int depth;         /* nested irq disables */
    spinlock_t lock;
} irq_desc_t;
```
下面介绍一下 `irq_desc_t` 结构各个字段的作用：
* `status`: IRQ线的状态。
* `handler`: 类型为 `hw_interrupt_type` 结构，表示IRQ线对应的硬件相关处理函数，比如 `8259A中断控制器` 接收到一个中断信号时，需要发送一个确认信号才会继续接收中断信号的，发送确认信号的函数就是 `hw_interrupt_type` 中的 `ack` 函数。
* `action`: 类型为 `irqaction` 结构，中断信号的处理入口。由于一条IRQ线可以被多个硬件共享，所以 `action` 是一个链表，每个 `action` 代表一个硬件的中断处理入口。
* `depth`: 防止多次开启和关闭IRQ线。
* `lock`: 防止多核CPU同时对IRQ进行操作的自旋锁。

`hw_interrupt_type` 这个结构与硬件相关，这里就不作介绍了，我们来看看 `irqaction` 这个结构：
```c
struct irqaction {
    void (*handler)(int, void *, struct pt_regs *);
    unsigned long flags;
    unsigned long mask;
    const char *name;
    void *dev_id;
    struct irqaction *next;
};
```
下面说说 `irqaction` 结构各个字段的作用：
* `handler`: 中断处理的入口函数，`handler` 的第一个参数是中断号，第二个参数是设备对应的ID，第三个参数是中断发生时由内核保存的各个寄存器的值。
* `next`: 每个硬件的中断处理入口对应一个 `irqaction` 结构，由于多个硬件可以共享同一条IRQ线，所以这里通过 `next` 字段来连接不同的硬件中断处理入口。

`irq_desc_t` 结构关系如下图：

![irq_desc_t](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/irq_desc_t.jpg)

