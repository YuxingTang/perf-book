## CPI and IPC 每指令周期数CPI与每周期指令数IPC {#sec:IPC}

Those are two fundamental metrics that stand for:

这两个基本指标分别代表：

* Instructions Per Cycle (IPC) - how many instructions were retired per cycle on average.

* 每周期指令数(IPC: Instructions Per Cycle) - 平均每个周期执行的指令数。

  $$
  IPC = \frac{INST\_RETIRED.ANY}{CPU\_CLK\_UNHALTED.THREAD},
  $$

where `INST_RETIRED.ANY` counts the number of retired instructions, and `CPU_CLK_UNHALTED.THREAD` counts the number of core cycles while the thread is not in a halt state.

其中，`INST\_RETIRED.ANY` 统计已执行完成的指令数，`CPU\_CLK\_UNHALTED.THREAD` 统计线程未处于停止状态时核心运行的周期数。

* Cycles Per Instruction (CPI) - how many cycles it took to retire one instruction on average.

* 每指令周期数(CPI: Cycles Per Instruction) - 平均执行一条指令所需的周期数。

$$
CPI = \frac{1}{IPC}
$$

Using one or another is a matter of preference. I prefer to use IPC as it is easier to compare. With IPC, we want as many instructions per cycle as possible, so the higher the IPC, the better. With `CPI`, it's the opposite: we want as few cycles per instruction as possible, so the lower the CPI the better. The comparison that uses "the higher the better" metric is simpler since you don't have to do the mental inversion every time. In the rest of the book, we will mostly use IPC, but again, there is nothing wrong with using CPI either.

使用哪个指标取决于个人偏好。我更倾向于使用IPC，因为它更容易比较。对于IPC而言，我们希望每个周期执行的指令数越多越好，因此IPC值越高越好。对于CPI（每指令周期数），情况则相反：我们希望每条指令所需的周期数越少越好，因此CPI越低越好。使用“越高越好”指标的比较方法更简单，因为无需每次都进行逻辑上的转换。本书后续内容将主要使用IPC（每指令周期数），但再次强调，使用CPI也完全没有问题。

The relationship between IPC and CPU clock frequency is very interesting. In the broad sense, `performance = work / time`, where we can express work as the number of instructions and time as seconds. The number of seconds a program was running can be calculated as `total cycles / frequency`: 

IPC和CPU时钟频率之间的关系非常有趣。广义上讲，`性能 = 工作量 / 时间`，其中工作量可以用指令数表示，时间可以用秒数表示。程序运行的秒数可以计算为 `总周期数 / 频率` ：

$$
Performance = \frac{instructions \times frequency}{cycles} = IPC \times frequency
$$

$$
性能 = \frac{指令数 \times 主频}{周期数} = IPC \times 主频
$$

As we can see, performance is proportional to IPC and frequency. If we increase any of the two metrics, the performance of a program will grow.

可以看出，性能与IPC和频率成正比。如果提高这两个指标中的任何一个，程序的性能都会提升。

From the perspective of benchmarking, IPC and frequency are two independent metrics. I've seen some engineers mistakenly mixing them up and thinking that if you increase the frequency, the IPC will also go up. But that's not true. If you clock a processor at 1 GHz instead of 5 GHz, for many applications you will still get the same IPC.[^1] It may sound very confusing, especially since IPC has everything to do with CPU clocks. However, frequency only tells us how fast a single clock cycle is, whereas IPC counts how much work is done every cycle. So, from the benchmarking perspective, IPC solely depends on the design of the processor regardless of the frequency. Out-of-order cores typically have a much higher IPC than in-order cores. When you increase the size of CPU caches or improve branch prediction, the IPC usually goes up.

