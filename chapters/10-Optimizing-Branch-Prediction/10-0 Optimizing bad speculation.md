# Optimizing Branch Prediction 优化分支预测 {#sec:ChapterBadSpec}

So far we've been talking about optimizing memory accesses and computations. However, we haven't discussed another important category of performance bottlenecks yet. It is related to speculative execution, a feature that is present in all modern high-performance CPU cores. To refresh your memory, turn to [@sec:SpeculativeExec] where we discussed how speculative execution can be used to improve performance. In this chapter, we will explore techniques to reduce the number of branch mispredictions.

到目前为止，我们一直在讨论如何优化存储访问和计算。然而，我们还没有讨论另一类重要的性能瓶颈。它与前瞻执行有关，前瞻执行是所有现代高性能CPU核心都具备的特性。为了帮助您回忆，请参阅 [@sec:SpeculativeExec]，我们在那里讨论了如何利用推测执行来提升性能。在本章中，我们将探讨减少分支预测错误次数的技术。

In general, modern processors are very good at predicting branch outcomes. They not only follow static prediction rules but also detect dynamic patterns. Usually, branch predictors save the history of previous outcomes for the branches and try to guess what the next result will be. However, when the pattern becomes hard for the CPU branch predictor to follow, it may hurt performance.

一般来说，现代处理器非常擅长预测分支结果。它们不仅遵循静态预测规则，还能检测动态模式。通常，分支预测器会保存分支的历史记录，并尝试预测下一个结果。然而，当CPU分支预测器难以追踪某种模式时，可能会影响性能。

Mispredicting a branch can add a significant penalty when it happens regularly. When such an event occurs, a CPU is required to clear all the speculative work that was done ahead of time and later was proven to be wrong. It also needs to flush the pipeline and start filling it with instructions from the correct path. Typically, modern CPUs experience a 10- to 25-cycle penalty as a result of a branch misprediction. The exact number of cycles depends on the microarchitecture design, namely, on the depth of the pipeline and the mechanism used to recover from a mispredict.

分支预测错误如果频繁发生，会造成显著的性能损失。当发生分支预测错误时，CPU需要清除所有预先执行但后来被证明错误的前瞻性工作。它还需要清空流水线，并开始从正确的路径填充指令。通常，现代CPU会因分支预测错误而承受10到25个周期的开销。具体的周期数取决于微体系结构设计，即流水线的深度以及用于从预测错误中恢复的机制。

Perhaps the most frequent reason for a branch misprediction is simply because it has a complicated outcome pattern (e.g., exhibits pseudorandom behavior), which is unpredictable for a processor. For completeness, let's cover the other less frequent reasons behind branch mispredicts. Branch predictors use caches and history registers and therefore are susceptible to the issues related to caches, namely:

分支预测错误最常见的原因或许在于其结果模式复杂（例如：表现出伪随机行为），而处理器无法预测这种模式。为了完整起见，我们再来讨论其他一些不太常见的分支预测错误原因。分支预测器使用缓存和历史寄存器，因此容易受到与缓存相关的问题的影响，例如：

- **Cold misses**: mispredictions may happen on the first dynamic occurrence of the branch when no dynamic history is available and static prediction is employed.
- **Capacity misses**: mispredictions arising from the loss of dynamic history due to a very high number of branches in the program or exceedingly long dynamic pattern.
- **Conflict misses**: branches are mapped into cache buckets (associative sets) using a combination of their virtual and/or physical addresses. If too many active branches are mapped to the same set, the loss of history can occur. Another instance of a conflict miss is aliasing when two independent branches are mapped to the same cache entry and interfere with each other potentially degrading the prediction history.

- **冷失效/冷未命中**：当没有动态历史记录可用且采用静态预测时，分支的首次动态执行可能会发生预测错误。
- **容量失效**：由于程序中分支数量过多或动态模式过长，导致动态历史记录丢失，从而引发预测错误。
- **冲突失效**：分支通过其虚拟地址和/或物理地址的组合映射到缓存桶（cache buckets，关联集associative sets）。如果太多活动分支映射到同一个关联集，则可能发生历史记录丢失。冲突失效的另一个例子是别名，即两个独立的分支映射到同一个缓存条目，彼此干扰，从而可能降低预测历史记录的质量。

