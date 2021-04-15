# 内核工作队列

在某些情景下内核会将一些耗时的、可延迟的工作放到 `工作队列` 中，内核会在适当的时机处理 `工作队列` 中的工作，就像应用层开发的 `消息队列` 一样。

## 一、工作队列对象

内核使用 `workqueue_struct` 对象来存储要延迟执行的工作，其定义如下：

```c
struct workqueue_struct {
    struct cpu_workqueue_struct *cpu_wq; // 真正存储延迟执行工作的地方, 每个拥有CPU一个
    struct list_head list;               // 用于把内核中所有的工作队列连接起来
    const char *name;                    // 工作队列的名字
    int singlethread;                    // 是否只启动一个工作线程
    ...
};
```

在 `workqueue_struct` 对象中，`cpu_wq` 字段才是真正存储延迟执行工作的地方，其类型为 `cpu_workqueue_struct`，我们来看看 `cpu_workqueue_struct` 的定义：

```c
struct cpu_workqueue_struct {
    spinlock_t lock;                   // 用于锁定工作队列
    struct list_head worklist;         // 存储延迟执行工作的队列
    wait_queue_head_t more_work;       // 用于唤醒工作队列线程
    struct work_struct *current_work;  // 当前正在执行的工作
    struct workqueue_struct *wq;       // 指向工作队列的指针
    struct task_struct *thread;        // 执行工作队列的进程
    ...
} ____cacheline_aligned;
```

`workqueue_struct` 与 `cpu_workqueue_struct` 之间的关系如下图所示：

![](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/workqueue/workqueue.png)







