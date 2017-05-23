<meta http-equiv="Content-Type" content="text/html; charset=utf-8">

#Slub#
Author: Dalink

## 一、概述##

Linux使用Slub控制虚拟内存的分配。

## 二、结构体定义##

```C++
struct kmem_cache {
	struct kmem_cache_cpu __percpu *cpu_slab;	//CPU高速缓存
	int size;									//包含元数据的对象大小
	int object_size;							//不包含元数据的大小
	struct kmem_cache_order_objects oo;
	struct kmem_cache_order_objects max;
	struct kmem_cache_order_objects min;
}
struct kmem_cache_order_objects {
	unsigned long x;
};
struct kmem_cache_cpu {  
	void **freelist;          //指向本地CPU的第一个空闲对象 
	struct page *page;        //分配给本地CPU的slab的页框 
    struct page *partial;     //空闲对象列表
	int node;                 //页框所处的节点，值为-1时表示DEBUG 
	unsigned int offset;      //空闲对象指针的偏移，以字长为单位
	unsigned int objsize;     //对象的大小
}; 
struct kmem_cache_node {
    unsigned long nr_partial;
    struct list_head partial;
};
struct page {
	union {
        unsigned counters;
        struct {
            union {
                unsigned int active;        /* SLAB */
                struct {
                    unsigned inuse:16;      //已经使用的slab对象个数
                    unsigned objects:15;    //总共的slab对象个数
                    unsigned frozen:1;
                };
            };
        };
    };
};
```

## 三、全局变量##

```c++
#define KMALLOC_SHIFT_HIGH	(PAGE_SHIFT + 1)
struct kmem_cache *kmalloc_caches[KMALLOC_SHIFT_HIGH + 1];
```



## 四、初始化##



## 五、分配内存##

分配内存的调用函数为kmalloc。它的调用顺序为

```c++
kmalloc(size_t size, gfp_t flags)
    __kmalloc(size, flags)
        s = kmalloc_slab(size, flags)
        ret = slab_alloc(s, flags, _RET_IP_)
            slab_alloc_node(s, gfpflags, NUMA_NO_NODE, addr)
                __slab_alloc(s, gfpflags, node, addr, c)
                    ___slab_alloc(s, gfpflags, node, addr, c)
```

kmalloc_slab通过size获取索引index，然后index作为参数从kmalloc_caches数组中获取有效的kmem_cache。__kmalloc调用slab_alloc从这个kmem_cache中分配内存。linux目前可提供SLAB\SLOB\SLUB三种分配机制，我们使用的分配机制为SLUB。slab_alloc调用的slab_alloc_node位于slub.c文件中。同时，slab_alloc_node的node参数传入NUMA_NO_NODE。下面给出slab_alloc_node的简化代码：

```c++
static __always_inline void *slab_alloc_node(struct kmem_cache *s,
    gfp_t gfpflags, int node, unsigned long addr) {
  	//首先从高速缓存中获取
    struct kmem_cache_cpu *c = raw_cpu_ptr(s->cpu_slab);
    object = c->freelist;
    page = c->page;
    //我们传入的node的值为NUMA_NO_NODE
    if (unlikely(!object || !node_match(page, node))) {
        //如果获取不到一个可用的对象，或则页面不符合要求，则再次分配一个对象
        object = __slab_alloc(s, gfpflags, node, addr, c);
        stat(s, ALLOC_SLOWPATH);
    } else {
        //如果从高速缓存中获取到一个可用的对象，把下一个对象地址存入高速缓存中
		void *next_object = get_freepointer_safe(s, object);
		prefetch_freepointer(s, next_object);
    }
	return object;
}
```

如何判断一个page是否符合指定的node，node_match怎么工作的？node_match判断page是否存在，或则node不为NUMA_NO_NODE的时候，判断page的node号与传入的node号一致。我们这里的node值为NUMA_NO_NODE，显然只需要判断page是否存在。

```c++
static inline int node_match(struct page *page, int node)
{
#ifdef CONFIG_NUMA
    if (!page || (node != NUMA_NO_NODE && page_to_nid(page) != node))
        return 0;
#endif
    return 1;
}
```

