# LVS原理与实现 - 实现篇

在上一篇文章中，我们主要介绍了 `LVS` 的原理，接下来我们将会介绍 `LVS` 的代码实现。

>   本文使用的内核版本是：2.4.23，而 LVS 的代码在路径：`/src/net/ipv4/ipvs` 中。

## Netfilter

在介绍 `LVS` 的实现前，我们需要了解以下 `Netfilter` 这个功能。

`Netfilter` 顾名思义就是网络过滤器，是 Linux 系统独有的，用于处理进出内核协议栈的网络数据包，其中 `iptables` 就是基于 `Netfilter` 实现的。

