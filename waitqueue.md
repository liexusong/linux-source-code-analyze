## `waitqueue` 原理与实现

当进程要获取某些资源（例如从网卡读取数据）的时候，但资源并没有准备好（例如网卡还没接收到数据），这时候内核必须切换到其他进程运行，直到资源准备好再唤醒进程。

`waitqueue (等待队列)` 就是内核用于管理等待资源的进程，当某个进程获取的资源没有准备好的时候，可以通过调用 `add_wait_queue()` 函数把进程添加到 `waitqueue` 中，然后切换到其他进程继续执行。当资源准备好，由资源提供方通过调用 `wake_up()` 函数来唤醒等待的进程。

### `waitqueue` 初始化

要使用 `waitqueue` 首先需要声明一个 `wait_queue_head_t` 结构的变量，`wait_queue_head_t` 结构定义如下：
```cpp
struct __wait_queue_head {
    spinlock_t lock;
    struct list_head task_list;
};
```
`waitqueue` 本质上是一个链表，而 `wait_queue_head_t` 结构是 `waitqueue` 的头部，`lock` 字段用于保护等待队列在多核环境下数据被破坏，而 `task_list` 字段用于保存等待资源的进程列表。

可以通过调用 `init_waitqueue_head()` 函数来初始化 `wait_queue_head_t` 结构，其实现如下：
```cpp
void init_waitqueue_head(wait_queue_head_t *q)
{
    spin_lock_init(&q->lock);
    INIT_LIST_HEAD(&q->task_list);
}
```
初始化过程很简单，首先调用 `spin_lock_init()` 来初始化自旋锁 `lock`，然后调用 `INIT_LIST_HEAD()` 来初始化进程链表。

### 向 `waitqueue` 添加等待进程

要向 `waitqueue` 添加等待进程，首先要声明一个 `wait_queue_t` 结构的变量，`wait_queue_t` 结构定义如下：
```cpp
typedef int (*wait_queue_func_t)(wait_queue_t *wait, unsigned mode, int sync, void *key);

struct __wait_queue {
    unsigned int flags;
    void *private;
    wait_queue_func_t func;
    struct list_head task_list;
};
```
下面说明一下各个成员的作用：
1. `flags`: 可以设置为 `WQ_FLAG_EXCLUSIVE`，表示等待的进程应该独占资源（解决惊群现象）。
2. `private`: 一般用于保存等待进程的进程描述符 `task_struct`。
3. `func`: 唤醒函数，一般设置为 `default_wake_function()` 函数，当然也可以设置为自定义的唤醒函数。
4. `task_list`: 用于连接其他等待资源的进程。

可以通过调用 `init_waitqueue_entry()` 函数来初始化 `wait_queue_t` 结构变量，其实现如下：
```cpp
static inline void init_waitqueue_entry(wait_queue_t *q, struct task_struct *p)
{
    q->flags = 0;
    q->private = p;
    q->func = default_wake_function;
}
```

也可以通过调用 `init_waitqueue_func_entry()` 函数来初始化为自定义的唤醒函数：
```cpp
static inline void init_waitqueue_func_entry(wait_queue_t *q, wait_queue_func_t func)
{
    q->flags = 0;
    q->private = NULL;
    q->func = func;
}
```
