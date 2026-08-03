## Sampling 采样 {#sec:profiling}

Sampling is the most frequently used approach for doing performance analysis. People usually associate it with finding hotspots in a program. To put it more broadly, sampling helps to find places in the code that contribute to the highest number of certain performance events. If we want to find hotspots, the problem can be reformulated as: "find a place in the code that consumes the biggest number of CPU cycles". People often use the term *profiling* for what is technically called *sampling*. According to [Wikipedia](https://en.wikipedia.org/wiki/Profiling_(computer_programming)),[^1] profiling is a much broader term and includes a wide variety of techniques to collect data, including sampling, code instrumentation, tracing, and others.

采样是进行性能分析最常用的方法。人们通常将其与查找程序中的热点联系起来。更广泛地说，采样有助于找到代码中导致某些性能事件数量最多的位置。如果我们想要找到热点，问题可以重新表述为：“在代码中找到消耗最多CPU 周期的位置”。人们经常使用术语“*轮廓分析profiling*”来表示技术上称为“*采样sampling*”的内容。根据 [维基百科Wikipedia](https://en.wikipedia.org/wiki/Profiling_(computer_programming))[^1]，轮廓分析Profiling是一个更广泛的术语，包括各种收集数据的技术，包括：采样、代码插桩检测、跟踪等。

It may come as a surprise, but the simplest sampling profiler one can imagine is a debugger. In fact, you can identify hotspots as follows: a) run the program under the debugger; b) pause the program every 10 seconds; and c) record the place where it stopped. If you repeat b) and c) many times, you can build a histogram from collected samples. The line of code where you stopped the most will be the hottest place in the program. Of course, this is not an efficient way to find hotspots, and we don't recommend doing this. It's just to illustrate the concept. Nevertheless, this is a simplified description of how real profiling tools work. Modern profilers are capable of collecting thousands of samples per second, which gives a pretty accurate estimate of the hottest places in a program.

这可能会让人感到惊讶，但人们可以想象的最简单的采样轮廓分析器就是调试器。事实上，您可以通过以下方式识别热点： a）在调试器下运行程序； b）每10秒暂停一次程序； c）记录其停止的位置。如果重复b）和c）多次，您可以根据收集的样本构建直方图。您停止次数最多的代码行将是程序中最热门的地方。当然，这不是查找热点的有效方法，我们不建议这样做。这只是为了说明这个概念。尽管如此，这是对真实轮廓分析工具如何工作的简化描述。现代轮廓分析器每秒能够收集数千个样本，这可以非常准确地估计程序中最热的位置。

As in the example with a debugger, the execution of the analyzed program is interrupted every time a new sample is captured. At the time of an interrupt, the profiler collects the snapshot of the program state, which constitutes one sample. Information collected for every sample may include an instruction address that was executed at the time of the interrupt, register state, call stack (see [@sec:secCollectCallStacks]), etc. Collected samples are stored in a dump file, which can be further used to display the most time-consuming parts of the program, a call graph, etc.

与调试器的示例一样，每次捕获新样本时，被分析程序的执行都会中断。在中断发生时，轮廓分析器会收集程序状态的快照，该快照构成一个样本。为每个样本收集的信息可能包括中断时执行的指令地址、寄存器状态、调用堆栈（请参阅[@sec:secCollectCallStacks]）等。收集的样本存储在一个转储文件中，该文件可进一步用于显示程序中最耗时的部分、调用图等。

### User-Mode and Hardware Event-based Sampling 用户模式和基于硬件事件的采样

Sampling can be performed in 2 different modes, using user-mode or hardware event-based sampling (EBS). User-mode sampling is a pure software approach that embeds an agent library into the profiled application. The agent sets up an OS timer for each thread in the application. Upon timer expiration, the application receives the `SIGPROF` signal that is handled by the agent. EBS uses hardware PMCs to trigger interrupts. In particular, the counter overflow feature of the PMU is used, which we will discuss shortly.

