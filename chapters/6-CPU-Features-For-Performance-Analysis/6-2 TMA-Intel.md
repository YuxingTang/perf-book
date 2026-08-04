### TMA on Intel Platforms 在Intel平台上的TMA{#sec:secTMA_Intel}

The TMA methodology was first proposed by Intel in 2014 and is supported starting from the Sandy Bridge family of processors. Intel's implementation supports nested categories for each high-level bucket that give a better understanding of the CPU performance bottlenecks in the program (see Figure @fig:TMA).

TMA 方法论最初由英特尔于 2014 年提出，并从 Sandy Bridge 系列处理器开始支持。英特尔的实现支持每个高级分类下嵌套的子类别，从而更好地了解程序中的 CPU 性能瓶颈（参见图 @fig:TMA）。

The workflow is designed to "drill down" to lower levels of the TMA hierarchy until we get to the very specific classification of a performance bottleneck. First, we collect metrics for the main four buckets: `Frontend Bound`, `Backend Bound`, `Retiring`, and `Bad Speculation`. Say, we found out that a big portion of the program execution was stalled by memory accesses (which is a `Backend Bound` bucket, see Figure @fig:TMA). The next step is to run the workload again and collect metrics specific to the `Memory Bound` bucket only. The process is repeated until we know the exact root cause, for example, `L3 Bound`.

该工作流程旨在“向下钻取”到 TMA 层次结构的更低层级，直到找到性能瓶颈的具体分类。首先，我们收集四个主要分类的指标：`前端瓶颈`、`后端瓶颈`、`内存访问终止`和`错误推测`。例如，如果我们发现程序执行的大部分时间都因内存访问而停滞（这属于 `后端瓶颈` 分类，参见图 @fig:TMA），下一步是再次运行工作负载，并仅收集 `内存瓶颈` 分类的指标。这个过程会重复进行，直到我们找到确切的根本原因，例如“L3 瓶颈”。

