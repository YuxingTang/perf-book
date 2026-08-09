# Optimizing Computations 优化计算 {#sec:CoreBound}

In the previous chapter, we discussed how to clear the path for efficient memory access. Once that is done, it's time to look at how well a CPU works with the data it brings from memory. Modern applications demand a large amount of CPU computations, especially applications involving complex graphics, artificial intelligence, cryptocurrency mining, and big data processing. In this chapter, we will focus on optimizing computations that can reduce the amount of work a CPU needs to do and improve the overall performance of a program.

在上一章中，我们讨论了如何为高效的内存访问扫清障碍。完成这一步后，接下来需要考察CPU处理从内存中获取的数据的效率。现代应用程序需要大量的CPU计算，尤其是涉及复杂图形、人工智能、加密货币挖矿和大数据处理的应用程序。本章将重点讨论如何优化计算，从而减少CPU的工作量，并提高程序的整体性能。

When the TMA methodology is applied, inefficient computations are usually reflected in the `Core Bound` and, to some extent, in the `Retiring` categories. The `Core Bound` category represents all the stalls inside a CPU out-of-order execution engine that were not caused by memory issues. There are two main categories:

当应用TMA方法时，低效的计算通常会体现在 `核心瓶颈（Core Bound）` 类别中，并在一定程度上体现在 `退出（Retiring）` 类别中。`核心瓶颈（Core Bound）` 类别代表CPU乱序执行引擎中所有非存储问题导致的停顿。主要分为两类：

* Data dependencies between software instructions are limiting the performance. For example, a long sequence of dependent operations may lead to low Instruction Level Parallelism (ILP) and wasting many execution slots. The next section discusses data dependency chains in more detail.
* A shortage in hardware computing resources. This indicates that certain execution units are overloaded (also known as *execution port contention*). This can happen when a workload frequently performs many instructions of the same type. For example, AI algorithms typically perform a lot of multiplications. Scientific applications may run many divisions and square root operations. However, there is a limited number of multipliers and dividers in any given CPU core. Thus when port contention occurs, instructions queue up waiting for their turn to be executed. This type of performance bottleneck is very specific to a particular CPU microarchitecture and usually doesn't have a cure.

* 软件指令之间的数据依赖性限制了性能。例如，过长的依赖操作序列可能导致指令级并行性(ILP: Instruction Level Parallelism)较低，并浪费大量执行槽。下一节将更详细地讨论数据依赖链。
* 硬件计算资源不足。这表明某些执行单元过载（也称为*执行端口争用（execution port contention）*）。当工作负载频繁执行大量相同类型的指令时，就会发生这种情况。例如，人工智能算法通常会执行大量的乘法运算。科学应用程序可能会运行大量的除法和平方根运算。然而，任何给定的CPU核心中的乘法器和除法器数量都是有限的。因此，当发生端口争用时，指令会排队等待执行。这种性能瓶颈与特定的CPU微体系结构密切相关，通常没有有效的解决方法。

In this chapter, we will take a look at well-known techniques like function inlining, vectorization, and loop optimizations. Those code transformations aim to reduce the total amount of executed instructions or replace them with more efficient ones.

在本章中，我们将介绍一些常用的技术，例如：函数内联（function inlining）、向量化（vectorization）和循环优化（loop optimization）。这些代码转换旨在减少执行的指令总数或用更高效的指令替换它们。