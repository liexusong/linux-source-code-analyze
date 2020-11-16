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

在分析 `ll_rw_block()` 函数前，我们先来介绍一下 `buffer_head` 这个结构，因为要理解 `ll_rw_block()` 函数必须先了解  `buffer_head` 结构。

 `struct buffer_head` 结构代表一个要进行读或者写的数据块，其定义如下：

```c
struct buffer_head {
    struct buffer_head *b_next;     /* 用于快速查找数据块缓存 */
    unsigned long b_blocknr;        /* 数据块号 */
    unsigned short b_size;          /* 数据块大小 */
    unsigned short b_list;          /* List that this buffer appears */
    kdev_t b_dev;                   /* 数据块所属设备 */

    atomic_t b_count;               /* 引用计数器 */
    kdev_t b_rdev;                  /* 数据块所属真正设备 */
    ...
};
```

为了让读者更加清晰，上面忽略了 `buffer_head` 结构的某些字段，上面比较重要的是 `b_blocknr` 字段和 `b_size` 字段， `b_blocknr` 字段指定了要读写的数据块号，而 `b_size` 字段指定了数据块的大小。还有就是 `b_rdev` 字段，其指定了数据块所属的设备。

有了 `buffer_head` 结构，就可以对数据块进行读写操作。接下来，我们看看 `ll_rw_block()` 函数的实现：

```c
void ll_rw_block(int rw, int nr, struct buffer_head *bhs[])
{
    ...
    for (i = 0; i < nr; i++) {
        struct buffer_head *bh = bhs[i];

        /* 上锁 */
        if (test_and_set_bit(BH_Lock, &bh->b_state))
            continue;

        /* 增加buffer_head的计数器 */
        atomic_inc(&bh->b_count);
        bh->b_end_io = end_buffer_io_sync;
        ...
        submit_bh(rw, bh);
    }
    return;
    ...
}
```

下面介绍一下 `ll_rw_block()` 函数各个参数的作用：

*   `rw`：要进行的读或者写操作，一般可选的值为 `READ`、`WRITE` 或者 `READA` 等。
*   `nr`：`bhs` 数组的大小。
*   `bhs`：要进行读写操作的数据块数组。

`ll_rw_block()` 函数的实现比较简单，遍历 `bhs` 数组，并且对所有的 `buffer_head` 进行上锁和增加其计数器，然后调用 `submit_bh()` 函数把其提交到 `IO调度层` 进行I/O操作。

我们接着看看 `submit_bh()` 函数的实现：

```c
void submit_bh(int rw, struct buffer_head *bh)
{
    int count = bh->b_size >> 9; // 一个数据块需要的扇区数
    ...
    set_bit(BH_Req, &bh->b_state);

    bh->b_rdev = bh->b_dev;
    bh->b_rsector = bh->b_blocknr * count;

    generic_make_request(rw, bh);

    switch (rw) {
    case WRITE:
        kstat.pgpgout += count;
        break;
    default:
        kstat.pgpgin += count;
        break;
    }
}
```

