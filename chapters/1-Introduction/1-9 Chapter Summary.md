## Chapter Summary {.unlisted .unnumbered} 本章小节

\markright{Summary}

* Single-threaded CPU performance is not increasing as rapidly as it used to a few decades ago. When it's no longer the case that each hardware generation provides a significant performance boost, developers should start optimizing the code of their software.
* 单线程CPU性能的增长速度已不如几十年前那么快。当每一代硬件都无法带来显著的性能提升时，开发者就应该开始优化软件代码。
* Modern software is massively inefficient. A regular server system in a public cloud, typically runs poorly optimized code, consuming more power than it could have consumed, which increases carbon emissions and contributes to other environmental issues.
* 现代软件效率极低。公共云中的通用服务器系统通常运行优化不佳的代码，消耗的电量远超其应有的水平，这不仅增加了碳排放，还加剧了其他环境问题。
* Certain limitations exist that prevent applications from reaching their full performance potential. CPUs cannot magically speed up slow algorithms. Compilers are far from generating optimal code for every program. Big O notation is not always a good indicator of performance as it doesn't account for hardware specifics.
* 某些限制阻碍了应用程序发挥其全部性能潜力。CPU无法神奇地加速低效算法。编译器也远未达到为每个程序生成最优代码的水平。大O表示法并非总是性能的良好指标，因为它没有考虑硬件的具体情况。
* For many years performance engineering was a nerdy niche. But now it's becoming mainstream as software vendors realize the impact that their poorly optimized software has on their bottom line.
* 多年来，性能工程一直是一个极客小众领域。但如今，随着软件供应商意识到其优化不佳的软件对利润的影响，性能工程正逐渐成为主流。
* People absolutely hate using slow software, especially when their productivity goes down because of it. Not all fast software is world-class, but all world-class software is fast. Performance is _the_ killer feature.
* 人们非常讨厌使用运行缓慢的软件，尤其是在生产力因此下降的情况下。并非所有运行快速的软件都是世界一流的，但所有世界一流的软件都运行快速。性能是*关键*所在。
* Software tuning is becoming more important than it has been for the last 40 years and it will be one of the key drivers for performance gains in the near future. The importance of low-level performance tuning should not be underestimated, even if it's just a 1% improvement. The cumulative effect of these small improvements is what makes the difference.
* 软件调优的重要性正变得远超过去40年，并将是未来提升性能的关键驱动力之一。即使只有1%的提升，底层性能调优的重要性也不容低估。正是这些微小改进的累积效应，最终才能带来显著的性能提升。
* To squeeze the last bit of performance you need to have a good mental model of how modern CPUs work.
* 要榨取最后一丝性能，你需要对现代CPU的工作原理有一个清晰的理解。
* Predicting the performance of a certain piece of code is nearly impossible since there are so many factors that affect the performance of modern platforms. When implementing software optimizations, developers should not rely on intuition but use careful performance analysis instead.
* 预测特定代码的性能几乎是不可能的，因为影响现代平台性能的因素太多了。在进行软件优化时，开发人员不应依赖直觉，而应进行细致的性能分析。

\sectionbreak
