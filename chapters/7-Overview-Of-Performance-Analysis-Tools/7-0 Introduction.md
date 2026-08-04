# Overview of Performance Analysis Tools 性能分析工具概述 {#sec:secOverviewPerfTools}

In the previous chapter, we explored the features implemented in modern processors to aid performance analysis. However, if you were to start directly using those features, it would become very nuanced very quickly as it requires a lot of low-level programming to make use of them. Luckily, performance analysis tools take care of all the complexity that is required to effectively use these hardware performance monitoring features. This makes profiling go smoothly, but it's still critical to have an intuition of how such tools obtain and interpret the data. That is why we now discuss analysis tools only after we have discussed CPU performance monitoring features.

在上一章中，我们探讨了现代处理器中为辅助性能分析而实现的功能特性。然而，如果您直接使用这些功能特性，很快就会发现其中的复杂性，因为它们需要大量的底层编程才能发挥作用。幸运的是，性能分析工具可以处理有效使用这些硬件性能监控功能所需的所有复杂性。这使得性能分析过程更加顺畅，但了解这些工具如何获取和解释数据仍然至关重要。因此，我们只有在讨论完CPU性能监控功能之后才会讨论性能分析工具。

This chapter gives a quick overview of the most popular performance analysis tools available on major platforms. Some of the tools are cross-platform but the majority are not, so it is important to know what tools are available to you. Profiling tools are usually developed and maintained by hardware vendors themselves because they are the ones who know how to properly use performance monitoring features available on their processors. So, the choice of a tool for advanced performance engineering work depends on which operating system and CPU you're using.

本章将简要概述主流平台上最常用的性能分析工具。有些工具是跨平台的，但大多数不是，因此了解您可以使用的工具非常重要。性能分析工具通常由硬件供应商自行开发和维护，因为他们最了解如何正确使用其处理器上的性能监控功能。因此，选择用于高级性能工程的工具取决于您使用的操作系统和CPU。

After reading the chapter, take the time to practice using tools that you may eventually use. Familiarize yourself with the interface and workflow of those tools. Profile the application that you work with daily. Even if you don't find any actionable insights, you will be much better prepared when the actual need arises.

阅读完本章后，请花些时间练习使用您可能最终会用到的工具。熟悉这些工具的界面和工作流程。对您日常使用的应用程序进行性能分析。即使您没有发现任何可操作的见解，当真正需要时，您也会做好更充分的准备。