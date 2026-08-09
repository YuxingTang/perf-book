## Questions and Exercises 问题与习题 {.unlisted .unnumbered}

\markright{Questions and Exercises问题与习题}

1. Solve `perf-ninja::pgo` and `perf-ninja::lto` lab assignments.
2. Experiment with using Huge Pages for the code section. Take a large application (access to source code is a plus but not necessary), with a binary size of more than 100MB. Try to remap its code section onto huge pages using one of the methods described in [@sec:FeTLB]. Observe any changes in performance, huge page allocation in `/proc/meminfo`, and CPU performance counters that measure ITLB loads and misses.
3. Run the application that you're working with daily. Apply PGO, llvm-bolt, or Propeller and check the result. Compare "before" and "after" profiles to understand where the speedups are coming from.

1. 完成 `perf-ninja::pgo` 和 `perf-ninja::lto` 的实验任务。
2. 尝试在代码段中使用大页内存。选择一个大型应用程序（最好能访问源代码，但并非必须），其二进制文件大小超过 100MB。尝试使用 [@sec:FeTLB] 中描述的方法之一，将其代码段重新映射到大页内存。观察性能变化、`/proc/meminfo` 中的大页内存分配情况，以及用于衡量ITLB加载和失效的CPU性能计数器。
3. 运行你日常使用的应用程序。应用：PGO、llvm-bolt或Propeller并检查结果。比较“之前”和“之后”的性能分析，以了解速度提升的来源。