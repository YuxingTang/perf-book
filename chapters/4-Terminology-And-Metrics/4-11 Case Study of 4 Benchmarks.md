## Case Study: Analyzing Performance Metrics of Four Benchmarks 案例研究：分析四个基准测试的性能指标 {#sec:PerfMetricsCaseStudy}

To put together everything we discussed so far in this chapter, let's look at some real-world examples. We ran four benchmarks from different domains and calculated their performance metrics. First of all, let's introduce the benchmarks.

为了总结本章目前为止讨论的所有内容，让我们来看一些实际案例。我们运行了来自不同领域的四个基准测试，并计算了它们的性能指标。首先，让我们介绍一下这些基准测试。

1. Blender 3.4 - an open-source 3D creation and modeling software project. This test is of Blender's Cycles performance with the BMW27 blend file. All hardware threads are used. URL: [https://download.blender.org/release](https://download.blender.org/release). Command line: `./blender -b bmw27_cpu.blend -noaudio --enable-autoexec -o output.test -x 1 -F JPEG -f 1`.
2. Stockfish 15 - an advanced open-source chess engine. This test is a stockfish built-in benchmark. A single hardware thread is used. URL: [https://stockfishchess.org](https://stockfishchess.org). Command line: `./stockfish bench 128 1 24 default depth`.
3. Clang 15 self-build - this test uses Clang 15 to build the Clang 15 compiler from sources. All hardware threads are used. URL: [https://www.llvm.org](https://www.llvm.org). Command line: `ninja -j16 clang`.
4. CloverLeaf 2018 - a Lagrangian-Eulerian hydrodynamics benchmark. All hardware threads are used. This test uses the clover_bm.in input file (Problem 5). URL: [http://uk-mac.github.io/CloverLeaf](http://uk-mac.github.io/CloverLeaf). Command line: `./clover_leaf`.

1. Blender 3.4 - 一个开源的3D创建和建模软件项目。此测试使用Blender的Cycles渲染器运行BMW27 blend文件，并测试其性能。所有硬件线程均被使用。URL：[https://download.blender.org/release](https://download.blender.org/release)。命令行：`./blender -b bmw27_cpu.blend -noaudio --enable-autoexec -o output.test -x 1 -F JPEG -f 1`。
2. Stockfish 15 - 一个高级的开源国际象棋引擎。此测试是Stockfish内置的基准测试。使用单个硬件线程。网址：[https://stockfishchess.org](https://stockfishchess.org)。命令行：`./stockfish bench 128 1 24 default depth`。
3. Clang 15 自编译 - 此测试使用Clang 15从源代码构建Clang 15编译器。使用所有硬件线程。网址：[https://www.llvm.org](https://www.llvm.org)。命令行：`ninja -j16 clang`。
4. CloverLeaf 2018 - 拉格朗日-欧拉流体动力学基准测试。使用所有硬件线程。此测试使用clover_bm.in 输入文件（问题5）。网址：[http://uk-mac.github.io/CloverLeaf](http://uk-mac.github.io/CloverLeaf)。命令行：`./clover_leaf`。


For this exercise, I ran all four benchmarks on the machine with the following characteristics:

为了进行这项测试，我在一台配置如下的机器上运行了全部四项基准测试：

* 12th Gen Alder Lake Intel&reg; Core&trade; i7-1260P CPU @ 2.10GHz (4.70GHz Turbo), 4P+8E cores, 18MB L3-cache
* 16 GB RAM, DDR4 @ 2400 MT/s
* 256GB NVMe PCIe M.2 SSD
* 64-bit Ubuntu 22.04.1 LTS (Jammy Jellyfish)
* Clang-15 C++ compiler with the following options: `-O3 -march=core-avx2`

* 第十二代Alder Lake Intel® Core™ 处理器i7-1260P CPU@2.10GHz（睿频4.70GHz），4个性能核心 + 8个效率核心，18MB的L3缓存
* 16GB内存，DDR4@2400MT/s
* 256GB NVMe PCIe M.2固态硬盘
* 64位 Ubuntu 22.04.1 LTS (Jammy Jellyfish)
* Clang-15 C++ 编译器，使用以下选项：`-O3 -march=core-avx2`

To collect performance metrics, I used the `toplev.py` script from Andi Kleen's [pmu-tools](https://github.com/andikleen/pmu-tools):[^1]

为了收集性能指标，我使用了Andi Kleen的[pmu-tools](https://github.com/andikleen/pmu-tools):[^1]中的 `toplev.py` 脚本：

```bash
$ ~/workspace/pmu-tools/toplev.py -m --global --no-desc -v -- <app with args>
```

Table {@tbl:perf_metrics_case_study} provides a side-by-side comparison of performance metrics for our four benchmarks. There is a lot we can learn about the nature of those workloads just by looking at the metrics. 

表格{@tbl:perf_metrics_case_study} 提供了我们的4个基准测试的性能指标并排比较。仅通过查看这些指标，我们就能了解这些工作负载的本质。

\small

--------------------------------------------------------------------------
Metric           Core        Blender     Stockfish   Clang15-   CloverLeaf
Name             Type                                selfbuild
---------------- ----------- ----------- ----------- ---------- ----------
Instructions     P-core      6.02E+12    6.59E+11    2.40E+13   1.06E+12

Core Cycles      P-core      4.31E+12    3.65E+11    3.78E+13   5.25E+12

IPC              P-core      1.40        1.80        0.64       0.20

CPI              P-core      0.72        0.55        1.57       4.96

Instructions     E-core      4.97E+12    0           1.43E+13   1.11E+12

Core Cycles      E-core      3.73E+12    0           3.19E+13   4.28E+12

IPC              E-core      1.33        0           0.45       0.26

CPI              E-core      0.75        0           2.23       3.85

L1MPKI           P-core      3.88        21.38       6.01       13.44

L2MPKI           P-core      0.15        1.67        1.09       3.58

L3MPKI           P-core      0.04        0.14        0.56       3.43

Br. Misp. Ratio  P-core      0.02        0.08        0.03       0.01

Code stlb MPKI   P-core      0           0.01        0.35       0.01

Ld stlb MPKI     P-core      0.08        0.04        0.51       0.03

St stlb MPKI     P-core      0           0.01        0.06       0.1

LdMissLat (Clk)  P-core      12.92       10.37       76.7       253.89

ILP              P-core      3.67        3.65        2.93       2.53

MLP              P-core      1.61        2.62        1.57       2.78

Dram Bw (GB/s)   All         1.58        1.42        10.67      24.57

IpCall           All         176.8       153.5       40.9       2,729

IpBranch         All         9.8         10.1        5.1        18.8

IpLoad           All         3.2         3.3         3.6        2.7

IpStore          All         7.2         7.7         5.9        22.0

IpMispredict     All         610.4       214.7       177.7      2,416

IpFLOP           All         1.1         1.82E+06    286,348    1.8

IpArith          All         4.5         7.96E+06    268,637    2.1

IpArith Scal SP  All         22.9        4.07E+09    280,583    2.60E+09

IpArith Scal DP  All         438.2       1.22E+07    4.65E+06   2.2

IpArith AVX128   All         6.9         0.0         1.09E+10   1.62E+09

IpArith AVX256   All         30.3        0.0         0.0        39.6

IpSWPF           All         90.2        2,565       105,933    172,348
--------------------------------------------------------------------------

Table: Performance Metrics of Four Benchmarks. 4个基准测试的性能指标 {#tbl:perf_metrics_case_study}

\normalsize

Here are the hypotheses we can make about the performance of the benchmarks:

以下是我们对基准测试性能的假设：

* __Blender__. The work is split fairly equally between P-cores and E-cores, with a decent IPC on both core types. The number of cache misses per kilo instructions is pretty low (see `L*MPKI`). Branch misprediction presents a minor bottleneck: the `Br. Misp. Ratio` metric is at `2%`; we get 1 misprediction for every `610` instructions (see `IpMispredict` metric), which is quite good. TLB is not a bottleneck as we very rarely miss in STLB. We ignore the `Load Miss Latency` metric since the number of cache misses is very low. The ILP is reasonably high. Golden Cove is a 6-wide architecture; an ILP of `3.67` means that the algorithm utilizes almost `2/3` of the core resources every cycle. Memory bandwidth demand is low (only 1.58 GB/s), far from the theoretical maximum for this machine. Looking at the `Ip*` metrics we can tell that Blender is a floating-point algorithm (see `IpFLOP` metric), a large portion of which is vectorized FP operations (see `IpArith AVX128`). But also, some portions of the algorithm are non-vectorized scalar FP single precision instructions (`IpArith Scal SP`). Also, notice that every 90th instruction is an explicit software memory prefetch (`IpSWPF`); we expect to see those hints in Blender's source code. Preliminary conclusion: Blender's performance is bound by FP compute.

* __Blender__。P核和E核的工作负载分配相当均衡，两种核心类型的IPC都相当不错。每千条指令的缓存未命中次数非常低（参见 `L*MPKI`）。分支预测错误是一个轻微的瓶颈：`Br. Misp. Ratio` 指标为 `2%`；平均每610条指令出现1次预测错误（参见 `IpMispredict` 指标），这相当好。TLB不是瓶颈，因为STLB的未命中率非常低。由于缓存未命中次数非常低，我们忽略了 `Load Miss Latency` 指标。指令级并行度(ILP)相当高。Golden Cove是一个6执行宽度体系结构；`3.67` 的ILP意味着该算法每个周期几乎利用了 `2/3` 的核心资源。内存带宽需求较低（仅为1.58GB/s），远低于该机器的理论最大值。查看 `Ip*` 指标可知，Blender是一个浮点算法（参见 `IpFLOP` 指标），其中很大一部分是向量化的浮点运算（参见 `IpArith AVX128`）。但算法中也包含一些非向量化的标量单精度浮点指令（`IpArith Scal SP`）。此外，请注意每90条指令中就有一条是显式的软件内存预取指令（`IpSWPF`）；我们预期会在Blender的源代码中看到这些提示。初步结论：Blender的性能受限于浮点计算。

* __Stockfish__. We ran it using only one hardware thread, so there is zero work on E-cores, as expected. The number of L1 misses is relatively high, but then most of them are contained in L2 and L3 caches. The branch misprediction ratio is high; we pay the misprediction penalty every `215` instructions. We can estimate that we get one mispredict every `215 (instructions) / 1.80 (IPC) = 120` cycles, which is very frequent. Similar to the Blender reasoning, we can say that TLB and DRAM bandwidth is not an issue for Stockfish. Going further, we see that there are almost no FP operations in the workload (see `IpFLOP` metric). Preliminary conclusion: Stockfish is an integer compute workload, which is heavily affected by branch mispredictions.

* __Stockfish__。我们仅使用一个硬件线程运行它，因此正如预期的那样，E核心没有工作。L1缓存未命中次数相对较高，但其中大部分都包含在L2和L3缓存中。分支预测错误率较高；我们每执行215条指令就会付出一次预测错误惩罚。我们可以估计，每 `215条指令 / 1.80 (IPC) = 120` 个周期就会出现一次预测错误，这非常频繁。与Blender的推理类似，我们可以说TLB和DRAM带宽对Stockfish来说不是问题。进一步分析，我们发现工作负载中几乎没有浮点运算（参见 `IpFLOP` 指标）。初步结论：Stockfish是一个整数计算工作负载，它极易受到分支预测错误的影响。

* __Clang 15 selfbuild__. Compilation of C++ code is one of the tasks that has a very flat performance profile, i.e., there are no big hotspots. You will see that the running time is attributed to many different functions. The first thing we spot is that P-cores are doing 68% more work than E-cores and have 42% better IPC. But both P- and E-cores have low IPC. The L*MPKI metrics don't look troubling at first glance; however, in combination with the load miss real latency (`LdMissLat`, in core clocks), we can see that the average cost of a cache miss is quite high (~77 cycles). Now, when we look at the `*STLB_MPKI` metrics, we notice substantial differences with any other benchmark we test. This is due to another aspect of the Clang compiler (and other compilers as well): the size of the binary is relatively big (more than 100 MB). The code constantly jumps to distant places causing high pressure on the TLB subsystem. As you can see the problem exists both for instructions (see `Code stlb MPKI`) and data (see `Ld stlb MPKI`). Let's proceed with our analysis. DRAM bandwidth use is higher than for the two previous benchmarks, but still is not reaching even half of the maximum memory bandwidth on our platform (which is ~34 GB/s). Another concern for us is the very small number of instructions per call (`IpCall`): only ~41 instructions per function call. This is unfortunately the nature of the compiler's codebase: it has thousands of small functions. The compiler needs to be more aggressive with inlining all those functions and wrappers.[^3] Yet, we suspect that the performance overhead associated with making a function call remains an issue for the Clang compiler. Also, one can spot the high `ipBranch` and `IpMispredict` metrics. For Clang compilation, every fifth instruction is a branch and one of every ~35 branches gets mispredicted. There are almost no FP or vector instructions, but this is not surprising. Preliminary conclusion: Clang has a large codebase, flat profile, many small functions, and "branchy" code; performance is affected by data cache misses and TLB misses, and branch mispredictions.

* __Clang 15 自编译__。C++代码的编译是性能曲线非常平坦的任务之一，也就是说，没有明显的性能瓶颈。你会发现运行时间是由许多不同的函数决定的。我们首先注意到的是，P核的工作量比E核多68%，并且IPC提高了42%。但P核和E核的IPC都很低。乍一看，L*MPKI指标似乎没什么问题；然而，结合加载Load未命中实际延迟（`LdMissLat`，以核心时钟频率为单位），我们可以看出缓存未命中的平均成本相当高（约77个时钟周期）。现在，当我们查看`*STLB_MPKI`指标时，会发现它与我们测试的任何其他基准测试都存在显著差异。这是由于Clang编译器（以及其他编译器）的另一个特性造成的：二进制文件相对较大（超过100MB）。代码不断跳转到较远的位置，导致TLB子系统压力过大。正如您所看到的，这个问题既存在于指令（参见`Code stlb MPKI`），也存在于数据（参见`Ld stlb MPKI`）。让我们继续分析。DRAM带宽使用率高于前两个基准测试，但仍然不到我们平台最大内存带宽（约34GB/s）的一半。我们关注的另一个问题是每次函数调用所需的指令数（`IpCall`）：每次函数调用仅约41条指令。不幸的是，这是编译器代码库的特性所致：它包含数千个小型函数。编译器需要更积极地内联所有这些函数和包装器[^3]。然而，我们怀疑函数调用带来的性能开销仍然是Clang编译器的一个问题。此外，还可以注意到较高的 `ipBranch` 和 `IpMispredict` 指标。对于Clang编译，每五条指令中就有一条是分支指令，而大约每35条分支指令中就有一条会被错误预测。几乎没有浮点指令或向量指令，但这并不令人意外。初步结论：Clang代码库庞大，性能扁平，包含大量小型函数和“分支”代码；性能受数据缓存未命中、TLB未命中和分支预测错误的影响。

* __CloverLeaf__. As before, we start with analyzing instructions and core cycles. The amount of work done by P- and E-cores is roughly the same, but it takes P-cores more time to do this work, resulting in a lower IPC of one logical thread on P-core compared to one physical E-core.[^2] The `L*MPKI` metrics are high, especially the number of L3 misses per kilo instructions. The load miss latency (`LdMissLat`) is off the charts, suggesting an extremely high price of the average cache miss. Next, we take a look at the `DRAM BW use` metric and see that memory bandwidth consumption is near its limits. That's the problem: all the cores in the system share the same memory bus, so they compete for access to the main memory, which effectively stalls the execution. CPUs are undersupplied with the data that they demand. Going further, we can see that CloverLeaf does not suffer from mispredictions or function call overhead. The instruction mix is dominated by FP double-precision scalar operations with some parts of the code being vectorized. Preliminary conclusion: multi-threaded CloverLeaf is bound by memory bandwidth.

* __CloverLeaf__。和之前一样，我们首先分析指令和核心周期。P核心和E核心完成的工作量大致相同，但P核心完成这些工作需要更多时间，导致P核心上单个逻辑线程的IPC低于单个物理E核心[^2]。`L*MPKI`指标很高，尤其是每千指令的L3缓存未命中次数。加载未命中延迟(`LdMissLat`)远超图表范围，表明平均缓存未命中代价极高。接下来，我们查看`DRAM BW use`指标，发现内存带宽消耗接近极限。问题在于：系统中的所有核心共享同一条内存总线，因此它们会争用主内存，这实际上会阻碍执行。CPU无法获得其所需的数据。进一步分析，我们可以看到CloverLeaf不存在预测错误或函数调用开销的问题。指令组合以浮点双精度标量运算为主，部分代码已向量化。初步结论：多线程CloverLeaf的性能受限于内存带宽。

As you can see from this study, there is a lot one can learn about the behavior of a program just by looking at the metrics. It answers the "what?" question, but doesn't tell you the "why?". For that, you will need to collect a performance profile, which we will introduce in later chapters. In Part 2 of this book, we will discuss how to mitigate the performance issues we suspect to exist in the four benchmarks that we have analyzed.

正如本研究所示，仅通过观察指标就能了解程序的许多行为信息。它回答了“是什么？”的问题，但无法解释“为什么？”。为此，您需要收集性能分析数据，我们将在后续章节中介绍。本书第二部分将讨论如何缓解我们分析的四个基准测试中可能存在的性能问题。

Keep in mind that the summary of performance metrics in Table {@tbl:perf_metrics_case_study} only tells you about the *average* behavior of a program. For example, we might be looking at CloverLeaf's IPC of `0.2`, while in reality, it may never run with such an IPC. Instead, it may have 2 phases of equal duration, one running with an IPC of `0.1`, and the second with an IPC of `0.3`. Performance tools tackle this by reporting statistical data for each metric along with the average value. Usually, having min, max, 95th percentile, and variation (stdev/avg) is enough to understand the distribution. Also, some tools allow plotting the data, so you can see how the value for a certain metric changed during the program running time. As an example, Figure @fig:CloverMetricCharts shows the dynamics of IPC, L*MPKI, DRAM BW, and average frequency for the CloverLeaf benchmark. The `pmu-tools` package can automatically build those charts once you add the `--xlsx` and `--xchart` options. The `-I 10000` option aggregates collected samples with 10-second intervals.

请注意，表 {@tbl:perf_metrics_case_study} 中的性能指标汇总仅反映了程序的*平均*性能。例如，我们可能看到CloverLeaf的IPC为 `0.2`，但实际上，它可能永远不会以这样的IPC运行。相反，它可能分为两个持续时间相等的阶段，一个阶段的IPC为 `0.1`，另一个阶段的IPC为 `0.3`。性能工具通过报告每个指标的统计数据及其平均值来解决这个问题。通常，最小值、最大值、第95百分位数和变异系数（标准差/平均值）足以了解分布情况。此外，一些工具允许绘制数据图，以便您可以查看特定指标的值在程序运行期间的变化情况。例如，图 @fig:CloverMetricCharts 显示了CloverLeaf基准测试中IPC、L*MPKI、DRAM带宽和平均频率的动态变化。添加 `--xlsx` 和 `--xchart` 选项后，`pmu-tools` 软件包可以自动生成这些图表。`-I 10000` 选项以10秒的间隔聚合收集的样本。

```bash
$ ~/workspace/pmu-tools/toplev.py -m --global --no-desc -v --xlsx workload.xlsx –xchart -I 10000 -- ./clover_leaf
```

![Performance metrics charts for the CloverLeaf benchmark with 10 second intervals. CloverLeaf基准测试的性能指标图表，间隔10秒。](../../img/terms-and-metrics/CloverMetricCharts2.png){#fig:CloverMetricCharts width=98% }

Even though the deviation from the average values reported in the summary is not very big, we can see that the workload is not stable. After looking at the IPC chart for P-core we can hypothesize that there are no distinct phases in the workload and the variation is caused by multiplexing between performance events (discussed in [@sec:counting]). Yet, this is only a hypothesis that needs to be confirmed or disproved. Possible ways to proceed would be to collect more data points by running collection with higher granularity (in our case it was 10 seconds). The chart that plots L*MPKI suggests that all three metrics hover around their average numbers without much deviation. The DRAM bandwidth utilization chart indicates that there are periods with varying pressure on the main memory. The last chart shows the average frequency of all CPU cores. As you may observe on this chart, throttling starts after the first 10 seconds. I recommend being careful when drawing conclusions just from looking at the aggregate numbers since they may not be a good representation of the workload behavior.

尽管与摘要中报告的平均值偏差不大，但我们可以看出工作负载并不稳定。查看P核的IPC图表后，我们可以推测工作负载没有明显的阶段，这种波动是由性能事件之间的多路复用造成的（详见 [@sec:counting]）。然而，这仅仅是一个假设，需要进一步验证。一种可行的方法是通过运行更细粒度的数据采集（在本例中为10秒）来收集更多数据点。绘制L*MPKI的图表显示，所有三个指标都围绕其平均值波动，偏差不大。DRAM 带宽利用率图表表明，主内存存在压力波动较大的时期。最后一个图表显示了所有CPU核心的平均频率。正如您在该图表中看到的，降频在前10秒后开始。我建议谨慎对待仅根据汇总数据得出结论的情况，因为这些数据可能无法很好地代表工作负载的实际行为。

Remember that collecting performance metrics is not a substitute for looking into the code. Always try to find explanation for the numbers that you see by checking relevant parts of the code.

请记住，收集性能指标并不能代替查看代码。务必通过检查代码的相关部分来寻找所见数值背后的原因。

In summary, performance metrics help you build the right mental model about what is and what is *not* happening in a program. Going further into analysis, these data will serve you well.

总而言之，性能指标有助于您构建正确的思维模型，了解程序中哪些功能正在运行，哪些功能没有运行。在进行更深入的分析时，这些数据将对您大有裨益。

[^1]: pmu-tools - [https://github.com/andikleen/pmu-tools](https://github.com/andikleen/pmu-tools)
[^2]: A possible explanation for that is because CloverLeaf is very memory-bandwidth bound. All P- and E-cores are equally stalled waiting on memory. Because P-cores have a higher frequency, they waste more CPU clocks than E-cores. 一种可能的解释是，CloverLeaf体系结构非常依赖内存带宽。所有P核和E核都会因等待内存而停滞不前。由于P核的频率更高，因此它们比E核浪费更多的 CPU 时钟周期。
[^3]: Perhaps by using Link Time Optimizations (LTO). 或许可以通过使用链接时优化(LTO: Link Time Optimization)来实现。
