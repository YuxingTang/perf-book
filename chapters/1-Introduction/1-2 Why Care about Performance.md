## Why Care about Performance? 为什么关心性能？

In addition to the slowing growth of hardware single-threaded performance, there are a couple of other business reasons to care about performance. During the PC era,[^12] the costs of slow software were paid by the users, as inefficient software was running on user computers. Software vendors were not directly incentivized to optimize the code of their applications. With the advent of SaaS (software as a service) and cloud computing, the costs of slow software are put back on the software providers, not their users. If you're a SaaS company like Meta or Netflix,[^4] it doesn't matter if you run your service on-premise hardware or you use the public cloud, you pay for the electricity your servers consume. Inefficient software cuts right into your margins and market valuation. According to Synergy Research Group,[^5] worldwide spending on cloud services topped $100 billion in 2020, and according to Gartner,[^6] it will surpass $675 billion in 2024.

除了硬件单线程性能增长放缓之外，还有其他一些商业原因需要关注性能。在个人电脑时代[^12]，软件运行缓慢的成本由用户承担，因为低效的软件运行在用户的计算机上。软件供应商并没有直接的动力去优化其应用程序的代码。随着软件即服务（SaaS: Software as a Service）和云计算的出现，软件运行缓慢的成本转回给了软件提供商，而不是用户。如果你是一家像Meta或Netflix[^4]这样的SaaS公司，无论你的服务是在本地硬件上运行还是使用公共云，你都需要为服务器消耗的电力付费。低效的软件会直接侵蚀你的利润和市场估值。根据Synergy Research Group[^5]的数据，2020年全球云服务支出超过1000亿美元，而Gartner[^6]预测，到2024年这一数字将超过6750亿美元。

For many years performance engineering was a nerdy niche, but now it's becoming mainstream. Many companies have already realized the importance of performance engineering and are willing to pay well for this work.

多年来，性能工程一直是一个略显冷门的领域，但现在它正逐渐成为主流。许多公司已经意识到性能工程的重要性，并愿意为此支付高额报酬。

It is fairly easy to reach performance level 4 in Table @tbl:PlentyOfRoom. In fact, you don't need this book to get there. Write your program in one of the native programming languages, distribute work among multiple threads, pick a good optimizing compiler and you'll get there. Unfortunately, the performance of your program will be about 200 times slower than the optimal target.

要达到表 @tbl:PlentyOfRoom 中的性能等级4相当容易。事实上，你甚至不需要这本书就能做到。只需使用一种原生编程语言编写程序，将任务分配到多个线程，选择一个优秀的优化编译器，就能达到目标性能。但遗憾的是，你的程序性能将比最佳目标慢大约200倍。

The methodologies in this book focus on squeezing out the last bit of performance from your application. Such transformations can be attributed along rows 6 and 7 in Table @tbl:PlentyOfRoom. The types of improvements that will be discussed are usually not big and often do not exceed 10%. However, do not underestimate the importance of a 10% speedup. SQLite is commonplace today not because its developers one day made it 50% faster, but because they meticulously made hundreds of 0.1% improvements over the years. The cumulative effect of these small improvements is what makes the difference.

本书中的方法论侧重于从应用程序中提取最后一丝性能。这些优化措施可以归因于表 @tbl:PlentyOfRoom 中的第6行和第7行。我们将要讨论的改进类型通常并不大，往往不超过10%。然而，千万不要低估10%的速度提升的重要性。SQLite如今如此普及，并非因为其开发者某天将其速度提升了50%，而是因为他们多年来一丝不苟地进行了数百次0.1%的改进。正是这些微小改进的累积效应造就了最终的成果。

The impact of small improvements is very relevant for large distributed applications running in the cloud. According to [@HennessyGoogleIO], in the year 2018, Google spent roughly the same amount of money on actual computing servers that run the cloud as it spent on power and cooling infrastructure. Energy efficiency is a very important problem, which can be improved by optimizing software.

对于运行在云端的大型分布式应用程序而言，微小改进的影响尤为显著。据[@HennessyGoogleIO]描述称，2018年，谷歌在运行云的实际计算服务器上的支出与在电力和冷却基础设施上的支出大致相当。能源效率是一个非常重要的问题，可以通过优化软件来提高。

>  "At such [Google] scale, understanding performance characteristics becomes critical---even small improvements in performance or utilization can translate into immense cost savings." [@GoogleProfiling]

> “在谷歌这样的规模下，了解性能特征至关重要——即使是性能或利用率的微小提升也能转化为巨大的成本节约。” [@GoogleProfiling]

In addition to cloud costs, there is another factor at play: how people perceive slow software. Google reported that a 500-millisecond delay in search caused a 20% reduction in traffic.[^3] For Yahoo! 400 milliseconds faster page load caused 5-9% more traffic.[^8] In the game of big numbers, small improvements can make a significant impact. Such examples prove that the slower a service works, the fewer people will use it. 

