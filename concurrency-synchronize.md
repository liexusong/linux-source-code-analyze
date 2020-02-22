## 并发同步

`并发` 是指在某一时间段内能够处理多个任务的能力，而 `并行` 是指同一时间能够处理多个任务的能力。并发和并行看起来很像，但实际上是有区别的，如下图（图片来源于网络）：

![concurrency-parallelism](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/concurrency-synchronize-1.png)

上图的意思是，有两条在排队买咖啡的队列，并发只有一架咖啡机在处理，而并行就有两架的咖啡机在处理。咖啡机的数量越多，并行能力就越强。

可以把上面的两条队列看成两个进程，并发就是指只有单个CPU在处理，而并行就有两个CPU在处理。为了让两个进程在单核CPU中也能得到执行，一般的做法就是让每个进程交替执行一段时间，比如让每个进程固定执行 `100毫秒`，执行时间使用完后切换到其他进程执行。而并行就没有这种问题，因为有两个CPU，所以两个进程可以同时执行。如下图：

![concurrency-parallelism](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/concurrency-synchronize-2.png)

### 原子操作

上面介绍过，并发有可能会打断当前执行的进程，然后替切换成其他进程执行。如果有两个进程同时对一个共享变量 `count` 进行加一操作，由于C语言的 `count++` 操作会被翻译成如下指令：
```asm
mov eax, [count]
inc eax
mov [count], eax
```
那么在并发的情况下，有可能出现如下问题：

![concurrency-problem](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/concurrency-synchronize-4.jpg)

假设count变量初始值为0：
* 进程1执行完 `mov eax, [count]` 后，寄存器eax内保存了count的值0。
* 进程2被调度执行。进程2执行 `count++` 的所有指令，将累加后的count值1写回到内存。
* 进程1再次被调度执行，计算count的累加值仍为1，写回到内存。

虽然进程1和进程2执行了两次 `count++` 操作，但是count最后的值为1，而不是2。

要解决这个问题就需要使用 `原子操作`，原子操作是指不能被打断的操作，在单核CPU中，一条指令就是原子操作。比如上面的问题可以把 `count++` 语句翻译成指令 `inc [count]` 即可。Linux也提供了这样的原子操作，如对整数加一操作的 `atomic_inc()`：
```cpp
static __inline__ void atomic_inc(atomic_t *v)
{
	__asm__ __volatile__(
		LOCK "incl %0"
		:"=m" (v->counter)
		:"m" (v->counter));
}
```

在多核CPU中，一条指令也不一定是原子操作，比如 `inc [count]` 指令在多核CPU中需要进行如下过程：
1. 从内存将count的数据读取到cpu。
2. 累加读取的值。
3. 将修改的值写回count内存。

`Intel x86 CPU` 提供了 `lock` 前缀来锁住总线，可以让指令保证不被其他CPU中断，如下：
```asm
lock
inc [count]
```

### 锁

`原子操作` 能够保证操作不被其他进程干扰，但有时候一个复杂的操作需要由多条指令来实现，那么就不能使用原子操作了，这时候可以使用 `锁` 来实现。
