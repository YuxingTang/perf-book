## Basic Block Placement 基本块放置 {#sec:secLIKELY}

Suppose we have a hot path in the program that has some error handling code (`coldFunc`) in between:

假设程序中有一条热门路径，其间穿插了一些错误处理代码（`coldFunc`）：

```cpp
// hot path
if (cond)
  coldFunc();
// hot path again
```
Figure @fig:BBLayout shows two possible physical layouts for this snippet of code. Figure @fig:BB_default is the layout most compilers will emit by default, given no hints are provided. The layout that is shown in Figure @fig:BB_better can be achieved if we invert the condition `cond` and place hot code as fall through.

图 @fig:BBLayout 展示了这段代码片段的两种可能的物理布局。图 @fig:BB_default 是大多数编译器在未提供任何提示的情况下默认生成的布局。图 @fig:BB_better 所示的布局可以通过反转条件 `cond` 并将热代码作为直接向下传递（fall through）代码放置来实现。

<div id="fig:BBLayout">
![default layout默认布局](../../img/cpu_fe_opts/BBLayout_Default.png){#fig:BB_default width=50%}
![improved layout改进布局](../../img/cpu_fe_opts/BBLayout_Better.png){#fig:BB_better width=50%}

Two versions of machine code layout for the snippet of code above. 上述代码片段的两种机器码布局版本
</div>

Which layout is better? Well, it depends on whether `cond` is usually true or false. If `cond` is usually true, then we would better choose the default layout because otherwise, we would be doing two jumps instead of one. Also, if `coldFunc` is a relatively small function, we would want to have it inlined. However, in this particular example, we know that `coldFunc` is an error-handling function and is likely not executed very often. By choosing layout @fig:BB_better, we maintain fall through between hot pieces of the code and convert the taken branch into not taken one.

哪种布局更好？这取决于 `cond` 通常为真还是假。如果 `cond` 通常为真，那么我们最好选择默认布局，否则我们需要执行两次跳转而不是一次。此外，如果 `coldFunc` 是一个相对较小的函数，我们希望将其内联。然而，在本例中，我们知道 `coldFunc` 是一个错误处理函数，并且可能执行频率不高。通过选择布局 @fig:BB_better，我们可以保持热代码片段之间的直接向下传递，并将跳转分支转换为不跳转分支。

There are a few reasons why the layout presented in Figure @fig:BB_better performs better. First of all, the layout in Figure @fig:BB_better makes better use of the instruction and $\mu$op-cache (DSB, see [@sec:uarchFE]). With all hot code contiguous, there is no cache line fragmentation: all the cache lines in the L1 I-cache are used by hot code. The same is true for the $\mu$op-cache since it caches based on the underlying code layout as well. Secondly, taken branches are also more expensive for the fetch unit. The Frontend of a CPU fetches contiguous aligned blocks of bytes, usually 16, 32, or 64 bytes, depending on the architecture. For every taken branch, bytes in a fetch block after the jump instruction and before the branch target are unused. This reduces the maximum effective fetch throughput. Finally, on some architectures, not-taken branches are fundamentally cheaper than taken. For instance, Intel Skylake CPUs can execute two untaken branches per cycle but only one taken branch every two cycles.[^2]

图 @fig:BB_better 所示的布局性能更佳的原因有几个。首先，图 @fig:BB_better 中的布局更好地利用了指令缓存和微操作缓存μop-cache（DSB，参见 [@sec:uarchFE]）。由于所有热代码都是连续的，因此不会出现缓存行碎片：L1指令缓存中的所有缓存行都被热代码使用。对于微操作缓存μop-cache来说也是如此，因为它也是基于底层代码布局进行缓存的。其次，对于取指单元来说，执行跳转分支的开销也更大。CPU前端会取指连续对齐的字节块，通常为16、32或64字节，具体取决于体系结构。对于每个跳转的分支指令，在跳转指令之后到分支目标之前的取指块中的字节都未被使用。这会降低最大有效取指吞吐率。最后，在某些体系结构上，未跳转分支的成本远低于跳转的分支。例如，Intel Skylake CPU每1个周期可以执行2个未跳转分支，但每2个周期只能执行1个跳转分支。[^2]

To suggest a compiler to generate an improved version of the machine code layout, you can provide a hint using `[[likely]]`	and `[[unlikely]]` attributes, which have been available since C++20. The code that uses this hint will look like this:

要建议编译器生成改进的机器代码布局，可以使用 `[[likely]]` 和 `[[unlikely]]` 属性提供提示，这些属性自C++20起可用。使用此提示的代码如下所示：

```cpp
// hot path
if (cond) [[unlikely]] 
  coldFunc();
// hot path again
```

In the code above, the `[[unlikely]]` hint will instruct the compiler that `cond` is unlikely to be true, so the compiler should adjust the code layout accordingly. Prior to C++20, developers could have used the [`__builtin_expect`](https://llvm.org/docs/BranchWeightMetadata.html#builtin-expect)[^3] construct. They usually created `LIKELY` wrapper hints to make the code more readable. For example:

在上面的代码中，`[[unlikely]]` 提示会告诉编译器 `cond` 不太可能为真，因此编译器应该相应地调整代码布局。在C++20之前，开发者可以使用 [`__builtin_expect`](https://llvm.org/docs/BranchWeightMetadata.html#builtin-expect)[^3]构造函数。他们通常会创建 `LIKELY` 包装提示来提高代码的可读性。例如：

```cpp
#define LIKELY(EXPR)   __builtin_expect((bool)(EXPR), true)
#define UNLIKELY(EXPR) __builtin_expect((bool)(EXPR), false)
// hot path
if (UNLIKELY(cond)) // NOT 
  coldFunc();
// hot path again
```

Optimizing compilers will not only improve code layout when they encounter "likely/unlikely" hints. They will also leverage this information in other places. For example, when the `[[unlikely]]` attribute is applied, the compiler will prevent inlining `coldFunc` since it now knows that it is unlikely to be executed often and it's more beneficial to optimize it for size, i.e., just leave a `CALL` to this function. 

优化编译器不仅会在遇到“可能/不可能（likely/unlikely）”提示时改进代码布局，还会将这些信息用于其他方面。例如，当应用 `[[unlikely]]` 属性时，编译器会阻止内联 `coldFunc`，因为它知道该函数不太可能经常执行，因此优化代码尺寸（即只保留对该函数的 `CALL` 调用）更为有利。

Inserting the `[[likely]]` attribute is also possible for a switch statement as presented in [@lst:BuiltinSwitch]. Using this hint, a compiler will be able to reorder code a little bit differently and optimize the hot switch for faster processing of `ADD` instructions.

`[[likely]]` 属性也可以插入到switch语句中，如 [@lst:BuiltinSwitch] 所示。使用此提示，编译器可以对代码进行一些细微的重新排序，并优化热switch以加快 `ADD` 指令的处理速度。

Listing: Likely attribute used in a switch statement
代码列表：在switch语句中使用Likely属性

~~~~ {#lst:BuiltinSwitch .cpp}
for (;;) {
  switch (instruction) {
               case NOP: handleNOP(); break;
    [[likely]] case ADD: handleADD(); break;
               case RET: handleRET(); break;
    // handle other instructions
  }
}
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

[^2]: However, there is a special small loop optimization that allows very small loops to have one taken branch per cycle. 然而，有一种特殊的小循环优化，允许非常小的循环在每个周期内只有一个跳转分支。
[^3]: More about builtin-expect here: 更多关于builtin-expect的信息请参见： [https://llvm.org/docs/BranchWeightMetadata.html#builtin-expect](https://llvm.org/docs/BranchWeightMetadata.html#builtin-expect).
[^10]: C++ standard `[[likely]]` attribute: C++标准的 `[[likely]]` 属性： [https://en.cppreference.com/w/cpp/language/attributes/likely](https://en.cppreference.com/w/cpp/language/attributes/likely).