从基准测试的角度来看，IPC和频率是两个独立的指标。我见过一些工程师错误地将频率和IPC混淆，认为提高频率就能提高IPC。但事实并非如此。如果将处理器频率从5GHz降至1GHz，对于许多应用来说，IPC仍然相同[^1]。这听起来可能很令人困惑，尤其因为IPC与CPU时钟频率密切相关。然而，频率仅表示单个时钟周期的速度，而IPC则统计每个周期完成的工作量。因此，从基准测试的角度来看，IPC完全取决于处理器的设计，与频率无关。乱序执行核心的IPC通常远高于顺序执行核心。增加CPU缓存的大小或改进分支预测通常可以提高IPC。

Now, if you ask a hardware architect, they will certainly tell you there is a dependency between IPC and frequency. From the CPU design perspective, you can deliberately downclock the processor, which will make every cycle longer and make it possible to squeeze more work into each cycle. In the end, you will get a higher IPC but a lower frequency. Hardware vendors approach this performance equation in different ways. For example, Intel and AMD chips usually have very high frequencies, with the recent Intel 13900KS processor providing a 6 GHz turbo frequency out of the box with no overclocking required. On the other hand, Apple M1/M2 chips have lower frequency but compensate with a higher IPC. Lower frequency facilitates lower power consumption. Higher IPC, on the other hand, usually requires a more complicated design, more transistors, and a larger die size. We will not go into all the design tradeoffs here, as they are topics for a different book.

当然，如果你问硬件体系结构设计师，他们肯定会告诉你IPC和频率之间存在依赖关系。从CPU设计的角度来看，你可以故意降低处理器频率，这将延长每个周期，从而在每个周期内完成更多的工作。最终，你会获得更高的IPC（每时钟周期指令数），但频率会降低。硬件厂商处理这种性能平衡的方式各不相同。例如，Intel和AMD的芯片通常具有非常高的频率，最新的Intel 13900KS处理器无需超频即可提供6GHz的默认睿频频率。另一方面，苹果M1/M2芯片的频率较低，但以更高的IPC来弥补。较低的频率有助于降低功耗。而更高的IPC通常需要更复杂的设计、更多的晶体管和更大的芯片管芯尺寸。我们在此不赘述所有设计上的权衡，因为这些内容值得另著一书详述。

IPC is useful for evaluating both hardware and software efficiency. Hardware engineers use this metric to compare CPU generations and CPUs from different vendors. Since IPC is the measure of the performance of a CPU microarchitecture, engineers and media use it to express gains over the previous generation. However, to make a fair comparison, you need to run both systems at the same frequency.

IPC对于评估硬件和软件的效率都非常有用。硬件工程师使用这一指标来比较不同代CPU以及不同厂商的CPU。由于IPC是衡量CPU微体系结构性能的指标，工程师和媒体用它来表示相比上一代产品的性能提升。然而，为了进行公平的比较，你需要让两个系统运行在相同的频率下。

IPC is also a useful metric for evaluating software. It gives you an intuition for how quickly instructions in your application progress through the CPU pipeline. Later in this chapter, you will see several production applications with varying IPCs. Memory-intensive applications usually have a low IPC (0--1), while computationally intensive workloads tend to have a high IPC (4--6).

IPC（每时钟周期指令数）也是评估软件性能的一个有用指标。它能让你直观地了解应用程序中的指令在CPU流水线中的执行速度。本章稍后将介绍几个IPC值不同的生产环境应用程序。内存密集型应用程序的IPC通常较低（0-1），而计算密集型工作负载的IPC则往往较高（4-6）。

Linux `perf` users can measure the IPC for their workload by running:

Linux `perf` 用户可以通过运行以下命令来测量其工作负载的IPC：

```bash
$ perf stat -e cycles,instructions -- a.exe
  2369632  cycles                               
  1725916  instructions  #    0,73  insn per cycle
# or as simple as:
$ perf stat -- ./a.exe
```

[^1]: When you lower CPU frequency, memory speed becomes faster relative to the CPU. This may hide actual memory bottlenecks and artificially increase IPC.降低CPU频率后，内存速度相对于CPU速度会加快。这可能会掩盖实际的内存瓶颈，并人为地提高IPC（每时钟周期指令数）。
