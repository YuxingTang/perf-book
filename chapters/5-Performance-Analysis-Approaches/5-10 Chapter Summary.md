## Chapter Summary 章节小结 {#sec:secApproachesSummary .unlisted .unnumbered}

\markright{Summary小结}

* Latency and throughput are often the ultimate metrics of the program performance. When seeking ways to improve them, we need to get more detailed information on how the application executes. Both hardware and software provide data that can be used for performance monitoring.

* 延迟和吞吐率通常是衡量程序性能的最终指标。为了改进这些指标，我们需要获取应用程序执行方式的更详细信息。硬件和软件都提供可用于性能监控的数据。

* Code instrumentation enables us to track many things in a program but causes relatively large overhead both on the development and runtime side. While most developers are not in the habit of manually instrumenting their code, this approach is still relevant for automated processes, e.g., Profile-Guided Optimizations (PGO).

* 代码插桩使我们能够跟踪程序中的许多内容，但会在开发和运行时都造成相对较大的开销。虽然大多数开发人员没有手动插桩代码的习惯，但这种方法对于自动化流程仍然适用，例如基于轮廓分析文件的优化(PGO: Profile-Guided Optimization)。

* Tracing is conceptually similar to instrumentation and is useful for exploring anomalies in a system. Tracing enables us to catch the entire sequence of events with timestamps attached to each event.

* 跟踪在概念上与插桩类似，可用于探索系统中的异常情况。跟踪使我们能够捕获整个事件序列，并为每个事件附加时间戳。

* Performance monitoring counters are a very important instrument of low-level performance analysis. They are generally used in two modes: "Counting" or "Sampling". The counting mode is primarily used for calculating various performance metrics. 

* 性能监控计数器是底层性能分析中非常重要的工具。它们通常以两种模式使用：“计数”或“采样”。计数模式主要用于计算各种性能指标。

* Sampling skips most of a program's execution and takes just one sample that is supposed to represent the entire interval. Despite this, sampling usually yields precise enough distributions. The most well-known use case of sampling is finding hotspots in a program. Sampling is the most popular analysis approach since it doesn't require recompilation of a program and has very little runtime overhead.

* 采样跳过了程序执行的大部分过程，仅采集一个样本来代表整个区间。尽管如此，采样通常也能产生足够精确的分布。采样最常见的应用场景是查找程序中的热点。采样是最常用的分析方法，因为它不需要重新编译程序，并且运行时开销极低。

* Generally, counting and sampling incur very low runtime overhead (usually below 2%). Counting gets more expensive once you start multiplexing between different events (5--15% overhead), while sampling gets more expensive with increasing sampling frequency [@Nowak2014TheOO].

* 通常，计数和采样都会产生非常低的运行时开销（通常低于2%）。一旦开始对不同事件进行多路复用时，计数开销会增加（5%到15%的开销），而采样开销会随着采样频率的增加而增加 [@Nowak2014TheOO]。

* The Roofline Performance Model is a throughput-oriented performance model that is heavily used in the High Performance Computing (HPC) world. It visualizes the performance of an application against hardware limitations. The Roofline model helps to identify performance bottlenecks, guides software optimizations, and keeps track of optimization progress.

* 屋顶线Roofline性能模型是一种面向吞吐率的性能模型，广泛应用于高性能计算(HPC)领域。它将应用程序的性能与硬件限制进行可视化对比。屋顶线Roofline模型有助于识别性能瓶颈、指导软件优化并跟踪优化进度。

* There are tools that try to statically analyze the performance of code. Such tools simulate a piece of code instead of executing it. Many limitations and constraints apply to this approach, but you get a very detailed and low-level report in return.

* 有些工具可以尝试对代码进行静态性能分析。这类工具模拟一段代码，而不是实际执行它。这种方法有很多局限性和限制，但可以生成非常详细且底层性能分析报告。

* Compiler Optimization reports help to find missing compiler optimizations.

* 编译器优化报告有助于发现缺失的编译器优化。

\sectionbreak
