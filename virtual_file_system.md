# 虚拟文件系统
通常我们使用的磁盘和光盘都属于块设备，也就是说它们都是按照 `数据块` 来进行读写的，可以把磁盘和光盘想象成一个由数据块组成的巨大数组。但这样的读写方式对于人类来说不太友好，所以一般要在磁盘或者光盘上面挂载 `文件系统` 才能使用。那么什么是 `文件系统` 呢？ `文件系统` 是一种存储和组织数据的方法，它使得对其访问和查找变得容易。通过挂载文件系统后，我们可以使用如 `/home/docs/test.txt` 的方式来访问磁盘中的数据，而不用使用数据块编号来进行访问。

在Linux系统中，可以使用多种文件系统来挂载不同的设备，如 ext2、ext3、nfs等等。但提供给用户的文件处理接口是一致的，也就是说不管使用 ext2 文件系统还是使用 ext3 文件系统，处理文件的接口都是一样的。这样的好处是，用户不用关心使用了什么文件系统，只需要使用统一的方式去处理文件即可。那么Linux是如何做到的呢？这就得益于 `虚拟文件系统(Virtual File System，简称 VFS)`。

`虚拟文件系统` 为不同的文件系统定义了一套规范，各个文件系统必须按照 `虚拟文件系统的规范` 编写才能接入到 `虚拟文件系统` 中。这有点像面向对象语言里面的 `接口`，当一个类实现了某个接口的所有方法时，便可以把这个类当做成此接口。VFS 主要为用户和内核架起一道桥梁，用户可以通过 VFS 提供的接口访问不同的文件系统，如下图：
![vfs](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/vfs.jpg)

下面我们开始分析 `虚拟文件系统` 的实现原理。

## 虚拟文件系统抽象数据结构
因为要为不同类型的文件系统定义统一的接口层，所以这些文件系统必须按照 VFS 的规范来编写程序，VFS 抽象了几个数据结构来管理不同的文件系统。我们必须先了解这些数据结构的定义和作用，这样在分析源码时遇到它们也不至于一脸愕然。

### 超级块(super block)
因为Linux支持多文件系统，所以在内核中必须通过一个数据结构来描述具体文件系统的一些信息和相关的操作等，VFS 定义了一个名为 `超级块（super_block）` 的数据结构来描述具体的文件系统，也就是说内核是通过超级块来认知具体的文件系统的，其定义如下（由于super_block的成员比较多，所以这里只列出部分）：
```cpp
struct file_system_type {
    const char *name;
    int fs_flags;
    struct super_block *(*read_super) (struct super_block *, void *, int); // 读取设备中文件系统超级块的方法
    ...
};

struct super_operations {
    void (*read_inode) (struct inode *);        // 把磁盘中的inode数据读取入到内存中
    void (*write_inode) (struct inode *, int);  // 把inode的数据写入到磁盘中
    void (*put_inode) (struct inode *);         // 释放inode占用的内存
    void (*delete_inode) (struct inode *);      // 删除磁盘中的一个inode
    void (*put_super) (struct super_block *);   // 释放超级块占用的内存
    void (*write_super) (struct super_block *); // 把超级块写入到磁盘中
    ...
};

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
下面我们介绍一下一些比较重要的成员：
* s_dev：用于保存设备的设备号
* s_blocksize：用于保存文件系统的数据块大小（文件系统是以数据块为单位的）
* s_type：文件系统的类型（提供了读取设备中文件系统超级块的方法）
* s_op：超级块相关的操作列表
* s_root：挂载的根目录

### 索引节点（inode）
`索引节点（inode）` 用于描述一个文件的meta（元）信息，其包含的是诸如文件的大小、拥有者、创建时间、磁盘位置等和文件相关的信息。`inode` 的定义如下（由于inode的成员也是非常多，所以这里也只列出部分成员，具体可以参考Linux源码）：
```cpp
struct inode_operations {
    int (*create) (struct inode *,struct dentry *,int);
    struct dentry * (*lookup) (struct inode *,struct dentry *);
    int (*link) (struct dentry *,struct inode *,struct dentry *);
    int (*unlink) (struct inode *,struct dentry *);
    int (*symlink) (struct inode *,struct dentry *,const char *);
    ...
};

