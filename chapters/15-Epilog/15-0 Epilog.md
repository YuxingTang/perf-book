\phantomsection
# Epilog 尾声 {.unnumbered}

\markboth{Epilog}{Epilog}

Thanks for reading through the whole book. I hope you enjoyed it and found it useful. I would be even happier if the book would help you solve a real-world problem. In such a case, I would consider it a success and proof that my efforts were not wasted. Before you continue with your endeavors, let me briefly highlight the essential points of the book and give you final recommendations:

感谢您通读全书。希望您喜欢并觉得本书有用。如果本书能帮助您解决实际问题，我会更加高兴。那样的话，我会认为本书取得了成功，也证明我的努力没有白费。在您继续探索之前，请允许我简要总结本书要点并给出一些最终建议：

* Modern software is massively inefficient. There are significant optimization opportunities to reduce carbon emissions and make a better user experience. People hate using slow software, especially when their productivity goes down because of it. Not all fast software is world-class, but all world-class software is fast. Performance is _the_ killer feature.
* 现代软件效率极低。存在着巨大的优化空间，可以减少碳排放并提升用户体验。人们讨厌使用运行缓慢的软件，尤其是在生产力因此下降的情况下。并非所有运行快速的软件都是世界一流的，但所有世界一流的软件都运行快速。性能是*决定性*因素。
* Single-threaded CPU performance is not increasing as rapidly as it used to a few decades ago. When it's no longer the case that each hardware generation provides a significant performance boost, developers should start optimizing the code of their software.
* 单线程 CPU 的性能提升速度已不如几十年前那么快。当每一代硬件都无法带来显著的性能提升时，开发人员就应该开始优化他们的软件代码。
* For many years performance engineering was a nerdy niche. But now it is mainstream as software vendors have realized the impact that their poorly optimized software has on their bottom line. Performance tuning is more critical than it has been for the last 40 years. It will be one of the key drivers for performance gains in the near future.
* 多年来，性能工程一直是一个略显小众的领域。但如今，随着软件供应商意识到软件优化不足对其利润的影响，性能工程已成为主流。性能调优的重要性远超过去40年，并将成为未来提升性能的关键驱动力之一。
* The importance of low-level performance tuning should not be underestimated, even if it's just a 1% improvement. The cumulative effect of these small improvements is what makes a difference.
* 即使只有1%的提升，底层性能调优的重要性也不容低估。这些微小改进的累积效应才是关键所在。
* There is a famous quote by Donald Knuth: "Premature optimization is the root of all evil".[@Knuth1974StructuredPW] The opposite is often true as well. Postponed performance engineering work may be too late and cause as much evil as premature optimization. Do not neglect performance aspects when designing your future products. Save your project by integrating automated performance benchmarking into your CI/CD pipeline. *Measure early, measure often.*
* 图灵奖得主Donald Knuth曾说过一句名言：“过早优化是万恶之源。”[@Knuth1974StructuredPW]。反过来也常常如此。推迟性能工程工作可能为时已晚，其危害与过早优化不相上下。在设计未来产品时，切勿忽视性能因素。将自动化性能基准测试集成到 CI/CD 流水线中，即可挽救您的项目。*尽早测量，频繁测量。*
* Knowledge of the CPU microarchitecture is required to reach peak performance. However, your mental model can never be as accurate as the actual microarchitecture design of a CPU. So don't solely rely on your intuition when you make a specific change in your code. Predicting the performance of a particular piece of code is nearly impossible. *Always measure!*
* 要达到最佳性能，必须了解 CPU 微架构。然而，你的心智模型永远无法像 CPU 的实际微架构设计那样精确。因此，在对代码进行特定更改时，不要仅仅依赖直觉。预测特定代码段的性能几乎是不可能的。*务必进行测量！*
* When measuring performance, understand the underlying technical reasons for the performance results you observe. Always measure one level deeper and collect as many metrics as possible to support your conclusions.
* 测量性能时，要理解观察到的性能结果背后的技术原因。始终深入测量更深一层，并收集尽可能多的指标来支持你的结论。
* Performance engineering is hard because there are no predetermined steps you should follow, no algorithm. Engineers need to tackle problems from different angles. Know performance analysis methods and tools (both hardware and software) available to you. I strongly suggest embracing the Top-down Microarchitecture Analysis (TMA) methodology. It will help you steer your work in the right direction. 
* 性能工程之所以困难，是因为没有预先设定的步骤或算法。工程师需要从不同的角度解决问题。了解可用的性能分析方法和工具（包括硬件和软件）。我强烈建议采用自顶向下的微体系结构分析(TMA)方法。它将帮助你朝着正确的方向前进。
* When you identify the performance-limiting factor of your application, you are more than halfway through. Based on my experience, the fix is often easier than finding the root cause of the problem.
In Part 2 we covered some essential optimizations for every type of CPU performance bottleneck: how to optimize memory accesses and computations, how to get rid of branch mispredictions, how to improve machine code layout, and several others. Use chapters from Part 2 as a reference to see what options are available when your application has one of these problems.
* 当你找到应用程序的性能瓶颈时，你就已经成功了一半以上。根据我的经验，修复问题通常比找到问题的根本原因更容易。在第二部分中，我们介绍了针对各种 CPU 性能瓶颈的一些关键优化方法：如何优化内存访问和计算、如何消除分支预测错误、如何改进机器代码布局等等。您可以参考第二部分中的章节，了解当您的应用程序遇到这些问题时有哪些可行的解决方案。
* Processors from different vendors are not created equal. They differ in terms of instruction set architecture (ISA) supported and microarchitectural implementation. Reaching peak performance on a given platform requires utilizing the latest ISA extensions, avoiding common microarchitecture-specific issues, and tuning your code according to the strengths of a particular CPU microarchitecture.
* 不同厂商的处理器性能并不相同。它们在支持的指令集架构 (ISA) 和微架构实现方面存在差异。要在特定平台上达到最佳性能，需要利用最新的 ISA 扩展、避免常见的微架构特有问题，并根据特定 CPU 微架构的优势调整代码。
* Multithreaded programs add one more dimension of complexity to performance tuning. They introduce new types of bottlenecks and require additional tools and methods to analyze and optimize. Examining how an application scales with the number of threads is an effective way to identify bottlenecks in multithreaded programs.
* 多线程程序为性能调优增加了另一个复杂性维度。它们引入了新的瓶颈类型，需要额外的工具和方法来分析和优化。检查应用程序如何随线程数扩展是识别多线程程序瓶颈的有效方法。

