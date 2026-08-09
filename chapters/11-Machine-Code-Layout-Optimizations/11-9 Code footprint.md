## Case Study: Measuring Code Footprint 案例研究：测量代码占用空间 {#sec:CodeFootprint}

As I mentioned a couple of times in this chapter, code layout optimizations are most impactful on applications with large amounts of code. The best way to clarify the uncertainty about the size of the hot code in your program is to measure its *code footprint*, which is defined as the number of bytes/cache lines/pages with machine instructions the program touches during its execution.

正如我在本章中多次提到的，代码布局优化对代码量大的应用程序影响最大。要明确程序中热点代码的大小，最好的方法是测量其*代码占用空间*，它定义为程序在执行过程中访问的机器指令的字节数/缓存行数/页数。

A large code footprint by itself doesn't necessarily negatively impact performance. Code footprint is not a decisive metric, and it doesn't immediately tell you if there is a problem. Nevertheless, it has proven to be useful as an additional data point in performance analysis. In conjunction with TMA's `Frontend_Bound`, L1-instruction cache miss rate, and other metrics, it may strengthen the argument for investing time in optimizing the machine code layout of your application.

一个较大的代码占用空间本身并不一定会对性能产生负面影响。代码占用空间并非决定性指标，它并不能立即告诉你是否存在问题。然而，它已被证明是性能分析中一个有用的附加数据点。结合TMA的 `前端瓶颈Frontend_Bound`、L1指令缓存未命中率和其他指标，它可以增强投入时间优化应用程序机器代码布局的必要性。

