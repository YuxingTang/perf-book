## SIMD Multiprocessors {#sec:SIMD}

Another technique to facilitate parallel processing is called Single Instruction Multiple Data (SIMD), which is used in nearly all high-performance processors. As the name indicates, in a SIMD processor, a single instruction operates on many data elements in a single cycle using many independent functional units. Operations on vectors and matrices lend themselves well to SIMD architectures as every element of a vector or matrix can be processed using the same instruction. A SIMD architecture enables more efficient processing of a large amount of data and works best for data-parallel applications that involve vector operations.

另外一种实现并行处理的技术被称为单指令流多数据流（SIMD: Single Instruction Multiple Data），几乎在所有的高性能处理器中都有使用。正如名字所暗示的，在一个SIMD处理器中，单一一条指令在单一周期内使用多个独立的功能单元同时操作多个数据元素。在向量和矩阵上的操作和SIMD体系结构非常匹配，因为在一个向量和矩阵中的每个一个元素单元都使用相同的指令来处理。一个SIMD体系结构能够对大量数据进行更高效的处理，并且对包含向量操作在内的数据并行应用是最适合的。

Figure @fig:SIMD shows scalar and SIMD execution modes for the code in @lst:SIMD. In a traditional Single Instruction Single Data (SISD) mode, also known as *scalar* mode, the addition operation is separately applied to each element of arrays `a` and `b`. However, in SIMD mode, addition is applied to multiple elements at the same time. If we target a CPU architecture that has execution units capable of performing operations on 256-bit vectors, we can process four double-precision elements with a single instruction. This leads to issuing 4x fewer instructions and can potentially gain a 4x speedup over four scalar computations.

图@fig:SIMD展示了在代码@list:SIMD中的标量和SIMD执行模式。在一个传统的单指令单数据（SISD: Single Instruction Single Data）模式，也就是*标量Scalar*模式，加法操作被单独的应用在数组`a`和`b`中的每一个元素上。但是，在SIMD模式中，加法在同时被引用于多个元素上。如果目标CPU体系结构具有能够在256位向量上执行操作的执行单元，我们就可以用一条指令处理4个双精度数据元素。这导致了发射的指令数量少了4倍，并且可以相对于4个标量计算获得4倍加速。

Listing: SIMD execution

