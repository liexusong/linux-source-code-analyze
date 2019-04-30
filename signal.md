## 什么是信号
信号本质上是在软件层次上对中断机制的一种模拟，其主要有以下几种来源：
* 程序错误：除零，非法内存访问等。
* 外部信号：终端 `Ctrl-C` 产生 `SGINT` 信号，定时器到期产生SIGALRM等。
* 显式请求：kill函数允许进程发送任何信号给其他进程或进程组。

目前 Linux 支持64种信号。信号分为非实时信号(不可靠信号)和实时信号（可靠信号）两种类型，对应于 Linux 的信号值为 1-31 和 34-64。

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
`sys_kill()` 的代码比较简单，首先初始化 `info` 变量的成员，接着调用 `kill_something_info()` 函数来处理发送信号的操作。`kill_something_info()` 函数的代码如下：
```cpp
static int kill_something_info(int sig, struct siginfo *info, int pid)
{
	if (!pid) {
		return kill_pg_info(sig, info, current->pgrp);
	} else if (pid == -1) {
		int retval = 0, count = 0;
		struct task_struct * p;

		read_lock(&tasklist_lock);
		for_each_task(p) {
			if (p->pid > 1 && p != current) {
				int err = send_sig_info(sig, info, p);
				++count;
				if (err != -EPERM)
					retval = err;
			}
		}
		read_unlock(&tasklist_lock);
		return count ? retval : -ESRCH;
	} else if (pid < 0) {
		return kill_pg_info(sig, info, -pid);
	} else {
		return kill_proc_info(sig, info, pid);
	}
}
```
`kill_something_info()` 函数根据传入`pid` 的不同来进行不同的操作，有如下4中可能：
* `pid` 等于0时，表示信号将送往所有与调用 `kill()` 的那个进程属同一个使用组的进程。
* `pid` 大于零时，`pid` 是信号要送往的进程ID。
* `pid` 等于-1时，信号将送往调用进程有权给其发送信号的所有进程，除了进程1(init)。
* `pid` 小于-1时，信号将送往以-pid为组标识的进程。

我们这里只分析 `pid` 大于0的情况，从上面的代码可以知道，当 `pid` 大于0时，会调用 `kill_proc_info()` 函数来处理信号发送操作，其代码如下：
```cpp
inline int
kill_proc_info(int sig, struct siginfo *info, pid_t pid)
{
	int error;
	struct task_struct *p;

	read_lock(&tasklist_lock);
	p = find_task_by_pid(pid);
	error = -ESRCH;
	if (p)
		error = send_sig_info(sig, info, p);
	read_unlock(&tasklist_lock);
	return error;
}
```
`kill_proc_info()` 首先通过调用 `find_task_by_pid()` 函数来获得 `pid` 对应的进程管理结构，然后通过 `send_sig_info()` 函数来发送信号给此进程，`send_sig_info()` 函数代码如下：
```cpp
int
send_sig_info(int sig, struct siginfo *info, struct task_struct *t)
{
    unsigned long flags;
    int ret;

    ret = -EINVAL;
    if (sig < 0 || sig > _NSIG)
        goto out_nolock;

    ret = -EPERM;
    if (bad_signal(sig, info, t))
        goto out_nolock;

    ret = 0;
    if (!sig || !t->sig)
        goto out_nolock;

    spin_lock_irqsave(&t->sigmask_lock, flags);
    handle_stop_signal(sig, t);

    if (ignored_signal(sig, t))
        goto out;

    if (sig < SIGRTMIN && sigismember(&t->pending.signal, sig))
        goto out;

    ret = deliver_signal(sig, info, t);
out:
    spin_unlock_irqrestore(&t->sigmask_lock, flags);
    if ((t->state & TASK_INTERRUPTIBLE) && signal_pending(t))
        wake_up_process(t);

out_nolock:
    return ret;
}
```
`send_sig_info()` 首先调用 `bad_signal()` 函数来检查是否有权发送信号给进程，然后调用 `ignored_signal()` 函数来检查信号是否被忽略，接着调用 `deliver_signal()` 函数开始发送信号，最后如果进程是睡眠状态就唤醒进程。我们接着来分析 `deliver_signal()` 函数：
```cpp
static int deliver_signal(int sig, struct siginfo *info, struct task_struct *t)
{
	int retval = send_signal(sig, info, &t->pending);

	if (!retval && !sigismember(&t->blocked, sig))
		signal_wake_up(t);

	return retval;
}
```
`deliver_signal()` 首先调用 `send_signal()` 函数进行信号的发送，然后调用 `signal_wake_up()` 函数唤醒进程。我们来分析一下最重要的函数 `send_signal()`：
```cpp
static int send_signal(int sig, struct siginfo *info, struct sigpending *signals)
{
    struct sigqueue * q = NULL;

    if (atomic_read(&nr_queued_signals) < max_queued_signals) {
        q = kmem_cache_alloc(sigqueue_cachep, GFP_ATOMIC);
    }

    if (q) {
        atomic_inc(&nr_queued_signals);
        q->next = NULL;
        *signals->tail = q;
        signals->tail = &q->next;
        switch ((unsigned long) info) {
            case 0:
                q->info.si_signo = sig;
                q->info.si_errno = 0;
                q->info.si_code = SI_USER;
                q->info.si_pid = current->pid;
                q->info.si_uid = current->uid;
                break;
            case 1:
                q->info.si_signo = sig;
                q->info.si_errno = 0;
                q->info.si_code = SI_KERNEL;
                q->info.si_pid = 0;
                q->info.si_uid = 0;
                break;
            default:
                copy_siginfo(&q->info, info);
                break;
        }
    } else if (sig >= SIGRTMIN && info && (unsigned long)info != 1
           && info->si_code != SI_USER) {
        return -EAGAIN;
    }

    sigaddset(&signals->signal, sig);
    return 0;
}
```
`send_signal()` 函数虽然比较长，但逻辑还是比较简单的。在 `信号处理相关的数据结构` 一节我们介绍过进程管理结构 `task_struct` 有个 `pending` 的成员变量，其用于保存接收到的信号队列。`send_signal()` 函数的第三个参数就是进程管理结构的 `pending` 成员变量。

