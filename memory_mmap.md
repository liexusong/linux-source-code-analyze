## Linux mmap完全剖析

### `mmap()` 系统调用介绍
`mmap()` 系统调用能够将文件映射到内存空间，然后可以通过读写内存来读写文件。我们先来看看 `mmap()` 系统调用的用法吧，`mmap()` 函数的原型如下：
```cpp
void *mmap(void *start, size_t length, int prot, int flags, int fd, off_t offset);
```
参数说明：
* `start`：指定要映射的内存地址，一般设置为 `NULL` 让操作系统自动选择合适的内存地址。
* `length`：映射地址空间的字节数，它从被映射文件开头 `offset` 个字节开始算起。
* `prot`：指定共享内存的访问权限。可取如下几个值的或：`PROT_READ`（可读）, `PROT_WRITE`（可写）, `PROT_EXEC`（可执行）, `PROT_NONE`（不可访问）。
* `flags`：由以下几个常值指定：`MAP_SHARED `（共享的） `MAP_PRIVATE`（私有的）, `MAP_FIXED`（表示必须使用 `start` 参数作为开始地址，如果失败不进行修正），其中，`MAP_SHARED` , `MAP_PRIVATE`必选其一，而 `MAP_FIXED` 则不推荐使用。
* `fd`：表示要映射的文件句柄。
* `offset`：表示映射文件的偏移量，一般设置为 `0` 表示从文件头部开始映射。

函数的返回值为最后文件映射到进程空间的地址，进程可直接操作起始地址为该值的有效地址。

下面通过一个例子来说明 `mmap()` 系统调用的用法：
```cpp
int main() {
    ...
    fd = open(name, flag, mode);
    if (fd < 0) {
        // error process...
        exit(1);
    }
    
    addr = mmap(NULL, len, PROT_READ|PROT_WRITE|PROT_EXEC, MAP_SHARED, fd, 0);
    if (addr < 0) {
        // error process...
        exit(1);
    }

    memset(addr, 0, len);
    ...
    exit(0);
}
```
上面的例子首先通过 `open()` 系统调用打开一个文件，然后通过调用 `mmap()` 把文件映射到内存空间，映射成功后就可以通过操作函数返回的内存地址来对文件进行读写操作。

### `mmap()` 底层原理
在分析 `mmap()` 系统调用原理之前，首先要知道操作系统的虚拟内存空间与物理内存空间的概念。

在 32位的 Linux 内核中，每个进程都独有 `4GB` 的虚拟内存空间，但所有进程却共用相同的物理内存空间。物理内存空间就是安装在电脑上的内存条，如果内存条只有 `1GB`，那么物理内存空间就只有 `1GB`。但虚拟内存空间是逻辑上的内存空间，虚拟内存空间必须映射到物理内存空间才能使用。

虚拟内存空间与物理内存空间映射关系如下：
![process-vm-mapping](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/process_vm.jpg)

映射是按内存页进行的，一个内存页为 `4KB` 大小。在上图中，`P1` 是进程1，`P2` 是进程2。进程1的虚拟内存页A映射到物理内存页A，进程2的虚拟内存页A映射到物理内存页B。进程1的虚拟内存页B和进程2的虚拟内存页B同时映射到物理内存页C，也就是说进程1和进程2共享了物理内存页C。
