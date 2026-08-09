## Cache-Friendly Data Structures 缓存友好的数据结构 {#sec:secCacheFriendly}

Writing cache-friendly algorithms and data structures is one of the key items in the recipe for a well-performing application. The key pillars of cache-friendly code are the principles of temporal and spatial locality that we introduced in [@sec:MemHierar]. The goal here is to have a predictable memory access pattern and store data efficiently.

编写缓存友好型算法和数据结构是打造高性能应用程序的关键要素之一。缓存友好型代码的核心支柱是我们在 [@sec:MemHierar] 中介绍的时间局部性和空间局部性原则。其目标是实现可预测的存储访问模式并高效地存储数据。

The cache line is the smallest unit of data that can be transferred between the cache and the main memory. When designing cache-friendly code, it's helpful to think not only of individual variables and their locations in memory but also of cache lines.

缓存行是缓存和主内存之间可以传输的最小数据单元。在设计缓存友好型代码时，不仅要考虑单个变量及其在内存中的位置，还要考虑缓存行，这一点非常重要。

Next, we will discuss several techniques to make data structures more cache-friendly.

接下来，我们将讨论几种使数据结构更适合缓存的技术。

### Access Data Sequentially 顺序访问数据

The best way to exploit the spatial locality of the caches is to make sequential memory accesses. By doing so, we enable the hardware prefetching mechanism (see [@sec:HwPrefetch]) to recognize the memory access pattern and bring in the next chunk of data ahead of time. An example of Row-major versus Column-Major traversal is shown in [@lst:CacheFriend]. Notice, that there is only one tiny change in the code (swapped `col` and `row` subscripts), but it has a large impact on performance.

利用缓存空间局部性的最佳方法是进行顺序存储访问。这样做可以使硬件预取机制（参见 [@sec:HwPrefetch]）识别存储访问模式并提前加载下一块数据。 [@lst:CacheFriend] 展示了行优先遍历和列优先遍历的示例。请注意，代码中只有一个很小的改动（交换了 `col` 和 `row` 下标），但却对性能产生了很大的影响。

The code on the left is not cache-friendly because it skips the `NCOLS` elements on every iteration of the inner loop. This results in a very inefficient use of caches: we aren't making full use of the entire prefetched cache line before it gets evicted. In contrast, the code on the right accesses elements of the matrix in the order in which they are laid out in memory. This guarantees that the cache line will be fully used before it gets evicted. Row-major traversal exploits spatial locality and is cache-friendly. Figure @fig:ColRowMajor illustrates the difference between the two traversal patterns.

左侧的代码对缓存不友好，因为它在内层循环的每次迭代中都会跳过 `NCOLS` 元素。这导致缓存利用率非常低：在预取的缓存行被驱逐之前，我们并没有充分利用它。相比之下，右侧的代码按照矩阵元素在内存中的排列顺序访问它们。这保证了缓存行在被驱逐之前会被完全使用。行优先遍历利用了空间局部性，因此对缓存友好。图 @fig:ColRowMajor 展示了两种遍历模式之间的区别。

Listing: Cache-friendly memory accesses.
代码列表：对缓存友好的内存访问

