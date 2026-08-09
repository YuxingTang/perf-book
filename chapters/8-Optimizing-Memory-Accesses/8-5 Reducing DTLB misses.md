## Reducing DTLB Misses 减少DTLB失效/未命中 {#sec:secDTLB}

As discussed in [@sec:TLBs], TLB is a fast but finite per-core cache for virtual-to-physical address translations of memory addresses. Without it, every memory access by an application would require a time-consuming page walk of the kernel page table to calculate the correct physical address for each referenced virtual address. In a system with a 5-level page table, it will require accessing at least 5 different memory locations to obtain an address translation. In section [@sec:FeTLB] we will discuss how huge pages can be used for code. Here we will see how they can be used for data.

如 [@sec:TLBs] 中所述，TLB是一个快速但容量有限的单核缓存，用于将内存地址从虚拟地址转换为物理地址。如果没有TLB，应用程序的每次内存访问都需要耗时地遍历内核页表，以计算每个引用的虚拟地址对应的正确物理地址。在一个5级页表的系统中，至少需要访问5个不同的内存位置才能获得地址转换结果。在 [@sec:FeTLB] 节中，我们将讨论如何将大页用于代码。这里我们将探讨如何将大页用于数据。

Any algorithm that does random accesses into a large memory region will likely suffer from DTLB misses. Examples of such applications are binary search in a big array, accessing a large hash table, and traversing a graph. The usage of huge pages has the potential to speed up such applications.

任何对大内存区域进行随机访问的算法都可能遭受DTL​​B失效/未命中。此类应用的示例包括在大数组中进行二分查找、访问大型哈希表以及遍历图。使用大页有可能加速此类应用。

On x86 platforms, the default page size is 4KB. Consider an application that frequently references memory space of 20 MBs. With 4KB pages, the OS needs to allocate many small pages. Also, the process will be touching many 4KB-sized pages, each of which will contend for a limited number of TLB entries. In contrast, using huge 2MB pages, 20MB of memory can be mapped with just ten pages, whereas with 4KB pages, you would need 5120 pages. This means fewer TLB entries are needed when using huge pages, which in turn reduces the number of TLB misses. It will not be a proportional reduction by a factor of 512 since the number of 2MB entries is much less. For example, in Intel's Skylake core families, L1 DTLB has 64 entries for 4KB pages and only 32 entries for 2MB pages. Besides 2MB huge pages, x86-based chips from AMD and Intel also support 1GB gigantic pages for data, but not for instructions. Using 1GB pages instead of 2MB pages reduces TLB pressure even more.

在x86平台上，默认页面大小为4KB。假设一个应用程序频繁访问20MB的内存空间。如果使用4KB的页面，操作系统需要分配许多小页面。此外，该进程还会访问许多4KB大小的页面，每个页面都会争用数量有限的TLB条目。相比之下，如果使用2MB的大页面，只需10个页面即可映射20MB的内存，而使用4KB的页面则需要5120个页面。这意味着使用大页面时所需的TLB条目更少，从而减少了TLB未命中次数。但这种减少并非成比例地减少512倍，因为2MB的条目数量要少得多。例如，在Intel的Skylake核心系列中，4KB页面的L1 DTLB有64个条目，而2MB页面只有32个条目。除了2MB的大页面（huge page）之外，AMD和Intel的x86芯片也支持1GB的巨型页面（gigantic page）用于数据，但不支持用于指令。使用1GB页面而非2MB页面可以进一步降低TLB压力。

Utilizing huge pages typically leads to fewer page walks, and the penalty for walking the kernel page table in the event of a TLB miss is reduced since the table itself is more compact. Performance gains from utilizing huge pages can sometimes go as high as 30%, depending on how much TLB pressure an application is experiencing. Expecting 2x speedups would be asking too much, as it is quite rare that TLB misses are the primary bottleneck. The paper [@Luo2015] presents the evaluation of using huge pages on the SPEC2006 benchmark suite. Results can be summarized as follows. Out of 29 benchmarks in the suite, 15 have a speedup within 1%, which can be discarded as noise. Six benchmarks have speedups in the range of 1%-4%. Four benchmarks have speedups in the range from 4% to 8%. Two benchmarks have speedups of 10%, and the two benchmarks that gain the most enjoyed 22% and 27% speedups respectively.

