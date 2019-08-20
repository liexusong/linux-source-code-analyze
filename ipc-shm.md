# 进程间通信 - 共享内存
在Linux系统中，每个进程都有独立的虚拟内存空间，也就是说不同的进程访问同一段虚拟内存地址所得到的数据是不一样的，这是因为不同进程相同的虚拟内存地址会映射到不同的物理内存地址上。

但有时候为了让不同进程之间进行通信，需要让不同进程共享相同的物理内存，Linux通过 `共享内存` 来实现这个功能。下面先来介绍一下Linux系统的共享内存的使用。

## 共享内存使用
### 1. 获取共享内存
要使用共享内存，首先需要使用 `shmget()` 函数获取共享内存，`shmget()` 函数的原型如下：
```cpp
int shmget(key_t key, size_t size, int shmflg);
```
* 参数 `key` 一般由 `ftok()` 函数生成，用于标识系统的唯一IPC资源。
* 参数 `size` 指定创建的共享内存大小。
* 参数 `shmflg` 指定 `shmget()` 函数的动作，比如传入 `IPC_CREAT` 表示要创建新的共享内存。

函数调用成功时返回一个新建或已经存在的的共享内存标识符，取决于shmflg的参数。失败返回-1，并设置错误码。

### 2. 关联共享内存
`shmget()` 函数返回的是一个标识符，而不是可用的内存地址，所以还需要调用 `shmat()` 函数把共享内存关联到某个虚拟内存地址上。`shmat()` 函数的原型如下：
```cpp
void *shmat(int shmid, const void *shmaddr, int shmflg);
```
* 参数 `shmid` 是 `shmget()` 函数返回的标识符。
* 参数 `shmaddr` 是要关联的虚拟内存地址，如果传入0，表示由系统自动选择合适的虚拟内存地址。
* 参数 `shmflg` 若指定了 `SHM_RDONLY` 位，则以只读方式连接此段，否则以读写方式连接此段。

函数调用成功返回一个可用的指针（虚拟内存地址），出错返回-1。

### 3. 取消关联共享内存
当一个进程不需要共享内存的时候，就需要取消共享内存与虚拟内存地址的关联。取消关联共享内存通过 `shmdt()` 函数实现，原型如下：
```cpp
int shmdt(const void *shmaddr);
```
* 参数 `shmaddr` 是要取消关联的虚拟内存地址，也就是 `shmat()` 函数返回的值。

函数调用成功返回0，出错返回-1。

### 共享内存使用例子
下面通过一个例子来介绍一下共享内存的使用方法。在这个例子中，有两个进程，分别为 `进程A` 和 `进程B`，`进程A` 创建一块共享内存，然后写入数据，`进程B` 获取这块共享内存并且读取其内容。
#### 进程A
```cpp
#include <stdio.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/ipc.h>
#include <sys/shm.h>

#define SHM_PATH "/tmp/shm"
#define SHM_SIZE 128

int main(int argc, char *argv[])
{
    int shmid;
    char *addr;
    key_t key = ftok(SHM_PATH, 0x6666);
    
    shmid = shmget(key, SHM_SIZE, IPC_CREAT|IPC_EXCL|0666);
    if (shmid < 0) {
        printf("failed to create share memory\n");
        return -1;
    }
    
    addr = shmat(shmid, NULL, 0);
    if (addr <= 0) {
        printf("failed to map share memory\n");
        return -1;
    }
    
    sprintf(addr, "%s", "Hello World\n");
    
    return 0;
}
```

#### 进程B
```cpp
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/ipc.h>
#include <sys/shm.h>

#define SHM_PATH "/tmp/shm"
#define SHM_SIZE 128

int main(int argc, char *argv[])
{
    int shmid;
    char *addr;
    key_t key = ftok(SHM_PATH, 0x6666);
    
    char buf[128];
    
    shmid = shmget(key, SHM_SIZE, IPC_CREAT);
    if (shmid < 0) {
        printf("failed to get share memory\n");
        return -1;
    }
    
    addr = shmat(shmid, NULL, 0);
    if (addr <= 0) {
        printf("failed to map share memory\n");
        return -1;
    }
    
    strcpy(buf, addr, 128);
    printf("%s", buf);
    
    return 0;
}
```
测试时先运行进程A，然后再运行进程B，可以看到进程B会打印出 “Hello World”，说明共享内存已经创建成功并且读取。

