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

>   本文使用的 Linux 2.4.16 版本的内核
>
>   看懂本文需要的基础：进程调度，内存管理和信号处理相关知识。

调用 `ptrace()` 系统函数时会触发调用内核的 `sys_ptrace()` 函数，由于不同的 CPU 架构有着不同的调试方式，所以 Linux 为每种不同的 CPU 架构实现了不同的 `sys_ptrace()` 函数，而本文主要介绍的是 `X86 CPU` 的调试方式，所以 `sys_ptrace()` 函数所在文件是 `linux-2.4.16/arch/i386/kernel/ptrace.c`。

`sys_ptrace()` 函数的主体是一个 `switch` 语句，会传入的 `request` 参数不同进行不同的操作，如下：

```c
asmlinkage int sys_ptrace(long request, long pid, long addr, long data)
{
    struct task_struct *child;
    struct user *dummy = NULL;
    int i, ret;

    ...

    read_lock(&tasklist_lock);
    child = find_task_by_pid(pid); // 获取 pid 对应的进程 task_struct 对象
    if (child)
        get_task_struct(child);
    read_unlock(&tasklist_lock);
    if (!child)
        goto out;

    if (request == PTRACE_ATTACH) {
        ret = ptrace_attach(child);
        goto out_tsk;
    }

    ...

    switch (request) {
    case PTRACE_PEEKTEXT:
    case PTRACE_PEEKDATA:
        ...
    case PTRACE_PEEKUSR:
        ...
    case PTRACE_POKETEXT:
    case PTRACE_POKEDATA:
        ...
    case PTRACE_POKEUSR:
        ...
    case PTRACE_SYSCALL:
    case PTRACE_CONT:
        ...
    case PTRACE_KILL: 
        ...
    case PTRACE_SINGLESTEP:
        ...
    case PTRACE_DETACH:
        ...
    }
out_tsk:
    free_task_struct(child);
out:
    unlock_kernel();
    return ret;
}
```

从上面的代码可以看出，`sys_ptrace()` 函数首先根据进程的 `pid` 获取到进程的 `task_struct` 对象。然后根据传入不同的 `request` 参数在 `switch` 语句中进行不同的操作。

`ptrace()` 支持的所有 `request` 操作定义在 `linux-2.4.16/include/linux/ptrace.h` 文件中，如下：

```c
#define PTRACE_TRACEME         0
#define PTRACE_PEEKTEXT        1
#define PTRACE_PEEKDATA        2
#define PTRACE_PEEKUSR         3
#define PTRACE_POKETEXT        4
#define PTRACE_POKEDATA        5
#define PTRACE_POKEUSR         6
#define PTRACE_CONT            7
#define PTRACE_KILL            8
#define PTRACE_SINGLESTEP      9
#define PTRACE_ATTACH       0x10
#define PTRACE_DETACH       0x11
#define PTRACE_SYSCALL        24
#define PTRACE_GETREGS        12
#define PTRACE_SETREGS        13
#define PTRACE_GETFPREGS      14
#define PTRACE_SETFPREGS      15
#define PTRACE_GETFPXREGS     18
#define PTRACE_SETFPXREGS     19
#define PTRACE_SETOPTIONS     21
```

由于 `ptrace()` 提供的操作比较多，所以本文只会挑选一些比较有代表性的操作进行解说，比如 `PTRACE_TRACEME`、`PTRACE_SINGLESTEP`、`PTRACE_PEEKTEXT`、`PTRACE_PEEKDATA` 和 `PTRACE_CONT` 等，而其他的操作，有兴趣的朋友可以自己去分析其实现原理。

### 进入被追踪模式（PTRACE_TRACEME操作）

当要调试一个进程时，需要使进程进入被追踪模式，怎么使进程进入被追踪模式呢？有两个方法：

*   被调试的进程调用 `ptrace(PTRACE_TRACEME, ...)` 来使自己进入被追踪模式。
*   调试进程（如GDB）调用 `ptrace(PTRACE_ATTACH, pid, ...)` 来使指定的进程进入被追踪模式。

第一种方式是进程自己主动进入被追踪模式，而第二种是进程被动进入被追踪模式。

