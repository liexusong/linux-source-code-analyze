# TUN/TAP设备原理与实现

`TUN/TAP设备` 是Linux下的一种虚拟设备，相对虚拟设备来说，一般计算机所使用的网卡就是物理设备。物理设备有发送和接收网络数据包的能力，而虚拟设备的作用就是模拟物理设备，提供特殊的发送和接收功能。

由于 `TUN/TAP设备` 是虚拟的设备，所以不能像网卡一样从网络中接收数据包（因为虚拟设备没有接口），那么 `TUN/TAP设备` 从哪里读取数据，又把数据发送到哪里去呢？下面通过一张图来阐述 `TUN/TAP设备` 的工作方式 (图片来源于https://www.ibm.com/developerworks/cn/linux/l-tuntap/index.html )：

![TUN/TAP](https://www.ibm.com/developerworks/cn/linux/l-tuntap/images/image002.jpg)

上图的右下角是物理网卡，它能从 `物理链路` 读取和发送数据。

而左下角是 `TUN/TAP设备`，可以看出 `TUN/TAP设备` 连接着 `字符驱动` 和 `TCP/IP协议栈`，这就是说可以通过 `read()` 和 `write()` 系统调用来对 `TUN/TAP设备` 进行数据的读写操作，而读写的数据会经过 `TCP/IP协议栈`。

其实 `TUN/TAP设备` 并没有对外网发送数据这个功能，如果需要对外网发送数据，必须通过路由转发才能实现。也就是通过构建发送到外网的数据包，然后通过路由转发来发送出去。

## TUN/TAP设备使用

下面例子主要介绍怎么创建和启动一个 TUN/TAP 设备（摘自openvpn开源项目 [http://openvpn.sourceforge.net](http://openvpn.sourceforge.net/)，tun.c文件）：

```c
int open_tun(const char *dev, char *actual, int size)
{
    struct ifreq ifr;
    int fd;
    char *device = "/dev/net/tun";

    if ((fd = open(device, O_RDWR)) < 0) //创建描述符
        msg(M_ERR, "Cannot open TUN/TAP dev %s", device);

    memset(&ifr, 0, sizeof (ifr));

    ifr.ifr_flags = IFF_NO_PI;
    if (!strncmp(dev, "tun", 3)) {  
        ifr.ifr_flags |= IFF_TUN;
    } else if (!strncmp(dev, "tap", 3)) {
        ifr.ifr_flags |= IFF_TAP;
    } else {
        msg(M_FATAL, "I don't recognize device %s as a TUN or TAP device", dev);
    }

    if (strlen(dev) > 3)
        strncpy(ifr.ifr_name, dev, IFNAMSIZ);

    if (ioctl(fd, TUNSETIFF, (void *) &ifr) < 0) //打开虚拟网卡
        msg(M_ERR, "Cannot ioctl TUNSETIFF %s", dev);

    set_nonblock(fd);

    msg(M_INFO, "TUN/TAP device %s opened", ifr.ifr_name);

    strncpynt(actual, ifr.ifr_name, size);

    return fd;
}
```

