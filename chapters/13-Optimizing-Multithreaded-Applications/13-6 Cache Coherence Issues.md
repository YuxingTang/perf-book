## Cache Coherence 缓存一致性 {#sec:TrueFalseSharing}

Multiprocessor systems incorporate means to ensure data coherence during shared usage of memory by each core containing its own, separate cache entity. Without such a protocol, if both CPU `A` and `B` read memory location `L` into their individual caches, and CPU `B` subsequently modifies its cached value for `L`, then the CPUs would have incoherent values of the same memory location `L`. Cache Coherency Protocols ensure that any updates to cached entries are dutifully updated or invalidated in any other cached entry of the same location.

多处理器系统包含一些机制，以确保每个核心共享内存时（每个核心都拥有独立的缓存实体）的数据一致性。如果没有这样的协议，如果 CPU `A` 和 `B` 都将内存位置 `L` 读取到各自的缓存中，而 CPU `B` 随后修改了其缓存中 `L` 的值，那么两个CPU对同一内存位置 `L` 的缓存值就会不一致。缓存一致性协议确保对缓存条目的任何更新都会在同一位置的任何其他缓存条目中得到相应的更新或失效。

### Cache Coherency Protocols 缓存一致性协议

One of the most well-known cache coherency protocols is MESI (**M**odified **E**xclusive **S**hared **I**nvalid), which is used to support writeback caches like those used in modern CPUs. Its acronym denotes the four states with which a cache line can be marked (see Figure @fig:MESI):

最著名的缓存一致性协议之一是MESI（**M**odified **E**xclusive **S**hared **I**nvalid），它用于支持像在现代处理器中使用的写回缓存。它的缩写表示缓存行可以标记为的四种状态（参见图 @fig:MESI）：

* **Modified**: a cache line is present only in the current cache and has been modified from its value in RAM
* **Exclusive**: a cache line is present only in the current cache and matches its value in RAM
* **Shared**: a cache line is present here and in other cache lines and matches its value in RAM
* **Invalid**: a cache line is unused (i.e., does not contain any RAM location)

* **已修改Modified**：缓存行仅存在于当前缓存中，其值已现对于RAM中的值进行了修改
* **独占Exclusive**：缓存行仅存在于当前缓存中，且其值与RAM中的值相同
* **共享Shared**：缓存行同时存在于当前缓存和其他缓存行中，且其值与RAM中的值相同
* **无效Invalid**：缓存行未使用（即，不包含任何RAM位置）

