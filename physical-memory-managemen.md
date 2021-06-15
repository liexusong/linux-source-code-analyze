## 物理内存管理
Linux的内存管理可分为物理内存管理和虚拟内存管理。所谓的虚拟内存就是指程序中使用的地址（32位系统可以使用4GB的虚拟内存地址），而物理地址是指计算机上真实安装的内存（例如我们安装了一条256MB的内存条，那么物理内存就只有256MB，但虚拟内存地址依然有4GB）。Linux内核需要通过一个映射关系才能把虚拟地址转换成物理地址，下面就来介绍Linux内存管理的细节。
### 内存管理区
Linux把物理内存划分为内存页进行管理, 一个内存页的大小为4KB, 所以在物理内存为1GB的电脑中可以分为 `262144 (计算公式为: 1024*1024*1024/1024/4)` 个内存页. Linux定义了一个名为page的结构体来描述物理内存页, 定义如下:
```cpp
typedef struct page {
    struct list_head list;
    struct address_space *mapping;
    unsigned long index;
    struct page *next_hash;
    atomic_t count;
    unsigned long flags;	/* atomic flags, some possibly updated asynchronously */
    struct list_head lru;
    unsigned long age;
    wait_queue_head_t wait;
    struct page **pprev_hash;
    struct buffer_head * buffers;
    void *virtual; /* non-NULL if kmapped */
    struct zone_struct *zone;
} mem_map_t;
```
每一个物理内存页都有一个对应的page结构来与之对应, Linux在初始化时会根据物理内存的大小分配对应的个数的page结构数组, 然后保存在mem_map全局变量中. mem_map变量的初始化如下:
```cpp
void __init free_area_init_core(int nid, pg_data_t *pgdat, struct page **gmap,
    unsigned long *zones_size, unsigned long zone_start_paddr,
    unsigned long *zholes_size, struct page *lmem_map)
{
    ...
    totalpages = 0;
    for (i = 0; i < MAX_NR_ZONES; i++) {
        unsigned long size = zones_size[i];
        totalpages += size;
    }
    realtotalpages = totalpages;
    if (zholes_size)
        for (i = 0; i < MAX_NR_ZONES; i++)
            realtotalpages -= zholes_size[i];
    ...
    /*
     * Some architectures (with lots of mem and discontinous memory
     * maps) have to search for a good mem_map area:
     * For discontigmem, the conceptual mem map array starts from
     * PAGE_OFFSET, we need to align the actual array onto a mem map
     * boundary, so that MAP_NR works.
     */
    map_size = (totalpages + 1)*sizeof(struct page);
    if (lmem_map == (struct page *)0) {
        lmem_map = (struct page *) alloc_bootmem_node(pgdat, map_size);
        lmem_map = (struct page *)(PAGE_OFFSET +
            MAP_ALIGN((unsigned long)lmem_map - PAGE_OFFSET));
    }
    ...
}

void __init free_area_init(unsigned long *zones_size)
{
    free_area_init_core(0, &contig_page_data, &mem_map, zones_size, 0, 0, 0);
}
```
free_area_init()函数会在系统初始化时被调用, 初始化mem_map的过程首先是计算系统中有多少个内存页 `totalpages`, 然后通过alloc_bootmem_node()分配page结构数组. 初始化后的内存结构如下图:

