## Advanced Analysis Tools 高级分析工具

Many tools have been developed to address specific use cases where traditional profilers don't provide enough visibility. In this section, we will introduce Coz and eBPF tools. We encourage you to do further research on these and other tools.

许多工具的开发旨在解决传统性能轮廓分析器无法提供足够可见性的特定用例。在本节中，我们将介绍Coz和eBPF工具。我们鼓励您进一步研究这些工具以及其他工具。

### Coz {#sec:COZ}

In [@sec:secAmdahl], we defined the challenge of identifying parts of code that affect the overall performance of a multithreaded program. Due to various reasons, optimizing one part of a multithreaded program might not always give visible results. Traditional sampling-based profilers only show code places where most of the time is spent. However, it does not necessarily correspond to where programmers should focus their optimization efforts. 

在 [@sec:secAmdahl] 中，我们定义了识别影响多线程程序整体性能的代码部分的挑战。由于各种原因，优化多线程程序的某个部分可能并不总是能产生明显的效果。传统的基于采样的性能轮廓分析器仅显示代码中耗时最长的部分。然而，这并不一定与程序员应该将优化工作集中在哪些地方相对应。

[Coz](https://github.com/plasma-umass/coz)[^16] is a profiler that addresses this problem. It uses a novel technique called *causal profiling*, whereby experiments are conducted during the runtime of an application by virtually speeding up segments of code to predict the overall effect of certain optimizations. It accomplishes these “virtual speedups” by inserting pauses that slow down all other concurrently running code. Also, Coz quantifies the potential impact of an optimization. [@CozPaper]

[Coz](https://github.com/plasma-umass/coz)[^16] 是一款解决此问题的性能分析器。它使用了一种称为“*因果轮廓分析（causal profiling）*”的新技术，该技术在应用程序运行时通过虚拟加速代码片段来预测某些优化的总体效果。它通过插入暂停来减慢所有其他并发运行的代码，从而实现这些“虚拟加速（virtual speedups）”。此外，Coz还会量化优化的潜在影响[@CozPaper]。

An example of applying the Coz profiler to the [C-Ray](https://github.com/jtsiomb/c-ray)[^15] benchmark is shown in Figure @fig:CozProfile. According to the chart, if we improve the performance of line 540 in `c-ray-mt.c` by 20%, Coz expects a corresponding increase in application performance of the C-Ray benchmark overall of about 17%. Once we reach ~45% improvement on that line, the impact on the application begins to level off by Coz’s estimation. For more details on this example, see the [article](https://easyperf.net/blog/2020/02/26/coz-vs-sampling-profilers)[^17] on the Easyperf blog.

图 @fig:CozProfile 展示了将Coz轮廓分析器应用于[C-Ray](https://github.com/jtsiomb/c-ray)[^15]基准测试的示例。根据图表，如果我们将 `c-ray-mt.c` 文件中第540行的性能提高20%，Coz预计C-Ray基准测试的整体应用程序性能将相应提高约 17%。一旦该行的性能提升达到约45%，根据Coz的估计，对应用程序的影响将开始趋于平缓。有关此示例的更多详细信息，请参阅Easyperf博客上的[文章](https://easyperf.net/blog/2020/02/26/coz-vs-sampling-profilers)[^17]。

![Coz profile for the C-Ray benchmark. 针对C-Ray测试程序的Coz轮廓信息](../../img/mt-perf/CozProfile.png){#fig:CozProfile width=60%}

[^15]: C-Ray benchmark C-Ray测试程序 - [https://github.com/jtsiomb/c-ray](https://github.com/jtsiomb/c-ray).
[^16]: COZ source code COZ源代码 - [https://github.com/plasma-umass/coz](https://github.com/plasma-umass/coz).
[^17]: Blog article "COZ vs Sampling Profilers" 博客文章“COZ对比采样Profilers” - [https://easyperf.net/blog/2020/02/26/coz-vs-sampling-profilers](https://easyperf.net/blog/2020/02/26/coz-vs-sampling-profilers).

### eBPF and GAPP eBPF和GAPP {#sec:secEBPF}

Linux supports a variety of thread synchronization primitives: mutexes, semaphores, condition variables, etc. The kernel supports these thread primitives via the `futex` system call. Therefore, by tracing the execution of the `futex` system call in the kernel while gathering useful metadata from the threads involved, contention bottlenecks can be more readily identified. Linux provides kernel tracing and profiling tools that make this possible, none more powerful than [Extended Berkeley Packet Filter](https://prototype-kernel.readthedocs.io/en/latest/bpf/)[^22] (eBPF).

Linux支持多种线程同步原语：互斥锁（mutexes）、信号量（semaphores）、条件变量（condition variables）等。内核通过 `futex` 系统调用支持这些线程原语。因此，通过跟踪内核中 `futex` 系统调用的执行，并从相关线程收集有用的元数据，可以更容易地识别竞争瓶颈。Linux提供了内核跟踪和轮廓分析工具来实现这一点，其中最强大的工具莫过于扩展伯克利数据包过滤器[Extended Berkeley Packet Filter](https://prototype-kernel.readthedocs.io/en/latest/bpf/)[^22] (eBPF)。

eBPF is based around a sandboxed virtual machine running in the kernel that allows the execution of user-defined programs safely and efficiently inside the kernel. The user-defined programs can be written in C and compiled into BPF bytecode by the [BCC compiler](https://github.com/iovisor/bcc)[^23] in preparation for loading into the kernel virtual machine. These BPF programs can be written to launch upon the execution of certain kernel events and communicate raw or processed data back to userspace via a variety of means. 

eBPF基于运行在内核中的沙盒虚拟机，它允许用户在内核内部安全高效地执行自定义程序。用户自定义程序可以用C语言编写，并由 [BCC编译器](https://github.com/iovisor/bcc)[^23] 编译成BPF字节码，以便加载到内核虚拟机中。这些BPF程序可以编写成在特定内核事件执行时启动，并通过各种方式将原始数据或处理后的数据返回给用户空间。

The open-source community has provided many eBPF programs for general use. One such tool is the Generic Automatic Parallel Profiler (GAPP),[^25] which helps to track multithreaded contention issues. GAPP uses eBPF to track the contention overhead of a multithreaded application by ranking the criticality of identified serialization bottlenecks and it collects stack traces of threads that were blocked and the one that caused the blocking. The best thing about GAPP is that it does not require code changes, expensive instrumentation, or recompilation. Creators of the GAPP profiler were able to confirm known bottlenecks and also expose new, previously unreported ones in [Parsec 3.0 Benchmark Suite](https://parsec.cs.princeton.edu/index.htm)[^24] and some large open-source projects. [@GAPP]

开源社区提供了许多通用的eBPF程序。其中一个工具是通用自动并行轮廓分析器(GAPP: Generic Automatic Parallel Profiler)[^25]，它有助于跟踪多线程争用问题。GAPP使用eBPF通过对已识别的序列化瓶颈的严重程度进行排序来跟踪多线程应用程序的争用开销，并收集阻塞线程和导致阻塞线程的堆栈跟踪信息。GAPP的最大优点在于它不需要修改代码、进行昂贵的插桩或重新编译。GAPP分析器的创建者们不仅确认了[Parsec3.0基准测试套件](https://parsec.cs.princeton.edu/index.htm)[^24]和一些大型开源项目中已知的瓶颈，还发现了之前未报告的新瓶颈[@GAPP]。

As a closing thought, I would like to reemphasize the importance of optimizing multithreaded applications. From everything we have discussed in this book, advice from this chapter may bring the most significant performance improvements. In multithreaded applications, the devil is in the details. A subtle synchronization issue or a small inefficiency in data sharing can lead to significant performance degradation. As we look to the future, the trend toward many-core processors and parallel workloads will only accelerate. The complexity of multithreaded optimization will grow, but so will the opportunities for those who master it. 

最后，我想再次强调优化多线程应用程序的重要性。本书讨论的所有内容中，本章的建议或许能带来最显著的性能提升。在多线程应用程序中，细节决定成败。一个细微的同步问题或数据共享中的微小效率低下都可能导致性能的显著下降。展望未来，多核处理器和并行工作负载的趋势只会加速发展。多线程优化的复杂性将会增加，但掌握这项技术的人也将拥有更多机会。

[^22]: eBPF docs eBPF文档 - [https://prototype-kernel.readthedocs.io/en/latest/bpf/](https://prototype-kernel.readthedocs.io/en/latest/bpf/)
[^23]: BCC compiler BCC编译器 - [https://github.com/iovisor/bcc](https://github.com/iovisor/bcc)
[^24]: Parsec 3.0 Benchmark Suite Parse3.0测试程序集 - [https://parsec.cs.princeton.edu/index.htm](https://parsec.cs.princeton.edu/index.htm)
[^25]: GAPP - [https://github.com/RN-dev-repo/GAPP/](https://github.com/RN-dev-repo/GAPP/)