使用大页面通常会减少页面遍历次数，并且由于页面表本身更加紧凑，因此在发生TLB未命中时遍历内核页面表的开销也会降低。使用大页面带来的性能提升有时可高达30%，具体取决于应用程序所面临的TLB压力。期望2倍的加速是不切实际的，因为 TLB 未命中很少成为主要瓶颈。论文 [@Luo2015] 对在SPEC2006基准测试套件上使用大页面进行了评估。结果总结如下：在套件中的29个基准测试中，15个测试的加速比在1%以内，可以忽略不计。6个测试的加速比在 1%到4%之间。4个测试的加速比在4%到8%之间。两项基准测试的加速比为10%，而加速比最高的两项基准测试分别达到了22%和27%。

Many real-world applications already take advantage of huge pages, for example, KVM, MySQL, PostgreSQL, Java's JVM, and others. Usually, those software packages provide an option to enable that feature. Whenever you're using a similar application, check its documentation to see if you can enable huge pages.

许多实际应用已经利用了大页内存，例如：KVM、MySQL、PostgreSQL、Java的JVM等。通常，这些软件包都提供了启用该功能的选项。当您使用类似的应用程序时，请查阅其文档，了解是否可以启用大页内存。

Both Windows and Linux allow applications to establish huge-page memory regions. Instructions on how to enable huge pages for Windows and Linux can be found in Appendix B. On Linux, there are two ways of using huge pages in an application: Explicit and Transparent Huge Pages. Windows support is not as rich as Linux and will be discussed later.

Windows和Linux都允许应用程序创建大页内存区域。有关如何在Windows和Linux中启用大页内存的说明，请参见附录B。在Linux中，应用程序可以使用两种方式来使用大页内存：显式大页和透明大页。Windows对大页内存的支持不如Linux完善，我们将在后面讨论。

### Explicit Huge Pages 显式大页面

Explicit Huge Pages (EHP) are available as part of the system memory, and are exposed as a huge page file system `hugetlbfs`. EHPs should be reserved either at system boot time or before an application starts. See Appendix B for instructions on how to do that. Reserving EHPs at boot time increases the possibility of successful allocation because the memory has not yet been significantly fragmented. Explicitly preallocated pages reside in a reserved chunk of physical memory and cannot be swapped out under memory pressure. Also, this memory space cannot be used for other purposes, so users should be careful and reserve only the number of pages they require.

显式大页(EHP: Explicit Hug Pages)作为系统内存的一部分可用，并以大页文件系统 `hugetlbfs` 的形式公开。EHP应在系统启动时或应用程序启动前预留。有关如何操作的说明，请参阅附录B。在启动时预留EHP可以提高分配成功的概率，因为此时内存尚未出现明显的碎片。显式预分配的页面位于物理内存的预留块中，不会在内存压力下被换出。此外，这部分内存空间不能用于其他用途，因此用户应谨慎操作，仅预留所需的页面数量。

The simplest method of using EHP in a Linux application is to call `mmap` with `MAP_HUGETLB` as shown in [@lst:ExplicitHugepages1]. In this code, the pointer `ptr` will point to a 2MB region of memory that was explicitly reserved for EHPs. Notice, that allocation may fail if the EHPs were not reserved in advance. Other less popular ways to use EHPs in user code are provided in Appendix B. Also, developers can write their own arena-based allocators that tap into EHPs.

在Linux应用程序中使用EHP的最简单方法是调用带有 `MAP_HUGETLB` 参数的 `mmap` 函数，如 [@lst:ExplicitHugepages1] 所示。在该代码中，指针 `ptr` 将指向显式为EHP预留的2MB内存区域。请注意，如果未预先预留EHP，则分配可能会失败。附录B中提供了其他一些不太常用的用户代码中使用EHP的方法。此外，开发者还可以编写自己的基于Arena的内存分配器来利用EHP。

