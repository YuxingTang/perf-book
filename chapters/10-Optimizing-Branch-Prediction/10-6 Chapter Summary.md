## Chapter Summary 本章小结 {.unlisted .unnumbered}

\markright{Summary小结}

* Modern processors are very good at predicting branch outcomes. So, I recommend paying attention to branch mispredictions only when the TMA points to a high `Bad Speculation` metric.
* When branch outcome patterns become hard for the CPU branch predictor to follow, the performance of the application may suffer. In this case, the branchless version of an algorithm can be more performant. In this chapter, I showed how branches could be replaced with lookup tables, arithmetic, and selection.
* Branchless algorithms are not universally beneficial. Always measure to find out what works better in your specific case.
* There are indirect ways to reduce the branch misprediction rate by reducing the dynamic number of branch instructions in a program. This approach helps because it alleviates the pressure on branch predictor structures. Examples of such techniques include loop unrolling/vectorization, replacing branches with bitwise operations, and using SIMD instructions.

* 现代处理器非常擅长预测分支结果。因此，我建议仅当TMA指标显示 `错误推测Bac Speculation` 较高时才关注分支预测错误。
* 当CPU分支预测器难以追踪分支结果模式时，应用程序的性能可能会受到影响。在这种情况下，无分支算法的性能可能更高。本章展示了如何使用查找表、算术运算和选择操作来替代分支。
* 无分支算法并非适用于所有情况。务必进行测试，以找出最适合您具体情况的算法。
* 可以通过减少程序中分支指令的动态数量来间接降低分支预测错误率。这种方法之所以有效，是因为它减轻了分支预测器结构的压力。此类技术的示例包括循环展开/向量化、使用位运算替代分支以及使用SIMD指令。

\sectionbreak
