## Active Benchmarking 主动基准测试

As you have seen in the previous sections, measuring performance is a complex task with many pitfalls along the way. As human beings, we tend to welcome favorable results and ignore unfavorable ones. This often leads to benchmarking done in a "run and forget" style, with no additional analysis, and overlooking any potential problems. Measurements done in this way are likely incomplete, misleading, or even erroneous. Consider the following two scenarios:

正如你在前几节中所看到的，性能测量是一项复杂的任务，其中充满了各种陷阱。作为人类，我们往往乐于接受有利的结果，而忽略不利的结果。这通常会导致基准测试以“运行后即忘”的方式进行，缺乏后续分析，并忽略任何潜在问题。以这种方式进行的测量很可能是不完整的、误导性的，甚至是错误的。请考虑以下两种情况：

* Developer A on a team meeting: "If we add the `final` keyword to the class declaration across our entire C++ codebase, it will make our code 5% faster, with some tests showing up to 30% speedup."
* 开发人员A在团队会议上说：“如果我们在整个C++代码库的类声明中添加 `final` 关键字，代码速度将提高5%，某些测试甚至显示速度提升高达30%。”
* Developer B on the next team meeting: "I looked closely at the performance impact of adding the `final` keyword to the class declarations. First, I performed longer tests and haven't measured speedups larger than 5%. The initially observed 30% speedups were outliers caused by test instability. I also noticed that the two machines used for measurements have different configurations: while the CPUs are identical, one of the machines has faster memory modules. I reran the tests on the same machine and observed a performance difference within 1%. I compared the generated machine code before and after the change and found no significant differences. Also, I compared the number of instructions executed, cache misses, page faults, context switches, etc., and haven't found any anomalies. At this point, I concluded that the performance impact of the `final` keyword is negligible compared to other optimizations we could make."
* 开发人员B在下次团队会议上说：“我仔细研究了在类声明中添加 `final` 关键字对性能的影响。首先，我进行了更长时间的测试，发现速度提升不超过5%。最初观察到的30 的速度提升是由于测试不稳定造成的异常值。我还注意到，用于测试的两台机器配置不同：虽然CPU相同，但其中一台机器的内存模块速度更快。我在同一台机器上重新运行了测试，发现性能差异在1%以内。我比较了更改前后生成的机器代码，没有发现显著差异。此外，我还比较了执行的指令数、缓存未命中数、页面错误数、上下文切换数等，也没有发现任何异常。因此，我认为与其他优化相比，`final` 关键字对性能的影响可以忽略不计。”

Benchmarking done by developer A was done in a passive way. The results were presented without any technical explanation, and the performance impact was exaggerated. In contrast, developer B performed *active benchmarking*.[^1] She ensured proper machine configuration, ran extensive testing, looked one level deeper, and collected as many metrics as possible to support her conclusions. Her analysis explains the underlying technical reason for the performance results she observed.

开发人员A进行的基准测试是被动的。他没有对结果进行任何技术解释就直接呈现，并且夸大了性能影响。相比之下，开发者B进行了*主动基准测试*[^1]。她确保了机器配置正确，进行了广泛的测试，深入分析了数据，并收集了尽可能多的指标来支持她的结论。她的分析解释了她观察到的性能结果背后的技术原因。

You should have a good intuition to spot suspicious benchmark results. Whenever you see publications that present benchmark results that look too good to be true and without any technical explanation, you should be skeptical. There is nothing wrong with presenting the results of your measurements, but as John Ousterhout said, "Performance measurements should be considered guilty until proven innocent." [@MeasureOneLevelDeeper] The best way to verify the results is through active benchmarking. Active benchmarking requires much more effort than passive benchmarking, but it is the only way to get reliable results.

你应该有敏锐的直觉来识别可疑的基准测试结果。每当你看到一些出版物展示的基准测试结果好得令人难以置信，而且没有任何技术解释时，你都应该保持怀疑。展示你的测量结果并没有错，但正如John Ousterhout所说，“性能测量结果应该被视为有罪，直到证明其无罪。” [@MeasureOneLevelDeeper] 验证结果的最佳方法是通过主动基准测试。主动基准测试比被动基准测试需要付出更多努力，但它是获得可靠结果的唯一途径。

[^1]: A term coined by Brendan Gregg 由Brendan Gregg创造的术语 - [https://www.brendangregg.com/activebenchmarking.html](https://www.brendangregg.com/activebenchmarking.html).
