## Microarchitecture-Specific Performance Issues 微体系结构特定性能问题 {#sec:UarchSpecificIssues}

In this section, we will discuss some common microarchitecture-specific issues that affect the majority of modern processors. I call them microarchitecture-specific because they are caused by the way a particular microarchitecture feature is implemented. These issues are very specific and do not frequently appear as a major performance bottleneck. Typically, they are diluted among other more significant performance problems. Thus, these microarchitecture-specific performance issues are considered corner cases and are less known than the other issues that we already discussed in the book. Nevertheless, they can cause very undesirable performance penalties. Note that the impact of a particular problem can be more/less pronounced on one platform than another. Also, keep in mind that the list of microarchitecture-specific issues covered below is not exhaustive.

在本节中，我们将讨论一些影响大多数现代处理器的常见微体系结构特定问题。之所以称之为微体系结构特定问题，是因为它们是由特定微体系结构特性的实现方式引起的。这些问题非常特殊，通常不会成为主要的性能瓶颈。它们往往被其他更重要的性能问题所掩盖。因此，这些微体系结构特定性能问题被认为是特殊情况，不如本书中已讨论的其他问题那样为人所知。尽管如此，它们仍然会造成非常不利的性能损失。请注意，特定问题的影响在不同的平台上可能有所不同。同时，请记住，下面列出的微体系结构特定问题并非全部。

### Memory Order Violations 存储序违例 {#sec:MemoryOrderViolations}

I introduced the concept of memory ordering in [@sec:uarchLSU]. Memory reordering is a crucial aspect of modern CPUs, as it enables them to execute memory requests in parallel and out-of-order. The key element in load/store reordering is memory disambiguation, which predicts if it is safe to let loads go ahead of earlier stores. Since memory disambiguation is speculative, it can lead to performance issues if not handled properly.

我在 [@sec:uarchLSU] 中引入了内存顺序/存储序的概念。内存重排序是现代CPU的一个关键特性，它使CPU能够并行且乱序地执行内存请求。加载/存储重排序的关键在于内存消歧（memory disambiguation），它预测是否可以安全地将加载操作置于先前的存储操作之前。由于内存消歧是前瞻性的，因此如果处理不当，可能会导致性能问题。

Consider an example in [@lst:MemOrderViolation], on the left. This code snippet calculates a histogram of an 8-bit grayscale image, i.e., how many times a certain color appears in the image. Besides countless other places, this code can be found in Otsu's thresholding algorithm[^1] which is used to convert a grayscale image to a binary image. Since the input image is 8-bit grayscale, there are only 256 different colors.

请看 [@lst:MemOrderViolation] 左侧的示例。这段代码计算了8位灰度图像的直方图，即：图像中某种颜色出现的次数。除了其他许多地方，这段代码还可以在Otsu阈值算法[^1]中找到，该算法用于将灰度图像转换为二值图像。由于输入图像是8位灰度图像，因此只有256种不同的颜色。

For each pixel on an image, you need to read the current histogram count of the color of the pixel, increment it, and store it back. This is a classic read-modify-write dependency through the memory. Imagine we have the following consecutive pixels in the image: `pixels = [0xFF,0xFF,0x00,0xFF,...]` and so on. The loaded value of the histogram count for `pixels[1]` comes from the result of the previous iteration (`pixels[0]`). The histogram count for `pixels[2]` comes from memory; it is independent and can be reordered. But then again, the histogram count for `pixels[3]` is dependent on the result of processing `pixels[1]`, and so on. Iterations 0, 1, and 3 are dependent and cannot be reordered.

对于图像上的每个像素，需要读取该像素颜色的当前直方图计数，将其加一，然后存储回去。这是一个典型的内存读写依赖问题。假设图像中有以下连续像素：`pixels = [0xFF,0xFF,0x00,0xFF,...]`，以此类推。`pixels[1]` 的直方图计数值来自前一次迭代的结果（`pixels[0]`）。`pixels[2]` 的直方图计数来自内存；它是独立的，可以重新排序。但是，`pixels[3]` 的直方图计数又依赖于处理 `pixels[1]` 的结果，以此类推。迭代0、1和3相互依赖，不能重新排序。

Listing: Memory Order Violation Example.
代码列表：存储序违例示例