![](https://raw.githubusercontent.com/liexusong/myblog/master/images/memory_page.jpg)

除了使用page结构管理内存外, Linux还根据不同的用途把内存划分为不同的区域, 主要分为3个内存区域: `ZONE_DMA`, `ZONE_NORMAL` 和 `ZONE_HIGHMEM`. ZONE_DMA是 `小于等于16MB` 的内存地址, 而ZONE_NORMAL是 `大于ZONE_DMA小于等于896MB` 的内存地址, ZONE_HIGHMEM则是 `大于896MB` 的内存地址.

Linux使用 `zone_struct` 结构体来管理不同的内存区域, 其定义如下:
```cpp
#define MAX_ORDER 10

typedef struct free_area_struct {
    struct list_head    free_list;
    unsigned int        *map;
} free_area_t;

typedef struct zone_struct {
    /*
     * Commonly accessed fields:
     */
    spinlock_t           lock;
    unsigned long        offset; // 表示当前区在mem_map中的起始页面号
    unsigned long        free_pages;
    unsigned long        inactive_clean_pages;
    unsigned long        inactive_dirty_pages;
    unsigned long        pages_min, pages_low, pages_high;

    /*
     * free areas of different sizes
     */
    struct list_head    inactive_clean_list;
    free_area_t         free_area[MAX_ORDER]; // 用于伙伴分配算法

    /*
     * rarely used fields:
     */
    char                *name;
    unsigned long        size;
    /*
     * Discontig memory support fields.
     */
    struct pglist_data   *zone_pgdat;
    unsigned long        zone_start_paddr;
    unsigned long        zone_start_mapnr;
    struct page          *zone_mem_map;
} zone_t;
```
在介绍内存管理区之前先介绍一下 `内存节点(struct pglist_data)`, 现代计算机存储架构可分为: `非一致性内存架构(NUMA)` 和 `一致性内存架构(UMA)`. UMA是指介质均匀的(访问速度一致), 地址连续的存储架构. 对于这种内存架构, 系统访问任何内存地址的速度都是一样的. 但对于现代SMP(对称多处理器)架构的系统, UMA内存架构就不太适用, 因为对于多核CPU, 每个核心都有自己的本地内存, 而且访问本地内存的速度要比访问非本地内存的速度快很多, 像这种有多种不同介质和访问速度的内存架构叫NUMA.

在NUMA内存架构中, Linux定义了一个 `pglist_data` 的结构体来管理不同介质和访问速度的内存节点. 定义如下:
```cpp
#define NR_GFPINDEX	0x100

typedef struct pglist_data {
    zone_t node_zones[MAX_NR_ZONES];
    zonelist_t node_zonelists[NR_GFPINDEX];
    struct page *node_mem_map;
    unsigned long *valid_addr_bitmap;
    struct bootmem_data *bdata;
    unsigned long node_start_paddr;
    unsigned long node_start_mapnr;
    unsigned long node_size;
    int node_id;
    struct pglist_data *node_next;
} pg_data_t;
```
就是说在Linux系统中, 内存管理的最顶层是 `内存节点(pglist_data)`, 接着是 `内存管理区(zone_struct)`, 最后才到 `内存页(page)`. 三者的关系如下图:
![](https://raw.githubusercontent.com/liexusong/myblog/master/images/memory_zone.gif)

所以Linux分配内存时首先找到合适的内存节点, 然后再到指定的内存区(DMA, NORMAL或者HIGHMEM)中分配一定数量的内存页. 由于内存区分配内存时使用的是伙伴系统算法, 所以分配出来的内存页物理地址一定是连续的.

### 物理内存页分配
Linux内核使用 `alloc_pages()` 函数来进行分配物理内存页，alloc_pages()函数的原型如下：
```cpp
struct page * alloc_pages(int gfp_mask, unsigned long order);
```
参数 `gfp_mask` 是分配策略, 而 `order` 是分配 `1<<order` 个页面.

Linux内核有UMA和NUMA两个版本的alloc_pages(), 为了简单起见, 我们直接选用UMA版本的, 因为UMA只有一个内存节点. 我们先来看看UMA版本的alloc_pages()源码:
```cpp
static bootmem_data_t contig_bootmem_data;
pg_data_t contig_page_data = { bdata: &contig_bootmem_data };

static inline struct page * alloc_pages(int gfp_mask, unsigned long order)
{
	/*
	 * Gets optimized away by the compiler.
	 */
	if (order >= MAX_ORDER)
		return NULL;
	return __alloc_pages(contig_page_data.node_zonelists+(gfp_mask), order);
}
```
UMA架构的系统只有一个内存节点, Linux使用变量 `contig_page_data` 来存储. 前面我们介绍过 `struct pglist_data` 结构, `node_zonelists` 字段是不同的分配策略的数组, 可以通过数组的下标来指定分配策略. Linux内核定义的分配策略下标有以下几种:
```cpp
// 标志位
#define __GFP_WAIT	0x01
#define __GFP_HIGH	0x02
#define __GFP_IO	0x04
#define __GFP_DMA	0x08
#ifdef CONFIG_HIGHMEM
#define __GFP_HIGHMEM	0x10
#else
#define __GFP_HIGHMEM	0x0 /* noop */
#endif

// 分配策略
#define GFP_BUFFER	(__GFP_HIGH | __GFP_WAIT)
#define GFP_ATOMIC	(__GFP_HIGH)
#define GFP_USER	(__GFP_WAIT | __GFP_IO)
#define GFP_HIGHUSER	(__GFP_WAIT | __GFP_IO | __GFP_HIGHMEM)
#define GFP_KERNEL	(__GFP_HIGH | __GFP_WAIT | __GFP_IO)
#define GFP_NFS		(__GFP_HIGH | __GFP_WAIT | __GFP_IO)
#define GFP_KSWAPD	(__GFP_IO)
#define GFP_DMA		__GFP_DMA
#define GFP_HIGHMEM	__GFP_HIGHMEM
```
有个疑问是, 按这种分配策略的组合来看最多也只有16种, 但是Linux定义 `node_zonelists` 数组的大小是256, 所以下标15以后的分配策略是永远都不会访问到的. Linux为什么要定义这么大数组呢, 难道是为了以后扩展?

我们在来看看怎么初始化一个内存节点中的分配策略, 分配策略的初始化是在 `build_zonelists()` 函数中完成:
```cpp
#define NR_GFPINDEX	0x100  // 10进制是256

static inline void build_zonelists(pg_data_t *pgdat)
{
	int i, j, k;

	for (i = 0; i < NR_GFPINDEX; i++) {
		zonelist_t *zonelist;
		zone_t *zone;

		zonelist = pgdat->node_zonelists + i;
		memset(zonelist, 0, sizeof(*zonelist));

		zonelist->gfp_mask = i;
		j = 0;
		k = ZONE_NORMAL;
		if (i & __GFP_HIGHMEM)
			k = ZONE_HIGHMEM;
		if (i & __GFP_DMA)
			k = ZONE_DMA;

		switch (k) {
			default:
				BUG();
			/*
			 * fallthrough:
			 */
			case ZONE_HIGHMEM:
				zone = pgdat->node_zones + ZONE_HIGHMEM;
				if (zone->size) {
#ifndef CONFIG_HIGHMEM
					BUG();
#endif
					zonelist->zones[j++] = zone;
				}
			case ZONE_NORMAL:
				zone = pgdat->node_zones + ZONE_NORMAL;
				if (zone->size)
					zonelist->zones[j++] = zone;
			case ZONE_DMA:
				zone = pgdat->node_zones + ZONE_DMA;
				if (zone->size)
					zonelist->zones[j++] = zone;
		}
		zonelist->zones[j++] = NULL;
	}
}
```
分配策略的初始化很简单, 就是根据标志位的不同来为 `zonelist` 添加不同的内存管理区.

现在回头来看看 `__alloc_pages()` 函数的实现, 由于这个函数比较长, 所以我们分段开看:
```cpp
struct page * __alloc_pages(zonelist_t *zonelist, unsigned long order)
{
	zone_t **zone;
	int direct_reclaim = 0;
	unsigned int gfp_mask = zonelist->gfp_mask;
	struct page * page;

	memory_pressure++;

	if (order == 0 && (gfp_mask & __GFP_WAIT) &&
		!(current->flags & PF_MEMALLOC))
		direct_reclaim = 1;  // 是否申请单个内存页并且可以阻塞的

	// 可用物理内存页是否足够
	if (inactive_shortage() > inactive_target / 2 && free_shortage())
		wakeup_kswapd(0);
	// 脏页面是否过多
	else if (free_shortage() && nr_inactive_dirty_pages > free_shortage()
			&& nr_inactive_dirty_pages >= freepages.high)
		wakeup_bdflush(0);
```
上面的代码首先判断是否申请单个内存页, 并且可以阻塞当前进程, 如果都满足, 就把 `direct_reclaim` 变量设置为1. 然后判断可用物理内存页是否足够, 如果不足就调用 `wakeup_kswapd()` 唤起kswapd内核线程进行回收一些物理内存页. 另外, 如果可用物理内存页不足时还需要判断非活跃脏页面是否超过一定的数量, 如果超过了就调用 `wakeup_bdflush()` 唤醒 `bdflush` 内核线程, 把脏页面写到磁盘中.
```cpp
try_again:
	/*
	 * First, see if we have any zones with lots of free memory.
	 *
	 * We allocate free memory first because it doesn't contain
	 * any data ... DUH!
	 */
	zone = zonelist->zones;
	for (;;) {
		zone_t *z = *(zone++);
		if (!z)
			break;
		if (!z->size)
			BUG();

		if (z->free_pages >= z->pages_low) {
			page = rmqueue(z, order);
			if (page)
				return page;
		} else if (z->free_pages < z->pages_min &&
			waitqueue_active(&kreclaimd_wait)) {
			wake_up_interruptible(&kreclaimd_wait);
		}
	}
```
然后开始遍历内存节点中的内存管理区, 如果某个内存管理区的空闲内存页足够, 那么调用 `rmqueue()` (使用伙伴系统算法分配, 下面会介绍)从内存管理区的空闲页面队列中分配内存页, 如果成功了就直接返回申请到的内存页. 如果内存管理区的空闲内存页小于最低水位时, 那么就唤醒 `kreclaimd` 内核线程释放内存管理区一些非活跃的并且干净的内存页给系统.
```cpp
	page = __alloc_pages_limit(zonelist, order, PAGES_HIGH, direct_reclaim);
	if (page)
		return page;

	page = __alloc_pages_limit(zonelist, order, PAGES_LOW, direct_reclaim);
	if (page)
		return page;
```
如果所有的内存管理区都没有足够的空闲页面, 那么就调用 `__alloc_pages_limit()` 函数来分配, 这个函数主要是把分配内存时的条件放宽一些, 这样就更容易分配到内存页.
```cpp
	wakeup_kswapd(0);
	if (gfp_mask & __GFP_WAIT) {
		__set_current_state(TASK_RUNNING);
		current->policy |= SCHED_YIELD;
		schedule();
	}

	page = __alloc_pages_limit(zonelist, order, PAGES_MIN, direct_reclaim);
	if (page)
		return page;
```
经过一番努力还是不能申请到内存页, 那么唤醒 `kswapd` 内核线程. 如果申请内存的进程设置了可以阻塞的标志, 那么就先让出CPU, 让 `kswapd` 内核线程先回收一些内存页. 接着接续调用 `__alloc_pages_limit()` 尝试申请内存页, 如果成功直接返回.
```cpp
	if (!(current->flags & PF_MEMALLOC)) {
		if (order > 0 && (gfp_mask & __GFP_WAIT)) {
			zone = zonelist->zones;
			/* First, clean some dirty pages. */
			current->flags |= PF_MEMALLOC;
			page_launder(gfp_mask, 1);
			current->flags &= ~PF_MEMALLOC;
			for (;;) {
				zone_t *z = *(zone++);
				if (!z)
					break;
				if (!z->size)
					continue;
				while (z->inactive_clean_pages) {
					struct page * page;
					/* Move one page to the free list. */
					page = reclaim_page(z);
					if (!page)
						break;
					__free_page(page);
					/* Try if the allocation succeeds. */
					page = rmqueue(z, order);
					if (page)
						return page;
				}
			}
		}
```
上面的过程说明当进程可以阻塞的时候, 可以调用 `page_launder()` 函数来把一些脏页面写到磁盘, 再尝试申请内存页.
```cpp
		if ((gfp_mask & (__GFP_WAIT|__GFP_IO)) == (__GFP_WAIT|__GFP_IO)) {
			wakeup_kswapd(1);
			memory_pressure++;
			if (!order)
				goto try_again;
		} else if (gfp_mask & __GFP_WAIT) {
			try_to_free_pages(gfp_mask);
			memory_pressure++;
			if (!order)
				goto try_again;
		}

	}
```
如果 gfp_mask 设置了 `(__GFP_WAIT|__GFP_IO)` 标志, 那么就唤醒 `kswapd` 内核线程, 并等待其执行完毕. 如果只设置了 `__GFP_WAIT` 标志, 那么就调用 `try_to_free_pages()` 尝试释放一些内存页.
```cpp
	zone = zonelist->zones;
	for (;;) {
		zone_t *z = *(zone++);
		struct page * page = NULL;
		if (!z)
			break;
		if (!z->size)
			BUG();

		if (direct_reclaim) {
			page = reclaim_page(z);
			if (page)
				return page;
		}

		/* XXX: is pages_min/4 a good amount to reserve for this? */
		if (z->free_pages < z->pages_min / 4 &&
				!(current->flags & PF_MEMALLOC))
			continue;
		page = rmqueue(z, order);
		if (page)
			return page;
	}
	return NULL;
```
最后尝试申请内存页, 如果进程设置了 `PF_MEMALLOC` 标志 (一般都是回收内存页的进程, 所以这些进程需要内存时必须满足他们, 因为满足他们能够获得更多的空闲内存页), 并且内存管理区空闲内存页大于最低水位的四分之一, 那么还是尝试调用 `rmqueue()` 来说申请内存页. 如果都失败的话, 那么只能返回NULL了.
