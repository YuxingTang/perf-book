## Collecting Performance Monitoring Events 收集性能监控事件 {#sec:counting}

Performance Monitoring Counters (PMCs) are a very important instrument of low-level performance analysis. They can provide unique information about the execution of a program. PMCs are generally used in two modes: "Counting" or "Sampling". The counting mode is primarily used for calculating various performance metrics that we discussed in [@sec:PerfMetrics]. The sampling mode is used for finding hotspots, which we will discuss soon.

性能监控计数器(PMC: Performance Monitoring Counter)是底层性能分析中非常重要的工具。它们可以提供关于程序执行的独特信息。PMC通常以两种模式使用：“计数”或“采样”。计数模式主要用于计算我们在 [@sec:PerfMetrics] 中讨论的各种性能指标。采样模式用于查找性能热点，我们稍后会讨论。

The idea behind counting is very simple: we want to count the total number of certain performance monitoring events while our program is running. PMCs are heavily used in the Top-down Microarchitecture Analysis (TMA) methodology, which we will closely look at in [@sec:TMA]. Figure @fig:Counting illustrates the process of counting performance events from the start to the end of a program.

计数背后的思想非常简单：我们希望统计程序运行时特定性能监控事件的总数。PMC在自顶向下微体系结构分析(TMA: Top-down Microarchitecture Analysis)方法论中被广泛使用，我们将在 [@sec:TMA] 中详细介绍。图 @fig:Counting 展示了从程序开始到结束统计性能事件的过程。

