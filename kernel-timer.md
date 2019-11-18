# Linux定时器实现

一般定时器实现的方式有以下几种：

#### 基于排序链表方式：
通过排序链表来保存定时器，由于链表是排序好的，所以获取最小（最早到期）的定时器的时间复杂度为 `O(1)`。但插入需要遍历整个链表，所以时间复杂度为 `O(n)`。如下图：

![timer-list](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/timer-list.jpg)

#### 基于最小堆方式：
通过最小堆来保存定时器，在最小堆中获取最小定时器的时间复杂度为 `O(1)`，但插入一个定时器的时间复杂度为 `O(log n)`。如下图：

![timer-heap](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/timer-heap.jpg)

#### 基于平衡二叉树方式：
使用平衡二叉树（如红黑树）保存定时器，在平衡二叉树中获取最小定时器的时间复杂度为 `O(log n)`（也可以通过缓存最小值的方法来达到 `O(1)`），而插入一个定时器的时间复杂度为 `O(log n)`。如下图：

![timer-tree](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/timer-tree.jpg)

### 时间轮：
但对于Linux这种对定时器依赖性比较高（网络子模块的TCP协议使用了大量的定时器）的操作系统来说，以上的数据结构都是不能满足要求的。所以Linux使用了效率更高的定时器算法：__时间轮__。

__时间轮__ 类似于日常生活的时钟，如下图：

![timer](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/timer.jpg)

日常生活的时钟，每当秒针转一圈时，分针就会走一格，而分针走一圈时，时针就会走一格。而时间轮的实现方式与时钟类似，就是把到期时间当成一个轮，然后把定时器挂在这个轮子上面，每当时间走一秒就移动时针，并且执行那个时针上的定时器，如下图：

![timer-wheel](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/timer-Wheel.jpg)

一般的定时器范围为一个32位整型的大小，也就是 `0 ~ 4294967295`，如果通过一个数组来存储的话，就需要一个元素个数为4294967296的数组，非常浪费内存。这个时候就可以通过类似于时钟的方式：通过多级数组来存储。时钟通过时分秒来进行分级，当然我们也可以这样，但对于计算机来说，时分秒的分级不太友好，所以Linux内核中，对32位整型分为5个级别，第一个等级存储`0 ~ 255秒` 的定时器，第二个等级为 `256秒 ~ 256*64秒`，第三个等级为 `256*64秒 ~ 256*64*64秒`，第四个等级为 `256*64*64秒 ~ 256*64*64*64秒`，第五个等级为 `256*64*64*64秒 ~ 256*64*64*64*64秒`。如下图：

![timer-vts](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/timer-vts.jpg)

> 注意：第二级至第五级数组的第一个槽是不挂任何定时器的。

每级数组上面都有一个指针，指向当前要执行的定时器。每当时间走一秒，Linux首先会移动第一级的指针，然后执行当前位置上的定时器。当指针变为0时，会移动下一级的指针，并把该位置上的定时器重新计算一次并且插入到时间轮中，其他级如此类推。如下图所示：

![timer-vts-pointer](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/timer-vts-pointer.jpg)

当要执行到期的定时器只需要移动第一级数组上的指针并且执行该位置上的定时器列表即可，所以时间复杂度为 `O(1)`，而插入一个定时器也很简单，先计算定时器的过期时间范围在哪一级数组上，并且连接到该位置上的链表即可，时间复杂度也是 `O(1)`。

## Linux时间轮的实现
那么接下来我们看看Linux内核是怎么实现时间轮算法的。

#### 定义五个等级的数组
```c
#define TVN_BITS 6
#define TVR_BITS 8
#define TVN_SIZE (1 << TVN_BITS)  // 64
#define TVR_SIZE (1 << TVR_BITS)  // 256
#define TVN_MASK (TVN_SIZE - 1)
#define TVR_MASK (TVR_SIZE - 1)

struct timer_vec {
    int index;
    struct list_head vec[TVN_SIZE];
};

struct timer_vec_root {
    int index;
    struct list_head vec[TVR_SIZE];
};

static struct timer_vec tv5;
static struct timer_vec tv4;
static struct timer_vec tv3;
static struct timer_vec tv2;
static struct timer_vec_root tv1;

void init_timervecs (void)
{
    int i;

    for (i = 0; i < TVN_SIZE; i++) {
        INIT_LIST_HEAD(tv5.vec + i);
        INIT_LIST_HEAD(tv4.vec + i);
        INIT_LIST_HEAD(tv3.vec + i);
        INIT_LIST_HEAD(tv2.vec + i);
    }
    for (i = 0; i < TVR_SIZE; i++)
        INIT_LIST_HEAD(tv1.vec + i);
}
```
上面的代码定义第一级数组为 `timer_vec_root` 类型，其 `index` 成员是当前要执行的定时器指针（对应 `vec` 成员的下标），而 `vec` 成员是一个链表数组，数组元素个数为256，每个元素上保存了该秒到期的定时器列表，其他等级的数组类似。

