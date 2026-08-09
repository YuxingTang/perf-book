## Loop Optimizations 循环优化 {#sec:LoopOpts}

Loops are at the heart of nearly all high-performance programs. Since loops represent a piece of code that is executed a large number of times, they account for the majority of the execution time. Small changes in such a critical piece of code may have a large impact on the performance of a program. That's why it is so important to carefully analyze the performance of hot loops in a program and know possible ways to improve them.

循环是几乎所有高性能程序的核心。由于循环代表一段会被执行多次的代码，因此它们占据了大部分的执行时间。对如此关键的代码进行微小的改动都可能对程序的性能产生巨大的影响。这就是为什么仔细分析程序中热点循环的性能并了解可能的改进方法如此重要的原因。

In this section, we will take a look at the most well-known loop optimizations that address the types of bottlenecks mentioned above. We first discuss low-level optimizations, in [@sec:LowLevelLoopOpts], that only move code around in a single loop. Next, in [@sec:LoopOptsHighLevel], we will take a look at high-level optimizations that restructure loops, which often affect multiple loops. Note, that what I present here is not a complete list of all known loop transformations. For more detailed information on each of the transformations discussed below, readers can refer to [@EngineeringACompilerBook] and [@OptimizingCompilersForModernArchs].

在本节中，我们将探讨一些最常用的循环优化方法，这些方法可以解决上述各种性能瓶颈。首先，我们将讨论底层优化（在 [@sec:LowLevelLoopOpts] 中），这些优化方法仅对单个循环中的代码进行位置调整。接下来，在 [@sec:LoopOptsHighLevel] 中，我们将探讨高层优化，这些优化方法会重构循环，通常会影响多个循环。请注意，这里列出的并非所有已知的循环转换方法。有关下文讨论的每种转换的更详细信息，读者可以参考 [@EngineeringACompilerBook] 和 [@OptimizingCompilersForModernArchs]。

### Low-level Optimizations. 底层优化。 {#sec:LowLevelLoopOpts}

Let's start with simple loop optimizations that transform the code inside a single loop: Loop Invariant Code Motion, Loop Unrolling, Loop Strength Reduction, and Loop Unswitching. Generally, compilers are good at doing such transformations; however, there are still cases when a compiler might need a developer's support. We will talk about that in subsequent sections.

我们先从简单的循环优化开始，这些优化会转换单个循环内部的代码：循环不变代码移动、循环展开、循环强度降低和循环判断外提。通常，编译器能够很好地完成这些转换；但是，在某些情况下，编译器仍然需要开发人员的支持。我们将在后续章节中讨论这一点。

**Loop Invariant Code Motion (LICM)**: an expression whose value never changes across iterations of a loop is called a *loop invariant*. Since its value doesn't change across loop iterations, we can move a loop invariant expression outside of the loop. We do so by storing the result in a temporary variable and using it inside the loop (see [@lst:LICM]).[^17] All decent compilers nowadays successfully perform LICM in the majority of cases.

**循环不变代码移动 (LICM: Loop Invariant Code Motion)**：在循环迭代中值始终不变的表达式称为*循环不变项（loop invariant）*。由于其值在循环迭代中保持不变，我们可以将循环不变表达式移到循环外部。我们通过将结果存储在一个临时变量中并在循环内部使用它来实现这一点（参见 [@lst:LICM]）[^17]。如今，大多数优秀的编译器都能成功地执行LICM（循环不变代码移动）。

Listing: Loop Invariant Code Motion
代码列表：循环不变代码移动

