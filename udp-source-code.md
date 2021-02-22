# UDP协议源码分析

`UDP协议` 是 **User Datagram Protocol** 的简称， 中文名是用户数据报协议，是 OSI（Open System Interconnection，开放式系统互联） 参考模型中一种无连接的传输层协议，提供面向事务的简单不可靠信息传送服务，位于 `TCP/IP协议` 模型的 `传输层`，如下图：

![tcp-ip-layer](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/udp/tcp-ip-layer.png)

也就是说 `UDP协议` 是建立中 `IP协议`（网络层）之上的，`IP协议` 用于区分网络上不同的主机（[IP协议源码分析](https://github.com/liexusong/linux-source-code-analyze/blob/master/ip-source-code.md)），而 `UDP协议` 用于区分同一台主机上不同的进程发送（接收）的网络数据，如下图所示：

![udp-schedule](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/udp/udp-schedule.png)

从上图可以看出，`UDP协议` 通过 `端口号` 来区分不同进程的数据包。

## UDP协议头

下面我们来看看 `UDP协议` 的协议头部，如下图所示：

![udp-header](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/udp/udp-header.png)

从上图可知，`UDP头部` 由四个字段组成：`源端口`、`目标端口`、`数据包长度` 和 `校验和`。

`源端口` 用于指示本机的进程，而 `目标端口` 用于指示远端的进程。`数据包长度` 表示这个 UDP 数据包总长度（包括UDP头部和 数据长度），而 `校验和` 用于校验数据包在传输的过程中是否损坏了。

下面我们看看 `UDP头部` 在内核中的表示方式，如下代码：

```c
struct udphdr {
    __u16   source;  // 源端口
    __u16   dest;    // 目标端口
    __u16   len;     // 数据包长度
    __u16   check;   // 校验和
};
```

可以看出，`udphdr` 结构的字段与 `UDP头部` 结构图中的字段一一对应。最后，我们来看看 `UDP头部` 在数据包的具体位置，如下图：

![udp-header](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/udp/udp-header-2.png)

下面我们主要通过 UDP 数据包的发送和接收两个过程来分析 UDP 在内核中的实现原理。

## UDP数据包发送

数据的发送是由应用层调用 `send()` 或者 `write()` 系统调用，将数据传递到传输层协议处理，如下图：

![udp-sendmsg](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/udp/udp-sendmsg.png)

从上图可以看出，`用户态` 的应用程序调用 `send()` 系统调用时会触发调用 `内核态` 的 `sys_send()` 内核函数，而 `sys_send()` 最终会调用 `inet_sendmsg()` 函数发送数据。

`inet_sendmsg()` 函数会根据用户使用的传输层协议选择不同的数据发送接口，比如 UDP 协议就会使用 `udp_sendmsg()` 函数发送数据。

我们来分析一下 UDP 协议的发送接口 `udp_sendmsg()` 函数的实现，代码如下（由于 `udp_sendmsg()` 函数的实现比较复杂，所以我们分段分析）：

```c
int udp_sendmsg(struct sock *sk, struct msghdr *msg, int len)
{
    int ulen = len + sizeof(struct udphdr);
    struct ipcm_cookie ipc;
    struct udpfakehdr ufh;
    struct rtable *rt = NULL;
    int free = 0;
    int connected = 0;
    u32 daddr;
    u8  tos;
    int err;
```

`udp_sendmsg()` 函数的参数含义如下：

*   `sk`：Socket 对象。
*   `msg`：要发送的数据实体，其类型为 `msghdr` 结构。
*   `len`：要发送的数据长度。

上面的代码主要定义了一些局部变量，如：

*   `ulen` 变量就是要发送的数据总长度（UDP头部长度和数据长度之和）。

*   `rt` 变量表示数据传输的路由信息，其类型为 `rtable` 结构。

*   `ufh` 变量是调用 IP 层 `ip_build_xmit()` 函数时的上下文，主要用于构建 `UDP协议头部` ，其类型为 `udpfakehdr` 结构。

我们接口分析 `udp_sendmsg()` 函数：

```c
    // 是否提供了接收数据的目标IP地址和端口
    if (msg->msg_name) {
        // 接收数据的目标IP地址和端口
        struct sockaddr_in *usin = (struct sockaddr_in*)msg->msg_name;

        if (msg->msg_namelen < sizeof(*usin))
            return -EINVAL;

        if (usin->sin_family != AF_INET) {
            if (usin->sin_family != AF_UNSPEC)
                return -EINVAL;
        }

        // 设置接收数据的目标IP地址和端口
        ufh.daddr = usin->sin_addr.s_addr;
        ufh.uh.dest = usin->sin_port;

        if (ufh.uh.dest == 0)
            return -EINVAL;
    } else {
        if (sk->state != TCP_ESTABLISHED)
            return -ENOTCONN;

        ufh.daddr = sk->daddr;   // 使用绑定Socket的IP地址
        ufh.uh.dest = sk->dport; // 使用绑定Socket的端口
        connected = 1;
    }
```

上面的代码主要完成以下几个工作：

*   如果用户发送数据时提供了目标 IP 地址和端口，就把用户提供的目标 IP 地址和端口复制到 `ufh` 变量中。
*   否则就把绑定到 Socket 对象的目标 IP 地址和端口复制到 `ufh` 变量中，并且设置 `connected` 变量为 1。

我们继续分析 `udp_sendmsg()` 函数，代码如下：

```c
    if (connected)
        rt = (struct rtable*)sk_dst_check(sk, 0); // 获取路由信息对象缓存

    if (rt == NULL) { // 如果路由信息对象还没被缓存
        // 调用 ip_route_output() 函数获取路由信息对象
        err = ip_route_output(&rt, daddr, ufh.saddr, tos, ipc.oif);
        if (err)
            goto out;

        err = -EACCES;
        if (rt->rt_flags & RTCF_BROADCAST && !sk->broadcast)
            goto out;

        if (connected)
            sk_dst_set(sk, dst_clone(&rt->u.dst)); // 设置路由信息对象缓存
    }
```

上面的代码比较简单，首先调用 `sk_dst_check()` 查看 `路由信息对象` 是否被缓存，如果已经缓存，那么直接使用此 `路由信息对象`。否则调用  `ip_route_output()` 函数获取 `路由信息对象`，并且调用 `sk_dst_set()` 设置 `路由信息对象` 缓存。

`路由信息对象` 指明数据在传送过程的 `下一跳` 主机的信息（通常为网关），有了 `下一跳` 主机的信息，就可以把数据转发给 `下一跳` 主机，然后由 `下一跳` 主机继续完成发送工作。

我们继续分析 `udp_sendmsg()` 函数，代码如下：

```c
    ufh.saddr = rt->rt_src; // 设置源IP地址
    if (!ipc.addr) // 如果没有提供目标IP地址，使用路由信息的目标IP地址
        ufh.daddr = ipc.addr = rt->rt_dst;
    ufh.uh.len = htons(ulen);
    ufh.uh.check = 0;
    ufh.iov = msg->msg_iov;
    ufh.wcheck = 0;

    // 构建MAC头部、IP头部和UDP头部并且下发给IP协议层
    err = ip_build_xmit(sk, (sk->no_check == UDP_CSUM_NOXMIT
                                                ? udp_getfrag_nosum
                                                : udp_getfrag),
                        &ufh, ulen, &ipc, rt, msg->msg_flags);

out:
    ip_rt_put(rt);
    ...
    return err;
}
```

上面的代码主要把 `路由信息对象` 的源IP地址复制到 `ufh` 变量中，然后调用 `ip_build_xmit()` 函数完成数据发送的后续工作。`ip_build_xmit()` 函数的第一个参数用于复制 `UDP头部` 和负载数据到数据包的函数指针，IP 层通过调用此函数把 `UDP头部` 和负载数据复制到数据包中。

`ip_build_xmit()` 函数是 IP 协议层的实现，这里就不作说明，可以参考此文章：[IP协议源码分析](https://github.com/liexusong/linux-source-code-analyze/blob/master/ip-source-code.md)。

总的来说，`udp_sendmsg()` 函数的主要工作就是为要发送的数据包构建 `UDP头部`，然后把数据包交由 IP 层完成接下来的发送操作，所以 `UDP协议` 的发送过程比较简单。

## UDP数据包接收

当网卡设备接收到数据包后，会交由内核协议栈处理。内核协议栈对数据包的处理是由下至上，如下图所示：

![udp-recv-process](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/udp/udp-recv-process.png)

也就是说，物理层处理完数据包后会交由链路层处理，而链路层处理完交由网络层处理，以此类推。

所以当网络层（IP协议）处理完数据包后，会交由传输层处理，在本文中介绍的传输层协议是 `UDP协议`，所以这里主要介绍的是 `UDP协议` 对数据包的处理过程。

当 IP 协议层处理完数据包后，如果 IP 头部的上层协议字段（`protocol` 字段）指明的是 `UDP协议`，那么就会调用 `udp_rcv()` 函数处理数据包。下面我们来分析一下 `udp_rcv()` 函数的实现，代码如下：

```c
int udp_rcv(struct sk_buff *skb, unsigned short len)
{
    struct sock *sk;
    struct udphdr *uh;
    unsigned short ulen;
    struct rtable *rt = (struct rtable*)skb->dst; // 路由信息对象
    u32 saddr = skb->nh.iph->saddr; // 远端IP地址(源IP地址)
    u32 daddr = skb->nh.iph->daddr; // 本地IP地址(目标IP地址)

    uh = skb->h.uh; // UDP头部
    ...
    // 根据目标端口获取对用的 Socket 对象
    sk = udp_v4_lookup(saddr, uh->source, daddr, uh->dest, skb->dev->ifindex);
    if (sk != NULL) {
        udp_queue_rcv_skb(sk, skb); // 把数据包添加到Socket对象的receive_queue队列中
        sock_put(sk);
        return 0;
    }
    ...
}
```

`udp_rcv()` 函数主要完成两个工作：

*   调用 `udp_v4_lookup()` 函数获取目标端口对应的 `Socket` 对象。
*   调用 `udp_queue_rcv_skb()` 函数把数据包添加到 `Socket` 对象的 `receive_queue` 队列中。

`UDP协议` 使用了一个名为 `udp_hash` 的哈希表来保存所有绑定了端口的 `Socket` 对象，当应用程序调用 `bind()` 系统调用为 `Socket` 对象绑定端口时，就会将此 `Socket` 对象添加到 `udp_hash` 哈希表中。

而 `udp_v4_lookup()` 函数就是根据目标端口从 `udp_hash` 哈希表中获取对应的 `Socket` 对象，`udp_v4_lookup()` 函数实现如下：

```c
__inline__ struct sock *
udp_v4_lookup(u32 saddr, u16 sport, u32 daddr, u16 dport, int dif)
{
    struct sock *sk;

    read_lock(&udp_hash_lock); // 为udp_hash哈希表上锁
    sk = udp_v4_lookup_longway(saddr, sport, daddr, dport, dif);
    if (sk)
        sock_hold(sk);
    read_unlock(&udp_hash_lock);
    return sk;
}
```

`udp_v4_lookup()` 函数首先为 `udp_hash` 哈希表上锁，然后调用 `udp_v4_lookup_longway()` 函数从 `udp_hash` 哈希表中获取对应目标端口的 `Socket` 对象，`udp_v4_lookup_longway()` 函数的实现如下：

```c
struct sock *
udp_v4_lookup_longway(u32 saddr, // 源IP地址(远端IP地址)
                      u16 sport, // 源端口(远端端口)
                      u32 daddr, // 目标IP地址(本地IP地址)
                      u16 dport, // 目标端口(本地端口)
                      int dif)
{
    struct sock *sk, *result = NULL;
    unsigned short hnum = ntohs(dport);
    int badness = -1;

    // 根据目标端口从 udp_hash 哈希表中获取对应的 Socket 对象
    for (sk = udp_hash[hnum&(UDP_HTABLE_SIZE - 1)]; sk != NULL; sk = sk->next) {
        if (sk->num == hnum) { // 对比目标端口是否匹配
            int score = 0;

            if (sk->rcv_saddr) { // 如果Socket设置了固定的本地接收IP
                if(sk->rcv_saddr != daddr) // 对比目标IP地址是否匹配
                    continue;
                score++;
            }

            if (sk->daddr) { // 如果Socket设置了固定的远端接收IP
                if(sk->daddr != saddr) // 对比源IP地址是否匹配
                    continue;
                score++;
            }

            if (sk->dport) { // 如果Socket设置了固定的远端接收端口
                if(sk->dport != sport) // 对比源端口是否匹配
                    continue;
                score++;
            }

            if (sk->bound_dev_if) { // 如果Socket设置了固定的接收网络设备
                if(sk->bound_dev_if != dif) // 对比接收设备是否匹配
                    continue;
                score++;
            }

            if (score == 4) { // 完美匹配, 那么直接返回即可
                result = sk;
                break;
            } else if(score > badness) { // 否则使用分数最高的Socket对象
                result = sk;
                badness = score;
            }
        }
    }

    return result;
}
```

`udp_v4_lookup_longway()` 函数的主要逻辑就是根据目标端口从 `udp_hash` 哈希表中获取对应的 `Socket` 对象。

由于同一个端口有可能绑定了多个 `Socket` 对象，所以 `udp_v4_lookup_longway()` 函数查找 `Socket` 对象时使用了最优匹配，也就是说除了匹配目标端口外，还可能会匹配源 IP 地址、源端口和目标 IP 地址等。

找到目标端口对应的 `Socket` 对象后，就可以调用 `udp_queue_rcv_skb()` 函数把数据包添加到 `Socket` 对象的 `receive_queue` 队列中，`udp_queue_rcv_skb()` 函数实现如下：

```c
static int udp_queue_rcv_skb(struct sock * sk, struct sk_buff *skb)
{
    ...
    if (sock_queue_rcv_skb(sk, skb) < 0) {
        ...
        return -1;
    }
    return 0;
}
```

从上面代码可以看出，`udp_queue_rcv_skb()` 函数最终会调用 `sock_queue_rcv_skb()` 函数完成任务，所以我们来分析一下 `sock_queue_rcv_skb()` 函数的实现，代码如下：

```c
static inline int sock_queue_rcv_skb(struct sock *sk, struct sk_buff *skb)
{
    ...
    // 把数据包添加到Socket对象的receive_queue队列
    skb_queue_tail(&sk->receive_queue, skb);
    if (!sk->dead)
        sk->data_ready(sk, skb->len); // 唤醒等待Socket对象就绪的进程
    return 0;
}
```

`sock_queue_rcv_skb()` 通过调用 `skb_queue_tail()` 函数把 `skb` 数据包添加到 `Socket` 对象的 `receive_queue` 队列中，并且唤醒等待 `Socket` 对象就绪的进程。

当把数据包添加到 `Socket` 对象的 `receive_queue` 队列后，`UDP协议` 的接收工作就此完毕。

