## Apple Xcode Instruments 苹果的Xcode Instruments

The most convenient way to do similar performance analysis on MacOS is to use Xcode Instruments. This is an application performance analyzer and visualizer that comes for free with Xcode. The Instruments profiler is built on top of the DTrace tracing framework that was ported to MacOS from Solaris. It has many tools to inspect the performance of an application and enables us to do most of the basic things that other profilers like Intel VTune can do. The easiest way to get the profiler is to install Xcode from the Apple App Store. The tool requires no configuration; once you install it you're ready to go.

在MacOS上进行类似性能分析的最便捷方法是使用Xcode Instruments。这是Xcode免费提供的应用程序性能分析器和可视化工具。Instruments探查器构建在从Solaris移植到MacOS的DTrace跟踪框架之上。它拥有许多工具来检查应用程序的性能，并使我们能够完成Intel VTune等其他分析器可以完成的大多数基本操作。获取profiler轮廓分析器的最简单方法是从Apple App Store安装 Xcode。该工具无需配置；一旦安装完毕，您就可以开始使用了。

In Instruments, you use specialized tools, known as instruments, to trace different aspects of your apps, processes, and devices over time. Instruments has a powerful visualization mechanism. It collects data as it profiles and presents the results to you in real time. You can gather different types of data and view them side by side, which enables you to see patterns in the execution, correlate system events and find very subtle performance issues. 

在Instruments中，您可以使用称为仪器instruments的专用工具来跟踪应用程序、进程和设备随时间的不同方面。Instruments具有强大的可视化机制。它在轮廓分析时收集数据并将结果实时呈现给您。您可以收集不同类型的数据且以并排方式查看它们，这使您能够查看执行中的模式、关联系统事件并发现非常微妙的性能问题。

