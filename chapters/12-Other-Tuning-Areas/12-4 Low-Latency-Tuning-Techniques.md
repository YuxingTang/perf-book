## Low Latency Tuning Techniques 低延迟调优技术 {#sec:LowLatency}

So far we have discussed a variety of software optimizations that aim at improving the overall performance of an application. In this section, we will discuss additional tuning techniques used in low-latency systems, such as real-time processing and high-frequency trading (HFT). In such an environment, the primary optimization goal is to make a certain portion of a program run as fast as possible. When you work in the HFT industry, every microsecond and nanosecond counts as it has a direct impact on profits. Usually, the low-latency portion implements a critical loop of a real-time or an HFT system, such as moving a robotic arm or sending an order to the exchange. Optimizing the latency of a critical path is sometimes done at the expense of other portions of a program. And some techniques even sacrifice the overall throughput of a system.

到目前为止，我们已经讨论了各种旨在提升应用程序整体性能的软件优化方法。在本节中，我们将讨论低延迟系统（例如实时处理和高频交易(HFT: High-Frequency Trading)中使用的其他调优技术。在这种环境下，主要的优化目标是使程序的特定部分尽可能快速地运行。在高频交易行业，每一微秒和每一纳秒都至关重要，因为它们直接影响利润。通常，低延迟部分实现的是实时系统或高频交易系统的关键循环，例如控制机械臂或向交易所发送订单。优化关键路径的延迟有时会以牺牲程序其他部分的性能为代价。有些技术甚至会牺牲系统的整体吞吐率。

When developers optimize for latency, they avoid any unnecessary costs they need to pay on a hot path. That usually involves system calls, memory allocation, I/O, and anything else that has non-deterministic latency. To reach the lowest possible latency, the hot path needs to have all the resources ready and available immediately. 

开发人员优化延迟时，会避免在关键路径上产生任何不必要的成本。这通常涉及系统调用、内存分配、I/O以及任何其他具有不确定延迟的操作。为了达到尽可能低的延迟，热点路径需要所有资源都已准备就绪并可立即使用。

One relatively simple technique is to precompute some of the operations you would do on the hot path. That comes with a cost of using more memory which will be unavailable to other processes in the system but it may save you some precious cycles on a critical path. However, keep in mind that sometimes it is faster to compute the thing than to fetch the result from memory.

一种相对简单的技巧是预先计算热路径上的一些操作。这样做会占用更多内存，导致系统中其他进程无法使用这些内存，但它可以节省关键路径上的一些宝贵周期。然而，请记住，有时计算结果比从内存中获取结果更快。

Since this is a book about low-level CPU performance, we will skip talking about higher-level techniques similar to the one we just mentioned. Instead, we will discuss how to avoid page faults, cache misses, TLB shootdowns, and core throttling on a critical path.

由于本书侧重于底层CPU性能优化，我们将略过讨论与刚才提到的技巧类似的高级技术。相反，我们将讨论如何避免关键路径上的页面错误、缓存未命中、TLB崩溃和核心节流。

### Avoid Minor Page Faults 避免次要页失效错误 {#sec:AvoidPageFaults}

While the term contains the word "minor", there's nothing minor about the impact of minor page faults on runtime latency. Recall that when a user code allocates memory, OS only commits to provide a page, but it doesn't immediately execute on the commitment by giving us a zeroed physical page. Instead, it will wait until the first time the user code will access it, and only then the operating system fulfills its duties. The very first write to a newly allocated page triggers a minor page fault, a hardware interrupt that is handled by the OS. The latency impact of minor faults can range from just under a microsecond up to several microseconds, especially if you're using a Linux kernel with 5-level page tables instead of 4-level page tables.

虽然名字中包含“微小/次要minor”二字，但次要页面错误对运行时延迟的影响却不容小觑。回想一下，当用户代码分配内存时，操作系统只会承诺分配一个页面，但并不会立即执行分配操作，也不会立即将物理页面置零。相反，它会等到用户代码首次访问该页面时，操作系统才会履行其职责。首次写入新分配的页面会触发一个微小页失效/缺页错误（Minor Page Fault），这是一个由操作系统处理的硬件中断。次要缺页错误的延迟影响范围从不到一微秒到几微秒不等，尤其是在使用具有五级页表而非四级页表的Linux内核时。

