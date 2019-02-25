# Unix Domain Socket 实现原理
上一节介绍了 `Socket族接口层` 的实现，接下来我们将会介绍 `Unix Domain Socket` 的实现。

`Unix Domain Socket` 是一种进程间通信方式(IPC)，用于同一台主机的不同进程间通信，不需要基于网络协议，主要是基于文件系统。而且 `Unix Domain Socket` 是一种全双工的通信方式，就是可以通过一个 `socket` 句柄来进行读写操作。

## 创建 Unix Domain Socket
现在我们先来介绍一下在用户态怎么创建和使用 `Unix Domain Socket` 吧，先来介绍一下服务端的创建：
```cpp
#define UNIX_SOCKET_PATH "/path/to/unixsocket"

int main(int argc, char *argv[])
{
    ...
    struct sockaddr_un addr;
    ...
    int sock = socket(AF_UNIX, SOCK_STREAM, 0);
    strcpy (addr.sun_path, UNIX_SOCKET_PATH);
    ...
    bind(sock, (structsockaddr *)&addr, sizeof(struct sockaddr_un));   
    ...
    listen(sock, 100);
    ...
    while (1) {
        ...
        newsock = accept(sock, 0, 0);
        ...
        rval = read(newsock, buf, 1024));
        ...
    }
}
```
从上面的代码可知，要创建一个 `Unix Domain Socket`，首先在调用 `socket()` 函数时第一个参数传入 `AF_UNIX`，这样就可以创建一个 `Unix Domain Socket`。然后调用 `bind()` 函数时需要传入一个 `struct sockaddr_un` 的结构体，这个结构体用来指定这个 `socket` 绑定的名字，因为每个 `Unix Domain Socket` 都需要绑定一个唯一的名字，接着调用 `listen()` 函数开始监听请求，最后在主循环中调用 `accpet()` 函数来接收请求的连接。

接下来我们介绍 `Unix Domain Socket` 客户端的编写：
```cpp
#define UNIX_SOCKET_PATH "/path/to/unixsocket"

int main(int argc, char *argv[])
{
    ...
    sock = socket(AF_UNIX, SOCK_STREAM, 0);
    strcpy(addr.sun_path, UNIX_SOCKET_PATH);
    ...
    if (connect(sock, (struct sockaddr *)&addr, sizeof(struct sockaddr_un)) < 0) {
        close(sock);
        exit(1);
    }

    write(sock, DATA, sizeof(DATA));

    close(sock);
}
```
客户端请求时需要指定要连接的 `Unix Domain Socket` 绑定的名字，然后调用 `connect()` 函数连接到服务端，并且开始进行通信。

## Unix Domain Socket 实现
前面介绍过，当用户程序调用 `socket()` 函数创建一个 `socket` 时，会触发调用内核函数 `sys_socketcall()`，而 `sys_socketcall()` 函数根据 `call` 参数的值来选择调用哪个内核函数，由于调用 `socket()` 函数时传入的是 `SOCKOP_socket` ( `SYS_SOCKET` )，所以 `sys_socketcall()` 最终会调用 `sys_socket()` 内核函数，`sys_socket()` 内核函数的实现如下：
```cpp
asmlinkage long sys_socket(int family, int type, int protocol)
{
    int retval;
    struct socket *sock;

    retval = sock_create(family, type, protocol, &sock);
    if (retval < 0)
        goto out;

    retval = sock_map_fd(sock);
    if (retval < 0)
        goto out_release;

out:
    return retval;

out_release:
    sock_release(sock);
    return retval;
}
```
`sys_socket()` 函数首先会调用 `sock_create()` 函数创建一个类型为 `struct socket` 结构的 sock，然后调用 `sock_map_fd()` 函数把 sock 映射到一个文件句柄上。创建 `Unix Domain Socket` 时传入的 `family` 值为 `AF_UNIX`，而 `type` 的值为 `SOCK_STREAM`。

下面接着来分析 `sock_create()` 函数的实现：
```cpp
static struct net_proto_family *net_families[NPROTO];

int sock_create(int family, int type, int protocol, struct socket **res)
{
    int i;
    struct socket *sock;
    ...
    net_family_read_lock();
    ...
    if (!(sock = sock_alloc())) {
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
`sock_create()` 函数首先调用 `sock_alloc()` 申请一个类型为 `struct socket` 结构的对象 `sock`，然后根据 `family` 的值从 `net_families` 数组中找到对应的 `net_proto_family` 结构，然后通过此 `net_proto_family` 结构的 `create` 函数来初始化 `sock`。

我们先来看看 `struct socket` 结构的定义：
```cpp
struct socket
{
    socket_state            state;

    unsigned long           flags;
    struct proto_ops        *ops;
    struct inode            *inode;
    struct fasync_struct    *fasync_list;
    struct file             *file;
    struct sock             *sk;
    wait_queue_head_t       wait;

    short                   type;
    unsigned char           passcred;
};

struct proto_ops {
  int   family;

