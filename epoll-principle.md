## Epoll 实现原理

`epoll` 是Linux平台下的一种特有的多路复用IO实现方式，与传统的 `select` 相比，`epoll` 在性能上有很大的提升。本文主要讲解 `epoll` 的实现原理，而对于 `epoll` 的使用可以参考相关的书籍或文章。

### epoll 的创建

要使用 `epoll` 首先需要调用 `epoll_create()` 函数创建一个 `epoll` 的句柄，`epoll_create()` 函数定义如下：
```cpp
int epoll_create(int size);
```
参数 `size` 是由于历史原因遗留下来的，现在不起作用。当用户调用 `epoll_create()` 函数时，会进入到内核空间，并且调用 `sys_epoll_create()` 内核函数来创建 `epoll` 句柄，`sys_epoll_create()` 函数代码如下：
```cpp
asmlinkage long sys_epoll_create(int size)
{
    int error, fd = -1;
    struct eventpoll *ep;

    error = -EINVAL;
    if (size <= 0 || (error = ep_alloc(&ep)) < 0) {
        fd = error;
        goto error_return;
    }

    fd = anon_inode_getfd("[eventpoll]", &eventpoll_fops, ep);
    if (fd < 0)
        ep_free(ep);

error_return:
    return fd;
}
```
`sys_epoll_create()` 主要做两件事情：
1. 调用 `ep_alloc()` 函数创建并初始化一个 `eventpoll` 对象。
2. 调用 `anon_inode_getfd()` 函数把 `eventpoll` 对象映射到一个文件句柄，并返回这个文件句柄。

我们先来看看 `eventpoll` 这个对象，`eventpoll` 对象用于管理 `epoll` 监听的文件列表，其定义如下：
```cpp
struct eventpoll {
    ...
    wait_queue_head_t wq;
    ...
    struct list_head rdllist;
    struct rb_root rbr;
    ...
};
```
先来说明一下 `eventpoll` 对象各个成员的作用：
1. `wq`: 等待队列，当调用 `epoll_wait(fd)` 时会把进程添加到 `eventpoll` 对象的 `wq` 等待队列中。
2. `rdllist`: 保存已经就绪的文件列表。
3. `rbr`: 使用红黑树来管理所有被监听的文件。

下图展示了 `eventpoll` 对象与被监听的文件关系：

![epoll-eventpoll](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/epoll-eventpoll.jpg)

由于被监听的文件是通过 `epitem` 对象来管理的，所以上图中的节点都是以 `epitem` 对象的形式存在的。为什么要使用红黑树来管理被监听的文件呢？这是为了能够通过文件句柄快速查找到其对应的 `epitem` 对象。红黑树是一种平衡二叉树，如果对其不了解可以查阅相关的文档。

### 向 epoll 添加文件句柄

前面介绍了怎么创建 `epoll`，接下来介绍一下怎么向 `epoll` 添加要监听的文件。

通过调用 `epoll_ctl()` 函数可以向 `epoll` 添加要监听的文件，其原型如下：
```cpp
long epoll_ctl(int epfd, int op, int fd,struct epoll_event *event);
```
下面说明一下各个参数的作用：
1. `epfd`: 通过调用 `epoll_create()` 函数返回的文件句柄。
2. `op`: 要进行的操作，有3个选项：
   * `EPOLL_CTL_ADD`：表示要进行添加操作。
   * `EPOLL_CTL_DEL`：表示要进行删除操作。
   * `EPOLL_CTL_MOD`：表示要进行修改操作。
3. `fd`: 要监听的文件句柄。
4. `event`: 告诉内核需要监听什么事。其定义如下：
```cpp
struct epoll_event {
    __uint32_t events;  /* Epoll events */
    epoll_data_t data;  /* User data variable */
};
```
`events` 可以是以下几个宏的集合：
* `EPOLLIN` ：表示对应的文件句柄可以读（包括对端SOCKET正常关闭）；
* `EPOLLOUT`：表示对应的文件句柄可以写；
* `EPOLLPRI`：表示对应的文件句柄有紧急的数据可读（这里应该表示有带外数据到来）；
* `EPOLLERR`：表示对应的文件句柄发生错误；
* `EPOLLHUP`：表示对应的文件句柄被挂断；
* `EPOLLET`：将EPOLL设为边缘触发(Edge Triggered)模式，这是相对于水平触发(Level Triggered)来说的。
* `EPOLLONESHOT`：只监听一次事件，当监听完这次事件之后，如果还需要继续监听这个socket的话，需要再次把这个socket加入到EPOLL队列里。

`data` 用来保存用户自定义数据。

`epoll_ctl()` 函数会调用 `sys_epoll_ctl()` 内核函数，`sys_epoll_ctl()` 内核函数的实现如下：
```cpp

```
