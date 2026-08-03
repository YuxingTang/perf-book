## The Roofline Performance Model 屋顶线Roofline性能模型 {#sec:roofline}

The Roofline performance model is a throughput-oriented performance model that is heavily used in the HPC world. It was developed at the University of California, Berkeley, in 2009 [@RooflinePaper]. The term "roofline" in this model expresses the fact that the performance of an application cannot exceed the capabilities of a machine. Every function and every loop in a program is limited by either the computing or memory bandwidth capacity of a machine. This concept is represented in Figure @fig:RooflineIntro. The performance of an application will always be limited by a certain "roofline" function.

屋顶线Roofline性能模型是一种面向吞吐量的性能模型，广泛应用于高性能计算(HPC: High Performance Computing)领域。它由加州大学伯克利分校于2009年提出并开发 [@RooflinePaper]。该模型中的“屋顶线roofline”一词表达了应用程序的性能不能超过机器的处理能力这一事实。程序中的每个函数和每个循环都受到机器计算能力或内存带宽的限制。这一概念如图 @fig:RooflineIntro 所示。应用程序的性能始终受限于某个特定的“roofline”函数。

![The Roofline Performance Model. The maximum performance of an application is limited by the minimum between peak FLOPS (horizontal line) and the platform bandwidth multiplied by arithmetic intensity (diagonal line). Roofline 性能模型。应用程序的最大性能受限于峰值浮点运算速度（水平线）与平台带宽乘以运算强度（对角线）之间的最小值.](../../img/perf-analysis/Roofline-intro.png){#fig:RooflineIntro width=80%}

Hardware has two main limitations: how fast it can make calculations (peak compute performance, FLOPS) and how fast it can move the data (peak memory bandwidth, GB/s). The maximum performance of an application is limited by the minimum between peak FLOPS (horizontal line) and the platform bandwidth multiplied by arithmetic intensity (diagonal line). The roofline chart in Figure @fig:RooflineIntro plots the performance of two applications `A` and `B` against hardware limitations. Application `A` has lower arithmetic intensity and its performance is bound by the memory bandwidth, while application `B` is more compute intensive and doesn't suffer as much from memory bottlenecks. Similar to this, `A` and `B` could represent two different functions within a program and have different performance characteristics. The Roofline performance model accounts for that and can display multiple functions and loops of an application on the same chart. However, keep in mind that the Roofline performance model is mainly applicable for HPC applications that have few compute-intensive loops. I do not recommend using it for general-purpose applications, such as compilers, web browsers, or databases.

硬件有两个主要限制：计算速度（峰值计算性能，FLOPS）和数据传输速度（峰值内存带宽，GB/s）。应用程序的最大性能受限于峰值浮点运算速度（水平线）与平台带宽乘以运算强度（对角线）之间的最小值。图 @fig:RooflineIntro 中的屋顶线图绘制了两个应用程序“A”和“B”的性能与硬件限制的关系。应用程序A的运算强度较低，其性能受限于内存带宽；而应用程序B的计算强度较高，受内存瓶颈的影响较小。类似地，A和B可以代表程序中的两个不同功能，并具有不同的性能特征。Roofline性能模型考虑到了这一点，可以在同一张图表中显示应用程序的多个功能和循环。但是，请记住，Roofline性能模型主要适用于由少数计算密集型循环构成的高性能计算(HPC)应用程序。我不建议将其用于通用应用程序，例如：编译器、Web浏览器或数据库。

*Arithmetic Intensity* is a ratio between Floating-point operations (FLOPs)[^7] and bytes, and it can be calculated for every loop in a program. Let's calculate the arithmetic intensity of code in [@lst:BasicMatMul]. In the innermost loop body, we have a floating-point addition and a multiplication; thus, we have 2 FLOPs. Also, we have three read operations and one write operation; thus, we transfer `4 operations * 4 bytes = 16` bytes. The arithmetic intensity of that code is `2 / 16 = 0.125`. Arithmetic intensity is the X-axis on the Roofline chart, while the Y-axis measures the performance of a given program.

*算术强度*是浮点运算次数(FLOPs: FLoating-point OPerations)[^7]与字节数的比值，可以计算程序中每个循环的运算强度。让我们计算一下 [@lst:BasicMatMul] 中代码的运算强度。在最内层的循环体中，我们有一个浮点加法和一个浮点乘法；因此，我们有2次浮点运算。此外，我们有三次读取操作和一次写入操作；因此，我们传输了 `4次操作 * 4字节 = 16` 字节。该代码的算术强度为 `2 / 16 = 0.125`。算术强度是Roofline图表的X轴，而Y轴衡量给定程序的性能。

Listing: Naive parallel matrix multiplication.
代码列表：天真的并行矩阵乘

~~~~ {#lst:BasicMatMul .cpp .numberLines}
void matmul(int N, float a[][2048], float b[][2048], float c[][2048]) {
    #pragma omp parallel for
    for(int i = 0; i < N; i++) {
        for(int j = 0; j < N; j++) {
            for(int k = 0; k < N; k++) {
                c[i][j] = c[i][j] + a[i][k] * b[k][j];
            }
        }
    }
}
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Traditional ways to speed up an application's performance is to fully utilize the SIMD and multicore capabilities of a machine. Usually, we need to optimize for many aspects: vectorization, memory, and threading. Roofline methodology can assist in assessing these characteristics of your application. On a roofline chart, we can plot theoretical maximums for scalar single-core, SIMD single-core, and SIMD multicore performance (see Figure @fig:RooflineIntro2). This will give us an understanding of the scope for improving the performance of an application. If we find that our application is compute-bound (i.e., has high arithmetic intensity) and is below the peak scalar single-core performance, we should consider forcing vectorization (see [@sec:Vectorization]) and distributing the work among multiple threads. Conversely, if an application has low arithmetic intensity, we should seek ways to improve memory accesses (see [@sec:MemBound]). The ultimate goal of optimizing performance using the Roofline model is to move the points up on the chart. Vectorization and threading move the dot up while optimizing memory accesses by increasing arithmetic intensity will move the dot to the right, and also likely improve performance.

提升应用程序性能的传统方法是充分利用机器的SIMD和多核能力。通常，我们需要优化多个方面：向量化、内存和线程。屋顶线Roofline方法可以帮助评估应用程序的这些特性。在屋顶线Roofline图上，我们可以绘制标量单核、SIMD单核和SIMD多核性能的理论最大值（参见图 @fig:RooflineIntro2）。这将帮助我们了解应用程序性能的提升空间。如果我们发现应用程序受限于计算能力（即：算术强度高），并且低于标量单核性能的峰值，则应考虑强制向量化（参见 [@sec:Vectorization]）并且将工作分配到多个线程。相反，如果应用程序的算术强度低，则应寻求优化内存访问的方法（参见 [@sec:MemBound]）。使用屋顶线Roofline模型优化性能的最终目标是将图表上的点向上移动。向量化和多线程可以向上移动点，而通过提高运算强度来优化内存访问则可以向右移动点，并且很可能也会提高性能。

![Roofline analysis of a program and potential ways to improve its performance. 程序的 Roofline 分析及其潜在性能提升方法。](../../img/perf-analysis/Roofline-intro2.jpg){#fig:RooflineIntro2 width=70%}

Theoretical maximums (rooflines) are often presented in a device specification and can be easily looked up. Also, theoretical maximums can be calculated based on the characteristics of the machine you are using. Usually, it is not hard to do once you know the parameters of your machine. For the Intel Core i5-8259U processor, the maximum number of FLOPS (single-precision floats) with AVX2 and 2 Fused Multiply Add (FMA) units can be calculated as:
理论最大值（Roofline）通常会在设备规格中给出，并且可以轻松查找。此外，还可以根据您使用的机器的特性计算理论最大值。通常，一旦您了解机器的参数，计算起来并不难。对于Intel Core i5-8259U处理器，在AVX2指令集和2个融合乘加(FMA: Fused Multiply Add)单元的情况下，其最大单精度浮点运算性能(FLOPS)可计算如下：
$$
\begin{aligned}
\textrm{Peak FLOPS} =& \textrm{ 8 (number of logical cores)}~\times~\frac{\textrm{256 (AVX bit width)}}{\textrm{32 bit (size of float)}} ~ \times ~ \\
& \textrm{ 2 (FMA)} \times ~ \textrm{3.8 GHz (Max Turbo Frequency)} \\
& = \textrm{486.4 GFLOPS}
\end{aligned}
$$
$$
\begin{aligned}
\textrm{峰值FLOPS} =& \textrm{ 8 (逻辑核心数)}~\times~\frac{\textrm{256 (AVX位宽)}}{\textrm{32位(浮点数大小)}} ~ \times ~ \\
& \textrm{ 2 (FMA)} \times ~ \textrm{3.8 GHz (最大睿频频率)} \\
& = \textrm{486.4 GFLOPS}
\end{aligned}
$$

The maximum memory bandwidth of Intel NUC Kit NUC8i5BEH, which I used for experiments, can be calculated as shown below. Remember, that DDR technology allows transfers of 64 bits or 8 bytes per memory access.

我用于实验的Intel NUC套件NUC8i5BEH的最大内存带宽计算如下。请记住，DDR技术允许每次内存访问传输64位或8字节的数据。

$$
\begin{aligned}
\textrm{Peak Memory Bandwidth} = &~\textrm{2400 (memory transfer rate)}~\times~ \textrm{2 (memory channels)} ~ \times \\ &~ \textrm{8 (bytes per memory access)} ~ \times \textrm{1 (socket)}= \textrm{38.4 GiB/s}
\end{aligned}
$$
$$
\begin{aligned}
\textrm{峰值内存带宽} = &~\textrm{2400 (内存传输速率)}~\times~ \textrm{2 (内存通道数)} ~ \times \\ &~ \textrm{8 (每次内存访问的字节数)} ~ \times \textrm{1 (处理器插槽数)}= \textrm{38.4 GiB/s}
\end{aligned}
$$

Automated tools like [Empirical Roofline Tool](https://bitbucket.org/berkeleylab/cs-roofline-toolkit/src/master/)[^2] and [Intel Advisor](https://software.intel.com/content/www/us/en/develop/tools/advisor.html)[^3] are capable of empirically determining theoretical maximums by running a set of prepared benchmarks. If a calculation can reuse the data in the cache, much higher FLOP rates are possible. Roofline can account for that by introducing a dedicated roofline for each level of the memory hierarchy (see Figure @fig:RooflineMatrix).

诸如[Empirical Roofline Tool](https://bitbucket.org/berkeleylab/cs-roofline-toolkit/src/master/)[^2]和[Intel Advisor](https://software.intel.com/content/www/us/en/develop/tools/advisor.html)[^3]之类的自动化工具能够通过运行一组预先准备好的基准测试，以经验方式确定理论最大值。如果计算能够重用缓存中的数据，则可以实现更高的浮点运算速率(FLOP)。屋顶线Roofline通过为存储层次结构的每一层引入专用的屋顶线Roofline来解决这个问题（参见图 @fig:RooflineMatrix）。

After hardware limitations are determined, we can start assessing the performance of an application against the roofline. Intel Advisor automatically builds a Roofline chart and provides hints for performance optimization of a given loop. An example of a Roofline chart generated by Intel Advisor is presented in Figure @fig:RooflineMatrix. Notice, that Roofline charts have logarithmic scales.

确定硬件限制后，我们可以开始根据屋顶线Roofline判定应用程序的性能。Intel Advisor会自动生成Roofline图表，并为给定循环的性能优化提供建议。图 @fig:RooflineMatrix 展示了Intel Advisor生成的 Roofline 图表示例。请注意，Roofline图表采用对数刻度。

![Roofline analysis for "before" and "after" versions of matrix multiplication on Intel NUC Kit NUC8i5BEH with 8GB RAM using the Clang 10 compiler. 使用Clang 10编译器，在配备8GB内存的Intel NUC套件NUC8i5BEH上对矩阵乘法进行“优化前”和“优化后”的屋顶线分析。](../../img/perf-analysis/roofline_matrix.png){#fig:RooflineMatrix width=90%}

Roofline methodology enables tracking optimization progress by plotting "before" and "after" points on the same chart. So, it is an iterative process that guides developers to help their applications fully utilize hardware capabilities. Figure @fig:RooflineMatrix shows performance gains from making the following two changes to the code shown earlier in [@lst:BasicMatMul]:
屋顶线方法通过在同一张图表上绘制“优化前”和“优化后”的数据点来跟踪优化进度。因此，这是一个迭代过程，可以指导开发人员帮助他们的应用程序充分利用硬件性能。图 @fig:RooflineMatrix 显示了对前面 [@lst:BasicMatMul] 中的代码进行以下两项更改后的性能提升：

* Interchange the two innermost loops (swap lines 4 and 5). This enables cache-friendly memory accesses (see [@sec:MemBound]).
* Enable autovectorization of the innermost loop using AVX2 instructions.

* 交换两个最内层循环（交换第4行和第5行）。这可以实现对缓存友好的存储访问（参见 [@sec:MemBound]）。
* 使用AVX2指令启用最内层循环的自动向量化。

In summary, the Roofline performance model can help to:
总而言之，屋顶线Roofline性能模型可以帮助我们：

* Identify performance bottlenecks.
* Guide software optimizations.
* Determine when we’re done optimizing.
* Assess performance relative to machine capabilities.

* 识别性能瓶颈。
* 指导软件优化。
* 确定何时完成优化。
* 评估相对于机器性能的性能。

[^2]: Empirical Roofline Tool - [https://bitbucket.org/berkeleylab/cs-roofline-toolkit/src/master/](https://bitbucket.org/berkeleylab/cs-roofline-toolkit/src/master/).
[^3]: Intel Advisor - [https://software.intel.com/content/www/us/en/develop/tools/advisor.html](https://software.intel.com/content/www/us/en/develop/tools/advisor.html).
[^7]: The Roofline performance model is not only applicable to floating-point calculations but can be also used for integer operations. However, the majority of HPC applications involve floating-point calculations, thus the Roofline model is mostly used with FLOPs. 屋顶线Roofline性能模型不仅适用于浮点运算，也可用于整数运算。然而，大多数高性能计算应用都涉及浮点运算，因此Roofline模型主要用于FLOPs单位。
