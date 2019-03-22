# 虚拟文件系统
通常我们使用的磁盘和光盘都属于块设备，也就是说它们都是按照 `数据块` 来进行读写的，可以把磁盘和光盘想象成一个由数据块组成的巨大数组。但这样的读写方式对于人类来说不太友好，所以一般要在磁盘或者光盘上面挂载 `文件系统` 才能使用。那么什么是 `文件系统` 呢？ `文件系统` 是一种存储和组织数据的方法，它使得对其访问和查找变得容易。通过挂载文件系统后，我们可以使用如 `/home/docs/test.txt` 的方式来访问磁盘中的数据，而不用使用数据块编号来进行访问。

在Linux系统中，可以使用多种文件系统来挂载不同的设备，如 ext2、ext3、nfs等等。但提供给用户的文件处理接口是一致的，也就是说不管使用 ext2 文件系统还是使用 ext3 文件系统，处理文件的接口都是一样的。这样的好处是，用户不用关心使用了什么文件系统，只需要使用统一的方式去处理文件即可。那么Linux是如何做到的呢？这就得益于 `虚拟文件系统(Virtual File System，简称 VFS)`。

`虚拟文件系统` 为不同的文件系统定义了一套规范，各个文件系统必须按照 `虚拟文件系统的规范` 编写才能接入到 `虚拟文件系统` 中。这有点像面向对象语言里面的 `接口`，当一个类实现了某个接口的所有方法时，便可以把这个类当做成此接口。VFS 主要为用户和内核架起一道桥梁，用户可以通过 VFS 提供的接口访问不同的文件系统，如下图：
![vfs](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/vfs.jpg)

下面我们开始分析 `虚拟文件系统` 的实现原理。

## 重要的数据结构
因为要为不同类型的文件系统定义统一的接口层，所以这些文件系统必须按照 VFS 的规范来编写程序，下面先来介绍一下在 VFS 中用于管理文件系统的数据结构。

### 超级块(super block)
因为Linux支持多文件系统，所以在内核中必须通过一个数据结构来描述具体文件系统的一些信息和相关的操作等，VFS 定义了一个名为 `超级块（super_block）` 的数据结构来描述具体的文件系统，其定义如下（由于super_block的成员比较多，所以这里只列出部分）：
```cpp
struct super_block {
    struct list_head    s_list;     /* Keep this first */
    kdev_t              s_dev;         // 设备号
    unsigned long       s_blocksize;   // 数据块大小
    unsigned char       s_blocksize_bits;
    unsigned char       s_lock;
    unsigned char       s_dirt;       // 是否脏
    struct file_system_type *s_type;  // 文件系统类型
    struct super_operations *s_op;    // 超级块相关的操作列表
    struct dquot_operations *dq_op;
    unsigned long       s_flags;
    unsigned long       s_magic;
    struct dentry       *s_root;      // 挂载的根目录
    wait_queue_head_t   s_wait;

    struct list_head    s_dirty;    /* dirty inodes */
    struct list_head    s_files;

    struct block_device *s_bdev;
    struct list_head    s_mounts;
    struct quota_mount_options s_dquot;

    union {
        struct minix_sb_info    minix_sb;
        struct ext2_sb_info ext2_sb;
        ...
    } u;
    ...
};
```
