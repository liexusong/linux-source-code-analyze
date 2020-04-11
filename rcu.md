## RCU原理与实现

Linux内核有多种锁机制，比如 `自旋锁`、`信号量` 和 `读写锁` 等。不同的场景使用不同的锁，如在读多写少的场景可以使用读写锁，而在锁粒度比较小的场景可以使用自旋锁。

本文主要介绍一种比较有趣的锁，名为：`RCU`，`RCU` 是 `Read Copy Update` 这几个单词的缩写，中文翻译是 `读 复制 更新`，顾名思义这个锁只需要三个步骤就能完成：__1) 读__、 __2) 复制__、 __3) 更新__。但是往往现实并不是那么美好的，这个锁机制要比这个名字复杂很多。

我们先来介绍一下 `RCU` 的使用场景，`RCU` 的特点是：多个 `reader（读者）` 可以同时读取共享的数据，而 `updater（更新者）` 更新共享的数据时需要复制一份，然后对副本进行修改，修改完把原来的共享数据替换成新的副本，而对旧数据的销毁（释放）等待到所有读者都不再引用旧数据时进行。

### RCU 原理

分析下面代码存在的问题（例子参考：《深入理解并行编程》）：
```cpp
struct foo {
    int a;
    char b;
    long c;
 };

DEFINE_SPINLOCK(foo_mutex);

struct foo *gbl_foo;

void foo_read(void)
{
    foo *fp = gbl_foo;
    if (fp != NULL)
        dosomething(fp->a, fp->b, fp->c);
}

void foo_update(foo* new_fp)
{
    spin_lock(&foo_mutex);
    foo *old_fp = gbl_foo;
    gbl_foo = new_fp;
    spin_unlock(&foo_mutex);
    free(old_fp);
}
```
假如有线程A和线程B同时执行 `foo_read()`，而另线程C执行 `foo_update()`，那么会出现以下情况：
1) 线程A和线程B同时读取到旧的 `gbl_foo` 的指针。
2) 线程A和线程B同时读取到新的 `gbl_foo` 的指针。
3) 线程A和线程B有一个读取到新的 `gbl_foo` 的指针，另外一个读取到旧的 `gbl_foo` 的指针。

如果线程A或线程B在读取旧的 `gbl_foo` 数据还没完成时，线程C释放了旧的 `gbl_foo` 指针，那么将会导致程序奔溃。

为了解决这个问题，`RCU` 提出 `宽限期` 的概念。

`宽限期` 是指线程引用旧数据结束前的一段时间，如下图（图片来源：[RCU原理分析](https://www.cnblogs.com/chaozhu/p/6265740.html)）：

![rcu-grace-period](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/rcu-grace-period.png)

如上图所示，线程1、线程2和线程5在删除（替换）旧数据前已经在使用旧数据，所以必须等待它们不再引用旧数据时才能对旧数据进行销毁，这个等待的时间就是 `宽限期`。由于线程3、线程4和线程6使用的是新数据（已经被替换成新的指针），所以不需要等到它们。

> 由于 `RCU` 的读者需要禁止抢占，所以对于 `RCU` 来说，`宽限期` 是所有CPU都进行一次用户态调度的时间。

上面的这段话是什么意思？

禁止抢占代表CPU不能调度到其他线程，CPU只能等待当前线程离开临界区（不再引用旧数据）才能进行调度。也就是说，如果CPU进行了一次调度，说明线程已经不再引用旧数据。如果所有CPU都进行了一次调度，就说明已经没有线程引用旧数据，那么就可以对旧数据进行销毁。

### RCU 使用

#### RCU 读者

要做Linux内核中使用 `RCU`，读者需要使用 `rcu_read_lock()` 来对临界区进行 “上锁”，本质上 `rcu_read_lock()` 就是禁止CPU进行抢占，如下代码：
```cpp
#define rcu_read_lock()     preempt_disable()  // 禁止抢占
```

当不再引用数据时，需要使用 `rcu_read_unlock()` 对临界区进行解锁，本质上 `rcu_read_unlock()` 就是开启抢占，如下代码：
```cpp
#define rcu_read_unlock()   preempt_enable()  // 开启抢占
```

#### RCU 更新者

对于更新者，有两种方式：
1. 调用 `call_rcu()` 异步销毁。
2. 调用 `synchronize_kernel()` 同步销毁。
