# 虚拟文件系统
Linux支持多种文件系统，但提供给用户的文件操作接口却是一样的（`open()`、`read()`、`write()` ...），那么Linux是如何实现的呢？这就得益于 `虚拟文件系统(Virtual File System, 简称VFS)` 了。虚拟文件系统是在软件层实现的一个中间层，它屏蔽了底层文件系统的实现细节，对用户提供统一的接口来访问真实的文件系统。譬如说当用户使用 `open()` 系统调用打开一个文件时，如果文件所在的文件系统是 `ext2`，真正调用的就是 `ext2_open_file()`。如果文件所在的文件系统是 `nfs`，那么真正调用的就是 `nfs_open()`。

## 重要的数据结构
在介绍虚拟文件系统实现之前，我们先来介绍几个重要的数据结构。

### 超级块(super block)
因为Linux支持多种文件系统，所以内核必须通过一个数据结构来管理不同的文件系统，此结构称为 `超级块(super block)`，每种不同的文件系统都需要通过一个 `超级块` 管理其信息和操作，`超级块` 的定义如下：
```cpp
// include/linux/fs.h

struct super_block {
	struct list_head	s_list;		/* Keep this first */
	kdev_t			s_dev;
	unsigned long		s_blocksize;
	unsigned char		s_blocksize_bits;
	unsigned char		s_lock;
	unsigned char		s_dirt;
	struct file_system_type	*s_type;
	struct super_operations	*s_op;
	struct dquot_operations	*dq_op;
	unsigned long		s_flags;
	unsigned long		s_magic;
	struct dentry		*s_root;
	wait_queue_head_t	s_wait;

	struct list_head	s_dirty;	    /* dirty inodes */
	struct list_head	s_files;

	struct block_device	*s_bdev;
	struct list_head	s_mounts;	    /* vfsmount(s) of this one */
	struct quota_mount_options s_dquot;	/* Diskquota specific options */

	union {
		struct minix_sb_info	minix_sb;
		struct ext2_sb_info	ext2_sb;
		...
	} u;

	struct semaphore s_vfs_rename_sem;	/* Kludge */

	struct semaphore s_nfsd_free_path_sem;
};
```
由于 `super_block` 结构的成员比较多，所以这里就不一一解释每个成员的作用（读者也没兴趣），这里就挑几个比较常用的来解释一下吧：
* s_list: Linux会把系统中所有文件系统的 `super_block` 结构连接到一个全局链表 `super_blocks` 中。
* s_dev: 当前已挂载的文件系统所属的设备号。
* s_blocksize: 文件系统中每个数据块的大小（Linux中的文件系统把磁盘划分为数据块来管理，一般一个数据块为4KB）。
* ...
* s_type: 这个成员比较重要，因为其定义了应该怎么读取文件系统超级块的数据。
* ...

我们接着分析一下 `s_type` 成员的类型 `file_system_type`:
```cpp
// include/linux/fs.h

struct file_system_type {
	const char *name;
	int fs_flags;
	struct super_block *(*read_super) (struct super_block *, void *, int); // 读取文件系统超级块的方法
	struct module *owner;
	struct vfsmount *kern_mnt; /* For kernel mount, if it's FS_SINGLE fs */
	struct file_system_type * next;
};
```
其中成员 `read_super` 是一个函数指针，其指向具体文件系统应该怎么读取超级块数据，譬如 `ext2` 文件系统的 `read_super` 指向的是 `ext2_read_super` 方法。

### 目录项(dentry)
在Linux内核中，每个目录都对应一个 `dentry` 结构，用于描述此目录的信息，定义如下：
```cpp
// include/linux/dcache.h

struct dentry {
	atomic_t d_count;
	unsigned int     d_flags;
	struct inode  *  d_inode;	/* Where the name belongs to - NULL is negative */
	struct dentry *  d_parent;	/* parent directory */
	struct list_head d_vfsmnt;
	struct list_head d_hash;	/* lookup hash list */
	struct list_head d_lru;		/* d_count = 0 LRU list */
	struct list_head d_child;	/* child of parent list */
	struct list_head d_subdirs;	/* our children */
	struct list_head d_alias;	/* inode alias list */
	struct qstr d_name;
	unsigned long d_time;		/* used by d_revalidate */
	struct dentry_operations  *d_op;
	struct super_block * d_sb;	/* The root of the dentry tree */
	unsigned long d_reftime;	/* last time referenced */
	void * d_fsdata;		/* fs-specific data */
	unsigned char d_iname[DNAME_INLINE_LEN]; /* small names */
};
```
下面简单介绍一下其成员的作用：
* d_count: 用于保存当前目录有多少个引用。
* d_flags: 目录的标志位。
* d_inode: 指向目录或者文件的inode结构(下面会介绍)。
* d_parent: 目录的父目录（上一级目录）。
* ...
* d_name: 目录的名字。
* ...

