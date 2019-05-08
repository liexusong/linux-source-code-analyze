## 什么是多路复用IO
`多路复用IO (IO multiplexing)` 是指内核一旦发现进程指定的一个或者多个IO条件准备读取，它就通知该进程。在Linux系统中，常用的 `多路复用IO` 手段有 `select`、`poll` 和 `epoll`。

`多路复用IO` 主要用于处理网络请求，例如可以把多个请求句柄添加到 `select` 中进行监听，当有请求可进行IO的时候就会告知进程，并且把就绪的请求句柄保存下来，进程只需要对这些就绪的请求进行IO操作即可。下面通过一幅图来展示 `select` 的使用方式（图片来源于网络）：

![select-model](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/select-model.png)

## 多路复用IO实现原理
为了更简明的解释 `多路复用IO` 的原理，这里使用 `select` 系统调用作为分析对象。因为 `select` 的实现比较简单，而现在流行的 `epoll` 由于处于性能考虑，实现则比较复杂，不便于理解 `多路复用IO` 的原理，当然当理解了 `select` 的实现原理后，对 `epoll` 的实现就能应刃而解了。

### select系统调用的使用
要使用 `select` 来监听socket是否可以进行IO，首先需要把其添加到一个类型为 `fd_set` 的结构中，然后通过调用 `select()` 系统调用来进行监听，下面代码介绍了怎么使用 `select` 来对socket进行监听的：
```cpp
int socket_can_read(int fd)
{
    int retval;
    fd_set set;
    struct timeval tv;

    FD_ZERO(&set);
    FD_SET(fd, &set);

    tv.tv_sec = tv.tv_usec = 0;

    retval = select(fd+1, &set, NULL, NULL, &tv);
    if (retval < 0) {
        return -1;
    }

    return FD_ISSET(fd, &set) ? 1 : 0;
}
```
通过上面的函数，可以监听一个socket句柄是否可读。

### select系统调用的实现
