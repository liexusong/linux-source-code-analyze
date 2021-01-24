# LVS原理与实现 - 原理篇

`LVS`，全称 Linux Virtual Server，是章文嵩博士发起的一个开源项目。在社区具有很大的热度，是一个基于四层、性能极高的反向代理服务器。至于什么是反向代理，这里就不作详细介绍了，如果不了解可以先去阅读反向代理相关的资料。

## LVS工作原理

下面先介绍一下 LVS 的工作原理。

LVS的工作模式分为三种：`NAT模式（网络地址转换）`、`DR模式（直接路由）` 和 `TUN模式（IP隧道）`。

下面将详细介绍每种工作模式的运行原理。

>   名词解析：
>
>   1.  Director服务器：直接接收用户请求的服务器，是LVS的入口。
>   2.  Real-Server服务器：真实服务器，用于处理用户请求的服务器。
>   3.  虚拟IP（VIP）：对外网暴露的 IP 地址，客户端可以通过 VIP 访问 LVS 集群。

### 1. NAT模式

`NAT模式` 的运行方式如下图：

![NAT-ARCH](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/nat-arch.jpg)

__请求过程说明：__

*   `client` 发送请求到 `LVS` 的 `VIP` 上，`Director` 服务器首先根据 `client` 的 IP 和端口从连接信息表中查询是否已经存在，如果存在就直接使用当前连接进行处理。否则根据负载算法选择一个 `Real-Server`（真正提供服务的服务器），并记录连接到连接信息表中，然后把 `client` 请求的目的 IP 地址修改为 `Real-Server` 的地址，将请求发给 `Real-Server`。

*   `Real-Server` 服务器收到请求包后，发现目的 IP 是自己的 IP，于是处理请求，然后发送回复给 `Director` 服务器。

*   `Director` 服务器收到回复包后，修改回复包的的源地址为VIP，发送给 `client`。

> 上图中的蓝色连接线表示请求的数据流向，而红色连接线表示回复的数据流向。由于进出流量都需要经过 `Director` 服务器，所以 `Director` 服务器可能会成功瓶颈。

下面通过一幅图来说明一个请求数据包在 LVS 服务器中的地址变化情况：

![NAT-PACKAGE](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/nat-package.jpg)

下面解释一下请求数据包的地址变化过程：

*   client 向 LVS 集群发起请求，源IP地址和源端口为：`192.168.11.100:11021`，而目标IP地址和端口为：`192.168.10.10:80`。当 `Director` 服务器接收到 client 的请求后，会根据调度算法选择一台合适的 `Real-Server` 服务器，并且把请求数据包的目标IP地址和端口改为 `Real-Server` 服务器的IP地址和端口，并记录连接信息到连接信息表中，如上图选择的 `Real-Server` 服务器的IP地址和端口为：`192.168.1.2:80`。

*   当 `Real-Server` 服务器接收到请求后，对请求进行处理，处理完后会把数据包的源IP地址和端口跟目标IP地址和端口交互，然后发送给网关 `192.168.1.1`（也就是 `Director` 服务器）。

*   `Director` 服务器接收到来自 `Real-Server` 服务器的回复数据，然后根据连接信息把源IP地址更改为虚拟IP地址。

> 由于 `Real-Server` 服务器需要把 `Director` 服务器设置为网关，所以 `Director` 服务器与 `Real-Server` 服务器需要部署在同一个网络下。

### 2. DR模式

`DR模式` 的运行方式如下图：

![DR-ARCH](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/dr-arch.jpg)

__请求过程说明：__

*   `client` 发送请求到 `LVS` 的 `VIP` 上，`Director` 服务器首先根据 client 的 IP 和端口从连接信息表中查询是否已经存在，如果存在就直接使用当前连接进行处理。否则根据负载算法选择一个 `Real-Server`（真正提供服务的服务器），并记录连接到连接信息表中，然后通过修改请求数据包的目标 MAC 地址为 `Real-Server` 服务器的 MAC 地址（注意：IP地址不修改），并通过局域网把数据包发送出去。

*   由于 `Director` 服务器与 `Real-Server` 服务器在同一局域网中，所以通过数据包的目标 MAC 地址可以找到对应的 `Real-Server` 服务器（以太网协议），而 `Real-Server` 服务器接收到数据包后，会对数据包进行处理。

*   `Real-Server` 服务器处理完请求后，把处理结果直接发送给 `client`，而不会通过 `Director` 服务器。

> 注意：`Real-Server` 服务器必须设置回环设备的 IP 地址为 VIP 地址，因为如果不设置 VIP，那么 `Real-Server` 服务器会认为这个数据包发送给本机的，从而丢弃这个数据包。

下面通过一幅图来说明一个请求数据包在 LVS 服务器中的地址变化情况：

![DR-PACKAGE](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/dr-package.jpg)

下面解释一下请求数据包的地址变化过程：

*   client 向 LVS 集群发起请求，源IP地址和源端口为：`192.168.11.100:11021`，而目标IP地址和端口为：`192.168.10.10:80`。当 `Director` 服务器接收到 client 的请求后，会根据调度算法选择一台合适的 `Real-Server` 服务器，并且把请求数据包的目标 MAC 地址改为 `Real-Server` 服务器的 MAC 地址，并记录连接信息到连接信息表中，然后通过局域网把数据包发送出去。