![The TMA hierarchy of performance bottlenecks. *© Image by Ahmad Yasin.* TMA 性能瓶颈层级结构。*© 图片由 Ahmad Yasin 提供。*](../../img/pmu-features/TMAM.png){#fig:TMA width=90%}

It is fine to run the workload several times, each time drilling down and focusing on specific metrics. But usually, it is sufficient to run the workload once and collect all the metrics required for all levels of TMA. Profiling tools achieve that by multiplexing (see [@sec:secMultiplex]) between different performance events during a single run. Also, in a real-world application, performance could be limited by several factors. For example, it can experience a large number of branch mispredictions (`Bad Speculation`) and cache misses (`Backend Bound`) at the same time. In this case, TMA will drill down into multiple buckets simultaneously and will identify the impact that each type of bottleneck makes on the performance of a program. Analysis tools such as Intel's VTune Profiler, AMD's uProf, and Linux `perf` can calculate all the TMA metrics with a single benchmark run. However, this only is acceptable if the workload is steady. Otherwise, you would better fall back to the original strategy of multiple runs and drilling down with each run.

多次运行工作负载，每次都深入分析并关注特定指标，这当然可以。但通常情况下，只需运行一次工作负载，收集 TMA 所有层级所需的所有指标即可。性能分析工具通过在单次运行期间对不同的性能事件进行多路复用（参见 [@sec:secMultiplex]）来实现这一点。此外，在实际应用中，性能可能受到多种因素的限制。例如，它可能同时经历大量的分支预测错误（“错误推测”）和缓存未命中（“后端瓶颈”）。在这种情况下，TMA 会同时深入分析多个瓶颈，并识别每种瓶颈类型对程序性能的影响。诸如 Intel 的 VTune Profiler、AMD 的 uProf 和 Linux 的 `perf` 等分析工具可以通过一次基准测试运行计算所有 TMA 指标。然而，这仅适用于工作负载稳定的情况。否则，最好还是采用多次运行并逐次深入分析的原始策略。

The top two levels of TMA metrics are expressed in the percentage of all pipeline slots (see [@sec:PipelineSlot]) that were available during the execution of a program. It allows TMA to give an accurate representation of CPU microarchitecture utilization, taking into account the full bandwidth of a processor. Up to this point, everything should sum up nicely to 100%. However, starting from Level 3, buckets may be expressed in a different count domain, e.g., clocks and stalls. So they are not necessarily directly comparable with other TMA buckets.

TMA 指标的前两级以程序执行期间所有可用流水线槽（参见 [@sec:PipelineSlot]）的百分比表示。这使得 TMA 能够准确地表示 CPU 微架构的利用率，并考虑处理器的全部带宽。到目前为止，所有指标的总和应该正好是 100%。但是，从第三级开始，瓶颈的计数范围可能不同，例如时钟频率和停顿次数。因此，它们不一定能与其他 TMA 指标直接比较。

Once we identify the performance bottleneck, we need to know where exactly in the code it is happening. The second step in TMA is to locate the source of the problem down to the exact line of code and corresponding assembly instructions. The analysis methodology provides performance events that you should use for each category of the performance problem. Then you can sample on this event to find the line in the source code that contributes to the performance bottleneck identified by the first stage. Don't worry if this process sounds complicated to you; everything will become clear once you read through the case study.

一旦我们确定了性能瓶颈，就需要知道它具体发生在代码的哪个位置。TMA 的第二步是将问题根源精确定位到代码行及其对应的汇编指令。该分析方法提供了针对每种性能问题类别应使用的性能事件。然后，您可以对这些事件进行采样，以找到导致第一阶段所识别的性能瓶颈的源代码行。如果您觉得这个过程很复杂，请不要担心；阅读完案例研究后，一切都会变得清晰明了。

### Case Study: Reduce The Number of Cache Misses with TMA 案例研究：使用TMA减少缓存未命中次数 {.unlisted .unnumbered}

As an example for this case study, I took a very simple benchmark, such that it is easy to understand and change. It is not representative of real-world applications, but it is good enough to demonstrate the workflow of TMA. I have a lot more practical examples in the second part of the book.

本案例研究选取了一个非常简单的基准测试用例，以便于理解和修改。它虽然不能代表实际应用，但足以演示 TMA 的工作流程。本书第二部分提供了更多实际应用示例。

Most readers of this book will likely apply TMA to their own applications, with which they are familiar. But TMA is very effective even if you see the application for the first time. For this reason, I don't start by showing you the source code of the benchmark. But here is a short description: the benchmark allocates a 200 MB array on the heap, then enters a loop of 100M iterations. On every iteration of the loop, it generates a random index into the allocated array, performs some dummy work, and then reads the value from that index.

本书的大多数读者可能会将 TMA 应用于他们熟悉的应用程序。但即使您是第一次接触这类应用程序，TMA 也非常有效。因此，我不会一开始就展示基准测试用例的源代码。这里简单介绍一下：该基准测试用例在堆上分配一个 200 MB 的数组，然后进入一个 1 亿次迭代的循环。在每次循环迭代中，它会生成一个指向已分配数组的随机索引，执行一些虚拟操作，然后从该索引读取值。

I ran the experiments on the machine equipped with an Intel Core i5-8259U CPU (Skylake-based) and 16GB of DRAM (DDR4 2400 MT/s), running 64-bit Ubuntu 20.04 (kernel version 5.13.0-27).

我在一台配备 Intel Core i5-8259U CPU（基于 Skylake 架构）和 16GB DRAM（DDR4 2400 MT/s）的机器上运行了实验，该机器运行的是 64 位 Ubuntu 20.04 操作系统（内核版本 5.13.0-27）。

### Step 1: Identify the Bottleneck 步骤1：识别瓶颈 {.unlisted .unnumbered}

As a first step, we run our microbenchmark and collect a limited set of events that will help us calculate Level 1 metrics. Here, we try to identify high-level performance bottlenecks of our application by attributing them to the four L1 buckets: `Frontend Bound`, `Backend Bound`, `Retiring`, and `Bad Speculation`. It is possible to collect Level 1 metrics using the Linux `perf` tool. The `perf stat` command has a dedicated `--topdown` option. In more recent versions it will output these metrics by default. Below is the breakdown for our benchmark. The output of all commands in this section is trimmed to save space.

首先，我们运行微基准测试并收集一组有限的事件，以帮助我们计算一级指标。在这里，我们尝试通过将应用程序的性能瓶颈归类到四个一级类别来识别它们：`前端瓶颈Frontedn Bound`、`后端瓶颈Backend Bound`、`退役Retiring`和`错误推测Bad Speculation`。可以使用 Linux 的 `perf` 工具收集一级指标。`perf stat` 命令有一个专门的 `--topdown` 选项。在较新的版本中，它会默认输出这些指标。以下是我们的基准测试结果明细。为了节省空间，本节中所有命令的输出结果均已精简。

```bash
$ perf stat -- ./benchmark.exe
...
  TopdownL1 (cpu_core)  #  53.4 %  tma_backend_bound    <==
                        #   0.2 %  tma_bad_speculation
                        #  13.8 %  tma_frontend_bound
                        #  32.5 %  tma_retiring
...
```

By looking at the output, we can tell that the performance of the application is bound by the CPU backend. Let's drill one level down. To obtain TMA metrics Level 2, 3, and further, I will use the `toplev` tool that is a part of [pmu-tools](https://github.com/andikleen/pmu-tools)[^7] written by Andi Kleen. It is implemented in `Python` and uses Linux `perf` under the hood. Specific Linux kernel settings must be enabled to use `toplev`; check the documentation for more details.

通过查看输出结果，我们可以判断应用程序的性能瓶颈在于 CPU 后端。让我们进一步分析。为了获取 TMA 指标的第 2 级、第 3 级及更高级别的数据，我将使用 `toplev` 工具，它是 Andi Kleen 编写的 [pmu-tools](https://github.com/andikleen/pmu-tools)[^7] 的一部分。该工具使用 Python 编写，底层基于 Linux 内核的 `perf` 工具。使用 `toplev` 需要启用特定的 Linux 内核设置；更多详情请参阅相关文档。

```bash
$ ~/pmu-tools/toplev.py --core S0-C0 -l2 -v --no-desc taskset -c 0 ./benchmark.exe
...
# Level 1
S0-C0  Frontend_Bound:                13.92 % Slots
S0-C0  Bad_Speculation:                0.23 % Slots
S0-C0  Backend_Bound:                 53.39 % Slots
S0-C0  Retiring:                      32.49 % Slots
# Level 2
S0-C0  Frontend_Bound.FE_Latency:     12.11 % Slots
S0-C0  Frontend_Bound.FE_Bandwidth:    1.84 % Slots
S0-C0  Bad_Speculation.Branch_Mispred: 0.22 % Slots
S0-C0  Bad_Speculation.Machine_Clears: 0.01 % Slots
S0-C0  Backend_Bound.Memory_Bound:    44.59 % Slots <==
S0-C0  Backend_Bound.Core_Bound:       8.80 % Slots
S0-C0  Retiring.Base:                 24.83 % Slots
S0-C0  Retiring.Microcode_Sequencer:   7.65 % Slots
```

In this command, we pinned the process to CPU0 (using `taskset -c 0`) and limited the output of `toplev` to this core only (`--core S0-C0`). The option `-l2` tells the tool to collect Level 2 metrics. The option `--no-desc` disables the description of each metric.

在此命令中，我们将进程锁定到 CPU0（使用 `taskset -c 0`），并将 `toplev` 的输出限制在该核心上（`--core S0-C0`）。`-l2` 选项指示工具收集二级指标。`--no-desc` 选项禁用每个指标的描述。

We can see that the application’s performance is bound by memory accesses (`Backend_Bound.Memory_Bound`). Almost half of the CPU execution resources were wasted waiting for memory requests to complete. Now let us dig one level deeper: [^17]

我们可以看到，应用程序的性能受限于内存访问（`Backend_Bound.Memory_Bound`）。几乎一半的 CPU 执行资源都浪费在等待内存请求完成上。现在让我们深入分析一下：[^17]

```bash
$ ~/pmu-tools/toplev.py --core S0-C0 -l3 -v --no-desc taskset -c 0 ./benchmark.exe
...
# Level 1
S0-C0    Frontend_Bound:                 13.91 % Slots
S0-C0    Bad_Speculation:                 0.24 % Slots
S0-C0    Backend_Bound:                  53.36 % Slots
S0-C0    Retiring:                       32.41 % Slots
# Level 2
S0-C0    FE_Bound.FE_Latency:            12.10 % Slots
S0-C0    FE_Bound.FE_Bandwidth:           1.85 % Slots
S0-C0    BE_Bound.Memory_Bound:          44.58 % Slots
S0-C0    BE_Bound.Core_Bound:             8.78 % Slots
# Level 3
S0-C0-T0 BE_Bound.Mem_Bound.L1_Bound:     4.39 % Stalls
S0-C0-T0 BE_Bound.Mem_Bound.L2_Bound:     2.42 % Stalls
S0-C0-T0 BE_Bound.Mem_Bound.L3_Bound:     5.75 % Stalls
S0-C0-T0 BE_Bound.Mem_Bound.DRAM_Bound:  47.11 % Stalls <==
S0-C0-T0 BE_Bound.Mem_Bound.Store_Bound:  0.69 % Stalls
S0-C0-T0 BE_Bound.Core_Bound.Divider:     8.56 % Clocks
S0-C0-T0 BE_Bound.Core_Bound.Ports_Util: 11.31 % Clocks
```

We found the bottleneck to be in `DRAM_Bound`. This tells us that many memory accesses miss in all levels of caches and go all the way down to the main memory. We can also confirm this if we collect the absolute number of L3 cache misses for the program. For the Skylake architecture, the `DRAM_Bound` metric is calculated using the `CYCLE_ACTIVITY.STALLS_L3_MISS` performance event. Let’s collect it manually:

我们发现瓶颈在于 `DRAM_Bound`。这表明许多内存访问在所有级别的缓存中都未命中，甚至一直到主内存。如果我们收集程序的 L3 缓存未命中绝对次数，也可以证实这一点。对于 Skylake 架构，`DRAM_Bound` 指标是使用 `CYCLE_ACTIVITY.STALLS_L3_MISS` 性能事件计算的。让我们手动收集一下：

```bash
$ perf stat -e cycles,cycle_activity.stalls_l3_miss -- ./benchmark.exe
  32226253316  cycles
  19764641315  cycle_activity.stalls_l3_miss
```

The `CYCLE_ACTIVITY.STALLS_L3_MISS` event counts cycles when execution stalls, while the L3 cache miss demand load is outstanding. We can see that there are ~60% of such cycles, which is pretty bad.

`CYCLE_ACTIVITY.STALLS_L3_MISS` 事件统计的是当 L3 缓存未命中需求负载未解决时，程序执行停滞的周期数。我们可以看到，此类周期数约占 60%，情况相当糟糕。

### Step 2: Locate the Place in the Code 步骤2：定位代码中的位置 {.unlisted .unnumbered}

The second step in the TMA process is to locate the place in the code where the identified performance event occurs most frequently. To do so, you should sample the workload using an event that corresponds to the type of bottleneck that was identified during Step 1.

TMA流程的第二步是定位代码中出现频率最高的性能事件的位置。为此，您应该使用与步骤 1 中识别出的瓶颈类型相对应的事件对工作负载进行采样。

A recommended way to find such an event is to run `toplev` tool with the `--show-sample` option that will suggest the `perf record` command line that can be used to locate the issue. To understand the mechanics of TMA, we also present the manual way to find an event associated with a particular performance bottleneck. Correspondence between performance bottlenecks and performance events that should be used for determining the location of bottlenecks in source code can be found with the help of the [TMA metrics](https://github.com/intel/perfmon/blob/main/TMA_Metrics.xlsx)[^2] table. The `Locate-with` column denotes a performance event that should be used to locate the exact place in the code where the issue occurs. In our case, to find memory accesses that contribute to such a high value of the `DRAM_Bound` metric (miss in the L3 cache), we should sample on `MEM_LOAD_RETIRED.L3_MISS_PS` precise event. Here is the example command:

推荐的查找此类事件的方法是运行带有 `--show-sample` 选项的 `toplev` 工具，该选项会显示可用于定位问题的 `perf record` 命令行。为了帮助您理解 TMA 的机制，我们还介绍了手动查找与特定性能瓶颈相关的事件的方法。性能瓶颈与性能事件之间的对应关系可以通过 [TMA 指标](https://github.com/intel/perfmon/blob/main/TMA_Metrics.xlsx)[^2] 表来确定源代码中瓶颈的位置。`Locate-with` 列表示一个性能事件，该事件用于定位代码中问题发生的确切位置。在本例中，为了找到导致 `DRAM_Bound` 指标值过高（L3 缓存未命中）的内存访问，我们应该对 `MEM_LOAD_RETIRED.L3_MISS_PS` 精确事件进行采样。以下是一个示例命令：

```bash
$ perf record -e cpu/event=0xd1,umask=0x20,name=MEM_LOAD_RETIRED.L3_MISS/ppp -- ./benchmark.exe
$ perf report -n --stdio
...
# Samples: 33K of event ‘MEM_LOAD_RETIRED.L3_MISS’
# Event count (approx.): 71363893
# Overhead   Samples  Shared Object   Symbol
# ........  ......... ..............  .................
#
    99.95%    33811   benchmark.exe   [.] foo
     0.03%       52   [kernel]        [k] get_page_from_freelist
     0.01%        3   [kernel]        [k] free_pages_prepare
     0.00%        1   [kernel]        [k] free_pcppages_bulk
```

Almost all L3 misses are caused by memory accesses in function `foo` inside executable `benchmark.exe`. Now it's time to look at the source code of the benchmark, which can be found on [GitHub](https://github.com/dendibakh/dendibakh.github.io/tree/master/_posts/code/TMAM).[^8]

几乎所有 L3 缓存未命中都是由可执行文件 `benchmark.exe` 中的函数 `foo` 的内存访问引起的。现在让我们来看看基准测试的源代码，它可以在 [GitHub](https://github.com/dendibakh/dendibakh.github.io/tree/master/_posts/code/TMAM)[^8] 上找到。

To avoid compiler optimizations, function `foo` is implemented in assembly language, which is presented in [@lst:TMA_asm]. The “driver” portion of the benchmark is implemented in the `main` function, as shown in [@lst:TMA_cpp]. We allocate a big enough array `a` to make it not fit in the 6MB L3 cache. The benchmark generates a random index into array `a` and passes this index to the `foo` function along with the address of array `a`. Later the `foo` function reads this random memory location.[^11]

为了避免编译器优化，函数 `foo` 是用汇编语言实现的，如 [@lst:TMA_asm] 所示。基准测试的“驱动”部分在 `main` 函数中实现，如 [@lst:TMA_cpp] 所示。我们分配了一个足够大的数组 `a`，使其无法放入 6MB 的 L3 缓存中。基准测试会生成数组 `a` 的一个随机索引，并将该索引连同数组 `a` 的地址一起传递给 `foo` 函数。之后，`foo` 函数会读取这个随机内存位置[^11]。

Listing: Assembly code of function foo.
代码列表：foo函数的汇编代码

~~~~ {#lst:TMA_asm .bash}
$ perf annotate --stdio -M intel foo
Percent |  Disassembly of benchmark.exe for MEM_LOAD_RETIRED.L3_MISS
------------------------------------------------------------
        :  Disassembly of section .text:
        :
        :  0000000000400a00 <foo>:
        :  foo():
   0.00 :    400a00:  nop  DWORD PTR [rax+rax*1+0x0]
   0.00 :    400a08:  nop  DWORD PTR [rax+rax*1+0x0]
                 ...  # more NOPs
 100.00 :    400e07:  mov  rax,QWORD PTR [rdi+rsi*1] <==
                 ...
   0.00 :    400e13:  xor  rax,rax
   0.00 :    400e16:  ret
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Listing: Source code of function main.
代码列表：main函数的源代码

~~~~ {#lst:TMA_cpp .cpp}
extern "C" { void foo(char* a, int n); }
const int _200MB = 1024*1024*200;
int main() {
  char* a = new char[_200MB]; // 200 MB buffer
  ...
  for (int i = 0; i < 100000000; i++) {
    int random_int = distribution(generator);
    foo(a, random_int);
  }
  ...
}
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

By looking at [@lst:TMA_asm], we can see that all L3 cache misses in function `foo` are tagged to a single instruction. Now that we know which instruction caused so many L3 misses, let’s fix it.

通过查看 [@lst:TMA_asm]，我们可以看到函数 `foo` 中所有 L3 缓存未命中都与同一条指令相关。既然我们已经知道是哪条指令导致了这么多 L3 缓存未命中，接下来就来修复它。

### Step 3: Fix the Issue 骤3：修复问题 {.unlisted .unnumbered}

There is dummy work emulated with NOPs at the beginning of the `foo` function. This creates a time window between the moment when we get the next address that will be accessed and the actual load instruction. The presence of the time window allows us to start prefetching the memory location in parallel with the dummy work. [@lst:TMA_prefetch] shows this idea in action. More information about the explicit memory prefetching technique can be found in [@sec:memPrefetch].

在 `foo` 函数的开头，有一些用 NOP 指令模拟的虚拟操作。这会在获取下一个要访问的地址和实际的加载指令之间创建一个时间窗口。这个时间窗口允许我们在执行虚拟操作的同时开始预取内存位置。[@lst:TMA_prefetch] 展示了这一思路的实际应用。关于显式内存预取技术的更多信息，请参阅 [@sec:memPrefetch]。

Listing: Inserting memory prefetch into main.
代码列表：在mian函数中插入存储预取

~~~~ {#lst:TMA_prefetch .cpp}
  for (int i = 0; i < 100000000; i++) {
    int random_int = distribution(generator);
+   __builtin_prefetch ( a + random_int, 0, 1);
    foo(a, random_int);
  }
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

This explicit memory prefetching hint decreases execution time from 8.5 seconds to 6.5 seconds. Also, the number of `CYCLE_ACTIVITY.STALLS_L3_MISS` events becomes almost ten times less: it goes from 19 billion down to 2 billion.

显式内存预取提示将执行时间从 8.5 秒缩短至 6.5 秒。此外，`CYCLE_ACTIVITY.STALLS_L3_MISS` 事件的数量也减少了近十倍：从 190 亿次降至 20 亿次。

TMA is an iterative process, so once we fix one problem, we need to repeat the process starting from Step 1. Likely it will move the bottleneck into another bucket, in this case, `Retiring`. This was an easy example demonstrating the workflow of TMA methodology. Analyzing real-world applications is unlikely to be that easy. Chapters in the second part of the book are organized to make them convenient for use with the TMA process. In particular, Chapter 8 covers the `Memory Bound` category, Chapter 9 covers `Core Bound`, Chapter 10 covers `Bad Speculation`, and Chapter 11 covers `Frontend Bound`. Such a structure is intended to form a checklist that you can use to drive code changes when you encounter a certain performance bottleneck.

TMA 是一个迭代过程，因此一旦我们解决了一个问题，就需要从步骤 1 开始重复该过程。这很可能会将瓶颈转移到另一个类别，在本例中为“退役”。这是一个演示 TMA 方法工作流程的简单示例。分析实际应用程序不太可能如此简单。本书第二部分的章节组织方式旨在方便与 TMA 流程配合使用。具体而言，第 8 章涵盖“内存密集型”类别，第 9 章涵盖“核心密集型”类别，第 10 章涵盖“错误推测”，第 11 章涵盖“前端密集型”类别。这种结构旨在形成一个清单，您可以在遇到特定性能瓶颈时使用该清单来指导代码更改。

[^2]: TMA metrics TMA指标 - [https://github.com/intel/perfmon/blob/main/TMA_Metrics.xlsx](https://github.com/intel/perfmon/blob/main/TMA_Metrics.xlsx).
[^7]: PMU tools PMU工具 - [https://github.com/andikleen/pmu-tools](https://github.com/andikleen/pmu-tools).
[^8]: Case study example 案例研究示例 - [https://github.com/dendibakh/dendibakh.github.io/tree/master/_posts/code/TMAM](https://github.com/dendibakh/dendibakh.github.io/tree/master/_posts/code/TMAM).
[^11]: According to x86 Linux calling conventions ([https://en.wikipedia.org/wiki/X86_calling_conventions](https://en.wikipedia.org/wiki/X86_calling_conventions)), the first 2 arguments land in `rdi` and `rsi` registers respectively. 根据x86 Linux调用约定（[https://en.wikipedia.org/wiki/X86_calling_conventions](https://en.wikipedia.org/wiki/X86_calling_conventions)），前两个参数分别写入 `rdi` 和 `rsi` 寄存器。
[^17]: Alternatively, we could use the `-l2 --nodes L1_Bound,L2_Bound,L3_Bound,DRAM_Bound,Store_Bound` option instead of `-l3` to limit the collection since we know the application is bound by memory. 或者，我们可以使用 `-l2 --nodes L1_Bound,L2_Bound,L3_Bound,DRAM_Bound,Store_Bound` 选项代替 `-l3` 来限制收集范围，因为我们知道应用程序受限于内存。
