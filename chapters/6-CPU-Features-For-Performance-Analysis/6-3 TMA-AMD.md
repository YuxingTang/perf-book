### TMA on AMD Platforms AMD平台上的TMA分析 {#sec:secTMA_AMD}

Starting from Zen4, AMD processors support Level-1 and Level-2 TMA analysis. According to AMD documentation, it is called "Pipeline Utilization" analysis, but the idea remains the same. The L1 and L2 buckets are also very similar to Intel's. Since kernel 6.2, Linux users can utilize the `perf` tool to collect the pipeline utilization data.

从Zen4开始，AMD处理器支持Level-1和Level-2 TMA分析。根据AMD的文档，它被称为“流水线利用率（Pipeline Utilization）”分析，但其原理与TMA分析相同。L1和L2的存储桶也与Intel的非常相似。从内核6.2开始，Linux用户可以使用`perf`工具来收集流水线利用率数据。

Next, we will examine the [Crypto++](https://github.com/weidai11/cryptopp)[^1] implementation of SHA-256 (Secure Hash Algorithm 256), the fundamental cryptographic algorithm in Bitcoin mining. Crypto++ is an open-source C++ class library of cryptographic algorithms and contains an implementation of many algorithms, not just SHA-256. However, for our example, I disabled benchmarking of other algorithms by commenting out corresponding lines in the `BenchmarkUnkeyedAlgorithms` function in `bench1.cpp`.

接下来，我们将研究[Crypto++](https://github.com/weidai11/cryptopp)[^1]中SHA-256（安全哈希算法256）的实现，SHA-256是比特币挖矿中的基础加密算法。Crypto++是一个开源的C++加密算法类库，其中包含许多算法的实现，而不仅仅是SHA-256。然而，在我们的示例中，我通过注释掉 `bench1.cpp` 文件中 `BenchmarkUnkeyedAlgorithms` 函数中相应的行，禁用了其他算法的基准测试。

I ran the test on an AMD Ryzen 9 7950X machine with Ubuntu 22.04, Linux kernel 6.5. I compiled Crypto++ version 8.9 using GCC 12.3 C++ compiler. I used the default `-O3` optimization option, but it doesn't impact performance much since the code is written with x86 intrinsics (see [@sec:secIntrinsics]) and utilizes the SHA x86 ISA extension. 

我在一台搭载AMD Ryzen 9 7950X处理器、Ubuntu 22.04操作系统和Linux内核6.5的机器上运行了测试。我使用GCC 12.3 C++编译器编译了Crypto++ 8.9版本。我使用了默认的 `-O3` 优化选项，但由于代码是用x86内联函数编写的（参见 [@sec:secIntrinsics]），并且利用了SHA x86 ISA扩展，因此它对性能的影响不大。

Below is the command I used to obtain L1 and L2 pipeline utilization metrics. The output was trimmed and some statistics were dropped to remove unnecessary distraction.

以下是我用来获取L1级和L2级流水线利用率指标的命令。输出结果经过精简，并删除了一些统计信息，以避免不必要的干扰。

```bash
$ perf stat -M PipelineL1,PipelineL2 -- ./cryptest.exe b1 10
 0.0 %  bad_speculation_mispredicts        (20.08%) 
 0.0 %  bad_speculation_pipeline_restarts  (20.08%)
 0.0 %  bad_speculation                    (20.08%)
 6.1 %  frontend_bound                     (20.00%)
 6.1 %  frontend_bound_bandwidth           (20.00%)
 0.1 %  frontend_bound_latency             (20.00%)
65.9 %  backend_bound_cpu                  (20.00%)
 1.7 %  backend_bound_memory               (20.00%)
67.5 %  backend_bound                      (20.00%)
26.3 %  retiring                           (20.08%)
20.2 %  retiring_fastpath                  (19.99%)
 6.1 %  retiring_microcode                 (19.99%)
```

In the output, numbers in brackets indicate the percentage of runtime duration, when a metric was monitored. As we can see, each metric was monitored only 20% of the time due to multiplexing. In our case it is likely not a concern as SHA256 has consistent behavior, however it may not always be the case. To minimize the impact of multiplexing, you can collect a limited set of metrics in a single run, e.g., `perf stat -M frontend_bound,backend_bound`.

输出结果中，括号内的数字表示指标被监控的运行时长百分比。我们可以看到，由于多路复用，每个指标的监控时间仅占20%。就我们的情况而言，这可能不是问题，因为SHA256的行为比较稳定，但情况并非总是如此。为了尽量减少多路复用的影响，您可以在单次运行中收集有限的指标集，例如，`perf stat -M frontend_bound,backend_bound`。

A description of pipeline utilization metrics shown above can be found in [@AMDUprofManual, Chapter 2.8 Pipeline Utilization]. By looking at the metrics, we can see that branch mispredictions are not happening in SHA256 (`bad_speculation` is 0%). Only 26.3% of the available dispatch slots were used (`retiring`), which means the remaining 73.7% were wasted due to frontend and backend stalls.

上述流水线利用率指标的说明可在 [@AMDUprofManual，第2.8章--流水线利用率] 中找到。通过查看这些指标，我们可以看到SHA256中没有出现分支预测错误（`bad_speculation` 为 0%）。可用调度槽位仅使用了26.3%（`retiring`），这意味着剩余的73.7%由于前端和后端停顿而被浪费。

Advanced cryptography instructions are not trivial, so internally they are broken into smaller pieces ($\mu$ops). Once a processor encounters such an instruction, it retrieves $\mu$ops for it from the microcode. Microoperations are fetched from the microcode sequencer with a lower bandwidth than from regular instruction decoders, making it a potential source of performance bottlenecks. Crypto++ SHA256 implementation heavily uses instructions such as `SHA256MSG2`, `SHA256RNDS2`, and others which consist of multiple $\mu$ops according to [uops.info](https://uops.info/table.html)[^2] website. 

高级加密指令并非易事，因此在内部会被拆分成更小的单元（微操作）。处理器一旦遇到此类指令，就会从微码中获取相应的微操作。微操作从微码序列器中获取的带宽低于常规指令译码器，这使其成为潜在的性能瓶颈。Crypto++ SHA256实现大量使用了诸如： `SHA256MSG2`、`SHA256RNDS2` 等指令，这些指令根据 [uops.info](https://uops.info/table.html)[^2] 网站的资料显示，通常由多个微操作组成。

The `retiring_microcode` metric indicates that 6.1% of dispatch slots were used by microcode operations that eventually retired. When comparing with its sibling metric `retiring_fastpath`, we can say that roughly every 4th instruction was a microcode operation. If we now look at the `frontend_bound_bandwidth` metric, we will see that 6.1% of dispatch slots were unused due to bandwidth bottleneck in the CPU frontend. This suggests that 6.1% of dispatch slots were wasted because the microcode sequencer has not been providing $\mu$ops while the backend could have consumed them. In this example, the `retiring_microcode` and `frontend_bound_bandwidth` metrics are tightly connected, however, the fact that they are equal is merely a coincidence. 

`retiring_microcode` 指标表明，6.1%的调度槽被最终完成的微码操作所占用。与同级指标 `retiring_fastpath` 相比，我们可以说，大约每4条指令中就有一条是微码操作。现在我们来看一下 `frontend_bound_bandwidth` 指标，会发现由于CPU前端的带宽瓶颈，有6.1%的调度槽未被使用。这表明，有6.1%的调度槽被浪费了，因为微码序列器没有提供微操作（μops），而后端本可以消耗这些微操作。在这个例子中，`retiring_microcode` 和 `frontend_bound_bandwidth` 指标紧密相关，但它们相等仅仅是巧合。

The majority of cycles are stalled in the CPU backend (`backend_bound`), but only 1.7% of cycles are stalled waiting for memory accesses (`backend_bound_memory`). So, we know that the benchmark is mostly limited by the computing capabilities of the machine. As you will know from Part 2 of this book, it could be related to either data flow dependencies or execution throughput of certain cryptographic operations. They are less frequent than traditional `ADD`, `SUB`, `CMP`, and other instructions and thus can be often executed only on a single execution unit. A large number of such operations may saturate the execution throughput of this particular unit. Further analysis should involve a closer look at the source code and generated assembly, checking execution port utilization, finding data dependencies, etc. 

大部分周期都停滞在CPU后端（`backend_bound`），但只有1.7%的周期停滞在等待内存访问（`backend_bound_memory`）。因此，我们知道基准测试主要受限于机器的计算能力。正如本书第二部分所述，这可能与数据流依赖性或某些加密操作的执行吞吐率有关。它们比传统的 `ADD`、`SUB`、`CMP` 等指令的执行频率低，因此通常只能在单个执行单元上执行。大量的此类操作可能会使该特定单元的执行吞吐量达到饱和。进一步的分析应包括仔细查看源代码和生成的汇编代码，检查执行端口利用率，查找数据依赖关系等。

In summary, the Crypto++ implementation of SHA-256 on AMD Ryzen 9 7950X utilizes only 26.3% of the available dispatch slots; 6.1% of the dispatch slots were wasted due to the microcode sequencer bandwidth, and 65.9% were stalled due to lack of computing resources of the machine. The algorithm certainly hits a few hardware limitations, so it's unclear if its performance can be improved or not.

总而言之，在AMD Ryzen 9 7950X上，Crypto++实现的SHA-256算法仅利用了26.3%的可用调度槽；6.1%的调度槽由于微码序列器带宽而被浪费，65.9%的调度槽由于机器计算资源不足而停滞。该算法显然遇到了一些硬件限制，因此其性能是否可以提高尚不清楚。

When it comes to Windows, at the time of writing, TMA methodology is only supported on AMD server platforms (codename Genoa), and not on client systems (codename Raphael). TMA support was added in AMD uProf version 4.1, but only in the command line tool `AMDuProfPcm` tool which is part of AMD uProf installation. You can consult [@AMDUprofManual, Chapter 2.8 Pipeline Utilization] for more details on how to run the analysis. The graphical version of AMD uProf doesn't have the TMA analysis yet. 

就Windows系统而言，截至撰写本文时，TMA方法仅在AMD服务器平台（代号：Genoa）上受支持，而不在客户端系统（代号：Raphael）上受支持。AMD的uProf 4.1版本新增了TMA支持，但仅限于命令行工具 `AMDuProfPcm`，该工具是AMD uProf安装包的一部分。您可以参考 [@AMDUprofManual，第2.8章--流水线利用率] 了解更多关于如何运行分析的详细信息。AMD uProf的图形化版本目前尚不支持TMA分析。

[^1]: Crypto++ - [https://github.com/weidai11/cryptopp](https://github.com/weidai11/cryptopp)
[^2]: uops.info - [https://uops.info/table.html](https://uops.info/table.html)
