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
### socket() 函数实现
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
`unix_create()` 函数接着会调用 `unix_create1()` 来进行下一步创建工作，代码如下：
```cpp
static struct sock * unix_create1(struct socket *sock)
{
    struct sock *sk;

    if (atomic_read(&unix_nr_socks) >= 2*files_stat.max_files)
        return NULL;

    MOD_INC_USE_COUNT;
    sk = sk_alloc(PF_UNIX, GFP_KERNEL, 1);
    if (!sk) {
        MOD_DEC_USE_COUNT;
        return NULL;
    }

    atomic_inc(&unix_nr_socks);

    sock_init_data(sock,sk);

    sk->write_space = unix_write_space;
    sk->max_ack_backlog = sysctl_unix_max_dgram_qlen;
    sk->destruct = unix_sock_destructor;

    sk->protinfo.af_unix.dentry = NULL;
    sk->protinfo.af_unix.mnt = NULL;
    sk->protinfo.af_unix.lock = RW_LOCK_UNLOCKED;

    atomic_set(&sk->protinfo.af_unix.inflight, 0);
    init_MUTEX(&sk->protinfo.af_unix.readsem);
    init_waitqueue_head(&sk->protinfo.af_unix.peer_wait);
    sk->protinfo.af_unix.list=NULL;
    unix_insert_socket(&unix_sockets_unbound, sk);

    return sk;
}
```
`unix_create1()` 函数主要是创建并初始化一个 `struct sock` 结构，然后保存到 `socket` 对象的 `sk` 字段中。这个 `struct sock` 结构就是 `Unix Domain Sockets` 的操作实体，也就是说所有对socket的操作最终都是作用于这个 `struct sock` 结构上。`struct sock` 结构的定义非常复杂，所以这里就不把这个结构列出来，在分析的过程中涉及到这个结构的时候再加以说明。

### bind() 函数实现
`bind()` 系统调用最终会调用 `sys_bind()` 函数，而 `sys_bind()` 函数继而调用 `unix_bind()` 函数，调用链为： `bind() -> sys_bind() -> unix_bind()`，我们来看看 `unix_bind()` 函数的实现：
```cpp
static int unix_bind(struct socket *sock, struct sockaddr *uaddr, int addr_len)
{
    ...
    write_lock(&unix_table_lock);

    if (!sunaddr->sun_path[0]) {
        err = -EADDRINUSE;
        if (__unix_find_socket_byname(sunaddr, addr_len,
                          sk->type, hash)) {
            unix_release_addr(addr);
            goto out_unlock;
        }

        list = &unix_socket_table[addr->hash];
    } else {
        // 根据文件路径查找一个inode对象, 然后通过这个inode的编号查找到合适的哈希链表
        list = &unix_socket_table[dentry->d_inode->i_ino & (UNIX_HASH_SIZE-1)];
        sk->protinfo.af_unix.dentry = nd.dentry;
        sk->protinfo.af_unix.mnt = nd.mnt;
    }

    err = 0;
    __unix_remove_socket(sk);
    sk->protinfo.af_unix.addr = addr;
    __unix_insert_socket(list, sk); // 把socket添加到全局哈希表中

out_unlock:
    write_unlock(&unix_table_lock);
out_up:
    up(&sk->protinfo.af_unix.readsem);
out:
    return err;
    ...
}
```
`unix_socket()` 函数有一大部分代码是基于文件系统的，主要就是根据 socket 绑定的文件路径创建一个 `inode` 对象，然后将这个 `inode` 的编号作为哈希值找到 `Unix Domain Sockets` 全局哈希表对应的哈希链表，最后把这个 socket 添加到哈希链表中。通过这个操作，就可以把 socket 与一个文件路径关联上。

### listen() 函数
`listen()` 函数用于把 socket 设置为监听状态，`listen()` 函数首先会触发 `sys_listen()` 函数，然后 `sys_listen()` 函数会调用 `unix_listen()` 函数把 socket 设置为监听状态，调用链为：`listen() -> sys_listen() -> unix_listen()`，`unix_listen()` 函数的实现如下：
```cpp
static int unix_listen(struct socket *sock, int backlog)
{
    ...
    sk->max_ack_backlog = backlog;
    sk->state = TCP_LISTEN;
    ...
    return err;
}
```
`unix_socket()` 函数的实现很简单，主要是把 socket 的 `state` 字段设置为 `TCP_LISTEN`，表示当前 socket 处于监听状态。同时还设置了 socket 的 `max_ack_backlog` 字段，表示当前 socket 能够接收最大用户连接的队列长度。
