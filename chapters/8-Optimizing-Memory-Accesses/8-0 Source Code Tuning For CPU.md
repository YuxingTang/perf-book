\phantomsection
# Part 2. Source Code Tuning 第二部分：源代码调优 {.unnumbered}

\markboth{Part 2. Source Code Tuning}{Part 2. Source Code Tuning}

In Part 1 of this book, we discussed how to find performance bottlenecks in code. In Part 2 we will discuss how to fix such
bottlenecks through the use of techniques for low-level source code optimization, also known as *tuning*.

本书第一部分讨论了如何查找代码中的性能瓶颈。第二部分将讨论如何通过使用底层源代码优化技术（也称为*调优tuning*）来解决这些瓶颈。

A modern CPU is a very complicated device, and it's nearly impossible to predict how fast certain pieces of code will run. Software and hardware performance depends on many factors, and the number of moving parts is too big for a human mind to contain. Hopefully, observing how your code runs from a CPU perspective is possible thanks to all the performance monitoring capabilities we discussed in the first part of the book. We will extensively rely on methods and tools we learned about earlier in the book to guide our performance engineering process.

现代CPU是一种非常复杂的器件设备，几乎不可能预测特定代码片段的运行速度。软件和硬件性能取决于许多因素，其涉及的组件数量之多，人类的思维难以完全掌握。希望借助本书第一部分讨论的所有性能监控功能，能够从CPU的角度观察代码的运行情况。我们将大量使用本书前面介绍的方法和工具来指导我们的性能工程过程。

At a very high level, software optimizations can be divided into five categories.

从宏观层面来看，软件优化可以分为五类。

* **Algorithmic optimizations**. Analyze algorithms and data structures used in a program, and see if you can find better ones. Example: use quicksort instead of bubble sort.
* **算法优化**。分析程序中使用的算法和数据结构，看看能否找到更优的方案。例如：使用快速排序代替冒泡排序。
* **Parallelizing computations**. If an algorithm is highly parallelizable, make the program multithreaded, or consider running it on a GPU. The goal is to do multiple things at the same time. Concurrency is already used in all the layers of the hardware and software stacks. Examples: distribute the work across several threads; balance load between many servers in the data center; use async IO to avoid blocking while waiting for IO operations; keep multiple concurrent network connections to overlap the request latency.
* **并行化计算**。如果某个算法高度可并行化，则应将其编写为多线程程序，或考虑在GPU上运行。目标是同时执行多项任务。并发性已应用于硬件和软件栈的各个层面。例如：将工作分配到多个线程；在数据中心的多个服务器之间平衡负载；使用异步I/O来避免等待I/O操作时阻塞；保持多个并发网络连接以重叠请求延迟。
* **Eliminating redundant work**. Don't do work that you don't need or have already done. Examples: leverage using more RAM to reduce the amount of CPU and IO you have to use (caching, look-up tables, compression); pre-compute values known at compile-time; move loop invariant computations outside of the loop; pass a C++ object by reference to get rid of excessive copies caused by passing by value.
* **消除冗余工作**。不要执行不需要或已经完成的工作。例如：利用更多存储来减少CPU和I/O 的使用量（缓存、查找表、压缩）；预先计算编译时已知的值；将循环不变的计算移到循环之外；按引用传递C++对象，以避免按值传递导致的过度复制。
* **Batching**. Aggregate multiple similar operations and do them in one go, thus reducing the overhead of repeating the action multiple times. Examples: send large TCP packets instead of many small ones; allocate a large block of memory rather than allocating space for hundreds of tiny objects.
* **批处理**。将多个类似的操作聚合起来，一次性执行，从而减少重复操作的开销。例如：发送大型TCP数据包而不是多个小型数据包；分配一大块内存而不是为数百个小型对象分配空间。
* **Ordering**. Reorder the sequence of operations in an algorithm. Examples: change data layout to enable sequential memory accesses; sort an array of C++ polymorphic objects based on their types to allow better prediction of virtual function calls; group hot functions together and place them closer to each other in a binary.
* **排序**。重新排列算法中操作的顺序。例如：更改数据布局以支持顺序内存访问；根据类型对C++多态对象数组进行排序，以便更好地预测虚函数调用；将热点函数分组并将它们在二进制文件中彼此靠近放置。

Many optimizations that we discuss in this book fall under multiple categories. For example, we can say that vectorization is a combination of parallelizing and batching; and loop blocking (tiling) is a manifestation of batching and eliminating redundant work.

