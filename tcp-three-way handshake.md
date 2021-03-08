# TCP源码分析 - 三次握手

本文主要分析 TCP 协议的实现，但由于 TCP 协议比较复杂，所以分几篇文章进行分析，这篇主要介绍 TCP 协议建立连接时的三次握手过程。

TCP 协议应该是 TCP/IP 协议栈中最为复杂的一个协议（没有之一），TCP 协议的复杂性来源于其面向连接和保证可靠传输。

如下图所示，TCP 协议位于 TCP/IP 协议栈的第四层，也就是传输层，其建立在网络层的 IP 协议。

![](F:\linux-source-code-analyze\images\tcp\tcp-ip-layer.png)

但由于 IP 协议是一个无连接不可靠的协议，所以 TCP 协议要实现面向连接的可靠传输，就必须为每个 CS（Client - Server） 连接维护一个连接状态。由此可知，TCP 协议的连接只是维护了一个连接状态，而非真正的连接。

由于本文主要介绍 Linux 内核是怎么实现 TCP 协议的，如果对 TCP 协议的原理不是很清楚的话，可以参考著名的《TCP/IP协议详解》。

## 三次握手过程

我们知道，TCP 协议是建立在无连接的 IP 协议之上，而为了实现面向连接，TCP 协议使用了一种协商的方式来建立连接状态，称为：`三次握手`。`三次握手` 的过程如下图：

![](F:\linux-source-code-analyze\images\tcp\three-way-handshake.png)

建立连接过程如下：

*   客户端需要发送一个 `SYN包` 到服务端（包含了客户端初始化序列号），并且将连接状态设置为 `SYN_SENT`。
*   服务端接收到客户端的 `SYN包` 后，需要回复一个 `SYN+ACK包` 给客户端（包含了服务端初始化序列号），并且设置连接状态为 `SYN_RCVD`。
*   客户端接收到服务端的 `SYN+ACK包` 后，设置连接状态为 `ESTABLISHED`（表示连接已经建立），并且回复一个 `ACK包` 给服务端。
*   服务端接收到客户端的 `ACK包` 后，将连接状态设置为 `ESTABLISHED`（表示连接已经建立）。

以上过程完成后，一个 TCP 连接就此建立完成。

## TCP 头部

要分析 TCP 协议就免不了要了解 TCP 协议头部，我们通过下面的图片来介绍 TCP 头部的格式：

![](F:\linux-source-code-analyze\images\tcp\tcp-header.png)

下面介绍一下 TCP 头部各个字段的作用：

*   `源端口号`：用于指定本地程序绑定的端口。
*   `目的端口号`：用于指定远端程序绑定的端口。
*   `序列号`：用于本地发送数据时所使用的序列号。
*   `确认号`：用于本地确认接收到远端发送过来的数据序列号。
*   `首部长度`：指示 TCP 头部的长度。
*   `标志位`：用于指示 TCP 数据包的类型。
*   `窗口大小`：用于流量控制，表示远端能够接收数据的能力。
*   `校验和`：用于校验数据包是否在传输时损坏了。
*   `紧急指针`：一般比较少用，用于指定紧急数据的偏移量（`URG` 标志位为1时有效）。
*   `可选项`：TCP的选项部分。

我们来看看 Linux 内核怎么定义 TCP 头部的结构，如下：

```c
struct tcphdr {
    __u16   source;   // 源端口
    __u16   dest;     // 目的端口
    __u32   seq;      // 序列号
    __u32   ack_seq;  // 确认号
    __u16   doff:4,   // 头部长度
            res1:4,   // 保留
            res2:2,   // 保留
            urg:1,    // 是否包含紧急数据
            ack:1,    // 是否ACK包
            psh:1,    // 是否Push包
            rst:1,    // 是否Reset包
            syn:1,    // 是否SYN包
            fin:1;    // 是否FIN包
    __u16   window;   // 滑动窗口
    __u16   check;    // 校验和
    __u16   urg_ptr;  // 紧急指针
};
```

从上面的定义可知，结构 `tcphdr` 的各个字段与 TCP 头部的各个字段一一对应。

## 客户端连接过程

一个 TCP 连接是由客户端发起的，当客户端程序调用 `connect()` 系统调用时，就会与服务端程序建立一个 TCP 连接。`connect()` 系统调用的原型如下：

```c
int connect(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
```

下面是 `connect()` 系统调用各个参数的作用：

*   `sockfd`：由 `socket()` 系统调用创建的文件句柄。
*   `addr`：指定要连接的远端 IP 地址和端口。
*   `addrlen`：指定参数 `addr` 的长度。

当客户端调用 `connect()` 函数时，会触发内核调用 `sys_connect()` 内核函数，`sys_connect()` 函数实现如下：

