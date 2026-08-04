## Specialized and Hybrid Profilers 专用和混合型性能轮廓分析器 {#sec:Tracy}

Most of the tools explored so far fall under the category of sampling profilers. These are great when you want to identify hotspots in your code, but in some cases, they might not provide the required granularity for analysis. Depending on the profiler sampling frequency and the behavior of your program, most functions could be fast enough that they don't show up in a profiler. In some scenarios, you might want to manually define which parts of your program need to be measured consistently. Video games, for instance, render frames (the final image shown on screen) on average at 60 frames per second (FPS); some monitors allow up to 144 FPS. At 60 FPS, each frame has as little as 16 milliseconds to complete the work before moving on to the next one. Developers pay particular attention to frames that go above this threshold, as this causes visible stutter in games and can ruin the player experience. This situation is hard to capture with a sampling profiler.

目前为止，我们探讨的大多数工具都属于采样轮廓性能分析器。这类分析器非常适合识别代码中的性能瓶颈，但在某些情况下，它们可能无法提供分析所需的粒度。根据性能分析器的采样频率和程序的行为，大多数函数的运行速度可能非常快，以至于不会被轮廓性能分析器检测到。在某些情况下，您可能需要手动定义程序中哪些部分需要持续测量。例如，视频游戏平均以每秒60帧(FPS: Frames Per Second)的速度渲染帧（屏幕上显示的最终图像）；有些显示器最高支持144FPS。在60FPS下，每一帧完成渲染所需的时间仅为16毫秒，之后才会进入下一帧。开发人员会特别关注超过此阈值的帧，因为这会导致游戏中出现明显的卡顿，从而影响玩家体验。这种情况很难用采样性能分析器捕捉到。

Developers have created profilers that provide features helpful in specific environments, usually with a marker API that you can use to manually instrument your code. This enables you to observe the performance of a particular function or a block of code (later referred to as a *zone*). Continuing with the game industry, there are several tools in this space: some are integrated directly into game engines like Unreal, while others are provided as external libraries and tools that can be integrated into your project. Some of the most commonly used profilers are Tracy, RAD Telemetry, Remotery, and Optick (Windows only). Next, we showcase Tracy,[^1] as this seems to be one of the most popular projects; however, these concepts apply to the other profilers as well.

开发者创建了各种性能分析器，它们提供在特定环境下非常有用的功能，通常带有标记API，可用于手动插桩代码。这使您可以观察特定函数或代码块（以下简称“*区域Zone*”）的性能。在游戏行业，有多种工具可供选择：有些直接集成到Unreal等游戏引擎中，而另一些则作为外部库和工具提供，可以集成到您的项目中。一些最常用的性能分析器包括：Tracy、RAD Telemetry、Remotery和Optick（仅限Windows）。接下来，我们将重点介绍Tracy[^1]，因为它似乎是最流行的项目之一；不过，这些概念也适用于其他性能分析器。

### What you can do with Tracy: Tracy的功能： {.unlisted .unnumbered}

- Debug performance anomalies in a program, e.g., slow frames.
- Correlate slow events with other events in a system.
- Find common characteristics among slow events.
- Inspect source code and assembly.
- Do a "before/after" comparison after a code change.

- 调试程序中的性能异常，例如：很慢的帧。
- 将慢事件与系统中的其他事件关联起来。
- 查找慢事件的共同特征。
- 检查源代码和汇编代码。
- 代码更改后进行“前后对比”。

### What you cannot do with Tracy: Tracy的局限性： {.unlisted .unnumbered}

- Examine CPU microarchitectural issues, e.g., collect various performance counters.

- 无法检查CPU微体系结构问题，例如，收集各种性能计数器。

### Case Study: Analyzing Slow Frames with Tracy 案例研究：使用Tracy分析慢帧 {.unlisted .unnumbered}

In this example, we will use the ToyPathTracer[^2] program, a simple path tracer, which is a simplified ray-tracing technique that shoots thousands of rays per pixel into the scene to render a realistic image. To process a frame, the implementation distributes the processing of each row of pixels to a separate thread.

