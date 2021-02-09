# ARP协议与邻居子系统剖析

学习过 TCP/IP 协议的同学都应该了解过 TCP/IP 五层网络模型，如下图：

![tcp-ip-layer](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/tcp-ip-layer.png)

从上图可以看出，`ARP协议` 位于 TCP/IP 五层网络模型的 `网络层`。那么，`ARP协议` 的用途是什么呢？

## ARP协议介绍

在局域网中（同一个路由器内），主机与主机之间需要通过 MAC 地址进行通讯。但由于 MAC 地址过于复杂，不容易被人类记忆。所以，人们更倾向于使用更容易记忆的 IP 地址。

但局域网只能使用 MAC 地址通讯，那有什么办法可以通过主机的 IP 地址来获取到主机的 MAC 地址呢？`ARP协议` 就应运而生。

>   *ARP（Address Resolution Protocol）*  即地址解析协议， 用于实现从 IP 地址到 MAC 地址的映射，即询问目标IP对应的MAC地址。

`ARP协议` 通过广播消息，向局域网的所有主机广播 `ARP请求消息`，从而询问主机的 IP 地址对应的 MAC 地址，如下图：

![arp-broadcast](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/arp-broadcast.png)

如上图所示，A主机要与B主机通信，但是只知道B主机的 IP 地址，所以这时可以向局域网广播一条 `ARP请求消息`，用于询问 IP 地址为 `192.168.1.2` 的主机所对应的 MAC 地址。

由于 `ARP请求消息` 是广播消息，所以局域网的所有主机都会收到这条消息，但只有对应 IP 地址的主机才会回答这条消息。如上图的B主机会回复一条 `ARP应答消息`，用于告诉A主机自己的 MAC 地址。

当A主机知道了B主机的 MAC 地址后，就能通过 MAC 地址与B主机进行通信了。

## ARP协议头部

每种网络协议都有其协议头部，用于本协议的通信，`ARP协议` 的头部格式如下：

![arp-header](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/arp-header.png)

上图是 `ARP协议` 头部各个字段的信息，其代码结构定义如下（路径: `/src/include/linux/if_arp.h`）：

```c
struct arphdr
{
    unsigned short  ar_hrd;     /* format of hardware address   */
    unsigned short  ar_pro;     /* format of protocol address   */
    unsigned char   ar_hln;     /* length of hardware address   */
    unsigned char   ar_pln;     /* length of protocol address   */
    unsigned short  ar_op;      /* ARP opcode (command)         */

#if 0
   /* 下面部分没有定义, 因为不同的链路层协议使用的地址格式不一定相同,
    * 所以下面只是以太网和IP协议的示例而已.
    *  Ethernet looks like this : This bit is variable sized however...
    */
    unsigned char       ar_sha[ETH_ALEN];   /* sender hardware address  */
    unsigned char       ar_sip[4];          /* sender IP address        */
    unsigned char       ar_tha[ETH_ALEN];   /* target hardware address  */
    unsigned char       ar_tip[4];          /* target IP address        */
#endif
}
```

从代码可以看出，`arphdr` 结构的各个字段与上图一一对应。下面说说各个字段的作用：

*   `ar_hrd`：硬件类型，如硬件类型是以太网，那么设置为 `1`。

*   `ar_pro`：协议类型，由于 ARP 协议支持将多种不同协议地址转换成 MAC 地址（如 IP 协议、AX.25 协议等），所以需要通过这个字段指明要转换的协议类型。如 IP 协议设置为 `0x0800`。

*   `ar_hln`：硬件地址长度，如以太网地址长度是 6。

*   `ar_hln`：协议地址长度，如 IP 地址长度是 4。

*   `ar_op`：操作码，如 `ARP请求` 设置为 1，而 `ARP应答` 设置为 2。

下面的字段是不定长的，根据硬件类型和协议类型的改变而改变。

