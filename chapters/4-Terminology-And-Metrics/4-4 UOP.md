

## Micro-operations 微操作 {#sec:sec_UOP}

Microprocessors with the x86 architecture translate complex CISC instructions into simple RISC microoperations, abbreviated as $\mu$ops. A simple register-to-register addition instruction such as `ADD rax, rbx` generates only one $\mu$op, while a more complex instruction like `ADD rax, [mem]` may generate two: one for loading from the `mem` memory location into a temporary (unnamed) register, and one for adding it to the `rax` register. The instruction `ADD [mem], rax` generates three $\mu$ops: one for loading from memory, one for adding, and one for storing the result back to memory.

采用x86体系结构的微处理器会将复杂的CISC指令转换为简单的RISC微操作，简称为微操作$\mu$ops。例如，一条简单的寄存器到寄存器的加法指令 `ADD rax, rbx` 只会生成一个 $\mu$ops，而一条更复杂的指令 `ADD rax, [mem]` 可能会生成2个微操作：一个用于将数据从 `mem` 内存地址加载到临时（未命名）寄存器，另一个用于将结果加到 `rax` 寄存器。指令 `ADD [mem], rax` 会生成3个微操作 $\mu$ops：一个用于从内存加载数据，一个用于加法运算，还有一个用于将结果存储回内存。

The main advantage of splitting instructions into micro-operations is that $\mu$ops can be executed:

将指令拆分为微操作的主要优势在于可以按照下面的方式来执行微操作：

\lstset{linewidth=10cm}

* **Out of order**: consider the `PUSH rbx` instruction, which decrements the stack pointer by 8 bytes and then stores the source operand on the top of the stack. Suppose that `PUSH rbx` is "cracked" into two dependent micro-operations after decoding:
* **乱序执行**：考虑 `PUSH rbx` 指令，该指令将栈指针减去8字节，然后将源操作数存储到栈顶。假设解码后，`PUSH rbx` 被“破开”为两个相关的微操作：
  ```
  SUB rsp, 8
  STORE [rsp], rbx
  ```
  Often, a function prologue saves multiple registers by using multiple `PUSH` instructions. In our case, the next `PUSH` instruction can start executing after the `SUB` $\mu$op of the previous `PUSH` instruction finishes and doesn't have to wait for the `STORE` $\mu$op, which can now execute asynchronously.
  
  通常，函数序言会使用多个 `PUSH` 指令来保存多个寄存器。在我们的例子中，下一个 `PUSH` 指令可以在前一个 `PUSH` 指令的 `SUB` 微操作完成后开始执行，而无需等待 `STORE` 微操作，后者现在可以异步执行。

* **In parallel**: consider `HADDPD xmm1, xmm2` instruction, which will sum up (reduce) two double-precision floating-point values from `xmm1` and `xmm2` and store two results in `xmm1` as follows: 
* **并行**：考虑指令 `HADDPD xmm1, xmm2`，该指令将 `xmm1` 和 `xmm2` 中的两个双精度浮点值相加（归约reduce），并将两个结果存储在 `xmm1` 中，如下所示：
  ```
  xmm1[63:0] = xmm2[127:64] + xmm2[63:0]
  xmm1[127:64] = xmm1[127:64] + xmm1[63:0]
  ```
  One way to microcode this instruction would be to do the following: 1) reduce `xmm2` and store the result in `xmm_tmp1[63:0]`, 2) reduce `xmm1` and store the result in `xmm_tmp2[63:0]`, 3) merge `xmm_tmp1` and `xmm_tmp2` into `xmm1`. Three $\mu$ops in total. Notice that steps 1) and 2) are independent and thus can be done in parallel.
  一种实现此指令的微代码方法是：1）归约 `xmm2` 并将结果存储在 `xmm_tmp1[63:0]` 中；2）归约 `xmm1` 并将结果存储在 `xmm_tmp2[63:0]` 中；3）将 `xmm_tmp1` 和 `xmm_tmp2` 合并到 `xmm1` 中。总共有三个μ操作。请注意，步骤1）和2）是独立的，因此可以并行执行。

