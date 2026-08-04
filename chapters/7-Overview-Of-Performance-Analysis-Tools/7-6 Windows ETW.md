## Event Tracing for Windows Windows事件跟踪 {#sec:ETW}

Microsoft has developed a system-wide tracing facility named Event Tracing for Windows (ETW). It was originally intended for helping device driver developers but later found use in analyzing general-purpose applications as well. ETW is available on all supported Windows platforms (x86 and ARM) with the corresponding platform-dependent installation packages. ETW records structured events in user and kernel code with full call stack trace support, which enables you to observe software dynamics in a running system and solve many challenging performance issues.

Microsoft 开发了一种名为“Windows 事件跟踪”(ETW: Event Tracing for Windows) 的系统范围跟踪工具。它最初旨在帮助设备驱动程序开发人员，但后来发现也可用于分析通用应用程序。ETW可在所有受支持的Windows平台（x86和ARM）上使用，并具有相应的依赖于平台的安装包。ETW通过完整的调用堆栈跟踪支持记录用户和内核代码中的结构化事件，使您能够观察正在运行的系统中的软件动态并解决许多具有挑战性的性能问题。

### How to configure it 如何配置 {.unlisted .unnumbered}

Recording ETW data is possible without any extra download since Windows 10 with `WPR.exe`. But to enable system-wide profiling you must be an administrator and have the `SeSystemProfilePrivilege` enabled. The \underline{W}indows \underline{P}erformance \underline{R}ecorder tool supports a set of built-in recording profiles that are suitable for common performance issues. You can tailor your recording needs by authoring a custom performance recorder profile xml file with the `.wprp` extension.

自Windows 10起，使用“WPR.exe”即可记录ETW数据，无需任何额外下载。但要启用系统范围的分析，您必须是管理员并启用 `SeSystemProfilePrivilege` 。 \underline{W}indows \underline{P}erformance \underline{R}ecorder 工具支持一组适用于常见性能问题的内置录制配置文件。您可以通过建立扩展名为“.wprp”的自定义性能记录轮廓xml文件来定制您的记录需求。

If you want to not only record but also view the recorded ETW data you need to install the Windows Performance Toolkit (WPT). You can download it from the Windows SDK[^1] or ADK[^2] download page. The Windows SDK is huge; you don't necessarily need all its parts. In our case, we just enabled the checkbox of the Windows Performance Toolkit. You are allowed to redistribute WPT as a part of your own application.

如果您不仅想记录还想查看记录的ETW数据，您需要安装Windows Performance Toolkit(WPT)。您可以从Windows SDK[^1]或ADK[^2]下载页面下载它。Windows SDK非常庞大；你不一定需要它的所有部分。在我们的例子中，我们只是启用了Windows Performance Toolkit的复选框。您可以将WPT作为您自己的应用程序的一部分进行重新分发。

### What you can do with it: 你可以用它做什么： {.unlisted .unnumbered}

- Identify hotspots with a configurable CPU sampling rate from 125 microseconds up to 10 seconds. The default is 1 millisecond which costs approximately 5--10% runtime overhead.
- Determine what blocks a certain thread and for how long (e.g., late event signals, unnecessary thread sleep, etc).
- Examine how fast a disk serves read/write requests and discover what initiates that work.
- Check file access performance and patterns (including cached read/writes that lead to no disk IO).
- Trace the TCP/IP stack to see how packets flow between network interfaces and computers.

- 通过可配置的CPU采样率（从125微秒到10秒）识别热点。默认值为1毫秒，这会导致大约5--10%的运行时开销。
- 确定是什么阻塞了某个线程以及阻塞了多长时间（例如，晚到的事件信号、不必要的线程睡眠等）。
- 检查磁盘处理读/写请求的速度并发现启动该工作的原因。
- 检查文件访问性能和模式（包括导致无磁盘IO的缓存读/写）。
- 跟踪TCP/IP堆栈以查看数据包如何在网络接口和计算机之间流动。

All the items listed above are recorded system-wide for all processes with configurable call stack traces (kernel and user mode call stacks are combined). It's also possible to add your own ETW provider to correlate the system-wide traces with your application behavior. You can extend the amount of data collected by instrumenting your code. For example, you can inject enter/leave ETW tracing hooks in functions into your source code to measure how often a certain function was executed.

上面列出的所有项目都在系统范围内记录所有具有可配置调用堆栈跟踪的进程（内核和用户模式调用堆栈相结合）。还可以添加您自己的ETW提供程序，将系统范围的跟踪与您的应用程序行为相关联。您可以通过检测代码来扩展收集的数据量。例如，您可以将函数中的Enter/leave ETW跟踪挂钩注入到源代码中，以测量某个函数的执行频率。

### What you cannot do with it: 你不能用它做什么： {.unlisted .unnumbered}

ETW traces are not useful for examining CPU microarchitectural bottlenecks. For that, use vendor-specific tools like Intel VTune, AMD uProf, Apple Instruments, etc.

ETW跟踪对于检查CPU微体系结构瓶颈没有用处。为此，请使用特定于供应商的工具，例如：Intel VTune、AMD uProf、Apple Instruments等。

ETW traces capture the dynamics of all processes at the system level, however, it may generate a lot of data. For example, capturing thread context switching data to observe various waits and delays can easily generate 1--2 GB per minute. That's why it is not practical to record high-volume events for hours without overwriting previously stored traces.

ETW跟踪捕获系统级别所有进程的动态，但是，它可能会生成大量数据。例如，捕获线程上下文切换数据以观察各种等待和延迟可以轻松生成每分钟1--2GB。这就是为什么在不覆盖以前存储的跟踪的情况下记录数小时的大量事件是不切实际的。

If you'd like to learn more about ETW, there is a more detailed discussion in Appendix D. We explore tools to record and analyze ETW and present a case study of debugging a slow start of a program.

如果您想了解有关ETW的更多信息，附录D中有更详细的讨论。我们探讨了记录和分析ETW的工具，并提供了调试程序缓慢启动的案例研究。

[^1]: Windows SDK Downloads - [https://developer.microsoft.com/en-us/windows/downloads/sdk-archive/](https://developer.microsoft.com/en-us/windows/downloads/sdk-archive/)
[^2]: Windows ADK Downloads - [https://learn.microsoft.com/en-us/windows-hardware/get-started/adk-install#other-adk-downloads](https://learn.microsoft.com/en-us/windows-hardware/get-started/adk-install#other-adk-downloads)
