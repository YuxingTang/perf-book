## Questions and Exercises 问题与习题 {.unlisted .unnumbered}

\markright{Questions and Exercises问题与习题}

1. Which performance analysis approaches would you use in the following scenarios?
- scenario 1: the client support team reports a customer issue: after upgrading to a new version of the application, the performance of a certain operation drops by 10%.
- scenario 2: the client support team reports a customer issue: some transactions run 2x longer than others with no particular pattern.
- scenario 3: you're evaluating three different compression algorithms and you want to know what types of performance bottlenecks (memory latency, computations, branch mispredictions, etc) each of them has.
- scenario 4: there is a new shiny library that claims to be faster than the one you currently have integrated into your project; you've decided to compare their performance.
- scenario 5: you were asked to analyze the performance of some unfamiliar code, which involves a hot loop; you want to know how many iterations the loop is doing.
2. Run the application that you're working with daily. Practice doing performance analysis using the approaches we discussed in this chapter. Collect raw counts for various CPU performance events, find hotspots, collect roofline data, and generate and study the compiler optimization report for the hot function(s) in your program.

1. 在以下场景中，您会使用哪些性能分析方法？
- 场景1：客户支持团队报告了一个客户问题：应用程序升级到新版本后，某个操作的性能下降了10%。
- 场景2：客户支持团队报告了一个客户问题：某些事务的运行时间是其他事务的2倍，且没有特定的规律。
- 场景3：您正在评估3种不同的压缩算法，并想知道每种算法存在哪些类型的性能瓶颈（内存延迟、计算、分支预测错误等）。
- 场景4：出现了一个新的库，声称比您当前集成到项目中的库更快；您决定比较它们的性能。
- 场景5：您被要求分析一些不熟悉的代码的性能，其中包含一个热循环；您想知道该循环执行了多少次迭代。
2. 运行你日常使用的应用程序。练习使用本章讨论的方法进行性能分析。收集各种CPU性能事件的原始计数，找出性能热点，收集性能曲线数据，并生成和研究程序中性能热点函数的编译器优化报告。