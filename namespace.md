## namespace介绍
`namespace（命名空间）` 是Linux提供的一种内核级别环境隔离的方法，很多编程语言也有 namespace 这样的功能，例如C++，Java等，编程语言的 namespace 是为了解决项目中能够在不同的命名空间里使用相同的函数名或者类名。而Linux的 namespace 也是为了实现资源能够在不同的命名空间里有相同的名称，譬如在 `A命名空间` 有个pid为1的进程，而在 `B命名空间` 中也可以有一个pid为1的进程。

有了 `namespace` 就可以实现基本的容器功能，著名的 `Docker` 也是使用了 namespace 来实现资源隔离的。

Linux支持6种资源的 `namespace`，分别为（[文档](https://lwn.net/Articles/531114/)）：

|Type              |  Parameter  |Linux Version|
|------------------|-------------|-------------|
| Mount namespaces | CLONE_NEWNS |Linux 2.4.19 |
|  UTS namespaces  | CLONE_NEWUTS|Linux 2.6.19 |
|  IPC namespaces  | CLONE_NEWIPC|Linux 2.6.19 |
|  PID namespaces  | CLONE_NEWPID|Linux 2.6.24 |
|Network namespaces| CLONE_NEWNET|Linux 2.6.24 |
| User namespaces  |CLONE_NEWUSER|Linux 2.6.23 |

在调用 `clone()` 系统调用时，传入以上的不同类型的参数就可以实现复制不同类型的namespace。比如传入 `CLONE_NEWPID` 参数时，就是复制 `pid命名空间`，在新的 `pid命名空间` 里可以使用与其他 `pid命名空间` 相同的pid。代码如下：
```cpp
#define _GNU_SOURCE
#include <sched.h>
#include <stdio.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/wait.h>
#include <signal.h>
#include <stdlib.h>
#include <errno.h>

char child_stack[5000];

int child(void* arg)
{
    printf("Child - %d\n", getpid());
    return 1;
}

int main()
{
    printf("Parent - fork child\n");
    int pid = clone(child, child_stack+5000, CLONE_NEWPID, NULL);
    if (pid == -1) {
        perror("clone:");
        exit(1);
    }
    waitpid(pid, NULL, 0);
    printf("Parent - child(%d) exit\n", pid);
    return 0;
}
```
输出如下：
```
Parent - fork child
Parent - child(9054) exit
Child - 1
```
从运行结果可以看出，在子进程的 `pid命名空间` 里当前进程的pid为1，但在父进程的 `pid命名空间` 中子进程的pid却是9045。

## namespace实现原理
为了让每个进程都可以从属于某一个namespace，Linux内核为进程描述符添加了一个 `struct nsproxy` 的结构，如下：
```cpp
struct task_struct {
    ...
    /* namespaces */
    struct nsproxy *nsproxy;
    ...
}

struct nsproxy {
    atomic_t count;
    struct uts_namespace  *uts_ns;
    struct ipc_namespace  *ipc_ns;
    struct mnt_namespace  *mnt_ns;
    struct pid_namespace  *pid_ns;
    struct user_namespace *user_ns;
    struct net            *net_ns;
};
```
