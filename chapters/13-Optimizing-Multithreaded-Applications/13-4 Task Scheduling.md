## Task Scheduling 任务调度

With the emergence of hybrid processors, task scheduling becomes very challenging. For example, recent Intel's Meteor Lake chips have three types of cores; all with different performance characteristics. As you will see in this section, it is very easy to pessimize the performance of a multithreaded application by scheduling tasks suboptimally. Implementing a generic task scheduling policy is tricky because it greatly depends on the nature of the running tasks. Here are some examples:

随着混合体系结构处理器的出现，任务调度变得极具挑战性。例如，Intel最新的Meteor Lake芯片拥有三种类型的核心，每种核心的性能特性各不相同。正如您将在本节中看到的，如果任务调度不合理，很容易导致多线程应用程序的性能下降。实现通用的任务调度策略非常棘手，因为它很大程度上取决于正在运行的任务的性质。以下是一些示例：

* Compute-intensive lightly-threaded workloads (e.g., data compression) must be served only on P-cores.
* Background tasks (e.g., video calls) could be run on E-cores to save power.
* For bursty applications that demand high responsiveness (e.g., productivity software), a system should only use P-cores.
* Multithreaded programs with sustained performance demand (e.g., video rendering) should utilize both P- and E-cores.

* 计算密集型、轻线程工作负载（例如：数据压缩）必须仅在P核上运行。
* 后台任务（例如：视频通话）可以在E核上运行以节省功耗。
* 对于需要高响应速度的突发性应用程序（例如：生产力软件），系统应仅使用P核。
* 具有持续性能需求的多线程程序（例如：视频渲染）应同时利用P核和E核。

For the most part, task schedulers in modern operating systems take care of these and many other corner cases. For example, Intel's Thread Director helps monitor and analyze performance data in real-time to seamlessly place the right application thread on the right core. My general recommendation here is to let the operating system do its job and not restrict it too much. The operating system knows how to schedule tasks to minimize contention,  maximize reuse of data in caches, and ultimately maximize performance. This will play a big role if you are developing cross-platform software that is intended to run on different hardware configurations.

大多数情况下，现代操作系统中的任务调度器会处理这些以及许多其他特殊情况。例如，Intel的线程调度器(Thread Director)可以帮助实时监控和分析性能数据，从而将合适的应用程序线程无缝地分配到合适的核心上。我的建议是，让操作系统自行完成其工作，不要过多地限制它。操作系统知道如何调度任务以最大限度地减少争用，最大限度地提高缓存中数据的重用率，并最终最大限度地提高性能。如果您正在开发旨在运行于不同硬件配置上的跨平台软件，这一点将至关重要。

Below I show a few typical pitfalls of task scheduling in asymmetric systems. I took the same system I used in the previous case study: 12th Gen Alder Lake Intel&reg; Core&trade; i7-1260P CPU, which has four P-cores and eight E-cores. For simplicity, I only enabled two P-cores and two E-cores; the rest of the cores were temporarily disabled. I also disabled SMT sibling threads on the two active P-cores. I wrote a simple OpenMP application, where each worker thread runs several bit manipulation operations on every 32-bit integer element of a large array. After a worker thread has finished processing, it hits a barrier and is forced to wait for other threads to finish their parts. After that, the main thread cleans up the array and the processing repeats. The program was compiled with GCC 13.2 and `-O3 -march=core-avx2`, which enables vectorization.

下面我将展示非对称系统中任务调度的一些典型陷阱。我使用了与上一个案例研究相同的系统：第12代 Alder Lake Intel® Core™ i7-1260P CPU，它有4个P核和8个E核。为了简化操作，我只启用了2个P核和2个E核；其余核心暂时禁用。我还禁用了两个已激活P核心上的SMT同级线程。我编写了一个简单的OpenMP应用程序，其中每个工作线程对一个大型数组中的每个32位整数元素执行若干位操作。工作线程处理完毕后，会遇到阻塞，被迫等待其他线程完成各自的部分。之后，主线程清理数组，处理过程重复进行。该程序使用GCC 13.2和 `-O3 -march=core-avx2` 参数编译，启用了向量化。

Figure @fig:OmpScheduling shows three strategies, which highlight common problems that I regularly see in practice. These screenshots were captured with Intel VTune. The bars on the timeline indicate CPU time, i.e., periods when a thread was running. For each software thread, there is one or two corresponding CPU cores. Using this view, we can see on which core each thread was running at any given moment.

图 @fig:OmpScheduling 展示了三种策略，突出了我经常在实践中遇到的常见问题。这些截图是用Intel VTune截取的。时间轴上的条形图表示CPU时间，即线程运行的时间段。每个软件线程对​​应一个或两个CPU核心。通过这种视图，我们可以查看每个线程在任何给定时刻运行在哪个核心上。

\begin{figure}[htbp]
\centering

