## Compiler Optimization Reports 编译器优化报告 {#sec:compilerOptReports}

Nowadays, software development relies very much on compilers to do performance optimizations. Compilers play a critical role in speeding up software. The majority of developers leave the job of optimizing code to compilers, interfering only when they see an opportunity to improve something compilers cannot accomplish. Fair to say, this is a good default strategy. But it doesn't work well when you're looking for the best performance possible. What if the compiler failed to perform a critical optimization like vectorizing a loop? How you would know about this? Luckily, all major compilers provide optimization reports which we will discuss now.

如今，软件开发非常依赖编译器进行性能优化。编译器在提升软件速度方面发挥着至关重要的作用。大多数开发者将代码优化工作交给编译器，只有在发现编译器无法完成的优化机会时才会进行干预。可以说，这是一种不错的默认策略。但如果您追求的是最佳性能，这种策略就不太奏效了。如果编译器未能执行诸如循环向量化之类的关键优化怎么办？您如何才能知道这一点？幸运的是，所有主流编译器都提供优化报告，我们将在下文中进行讨论。

Suppose you want to know if a critical loop was unrolled or not. If it was unrolled, what is the unroll factor? There is a hard way to know this: by studying generated assembly instructions. Unfortunately, not all people are comfortable reading assembly language. This can be especially difficult if the function is big, it calls other functions or has many loops that were also vectorized, or if the compiler created multiple versions of the same loop. Most compilers, including GCC, Clang, Intel compiler, and MSVC[^9] provide optimization reports to check what optimizations were done for a particular piece of code.

假设您想知道一个关键循环是否被展开。如果被展开，展开因子是多少？有一种比较繁琐的方法可以知道这一点：研究生成的汇编指令。遗憾的是，并非所有人都擅长阅读汇编语言。如果函数很大、调用了其他函数、包含许多也已向量化的循环，或者编译器创建了同一个循环的多个版本，那么这种情况就尤其棘手。大多数编译器，包括：GCC、Clang、Intel编译器和微软编译器MSVC[^9]，都会提供优化报告，用于检查特定代码段进行了哪些优化。

Let's take a look at [@lst:optReport], which shows an example of a loop that is not vectorized by `clang 16.0`.

我们来看一下 [@lst:optReport]，它展示了一个未被 `clang 16.0` 向量化的循环示例。

Listing: a.c
代码列表：a.c文件

