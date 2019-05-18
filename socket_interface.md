# Socket接口的分层
Socket的英文原本意思是 `孔` 或 `插座`。但在计算机科学中通常被称作为 `套接字`，主要用于相同机器的不同进程间或者不同机器间的通信。Socket的使用很多网络编程的书籍都有介绍，所以本文不打算介绍Socket的使用，只讨论Socket的具体实现，所以如果对Socket不太了解的同学可以先查阅Socket相关的资料或者书籍。

在Linux内核中，Socket的实现分为三层，第一层是 `GLIBC接口层`，第二层是 `BSD接口层`，第三层是 `具体的协议层`（如Unix sokcet或者INET socket）。如下图所示：
![socket layer](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/socket-layer.jpg)

GLIBC层在用户态实现，提供一系列的socket族系统调用让用户使用。BSD层在内核态实现，主要是为了让不同的协议能够使用同一套接口来访问而创造的，如上图所示， `Unix socket` 和 `Inet socket` 都可以通过接入 `BSD接口层` 来向用户提供相同的接口。 `具体的协议层` 是为了实现不同的协议或者功能而存在的，如 `Unix socket` 主要是用于进程间通信，`Inet socket` 主要用于网络数据传输等。

## GLIBC接口层
`GLIBC接口层` 提供了一系列的接口函数供用户使用（可以成为 `Socket族系统调用`），如下：
* socket()
* bind()
* listen()
* accept()
* connect()
* recv()
* send()
* recvfrom()
* sendto()
* ...

例如 `socket()` 接口用于创建一个socket句柄，而 `bind()` 函数将一个socket绑定到指定的IP和端口上。当然，系统调用最终都会调用到内核态的某个内核函数来进行处理，在系统调用一章我们介绍过相关的原理，所以这里只会介绍一下这些系统调用最终会调用哪些内核函数。

### GLIBC层实现原理
我们先来看看 `GLIBC` 是怎么定义这些系统调用的吧，首先来看看 `socket()` 函数的定义如下：
```asm
#define P(a, b) P2(a, b)
#define P2(a, b) a##b

    .text

.globl P(__,socket)
ENTRY (P(__,socket))
    movl %ebx, %edx

    movl $SYS_ify(socketcall), %eax        // 系统调用号

    movl $P(SOCKOP_,socket), %ebx          // 系统调用的第一个参数
    lea 4(%esp), %ecx                      // 系统调用的第二个参数

    int $0x80

    movl %edx, %ebx

    cmpl $-125, %eax
    jae syscall_error

    ret
```
虽然 `socket()` 函数是使用汇编来实现的，但是也比较容易理解，我们已经知道在用户态必须使用 `int 0x80` 中断来触发系统调用的，而要调用的系统调用编号保存在寄存器 `eax` 中，第一个参数保存在 `ebx` 寄存器中，而第二个参数保存在 `ecx` 中。

所以从上面的代码可以看出，调用 `socket()` 函数时会把 `eax` 的值设置为 `sys_socketcall`，把 `ebx` 的值会设置为 `SOCKOP_socket`，而把 `ecx` 的值设置为调用 `socket()` 函数时第一个参数的地址。然后通过代码 `int 0x80` 来触发一次系统调用中断，那么最终调用的是 `sys_socketcall()` 内核函数，而第一个参数的值为 `SOCKOP_socket`，第二个参数的值为调用 `socket()` 函数时第一个参数的地址。

那么 `bind()` 函数又是怎么定义的呢？因为有了 `socket()` 函数的定义，那么所有 `Socket族系统调用` 都可以使用这个模板来实现，例如 `bind()` 函数的定义如下：
```cpp
#define	socket	bind
#include <socket.S>
```
可以看到，`bind()` 函数直接套用了 `socket()` 函数实现的模板，只是把 `socket` 这个名字替换成 `bind` 而已，替换之后 `ebx` 的值就会变成 `SOCKOP_bind`，其他都跟 `socket()` 函数一样，所以这时传给 `sys_socketcall()` 函数的第一个参数就变成 `SOCKOP_bind`了。

## BSD接口层
前面说了，`BSD接口层` 是为了能够使用相同的接口来操作不同协议而创造的。有面向对象编程经验的读者可能会发现，`BSD接口层` 使用的技巧与面向对象的 `接口` 概念非常相似。主要的方式是 `BSD接口层` 定义了一些接口，`具体的协议层` 必须实现这些接口才能接入到 `BSD接口层`。

