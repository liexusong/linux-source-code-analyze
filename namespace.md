## namespace介绍
`namespace（命名空间）` 是Linux提供的一种内核级别环境隔离的方法，很多编程语言也有 namespace 这样的功能，例如C++，Java等，编程语言的 namespace 是为了解决项目中能够在不同的命名空间里使用相同的函数名或者类名。而Linux的 namespace 也是为了实现资源能够在不同的命名空间里有相同的名称，譬如在 `A命名空间` 有个pid为1的进程，而在 `B命名空间` 中也可以有一个pid为1的进程。

有了 `namespace` 就可以实现基本的容器功能，著名的 `Docker` 也是使用了 namespace 来实现资源隔离的。

Linux支持6种资源的 `namespace`，分别为（[文档](https://lwn.net/Articles/531114/)）：

|Type              |  Parameter  |Linux Version|
-------------------|-------------|-------------|
| Mount namespaces | CLONE_NEWNS |Linux 2.4.19 |
|  UTS namespaces  | CLONE_NEWUTS|Linux 2.6.19 |
|  IPC namespaces  | CLONE_NEWIPC|Linux 2.6.19 |
|  PID namespaces  | CLONE_NEWPID|Linux 2.6.24 |
|Network namespaces| CLONE_NEWNET|Linux 2.6.24 |
| User namespaces  |CLONE_NEWUSER|Linux 2.6.23 |

