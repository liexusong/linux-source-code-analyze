## MINIX文件系统分析

硬盘是用来持久化存储数据的硬件，一般来说，硬盘分为磁盘和固态硬盘，但这里并不会介绍太多关于硬件方面的知识，我们主要详细介绍 `MINIX文件系统` 的原理和现实。

### 什么是文件系统

硬盘属于 `块设备` 硬件，也就是说对硬盘的操作都是通过块来进行的。对于一般的磁盘来说，一个块为512字节（称为扇区）。硬盘就是按照块大小来划分成多个块，每个块都有唯一的编号。

但是对于人类来说，通过块来操作硬盘是非常不方便的，譬如你不知道某个数据块是否已经被使用（当然也可以通过在数据块的第一个字节记录是否已经使用），而且你也不知道这个数据块保存了什么数据。所以这个时候就需要 `文件系统` 来帮助管理这些数据。

那么什么是 `文件系统` 呢？

> 百科上的定义： `文件系统` 是操作系统用于明确磁盘或分区上的文件的方法和数据结构，即在磁盘上组织文件的方法。

前面说过，内存通过内存管理系统进行管理，所以硬盘也需要硬盘管理系统来管理，也就是 `文件系统`。`文件系统` 通过文件的方式来管理硬盘上的数据，并且提供给用户一些简单的API（如 `read()` 和 `write()` 等接口）来操作这些文件。

### 什么是MINIX文件系统

`MINIX文件系统` 是从MINIX操作系统移植到Linux的，其优点是简单清晰，适合用于教学使用。但缺点是过于简单，不适合生产环境使用。不过通过对MINIX文件系统的分析，有助于理解其他复杂的文件系统。

### MINIX文件系统结构

在分析MINIX文件系统实现前，我们先来了解一下MINIX文件系统在硬盘的结构组织，如下图：

![minix_filesystem](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/minix_filesystem.png)

可以把硬盘当成一个由多个数据块组成的设备（对于MINIX文件系统一个数据块的大小为1024字节），MINIX文件系统就是组织和管理这些数据块的一种算法。下面来介绍一下上图中各个部分的作用：

* `超级块`：用于记录文件系统的一些信息，比如inode的数量和inode位图占用的数据块数量等。
* `inode位图`：用于记录inode的使用情况，每个位代表inode的使用情况，1表示已经使用，0代表空闲。
* `逻辑块位图`：用于记录逻辑数据块的使用情况，每个位代表一个逻辑数据块的使用情况，1表示已经使用，0代表空闲。
* `inode表`：用于记录所有inode的信息，每个记录代表一个inode。
* `逻辑数据块`：真正保存数据的数据块。

从上面的介绍可以看到，MINIX文件系统的结构比较简单，接下来我们分析一下MINIX文件系统的实现。

### MINIX文件系统相关数据结构

在分析MINIX文件系统的实现前，先来介绍一下两个重要的数据结构：`minix_super_block` 和 `minix2_inode`，`minix_super_block` 对应MINIX文件系统的超级块，而 `minix_super_block` 则对应MINIX文件系统的inode节点。

#### minix_super_block结构
```c
struct minix_super_block {
    __u16 s_ninodes;       // inode的个数
    __u16 s_nzones;        // 逻辑数据块个数(v1版)
    __u16 s_imap_blocks;   // inode位图占用的数据块数量
    __u16 s_zmap_blocks;   // 数据块位图占用的数据块数量
    __u16 s_firstdatazone; // 第一个逻辑数据块起始号
    __u16 s_log_zone_size; // 使用2为底的对数表示的每个逻辑数据块包含的磁盘块数
    __u32 s_max_size;      // 文件最大尺寸
    __u16 s_magic;         // 魔数
    __u16 s_state;         // 文件系统状态
    __u32 s_zones;         // 逻辑数据块个数(v2版)
};
```

#### minix2_inode结构
```c
struct minix2_inode {
    __u16 i_mode;     // 文件模式
    __u16 i_nlinks;   // 链接数
    __u16 i_uid;      // 所属用户ID
    __u16 i_gid;      // 所属组ID
    __u32 i_size;     // 文件大小
    __u32 i_atime;    // 访问时间
    __u32 i_mtime;    // 修改时间
    __u32 i_ctime;    // 创建时间
    __u32 i_zone[10]; // 文件数据存储的逻辑数据块编号
};
```
`minix2_inode` 的 `i_zone` 字段记录了文件数据存储在哪些逻辑数据块上，可以看到 `i_zone` 字段是一个有10个元素的数组，前7个元素是直接指向的数据块，就是数据会直接存储在这些数据块上。而第8个元素是一级间接指向，第9个元素是二级间接指向，第10个元素是三级间接指向，原理如下图：

![minix_filesystem_inode](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/minix_filesystem_inode.jpg)

