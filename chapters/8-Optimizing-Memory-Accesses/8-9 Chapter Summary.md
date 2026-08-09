## Chapter Summary 本章小结 {.unlisted .unnumbered}

\markright{Summary小结}

* Most real-world applications experience memory-related performance bottlenecks. Emerging application domains, such as machine learning and big data, are particularly demanding in terms of memory bandwidth and latency.
* The performance of the memory subsystem is not growing as fast as the CPU performance. Yet, memory accesses are a frequent source of performance problems in many applications. Speeding up such programs requires revising the way they access memory.
* In [@sec:MemBound], we discussed frequently used recipes for developing cache-friendly data structures, explored data reorganization techniques, learned how to utilize huge memory pages to improve DTLB performance, and how to use explicit memory prefetching to reduce the number of cache misses.

* 大多数实际应用都会遇到内存相关的性能瓶颈。新兴应用领域，例如机器学习和大数据，对内存带宽和延迟的要求尤其高。
* 内存子系统的性能增长速度不及CPU性能。然而，内存访问却是许多应用中常见的性能问题根源。要提升这类程序的运行速度，就需要改进它们的内存访问方式。
* 在 [@sec:MemBound] 中，我们讨论了开发缓存友好型数据结构的常用方法，探索了数据重组技术，学习了如何利用大内存页来提升DTLB性能，以及如何使用显式内存预取来减少缓存未命中次数。

\sectionbreak
