# Introduction {#sec:chapter1}

Performance is king: this was true a decade ago, and it certainly is now. According to [@Domo2017], in 2017 the world has been creating 2.5 quintillion[^1] bytes of data every day. [@Statista2024] predicts that number to reach 400 quintillion bytes per day in 2024. In our increasingly data-centric world, the growth of information exchange requires both faster software and faster hardware.

性能为王：在10多年前这就是真理，并且现在也是如此。根据文献[@Domo2017]，在2017年整个世界已经每天产生2.5百亿亿[^1]（Quintillion）字节的数据。[@Statista2024]预计这一数字在2024年会到达每天400百亿亿字节。在这个越来越以数据为中心的世界，信息交互的增长需求同时对更快的软件和更快的硬件提出了要求。

Software programmers have had an "easy ride" for decades, thanks to Moore’s law. Software vendors could rely on new generations of hardware to speed up their software products, even if they did not spend human resources on making improvements in their code. This strategy doesn't work any longer. By looking at Figure @fig:50YearsProcessorTrend, we can see that single-threaded[^2] performance growth is slowing down. From 1990 to 2000, single-threaded performance on SPECint benchmarks increased by a factor of approximately 25 to 30, driven largely by higher CPU frequencies and improved microarchitecture.

几十年来，得益于摩尔定律，软件程序员一直过着“轻松”的日子。软件供应商可以依靠新一代硬件来提升软件产品的运行速度，即使他们不投入人力资源来改进代码。但这种策略如今已不再奏效。从图@fig:50YearsProcessorTrend可以看出，单线程性能的增长速度正在放缓。从1990年到2000年，SPECint基准测试中的单线程性能提升了约25到30倍，这主要得益于更高的CPU频率和改进的微体系结构。

![50 Years of Microprocessor Trend Data. *© Image by K. Rupp via karlrupp.net*. Original data up to the year 2010 was collected and plotted by M. Horowitz, F. Labonte, O. Shacham, K. Olukotun, L. Hammond, and C. Batten. New plot and data collected for 2010-2021 by K. Rupp.](../../img/intro/50-years-processor-trend.png){#fig:50YearsProcessorTrend width=100%}

Single-threaded CPU performance growth was more modest from 2000 to 2010 (a factor between four and five). At that time, clock speeds topped out around 4GHz due to power consumption, heat dissipation challenges, limitations in voltage scaling (Dennard Scaling[^3]), and other fundamental problems. Despite clock speed stagnation, architectural advancements continued: better branch prediction, deeper pipelines, larger caches, and more efficient execution units.

从2000年到2010年，单线程CPU性能的增长较为温和（仅增长了4到5倍）。当时，由于功耗、散热难题、电压调节限制（丹纳德缩放Dennard Scaling[^3]）以及其他一些根本性问题，时钟频率最高只能达到4GHz左右。尽管时钟频率增长停滞，但体系结构改进仍在继续：更精确的分支预测、更深的流水线、更大的缓存以及更高效的执行单元。

From 2010 to 2020, single-threaded performance grew only by a factor between two and three. During this period, CPU manufacturers began to focus more on multi-core processors and parallelism rather than solely increasing single-threaded performance.

从2010年到2020年，单线程性能仅增长了2到3倍。在此期间，CPU制造商开始将更多精力放在多核处理器和并行处理上，而不是仅仅提升单线程性能。

Transistor counts continue to increase in modern processors. For instance, the number of transistors in Apple chips grew from 16 billion in M1 to 20 billion in M2, to 25 billion in M3, to 28 billion in M4 in a span of roughly four years. The growth in transistor count enables manufacturers to add more cores to a processor. As of 2024, you can buy a high-end server processor that will have more than 100 logical cores on a single CPU socket. This is very impressive. Unfortunately, it doesn't always translate into better performance. Very often, application performance doesn't scale with extra CPU cores.

现代处理器中的晶体管数量持续增长。例如，苹果芯片中的晶体管数量在短短4年内就从M1的160亿增长到M2的200亿，再到M3的250亿，最后到M4的280亿。晶体管数量的增长使制造商能够在处理器中添加更多核心。到2024年，您就可以购买到高端服务器处理器，其单个CPU插槽上将拥有超过100个逻辑核心。这令人印象深刻。然而，这并不总是能转化为更好的性能。很多时候，应用程序的性能并不会随着额外CPU核心数量增加而提升。

As it's no longer the case that each hardware generation provides a significant performance boost, we must start paying more attention to how fast our code runs. When seeking ways to improve performance, developers should not rely on hardware. Instead, they should start optimizing the code of their applications.

由于每一代硬件都无法带来显著的性能提升，我们必须开始更加关注代码的运行速度。在寻求提升性能的方法时，开发人员不应依赖硬件，而应着手优化应用程序的代码。

> “Software today is massively inefficient; it’s become prime time again for software programmers to get really good at optimization.” - Marc Andreessen, the US entrepreneur and investor (a16z Podcast)

> “如今的软件效率极低；软件程序员们再次迎来提升优化能力的关键时期。”——美国企业家兼投资人马克·安德森Marc Andreessen（a16z播客）

[^1]: A quintillion is a thousand raised to the power of six (10^18^).
[^1]: 百亿亿quintillion是千的6次方(10^18^)，也就是1后面18个零。
[^2]: Single-threaded performance is the performance of a single hardware thread inside a CPU core when measured in isolation.
[^2]：单线程性能是指在CPU核心内单独测量单个硬件线程的性能。
[^3]: Dennard Scaling - [https://en.wikipedia.org/wiki/Dennard_scaling](https://en.wikipedia.org/wiki/Dennard_scaling)
[^3]：丹纳德缩放（Dennard Scaling）——[参见英文维基百科对应页面](https://en.wikipedia.org/wiki/Dennard_scaling)
