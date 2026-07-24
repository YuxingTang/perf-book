## Questions and Exercises 问题与习题 {.unlisted .unnumbered}

\markright{Questions and Exercises 问题与习题}

1. What is the difference between the CPU core clock and the reference clock?
2. What is the difference between retired and executed instructions?
3. When you increase the frequency, does IPC go up, down, or stay the same?
4. Take a look at the `DRAM BW Use` formula in Table {@tbl:perf_metrics}. Why do you think there is a constant `64`?
5. Measure the bandwidth and latency of the cache hierarchy and memory on the machine you use for development/benchmarking using Intel MLC, Stream, or other tools.
6. Run the application that you're working with on a daily basis. Collect performance metrics. Does anything surprise you?

1. CPU核心时钟频率和参考时钟频率有什么区别？
2. 已执行指令和已完成指令有什么区别？
3. 提高频率时，IPC会上升、下降还是保持不变？
4. 查看表 {@tbl:perf_metrics} 中的 `DRAM BW Use` 公式。你认为为什么会有常数 `64`？
5. 使用Intel MLC、Stream或其他工具，测量你用于开发/基准测试的机器上缓存层次结构和内存的带宽和延迟。
6. 运行你日常使用的应用程序。收集性能指标。有什么让你感到意外的吗？

**Capacity Planning Exercise**: Imagine you are the owner of four applications we benchmarked in the case study. The management of your company has asked you to build a small computing farm for each of those applications with the primary goal being maximum performance (throughput). The spending budget you were given is tight but enough to buy 1 mid-level server system (Mac Studio, Supermicro/Dell/HPE server rack, etc.) or 1 high-end desktop (with overclocked CPU, liquid cooling, top GPU, fast DRAM) to run each workload, so 4 machines in total. Those could be all four different systems. Also, you can use the money to buy 3-4 low-end systems; the choice is yours. The management wants to keep it under $10,000 per application, but they are flexible (10--20%) if you can justify the expense. Assume that Stockfish remains single-threaded. Look at the performance characteristics for the four applications once again and write down which computer parts (CPU, memory, discrete GPU if needed) you would buy for each of those workloads. Which parameters you will prioritize? Where will you go with the most expensive part? Where you can save money? Try to describe it in as much detail as possible, and search the web for exact components and their prices. Account for all the components of the system: motherboard, disk drive, cooling solution, power delivery unit, rack/case/tower, etc. What additional performance experiments would you run to guide your decision?

**容量规划练习**：假设你是我们在案例研究中测试的四个应用程序的所有者。公司管理层要求你为每个应用程序搭建一个小型计算集群，主要目标是实现最高性能（吞吐量）。预算虽然紧张，但足以购买一台中端服务器系统（例如 Mac Studio、Supermicro/Dell/HPE服务器机架等）或一台高端台式机（配备超频CPU、液冷散热、顶级GPU和高速DRAM）来运行每个工作负载，总共需要4台机器。这4台机器可以是不同的系统。此外，你还可以用这笔钱购买3-4台低端系统；选择权在你。管理层希望每个应用程序的成本控制在1万美元以内，但如果你能证明这笔支出的合理性，他们可以灵活调整（10-20%）。假设Stockfish仍然是单线程的。请再次查看这四个应用程序的性能特征，并写下你会为每个工作负载购买哪些计算机组件（CPU、内存、如果需要的话也包含独立GPU）。你会优先考虑哪些参数？你会把最贵的组件用在哪里？哪些地方可以节省成本？请尽可能详细地描述您的系统配置，并在网上搜索具体的组件及其价格。请考虑系统的所有组件：主板、硬盘、散热方案、电源、机架/机箱/塔式机箱等等。为了更好地做出决定，您还会进行哪些额外的性能测试？