  int   (*release)(struct socket *sock);
  int   (*bind)(struct socket *sock, struct sockaddr *umyaddr,
             int sockaddr_len);
  int   (*connect)(struct socket *sock, struct sockaddr *uservaddr,
             int sockaddr_len, int flags);
  int   (*socketpair)(struct socket *sock1, struct socket *sock2);
  int   (*accept)(struct socket *sock, struct socket *newsock,
             int flags);
  int   (*getname)(struct socket *sock, struct sockaddr *uaddr,
             int *usockaddr_len, int peer);
  unsigned int (*poll)(struct file *file, struct socket *sock, struct poll_table_struct *wait);
  int   (*ioctl)(struct socket *sock, unsigned int cmd,
             unsigned long arg);
  int   (*listen)(struct socket *sock, int len);
  int   (*shutdown)(struct socket *sock, int flags);
  int   (*setsockopt)(struct socket *sock, int level, int optname,
             char *optval, int optlen);
  int   (*getsockopt)(struct socket *sock, int level, int optname,
             char *optval, int *optlen);
  int   (*sendmsg)(struct socket *sock, struct msghdr *m, int total_len, struct scm_cookie *scm);
  int   (*recvmsg)(struct socket *sock, struct msghdr *m, int total_len, int flags, struct scm_cookie *scm);
  int   (*mmap)(struct file *file, struct socket *sock, struct vm_area_struct * vma);
};
```
`struct socket` 结构的 `ops` 字段定义了一系列用于操作 `struct socket` 结构的函数，譬如当调用 `bind()` 函数将IP和端口绑定到一个socket时，便会调用 `ops` 字段的 `bind()` 函数来处理。那么这个 `ops` 字段什么时候被设置的呢？就是在调用 `net_proto_family` 结构的 `create()` 函数时被设置的。

现在又有个问题，`family` 类型的 `net_proto_family` 结构什么时候注册到 `net_families` 数组的呢？注册一个 `family` 类型的 `net_proto_family` 结构需要调用 `sock_register()` 函数，代码如下：
```cpp
int sock_register(struct net_proto_family *ops)
{
    int err;

    if (ops->family >= NPROTO) {
        printk(KERN_CRIT "protocol %d >= NPROTO(%d)\n", ops->family, NPROTO);
        return -ENOBUFS;
    }
    net_family_write_lock();
    err = -EEXIST;
    if (net_families[ops->family] == NULL) {
        net_families[ops->family]=ops;
        err = 0;
    }
    net_family_write_unlock();
    return err;
}
```
函数的实现比较简单，就是向 `net_families` 数组添加一个 `family` 类型的 `net_proto_family` 结构而已。那么 `Unix Domain Socket` 类型的 `net_proto_family` 结构什么时候注册呢？我们可以在文件 `net/unix/af_unix.c` 中看到如下代码：
```cpp
static int __init af_unix_init(void)
{
    ...
    sock_register(&unix_family_ops);
    ...
    return 0;
}

module_init(af_unix_init);
```
代码 `module_init(af_unix_init);` 表示在系统启动的时候会调用 `af_unix_init()` 函数，所以在系统启动时会注册一个类型为 `unix_family_ops` 的 `net_proto_family` 结构。我们再来看看 `unix_family_ops` 的定义：
```cpp
struct net_proto_family unix_family_ops = {
    PF_UNIX,
    unix_create
};
```
从上面的定义可以知道，当调用 `net_proto_family` 结构的 `create()` 函数时，就会调用 `unix_create()` 函数初始化 `socket`，`unix_create()` 函数的定义如下：
```cpp
static int unix_create(struct socket *sock, int protocol)
{
    if (protocol && protocol != PF_UNIX)
        return -EPROTONOSUPPORT;

    sock->state = SS_UNCONNECTED;

    switch (sock->type) {
    case SOCK_STREAM:
        sock->ops = &unix_stream_ops;
        break;
    case SOCK_RAW:
        sock->type = SOCK_DGRAM;
    case SOCK_DGRAM:
        sock->ops = &unix_dgram_ops;
        break;
    default:
        return -ESOCKTNOSUPPORT;
    }

    return unix_create1(sock) ? 0 : -ENOMEM;
}
```
`unix_create()` 函数会根据 `protocol` 变量的值来设置 `socket` 结构的 `ops` 字段，譬如 `protocol` 是 `SOCK_STREAM` （流式）时将会设置为 `unix_stream_ops`，而当 `protocol` 是 `SOCK_DGRAM` （数据报）或者 `SOCK_RAW` （原生）时，`ops` 字段将会设置为 `unix_dgram_ops`。下面来看看流式操作 `unix_stream_ops` 的定义：
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
```
所以，当对一个 `Unix Domain Socket` 调用 `bind()` 函数时，最终便会调用 `unix_bind()` 函数去处理。

另外，我们还发现 `unix_create()` 函数最后还调用了 `unix_create1()` 函数，这个函数主要是用来创建一个 `struct sock` 结构与 `struct socket` 结构相对应，`struct sock` 结构是功能实现的主体，很多功能相关的数据都存储在这个结构上。由于 `struct sock` 结构的定义比较复杂，所以这里就展示这个结构的定义了，有兴趣可以查阅文件 `include/net/sock.h` 中的定义。

## Unix Domain Socket 流程分析
前面我们介绍过创建一个 `Unix Domain Socket` 的服务端和客户端，那么现在我们来分析一下内核是怎么处理这些流程的。

* 创建一个 `Unix Domain Socket`

当在用户态调用 `socket()` 函数创建一个 `Unix Domain Socket` 时的调用链如下图：
![socket_unix_socket_call_stack](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/socket_unix_socket_call_stack.jpg)
