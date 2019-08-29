# 虚拟文件系统
通常我们使用的磁盘和光盘都属于块设备，也就是说它们都是按照 `数据块` 来进行读写的，可以把磁盘和光盘想象成一个由数据块组成的巨大数组。但这样的读写方式对于人类来说不太友好，所以一般要在磁盘或者光盘上面挂载 `文件系统` 才能使用。那么什么是 `文件系统` 呢？ `文件系统` 是一种存储和组织数据的方法，它使得对其访问和查找变得容易。通过挂载文件系统后，我们可以使用如 `/home/docs/test.txt` 的方式来访问磁盘中的数据，而不用使用数据块编号来进行访问。

在Linux系统中，可以使用多种文件系统来挂载不同的设备，如 ext2、ext3、nfs等等。但提供给用户的文件处理接口是一致的，也就是说不管使用 ext2 文件系统还是使用 ext3 文件系统，处理文件的接口都是一样的。这样的好处是，用户不用关心使用了什么文件系统，只需要使用统一的方式去处理文件即可。那么Linux是如何做到的呢？这就得益于 `虚拟文件系统(Virtual File System，简称 VFS)`。

`虚拟文件系统` 为不同的文件系统定义了一套规范，各个文件系统必须按照 `虚拟文件系统的规范` 编写才能接入到 `虚拟文件系统` 中。这有点像面向对象语言里面的 `接口`，当一个类实现了某个接口的所有方法时，便可以把这个类当做成此接口。VFS 主要为用户和内核架起一道桥梁，用户可以通过 VFS 提供的接口访问不同的文件系统，如下图：
![vfs](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/vfs.jpg)

下面我们开始分析 `虚拟文件系统` 的实现原理。

## 虚拟文件系统抽象数据结构
Linux奉行了Unix的理念：`一切皆文件`，比如一个目录是一个文件，一个设备也是一个文件等，因而文件系统在Linux中占有非常重要的地位。

因为要为不同类型的文件系统定义统一的接口层，所以 VFS 定义了一系列的规范，真实的文件系统必现按照 VFS 的规范来编写程序。VFS 抽象了几个数据结构来组织和管理不同的文件系统，分别为：`超级块（super_block）`、`索引节点（inode）`、`目录结构（dentry）` 和 `文件结构（file）`，要理解 VFS 就必须先了解这些数据结构的定义和作用。

### 超级块(super block)
因为Linux支持多文件系统，所以在内核中必须通过一个数据结构来描述具体文件系统的信息和相关的操作等，VFS 定义了一个名为 `超级块（super_block）` 的数据结构来描述具体的文件系统，也就是说内核是通过超级块来认知具体的文件系统的，一个具体的文件系统会对应一个超级块结构，其定义如下（由于super_block的成员比较多，所以这里只列出部分）：
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
`索引节点（inode）` 是 VFS 中最为重要的一个结构，用于描述一个文件的meta（元）信息，其包含的是诸如文件的大小、拥有者、创建时间、磁盘位置等和文件相关的信息，所有文件都有一个对应的 `inode` 结构。`inode` 的定义如下（由于inode的成员也是非常多，所以这里也只列出部分成员，具体可以参考Linux源码）：
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

### 文件结构（file）
文件结构用于描述一个已打开的文件，其包含文件当前的读写偏移量，文件打开模式和文件操作函数列表等，文件结构定义如下：
```cpp
struct file {
    struct list_head         f_list;
    struct dentry           *f_dentry;  // 文件所属的dentry结构
    struct file_operations  *f_op;      // 文件的操作列表
    atomic_t                 f_count;   // 计数器（表示有多少个用户打开此文件）
    unsigned int             f_flags;   // 标识位  
    mode_t                   f_mode;    // 打开模式
    loff_t                   f_pos;     // 读写偏移量
    unsigned long            f_reada, f_ramax, f_raend, f_ralen, f_rawin;
    struct fown_struct       f_owner;   // 所属者信息
    unsigned int             f_uid, f_gid;  // 打开的用户id和组id
    int                      f_error;
    unsigned long            f_version;

    /* needed for tty driver, and maybe others */
    void                    *private_data;
};
```

下图展示了各个数据结构之间的关系：
![vfs-struct](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/vfs-struct.jpg)

## 虚拟文件系统的实现
接下来我们分析一下虚拟文件系统的实现。

