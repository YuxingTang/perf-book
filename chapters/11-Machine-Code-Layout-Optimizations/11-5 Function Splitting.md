## Function Splitting 函数拆分 

The idea behind function splitting is to separate the hot code from the cold. Such transformation is also often called *function outlining*. This optimization is beneficial for relatively big functions with a complex control flow graph and large chunks of cold code inside a hot path. An example of code when such transformation might be profitable is shown in [@lst:FunctionSplitting1]. To remove cold basic blocks from the hot path, we cut and paste them into a new function and create a call to it.

函数拆分的理念是将热代码与冷代码分离。这种转换也常被称为“函数大纲化”。对于控制流图复杂、热路径中包含大量冷代码的大型函数，这种优化尤为有效。[@lst:FunctionSplitting1] 展示了一个适合进行这种转换的代码示例。为了从热路径中移除冷代码块，我们将其剪切并粘贴到一个新函数中，然后调用该新函数。

Listing: Function splitting: cold code outlined to the new functions.
代码列表：函数拆分：将冷代码外移转换为新函数

~~~~ {#lst:FunctionSplitting1 .cpp}
void foo(bool cond1,                void foo(bool cond1,
         bool cond2) {                       bool cond2) {
  // hot path                         // hot path
  if (cond1) {                        if (cond1) {
    /* cold code (1) */                 cold1(); 
  }                                   }
  // hot path                         // hot path
  if (cond2) {              =>        if (cond2) {
    /* cold code (2) */                 cold2(); 
  }                                   }
}                                   }
                                    void cold1() __attribute__((noinline)) 
                                    { /* cold code (1) */ }
                                    void cold2() __attribute__((noinline))
                                    { /* cold code (2) */ }
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Notice, that we disable the inlining of cold functions by using the `noinline` attribute. Because without it, a compiler may decide to inline it, which will effectively undo our transformation. Alternatively, we could apply the `[[unlikely]]` macro (see [@sec:secLIKELY]) on both `cond1` and `cond2` branches to convey to the compiler that inlining `cold1` and `cold2` functions is not desired.

请注意，我们使用 `noinline` 属性禁用了冷函数的内联。因为如果没有这个属性，编译器可能会决定内联这些冷函数，这将有效地撤销我们的转换。或者，我们可以对 `cond1` 和 `cond2` 分支都应用 `[[unlikely]]` 宏（参见 [@sec:secLIKELY]），以告知编译器不希望内联 `cold1` 和 `cold2` 函数。

<div id="fig:FunctionSplitting">
![default layout默认布局](../../img/cpu_fe_opts/FunctionSplitting_Default.png){#fig:FuncSplit_default width=50%}
![improved layout改进布局](../../img/cpu_fe_opts/FunctionSplitting_Improved.png){#fig:FuncSplit_better width=50%}

Splitting cold code into a separate function. 将冷代码拆分成一个单独的函数。
</div>

Figure @fig:FunctionSplitting gives a graphical representation of this transformation. In the improved layout, we left just a `CALL` instruction inside the hot path, the next hot instruction will likely reside in the same cache line as the previous one. This improves the utilization of CPU Frontend data structures such as I-cache and $\mu$op-cache.

图 @fig:FunctionSplitting 以图形方式展示了这种转换。在改进后的布局中，我们只在热路径中保留了一条 `CALL` 指令，下一条热指令很可能与前一条热指令位于同一缓存行。这提高了CPU 前端数据结构（例如：指令缓存I-cache和微操作缓存μop-cache）的利用率。

Outlined functions should be created outside of the `.text` segment, for example in `.text.cold`. This improves memory footprint if the function is never called since it won't be loaded into memory at runtime.

应该在 `.text` 段之外创建已外移的函数，例如在 `.text.cold` 中。如果函数从未被调用，则由于运行时不会将其加载到内存中，因此可以减少内存占用。