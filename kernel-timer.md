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

每级数组上面都有一个指针，指向当前要执行的定时器。每当时间走一秒，Linux首先会移动第一级的指针，然后执行当前位置上的定时器。当指针变为0时，会移动下一级的指针，并把该位置上的定时器重新计算一次并且插入到时间轮中，其他级如此类推。