~~~~ {#lst:SIMD .cpp}
double *a, *b, *c;
for (int i = 0; i < N; ++i) {
  c[i] = a[i] + b[i];
}
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

![Example of scalar and SIMD operations.](../../img/uarch/SIMD.png){#fig:SIMD width=80%}

For regular integer SISD instructions, processors utilize general-purpose registers. Similarly, for SIMD instructions, CPUs have a set of SIMD registers to keep the data loaded from memory and store the intermediate results of computations. In our example, two regions of 256 bits of contiguous data corresponding to arrays `a` and `b` will be loaded from memory and stored in two separate vector registers. Next, the element-wise addition will be done and the result will be stored in a new 256-bit vector register. Finally, the result will be written from the vector register to a 256-bit memory region corresponding to array `c`. Note, that the data elements can be either integers or floating-point numbers.

对于常规的整数SISD指令，处理器使用通用寄存器。类似地，对于SIMD指令，CPU拥有一组SIMD寄存器，用于保存从内存加载的数据以及计算的中间结果。在我们的例子中，对应于数组 `a` 和 `b` 的两段256位连续数据将从内存加载并存储在两个独立的向量寄存器中。接下来，进行逐元素加法运算，并将结果存储在一个新的256位向量寄存器中。最后，将结果从向量寄存器写入对应于数组 `c` 的256位内存区域。需要注意的是，数据元素可以是整数或浮点数。

A vector execution unit is logically divided into *lanes*. In the context of SIMD, a lane refers to a distinct data pathway within the SIMD execution unit and processes one element of the vector. In our example, each lane processes 64-bit elements (double-precision), so there will be 4 lanes in a 256-bit register.

向量执行单元在逻辑上被划分为多个*通道lane*。在SIMD的上下文中，通道指的是SIMD执行单元内的一个独立数据通路，它处理向量中的一个元素。在我们的示例中，每个通道处理64位元素（双精度），因此256位寄存器中将有4个通道。

Most of the popular CPU architectures feature vector instructions, including x86, PowerPC, ARM, and RISC-V. In 1996 Intel released MMX, a SIMD instruction set, that was designed for multimedia applications. Following MMX, Intel introduced new instruction sets with added capabilities and increased vector size: SSE, AVX, AVX2, and AVX-512. ARM has optionally supported the 128-bit NEON instruction set in various versions of its architecture. In version 8 (aarch64), this support was made mandatory, and new instructions were added.

大多数流行的CPU体系结构都支持向量指令，包括：x86、PowerPC、ARM和RISC-V。1996年，Intel发布了MMX，一种专为多媒体应用设计的SIMD指令集。继MMX之后，Intel推出了具有更多功能和更大向量尺寸的新指令集：SSE、AVX、AVX2和AVX-512。ARM在其体系结构的多个版本中可选地支持128位NEON 指令集。从版本8(aarch64)开始，强制支持NEON，并且添加了新的指令。

As the new instruction sets became available, work began to make them usable to software engineers. The software changes required to exploit SIMD instructions are known as *code vectorization*. Initially, SIMD instructions were programmed in assembly. Later, special compiler intrinsics, which are small functions providing a one-to-one mapping to SIMD instructions, were introduced. Today all the major compilers support autovectorization for the popular processors, i.e., they can generate SIMD instructions straight from high-level code written in C/C++, Java, Rust, and other languages.

随着新指令集的出现，人们开始着手使其能够被软件工程师使用。利用SIMD指令所需的软件更改被称为*代码向量化code vectorization*。最初，SIMD指令是用汇编语言进行编程。后来，引入了特殊的编译器内部函数，这些小函数能够提供与SIMD指令的一一对应关系。如今，所有主流编译器都支持常用处理器的自动向量化，也就是说，它们可以直接从用C/C++、Java、Rust和其他语言编写的高级代码生成SIMD指令。

To enable code to run on systems that support different vector lengths, Arm introduced the SVE instruction set. Its defining characteristic is the concept of *scalable vectors*: their length is unknown at compile time. With SVE, there is no need to port software to every possible vector length. Users don't have to recompile the source code of their applications to leverage wider vectors when they become available in newer CPU generations. Another example of scalable vectors is the RISC-V V extension (RVV), which was ratified in late 2021. Some implementations support quite wide (2048-bit) vectors, and up to eight can be grouped together to yield 16384-bit vectors, which greatly reduces the number of instructions executed. At each loop iteration, SVE code typically does `ptr += number_of_lanes`, where `number_of_lanes` is not known at compile time. ARM SVE provides special instructions for such length-dependent operations, while RVV enables a programmer to query/set the `number_of_lanes`.

为了使代码能够在支持不同向量长度的系统上运行，Arm推出了SVE指令集。它的主要特点是可扩展向量的概念：向量的长度在编译时是未知的。有了SVE，无需将软件移植到每种可能的向量长度。当新一代CPU支持更长的向量时，用户无需重新编译应用程序的源代码即可利用这些向量。可扩展向量的另一个例子是RISC-V的V扩展(RVV)，它于2021年底获得批准。一些实现支持相当宽的向量（2048位），最多可以将8个相同向量组合在一起，形成16384位向量，从而大大减少执行的指令数量。在每次循环迭代中，SVE代码通常会执行 `ptr += number_of_lanes`（向量指针按照通道数量增加），其中 `number_of_lanes`（通道数量）在编译时是未知的。ARM SVE提供了用于此类长度相关操作的特殊指令，而RVV则允许程序员查询/设置 `number_of_lanes`。

Going back to the example in @lst:SIMD, if `N` equals 5, and we have a 256-bit vector, we cannot process all the elements in a single iteration. We can process the first four elements using a single SIMD instruction, but the 5th element needs to be processed individually. This is known as the *loop remainder*. Loop remainder is a portion of a loop that must process fewer elements than the vector width, requiring additional scalar code to handle the leftover elements. Scalable vector ISA extensions do not have this problem, as they can process any number of elements in a single instruction. Another solution to the loop remainder problem is to use *masking*, which allows selectively enabling or disabling SIMD lanes based on a condition.

回到@lst:SIMD中的示例，如果N等于5，并且我们有一个256位向量，则无法在一次迭代中处理所有元素。我们可以使用一条SIMD指令处理前四个元素，但第五个元素需要单独处理。这被称为*循环余数loop remainder*。循环余数是指循环中需要处理的元素数量少于向量宽度的部分，这需要额外的标量代码来处理剩余元素。可扩展向量指令集体系结构(ISA)扩展不存在这个问题，因为它们可以在单条指令中处理任意数量的元素。解决循环余数问题的另一种方法是使用掩码，它允许根据特定条件选择性地启用或禁用SIMD通道。

Also, CPUs increasingly accelerate the matrix multiplications often used in machine learning. Intel's AMX extension, supported in server processors since 2023, multiplies 8-bit matrices of shape 16x64 and 64x16, accumulating into a 32-bit 16x16 matrix. By contrast, the unrelated but identically named AMX extension in Apple CPUs, as well as ARM's SME extension, compute outer products of a row and column, respectively stored in special 512-bit registers or scalable vectors.

此外，CPU正在不断加速机器学习中常用的矩阵乘法运算。Intel的AMX扩展自2023年起在服务器级处理器中得到支持，它可以计算16x64和64x16形状的8位矩阵的乘积，并将结果累加到一个32位16x16矩阵中。相比之下，苹果CPU中名称相同但功能不同的AMX扩展以及ARM的SME扩展分别计算行和列的外积，并将结果存储在特殊的512位寄存器或可扩展向量中。

Initially, SIMD was driven by multimedia applications and scientific computations, but later found uses in many other domains. Over time, the set of operations supported in SIMD instruction sets has steadily increased. In addition to straightforward arithmetic as shown in Figure @fig:SIMD, newer use cases of SIMD include:

最初，SIMD指令集主要应用于多媒体应用和科学计算，但后来在许多其他领域也得到了应用。随着时间的推移，SIMD指令集支持的操作种类不断增加。除了如图@fig:SIMD所示的简单算术运算之外，SIMD的新应用场景还包括：

- String processing: finding characters, validating UTF-8,[^1] parsing JSON[^2] and CSV;[^3]字符串处理：查找字符、验证UTF-8编码、[^1]解析JSON[^3]和CSV[^3]文件；
- Hashing,[^4] random generation,[^5] cryptography(AES);哈希、[^4]随机数生成、[^5]加密（AES）；
- Columnar databases (bit packing, filtering, joins);列式数据库（位打包、过滤、连接）；
- Sorting built-in types (VQSort,[^6] QuickSelect); 内置排序类型（VQSort、[^6]QuickSelect）；
- Machine Learning and Artificial Intelligence (speeding up PyTorch, TensorFlow).机器学习和人工智能（加速PyTorch和TensorFlow）。

[^1]: UTF-8 validationUTF-8 验证 - [https://github.com/rusticstuff/simdutf8](https://github.com/rusticstuff/simdutf8)
[^2]: Parsing JSON JSON 解析 - [https://github.com/simdjson/simdjson](https://github.com/simdjson/simdjson)
[^3]: Parsing CSV CSV 解析 - [https://github.com/geofflangdale/simdcsv](https://github.com/geofflangdale/simdcsv)
[^4]: SIMD hashing SIMD 哈希 - [https://github.com/google/highwayhash](https://github.com/google/highwayhash)
[^5]: Random generation 随机数生成 - [abseil library](https://github.com/abseil/abseil-cpp/blob/master/absl/random/internal/randen.h)
[^6]: Sorting 排序 - [VQSort](https://github.com/google/highway/tree/master/hwy/contrib/sort)
