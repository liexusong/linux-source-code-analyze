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

![lvs-roles](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/lvs-roles.png)

从上图可以看出，`ip_vs_service` 对象的 `destinations` 字段用于保存 `ip_vs_dest` 对象的列表，而 `scheduler` 字段指向了一个  `ip_vs_scheduler` 对象。

`ip_vs_scheduler` 对象的 `schedule` 字段指向了一个调度算法函数，通过这个调度函数可以从 `ip_vs_service` 对象的  `ip_vs_dest` 对象列表中选择一个合适的真实服务器。

那么，`ip_vs_service` 对象和 `ip_vs_dest` 对象的信息怎么来的呢？答案是通过用户配置创建。例如可以通过下面的命令来创建 `ip_vs_service` 对象和 `ip_vs_dest` 对象：

```bash
node1 ]# ipvsadm -A -t node1:80 -s wrr
node1 ]# ipvsadm -a -t node1:80 -r node2 -m -w 3
node1 ]# ipvsadm -a -t node1:80 -r node3 -m -w 5
```

第一行用于创建一个 `ip_vs_service` 对象，而第二和第三行用于向 `ip_vs_service` 对象添加 `ip_vs_dest` 对象到 `destinations` 列表中。关于 LVS 的配置这里不作详细介绍，读者可以参考其他关于 LVS 配置的资料。

#### ip_vs_service 对象创建

我们来看看 LVS 源码是怎么创建一个 `ip_vs_service` 对象的，创建 `ip_vs_service` 对象通过 `ip_vs_add_service()` 函数完成，如下：

```c
static int
ip_vs_add_service(struct ip_vs_rule_user *ur, struct ip_vs_service **svc_p)
{
    int ret = 0;
    struct ip_vs_scheduler *sched;
    struct ip_vs_service *svc = NULL;

    sched = ip_vs_scheduler_get(ur->sched_name); // 根据调度器名称获取调度策略对象
    ...
    // 申请一个 ip_vs_service 对象
    svc = (struct ip_vs_service *)kmalloc(sizeof(struct ip_vs_service), GFP_ATOMIC);
    ...
    memset(svc, 0, sizeof(struct ip_vs_service));
    // 设置 ip_vs_service 对象的各个字段
    svc->protocol = ur->protocol;       // 协议
    svc->addr = ur->vaddr;              // 虚拟IP
    svc->port = ur->vport;              // 虚拟端口
    svc->fwmark = ur->vfwmark;          // 防火墙标记
    svc->flags = ur->vs_flags;          // 标志位
    svc->timeout = ur->timeout * HZ;    // 超时时间
    svc->netmask = ur->netmask;         // 网络掩码

    INIT_LIST_HEAD(&svc->destinations);
    svc->sched_lock = RW_LOCK_UNLOCKED;
    svc->stats.lock = SPIN_LOCK_UNLOCKED;

    ret = ip_vs_bind_scheduler(svc, sched); // 绑定调度器
    ...
    ip_vs_svc_hash(svc); // 添加到ip_vs_service对象的hash表中
    ...
    *svc_p = svc;
    return 0;
}
```

先说明一下，参数 `ur` 是用户通过命令行配置的规则信息。上面的代码主要完成以下几个工作：

*   通过调用 `ip_vs_scheduler_get()` 函数来获取一个 `ip_vs_scheduler` (调度器) 对象。

*   然后申请一个 `ip_vs_service` 对象并且根据用户的配置设置其各个参数，并且把调度器对象绑定这个 `ip_vs_service` 对象。

*   最后把 `ip_vs_service` 对象添加到 `ip_vs_service` 对象的全局哈希表中（这是由于可以创建多个 `ip_vs_service` 对象，这些对象通过一个全局哈希表来存储）。

#### ip_vs_dest 对象创建

创建 `ip_vs_dest` 对象通过 `ip_vs_add_dest()` 函数完成，代码如下：

