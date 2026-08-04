### TMA On Arm Platforms Arm平台上的TMA

Arm CPU architects also have developed a TMA performance analysis methodology for their processors, which we will discuss next. Arm calls it "Topdown" in their documentation [@ARMNeoverseV1TopDown], so we will use their naming. At the time of this writing (late 2023), Topdown is only supported on cores designed by Arm, e.g. Neoverse N1 and Neoverse V1,[^5] and their derivatives, e.g. Ampere Altra and AWS Graviton3. Refer to the list of major CPU microarchitectures at the end of this book if you need to refresh your memory on Arm chip families. Processors designed by Apple don't support the Arm Topdown performance analysis methodology yet.

Arm公司CPU体系结构设计师还为其处理器开发了一种TMA性能分析方法，我们将在下文中讨论。Arm在其文档中将其称为“自顶向下Topdown）”（[@ARMNeoverseV1TopDown]，因此我们将沿用他们的命名。截至本文撰写之时（2023年末），自顶向下方法仅支持Arm公司设计的核心，例如Neoverse N1和Neoverse V1[^5]及其衍生产品，例如Ampere Altra和AWS Graviton3。如果您需要回顾Arm芯片系列，请参阅本书末尾的主要CPU微架构列表。Apple设计的处理器目前尚不支持Arm自顶向下性能分析方法。

Arm Neoverse V1 is the first CPU in the Neoverse family that supports a full set of level 1 Topdown metrics: `错误前瞻Bad Speculation`, `前端瓶颈Frontend Bound`, `后端瓶颈Backend Bound`, and `退役Retiring`. Prior to V1 cores, Neoverse N1 supported only two L1 categories: `Frontend Stalled Cycles` and `Backend Stalled Cycles`.[^6]

Arm公司Neoverse V1是Neoverse系列中首款支持全套L1级自顶向下指标的CPU，这些指标包括：`Bad Speculation`、`Frontend Bound`、`Backend Bound`和`Retiring`。在V1核心之前，Neoverse N1仅支持两种L1级类别：前端停滞周期 (Frontend Stalled Cycles) 和后端停滞周期 (Backend Stalled Cycles)[^6]。

To demonstrate Arm's Topdown analysis on a V1-based processor, I launched an AWS EC2 `m7g.metal` instance powered by the AWS Graviton3. Note that Topdown might not work on other non-`metal` instance types due to virtualization. I used 64-bit ARM `Ubuntu 22.04 LTS` with `Linux kernel 6.2` managed by AWS. The provided `m7g.metal` instance had 64 vCPUs and 256 GB of RAM.

为了演示Arm的自顶向下分析方法在基于V1处理器上的应用，我启动了一个由AWS Graviton3提供运行支持的AWS EC2 `m7g.metal` 实例。请注意，由于虚拟化技术的限制，自顶向下分析方法可能无法在其他非 `metal` 实例类型上运行。我使用的是由AWS管理的64位ARM操作系统 `Ubuntu 22.04 LTS`，内核版本为 `Linux kernel 6.2`。AWS所提供的 `m7g.metal` 实例拥有64个vCPU核心与256GB内存。