A program will always experience a non-zero number of branch mispredictions. You can find out how much a program suffers from branch mispredictions by looking at the TMA `Bad Speculation` metric. It is normal for a general-purpose application to have a `Bad Speculation` metric in the range of 5--10\%. My recommendation is to pay close attention once this metric goes higher than 10\%.

程序总是会经历一定数量的分支预测错误。您可以通过查看TMA的 `错误前瞻Bad Speculation` 指标来了解程序受分支预测错误的影响程度。通用应用程序的 `错误前瞻Bad Speculation` 指标通常在5%到10%之间。我的建议是，一旦该指标超过10%，就需要密切关注。

In the past, developers had an option of providing a prediction hint to an x86 processor in the form of a prefix to the branch instruction (`0x2E: Branch Not Taken`, `0x3E: Branch Taken`). This could potentially improve performance on older microarchitectures, like Pentium 4. However, modern x86 processors used to ignore those hints until Intel's RedwoodCove started using it again. Its branch predictor is still good at finding dynamic patterns, but now it will use the encoded prediction hint for branches that have never been seen before (i.e. when there is no stored information about a branch). [@IntelOptimizationManual, Section 2.1.1.1 Branch Hint]

过去，开发者可以选择在分支指令前添加前缀（例如 `0x2E: Branch Not Taken`、`0x3E: Branch Taken`），以此向x86处理器提供预测提示。这有可能提升老旧微体系结构（例如：Pentium4）的性能。然而，现代x86处理器通常会忽略这些提示，直到Intel的RedwoodCove重新启用它们。RedwoodCove的分支预测器仍然擅长发现动态模式，但现在它会对从未出现过的分支（即：没有存储这条分支任何信息的情况）使用编码后的预测提示。[@IntelOptimizationManual，第2.1.1.1节--分支提示]

There are indirect ways to reduce the branch misprediction rate by reducing the dynamic number of branch instructions. This approach helps because it alleviates the pressure on branch predictor structures. When a program executes fewer branch instructions, it may indirectly improve the prediction of branches that previously suffered from capacity and conflict misses. Compiler transformations such as loop unrolling and vectorization help reduce the dynamic branch count, though they don't specifically aim to improve the prediction rate of any given conditional statement. Profile-Guided Optimizations (PGO) and post-link optimizers (e.g., BOLT) are also effective at reducing branch mispredictions thanks to improving the fallthrough rate (straightening the code). We will discuss those techniques in the next chapter.[^1]

可以通过减少分支指令的动态数量来间接降低分支预测错误率。这种方法之所以有效，是因为它减轻了分支预测器结构的压力。当程序执行的分支指令减少时，它可能会间接地改善之前因容量不足和冲突失效而遭受痛苦的分支预测。编译器转换（例如：循环展开和向量化）有助于减少动态分支数量，但它们并非专门针对提高任何给定条件语句的预测率。基于轮廓信息的优化(PGO: Profile-Guided Optimization)和链接后优化器（例如：BOLT）也能有效减少分支预测错误，因为它们提高了向下传递率（使代码更直）。我们将在下一章讨论这些技术[^1]。

The only direct way to get rid of branch mispredictions is to get rid of the branch instruction itself. In subsequent sections, we will take a look at both direct and indirect ways to improve branch prediction. In particular, we will explore the following techniques: replacing branches with lookup tables, arithmetic, selection, and SIMD instructions.

消除分支预测错误的唯一直接方法是移除分支指令本身。在后续章节中，我们将探讨改进分支预测的直接和间接方法。具体来说，我们将研究以下技术：用查找表、算术运算、选择指令和SIMD指令替换分支。

[^1]: There is a conventional wisdom that never-taken branches are transparent to the branch prediction and can't affect performance, and therefore it doesn't make much sense to remove them, at least from a prediction perspective. However, contrary to the wisdom, an experiment conducted by authors of BOLT optimizer demonstrated that replacing never-taken branches with equal-sized no-ops in a large code footprint application, such as Clang C++ compiler, leads to approximately 5\% speedup on modern Intel CPUs. So it still pays to try to eliminate all branches. 人们普遍认为，从未执行跳转的分支对分支预测是透明的，不会影响性能，因此至少从预测的角度来看，移除它们意义不大。然而，与这种观点相反，BOLT优化器的作者进行的一项实验表明，在大型代码应用（例如：Clang C++ 编译器）中，用等长的空操作替换从未执行跳转的分支，在现代Intel CPU上可以带来大约5%的速度提升。因此，尝试消除所有分支仍然是值得的。