![MESI States Diagram. *© Source: University of Washington via courses.cs.washington.edu.* MESI 状态图。*© 来源：华盛顿大学，来自courses.cs.washington.edu。*](../../img/mt-perf/MESI_Cache_Diagram.jpg){#fig:MESI width=60%}

When fetched from memory, each cache line has one of the states encoded into its tag. Then the cache line state keeps transiting from one state to another.[^25] In reality, CPU vendors usually implement slightly improved variants of MESI. For example, Intel uses [MESIF](https://en.wikipedia.org/wiki/MESIF_protocol),[^26] which adds a Forwarding (F) state, while AMD employs [MOESI](https://en.wikipedia.org/wiki/MOESI_protocol),[^27] which adds the Owning (O) state. However, these protocols still maintain the essence of the base MESI protocol.

从内存中读取时，每个缓存行的标签中都编码了上述四种状态之一。然后，缓存行状态会不断地在不同状态之间转换[^25]。实际上，CPU厂商通常会实现MESI协议的改进版本。例如，Intel使用[MESIF](https://en.wikipedia.org/wiki/MESIF_protocol)[^26]，它增加了一个转发(F: Forwarding)状态；而AMD使用 [MOESI](https://en.wikipedia.org/wiki/MOESI_protocol)[^27]，它增加了一个拥有(O: Owning)状态。然而，这些协议仍然保留了MESI协议的基本特性。

Lack of cache coherency can cause sequentially inconsistent programs. This problem can be mitigated by having _snoop_ caches watch all memory transactions and cooperate with each other to maintain memory consistency. Unfortunately, it comes with a cost since modification done by one core invalidates the corresponding cache line in another core's cache. This causes memory stalls and wastes system bandwidth. In contrast to serialization and locking issues, which can only put a ceiling on the performance of the application, coherency issues can cause retrograde effects as attributed by USL in [@sec:secAmdahl]. Two widely known types of coherency problems are *true sharing* and *false sharing*, which we will explore next.

缺乏缓存一致性会导致顺序不一致的程序。这个问题可以通过让“*监测（snoop）*缓存”监视所有内存事务并相互协作来维护内存一致性。但这样做是有代价的，因为一个核心的修改会使另一个核心缓存中对应的缓存行失效。这会导致内存停顿并浪费系统带宽。与串行序列化和锁定问题（它们只能限制应用程序的性能）不同，一致性问题会导致逆行效应(retrograde effects)，正如USL在 [@sec:secAmdahl] 中所述。两种广为人知的一致性问题是*真共享（true sharing）*和*假共享（false sharing）*，我们将在下文中进行探讨。

### True Sharing 真共享 {#sec:secTrueSharing}

True sharing occurs when two different cores access the same variable (see [@lst:TrueSharing]).

当两个不同的核心访问同一个变量时，就会发生真共享（参见 [@lst:TrueSharing]）。

Listing: True Sharing Example.
代码列表：真共享示例。

~~~~ {#lst:TrueSharing .cpp}
unsigned int sum; // shared between all threads
{ // code executed by thread A      │ { // code executed by thread B
  for (int i = 0; i < N; i++)       │   for (int i = 0; i < N; i++)
    sum += a[i];                    │     sum += b[i];
}                                   │ }
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

First of all, we have a bigger problem here besides true sharing. We actually have a *data race*, which sometimes can be quite tricky to detect. Notice, that we don't have a proper synchronization mechanism in place, which can lead to unpredictable or incorrect program behavior, because the operations on the shared data might interfere with one another. Fortunately, there are tools that can help identify such issues. [Thread sanitizer](https://clang.llvm.org/docs/ThreadSanitizer.html)[^30] from Clang and [helgrind](https://www.valgrind.org/docs/manual/hg-manual.html)[^31] are among such tools. To prevent the data race in [@lst:TrueSharing], you should declare the `sum` variable as `std::atomic<unsigned int> sum`.

首先，除了真正的数据共享之外，我们还有一个更大的问题。实际上，我们遇到了*数据竞争（data race）*，这有时很难检测。请注意，我们没有合适的同步机制，这会导致程序行为不可预测或错误，因为对共享数据的操作可能会相互干扰。幸运的是，有一些工具可以帮助我们识别这类问题。Clang的线程清理器[Thread sanitizer](https://clang.llvm.org/docs/ThreadSanitizer.html)[^30] 和开源valgrind项目中的helpgrind[helgrind](https://www.valgrind.org/docs/manual/hg-manual.html)[^31] 就是其中的一些工具。为了防止 [@lst:TrueSharing] 中出现数据竞争，您应该将 `sum` 变量声明为 `std::atomic<unsigned int> sum`。

Using C++ atomics can help to solve data races when true sharing happens. However, it effectively serializes accesses to the atomic variable, which may hurt performance. A better way of solving our true sharing issue is by using Thread Local Storage (TLS). TLS is the method by which each thread in a given multithreaded process can allocate memory to store thread-specific data. By doing so, threads modify their local copies instead of contending for a globally available memory location. The example in [@lst:TrueSharing] can be fixed by declaring `sum` with a TLS class specifier: `thread_local unsigned int sum` (since C++11). The main thread should then incorporate results from all the local copies of each worker thread.

使用C++原子描述可以帮助解决真正数据共享时出现的数据竞争问题。然而，它实际上会串行化对原子变量的访问，这可能会降低性能。解决真共享问题的更好方法是使用线程本地存储(TLS: Thread Local Storage)。TLS是一种允许给定多线程进程中的每个线程分配内存来存储线程特定数据的方法。通过这种方式，线程可以修改其本地副本，而不是争用全局可用的内存位置。[@lst:TrueSharing] 中的示例可以通过使用TLS类说明符声明 `sum` 来修复：`thread_local unsigned int sum`（自C++11起）。然后，主线程应该合并来自每个工作线程所有本地副本的结果。

### False Sharing 伪共享 {#sec:secFalseSharing}

If not careful, you may attempt to solve the true sharing issue as shown in [@lst:FalseSharing]. This solution introduces another problem: *false sharing*. It occurs when two different cores modify different variables that happen to reside on the same cache line. In the code sample shown in [@lst:FalseSharing], even though threads `A` and `B` update different fields of struct `S`, they are very likely to reside on the same cache line, which will trigger a false sharing issue. Figure @fig:FalseSharing illustrates this problem.

如果不小心，您可能会尝试使用 [@lst:FalseSharing] 中所示的方法来解决真共享问题。这种解决方案会引入另一个问题：*伪共享（false sharing）*。当两个不同的核心修改恰好位于同一缓存行上的不同变量时，就会发生伪共享。在 [@lst:FalseSharing] 所示的代码示例中，即使线程 `A` 和 `B` 更新的是结构体 `S` 的不同字段，它们也极有可能位于同一缓存行，从而触发伪共享问题。图 @fig:FalseSharing 展示了这个问题。

Listing: False Sharing Example.
代码列表：伪共享示例

~~~~ {#lst:FalseSharing .cpp}
struct S {
  int sumA; // sumA and sumB are likely to
  int sumB; // reside in the same cache line
};
S s;

{ // code executed by thread A     │     { // code executed by thread B
  for (int i = 0; i < N; i++)      │       for (int i = 0; i < N; i++)
    s.sumA += a[i];                │         s.sumB += b[i];
}                                  │     }
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

![False Sharing: two threads access the same cache line. 伪共享：两个线程访问同一缓存行。](../../img/mt-perf/FalseSharing.jpg){#fig:FalseSharing width=60%}

False sharing is a frequent source of performance issues for multithreaded applications. Because of that, modern analysis tools have built-in support for detecting such cases. For applications that experience true/false sharing, TMA will likely show a high  &rarr; `L3 Bound` &rarr; `Contested Accesses` metric.[^18]

对于多线程应用程序而言，伪共享是导致性能问题的常见原因。因此，现代分析工具都内置了对此类情况的检测支持。对于存在真/假共享的应用程序，TMA分析工具通常会显示较高的“`内存瓶颈Memory Bound`” --> “`L3瓶颈L3 Bound`”和“`竞争访问Contested Accesses`”指标[^18]。

When using Intel VTune Profiler, I recommend running two types of analysis to find and eliminate false sharing issues. First, run a *Microarchitecture Exploration* analysis that implements TMA methodology to detect the presence of false sharing in an application. As noted before, the high value for the *Contested Accesses* metric prompts us to dig deeper and run the *Memory Access* analysis with the *Analyze dynamic memory objects* checkbox enabled. This analysis helps in finding out memory accesses to the data structure that caused contention issues. Typically, such memory accesses have high latency, which will be revealed by the analysis. See an example of using Intel VTune Profiler for fixing false sharing issues in [Intel Developer Zone](https://software.intel.com/en-us/vtune-cookbook-false-sharing).[^20]

在使用Intel VTune Profiler时，我建议运行两种类型的分析来查找并消除伪共享问题。首先，运行“*微体系结构探索Microarchitecture Exploration*”分析，该分析采用TMA方法来检测应用程序中是否存在伪共享。如前所述，*竞争访问（Contested Accesses）*指标的高值提示我们需要深入挖掘，并运行 *内存访问（Memory Access）* 分析，同时启用 *分析动态内存对象（Analyze dynamic memory objects）* 复选框。此分析有助于找出导致争用问题的数据结构的存储访问。通常，此类存储访问具有较高的延迟，分析结果会揭示这一点。请参阅[Intel Developer Zone](https://software.intel.com/en-us/vtune-cookbook-false-sharing)[^20]中的示例，了解如何使用Intel VTune Profiler修复伪共享问题。

Linux `perf` has support for finding false sharing as well. As with the Intel VTune profiler, run TMA first (see [@sec:secTMA_Intel]) to find out if the program experiences false/true sharing issues. If that's the case, use the `perf c2c` tool to detect memory accesses with high cache coherency costs. `perf c2c` matches store/load addresses for different threads and checks if the hit in a modified cache line occurred. Readers can find a detailed explanation of the process and how to use the tool in a dedicated [blog post](https://joemario.github.io/blog/2016/09/01/c2c-blog/).[^21]

Linux的 `perf` 工具也支持查找伪共享。与Intel VTune Profiler类似，首先运行TMA（参见 [@sec:secTMA_Intel]）以确定程序是否存在伪共享/真共享问题。如果存在伪共享问题，则使用 `perf c2c` 工具检测缓存一致性开销高的内存访问。`perf c2c` 会匹配不同线程的存储/加载地址，并检查是否命中了已修改的缓存行。读者可以在专门的[博客文章](https://joemario.github.io/blog/2016/09/01/c2c-blog/)[^21]中找到关于该过程以及如何使用该工具的详细说明。

It is possible to eliminate false sharing with the help of aligning/padding memory objects. Example in [@sec:secFalseSharing] can be fixed by ensuring `sumA` and `sumB` do not share the same cache line as shown in [@lst:PadFalseSharing].[^32]

通过对齐/填充内存对象，可以消除伪共享。例如，[@sec:secFalseSharing] 中的问题可以通过确保 `sumA` 和 `sumB` 不共享同一缓存行来解决，如 [@lst:PadFalseSharing] 所示[^32]。

Listing: Data padding to avoid false sharing.
代码列表：数据填充以避免伪共享。

~~~~ {#lst:PadFalseSharing .cpp}
                              constexpr int CacheLineAlign = 64;
struct S {                    struct S {
  int sumA;        =>           int sumA; 
  int sumB;                     alignas(CacheLineAlign) int sumB;
};                            };
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

False sharing can not only be observed in native languages, like C and C++, but also in managed ones, like Java and C#. From a general performance perspective, the most important thing to consider is the cost of the possible state transitions. Of all cache states, the only ones that do not involve a costly cross-cache subsystem communication and data transfer during CPU read/write operations are the Modified (M) and Exclusive (E) states. Thus, the longer the cache line maintains the `M` or `E` states (i.e., the less sharing of data across caches), the lower the coherence cost incurred by a multithreaded application. An example demonstrating how this property has been employed can be found in Nitsan Wakart's blog post "[Diving Deeper into Cache Coherency](http://psy-lob-saw.blogspot.com/2013/09/diving-deeper-into-cache-coherency.html)".[^28]


伪共享不仅存在于C和C++等原生语言中，也存在于Java和C#等托管语言中。从整体通用性能角度来看，最需要考虑的是状态转换的成本。在所有缓存状态中，只有修改(M)状态和独占(E)状态在CPU读/写操作期间不涉及代价高昂的跨缓存子系统通信和数据传输。因此，缓存行保持 `M` 或 `E` 状态的时间越长（即：跨缓存的数据共享越少），多线程应用程序的一致性成本就越低。Nitsan Wakart的博客文章“深入探索缓存一致性”(http://psy-lob-saw.blogspot.com/2013/09/diving-deeper-into-cache-coherency.html)[28]提供了一个示例，展示了如何应用此属性。

[^18]: See the Intel VTune user guide for a description of the *Contested Accesses* metric. 有关“*竞争访问（Contested Accesses）*”指标的说明，请参阅Intel VTune用户指南。
[^20]: VTune cookbook: false-sharing - [https://software.intel.com/en-us/vtune-cookbook-false-sharing](https://software.intel.com/en-us/vtune-cookbook-false-sharing).
[^21]: An article on `perf c2c` 一篇关于 `perf c2c` 的文章 - [https://joemario.github.io/blog/2016/09/01/c2c-blog/](https://joemario.github.io/blog/2016/09/01/c2c-blog/).
[^25]: There is an animated demonstration of the MESI protocol MESI协议的动画演示- [https://www.scss.tcd.ie/Jeremy.Jones/vivio/caches/MESI.htm](https://www.scss.tcd.ie/Jeremy.Jones/vivio/caches/MESI.htm).
[^26]: MESIF - [https://en.wikipedia.org/wiki/MESIF_protocol](https://en.wikipedia.org/wiki/MESIF_protocol)
[^27]: MOESI - [https://en.wikipedia.org/wiki/MOESI_protocol](https://en.wikipedia.org/wiki/MOESI_protocol)
[^28]: Blog post "Diving Deeper into Cache Coherency" 博客文章“深入探讨缓存一致性” - [http://psy-lob-saw.blogspot.com/2013/09/diving-deeper-into-cache-coherency.html](http://psy-lob-saw.blogspot.com/2013/09/diving-deeper-into-cache-coherency.html)
[^30]: Clang's thread sanitizer tool: Clang的线程清理工具： [https://clang.llvm.org/docs/ThreadSanitizer.html](https://clang.llvm.org/docs/ThreadSanitizer.html).
[^31]: Helgrind, a thread error detector tool: Helgrind，一个线程错误检测工具： [https://www.valgrind.org/docs/manual/hg-manual.html](https://www.valgrind.org/docs/manual/hg-manual.html).
[^32]: Do not take the size of a cache line as a constant value. For example, in Apple processors such as M1, M2, and later, the L2 cache operates on 128B cache lines. 不要将缓存行的大小视为常量值。例如，在苹果的M1、M2及更高版本的处理器中，L2缓存以128字节的缓存行进行操作。
