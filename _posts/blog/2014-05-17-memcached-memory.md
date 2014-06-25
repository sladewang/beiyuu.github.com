---
layout: post
title: Memcached源码学习笔记-内存管理
description: Memcached源码学习笔记-内存管理
category: blog
---

最近看了下memcached的内存池实现（源码基于memcached 1.4.19），做一些记录。Memcached作为一个高性能的内存缓存server，使用了[Slab allocation][1]内存池方式管理内存。避免了直接使用glibc通用内存分配器ptmalloc（malloc和free）可能会导致的[碎片][2]，["内存泄漏"][3]这些问题。Slab allocation原理是先从操作系统中预分配一大块内存，然后将大块内存分割成一系列固定尺寸大小的块(chunk)，并把尺寸相同的块集合成组(slabclass)。需要使用内存的时候从内存池中取一块大小合适的内存，用完了还给内存池循环利用，减少内存分配和释放的开销，也避免了外部碎片。

如下图，可以把memcached使用的内存看成一个slabclass数组。每一个元素slabclass[i]里都有若干slab，每一个slab默认是1M大小的内存。这些1M大小的slab进一步被切分成大小相等的chunk，chunk就是用来存储业务需要cache的数据（item），通常chunk会比item略大。每个slabclass中所有chunk大小是一样的，不同slabclass之间chunk大小不同，这和设置中的factor变量有关。factor是用来控制相邻slabclass之间chunk大小增长比例的，可以自定义，默认为 1.25。如果 slabclass[0]里chunk大小为 88 byte，那么88 * 1.25 = 110，slabclass\[1\]里chunk大小就是112 byte（chunk需要8字节对齐）。Facebook使用memcached的factor取值为1.07，目的是让64 byte和128 byte可以在chunk size离散值范围内，匹配CPU cache line，详见[Scaling Memcache at Facebook][4]。使用内存的时候，memcached根据收到的item数据大小，选择大小最合适的slabclass。由于chunk size是一系列预先固定的离散值，所以chunk中无法避免存在着内部碎片。
![slab_overview](/images/memcached_memory/slab_overview.jpg)

代码主要有slab.c和item.c两部分，slab.c中是slab内存操作相关的函数，为item.c中item存取操作函数提供接口。先来看一下slabclass的结构定义：

	typedef struct {
	    unsigned int size;      /* 该slabclass的chunk size */
	    unsigned int perslab;   /* 每个slab可以放多少item(chunk) */

	    void *slots;            /* 内存可用的item链表 */
	    unsigned int sl_curr;   /* 内存可用链表的元素总量 */

	    unsigned int slabs;     /* slabclass中内存已被使用的slab数量 */
	    void **slab_list;       /* slab指针数组 */
	    unsigned int list_size; /* slab指针数组长度，当slabs等于list_size的时候，
                                       需要realloc一个slab指针数组 */

	    unsigned int killing;  /* index+1 of dying slab, or zero if none，slab rebalance的时候用到 */
	    size_t requested; /* slabclass已被使用内存总量 */
	} slabclass_t;

	static slabclass_t slabclass[MAX_NUMBER_OF_SLAB_CLASSES];

每个slabclass通过slots指针指向内存可用的item双向链表，sl_curr表示该链表中item的总数量。因为slabclass中chunk size固定，所以分配slab内存的时候不需要像通用内存分配器中first-fit，best-fit那样查找链表。使用内存就直接从对应slabclass的可用item链表拿第一个节点来用，归还内存的时候，把item插入到链表头即可，申请和归还都是O(1)操作。slab.c对外提供的接口主要有下面这几个（省略了slab stats和slab rebalance相关的接口）。

	/* slab模块初始化，limit是slab allocator内存大小限制，factor是相邻slabclass的
	   chunk size增长因子，prealloc内存是预分配还是按需分配 */
	void slabs_init(const size_t limit, const double factor, const bool prealloc);

	/* 给定item size，找到最合适的slabclass */
	unsigned int slabs_clsid(const size_t size);

	/* 从slab上分配内存 */
	void *slabs_alloc(const size_t size, unsigned int id);

	/* 归还slab内存 */
	void slabs_free(void *ptr, size_t size, unsigned int id);