```c
static int ip_vs_add_dest(struct ip_vs_service *svc, struct ip_vs_rule_user *ur)
{
    struct ip_vs_dest *dest;
    __u32 daddr = ur->daddr; // 目的IP
    __u16 dport = ur->dport; // 目的端口
    int ret;
    ...
    // 调用 ip_vs_new_dest() 函数创建一个 ip_vs_dest 对象
    ret = ip_vs_new_dest(svc, ur, &dest);
    ...
    // 把 ip_vs_dest 对象添加到 ip_vs_service 对象的 destinations 列表中
    list_add(&dest->n_list, &svc->destinations); 
    svc->num_dests++;

    /* 调用调度器的 update_service() 方法更新 ip_vs_service 对象 */
    svc->scheduler->update_service(svc);
    ...
    return 0;
}
```

`ip_vs_add_dest()` 函数主要通过调用 `ip_vs_new_dest()` 创建一个 `ip_vs_dest` 对象，然后将其添加到 `ip_vs_service` 对象的 `destinations` 列表中。我们来看看 `ip_vs_new_dest()` 函数的实现：

```c
static int
ip_vs_new_dest(struct ip_vs_service *svc,
               struct ip_vs_rule_user *ur,
               struct ip_vs_dest **destp)
{
    struct ip_vs_dest *dest;
    ...
    *destp = dest = (struct ip_vs_dest*)kmalloc(sizeof(struct ip_vs_dest), GFP_ATOMIC);
    ...
    memset(dest, 0, sizeof(struct ip_vs_dest));
    // 设置 ip_vs_dest 对象的各个字段
    dest->protocol = svc->protocol; // 协议
    dest->vaddr = svc->addr;        // 虚拟IP
    dest->vport = svc->port;        // 虚拟端口
    dest->vfwmark = svc->fwmark;    // 虚拟网络掩码
    dest->addr = ur->daddr;         // 真实IP
    dest->port = ur->dport;         // 真实端口

    atomic_set(&dest->activeconns, 0);
    atomic_set(&dest->inactconns, 0);
    atomic_set(&dest->refcnt, 0);

    INIT_LIST_HEAD(&dest->d_list);
    dest->dst_lock = SPIN_LOCK_UNLOCKED;
    dest->stats.lock = SPIN_LOCK_UNLOCKED;
    __ip_vs_update_dest(svc, dest, ur);
    ...
    return 0;
}
```

`ip_vs_new_dest()` 函数的实现也比较简单，首先通过调用 `kmalloc()` 函数申请一个 `ip_vs_dest` 对象，然后根据用户配置的规则信息来初始化 `ip_vs_dest` 对象的各个字段。

#### ip_vs_scheduler 对象

`ip_vs_scheduler` (调度器) 对象用于从 `ip_vs_service` 对象的 `destinations` 列表中选择一个合适的 `ip_vs_dest` 对象，其定义如下：

```c
struct ip_vs_scheduler {
    struct list_head    n_list;     // 连接所有调度策略
    char                *name;      // 调度策略名称
    atomic_t            refcnt;     // 应用计数器
    struct module       *module;    // 模块对象(如果是通过模块引入的)

    int (*init_service)(struct ip_vs_service *svc);   // 用于初始化服务
    int (*done_service)(struct ip_vs_service *svc);   // 用于停止服务
    int (*update_service)(struct ip_vs_service *svc); // 用于更新服务

    // 用于获取一个真实服务器对象 (Real-Server)
    struct ip_vs_dest *(*schedule)(struct ip_vs_service *svc, struct iphdr *iph);
};
```

`ip_vs_scheduler` 对象的各个字段都在注释说明了，其中 `schedule` 字段是一个函数的指针，其指向一个调度函数，用于从 `ip_vs_service` 对象的 `destinations` 列表中选择一个合适的 `ip_vs_dest` 对象。

我们可以通过一个最简单的调度模块（轮询调度模块）来分析 `ip_vs_scheduler` 对象的工作原理（文件路径：`/net/ipv4/ipvs/ip_vs_rr.c`）：

```c
static struct ip_vs_scheduler ip_vs_rr_scheduler = {
    {0},                /* n_list */
    "rr",               /* name */
    ATOMIC_INIT(0),     /* refcnt */
    THIS_MODULE,        /* this module */
    ip_vs_rr_init_svc,  /* service initializer */
    ip_vs_rr_done_svc,  /* service done */
    ip_vs_rr_update_svc,/* service updater */
    ip_vs_rr_schedule,  /* select a server from the destination list */
};
```

