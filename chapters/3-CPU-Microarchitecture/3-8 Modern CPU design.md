## Modern CPU Design 现代CPU设计

To see how all the concepts we talked about in this chapter are used in practice, let's take a look at the implementation of Intel’s 12th-generation core, Golden Cove, which became available in 2021. This core is used as the P-core inside the Alder Lake and Sapphire Rapids platforms. Figure @fig:Goldencove_diag shows the block diagram of the Golden Cove core. Notice that this section only describes a single core, not the entire processor. So, we will skip the discussion about frequencies, core counts, L3 caches, core interconnects, memory latency, and bandwidth.

为了了解本章讨论的所有概念在实践中的应用，我们来看看Intel公司第十二代Core处理器核Golden Cove的实现，它于2021年发布。该核心被用作Alder Lake和Sapphire Rapids平台中的性能核心（P-Core）。图 @fig:Goldencove_diag 展示了Golden Cove核心的框图。请注意，本节仅描述单个核心，而非整个处理器。因此，我们将略过对频率、核心数量、L3缓存、核心互连、内存延迟和带宽的讨论。

![Block diagram of a CPU core in the Intel Golden Cove Microarchitecture. Intel公司Golden Cove微体系结构CPU核心的模块图](../../img/uarch/goldencove_block_diagram.png){#fig:Goldencove_diag width=100%}

The core is split into an in-order frontend that fetches and decodes x86 instructions into $\mu$ops[^9] and a 6-wide superscalar, out-of-order backend. The Golden Cove core supports 2-way SMT. It has a 32KB first-level instruction cache (L1 I-cache), and a 48KB first-level data cache (L1 D-cache). The L1 caches are backed up by a unified 1.25MB (2MB in server chips) L2 cache. The L1 and L2 caches are private to each core. At the end of this section, we also take a look at the TLB hierarchy.

该核心分为一个顺序执行的前端和一个6指令宽的超标量乱序执行的后端。顺序执行的前端负责x86指令的获取和译码，生成微操作μop[^9]。Golden Cove核心支持双路同时多线程(2-way SMT)。它拥有32KB的一级指令缓存(L1 I-cache)和一个48KB的一级数据缓存(L1 D-cache)。L1缓存由一个统一的1.25MB（服务器级芯片为2MB）二级缓存(L2 cache)做后备。L1和L2缓存是每个核心私有的。在本节末尾，我们还将介绍TLB层次结构。

### CPU Frontend CPU前端 {#sec:uarchFE}

The CPU Frontend consists of several functional units that fetch and decode instructions from memory. Its main purpose is to feed prepared instructions to the CPU Backend, which is responsible for the actual execution of instructions.

CPU前端由多个功能单元组成，这些单元负责从内存中获取指令和译码指令。其主要目的是将准备好的指令提供给CPU后端，后端负责指令的实际执行。

Technically, instruction fetch is the first stage to execute an instruction. But once a program reaches a steady state, the branch predictor unit (BPU) steers the work of the CPU Frontend. That is indicated by the arrow that goes from the BPU to the instruction cache. The BPU predicts the target of all branch instructions and steers the next instruction fetch based on this prediction.

从技术上讲，指令获取是执行指令的第一阶段。但是，一旦程序达到稳定状态，分支预测单元(BPU: Branch Predictor Unit)就会引导CPU前端的工作。这由从BPU指向指令缓存的箭头表示。BPU预测所有分支指令的目标地址，并根据此预测来引导下一次指令获取。

The heart of the BPU is a branch target buffer (BTB) with 12K entries containing information about branches and their targets. This information is used by the prediction algorithms. Every cycle, the BPU generates the next fetch address and passes it to the CPU Frontend.

BPU的核心是一个包含12K个条目的分支目标缓冲(BTB: Branch Target Buffer)，其中包含有关分支及其目标地址的信息。这些信息供预测算法使用。每个周期，BPU都会生成下一个获取地址并将其传递给CPU前端。

The CPU Frontend fetches 32 bytes per cycle of x86 instructions from the L1 I-cache. This is shared among the two threads (if SMT is enabled), so each thread gets 32 bytes every other cycle. These are complex, variable-length x86 instructions. First, the pre-decode stage determines and marks the boundaries of the variable instructions by inspecting the chunk. In x86, the instruction length can range from 1 to 15 bytes. This stage also identifies branch instructions. The pre-decode stage moves up to 6 instructions (also referred to as *macroinstructions*) to the Instruction Queue that is split between the two threads. The instruction queue also supports a macro-op fusion unit that detects when two macroinstructions can be fused into a single micro-operation ($\mu$op). This optimization saves bandwidth in the rest of the pipeline.

CPU前端每个周期从L1指令缓存中获取32字节的x86指令。如果启用了同时多线程(SMT)，则两个线程共享这32字节的数据，因此实际上每个线程是每隔一个周期获得32字节。这些是复杂的、可变长度的x86指令。首先，预译码阶段通过检查数据块来确定并标记可变指令的边界。在x86中，指令长度范围为1到15字节。此阶段还会识别分支指令。预译码阶段将最多6条指令（也称为*宏指令macro instruction*）移动到分配给两个线程的指令队列中。指令队列还支持一个宏操作macro-op融合单元，用于检测何时可以将两条宏指令融合为一条微操作（μop: micro-operation）。此优化可以节省流水线其余部分的带宽。

Later, up to six pre-decoded instructions are sent from the Instruction Queue to the decoder unit every cycle. The two SMT threads alternate every cycle to access this interface. The 6-way decoder converts the complex macro-Ops into fixed-length $\mu$ops. Decoded $\mu$ops are queued into the Instruction Decode Queue (IDQ), labeled as "$\mu$op Queue" on the diagram.

之后，每个周期最多有6条预译码指令从指令队列发送到译码器单元。两个SMT线程每个周期交替访问此接口。6路译码器将复杂的宏操作转换为固定长度的微操作(μop)。译码后的微操作μop会被放入指令译码队列(IDQ: Instruction Decode Queue) 中，图中标记为“微操作队列(μop Queue)”。

A major performance-boosting feature of the Frontend is the $\mu$op Cache. Also, you could often see people call it Decoded Stream Buffer (DSB). The motivation is to cache the macro-ops to $\mu$ops conversion in a separate structure that works in parallel with the L1 I-cache. When the BPU generates a new address to fetch, the $\mu$op Cache is also checked to see if the $\mu$ops translations are already available. Frequently occurring macro-ops will hit in the $\mu$op Cache, and the pipeline will avoid repeating the expensive pre-decode and decode operations for the 32-byte bundle. The $\mu$op Cache can provide eight $\mu$ops per cycle and can hold up to 4K entries.

前端的一项主要性能提升特性是微操作μop缓存。有时人们也会称其为译码流缓冲区(DSB: Decoded Stream Buffer)。其目的是将宏操作到微操作的转换缓存在一个与L1指令缓存并行工作的独立结构中。当BPU生成新的取地址时，也会检查微操作μop缓存，以确定相应的微操作μop转换是否已存在。频繁出现的微操作μop会命中微操作μop缓存，从而避免流水线重复执行32字节数据包的昂贵预译码和译码操作。微操作μop缓存每个周期可以提供8个微操作μop，最多可容纳4K个条目。

Some very complicated instructions may require more $\mu$ops than decoders can handle. $\mu$ops for such instruction are served from the Microcode Sequencer (MSROM). Examples of such instructions include hardware operation support for string manipulation, encryption, synchronization, and others. Also, MSROM keeps the microcode operations to handle exceptional situations like branch misprediction (which requires a pipeline flush), floating-point assist (e.g., when an instruction operates with a denormalized floating-point value), and others. MSROM can push up to 4 $\mu$ops per cycle into the IDQ.

一些非常复杂的指令可能需要比译码器处理能力之外的更多微操作（$\mu$ops）。这类指令的微操作由微码序列器（MSROM: Microcode Sequencer ROM）提供。此类指令的示例包括用于字符串操作、加密、同步等的硬件操作支持。此外，MSROM还保存微码操作以处理诸如分支预测错误（需要流水线刷新）、浮点辅助（例如，当指令操作非规范化浮点值时）等特殊情况。MSROM每个周期最多可以将4个微操作推送到指令译码队列（IDQ）中。

The Instruction Decode Queue (IDQ) provides the interface between the in-order frontend and the out-of-order backend. The IDQ queues up the $\mu$ops in order and can hold 144 $\mu$ops per logical processor in single thread mode, or 72 $\mu$ops per thread when SMT is active. This is where the in-order CPU Frontend finishes and the out-of-order CPU Backend starts.

指令解码队列(IDQ: Instruction Decode Queue)提供顺序前端和乱序后端之间的接口。IDQ按顺序对微操作μop进行排队，在单线程模式下每个逻辑处理器可容纳144个微操作μop，在SMT激活时每个线程可容纳72个微操作μop。顺序CPU前端在此结束，乱序CPU后端由此开始。

### CPU Backend CPU后端 {#sec:uarchBE}

The CPU Backend employs an OOO engine that executes instructions and stores results. I repeated a part of the diagram that depicts the Golden Cove OOO engine in Figure @fig:Goldencove_OOO.

CPU后端采用乱序执行引擎来执行指令并存储结果。我重复了图 @fig:Goldencove_OOO 中描述Golden Cove乱序执行引擎的部分图示。

The heart of the OOO engine is the 512-entry ReOrder Buffer (ROB). It serves a few purposes. First, it provides register renaming.[^5] There are only 16 general-purpose integer and 32 floating-point/SIMD architectural registers, however, the number of physical registers is much higher.[^1] Physical registers are located in a structure called the Physical Register File (PRF). There are separate PRFs for integer and floating-point/SIMD registers. The mappings from architecture-visible registers to the physical registers are kept in the register alias table (RAT).

乱序执行引擎的核心是512项重排序/再定序缓冲(ROB: ReOrder Buffer)。它有多种用途。首先，它提供寄存器重命名功能[^5]。体系结构中只有16个通用整数寄存器和32个浮点/SIMD寄存器，但物理寄存器的数量要多得多[^1]。物理寄存器位于一个名为物理寄存器文件(PRF: Physical Register File)的结构中。整数寄存器和浮点/SIMD寄存器分别使用不同的PRF。体系结构可见寄存器到物理寄存器的映射关系保存在寄存器别名表(RAT: Register Alias Table)中。

![Block diagram of the CPU Backend of the Intel Golden Cove Microarchitecture. Intel公司Golden Cove微体系结构CPU后端模块图](../../img/uarch/goldencove_OOO.png){#fig:Goldencove_OOO width=100%}

Second, the ROB allocates execution resources. When an instruction enters the ROB, a new entry is allocated and resources are assigned to it, mainly an execution unit and the destination physical register. The ROB can allocate up to 6 $\mu$ops per cycle.

其次，ROB分配执行资源。当一条指令进入ROB时，会分配一个新的条目，并为其分配资源，主要是执行单元和目标物理寄存器。ROB每个周期最多可以分配6个微操作μop。

Third, the ROB tracks speculative execution. When an instruction has finished its execution, its status gets updated and it stays there until the previous instructions finish. It's done that way because instructions must retire in program order. Once an instruction retires, its ROB entry is deallocated and the results of the instruction become visible. The retiring stage is wider than the allocation: the ROB can retire 8 instructions per cycle.

第三，ROB跟踪前瞻执行。当一条指令执行完毕后，其状态会被更新，并保持更新状态直到之前的指令执行完毕。这是因为指令必须按程序顺序完成执行（Retire）。指令完成执行后，其ROB条目会被释放，指令的执行结果也会随之显示。完成执行阶段比分配阶段更宽：ROB每个周期可以完成8条指令的执行。

There are certain operations that processors handle in a specific manner, often called idioms, which require no or less costly execution. Processors recognize such cases and allow them to run faster than regular instructions. Here are some of such cases:

处理器会以特定的方式处理某些操作，这些操作通常被称为惯用操作(idioms)，它们不需要或只需要极低的执行成本。处理器能够识别这些情况，并允许它们比常规指令更快地运行。以下是一些此类情况：

* **Zeroing**: to assign zero to a register, compilers often use `XOR / PXOR / XORPS / XORPD` instructions, e.g., `XOR EAX, EAX`, which are preferred by compilers instead of the equivalent `MOV EAX, 0x0` instruction as the XOR encoding uses fewer encoding bytes. Such zeroing idioms are not executed as any other regular instruction and are resolved in the CPU frontend, which saves execution resources. The instruction later retires as usual.
* **清零**：为了将寄存器赋值为零，编译器通常使用 `XOR / PXOR / XORPS / XORPD` 指令，例如 `XOR EAX, EAX`。编译器更倾向于使用XOR指令而不是等效的 `MOV EAX, 0x0` 指令，因为XOR编码使用的编码字节更少。此类清零指令的执行方式与其他常规指令不同，而是在CPU前端进行解析，从而节省执行资源。指令随后会像往常一样执行完毕。
* **Move elimination**: similar to the previous one, register-to-register `mov` operations, e.g., `MOV EAX, EBX`, are executed with zero cycle delay.
* **移动消除**：与上述清零类似，寄存器到寄存器的数据转移 `mov` 操作（例如 `MOV EAX, EBX`）的执行延迟为零个周期。
* **NOP instruction**: `NOP` is often used for padding or alignment purposes. It simply gets marked as completed without allocating it to the reservation station.
* **NOP空操作指令**：无操作的 `NOP` 指令通常用于填充或对齐。它只是被标记为已完成，而不会将其分配给保留单元。
* **Other bypasses**: CPU architects also optimized certain arithmetic operations. For example, multiplying any number by one will always yield the same number. The same goes for dividing any number by one. Multiplying any number by zero always yields zero, etc. Some CPUs can recognize such cases at runtime and execute them with shorter latency than regular multiplication or divide.
* **其他旁路**：CPU体系结构设计师还优化了某些算术运算。例如，任何数乘以1的结果始终是该数本身。同样，任何数除以1的结果也始终是该数本身。任何数乘以0的结果始终是0，以此类推。一些CPU可以在运行时识别这些情况，并以比常规乘法或除法更短的延迟执行它们。

The "Scheduler / Reservation Station" (RS) is the structure that tracks the availability of all resources for a given $\mu$op and dispatches the $\mu$op to an *execution port* once it is ready. An execution port is a pathway that connects the scheduler to its execution units. Each execution port may be connected to multiple execution units. When an instruction enters the RS, the scheduler starts tracking its data dependencies. Once all the source operands become available, the RS attempts to dispatch the $\mu$op to a free execution port. The RS has fewer entries[^4] than the ROB. It can dispatch up to 6 $\mu$ops per cycle.

“调度器Scheduler/保留站（RS: Reservation Station）”是一个结构，它跟踪给定微操作μop的所有可用资源，并在资源准备就绪后将该微操作μop分派到*执行端口（executioin port）*。一个执行端口是连接调度器和执行单元的路径。每个执行端口可以连接到多个执行单元。当一条指令进入RS时，调度器开始跟踪其数据依赖关系。一旦所有源操作数都可用，RS就会尝试将该微操作μop分派到空闲的执行端口。RS的条目数比ROB少[^4]。它每个周期最多可以**分派dispatch**6个微操作μop。

I repeated a part of the diagram that depicts the Golden Cove execution engine and Load-Store unit in Figure @fig:Goldencove_BE_LSU. There are 12 execution ports:

我重复了图 @fig:Goldencove_BE_LSU 中描绘Golden Cove执行引擎和加载-存储单元的部分图表。有12个执行端口：

* Ports 0, 1, 5, 6, and 10 provide integer (INT) operations, and some of them handle floating-point and vector (FP/VEC) operations.
* Ports 2, 3, and 11 are used for address generation (AGU) and for load operations. 
* Ports 4 and 9 are used for store operations (STD).
* Ports 7 and 8 are used for address generation.

* 端口0、1、5、6和10都提供整数(INT)运算，其中一些端口也处理浮点和向量(FP/VEC)运算。
* 端口2、3和11用于地址生成(AGU: Address Generation Unit)和加载(Load)操作。
* 端口4和9用于存储操作(STD: Store Date)。
* 端口7和8用于地址生成。

![Block diagram of the execution engine and the Load-Store unit in the Intel Golden Cove Microarchitecture. Intel公司Golden Cove微体系结构中执行引擎和加载-存储单元的模块图](../../img/uarch/goldencove_BE_LSU.png){#fig:Goldencove_BE_LSU width=100%}

Instructions that require memory operations are handled by the Load-Store unit (ports 2, 3, 11, 4, 9, 7, and 8) which we will discuss in the next section. If an operation does not involve loading or storing data, then it will be dispatched to the execution engine (ports 0, 1, 5, 6, and 10). Some instructions may require two $\mu$ops that must be executed on different execution ports, e.g., load and add.

需要内存操作的指令由加载与存储单元（LSU: Load-Store Unit）（端口2、3、11、4、9、7和8）处理，我们将在下一节中讨论。如果操作不涉及加载Load或存储Store数据，那么它将被分派到执行引擎（端口0、1、5、6和10）。某些指令可能需要两个必须在不同执行端口上执行的微操作，例如加载并执行加法（Load&Add）。

For example, an `Integer Shift` operation can go only to either port 0 or 6, while a `Floating-Point Divide` operation can only be dispatched to port 0. In a situation when a scheduler has to dispatch two operations that require the same execution port, one of them will have to be delayed.

例如，“整数移位”操作只能发送到端口0或6，而“浮点除法”操作只能发送到端口0。在调度器必须发送两个需要相同执行端口的操作情况下，其中一个操作必须被延迟。

The FP/VEC stack does floating-point scalar and *all* packed (SIMD) operations. For instance, ports 0, 1, and 5 can handle ALU operations of the following types: packed integer, packed floating-point, and scalar floating-point. Integer and Vector/FP register files are located separately. Operations that move values from the INT stack to FP/VEC and vice-versa (e.g., convert, extract, or insert) incur additional penalties.

FP/VEC堆栈执行浮点标量和*全*打包的单指令多数据(SIMD)操作。例如，端口0、1和5可以处理以下类型的ALU操作：压缩整数、压缩浮点和标量浮点。整数和向量/FP寄存器文件位于单独的位置。将值从INT堆栈移动到FP/VEC的操作（例如：转换、提取或插入）会产生额外的执行开销，反之亦然。

### Load-Store Unit 加载-存储单元 {#sec:uarchLSU}

The Load-Store Unit (LSU) is responsible for operations with memory. The Golden Cove core can issue up to three loads (three 256-bit or two 512-bit) by using ports 2, 3, and 11. AGU stands for Address Generation Unit, which is required to access a memory location. It can also issue up to two stores (two 256-bit or one 512-bit) per cycle via ports 4, 9, 7, and 8. STD stands for Store Data.

加载与存储单元(LSU)负责内存操作。Golden Cove核心可以通过使用端口2、3和11发出最多3个载入Load（3个256位或2个512 位）。AGU代表地址生成单元，访问内存位置需要它。它还可以通过端口4、9、7和8每个周期发出最多2个存储Store（2个256位或1个512位）。STD代表存储数据。

Notice that the AGU is required for both load and store operations to perform dynamic address calculation. For example, in the instruction `vmovss DWORD PTR [rsi+0x4],xmm0`, the AGU will be responsible for calculating `rsi+0x4`, which will be used to store data from xmm0.

请注意，加载Load和存储Store操作都需要AGU来执行动态地址计算。例如，在指令“vmovss DWORD PTR [rsi+0x4],xmm0”中，AGU将负责计算“rsi+0x4”，该值将用于存储来自xmm0的数据。

Once a load or a store leaves the scheduler, the LSU is responsible for accessing the data. Load operations save the fetched value in a register. Store operations transfer value from a register to a location in memory. LSU has a Load Buffer (also known as Load Queue) and a Store Buffer (also known as Store Queue); their sizes are not disclosed.[^2] Both Load Buffer and Store Buffer receive operations at dispatch from the scheduler.

一旦加载或存储离开调度程序，LSU就负责访问数据。加载操作将获取的值保存在寄存器中。存储操作将值从寄存器传输到内存中的某个位置。LSU有一个加载缓冲（Load Buffer又称为Load Queue）和一个存储缓冲（Store Buffer又称为Store Queue）；它们的大小没有公开[^2]。加载缓冲区和存储缓冲区都在调度器进行分派时接收操作。

When a memory load request comes, the LSU queries the L1 cache using a virtual address and looks up the physical address translation in the TLB. Those two operations are initiated simultaneously. The size of the L1 D-cache is 48KB. If both operations result in a hit, the load delivers data to the integer or floating-point register and leaves the Load Buffer. Similarly, a store would write the data to the data cache and exit the Store Buffer.

当内存加载请求到来时，LSU使用虚拟地址查询L1缓存，并在TLB中查找物理地址转换。这两个操作同时启动。 L1D高速缓存的大小为48KB。如果两个操作都命中，则加载会将数据传送到整数或浮点寄存器并离开加载缓冲区。类似地，存储会将数据写入数据缓存并退出存储缓冲区。

In case of an L1 miss, the hardware initiates a query of the (private) L2 cache tags. While the L2 cache is being queried, a 64-byte wide fill buffer (FB) entry is allocated, which will keep the cache line once it arrives. The Golden Cove core has 16 fill buffers. As a way to lower the latency, a speculative query is sent to the L3 cache in parallel with the L2 cache lookup. Also, if two loads access the same cache line, they will hit the same FB. Such two loads will be "glued" together and only one memory request will be initiated.

如果发生L1未命中，硬件将发起对（私有）L2缓存标签的查询。当查询二级缓存时，会分配一个64字节宽的填充缓冲区(FB: Fill Buffer)条目，一旦缓存行到达，该条目将保留内容。Golden Cove核心有16个填充缓冲区。作为降低延迟的一种方法，前瞻查询与L2缓存查找并行发送到L3缓存。此外，如果两个加载Load访问同一缓存行，它们将命中同一个FB。这样两个加载将被“粘合”在一起，并且只会发起一个内存请求。

In case the L2 miss is confirmed, the load continues to wait for the results of the L3 cache, which incurs much higher latency. From that point, the request leaves the core and enters the *uncore*, the term you may sometimes see in profiling tools. The outstanding misses from the core are tracked in the Super Queue (SQ, not shown on the diagram), which can track up to 48 uncore requests. In a scenario of L3 miss, the processor begins to set up a memory access. Further details are beyond the scope of this chapter.

如果确认L2未命中，Load操作将继续等待L3缓存的结果，这会导致更高的延迟。从那时起，请求离开核心并进入*非处理器核uncore*，您有时可能会在分析工具中看到该术语。来自核心的未命中未命中在超级队列（SQ: Super Queue，图中未显示）中进行跟踪，该队列最多可跟踪48个非核心请求。在L3未命中的情况下，处理器开始建立内存访问。更多细节超出了本章的范围。

When a store modifies a memory location, the processor needs to load the full cache line, change it, and then write it back to memory. If the address to write is not in the cache, it goes through a very similar mechanism as with loads to bring that data in. The store cannot be complete until the data is written to the cache hierarchy.

当存储Store操作修改内存位置时，处理器需要加载load完整的缓存行，更改它，然后将其写回内存。如果要写入的地址不在高速缓存中，则它会通过与加载非常相似的机制来将该数据带入。直到将数据写入高速缓存层次结构后，存储Store才能完成。

Of course, there are a few optimizations done for store operations as well. First, if we're dealing with a store or multiple adjacent stores (also known as *streaming stores*) that modify an entire cache line, there is no need to read the data first as all of the bytes will be clobbered anyway. So, the processor will try to combine writes to fill an entire cache line. If this succeeds no memory read operation is needed.

当然，存储Store操作也有一些优化。首先，如果要执行一个或多个相邻的存储操作（也称为*流式存储Stream Stores*），并且这些操作会修改整个缓存行，则无需先读取数据，因为所有字节都会被覆盖。因此，处理器会尝试合并写入操作以填充整个缓存行。如果合并成功，则无需进行内存读取操作。

Second, write combining enables multiple stores to be assembled and written further out in the cache hierarchy as a unit. So, if multiple stores modify the same cache line, only one memory write will be issued to the memory subsystem. All these optimizations are done inside the Store Buffer. A store instruction copies the data that will be written from a register into the Store Buffer. From there it may be written to the L1 cache or it may be combined with other stores to the same cache line. The Store Buffer capacity is limited, so it can hold requests for partial writing to a cache line only for some time. However, while the data sits in the Store Buffer waiting to be written, other load instructions can read the data straight from the store buffers (store-to-load forwarding). Also, the LSU supports store-to-load forwarding when there is an older store containing all of the load's bytes, and the store's data has been produced and is available in the store queue.

其次，写入合并使得多个存储操作可以作为一个整体被组合起来，并写入到缓存层次结构的更远位置。因此，如果多个存储操作修改了同一缓存行，则只会向内存子系统发出一次内存写入指令。所有这些优化都在存储缓冲中完成。存储指令会将要写入的数据从寄存器复制到存储缓冲区。然后，数据可能会被写入L1缓存，或者与其他写入同一缓存行的存储操作合并。存储缓冲区的容量有限，因此它只能暂时保存对缓存行的部分写入请求。然而，当数据位于存储缓冲区中等待写入时，其他加载Load指令可以直接从存储缓冲区读取数据（存储到加载旁路转发）。此外，当存在包含所有加载字节的旧存储指令，且存储指令的数据已生成并位于存储队列中时，LSU也支持存储到加载转发。

Finally, there are cases when we can improve cache utilization by using so-called *non-temporal* memory accesses. If we execute a partial store (e.g., we overwrite 8 bytes in a cache line), we need to read the cache line first. This new cache line will displace another line in the cache. However, if we know that we won't need this data again, then it would be better not to allocate space in the cache for that line. Non-temporal memory accesses are special CPU instructions that do not keep the fetched line in the cache and drop it immediately after use.

最后，在某些情况下，我们可以通过使用所谓的*非临时non-temporal*内存访问来提高缓存利用率。如果我们执行部分存储操作（例如：覆盖缓存行中的8个字节），则需要先读取该缓存行。这个新的缓存行会占用缓存中的另一行。但是，如果我们知道以后不再需要这些数据，那么最好不要为该行分配缓存空间。非临时内存访问是特殊的CPU指令，它们不会将读取到的缓存行保留在缓存中，而是在使用后立即将其释放。

During a typical program execution, there could be dozens of memory accesses in flight. In most high-performance processors, the order of load and store operations is not necessarily required to be the same as the program order, which is known as a _weakly ordered memory model_. For optimization purposes, the processor can reorder memory read and write operations. Consider a situation when a load runs into a cache miss and has to wait until the data comes from memory. The processor allows subsequent loads to proceed ahead of the load that is waiting for data. This allows later loads to finish before the earlier load and doesn't unnecessarily block the execution. Such load/store reordering enables memory units to process multiple memory accesses in parallel, which translates directly into higher performance.

在典型的程序执行过程中，可能会有数十个内存访问操作同时进行。在大多数高性能处理器中，加载和存储操作的顺序不一定必须与程序顺序一致，这被称为*弱有序内存模型（_weakly ordered memory moddel_）*。为了优化性能，处理器可以重新排列内存读写操作的顺序。例如，当加载操作遇到缓存未命中，必须等待数据从内存中返回时，处理器会允许后续的加载操作优先于正在等待数据的加载操作执行。这样，后面的加载操作就能在前面的加载操作之前完成，从而避免不必要的阻塞。这种加载/存储重排序机制使得内存单元能够并行处理多个内存访问操作，从而直接提升性能。

The LSU dynamically reorders operations, supporting both loads bypassing older loads and loads bypassing older non-conflicting stores. However, there are a few exceptions. Just like with dependencies through regular arithmetic instructions, there are memory dependencies through loads and stores. In other words, a load can depend on an earlier store and vice-versa. First of all, stores cannot be reordered with older loads:

LSU能够动态地重新排列操作顺序，支持加载操作绕过较早的加载操作，以及加载操作绕过较早的非冲突存储操作。然而，也存在一些例外情况。就像常规算术指令之间存在依赖关系一样，加载和存储操作之间也存在内存依赖关系。换句话说，一个加载操作可能依赖于较早的存储操作，反之亦然。最重要的是，存储Store不能与之前更老的加载Load乱序：

```
Load R1, MEM_LOC_X
Store MEM_LOC_X, 0
```

If we allow the store to go before the load, then the `R1` register may read the wrong value from the memory location `MEM_LOC_X`.
如果允许存储操作先于加载操作执行，那么 `R1` 寄存器可能会从内存地址 `MEM_LOC_X` 读取到错误的值。

Another interesting situation happens when a load consumes data from an earlier store:
另一种有趣的情况是，当加载操作消耗了先前存储操作中的数据时：

```
Store MEM_LOC, 0
Load R1, MEM_LOC
```

If a load consumes data from a store that hasn't yet finished, we should not allow the load to proceed. But what if we don't yet know the address of the store? In this case, the processor predicts whether there will be any potential data forwarding between the load and the store and if reordering is safe. This is known as _memory disambiguation_. When a load starts executing, it has to be checked against all older stores for potential store forwarding. There are four possible scenarios:

如果加载操作消耗了尚未完成的存储操作中的数据，我们不应该允许加载操作继续进行。但如果我们还不知道存储操作的地址呢？在这种情况下，处理器会预测加载操作和存储操作之间是否存在潜在的数据转发，以及重新排序是否安全。这被称为*内存消歧（_memory disabiguation）*。当加载操作开始执行时，必须将其与所有之前的存储操作进行比较，以确定是否存在潜在的存储转发。有四种可能的情况：

* Prediction: Not dependent; Outcome: Not dependent. This is a case of a successful memory disambiguation, which yields optimal performance.
* Prediction: Dependent; Outcome: Not dependent. In this case, the processor was overly conservative and did not let the load go ahead of the store. This is a missed opportunity for performance optimization.
* Prediction: Not dependent; Outcome: Dependent. This is a _memory order violation_. Similar to the case of a branch misprediction, the processor has to flush the pipeline, roll back the execution, and start over. It is very costly.
* Prediction: Dependent; Outcome: Dependent. There is a memory dependency between the load and the store, and the processor predicted it correctly. No missed opportunities.

* 预测：不依赖；结果：不依赖。这是内存消歧成功的情况，可以获得最佳性能。
* 预测：依赖；结果：不依赖。在这种情况下，处理器过于保守，没有让加载操作优先于存储操作执行。这错失了性能优化的机会。
* 预测：不依赖；结果：依赖。这是*内存序违例（_memory order violation）*。与分支预测错误的情况类似，处理器必须清空流水线，回滚执行，然后重新开始。这代价非常高昂。
* 预测：依赖；结果：依赖。加载和存储之间存在内存依赖关系，处理器正确预测了这一点。没有错过任何机会。

It's worth mentioning that forwarding from a store to a load occurs in real code quite often. In particular, any code that uses read-modify-write accesses to its data structures is likely to trigger these sorts of problems. Due to the large out-of-order window, the CPU can easily attempt to process multiple read-modify-write sequences at once, so the read of one sequence can occur before the write of the previous sequence is complete. One such example is presented in [@sec:UarchSpecificIssues].

值得一提的是，在实际代码中，从存储到加载的定向转发非常常见。特别是，任何使用读-修改-写访问其数据结构的代码都可能触发此类问题。由于乱序执行窗口较大，CPU很容易尝试同时处理多个读-修改-写序列，因此一个序列的读取操作可能在前一个序列的写入操作完成之前发生。[@sec:UarchSpecificIssues] 中给出了一个这样的例子。

### TLB Hierarchy TLB层次结构

Recall from [@sec:TLBs] that translations from virtual to physical addresses are cached in the TLB. Golden Cove's TLB hierarchy is presented in Figure @fig:GLC_TLB. Similar to a regular data cache, it has two levels, where level 1 has separate instances for instructions (ITLB) and data (DTLB). L1 ITLB has 256 entries for regular 4K pages and covers 1MB of memory, while L1 DTLB has 96 entries that cover 384 KB. 

回顾 [@sec:TLBs]，虚拟地址到物理地址的转换被缓存在TLB中。Golden Cove的TLB层级结构如图 @fig:GLC_TLB 所示。与常规数据缓存类似，它分为两级，其中一级TLB包含指令TLB(ITLB: Instruciton TLB)和数据TLB(DTLB: Data TLB)的独立实例。L1 ITLB包含256个条目，用于存储标准的4KB页，覆盖1MB存储空间；而L1 DTLB包含96个条目，覆盖384KB内存。

![TLB hierarchy of Intel's Golden Cove microarchitecture. Intel公司Golden Cove微体系结构的TLB层次结构。](../../img/uarch/GLC_TLB_hierarchy.png){#fig:GLC_TLB width=60%}

The second level of the hierarchy (STLB) caches translations for both instructions and data. It is a larger storage for serving requests that missed in the L1 TLBs. L2 STLB can accommodate 2048 recent data and instruction page address translations, which covers a total of 8MB of memory space. There are fewer entries available for 2MB huge pages: L1 ITLB has 32 entries, L1 DTLB has 32 entries, and L2 STLB can only use 1024 entries that are also shared with regular 4KB pages.

该层级结构的第二级（STLB）缓存指令和数据的地址转换。它是一个更大的存储空间，用于处理L1 TLB中遗漏的请求。L2 STLB可以容纳2048个最近的指令和数据页地址转换，总共覆盖8MB内存空间。2MB巨页可用的条目较少：L1 ITLB有32个条目，L1 DTLB有32个条目，而L2 STLB只能使用1024个条目，这些条目也与常规4KB页共享。

In case a translation was not found in the TLB hierarchy, it has to be retrieved from the DRAM by "walking" the kernel page tables. Recall that the page table is built as a radix tree of subtables, with each entry of the subtable holding a pointer to the next level of the tree. 

如果在TLB层次结构中找不到转换，则必须通过“遍历walking”内核页表从DRAM中检索。回想一下，页表构建为子表的基数树，子表中的每个条目都包含指向树下一级的指针。

The key element to speed up the page walk procedure is a set of Paging-Structure Caches[^3] that cache the hot entries in the page table structure. For the 4-level page table, we have the least significant twelve bits (11:0) for page offset (not translated), and bits 47:12 for the page number. While each entry in a TLB is an individual complete translation, Paging-Structure Caches cover only the upper 3 levels (bits 47:21). The idea is to reduce the number of loads required to execute in case of a TLB miss. For example, without such caches, we would have to execute 4 loads, which would add latency to the instruction completion. But with the help of the Paging-Structure Caches, if we find a translation for levels 1 and 2 of the address (bits 47:30), we only have to do the remaining 2 loads.

加速页表遍历过程的关键在于一组分页结构缓存[^3]，它们缓存页表结构中的热点条目。对于4级页表，最低12位（11:0）用于存储页偏移量（无需转换），第47位到第12位用于存储页号。TLB中的每个条目都是一个完整的地址转换，而分页结构缓存仅覆盖地址的前三级（第47:21位）。其目的是减少TLB未命中时需要执行的加载操作次数。例如，如果没有分页结构缓存，我们需要执行4次加载操作，这会增加指令完成的延迟。但借助分页结构缓存，如果我们找到了地址第1级和第2级（第47:30位）的地址转换，则只需执行剩余的2次加载操作。

The Golden Cove microarchitecture has four dedicated page walkers, which allows it to process 4 page walks simultaneously. In the event of a TLB miss, these hardware units will issue the required loads into the memory subsystem and populate the TLB hierarchy with new entries. The page-table loads generated by the page walkers can hit in L1, L2, or L3 caches (details are not disclosed). Finally, page walkers can anticipate a future TLB miss and speculatively do a page walk to update TLB entries before a miss actually happens.

Golden Cove微体系结构拥有四个专用的页面遍历器，使其能够同时处理4次页面遍历。当TLB未命中时，这些硬件单元会将所需的加载操作发送到内存子系统，并用新的条目填充TLB层次结构。页面遍历器生成的页表加载操作可以命中L1、L2或L3缓存（具体细节未公开）。最后，页面遍历器可以预测未来的TLB未命中，并在未命中实际发生之前进行推测性页面遍历以更新TLB条目。

The Golden Cove specification doesn't disclose how resources are shared between two SMT threads. But in general, caches, TLBs, and execution units are fully shared to improve the dynamic utilization of those resources. On the other hand, buffers for staging instructions between major pipe stages are either replicated or partitioned. These buffers include IDQ, ROB, RAT, RS, Load Buffer, and the Store Buffer. PRF is also replicated.

Golden Cove规范并未公开两个SMT线程之间如何共享资源。但一般来说，缓存、TLB和执行单元是完全共享的，以提高这些资源的动态利用率。另一方面，用于在主要流水线阶段之间暂存指令的缓冲区要么被复制，要么被分区。这些缓冲区包括IDQ、ROB、RAT、RS、加载Load缓冲区和存储Store缓冲区。物理寄存器文件PRF也被复制。

[^1]: There are around 300 physical general-purpose registers (GPRs) and a similar number of vector registers. The actual number of registers is not disclosed. 大约有300个物理通用寄存器(GPR: General Purpose Registers)和数量相近的向量寄存器。寄存器的实际数量未公开。
[^2]: Load Buffer and Store Buffer sizes are not disclosed, but people have measured 192 and 114 entries respectively. 加载缓冲区和存储缓冲区的大小未公开，但据测量，它们分别有192和114个条目。
[^3]: AMD's equivalent is called Page Walk Caches. AMD的等效技术称为页遍历缓存
[^4]: People have measured  ~200 entries in the RS, however the actual number of entries is not disclosed. 据测量，RS中有大约200个条目，但实际条目数未公开。
[^5]: Renaming must be done in program order. 重命名必须按程序顺序进行。
[^9]: Complex CISC instructions are translated into simple RISC microoperations, see [@sec:sec_UOP]. 复杂的CISC指令被转换为简单的RISC微操作，参见 [@sec:sec_UOP]。
