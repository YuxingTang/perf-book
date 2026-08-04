## Chapter Summary 章节小结 {.unlisted .unnumbered}

\markright{Summary小结}

* Modern processors provide features that enhance performance analysis. Using those features greatly simplifies finding opportunities for low-level optimization.
* 现代处理器提供了增强性能分析的功能。利用这些功能可以极大地简化寻找底层优化机会的过程。
* Top-down Microarchitecture Analysis (TMA) methodology is a powerful technique for identifying ineffective usage of CPU microarchitecture by a program, and is easy to use even for inexperienced developers. TMA is an iterative process that consists of multiple steps, including characterizing the workload and locating the exact place in the source code where the bottleneck occurs. We advise that TMA should be one of the starting points for every low-level tuning effort.
* 自顶向下微架构分析(TMA: Top-down Microarchitecture)方法是一种强大的技术，可以识别程序对CPU微体系结构的低效使用，即使对于经验不足的开发人员也易于使用。TMA是一个迭代过程，包含多个步骤，包括表征工作负载和定位源代码中瓶颈发生的确切位置。我们建议将TMA作为所有底层调优工作的起点之一。
* Branch Record mechanisms such as Intel's LBR, AMD's LBR, and ARM's BRBE continuously log the most recent branch outcomes in parallel with executing the program, causing a minimal slowdown. One of the primary usages of these facilities is to collect call stacks. Also, they help identify hot branches, and misprediction rates and enable precise timing of machine code.
* 分支记录机制（例如：Intel的LBR、AMD的LBR和ARM的BRBE）会在程序执行的同时持续记录最新的分支结果，从而最大限度地减少性能下降。这些机制的主要用途之一是收集调用堆栈。此外，它们还有助于识别热点分支和误预测率，并实现对机器代码的精确计时。
* Modern processors often provide Hardware-Based Sampling features for advanced profiling. Such features lower the sampling overhead by storing multiple samples in a dedicated buffer without software interrupts. They also introduce "Precise Events" that enable pinpointing the exact instruction that caused a particular performance event. In addition, there are several other less important use cases. Example implementations of such Hardware-Based Sampling features include Intel's PEBS, AMD's IBS, and ARM's SPE.
* 现代处理器通常提供基于硬件的采样功能，用于高级性能分析。这些特性通过在专用缓冲​​区中存储多个样本来降低采样开销，而无需软件中断。它们还引入了“精确事件Precise Event”，能够精确定位导致特定性能事件的指令。此外，还有一些其他不太重要的用例。此类基于硬件的采样特性的示例实现包括Intel的PEBS、AMD的IBS和ARM的SPE。
* Intel Processor Traces (PT) is a CPU feature that records the program execution by encoding packets in a highly compressed binary format that can be used to reconstruct execution flow with a timestamp on every instruction. PT has extensive coverage and a relatively small overhead. Its main usages are postmortem analysis and finding the root cause(s) of performance glitches. The Intel PT feature is covered in Appendix C. Processors based on ARM architecture also have a tracing capability called Arm [CoreSight](https://developer.arm.com/ip-products/system-ip/coresight-debug-and-trace),[^2] but it is mostly used for debugging rather than for performance analysis.
* Intel处理器跟踪(PT: Processor Trace)是一项 CPU 特性，它通过将数据包编码为高度压缩的二进制格式来记录程序执行过程，该格式可用于重建执行流程，并为每条指令添加时间戳。PT覆盖范围广，开销相对较小。其主要用途是事后分析和查找性能故障的根本原因。Intel PT功能特性在附录C中进行了介绍。基于ARM体系结构的处理器也具备名为Arm CoreSight的跟踪功能，但它主要用于调试而非性能分析。

[^2]: Arm CoreSight - [https://developer.arm.com/ip-products/system-ip/coresight-debug-and-trace](https://developer.arm.com/ip-products/system-ip/coresight-debug-and-trace)

\sectionbreak
