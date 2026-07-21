## Software and Hardware Timers 软件和硬件定时器 {#sec:timers}

To benchmark execution time, engineers usually use two different timers, which all the modern platforms provide:

为了测试执行时间，工程师通常使用两种不同的定时器，所有现代平台都提供这两种定时器：

 - **System-wide high-resolution timer**: this is a system timer that is typically implemented as a simple count of the number of ticks that have transpired since an arbitrary starting date, called the [epoch](https://en.wikipedia.org/wiki/Epoch_(computing))[^1]. This clock is monotonic; i.e., it always goes up. System time can be retrieved from the OS with a system call. Accessing the system timer on Linux systems is possible via the `clock_gettime` system call. The system timer has a nanosecond resolution, is consistent between all the CPUs, and is independent of the CPU frequency. Even though the system timer can return timestamps with a nanosecond accuracy, it is not suitable for measuring short-running events because it takes a long time to obtain the timestamp via the `clock_gettime` system call. But it is OK to measure events with a duration of more than a microsecond. The standard way of accessing the system timer in C++ is using `std::chrono` as shown in [@lst:Chrono].

- **系统级高分辨率定时器**：这是一种系统定时器，通常实现为对自任意起始日期（称为纪元[epoch](https://en.wikipedia.org/wiki/Epoch_(computing))[^1] 以来经过的时钟滴答数进行简单计数。该时钟是单调递增的，也就是说，它始终递增。可以通过系统调用从操作系统获取系统时间。在Linux系统上，可以通过 `clock_gettime` 系统调用访问系统定时器。系统定时器具有纳秒级精度，在所有CPU上保持一致，并且与CPU频率无关。尽管系统定时器可以返回纳秒级精度的时间戳，但它不适合测量短时事件，因为通过 `clock_gettime` 系统调用获取时间戳需要很长时间。但是，测量持续时间超过一微秒的事件是可以的。在C++中访问系统计时器的标准方法是使用 `std::chrono`，如[@lst:Chrono]所示。

   Listing: Using C++ std::chrono to access system timer
   代码列表: 使用 C++ std::chrono 来访问系统定时器

   ~~~~ {#lst:Chrono .cpp}
   #include <cstdint>
   #include <chrono>

   // returns elapsed time in nanoseconds 以纳秒ns量级返回时间
   uint64_t timeWithChrono() {
     using namespace std::chrono;
     auto start = steady_clock::now();
     // run something 运行某些步骤
     auto end = steady_clock::now();
     uint64_t delta = duration_cast<nanoseconds>(end - start).count();
     return delta;
   }
   ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

 - **Time Stamp Counter (TSC)**: this is a hardware timer that is implemented as a hardware register. TSC is monotonic and has a constant rate, i.e., it doesn't account for frequency changes. Every CPU has its own TSC, which is simply the number of reference cycles (see [@sec:secRefCycles]) elapsed. It is suitable for measuring short events with a duration from nanoseconds up to one minute. On x86 platforms, the value of TSC can be retrieved by using the compiler's built-in function `__rdtsc` as shown in [@lst:TSC], which uses the `RDTSC` assembly instruction under the hood. More low-level details on benchmarking the code using the `RDTSC` assembly instruction can be found in the white paper [@IntelRDTSC]. On ARM platforms you can read `CNTVCT_EL0`, Counter-timer Virtual Count Register.

- **时间戳计数器(TSC: Time Stamp Counter)**：这是一个硬件定时器，以硬件寄存器的形式实现。TSC是单调递增的，且具有恒定的速率，也就是说，它不考虑频率变化。每个CPU都有自己的TSC，它简单地表示经过的参考周期数（参见[@sec:secRefCycles]）。它适用于测量持续时间从纳秒到一分钟的短事件。在x86平台上，可以使用编译器内置函数 `__rdtsc` 获取TSC的值，如[@lst:TSC]所示，该函数底层使用了汇编指令 `RDTSC`。有关使用汇编指令 `RDTSC` 对代码进行基准测试的更多底层细节，请参阅白皮书[@IntelRDTSC]。在ARM平台上，您可以读取 `CNTVCT_EL0`，即计数器定时器对应的虚拟计数寄存器。（译者注：可以在CPP中使用内联汇编`asm volatile("mrs %0, cntvct_el0" : "=r"(value));`或者GCC/Clang下都可以使用的内置函数read_cntvct_el0。）

   Listing: Using __rdtsc compiler builtins to access TSC
   代码列表: 使用编译器内置 __rdtsc 来访问时间戳计数器TSC

   ~~~~ {#lst:TSC .cpp}
   #include <x86intrin.h>
   #include <cstdint>

   // returns the number of elapsed reference clocks 返回参考时钟消逝的周期数
   uint64_t timeWithTSC() {
       uint64_t start = __rdtsc();
       // run something
       return __rdtsc() - start;
   }
   ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Choosing which timer to use is very simple and depends on how long the thing is that you want to measure. If you measure something over a very short time period, TSC will give you better accuracy. Conversely, it's pointless to use the TSC to measure a program that runs for hours. Unless you need cycle accuracy, the system timer should be enough for a large proportion of cases. It's important to keep in mind that accessing the system timer usually has a higher latency than accessing TSC. Making a `clock_gettime` system call can be much slower than executing `RDTSC` instruction. The latter takes about 5 ns (20 CPU cycles), while the former takes about 500 ns. This may become important for minimizing measurement overhead, especially in the production environment. A performance comparison of different APIs for accessing timers on various platforms is available on the wiki page of the CppPerformanceBenchmarks repository.[^3]

选择使用哪个定时器非常简单，取决于您要测量的时间段长短。如果测量的时间段很短，TSC定时器会提供更高的精度。相反，使用TSC定时器测量运行数小时的程序是没有意义的。除非您需要周期级的精度，否则系统定时器在大多数情况下都足够用了。需要注意的是，访问系统定时器的延迟通常比访问TSC定时器要高。调用 `clock_gettime` 系统定时器可能比执行 `RDTSC` 指令慢得多。后者大约需要5纳秒（约20个CPU周期），而前者大约需要500纳秒。这对于最大限度地减少测量开销至关重要，尤其是在生产环境中。CppPerformanceBenchmarks代码库的wiki页面上提供了不同平台上访问定时器的各种API的性能比较。[^3]

[^1]: Unix epoch starts on 1 January 1970 00:00:00 UT: Unix纪元始于1970年1月1日 00:00:00世界时：[https://en.wikipedia.org/wiki/Unix_epoch](https://en.wikipedia.org/wiki/Unix_epoch).
[^3]: CppPerformanceBenchmarks wiki - [https://gitlab.com/chriscox/CppPerformanceBenchmarks/-/wikis/ClockTimeAnalysis](https://gitlab.com/chriscox/CppPerformanceBenchmarks/-/wikis/ClockTimeAnalysis)
