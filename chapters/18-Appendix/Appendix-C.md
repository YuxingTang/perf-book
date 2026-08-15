\phantomsection
# Appendix C. Intel Processor Traces 附录C. Intel处理器跟踪 {.unnumbered}

\markboth{Appendix C}{Appendix C}

The Intel Processor Traces (PT) is a CPU feature that records the program execution by encoding packets in a highly compressed binary format that can be used to reconstruct execution flow with a timestamp on every instruction. PT has extensive coverage and relatively small overhead,[^1] which is usually below `5%`. Its main usages are postmortem analysis and root-causing performance glitches.

Intel处理器跟踪(PT: Processor Traces)是一项CPU功能特性，它通过将数据包编码为高度压缩的二进制格式来记录程序执行过程，这些数据包可用于重建执行流程，并为每条指令添加时间戳。PT具有广泛的覆盖范围和相对较小的开销[^1]，通常低于 `5%`。它的主要用途是事后分析和性能故障的根本原因分析。

## Workflow 工作流程 {.unnumbered .unlisted}

Similar to sampling techniques, PT does not require any modifications to the source code. All you need to collect traces is just to run the program under the tool that supports PT. Once PT is enabled and the benchmark launches, the analysis tool starts writing PT packets to DRAM. 

与采样技术类似，PT不需要对源代码进行任何修改。您只需使用支持PT的工具运行程序即可收集跟踪数据。启用PT并启动基准测试后，分析工具会开始将PT数据包写入DRAM。

Similar to LBR (Last Branch Records), Intel PT works by recording branches. At runtime, whenever a CPU encounters any branch instruction, PT will record the outcome of this branch. For a simple conditional jump instruction, a CPU will record whether it was taken (`T`) or not taken (`NT`) using just 1 bit. For an indirect call, PT will record the destination address. Note that unconditional branches are ignored since we statically know their targets. 

与最后分支记录（LBR: Last Branch Records）类似，Intel PT的工作原理是记录分支。在运行时，每当CPU遇到任何分支指令时，PT都会记录该分支的结果。对于简单的条件跳转指令，CPU只需使用1位即可记录其是否跳转（`T`）或未挑战（`NT`）。对于间接调用，PT会记录目标地址。请注意，无条件分支会被忽略，因为我们静态地知道它们的目标地址。

An example of encoding for a small instruction sequence is shown in Figure @fig:PT_encoding. Instructions like `PUSH`, `MOV`, `ADD`, and `CMP` are ignored because they don't change the control flow. However, the `JE` instruction may jump to `.label`, so its result needs to be recorded. Later there is an indirect call for which the destination address is saved.

图 @fig:PT_encoding 展示了一个小型指令序列的编码示例。像 `PUSH`、`MOV`、`ADD` 和 `CMP` 这样的指令会被忽略，因为它们不会改变控制流。但是，`JE` 指令可能会跳转到 `.label`，因此需要记录其结果。之后还有一个间接调用，其目标地址会被保存。

