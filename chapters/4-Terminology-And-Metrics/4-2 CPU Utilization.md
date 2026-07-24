## CPU Utilization CPU利用率

CPU utilization is the percentage of time the core was busy during a time period. Technically, a CPU is considered utilized when it is not running the kernel `idle` thread.

CPU 利用率是指CPU核心在特定时间段内处于忙碌状态的时间百分比。从技术上讲，当CPU未运行内核“空闲idle”线程时，才算处于忙碌状态。

$$
CPU~Utilization = \frac{CPU\_CLK\_UNHALTED.REF\_TSC}{TSC},
$$

$$
CPU利用率 = \frac{CPU\_CLK\_UNHALTED.REF\_TSC}{TSC},
$$

where `CPU_CLK_UNHALTED.REF_TSC` counts the number of reference cycles when the core is not in a halt state. `TSC` stands for timestamp counter (discussed in [@sec:timers]), which is always ticking.

其中，`CPU_CLK_UNHALTED.REF_TSC` 统计的是CPU核心未处于停止状态时的引用周期数。`TSC`代表时间戳计数器（详见 [@sec:timers]），它始终处于滴答状态。

If CPU utilization is low, it usually translates into a poor performance of an application since a portion of time was wasted by a CPU. However, high CPU utilization is not always an indication of good performance. It is merely a sign that the system is doing some work but does not say what it is doing: the CPU might be highly utilized even though it is stalled waiting on memory accesses. In a multithreaded context, a thread can also spin while waiting for resources to proceed. Later, in [@sec:secMT_metrics], we will discuss parallel efficiency metrics, and in particular, take a look at "Effective CPU utilization" which filters spinning time.

如果CPU利用率低，通常意味着应用程序性能较差，因为CPU浪费了一部分时间。然而，高CPU利用率并不总是性能良好的标志。它仅仅表明系统正在执行某些工作，但并未说明具体执行的是哪些工作：即使CPU处于等待内存访问的状态，其利用率也可能很高。在多线程环境下，线程在等待资源时也可能处于空转状态。稍后，在 [@sec:secMT_metrics] 中，我们将讨论并行效率指标，特别是“有效CPU利用率”，它会过滤掉空转时间。

Linux `perf` automatically calculates CPU utilization across all CPUs on the system:

Linux `perf` 会自动计算系统中所有 CPU 的利用率：

```bash
$ perf stat -- a.exe
  0.634874  task-clock (msec) #    0.773 CPUs utilized   
```
