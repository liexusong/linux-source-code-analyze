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
从 `struct nsproxy` 结构的定义可以看出，Linux为每种不同类型的资源定义了不同的命名空间结构体进行管理。比如对于 `pid命名空间` 定义了 `struct pid_namespace` 结构来管理 。由于 namespace 涉及的资源种类比较多，所以本文主要以 `pid命名空间` 作为分析的对象。

我们先来看看管理 `pid命名空间` 的 `struct pid_namespace` 结构的定义：
```cpp
struct pid_namespace {
    struct kref kref;
    struct pidmap pidmap[PIDMAP_ENTRIES];
    int last_pid;
    struct task_struct *child_reaper;
    struct kmem_cache *pid_cachep;
    unsigned int level;
    struct pid_namespace *parent;
#ifdef CONFIG_PROC_FS
    struct vfsmount *proc_mnt;
#endif
};
```
因为 `struct pid_namespace` 结构主要用于为当前 `pid命名空间` 分配空闲的pid，所以定义比较简单：
* `kref` 成员是一个引用计数器，用于记录引用这个结构的进程数
* `pidmap` 成员用于快速找到可用pid的位图
* `last_pid` 成员是记录最后一个可用的pid
* `level` 成员记录当前 `pid命名空间` 所在的层次
* `parent` 成员记录当前 `pid命名空间` 的父命名空间

由于 `pid命名空间` 是分层的，也就是说新创建一个 `pid命名空间` 时会记录父级 `pid命名空间` 到 `parent` 字段中，所以随着 `pid命名空间` 的创建，在内核中会形成一颗 `pid命名空间` 的树，如下图（[图片来源](http://www.zhongruitech.com/256011226.html)）：

![pid-namespace](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/pid-namespace-level.png)

第0层的 `pid命名空间` 是 `init` 进程所在的命名空间。如果一个进程所在的 `pid命名空间` 为 `N`，那么其在 `0 ~ N 层pid命名空间` 都有一个唯一的pid号。也就是说 `高层pid命名空间` 的进程对 `低层pid命名空间` 的进程是可见的，但是 `低层pid命名空间` 的进程对 `高层pid命名空间` 的进程是不可见的。

由于在 `第N层pid命名空间` 的进程其在 `0 ~ N层pid命名空间` 都有一个唯一的pid号，所以在进程描述符中通过 `pids` 成员来记录其在每个层的pid号，代码如下：
```cpp
struct task_struct {
    ...
    struct pid_link pids[PIDTYPE_MAX];
    ...
}

enum pid_type {
    PIDTYPE_PID,
    PIDTYPE_PGID,
    PIDTYPE_SID,
    PIDTYPE_MAX
};

struct upid {
    int nr;
    struct pid_namespace *ns;
    struct hlist_node pid_chain;
};

struct pid {
    atomic_t count;
    struct hlist_head tasks[PIDTYPE_MAX];
    struct rcu_head rcu;
    unsigned int level;
    struct upid numbers[1];
};

struct pid_link {
    struct hlist_node node;
    struct pid *pid;
};
```
这几个结构的关系如下图：

![pid-namespace-structs](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/pid-namespace-structs.png)

我们主要关注 `struct pid` 这个结构，`struct pid` 有个类型为 `struct upid` 的成员 `numbers`，其定义为只有一个元素的数组，但是其实是一个动态的数据，它的元素个数与 `level` 的值一致，也就是说当 `level` 的值为5时，那么 `numbers` 成员就是一个拥有5个元素的数组。而每个元素记录了其在每层 `pid命名空间` 的pid号，而 `struct upid` 结构的 `nr` 成员就是用于记录进程在不同层级 `pid命名空间` 的pid号。

我们通过代码来看看怎么为进程分配pid号的，在内核中是用过 `alloc_pid()` 函数分配pid号的，代码如下：
```cpp
struct pid *alloc_pid(struct pid_namespace *ns)
{
    struct pid *pid;
    enum pid_type type;
    int i, nr;
    struct pid_namespace *tmp;
    struct upid *upid;

    pid = kmem_cache_alloc(ns->pid_cachep, GFP_KERNEL);
    if (!pid)
        goto out;

    tmp = ns;
    for (i = ns->level; i >= 0; i--) {
        nr = alloc_pidmap(tmp);    // 为当前进程所在的不同层级pid命名空间分配一个pid
        if (nr < 0)
            goto out_free;

        pid->numbers[i].nr = nr;   // 对应i层namespace中的pid数字
        pid->numbers[i].ns = tmp;  // 对应i层namespace的实体
        tmp = tmp->parent;
    }

    get_pid_ns(ns);
    pid->level = ns->level;
    atomic_set(&pid->count, 1);
    for (type = 0; type < PIDTYPE_MAX; ++type)
        INIT_HLIST_HEAD(&pid->tasks[type]);

    spin_lock_irq(&pidmap_lock);
    for (i = ns->level; i >= 0; i--) {
        upid = &pid->numbers[i];
        // 把upid连接到全局pid中, 用于快速查找pid
        hlist_add_head_rcu(&upid->pid_chain,
                &pid_hash[pid_hashfn(upid->nr, upid->ns)]);
    }
    spin_unlock_irq(&pidmap_lock);

out:
    return pid;

    ...
}
```
上面的代码中，那个 `for (i = ns->level; i >= 0; i--)` 就是通过 `parent` 成员不断向上检索为不同层级的 `pid命名空间` 分配一个唯一的pid号，并且保存到对应的 `nr` 字段中。另外，还会把进程所在各个层级的pid号添加到全局pid哈希表中，这样做是为了通过pid号快速找到进程。

现在我们来看看怎么通过pid号快速找到一个进程，在内核中 `find_get_pid()` 函数用来通过pid号查找对应的 `struct pid` 结构，代码如下（find_get_pid() -> find_vpid() -> find_pid_ns()）：
```cpp
struct pid *find_get_pid(pid_t nr)
{
    struct pid *pid;

    rcu_read_lock();
    pid = get_pid(find_vpid(nr));
    rcu_read_unlock();

    return pid;
}

struct pid *find_vpid(int nr)
{
    return find_pid_ns(nr, current->nsproxy->pid_ns);
}

struct pid *find_pid_ns(int nr, struct pid_namespace *ns)
{
    struct hlist_node *elem;
    struct upid *pnr;

    hlist_for_each_entry_rcu(pnr, elem,
            &pid_hash[pid_hashfn(nr, ns)], pid_chain)
        if (pnr->nr == nr && pnr->ns == ns)
            return container_of(pnr, struct pid,
                    numbers[ns->level]);

    return NULL;
}
```
通过pid号查找 `struct pid` 结构时，首先会把进程pid号和当前进程的 `pid命名空间` 传入到 `find_pid_ns()` 函数，而在 `find_pid_ns()` 函数中通过全局pid哈希表来快速查找对应的 `struct pid` 结构。获取到 `struct pid` 结构后就可以很容易地获取到进程对应的进程描述符，例如可以通过 `pid_task()` 函数来获取 `struct pid` 结构对应进程描述符，由于代码比较简单，这里就不再分析了。
