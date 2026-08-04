## Branch Recording Mechanisms  分支记录机制 {#sec:lbr}

Modern high-performance CPUs provide branch recording mechanisms that enable a processor to continuously log a set of previously executed branches. But before going into the details, you may ask: Why are we so interested in branches? Well, because this is how we can determine the control flow of a program. We largely ignore other instructions in a basic block (see [@sec:BasicBlock]) because a branch is always the last instruction in a basic block. Since all instructions in a basic block are guaranteed to be executed once, we can only focus on branches that will “represent” the entire basic block. Thus, it’s possible to reconstruct the entire line-by-line execution path of the program if we track the outcome of every branch. This is what the Intel Processor Traces (PT) feature is capable of doing, which is discussed in Appendix C. Branch recording mechanisms that we will discuss in this section are based on sampling, not tracing, and thus have different use cases and capabilities.

现代高性能CPU提供分支记录机制，使处理器能够持续记录一组先前执行过的分支。但在深入探讨细节之前，你可能会问：为什么我们如此关注分支？这是因为我们可以通过分支来确定程序的控制流。我们通常会忽略基本块中的其他指令（参见 [@sec:BasicBlock]），因为分支始终是基本块中的最后一条指令。由于基本块中的所有指令都保证执行一次，因此我们只需关注那些能够“代表”整个基本块的分支。因此，如果我们跟踪每个分支的结果，就可以重建程序的逐行执行路径。这正是Intel处理器跟踪(PT: Processor Traces)特性的能力所在，该特性将在附录C中讨论。本节将讨论的分支记录机制基于采样而非跟踪，因此具有不同的应用场景和功能。

Processors designed by Intel, AMD, and Arm all have announced their branch recording extensions. Exact implementations may vary but the idea is the same: hardware logs the “from” and “to” addresses of each branch along with some additional data in parallel with executing the program. If we collect a long enough history of source-destination pairs, we will be able to unwind the control flow of our program, just like a call stack, but with limited depth. Such extensions are designed to cause minimal slowdown to a running program (often within 1%).

Intel、AMD和Arm设计的处理器都已宣布了其分支记录扩展功能。具体的实现方式可能有所不同，但基本思路相同：硬件会在程序执行的同时记录每个分支的“源地址”和“目标地址”，以及一些其他数据。如果我们收集到足够长的源与目标地址配对历史记录，就能像调用栈一样回溯程序的控制流，但深度有限。此类扩展旨在将程序运行速度的影响降到最低（通常在1%以内）。