#### 插入定时器
```c
static inline void internal_add_timer(struct timer_list *timer)
{
    /*
     * must be cli-ed when calling this
     */
    unsigned long expires = timer->expires;
    unsigned long idx = expires - timer_jiffies;
    struct list_head * vec;

    if (idx < TVR_SIZE) { // 0 ~ 255
        int i = expires & TVR_MASK;
        vec = tv1.vec + i;
    } else if (idx < 1 << (TVR_BITS + TVN_BITS)) { // 256 ~ 16191
        int i = (expires >> TVR_BITS) & TVN_MASK;
        vec = tv2.vec + i;
    } else if (idx < 1 << (TVR_BITS + 2 * TVN_BITS)) {
        int i = (expires >> (TVR_BITS + TVN_BITS)) & TVN_MASK;
        vec =  tv3.vec + i;
    } else if (idx < 1 << (TVR_BITS + 3 * TVN_BITS)) {
        int i = (expires >> (TVR_BITS + 2 * TVN_BITS)) & TVN_MASK;
        vec = tv4.vec + i;
    } else if ((signed long) idx < 0) {
        /* can happen if you add a timer with expires == jiffies,
         * or you set a timer to go off in the past
         */
        vec = tv1.vec + tv1.index;
    } else if (idx <= 0xffffffffUL) {
        int i = (expires >> (TVR_BITS + 3 * TVN_BITS)) & TVN_MASK;
        vec = tv5.vec + i;
    } else {
        /* Can only get here on architectures with 64-bit jiffies */
        INIT_LIST_HEAD(&timer->list);
        return;
    }
    /*
     * 添加到链表中
     */
    list_add(&timer->list, vec->prev);
}
```
`internal_add_timer()` 函数的主要工作是计算定时器到期时间所属的等级范围，然后把定时器添加到链表中。

#### 执行到期的定时器
```c
static inline void cascade_timers(struct timer_vec *tv)
{
    /* cascade all the timers from tv up one level */
    struct list_head *head, *curr, *next;

    head = tv->vec + tv->index;
    curr = head->next;
    /*
     * We are removing _all_ timers from the list, so we don't  have to
     * detach them individually, just clear the list afterwards.
     */
    while (curr != head) {
        struct timer_list *tmp;

        tmp = list_entry(curr, struct timer_list, list);
        next = curr->next;
        list_del(curr);
        internal_add_timer(tmp);
        curr = next;
    }
    INIT_LIST_HEAD(head);
    tv->index = (tv->index + 1) & TVN_MASK;
}

static inline void run_timer_list(void)
{
    spin_lock_irq(&timerlist_lock);
    while ((long)(jiffies - timer_jiffies) >= 0) {
        struct list_head *head, *curr;
        if (!tv1.index) { // 完成了一个轮回, 移动下一个单位的定时器
            int n = 1;
            do {
                cascade_timers(tvecs[n]);
            } while (tvecs[n]->index == 1 && ++n < NOOF_TVECS);
        }
repeat:
        head = tv1.vec + tv1.index;
        curr = head->next;
        if (curr != head) {
            struct timer_list *timer;
            void (*fn)(unsigned long);
            unsigned long data;

            timer = list_entry(curr, struct timer_list, list);
            fn = timer->function;
            data= timer->data;

            detach_timer(timer);
            timer->list.next = timer->list.prev = NULL;
            timer_enter(timer);
            spin_unlock_irq(&timerlist_lock);
            fn(data);
            spin_lock_irq(&timerlist_lock);
            timer_exit();
            goto repeat;
        }
        ++timer_jiffies;
        tv1.index = (tv1.index + 1) & TVR_MASK;
    }
    spin_unlock_irq(&timerlist_lock);
}
```
执行到期的定时器主要通过 `run_timer_list()` 函数完成，该函数首先比较当前时间与最后一次运行 `run_timer_list()` 函数时间的差值，然后循环这个差值的次数，并执行当前指针位置上的定时器。每循环一次对第一级数组指针进行加一操作，当第一级数组指针变为0（即所有定时器都执行完），那么就移动下一个等级的指针，并把该位置上的定时器重新计算插入到时间轮中，重新计算定时器通过 `cascade_timers()` 函数实现。
