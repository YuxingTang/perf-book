## Explicit Memory Prefetching 显式内存预取 {#sec:memPrefetch}

By now, you should know that memory accesses not resolved from caches are often very expensive. Modern CPUs try very hard to lower the penalty of cache misses by predicting which memory locations a program will access in the future and prefetch them ahead of time. If the requested memory location is not in the cache at the time a program demands it, then we will suffer the cache miss penalty as we have to go to the DRAM and fetch the data anyway. But if the CPU manages to bring that memory location into caches in time or if the request was predicted and data is underway, then the penalty of a cache miss will be much lower.

现在你应该知道，无法从缓存中解析的内存访问通常代价很高。现代CPU会尽力降低缓存未命中的代价，方法是预测程序未来将要访问的内存位置并提前预取。如果程序请求的内存位置在请求时不在缓存中，那么我们仍然会遭受缓存未命中的代价，因为我们必须访问 DRAM 并获取数据。但是，如果 CPU 能够及时将该内存位置加载到缓存中，或者请求已被预测且数据正在传输中，那么缓存未命中的代价就会低得多。

Modern CPUs have two automatic mechanisms for solving that problem: hardware prefetching and OOO execution. Hardware prefetchers help to hide the memory access latency by initiating prefetching requests on repetitive memory access patterns. An OOO engine looks `N` instructions into the future and issues loads early to enable the smooth execution of future instructions that will demand this data.

现代CPU有两种自动机制来解决这个问题：硬件预取和OOO执行。硬件预取器通过在重复的内存访问模式上发起预取请求来帮助隐藏内存访问延迟。OOO引擎会预测未来 `N` 条指令，并提前发出加载请求，以确保未来需要这些数据的指令能够顺利执行。

Hardware prefetchers fail when data access patterns are too complicated to predict. And there is nothing software developers can do about it as we cannot control the behavior of this unit. On the other hand, an OOO engine does not try to predict memory locations that will be needed in the future as hardware prefetching does. So, the only measure of success for it is how much latency it was able to hide by scheduling the load in advance.

当数据访问模式过于复杂而无法预测时，硬件预取器就会失效。软件开发人员对此也无能为力，因为我们无法控制硬件预取器的行为。另一方面，一个乱序执行OOO引擎不会像硬件预取那样尝试预测未来需要的内存位置。因此，衡量其成功与否的唯一标准是它通过提前调度加载操作隐藏了多少延迟。

Consider a small snippet of code in [@lst:MemPrefetch1], where `arr` is an array of one million integers. The index `idx`, which is assigned to a random value, is immediately used to access a location in `arr`, which almost certainly misses in caches as it is random. A hardware prefetcher can't predict it since every time the load goes to a completely new place in memory. The interval from the time the address of a memory location is known (returned from the function `random_distribution`) until the value of that memory location is demanded (call to `doSomeExtensiveComputation`) is called *prefetching window*. In this example, the OOO engine doesn't have the opportunity to issue the load early since the prefetching window is very small. This leads to the latency of the memory access `arr[idx]` to stand on a critical path while executing the loop as shown in Figure @fig:SWmemprefetch1. The program waits for the value to come back (the hatched fill rectangle in the diagram) without making forward progress.

考虑以下代码片段：[@lst:MemPrefetch1]，其中 `arr` 是一个包含一百万个整数的数组。索引 `idx` 被赋予一个随机值，并立即用于访问 `arr` 中的一个位置，由于它是随机的，因此几乎肯定会在缓存中未命中。硬件预取器无法预测这种情况，因为每次加载都会访问内存中一个全新的位置。从知道内存位置的地址（由函数 `random_distribution` 返回）到需要该内存位置的值（调用 `doSomeExtensiveComputation`）之间的时间间隔称为*预取窗口（prefetch window）*。在本例中，由于预取窗口非常小，OOO引擎没有机会提前发出加载请求。这导致内存访问 `arr[idx]` 的延迟出现在循环执行的关键路径上，如图 @fig:SWmemprefetch1 所示。程序会等待返回值（图中带阴影的矩形区域），而无法继续执行后续操作。

