## 什么是中断
`中断` 是为了解决外部设备完成某些工作后通知CPU的一种机制（譬如硬盘完成读写操作后通过中断告知CPU已经完成）。早期没有中断机制的计算机就不得不通过轮询来查询外部设备的状态，由于轮询是试探查询的（也就是说设备不一定是就绪状态），所以往往要做很多无用的查询，从而导致效率非常低下。由于中断是由外部设备主动通知CPU的，所以不需要CPU进行轮询去查询，效率大大提升。

从物理学的角度看，中断是一种电信号，由硬件设备产生，并直接送入中断控制器（如 8259A）的输入引脚上，然后再由中断控制器向处理器发送相应的信号。处理器一经检测到该信号，便中断自己当前正在处理的工作，转而去处理中断。此后，处理器会通知 OS 已经产生中断。这样，OS 就可以对这个中断进行适当的处理。不同的设备对应的中断不同，而每个中断都通过一个唯一的数字标识，这些值通常被称为中断请求线。

## 中断控制器
X86计算机的 CPU 为中断只提供了两条外接引脚：NMI 和 INTR。其中 NMI 是不可屏蔽中断，它通常用于电源掉电和物理存储器奇偶校验；INTR是可屏蔽中断，可以通过设置中断屏蔽位来进行中断屏蔽，它主要用于接受外部硬件的中断信号，这些信号由中断控制器传递给 CPU。

常见的中断控制器有两种：

### 可编程中断控制器8259A

传统的 PIC（Programmable Interrupt Controller，可编程中断控制器）是由两片 8259A 风格的外部芯片以“级联”的方式连接在一起。每个芯片可处理多达 8 个不同的 IRQ。因为从 PIC 的 INT 输出线连接到主 PIC 的 IRQ2 引脚，所以可用 IRQ 线的个数达到 15 个，如图下所示。

![8259A](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/8259A.png)

### 高级可编程中断控制器（APIC）

8259A 只适合单 CPU 的情况，为了充分挖掘 SMP 体系结构的并行性，能够把中断传递给系统中的每个 CPU 至关重要。基于此理由，Intel 引入了一种名为 I/O 高级可编程控制器的新组件，来替代老式的 8259A 可编程中断控制器。该组件包含两大组成部分：一是“本地 APIC”，主要负责传递中断信号到指定的处理器；举例来说，一台具有三个处理器的机器，则它必须相对的要有三个本地 APIC。另外一个重要的部分是 I/O APIC，主要是收集来自 I/O 装置的 Interrupt 信号且在当那些装置需要中断时发送信号到本地 APIC，系统中最多可拥有 8 个 I/O APIC。

每个本地 APIC 都有 32 位的寄存器，一个内部时钟，一个本地定时设备以及为本地中断保留的两条额外的 IRQ 线 LINT0 和 LINT1。所有本地 APIC 都连接到 I/O APIC，形成一个多级 APIC 系统，如图下所示。

![APIC](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/APIC.gif)

目前大部分单处理器系统都包含一个 I/O APIC 芯片，可以通过以下两种方式来对这种芯片进行配置：
* 作为一种标准的 8259A 工作方式。本地 APIC 被禁止，外部 I/O APIC 连接到 CPU，两条 LINT0 和 LINT1 分别连接到 INTR 和 NMI 引脚。
* 作为一种标准外部 I/O APIC。本地 APIC 被激活，且所有的外部中断都通过 I/O APIC 接收。

辨别一个系统是否正在使用 I/O APIC，可以在命令行输入如下命令：
```shell
# cat /proc/interrupts
           CPU0       
  0:      90504    IO-APIC-edge  timer
  1:        131    IO-APIC-edge  i8042
  8:          4    IO-APIC-edge  rtc
  9:          0    IO-APIC-level  acpi
 12:        111    IO-APIC-edge  i8042
 14:       1862    IO-APIC-edge  ide0
 15:         28    IO-APIC-edge  ide1
177:          9    IO-APIC-level  eth0
185:          0    IO-APIC-level  via82cxxx
...
```
如果输出结果中列出了 IO-APIC，说明您的系统正在使用 APIC。如果看到 XT-PIC，意味着您的系统正在使用 8259A 芯片。

## 中断分类
中断可分为同步（synchronous）中断和异步（asynchronous）中断：
* 同步中断是当指令执行时由 CPU 控制单元产生，之所以称为同步，是因为只有在一条指令执行完毕后 CPU 才会发出中断，而不是发生在代码指令执行期间，比如系统调用。
* 异步中断是指由其他硬件设备依照 CPU 时钟信号随机产生，即意味着中断能够在指令之间发生，例如键盘中断。

根据 Intel 官方资料，同步中断称为异常（exception），异步中断被称为中断（interrupt）。

中断可分为 `可屏蔽中断`（Maskable interrupt）和 `非屏蔽中断`（Nomaskable interrupt）。异常可分为 `故障`（fault）、`陷阱`（trap）、`终止`（abort）三类。

从广义上讲，中断可分为四类：`中断`、`故障`、`陷阱`、`终止`。这些类别之间的异同点请参看 表。

表：中断类别及其行为

类别|原因|异步/同步|返回行为
:-: | :-: | :-: | :-:
中断|来自I/O设备的信号|异步|总是返回到下一条指令
陷阱|有意的异常|同步|总是返回到下一条指令
故障|潜在可恢复的错误|同步|返回到当前指令
终止|不可恢复的错误|同步|不会返回

X86 体系结构的每个中断都被赋予一个唯一的编号或者向量（8 位无符号整数）。非屏蔽中断和异常向量是固定的，而可屏蔽中断向量可以通过对中断控制器的编程来改变。

> 本文来源于：[Linux 内核中断内幕](https://www.ibm.com/developerworks/cn/linux/l-cn-linuxkernelint/)
> 参考资料：[详解8259A](https://blog.csdn.net/longintchar/article/details/79439466)
