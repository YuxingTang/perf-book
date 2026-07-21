\phantomsection
# Part 1. Performance Analysis on a Modern CPU 第一部分：在现代处理器上的性能分析 {.unnumbered}

# Measuring Performance 测量性能 {#sec:secMeasPerf}

The first step to understanding the performance of an application is to measure it. Anyone ever concerned with performance evaluations likely knows how hard it is sometimes to conduct fair performance measurements and draw accurate conclusions from them. Performance measurements can be very unexpected and counterintuitive. Changing a seemingly unrelated part of the source code can surprise us with a significant impact on the performance of the program. For various reasons, measurements may consistently overestimate or underestimate the true performance, which leads to distorted results that do not accurately reflect reality. This phenomenon is called *measurement bias*.

了解应用程序性能的第一步是对其进行测量。任何关心绩效评估的人都可能知道，有时进行公平的绩效评估并从中得出准确的结论是多么困难。性能测量可能非常出乎意料且违反直觉。更改源代码中看似不相关的部分可能会对程序的性能产生重大影响，这会让我们感到惊讶。由于各种原因，测量结果可能始终高估或低估真实性能，从而导致结果失真，无法准确反映现实。这种现象称为“测量偏差measurement bias”。

Performance problems are often harder to reproduce and root cause than most functional issues. Every run of a program is usually functionally the same but somewhat different from a performance standpoint. For example, when unpacking a zip file, we get the same result over and over again, which means this operation is reproducible. However, it's impossible to reproduce the same CPU cycle-by-cycle performance profile of this operation.

性能问题通常比大多数功能问题更难以重现和根本原因。程序的每次运行通常在功能上是相同的，但从性能的角度来看有些不同。例如，当解压 zip 文件时，我们一遍又一遍地得到相同的结果，这意味着此操作是可重现的。然而，不可能重现此操作的相同 CPU 逐周期性能概况。

Conducting fair performance experiments is an essential step towards getting accurate and meaningful results. You need to ensure you're looking at the right problem and are not debugging some unrelated issue. Designing performance tests and configuring the environment are both important components in the process of evaluating performance. 

进行公平的性能实验是获得准确且有意义的结果的重要一步。您需要确保您正在寻找正确的问题，并且没有调试一些不相关的问题。设计性能测试和配置环境都是性能评估过程中的重要组成部分。

Because of the measurement bias, performance evaluations often involve statistical methods, which deserve a whole book just for themselves. There are many corner cases and a huge amount of research done in this field. We will not dive into statistical methods for evaluating performance measurements. Instead, we only discuss high-level ideas and give basic directions to follow. We encourage you to research deeper on your own.

由于测量偏差，绩效评估经常涉及统计方法，这些方法本身就值得写一整本书。该领域有许多极端案例和大量研究。我们不会深入探讨评估绩效衡量的统计方法。相反，我们只讨论高层次的想法并给出可遵循的基本方向。我们鼓励您自己进行更深入的研究。

In this chapter, we:

在本章中，我们：

- Give a brief introduction to why modern systems yield noisy performance measurements and what you can do about it. 
- 简要介绍为什么现代系统会产生嘈杂的性能测量结果以及您可以采取哪些措施。
- Explain why it is important to measure performance in production deployments.
- 解释为什么在生产部署中的测试性能很重要。
- Provide general guidance on how to properly collect and analyze performance measurements. 
- 提供有关如何正确收集和分析性能测量结果的一般指导。
- Explore how to automatically detect performance regressions as you implement changes in your codebase.
- 探索在代码库中实施更改时如何自动检测性能回归。
- Describe software and hardware timers that can be used by developers in time-based measurements. 
- 描述开发人员可以在基于时间的测量中使用的软件和硬件计时器。
- Discuss how to write a good microbenchmark and some common pitfalls you may encounter while doing it.
- 讨论如何编写良好的微基准测试以及在编写过程中可能遇到的一些常见陷阱。