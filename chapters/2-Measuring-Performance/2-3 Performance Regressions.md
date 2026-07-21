## Continuous Benchmarking 持续基准测试

We just discussed why you should monitor performance in production. On the other hand, it is still beneficial to set up continuous "in-house" testing to catch performance problems early, even though not every performance regression can be caught in a lab.

我们刚刚讨论了为什么要在生产环境中监控性能。另一方面，即使并非所有性能退化都能在实验室环境中被发现，建立持续的“内部”测试来及早发现性能问题仍然十分有益。

Software vendors constantly seek ways to accelerate the pace of delivering their products to the market. Many companies deploy newly written code every couple of months or weeks. Unfortunately, software products don't get better performance with each new release. Performance defects tend to leak into production software at an alarming rate [@UnderstandingPerfRegress]. A large number of code changes pose a challenge to thorough analysis of their performance impact.

软件供应商不断寻求加快产品上市速度的方法。许多公司每隔几个月甚至几周就会部署新编写的代码。然而，软件产品的性能并不会随着每次新版本的发布而提升。性能缺陷往往会以惊人的速度渗入生产软件中 [@UnderstandingPerfRegress]。大量的代码变更使得对其进行性能影响的全面分析带来了挑战。

Performance regressions are defects that make the software run slower compared to the previous version. Catching performance regressions (or improvements) requires detecting the commit that has changed the performance of the program. From database systems to search engines to compilers, performance regressions are commonly experienced by almost all large-scale software systems during their continuous evolution and deployment life cycle. It may be impossible to entirely avoid performance regressions during software development, but with proper testing and diagnostic tools, the likelihood of such defects silently leaking into production code can be reduced significantly.

性能退化（Performance Regressions）是指导致软件运行速度比先前版本更慢的缺陷。发现性能退化（或性能提升）需要找到改变程序性能的提交。从数据库系统到搜索引擎再到编译器，几乎所有大型软件系统在其持续演进和部署的生命周期中都会遇到性能退化问题。软件开发过程中完全避免性能下降或许是不可能的，但借助适当的测试和诊断工具，可以显著降低此类缺陷悄然渗入生产代码的可能性。

It is useful to track the performance of your application with charts, like the one shown in Figure @fig:PerfRegress. Using such a chart you can see historical trends and find moments where performance improved or degraded. Typically, you will have a separate line for each performance test you're tracking. Do not include too many benchmarks on a single chart as it will become very noisy.

使用图表跟踪应用程序的性能非常有用，例如图 @fig:PerfRegress 中所示的图表。通过此类图表，您可以查看历史趋势，并找到性能提升或下降的时刻。通常，你需要为每个跟踪的性能测试绘制一条单独的曲线。请勿在单个图表中包含过多的基准测试，否则图表会变得非常嘈杂。

