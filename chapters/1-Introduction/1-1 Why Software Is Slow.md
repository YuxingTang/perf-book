## Why Is Software Slow?为什么软件非常慢？

If all the software in the world utilized all available hardware resources efficiently, then this book would not exist. We would not need any changes on the software side and would rely on what existing processors have to offer. But you already know that the reality is different, right? The reality is that modern software is *massively* inefficient. A regular server system in a public cloud typically runs poorly optimized code, consuming more power than it could have consumed (increasing carbon emissions and contributing to other environmental issues). If we could make all software run two times faster, we would potentially reduce the carbon footprint of computing by a factor of two.

如果世界上所有的软件都能高效利用所有可用的硬件资源，那么这本书就不存在了。我们无需对软件进行任何改动，只需依赖现有处理器提供的内容即可。但你也知道，现实并非如此，对吧？现实是，现代软件的效率极其低下。公有云中的普通服务器系统通常运行的是优化程度很低的代码，消耗的电量远超其应有的水平（这不仅增加了碳排放，还加剧了其他环境问题）。如果我们能让所有软件的运行速度提高一倍，那么计算的碳足迹就有可能减少一半。

The authors of the paper [@Leisersoneaam9744] provide an excellent example that illustrates the performance gap between "default" and highly optimized software. Table @tbl:PlentyOfRoom summarizes speedups from performance engineering a program that multiplies two 4096-by-4096 matrices. The end result of applying several optimizations is a program that runs over 60,000 times faster. The reason for providing this example is not to pick on Python or Java (which are great languages), but rather to break beliefs that software has "good enough" performance by default. The majority of programs are within rows 1--5. The potential for source-code-level improvements is significant.

《科学》期刊论文作者 [@Leisersoneaam9744] 提供了一个很好的例子，说明了“默认”软件和高度优化软件之间的性能差距。表 @tbl:PlentyOfRoom 总结了通过性能工程优化一个计算2个4096×4096矩阵乘积的程序所获得的加速效果。经过多项优化，最终程序的运行速度提高了6万倍以上。提供这个例子并非意在贬低Python或Java（它们都是优秀的语言），而是为了打破“软件默认性能就足够好”的固有观念。大多数程序的主体都集中在第1到5行。源代码层面的改进潜力巨大。

-------------------------------------------------------------
Version   Implementation                 Absolute    Relative 
                                         speedup     speedup

-------   ----------------------------   --------    --------
   1         Python                         1            —

   2          Java                         11          10.8

   3           C                           47           4.4

   4      Parallel loops                   366          7.8

   5      Parallel divide and conquer     6,727        18.4
            
   6       plus vectorization            23,224         3.5
           
   7       plus AVX intrinsics           62,806         2.7

--------------------------------------------------------------

