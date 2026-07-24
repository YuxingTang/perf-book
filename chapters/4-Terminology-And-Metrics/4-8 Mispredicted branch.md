## Mispredicted Branch 分支预测错误 {#sec:BbMisp}

Modern CPUs try to predict the outcome of a conditional branch instruction (taken or not taken). For example, when the processor sees code like this:

现代CPU会尝试预测条件分支指令的结果（分支发生或不发生）。例如，当处理器看到如下代码时：

```bash
dec eax
jz .zero
# eax is not 0
...
zero:
# eax is 0
```

In the above example, the `jz` instruction is a conditional branch. To increase performance, modern processors will try to guess the outcome every time they see a branch instruction. This is called *Speculative Execution* which we discussed in [@sec:SpeculativeExec]. The processor will speculate that, for example, the branch will not be taken and will execute the code that corresponds to the situation when `eax is not 0`. However, if the guess is wrong, this is called *branch misprediction*, and the CPU is required to undo all the speculative work that it has done recently.

在上面的例子中，`jz` 指令是一条条件分支指令。为了提高性能，现代处理器会在每次遇到分支指令时尝试猜测其结果。这被称为*前瞻执行Speculative Execution*，我们在 [@sec:SpeculativeExec] 中讨论过。处理器会前瞻，例如，分支指令不会发生跳转，并执行与 `eax 不为 0` 的情况对应的代码。然而，如果猜测错误，则称为*分支预测错误misprediction*，CPU需要撤销最近执行的所有前瞻性操作。

A mispredicted branch typically involves a penalty between 10 and 25 clock cycles. First, all the instructions that were fetched and executed based on the incorrect prediction need to be flushed from the pipeline. After that, some buffers may require cleanup to restore the state from where the bad speculation started.

一次分支预测错误通常会导致10到25个时钟周期的惩罚。首先，所有基于错误预测而提取和执行的指令都需要从流水线中清除。之后，可能需要清理一些缓冲区，以恢复到错误预测开始时的状态。

Linux `perf` users can check the number of branch mispredictions by running:

Linux `perf` 用户可以通过运行以下命令来检查分支预测错误的数量：

```bash
$ perf stat -e branches,branch-misses -- a.exe
   358209  branches
    14026  branch-misses #    3,92% of all branches        
# or simply do:
$ perf stat -- a.exe
```