![Performance graph (higher better) for an application showing a big drop in performance on August 7th and smaller ones later.性能图表（数值越高越好）显示，某应用程序在8月7日性能大幅下降，之后性能下降幅度较小。](../../img/measurements/PerfRegressions.png){#fig:PerfRegress width=100%}

Let's consider some potential solutions for detecting performance regressions. The first option that comes to mind is: having humans look at the graphs. For the chart in Figure @fig:PerfRegress, humans will likely catch performance regression that happened on August 7th, but it's not obvious that they will detect later smaller regressions. People tend to lose focus quickly and can miss regressions, especially on a busy chart. In addition to that, it is a time-consuming and boring job that must be performed daily.

让我们来探讨一些检测性能下降的潜在解决方案。首先想到的方案是：人工查看图表。对于图 @fig:PerfRegress 中的图表，人工很可能能够发现8月7日发生的性能下降，但他们能否发现之后较小的下降则未必可行。人们很容易注意力分散，尤其是在图表内容繁杂的情况下，很容易忽略性能下降。此外，这是一项耗时且枯燥的工作，需要每天进行。

There is another interesting performance drop on August 3rd. A developer will also likely catch it, however, most of us would be tempted to dismiss it since performance recovered the next day. But are we sure that it was merely a glitch in measurements? What if this was a real regression that was compensated by an optimization on August 4th? If we could fix the regression *and* keep the optimization, we would have a performance score of around 4500. Do not dismiss such cases. One way to proceed here would be to repeat the measurements for the dates Aug 02--Aug 04 and inspect code changes during that period.

8月3日也出现了一次值得关注的性能下降。开发人员很可能也会注意到，但由于性能在第二天就恢复了，我们大多数人可能会忽略它。但我们真的确定这仅仅是测量误差吗？如果这是一次真正的性能下降，只是在8月4日通过优化得到了补偿呢？如果我们能够修复性能下降并保留优化，性能得分将达到4500分左右。不要忽视这种情况。一种方法是重复测量8月2日至8月4日期间的代码变更，并检查该期间的代码变更。

The second option is to have a threshold, say, 2%. Every code modification that has performance within that threshold is considered noise and everything above the threshold is considered a regression. It is somewhat better than the first option but still has its own drawbacks. Fluctuations in performance tests are inevitable: sometimes, even a harmless code change can trigger performance variation in a benchmark.[^3] Choosing the right value for the threshold is extremely hard and does not guarantee a low rate of false-positive as well as false-negative alarms. Setting the threshold too low might lead to analyzing a bunch of small regressions that were not caused by the change in source code but due to some random noise. Setting the threshold too high might lead to filtering out real performance regressions. 

第二种方法是设置一个阈值，例如：2%。性能变化低于该阈值的任何代码修改都被视为噪声，高于该阈值的任何修改都被视为性能退化。这种方法比第一种方法略好，但仍然存在一些缺点。性能测试中的波动是不可避免的：有时，即使是看似无害的代码变更也可能导致基准测试中的性能波动[^3]。选择合适的阈值非常困难，并且无法保证误报率和漏报率都很低。阈值设置过低可能会导致分析大量并非由源代码变更而是由某些随机噪声引起的小性能退化。阈值设置过高则可能导致过滤掉真正的性能退化。

Small regressions can pile up slowly into a bigger regression, which can be left unnoticed. Going back to Figure @fig:PerfRegress, notice a downward trend that lasted from Aug 11 to Aug 21. The period started with a score of 3000 and ended with 2600. That is roughly a 15% regression over 10 days or 1.5% per day on average. If we set a 2% threshold all regressions will be filtered out. However, as we can see, the accumulated regression is much bigger than the threshold. 

小的性能退化会逐渐累积成更大的性能退化，而这些退化可能被忽略。回到图 @fig:PerfRegress，注意从8月11日到8月21日期间的下降趋势。这段时间开始时的分数为3000，结束时为2600。这大约是10天内下降了15%，平均每天下降1.5%。如果我们设置2%的阈值，所有下降都会被过滤掉。然而，正如我们所见，累积的下降幅度远大于阈值。

Nevertheless, this option works reasonably well for many projects, especially if the level of noise in a benchmark is very low. Also, you can adjust the threshold for each test. An example of a Continuous Integration (CI) system where each test requires setting explicit threshold values for alerting a regression is [LUCI](https://chromium.googlesource.com/chromium/src.git/+/master/docs/tour_of_luci_ui.md),[^2] which is a part of the Chromium project.

尽管如此，对于许多项目来说，这个选项仍然相当有效，尤其是在基准测试中的噪声水平非常低的情况下。此外，你可以为每个测试调整阈值。[LUCI](https://chromium.googlesource.com/chromium/src.git/+/master/docs/tour_of_luci_ui.md)[^2] 就是一个持续集成(CI: Continuous Integration)系统的示例，其中每个测试都需要设置明确的阈值来发出下降警报，它是Chromium项目的一部分。

It's worth mentioning that tracking performance results over time requires that you maintain the same configuration of the machine(s) that you use to run benchmarks. A change in the configuration may invalidate all the previous performance results. You may decide to recollect all historical measurements with a new configuration, but this is very expensive.

值得一提的是，要长期跟踪性能结果，你需要保持用于运行基准测试的机器配置不变。配置的任何更改都可能导致之前所有性能结果失效。您可以选择在新配置下重新收集所有历史测量数据，但这成本非常高昂。

Another option that recently became popular uses a statistical approach to identify performance regressions. It leverages an algorithm called "Change Point Detection" (CPD, see [@ChangePointAnalysis]), which utilizes historical data and identifies points in time where performance has changed. Many performance monitoring systems embraced the CPD algorithm, including several open-source projects. You can search the web to find the one that better suits your needs.

另一种近期流行的方案是使用统计方法来识别性能退化。它利用一种名为“变化点检测”（CPD: Change Point Detection，参见[@ChangePointAnalysis]）的算法，该算法利用历史数据来识别性能发生变化的时间点。许多性能监控系统都采用了CPD算法，包括一些开源项目。你可以在网上搜索寻找更符合需求的方案。

The notable advantage of CPD is that it does not require setting thresholds. The algorithm evaluates a large window of recent results, which allows it to ignore outliers as noise and produce fewer false positives. The downside for CPD is the lack of immediate feedback. For example, consider a performance test with the following historical measurements of running time: 5 sec, 6 sec, 5 sec, 5 sec, 7 sec. If the next benchmark result comes at 11 seconds, then the threshold would likely be exceeded and an alert would be generated immediately. However, in the case of using the CPD algorithm, it wouldn't do anything at this point. If in the next run, performance is restored to 5 seconds, then it would likely dismiss it as a false positive and not generate an alert. Conversely, if the next run or two resulted in 10 sec and 12 sec respectively, only then would the CPD algorithm trigger an alert.

CPD的显著优势在于它不需要设置阈值。该算法会评估近期结果的较大窗口，从而能够将异常值视为噪声并减少误报。CPD的缺点是缺乏即时反馈。例如，假设进行一项性能测试，其历史运行时间分别为：5秒、6秒、5秒、5秒、7秒。如果下一次基准测试结果为11秒，则很可能超过阈值，系统会立即发出警报。但是，如果使用CPD算法，此时它不会采取任何行动。如果在下一次运行中，性能恢复到5秒，则算法可能会将其视为误报，而不会发出警报。相反，只有当接下来的几次运行结果分别为10秒和12秒时，CPD算法才会触发警报。

There is no clear answer to which approach is better. If your development flow requires immediate feedback, e.g., evaluating a pull request before it gets merged, then using thresholds is a better choice. Also, if you can remove a lot of noise from your system and achieve stable performance results, then using thresholds is more appropriate. In a very quiet system, the 11 second measurement mentioned before likely indicates a real performance regression, thus we need to flag it as early as possible. In contrast, if you have a lot of noise in your system, e.g., you run distributed macro-benchmarks, then that 11 second result may just be a false positive. In this case, you may be better off using Change Point Detection.

哪种方法更好并没有明确的答案。如果您的开发流程需要即时反馈，例如在合并拉取请求之前对其进行评估，那么使用阈值是更好的选择。此外，如果您能够消除系统中的大量噪声并获得稳定的性能结果，那么使用阈值也更合适。在非常安静的系统中，前面提到的11秒测量值很可能表明性能出现了真正的下降，因此我们需要尽早发出警报。相反，如果您的系统中存在大量噪声，例如您运行分布式宏基准测试，那么这11秒的结果可能只是一个误报。在这种情况下，您可能更适合使用变化点检测CPD。

A typical CI performance tracking system should automate the following actions:
一个典型的持续集成CI性能跟踪系统应该自动执行以下操作：

1. Set up a system under test.
2. Run a benchmark suite.
3. Report the results.
4. Determine if performance has changed.
5. Alert on unexpected changes in performance.
6. Visualize the results for a human to analyze.

1. 设置被测系统。
2. 运行基准测试套件。
3. 报告结果。
4. 确定性能是否发生变化。
5. 对性能的意外变化发出警报。
6. 将结果可视化，以便人工分析。

Another desirable feature of a CI performance tracking system is to allow developers to submit performance evaluation jobs for their patches before they commit them to the codebase. This greatly simplifies the developer's job and facilitates quicker turnaround of experiments. The performance impact of a code change is frequently included in the list of check-in criteria. 

CI性能跟踪系统的另一个理想特性是允许开发人员在将补丁提交到代码库之前提交性能评估任务。这极大地简化了开发人员的工作，并加快了实验的周转速度。代码变更的性能影响通常会被列入提交标准中。

If, for some reason, a performance regression has slipped into the codebase, it is very important to detect it promptly. First, because fewer changes were merged since it happened. This allows us to have a person responsible for the regression look into the problem before they move to another task. Also, it is a lot easier for a developer to approach the regression since all the details are still fresh in their head as opposed to several weeks after that.

如果由于某种原因，性能退化问题悄然进入代码库，及时发现至关重要。首先，因为自问题发生以来合并的变更较少。这使得负责该退化的人员能够在转而处理其他任务之前调查问题。此外，由于所有细节都还记忆犹新，开发人员更容易处理退化问题，而不是在几周之后才想起。

Lastly, the CI system should alert, not just on software performance regressions, but on unexpected performance improvements, too. For example, someone may check in a seemingly innocuous commit which improves performance by 10% in the automated tracking harness. Your initial instinct may be to celebrate this fortuitous performance boost and proceed with your day. However, while this commit may have passed all functional tests in your CI pipeline, chances are that this unexpected improvement uncovered a gap in functional testing which only manifested itself in performance regression results. For instance, the change caused the application to skip some parts of work, which was not covered by functional tests. This scenario occurs often enough that it warrants explicit mention: treat the automated performance regression harness as part of a holistic software testing framework.

最后，CI系统不仅应该对软件性能退化发出警报，还应该对意外的性能提升发出警报。例如，有人可能提交了一个看似无关紧要的提交，却使自动化跟踪工具中的性能提升了10%。你最初的想法可能是庆祝这一意外的性能提升，然后继续你的工作。然而，尽管这次提交可能通过了CI流水线中的所有功能测试，但这种意料之外的性能提升很可能暴露了功能测试中的漏洞，而这些漏洞仅体现在性能回归测试结果中。例如，此次变更导致应用程序跳过了部分工作，而这些工作并未被功能测试覆盖。这种情况经常发生，因此值得特别强调：请将自动化性能回归测试视为整体软件测试框架的一部分。

To wrap it up, we highly recommend setting up an automated statistical performance tracking system. Try using different algorithms and see which works best for your application. It will certainly take time, but it will be a solid investment in the future performance health of your project.

总而言之，我们强烈建议您搭建一个自动化的统计性能跟踪系统。尝试使用不同的算法，看看哪种最适合您的应用程序。这无疑需要花费一些时间，但对于项目未来的性能健康而言，这将是一项稳妥的投资。

[^2]: LUCI - [https://chromium.googlesource.com/chromium/src.git/+/master/docs/tour_of_luci_ui.md](https://chromium.googlesource.com/chromium/src.git/+/master/docs/tour_of_luci_ui.md)
[^3]: The following article shows that changing the order of the functions or removing dead functions can cause variations in performance: 以下文章表明，改变函数顺序或移除无用函数可能会导致性能变化：[https://easyperf.net/blog/2018/01/18/Code_alignment_issues](https://easyperf.net/blog/2018/01/18/Code_alignment_issues)
