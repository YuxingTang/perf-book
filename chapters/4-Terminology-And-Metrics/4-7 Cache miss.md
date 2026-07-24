## Cache Miss 缓存失效/缓存未命中

As discussed in [@sec:MemHierar], any memory request missing in a particular level of cache must be serviced by higher-level caches or DRAM. This implies a significant increase in the latency of such memory access. The typical latency of memory subsystem components is shown in Table {@tbl:mem_latency}. Performance greatly suffers when a memory request misses in the Last Level Cache (LLC) and goes all the way down to the main memory.[^3]
如 [@sec:MemHierar] 中所述，任何在特定级别缓存中未命中的内存请求都必须由更高级别的缓存或DRAM来处理。这意味着此类存储访问的延迟会显著增加。内存子系统组件的典型延迟如表 {@tbl:mem_latency} 所示。当存储请求未命中末级缓存(LLC: Last Level Cache)并一直向下跳转到主内存时，性能会大幅下降[^3]。

-------------------------------------------------
Memory Hierarchy Component   Latency (cycle/time)
存储层次组件                   延迟（周期/时间）

--------------------------   --------------------
L1 Cache                     4 cycles (~1 ns)
L1缓存                        4周期（约1纳秒）

L2 Cache                     10-25 cycles (5-10 ns)
L2缓存                        10到25周期（5-10纳秒）

L3 Cache                     ~40 cycles (20 ns)
L3缓存                        约40周期（20纳秒）

Main                         200+ cycles (100 ns)
Memory
主内存                        超过200周期（100纳秒）

-------------------------------------------------

Table: Typical latency of a memory subsystem in x86-based platforms. x86平台的存储子系统的典型延迟。{#tbl:mem_latency}

Both instruction and data fetches can miss in the cache. According to Top-down Microarchitecture Analysis (see [@sec:TMA]), an instruction cache (I-cache) miss is characterized as a Frontend stall, while a data cache (D-cache) miss is characterized as a Backend stall. Instruction cache misses happen very early in the CPU pipeline during instruction fetch. Data cache misses happen much later during the instruction execution phase.

指令和数据读取都可能发生缓存未命中。根据自顶向下微体系结构分析（参见[@sec:TMA]），指令缓存（I-cache）未命中被定义为前端停顿，而数据缓存（D-cache）未命中被定义为后端停顿。指令缓存未命中发生在CPU流水线的早期取指阶段。数据缓存未命中则发生在指令执行阶段的后期。

Linux `perf` users can collect the number of L1 cache misses by running:
Linux `perf` 用户可以通过运行以下命令来收集L1缓存未命中次数：

```bash
$ perf stat -e mem_load_retired.fb_hit,mem_load_retired.l1_miss,
  mem_load_retired.l1_hit,mem_inst_retired.all_loads -- a.exe
   29580  mem_load_retired.fb_hit
   19036  mem_load_retired.l1_miss
  497204  mem_load_retired.l1_hit
  546230  mem_inst_retired.all_loads
```

Above is the breakdown of all loads for the L1 data cache and fill buffers. A load might either hit the already allocated fill buffer (`fb_hit`), hit the L1 cache (`l1_hit`), or miss both (`l1_miss`), thus `all_loads = fb_hit + l1_hit + l1_miss`.[^2] We can see that only 3.5% of all loads miss in the L1 cache, thus the *L1 hit rate* is 96.5%.

以上是L1数据缓存和填充缓冲区所有加载操作的详细统计。加载Load操作可能命中已分配的填充缓冲区（`fb_hit`），命中L1缓存（`l1_hit`），或者两者都未命中（`l1_miss`），因此 `all_loads = fb_hit + l1_hit + l1_miss`[^2]。我们可以看到，所有加载操作中只有3.5%未命中L1缓存，因此L1缓存命中率高达96.5%。

We can further break down L1 data misses and analyze L2 cache behavior by running:

我们可以通过运行以下命令进一步细分L1数据未命中情况并分析L2缓存的行为：

```bash
$ perf stat -e mem_load_retired.l1_miss,
  mem_load_retired.l2_hit,mem_load_retired.l2_miss -- a.exe
  19521  mem_load_retired.l1_miss
  12360  mem_load_retired.l2_hit
   7188  mem_load_retired.l2_miss
```

From this example, we can see that 37% of loads that missed in the L1 D-cache also missed in the L2 cache, thus the *L2 hit rate* is 63%. A breakdown for the L3 cache can be made similarly.

从这个例子中我们可以看出，L1数据缓存中未命中的请求有37%在L2缓存中也未命中，因此L2缓存的命中率为 63%。L3缓存的分析方法类似。

[^2]: Careful readers may notice a discrepancy in the numbers: `fb_hit + l1_hit + l1_miss = 545,820`, which doesn't exactly match `all_loads`. Most likely it's due to slight inaccuracy in hardware event collection, but we did not investigate this since the numbers are very close.细心的读者可能会注意到数字上的差异：`fb_hit + l1_hit + l1_miss = 545,820`，这与 `all_loads` 的值并不完全一致。这很可能是由于硬件事件收集存在轻微误差造成的，但由于数值非常接近，我们没有对此进行深入调查。
[^3]: There is also an interactive view that visualizes the latency of different operations in modern systems 此外，还有一个交互式视图可以可视化现代系统中不同操作的延迟 - [https://colin-scott.github.io/personal_website/research/interactive_latency.html](https://colin-scott.github.io/personal_website/research/interactive_latency.html)
