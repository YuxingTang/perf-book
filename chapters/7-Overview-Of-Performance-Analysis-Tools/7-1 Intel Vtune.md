## Intel VTune Profiler {#sec:IntelVtuneOverview}

VTune Profiler (formerly VTune Amplifier) is a performance analysis tool for x86-based machines with a rich graphical interface. It can be run on Linux or Windows operating systems. We skip discussion about MacOS support for VTune since it doesn't work on Apple's chips (e.g., M1 and M2), and Intel-based Macbooks are quickly becoming obsolete.

VTune Profiler（原名：VTune Amplifier）是一款适用于x86体系结构机器的性能分析工具，拥有丰富的图形界面。它可以在Linux或Windows操作系统上运行。由于VTune不支持苹果的芯片（例如：M1和M2），而且基于Intel处理器的MacBook也正在迅速被淘汰，因此我们不讨论macOS对VTune的支持情况。

VTune can be used on both Intel and AMD systems. However, advanced hardware-based sampling requires an Intel-manufactured CPU. For example, you won't be able to collect hardware performance counters on an AMD system with Intel VTune.

VTune可用于Intel和AMD系统。但是，高级的硬件采样功能需要Intel制造的CPU。例如，在AMD系统上使用 Intel VTune将无法收集硬件性能计数器。

As of early 2023, VTune is available for free as a stand-alone tool or as part of the Intel oneAPI Base Toolkit.[^1]

截至2023年初，VTune可作为独立工具免费使用，也可作为Intel oneAPI基础工具包的一部分使用[^1]。

### How to configure it 如何配置 {.unlisted .unnumbered}

On Linux, VTune can use two data collectors: Linux perf and VTune's own driver called SEP. The first type is used for user-mode sampling, but if you want to perform advanced analysis, you need to build and install the SEP driver, which is not too hard.

在Linux系统上，VTune可以使用两种数据采集器：Linux perf和VTune自带的名为SEP的驱动程序。前者用于用户模式采样，但如果要进行高级分析，则需要构建并安装SEP驱动程序，这并不难。

```bash
# go to the sepdk folder in vtune's installation
$ cd ~/intel/oneapi/vtune/latest/sepdk/src
# build the drivers
$ ./build-driver
# add vtune group and add your user to that group
# create a new shell, or reboot the system
$ sudo groupadd vtune
$ sudo usermod -a -G vtune `whoami`
# install sep driver
$ sudo ./insmod-sep -r -g vtune
```

After you've done with the steps above, you should be able to use advanced analysis types like Microarchitectural Exploration and Memory Access.

完成上述步骤后，您应该能够使用微体系结构探索和存储访问等高级分析类型。

Windows does not require any additional configuration after you install VTune. Collecting hardware performance events requires administrator privileges.

安装VTune后，Windows系统无需任何额外配置。收集硬件性能事件需要管理员权限。

### What you can do with it: VTune的功能：{.unlisted .unnumbered}

- Find hotspots: functions, loops, statements.
- Monitor various CPU-specific performance events, e.g., branch mispredictions and L3 cache misses.
- Locate lines of code where these events happen.
- Characterize CPU performance bottlenecks with TMA methodology.
- Filter data for a specific function, process, time period, or logical core.
- Observe the workload behavior over time (including CPU frequency, memory bandwidth utilization, etc).

- 查找性能热点：函数、循环、语句。
- 监控各种CPU特定性能事件，例如分支预测错误和 L3 缓存未命中。
- 定位这些事件发生的代码行。
- 使用TMA方法分析CPU性能瓶颈。
- 按特定函数、进程、时间段或逻辑核心筛选数据。
- 观察工作负载随时间的变化（包括：CPU频率、内存带宽利用率等）。

VTune can provide very rich information about a running process. It is the right tool for you if you're looking to improve the overall performance of an application. VTune always provides aggregated data over a period of time, so it can be used for finding optimization opportunities for the "average case". 

VTune可以提供关于运行进程的非常丰富的信息。如果您希望改进应用程序的整体性能，VTune是您的理想之选。 VTune始终提供一段时间内的聚合数据，因此可用于查找“平均情况”下的优化机会。

### What you cannot do with it: VTune不能做什么： {.unlisted .unnumbered}

- Analyze very short execution anomalies.
- Observe system-wide complicated software dynamics.

- 分析极短的执行异常。
- 观察系统级的复杂软件动态。

Due to the sampling nature of the tool, it will eventually miss events with a very short duration (e.g., sub-microsecond).

由于该工具的采样特性，它最终会遗漏持续时间极短（例如：亚微秒级）的事件。

### Example 示例 {.unlisted .unnumbered}

Below is a series of screenshots of VTune's most interesting features. For this example, I took POV-Ray, which is a ray tracer used to create 3D graphics. Figure @fig:VtuneHotspots shows the hotpots analysis of the built-in POV-Ray 3.7 benchmark, compiled with clang14 compiler with `-O3 -ffast-math -march=native -g` options, and run on an Intel Alder Lake system (Core i7-1260P, 4 P-cores + 8 E-cores) with 4 worker threads. 

