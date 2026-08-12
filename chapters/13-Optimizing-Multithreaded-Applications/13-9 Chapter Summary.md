## Chapter Summary 本章小结 {.unlisted .unnumbered}

\markright{Summary小结}

* Applications that do not take advantage of modern multicore CPUs are lagging behind their competitors. Preparing software to scale well with a growing amount of CPU cores is very important for the future success of your application.
* When dealing with a single-threaded application, optimizing one portion of the program usually yields positive results on performance. However, this is not necessarily the case for multithreaded applications. This effect is widely known as Amdahl's law, which constitutes that the speedup of a parallel program is limited by its serial part.
* Thread communication can yield retrograde speedup as explained by Universal Scalability Law. This poses additional challenges for tuning multithreaded programs. 
* As we saw in our thread count case study, frequency throttling, memory bandwidth saturation, and other issues can lead to poor performance scaling.
* Task scheduling on hybrid processors is challenging. Watch out for suboptimal job scheduling and do not restrict the OS scheduler when it is not necessary.
* Optimizing the performance of multithreaded applications also involves detecting and mitigating the effects of cache coherence, such as true sharing and false sharing.
* During the past years, new tools emerged that cover gaps in analyzing performance of multithreaded applications, that traditional profilers cannot cover. We introduced Coz and GAPP which have a unique set of features.

* 未能充分利用现代多核CPU的应用正在落后于竞争对手。确保软件能够随着CPU核心数量的增长而良好扩展，对于应用的未来成功至关重要。
* 对于单线程应用，优化程序的一部分通常会带来性能提升。然而，对于多线程应用，情况并非总是如此。这种现象被称为阿姆达尔定律，它指出并行程序的加速比受限于其串行部分。
* 根据通用可扩展性定律，线程通信可能会导致速度下降。这给多线程程序的调优带来了额外的挑战。
* 正如我们在线程数案例研究中所看到的，频率节流限制、内存带宽饱和以及其他问题都可能导致性能扩展性差。
* 在混合处理器上进行任务调度极具挑战性。注意避免次优的作业调度，并在不必要的情况下不要限制操作系统调度程序。
* 优化多线程应用程序的性能还包括检测和缓解缓存一致性的影响，例如真共享和假共享。
* 近年来，涌现出一些新的工具，弥补了传统性能分析器在多线程应用程序性能分析方面的不足。我们介绍了Coz和GAPP，它们拥有独特的功能集。

\sectionbreak
