## Continuous Profiling 持续轮廓性能分析 {#sec:ContinuousProfiling}

In [@sec:sec_PerfApproaches], we covered the various approaches available for conducting performance analysis, including but not limited to instrumentation, tracing, and sampling. Among these three approaches, sampling imposes relatively minor runtime overhead and requires the least amount of upfront work while still offering valuable insight into application hotspots. However, this insight is limited to the specific point in time when the samples are gathered. What if we could add a time dimension to this sampling? Instead of knowing that FunctionA consumes 30% of CPU cycles at one particular point in time, what if we could track changes in FunctionA’s CPU usage over days, weeks, or months? Or detect changes in its stack trace over that same timespan, all in production? Continuous Profiling has emerged to turn these goals into reality.

在 [@sec:sec_PerfApproaches] 中，我们介绍了各种可用于性能分析的方法，包括但不限于插桩instrumentation、跟踪tracing和采样sampling。在这三种方法中，采样带来的运行时开销相对较小，前期工作量也最少，同时还能提供关于应用程序性能瓶颈的宝贵信息。然而，这种信息仅限于样本采集时的特定时间点。如果我们能为采样添加时间维度呢？与其仅仅知道FunctionA在某个特定时间点消耗了30%的CPU周期，不如跟踪FunctionA在几天、几周甚至几个月内的CPU使用率变化？或者在同一时间段内检测其堆栈踪迹的变化，所有这些都在生产环境中实现？持续性能轮廓分析的出现正是为了将这些目标变为现实。

Continuous Profiling (CP) is a systemwide, sample-based profiler that is always on, albeit at a low sample rate to minimize runtime impact. Continuously collecting data from all processes facilitates analysis of why the execution of code was different at different times and also aids debugging of incidents even after they have happened. CP tools provide valuable insights into which code uses the most resources, and this helps engineers reduce resource usage in their production environments and thus save money. Unlike typical profilers like Linux perf or Intel VTune, CP can pinpoint a performance issue from the application stack down to the kernel stack from *any* given date and time and supports call stack comparisons between any two arbitrary dates/times to highlight performance differences.

持续轮廓性能分析(CP: Continuous Profiling)是一种系统级的、基于样本的轮廓性能分析器，它始终处于运行状态，但采样率较低，以最大限度地减少对运行时的影响。持续收集所有进程的数据有助于分析代码执行在不同时间点出现差异的原因，并有助于在问题发生后进行调试。CP工具能够提供关于哪些代码占用资源最多的宝贵信息，这有助于工程师在生产环境中降低资源消耗，从而节省成本。与Linux perf或Intel VTune等传统性能轮廓分析器不同，CP可以从*任意*日期和时间点，精确定位从应用程序堆栈到内核堆栈的性能问题，并支持比较任意两个日期/时间点之间的调用堆栈，以突出显示性能差异。

