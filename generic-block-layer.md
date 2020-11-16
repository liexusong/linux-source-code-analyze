## 概述

由于不同块设备（如磁盘，机械硬盘等）有着不同的设备驱动程序，为了让文件系统有统一的读写块设备接口，Linux实现了一个 `通用块层`。如下图中的红色部分：

![linux-filesystem](F:\work\markdown\linux-filesystem.jpg)

`通用块层` 的引入为了提供一个统一的接口让文件系统实现者使用，而不用关心不同设备驱动程序的差异，这样实现出来的文件系统就能用于任何的块设备。

`通用块层` 将对不同块设备的操作转换成对逻辑数据块的操作，也就是将不同的块设备都抽象成是一个数据块数组，而文件系统就是对这些数据块进行管理。如下图：

>   注意：不同的文件系统可能对逻辑数据块定义的大小不一样，比如 ext2文件系统 的逻辑数据块大小为 4KB。

![device-block](F:\work\markdown\device-block.jpg)

通过对设备进行抽象后，不管是磁盘还是机械硬盘，对于文件系统都可以使用相同的接口对逻辑数据块进行读写操作。

## 通用块读写接口

`通用块层` 提供了 `ll_rw_block()` 函数对逻辑块进行读写操作，`ll_rw_block()` 函数的原型如下：

```c
void ll_rw_block(int rw, int nr, struct buffer_head *bhs[]);
```

在分析 `ll_rw_block()` 函数前，我们先来介绍一下 `struct buffer_head` 这个结构，因为要理解 `ll_rw_block()` 函数必须先了解  `struct buffer_head` 结构。

 `struct buffer_head` 结构代表一个要进行读或者写的数据块，其定义如下：

```c
struct buffer_head {
    struct buffer_head *b_next;     /* 用于快速查找 */
    unsigned long b_blocknr;        /* 数据块号 */
    unsigned short b_size;          /* 数据块大小 */
    unsigned short b_list;          /* List that this buffer appears */
    kdev_t b_dev;                   /* device (B_FREE = free) */

    atomic_t b_count;               /* users using this block */
    kdev_t b_rdev;                  /* Real device */
    ...
};
```




