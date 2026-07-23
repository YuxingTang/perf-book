## Memory Hierarchy 存储层次 {#sec:MemHierar}

To effectively utilize all the hardware resources provisioned in a CPU, the machine needs to be fed with the right data at the right time. Failing to do so requires fetching a variable from the main memory, which takes around 100 ns. From a CPU perspective, it is a very long time. Understanding the memory hierarchy is critically important to delivering the performance capabilities of a CPU. Most programs exhibit the property of locality: they don’t access all code or data uniformly. A CPU memory hierarchy is built on two fundamental properties:

为了有效利用CPU中配置的所有硬件资源，机器需要在正确的时间获得正确的数据。否则，就需要从主内存中获取变量值，这大约需要100纳秒。从CPU的角度来看，这非常漫长。理解存储层次结构对于发挥CPU的性能至关重要。大多数程序都具有局部性：它们并非以统一的方式访问所有代码或数据。CPU存储层次结构基于两个基本属性：

* **Temporal locality**: when a given memory location is accessed, the same location will likely be accessed again soon. Ideally, we want this information to be in the cache next time we need it.
* **时间局部性**：当访问某个存储位置时，该位置很可能很快会被再次访问。理想情况下，我们希望下次需要时，该信息已经存在于缓存中。
* **Spatial locality**: when a given memory location is accessed, nearby locations will likely be accessed soon. This refers to placing related data close to each other. When a program reads a single byte from memory, typically, a larger chunk of memory (a cache line) is fetched because very often, the program will require that data soon.
* **空间局部性**：当访问某个存储位置时，附近的存储位置很可能很快会被访问。这指的是将相关数据放置在彼此靠近的位置。当程序从存储中读取单个字节时，通常会先获取一大块存储区（一条缓存行），因为程序很可能很快就会需要这些数据。

This section provides a summary of the key attributes of memory hierarchy systems supported on modern CPUs.

本节概述了现代CPU所支持存储层次结构系统的关键属性。

### Cache Hierarchy 高速缓存层次 {#sec:CacheHierarchy}

A cache is the first level of the memory hierarchy for any request (for code or data) issued from the CPU pipeline. Ideally, the pipeline performs best with an infinite cache with the smallest access latency. In reality, the access time for any cache increases as a function of the size. Therefore, the cache is organized as a hierarchy of small, fast storage blocks closest to the execution units, backed up by larger, slower blocks. A particular level of the cache hierarchy can be used exclusively for code (instruction cache, I-cache) or for data (data cache, D-cache), or shared between code and data (unified cache). Furthermore, some levels of the hierarchy can be private to a particular core, while other levels can be shared among cores.

缓存是CPU流水线发出的任何请求（代码或数据）所遇到存储层次结构的第一层。理想情况下，流水线使用无限大的缓存，访问延迟最小，性能最佳。但实际上，任何缓存的访问时间都会随着其尺寸大小的增加而增加。因此，缓存被组织成一个层次结构，其中靠近执行单元的小型快速存储块由较大的、速度较慢的存储块作为备份。缓存层次结构的特定层可以专门用于代码（指令缓存，I-cache）或数据（数据缓存，D-cache），也可以在代码和数据之间共享（统一缓存）。此外，层次结构的某些层可以是特定核心私有的，而其他层可以在多个核心之间共享。

Caches are organized as blocks with a defined size, also known as *cache lines*. The typical cache line size in modern CPUs is 64 bytes. However, the notable exception here is the L2 cache in Apple processors (such as M1, M2, and later), which operates on 128B cache lines. Caches closest to the execution pipeline typically range in size from 32 KB to 128 KB. Mid-level caches tend to have 1MB and above. Last-level caches in modern CPUs can be tens or even hundreds of megabytes.

缓存以具有固定大小的块，也称为*缓存行（cache line）*的形式组织。现代CPU的典型缓存行大小为64字节。然而，值得注意的是，苹果处理器（例如：M1、M2及后续处理器）中的L2缓存是一个例外，它使用128字节的缓存行。最靠近执行流水线的缓存大小通常在32KB到128KB之间。中级缓存的大小通常为1MB或更大。现代CPU中的末级缓存可以达到数十甚至数百兆字节。

#### Placement of Data within the Cache. 数据在缓存中的放置

The address for a request is used to access the cache. In *direct-mapped* caches, a given block address can appear only in one location in the cache and is defined by a mapping function shown below. Dirrect-mapped caches are relatively easy to build and have fast access time, however, they have a high miss rate.

