## SLAB分配算法
上一节说过Linux内核使用伙伴系统算法来管理内存页, 但伙伴系统算法分配的单位是内存页, 就是至少要分配一个或以上的内存块. 但很多时候我们并不需要分配一个内存页, 例如我们要申请一个大小为200字节的结构体时, 如果使用伙伴系统分配算法至少申请一个内存页, 但只使用了200字节的内存, 那么剩余的3896字节就被浪费掉了.

为了解决小块内存申请的问题, Linux内核引入了 `SLAB 分配算法`.  Linux 所使用的 `SLAB 分配算法` 的基础是 Jeff Bonwick 为SunOS 操作系统首次引入的一种算法。在内核中，会为有限的对象集（例如文件描述符和其他常见结构）分配大量内存。Jeff发现对内核中普通对象进行初始化所需的时间超过了对其进行分配和释放所需的时间。因此他的结论是不应该将内存释放回一个全局的内存池，而是将内存保持为针对特定目而初始化的状态。

为了更好的理解 `SLAB分配算法`, 我们先来介绍一下算法使用到的数据结构.

### 相关数据结构
`SLAB分配算法` 有两个重要的数据结构, 一个是 `kmem_cache_t`，另外一个是 `slab_t` . 下面我们先来看看 `kmem_cache_t` 结构:
```cpp
struct kmem_cache_s {
/* 1) each alloc & free */
    /* full, partial first, then free */
    struct list_head    slabs;
    struct list_head    *firstnotfull;
    unsigned int        objsize;
    unsigned int        flags;  /* constant flags */
    unsigned int        num;    /* # of objs per slab */
    spinlock_t      spinlock;
#ifdef CONFIG_SMP
    unsigned int        batchcount;
#endif

/* 2) slab additions /removals */
    /* order of pgs per slab (2^n) */
    unsigned int        gfporder;

    /* force GFP flags, e.g. GFP_DMA */
    unsigned int        gfpflags;

    size_t          colour;     /* cache colouring range */
    unsigned int        colour_off; /* colour offset */
    unsigned int        colour_next;    /* cache colouring */
    kmem_cache_t        *slabp_cache;
    unsigned int        growing;
    unsigned int        dflags;     /* dynamic flags */

    /* constructor func */
    void (*ctor)(void *, kmem_cache_t *, unsigned long);

    /* de-constructor func */
    void (*dtor)(void *, kmem_cache_t *, unsigned long);

    unsigned long       failures;

/* 3) cache creation/removal */
    char            name[CACHE_NAMELEN];
    struct list_head    next;
    ...
};
```

### SLAB分配算法初始化
SLAB分配算法的初始化由 `kmem_cache_init()` 函数完成，如下：
```cpp
void __init kmem_cache_init(void)
{
    size_t left_over;

    init_MUTEX(&cache_chain_sem);
    INIT_LIST_HEAD(&cache_chain);

    kmem_cache_estimate(0, cache_cache.objsize, 0,
            &left_over, &cache_cache.num);
    if (!cache_cache.num)
        BUG();

    cache_cache.colour = left_over/cache_cache.colour_off;
    cache_cache.colour_next = 0;
}
```
这个函数主要用来初始化 `cache_cache` 这个变量，`cache_cache` 是一个类型为 `kmem_cache_t` 的结构体变量，定义如下：
```cpp
static kmem_cache_t cache_cache = {
    slabs:      LIST_HEAD_INIT(cache_cache.slabs),
    firstnotfull:   &cache_cache.slabs,
    objsize:    sizeof(kmem_cache_t),
    flags:      SLAB_NO_REAP,
    spinlock:   SPIN_LOCK_UNLOCKED,
    colour_off: L1_CACHE_BYTES,
    name:       "kmem_cache",
};
```
为什么需要一个这样的对象呢？因为本身 `kmem_cache_t` 结构体也是小内存对象，所以也应该有slab分配器来分配的，但这样就出现“鸡蛋和鸡谁先出现”的问题。在系统初始化的时候，slab分配器还没有初始化，所以并不能使用slab分配器来分配一个 `kmem_cache_t` 对象，这时候只能通过定义一个 `kmem_cache_t` 类型的静态变量来来管理slab分配器了，所以 `cache_cache` 静态变量就是用来管理slab分配器的。

从上面的代码可知，`cache_cache` 的 `objsize字段` 被设置为 `sizeof(kmem_cache_t)` 的大小，所以这个对象主要是用来分配不同类型的 `kmem_cache_t` 对象的。

`kmem_cache_init()` 函数调用了 `kmem_cache_estimate()` 函数来计算一个slab能够保存多少个大小为 `cache_cache.objsize` 的对象，并保存到 `cache_cache.num` 字段中。一个slab中不可能全部都用来分配对象的，举个例子：一个4096字节大小的slab用来分配大小为22字节的对象，可以划分为186个，但还剩余4字节不能使用的，所以这部分内存用来作为着色区。着色区的作用是为了错开不同的slab，让CPU更有效的缓存slab。当然这属于优化部分，对slab分配算法没有多大的影响。就是说就算不对slab进行着色操作，slab分配算法还是可以工作起来的。
