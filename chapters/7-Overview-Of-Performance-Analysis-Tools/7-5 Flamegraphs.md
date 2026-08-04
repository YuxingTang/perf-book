## Flame Graphs 火焰图 {#sec:secFlameGraphs}

A flame graph is a popular way of visualizing profiling data and the most frequent code paths in a program. It enables us to see which function calls take the largest portion of execution time. Figure @fig:FlameGraph shows an example of a flame graph for the [x264](https://openbenchmarking.org/test/pts/x264) video encoding benchmark, generated with open-source [scripts](https://github.com/brendangregg/FlameGraph)[^1] developed by Brendan Gregg. Nowadays, nearly all profilers can automatically generate a flame graph as long as the call stacks are collected during the profiling session.

火焰图是一种常用的可视化性能分析数据和程序中最常用代码路径的方法。它能够帮助我们了解哪些函数调用占用了大部分执行时间。图 @fig:FlameGraph 展示了使用Brendan Gregg开发的开源脚本[scripts](https://github.com/brendangregg/FlameGraph)[^1] 生成的 [x264](https://openbenchmarking.org/test/pts/x264) 视频编码基准测试的火焰图示例。如今，几乎所有性能分析器都可以在性能分析会话期间收集调用堆栈后自动生成火焰图。

![A flame graph for the x264 benchmark. x264基准测试的火焰图。](../../img/perf-tools/Flamegraph.jpg){#fig:FlameGraph width=100%}

On the flame graph, each rectangle (horizontal bar) represents a function call, and the width of the rectangle indicates the relative execution time taken by the function itself and by its callees. The function calls happen from the bottom to the top, so we can see that the hottest path in the program is `x264` &rarr; `threadpool_thread_internal` &rarr; `...` &rarr; `x264_8_macroblock_analyse`. The function `threadpool_thread_internal` and its callees account for 74% of the time spent in the program. But the self-time, i.e., time spent in the function itself is rather small. Similarly, we can do the same analysis for `x264_8_macroblock_analyse`, which accounts for 66% of the runtime. This visualization gives you a very good intuition on where the most time is spent.

在火焰图中，每个矩形（水平条）代表一次函数调用，矩形的宽度表示函数自身及其被调用者所占用的相对执行时间。函数调用从下往上进行，因此我们可以看到程序中最耗时的路径是 `x264`--> `threadpool_thread_internal` --> `...` --> `x264_8_macroblock_analyse`。函数 `threadpool_thread_internal` 及其被调用者占用了程序总执行时间的74%。但函数自身执行的时间（即函数自身所占用的时间）却相当短。同样，我们可以对 `x264_8_macroblock_analyse` 进行同样的分析，它占用了66%的运行时间。这种可视化方式能让你直观地了解时间都花在了哪里。

Flame graphs are interactive. You can click on any bar on the image and it will zoom into that particular code path. You can keep zooming until you find a place that doesn't look according to your expectations or you reach a leaf/tail function---now you have actionable information you can use in your analysis. Another strategy is to figure out what is the hottest function in the program (not immediately clear from this flame graph) and go bottom-up through the flame graph, trying to understand from where this hottest function gets called. 

火焰图是交互式的。你可以点击图像中的任何柱状图，它会放大到相应的代码路径。你可以不断放大，直到找到与预期不符的地方，或者到达叶节点/尾节点——这时你就获得了可用于分析的实用信息。另一种策略是找出程序中最热门的函数（从这个火焰图中可能看不出来），然后自下而上地分析火焰图，尝试了解这个最热门的函数是从哪里调用的。

Some tools prefer to use an *icicle graph*, which is the upside-down version of a flame graph (see an example in [@sec:ContinuousProfiling]).

有些工具更倾向于使用“冰柱图”，它是火焰图的倒置版本（参见 [@sec:ContinuousProfiling] 中的示例）。

[^1]: Flame graphs by Brendan Gregg Brendan Gregg的火焰图 - [https://github.com/brendangregg/FlameGraph](https://github.com/brendangregg/FlameGraph)
[^2]: x264 video encoding benchmark x264视频编码基准测试 - [https://openbenchmarking.org/test/pts/x264](https://openbenchmarking.org/test/pts/x264)