## 共享内存实现原理
我们先通过一幅图来了解一下共享内存的大概原理，如下图：
![shm-map](https://raw.githubusercontent.com/liexusong/linux-source-code-analyze/master/images/shm-map.jpg)

通过上图可知，共享内存是通过将不同进程的虚拟内存地址映射到相同的物理内存地址来实现的，下面将会介绍Linux的实现方式。

在Linux内核中，每个共享内存都由一个名为 `struct shmid_kernel` 的结构体来管理，而且Linux限制了系统最大能创建的共享内存为128个。通过类型为 `struct shmid_kernel` 结构的数组来管理，如下：
```cpp
struct shmid_ds {
	struct ipc_perm		shm_perm;	/* operation perms */
	int			shm_segsz;	/* size of segment (bytes) */
	__kernel_time_t		shm_atime;	/* last attach time */
	__kernel_time_t		shm_dtime;	/* last detach time */
	__kernel_time_t		shm_ctime;	/* last change time */
	__kernel_ipc_pid_t	shm_cpid;	/* pid of creator */
	__kernel_ipc_pid_t	shm_lpid;	/* pid of last operator */
	unsigned short		shm_nattch;	/* no. of current attaches */
	unsigned short 		shm_unused;	/* compatibility */
	void 			*shm_unused2;	/* ditto - used by DIPC */
	void			*shm_unused3;	/* unused */
};

struct shmid_kernel
{	
	struct shmid_ds		u;
	/* the following are private */
	unsigned long		shm_npages;	/* size of segment (pages) */
	pte_t			*shm_pages;	/* array of ptrs to frames -> SHMMAX */ 
	struct vm_area_struct	*attaches;	/* descriptors for attaches */
};

static struct shmid_kernel *shm_segs[SHMMNI]; // SHMMNI等于128
```
从注释可以知道 `struct shmid_kernel` 结构体各个字段的作用，比如 `shm_npages` 字段表示共享内存使用了多少个内存页。而 `shm_pages` 字段指向了共享内存映射的虚拟内存页表项数组等。

### shmget() 函数实现
通过前面的例子可知，要使用共享内存，首先需要调用 `shmget()` 函数来创建或者获取一块共享内存。`shmget()` 函数的实现如下：
```cpp
asmlinkage long sys_shmget (key_t key, int size, int shmflg)
{
	struct shmid_kernel *shp;
	int err, id = 0;

	down(&current->mm->mmap_sem);
	spin_lock(&shm_lock);
	if (size < 0 || size > shmmax) {
		err = -EINVAL;
	} else if (key == IPC_PRIVATE) {
		err = newseg(key, shmflg, size);
	} else if ((id = findkey (key)) == -1) {
		if (!(shmflg & IPC_CREAT))
			err = -ENOENT;
		else
			err = newseg(key, shmflg, size);
	} else if ((shmflg & IPC_CREAT) && (shmflg & IPC_EXCL)) {
		err = -EEXIST;
	} else {
		shp = shm_segs[id];
		if (shp->u.shm_perm.mode & SHM_DEST)
			err = -EIDRM;
		else if (size > shp->u.shm_segsz)
			err = -EINVAL;
		else if (ipcperms (&shp->u.shm_perm, shmflg))
			err = -EACCES;
		else
			err = (int) shp->u.shm_perm.seq * SHMMNI + id;
	}
	spin_unlock(&shm_lock);
	up(&current->mm->mmap_sem);
	return err;
}
```
`shmget()` 函数的实现比较简单，首先调用 `findkey()` 函数查找值为key的共享内存是否已经被创建。如果没被创建，那么就调用 `newseg()` 函数创建新的共享内存。
