# Virtual address space

Linux memory management adopts virtual address spaces, meaning that allocated
pages are identified by addresses that do not correspond (but are mapped) to
the underlying physical ones.

There is a separation between kernel and process memory space.
Each process is assigned with a set of numbered pages, mapped both to the
process and to the physical address space.

## Kernel space

The kernel space is divided in two parts: the logical and the virtual kernel
address spaces.
Logical pages are allocated with `kmalloc` and map one-to-one with physical
ones (just offset) as they reside in the first reserved RAM addresses.
Contiguously allocated pages will be contiguous both virtually and physically.
The virtual space (allocated with `vmalloc`) instead does not map directly to
physical addresses.

### `kmalloc`

```c
void *kmalloc(size_t size, gfp_t flags)
```

`kmalloc` is used for small allocations of continuous physical memory.
Properties of the memory location and allocation process change based on flags.

- `kmalloc` + `GFP_KERNEL`: default for small kernel objects (structs, lists),
    space is physically contiguous and provides fast access. It may sleep.
- `kmalloc` + `GFP_USER`: used to create user-space buffers with non-movable
    pages (memory shared by two processes, when DMA accesses are required).
    It may sleep.
- `kmalloc` + `GFP_ATOMIC`: used to allocate memory from interrupt handlers.
    It never sleeps

### Get free pages

```c
unsigned long __get_free_pages(gfp_t gfp_mask, unsigned int order)
```

`kmalloc` has some overhead and may lack controllability on some specific
actions.
`__get_free_pages` returns large chunks of contiguous pages free of built-in
protection mechanisms, giving the highest of control and performance but
requiring manual management, which if done wrong may lead to system
degradation.

### Allocate pages

```c
struct page *alloc_pages_node(int nid, gfp_t gfp_mask, unsigned int order)
```

In NUMA architectures (Non-Uniform Memory Access) pieces of memory can be tight
to the CPU they are closer to, to speedup accesses.
Other processors can access others' memory but access time increases; for this
reason, pages can be allocated providing the specific NUMA node (core) they get
assigned to.

### `vmalloc`

```c
void *vmalloc(unsigned long size)
```

`vmalloc` allocates a large virtually contiguous kernel heap with scattered
physical pages.
It is slower than other allocation methods as it needs to setup and modify
memory table mapping pages.
Furthermore, it may sleep, thus cannot be used in atomic contexts.

### Slab allocator

```c
void *kmem_cache_alloc(struct kmem_cache *cachep, gfp_t flags)
```

Page‑based allocation is not suitable when dealing with objects.
To handle fixed‑size data structures efficiently the kernel uses slabs.

A slab is an allocated space dedicated to store structs of a specific size.
The space allocated for the slab is subdivided in consecutive blocks of that
specific object size.
Each slab maintains a free-list, marking each internal block as used or not,
based on requested object allocations and frees.

As said, a slab is initialized at a given size, which may be multiple pages
long, but when it fills up, another can be instantiated.
Therefore, when allocating an object multiple slabs may be available for that
object size.
A struct named `kmem_cache` is used to list full, partial and empty slabs for
their specific object they map.
Multiple cache instances are created, one for each distinct object size.
Moreover, each NUMA node manages its own `kmem_cache` structs.

The slab allocator provides two main classes of caches: dedicated caches are
used to map commonly used kernel types with their optimized size; generic
caches are instead general purpose and provide slots of set sizes (usually
powers of two).