With a branch recording mechanism in place, we can sample on branches (or cycles, it doesn't matter), but during each sample, look at the previous N branches that were executed. This gives us reasonable coverage of the control flow in the hot code paths but does not overwhelm us with too much information, as only a small number of total branches are examined. It is important to keep in mind that this is still sampling, so not every executed branch can be examined. A CPU generally executes too fast for that to be feasible.

有了分支记录机制，我们可以对分支（或循环，两者区分并不重要）进行采样，但在每次采样时，只查看之前执行的N个分支。这使我们能够合理地覆盖热点代码路径中的控制流，同时又不会导致信息过载，因为只检查少量分支。需要注意的是，这仍然是采样，因此并非每个已执行的分支都能被检查。CPU的执行速度通常太快，无法做到这一点。

It is very important to keep in mind that only taken branches are being logged. [@lst:LogBranches] shows an example of how branch results are being tracked. This code represents a loop with three instructions that may change the execution path of the program, namely loop back edge `JNE` (1), conditional branch `JNS` (2), function `CALL` (3), and return address from this function (4).

务必注意，只有实际执行的分支才会被记录。[@lst:LogBranches] 展示了如何跟踪分支结果的示例。这段代码表示一个循环，其中包含三条可能改变程序执行路径的指令，分别是循环回跳边缘 `JNE` (1)、条件分支 `JNS` (2)、函数 `CALL` (3) 以及该函数的返回地址(4)。

Listing: Example of logging branches.
代码列表：对分支进行日志记录的样例

~~~~ {#lst:LogBranches .asm}
----> 4eda10:  mov   edi,DWORD PTR [rbx]
|     4eda12:  test  edi,edi
| --- 4eda14:  jns   4eda1e              <== (2)
| |   4eda16:  mov   eax,edi
| |   4eda18:  shl   eax,0x7
| |   4eda1b:  lea   edi,[rax+rdi*8]
| └─> 4eda1e:  call  4edb26              <== (3)
|     4eda23:  add   rbx,0x4             <== (4)
|     4eda27:  mov   DWORD PTR [rbx-0x4],eax
|     4eda2a:  cmp   rbx,rbp
----- 4eda2d:  jne   4eda10              <== (1)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

[@lst:BranchHistory] shows one possible branch history that can be logged with a branch recording mechanism. It shows the last 7 branch outcomes (many more not shown) at the moment we executed the `CALL` instruction. Because on the latest iteration of the loop, the `JNS` branch (`4eda14` &rarr; `4eda1e`) was not taken, it is not logged and thus does not appear in the history.

[@lst:BranchHistory] ​​显示了使用分支记录机制可以日志记录的一种可能的分支历史。它显示了执行 `CALL` 指令时最近的7个分支结果（还有更多未显示）。由于在循环的最近一次迭代中，`JNS` 分支（`4eda14` 和 `4eda1e`）没有被执行，因此它没有被记录，也因此不会出现在历史记录中。

Listing: Possible branch history.
代码列表：可能的分支历史

~~~~ {#lst:BranchHistory .asm}
    Source Address    Destination Address
    ...               ...
(1) 4eda2d            4eda10    <== next iteration              │
(2) 4eda14            4eda1e    <== jns taken                   │
(3) 4eda1e            4edb26    <== call a function             │ 
(4) 4b01cd            4eda23    <== return from a function      │ time
(1) 4eda2d            4eda10    <== next iteration              │ 
(3) 4eda1e            4edb26    <== latest branch               V 
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The fact that untaken branches are not logged might add a burden for analysis but usually, it doesn’t complicate it too much. We can still infer the complete execution path since we know that the control flow was sequential from the destination address in the entry `N-1` to the source address in the entry `N`.

未发生的分支未被日志记录这一事实可能会增加分析的负担，但通常不会造成太大的复杂化。我们仍然可以推断出完整的执行路径，因为我们知道控制流是从条目 `N-1` 中的目标地址到条目 `N` 中的源地址的顺序执行的。

Next, we will take a look at each vendor's branch recording mechanism and then explore how they can be used in performance analysis.

接下来，我们将了解各厂商的分支记录机制，并探讨它们在性能分析中的应用。

### LBR on Intel Platforms Intel平台上的LBR

Intel first implemented its Last Branch Record (LBR) facility in the NetBurst microarchitecture. Initially, it could record only the 4 most recent branch outcomes. It was later enhanced to 16 starting with Nehalem and to 32 starting from Skylake. Prior to the Golden Cove microarchitecture, LBR was implemented as a set of model-specific registers (MSRs), but now it works within architectural registers.[^12]

Intel最初在NetBurst微体系结构中实现了其最后分支记录(LBR: Last Branch Record)功能。最初，它只能记录最近的4个分支结果。后来，从Nehalem体系结构开始，记录数量增加到16个，从Skylake体系结构开始，增加到32个。在Golden Cove微体系结构之前，LBR是以一组特定于型号的寄存器(MSR: Model-Specific Register)的形式实现的，但现在它工作在体系结构寄存器中[^12]。

The LBR registers act like a ring buffer that is continuously overwritten and provides only 32 most recent branch outcomes. Each LBR entry is comprised of three 64-bit values:

LBR寄存器就像一个不断被覆盖的环形缓冲区，只提供最近的32个分支结果。每个LB 条目由三个64位值组成：

- The source address of the branch (`From IP`).
- The destination address of the branch (`To IP`).
- Metadata for the operation, including mispredict, and elapsed cycle time information.

- 分支的源地址（“From IP”）。
- 分支的目标地址（“To IP”）。
- 操作的元数据，包括：预测错误和经过的周期时间信息。

There are important applications to the additional information saved besides just source and destination addresses, which we will discuss later.

除了源地址和目标地址之外，保存的其他信息还有重要的应用，我们将在后面讨论。

When a sampling counter overflows and a Performance Monitoring Interrupt (PMI) is triggered, the LBR logging freezes until the software captures the LBR records and resumes collection.

当采样计数器溢出并触发性能监控中断(PMI: Performance Monitoring Interrupt)时，LBR日志记录会冻结，直到软件捕获LBR记录并恢复收集。

LBR collection can be limited to a set of specific branch types, for example, a user may choose to log only function calls and returns. Applying such a filter to the code in [@lst:LogBranches], we would only see branches (3) and (4) in the history. Users can also filter in/out conditional and unconditional jumps, indirect jumps and calls, system calls, interrupts, etc. In Linux perf, there is a `-j` option that enables/disables the recording of various branch types.

LBR收集可以限制为一组特定的分支类型，例如，用户可以选择仅记录函数调用和返回。将此类过滤器应用于 [@lst:LogBranches] 中的代码，我们将只在历史记录中看到分支(3)和(4)。用户还可以筛选条件跳转、无条件跳转、间接跳转和调用、系统调用、中断等。在Linux perf中，有一个 `-j` 选项可以启用/禁用各种分支类型的记录。

By default, the LBR array works as a ring buffer that captures control flow transitions. However, the depth of the LBR array is limited, which can be a limiting factor when profiling applications in which a transition of the execution flow is accompanied by a large number of leaf function calls. These calls to leaf functions, and their returns, are likely to displace the main execution context from the LBRs. Consider the example in [@lst:LogBranches] again. Say we want to unwind the call stack from the history in LBR, and so we configured LBR to capture only function calls and returns. If the loop runs thousands of iterations, then taking into account that the LBR array is only 32 entries deep, there is a very high chance we would only see 16 pairs of entries (3) and (4). In such a scenario, the LBR array is cluttered with leaf function calls which don't help us to unwind the current call stack.

默认情况下，LBR阵列用作环形缓冲区，用于捕获控制流转换。但是，LBR阵列的深度有限，这在分析执行流转换过程中伴随大量叶子函数调用的应用程序时可能成为一个限制因素。这些叶子函数调用及其返回值很可能会将主执行上下文从LBR中移除。再次考虑 [@lst:LogBranches] 中的示例。假设我们想要从LBR的历史记录中回溯调用栈，因此我们将LBR配置为仅捕获函数调用和返回值。如果循环运行数千次迭代，考虑到LBR阵列只有32个条目的深度，我们很可能只能看到16对条目(3)和(4)。在这种情况下，LBR阵列中充斥着大量的叶子函数调用，这不利于我们展开当前的调用栈。

This is why LBR supports call-stack mode. With this mode enabled, the LBR array captures function calls as before, but as return instructions are executed the last captured branch (`call`) record is flushed from the array in a last-in-first-out (LIFO) manner. Thus, branch records with completed leaf functions will not be retained, while preserving the call stack information of the main line execution path. When configured in this manner, the LBR array emulates a call stack, where a `CALL` instruction pushes and a `RET` instruction pops entry from the stack. If the depth of the call stack in your application never goes beyond 32 nested frames, LBRs will give you very accurate information. [@IntelOptimizationManual, Volume 3B, Chapter 19 Last Branch Records]

这就是LBR支持调用栈模式的原因。启用此模式后，LBR数组仍然会像以前一样捕获函数调用，但当执行返回指令时，最后捕获的分支（`调用call`）记录会以后进先出(LIFO: Last-In-First-Out)的方式从阵列中刷新。因此，包含已完成叶子函数的分支记录不会被保留，同时保留了主执行路径的调用栈信息。以这种方式配置时，LBR阵列模拟了一个调用栈，其中 `调用CALL` 指令会将条目压入栈中，而 `返回RET` 指令会从栈中弹出条目。如果应用程序中的调用栈深度不超过32个嵌套帧，LBR将提供非常精确的信息。[@IntelOptimizationManual，3B卷，第19章--最后分支记录]

You can make sure LBRs are enabled on your system with the following command:

你可以使用以下命令确保系统已启用LBR：

```bash
$ dmesg | grep -i lbr
[    0.228149] Performance Events: PEBS fmt3+, 32-deep LBR, Skylake events, full-width counters, Intel PMU driver.
```

With Linux `perf`, you can collect LBR stacks using the following command:

借助Linux `perf`，你可以通过使用下面的命令来收集LBR栈：

```bash
$ perf record -b -e cycles -- ./benchmark.exe
[ perf record: Woken up 68 times to write data ]
[ perf record: Captured and wrote 17.205 MB perf.data (22089 samples) ]
```

LBR stacks can also be collected using the `perf record --call-graph lbr` command, but the amount of information collected is less than using `perf record -b`. For example, branch misprediction and cycles data are not collected when running `perf record --call-graph lbr`.

也可以使用 `perf record --call-graph lbr` 命令收集LBR堆栈，但收集的信息量比使用 `perf record -b` 命令要少。例如，运行 `perf record --call-graph lbr` 时不会收集分支预测错误和循环数据。

Because each collected sample captures the entire LBR stack (32 last branch records), the size of collected data (`perf.data`) is significantly bigger than sampling without LBRs. Still, the runtime overhead for the majority of LBR use cases is below 1%. [@Nowak2014TheOO]

由于每个收集的样本都会捕获完整的LBR堆栈（32条最后分支记录），因此收集的数据（`perf.data`）的大小明显大于不使用LBR进行采样的情况。尽管如此，对于大多数LBR用例而言，运行时开销仍然低于1%。[@Nowak2014TheOO]

Users can export raw LBR stacks for custom analysis. Below is the Linux perf command you can use to dump the contents of collected branch stacks:

用户可以导出原始LBR堆栈以进行自定义分析。以下是可用于转储已收集分支堆栈内容的Linux perf命令：

```bash
$ perf record -b -e cycles -- ./benchmark.exe
$ perf script -F brstack &> dump.txt
```

The `dump.txt` file, which can be quite large, contains lines like those shown below:

`dump.txt` 文件可能非常大，其中包含类似如下所示的行：

```
...
0x4edaf9/0x4edab0/P/-/-/29
0x4edabd/0x4edad0/P/-/-/2
0x4edadd/0x4edb00/M/-/-/4
0x4edb24/0x4edab0/P/-/-/24
0x4edabd/0x4edad0/P/-/-/2
0x4edadd/0x4edb00/M/-/-/1
0x4edb24/0x4edab0/P/-/-/3
0x4edabd/0x4edad0/P/-/-/1
...
```

In the output above, we present eight entries from the LBR stack, which typically consists of 32 LBR entries. Each entry has `FROM` and `TO` addresses (hexadecimal values), a predicted flag (this one single branch outcome was `M` - Mispredicted, `P` - Predicted), and the number of cycles since the previous record (number in the last position of each entry). Components marked with "`-`" are related to transactional memory extension (TSX), which we won't discuss here. Curious readers can look up the format of a decoded LBR entry in the `perf script` [specification](http://man7.org/linux/man-pages/man1/perf-script.1.html)[^2].

上面的输出展示了LBR堆栈中的8个条目，通常情况下，LBR堆栈包含32个条目。每个条目都包含 `FROM` 和 `TO` 地址（十六进制值）、一个预测标志（此分支结果为 `M` - 预测错误，`P` - 预测正确）以及自上一条记录以来的周期数（每个条目末尾的数字）。标有“-”的组件与事务内存扩展(TSX: TranSactional memory eXtension)相关，此处不做讨论。感兴趣的读者可以查阅 `perf script` [规范](http://man7.org/linux/man-pages/man1/perf-script.1.html)[^2] 了解解码后的LBR条目的格式。

### LBR on AMD Platforms AMD平台上的LBR

AMD processors also support Last Branch Record (LBR) on AMD Zen4 processors. Zen4 can log 16 pairs of "from" and "to" addresses along with some additional metadata. Similar to Intel LBR, AMD processors can record various types of branches. The main difference from Intel LBR is that AMD processors don't support call stack mode yet, hence the LBR feature can't be used for call stack collection. Another noticeable difference is that there is no cycle count field in the AMD LBR record. For more details see [@AMDProgrammingManual, 13.1.1.9 Last Branch Stack Registers].

AMD也在Zen4处理器上支持最后分支记录(LBR: Last Branch Record)。Zen4可以记录16对“源from地址”和“目标to地址”以及一些额外的元数据。与Intel LBR类似，AMD处理器可以记录各种类型的分支。与Intel LBR的主要区别在于AMD处理器目前尚不支持调用栈模式，因此LBR 功能无法用于调用栈收集。另一个显著区别是 AMD LBR 记录中没有循环计数字段。更多详情请参阅 [@AMDProgrammingManual, 13.1.1.9 最后分支栈寄存器]。

Since Linux kernel 6.1 onwards, Linux `perf` on AMD Zen4 processors supports the branch analysis use cases we discuss below unless explicitly mentioned otherwise. The Linux `perf` commands use the same `-b` and `-j` options as on Intel processors.

从Linux内核6.1开始，除非另有明确说明，否则AMD Zen4处理器上的Linux `perf` 命令支持我们将在下文讨论的分支分析用例。Linux `perf` 命令使用的 `-b` 和 `-j` 选项与Intel处理器相同。

Branch analysis is also possible with the AMD uProf CLI tool. The example command below dumps collected raw LBR records and generates a CSV report:

此外，AMD uProf命令行工具也支持分支分析。以下示例命令会导出收集到的原始LBR记录并生成CSV报告：

```bash
$ AMDuProfCLI collect --branch-filter -o /tmp/ ./AMDTClassicMatMul-bin
```

### BRBE on Arm Platforms Arm平台上的BRBE

Arm introduced its branch recording extension called BRBE in 2020 as a part of the ARMv9.2-A ISA. Arm BRBE is very similar to Intel's LBR and provides many similar features. Just like Intel's LBR, BRBE records contain source and destination addresses, a misprediction bit, and a cycle count value. According to the latest available BRBE specification, the call stack mode is not supported. The Branch records only contain information for a branch that is architecturally executed, i.e., not on a mispredicted path. Users can also filter records based on specific branch types. One notable difference is that BRBE supports configurable depth of the BRBE buffer: processors can choose the capacity of the BRBE buffer to be 8, 16, 32, or 64 records. More details are available in [@Armv9ManualSupplement, Chapter F1 "Branch Record Buffer Extension"].

Arm于2020年推出了名为BRBE的分支记录扩展，作为ARMv9.2-A指令集体系结构(ISA)的一部分。Arm BRBE与Intel的LBR非常相似，并提供了许多类似的功能特性。与Intel的LBR一样，BRBE记录包含源地址和目标地址、预测错误位以及循环计数。根据最新的BRBE规范，调用栈模式不受支持。分支记录仅包含体系结构执行的分支信息，即不在预测错误的路径上的分支信息。用户还可以根据特定的分支类型筛选记录。一个显著的区别是，BRBE支持可配置的BRBE缓冲区深度：处理器可以选择BRBE缓冲区的容量为8、16、32或64条记录。更多详情请参阅 [@Armv9ManualSupplement，F1章“分支记录缓冲区扩展”]。

At the time of writing, there were no commercially available machines that implement ARMv9.2-A, so it is not possible to test the BRBE extension in action.

撰写本文时，市面上还没有实现ARMv9.2-A的商用机器，因此无法测试BRBE扩展的实际效果。

Several use cases become possible thanks to branch recording. Next, we will cover the most important ones.

分支记录使得多种应用场景成为可能。接下来，我们将介绍其中最重要的几个。

### Capture Call Stacks 捕获调用堆栈

One of the most popular use cases for branch recording is capturing call stacks. I already covered why we need to collect them in [@sec:secCollectCallStacks]. Branch recording can be used as a lightweight substitution for collecting call graph information even if you compiled a program without frame pointers or debug information.

分支记录最常见的应用场景之一是捕获调用堆栈。我在 [@sec:secCollectCallStacks] 中已经解释过为什么需要收集调用堆栈。即使编译程序时没有包含帧指针或调试信息，分支记录也可以作为一种轻量级的替代方案来收集调用图信息。

At the time of writing (late 2023), AMD's LBR and Arm's BRBE don't support call stack collection, but Intel's LBR does. Here is how you can do it with Intel LBR:

截至撰写本文时（2023年末），AMD的LBR和Arm的BRBE均不支持调用栈收集，但Intel的LBR支持。以下是如何使用Intel LBR实现调用栈收集的方法：

```bash
$ perf record --call-graph lbr -- ./a.exe
$ perf report -n --stdio
# Children   Self    Samples  Command  Object  Symbol
# ........  .......  .......  .......  ......  .......
	99.96%  99.94%    65447    a.exe    a.exe  [.] bar
            |
             --99.94%--main
                       |
                       |--90.86%--foo
                       |          |
                       |           --90.86%--bar
                       |
                        --9.08%--zoo
                                  bar
```

As you can see, we identified the hottest function in the program (which is `bar`). Also, we identified callers that contribute to the most time spent in function `bar`: 91% of the time the tool captured the `main` &rarr; `foo` &rarr; `bar` call stack, and 9% of the time it captured `main` &rarr; `zoo` &rarr; `bar`. In other words, 91% of samples in `bar` have `foo` as its caller function.

如您所见，我们已识别出程序中最热门的函数（即 `bar`）。此外，我们还识别出了对 `bar` 函数耗时贡献最大的调用者：91%的情况下，工具捕获到的调用栈为 `main` -> `foo` -> `bar`；9%的情况下，捕获到的调用栈为 `main` -> `zoo` -> `bar`。换句话说，`bar` 中91%的样本都以 `foo` 为调用者。

It's important to mention that we cannot necessarily drive conclusions about function call counts in this case. For example, we cannot say that `foo` calls `bar` 10 times more frequently than `zoo`. It could be the case that `foo` calls `bar` once, but it executes an expensive path inside `bar` while `zoo` calls `bar` many times but returns quickly from it.

需要指出的是，在这种情况下，我们无法就函数调用次数得出确切结论。例如，我们不能断言 `foo` 调用 `bar` 的频率是 `zoo` 的10倍。`foo` 可能只调用了 `bar` 一次，但它在 `bar` 内部执行了一条耗时较长的路径；而 `zoo` 可能多次调用 `bar`，但每次都很快返回。

### Identify Hot Branches 识别热点分支 {#sec:lbr_hot_branch}

Branch recording also enables us to identify the most frequently taken branches. It is supported on Intel and AMD. According to Arm's BRBE specification, it can be supported, but due to the unavailability of processors that implement this extension, it is not possible to verify. Here is an example:

分支记录功能还能帮助我们识别最常执行的分支。Intel和AMD处理器均支持此功能。根据Arm的BRBE规范，Arm也应该支持此功能，但由于目前尚无处理器支持该扩展，因此无法验证。以下是一个示例：

```bash
$ perf record -e cycles -b -- ./a.exe
[ perf record: Woken up 3 times to write data ]
[ perf record: Captured and wrote 0.535 MB perf.data (670 samples) ]
$ perf report -n --sort overhead,srcline_from,srcline_to -F +dso,symbol_from,symbol_to --stdio
# Samples: 21K of event 'cycles'
# Event count (approx.): 21440
# Overhead  Samples  Object  Source Sym  Target Sym  From Line  To Line
# ........  .......  ......  ..........  ..........  .........  .......
  51.65%      11074   a.exe   [.] bar    [.] bar      a.c:4      a.c:5
  22.30%       4782   a.exe   [.] foo    [.] bar      a.c:10     (null)
  21.89%       4693   a.exe   [.] foo    [.] zoo      a.c:11     (null)
   4.03%        863   a.exe   [.] main   [.] foo      a.c:21     (null)
```

From this example, we can see that more than 50% of taken branches are within the `bar` function, 22% of branches are function calls from `foo` to `bar`, and so on. Notice how `perf` switched from `cycles` event to analyzing LBR stacks: only 670 samples were collected, yet we have an entire 32-entry LBR stack captured with every sample. This gives us `670 * 32 = 21440` LBR entries (branch outcomes) for analysis.[^5]

从这个例子中我们可以看出，超过50%的分支都位于 `bar` 函数内部，22%的分支是从 `foo` 到 `bar` 的函数调用，以此类推。注意 `perf` 是如何从分析 `cycles` 事件切换到分析LBR堆栈的：虽然只收集了670个样本，但每个样本都捕获了一个完整的32条目的LBR堆栈。这为我们提供了 `670 * 32 = 21440` 个LBR条目（分支结果）用于分析[^5]。

Most of the time, it’s possible to determine the location of the branch just from the line of code and target symbol. However, theoretically, you could write code with two `if` statements written on a single line. Also, when expanding the macro definition, all the expanded code is attributed to the same source line, which is another situation when this might happen. This issue does not completely block the analysis but only makes it a little more difficult. To disambiguate two branches, you likely need to analyze raw LBR stacks yourself (see example on [easyperf](https://easyperf.net/blog/2019/05/06/Estimating-branch-probability)[^6] blog).

大多数情况下，只需代码行和目标符号即可确定分支的位置。然而，理论上，你可以编写包含两个 `if` 语句的代码，它们写在同一行上。此外，展开宏定义时，所有展开的代码都会被归入同一源代码行，这也是可能出现这种情况的另一种原因。这个问题不会完全阻止分析，只是让分析稍微复杂一些。要区分两个分支，您可能需要自行分析原始LBR堆栈（参见 [easyperf](https://easyperf.net/blog/2019/05/06/Estimating-branch-probability)[^6] 博客上的示例）。

Using branch recording, we can also find a *hyperblock* (sometimes called *superblock*), which is a chain of hot basic blocks in a function that are not necessarily laid out in the sequential physical order but are executed sequentially. Thus, a hyperblock represents a typical hot path inside a function.

使用分支记录，我们还可以找到*超块hyperblock*（有时也称为*超级块superblock*），它是函数中一系列热门基本块的链，这些基本块不一定按物理顺序排列，但会按顺序执行。因此，超块代表函数内部的典型热门路径。

### Analyze Branch Misprediction Rate 分析分支预测错误率 {#sec:secLBR_misp_rate}

Thanks to the mispredict bit in the additional information saved inside each record, it is also possible to know the misprediction rate for hot branches. In this example, we take a C-code-only version of the 7-zip benchmark from the LLVM test suite.[^7] The output of the `perf report` command is slightly trimmed to fit nicely on a page. The following use case is supported on Intel and AMD. According to Arm's BRBE specification, it can be supported, but due to the unavailability of processors that implement this extension, it is not possible to verify.

由于每个记录中保存的附加信息包含预测错误位，因此还可以了解热门分支的预测错误率。在本例中，我们采用LLVM测试套件中7-zip基准测试的纯C代码版本[^7]。`perf report`命令的输出结果经过轻微截断，以便更好地显示在页面上。以下用例在Intel和AMD平台上均受支持。根据Arm的BRBE规范，该用例也应该受支持，但由于目前尚无实现此扩展的处理器，因此无法进行验证。

```bash
$ perf record -e cycles -b -- ./7zip.exe b
$ perf report -n --sort symbol_from,symbol_to -F +mispredict,srcline_from,srcline_to --stdio
# Samples: 657K of event 'cycles'
# Event count (approx.): 657888

# Overhead  Samples  Mis  From Line  To Line  Source Sym  Target Sym
# ........  .......  ...  .........  .......  ..........  ..........
    46.12%   303391   N   dec.c:36   dec.c:40  LzmaDec     LzmaDec
    22.33%   146900   N   enc.c:25   enc.c:26  LzmaFind    LzmaFind
     6.70%    44074   N   lz.c:13    lz.c:27   LzmaEnc     LzmaEnc
     6.33%    41665   Y   dec.c:36   dec.c:40  LzmaDec     LzmaDec
```

In this example, the lines that correspond to function `LzmaDec` are of particular interest to us. In the output that Linux `perf` provides, we can spot two entries that correspond to the `LzmaDec` function: one with `Y` and one with `N` letters. We can conclude that the branch on source line `dec.c:36` is the most executed in the benchmark since more than 50% of samples are attributed to it. Analyzing those two entries together gives us a misprediction rate for the branch. We know that the branch on line `dec.c:36` was predicted `303391` times (corresponds to `N`) and was mispredicted `41665` times (corresponds to `Y`), which gives us an `88%` prediction rate.

在这个例子中，我们特别关注与函数 `LzmaDec` 对应的代码行。在Linux `perf` 的输出中，我们可以找到两个与 `LzmaDec` 函数对应的条目：一个包含字母 `Y`，另一个包含字母 `N`。我们可以得出结论，源文件 `dec.c:36` 上的分支在基准测试中执行次数最多，因为超过50%的样本都归因于此。将这两个条目结合起来分析，我们可以得到该分支的误预测率。我们知道，`dec.c:36` 上的分支被预测了303391次（对应于`N`），被错误预测了 41665 次（对应于`Y`），因此预测准确率为88%。

Linux `perf` calculates the misprediction rate by analyzing each LBR entry and extracting a misprediction bit from it. So for every branch, we have a number of times it was predicted correctly and a number of mispredictions. Again, due to the nature of sampling, some branches might have an `N` entry but no corresponding `Y` entry. It means there are no LBR entries for the branch being mispredicted, but that doesn’t necessarily mean the prediction rate is `100%`.

Linux `perf`通过分析每个LBR条目并从中提取一个误预测位来计算误预测率。因此，对于每个分支，我们都有预测正确的次数和预测错误的次数。同样，由于采样特性，某些分支可能存在 `N` 条目，但没有对应的 `Y` 条目。这意味着对于预测错误的分支，没有对应的LBR条目，但这并不一定意味着预测率为 `100%`。

### Precise Timing of Machine Code 机器代码的精确时序 {#sec:timed_lbr}

As we showed in Intel's LBR section, starting from Skylake microarchitecture, there is a special `Cycle Count` field in the LBR entry. This additional field specifies the number of elapsed cycles between two taken branches. Since the target address in the previous (N-1) LBR entry is the beginning of a basic block (BB) and the source address of the current (N) LBR entry is the last instruction of the same basic block, then the cycle count is the latency of this basic block.

正如我们在Intel的LBR部分中所述，从Skylake微体系结构开始，LBR条目中有一个特殊的 `Cycle Count` 字段。该附加字段指定了两个分支之间经过的周期数。由于前一个(N-1)个LBR条目的目标地址是基本块(BB)的起始地址，而当前(N)个LB 条目的源地址是同一基本块的最后一条指令，因此周期数就是该基本块的延迟。

This type of analysis is not supported on AMD platforms since they don't record a cycle count in the LBR record. According to Arm's BRBE specification, it can be supported, but due to the unavailability of processors that implement this extension, it is not possible to verify. However, Intel supports it. Here is an example:

由于AMD平台的LBR记录中不包含周期计数，因此不支持此类分析。根据Arm的BRBE规范，理论上可以支持，但由于目前尚无处理器支持此扩展，因此无法验证。不过，Intel支持此功能。以下是一个示例：

```
400618:   movb  $0x0, (%rbp,%rdx,1)    <= start of the BB
40061d:   add $0x1, %rdx
400621:   cmp $0xc800000, %rdx
400628:   jnz 0x400644                 <= end of the BB
```

Suppose we have the following entries in the LBR stack:
假设我们在LBR堆栈里面有下面的条目记录：

```
  FROM_IP   TO_IP    Cycle Count
  ...       ...      ...        <== 26 entries
  40060a    400618    10
  400628    400644    80        <== occurrence 1 
  40064e    400600     3
  40060a    400618     9
  400628    400644   300        <== occurrence 2
  40064e    400600     3        <== LBR TOS
```

Given that information, we have two occurrences of the basic block that starts at offset `400618`. The first one was completed in 80 cycles, while the second one took 300 cycles. If we collect enough samples like that, we could plot an occurrence rate chart of latency for that basic block.

根据这些信息，我们找到了两个从偏移量 `400618` 开始的基本块。第一个基本块耗时80个时钟周期完成，第二个基本块耗时300个时钟周期。如果我们收集足够多的此类样本，就可以绘制该基本块的延迟出现率图表。

An example of such a chart is shown in Figure @fig:LBR_timing_BB. It was compiled by analyzing relevant LBR entries. The way to read this chart is as follows: it tells what was the rate of occurrence of a given latency value. For example, the basic block latency was measured as 100 cycles roughly 2% of the time, 14% of the time we measured 280 cycles, and never saw anything between 150 and 200 cycles. Another way to read is: based on the collected data, what is the probability of seeing a certain basic block latency if you were to measure it?

图 @fig:LBR_timing_BB 展示了这样一个图表的示例。该图表是通过分析相关的LBR条目编制的。解读此图表的方法如下：它显示了给定延迟值的出现频率。例如，基本块延迟约为100个时钟周期时，约占2%；约为280个时钟周期时，约占14%；而150到200个时钟周期之间的延迟值从未出现过。另一种解读方式是：基于收集到的数据，如果您测量某个基本块延迟，那么出现该延迟的概率是多少？

![Occurrence rate chart for latency of the basic block that starts at address `0x400618`. 起始地址为`0x400618`的基本块延迟发生率图。](../../img/pmu-features/LBR_timing_BB.png){#fig:LBR_timing_BB width=90%}

We can see two humps: a small one around 80 cycles \circled{1} and two bigger ones at 280 and 305 cycles \circled{2}. The block has a random load from a large array that doesn’t fit in the CPU L3 cache, so the latency of the basic block largely depends on this load. Based on the chart we can conclude that the first spike \circled{1} corresponds to the L3 cache hit and the second spike \circled{2} corresponds to the L3 cache miss where the load request goes all the way down to the main memory.

我们可以看到两个峰值：一个较小的峰值在80个周期左右（图中①），两个较大的峰值分别在280和305个周期（图中②）。该块会从一个无法放入CPU L3缓存的大型数组中随机加载数据，因此基本块的延迟很大程度上取决于此加载。根据图表，我们可以得出结论：第一个峰值（图中①）对应于L3缓存命中，第二个峰值（图中②）对应于L3缓存未命中，此时加载请求会一直向下跳转到主内存。

This information can be used for a fine-grained tuning of this basic block. This example might benefit from memory prefetching, which we will discuss in [@sec:memPrefetch]. Also, cycle count information can be used for timing loop iterations, where every loop iteration ends with a taken branch (back edge).

这些信息可用于对该基本块进行精细调优。这个例子或许可以从内存预取中受益，我们将在 [@sec:memPrefetch] 中讨论。此外，循环计数信息可用于循环迭代计时，其中每次循环迭代都以一个已执行的分支（回溯边）结束。

Before the proper support from profiling tools was in place, building graphs similar to Figure @fig:LBR_timing_BB required manual parsing of raw LBR dumps. An example of how to do this can be found on the [easyperf blog](https://easyperf.net/blog/2019/04/03/Precise-timing-of-machine-code-with-Linux-perf)[^9]. Luckily, in newer versions of Linux perf, obtaining this information is much easier. The example below demonstrates this method directly using Linux perf on the same 7-zip benchmark from the LLVM test suite we introduced earlier:

在性能分析工具提供完善的支持之前，构建类似于图 @fig:LBR_timing_BB 的图表需要手动解析原始LBR转储文件。[easyperf博客](https://easyperf.net/blog/2019/04/03/Precise-timing-of-machine-code-with-Linux-perf)[^9] 上提供了一个示例。幸运的是，在较新版本的Linux perf中，获取此信息要容易得多。以下示例直接使用Linux perf对我们之前介绍的LLVM测试套件中的同一个7-zip基准测试程序进行了演示：

```bash
$ perf record -e cycles -b -- ./7zip.exe b
$ perf report -n --sort symbol_from,symbol_to -F +cycles,srcline_from,srcline_to --stdio
# Samples: 658K of event 'cycles'
# Event count (approx.): 658240
 
 
# Overhead  Samples  BBCycles  FromSrcLine  ToSrcLine
# ........  .......  ........  ...........  ..........
     2.82%   18581      1      dec.c:325    dec.c:326
     2.54%   16728      2      dec.c:174    dec.c:174
     2.40%   15815      4      dec.c:174    dec.c:174
     2.28%   15032      2      find.c:375   find.c:376
     1.59%   10484      1      dec.c:174    dec.c:174
     1.44%   9474       1      enc.c:1310   enc.c:1315
     1.43%   9392      10      7zCrc.c:15   7zCrc.c:17
     0.85%   5567      32      dec.c:174    dec.c:174
     0.78%   5126       1      enc.c:820    find.c:540
     0.77%   5066       1      enc.c:1335   enc.c:1325
     0.76%   5014       6      dec.c:299    dec.c:299
     0.72%   4770       6      dec.c:174    dec.c:174
     0.71%   4681       2      dec.c:396    dec.c:395
     0.69%   4563       3      dec.c:174    dec.c:174
     0.58%   3804      24      dec.c:174    dec.c:174
```

Notice we've added the `-F +cycles` option to show cycle counts in the output (`BBCycles` column). Several insignificant lines were removed from the output of the `perf report` to make it fit on the page. Let's focus on lines in which the source and destination are `dec.c:174`, there are seven such lines in the output. In the source code, the line `dec.c:174` expands a macro that has a self-contained branch. That’s why the source and destination point to the same line.

请注意，我们添加了 `-F +cycles` 选项，以便在输出中显示周期数量（`BBCycles` 列）。为了使 `perf report` 的输出内容适合页面显示，我们删除了其中几行无关紧要的代码。让我们重点关注源和目标均为 `dec.c:174` 的代码行，输出中共有7行这样的代码。在源代码中，`dec.c:174` 展开了一个包含独立分支的宏。这就是为什么源和目标指向同一行的原因。

Linux `perf` sorts entries by overhead first, so we need to manually filter entries for the branch in which we are interested. Luckily, they can be grepped very easily. In fact, if we filter them, we will get the latency distribution for the basic block that ends with this branch, as shown in Table {@tbl:bb_latency}. This data can be plotted to obtain a chart similar to the one shown in Figure @fig:LBR_timing_BB.

Linux `perf` 首先按开销对条目进行排序，因此我们需要手动筛选出我们感兴趣的分支的条目。幸运的是，我们可以非常轻松地使用grep命令找到它们。实际上，如果我们进行筛选，我们将得到以该分支结尾的基本块的延迟分布，如表 {@tbl:bb_latency} 所示。我们可以将这些数据绘制成图表，得到类似于图 @fig:LBR_timing_BB 所示的图表。

----------------------------------------------
Cycles  Number of samples  Occurrence rate
------  -----------------  -------------------
1       10484              17.0%

2       16728              27.1%

3       4563                7.4%

4       15815              25.6%

6       4770                7.7%

24      3804                6.2%

32      5567                9.0%
----------------------------------------------

Table: Occurrence rate for basic block latency. 基本块延迟的出现率。{#tbl:bb_latency}

Here is how we can interpret the data: from all the collected samples, 17% of the time the latency of the basic block was one cycle, 27% of the time it was 2 cycles, and so on. Notice a distribution mostly concentrates from 1 to 6 cycles, but also there is a second mode of much higher latency of 24 and 32 cycles, which likely corresponds to branch misprediction penalty. The second mode in the distribution accounts for 15% of all samples.

数据解读如下：在所有收集的样本中，17%的情况下基本块的延迟为1个周期，27%的情况下为2个周期，以此类推。值得注意的是，延迟分布主要集中在1到6个周期之间，但也存在第二个峰值，延迟高达24和32个周期，这可能对应于分支预测错误惩罚。分布中的第二个峰值占所有样本的15%。

This example shows that it is feasible to plot basic block latencies not only for tiny microbenchmarks but for real-world applications as well. Currently, LBR is the most precise cycle-accurate source of timing information on Intel systems.

此示例表明，绘制基本块延迟图不仅适用于小型微基准测试，也适用于实际应用。目前，LBR是Intel系统上最精确的周期级时序信息来源。

### Estimating Branch Outcome Probability 分支结果概率估计

Later in [@sec:secFEOpt], we will discuss the importance of code layout for performance. Going forward a little bit, having a hot path fall through[^11] generally improves the performance of a program. Knowing the most frequent outcome of a certain branch enables developers and compilers to make better optimization decisions. For example, given that a branch is taken 99% of the time, we can try to invert the condition and convert it to a non-taken branch.

稍后在 [@sec:secFEOpt] 中，我们将讨论代码布局对性能的重要性。进一步来说，让热门分支执行顺序垂落[^11]通常可以提升程序的性能。了解特定分支最常见的执行结果，有助于开发者和编译器做出更优的优化决策。例如，假设某个分支的执行概率为99%，我们可以尝试反转该条件，将其转换为不发生挑战的分支。

LBR enables us to collect this data without instrumenting the code. As the outcome from the analysis, a user will get a ratio between true and false outcomes of the condition, i.e., how many times the branch was taken and how much was not taken. This feature especially shines when analyzing indirect jumps (switch statements) and indirect calls (virtual calls). You can find examples of using it on a real-world application on the [easyperf blog](https://easyperf.net/blog/2019/05/06/Estimating-branch-probability)[^6].

LBR允许我们在不修改代码的情况下收集这些数据。分析结果会给出该条件真假结果的比率，即分支执行的次数与未执行的次数之比。此功能在分析间接跳转（switch语句）和间接调用（虚调用）时尤为有效。您可以在 [easyperf博客](https://easyperf.net/blog/2019/05/06/Estimating-branch-probability)[^6]上找到实际应用案例。

### Providing Compiler Feedback Data 提供编译器反馈数据

We will discuss Profile Guided Optimizations (PGO) later in [@sec:secPGO], so just a quick mention here. Branch recording mechanisms can provide profiling feedback data for optimizing compilers. Imagine that we can feed all the data we discovered in the previous sections back to the compiler. In some cases, this data cannot be obtained using traditional static code instrumentation, so branch recording mechanisms are not only a better choice because of the lower overhead, but also because of richer profiling data. PGO workflows that rely on data collected from the hardware PMU are becoming more popular and likely will take off sharply once the support in AMD and Arm matures.

我们将在 [@sec:secPGO] 中讨论性能轮廓分析引导优化(PGO: Profile Guilded Optimization)，这里仅作简要介绍。分支记录机制可以为优化编译器提供性能分析反馈数据。想象一下，我们可以将前面章节中发现的所有数据反馈给编译器。在某些情况下，使用传统的静态代码插桩无法获得这些数据，因此分支记录机制不仅开销更低，而且性能分析数据也更丰富，因此是更好的选择。依赖于从硬件性能监测单元(PMU)收集的数据的PGO工作流程正变得越来越流行，并且一旦AMD和Arm的支持成熟，很可能会迅速发展。

[^2]: Linux `perf script` manual page Linux的`perf脚本`手册页- [http://man7.org/linux/man-pages/man1/perf-script.1.html](http://man7.org/linux/man-pages/man1/perf-script.1.html).
[^5]: The report header generated by perf might still be confusing because it says `21K of event cycles`. But there are `21K` LBR entries, not `cycles`. perf 生成的报告标题可能仍然会让人困惑，因为它显示 `21K个事件周期` 。但实际上有`21K`个 LBR 条目，而不是`周期`。
[^6]: Easyperf: Estimating Branch Probability - [https://easyperf.net/blog/2019/05/06/Estimating-branch-probability](https://easyperf.net/blog/2019/05/06/Estimating-branch-probability) Easyperf：估算分支概率 - [https://easyperf.net/blog/2019/05/06/Estimating-branch-probability](https://easyperf.net/blog/2019/05/06/Estimating-branch-probability)
[^7]: LLVM test-suite 7zip benchmark - [https://github.com/llvm-mirror/test-suite/tree/master/MultiSource/Benchmarks/7zip](https://github.com/llvm-mirror/test-suite/tree/master/MultiSource/Benchmarks/7zip) LLVM 测试套件 7zip 基准测试 - [https://github.com/llvm-mirror/test-suite/tree/master/MultiSource/Benchmarks/7zip](https://github.com/llvm-mirror/test-suite/tree/master/MultiSource/Benchmarks/7zip)
[^9]: Easyperf: Building a probability density chart for the latency of an arbitrary basic block - [https://easyperf.net/blog/2019/04/03/Precise-timing-of-machine-code-with-Linux-perf](https://easyperf.net/blog/2019/04/03/Precise-timing-of-machine-code-with-Linux-perf). Easyperf：构建任意基本块延迟的概率密度图 - [https://easyperf.net/blog/2019/04/03/Precise-timing-of-machine-code-with-Linux-perf](https://easyperf.net/blog/2019/04/03/Precise-timing-of-machine-code-with-Linux-perf).
[^11]: I.e., when the hot branches are not taken. 即：当热分支没有发生跳转时。
[^12]: Its primary advantage is that LBR features are clearly defined and there is no need to check the exact model number of the current CPU. It makes support in the OS and profiling tools much easier. Also, LBR entries can be configured to be included in the PEBS records (see [@sec:secPEBS]). 其主要优势在于LBR特性定义清晰，无需检查当前CPU的确切型号。这使得操作系统和性能分析工具的支持更加便捷。此外，LBR条目可以配置为包含在PEBS记录中（参见 [@sec:secPEBS]）。