被调试的进程必须进入被追踪模式才能进行调试，因为 Linux 会对被追踪的进程进行一些特殊的处理。下面我们主要介绍第一种进入被追踪模式的实现，就是 `PTRACE_TRACEME` 的操作过程，代码如下：

```c
asmlinkage int sys_ptrace(long request, long pid, long addr, long data)
{
    ...
    if (request == PTRACE_TRACEME) {
        if (current->ptrace & PT_PTRACED)
            goto out;
        current->ptrace |= PT_PTRACED; // 标志 PTRACE 状态
        ret = 0;
        goto out;
    }
    ...
}
```

从上面的代码可以发现，`ptrace()` 对 `PTRACE_TRACEME` 的处理就是把当前进程标志为 `PTRACE` 状态。

当然事情不会这么简单，因为当一个进程被标记为 `PTRACE` 状态后，当调用 `exec()` 函数去执行一个外部程序时，将会暂停当前进程的运行，并且发送一个 `SIGCHLD` 给父进程。父进程接收到 `SIGCHLD` 信号后就可以对被调试的进程进行调试。

我们来看看 `exec()` 函数是怎样实现上述功能的，`exec()` 函数的执行过程为 `sys_execve() -> do_execve() -> load_elf_binary()`：

```c
static int load_elf_binary(struct linux_binprm * bprm, struct pt_regs * regs)
{
    ...
    if (current->ptrace & PT_PTRACED)
        send_sig(SIGTRAP, current, 0);
    ...
}
```

从上面代码可以看出，当进程被标记为 `PTRACE` 状态时，执行 `exec()` 函数后便会发送一个 `SIGTRAP` 的信号给当前进程。

我们再来看看，进程是怎么处理 `SIGTRAP` 信号的。信号是通过 `do_signal()` 函数进行处理的，而对 `SIGTRAP` 信号的处理逻辑如下：

```c
int do_signal(struct pt_regs *regs, sigset_t *oldset) 
{
    for (;;) {
        unsigned long signr;

        spin_lock_irq(&current->sigmask_lock);
        signr = dequeue_signal(&current->blocked, &info);
        spin_unlock_irq(&current->sigmask_lock);

        // 如果进程被标记为 PTRACE 状态
        if ((current->ptrace & PT_PTRACED) && signr != SIGKILL) {
            /* 让调试器运行  */
            current->exit_code = signr;
            current->state = TASK_STOPPED;   // 让自己进入停止运行状态
            notify_parent(current, SIGCHLD); // 发送 SIGCHLD 信号给父进程
            schedule();                      // 让出CPU的执行权限
            ...
        }
    }
}
```

上面的代码主要做了3件事：

1.  如果当前进程被标记为 PTRACE 状态，那么就使自己进入停止运行状态。
2.  发送 SIGCHLD 信号给父进程。
3.  让出 CPU 的执行权限，使 CPU 执行其他进程。

执行以上过程后，被追踪进程便进入了调试模式，过程如下图：

![traceme](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/traceme.jpg)

当父进程（调试进程）接收到 `SIGCHLD` 信号后，表示被调试进程已经标记为被追踪状态并且停止运行，那么调试进程就可以开始进行调试了。

### 获取被调试进程的内存数据（PTRACE_PEEKTEXT / PTRACE_PEEKDATA）

调试进程（如GDB）可以通过调用 `ptrace(PTRACE_PEEKDATA, pid, addr, data)` 来获取被调试进程 `addr` 处虚拟内存地址的数据，但每次只能读取一个大小为 4字节的数据。

我们来看看 `ptrace()` 对 `PTRACE_PEEKDATA` 操作的处理过程，代码如下：

```c
asmlinkage int sys_ptrace(long request, long pid, long addr, long data)
{
    ...
    switch (request) {
    case PTRACE_PEEKTEXT:
    case PTRACE_PEEKDATA: {
        unsigned long tmp;
        int copied;

        copied = access_process_vm(child, addr, &tmp, sizeof(tmp), 0);
        ret = -EIO;
        if (copied != sizeof(tmp))
            break;
        ret = put_user(tmp, (unsigned long *)data);
        break;
    }
    ...
}
```

从上面代码可以看出，对 `PTRACE_PEEKTEXT` 和 `PTRACE_PEEKDATA` 的处理是相同的，主要是通过调用 `access_process_vm()` 函数来读取被调试进程 `addr` 处的虚拟内存地址的数据。

