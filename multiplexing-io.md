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
接下来我们分析一下 `select` 系统调用的实现，用户程序通过调用 `select` 系统调用后会进入到内核态并且调用 `sys_select()` 函数，`sys_select()` 函数的实现如下：
```cpp
asmlinkage long
sys_select(int n, fd_set *inp, fd_set *outp, fd_set *exp, struct timeval *tvp)
{
    fd_set_bits fds;
    char *bits;
    long timeout;
    int ret, size;

    timeout = MAX_SCHEDULE_TIMEOUT;
    if (tvp) {
        time_t sec, usec;
        ...
        if ((unsigned long) sec < MAX_SELECT_SECONDS) {
            timeout = ROUND_UP(usec, 1000000/HZ);
            timeout += sec * (unsigned long) HZ;
        }
    }

    if (n > current->files->max_fdset)
        n = current->files->max_fdset;

    ret = -ENOMEM;
    size = FDS_BYTES(n);
    bits = select_bits_alloc(size);

    fds.in = (unsigned long *)bits;
    fds.out = (unsigned long *)(bits + size);
    fds.ex = (unsigned long *)(bits + 2*size);
    fds.res_in = (unsigned long *)(bits + 3*size);
    fds.res_out = (unsigned long *)(bits + 4*size);
    fds.res_ex = (unsigned long *)(bits + 5*size);

    if ((ret = get_fd_set(n, inp, fds.in)) ||
        (ret = get_fd_set(n, outp, fds.out)) ||
        (ret = get_fd_set(n, exp, fds.ex)))
        goto out;
    zero_fd_set(n, fds.res_in);
    zero_fd_set(n, fds.res_out);
    zero_fd_set(n, fds.res_ex);

    ret = do_select(n, &fds, &timeout);
    ...
    set_fd_set(n, inp, fds.res_in);
    set_fd_set(n, outp, fds.res_out);
    set_fd_set(n, exp, fds.res_ex);

out:
    select_bits_free(bits, size);
out_nofds:
    return ret;
}
```
`sys_select()` 函数主要把用户态的参数复制到内核态，然后再通过调用 `do_select()` 函数进行监听操作， `do_select()` 函数实现如下（由于实现有点复杂，所以我们分段来分析）：
```cpp
int do_select(int n, fd_set_bits *fds, long *timeout)
{
    poll_table table, *wait;
    int retval, i, off;
    long __timeout = *timeout;
    ...
    poll_initwait(&table);
    wait = &table;
    if (!__timeout)
        wait = NULL;
    retval = 0;
```
上面这段代码主要通过调用 `poll_initwait()` 函数来初始化类型为 `poll_table` 结构的变量 `table`。要理解 `poll_table` 结构的作用，我们先来看看下面的知识点：

> 因为每个socket都有个等待队列，当某个进程需要对socket进行读写的时候，如果发现此socket并不能读写，
> 那么就可以添加到此socket的等待队列中进行休眠，当此socket可以读写时再唤醒队列中的进程。  