~~~~ {#lst:MemOrderViolation .cpp}
std::array<uint32_t, 256> hist;           std::array<uint32_t, 256> hist1;
hist.fill(0);                             std::array<uint32_t, 256> hist2;
int N = width * height;                   hist1.fill(0);         
for (int i = 0; i < N; ++i)       =>      hist2.fill(0);
  hist[image[i]]++;                       int N = width * height;         
                                          int i = 0;
                                          for (; i + 1 < N; i += 2) {
                                            hist1[image[i+0]]++;
                                            hist2[image[i+1]]++;
                                          }
                                          // remainder
                                          for (; i < N; ++i)
                                            hist1[image[i]]++;
                                          // combine partial histograms
                                          for (int i = 0; i < hist1.size(); ++i)
                                            hist1[i] += hist2[i];
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Recall from [@sec:uarchLSU] that the processor doesn't necessarily know about a potential store-to-load forwarding, so it has to make a prediction. If it correctly predicts a memory order violation between two updates of color `0xFF`, then these accesses will be serialized. The performance will not be great, but it is the best we could hope for with the initial code. On the contrary, if the processor predicts that there is no memory order violation, it will speculatively let the two updates run in parallel. Later it will recognize the mistake, flush the pipeline, and re-execute the youngest of the two updates. This is very hurtful for performance.

回顾[@sec:uarchLSU]的说明，处理器未必知道潜在的存储store到加载load转发，因此它必须进行预测。如果它正确预测了两次颜色为`0xFF`的更新之间存在内存顺序违例，那么这些访问将被串行化。性能不会很好，但对于初始代码来说，这已经是最佳结果了。相反，如果处理器预测不存在内存顺序违例，它会前瞻性地允许两次更新并行执行。之后它会识别出错误，清空流水线，并重新执行两次更新中较新的那次。这会严重影响性能。

Performance will greatly depend on the color patterns of the input image. Images with long sequences of pixels with the same color will have worse performance than images where colors don't repeat often. The performance of the initial version will be good as long as the distance between two pixels of the same color is long enough. The phrase "long enough" in this context is determined by the size of the out-of-order instruction window. Repeating read-modify-writes of the same color may trigger ordering violations if they occur within a few loop iterations of each other, but not if they occur more than a hundred loop iterations apart.

性能很大程度上取决于输入图像的颜色模式。像素重复次数较多的图像性能会比颜色不经常重复的图像差。只要两个相同颜色像素之间的距离足够长，初始版本的性能就会很好。这里的“足够长”指的是乱序指令窗口的大小。如果对相同颜色的数据进行重复的读-修改-写操作在相隔几个循环迭代的时间内发生，则可能触发顺序违例；但如果相隔超过一百个循环迭代，则不会触发。

A cure for the memory order violation problem is shown in [@lst:MemOrderViolation], on the right. As you can see, I duplicated the histogram, and now the processing of pixels alternates between two partial histograms. In the end, we combine the two partial histograms to get a final result. This new version with two partial histograms is still prone to potentially problematic patterns, such as `0xFF 0x00 0xFF 0x00 0xFF ...` However, with this change, the original worst-case scenario (e.g., `0xFF 0xFF 0xFF ...`) will run twice as fast as before. It may be beneficial to create four or eight partial histograms depending on the color pattern of input images. This exact code is featured in the [mem_order_violation_1](https://github.com/dendibakh/perf-ninja/tree/main/labs/memory_bound/mem_order_violation_1)[^2] lab assignment of the Performance Ninja course, so feel free to experiment. 

右侧的 [@lst:MemOrderViolation] 展示了一种解决内存顺序违例问题的方案。如你所见，我复制了直方图，现在像素的处理在2个部分直方图之间交替进行。最后，我们将这两个部分直方图合并以获得最终结果。这个使用两个部分直方图的新版本仍然容易出现一些潜在的问题模式，例如 `0xFF 0x00 0xFF 0x00 0xFF ...`。但是，通过这种改进，原本最坏情况下的场景（例如 `0xFF 0xFF 0xFF ...`）的运行速度将比以前快一倍。根据输入图像的颜色模式，创建4个或8个部分直方图可能更有益。这段代码正是性能忍者（Performance Ninja）课程的存储序违例一 [mem_order_violation_1](https://github.com/dendibakh/perf-ninja/tree/main/labs/memory_bound/mem_order_violation_1)[^2]实验作业中使用的代码，欢迎大家进行实验。

On a small set of input images, I observed from 10% to 50% speedup on various platforms. It is worth mentioning that the version on the right consumes 1 KB of additional memory, which may not be huge in this case but is something to watch out for.

在一小组输入图像上，我观察到在不同平台上速度提升了10%到50%。值得一提的是，右侧的版本会额外占用1KB的内存，虽然在这个例子中可能不算多，但还是需要注意。

### Misaligned Memory Accesses 非对齐存储访问 {#sec:MisalignedMemoryAccesses}

A variable is accessed most efficiently if it is stored at a memory address that is divisible by the size of the variable. For example, an `int` requires a 4-byte alignment, meaning its address should be a multiple of 4. In C++, it is called *natural alignment*, which occurs by default for fundamental data types, such as integer, float, or double. When you declare variables of these types, the compiler ensures that they are stored in memory at addresses that are multiples of their size. In contrast, arrays, structs, and classes may require special alignment as you'll learn in this section.

如果变量存储在能够被其大小整除的内存地址上，则访问该变量的效率最高。例如，`int` 类型需要4字节对齐，这意味着它的地址必须是4的倍数。在C++中，这被称为*自然对齐（natural alignment）*，对于整数、浮点数或双精度浮点数等基本数据类型，这是默认的对齐方式。当您声明这些类型的变量时，编译器会确保它们存储在内存中，地址是其大小的倍数。相比之下，数组、结构体和类可能需要特殊的对齐方式，您将在本节中学习到。

A typical case where data alignment is important is SIMD code, where loads and stores access large chunks of data with a single operation. In most processors, the L1 cache is designed to be able to read/write data at any alignment. Generally, even if a load/store is misaligned but does not cross the cache line boundary, it won't have any performance penalty. 

数据对齐非常重要的一个典型例子是SIMD代码，其中加载load和存储store操作会通过单个操作访问大量数据。在大多数处理器中，L1缓存被设计为能够以任何对齐方式读取/写入数据。通常，即使加载/存储操作未对齐，但只要没有跨越缓存行边界，就不会造成任何性能损失。

However, when a load or store crosses the cache line boundary, such access requires two cache line reads (*split load/store*). It requires using a *split register*, which keeps the two parts and once both parts are fetched, they are combined into a single register. The number of split registers is limited. When executed sporadically, split accesses complete without any observable performance impact on overall execution. However, if that happens frequently, misaligned memory accesses will suffer delays.

但是，当加载或存储操作跨越缓存行边界时，这种访问需要两次缓存行读取（*分离式加载/存储split load/store*）。这需要使用一个*分割寄存器split register*，它将保存两个部分，并在两部分都被读取后合并到一个寄存器中。分割寄存器的数量有限。如果分割访问只是偶尔执行，则不会对整体执行性能造成任何明显影响。但是，如果这种情况频繁发生，未对齐的内存访问将会出现延迟。

A memory address is said to be *aligned* if it is a multiple of a specific size. For example, when a 16-byte object is aligned on the 64-byte boundary, the low 6 bits of its address are zero. Otherwise, when a 16-byte object crosses the 64-byte boundary, it is said to be *misaligned*. In the literature, you can also encounter the term *split load/store* to describe such a situation. Split loads/stores may incur performance penalties if many of them in a row consume all available split registers. Intel's TMA methodology tracks this with the `Memory_Bound` &rarr; `L1_Bound` &rarr; `Split Loads` metric.

如果一个内存地址是特定大小的倍数，则称该地址*已对齐aligned*。例如，当一个16字节的对象沿64字节边界对齐时，其地址的低6位为零。否则，当一个16字节的对象跨越64字节边界时，则称其*未对齐misaligned*。在相关文献中，你还可以遇到术语*分割加载/存储split load/store*来描述这种情况。如果连续执行多个分割加载/存储操作并耗尽所有可用的分割寄存器，则可能会导致性能损失。Intel的TMA方法论使用 `存储瓶颈Memory_Bound` -> `L1_Bound` -> `Split Loads` 指标来跟踪这种情况。

For instance, AVX2 memory operations can access up to 32 bytes. If an array starts at offset `0x30` (48 bytes), the first AVX2 load will fetch data from `0x30` to `0x4F`, the second load will fetch data from 0x50 to 0x6F, and so on. The first load crosses the cache line boundary (`0x40`). In fact, every second load will cross the cache line boundary which may slow down the execution. Figure @fig:SplitLoads illustrates this. Pushing the data forward by 16 bytes would align the array to the cache line boundary and eliminate split loads. [@lst:AligningData] shows how to fix this example using the C++11 `alignas` keyword.

例如，AVX2内存操作最多可以访问32字节。如果一个数组从偏移量 `0x30`（48字节）开始，第一次AVX2加载将从 `0x30` 到 `0x4F` 获取数据，第二次加载将从 `0x50` 到 `0x6F` 获取数据，依此类推。第一次加载跨越了缓存行边界 (`0x40`)。实际上，每隔一次加载都会跨越缓存行边界，这可能会降低执行速度。图 @fig:SplitLoads 说明了这一点。将数据向前移动16字节可以将数组对齐到缓存行边界，从而消除分割加载（split load）。[@lst:AligningData] 展示了如何使用C++11的 `alignas` 关键字修复此示例。

![AVX2 loads in a misaligned array. Every second load crosses the cache line boundary.AVX2加载到未对齐的数组中。每隔一次加载都会跨越缓存行边界。](../../img/memory-access-opts/SplitLoads.png){#fig:SplitLoads width=50%}

Listing: Aligning data using the "alignas" keyword.
代码立碑：使用“alignas”关键词对齐数据

~~~~ {#lst:AligningData .cpp}
// Array of 16-bit integers aligned at a 64-byte boundary
#define CACHELINE_ALIGN alignas(64) 
CACHELINE_ALIGN int16_t a[N];
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

When it comes to dynamic allocations, C++17 made it much easier. Operator `new` now takes an additional argument, which you can use to control the alignment of dynamically allocated memory. When using standard containers, such as `std::vector`, you can define a custom allocator. [@lst:AlignedStdVector] shows a minimal example of a custom allocator that aligns the memory buffer at the cache line boundary.

在动态分配方面，C++17让操作变得更加简单。运算符 `new` 现在接受一个额外的参数，您可以使用该参数来控制动态分配内存的对齐方式。使用标准容器（例如 `std::vector`）时，您可以定义自定义分配器。 [@lst:AlignedStdVector] 展示了一个自定义分配器的最小示例，该分配器将内存缓冲区对齐到缓存行边界。

Listing: Defining std::vector aligned at the cache line boundary.
代码列表：定义一个对齐到缓存行边界的 std::vector

~~~~ {#lst:AlignedStdVector .cpp}
// Returns aligned pointers when allocations are requested. 
template <typename T>
class CacheLineAlignedAllocator {
public:
  using value_type = T;
  static std::align_val_t constexpr ALIGNMENT{64};
  [[nodiscard]] T* allocate(std::size_t N) {
    return reinterpret_cast<T*>(::operator new[](N * sizeof(T), ALIGNMENT));
  }
  void deallocate(T* allocPtr, [[maybe_unused]] std::size_t N) {
    ::operator delete[](allocPtr, ALIGNMENT);
  }
};
template<typename T> 
using AlignedVector = std::vector<T, CacheLineAlignedAllocator<T> >;
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

To demonstrate the effect of misaligned memory accesses, I created the [mem_alignment_1](https://github.com/dendibakh/perf-ninja/tree/main/labs/memory_bound/mem_alignment_1)[^5] lab assignment in the Performance Ninja online course. It features a very simple matrix multiplication example, where the initial version doesn't take any care of the alignment of the matrices. The assignment asks to align the matrices to the cache line boundary and measure the performance difference. Feel free to experiment with the code and measure the effect on your platform.

为了演示内存访问未对齐的影响，我在性能忍者（Performance Ninja）在线课程中创建了内存对齐一[mem_alignment_1](https://github.com/dendibakh/perf-ninja/tree/main/labs/memory_bound/mem_alignment_1)[^5] 实验作业。该作业包含一个非常简单的矩阵乘法示例，初始版本并未考虑矩阵的对齐方式。作业要求将矩阵对齐到缓存行边界，并测量性能差异。你可以随意尝试修改代码，并在你自己的平台上测量其影响。

The first step to mitigate split loads/stores in this assignment is to align the starting offset of a matrix. The operating system might allocate memory for the matrix such that it is already aligned to the cache line boundary. However, you should not rely on this behavior, as it is not guaranteed. A simple way to fix this is to use `AlignedVector` from [@lst:AlignedStdVector] to allocate memory for the matrices. 

缓解此作业中分割加载/存储操作的第一步是对齐矩阵的起始偏移量。操作系统可能会为矩阵分配内存，使其已与缓存行边界对齐。但是，你不应该依赖此行为，因为它并没有保障。一个简单的解决方法是使用 [@lst:AlignedStdVector] 中的 `AlignedVector` 为矩阵分配内存。

However, it's not enough to only align the starting offset of a matrix. Consider an example of a `9x9` matrix of `float` values shown in Figure @fig:MemAlignment. If a cache line is 64 bytes, it can store 16 `float` values. When using AVX2 instructions, the program will load/store 8 elements (256 bits) at a time. In each row, the first eight elements will be processed in a SIMD way, while the last element will be processed in a scalar way by the loop remainder. The second vector load/store (elements 10-17) crosses the cache line boundary as many other subsequent vector loads/stores. The problem highlighted in Figure @fig:MemAlignment affects any matrix with the number of columns that is not a multiple of 8 (for AVX2 vectorization). The SSE and ARM Neon vectorization requires 16-byte alignment; AVX-512 requires 64-byte alignment.

然而，仅仅对齐矩阵的起始偏移量是不够的。考虑一个如图 @fig:MemAlignment 所示的 9x9 浮点数矩阵示例。如果缓存行大小为64字节，则可以存储16个浮点数。使用AVX2指令时，程序每次加载/存储8个元素（256位）。在每一行中，前八个元素将以SIMD方式处理，而最后一个元素将由循环余数以标量方式处理。第二个向量加载/存储操作（元素10-17）以及许多后续的向量加载/存储操作都会跨越缓存行边界。如图 @fig:MemAlignment 所示，任何列数不是8倍数的矩阵（对于AVX2向量化）都会受到影响。SSE和ARM Neon向量化需要16字节对齐；AVX-512需要64字节对齐。

![Split loads/stores inside a 9x9 matrix when using AVX2 vectorization. The split memory access is highlighted in yellow. 使用AVX2向量化时，9x9矩阵内部的加载/存储操作会进行拆分。拆分的内存访问以黄色高亮显示](../../img/memory-access-opts/MemAlignment.png){#fig:MemAlignment width=80%}

So, in addition to aligning the starting offset, each row of the matrix should be aligned as well. For example in Figure @fig:MemAlignment, it can be achieved by inserting seven dummy columns into the matrix, effectively making it a `9x16` matrix. This will align the second row (elements 10-18) at the offset `0x40`. Similarly, all the other rows will be aligned as well. The dummy columns will not be processed by the algorithm, but they will ensure that the actual data is aligned at the cache line boundary. In my testing, the performance impact of this change was up to 30%, depending on the matrix size and platform configuration.

因此，除了对齐起始偏移量之外，矩阵的每一行也应该对齐。例如，在图 @fig:MemAlignment 中，可以通过在矩阵中插入七个虚拟列来实现，从而有效地将其变成一个 `9x16` 矩阵。这将使第二行（元素10-18）在偏移量 `0x40` 处对齐。类似地，所有其他行也将对齐。虚拟列不会被算法处理，但它们可以确保实际数据在缓存行边界处对齐。在我的测试中，这种更改对性能的影响高达30%，具体取决于矩阵大小和平台配置。

Alignment and padding cause holes with unused bytes, which potentially decreases memory bandwidth utilization. For small matrices, like our 9x9 matrix, padding will cause almost half of each row to be unused. However, for large matrices, like 1025x1025 the impact of padding is not that big. Nevertheless, for some algorithms, e.g., in AI, memory bandwidth can be a bigger concern. Use these techniques with care and always measure to see if the performance gain from alignment is worth the cost of unused bytes.

对齐和填充会导致内存中出现未使用的字节，从而可能降低内存带宽利用率。对于像我们9x9矩阵这样的小矩阵，填充会导致几乎每行的一半内存都未使用。然而，对于像1025x1025这样的大矩阵，填充的影响并不大。尽管如此，对于某些算法（例如：人工智能算法），内存带宽可能是一个更重要的问题。因此，请谨慎使用这些技术，并始终进行测量，以确定对齐带来的性能提升是否值得以未使用的字节为代价。

Accesses that cross a 4 KB boundary introduce more complications because virtual to physical address translations are usually handled in 4 KB pages. Handling such access would require accessing two TLB entries as well. Unless a TLB supports multiple lookups per cycle, such loads can cause a significant slowdown.

跨越4KB边界的访问会引入更多复杂性，因为虚拟地址到物理地址的转换通常是在4KB的页面中进行的。处理此类访问还需要访问两个TLB条目。除非TLB支持每个周期多次查找，否则此类加载可能会导致显著的性能下降。

### Cache Aliasing 缓存别名 {#sec:CacheTrashing}

There are specific data access patterns that may cause unpleasant performance issues. These corner cases are tightly connected with cache organization, e.g., the number of sets and ways in the cache. We discussed cache organization in [@sec:CacheHierarchy], in case you want to revisit it. The placement of a memory location in the cache is determined by its address. Based on the address bits, the cache controller does set selection, i.e., it determines the set to which a cache line with the fetched memory location will go. 

某些特定的数据访问模式可能会导致性能问题。这些特殊情况与缓存组织结构密切相关，例如缓存中的集合数量和路数。我们在 [@sec:CacheHierarchy] 中讨论过缓存组织结构，如果你想回顾一下，可以参考该部分。缓存中内存位置的放置由其地址决定。缓存控制器根据地址位进行集合选择，即确定包含所获取内存位置的缓存行将放入哪个集合。

If two memory locations map to the same set, they will compete for the limited number of available slots (ways) in a set. When a program repeatedly accesses memory locations that map to the same set, they will be constantly evicting each other. This may cause saturation of one set in the cache and underutilization of other sets. This is known as *cache aliasing*, though you may find people use the terms *cache contention*, *cache conflicts*, or *cache trashing* to describe this effect.

如果两个内存位置映射到同一个集合，它们将竞争该集合中有限的可用槽位（路数）。当程序反复访问映射到同一个集合的内存位置时，它们会不断地互相驱逐。这可能会导致缓存中某个集合饱和，而其他集合利用率不足。这种情况被称为“*缓存别名cache aliasing*”，不过您可能也会看到人们使用“*缓存争用cache contention*”、“*缓存冲突cache conflicts*”或“*缓存丢弃cache trashing*”等术语来描述这种现象。

A simple example of cache aliasing can be observed in matrix transposition as explained in detail in [@fogOptimizeCpp, section 9.10 Cache contentions in large data structures]. I encourage readers to study this manual to learn more about why it happens. I repeated the experiment on a few modern processors and confirmed that it remains a relevant issue. Figure @fig:CacheAliasing shows the performance of transposing matrices of 32-bit floating-point values on Intel's 12th-gen core i7-1260P processor. 

缓存别名的一个简单例子可以在矩阵转置中观察到，详情请参见[@fogOptimizeCpp，第9.10节“大型数据结构中的缓存争用”]。我建议读者仔细阅读该手册，以了解其发生的原因。我在几款现代处理器上重复了该实验，并确认该问题仍然存在。图 @fig:CacheAliasing 展示了在Intel第12代酷睿Core i7-1260P处理器上转置32位浮点值矩阵的性能。

![Cache aliasing effects observed in matrix transposition ran on Intel's 12th-gen processor. Matrix sizes that are powers of two or multiples of 128 cause more than 10x performance drop. 在Intel第12代处理器上运行的矩阵转置中观察到的缓存别名效应。矩阵大小为2的幂或128的倍数会导致性能下降10倍以上。](../../img/memory-access-opts/CacheAliasing.png){#fig:CacheAliasing width=100%}

There are several spikes in the chart, which correspond to the matrix sizes that cause cache aliasing. Performance drops significantly when the matrix size is a power of two (e.g., 256, 512) or is a multiple of 128 (e.g., 384, 640, 768, 896).[^6] This happens because memory locations that belong to the same column are mapped to the same set in the L1D and L2 caches. These memory locations compete for the limited number of ways in the set which causes the same cache line to be reloaded many times before every element on this line is processed.

图表中出现了几个峰值，这些峰值对应于导致缓存别名的矩阵大小。当矩阵大小为2的幂（例如：256、512）或128 的倍数（例如：384、640、768、896）时，性能会显著下降[^6]。这是因为属于同一列的内存位置在L1D和L2缓存中被映射到同一个集合。这些内存位置会竞争集合中有限的路数，导致同一缓存行在处理完该行上的所有元素之前被多次重新加载。

On Intel's processors, this issue can be diagnosed with the help of the `L1D.REPLACEMENT` performance event, which counts L1 cache line replacements. For instance, there are 17 times more cache line replacements for the matrix size `256x256` than for the size `255x255`. I tested all sizes from `64x64` up to `10,000x10,000` and found that the pattern repeats very consistently. I also ran the same experiment on an Intel Skylake-based processor as well as an Apple M1 chip and confirmed that these chips are prone to cache aliasing effects. 

在Intel处理器上，可以通过 `L1D.REPLACEMENT` 性能事件来诊断此问题，该事件会统计L1缓存行替换次数。例如，矩阵大小为 `256x256` 时，缓存行替换次数是大小为 `255x255` 时的17倍。我测试了从 `64x64` 到 `10,000x10,000` 的所有大小，发现这种模式非常一致地重复出现。我还在基于Intel Skylake体系结构的处理器和苹果M1芯片上进行了相同的实验，并证实这些芯片容易受到缓存别名效应的影响。

To mitigate cache aliasing, you can use cache blocking as we discussed in [@sec:LoopOptsHighLevel]. The idea is to process the matrix in smaller blocks that fit into the cache. That way you will avoid cache line eviction since there will be enough space in the cache. Another way to solve this is to pad the matrix with extra columns, e.g., instead of a `256x256` matrix, you would allocate a `256x264` matrix; in a similar way we did in the previous section. But be careful not to run into misaligned memory access issues.

为了缓解缓存别名，可以使用我们在 [@sec:LoopOptsHighLevel] 中讨论过的缓存分块。其思路是将矩阵分成更小的块进行处理，这些块可以放入缓存中。这样，由于缓存中有足够的空间，就可以避免缓存行被驱逐。另一种解决方法是用额外的列填充矩阵，例如，用一个256x264的矩阵代替一个256x256的矩阵；这与我们在上一节中所做的类似。但要注意避免出现内存访问错位的问题。

### Slow Floating-Point Arithmetic 缓慢的浮点算术 {#sec:SlowFloatingPointArithmetic}

Some applications that do extensive computations with floating-point (FP) values are prone to one very subtle issue that can cause performance slowdown. This issue arises when an application hits a _subnormal_ FP value, which we will discuss in this section. You can also find a term *denormal* FP value, which refers to the same thing. According to the IEEE Standard 754,[^4] a subnormal is a non-zero number with an exponent smaller than the smallest normal number.[^3] [@lst:Subnormals] shows a very simple instantiation of a subnormal value. 

一些使用浮点(FP)值进行大量计算的应用程序容易遇到一个非常微妙的问题，导致性能下降。当应用程序遇到*次规格（subnormal）*浮点值时，就会出现这个问题，我们将在本节中讨论这个问题。您还可以找到术语*非正规denormal*浮点值，它指的是同一件事。根据IEEE 754标准[^4]，次规格值是指指数小于最小规格数的非零数。[^3] [@lst:Subnormals] 展示了一个非常简单的次正规值示例。

In real-world applications, a subnormal value usually represents a signal so small that it is indistinguishable from zero. In audio, it can mean a signal so quiet that it is out of the human hearing range. In image processing, it can mean any of the RGB color components of a pixel to be very close to zero, and so on. Interestingly, subnormal values are present in many production software packages, including weather forecasting, ray tracing, physics simulations, and others.

在实际应用中，次规格值通常表示信号非常小，以至于无法与零区分开来。在音频领域，它可能意味着信号非常微弱，超出了人耳的听觉范围。在图像处理中，它可能意味着像素的任何RGB颜色分量都非常接近于零，等等。有趣的是，许多生产软件中都存在次规格值，包括：天气预报、光线追踪、物理模拟等等。

Listing: Instantiating a normal and subnormal FP value
代码列表：实例化一个规格化和一个次规格化的浮点值

~~~~ {#lst:Subnormals .cpp}
unsigned usub = 0x80200000; // -2.93873587706e-39 (subnormal)
unsigned unorm = 0x411a428e; // 9.641248703 (normal)
float sub = *((float*)&usub);
float norm = *((float*)&unorm);
assert(std::fpclassify(sub) == FP_SUBNORMAL);
assert(std::fpclassify(norm) != FP_SUBNORMAL);
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Without subnormal values, the subtraction of two FP values `a - b` can underflow and produce zero even though the values are not equal. Subnormal values allow calculations to gradually lose precision without rounding the result to zero. Although, it can come with a cost as we shall see later. Subnormal values also may occur in production software when a value keeps decreasing in a loop with subtraction or division.

如果没有次规格数，两个浮点数 `a - b` 的减法运算可能会下溢，即使这两个值并不相等，结果也为零。次正规数允许计算逐渐降低精度，而无需将结果舍入为零。然而，正如我们稍后将看到的，这样做会带来一些性能开销。在生产软件中，当一个值在循环中进行减法或除法运算时持续递减时，也可能出现次正规数。

From the hardware perspective, handling subnormals is more difficult than handling normal FP values as it requires special treatment and generally, is considered as an exceptional situation. The application will not crash, but it will get a performance penalty. Calculations that produce or consume subnormal numbers are slower than similar calculations on normal numbers and can run 10 times slower or more. For instance, Intel processors currently handle operations on subnormals with a microcode *assist*. When a processor recognizes a subnormal FP value, a Microcode Sequencer (MSROM) will provide the necessary microoperations ($\mu$ops) to compute the result.

从硬件角度来看，处理次规格数比处理规格浮点数更困难，因为它需要特殊处理，通常被视为一种特殊情况。应用程序不会崩溃，但性能会受到影响。产生或使用次规格数的计算比对规格数进行的类似计算要慢，速度可能会慢10倍甚至更多。例如，Intel处理器目前使用微码*辅助assist*来处理次正规数运算。当处理器识别到次规格浮点数时，微代码序列器(MSROM)将提供必要的微操作(μop)来计算结果。

In many cases, subnormal values are generated naturally by the algorithm and thus are unavoidable. Most processors give the option to flush subnormal values to zero and not generate subnormals in the first place. Developers of performance-critical applications perhaps would prefer to have slightly less accurate results than slowing down the code.

在许多情况下，算法会自然生成次规格值，因此无法避免。大多数处理器都提供了将次规格值清零的选项，从而从一开始就避免生成次规格值。对于性能至关重要的应用程序的开发者来说，他们或许宁愿接受略微降低的精度，也不愿降低代码的运行速度。

Suppose your application doesn't need subnormal values, how do you detect and mitigate associated costs? While you can use runtime checks as shown in [@lst:Subnormals], inserting them all over the codebase is not practical. There is a better way to detect if your application is producing subnormal values using PMU (Performance Monitoring Unit). On Intel CPUs, you can collect the `FP_ASSIST.ANY` performance event, which gets incremented every time a subnormal value is used or produced. The TMA methodology classifies such bottlenecks under the `Retiring` category, and yes, this is another situation when a high `Retiring` doesn't mean a good thing.

假设您的应用程序不需要次规格值，那么如何检测并降低相关的性能开销呢？虽然您可以使用运行时检查（如 [@lst:Subnormals] 中所示），但将它们插入到代码库的各个角落并不实际。使用性能监控单元(PMU)可以更好地检测应用程序是否生成了次正规值。在Intel CPU上，您可以收集 `FP_ASSIST.ANY` 性能事件，该事件会在每次使用或生成次规格值时递增。TMA方法论将此类瓶颈归类为 `Retiring` 类别，而高 `Retiring` 值在这种情况下并非好事。

Once you confirm subnormal values are there, you can enable the FTZ and DAZ modes:

确认存在非正规值后，即可启用FTZ和DAZ模式：

* __DAZ__ (Denormals Are Zero). Any denormal inputs are replaced by zero before use.
* __FTZ__ (Flush To Zero). Any outputs that would be denormal are replaced by zero.

* __DAZ__（非规格值归零）。所有非规格输入在使用前都会被替换为零。
* __FTZ__（清零）。所有非规格输出都会被替换为零。

When they are enabled, there is no need for costly handling of subnormal values in a CPU floating-point arithmetic. In x86-based platforms, there are two separate bit fields in the `MXCSR`, global control and status register. In ARM Aarch64, two modes are controlled with `FZ` and `AH` bits of the `FPCR` control register. If you compile your application with `-ffast-math`, you have nothing to worry about, the compiler will automatically insert the required code to enable both flags at the start of the program. The `-ffast-math` compiler option is a little overloaded, so GCC developers created a separate `-mdaz-ftz` option that only controls the behavior of subnormal values. If you'd rather control it from the source code, [@lst:EnableFTZDAZ] shows an example that you can use. If you choose this option, avoid frequent changes to the `MXCSR` register because the operation is relatively expensive. A read of the MXCSR register has a fairly long latency, and a write to the register is a serializing instruction.

启用这些模式后，CPU浮点运算中无需对非规格值进行耗时的计算。在基于x86的平台上，`MXCSR` 寄存器中有两个独立的位域：全局控制位域和状态位域。在ARM Aarch64体系结构中，`FPCR` 控制寄存器的 `FZ` 和 `AH` 位控制着这两种模式。如果你使用 `-ffast-math` 编译应用程序，则无需担心，编译器会在程序开始时自动插入启用这两个标志所需的代码。 `-ffast-math` 编译器选项功能略显繁杂，因此GCC开发者创建了一个单独的 `-mdaz-ftz` 选项，专门用于控制次规格值的行为。如果您更倾向于从源代码进行控制，可以参考 [@lst:EnableFTZDAZ] 提供的示例。如果您选择此选项，请避免频繁修改 `MXCSR` 寄存器，因为该操作开销相对较大。读取MXCSR寄存器的延迟相当长，而写入该寄存器则属于串行指令。

Listing: Enabling FTZ and DAZ modes manually
代码列表：手动启用FTZ和DAZ

~~~~ {#lst:EnableFTZDAZ .cpp}
unsigned FTZ = 0x8000;
unsigned DAZ = 0x0040;
unsigned MXCSR = _mm_getcsr();
_mm_setcsr(MXCSR | FTZ | DAZ);
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Keep in mind, that both `FTZ` and `DAZ` modes are incompatible with the IEEE Standard 754. They are implemented in hardware to improve performance for applications where underflow is common and generating a denormalized result is unnecessary. I have observed a 3%-5% performance penalty on some production floating-point applications that were using subnormal values.

请注意，`FTZ` 和 `DAZ` 模式均与IEEE 754标准不兼容。它们在硬件中实现，旨在提高经常出现下溢且无需生成非规范化结果的应用的性能。我观察到，在一些使用次规范化值的生产级浮点应用中，性能损失约为3%-5%。

[^1]: Otsu's thresholding method Otsu阈值法 - [https://en.wikipedia.org/wiki/Otsu%27s_method](https://en.wikipedia.org/wiki/Otsu%27s_method)
[^2]: Performance Ninja lab assignment: Memory Order Violation 性能忍者实验作业：存储序违例 - [https://github.com/dendibakh/perf-ninja/tree/main/labs/memory_bound/mem_order_violation_1](https://github.com/dendibakh/perf-ninja/tree/main/labs/memory_bound/mem_order_violation_1)
[^3]: Subnormal number 次规格数 - [https://en.wikipedia.org/wiki/Subnormal_number](https://en.wikipedia.org/wiki/Subnormal_number)
[^4]: IEEE Standard 754 IEEE754标准 - [https://ieeexplore.ieee.org/document/8766229](https://ieeexplore.ieee.org/document/8766229)
[^5]: Performance Ninja lab assignment: Memory Alignment 性能忍者实验作业：内存对齐 - [https://github.com/dendibakh/perf-ninja/tree/main/labs/memory_bound/mem_alignment_1](https://github.com/dendibakh/perf-ninja/tree/main/labs/memory_bound/mem_alignment_1)
[^6]: Also, there are a few spikes at the sizes 341, 683, and 819. Supposedly, these sizes suffer from the same cache aliasing effect, but I haven't investigated them further. 此外，在341、683和819这几个大小处也出现了一些峰值。据推测，这些大小也存在同样的缓存别名效应，但我还没有对此进行深入研究。