~~~~ {#lst:CacheFriend .cpp}
// Column-major order                              // Row-major order
for (row = 0; row < NROWS; row++)                  for (row = 0; row < NROWS; row++)
  for (col = 0; col < NCOLS; col++)                  for (col = 0; col < NCOLS; col++)
    matrix[col][row] = row + col;          =>          matrix[row][col] = row + col;
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

![Column-major versus Row-major traversal. 列优先遍历 对比 行优先遍历](../../img/memory-access-opts/ColumnRowMajor.png){#fig:ColRowMajor width=60%}

The example presented above is classical, but usually, real-world applications are much more complicated than this. Sometimes you need to go an additional mile to write cache-friendly code. If the data is not laid out in memory in a way that is optimal for the algorithm, it may require to rearrange the data first.

上面的例子很经典，但通常实际应用要复杂得多。有时，为了编写对缓存友好的代码，你需要付出更多努力。如果数据在内存中的布局并非算法的最佳方式，则可能需要先重新排列数据。

Consider a standard implementation of binary search in a large sorted array, where on each iteration, you access the middle element, compare it with the value you're searching for, and go either left or right. This algorithm does not exploit spatial locality since it tests elements in different locations that are far away from each other and do not share the same cache line. The most famous way of solving this problem is storing elements of the array using the Eytzinger layout [@EytzingerArray]. The idea is to maintain an implicit binary search tree packed into an array using the BFS-like layout, usually seen with binary heaps. If the code performs a large number of binary searches in the array, it may be beneficial to convert it to the Eytzinger layout. 

考虑一个在大型有序数组中进行二分查找的标准实现：每次迭代，你都会访问中间元素，将其与要查找的值进行比较，然后向左或向右移动。由于该算法测试的是位于不同位置的元素，这些位置彼此相距甚远，并且不共享同一缓存行，因此它没有利用空间局部性。解决此问题的最著名方法是使用Eytzinger布局 [@EytzingerArray] 存储数组元素。其思路是维护一个隐式二叉搜索树，并将其打包到一个数组中，采用类似广度优先搜索（BFS）的布局，这种布局常见于二叉堆。如果代码在数组中执行大量二分查找，则将其转换为Eytzinger布局可能更有利。

### Use Appropriate Containers. 使用合适的容器。

There is a wide variety of ready-to-use containers in almost any language. But it's important to know their underlying storage and performance implications. Keep in mind how the data will be accessed and manipulated. You should consider not only the time and space complexity of operations with a data structure but also the hardware effects associated with them.

几乎所有编程语言都提供了各种现成的容器。但了解它们底层的存储和性能影响至关重要。务必牢记数据的访问和操作方式。你不仅应该考虑数据结构操作的时间和空间复杂度，还应该考虑与之相关的硬件影响。

By default, stay away from data structures that rely on pointers, e.g. linked lists or trees. When traversing elements, they require additional memory accesses to follow the pointers. If the maximum number of elements is relatively small and known at compile time, C++ `std::array` might be a better option than `std::vector`. If you need an associative container but don't need to store the elements in sorted order, `std::unordered_map` should be faster than `std::map`. A good step-by-step guide for choosing appropriate C++ containers can be found in [@fogOptimizeCpp, Section 9.7 Data structures, and container classes].

默认情况下，应避免使用依赖指针的数据结构，例如：链表或树。在遍历元素时，它们需要额外的内存访问来跟踪指针。如果最大元素数量相对较小且在编译时已知，则C++的 `std::array` 可能比 `std::vector` 更合适。如果您需要一个关联容器，但不需要按排序顺序存储元素，那么 `std::unordered_map` 应该比 `std::map` 更快。关于如何选择合适的C++容器，您可以参考 [@fogOptimizeCpp，第9.7节--数据结构和容器类]。

Sometimes, it's more efficient to store pointers to contained objects, instead of objects themselves. Consider a situation when you need to store many objects in an array while the size of each object is big. In addition, the objects are frequently shuffled, removed, and inserted. Storing objects in an array will require moving large chunks of memory every time the order of objects is changed, which is expensive. In this case, it's better to store pointers to objects in the array. This way, only the pointers are moved, which is much cheaper. However, this approach has its drawbacks. It requires additional memory for the pointers and introduces an additional level of indirection.

有时，存储指向被存储对象的指针比存储对象本身更高效。例如，假设您需要在一个数组中存储大量对象，而每个对象的大小都很大。此外，这些对象还会频繁地被打乱、删除和插入。如果每次改变对象的顺序，将对象存储在数组中都需要移动大量的内存，这会带来很高的开销。在这种情况下，最好存储指向数组中对象的指针。这样，只需要移动指针，开销要小得多。然而，这种方法也有缺点。它需要额外的内存来存储指针，并且引入了额外的间接层。

### Packing the Data 数据封包

The utilization of data caches can be also improved by making data more compact. There are many ways to pack data. One of the classic examples is to use bitfields. An example of code when packing data might be profitable is shown in [@lst:DataPacking]. If we know that `a`, `b`, and `c` represent enum values that take a certain number of bits to encode, we can reduce the storage of the struct `S`.

通过使数据更紧凑，可以提高数据缓存的利用率。数据打包的方法有很多种。一个经典的例子是使用位域。[@lst:DataPacking] 中展示了一个数据封包可能带来收益的代码示例。如果我们知道 `a`、`b` 和 `c` 代表枚举值，并且每个枚举值需要一定数量的比特来编码，那么我们可以减少结构体 `S` 的存储空间。

Listing: Data Packing
代码列表：数据封包

~~~~ {#lst:DataPacking .cpp}
// S is 3 bytes                         // S is 1 byte
struct S {                              struct S {
  unsigned char a;                        unsigned char a:4;
  unsigned char b;                =>      unsigned char b:2;
  unsigned char c;                        unsigned char c:2;
};                                      };
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Notice the three times less space required to store an object of the packed version of `S`. This greatly reduces the amount of memory transferred back and forth and saves cache space. However, using bitfields comes with additional costs.[^15] Since the bits of `a`, `b`, and `c` are packed into a single byte, the compiler needs to perform additional bit manipulation operations to extract and insert them. For example, to load `b`, you need to shift the byte value right (`>>`) by 2 and do logical AND (`&`) with `0x3`. Similarly, shift left (`<<`) and logical OR (`|`) operations are needed to store the updated value back into the packed format. Data packing is beneficial in places where additional computation is cheaper than the delay caused by inefficient memory transfers.

注意，存储打包后的 `S` 对象所需的空间减少了三倍。这大大减少了内存的来回传输，从而节省了缓存空间。然而，使用位域也会带来额外的开销[^15]。由于 `a`、`b` 和 `c` 的位被打包成一个字节，编译器需要执行额外的位操作来提取和插入它们。例如，要加载 `b`，需要将字节值右移（`>>`）2位，然后与 `0x3` 进行逻辑与运算（`&`）。类似地，需要左移（`<<`）2位和逻辑或运算（`|`）才能将更新后的值存储回打包格式。在额外计算比低效内存传输造成的延迟更有利的情况下，数据打包是有益的。

Also, a programmer can reduce memory usage by rearranging fields in a struct or class when it avoids padding added by a compiler. Inserting unused bytes of memory (pads) enables efficient storing and fetching of individual members of a struct. In the example in [@lst:AvoidPadding], the size of `S` can be reduced if its members are declared in the order of decreasing size. Figure @fig:AvoidPadding illustrates the effect of rearranging the fields in struct `S`.

此外，程序员还可以通过重新排列结构体或类中的字段来减少内存使用量，从而避免编译器添加填充。插入未使用的内存字节（填充）可以高效地存储和读取结构体的各个成员。在示例 [@lst:AvoidPadding] 中，如果按大小递减的顺序声明结构体 `S` 的成员，则可以减小其大小。图 @fig:AvoidPadding 展示了重新排列结构体 `S` 中字段的效果。

Listing: Avoid compiler padding.
代码列表：避免编译器填充

~~~~ {#lst:AvoidPadding .cpp}
// S is `sizeof(int) * 3` bytes          // S is `sizeof(int) * 2` bytes
struct S {                               struct S {
  bool b;                                  int i;
  int i;                         =>        short s;
  short s;                                 bool b;
};                                       };

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

![Avoid compiler padding by rearranging the fields. Blank cells represent compiler padding. 通过重新排列字段来避免编译器填充。空白单元格表示编译器填充。](../../img/memory-access-opts/AvoidPadding.png){#fig:AvoidPadding width=90%}

### Field Reordering 字段重排序

Reordering fields in a data structure can also be beneficial for another reason. Consider an example in [@lst:FieldReordering]. Suppose that the `Soldier` structure is used to track each one of the thousands of units on the battlefield in a game. The game has three phases: battle, movement, and trade. During the battle phase, the `attack`, `defense`, and `health` fields are used. During the movement phase, the `coords`, and `speed` fields are used. During the trade phase, only the `money` field is used.

重新排列数据结构中的字段还有另一个好处。考虑一下 [@lst:FieldReordering] 中的示例。假设使用 `Soldier` 结构来跟踪游戏中战场上数千个单位中的每一个。游戏分为三个阶段：战斗、移动和交易。在战斗阶段，使用 `attack`、`defense` 和 `health` 字段。在移动阶段，使用 `coords` 和 `speed` 字段。在交易阶段，只使用 `money` 字段。

The problem with the organization of the `Soldier` struct in the code on the left is that the fields are not grouped according to the phases of the game. For example, during the battle phase, the program needs to access two different cache lines to fetch the required fields. The fields `attack` and `defense` are very likely to reside on the same cache line, but the `health` field is always pushed to the next cache line. The same applies to the movement phase (`speed` and `coords` fields).

左侧代码中 `Soldier` 结构体的组织方式存在问题，即字段没有根据游戏阶段进行分组。例如，在战斗阶段，程序需要访问两个不同的缓存行来获取所需的字段。`attack` 和 `defense` 字段很可能位于同一缓存行，但 `health` 字段总是被推到下一个缓存行。移动阶段也存在同样的问题（`speed` 和 `coords` 字段）。

We can make the `Soldier` struct more cache-friendly by reordering the fields as shown in [@lst:FieldReordering] on the right. With that change, the fields that are accessed together are grouped together.

我们可以通过重新排列字段顺序来使 `Soldier` 结构体更利于缓存，如右侧的 [@lst:FieldReordering] 所示。通过这种更改，需要同时访问的字段将被分组在一起。

Listing: Field Reordering.
代码列表：字段重排序

~~~~ {#lst:FieldReordering .cpp}
struct Soldier {                                 struct Soldier {
  2DCoords coords;   /*  8 bytes */                unsigned attack;  // 1. battle
  unsigned attack;                                 unsigned defense; // 1. battle
  unsigned defense;                     =>         unsigned health;  // 1. battle
  /* other fields */ /* 64 bytes */                2DCoords coords;  // 2. move
  unsigned speed;                                  unsigned speed;   // 2. move
  unsigned money;                                  // other fields
  unsigned health;                                 unsigned money;   // 3. trade
};                                                };
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Since Linux kernel 6.8, there is a new functionality in the `perf` tool that allows you to find data structure reordering opportunities. The `perf mem record` command can now be used to profile data structure access patterns. The `perf annotate --data-type` command will show you the data structure layout along with profiling samples attributed to each field of the data structure. Using this information you can identify fields that are accessed together.[^5]

自Linux内核6.8起，`perf` 工具新增了一项功能，可用于查找数据结构重排序的机会。现在可以使用 `perf mem record` 命令来分析数据结构的访问模式。`perf annotate --data-type` 命令将显示数据结构的布局，以及与数据结构中每个字段关联的性能轮廓分析样本。利用这些信息，您可以识别出哪些字段经常被一起访问[^5]。

Data-type profiling is very effective at finding opportunities to improve cache utilization. Recent Linux kernel history contains many examples of commits that reorder structures,[^1] pad fields,[^3], or pack[^2] them to improve performance.

数据类型轮廓分析在查找提高缓存利用率的机会方面非常有效。最近的Linux内核历史记录中包含许多通过重排序结构[^1]、填充字段[^3]或打包[^2]来提高性能的提交示例。

### Other Data Structure Reorganization Techniques 其他数据结构重组技术

To close the topic of cache-friendly data structures, we will briefly mention two other techniques that can be used to improve cache utilization: *structure splitting* and *pointer inlining*.

为了结束关于缓存友好型数据结构的讨论，我们将简要介绍另外两种可用于提高缓存利用率的技术：*结构体拆分（structure splitting）*和*指针内联（pointer inlining）*。

**Structure splitting**. Splitting a large structure into smaller ones can improve cache utilization. For example, if you have a structure that contains a large number of fields, but only a few of them are accessed together, you can split the structure into two or more smaller ones. This way, you can avoid loading unnecessary data into the cache. An example of structure splitting is shown in [@lst:StructureSplitting]. By splitting the `Point` structure into `PointCoords` and `PointInfo`, we can avoid loading the `PointInfo` data into caches when we only need `PointCoords`. This way, we can fit more points on a single cache line.

**结构体拆分**。将大型结构体拆分为较小的结构体可以提高缓存利用率。例如，如果一个结构体包含大量字段，但只有少数字段会同时被访问，则可以将该结构体拆分为两个或多个较小的结构体。这样可以避免将不必要的数据加载到缓存中。结构体拆分的示例见 [@lst:StructureSplitting]。通过将 `Point` 结构体拆分为 `PointCoords` 和 `PointInfo`，我们可以避免在只需要 `PointCoords` 时将 `PointInfo` 数据加载到缓存中。这样，我们就可以在单个缓存行中容纳更多点。

Listing: Structure Splitting.
代码列表：结构体拆分

~~~~ {#lst:StructureSplitting .cpp}
struct Point {                                struct PointCoords {
  int X;                                        int X;
  int Y;                                        int Y;
  int Z;                                        int Z;
  /*many other fields*/            =>         };
};                                            struct PointInfo {
std::vector<Point> points;                      /*many other fields*/
                                              };
                                              std::vector<PointCoords> pointCoords;
                                              std::vector<PointInfo> pointInfos;
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**Pointer inlining**. Inlining a pointer into a structure can improve cache utilization. For example, if you have a structure that contains a pointer to another structure, you can inline the pointer into the first structure. This way, you can avoid additional memory access to fetch the second structure. An example of pointer inlining is shown in [@lst:PointerInlining]. The `weight` parameter is used in many graph algorithms, and thus, it is frequently accessed. However, in the original version on the left, retrieving the edge weight requires additional memory access, which can result in a cache miss. By moving the `weight` parameter into the `GraphEdge` structure, we avoid such issues.

**指针内联**。将指针内联到结构体中可以提高缓存利用率。例如，如果一个结构体包含指向另一个结构体的指针，则可以将该指针内联到第一个结构体中。这样，就可以避免为了获取第二个结构体而进行额外的内存访问。指针内联的示例见 [@lst:PointerInlining]。`weight` 参数在许多图算法中都有使用，因此访问频率很高。然而，在左侧的原始版本中，获取边的权重需要额外的内存访问，这可能会导致缓存未命中。通过将 `weight` 参数移到 `GraphEdge` 结构体中，我们可以避免此类问题。

Listing: Moving the `weight` parameter into the parent structure.
代码列表：将 `weight` 参数移到父结构体中。

~~~~ {#lst:PointerInlining .cpp}
struct GraphEdge {                            struct GraphEdge {
  unsigned int from;                            unsigned int from;
  unsigned int to;                              unsigned int to;
  GraphEdgeProperties* prop;                    float weight;
};                                 =>           GraphEdgeProperties* prop;
struct GraphEdgeProperties {                  };
  float weight;                               struct GraphEdgeProperties {
  std::string label;                            std::string label;
  // ...                                        // ...
};                                            };
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

[^1]: Linux commit Linux内核代码提交 [54ff8ad69c6e93c0767451ae170b41c000e565dd](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=54ff8ad69c6e93c0767451ae170b41c000e565dd)
[^2]: Linux commit Linux内核代码提交 [e5598d6ae62626d261b046a2f19347c38681ff51](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=e5598d6ae62626d261b046a2f19347c38681ff51)
[^3]: Linux commit Linux内核代码提交 [aee79d4e5271cee4ffa89ed830189929a6272eb8](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=aee79d4e5271cee4ffa89ed830189929a6272eb8)

[^5]: Linux `perf` data-type profiling Linux `perf` 数据类型轮廓分析 - [https://lwn.net/Articles/955709/](https://lwn.net/Articles/955709/)

[^12]: aligned_alloc - [https://en.cppreference.com/w/c/memory/aligned_alloc](https://en.cppreference.com/w/c/memory/aligned_alloc)
[^13]: Linux manual page for `memalign` Linux上的 `memalign` 手册页面- [https://linux.die.net/man/3/memalign](https://linux.die.net/man/3/memalign)
[^14]: Generating aligned memory 产生对齐的内存 - [https://embeddedartistry.com/blog/2017/02/22/generating-aligned-memory/](https://embeddedartistry.com/blog/2017/02/22/generating-aligned-memory/)
[^15]: Also, you cannot take the address of a bitfield. 另外，你不能获取位域的地址。