譬如：如果是硬件类型是以太网，并且协议类型是 IP 协议，那么 `ar_sha` 字段和 `ar_tha` 字段分别为 6 个字节，而 `ar_sip` 字段和 `ar_tip` 字段分别为 4 个字节。

## 邻居子系统

首先说明一下什么是 `邻居`：在同一局域网中，每一台主机都可以称为其他主机的 `邻居`。例如 `Windows` 系统可以在网络中查看到同一局域网的邻居主机，如下图：

![image-20210202143741734](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/neighbour-nodes.png)

如上图所示，每一台主机都可以称为 `邻居节点`。

在 Linux 内核中，也把局域网中的每台主机称为 `邻居节点`，使用结构 `neighbour` 来描述，`neighbour` 结构定义如下：

```c
struct neighbour
{
  struct neighbour    *next;      // 用于连接哈希表中相同哈希值的邻居节点
  struct neigh_table  *tbl;       // 管理邻居节点的邻居表结构
  struct neigh_parms  *parms;     // 参数列表
  struct net_device   *dev;       // 可以与这个邻居节点通信的设备
  unsigned long       used;       // 最后使用时间
  unsigned long       confirmed;  // 最后确认时间
  unsigned long       updated;    // 最后更新时间
  __u8                flags;      // 标志位
  __u8                nud_state;  // 邻居节点所处于的状态
  __u8                type;       // 类型
  __u8                dead;       // 是否已经失效
  atomic_t            probes;
  rwlock_t            lock;       // 锁
                                  // 邻居节点的硬件地址
  unsigned char       ha[(MAX_ADDR_LEN+sizeof(unsigned long)-1)&~(sizeof(unsigned long)-1)];
  struct hh_cache     *hh;
  atomic_t            refcnt;         // 引用计数器
  int                 (*output)(struct sk_buff *skb); // 发送数据给此邻居节点的接口
  struct sk_buff_head arp_queue;      // 等待ARP回复的数据包列表(需要发送的数据包列表)
  struct timer_list   timer;          // 定时器
  struct neigh_ops    *ops;           // 操作方法列表
  u8                  primary_key[0]; // 要转换成MAC地址的上层协议地址(如IP地址)
};
```

在 `neighbour` 结构中，比较重要的字段有：

*   `ha`：邻居节点的硬件地址，因为与邻居节点通信需要知道其硬件地址（MAC地址）。
*   `output`：向邻居节点发送数据的接口，当要向邻居节点发送数据时，使用这个接口把数据发送出去。
*   `dev`：输出设备，如果向当前邻居节点发送数据，需要通过这个设备来发送。
*   `primary_key`：要转换成 MAC 地址的上层协议地址（如 IP 地址），由于上层协议不确定，所以这里设置 `primary_key` 为 `柔性数组（可变大小数组）`（不了解柔性数组可以查阅相关的资料）。

如下图所示：

![](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/neighbour-struct.png)

所以，当本机需要向某一台 `邻居节点` 主机发送数据时，首先需要通过上层协议地址与输出设备查找对应的 `neighbour` 对象是否已经存在。如果存在，那么就使用这个 `neighbour` 对象。否则，就新创建一个 `neighbour` 对象，并初始化其各个字段。

## 查找邻居节点信息

要查找一个 `邻居节点` 的信息，可以通过调用 `neigh_lookup()` 函数来完成，其实现如下：

```c
struct neighbour *
neigh_lookup(struct neigh_table *tbl, const void *pkey, struct net_device *dev)
{
  struct neighbour *n;
  u32 hash_val;
  int key_len = tbl->key_len;

  hash_val = tbl->hash(pkey, dev); // 通过IP地址与设备计算邻居节点信息的哈希值

  read_lock_bh(&tbl->lock);

  // 通过设备和IP地址从邻居节点哈希表中查找邻居节点信息
  for (n = tbl->hash_buckets[hash_val]; n; n = n->next) {
    if (dev == n->dev && memcmp(n->primary_key, pkey, key_len) == 0) {
      neigh_hold(n);
      break;
    }
  }

  read_unlock_bh(&tbl->lock);

  return n; // 返回邻居节点信息对象
}
```

