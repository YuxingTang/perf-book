## What Is Not Discussed in this Book? 本书不讨论什么？

System performance depends on different components: CPU, DRAM, I/O, and network devices, etc. Applications may benefit from tuning various components of the system, depending on where a bottleneck is. In general, engineers should analyze the performance of the whole system. However, the biggest factor in a system's performance is its heart, the CPU. This is why this book primarily focuses on performance analysis from a CPU perspective. We also discuss the memory subsystem quite extensively, but we don't explore I/O and network performance.

系统性能取决于不同的组件：CPU、DRAM、I/O和网络设备等。应用程序可以通过调整系统的各个组件来提升性能，具体取决于瓶颈所在。通常，工程师应该分析整个系统的性能。然而，影响系统性能的最关键因素是其核心——CPU。因此，本书主要从CPU的角度进行性能分析。我们也会深入探讨内存子系统，但不会涉及I/O和网络性能。

Likewise, the software stack includes many layers, e.g., firmware, BIOS, OS, libraries, and the source code of an application. However, since most of the lower layers are not under our direct control, the primary focus will be on the source code level.

同样，软件栈包含许多层，例如固件、BIOS、操作系统、库以及应用程序的源代码。然而，由于大多数底层不受我们直接控制，因此本书将主要关注源代码层面。

The scope of the book does not go beyond a single CPU socket, so we will not discuss optimization techniques for distributed, NUMA, and heterogeneous systems. Offloading computations to accelerators (GPU, FPGA, etc.) using solutions like OpenCL and openMP is not discussed in this book. 

本书的范围仅限于单个CPU插槽，因此我们不会讨论分布式系统、NUMA系统和异构系统的优化技术。本书也不讨论使用OpenCL和OpenMP等解决方案将计算任务卸载到加速器（GPU、FPGA等）的方法。

I tried to make this book to be applicable to most modern CPUs, including Intel, AMD, Apple, and other ARM-based processors. I'm sorry if it doesn't cover your favorite architecture. Nevertheless, many of the principles discussed in this book apply well to other processors. Similarly, most examples in this book were run on Linux, but again, most of the time it doesn't matter since the same techniques benefit applications that run on Windows and macOS operating systems.

我力求使本书适用于大多数现代CPU，包括：Intel、AMD、Apple和其他基于ARM指令集的处理器。如果本书未能涵盖您常用的体系结构，敬请谅解。不过，本书讨论的许多原理同样适用于其他处理器。此外，本书中的大多数示例都是在Linux系统上运行的，但通常情况下，这并不重要，因为相同的技术同样适用于在Windows和macOS操作系统上运行的应用程序。

Code snippets in this book are written in C or C++, but to a large degree, ideas from this book can be applied to other languages that are compiled to native code like Rust, Go, and even Fortran. Since this book targets user-mode applications that run close to the hardware, we will not discuss managed environments, e.g., Java. 

本书中的代码片段是用C或C++编写的，但本书中的许多理念也适用于其他编译成本地代码的语言，例如：Rust、Go甚至Fortran。由于本书面向的是接近硬件运行的用户模式应用程序，因此我们不会讨论托管环境，例如：Java。

Finally, I assume that readers have full control over the software that they develop, including the choice of libraries and compilers they use. Hence, this book is not about tuning purchased commercial packages, e.g., tuning SQL database queries.

最后，我假设读者可以完全控制他们开发的软件，包括选择他们使用的库和编译器。因此，本书并非关于如何优化购买的商业软件包，例如：如何优化SQL数据库查询。
