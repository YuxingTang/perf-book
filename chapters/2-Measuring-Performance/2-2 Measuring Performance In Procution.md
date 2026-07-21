

## Measuring Performance in Production 生产环境性能测量

When an application runs in a shared infrastructure, e.g., in a public cloud, there usually will be workloads from other customers running on the same servers. With technologies like virtualization and containers becoming more popular, public cloud providers try to fully utilize the capacity of their servers. Unfortunately, it creates additional obstacles for measuring performance in such an environment. When your application shares resources with neighbor processes, its performance can become very unpredictable.

当应用程序运行在共享基础设施（例如：公有云）中时，通常会有其他客户的工作负载运行在同一服务器上。随着虚拟化和容器等技术的日益普及，公有云提供商力求充分利用其服务器容量。然而，这也给在这种环境下测量性能带来了额外的挑战。当应用程序与相邻进程共享资源时，其性能可能会变得非常难以预测。

Analyzing production workloads by recreating a specific scenario in a lab can be quite tricky. Sometimes it's not possible to reproduce exact behavior for "in-house" performance testing. This is why cloud providers and hyperscalers provide tools to monitor performance directly on production systems. A code change that performs well in a lab environment does not necessarily always perform well in production. Consult with your cloud service provider to see how you can enable performance monitoring of production instances. We provide an overview of continuous profilers in [@sec:ContinuousProfiling].

通过在实验室中重现特定场景来分析生产力工作负载可能相当棘手。有时，无法在“内部”性能测试中完全复现实际行为。因此，云提供商和超大规模云服务商会提供工具来直接监控生产系统的性能。在实验室环境中表现良好的代码更改，在生产环境中未必也能表现良好。请咨询您的云服务提供商，了解如何启用生产实例的性能监控。我们在 [@sec:ContinuousProfiling] 中提供了持续性能分析器的概述。

It's becoming a trend for large service providers to implement telemetry systems that monitor performance on user devices. One such example is the Netflix Icarus[^1] telemetry service, which runs on thousands of different devices spread around the world. Such a telemetry system helps Netflix understand how users perceive Netflix's app performance. It enables Netflix engineers to analyze data collected from many devices and to find issues that otherwise would be impossible to find. This kind of data enables making better-informed decisions on where to focus optimization efforts.

大型服务提供商部署遥测系统来监控用户设备的性能，正逐渐成为一种趋势。网飞Netflix的Icarus[^1]遥测服务就是一个例子，它运行在全球数千台不同的设备上。这种遥测系统帮助Netflix了解用户对Netflix应用性能的感知。它使Netflix工程师能够分析从众多设备收集的数据，并发现原本难以发现的问题。这类数据有助于他们更好地确定优化工作的重点方向。

One important caveat of monitoring production deployments is measurement overhead. Because any kind of monitoring affects the performance of a running service, we recommended using lightweight profiling methods. According to [@GoogleWideProfiling]: "To conduct continuous profiling on datacenter machines serving real traffic, extremely low overhead is paramount". Usually, acceptable aggregated overhead is considered below 1%. Performance monitoring overhead can be reduced by limiting the set of profiled machines as well as capturing data samples less frequently.

监控生产环境部署的一个重要注意事项是测量开销。由于任何类型的监控都会影响正在运行的服务的性能，我们建议使用轻量级的性能轮廓分析方法。根据[@GoogleWideProfiling]的说法：“要在处理真实流量的数据中心机器上进行持续的性能轮廓分析，极低的开销至关重要。”通常，可接受的总体开销低于1%。可以通过限制被分析的机器数量以及降低数据采样频率来减少性能监控开销。

[^1]: Presented at CMG 2019 在计算机测量组CMG2019会议上网飞工程师Martin Spier的演讲, [https://www.youtube.com/watch?v=4RG2DUK03_0](https://www.youtube.com/watch?v=4RG2DUK03_0).