`neigh_lookup()` 函数的参数含义如下：

*   `tbl`：邻居节点管理表。
*   `pkey`：上层协议地址（如 IP 地址）。
*   `dev`：输出设备对象。

`neigh_lookup()` 函数工作原理如下：

*   首先通过上层协议地址与设备计算邻居节点信息的哈希值。
*   然后在邻居节点哈希表中查找对应的邻居节点信息，如果找到即返回邻居节点信息，否则返回NULL。

## 创建邻居节点信息

当 `邻居节点` 信息不存在时，可以通过调用 `neigh_create()` 函数来创建，其实现如下：

```c
struct neighbour *
neigh_create(struct neigh_table *tbl, const void *pkey, struct net_device *dev)
{
    struct neighbour *n, *n1;
    u32 hash_val;
    int key_len = tbl->key_len;
    int error;

    n = neigh_alloc(tbl); // 创建一个新的邻居节点信息对象
    if (n == NULL)
        return ERR_PTR(-ENOBUFS);

    memcpy(n->primary_key, pkey, key_len); // 复制上层协议地址到 primary_key 字段

    n->dev = dev; // 绑定输出设备
    dev_hold(dev);

    // 对于ARP协议会调用 arp_constructor() 函数
    if (tbl->constructor && (error = tbl->constructor(n)) < 0) {
        neigh_release(n);
        return ERR_PTR(error);
    }
    ...
    hash_val = tbl->hash(pkey, dev);

    write_lock_bh(&tbl->lock);
    ...
    // 把邻居节点信息添加到邻居节点信息管理哈希表中
    n->next = tbl->hash_buckets[hash_val];
    tbl->hash_buckets[hash_val] = n;
    n->dead = 0;
    neigh_hold(n);

    write_unlock_bh(&tbl->lock);

    return n;
}
```

`neigh_create()` 函数的工作原理如下：

*   调用 `neigh_alloc()` 函数向内存申请一个新的邻居节点信息对象。
*   把上层协议地址（如 IP 地址）复制到 `primary_key` 字段中。
*   绑定输出设备到 `dev` 字段。
*   调用邻居节点信息管理哈希表的 `constructor()` 方法来初始化邻居节点信息对象，对于 `ARP协议` 这里调用的是 `arp_constructor()` 函数。
*   把新创建的邻居节点信息对象添加到邻居节点信息管理哈希表中。

对于 `arp_constructor()` 函数，主要是用于初始化邻居节点信息对象的操作方法列表和 `output` 字段，如下：

```c
static struct neigh_ops arp_generic_ops = {
    AF_INET,                  // family
    NULL,                     // destructor
    arp_solicit,              // solicit
    arp_error_report,         // error_report
    neigh_resolve_output,     // output
    neigh_connected_output,   // connected_output
    dev_queue_xmit,           // hh_output
    dev_queue_xmit            // queue_xmit
};

static int arp_constructor(struct neighbour *neigh)
{
    u32 addr = *(u32*)neigh->primary_key;
    struct net_device *dev = neigh->dev;
    ...
    neigh->type = inet_addr_type(addr);
    ...
    if (dev->hard_header_cache)
        neigh->ops = &arp_hh_ops;
    else
        neigh->ops = &arp_generic_ops;

    if (neigh->nud_state & NUD_VALID)
        neigh->output = neigh->ops->connected_output;
    else
        neigh->output = neigh->ops->output;

    return 0;
}
```

从 `arp_constructor()` 函数的实现可以知道，邻居节点信息对象的`ops` 字段被设置为 `arp_generic_ops`，而  `output` 字段会被设置为 `neigh_resolve_output()` 函数（当邻居节点信息对象不可用时）或者 `neigh_connected_output()` 函数（当邻居节点信息对象可用时）。