`send_signal()` 首先调用 `kmem_cache_alloc()` 函数来申请一个类型为 `struct sigqueue` 的队列节点，然后把节点添加到 `pending` 队列中，接着根据参数 `info` 的值来进行不同的操作，最后通过 `sigaddset()` 函数来设置信号对应的标志位，表示进程接收到该信号。

`signal_wake_up()` 函数会把进程的 `sigpending` 成员变量设置为1，表示有信号需要处理，如果进程是睡眠可中断状态还会唤醒进程。

至此，发送信号的流程已经完成，我们可以通过下面的调用链来更加直观的理解此过程：
```text
kill()   
  |                    User Space
---------------------------------------------------------------------------------------
  |                    Kernel Space
sys_kill()
  |---> kill_something_info()
               |---> kill_proc_info()
                           |---> find_task_by_pid()
                           |---> send_sig_info()
                                       |---> bad_signal()
                                       |---> handle_stop_signal()
                                       |---> ignored_signal()
                                       |---> deliver_signal()
                                                   |---> send_signal()
                                                   |          |---> kmem_cache_alloc()
                                                   |          |---> sigaddset()
                                                   |---> signal_wake_up()
```

### 内核触发信号处理函数
上面介绍了怎么发生一个信号给指定的进程，但是什么时候会触发信号相应的处理函数呢？为了尽快让信号得到处理，Linux把信号处理过程放置在从内核态返回到用户态的过程中，就是在 `ret_from_sys_call` 处：
```asm
ENTRY(ret_from_sys_call)
	...
ret_with_reschedule:
	...
	cmpl $0, sigpending(%ebx)  // 检查进程的sigpending成员是否等于1
	jne signal_return          // 如果是就跳转到 signal_return 处执行
restore_all:
	RESTORE_ALL

	ALIGN
signal_return:
	sti                             // 开启硬件中断
	testl $(VM_MASK),EFLAGS(%esp)
	movl %esp,%eax
	jne v86_signal_return
	xorl %edx,%edx
	call SYMBOL_NAME(do_signal)    // 调用do_signal()函数进行处理
	jmp restore_all
```
由于这是一段汇编代码，有点不太直观，所以我在代码中进行了注释。主要的逻辑就是首先检查进程的 `sigpending` 成员是否等于1，如果是调用 `do_signal()` 函数进行处理，由于 `do_signal()` 函数代码比较长，所以我们分段来说明，如下：
```cpp
int do_signal(struct pt_regs *regs, sigset_t *oldset)
{
	siginfo_t info;
	struct k_sigaction *ka;

	if ((regs->xcs & 3) != 3)
		return 1;

	if (!oldset)
		oldset = &current->blocked;

	for (;;) {
		unsigned long signr;

		spin_lock_irq(&current->sigmask_lock);
		signr = dequeue_signal(&current->blocked, &info);
		spin_unlock_irq(&current->sigmask_lock);

		if (!signr)
			break;

```
上面这段代码的主要逻辑是通过 `dequeue_signal()` 函数获取到进程接收队列中的一个信号，如果没有信号，那么就跳出循环。我们接着来分析：
```cpp
		ka = &current->sig->action[signr-1];
		if (ka->sa.sa_handler == SIG_IGN) {
			if (signr != SIGCHLD)
				continue;
			/* Check for SIGCHLD: it's special.  */
			while (sys_wait4(-1, NULL, WNOHANG, NULL) > 0)
				/* nothing */;
			continue;
		}
```
上面这段代码首先获取到信号对应的处理方法，如果对此信号的处理是忽略的话，那么就直接跳过。
```cpp
		if (ka->sa.sa_handler == SIG_DFL) {
			int exit_code = signr;

			/* Init gets no signals it doesn't want.  */
			if (current->pid == 1)
				continue;

			switch (signr) {
			case SIGCONT: case SIGCHLD: case SIGWINCH:
				continue;

			case SIGTSTP: case SIGTTIN: case SIGTTOU:
				if (is_orphaned_pgrp(current->pgrp))
					continue;
				/* FALLTHRU */

			case SIGSTOP:
				current->state = TASK_STOPPED;
				current->exit_code = signr;
				if (!(current->p_pptr->sig->action[SIGCHLD-1].sa.sa_flags & SA_NOCLDSTOP))
					notify_parent(current, SIGCHLD);
				schedule();
				continue;

			case SIGQUIT: case SIGILL: case SIGTRAP:
			case SIGABRT: case SIGFPE: case SIGSEGV:
			case SIGBUS: case SIGSYS: case SIGXCPU: case SIGXFSZ:
				if (do_coredump(signr, regs))
					exit_code |= 0x80;
				/* FALLTHRU */

			default:
				sigaddset(&current->pending.signal, signr);
				recalc_sigpending(current);
				current->flags |= PF_SIGNALED;
				do_exit(exit_code);
				/* NOTREACHED */
			}
		}
		...
		handle_signal(signr, ka, &info, oldset, regs);
		return 1;
	}
	...
	return 0;
}
```
上面的代码表示，如果指定为默认的处理方法，那么就使用系统的默认处理方法去处理信号，比如 `SIGSEGV` 信号的默认处理方法就是使用 `do_coredump()` 函数来生成一个 `core dump` 文件，并且通过调用 `do_exit()` 函数退出进程。

如果指定了自定义的处理方法，那么就通过 `handle_signal()` 函数去进行处理，`handle_signal()` 函数代码如下：
```cpp
static void
handle_signal(unsigned long sig, struct k_sigaction *ka,
	      siginfo_t *info, sigset_t *oldset, struct pt_regs * regs)
{
	...
	if (ka->sa.sa_flags & SA_SIGINFO)
		setup_rt_frame(sig, ka, info, oldset, regs);
	else
		setup_frame(sig, ka, oldset, regs);

	if (ka->sa.sa_flags & SA_ONESHOT)
		ka->sa.sa_handler = SIG_DFL;

	if (!(ka->sa.sa_flags & SA_NODEFER)) {
		spin_lock_irq(&current->sigmask_lock);
		sigorsets(&current->blocked,&current->blocked,&ka->sa.sa_mask);
		sigaddset(&current->blocked,sig);
		recalc_sigpending(current);
		spin_unlock_irq(&current->sigmask_lock);
	}
}
```
由于信号处理程序是由用户提供的，所以代码在用户态中。而从内核态回到用户态前还是属于内核态，内核态是不能执行用户态的函数的，那么怎么办？答案就是在用户态栈空间构建一个框架，当从内核态返回到用户态时自动触发信号处理程序。
