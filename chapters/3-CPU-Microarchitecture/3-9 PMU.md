

## Performance Monitoring Unit 性能监控单元 {#sec:PMU}

Every modern CPU provides facilities to monitor performance, which are combined into the Performance Monitoring Unit (PMU). This unit incorporates features that help developers analyze the performance of their applications. An example of a PMU in a modern Intel CPU is provided in Figure @fig:PMU. Most modern PMUs have a set of Performance Monitoring Counters (PMC) that can be used to collect various performance events that happen during the execution of a program. Later in [@sec:counting], we will discuss how PMCs can be used for performance analysis. Also, the PMU has other features that enhance performance analysis, like LBR, PEBS, and PT, topics to which [@sec:PmuChapter] is devoted.

每个现代CPU都提供性能监控功能，这些功能被集成到性能监控单元(PMU: Performance Monitoring Unit)中。该单元包含多种功能，可帮助开发人员分析应用程序的性能。图 @fig:PMU 展示了现代Intel CPU中的PMU示例。大多数现代PMU都包含一组性能监控计数器(PMC: Performance Monitoring Counter)，可用于收集程序执行期间发生的各种性能事件。稍后在 [@sec:counting] 中，我们将讨论如何使用PMC进行性能分析。此外，PMU还具有其他增强性能分析的功能，例如：LBR、PEBS和PT，这些内容将在 [@sec:PmuChapter] 中详细介绍。