所以，一个新创建的 `邻居节点信息对象` 各个字段的值大概如下图所示：

![](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/neighbour-struct-new.png)

由于此时还不知道邻居节点的 MAC 地址，所以其 `ha` 字段的值为 0。

## 向邻居节点发送数据

当向邻居节点发送数据时，需要调用邻居节点信息对象的 `output` 接口。根据前面的分析，`output` 接口被设置为 `neigh_resolve_output()` 函数。我们来分析一下 `neigh_resolve_output()` 函数的实现：

```c
int neigh_resolve_output(struct sk_buff *skb)
{
    struct dst_entry *dst = skb->dst;
    struct neighbour *neigh;
    ...
    if (neigh_event_send(neigh, skb) == 0) {
        int err;
        struct net_device *dev = neigh->dev;

        if (dev->hard_header_cache && dst->hh == NULL) {
            ...
        } else {
            read_lock_bh(&neigh->lock);
            err = dev->hard_header(skb, dev, ntohs(skb->protocol), 
                                   neigh->ha, NULL, skb->len);
            read_unlock_bh(&neigh->lock);
        }
        if (err >= 0)
            return neigh->ops->queue_xmit(skb);
        kfree_skb(skb);
        return -EINVAL;
    }
    return 0;
    ...
}
```

`neigh_resolve_output()` 函数主要完成三件事：

*   调用 `neigh_event_send()` 函数发送一个查询邻居节点 MAC 地址的 ARP 请求。
*   调用设备的 `hard_header()` 方法设置数据包的目标 MAC 地址（如果邻居节点的 MAC 地址已经获取到）。
*   如果数据包的目标 MAC 地址设置成功，调用邻居节点信息对象的 `queue_xmit()` 方法把数据发送出去（对于 ARP 协议，`queue_xmit()` 方法对应的是 `dev_queue_xmit()` 函数）。

我们再来看看 `neigh_event_send()` 函数怎么发送 ARP 请求：

```c
int __neigh_event_send(struct neighbour *neigh, struct sk_buff *skb)
{
    write_lock_bh(&neigh->lock);
    if (!(neigh->nud_state & (NUD_CONNECTED|NUD_DELAY|NUD_PROBE))) {
        if (!(neigh->nud_state & (NUD_STALE|NUD_INCOMPLETE))) {
            if (neigh->parms->mcast_probes + neigh->parms->app_probes) {
                ...
                neigh->nud_state = NUD_INCOMPLETE;
                ...
                neigh->ops->solicit(neigh, skb); // 发送查询邻居节点MAC地址的ARP请求
                ...
            } else {
                ...
            }
        }

        if (neigh->nud_state == NUD_INCOMPLETE) {
            if (skb) {
                ...
                __skb_queue_head(&neigh->arp_queue, skb); // 把数据包添加到arp_queue队列中
            }
            write_unlock_bh(&neigh->lock);
            return 1;
        }
        ...
    }
    write_unlock_bh(&neigh->lock);
    return 0;
}
```

`__neigh_event_send()` 函数主要完成两个工作：

*   首先调用邻居节点信息对象的 `solicit()` 方法发送一个 ARP 请求，从前面的分析可知，`solicit()` 方法会被设置为 `arp_solicit()` 函数。
*   然后把要发送的数据包添加到邻居节点信息对象的 `arp_queue` 队列中，等待获取到邻居节点 MAC 地址后重新发送这个数据包。

邻居节点信息对象的 `arp_queue` 队列用于缓存等待发送的数据包，如下图：

![](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/neighbour-arp-queue.png)



## 发送 ARP 请求

通过前面的分析可知，当向邻居节点发送数据时，如果还不知道邻居节点的 MAC 地址，那么首先会调用 `arp_solicit()` 函数发送一个 `ARP请求` 来获取邻居节点的 MAC 地址，其实现如下：

