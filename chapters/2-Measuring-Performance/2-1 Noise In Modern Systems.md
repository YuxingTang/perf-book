## Noise in Modern Systems 现代系统重的噪声 {#sec:secFairExperiments}

There are many features in hardware and software that are designed to increase performance, but not all of them have deterministic behavior. Consider Dynamic Frequency Scaling (DFS), a feature that allows a CPU to increase its frequency far above the base frequency, allowing it to run significantly faster. DFS is also frequently referred to as *turbo* mode. Unfortunately, a CPU cannot stay in the turbo mode for a long time, otherwise it may face the risk of overheating. So after some time, it decreases its frequency to stay within its thermal limits. DFS usually depends a lot on the current system load and external factors, such as core temperature, which makes it hard to predict the impact on performance measurements.

硬件和软件中有很多旨在提升性能的童年特性，但并非所有功能都具有确定性的行为。以动态频率调节(DFS: Dynamic Frequency Scaling)为例，该功能允许CPU将其频率提升到远高于基础频率的水平，从而显著提高运行速度。DFS也常被称为“*睿频加速turbo*”模式。然而，CPU无法长时间保持睿频加速模式，否则可能会面临过热的风险。因此，一段时间后，CPU会降低频率以保持在温度限制范围内。DFS通常很大程度上取决于当前系统负载和外部因素，例如：核心温度，这使得预测其对性能测量的影响变得困难。

Figure @fig:FreqScaling shows a typical example where DFS can cause variance in performance. In our scenario, we started two runs of a benchmark, one right after another on a "cold" processor.[^1] During the first second, the first iteration of the benchmark was running on the maximum turbo frequency of 4.4 GHz but later the CPU had to decrease its frequency below 4 GHz. The second run did not have the advantage of boosting the CPU frequency and did not enter the turbo mode. Even though we ran the exact same version of the benchmark two times, the environment in which they ran was not the same. As you can see, the first run is 200 milliseconds faster than the second run due to the fact that it was running with a higher CPU frequency in the beginning. Such a scenario can frequently happen when you benchmark software on a laptop since laptops have limited heat dissipation.[^4]

图 @fig:FreqScaling 展示了一个DFS可能导致性能波动的典型示例。在我们的测试场景中，我们在“冷”处理器上连续运行了两次基准测试[^1]。在第一次运行的前一秒，基准测试以最高睿频频率4.4GHz 运行，但随后CPU频率不得不降至4GHz以下。第二次运行没有利用CPU频率提升的优势，也没有进入睿频模式。尽管我们两次运行的是完全相同的基准测试版本，但它们的运行环境并不相同。正如您所看到的，由于第一次运行的CPU频率更高，因此它比第二次运行快了200毫秒。在笔记本电脑上进行软件基准测试时，由于笔记本电脑散热能力有限，这种情况经常发生[^4]。