![Counting performance events. 统计性能事件。](../../img/perf-analysis/CountingFlow.png){#fig:Counting width=80%}

The steps outlined in Figure @fig:Counting roughly represent what a typical analysis tool will do to count performance events. A similar process is implemented in the `perf stat` tool, which can be used to count various hardware events, like the number of instructions, cycles, cache misses, etc. Below is an example of the output from `perf stat`:

图 @fig:Counting 中概述的步骤大致展示了典型的分析工具统计性能事件的过程。 `perf stat` 工具实现了类似的过程，它可以用来统计各种硬件事件，例如指令数、周期数、缓存未命中数等。以下是 `perf stat` 的输出示例：

```bash
$ perf stat -- ./my_program.exe
 10580290629  cycles         #    3,677 GHz
  8067576938  instructions   #    0,76  insn per cycle
  3005772086  branches       # 1044,472 M/sec
   239298395  branch-misses  #    7,96% of all branches 
```

This data may become quite handy. First of all, it enables us to quickly spot some anomalies, such as a high branch misprediction rate or low IPC. In addition, it might come in handy when you've made a code change and you want to verify that the change has improved performance. Looking at relevant events might help you justify or reject the code change. The `perf stat` utility can be used as a lightweight benchmark wrapper. It may serve as a first step in performance investigation. Sometimes anomalies can be spotted right away, which can save you some analysis time.

这些数据非常有用。首先，它可以帮助我们快速发现一些异常情况，例如分支预测错误率过高或IPC过低。此外，当您修改代码并希望验证修改是否提升性能时，`perf stat` 工具也很有用。查看相关事件可以帮助您判断代码修改是否合理。`perf stat` 工具可以作为轻量级的基准测试封装器。它可以被用于性能调查的第一步。有时，异常情况可以立即被发现，从而节省您的分析时间。

A full list of available event names can be viewed with `perf list`:

你可以使用 `perf list` 查看所有可用事件名称的完整列表：

```bash
$ perf list
  cycles            [Hardware event]
  ref-cycles        [Hardware event]
  instructions      [Hardware event]
  branches          [Hardware event]
  branch-misses     [Hardware event]
  ...
cache:
  mem_load_retired.l1_hit
  mem_load_retired.l1_miss
  ...
```

Modern CPUs have hundreds of observable performance events. It's very hard to remember all of them and their meanings. Understanding when to use a particular event is even harder. That is why generally, I don't recommend manually collecting a specific event unless you really know what you are doing. Instead, I recommend using tools like Intel VTune Profiler that automatically collect required events to calculate various metrics.

现代CPU拥有数百个可观测的性能事件。要记住所有这些事件及其含义非常困难。理解何时使用特定事件则更加困难。因此，通常情况下，除非您非常清楚自己在做什么，否则我不建议手动收集特定事件。相反，我建议使用像 Intel VTune Profiler这样的工具，它们可以自动收集计算各种指标所需的事件。

Performance events are not available in every environment since accessing PMCs requires root access, which applications running in a virtualized environment typically do not have. For programs executing in a public cloud, running a PMU-based profiler directly in a guest container does not result in useful output if a virtual machine (VM) manager does not expose the PMU programming interfaces properly to a guest. Thus profilers based on CPU performance monitoring counters do not work well in a virtualized and cloud environment [@PMC_virtual], although the situation is improving. VMware® was one of the first VM managers to enable[^4] virtual Performance Monitoring Counters (vPMC). The AWS EC2 cloud has also enabled[^5] PMCs for dedicated hosts.

并非所有环境都支持性能事件，因为访问PMC需要root权限，而运行在虚拟化环境中的应用程序通常不具备此权限。对于在公有云中运行的程序，如果虚拟机(VM: Virtual Machine)管理器没有正确地向客户机公开PMU编程接口，则直接在客户机容器中运行基于PMU的分析器不会产生有用的输出。因此，基于CPU性能监控计数器的分析器在虚拟化和云环境中效果不佳，尽管这种情况正在改善。VMware®是最早启用虚拟性能监控计数器(vPMC: virtual Performance Monitoring Counters)的虚拟机管理器之一。 AWS EC2云平台也为专用主机启用了PMC[^5]。

### Multiplexing and Scaling Events 事件复用和扩展 {#sec:secMultiplex}

There are situations when we want to count many different events at the same time. However, with one counter, it's possible to count only one event at a time. That's why PMUs contain multiple counters (in Intel's recent Golden Cove microarchitecture there are 12 programmable PMCs, 6 per hardware thread). Even then, the number of fixed and programmable counters is not always sufficient. Top-down Microarchitecture Analysis (TMA) methodology requires collecting up to 100 different performance events in a single execution of a program. Modern CPUs don't have that many counters, and here is when multiplexing comes into play.

有时我们需要同时统计多个不同的事件。然而，单个计数器一次只能统计一个事件。因此，PMU包含多个计数器（例如，在Intel最新的Golden Cove微体系结构中，每个硬件线程有6个可编程PMC，共12个）。即便如此，固定计数器和可编程计数器的数量也并非总是足够。自顶向下的微体系结构分析(TMA)方法要求在程序的单次执行过程中收集多达100个不同的性能事件。现代CPU的计数器数量有限，这时就需要用到多路复用技术。

If you need to collect more events than the number of available PMCs, the analysis tool uses time multiplexing to give each event a chance to access the monitoring hardware. Figure @fig:Multiplexing1 shows an example of multiplexing between 8 performance events with only 4 counters available.

如果需要收集的事件数量超过可用PMC的数量，分析工具会使用时间复用技术，让每个事件都有机会访问监控硬件。图 @fig:Multiplexing1 显示了在只有4个计数器的情况下，8个性能事件之间进行多路复用的示例。

<div id="fig:Multiplexing">
![](../../img/perf-analysis/Multiplexing1.png){#fig:Multiplexing1 width=70%}

![](../../img/perf-analysis/Multiplexing2.png){#fig:Multiplexing2 width=80%}

Multiplexing between 8 performance events with only 4 PMCs available. 在只有4个性能计数器PMC可用时在8个性能事件中复用。
</div>

With multiplexing, an event is not measured all the time, but rather only during a portion of time. At the end of the run, a profiling tool needs to scale the raw count based on the total time enabled:
使用多路复用技术时，事件并非始终被测量，而仅在部分时间段内进行测量。运行结束后，性能分析工具需要根据总启用时间对原始计数进行缩放：
$$
final~count = raw~count \times ( time~running / time~enabled )
$$
$$
最终计数 = 原始计数 \times ( 运行时间 / 启用时间 )
$$
Let's take Figure @fig:Multiplexing2 as an example. Say, during profiling, we were able to measure an event from group 1 during three time intervals. Each measurement interval lasted 100ms (`time enabled`). The program running time was 500ms (`time running`). The total number of events for this counter was measured as 10,000 (`raw count`). So, the final count needs to be scaled as follows:
以图 @fig:Multiplexing2 为例。假设在性能分析过程中，我们能够在三个时间间隔内测量分组1中的事件。每个测量间隔持续100毫秒（“启用时间”）。程序运行时间为500毫秒（“运行时间”）。该计数器的事件总数测量值为 10,000（“原始计数”）。因此，最终计数需要按如下方式缩放：
$$
final~count = 10,000 \times ( 500ms / ( 100ms \times 3) ) = 16,666
$$
$$
最终计数 = 10,000 \times ( 500ms / ( 100ms \times 3) ) = 16,666
$$
This provides an estimate of what the count would have been had the event been measured during the entire run. It is very important to understand that this is still an estimate, not an actual count. Multiplexing and scaling can be used safely on steady workloads that execute the same code during long time intervals. However, if the program regularly jumps between different hotspots, i.e., has different phases, there will be blind spots that can introduce errors during scaling. To avoid scaling, you can reduce the number of events to no more than the number of physical PMCs available. However, you'll have to run the benchmark multiple times to measure all the events.

这提供了一个估计值，即如果在整个运行期间都测量了该事件，其计数应该是多少。务必理解，这仍然是一个估计值，而不是实际计数。对于在较长时间间隔内执行相同代码的稳定工作负载，可以安全地使用多路复用和缩放。但是，如果程序经常在不同的热点之间跳转（即具有不同的阶段），则会出现盲点，从而在缩放过程中引入误差。为了避免缩放，您可以将事件数量减少到不超过可用物理PMC的数量。但是，您需要多次运行基准测试才能测量所有事件。

[^4]: VMware PMCs - [https://www.vladan.fr/what-are-vmware-virtual-cpu-performance-monitoring-counters-vpmcs/](https://www.vladan.fr/what-are-vmware-virtual-cpu-performance-monitoring-counters-vpmcs/)
[^5]: Amazon EC2 PMCs - [http://www.brendangregg.com/blog/2017-05-04/the-pmcs-of-ec2.html](http://www.brendangregg.com/blog/2017-05-04/the-pmcs-of-ec2.html)
