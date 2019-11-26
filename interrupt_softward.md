## Linux中断处理

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

struct hw_interrupt_type {
    const char * typename;
    unsigned int (*startup)(unsigned int irq);
    void (*shutdown)(unsigned int irq);
    void (*enable)(unsigned int irq);
    void (*disable)(unsigned int irq);
    void (*ack)(unsigned int irq);
    void (*end)(unsigned int irq);
    void (*set_affinity)(unsigned int irq, unsigned long mask);
};

struct irqaction {
    void (*handler)(int, void *, struct pt_regs *);
    unsigned long flags;
    unsigned long mask;
    const char *name;
    void *dev_id;
    struct irqaction *next;
};
```