首先轮询调度模块定义了一个 `ip_vs_scheduler` 对象，其中 `schedule` 字段设置为 `ip_vs_rr_schedule()` 函数。我们来看看 `ip_vs_rr_schedule()` 函数的实现：

```c
static struct ip_vs_dest *
ip_vs_rr_schedule(struct ip_vs_service *svc, struct iphdr *iph)
{
    register struct list_head *p, *q;
    struct ip_vs_dest *dest;

    write_lock(&svc->sched_lock);
    p = (struct list_head *)svc->sched_data; // 最后一次被调度的位置
    p = p->next;
    q = p;
    // 遍历 destinations 列表
    do {
        if (q == &svc->destinations) {
            q = q->next;
            continue;
        }
        dest = list_entry(q, struct ip_vs_dest, n_list);
        // 找到一个权限值大于 0 的 ip_vs_dest 对象
        if (atomic_read(&dest->weight) > 0)
            goto out;
        q = q->next;
    } while (q != p);
    write_unlock(&svc->sched_lock);

    return NULL;

  out:
    svc->sched_data = q; // 设置最后一次被调度的位置
    ...
    return dest;
}
```

`ip_vs_rr_schedule()` 函数是轮询调度算法的实现，其实现原理如下：

*   `ip_vs_service` 对象的 `sched_data` 字段保存了最后一次调度的位置，所以每次调度时都是从这个字段读取到最后一次调度的位置。

*   从最后一次调度的位置开始遍历，找到一个权限值（weight）大于 0 的 `ip_vs_dest` 对象。

*   如果找到就把 `ip_vs_service` 对象的 `sched_data` 字段设置为最后被选择的 `ip_vs_dest` 对象的位置。

其原理可以通过以下图片说明：

![](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/lvs-scheduler.png)

上图描述的原理还是比较简单，首先从 `sched_data` 处开始遍历，查找一个合适的 `ip_vs_dest` 对象，然后更新 `sched_data` 的位置。

另外，由于 `LVS` 可以存在多种不同的调度对象（提供不同的调度算法），所以 `LVS` 把这些调度对象通过一个链表（`ip_vs_schedulers`）存储起来，而这些调度对象可以通过调度对象的名字（`name` 字段）来查询。

可以通过调用 `register_ip_vs_scheduler()` 函数向 `LVS` 注册调度对象，而通过调用 `ip_vs_scheduler_get()` 函数来获取指定名字的调度对象，这两个函数的实现比较简单，这里就不作详细介绍了。

#### ip_vs_conn 对象

`ip_vs_conn` 对象用于维护 `客户端` 与 `真实服务器` 之间的关系，为什么需要维护它们之间的关系？原因是 `TCP协议` 面向连接的协议，所以每次调度都必须选择相同的真实服务器，否则连接就会失效。

![lvs-connection](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/lvs-connection.png)

如上图所示，刚开始时调度器选择了 `Real-Server(1)` 服务器进行处理客户端请求，但第二次调度时却选择了 `Real-Server(2)` 来处理客户端请求。

由于 `TCP协议` 需要客户端与服务器进行连接，但第二次请求的服务器发生了变化，所以连接状态就失效了，这就为什么 `LVS` 需要维持客户端与真实服务器连接关系的原因。

`LVS` 通过 `ip_vs_conn` 对象来维护客户端与真实服务器之间的连接关系，其定义如下：

```c
struct ip_vs_conn {
    struct list_head    c_list;     /* 用于连接到哈希表 */

    __u32               caddr;      /* 客户端IP地址 */
    __u32               vaddr;      /* 虚拟IP地址 */
    __u32               daddr;      /* 真实服务器IP地址 */
    __u16               cport;      /* 客户端端口 */
    __u16               vport;      /* 虚拟端口 */
    __u16               dport;      /* 真实端口 */
    __u16               protocol;   /* 协议类型（UPD/TCP） */
    ...
    /* 用于发送数据包的函数 */
    int (*packet_xmit)(struct sk_buff *skb, struct ip_vs_conn *cp);
    ...
};
```

