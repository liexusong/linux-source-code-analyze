## 虚拟内存管理
在Linux系统中, 用户进程使用的内存地址叫虚拟内存地址, 虚拟内存地址必须通过 `MMU(Memory Management Unit)` 转换成物理内存地址才能被CPU使用. 而转换过程需要借助一个名叫 `页表` 的数据结构来实现 (学过数据结构的同学应该接触过Hash表, Hash表也是一种映射关系). 

可能有人会问, 为什么进程不使用物理地址, 而要多此一举来将虚拟地址映射成物理地址. 这是因为通过这种映射机制, 每个进程都可以使用4GB的虚拟地址, 而不受物理地址大小的影响. 另外, 虚拟地址除了可以映射到物理地址外, 还可以映射到其他一些设备(如文件)中, 这样就可以像读写内存一样操作这些设备.

那么Linux内核是怎样通过虚拟内存地址映射到物理内存地址的呢? X86 CPU把虚拟内存地址分为3个部分, 如下图:
![enter image description here](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/memory_map.jpg)