### 注册文件系统
Linux为了支持不同的文件系统而创造了虚拟文件系统，虚拟文件系统更像一个规范(或者说接口)，真实的文件系统需要实现虚拟文件系统的规范(接口)才能接入到Linux内核中。

要让Linux内核能够发现真实的文件系统，那么必须先使用 `register_filesystem()` 函数注册文件系统，`register_filesystem()` 函数实现如下：
```cpp
int register_filesystem(struct file_system_type * fs)
{
    struct file_system_type ** tmp;

    if (!fs)
        return -EINVAL;
    if (fs->next)
        return -EBUSY;
    tmp = &file_systems;
    while (*tmp) {
        if (strcmp((*tmp)->name, fs->name) == 0)
            return -EBUSY;
        tmp = &(*tmp)->next;
    }
    *tmp = fs;
    return 0;
}
```
`register_filesystem()` 函数的实现很简单，就是把类型为 `struct file_system_type` 的 `fs` 添加到 `file_systems` 全局链表中。`struct file_system_type` 结构的定义如下：
```cpp
struct file_system_type {
    const char *name;
    int fs_flags;
    struct super_block *(*read_super) (struct super_block *, void *, int);
    struct file_system_type * next;
};
```
其中比较重要的字段是 `read_super`，用于读取文件系统的超级块结构。在Linux初始化时会注册各种文件系统，比如 `ext2` 文件系统会调用 `register_filesystem(&ext2_fs_type)` 来注册。

当安装Linux系统时，需要把磁盘格式化为指定的文件系统，其实格式化就是把文件系统超级块信息写入到磁盘中。但Linux系统启动时，就会遍历所有注册过的文件系统，然后调用其 `read_super()` 接口来尝试读取超级块信息，因为每种文件系统的超级块都有不同的魔数，用于识别不同的文件系统，所以当调用 `read_super()` 接口返回成功时，表示读取超级块成功，而且识别出磁盘所使用的文件系统。具体过程可以通过 `mount_root()` 函数得知：
```cpp
void __init mount_root(void)
{
    ...

    memset(&filp, 0, sizeof(filp));
    d_inode = get_empty_inode(); // 获取一个新的inode
    d_inode->i_rdev = ROOT_DEV;
    filp.f_dentry = NULL;
    if ( root_mountflags & MS_RDONLY)
        filp.f_mode = 1; /* read only */
    else
        filp.f_mode = 3; /* read write */
    retval = blkdev_open(d_inode, &filp);
    if (retval == -EROFS) {
        root_mountflags |= MS_RDONLY;
        filp.f_mode = 1;
        retval = blkdev_open(d_inode, &filp);
    }

    iput(d_inode);

    if (retval)
        printk("VFS: Cannot open root device %s\n", kdevname(ROOT_DEV));
    else {
        for (fs_type = file_systems ; fs_type ; fs_type = fs_type->next) { // 试探性读取超级块
            if (!(fs_type->fs_flags & FS_REQUIRES_DEV))
                continue;
            sb = read_super(ROOT_DEV,fs_type->name,root_mountflags,NULL,1); // 读取超级块
            if (sb) {
                sb->s_flags = root_mountflags;
                current->fs->root = dget(sb->s_root);  // 设置根目录
                current->fs->pwd = dget(sb->s_root);   // 设置当前工作目录
                vfsmnt = add_vfsmnt(sb, "/dev/root", "/");
                if (vfsmnt)
                    return;
            }
        }
    }
}
```
在上面的for循环中，遍历了所有已注册的文件系统，并且调用其 `read_super()` 接口来尝试读取超级块信息，如果成功表示磁盘所使用的文件系统就是当前文件系统。成功读取超级块信息后，会把根目录的 `dentry` 结构保存到当前进程的 `root` 和 `pwd` 字段中，`root` 表示根目录，`pwd` 表示当前工作目录。

### 打开文件
要使用一个文件前必须打开文件，打开文件使用 `open()` 系统调用来实现，而 `open()` 系统调用最终会调用内核的 `sys_open()` 函数，`sys_open()` 函数实现如下：
```cpp
asmlinkage long sys_open(const char * filename, int flags, int mode)
{
    char * tmp;
    int fd, error;

    tmp = getname(filename);
    fd = PTR_ERR(tmp);
    if (!IS_ERR(tmp)) {
        fd = get_unused_fd();
        if (fd >= 0) {
            struct file * f;
            lock_kernel();
            f = filp_open(tmp, flags, mode);
            unlock_kernel();
            error = PTR_ERR(f);
            if (IS_ERR(f))
                goto out_error;
            fd_install(fd, f);
        }
out:
        putname(tmp);
    }
    return fd;

out_error:
    put_unused_fd(fd);
    fd = error;
    goto out;
}
```
`sys_open()` 函数的主要流程是：
* 通过调用 `get_unused_fd()` 函数获取一个空闲的文件描述符。
* 调用 `filp_open()` 函数打开文件，返回打开文件的file结构。
* 调用 `fd_install()` 函数把文件描述符与file结构关联起来。
* 返回文件描述符，也就是 `open()` 系统调用的返回值。

