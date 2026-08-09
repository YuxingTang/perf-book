## Chapter Summary 本章小结 {.unlisted .unnumbered}

\markright{Summary小结}

A summary of CPU Frontend optimizations is presented in Table {@tbl:CPU_FE_OPT}.
表 {@tbl:CPU_FE_OPT} 总结了CPU前端优化。

--------------------------------------------------------------------------
Transform  How transformed?  Why helps?    Works best for        Done by
转换        如何转换？          为什么有效？   最适用于                执行者
---------  ----------------  ------------  --------------------  ---------
Basic      maintain          not taken     any code, especially  compiler
block      fall through      branches are  with a lot of 
placement  hot code          cheaper;      branches
                             better cache
                             utilization
基本块放置   保持热点代码垂直    不跳转的分支更  任何代码，特别是具有大量   编译器
                             简单：更好的缓  分支
                             存利用率

Basic      shift the hot     better cache  hot loops             compiler
block      code using NOPs   utilization 
alignment
基本块对齐   用空操作NOP移动     更好的缓存利用   热点循环               编译器
            热点代码           率

Function   split cold        better cache  functions with        compiler
splitting  blocks of code    utilization   complex CFG when 
           and place them                  there are big blocks 
           in separate                     of cold code between 
           functions                       hot parts
函数拆分    将代码中的冷块       更好的缓存利用  具有复杂控制流图CFG的    编译器
           分割出来并放置在     率            函数，当在热路径之间有
           独立的函数中                      大块的冷代码

Function   group hot         better cache  many small            linker
reorder    functions         utilization   hot functions
           together
函数重排序   将热函数分组在一起   更好的缓存利用  许多小尺寸热函数         连接器
                              率
--------------------------------------------------------------------------

Table: Summary of CPU Frontend optimizations. CPU前端优化总结。 {#tbl:CPU_FE_OPT}

* Code layout improvements are often underestimated and overlooked. CPU Frontend performance issues like I-cache and ITLB misses represent a large portion of wasted cycles, especially for applications with large codebases. But even small- and medium-sized applications can benefit from optimizing the machine code layout.
* It is usually the best option to use LTO, PGO, BOLT, and similar tools to improve the code layout if you can come up with a set of typical use cases for your application. For large applications, it is the only practical option.

* 代码布局改进经常被低估和忽视。CPU前端性能问题，例如：指令缓存（I-cache）和指令地址翻译缓冲（ITLB）未命中，会造成大量CPU周期浪费，尤其对于代码库庞大的应用程序而言更是如此。但即使是中小型应用程序也能从优化机器代码布局中获益。
* 如果你能为你的应用程序找到一系列典型用例，那么使用：LTO、PGO、BOLT等工具来优化代码布局通常是最佳选择。对于大型应用程序而言，这几乎是唯一可行的方案。

\sectionbreak
