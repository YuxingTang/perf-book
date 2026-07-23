## Virtual Memory 虚拟内存/虚存 {#sec:VirtMem}

Virtual memory is the mechanism to share the physical memory attached to a CPU with all the processes executing on it. Virtual memory provides a protection mechanism that prevents access to the memory allocated to a given process from other processes. Virtual memory also provides relocation, which is the ability to load a program anywhere in physical memory without changing the addresses in the program. 

虚拟内存是一种机制，它允许CPU上的所有进程共享物理内存。虚拟内存提供了一种保护机制，防止其他进程访问分配给特定进程的内存。虚拟内存还支持重定位，即程序可以在不更改程序地址的情况下加载到物理内存中的任何位置。

In a CPU that supports virtual memory, programs use virtual addresses for their accesses. But while user code operates on virtual addresses, retrieving data from memory requires physical addresses. Also, to effectively manage the scarce physical memory, it is divided into *pages*. Thus, applications operate on a set of pages that an operating system has provided.

在支持虚拟内存的CPU中，程序使用虚拟地址进行访问。但是，虽然用户代码操作的是虚拟地址，但从内存中检索数据需要使用物理地址。此外，为了有效地管理有限的物理内存，它被划分为*页page*。因此，应用程序操作的是操作系统提供的一组页。

