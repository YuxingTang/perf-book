## Linux Perf

Linux Perf is probably the most used performance profiler in the world since it is available on most Linux distributions, which makes it accessible to a wide range of users. Perf is natively supported in many popular Linux distributions, including Ubuntu, Red Hat, and Debian. It is included in the kernel, so you can get OS-level statistics (page faults, CPU migrations, etc.) on any system that runs Linux. As of mid-2024, the profiler supports x86, ARM, PowerPC64, UltraSPARC, and a few other CPU types.[^2] On such platforms, `perf` provides access to the hardware performance monitoring features, for example, performance counters. More information about Linux `perf` is available on its [wiki page](https://perf.wiki.kernel.org/index.php/Main_Page)[^1].

Linux Per 可能是世界上使用最广泛的性能分析工具，因为它几乎在所有Linux发行版中都可用，因此用户群体非常广泛。许多流行的Linux发行版都原生支持Perf，包括：Ubuntu、Red Hat和Debian。它包含在内核中，因此您可以在任何运行Linux的系统上获取操作系统级别的统计信息（例如：页面错误、CPU迁移等）。截至2024年中期，该分析工具支持：x86、ARM、PowerPC64、UltraSPARC以及其他一些CPU类型[^2]。在这些平台上，`perf` 可以访问硬件性能监控功能，例如性能计数器。有关Linux `perf` 的更多信息，请访问其[wiki页面](https://perf.wiki.kernel.org/index.php/Main_Page)[^1]。

### How to configure it 如何配置 {.unlisted .unnumbered}

Installing Linux perf is very simple and can be done with a single command:

安装Linux perf非常简单，只需一条命令即可完成：

```bash
$ sudo apt-get install linux-tools-common linux-tools-generic linux-tools-`uname -r`
```

Also, consider changing the following defaults unless security is a concern:

此外，除非出于安全考虑，否则请考虑更改以下默认设置：

```bash
# Allow kernel profiling and access to CPU events for unprivileged users允许非特权用户进行内核分析并访问CPU事件
$ echo 0 | sudo tee /proc/sys/kernel/perf_event_paranoid
$ echo kernel.perf_event_paranoid=0 | sudo tee -a /etc/sysctl.d/local.conf
# Enable kernel modules symbols resolution for unprivileged users 为非特权用户启用内核模块符号解析
$ echo 0 | sudo tee /proc/sys/kernel/kptr_restrict
$ echo kernel.kptr_restrict=0 | sudo tee -a /etc/sysctl.d/local.conf
```

### What you can do with it: 功能介绍： {.unlisted .unnumbered}

Generally, Linux `perf` can do most of the same things that other profilers can do. Hardware vendors prioritize enabling their features in Linux `perf` so that by the time a new CPU is available on the market, `perf` already supports it. There are two main commands that most people use. The first, `perf stat`, reports a count of specified performance events. The second, `perf record`, profiles an application or system in sampling mode and is often followed by `perf report` to generate a report from the sampling data.

一般来说，Linux `perf` 可以完成其他性能分析工具的大部分功能。硬件厂商会优先在Linux `perf` 中启用其功能，以便在新CPU上市时，`perf` 已经支持它。大多数用户主要使用两个命令。第一个是 `perf stat`，用于报告指定性能事件的计数。第二个是 `perf record`，用于以采样模式分析应用程序或系统，通常后面会跟 `perf report` 来根据采样数据生成报告。

The output of the `perf record` command is a raw dump of samples. Many tools, built on top of Linux `perf`, parse raw dump files and provide new analysis types. Here are the most notable ones:

`perf record` 命令的输出是原始采样转储。许多基于Linux `perf` 构建的工具可以解析原始转储文件并提供新的分析类型。以下是一些最值得注意的类型：

- Flame graphs, discussed in [@sec:secFlameGraphs].
- [KDAB Hotspot](https://github.com/KDAB/hotspot),[^3] a tool that visualizes Linux `perf` data with an interface very similar to Intel VTune. If you have worked with Intel VTune, KDAB Hotspot will seem very familiar to you.
- Netflix [Flamescope](https://github.com/Netflix/flamescope).[^4] This tool displays a heat map of sampled events over application runtime. You can observe different phases and patterns in the behavior of a workload. Netflix engineers found some very subtle performance bugs using this tool. Also, you can select a time range on the heat map and generate a flame graph for that time range.

- 火焰图，详见 [@sec:secFlameGraphs]。
- [KDAB Hotspot](https://github.com/KDAB/hotspot)[^3] 是一款可视化Linux `perf` 数据的工具，其界面与Intel VTune非常相似。如果您使用过Intel VTune，那么KDAB Hotspot对您来说会非常熟悉。
- 网飞Netflix [Flamescope](https://github.com/Netflix/flamescope)[^4] 该工具以热图形式显示应用程序运行时采样事件。您可以观察工作负载行为的不同阶段和模式。Netflix的工程师使用此工具发现了一些非常细微的性能缺陷。此外，您还可以在热图上选择一个时间范围，并生成该时间范围的火焰图。

### What you cannot do with it: 它的功能限制： {.unlisted .unnumbered}

Linux perf is a command-line tool and lacks a Graphical User Interface (GUI), which makes it hard to filter data, observe how the workload behavior changes over time, zoom into a portion of the runtime, etc. There is a limited console output provided through the `perf report` command, which is fine for quick analysis, although not as convenient as other GUI profilers. Luckily, as we just mentioned, there are GUI tools that can post-process and visualize the raw output of Linux `perf`.

Linux perf是一个命令行工具，缺少图形用户界面(GUI)，这使得它难以进行数据筛选、观察工作负载行为随时间的变化、放大查看运行时的某个特定部分等等。虽然 `perf report` 命令提供了有限的控制台输出，足以进行快速分析，但不如其他GUI轮廓分析器方便。幸运的是，正如我们刚才提到的，有一些GUI工具可以对Linux `perf` 的原始输出进行后处理和可视化。

[^1]: Linux perf wiki - [https://perf.wiki.kernel.org/index.php/Main_Page](https://perf.wiki.kernel.org/index.php/Main_Page).
[^2]: RISCV is not supported yet as a part of the official kernel, although custom tools from vendors exist. RISCV目前尚未被官方内核支持，但有一些厂商提供了相应的自定义工具。
[^3]: KDAB Hotspot - [https://github.com/KDAB/hotspot](https://github.com/KDAB/hotspot).
[^4]: Netflix Flamescope - [https://github.com/Netflix/flamescope](https://github.com/Netflix/flamescope).
