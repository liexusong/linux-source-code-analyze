# Linux网桥工作原理

Linux 的 `网桥` 是一种虚拟设备，可以将 `网桥` 看成网络设备中的的 `交换机`，`交换机` 的作用就是将多台主机连接起来组织成 `局域网`，如下图所示：

![switch-device](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/net-bridge/switch.png)

而 `网桥` 是 Linux 内核内部虚拟的设备，所以其只能连接本机的设备，而不能像真实 `交换机` 一样连接真实的主机，如下图所示：

![bridge](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/net-bridge/bridge.jpg)

