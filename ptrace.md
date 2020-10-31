# GDB原理之ptrace

在程序出现bug的时候，最好的解决办法就是通过 `GDB` 调试程序，然后找到程序出现问题的地方。比如程序出现 `段错误`（内存地址不合法）时，就可以通过 `GDB` 找到程序哪里访问了不合法的内存地址而导致的。

本文不是介绍 GDB 的使用方式，而是大概介绍 GDB 的实现原理，当然 GDB 是一个庞大而复杂的项目，不可能只通过一篇文章就能解释清楚，所以本文主要是介绍 GDB 使用的核心的技术 - `ptrace`。

## ptrace系统调用

`ptrace()` 系统调用是 Linux 提供的一个调试进程的工具，`ptrace()` 系统调用非常强大，它提供非常多的调试方式让我们去调试某一个进程，下面是 `ptrace()` 系统调用的定义：

```cpp
long ptrace(enum __ptrace_request request,  pid_t pid, void *addr,  void *data);
```

下面解释一下 `ptrace()` 各个参数的作用：

*   `request`：指定调试的指令，指令的类型很多，如：`PTRACE_TRACEME`、`PTRACE_PEEKUSER`、`PTRACE_CONT`、`PTRACE_GETREGS`等等，下面会介绍不同指令的作用。
*   `pid`：进程的ID（这个不用解释了）。
*   `addr`：进程的某个地址空间，可以通过这个参数对进程的某个地址进行读或写操作。
*   `data`：根据不同的指令，有不同的用途，下面会介绍。

`ptrace()` 系统调用详细的介绍可以参考以下链接：https://man7.org/linux/man-pages/man2/ptrace.2.html

## ptrace使用示例

下面通过一个简单例子来说明 `ptrace()` 系统调用的使用，这个例子主要介绍怎么使用 `ptrace()` 系统调用获取当前被调试（追踪）进程的各个寄存器的值，代码如下（ptrace.c）：

```c
#include <sys/ptrace.h>
#include <sys/types.h>
#include <sys/wait.h>
#include <unistd.h>
#include <sys/user.h>
#include <stdio.h>

int main()
{   pid_t child;
    struct user_regs_struct regs;

    child = fork();  // 创建一个子进程
    if(child == 0) { // 子进程
        ptrace(PTRACE_TRACEME, 0, NULL, NULL); // 表示当前进程进入被追踪状态
        execl("/bin/ls", "ls", NULL);          // 执行 `/bin/ls` 程序
    } 
    else { // 父进程
        wait(NULL); // 等待子进程发送一个 SIGCHLD 信号
        ptrace(PTRACE_GETREGS, child, NULL, &regs); // 获取子进程的各个寄存器的值
        printf("Register: rdi[%ld], rsi[%ld], rdx[%ld], rax[%ld], orig_rax[%ld]\n",
                regs.rdi, regs.rsi, regs.rdx,regs.rax, regs.orig_rax); // 打印寄存器的值
        ptrace(PTRACE_CONT, child, NULL, NULL); // 继续运行子进程
        sleep(1);
    }
    return 0;
}
```

通过命令 `gcc ptrace.c -o ptrace` 编译并运行上面的程序会输出如下结果：

```shell
Register: rdi[0], rsi[0], rdx[0], rax[0], orig_rax[59]
ptrace  ptrace.c
```

上面结果的第一行是由父进程输出的，主要是打印了子进程执行 `/bin/ls` 程序后各个寄存器的值。而第二行是由子进程输出的，主要是打印了执行 `/bin/ls` 程序后输出的结果。

下面解释一下上面程序的执行流程：

1.  主进程调用 `fork()` 系统调用创建一个子进程。
2.  子进程调用 `ptrace(PTRACE_TRACEME,...)` 把自己设置为被追踪状态，并且调用 `execl()` 执行 `/bin/ls` 程序。
3.  被设置为追踪（TRACE）状态的子进程执行 `execl()` 的程序后，会向父进程发送 `SIGCHLD` 信号，并且暂停自身的执行。
4.  父进程通过调用 `wait()` 接收子进程发送过来的信号，并且开始追踪子进程。
5.  父进程通过调用 `ptrace(PTRACE_GETREGS, child, ...)` 来获取到子进程各个寄存器的值，并且打印寄存器的值。
6.  父进程通过调用 `ptrace(PTRACE_CONT, child, ...)` 让子进程继续执行下去。

从上面的例子可以知道，通过向 `ptrace()` 函数的 `request` 参数传入不同的值时，就有不同的效果。比如传入 `PTRACE_TRACEME` 就可以让进程进入被追踪状态，而传入 `PTRACE_GETREGS` 时，就可以获取被追踪的子进程各个寄存器的值等。

本来我想使用 `ptrace` 实现一个简单的调试工具，但在网上找到了一位 Google 的大神 `Eli Bendersky` 写了类似的系列文章，所以我就不再重复工作了，在这里贴一下文章的链接：

* https://eli.thegreenplace.net/2011/01/23/how-debuggers-work-part-1/
* https://eli.thegreenplace.net/2011/01/27/how-debuggers-work-part-2-breakpoints
* https://eli.thegreenplace.net/2011/02/07/how-debuggers-work-part-3-debugging-information

但由于 `Eli Bendersky` 大神的文章只是介绍使用 `ptrace` 实现一个简单的进程调试工具，而没有介绍 `ptrace` 的原理和实现，所以这里为了填补这个空缺，下面就详细介绍一下 `ptrace` 的原理与实现。

## ptrace实现原理

