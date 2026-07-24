## Pipeline Slot 流水线槽 {#sec:PipelineSlot}

Another important metric that some performance tools use is the concept of a *pipeline slot*. A pipeline slot represents the hardware resources needed to process one $\mu$op. Figure @fig:PipelineSlot demonstrates the execution pipeline of a CPU that has 4 allocation slots every cycle. That means that the core can assign execution resources (renamed source and destination registers, execution port, ROB entries, etc.) to 4 new $\mu$ops every cycle. Such a processor is usually called a *4-wide machine*. During six consecutive cycles on the diagram, only half of the available slots were utilized (highlighted in yellow). From a microarchitecture perspective, the efficiency of executing such code is only 50%.

一些性能工具使用的另一个重要指标是“流水线槽”的概念。流水线槽代表处理一个μ操作所需的硬件资源。图 @fig:PipelineSlot 展示了一个CPU的执行流水线，该CPU每个周期有4个分配槽。这意味着核心每个周期可以为 4个新的μ操作分配执行资源（重命名的源寄存器和目标寄存器、执行端口、ROB条目等）。这样的处理器通常被称为“4运行宽度机器”。在图中连续6个周期内，只有一半的可用槽被使用（黄色高亮部分）。从微体系结构的角度来看，执行此类代码的效率只有50%。

![Pipeline diagram of a 4-wide CPU. 4运行宽度CPU流水线图](../../img/terms-and-metrics/PipelineSlot.jpg){#fig:PipelineSlot width=40% }

Intel's Skylake and AMD Zen3 cores have a 4-wide allocation. Intel's Sunny Cove microarchitecture was a 5-wide design. As of the end of 2023, the most recent Golden Cove and Zen4 architectures both have a 6-wide allocation. Apple M1 and M2 designs are 8-wide, and Apple M3 is 9-$\mu$op execution bandwidth see [@AppleOptimizationGuide, Table 4.10]. The width of a machine puts a cap on the IPC. This means that the maximum achievable IPC of a processor equals its width.[^2] For example, when your calculations show more than 6 IPC on a Golden Cove core, you should be suspicious.

Inte的Skylake核心和AMD的Zen3核心采用4宽分配。Intel的Sunny Cove微体系结构采用5宽设计。截至2023年底，最新的Golden Cove和Zen4体系结构均采用6宽分配。苹果M1和M2设计采用8宽，而苹果M3的执行带宽为9个微操作μop（参见 [@AppleOptimizationGuide，表 4.10]）。机器的宽度限制了每时钟周期指令数（IPC）。这意味着处理器可达到的最大IPC等于其宽度[^2]。例如，如果你计算出的Golden Cove核心的IPC超过6，则应引起警惕。

Very few applications can achieve the maximum IPC of a machine. For example, Intel Golden Cove core can theoretically execute four integer additions/subtractions, plus one load, plus one store (for a total of six instructions) per clock, but an application is highly unlikely to have the appropriate mix of independent instructions adjacent to each other to exploit all that potential parallelism.

极少有应用程序能够达到机器的最大IPC（每时钟周期指令数）。例如，Intel的Golden Cove核心理论上每个时钟周期可以执行4次整数加减运算、1次加载运算和1次存储运算（共6条指令），但应用程序极不可能拥有足够多的相邻独立指令组合来充分利用这种潜在的并行性。

Pipeline slot utilization is one of the core metrics in Top-down Microarchitecture Analysis (see [@sec:TMA]). For example, Frontend Bound and Backend Bound metrics are expressed as a percentage of unutilized pipeline slots due to various bottlenecks.

流水线槽利用率是自顶向下微架构分析（参见 [@sec:TMA]）的核心指标之一。例如，前端瓶颈和后端瓶颈指标就是由于各种瓶颈导致未利用的流水线槽百分比。

[^2]: Although there are some exceptions. For instance, macrofused compare-and-branch instructions only require a single pipeline slot but are counted as two instructions. In some extreme cases, this may cause IPC to be greater than the machine width. 尽管也有一些例外。例如，宏融合的比较分支指令只需要一个流水线槽，但却被计为两条指令。在某些极端情况下，这可能会导致IPC大于机器宽度。
