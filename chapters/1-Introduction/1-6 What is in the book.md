## What Is Discussed in this Book? 本书讨论什么内容？

This book is written to help developers better understand the performance of their applications, learn to find inefficiencies, and eliminate them. 

本书旨在帮助开发者更好地理解应用程序的性能，学习如何发现并消除效率低下的环节。

* Why did my change cause a 2x performance drop? 
* 为什么我的修改导致性能下降了2倍？
* Our customers complain about the slowness of our application. How should I investigate it?
* 我们的客户抱怨我们的应用程序运行缓慢。我应该如何调查原因？
* Why does my handwritten compression algorithm perform slower than the conventional one?
* 为什么我手写的压缩算法比传统的算法运行得更慢？
* Have I optimized my program to its full potential? 
* 我的程序是否已优化到最佳状态？
* What performance analysis tools are available on my platform? 
* 我的平台上有哪些性能分析工具可用？
* What are techniques to reduce the number of cache misses and branch mispredictions?
* 有哪些技术可以减少缓存未命中和分支预测错误？

I hope that by the end of this book, you will be able to answer those questions.

我希望读完本书后，您能够回答这些问题。

The book is split into two parts. The first part (chapters 2--7) teaches you how to find performance problems, and the second part (chapters 8--13) teaches you how to fix them.

本书分为两部分。第一部分（第2章到第7章）教您如何查找性能问题，第二部分（第8章到第13章）教您如何解决这些问题。

* Chapter 2 discusses fair performance experiments and their analysis. It introduces the best practices for performance testing and comparing results.
* Chapter 3 introduces CPU microarchitecture, with a close look at Intel's Golden Cove microarchitecture. 
* Chapter 4 covers terminology and metrics used in performance analysis. At the end of the chapter, we present a case study that features various performance metrics collected on four real-world applications.
* Chapter 5 explores the most popular performance analysis approaches. We describe how profiling tools work and what sort of data they can collect.
* Chapter 6 examines features provided by modern Intel, AMD, and ARM-based CPUs to support and enhance performance analysis. It shows how they work and what problems they help to solve.
* Chapter 7 gives an overview of the most popular performance analysis tools available on Linux, Windows, and MacOS.
* Chapter 8 is about optimizing memory accesses, cache-friendly code, data structure reorganization, and other techniques.
* Chapter 9 is about optimizing computations; it explores data dependencies, function inlining, loop optimizations, and vectorization.
* Chapter 10 is about branchless programming, which is used to avoid branch misprediction.
* Chapter 11 is about machine code layout optimizations, such as basic block placement, function splitting, and profile-guided optimizations.
* Chapter 12 contains optimization topics not considered in the previous four chapters, but still important enough to find their place in this book. In this chapter, we discuss CPU-specific optimizations, examine several microarchitecture-related performance problems, explore techniques used for optimizing low-latency applications, and give you advice on tuning your system for the best performance.
* Chapter 13 discusses techniques for analyzing multithreaded applications. It digs into some of the most important challenges of optimizing multithreaded applications. We provide a case study of five real-world multithreaded applications, where we explain why their performance doesn't scale with the number of CPU threads. We also discuss cache coherency issues (e.g., "false sharing") and a few tools that are designed to analyze multithreaded applications.

* 第2章讨论了公平的性能实验及其分析。它介绍了性能测试和结果比较的最佳实践。
* 第3章介绍CPU微体系结构，重点关注Intel的Golden Cove微体系结构。
* 第4章涵盖性能分析中使用的术语和指标。本章末尾提供了一个案例研究，展示了在四个实际应用程序中收集的各种性能指标。
* 第5章探讨了最常用的性能分析方法。我们描述了性能分析工具的工作原理以及它们可以收集哪些类型的数据。
* 第6章考察了现代Intel、AMD和ARM体系结构CPU为支持和增强性能分析而提供的功能。本章展示了这些功能的工作原理以及它们可以帮助解决哪些问题。
* 第7章概述了Linux、Windows和macOS上最常用的性能分析工具。
* 第8章讨论了内存访问优化、缓存友好型代码、数据结构重组和其他技术。
* 第9章讨论了计算优化；探讨数据依赖、函数内联、循环优化和向量化。
* 第10章介绍无分支编程，它用于避免分支预测错误。
* 第11章介绍机器代码布局优化，例如基本块放置、函数拆分和基于性能分析的优化。
* 第12章包含前四章未涉及但仍然十分重要的优化主题。本章将讨论CPU特定的优化，探讨几个与微体系结构相关的性能问题，探索用于优化低延迟应用程序的技术，并就如何调整系统以获得最佳性能提供建议。
* 第13章讨论分析多线程应用程序的技术。本章深入探讨了优化多线程应用程序的一些最重要挑战。我们提供了五个真实世界多线程应用程序的案例研究，解释了为什么它们的性能不会随着CPU线程数的增加而扩展。我们还将讨论缓存一致性问题（例如：“伪共享”）以及一些用于分析多线程应用程序的工具。

At the end of the book, there is a glossary and a list of microarchitectures for major CPU vendors. Whenever you see an unfamiliar acronym or you need to refresh your memory on recent Intel, AMD, and ARM chip families, refer to these resources.

本书末尾附有术语表和主要CPU厂商的微体系结构列表。一旦您遇到不熟悉的缩写词，或者需要回顾最新的Intel、AMD和ARM芯片家族系列，请参考这些资源。

Examples provided in this book are primarily based on open-source software: Linux as the operating system, the LLVM-based Clang compiler for C and C++ languages, and various open-source applications and benchmarks[^1] that you can build and run. The reason is not only the popularity of these projects but also the fact that their source code is open, which enables us to better understand the underlying mechanism of how they work. This is especially useful for learning the concepts presented in this book. This doesn't mean that we will never showcase proprietary tools. For example, we extensively use Intel® VTune™ Profiler.

本书提供的示例主要基于开源软件：Linux作为操作系统、基于LLVM的Clang编译器用于C和C++语言、以及各种您可以构建和运行的开源应用程序和基准测试程序[^1]。这样做的原因不仅在于这些项目的流行度，还在于它们的源代码是开放的，这使我们能够更好地理解其底层工作原理。这对于学习本书中介绍的概念尤为重要。但这并不意味着我们永远不会展示专有工具。例如，我们大量使用了Intel® VTune™ Profiler。

Sometimes it's possible to obtain attractive speedups by forcing the compiler to generate desired machine code through various hints. You will find many such examples throughout the book. While prior compiler experience helps a lot in performance work, most of the time you don't have to be a compiler expert to drive performance improvements in your application. The majority of optimizations can be done at a source code level without the need to dig down into compiler sources. 

有时，通过各种提示强制编译器生成所需的机器代码，可以获得显著的性能提升。您将在本书中找到许多这样的示例。虽然以往的编译器经验对性能优化大有裨益，但大多数情况下，你无需成为编译器专家也能提升应用程序的性能。大多数优化都可以在源代码层面完成，无需深入研究编译器源代码。

[^1]: Some people don't like when their application is called a "benchmark". They think that a benchmark is something that is synthesized and contrived, and does a poor job of representing real-world scenarios. In this book, we use the terms "benchmark", "workload", and "application" interchangeably and don't mean to offend anyone.


[^1]：有些人不喜欢他们的应用程序被称为“基准测试benchmark”。他们认为基准测试是人为设计出来的，无法很好地代表真实场景。本书中，“基准测试benchmark”、“工作负载workload”和“应用程序application”这几个术语可以互换使用，并无冒犯之意。