You're probably thinking: "But the next iteration of the loop should start executing speculatively in parallel". That's true, and indeed, it is reflected in Figure @fig:SWmemprefetch1. The `doSomeExtensiveComputation` function requires a lot of work, and when execution gets closer to the finish of the first iteration, a CPU speculatively starts executing instructions from the next iteration. It creates a positive overlap in the execution between iterations. In fact, we presented an optimistic scenario where a processor was able to generate the next random number and issue a load in parallel with the previous iteration of the loop. However, a CPU wasn't able to fully hide the latency of the load, because it cannot look that far ahead of the current iteration to issue the load early enough. Maybe future processors will have more powerful OOO engines, but for now, there are cases where a programmer's intervention is needed.

您可能会想：“但是循环的下一次迭代应该会并行地开始前瞻性执行。”没错，图 @fig:SWmemprefetch1 也确实反映了这一点。 `doSomeExtensiveComputation` 函数需要大量的计算，当程序接近第一次迭代结束时，CPU会前瞻地开始执行下一次迭代的指令。这会在迭代之间造成执行时间的重叠。实际上，我们设想了一种理想情况：处理器能够生成下一个随机数，并在循环的上一次迭代中并行执行加载操作。然而，CPU无法完全隐藏加载操作的延迟，因为它无法提前足够长的时间预测当前迭代之后的指令，从而提前执行加载操作。或许未来的处理器会拥有更强大的乱序执行OOO引擎，但就目前而言，在某些情况下，程序员的干预是必要的。

Listing: A random number is an index for a subsequent load.
代码列表：随机数是后续加载load操作的索引。

