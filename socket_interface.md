# BSD层接口
Socket的英文原本意思是 `孔` 或 `插座`。但在计算机科学中通常被称作为 `套接字`，主要用于相同机器的不同进程间或者不同机器间的通信。Socket的使用很多网络编程的书籍都有介绍，所以本文不打算介绍Socket的使用，只讨论Socket的具体实现，所以如果对Socket不太了解的同学可以先查阅Socket相关的资料或者书籍。

## Socket族系统调用
`Soket族系统调用` 提供了一系列的接口函数供用户使用，如下：
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

### Socket族系统调用在 GLIBC 中的定义
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

