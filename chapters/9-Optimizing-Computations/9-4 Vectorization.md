## Vectorization 向量化 {#sec:Vectorization}

On modern processors, the use of SIMD instructions can result in a great speedup over regular un-vectorized (scalar) code. When doing performance analysis, one of the top priorities of the software engineer is to ensure that the hot parts of the code are vectorized. This section guides engineers toward discovering vectorization opportunities. For a recap of the SIMD capabilities of modern CPUs, readers can take a look at [@sec:SIMD].

在现代处理器上，使用SIMD指令可以显著提升代码速度，远超普通的非向量化（标量scalar）代码。在进行性能分析时，软件工程师的首要任务之一是确保代码中的热点部分实现向量化。本节将指导工程师发现向量化的机会。如需回顾现代CPU的SIMD功能，读者可以参考 [@sec:SIMD]。

Vectorization often happens automatically without any user intervention; this is called compiler *autovectorization*. In such a situation, a compiler automatically recognizes the opportunity to produce SIMD machine code from the source code. 

向量化通常无需用户干预即可自动完成；这被称为编译器*自动向量化autovectorization*。在这种情况下，编译器会自动识别源代码中生成SIMD机器代码的机会。

Autovectorization is very convenient because modern compilers can automatically generate fast SIMD code for a wide variety of programs. However, in some cases, autovectorization does not succeed without intervention by a software engineer. Modern compilers have extensions that allow power users to control the autovectorization process and make sure that certain parts of the code are vectorized efficiently. We will provide several examples of using compiler autovectorization hints.

自动向量化非常便捷，因为现代编译器可以自动为各种程序生成快速的SIMD代码。然而，在某些情况下，如果没有软件工程师的干预，自动向量化可能无法成功。现代编译器提供了一些扩展，允许高级用户控制自动向量化过程，并确保代码的特定部分能够高效地进行向量化。我们将提供几个使用编译器自动向量化提示的示例。

In this section, we will discuss how to harness compiler autovectorization, especially inner loop vectorization because it is the most common type of autovectorization. The other two types (outer loop vectorization, and Superword-Level Parallelism vectorization) are not discussed in this book.

在本节中，我们将讨论如何利用编译器自动向量化，特别是内层循环向量化，因为它是最常见的自动向量化类型。本书不讨论其他两种类型（外层循环向量化和超字级并行向量化）。

### Compiler Autovectorization 编译器自动向量化

Multiple hurdles can prevent autovectorization, some of which are inherent to the semantics of programming languages. For example, the compiler must assume that loop indices may overflow, and this can prevent certain loop transformations. Another example is the assumption that the C programming language makes: pointers in the program may point to overlapping memory regions, which can make the analysis of the program very difficult. 

自动向量化会受到多种因素的阻碍，其中一些是编程语言语义固有的。例如，编译器必须假设循环索引可能会溢出，这会阻止某些循环转换。另一个例子是C语言的假设：程序中的指针可能指向重叠的内存区域，这会使程序分析变得非常困难。

Another major hurdle is the design of the processor itself. In some cases, processors don’t have efficient vector instructions for certain operations. For example, predicated (bitmask-controlled) load and store operations are not available on most processors. Despite all of the challenges, you can work around many of them and enable autovectorization. Later in this section, we provide guidance on how to work with the compiler and ensure that the hot code is vectorized by the compiler.

另一个主要障碍是处理器本身的设计。在某些情况下，处理器没有针对某些操作的高效向量指令。例如，大多数处理器不支持谓词控制（位掩码控制）的加载和存储操作。尽管面临诸多挑战，但你可以克服其中许多挑战并启用自动向量化。本节稍后将提供如何与编译器协作并确保热代码由编译器进行向量化的指导。

The vectorizer is usually structured in three phases: legality-check, profitability-check, and transformation itself:

向量化器通常分为三个阶段：合法性检查、收益检查和转换本身：

* **Legality-check**: in this phase, the compiler checks if it is legal to transform the loop (or another type of code region) into using vectors. The legality phase collects a list of requirements that need to happen for the vectorization of the loop to be legal. The loop vectorizer checks that the iterations of the loop are consecutive, which means that the loop progresses linearly. The vectorizer also ensures that all of the memory and arithmetic operations in the loop can be widened into consecutive operations. That the control flow of the loop is uniform across all lanes and that the memory access patterns are uniform. The compiler has to check or ensure that the generated code won’t touch memory that it is not supposed to and that the order of operations will be preserved. The compiler needs to analyze the possible range of pointers, and if it has some missing information, it has to assume that the transformation is illegal.

