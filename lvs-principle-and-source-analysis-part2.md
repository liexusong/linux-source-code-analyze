# LVS原理与实现 - 实现篇

在上一篇文章中，我们主要介绍了 `LVS` 的原理，接下来我们将会介绍 `LVS` 的代码实现。

>   本文使用的内核版本是：2.4.23，而 LVS 的代码在路径：`/src/net/ipv4/ipvs` 中。

## Netfilter

在介绍 `LVS` 的实现前，我们需要了解以下 `Netfilter` 这个功能，因为 `LVS` 的实现使用了 `Netfilter` 的功能。

>   `Netfilter`：顾名思义就是网络过滤器（Network Filter），是 Linux 系统特有的网络子系统，用于过滤或修改进出内核协议栈的网络数据包。一般可以用来实现网络防火墙功能，其中 `iptables` 就是基于 `Netfilter` 实现的。

Linux 内核处理进出网络协议栈的数据包分为5个不同的阶段，`Netfilter` 通过这5个阶段注入钩子函数（Hooks Function）来实现对数据包的过滤和修改。如下图的蓝色方框所示：

![netfilter-hooks](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/netfilter-hooks.png)

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
*   `pf`：协议类型，因为 `Netfilter` 可以用于不同的协议，如 IPV4 和 IPV6 等。
*   `hooknum`：所处的阶段，也就是上面所说的5个不同的阶段。
*   `priority`：优先级，值越大优先级约小。

所以要使用 `Netfilter` 对网络数据包进行处理，只需要编写好处理数据包的钩子函数，然后通过调用 `nf_register_hook()` 函数向 `Netfilter` 注册即可。

另外，钩子函数 `nf_hookfn` 的原型如下：

```c
typedef unsigned int nf_hookfn(unsigned int hooknum, struct sk_buff **skb, 
    const struct net_device *in, const struct net_device *out, int (*okfn)(struct sk_buff *));
```

其参数说明如下：

*   `hooknum`：所处的阶段，也就是上面所说的5个不同的阶段。
*   `skb`：要处理的数据包。
*   `in`：输入设备。
*   `out`：输出设备。
*   `okfn`：如果钩子函数执行成功，即调用这个函数完成对数据包的后续处理工作。

`Netfilter` 相关的知识点就介绍到这里，以后有机会会详解讲解 `Netfilter` 的原理和现实。

## LVS 实现

前面我们主要简单介绍了 `Netfilter` 的使用，接下来我们将要分析 `LVS` 的代码实现。

### 1. 钩子函数注册

`LVS` 主要通过向 `Netfilter` 的3个阶段注册钩子函数来对数据包进行处理，如下图：

![lvs-hooks](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/lvs-hooks.png)

*   在 `LOCAL_IN` 阶段注册了 `ip_vs_in()` 钩子函数。
*   在 `FORWARD` 阶段注册了 `ip_vs_out()` 钩子函数。
*   在 `POST_ROUTING` 阶段注册了 `ip_vs_post_routing()` 钩子函数。

我们在 LVS 的初始化函数 `ip_vs_init()` 可以找到这些钩子函数的注册代码，如下：

```c
static struct nf_hook_ops ip_vs_in_ops = {
    { NULL, NULL },
    ip_vs_in, PF_INET, NF_IP_LOCAL_IN, 100
};

static struct nf_hook_ops ip_vs_out_ops = {
    { NULL, NULL },
    ip_vs_out, PF_INET, NF_IP_FORWARD, 100
};

static struct nf_hook_ops ip_vs_post_routing_ops = {
    { NULL, NULL },
    ip_vs_post_routing, PF_INET, NF_IP_POST_ROUTING, NF_IP_PRI_NAT_SRC-1
};

static int __init ip_vs_init(void)
{
    int ret;
    ...
    ret = nf_register_hook(&ip_vs_in_ops);
    ...
    ret = nf_register_hook(&ip_vs_out_ops);
    ...
    ret = nf_register_hook(&ip_vs_post_routing_ops);
    ...
    return ret;
}
```

*   `LOCAL_IN` 阶段：在路由判决之后，如果发现数据包是发送给本机的，那么就调用 `ip_vs_in()` 函数对数据包进行处理。

*   `FORWARD` 阶段：在路由判决之后，如果发现数据包不是发送给本机的，调用 `ip_vs_out()` 函数对数据包进行处理。

*   `POST_ROUTING` 阶段：在发送数据前，需要调用 `ip_vs_post_routing()` 函数对数据包进行处理。

### 2. LVS 角色介绍

在介绍这些钩子函数之前，我们先来了解一下 `LVS` 中的四个角色。如下：

*   `ip_vs_service`：服务配置对象，主要用于保存 LVS 的配置信息，如 支持的 `传输层协议`、`虚拟IP` 和 `端口` 等。

*   `ip_vs_dest`：真实服务器对象，主要用于保存真实服务器 (Real-Server) 的配置，如 `真实IP`、`端口` 和 `权重` 等。

*   `ip_vs_scheduler`：调度器对象，主要通过使用不同的调度算法来选择合适的真实服务器对象。

*   `ip_vs_conn`：连接对象，主要为了维护相同的客户端与真实服务器之间的连接关系。这是由于 TCP 协议是面向连接的，所以同一个的客户端每次选择真实服务器的时候必须保存一致，否则会出现连接中断的情况，而连接对象就是为了维护这种关系。

各个角色之间的关系如下图所示：

![lvs-roles](F:\linux-source-code-analyze\images\lvs-roles.png)