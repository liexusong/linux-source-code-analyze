# LVS原理与实现

`LVS`，全称 Linux Virtual Server，是章文嵩博士发起的一个开源项目。在社区具有很大的热度，是一个基于四层、性能极高的反向代理服务器。至于什么是反向代理，这里就不作详细介绍了，如果不了解可以先去阅读反向代理相关的资料。

## LVS工作原理

下面先介绍一下 LVS 的工作原理。

LVS的工作模式分为三种：`NAT模式（NAT）`、`直接路由模式（DR）` 和 `IP隧道模式（TUN）`。

下面将详细介绍每种工作模式的运行原理。

### 1. NAT模式

NAT模式的运行方式如下图：

![NAT](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/nat-arch.jpg)

__整个请求过程示意：__

*   `client` 发送请求到 `LVS` 的 `VIP` 上，LVS 首先根据 client 的 IP 和端口从连接信息表中查询是否已经存在，如果存在就直接使用当前连接进行处理。否则根据负载算法选择一个 `Real-Server`（真正提供服务的服务器），并记录连接到连接信息表中，然后把 client 请求的目的 IP 地址修改为 Real-Server 的地址，将请求发给 Real-Server。

*   Real-Server 收到请求包后，发现目的 IP 是自己的 IP，于是处理请求，然后发送回复给 LVS。

*   LVS 收到回复包后，修改回复包的的源地址为VIP，发送给 client。

*   当 client 发送完毕，此次连接结束或者连接超时，那么 LVS 自动将连接从连接信息表中删除此条记录。

上图中的蓝色连接线表示请求的数据流向，而红色连接线表示回复的数据流向。由于进出流量都需要经过`LVS` 服务器，所以 `LVS` 服务器可能会成功瓶颈。



