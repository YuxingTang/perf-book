## Function Reordering 函数重排序 

Following the principles described in previous sections, hot functions can be grouped together to further improve the utilization of caches in the CPU Frontend. When hot functions are grouped, they start sharing cache lines, which reduces the *code footprint*, the total number of cache lines a CPU needs to fetch.

遵循前几节所述的原则，可以将热函数分组，以进一步提高CPU前端缓存的利用率。热函数分组后，它们开始共享缓存行，从而减少*代码占用空间（code footprint）*，即CPU需要获取的缓存行总数。

Figure @fig:FunctionGrouping gives a graphical representation of reordering hot functions `foo`, `bar`, and `zoo`. The arrows on the image show the most frequent call pattern, i.e., `foo` calls `zoo`, which in turn calls `bar`. In the default layout (see Figure @fig:FuncGroup_default), hot functions are not adjacent to each other with some cold functions placed between them. Thus the sequence of two function calls (`foo` → `zoo` → `bar`) requires four cache line reads.[^4]

图 @fig:FunctionGrouping 以图形方式展示了对热函数 `foo`、`bar` 和 `zoo` 进行重排序的过程。图中的箭头显示了最常见的调用模式，即 `foo` 调用 `zoo`，而 `zoo` 又调用 `bar`。在默认布局中（参见图 @fig:FuncGroup_default），热函数彼此并不相邻，它们之间夹杂着一些冷函数。因此，两个函数调用（`foo` → `zoo` → `bar`）需要读取4条缓存行[^4]。

We can rearrange the order of the functions such that hot functions are placed close to each other (see Figure @fig:FuncGroup_better). In the improved version, the code of the `foo`, `bar`, and `zoo` functions fits in three cache lines. Also, notice that function `zoo` now is placed between `foo` and `bar` according to the order in which function calls are being made. When we call `zoo` from `foo`, the beginning of `zoo` is already in the I-cache.

我们可以重新排列函数的顺序，使热门函数彼此靠近（参见图 @fig:FuncGroup_better）。在改进后的版本中，`foo`、`bar` 和 `zoo` 函数的代码可以放入3行缓存。此外，请注意，根据函数调用的顺序，函数 `zoo` 现在位于 `foo` 和 `bar` 之间。当我们从 `foo` 调用 `zoo` 时，`zoo` 的开头部分已经存在于I-cache中。

<div id="fig:FunctionGrouping">
![default layout默认布局](../../img/cpu_fe_opts/FunctionGrouping_Default.png){#fig:FuncGroup_default width=50%}
![improved layout改进布局](../../img/cpu_fe_opts/FunctionGrouping_Better.png){#fig:FuncGroup_better width=50%}

Reordering hot functions. 重排热点函数
</div>

Similar to previous optimizations, function reordering improves the utilization of I-cache and $\mu$op-cache. This optimization works best when there are many small hot functions. 

与之前的优化类似，函数重排序可以提高指令缓存(I-cache)和微操作缓存μop-cache的利用率。当存在大量小型热点函数时，此优化效果最佳。

The linker is responsible for laying out all the functions of the program in the resulting binary output. While developers can try to reorder functions in a program themselves, there is no guarantee of the desired physical layout. For decades people have been using linker scripts to achieve this goal. This is still the way to go if you are using the GNU linker. The Gold linker (`ld.gold`) has an easier approach to this problem. To get the desired ordering of functions in the binary with the Gold linker, you can first compile the code with the `-ffunction-sections` flag, which will put each function into a separate section. Then use [`--section-ordering-file=order.txt`](https://manpages.debian.org/unstable/binutils/x86_64-linux-gnu-ld.gold.1.en.html) option to provide a file with a sorted list of function names that reflects the desired final layout. The same feature exists in the LLD linker, which is a part of the LLVM compiler infrastructure and is accessible via the `--symbol-ordering-file` option.

链接器负责在生成的二进制输出中布局程序的所有函数。虽然开发人员可以尝试自行重排序程序中的函数，但无法保证获得理想的物理布局。几十年来，人们一直使用链接器脚本来实现这一目标。如果您使用的是GNU链接器，这仍然是一种方法。Gold链接器 (`ld.gold`) 提供了一种更简便的方法来解决这个问题。要使用Gold链接器在二进制文件中获得所需的函数顺序，您可以先使用 `-ffunction-sections` 标志编译代码，该标志会将每个函数放入一个单独的节中。然后使用 [`--section-ordering-file=order.txt`]（https://manpages.debian.org/unstable/binutils/x86_64-linux-gnu-ld.gold.1.en.html）选项提供一个包含函数名称排序列表的文件，以反映所需的最终布局。LLD链接器也具有相同的功能，它是LLVM编译器基础设施的一部分，可通过 `--symbol-ordering-file` 选项访问。

An interesting approach to solving the problem of grouping hot functions was introduced in 2017 by engineers from Meta. They implemented a tool called [HFSort](https://github.com/facebook/hhvm/tree/master/hphp/tools/hfsort)[^1], that generates the section ordering file automatically based on profiling data [@HfSort]. Using this tool, they observed a 2\% performance speedup of large distributed cloud applications like Facebook, Baidu, and Wikipedia. HFSort has been integrated into Meta's HHVM, LLVM BOLT, and LLD linker[^2]. Since then, the algorithm has been superseded first by HFSort+, and most recently by Cache-Directed Sort (CDSort[^3]), with more improvements for workloads with a large code footprint.

2017年，Meta的工程师提出了一种解决热点函数分组问题的有趣方法。他们实现了一个名为 [HFSort](https://github.com/facebook/hhvm/tree/master/hphp/tools/hfsort)[^1]的工具，该工具基于性能轮廓分析数据自动生成节排序文件（[@HfSort]）。使用该工具，他们观察到Facebook、百度和维基百科等大型分布式云应用程序的性能提升了2%。HFSort已集成到Meta的HHVM、LLVM BOLT和LLD链接器中[^2]。此后，该算法先是被HFSort+取代，最近又被缓存定向排序(CDSort[^3])取代，后者针对代码量大的工作负载进行了更多改进。

[^1]: HFSort - [https://github.com/facebook/hhvm/tree/master/hphp/tools/hfsort](https://github.com/facebook/hhvm/tree/master/hphp/tools/hfsort)
[^2]: HFSort in LLD - [https://github.com/llvm-project/lld/blob/master/ELF/CallGraphSort.cpp](https://github.com/llvm-project/lld/blob/master/ELF/CallGraphSort.cpp)
[^3]: Cache-Directed Sort in LLVM LLVM中的缓存定向排序 - [https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Utils/CodeLayout.cpp](https://github.com/llvm/llvm-project/blob/main/llvm/lib/Transforms/Utils/CodeLayout.cpp)
[^4]: Also, functions located in shared libraries do not participate in the careful layout of machine code. 同样，位于共享库中的函数不参与机器代码的精心布局。
