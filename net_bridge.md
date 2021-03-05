# Linux网桥工作原理与实现

Linux 的 `网桥` 是一种虚拟设备（使用软件实现），可以将 Linux 内部多个网络接口连接起来，如下图所示：

![bridge](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/net-bridge/bridge.jpg)

而将网络接口连接起来的结果就是，一个网络接口接收到网络数据包后，会复制到其他网络接口中，如下图所示：

![bridge-packet](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/net-bridge/bridge-packet.jpg)

如上图所示，当网络接口A接收到数据包后，`网桥` 会将数据包复制并且发送给连接到 `网桥` 的其他网络接口（如上图中的网卡B和网卡C）。

Docker 就是使用 `网桥` 来进行容器间通讯的，我们来看看 Docker 是怎么利用 `网桥` 来进行容器间通讯的，原理如下图：

![docker-bridge](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/net-bridge/docker-bridge.png)

Docker 在启动时，会创建一个名为 `docker0` 的 `网桥`，并且把其 IP 地址设置为 `172.17.0.1/16`（私有 IP 地址）。然后使用虚拟设备对 `veth-pair` 来将容器与 `网桥` 连接起来，如上图所示。而对于 `172.17.0.0/16` 网段的数据包，Docker 会定义一条 `iptables NAT` 的规则来将这些数据包的 IP 地址转换成公网 IP 地址，然后通过真实网络接口（如上图的 `ens160` 接口）发送出去。

接下来，我们主要通过代码来分析 `网桥` 的实现。

## 网桥的实现

### 1. 网桥的创建

我们可以通过下面命令来添加一个名为 `br0` 的 `网桥` 设备对象：

```shell
[root@vagrant]# brctl addbr br0
```

然后，我们可以通过命令 `brctl show` 来查看系统中所有的 `网桥` 设备列表，如下：

```shell
[root@vagrant]# brctl show
bridge name     bridge id               STP enabled     interfaces
br0             8000.000000000000       no
docker0         8000.000000000000       no
```

当使用命令创建一个新的 `网桥` 设备时，会触发内核调用 `br_add_bridge()` 函数，其实现如下：
```c
int br_add_bridge(char *name)
{
    struct net_bridge *br;

    if ((br = new_nb(name)) == NULL) // 创建一个网桥设备对象
        return -ENOMEM;

    if (__dev_get_by_name(name) != NULL) { // 设备名是否已经注册过?
        kfree(br);
        return -EEXIST; // 返回错误, 不能重复注册相同名字的设备
    }

    // 添加到网桥列表中
    br->next = bridge_list;
    bridge_list = br;
    ...
    register_netdev(&br->dev); // 把网桥注册到网络设备中

    return 0;
}
```

`br_add_bridge()` 函数主要完成以下几个工作：
* 调用 `new_nb()` 函数创建一个 `网桥` 设备对象。
* 调用 `__dev_get_by_name()` 函数检查设备名是否已经被注册过，如果注册过返回错误信息。
* 将 `网桥` 设备对象添加到 `bridge_list` 链表中，内核使用 `bridge_list` 链表来保存所有 `网桥` 设备。
* 调用 `register_netdev()` 将网桥设备注册到网络设备中。

从上面的代码可知，`网桥` 设备使用了 `net_bridge` 结构来描述，其定义如下：
```c
struct net_bridge
{
    struct net_bridge           *next;               // 连接内核中所有的网桥对象
    rwlock_t                    lock;                // 锁
    struct net_bridge_port      *port_list;          // 网桥端口列表
    struct net_device           dev;                 // 网桥设备信息
    struct net_device_stats     statistics;          // 信息统计
    rwlock_t                    hash_lock;           // 用于锁定CAM表
    struct net_bridge_fdb_entry *hash[BR_HASH_SIZE]; // CAM表
    struct timer_list           tick;

    /* STP */
    ...
};
```

在 `net_bridge` 结构中，比较重要的字段为 `port_list` 和 `hash`：

* `port_list`：网桥端口列表，保存着绑定到 `网桥` 的网络接口列表。
* `hash`：保存着以网络接口 `MAC地址` 为键值，以网桥端口为值的哈希表。