struct file_operations {
    struct module *owner;
    loff_t (*llseek) (struct file *, loff_t, int);
    ssize_t (*read) (struct file *, char *, size_t, loff_t *);
    ssize_t (*write) (struct file *, const char *, size_t, loff_t *);
    ...
};

struct inode {
    ...
    unsigned long       i_ino;
    atomic_t            i_count;
    kdev_t              i_dev;
    umode_t             i_mode;
    nlink_t             i_nlink;
    uid_t               i_uid;
    gid_t               i_gid;
    kdev_t              i_rdev;
    loff_t              i_size;
    time_t              i_atime;
    time_t              i_mtime;
    time_t              i_ctime;
    ...
    struct inode_operations *i_op;
    struct file_operations  *i_fop;
    struct super_block      *i_sb;
    ...
    union {
        struct minix_inode_info     minix_i;
        struct ext2_inode_info      ext2_i;
        ...
    } u;
};
```
下面也介绍一下 `inode` 中几个比较重要的成员：
* i_uid：文件所属的用户
* i_gid：文件所属的组
* i_rdev：文件所在的设备号
* i_size：文件的大小
* i_atime：文件的最后访问时间
* i_mtime：文件的最后修改时间
* i_ctime：文件的创建时间
* i_op：inode相关的操作列表
* i_fop：文件相关的操作列表
* i_sb：文件所在文件系统的超级块

我们应该重点关注 `i_op` 和 `i_fop` 这两个成员。`i_op` 成员定义对目录相关的操作方法列表，譬如 `mkdir()`系统调用会触发 `inode->i_op->mkdir()` 方法，而 `link()` 系统调用会触发 `inode->i_op->link()` 方法。而 `i_fop` 成员则定义了对打开文件后对文件的操作方法列表，譬如 `read()` 系统调用会触发 `inode->i_fop->read()` 方法，而 `write()` 系统调用会触发 `inode->i_fop->write()` 方法。

    上面的说法有点不当，因为打开文件后，对文件的操作实体是 file 结构。而打开文件时， file 结构会复制 inode 的 i_fop 成员到其 f_op 成员中，所以当调用 open() 系统调用时真实被触发的是 file->f_op->read()，但其实是一样的。

### 目录项（dentry）
目录项的主要作用是方便查找文件。一个路径的各个组成部分，不管是目录还是普通的文件，都是一个目录项对象。如，在路径 `/home/liexusong/example.c` 中，目录 `/`, `home/`, `liexusong/` 和文件 `example.c` 都对应一个目录项对象。不同于前面的两个对象，目录项对象没有对应的磁盘数据结构，VFS 在遍历路径名的过程中现场将它们逐个地解析成目录项对象。其定义如下：
```cpp
struct dentry_operations {
    int (*d_revalidate)(struct dentry *, int);
    int (*d_hash) (struct dentry *, struct qstr *);
    int (*d_compare) (struct dentry *, struct qstr *, struct qstr *);
    int (*d_delete)(struct dentry *);
    void (*d_release)(struct dentry *);
    void (*d_iput)(struct dentry *, struct inode *);
};

struct dentry {
    ...
    struct inode  * d_inode;    // 目录项对应的inode
    struct dentry * d_parent;   // 当前目录项对应的父目录
    ...
    struct qstr d_name;         // 目录的名字
    unsigned long d_time;
    struct dentry_operations  *d_op; // 目录项的辅助方法
    struct super_block * d_sb;       // 所在文件系统的超级块对象
    ...
    unsigned char d_iname[DNAME_INLINE_LEN]; // 当目录名不超过16个字符时使用
};
```
