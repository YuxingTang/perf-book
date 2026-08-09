## Basic Block {#sec:BasicBlock}

A *basic block* is a sequence of instructions with a single entry and a single exit. Figure @fig:BasicBlock shows a simple example of a basic block, where the `MOV` instruction is an entry, and `JA` is an exit instruction. Basic block `BB1` is a *predecessor* of basic block `BB2` if the control flow can go from `BB1` to `BB2`. Similarly, basic block `BB3` is a *successor* of basic block `BB2` if the control flow can go from `BB2` to `BB3`. While a basic block can have one or many predecessors and successors, no instruction in the middle can enter or exit a basic block.

基本块是具有单个入口和单个出口的指令序列。图 @fig:BasicBlock 展示了一个简单的基本块示例，其中 `MOV` 指令是入口指令，`JA` 指令是出口指令。如果控制流可以从 `BB1` 流向 `BB2`，则基本块 `BB1` 是基本块 `BB2` 的*前驱predecessor*。类似地，如果控制流可以从 `BB2` 流向 `BB3`，则基本块 `BB3` 是基本块 `BB2` 的*后继successor*。虽然一个基本块可以有一个或多个前驱和后继，但中间的指令不能进入或退出基本块。

![Basic Block of assembly instructions. 汇编指令的基本块.](../../img/cpu_fe_opts/BasicBlock.png){#fig:BasicBlock width=50% }

It is guaranteed that every instruction in the basic block will be executed only once. This is an important property that is leveraged by many compiler transformations. For example, it greatly reduces the problem of control flow graph analysis and transformations since, for some classes of problems, we can treat all instructions in the basic block as one entity.

基本块中的每条指令都保证只执行一次。这是许多编译器转换所利用的重要特性。例如，它极大地简化了控制流图分析和转换的难度，因为对于某些类型的问题，我们可以将基本块中的所有指令视为一个整体。