* **合法性检查**：在此阶段，编译器会检查将循环（或其他类型的代码区域）转换为使用向量是否合法。合法性阶段会收集一系列要求，以确保将循环向量化是合法的。循环向量化器会检查循环的迭代是否连续，这意味着循环是线性进行的。向量化器还会确保循环中的所有存储和算术运算都可以扩展为连续的操作，循环的控制流在所有通道上保持一致，并且存储访问模式也保持一致。编译器必须检查或确保生成的代码不会访问不应访问的内存，并且操作顺序能够得到保留。编译器需要分析指针的可能范围，如果缺少某些信息，则必须假定转换是非法的。

* **Profitability-check**: next, the vectorizer checks if a transformation is profitable. It compares different vectorization widths and figures out which one would be the fastest to execute. The vectorizer uses a cost model to predict the cost of different operations, such as scalar add or vector load. It needs to take into account the added instructions that shuffle data into registers, predict register pressure, and estimate the cost of the loop guards that ensure that preconditions that allow vectorizations are met. The algorithm for checking profitability is simple: 1) add up the cost of all of the operations in the code, 2) compare the costs of each version of the code, and 3) divide the cost by the expected execution count. For example, if the scalar code costs 8 cycles, and the vectorized code costs 12 cycles but performs 4 loop iterations at once, then the vectorized version of the loop is probably faster.

* **收益检查**：接下来，向量化器会检查转换是否有收益。它会比较不同的向量化宽度，并计算出执行速度最快的向量化宽度。向量化器使用成本模型来预测不同操作的成本，例如：标量加法或向量加载。它需要考虑将数据混入寄存器的附加指令、预测寄存器压力，并估算循环保护的成本，以确保满足向量化的前提条件。收益检查的算法很简单：1）计算代码中所有操作的成本之和；2）比较每个版本代码的成本；3）将总成本除以预期执行次数。例如，如果标量代码耗时8个周期，而向量化代码耗时12个周期，但一次执行4次循环迭代，那么向量化版本的循环可能更快。

* **Transformation**: finally, after the vectorizer figures out that the transformation is legal and profitable, it transforms the code. This process also includes the insertion of guards that enable vectorization. For example, most loops use an unknown iteration count, so the compiler has to generate a scalar version of the loop (remainder), in addition to the vectorized version of the loop, to handle the last few iterations. The compiler also has to check if pointers don’t overlap, etc. All of these transformations are done using information that is collected during the legality check phase.

* **转换**：最后，在向量化器确定转换合法且有利可图之后，它会转换代码。此过程还包括插入启用向量化的保护机制。例如，大多数循环的迭代次数未知，因此编译器除了生成循环的向量化版本外，还必须生成循环的标量版本（剩余部分remainder），以处理最后几次迭代。编译器还必须检查指针是否重叠等等。所有这些转换都是利用在合法性检查阶段收集的信息完成的。

### Discovering Vectorization Opportunities. 发现向量化机会。 {#sec:DiscoverVectOpptnt}

Discovering opportunities for improving vectorization should start by analyzing hot loops in the program and checking what optimizations were performed by the compiler. Checking compiler vectorization reports (see [@sec:compilerOptReports]) is the easiest way to know that. Modern compilers can report whether a certain loop was vectorized, and provide additional details, e.g., vectorization factor (VF). In the case when the compiler cannot vectorize a loop, it is also able to tell the reason why it failed. 

发现改进向量化的机会应该从分析程序中的热点循环并检查编译器执行了哪些优化开始。查看编译器向量化报告（参见 [@sec:compilerOptReports]）是了解这一点的最简单方法。现代编译器可以报告某个循环是否已向量化，并提供其他详细信息，例如：向量化因子 (VF: Vectorization Factor)。如果编译器无法向量化循环，它也能指出失败的原因。

An alternative way to use compiler optimization reports is to check assembly output. It is best to analyze the output from a profiling tool that shows the correspondence between the source code and generated assembly instructions for a given loop. That way you only focus on the code that matters, i.e., the hot code. However, understanding assembly language is much more difficult than a high-level language like C++. It may take some time to figure out the semantics of the instructions generated by the compiler. However, this skill is highly rewarding and often provides valuable insights. 

另一种利用编译器优化报告的方法是检查汇编输出。最好使用轮廓性能分析工具来分析输出结果，该工具可以显示给定循环的源代码和生成的汇编指令之间的对应关系。这样，您就可以只关注关键代码，即热点代码。然而，理解汇编语言比理解C++等高级语言要困难得多。理解编译器生成的指令语义可能需要一些时间。但是，这项技能非常有价值，并且通常能提供宝贵的见解。