~~~~ {#lst:optReport .cpp .numberLines}
void foo(float* __restrict__ a, 
         float* __restrict__ b, 
         float* __restrict__ c,
         unsigned N) {
  for (unsigned i = 1; i < N; i++) {
    a[i] = c[i-1]; // value is carried over from previous iteration
    c[i] = b[i];
  }
}
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

To emit an optimization report in the Clang compiler, you need to use [-Rpass*](https://llvm.org/docs/Vectorizers.html#diagnostics) flags:

要在Clang编译器中生成优化报告，您需要使用命令行的[-Rpass*](https://llvm.org/docs/Vectorizers.html#diagnostics)参数：

```bash
$ clang -O3 -Rpass-analysis=.* -Rpass=.* -Rpass-missed=.* a.c -c
a.c:5:3: remark: loop not vectorized [-Rpass-missed=loop-vectorize]
  for (unsigned i = 1; i < N; i++) {
  ^
a.c:5:3: remark: unrolled loop by a factor of 8 with run-time trip count [-Rpass=loop-unroll]
  for (unsigned i = 1; i < N; i++) {
  ^
```

By checking the optimization report above, we could see that the loop was not vectorized, but it was unrolled instead. It's not always easy for a developer to recognize the existence of a loop-carry dependency in the loop on line 6 in [@lst:optReport]. The value that is loaded by `c[i-1]` depends on the store from the previous iteration (see operations \circled{2} and \circled{3} in Figure @fig:VectorDep). The dependency can be revealed by manually unrolling the first few iterations of the loop:

通过查看上面的优化报告，我们可以看到循环没有被向量化，而是被展开了。对于开发者来说，识别第6行循环中存在的循环进位依赖关系并不总是那么容易（参见图 @fig:VectorDep 中的操作 \circled{2} 和 \circled{3}）。可以通过手动展开循环的前几次迭代来发现这种依赖关系：

```cpp
 
// iteration 1
  a[1] = c[0];
  c[1] = b[1]; // writing the value to c[1]
// iteration 2
  a[2] = c[1]; // reading the value of c[1]
  c[2] = b[2];
...
```

![Visualizing the order of operations in [@lst:optReport]. 对[@lst:optReport]中的运算顺序的可视化。](../../img/perf-analysis/VectorDep.png){#fig:VectorDep width=40%}

If we were to vectorize the code in [@lst:optReport], it would result in the wrong values written in the array `a`. Assuming a CPU SIMD unit can process four floats at a time, we would get the code that can be expressed with the following pseudocode:

如果我们对 [@lst:optReport] 中的代码进行向量化，则数组 `a` 中写入的值会出错。假设CPU的SIMD单元一次可以处理4个浮点数，那么我们将得到以下伪代码：

```cpp
// iteration 1
  a[1..4] = c[0..3]; // oops!, a[2..4] get wrong values
  c[1..4] = b[1..4]; 
...
```

The code in [@lst:optReport] cannot be vectorized because the order of operations inside the loop matters. This example can be fixed by swapping lines 6 and 7 as shown in [@lst:optReport2]. This does not change the semantics of the code, so it's a perfectly legal change. Alternatively, the code can be improved by splitting the loop into two separate loops. Doing this would double the loop overhead, but this drawback would be outweighed by the performance improvement gained through vectorization.

[@lst:optReport] 中的代码无法向量化，因为循环内部的操作顺序很重要。可以通过交换第6行和第7行来修复此示例，如 [@lst:optReport2] 所示。这不会改变代码的语义，因此是完全合法的更改。或者，可以通过将循环拆分为两个独立的循环来改进代码。这样做会使循环开销翻倍，但向量化带来的性能提升足以弥补这一缺点。

Listing: a.c
代码列表：a.c文件

~~~~ {#lst:optReport2 .cpp .numberLines}
void foo(float* __restrict__ a, 
         float* __restrict__ b, 
         float* __restrict__ c,
         unsigned N) {
  for (unsigned i = 1; i < N; i++) {
    c[i] = b[i];
    a[i] = c[i-1];
  }
}
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

In the optimization report, we can now see that the loop was vectorized successfully:

在优化报告中，我们现在可以看到循环已成功向量化：

```bash
$ clang -O3 -Rpass-analysis=.* -Rpass=.* -Rpass-missed=.* a.c -c
a.cpp:5:3: remark: vectorized loop (vectorization width: 8, interleaved count: 4) [-Rpass=loop-vectorize]
  for (unsigned i = 1; i < N; i++) {
  ^
```

This was just one example of using optimization reports; we will provide more examples in [@sec:DiscoverVectOpptnt], where we discuss how to discover vectorization opportunities. Compiler optimization reports can help you find missed optimization opportunities, and understand why those opportunities were missed. In addition, compiler optimization reports are useful for testing a hypothesis. Compilers often decide whether a certain transformation will be beneficial based on their cost model analysis. But compilers don't always make the optimal choice. Once you detect a key missing optimization in the report, you can attempt to rectify it by changing the source code or by providing hints to the compiler in the form of a `#pragma`, an attribute, a compiler built-in, etc. As always, verify your hypothesis by measuring it in a practical environment.

这只是使用优化报告的一个例子；我们将在[@sec:DiscoverVectOpptnt]中提供更多示例，讨论如何发现向量化机会。编译器优化报告可以帮助您找到错过的优化机会，并了解错过这些机会的原因。此外，编译器优化报告还有助于验证假设。编译器通常会根据其成本模型分析来决定某个转换是否有益。但编译器并非总是做出最优选择。一旦您在报告中发现关键的缺失优化，您可以尝试通过修改源代码或以`#pragma`、属性attribute、编译器内置函数等形式向编译器提供提示来纠正它。与往常一样，请在实际环境中进行测量以验证您的假设。

Compiler reports can be quite large, and a separate report is generated for each source-code file. Sometimes, finding relevant records in the output file can become a challenge. We should mention that initially these reports were explicitly designed to be used by compiler writers to improve optimization passes. Over the years, several tools have been developed to make optimization reports more accessible and actionable by application developers, most notably [opt-viewer](https://github.com/llvm/llvm-project/tree/main/llvm/tools/opt-viewer)[^7] and [optview2](https://github.com/OfekShilon/optview2).[^8] Also, the [Compiler Explorer](https://godbolt.org/) website has the "Optimization Output" tool for LLVM-based compilers that reports performed transformations when you hover your mouse over the corresponding line of source code. All of these tools help visualize successful and failed code transformations by LLVM-based compilers.

编译器报告可能非常庞大，并且每个源代码文件都会生成一个单独的报告。有时，在输出文件中查找相关记录可能颇具挑战性。需要说明的是，这些报告最初是专门为编译器编写者设计的，用于改进优化过程。多年来，人们开发了多种工具，旨在让应用程序开发人员更方便地访问和使用优化报告，其中最著名的是 [opt-viewer](https://github.com/llvm/llvm-project/tree/main/llvm/tools/opt-viewer)[^7]和[optview2](https://github.com/OfekShilon/optview2)[^8]。此外，[Compiler Explorer](https://godbolt.org/) 网站也提供了一个针对基于LLVM的编译器的“优化输出”工具，当您将鼠标悬停在相应的源代码行上时，该工具会报告已执行的转换。所有这些工具都有助于对基于LLVM的编译器执行的成功和失败的代码转换进行可视化。

In the Link-Time Optimization (LTO)[^5] mode, some optimizations are made during the linking stage. To emit compiler reports from both the compilation and linking stages, you should pass dedicated options to both the compiler and the linker. See the LLVM "Remarks" [guide](https://llvm.org/docs/Remarks.html)[^6] for more information. 

在链接时优化(LTO: Link-Time Opttimization)[^5]模式下，一些优化是在链接阶段进行的。要同时生成编译和链接阶段的编译器报告，您需要向编译器和链接器传递相应的选项。更多信息请参阅LLVM的“备注Remarks”[指南](https://llvm.org/docs/Remarks.html)[^6]。

A slightly different way of reporting missing optimizations is taken by the Intel® [ISPC](https://ispc.github.io/ispc.html)[^3] compiler (discussed in [@sec:ISPC]). It issues warnings for code constructs that compile to relatively inefficient code. Either way, compiler optimization reports should be one of the key tools in your toolbox. They are a fast way to check what optimizations were done for a particular hotspot and see if some important ones failed. I have found many improvement opportunities thanks to compiler optimization reports.

Intel® [ISPC](https://ispc.github.io/ispc.html)[^3]编译器（在 [@sec:ISPC] 中讨论过）采用了一种略有不同的方式来报告缺失的优化。它会针对编译成效率相对较低的代码结构发出警告。无论采用哪种方式，编译器优化报告都应该是您工具箱中的关键工具之一。它们可以快速检查针对特定热点进行了哪些优化，以及是否存在一些重要的优化失败。我通过编译器优化报告发现了许多改进机会。

[^1]: Using compiler optimization pragmas 使用编译器优化编译指示 - [https://easyperf.net/blog/2017/11/09/Multiversioning_by_trip_counts](https://easyperf.net/blog/2017/11/09/Multiversioning_by_trip_counts)
[^3]: ISPC - [https://ispc.github.io/ispc.html](https://ispc.github.io/ispc.html)
[^5]: Link-Time Optimizations, also called InterProcedural Optimizations (IPO). Read more here: 链接时优化，也称为过程间优化(IPO: InterProcedural Optimization)。更多信息请参阅：[https://en.wikipedia.org/wiki/Interprocedural_optimization](https://en.wikipedia.org/wiki/Interprocedural_optimization)
[^6]: LLVM compiler remarks LLVM编译器备注 - [https://llvm.org/docs/Remarks.html](https://llvm.org/docs/Remarks.html)
[^7]: opt-viewer - [https://github.com/llvm/llvm-project/tree/main/llvm/tools/opt-viewer](https://github.com/llvm/llvm-project/tree/main/llvm/tools/opt-viewer)
[^8]: optview2 - [https://github.com/OfekShilon/optview2](https://github.com/OfekShilon/optview2)
[^9]: At the time of writing (2024), MSVC provides only vectorization reports. 截至撰写本文时（2024年），MSVC仅提供向量化报告。
