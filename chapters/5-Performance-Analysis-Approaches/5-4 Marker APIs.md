### Using Marker APIs 使用标记API {#sec:MarkerAPI}

In certain scenarios, we might be interested in analyzing the performance of a specific code region, not an entire application. This can be a situation when you're developing a new piece of code and want to focus just on that code. Naturally, you would like to track optimization progress and capture additional performance data that will help you along the way. Most performance analysis tools provide specific *marker APIs* that let you do that. Here are two examples:

在某些情况下，我们可能只想分析特定代码区域的性能，而不是整个应用程序的性能。例如，当您开发一段新代码并希望专注于该代码时，就可能遇到这种情况。当然，您也希望跟踪优化进度并收集有助于您改进的其他性能数据。大多数性能分析工具都提供了特定的 *标记API* 来实现这一点。以下是两个示例：

* Intel VTune has `__itt_task_begin / __itt_task_end` functions.
* AMD uProf has `amdProfileResume / amdProfilePause` functions.

* Intel VTune具有 `__itt_task_begin / __itt_task_end` 函数。
* AMD uProf具有 `amdProfileResume / amdProfilePause` 函数。

Such a hybrid approach combines the benefits of instrumentation and performance event counting. Instead of measuring the whole program, marker APIs allow us to attribute performance statistics to code regions (loops, functions) or functional pieces (remote procedure calls (RPCs), input events, etc.). The quality of the data you get back can easily justify the effort. For example, while investigating a performance bug that happens only with a specific type of RPC, you can enable monitoring just for that type of RPC.

这种混合方法结合了插桩和性能事件计数的优势。标记API允许我们将性能统计信息归因于代码区域（循环、函数）或功能片段（(远程过程调用RPC: Remote Procedure Call)、输入事件等），而不是测量整个程序。您获得的数据质量足以证明这项工作的价值。例如，在调查仅在特定类型的RPC调用中出现的性能缺陷时，您可以仅针对该类型的RPC启用监控。

