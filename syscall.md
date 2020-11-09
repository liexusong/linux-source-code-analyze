# 系统调用

## 一、什么是系统调用

`系统调用` 跟用户自定义函数一样也是一个函数，不同的是 `系统调用` 运行在内核态，而用户自定义函数运行在用户态。由于某些指令（如设置时钟、关闭/打开中断和I/O操作等）只能运行在内核态，所以操作系统必须提供一种能够进入内核态的方式，`系统调用` 就是这样的一种机制。

`系统调用` 是 Linux 内核提供的一段代码（函数），其实现了一些特定的功能，用户可以通过 `int 0x80` 中断（x86 CPU）或者 `syscall` 指令（x64 CPU）来调用 `系统调用`。

## 二、进入系统调用

> 本文主要介绍的是 x86 CPU 进入系统调用的方式

Linux 提供了 `int 0x80` 中断来让用户程序进入 `系统调用`，我们来看看 Linux 对 `int 0x80` 中断的处理初始化过程：

```c
void __init trap_init(void)
{
    ...
    set_system_gate(SYSCALL_VECTOR, &system_call);
    ...
}
```

系统初始化时，会在 `trap_init()` 函数中对 `int 0x80` 中断进行初始化，设置其中断处理过程入口为 `system_call`。`system_call` 是一段由汇编语言编写的代码，如下：
```asm
ENTRY(system_call)
    pushl %eax          # save orig_eax
    SAVE_ALL
    GET_CURRENT(%ebx)
    testb $0x02,tsk_ptrace(%ebx)    # PT_TRACESYS
    jne tracesys
    cmpl $(NR_syscalls),%eax
    jae badsys
    call *SYMBOL_NAME(sys_call_table)(,%eax,4)
    movl %eax,EAX(%esp)     # save the return value
ENTRY(ret_from_sys_call)
    cli             # need_resched and signals atomic test
    cmpl $0,need_resched(%ebx)
    jne reschedule
    cmpl $0,sigpending(%ebx)
    jne signal_return
restore_all:
    RESTORE_ALL
```
