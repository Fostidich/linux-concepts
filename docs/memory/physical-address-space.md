# Physical address space

## Buddy allocator

Physical memory is allocated big chunks at a time; to efficiently allocate many
smaller and different-size objects, the memory block gets partitioned with the
buddy algorithm.

First, round up the request to the next power-of-two size.
Check the free list for that size; if available, pop and use one block.
If empty, take a block from the next larger free list, recursively split into
buddies, push the unused buddies to their sizes list, and allocate the one you
need.

If no space is available, allocate pages to create a new buddy block.
When releasing a block, check if its buddy is also free, and in that case,
merge them into a one big free block, possibly going up recursively.

`kmalloc` is built on top of the slab allocator, itself sitting on top of the
buddy allocator.
`vmalloc` lives directly on top of the buddy allocator.

## Zonal page allocation

System's memory is partitioned among each NUMA node.
Each node has an instance in the `pgdata_list`, with which it manages its own
chunk of memory.

Each node's memory is partitioned in "zones", each being a physical memory
range addressed to specific use cases.
Each zone manages its free pages, storing page descriptors in a free list.

Essential zones are the `ZONE_NORMAL` (upper addresses) and `ZONE_DMA` (lower
addresses): the first is mapped by the kernel while the second is used by
devices.

## User space page caching

Physically allocated pages can be anonymous (swap) or linked to files.
Each page is cached in a `struct page` descriptor.

Two types of mapping exist.

### Forward mapping

Forward mapping is used to find the `struct page` given a virtual address.

For example, given a file `struct inode`, it contains a `struct address_space`,
which stores mappings between file's offsets and corresponding physical pages.

Anonymous pages (pages not linked to files) are linked to the swapfile; when
they can be swapped out, they enter the swap cache.
The swap cache is implemented using a single global `struct address_space`
named `swapper_space`, indexed by swap location rather than file offset.

Given a `struct file` or a swap location it is therefore possible to get the
corresponding underlying physical `struct page`.
This is useful for example to check residency: the `struct page` can tell if
its data is stored in RAM, if it's up to date or if it's to be reloaded from
disk.

### Reverse mapping

Reverse mapping from a `struct page` identifies VMAs mapping that physical
page.
This is necessary when evicting the page, as all referencing virtual
pages must be invalidated in the PTEs (Page Table Entry) of processes.
This struct also keeps flags representing the dirty state of the page.

The `struct page` contains a `mapping` pointer referencing either a file's
`address_space` or an anonymous memory range (`anon_vma`) of a process using
the page.
Via these struct it's possible to reach a list of VMAs using the page.
By walking these VMAs, each is checked if containing virtual pages pointing to
the evicted page, possibly removing it from PTEs.

## PFRA

The Page Frame Reclaim Algorithm is used to unload pages when free space
shrinks.
It is mainly operated by the `kswapd` daemon, which runs periodically, but
under high loads the buddy allocator can run it itself.
Pages are reclaimed via reverse mapping.

The reclaim policy is derived from the LRU-like "clock algorithm".
A circular list keeps pointers to all pages, each containing a R bit
representing recent references.
When a program uses a page, it sets R=1.
When in need of reclaiming a page, the page under the hand (index) is evicted
if R=0, otherwise, if R=1, R is set to 0 and the hand moves forward until a
reclaimable page is found.

File pages are majorly clean at the time of eviction, whilst anonymous ones
are always dirty.
For this reason, Linux keeps two different groups based on page type, to better
balance use cases.

Each group (file's and anonymous pages) applies the clock algorithm on two
different circular list: actives and inactives.
When a file is accessed, it is put in inactives with R=1.
When `shrink_list` is called, it runs the clock algorithm on inactives.
When `refill_inactive_zone` is called, it runs the algorithm on actives,
evicting pages to inactives.
`mark_page_accessed` sets R=1 on the page; when it gets called on a R=1
inactive page, that gets moved to actives, setting R=0.

The inactive list should to be kept balanced with the active one, with its size
matching the working set (number of "recently" used pages).
The PFRA algorithm can thus be called periodically reallocating actives to
inactives.
This is needed as a small inactive list can lead to thrashing, resulting in
premature evictions and frequent reloads.

