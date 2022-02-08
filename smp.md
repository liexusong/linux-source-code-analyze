## 1. SMP 硬件体系结构：
对于 SMP 最简单可以理解为系统存在多个完全相同的 CPU ，所有 CPU 共享总线，拥有自己的寄存器。对于内存和外部设备访问，由于共享总线，所以是共享的。 Linux 操作系统多个 CPU 共享在系统空间上映射相同，是完全对等的。

由于系统中存在多个 CPU ，这是就引入一个问题，当外部设备产生中断的时候，具体有哪一个 CPU 进行处理？

为此， intel 公司提出了 IO APCI 和 LOCAL APCI 的体系结构。

IO APIC 连接各个外部设备，并可以设置分发类型，根据设定的分发类型，中断信号发送的对应 CPU 的 LOCAL APIC上。

LOCAL APIC 负责本地 CPU 的中断处理， LOCAL APIC 不仅可以接受 IO APIC 的中断，也需要处理本地 CPU 产生的异常。同时 LOCAL APIC 还提供了一个定时器。

### 如何确定那个 CPU 是引导 CPU ？
根据 intel 公司中的资料，系统上电后，会根据 MP Initialization Protocol 随机选择一个 CPU 作为 BSP ，只有 BSP 会运行BIOS 程序，其他 AP 都进入等待状态， BSP 发送 IPI 中断触发后才可以运行。具体的 MP Initialization Protocol 细节，可以参考 Intel® 64 and IA-32 Architectures Software Developer’s Manual Volume 3A: System Programming
Guide, Part 1 第 8 章。

### 引导 CPU 如何控制其他 CPU 开始运行？
BSP 可以通过 IPI 消息控制 AP 从指定的起始地址运行。 CPU 中集成的 LOCAL APIC 提供了这个功能。可以通过写LOCAL APIC 中提供的相关寄存器，发送 IPI 消息到指定的 CPU 上。

### 如何获取系统硬件 CPU 信息的？
在系统初始化后，硬件会在内存的规定位置提供关于 CPU ，总线 , IO APIC 等的信息，即 SMP MP table 。在 linux 初始化的过程，会读取该位置，获取系统相关的硬件信息。

## 2. linux SMP 启动过程流程简介
```c
start_kernel
  |-> setup_arch()
  |      |-> setup_memory();
  |      |      |-> reserve_bootmem(PAGE_SIZE, PAGE_SIZE);
  |      |-> find_smp_config();  // 查找 smp_mp_table 的位置
  |      |-> smp_alloc_memory();
  |      |       |-> trampoline_base = (void *) alloc_bootmem_low_pages(PAGE_SIZE); // 分配 trampoline ，用于启动 AP 的引导代码。
  |      |-> get_smp_config();  // 根据 smp_mp_table ，获取具体的硬件信息
  |-> trap_init()
  |      |-> init_apic_mappings();
  |-> mem_init()
  |      |-> zap_low_mappings(); // 如果没有定义 SMP 的话，清楚用户空间的地址映射。
  |-> rest_init();
  |      |-> kernel_thread(init, NULL, CLONE_FS | CLONE_SIGHAND);
  |      |       |-> init();
  
  |-> set_cpus_allowed(current, CPU_MASK_ALL);
  |-> smp_prepare_cpus(max_cpus);
  |       |-> smp_boot_cpus(max_cpus);
  |       |-> connect_bsp_APIC();
  |-> setup_local_APIC(); // 初始化 BSP 的 LOCAL APCI 。
  |-> map_cpu_to_logical_apicid();
  |-> 针对每个 CPU 调用 do_boot_cpu(apicid, cpu)
  |-> smp_init(); // 每个 CPU 开始进行调度
```

trampoline.S AP 引导代码，为 16 进制代码，启用保护模式

head.s 为 AP 创建分页管理

initialize_secondary 根据之前 fork 创建设置的信息，跳转到 start_secondary 处

start_secondary 判断 BSP 是否启动，如果启动 AP 进行任务调度。

## 3. 代码学习总结
find_smp_config() 查找 MP table 在内存中的位置。具体协议可以参考 MP 协议的第 4 章。

这个表的作用在于描述系统 CPU ，总线， IO APIC 等的硬件信息。

相关的两个全局变量： smp_found_config 是否找到 SMP MP table ， mpf_found SMP MP table 的线性地址。

* smp_alloc_memory() 为启动 AP 的启动程序分配内存空间。相关全局变量 trampoline_base ，分配的启动地址的线性地址。
* get_smp_config() 根据 MP table 中提供的内容，获取硬件的信息。
* init_apic_mappings(); 获取 IO APIC 和 LOCAL APIC 的映射地址 。
* zap_low_mappings(); 如果没有定义 SMP 的话，清楚用户空间的地址映射。将 swapper_pg_dir 中表项清零。
* setup_local_APIC();  初始化 BSP 的 LOCAL APCI 。

