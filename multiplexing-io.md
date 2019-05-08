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

而 `poll_table` 结构就是为了把进程添加到socket的等待队列中而创造的，我们先跳过这部分，后面分析到socket相关的知识点再来说明。

我们接着分析 `do_select()` 函数的实现：
```cpp
    for (;;) {
        set_current_state(TASK_INTERRUPTIBLE);
        for (i = 0 ; i < n; i++) {
            ...
            file = fget(i);
            mask = POLLNVAL;
            if (file) {
                mask = DEFAULT_POLLMASK;
                if (file->f_op && file->f_op->poll)
                    mask = file->f_op->poll(file, wait);
                fput(file);
            }
```
这段代码首先通过调用文件句柄的 `poll()` 接口来检查文件是否能够进行IO操作，对于socket来说，这个 `poll()` 接口就是 `sock_poll()`，所以我们来看看 `sock_poll()` 函数的实现：
```cpp
static unsigned int sock_poll(struct file *file, poll_table * wait)
{
    struct socket *sock;

    sock = socki_lookup(file->f_dentry->d_inode);
    return sock->ops->poll(file, sock, wait);
}
```
`sock_poll()` 函数的实现很简单，首先通过 `socki_lookup()` 函数来把文件句柄转换成socket结构，接着调用socket结构的 `poll()` 接口，而对应 `TCP` 类型的socket，这个接口对应的是 `tcp_poll()` 函数，实现如下：
```cpp
unsigned int tcp_poll(struct file * file, struct socket *sock, poll_table *wait)
{
    unsigned int mask;
    struct sock *sk = sock->sk;
    struct tcp_opt *tp = &(sk->tp_pinfo.af_tcp);

    poll_wait(file, sk->sleep, wait); // 把文件添加到sk->sleep队列中进行等待
    ...
    return mask;
}
```
`tcp_poll()` 函数通过调用 `poll_wait()` 函数把进程添加到socket的等待队列中。然后检测socket是否可读写，并通过mask返回可读写的状态。所以在 `do_select()` 函数中的 `mask = file->f_op->poll(file, wait);` 这行代码其实调用的是 `tcp_poll()` 函数。

接着分析 `do_select()` 函数：
```cpp
            if ((mask & POLLIN_SET) && ISSET(bit, __IN(fds,off))) {
                SET(bit, __RES_IN(fds,off));
                retval++;
                wait = NULL;
            }
            if ((mask & POLLOUT_SET) && ISSET(bit, __OUT(fds,off))) {
                SET(bit, __RES_OUT(fds,off));
                retval++;
                wait = NULL;
            }
            if ((mask & POLLEX_SET) && ISSET(bit, __EX(fds,off))) {
                SET(bit, __RES_EX(fds,off));
                retval++;
                wait = NULL;
            }
```
因为 `mask` 变量保存了socket的可读写状态，所以上面这段代码主要通过判断socket的可读写状态来把socket放置到合适的返回集合中。如果socket可读，那么就把socket放置到可读集合中，如果socket可写，那么就放置到可写集合中。

```cpp
        wait = NULL;
        if (retval || !__timeout || signal_pending(current))
            break;
        if(table.error) {
            retval = table.error;
            break;
        }
        __timeout = schedule_timeout(__timeout);
    }
    current->state = TASK_RUNNING;

    poll_freewait(&table);

    *timeout = __timeout;
    return retval;
}
```
最后这段代码的作用是，如果监听的socket集合中有可读写的socket，那么就直接返回（retval不为0时）。另外，如果调用 `select()` 时超时了，或者进程接收到信号，也需要返回。

否则，通过调用 `schedule_timeout()` 来进行一次进程调度。因为前面把进程的运行状态设置成 `TASK_INTERRUPTIBLE`，所以进行进程调度时就会把当前进程从运行队列中移除，进程进入休眠状态。那么什么时候进程才会变回运行状态呢？

前面我们说过，每个socket都有个等待队列，所以当socket可读写时便会把队列中的进程唤醒。这里分析一下当socket变成可读时，怎么唤醒等待队列中的进程的。

网卡接收到数据时，会进行一系列的接收数据操作，对于TCP协议来说，接收数据的调用链是： `tcp_v4_rcv() -> tcp_data() -> tcp_data_queue() -> sock_def_readable()`，我们来看看 `sock_def_readable()` 函数的实现：
```cpp
void sock_def_readable(struct sock *sk, int len)
{
    read_lock(&sk->callback_lock);
    if (sk->sleep && waitqueue_active(sk->sleep))
        wake_up_interruptible(sk->sleep);
    sk_wake_async(sk,1,POLL_IN);
    read_unlock(&sk->callback_lock);
}
```
可以看出 `sock_def_readable()` 函数最终会调用 `wake_up_interruptible()` 函数来把等待队列中的进程唤醒，这时调用 `select()` 的进程从休眠状态变回运行状态。
