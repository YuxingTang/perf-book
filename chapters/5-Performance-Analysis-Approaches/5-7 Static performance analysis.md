## Static Performance Analysis 静态性能分析

Today we have extensive tooling for static code analysis. For the C and C++ languages, we have well-known tools like Clang static analyzer, Klocwork, Cppcheck, and others. Such tools aim at checking the correctness and semantics of code. Likewise, some tools try to address the performance aspect of code. Static performance analyzers don't execute or profile the program. Instead, they simulate the code as if it is executed on real hardware. Statically predicting performance is almost impossible, so there are many limitations to this type of analysis.

如今，我们拥有丰富的静态代码分析工具。对于C和C++语言，我们有诸如Clang静态分析器、Klocwork、Cppcheck等知名工具。这些工具旨在检查代码的正确性和语义。同样，一些工具也尝试分析代码的性能。静态性能分析器不会执行或分析程序，而是模拟代码在真实硬件上的运行情况。静态预测性能几乎是不可能的，因此这类分析存在诸多局限性。

First, it is not possible to statically analyze C/C++ code for performance since we don't know the machine code to which it will be compiled. So, static performance analysis works on assembly code.

首先，由于我们不知道C/C++代码最终会被编译成什么机器代码，因此无法对其进行静态性能分析。所以，静态性能分析只能针对汇编代码。

Second, static analysis tools simulate the workload instead of executing it. It is obviously very slow, so it's not possible to statically analyze the entire program. Instead, tools take a snippet of assembly code and try to predict how it will behave on real hardware. The user should pick specific assembly instructions (usually a small loop) for analysis. So, the scope of static performance analysis is very narrow.

其次，静态分析工具模拟工作负载，而不是实际执行它。这显然非常缓慢，因此无法对整个程序进行静态分析。相反，这些工具会选取一段汇编代码，并尝试预测它在真实硬件上的运行情况。用户需要选择特定的汇编指令（通常是一个小循环）进行分析。因此，静态性能分析的范围非常狭窄。

The output of static performance analyzers is fairly low-level and often breaks execution down to CPU cycles. Usually, developers use it for fine-grained tuning of a critical code region in which every CPU cycle matters.

静态性能分析器的输出相当底层，通常会将执行过程分解到CPU周期。开发人员通常使用它来对关键代码区域进行细粒度的调优，因为每个CPU周期都至关重要。

### Static vs. Dynamic Analyzers 静态分析器与动态分析器 {.unlisted .unnumbered}

**Static tools**. They don't run actual code but try to simulate the execution, keeping as many microarchitectural details as they can. They are not capable of doing real measurements (execution time, performance counters) because they don't run the code. The upside here is that you don't need to have real hardware and can simulate the code for different CPU generations. Another benefit is that you don't need to worry about the consistency of the results: static analyzers will always give you deterministic output because simulation (in comparison with the execution on real hardware) is not biased in any way. The downside of static tools is that they usually can't predict and simulate everything inside a modern CPU: they are based on a model that may have bugs and limitations. Examples of static performance analyzers are [UICA](https://uica.uops.info/)[^2] and [llvm-mca](https://llvm.org/docs/CommandGuide/llvm-mca.html).[^3]