可以使用用户模式或基于硬件事件的采样(EBS: hardware Event-Based Sampling) 以2种不同的模式执行采样。用户模式采样是一种纯软件方法，它将一个代理库嵌入到分析的应用程序中。代理为应用程序中的每个线程设置一个操作系统计时器。计时器到期后，应用程序会收到由代理处理的“SIGPROF”信号。EBS使用硬件PMC来触发中断。特别是使用了PMU的计数器溢出功能特性，我们稍后将对此进行讨论。

User-mode sampling can only be used to identify hotspots, while EBS can be used for additional analysis types that involve PMCs, e.g., sampling on cache-misses, Top-down Microarchitecture Analysis (see [@sec:TMA]), etc.

用户模式采样只能用于识别热点，而EBS可用于涉及PMC的其他分析类型，例如：缓存未命中采样、自顶向下微体系结构分析（参见 [@sec:TMA]）等。

User-mode sampling incurs higher runtime overhead than EBS. The average overhead of the user-mode sampling is about 5% when sampling with an interval of 10ms, while EBS has less than 1% overhead. Because of less overhead, you can use EBS with a higher sampling rate which will give more accurate data. However, user-mode sampling generates fewer data to analyze, and it takes less time to process it. 

用户模式采样会产生比EBS更高的运行时开销。用户态采样在10ms采样间隔时平均开销约为5%，而EBS的开销不到1%。由于开销较小，您可以使用具有更高采样率的EBS，这将提供更准确的数据。然而，用户模式采样生成的分析数据较少，处理数据所需的时间也较少。

### Finding Hotspots 寻找性能热点

In this section, we will discuss the mechanics of using PMCs with EBS. Figure @fig:Sampling illustrates the counter overflow feature of the PMU, which is used to trigger a Performance Monitoring Interrupt (PMI), also known as `SIGPROF`. At the start of a benchmark, we configure the event that we want to sample. Sampling on cycles is a default for many profiling tools since we want to know where the program spends most of the time. However, it is not necessarily a strict rule; we can sample on any performance event we want. For example, if we would like to know the place where the program experiences the biggest number of L3-cache misses, we would sample on the corresponding event, i.e., `MEM_LOAD_RETIRED.L3_MISS`.

在本节中，我们将讨论如何将性能监控控制器(PMC)与EBS结合使用。图 @fig:Sampling 展示了性能监控单元(PMU)的计数器溢出功能，该功能用于触发性能监控中断(PMI: Performance Monitoring Interrupt)，也称为 `SIGPROF`。在基准测试开始时，我们需要配置要采样的事件。许多性能分析工具默认按周期采样，因为我们需要了解程序在哪些地方花费了最多的时间。然而，这并非绝对规则；我们可以对任何所需的性能事件进行采样。例如，如果我们想知道程序在哪个地方经历了最多的L3缓存未命中，我们可以对相应的事件进行采样，即 `MEM_LOAD_RETIRED.L3_MISS`。

