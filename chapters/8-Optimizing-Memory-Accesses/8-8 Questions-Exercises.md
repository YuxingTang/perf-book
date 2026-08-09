## Questions and Exercises 问题与习题 {.unlisted .unnumbered}

\markright{Questions and Exercises问题与习题}

1. Solve the `perf-ninja::data_packing` lab assignment, in which you need to make a data structure more compact.
2. Solve the `perf-ninja::huge_pages_1` lab assignment using methods we discussed in [@sec:secDTLB]. Observe any changes in performance, huge page allocation in `/proc/meminfo`, and CPU performance counters that measure DTLB loads and misses.
3. Solve the `perf-ninja::swmem_prefetch_1` lab assignment by implementing explicit memory prefetching for future loop iterations.
4. Describe in general terms what it takes for a piece of code to be cache-friendly.
5. Run the application that you're working with daily. Measure its memory utilization and analyze heap allocations using memory profilers that we discussed in [@sec:MemoryProfiling]. Identify hot memory accesses using Linux perf, Intel VTune, or other profiler. Is there a way to improve those accesses?

1. 完成 `perf-ninja::data_packing` 实验作业，要求你优化数据结构，使其更加紧凑。
2. 使用我们在 [@sec:secDTLB] 中讨论的方法完成 `perf-ninja::huge_pages_1` 实验作业。观察性能变化、`/proc/meminfo` 中的大页分配情况，以及用于衡量DTLB加载和未命中的CPU性能计数器。
3. 通过为后续循环迭代实现显式内存预取，完成 `perf-ninja::swmem_prefetch_1` 实验作业。
4. 概括描述一段代码如何才能对缓存友好。
5. 运行你日常使用的应用程序。使用我们在 [@sec:MemoryProfiling] 中讨论的内存分析器测量其内存利用率并分析堆分配情况。使用Linux perf、Intel VTune或其他分析器识别热点内存访问。有什么办法可以改善这些访问方式吗？