```c
static void arp_solicit(struct neighbour *neigh, struct sk_buff *skb)
{
    u32 saddr;                               // 源IP地址(本地IP地址)
    u8 *dst_ha = NULL;                       // 接收ARP请求目标MAC地址(发广播信息设置为NULL)
    struct net_device *dev = neigh->dev;     // 输出设备
    u32 target = *(u32*)neigh->primary_key;  // 要查询的邻居节点IP地址
    ...

    if (skb && inet_addr_type(skb->nh.iph->saddr) == RTN_LOCAL)
        saddr = skb->nh.iph->saddr;
    else
        saddr = inet_select_addr(dev, target, RT_SCOPE_LINK);
    ...

    arp_send(ARPOP_REQUEST, ETH_P_ARP, target, dev, saddr,
             dst_ha, dev->dev_addr, NULL);
    ...
}
```

`arp_solicit()` 函数最终会调用 `arp_send()` 函数发送一个 ARP 请求，我们来分析一下 `arp_send()` 函数的实现：

```c
void arp_send(int type, int ptype, u32 dest_ip,
              struct net_device *dev, u32 src_ip,
              unsigned char *dest_hw, unsigned char *src_hw,
              unsigned char *target_hw)
{
    struct sk_buff *skb;
    struct arphdr *arp;
    unsigned char *arp_ptr;
    ...
    // 申请一个数据包对象
    skb = alloc_skb(sizeof(struct arphdr) + 2*(dev->addr_len+4)
                        + dev->hard_header_len + 15, GFP_ATOMIC);
    ...

    // ARP请求头部
    arp = (struct arphdr *)skb_put(skb, sizeof(struct arphdr) + 2*(dev->addr_len+4));

    skb->dev = dev;
    skb->protocol = __constant_htons(ETH_P_ARP);

    if (src_hw == NULL)
        src_hw = dev->dev_addr;
    if (dest_hw == NULL)
        dest_hw = dev->broadcast;

    // 下面设置ARP头部的各个字段信息
    ...
    switch (dev->type) {
    default:
        arp->ar_hrd = htons(dev->type);            // 设置硬件类型
        arp->ar_pro = __constant_htons(ETH_P_IP);  // 设置上层协议类型为IP协议
        break;
    ...
    }

    arp->ar_hln = dev->addr_len;            // 设置硬件地址长度
    arp->ar_pln = 4;                        // 设置上层协议地址长度
    arp->ar_op = htons(type);               // ARP请求操作码类型

    arp_ptr = (unsigned char *)(arp + 1);

    memcpy(arp_ptr, src_hw, dev->addr_len); // 复制源MAC地址(本机MAC地址)

    arp_ptr += dev->addr_len;
    memcpy(arp_ptr, &src_ip,4);             // 复制源IP地址(本机IP地址)

    // 复制目标MAC地址(对于查询请求设置为0)
    arp_ptr += 4;
    if (target_hw != NULL)
        memcpy(arp_ptr, target_hw, dev->addr_len);
    else
        memset(arp_ptr, 0, dev->addr_len);

    // 复制目标IP地址
    arp_ptr += dev->addr_len;
    memcpy(arp_ptr, &dest_ip, 4);
    ...
    dev_queue_xmit(skb); // 把数据发送出去
    return;
}
```

`arp_send()` 函数的工作也比较清晰：

*   首先调用 `alloc_skb()` 函数申请一个数据包对象。
*   然后设置 ARP 头部各个字段的信息。
*   调用 `dev_queue_xmit()` 函数把数据发送出去。

## 处理 ARP 回复

当邻居节点回复 MAC 地址查询 ARP 请求时，本机需要处理此 ARP 回复。本机通过 `arp_rcv()` 函数来处理 ARP 回复，代码如下：

