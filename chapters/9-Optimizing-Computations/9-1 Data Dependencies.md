## Data Dependencies 数据依赖

When a program statement refers to the output of a preceding statement, we say that there is a *data dependency* between the two statements. Sometimes people also use the terms _dependency chain_ or *data flow dependencies*. The example we are most familiar with is traversing a linked list (see Figure @fig:LinkedListChasing). To access node `N+1`, we should first dereference the pointer `N->next`. For the loop on the right, this is a *recurrent* data dependency, meaning it spans multiple iterations of the loop. Traversing a linked list is one very long dependency chain.

当一个程序语句引用前一个语句的输出时，我们称这两个语句之间存在*数据依赖*。有时人们也会使用术语*依赖链（dependency chain）*或*数据流依赖（data flow dependencies）*。我们最熟悉的例子是遍历链表（参见图 @fig:LinkedListChasing）。要访问节点 `N+1`，我们首先需要解引用指针 `N->next`。对于右侧的循环，这是一个*循环*数据依赖，意味着它跨越了循环的多次迭代。遍历链表就是一个非常长的依赖链。

![Data dependency while traversing a linked list.遍历链表时的数据依赖。](../../img/computation-opts/LinkedListChasing.png){#fig:LinkedListChasing width=80%}

Conventional programs are written assuming the sequential execution model. Under this model, instructions execute one after the other, atomically and in the order specified by the program. However, as we already know, this is not how modern CPUs are built. They are designed to execute instructions out-of-order, in parallel, and in a way that maximizes the utilization of the available execution units.

传统的程序编写基于顺序执行模型。在这种模型下，指令会按照程序指定的顺序逐条原子地执行。然而，我们都知道，现代CPU的设计并非如此。它们的设计目标是以乱序并行的方式执行指令，并最大限度地利用可用执行单元。

When long data dependencies do come up, processors are forced to execute code sequentially, utilizing only a part of their full capabilities. Long dependency chains hinder parallelism, which defeats the main advantage of modern superscalar CPUs. For example, pointer chasing doesn't benefit from OOO execution and thus will run at the speed of an in-order CPU. As we will see in this section, dependency chains are a major source of performance bottlenecks.

当出现较长的数据依赖关系时，处理器将被迫顺序执行代码，只能发挥其全部性能的一部分。过长的依赖链会阻碍并行性，从而抵消了现代超标量CPU的主要优势。例如，指针追踪无法从乱序执行中获益，因此其运行速度与顺序CPU相当。正如我们将在本节中看到的，依赖链是性能瓶颈的主要来源。

You cannot eliminate data dependencies; they are a fundamental property of programs. Any program takes an input to compute something. In fact, people have developed techniques to discover data dependencies among statements and build data flow graphs. This is called *dependence analysis* and is more appropriate for compiler developers, rather than performance engineers. We are not interested in building data flow graphs for the whole program. Instead, we want to find a critical dependency chain in a hot piece of code, such as a loop or function.

数据依赖关系无法消除；它们是程序的基本属性。任何程序都需要输入数据来进行计算。事实上，人们已经开发出一些技术来发现语句之间的数据依赖关系并构建数据流图。这被称为*依赖性分析（dependence analysis）*，更适合编译器开发人员，而非性能工程师。我们并不打算为整个程序构建数据流图。相反，我们希望在一段热门代码（例如：循环或函数）中找到关键的依赖链。

You may wonder: "If you cannot get rid of dependency chains, what *can* you do?" Well, sometimes this will be a limiting factor for performance, and unfortunately, you will have to live with it. But there are cases where you can break unnecessary data dependency chains or overlap their execution. One such example is shown in [@lst:DepChain]. Similar to a few other cases, we present the source code on the left along with the corresponding ARM assembly on the right. Also, this code example is included in the `dep_chains_2` lab assignment of the Performance Ninja online course, so you can try it yourself.[^2]

你可能会问：“如果无法消除依赖链，那*还能*做什么呢？” 嗯，有时依赖链会成为性能的限制因素，很遗憾，你不得不接受这一点。但有些情况下，你可以打破不必要的依赖链，或者让它们的执行重叠。[@lst:DepChain] 中就展示了这样一个例子。与其他一些例子类似，我们在左侧展示了源代码，并在右侧展示了相应的ARM汇编代码。此外，这个代码示例包含在“性能忍者（Performance Ninja）”在线课程的 `dep_chains_2` 实验作业中，你可以自己尝试一下[^2]。

This small program simulates random particle movement. We have 1000 particles moving on a 2D surface without constraints, which means they can go as far from their starting position as they want. Each particle is defined by its x and y coordinates on a 2D surface and speed. The initial x and y coordinates are in the range [-1000,1000] and the speed is in the range [0,1], which doesn't change. The program simulates 1000 movement steps for each particle. For each step, we use a random number generator (RNG) to produce an angle, which sets the movement direction for a particle. Then we adjust the coordinates of a particle accordingly.

这个小程序模拟了随机粒子运动。我们有100 个粒子在一个二维平面上无约束地运动，这意味着它们可以偏离起始位置任意远。每个粒子都由其在二维平面上的x和y坐标以及速度定义。初始x和y坐标在 [-1000,1000] 范围内，速度在[0,1]范围内，且保持不变。程序模拟每个粒子1000个运动步骤。对于每个步骤，我们使用随机数生成器(RNG)生成一个角度，该角度决定粒子的运动方向。然后，我们相应地调整粒子的坐标。

Given the task at hand, you decide to roll your own RNG, sine, and cosine functions to sacrifice some accuracy and make it as fast as possible. After all, this is *random* movement, so it is a good trade-off to make. You choose a medium-quality `XorShift` RNG as it only has 3 shifts and 3 XORs inside. What can be simpler? Also, you searched the web and found algorithms for sine and cosine approximation using polynomials, which are accurate enough and very fast.

鉴于当前任务，您决定自行编写RNG、正弦函数和余弦函数，以牺牲一些精度来尽可能提高速度。毕竟，这是*随机*运动，因此这是一个合理的权衡。您选择了一个中等精度的 `XorShift` 作为RNG，因为它内部只有3个移位和3个异或运算。还有什么比这更简单呢？此外，您还在网上搜索到了使用多项式逼近正弦和余弦的算法，这些算法精度足够高，速度也非常快。

Listing: Random Particle Motion on a 2D Surface
代码列表：在一个二维表面上的随机粒子运动

~~~~ {#lst:DepChain .cpp .numberLines}
struct Particle {                                    │
  float x; float y; float velocity;                  │
};                                                   │
                                                     │
class XorShift32 {                                   │
  uint32_t val;                                      │
public:                                              │
  XorShift32 (uint32_t seed) : val(seed) {}          │
  uint32_t gen() {                                   │
    val ^= (val << 13);                              │
    val ^= (val >> 17);                              │
    val ^= (val << 5);                               │
    return val;                                      │ .loop:
  }                                                  │   eor    w0, w0, w0, lsl #13
};                                                   │   eor    w0, w0, w0, lsr #17
                                                     │   eor    w0, w0, w0, lsl #5
static float sine(float x) {                         │   ucvtf  s1, w0
  const float B = 4 / PI_F;                          │   fmov   s2, w9
  const float C = -4 / ( PI_F * PI_F);               │   fmul   s2, s1, s2
  return B * x + C * x * std::abs(x);                │   fmov   s3, w10
}                                                    │   fadd   s3, s2, s3
static float cosine(float x) {                       │   fmov   s4, w11
  return sine(x + (PI_F / 2));                       │   fmul   s5, s3, s3
}                                                    │   fmov   s6, w12
                                                     │   fmul   s5, s5, s6
/* Map degrees [0;UINT32_MAX) to radians [0;2*pi)*/  │   fmadd  s3, s3, s4, s5
float DEGREE_TO_RADIAN = (2 * PI_D) / UINT32_MAX;    │   ldp    s6, s4, [x1, #0x4]
                                                     │   ldr    s5, [x1]
void particleMotion(vector<Particle> &particles,     │   fmadd  s3, s3, s4, s5
                    uint32_t seed) {                 │   fmov   s5, w13
 XorShift32 rng(seed);                               │   fmul   s5, s1, s5
 for (int i = 0; i < STEPS; i++)                     │   fmul   s2, s5, s2
  for (auto &p : particles) {                        │   fmadd  s1, s1, s0, s2
   uint32_t angle = rng.gen();                       │   fmadd  s1, s1, s4, s6
   float angle_rad = angle * DEGREE_TO_RADIAN;       │   stp    s3, s1, [x1], #0xc
   p.x += cosine(angle_rad) * p.velocity;            │   cmp    x1, x16
   p.y += sine(angle_rad) * p.velocity;              │   b.ne   .loop
  }                                                  │
}                                                    │
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

I compiled the code using the Clang-17 C++ compiler and ran it on a Mac mini (Apple M1, 2020). Let us examine the generated ARM assembly code:

我使用Clang-17 C++编译器编译了代码，并在Mac mini（Apple M1，2020）上运行了它。让我们来看一下生成的ARM汇编代码：

* The first three `eor` (exclusive OR) instructions combined with `lsl` (shift left) or `lsr` (shift right) correspond to the `XorShift32::gen` function.
* Next `ucvtf` (convert unsigned integer to floating-point) and `fmul` (floating-point multiply) are used to convert the angle from degrees to radians (line 35 in the code).
* Sine and Cosine functions both have two `fmul` instructions and one `fmadd` (floating-point fused multiply-add) instruction. Cosine also has an additional `fadd` (floating-point add) instruction.
* Finally, we have one more pair of `fmadd` instructions to calculate x and y respectively, and an `stp` instruction to store the pair of coordinates.

* 前三条 `eor`（异或）指令与 `lsl`（左移）或 `lsr`（右移）指令组合，对应于 `XorShift32::gen` 函数。
* 接下来使用 `ucvtf`（将无符号整数转换为浮点数）和 `fmul`（浮点乘法）指令将角度从度degree转换为弧度radian（代码中的第35行）。
* 正弦和余弦函数都包含两条 `fmul` 指令和一条 `fmadd`（浮点融合乘加）指令。余弦函数还包含一条额外的 `fadd`（浮点加法）指令。
* 最后，我们还有一对 `fmadd` 指令分别用于计算x和y坐标，以及一条 `stp` 指令用于存储这对坐标。

You expect this code to "fly", however, there is one very nasty performance problem that slows down the program. Without looking ahead in the text, can you find a recurrent dependency chain in the code?

你可能期望这段代码运行速度很快，然而，它存在一个非常棘手的性能问题，会拖慢程序的运行速度。在不查看后续文本的情况下，你能在代码中找到一个循环依赖链吗？

Congratulations if you've found it. There is a recurrent loop dependency on `XorShift32::val`. To generate the next random number, the generator has to produce the previous number first. The next call of method `XorShift32::gen` will generate the number based on the previous one. Figure @fig:DepChain visualizes the problematic loop-carry dependency. Notice, that the code for calculating particle coordinates (convert the angle to radians, sine, cosine, multiple results by velocity) starts executing as soon as the corresponding random number is ready, but not sooner.

如果你找到了，恭喜你。代码中存在一个对 `XorShift32::val` 的循环依赖。为了生成下一个随机数，生成器必须先生成前一个随机数。下一次调用 `XorShift32::gen` 方法时，会基于前一个随机数生成下一个随机数。图 @fig:DepChain 可视化了这种循环进位依赖关系。请注意，用于计算粒子坐标（将角度转换为弧度、计算正弦、余弦值，并将结果乘以速度）的代码会在相应的随机数生成后立即开始执行，不会提前执行。

![Visualization of dependent execution in [@lst:DepChain] DepChain[@lst:DepChain]中依赖执行的可视化](../../img/computation-opts/DepChain.png){#fig:DepChain width=90%}

The code that calculates the coordinates of particle `N` is not dependent on particle `N-1`, so it could be beneficial to pull them left to overlap their execution even more. You probably want to ask: "But how can those three (or six) instructions drag down the performance of the whole loop?". Indeed, there are many other "heavy" instructions in the loop, like `fmul` and `fmadd`. However, they are not on the critical path, so they can be executed in parallel with other instructions. And because modern CPUs are very wide, they will execute instructions from multiple iterations at the same time. This allows the OOO engine to effectively find parallelism (independent instructions) within different iterations of the loop. 

计算粒子 `N` 坐标的代码并不依赖于粒子 `N-1`，因此将它们的执行路径向左移动，使其执行重叠更多，可能是有益的。你可能会问：“但这3条（或6条）指令怎么会降低整个循环的性能呢？”。的确，循环中还有许多其他“耗时”的指令，例如 `fmul` 和 `fmadd`。然而，它们并不在关键路径上，因此可以与其他指令并行执行。而且，由于现代CPU的带宽非常大，它们可以同时执行来自多个迭代的指令。这使得乱序执行(OOO)引擎能够有效地在循环的不同迭代中找到并行性（独立指令）。

Let's do some back-of-the-envelope calculations.[^1] Each `eor` and `lsl` instruction incurs 2 cycles of latency: one cycle for the shift and one for the XOR. We have three dependent `eor + lsl` pairs, so it takes 6 cycles to generate the next random number. This is our absolute minimum for this loop: we cannot run faster than 6 cycles per iteration. The code that follows takes at least 20 cycles of latency to finish all the `fmul` and `fmadd` instructions. But it doesn't matter, because they are not on the critical path. The thing that matters is the throughput of these instructions. A useful rule of thumb: if an instruction is on a critical path, look at its latency, otherwise look at its throughput. On every loop iteration, we have 5 `fmul` and 4 `fmadd` instructions that are served on the same set of execution units. The M1 processor can run 4 instructions per cycle of this type, so it will take at least `9/4 = 2.25` cycles to issue all the `fmul` and `fmadd` instructions. So, we have two performance limits: the first is imposed by the software (6 cycles per iteration due to the dependency chain), and the second is imposed by the hardware (2.25 cycles per iteration due to the throughput of the execution units). Right now we are bound by the first limit, but we can try to break the dependency chain to get closer to the second limit.

让我们做一些粗略的估算[^1]。每条 `eor` 和 `lsl` 指令都会产生2个时钟周期的延迟：一个周期用于移位，一个周期用于异或运算。我们有3对相关的 `eor + lsl` 指令，因此生成下一个随机数需要6个时钟周期。这是此循环的绝对最小延迟：每次迭代的运行速度不能超过6个时钟周期。接下来的代码至少需要20个时钟周期的延迟才能完成所有 `fmul` 和 `fmadd` 指令。但这并不重要，因为它们不在关键路径上。重要的是这些指令的吞吐率。一个有用的经验法则是：如果一条指令在关键路径上，则关注其延迟；否则，关注其吞吐率。在每次循环迭代中，我们有5条 `fmul` 指令和4条 `fmadd` 指令在同一组执行单元上执行。 M1处理器每个周期可以执行4条此类指令，因此执行所有 `fmul` 和 `fmadd` 指令至少需要 `9/4 = 2.25` 个周期。所以，我们面临两个性能限制：第一个限制来自软件（由于依赖链，每次迭代需要6个周期），第二个限制来自硬件（由于执行单元的吞吐率，每次迭代需要2.25个周期）。目前我们受限于第一个限制，但我们可以尝试打破依赖链，以更接近第二个限制。

One of the ways to solve this would be to employ an additional RNG object so that one of them feeds even iterations and another feeds odd iterations of the loop as shown in [@lst:DepChainFixed]. Notice, that we also manually unrolled the loop. Now we have two separate dependency chains, which can be executed in parallel. You can argue that this changes the functionality of the program, but users would not be able to tell the difference since the motion of particles is random anyway. An alternative solution would be to pick a different RNG that has a less expensive internal dependency chain.

一种解决方法是使用额外的RNG对象，其中一个用于循环的偶数次迭代，另一个用于循环的奇数次迭代，如 [@lst:DepChainFixed] 所示。请注意，我们还手动展开了循环。现在我们有了两条独立的依赖链，可以并行执行。你可能会认为这会改变程序的功能，但用户无法察觉到任何区别，因为粒子的运动本来就是随机的。另一种解决方案是选择一个内部依赖链开销更小的随机数生成器（RNG）。

Listing: Random Particle Motion on a 2D Surface
代码列表：一个二维表面上的随机粒子运动
		
~~~~ {#lst:DepChainFixed .cpp}
void particleMotion(vector<Particle> &particles, 
                    uint32_t seed1, uint32_t seed2) {
  XorShift32 rng1(seed1);
  XorShift32 rng2(seed2);
  for (int i = 0; i < STEPS; i++) {
    for (int j = 0; j + 1 < particles.size(); j += 2) {
      uint32_t angle1 = rng1.gen();
      float angle_rad1 = angle1 * DEGREE_TO_RADIAN;
      particles[j].x += cosine(angle_rad1) * particles[j].velocity;
      particles[j].y += sine(angle_rad1)   * particles[j].velocity;
      uint32_t angle2 = rng2.gen();
      float angle_rad2 = angle2 * DEGREE_TO_RADIAN;
      particles[j+1].x += cosine(angle_rad2) * particles[j+1].velocity;
      particles[j+1].y += sine(angle_rad2)   * particles[j+1].velocity;
    }
    // remainder (not shown)
  }
}
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Once you do this transformation, the compiler starts autovectorizing the body of the loop, i.e., it glues two chains together and uses SIMD instructions to process them in parallel. To isolate the effect of breaking the dependency chain, I disabled compiler vectorization. 

完成此转换后，编译器会开始自动向量化循环体，也就是说，它会将两条链连接起来，并使用SIMD指令并行处理它们。为了隔离破坏依赖链的影响，我禁用了编译器向量化。

To measure the performance impact of the change, I ran "before" and "after" versions and observed the running time go down from 19ms per iteration to 10ms per iteration. This is almost a 2x speedup. The `IPC` also goes up from 4.0 to 7.1. To do my due diligence, I also measured other metrics to make sure performance didn't accidentally improve for other reasons. In the original code, the `MPKI` is 0.01, and `BranchMispredRate` is 0.2%, which means the program initially did not suffer from cache misses or branch mispredictions. Here is another data point: when running the same code on Intel's Alder Lake system, it shows 74% Retiring and 24% Core Bound, which confirms the performance is bound by computations.

为了衡量此更改对性能的影响，我运行了“更改前”和“更改后”的版本，观察到每次迭代的运行时间从19毫秒降至10毫秒。这几乎是两倍的加速。`IPC` 也从4.0提升至7.1。为了确保性能提升并非由于其他原因，我还测量了其他指标。在原始代码中，`MPKI` 为0.01，`BranchMispredRate` 为0.2%，这意味着程序最初没有出现缓存未命中或分支预测错误的情况。这里还有另一个数据点：在Intel Alder Lake系统上运行相同的代码时，结果显示74%的处于“完成Retiring”状态，24%的处于“核心瓶颈”状态，这证实了性能受限于计算量。

With a few additional changes, you can generalize this solution to have as many dependency chains as you want. For the M1 processor, the measurements show that having 2 dependency chains is enough to get very close to the hardware limit. Having more than 2 chains brings a negligible performance improvement. However, there is a trend that CPUs are getting wider, i.e., they become increasingly capable of running multiple dependency chains in parallel. That means future processors could benefit from having more than 2 dependency chains. As always you should measure and find the sweet spot for the platforms your code will be running on.

只需稍作修改，即可将此解决方案推广到任意数量的依赖链。对于M 处理器，测量结果表明，两条依赖链就足以接近硬件极限。超过两条依赖链带来的性能提升微乎其微。然而，CPU的发展趋势是“加宽”，也就是说，它们越来越能够并行运行多条依赖链。这意味着未来的处理器可能会受益于两条以上的依赖链。一如既往，您应该测量并找到代码运行平台的最佳性能点。

Sometimes it's not enough just to break dependency chains. Imagine that instead of a simple RNG, you have a very complicated cryptographic algorithm that is `10,000` instructions long. So, instead of a very short 6-instruction dependency chain, we now have `10,000` instructions standing on the critical path. You immediately do the same change we did above anticipating a nice 2x speedup, but see only 5% better performance. What's going on?

有时，仅仅打破依赖链是不够的。想象一下，您面对的不是一个简单的随机数生成器，而是一个包含10,000条指令的极其复杂的加密算法。因此，原本只有6条指令的短依赖链，现在关键路径上却有10,000条指令。你立即按照之前的方法进行修改，预期速度会提升2倍，但结果却只提升了5%。这是怎么回事？

The problem here is that the CPU simply cannot "see" the second dependency chain to start executing it. Recall from Chapter 3, that the Reservation Station (RS) capacity is not enough to see `10,000` instructions ahead as its number of entries is much smaller. So, the CPU will not be able to overlap the execution of two dependency chains. To fix this, we need to *interleave* those two dependency chains. With this approach, you need to change the code so that the RNG object will generate two numbers simultaneously, with *every* statement within the function `XorShift32::gen` duplicated and interleaved. Even if a compiler inlines all the code and can clearly see both chains, it doesn't automatically interleave them, so you need to watch out for this. Another limitation you may hit is register pressure. Running multiple dependency chains in parallel requires keeping more state and thus more registers. If you run out of architectural registers, the compiler will start spilling them to the stack, which will slow down the program.

问题在于CPU根本“看不到”第​​二条依赖链，所以无法开始执行。回想一下第三章的内容，保留站(RS: Reservation Station)的容量不足以看到10,000条指令，因为它的条目数量远小于此。因此，CPU无法同时执行两条依赖链。为了解决这个问题，我们需要交错执行这两条依赖链。采用这种方法，你需要修改代码，使RNG对象同时生成两个数字，并将函数 `XorShift32::gen` 中的每一条语句复制并交错执行。即使编译器内联了所有代码，并且能够清楚地看到两条依赖链，它也不会自动交错执行，因此您需要注意这一点。您可能遇到的另一个限制是寄存器压力。并行运行多条依赖链需要维护更多状态，因此也需要更多寄存器。如果体系结构寄存器耗尽，编译器会将数据溢出到栈，这将降低程序速度。

It is worth mentioning that data dependencies can also be created through memory. For example, if you write to memory location `M` on loop iteration `N` and read from this location on iteration `N+1`, there will effectively be a dependency chain. The stored value may be forwarded to a load, but these instructions cannot be reordered and executed in parallel.

值得一提的是，数据依赖也可以通过内存创建。例如，如果您在循环迭代 `N` 时写入内存地址 `M`，并在迭代 `N+1` 时从该地址读取数据，则实际上就存在一条依赖链。存储的值可能会被转发到加载指令，但这些指令不能重新排序并并行执行。

As a closing thought, I would like to emphasize the importance of finding that critical dependency chain. It is not always easy, but it is crucial to know what stands on the critical path in your loop, function, or other block of code. Otherwise, you may find yourself fixing secondary issues that barely make a difference.

最后，我想强调找到关键依赖链的重要性。这并不总是容易的，但了解循环、函数或其他代码块中的关键路径至关重要。否则，您可能会发现自己在修复一些几乎无关紧要的次要问题。

[^1]: Apple published instruction latency and throughput data in [@AppleOptimizationGuide, Appendix A]. 苹果公司在[@AppleOptimizationGuide，附录A]中发布了指令延迟和吞吐率数据。
[^2]: Performance Ninja: Dependency Chains 2 - [https://github.com/dendibakh/perf-ninja/tree/main/labs/core_bound/dep_chains_2](https://github.com/dendibakh/perf-ninja/tree/main/labs/core_bound/dep_chains_2)
