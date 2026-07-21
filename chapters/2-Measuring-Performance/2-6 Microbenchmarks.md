## Microbenchmarks 微基准测试

Microbenchmarks are small self-contained programs that people write to quickly test a hypothesis. Usually, microbenchmarks are used to choose the best implementation of a certain relatively small algorithm or functionality. Nearly all modern languages have benchmarking frameworks. In C++, you can use the Google [benchmark](https://github.com/google/benchmark)[^3] library, C# has the [BenchmarkDotNet](https://github.com/dotnet/BenchmarkDotNet)[^4] library, Julia has the [BenchmarkTools](https://github.com/JuliaCI/BenchmarkTools.jl)[^5] package, Java has [JMH](http://openjdk.java.net/projects/code-tools/jmh/etc)[^6] (Java Microbenchmark Harness), Rust has the Criterion[^8] package, etc.

微基准测试是人们编写的小型独立程序，用于快速验证假设。通常，微基准测试用于选择某个相对较小的算法或功能的最佳实现。几乎所有现代编程语言都拥有基准测试框架。在C++中，您可以使用Google的基准测试[benchmark](https://github.com/google/benchmark)[^3]库；C#有[BenchmarkDotNet](https://github.com/dotnet/BenchmarkDotNet)[^4]库；Julia有[BenchmarkTools](https://github.com/JuliaCI/BenchmarkTools.jl)[^5]包；Java有 [JMH](http://openjdk.java.net/projects/code-tools/jmh/etc)[^6]（Java Microbenchmark Harness）；Rust有Criterion[^8]包，等等。

When writing microbenchmarks, it's very important to ensure that the scenario you want to test is actually executed by your microbenchmark at runtime. Optimizing compilers can eliminate important code that could render the experiment useless, or even worse, drive you to the wrong conclusion. In the example below, modern compilers are likely to eliminate the whole loop:

编写微基准测试时，务必确保您要测试的场景在运行时能够被微基准测试实际执行。优化编译器可能会删除一些重要的代码，导致实验失效，甚至更糟的是，驱动你得出错误的结论。在下面的示例中，现代编译器很可能会删除整个循环：

```cpp
// foo DOES NOT benchmark string creation foo实际上没有对字符串创建构成微基准测试
void foo() {
  for (int i = 0; i < 1000; i++)
    std::string s("hi");
}
```

Blunders like that one are nicely captured in the paper "Always Measure One Level Deeper" [@MeasureOneLevelDeeper], where the author advocates for a more scientific approach, and measuring performance from different angles. Following advice from the paper, we should inspect the performance profile of the benchmark and make sure the intended code stands out as the hotspot. Sometimes abnormal timings can be spotted instantly, so use common sense while analyzing and comparing benchmark runs. 

类似这样的错误在论文《始终深入测量一层》（Always Measure One Level Deeper）[@MeasureOneLevelDeeper] 中得到了很好的体现。论文作者提倡采用更科学的方法，从不同角度测量性能。根据论文的建议，我们应该检查基准测试的性能轮廓，确保目标代码能够突出成为热点。有时异常的运行时间可以立即被发现，因此在分析和比较基准测试结果时，请运用常识。

One of the popular ways to keep the compiler from optimizing away important code is to use [`DoNotOptimize`](https://github.com/google/benchmark/blob/c078337494086f9372a46b4ed31a3ae7b3f1a6a2/include/benchmark/benchmark.h#L307)-like[^7] helper functions, which do the necessary inline assembly magic under the hood:

防止编译器优化掉重要代码的一种常用方法是使用类似“不要优化”[`DoNotOptimize`](https://github.com/google/benchmark/blob/c078337494086f9372a46b4ed31a3ae7b3f1a6a2/include/benchmark/benchmark.h#L307)的辅助函数[^7]，这些函数会在底层执行必要的内联汇编操作：

```cpp
// foo benchmarks string creation foo建立了字符串创建的微基准测试
void foo() {
  for (int i = 0; i < 1000; i++) {
    std::string s("hi");
    DoNotOptimize(s);
  }
}
```

If written well, microbenchmarks can be a good source of performance data. They are often used for comparing the performance of different implementations of a critical function. A good benchmark tests performance in realistic conditions. In contrast, if a benchmark uses synthetic input that is different from what will be given in practice, then the benchmark will likely mislead you and will drive you to the wrong conclusions. Besides that, when a benchmark runs on a system free from other demanding processes, it has all resources available to it, including DRAM and cache space. Such a benchmark will likely champion the faster version of the function even if it consumes more memory than the other version. However, the outcome can be the opposite if there are neighbor processes that consume a significant part of DRAM, which causes memory regions that belong to the benchmark process to be swapped to the disk. 

如果编写得当，微基准测试可以成为性能数据的良好来源。它们通常用于比较一个关键函数功能的不同实现性能。一个好的基准测试会在实际条件下测试性能。相反，如果基准测试使用与实际应用不同的合成输入，那么它很可能会误导你，并导致你得出错误的结论。此外，当基准测试运行在一个没有其他高负载进程的系统上时，它可以利用所有资源，包括DRAM和高速缓存空间。这样的基准测试很可能会优先选择速度更快的函数功能实现版本，即使它消耗的内存比另一个版本更多。然而，如果存在占用大量DRAM的相邻进程，导致属于基准测试进程的内存区域被交换到磁盘，那么结果可能会相反。

For the same reason, be careful when concluding results obtained from unit-testing a function. Modern unit-testing frameworks, e.g., GoogleTest, provide the duration of each test. However, this information cannot substitute a carefully written benchmark that tests the function in practical conditions using realistic input (see more in [@fogOptimizeCpp, chapter 16.2]). It is not always possible to replicate the exact input and environment as it will be in practice, but it is something developers should take into account when writing a good benchmark.

出于同样的原因，在解读单元测试（unit-testing）的结果时要格外谨慎。现代单元测试框架，例如GoogleTest，会提供每次测试的持续时间。然而，这些信息无法替代精心编写的基准测试，该测试需要在实际条件下使用真实输入来检验函数功能（详见[@fogOptimizeCpp，第 16.2 章]）。虽然并非总是能够完全复现实际应用中的输入和环境，但开发者在编写优秀的基准测试时应该考虑到这一点。

[^3]: Google benchmark library 谷歌基准测试库 - [https://github.com/google/benchmark](https://github.com/google/benchmark)
[^4]: BenchmarkDotNet .Net基准测试 - [https://github.com/dotnet/BenchmarkDotNet](https://github.com/dotnet/BenchmarkDotNet)
[^5]: Julia BenchmarkTools Julia基准测试- [https://github.com/JuliaCI/BenchmarkTools.jl](https://github.com/JuliaCI/BenchmarkTools.jl)
[^6]: Java Microbenchmark Harness - [http://openjdk.java.net/projects/code-tools/jmh/etc](http://openjdk.java.net/projects/code-tools/jmh/etc)
[^7]: For JMH, this is known as the `Blackhole.consume()`.
[^8]: Criterion.rs - [https://github.com/bheisler/criterion.rs](https://github.com/bheisler/criterion.rs)