![Performance Monitoring Unit of a modern Intel CPU. 在一颗现代Intel处理器中的性能监控单元。](../../img/uarch/PMU.png){#fig:PMU width=100%}

As CPU design evolves with every new generation, so do their PMUs. On Linux, it is possible to determine the version of the PMU in your CPU using the `cpuid` command, as shown in [@lst:QueryPMU]. Similar information can be extracted from the kernel message buffer by checking the output of the `dmesg` command. Characteristics of each Intel PMU version, as well as changes from the previous version, can be found in [@IntelOptimizationManual, Volume 3B, Chapter 20].

随着CPU设计在每一代产品中不断发展，其PMU也随之演变。在Linux系统中，可以使用 `cpuid` 命令确定 CPU中PMU的版本，如 [@lst:QueryPMU] 所示。也可以通过检查 `dmesg` 命令的输出，从内核消息缓冲区中提取类似信息。每个Intel PMU版本的特性以及与先前版本相比的变化，可以在Intel官方软件优化手册卷3B第20章 [@IntelOptimizationManual, Volume 3B, Chapter 20] 中找到。

Listing: Querying your PMU
代码列表：查询你的PMU

~~~~ {#lst:QueryPMU .bash}
$ cpuid
...
Architecture Performance Monitoring Features (0xa/eax):
      version ID                               = 0x4 (4)
      number of counters per logical processor = 0x4 (4)
      bit width of counter                     = 0x30 (48)
...
Architecture Performance Monitoring Features (0xa/edx):
      number of fixed counters    = 0x3 (3)
      bit width of fixed counters = 0x30 (48)
...
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

### Performance Monitoring Counters 性能监控计数器 {#sec:PMC}

If we imagine a simplified view of a processor, it may look something like what is shown in Figure @fig:PMC. As we discussed earlier in this chapter, a modern CPU has caches, a branch predictor, an execution pipeline, and other units. When connected to multiple units, a PMC can collect interesting statistics from them. For example, it can count how many clock cycles have passed, how many instructions were executed, how many cache misses or branch mispredictions happened during that time, and other performance events.

如果我们设想一个简化的处理器视图，它可能类似于图 @fig:PMC 所示。正如我们在本章前面讨论过的，现代CPU包含缓存、分支预测器、执行流水线和其他单元。当连接到多个单元时，PMC可以从中收集有用的统计信息。例如，它可以统计经过了多少个时钟周期、执行了多少条指令、在此期间发生了多少次缓存未命中或分支预测错误，以及其他性能事件。

![Simplified view of a CPU with a performance monitoring counter. 带有性能监控计数器的CPU简化视图。](../../img/uarch/PMC.png){#fig:PMC width=60%}

Typically, PMCs are 48-bit wide, which enables analysis tools to run for a long time without interrupting a program's execution.[^2] Performance counter is a hardware register implemented as a Model-Specific Register (MSR). That means the number of counters and their width can vary from model to model, and you cannot rely on the same number of counters in your CPU. You should always query that first, using tools like `cpuid`, for example. PMCs are accessible via the `RDMSR` and `WRMSR` instructions, which can only be executed from kernel space. Luckily, you only have to care about this if you're a developer of a performance analysis tool, like Linux `perf` or Intel VTune profiler. Those tools handle all the complexity of programming PMCs.

通常，PMC的宽度为48位，这使得分析工具可以长时间运行而不会中断程序的执行[^2]。性能计数器是一个硬件寄存器，作为型号特定寄存器(MSR: Model-Specific Register)的形式实现。这意味着计数器的数量和宽度会因CPU型号而异，你不能依赖CPU中始终存在相同数量的计数器。您应该始终首先查询计数器数量，例如使用 `cpuid` 等工具。PMC可通过 `RDMSR` 和 `WRMSR` 指令访问，这些指令只能在内核空间执行。幸运的是，只有当你是性能分析工具（例如：Linux `perf` 或 Intel VTune 分析器）的开发者时才需要关注这一点。这些工具会处理PMC编程的所有复杂性。

When engineers analyze their applications, it is very common for them to collect the number of executed instructions and elapsed cycles. That is the reason why some PMUs have dedicated PMCs for collecting such events. Fixed counters always measure the same thing inside the CPU core. With programmable counters, it's up to the user to choose what they want to measure. 

工程师在分析应用程序时，通常会收集已执行指令的数量和经过的周期数。这就是为什么一些PMU具有专用的PMC来收集此类事件的原因。固定计数器始终测量CPU核心内部的相同内容。而对于可编程计数器，用户可以自行选择要测量的内容。

For example, in the Intel Skylake architecture (PMU version 4, see [@lst:QueryPMU]), each physical core has three fixed and eight programmable counters. The three fixed counters are set to count core clocks, reference clocks, and instructions retired (see [@sec:secMetrics] for more details on these metrics). AMD Zen4 and Arm Neoverse V1 cores support 6 programmable performance monitoring counters per processor core, with no fixed counters.

例如，在英特尔 Skylake 架构（PMU 版本 4，参见 [@lst:QueryPMU]）中，每个物理核心都有三个固定计数器和八个可编程计数器。这三个固定计数器分别用于统计核心时钟频率、参考时钟频率和指令执行次数（有关这些指标的更多详细信息，请参见 [@sec:secMetrics]）。AMD Zen4 和 Arm Neoverse V1 内核每个处理器核心支持 6 个可编程性能监控计数器，但没有固定计数器。

It's not unusual for the PMU to provide more than one hundred events available for monitoring. Figure @fig:PMU shows just a small subset of the performance monitoring events available for monitoring on a modern Intel CPU. It's not hard to notice that the number of available PMCs is much smaller than the number of performance events. It's not possible to count all the events at the same time, but analysis tools solve this problem by multiplexing between groups of performance events during the execution of a program (see [@sec:secMultiplex]).

PMU 提供超过一百个可供监控的事件并不罕见。图 @fig:PMU 仅显示了现代英特尔 CPU 上可供监控的性能监控事件的一小部分。不难看出，可用的 PMC 数量远小于性能事件的数量。虽然无法同时统计所有事件，但分析工具通过在程序执行期间对性能事件组进行多路复用来解决这个问题（参见 [@sec:secMultiplex]）。

- For Intel CPUs, the complete list of performance events can be found in [@IntelOptimizationManual, Volume 3B, Chapter 20] or at [perfmon-events.intel.com](https://perfmon-events.intel.com/). 
- 对于Intel CPU，完整的性能事件列表可在 [@IntelOptimizationManual, Volume 3B, Chapter 20] 或 [perfmon-events.intel.com](https://perfmon-events.intel.com/) 上找到。
- AMD doesn't publish a list of performance monitoring events for every AMD processor. Curious readers may find some information in the Linux `perf` source [code](https://github.com/torvalds/linux/blob/master/arch/x86/events/amd/core.c)[^3]. Also, you can list performance events available for monitoring using the AMD uProf command line tool. General information about AMD performance counters can be found in [@AMDProgrammingManual, 13.2 Performance Monitoring Counters].
- AMD并未公布所有AMD处理器的性能监控事件列表。感兴趣的读者可以在Linux `perf`源代码[code](https://github.com/torvalds/linux/blob/master/arch/x86/events/amd/core.c)[^3]中找到一些信息。此外，您还可以使用AMD uProf命令行工具列出可监控的性能事件。有关AMD性能计数器的一般信息，请参阅AMD编程手册 [@AMDProgrammingManual, 13.2性能监控计数器]。
- For ARM chips, performance events are not so well defined. Vendors implement cores following an ARM architecture, but performance events vary, both in what they mean and what events are supported. For the Arm Neoverse V1 core, that Arm designs themselves, the list of performance events can be found in [@ARMNeoverseV1]. For the Arm Neoverse V2 and V3 microarchitectures, the list of performance events can be found on Arm's website.[^4]
- 对于ARM芯片，性能事件的定义并不十分明确。虽然厂商会根据ARM体系结构实现核心，但性能事件的含义和支持的事件种类却各不相同。对于Arm自行设计的Arm Neoverse V1核心，性能事件列表可在 [@ARMNeoverseV1] 中找到。对于Arm Neoverse V2和V3微体系结构，性能事件列表可在Arm网站上找到。[^4]

[^2]: When the value of PMCs overflows, the execution of a program must be interrupted. A profiling tool then should save the fact of an overflow. We will discuss it in more detail in [@sec:sec_PerfApproaches]. 当PMC的值溢出时，程序的执行必须中断。此时，性能分析工具应记录溢出事件。我们将在 [@sec:sec_PerfApproaches] 中详细讨论这一点。
[^3]: Linux source code for AMD cores 适用于AMD内核的Linux源代 - [https://github.com/torvalds/linux/blob/master/arch/x86/events/amd/core.c](https://github.com/torvalds/linux/blob/master/arch/x86/events/amd/core.c)
[^4]: Arm telemetry Arm的遥测 - [https://developer.arm.com/telemetry](https://developer.arm.com/telemetry)