Listing: Mapping a memory region from an explicitly allocated huge page.
代码列表：从显式分配的大页映射内存区域。

~~~~ {#lst:ExplicitHugepages1 .cpp}
void ptr = mmap(nullptr, size, PROT_READ | PROT_WRITE,
                MAP_PRIVATE | MAP_ANONYMOUS | MAP_HUGETLB, -1, 0);
if (ptr == MAP_FAILED)
  throw std::bad_alloc{};                
...
munmap(ptr, size);
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

In the past, there was an option to use the [libhugetlbfs](https://github.com/libhugetlbfs/libhugetlbfs)[^1] library, which overrode `malloc` calls used in existing dynamically linked executables, to allocate memory in EHPs. Unfortunately, this project is no longer maintained. It didn't require users to modify the code or to relink the binary. They could simply prepend the command line with `LD_PRELOAD=libhugetlbfs.so HUGETLB_MORECORE=yes <your app command line>` to make use of it. But luckily, other libraries enable the use of huge pages (not EHPs) with `malloc`, which we will discuss next.

过去，可以使用 [libhugetlbfs](https://github.com/libhugetlbfs/libhugetlbfs)[^1] 库来重载现有动态链接可执行文件中使用的 `malloc` 调用，从而在EHP中分配内存。可惜的是，该项目已停止维护。它无需用户修改代码或重新链接二进制文件，只需在命令行前添加 `LD_PRELOAD=libhugetlbfs.so HUGETLB_MORECORE=yes <你的应用程序命令行>` 即可使用。不过幸运的是，其他库也支持使用 `malloc` 来分配大页（而非EHP），我们将在下文中讨论。

### Transparent Huge Pages 透明大页

Linux also offers Transparent Huge Page Support (THP), which has two modes of operation: system-wide and per-process. When THP is enabled system-wide, the kernel manages huge pages automatically and it is transparent for applications. The OS kernel tries to assign huge pages to any process when large blocks of memory are needed and it is possible to allocate such, so huge pages do not need to be reserved manually. If THP is enabled per process, the kernel only assigns huge pages to individual processes' memory areas attributed to the `madvise` system call. You can check if THP is enabled in the system with:

Linux还提供了透明大页(THP: Transparent Huge Page)支持，它有两种运行模式：系统级和进程级。当THP以系统级启用时，内核会自动管理大页，这对应用程序来说是透明的。当需要大块内存且内存分配可行时，操作系统内核会尝试为进程分配大页，因此无需手动预留大页。如果每个进程都启用了THP，内核只会将大页分配给由 `madvise` 系统调用指定的进程内存区域。您可以使用以下命令检查系统中是否启用了THP：

```bash
$ cat /sys/kernel/mm/transparent_hugepage/enabled
always [madvise] never
```

The value shown in brackets is the current setting. If this value is `always` (system-wide) or `madvise` (per-process), then THP is available for your application. A detailed specification for every option can be found in the Linux kernel [documentation](https://www.kernel.org/doc/Documentation/vm/transhuge.txt)[^2] regarding THP. 

括号中显示的值是当前设置。如果此值为 `always`（系统级）或 `madvise`（进程级），则您的应用程序可以使用THP。有关每个选项的详细规范，请参阅Linux内核关于THP的文档 [文档](https://www.kernel.org/doc/Documentation/vm/transhuge.txt)[^2]。

When THP is enabled system-wide, huge pages are used automatically for normal memory allocations, without an explicit request from applications. To observe the effect of huge pages on their application, a user just needs to enable system-wide THPs with `echo "always" | sudo tee /sys/kernel/mm/transparent_hugepage/enabled`. It will automatically launch a daemon process named `khugepaged` which starts scanning the application’s memory space to promote regular pages to huge pages. Sometimes the kernel may fail to combine multiple regular pages into a huge page in case it cannot find a contiguous 2MB chunk of memory.

当THP在系统级启用时，系统会自动使用大页进行常规内存分配，而无需应用程序显式请求。要观察大页对其应用程序的影响，用户只需使用 `echo "always" | sudo tee /sys/kernel/mm/transparent_hugepage/enabled` 命令启用系统级THP。这将自动启动一个名为 `khugepaged` 的守护进程，该进程会开始扫描应用程序的内存空间，将常规页面提升为大页。有时，如果内核找不到连续的2MB内存块，则可能无法将多个常规页面合并为一个大页。

System-wide THPs mode is good for quick experiments to check if huge pages can improve performance. It works automatically, even for applications that are not aware of THPs, so developers don't have to change the code to see the benefit of huge pages for their application. When huge pages are enabled system-wide, applications may end up allocating more memory resources than needed. This is why the system-wide mode is disabled by default. Don't forget to disable system-wide THPs after you've finished your experiments as it may hurt overall system performance.

系统级透明大页(STHP: System-wide THP)模式非常适合快速实验，以检验大页是否能提升性能。即使对于不支持THP的应用程序，它也能自动运行，因此开发者无需修改代码即可体验大页带来的优势。启用系统级大页后，应用程序可能会分配超出实际需求的内存资源。因此，系统级模式默认处于禁用状态。实验结束后，请务必禁用系统级THP，因为它可能会影响系统整体性能。

With the `madvise` (per-process) option, THP is enabled only inside memory regions attributed via the `madvise` system call with the `MADV_HUGEPAGE` flag. As shown in [@lst:TransparentHugepages1], the pointer `ptr` will point to a 2MB region of the anonymous (transparent) memory region, which the kernel allocates dynamically. The `mmap` call will fail if the kernel cannot find a contiguous 2MB chunk of memory.

使用 `madvise`（进程级per-process）选项，THP仅在通过 `madvise` 系统调用并带有 `MADV_HUGEPAGE` 标志指定的内存区域内启用。如 [@lst:TransparentHugepages1] 所示，指针 `ptr` 将指向匿名（透明）内存区域中2MB的区域，该区域由内核动态分配。如果内核找不到连续的2MB内存块，`mmap` 调用将会失败。

Listing: Mapping a memory region to a transparent huge page.
代码列表：将内存区域映射到透明大页。

~~~~ {#lst:TransparentHugepages1 .cpp}
void ptr = mmap(nullptr, size, PROT_READ | PROT_WRITE | PROT_EXEC,
                MAP_PRIVATE | MAP_ANONYMOUS, -1 , 0);
if (ptr == MAP_FAILED)
  throw std::bad_alloc{};
madvise(ptr, size, MADV_HUGEPAGE);
// use the memory region `ptr`
munmap(ptr, size);
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Developers can build custom THP allocators based on the code in [@lst:TransparentHugepages1]. But also, it's possible to use THPs inside `malloc` calls that their application is making. Many memory allocation libraries provide that feature by overriding the `libc`'s implementation of `malloc`. Here is an example of using `jemalloc`, which is one of the most popular options. If you have access to the source code of the application, you can relink the binary with an additional `-ljemalloc` option. This will dynamically link your application against the `jemalloc` library, which will handle all the `malloc` calls. Then use the following option to enable THPs for heap allocations:

开发者可以基于 [@lst:TransparentHugepages1] 中的代码构建自定义的THP分配器。此外，也可以在应用程序的 `malloc` 调用中使用THP。许多内存分配库通过重写 `libc` 的 `malloc` 实现来提供此功能。以下是使用 `jemalloc` 的示例，它是最常用的选项之一。如果您可以访问应用程序的源代码，则可以使用额外的 `-ljemalloc` 选项重新链接二进制文件。这将动态地将您的应用程序链接到 `jemalloc` 库，该库将处理所有 `malloc` 调用。然后使用以下选项为堆分配启用THP：

```bash
$ MALLOC_CONF="thp:always" <your app command line>
```

If you don't have access to the source code, you can still make use of `jemalloc` by preloading the dynamic library:

即使无法访问源代码，您仍然可以通过预加载动态库来使用 `jemalloc`：

```bash
$ LD_PRELOAD=/usr/local/libjemalloc.so.2 MALLOC_CONF="thp:always" <your app command line>
```

Windows only offers using huge pages in a way similar to the Linux THP per-process mode via the `VirtualAlloc` system call. See details in Appendix B.

Windows仅通过 `VirtualAlloc` 系统调用以类似于Linux进程级透明大页(per-process mode THP)的方式提供大页的使用。详情请参见附录B。

### Explicit vs. Transparent Huge Pages 显式大页与透明大页

Linux users can use huge pages in three different modes:

Linux用户可以使用三种不同的模式使用大页：

* Explicit Huge Pages
* System-wide Transparent Huge Pages
* Per-process Transparent Huge Pages

* 显式大页(EHP: Explicit Huge Pages)
* 系统级透明大页(STHP: System-wide Transparent Huge Pages)
* 进程级透明大页(PTHP: Per-process Transparent Huge Pages)

Let's compare those options. First, EHPs are reserved in virtual memory upfront, THPs are not. That makes it harder to ship software packages that use EHPs, as they rely on specific configuration settings made by an administrator of a machine. Moreover, EHPs statically sit in memory, consuming precious DRAM, even when they are not used.

让我们比较一下这些选项。首先，EHP会预先在虚拟内存中预留空间，而THP则不会。这使得使用EHP的软件包更难发布，因为它们依赖于机器管理员进行的特定配置设置。此外，即使不使用，EHP也会静态地驻留在内存中，占用宝贵的 DRAM。

System-wide Transparent Huge Pages are great for quick experiments. No changes in the user code are required to test the benefit of using huge pages in your application. However, it will not be wise to ship a software package to the customers and ask them to enable system-wide THPs, as it may negatively affect other running programs on that system. Usually, developers identify allocations in the code that could benefit from huge pages and use `madvise` hints in these places (per-process mode).

系统级透明大页非常适合快速实验。无需更改用户代码即可测试在应用程序中使用大页的优势。然而，将软件包交付给客户并要求他们启用系统级透明大页（STHP）并不明智，因为这可能会对系统上其他正在运行的程序产生负面影响。通常，开发人员会在代码中识别出可以从大页中受益的内存分配，并在这些地方使用 `madvise` 提示（进程级模式）。

Per-process THPs don't have either of the downsides mentioned above, but they have another one. Previously we discussed that THP allocation by the kernel happens transparently to the user. The allocation process can potentially involve several kernel processes responsible for making space in virtual memory, which may include swapping memory to disk, fragmentation, or promoting pages. Background maintenance of transparent huge pages incurs non-deterministic latency overhead from the kernel as it manages the inevitable fragmentation and swapping issues. EHPs are not subject to memory fragmentation and cannot be swapped to disk, so they incur much less latency overhead.

进程级THP没有上述任何缺点，但它还有另一个缺点。我们之前讨论过，内核对THP的分配对用户是透明的。分配过程可能涉及多个负责在虚拟内存中腾出空间的内核进程，这可能包括将内存交换到磁盘、碎片化或提升页面。透明大页的后台维护会给内核带来不确定的延迟开销，因为它需要处理不可避免的碎片化和交换问题。透明大页（EHP）不受内存碎片的影响，也不能交换到磁盘，因此延迟开销要小得多。

All in all, THPs are easier to use, but incur bigger allocation latency overhead. That is the reason why THPs are not popular in High-Frequency Trading and other ultra-low-latency industries; they prefer to use EHPs instead. On the other hand, virtual machine providers and databases tend to use per-process THPs since requiring additional system configuration can become a burden for their users.

总而言之，THP更易于使用，但分配延迟开销更大。这就是为什么THP在高频交易和其他超低延迟行业并不流行；他们更倾向于使用EHP。另一方面，虚拟机提供商和数据库倾向于使用进程级THP，因为额外的系统配置可能会给用户带来负担。

[^1]: libhugetlbfs - [https://github.com/libhugetlbfs/libhugetlbfs](https://github.com/libhugetlbfs/libhugetlbfs).
[^2]: Linux kernel THP documentation Linux内核THP文档 - [https://www.kernel.org/doc/Documentation/vm/transhuge.txt](https://www.kernel.org/doc/Documentation/vm/transhuge.txt)