___slab_alloc的简化代码如下。它从kmem_cache_cpu尝试获取一个快速页面，如果活到一个页面，然后对齐做node检测，。

当无法快速获取一个可用内存的时候，则执行goto new_slab`以从buddy系统获取一个可用对象。


```c++
static void *___slab_alloc(struct kmem_cache *s, gfp_t gfpflags, int node,
              unsigned long addr, struct kmem_cache_cpu *c)
{
    void *freelist;
    struct page *page;

    page = c->page;
    if (!page)
        goto new_slab;
redo:
    freelist = c->freelist;
    if (freelist)
    	//如果快速可用内存，则继续
        goto load_freelist;
	//取page的freelist
    freelist = get_freelist(s, page);

    if (!freelist) {
        c->page = NULL;
        stat(s, DEACTIVATE_BYPASS);
        goto new_slab;
    }
load_freelist:
    //将下一个可用的内存地址,即*(freelist + size)，装入快速分配列表
    c->freelist = get_freepointer(s, freelist);
    c->tid = next_tid(c->tid);
    return freelist;

//快速分配器已经没有空闲对象
new_slab:
    //如果本地空闲链表存在对象
    if (c->partial) {
        //则把空闲的一个页面装入快速分配页面page中
        page = c->page = c->partial;
        //并把这个页面从空闲列表中移除
        c->partial = page->next;
        stat(s, CPU_PARTIAL_ALLOC);
        //不知道
        c->freelist = NULL;
        goto redo;
    }
	//从buddy中分配页面
    freelist = new_slab_objects(s, gfpflags, node, &c);

    if (unlikely(!freelist)) {
	    //没有分配到页面
        return NULL;
    }

    page = c->page;
    if (likely(!kmem_cache_debug(s) && pfmemalloc_match(page, gfpflags)))
        goto load_freelist;

    deactivate_slab(s, page, get_freepointer(s, freelist));
    c->page = NULL;
    c->freelist = NULL;
    return freelist;
}
```

new_slab_objects的定义为

```c++
static inline void *new_slab_objects(struct kmem_cache *s, gfp_t flags, int node, struct kmem_cache_cpu **pc)
{
    struct kmem_cache_cpu *c = *pc;
    struct page *page;
    //这里看不懂
    void *freelist = get_partial(s, flags, node, c);
    if (freelist)
        return freelist;
	//获取新的页面
    struct page *page = new_slab(s, flags, node);
    if (page) {
        c = raw_cpu_ptr(s->cpu_slab);
        if (c->page)
	        //如果原来的c已经拥有page，则销毁
            flush_slab(s, c);
        freelist = page->freelist;
        //这里为何设置为null
        page->freelist = NULL;
		//把page存入c
        c->page = page;
    } else
        freelist = NULL;
    return freelist;
}
```

new_slab调用allocate_slab继续分配页面。allocate_slab调用多次alloc_slab_page完成页面分配。调用多次是因为它降低了分配级别，比如高级没有可用内存，就开始低速模式的分配。在调用alloc_slab_page函数时，传入oo对象。alloc_slab_page调用alloc_pages完成页面分配，页面分配的size为oo参数的16幂底。当分配得到内存之后，需要在这段内存中建立可用内存链表。这部分代码定义在slub.c文件的1587行。

```c++
//slub.c : 1587
//首先根据page对象获取该页面所对应的虚拟地址
start = page_address(page);

//s代表为kmem_cache，idx为int星数据,p为void*
for_each_object_idx(p, idx, s, start, page->objects) {
    setup_object(s, page, p);
    if (likely(idx < page->objects))
        set_freepointer(s, p, p + s->size);
    else
        set_freepointer(s, p, NULL);
}
//把当前page表达的首个slab对象地址存入page->freelist中。
page->freelist = fixup_red_left(s, start);
```


```c++
page->inuse = page->objects;
page->frozen = 1;
```

for_each_object_idx宏展开后为

    for (p = fixup_red_left(s, start), idx = 1; idx <= page->objects; p += (s)->size, idx++) {
        if (likely(idx < page->objects))
            set_freepointer(s, p, p + s->size);
        else
            set_freepointer(s, p, NULL);
    }

调用把下一个数据的地址装入slab对象，最后一个对象存储的地址为NULL。

```c++
//object为slab对象，fp为下一个对象的地址
static inline void set_freepointer(struct kmem_cache *s, void *object, void *fp)
{
    *(void **)(object + s->offset) = fp;
}
```

get_partial函数总是调用get_any_partial获取。还是看不懂
get_any_partial的简化代码为：

```c++
static void *get_any_partial(struct kmem_cache *s, gfp_t flags,
        struct kmem_cache_cpu *c)
{
    enum zone_type high_zoneidx = gfp_zone(flags);