![Using performance counter for sampling 使用性能计数器进行采样](../../img/perf-analysis/SamplingFlow.png){#fig:Sampling width=80%}

After we have initialized the register, we start counting and let the benchmark run. Since we have configured a PMC to count cycles, it will be incremented every cycle. Eventually, it will overflow. At the time the register overflows, the hardware will raise a PMI. The profiling tool is configured to capture PMIs and has an Interrupt Service Routine (ISR) for handling them. We do multiple steps inside the ISR: first of all, we disable counting; after that, we record the instruction that was executed by the CPU at the time the counter overflowed; then, we reset the counter to `N` and resume the benchmark.

初始化寄存器后，我们开始计数并运行基准测试。由于我们配置了一个性能计数器(PMC)来计数周期，因此每个周期计数器都会递增。最终，计数器会溢出。当寄存器溢出时，硬件会触发一个PMI（性能中断信息）。性能分析工具配置为捕获PMI，并包含一个用于处理PMI的中断服务例程(ISR: Interrupt Service Routine)。我们在ISR中执行以下几个步骤：首先，禁用计数；然后，记录计数器溢出时CPU执行的指令；最后，将计数器重置为 `N` 并恢复基准测试。

Now, let us go back to the value `N`. Using this value, we can control how frequently we want to get a new interrupt. Say we want a finer granularity and have one sample every 1 million cycles. To achieve this, we can set the counter to `(unsigned) -1,000,000` so that it will overflow after every 1 million cycles. This value is also referred to as the *sample after* value.

现在，让我们回到 `N` 这个值。通过这个值，我们可以控制获取新中断的频率。例如，假设我们需要更精细的粒度，每100万个周期采样一次。为了实现这一点，我们可以将计数器设置为 `(unsigned) -1,000,000`，使其每循环100万次后溢出。该值也称为“采样间隔sample after”值。

We repeat the process many times to build a sufficient collection of samples. If we later aggregate those samples, we could build a histogram of the hottest places in our program, like the one shown in the output from Linux `perf record/report` below. This gives us the breakdown of the overhead for functions of a program sorted in descending order (hotspots). An example of sampling the [x264](https://openbenchmarking.org/test/pts/x264)[^7] benchmark from the [Phoronix test suite](https://www.phoronix-test-suite.com/)[^8] is shown below:

我们重复此过程多次，以构建足够的样本集合。如果我们之后聚合这些样本，就可以构建程序中性能最集中位置的直方图，例如下面Linux `perf record/report` 的输出所示。这可以让我们了解程序中各个函数的开销分布情况（按降序排列，即性能热点）。下面展示了从[Phoronix 测试套件](https://www.phoronix-test-suite.com/)[^8]中对[x264](https://openbenchmarking.org/test/pts/x264)[^7]基准测试进行采样的示例：

```bash
$ time -p perf record -F 1000 -- ./x264 -o /dev/null --slow --threads 1 ../Bosphorus_1920x1080_120fps_420_8bit_YUV.y4m
[ perf record: Captured and wrote 1.625 MB perf.data (35035 samples) ]
real 36.20 sec
$ perf report -n --stdio
# Samples: 35K of event 'cpu_core/cycles/'
# Event count (approx.): 156756064947

# Overhead  Samples  Shared Object  Symbol                                                     
# ........  .......  .............  ........................................
  7.50%     2620     x264           [.] x264_8_me_search_ref
  7.38%     2577     x264           [.] refine_subpel.lto_priv.0
  6.51%     2281     x264           [.] x264_8_pixel_satd_8x8_internal_avx2
  6.29%     2212     x264           [.] get_ref_avx2.lto_priv.0
  5.07%     1787     x264           [.] x264_8_pixel_avg2_w16_sse2
  3.26%     1145     x264           [.] x264_8_mc_chroma_avx2
  2.88%     1013     x264           [.] x264_8_pixel_satd_16x8_internal_avx2
  2.87%     1006     x264           [.] x264_8_pixel_avg2_w8_mmx2
  2.58%      904     x264           [.] x264_8_pixel_satd_8x8_avx2
  2.51%      882     x264           [.] x264_8_pixel_sad_16x16_sse2
  ...
```

Linux `perf` collected `35,035` samples, which means that there were the same number of process interrupts. We also used `-F 1000` which sets the sampling rate at 1000 samples per second. This roughly matches the overall runtime of 36.2 seconds. Notice, that Linux `perf` provided the approximate number of total cycles elapsed. If we divide it by the number of samples, we'll have `156756064947 cycles / 35035 samples = 4.5 million cycles` per sample. That means that Linux `perf` set the number `N` to roughly `4500000` to collect 1000 samples per second. The number `N` can be adjusted by Linux `perf` dynamically according to the actual CPU frequency.

Linux `perf` 收集了35,035个样本，这意味着进程中断的数量也相同。我们还使用了 `-F 1000` 参数，将采样率设置为每秒1000个样本。这与36.2秒的总运行时间大致吻合。请注意，Linux `perf` 提供了经过的总周期数的近似值。如果我们将其除以样本数，则每个样本需要 `156756064947个周期 / 35035个样本 = 450万个周期` 。这意味着Linux `perf` 将 `N` 值设置为大约4500000，以每秒收集1000个样本。Linux `perf`可以根据实际CPU频率动态调整 `N` 值。

And of course, most valuable for us is the list of hotspots sorted by the number of samples attributed to each function. After we know what are the hottest functions, we may want to look one level deeper: what are the hot parts of code inside every function? To see the profiling data for functions that were inlined as well as assembly code generated for a particular source code region, we need to build the application with debug information (`-g` compiler flag). 

当然，对我们来说最有价值的是按每个函数的样本数排序的热点列表。在了解了哪些函数最热门之后，我们可能需要更深入地探究：每个函数内部的热门代码段是什么？要查看特定源代码区域内联函数的性能分析数据以及生成的汇编代码，我们需要使用调试信息（`-g` 编译器标志）构建应用程序。

Linux `perf` doesn't have rich graphic support, so viewing hot parts of source code is not very convenient, but doable. Linux `perf` intermixes source code with the generated assembly, as shown below:

Linux `perf` 的图形支持并不完善，因此查看源代码的热门代码段不太方便，但并非不可能。Linux `perf` 会将源代码与生成的汇编代码混合显示，如下所示：

```bash
# snippet of annotating source code of 'x264_8_me_search_ref' function
$ perf annotate x264_8_me_search_ref --stdio
Percent | Source code & Disassembly of x264 for cycles:ppp 
----------------------------------------------------------
  ...
        :                 bmx += square1[bcost&15][0];   <== source code
  1.43  : 4eb10d:  movsx  ecx,BYTE PTR [r8+rdx*2]        <== corresponding machine code
        :                 bmy += square1[bcost&15][1];
  0.36  : 4eb112:  movsx  r12d,BYTE PTR [r8+rdx*2+0x1]
        :                 bmx += square1[bcost&15][0];
  0.63  : 4eb118:  add    DWORD PTR [rsp+0x38],ecx
        :                 bmy += square1[bcost&15][1];
  ...
```

Most profilers with a Graphical User Interface (GUI), like Intel VTune Profiler, can show source code and associated assembly side-by-side. Also, there are tools that can visualize the output of Linux `perf` raw data with a rich graphical interface similar to Intel VTune and other tools. You'll see all that in more detail in [@sec:secOverviewPerfTools].

大多数带有图形用户界面(GUI: Graphical User Interface)的性能分析器，例如Intel VTune Profiler，都可以并排显示源代码和相关的汇编代码。此外，还有一些工具能够以类似于Intel VTune和其他工具的丰富图形界面可视化Linux `perf` 原始数据的输出。您可以在 [@sec:secOverviewPerfTools] 中看到更详细的介绍。

Sampling gives a good statistical representation of a program's execution, however, one of the downsides of this technique is that it has blind spots and is not suitable for detecting abnormal behaviors. Each sample represents an aggregated view of a portion of a program's execution. Aggregation doesn't give us enough details of what exactly happened during that time interval. We cannot zoom in to learn more about execution nuances. When we squash time intervals into samples, we lose valuable information and it becomes useless for analyzing events with a very short duration. For instance, profiling a program that reacts to network packets (such as stock trading software) may not be very informative as it will attribute most samples to the busy wait loop. Increasing the sampling interval, e.g., more than 1000 samples per second may give you a better picture, but may still not be enough. As a solution, you should use tracing as it doesn't skip events of interest.

采样可以很好地统计程序的执行情况，但这种技术的缺点之一是它存在盲点，不适用于检测异常行为。每个样本代表程序执行过程中一部分的聚合视图。聚合无法提供该时间段内究竟发生了什么的足够细节。我们无法放大以了解更多执行细节。当我们把时间间隔压缩成样本时，我们会丢失宝贵的信息，并且这种方法对于分析持续时间非常短的事件毫无用处。例如，对响应网络数据包的程序（例如：股票交易软件）进行性能分析可能意义不大，因为大部分采样点都会被归因于忙等待循环。增加采样间隔（例如：每秒超过1000个采样点）或许能提供更清晰的分析结果，但可能仍然不够。因此，建议使用跟踪功能，因为它不会遗漏任何感​​兴趣的事件。

### Collecting Call Stacks 收集调用堆栈 {#sec:secCollectCallStacks}

Often when sampling, we might encounter a situation when the hottest function in a program gets called from multiple functions. An example of such a scenario is shown in Figure @fig:CallStacks. The output from the profiling tool might reveal that `foo` is one of the hottest functions in the program, but if it has multiple callers, we would like to know which one of them calls `foo` the most number of times. It is a typical situation for applications that have library functions like `memcpy` or `sqrt` appear in the hotspots. To understand why a particular function appeared as a hotspot, we need to know which path in the Control Flow Graph (CFG) of the program is responsible for it.

在采样过程中，我们经常会遇到程序中最热门的函数被多个函数调用的情况。图 @fig:CallStacks 展示了一个这样的示例。性能分析工具的输出可能显示 `foo` 是程序中最热门的函数之一，但如果它被多个函数调用，我们想知道哪个函数调用 `foo` 的次数最多。对于包含 `memcpy` 或 `sqrt` 等库函数的应用程序来说，它们通常会作为热点函数出现。为了理解某个函数为何会成为热点函数，我们需要知道程序控制流图(CFG: Control Flow Graph)中的哪条路径导致了这一点。

![Control Flow Graph: hot function "foo" has multiple callers.控制流图：热点函数“foo”有多个调用者。](../../img/perf-analysis/CallStacksCFG.png){#fig:CallStacks width=70%}

Analyzing the source code of all the callers of `foo` might be very time-consuming. We want to focus only on those callers that caused `foo` to appear as a hotspot. In other words, we want to figure out the hottest path in the CFG of a program. Profiling tools achieve this by capturing the call stack of the process along with other information at the time of collecting performance samples. Then, all collected stacks are grouped, allowing us to see the hottest path that led to a particular function.

分析所有 `foo` 调用者的源代码可能非常耗时。我们只想关注那些导致 `foo` 成为热点函数的调用者。换句话说，我们希望找出程序CFG中最热门的路径。性能轮廓分析工具通过在收集性能样本时捕获进程的调用堆栈以及其他信息来实现这一点。然后，所有收集到的调用栈会被分组，从而让我们能够看到通往特定函数的最热路径。

Collecting call stacks in Linux `perf` is possible with three methods:

可以通过三种方法收集在Linux `perf` 中的调用堆栈：

1. Frame pointers (`perf record --call-graph fp`). It requires that the binary be built with `--fno-omit-frame-pointer`. Historically, the frame pointer (`RBP` register) was used for debugging since it enables us to get the call stack without popping all the arguments from the stack (also known as *stack unwinding*). The frame pointer can tell the return address immediately. It enables very cheap stack unwinding, which reduces profiling overhead, however, it consumes one additional register just for this purpose. At the time when the number of architectural registers was small, using frame pointers was expensive in terms of runtime performance. Nowadays, the Linux community is moving back to using frame pointers, because it provides better quality call stacks and low profiling overhead.
2. DWARF debug info (`perf record --call-graph dwarf`). It requires that the binary be built with DWARF debug information (`-g`). It also obtains call stacks through the stack unwinding procedure, but this method is more expensive than using frame pointers.
3. Intel Last Branch Record (LBR). This method makes use of a hardware feature, and is accessed with the following command: `perf record --call-graph lbr`. It obtains call stacks by parsing the LBR stack (a set of hardware registers). The resulting call graph is not as deep as those produced by the first two methods. See more information about the LBR call-stack mode in [@sec:lbr].

1. 帧指针（`perf record --call-graph fp`）。这要求二进制文件在构建时使用 `--fno-omit-frame-pointer` 参数。历史上，帧指针（`RBP` 寄存器）曾被用于调试，因为它允许我们在不弹出栈中所有参数（也称为*栈展开stack unwiding*）的情况下获取调用栈。帧指针可以立即指示返回地址。它能够实现非常高效的栈展开，从而降低性能分析的开销，但为此它会占用一个额外的寄存器。在架构寄存器数量较少的时代，使用帧指针会显著影响运行时性能。如今，Linux社区正在重新使用帧指针，因为它能够提供更高质量的调用栈并降低性能分析的开销。
2. DWARF调试信息（`perf record --call-graph dwarf`）。它要求二进制文件在构建时启用了DWARF调试信息（`-g`）。它通过栈展开过程获取调用栈，但这种方法比使用帧指针更耗时。
3. Intel的最后分支记录(LBR: Last Branch Record)。此方法利用了一个硬件特性，可通过以下命令访问：`perf record --call-graph lbr`。它通过解析LBR栈（一组硬件寄存器）来获取调用栈。生成的调用图不如前两种方法生成的调用图深。有关LBR调用栈模式的更多信息，请参阅 [@sec:lbr]。

Below is an example of collecting call stacks in a program using LBR. By looking at the output, we know that 55% of the time `foo` was called from `func1`, 33% of the time from `func2`, and 11% from `fun3`. We can clearly see the distribution of the overhead between callers of `foo` and can now focus our attention on the hottest edge in the CFG of the program, which is `func1` &rarr; `foo`, but we should probably also pay attention to the edge `func2` &rarr; `foo`.

以下示例展示了如何使用LBR收集程序中的调用栈。通过查看输出，我们知道 `foo` 函数有55%的概率是从 `func1` 调用的，33%的概率是从 `func2` 调用的，11%的概率是从 `func3` 调用的。我们可以清楚地看到 `foo` 的调用者之间的开销分布情况，现在可以将注意力集中在程 CFG中最热门的边，即 `func1` --> `foo`，但我们可能也应该注意 `func2` --> `foo`。

```bash
$ perf record --call-graph lbr -- ./a.out
$ perf report -n --stdio --no-children
# Samples: 65K of event 'cycles:ppp'
# Event count (approx.): 61363317007
# Overhead       Samples  Command  Shared Object     Symbol
# ........  ............  .......  ................  ......................
    99.96%         65217  a.out    a.out             [.] foo
            |
             --99.96%--foo
                       |
                       |--55.52%--func1
                       |          main
                       |          __libc_start_main
                       |          _start
                       |
                       |--33.32%--func2
                       |          main
                       |          __libc_start_main
                       |          _start
                       |
                        --11.12%--func3
                                  main
                                  __libc_start_main
                                  _start
```

When using Intel VTune Profiler, you can collect call stacks data by checking the corresponding "Collect stacks" box while configuring analysis. When using the command-line interface, specify the `-knob enable-stack-collection=true` option.

使用Intel VTune Profiler时，您可以在配置分析时勾选相应的“收集堆栈Collect stacks”复选框来收集调用堆栈数据。使用命令行界面时，请指定 `-knob enable-stack-collection=true` 选项。

[^1]: Profiling(wikipedia) 轮廓（维基百科）- [https://en.wikipedia.org/wiki/Profiling_(computer_programming)](https://en.wikipedia.org/wiki/Profiling_(computer_programming)).
[^7]: x264 benchmark x264基准测试 - [https://openbenchmarking.org/test/pts/x264](https://openbenchmarking.org/test/pts/x264).
[^8]: Phoronix test suite Phoronix测试集 - [https://www.phoronix-test-suite.com/](https://www.phoronix-test-suite.com/).
