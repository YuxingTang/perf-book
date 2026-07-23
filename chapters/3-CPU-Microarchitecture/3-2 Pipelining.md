## Pipelining 流水线

Pipelining is a foundational technique used to make CPUs fast, wherein multiple instructions overlap during their execution. Pipelining in CPUs drew inspiration from automotive assembly lines. The processing of instructions is divided into stages. The stages operate in parallel, working on different parts of different instructions. DLX is a relatively simple architecture designed by John L. Hennessy and David A. Patterson in 1994. As defined in [@Hennessy], it has a 5-stage pipeline which consists of:

流水线是提高CPU速度的基础技术，它允许多条指令在执行过程中重叠执行。CPU流水线的设计灵感来源于汽车装配线。指令的处理被划分为多个阶段。这些阶段并行运行，分别处理不同指令的不同部分。DLX是 John L. Hennessy和David A. Patterson于1994年设计的一种相对简单的体系结构。根据 [@Hennessy] 的定义，它具有一个5级流水线，包括：

1. Instruction fetch (IF)
2. Instruction decode (ID)
3. Execute (EXE)
4. Memory access (MEM)
5. Write back (WB)

1. 指令获取/取指(IF: Instruction Fetch)
2. 指令译码/译码(ID: Instruction Decode)
3. 执行(EXE: EXEcute)
4. 内存访问(MEM: MEMory access)
5. 结果写回/写回(WB: Write Back)