~~~~ {#lst:MemPrefetch1 .cpp}
for (int i = 0; i < N; ++i) {
  size_t idx = random_distribution(generator);
  int x = arr[idx]; // cache miss
  doSomeExtensiveComputation(x);
}
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

![Execution timeline that shows the load latency standing on a critical path. 显示加载延迟的关键路径执行时间线。](../../img/memory-access-opts/SWmemprefetch1.png){#fig:SWmemprefetch1 width=80%}

Luckily, it's not a dead end as there is a way to speed up this code by fully overlapping the load with the execution of `doSomeExtensiveComputation`, which will hide the latency of a cache miss. We can achieve this with techniques called *software pipelining* and *explicit memory prefetching*. Implementation of this idea is shown in [@lst:MemPrefetch2]. We pipeline generation of random numbers and start prefetching memory location for the next iteration in parallel with `doSomeExtensiveComputation`.

幸运的是，这并非死路一条，因为我们可以通过将加载操作与 `doSomeExtensiveComputation` 的执行完全重叠来加速这段代码，从而隐藏缓存未命中的延迟。我们可以使用称为“软件流水线”和“显式内存预取”的技术来实现这一点。此思想的实现如 [@lst:MemPrefetch2] 所示。我们对随机数的生成进行流水线处理，并与 `doSomeExtensiveComputation` 并行地预取下一次迭代所需的内存位置。

Listing: Utilizing Explicit Software Memory Prefetching hints.
代码列表：利用显式软件内存预取提示。

~~~~ {#lst:MemPrefetch2 .cpp}
size_t idx = random_distribution(generator);
for (int i = 0; i < N; ++i) {
  int x = arr[idx]; 
  idx = random_distribution(generator);
  // prefetch the element for the next iteration
  __builtin_prefetch(&arr[idx]);
  doSomeExtensiveComputation(x);
}
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

A graphical illustration of this transformation is shown in Figure @fig:SWmemprefetch2. We utilized software pipelining to generate random numbers for the next iteration. In other words, on iteration `M`, we produce a random number that will be consumed on iteration `M+1`. This enables us to issue the memory request early since we already know the next index in the array. This transformation makes our prefetching window much larger and fully hides the latency of a cache miss. On the iteration `M+1`, the actual load has a very high chance to hit caches, because it was prefetched on iteration `M`.

图 @fig:SWmemprefetch2 以图形方式展示了这种转换。我们利用软件流水线技术为下一次迭代生成随机数。换句话说，在迭代 `M` 时，我们生成一个随机数，该随机数将在迭代 `M+1` 时使用。由于我们已经知道数组中的下一个索引，因此可以提前发出内存请求。这种转换极大地增大了预取窗口，并完全隐藏了缓存未命中的延迟。在迭代 `M+1` 时，实际加载操作有很高的概率命中缓存，因为它在迭代 `M` 时已经进行了预取。

![Hiding the cache miss latency by overlapping it with other execution. 通过与其他执行重叠来隐藏缓存未命中延迟。](../../img/memory-access-opts/SWmemprefetch2.png){#fig:SWmemprefetch2 width=80%}

Notice the usage of [`__builtin_prefetch`](https://gcc.gnu.org/onlinedocs/gcc/Other-Builtins.html),[^4] a special hint that developers can use to explicitly request a CPU to prefetch a certain memory location. Another option is to use compiler intrinsics. On x86 platforms there is `_mm_prefetch` intrinsic, on ARM platforms there is `__pld` intrinsic. Compilers will generate the `PREFETCH` instruction for x86 and the `pld` instruction for ARM.

请注意[`__builtin_prefetch`](https://gcc.gnu.org/onlinedocs/gcc/Other-Builtins.html)[^4],的用法，这是一个特殊的提示，开发者可以使用它来显式地请求CPU预取特定的内存位置。另一种选择是使用编译器内部函数。在x86平台上，有`_mm_prefetch`内部函数；在ARM平台上，有`__pld`内部函数。编译器会为x86生成`PREFETCH`指令，为ARM生成`pld`指令。

There are situations when software memory prefetching is not possible. For example, when traversing a linked list, the prefetching window is tiny and it is not possible to hide the latency of pointer chasing.

在某些情况下，软件内存预取是行不通的。例如，在遍历链表时，预取窗口非常小，无法隐藏指针追踪的延迟。

In [@lst:MemPrefetch2] we saw an example of prefetching for the next iteration, but also you may frequently encounter a need to prefetch for 2, 4, 8, and sometimes even more iterations. The code in [@lst:MemPrefetch3] is one of those cases when it could be beneficial. It presents a typical code for populating a graph with edges. If the graph is very sparse and has a lot of vertices, it is very likely that accesses to `this->out_neighbors` and `this->in_neighbors` vectors will frequently miss in caches. This happens because every edge is likely to connect new vertices that are not currently in caches.

在 [@lst:MemPrefetch2] 中，我们看到了一个为下一次迭代预取数据的示例，但您可能经常需要预取2次、4次、8次甚至更多次迭代的数据。[@lst:MemPrefetch3] 中的代码就是一个可以应用预取的例子。它展示了一个典型的为图添加边的代码。如果图非常稀疏且顶点数量很多，那么对 `this->out_neighbors` 和 `this->in_neighbors` 向量的访问很可能会频繁地出现缓存未命中的情况。这是因为每条边都可能连接当前不在缓存中的新顶点。

This code is different from the previous example as there are no extensive computations on every iteration, so the penalty of cache misses likely dominates the latency of each iteration. But we can leverage the fact that we know all the elements that will be accessed in the future. The elements of vector `edges` are accessed sequentially and thus are likely to be timely brought to the L1 cache by the hardware prefetcher. Our goal here is to overlap the latency of a cache miss with executing enough iterations to completely hide it.

这段代码与之前的示例不同，因为每次迭代都没有大量的计算，所以缓存未命中造成的延迟很可能远大于每次迭代的延迟。但我们可以利用已知所有未来将要访问的元素这一事实。向量 `edges` 的元素是顺序访问的，因此硬件预取器很可能会及时将它们加载到L1缓存中。我们的目标是通过执行足够多的迭代来抵消缓存未命中的延迟，从而完全隐藏缓存未命中的影响。

As a general rule, for a prefetch hint to be effective, it must be inserted well ahead of time so that by the time the loaded value is used in other calculations, it will be already in the cache. However, it also shouldn't be inserted too early since it may pollute the cache with data that is not used for a long time. Notice, in [@lst:MemPrefetch3], `lookAhead` is a template parameter, which enables a programmer to try different values and see which gives the best performance. More advanced users can try to estimate the prefetching window using the method described in [@sec:timed_lbr]; an example of using this method can be found on the Easyperf blog.[^5]

一般来说，预取提示要想有效，必须提前插入，以便在加载的值被用于其他计算时，它已经存在于缓存中。但是，预取提示也不应该插入得太早，因为它可能会用长时间未使用的数据污染缓存。请注意，在 [@lst:MemPrefetch3] 中，`lookAhead` 是一个模板参数，程序员可以使用它来尝试不同的值，并找出性能最佳的值。更高级的用户可以尝试使用 [@sec:timed_lbr] 中描述的方法来估算预取窗口；Easyperf博客上提供了一个使用此方法的示例[^5]。

Listing: Example of a software prefetching for the next 8 iterations.
代码列表：针对接下来8次迭代的软件预取示例。

~~~~ {#lst:MemPrefetch3 .cpp}
template <int lookAhead = 8>
void Graph::update(const std::vector<Edge>& edges) {
  for(int i = 0; i + lookAhead < edges.size(); i++) {
    VertexID v = edges[i].from;
    VertexID u = edges[i].to;
    this->out_neighbors[u].push_back(v);
    this->in_neighbors[v].push_back(u);

    // prefetch elements for future iterations
    VertexID v_next = edges[i + lookAhead].from;
    VertexID u_next = edges[i + lookAhead].to;
    __builtin_prefetch(this->out_neighbors.data() + v_next);
    __builtin_prefetch(this->in_neighbors.data()  + u_next);
  }
  // process the remainder of the vector `edges` ...
}
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Explicit memory prefetching is most frequently used in loops, but also you can insert those hints into a parent function; again, it all depends on the available prefetching window.

显式内存预取最常用于循环中，但你也可以将这些提示插入到父函数中；同样，这完全取决于可用的预取窗口。

This technique is a powerful weapon, however, it should be used with extreme care as it is not easy to get it right. First of all, explicit memory prefetching is not portable, meaning that if it gives performance gains on one platform, it doesn't guarantee similar speedups on another platform. It is very implementation-specific and platforms are not required to honor those hints. In such a case it will likely degrade performance. My recommendation would be to verify that the impact is positive with all available tools. Not only check the performance numbers but also make sure that the number of cache misses (L3 in particular) went down. Once the change is committed into the code base, monitor performance on all the platforms that you run your application on, as it could be very sensitive to changes in the surrounding code. Consider dropping the idea if the benefits do not outweigh the potential maintenance burden.

这项技术非常强大，但必须极其谨慎地使用，因为它并不容易正确应用。首先，显式内存预取不具备可移植性，这意味着即使它在一个平台上带来了性能提升，也不能保证在其他平台上也能获得类似的加速。它与具体的实现密切相关，而且平台并非必须遵循这些提示。在这种情况下，它很可能会降低性能。我的建议是使用所有可用的工具来验证其影响是否为正面。不仅要检查性能数据，还要确保缓存未命中次数（尤其是L3缓存）有所减少。一旦更改提交到代码库，请监控应用程序运行的所有平台上的性能，因为它可能对周围代码的更改非常敏感。如果收益不足以抵消潜在的维护负担，请考虑放弃这个想法。

For some complicated scenarios, make sure that the code prefetches the right memory locations. It can get tricky when a current iteration of a loop depends on the previous iteration, e.g., there is a `continue` statement or changing the next element to be processed is guarded by an `if` condition. In this case, my recommendation is to instrument the code to test the accuracy of your prefetching hints. Because when used badly, it can degrade the performance of caches by evicting other useful data.

对于某些复杂场景，请确保代码预取正确的内存位置。当循环的当前迭代依赖于前一次迭代时，情况会变得棘手，例如，存在 `continue` 语句，或者要处理的下一个元素由 `if` 条件语句控制。在这种情况下，我建议对代码进行插桩，以测试预取提示的准确性。因为如果使用不当，预取提示会通过驱逐其他有用数据来降低缓存性能。

Finally, explicit prefetching increases code size and adds pressure on the CPU Frontend. A prefetch hint is just a fake load that goes into the memory subsystem but does not have a destination register. And just like any other instruction, it consumes CPU resources. Apply it with extreme care, because when used wrong, it can pessimize the performance of a program.

最后，显式预取会增加代码大小并增加CPU前端的压力。预取提示只是一个虚拟加载操作，它会进入内存子系统，但没有目标寄存器。与其他任何指令一样，它也会消耗CPU资源。因此，务必谨慎使用预取提示，因为如果使用不当，它会降低程序的性能。

[^4]: GCC builtins GCC编译器的内置函数 - [https://gcc.gnu.org/onlinedocs/gcc/Other-Builtins.html](https://gcc.gnu.org/onlinedocs/gcc/Other-Builtins.html).
[^5]: "Precise timing of machine code with Linux perf" “用Linux perf对机器代码进去计时” - [https://easyperf.net/blog/2019/04/03/Precise-timing-of-machine-code-with-Linux-perf#application-estimating-prefetch-window](https://easyperf.net/blog/2019/04/03/Precise-timing-of-machine-code-with-Linux-perf#application-estimating-prefetch-window).