![Virtual-to-physical address translation for 4KB pages. 4KB页的虚拟地址到物理地址转换。](../../img/uarch/VirtualMem.png){#fig:VirtualMem width=75%}

Virtual-to-physical address translation is required for accessing data as well as code (instructions). The translation mechanism for a system with a page size of 4KB is shown in Figure @fig:VirtualMem. The virtual address is split into two parts. The virtual page number (52 most significant bits) is used to index into the page table to produce a mapping between the virtual page number and the corresponding physical page. The 12 least significant bits are used to offset within a 4KB page. These bits do not require translation and are used "as-is" to access the physical memory location.

访问数据和代码（指令）都需要进行虚拟地址到物理地址的转换。图 @fig:VirtualMem 展示了页大小为4KB的系统转换机制。虚拟地址分为两部分。虚拟页号（最高52位）用于索引页表，从而建立虚拟页号与相应物理页之间的映射关系。最低12位用于在4KB页内偏移。这12位无需转换，可以直接用于访问物理内存位置。

![Example of a 2-level page table. 两级页表示例。](../../img/uarch/L2PageTables.png){#fig:L2PageTables width=90%}

The page table can be either single-level or nested. Figure @fig:L2PageTables shows one example of a 2-level page table. Notice how the address gets split into more pieces. The first thing to mention is that the 16 most significant bits are not used. This can seem like a waste of bits, but even with the remaining 48 bits we can address 256 TB of total memory (2^48^). Some applications use those unused bits to keep metadata, also known as *pointer tagging*.

页表可以是单级的，也可以是嵌套的。图 @fig:L2PageTables 展示了一个两级页表的示例。请注意地址是如何被分割成更多部分的。首先需要说明的是，最高16位未使用。这看起来似乎浪费了这些位，但即使只使用剩余的48位，我们也能寻址256TB的总内存(2^48^)。一些应用程序会使用这些未使用的位来保存元数据，也称为*指针标记（pointer tagging）*。

A nested page table is a radix tree that keeps physical page addresses along with some metadata. To find a translation within a 2-level page table, we first use bits 32..47 as an index into the Level-1 page table also known as the *page table directory*. Every descriptor in the directory points to one of the 2^16^ blocks of Level-2 tables. Once we find the appropriate L2 block, we use bits 12..31 to find the physical page address. Concatenating it with the page offset (bits 0..11) gives us the physical address, which can be used to retrieve the data from the DRAM.

嵌套页表是一个基数树，它保存物理页地址以及一些元数据。要在两级页表中查找转换，我们首先使用第32到47位作为索引，访问一级页表，也称为*页表目录（page table directory）*。目录中的每个描述符都指向二级页表的 2^16^ 个块之一。找到合适的二级块后，我们使用第12到31位来确定物理页地址。将其与页偏移量（第0到11位）连接起来，即可得到物理地址，该地址可用于从DRAM中检索数据。

The exact format of the page table is dictated by the CPU for reasons we will discuss a few paragraphs later. Thus the variations of page table organization are limited by what a CPU supports. Modern CPUs support both 4-level page tables with 48-bit pointers (256 TB of total memory) and 5-level page tables with 57-bit pointers (128 PB of total memory).

页表的具体格式由CPU决定，原因我们将在后面几段讨论。因此，页表组织方式的多样性受限于CPU的支持。现代CPU支持4级页表（48位指针，总内存256TB）和5级页表（57位指针，总内存128PB）。

Breaking the page table into multiple levels doesn't change the total addressable memory. However, a nested approach does not require storing the entire page table as a contiguous array and does not allocate blocks that have no descriptors. This saves memory space but adds overhead when traversing the page table.

将页表拆分为多个级别不会改变总可寻址内存。但是，嵌套方法不需要将整个页表存储为连续数组，也不会分配没有描述符的块。这节省了内存空间，但增加了遍历页表的开销。

Failure to provide a physical address mapping is called a *page fault*. It occurs if a requested page is invalid or is not currently in the main memory. The two most common reasons are: 1) the operating system committed to allocating a page but hasn't yet backed it with a physical page, and 2) an accessed page was swapped out to disk and is not currently stored in RAM.

未能提供物理地址映射称为“*缺页错误/页失效（page fault）*”。当请求的页面无效或当前不在主内存中时，就会发生缺页错误。最常见的两个原因是：1）操作系统已承诺分配页面，但尚未分配物理页面；2）访问的页面已换出到磁盘，当前未存储在RAM中。

### Translation Lookaside Buffer (TLB) 地址转换后备缓冲区(TLB) {#sec:TLBs}

A search in a hierarchical page table could be expensive, requiring traversing through the hierarchy potentially making several indirect accesses. Such a traversal is called a *page walk*. To reduce the address translation time, CPUs support a hardware structure called a translation lookaside buffer (TLB) to cache the most recently used translations. Similar to regular caches, TLBs are often designed as a hierarchy of L1 ITLB (Instructions), and L1 DTLB (Data), followed by a shared (instructions and data) L2 STLB.

在一个层次化页表中进行搜索可能开销很大，需要遍历整个层次结构，这可能会导致多次间接访问。这种遍历称为“*页遍历（page walk）*”。为了减少地址转换时间，CPU支持一种称为地址转换后备缓冲区(TLB: Translation Lookaside Buffer)的硬件结构，用于缓存最近使用的地址转换。与常规缓存类似，TLB通常设计为L1 ITLB（指令）和L1 DTLB（数据）的层次结构，之后是共享的（指令和数据）L2 STLB。

To lower memory access latency, the L1 cache lookup can be partially overlapped with the DTLB lookup thanks to a constraint on the cache associativity and size that allows the L1 set selection without the physical address.[^1] However, higher level caches (L2 and L3) - which are also typically Physically Indexed and Physically Tagged (PIPT) but cannot benefit from this optimization - therefore require the address translation before the cache lookup.

为了降低内存访问延迟，由于缓存关联度和大小的限制，L1缓存查找可以与DTLB查找部分重叠，从而允许在不使用物理地址的情况下选择L1缓存集[^1]。然而，更高级别的缓存（L2和L3）——它们通常也采用物理索引和物理标记(PIPT: Physically Indexed and Physically Tagged)技术，但无法从这种优化中受益——因此需要在缓存查找之前进行地址转换。

The TLB hierarchy keeps translations for a relatively large memory space. Still, misses in the TLB can be very costly. To speed up the handling of TLB misses, CPUs have a mechanism called a *hardware page walker*. Such a unit can perform a page walk directly in hardware by issuing the required instructions to traverse the page table, all without interrupting the kernel. This is the reason why the format of the page table is dictated by the CPU, to which operating systems must comply. High-end processors have several hardware page walkers that can handle multiple TLB misses simultaneously. However, even with all the acceleration offered by modern CPUs, TLB misses still cause performance bottlenecks for many applications.

TLB层级结构维护着相对较大的内存空间的转换信息。然而，TLB未命中会造成非常大的开销。为了加速TLB未命中的处理，CPU提供了一种称为“*硬件页遍历器（hardware page walker）*”的机制。这种单元可以直接在硬件中执行页遍历，通过发出必要的指令来遍历页表，而无需中断OS内核。这就是为什么页表的格式由CPU决定，操作系统必须遵守的原因。高端处理器拥有多个硬件页遍历器，可以同时处理多个TLB未命中。然而，即使现代CPU提供了各种加速，TLB未命中仍然会成为许多应用程序的性能瓶颈。

### Huge Pages 巨页 {#sec:ArchHugePages}

Having a small page size makes it possible to manage the available memory more efficiently and reduce fragmentation. The drawback though is that it requires having more page table entries to cover the same memory region. Consider two page sizes: 4KB, which is a default on x86, and a 2MB *huge page* size. For an application that operates on 10MB of data, we need 2560 entries in the first case, but just 5 entries if we would map the address space onto huge pages. Those are named *Huge Pages* on Linux, *Super Pages* on FreeBSD, and *Large Pages* on Windows, but they all mean the same thing. Throughout the rest of this book, we will refer to them as Huge Pages.

较小的页面大小可以更有效地管理可用内存并减少碎片。但缺点是，覆盖相同的内存区域需要更多的页表项。考虑两种页面大小：4KB（x86结构的默认值）和2MB的“*巨页（huge page）*”。对于一个处理10MB数据的应用程序，第一种情况需要2560个条目，但如果将地址空间映射到巨页，则只需要5个条目。在Linux系统中，更大的页面被称为“*巨页（Huge Pages）*”，在FreeBSD系统中被称为“*超级页（Super Pages）*”，在 Windows系统中被称为“*大页Large Pages）*”（，但它们指的都是同一件事。本书后续部分将统一使用“巨页”这一术语。

An example of an address that points to data within a huge page is shown in Figure @fig:HugePageVirtualAddress. Just like with a default page size, the exact address format when using huge pages is dictated by the hardware, but luckily we as programmers usually don't have to worry about it.

图 @fig:HugePageVirtualAddress 展示了一个指向巨页内数据的地址示例。与默认页面大小一样，使用巨页时的具体地址格式由硬件决定，但幸运的是，作为程序员，我们通常无需担心这一点。

![Virtual address that points within a 2MB page. 指向 2MB 页内的虚拟地址。](../../img/uarch/HugePageVirtualAddress.png){#fig:HugePageVirtualAddress width=90%}

Using huge pages drastically reduces the pressure on the TLB hierarchy since fewer TLB entries are required. It greatly increases the chance of a TLB hit. We will discuss how to use huge pages to reduce the frequency of TLB misses in [@sec:secDTLB] and [@sec:FeTLB]. The downsides of using huge pages are memory fragmentation and, in some cases, nondeterministic page allocation latency because it is harder for the operating system to manage large blocks of memory and to ensure effective utilization of available memory. To satisfy a 2MB huge page allocation request at runtime, an OS needs to find a contiguous chunk of 2MB. If this cannot be found, the OS needs to reorganize the pages, resulting in a longer allocation latency.

使用巨页可以显著降低TLB层次结构的压力，因为所需的TLB条目更少。这大大提高了TLB命中的概率。我们将在 [@sec:secDTLB] 和 [@sec:FeTLB] 中讨论如何使用巨页来降低TLB未命中的频率。使用巨页的缺点是内存碎片化，并且在某些情况下，由于操作系统更难管理大块内存并确保有效利用可用内存，因此会导致页面分配延迟不确定。为了在运行时满足2MB巨页的分配请求，操作系统需要找到一块连续的2MB内存块。如果找不到，操作系统需要重新组织页面，从而导致更长的分配延迟。

[^1]: Minimum associativity for a PIPT L1 cache to also be VIPT, accessing a set without translating the index to physical 最小关联度的PIPT L1缓存同时也是VIPT，访问集合时无需将索引转换为物理地址索引 - [https://stackoverflow.com/a/59288185](https://stackoverflow.com/a/59288185).
