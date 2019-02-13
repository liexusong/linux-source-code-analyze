## 内存交换
由于计算机的物理内存是有限的, 而进程对内存的使用是不确定的, 所以物理内存总有用完的可能性. 那么当系统的物理内存不足时, Linux内核使用什么方案来避免申请不到物理内存这个问题呢?

相对于内存来说, 磁盘的容量是非常大的, 所以Linux内核实现了一个叫 `内存交换` 的功能 -- 把某些进程的一些暂时用不到的内存页保存到磁盘中, 然后把物理内存页分配给更紧急的用户使用, 当进程用到时再从磁盘读回到内存中即可. 有了 `内存交换` 功能, 系统可使用的内存就可以远远大于物理内存的容量.

### kswapd内核线程
在Linux系统启动时会调用 `kswapd_init()` 函数, 代码如下:
```cpp
static int __init kswapd_init(void)
{
    printk("Starting kswapd v1.8\n");
    swap_setup();
    kernel_thread(kswapd, NULL, CLONE_FS | CLONE_FILES | CLONE_SIGNAL);
    kernel_thread(kreclaimd, NULL, CLONE_FS | CLONE_FILES | CLONE_SIGNAL);
    return 0;
}
```