*   当 `Real-Server` 服务器处理完请求后，会把处理结果直接发送给 `client`。如果 `Real-Server` 服务器与 `client` 在同一个局域网，那么直接通过把目标 MAC 地址修改为 `client` 的 MAC 地址。否则，通过把目标 MAC 地址修改为路由器的 MAC 地址，然后通过路由器发送出去。

> 从上图可以看出，整个请求过程中，数据包只有 MAC 地址发生变化。
> 另外，由于 `DR模式` 只有入口需要经过 `Director` 服务器，而出口不需要经过 `Director` 服务器，所以性能比 `NAT模式` 要高。

### 3. TUN模式

`TUN模式` 比较复杂一些，并且国内使用得比较少，所以这里就不作介绍，有兴趣自己查阅相关资料。

## 调度算法

上面介绍了 LVS 的工作模式，下面介绍一下 LVS 的调度算法。

由于 LVS 需要选择合适的 `Real-Server（RS）` 服务器处理请求，所以需要根据不同的需求选择不同的调度算法来选择 `Real-Server` 服务器。LVS 的调度算法主要有以下几种：

* __1. 轮询调度（Round-Robin,RR）__

最简单的调度算法，按照顺序将请求依次转发给后端的RS。大部分情况下，RS的性能状态都是各不一致的，这种算法显然无法满足合理利用资源的要求。

* __2. 带权重的轮询调度（Weighted Round-Robin,WRR）__

在轮询算法的基础上加上权重设置，权重越高的RS被分配到的请求越多。适用于按照服务器性能高低，配置不同的权重，以达到合理的资源利用。

* __3. 最小连接调度（Least-Connection, LC）__

把新的请求分配给连接数最少的RS。连接数少说明服务器空闲。

* __4. 带权重的最小连接调度（Weight Least-Connection, WLC）__

在最小连接算法的基础上加上权重设置，这样可以人为地控制请求分配。

* __5. 基于局部性的最小连接调度（Locality-Based Least Connection, LBLC）__

针对请求报文目标IP地址的负载均衡调度。目前主要用于Cache集群系统，因为在Cache集群中客户请求报文的目标IP地址是变化的。

算法的设计目标是在服务器的负载基本平衡情况下，将相同目标IP地址的请求调度到同一台服务器，来提高各台服务器的访问局部性和主存Cache命中率，提升整个集群系统的处理能力。

LBLC调度算法先根据请求的目标IP地址找出该目标IP地址最近使用的服务器，若该服务器是可用的且没有超载，将请求发送到该服务器；若服务器不存在，或者该服务器超载且有服务器处于其一半的工作负载，则用“最小连接”的原则选出一个可用的服务器，将请求发送到该服务器。

* __6. 带复制的基于局部性最小连接调度（Locality-Based Least Connections with Replication, LBLCR）__

也是针对请求报文目标IP地址的负载均衡调度，与LBLC算法不同之处：LBLC维护一个目标IP到一台服务器的映射，而LBLCR则需要维护一个目标IP到一组服务器的映射。

LBLCR调度算法先根据请求的目标IP地址找到对应的服务器组，按“最小连接”原则从该服务器组中选出一台服务器，若服务器没有超载，则将请求发送到该服务器；若服务器超载，则按“最小连接”原则从整个集群中选出一台服务器，将该服务器加入到服务组中，将请求发送给这台服务器。同时，当该服务器组有一段时间没有被修改，将最忙的服务器从服务器组中删除，以降低复制的程度。

* __7. 目标地址散列调度（Destination Hashing, DH）__

也是针对请求报文目标IP地址的负载均衡调度，但它是一种静态映射算法，通过一个散列（Hash）函数将一个目标IP地址映射到一台服务器。DH算法先根据请求的目标IP地址，作为散列键（Hash Key）从静态分配的散列表找出对应的服务器，若该服务器是可用的且为超载，将请求发送到该服务器，否则返回空。

* __8. 源地址散列调度（Source Hashing, SH）__

该算法正好与DH调度算法相反，它根据请求的源IP地址，作为散列键从静态分配的散列表找出对应的服务器，若该服务器是可用的且未超载，将请求发送到该服务器，否则返回空。算法流程与目标地址散列调度算法基本相似，只不过将请求的目标IP地址换成请求的源IP地址。

## 总结

本文主要简单的介绍了 LVS 的运行原理与调度算法，更多相关的资料可以查阅参考链接，而 LVS 的实现部分将会在另外一篇文章介绍。

参考链接：

[http://www.linuxvirtualserver.org/Documents.html](http://www.linuxvirtualserver.org/Documents.html)

[http://www.linuxvirtualserver.org/VS-NAT.html](http://www.linuxvirtualserver.org/VS-NAT.html)

[http://www.linuxvirtualserver.org/VS-DRouting.html](http://www.linuxvirtualserver.org/VS-DRouting.html)

[http://www.linuxvirtualserver.org/VS-IPTunneling.html](http://www.linuxvirtualserver.org/VS-IPTunneling.html)

[https://blog.51cto.com/blief/1745134](https://blog.51cto.com/blief/1745134)
