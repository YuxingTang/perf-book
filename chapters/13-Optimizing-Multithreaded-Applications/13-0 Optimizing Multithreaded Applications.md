# Optimizing Multithreaded Applications 优化多线程应用程序 {#sec:secOptMTApps}

Modern CPUs are getting more and more cores each year. As of 2024, you can buy a server processor which will have more than 200 cores! And even a laptop with 16 execution threads is a pretty usual setup nowadays. Since there is so much processing power in every CPU, effective utilization of all the hardware threads becomes more challenging. Preparing software to scale well with a growing amount of CPU cores is very important for the future success of your application.

现代CPU的核心数量逐年增加。到2024年，您就能买到拥有超过200个核心的服务器级处理器！如今，即使是配备16个执行线程的笔记本电脑也相当常见。由于每个CPU的处理能力如此强大，如何有效利用所有硬件线程就成了一项更大的挑战。让软件能够随着CPU核心数量的增长而良好扩展，对于应用程序未来的成功至关重要。

There is a difference in how server and client products exploit parallelism. Most server platforms are designed to process requests from a large number of customers. Those requests are usually independent of each other, so the server can process them in parallel. If there is enough load on the system, applications themselves could be single-threaded, and still platform utilization will be high. However, when you use your server platform for HPC or AI computations; then you need all the computing power you have. On the other hand, client platforms, such as laptops and desktops, have all the resources to serve a single user. In this case, an application has to make use of all the available cores to provide the best user experience. In this chapter, we will focus on applications that can scale to a large number of cores.

服务器和客户端产品利用并行性的方式有所不同。大多数服务器平台旨在处理来自大量客户的请求。这些请求通常彼此独立，因此服务器可以并行处理它们。如果系统负载足够大，应用程序本身可以是单线程的，平台利用率仍然会很高。但是，当你将服务器平台用于高性能计算(HPC)或人工智能(AI)计算时，则需要充分利用所有计算能力。另一方面，笔记本电脑和台式机等客户端平台拥有服务单个用户所需的所有资源。在这种情况下，应用程序必须利用所有可用核心才能提供最佳用户体验。本章将重点讨论可扩展到大量核心的应用程序。

From the software perspective, there are two primary ways to achieve parallelism: multiprocessing and multithreading. In a multiprocess application, multiple independent processes run concurrently. Each process has its own memory space and communicates with other processes through inter-process communication mechanisms such as pipes, sockets, or shared memory. In a multithreaded application, a single process contains multiple threads, which share the same memory space and resources of the process. Threads within the same process can communicate and share data more easily because they have direct access to the same memory space. However, synchronization between threads is usually more complex and is prone to issues like race conditions and deadlocks. In this chapter we will mostly focus on multithreaded applications, however, some techniques can be applied to multiprocess applications as well. We will show examples of both types of applications in this chapter.

从软件角度来看，实现并行性主要有两种方式：多进程和多线程。在多进程应用程序中，多个独立的进程并发运行。每个进程都有自己的内存空间，并通过进程间通信机制（例如：管道、套接字或共享内存）与其他进程通信。在多线程应用程序中，单个进程包含多个线程，这些线程共享进程的同一个内存空间和资源。同一进程内的线程可以更轻松地通信和共享数据，因为它们可以直接访问同一内存空间。然而，线程间的同步通常更复杂，并且容易出现竞态条件和死锁等问题。本章将主要关注多线程应用程序，但某些技术也适用于多进程应用程序。本章将展示两种类型的应用程序示例。

When talking about throughput-oriented applications, we can distinguish the following two types of applications:

当谈到面向吞吐率的应用程序，我们可以区分以下两种类型：

* **Massively parallel applications**. Such applications usually scale well with the number of cores. They are designed to process a large number of independent tasks. Massively parallel programs often use the divide-and-conquer technique to split the work into smaller tasks (also called *worker threads*) and process them in parallel. Examples of such applications are scientific computations, video rendering, data analytics, AI, and many others. The main obstacle for such applications is the saturation of a shared resource, such as memory bandwidth, that can effectively stall all the worker threads in the process.
* **Applications that require synchronization**. Such applications have workers share resources to complete their tasks. Worker threads depend on each other, which creates periods when some threads are stalled. Examples of such applications are databases, web servers, and other server applications. The main challenge for such applications is to minimize required synchronization and to avoid contention on shared resources.

* **大规模并行应用程序**。这类应用程序通常能够很好地利用核心数量进行扩展。它们旨在处理大量独立任务。大规模并行程序通常采用分而治之的技术，将工作拆分成更小的任务（也称为*工作线程（worker threads）*）同时并行处理。此类应用程序的示例包括：科学计算、视频渲染、数据分析、人工智能和其他等等。这类应用程序的主要障碍是共享资源（例如：内存带宽）的饱和，这会导致进程中的所有工作线程都出现阻塞。
* **需要同步的应用程序**。这类应用程序的工作线程共享资源来完成任务。工作线程之间相互依赖，这会导致某些线程出现阻塞。此类应用程序的示例包括：数据库、We 服务器和其他服务器应用程序。这类应用程序的主要挑战是最大限度地减少所需的同步操作，并避免对共享资源的争用。

In this chapter, we will explore how to analyze the performance of both types of applications. Since this is a book about low-level performance, we will not discuss algorithm-level optimizations such as lock-free data structures, which are well covered in other books.

本章我们将探讨如何分析这两类应用程序的性能。由于本书侧重于底层性能优化，因此不会讨论算法层面的优化，例如：无锁数据结构，这些内容在其他书籍中已有详细介绍。