为了实现这种机制，Linux定义了一个 `struct socket` 的结构体，每个socket都与一个 `struct socket` 的结构对应，其定义如下：
```cpp
struct socket
{
    socket_state          state;

    unsigned long         flags;
    struct proto_ops     *ops;
    struct inode         *inode;
    struct fasync_struct *fasync_list;
    struct file          *file;
    struct sock          *sk;
    wait_queue_head_t    wait;

    short                type;
    unsigned char        passcred;
};
```
可以把这个结构体想象成钩子，要在上面挂什么由用户自己决定。其比较重要的字段是 `ops` 和 `sk`。`ops` 字段类型为 `struct proto_ops`，其定义了一系列操作socket的方法。而 `sk` 字段的类型为 `struct sock`， 用于保存具体协议所操作的真实对象。

我们先来看看 `struct proto_ops` 结构的定义：
```cpp
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
  unsigned int (*poll)  (struct file *file, struct socket *sock, struct poll_table_struct *wait);
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
从上面的代码可以看出，`struct proto_ops` 结构主要是定义一系列的函数接口，每个 `具体的协议层` 必须提供一个 `struct proto_ops` 结构挂载到 `struct socket` 结构的 `ops` 字段上。所以当用户调用 `bind()` 系统调用时真实调用的是：`socket->ops->bind()`。

### sys_socketcall()函数
前面说过，所有的 `Socket族系统调用` 最终都会调用 `sys_socketcall()` 函数来处理用户的请求，我们来看看 `sys_socketcall()` 函数的实现：
```cpp
asmlinkage long sys_socketcall(int call, unsigned long *args)
{
    unsigned long a[6];
    unsigned long a0,a1;
    int err;

    if(call<1||call>SYS_RECVMSG)
        return -EINVAL;

    /* copy_from_user should be SMP safe. */
    if (copy_from_user(a, args, nargs[call]))
        return -EFAULT;
        
    a0=a[0];
    a1=a[1];
    
    switch(call) 
    {
        case SYS_SOCKET:
            err = sys_socket(a0,a1,a[2]);
            break;
        case SYS_BIND:
            err = sys_bind(a0,(struct sockaddr *)a1, a[2]);
            break;
        case SYS_CONNECT:
            err = sys_connect(a0, (struct sockaddr *)a1, a[2]);
            break;
        ...
    }
    return err;
}
```
从 `sys_socketcall()` 函数可以看出，根据参数 `call` 不同的值会调用不同的内核函数，譬如 `call` 的值为 `SYS_SOCKET` 时会调用 `sys_socket()` 函数，而 `call` 的值为 `SYS_BIND` 时会调用 `sys_bind()` 函数。而参数 `args` 就是在用户态给 `Socket族系统调用` 传入的参数列表地址，Linux内核会先使用 `copy_from_user()` 函数把这些参数复制到内核空间。

前面说过，在用户空间调用 `socket()` 系统调用时会把参数 `call` 的值设置为 `SOCKOP_socket`，它的值跟 `sys_socketcall()` 函数中 `SYS_SOCKET` 是一致的，我们可以通过下面的代码看出端倪：
```cpp
// GLIBC 的定义
#define SOCKOP_socket       1
#define SOCKOP_bind         2
#define SOCKOP_connect      3
#define SOCKOP_listen       4
#define SOCKOP_accept       5
#define SOCKOP_getsockname  6
#define SOCKOP_getpeername  7
#define SOCKOP_socketpair   8
#define SOCKOP_send         9
#define SOCKOP_recv         10
#define SOCKOP_sendto       11
#define SOCKOP_recvfrom     12
#define SOCKOP_shutdown     13
#define SOCKOP_setsockopt   14
#define SOCKOP_getsockopt   15
#define SOCKOP_sendmsg      16
#define SOCKOP_recvmsg      17

