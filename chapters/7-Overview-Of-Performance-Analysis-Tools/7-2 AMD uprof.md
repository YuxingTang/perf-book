## AMD uProf

The [uProf](https://www.amd.com/en/developer/uprof.html) profiler is a tool developed by AMD for monitoring the performance of applications running on AMD processors. While uProf can be used on Intel processors as well, you will be able to use only CPU-independent features. The profiler is available for free to download and can be used on Windows, Linux, and FreeBSD. AMD uProf can be used for profiling on multiple virtual machines (VMs), including Microsoft Hyper-V, KVM, VMware ESXi, and Citrix Xen, but not all features are available on all VMs. Also, uProf supports analyzing applications written in various languages, including C, C++, Java, .NET/CLR.

[uProf](https://www.amd.com/en/developer/uprof.html)性能轮廓分析器是由AMD开发的一款工具，用于监控运行在AMD处理器上的应用程序的性能。虽然uProf也可以用于Intel处理器，但您只能使用与CPU无关的功能。该性能分析器可免费下载，并可在Windows、Linux和FreeBSD系统上运行。AMD uProf可用于分析多种虚拟机（VM: Virtual Machine）的性能，包括：Microsoft Hyper-V、KVM、VMware ESXi和Citrix Xen，但并非所有功能在所有虚拟机上都可用。此外，uProf支持分析使用多种语言编写的应用程序，包括C、C++、Java和.NET/CLR。

### How to configure it 如何配置 {.unlisted .unnumbered}

On Linux, uProf uses Linux perf for data collection. On Windows, uProf uses its own sampling driver that gets installed when you install uProf, no additional configuration is required. AMD uProf supports both command-line interface (CLI) and graphical interface. The CLI interface requires two separate steps---collect and report, similar to Linux perf.

在Linux系统上，uProf使用Linux perf进行数据收集。在Windows系统上，uProf使用其自身的采样驱动程序，该驱动程序会在安装uProf时自动安装，无需额外配置。AMD uProf同时支持命令行界面(CLI: Command Line Interface)和图形界面。CLI界面需要两个独立的步骤——收集数据和生成报告，类似于Linux上的perf命令。

### What you can do with it: 能用来做什么： {.unlisted .unnumbered}

- Find hotspots: functions, statements, instructions.
- Monitor various hardware performance events and locate lines of code where these events happen.
- Filter data for a specific function or thread.
- Observe the workload behavior over time: view various performance events in the timeline chart.
- Analyze hot callpaths: call graph, flame graph, and bottom-up charts.

- 查找性能热点：函数、语句、指令。
- 监控各种硬件性能事件，并定位这些事件发生的代码行。
- 筛选特定函数或线程的数据。
- 观察工作负载随时间的变化：在时间线图表中查看各种性能事件。
- 分析热点调用路径：调用图、火焰图和自底向上图表。

In addition, uProf can monitor various OS events on Linux: thread state, thread synchronization, system calls, page faults, and others. You can use it to analyze OpenMP applications to detect thread imbalance and analyze MPI[^3] applications to detect the load imbalance among the nodes of the MPI cluster. More details on various features of uProf can be found in the [User Guide](https://www.amd.com/en/developer/uprof.html#documentation)[^1].

此外，uProf还可以监控Linux系统上的各种操作系统事件：线程状态、线程同步、系统调用、页面错误等。您可以利用它分析OpenMP应用程序以检测线程不平衡，并分析MPI[^3]应用程序以检测MPI集群节点间的负载不平衡。有关uProf各项功能的更多详细信息，请参阅[用户指南](https://www.amd.com/en/developer/uprof.html#documentation)[^1]。

### What you cannot do with it: 不能用来做什么： {.unlisted .unnumbered}

Due to the sampling nature of the tool, it will eventually miss events with a very short duration. The reported samples are statistically estimated numbers, which are most of the time sufficient to analyze the performance but not the exact count of the events.

由于该工具采用采样方式，因此最终会遗漏持续时间极短的事件。报告的样本是统计估计值，通常足以分析性能，但并非事件的精确计数。

### Example 示例 {.unlisted .unnumbered}

To demonstrate the look-and-feel of the AMD uProf tool, we ran the dense LU matrix factorization component from the [Scimark2](https://math.nist.gov/scimark2/index.html)[^2] benchmark on an AMD Ryzen 9 7950X, running Windows 11, with 64 GB RAM.

为了演示AMD uProf工具的外观和操作方式，我们在一台配备AMD Ryzen 9 7950X处理器、Windows 11系统和 64GB内存的计算机上运行了[Scimark2](https://math.nist.gov/scimark2/index.html)[^2]基准测试中的密集LU矩阵分解组件。

![uProf's Function Hotspots view. uProf的函数热点视图.](../../img/perf-tools/uProf_Hopspot.png){#fig:uProfHotspots width=100% }

Figure @fig:uProfHotspots shows *Function Hotpots* analysis (selected in the menu list on the left side of the image). At the top of the image, you can see an event timeline showing the number of events observed at various times of the application execution. On the right, you can select which metric to plot; we selected `RETIRED_BR_INST_MISP`. Notice a spike in branch mispredictions in the time range from 20s to 40s. You can select this region to analyze closely what's going on there. Once you do that, it will update the bottom panels to show statistics only for that time interval.

图 @fig:uProfHotspots 显示了“函数热点”分析（在图像左侧的菜单列表中选择）。在图像顶部，您可以看到一个事件时间线，显示了在应用程序执行的不同时间点观察到的事件数量。在右侧，您可以选择要绘制的指标；我们选择了 `RETIRED_BR_INST_MISP`。请注意，在20秒到40秒的时间范围内，分支预测错误出现了一个峰值。您可以选择此区域，以便仔细分析其中发生的情况。选择后，底部面板将更新，仅显示该时间段的统计数据。

Below the timeline graph, you can see a list of hot functions, along with corresponding sampled performance events and calculated metrics. Event counts can be viewed as sample count, raw event count, or percentage. There are many interesting numbers to look at, but we will not dive deep into the analysis. Instead, readers are encouraged to figure out the performance impact of branch mispredictions and find their source.

在时间线图下方，您可以查看热门函数列表，以及相应的采样性能事件和计算指标。事件计数可以以样本计数、原始事件计数或百分比的形式查看。有很多有趣的数据值得关注，但我们不会深入分析。相反，我们鼓励读者自行找出分支预测错误对性能的影响并找到其根源。

Below the functions table, you can see a bottom-up call stack view for the selected function in the functions table. As we can see, the selected `LU_factor` function is called from `kernel_measureLU`, which in turn is called from `main`. In the Scimark2 benchmark, this is the only call stack for `LU_factor`, even though it shows `Call Stacks [5]`. This is an artifact of collection that can be ignored. But in other applications, a hot function can be called from many different places, so you would want to examine other call stacks as well. 

在函数表下方，您可以查看所选函数的自底向上调用堆栈视图。我们可以看到，所选的 `LU_factor` 函数是从 `kernel_measureLU` 函数调用的，而 `kernel_measureLU` 函数又是从 `main` 函数调用的。在Scimark2基准测试中，即使显示“`调用堆栈Call Stacks[5]`”，`LU_factor` 也只有这一个调用堆栈。这是数据收集过程中产生的瑕疵，可以忽略。但在其他应用程序中，一个热门函数可能被多个不同位置调用，因此您还需要检查其他调用堆栈。

If you double-click on any function, uProf will open the source/assembly view for that function. We don't show this view for brevity. On the left panel, there are other views available, like Metrics, Flame Graph, Call Graph view, and Thread Concurrency. They are useful for analysis as well, however we decided to skip them. Readers can experiment and look at those views on their own.

双击任何函数，uProf将打开该函数的源代码/汇编视图。为了简洁起见，我们在此不展示此视图。左侧面板还有其他视图可用，例如指标、火焰图、调用图和线程并发视图。它们也对分析很有用，但我们决定略过它们。读者可以自行尝试并查看这些视图。

[^1]: AMD uProf User Guide AMD uProf用户指南 - [https://www.amd.com/en/developer/uprof.html#documentation](https://www.amd.com/en/developer/uprof.html#documentation)
[^2]: Scimark2 - [https://math.nist.gov/scimark2/index.html](https://math.nist.gov/scimark2/index.html)
[^3]: MPI - Message Passing Interface, a standard for parallel programming on distributed memory systems. MPI--消息传递接口（Message Passing Interface），一种用于分布式内存系统并行编程的标准。