Table: Speedups from performance engineering a program that multiplies two 4096-by-4096 matrices running on a dual-socket Intel Xeon E5-2666 v3 system with a total of 60 GB of memory. From [@Leisersoneaam9744]. 表：在一台总计40GB内存的双路Intel Xeon E5-2666 V3系统上对两个4096x4096矩阵相乘的程序进行性能工程的加速结果。源自[@Leisersoneaam9744]。{#tbl:PlentyOfRoom}

So, let's talk about what prevents systems from achieving optimal performance by default. Here are some of the most important factors:

那么，让我们来谈谈是什么阻碍了系统默认达到最佳性能。以下是一些最重要的因素：

1. **CPU limitations**: It's so tempting to ask: "*Why doesn't hardware solve all our problems?*" Modern CPUs execute instructions incredibly quickly, and are getting better with every generation. But still, they cannot do much if instructions that are used to perform the job are not optimal or even redundant. Processors cannot magically transform suboptimal code into something that performs better. For example, if we implement a bubble sort, a CPU will not make any attempts to recognize it and use better alternatives (e.g. quicksort). It will blindly execute whatever it was told to do.
1. **CPU的局限**：我们很容易会问：“*为什么硬件不能解决我们所有的问题？*” 现代CPU执行指令的速度非常快，而且每一代都在进步。但是，如果用于执行任务的指令不是最优的，甚至是冗余的，它们也无能为力。处理器无法神奇地将次优代码转换成性能更好的代码。例如，如果我们实现了一个冒泡排序，CPU不会尝试识别它并使用更优的替代方案（例如：快速排序）。它会盲目地执行被告知要执行的任何操作。
2. **Compiler limitations**: "*But isn't that what compilers are supposed to do? Why don't compilers solve all our problems?*" Indeed, compilers are amazingly smart nowadays, but can still generate suboptimal code. Compilers are great at eliminating redundant work, but when it comes to making more complex decisions like vectorization, they may not generate the best possible code. Performance experts often can come up with clever ways to vectorize loops beyond the capabilities of compilers. When compilers have to make a decision whether to perform a code transformation or not, they rely on complex cost models and heuristics, which may not work for every possible scenario. For example, there is no binary "yes" or "no" answer to the question of whether a compiler should always inline a function into the place where it's called. It usually depends on many factors which a compiler should take into account. Additionally, compilers cannot perform optimizations unless they are absolutely certain it is safe to do so. It may be very difficult for a compiler to prove that an optimization is correct under all possible circumstances, disallowing some transformations. Finally, compilers generally do not attempt "heroic" optimizations, like transforming data structures used by a program.
2. **编译器的局限**：“*但编译器不就是应该做这些的吗？为什么编译器不能解决我们所有的问题？*” 诚然，如今的编译器非常智能，但仍然可能生成次优代码。编译器擅长消除冗余工作，但在处理向量化等更复杂的决策时，它们可能无法生成最优代码。性能专家通常能够找到巧妙的方法来对循环进行向量化，而这些方法超出了编译器的能力范围。当编译器必须决定是否执行代码转换时，它们依赖于复杂的成本模型和启发式算法，而这些模型和算法可能并不适用于所有情况。例如，对于“编译器是否应该始终将函数内联到其调用位置”这个问题，并没有简单的“是”或“否”的答案。这通常取决于编译器需要考虑的诸多因素。此外，编译器只有在绝对确定这样做是安全的的情况下才能执行优化。编译器可能很难证明某种优化在所有情况下都是正确的，因此某些转换可能无法实现。最后，编译器通常不会尝试“冒险/英雄式”的优化，例如（通常不会试图）转换程序所使用的数据结构。
3. **Algorithmic complexity analysis limitations**: Some developers are overly obsessed with algorithmic complexity analysis, which leads them to choose a popular algorithm with the optimal algorithmic complexity, even though it may not be the most efficient for a given problem. Considering two sorting algorithms, insertion sort and quicksort, the latter clearly wins in terms of Big O notation for the average case: insertion sort is O(N^2^) while quickSort is only O(N log N). Yet for relatively small sizes of `N` (up to 50 elements), insertion sort outperforms quickSort. Complexity analysis cannot account for all the low-level performance effects of various algorithms, so people just encapsulate them in an implicit constant `C`, which sometimes can make a large impact on performance. Only counting comparisons and swaps that are used for sorting, ignores cache misses and branch mispredictions, which, today, are actually very costly. Blindly trusting Big O notation without testing on the target workload could lead developers down an incorrect path. So, the best-known algorithm for a certain problem is not necessarily the most performant in practice for every possible input.
3. **算法复杂度分析的局限性**：一些开发者过度关注算法复杂度分析，导致他们选择算法复杂度最优的流行算法，即使该算法对于特定问题可能并非最高效。以插入排序和快速排序为例，在平均情况下，快速排序在O符号后很大时明显更胜一筹：插入排序的复杂度为O(N²)，而快速排序仅为O(N log N)。然而，对于相对较小的`N`值（最多50个元素），插入排序的性能优于快速排序。复杂度分析无法涵盖各种算法的所有底层性能影响，因此人们通常将其封装在一个隐式常量`C`中，而这有时会对性能产生显著影响。仅计算排序过程中使用的比较和交换操作，忽略了缓存未命中和分支预测错误，而这些错误在当今实际上代价非常高昂。盲目信任大O符号而不针对目标工作负载进行测试，可能会误导开发者。因此，针对特定问题的最佳算法未必是所有可能输入情况下性能最优的算法。
In addition to the limitations described above, there are overheads created by programming paradigms. Coding practices that prioritize code clarity, readability, and maintainability can reduce performance. Highly generalized and reusable code can introduce unnecessary copies, runtime checks, function calls, memory allocations, etc. For instance, polymorphism in object-oriented programming is usually implemented using virtual functions, which introduce a performance overhead.[^1]
除了上述限制之外，编程范式也会带来额外的开销。优先考虑代码清晰度、可读性和可维护性的编码实践可能会降低性能。高度通用和可重用的代码可能会引入不必要的复制、运行时检查、函数调用、内存分配等。例如，面向对象编程中的多态性通常使用虚函数来实现，这会带来性能开销。[^1]
All the factors mentioned above assess a "performance tax" on the software. There are very often substantial opportunities for tuning the performance of our software to reach its full potential.
上述所有因素都会对软件性能造成“负担/性能税”。我们通常有很多机会可以优化软件性能，使其发挥全部潜力。
[^1]: I do not dismiss design patterns and clean code principles, but I encourage a more nuanced approach where performance is also a key consideration in the development process.我并不否定设计模式和代码整洁原则，但我鼓励采用更细致的方法，将性能也作为开发过程中的一个关键考虑因素。
