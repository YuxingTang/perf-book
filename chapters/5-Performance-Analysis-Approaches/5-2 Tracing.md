## Tracing 跟踪

Tracing is conceptually very similar to instrumentation, yet slightly different. Code instrumentation assumes that the user has full access to the source code of their application. On the other hand, tracing relies on the existing instrumentation. For example, the `strace` tool enables us to trace system calls and can be thought of as instrumentation of the Linux kernel. Intel Processor Traces (Intel PT, see Appendix C) enable you to log instructions executed by a processor and can be thought of as instrumentation of a CPU. Traces can be obtained from components that were appropriately instrumented in advance and are not subject to change. Tracing is often used as a black-box approach, where a user cannot modify the code of an application, yet they want to get insights into what the program is doing.

跟踪在概念上与代码插桩非常相似，但又略有不同。代码插桩假定用户拥有对其应用程序源代码的完全访问权限。而跟踪则依赖于现有的插桩。例如，`strace` 工具使我们能够跟踪系统调用，可以将其视为对 Linux 内核的插桩。Intel 处理器跟踪（Intel PT，参见附录 C）允许您记录处理器执行的指令，可以将其视为对 CPU 的插桩。跟踪信息可以从预先进行适当插桩且不会发生更改的组件中获取。跟踪通常用作黑盒方法，用户无法修改应用程序的代码，但希望了解程序正在执行的操作。

An example of tracing system calls with the Linux `strace` tool is provided in [@lst:strace], which shows the first several lines of output when running the `git status` command. By tracing system calls with `strace` it's possible to know the timestamp for each system call (the leftmost column), its exit status (after the `=` sign), and the duration of each system call (in angle brackets).

[@lst:strace] 中提供了一个使用 Linux `strace` 工具跟踪系统调用的示例，其中显示了运行 `git status` 命令时输出的前几行。通过使用 `strace` 跟踪系统调用，可以了解每个系统调用的时间戳（最左侧一列）、退出状态（等号 `=` 后）以及持续时间（尖括号内）。

Listing: Tracing system calls with strace.
代码列表：使用strace跟踪系统调用。

~~~~ {#lst:strace .bash}
$ strace -tt -T -- git status
17:46:16.798861 execve("/usr/bin/git", ["git", "status"], 0x7ffe705dcd78
                  /* 75 vars */) = 0 <0.000300>
17:46:16.799493 brk(NULL)               = 0x55f81d929000 <0.000062>
17:46:16.799692 access("/etc/ld.so.nohwcap", F_OK) = -1 ENOENT
                  (No such file or directory) <0.000063>
17:46:16.799863 access("/etc/ld.so.preload", R_OK) = -1 ENOENT
                  (No such file or directory) <0.000074>
17:46:16.800032 openat(AT_FDCWD, "/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
                  <0.000072>
17:46:16.800255 fstat(3, {st_mode=S_IFREG|0644, st_size=144852, ...}) = 0
                  <0.000058>
17:46:16.800408 mmap(NULL, 144852, PROT_READ, MAP_PRIVATE, 3, 0)
                  = 0x7f6ea7e48000 <0.000066>
17:46:16.800619 close(3)                = 0 <0.000123>
...
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The overhead of tracing depends on what exactly we try to trace. For example, if we trace a program that rarely makes system calls, the overhead of running it under `strace` will be close to zero. On the other hand, if we trace a program that heavily relies on system calls, the overhead could be very large, e.g. 100x.[^1] Also, tracing can generate a massive amount of data since it doesn't skip any sample. To compensate for this, tracing tools provide filters that enable you to restrict data collection to a specific time slice or for a specific section of code.

跟踪的开销取决于我们具体要跟踪的内容。例如，如果我们跟踪一个很少进行系统调用的程序，那么使用 `strace` 运行它的开销几乎为零。另一方面，如果我们跟踪一个严重依赖系统调用的程序，开销可能会非常大，例如 100 倍。[^1] 此外，由于跟踪不会跳过任何样本，因此会生成大量数据。为了弥补这一点，跟踪工具提供了过滤器，使您可以将数据收集限制在特定的时间段或特定的代码段。

Similar to instrumentation, tracing can be used for exploring anomalies in a system. For example, you may want to determine what was going on in an application during a 10s period of unresponsiveness. As you will see later, sampling methods are not designed for this, but with tracing, you can see what leads to the program being unresponsive. For example, with Intel PT, you can reconstruct the control flow of the program and know exactly what instructions were executed.

与插桩类似，跟踪可以用于探索系统中的异常情况。例如，您可能想要确定应用程序在 10 秒无响应期间发生了什么。正如您稍后将看到的，采样方法并非为此而设计，但通过跟踪，您可以了解导致程序无响应的原因。例如，使用 Intel PT，您可以重建程序的控制流，并准确了解执行了哪些指令。

Tracing is also very useful for debugging. Its underlying nature enables "record and replay" use cases based on recorded traces. One such tool is the Mozilla [rr](https://rr-project.org/)[^2] debugger, which performs record and replay of processes, supports backward single stepping, and much more. Most tracing tools are capable of decorating events with timestamps, which enables us to find correlations with external events that were happening during that time. That is, when we observe a glitch in a program, we can take a look at the traces of our application and correlate this glitch with what was happening in the whole system during that time.

跟踪对于调试也非常有用。其底层特性支持基于记录跟踪的“记录和回放”用例。Mozilla [rr](https://rr-project.org/)[^2] 调试器就是这样一种工具，它可以执行进程的记录和回放，支持单步回退等诸多功能。大多数跟踪工具都能够为事件添加时间戳，这使我们能够找到与当时发生的外部事件的关联。也就是说，当我们观察到程序出现故障时，我们可以查看应用程序的跟踪记录，并将此故障与当时整个系统中发生的情况关联起来。

[^1]: An article about `strace` by B. Gregg 由B. Gregg撰写的关于“strace”的文章 - [http://www.brendangregg.com/blog/2014-05-11/strace-wow-much-syscall.html](http://www.brendangregg.com/blog/2014-05-11/strace-wow-much-syscall.html)

[^2]: Mozilla rr debugger Mozilla的rr调试器 - [https://rr-project.org/](https://rr-project.org/).
