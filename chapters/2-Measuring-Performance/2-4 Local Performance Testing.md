## Manual Performance Testing 手动性能测试

In the previous section, we discussed how CI systems can help with evaluating the performance impact of a code change. However, it may not always be possible to leverage such a system due to reasons such as hardware unavailability, setup being too complicated for the testing infrastructure, a need to collect additional metrics, etc. In this section, we provide basic advice for local performance evaluations.

在上一节中，我们讨论了持续集成(CI)系统如何帮助评估代码更改对性能的影响。然而，由于硬件不可用、测试基础设施设置过于复杂、需要收集其他指标等原因，并非总是能够利用此类系统。本节将提供一些本地性能评估的基本建议。

We typically measure the performance impact of our code change by 1) measuring the baseline performance, 2) measuring the performance of the modified program, and 3) comparing them with each other. For example, we had a program that calculates Fibonacci numbers recursively (baseline), and we decided to rewrite it with a loop (modified). Both versions are functionally correct and yield the same Fibonacci numbers. Now we need to compare the performance of the two versions of the program.

我们通常通过以下步骤来衡量代码更改对性能的影响：1）测量基准程序性能；2）测量修改后程序的性能；3）将两者进行比较。例如，我们有一个递归计算斐波那契数列的程序（基准程序），现在我们决定用循环重写它（修改后程序）。两个版本功能都正确，并且计算出的斐波那契数列相同。现在我们需要比较这两个版本的程序性能。

It is highly recommended to get not just a single measurement but to run the benchmark multiple times. If you make comparisons based on a single measurement, you're increasing the risk of having your numbers skewed by the measurement bias that we discussed in [@sec:secFairExperiments]. So, we collected `N` performance measurements for the baseline and `N` measurements for the modified version of the program. We call a set of performance measurements a *performance distribution*. Now we need to aggregate and compare those two distributions to decide which version of the program is faster.

强烈建议不要只进行一次测量，而是多次运行基准测试。如果仅基于单一测量结果进行比较，就会增加测量偏差导致数据失真的风险，我们在[@sec:secFairExperiments]中讨论过这个问题。因此，我们收集了基线程序的N个性能测量值，以及修改后程序的N个性能测量值。我们将这组性能测量值称为*性能分布*。现在，我们需要汇总并比较这两个分布，以确定哪个版本的程序速度更快。

The most straightforward way to compare two performance distributions is to take the average of `N` measurements from both distributions and calculate the ratio. For the types of code improvements we discuss in this book, this simple method works well in most cases. However, comparing performance distributions is quite nuanced, and there are many ways how you can be fooled by measurements and potentially derive wrong conclusions. We will not get into the details of statistical analysis, instead, we recommend you read a textbook on the subject. A good reference specifically for performance engineers is a book by Dror G. Feitelson, "Workload Modeling for Computer Systems Performance Evaluation",[^12] that has more information on modal distributions, skewness, and other related topics.

比较两个性能分布最直接的方法是分别取两个分布中 `N` 个测量值的平均值，然后计算比值。对于本书讨论的代码改进类型，这种简单的方法在大多数情况下都适用。然而，性能分布的比较相当微妙，测量结果很容易误导人，导致得出错误的结论。我们不会深入探讨统计分析的细节，而是建议您阅读相关教材。对于性能工程师来说，Dror G. Feitelson的著作《计算机系统性能评估的工作负载建模》[^12]是一本很好的参考书，其中包含更多关于模态分布、偏斜度和其他相关主题的信息。

Data scientists often present measurements by plotting them. This eliminates biased conclusions and allows readers to interpret the data for themselves. One of the popular ways to plot distributions is by using box plots (also known as a box-and-whisker plot). In Figure @fig:BoxPlot, we visualized performance distributions of two versions of the same functional program ("before" and "after"). There are 70 performance data points in each distribution.

数据科学家通常通过绘制图表来呈现测量结果。这可以消除有偏的结论，并允许读者自行解读数据。绘制分布图的一种常用方法是使用箱线图（也称为箱须图box-and-whisker plot）。在图 @fig:BoxPlot 中，我们可视化了同一功能程序的两个版本（“之前”和“之后”）的性能分布。每个分布包含70个性能数据点。