除了云成本之外，还有另一个因素在起作用：用户对软件运行速度慢的感知。谷歌报告称，搜索延迟500 毫秒会导致流量下降20%。[^3] 雅虎的页面加载速度提升400毫秒则能带来5-9%的流量增长。[^8] 在庞大的数据规模下，微小的改进也能产生显著的影响。这些例子证明，服务运行速度越慢，用户就越少。

Outside cloud services, there are many other performance-critical industries where performance engineering does not need to be justified, such as Artificial Intelligence (AI), High-Performance Computing (HPC), High-Frequency Trading (HFT), game development, etc. Moreover, performance is not only required in highly specialized areas, it is also relevant for general-purpose applications and services. Many tools that we use every day simply would not exist if they failed to meet their performance requirements. For example, Visual C++ IntelliSense[^2] features that are integrated into Microsoft Visual Studio IDE have very tight performance constraints. For the IntelliSense autocomplete feature to work, it must parse the entire source codebase in milliseconds.[^9] Nobody will use a source code editor if it takes several seconds to suggest autocomplete options. Such a feature has to be very responsive and provide valid continuations as the user types new code.

除了云服务之外，还有许多其他对性能要求极高的行业，例如：人工智能(AI: Artificial Intelligence)、高性能计算(HPC: High Performance Computing)、高频交易(HFT: High Frequency Trading)、游戏开发等，这些行业无需进行性能工程方面的论证。此外，性能不仅在高度专业化的领域至关重要，对于通用应用程序和服务也同样重要。许多我们日常使用的工具，如果无法满足性能要求，根本就不会存在。例如，集成到Microsoft Visual Studio IDE中的Visual C++ IntelliSense[^2]功能对性能要求非常严格。为了使IntelliSense自动补全功能正常工作，它必须在几毫秒内解析整个源代码库。[^9]如果自动补全选项需要几秒钟才能给出建议，那么没有人会使用源代码编辑器。此类功能必须响应迅速，并在用户输入新代码时提供有效的后续选项。

> "Not all fast software is world-class, but all world-class software is fast. Performance is _the_ killer feature." ---Tobi Lutke, CEO of Shopify.

> “并非所有运行速度快的软件都是世界一流的，但所有世界一流的软件都运行速度快。性能是杀手级特性。” ---Shopify 首席执行官Tobi Lutke

I hope it goes without saying that people hate using slow software, especially when their productivity goes down because of it. Table 1.2 shows that most people consider a delay of 2 seconds or more to be a "long wait," and would switch to something else after 10 seconds of waiting (I think much sooner). If you want to keep users' attention, your application must react quickly. 

人们讨厌使用运行缓慢的软件，尤其是在生产力因此下降的情况下，这一点应该无需赘述。表1.2显示，大多数人认为2秒或更长时间的延迟属于“长时间等待”，并且在等待10秒后就会切换到其他程序（我认为会更早）。如果你想吸引用户的注意力，你的应用程序必须反应迅速。

\small

-----------------------------------------------------------------------------
Interaction   Human Perception                                 Response Time
Class交互类型  人类感知                                           响应时间 

------------- -----------------------------------------------  --------------
Fast          Minimally noticeable delay                       100ms--200ms

Interactive   Quick, but too slow to be described as Fast      300ms--500ms
                
Pause         Not quick but still feels responsive             500ms--1 sec
               
Wait          Not quick due to amount of work for scenario     1 sec--3 sec
               
Long Wait     No longer feels responsive                       2 sec--5 sec

Captive       Reserved for unavoidably long/complex scenarios  5 sec--10 sec
               
Long-running  User will probably switch away during operation  10 sec--30 sec

------------------------------------------------------------------------------

