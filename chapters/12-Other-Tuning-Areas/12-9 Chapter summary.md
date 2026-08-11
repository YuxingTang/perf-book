## Chapter Summary 本章小结 {.unlisted .unnumbered}

\markright{Summary小结}

* Processors from different vendors are not created equal. They differ in terms of instruction set architecture (ISA) that they support and microarchitecture implementation. Reaching peak performance often requires leveraging the latest ISA extensions and tuning the application for a specific CPU microarchitecture.
* CPU dispatching is a technique that enables you to introduce platform-specific optimizations. Using it, you can provide a fast path for a specific microarchitecture while keeping a generic implementation for other platforms.
* We explored several performance corner cases that are caused by the interaction of the application with the CPU microarchitecture. These include memory ordering violations, misaligned memory accesses, cache aliasing, and denormal floating-point numbers.
* We also discussed a few low-latency tuning techniques that are essential for applications that require fast response times. We showed how to avoid page faults, cache misses, TLB shootdowns, and core throttling on a critical path.
* System tuning is the last piece of the puzzle. Some knobs and settings may affect the performance of your application. It is crucial to ensure that the system firmware, the OS, or the kernel does not destroy all the efforts put into tuning the application. 

* 不同厂商的处理器性能并不相同。它们支持的指令集体系结构(ISA)和微体系结构实现各不相同。要达到最佳性能，通常需要利用最新的ISA扩展，并针对特定的CPU微体系结构对应用程序进行调优。
* CPU调度是一种可以引入平台特定优化的技术。利用它，您可以为特定的微体系结构提供快速路径，同时保持对其他平台的通用实现。
* 我们探讨了由应用程序与CPU微体系结构交互引起的几种性能极端情况。这些情况包括：存储序违反、内存访问未对齐、缓存别名和非规格化浮点数。
* 我们还讨论了一些低延迟调优技术，这些技术对于需要快速响应时间的应用程序至关重要。我们展示了如何避免页面错误、缓存未命中、TLB崩溃以及关键路径上的核心节流。
* 系统调优是最后一块拼图。某些参数和设置可能会影响应用程序的性能。必须确保系统固件、操作系统或内核不会破坏所有为调整应用程序而付出的努力。

\sectionbreak