Currently, there are very few tools available that can reliably measure code footprint. In this case study, I will demonstrate [perf-tools](https://github.com/aayasin/perf-tools),[^1] an open-source collection of profiling tools built on top of Linux `perf`. To estimate[^2] code footprint, `perf-tools` leverages Intel's LBR (see [@sec:lbr]), so it currently doesn't work on AMD- or ARM-based systems. Below is a sample command to collect the code footprint data:

目前，能够可靠地测量代码占用空间的工具非常少。在本案例研究中，我将演示基于Linux `perf` 构建的开源性能分析工具集[perf-tools](https://github.com/aayasin/perf-tools)[^1]。为了估算[^2]代码占用空间，`perf-tools` 利用了Intel的LBR（参见[@sec:lbr]），因此目前它无法在基于AMD或ARM的系统上运行。以下是一个用于收集代码占用空间数据的示例命令：

```
$ perf-tools/do.py profile --profile-mask 100 -a <your benchmark>
```

`--profile-mask 100` initiates LBR sampling, and `-a` enables you to specify a program to run. This command will collect code footprint along with various other data. I don't show the output of the tool, curious readers are welcome to study documentation and experiment with the tool.

`--profile-mask 100` 参数用于启动LBR采样，`-a` 参数用于指定要运行的程序。该命令将收集代码占用信息以及其他各种数据。我不展示该工具的输出结果，感兴趣的读者可以查阅文档并自行尝试使用该工具。

I took a set of four benchmarks: Clang C++ compilation, Blender ray tracing, Cloverleaf hydrodynamics, and Stockfish chess engine; these workloads should be already familiar to you from [@sec:PerfMetricsCaseStudy] where we analyzed their performance characteristics. I ran them on an Intel's Alder Lake-based processor.[^5]

我选取了4个基准测试：Clang C++编译、Blender光线追踪、Cloverleaf流体动力学和 Stockfish国际象棋引擎；这些工作负载您应该已经通过 [@sec:PerfMetricsCaseStudy] 了解，我们在该案例中分析了它们的性能特征。我在Intel Alder Lake处理器上运行这些测试的[^5]。

Before we start looking at the results, let's spend some time on terminology. Different parts of a program's code may be exercised with different frequencies, so some parts will be hotter than others. The `perf-tools` package doesn't make this distinction and uses the term "non-cold code" to refer to code that was executed at least once. This is called *two-way splitting* since it splits the code into cold and non-cold parts. Other tools (e.g., Meta's HHVM) use *three-way splitting* and distinguish between hot, warm, and cold code with an adjustable threshold between warm and hot. In this section, we use the term "hot code" to refer to the non-cold code.

在开始查看结果之前，我们先来了解一下术语。程序的不同代码部分可能以不同的频率执行，因此某些部分会比其他部分更“热”。`perf-tools` 包没有区分这一点，而是使用术语“非冷代码（non-cold code）”来指代至少执行过一次的代码。这被称为“*双向分割（two-way splitting）*”，因为它将代码分割成冷代码和非冷代码两个部分。其他工具（例如：Meta的HHVM）使用“*三向分割（three-way splitting）*”，通过一个可调节的阈值来区分热代码、温代码和冷代码。在本节中，我们使用术语“热代码（hot code）”来指代非冷代码。

Results for each of the four benchmarks are presented in Table @tbl:code_footprint. The binary and `.text` sizes were obtained with a standard Linux `readelf` utility, while other metrics were collected with `perf-tools`. The `non-cold code footprint [KB]` metric is the number of kilobytes with machine instructions that a program touched at least once. The metric `non-cold code [4KB-pages]` tells us the number of non-cold 4KB-pages with machine instructions that a program touched at least once. Together they help us to understand how dense or sparse those non-cold memory locations are. It will become clear once we dig into the numbers. Finally, we also present Frontend Bound percentages, a metric that should be already familiar to you from [@sec:TMA] about TMA.

4个基准测试的结果分别列于表 @tbl:code_footprint 中。二进制文件和 `.text` 文件的大小使用标准的Linux工具 `readelf` 获取，而其他指标则使用 `perf-tools` 收集。“非冷代码占用空间[KB]”指标表示程序至少访问过一次的包含机器指令的内存空间KB值。“非冷代码[4KB页]”指标表示程序至少访问过一次的包含机器指令的非冷4KB页的数量。这两个指标共同帮助我们了解这些非冷内存位置的密集程度或稀疏程度。一旦我们深入分析数据，一切就会明朗起来。最后，我们还会展示前端瓶颈百分比，您应该已经从 [@sec:TMA] 关于TMA的介绍中了解过这个指标。

--------------------------------------------------------------------------------
Metric                                  Clang17   Blender  CloverLeaf  Stockfish      
                                    compilation                                
----------------------------------- ----------- --------- ----------- ----------
Binary size [KB]                         113844    223914         672      39583

`.text` size [KB]                         67309    133009         598        238

non-cold code footprint [KB]               5042       313         104         99

non-cold code [4KB-pages]                  6614       546         104         61

Frontend Bound [%]                         52.3      29.4         5.3       25.8
--------------------------------------------------------------------------------

Table: Code footprint of the benchmarks used in the case study. 案例研究中使用的基准测试的代码占用情况。{#tbl:code_footprint}

Let's first look at the binary and `.text` sizes. CloverLeaf is a tiny application compared to Clang17 and Blender; Stockfish embeds the neural network file which accounts for the largest part of the binary, but its code section is relatively small; Clang17 and Blender have gigantic code bases. The `.text size` metric is the upper bound for our applications, i.e. we assume[^3] the code footprint should not exceed the `.text` size.

我们首先来看一下二进制和 `.text` 代码段的大小。与Clang17和Blender相比，CloverLeaf是一个很小的应用程序；Stockfish嵌入了神经网络文件，该文件占据了二进制文件的大部分，但其代码部分相对较小；Clang17和Blender的代码库非常庞大。代码段尺寸 `.text size` 是我们应用程序的上限指标，也就是说，我们假设[^3] 代码占用情况不应超过 `.text` 的大小。

A few interesting observations can be made by analyzing the code footprint data. First, even though the Blender `.text` section is very large, less than 1% of Blender's code is non-cold: 313 KB out of 133 MB. So, just because a binary size is large, doesn't mean the application suffers from CPU Frontend bottlenecks. It's the amount of hot code that matters. For other benchmarks this ratio is higher: Clang17 7.5%, CloverLeaf 17.4%, Stockfish 41.6%. In absolute numbers, the Clang17 compilation touches an order of magnitude more bytes with machine instructions than the other three applications combined.

通过分析代码占用情况数据，我们可以得出一些有趣的结论。首先，尽管Blender的 `.text` 段非常大，但Blender代码中只有不到1%是非冷代码：133MB中的313KB。因此，二进制文件大小大并不意味着应用程序会受到CPU前端瓶颈的影响。真正重要的是热代码的数量。对于其他基准测试，这个比例更高：Clang17为7.5%，CloverLeaf为17.4%，Stockfish为41.6%。从绝对值来看，Clang17编译过程中机器指令占用的字节数比其他三个应用程序加起来还要多一个数量级。

Second, let's examine the `non-cold code [4KB-pages]` row in the table. For Clang17, non-cold 5042 KB are spread over 6614 4KB pages, which gives us `5042 / (6614 * 4) = 19%` page utilization. This metric tells us how dense/sparse the hot parts of the code are. The closer each hot cache line is located to another hot cache line, the fewer pages are required to store the hot code. The higher the page utilization the better. Basic block placement and function reordering that we discussed earlier in this chapter are perfect examples of a transformation that improves page utilization. For other benchmarks, the percentages are: Blender 14%, CloverLeaf 25%, and Stockfish 41%. 

其次，我们来看一下表格中的“`非冷代码non-cold code [4KB-pages]`”这一行。对于Clang17，5042KB的非冷代码分布在6614个4KB的页面上，页面利用率为 `5042 / (6614 * 4) = 19%`。这个指标反映了代码热点部分的密集程度。每个热点缓存行与其他热点缓存行之间的距离越近，存储热点代码所​​需的页面就越少。页面利用率越高越好。我们在本章前面讨论过的基本块放置和函数重排序就是提高页面利用率的绝佳例子。其他基准测试的百分比分别为：Blender 14%，CloverLeaf 25%，Stockfish 41%。

Now that we quantified the code footprints of the four applications, it's tempting to think about the size of L1-instruction and L2 caches and whether the hot code fits or not. On my Alder Lake-based machine, the L1 I-cache is only 32 KB, which is not enough to fully cover any of the benchmarks that we've analyzed. But remember, at the beginning of this section we said that a large code footprint doesn't immediately point to a problem. Yes, a large codebase puts more pressure on the CPU Frontend, but an instruction access pattern is also crucial for performance. The same locality principles as for data accesses apply. That's why we accompanied it with the Frontend Bound metric from Topdown analysis. 

既然我们已经量化了这4个应用程序的代码占用空间，那么很容易就会想到L1指令缓存和L2缓存的大小，以及热代码是否能够容纳在缓存中。在我的Alder Lake机器上，L1指令缓存只有32KB，这不足以完全覆盖我们分析的任何基准测试。但请记住，在本节开头我们说过，代码占用空间大并不一定意味着存在问题。没错，庞大的代码库会给CPU前端带来更大的压力，但指令访问模式对性能也至关重要。与数据访问相同的局部性原则同样适用于指令访问。这就是为什么我们结合自顶向下分析中的前端瓶颈指标来讨论这个问题。

For Clang17, the 5 MB of non-cold code causes a huge 52.3% Frontend Bound performance bottleneck: more than half of the cycles are wasted waiting for instructions. From all the presented benchmarks, it benefits the most from PGO-type optimizations. CloverLeaf doesn't suffer from inefficient instruction fetch; 75% of its branches are backward jumps, which suggests that those could be relatively small loops executed over and over again. Stockfish, while having roughly the same non-cold code footprint as CloverLeaf, poses a far greater challenge for the CPU Frontend (25.8%). It has a lot more indirect jumps and function calls. Finally, Blender has even more indirect jumps and calls than Stockfish. 

对于Clang17来说，5MB的非冷启动代码导致了高达52.3%的前端性能瓶颈：超过一半的周期都浪费在等待指令上。在所有展示的基准测试中，它从PGO类型的优化中获益最大。CloverLeaf不存在指令获取效率低下的问题；其75%的分支都是向后跳转，这表明这些分支可能是相对较小的循环，可以反复执行。Stockfish虽然与CloverLeaf的非冷启动代码量大致相同，但对CPU前端提出了更大的挑战（25.8%）。它有更多的间接跳转和函数调用。最后，Blender的间接跳转和函数调用甚至比Stockfish还要多。

I stop my analysis at this point as further investigations are outside the scope of this case study. For readers who are interested in continuing the analysis, I suggest drilling down into the Frontend Bound category according to the TMA methodology and looking at metrics such as `ICache_Misses`, `ITLB_Misses`, `DSB coverage`, and others.

我的分析到此结束，因为进一步的研究超出了本案例研究的范围。对于有兴趣继续分析的读者，我建议根据TMA方法论深入研究前端瓶颈类别，并查看诸如 `ICache_Misses`、`ITLB_Misses`、`DSB coverage` 等指标。

Another useful tool to study the code footprint is [llvm-bolt-heatmap](https://github.com/llvm/llvm-project/blob/main/bolt/docs/Heatmaps.md)[^4], which is a part of llvm's BOLT project. This tool can produce code heatmaps that give a fine-grained understanding of the code layout in your application. It is primarily used to evaluate the original layout of hot code and confirm that the optimized layout is more compact.

另一个用于研究代码占用空间的实用工具是[llvm-bolt-heatmap](https://github.com/llvm/llvm-project/blob/main/bolt/docs/Heatmaps.md)[^4]，它是llvm BOLT项目的一部分。该工具可以生成代码热图，从而更细致地了解应用程序中的代码布局。它主要用于评估热代码的原始布局，并确认优化后的布局是否更加紧凑。

[^1]: perf-tools - [https://github.com/aayasin/perf-tools](https://github.com/aayasin/perf-tools)
[^2]: The code footprint data collected by `perf-tools` is not exact since it is based on sampling LBR records. Other tools like Intel's `sde -footprint`, unfortunately, don't provide code footprint. However, it is not hard to write a PIN-based tool yourself that will measure the exact code footprint. `perf-tools` 收集的代码占用数据并不精确，因为它基于对LBR记录的采样。其他工具，例如：Intel的 `sde -footprint`，则无法提供代码占用数据。不过，自己编写一个基于PIN的工具来测量精确的代码占用并不难。
[^3]: It is not always true: an application itself may be tiny, but call into multiple other dynamically linked libraries, or it may make heavy use of kernel code. 情况并非总是如此：应用程序本身可能很小，但会调用多个其他动态链接库，或者大量使用内核代码。
[^4]: llvm-bolt-heatmap - [https://github.com/llvm/llvm-project/blob/main/bolt/docs/Heatmaps.md](https://github.com/llvm/llvm-project/blob/main/bolt/docs/Heatmaps.md)
[^5]: It doesn't matter which machine you use for collecting code footprint as it depends on the program and input data, and not on the characteristics of a particular machine. As a sanity check, I ran it on a Skylake-based machine and got very similar results. 用哪台机器收集代码足迹并不重要，因为它取决于程序和输入数据，而不是特定机器的特性。为了验证这一点，我在一台基于Skylake的机器上运行了它，得到了非常相似的结果。
