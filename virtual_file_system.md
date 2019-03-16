# 虚拟文件系统
Linux支持多种文件系统，但提供给用户的文件操作接口却是一样的（`open()`、`read()`、`write()` ...），那么Linux是如何实现的呢？这就得益于 `虚拟文件系统(Virtual File System, 简称VFS)` 了。虚拟文件系统是在软件层实现的一个中间层，它屏蔽了底层文件系统的实现细节，对用户提供统一的接口来访问真实的文件系统。譬如说当用户使用 `open()` 系统调用打开一个文件时，如果文件所在的文件系统是 `ext2`，真正调用的就是 `ext2_open_file()`。如果文件所在的文件系统是 `nfs`，那么真正调用的就是 `nfs_open()`。

## 接口
那么虚拟文件系统究竟用了什么魔法来实现这个神奇的功能呢？其实用过面向对象编程语言（如Java、C++等）的同学都应该知道有个 `接口(interface)` 的概念，`接口` 是一系列方法的声明，是一些方法特征的集合。例如，我们使用Java定义一个 人类(Human) 的接口:
```java
public interface Human {
    public void eat();
    public void walk();
    public void speak(String content);
}
```
上面定义了一个 Human 的接口，接口中有3个方法：`eat()`、`walk()` 和 `speak()`，现在我们定义一个名为 Man 和一个名为 Woman 的类来实现 Human 这个接口：
```java
public class Man implements Human {
    public void eat() {
        System.out.println("Man eating...");
    }
    
    public void walk() {
        System.out.println("Man walking...");
    }
    
    public void speak(String content) {
        System.out.println("Man speaking..." + content);
    }
}

public class Woman implements Human {
    public void eat() {
        System.out.println("Woman eating...");
    }
    
    public void walk() {
        System.out.println("Woman walking...");
    }
    
    public void speak(String content) {
        System.out.println("Woman speaking..." + content);
    }
}
```
因为 Man 和 Woman 类都实现了 Human 这个接口，所以我们可以通过 Human 这个接口来访问这两个类的方法：
```java
public class Main {
    public static void main(String[] args) {
        Human human;
        Man man = new Man();
        Woman woman = new Woman();
        
        human = man;
        human.eat();
        
        human = woman;
        human.eat();
    }
}
```
上面代码输出如下：
```
Man eating...
Woman eating...
```
可以看出，Man 对象和 Woman 对象都可以当成 Human 接口类型来使用。读者可以有些疑惑，为什么讲 `虚拟文件系统` 会扯到Java去了，原因是 `虚拟文件系统` 使用的方式与面向对象的接口非常相似。但我们知道，Linux是使用 `C语言` 实现的， C语言是没有接口这个概念的，那么Linux是怎么模拟接口呢？虽然C语言没有接口，但是C语言有函数指针这个概念，Linux就是使用函数指针来模拟接口的。

## 虚拟文件系统实现
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
