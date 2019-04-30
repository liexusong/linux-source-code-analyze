## 什么是信号
信号本质上是在软件层次上对中断机制的一种模拟，其主要有以下几种来源：
* 程序错误：除零，非法内存访问等。
* 外部信号：终端Ctrl-C产生SGINT信号，定时器到期产生SIGALRM等。
* 显式请求：kill函数允许进程发送任何信号给其他进程或进程组。

目前Linux支持64种信号。信号分为非实时信号(不可靠信号)和实时信号（可靠信号）两种类型，对应于 Linux 的信号值为 1-31 和 34-64。

信号是异步的，一个进程不必通过任何操作来等待信号的到达。事实上，进程也不知道信号到底什么时候到达。一般来说，我们只需要在进程中设置信号相应的处理函数，当有信号到达的时候，由系统异步触发相应的处理函数即可。如下代码：
```cpp
#include <signal.h>
#include <unistd.h>
#include <stdio.h>

void sigcb(int signo) {
    switch (signo) {
    case SIGHUP:
        printf("Get a signal -- SIGHUP\n");
        break;
    case SIGINT:
        printf("Get a signal -- SIGINT\n");
        break;
    case SIGQUIT:
        printf("Get a signal -- SIGQUIT\n");
        break;
    }
    return;
}

int main() {
    signal(SIGHUP, sigcb);
    signal(SIGINT, sigcb);
    signal(SIGQUIT, sigcb);
    for (;;) {
        sleep(1);
    }
}
```
运行程序后，当我们按下 `Ctrl+C` 后，屏幕上将会打印 `Get a signal -- SIGINT`。当然我们可以使用 `kill -s SIGINT pid` 命令来发送一个信号给进程，屏幕同样打印出 `Get a signal -- SIGINT` 的信息。

## 信号实现原理
接下来我们分析一下Linux对信号处理机制的实现原理。

### 信号处理相关的数据结构
在进程管理结构 `task_struct` 中有几个与信号处理相关的字段，如下：
```cpp
struct task_struct {
    ...
    int sigpending;
    ...
    struct signal_struct *sig;
    sigset_t blocked;
    struct sigpending pending;
    ...
}
```
成员 `sigpending` 表示进程是否有信号需要处理（1表示有，0表示没有）。成员 `blocked` 表示被屏蔽的信息，每个位代表一个被屏蔽的信号。成员 `sig` 表示信号相应的处理方法，其类型是 `struct signal_struct`，定义如下：
```cpp
#define  _NSIG  64

struct signal_struct {
	atomic_t		count;
	struct k_sigaction	action[_NSIG];
	spinlock_t		siglock;
};

typedef void (*__sighandler_t)(int);

struct sigaction {
	__sighandler_t sa_handler;
	unsigned long sa_flags;
	void (*sa_restorer)(void);
	sigset_t sa_mask;
};

struct k_sigaction {
	struct sigaction sa;
};
```
可以看出，`struct signal_struct` 是个比较复杂的结构，其 `action` 成员是个 `struct k_sigaction` 结构的数组，数组中的每个成员代表着相应信号的处理信息，而 `struct k_sigaction` 结构其实是 `struct sigaction` 的简单封装。

我们再来看看 `struct  sigaction` 这个结构，其中 `sa_handler` 成员是类型为 `__sighandler_t` 的函数指针，代表着信号处理的方法。

最后我们来看看 `struct task_struct` 结构的 `pending` 成员，其类型为 `struct sigpending`，存储着进程接收到的信号队列，`struct sigpending` 的定义如下：
```cpp
struct sigqueue {
	struct sigqueue *next;
	siginfo_t info;
};

struct sigpending {
	struct sigqueue *head, **tail;
	sigset_t signal;
};
```
当进程接收到一个信号时，就需要把接收到的信号添加 `pending` 这个队列中。

### 发送信号
可以通过 `kill()` 系统调用发送一个信号给指定的进程，其原型如下：
```cpp
int kill(pid_t pid, int sig);
```
参数 `pid` 指定要接收信号进程的ID，而参数 `sig` 是要发送的信号。`kill()` 系统调用最终会进入内核态，并且调用内核函数 `sys_kill()`，代码如下：
```cpp
asmlinkage long
sys_kill(int pid, int sig)
{
	struct siginfo info;

	info.si_signo = sig;
	info.si_errno = 0;
	info.si_code = SI_USER;
	info.si_pid = current->pid;
	info.si_uid = current->uid;

	return kill_something_info(sig, &info, pid);
}
```