```c
int sys_connect(int fd, struct sockaddr *uservaddr, int addrlen)
{
    struct socket *sock;
    char address[MAX_SOCK_ADDR];
    int err;
    ...
    // 获取文件句柄对应的socket对象
    sock = sockfd_lookup(fd, &err);
    ...
    // 从用户空间复制要连接的远端IP地址和端口信息
    err = move_addr_to_kernel(uservaddr, addrlen, address);
    ...
    // 调用 inet_stream_connect() 函数完成连接操作
    err = sock->ops->connect(sock, (struct sockaddr *)address, addrlen,
                             sock->file->f_flags);
    ...
    return err;
}
```

`sys_connect()` 内核函数主要完成 3 个步骤：

*   调用 `sockfd_lookup()` 函数获取 `fd` 文件句柄对应的 socket 对象。
*   调用 `move_addr_to_kernel()` 函数从用户空间复制要连接的远端 IP 地址和端口信息。
*   调用 `inet_stream_connect()` 函数完成连接操作。

我们继续分析 `inet_stream_connect()` 函数的实现：

```c
int inet_stream_connect(struct socket *sock, struct sockaddr * uaddr,
                        int addr_len, int flags)
{
    struct sock *sk = sock->sk;
    int err;
    ...
    if (sock->state == SS_CONNECTING) {
        ...
    } else {
        // 尝试自动绑定一个本地端口
        if (inet_autobind(sk) != 0) 
            return(-EAGAIN);
        ...
        // 调用 tcp_v4_connect() 进行连接操作
        err = sk->prot->connect(sk, uaddr, addr_len);
        if (err < 0)
            return(err);
        sock->state = SS_CONNECTING;
    }
    ...
    // 如果 socket 设置了非阻塞, 并且连接还没建立, 那么返回 EINPROGRESS 错误
    if (sk->state != TCP_ESTABLISHED && (flags & O_NONBLOCK))
        return (-EINPROGRESS);

    // 等待连接过程完成
    if (sk->state == TCP_SYN_SENT || sk->state == TCP_SYN_RECV) {
        inet_wait_for_connect(sk);
        if (signal_pending(current))
            return -ERESTARTSYS;
    }
    sock->state = SS_CONNECTED; // 设置socket的状态为connected
    ...
    return(0);
}
```

`inet_stream_connect()` 函数的主要操作有以下几个步骤：

*   调用 `inet_autobind()` 函数尝试自动绑定一个本地端口。
*   调用 `tcp_v4_connect()` 函数进行 TCP 协议的连接操作。
*   如果 socket 设置了非阻塞，并且连接还没建立完成，那么返回 EINPROGRESS 错误。
*   调用 `inet_wait_for_connect()` 函数等待连接服务端操作完成。
*   设置 socket 的状态为 `SS_CONNECTED`，表示连接已经建立完成。

在上面的步骤中，最重要的是调用 `tcp_v4_connect()` 函数进行连接操作，我们来分析一下 `tcp_v4_connect()` 函数的实现：

```c
int tcp_v4_connect(struct sock *sk, struct sockaddr *uaddr, int addr_len)
{
    struct tcp_opt *tp = &(sk->tp_pinfo.af_tcp);
    struct sockaddr_in *usin = (struct sockaddr_in *)uaddr;
    struct sk_buff *buff;
    struct rtable *rt;
    u32 daddr, nexthop;
    int tmp;
    ...
    nexthop = daddr = usin->sin_addr.s_addr;
    ...
    // 获取数据输出路由信息对象
    tmp = ip_route_connect(&rt, nexthop, sk->saddr,
                           RT_TOS(sk->ip_tos)|RTO_CONN|sk->localroute,
                           sk->bound_dev_if);
    ...
    dst_release(xchg(&sk->dst_cache, rt)); // 设置sk的路由信息

    // 申请一个skb数据包对象
    buff = sock_wmalloc(sk, (MAX_HEADER + sk->prot->max_header), 0, GFP_KERNEL);
    ...
    sk->dport = usin->sin_port; // 设置目的端口
    sk->daddr = rt->rt_dst;     // 设置目的IP地址
    ...
    if (!sk->saddr)
        sk->saddr = rt->rt_src;
    sk->rcv_saddr = sk->saddr;
    ...
    // 初始化TCP序列号
    tp->write_seq = secure_tcp_sequence_number(sk->saddr, sk->daddr, sk->sport,
                                               usin->sin_port);
    ...
    /* Reset mss clamp */
    tp->mss_clamp = ~0;
    ...
    // 调用 tcp_connect() 函数继续进行连接操作
    tcp_connect(sk, buff, rt->u.dst.pmtu);
    return 0;
}
```