// Linux 内核的定义
#define SYS_SOCKET      1       /* sys_socket(2)        */
#define SYS_BIND        2       /* sys_bind(2)          */
#define SYS_CONNECT     3       /* sys_connect(2)       */
#define SYS_LISTEN      4       /* sys_listen(2)        */
#define SYS_ACCEPT      5       /* sys_accept(2)        */
#define SYS_GETSOCKNAME 6       /* sys_getsockname(2)   */
#define SYS_GETPEERNAME 7       /* sys_getpeername(2)   */
#define SYS_SOCKETPAIR  8       /* sys_socketpair(2)    */
#define SYS_SEND        9       /* sys_send(2)          */
#define SYS_RECV        10      /* sys_recv(2)          */
#define SYS_SENDTO      11      /* sys_sendto(2)        */
#define SYS_RECVFROM    12      /* sys_recvfrom(2)      */
#define SYS_SHUTDOWN    13      /* sys_shutdown(2)      */
#define SYS_SETSOCKOPT  14      /* sys_setsockopt(2)    */
#define SYS_GETSOCKOPT  15      /* sys_getsockopt(2)    */
#define SYS_SENDMSG     16      /* sys_sendmsg(2)       */
#define SYS_RECVMSG     17      /* sys_recvmsg(2)       */
```
从上面的定义可以看出，在 GLIBC 中的定义跟 Linux 内核中的定义是一一对应的。

所以从中得到，当在用户态调用 `socket()` 函数时实际调用的是 `sys_socket()` 内核函数，其他的 `Socket族系统调用` 道理与 `socket()` 系统调用一致。

通过下面一幅图来展示 `Socket族系统调用` 的原理：
![socket interfaces](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/socket_interface.jpg)

### sys_socket()函数
`sys_socket()` 函数用于创建一个 socket 对象，并且返回一个文件描述符。其实现如下：
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
参数 `family` 指定 `具体协议层`，可以选择的协议非常多，下面列举几个：
```cpp
#define AF_UNIX     1   /* Unix domain sockets      */
#define AF_LOCAL    1   /* POSIX name for AF_UNIX   */
#define AF_INET     2   /* Internet IP Protocol     */
#define AF_AX25     3   /* Amateur Radio AX.25      */
#define AF_IPX      4   /* Novell IPX               */
...
```
例如 `AF_UNIX` 指定的是 `Unix socket`，`AF_INET` 指定的是 `以太网协议` 等。而参数 `type` 用于指定传输数据的类型，有一下几种选择：
```cpp
#define SOCK_STREAM 1       /* stream (connection) socket   */
#define SOCK_DGRAM  2       /* datagram (conn.less) socket  */
#define SOCK_RAW    3       /* raw socket                   */
#define SOCK_RDM    4       /* reliably-delivered message   */
#define SOCK_SEQPACKET  5   /* sequential packet socket     */
#define SOCK_PACKET 10      /* linux specific way of        */
```
例如 `SOCK_STREAM` 类型指定的是流方式，而 `SOCK_DGRAM` 类型指定的是数据报方式等。最后一个 `protocol` 参数看起来也是协议的意思，跟 `family` 好像重复了。事实上 `family` 所指定的协议偏向于物理介质，如 `Unix socket` 是用于进程间通信的，而 `Inet socket` 是用于以太网传输数据的。而 `protocol` 所指定的协议偏向于逻辑上的协议，如 `TCP`、`UDP` 等。举个栗子，如果把 `family` 比作是不同交通工具（飞机、汽车、火车等）的话，那么 `protocol` 就是大巴、的士和小车。

`sys_socket()` 函数首先调用 `sock_create()` 创建一个 `struct socket` 结构，然后通过调用 `sock_map_fd()` 函数把此 `struct  socket` 结构与一个文件描述符关联起来，最后把文件描述符返回给用户。我们先来看看 `sock_create()` 函数的实现：
```cpp
int sock_create(int family, int type, int protocol, struct socket **res)
{
    int i;
    struct socket *sock;

    ...
    net_family_read_lock();
    ...

    if (!(sock = sock_alloc())) {
        printk(KERN_WARNING "socket: no more sockets\n");
        i = -ENFILE;
        goto out;
    }

    sock->type  = type;

    if ((i = net_families[family]->create(sock, protocol)) < 0)  {
        sock_release(sock);
        goto out;
    }

    *res = sock;

out:
    net_family_read_unlock();
    return i;
}
```
`sock_create()` 函数首先调用 `sock_alloc()` 申请一个 `struct socket` 结构，然后调用指定协议族的 `create()` 函数（`net_families[family]->create()`）进行进一步的创建功能。`net_families` 变量的类型为 `struct net_proto_family`，其定义如下：
```cpp
struct net_proto_family {
    int family;
    int (*create)(struct socket *sock, int protocol);
    ...
};
```
`family` 字段对应的就是具体的协议族，而 `create` 字段指定了其创建socket的方法。一个具体协议族需要通过调用 `sock_register()` 函数向系统注册其创建socket的方法。例如 `Unix socket` 就在初始化时通过下面的代码注册：
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
所以从上面的代码可以指定，对于 `Unix socket` 的话，`net_families[family]->create()` 这行代码实际调用的是 `unix_create()` 函数。