先来看初始化slab的slabs_init函数，这个函数的流程如下：
* 如果prealloc参数为true指定了预分配的话就会预先从操作系统分配一超大块内存，以prealloc等于true为例
* 首先根据factor参数为每个slabclass设置chunk size和perslab
* 然后调用slabs_preallocate函数为每一个slabclass分配内存。调用do_slabs_newslab从大块内存中拿出1M内存给slabclass。do_slabs_newslab中还会调用grow_slab_list初始化slab_list这个指针数组，初始为16个元素，后续不够了再realloc重新分配。
* slab内存分配完成之后，再通过split_slab_page_into_freelist函数，把slab上所有的chunk（item）加到slabclass的slots可用链表上。

至此，整个slab_init初始化流程就完成了，chunk size为96的slabclass内存结构示意图如下图，其他slabclass也类似。需要注意的是，指定prealloc预分配的话需要操作系统内存管理支持设置大页面，见memcached.c中的enable_large_pages函数。注释上说这主要是为了减少TLB（页表缓存）cache miss，因为如果从操作系统预分配大块内存，还使用4KB内存页的话，进程页表会很大，不能全部放在cpu cache中。如果不是preallocate的话，每次当slabclass内存不够时，memcached会以一块slab内存大小为单位向操作系统申请，slabclass内存结构和preallocate的情况是一样的。

![slabclass](/images/memcached_memory/slabclass.jpg)

memcached默认开启4个worker线程，slab allocator用了一个全局互斥锁slabs_lock来进行多线程同步。slabs_alloc和slabs_free两个函数就是加锁之后操作slabclass的slots链表来进行内存分配和释放。这里slabs_lock似乎是个粗粒度的锁，不过应该问题貌似不大，大概有两个原因：1，临界区的操作非常快速，只是简单修改链表指针。2，item模块还有LRU链表，item使用内存的时候会先从LRU链表上查找是否有过期item的内存可以重用，不会每次都从slabclass分配内存。  

	/* Structure for storing items within memcached. */
	typedef struct _stritem {
	    struct _stritem *next;
	    struct _stritem *prev;	/* slab可用内存和LRU都是通过双向链表来维护 */
	    struct _stritem *h_next;    /* hash chain next，hash表是开链解决冲突 */
	    rel_time_t      time;       /* least recent access，最近操作（查询）时间 */
	    rel_time_t      exptime;    /* expire time，过期时间 */
	    int             nbytes;     /* size of data */
	    unsigned short  refcount;   /* 引用计数，表示item是否正在被使用 */
	    uint8_t         nsuffix;    /* length of flags-and-length string */
	    uint8_t         it_flags;   /* ITEM_* above */
	    uint8_t         slabs_clsid;/* which slab class we're in */
	    uint8_t         nkey;       /* key length */
	    union {
		uint64_t cas;
		char end;
	    } data[];
	} item;

上面提到并不是每次item需要使用内存就从slabclass中分配。每一个slabclass集合都有一个对应的LRU双向链表。这个链表并不定义在slabclass结构体里，它是全局变量 \*\*head 与\*\*tail，head[i]和 tail[i]就是第i个slabclass的 LRU 链表头尾了。LRU链表是按item元素使用时间降序排序的，头部的item表示最近刚被使用过，尾部的item是较长时间没有被用的。item每次被查询都会更新item使用时间以及在LRU链表中的位置，由于是双向链表，更新操作也是O(1)的。item需要使用内存的时候会先快速扫描对应slabclass的LRU链表尾部的一些元素（只检查尾巴几个），如果有已经过期的item，就重用这部分内存，不需要从slab上重新分配。如果没找到过期的，就从slab上分配内存，假如分配失败那么从LRU链表尾部找一块item内存来用。