Even though we were just talking about how instructions are split into smaller pieces, sometimes, $\mu$ops can also be fused together. There are two types of fusion in modern x86 CPUs:
尽管我们刚才讨论的是如何将指令拆分成更小的单元，但有时，μ操作也可以融合在一起。现代x86 CPU中有两种融合方式：

* **Microfusion**: fuse $\mu$ops from the same machine instruction. Microfusion can only be applied to two types of combinations: memory write operations and read-modify operations. For example:
* **微融合**：融合来自同一条机器指令的μ操作。微融合只能应用于两种类型的组合：内存写入操作和读修改操作。例如：

  ```bash
  add    eax, [mem]
  ```
  There are two $\mu$ops in this instruction: 1) read the memory location `mem`, and 2) add it to `eax`. With microfusion, two $\mu$ops are fused into one at the decoding step.
  这条指令包含两个μ操作：1）读取内存地址 `mem`，以及2）将其添加到 `eax`。通过微融合，两个μ操作在译码步骤中融合为一个。
  
* **Macrofusion**: fuse $\mu$ops from different machine instructions. The decoders can fuse arithmetic or logic instructions with a subsequent conditional jump instruction into a single compute-and-branch $\mu$op in certain cases. For example:
* **宏融合**：融合来自不同机器指令的μ操作。在某些情况下，译码器可以将算术或逻辑指令与后续的条件跳转指令融合为一个计算并分支的微操作。例如：

  ```bash
  .loop:
    dec rdi
    jnz .loop
  ```
  With macrofusion, two $\mu$ops from the `DEC` and `JNZ` instructions are fused into one. The Zen4 microarchitecture also added support for DIV/IDIV and NOP macrofusion [@amd_zen4, sections 2.9.4 and 2.9.5].
  通过宏融合，来自 `DEC` 和 `JNZ` 指令的两个微操作被融合为一个。Zen4微体系结构还增加了对DIV/IDIV 和NOP宏融合的支持 [@amd_zen4，第2.9.4和2.9.5节]。

\lstset{linewidth=\textwidth}

Both micro- and macrofusion save bandwidth in all stages of the pipeline, from decoding to retirement. The fused operations share a single entry in the reorder buffer (ROB). The capacity of the ROB is utilized better when a fused $\mu$op uses only one entry. Such a fused ROB entry is later dispatched to two different execution ports but is retired again as a single unit. Readers can learn more about $\mu$op fusion in [@fogMicroarchitecture].

从译码到完成结束，微融合和宏融合都能在流水线的各个阶段节省带宽。融合后的操作共享重排序缓冲区(ROB)中的单个条目。当融合后的宏操作仅使用一个条目时，ROB的容量利用率更高。这种融合后的ROB条目随后会被分派到两个不同的执行端口，但最终会作为一个整体被回收。读者可以在 [@fogMicroarchitecture] 中了解更多关于 微操作融合的信息。

To collect the number of issued, executed, and retired $\mu$ops for an application, you can use Linux `perf` as follows:
要收集应用程序已发出、已执行和已回收的微操作的数量，可以使用Linux `perf` 命令，如下所示：

```bash
$ perf stat -e uops_issued.any,uops_executed.thread,uops_retired.slots -- ./a.exe
  2856278  uops_issued.any             
  2720241  uops_executed.thread
  2557884  uops_retired.slots
```

The way instructions are split into micro-operations may vary across CPU generations. Usually, a lower number of $\mu$ops used for an instruction means that hardware has better support for it and is likely to have lower latency and higher throughput. For the latest Intel and AMD CPUs, the vast majority of instructions generate only one $\mu$op. Latency, throughput, port usage, and the number of $\mu$ops for x86 instructions on recent microarchitectures can be found at the [uops.info](https://uops.info/table.html)[^1] website.
指令拆分成微操作的方式可能因CPU代次不同而异。通常，指令使用的微操作μop数越少，意味着硬件对该指令的支持越好，延迟可能越低，吞吐率可能越高。对于最新的Intel和AMD CPU，绝大多数指令仅生成一个μ操作。您可以在 [uops.info](https://uops.info/table.html)[^1] 网站上找到最新微体系结构上x86指令的延迟、吞吐率、端口使用情况以及微操作μop数。

[^1]: x86 instruction latency and throughput x86指令延迟和吞吐率 - [https://uops.info/table.html](https://uops.info/table.html)
