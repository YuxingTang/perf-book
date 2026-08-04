# CPU Features for Performance Analysis 针对性能分析的CPU特性 {#sec:PmuChapter}

The ultimate goal of performance analysis is to identify performance bottlenecks and locate parts of the code that are associated with them. Unfortunately, there are no predetermined steps to follow, so it can be approached in many different ways. 

性能分析的最终目标是识别性能瓶颈并定位与其相关的代码部分。遗憾的是，没有预先设定的步骤，因此可以采用多种不同的方法。

Usually, profiling an application can give quick insights into the hotspots of the application. Sometimes it’s all that’s needed to help developers find and fix the performance problems. Especially high-level performance problems can often be revealed by profiling. For example, consider a situation when you've just made a change to the function `foo` in your application and suddenly see a noticeable performance degradation. So, you decide to profile the application. According to your mental model of the application, you expect that `foo` is a cold function and it doesn't show up in the top-10 list of hot functions. But when you open the profile, you see it consumes a lot more time than before. You quickly realize the mistake you've made in the code and fix it. If all issues in performance engineering were that easy to fix, this book would not exist.

通常，对应用程序进行性能分析可以快速了解应用程序的性能瓶颈。有时，这足以帮助开发人员找到并修复性能问题。尤其是一些高层级的性能问题，通常可以通过性能分析发现。例如，假设您刚刚修改了应用程序中的 `foo` 函数，突然发现性能明显下降。于是，您决定对应用程序进行性能分析。根据您对应用程序的理解，您认为 `foo` 是一个冷函数，不会出现在前十个热门函数的列表中。但是，当您打开性能分析报告时，发现它比以前消耗的时间多了很多。您很快意识到代码中的错误并进行了修复。如果性能工程中的所有问题都如此容易解决，这本书也就不存在了。

When you embark on a journey to squeeze the last bit of performance from your application, simply looking at the list of hotspots is not enough. Unless you have a crystal ball or an accurate model of an entire CPU in your head, you need additional support to understand what the performance bottlenecks are.

当你开始探索如何榨干应用程序的最后一丝性能时，仅仅查看性能瓶颈列表是不够的。除非你拥有水晶球或对整个CPU的运作机制了如指掌，否则你需要额外的帮助才能理解性能瓶颈究竟在哪里。

Some developers rely on their intuition and proceed with random experiments, trying to force various compiler optimizations like loop unrolling, vectorization, inlining, you name it. Indeed, sometimes you can be lucky and enjoy a portion of compliments from your colleagues and maybe even claim an unofficial title of performance guru on your team. But usually, you need to have a very good intuition and luck. In this book, we don't teach you how to be lucky. Instead, we show methods that have proved to be working in practice.

一些开发者依靠直觉进行随机实验，尝试各种编译器优化，例如：循环展开、向量化、内联等等。的确，有时你可能会走运，赢得同事的赞赏，甚至在团队中赢得一个非正式的“性能大师”称号。但通常情况下，你需要非常敏锐的直觉和一点运气。本书并非教你如何获得好运，而是展示那些在实践中已被证明行之有效的方法。

Modern CPUs are constantly getting new features that enhance performance analysis in different ways. Using those features greatly simplifies finding low-level issues like cache misses, branch mispredictions, etc. In this chapter, we will take a look at a few hardware performance monitoring capabilities available on modern CPUs. Processors from different vendors do not necessarily have the same set of features. We will explore performance monitoring capabilities available in Intel, AMD, and Arm processors.[^1]

现代CPU不断推出新功能特性，从各个方面增强性能分析。使用这些特性可以极大地简化查找底层问题，例如：缓存未命中、分支预测错误等。本章将介绍现代CPU上可用的一些硬件性能监控功能。不同厂商的处理器不一定具备相同的功能集。我们将探讨在Intel、AMD和Arm处理器中可用的性能监控功能[^1]。

* **Top-down Microarchitecture Analysis** (TMA) methodology, discussed in [@sec:TMA]. This is a powerful technique for identifying ineffective usage of CPU microarchitecture by a program. It characterizes the bottleneck of a workload and allows locating the exact place in the source code where it occurs. It abstracts away the intricacies of the CPU microarchitecture and is relatively easy to use even for inexperienced developers.
* **Branch Recording**, discussed in [@sec:lbr]. This is a mechanism that continuously logs the most recent branch outcomes in parallel with executing the program. It is used for collecting call stacks, identifying hot branches, calculating misprediction rates of individual branches, and more.
* **Hardware-Based Sampling**, discussed in [@sec:secPEBS]. This is a feature that enhances sampling. Its primary benefits include: lowering the overhead of sampling and providing "Precise Events" capability, that enables pinpointing of the exact instruction that caused a particular performance event.
* **Intel Processor Traces** (PT), discussed in Appendix C. It is a facility to record and reconstruct the program execution with a timestamp on *every* instruction. Its main usages are postmortem 
analysis and root-causing performance glitches.

* **自顶向下微体系结构分析** (TMA: Top-down Microarchitecture Analysis) 方法，详见 [@sec:TMA]。这是一种强大的技术，用于识别程序对CPU微体系结构的低效使用。它可以描述工作负载的瓶颈，并允许定位源代码中瓶颈发生的确切位置。它抽象化了CPU微体系结构的复杂性，即使对于经验不足的开发人员来说也相对容易使用。
* **分支记录**，详见 [@sec:lbr]。这是一种在程序执行的同时持续记录最新分支结果的机制。它用于收集调用堆栈、识别热点分支、计算单个分支的误预测率等等。
* **基于硬件的采样**，详见[@sec:secPEBS]。这是一项增强采样的功能体系。其主要优势包括：降低采样开销，并提供“精确事件”功能，从而能够精确定位导致特定性能事件的指令。
* **Intel处理器跟踪** (PT: Processor Traces)，详见附录C。它是一种记录和重建程序执行过程的工具，并为*每条*指令添加时间戳。其主要用途是事后分析和性能故障的根本原因分析。

The Intel PT feature is covered in Appendix C. Intel PT was supposed to be an "end game" for performance analysis. With its low runtime overhead, it is a very powerful analysis feature. But it turns out to be not very popular among performance engineers. Partially because the support in the tools is not mature, and partially because in many cases it is overkill, and it's just easier to use a sampling profiler. Also, it produces a lot of data, which is not practical for long-running workloads. Nevertheless, it is popular in some industries, such as high-frequency trading (HFT).

Intel处理器跟踪PT功能特性详见附录C。Intel处理器PT原本被认为是性能分析的“终极方案”。凭借其低运行时开销，它是一项非常强大的分析功能。但事实证明，它在性能工程师中并不受欢迎。部分原因是相关工具的支持尚不成熟，部分原因是很多情况下它过于复杂，使用采样分析器反而更简单。此外，它会产生大量数据，这对于长时间运行的工作负载来说并不实用。尽管如此，它在某些行业，例如高频交易(HFT: High-Frequency Trading)领域，仍然很受欢迎。

The hardware performance monitoring features mentioned above provide insights into the efficiency of a program from the CPU perspective. In the next chapter, we will discuss how profiling tools use these features to provide many different types of analysis.

上文提到的硬件性能监控功能特性可以从CPU的角度深入了解程序的效率。下一章我们将讨论分析工具如何利用这些功能提供多种类型的分析。

[^1]: The RISC-V ecosystem does not yet have a mature performance monitoring infrastructure, so we will not cover it in this book. RISC-V生态系统目前还没有成熟的性能监控基础设施，因此本书暂不涉及这方面内容。
