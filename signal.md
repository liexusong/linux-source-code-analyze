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
上面介绍了怎么发生一个信号给指定的进程，但是什么时候会触发信号相应的处理函数呢？为了尽快让信号得到处理，Linux把信号处理过程放置在进程从内核态返回到用户态前，也就是在 `ret_from_sys_call` 处：
```asm
// arch/i386/kernel/entry.S

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
由于这是一段汇编代码，有点不太直观（大概知道意思就可以了），所以我在代码中进行了注释。主要的逻辑就是首先检查进程的 `sigpending` 成员是否等于1，如果是调用 `do_signal()` 函数进行处理，由于 `do_signal()` 函数代码比较长，所以我们分段来说明，如下：
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
由于信号处理程序是由用户提供的，所以信号处理程序的代码是在用户态的。而从系统调用返回到用户态前还是属于内核态，CPU是禁止内核态执行用户态代码的，那么怎么办？

答案先返回到用户态执行信号处理程序，执行完信号处理程序后再返回到内核态，再在内核态完成收尾工作。听起来有点绕，事实也的确是这样。下面通过一副图片来直观的展示这个过程（图片来源网络）：

![signal](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/signal1.png)

为了达到这个目的，Linux经历了一个十分崎岖的过程。我们知道，从内核态返回到用户态时，CPU要从内核栈中找到返回到用户态的地址（就是调用系统调用的下一条代码指令地址），Linux为了先让信号处理程序执行，所以就需要把这个返回地址修改为信号处理程序的入口，这样当从系统调用返回到用户态时，就可以执行信号处理程序了。

所以，`handle_signal()` 调用了 `setup_frame()` 函数来构建这个过程的运行环境（其实就是修改内核栈和用户栈相应的数据来完成）。我们先来看看内核栈的内存布局图：

![signal-kernel-stack](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/signal-kernel-stack.png)

图中的 `eip` 就是内核态返回到用户态后开始执行的第一条指令地址，所以把 `eip` 改成信号处理程序的地址就可以在内核态返回到用户态的时候自动执行信号处理程序了。我们看看 `setup_frame()` 函数其中有一行代码就是修改 `eip` 的值，如下：
```cpp
static void setup_frame(int sig, struct k_sigaction *ka,
			sigset_t *set, struct pt_regs * regs)
{
    ...
    regs->eip = (unsigned long) ka->sa.sa_handler; // regs是内核栈中保存的寄存器集合
    ...
}
```
现在可以在内核态返回到用户态时自动执行信号处理程序了，但是当信号处理程序执行完怎么返回到内核态呢？Linux的做法就是在用户态栈空间构建一个 `Frame（帧）`（我也不知道为什么要这样叫），构建这个帧的目的就是为了执行完信号处理程序后返回到内核态，并恢复原来内核栈的内容。返回到内核态的方式是调用一个名为 `sigreturn()` 系统调用，然后再 `sigreturn()` 中恢复原来内核栈的内容。

怎样能在执行完信号处理程序后调用 `sigreturn()` 系统调用呢？其实跟前面修改内核栈 `eip` 的值一样，这里修改的是用户栈 `eip` 的值，修改后跳转到一个执行下面代码的地方（用户栈的某一处）：
```asm
popl %eax 
movl $__NR_sigreturn,%eax 
int $0x80
```
从上面的汇编代码可以知道，这里就是调用了 `sigreturn()` 系统调用。修改用户栈的代码在 `setup_frame()` 中，代码如下：
```cpp
static void setup_frame(int sig, struct k_sigaction *ka,
			sigset_t *set, struct pt_regs * regs)
{
	...
		err |= __put_user(frame->retcode, &frame->pretcode);
		/* This is popl %eax ; movl $,%eax ; int $0x80 */
		err |= __put_user(0xb858, (short *)(frame->retcode+0));
		err |= __put_user(__NR_sigreturn, (int *)(frame->retcode+2));
		err |= __put_user(0x80cd, (short *)(frame->retcode+6));
	...
}
```
这几行代码比较难懂，其实就是修改信号程序程序返回后要执行代码的地址。修改后如下图：

![signal-user-stack](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/signal-user-stack.png)

这样执行完信号处理程序后就会调用 `sigreturn()`，而 `sigreturn()` 要做的工作就是恢复原来内核栈的内容了，我们来看看 `sigreturn()` 的代码：
```cpp
asmlinkage int sys_sigreturn(unsigned long __unused)
{
	struct pt_regs *regs = (struct pt_regs *) &__unused;
	struct sigframe *frame = (struct sigframe *)(regs->esp - 8);
	sigset_t set;
	int eax;

	if (verify_area(VERIFY_READ, frame, sizeof(*frame)))
		goto badframe;
	if (__get_user(set.sig[0], &frame->sc.oldmask)
	    || (_NSIG_WORDS > 1
		&& __copy_from_user(&set.sig[1], &frame->extramask,
				    sizeof(frame->extramask))))
		goto badframe;

	sigdelsetmask(&set, ~_BLOCKABLE);
	spin_lock_irq(&current->sigmask_lock);
	current->blocked = set;
	recalc_sigpending(current);
	spin_unlock_irq(&current->sigmask_lock);

	if (restore_sigcontext(regs, &frame->sc, &eax))
		goto badframe;
	return eax;

badframe:
	force_sig(SIGSEGV, current);
	return 0;
}
```
其中最重要的是调用 `restore_sigcontext()` 恢复原来内核栈的内容，要恢复原来内核栈的内容首先是要指定原来内核栈的内容，所以先要保存原来内核栈的内容。保存原来内核栈的内容也是在 `setup_frame()` 函数中，`setup_frame()` 函数把原来内核栈的内容保存到用户栈中（也就是上面所说的 `帧` 中）。`restore_sigcontext()` 函数就是从用户栈中读取原来内核栈的数据，然后恢复之。保存内核栈内容主要由 `setup_sigcontext()` 函数完成，有兴趣可以查阅代码，这里就不做详细说明了。

这样，当从 `sigreturn()` 系统调用返回时，就可以按原来的路径返回到用户程序的下一个执行点（比如调用系统调用的下一行代码）。

### 设置信号处理程序
最后我们来分析一下怎么设置一个信号处理程序。

用户可以通过 `signal()` 系统调用设置一个信号处理程序，我们来看看 `signal()` 系统调用的代码：
```cpp
asmlinkage unsigned long
sys_signal(int sig, __sighandler_t handler)
{
	struct k_sigaction new_sa, old_sa;
	int ret;

	new_sa.sa.sa_handler = handler;
	new_sa.sa.sa_flags = SA_ONESHOT | SA_NOMASK;

	ret = do_sigaction(sig, &new_sa, &old_sa);

	return ret ? ret : (unsigned long)old_sa.sa.sa_handler;
}
```
代码比较简单，就是先设置一个新的 `struct k_sigaction` 结构，把其 `sa.sa_handler` 字段设置为用户自定义的处理程序。然后通过 `do_sigaction()` 函数进行设置，代码如下：
```cpp
int
do_sigaction(int sig, const struct k_sigaction *act, struct k_sigaction *oact)
{
    struct k_sigaction *k;

    if (sig < 1 || sig > _NSIG ||
        (act && (sig == SIGKILL || sig == SIGSTOP)))
        return -EINVAL;

    k = &current->sig->action[sig-1];

    spin_lock(&current->sig->siglock);

    if (oact)
        *oact = *k;

    if (act) {
        *k = *act;
        sigdelsetmask(&k->sa.sa_mask, sigmask(SIGKILL) | sigmask(SIGSTOP));
        
        if (k->sa.sa_handler == SIG_IGN
            || (k->sa.sa_handler == SIG_DFL
            && (sig == SIGCONT ||
                sig == SIGCHLD ||
                sig == SIGWINCH))) {
            spin_lock_irq(&current->sigmask_lock);
            if (rm_sig_from_queue(sig, current))
                recalc_sigpending(current);
            spin_unlock_irq(&current->sigmask_lock);
        }
    }

    spin_unlock(&current->sig->siglock);
    return 0;
}
```
这个函数也不难，我们上面介绍过，进程管理结构中有个 `sig` 的字段，它是一个 `struct k_sigaction` 结构的数组，每个元素保存着对应信号的处理程序，所以 `do_sigaction()` 函数就是修改这个信号处理程序。代码 `k = &current->sig->action[sig-1]` 就是获取对应信号的处理程序，然后把其设置为新的信号处理程序即可。