In this chapter, we will only showcase the "CPU Counters" instrument, which is the most relevant for this book. Instruments can also visualize GPU, network, and disk activity, track memory allocations, and releases, capture user events, such as mouse clicks, provide insights into power efficiency, and more. You can read more about those use cases in the Instruments [documentation](https://help.apple.com/instruments/mac/current).[^1]

在本章中，我们将仅展示与本书最相关的“处理器计数器CPU Counters”工具。Instruments还可以可视化GPU、网络和磁盘活动，跟踪内存分配和释放，捕获用户事件（例如鼠标单击），提供对电源效率的见解等。您可以在Instruments[文档](https://help.apple.com/instruments/mac/current) 中阅读有关这些用例的更多信息[^1]。

### What you can do with it: 你可以用它做什么：{.unlisted .unnumbered}

- Access hardware performance counters on Apple processors.
- Find hotspots in a program along with their call stacks.
- Inspect generated ARM assembly code side-by-side with the source code.
- Filter data for a selected interval on the timeline.

- 访问Apple处理器上的硬件性能计数器。
- 查找程序中的热点及其调用堆栈。
- 与源代码并排检查生成的ARM汇编代码。
- 过滤时间线上选定时间间隔的数据。

### What you cannot do with it: 你不能用它做什么：{.unlisted .unnumbered}

Similar to other sampling-based profilers, Xcode Instruments has the same blind spots as VTune and uProf.

与其他基于采样的轮廓分析器类似，Xcode Instruments与VTune和uProf具有相同的盲点。

### Example: Profiling Clang Compilation 示例：分析Clang编译 {.unlisted .unnumbered}

In this example, I will show how to collect hardware performance counters on an Apple Mac mini with the M1 processor, macOS 13.5.1 Ventura, and 16 GB RAM. I took one of the largest files in the LLVM codebase and profiled its compilation using version 15.0 of the Clang C++ compiler. 

在此示例中，我将展示如何在配备M1处理器、macOS 13.5.1 Ventura和16GB RAM的Apple Mac mini上收集硬件性能计数器。我以LLVM代码库中最大的文件之一，并使用15.0版本的Clang C++编译器，对它的编译情况进行轮廓分析。

![Xcode Instruments: timeline and statistics panels. Xcode Instruments：时间线和统计面板。](../../img/perf-tools/XcodeInstrumentsView.jpg){#fig:InstrumentsView width=100% }

Here is the command line that I used:

这是我使用的命令行：

```bash
$ clang++ -O3 -DNDEBUG -arch arm64 <other options ...> -c llvm/lib/Transforms/Vectorize/LoopVectorize.cpp
```

Figure @fig:InstrumentsView shows the main timeline view of Xcode Instruments. This screenshot was taken after the compilation had finished. We will get back to it a bit later, but first, let us show how to start the profiling session.

图 @fig:InstrumentsView 显示了Xcode Instruments的主时间线视图。此屏幕截图是在编译完成后拍摄的。我们稍后会再讨论这个问题，但首先，让我们展示如何启动分析会话。

To begin, open *Instruments* and choose the *CPU Counters* analysis type. The first step you need to do is configure the collection. Click and hold the red target icon (see \circled{1} in Figure @fig:InstrumentsView), then select *Recording Options...* from the menu. It will display the dialog window shown in Figure @fig:InstrumentsDialog. This is where you can add hardware performance monitoring events for collection. Apple has documented its hardware performance monitoring events in its manual [@AppleOptimizationGuide, Section 6.2 Performance Monitoring Events].

首先，打开 *Instruments* 并选择 *CPU Counters* 分析类型。您需要做的第一步是配置集合。单击并按住红色目标图标（参见图 @fig:InstrumentsView 中的 \circled{1}），然后从菜单中选择 *Recording Options...*。它将显示如图 @fig:InstrumentsDialog 所示的对话框窗口。您可以在此处添加硬件性能监控事件以进行收集。Apple在其手册 [@AppleOptimizationGuide，第6.2节性能监控事件] 中记录了其硬件性能监控事件。

![Xcode Instruments: CPU Counters options. Xcode Instruments：CPU 计数器选项。](../../img/perf-tools/XcodeInstrumentsDialog.png){#fig:InstrumentsDialog width=70% }

The second step is to set the profiling target. To do that, click and hold the name of an application (marked \circled{2} in Figure @fig:InstrumentsView) and choose the one you're interested in. Set the arguments and environment variables if needed. Now, you're ready to start the collection; press the red target icon \circled{1}.

第二步是设置轮廓分析目标。为此，请单击并按住应用程序的名称（在图 @fig:InstrumentsView 中标记为 \circled{2}），然后选择您感兴趣的应用程序。如果需要，请设置参数和环境变量。现在，您已准备好开始收集；按红色目标图标 \circled{1}。

Instruments shows a timeline and constantly updates statistics about the running application. Once the program finishes, Instruments will display the results like those shown in Figure @fig:InstrumentsView. The compilation took 7.3 seconds and we can see how the volume of events changed over time. For example, the number of executed branch instructions and mispredictions increased towards the end of the runtime. You can zoom in to that interval on the timeline to examine the functions involved.

Instruments显示时间线并不断更新有关正在运行的应用程序的统计信息。程序完成后，Instruments将显示如图 @fig:InstrumentsView 所示的结果。编译耗时7.3秒，我们可以看到事件量随时间的变化。例如，执行的分支指令和错误预测的数量在运行时结束时增加。您可以放大时间轴上的该间隔来检查所涉及的功能。

The bottom panel shows numerical statistics. To inspect the hotspots similar to Intel VTune's bottom-up view, select *Profile* in the menu \circled{3}, then click the *Call Tree* menu \circled{4} and check the *Invert Call Tree* box. This is exactly what we did in Figure @fig:InstrumentsView.

底部面板显示数字统计数据。要检查类似于Intel VTune自下而上视图的热点，请在菜单 \circled{3} 中选择 *Profile*，然后单击 *Call Tree* 菜单 \circled{4} 并选中 *Invert Call Tree* 框。这正是我们在图 @fig:InstrumentsView 中所做的。

Instruments show raw counts along with the percentages of the total, which is useful if you want to calculate secondary metrics like IPC, MPKI, etc. On the right side, we have the hottest call stack for the function `llvm::FoldingSetBase::FindNodeOrInsertPos`. If you double-click on a function, you can inspect ARM assembly instructions generated for the source code.

Instruments会显示原始计数以及占总数的百分比，这在计算IPC、MPKI等辅助指标时非常有用。右侧显示的是函数 `llvm::FoldingSetBase::FindNodeOrInsertPos` 的最热调用堆栈。双击某个函数，即可查看为源代码生成的ARM汇编指令。

To the best of my knowledge, there are no alternative profiling tools of similar quality available on MacOS platforms. Power users could use the `dtrace` framework itself by writing short (or long) command-line scripts, but a discussion of how to do so is beyond the scope of this book.

据我所知，macOS平台上没有其他质量相当的性能分析工具。高级用户可以通过编写简短（或冗长）的命令行脚本来使用 `dtrace` 框架本身，但具体操作方法超出了本书的范围。

[^1]: Instruments documentation Instruments文档 - [https://help.apple.com/instruments/mac/current](https://help.apple.com/instruments/mac/current)
