## Workaround Memory Bandwidth Limitations 绕过内存带宽限制的方法

As discussed in [@sec:UarchMainmemory], a processor gets the data from memory through a memory bus. With the latest DDR5 memory technology, the maximum theoretical memory bandwidth is 51.2 GB/s per channel. Modern systems have multiple memory channels; for example, a typical laptop usually has two memory channels, while server systems can have from 4 to 12 channels. It may seem that even a laptop can move a lot of data back and forth each second, but in reality, memory bandwidth becomes a limitation in many applications.

如 [@sec:UarchMainmemory] ​​中所述，处理器通过内存总线从内存获取数据。采用最新的DDR5内存技术，每个通道的理论最大内存带宽为51.2GB/s。现代系统拥有多个内存通道；例如，一台典型的笔记本电脑通常有2个内存通道，而服务器系统则可能有4到12个通道。虽然笔记本电脑似乎每秒也能来回传输大量数据，但实际上，内存带宽在许多应用中都成为了瓶颈。

We should keep in mind that memory channels are shared between all the cores in a system. Once many cores engage in a memory-intensive activity, the traffic flowing through the memory bus can become congested. This may lead to increased wait times for memory requests to return. Modern systems are designed to accommodate multiple memory-demanding threads working at the same time, so usually you need multiple threads to saturate the memory bandwidth. Emerging AI workloads are known to be extremely "memory hungry" and highly parallelized, so memory bandwidth is the number one bottleneck for them.

我们需要记住，内存通道在系统中的所有核心之间共享。一旦多个核心同时执行内存密集型任务，流经内存总线的流量就会拥塞。这可能会导致内存请求返回的等待时间增加。现代系统旨在支持多个内存密集型线程同时运行，因此通常需要多个线程才能充分利用内存带宽。新兴的人工智能工作负载以极高的内存需求和高度并行化而著称，因此内存带宽是这些应用的性能瓶颈所在。

The first step in addressing memory bandwidth limitations is to determine the maximum theoretical and expected memory bandwidth. The theoretical maximum memory bandwidth can be calculated from the memory technology specifications as we have shown in [@sec:roofline]. The expected memory bandwidth can be measured using tools like Intel Memory Latency Checker or `lmbench`, which we discussed in [@sec:MemLatBw]. Intel VTune can automatically measure memory bandwidth before the analyzed application starts.

解决内存带宽限制的第一步是确定最大理论内存带宽和预期内存带宽。理论最大内存带宽可以根据内存技术规格计算得出，正如我们在[@sec:roofline]中所述。预期内存带宽可以使用诸如Intel Memory Latency Checker或 `lmbench` 之类的工具进行测量，我们在[@sec:MemLatBw]中讨论过这些工具。Intel VTune可以在被分析应用程序启动之前自动测量内存带宽。

The second step is to measure the memory bandwidth utilization while your application is running. If the amount of memory traffic is close to the maximum measured bandwidth, then the performance of your application is likely to be bound by memory bandwidth. It is a good idea to plot the memory bandwidth utilization over time to see if there are different phases where memory intensity spikes or takes a dip. Intel VTune can provide such a chart if you tick the "Evaluate max DRAM bandwidth" checkbox in the analysis configuration.

第二步是在应用程序运行时测量内存带宽利用率。如果内存流量接近测得的最大带宽，则应用程序的性能很可能受限于内存带宽。最好绘制内存带宽利用率随时间变化的图表，以查看是否存在内存强度峰值或谷值的不同阶段。如果您在分析配置中勾选“评估最大DRAM带宽（Evaluate max DRAM bandwidth）”复选框，Intel VTune可以提供此类图表。

If you have determined that your application is memory bandwidth bound, the first suggestion is to see if you can decrease the memory intensity of your application. It is not always possible, but you can consider disabling some memory-hungry features of your application, recomputing data on the fly instead of caching results, or compressing your data. In the AI space, most Large Language Models (LLMs) are supplied in fp32 precision, which means that each parameter takes 4 bytes. The biggest performance gain can be achieved with quantization techniques, which reduce the precision of the parameters to fp16 or int8. This will reduce the memory traffic by 2x or 4x, respectively. Sometimes, 4-bit and even 5-bit quantization is used, to reduce memory traffic and strike the right balance between inference performance and quality.

如果您已确定应用程序受限于内存带宽，首先建议尝试降低应用程序的内存占用。虽然并非总是可行，但您可以考虑禁用应用程序中一些内存密集型功能、动态的重新计算数据而非缓存结果，或者压缩数据。在人工智能领域，大多数大型语言模型(LLM)都采用fp32精度，这意味着每个参数占用4个字节。使用量化技术可以显著提升性能，将参数精度降低到fp16或int8。这将内存流量分别减少2倍或4倍。有时，为了减少内存流量并在推理性能和质量之间取得平衡，会使用4位甚至5位量化。

It is important to mention that for workloads that saturate available memory bandwidth, code optimizations don't play a big role as they do for compute-bound workloads. For compute-bound applications, code optimizations like vectorization usually translate into large performance gains. However, for memory-bound workloads, vectorization may not have a similar effect since a processor can't make forward progress, simply because it lacks data to work with. We cannot make the memory bus run faster, this is why memory bandwidth is often a hard limitation to overcome.

需要指出的是，对于内存带宽饱和的工作负载，代码优化不像在计算密集型工作负载中那样起着重要作用。对于计算密集型应用，诸如向量化之类的代码优化通常能带来显著的性能提升。然而，对于内存密集型工作负载，向量化可能不会产生类似的效果，因为处理器无法继续执行，原因很简单：它缺少可供处理的数据。我们无法提高内存总线的运行速度，因此内存带宽通常是一个难以突破的瓶颈。

Finally, if all the options have been exhausted, and the memory bandwidth is still a limitation, the only way to improve the situation is to buy better hardware. You can invest money in a server with more memory channels, or DRAM modules with faster transfer speed. This could be an expensive but still, a viable option to speed up your application.

最后，如果所有方法都已尝试过，而内存带宽仍然是瓶颈，那么改善这种情况的唯一途径就是购买更好的硬件。您可以投资购买拥有更多内存通道的服务器，或者购买传输速度更快的DRAM模块。虽然这可能成本较高，但仍然是提升应用程序速度的可行方案。