    struct zoneref *z;
    struct zone *zone;
    struct zonelist *zonelist = node_zonelist(mempolicy_slab_node(), flags);
    for_each_zone_zonelist(zone, z, zonelist, high_zoneidx) {
        struct kmem_cache_node *n = get_node(s, zone_to_nid(zone));

        if (n && cpuset_zone_allowed(zone, flags) &&
                    n->nr_partial > s->min_partial) {
            void *object = get_partial_node(s, n, c, flags);
            if (object) {
                return object;
            }
        }
    }
    return NULL;
}
```

## 六、回收内存##

kfree调用栈如下：

```
void kfree(const void *x)
	slab_free(page->slab_cache, page, object, NULL, 1, _RET_IP_);
		do_slab_free(s, page, head, tail, cnt, addr)
			if (likely(page == c->page))
            	set_freepointer(s, tail_obj, c->freelist);
			else
            	__slab_free(s, page, head, tail_obj, cnt, addr);
```

kfree调用`slab_free(page->slab_cache, page, object, NULL, 1, _RET_IP_)`继续完成动作，这里slab_free函数的tail参数传入NULL。slab_free继而调用do_slab_free。do_slab_free函数首先得到当前释放对象的地址。通过如下：

    void *tail_obj = tail ? : head;
这里tail参数为NULL，所以tail_obj的值为head。继续，do_slab_free通过判断目标page是否是当前cpu的page。如果是当前cpu的page，则直接调用set_freepointer把释放的对象加入slab列表，这个情况十分简单。如果，继续调用__slab_free函数做进一步的释放。

```c++
//head tail cnt addr
static void __slab_free(struct kmem_cache *s, struct page *page,
			void *head, void *tail, int cnt,
			unsigned long addr)
{
	void *prior;
	int was_frozen;
	struct page new;
	unsigned long counters;
	struct kmem_cache_node *n = NULL;
	unsigned long uninitialized_var(flags);
	do {
		prior = page->freelist;
		counters = page->counters;
      	//把空闲对象链入Slab
		set_freepointer(s, tail, prior);
		new.counters = counters;
		was_frozen = new.frozen;
		new.inuse -= cnt;
		if ((!new.inuse || !prior) && !was_frozen) {
			if (kmem_cache_has_cpu_partial(s) && !prior) {
				new.frozen = 1;
			} else {
				n = get_node(s, page_to_nid(page));
				spin_lock_irqsave(&n->list_lock, flags);
			}
		}
	} while (!cmpxchg_double_slab(s, page,
		prior, counters,
		head, new.counters,
		"__slab_free"));

	if (likely(!n)) {
		if (new.frozen && !was_frozen) {
			put_cpu_partial(s, page, 1);
			stat(s, CPU_PARTIAL_FREE);
		}
		if (was_frozen)
			stat(s, FREE_FROZEN);
		return;
	}

	if (unlikely(!new.inuse && n->nr_partial >= s->min_partial))
		goto slab_empty;

	if (!kmem_cache_has_cpu_partial(s) && unlikely(!prior)) {
		if (kmem_cache_debug(s))
			remove_full(s, n, page);
		add_partial(n, page, DEACTIVATE_TO_TAIL);
		stat(s, FREE_ADD_PARTIAL);
	}
	spin_unlock_irqrestore(&n->list_lock, flags);
	return;

slab_empty:
	if (prior) {
		remove_partial(n, page);
		stat(s, FREE_REMOVE_PARTIAL);
	} else {
		remove_full(s, n, page);
	}
	stat(s, FREE_SLAB);
	discard_slab(s, page);
}
```