在本例中，我们将使用ToyPathTracer[^2]程序，这是一个简单的路径追踪器，它是一种简化的光线追踪技术，通过向场景中每个像素发射数千条光线来渲染逼真的图像。为了处理一帧，该实现将每行像素的处理分配给单独的线程。

To emulate a typical scenario where Tracy can help to diagnose the root cause of the problem, we have manually modified the code so that some frames will consume more time than others. [@lst:TracyInstrumentation] shows an outline of the code along with added Tracy instrumentation. Notice, that we randomly select frames to slow down. Also, we included Tracy's header and added the `ZoneScoped` and `FrameMark` macros to the functions that we want to track. The `FrameMark` macro can be inserted to identify individual frames in the profiler. The duration of each frame will be visible on the timeline, which is very useful.

为了仿真Tracy可以帮助诊断问题根本原因的典型场景，我们手动修改了代码，使某些帧的运行时间比其他帧更长。[@lst:TracyInstrumentation] 显示了代码的概要以及添加的Tracy检测。请注意，我们会随机选择帧进行慢放。此外，我们包含了Tracy的头文件，并在需要跟踪的函数中添加了 `ZoneScoped` 和 `FrameMark` 宏。`FrameMark` 宏可用于在分析器中识别单个帧。每个帧的持续时间将显示在时间轴上，这非常有用。

Listing: Tracy Instrumentation
代码列表：Tracy插桩

