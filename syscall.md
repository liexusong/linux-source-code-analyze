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

系统初始化时，会在 `trap_init()` 函数中对 `int 0x80` 中断处理进行初始化，设置其中断处理过程入口为 `system_call`。`system_call` 是一段由汇编语言编写的代码，我们看看关键部分，如下：
```asm
ENTRY(system_call)
    ...
    call *SYMBOL_NAME(sys_call_table)(,%eax,4)
    movl %eax,EAX(%esp)     # save the return value
    ...
```

我们把上面的汇编改写成 C 代码如下：

```c
void system_call()
{
    ...
    // 变量 eax 代表 eax 寄存器的值
    syscall = sys_call_table[eax];
    eax = syscall();
    ...
}
```

`sys_call_table` 变量是一个数组，数组的每一个元素代表一个 `系统调用` 的入口，其定义如下（在文件 arch/i386/kernel/entry.S 中）：

```asm
.data
ENTRY(sys_call_table)
    .long SYMBOL_NAME(sys_ni_syscall)
    .long SYMBOL_NAME(sys_exit)
    .long SYMBOL_NAME(sys_fork)
    .long SYMBOL_NAME(sys_read)
    .long SYMBOL_NAME(sys_write)
    .long SYMBOL_NAME(sys_open)
    .long SYMBOL_NAME(sys_close)
    ...
```

翻译成 C 代码如下：

```c
long sys_call_table[] = {
   sys_ni_syscall,
   sys_exit,
   sys_fork,
   sys_read,
   sys_write,
   sys_open,
   sys_close,
   ...
};
```

用户调用 `系统调用` 时，通过向 `eax` 寄存器写入要调用的 `系统调用` 编号，这个编号就是 `sys_call_table` 数组的下标。 `system_call` 过程获取 `eax` 寄存器的值，然后通过 `eax` 寄存器的值找到要调用的 `系统调用` 入口，并且进行调用。调用完成后，`系统调用` 会把返回值保存到 `eax` 寄存器中。

原理如下图（图片来源 https://developer.ibm.com/zh/technologies/linux/tutorials/l-system-calls/ ）：

![system_call](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/system_call.gif)

## 三、系统调用实现

当用户要调用 `系统调用` 时，需要通过向 `eax` 寄存器写入要调用的 `系统调用` 编号。因为 `用户态` 和 `内核态` 使用的栈不同，而调用 `系统调用` 是在用户态调用的，而进入 `系统调用` 后会变成内核态，所以参数就不能通过栈来传递。Linux 使用寄存器来传递参数，参数与寄存器的关系如下：

* 第1个参数放置在 `ebx` 寄存器。
* 第2个参数放置在 `ecx` 寄存器。
* 第3个参数放置在 `edx` 寄存器。
* 第4个参数放置在 `esi` 寄存器。
* 第5个参数放置在 `edi` 寄存器。
* 第6个参数放置在 `ebp` 寄存器。

而 Linux 进入中断处理程序时，会把这些寄存器的值保存到内核栈中，这样 `系统调用` 就能通过内核栈来获取到参数。

下面我们通过 `sys_open()` 系统调用来说明一下 `系统调用` 的运作方式，`sys_open()` 实现如下：

```c
asmlinkage long sys_open(const char *filename, int flags, int mode)
{
    ...
}
```

一般 `系统调用` 都需要使用 `asmlinkage` 编译选项，`asmlinkage` 编译选项是告诉编译器从栈中读取参数，其实际是封装了 GCC 的编译选项，如下：

```c
#define asmlinkage CPP_ASMLINKAGE __attribute__((regparm(0)))
```

`__attribute__((regparm(0)))` 就是告诉 GCC 所有参数都从栈中读取，而 Linux 进入中断处理上下文时，会把 `ebx`、`ecx`、`edx`、`esi`、`edi`、`ebp` 寄存器的值保存到内核栈中，那么 `系统调用` 就可以从内核栈获取到参数的值。

但由于寄存器只能传递 32 位的整型值（x86 CPU），所以参数一般只能传递指针或者整型的数值，如果要获取指针对应结构的数据，就必须通过从用户空间复制到内核空间，如 `sys_open()` 系统调用获取要打开的文件路径：

```c
asmlinkage long sys_open(const char *filename, int flags, int mode)
{
    char * tmp;
    ...
    tmp = getname(filename);
    ...
}
```

`getname()` 函数就是用于从用户空间复制数据到内核空间。