在 `dentry` 结构的所有成员中，最重要的是类型为 `inode` 的 `d_inode` 这个成员，下面我们介绍一下 `inode` 这个结构。

### 索引节点(inode)
`索引节点(inode)` 的作用是保存一个目录或文件的信息，譬如修改时间、所属用户和用户组，最重要的是其定义了操作一个目录或者文件的方法集，其定义如下：
```cpp
// include/linux/fs.h

struct inode {
	struct list_head	i_hash;
	struct list_head	i_list;
	struct list_head	i_dentry;
	...
	uid_t			i_uid;
	gid_t			i_gid;
	kdev_t			i_rdev;
	loff_t			i_size;
	time_t			i_atime;
	time_t			i_mtime;
	time_t			i_ctime;
	unsigned long		i_blksize;
	unsigned long		i_blocks;
	unsigned long		i_version;
	struct semaphore	i_sem;
	struct semaphore	i_zombie;
	struct inode_operations	*i_op;
	struct file_operations	*i_fop;	/* former ->i_op->default_file_ops */
	struct super_block	*i_sb;
	wait_queue_head_t	i_wait;
	struct file_lock	*i_flock;
	struct address_space	*i_mapping;
	...
	union {
		struct minix_inode_info		minix_i;
		struct ext2_inode_info		ext2_i;
		...
	} u;
};
```
由于 `inode` 结构的定义比较庞大，所以这里只截取了部分成员。下面简单介绍一下其中一些成员的作用：
* i_uid: 文件或目录所属的用户
* i_gid: 文件或目录所属的用户组
* i_atime: 文件或目录最后被访问的时间
* i_fop: 文件或目录对应的操作方法集
* i_sb: 文件或目录所属的文件系统超级块
* ...

有同学可能会问，为什么已经有 `dentry` 结构了还需要定义一个 `inode` 结构？这是因为在Linux中，不同的路径可以指向同一个文件，譬如硬连接。所以说 `dentry` 只能表示一个路径，而 `inode` 才能代表一个真实目录或者文件，我们看到 `inode` 结构中有个 `i_dentry` 的链表就是用于保存指向同一个文件或者目录的所有 `dentry`。

我们通过一张图片来描述一下这三个数据结构的关系：
![vfs structs](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/vfs_structs.jpg)

因为 `inode` 结构中有文件的操作方法集，所以找到了文件对应的 `inode`，那么内核就知道如何处理文件。我们看看 `inode` 的 `i_fop` 成员定义：
```cpp
struct file_operations {
	struct module *owner;
	loff_t (*llseek) (struct file *, loff_t, int);
	ssize_t (*read) (struct file *, char *, size_t, loff_t *);
	ssize_t (*write) (struct file *, const char *, size_t, loff_t *);
	int (*readdir) (struct file *, void *, filldir_t);
	unsigned int (*poll) (struct file *, struct poll_table_struct *);
	int (*ioctl) (struct inode *, struct file *, unsigned int, unsigned long);
	int (*mmap) (struct file *, struct vm_area_struct *);
	int (*open) (struct inode *, struct file *);
	int (*flush) (struct file *);
	int (*release) (struct inode *, struct file *);
	int (*fsync) (struct file *, struct dentry *, int datasync);
	int (*fasync) (int, struct file *, int);
	int (*lock) (struct file *, int, struct file_lock *);
	ssize_t (*readv) (struct file *, const struct iovec *, unsigned long, loff_t *);
	ssize_t (*writev) (struct file *, const struct iovec *, unsigned long, loff_t *);
};
```
从上面的结构可以看出，对文件的每个操作都有一个对应的接口与之相关联，例如 `minix` 文件系统只提供了 `read()`、`write()`、`mmap()` 和 `fsync()` 相关的接口，定义如下：
```cpp
struct file_operations minix_file_operations = {
	read:		generic_file_read,
	write:		generic_file_write,
	mmap:		generic_file_mmap,
	fsync:		minix_sync_file,
};
```
当对 `minix` 文件系统下的文件进行 `read()` 操作时，实际上调用的是 `generic_file_read()` 函数，后面我们还会对这个过程进行详细的解释。