本书中讨论的许多优化都属于多个类别。例如，我们可以说向量化是并行化和批处理的结合；循环阻塞（分块tiling）是批处理和消除冗余工作的一种体现。

To complete the picture, let us also list other possibly obvious but still quite reasonable ways to speed up things:

为了更全面地了解情况，我们不妨再列举一些其他可能显而易见但仍然非常合理的加速方法：

* **Rewrite the code in another language**: if a program is written using interpreted languages (Python, JavaScript, etc.), rewrite its performance-critical portion in a language with less overhead, e.g., C++, Rust, Go, etc.
* **使用其他语言重写代码**：如果程序是用解释型语言（例如：Python、JavaScript等）编写的，请将其性能关键部分重写为开销更小的语言，例如：C++、Rust、Go等。
* **Tune compiler options**: check that you use at least these three compiler flags: `-O3` (enables machine-independent optimizations), `-march` (enables optimizations for particular CPU architecture), `-flto` (enables inter-procedural optimizations). But don't stop here; there are many other options that affect performance. We will look at some of these in later chapters. You may consider mining the best set of options for an application; commercial products that automate this process are available.
* **调整编译器选项**：确保至少使用了以下三个编译器标志：`-O3`（启用与机器无关的优化）、`-march`（启用针对特定CPU体系结构的优化）和 `-flto`（启用过程间优化）。但这还不够；还有许多其他选项会影响性能。我们将在后续章节中介绍其中的一些。您可以考虑为应用程序找到最佳的选项组合；市面上有一些商业产品可以自动完成此过程。
* **Optimize third-party software packages**: the vast majority of software projects leverage layers of proprietary and open-source code. This includes OS, libraries, and frameworks. You can seek improvements by replacing, modifying, or reconfiguring one of those pieces.
* **优化第三方软件包**：绝大多数软件项目都利用了多层专有代码和开源代码。这包括操作系统、库和框架。您可以通过更换、修改或重新配置这些组件来寻求改进。
* **Buy faster hardware**: obviously, this is a business decision that comes with an associated cost, but sometimes it's the only way to improve performance when other options have already been exhausted. It is much easier to justify a purchase when you identify performance bottlenecks in your application and communicate them clearly to the upper management. For example, once you find that memory bandwidth is limiting the performance of your multithreaded program, you may suggest buying server motherboards and processors with more memory channels and DIMM slots.
* **购买更快的硬件**：显然，这是一项需要投入成本的商业决策，但有时当其他方法都已尝试无效时，这可能是提升性能的唯一途径。如果您能识别出应用程序中的性能瓶颈并将其清晰地传达给高层管理人员，那么购买硬件就更容易被接受。例如，一旦您发现内存带宽限制了多线程程序的性能，您可以建议购买具有更多内存通道和DIMM插槽的服务器主板和处理器。

### Algorithmic Optimizations 算法优化 {.unlisted .unnumbered}

Standard algorithms and data structures don't always work well for performance-critical workloads. For example, traditionally, every new node of a linked list is dynamically allocated. Besides potentially invoking many costly memory allocations, this will likely result in a situation where all the elements of the list are scattered in memory. Traversing such a data structure is not cache-friendly. Even though algorithmic complexity is still O(N), in practice, the timings will be much worse than those of a plain array. Some data structures, like binary trees, have a natural linked-list-like representation, so it might be tempting to implement them in a pointer-chasing manner. However, more efficient "flat" versions of those data structures exist, e.g., `boost::flat_map` and `boost::flat_set`.

标准算法和数据结构并非总能很好地应对对性能要求极高的工作负载。例如，传统上，链表的每个新节点都是动态分配的。这不仅可能导致大量代价高昂的内存分配，还可能导致链表的所有元素分散在内存中。遍历这样的数据结构不利于缓存优化。即使算法复杂度仍然是O(N)，但实际上，其运行时间会比普通数组长得多。某些数据结构，例如：二叉树，具有类似链表的自然表示形式，因此很容易被指针追踪的方式所吸引。然而，这些数据结构存在更高效的“扁平化”版本，例如 `boost::flat_map` 和 `boost::flat_set`。

When selecting an algorithm for a problem at hand, you might quickly pick the most popular option and move on... even though it could not be the best for your particular case. Let's assume you need to find an element in a sorted array. The first option that most developers consider is binary search, right? It is very well-known and is optimal in terms of algorithmic complexity, O(logN). Will you change your decision if I say that the array holds 32-bit integer values and the size of an array is usually very small (less than 20 elements)? In the end, measurements should guide your decision, but binary search suffers from branch mispredictions since every test of the element value has a 50% chance of being true. This is why on a small array, a linear scan is usually faster even though it has worse algorithmic complexity.[^1]

