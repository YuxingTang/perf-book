## Case Study: Sensitivity to Last Level Cache Size 案例研究：末级缓存大小的敏感性 {#sec:Sensitivity2LLC}

In this case study, we run the same set of applications multiple times with varying LLC sizes. Modern server processors let users control the allocation of LLC space to processor threads. In this way, a user can limit each thread to only use its allocated amount of shared resources. Such facilities are often called Quality of Service (QoS) extensions. They can be used to prioritize performance-critical applications and to reduce interference with other threads in the same system. Besides LLC allocation, QoS extensions support limiting memory read bandwidth.

在本案例研究中，我们多次运行同一组应用程序，每次使用不同的末级缓存大小。现代服务器处理器允许用户控制分配给处理器线程的末级缓存空间。这样，用户可以限制每个线程只能使用其分配的共享资源量。这种功能通常被称为服务质量(QoS: Quality of Service)扩展。它们可用于优先处理性能关键型应用程序，并减少对同一系统中其他线程的干扰。除了末级缓存分配之外，QoS扩展还支持限制内存读取带宽。

Our analysis will help us identify applications whose performance drops significantly when decreasing the size of the LLC. We say that such applications are sensitive to the size of the LLC. Also, we identified applications that are not sensitive, i.e., LLC size doesn't have an impact on performance. This result can be applied to properly size the processor LLC, especially considering the wide range available on the market. For example, we can determine that an application benefits from a larger LLC. Then perhaps an investment in new hardware is justified. Conversely, if the performance of an application doesn't improve from having a large LLC, then we can probably buy a cheaper processor.

我们的分析将帮助我们识别那些在减小末级缓存大小时性能显著下降的应用程序。我们称这些应用程序对末级缓存大小敏感。此外，我们还识别出一些不敏感的应用程序，即末级缓存大小对其性能没有影响。考虑到市场上可供选择的末级缓存种类繁多，这一结果可用于合理设置处理器末级缓存的大小。例如，我们可以确定某个应用程序受益于更大的末级缓存。那么，投资新的硬件或许是合理的。反之，如果应用程序的性能并未因使用较大的LLC而得到提升，那么我们或许可以考虑购买更便宜的处理器。

For this case study, we use an AMD Milan processor, but other server processors such as Intel Xeon [@QoSXeon], and Arm ThunderX [@QoSThunderX], also include hardware support for users to control the allocation of both LLC space and memory read bandwidth to processor threads. Based on our tests, the method described in this section works equally well on AMD Zen4-based desktop processors, such as 7950X and 7950X3D.

在本案例研究中，我们使用AMD Milan处理器，但其他服务器处理器，例如Intel Xeon [@QoSXeon]和Arm ThunderX[@QoSThunderX]，也提供硬件支持，允许用户控制分配给处理器线程的LLC空间和内存读取带宽。根据我们的测试，本节所述方法同样适用于基于AMD Zen4体系结构的桌面处理器，例如：7950X和7950X3D。

### Target machine: AMD EPYC 7313P 目标机器：AMD EPYC 7313P {.unlisted .unnumbered}

We have used a server system with a 16-core AMD EPYC 7313P processor, code-named Milan, which AMD launched in 2021. The main characteristics of this system are specified in table @tbl:experimental_setup. 

我们使用了一台搭载16核AMD EPYC 7313P处理器（代号Milan）的服务器系统，该处理器由AMD于2021年发布。该系统的主要特性详见表 @tbl:experimental_setup。

\small

---------------------------------------------------------------------------
Feature               Value
----------------      -----------------------------------------------------
Processor             AMD EPYC 7313P

Cores x threads       16 $\times$ 2
 
Configuration         4 CCX $\times$ 4 cores/CCX
 
Frequency             3.0/3.7 GHz, base/max

L1 cache (I, D)       8-ways, 32 KiB (per core), 64-byte lines

L2 cache              8-ways, 512 KiB (per core), 64-byte lines

LLC                   16-ways, 32 MB, non-inclusive (per CCX), 64-byte lines
 
Main Memory           512 GiB DDR4, 8 channels, nominal peak BW: 204.8 GB/s

TurboBoost            Disabled

Hyperthreading        Disabled (1 thread/core)

OS                    Ubuntu 22.04, kernel 5.15.0-76