\subfloat[Static partitioning with pinning threads to the cores:
\passthrough{\lstinline!\#pragma omp for schedule(static)!} with
\passthrough{\lstinline!OMP\_PROC\_BIND=true!}.]{\includegraphics[width=0.8\textwidth,height=\textheight]{../../img/mt-perf/OmpAffinity.png}\label{fig:OmpAffinity}}

\subfloat[Static partitioning, no thread affinity:
\passthrough{\lstinline!\#pragma omp for schedule(static)!}.]{\includegraphics[width=0.8\textwidth,height=\textheight]{../../img/mt-perf/OmpStatic.png}\label{fig:OmpStatic}}

\subfloat[Dynamic partitioning with 16 chunks:
\passthrough{\lstinline!\#pragma omp for schedule(dynamic, N/16)!}.]{\includegraphics[width=0.8\textwidth,height=\textheight]{../../img/mt-perf/OmpDynamic.png}\label{fig:OmpDynamic}}

\caption{Typical task scheduling pitfalls: core affinity blocks thread
migration, partitioning jobs with large granularity fails to maximize
CPU utilization. 典型的任务调度陷阱：核心亲和性会阻碍线程迁移；对作业进行大粒度划分无法最大化CPU利用率。}

\label{fig:OmpScheduling}

\end{figure}

Our first example uses static partitioning, which divides the processing of our large array into four equal chunks (since I have four cores enabled). For each chunk, the OpenMP runtime spawns a new thread. Also, I used `OMP_PROC_BIND=true`, which instructs OpenMP runtime to pin spawned threads to the CPU cores. Figure @fig:OmpAffinity demonstrates the effect: P-cores are much better at handling SIMD instructions than E-cores and they finish their jobs two times faster (see *Thread 1* and *Thread 2*). However, thread affinity does not allow *Thread 3* and *Thread 4* to migrate to P-cores, which are waiting at the barrier. That results in a high latency, which is limited by the speed of E-cores.

我们的第一个示例使用了静态分区，将大型数组的处理分成4个相等的块（因为我启用了4个核心）。对于每个块，OpenMP运行时都会创建一个新线程。此外，我还使用了 `OMP_PROC_BIND=true`，它指示OpenMP运行时将创建的线程绑定到CPU核心。图 @fig:OmpAffinity 展示了其效果：P核心处理SIMD指令的能力远胜于E核心，并且它们完成任务的速度是E核心的2倍（参见 *线程一Thread 1* 和 *线程二Thread 2*）。然而，线程亲和性（Thread Affinity）不允许 *线程三Thread 3* 和 *线程四Thread 4* 迁移到P核，它们只能在屏障处等待。这导致了较高的延迟，而延迟又受限于E核的速度。

My recommendation is to avoid pinning threads to cores. With unbalanced work, pinning might restrict the work stealing, leaving the long execution tail for E-cores. On macOS, it is not possible to pin threads to cores since the operating system does not provide an API for that.

我的建议是避免将线程绑定到核心。在工作不均衡的情况下，绑定可能会限制工作窃取，从而导致E核承担较长的执行尾部。在macOS上，由于操作系统没有提供相应的API，因此无法将线程绑定到特定核心。

In the second example, I don't pin threads anymore, but the partitioning scheme remains the same (four equal chunks). Figure @fig:OmpStatic illustrates the effect of this change. As in the previous scenario, *Thread 1* and *Thread 4* finished their jobs early, because they were using P-cores. *Thread 2* and *Thread 3* started running on E-cores, but once P-cores became available, they migrated. It solved the problem we had before, but now E-cores remain idle until the end of the processing.

在第二个示例中，我不再绑定线程，但划分方案保持不变（4个大小相等的块）。图 @fig:OmpStatic 展示了这一更改的效果。与之前的场景一样，*线程一Thread 1* 和 *线程四Thread 4* 提前完成了任务，因为它们使用的是P核。*线程二Thread 2* 和 *线程三Thread 3* 最初在E核上运行，但一旦P核可用，它们就会迁移过去。这解决了之前的问题，但现在E核会一直处于空闲状态，直到处理结束。

My second piece of advice is to avoid static partitioning on systems with asymmetric cores. Equal-sized chunks will likely be processed faster on P-cores than on E-cores which will introduce dynamic load imbalance.

我的第二个建议是，在具有非对称核心的系统上避免使用静态分区。大小相等的块在P核心上的处理速度可能比在E核心上更快，这会导致动态负载不均衡。

In the final example, I switch to using dynamic partitioning. With dynamic partitioning, chunks are distributed to threads dynamically. Each thread processes a chunk of elements, then requests another chunk, until no chunks remain to be distributed. Figure @fig:OmpDynamic shows the result of using dynamic partitioning by dividing the array into 16 chunks. With this scheme, each task becomes more granular, which enables OpenMP runtime to balance the work even when P-cores run two times faster than E-cores. However, notice that there is still some idle time on E-cores. 

在最后一个示例中，我切换到使用动态划分。使用动态划分，块会动态地分配给各个线程。每个线程处理一部分元素，然后请求另一部分，直到没有剩余的元素块需要分配为止。图 @fig:OmpDynamic 展示了将数组划分为16个元素块后使用动态划分的结果。采用这种方案，每个任务的粒度都更细，即使P核的运行速度是E核的2倍，OpenMP运行时也能平衡工作负载。但是，请注意E核上仍然存在一些空闲时间。

Performance can be slightly improved if we partition the work into 128 chunks instead of 16. But don't make the jobs too small, otherwise it will result in increased management overhead. The result summary of my experiments is shown in Table @tbl:TaskSchedulingResults. Partitioning the work into 128 chunks turns out to be the sweet spot for our example. Even though this example is very simple, lessons from it can be applied to production-grade multithreaded software.

如果我们将工作划分为128个元素块而不是16个，性能可以略有提升。但不要将任务划分得太小，否则会导致管理开销增加。我的实验结果总结在表 @tbl:TaskSchedulingResults 中。结果表明，将工作划分为128个元素块是本示例的最佳方案。尽管这个示例非常简单，但从中汲取的经验教训可以应用于生产级多线程软件。

------------------------------------------------------------------------------------------------
                               Affinity  Static   Dynamic,    Dynamic,   Dynamic,    Dynamic,
                                                  4 chunks   16 chunks  128 chunks   1024 chunks
----------------------------- ---------- ------- ---------- ----------- ----------- ------------
Avg latency of 10 runs, ms     864       567       570         541        517          560

------------------------------------------------------------------------------------------------

Table: Results of the task scheduling experiments. 任务调度实验的结果 {#tbl:TaskSchedulingResults}