~~~~ {#lst:TracyInstrumentation .cpp}
#include "tracy/Tracy.hpp"

void DoExtraWork() {
  ZoneScoped;
  // imitate useful work
}

void TraceRowJob() {
  ZoneScoped;
  if (frameCount == randomlySelected)
    DoExtraWork();
  // ...
}

void RenderFrame() {
  ZoneScoped;
  for (...) {
    TraceRowJob();
  }
  FrameMark;
}
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Each frame can contain many zones, designated by the `ZoneScoped` macro. Similar to frames, there are many instances of a zone. Every time we enter a zone, Tracy captures statistics for a new instance of that zone. The `ZoneScoped` macro creates a C++ object on the stack that will record the runtime activity of the code within the scope of the object's lifetime. Tracy refers to this scope as a *zone*. At the zone entry, the current timestamp is captured. Once the function exits, the object's destructor will record a new timestamp and store this timing data, along with the function name.

每个帧可以包含多个区域，这些区域由 `ZoneScoped` 宏指定。与帧类似，一个区域可以有多个实例。每次进入一个区域时，Tracy都会捕获该区域新实例的统计信息。`ZoneScoped` 宏会在栈上创建一个C++对象，该对象会记录其生命周期范围内代码的运行时活动。Tracy将此范围称为“*区域zone*”。进入区域时，Tracy 捕获当前时间戳。函数退出后，对象的析构函数会记录一个新的时间戳，并将此计时数据以及函数名称一起存储。

Tracy has two operation modes: it can store all the timing data until the profiler is connected to the application (the default mode), or it can only start recording when a profiler is connected. The latter option can be enabled by specifying the `TRACY_ON_DEMAND` pre-processor macro when compiling the application. This mode should be preferred if you want to distribute an application that can be profiled as needed. With this option, the tracing code can be compiled into the application and it will cause little to no overhead to the running program unless the profiler is attached. The profiler is a separate application that connects to a running application to capture and display the live profiling data, also known as the "flight recorder" mode. The profiler can be run on a separate machine so that it doesn't interfere with the running application. Note, however, that this doesn't mean that the runtime overhead caused by the instrumentation code disappears. It is still there, but the overhead of visualizing the data is avoided in this case.

Tracy有两种运行模式：它可以存储所有计时数据，直到轮廓分析器连接到应用程序（默认模式）；或者，它也可以仅在分析器连接时才开始记录。后一种模式可以通过在编译应用程序时指定 `TRACY_ON_DEMAND` 预处理器宏来启用。如果您希望分发一个可以根据需要进行分析的应用程序，则应优先选择此模式。使用此选项，跟踪代码可以编译到应用程序中，除非附加了性能分析器，否则几乎不会对正在运行的程序造成任何额外开销。性能轮廓分析器是一个独立的应用程序，它连接到正在运行的应用程序，以捕获和显示实时性能分析数据，也称为“飞行记录器（flight recorder）”模式。性能轮廓分析器可以在单独的计算机上运行，​​因此不会干扰正在运行的应用程序。但是请注意，这并不意味着由检测代码引起的运行时开销会消失。它仍然存在，但在这种情况下避免了可视的数据开销。

We used Tracy to debug the program and find the reason why some frames are slower than others. The data was captured on a Windows 11 machine, equipped with a Ryzen 7 5800X processor. The program was compiled with MSVC 19.36.32532. Tracy's graphical interface is quite rich, but unfortunately contains too much detail to fit on a single screenshot, so we break it down into pieces. At the top, there is a timeline view as shown in Figure @fig:Tracy_Main_View, cropped to fit onto the page. It shows only a portion of frame 76, which took 44.1 ms to render. On that diagram, we see the `Main thread` and five `WorkerThread`s that were active during that frame. All threads, including the main thread, are performing work to advance progress in rendering the final image. As we said earlier, each thread processes a row of pixels inside the `TraceRowJob` zone. Each `TraceRowJob` zone instance contains many smaller zones, that are not visible. Tracy collapses inner zones and only shows the number of collapsed instances. This is what, for example, number `4,109` means under the first `TraceRowJob` in the Main Thread. Notice the instances of `DoExtraWork` zones, nested under `TraceRowJob` zones. This observation already can lead to a discovery, but in a real application, it may not be so obvious. Let's leave this for now.

我们使用Tracy调试程序，并找出某些帧比其他帧慢的原因。数据是在一台配备Ryzen 7 5800X处理器的Windows 11计算机上捕获的。程序使用MSVC 19.36.32532编译。Tracy的图形界面非常丰富，但遗憾的是，它包含的细节太多，无法在一张屏幕截图中显示，因此我们将其拆分成多个部分。顶部是一个时间线视图，如图 @fig:Tracy_Main_View 所示，为了适应页面，内容已裁剪。它仅显示了第76帧的一部分，该帧的渲染耗时44.1毫秒。在该图中，我们可以看到在该帧期间处于活动状态的 `主线程` 和5个 `工作线程` 。所有线程（包括主线程）都在执行工作，以推进最终图像的渲染进度。正如我们之前提到的，每个线程处理 `TraceRowJob` 区域内的一行像素。每个 `TraceRowJob` 区域实例包含许多更小的区域，这些区域不可见。Tracy会折叠内部区域，仅显示折叠后的实例数量。例如，主线程中第一个 `TraceRowJob` 下的数字 `4,109` 就代表了这一点。请注意嵌套在 `TraceRowJob` 区域下的 `DoExtraWork` 区域实例。这一观察结果本身就可能有助于发现问题，但在实际应用中，它可能并不那么显而易见。我们先到这里。

![Tracy main timeline view. It shows the main thread and five worker threads while rendering a frame. Tracy主时间线视图。它显示了渲染帧时的主线程和5个工作线程。](../../img/perf-tools/tracy/tracy_main_timeline.png){#fig:Tracy_Main_View width=100%}

Right above the main panel, there is a histogram that displays the times for all the recorded frames (see Figure @fig:Tracy_Frame_Time_View). It makes it easier to spot those frames that took longer than average to complete. In this example, most frames take around 33 ms (the yellow bars). However, some frames take longer than this and are marked in red. As seen in the screenshot, a tooltip showing the details of a given frame is displayed when you point the mouse at a bar in the histogram. In this example, we are showing the details for the last frame.

在主面板的正上方，有一个直方图，显示了所有已记录帧的渲染时间（参见图 @fig:Tracy_Frame_Time_View）。这使得更容易发现那些渲染时间超过平均水平的帧。在本例中，大多数帧的渲染时间约为33毫秒（黄色柱状图）。但是，有些帧的渲染时间更长，并以红色标记。如屏幕截图所示，当您将鼠标悬停在直方图中的某个柱状图上时，会显示一个工具提示，其中包含给定帧的详细信息。在本例中，我们显示的是最后一帧的详细信息。

![Tracy frame timings. You can find frames that take more time to render than other frames. Tracy帧计时。你会发现有些帧的渲染时间比其他帧更长。](../../img/perf-tools/tracy/tracy_frame_view.png){#fig:Tracy_Frame_Time_View width=90%}

Figure @fig:Tracy_CPU_Data illustrates the CPU data section of the profiler. This area shows which core a given thread is executing on and it also displays context switches. This section will also display other programs that are running on the CPU. As seen in the image, the details for a given thread are displayed when hovering the mouse on a given section in the CPU data view. Details include the CPU the thread is running on, the parent program, the individual thread, and timing information. We can see that the `TestCpu.exe` thread was active on CPU 1 only for 4.4 ms during the entire run of the program.

图 @fig:Tracy_CPU_Data 展示了坤坤性能分析器的CPU数据部分。该区域显示了给定线程正在哪个核心上执行，以及上下文切换情况。此外，该部分还会显示CPU上运行的其他程序。如图所示，将鼠标悬停在CPU数据视图中的特定区域时，会显示该线程的详细信息。详细信息包括线程运行所在的CPU、父程序、线程本身以及计时信息。我们可以看到，在整个程序运行过程中，`TestCpu.exe` 线程仅在CPU 1上活动了4.4毫秒。

![Tracy CPU data view. You can see what each CPU core was doing at any given moment. Tracy CPU数据视图。您可以查看每个CPU核心在任何给定时刻的运行情况。](../../img/perf-tools/tracy/tracy_cpu_view.png){#fig:Tracy_CPU_Data width=100%}

Next comes the panel that provides information on where our program spends its time (hotspots). Figure @fig:Tracy_Hotspots is a screenshot of Tracy's statistics window. We can check the recorded data, including the total time a given function was active, how many times it was invoked, etc. It's also possible to select a time range in the main view to filter information corresponding to a time interval.

接下来是面板，它提供有关程序运行时间分布（热点）的信息。图 @fig:Tracy_Hotspots 是Tracy统计窗口的屏幕截图。我们可以查看记录的数据，包括给定函数的总活动时间、调用次数等。也可以在主视图中选择时间范围，以筛选与该时间间隔对应的信息。

![Tracy function statistics. A regular "hotspot" view that provides information where a program spends time. Tracy函数统计信息。一个常规的“热点”视图，提供程序耗时位置的信息。](../../img/perf-tools/tracy/tracy_hotspots.png){#fig:Tracy_Hotspots width=100%}

The last set of panels that we show, enables us to analyze individual zone instances in more depth. Once you click on any zone instance, say, on the main timeline view or on the *CPU data* view, Tracy will open a *Zone Info* window (see the left panel in Figure @fig:Tracy_Zone_Details) with the details for this zone instance. It shows how much of the execution time is consumed by the zone itself or its children. In this example, execution of the `TraceRowJob` function took 19.24 ms, but the time consumed by the function itself without its callees (self time) takes 1.36 ms, which is only 7%. The rest of the time is consumed by child zones.

我们展示的最后一组面板使我们能够更深入地分析各个区域实例。单击任何区域实例后，例如在主时间线视图或“*CPU数据*”视图中，Tracy将打开一个“*区域信息（Zeon Info）*”窗口（参见图 @fig:Tracy_Zone_Details 中的左侧面板），其中包含该区域实例的详细信息。它显示了执行时间中有多少被区域本身或其子区域消耗。在本例中，`TraceRowJob` 函数的执行耗时19.24毫秒，但函数本身（不包括其调用者）的执行时间（自身时间）仅为1.36毫秒，占比仅为7%。其余时间均被子区域消耗。

It's easy to spot a call to `DoExtraWork` that takes the bulk of the time, 16.99 ms out of 19.24 ms (see the left panel in Figure @fig:Tracy_Zone_Details). Notice that this particular `TraceRowJob` instance runs almost 4.4 times as long as the average case (indicated by "437.93% of the mean time" on the image). Bingo! We found one of the slow instances where the `TraceRowJob` function was slowed down because of some extra work. One way to proceed would be to click on the `DoExtraWork` row to inspect this zone instance. This will update the Zone Info view with the details of the `DoExtraWork` instance so that we can dig down to understand what caused the performance issue. This view also shows the source file and line of code where the zone starts. So, another strategy would be to check the source code to understand why the current `TraceRowJob` instance takes more time than usual.

很容易发现，对 `DoExtraWork` 的调用消耗了大部分时间，在19.24毫秒中占了16.99毫秒（参见图 @fig:Tracy_Zone_Details 的左侧面板）。请注意，这个特定的 `TraceRowJob` 实例的运行时间几乎是平均情况的4.4倍（图中显示为“平均时间的437.93%”）。Bingo！我们找到了一个 `TraceRowJob` 函数因额外工作而运行缓慢的实例。一种方法是点击 `DoExtraWork` 行来检查此区域实例。这将更新“区域信息”视图，显示 `DoExtraWork` 实例的详细信息，以便我们深入分析导致性能问题的原因。此视图还会显示区域开始的源文件和代码行。因此，另一种策略是检查源代码，以了解当前 `TraceRowJob` 实例为何耗时比平时更长。
 
![Tracy zone detail windows. It shows statistics for a slow instance of the `TraceRowJob` zone. Tracy区域详细信息窗口。它显示了 `TraceRowJob` 区域慢实例的统计信息。](../../img/perf-tools/tracy/tracy_zone_details.png){#fig:Tracy_Zone_Details width=100%}

Remember, we saw in Figure @fig:Tracy_Frame_Time_View, that there are other slow frames. Let's see if this is the common problem among all the slow frames. If we click on the *Statistics* button, it will display the *Find Zone* panel (on the right of Figure @fig:Tracy_Zone_Details). Here we can see the time histogram that aggregates all zone instances. This is particularly useful to determine how much variation there is when executing a function. Looking at the histogram on the right, we see that the median duration for the `TraceRowJob` function is 3.59 ms, with most calls taking between 1 and 7 ms. However, there are a few instances that take longer than 10 ms, with a peak of 23 ms. Note that the time axis is logarithmic. The Find Zone window also provides other data points, including the mean, median, and standard deviation for the inspected zone.

请记住，我们在图 @fig:Tracy_Frame_Time_View 中看到，还有其他慢帧。让我们看看这是否是所有慢帧的共同问题。点击“统计”按钮，将显示“查找区域”面板（位于图 @fig:Tracy_Zone_Details 的右侧）。在这里，我们可以看到汇总所有区域实例的时间直方图。这对于确定函数执行过程中的时间波动情况尤为有用。观察右侧的直方图，我们可以看到 `TraceRowJob` 函数的执行时间中位数为3.59毫秒，大多数调用耗时在1到7毫秒之间。但是，也有一些实例耗时超过10毫秒，峰值达到23毫秒。请注意，时间轴为对数坐标。“查找区域”窗口还提供其他数据点，包括所检查区域的平均值、中位数和标准差。

Now we can examine other slow instances to find what is common between them, which will help us to determine the root cause of the issue. From this view, you can select one of the slow zones. This will update the *Zone Info* window with the details of that zone instance and by clicking the *Zoom to zone* button, the main window will focus on this slow zone. From here we can check if the selected `TraceRowJob` instance has similar characteristics as the one that we just analyzed.

现在，我们可以检查其他运行缓慢的实例，找出它们之间的共同点，这将有助于我们确定问题的根本原因。在此视图中，您可以选择一个运行缓慢的区域。这将更新“*区域信息(Zone Info)*”窗口，显示该区域实例的详细信息。点击“*缩放到区域(Zoom to zeon)*”按钮，主窗口将聚焦于该慢速区域。在这里，我们可以检查所选的 `TraceRowJob` 实例是否与我们刚刚分析的实例具有相似的特征。

### Other Features of Tracy Tracy的其他特性 {.unlisted .unnumbered}

Tracy monitors the performance of the whole system, not just the application itself. It also behaves like a traditional sampling profiler as it reports data for applications that are running concurrently with the profiled program. The tool monitors thread migration and idle time by tracing kernel context switches (administrator privileges are required). Zone statistics (call counts, time, histogram) are exact because Tracy captures every zone entry/exit, but system-level data and source-code-level data are sampled.

Tracy监控的是整个系统的性能，而不仅仅是应用程序本身。它还像传统的采样分析器一样，会报告与被分析程序同时运行的应用程序的数据。该工具通过跟踪内核上下文切换来监控线程迁移和空闲时间（需要管理员权限）。由于Tracy会捕获每个区域的入口/出口，因此区域统计信息（调用计数、时间、直方图）非常精确，但系统级数据和源代码级数据是采样的。

In the example, we used manual markup of interesting areas in the code. However, doing this is not a strict requirement to start using Tracy. You can profile an unmodified application and add instrumentation later when you know where it’s needed. Tracy provides many other features, too many to cover in this overview. Here are some of the notable ones:

在示例中，我们手动标记了代码中感兴趣的区域。但是，这样做并非使用Tracy的必要条件。您可以先分析一个未修改的应用程序，然后在需要时添加插桩。Tracy还提供了许多其他功能，无法在此概述中一一列举。以下是一些值得注意的功能：

* Tracking memory allocations and locks.
* Session comparison. This is vital to ensure a change provides the expected benefits. It's possible to load two profiling sessions and compare zone data before and after the change was made.
* Source code and assembly view. If debug symbols are available, Tracy can also display hotspots in the source code and related assembly just like Intel VTune and other profilers.

* 跟踪内存分配和锁。
* 会话比较。这对于确保更改能够带来预期效果至关重要。可以加载两个性能分析会话，并比较更改前后的区域数据。
* 源代码和汇编视图。如果调试符号可用，Tracy还可以像Intel VTune和其他性能分析器一样，显示源代码和相关汇编中的热点。

In comparison with other tools like Intel VTune and AMD uProf, with Tracy, you cannot get the same level of CPU microarchitectural insights (e.g., various performance events). This is because Tracy does not leverage the hardware features specific to a particular platform.

与其他工具（例如：Intel VTune和AMD uProf）相比，Tracy无法提供相同级别的CPU微体系结构洞察（例如：各种性能事件）。这是因为Tracy没有利用特定平台特有的硬件特性。

The overhead of profiling with Tracy depends on how many zones you have activated. The author of Tracy provides some data points that he measured on a program that does image compression: an overhead of 18% and 34% with two different compression schemes. A total of 200M zones were profiled, with an average overhead of 2.25 ns per zone. This test instrumented a very hot function. In other scenarios, the overhead will be much lower. While it's possible to keep the overhead small, you need to be careful about which sections of code you want to instrument, especially if you decide to use it in production.

使用Tracy进行性能分析的开销取决于激活的区域数量。Tracy的作者提供了一些他在执行图像压缩的程序上测得的数据点：两种不同的压缩方案的开销分别为18%和34%。总共轮廓分析了2亿个区域，每个区域的平均开销为2.25纳秒。此测试针对的是一个非常热门的函数。在其他情况下，开销会低得多。虽然可以尽可能降低开销，但您需要谨慎选择要检测的代码段，尤其是在生产环境中使用时。

[^1]: Tracy - [https://github.com/wolfpld/tracy](https://github.com/wolfpld/tracy)
[^2]: ToyPathTracer - [https://github.com/wolfpld/tracy/tree/master/examples/ToyPathTracer](https://github.com/wolfpld/tracy/tree/master/examples/ToyPathTracer)