![Screenshot of the Parca Continuous Profiler Web UI. Parca连续轮廓性能分析器Web用户界面UI截图。](../../img/perf-tools/Continuous_profiling.png){#fig:Continuous_profiling width=100%}

To showcase the look-and-feel of a typical CP tool, let’s look at the Web UI of [Parca](https://github.com/parca-dev/parca),[^1] one of the open-source CP tools, depicted in Figure @fig:Continuous_profiling. The top panel displays a time series graph of the number of CPU samples gathered from various processes on the machine during the period selected from the time window dropdown list, which in this case is "Last 15 minutes". However, to make it fit on the page, the image was cut to show only the last 10 minutes. 

为了展示典型的连续性能分析工具的外观和操作方式，我们来看一下开源连续性能分析工具之一 [Parca](https://github.com/parca-dev/parca)[^1] 的Web用户界面UI，如图 @fig:Continuous_profiling 所示。顶部面板显示了从机器上各个进程收集的CPU样本数量的时间序列图，时间段可从时间窗口下拉列表中选择，在本例中为“最近15分钟”。但是，为了适应页面大小，图像被裁剪，仅显示了最近10分钟的数据。

By default, Parca collects 19 samples per second. For each sample, it collects stack traces from all the processes that run on the host system. The more samples are attributed to a certain process, the more CPU activity it had during a period of time. In our example, you can see the hottest process (top line) had a bursty behavior with spikes and dips in CPU activity. If you were the lead developer behind this application you would probably be curious why this happens. When you roll out a new version of your application and suddenly see an unexpected spike in the CPU samples attributed to the process, that is an indication that something is going wrong.

默认情况下，Parca每秒收集19个样本。对于每个样本，它会收集主机系统上运行的所有进程的堆栈跟踪信息。某个进程的采样次数越多，说明它在一段时间内的CPU活动就越高。在我们的示例中，可以看到最活跃的进程（顶部线）的CPU活动呈现突发性波动，出现峰值和低谷。如果您是该应用程序的首席开发人员，您可能会想知道为什么会发生这种情况。当您发布应用程序的新版本，并突然发现某个进程的CPU采样次数出现意外峰值时，这表明某些环节出了问题。

Continuous profiling tools make it easier not only to spot the point in time when performance change occurred but also to determine the root cause of the issue. Once you click on any point of interest on the chart, the tool displays an icicle graph associated with that period in the bottom panel. An icicle graph is the upside-down version of a flame graph. Using it, you can compare call stacks before and after to help you find what is causing performance problems.

持续性能分析工具不仅可以帮助您轻松找到性能变化发生的时间点，还可以帮助您确定问题的根本原因。单击图表上的任何感兴趣点后，该工具会在底部面板中显示与该时间段相关的冰柱图。冰柱图是火焰图的倒置版本。使用冰柱图，您可以比较前后调用堆栈，从而帮助您找到导致性能问题的原因。

Imagine, you merged a code change into production and after it has been running for a while, you receive reports of intermittent response time spikes. These may or may not correlate with user traffic or with any particular time of day. This is an area where CP shines. You can pull up the CP Web UI and do a search for stack traces at the dates and times of those response time spikes, and then compare them to stack traces of other dates and times to identify anomalous executions at the application and/or kernel stack level. This type of “visual diff” is supported directly in the UI, like a graphical “perf diff” or a differential flamegraph.[^2]

想象一下，您将一段代码更改合并到生产环境中，运行一段时间后，您收到有关间歇性响应时间峰值的报告。这些峰值可能与用户流量或特定时间段相关，也可能不相关。而这正是性能轮廓分析(CP)的优势所在。您可以打开CP的 Web UI，搜索响应时间峰值出现日期和时间的堆栈跟踪，然后将其与其他日期和时间的堆栈跟踪行比较，从而识别应用程序和/或内核堆栈级别的异常执行。这种“可视化差异”在用户界面UI中得到直接支持，类似于图形化的“性能差异”或差异火焰图[^2]。

Google introduced the CP concept in the 2010 paper “Google-Wide Profiling” [@GoogleWideProfiling], which championed the value of always-on profiling in production environments. However, it took nearly a decade before it gained traction in the industry:

谷歌在2010年发表的论文《Google-Wide Profiling》[@GoogleWideProfiling] 中引入了CP的概念，该论文倡导在生产环境中持续进行性能分析的价值。然而，CP花了近十年时间才在业界获得广泛认可：

1. In March 2019, Google Cloud released its Continuous Profiler.
2. In July 2020, AWS released CodeGuru Profiler.
3. In August 2020, Datadog released its Continuous Profiler.
4. In December 2020, New Relic acquired the Pixie Continuous Profiler.
5. In Jan 2021, Pyroscope released its open-source Continuous Profiler.
6. In October 2021, Elastic acquired Optimyze and its Continuous Profiler (Prodfiler); Polar Signals released its Parca Continuous Profiler. It was open-source in April 2024.
7. In December 2021, Splunk released its AlwaysOn Profiler.
8. In March 2022, Intel acquired Granulate and its Continuous Profiler (gProfiler). It was made open-source in March 2024.

1. 2019年3月，谷歌云发布了其持续性能分析器（Continuous Profiler）。
2. 2020年7月，AWS发布了CodeGuru性能分析器。
3. 2020年8月，Datadog发布了其持续性能分析器。
4. 2020年12月，New Relic收购了Pixie持续性能分析器。
5. 2021年1月，Pyroscope发布了其开源的持续性能分析器。
6. 2021年10月，Elastic收购了Optimyze及其持续性能分析器（Prodfiler）；Polar Signals发布了其Parca持续性能分析器，并于2024年4月开源。
7. 2021年12月，Splunk发布了其AlwaysOn性能分析器。
8. 2022年3月，Intel收购了Granulate及其持续性能分析器（gProfiler）。它于2024年3月开源。

New entrants into this space continue to pop up in both open-source and commercial varieties. Some of these offerings require more hand-holding than others. For example, some require source code or configuration file changes to begin profiling. Others require different agents for different language runtimes (e.g., Ruby, Python, Golang, C/C++/Rust). The best of them have crafted a secret sauce around eBPF so that nothing other than simply installing the runtime agent is necessary.

开源和商业领域的新产品层出不穷。其中一些产品需要更多的手把手指导。例如，有些需要修改源代码或配置文件才能开始性能分析。另一些则需要针对不同的语言运行时（例如：Ruby、Python、Golang、C/C++/Rust）使用不同的代理。最好的产品围绕eBPF精心打造了一套独特的机制，只需安装运行时代理即可。

They also differ in the number of language runtimes supported, the work required for obtaining debug symbols for readable stack traces, and the type of system resources that can be profiled aside from the CPU (e.g., memory, I/O, or locking). While Continuous Profilers differ in the aforementioned aspects, they all share the common function of providing low-overhead, sample-based profiling for various language runtimes, along with remote stack trace storage for web-based search and query capability.

它们在支持的语言运行时数量、获取可读堆栈跟踪的调试符号所需的工作量以及除CPU之外可分析的系统资源类型（例如：内存、I/O或锁机制）方面也存在差异。尽管持续性能分析器在上述方面有所不同，但它们都具有一个共同的功能：为各种语言运行时提供低开销、基于样本的性能分析，并提供远程堆栈踪迹存储，以便进行基于Web的搜索和查询。

Where is Continuous Profiling headed? Thomas Dullien, co-founder of Optimyze which developed the innovative Continuous Profiler Prodfiler, delivered the Keynote at QCon London 2023 in which he expressed his wish for a cluster-wide tool that could answer the questions, “Why is this request slow?” or “Why is this request expensive?” In a multithreaded application, one particular function may show up on a profile as the highest CPU and memory consumer, yet its duties might be completely outside an application's critical path, e.g., a housekeeping thread. Meanwhile, another function with such insignificant CPU execution time that it barely registers in a profile may exhibit an outsized effect on overall application latency and/or throughput. Typical profilers fail to address this shortcoming. And since CP tools are basically profilers that run at all times, they inherit this same blind spot.

持续性能分析的未来发展方向是什么？Thomas Dullien是Optimyze的联合创始人，该公司开发了创新的持续性能分析工具Prodfiler。他在2023年伦敦QCon大会上发表了主题演讲，表达了他对一款能够解答“为什么这个请求这么慢？”或“为什么这个请求这么耗时？”这类问题的集群级工具的期望。在多线程应用程序中，某个函数可能在性能分析中显示为CPU和内存消耗最高的函数，但它的职责可能完全不在应用程序的关键路径上，例如，它可能只是一个维护线程。与此同时，另一个CPU执行时间微乎其微、几乎无法在性能分析中体现的函数，却可能对应用程序的整体延迟和/或吞吐率产生显著影响。传统的性能分析工具无法解决这一缺陷。由于持续性能分析工具本质上是始终运行的性能分析工具，它们也存在同样的盲点。

Thankfully, a new generation of CP tools has emerged that employ AI with Large Language Model-inspired architectures to process profile samples, analyze the relationships between functions, and finally pinpoint with high accuracy the functions and libraries that directly impact overall throughput and latency. One such company that offers this today is Raven.io. As competition intensifies in this space, innovative capabilities will continue to grow so that CP tooling becomes as powerful and robust as that of typical profilers.

值得庆幸的是，新一代的持续性能分析工具已经出现，它们采用人工智能技术，并借鉴大型语言模型（LLM）激发的体系结构来处理性能分析样本，分析函数之间的关系，最终能够高精度地定位直接影响整体吞吐量和延迟的函数和库。目前提供此类服务的公司之一是Raven.io。随着该领域竞争的加剧，创新功能将不断发展，从而使CP工具变得像传统性能分析器一样强大而稳定。

[^1]: Parca - [https://github.com/parca-dev/parca](https://github.com/parca-dev/parca)
[^2]: Differential flamegraph 差异火焰图 - [https://www.brendangregg.com/blog/2014-11-09/differential-flame-graphs.html](https://www.brendangregg.com/blog/2014-11-09/differential-flame-graphs.html)