### MINIX文件系统实现

要在一个硬盘或者分区安装文件系统，首先需要对硬盘或者分区进行格式化操作，Linux系统可以使用 `mkfs` 命令对硬盘或者分区进行格式化，例如可以使用下面命令对设备进行格式化：
```shell
$ mkfs -t ext4 -b 4096 /dev/sdb5
```
上面的命令把设备 `/dev/sdb5` 格式化为 `ext4` 文件系统，并且每个逻辑数据块大小为4k。格式化过程就是按照具体文件系统指定的结构来编排数据，比如格式化为MINIX文件系统，就需要写入超级块、inode位图、逻辑数据块位图、inode表等数据结构。

格式化完后可以通过 `mount` 命名将设备挂载到某个目录下，这样就可以访问这个设备了。如下：
```shell
$ mount /dev/sdb5 /mnt/foo
$ cd /mnt/foo
```
挂载过程就是读入文件系统的超级块到内存，对于MINIX文件系统，读入超级块数据是通过 `minix_read_super()` 函数实现的，代码如下：
```c
static struct super_block *minix_read_super(
    struct super_block *s, void *data, int silent)
{
    struct minix_super_block *ms;

    ...
    if (!(bh = bread(dev,1,BLOCK_SIZE)))
        goto out_bad_sb;

    ms = (struct minix_super_block *) bh->b_data;
    ...

    /*
     * 申请inode位图和逻辑数据块位图的内存
     */
    i = (s->u.minix_sb.s_imap_blocks + s->u.minix_sb.s_zmap_blocks) * sizeof(bh);
    map = kmalloc(i, GFP_KERNEL);
    if (!map)
        goto out_no_map;
    memset(map, 0, i);
    s->u.minix_sb.s_imap = &map[0];
    s->u.minix_sb.s_zmap = &map[s->u.minix_sb.s_imap_blocks];

    block=2;
    for (i=0 ; i < s->u.minix_sb.s_imap_blocks ; i++) { // 读取inode位图
        if (!(s->u.minix_sb.s_imap[i]=bread(dev,block,BLOCK_SIZE)))
            goto out_no_bitmap;
        block++;
    }

    for (i=0 ; i < s->u.minix_sb.s_zmap_blocks ; i++) { // 读取数据块位图
        if (!(s->u.minix_sb.s_zmap[i]=bread(dev,block,BLOCK_SIZE)))
            goto out_no_bitmap;
        block++;
    }

    // 设置第一inode和第一个逻辑数据块为已被使用(文件系统的根目录)
    minix_set_bit(0,s->u.minix_sb.s_imap[0]->b_data);
    minix_set_bit(0,s->u.minix_sb.s_zmap[0]->b_data);

    s->s_op = &minix_sops;
    root_inode = iget(s, MINIX_ROOT_INO);
    if (!root_inode)
        goto out_no_root;

    // 读取根目录inode节点
    s->s_root = d_alloc_root(root_inode);
    if (!s->s_root)
        goto out_iput;

    s->s_root->d_op = &minix_dentry_operations;

    ...
    return s;
    ...
}
```
`minix_read_super()` 函数首先通过 `bread()` 读取设备的第2个数据块（第1个数据块是引导区），超级块就保存在这个数据块中。然后根据MINIX文件系统的超级块数据设置虚拟文件系统（VFS）的超级块，接着为inode位图和逻辑数据块位图申请内存，并且读入设备中的inode位图和逻辑数据块位图到内存中。最后读取根目录的inode节点数据，并保存到虚拟文件系统超级块的 `s_root` 字段中。

#### 读取文件数据
读取MINIX文件系统的文件过程如下图：

![minix-filesystem-read](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/minix-filesystem-read.jpg)

读文件时，首先需要打开文件，然后通过调用 `read()` 系统调用来读取文件中的内容。`read()` 系统调用原型如下：
```cpp
ssize_t read(int fd, void *buf, size_t count);
```
参数的意义：
1. `fd`: 打开的文件句柄。
2. `buf`: 用于存放读取内容的内存地址。
3. `count`: 需要从文件中读取多少字节的数据。

`read()` 系统调用会触发调用 `虚拟文件系统层（VFS）` 的 `sys_read()` 函数，而对于 MINIX 文件系统，`sys_read()` 函数会接着调用 `generic_file_read()` 函数，`generic_file_read()` 函数又接着调用 `do_generic_file_read()` 函数。

`do_generic_file_read()` 函数的实现比较复杂，首先会去缓存中查找要读取的内容是否已经存在，如果存在直接返回缓存的数据即可。如果还没有缓存，那么调用 `minix_readpage()` 从磁盘中读取文件的内容到缓存中，最后调用 `file_read_actor()` 函数把数据复制都用户空间的 `buf` 参数中。
