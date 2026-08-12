## Parallel Efficiency Metrics 并行效率指标 {#sec:secMT_metrics}

Let's start by introducing a few metrics that are important for analyzing the performance of multithreaded applications. When dealing with multithreaded applications, engineers should be careful in analyzing basic metrics, for example, CPU utilization. One of the threads might show high CPU utilization, but it could turn out that the thread was just spinning in a busy-wait loop while waiting for a lock. That's why, when evaluating the parallel efficiency of an application, it's recommended to use *Effective CPU Utilization*, which is based only on the *Effective time*.

首先，我们来介绍一些对分析多线程应用程序性能至关重要的指标。在处理多线程应用程序时，工程师在分析基本指标（例如：CPU利用率）时应格外谨慎。某个线程可能显示CPU利用率很高，但实际上该线程可能只是在等待锁的过程中陷入了忙等待循环。因此，在评估应用程序的并行效率时，建议使用仅基于*有效时间（Effective time）*的“*有效CPU利用率（Effective CPU Utilization）*”。

### Effective CPU Utilization 有效CPU利用率 {.unlisted .unnumbered}

This metric represents how efficiently an application utilizes the available CPUs. It shows the percent of average CPU utilization by all logical CPUs on the system. It is based only on the *Effective time* and does not include the overhead introduced by the parallel runtime system[^11] and Spin time. An *Effective CPU utilization* of 100% means that your application keeps all the logical CPU cores busy for the entire time that it runs.

该指标表示应用程序对可用CPU的利用效率。它显示系统中所有逻辑CPU的平均CPU利用率百分比。它仅基于*有效时间*，不包含并行运行时系统引入的开销[^11]和自旋时间。100%的*有效CPU利用率*意味着您的应用程序在整个运行期间都繁忙的占用所有逻辑CPU核心。

For a specified time interval `T`, *Effective CPU Utilization* can be calculated as
对于指定的时间间隔 `T`，有效CPU利用率可以计算如下：

$$
\textrm{Effective CPU Utilization} = \frac{\sum_{i=1}^{\textrm{ThreadCount}}\textrm{Effective CPU Time(T,i)}}{\textrm{T}~\times~\textrm{ThreadCount}}
$$

$$
\textrm{Effective CPU Time} = \textrm{CPU Time}~-~(\textrm{Overhead Time}~+~\textrm{Spin Time})
$$

$$
\textrm{有效CPU利用率} = \frac{\sum_{i=1}^{\textrm{线程数}}\textrm{有效CPU时间(T,i)}}{\textrm{T}~\times~\textrm{线程数}}
$$

$$
\textrm{有效CPU时间} = \textrm{CPU时间}~-~(\textrm{开销时间OverheadTime}~+~\textrm{自旋时间SpinTime})
$$

Measuring overhead and spin time can be challenging, and I recommend using a performance analysis tool like Intel VTune Profiler, which can provide these metrics.

测量开销和自旋时间可能颇具挑战性，我建议使用Intel VTune Profiler等性能分析工具，它可以提供这些指标。

### Thread Count 线程数 {.unlisted .unnumbered}

Most parallel applications have a configurable number of threads, which allows them to run efficiently on platforms with a different number of cores. Running an application using a lower number of threads than is available on the system underutilizes its resources. On the other hand, running an excessive number of threads can cause *oversubscription*; some threads will be waiting for their turn to run.

大多数并行应用程序都具有可配置的线程数，这使得它们能够在具有不同核心数的平台上高效运行。如果应用程序使用的线程数少于系统可用线程数，则会造成资源利用不足。另一方面，运行过多的线程会导致*超订oversubscription*；一些线程将等待轮到自己运行。

Besides actual worker threads, multithreaded applications usually have other housekeeping threads: main thread, input/output threads, etc. If those threads consume significant time, they will take execution time away from worker threads, as they too require CPU cores to run. This is why it is important to know the total thread count and configure the number of worker threads properly.

