# Optimizing Memory Accesses 优化内存访问 {#sec:MemBound}

Modern computers are still being built based on the classical Von Neumann architecture which decouples CPU, memory, and input/output units. Nowadays, operations with memory (loads and stores) account for the largest portion of performance bottlenecks and power consumption. It is no surprise that we start with this category.

现代计算机仍然基于经典的冯·诺依曼体系结构构建，该体系结构将CPU、内存和输入/输出单元解耦。如今，存储操作（加载load和存储store）是性能瓶颈和功耗的主要来源。因此，我们从这一类别入手也就不足为奇了。

The statement that memory hierarchy performance is critical can be illustrated by Figure @fig:CpuMemGap. It shows the growth of the gap in performance between memory and processors. The vertical axis is on a logarithmic scale and shows the growth of the CPU-DRAM performance gap. The memory baseline is the latency of memory access of 64 KB DRAM chips from 1980. Typical DRAM performance improvement is 7% per year, while CPUs enjoy 20-50% improvement per year. According to this picture, processor performance has plateaued, but even then, the gap in performance remains. [@Hennessy]

存储层次结构性能至关重要，这一点可以从图 @fig:CpuMemGap 中看出。该图显示了内存和处理器之间性能差距的增长。纵轴采用对数刻度，表示CPU和DRAM性能差距的增长。内存基准是1980年64KB DRAM芯片的存储访问延迟。DRAM的性能通常每年提升7%，而CPU的性能每年提升20-50%。根据这张图，处理器性能已经趋于稳定，但即便如此，性能差距依然存在。[@Hennessy]

![The gap in performance between memory and processors. *© Source: [@Hennessy].* 内存和处理器之间的性能差距。*© 来源：[@Hennessy].*](../../img/memory-access-opts/ProcessorMemoryGap.png){#fig:CpuMemGap width=90%}

A variable can be fetched from the smallest L1 cache in just a few clock cycles, but it can take more than three hundred clock cycles to fetch a variable from DRAM if it is not in the CPU cache. From a CPU perspective, a last-level cache miss feels like a *very* long time, especially if the processor is not doing any useful work during that time. Execution threads may also be starved when the system is highly loaded with threads accessing memory at a very high rate and there is no available memory bandwidth to satisfy all loads and stores promptly.

从最小的L1缓存中获取一个变量只需几个时钟周期，但如果变量不在CPU缓存中，则从DRAM中获取该变量可能需要超过三百个时钟周期。从CPU的角度来看，末级缓存未命中感觉就像*非常*长的时间，尤其是在处理器在此期间没有执行任何有效工作的情况下。当系统负载很高，线程以极高的频率访问内存，而没有足够的可用内存带宽来及时满足所有加载和存储操作时，执行线程也可能出现内存不足的饥饿情况。

When an application executes a large number of memory accesses and spends significant time waiting for them to finish, such an application is characterized as being bounded by memory. It means that to further improve its performance, we likely need to improve how we access memory, reduce the number of such accesses, or upgrade the memory subsystem itself. 

当应用程序执行大量内存访问并花费大量时间等待这些访问完成时，这样的应用程序就被认为是受内存限制的。这意味着，为了进一步提升性能，我们可能需要改进内存访问方式、减少内存访问次数，或者升级内存子系统本身。

In the TMA methodology, the `Memory Bound` metric estimates a fraction of slots where a CPU pipeline is likely stalled due to demand for load or store instructions. The first step to solving such a performance problem is to locate the memory accesses that contribute to the high `Memory Bound` metric (see [@sec:secTMA_Intel]). Once a troublesome memory access issue is identified, several optimization strategies might be applied. In this chapter, we will discuss techniques to improve memory access patterns.

在TMA方法论中，`内存瓶颈Memory Bound` 指标用于估算CPU流水线因加载load或存储store指令需求而可能停滞的槽位比例。解决此类性能问题的第一步是定位导致 `内存瓶颈Memory Bound` 指标较高的内存访问（参见[@sec:secTMA_Intel]）。一旦确定了棘手的内存访问问题，就可以应用多种优化策略。本章将讨论改进内存访问模式的技术。