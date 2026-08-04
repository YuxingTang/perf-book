### TMA Summary TMA总结

TMA is great for identifying CPU performance bottlenecks. Ideally, we would like to see the `Retiring` metric at 100%. Although there are exceptions. Having the `Retiring` metric at 100% means a CPU is fully saturated and it crunches instructions at full speed. But it doesn't say anything about the quality of those instructions. A program can spin in a tight loop waiting for a lock; that would show a high `Retiring` metric, but it doesn't do any useful work. 

TMA 常适合识别CPU性能瓶颈。理想情况下，我们希望看到 `退役Retiring`（指令执行率）指标达到100%。当然，也有例外情况。 `退役Retiring`指标达到100%意味着CPU已完全饱和，正在全速执行指令。但这并不代表这些指令的质量。例如，程序可能陷入一个紧密的循环等待锁；这种情况会导致 `退役Retiring`指标很高，但实际上并没有执行任何有效的工作。

Another example in which you might see a high `Retiring` metric but slow overall performance is when a program has a hotspot that was not vectorized. You give a processor an "easy" time by letting it run simple non-vectorized operations, but is it an optimal way of using available CPU resources? Of course, no. If a CPU doesn't have problems executing your code, that doesn't mean performance cannot be improved. Watch out for such cases and remember that TMA identifies CPU performance bottlenecks but doesn't correlate them with the performance of your program. You will find it out once you do the necessary experiments.

另一个可能出现 `退役Retiring` 指标很高但整体性能却很慢的例子是，当程序中存在未进行向量化的热点时。通过让处理器执行简单的非向量化操作，可以让处理器简单的工作起来，但这真的是利用可用CPU资源的最佳方式吗？当然不是。即使CPU执行你的代码没有问题，也不意味着性能无法提升。注意此类情况，并记住TMA可以识别CPU性能瓶颈，但不会将其与程序的实际性能关联起来。您需要进行必要的实验才能找到答案。

While it is possible to achieve `Retiring` close to 100% on a toy program, real-world applications are far from getting there. Figure @fig:TMA_google shows top-level TMA metrics for Google's datacenter workloads along with several [SPEC CPU2006](http://spec.org/cpu2006/)[^13] benchmarks running on Intel's Ivy Bridge server processors. We can see that most data center workloads have a very small fraction in the `Retiring` bucket. This implies that most data center workloads spend time stalled on various bottlenecks. `BackendBound` is the primary source of performance issues. The `FrontendBound` category represents a bigger problem for data center workloads than in SPEC2006 because those applications typically have large codebases with poor locality. Finally, some workloads suffer from branch mispredictions more than others, e.g., `search2` and `445.gobmk`.

虽然在玩具程序上可以实现接近100% 的`退役Retiring`状态，但实际应用远未达到这种程度。图 @fig:TMA_google 展示了Google数据中心工作负载的顶级TMA指标，以及在Intel Ivy Bridge服务器处理器上运行的多个[SPEC CPU2006](http://spec.org/cpu2006/)[^13]基准测试。我们可以看到，大多数数据中心工作负载中`退役Retiring`状态的比例非常小。这意味着大多数数据中心工作负载都会因各种瓶颈而停滞不前。`BackendBound后端瓶颈`是性能问题的主要来源。`FrontendBound前端瓶颈` 类别对于数据中心工作负载来说比在SPEC2006中更为严重，因为这些应用程序通常具有庞大的代码库，且局部性较差。最后，某些工作负载比其他工作负载更容易出现分支预测错误，例如 `search2` 和 `445.gobmk`。

![TMA breakdown of Google's datacenter workloads along with several SPEC CPU2006 benchmarks, *© Source: [@GoogleProfiling]* TMA 对 Google 数据中心工作负载的分析以及几个 SPEC CPU2006 基准测试结果，*© 来源：[@GoogleProfiling]*](../../img/pmu-features/TMA_google.jpg){#fig:TMA_google width=80%}

Keep in mind that the numbers are likely to change for other CPU generations as architects constantly try to improve the CPU design. The numbers are also likely to change for other instruction set architectures (ISA) and compiler versions.

请记住，由于体系结构设计师不断改进CPU设计，这些数字可能会随着CPU代次的变化而变化。此外，这些数字也可能随着指令集体系结构(ISA)和编译器版本的不同而变化。

A few final thoughts before we move on... As we mentioned at the beginning of this chapter, using TMA on code that has major performance flaws is not recommended because it will likely steer you in the wrong direction, and instead of fixing real high-level performance problems, you will be tuning bad code, which is just a waste of time. Similarly, make sure the environment doesn’t get in the way of profiling. For example, if you drop the filesystem cache and run the benchmark under TMA, it will likely show that your application is Memory Bound, which may in fact be false when the filesystem cache is warmed up.

在继续之前，还有几点需要说明…… 正如我们在本章开头提到的，不建议对存在严重性能缺陷的代码使用TMA，因为它很可能会误导您，让您误入歧途，最终只是在浪费时间地调整糟糕的代码，而不是真正解决根本的性能问题。同样，要确保环境不会干扰性能分析。例如，如果您禁用文件系统缓存并在TMA下运行基准测试，则测试结果很可能显示您的应用程序受限于内存，但当文件系统缓存预热后，这一结果可能并非如此。

Workload characterization provided by TMA can increase the scope of potential optimizations beyond source code. For example, if an application is bound by memory bandwidth and all possible ways to speed it up on the software level have been exhausted, it may be possible to improve performance by upgrading the memory subsystem with faster memory chips. This demonstrates how using TMA to diagnose performance bottlenecks can support your decision to spend money on new hardware.

TMA提供的工作负载特征分析可以将潜在优化范围扩展到源代码之外。例如，如果应用程序受限于内存带宽，并且所有可能的软件加速方法都已用尽，则可以通过升级内存子系统，使用更快的内存芯片来提高性能。这表明，使用TMA诊断性能瓶颈可以帮助您决定是否投资购买新硬件。

[^13]: SPEC CPU 2006 - [http://spec.org/cpu2006/](http://spec.org/cpu2006/).