![Simple 5-stage pipeline diagram.简单的五级流水线示意图。](../../img/uarch/Pipelining.png){#fig:Pipelining width=80%}

Figure @fig:Pipelining shows an ideal pipeline view of the 5-stage pipeline CPU. In cycle 1, instruction x enters the IF stage of the pipeline. In the next cycle, as instruction x moves to the ID stage, the next instruction in the program enters the IF stage, and so on. Once the pipeline is full, as in cycle 5 above, all pipeline stages of the CPU are busy working on different instructions. Without pipelining, the instruction `x+1` couldn't start its execution until after instruction `x` had finished its work.

图 @fig:Pipelining 展示了5级流水线CPU的理想流水线视图。在第一个周期，指令x进入流水线的IF阶段。在下一个周期中，当指令x进入ID阶段时，程序中的下一条指令进入IF阶段，依此类推。一旦流水线被填满（如：上文第5个周期所示），CPU的所有流水线阶段都将忙于处理不同的指令。如果没有流水线技术，指令 `x+1` 必须等到指令 `x` 完成其工作后才能开始执行。

Modern high-performance CPUs have multiple pipeline stages, often ranging from 10 to 20 (sometimes more), depending on the architecture and design goals. This involves a much more complicated design than a simple 5-stage pipeline introduced earlier. For example, the decode stage may be split into several new stages. We may also add new stages before the execute stage to buffer decoded instructions and so on.

现代高性能CPU具有多个流水线阶段，通常为10到20个（有时更多），具体取决于体系结构和设计目标。这比之前介绍的简单5级流水线设计要复杂得多。例如，译码阶段可以拆分为多个新阶段。我们还可以在执行阶段之前添加新阶段来缓冲译码后的指令等等。

The *throughput* of a pipelined CPU is defined as the number of instructions that complete and exit the pipeline per unit of time. The *latency* for any given instruction is the total time through all the stages of the pipeline. Since all the stages of the pipeline are linked together, each stage must be ready to move to the next instruction in lockstep. The time required to move an instruction from one stage to the next defines the basic machine *cycle* or clock for the CPU. The value chosen for the clock for a given pipeline is defined by the slowest stage of the pipeline. CPU hardware designers strive to balance the amount of work that can be done in a stage as this directly affects the frequency of operation of the CPU.

流水线CPU的*吞吐率throughtput*定义为单位时间内完成并退出流水线的指令数量。任何给定指令的*延迟latency*是指其通过流水线所有阶段的总时间。由于流水线的各个阶段相互连接，每个阶段都必须做好准备，以锁步的方式执行下一条指令。指令从一个阶段移动到下一个阶段所需的时间定义了CPU的基本机制*周期cycle*或时钟。给定流水线的时钟值由流水线中最慢的阶段决定。CPU硬件设计人员力求平衡每个阶段可以完成的工作量，因为这直接影响CPU的运行频率。

In real implementations, pipelining introduces several constraints that limit the nicely-flowing execution illustrated in Figure @fig:Pipelining. Pipeline hazards prevent the ideal pipeline behavior, resulting in stalls. The three classes of hazards are structural hazards, data hazards, and control hazards. Luckily for the programmer, in modern CPUs, all classes of hazards are handled by the hardware.

在实际实现中，流水线引入了一些限制约束，从而影响了图@fig:Pipelining所示的流畅执行。流水线冲突会阻碍理想的流水线行为，导致停顿。冲突分为三类：结构冲突、数据冲突和控制冲突。幸运的是，在现代CPU中，所有类型的冲突都由硬件处理。

\lstset{linewidth=10cm}

* **Structural hazards**: are caused by resource conflicts, i.e., when there are two instructions competing for the same resource. An example of such a hazard is when two 32-bit addition instructions are ready to execute in the same cycle, but there is only one execution unit available in that cycle. In this case, we need to choose which one of the two instructions to execute and which one will be executed in the next cycle. To a large extent, they could be eliminated by replicating hardware resources, such as using multiple execution units, instruction decoders, multi-ported register files, etc. However, this could potentially become quite expensive in terms of silicon area and power.

* **结构冲突**：由资源冲突引起，即：两条指令竞争同一资源。例如，当两条32位加法指令准备在同一周期执行时，如果该周期只有一个执行单元可用，就会出现结构冲突。在这种情况下，我们需要选择执行哪条指令，以及在下一个周期执行哪条指令。在很大程度上，可以通过复制硬件资源来消除结构冲突，例如使用多个执行单元、指令译码器、多端口寄存器文件等。然而，这可能会显著增加芯片面积和功耗。

* **Data hazards**: are caused by data dependencies in the program and are classified into three types:

* **数据冲突**：由程序中的数据依赖关系引起，分为三种类型：

  A *read-after-write* (RAW) hazard requires a dependent read to execute after a write. It occurs when instruction `x+1` reads a source before previous instruction `x` writes to the source, resulting in the wrong value being read. CPUs implement data forwarding from a later stage of the pipeline to an earlier stage (called "*bypassing*") to mitigate the penalty associated with the RAW hazard. The idea is that results from instruction `x` can be forwarded to instruction `x+1` before instruction `x` is fully completed. If we take a look at the example:

  *写后读*(RAW: Read-After-Write)冲突要求在写入操作之后执行依赖的读取操作。当指令 `x+1` 在前一条指令 `x` 写入源数据之前读取该源数据时，就会发生这种情况，导致读取到错误的值。CPU实现了从流水线后期阶段到前阶段的数据转发（称为“*旁路bypass*”），以减轻RAW冲突带来的性能损失。其思想是，指令 `x` 的结果可以在指令 `x` 完全执行完成之前转发给指令 `x+1`。例如：

  ```
  R1 = R0 ADD 1
  R2 = R1 ADD 2
  ```

  There is a RAW dependency for register R1. If we take the value directly after the addition `R0 ADD 1` is done (from the `EXE` pipeline stage), we don't need to wait until the `WB` stage finishes (when the value will be written to the register file). Bypassing helps to save a few cycles. The longer the pipeline, the more effective bypassing becomes.

  寄存器R1存在RAW依赖关系。如果我们在加法指令 `R0 ADD 1` 执行完毕后（在 `EXE` 流水线阶段）直接获取其值，则无需等待 `WB` 阶段完成（此时值将被写入寄存器文件）。这种旁路机制有助于节省一些周期。流水线越长，旁路机制的效果就越好。

  A *write-after-read* (WAR) hazard requires a dependent write to execute after a read. It occurs when an instruction writes a register before an earlier instruction reads the source, resulting in the wrong new value being read. A WAR hazard is not a true dependency and can be eliminated by a technique called *register renaming*. It is a technique that abstracts logical registers from physical registers. CPUs support register renaming by keeping a large number of physical registers. Logical (*architectural*) registers, the ones that are defined by the ISA, are just aliases over a wider register file. With such decoupling of the architectural state, solving WAR hazards is simple: we just need to use a different physical register for the write operation. For example:

  *读后写*(WAR: Write-After-Read)冲突是指在读取操作之后执行依赖写入操作。当一条指令在之前的指令读取源数据之前写入寄存器时，就会发生WAR冲突，导致读取到错误的新值。WAR冲突并非真正的依赖关系，可以通过一种称为*寄存器重命名register renaming*的技术来消除。寄存器重命名是一种将逻辑寄存器与物理寄存器分离的技术。CPU通过维护大量的物理寄存器来支持寄存器重命名。逻辑（*体系结构architectural*）寄存器，即由指令集体系结构(ISA)定义的寄存器，只是对更广泛的寄存器文件的别名。通过这种体系结构状态的解耦，解决WAR冲突就变得很简单：我们只需要为写入操作使用不同的物理寄存器即可。例如：

  ```
  ; machine code, WAR hazard机器码，WAR冲突 ; after register renaming寄存器重命名后 
  ; (architectural registers)(结构寄存器)   ; (physical registers)(物理寄存器) 
  R1 = R0 ADD 1                  =>       R101 = R100 ADD 1
  R0 = R2 ADD 2                           R103 = R102 ADD 2
  ```

  In the original assembly code, there is a WAR dependency for register `R0`. For the code on the left, we cannot reorder the execution of the instructions, because it could leave the wrong value in `R1`. However, we can leverage our large pool of physical registers to overcome this limitation. To do that we need to rename all the occurrences of the `R0` register starting from the write operation (`R0 = R2 ADD 2`) and below to use a free register. After renaming, we give these registers new names that correspond to physical registers, say `R103`. By renaming registers, we eliminated a WAR hazard in the initial code, and we can safely execute the two operations in any order.

  在原始汇编代码中，寄存器 `R0` 存在写后写入(WAR)依赖关系。对于左侧的代码，我们不能重新排列指令的执行顺序，因为这可能会导致 `R1` 的值错误。然而，我们可以利用大量的物理寄存器来克服这个限制。为此，我们需要将从写入操作 `R0 = R2 ADD 2` 开始的所有 `R0` 寄存器的出现位置重命名，使其使用空闲寄存器。重命名后，我们将这些寄存器赋予与物理寄存器对应的新名称，例如 `R103`。通过重命名寄存器，我们消除了初始代码中的写后写入冲突，现在可以安全地以任意顺序执行这两个操作。

  A *write-after-write* (WAW) hazard requires a dependent write to execute after a write. It occurs when an instruction writes to a register before an earlier instruction writes to the same register, resulting in the wrong value being stored. WAW hazards are also eliminated by register renaming, allowing both writes to execute in any order while preserving the correct final result. Below is an example of eliminating WAW hazards.

  *写后写*(WAW: Write-After-Write)冲突是指在写入操作之后执行有依赖关系的一个写入操作。当一条指令在之前的指令写入同一寄存器之前写入该寄存器时，就会发生写后写入冲突，导致存储的值错误。通过寄存器重命名，还可以消除WAW冲突，允许两个写入操作以任意顺序执行，同时保持最终结果的正确性。以下是一个消除WAW冲突的示例。

  ```
  ; machine code, WAW hazard机器码，WAR冲突 ; after register renaming寄存器重命名后
  (architectural registers)(结构寄存器)     (physical registers)(物理寄存器)
  R1 = R0 ADD 1                  =>       R101 = R100 ADD 1
  R2 = R1 SUB R3  ; RAW                   R102 = R101 SUB R103 ; RAW
  R1 = R0 MUL 3   ; WAW and WAR           R104 = R100 MUL 3
  ```

  You will see similar code in many production programs. In our example, `R1` keeps the temporary result of the `ADD` operation. Once the `SUB` instruction is complete, `R1` is immediately reused to store the result of the `MUL` operation. The original code on the left features all three types of data hazards. There is a RAW dependency over `R1` between `ADD` and `SUB`, and it must survive register renaming. Also, we have WAW and WAR hazards over the same `R1` register for the `MUL` operation. Again, we need to rename registers to eliminate those two hazards. Notice that after register renaming we have a new destination register (`R104`) for the `MUL` operation. Now we can safely reorder `MUL` with the other two operations.

  你会在许多生产程序中看到类似的代码。在我们的示例中，`R1` 寄存器用于保存 `ADD` 操作的临时结果。`SUB` 指令执行完毕后，`R1` 寄存器会立即被重用，用于存储 `MUL` 操作的结果。左侧的原始代码包含了所有三种类型的数据冒险。`ADD` 和 `SUB` 指令之间存在对 `R1` 寄存器的 RAW 依赖关系，这种依赖关系必须在寄存器重命名后仍然存在。此外，`MUL` 操作还存在对同一个 `R1` 寄存器的 WAW 和 WAR 冒险。同样，我们需要重命名寄存器来消除这两种冒险。请注意，寄存器重命名后，`MUL` 操作有了一个新的目标寄存器（`R104`）。现在我们可以安全地将 `MUL` 操作与其他两个操作重新排序。

* **Control hazards**: are caused due to changes in the program flow. They arise from pipelining branches and other instructions that change the program flow. The branch condition that determines the direction of the branch (taken vs. not taken) is resolved in the execute pipeline stage. As a result, the fetch of the next instruction cannot be pipelined unless the control hazard is eliminated. Techniques such as dynamic branch prediction and speculative execution described in the next section are used to mitigate control hazards.

* **控制冲突**：是由程序流程的变化引起的。它们源于流水线分支和其他会改变程序流程的指令。决定分支方向（执行或不执行）的分支条件在执行流水线阶段解决。因此，除非控制冲突被消除，否则无法对下一条指令进行流水线式读取。下一节中描述的动态分支预测和推测执行等技术用于缓解控制冒险。

\lstset{linewidth=\textwidth}