`网桥端口` 使用结构体 `net_bridge_port` 来描述，其定义如下：
```c
struct net_bridge_port
{
    struct net_bridge_port  *next;   // 指向下一个端口
    struct net_bridge       *br;     // 所属网桥设备对象
    struct net_device       *dev;    // 网络接口设备对象
    int                     port_no; // 端口号

    /* STP */
    ...
};
```

而 `net_bridge_fdb_entry` 结构用于描述网络接口设备 `MAC地址` 与 `网桥端口` 的对应关系，其定义如下：
```c
struct net_bridge_fdb_entry
{
    struct net_bridge_fdb_entry *next_hash;
    struct net_bridge_fdb_entry **pprev_hash;
    atomic_t                    use_count;
    mac_addr                    addr;  // 网络接口设备MAC地址
    struct net_bridge_port      *dst;  // 网桥端口
    ...
};
```

这三个结构的对应关系如下图所示：

![net-bridge](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/net-bridge/net-bridge.png)

可见，要将 `网络接口设备` 绑定到一个 `网桥` 上，需要使用 `net_bridge_port` 结构来关联的，下面我们来分析怎么将一个 `网络接口设备` 绑定到一个 `网桥` 中。

> 网桥是工作在 TCP/IP 协议栈的第二层，也就是说，网桥能够根据目标 MAC 地址对数据包进行广播或者单播。当目标 MAC 地址能够从网桥的 hash 表中找到对应的网桥端口，说明此数据包是单播的数据包，否则就是广播的数据包。

### 2. 将网络接口绑定到网桥

要将一个 `网络接口设备` 绑定到一个 `网桥` 上，可以使用以下命令：

```shell
[root@vagrant]# brctl addif br0 eth0
```

上面的命令让网络接口 `eth0` 绑定到网桥 `br0` 上。

当调用命令将网络接口设备绑定到网桥上时，内核会触发调用 `br_add_if()` 函数来实现，其代码如下：

```c
int br_add_if(struct net_bridge *br, struct net_device *dev)
{
    struct net_bridge_port *p;
    ...
    write_lock_bh(&br->lock);

    // 创建一个新的网桥端口对象, 并添加到网桥的port_list链表中
    if ((p = new_nbp(br, dev)) == NULL) { 
        write_unlock_bh(&br->lock);
        dev_put(dev);
        return -EXFULL;
    }

    // 设置网络接口设备为混杂模式
    dev_set_promiscuity(dev, 1);
    ...
    // 添加到网络接口MAC地址与网桥端口对应的哈希表中
    br_fdb_insert(br, p, dev->dev_addr, 1);
    ...
    write_unlock_bh(&br->lock);

    return 0;
}
```

`br_add_if()` 函数主要完成以下工作：
* 调用 `new_nbp()` 函数创建一个新的 `网桥端口` 并且添加到 `网桥` 的 `port_list` 链表中。
* 将网络接口设备设置为 `混杂模式`。
* 调用 `br_fdb_insert()` 函数将新建的 `网桥端口` 插入到网络接口 `MAC地址` 对应的哈希表中。

也就是说，`br_add_if()` 函数主要建立 `网络接口设备` 与 `网桥` 的关系。

### 3. 网桥中的网络接口接收数据

当某个 `网络接口` 接收到数据包时，会判断这个 `网络接口` 是否绑定到某个 `网桥` 上，如果绑定了，那么就调用 `handle_bridge()` 函数处理这个数据包。`handle_bridge()` 函数实现如下：

```c
static int __inline__
handle_bridge(struct sk_buff *skb, struct packet_type *pt_prev)
{
    int ret = NET_RX_DROP;
    ...
    br_handle_frame_hook(skb);
    return ret;
}
```

`br_handle_frame_hook` 是一个函数指针，其指向 `br_handle_frame()` 函数，我们来分析 `br_handle_frame()` 函数的实现：

```c
void br_handle_frame(struct sk_buff *skb)
{
    struct net_bridge *br;

    br = skb->dev->br_port->br; // 获取设备连接的网桥对象

    read_lock(&br->lock);   // 对网桥上锁
    __br_handle_frame(skb); // 调用__br_handle_frame()函数处理数据包
    read_unlock(&br->lock);
}
```