```c
do_boot_cpu(apicid, cpu)
 |-> idle = alloc_idle_task(cpu);
 |-> task = copy_process(CLONE_VM, 0, idle_regs(&regs), 0, NULL, NULL, 0);
 |-> init_idle(task, cpu);
```

将 init 进程使用 copy_process 复制，并且调用 init_idle 函数，设置可以运行的 CPU 。
```c
idle->thread.eip = (unsigned long) start_secondary;
```

修改 task_struct 中的 thread.eip ，使得 AP 初始化完成后，就运行 start_secondary 函数。
```c
start_eip = setup_trampoline();
```

调用 setup_trampoline() 函数，复制 trampoline_data 到 trampoline_end 之间的代码到 trampoline_base 处， trampoline_base 就是之前在 setup_arch 处申请的内存。 start_eip 返回值是 trampoline_base 对应的物理地址。

smpboot_setup_warm_reset_vector(start_eip); 设置内存 40:67h 处为 start_eip 为启动地址。

wakeup_secondary_cpu(apicid, start_eip); 在这个函数中通过操作 APIC_ICR 寄存器， BSP 向目标 AP 发送 IPI 消息，触发目标 AP 从 start_eip 地址处，从实模式开始运行。
```asm
trampoline.S

       ENTRY(trampoline_data)
r_base = .
       wbinvd               # Needed for NUMA-Q should be harmless for others
       mov %cs, %ax         # Code and data in the same place
       mov %ax, %ds

       cli                  # We should be safe anyway

       movl       $0xA5A5A5A5, trampoline_data - r_base
```

这个是设置标识，以便 BSP 知道 AP 运行到这里了。
```asm
       lidtl boot_idt - r_base    # load idt with 0, 0
       lgdtl boot_gdt - r_base    # load gdt with whatever is appropriate
```

加载 ldt 和 gdt

```asm
       xor  %ax, %ax
       inc   %ax        # protected mode (PE) bit
       lmsw       %ax        # into protected mode

       # flush prefetch and jump to startup_32_smp in arch/i386/kernel/head.S
       ljmpl       $__BOOT_CS, $(startup_32_smp-__PAGE_OFFSET)
```

启动保护模式，跳转到 startup_32_smp 处
```asm
       # These need to be in the same 64K segment as the above;
       # hence we don't use the boot_gdt_descr defined in head.S

boot_gdt:
       .word      __BOOT_DS + 7                 # gdt limit
       .long       boot_gdt_table-__PAGE_OFFSET # gdt base

boot_idt:
       .word       0                         # idt limit = 0
       .long       0                         # idt base = 0L

.globl trampoline_end

trampoline_end:
```

在这段代码中，设置标识，以便 BSP 知道该 AP 已经运行到这段代码，加载 GDT 和 LDT 表基址。

然后启动保护模式，跳转到 startup_32_smp 处。


head.s 部分代码：
```c
ENTRY(startup_32_smp)
       cld
       movl $(__BOOT_DS),%eax
       movl %eax,%ds
       movl %eax,%es
       movl %eax,%fs
       movl %eax,%gs

       xorl %ebx,%ebx
       incl %ebx
```

如果是 AP 的话，将 bx 设置为 1

```c
       movl $swapper_pg_dir-__PAGE_OFFSET,%eax
       movl %eax,%cr3           /* set the page table pointer.. */
       movl %cr0,%eax
       orl $0x80000000,%eax
       movl %eax,%cr0           /* ..and set paging (PG) bit */
       ljmp $__BOOT_CS,$1f /* Clear prefetch and normalize %eip */
```

启用分页，
```c
       lss stack_start,%esp
```

使 esp 执行 fork 创建的进程内核堆栈部分，以便后续跳转到 start_secondary


```asm
#ifdef CONFIG_SMP
       movb ready, %cl
       movb $1, ready
       cmpb $0,%cl

       je 1f               # the first CPU calls start_kernel
                           # all other CPUs call initialize_secondary

       call initialize_secondary
       jmp L6

1:

#endif /* CONFIG_SMP */
       call start_kernel
```

如果是 AP 启动的话，就调用 initialize_secondary 函数。
```c
void __devinit initialize_secondary(void)
{
       /*
        * We don't actually need to load the full TSS,
        * basically just the stack pointer and the eip.
        */
       asm volatile(
              "movl %0,%%esp/n/t"
              "jmp *%1"
              :
              :"r" (current->thread.esp),"r" (current->thread.eip));
}
```
设置堆栈为 fork 创建时的堆栈， ip 为 fork 时的 ip ，这样就跳转的了 start_secondary 。

start_secondary 函数中处理如下：
```c
       while (!cpu_isset(smp_processor_id(), smp_commenced_mask))
           rep_nop();
```

进行 smp_commenced_mask 判断，是否启动 AP 运行。 smp_commenced_mask 在 smp_init() 中设置。
```c
       cpu_idle();
```
如果启动了，调用 cpu_idle 进行任务调度。