**静态工具**。它们不运行实际代码，而是尝试模拟执行过程，并尽可能保留微体系结构细节。由于它们不运行代码，因此无法进行实际测量（执行时间、性能计数器）。其优点在于，您无需拥有真实的硬件，并且可以模拟对于不同代次CPU的代码性能。另一个优点是您无需担心结果的一致性：静态分析器始终会给出确定性的输出，因为模拟（与在真实硬件上执行相比）没有任何偏差。静态工具的缺点在于它们通常无法预测和模拟现代CPU内部的所有情况：它们基于可能存在缺陷和局限性的模型。静态性能分析器的例子包括 [UICA](https://uica.uops.info/)[^2]和[llvm-mca](https://llvm.org/docs/CommandGuide/llvm-mca.html)[^3]。

**Dynamic tools**. They are based on running code on real hardware and collecting all sorts of information about the execution. This is the only 100% reliable method of proving any performance hypothesis. As a downside, usually, you are required to have privileged access rights to collect low-level performance data like PMCs. It's not always easy to write a good benchmark and measure what you want to measure. Finally, you need to filter the noise and different kinds of side effects. Two examples of dynamic microarchitectural performance analyzers are [nanoBench](https://github.com/andreas-abel/nanoBench)[^5] and [uarch-bench](https://github.com/travisdowns/uarch-bench).[^4] 

**动态工具**。它们基于在真实硬件上运行代码并收集各种执行信息。这是验证任何性能假设的唯一100%可靠的方法。缺点是，通常需要特权访问权限才能收集底层性能数据，例如：PMC（性能监测计数器）。编写一个好的基准测试并测量所需指标并非易事。最后，还需要过滤噪声和各种副作用。动态微体系结构性能分析工具的两个例子是[nanoBench](https://github.com/andreas-abel/nanoBench)[^5]和[uarch-bench](https://github.com/travisdowns/uarch-bench)[^4]。

A bigger collection of tools both for static and dynamic microarchitectural performance analysis is available [here](https://github.com/MattPD/cpplinks/blob/master/performance.tools.md#microarchitecture).[^7]

[此处](https://github.com/MattPD/cpplinks/blob/master/performance.tools.md#microarchitecture).[^7]提供了更全面的静态和动态微体系结构性能分析工具集合。

### Case Study: Using UICA to Optimize FMA Throughput 案例研究：使用UICA优化FMA吞吐率 {#sec:FMAThroughput}

One of the questions developers often ask is: "The latest processors have 10+ execution units; how do I write my code to keep them busy all the time?" This is indeed one of the hardest questions to tackle. Sometimes it requires looking under the microscope at how the program is running. One such microscope is the UICA simulator which helps you gain insights into how your code could be flowing through a modern processor.

开发者经常会问：“最新的处理器拥有10多个执行单元；我该如何编写代码才能让它们始终保持忙碌？”这确实是最难解决的问题之一。有时，我们需要仔细分析程序的运行机制。UICA模拟器就是这样一款工具，它能帮助你深入了解代码在现代处理器中的运行情况。

Let's look at the code in [@lst:FMAthroughput]. I intentionally try to make the examples as simple as possible. Though real-world codes are of course usually more complicated than this. The code scales every element of array `a` by the floating-point value `B` and accumulates products into `sum`. On the right, I present the machine code for the loop generated by Clang-16 when compiled with `-O3 -ffast-math -march=core-avx2`.

我们来看一下 [@lst:FMAthroughput] 中的代码。我特意让示例尽可能简单。当然，实际代码通常比这复杂得多。这段代码将数组 `a` 的每个元素乘以浮点值 `B`，并将乘积累加到 `sum` 中。右侧展示的是使用 `-O3 -ffast-math -march=core-avx2` 编译时，Clang-16 生成的循环机器代码。

Listing: FMA throughput
代码列表：FMA吞吐率

~~~~ {#lst:FMAthroughput .cpp}
float foo(float * a, float B, int N){  │ .loop:
  float sum = 0;                       │  vfmadd231ps ymm2, ymm1, [rdi + rsi]
  for (int i = 0; i < N; i++)          │  vfmadd231ps ymm3, ymm1, [rdi + rsi + 32]
    sum += a[i] * B;                   │  vfmadd231ps ymm4, ymm1, [rdi + rsi + 64]
  return sum;                          │  vfmadd231ps ymm5, ymm1, [rdi + rsi + 96]
}                                      │  sub rsi, -128
                                       │  cmp rdx, rsi
                                       │  jne .loop
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

This is a reduction loop, i.e., we need to sum up all the products and in the end, return a single float value. The way this code is written, there is a loop-carry dependency over `sum`. You cannot overwrite `sum` until you accumulate the previous product. A smart way to parallelize this is to have multiple accumulators and roll them up in the end. So, instead of a single `sum`, we could have `sum1` to accumulate results from even iterations and `sum2` from odd iterations. 

这是一个归约循环，也就是说，我们需要将所有乘积相加，最终返回一个浮点值。这段代码的编写方式存在循环进位依赖关系，即在累加前一次乘积之前无法覆盖 `sum`。一种巧妙的并行化方法是使用多个累加器，并在最后将它们合并。因此，我们可以用 `sum1` 来累加偶数次迭代的结果，用 `sum2` 来累加奇数次迭代的结果，而不是使用单个 `sum`。

This is what Clang-16 has done: it has 4 vectors (`ymm2`-`ymm5`) each holding 8 floating-point accumulators, plus it used FMA to fuse multiplication and addition into a single instruction. The constant `B` is broadcast into the `ymm1` register. The `-ffast-math` option allows a compiler to reassociate floating-point operations; we will discuss how this option can aid optimizations in [@sec:Vectorization].[^9]

Clang-16正是这样做的：它使用了4个向量（`ymm2` 到 `ymm5`），每个向量包含8个浮点累加器，并且使用了FMA将乘法和加法合并成一条指令。常量 `B` 被广播到 `ymm1` 寄存器中。`-ffast-math` 选项允许编译器重新关联浮点运算。我们将讨论此选项如何帮助优化 [@sec:Vectorization].[^9]

The code looks good, but is it optimal? Let's find out. We took the assembly snippet from [@lst:FMAthroughput] to UICA and ran simulations. At the time of writing, Alder Lake (Intel's 12th gen, based on Golden Cove) is not supported by UICA, so we ran it on the latest available, which is Rocket Lake (Intel's 11th gen, based on Sunny Cove). Although the architectures differ, the issue exposed by this experiment is equally visible in both. The result of the simulation is shown in Figure @fig:FMA_tput_UICA. This is a pipeline diagram similar to what we have shown in Chapter 3. We skipped the first two iterations, and show only iterations 2 and 3 (leftmost column "It."). This is when the execution reaches a steady state, and all further iterations look very similar.

代码看起来不错，但它是否最优呢？让我们一探究竟。我们将 [@lst:FMAthroughput] 中的汇编代码片段导入UICA并运行模拟。截至撰写本文时，UICA尚不支持Alder Lake（Intel第12代Core处理器，基于Golden Cove体系结构），因此我们使用了最新的可用体系结构Rocket Lake（Inte第11代Core处理器，基于Sunny Cove体系结构）进行测试。尽管体系结构不同，但此实验暴露的问题在两者中同样存在。仿真结果如图 @fig:FMA_tput_UICA 所示。这是一个类似于我们在第3章中展示的流水线图。我们跳过了前两次迭代，仅显示迭代2和3（最左侧的“It.”列）。此时执行达到稳定状态，所有后续迭代看起来都非常相似。

![UICA pipeline diagram. `I` = issued, `r` = ready for dispatch, `D` = dispatched, `E` = executed, `R` = retired. UICA流水线图。 `I` = 已发射，`r` = 准备派发，`D` = 已派发，`E` = 已执行，`R` = 已完成。](../../img/perf-analysis/fma_tput_uica.png){#fig:FMA_tput_UICA width=100%}

UICA is a very simplified model of the actual CPU pipeline. For example, you may notice that the instruction fetch and decode stages are missing. Also, UICA doesn't account for cache misses and branch mispredictions, so it assumes that all memory accesses always hit in the L1 cache and branches are always predicted correctly, which we know is not the case in modern processors. Again, this is irrelevant to our experiment as we could still use the simulation results to find a way to improve the code. 

UICA是实际CPU流水线的一个非常简化的模型。例如，你可能会注意到指令提取和解码阶段缺失。此外，UICA没有考虑缓存未命中和分支预测错误，因此它假设所有内存访问总是命中L1缓存，并且分支总是被正确预测，但我们知道这在现代处理器中并非如此。不过，这与我们的实验无关，因为我们仍然可以使用模拟结果来找到改进代码的方法。

Can you see the performance issue? Let's examine the diagram. First of all, every `FMA` instruction is broken into two $\mu$ops (see \circled{1} in Figure @fig:FMA_tput_UICA): a load $\mu$op that goes to ports `{2,3}` and an FMA $\mu$op that can go to ports `{0,1}`. The load $\mu$op has a latency of 5 cycles: it starts at cycle 7 and finishes at cycle 11. The FMA $\mu$op has a latency of 4 cycles: it starts at cycle 15 and finishes at cycle 18. All FMA $\mu$ops depend on load $\mu$ops, as we can see in the diagram: FMA $\mu$ops always start after the corresponding load $\mu$op finishes. Now find two `r` cells at cycle 6, they are ready to be dispatched, but Rocket Lake has only two load ports, and both are already occupied in the same cycle. So, these two loads are issued in the next cycle.

您能看出性能问题吗？让我们来看一下图表。首先，每个 `FMA` 指令都被拆分成两个微操作μop（参见图 @fig:FMA_tput_UICA 中的 \circled{1}）：一个加载微操作μop，目标端口为 `{2,3}`；一个FMA微操作μop，目标端口为 `{0,1}`。加载微操作μop的延迟为5个时钟周期：它在第7个时钟周期开始，在第11个时钟周期结束。FMA微操作μop的延迟为4个时钟周期：它在第15个时钟周期开始，在第18个时钟周期结束。所有FMA微操作μop都依赖于加载微操作μop，如图所示：FMA微操作μop总是在相应的加载微操作μop完成后开始。现在假设在第 6个时钟周期找到两个 `r` 单元，它们已准备好被调度，但Rocket Lake只有两个加载端口，并且在同一时钟周期内都已被占用。因此，这两个加载load操作将在下一个循环中发出。

The loop has four cross-iteration dependencies over `ymm2-ymm5`. The FMA $\mu$op from instruction \circled{2} that writes into `ymm2` cannot start execution before instruction \circled{1} from the previous iteration finishes. Notice that the FMA $\mu$op from instruction \circled{2} was dispatched in the same cycle 18 as instruction \circled{1} finished its execution. There is a data dependency between instruction \circled{1} and instruction \circled{2}. You can observe this pattern for other FMA instructions as well.

该循环对 `ymm2-ymm5` 存在四个跨迭代的依赖关系。来自指令 \circled{2} 的写入 `ymm2` 的FMA微操作μop必须在上一次迭代的指令 \circled{1} 执行完毕后才能开始执行。请注意，指令 \circled{2} 的 FMA微操作μop与指令 \circled{1} 执行完毕是在同一循环（第18个循环）中分派的。指令 \circled{1} 和指令 \circled{2} 之间存在数据依赖关系。您也可以在其他 FMA 指令中观察到这种模式。

So, "What is the problem?", you ask. Look at the top right corner of the image. For each cycle, we added the number of executed FMA $\mu$ops (this is not printed by UICA). It goes like `1,2,1,0,1,2,1,...`, or an average of one FMA $\mu$op per cycle. Most of the recent Intel processors have two FMA execution units, thus can issue two FMA $\mu$ops per cycle. Thus, we utilize only half of the available FMA execution throughput. The diagram clearly shows the gap as every fourth cycle there are no FMAs executed. As we figured out before, no FMA $\mu$ops can be dispatched because their inputs (`ymm2-ymm5`) are not ready.

那么，“问题出在哪里呢？”您可能会问。请查看图像的右上角。对于每个循环，我们都添加了已执行的FMA微操作μop的数量（UICA不会打印此值）。它的流程是 `1,2,1,0,1,2,1,...`，平均每个周期执行一次FMA微操作μop。大多数最新的Intel处理器都配备了两个FMA执行单元，因此每个周期可以执行两次FMA微操作μop。这样一来，我们只利用了可用FMA执行吞吐量的一半。图中清晰地显示了这种差距：每四个周期就没有FMA操作执行。正如我们之前所发现的，由于输入（`ymm2-ymm5`）尚未准备就绪，因此无法调度FMA微操作μop。

To increase the utilization of FMA execution units from 50% to 100%, we need to double the number of accumulators from 4 to 8, effectively unrolling the loop by a factor of two. Instead of 4 independent data flow chains, we will have 8. I'm not showing simulations of the unrolled version here; you can experiment on your own. Instead, let us confirm the hypothesis by running both versions on real hardware. By the way, it is always a good idea to verify, because static performance analyzers like UICA are not accurate models. Below, we show the output of two [nanoBench](https://github.com/andreas-abel/nanoBench) tests that we ran on an Alder Lake processor. The tool takes provided assembly instructions (`-asm` option) and creates a benchmark kernel. Readers can look up the meaning of other parameters in the nanoBench documentation. The original code on the left executes 4 instructions in 4 cycles, while the improved version can execute 8 instructions in 4 cycles. Now we can be sure we have maximized the FMA execution throughput since the code on the right keeps the FMA units busy all the time.

为了将FMA执行单元的利用率从50%提升到100%，我们需要将累加器的数量从4个增加到8个，这相当于将循环展开了两倍。这样一来，我们将拥有8条独立的数据流链，而不是原来的4条。这里我没有展示展开后的版本的模拟结果；您可以自行实验。相反，让我们通过在实际硬件上运行这两个版本来验证这个假设。顺便说一句，验证总是有益的，因为像UICA这样的静态性能分析器并非精确的模型。下面，我们展示了在Alder Lake处理器上运行的两个[nanoBench](https://github.com/andreas-abel/nanoBench)测试的输出结果。该工具接受提供的汇编指令（`-asm` 选项），并创建一个基准测试内核。读者可以在nanoBench文档中查找其他参数的含义。左侧的原始代码在4个周期内执行了4条指令，而改进后的版本可以在4个周期内执行8条指令。现在我们可以确信，我们已经最大限度地提高了FMA执行吞吐量，因为右侧的代码使FMA单元始终保持忙碌状态。

```
# ran on Intel Core i7-1260P (Alder Lake)
$ sudo ./kernel-nanoBench.sh -f    │  $ sudo ./kernel-nanoBench.sh -f 
 -basic -loop 100 -unroll 1000     │   -basic -loop 100 -unroll 1000 
 -warm_up_count 10 -asm "          │   -warm_up_count 10  -asm "
VFMADD231PS YMM0, YMM1, [R14];     │  VFMADD231PS YMM0, YMM1, [R14];
VFMADD231PS YMM2, YMM1, [R14+32];  │  VFMADD231PS YMM2, YMM1, [R14+32];
VFMADD231PS YMM3, YMM1, [R14+64];  │  VFMADD231PS YMM3, YMM1, [R14+64];
VFMADD231PS YMM4, YMM1, [R14+96];" │  VFMADD231PS YMM4, YMM1, [R14+96];
-asm_init "<not shown>"            │  VFMADD231PS YMM5, YMM1, [R14+128];
                                   │  VFMADD231PS YMM6, YMM1, [R14+160];
Instructions retired: 4.00         │  VFMADD231PS YMM7, YMM1, [R14+192];
Core cycles: 4.00                  │  VFMADD231PS YMM8, YMM1, [R14+224]"
                                   │  -asm_init "<not shown>"
                                   │
                                   │  Instructions retired: 8.00
                                   │  Core cycles: 4.00
```

As a rule of thumb, in such situations, the loop must be unrolled by a factor of `T * L`, where `T` is the throughput of an instruction, and `L` is its latency. In our case, we should have unrolled it by `2 * 4 = 8` to achieve maximum FMA port utilization since the throughput of FMA on Alder Lake is 2 and the latency of FMA is 4 cycles. This creates 8 separate data flow chains that can be executed independently.

根据经验，在这种情况下，循环必须按 `T * L` 因子展开，其中 `T` 是指令的吞吐量， `L` 是其延迟。在我们的例子中，我们应该按 `2 * 4 = 8` 展开它，以实现最大的FMA端口利用率，因为Alder Lake上FMA的吞吐率为2，FMA的延迟为4个周期。这创建了8个可以独立执行的独立数据流链。

It's worth mentioning that you will not always see a 2x speedup in practice. This can be achieved only in an idealized environment like UICA or nanoBench. In a real application, even though you maximized the execution throughput of FMA, the gains may be hindered by eventual cache misses and other pipeline hazards. When that happens, the effect of cache misses outweighs the effect of suboptimal FMA port utilization, which could easily result in a much more disappointing 5% speedup. But don't worry; you've still done the right thing. 

值得一提的是，在实践中您并不总是会看到2倍的加速。这只能在UICA或nanoBench等理想化环境中实现。在实际应用程序中，即使您最大化了FMA的执行吞吐量，最终的缓存未命中和其他流水线冲突也可能会阻碍收益。当这种情况发生时，缓存未命中的影响超过了次优FMA端口利用率的影响，这很容易导致令人失望的5%加速。但别担心；你仍然做对了。

As a closing thought, let us remind you that UICA or any other static performance analyzer is not suitable for analyzing large portions of code. But they are great for exploring microarchitectural effects. Also, they help you to build up a mental model of how a CPU works. Another very important use case for UICA is to find critical dependency chains in a loop as described in a [post](https://easyperf.net/blog/2022/05/11/Visualizing-Performance-Critical-Dependency-Chains)[^8] on the Easyperf blog.

最后，让我们提醒您，UICA或任何其他静态性能分析器都不适合分析大部分代码。但它们非常适合探索微体系结构效果。此外，它们还可以帮助您建立CPU工作原理的心理模型。UICA的另一个非常重要的用例是在循环中查找关键依赖链，如Easyperf博客上的[帖子](https://easyperf.net/blog/2022/05/11/Visualizing-Performance-Critical-Dependency-Chains)[^8]中所述。

[^2]: UICA - [https://uica.uops.info/](https://uica.uops.info/)
[^3]: LLVM MCA - [https://llvm.org/docs/CommandGuide/llvm-mca.html](https://llvm.org/docs/CommandGuide/llvm-mca.html)
[^4]: uarch-bench - [https://github.com/travisdowns/uarch-bench](https://github.com/travisdowns/uarch-bench)
[^5]: nanoBench - [https://github.com/andreas-abel/nanoBench](https://github.com/andreas-abel/nanoBench)
[^7]: Collection of links for C++ performance tools C++性能工具链接集合 - [https://github.com/MattPD/cpplinks/blob/master/performance.tools.md#microarchitecture](https://github.com/MattPD/cpplinks/blob/master/performance.tools.md#microarchitecture).
[^8]: Easyperf blog Easyperf博客 - [https://easyperf.net/blog/2022/05/11/Visualizing-Performance-Critical-Dependency-Chains](https://easyperf.net/blog/2022/05/11/Visualizing-Performance-Critical-Dependency-Chains)
[^9]: By the way, it would be possible to sum all the elements in `a` and then multiply it by `B` once outside of the loop. This is an oversight by the programmer, but hopefully, compilers will be able to handle it in the future. 顺便说一句，可以将“a”中的所有元素相加，然后在循环外将其乘以“B”。这是程序员的疏忽，但希望编译器将来能够处理它。
