## Chapter Summary 本章小结 {.unlisted .unnumbered}

\markright{Summary小结}

* We gave a quick overview of the most popular tools available on three major platforms: Linux, Windows, and MacOS. Depending on the CPU vendor, the choice of a profiling tool will vary. For systems with an Intel processor we recommend using VTune; for systems with an AMD processor use uProf; on Apple platforms use Xcode Instruments. 
* Linux perf is probably the most frequently used profiling tool on Linux. It has support for processors from all major CPU vendors. It doesn't have a graphical interface. However, there are tools that can visualize `perf`'s profiling data.
* We also discussed Windows Event Tracing (ETW), which is designed to observe software dynamics in a running system. Linux has a similar tool called [KUtrace](https://github.com/dicksites/KUtrace),[^1] which is covered in the book [@DickSitesBook].
* There are hybrid profilers that combine techniques like code instrumentation, sampling, and tracing. This takes the best of these approaches and allows users to get very detailed information on a specific piece of code. In this chapter, we looked at Tracy, which is quite popular among game developers.
* Memory profilers provide information about memory usage, heap allocations, memory footprint, and other metrics. Memory profiling helps you understand how an application uses memory over time.
* Continuous profiling tools have already become an essential part of monitoring performance in production environments. They collect system-wide performance metrics with call stacks for days, weeks, or even months. Such tools make it easier to spot the point in time when a performance change started and determine the root cause of an issue.

* 我们简要概述了三大主流平台（Linux、Windows和macOS）上最常用的性能分析工具。根据 CPU厂商的不同，性能分析工具的选择也会有所不同。对于使用Intel处理器的系统，我们推荐使用 VTune；对于使用AMD处理器的系统，建议使用uProf；在Apple平台上，建议使用Xcode Instruments。
* Linux perf可能是Linux 统上最常用的性能分析工具。它支持所有主流CPU厂商的处理器。它没有图形界面。不过，有一些工具可以可视化 `perf` 的性能分析数据。
* 我们还讨论了Windows事件跟踪(ETW: Event Tracing of Windows)，它旨在观察运行系统中的软件动态。Linux有一个类似的工具，名为[KUtrace](https://github.com/dicksites/KUtrace)[^1]，[@DickSitesBook](https://github.com/dicksites/KUtrace)一书对此进行了介绍。
* 还有一些混合型性能分析器，它们结合了代码插桩、采样和跟踪等技术。这种方法融合了上述方法的优点，使用户能够获取特定代码段的非常详细的信息。本章我们介绍了Tracy，它在游戏开发者中非常流行。
* 内存分析器提供有关内存使用情况、堆分配、内存占用和其他指标的信息。内存分析有助于您了解应用程序随时间推移的内存使用情况。
* 持续性能分析工具已成为生产环境中性能监控的重要组成部分。它们可以收集数天、数周甚至数月的系统级性能指标以及调用堆栈。此类工具可以更轻松地找到性能变化开始的时间点，并确定问题的根本原因。

[^1]: KUtrace - [https://github.com/dicksites/KUtrace](https://github.com/dicksites/KUtrace)

\sectionbreak
