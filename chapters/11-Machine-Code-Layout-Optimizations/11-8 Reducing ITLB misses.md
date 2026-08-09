## Reducing ITLB Misses 减少ITLB失效/未命中 {#sec:FeTLB}

Another important area of tuning Frontend efficiency is the virtual-to-physical address translation of memory addresses. Primarily those translations are served by the TLB (see [@sec:TLBs]), which caches the most recently used memory page translations in dedicated entries. When TLB cannot serve the translation request, a time-consuming page walk of the kernel page table takes place to calculate the correct physical address for each referenced virtual address. Whenever you see a high percentage of ITLB overhead in the TMA summary, the advice in this section may become handy. 

前端效率调优的另一个重要方面是内存地址的虚拟地址到物理地址的转换。这些转换主要由TLB（参见 [@sec:TLBs]）负责，TLB会将最近使用的内存页转换缓存到专用条目中。当TLB无法处理转换请求时，需要对内核页表进行耗时的页面遍历，以计算每个被引用虚拟地址对应的正确物理地址。如果您在TMA摘要中看到ITLB开销占比很高，本节中的建议可能会对您有所帮助。

In general, relatively small applications are not susceptible to ITLB misses. For example, Golden Cove microarchitecture can cover memory space up to 1MB in its ITLB. If the machine code of your application fits in 1MB you should not be affected by ITLB misses. The problem starts to appear when frequently executed parts of an application are scattered around in memory. When many functions begin to frequently call each other, they start competing for the entries in the ITLB. One of the examples is the Clang compiler, which at the time of writing, has a code section of ~60MB. ITLB overhead running on a laptop with a mainstream Intel Coffee Lake processor is ~7%, which means that 7% of cycles are spent handling ITLB misses: doing demanded page walks and populating TLB entries.

一般来说，相对较小的应用程序不易受到ITLB未命中的影响。例如，Golden Cove微体系结构的ITLB可以覆盖高达1MB的内存空间。如果您的应用程序机器代码大小在1MB以内，则不会受到ITLB失效/未命中的影响。当应用程序中频繁执行的部分分散在内存中时，问题就会开始出现。当许多函数开始频繁地相互调用时，它们就会开始争用ITLB中的条目。例如，Clang编译器在撰写本文时，其代码段（code section）大小约为60MB。在搭载主流Intel Coffee Lake处理器的笔记本电脑上运行，ITLB开销约为7%，这意味着7%的CPU周期都用于处理ITLB失效/未命中：执行按需的页面遍历并填充TLB条目。

Another set of large memory applications that frequently benefit from using huge pages include relational databases (e.g., MySQL, PostgreSQL, Oracle), managed runtimes (e.g., JavaScript V8, Java JVM), cloud services (e.g., web search), web tooling (e.g., node.js).

另一类经常受益于使用大页（huge page）的大内存应用程序包括：关系型数据库（例如：MySQL、PostgreSQL、Oracle）、托管运行时（例如：JavaScript V8、Java JVM）、云服务（例如：web网络搜索）和Web工具（例如：Node.js）。

The general idea of reducing ITLB pressure is to map the portions of the performance-critical code of an application onto 2MB (huge) pages. Usually, the entire code section of an application gets remapped for simplicity. The key requirement for that transformation to happen is to have the code section aligned on a 2MB boundary. When on Linux, this can be achieved in two different ways: relinking the binary with an additional linker option or remapping the code sections at runtime. Both options are showcased on the Easyperf[^1] blog. To the best of my knowledge, it is not possible on Windows, so I will only show how to do it on Linux.

降低ITLB压力的通用思路是将应用程序中性能关键代码的部分映射到2MB（大）页上。通常，为了简化操作，应用程序的整个代码段都会被重新映射。实现这种转换的关键要求是代码段必须以2MB为边界对齐。在Linux系统上，可以通过两种不同的方式实现：使用额外的链接器选项重新链接二进制文件，或者在运行时重新映射代码段。Easyperf[^1]博客上展示了这两种方法。据我所知，Windows系统无法实现，因此我将仅展示如何在Linux系统上操作。

