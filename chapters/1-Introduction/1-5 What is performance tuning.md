## What Is Performance Tuning? 什么是性能调优？

Locating a performance bottleneck is only half of an engineer’s job. The second half is to fix it properly. Sometimes changing one line in the source code of a program can yield a drastic performance boost. Missing such opportunities can be quite wasteful. Performance analysis and tuning are all about finding and fixing this line.

找到性能瓶颈只是工程师工作的一半，另一半在于如何正确地解决它。有时，修改程序源代码中的一行代码就能带来性能的显著提升。错过这样的机会会造成巨大的浪费。性能分析和调优的核心就在于找到并修复这行代码。

To take advantage of all the computing power of modern CPUs, you need to understand how they work. Or as performance engineers like to say, you need to have "mechanical sympathy". This term was borrowed from the car racing world. It means that a racing driver with a good understanding of how the car works has an edge over its competitors who don't. The same applies to performance engineering. It is not possible to know all the details of how a modern CPU operates, but you need to have a good mental model of it to squeeze the last bit of performance.

要充分利用现代CPU的计算能力，你需要了解它们的工作原理。或者正如性能工程师常说的，你需要具备“机械共鸣（mechanical sympathy）”。这个术语源于赛车领域。它的意思是，对赛车工作原理有深入了解的赛车手比不了解的竞争对手更有优势。这同样适用于性能工程。我们不可能了解现代CPU运行的所有细节，但你需要对其有一个清晰的理解，才能榨取最后一丝性能。

This is what I mean by *low-level optimizations*. This is a type of optimization that takes into account the details of the underlying hardware capabilities. It is different from *high-level optimizations* which are more about application-level logic, algorithms, and data structures. As you will see in the book, the majority of low-level optimizations can be applied to a wide variety of modern processors. To successfully implement low-level optimizations, you need to have a good understanding of the underlying hardware. 

这就是我所说的*底层优化*。这种优化方式会考虑到底层硬件能力的细节。它不同于*高级优化*，后者更多地关注应用层面的逻辑、算法和数据结构。正如您将在书中看到的，大多数底​​层优化都可以应用于各种现代处理器。要成功实现底层优化，您需要对底层硬件有深入的了解。

> "During the post-Moore era, it will become ever more important to make code run fast and, in particular, to tailor it to the hardware on which it runs." [@Leisersoneaam9744]

> “在后摩尔时代，让代码运行快速，尤其是裁剪使其适应运行的硬件，将变得越来越重要。” [@Leisersoneaam9744]

In the past, software developers had more mechanical sympathy, as they often had to deal with nuances of the hardware implementation. During the PC era, developers usually were programming directly on top of the operating system, with possibly a few libraries in between. As the world moved to the cloud era, the software stack grew deeper, broader, and more complex. The top layer of the stack (on which most developers work) has moved further away from the hardware. The negative side of such evolution is that developers of modern applications have less affinity for the actual hardware on which their software is running. This book will help you build a strong connection with modern processors.

过去，软件开发人员对有更多的机械共鸣，因为他们经常需要处理硬件实现的各种细节。在个人电脑时代，开发人员通常直接在操作系统之上进行编程，中间可能只需要一些库。随着世界迈向云计算时代，软件栈变得越来越深、越来越广、越来越复杂。栈的顶层（大多数开发人员的工作层）已经远离了硬件。这种演变的负面影响在于，现代应用程序的开发者对运行其软件的实际硬件的了解程度有所下降。本书将帮助你与现代处理器建立更紧密的联系。

There is a famous quote by Donald Knuth: "Premature optimization is the root of all evil".[@Knuth1974StructuredPW] But the opposite is often true as well. Postponed performance engineering may be too late and cause as much evil as premature optimization. For developers working with performance-critical projects, it is crucial to know how underlying hardware works. In such roles, program development without a hardware focus is a failure from the beginning.[^1] Performance characteristics of software must be a primary objective alongside correctness and security from day one. Poor performance can kill a product just as easily as security vulnerabilities.

图灵奖得主高德纳（Donald Knuth）曾说过一句名言：“过早优化是万恶之源”[@Knuth1974StructuredPW]。但反过来也常常如此。延迟进行性能工程可能为时已晚，其危害与过早优化不相上下。对于从事性能关键型项目的开发者而言，了解底层硬件的工作原理至关重要。在这些岗位上，缺乏硬件关注的程序开发从一开始就注定失败[^1]。从一开始，软件的性能特性就必须与正确性和安全性并列为首要目标。糟糕的性能与安全漏洞一样，都可能毁掉一个产品。

Performance engineering is important and rewarding work, but it may be very time-consuming. In fact, performance optimization is a game with no end. There will always be something to optimize. Inevitably, a developer will reach the point of diminishing returns at which further improvement is not justified by expected engineering costs. Knowing when to stop optimizing is a critical aspect of performance work.

性能工程是一项重要且回报丰厚的工作，但它可能非常耗时。事实上，性能优化是一个永无止境的过程。总会有需要优化的地方。开发者不可避免地会遇到收益递减点，此时进一步改进的收益将超过预期的工程成本。懂得何时停止优化是性能优化工作的关键所在。

[^1]: ClickHouse DB is an example of a successful software product that was built around a small but very efficient core. ClickHouse DB就是一个成功的软件产品案例，它围绕着一个精简但高效的核心构建而成。