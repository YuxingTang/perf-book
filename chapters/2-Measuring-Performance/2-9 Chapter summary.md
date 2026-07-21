## Chapter Summary 本章小结 {.unlisted .unnumbered}

\markright{Summary}

* Modern systems have nondeterministic performance. Eliminating nondeterminism in a system is helpful for well-defined, stable performance tests, e.g., microbenchmarks.
* Measuring performance in production is required to assess how users perceive the responsiveness of your services. However, this requires dealing with noisy environments and using statistical methods for analyzing results. 
* It is beneficial to employ an automated performance tracking system to prevent performance regressions from leaking into production software. Such CI systems are supposed to run automated performance tests, visualize results, and alert on discovered performance anomalies.
* Visualizing performance distributions helps compare performance results. It is a safe way of presenting performance results to a wide audience.
* To benchmark execution time, engineers can use two different timers: the system-wide high-resolution timer and the Time Stamp Counter. The former is suitable for measuring events whose duration is more than a microsecond. The latter can be used for measuring short events with high accuracy.
* Microbenchmarks are good for quick experiments, but you should always verify your ideas on a real application in practical conditions. Make sure that you are benchmarking the right code by checking performance profiles.
* Always measure one level deeper, collect as many metrics as possible to support your conclusions, and be ready to explain the underlying technical reasons for the performance results you observe.

* 现代系统的性能具有不确定性。消除系统中的不确定性有助于进行定义明确、稳定的性能测试，例如微基准测试。
* 测量生产环境中的性能对于评估用户对服务响应速度的感知至关重要。然而，这需要处理嘈杂的环境并使用统计方法分析结果。
* 采用自动化性能跟踪系统有助于防止性能退化影响生产级软件。此类持续集成(CI)系统应运行自动化性能测试、可视化结果，并在发现性能异常时发出警报。
* 可视化性能分布有助于比较性能结果。这是一种向广大受众安全展示性能结果的方法。
* 为了对执行时间进行基准测试，工程师可以使用两种不同的计时器：系统级高分辨率计时器和时间戳计数器。前者适用于测量持续时间超过一微秒的事件。后者可用于高精度测量短时事件。
* 微基准测试适用于快速实验，但您始终应该在实际应用中验证您的想法。通过检查性能分析，确保您测试的是正确的代码。
* 始终进行更深层次的测量，收集尽可能多的指标来支持您的结论，并准备好解释您观察到的性能结果背后的技术原因。

\sectionbreak



