## Performance Scaling in Multithreaded Programs 多线程程序中的性能扩展 {#sec:secAmdahl}

When dealing with a single-threaded application, optimizing one portion of a program usually yields positive results on performance. However, this is not necessarily the case for multithreaded applications. There could be an application in which thread `A` executes a long-running operation, while thread `B` finishes its task early and just waits for thread `A` to finish. No matter how much we improve thread `B`,  application latency will not be reduced since it will be limited by a longer-running thread `A`. 

对于单线程应用程序，优化程序的一部分通常会带来性能提升。然而，对于多线程应用程序，情况并非总是如此。例如，在这样的应用程序中，线程 `A` 执行耗时较长的操作，而线程 `B` 则提前完成任务，只需等待线程 `A` 完成即可。无论我们如何优化线程 `B` ，应用程序的延迟都不会降低，因为它仍然受限于运行时间较长的线程 `A`。

This effect is widely known as [Amdahl's law](https://en.wikipedia.org/wiki/Amdahl's_law),[^6] which constitutes that the speedup of a parallel program is limited by its serial part. Figure @fig:MT_AmdahlsLaw illustrates the theoretical speedup of the latency of the execution of a program as a function of the number of processors executing it. For a program, 75% of which is parallel, the speedup factor converges to 4.

这种现象被称为阿姆达尔定律（Amdahl's law）[^6]，它指出并行程序的加速比受限于其串行部分。图 @fig:MT_AmdahlsLaw 展示了程序执行延迟的理论加速比与执行该程序的处理器数量之间的关系。对于一个并行度为 75%的程序，加速因子会收敛到4。

<div id="fig:AmdahlUSLLaws">
![The theoretical speedup limit as a function of the number of processors, according to Amdahl's law. 根据阿姆达尔定律，理论加速极限与处理器数量的关系。](../../img/mt-perf/AmdahlsLaw.png){#fig:MT_AmdahlsLaw width=45%}
![Linear speedup, Amdahl's law, and Universal Scalability Law. 线性加速、阿姆达尔定律和通用可扩展性定律。](../../img/mt-perf/USL.png){#fig:MT_USL width=45%}

Amdahl's Law and Universal Scalability Law. 阿姆达尔定律和通用可扩展性定律。
</div>

In reality, further adding computing nodes to the system may yield retrograde speed up. We will see examples of it in the next section. This effect is explained by Neil Gunther as the [Universal Scalability Law](http://www.perfdynamics.com/Manifesto/USLscalability.html#tth_sEc1)[^8] (USL), which is an extension of Amdahl's law. USL describes communication between computing nodes (threads) as yet another factor gating performance. As the system is scaled up, overheads start to neutralize the gains. Beyond a critical point, the capability of the system starts to decrease (see Figure @fig:MT_USL). USL is widely used for modeling the capacity and scalability of systems.

实际上，向系统中进一步增加计算节点可能会导致加速倒退。我们将在下一节中看到一些例子。尼尔·冈瑟（Neil Gunther）将这种效应解释为通用可扩展性定律（USL: Universal Scalability law）[Universal Scalability Law](http://www.perfdynamics.com/Manifesto/USLscalability.html#tth_sEc1)[^8]，它是阿姆达尔定律的扩展。USL将计算节点（线程）之间的通信描述为影响性能的另一个关键因素。随着系统规模的扩大，开销开始抵消性能提升。超过某个临界点后，系统的性能开始下降（参见图 @fig:MT_USL）。USL被广泛用于系统容量和可扩展性的建模。

The slowdowns described by USL are driven by several factors. First, as the number of computing nodes increases, they start to compete for resources (contention). This results in additional time being spent on synchronizing those accesses. Another issue occurs with resources that are shared between many workers. We need to maintain a consistent state of the shared resources between many workers (coherence). For example, when multiple workers frequently change a globally visible object, those changes need to be broadcast to all nodes that use that object. Suddenly, usual operations start taking more time to finish due to the additional need to maintain coherence. Optimizing multithreaded applications not only involves all the techniques described in this book so far but also involves detecting and mitigating the aforementioned effects of contention and coherence.

USL所描述的性能下降是由多种因素驱动的。首先，随着计算节点数量的增加，它们开始争夺资源（竞争）。这会导致额外的时间用于同步这些访问。另一个问题是多个工作节点共享资源。我们需要维护多个工作节点之间共享资源的一致性（consistent）状态（相关一致性Coherence）。例如，当多个工作线程频繁修改一个全局可见的对象时，这些修改需要广播到所有使用该对象的节点。由于需要额外维护一致性，原本的操作完成时间会突然延长。优化多线程应用程序不仅涉及本书迄今为止描述的所有技术，还包括检测和缓解上述争用和一致性问题的影响。

[^6]: Amdahl's law 阿姆达尔定律 - [https://en.wikipedia.org/wiki/Amdahl's_law](https://en.wikipedia.org/wiki/Amdahl's_law).
[^8]: USL law USL定律 - [http://www.perfdynamics.com/Manifesto/USLscalability.html#tth_sEc1](http://www.perfdynamics.com/Manifesto/USLscalability.html#tth_sEc1).