```c
int arp_rcv(struct sk_buff *skb, struct net_device *dev, struct packet_type *pt)
{
    struct arphdr *arp = skb->nh.arph;
    unsigned char *arp_ptr = (unsigned char *)(arp+1);
    struct rtable *rt;
    unsigned char *sha, *tha;
    u32 sip, tip;
    u16 dev_type = dev->type;
    int addr_type;
    struct in_device *in_dev = in_dev_get(dev);
    struct neighbour *n;
    ...
    // 从ARP回复中获取数据
    sha = arp_ptr;             // 源MAC地址(邻居节点的MAC地址)
    arp_ptr += dev->addr_len;
    memcpy(&sip, arp_ptr, 4);  // 源IP地址(邻居节点的IP地址)

    arp_ptr += 4;
    tha = arp_ptr;             // 目标MAC地址(本机的MAC地址)

    arp_ptr += dev->addr_len;
    memcpy(&tip, arp_ptr, 4);  // 目标IP地址(本机的IP地址)
    ...

    n = __neigh_lookup(&arp_tbl, &sip, dev, 0); // 查找邻居节点信息对象
    if (n) {
        int state = NUD_REACHABLE;
        int override = 0;

        if (jiffies - n->updated >= n->parms->locktime)
            override = 1;

        if (arp->ar_op != __constant_htons(ARPOP_REPLY) 
            || skb->pkt_type != PACKET_HOST)
            state = NUD_STALE;

        neigh_update(n, sha, state, override, 1); // 更新邻居节点信息对象
        neigh_release(n);
    }
    ...
    return 0;
}
```

`arp_rcv()` 函数主要完成以下工作：

*   通过从 ARP 回复中获取到邻居节点的 MAC 地址和 IP 地址。
*   通过邻居节点的 IP 地址和输入设备来查找对应的邻居节点信息对象。
*   如果邻居节点信息对象已经存在，那么调用 `neigh_update()` 函数更新邻居节点信息对象的数据。

我们来看看  `neigh_update()` 函数怎么更新邻居节点信息对象的数据：

```c
int neigh_update(struct neighbour *neigh, const u8 *lladdr, u8 new, int override, int arp)
{
    u8 old;
    int err;
    int notify = 0;
    struct net_device *dev = neigh->dev;
    ...
    old = neigh->nud_state;    // 更新前邻居节点信息对象的状态
    ...
    neigh->nud_state = new;    // 更新邻居节点信息对象的状态
    if (lladdr != neigh->ha) { // 更新邻居节点信息对象的MAC地址
        memcpy(&neigh->ha, lladdr, dev->addr_len);
        ...
    }
    ...
    if (!(old&NUD_VALID)) {
        struct sk_buff *skb;

        // 如果 arp_queue 队列中有等待发送的数据包, 现在可以把这些数据包发送出去
        while (neigh->nud_state & NUD_VALID &&
               (skb = __skb_dequeue(&neigh->arp_queue)) != NULL) 
        {
            struct neighbour *n1 = neigh;
            write_unlock_bh(&neigh->lock);
            ...
            n1->output(skb);
            write_lock_bh(&neigh->lock);
        }
        skb_queue_purge(&neigh->arp_queue);
    }
    ...
    return err;
}
```

`neigh_update()` 函数主要完成以下工作：

*   更新邻居节点信息对象的 `ha` 字段（也就是 MAC 地址）为 ARP 回复中获得的邻居节点 MAC 地址。
*   如果邻居节点对象的 `arp_queue` 队列不为空，说明有等待发送的数据包，此时调用邻居节点信息的 `output()` 接口把这些数据发送出去。从上下文可知，此时的 `output()` 还是为 `neigh_resolve_output()` 函数。但由于邻居节点的 MAC 地址已经获得，所以不会再发送 ARP 请求。而是调用设备的 `hard_header()` 接口设置数据包的目标 MAC 地址，然后 调用 `dev_queue_xmit()` 函数把数据包发送出去。

