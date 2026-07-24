## Chapter Summary 本章小结 {.unlisted .unnumbered}

\markright{Summary小结}

* In this chapter, we introduced the basic metrics in performance analysis such as retired/executed instructions, CPU utilization, IPC/CPI, $\mu$ops, pipeline slots, core/reference clocks, cache misses, and branch mispredictions. We showed how each of these metrics can be collected with Linux `perf`.
* 本章介绍了性能分析中的基本指标，例如：已执行/已完成指令数、CPU利用率、IPC/CPI、微操作μops、流水线槽位、核心/参考时钟频率、缓存未命中和分支预测错误。我们展示了如何使用Linux `perf` 命令收集这些指标。
* For more advanced performance analysis, there are many derivative metrics that you can collect. For instance, cache misses per kilo instructions (MPKI), instructions per function call, branch, load, etc. (Ip*), ILP, MLP, and others. The case studies in this chapter show how you can get actionable insights from analyzing these metrics. 
* 对于更高级的性能分析，您可以收集许多衍生指标。例如，每千条指令缓存未命中数(MPKI)、每函数调用指令数、分支、加载等指令数 (Ip*)、指令级并行度 (ILP)、存储级并行度(MLP)等。本章的案例研究展示了如何通过分析这些指标获得可操作的见解。
* Be careful about drawing conclusions just by looking at the aggregate numbers. Don't fall into the trap of "Excel performance engineering", i.e., only collecting performance metrics and never looking at the code. Always seek a second source of data (e.g., performance profiles, discussed later) to verify your ideas.
* 切勿仅凭汇总数据就得出结论。不要陷入“Excel性能工程”的陷阱，即只收集性能指标而不查看代码。始终寻找第二个数据来源（例如：性能分析轮廓报告，稍后讨论）来验证您的想法。
* Memory bandwidth and latency are crucial factors in the performance of many production software packages nowadays, including AI, HPC, databases, and many general-purpose applications. Memory bandwidth depends on the DRAM speed (in MT/s) and the number of memory channels. Modern high-end server platforms have 8--12 memory channels and can reach up to 500 GB/s for the whole system and up to 50 GB/s in single-threaded mode. Memory latency nowadays doesn't change a lot, in fact, it is getting slightly worse with new DDR4 and DDR5 generations. The majority of modern client-facing systems fall in the range of 70--110 ns latency per memory access. Server platforms may have higher memory latencies.
* 内存带宽和延迟是当今许多生产力软件（包括：人工智能、高性能计算、数据库和许多通用应用程序）性能的关键因素。内存带宽取决于DRAM速度（以MT/s为单位）和内存通道数。现代高端服务器平台拥有8到12个内存通道，整个系统的内存带宽可达500GB/s，单线程模式下可达50GB/s。如今，内存延迟变化不大，实际上，随着DDR4和DDR5新一代内存的推出，延迟略有增加。大多数现代面向客户端的系统每次内存访问的延迟在70到110纳秒之间。服务器平台的内存延迟可能更高。

\sectionbreak