---------------------------------------------------------------------------

Table: Main features of the server used in the experiments. 实验中所用服务器的主要特性。 {#tbl:experimental_setup}

\normalsize

Figure @fig:milan7313P shows the clustered memory hierarchy of an AMD Milan 7313P processor. It consists of four Core Complex Dies (CCDs) connected to each other and to off-chip memory via an I/O chiplet. Each CCD integrates a Core CompleX (CCX) and an I/O connection. In turn, each CCX has four Zen3 cores that share a 32 MB victim LLC.[^11]

图 @fig:milan7313P 展示了AMD Milan 7313P处理器的集群内存层次结构。它由4个核心复合体芯片管芯(CCD: Core Complex Die) 组成，这些芯粒彼此连接，并通过I/O芯粒连接到片外内存。每个CCD都集成了一个核心复合体 (CCX: Core CompleX)和一个I/O连接。每个CCX包含4个Zen3核心，它们共享一个32MB的最外一级缓存（LLC: Last Level Cache）[^11]。

![The clustered memory hierarchy of the AMD Milan 7313P processor.AMD Milan 7313P处理器的集群存储层次结构图。](../../img/other-tuning/Milan7313P.png){#fig:milan7313P width=80%}

Although there is a total of 128 MB of LLC (32 MB/CCX x 4 CCX), the four cores of a CCX cannot store cache lines in an LLC other than their own 32 MB LLC. Since we will be running single-threaded benchmarks, we can focus on a single CCX. The LLC size in our experiments will vary from 0 to 32 MB with steps of 2 MB. This is directly related to having a 16-way LLC: by disabling one of 16 ways, we reduce the LLC size by 2 MB.

虽然总共有128MB的LLC（32MB/CCX x 4个CCX），但一个CCX的4个核心无法将缓存行存储在除自身32MBLLC之外的其他LLC中。由于我们将运行单线程基准测试，因此可以专注于单个CCX。在我们的实验中，LLC的大小将从0到32MB不等，步长为2MB。这与16路LLC直接相关：禁用其中一条路，LLC的大小就会减少2MB。

### Workload: SPEC CPU2017 工作负载：SPEC CPU2017 {.unlisted .unnumbered}

We use a subset of applications from the SPEC CPU2017 suite[^4]. SPEC CPU2017 contains a collection of industry-standardized workloads that benchmark the performance of the processor, memory subsystem, and compiler. It is widely used to compare the performance of high-performance systems and in computer architecture research. 

我们使用SPEC CPU2017测试套件[^4]中的部分应用程序。SPEC CPU2017包含一系列行业标准化的工作负载，用于评估处理器、内存子系统和编译器的性能。它被广泛用于比较高性能系统的性能以及计算机体系结构研究。

Specifically, we selected 15 memory-intensive benchmarks from SPEC CPU2017 (6 INT and 9 FP) as suggested in [@MemCharacterizationSPEC2006]. These applications have been compiled with GCC 6.3.1 and the following compiler options: `-O3 -march=native -fno-unsafe-math-optimizations`.

具体来说，我们根据 [@MemCharacterizationSPEC2006] 中的建议，从SPEC CPU2017中选择了15个内存密集型基准测试程序（6个整数和9个浮点运算）。这些应用程序使用GCC 6.3.1编译，并采用了以下编译器选项：`-O3 -march=native -fno-unsafe-math-optimizations`。

### Controlling and Monitoring LLC allocation 控制和监控LLC分配 {.unlisted .unnumbered}

To monitor and enforce limits on LLC allocation and memory _read_ bandwidth, we will use _AMD64 Technology Platform Quality of Service Extensions_ [@QoSAMD]. Users can manage this QoS extension through the banks of model-specific registers (MSRs). First, a thread or a group of threads must be assigned a resource management identifier (RMID), and a class of service (COS) by writing to the `PQR_ASSOC` register (MSR `0xC8F`). Here is a sample command for the hardware thread 1:

为了监控和强制执行LLC分配和内存读取带宽的限制，我们将使用*AMD64技术平台服务质量扩展(AMD64 Technology Platform Quality of Service Extensions)* [@QoSAMD]。用户可以通过特定于型号的寄存器(MSR: Model-Specific Register)组来管理此QoS扩展。首先，必须通过写入 `PQR_ASSOC` 寄存器（MSR `0xC8F`）为线程或线程组分配资源管理标识符(RMID)和服务等级(COS)。以下是硬件线程1的示例命令：

```bash
# write PQR_ASSOC (MSR 0xC8F): RMID=1, COS=2 -> (COS << 32) + RMID
$ wrmsr -p 1 0xC8F 0x200000001
```

where `-p 1` refers to the hardware thread 1. All `rdmsr` and `wrmsr` commands that we show require root access.

其中 `-p 1` 指的是硬件线程1。我们展示的所有 `rdmsr` 和 `wrmsr` 命令都需要root权限。

LLC space management is performed by writing to a 16-bit per-thread binary mask. Each bit of the mask allows a thread to use a given sixteenth fraction of the LLC (1/16 = 2 MB in the case of the AMD Milan 7313P). Multiple threads can use the same fraction(s), implying a competitive shared use of the same subset of LLC.

LLC空间管理通过写入一个16位线程二进制掩码来实现。掩码的每一位允许一个线程使用LLC的十六分之一空间（在 AMD Milan 7313P 中，1/16 = 2 MB）。多个线程可以使用相同的空间比例，这意味着它们可以竞争性地共享同一LLC子集。

To set limits on the LLC usage by thread 1, we need to write to the `L3_MASK_n` register, where `n` is the COS, the cache partitions that can be used by the corresponding COS. For example, to limit thread 1 to use only half of the available space in the LLC, run the following command:

要限制线程1对LLC的使用，我们需要写入 `L3_MASK_n` 寄存器，其中 `n` 是COS，即对应COS可以使用的缓存分区。例如，要限制线程1只能使用LLC中可用空间的一半，请运行以下命令：

```bash
# write L3_MASK_2 (MSR 0xC92): 0x00FF (half of the LLC space)
$ wrmsr -p 1 0xC92 0x00FF
```

Similarly, the memory read bandwidth allocated to a thread can be limited. This is achieved by writing an unsigned integer to a specific MSR register, which sets a maximum read bandwidth in 1/8 GB/s increments. Interested readers are welcome to read [@QoSAMD] for more details. 

类似地，可以限制分配给线程的内存读取带宽。这可以通过向特定于型号的寄存器(MSR)写入一个无符号整数来实现，该寄存器以1/8 GB/s的增量设置最大读取带宽。感兴趣的读者可以阅读 [@QoSAMD] 了解更多详情。

### Metrics 指标 {.unlisted .unnumbered}

The ultimate metric for quantifying the performance of an application is execution time. To analyze the impact of the memory hierarchy on system performance, we will also use the following three metrics: 1) CPI, cycles per instruction, 2) DMPKI, demand misses in the LLC per thousand instructions, and 3) MPKI, total misses (demand + prefetch) in the LLC per thousand instructions. While CPI has a direct correlation with the performance of an application, DMPKI and MPKI do not necessarily impact performance. Table @tbl:metrics shows the formulas used to calculate each metric from specific hardware counters. Detailed descriptions for each of the counters are available in AMD's Processor Programming Reference [@amd_ppr].