~~~~ {#lst:LICM .cpp}
for (int i = 0; i < N; ++i)             for (int i = 0; i < N; ++i) {
  for (int j = 0; j < N; ++j)    =>       auto temp = c[i];
    a[j] = b[j] * c[i];                   for (int j = 0; j < N; ++j)
                                            a[j] = b[j] * temp;
                                        }
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**Loop Unrolling**: an _induction variable_ is a variable in a loop, whose value is a function of the loop iteration number. For example, `v = f(i)`, where `i` is an iteration number. Instead of modifying an induction variable on each iteration, we can unroll a loop and perform multiple iterations for each increment of the induction variable (see [@lst:Unrol]).

**循环展开（Loop Unrolling）**：归纳变量（induction variable）是循环中的一个变量，其值是循环迭代次数的函数。例如，`v = f(i)`，其中 `i` 是迭代次数。与其在每次迭代中都修改归纳变量，我们可以展开循环，并针对归纳变量的每个增量执行多次迭代（参见 [@lst:Unrol]）。

Listing: Loop Unrolling
代码列表：循环展开

~~~~ {#lst:Unrol .cpp}
                                        int i = 0;
for (int i = 0; i < N; ++i)             for (; i+1 < N; i+=2) {
  a[i] = b[i] * c[i];          =>         a[i]   = b[i]   * c[i];
                                          a[i+1] = b[i+1] * c[i+1];
                                        }
                                        // remainder (when N is odd)
                                        for (; i < N; ++i)
                                          a[i] = b[i] * c[i];
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The primary benefit of loop unrolling is to perform more computations per iteration. At the end of each iteration, the index value must be incremented and tested, then we go back to the top of the loop if it has more iterations to process. This work is commonly referred to as "loop overhead" or "loop tax", and it can be reduced with loop unrolling. For example, by unrolling the loop in [@lst:Unrol] by a factor of 2, we reduce the number of executed compare and branch instructions by 2x.

循环展开的主要好处是每次迭代执行更多计算。每次迭代结束时，索引值必须递增并进行测试，如果循环还有更多迭代需要处理，则返回循环顶部。这项工作通常被称为“循环开销（loop overhead）”或“循环税（loop tax）”，可以通过循环展开来减少。例如，将 [@lst:Unrol] 中的循环展开2倍，可以将执行的比较和分支指令的数量减少一半。

I do not recommend unrolling loops manually except in cases when you need to break loop-carry dependencies as shown in [@lst:DepChain]. First, because compilers are very good at doing this automatically and usually can choose the optimal unrolling factor. The second reason is that processors have an "embedded unroller" thanks to their out-of-order speculative execution engine (see [@sec:uarch]). While the processor is waiting for long-latency instructions from the first iteration to finish (e.g. loads, divisions, microcoded instructions, long dependency chains), it will speculatively start executing instructions from the second iteration and only wait on loop-carried dependencies. This spans multiple iterations ahead, effectively unrolling the loop in the instruction Reorder Buffer (ROB). The third reason is that unrolling too much could lead to negative consequences due to code bloat.

我不建议手动展开循环，除非需要打破循环进位依赖关系，如 [@lst:DepChain] 所示。首先，编译器非常擅长自动完成这项工作，通常可以选择最佳的展开倍数。其次，由于处理器具有乱序前瞻执行引擎（参见 [@sec:uarch]），因此它们拥有“嵌入式展开器（embeded unroller）”。当处理器等待第一次迭代中延迟较长的指令（例如：加载、除法、微码指令、长依赖链）完成时，它会前瞻性地开始执行第二次迭代中的指令，并且只等待循环进位的依赖项。这会跨越多个迭代周期，有效地将循环展开到指令再定序缓冲区(ROB)中。第三个原因是，过度展开循环会导致代码膨胀，从而产生负面影响。

**Loop Strength Reduction (LSR)**: replace expensive instructions with cheaper ones. Such transformation can be applied to all expressions that use an induction variable. Strength reduction is often applied to array indexing. Compilers perform LSR by analyzing how the value of a variable evolves across the loop iterations. In LLVM, it is known as Scalar Evolution (SCEV). In [@lst:LSR], it is relatively easy for a compiler to prove that the memory location `b[i*10]` is a linear function of the loop iteration number `i`, thus it can replace the expensive multiplication with a cheaper addition.

**循环强度缩减(LSR: Loop Strength Reduction)**：用更便宜的指令替换昂贵的指令。这种转换可以应用于所有使用归纳变量的表达式。强度缩减通常应用于数组索引。编译器通过分析变量值在循环迭代中的演变来执行LSR。在LLVM中，它被称为标量演化(SCEV: SCalar EVolution)。在循环强度缩减(LSR)中，编译器相对容易证明内存位置 `b[i*10]` 是循环迭代次数 `i` 的线性函数，因此可以用更高效的加法来代替耗时的乘法。

Listing: Loop Strength Reduction
代码列表：循环强度缩减

~~~~ {#lst:LSR .cpp}
for (int i = 0; i < N; ++i)             int j = 0;
  a[i] = b[i * 10] * c[i];      =>      for (int i = 0; i < N; ++i) {
                                          a[i] = b[j] * c[i];
                                          j += 10;
                                        }
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**Loop Unswitching**: if a loop has a conditional statement inside and it is invariant, we can move it outside of the loop. We do so by duplicating the body of the loop and placing a version of it inside each of the `if` and `else` clauses of the conditional statement (see [@lst:Unswitch]). While loop unswitching may double the amount of code written, each of these new loops may now be optimized separately.

**循环判断外提（Loop Unswitching）**：如果循环内部包含一个不变的条件语句，我们可以将其移到循环外部。具体做法是复制循环体，并将复制的循环体分别放入条件语句的 `if` 和 `else` 子句中（参见 [@lst:Unswitch]）。虽然循环取消循环切换可能会使代码量翻倍，但每个新增的循环都可以单独进行优化。

Listing: Loop Unswitching
代码列表：循环判断外提

~~~~ {#lst:Unswitch .cpp}
for (i = 0; i < N; i++) {               if (c)
  a[i] += b[i];                           for (i = 0; i < N; i++) {
  if (c)                       =>           a[i] += b[i];
    b[i] = 0;                               b[i] = 0;
}                                         }
                                        else
                                          for (i = 0; i < N; i++) {
                                            a[i] += b[i];
                                          }
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

### High-level Optimizations. 高级优化 {#sec:LoopOptsHighLevel}

High-level loop transformations change the structure of loops and often affect multiple nested loops. In this section, we will take a look at Loop Interchange, Loop Blocking (Tiling), Loop Fusion and Distribution (Fission), and Loop Unroll and Jam. This set of transformations aims at improving memory access and eliminating memory bandwidth and memory latency bottlenecks. From a compiler perspective, it is very difficult to prove the legality of such transformations and justify their performance benefit. In that sense, developers are in a better position since they only have to care about the legality of the transformation in their particular piece of code. Unfortunately, this means usually we have to do such transformations manually.

高级循环转换会改变循环的结构，并且通常会影响多个嵌套循环。在本节中，我们将探讨循环交换、循环分块（切片）、循环融合与分布（分裂）以及循环展开与阻塞。这些转换旨在改善存储访问，消除内存带宽和内存延迟瓶颈。从编译器的角度来看，很难证明此类转换的合法性并证明其性能提升的合理性。从这个意义上讲，开发人员的优势在于他们只需关注转换在其特定代码片段中的合法性即可。遗憾的是，这意味着我们通常需要手动执行此类转换。

**Loop Interchange**: is a process of exchanging the loop order of nested loops. The induction variable used in the inner loop switches to the outer loop, and vice versa. [@lst:Interchange] shows an example of interchanging nested loops for `i` and `j`. The main purpose of this loop interchange is to perform sequential memory accesses to the elements of a multi-dimensional array. By following the order in which elements are laid out in memory, we can improve the spatial locality of memory accesses and make our code more cache-friendly. This transformation helps to eliminate memory bandwidth and memory latency bottlenecks.

**循环交换（Loop Interchange）**：是指交换嵌套循环的循环顺序。内层循环中使用的归纳变量会切换到外层循环，反之亦然。[@lst:Interchange] 展示了交换嵌套循环 `i` 和 `j` 的示例。此循环交换的主要目的是对多维数组的元素进行顺序内存访问。通过遵循元素在内存中的布局顺序，我们可以提高存储访问的空间局部性，并使代码更利于缓存。这种转换有助于消除存储带宽和存储延迟瓶颈。

Listing: Loop Interchange
代码列表：循环交换

~~~~ {#lst:Interchange .cpp}
for (i = 0; i < N; i++)                 for (j = 0; j < N; j++)
  for (j = 0; j < N; j++)          =>     for (i = 0; i < N; i++)
    a[j][i] += b[j][i] * c[j][i];           a[j][i] += b[j][i] * c[j][i];
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Loop Interchange is only legal if loops are *perfectly nested*. A perfectly nested loop is one wherein all the statements are in the innermost loop. Interchanging imperfect loop nests is harder to do but still possible; check an example in the [Codee](https://www.codee.com/catalog/glossary-perfect-loop-nesting/)[^1] catalog.

循环交换仅在循环完全嵌套的情况下才合法。完全嵌套的循环是指所有语句都位于最内层循环中。交换不完全嵌套的循环比较困难，但仍然可行；请参阅 [Codee](https://www.codee.com/catalog/glossary-perfect-loop-nesting/)[^1] 目录中的示例。

**Loop Blocking (Tiling)**: the idea of this transformation is to split the multi-dimensional execution range into smaller chunks (blocks or tiles) so that each block will fit in the CPU caches. If an algorithm works with large multi-dimensional arrays and performs strided accesses to their elements, there is a high chance of poor cache utilization. Every such access may push the data that will be requested by future accesses out of the cache (cache eviction). By partitioning an algorithm into smaller multi-dimensional blocks, we ensure the data used in a loop stays in the cache until it is reused.

**循环分块（切片）（Loop Blocking (Tiling)）**：这种转换的思想是将多维执行范围分割成更小的块（块block或切片tile），以便每个块都能放入CPU缓存中。如果算法处理大型多维数组并对其元素执行步长访问，则很可能出现缓存利用率低的情况。每次此类访问都可能将后续访问所需的数据从缓存中推出（缓存驱逐cache eviction）。通过将算法分割成更小的多维块，我们可以确保循环中使用的数据保留在缓存中，直到再次被重用。

In the example shown in [@lst:blocking], an algorithm performs row-major traversal of elements of array `a` while doing column-major traversal of array `b`. The loop nest can be partitioned into smaller blocks to maximize the reuse of elements in array `b`.

在 [@lst:blocking] 示例中，算法对数组 `a` 的元素进行行优先遍历，同时对数组 `b` 的元素进行列优先遍历。循环嵌套可以分割成更小的块，以最大限度地重用数组 `b` 中的元素。

Listing: Loop Blocking
代码列表：循环分块

~~~~ {#lst:blocking .cpp}
// linear traversal                     // traverse in 8*8 blocks
for (int i = 0; i < N; i++)             for (int ii = 0; ii < N; ii+=8)
  for (int j = 0; j < N; j++)    =>      for (int jj = 0; jj < N; jj+=8)
    a[i][j] += b[j][i];                   for (int i = ii; i < ii+8; i++)
                                           for (int j = jj; j < jj+8; j++)
                                            a[i][j] += b[j][i];
                                        // remainder (not shown)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Loop Blocking is a widely known method of optimizing GEneral Matrix Multiplication (GEMM) algorithms. It enhances the cache reuse of memory accesses and improves the memory bandwidth utilization and memory latency.

循环分块优化是一种广为人知的通用矩阵乘法 (GEMM) 算法优化方法。它能提高存储访问的缓存重用率，并改善存储带宽利用率和降低存储延迟。

Typically, engineers optimize a tiled algorithm for the size of caches that are private to each CPU core (L1 or L2 for Intel and AMD, L1 for Apple). However, the sizes of private caches are changing from generation to generation, so hardcoding a block size presents its own set of challenges. As an alternative solution, you can use [cache-oblivious](https://en.wikipedia.org/wiki/Cache-oblivious_algorithm)[^2] algorithms whose goal is to work reasonably well for any size of the cache.

通常，工程师会针对每个CPU核心的私有缓存大小（Intel和AMD的L1或L2缓存，Apple的L1缓存）优化分块算法。然而，私有缓存的大小会随着CPU代次的变化而变化，因此硬编码块大小会带来一系列挑战。作为一种替代方案，可以使用[缓存无关算法Cache-Oblivious](https://en.wikipedia.org/wiki/Cache-oblivious_algorithm)[^2]，这类算法的目标是在任意大小的缓存下都能良好运行。

**Loop Fusion and Distribution (Fission)**: separate loops can be fused when they iterate over the same range and do not reference each other's data. An example of a Loop Fusion is shown in [@lst:fusion]. The opposite procedure is called Loop Distribution (Fission) when the loop is split into separate loops.

**循环融合与分离（分裂）（Loop Fusion and Distribution (Fission)）**：当两个独立的循环迭代相同的范围且彼此不引用对方的数据时，可以将它们融合在一起。循环融合的示例见[@lst:fusion]。与之相反的过程称为循环分配（分裂），即将一个循环分裂成多个独立的循环。

Listing: Loop Fusion and Distribution
代码列表：循环融合与分离

~~~~ {#lst:fusion .cpp}
for (int i = 0; i < N; i++)             for (int i = 0; i < N; i++) {
  a[i].x = b[i].x;                        a[i].x = b[i].x;
                               =>         a[i].y = b[i].y;
for (int i = 0; i < N; i++)             }
  a[i].y = b[i].y;                      
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Loop Fusion helps to reduce the loop overhead (similar to Loop Unrolling) since both loops can use the same induction variable. Also, loop fusion can help to improve the temporal locality of memory accesses. In [@lst:fusion], if both `x` and `y` members of a structure happen to reside on the same cache line, it is better to fuse the two loops since we can avoid loading the same cache line twice. This will reduce the cache footprint and improve memory bandwidth utilization.

循环融合有助于降低循环开销（类似于循环展开），因为两个循环可以使用同一个归纳变量。此外，循环融合还有助于提高内存访问的时间局部性。在 [@lst:fusion] 中，如果结构体的 `x` 和 `y` 成员恰好位于同一缓存行，则最好融合这两个循环，因为这样可以避免重复加载同一缓存行。这将减少缓存占用并提高内存带宽利用率。

However, loop fusion does not always improve performance. Sometimes it is better to split a loop into multiple passes, pre-filter the data, sort and reorganize it, etc. By distributing the large loop into multiple smaller ones, we limit the amount of data required for each iteration of the loop and increase the temporal locality of memory accesses. This helps in situations with a high cache contention, which typically happens in large loops. Loop distribution also reduces register pressure since, again, fewer operations are being done within each iteration of the loop. Also, breaking a big loop into multiple smaller ones will likely be beneficial for the performance of the CPU Frontend because of better instruction cache utilization. Finally, when distributed, each small loop can be further optimized separately by the compiler.

然而，循环融合并非总是能提高性能。有时，将循环拆分为多个遍历、预过滤数据、排序和重组数据等操作会更好。通过将大型循环拆分为多个小型循环，我们可以限制每次循环迭代所需的数据量，并提高内存访问的时间局部性。这有助于解决缓存争用高的问题，这种情况通常发生在大型循环中。循环拆分还可以降低寄存器压力，因为每次循环迭代中执行的操作更少。此外，将一个大循环拆分成多个小循环，由于指令缓存利用率更高，通常有利于提升CPU前端的性能。最后，分布式循环中，每个小循环都可以由编译器进行单独优化。

**Loop Unroll and Jam**: to perform this transformation, you need to unroll the outer loop first, then jam (fuse) multiple inner loops together as shown in [@lst:unrolljam]. This transformation increases the ILP (Instruction-Level Parallelism) of the inner loop since more independent instructions are executed inside the inner loop. In the code example, the inner loop is a reduction operation that accumulates the deltas between elements of arrays `a` and `b`. When we unroll and jam the loop nest by a factor of two, we effectively execute two iterations of the original outer loop simultaneously. This is emphasized by having two independent accumulators. This breaks the dependency chain over `diffs` in the initial variant.

**循环展开和合并（Loop Unroll and Jam）**：要进行这种转换，需要先展开外层循环，然后像 [@lst:unrolljam] 中所示那样，将多个内层循环合并（合并）在一起。这种转换提高了内层循环的指令级并行度 (ILP)，因为内层循环内部执行了更多独立的指令。在代码示例中，内层循环是一个归约操作，用于累加数组 `a` 和 `b` 元素之间的差异。当我们将循环嵌套展开并合并两倍时，实际上同时执行了原始外层循环的两次迭代。两个独立的累加器进一步强化了这一点。这打破了初始版本中对 `diffs` 的依赖关系。

Listing: Loop Unroll and Jam
代码列表：循环展开和合并

~~~~ {#lst:unrolljam .cpp}
for (int i = 0; i < N; i++)           for (int i = 0; i+1 < N; i+=2)
  for (int j = 0; j < M; j++)           for (int j = 0; j < M; j++) {
    diffs += a[i][j] - b[i][j];   =>      diffs1 += a[i][j]   - b[i][j];
                                          diffs2 += a[i+1][j] - b[i+1][j];
                                        }
                                      diffs = diffs1 + diffs2;
                                      // remainder (not shown)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Loop Unroll and Jam can be performed as long as there are no cross-iteration dependencies on the outer loops, in other words, two iterations of the inner loop can be executed in parallel. Also, this transformation makes sense if the inner loop has memory accesses that are strided on the outer loop index (`i` in this case), otherwise, other transformations likely apply better. The Unroll and Jam technique is especially useful when the trip count of the inner loop is low, e.g., less than 4. By doing the transformation, we pack more independent operations into the inner loop, which increases the ILP.

只要外层循环之间不存在交叉迭代依赖关系，也就是说，内层循环的两次迭代可以并行执行，就可以执行循环展开和合并操作。此外，如果内层循环的内存访问步长与外层循环的索引（本例中为 `i`）相同，则此转换也适用；否则，其他转换可能更合适。当内层循环的迭代次数较低时（例如，小于4次），循环展开和合并技术尤其有用。通过此转换，我们可以将更多独立的操作打包到内层循环中，从而提高指令级并行性(ILP)。

The Unroll and Jam transformation sometimes is very useful for outer loop vectorization, which, at the time of writing, compilers cannot do automatically. In a situation when the trip count of the inner loop is not visible to a compiler, the compiler could still vectorize the original inner loop, hoping that it will execute enough iterations to hit the vectorized code (more on vectorization in the next section). But if the trip count is low, the program will use a slow scalar version of the loop. By performing Unroll and Jam, we enable the compiler to vectorize the code differently: now "gluing" the independent instructions in the inner loop together. This technique is also known as Superword-Level Parallelism (SLP) vectorization.

循环展开和合并转换有时对外部循环向量化非常有用，而截至撰写本文时，编译器尚无法自动执行外层循环向量化。在编译器无法感知内层循环迭代次数的情况下，编译器仍然可以对原始内层循环进行向量化，希望它能够执行足够的迭代次数来触发向量化后的代码（下一节将详细介绍向量化）。但如果迭代次数较低，程序将使用速度较慢的标量循环版本。通过执行展开和合并操作，我们可以让编译器以不同的方式向量化代码：将内循环中的独立指令“粘合”在一起。这种技术也称为超字级并行（SLP: Superword-Level Parallelism）向量化。

### Discovering Loop Optimization Opportunities. 发现循环优化机会。 {#sec:DiscoverLoopOpts}

As we discussed at the beginning of this section, compilers will do the heavy-lifting part of optimizing your loops. You can count on them to make all the obvious improvements in the code of your loops, like eliminating unnecessary work, doing various peephole optimizations, etc. Sometimes a compiler is clever enough to generate the fast version of a loop automatically, but other times we have to do some rewriting to help the compiler. As we said earlier, from a compiler's perspective, doing loop transformations legally and automatically is very difficult. Often, compilers have to be conservative when they cannot prove the legality of a transformation. 

正如我们在本节开头讨论的那样，编译器会承担循环优化的繁重工作。您可以指望它们对循环代码进行所有显而易见的改进，例如消除不必要的操作、执行各种窥孔优化等等。有时编译器足够智能，可以自动生成循环的快速版本，但有时我们需要进行一些重写来帮助编译器。正如我们之前所说，从编译器的角度来看，合法且自动地进行循环转换非常困难。通常，当编译器无法证明转换的合法性时，它们必须采取保守的做法。

As a first step, you can check compiler optimization reports or examine the machine code[^16] of the loop to search for easy improvements. Sometimes, it's possible to adjust compiler transformations using user directives. There are compiler pragmas that control certain transformations, like loop vectorization, loop unrolling, loop distribution, and others. For a complete list of user directives, check your compiler's manual.

首先，您可以查看编译器优化报告或检查循环的机器代码[^16]，以寻找易于改进的地方。有时，可以使用用户指令调整编译器转换。有一些编译器编译指示可以控制某些转换，例如：循环向量化、循环展开、循环分布等等。有关用户指令的完整列表，请查阅编译器手册。

Next, you should identify the bottlenecks in the loop and assess performance against the hardware theoretical maximum. The Top-down Microarchitecture Analysis (TMA, see [@sec:TMA]) methodology and the Roofline performance model ([@sec:roofline]) both help with that. The performance of a loop is limited by one or many of the following factors: memory latency, memory bandwidth, or the computing capabilities of a machine. Once the performance bottlenecks in a loop have been identified, try applying the required transformations that we discussed in a few previous sections.

接下来，您应该识别循环中的瓶颈，并根据硬件理论最大值评估性能。自顶向下微体系结构分析 (TMA，参见 [@sec:TMA]) 方法和屋顶线Roofline性能模型 ([@sec:roofline]) 都有助于此。循环的性能受以下一个或多个因素的限制：存储延迟、存储带宽或机器的计算能力。一旦确定了循环中的性能瓶颈，请尝试应用我们在前面几节中讨论的必要转换。

Even though there are well-known optimization techniques for a particular set of computational problems, loop optimizations remain a "black art" that comes with experience. I recommend that you rely on your compiler and complement it with manually transforming code when necessary. Above all, keep the code as simple as possible and do not introduce unreasonably complicated changes if the performance benefits are negligible.

尽管针对特定计算问题存在一些众所周知的优化技术，但循环优化仍然是一门需要经验积累的“黑魔法”。我建议您主要依靠编译器，并在必要时辅以手动代码修改。最重要的是，尽可能保持代码简洁，如果性能提升微乎其微，就不要引入过于复杂的改动。

[^1]: Codee: perfect loop nesting Codee：完美循环嵌套 - [https://www.codee.com/catalog/glossary-perfect-loop-nesting/](https://www.codee.com/catalog/glossary-perfect-loop-nesting/)
[^2]: Cache-oblivious algorithm 缓存无关算法 - [https://en.wikipedia.org/wiki/Cache-oblivious_algorithm](https://en.wikipedia.org/wiki/Cache-oblivious_algorithm)
[^16]: Sometimes difficult, but always a rewarding activity. 有时难度较高，但总能带来丰厚的回报。
[^17]: The compiler will perform the transformation only if it can prove that `a` and `c` don’t alias. 编译器只有在能够证明 `a` 和 `c` 不构成别名时才会执行转换。