`br_handle_frame()` 函数的实现比较简单，首先对 `网桥` 进行上锁操作，然后调用 `__br_handle_frame()` 处理数据包，我们来分析 `__br_handle_frame()` 函数的实现：

```c
static void __br_handle_frame(struct sk_buff *skb)
{
    struct net_bridge *br;
    unsigned char *dest;
    struct net_bridge_fdb_entry *dst;
    struct net_bridge_port *p;
    int passedup;

    dest = skb->mac.ethernet->h_dest; // 目标MAC地址
    p = skb->dev->br_port;            // 网络接口绑定的端口
    br = p->br;
    passedup = 0;
    ...
    // 将学习到的MAC地址插入到网桥的hash表中
    if (p->state == BR_STATE_LEARNING || p->state == BR_STATE_FORWARDING)
        br_fdb_insert(br, p, skb->mac.ethernet->h_source, 0);
    ...
    if (dest[0] & 1) {        // 如果是一个广播包
        br_flood(br, skb, 1); // 把数据包发送给连接到网桥上的所有网络接口
        if (!passedup)
            br_pass_frame_up(br, skb);
        else
            kfree_skb(skb);
        return;
    }

    dst = br_fdb_get(br, dest);    // 获取目标MAC地址对应的网桥端口
    ...
    if (dst != NULL) {             // 如果目标MAC地址对应的网桥端口存在
        br_forward(dst->dst, skb); // 那么只将数据包转发给此端口
        br_fdb_put(dst);
        return;
    }

    br_flood(br, skb, 0); // 否则发送给连接到此网桥上的所有网络接口
    return;
    ...
}
```

`__br_handle_frame()` 函数主要完成以下几个工作：

* 首先将从数据包中学习到的MAC地址插入到网桥的hash表中。
* 如果数据包是一个广播包（目标MAC地址的第一位为1），那么调用 `br_flood()` 函数把数据包发送给连接到网桥上的所有网络接口。
* 调用 `br_fdb_get()` 获取目标MAC地址对应的网桥端口，如果目标MAC地址对应的网桥端口存在，那么调用 `br_forward()` 函数把数据包转发给此端口。
* 否则调用 调用 `br_flood()` 函数把数据包发送给连接到网桥上的所有网络接口。

函数 `br_forward()` 用于把数据包发送给指定的网桥端口，其实现如下：
```c
static void __br_forward(struct net_bridge_port *to, struct sk_buff *skb)
{
    skb->dev = to->dev;
    dev_queue_xmit(skb);
}

void br_forward(struct net_bridge_port *to, struct sk_buff *skb)
{
    if (should_forward(to, skb)) { // 端口是否能够接收数据?
        __br_forward(to, skb);
        return;
    }
    kfree_skb(skb);
}
```

`br_forward()` 函数通过调用 `__br_forward()` 函数来发送数据给指定的网桥端口，`__br_forward()` 函数首先将数据包的输出接口设备设置为网桥端口绑定的设备，然后调用 `dev_queue_xmit()` 函数将数据包发送出去。

而 `br_flood()` 函数用于将数据包发送给绑定到 `网桥` 上的所有网络接口设备，其实现如下：

```c
void br_flood(struct net_bridge *br, struct sk_buff *skb, int clone)
{
    struct net_bridge_port *p;
    struct net_bridge_port *prev;
    ...
    prev = NULL;

    p = br->port_list;
    while (p != NULL) {              // 遍历绑定到网桥的所有网络接口设备
        if (should_forward(p, skb)) { // 端口是否能够接收数据包?
            if (prev != NULL) {
                struct sk_buff *skb2;

                // 克隆一个数据包
                if ((skb2 = skb_clone(skb, GFP_ATOMIC)) == NULL) { 
                    br->statistics.tx_dropped++;
                    kfree_skb(skb);
                    return;
                }

                __br_forward(prev, skb2); // 把数据包发送给设备
            }

            prev = p;
        }

        p = p->next;
    }

    if (prev != NULL) {
        __br_forward(prev, skb);
        return;
    }

    kfree_skb(skb);
}
```

`br_flood()` 函数的实现也比较简单，主要是遍历绑定到网桥的所有网络接口设备，然后调用 `__br_forward()` 函数将数据包转发给设备对应的端口。