请求的地址用于访问缓存。在直接映射缓存中，给定的块地址只能出现在缓存中的一个位置，并且由如下所示的映射函数定义。直接映射缓存相对容易构建，访问速度快，但其未命中率/失效率较高。

$$
\textrm{Number of Blocks in the Cache} = \frac{\textrm{Cache Size}}{\textrm{Cache Block Size}}
$$
$$
\textrm{Direct mapped location} = \textrm{(block address)  mod  (Number of Blocks in the Cache )}
$$

$$
\textrm{在缓存中的块数量} = \frac{\textrm{缓存尺寸}}{\textrm{缓存块尺寸}}
$$
$$
\textrm{直接映射位置} = \textrm{(块地址)  mod  (在缓存中块的数量)}
$$

In a *fully associative* cache, a given block can be placed in any location in the cache. This approach involves high hardware complexity and slow access time, thus considered impractical for most use cases.

在*全相联（fully associative）*缓存中，给定的数据块可以放置在缓存中的任意位置。这种方法硬件复杂度高，访问速度慢，因此对于大多数应用场景来说并不实用。

An intermediate option between direct mapping and fully associative mapping is a *set-associative* mapping. In such a cache, the blocks are organized as sets, typically each set containing 2, 4, 8, or 16 blocks. A given address is first mapped to a set. Within a set, the address can be placed anywhere, among the blocks in that set. A cache with m blocks per set is described as an m-way set-associative cache. The formulas for a set-associative cache are:

介于直接映射和全相联映射之间的一种中间方案是*组相联（set-associative）*映射。在这种缓存中，数据块被组织成组，通常每个组包含2、4、8或16个数据块。给定的地址首先被映射到一个组。在同一个组内，该地址可以放置在该组内的任何数据块中。每个组包含m个数据块的缓存被称为m路组相联缓存。组相联缓存的公式如下：

$$
\textrm{Number of Sets in the Cache} = \frac{\textrm{Number of Blocks in the Cache}}{\textrm{Number of Blocks per Set (associativity)}}
$$
$$
\textrm{Set (m-way) associative location} = \textrm{(block address)  mod  (Number of Sets in the Cache)}
$$

$$
\textrm{在缓存中的组数量} = \frac{\textrm{在缓存中的块数量}}{\textrm{每个组中的块数量(相连度)}}
$$
$$
\textrm{(m路)组相连位置} = \textrm{(块地址)  mod  (在缓存中的组数量)}
$$

Consider an example of an L1 cache, whose size is 32 KB with 64 bytes cache lines, 64 sets, and 8 ways. The total number of cache lines in such a cache is `32 KB / 64 bytes = 512 lines`. A new line can only be inserted in its appropriate set (one of the 64 sets). Once the set is determined, a new line can go to one of the 8 ways in this set. Similarly, when you later search for this cache line, you determine the set first, and then you only need to examine up to 8 ways in the set.

考虑一个L1缓存的例子，其大小为32KB，缓存行为64字节，共有64个组，组相联方式为8。这样的缓存中总共有512条缓存行。新行只能插入到其对应的组（64个组中的一个）中。一旦确定了组，新行就可以放入该组中的8条路中的任意一个。同样地，当稍后查找该缓存行时，首先确定组，然后只需检查该组中的最多8个相联路即可。

Here is another example of the cache organization of the Apple M1 processor. The L1 data cache inside each performance core can store 128 KB, has 256 sets with 8 ways in each set, and operates on 64-byte lines. Performance cores form a cluster and share the L2 cache, which can keep 12 MB, is 12-way set-associative, and operates on 128-byte lines. [@AppleOptimizationGuide]

以下是Apple M1处理器缓存组织结构的另一个例子。每个性能核心内部的L1数据缓存可以存储128KB的数据，有256个组，每个组有8个相联路，并且以64字节的缓存行进行操作。性能核心组成一个集群并共享L2缓存，L2缓存可以存储12MB的数据，采用12路组相联方式，并且以128字节的缓存行进行操作。 [@AppleOptimizationGuide]

#### Finding Data in the Cache. 在缓存中查找数据

Every block in the m-way set-associative cache has an address tag associated with it. In addition, the tag also contains state bits such as a bit to indicate whether the data is valid. Tags can also contain additional bits to indicate access information, sharing information, etc.

m路组相联缓存中的每个数据块都关联一个地址标签。此外，标签还包含状态位，例如指示数据是否有效的位。标签还可以包含其他位，用于指示访问信息、共享信息等。