## 具体实现
那么内核是怎么找到文件的 `inode` 结构呢？一般来说，要使用一个文件时必须先打开文件，而Linux提供打开文件的系统调用是 `open()`，而 `open()` 最终会调用内核态的 `sys_open()` 函数：
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
			struct file *f = filp_open(tmp, flags, mode);
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
`sys_open()` 首先会调用 `get_unused_fd()` 获取一个进程没有使用的文件句柄fd，然后调用 `filp_open()` 打开文件并返回一个类型为 `struct file` 的结构f，最后通过调用 `fd_install()` 把文件句柄fd与文件结构f关联起来。

我们主要来分析一下 `filp_open()` 函数的实现，代码如下：
```cpp
struct file *filp_open(const char * filename, int flags, int mode)
{
	int namei_flags, error;
	struct nameidata nd;

	namei_flags = flags;
	if ((namei_flags+1) & O_ACCMODE)
		namei_flags++;
	if (namei_flags & O_TRUNC)
		namei_flags |= 2;

	error = open_namei(filename, namei_flags, mode, &nd);
	if (!error)
		return dentry_open(nd.dentry, nd.mnt, flags);

	return ERR_PTR(error);
}
```
`filp_open()` 函数分别调用了 `open_namei()` 和 `dentry_open()` 这两个函数，我们先来分析 `open_namei()` 这个函数的实现，由于 `open_namei()` 函数比较繁琐，所以我们分段来分析这个函数：
```cpp
int open_namei(const char * pathname, int flag, int mode, struct nameidata *nd)
{
	int acc_mode, error = 0;
	struct inode *inode;
	struct dentry *dentry;
	struct dentry *dir;
	int count = 0;

	acc_mode = ACC_MODE(flag);

	if (!(flag & O_CREAT)) {
		if (path_init(pathname, lookup_flags(flag), nd))
			error = path_walk(pathname, nd);
		if (error)
			return error;
		dentry = nd->dentry;
		goto ok;
	}
```
如果传入的 `flag` 参数设置了 `O_CREAT` 标志，说明我们要创建一个新的文件，所以此时调用 `path_init()` 和 `path_walk()` 函数打开文件夹。`path_init()` 和 `path_walk()` 函数在Linux中用得比较多，`path_init()` 函数主要用于初始化 `struct nameidata` 结构，代码如下：
```cpp
struct nameidata {
	struct dentry *dentry; // 当前目录的dentry结构
	struct vfsmount *mnt;  // 目录所使用挂载点
	struct qstr last;      // 最后一级目录的名字
	unsigned int flags;    // 标志位
	int last_type;         // 最后一级目录的类型
};

static inline int
walk_init_root(const char *name, struct nameidata *nd)
{
	read_lock(&current->fs->lock);
	...
	nd->mnt = mntget(current->fs->rootmnt);
	nd->dentry = dget(current->fs->root);
	read_unlock(&current->fs->lock);
	return 1;
}

int path_init(const char *name, unsigned int flags, struct nameidata *nd)
{
	nd->last_type = LAST_ROOT; /* if there are only slashes... */
	nd->flags = flags;
	if (*name=='/') // 如果是绝对路径
		return walk_init_root(name,nd);
	read_lock(&current->fs->lock);
	nd->mnt = mntget(current->fs->pwdmnt);
	nd->dentry = dget(current->fs->pwd);
	read_unlock(&current->fs->lock);
	return 1;
}
```
如果要打开的文件是相对路径，那么就把 `struct nameidata` 结构的 `mnt` 成员设置为工作目录的文件系统挂载点（暂时不知道不禁要，后面会解释），把 `dentry` 成员设置为工作目录的目录项结构。如果要打开文件是绝对路径，那么就把 `struct nameidata` 结构的 `mnt` 成员设置为根目录的文件系统挂载点，把 `dentry` 成员设置为根目录的目录项结构。

