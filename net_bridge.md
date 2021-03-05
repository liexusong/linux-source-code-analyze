# Linux网桥工作原理

Linux 的 `网桥` 是一种虚拟设备（使用软件实现），可以将 Linux 内部多个网络接口连接起来，如下图所示：

![bridge](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/net-bridge/bridge.jpg)

而将网络接口连接起来的结果就是，一个网络接口接收到网络数据包后，会复制到其他网络接口中，如下图所示：

![bridge-packet](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/net-bridge/bridge-packet.jpg)

如上图所示，当网络接口A接收到数据包后，`网桥` 会将数据包复制并且发送给连接到 `网桥` 的其他网络接口（如上图中的网卡B和网卡C）。

Docker 就是使用 `网桥` 来进行容器间通讯的，我们来看看 Docker 是怎么利用 `网桥` 来进行容器间通讯的，原理如下图：

![docker-bridge](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/net-bridge/docker-bridge.png)

Docker 在启动时，会创建一个名为 `docker0` 的 `网桥`，并且把其 IP 地址设置为 `172.17.0.1/16`。然后使用对虚拟设备对 `veth-pair` 来将容器与 `网桥` 连接起来。