![Address organization for cache lookup. 缓存查找的地址组织。](../../img/uarch/CacheLookup.png){#fig:CacheLookup width=90%}

Figure @fig:CacheLookup shows how the address generated from the pipeline is used to check the caches. The lowest order address bits define the offset within a given block; the block offset bits (5 bits for 32-byte cache lines, 6 bits for 64-byte cache lines). The set is selected using the index bits based on the formulas described above. Once the set is selected, the tag bits are used to compare against all the tags in that set. If one of the tags matches the tag of the incoming request and the valid bit is set, a cache hit results. The data associated with that block entry (read out of the data array of the cache in parallel to the tag lookup) is provided to the execution pipeline. A cache miss occurs in cases where the tag is not a match.

图 @fig:CacheLookup 展示了如何使用流水线生成的地址来检查缓存。最低位地址位定义了给定块内的偏移量；块偏移位（32字节缓存行使用5位，64字节缓存行使用6位）。根据上述公式，使用索引位选择组。选择组后，使用标签位与该组中的所有标签进行比较。如果某个标签与传入请求的标签匹配，并且有效位已设置，则缓存命中。与该块条目关联的数据（在标签查找的同时从缓存的数据数组中读取）将提供给执行流水线。如果标签不匹配，则发生缓存未命中/失效。

#### Managing Misses. 管理失效/未命中

When a cache miss occurs, the cache controller must select a block in the cache to be replaced to allocate the address that incurred the miss. For a direct-mapped cache, since the new address can be allocated only in a single location, the previous entry mapping to that location is deallocated, and the new entry is installed in its place. In a set-associative cache, since the new cache block can be placed in any of the blocks of the set, a replacement algorithm is required. The typical replacement algorithm used is the LRU (least recently used) policy, where the block that was least recently accessed is evicted to make room for the new data. Another alternative is to randomly select one of the blocks as the victim block.

当发生缓存未命中时，缓存控制器必须选择缓存中的一个块来替换导致未命中的地址。对于直接映射缓存，由于新地址只能分配在单个位置，因此会释放映射到该位置的先前条目，并将新条目放入到其对应位置。在组相联缓存中，由于新的缓存块可以放置在组中的任何块中，因此需要一种替换算法。常用的替换算法是最近最少使用（LRU: least Recently Used）策略，其中最近最少访问的块会被驱逐，以便为新数据腾出空间。另一种方法是随机选择一个块作为牺牲块（Victim Block）。

#### Managing Writes. 管理写入

Write accesses to caches are less frequent than data reads. Handling writes in caches is harder, and CPU implementations use various techniques to handle this complexity. Software developers should pay special attention to the various write caching flows supported by the hardware to ensure the best performance of their code.

对缓存的写入访问频率低于数据读取。处理缓存中的写入操作更为复杂，CPU实现采用各种技术来应对这种复杂性。软件开发人员应特别关注硬件支持的各种写入缓存流程，以确保代码的最佳性能。

CPU designs use two basic mechanisms to handle writes that hit in the cache:

CPU设计使用两种基本机制来处理缓存命中的写入操作：

* In a write-through cache, hit data is written to both the block in the cache and to the next lower level of the hierarchy.
* 在*写穿透write-through*缓存中，命中数据会同时写入缓存中的相应块以及其层次结构的下一级。
* In a write-back cache, hit data is only written to the cache. Subsequently, lower levels of the hierarchy contain stale data. The state of the modified line is tracked through a dirty bit in the tag. When a modified cache line is eventually evicted from the cache, a write-back operation forces the data to be written back to the next lower level.
* 在*写回式wirte-back*缓存中，命中数据仅写入缓存。随后，层次结构的下一级包含的是过期数据。已修改行的状态通过标记中的脏位进行跟踪。当已修改的缓存行最终从缓存中移除时，写回操作会将数据强制写回其下一级。

Cache misses on write operations can be handled in two ways:
写入操作的缓存未命中可以通过两种方式处理：

* In a *write-allocate* cache, the data for the missed location is loaded into the cache from the lower level of the hierarchy, and the write operation is subsequently handled like a write hit.
* 在*写分配*缓存中，未命中位置的数据会从缓存层次结构的较低层级加载到缓存中，然后该写入操作会像写入命中一样处理。
* If the cache uses a *no-write-allocate* policy, the cache miss transaction is sent directly to the lower levels of the hierarchy, and the block is not loaded into the cache.
* 如果缓存采用*无写分配*策略，则缓存未命中事务会直接发送到缓存层次结构的较低层级，并且该数据块不会被加载到缓存中。

Out of these options, most designs typically choose to implement a write-back cache with a write-allocate policy as both of these techniques try to convert subsequent write transactions into cache hits, without additional traffic to the lower levels of the hierarchy. Write-through caches typically use the no-write-allocate policy.

在这些选项中，大多数设计通常选择实现采用具备写分配策略的回写式缓存，因为这两种技术都试图将后续的写入事务转换为缓存命中，而无需向较低层级发送额外的流量。写直达缓存通常采用无写分配策略。

#### Other Cache Optimization Techniques. 其他缓存优化技术

For a programmer, understanding the behavior of the cache hierarchy is critical to extracting performance from any application. From the perspective of the CPU pipeline, the latency to access any request is given by the following formula that can be applied recursively to all the levels of the cache hierarchy up to the main memory:

对于程序员而言，理解缓存层次结构的行为对于提升任何应用程序的性能都至关重要。从CPU流水线的角度来看，访问任何请求的延迟可以用以下公式表示，该公式可以递归地应用于缓存层次结构的所有级别，直至主内存：

$$
\textrm{Average Access Latency} = \textrm{Hit Time } + \textrm{ Miss Rate } \times \textrm{ Miss Penalty}
$$
$$
\textrm{平均访问延迟} = \textrm{命中访问时间 } + \textrm{ 失效/未命中率 } \times \textrm{ 失效/未命中惩罚 }
$$
Hardware designers take on the challenge of reducing the hit time and miss penalty through many novel micro-architecture techniques. Fundamentally, cache misses stall the pipeline and hurt performance. The miss rate for any cache is highly dependent on the cache architecture (block size, associativity) and the software running on the machine.

硬件设计人员致力于通过许多新颖的微体系结构构技术来降低命中时间和未命中惩罚。从根本上讲，缓存未命中会阻塞流水线并降低性能。任何缓存的未命中率都高度依赖于缓存体系结构（块大小、关联度）以及机器上运行的软件。

#### Hardware and Software Prefetching. 硬件和软件预取。 {#sec:HwPrefetch}

One method to avoid cache misses and subsequent stalls is to prefetch data into caches prior to when the pipeline demands it. The assumption is the time to handle the miss penalty can be mostly hidden if the prefetch request is issued sufficiently ahead in the pipeline. Most CPUs provide implicit hardware-based prefetching that is complemented by explicit software prefetching that programmers can control.

避免缓存未命中和随之而来的停顿的一种方法是：在流水线需要数据之前将其预取到缓存中。其假设是，如果预取请求在流水线中足够提前发出，则处理缓存未命中惩罚的时间可以大部分被忽略。大多数CPU提供隐式的硬件预取，并辅以程序员可以控制的显式软件预取。

Hardware prefetchers observe the behavior of a running application and initiate prefetching on repetitive patterns of cache misses. Hardware prefetching can automatically adapt to the dynamic behavior of an application, such as varying data sets, and does not require support from an optimizing compiler. Also, the hardware prefetching works without the overhead of additional address generation and prefetch instructions. However, hardware prefetching works for a limited set of commonly used data access patterns.

硬件预取器会观察正在运行的应用程序的行为，并在缓存未命中出现重复模式时启动预取。硬件预取可以自动适应应用程序的动态行为，例如不同的数据集，并且不需要优化编译器的支持。此外，硬件预取无需额外的地址生成和预取指令开销。然而，硬件预取仅适用于一组有限的常用数据访问模式。

Software memory prefetching complements prefetching done by hardware. Developers can specify which memory locations are needed ahead of time via dedicated hardware instruction (see [@sec:memPrefetch]). Compilers can also automatically add prefetch instructions into the code to request data before it is required. Prefetch techniques need to balance between demand and prefetch requests to guard against prefetch traffic slowing down demand traffic.

软件内存预取是对硬件预取的补充。开发人员可以通过专用的硬件指令预先指定所需的内存位置（参见 [@sec:memPrefetch]）。编译器还可以自动在代码中添加预取指令，以便在需要数据之前请求数据。预取技术需要在需求流量和预取请求之间取得平衡，以防止预取流量拖慢需求流量。

### Main Memory 主内存 {#sec:UarchMainmemory}

Main memory is the next level of the hierarchy, downstream from the caches. Requests to load and store data are initiated by the Memory Controller Unit (MCU). In the past, this circuit was located in the northbridge chip on the motherboard. But nowadays, most processors have this component embedded, so the CPU has a dedicated memory bus connecting it to the main memory.

主内存是缓存层级结构中的下一级，位于缓存的下游。加载lost和存储Store数据的请求由内存控制器单元 (MCU: Memory Controller Unit) 发起。过去，该电路位于主板上的北桥芯片中。但如今，大多数处理器都集成了该组件，因此CPU拥有专门的内存总线将其连接到主内存。

Main memory uses DRAM (Dynamic Random Access Memory) technology that supports large capacities at reasonable cost points. When comparing DRAM modules, people usually look at memory density and memory speed, along with its price of course. Memory density defines the capacity of the module measured in GB. Obviously, the more available memory the better as it is a precious resource used by the OS and applications.

主内存采用动态随机存取存储器（DRAM: Dynamic Random Access Memory）技术，以合理的成本支持大容量。在比较DRAM模块时，人们通常会考虑内存密度、内存速度以及价格。内存密度定义了模块的容量，以GB为单位。显然，可用内存越多越好，因为它是操作系统和应用程序使用的宝贵资源。

The performance of the main memory is described by latency and bandwidth. Memory latency is the time elapsed between the memory access request being issued and when the data is available to use by the CPU. Memory bandwidth defines how many bytes can be fetched per some period of time, and is usually measured in gigabytes per second.

主内存的性能由延迟和带宽来描述。内存延迟是指从发出内存访问请求到CPU可以使用数据之间的时间间隔。内存带宽定义了在一定时间内可以读取多少字节的数据，通常以千兆字节/秒(GB/s)为单位。

#### DDR

(Double Data Rate) is the predominant DRAM technology supported by most CPUs. Historically, DRAM bandwidths have improved every generation while the DRAM latencies have stayed the same or increased. Table @tbl:mem_rate shows the top data rate, peak bandwidth, and the corresponding reading latency for the last three generations of DDR technologies. The data rate is measured in millions of transfers per second (MT/s). The latencies shown in this table correspond to the latency in the DRAM device itself. Typically, the latencies as seen from the CPU pipeline (cache miss on a load to use) are higher (in the 50ns-150ns range) due to additional latencies and queuing delays incurred in the cache controllers, memory controllers, and on-die interconnects. You can see an example of measuring observed memory latency and bandwidth in [@sec:MemLatBw].

双倍数据速率（DDR: Double Data Rate）是大多数CPU支持的主要DRAM技术。从历史上看，DRAM带宽每一代都在提高，而DRAM延迟则保持不变或有所增加。表 @tbl:mem_rate 显示了过去三代DDR技术的最高数据速率、峰值带宽和相应的读取延迟。数据速率以每秒百万次传输 (MT/s) 为单位。此表中显示的延迟对应于 DRAM器件本身的延迟。通常情况下，由于缓存控制器、内存控制器和片上互连中产生的额外延迟和排队延迟，从CPU流水线（加载时缓存未命中）观察到的延迟会更高（在50ns到150ns范围内）。你可以在 [@sec:MemLatBw] 中查看测量观察到的内存延迟和带宽的示例。

-----------------------------------------------------------------
   DDR       Year   Highest Data   Peak Bandwidth  In-device Read
Generation           Rate(MT/s)       (GB/s)         Latency(ns)
----------  ------  ------------   --------------  --------------
  DDR3       2007      2133            17.1            10.3

  DDR4       2014      3200            25.6            12.5

  DDR5       2020      6400            51.2            14

-----------------------------------------------------------------

Table: Performance characteristics for the last three generations of DDR technologies. 过去3代DDR技术的性能特征。 {#tbl:mem_rate}

It is worth mentioning that DRAM chips require their memory cells to be refreshed periodically. This is because the bit value is stored as the presence of an electric charge on a tiny capacitor, so it can lose its charge over time. To prevent this, there is special circuitry that reads each cell and writes it back, effectively restoring the capacitor's charge. While a DRAM chip is in its refresh procedure, it is not serving memory access requests.

值得一提的是，DRAM芯片需要定期刷新其存储单元。这是因为数据位值是以微型电容器上的电荷形式存储的，因此电容器会随着时间的推移而失去电荷。为了防止这种情况发生，芯片内部有特殊的电路来读取每个存储单元并将其写回，从而有效地恢复电容器的电荷。DRAM芯片在刷新过程中不会处理内存访问请求。

A DRAM module is organized as a set of DRAM chips. Memory *rank* is a term that describes how many sets of DRAM chips exist on a module. For example, a single-rank (1R) memory module contains one set of DRAM chips. A dual-rank (2R) memory module has two sets of DRAM chips, therefore doubling the capacity of a single-rank module. Likewise, there are quad-rank (4R) and octa-rank (8R) memory modules available for purchase.

DRAM模块由一组DRAM芯片组成。内存“秩rank”是指模块上DRAM芯片组的数量。例如，单秩（1R）内存模块包含一组DRAM芯片。双秩（2R）内存模块包含2组DRAM芯片，因此容量是单秩模块的2倍。同样，市面上也有2秩（4R）和8秩（8R）内存模块可供选择。

Each rank consists of multiple DRAM chips. Memory *width* defines how wide the bus of each DRAM chip is. And since each rank is 64 bits wide (or 72 bits wide for ECC RAM), it also defines the number of DRAM chips present within the rank. Memory width can be one of three values: `x4`, `x8`, or `x16`, and defines how wide is the bus that goes to each chip. As an example, Figure @fig:Dram_ranks shows the organization of a 2Rx16 dual-rank DRAM DDR4 module, with a total of 2GB capacity. There are four chips in each rank, with a 16-bit wide bus. Combined, the four chips provide 64-bit output. The two ranks are selected one at a time through a rank-select signal.

每个Rank由多个DRAM芯片组成。内存宽度定义了每个DRAM芯片的总线宽度。由于每个Rank的宽度为64位（纠错码ECC保护的RAM为72位），因此内存宽度也决定了该Rank内DRAM芯片的数量。内存宽度可以是三个值之一：`x4`、`x8` 或 `x16`，它定义了连接到每个芯片的总线宽度。例如，图 @fig:Dram_ranks 展示了一个2Rx16双Rank DDR4 DRAM模块的结构，总容量为2GB。每个Rank中有四个芯片，总线宽度为16位。这4个芯片组合起来提供64位输出。两个Rank通过Rank选择信号依次选择。

![Organization of a 2Rx16 dual-rank DRAM DDR4 module with a total capacity of 2GB.2Rx16的双秩DRAM模块的结构组织，总容量为2GB。](../../img/uarch/DRAM_ranks.png){#fig:Dram_ranks width=90%}

There is no direct answer as to whether the performance of single-rank or dual-rank is better as it depends on the type of application. Single-rank modules generally produce less heat and are less likely to fail. Also, multi-rank modules require a rank select signal to switch from one rank to another, which needs additional clock cycles and may increase the access latency. On the other hand, if a rank is not accessed, it can go through its refresh cycles in parallel while other ranks are busy. As soon as the previous rank completes data transmission, the next rank can immediately start its transmission.

单Rank和双Rank哪种性能更好并没有标准答案，这取决于应用类型。单Rank模块通常发热量更低，故障率也更低。此外，双Rank模块需要Rank选择信号才能在不同Rank之间切换，这需要额外的时钟周期，可能会增加访问延迟。另一方面，如果某个Rank未被访问，它可以并行执行刷新周期，而其他Rank则处于忙碌状态。一旦前一个Rank完成数据传输，下一个Rank就可以立即开始传输。

Going further, we can install multiple DRAM modules in a system to not only increase memory capacity but also memory bandwidth. Setups with multiple memory channels are used to scale up the communication speed between the memory controller and the DRAM.

此外，我们可以在系统中安装多个DRAM模块，不仅可以增加内存容量，还可以增加内存带宽。多通道内存体系结构用于提升内存控制器和DRAM之间的通信速度。

A system with a single memory channel has a 64-bit wide data bus between the DRAM and memory controller. The multi-channel architectures increase the width of the memory bus, allowing DRAM modules to be accessed simultaneously. For example, the dual-channel architecture expands the width of the memory data bus from 64 bits to 128 bits, doubling the available bandwidth, see Figure @fig:Dram_channels. Notice, that each memory module, is still a 64-bit device, but we connect them differently. It is very typical nowadays for server machines to have four or eight memory channels.

单通道内存系统在DRAM和内存控制器之间使用64位宽的数据总线。多通道体系结构增加了内存总线的宽度，从而允许同时访问多个DRAM模块。例如，双通道体系结构将内存数据总线的宽度从64位扩展到128位，使可用带宽翻倍，参见图@fig:Dram_channels。需要注意的是，每个内存模块仍然是64位器件，但它们的连接方式不同。如今，服务器通常配备4个或8个内存通道。

![Organization of a dual-channel DRAM setup. 双通道DRAM结构组织示意图。](../../img/uarch/DRAM_channels.png){#fig:Dram_channels width=60%}

Alternatively, you could also encounter setups with duplicated memory controllers. For example, a processor may have two integrated memory controllers, each of them capable of supporting several memory channels. The two controllers are independent and only view their own slice of the total physical memory address space.

此外，您也可能遇到使用多个内存控制器的体系结构。例如，一个处理器可能集成了2个内存控制器，每个控制器都能支持多个内存通道。这2个控制器相互独立，各自只能访问物理内存地址空间的一部分。

We can do a quick calculation to determine the maximum memory bandwidth for a given memory technology, using the simple formula below:
我们可以使用以下简单公式快速计算给定内存技术的最大内存带宽：
$$
\textrm{Max. Memory Bandwidth} = \textrm{Data Rate } \times \textrm{ Bytes per cycle }
$$
$$
\textrm{最大内存带宽} = \textrm{数据速率} \times \textrm{每周期字节数}
$$

For example, for a single-channel DDR4 configuration with a data rate of 2400 MT/s and 64 bits (8 bytes) per transfer, the maximum bandwidth equals `2400 * 8 = 19.2 GB/s`. Dual-channel or dual memory controller setups double the bandwidth to 38.4 GB/s. Remember though, those numbers are theoretical maximums that assume that a data transfer will occur at each memory clock cycle, which in fact never happens in practice. So, when measuring actual memory speed, you will always see a value lower than the maximum theoretical transfer bandwidth.

例如，对于数据速率为2400MT/s、每次传输64位（8字节）的单通道DDR4配置，最大带宽为 `2400 * 8 = 19.2GB/s`。双通道或双内存控制器配置可将带宽翻倍至38.4GB/s。但请记住，这些数字是理论最大值，假设每个内存时钟周期都会发生一次数据传输，而这在实际应用中几乎不会发生。因此，在测量实际内存速度时，您总是会看到低于最大理论传输带宽的值。

To enable multi-channel configuration, you need to have a CPU and motherboard that support such an architecture and install an even number of identical memory modules in the correct memory slots on the motherboard. The quickest way to check the setup on Windows is by running a hardware identification utility like `CPU-Z` or `HwInfo`; on Linux, you can use the `dmidecode` command. Alternatively, you can run memory bandwidth benchmarks like Intel MLC or Stream.

要启用多通道配置，您需要使用支持这种体系结构的CPU和主板，并在主板上正确的内存插槽中安装偶数个相同的内存模块。在Windows系统上，检查配置的最快方法是运行硬件识别工具，例如 `CPU-Z` 或 `HwInfo`；在Linux系统上，您可以使用 `dmidecode` 命令。或者，您可以运行Intel MLC或Stream等内存带宽基准测试程序。

To make use of multiple memory channels in a system, there is a technique called *interleaving*. It spreads adjacent addresses within a page across multiple memory devices. An example of a 2-way interleaving for sequential memory accesses is shown in Figure @fig:Dram_channel_interleaving. As before, we have a dual-channel memory configuration (channels A and B) with two independent memory controllers. Modern processors interleave per four cache lines (256 bytes), i.e., the first four adjacent cache lines go to channel A, and then the next set of four cache lines go to channel B.

为了在系统中利用多个内存通道，有一种称为*交错interleaving*的技术。它将页面内相邻的地址分布到多个内存器件上。图 @fig:Dram_channel_interleaving 展示了一个用于顺序内存访问的双路交错示例。与之前一样，我们有一个双通道内存配置（通道A和B），其中包含两个独立的内存控制器。现代处理器以四行缓存（256字节）为单位进行交错，也就是说，前四行相邻的缓存被发送到通道A，然后接下来的四行缓存被发送到通道B。

![2-way interleaving for sequential memory access. 用于顺序内存访问的双路交错。](../../img/uarch/DRAM_channel_interleaving.png){#fig:Dram_channel_interleaving width=80%}

Without interleaving, consecutive adjacent accesses would be sent to the same memory controller, not utilizing the second available controller. In contrast, interleaving enables hardware parallelism to better utilize available memory bandwidth. For most workloads, performance is maximized when all the channels are populated as it spreads a single memory region across as many DRAM modules as possible.

如果没有交错，连续的相邻访问将被发送到同一个内存控制器，而无法利用第二个可用的控制器。相比之下，交错能够实现硬件并行，从而更好地利用可用的内存带宽。对于大多数工作负载而言，当所有通道都被占用时，性能达到最大化，因为这样可以将单个内存区域扩展到尽可能多的DRAM模块上。

While increased memory bandwidth is generally good, it does not always translate into better system performance and is highly dependent on the application. On the other hand, it's important to watch out for available and utilized memory bandwidth, because once it becomes the primary bottleneck, the application stops scaling, i.e., adding more cores doesn't make it run faster.

虽然提高内存带宽通常是好事，但这并不总是能转化为更好的系统性能，而且很大程度上取决于应用程序。另一方面，密切关注可用和已用内存带宽至关重要，因为一旦它成为主要瓶颈，应用程序的扩展性就会受到限制，也就是说，增加更多核心并不能提高运行速度。

#### GDDR and HBM 和高带宽内存

Besides multi-channel DDR, there are other technologies that target workloads where higher memory bandwidth is required to achieve greater performance. Technologies such as GDDR (Graphics DDR) and HBM (High Bandwidth Memory) are the most notable ones. They find their use in high-end graphics, high-performance computing such as climate modeling, molecular dynamics, and physics simulation, but also autonomous driving, and of course, AI/ML. They are a natural fit there because such applications require moving large amounts of data very quickly.

除了多通道DDR之外，还有其他一些技术专门针对需要更高内存带宽以实现更高性能的工作负载。图形显存（图形显存：Grapics DDR）和高带宽内存（HBM: High Bandwidth Memory）等技术最为著名。它们广泛应用于高端图形、高性能计算（例如：气候建模、分子动力学和物理模拟）、自动驾驶以及人工智能/机器学习等领域。这些技术非常适合这些应用，因为此类应用需要快速传输大量数据。

GDDR was primarily designed for graphics and nowadays it is used on virtually every high-performance graphics card. While GDDR shares some characteristics with DDR, it is also quite different. While DRAM DDR is designed for lower latencies, GDDR is built for much higher bandwidth, because it is located in the same package as the processor chip itself. Similar to DDR, the GDDR interface transfers two 32-bit words (64 bits in total) per clock cycle. The latest GDDR6X standard can achieve up to 168 GB/s bandwidth, operating at a relatively low 656 MHz frequency.

GDDR最初是为图形处理而设计的，如今几乎所有高性能显卡都使用GDDR。虽然GDDR与DDR有一些共同的特性，但也存在显著差异。DRAM DDR的设计目标是降低延迟，而GDDR则是为了更高的带宽而打造的，因为它会与处理器芯片放在同一封装内。与DDR类似，GDDR接口每个时钟周期传输两个32位字（总共64位）。最新的GDDR6X 标准在相对较低的656MHz频率下，带宽可达168GB/s。

HBM is a new type of CPU/GPU memory that vertically stacks memory chips, also called 3D stacking. Similar to GDDR, HBM drastically shortens the distance data needs to travel to reach a processor. The main difference from DDR and GDDR is that the HBM memory bus is very wide: 1024 bits for each HBM stack. This enables HBM to achieve ultra-high bandwidth. The latest HBM3 standard supports up to 665 GB/s bandwidth per package. It also operates at a low frequency of 500 MHz and has a memory density of up to 48 GB per package.

HBM是一种新型的CPU/GPU内存，它采用垂直堆叠的方式组织内存芯片，也称为3D堆叠。与GDDR类似，HBM也大幅缩短了数据传输到处理器的距离。HBM与DDR的主要区别在于其内存总线非常宽：每个HBM堆叠拥有1024位。这使得HBM能够实现超高的带宽。最新的HBM3标准支持单封装高达665GB/s的带宽。它还以500MHz的低频率运行，单封装容量高达48GB。

A system with HBM onboard will be a good choice if you want to maximize data transfer throughput. However, at the time of writing, this technology is quite expensive. As GDDR is predominantly used in graphics cards, HBM may be a good option to accelerate certain workloads that run on a CPU. In fact, the first x86 general-purpose server chips with integrated HBM are now available.

如果您希望最大限度地提高数据传输吞吐率，那么板载HBM的系统将是一个不错的选择。然而，截至撰写本文时，这项技术价格相当昂贵。由于GDDR主要用于显卡，因此HBM可能是加速某些CPU工作负载的理想选择。事实上，首批集成HBM的x86通用服务器芯片现已上市。