Table: Human-software interaction classes. *Source: Microsoft Windows Blogs*.[^11] 人与软件交互分类。*源自：微软Windows博客*[^11]。 {#tbl:WindowsResponsiveness}

\normalsize

Application performance can drive your customers to a competitor's product. By emphasizing performance, you can give your product a competitive advantage.

应用程序性能（不佳）可能会导致客户转向竞争对手的产品。通过强调性能，你可以为你的产品赋予竞争优势。

Sometimes stfa tools find applications for which they were not initially designed. For example, game engines like Unreal and Unity are used in architecture, 3D visualization, filmmaking, and other areas. Because game engines are so performant, they are a natural choice for applications that require 2D and 3D rendering, physics simulation, collision detection, sound, animation, etc.

有时，高性能工具的应用场景并非其最初设计的目标。例如，Unreal和Unity等游戏引擎被广泛应用于建筑、3D可视化、电影制作等领域。由于游戏引擎性能卓越，它们自然成为需要2D和3D渲染、物理模拟、碰撞检测、声音、动画等功能的应用程序的理想选择。

> “Fast tools don’t just allow users to accomplish tasks faster; they allow users to accomplish entirely new types of tasks, in entirely new ways.” - Nelson Elhage wrote in his blog.[^1]

> “高性能工具不仅能让用户更快地完成任务，还能让用户以全新的方式完成全新的任务类型。”——Nelson Elhage在他的博客中写道。[^1]

Before starting performance-related work, make sure you have a strong reason to do so. Optimization just for optimization’s sake is useless if it doesn’t add value to your product.[^10] Mindful performance engineering starts with clearly defined performance goals. Understand clearly what you are trying to achieve, and justify the work. Establish metrics that you will use to measure success.

在开始性能优化工作之前，请确保您有充分的理由。为了优化而优化，如果不能为您的产品增添价值，那就毫无意义。[^10] 精心设计的性能工程始于明确定义性能目标。清楚地了解你想要实现的目标，并为这项工作提供合理的理由。建立衡量成功的指标。

Now that we've talked about the value of performance engineering, let's uncover what it consists of. When you're trying to improve the performance of a program, you need to find problems (performance analysis) and then improve them (tuning), a task very similar to a regular debugging activity. This is what we will discuss next.

既然我们已经讨论了性能工程的价值，接下来让我们了解一下它的具体内容。当你试图提升程序的性能时，你需要找出问题（性能分析），然后改进这些问题（调优），这与常规的调试工作非常相似。接下来我们将讨论的就是这一点。

[^12]: The late 1990s and early 2000s, a time when personal computers dominated the market of computing devices. 20世纪90年代末和21世纪初，个人电脑主导了计算设备市场。
[^4]: In 2024, Meta uses mostly on-premise cloud, while Netflix uses AWS public cloud. 2024年，Meta主要使用本地云，而网飞Netflix使用亚马逊AWS公有云。
[^5]: Worldwide spending on cloud services in 2020 2020年全球云服务支出 - [https://www.srgresearch.com/articles/2020-the-year-that-cloud-service-revenues-finally-dwarfed-enterprise-spending-on-data-centers](https://www.srgresearch.com/articles/2020-the-year-that-cloud-service-revenues-finally-dwarfed-enterprise-spending-on-data-centers)
[^6]: Worldwide spending on cloud services in 2024 2024年全球云服务支出 - [https://www.gartner.com/en/newsroom/press-releases/2024-05-20-gartner-forecasts-worldwide-public-cloud-end-user-spending-to-surpass-675-billion-in-2024](https://www.gartner.com/en/newsroom/press-releases/2024-05-20-gartner-forecasts-worldwide-public-cloud-end-user-spending-to-surpass-675-billion-in-2024)

[^1]: Reflections on software performance by N. Elhage N. Elhage的软件性能反思 - [https://blog.nelhage.com/post/reflections-on-performance/](https://blog.nelhage.com/post/reflections-on-performance/)
[^2]: Visual C++ IntelliSense - [https://docs.microsoft.com/en-us/visualstudio/ide/visual-cpp-intellisense](https://docs.microsoft.com/en-us/visualstudio/ide/visual-cpp-intellisense)
[^3]: Google I/O '08 Keynote by Marissa Mayer Marissa Mayer在 Google I/O '08 大会上的主题演讲 - [https://www.youtube.com/watch?v=6x0cAzQ7PVs](https://www.youtube.com/watch?v=6x0cAzQ7PVs)
[^8]: Slides by Stoyan Stefanov Stoyan Stefanov的幻灯片 - [https://www.slideshare.net/stoyan/dont-make-me-wait-or-building-highperformance-web-applications](https://www.slideshare.net/stoyan/dont-make-me-wait-or-building-highperformance-web-applications)
[^9]: In fact, it's not possible to parse the entire codebase in the order of milliseconds. Instead, IntelliSense only reconstructs the portions of AST that have been changed. Watch more details on how the Microsoft team achieves this in the video: 事实上，无法在毫秒级的尺度解析整个代码库。相反，IntelliSense只会重建已更改的AST部分。了解微软团队如何实现这一点的更多详细信息可以观看视频： [https://channel9.msdn.com/Blogs/Seth-Juarez/Anders-Hejlsberg-on-Modern-Compiler-Construction](https://channel9.msdn.com/Blogs/Seth-Juarez/Anders-Hejlsberg-on-Modern-Compiler-Construction)
[^10]: Unless you just want to practice performance optimizations, which is fine too. 除非你只是想练习性能优化，这也没问题。
[^11]: Microsoft Windows Blogs 微软Windows博客 - [https://blogs.windows.com/windowsdeveloper/2023/05/26/delivering-delightful-performance-for-more-than-one-billion-users-worldwide/](https://blogs.windows.com/windowsdeveloper/2023/05/26/delivering-delightful-performance-for-more-than-one-billion-users-worldwide/)