Below we provide a very basic example of using [libpfm4](https://sourceforge.net/p/perfmon2/libpfm4/ci/master/tree/),[^1] one of the popular Linux libraries for collecting performance monitoring events. It is built on top of the Linux `perf_events` subsystem, which lets you access performance event counters directly. The `perf_events` subsystem is rather low-level, so the `libpfm4` package is useful here, as it adds both a discovery tool for identifying available events on your CPU and a wrapper library around the raw `perf_event_open` system call. The code listing below shows how you can use `libpfm4` to instrument the `render` function of the [C-Ray](https://openbenchmarking.org/test/pts/c-ray)[^2] benchmark.

下面我们提供一个使用 [libpfm4](https://sourceforge.net/p/perfmon2/libpfm4/ci/master/tree/)[^1],的基本示例，libpfm4是用于收集性能监控事件的常用Linux库之一。它基于Linux `perf_events` 子系统构建，允许您直接访问性能事件计数器。`perf_events` 子系统相当底层，因此 `libpfm4` 包在这里非常有用，因为它既提供了一个用于识别CPU上可用事件的发现工具，又提供了一个针对原始 `perf_event_open` 系统调用的包装库。下面的代码清单展示了如何使用 `libpfm4` 来检测 [C-Ray](https://openbenchmarking.org/test/pts/c-ray)[^2]基准测试的 `render` 函数。

```cpp
+#include <perfmon/pfmlib.h>
+#include <perfmon/pfmlib_perf_event.h>
...
/* render a frame of xsz/ysz dimensions into the provided framebuffer */
void render(int xsz, int ysz, uint32_t *fb, int samples) {
   ...
+  pfm_initialize();
+  struct perf_event_attr perf_attr;
+  memset(&perf_attr, 0, sizeof(perf_attr));
+  perf_attr.size = sizeof(struct perf_event_attr);
+  perf_attr.read_format = PERF_FORMAT_TOTAL_TIME_ENABLED | 
+                          PERF_FORMAT_TOTAL_TIME_RUNNING | PERF_FORMAT_GROUP;
+   
+  pfm_perf_encode_arg_t arg;
+  memset(&arg, 0, sizeof(pfm_perf_encode_arg_t));
+  arg.size = sizeof(pfm_perf_encode_arg_t);
+  arg.attr = &perf_attr;
+   
+  pfm_get_os_event_encoding("instructions", PFM_PLM3, PFM_OS_PERF_EVENT_EXT, &arg);
+  int leader_fd = perf_event_open(&perf_attr, 0, -1, -1, 0);
+  pfm_get_os_event_encoding("cycles", PFM_PLM3, PFM_OS_PERF_EVENT_EXT, &arg);
+  int event_fd = perf_event_open(&perf_attr, 0, -1, leader_fd, 0);
+  pfm_get_os_event_encoding("branches", PFM_PLM3, PFM_OS_PERF_EVENT_EXT, &arg);
+  event_fd = perf_event_open(&perf_attr, 0, -1, leader_fd, 0);
+  pfm_get_os_event_encoding("branch-misses", PFM_PLM3, PFM_OS_PERF_EVENT_EXT, &arg);
+  event_fd = perf_event_open(&perf_attr, 0, -1, leader_fd, 0);
+
+  struct read_format { uint64_t nr, time_enabled, time_running, values[4]; };
+  struct read_format before, after;

  for(j=0; j<ysz; j++) {
    for(i=0; i<xsz; i++) {
      double r = 0.0, g = 0.0, b = 0.0;
+     // capture counters before ray tracing
+     read(event_fd, &before, sizeof(struct read_format));

      for(s=0; s<samples; s++) {
        struct vec3 col = trace(get_primary_ray(i, j, s), 0);
        r += col.x;
        g += col.y;
        b += col.z;
      }
+     // capture counters after ray tracing
+     read(event_fd, &after, sizeof(struct read_format));

+     // save deltas in separate arrays
+     nanosecs[j * xsz + i] = after.time_running - before.time_running;
+     instrs  [j * xsz + i] = after.values[0] - before.values[0];
+     cycles  [j * xsz + i] = after.values[1] - before.values[1];
+     branches[j * xsz + i] = after.values[2] - before.values[2];
+     br_misps[j * xsz + i] = after.values[3] - before.values[3];

      *fb++ = ((uint32_t)(MIN(r * rcp_samples, 1.0) * 255.0) & 0xff) << RSHIFT |
              ((uint32_t)(MIN(g * rcp_samples, 1.0) * 255.0) & 0xff) << GSHIFT |
              ((uint32_t)(MIN(b * rcp_samples, 1.0) * 255.0) & 0xff) << BSHIFT;
  } }
+ // aggregate statistics and print it
  ...
}
```

In this code example, we first initialize the `libpfm` library and then configure performance events and the format that we will use to read them. In the C-Ray benchmark, the `render` function is only called once. In your own code, be careful about not doing `libpfm` initialization multiple times. 

在此代码示例中，我们首先初始化“libpfm”库，然后配置性能事件以及用于读取它们的格式。在C-Ray基准测试中，“render”函数仅被调用一次。在您自己的代码中，请注意不要多次执行“libpfm”初始化。

Then, we choose the code region we want to analyze. In our case, it is a loop with a `trace` function call inside. We surround this code region with two `read` system calls that will capture values of performance counters before and after the loop. Next, we save the deltas for later processing. In this case, we aggregated (code is not shown) it by calculating average, 90th percentile, and maximum values. Running it on an Intel Alder Lake machine, we get the output shown below. Root privileges are not required, but `/proc/sys/kernel/perf_event_paranoid` should be set to less than 1. When reading counters for a thread, the values are for that thread alone. It can optionally include kernel code that was attributed to the thread.

然后，我们选择要分析的代码区域。在我们的例子中，它是一个内部有“trace”函数调用的循环。我们2两个“read”系统调用包围该代码区域，这些系统调用将捕获循环之前和之后的性能计数器的值。接下来，我们保存增量以供以后处理。在本例中，我们通过计算平均值、第90个百分位数和最大值来聚合（这些代码没有展示）。在Intel Alder Lake机器上运行它，我们得到如下所示的输出。不需要root权限，但 `/proc/sys/kernel/perf_event_paranoid` 应设置为小于1。当从一个线程读取计数器时，这些值仅适用于该线程。它可以选择包含归属于线程的内核代码。

```bash
$ ./c-ray-f -s 1024x768 -r 2 -i sphfract -o output.ppm
Per-pixel ray tracing stats:
                      avg         p90         max
-------------------------------------------------
nanoseconds   |      4571 |      6139 |     25567
instructions  |     71927 |     96172 |    165608
cycles        |     20474 |     27837 |    118921
branches      |      5283 |      7061 |     12149
branch-misses |        18 |        35 |       146
```

Remember, that our instrumentation measures the per-pixel ray tracing stats. Multiplying average numbers by the number of pixels (`1024x768`) should give us roughly the total stats for the program. A good sanity check in this case is to run `perf stat` and compare the overall C-Ray statistics for the performance events that we've collected.

请记住，我们的检测工具测量的是每个像素的光线追踪统计数据。将平均值乘以像素数（`1024x768`）应该可以大致得出程序的总体统计数据。在这种情况下，一个很好的验证方法是运行 `perf stat` 命令，并比较我们收集到的性能事件的C-Ray总体统计数据。

The C-ray benchmark primarily stresses the floating-point performance of a CPU core, which generally should not cause high variance in the measurements, in other words, we expect all the measurements to be very close to each other. However, we see that it's not the case, as `p90` values are 1.33x average numbers and `max` is 5x slower than the average case. The most likely explanation here is that for some pixels the algorithm hits a corner case, executes more instructions, and subsequently runs longer. But it's always good to confirm a hypothesis by studying the source code or extending the instrumentation to capture more data for the "slow" pixels.

C-Ray基准测试主要考察CPU核心的浮点运算性能，这通常不会导致测量结果出现较大差异，换句话说，我们预期所有测量结果应该非常接近。然而，我们发现情况并非如此，`p90` 值是平均值的1.33倍，而 `max` 值比平均情况慢5倍。最可能的解释是，对于某些像素，算法遇到了边界情况，执行了更多指令，因此运行时间更长。但是，通过研究源代码或扩展检测工具来捕获更多“慢速”像素的数据来验证假设总是很有益的。

The additional instrumentation code in our example causes 17% overhead, which is OK for local experiments, but quite high to run in production. Most large distributed systems aim for less than 1% overhead, and for some up to 5% can be tolerable, but it's unlikely that users would be happy with a 17% slowdown. Managing the overhead of your instrumentation is critical, especially if you choose to enable it in a production environment.

示例中的额外插桩代码会造成17%的开销，这对于本地实验来说是可以接受的，但在生产环境中运行则相当高。大多数大型分布式系统的目标是将开销控制在1%以下，有些系统甚至可以容忍高达5%的开销，但用户不太可能乐意接受17%的性能下降。管理插桩的开销至关重要，尤其是在生产环境中启用插桩时。

Overhead is usefully calculated as the occurrence rate per unit of time or work (RPC, database query, loop iteration, etc.). If a `read` system call on our system takes roughly 1.6 microseconds of CPU time, and we call it twice for each pixel (iteration of the outer loop), the overhead is 3.2 microseconds of CPU time per pixel.

开销通常以单位时间或工作量（例如：RPC、数据库查询、循环迭代等）的发生率来计算。假设我们系统中的 `read` 系统调用大约需要1.6微秒的CPU时间，并且我们为每个像素调用两次（即外层循环的迭代），那么每个像素的开销就是3.2微秒的CPU时间。

There are many strategies to bring the overhead down. As a general rule, your instrumentation should always have a fixed cost, e.g., a deterministic syscall, but not a list traversal or dynamic memory allocation. It will otherwise interfere with the measurements. The instrumentation code has three logical parts: collecting the information, storing it, and reporting it. To lower the overhead of the first part (collection), we can decrease the sampling rate, e.g., sample each 10th RPC and skip the rest. For a long-running application, performance can be monitored with a relatively cheap random sampling, i.e., randomly select which RPCs to monitor for each sample. These methods sacrifice collection accuracy but still provide a good estimate of the overall performance characteristics while incurring a very low overhead.

有很多方法可以降低开销。一般来说，插桩应该始终具有固定的成本，例如确定性的系统调用，而不是列表遍历或动态内存分配。否则，它会干扰测量。检测代码包含三个逻辑部分：信息收集、信息存储和信息报告。为了降低第一部分（信息收集）的开销，我们可以降低采样率，例如，每10个RP 采样一次，其余的RPC则跳过。对于长时间运行的应用程序，可以使用相对低成本的随机采样来监控性能，即随机选择每个样本要监控的RPC。这些方法会牺牲信息收集的准确性，但仍然可以很好地估计整体性能特征，同时开销非常低。

For the second and third parts (storing and aggregating), the recommendation is to collect, process, and retain only as much data as you need to understand the performance of the system. You can avoid storing every sample in memory by using "online" algorithms for calculating mean, variance, min, max, and other metrics. This will drastically reduce the memory footprint of the instrumentation. For instance, variance and standard deviation can be calculated using Knuth's online-variance algorithm. A good implementation[^3] uses less than 50 bytes of memory.

对于第二部分和第三部分（存储和聚合），建议仅收集、处理和保留了解系统性能所需的数据量。您可以使用“在线”算法来计算均值、方差、最小值、最大值和其他指标，从而避免将每个样本都存储在内存中。这将大幅减少检测代码的内存占用。例如，可以使用Knuth的在线方差算法来计算方差和标准差。一个好的实现[^3]占用内存少于50字节。

For long routines, you can collect counters at the beginning and end, and some parts in the middle. Over consecutive runs, you can binary search for the part of the routine that performs most poorly and optimize it. Repeat this until all the poorly performing spots are removed. If tail latency is of primary concern, emitting log messages on a particularly slow run can provide useful insights.

对于较长的例程，您可以在开头和结尾收集计数器，并在中间部分收集计数器。在连续运行过程中，您可以对例程中性能最差的部分进行二分查找并进行优化。重复此操作，直到所有性能不佳的部分都被移除。如果尾延迟是主要关注点，那么在运行速度特别慢的情况下输出日志消息可以提供有用的信息。

In our example, we collected 4 events simultaneously, though the CPU has 6 programmable counters. You can open up additional groups with different sets of events enabled. The kernel will select different groups to run at a time. The `time_enabled` and `time_running` fields indicate the multiplexing. They both indicate duration in nanoseconds. The `time_enabled` field indicates how many nanoseconds the event group has been enabled. The `time_running` field indicates how much of that enabled time the events have been collected. If you had two event groups enabled simultaneously that couldn't fit together on the hardware counters, you might see the running time for both groups converge to `time_running = 0.5 * time_enabled`.

在我们的示例中，我们同时收集了4个事件，尽管 CPU有6个可编程计数器。您可以启用不同的事件组。内核会选择不同的事件组同时运行。`time_enabled` 和 `time_running` 字段指示多路复用情况。它们都以纳秒为单位指示持续时间。`time_enabled` 字段指示事件组已启用多少纳秒。`time_running` 字段指示事件已在启用时间内收集了多少纳秒。如果同时启用了两个事件组，而这两个事件组无法同时在硬件计数器上运行，则可能会看到两个事件组的运行时间收敛到 `time_running = 0.5 * time_enabled`。

Capturing multiple events simultaneously makes it possible to calculate various metrics that we discussed in Chapter 4. For example, capturing `INSTRUCTIONS_RETIRED` and `UNHALTED_CLOCK_CYCLES` enables us to measure IPC. We can observe the effects of frequency scaling by comparing CPU cycles (`UNHALTED_CORE_CYCLES`) with the fixed-frequency reference clock (`UNHALTED_REFERENCE_CYCLES`). It is possible to detect when the thread wasn't running by requesting CPU cycles consumed (`UNHALTED_CORE_CYCLES`, only counts when the thread is running) and comparing it against wall-clock time. Also, we can normalize the numbers to get the event rate per second/clock/instruction. For instance, by measuring `MEM_LOAD_RETIRED.L3_MISS` and `INSTRUCTIONS_RETIRED` we can get the `L3MPKI` metric. As you can see, this gives a lot of flexibility.

同时捕获多个事件可以计算我们在第四章讨论过的各种指标。例如，捕获 `INSTRUCTIONS_RETIRED` 和 `UNHALTED_CLOCK_CYCLES` 可以让我们测量IPC（每时钟周期数）。我们可以通过比较CPU周期数（`UNHALTED_CORE_CYCLES`）和固定频率的参考时钟周期数（`UNHALTED_REFERENCE_CYCLES`）来观察频率缩放的影响。通过请求消耗的CPU周期数（`UNHALTED_CORE_CYCLES`，仅在线程运行时计数）并将其与实际运行时间进行比较，可以检测线程何时未运行。此外，我们可以对这些数值进行归一化，以获得每秒/时钟/指令的事件发生率。例如，通过测量 `MEM_LOAD_RETIRED.L3_MISS` 和 `INSTRUCTIONS_RETIRED`，我们可以获得 `L3MPKI` 指标。如您所见，这提供了很大的灵活性。

The important property of grouping events is that the counters will be available atomically under the same `read` system call. These atomic bundles are very useful. First, it allows us to correlate events within each group. For example, let's assume we measure IPC for a region of code, and found that it is very low. In this case, we can pair two events (instructions and cycles) with a third one, say L3 cache misses, to check if this event contributes to the low IPC that we're dealing with. If it doesn't, we can continue factor analysis using other events. Second, event grouping helps to mitigate bias in case a workload has different phases. Since all the events within a group are measured at the same time, they always capture the same phase.

事件分组的重要特性在于，计数器在同一个 `read` 系统调用下可以原子地获取。这些原子性的事件包非常有用。首先，它允许我们关联每个组内的事件。例如，假设我们测量了一段代码的IPC，发现它非常低。在这种情况下，我们可以将两个事件（指令和周期）与第三个事件（例如：L3缓存未命中）配对，以检查该事件是否导致了我们遇到的低IPC。如果不是，我们可以继续使用其他事件进行因子分析。其次，事件分组有助于减少工作负载不同阶段带来的偏差。由于组内的所有事件都是同时测量的，因此它们始终捕获的是同一阶段。

In some scenarios, instrumentation may become a part of a functionality or a feature. For example, a developer can implement an instrumentation logic that detects a decrease in IPC (e.g., when there is a busy sibling hardware thread running) or decreasing CPU frequency (e.g., system throttling due to heavy load). When such an event occurs, the application automatically defers low-priority work to compensate for the temporarily increased load.

在某些情况下，插桩可能成为功能或特性的一部分。例如，开发人员可以实现一种检测逻辑，用于检测进程间通信(IPC)的下降（例如，当有繁忙的同级硬件线程正在运行时）或CPU频率的降低（例如：由于负载过重导致系统降频）。当发生此类事件时，应用程序会自动延迟执行低优先级任务，以补偿暂时增加的负载。

[^1]: libpfm4 - [https://sourceforge.net/p/perfmon2/libpfm4/ci/master/tree/](https://sourceforge.net/p/perfmon2/libpfm4/ci/master/tree/)

[^2]: C-Ray benchmark C-Ray基准测试 - [https://openbenchmarking.org/test/pts/c-ray](https://openbenchmarking.org/test/pts/c-ray)

[^3]: Accurately computing running variance 精确计算运行方差 - [https://www.johndcook.com/blog/standard_deviation/](https://www.johndcook.com/blog/standard_deviation/)
