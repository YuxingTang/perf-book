\phantomsection
# Appendix A. Reducing Measurement Noise 附录A. 降低测量噪声 {.unnumbered}

\markboth{Appendix A}{Appendix A}

Below are some examples of features that can contribute to increased non-determinism in performance measurements and a few techniques to reduce noise. I provided an introduction to the topic in [@sec:secFairExperiments].

以下列举了一些可能导致性能测量不确定性增加的特性示例，以及一些降低噪声的技术。我在 [@sec:secFairExperiments] 中对该主题进行了介绍。

This section is mostly specific to the Linux operating system. Readers are encouraged to search the web for instructions on how to configure other operating systems.

本节主要针对 Linux 操作系统。建议读者在网上搜索有关如何配置其他操作系统的说明。

## Dynamic Frequency Scaling 动态频率调节 {.unnumbered .unlisted}

[Dynamic Frequency Scaling](https://en.wikipedia.org/wiki/Dynamic_frequency_scaling)[^11] (DFS) is a technique to increase the performance of a system by automatically raising CPU operating frequency when it runs demanding tasks. As an example of DFS implementation, Intel CPUs have a feature called Turbo Boost, and AMD CPUs employ Turbo Core functionality. 

动态频率调节[Dynamic Frequency Scaling](https://en.wikipedia.org/wiki/Dynamic_frequency_scaling)[^11]是一种通过在系统运行高负载任务时自动提高CPU运行频率来提升系统性能的技术。例如，Intel CPU具有名为睿频（Turbo Boost）的功能，而AMD CPU则使用动态超频（Turbo Core）来描述这一功能，这些都是DFS实现的典型例子。

Here is an example of the impact Turbo Boost can make for a single-threaded workload running on Intel® Core™ i5-8259U:

以下示例展示了睿频加速技术对运行在Intel® Core™ i5-8259U上的单线程工作负载的影响：

```bash
# TurboBoost enabled
$ cat /sys/devices/system/cpu/intel_pstate/no_turbo
0
$ perf stat -e task-clock,cycles -- ./a.exe 
    11984.691958  task-clock (msec) #    1.000 CPUs utilized
  32,427,294,227  cycles            #    2.706 GHz
      11.989164338 seconds time elapsed

# TurboBoost disabled
$ echo 1 | sudo tee /sys/devices/system/cpu/intel_pstate/no_turbo
1
$ perf stat -e task-clock,cycles -- ./a.exe 
    13055.200832  task-clock (msec) #    0.993 CPUs utilized
  29,946,969,255  cycles            #    2.294 GHz
      13.142983989 seconds time elapsed
```

The average frequency is higher when Turbo Boost is on (2.7 GHz vs. 2.3 GHz).
开启睿频加速后，平均频率更高（2.7GHz 对比 2.3GHz）。

DFS can be permanently disabled in BIOS. To programmatically disable the DFS feature on Linux systems, you need root access. Here is how one can achieve this:

DFS可以在BIOS中永久禁用。要在Linu 系统上以编程方式禁用DFS功能，您需要root权限。以下是具体操作方法：

```bash
# Intel
echo 1 > /sys/devices/system/cpu/intel_pstate/no_turbo
# AMD
echo 0 > /sys/devices/system/cpu/cpufreq/boost
```
## Simultaneous Multithreading 同时多线程 {.unnumbered .unlisted}

Many modern CPU cores support simultaneous multithreading (see [@sec:SMT]). SMT can be permanently disabled in BIOS. To programmatically disable SMT on Linux systems, you need root access. The sibling pairs of CPU threads can be found in the following files:

许多现代CPU核心都支持同时多线程（参见 [@sec:SMT]）。SMT可以在BIOS中永久禁用。要在Linux系统上以编程方式禁用 SMT，您需要root权限。CPU线程的同级线程对可以在以下文件中找到：

```bash
/sys/devices/system/cpu/cpuN/topology/thread_siblings_list
```

Here is how you can disable a sibling thread of core 0 on Intel® Core™ i5-8259U, which has 4 cores and 8 threads:

以下是如何在拥有4个核心和8个线程的Intel® Core™ i5-8259U处理器上禁用核心0的同级线程：

```bash
# all 8 hardware threads enabled:
$ lscpu
...
CPU(s):              8
On-line CPU(s) list: 0-7
...
$ cat /sys/devices/system/cpu/cpu0/topology/thread_siblings_list
0,4
$ cat /sys/devices/system/cpu/cpu1/topology/thread_siblings_list
1,5
$ cat /sys/devices/system/cpu/cpu2/topology/thread_siblings_list
2,6
$ cat /sys/devices/system/cpu/cpu3/topology/thread_siblings_list
3,7

# Disabling SMT on core 0
$ echo 0 | sudo tee /sys/devices/system/cpu/cpu4/online
0
$ lscpu
CPU(s):               8
On-line CPU(s) list:  0-3,5-7
Off-line CPU(s) list: 4
...
$ cat /sys/devices/system/cpu/cpu0/topology/thread_siblings_list
0
```

Also, the `lscpu --all --extended` command can be very helpful to see the sibling threads.

此外，`lscpu --all --extended` 命令对于查看同级线程非常有用

## Scaling Governor 频率调节器 {.unnumbered .unlisted}

Linux kernel can control CPU frequency for different purposes. One such purpose is to save power. In this case, the scaling governor may decide to decrease the CPU frequency. For performance measurements, it is recommended to set the scaling governor policy to `performance` to avoid sub-nominal clocking. Here is how we can set it for all the cores:

Linux内核可以出于各种目的控制CPU频率。其中一个目的是节省功耗。在这种情况下，频率调节器可能会决定降低CPU频率。对于性能测试，建议将频率调节器策略设置为 `性能模式performance`，以避免低于标称频率。以下是如何为所有核心设置此策略：

```bash
echo performance | sudo tee /sys/devices/system/cpu/cpufreq/policy*/scaling_governor
```

## CPU Affinity CPU亲和性 {.unnumbered .unlisted}

[Processor affinity](https://en.wikipedia.org/wiki/Processor_affinity)[^8] enables the binding of a process to a certain CPU core(s). In Linux, you can do this with [`taskset`](https://linux.die.net/man/1/taskset)[^9] tool.

[处理器亲和性](https://en.wikipedia.org/wiki/Processor_affinity)[^8] 允许将进程绑定到特定的CPU核心。在 Linux系统中，您可以使用 [`taskset`](https://linux.die.net/man/1/taskset)[^9]。

```bash
# no affinity
$ perf stat -e context-switches,cpu-migrations -r 10 -- a.exe
               151      context-switches
                10      cpu-migrations

# process is bound to the CPU0
$ perf stat -e context-switches,cpu-migrations -r 10 -- taskset -c 0 a.exe 
               102      context-switches
                 0      cpu-migrations
```

Notice the number of `cpu-migrations` gets down to `0`, i.e., the process never leaves `core0`.

注意 `cpu-migrations` 的数量会降至 `0`，也就是说，进程始终占用 `core0` 核心。

Alternatively, you can use [cset](https://github.com/lpechacek/cpuset)[^10] tool to reserve CPUs for just the program you are benchmarking. If using `Linux perf`, leave at least two cores so that `perf` runs on one core, and your program runs in another. The command below will move all threads out of N1 and N2 (`-k on` means that even kernel threads are moved out):

或者，您可以使用 [cset](https://github.com/lpechacek/cpuset)[^10] 工具为正在进行基准测试的程序预留CPU。如果使用 `Linux perf`，请至少保留两个核心，以便 `perf` 在一个核心上运行，而您的程序在另一个核心上运行。以下命令会将所有线程移出N1和N2核心（`-k on` 表示甚至内核线程也会被移出）：

```bash
$ cset shield -c N1,N2 -k on
```

The command below will run the command after `--` in the isolated CPUs: 
以下命令会在隔离的 CPU 上运行 `--` 后面的命令：
```bash
$ cset shield --exec -- perf stat -r 10 <cmd>
```

On Windows, a program can be pinned to a specific core using the following command:
在Windows系统中，可以使用以下命令将程序绑定到特定核心：

```bash
$ start /wait /b /affinity 0xC0 myapp.exe
```

where the `/wait` option waits for the application to terminate, `/b` starts the application without opening a new command window, and `/affinity` specifies the CPU affinity mask. In this case, the mask `0xC0` means that the application will run on cores 6 and 7.

其中，`/wait` 选项等待应用程序终止，`/b` 启动应用程序而不打开新的命令窗口，`/affinity` 指定 CPU 亲和性掩码。在本例中，掩码 `0xC0` 表示应用程序将在 6 号和 7 号核心上运行。

On macOS, it is not possible to pin threads to cores since the operating system does not provide an API for that.

在 macOS 上，由于操作系统没有提供相应的 API，因此无法将线程绑定到核心。

## Process Priority 进程优先级 {.unnumbered .unlisted}

In Linux, you can increase process priority using the `nice` tool. By increasing priority, the process gets more CPU time, and the scheduler favors it more in comparison with processes with normal priority. Niceness ranges from `-20` (highest priority value) to `19` (lowest priority value) with the default of `0`.

在 Linux 中，可以使用 `nice` 工具提高进程优先级。提高优先级后，进程将获得更多 CPU 时间，调度器也会优先处理优先级更高的进程。优先级范围从 `-20`（最高优先级）到 `19`（最低优先级），默认值为 `0`。

Notice in the previous example, that the execution of the benchmarked process was interrupted by the OS more than 100 times. If we increase process priority by running the benchmark with `sudo nice -n -<N>`:

请注意，在前面的示例中，被测进程的执行被操作系统中断了 100 多次。如果我们通过使用 `sudo nice -n -<N>` 运行基准测试来提高进程优先级：

```bash
$ perf stat -r 10 -- sudo nice -n -5 taskset -c 1 a.exe
    0   context-switches
    0   cpu-migrations
```

Notice the number of context switches gets to `0`, so the process received all the computation time uninterrupted.

请注意，上下文切换次数变为 `0`，这意味着该进程获得了所有计算时间，且未受到任何干扰。

[^4]: SMT - [https://en.wikipedia.org/wiki/Simultaneous_multithreading](https://en.wikipedia.org/wiki/Simultaneous_multithreading).
[^7]: Documentation for Linux CPU frequency governors: Linux的CPU频率调节器文档：[https://www.kernel.org/doc/Documentation/cpu-freq/governors.txt](https://www.kernel.org/doc/Documentation/cpu-freq/governors.txt).
[^8]: Processor affinity 处理器亲和性 - [https://en.wikipedia.org/wiki/Processor_affinity](https://en.wikipedia.org/wiki/Processor_affinity).
[^9]: `taskset` manual `taskset` 手册 - [https://linux.die.net/man/1/taskset](https://linux.die.net/man/1/taskset).
[^10]: `cpuset` manual `cpuset` 手册 - [https://github.com/lpechacek/cpuset](https://github.com/lpechacek/cpuset).
[^11]: Dynamic frequency scaling 动态频率缩放 - [https://en.wikipedia.org/wiki/Dynamic_frequency_scaling](https://en.wikipedia.org/wiki/Dynamic_frequency_scaling).