Experienced developers can quickly tell whether the code was vectorized or not just by looking at instruction mnemonics and the register names used by those instructions. For example, in x86 ISA, vector instructions operate on packed data (thus have `P` in their name) and use `XMM`, `YMM`, or `ZMM` registers, e.g., `VMULPS XMM1, XMM2, XMM3` multiplies four single precision floats in `XMM2` and `XMM3` and saves the result in `XMM1`. But be careful, often people conclude from seeing the `XMM` register being used, that it is vector code---not necessarily. For instance, the `VMULSS XMM1, XMM2, XMM3` instruction will only multiply one single-precision floating-point value, not four.

经验丰富的开发人员只需查看指令助记符和这些指令使用的寄存器名称，就能快速判断代码是否已向量化。例如，在 x86指令集体系结构(ISA)中，向量指令操作的是打包数据（因此指令名称中带有 `P`），并使用 `XMM`、`YMM` 或 `ZMM` 寄存器。例如，`VMULPS XMM1, XMM2, XMM3` 指令将 `XMM2` 和 `XMM3` 中的4个单精度浮点数相乘，并将结果保存到 `XMM1` 中。但要注意，人们常常会因为看到使用了 `XMM` 寄存器就断定这是向量代码——但这并非必然。例如，`VMULSS XMM1, XMM2, XMM3` 指令只会将1个单精度浮点数相乘，而不是4个。

Another indicator for potential vectorization opportunities is a high `Retiring` metric (above 80%). In [@sec:TMA], we said that the `Retiring` metric is a good indicator of well-performing code. The rationale behind it is that execution is not stalled and a CPU is retiring instructions at a high rate. However, sometimes it may hide the real performance problem, that is, inefficient computations. Perhaps a workload executes a lot of simple instructions that can be replaced by vector instructions. In such situations, high `Retiring` metric doesn't translate into high performance.

另一个判断代码是否适合向量化的重要指标是较高的 `完成Retiring` 指标（高于80%）。在 [@sec:TMA] 中，我们提到 `Retiring` 指标是衡量代码性能的一个良好指标。其背后的原理是执行不会停滞，CPU可以高速地完成指令。然而，有时这可能会掩盖真正的性能问题，即低效的计算。例如，工作负载可能执行了大量可以用向量指令替代的简单指令。在这种情况下，较高的 `完成Retiring` 指标并不意味着高性能。

There are a few common cases that developers frequently run into when trying to accelerate vectorizable code. Below we present four typical scenarios and give general guidance on how to proceed in each case.

开发人员在尝试加速可向量化代码时经常会遇到一些常见情况。下面我们将介绍四个典型场景，并针对每种情况提供一般性指导。

#### Vectorization Is Illegal. 向量化是非法的。

In some cases, the code that iterates over elements of an array is simply not vectorizable. Optimization reports are very effective at explaining what went wrong and why the compiler can’t vectorize the code. [@lst:VectDep] shows an example of dependence inside a loop that prevents vectorization.[^31]

在某些情况下，遍历数组元素的代码根本无法向量化。优化报告可以非常有效地解释哪里出了问题，以及编译器为什么无法向量化代码。[@lst:VectDep] 展示了一个循环内部依赖关系阻止向量化的示例。[^31]

Listing: Vectorization: read-after-write dependence.
代码列表：向量化：写后堵依赖关系