The first option can be achieved by linking the binary with the following options: `-Wl,-zcommon-page-size=2097152` `-Wl,-zmax-page-size=2097152`. These options instruct the linker to place the code section at the 2MB boundary in preparation for it to be placed on 2MB pages by the loader at startup. The downside of such placement is that the linker will be forced to insert up to 2MB of padded (wasted) bytes, bloating the binary even more. In the example with the Clang compiler, it increased the size of the binary from 111 MB to 114 MB. After relinking the binary, we set a special bit in the ELF binary header that determines if the text segment should be backed with huge pages by default. The simplest way to do it is using the `hugeedit` or `hugectl` utilities from [libhugetlbfs](https://github.com/libhugetlbfs/libhugetlbfs/blob/master/HOWTO)[^12] package. For example:

第一种方法可以通过使用以下选项链接二进制文件来实现：`-Wl,-zcommon-page-size=2097152` 和 `-Wl,-zmax-page-size=2097152`。这些选项指示链接器将代码段放置在2MB边界处，以便在启动时由加载器将其放置在2MB的页面上。这种放置方式的缺点是，链接器将被迫插入最多2MB的填充（浪费）字节，从而进一步增大二进制文件的大小。在Clang编译器的示例中，二进制文件的大小从111MB增加到114MB。重新链接二进制文件后，我们在ELF二进制文件头中设置一个特殊位，该位决定文本段是否默认使用大页存储。最简单的方法是使用[libhugetlbfs](https://github.com/libhugetlbfs/libhugetlbfs/blob/master/HOWTO)[^12]包中的 `hugeedit` 或 `hugectl` 工具。例如：

```bash
# Permanently set a special bit in the ELF binary header.
$ hugeedit --text /path/to/clang++
# Code section will be loaded using huge pages by default.
$ /path/to/clang++ a.cpp

# Overwrite default behavior at runtime.
$ hugectl --text /path/to/clang++ a.cpp
```

The second option is to remap the code section at runtime. This option does not require the code section to be aligned to a 2MB boundary and thus can work without recompiling the application. This is especially useful when you don’t have access to the source code. The idea behind this method is to allocate huge pages at the startup of the program and transfer the code section there. The reference implementation of that approach is implemented in the [iodlr](https://github.com/intel/iodlr)[^2] library. One option would be to call that functionality from your `main` function. Another option, which is simpler, is to build the dynamic library and preload it in the command line:

第二种方法是在运行时重新映射代码段。这种方法不需要代码段对齐到2MB边界，因此无需重新编译应用程序即可工作。当您无法访问源代码时，这种方法尤其有用。该方法的思路是在程序启动时分配大页内存，并将代码段转移到这些大页内存中。该方法的参考实现位于[iodlr](https://github.com/intel/iodlr)[^2]库中。一种方法是从 `main` 函数调用该功能。另一种更简单的方法是构建动态库并在命令行中预加载它：

```bash
$ LD_PRELOAD=/usr/lib64/liblppreload.so clang++ a.cpp
```

While the first method only works with explicit huge pages, the second approach which uses `iodlr` works both with explicit and transparent huge pages. Instructions on how to enable huge pages for Windows and Linux can be found in Appendix B.

第一种方法仅适用于显式大页，而第二种方法使用 `iodlr`，它既适用于显式大页也适用于透明大页。有关如何在Windows和Linux上启用大页的说明，请参阅附录B。

Mapping code sections onto huge pages can reduce the number of ITLB misses by up to 50% [@IntelBlueprint], which yields speedups of up to 10% for some applications. However, as it is with many other features, huge pages are not for every application. Small programs with an executable file of only a few KB in size would be better off using regular 4KB pages rather than 2MB huge pages; that way, memory is used more efficiently.

将代码段映射到大页可以将ITLB未命中次数减少高达50%[@IntelBlueprint]，这可以为某些应用程序带来高达10%的速度提升。然而，与许多其他特性一样，大页并非适用于所有应用程序。对于可执行文件只有几KB大小的小型程序，使用常规的4KB页面比使用2MB的大页更好；这样可以更有效地利用内存。

Besides employing huge pages, standard techniques for optimizing I-cache performance can be used to improve ITLB performance. Namely, reordering functions so that hot functions are collocated better, reducing the size of hot regions via Link-Time Optimizations (LTO/IPO), using Profile-Guided Optimizations (PGO) and BOLT, and less aggressive inlining.

除了使用大页之外，还可以使用优化I-cache性能的标准技术来提高ITLB性能。具体来说，例如重新排序函数以使热点函数更好地位于同一位置、通过链接时优化(LTO/IPO: Link-Time Optimization)减小热点区域的大小、使用性能轮廓分析文件指导优化(PGO)和BOLT工具，以及降低内联的激进程度。

BOLT provides the `-hugify` option to automatically use huge pages for hot code based on profile data. When this option is used, `llvm-bolt` will inject the code to put hot code on 2MB pages at runtime. The implementation leverages Linux Transparent Huge Pages (THP). The benefit of this approach is that only a small portion of the code is mapped to the huge pages and the number of required huge pages is minimized, and as a consequence, page fragmentation is reduced. 

BOLT提供 `-hugify` 选项，可根据性能分析数据自动将热代码分配到大页。启用此选项后，`llvm-bolt` 会在运行时注入代码，将热代码分配到2MB的页面上。此实现利用了Linux透明大页(THP: Transparent Huge Pages)。这种方法的优势在于，只有一小部分代码会被映射到大页，从而最大限度地减少了所需的大页数量，进而降低了页面碎片。

[^1]: "Performance Benefits of Using Huge Pages for Code" “使用大页处理代码的性能优势” - [https://easyperf.net/blog/2022/09/01/Utilizing-Huge-Pages-For-Code](https://easyperf.net/blog/2022/09/01/Utilizing-Huge-Pages-For-Code).
[^2]: iodlr library, Linux-only iodlr库，仅限Linux - [https://github.com/intel/iodlr](https://github.com/intel/iodlr).
[^12]: libhugetlbfs - [https://github.com/libhugetlbfs/libhugetlbfs/blob/master/HOWTO](https://github.com/libhugetlbfs/libhugetlbfs/blob/master/HOWTO).