除了实际的工作线程之外，多线程应用程序通常还有其他管理线程：主线程、输入/输出线程等。如果这些线程消耗大量时间，它们就会占用工作线程的执行时间，因为它们也需要CPU核心才能运行。因此，了解线程总数并正确配置工作线程数至关重要。

To avoid a penalty for thread creation and destruction, engineers usually allocate a [pool of threads](https://en.wikipedia.org/wiki/Thread_pool)[^14] with multiple threads waiting for tasks to be allocated for concurrent execution by the supervising program. This is especially beneficial for executing short-lived tasks.

为了避免线程创建和销毁带来的性能损失，工程师通常会分配一个线程池[pool of threads](https://en.wikipedia.org/wiki/Thread_pool)[^14]，其中包含多个等待主程序分配任务以进行并发执行的线程。这对于执行生命周期较短的任务尤其有利。

### Wait Time 等待时间 {.unlisted .unnumbered}

*Wait Time* occurs when software threads are waiting due to APIs that block or cause a context switch. Wait Time is per thread; therefore, the total *Wait Time* can exceed the application elapsed time.

当软件线程由于API阻塞或导致上下文切换而等待时，就会产生*等待时间WaitTime*。等待时间是按线程计算的；因此，总等待时间可能超过应用程序的运行时间。

A thread can be switched off from execution by the OS scheduler due to either synchronization or preemption. So, *Wait Time* can be further divided into *Sync Wait Time* and *Preemption Wait Time*. A large amount of *Sync Wait Time* likely indicates that the application has highly contended synchronization objects. We will explore how to find them in the following sections. Significant *Preemption Wait Time* can signal a thread oversubscription problem either because of a large number of application threads or a conflict with OS threads or other applications on the system. In this case, the developer should consider reducing the total number of threads or increasing task granularity for every worker thread.

线程可能由于同步或抢占而被操作系统调度程序停止执行。因此，*等待时间*可以进一步分为*同步等待时间（Sync Wait Time）*和*抢占等待时间（Preemption Wait Time）*。大量的*同步等待时间*可能表明应用程序存在大量争用的同步对象。我们将在以下章节中探讨如何查找这些对象。显著的*抢占等待时间*可能表明线程过度分配问题，原因可能是应用程序线程数量过多，也可能是与操作系统线程或系统上的其他应用程序发生冲突。在这种情况下，开发人员应考虑减少线程总数或增加每个工作线程的任务粒度。

### Spin Time 自旋时间 {.unlisted .unnumbered}

*Spin time* is *Wait Time*, during which the CPU is busy. This often occurs when a synchronization API causes the CPU to poll while the software thread is waiting. In reality, the implementation of kernel synchronization primitives spins on a lock for some time instead of immediately yielding to another thread. Too much Spin Time, however, can reflect the lost opportunity for productive work. 

*自旋时间SpinTime*是指CPU处于忙碌状态的*等待时间WaitTime*。这种情况通常发生在同步API导致CPU在软件线程等待时进行轮询时。实际上，内核同步原语的实现会在锁上自旋一段时间，而不是立即将锁让给其他线程。然而，过长的自旋时间可能意味着错失了进行有效工作的机会。

A list of other parallel efficiency metrics can be found on Intel's VTune [page](https://software.intel.com/en-us/vtune-help-cpu-metrics-reference).[^15]

其他并行效率指标列表可在Intel VTune页面[page](https://software.intel.com/en-us/vtune-help-cpu-metrics-reference)中找到[^15]。

[^11]: Threading libraries such as `pthread`, `OpenMP`, and `Intel TBB` incur additional overhead for creating and managing threads. 诸如 `pthread`、`OpenMP` 和 `Intel TBB` 之类的线程库在创建和管理线程时会产生额外的开销。
[^14]: Thread pool 线程池 - [https://en.wikipedia.org/wiki/Thread_pool](https://en.wikipedia.org/wiki/Thread_pool)
[^15]: CPU metrics reference CPU指标参考 - [https://software.intel.com/en-us/vtune-help-cpu-metrics-reference](https://software.intel.com/en-us/vtune-help-cpu-metrics-reference)