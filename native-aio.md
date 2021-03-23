# Linux 原生异步 IO 原理与使用（Native AIO）

什么是异步 IO？

>   异步 IO：当应用程序发起一个 IO 操作后，调用者不能立刻得到结果，而是在内核完成 IO 操作后，通过信号或回调来通知调用者。

异步 IO 与同步 IO 的区别如 图1 所示：

![](F:\books\markdown\aio\aio.png)

从上图可知，同步 IO 必须等待内核把 IO 操作处理完成后才返回。而异步 IO 不必等待 IO 操作完成，而是向内核发起一个 IO 操作就立刻返回，当内核完成 IO 操作后，会通过信号的方式通知应用程序。

## Linux 原生 AIO 原理

`Linux Native AIO` 是 Linux 支持的原生 AIO，为什么要加原生这个词呢？因为Linux存在很多第三方的异步 IO 库，如 `libeio` 和 `glibc AIO`。所以为了加以区别，Linux 的内核提供的异步 IO 就称为原生异步 IO。 

很多第三方的异步 IO 库都不是真正的异步 IO，而是使用多线程来模拟异步 IO，如 `libeio` 就是使用多线程来模拟异步 IO 的。

本文主要介绍 Linux 原生 AIO 的原理和使用，所以不会对其他第三方的异步 IO 库进行分析，下面我们先来介绍 Linux 原生 AIO 的原理。

如 图2 所示：

![](F:\books\markdown\aio\linux-native-aio.png)

Linux 原生 AIO 处理流程：

*   当应用程序调用 `io_submit` 系统调用发起一个异步 IO 操作后，会向内核的 IO 任务队列中添加一个 IO 任务，并且返回成功。
*   内核会在后台处理 IO 任务队列中的 IO 任务，然后把处理结果存储在 IO 任务中。
*   应用程序可以调用 `io_getevents` 系统调用来获取异步 IO 的处理结果，如果 IO 操作还没完成，那么返回失败信息，否则会返回 IO 处理结果。

从上面的流程可以看出，Linux 的异步 IO 操作主要由两个步骤组成：

*   1) 调用 `io_submit` 函数发起一个异步 IO 操作。
*   2) 调用 `io_getevents` 函数获取异步 IO 的结果。

下面我们主要分析，Linux 内核是怎么实现异步 IO 的。

## Linux 原生 AIO 使用

在介绍 Linux 原生 AIO 的实现之前，先通过一个简单的例子来介绍其使用过程：

```c
#define _GNU_SOURCE

#include <stdlib.h>
#include <string.h>
#include <libaio.h>
#include <errno.h>
#include <stdio.h>
#include <unistd.h>
#include <fcntl.h>

#define FILEPATH "./aio.txt"

int main()
{
    io_context_t context;
    struct iocb io[1], *p[1] = {&io[0]};
    struct io_event e[1];
    unsigned nr_events = 10;
    struct timespec timeout;
    char *wbuf;
    int wbuflen = 1024;
    int ret, num = 0, i;

    posix_memalign((void **)&wbuf, 512, wbuflen);

    memset(wbuf, '@', wbuflen);
    memset(&context, 0, sizeof(io_context_t));

    timeout.tv_sec = 0;
    timeout.tv_nsec = 10000000;

    int fd = open(FILEPATH, O_CREAT|O_RDWR|O_DIRECT, 0644); // 1. 打开要进行异步IO的文件
    if (fd < 0) {
        printf("open error: %d\n", errno);
        return 0;
    }

    if (0 != io_setup(nr_events, &context)) {               // 2. 创建一个异步IO上下文
        printf("io_setup error: %d\n", errno);
        return 0;
    }

    io_prep_pwrite(&io[0], fd, wbuf, wbuflen, 0);           // 3. 创建一个异步IO任务

    if ((ret = io_submit(context, 1, p)) != 1) {            // 4. 提交异步IO任务
        printf("io_submit error: %d\n", ret);
        io_destroy(context);
        return -1;
    }

    while (1) {
        ret = io_getevents(context, 1, 1, e, &timeout);     // 5. 获取异步IO的结果
        if (ret < 0) {
            printf("io_getevents error: %d\n", ret);
            break;
        }

        if (ret > 0) {
            printf("result, res2: %d, res: %d\n", e[0].res2, e[0].res);
            break;
        }
    }

    return 0;
}
```

上面通过一个简单的例子来展示了 Linux 原生 AIO 的使用过程，主要有以下步骤：

*   通过调用 `open` 系统调用打开要进行异步 IO 的文件，要注意的是 AIO 操作必须设置 `O_DIRECT` 直接 IO 标志位。
*   调用 `io_setup` 系统调用创建一个异步 IO 上下文。
*   调用 `io_prep_pwrite` 或者 `io_prep_pread` 函数创建一个异步写或者异步读任务。
*   调用 `io_submit` 系统调用把异步 IO 任务提交到内核。
*   调用 `io_getevents` 系统调用获取异步 IO 的结果。

在上面的例子中，我们获取异步 IO 操作的结果是在一个无限循环中进行的，其实 Linux 还支持一种基于 `eventfd` 事件通知的机制，可以通过 `eventfd` 和 `epoll` 结合来实现事件驱动的方式来获取异步 IO 操作的结果，有兴趣可以查阅相关的内容。

## 总结

本文主要介绍了 Linux 原生 AIO 的原理和使用，Linux 原生 AIO 的使用比较简单，但其内部实现比较复杂，在下篇文章中将会介绍 Linux 原生 AIO 的实现。