`access_process_vm()` 函数的实现主要涉及到 `内存管理` 相关的知识，可以参考我以前对内存管理分析的文章，这里主要大概说明一下 `access_process_vm()` 的原理。

我们知道每个进程都有个 `mm_struct` 的内存管理对象，而 `mm_struct` 对象有个表示虚拟内存与物理内存映射关系的页目录的指针 `pgd`。如下：

```c
struct mm_struct {
    ...
    pgd_t *pgd; /* 页目录指针 */
    ...
}
```

而 `access_process_vm()` 函数就是通过进程的页目录来找到 `addr` 虚拟内存地址映射的物理内存地址，然后把此物理内存地址处的数据复制到 `data` 变量中。如下图所示：

![memory_map](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/memory_map.jpg)

`access_process_vm()` 函数的实现这里就不分析了，有兴趣的读者可以参考我之前对内存管理分析的文章自行进行分析。

### 单步调试模式（PTRACE_SINGLESTEP）

单步调试是一个比较有趣的功能，当把被调试进程设置为单步调试模式后，被调试进程没执行一条CPU指令都会停止执行，并且向父进程（调试进程）发送一个 SIGCHLD 信号。

我们来看看 `ptrace()` 函数对 `PTRACE_SINGLESTEP` 操作的处理过程，代码如下：

```c
asmlinkage int sys_ptrace(long request, long pid, long addr, long data)
{
    ...
    switch (request) {
    case PTRACE_SINGLESTEP: {  /* set the trap flag. */
        long tmp;
        ...
        tmp = get_stack_long(child, EFL_OFFSET) | TRAP_FLAG;
        put_stack_long(child, EFL_OFFSET, tmp);
        child->exit_code = data;
        /* give it a chance to run. */
        wake_up_process(child);
        ret = 0;
        break;
    }
    ...
}
```

要把被调试的进程设置为单步调试模式，英特尔的 X86 CPU 提供了一个硬件的机制，就是通过把 `eflags` 寄存器的 `Trap Flag` 设置为1即可。

当把 `eflags` 寄存器的 `Trap Flag` 设置为1后，CPU 每执行一条指令便会产生一个异常，然后会触发 Linux 的异常处理，Linux 便会发送一个 `SIGTRAP` 信号给被调试的进程。`eflags` 寄存器的各个标志如下图：

![eflags-register](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/eflags-register.png)

从上图可知，`eflags` 寄存器的第8位就是单步调试模式的标志。

所以 `ptrace()` 函数的以下2行代码就是设置 `eflags` 进程的单步调试标志：

```c
tmp = get_stack_long(child, EFL_OFFSET) | TRAP_FLAG;
put_stack_long(child, EFL_OFFSET, tmp);
```

而 `get_stack_long(proccess, offset)` 函数用于获取进程栈 `offset` 处的值，而 `EFL_OFFSET` 偏移量就是 `eflags` 寄存器的值。所以上面两行代码的意思就是：

1.  获取进程的 `eflags` 寄存器的值，并且设置 `Trap Flag` 标志。
2.  把新的值设置到进程的 `eflags` 寄存器中。

设置完 `eflags` 寄存器的值后，就调用 `wake_up_process()` 函数把被调试的进程唤醒，让其进入运行状态。单步调试过程如下图：

![single-trace](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/single-trace.jpg)

处于单步调试模式时，被调试进程每执行一条指令都会触发一次 `SIGTRAP` 信号，而被调试进程处理 `SIGTRAP` 信号时会发送一个 `SIGCHLD` 信号给父进程（调试进程），并且让自己停止执行。

而父进程（调试进程）接收到 `SIGCHLD` 后，就可以对被调试的进程进行各种操作，比如读取被调试进程内存的数据和寄存器的数据，或者通过调用 `ptrace(PTRACE_CONT, child,...)` 来让被调试进程进行运行等。

## 小结

由于 `ptrace()` 的功能十分强大，所以本文只能抛砖引玉，没能对其所有功能进行分析。另外断点功能并不是通过 `ptrace()` 函数实现的，而是通过 `int3` 指令来实现的，在 `Eli Bendersky` 大神的文章有介绍。而对于 `ptrace()` 的所有功能，只能读者自己慢慢看代码来体会了。

