## O(1)调度算法

Linux是一个支持多任务的操作系统，而多个任务之间的切换是通过 `调度器` 来完成，`调度器` 使用不同的调度算法会有不同的效果。

Linux2.4版本使用的调度算法的时间复杂度为O(n)，其主要原理是通过轮询所有可运行任务列表，然后挑选一个最合适的任务运行，所以其时间复杂度与可运行任务队列的长度成正比。

而Linux2.6开始替换成名为 `O(1)调度算法`，顾名思义，其时间复杂度为O(1)。虽然在后面的版本开始使用 `CFS调度算法（完全公平调度算法）`，但了解 `O(1)调度算法` 对学习Linux调度器还是有很大帮助的，所以本文主要介绍 `O(1)调度算法` 的原理与实现。

### O(1)调度算法原理

`O(1)调度算法` 通过优先级来对任务进行分组，可分为140个优先级（0 ~ 139），每个优先级的任务由一个队列来维护。`prio_array` 结构就是用来维护这些任务队列，如下代码：
```cpp
#define MAX_PRIO  (100 + 40)

struct prio_array {
    int nr_active;
    unsigned long bitmap[BITMAP_SIZE];
    struct list_head queue[MAX_PRIO];
};
```
下面介绍 `prio_array` 结构各个字段的作用：
1. `nr_active`: 所有优先级队列中的总任务数。
2. `bitmap`: 位图，每个位对应一个优先级队列，用于记录哪个优先级队列不为空。
3. `queue`: 优先级队列数组，每个元素维护一个优先级队列，比如索引为0的元素维护着优先级为0的任务队列。

下图更直观地展示了 `prio_array` 结构各个字段的关系：

![prio_array](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/process-schedule-o1.jpg)

从上图可以看出，`bitmap` 的第2位和第6位为1（红色代表为1，白色代表为0），表示优先级为2和6的任务队列不为空，也就是说 `queue` 数组的第2个元素和第6个元素的队列不为空。