以下是VTune一些最有趣功能的屏幕截图。本示例使用了POV-Ray，它是一款用于创建3D图形的光线追踪器。图 @fig:VtuneHotspots 显示了内置POV-Ray 3.7基准测试的热点分析，该基准测试使用clang14编译器编译，并带有 `-O3 -ffast-math -march=native -g` 选项，并在具有4个工作线程的Intel Alder Lake系统（Core i7-1260P，4个P核 + 8个E核）上运行。

![VTune's hotspots view of povray built-in benchmark. VTune的povray内置基准测试热点视图。](../../img/perf-tools/VtunePovray.png){#fig:VtuneHotspots width=100% }

![VTune's source code view of povray built-in benchmark. VTune的povray内置基准测试源代码视图。](../../img/perf-tools/VtunePovray_SourceView.png){#fig:VtuneSourceView width=100% }

At the left part of the image, you can see a list of hot functions in the workload along with the corresponding CPU time percentage and the number of retired instructions. On the right panel, you can see the most frequent call stack that leads to calling the function `pov::Noise`. According to that screenshot, `44.4%` of the time function `pov::Noise`, was called from `pov::Evaluate_TPat`, which in turn was called from `pov::Compute_Pigment`.[^20] 

在图像左侧，您可以看到工作负载中的热点函数列表，以及相应的CPU时间百分比和已执行指令数。在右侧面板中，您可以看到导致调用 `pov::Noise` 函数的最频繁调用堆栈。根据截图显示，`pov::Noise` 函数的44.4%时间是由 `pov::Evaluate_TPat` 调用的，而 `pov::Evaluate_TPat` 又是由 `pov::Compute_Pigment` 调用的[^20]。

If you double-click on the `pov::Noise` function, you will see an image that is shown in Figure @fig:VtuneSourceView. For the interest of space, only the most important columns are shown. The left panel shows the source code and CPU time that corresponds to each line of code. On the right, you can see assembly instructions along with the CPU time that was attributed to them. Highlighted machine instructions correspond to line 476 in the left panel. The sum of all CPU time percentages (not just the ones that are visible) in each panel equals to the total CPU time attributed to the `pov::Noise` function, which is `26.8%`.

双击 `pov::Noise` 函数，您将看到如图 @fig:VtuneSourceView 所示的图像。为了节省空间，图中仅显示了最重要的列。左侧面板显示了源代码以及每行代码对应的CPU时间。右侧面板显示了汇编指令以及分配给它们的CPU时间。高亮显示的机器指令对应于左侧面板中的第476行。每个面板中所有CPU时间百分比（不仅仅是可见的百分比）的总和等于分配给 `pov::Noise` 函数的总CPU时间，即26.8%。

![VTune's perf events timeline view of povray built-in benchmark. VTune内置Povray基准测试的性能事件时间线视图。](../../img/perf-tools/VtunePovray_EventTimeline.jpg){#fig:VtuneTimelineView width=100% }

When you use VTune to profile applications running on Intel CPUs, it can collect many different performance events. To illustrate this, I ran a different analysis type, Microarchitecture Exploration. To access raw event counts, you can switch the view to Hardware Events as shown in Figure @fig:VtuneTimelineView. To enable switching views, you need to tick the mark in *Options* &rarr; *General* &rarr; *Show all applicable viewpoints*. Near the top of Figure @fig:VtuneTimelineView, you can see that the *Platform* tab is selected. Two other pages are also useful. The *Summary* page gives you the absolute number of raw performance events as collected from CPU counters. The *Event Count* page gives you the same data with a per-function breakdown.

使用VTune对运行在Intel CPU上的应用程序进行轮廓分析时，它可以收集许多不同的性能事件。为了说明这一点，我运行了一种不同的分析类型：微体系结构探索。要访问原始事件计数，您可以将视图切换到“硬件事件”，如图 @fig:VtuneTimelineView 所示。要启用视图切换，您需要在“*选项Options*”->“*常规General*”->“*显示所有适用视点Show all applicable viewpoints*”中勾选相应的复选框。在图 @fig:VtuneTimelineView 的顶部附近，您可以看到已选中“*平台Platform*”选项卡。另外两个页面也很有用。“*摘要Summary*”页面会显示从CPU计数器收集的原始性能事件的绝对数量。 “*事件计数Event Count*”页面提供相同的数据，但按函数进行了细分。

Figure @fig:VtuneTimelineView is quite busy and requires some explanation. The top panel, indicated with \circled{1}, is a timeline view that shows the behavior of our four worker threads over time with respect to L1 cache misses, plus some tiny activity of the main thread (TID: `3102135`), which spawns all the worker threads. The higher the black bar, the more events (L1 cache misses in this case) happened at any given moment. Notice occasional spikes in L1 misses for all four worker threads. We can use this view to observe different or repeatable phases of the workload. Then to figure out which functions were executed at that time, we can select an interval and click "filter in" to focus just on that portion of the running time. The region indicated with \circled{2} is an example of such filtering. To see the updated list of functions, you can go to the *Event Count* view. Such filtering and zooming features are available on all VTune timeline views.

图 @fig:VtuneTimelineView 内容较为丰富，需要一些解释。顶部面板（标记为 \circled{1}）是一个时间线视图，显示了四个工作线程随时间推移的L1缓存未命中情况，以及主线程（TID：`3102135`）的一些少量活动，该线程会生成所有工作线程。黑色条形越高，表示在任何给定时刻发生的事件（在本例中为 L1缓存未命中）就越多。请注意，所有四个工作线程的L1缓存未命中次数偶尔会出现峰值。我们可以使用此视图来观察工作负载的不同或可重复的阶段。然后，为了确定在特定时间执行了哪些函数，我们可以选择一个时间间隔，然后单击“筛选”按钮，将注意力集中在运行时间的该部分。标记为 \circled{2} 的区域就是一个此类筛选的示例。要查看更新后的函数列表，您可以转到“事件计数”视图。所有VTune时间线视图均提供此类筛选和缩放功能。

The region indicated with \circled{3} shows performance events that were collected and their distribution over time. This time it is not a per-thread view, but rather it shows aggregated data across all the threads. In addition to observing execution phases, you can also visually extract some interesting information. For example, we can see that the number of executed branches is high (`BR_INST_RETIRED.ALL_BRANCHES`), but the misprediction rate is quite low (`BR_MISP_RETIRED.ALL_BRANCHES`). This can lead you to the conclusion that branch misprediction is not a bottleneck for POV-Ray. If you scroll down, you will see that the number of L3 misses is zero, and L2 cache misses are very rare as well. This tells us that 99% of memory access requests are served by L1, and the rest of them are served by L2. By combining these two observations, we can conclude that the application is likely bound by compute, i.e., the CPU is busy calculating something, not waiting for memory or recovering from a misprediction.

标有 \circled{3} 的区域显示了收集到的性能事件及其随时间变化的分布情况。这次显示的不是单个线程的视图，而是所有线程的聚合数据。除了观察执行阶段之外，您还可以直观地提取一些有趣的信息。例如，我们可以看到已执行的分支数量很高 (`BR_INST_RETIRED.ALL_BRANCHES`)，但分支预测错误率却很低 (`BR_MISP_RETIRED.ALL_BRANCHES`)。这可以得出结论：分支预测错误并非POV-Ray的瓶颈。向下滚动，您会看到L3缓存未命中次数为零，L2缓存未命中次数也非常少。这表明99%的存储访问请求由L1缓存处理，其余请求由L2缓存处理。结合以上两点观察，我们可以得出结论：该应用程序很可能受限于计算能力，也就是说，CPU正忙于计算，而不是在等待内存或从错误预测中恢复。

Finally, the bottom panel \circled{4} shows the CPU frequency chart for four hardware threads. Hovering over different time slices tells us that the frequency of those cores fluctuates in the 3.2--3.4GHz region.

最后，底部面板 \circled{4} 显示了四个硬件线程的CPU频率图。将鼠标悬停在不同的时间切片上，我们可以看到这些核心的频率在3.2GHz到3.4GHz之间波动。

[^1]: Intel oneAPI Base Toolkit Intel公司oneAPI基础工具包- [https://www.intel.com/content/www/us/en/developer/tools/oneapi/base-toolkit.html](https://www.intel.com/content/www/us/en/developer/tools/oneapi/base-toolkit.html)
[^3]: VTune microarchitecture analysis VTune微体系结构分析 - [https://software.intel.com/en-us/vtune-help-general-exploration-analysis](https://software.intel.com/en-us/vtune-help-general-exploration-analysis). In pre-2019 versions of Intel® VTune Profiler, it was called as “General Exploration” analysis. 在2019年之前的Intel® VTune Profiler版本中，它被称为“通用探索”分析。
[^4]: 7zip benchmark - [https://github.com/llvm-mirror/test-suite/tree/master/MultiSource/Benchmarks/7zip](https://github.com/llvm-mirror/test-suite/tree/master/MultiSource/Benchmarks/7zip).
[^19]: Per-function view of TMA metrics is a feature unique to Intel® VTune profiler. 按函数查看TMA指标是Intel® VTune轮廓分析器独有的功能。
[^20]: Notice that the call stack doesn't lead all the way to the `main` function. This happens because, with the hardware-based collection, VTune uses LBR to sample call stacks, which provides limited depth. Most likely we're dealing with recursive functions here, and to investigate that further users will have to dig into the code. 请注意，调用堆栈并未一直指向 `main` 函数。这是因为，在基于硬件的收集过程中，VTune使用LBR对调用堆栈进行采样，而LBR的采样深度有限。这里很可能存在递归函数，要进一步调查，用户需要深入研究代码。