![Intel Processor Traces encoding Intel处理器跟踪编码](../../img/appendix-D/PT_encoding.jpg){#fig:PT_encoding width=80%}

At the time of analysis, we bring together the application binary and the collected PT trace. A software decoder needs the application binary file to reconstruct the execution flow of the program. It starts from the entry point and then uses collected traces as a lookup reference to determine the control flow. 

在分析时，我们将应用程序二进制文件和收集到的PT跟踪文件放在一起。软件解码器需要应用程序二进制文件来重构程序的执行流程。它从入口点开始，然后使用收集到的跟踪文件作为查找参考来确定控制流。

Figure @fig:PT_decoding shows an example of decoding Intel Processor Traces. Suppose that the `PUSH` instruction is an entry point of the application binary file. Then `PUSH`, `MOV`, `ADD`, and `CMP` are reconstructed as-is without looking into encoded traces. Later, the software decoder encounters a `JE` instruction, which is a conditional branch and for which we need to look up the outcome. According to the traces in Figure @fig:PT_decoding, `JE` was taken (`T`), so we skip the next `MOV` instruction and go to the `CALL` instruction. Again, `CALL(edx)` is an instruction that changes the control flow, so we look up the destination address in encoded traces, which is `0x407e1d8`. 

图 @fig:PT_decoding 展示了一个解码Intel处理器跟踪文件的示例。假设 `PUSH` 指令是应用程序二进制文件的入口点。那么，`PUSH`、`MOV`、`ADD` 和 `CMP` 指令将直接重构，无需查看编码后的跟踪文件。之后，软件解码器会遇到 `JE` 指令，这是一个条件分支指令，我们需要查找其执行结果。根据图 @fig:PT_decoding 中的跟踪信息，`JE` 指令被执行（`T`），因此我们跳过下一条 `MOV` 指令，直接执行 `CALL` 指令。同样，`CALL(edx)` 是一条改变控制流的指令，所以我们在编码后的跟踪信息中查找目标地址，即 `0x407e1d8`。

![Intel Processor Traces decoding Intel处理器跟踪解码](../../img/appendix-D/PT_decoding.jpg){#fig:PT_decoding width=90%}

Instructions highlighted in yellow were executed when our program was running. Note that this is an *exact* reconstruction of program execution; we did not skip any instructions. Later we can map assembly instructions back to the source code by using debug information and have a log of source code that was executed line by line.

黄色高亮显示的指令是程序运行时执行的。请注意，这是程序执行过程的*精确（exact）*重构；我们没有跳过任何指令。之后，我们可以利用调试信息将汇编指令映射回源代码，并获得逐行执行的源代码日志。

## Timing Packets 时序包 {.unnumbered .unlisted}

With Intel PT, not only execution flow can be traced but also timing information. In addition to saving jump destinations, PT can also emit timing packets. Figure @fig:PT_timings provides a visualization of how time packets can be used to restore timestamps for instructions. As in the previous example, we first see that `JNZ` was not taken (`NT`), so we update it and all the instructions above with timestamp 0ns. Then we see a timing update of 2ns and `JE` being taken, so we update it and all the instructions above `JE` (and below `JNZ`) with timestamp 2ns. After that, there is an indirect call (`CALL(edx)`), but no timing packet is attached to it, so we do not update timestamps. Then we see that 100ns elapsed, and `JB` was not taken, so we update all the instructions above it with the timestamp of 102ns.

借助Intel PT，不仅可以追踪执行流程，还可以追踪时间信息。除了保存跳转目标之外，PT 还可以发出时间包。图 @fig:PT_timings 展示了如何使用时序包来恢复指令的时间戳。与之前的示例一样，我们首先看到 `JNZ` 指令未跳转（即 `NT`），因此我们将它及其上方所有指令的时间戳更新为 0ns。然后我们看到时间更新为2ns，并且执行了 `JE` 指令，因此我们将它及其上方（以及 `JNZ` 下方）所有指令的时间戳更新为2ns。之后，出现了一个间接调用 `CALL(edx)`，但由于没有时间包附加到该指令，因此我们不更新其时间戳。然后我们看到100ns过去了，`JB` 没有被执行，所以我们用102ns的时间戳更新它上面的所有指令。

![Intel Processor Traces timings Intel处理器跟踪时序](../../img/appendix-D/PT_timings.jpg){#fig:PT_timings width=90%}

In the example shown in Figure @fig:PT_timings, instruction data (control flow) is perfectly accurate, but timing information is less accurate. Obviously, `CALL(edx)`, `TEST`, and `JB` instructions were not happening at the same time, yet we do not have more accurate timing information for them. Having timestamps enables us to align the time interval of our program with another event in the system, and it's easy to compare to wall clock time. Trace timing in some implementations can further be improved by a cycle-accurate mode, in which the hardware keeps a record of cycle counts between normal packets (see more details in [@IntelOptimizationManual, Volume 3C, Chapter 36]).

在图 @fig:PT_timings 所示的示例中，指令数据（控制流）非常精确，但时间信息精度较低。显然，`CALL(edx)`、`TEST` 和 `JB` 指令并非同时执行，但我们却没有更精确的时序信息。时间戳使我们能够将程序的时间间隔与系统中的其他事件对齐，并且可以轻松地与实际时钟时间进行比较。在某些实现中，可以通过周期精确模式进一步提高跟踪计时精度，在这种模式下，硬件会记录正常数据包之间的周期计数（更多详情请参见 [@IntelOptimizationManual, Volume 3C, Chapter 36]）。

## Collecting and Decoding Traces 收集和解码跟踪信息 {.unnumbered .unlisted}

Intel PT traces can be easily collected with the Linux `perf` tool:

可以使用Linux `perf` 工具轻松收集Intel PT跟踪信息：

```bash
$ perf record -e intel_pt/cyc=1/u -- ./a.out
```

In the command line above, I asked the PT mechanism to update timing information every cycle. But likely, it will not increase our accuracy greatly since timing packets will only be sent when paired with another control flow packet.

在上面的命令行中，我要求PT机制每个周期更新一次时间信息。但这样做可能不会显著提高精度，因为时间数据包只有在与另一个控制流数据包配对时才会发送。

After collecting, raw PT traces can be obtained by executing:

收集完成后，可以通过执行以下命令获取原始PT跟踪数据：

```bash
$ perf report -D > trace.dump
```

PT bundles up to 6 conditional branches before it emits a timing packet. Since the Intel Skylake CPU generation, timing packets have cycle count elapsed from the previous packet. If we then look into the `trace.dump`, we might see something like the following:

PT在发出计时数据包之前最多会打包6个条件分支。自Intel Skylake CPU一代以来，计时数据包会包含从上一个数据包开始计算的周期计数。如果我们查看 `trace.dump` 文件，可能会看到类似以下内容：

```
000073b3: 2d 98 8c  TIP 0x8c98     // target address (IP)
000073b6: 13        CYC 0x2        // timing update
000073b7: c0        TNT TNNNNN (6) // 6 conditional branches
000073b8: 43        CYC 0x8        // 8 cycles passed
000073b9: b6        TNT NTTNTT (6)
```

The raw PT packets shown above are not very useful for performance analysis. To decode processor traces to human-readable form, you can execute:

上面显示的原始PT数据包对于性能分析的用处不大。要将处理器跟踪解码为人类可读的形式，您可以执行以下命令：

```bash
$ perf script --ns --itrace=i1t -F time,srcline,insn,srccode
```

Below is an example of decoded traces:

以下是解码后的跟踪示例：

```
timestamp       srcline   instruction      srccode
...
253.555413143:  a.cpp:24  call 0x35c       foo(arr, j);
253.555413143:  b.cpp:7   test esi, esi    for (int i = 0; i <= n; i++)
253.555413508:  b.cpp:7   js 0x1e
253.555413508:  b.cpp:7   movsxd rsi, esi
...
```

I only show a small snippet from the long execution log. In this log, we have traces of *every* instruction executed while our program was running. We can literally observe every step that was made by the program. It is a very strong foundation for further functional and performance analysis.

我只展示了完整执行日志中的一小部分。在这个日志中，我们记录了程序运行期间*每一条*指令的执行轨迹。我们可以清晰地观察到程序执行的每一个步骤。这为进一步的功能和性能分析奠定了坚实的基础。

## Use Cases 使用案例 {.unnumbered .unlisted}

1. **Analyze performance glitches**: because PT captures the entire instruction stream, it is possible to analyze what was going on during the small-time period when the application was not responding. More detailed examples can be found in an [article](https://easyperf.net/blog/2019/09/06/Intel-PT-part3)[^2] on Easyperf blog.
2. **Postmortem debugging**: PT traces can be replayed by traditional debuggers like `gdb`. In addition to that, PT provides call stack information, which is *always* valid even if the stack is corrupted.[^3] PT traces could be collected on a remote machine once and then analyzed offline. This is especially useful when the issue is hard to reproduce or access to the system is limited. 
3. **Introspect execution of the program**:
   - We can immediately tell if a code path was never executed. 
   - Thanks to timestamps, it's possible to calculate how much time was spent waiting while spinning on a lock attempt, etc.
   - Security mitigation by detecting specific instruction patterns.

1. **分析性能故障**：由于PT会捕获整个指令流，因此可以分析应用程序无响应的短暂时间段内发生了什么。更多详细示例请参见Easyperf博客上的[文章](https://easyperf.net/blog/2019/09/06/Intel-PT-part3)[^2]。
2. **事后调试**：P 跟踪信息可以使用诸如 `gdb` 之类的传统调试器进行重放。此外，PT还提供调用堆栈信息，即使堆栈损坏，该信息也*始终always*有效[^3]。PT跟踪可以在远程机器上一次性收集，然后离线分析。这在问题难以重现或系统访问受限时尤其有用。
3. **对程序执行情况进行反查**：
   - 我们可以立即判断某个代码路径是否从未执行过。
   - 借助时间戳，可以计算在尝试锁定时等待的时间等。
   - 通过检测特定指令模式来缓解安全问题。

## Disk Space and Decoding Time 磁盘空间和解码时间 {.unnumbered .unlisted}

Even taking into account the compressed format of the trace, encoded data can consume a lot of disk space. Typically, it's less than 1 byte per instruction, however taking into account the speed at which CPU executes instructions, it is still a lot. Depending on the workload, it's very common for the CPU to encode PT at a speed of 100 MB/s. Decoded traces might easily be ten times more (~1GB/s). This makes PT not practical for use on long-running workloads. But it is affordable to run it for a short time, even on a big workload. In this case, the user can attach to the running process just for the time when the glitch happened. Or they can use a circular buffer, where new traces will overwrite old ones, i.e., always having traces for the last 10 seconds or so.

即使考虑到跟踪的压缩格式，编码数据仍然会占用大量磁盘空间。通常，每条指令占用不到1字节，但考虑到CPU执行指令的速度，这仍然相当可观。根据工作负载的不同，CPU通常以100MB/s 的速度对程序跟踪(PT)进行编码。解码后的跟踪数据量可能轻松达到编码速度的10倍多（约1GB/s）。这使得PT不适用于长时间运行的工作负载。但即使在大型工作负载下，短时间运行PT也是可行的。在这种情况下，用户可以仅在故障发生时附加到正在运行的进程。或者，他们可以使用循环缓冲区，新的跟踪数据会覆盖旧的，也就是说，始终保留最近10秒左右的跟踪数据。

Users can limit collection even further in several ways. They can limit collecting traces only on user/kernel space code. Also, there is an address range filter, so it's possible to opt in and opt out of tracing dynamically to limit the memory bandwidth. This allows us to trace just a single function or even a single loop.

用户还可以通过多种方式进一步限制跟踪数据收集。他们可以将跟踪范围限制在用户/内核空间代码。此外，还有一个地址范围过滤器，因此可以动态地选择启用或禁用跟踪，从而限制内存带宽。这允许我们仅跟踪单个函数甚至单个循环。

Decoding PT traces can take a long time because it has to follow along with disassembled instructions from the binary and reconstruct the flow. On an Intel Core i5-8259U machine, for a workload that runs for 7 milliseconds, encoded PT trace consumes around 1MB of disk space. Decoding this trace using `perf script -F time,ip,sym,symoff,insn` takes ~20 seconds[^4] and the output consumes ~1.3GB of disk space. 

解码PT跟踪可能耗时较长，因为它需要跟踪从二进制文件中反汇编的指令并重建流程。在Intel Core i5-8259U机器上，对于运行时间为7毫秒的工作负载，编码后的PT跟踪大约占用1MB的磁盘空间。使用 `perf script -F time,ip,sym,symoff,insn` 解码此跟踪大约需要20秒[^4]，并且输出结果占用约1.3GB的磁盘空间。

## Tools 工具 {.unnumbered .unlisted}

Besides Linux perf, several other tools support Intel PT. First, Intel VTune Profiler has *Anomaly Detection* analysis type that uses Intel PT. Another popular tool worth mentioning is magic-trace[^5], which collects and displays high-resolution traces of a process.

除Linux perf之外，还有其他一些工具支持Intel PT。首先，Intel VTune Profiler具有使用Intel PT的“*异常检测（Anomaly Detection）*”分析类型。另一个值得一提的常用工具是magic-trace[^5]，它可以收集并显示进程的高分辨率跟踪信息。

## Intel PT References and links Intel PT参考资料和链接 {.unnumbered .unlisted}

* Intel® 64 and IA-32 Architectures Software Developer Manuals [@IntelOptimizationManual, Volume 3C, Chapter 36].
* Whitepaper "Hardware-assisted instruction profiling and latency detection" [@IntelPTPaper].
* Andi Kleen article on LWN, URL: [https://lwn.net/Articles/648154](https://lwn.net/Articles/648154).
* Intel PT Micro Tutorial, URL: [https://sites.google.com/site/intelptmicrotutorial/](https://sites.google.com/site/intelptmicrotutorial/).
* Intel PT documentation in the Linux kernel, URL: [https://github.com/torvalds/linux/blob/master/tools/perf/Documentation/intel-pt.txt](https://github.com/torvalds/linux/blob/master/tools/perf/Documentation/intel-pt.txt).
* Cheatsheet for Intel Processor Trace, URL: [http://halobates.de/blog/p/410](http://halobates.de/blog/p/410).

* Intel® 64和IA-32体系结构软件开发人员手册 [@IntelOptimizationManual，第3C卷，第36章]。
* 白皮书《硬件辅助指令轮廓分析和延迟检测》 [@IntelPTPaper]。
* Andi Kleen在LWN上发表的文章，网址：[https://lwn.net/Articles/648154](https://lwn.net/Articles/648154)。
* Intel PT微教程，网址：[https://sites.google.com/site/intelptmicrotutorial/](https://sites.google.com/site/intelptmicrotutorial/)。
* Linux 内核中的Intel PT文档，网址：[https://github.com/torvalds/linux/blob/master/tools/perf/Documentation/intel-pt.txt](https://github.com/torvalds/linux/blob/master/tools/perf/Documentation/intel-pt.txt)。
* Intel处理器跟踪速查表，网址：[http://halobates.de/blog/p/410](http://halobates.de/blog/p/410)。

[^1]: See more information about Intel PT overhead in [@IntelPTPaper]. 有关Intel PT开销的更多信息，请参阅 [@IntelPTPaper]。
[^2]: Analyze performance glitches with Intel PT 使用Intel PT分析性能故障 -- [https://easyperf.net/blog/2019/09/06/Intel-PT-part3](https://easyperf.net/blog/2019/09/06/Intel-PT-part3)
[^3]: Postmortem debugging with Intel PT 使用Intel PT进行事后分析调试 -- [https://easyperf.net/blog/2019/08/30/Intel-PT-part2](https://easyperf.net/blog/2019/08/30/Intel-PT-part2)
[^4]: When you decode traces with `perf script -F` with `+srcline` or `+srccode` to emit source code, it gets even slower. 当您使用 `perf script -F` 并添加 `+srcline` 或 `+srccode` 参数来解码跟踪信息以生成源代码时，速度会变得更慢。
[^5]: magic-trace - [https://github.com/janestreet/magic-trace](https://github.com/janestreet/magic-trace)
[^6]: Notice that there are instructions executed as a result of the function call (denoted with `...`). 注意，函数调用会执行一些指令（用 `...` 表示）。
