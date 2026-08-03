# Performance Analysis Approaches 性能分析方法 {#sec:sec_PerfApproaches}

When you're working on a high-level optimization, e.g., integrating a better algorithm into an application, it is usually easy to tell whether the performance improves or not since the benchmarking results are usually pronounced. Big speedups, like 2x, 3x, etc., are relatively obvious from a performance analysis perspective. When you eliminate an extensive computation from a program, you expect to see a visible difference in the running time. 

当您进行高层次优化时，例如将更优的算法集成到应用程序中，通常很容易判断性能是否有所提升，因为基准测试结果通常非常显著。例如，2倍、3倍等大幅提升，从性能分析的角度来看相对容易察觉。当您从程序中移除大量计算时，您会期望看到运行时间的明显变化。

But also, there are situations when you see a small change in the execution time, say 5%, and you have no clue where it's coming from. Timing or throughput measurements alone do not provide any explanation for why performance goes up or down. In this case, we need more insights about how a program executes. That is the situation when we need to do performance analysis to understand the underlying nature of the slowdown or speedup that we observe.

但是，有时您会发现执行时间的变化很小，比如5%，却完全不知道原因。仅凭计时或吞吐率测量无法解释性能上升或下降的原因。在这种情况下，我们需要更深入地了解程序的执行方式。这时就需要进行性能分析，以理解我们观察到的性能下降或提升的根本原因。

Performance analysis is akin to detective work. To solve a performance mystery, you need to gather all the data that you can and try to form a hypothesis. Once a hypothesis is made, you design an experiment that will either prove or disprove it. It can go back and forth a few times before you find a clue. And just like a good detective, you try to collect as many pieces of evidence as possible to confirm or refute your hypothesis. Once you have enough clues, you make a compelling explanation for the behavior you're observing.

性能分析就像侦探工作一样。要解决性能问题， 你需要收集尽可能多的数据并尝试提出假设。一旦提出假设，你就可以设计实验来验证或推翻它。在找到线索之前，可能需要反复尝试几次。就像优秀的侦探一样，你会尽可能多地收集证据来证实或反驳你的假设。一旦掌握了足够的线索，你就可以对你观察到的现象做出令人信服的解释。

When you just start working on a performance issue, you probably only have measurements, e.g., before and after the code change. Based on those measurements you conclude that the program became slower by `X` percent. If you know that the slowdown occurred right after a certain commit, that may already give you enough information to fix the problem. But if you don't have good reference points, then the set of possible reasons for the slowdown is endless, and you need to gather more data. One of the most popular approaches for collecting such data is to profile an application and look at the hotspots. This chapter introduces this and several other approaches for gathering data that have proven to be useful in performance engineering. 

当你刚开始处理性能问题时，你可能只有一些测量数据，例如：代码更改前后的测量数据。基于这些测量数据，你得出结论：程序速度降低了 `X` %。如果你知道速度下降发生在某个提交之后，这可能已经提供了足够的信息来解决问题。但是，如果你没有可靠的参考点，那么导致速度下降的原因就无穷无尽，你需要收集更多的数据。收集此类数据最常用的方法之一是对应用程序进行性能分析，并查看性能瓶颈。本章将介绍这种方法以及其他几种已被证明在性能工程中有效的性能数据收集方法。

The next question comes: "What performance data are available and how to collect them?" Both hardware and software layers of the stack have facilities to track performance events and record them while a program is running. In this context, by hardware, we mean the CPU, which executes the program, and by software, we mean the OS, libraries, the application itself, and other tools used for the analysis. Typically, the software stack provides high-level metrics like time, number of context switches, and page faults, while the CPU monitors cache misses, branch mispredictions, and other CPU-related events. Depending on the problem you are trying to solve, some metrics are more useful than others. So, it doesn't mean that hardware metrics will always give us a more precise overview of the program execution. They are just different. Some metrics, like the number of context switches, for instance, cannot be provided by a CPU. Performance analysis tools, like Linux `perf`, can consume data from both the OS and the CPU.

接下来的问题是：“有哪些性能数据可用，以及如何收集这些数据？”。 计算栈的硬件层和软件层都提供了跟踪性能事件并在程序运行时记录这些事件的功能。在此地的上下文中，“硬件”指的是执行程序的CPU，而“软件”指的是操作系统、库、应用程序本身以及用于分析的其他工具。通常，软件栈提供诸如时间、上下文切换次数和缺页错误等高级指标，而CPU则监控缓存未命中、分支预测错误和其他CPU相关事件。根据你尝试解决的问题，某些指标比其他指标更有用。因此，硬件指标并非总是能提供更精确的程序执行概览。它们只是有所不同。例如，某些指标（如：上下文切换次数）无法由CPU提供。性能分析工具（例如Linux的 `perf`）可以同时使用来自操作系统和CPU的数据。

As you have probably guessed, there are hundreds of data sources that a performance engineer may use. This chapter is mostly about collecting hardware-level information. We will introduce some of the most popular performance analysis techniques: code instrumentation, tracing, characterization, sampling, and the Roofline model. We also discuss static performance analysis techniques and compiler optimization reports that do not involve running the actual application.

正如您可能已经猜到的，性能工程师可以使用数百种数据源。本章主要介绍硬件级信息的收集。我们将介绍一些最常用的性能分析技术：代码插桩、跟踪、特征分析、采样和屋顶Roofline模型。我们还将讨论静态性能分析技术和编译器优化报告，这些技术无需运行实际应用程序。