量化应用程序性能的最终指标是执行时间。为了分析内存层次结构对系统性能的影响，我们还将使用以下三个指标：1）CPI（每条指令的周期数）；2）DMPKI（每千条指令LLC中的请求未命中次数）；以及3）MPKI（每千条指令LLC中的总未命中次数，包括请求未命中和预取未命中）。虽然CPI与应用程序性能直接相关，但DMPKI和MPKI不一定会影响性能。表 @tbl:metrics 显示了用于根据特定硬件计数器计算每个指标的公式。AMD的处理器编程参考手册 [@amd_ppr] 中提供了每个计数器的详细说明。

\small

------   ----------------------------------------------------------------------------
Metric                                     Formula                   
------   ----------------------------------------------------------------------------
CPI      Cycles not in Halt (PMCx076) / Retired Instructions (PMCx0C0)

DMPKI    Demand Data Cache Fills[^9] (PMCx043) / (Retired Instr (PMCx0C0) / 1000)

MPKI     L3 Misses[^8] (L3PMCx04) / (Retired Instructions (PMCx0C0) / 1000)

------   ----------------------------------------------------------------------------

Table: Formulas for calculating metrics used in the case study. 案例研究中使用的指标计算公式。 {#tbl:metrics}

\normalsize

The methodology used in this case study is described in more detail in [@Balancer2023]. It also explains how we configured and read hardware counters. The code and the information necessary to reproduce the experiments can be found in the following public repository: [https://github.com/agusnt/BALANCER](https://github.com/agusnt/BALANCER).

本案例研究中使用的方法在 [@Balancer2023] 中有更详细的描述。它还解释了我们如何配置和读取硬件计数器。重现实验所需的代码和信息可以在以下公共仓库中找到：[https://github.com/agusnt/BALANCER](https://github.com/agusnt/BALANCER)。

### Results 结果 {.unlisted .unnumbered}

We run a set of SPEC CPU2017 benchmarks *alone* in the system using only one instance and a single hardware thread. We repeat those runs while changing the available LLC size from 0 to 32 MB in 2 MB steps. 

我们在系统中*单独*运行了一组SPEC CPU2017基准测试，仅使用一个实例和一个硬件线程。我们重复这些运行，同时将可用的LLC大小从0增加到32MB，步长为2MB。

Figure @fig:characterization_llc shows in graphs, from left to right, CPI, DMPKI, and MPKI for each assigned LLC size. We only show three workloads, namely `503.bwaves` (blue), `520.omnetpp` (green), and `554.roms` (red). They cover the three main trends observed in all other applications. Thus, we do not show the rest of the benchmarks.

图 @fig:characterization_llc 以图表形式从左到右显示了每个分配的LLC大小对应的CPI、DMPKI和MPKI。我们仅展示3个工作负载，分别是 `503.bwaves`（蓝色）、`520.omnetpp`（绿色）和 `554.roms`（红色）。它们涵盖了所有其他应用程序中观察到的三个主要趋势。因此，我们不展示其余的基准测试结果。

For the CPI chart, a lower value on the Y-axis means better performance. Also, since the frequency on the system is fixed, the CPI chart is reflective of absolute scores. For example, `520.omnetpp` (green line) with 32 MB LLC is 2.5 times faster than with 0 MB LLC. For the DMPKI and MPKI charts, the lower the value on the Y-axis, the better.

对于CPI图表，Y轴上的数值越低表示性能越好。此外，由于系统频率是固定的，CPI图表反映的是绝对分数。例如，使用32MB LLC的 `520.omnetpp`（绿线）比使用0MB LLC 的 `520.omnetpp` 快2.5倍。对于DMPKI和MPKI图表，Y轴上的数值越低越好。

![CPI, DMPKI, and MPKI for increasing LLC allocation limits (2 MB steps). CPI、DMPKI和MPKI随LLC分配限制增加（以2MB为步长）的变化。](../../img/other-tuning/llc-bw.png){#fig:characterization_llc width=100%}

Two different behaviors can be observed in the CPI and DMPKI graphs. On one hand, `520.omnetpp` takes advantage of its available space in the LLC: both CPI and DMPKI decrease significantly as the space allocated in the LLC increases. We can say that the behavior of `520.omnetpp` is sensitive to the size available in the LLC. Increasing the allocated LLC space improves performance because it avoids evicting cache lines that will be used in the future.

在CPI和DMPKI图中可以观察到两种不同的行为。一方面，`520.omnetpp` 充分利用了LLC中的可用空间：随着LLC中分配空间的增加，CPI和DMPKI均显著下降。我们可以说，`520.omnetpp` 的行为对LLC中的可用空间非常敏感。增加分配的LLC空间可以提高性能，因为它避免了驱逐将来会使用的缓存行。

In contrast, `503.bwaves` and `554.roms` don't make use of all available LLC space. For both benchmarks, CPI and DMPKI remain roughly constant as the allocation limit in the LLC grows. We can say that the performance of these two applications is insensitive to their available space in the LLC.[^12] If your application shows similar behavior, you can buy a cheaper processor with a smaller LLC size without sacrificing performance.

相比之下，`503.bwaves` 和 `554.roms` 并没有充分利用所有可用的LLC空间。对于这两个基准测试，随着LLC分配限制的增加，CPI和DMPKI基本保持不变。我们可以说，这两个应用程序的性能对LLC中的可用空间并不敏感[^12]。 如果你的应用程序也表现出类似的行为，你可以购买LLC容量更小、价格更低的处理器，而不会牺牲性能。

Let's now analyze the MPKI graph, that combines both LLC demand misses and prefetch requests. First of all, we can see that the MPKI values are always much higher than the DMPKI values. That is, most of the blocks are loaded from memory into the on-chip hierarchy by the prefetcher. This behavior is due to the fact that the prefetcher is efficient in preloading the private caches with the data to be used, thus eliminating most of the demand misses.

现在让我们分析MPKI图，该图结合了LLC请求未命中和预取请求。首先，我们可以看到MPKI值始终远高于DMPKI值。也就是说，大部分数据块都是由预取器从内存加载到片上缓存层次结构中的。这种现象是由于预取器能够高效地将要使用的数据预加载到私有缓存中，从而消除了大部分请求未命中。

For `503.bwaves`, we observe that MPKI remains roughly at the same level, similar to CPI and DMPKI charts. There is likely not much data reuse in the benchmark and/or the memory traffic is very low. The `520.omnetpp` workload behaves as we identified earlier: MPKI decreases as the available space increases. 

对于 `503.bwaves`，我们观察到MPKI值基本保持不变，与CPI和DMPKI图表类似。基准测试中可能没有太多数据重用，或者内存流量非常低。`520.omnetpp` 工作负载的行为与我们之前观察到的一致：MPKI值随着可用空间的增加而降低。

However, for `554.roms`, the MPKI chart shows a large drop in total misses as the available space increases while CPI and DMPKI remain unchanged. In this case, there is data reuse in the benchmark, but it is not consequential to the performance. The prefetcher can bring required data ahead of time, eliminating demand misses, regardless of the available space in the LLC. However, as the available space decreases, the probability that the prefetcher will not find the blocks in the LLC and will have to load them from memory increases. So, giving more LLC capacity to `554.roms` does not directly benefit its performance, but it does benefit the system since it reduces memory traffic. So, it is better not to limit available LLC space for `554.roms` as it may negatively affect the performance of other applications running on the system. [@Balancer2023]

然而，对于 `554.roms`，MPKI图表显示，随着可用空间的增加，总未命中次数大幅下降，而CPI和DMPKI保持不变。在这种情况下，基准测试中存在数据重用，但这不会对性能造成影响。预取器可以提前获取所需数据，从而消除按需未命中，而无需考虑LLC中的可用空间。但是，随着可用空间的减少，预取器在LLC中找不到数据块并必须从内存加载的概率会增加。因此，为 `554.roms` 提供更多LLC容量并不会直接提升其性能，但由于减少了内存流量，因此对系统有益。所以，最好不要限制 `554.roms` 的可用LLC空间，因为这可能会对系统上运行的其他应用程序的性能产生负面影响[@Balancer2023]。

[^4]: SPEC CPU® 2017 - [https://www.spec.org/cpu2017/](https://www.spec.org/cpu2017/).
[^8]: We used a mask to count only L3 Misses, specifically, `L3Event[0x0300C00000400104]`. 我们使用掩码仅统计 L3 缓存未命中，具体来说是 `L3Event[0x0300C00000400104]`。
[^9]: We used subevents `MemIoRemote` and `MemIoLocal`, that count demand data cache fills from DRAM or IO connected in remote/local NUMA node. 我们使用子事件 `MemIoRemote` 和 `MemIoLocal`，它们统计来自远程/本地NUMA节点连接的DRAM或I/O的按需数据缓存填充次数。
[^11]: "Victim" means that the LLC is filled with the cache lines evicted from the four L2 caches of a CCX. “Victim” 表示LLC被CCX的4个L2缓存中驱逐的缓​​存行填充。
[^12]: However, for `554.roms`, we can observe sensitivity to the available space in the LLC in the range 0-4 MB. Once the LLC size is 4 MB and above, the performance remains constant. That means that `554.roms` doesn't require more than 4 MB of LLC space to perform well. 然而，对于 `554.roms`，我们可以观察到LLC可用空间在0-4MB范围内的敏感性。当LLC大小达到4MB及以上时，性能将保持稳定。这意味着 `554.roms` 只需要4MB的LLC空间即可良好运行。