~~~~ {#lst:VectDep .cpp}
void vectorDependence(int *A, int n) {
  for (int i = 1; i < n; i++)
    A[i] = A[i-1] * 2;
}
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

While some loops cannot be vectorized due to the hard limitations (such as read-after-write dependence), others could be vectorized when certain constraints are relaxed. For example, the code in [@lst:VectIllegal] cannot be autovectorized by the compiler, because it will change the order of floating-point operations and may lead to different rounding and a slightly different result. Floating-point addition is commutative, which means that you can swap the left-hand side and the right-hand side without changing the result: `(a + b == b + a)`. However, it is not associative, because rounding happens at different times: `((a + b) + c) != (a + (b + c))`.

虽然有些循环由于硬性限制（例如：读写后依赖）而无法向量化，但当某些约束条件放宽时，其他循环就可以向量化。例如，[@lst:VectIllegal] 中的代码无法由编译器自动向量化，因为它会改变浮点运算的顺序，并可能导致不同的舍入方式和略微不同的结果。浮点加法满足交换律，这意味着您可以交换等式左右两边的位置而不改变结果：`(a + b == b + a)`。但是，它不满足结合律，因为舍入操作发生在不同的时间：`((a + b) + c) != (a + (b + c))`。

Listing: Vectorization: floating-point arithmetic.
代码列表：向量化：浮点算术

~~~~ {#lst:VectIllegal .cpp .numberLines}
// a.cpp
float calcSum(float* a, unsigned N) {
  float sum = 0.0f;
  for (unsigned i = 0; i < N; i++) {
    sum += a[i];
  }
  return sum;
}
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

If you tell the compiler that you can tolerate a bit of variation in the final result, it  will autovectorize the code for you. Clang and GCC compilers have a flag, `-ffast-math`,[^29] that allows this kind of transformation even though the resulting program may give slightly different results:

如果你告诉编译器你可以容忍最终结果存在一些偏差，它会自动为你向量化代码。Clang和GCC编译器都有一个标志 `-ffast-math`[^29]，允许进行这种转换，即使生成的程序可能会给出略微不同的结果：

```bash
$ clang++ -c a.cpp -O3 -march=core-avx2 -Rpass-analysis=.*
...
a.cpp:5:9: remark: loop not vectorized: cannot prove it is safe to reorder floating-point operations; allow reordering by specifying '#pragma clang loop vectorize(enable)' before the loop or by providing the compiler option '-ffast-math'. [-Rpass-analysis=loop-vectorize]
...
$ clang++ -c a.cpp -O3 -march=core-avx2 -ffast-math -Rpass=.*
...
a.cpp:4:3: remark: vectorized loop (vectorization width: 4, interleaved count: 2) [-Rpass=loop-vectorize]
...
```

Unfortunately, this flag involves subtle and potentially dangerous behavior changes, including for Not-a-Number, signed zero, infinity, and subnormals. Because third-party code may not be ready for these effects, this flag should not be enabled across large sections of code without careful validation of the results, including for edge cases. Since Clang 18, you can limit the scope of transformations by using dedicated pragmas, e.g., `#pragma clang fp reassociate(on)`.[^4]

不幸的是，此标志涉及一些微妙且可能危险的行为变化，包括：对非数字、有符号零、无穷大和次正规数的处理。由于第三方代码可能尚未准备好应对这些影响，因此在未仔细验证结果（包括边界情况）的情况下，不应在代码的大部分区域启用此标志。自Clang 18起，您可以使用专用编译指示来限制转换的范围，例如 `#pragma clang fp reassociate(on)`[^4]。

Let's look at another typical situation when a compiler may need support from a developer to perform vectorization. When compilers cannot prove that a loop operates on arrays with non-overlapping memory regions, they usually choose to be on the safe side. Given the code in [@lst:OverlappingMemRefions], compilers should account for the situation when the memory regions of arrays `a`, `b`, and `c` overlap.

让我们来看另一个编译器可能需要开发人员支持才能执行向量化的典型情况。当编译器无法证明循环操作的数组具有不重叠的内存区域时，它们通常会选择保守的做法。对于 [@lst:OverlappingMemRefions] 中的代码，编译器应考虑数组 `a`、`b` 和 `c` 的内存区域重叠的情况。

Listing: a.c
代码列表：a.c

~~~~ {#lst:OverlappingMemRefions .cpp .numberLines}
void foo(float* a, float* b, float* c, unsigned N) {
  for (unsigned i = 1; i < N; i++) {
    c[i] = b[i];
    a[i] = c[i-1];
  }
}
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Here is the optimization report (enabled with `-fopt-info`) provided by GCC 10.2:
以下是GCC 10.2提供的优化报告（使用 `-fopt-info` 启用）：

```bash
$ gcc -O3 -march=core-avx2 -fopt-info
a.cpp:2:26: optimized: loop vectorized using 32-byte vectors
a.cpp:2:26: optimized:  loop versioned for vectorization because of possible aliasing
```

GCC has recognized potential overlap between memory regions and created multiple versions of the loop. The compiler inserted runtime checks[^36] to detect if the memory regions overlap. Based on those checks, it dispatches between vectorized and scalar versions. In this case, vectorization comes with the cost of inserting potentially expensive runtime checks. If a developer knows that memory regions of arrays `a`, `b`, and `c` do not overlap, it can insert `#pragma GCC ivdep`[^37] right before the loop or use the `__restrict__  ` keyword as shown in [@sec:compilerOptReports]. Such compiler hints will eliminate the need for the GCC compiler to insert the runtime checks mentioned earlier.

GCC已识别出内存区域之间可能存在重叠，并创建了循环的多个版本。编译器插入了运行时检查[^36]来检测内存区域是否重叠。基于这些检查，它会在向量化版本和标量版本之间进行切换。在这种情况下，向量化的代价是插入可能代价高昂的运行时检查。如果开发人员知道数组 `a`、`b` 和 `c` 的内存区域不重叠，则可以在循环之前插入 `#pragma GCC ivdep`[^37] 或使用 `__restrict__` 关键字，如 [@sec:compilerOptReports] 所示。此类编译器提示将消除GCC编译器插入前面提到的运行时检查的必要性。

Some dynamic tools, such as Intel Advisor, can detect if issues like cross-iteration dependence or access to arrays with overlapping memory regions occur in a loop. But be aware that such tools only provide a suggestion. Carelessly inserting compiler hints can cause real problems.

一些动态工具，例如Intel Advisor，可以检测循环中是否存在诸如跨迭代依赖或访问具有重叠内存区域的数组之类的问题。但请注意，此类工具仅提供建议。随意插入编译器提示可能会导致实际问题。

#### Vectorization Is Not Beneficial. 向量化没有收益。

In some cases, the compiler can vectorize the loop but decide that doing so is not profitable. In the code presented in [@lst:VectNotProfit], the compiler could vectorize the memory access to array `A` but would need to split the access to array `B` into multiple scalar loads. The scatter/gather pattern is relatively expensive, and compilers that can simulate the cost of operations often decide to avoid vectorizing code with such patterns. 

在某些情况下，编译器可以向量化循环，但会认为这样做并不划算。在 [@lst:VectNotProfit] 中提供的代码中，编译器可以向量化对数组 `A` 的内存访问，但需要将对数组 `B` 的访问拆分成多个标量加载操作。分散/聚集（scatter/gather）模式的开销相对较大，能够模拟操作成本的编译器通常会避免对包含此类模式的代码进行向量化。

Listing: Vectorization: not beneficial.
代码列表：向量化：没有收益

~~~~ {#lst:VectNotProfit .cpp .numberLines}
// a.cpp
void stridedLoads(int *A, int *B, int n) {
  for (int i = 0; i < n; i++)
    A[i] += B[i * 3];
}
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Here is the compiler optimization report for the code in [@lst:VectNotProfit]: 
以下是 [@lst:VectNotProfit] 中代码的编译器优化报告：

```bash
$ clang -c -O3 -march=core-avx2 a.cpp -Rpass-missed=loop-vectorize
a.cpp:3:3: remark: the cost-model indicates that vectorization is not beneficial [-Rpass-missed=loop-vectorize]
  for (int i = 0; i < n; i++)
  ^
```

Users can force the Clang compiler to vectorize the loop by using the `#pragma` hint, as shown in [@lst:VectNotProfitOverriden]. However, keep in mind that whether vectorization is profitable largely depends on the runtime data, for example, the number of iterations of the loop. Compilers don't have this information available,[^1] so they often tend to be conservative. Though you can use such hints for performance experiments.

用户可以使用 `#pragma` 指令强制Clang编译器对循环进行向量化，如 [@lst:VectNotProfitOverriden] 所示。但是，请记住，向量化是否有利很大程度上取决于运行时数据，例如循环的迭代次数。编译器无法获取这些信息[^1]，因此它们通常倾向于保守处理。不过，您可以使用此类指令进行性能实验。

Listing: Vectorization: not beneficial.
代码列表：向量化：没有收益

~~~~ {#lst:VectNotProfitOverriden .cpp .numberLines}
// a.cpp
void stridedLoads(int *A, int *B, int n) {
#pragma clang loop vectorize(enable)
  for (int i = 0; i < n; i++)
    A[i] += B[i * 3];
}
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Developers should be aware of the hidden cost of using vectorized code. Using AVX and especially AVX-512 vector instructions could lead to frequency downclocking or startup overhead, which on certain CPUs can also affect subsequent code for several microseconds. The vectorized portion of the code should be hot enough to justify using AVX-512.[^38] For example, sorting 80 KiB was found to be sufficient to amortize this overhead and make vectorization worthwhile.[^39]

开发者应注意使用向量化代码的隐性成本。使用AVX指令集，尤其是AVX-512向量指令集，可能会导致频率降低或启动开销，在某些CPU上，这甚至会影响后续代码数微秒的执行。只有当代码的向量化部分足够“热”时，使用AVX-512指令集才算合理[^38]。例如，排序80KiB的数据足以抵消这种开销，从而使向量化变得值得[^39]。

#### Loop Vectorized but Scalar Version Used. 循环向量化但使用了标量版本

In some scenarios, the compiler successfully vectorizes the code, but it does not show up as being executed in the profiler. When inspecting the corresponding assembly of a loop, it is usually easy to find the vectorized version of the loop body because it uses vector registers, which are not commonly used in other parts of the program.

在某些情况下，编译器成功向量化了代码，但分析器中并未显示该代码正在执行。检查循环的汇编代码时，通常很容易找到循环体的向量化版本，因为它使用了向量寄存器，而这些寄存器在程序的其他部分并不常用。

If the vector code is not executed, one possible reason for this is that the generated code assumes loop trip counts that are higher than what the program uses. For example, a compiler may decide to vectorize and unroll the loop in such a way as to process 64 elements per iteration. An input array may not have enough elements even for a single iteration of the loop. In this case, the scalar version (remainder) of the loop will be used instead. It is easy to detect these cases because the scalar loop would light up in the profiler, and the vectorized code would remain cold. 

如果向量化代码没有执行，一个可能的原因是生成的代码假定的循环次数高于程序实际使用的次数。例如，编译器可能会决定将循环向量化并展开，以便每次迭代处理64个元素。但输入数组的元素数量可能不足以完成一次循环迭代。在这种情况下，编译器会使用循环的标量版本（余数部分remainder）。这种情况很容易检测，因为标量循环会在性能分析器中显示出来，而向量化代码则不会显示任何信息。

The solution to this problem is to force the vectorizer to use a lower vectorization factor or unroll count, to reduce the number of elements that the loop processes. You can achieve that with the help of `#pragma` hints. For the Clang compiler, you can use `#pragma clang loop vectorize_width(N)` as shown in the article on the Easyperf blog.[^30]

解决此问题的方法是强制向量化器使用更低的向量化因子或展开次数，以减少循环处理的元素数量。您可以使用 `#pragma` 指令来实现这一点。对于Clang编译器，您可以使用 `#pragma clang loop vectorize_width(N)`，如Easyperf博客文章中所示[^30]。

#### Loop Vectorized in a Suboptimal Way. 循环向量化以一种次优的方式进行

When you see a loop being autovectorized and executed at runtime, there is a high chance that this part of the program already performs well. However, there are exceptions. There are situations when the scalar un-vectorized version of a loop performs better than the vectorized one. This could happen due to expensive vector operations like `gather/scatter` loads, masking, `inserting/extracting` elements, data shuffling, etc., if the compiler is required to use them to make vectorization happen. Performance engineers could also try to disable vectorization in different ways. For the Clang compiler, it can be done via compiler options `-fno-vectorize` and `-fno-slp-vectorize`, or with a hint specific to a particular loop, e.g., `#pragma clang loop vectorize(disable)`.

当您看到循环在运行时自动向量化并执行时，很可能程序的这部分已经运行良好。然而，也有例外。在某些情况下，循环的标量非向量化版本性能优于向量化版本。这可能是由于诸如 `gather/scatter` 的加载loader、掩码masking、`插入/抽取insert/extract` 元素、数据混洗（data shuffling）等代价高昂的向量操作造成的，如果编译器需要使用这些操作来实现向量化的话。性能工程师还可以尝试通过不同的方式禁用向量化。对于Clang编译器，可以通过编译器选项 `-fno-vectorize` 和 `-fno-slp-vectorize` 来实现，或者使用针对特定循环的提示，例如 `#pragma clang loop vectorize(disable)`。

It is important to note that there is a range of problems where SIMD is important and where autovectorization just does not work and is not likely to work in the near future. One example can be found in [@Mula_Lemire_2019]. Another example is outer loop autovectorization, which is not currently attempted by compilers. Vectorizing floating-point code is problematic because reordering arithmetic floating-point operations results in different rounding and slightly different values.

值得注意的是，在某些问题中，SIMD至关重要，而自动向量化在这些问题中根本不起作用，而且在不久的将来也不太可能起作用。例如，[@Mula_Lemire_2019] 中就提到了这一点。另一个例子是外层循环的自动向量化，目前编译器不会尝试进行这种操作。浮点代码向量化存在问题，因为重新排列浮点运算顺序会导致不同的舍入方式，从而产生略微不同的值。

There is one more subtle problem with autovectorization. As compilers evolve, optimizations that they make are changing. The successful autovectorization of code that was done in the previous compiler version may stop working in the next version, or vice versa. Also, during code maintenance or refactoring, the structure of the code may change, such that autovectorization suddenly starts failing. This may occur long after the original software was written, so it would be more expensive to fix or redo the implementation at this point.

自动向量化还有一个更微妙的问题。随着编译器的演进，其优化方式也在不断变化。在之前的编译器版本中成功实现的自动向量化，在下一个版本中可能失效，反之亦然。此外，在代码维护或重构过程中，代码结构可能会发生变化，导致自动向量化突然失效。这种情况可能发生在原始软件编写很久之后，因此此时修复或重写实现的成本会更高。

#### Languages with Explicit Vectorization. 支持显式向量化的语言。 {#sec:ISPC}

Vectorization can also be achieved by rewriting parts of a program in a programming language that is dedicated to parallel computing. Those languages use special constructs and knowledge of the program's data to compile the code efficiently into parallel programs. Originally such languages were mainly used to offload work to specific processing units such as graphics processing units (GPU), digital signal processors (DSP), or field-programmable gate arrays (FPGAs). However, some of those programming models can also target your CPU (such as OpenCL and OpenMP).

向量化也可以通过使用专用于并行计算的编程语言重写程序的部分代码来实现。这些语言利用特殊的结构和对程序数据的了解，高效地将代码编译成并行程序。最初，这类语言主要用于将工作卸载到特定的处理单元，例如图形处理器(GPU: Graphics Processing Unit)、数字信号处理器(DSP: Digital Signal Processor)或现场可编程门阵列(FPGA: Field-Programmable Gate Arrays)。然而，其中一些编程模型也可以面向CPU（例如：OpenCL和OpenMP）。

One such parallel language is Intel® Implicit SPMD Program Compiler [(ISPC)](https://ispc.github.io/),[^33] which I will briefly cover in this section. The ISPC language is based on the C programming language and uses the LLVM compiler infrastructure to generate optimized code for many different architectures. The key feature of ISPC is the "close to the metal" programming model and performance portability across SIMD architectures. It requires a shift from the traditional thinking of writing programs but gives programmers more control over CPU resource utilization.

Intel®公司的隐式SPMD程序编译器 [(ISPC)](https://ispc.github.io/)[^33]，就是这样一种并行语言，我将在本节中简要介绍它。ISPC语言基于C语言，并使用LLVM编译器基础设施为多种不同的体系结构生成优化代码。ISPC的关键特性是“接近底层金属（close to the metal）”的编程模型以及跨SIMD体系结构的性能可移植性。它需要程序员转变传统的程序编写思路，但赋予了程序员对CPU资源利用更大的控制权。

Another advantage of ISPC is its interoperability and ease of use. ISPC compiler generates standard object files that can be linked with the code generated by conventional C/C++ compilers. ISPC code can be easily plugged into any native project since functions written with ISPC can be called as if it was C code.

ISPC的另一个优势在于其互操作性和易用性。ISPC编译器生成的标准目标文件可以与传统C/C++编译器生成的代码链接。由于用ISPC编写的函数可以像C代码一样被调用，因此 ISPC代码可以轻松集成到任何原生项目中。

[@lst:ISPC_code] shows an ISPC version of a function that I presented earlier in [@lst:VectIllegal]. ISPC considers that the program will run in parallel instances, based on the target instruction set. For example, when using SSE with `float`s, it can compute 4 operations in parallel. Each program instance would operate on vector values of `i` being `(0,1,2,3)`, then `(4,5,6,7)`, and so on, effectively computing 4 sums at a time. As you can see, a few keywords not typical for C and C++ are used:

[@lst:ISPC_code] 展示了我之前在 [@lst:VectIllegal] 中介绍的一个函数的ISPC版本。ISPC考虑程序将基于目标指令集以并行实例的形式运行。例如，当使用SSE指令集处理 `float` 类型时，它可以并行计算4个操作。每个程序实例将分别处理向量 `i` 的值，例如 `(0,1,2,3)`、`(4,5,6,7)`，依此类推，从而有效地同时计算4个求和。如您所见，这里使用了一些C和C++中不常见的关键字：

* The `export` keyword means that the function can be called from a C-compatible language.
* `export` 关键字表示该函数可以从C兼容的语言中调用。

* The `uniform` keyword means that a variable is shared between program instances.
* `uniform` 关键字表示变量在程序实例之间共享。

* The `varying` keyword means that each program instance has its own local copy of the variable.
* `varying` 关键字表示每个程序实例都有其自身的变量副本。

* The `foreach` is the same as a classic `for` loop except that it will distribute the work across the different program instances. 
* `foreach` 与经典的 `for` 循环类似，区别在于它会将工作分配到不同的程序实例上。

Listing: ISPC version of summing elements of an array.
代码列表：ISPC版本的一个数组元素求和

~~~~ {#lst:ISPC_code .cpp}
export uniform float calcSum(const uniform float array[], 
                             uniform ptrdiff_t count)
{
    varying float sum = 0;
    foreach (i = 0 ... count)
        sum += array[i];
    return reduce_add(sum);
}
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Since the function `calcSum` must return a single value (a `uniform` variable) and our `sum` variable is `varying`, we then need to *gather* the values of each program instance using the `reduce_add` function. ISPC also takes care of generating peeled and remainder loops as needed to take into account the data that is not correctly aligned or that is not a multiple of the vector width. 

由于函数 `calcSum` 必须返回单个值（一个 `uniform` 变量），而我们的 `sum` 变量是 `varying` 的，因此我们需要使用 `reduce_add` 函数来*收集gather*每个程序实例的值。ISPC还会根据需要生成剥离循环和剩余循环，以处理未正确对齐或不是向量宽度倍数的数据。

**"Close to the metal" programming model**: one of the problems with traditional C and C++ languages is that the compiler doesn't always vectorize critical parts of code. ISPC helps to resolve this problem by assuming every operation is SIMD by default. For example, the ISPC statement `sum += array[i]` is implicitly considered as a SIMD operation that makes multiple additions in parallel. ISPC is not an autovectorizing compiler, and it does not automatically discover vectorization opportunities. Since the ISPC language is very similar to C and C++, it is more readable than intrinsics (see [@sec:secIntrinsics]) as it allows you to focus on the algorithm rather than the low-level instructions. Also, it has reportedly matched [@ISPC_Paper] or beaten[^34] hand-written intrinsics code in terms of performance.

**“接近底层金属”的编程模型**：传统C和C++语言的一个问题是编译器并非总是对代码的关键部分进行向量化。ISPC通过默认假设每个操作都是SIMD操作来帮助解决这个问题。例如，ISPC语句 `sum += array[i]` 被隐式地视为一个并行执行多个加法的SIMD操作。ISPC不是一个自动向量化的编译器，它不会自动发现向量化的机会。由于ISPC语言与C和C+ 非常相似，因此它比内部函数（参见 [@sec:secIntrinsics]）更易读，因为它允许您专注于算法而不是底层指令。此外，据报道，在性能方面，它已经达到 [@ISPC_Paper] 甚至超过了 [^34] 手写的内部函数代码。

**Performance portability**: ISPC can automatically detect features of your CPU to fully utilize all the resources available. Programmers can write ISPC code once and compile to many vector instruction sets, such as SSE4, AVX2, and ARM NEON.

**性能可移植性**：ISPC可以自动检测CPU的特性，从而充分利用所有可用资源。程序员只需编写一次ISPC代码，即可将其编译为多种向量指令集，例如：SSE4、AVX2和ARM NEON。

[^1]: Besides Profile Guided Optimizations 除了轮廓信息指导优化之外 (see [@sec:secPGO]).
[^2]: For example, compiler optimization reports, see 例如，编译器优化报告，参见 [@sec:compilerOptReports].
[^29]: The compiler flag `-Ofast` enables `-ffast-math` as well as the `-O3` compilation mode. 编译器标志 `-Ofast` 启用 `-ffast-math` 以及 `-O3` 编译模式。
[^30]: Using Clang's optimization pragmas  使用Clang的优化编译指示 - [https://easyperf.net/blog/2017/11/09/Multiversioning_by_trip_counts](https://easyperf.net/blog/2017/11/09/Multiversioning_by_trip_counts)
[^31]: It is easy to spot a read-after-write dependency once you unroll a couple of iterations of the loop. See the example in 展开循环几次后，很容易发现写后读依赖关系。参见 [@sec:compilerOptReports].
[^33]: ISPC compiler: ISPC编译器： [https://ispc.github.io/](https://ispc.github.io/).
[^34]: Some parts of the Unreal Engine that used SIMD intrinsics were rewritten using ISPC, which gave speedups: 虚幻引擎（Unreal Engine）中一些使用 SIMD 内联函数的部分代码已使用 ISPC 重写，从而提高了速度：[https://software.intel.com/content/www/us/en/develop/articles/unreal-engines-new-chaos-physics-system-screams-with-in-depth-intel-cpu-optimizations.html](https://software.intel.com/content/www/us/en/develop/articles/unreal-engines-new-chaos-physics-system-screams-with-in-depth-intel-cpu-optimizations.html).
[^36]: See the example on the Easyperf blog: 请参阅Easyperf博客上的示例： [https://easyperf.net/blog/2017/11/03/Multiversioning_by_DD](https://easyperf.net/blog/2017/11/03/Multiversioning_by_DD).
[^37]: It is a GCC-specific pragma. For other compilers, check the corresponding manuals. 这是一个GCC特有的编译指示。对于其他编译器，请查阅相应的手册。
[^38]: For more details read this blog post: 更多详情请阅读这篇博文： [https://travisdowns.github.io/blog/2020/01/17/avxfreq1.html](https://travisdowns.github.io/blog/2020/01/17/avxfreq1.html).
[^39]: Study of AVX-512 downclocking: in AVX-512 降频研究：参见 [VQSort readme](https://github.com/google/highway/blob/master/hwy/contrib/sort/README.md#study-of-avx-512-downclocking)
[^4]: LLVM extensions to specify floating-point flags 用于指定浮点标志的LLVM扩展 - [https://clang.llvm.org/docs/LanguageExtensions.html#extensions-to-specify-floating-point-flags](https://clang.llvm.org/docs/LanguageExtensions.html#extensions-to-specify-floating-point-flags)