LRU链表是用全局锁cache_lock保护的。在多线程下，get命令查数据不用加锁cache_lock，set命令先锁cache_lock，然后申请新的item内存，把LRU中item更新(此处忽略对Hash表操作)，原来的旧数据等待被回收到slabclass。旧数据的item内存具体何时被回收呢，这就轮到item的refcount出场了。item在LRU链表上初始refcount为1，当某个worker线程event loop中收到查询该item的请求，process_get_command函数会调用item_get接口，item_get会增加该item的refcount，变为2。同时查询hash表得到item数据后，收到这个item查询请求的socket连接的发送缓冲区就指向该item数据所在内存位置。此时refcount一直为2，表示该item使用的内存受保护着，不会因为LRU被重用，也不会被归还给slabclass。等到event loop的状态机把item数据全部发送给客户端之后，conn_release_items函数再调用item_remove接口减少refcount。refcount是保护item当前所用的这块内存的，保护数据还是需要互斥锁。

	/* 为item分配内存 */
	item *do_item_alloc(char *key, const size_t nkey, const int flags,
                    	    const rel_time_t exptime, const int nbytes,
                            const uint32_t cur_hv);
根据代码来说明下do_item_alloc这个函数的具体逻辑：

* 先计算这个item的总长度，应该放在哪个slabclass中，设slabclass数组下标为id
* 从该slabclass对应的LRU链表尾部即*tail[id]向前扫描5个（hard code）元素。如果item_lock被锁或者item refcount大于1则直接跳过。遇到没有被锁的item，就检查该item是否已失效。失效有两种可能：1，过期了；2，flush操作改变了oldest_live参数，导致这个时间之前的item全部失效。如果该item失效了，就可以重用这块内存，把item从hash表和LRU链表中删除，并做一些初始化工作就直接返回。
* 如果LRU尾部5个item没有失效的，那么就调用slabs_alloc函数从slabclass上分配内存。
* 如果从slabclass上申请内存失败，根据settings.evict_to_free参数，选择是否触发LRU。触发LRU则强制删除刚才找到的未被使用的item，重用其内存。同时更新itemstats[id].evicted（替换次数）统计信息。
* 如果LRU尾部5个item全部正在被使用（被锁或者recount大于1），那么直接从slabclass上申请内存，此时如果申请失败的话，LRU也回天乏力了，直接返回NULL。

**lazy expiration**
	/** 查询hash表加上item过期判断（lazy expiration） */
	item *do_item_get(const char *key, const size_t nkey, const uint32_t hv);
set命令可以item的过期时间，但是memcached对于过期的item并不是定时触发删除，而是等到item每次被get命令查询时候才判断是否已过期，过期的话从LRU链表和Hash表中删除，返回查不到数据。另外，memcached也支持发送命令主动触发删除过期item，详见protocol.txt中有关LRU_Crawler的说明。

**slab rebalance**

在memcached使用过程中，可能会出现各slabclass内存使用需求不平均的情况。假设一种极端情况，刚开始内存大部分都被分配给96 bytes的slabclass了，而后续需要存储的是大部分都是112 bytes的item，那么由于memcached不能把96 bytes的slabclass内存给112 bytes的item用，112 bytes的slabclass就会发生LRU替换，大大增加cache miss。slab rebalance就是为了解决这个问题，大体思路是从最近比较稳定（LRU item替换次数较少）的slabclass中牺牲一块slab出来，给急需内存的slabclass使用。前面在do_item_alloc函数逻辑中说到发生LRU替换的时候会统计slabclass的替换次数信息，在这里就可以用到了。memcached会开启两个线程，一个线程负责统计每个slabclass最近几个周期（5s）内LRU替换次数，选择最近3个周期内没有发生替换的slabclass作为source，最近3个周期内都是LRU替换最高的slabclass作为dest。如果找到了这样的source和dest，就用条件变量通知第二个线程。第二个线程负责source移动一块slab到dest中。移动的时候需要先清除原来的item，从LRU链表和Hash表中删除。具体就不放展开了，可以参考下这篇文章[Memcached slab move机制][5]。

最后，笔记中肯定有一些不对的地方，请以memcached源码为准，也欢迎拍砖

[1]: http://en.wikipedia.org/wiki/Slab_allocation
[2]: http://blog.csdn.net/baiduforum/article/details/6126337
[3]: http://www.nosqlnotes.net/archives/105
[4]: https://www.usenix.org/system/files/conference/nsdi13/nsdi13-final170_update.pdf
[5]: http://blog.chinaunix.net/uid-27767798-id-3404133.html
