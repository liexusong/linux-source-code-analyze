## Epoll 实现原理

`epoll` 是Linux平台下的一种特有的多路复用IO实现方式，与传统的 `select` 相比，`epoll` 在性能上有很大的提升。本文主要讲解 `epoll` 的实现原理，而对于 `epoll` 的使用可以参考相关的书籍或文章。

### epoll创建

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
1. 调用 `ep_alloc()` 函数创建一个 `eventpoll` 对象。
2. 调用 `anon_inode_getfd()` 函数把 `eventpoll` 对象映射到一个文件句柄并且返回这个句柄。

我们先来看看 `eventpoll` 这个对象，`eventpoll` 对象用于管理 `epoll` 监听的文件列表，其定义如下：
```cpp
struct eventpoll {
    ...
    wait_queue_head_t wq;
    ...
    struct list_head rdllist;
    struct rb_root rbr;
    struct epitem *ovflist;
};
```
先来说明一下 `eventpoll` 对象各个成员的作用：
1. `wq`: 等待队列，当调用 `epoll_wait(fd)` 时会把进程添加到 `eventpoll` 对象的 `wq` 等待队列中。
2. `rdllist`: 保存已经就绪的文件列表。
3. `rbr`: 使用红黑树来管理所有被监听的文件。
4. `ovflist`: 使用链表来管理所有被监听的文件。

下图展示了 `eventpoll` 对象与被监听的文件关系：
![epoll-eventpoll]()
