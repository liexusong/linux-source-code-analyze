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

#### `vm_area_struct` 结构
在Linux内核中，虚拟内存是用过结构体 `vm_area_struct` 来管理的，通过 `vm_area_struct` 结构体可以把虚拟内存划分为多个用途不相同的内存区，比如可以划分为数据段区、代码段区等等，`vm_area_struct` 结构体的定义如下：

```cpp
struct vm_area_struct {
    struct mm_struct * vm_mm;   /* The address space we belong to. */
    unsigned long vm_start;     /* Our start address within vm_mm. */
    unsigned long vm_end;       /* The first byte after our end address
                       within vm_mm. */

    /* linked list of VM areas per task, sorted by address */
    struct vm_area_struct *vm_next;

    pgprot_t vm_page_prot;      /* Access permissions of this VMA. */
    unsigned long vm_flags;     /* Flags, listed below. */

    rb_node_t vm_rb;

    struct vm_area_struct *vm_next_share;
    struct vm_area_struct **vm_pprev_share;

    /* Function pointers to deal with this struct. */
    struct vm_operations_struct * vm_ops;
    ...
    struct file * vm_file;       /* File we map to (can be NULL). */
    ...
};
```

`vm_area_struct` 结构各个字段作用：
* `vm_mm`：指向进程内存空间管理对象。
* `vm_start`：内存区的开始地址。
* `vm_end`：内存区的结束地址。
* `vm_next`：用于连接进程的所有内存区。
* `vm_page_prot`：指定内存区的访问权限。
* `vm_flags`：内存区的一些标志。
* `vm_file`：指向映射的文件对象。
* `vm_ops`：内存区的一些操作函数。

`vm_area_struct` 结构与虚拟内存地址的关系如下图：

![vm_address](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/vm_address.png)

每个进程都由 `task_struct` 结构进行管理，`task_struct` 结构中的 `mm` 成员指向了每个进程的内存管理结构 `mm_struct`，而 `mm_struct` 结构的 `mmap` 成员记录了进程虚拟内存空间各个内存区的 `vm_area_struct` 结构链表。

当调用 `mmap()` 时，内核会创建一个 `vm_area_struct` 结构，并且把 `vm_start` 和 `vm_end` 指向虚拟内存空间的某个内存区，并且把 `vm_file` 字段指向映射的文件对象。然后调用文件对象的 `mmap` 接口来对 `vm_area_struct` 结构的 `vm_ops` 成员进行初始化，如 `ext2` 文件系统的文件对象会调用 `generic_file_mmap()` 函数进行初始化，代码如下：

```cpp
static struct vm_operations_struct generic_file_vm_ops = {
    nopage:     filemap_nopage,
};

int generic_file_mmap(struct file * file, struct vm_area_struct * vma)
{
    struct address_space *mapping = file->f_dentry->d_inode->i_mapping;
    struct inode *inode = mapping->host;

    if ((vma->vm_flags & VM_SHARED) && (vma->vm_flags & VM_MAYWRITE)) {
        if (!mapping->a_ops->writepage)
            return -EINVAL;
    }
    if (!mapping->a_ops->readpage)
        return -ENOEXEC;
    UPDATE_ATIME(inode);
    vma->vm_ops = &generic_file_vm_ops;
    return 0;
}
```

`vm_operations_struct` 结构的 `nopage` 接口会在访问内存发生异常时被调用，上面指向的是 `filemap_nopage()` 函数，`filemap_nopage()` 函数的主要工作是：
1. 把映射到虚拟内存区的文件内容读入到物理内存页中。
2. 对访问发生异常的虚拟内存地址与物理内存地址进行映射。

处理过程如下图：

![vma-pma-maping](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/vma-pma-maping.png)

如上图所示，`虚拟内存页m` 映射到 `物理内存页x`，并且把映射的文件的内容读入到物理内存中，这样就把内存与文件的映射关系建立起来，对映射的内存区进行读写操作实际上就是对文件的读写操作。

一般来说，对映射的内存空间进行读写并不会实时写入到文件中，所以要对内存与文件进行同步时需要调用 `msync()` 函数来实现。

#### 对文件的读写
像 `read()/write()` 这些系统调用，首先需要进行内核空间，然后把文件内容读入到缓存中，然后再对缓存进行读写操作，最后由内核定时同步到文件中。过程如下图：

![read-write-system-call](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/read-write-system-call.png)

而调用 `mmap()` 系统调用对文件进行映射后，用户对映射后的内存进行读写实际上是对文件缓存的读写，所以减少了一次系统调用，从而加速了对文件读写的效率。如下图：

![mmap-memory-address](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/mmap-memory-address.png)