I applied the Topdown methodology to [AI Benchmark Alpha](https://ai-benchmark.com/alpha.html),[^1] which is an open-source Python library for evaluating AI performance of various hardware platforms, including CPUs, GPUs, and TPUs. The benchmark relies on the TensorFlow machine learning library to measure inference and training speed for key Deep Learning models. In total, AI Benchmark Alpha consists of 42 tests, including classification, image segmentation, text translation, and others.

我将自顶向下分析方法应用于[AI Benchmark Alpha](https://ai-benchmark.com/alpha.html)[^1]，这是一个用于评估各种硬件平台（包括：CPU、GPU和TPU）上AI性能的开源Python库。该基准测试依赖于TensorFlow机器学习库来衡量关键深度学习模型的推理和训练速度。AI Benchmark Alpha总共包含42项测试，涵盖：分类、图像分割、文本翻译等。

Arm engineers have developed the [topdown-tool](https://learn.arm.com/install-guides/topdown-tool/)[^2] that we will use below. The tool works both on Linux and Windows on ARM. On Linux, it utilizes a standard perf tool, while on Windows it uses [WindowsPerf](https://gitlab.com/Linaro/WindowsPerf/windowsperf)[^3], which is a Windows on ARM performance profiling tool. Similar to Intel's TMA, the Arm methodology employs the "drill-down" concept, i.e. you first determine the high-level performance bottleneck and then drill down for a more nuanced root cause analysis. Here is the command we used:

Arm工程师开发了我们将在下文中使用的 [topdown-tool](https://learn.arm.com/install-guides/topdown-tool/)[^2]。该工具可在Linux和Windows on ARM上运行。在Linux上，它使用标准的perf工具；而在Windows上，它使用 [WindowsPerf](https://gitlab.com/Linaro/WindowsPerf/windowsperf)[^3]，这是一个Windows on ARM性能分析工具。与Intel的TMA类似，Arm的方法论采用了“向下钻取（drill-down）”的概念，即首先确定高层次的性能瓶颈，然后深入分析更细致的根本原因。以下是我们使用的命令：

```bash
$ topdown-tool --all-cpus -m Topdown_L1 -- python -c "from ai_benchmark import AIBenchmark; results = AIBenchmark(use_CPU=True).run()"
Stage 1 (Topdown metrics)
=========================
[Topdown Level 1]
Frontend Bound... 16.48% slots
Backend Bound.... 54.92% slots
Retiring......... 27.99% slots
Bad Speculation..  0.59% slots
```

where the `--all-cpus` option enables system-wide collection for all CPUs, and `-m Topdown_L1` collects Topdown Level 1 metrics. Everything that follows `--` is the command line to run the AI Benchmark Alpha suite.

其中，`--all-cpus` 选项启用系统级所有CPU的数据收集，`-m Topdown_L1` 选项收集Topdown第一级L1指标。`--` 后面的所有内容都是运行AI Benchmark Alpha测试套件的命令行参数。

From the output above, we conclude that the benchmark doesn't suffer from branch mispredictions. Also, without a deeper understanding of the workloads involved, it's hard to say if the `Frontend Bound` of 16.5% is worth looking at, so we turn our attention to the `Backend Bound` metric, which is the main source of stalled cycles. Neoverse V1-based chips don't have the second-level breakdown, instead, the methodology suggests further exploring the problematic category by collecting a set of corresponding metrics. Here is how we drill down into the more detailed `Backend Bound` analysis:

从上面的输出可以看出，该基准测试不存在分支预测错误的问题。此外，由于缺乏对相关工作负载的深入了解，很难判断16.5%的 `前端瓶颈` 是否值得关注，因此我们将注意力转向 `后端瓶颈` 指标，它是导致周期停滞的主要原因。基于Neoverse V1的芯片没有第二级L2细分，因此，该方法建议通过收集一组相应的指标来进一步探究这一问题类别。以下是我们如何深入分析更详细的`后端瓶颈`：

```bash
$ topdown-tool --all-cpus -n BackendBound -- python -c "from ai_benchmark import AIBenchmark; results = AIBenchmark(use_CPU=True).run()"
Stage 1 (Topdown metrics)
=========================
[Topdown Level 1]
Backend Bound......................... 54.70% slots

Stage 2 (uarch metrics)
=======================
  [Data TLB Effectiveness]
  DTLB MPKI........................... 0.413 misses per 1,000 instructions
  L1 Data TLB MPKI.................... 3.779 misses per 1,000 instructions
  L2 Unified TLB MPKI................. 0.407 misses per 1,000 instructions
  DTLB Walk Ratio..................... 0.001 per TLB access
  L1 Data TLB Miss Ratio.............. 0.013 per TLB access
  L2 Unified TLB Miss Ratio........... 0.112 per TLB access

  [L1 Data Cache Effectiveness]
  L1D Cache MPKI...................... 13.114 misses per 1,000 instructions
  L1D Cache Miss Ratio................  0.046 per cache access

  [L2 Unified Cache Effectiveness]
  L2 Cache MPKI....................... 1.458 misses per 1,000 instructions
  L2 Cache Miss Ratio................. 0.027 per cache access

  [Last Level Cache Effectiveness]
  LL Cache Read MPKI.................. 2.505 misses per 1,000 instructions
  LL Cache Read Miss Ratio............ 0.219 per cache access
  LL Cache Read Hit Ratio............. 0.783 per cache access

  [Speculative Operation Mix]
  Load Operations Percentage.......... 25.36% operations
  Store Operations Percentage.........  2.54% operations
  Integer Operations Percentage....... 29.60% operations
  Advanced SIMD Operations Percentage. 10.93% operations
  Floating Point Operations Percentage  6.85% operations
  Branch Operations Percentage........ 10.04% operations
  Crypto Operations Percentage........  0.00% operations
```

In the command above, the option `-n BackendBound` collects all the metrics associated with the `Backend Bound` category as well as its descendants. The description for every metric in the output is given in [@ARMNeoverseV1TopDown]. Note, that they are quite similar to what we have discussed in [@sec:PerfMetrics], so you may want to revisit it as well. 

在上述命令中，选项 `-n BackendBound` 会收集所有与 `Backend Bound` 类别及其子类别关联的指标。输出中每个指标的描述请参见 [@ARMNeoverseV1TopDown]。请注意，这些指标与我们在 [@sec:PerfMetrics] 中讨论的内容非常相似，因此您可能也需要重新查看 [@sec:PerfMetrics]。

We don't have a goal of optimizing the benchmark, rather we want to characterize performance bottlenecks. However, if given such a task, here is how our analysis could continue. There are a substantial number of `L1 Data TLB` misses (3.8 MPKI), but then 90% of those misses hit in L2 TLB (see `L2 Unified TLB Miss Ratio`). All in all, only 0.1% of all TLB accesses result in a page table walk (see `DTLB Walk Ratio`), which suggests that it is not our primary concern, although a quick experiment that utilizes huge pages is still worthwhile.

我们的目标并非优化基准测试，而是找出性能瓶颈。但是，如果确实需要进行此类优化，我们可以继续进行以下分析。存在大量的 `L1 Data TLB` 未命中（3.8MPKI），但其中90%的未命中发生 L2 TLB中（参见 `L2 Unified TLB Miss Ratio`）。总而言之，只有0.1%的TLB访问会导致页表遍历（参见 `DTLB Walk Ratio` ），这表明页表遍历并非我们主要关注的问题，尽管使用大页进行快速实验仍然值得。

Looking at the `L1/L2/LL Cache Effectiveness` metrics, we can spot a potential problem with data cache misses. One in ~22 accesses to the L1D cache results in a miss (see `L1D Cache Miss Ratio`), which is tolerable but still expensive. For L2, this number is one in 37 (see `L2 Cache Miss Ratio`), which is much better. However for LLC, the `LL Cache Read Miss Ratio` is unsatisfying: every 5th access results in a miss. Since this is an AI benchmark, where the bulk of the time is likely spent in matrix multiplication, code transformations like loop blocking may help (see [@sec:LoopOpts]). AI algorithms are known for being "memory hungry", however, Neoverse V1 Topdown doesn't show if there are stalls that can be attributed to saturated memory bandwidth.

查看 `L1/L2/LL缓存有效性 L1/L2/LL Cache Effectiveness缓存有效性` 指标，我们可以发现数据缓存未命中存在潜在问题。大约每 22次L1D缓存访问中就有1次会导致未命中（参见 `L1D缓存未命中率 L1D Cache Miss Ratio` ），虽然可以接受，但仍然代价高昂。L2缓存的未命中率为1/37（参见 `L2缓存未命中率 L2 Cache Miss Ratio` ），情况要好得多。然而，LLC缓存的`LL缓存读取未命中率 LL Cache Read Miss Ratio` 却不尽如人意：每5次访问中就有1次会导致未命中。由于这是一个AI基准测试，大部分时间可能都消耗在矩阵乘法上，因此循环阻塞（loop blocking）等代码优化或许有所帮助（参见 [@sec:LoopOpts]）。众所周知，人工智能算法“内存消耗巨大（memory hungry）”，然而，Neoverse V1自顶向下分析工具并未显示是否存在因内存带宽饱和而导致的停顿。

The final category provides the operation mix, which can be useful in some scenarios. The percentages give us an estimate of how many instructions of a certain type were executed, including speculatively executed instructions. The numbers don't sum up to 100%, because the rest is attributed to the implicit "Others" category (not printed by the `topdown-tool`), which is about 15% in our case. We should be concerned by the low percentage of SIMD operations, especially given that the highly optimized `Tensorflow` and `numpy` libraries are used. In contrast, the percentage of integer operations and branches seems high. I checked that the majority of executed branch instructions are loop backward jumps to the next loop iteration. The high percentage of integer operations could be caused by lack of vectorization, or due to thread synchronization. [@ARMNeoverseV1TopDown] gives an example of discovering a vectorization opportunity using data from the `Speculative Operation Mix` category. 

最后一类提供了操作组合信息，这在某些情况下很有用。百分比数据可以估算出特定类型指令的执行数量，包括前瞻执行的指令。这些数字加起来并不等于100%，因为剩余部分归属于隐式的“其他”类别（`topdown-tool` 工具不会打印出来），在本例中约为15%。我们应该关注SIMD操作的低百分比，尤其是在使用了高度优化的 `Tensorflow` 和 `numpy` 库的情况下。相比之下，整数运算和分支的百分比似乎很高。我检查发现，大多数执行的分支指令都是跳转到下一次循环迭代的循环后退（loop backward）操作。整数运算的高百分比可能是由于缺乏向量化或线程同步造成的。 [@ARMNeoverseV1TopDown] 提供了一个使用 `前瞻性操作混合Speculative Operation Mix` 类别数据发现向量化机会的示例。

In our case study, we ran the benchmark two times, however, in practice, one run is usually sufficient. Running the `topdown-tool` without options will collect all the available metrics using a single run. Also, the `-s combined` option will group the metrics by the L1 category, and output data in a format similar to `Intel VTune`, `toplev`, and other tools. The only practical reason for making multiple runs is when a workload has a bursty behavior with very short phases that have different performance characteristics. In such a scenario, you would like to avoid event multiplexing (see [@sec:secMultiplex]) and improve collection accuracy by running the workload multiple times. 

在我们的案例研究中，我们运行了两次基准测试，但实际上，一次运行通常就足够了。运行不带任何选项的 `topdown-tool` 工具，只需一次运行即可收集所有可用指标。此外，`-s combined` 选项会将指标按L1类别分组，并以类似于 `Intel VTune`、`toplev`和其他工具的格式输出数据。进行多次运行的唯一实际原因是工作负载具有突发性行为，包含性能特征不同的极短阶段。在这种情况下，您需要避免事件复用（参见 [@sec:secMultiplex]），并通过多次运行工作负载来提高数据收集的准确性。

The AI Benchmark Alpha has various tests that could exhibit different performance characteristics. The output presented above aggregates all 42 tests and gives an overall breakdown. This is generally not a good idea if the individual tests indeed have different performance bottlenecks. You need to have a separate Topdown analysis for each of the tests. One way the `topdown-tool` can help is to use the `-i` option which will output data per configurable interval of time. You can then compare the intervals and decide on the next steps.

AI基准测试Alpha包含各种可能展现不同性能特征的测试。上面显示的输出汇总了所有42项测试，并给出了总体分析。如果各个测试确实存在不同的性能瓶颈，那么这种做法通常并不明智。你需要对每个测试分别进行自顶向下分析。`topdown-tool` 的一个帮助方法是使用 `-i` 选项，该选项会按可配置的时间间隔输出数据。然后你可以比较这些时间间隔的数据，并决定下一步的步骤。

[^1]: AI Benchmark Alpha - [https://ai-benchmark.com/alpha.html](https://ai-benchmark.com/alpha.html)
[^2]: Arm `topdown-tool` - [https://learn.arm.com/install-guides/topdown-tool/](https://learn.arm.com/install-guides/topdown-tool/)
[^3]: WindowsPerf - [https://gitlab.com/Linaro/WindowsPerf/windowsperf](https://gitlab.com/Linaro/WindowsPerf/windowsperf)
[^5]: In September 2024, Arm published performance analysis for Neoverse V2 and V3 cores - [https://developer.arm.com/telemetry](https://developer.arm.com/telemetry) 2024年9月，Arm发布了Neoverse V2和V3内核的性能分析 - [https://developer.arm.com/telemetry](https://developer.arm.com/telemetry)
[^6]: Performance analysis guides for Neoverse V2 and V3 cores became available after I wrote this section. See the following page - [https://developer.arm.com/telemetry](https://developer.arm.com/telemetry). 在我撰写本节内容之后，Neoverse V2和V3内核的性能分析指南才发布。请参阅以下页面 - [https://developer.arm.com/telemetry](https://developer.arm.com/telemetry)。