![Performance measurements (lower is better) of "Before" and "After" versions of a program presented as box plots.程序“改进前”和“改进后”版本的性能测量结果（数值越低越好），以箱线图形式呈现。](../../img/measurements/BoxPlots.png){#fig:BoxPlot width=90%}

Let's describe the terms indicated on the image:
让我们解释一下图中所示的术语：

* The *mean* (often referred to as the *average*) is the sum of all values in a dataset divided by the number of values. Indicated with X.
* *平均值*（通常也称为*均值*）是数据集中所有值的总和除以值的数量。图中用X表示。
* The *median* is the middle value of a dataset when the values are sorted. The same as *50th percentile* (p50).
* *中位数*是数据集排序后位于中间位置的值。与*第50百分位数*(p50)相同。
* The *25th percentile* (p25) divides the lowest 25% of the data from the highest 75%.
* *第25百分位数*(p25)将最低的25%数据与最高的75%数据分开。
* The *75th percentile* (p75) divides the lowest 75% of the data from the highest 25%.
* *第75百分位数*(p75)将最低的75%数据与最高的25%数据分开。
* An *outlier* is a data point that differs significantly from other samples in the dataset. Outliers can be caused by variability in the data or experimental errors.
* *异常值*是指与数据集中其他样本显著不同的数据点。异常值可能是由数据变异性或实验误差造成的。
* The *min* and *max* (whiskers) represent the most extreme data points that are not considered outliers. 
* *最小值*和*最大值*（须线）代表最极端的数据点，这些数据点不被视为异常值。

By looking at the box plot in Figure @fig:BoxPlot, we can sense that our code change has a positive impact on performance since "after" samples are generally faster than "before". However, there are some "before" measurements that are faster than "after". Box plots allow comparisons of multiple distributions on the same chart. The benefits of using box plots for visualizing performance distributions are described in a blog post by Stefan Marr.[^13]

通过观察图 @fig:BoxPlot 中的箱线图，我们可以感受到代码更改对性能产生了积极影响，因为“更改后”的样本通常比“更改前”的样本更快。然而，也有一些“更改前”的测量结果比“更改后”的更快。箱线图允许在同一张图表上比较多个分布。Stefan Marr在一篇博文中描述了使用箱线图可视化性能分布的优势[^13]。

Performance speedups can be calculated by taking a ratio between the two means. In some cases, you can use other metrics to calculate speedups, including median, min, and 95th percentile, depending on which one is more representative of your distribution.

性能提升可以通过计算两个均值的比值来获得。在某些情况下，您可以使用其他指标来计算提升，例如中位数、最小值和第95百分位数，具体取决于哪个指标更能代表您的分布。

*Standard deviation* quantifies how much the values in a dataset deviate from the mean on average. A low standard deviation indicates that the data points are close to the mean, while a high standard deviation indicates that they are spread out over a wider range. Unless distributions have low standard deviation, do not calculate speedups. If the standard deviation in the measurements is on the same order of magnitude as the mean, the average is not a representative metric. Consider taking steps to reduce noise in your measurements. If that is not possible, present your results as a combination of the key metrics such as mean, median, standard deviation, percentiles, min, max, etc.

*标准差*量化了数据集中的值与均值的平均偏差程度。较低的标准差表示数据点接近均值，而较高的标准差表示它们分布范围更广。除非分布的标准差较低，否则不要计算加速比。如果测量值的标准差与平均值处于同一数量级，则平均值不具有代表性。请考虑采取措施降低测量中的噪声。如果无法降低噪声，请将结果以关键指标的组合形式呈现，例如：平均值、中位数、标准差、百分位数、最小值、最大值等。

Performance gains are usually represented in two ways: as a speedup factor or percentage improvement. If a program originally took 10 seconds to run, and you optimized it down to 1 second, that's a 10x speedup. We shaved off 9 seconds of running time from the original program, that's a 90% reduction in time. The formula to calculate percentage improvement is shown below. In the book, we will use both ways of representing speedups.

性能提升通常以两种方式表示：加速比因子或百分比改进。如果一个程序最初运行时间为10秒，而你将其优化到1秒，则加速比为10倍。如果程序运行时间缩短了9秒，则时间减少了90%。计算百分比改进的公式如下所示。本书将使用这两种方式来表示加速。

$$
\textrm{Percentage Speedup} = (1 - \frac{\textrm{New Time}}{\textrm{Old Time}}) ~\times~100\%
$$

$$
\textrm{加速比百分比} = (1 - \frac{\textrm{新运行时间}}{\textrm{旧运行时间}}) ~\times~100\%
$$

One of the most important factors in calculating accurate speedup ratios is collecting a rich collection of samples, i.e., running a benchmark a large number of times. This may sound obvious, but it is not always achievable. For example, some of the [SPEC CPU 2017 benchmarks](http://spec.org/cpu2017/Docs/overview.html#benchmarks)[^1] run for more than 10 minutes on a modern machine. That means it would take 1 hour to produce just three samples: 30 minutes for each version of the program. Imagine that you have not just a single benchmark in your suite, but hundreds. It would become very expensive to collect statistically sufficient data even if you distribute the work across multiple machines.

计算准确加速比的关键因素之一是收集丰富的样本，即多次运行基准测试。这听起来显而易见，但并非总是可行。例如，[SPEC CPU 2017 基准测试](http://spec.org/cpu2017/Docs/overview.html#benchmarks)[^1]中的一些测试在现代机器上运行时间超过10分钟。这意味着仅生成3个样本就需要1小时：每个版本的程序运行30分钟。想象一下，如果你的测试套件中不止一个基准测试，而是数百个。即使将工作分配到多台机器上，收集足够的统计数据也会非常昂贵。

If obtaining new measurements is expensive, don't rush to collect many samples. Often you can learn a lot from just three runs. If you see a very low standard deviation within those three samples, you will probably learn nothing new from collecting more measurements. This is very typical of programs with underlying consistency (e.g., static benchmarks). However, if you see an abnormally high standard deviation, I do not recommend launching new runs and hoping to have "better statistics". You should figure out what is causing performance variance and how to reduce it.

如果获取新的测量数据成本很高，就不要急于收集大量样本。通常，只需3次运行就能获得很多信息。如果这3次运行的样本标准差非常低，那么收集更多数据可能并不会有什么新发现。这对于具有内在一致性的程序（例如：静态基准测试）来说非常典型。但是，如果标准差异常高，我不建议启动新的运行并寄希望于获得“更好的统计数据”。你应该找出性能差异的原因以及如何降低这种差异。

In an automated setting, you can implement an adaptive strategy by dynamically limiting the number of benchmark iterations based on standard deviation, i.e., you collect samples until you get a standard deviation that lies in a certain range. The lower the standard deviation in the distribution, the lower the number of samples you need. Once you have a standard deviation lower than the threshold, you can stop collecting measurements. This strategy is explained in more detail in [@Akinshin2019, Chapter 4].

在自动化环境中，您可以根据标准差动态限制基准测试的迭代次数，从而实现自适应策略。也就是说，您可以持续收集样本，直到标准差落在某个特定范围内。分布中的标准差越低，所需的样本数量就越少。一旦标准差低于阈值，就可以停止收集数据。此策略在[@Akinshin2019，第四章]中有更详细的解释。

An important thing to watch out for is the presence of outliers. It is OK to discard some samples (for example, cold runs) as outliers, but do not deliberately discard unwanted samples from the measurement set. Outliers can be one of the most important metrics for some types of benchmarks. For example, when benchmarking software that has real-time constraints, the 99 percentile could be very interesting.

需要注意的一个重要事项是异常值的存在。可以舍弃一些样本（例如：冷启动样本）作为异常值，但不要故意从测量集中剔除不需要的样本。异常值可能是某些类型基准测试中最重要的指标之一。例如，在对具有实时性约束的软件进行基准测试时，第99百分位数可能非常有用。

I recommend using benchmarking tools as they automate performance measurements. For example, Hyperfine[^4] is a popular cross platform command-line benchmarking tool that automatically determines the number of runs, and can visualize the results as a table with mean, min, max, or as a box plot.

我建议使用基准测试工具，因为它们可以自动测量性能。例如，Hyperfine[^4]是一款流行的跨平台命令行基准测试工具，它可以自动确定运行次数，并将结果可视化为包含均值、最小值和最大值的表格，或者以箱线图的形式呈现。

In the next two sections, we will discuss how to measure wall clock time (latency), which is the most common case. However, sometimes we also may want to measure other things, like the number of requests per second (throughput), heap allocations, context switches, etc.

在接下来的两节中，我们将讨论如何测量实际运行时间（延迟），这是最常见的情况。然而，有时我们也可能需要测量其他指标，例如每秒请求数（吞吐率）、堆分配、上下文切换等。

[^1]: SPEC CPU 2017 benchmarks SPEC CPU 2017基准测试 - [http://spec.org/cpu2017/Docs/overview.html#benchmarks](http://spec.org/cpu2017/Docs/overview.html#benchmarks)
[^12]: Book "Workload Modeling for Computer Systems Performance Evaluation" 书籍《计算机系统性能评估的工作负载建模》 - [https://www.cs.huji.ac.il/~feit/wlmod/](http://cs.huji.ac.il/~feit/wlmod/)
[^13]: Stefan Marr's blog post about box plots Stefan Marr关于箱线图的博客文章 - [https://stefan-marr.de/2024/06/5-reasons-for-box-plots-as-default/](https://stefan-marr.de/2024/06/5-reasons-for-box-plots-as-default/)
[^4]: hyperfine - [https://github.com/sharkdp/hyperfine](https://github.com/sharkdp/hyperfine)
