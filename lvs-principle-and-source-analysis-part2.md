# LVS原理与实现 - 实现篇

在上一篇文章中，我们主要介绍了 `LVS` 的原理，接下来我们将会介绍 `LVS` 的代码实现。

>   本文使用的内核版本是：2.4.23，而 LVS 的代码在路径：`/src/net/ipv4/ipvs` 中。

## Netfilter

在介绍 `LVS` 的实现前，我们需要了解以下 `Netfilter` 这个功能，因为 `LVS` 的实现使用了 `Netfilter` 的功能。

>   `Netfilter`：顾名思义就是网络过滤器（Network Filter），是 Linux 系统特有的网络子系统，用于过滤或修改进出内核协议栈的网络数据包。一般可以用来实现网络防火墙功能，其中 `iptables` 就是基于 `Netfilter` 实现的。

Linux 内核处理进出网络协议栈的数据包分为5个不同的阶段，`Netfilter` 通过这5个阶段注入钩子函数（Hooks Function）来实现对数据包的过滤和修改。如下图的蓝色方框所示：

![netfilter-hooks](C:\books\linux-source-code-analyze\images\netfilter-hooks.png)

这5个阶段分为：

*   `PER_ROUTING`：**路由前阶段**，发生在内核对数据包进行路由判决前。
*   `LOCAL_IN`：**本地上送阶段**，发生在内核通过路由判决后。如果数据包是发送给本机的，那么就把数据包上送到上层协议栈。
*   `FORWARD`：**转发阶段**，发生在内核通过路由判决后。如果数据包不是发送给本机的，那么就把数据包转发出去。
*   `LOCAL_OUT`：**本地发送阶段**，发生在对发送数据包进行路由判决之前。
*   `POST_ROUTING`：**路由后阶段**，发生在对发送数据包进行路由判决之后。

当向 `Netfilter` 的这5个阶段注册钩子函数后，内核会在处理数据包时，根据所在的不同阶段来调用这些钩子函数对数据包进行处理。向 `Netfilter` 注册钩子函数可以通过函数 `nf_register_hook()` 来进行，`nf_register_hook()` 函数的原型如下：

```c
int nf_register_hook(struct nf_hook_ops *reg);
```

其中参数 `reg` 是类型为 `struct nf_hook_ops` 结构的指针，`struct nf_hook_ops` 结构的定义如下：

```c
typedef unsigned int nf_hookfn(unsigned int hooknum,
                               struct sk_buff **skb,
                               const struct net_device *in,
                               const struct net_device *out,
                               int (*okfn)(struct sk_buff *));

struct nf_hook_ops
{
    struct list_head list;
    nf_hookfn *hook;
    int pf;
    int hooknum;
    int priority;
};
```

`struct nf_hook_ops` 结构各个字段的作用如下：

*   `list`：用于连接同一阶段中所有相同的钩子函数列表。
*   `hook`：钩子函数指针。
*   `pf`：协议类型，因为 `Netfilter` 可以用于不同的协议，如IPV4和IPV6等。
*   `hooknum`：所处的阶段，也就是上面所说的5个不同的阶段。
*   `priority`：优先级，值越大优先级约小。