在为当前问题选择算法时，你可能会迅速选择最流行的方案并继续前进……即使它可能并非最适合你的特定情况。假设你需要在一个已排序的数组中查找一个元素。大多数开发者首先考虑的选项是二分查找，对吧？它非常知名，并且在算法复杂度方面是最优的，为O(logN)。如果我说数组存储的是32位整数值，并且数组的大小通常很小（少于20个元素），你会改变主意吗？最终，实际应用场景应该指导你的决策，但二分查找存在分支预测错误的问题，因为每次元素值测试都有50%的概率为真。这就是为什么对于小型数组，即使线性扫描的算法复杂度更高，它通常也更快[^1]。

### Data-Driven Development 数据驱动开发 {.unlisted .unnumbered}

The main idea in Data-Driven Development (DDD), is to study how a program accesses data: how it is laid out in memory and how it is transformed throughout the program; then modify the program accordingly (change the data layout, change the access patterns). A classic example of such an approach is the Array-of-Structures (AOS) to Structure-of-Array (SOA) transformation, which is shown in [@lst:AOStoSOA]. 

数据驱动开发(DDD: Data-Driven Development)的核心思想是研究程序如何访问数据：数据在内存中的布局方式以及在整个程序运行过程中如何转换；然后相应地修改程序（更改数据布局，更改访问模式）。这种方法的经典示例是结构体数组(AOS: Array-of-Structures)到数组结构体(SOA: Structure-of-Array)的转换，如 [@lst:AOStoSOA] 所示。

Listing: SOA to AOS transformation.
代码列表：SOA到AOS的转换

