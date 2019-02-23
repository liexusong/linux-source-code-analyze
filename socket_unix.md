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
`sock_create()` 函数首先调用 `sock_alloc()` 申请一个 `struct socket` 结构，然后根据 `family` 的值从 `net_families` 数组中找到对应的 `net_proto_family` 结构，然后通过此 `net_proto_family` 结构的 `create` 函数来初始化此 `struct socket` 结构。