在上面的过程中，最重要的是调用 `filp_open()` 函数打开文件，`filp_open()` 函数的实现如下：
```cpp
struct file *filp_open(const char * filename, int flags, int mode)
{
    struct inode * inode;
    struct dentry * dentry;
    struct file * f;
    int flag,error;

    error = -ENFILE;
    f = get_empty_filp(); // 获取一个空闲的file结构
    ...
    dentry = open_namei(filename,flag,mode); // 通过文件路径打开文件
    inode = dentry->d_inode;
    ...
    f->f_dentry = dentry; // 设置file结构的f_dentry字段为打开文件的dentry结构
    f->f_pos = 0;
    f->f_reada = 0;
    f->f_op = NULL;
    if (inode->i_op)
        f->f_op = inode->i_op->default_file_ops; // 把文件的操作函数列表从inode结构复制到file结构中
    if (inode->i_sb)
        file_move(f, &inode->i_sb->s_files);
    if (f->f_op && f->f_op->open) {
        error = f->f_op->open(inode,f);
        if (error)
            goto cleanup_all;
    }
    f->f_flags &= ~(O_CREAT | O_EXCL | O_NOCTTY | O_TRUNC);

    return f;
    ...
}
```
`filp_open()` 函数首先调用 `get_empty_filp()` 函数获取一个空闲的file结构，然后调用 `open_namei()` 函数来打开对应路径的文件。`open_namei()` 函数会返回一个 `dentry结构`，就是对应文件路径的 `dentry结构`。所以 `open_namei()` 函数才是打开文件的核心函数，其实现如下：
```cpp
struct dentry * open_namei(const char * pathname, int flag, int mode)
{
    int acc_mode, error;
    struct inode *inode;
    struct dentry *dentry;

    mode &= S_IALLUGO & ~current->fs->umask;
    mode |= S_IFREG;

    dentry = lookup_dentry(pathname, NULL, lookup_flags(flag)); // 通过路径一级一级的查找对应的dentry
    if (IS_ERR(dentry))
        return dentry;

    acc_mode = ACC_MODE(flag);
    if (flag & O_CREAT) {  // 如果是创建文件
        struct dentry *dir;

        if (dentry->d_inode) {
            if (!(flag & O_EXCL))
                goto nocreate;
            error = -EEXIST;  // 已经存在返回错误
            goto exit;
        }

        ...

        if (dentry->d_inode) {
            error = 0;
            if (flag & O_EXCL)
                error = -EEXIST;
        } else { // 如果文件不存在
            error = vfs_create(dir->d_inode, dentry,mode); // 创建文件
            ...
        }
        ...
    }

    ...

    return dentry;
}
```
上面的代码去掉了很多权限验证的代码，`open_namei()` 函数首先会调用 `lookup_dentry()` 函数打开文件并获得文件打开后的 `dentry结构`，如果文件不存在并且打开文件的时候设置了 `O_CREAT` 标志位，那么就调用 `vfs_create()` 函数创建文件。我们先来看看 `vfs_create()` 函数的实现：
```cpp
int vfs_create(struct inode *dir, struct dentry *dentry, int mode)
{
    int error;

    error = may_create(dir, dentry);
    if (error)
        goto exit_lock;

    error = -EACCES;    /* shouldn't it be ENOSYS? */
    if (!dir->i_op || !dir->i_op->create)
        goto exit_lock;

    DQUOT_INIT(dir);
    error = dir->i_op->create(dir, dentry, mode);
exit_lock:
    return error;
}
```
从 `vfs_create()` 函数的实现可知，最终会调用 `inode结构` 的 `create()` 方法来创建文件。这个方法由真实的文件系统提供，所以真实文件系统只需要把创建文件的方法挂载到 `inode结构` 上即可，虚拟文件系统不需要知道真实文件系统的实现过程，这就是虚拟文件系统可以支持多种文件系统的真正原因。