How do you detect runtime minor page faults in your application? One simple way is by using the `top` utility (add the `-H` option for a thread-level view). Add the `vMn` field to the default selection of display columns to view the number of minor page faults occurring per display refresh interval. [@lst:DumpTopWithMinorFaults] shows a dump of the `top` command with the top-10 processes while compiling a large C++ project. The additional `vMn` column shows the number of minor page faults that occurred during the last 3 seconds.

如何在应用程序中检测运行时次要缺页错误？一个简单的方法是使用 `top` 工具（添加 `-H` 选项可查看线程级别）。将 `vMn` 字段添加到默认显示列中，即可查看每个显示刷新间隔内发生的次要缺页错误的数量。 [@lst:DumpTopWithMinorFaults] 显示了编译大型C++项目时 `top` 命令的输出结果，其中包含前10个进程。新增的 `vMn` 列显示了过去3秒内发生的次要页面错误 (minor page fault) 的数量。

Listing: A dump of Linux top command with additional vMn field while compiling a large C++ project.
代码列表：在编译大型C++项目是包含额外vMn字段的Linux tom命令输出结果

~~~~ {#lst:DumpTopWithMinorFaults .cpp}
   PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND  vMn
341763 dendiba+  20   0  303332 165396  83200 R  99.3   1.0   0:05.09 c++      13k
341705 dendiba+  20   0  285768 153872  87808 R  99.0   1.0   0:07.18 c++       5k
341719 dendiba+  20   0  313476 176236  83328 R  94.7   1.1   0:06.49 c++       8k
341709 dendiba+  20   0  301088 162800  82944 R  93.4   1.0   0:06.46 c++       2k
341779 dendiba+  20   0  286468 152376  87424 R  92.4   1.0   0:03.08 c++      26k
341769 dendiba+  20   0  293260 155068  83072 R  91.7   1.0   0:03.90 c++      22k
341749 dendiba+  20   0  360664 214328  75904 R  88.1   1.3   0:05.14 c++      18k
341765 dendiba+  20   0  351036 205268  76288 R  87.1   1.3   0:04.75 c++      18k
341771 dendiba+  20   0  341148 194668  75776 R  86.4   1.2   0:03.43 c++      20k
341776 dendiba+  20   0  286496 147460  82432 R  76.2   0.9   0:02.64 c++      25k
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Another way of detecting runtime minor page faults involves attaching to the running process with `perf stat -e page-faults`. 

另一种检测运行时次要缺页错误的方法是使用 `perf stat -e page-faults` 命令附加到正在运行的进程。

In the HFT world, anything more than `0` is a problem. But for low latency applications in other business domains, a constant occurrence in the range of 100-1000 faults per second should prompt further investigation. Investigating the root cause of runtime minor page faults can be as simple as firing up `perf record -e page-faults` and then `perf report` to locate offending source code lines.

在高频交易领域，任何大于 `0` 的值都算作问题。但对于其他业务领域的低延迟应用程序，如果每秒出现100到1000次缺页错误，则应进行进一步调查。调查运行时次要缺页错误的根本原因可以很简单，只需运行 `perf record -e page-faults` 命令，然后运行 ​​`perf report` 命令来定位导致问题的源代码行即可。

To avoid page fault penalties during runtime, you should pre-fault all the memory for the application at startup time. A toy example might look something like this:

为了避免运行时缺页错误带来的性能损失，您应该在启动时预先制造应用程序的所有内存页失效。一个简单的示例可能如下所示：

```cpp
char *mem = malloc(size);
int pageSize = sysconf(_SC_PAGESIZE)
for (int i = 0; i < size; i += pageSize)
  mem[i] = 0;
```

First, this sample code allocates a `size` amount of memory on the heap as usual. However, immediately after that, it steps by and writes to the first byte of each page of newly allocated memory to ensure each one is brought into RAM. This method helps to avoid runtime delays caused by minor page faults during future accesses.

首先，这段示例代码像往常一样在堆上分配 `size` 大小的内存。然而，紧接着，它会逐个写入每个新分配内存页的第一个字节，以确保每个页面都被加载到RAM中。这种方法有助于避免在后续访问过程中因次要缺页错误而导致的运行时延迟。

Take a look at [@lst:LockPagesAndNoRelease] with a more comprehensive approach to tuning the glibc allocator in conjunction with `mlock/mlockall` syscalls (taken from the "Real-time Linux Wiki" [^1]).

请参阅 [@lst:LockPagesAndNoRelease]，它提供了一种更全面的方法，结合 `mlock/mlockall` 系统调用来调整glibc内存分配器（摘自“实时 Linux Wiki”[^1]）。

Listing: Tuning the glibc allocator to lock pages in RAM and prevent releasing them to the OS.
代码列表：调整glibc内存分配器，将页面锁定在RAM中，并阻止将其释放给操作系统。

~~~~ {#lst:LockPagesAndNoRelease .cpp}
#include <malloc.h>
#include <sys/mman.h>

mallopt(M_MMAP_MAX, 0);
mallopt(M_TRIM_THRESHOLD, -1);
mallopt(M_ARENA_MAX, 1);

mlockall(MCL_CURRENT | MCL_FUTURE);

char *mem = malloc(size);
for (int i = 0; i < size; i += sysconf(_SC_PAGESIZE))
    mem[i] = 0;
//...
free(mem);
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The code in [@lst:LockPagesAndNoRelease] tunes three glibc malloc settings: `M_MMAP_MAX`, `M_TRIM_THRESHOLD`, and `M_ARENA_MAX`.

[@lst:LockPagesAndNoRelease] 中的代码调整了glibc malloc的三个设置：`M_MMAP_MAX`、`M_TRIM_THRESHOLD` 和 `M_ARENA_MAX`。

- Setting `M_MMAP_MAX` to `0` disables underlying `mmap` syscall usage for large allocations – this is necessary because the `mlockall` can be undone by library usage of `munmap` when it attempts to release `mmap`-ed segments back to the OS, defeating the purpose of our efforts.
- Setting `M_TRIM_THRESHOLD` to `-1` prevents glibc from returning memory to the OS after calls to `free`. As indicated before, this option has no effect on `mmap`-ed segments.
- Finally, setting `M_ARENA_MAX` to `1` prevents glibc from allocating multiple arenas via `mmap` to accommodate multiple cores. Keep in mind, that the latter hinders the glibc allocator's multithreaded scalability feature.

- 将 `M_MMAP_MAX` 设置为 `0` 会禁用底层 `mmap` 系统调用对大内存分配的使用——这是必要的，因为当库尝试将通过 `mmap` 分配的内存段释放回操作系统时，`mlockall` 操作可能会被库调用 `munmap` 撤销，从而使我们的努力付诸东流。
- 将 `M_TRIM_THRESHOLD` 设置为 `-1` 会阻止glibc在调用 `free` 后将内存释放回操作系统。如前所述，此选项对通过 `mmap` 分配的内存段无效。
- 最后，将 `M_ARENA_MAX` 设置为 `1` 会阻止 glibc 通过 `mmap` 分配多个arena来适应多核处理器。请注意，后者会阻碍glibc分配器的多线程可扩展性。

Combined, these settings force glibc into heap allocations which will not release memory back to the OS until the application ends. As a result, the heap will remain the same size after the final call to `free(mem)` in the code above. Any subsequent runtime calls to `malloc` or `new` simply will reuse space in this pre-allocated/pre-faulted heap area if it is sufficiently sized at initialization.

这些设置共同作用，强制glibc进行堆内存分配，直到应用程序结束才会将内存释放回操作系统。因此，在上述代码中最后一次调用 `free(mem)` 后，堆的大小将保持不变。任何后续的运行时 `malloc` 或 `new` 调用，如果初始化时预分配/预锁定的堆区域大小足够，则只会重用其中的空间。

More importantly, all that heap memory that was pre-faulted in the `for`-loop will persist in RAM due to the previous `mlockall` call – the option `MCL_CURRENT` locks all pages that are currently mapped, while `MCL_FUTURE` locks all pages that will become mapped in the future. An added benefit of using `mlockall` this way is that any thread spawned by this process will have its stack pre-faulted and locked, as well. For the finer control of page locking, developers should use `mlock` system call which gives you the option to choose which pages should persist in RAM. A downside of this technique is that it reduces the amount of memory available to other processes running on the system.

更重要的是，由于之前的 `mlockall` 调用，所有在 `for` 循环中预锁定的堆内存都将保留在RAM中——选项 `MCL_CURRENT` 会锁定所有当前已映射的页面，而 `MCL_FUTURE` 会锁定所有将来会映射的页面。以这种方式使用 `mlockall` 的另一个好处是，此进程创建的任何线程的栈也会被预锁定。为了更精细地控制页面锁定，开发者应该使用 `mlock` 系统调用，它允许你选择哪些页面应该持久保存在RAM中。这种方法的缺点是会减少系统上其他进程可用的内存量。

Developers of applications for Windows should look into the following APIs: lock pages with `VirtualLock`, avoid immediate release of memory with `VirtualFree` with `MEM_DECOMMIT`, but not the `MEM_RELEASE` flag.

Windows应用程序的开发者应该考虑以下API：使用 `VirtualLock` 锁定页面，使用 `VirtualFree` 并配合 `MEM_DECOMMIT` 标志（但不要使用 `MEM_RELEASE` 标志）来避免立即释放内存。

These are just two example methods for preventing runtime minor faults. Some or all of these techniques may be already integrated into memory allocation libraries such as jemalloc, tcmalloc, or mimalloc. Check the documentation of your library to see what is available.

以上只是防止运行时次要页故障的两种示例方法。部分或全部这些技术可能已经集成到内存分配库中，例如：jemalloc、tcmalloc或mimalloc。请查阅你所用库的文档以了解有哪些可用功能。

### Cache Warming 缓存预热 {#sec:CacheWarm}

In some applications, the portions of code that are most latency-sensitive are the least frequently executed. An example of such an application might be an HFT application that continuously reads market data signals from the stock exchange and, once a favorable market signal is detected, sends an order to the exchange. In the aforementioned workload, the code paths involved with reading the market data are most commonly executed, while the code paths for executing an order are rarely executed.

在某些应用中，对延迟最敏感的代码部分执行频率最低。例如，高频交易（HFT）应用会持续读取交易所的市场数据信号，一旦检测到有利的市场信号，便会向交易所发送订单。在上述工作负载中，读取市场数据的代码路径执行频率最高，而执行订单的代码路径则很少执行。

Since other players in the market are likely to catch the same market signal, the success of the strategy largely relies on how fast we can react, in other words, how fast we send the order to the exchange. When we want our order to reach the exchange as fast as possible and to take advantage of the favorable signal detected in the market data, the last thing we want is to meet roadblocks right at the moment we decide to take off. 

由于市场上的其他参与者也可能捕捉到相同的市场信号，因此该策略的成功很大程度上取决于我们的反应速度，也就是向交易所发送订单的速度。当我们希望订单尽快到达交易所并利用市场数据中检测到的有利信号时，最不希望看到的就是在准备出发的瞬间遇到阻碍。

When a certain code path is not exercised for a while, its instructions and associated data are likely to be evicted from the I-cache and D-cache. Then, just when we need that critical piece of rarely executed code to run, we take I-cache and D-cache miss penalties, which may cause us to lose the race. This is where the technique of *cache warming* is helpful.

当某个代码路径长时间未被执行时，其指令和相关数据很可能会被从指令缓存（I-cache）和数据缓存（D-cache）中移除。然后，就在我们需要运行那段很少执行的关键代码时，却要承受指令缓存 (I-cache) 和数据缓存 (D-cache) 未命中的惩罚，这可能会导致我们在竞争中失败。这时，*缓存预热（Cache Warming）* 技术就派上了用场。

Cache warming involves periodically exercising the latency-sensitive code to keep it in the cache while ensuring it does not follow all the way through with any unwanted actions. Exercising the latency-sensitive code also "warms up" the D-cache by bringing latency-sensitive data into it. This technique is routinely employed for HFT applications. While I will not provide an example implementation, you can get a taste of it in a [CppCon 2018 lightning talk](https://www.youtube.com/watch?v=XzRxikGgaHI)[^4].

缓存预热是指定期执行对延迟敏感的代码，使其保持在缓存中，同时确保它不会执行任何不必要的操作。执行对延迟敏感的代码还可以通过将对延迟敏感的数据放入数据缓存来“预热（warms up）”数据缓存。这项技术通常用于高频交易 (HFT) 应用。虽然我不会提供示例实现，但您可以在[CppCon 2018 闪电演讲](https://www.youtube.com/watch?v=XzRxikGgaHI)[^4]中了解一下。

### Avoid TLB Shootdowns 避免TLB崩溃

We learned from earlier chapters that the TLB is a fast but finite per-core cache for virtual-to-physical memory address translations that reduces the need for time-consuming kernel page table walks. Unlike the case with MESI-based protocols and per-core CPU caches (i.e., L1, L2, and LLC), the hardware itself is not maintaining core-to-core TLB coherency. Therefore, this task must be performed in software by the operating system. 

我们从前面的章节了解到，TLB是一个快速但容量有限的单核缓存，用于虚拟内存地址到物理内存地址的转换，从而减少耗时的内核页表遍历。与基于MESI协议和单核CPU缓存（例如：L1、L2和LLC）不同，硬件本身并不维护核心间TLB的一致性。因此，这项任务必须由操作系统以软件方式执行。

In a multithreaded application, process threads share the virtual address space. Therefore, the kernel must communicate specific types of updates to that shared address space among the TLBs of the cores on which any of the participating threads execute. For example, commonly used syscalls such as `munmap` (which can be disabled from glibc allocator usage, see [@sec:AvoidPageFaults]), `mprotect`, and `madvise` may invalidate TLB entries.

在多线程应用程序中，进程线程共享虚拟地址空间。因此，内核必须在参与线程执行的各个核心的TLB之间传递对该共享地址空间的特定类型的更新。例如，常用的系统调用，如 `munmap`（可以通过禁用glibc分配器来禁用，参见 [@sec:AvoidPageFaults]）、`mprotect` 和 `madvise` 可能会使TLB条目失效。

These updates must be communicated among the constituent threads of a process. The kernel performs this job using a specific type of Inter Processor Interrupts (IPI), called *TLB shootdowns*, which on x86 platforms are implemented via the `INVLPG` assembly instruction. TLB shootdowns are one of the most overlooked pitfalls to achieving low latency with multithreaded applications.

这些更新必须在进程的各个线程之间传递。内核使用一种称为“TLB崩溃”的特定类型的处理器间中断(IPI: Inter Processor Interrupts)来执行此任务。在x86平台上，TLB崩溃通过汇编指令 `INVLPG` 实现。TLB崩溃是多线程应用程序实现低延迟时最容易被忽视的陷阱之一。

Though a developer may avoid explicitly using these syscalls in his/her code, TLB shootdowns may still erupt from external sources – e.g., memory allocation shared libraries or OS facilities. Not only will this type of IPI disrupt runtime application performance, but the magnitude of its impact grows with the number of threads involved since the interrupts are delivered in software.

即使开发人员避免在代码中显式使用这些系统调用，TLB崩溃仍然可能由外部来源触发，例如内存分配共享库或操作系统功能。这种类型的IPI不仅会影响应用程序的运行时性能，而且由于中断是通过软件传递的，其影响程度会随着线程数的增加而增大。

How do you detect TLB shootdowns in your multithreaded application? One simple way is to check the TLB row in `/proc/interrupts`. A useful method of detecting continuous TLB interrupts during runtime is to use the `watch` command while viewing this file. For example, you might run `watch -n5 -d 'grep TLB /proc/interrupts'`, where the `-n 5` option refreshes the view every 5 seconds while `-d` highlights the delta between each refresh output. 

如何在多线程应用程序中检测TLB崩溃？一个简单的方法是检查 `/proc/interrupts` 文件中的TLB行。在运行时检测连续TLB中断的一个有效方法是在查看此文件时使用 `watch` 命令。例如，您可以运行 `watch -n5 -d 'grep TLB /proc/interrupts'`，其中 `-n 5` 选项每5秒刷新一次视图，而 `-d` 则高亮显示每次刷新输出之间的差值。

[@lst:ProcInterrupts] shows a dump of `/proc/interrupts` with a large number of TLB shootdowns on the `CPU2` processor that ran the latency-critical thread. Notice the order of magnitude difference between other cores. In that scenario, the culprit of such behavior was a Linux kernel feature called Automatic NUMA Balancing, which can be easily disarmed with `sysctl -w numa_balancing=0`.

[@lst:ProcInterrupts] 显示了 `/proc/interrupts` 的转储，其中包含运行延迟关键线程的 `CPU2` 处理器上的大量TLB崩溃。请注意与其他核心之间的数量级差异。在这种情况下，导致这种行为的罪魁祸首是Linux内核的一项名为“自动NUMA平衡”的功能，可以使用 `sysctl -w numa_balancing=0` 轻松禁用它。

Listing: A dump of /proc/interrupts that shows a large number of TLB shootdowns on CPU2
代码列表：`/proc/interrupts`的转储，显示CPU2上的大量TLB崩溃

~~~~ {#lst:ProcInterrupts .cpp}
           CPU0       CPU1       CPU2       CPU3       
...
NMI:          0          0          0          0   Non-maskable interrupts
LOC:     552219    1010298    2272333    3179890   Local timer interrupts
SPU:          0          0          0          0   Spurious interrupts
...
IWI:          0          0          0          0   IRQ work interrupts
RTR:          7          0          0          0   APIC ICR read retries
RES:      18708       9550        771        528   Rescheduling interrupts
CAL:        711        934       1312       1261   Function call interrupts
TLB:       4493       6108      73789       5014   TLB shootdowns
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

But that's not the only source of TLB shootdowns. Others include Transparent Huge Pages, memory compaction, page migration, and page cache writeback. Garbage collectors also can initiate TLB shootdowns. These features either relocate pages and/or alter permissions on pages in the process of fulfilling their duties, which require page table updates and, thus, TLB shootdowns.

但这并非TLB崩溃的唯一原因。其他原因包括透明大页（THP）、内存压缩、页面迁移和页面缓存回写。垃圾回收器也可能引发TLB崩溃。这些功能在履行职责的过程中会重新定位页面和/或更改页面权限，这需要更新页表，从而导致TLB崩溃。

Preventing TLB shootdowns requires limiting the number of updates made to the shared process address space. On the source code level, you should avoid runtime execution of the aforementioned list of syscalls, namely `munmap`, `mprotect`, and `madvise`. On the OS level, disable kernel features that induce TLB shootdowns as a consequence of its function, such as Transparent Huge Pages and Automatic NUMA Balancing. For a more nuanced discussion on TLB shootdowns, along with their detection and prevention, read a related article[^5] on the JabPerf blog.

防止TLB崩溃需要限制对共享进程地址空间的更新次数。在源代码层面，应避免运行时执行上述系统调用，即`munmap`、`mprotect`和`madvise`。在操作系统层面，应禁用那些会因其功能而导致TLB崩溃的内核功能，例如透明大页（THP: Transparent Huge Pages）和自动NUMA平衡（AMB: Automatic NUMA Balancing）。有关TLB崩溃及其检测和预防的更详细讨论，请阅读JabPerf博客上的相关文章[^5]。

### Prevent Unintentional Core Throttling 防止意外的核心节流

C/C++ compilers are a wonderful feat of engineering. However, they sometimes generate surprising results that may lead you on a wild goose chase. A real-life example is an instance where the compiler optimizer emits heavy AVX512 instructions that you never intended. While less of an issue on more modern chips, many older generations of CPUs (which remain in active usage on-premises and in the cloud) exhibit heavy core throttling/downclocking when executing heavy AVX512 instructions. If your compiler produces these instructions without your explicit knowledge or consent, you may experience unexplained latency anomalies during application runtime.

C/C++编译器是一项精妙的工程杰作。然而，它们有时会产生意想不到的结果，让您白费力气。一个实际的例子是，编译器优化器可能会生成您从未打算使用的AVX512指令。虽然在较新的芯片上这个问题并不常见，但许多老一代的CPU（目前仍在本地和云端使用）在执行AVX512指令时会出现严重的核心节流/降频现象。如果您的编译器在您不知情或未经您同意的情况下生成了这些指令，则可能会在应用程序运行时遇到无法解释的延迟异常。

For this specific case, if heavy AVX512 instruction usage is not desired, include `-mprefer-vector-width=###` to your compilation flags to pin the highest width instruction set to either 128 or 256. Again, if your entire server fleet runs on the latest chips then this is much less of a concern since the throttling impact of AVX instruction sets is negligible nowadays.

就此特定情况而言，如果不希望大量使用AVX512指令，请在编译标志中添加 `-mprefer-vector-width=###`，将最大宽度指令集限制为128或256。此外，如果您的所有服务器都运行在最新的芯片上，则无需过多担心，因为如今AVX指令集的性能限制影响可以忽略不计。

[^1]: The Linux Foundation Wiki: Memory for Real-time Applications Linux基金会维基：实时应用程序的内存管理 - [https://wiki.linuxfoundation.org/realtime/documentation/howto/applications/memory](https://wiki.linuxfoundation.org/realtime/documentation/howto/applications/memory)
[^4]: Cache Warming technique 缓存预热技术 - [https://www.youtube.com/watch?v=XzRxikGgaHI](https://www.youtube.com/watch?v=XzRxikGgaHI)
[^5]: JabPerf blog: TLB Shootdowns JabPerf博客：TLB崩溃 - [https://www.jabperf.com/how-to-deter-or-disarm-tlb-shootdowns/](https://www.jabperf.com/how-to-deter-or-disarm-tlb-shootdowns/)