![Variance in performance caused by dynamic frequency scaling: the first run is 200 milliseconds faster than the second.动态频率缩放导致的性能差异：第一次运行比第二次快 200 毫秒。](../../img/measurements/FreqScaling.jpg){#fig:FreqScaling width=90%}

Frequency Scaling is an example of how a hardware feature can cause variations in our measurements, however, they could also come from software. Consider benchmarking a `git status` command, which accesses many files on the disk. The filesystem plays a big role in performance in this scenario; in particular, the filesystem cache. On the first run, the required entries in the filesystem cache are missing. The filesystem cache is not effective and our `git status` command runs very slowly. However, the second time, the filesystem cache will be warmed up, making it much faster than the first run.

频率缩放是硬件特性如何导致测量结果出现差异的一个例子，但软件也可能导致这种差异。考虑对 `git status` 命令进行基准测试，该命令会访问磁盘上的许多文件。在这种情况下，文件系统对性能起着至关重要的作用；特别是文件系统缓存。在第一次运行时，文件系统缓存中缺少所需的条目。文件系统缓存无法有效工作，因此 `git status` 命令运行速度非常慢。但是，在第二次运行时，文件系统缓存已经预热，速度比第一次快得多。

You're probably thinking about including a dry run before taking measurements. That certainly helps, unfortunately, measurement bias can persist through the runs as well. [@Mytkowicz09] paper demonstrates that UNIX environment size (i.e., the total number of bytes required to store the environment variables) or the link order (the order of object files that are given to the linker) can affect performance in unpredictable ways. There are numerous other ways how memory layout may affect performance measurements.[^2]

您可能正在考虑在正式测量之前进行一次预热运行。这当然有所帮助，但遗憾的是，测量偏差也可能在多次运行中持续存在。[@Mytkowicz09] 的论文表明，UNIX环境大小（即：存储环境变量所需的总字节数）或链接顺序（传递给链接器的目标文件顺序）会以不可预测的方式影响性能。内存布局影响性能测量的方式还有很多[^2]。

Having consistent performance requires running all iterations of the benchmark with the same conditions. It is impossible to achieve 100% consistent results on every run of a benchmark, but perhaps you can get close by carefully controlling the environment. Eliminating nondeterminism in a system is helpful for well-defined, stable performance tests, e.g., microbenchmarks. 

要获得一致的性能，需要在相同的条件下运行基准测试的所有迭代。虽然不可能在每次基准测试运行中都获得100%一致的结果，但通过仔细控制环境，或许可以尽可能接近这个目标。消除系统中的不确定性对于定义明确、稳定的性能测试（例如微基准测试）非常有帮助。

Consider a situation when you implemented a code change and want to know the relative speedup ratio by benchmarking the "before" and "after" versions of the program. This is a scenario in which you can control most of the variability in a system, including HW configuration, OS settings, background processes, etc. Disabling features with nondeterministic performance impact will help you get a more consistent and accurate comparison. You can find examples of such features and how to disable them in Appendix A. Also, there are tools that can set up the environment to ensure benchmarking results with a low variance; one such tool is [temci](https://github.com/parttimenerd/temci)[^14].

设想这样一种情况：当你实现了代码更改，并希望通过对程序的“更改前”和“更改后”版本进行基准测试来了解相对加速比。在这种情况下，你可以控制系统中的大部分可变因素，包括硬件配置、操作系统设置、后台进程等等。禁用那些对性能有非确定性影响的功能特性将有助于您获得更一致、更准确的比较结果。您可以在附录A中找到此类功能特性的示例以及禁用方法。此外，还有一些工具可以设置环境，以确保基准测试结果具有较低的方差；[temci](https://github.com/parttimenerd/temci)就是这样一款工具[^14]。

However, it is not possible to replicate the exact same environment and eliminate bias completely: there could be different temperature conditions, power delivery spikes, unexpected system interrupts, etc. Chasing all potential sources of noise and variation in a system can be a never-ending story. Sometimes it cannot be achieved, for example, when you're benchmarking a large distributed cloud service.

然而，完全复制相同的环境并彻底消除偏差是不可能的：温度条件、电源波动、意外的系统中断等等都可能存在差异。追踪系统中所有潜在的噪声和变异来源可能是一个永无止境的过程。有时，这根本无法实现，例如，当您对大型分布式云服务进行基准测试时。

You should not eliminate system nondeterministic behavior when you want to measure the real-world performance impact of your change. Yes, these features may contribute to performance instabilities, but they are designed to improve the overall performance of the system. Users of your application are likely to have them enabled to provide better performance. So, when you analyze the performance of a production application, you should try to replicate the target system configuration, which you are optimizing for. Introducing any artificial tuning to the system will change results from what users of your service will see in practice.[^3]

当您想要衡量更改对实际性能的影响时，不应消除系统的非确定性行为。诚然，这些特性可能会导致性能不稳定，但它们可能就是被设计用来提升系统的整体性能。你的应用程序使用者很可能为了获得更好的性能而启用这些特性。因此，在分析生产应用程序的性能时，你应该尝试复现你正在优化的目标系统配置。任何人为地对系统进行调整都会改变结果，使其与用户实际体验到的性能有所不同。[^3]

[^1]: By cold processor, we mean the CPU that stayed in idle mode for a while, allowing it to cool down its temperature. 
[^1]：所谓“冷处理器”，指的是CPU处于空闲状态一段时间，使其温度得以降低。
[^2]: One approach to enable statistically sound performance analysis was presented in [@Curtsinger13]. This work showed that it's possible to eliminate measurement bias that comes from memory layout by repeatedly randomizing the placement of code, stack, and heap objects at runtime. Sadly, these ideas didn't go much further, and right now, this project is almost abandoned.
[^2]：[@Curtsinger13]提出了一种实现统计上可靠的性能分析的方法。这项工作表明，通过在运行时反复随机化代码、堆栈和堆对象的放置，可以消除由存储布局引起的测量偏差。遗憾的是，这些想法并未得到进一步发展，目前该项目几乎已被放弃。
[^3]: Another downside of disabling nondeterministic performance features is that it makes a benchmark run longer. This is especially important for Continuous Integration and Continuous Delivery (CI/CD) performance testing when there are time limits for how long it should take to run the whole benchmark suite.
[^3]：禁用非确定性性能特性的另一个缺点是会延长基准测试的运行时间。这对于持续集成和持续交付(CI/CD)性能测试尤为重要，有时CI/CD对运行整个基准测试套件的时间存在约束限制。
[^4]: Remember that even running Windows task manager or Linux `top` programs, can affect measurements since an additional CPU core will be activated and assigned to it. This might affect the frequency of the core that is running the actual benchmark.
[^4]：请记住，即使运行Windows任务管理器或Linux `top` 程序也会影响测量结果，因为额外的CPU核心会被激活并分配给它。这可能会影响运行实际基准测试的核心的频率。
[^14]: Temci - [https://github.com/parttimenerd/temci](https://github.com/parttimenerd/temci).
