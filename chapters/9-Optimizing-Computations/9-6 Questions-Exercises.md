## Questions and Exercises 问题和习题 {.unlisted .unnumbered}

\markright{Questions and Exercises问题和习题}

1. Solve the following lab assignments using techniques we discussed in this chapter:
- `perf-ninja::function_inlining_1` 
- `perf-ninja::vectorization` 1 and 2
- `perf-ninja::dep_chains` 1 and 2
- `perf-ninja::compiler_intrinsics` 1 and 2
- `perf-ninja::loop_interchange` 1 and 2
- `perf-ninja::loop_tiling_1`
2. Describe the steps you will take to find out if an application is using all the opportunities for utilizing SIMD code.
3. Practice doing loop optimizations manually on a real code. Make sure that all the tests are still passing.
4. Suppose you're dealing with an application that has a very low IpCall (instructions per call) metric. What optimizations you will try to apply/force?
5. Run the application that you're working with daily. Find the hottest loop. Is it vectorized? Is it possible to force compiler autovectorization? Is the loop bottlenecked by dependency chains or execution throughput?

1. 使用本章讨论的技术完成以下实验作业：
- `perf-ninja::function_inlining_1`
- `perf-ninja::vectorization` 1和2
- `perf-ninja::dep_chains` 1和2
- `perf-ninja::compiler_intrinsics` 1和2
- `perf-ninja::loop_interchange` 1和2
- `perf-ninja::loop_tiling_1`
2. 描述你将采取哪些步骤来确定应用程序是否充分利用了SIMD代码。
3. 在实际代码上手动练习循环优化。确保所有测试仍然通过。
4. 假设你正在处理一个IpCall（每次调用指令数）指标非常低的应用程序。你会尝试应用/强制执行哪些优化？
5. 运行你每天都在使用的应用程序。找出最耗时的循环。它是否已向量化？是否可以强制编译器自动向量化？该循环的瓶颈是依赖链还是执行吞吐量？