## Top-down Microarchitecture Analysis 自上而下的微体系结构分析 {#sec:TMA}

Top-down Microarchitecture Analysis (TMA) methodology is a very powerful technique for identifying CPU bottlenecks in a program. The best part of this methodology is that it does not require a developer to have a deep understanding of the microarchitecture and PMCs in the system and still efficiently find CPU bottlenecks.

自顶向下微体系结构分析(TMA: Top-down Mircoarchitecture Analysis)方法是一种非常强大的技术，用于识别程序中的CPU瓶颈。这一方法最好的部分是，它不需要开发人员深入了解系统中的微体系结构和PMC，仍然可以有效地找到 CPU 瓶颈。

At a conceptual level, TMA identifies what is stalling the execution of a program. Figure @fig:TMA_concept illustrates the core idea of TMA. Here is a short guide on how to read this diagram. As we know from [@sec:uarch], there are internal buffers in the CPU that keep track of information about $\mu$ops that are being executed. Whenever a new instruction is fetched and decoded, new entries in those buffers are allocated. If a $\mu$op for the instruction was not allocated during a particular cycle of execution, it could be for one of two reasons: either we were not able to fetch and decode it (`Frontend Bound`), or the Backend was overloaded with work, and resources for the new $\mu$op could not be allocated (`Backend Bound`). If a $\mu$op was allocated and scheduled for execution but never retired, this means it came from a mispredicted path (`Bad Speculation`). Finally, `Retiring` represents a normal execution. It is the bucket where we want all our $\mu$ops to be, although there are exceptions which we will talk about later.

在概念层面上，TMA识别是什么阻碍了程序的执行。图 @fig:TMA_concept 说明了TMA的核心思想。这是有关如何阅读此图的简短指南。正如我们从 [@sec:uarch] 中了解到的，CPU中有内部缓冲区来跟踪正在执行的微操作的信息。无论一条新指令何时被获取并被译码，缓冲区中会分配一个新的条目。如果在特定的执行周期内未分配指令的微操作，可能是由于以下两个原因之一：要么我们无法获取并译码指令（`前端瓶颈Frontend Bound`），要么后端工作超载，并且无法为新的微操作分配资源（`后端瓶颈Backend Bound`）。如果微操作被分配并计划执行但从未完成，这意味着它来自错误预测的路径（`错误前瞻Bad Speculation`）。最后`退役Retiring`代表正常执行。这是我们希望所有微操作进入的存储桶bucket，尽管也有一些例外情况，我们稍后将讨论。

![The concept behind TMA's top-level breakdown. *© Source: [@TMA_ISPASS]* TMA 顶层细分背后的概念。 *©来源：[@TMA_ISPASS]*](../../img/pmu-features/TMAM_diag.png){#fig:TMA_concept width=80%}

This is not how the analysis works in practice because analyzing every single microoperation ($\mu$op) would be terribly slow. Instead, TMA observes the execution of a program by monitoring a specific set of performance events and then calculates metrics based on predefined formulas. Using these metrics, TMA characterizes the program by assigning it to one of the four high-level buckets. Each of the four high-level categories has several nested levels, which CPU vendors may choose to implement differently. Each generation of processors may have different formulas for calculating those metrics, so it's better to rely on tools to do the analysis rather than trying to calculate them yourself.

这不是分析在实践中的工作方式，因为分析每个微操作 ($\mu$op) 会非常慢。相反，TMA 通过监视一组特定的性能事件来观察程序的执行情况，然后根据预定义的公式计算指标。使用这些指标，TMA通过将程序分配到4个高级存储桶Buckets之一来表征程序。四个高级类别中的每一个都有多个嵌套级别，CPU供应商可能会选择不同的实现方式。每一代处理器可能有不同的公式来计算这些指标，因此最好依靠工具来进行分析，而不是尝试自己计算它们。

In the upcoming sections, we will discuss the TMA implementation in AMD, Arm, and Intel processors.

在接下来的部分中，我们将讨论AMD、Arm和Intel处理器中的TMA实现。