而 `lookup_dentry()` 函数最终会调用 `real_lookup()` 函数来逐级目录查找并打开。`real_lookup()` 函数代码如下：
```cpp
static struct dentry * real_lookup(struct dentry * parent, struct qstr * name, int flags)
{
    struct dentry * result;
    struct inode *dir = parent->d_inode;

    down(&dir->i_sem);
    result = d_lookup(parent, name);
    if (!result) {
        struct dentry * dentry = d_alloc(parent, name);
        result = ERR_PTR(-ENOMEM);
        if (dentry) {
            result = dir->i_op->lookup(dir, dentry);
            if (result)
                dput(dentry);
            else
                result = dentry;
        }
        up(&dir->i_sem);
        return result;
    }
    up(&dir->i_sem);
    if (result->d_op && result->d_op->d_revalidate)
        result->d_op->d_revalidate(result, flags);
    return result;
}
```
参数 `parent` 是父目录的 `dentry结构`，而参数 `name` 是要打开的目录或者文件的名称。`real_lookup()` 函数最终也会调用父目录的 `inode结构` 的 `lookup()` 方法来查找并打开文件，然后返回打开后的子目录或者文件的 `dentry结构`。`lookup()` 方法需要把要打开的目录或者文件的 `inode结构` 从磁盘中读入到内存中（如果目录或者文件存在的话），并且把其 `inode结构` 保存到 `dentry结构` 的 `d_inode` 字段中。

`filp_open()` 函数会把 `inode结构` 的文件操作函数列表复制到 `file结构` 中，如下：
```cpp
struct file *filp_open(const char * filename, int flags, int mode)
{
    ...
    f->f_op = inode->i_op->default_file_ops;
    ...
}
```
这样，`file结构` 就有操作文件的函数列表。

### 读写文件
读取文件内容通过 `read()` 系统调用完成，而 `read()` 系统调用最终会调用 `sys_read()` 内核函数，`sys_read()` 内核函数的实现如下：
```cpp
asmlinkage ssize_t sys_read(unsigned int fd, char * buf, size_t count)
{
    ssize_t ret;
    struct file * file;

    ret = -EBADF;
    file = fget(fd);
    if (file) {
        if (file->f_mode & FMODE_READ) {
            ret = locks_verify_area(FLOCK_VERIFY_READ, file->f_dentry->d_inode,
                        file, file->f_pos, count);
            if (!ret) {
                ssize_t (*read)(struct file *, char *, size_t, loff_t *);
                ret = -EINVAL;
                if (file->f_op && (read = file->f_op->read) != NULL)
                    ret = read(file, buf, count, &file->f_pos);
            }
        }
        fput(file);
    }
    return ret;
}
```
`sys_read()` 函数首先会调用 `fget()` 函数把文件描述符转换成 `file结构`，然后再通过调用 `file结构` 的 `read()` 方法来读取文件内容，`read()` 方法是由真实文件系统提供的，所以最终的过程会根据不同的文件系统而进行不同的操作，比如ext2文件系统最终会调用 `generic_file_read()` 函数来读取文件的内容。

把内容写入到文件是通过调用 `write()` 系统调用实现，而 `write()` 系统调用最终会调用 `sys_write()` 内核函数，`sys_write()` 函数的实现如下：
```cpp
asmlinkage ssize_t sys_write(unsigned int fd, const char * buf, size_t count)
{
    ssize_t ret;
    struct file * file;

    ret = -EBADF;
    file = fget(fd);
    if (file) {
        if (file->f_mode & FMODE_WRITE) {
            struct inode *inode = file->f_dentry->d_inode;
            ret = locks_verify_area(FLOCK_VERIFY_WRITE, inode, file,
                file->f_pos, count);
            if (!ret) {
                ssize_t (*write)(struct file *, const char *, size_t, loff_t *);
                ret = -EINVAL;
                if (file->f_op && (write = file->f_op->write) != NULL)
                    ret = write(file, buf, count, &file->f_pos);
            }
        }
        fput(file);
    }
    return ret;
}
```
`sys_write()` 函数的实现与 `sys_read()` 类似，首先会调用 `fget()` 函数把文件描述符转换成 `file结构`，然后再通过调用 `file结构` 的 `write()` 方法来把内容写入到文件中，对于ext2文件系统，`write()` 方法对应的是 `ext2_file_write()` 函数。
