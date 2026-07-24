## Retired vs. Executed Instruction  已完成Retired vs. 已执行Executed指令

Modern processors typically execute more instructions than the program flow requires. This happens because some instructions are executed speculatively, as discussed in [@sec:SpeculativeExec]. For most instructions, the CPU commits results once they are available, and all preceding instructions have already been retired. But for instructions executed speculatively, the CPU keeps their results without immediately committing their results. When the speculation turns out to be correct, the CPU unblocks such instructions and proceeds as normal. But when the speculation turns out to be wrong, the CPU throws away all the changes done by speculative instructions and does not retire them. So, an instruction processed by the CPU can be executed but not necessarily retired. Taking this into account, we can usually expect the number of executed instructions to be higher than the number of retired instructions.

现代处理器通常会执行比程序流程所需更多的指令。这是因为某些指令是前瞻性执行的，如 [@sec:SpeculativeExec] 中所述。对于大多数指令，CPU会在结果可用时提交结果，此时所有先前的指令都已被完成。但对于前瞻性执行的指令，CPU会保留其结果，但不会立即提交。当前瞻正确时，CPU会解除阻塞这些指令并继续执行。但当前瞻错误时，CPU会丢弃前瞻性指令所做的所有更改，并且不会完成它们。因此，CPU处理的指令可能已被执行，但不一定已被完成。考虑到这一点，我们通常可以预期已执行的指令数量会高于已完成的指令数量。

There is an exception. Certain instructions are recognized as idioms and are resolved without actual execution. Some examples of this are NOP, move elimination, and zeroing, as discussed in [@sec:uarchBE]. Such instructions do not require an execution unit but are still retired. So, theoretically, there could be a case when the number of retired instructions is higher than the number of executed instructions.

但也有例外。某些指令被识别为惯用法idioms，无需实际执行即可解析。例如，NOP指令、移动消除指令和清零指令，如 [@sec:uarchBE] 中所述。这类指令不需要执行单元，但仍然会被完成retired。因此，理论上，可能会出现已完成指令数大于已执行指令数的情况。

There is a performance monitoring counter (PMC) in most modern processors that collects the number of retired instructions. There is no performance event to collect executed instructions, though there is a way to collect executed and retired *$\mu$ops* as we shall see soon. The number of retired instructions can be easily obtained with Linux `perf` by running:

大多数现代处理器中都有一个性能监控计数器(PMC)，用于收集已完成指令数。虽然没有性能事件来收集已执行指令数，但正如我们稍后将看到的，有一种方法可以收集已执行和已完成的微操作 *$\mu$ops*。使用 Linux 的 `perf` 命令可以轻松获取已完成指令数，只需运行以下命令：

```bash
$ perf stat -e instructions -- ./a.exe
  2173414  instructions  #    0.80  insn per cycle 
# or just simply run:
$ perf stat -- ./a.exe
```
