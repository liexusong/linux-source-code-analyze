# Linux网桥工作原理

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

    if ((br = new_nb(name)) == NULL) // 创建一个网桥对象
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
