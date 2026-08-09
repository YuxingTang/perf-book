

## Basic Block Alignment 基本块对齐

Sometimes performance can significantly change depending on the offset at which instructions are laid out in memory. Consider a simple function presented in [@lst:LoopAlignment] along with the corresponding machine code when compiled with `-O3 -march=core-avx2 -fno-unroll-loops` (loop unrolling is disabled to illustrate the idea).

有时，指令在存储中的布局偏移量会对性能产生显著影响。考虑一下 [@lst:LoopAlignment] 中介绍的一个简单函数，以及使用 `-O3 -march=core-avx2 -fno-unroll-loops` 编译时对应的机器代码（为了便于说明，此处禁用了循环展开）。

Listing: Basic block alignment
代码列表：基本块对齐

~~~~ {#lst:LoopAlignment .cpp}
void benchmark_func(int* a) {    │ 00000000004046a0 <_Z14benchmark_funcPi>:
  for (int i = 0; i < 32; ++i)   │ 4046a0: mov rax,0xffffffffffffff80
    a[i] += 1;                   │ 4046a7: vpcmpeqd ymm0,ymm0,ymm0
}                                │ 4046ab: nop DWORD [rax+rax+0x0]
                                 │ 4046b0: vmovdqu ymm1,[rdi+rax+0x80] # loop begins
                                 │ 4046b9: vpsubd ymm1,ymm1,ymm0
                                 │ 4046bd: vmovdqu [rdi+rax+0x80],ymm1
                                 │ 4046c6: add rax,0x20
                                 │ 4046ca: jne 4046b0                  # loop ends
                                 │ 4046cc: vzeroupper 
                                 │ 4046cf: ret 
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The code itself is reasonable, but its layout is not perfect (see Figure @fig:Loop_default). Instructions that correspond to the loop are highlighted in yellow. Thick boxes denote cache line borders. Cache lines are 64 bytes long. 

代码本身尚合理，但布局并不完美（参见图 @fig:Loop_default）。与循环对应的指令以黄色高亮显示。粗框表示缓存行边界。缓存行长度为64字节。

<div id="fig:LoopLayout">
![default layout默认布局](../../img/cpu_fe_opts/LoopAlignment_Default.png){#fig:Loop_default width=100%}

![improved layout改进布局](../../img/cpu_fe_opts/LoopAlignment_Better.png){#fig:Loop_better width=100%}

Two different code layouts for the loop in [@lst:LoopAlignment]. [@lst:LoopAlignment]中循环的两种不同代码布局。
</div>

Notice that the loop spans multiple cache lines: it begins on the cache line `0x80-0xBF` and ends in the cache line `0xC0-0xFF`. To fetch instructions that are executed in the loop, a processor needs to read two cache lines. These kinds of situations sometimes cause performance problems for the CPU Frontend, especially for the small loops like those presented in [@lst:LoopAlignment].

请注意，该循环跨越了多个缓存行：它从缓存行 `0x80-0xBF` 开始，到缓存行 `0xC0-0xFF` 结束。为了获取循环中执行的指令，处理器需要读取两个缓存行。这种情况有时会导致CPU前端性能下降，尤其对于像 [@lst:LoopAlignment] 中所示的小型循环而言。

To fix this, we can shift the loop instructions forward by 16 bytes using a single NOP instruction so that the whole loop will reside in one cache line. Figure @fig:Loop_better shows the effect of doing this with the NOP instruction highlighted in blue. 

为了解决这个问题，我们可以使用一条NOP指令将循环指令向前移动16个字节，从而使整个循环位于一个缓存行中。图 @fig:Loop_better 展示了这样做的效果，其中NOP指令以蓝色高亮显示。

Interestingly, the performance impact is visible even if you run nothing but this hot loop in a microbenchmark. It is somewhat puzzling since the amount of code is tiny and it shouldn't saturate the L1 I-cache size on any modern CPU. The reason for the better performance of the layout in Figure @fig:Loop_better is not trivial to explain and will involve a fair amount of microarchitectural details, which we don't discuss in this book. Interested readers can find more information in the related article on the Easyperf blog.[^1]

有趣的是，即使只在微基准测试中运行这个热点循环，也能观察到性能影响。这有点令人费解，因为代码量很小，在任何现代CPU上都不应该占用L1指令缓存的全部容量。图 @fig:Loop_better 中布局性能更佳的原因并非显而易见，需要深入解释，涉及相当多的微体系结构细节，本书不作赘述。感兴趣的读者可以参阅Easyperf博客上的相关文章[^1]。

By default, the LLVM compiler recognizes loops and aligns them at 16B boundaries, as we saw in Figure @fig:Loop_default. To reach the desired code placement for our example, as shown in Figure @fig:Loop_better, you can use the `-mllvm -align-all-blocks=5` option that will align every basic block in an object file at a 32 bytes boundary. However, I do not recommend using this and similar options as they affect the code layout of all the functions in the translation unit. There are other less intrusive options.

默认情况下，LLVM编译器会识别循环并将其对齐到16字节边界，如图 @fig:Loop_default 所示。为了达到示例所需的代码布局效果（如图 @fig:Loop_better 所示），可以使用 `-mllvm -align-all-blocks=5` 选项，该选项会将目标文件中的每个基本块对齐到32字节边界。但是，我不建议使用此选项及类似选项，因为它们会影响翻译单元中所有函数的代码布局。还有其他一些影响较小的选项。

A recent addition to the LLVM compiler is the new `[[clang::code_align()]]` loop attribute, which allows developers to specify the alignment of a loop in the source code. This gives a very fine-grained control over machine code layout. The following code shows how the new Clang attribute can be used to align a loop at a 64 bytes boundary: 

LLVM编译器最近新增了 `[[clang::code_align()]]` 循环属性，允许开发者指定源代码中循环的对齐方式。这使得开发者能够对机器代码布局进行非常精细的控制。以下代码展示了如何使用新的 Clang属性将循环对齐到64字节边界：

```cpp
void benchmark_func(int* a) {
  [[clang::code_align(64)]]
  for (int i = 0; i < 32; ++i)
    a[i] += 1;
}
```

Before this attribute was introduced, developers had to resort to some less practical solutions like injecting `asm(".align 64;")` statements of inline assembly in the source code.

在引入此属性之前，开发人员不得不采用一些不太实用的解决方案，例如在源代码中注入 `asm(".align 64;")` 语句来执行内联汇编。

Even though CPU architects work hard to minimize the impact of machine code layout, there are still cases when code placement (alignment) can make a difference in performance. Machine code layout is also one of the main sources of noise in performance measurements. It makes it harder to distinguish a real performance improvement or regression from the accidental one, that was caused by the change in the code layout.

尽管CPU体系结构设计师努力将机器代码布局的影响降到最低，但代码放置位置（对齐）仍然会对性能产生影响。机器代码布局也是性能测量中的主要噪声来源之一。它使得区分真正的性能提升或下降与由代码布局变化引起的偶然性能变化变得更加困难。

[^1]: "Code alignment issues" "代码对齐问题" -- [https://easyperf.net/blog/2018/01/18/Code_alignment_issues](https://easyperf.net/blog/2018/01/18/Code_alignment_issues)
