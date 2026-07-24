## Performance Metrics 性能指标 {#sec:PerfMetrics}

Being able to collect various performance events is very helpful in performance analysis. However, there is a caveat. Say, you ran a program and collected the `MEM_LOAD_RETIRED.L3_MISS` event, which counts the LLC misses, and it shows you a value of one billion. Sure it sounds like a lot, so you decided to investigate where these cache misses are coming from. Wrong! Are you sure it is an issue? If a program only does two billion loads, then yes, it is a problem as half of the loads miss in the LLC. In contrast, if a program does one trillion loads, then only one in a thousand loads results in a L3 cache miss.

收集各种性能事件对于性能分析非常有帮助。但是，这里需要注意一点。假设你运行了一个程序，并收集了 `MEM_LOAD_RETIRED.L3_MISS` 事件，该事件用于统计LLC缓存未命中次数，结果显示为十亿。这听起来确实很多，所以你决定调查这些缓存未命中来自哪里。错了！你确定这是一个问题吗？如果一个程序只执行了二十亿次加载，那么这确实是个问题，因为其中一半的加载都发生在LLC缓存中。相比之下，如果一个程序执行了一万亿次加载，那么只有千分之一的加载会导致L3缓存未命中。

That's why in addition to the hardware performance events, performance engineers frequently use metrics, which are built on top of raw events. Table {@tbl:perf_metrics} shows a list of metrics for Intel's 12th-gen Golden Cove architecture along with descriptions and formulas. The list is not exhaustive, but it shows the most important metrics. A complete list of metrics for Intel CPUs and their formulas can be found in [TMA_metrics.xlsx](https://github.com/intel/perfmon/blob/main/TMA_Metrics.xlsx).[^1] [@sec:PerfMetricsCaseStudy] shows how performance metrics can be used in practice.

这就是为什么除了硬件性能事件之外，性能工程师还经常使用基于原始事件构建的指标。表 {@tbl:perf_metrics} 列出了Intel第十二代Core微体系结构Golden Cove的指标，以及相应的描述和计算公式。该列表并不完整，但列出了最重要的指标。完整的Intel CPU指标列表及其计算公式可在 [TMA_metrics.xlsx](https://github.com/intel/perfmon/blob/main/TMA_Metrics.xlsx) 中找到。[^1] [@sec:PerfMetricsCaseStudy] 展示了性能指标在实践中的应用。

\small

--------------------------------------------------------------------------
Metric  Description                   Formula
Name指标 描述                          公式  
名字
------- -------------------------- ---------------------------------------
L1MPKI  L1 cache true misses       1000 * MEM_LOAD_RETIRED.L1_MISS_PS /
        per kilo instruction for   INST_RETIRED.ANY
        retired demand loads.
        每千条指令中实际完成加载指令L1
        缓存真实失效次数

L2MPKI  L2 cache true misses       1000 * MEM_LOAD_RETIRED.L2_MISS_PS /
        per kilo instruction for   INST_RETIRED.ANY
        retired demand loads.
        每千条指令中实际完成加载指令L2
        缓存真实失效次数

L3MPKI  L3 cache true misses       1000 * MEM_LOAD_RETIRED.L3_MISS_PS /
        per kilo instruction for   INST_RETIRED.ANY
        retired demand loads.
        每千条指令中实际完成加载指令L3
        缓存真实失效次数

Branch  Ratio of all branches      BR_MISP_RETIRED.ALL_BRANCHES /
Mispr.  which mispredict           BR_INST_RETIRED.ALL_BRANCHES
Ratio   所有分支指令总预测错误比率

Code    STLB (2nd level TLB) code  1000 * ITLB_MISSES.WALK_COMPLETED
STLB    speculative misses per     / INST_RETIRED.ANY
MPKI    kilo instruction (misses
        of any page size that
        complete the page walk)
        每千条指令STLB二级TLB代码
        前瞻失效数量（完成页表遍历
        等任何尺寸页失效）

Load    STLB data load             1000 * DTLB_LD_MISSES.WALK_COMPLETED
STLB    speculative misses         / INST_RETIRED.ANY
MPKI    per kilo instruction
        每千条指令中STLB数据加载前瞻
        失效次数

Store   STLB data store            1000 * DTLB_ST_MISSES.WALK_COMPLETED
STLB    speculative misses         / INST_RETIRED.ANY
MPKI    per kilo instruction
        每千条指令中STLB数据加载前瞻
        失效次数

Load    Average latency for        L1D_PEND_MISS.PENDING /
Miss    L1 D-cache miss demand     MEM_LD_COMPLETED.L1_MISS_ANY
Real    load operations
Latency (in core cycles)
        对L1D缓存失效Load操作的平均 
        延迟（以核心周期衡量）

ILP     Instr. level parallelism   UOPS_EXECUTED.THREAD /
        per core (average number   UOPS_EXECUTED.CORE_CYCLES_GE1,
        of $\mu$ops executed when      divide by 2 if SMT is enabled
        there is execution)
        每个核心的指令级并行（当有在执行
        时的平均未操作执行数量）

MLP     Memory level parallelism   L1D_PEND_MISS.PENDING /
        per-thread (average number L1D_PEND_MISS.PENDING_CYCLES
        of L1 miss demand loads
        when there is at least one
        such miss.)
        每个线程的存储级并行（当至少有
        1个Load在L1缓存失效时，平均每
        周期未完成/待处理的请求数）

DRAM    Average external memory    ( 64 * ( UNC_M_CAS_COUNT.RD +
BW Use  bandwidth use for reads             UNC_M_CAS_COUNT.WR )
GB/sec  and writes                 / 1GB ) / Time
        用于读和写的平均外部访存带宽

IpCall  Instructions per near call INST_RETIRED.ANY /
        (lower number means higher BR_INST_RETIRED.NEAR_CALL
        occurrence rate)
        每次邻近调用的指令数（更低数字
        意味着更高的触发频率

Ip      Instructions per branch    INST_RETIRED.ANY /
Branch                             BR_INST_RETIRED.ALL_BRANCHES
        每条分支后执行的指令

IpLoad  Instructions per load      INST_RETIRED.ANY /
                                   MEM_INST_RETIRED.ALL_LOADS_PS
        每条载入load后执行的指令

IpStore Instructions per store     INST_RETIRED.ANY /
                                   MEM_INST_RETIRED.ALL_STORES_PS
        每条存储store后执行的指令

IpMisp  Number of instructions per INST_RETIRED.ANY /
redict  non-speculative branch     BR_MISP_RETIRED.ALL_BRANCHES
        misprediction
        每条非前瞻分支预测错误后执行
        的指令

IpFLOP  Instructions per FP        See TMA_metrics.xlsx
        (floating point) operation
        每条浮点FP操作对应的指令执行    参见TMA_metrics.xlsx

IpArith Instructions per FP        See TMA_metrics.xlsx
        arithmetic instruction
        每条浮点FP算术操作对应的指令执行 参见TMA_metrics.xlsx

IpArith Instructions per FP arith. INST_RETIRED.ANY /
Scalar  scalar single-precision    FP_ARITH_INST.SCALAR_SINGLE
SP      instruction
        每条浮点FP算术标量单精度执行
        对应的指令执行

IpArith Instructions per FP arith. INST_RETIRED.ANY /
Scalar  scalar double-precision    FP_ARITH_INST.SCALAR_DOUBLE
DP      instruction
        每条浮点FP算术标量双精度执行
        对应的指令执行

Ip      Instructions per           INST_RETIRED.ANY / (
Arith   arithmetic AVX/SSE         FP_ARITH_INST.128B_PACKED_DOUBLE+
AVX128  128-bit instruction        FP_ARITH_INST.128B_PACKED_SINGLE)
        每条算术AVX/SSE 128位指令
        执行对应的指令数

Ip      Instructions per           INST_RETIRED.ANY / (
Arith   arithmetic AVX*            FP_ARITH_INST.256B_PACKED_DOUBLE+
AVX256  256-bit instruction        FP_ARITH_INST.256B_PACKED_SINGLE)
        每条算术AVX 256位执行执行
        对应的指令数

Ip      Instructions per software  INST_RETIRED.ANY /
SWPF    prefetch instruction       SW_PREFETCH_ACCESS.T0:u0xF
        (of any type)
        每次软件预取指令（任何类型）
        执行所对应的指令数
        
--------------------------------------------------------------------------

Table: A list (not exhaustive) of performance metrics along with descriptions and formulas for the Intel Golden Cove architecture. 表格：Intel Golden Cove 架构的性能指标列表（并非详尽无遗），包含描述和计算公式。{#tbl:perf_metrics}

\normalsize

A few notes on those metrics. First, the ILP and MLP metrics do not represent theoretical maximums for an application; rather they measure the actual ILP and MLP of an application on a given machine. On an ideal machine with infinite resources, these numbers would be higher. Second, all metrics besides "DRAM BW Use" and "Load Miss Real Latency" are fractions; we can apply fairly straightforward reasoning to each of them to tell whether a specific metric is high or low. But to make sense of "DRAM BW Use" and "Load Miss Real Latency" metrics, we need to put them in context. For the former, we would like to know if a program saturates the memory bandwidth or not. The latter gives you an idea of the average cost of a cache miss, which is useless by itself unless you know the latencies of each component in the cache hierarchy. We will discuss how to find out cache latencies and peak memory bandwidth in the next section.

关于这些指标的几点说明。首先，指令级并行(ILP)和存储级并行(MLP)指标并非应用程序的理论最大值；它们衡量的是应用程序在给定机器上的实际ILP和MLP。在拥有无限资源的理想机器上，这些数值会更高。其次，除了“DRAM带宽使用”和“加载Load未命中实际延迟”之外，所有指标均为分数；我们可以对每个指标进行相当简单的推理，以判断其高低。但是，要理解“DRAM带宽使用”和“加载Load未命中实际延迟”这两个指标，我们需要将其置于上下文中。对于前者，我们想知道程序是否达到了内存带宽饱和。后者则提供了缓存未命中的平均成本，但除非您知道缓存层次结构中每个组件的延迟，否则该值本身毫无意义。我们将在下一节讨论如何查找缓存延迟和峰值内存带宽。

Some tools can report performance metrics automatically. If not, you can always calculate those metrics manually since you know the formulas and corresponding performance events that must be collected. Table {@tbl:perf_metrics} provides formulas for the Intel Golden Cove architecture, but you can build similar metrics on another platform as long as underlying performance events are available.

有些工具可以自动报告性能指标。如果没有，您可以手动计算这些指标，因为您知道公式以及需要收集的相应性能事件。表 {@tbl:perf_metrics} 提供了Intel公司Golden Cove体系结构的公式，但只要底层性能事件可用，您也可以在其他平台上构建类似的指标。

[^1]: TMA metrics - [https://github.com/intel/perfmon/blob/main/TMA_Metrics.xlsx](https://github.com/intel/perfmon/blob/main/TMA_Metrics.xlsx).
