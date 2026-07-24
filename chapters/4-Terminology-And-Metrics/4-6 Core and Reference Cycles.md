## Core vs. Reference Cycles 核心周期 vs. 参考周期 {#sec:secRefCycles}

Most CPUs employ a clock signal to pace their sequential operations. The clock signal is produced by an external generator that provides a consistent number of pulses each second. The frequency of the clock pulses determines the rate at which a CPU executes instructions. Consequently, the faster the clock, the more instructions the CPU will execute each second.
大多数CPU使用时钟信号来控制其顺序操作。时钟信号由外部发生器产生，每秒提供固定数量的脉冲。时钟脉冲的频率决定了CPU执行指令的速率。因此，时钟频率越高，CPU每秒执行的指令就越多。
$$
Frequency = \frac{Clockticks}{Time}
$$
$$
频率 = \frac{时钟周期数}{时间}
$$
The majority of modern CPUs, including Intel and AMD CPUs, don't have a fixed frequency at which they operate. Instead, they implement dynamic frequency scaling, which is called *Turbo Boost* in Intel's CPUs, and *Turbo Core* in AMD processors. It enables the CPU to increase and decrease its frequency dynamically. Decreasing the frequency reduces power consumption at the expense of performance, and increasing the frequency improves performance but sacrifices power savings.

大多数现代CPU，包括Intel和AMD的CPU，都没有固定的运行频率。相反，它们实现了动态频率调节，在Intel的CPU中称为“*睿频加速（Turbo Boost）*”，在AMD的处理器中称为“*睿频核心（Turbo Core）*”。它使CPU能够动态地提高或降低其频率。降低频率可以降低功耗，但会牺牲性能；而提高频率可以提高性能，但会牺牲一些功耗。

The core clock cycles counter is counting clock cycles at the actual frequency that the CPU core is running at. The reference clock event counts cycles as if the processor is running at the base frequency. Let's take a look at an experiment on a Skylake i7-6000 processor running a single-threaded application, which has a base frequency of 3.4 GHz:
核心时钟周期计数器统计的是CPU核心实际运行频率下的时钟周期数。参考时钟事件则统计的是处理器以基准频率运行时的时钟周期数。我们来看一个在Skylake i7-6000处理器上运行单线程应用程序的实验，该处理器的基准频率为3.4GHz：

```bash
$ perf stat -e cycles,ref-cycles -- ./a.exe
  43340884632  cycles		# 3.97 GHz
  37028245322  ref-cycles	# 3.39 GHz
      10,899462364 seconds time elapsed
```

The `ref-cycles` event counts cycles as if there were no frequency scaling. The external clock on the platform has a frequency of 100 MHz, and if we scale it by the *clock multiplier*, we will get the base frequency of the processor. The clock multiplier for the Skylake i7-6000 processor equals 34: it means that for every external pulse, the CPU executes 34 internal cycles when it's running on the base frequency (i.e., 3.4 GHz).

`ref-cycles` 事件统计的是没有频率缩放时的时钟周期数。平台上的外部时钟频率为100MHz，乘以时钟倍频器后，即可得到处理器的基准频率。Skylake i7-6000处理器的时钟倍频器为34：这意味着，当CPU以基准频率（即3.4GHz）运行时，每接收到一个外部时钟脉冲，CPU就会执行34个内部时钟周期。

The `cycles` event counts real CPU cycles and takes into account frequency scaling. Using the formula above we can confirm that the average operating frequency was `43340884632 cycles / 10.899 sec = 3.97 GHz`. When you compare the performance of two versions of a small piece of code, measuring the time in clock cycles is better than in nanoseconds, because you avoid the problem of the clock frequency going up and down. 

`cycles` 事件统计的是实际的CPU时钟周期数，并考虑了频率缩放的影响。使用上述公式，我们可以确认平均运行频率为 `43340884632 cycles / 10.899 sec = 3.97 GHz`。在比较同一段代码的两个版本性能时，以时钟周期为单位测量时间比以纳秒为单位测量时间更准确，因为这样可以避免时钟频率波动带来的影响。
