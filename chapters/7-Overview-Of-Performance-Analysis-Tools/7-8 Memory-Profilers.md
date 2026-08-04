## Memory Profiling 内存轮廓分析 {#sec:MemoryProfiling}

So far in this chapter, we have discussed tools that identify places where a program spends most of its time. In this section, we will focus on a program's interaction with memory. This is usually called *memory profiling*. In particular, we will learn how to collect memory usage, profile heap allocations and measure memory footprint. Memory profiling helps you understand how an application uses memory over time and helps you build an accurate mental model of a program's interaction with memory. Here are some questions it can answer:

本章到目前为止，我们讨论了用于识别程序运行时间最长的内存位置的工具。在本节中，我们将重点关注程序与内存的交互。这通常被称为*内存轮廓分析（memory profiling）*。具体来说，我们将学习如何收集内存使用情况、分析堆分配以及测量内存占用量。内存分析可以帮助您了解应用程序如何随时间使用内存，并帮助您构建程序与内存交互的精确模型。它可以回答以下一些问题：

* What is a program's total virtual memory consumption and how does it change over time?
* Where and when does a program make heap allocations?
* What are the code places with the largest amount of allocated memory?
* How much memory does a program access every second?
* What is the total memory footprint of a program?

* 程序的总虚拟内存消耗是多少？它如何随时间变化？
* 程序在何时何地进行堆分配？
* 哪些代码段分配的内存最多？
* 程序每秒访问多少内存？
* 程序的总内存占用量是多少？

### Memory Usage 内存使用情况

Memory usage is frequently described by Virtual Memory Size (VSZ) and Resident Set Size (RSS). VSZ includes all memory that a process can access, e.g., stack, heap, the memory used to encode instructions of an executable, and instructions from linked shared libraries, including the memory that is swapped out to disk/SSD. On the other hand, RSS measures how much memory allocated to a process resides in RAM. Thus, RSS does not include memory that is swapped out or was never touched yet by that process. Also, RSS does not include memory from shared libraries that were not loaded to memory. Files that are mapped to memory with `mmap` also contribute to VSZ and RSS usage.

内存使用情况通常用虚拟内存大小(VSZ: Virtual Memory Size)和驻留内存集大小(RSS: Resident Set Size)来描述。VSZ包括进程可以访问的所有内存，例如：栈、堆、用于编码可执行文件指令的内存以及链接的共享库中的指令，包括已交换到磁盘/SSD的内存。另一方面，RSS衡量分配给进程的内存中实际驻留在RAM中的内存量。因此，RSS不包括已交换到磁盘或进程从未访问过的存储。此外，RSS也不包括未加载到内存中的共享库存储。使用 `mmap` 映射到内存的文件也会影响VSZ和RSS的使用量。

Consider an example. Process `A` has 200K of stack and heap allocations of which 100K resides in the main memory; the rest is swapped out or unused. It has a 500K binary, from which only 400K was touched. Process `A` is linked against 2500K of shared libraries and has only loaded 1000K in the main memory.

例如，进程 `A` 分配了200K的栈和堆内存，其中100K驻留在主内存中；其余部分已交换到磁盘或未使用。它有一个500K的二进制文件，但只访问了其中的400K。进程 `A` 链接了2500K的共享库，但只有1000K加载到主内存中。

```
VSZ: 200K + 500K + 2500K = 3200K
RSS: 100K + 400K + 1000K = 1500K
```

