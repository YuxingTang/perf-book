# Machine Code Layout Optimizations 机器代码布局优化 {#sec:secFEOpt}

The CPU Frontend (FE) is responsible for fetching and decoding instructions and delivering them to the out-of-order Backend (BE). As the newer processors get more execution "horsepower", the CPU FE needs to be as powerful to keep the machine balanced. If the FE cannot keep up with supplying instructions, the BE will be underutilized, and the overall performance will suffer. That's why the FE is designed to always run well ahead of the actual execution to smooth out any hiccups that may occur and always have instructions ready to be executed. For example, Intel Skylake, released in 2016, can fetch up to 16 instructions per cycle.

CPU前端(FE: FrontEnd)负责获取和译码指令，并将其传递给乱序执行的后端(BE: BackEnd)。随着新型处理器执行能力的提升，CPU前端也需要具备相应的性能，以保持机器的平衡。如果前端无法及时提供指令，后端将无法被充分利用，从而影响整体性能。因此，前端的设计使其始终运行在实际执行之前，以平滑可能出现的任何延迟，并始终准备好等待执行的指令。例如，2016年发布的Intel Skylake处理器每个周期最多可以获取16条指令。

Most of the time, inefficiencies in the CPU FE can be described as a situation when the Backend is waiting for instructions to execute, but the Frontend is not able to provide them. As a result, CPU cycles are wasted without doing any actual useful work. Recall that modern CPUs can process multiple instructions every cycle, nowadays ranging from 4- to 9-wide. Situations when not all available slots are filled happen very often. This represents a source of inefficiency for applications in many domains, such as databases, compilers, web browsers, and many others.

大多数情况下，CPU前端的效率低下表现为后端正在等待指令执行，但前端却无法提供。结果，CPU周期被浪费，没有执行任何实际有用的工作。请记住，现代CPU每个周期可以处理多条指令，如今通常可以处理4到9条指令宽度。并非所有可用指令槽都被填满的情况非常常见。这会导致许多领域的应用程序效率低下，例如：数据库、编译器、Web浏览器等等。

The TMA methodology captures FE performance issues in the `Frontend Bound` metric. It represents the percentage of cycles when the CPU FE is not able to deliver instructions to the BE, while it could have accepted them. Most of the real-world applications experience a non-zero 'Frontend Bound' metric, meaning that some percentage of running time will be lost on suboptimal instruction fetching and decoding. Below 10\% is the norm. If you see the "Frontend Bound" metric being more than 20\%, it's worth spending time on it.

TMA方法论通过 `前端瓶颈Frontend Bound` 指标来捕捉前端性能问题。该指标表示CPU前端无法将指令传递给后端，而前端本可以接收这些指令的周期百分比。大多数实际应用程序都会遇到非零的“前端瓶颈”指标，这意味着一部分运行时间会因指令获取和译码效率低下而损失。低于10%是正常情况。如果“前端瓶颈”指标超过20%，则值得花时间进行分析。

There could be many reasons why FE cannot deliver instructions to the execution units. Most of the time, it is due to suboptimal code layout, which leads to poor I-cache and ITLB utilization. Applications with a large codebase, e.g., millions of lines of code, are especially vulnerable to FE performance issues. In this chapter, we will take a look at some typical optimizations to improve machine code layout.

前端无法将指令传递给执行单元的原因可能有很多。大多数情况下，性能问题是由于代码布局不合理造成的，这会导致指令缓存(I-cache)和指令地址转换查找缓冲器(ITLB)的利用率低下。代码库庞大的应用程序（例如：数百万行代码）尤其容易受到前端性能问题的影响。本章将介绍一些典型的优化方法来改进机器代码布局。

## Machine Code Layout 机器代码布局

When a compiler translates source code into machine code, it generates a linear byte sequence. [@lst:MachineCodeLayout] shows an example of a binary layout for a small snippet of C++ code. Once the compiler finishes generating assembly instructions, it needs to encode them and lay them out in memory sequentially.

编译器将源代码翻译成机器代码时，会生成一个线性字节序列。[@lst:MachineCodeLayout] 展示了一小段C++代码的二进制布局示例。编译器生成汇编指令后，需要对其进行编码，并按顺序将其排列在存储中。

Listing: Example of machine code layout
代码列表：机器代码布局的示例

~~~~ {#lst:MachineCodeLayout .cpp}
  C++ Code      │    Assembly Listing     │    Disassembled Machine Code
  ........      │    ................     │    ......................... 
if (a <= b)     │     ; a is in edi       │    401125 cmp esi, edi
  bar();        │     ; b is in esi       │    401128 jb 401131
else            │     cmp esi, edi        │    40112a call bar
  baz();        │     jb .label1          │    40112f jmp 401136
                │     call bar()          │    401131 call baz
                │     jmp .label2         │    401136 ...
                │   .label1:              │
                │     call baz()          │
                │   .label2:              │
                │     ...                 │
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The way code is placed in an object file is called *machine code layout*. Note that for the same program, it's possible to lay out the code in many different ways. For the code in [@lst:MachineCodeLayout], a compiler may decide to reverse the branch in such a way that a call to `baz` will come first. Also, bodies of the functions `bar` and `baz` can be placed in two different orders: we can place `bar` first in the executable image and then `baz` or reverse the order. This affects offsets at which instructions will be placed in memory, which in turn may affect the performance of the generated program as you will see later. In the following sections of this chapter, we will take a look at some typical optimizations for the machine code layout.

代码在目标文件中的排列方式称为*机器代码布局*。请注意，对于同一个程序，代码可以采用多种不同的布局方式。例如，对于[@lst:MachineCodeLayout]中的代码，编译器可能会决定反转分支，使对 `baz` 的调用优先执行。此外，函数 `bar` 和 `baz` 的函数体可以以两种不同的顺序排列：我们可以先将 `bar` 放在可执行文件中，然后再放置 `baz` ，或者反过来。这会影响指令在内存中的偏移量，进而影响生成程序的性能，这一点您将在后面看到。在本章的后续章节中，我们将探讨一些典型的机器代码布局优化方法。
