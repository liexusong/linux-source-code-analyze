# TUN/TAP设备原理与实现

`TUN/TAP设备` 是Linux下的一种虚拟设备，相对虚拟设备来说，一般计算机所使用的网卡就是物理设备。物理设备有发送和接收网络数据包的能力，而虚拟设备的作用就是模拟物理设备，提供特殊的发送和接收功能。

由于 `TUN/TAP设备` 是虚拟的设备，所以不能像网卡一样从网络中接收数据包（因为虚拟设备没有接口），那么 `TUN/TAP设备` 从哪里读取数据，又把数据发送到哪里去呢？下面通过一张图来阐述 `TUN/TAP设备` 的工作方式 (图片来源于https://www.ibm.com/developerworks/cn/linux/l-tuntap/index.html )：

![TUN/TAP](https://www.ibm.com/developerworks/cn/linux/l-tuntap/images/image002.jpg)

上图的右下角是物理网卡，它能从 `物理链路` 读取和发送数据。

而左下角是 `TUN/TAP设备`，可以看出 `TUN/TAP设备` 连接着 `字符驱动` 和 `TCP/IP协议栈`，这就是说可以通过 `read()` 和 `write()` 系统调用来对 `TUN/TAP设备` 进行数据的读写操作，而读写的数据会经过 `TCP/IP协议栈`。

其实 `TUN/TAP设备` 并没有对外网发送数据这个功能，如果需要对外网发送数据，必须通过路由转发才能实现。也就是通过构建发送到外网的数据包，然后通过路由转发来发送出去。

## TUN/TAP设备使用

在使用 TUN/TAP 设备前，先确保已经装载 TUN/TAP 模块并建立设备文件：

```shell
root@vagrant]# modprobe tun
root@vagrant]# mknod /dev/net/tun c 10 200
```

参数 `c` 表示是字符设备, 10 和 200 分别是主设备号和次设备号。

这样，我们就可以在程序中使用该驱动了。

下面例子主要介绍怎么创建和启动一个 TUN/TAP 设备（摘自openvpn开源项目 [http://openvpn.sourceforge.net](http://openvpn.sourceforge.net/)，tun.c文件）：

```c
int open_tun(const char *dev, char *actual, int size)
{
    struct ifreq ifr;
    int fd;
    char *device = "/dev/net/tun";

    if ((fd = open(device, O_RDWR)) < 0) // 创建描述符
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

    if (ioctl(fd, TUNSETIFF, (void *) &ifr) < 0) // 打开虚拟网卡设备
        msg(M_ERR, "Cannot ioctl TUNSETIFF %s", dev);

    set_nonblock(fd);

    msg(M_INFO, "TUN/TAP device %s opened", ifr.ifr_name);

    strncpynt(actual, ifr.ifr_name, size);

    return fd;
}
```

`open_tun()` 函数会创建并启动一个 TUN/TAP 设备文件句柄，然后就可以通过对这个文件句柄进行 `read()` 和 `write()` 系统调用来读写此 TUN/TAP 设备。

## TUN/TAP 设备实现

上面主要介绍了 TUN/TAP 设备的原理与使用方式，接下来主要分析 TUN/TAP 设备的实现过程。

### 1. 打开 TUN/TAP 设备

当调用 `open()` 系统调用打开一个 TUN/TAP 设备时，会触发调用内核态的 `tun_chr_open()` 函数，其实现如下：

```c
static int tun_chr_open(struct inode *inode, struct file *file)
{
    struct tun_struct *tun = NULL;

    // 申请一个 TUN 对象结构
    tun = kmalloc(sizeof(struct tun_struct), GFP_KERNEL);
    if (tun == NULL)
        return -ENOMEM;

    memset(tun, 0, sizeof(struct tun_struct));
    file->private_data = tun; // 将 TUN 对象结构与文件描述符绑定

    skb_queue_head_init(&tun->txq);       // 初始化txq队列
    init_waitqueue_head(&tun->read_wait); // 初始化等待队列

    sprintf(tun->name, "tunX"); // 设置设备的名称

    tun->dev.init = tun_net_init; // 设置启动TUN设备的初始化函数
    tun->dev.priv = tun;

    return 0;
}
```

`tun_chr_open()` 函数主要完成以下几个步骤：

*   调用 `kmalloc()` 函数创建一个 `tun` 对象结构。
*   将 `TUN` 对象结构与文件描述符绑定。
*   初始化 `TUN` 对象的 `txq` 队列和等待队列。
*   设置 `TUN` 设备的名称。
*   设置启动 `TUN` 设备的初始化函数。

我们先来看看 `TUN` 对象结构的定义：

```c
struct tun_struct {
    char                    name[8];      // TUN设备的名字
    unsigned long           flags;        // 设备类型: TUN或者TAP
    struct fasync_struct    *fasync;
    wait_queue_head_t       read_wait;    // 等待此设备可读的进程队列
    struct net_device       dev;          // TUN设备关联的网络设备对象
    struct sk_buff_head     txq;          // 数据队列(接收到的数据包会保存到这里)
    ...
};
```