Developers can observe both RSS and VSZ on Linux with a standard `top` utility, however, both metrics can change very rapidly. Luckily, some tools can record and visualize memory usage over time. Figure @fig:MemoryUsageAIBench shows the memory usage of the PSPNet image segmentation algorithm, which is a part of the [AI Benchmark Alpha](https://ai-benchmark.com/alpha.html).[^5] This chart was created based on the output of a tool called [memory_profiler](https://github.com/pythonprofilers/memory_profiler)[^6], a Python library built on top of the cross-platform [psutil](https://github.com/giampaolo/psutil)[^7] package.

开发者可以使用标准的 `top` 命令在Linux系统上观察RSS和VSZ值，然而，这两个指标的变化速度都非常快。幸运的是，一些工具可以记录并可视化内存使用情况随时间的变化。图 @fig:MemoryUsageAIBench 展示了PSPNet图像分割算法的内存使用情况，该算法是[AI Benchmark Alpha](https://ai-benchmark.com/alpha.html)[^5]的一部分。该图表基于名为[memory_profiler](https://github.com/pythonprofilers/memory_profiler)[^6]的工具的输出结果生成，[memory_profiler](https://github.com/pythonprofilers/memory_profiler)[^6]是一个基于跨平台[psutil](https://github.com/giampaolo/psutil)[^7]包构建的Python库。

![RSS and VSZ memory utilization of AI_bench PSPNet image segmentation. AI_bench PSPNet 图像分割的 RSS 和 VSZ 内存利用率。](../../img/memory-access-opts/MemoryUsageAIBench.png){#fig:MemoryUsageAIBench width=100%}

In addition to standard RSS and VSZ metrics, people have developed a few more sophisticated metrics. Since RSS includes both the memory that is unique to the process and the memory shared with other processes, it's not clear how much memory a process has for itself. The USS (Unique Set Size) is the memory that is unique to a process and which would be freed if the process was terminated right now. The PSS (Proportional Set Size) represents unique memory plus the amount of shared memory, evenly divided between the processes that share it. E.g. if a process has 10 MB all to itself (USS) and 10 MBs shared with another process, its PSS will be 15 MBs. The `psutil` library supports measuring these metrics (Linux-only), which can be visualized by `memory_profiler`. 

除了标准的RSS和VSZ指标外，人们还开发了一些更复杂的指标。由于RSS既包含进程独有的内存，也包含与其他进程共享的内存，因此很难明确一个进程究竟拥有多少内存。唯一内存集大小（USS: Unique Set Size）是指进程独有的内存，如果进程立即终止，这部分内存将被释放。比例内存集大小（PSS: Proportional Set Size）表示独有内存加上与其他进程共享的内存，后者平均分配给所有共享内存的进程。例如，如果一个进程拥有10MB的独有内存（USS）和10MB的与另外一个进程共享的内存，那么它的PSS就是15MB。 `psutil` 库支持测量这些指标（仅限Linux），这些指标可以通过 `memory_profiler` 进行可视化。

On Windows, similar concepts are defined by Committed Memory Size and Working Set Size. They are not direct equivalents to VSZ and RSS but can be used to effectively estimate the memory usage of Windows applications. The [RAMMap](https://learn.microsoft.com/en-us/sysinternals/downloads/rammap)[^8] tool provides a rich set of information about memory usage for the system and individual processes.

在Windows系统中，类似的概念由已提交内存大小(Committed Memory Size)和工作集大小(Working Set Size)定义。它们并非VSZ和RSS的直接等价物，但可用于有效地估算Windows应用程序的内存使用情况。微软的[RAMMap](https://learn.microsoft.com/en-us/sysinternals/downloads/rammap)[^8]工具提供了丰富的系统和各个进程的内存使用信息。

When developers talk about memory consumption, they implicitly mean heap usage. Heap is, in fact, the biggest memory consumer in most applications as it accommodates all dynamically allocated objects. But heap is not the only memory consumer. For completeness, let's mention others:

开发人员在谈论内存消耗时，通常指的是堆内存的使用。事实上，堆内存是大多数应用程序中最大的内存消耗者，因为它容纳了所有动态分配的对象。但堆内存并非唯一的内存消耗者。为了完整起见，我们再提及其他内存消耗者：

* Stack: Memory used by stack frames in an application. Each thread inside an application gets its own stack memory space. Usually, the stack size is only a few MB, and the application will crash if it exceeds the limit. For example, the default size of stack memory on Linux is usually 8MB, although it may vary depending on the distribution and kernel settings. The default stack size on macOS is also 8MB, but on Windows, it's only 1 MB. The total stack memory consumption is proportional to the number of threads running in the system.
* Code: Memory that is used to store the code (instructions) of an application and its libraries. In most cases, it doesn't contribute much to memory consumption, but there are exceptions. For example, the Clang 17 C++ compiler has a 33 MB code section, while the latest Windows Chrome browser has 187MB of its 219MB `chrome.dll` dedicated to code. However, not all parts of the code are frequently exercised while a program is running. We show how to measure code footprint in [@sec:CodeFootprint].

* 栈：应用程序中栈帧使用的内存。应用程序中的每个线程都有自己的栈内存空间。通常，栈的大小只有几MB，如果超过限制，应用程序将会崩溃。例如，Linux的默认栈内存大小通常为8MB，但具体大小可能因发行版和内核设置而异。macOS的默认栈内存大小也是8MB，而Windows则只有1MB。栈内存的总消耗量与系统中运行的线程数成正比。
* 代码：用于存储应用程序及其库的代码（指令）的内存。在大多数情况下，代码对内存消耗的贡献不大，但也存在例外。例如，Clang 17 C++编译器有一个33MB的代码段，而最新的Windows版Chrome浏览器在其219MB的 `chrome.dll` 文件中，有187MB专门用于存储代码。然而，程序运行时并非所有代码都会频繁执行。我们在 [@sec:CodeFootprint] 中展示了如何测量代码占用空间。

Since the heap is usually the largest consumer of memory resources, it makes sense for developers to focus on this part of memory when they analyze the memory utilization of their applications. In the following section, we will examine heap consumption and memory allocations in a popular real-world application.

由于堆通常是内存资源的最大消耗者，因此开发人员在分析应用程序的内存利用率时，重点关注这部分内存是很有意义的。在下一节中，我们将研究一个流行的实际应用程序中的堆消耗和内存分配。

### Case Study: Analyzing Stockfish's Heap Allocations 案例研究：分析Stockfish的堆内存分配 {#sec:HeaptrackCaseStudy}

In this case study, I use [heaptrack](https://github.com/KDE/heaptrack)[^2], an open-sourced heap memory profiler for Linux developed by KDE. Ubuntu users can install it very easily with `apt install heaptrack heaptrack-gui`. Heaptrack can find places in the code where the largest and most frequent allocations happen among many other things. On Windows, you can use [Mtuner](https://github.com/milostosic/MTuner)[^3] which has similar capabilities to Heaptrack.

在本案例研究中，我使用了堆跟踪[heaptrack](https://github.com/KDE/heaptrack)[^2]，这是一个由KDE开发的开源Linux堆内存分析器。Ubuntu用户可以使用 `apt install heaptrack heaptrack-gui` 轻松安装它。Heaptrack可以找到代码中内存分配最频繁和最大的地方，以及进行其他许多操作。在Windows系统上，您可以使用[Mtuner](https://github.com/milostosic/MTuner)[^3]，它具有与Heaptrack类似的功能。

I analyzed [Stockfish's](https://github.com/official-stockfish/Stockfish)[^4] chess engine built-in benchmark, which we have already looked at in [@sec:PerfMetricsCaseStudy]. As before, I compiled it using the Clang 15 compiler with `-O3 -mavx2` options. I collected the Heaptrack memory profile of a single-threaded Stockfish built-in benchmark on an Intel Alder Lake i7-1260P processor using the following command:

我分析了[Stockfish](https://github.com/official-stockfish/Stockfish)[^4]内置的国际象棋引擎基准测试，我们在 [@sec:PerfMetricsCaseStudy] 中已经讨论过它。与之前一样，我使用Clang 15编译器，并启用 `-O3 -mavx2` 选项进行编译。我使用以下命令在Intel Alder Lake i7-1260P处理器上收集了单线程Stockfish内置基准测试的Heaptrack内存分析数据：

```bash
$ heaptrack ./stockfish bench 128 1 24 default depth
```

Figure @fig:StockfishSummary shows us a summary view of the Stockfish memory profile. Here are some interesting facts we can learn from it:

图 @fig:StockfishSummary 显示了Stockfish内存轮廓分析的概要视图。以下是一些值得注意的信息：

- The total number of allocations is 10614.
- Almost half of the allocations are temporary, i.e., allocations that are directly followed by their deallocation.
- Peak heap memory consumption is 204 MB.
- `Stockfish::std_aligned_alloc` is responsible for the largest portion of the allocated heap space (182 MB). But it is not among the most frequent allocation spots (middle table), so it is likely allocated once and stays alive until the end of the program.
- Almost half of all the allocation calls come from `operator new`, which are all temporary allocations. Can we get rid of temporary allocations? You will find out later.
- Leaked memory is not a concern for this case study.

- 总分配次数为10614次。
- 近一半的分配是临时分配，即分配后立即被释放。
- 堆内存峰值消耗为204MB。
- `Stockfish::std_aligned_alloc` 占用了大部分堆内存（182MB）。但它并非最常用的内存分配位置（中间表），因此很可能只分配一次，并一直保持活动状态直到程序结束。
- 几乎一半的内存分配调用都来自 `operator new`，而这些调用都是临时分配。我们能否消除临时分配？稍后您将找到答案。
- 内存泄漏在本案例研究中不是问题。

![Stockfish memory profile with Heaptrack, summary view. Stockfish使用Heaptrack的内存轮廓分析，摘要视图。](../../img/memory-access-opts/StockfishSummary.png){#fig:StockfishSummary width=100%}

Notice, that there are many tabs on the top of the image; we will explore some of them. Figure @fig:StockfishMemUsage shows the memory usage of the Stockfish built-in benchmark. The memory usage stays constant at 200 MB throughout the entire run of the program. Total consumed memory is broken into slices, e.g., regions \circled{1} and \circled{2} on the image. Each slice corresponds to a particular allocation. Interestingly, it was not a single big 182 MB allocation that was done through `Stockfish::std_aligned_alloc` as we thought earlier. Instead, there are two: slice \circled{1} 134.2 MB and slice \circled{2} 48.4 MB. Both allocations stay alive until the very end of the benchmark. 

请注意，图像顶部有很多选项卡；我们将探索其中的一些。图 @fig:StockfishMemUsage 显示了Stockfish内置基准测试的内存使用情况。在整个程序运行过程中，内存使用量始终保持在200MB。总内存使用量被分割成多个区域，例如图中的 \circled{1} 和 \circled{2}。每个区域对应一次特定的内存分配。有趣的是，它并非像我们之前认为的那样，通过 `Stockfish::std_aligned_alloc` 一次性分配了182MB的内存。实际上，它分配了两笔内存：区域 \circled{1} 为134.2MB，区域 \circled{2} 为48.4MB。这两笔内存分配一直持续到基准测试结束。

![Stockfish memory profile with Heaptrack, memory usage over time stays constant. Stockfish使用Heaptrack进行内存分析，内存使用量随时间保持稳定。](../../img/memory-access-opts/Stockfish_consumed.png){#fig:StockfishMemUsage width=90%}

Does it mean that there are no memory allocations after the startup phase? Let's find out. Figure @fig:StockfishAllocations shows the accumulated number of allocations over time. Similar to the consumed memory chart (Figure @fig:StockfishMemUsage), allocations are sliced according to the accumulated number of memory allocations attributed to each function. As we can see, new allocations keep coming from not just a single place, but many. The most frequent allocations are done through `operator new` that corresponds to region \circled{1} on the image.

这是否意味着启动阶段之后就没有内存分配了？让我们来验证一下。图 @fig:StockfishAllocations 显示了内存分配次数随时间的变化。与内存消耗图表（图 @fig:StockfishMemUsage）类似，内存分配根据每个函数累计的内存分配次数进行切片。我们可以看到，新的内存分配并非来自单一来源，而是来自多个来源。最频繁的分配是通过 `operator new` 完成的，这对应于图中的 \circled{1} 区域。

Notice there are new allocations at a steady pace throughout the life of the program. However, as we just saw, memory consumption doesn't change; how is that possible? Well, it can be possible if we deallocate previously allocated buffers and allocate new ones of the same size (also known as *temporary allocations*).

请注意，在程序运行期间，新的内存分配以稳定的速度进行。然而，正如我们刚才看到的，内存消耗并没有改变；这怎么可能呢？如果我们释放之前分配的缓冲区并分配相同大小的新缓冲区（也称为*临时分配（temporary allocation）*），这种情况就可能发生。

![Stockfish memory profile with Heaptrack, the number of allocations is growing. Stockfish使用Heaptrack进行内存分析，内存分配数量不断增加。](../../img/memory-access-opts/Stockfish_allocations.png){#fig:StockfishAllocations width=100%}

Since the number of allocations is growing but the total consumed memory doesn't change, we are dealing with temporary allocations. Let's find out where in the code they are coming from. It is easy to do with the help of a flame graph shown in Figure @fig:StockfishFlamegraph. There are 4800 temporary allocations in total with 90.8% of those coming from `operator new`. Thanks to the flame graph we know the entire call stack that leads to 4360 temporary allocations. Interestingly, those temporary allocations are initiated by `std::stable_sort` which allocates a temporary buffer to do the sorting. One way to get rid of those temporary allocations would be to use an in-place stable sorting algorithm. However, by doing so I observed an 8% drop in performance, so I discarded this change.

由于内存分配数量不断增加，但总内存消耗量却没有变化，这表明内存分配是临时性的。让我们找出这些临时分配的来源。借助图 @fig:StockfishFlamegraph 所示的火焰图，我们可以轻松做到这一点。总共有4800个临时分配，其中90.8%来自 `operator new`。通过火焰图，我们可以了解导致这4360个临时分配的完整调用栈。有趣的是，这些临时分配是由 `std::stable_sort` 函数发起的，该函数会分配一个临时缓冲区来进行排序。一种消除这些临时分配的方法是使用原地进行的稳定排序算法。然而，这样做导致性能下降了8%，所以我放弃了这个改动。

![Stockfish memory profile with Heaptrack, temporary allocations flamegraph. 使用Heaptrack的Stockfish内存轮廓分析，临时内存分配火焰图。](../../img/memory-access-opts/Stockfish_flamegraph.png){#fig:StockfishFlamegraph width=80%}

Similar to temporary allocations, you can also find the paths that lead to the largest allocations in a program. In the dropdown menu at the top of Figure @fig:StockfishFlamegraph, you would need to select the "Consumed" flame graph.

与临时分配类似，您也可以找到程序中导致最大内存分配的路径。在图 @fig:StockfishFlamegraph 顶部的下拉菜单中，您需要选择“已消耗”火焰图。

### Memory Intensity and Footprint 内存强度和内存占用 {#sec:MemoryIntensityFootprint}

In this section, I will show how to measure the memory *intensity* and memory *footprint* of a program. Memory intensity refers to the size of data being accessed by a program during an interval, measured, for example, in MB per second. A program with high memory intensity makes heavy use of the memory system, often accessing large amounts of data. On the other hand, a program with low memory intensity makes relatively fewer memory accesses and may be more compute-bound, meaning it spends more time performing calculations rather than waiting for data from memory. Measuring memory intensity during short time intervals enables us to observe how it changes over time. 

在本节中，我将展示如何测量程序的内存*强度intensity*和内存*占用footprint*。内存强度是指程序在一段时间内访问的数据量，例如以每秒MB为单位。内存强度高的程序会大量使用内存系统，经常访问大量数据。另一方面，内存强度低的程序内存访问相对较少，可能更受计算密集型任务的制约，这意味着它花费更多时间执行计算，而不是等待内存中的数据。在短时间间隔内测量内存强度，可以让我们观察它随时间的变化。

Memory footprint measures the total number of bytes accessed by an application. While calculating memory footprint, we only consider unique memory locations. That is, if a memory location was accessed twice during the lifetime of a program, we count the touched memory only once.

内存占用衡量应用程序访问的总字节数。在计算内存占用时，我们只考虑唯一内存位置。也就是说，如果一个内存位置在程序运行期间被访问了两次，我们只计算一次。

Figure @fig:MemFootCaseStudyFourBench shows the memory intensity and footprint of four workloads: Blender ray tracing, Stockfish chess engine, Clang++ compilation, and AI_bench PSPNet segmentation. We collect data for the chart with the Intel SDE (Software Development Emulator) tool using the method described on the Easyperf [blog](https://easyperf.net/blog/2024/02/12/Memory-Profiling-Part3)[^6] with intervals of one billion instructions. 

图 @fig:MemFootCaseStudyFourBench 展示了四种工作负载的内存强度和占用情况：Blender光线追踪、Stockfish国际象棋引擎、Clang++编译和AI_bench PSPNet分割。我们使用Intel软件开发仿真器（SDE: Software Development Emulator）工具，按照Easyperf博客[blog](https://easyperf.net/blog/2024/02/12/Memory-Profiling-Part3)[^6]中描述的方法，以十亿条指令为间隔收集图表数据。

![Memory intensity and footprint of four workloads. Intensity: total memory accessed during 1B instructions interval. Footprint: accessed memory that has not been seen before. 四种工作负载的内存强度和占用情况。强度：在十亿条指令间隔内访问的总内存量。](../../img/memory-access-opts/MemFootCaseStudyFourBench.png){#fig:MemFootCaseStudyFourBench width=100%}

The solid line (Intensity) tracks the number of bytes accessed during each interval of 1B instructions. Here, we don't count how many times a certain memory location was accessed. If a memory location was loaded twice during an interval `I`, we count the touched memory only once. However, if this memory location is accessed the third time in the subsequent interval `I+1`, it contributes to the memory intensity of the interval `I+1`. Because of this, we cannot aggregate time intervals. For example, we can see that, on average, the Blender benchmark touches roughly 20MB every interval. We cannot aggregate its 150 consecutive intervals and say that the memory footprint of Blender was `150 * 20MB = 3GB`. It would be true only if a program never repeats memory accesses across intervals.

实线（强度）追踪每1B指令间隔内访问的字节数。这里，我们不统计特定内存位置被访问的次数。如果某个内存位置在间隔 `I` 内被加载两次，我们只计算一次。但是，如果该内存位置在下一个间隔 `I+1` 内被第三次访问，则会计入间隔 `I+1` 的内存强度。因此，我们不能将时间间隔累加。例如，我们可以看到，Blender基准测试平均每个间隔访问约20MB内存。我们不能将其150个连续间隔累加，然后说Blender的内存占用为 `150 * 20MB = 3GB`。只有当程序在不同时间间隔内不会重复访问内存时，上述说法才成立。

The dashed line (Footprint) tracks the size of the new data accessed every interval since the start of the program. Here, we count the number of bytes accessed during each 1B instruction interval that have never been touched before by the program. Aggregating all the intervals (cross-hatched area under that dashed line) gives us the total memory footprint for a program. Here are the memory footprint numbers for our benchmarks: Clang C++ compilation (487MB); Stockfish (188MB); PSPNet (1888MB); and Blender (149MB). Keep in mind, that these unique memory locations can be accessed many times, so the overall pressure on the memory subsystem may be high even if the footprint is relatively small.

虚线（内存占用量）记录了自程序启动以来每个时间间隔内访问的新数据的大小。这里，我们统计了每个1B指令间隔内程序访问的、之前从未被访问过的字节数。将所有时间间隔的数据汇总（虚线下方的交叉阴影区域）即可得到程序的总内存占用量。以下是我们基准测试的内存占用量：Clang C++ 编译（487MB）；Stockfish（188MB）；PSPNet（1888MB）；以及Blender（149MB）。请注意，这些独特的内存位置可能会被多次访问，因此即使内存占用量相对较小，内存子系统的整体压力也可能很高。

As you can see our workloads have very different behavior. Clang compilation has high memory intensity at the beginning, sometimes spiking to 100MB per 1B instructions, but after that, it decreases to about 15MB per 1B instructions. Any of the spikes on the chart may be concerning to a Clang developer: are they expected? Could they be related to some memory-hungry optimization pass? Can the accessed memory locations be compacted?

正如您所见，我们的工作负载表现出截然不同的行为。 Clang编译初期内存占用较高，有时甚至会飙升至每100亿条指令100MB，但之后会降至每10亿条指令约15MB。图表中的任何峰值都可能引起Clang开发者的担忧：它们是否符合预期？是否与某些内存密集型优化步骤有关？访问的内存位置是否可以压缩？

The Blender benchmark is very stable; we can see the start and the end of each rendered frame. This enables us to focus on just a single frame, without looking at the entire workload of 1000+ frames. The Stockfish benchmark is a lot more chaotic, probably because the chess engine crunches different positions which require different amounts of resources. Finally, the PSPNet segmentation memory intensity is very interesting as we can spot repetitive patterns. After the initial startup, there are five or six sine waves from `40B` to `95B`, then three regions that end with a sharp spike to 200MB, and then again three mostly flat regions hovering around 25MB per 1B instructions. This is actionable information that can be used to optimize an application.

Blender基准测试非常稳定；我们可以看到每个渲染帧的开始和结束。这使我们能够专注于单个帧，而无需查看1000多帧的整个工作负载。Stockfish基准测试则更加混乱，可能是因为国际象棋引擎需要处理不同的局面，而每个局面所需的资源量各不相同。最后，PSPNet分割的内存占用情况非常有趣，因为我们可以发现重复出现的模式。初始启动后，内存占用会出现五六个从40MB到95MB的正弦波，然后是三个以200MB的尖峰结束的区域，之后又是三个基本平坦的区域，每条1B指令的内存占用约为25MB。这些信息可用于优化应用程序。

Such charts help us estimate memory intensity and measure the memory footprint of an application. By looking at the chart, you can observe phases and correlate them with the underlying algorithm. Ask yourself: "Does it look according to your expectations, or the workload is doing something sneaky?" You may encounter unexpected spikes in memory intensity. On many occasions, memory profiling helped identify a problem or served as an additional data point to support the conclusions that were made during regular profiling.

这类图表有助于我们估算应用程序的内存使用强度并测量其内存占用量。通过查看图表，您可以观察各个阶段，并将其与底层算法关联起来。不妨问问自己：“这是否符合您的预期？或者工作负载是否在暗中进行了一些操作？”您可能会遇到意料之外的内存使用强度峰值。在许多情况下，内存分析有助于识别问题，或作为补充数据点，支持常规分析中得出的结论。

In some scenarios, memory footprint helps us estimate the pressure on the memory subsystem. For instance, if the memory footprint is small (several megabytes), we might suspect that the pressure on the memory subsystem is low since the data will likely reside in the L3 cache; remember that available memory bandwidth in modern processors ranges from tens to hundreds of GB/s and is getting close to 1 TB/s. On the other hand, when we're dealing with an application that accesses 10 GB/s and the memory footprint is much bigger than the size of the L3 cache, then the workload might put significant pressure on the memory subsystem.

在某些情况下，内存占用量有助于我们估算内存子系统的压力。例如，如果内存占用量很小（几兆字节MB），我们可能会认为内存子系统的压力较低，因为数据很可能驻留在L3缓存中；请记住，现代处理器的可用内存带宽范围从几十到几百GB/s，并且已经接近1TB/s。另一方面，如果应用程序的访问速度达到10GB/s，并且内存占用量远大于L3缓存的大小，那么工作负载可能会给内存子系统带来显著的压力。

While memory profiling techniques discussed in this chapter certainly help better understand the behavior of a workload, this is not enough to fully assess the temporal and spatial locality of memory accesses. We still have no visibility into how many times a certain memory location was accessed, what is the time interval between two consecutive accesses to the same memory location, and whether the memory is accessed sequentially, with strides, or completely random. The topic of data locality of applications has been researched for a long time. Unfortunately, as of early 2024, there are no production-quality tools available that would give us such information. Further details are beyond the scope of this book, however, curious readers are welcome to read the related [article](https://easyperf.net/blog/2024/02/12/Memory-Profiling-Part5)[^9] on the Easyperf blog.

虽然本章讨论的内轮廓存分析技术确实有助于更好地理解工作负载的行为，但这不足以全面评估内存访问的时间和空间局部性。我们仍然无法了解特定内存位置被访问了多少次，两次连续访问同一内存位置之间的时间间隔是多少，以及内存访问是顺序的、有步长的还是完全随机的。应用程序的数据局部性问题已被研究多年。遗憾的是，截至2024年初，还没有生产级工具能够提供此类信息。更多细节超出了本书的范围，但感兴趣的读者可以阅读Easyperf博客上的相关文章(https://easyperf.net/blog/2024/02/12/Memory-Profiling-Part5)[^9]。

[^1]: Intel SDE - [https://www.intel.com/content/www/us/en/developer/articles/tool/software-development-emulator.html](https://www.intel.com/content/www/us/en/developer/articles/tool/software-development-emulator.html)
[^2]: Heaptrack - [https://github.com/KDE/heaptrack](https://github.com/KDE/heaptrack)
[^3]: MTuner - [https://github.com/milostosic/MTuner](https://github.com/milostosic/MTuner)
[^4]: Stockfish - [https://github.com/official-stockfish/Stockfish](https://github.com/official-stockfish/Stockfish)
[^5]: AI Benchmark Alpha - [https://ai-benchmark.com/alpha.html](https://ai-benchmark.com/alpha.html)
[^6]: Easyper blog: Measuring memory footprint with SDE  Easyper博客：使用SDE测量内存占用 - [https://easyperf.net/blog/2024/02/12/Memory-Profiling-Part3](https://easyperf.net/blog/2024/02/12/Memory-Profiling-Part3)
[^7]: psutil - [https://github.com/giampaolo/psutil](https://github.com/giampaolo/psutil)
[^8]: RAMMap - [https://learn.microsoft.com/en-us/sysinternals/downloads/rammap](https://learn.microsoft.com/en-us/sysinternals/downloads/rammap)
[^9]: Easyperf blog: Data Locality and Reuse Distances Easyperf博客：数据局部性和重用距离 - [https://easyperf.net/blog/2024/02/12/Memory-Profiling-Part5](https://easyperf.net/blog/2024/02/12/Memory-Profiling-Part5)
