## Replace Branches with Selection 用选择替换分支 {#sec:BranchlessSelection}

Some branches could be effectively eliminated by executing both parts of the branch and then selecting the right result. An example of code when such transformation might be profitable is shown in [@lst:ReplaceBranchesWithSelection]. If TMA suggests that the `if (cond)` branch has a very high number of mispredictions, you can try to eliminate the branch by doing the transformation shown on the right.

通过同时执行分支的两个部分然后选择正确的结果，可以有效地消除一些分支。 [@lst:ReplaceBranchesWithSelection] 中显示了此类转换可能有利可图的代码示例。如果TMA表明 `if (cond)` 分支有大量错误预测，您可以尝试通过执行右侧所示的转换来消除该分支。

Listing: Replacing Branches with Selection.
代码列表： 用选择替换分支。

~~~~ {#lst:ReplaceBranchesWithSelection .cpp}
int a;                                             int x = computeX();
if (cond) { /* frequently mispredicted */   =>     int y = computeY();
  a = computeX();                                  int a = cond ? x : y;
} else {                                           foo(a);
  a = computeY();
}
foo(a);
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

For the code on the right, the compiler can replace the branch that comes from the ternary operator, and generate a `CMOV` x86 instruction instead. A `CMOVcc` instruction checks the state of one or more of the status flags in the `EFLAGS` register (`CF, OF, PF, SF` and `ZF`) and performs a move operation if the flags are in a specified state or condition. A similar transformation can be done for floating-point numbers with `FCMOVcc` and `VMAXSS/VMINSS` instructions. In the ARM ISA, there is the `CSEL` (conditional selection) instruction, but also `CSINC` (select and increment), `CSNEG` (select and negate), and a few other conditional instructions.

对于右侧的代码，编译器可以替换三元运算符产生的分支，并生成一条x86指令 `CMOV`。`CMOVcc` 指令检查 `EFLAGS` 寄存器中一个或多个状态标志（`CF`、`OF`、`PF`、`SF` 和 `ZF`）的状态，如果这些标志处于指定的状态或条件，则执行移动操作。对于浮点数，可以使用 `FCMOVcc` 和 `VMAXSS/VMINSS` 指令进行类似的转换。在ARM指令集体系结构 (ISA) 中，除了 `CSEL`（条件选择）指令外，还有 `CSINC`（选择并递增）、`CSNEG`（选择并取反）以及其他一些条件指令。

Listing: Replacing Branches with Selection - x86 assembly code.
代码列表：用选择替换分支 - x86汇编代码。

~~~~ {#lst:ReplaceBranchesWithSelectionAsm .bash}
# original version              # branchless version
400504: test ebx,ebx            400537: mov eax,0x0
400506: je 400514               40053c: call <computeX> # compute x; a = x
400508: mov eax,0x0             400541: mov ebp,eax     # ebp = x
40050d: call <computeX>    =>   400543: mov eax,0x0
400512: jmp 40051e              400548: call <computeY> # compute y; a = y
400514: mov eax,0x0             40054d: test ebx,ebx    # test cond
400519: call <computeY>         40054f: cmovne eax,ebp  # override a with x if needed
40051e: mov edi,eax             400552: mov edi,eax
400521: call <foo>              400554: call <foo>
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

[@lst:ReplaceBranchesWithSelectionAsm] shows assembly listings for the original and the branchless version. In contrast with the original version, the branchless version doesn't have jump instructions. However, the branchless version calculates both `x` and `y` independently, and then selects one of the values and discards the other. While this transformation eliminates the penalty of a branch misprediction, it is doing more work than the original code. 

[@lst:ReplaceBranchesWithSelectionAsm] 显示了原始版本和无分支版本的汇编代码。与原始版本相比，无分支版本没有跳转指令。然而，无分支版本会独立计算 `x` 和 `y`，然后选择其中一个值并丢弃另一个。虽然这种转换消除了分支预测错误带来的性能损失，但它比原始代码执行了更多的工作。

We already know that the branch in the original version on the left is hard to predict. This is what motivated us to try a branchless version in the first place. In this example, the performance gain of this change depends on the characteristics of the `computeX` and `computeY` functions. If the functions are small[^1] and the compiler can inline them, then selection might bring noticeable performance benefits. If the functions are big[^2], it might be cheaper to take the cost of a branch mispredict than to execute both `computeX` and `computeY` functions. Ultimately, performance measurements always decide which version is better.

我们已经知道，左侧原始版本中的分支很难预测。这正是我们最初尝试无分支版本的原因。在这个例子中，这种更改带来的性能提升取决于 `computeX` 和 `computeY` 函数的特性。如果函数很小[^1]，编译器可以内联它们，那么选择可能会带来显著的性能提升。如果函数很大[^2]，那么承担分支预测错误的成本可能比同时执行 `computeX` 和 `computeY` 函数更划算。最终，性能指标总是决定哪个版本更好。

Take a look at [@lst:ReplaceBranchesWithSelectionAsm] one more time. On the left, a processor can predict, for example, that the `je 400514` branch will be taken, speculatively call `computeY`, and start running code from the function `foo`. Remember, branch prediction happens many cycles before we know the actual outcome of the branch. By the time we start resolving the branch, we could be already halfway through the `foo` function, despite it is still speculative. If we are correct, we've saved a lot of cycles. If we are wrong, we have to take the penalty and start over from the correct path. In the latter case, we don't gain anything from the fact that we have already completed a portion of `foo`, it all must be thrown away. If the mispredictions occur too often, the recovering penalty outweighs the gains from speculative execution.

再看一下 [@lst:ReplaceBranchesWithSelectionAsm]。在左侧，处理器可以预测，例如，会执行 `je 400514` 分支，前瞻性地调用 `computeY`，然后开始执行函数 `foo` 中的代码。记住，分支预测发生在我们知道分支实际结果之前很多周期。当我们开始解析分支时，即使仍在前瞻执行，我们可能已经执行了 `foo` 函数的一半。如果我们预测正确，就节省了很多周期。如果我们预测错误，就必须承受惩罚，并从正确的路径重新开始。在后一种情况下，我们已经完成了 `foo` 函数的一部分，但这部分代码必须全部丢弃，没有任何意义。如果预测错误过于频繁，恢复的惩罚将超过前瞻执行带来的收益。

With conditional selection, it is different. There are no branches, so the processor doesn't have to speculate. It can execute `computeX` and `computeY` functions in parallel. However, it cannot start running the code from `foo` until it computes the result of the `CMOVNE` instruction since `foo` uses it as an argument (data dependency). When you use conditional select instructions, you convert a control flow dependency into a data flow dependency. 

使用条件选择则有所不同。由于没有分支，处理器无需进行前瞻。它可以并行执行 `computeX` 和 `computeY` 函数。但是，由于 `foo` 使用了 `CMOVNE` 指令的结果作为参数（数据依赖），因此处理器必须先计算出 `foo` 的结果才能开始执行 `foo` 的代码。使用条件选择指令时，您将控制流依赖转换为数据流依赖。

To sum it up, for small `if-else` statements that perform simple operations, conditional selects can be more efficient than branches, but only if the branch is hard to predict. So don't force the compiler to generate conditional selects for every conditional statement. For conditional statements that are always correctly predicted, having a branch instruction is likely an optimal choice, because you allow the processor to speculate (correctly) and run ahead of the actual execution. And don't forget to measure the impact of your changes.

总而言之，对于执行简单操作的小型 `if-else` 语句，条件选择可能比分支更高效，但前提是分支难以预测。因此，不要强制编译器为每个条件语句都生成条件选择。对于总是能正确预测的条件语句，使用分支指令可能是最佳选择，因为它允许处理器（正确地）进行前瞻，并在实际执行之前运行。此外，不要忘记评估您的更改的影响。

Without profiling data, compilers don't have visibility into the misprediction rates. As a result, they usually prefer to generate branch instructions by default. Compilers are conservative at using selection and may resist generating `CMOV` instructions even in simple cases. Again, the tradeoffs are complicated, and it is hard to make the right decision without the runtime data.[^4] Starting from Clang-17, the compiler now honors a `__builtin_unpredictable` hint for the x86 target, which indicates to the compiler that a branch condition is unpredictable. It can help influence the compiler's decision but does not guarantee that the `CMOV` instruction will be generated. Here’s an example of how to use `__builtin_unpredictable`:

如果没有性能轮廓分析数据，编译器就无法了解预测错误率。因此，编译器通常默认倾向于生成分支指令。编译器在使用选择操作时较为保守，即使在简单的情况下也可能拒绝生成 `CMOV` 指令。同样，权衡利弊的情况很复杂，如果没有运行时数据，很难做出正确的决定[^4]。从 Clang-17开始，编译器现在会响应以x86目标的`__builtin_unpredictable` 提示，该提示会告知编译器分支条件是不可预测的。它可以帮助影响编译器的决策，但并不能保证一定会生成 `CMOV` 指令。以下是如何使用 `__builtin_unpredictable` 的示例：

```cpp
int a;
if (__builtin_unpredictable(cond)) { 
  a = computeX();
} else {
  a = computeY();
}
```

[^1]: Just a handful of instructions that can be completed in a few cycles. 只需几条指令，几个周期即可完成。
[^2]: More than twenty instructions that take more than twenty cycles. 超过二十条指令，耗时超过二十个周期。
[^4]: Hardware-based PGO (see [@sec:secPGO]) will be a huge step forward here. 基于硬件的PGO（参见 [@sec:secPGO]）将是这方面的一大进步。
