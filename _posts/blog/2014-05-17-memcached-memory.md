---
layout: post
title: Memcached源码学习笔记-内存管理
description: Memcached源码学习笔记-内存管理
category: blog
---

Memcached作为一个高性能的内存缓存server，使用了[Slab allocation][1]内存池方式管理内存。避免了直接使用glibc通用内存分配器malloc和free可能会导致的[碎片][2]，["内存泄漏"][3]这些问题。Slab allocation原理是先从操作系统中预分配一大块内存，然后将分配的内存分割成一系列固定尺寸大小的块(chunk)，并把尺寸相同的块集合成组(slabclass)。需要使用内存的时候从内存池中取一块大小合适的内存，用完了还给内存池循环利用。

如下图，可以把memcached使用的内存看成一个slabclass数组。每一个元素slabclass[i]里都有若干slab，每一个slab是1M大小的内存。这些1M大小的slab进一步被切分成大小相等的chunk，这些chunk就是用来存储item的，通常chunk比item略大。每个slabclass中所有chunk大小是一样的，不同slabclass之间chunk大小和settings结构体里factor变量有关。factor是用来控制相邻slabclass之间chunk大小增长比例的，可以自定义，默认为 1.25。如果 slabclass[0]里chunk大小为 88B，那么88 * 1.25 = 110，另外chunk大小需要8字节的倍数，slabclass[1]里chunk大小就是112B。Facebook使用memcached的factor取值为1.07，目的是让64 Byte和128 Byte可以在chunk size离散值范围内，匹配CPU cache line，详见[Scaling Memcache at Facebook][4]。使用内存的时候，memcached根据收到的item数据大小，选择大小最合适的slabclass。由于chunk size是一系列预先固定的离散值，所以chunk中无法避免存在着内部碎片。
![slab_overview](/images/memcached_memory/slab_overview.jpg)


代码主要有slab.c和item.c两部分，slab.c中是slab内存操作相关的函数，为item.c中item存取操作函数提供接口。先来看一下slabclass和item的结构定义：
	typedef struct {
	    unsigned int size;      /* sizes of items，该slabclass的chunk size（可存放的item最大size） */
	    unsigned int perslab;   /* how many items per slab， 每个slab可以放多少chunk */

	    void *slots;           /* list of item ptrs，内存可用的item链表 */
	    unsigned int sl_curr;   /* total free items in list，可用链表item元素的总量 */

	    unsigned int slabs;     /* how many slabs were allocated for this class， slabclass中内存已被使用的slab数量 */

	    void **slab_list;       /* array of slab pointers， slab指针数组 */
	    unsigned int list_size; /* size of prev array， slab指针数组长度，当slabs==list_size的时候，需要realloc一个slab指针数组 */

	    unsigned int killing;  /* index+1 of dying slab, or zero if none，slab rebalance的时候用到 */
	    size_t requested; /* The number of requested bytes， 已被使用的内存总量 */
	} slabclass_t;

	static slabclass_t slabclass[MAX_NUMBER_OF_SLAB_CLASSES];

	/**
	 * Structure for storing items within memcached.
	 */
	typedef struct _stritem {
	    struct _stritem *next;
	    struct _stritem *prev;	//slab可用内存和LRU都是通过双向链表来维护
	    struct _stritem *h_next;    /* hash chain next */
	    rel_time_t      time;       /* least recent access */
	    rel_time_t      exptime;    /* expire time */
	    int             nbytes;     /* size of data */
	    unsigned short  refcount;
	    uint8_t         nsuffix;    /* length of flags-and-length string */
	    uint8_t         it_flags;   /* ITEM_* above */
	    uint8_t         slabs_clsid;/* which slab class we're in */
	    uint8_t         nkey;       /* key length */
	    union {
		uint64_t cas;
		char end;
	    } data[];
	} item;

每个slabclass通过slots指针指向内存可用的item双向链表，sl_curr表示该链表中item的总数量。因为slabclass中chunk size固定，所以分配slab内存的时候不需要像通用内存分配器中first-fit，best-fit那样查找链表。使用内存就直接从对应slabclass的可用item链表拿第一个节点来用，归还内存的时候，把item插入到链表头即可，申请和归还都是O(1)操作。

	void slabs_init(const size_t limit, const double factor, const bool prealloc);
先来看初始化slab的slabs_init函数，这个函数首先根据factor参数为每个slabclass设置size和perslab信息。如果prealloc为true指定了预分配的话就会预先从操作系统分配一大块内存，然后调用slabs_preallocate函数为每一个slabclass初始化。为每一个slabclass调用do_slabs_newslab分配一个slab，即从大块内存中拿1M给一个slab。do_slabs_newslab中还会调用grow_slab_list初始化slab_list这个指针数组，初始为16个元素，后续不够了再realloc重新分配。slab内存分配完成之后，再通过split_slab_page_into_freelist函数，把slab的的所有item加到slabclass的slots可用链表上。至此，整个slab_init初始化流程就完成了，chunk size为96的slabclass内存结构示意图如下图，其他slabclass也类似。需要注意的是，指定prealloc预分配的话需要操作系统内存管理支持设置大页面。见memcached.c中的enable_large_pages函数。主要是为了减少TLB cache miss，因为如果从操作系统预分配大块内存，还使用4KB内存页的话，进程页表会很大，不能全部放在cpu cache中。

![slabclass](/images/memcached_memory/slabclass.jpg)

当然，并不是每次item需要使用内存就从slabclass中分配。每一个 slabclass 集合都有一个LRU双向链表,以 head 开头,tail 结尾。这个链表并不定义在 slabclass 结构体里,它是全局变量——**head 与**tail。于是 head[i]和 tail[i]就是第 i 个slabclass的 LRU 链表头尾了。LRU链表是按item元素使用时间排序的，头部的item表示最近刚被使用过，尾部的item是较长时间没有被用的。item每次被查询都会更新item使用时间以及在LRU链表中的位置，由于是双向链表，更新操作也是O(1)的。item需要使用内存的时候会先快速扫描对应slabclass的LRU链表尾部的一些元素（只检查尾巴几个），如果有已经过期的item，就重用这部分内存，不需要从slab上重新分配。如果没找到过期的，就从slab上分配内存，假如分配失败那么从LRU链表尾部找一块item内存来用。

每一个 slabclass 集合都有一个双向链表,以 head 开头,tail 结尾。这个链表并不定义在 slabclass 结构体里,它是全局变量——**head 与**tail。于是 head[i]和 tail[i]就是第 i 个集合的 LRU 链表头尾了。

do_slabs_alloc如果当前slabclass可用链表上有元素就拿一个返回，没有就调用do_slabs_newslab分配一个新的slab再从可用链表上拿一个元素返回。

do_slabs_free把item内存归还，插入到slabclass的slots可用链表上，item加标记ITEM_SLABBED。

slabs_alloc，slabs_free是提供给item使用的版本加了slab_lock。

slabs_clsid()
计算出哪个slabclass适合用来储存大小给定为size的item


[1]: http://en.wikipedia.org/wiki/Slab_allocation
[2]: http://blog.csdn.net/baiduforum/article/details/6126337
[3]: http://www.nosqlnotes.net/archives/105
[4]: https://www.usenix.org/system/files/conference/nsdi13/nsdi13-final170_update.pdf