~~~~ {#lst:AOStoSOA .cpp}
// Array of Structures (AOS)             // Structure of Arrays (SOA)
struct S {                               struct S {
  int a;                                   int a[N];
  int b;                        <=>        int b[N];
  int c;                                   int c[N];
  // other fields                          // other arrays
};                                       };
S s[N];                                  S s;
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The answer to the question of which layout is better depends on how the code is accessing the data. If it iterates over the data structure `S` and only accesses field `b`, then SOA is better because all memory accesses will be sequential. However, if a program iterates over the data structure and does *extensive* operations on all the fields of the object, then AOS may give better memory bandwidth utilization and in some cases, better performance. In the AOS scenario, members of the struct are likely to reside in the same cache line, and thus require fewer cache line reads and use less cache space. But more often, we see SOA gives better performance as it enables other important transformations, for example, vectorization.

哪种布局更好取决于代码访问数据的方式。如果代码遍历数据结构 `S` 并且只访问字段 `b`，那么SOA更优，因为所有内存访问都是顺序的。然而，如果程序遍历数据结构并对对象的所有字段执行*大量*操作，那么AOS可能提供更高的内存带宽利用率，在某些情况下还能带来更好的性能。在AOS场景下，结构体的成员很可能驻留在同一缓存行中，因此需要的缓存行读取次数更少，占用的缓存空间也更少。但更常见的情况是，SOA能够带来更好的性能，因为它支持其他重要的转换，例如向量化。

Another widespread example of DDD is small-size optimization. The idea is to statically preallocate some amount of memory to avoid dynamic memory allocations. It is especially useful for small and medium-sized containers when the upper limit of elements can be well-predicted. Modern C++ STL implementations of `std::string` keep the first 15--20 characters in an internal buffer allocated in the stack memory and allocate memory on the heap only for longer strings. Other instances of this approach can be found in LLVM's `SmallVector` and Boost's `static_vector`.

DDD的另一个常见例子是小尺寸优化。其思想是静态预分配一定量的内存，以避免动态内存分配。当元素数量的上限可以很好地预测时，这种方法对于中小型容器尤其有用。现代C++ STL实现的 `std::string` 会将字符串的前15-20个字符保存在栈内存分配的内部缓冲区中，只有当字符串更长时才会分配堆内存。LLVM的 `SmallVector` 和Boost的 `static_vector` 也采用了类似的方法。

### Low-Level Optimizations 底层优化 {.unlisted .unnumbered}

Performance engineering is an art. And like in any art, the set of possible scenarios is endless. It's impossible to cover all the various optimizations one can imagine. The chapters in Part 2 primarily address optimizations specific to modern CPU architectures. 

性能工程是一门艺术。就像任何艺术一样，可能的情况无穷无尽。我们不可能涵盖所有可以想象到的优化方法。第二部分的章节主要讨论针对现代CPU体系结构的优化。

Before we jump into particular source code tuning techniques, there are a few caution notes to make. First, avoid tuning bad code. If a piece of code has a high-level performance inefficiency, you shouldn't apply machine-specific optimizations to it. Always focus on fixing the major problem first. Only once you're sure that the algorithms and data structures are optimal for the problem you're trying to solve should you try applying low-level improvements.

在深入探讨具体的源代码调优技巧之前，有几点需要注意。首先，避免对糟糕的代码进行调优。如果一段代码存在严重的性能缺陷，就不应该对其应用针对特定机器的优化。始终应该首先专注于解决根本问题。只有当您确定算法和数据结构对于您要解决的问题而言是最优的，才应该尝试应用底层改进。

Second, remember that an optimization you implement might not be beneficial on every platform. For example, Loop Blocking (tiling), which is discussed in [@sec:LoopOptsHighLevel], depends on characteristics of the memory hierarchy in a system, especially L2 and L3 cache sizes. So, an algorithm tuned for a CPU with particular sizes of L2 and L3 caches might not work well for CPUs with smaller caches. It is important to test the change on the platforms your application will be running on.

其次，请记住，您实现的优化可能并非在所有平台上都有效。例如，循环阻塞（分块）技术（在 [@sec:LoopOptsHighLevel] 中讨论过）取决于系统中的存储层次结构特性，尤其是L2和L3缓存的大小。因此，针对具有特定大小L2和L3缓存的CPU优化的算法可能不适用于缓存较小的 CPU。务必在应用程序将要运行的平台上测试更改。

The next four chapters are organized according to the TMA classification (see [@sec:TMA]):

接下来的四章将根据TMA分类进行组织（参见[@sec:TMA]）：

* [@sec:MemBound]. Optimizing Memory Accesses---the `TMA:MemoryBound` category.
* [@sec:CoreBound]. Optimizing Computations---the `TMA:CoreBound` category.
* [@sec:ChapterBadSpec]. Optimizing Branch Prediction---the `TMA:BadSpeculation` category.
* [@sec:secFEOpt]. Machine Code Layout Optimizations---the `TMA:FrontendBound` category.

* [@sec:MemBound]。优化内存访问—— `TMA:内存瓶颈MemoryBound` 类别。
* [@sec:CoreBound]。优化计算—— `TMA:核心瓶颈CoreBound` 类别。
* [@sec:ChapterBadSpec] 优化分支预测——属于 `TMA:前瞻错误BadSpeculation` 类别。
* [@sec:secFEOpt] 机器代码布局优化——属于 `TMA:前端布局FrontendBound` 类别。

The idea behind this classification is to offer a checklist for developers when they are using TMA methodology in their performance engineering work. Whenever TMA attributes a performance bottleneck to one of the categories mentioned above, feel free to consult one of the corresponding chapters to learn about your options.

此分类背后的含义是旨在为开发者在使用TMA方法进行性能工程工作时提供一份检查清单。如果TMA将性能瓶颈归因于上述任何类别，您可以参考相应的章节了解相关选项。

[@sec:ChapterOtherTuning] covers optimization topics not specifically related to any of the categories covered in the previous four chapters. In this chapter, we will discuss CPU-specific optimizations, examine several microarchitecture-related performance problems, explore techniques used for optimizing low-latency applications, and give you advice on tuning your system for the best performance.

[@sec:ChapterOtherTuning] 涵盖了与前四章所述类别无关的优化主题。本章将讨论CPU相关的优化，探讨一些与微体系结构相关的性能问题，探索用于优化低延迟应用程序的技术，并就如何调整系统以获得最佳性能提供建议。

[@sec:secOptMTApps] addresses some common problems in optimizing multithreaded applications. It provides a case study of five real-world multithreaded applications, where we explain why their performance doesn't scale with the increasing number of CPU threads. We also discuss cache coherency issues, such as "False Sharing" and a few tools that are designed to analyze multithreaded applications.

[@sec:secOptMTApps] 解决了一些优化多线程应用程序的常见问题。本文提供了5个真实世界多线程应用程序的案例研究，解释了为什么它们的性能无法随着CPU线程数的增加而提升。我们还讨论了缓存一致性问题，例如“伪共享（False Sharing）”，以及一些用于分析多线程应用程序的工具。

[^1]: In addition, linear scan does not require elements to be sorted. 另外，线性扫描不需要元素预先被排序好。