I hope you now have a better understanding of low-level performance optimizations. Of course, this book doesn't cover every possible scenario you may encounter in your daily job. My goal was to give you a starting point and to show you potential options and strategies for dealing with performance analysis and tuning on modern CPUs. I wish you experience the joy of discovering performance bottlenecks in your application and the satisfaction of fixing them.

希望您现在对底层性能优化有了更深入的了解。当然，本书不可能涵盖您日常工作中可能遇到的所有情况。我的目标是为您提供一个起点，并向您展示在现代 CPU 上进行性能分析和调优的潜在选项和策略。我希望您能体验到发现应用程序中性能瓶颈的乐趣，以及解决瓶颈带来的成就感。

**Happy performance tuning!**

**性能调优愉快！**

I will post errata and other information about the book on my blog at the following URL: [https://easyperf.net/blog/2024/11/11/Book-Updates-Errata](https://easyperf.net/blog/2024/11/11/Book-Updates-Errata).

我会在以下网址的博客上发布本书的勘误和其他信息：[https://easyperf.net/blog/2024/11/11/Book-Updates-Errata](https://easyperf.net/blog/2024/11/11/Book-Updates-Errata)。

If you haven't solved the `perf-ninja` exercises yet, I encourage you to take the time to do so. They will help you to solidify your knowledge and prepare you for real-world performance engineering challenges.

如果你还没完成“性能忍者”练习，我强烈建议你花时间去做一下。这些练习能帮助你巩固知识，并为应对实际的性能工程挑战做好准备。

P.S. If you enjoyed reading this book, make sure to pass it on to your friends and colleagues. I would appreciate your help in spreading the word about the book by endorsing it on social media platforms.

附：如果你喜欢这本书，请务必把它分享给你的朋友和同事。如果你能在社交媒体平台上推荐这本书，我会非常感谢你的帮助。