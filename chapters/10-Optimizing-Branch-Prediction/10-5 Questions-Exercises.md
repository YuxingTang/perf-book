## Questions and Exercises 问题与习题 {.unlisted .unnumbered}

\markright{Questions and Exercises问题与习题}

1. Revisit the code example shown in [@lst:LookupBranches] on the right. Suppose we start frequently getting numbers outside of the `[0-50)` range. This will introduce many new mispredictions for the branch that guards against out-of-bounds access to the `buckets` array. How you will change the code to eliminate those newly introduced mispredictions?
2. Solve the following lab assignments using techniques we discussed in this chapter:
- `perf-ninja::branches_to_cmov_1`
- `perf-ninja::lookup_tables_1`
- `perf-ninja::virtual_call_mispredict`
- `perf-ninja::conditional_store_1`
3. Run the application that you're working with daily. Collect the TMA breakdown and check the `BadSpeculation` metric. Look at the code that is attributed with the most number of branch mispredictions. Is there a way to avoid branches using the techniques we discussed in this chapter?

**Coding exercise**: write a microbenchmark that will experience a 50% misprediction rate or get as close as possible. Your goal is to write a code in which half of all branch instructions are mispredicted. That is not as simple as you may think. Some hints and ideas:

* Branch misprediction rate is measured as `BR_MISP_RETIRED.ALL_BRANCHES / BR_INST_RETIRED.ALL_BRANCHES`.
* If you're coding in C++, you can use 1) the Google benchmark library similar to perf-ninja, 2) write a regular console program and collect CPU counters with Linux `perf`, or 3) integrate the libpfm library into the microbenchmark (see [@sec:MarkerAPI]).
* There is no need to invent some complicated algorithm. A simple approach would be to generate a pseudo-random number in the range `[0;100)` and check if it is less than 50. Random numbers can be pre-generated ahead of time.
* Keep in mind that modern CPUs can remember long (but still limited) sequences of branch outcomes.

1. 回顾右侧 [@lst:LookupBranches] 中所示的代码示例。假设我们开始频繁地获取超出 `[0-50)` 范围的数字。这将导致用于防止越界访问 `buckets` 数组的分支出现许多新的预测错误。您将如何修改代码以消除这些新引入的预测错误？
2. 使用本章讨论的技术完成以下实验作业：
- `perf-ninja::branches_to_cmov_1`
- `perf-ninja::lookup_tables_1`
- `perf-ninja::virtual_call_mispredict`
- `perf-ninja::conditional_store_1`
3. 运行您日常使用的应用程序。收集TMA细分数据并检查 `BadSpeculation` 指标。查看分支预测错误次数最多的代码。有没有办法利用本章讨论的技术来避免分支？

**编程练习**：编写一个微基准测试程序，使其分支预测错误率达到50%或尽可能接近该值。你的目标是编写一段代码，使一半的分支指令预测错误。这并不像你想象的那么简单。以下是一些提示和思路：

* 分支预测错误率的计算公式为 `BR_MISP_RETIRED.ALL_BRANCHES / BR_INST_RETIRED.ALL_BRANCHES`。
* 如果你使用C++编程，可以使用以下方法：1）使用类似于perf-ninja的Google基准测试库；2）编写一个普通的控制台程序，并使用Linux的 `perf` 命令收集CPU计数器；或者3）将libpfm库集成到微基准测试程序中（参见 [@sec:MarkerAPI]）。
* 无需发明复杂的算法。一个简单的办法是生成一个介于0到100之间的伪随机数，并检查它是否小于50。随机数可以预先生成。
* 请注意，现代CPU可以记住较长（但仍然有限）的分支结果序列。
