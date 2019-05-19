## Unix Domain Sockets使用
上一章介绍了Socket接口层的实现，接下来我们将会介绍具体的协议层实现，这一章将会介绍用于进程间通信的 `Unix Doamin Sockets` 的实现。要使用 `Unix Domain Sockets` 需要在创建socket时为 `family` 参数传入 `AF_UNIX`，如下代码：
```cpp
fd = socket(AF_UNIX, SOCK_STREAM, 0);
```
这样就可以创建一个类型为 `Unix Domain Sockets` 的socket描述符，如果我们编写的是服务端程序，那就需要在调用 `bind()` 函数时为其指定一个唯一的文件路径，客户端就可以通过这个文件路径来连接服务端程序。唯一路径是通过结构体 `struct sockaddr_un` 来设置的，如下代码：

服务端：
```cpp
...
int fd;
struct sockaddr_un addr;

fd = socket(AF_UNIX, SOCK_STREAM, 0);

memset(&addr, 0, sizeof(addr));

addr.sun_family = AF_UNIX;
strcpy(addr.sun_path, "/tmp/server.sock");

bind(fd, (struct sockaddr *)&addr, sizeof(addr));
...
```
客户端：
```cpp
...
int fd;
struct sockaddr_un addr;

fd = socket(AF_UNIX, SOCK_STREAM, 0);

memset(&addr, 0, sizeof(addr));

addr.sun_family = AF_UNIX;
strcpy(addr.sun_path, "/tmp/server.sock");

connect(fd, (struct sockaddr *)&addr, sizeof(addr));
...
```

## Unix Domain Sockets实现
上一章介绍过，在应用程序中调用 `socket()` 函数时候，最终会调用 `sys_socket()` 函数，`sys_socket()` 接着通过调用 `sock_create()` 函数创建socket对象，我们来看看 `sock_create()` 函数的实现：
```cpp
int sock_create(int family, int type, int protocol, struct socket **res)
{
    int i;
    struct socket *sock;

    ...

    if (!(sock = sock_alloc())) {
        printk(KERN_WARNING "socket: no more sockets\n");
        i = -ENFILE;
        goto out;
    }

    sock->type  = type;

    if ((i = net_families[family]->create(sock, protocol)) < 0) {
        sock_release(sock);
        goto out;
    }

    *res = sock;

out:
    net_family_read_unlock();
    return i;
}
```
因为创建 `Unix Domain Sockets` 时需要为 `family` 参数传入 `AF_UNIX`，所以代码 `net_families[family]->create()` 就是调用 `AF_UNIX` 类型的 `create()` 函数。上一章也介绍过，需要通过调用 `sock_register()` 函数向 `net_families` 注册具体协议的创建函数，对于 `Unix Domain Sockets` 在系统初始化时会在 `af_unix_init()` 函数中注册其创建函数，代码如下：
```cpp
struct net_proto_family unix_family_ops = {
    PF_UNIX,
    unix_create
};

static int __init af_unix_init(void)
{
    ...
    sock_register(&unix_family_ops);
    ...
    return 0;
}
```
所以对于代码 `net_families[AF_UNIX]->create()` 实际调用的就是 `unix_create()` 函数，`unix_create()` 函数实现如下：
```cpp
static int unix_create(struct socket *sock, int protocol)
{
    ...
    sock->state = SS_UNCONNECTED;

    switch (sock->type) {
    case SOCK_STREAM:
        sock->ops = &unix_stream_ops;
        break;
    case SOCK_RAW:
        sock->type=SOCK_DGRAM;
    case SOCK_DGRAM:
        sock->ops = &unix_dgram_ops;
        break;
    default:
        return -ESOCKTNOSUPPORT;
    }

    return unix_create1(sock) ? 0 : -ENOMEM;
}
```
从 `unix_create()` 函数的实现可知，当向参数 `protocol` 传入 `SOCK_STREAM` 时，就把 sock 的 `ops` 字段设置为 `unix_stream_ops`，如果传入的是 `SOCK_RAW`，那么就把 sock 的 `ops` 字段设置为 `unix_dgram_ops`。这两个结构的定义如下：
```cpp
struct proto_ops unix_stream_ops = {
    family:     PF_UNIX,
    
    release:    unix_release,
    bind:       unix_bind,
    connect:    unix_stream_connect,
    socketpair: unix_socketpair,
    accept:     unix_accept,
    getname:    unix_getname,
    poll:       unix_poll,
    ioctl:      unix_ioctl,
    listen:     unix_listen,
    shutdown:   unix_shutdown,
    setsockopt: sock_no_setsockopt,
    getsockopt: sock_no_getsockopt,
    sendmsg:    unix_stream_sendmsg,
    recvmsg:    unix_stream_recvmsg,
    mmap:       sock_no_mmap,
};

struct proto_ops unix_dgram_ops = {
    family:     PF_UNIX,
    
    release:    unix_release,
    bind:       unix_bind,
    connect:    unix_dgram_connect,
    socketpair: unix_socketpair,
    accept:     sock_no_accept,
    getname:    unix_getname,
    poll:       datagram_poll,
    ioctl:      unix_ioctl,
    listen:     sock_no_listen,
    shutdown:   unix_shutdown,
    setsockopt: sock_no_setsockopt,
    getsockopt: sock_no_getsockopt,
    sendmsg:    unix_dgram_sendmsg,
    recvmsg:    unix_dgram_recvmsg,
    mmap:       sock_no_mmap,
};
```
