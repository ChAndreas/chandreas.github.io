---
layout: post
title: "Introduction to Memory Management"
date:	Sat Apr 06 17:30:54 EEST 2019
---

**MMU**

The Memory Management Unit (MMU) is a hardware component which maps physical frames to virtual addresses. The MMU operates on basic units of memory called pages. Page size can be different base the architecture.

**Page**

Page is a unit of memory sized and aligned at the page size and page frame or frame refers to page size.

**Page Table**

A page table is the data structure used by a virtual memory system in a operating system to store the mapping between virtual addresses and physical addresses.

**Page Faults**

When a process accesses a region of memory that is not mapped or the processes has insufficient permissions for the address requested the MMU will generate a page fault exception.

**Page allocator**

The Linux page allocator is based on a buddy allocator, implemented in `mm/page_alloc.c` and tracks the free pages of different orders.

**Translation Lookaside Buffer (TLB)**

Translation lookaside buffer (TLB) is a memory cache that is used to reduce the time taken to access a user memory location. It is a part of the chip’s memory-management unit (MMU) and stores the recent translations of virtual memory to physical memory, also can be called an address-translation cache. TLB contains the virtual address, physical address and permissions of the mapping entries, if the the address is in the TLB the MMU will look up the physical resource, if not in TLB the MMU will generate a page fault exception and interrupt the CPU.

**Non-Uniform Memory Access Numa**

NUMA is used on multiprocessor systems whose memory is divided into multiple memory nodes. The access time of a memory node depends on the relative locations of the accessing CPU and accessed node, which it means that CPU access to memory depends on the distance cost between them. Memory is divided to node and each node is divided to many blocks called zones and a zone contains pages.

**Memory Zones**

Direct memory access (DMA) is a feature of computer systems that allows certain hardware subsystems to access main system memory (random-access memory), independent of the central processing unit (CPU). 
Examples of hardware systems that are using DMA are disk drive controllers, graphics cards, network cards and sound cards.
Linux kernel divides pages into different zones.

```bash
cat /proc/zoneinfo
```

`ZONE_DMA` - This zone contains pages that can undergo DMA

`ZONE_DMA32` - Like `ZOME_DMA`, this zone contains pages that can undergo DMA. Unlike `ZONE_DMA`, these pages are accessible only by 32-bit devices. On some architectures, this zone is a larger subset of memory.

`ZONE_NORMAL` - This zone contains normal, regularly mapped, pages.

**Linux Kernel Memory Management APIs**

**kmalloc()**

Kmalloc function is similar with userspace `malloc()` with the exception of the additional flag parameters. The `kmalloc()` function is a simple interface for obtaining kernel memory in byte-sized chunk and guarantees that the pages are physically contiguous (and virtually contiguous). 

Further similar to calloc on userspace is **kzalloc** where is setting the memory with zero.

Example
```c
p = kmalloc(sizeof(struct example), GFP_KERNEL); 
```
	
**gfp_t flags**

The flags are broken up into three categories: `action modifiers`, `zone modifiers`, and `types`. Action modifiers specify how the kernel is supposed to allocate the requested memory. Zone modifiers specify from which memory zone the allocation should originate and the type flags specify the required action and zone modifiers to fulfill a particular type of transaction.

Usual usage of flags

Process context which can sleep - `GFP_KERNEL`

Process context which cannot sleep - `GFP_ATOMIC`

Interrupt handler - `GFP_ATOMIC`

Softirq - `GFP_ATOMIC`

Tasklet -  `GFP_ATOMIC`

DMA-able memory which can sleep - (`GFP_DMA | GFP_KERNEL`)

DMA-able memory which cannot sleep - (`GFP_DMA | GFP_ATOMIC`)

*The `GFP_DMA` flag is used to specify that the allocator must satisfy the request from `ZONE_DMA`. This flag is used by device drivers, which need DMA-able memory for their devices, normally, you combine this flag with the `GFP_ATOMIC` or `GFP_KERNEL` flag.*

**kfree**

The kfree() method frees a block of memory previously allocated with kmalloc() similar with free that we can find in userland.

```c
kfree(p);
```

**vmalloc**

The vmalloc() function is similar to kmalloc(), except it allocates memory that is only virtually contiguous and not physically contiguous. The vmalloc() function ensures only that the pages are contiguous within the virtual address space and it does this by allocating potentially noncontinuous chunks of physical memory and "fixing up" the page tables to map the memory into a contiguous chunk of the logical address space.

```c
p = vmalloc(16 * PAGE_SIZE);
```

and free with

```c 
vfree(p);
```


**Slab Allocator**

The slab allocator is an abstraction layer to make easier allocation of numerous objects of a same type. 
The basic idea behind the slab allocator is to have caches of commonly used objects kept in an initialized state available for use by the kernel. The Linux kernel has three main different memory allocators: SLAB, SLUB, and SLOB. The slab layer acts as a generic data structure-caching layer. In this sense, the free list acts as an object cache, caching a frequently used type of object. 

Slab usage

`kmem_cache_create` use arguments such as name, size, flags and ctor (constructor function to call when new objects are added to the cache) to create the cache

The following flags can be used for `kmem_cache_create`

`SLAB_NO_REAP` - slab layer will reap objects in the cache when memory is low

`SLAB_HWCACHE_ALIGN` - align each object within a slab to different cache lines

`SLAB_MUST_HWCACHE_ALIGN` - debugging purpose

`SLAB_POISON` - fill memory with 0xa5a5a5a5, useful for catching access to uninitialized memory

`SLAB_RED_ZONE` - insert “red zone” around the allocated memory to detect buffer overruns

`SLAB_PANIC` - make slab panic() if allocation fails

`SLAB_CACHE_DMA` - memory allocated from the slab must come from ZONE_DMA


Example of *kmem_cache_create*

```c
struct kmem_cache *kmem_cache_create(const char *name, unsigned int size, unsigned int align,slab_flags_t flags, void (*ctor)(void *));
p_cache = kmem_cache_create("Example cache",sizeof(struct example),0,SLAB_HWCACHE_ALIGN,NULL_NULL);
```

Once a cache of objects is created, you can allocate objects from it by calling kmem_cache_alloc

```c
void *kmem_cache_alloc (struct kmem_cache * cachep, gfp_t flags);
p = kmem_cache_alloc(p_cache,GFP_KERNEL);
```

Deallocate an object

```c
void kmem_cache_free(struct kmem_cache *cachep, void *obj);
kmem_cache_free(p_cache, p);
```

Finally, at module unload time, we have to return the cache to the system:

```c
void kmem_cache_destroy(struct kmem_cache *s)
kmem_cache_destroy(p_cache);
```
Use ksize To find the actual amount of memory allocated for an object

```c
size_t ksize(const void *objp);
```

Further you can see the kernel slab cache information in real time with slabtop

References

	https://www.kernel.org/doc/htmldocs/kernel-api/mm.html
	https://elixir.bootlin.com/linux/v5.1-rc5/source/kernel/sched/core.c#L2837
	http://makelinux.net/ldd3/chp-8-sect-2.shtml
