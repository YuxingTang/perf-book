## Hardware-Based Sampling Features 基于硬件的采样特性

Major CPU vendors provide a set of additional features to enhance sampling. Since CPU vendors approach performance monitoring in different ways, those capabilities vary in not only how they are called but also what you can do with them. In Intel processors, it is called Processor Event-Based Sampling (PEBS), first introduced in NetBurst microarchitecture. A similar feature on AMD processors is called Instruction Based Sampling (IBS) and is available starting with the AMD Opteron Family (10h generation) of cores. Next, we will discuss those features in more detail, including their similarities and differences.

主流CPU厂商提供了一系列额外的特性来增强采样效果。由于CPU厂商采用不同的性能监控方式，这些特性不仅名称各异，其功能也各有不同。在Intel处理器中，这项特性被称为处理器事件采样(PEBS: Processor Event-Based Sampling)，最初在NetBurst微体系结构中引入。AMD处理器上的类似特性被称为指令采样(IBS: Instruction Based Sampling)，从AMD Opteron系列（第十代）核心开始提供。接下来，我们将更详细地讨论这些特性，包括它们的异同。

### PEBS on Intel Platforms Intel平台上的PEBS {#sec:secPEBS}

Similar to the Last Branch Record feature, PEBS is used while profiling the program to capture additional data with every collected sample. When a performance counter is configured for PEBS, the processor saves the set of additional data, which has a defined format and is called the PEBS record. The format of a PEBS record for the Intel Skylake CPU is shown in Figure @fig:PEBS_record. It contains the state of general-purpose registers (`EAX`, `EBX`, `ESP`, etc.), `EventingIP`, `Data Linear Address`, and `Latency value`, and a few other fields. The content layout of a PEBS record varies across different microarchitectures, see [@IntelOptimizationManual, Volume 3B, Chapter 20 Performance Monitoring].

与最后分支记录(LBR)特性类似，PEBS 于程序性能分析，以便在每次采集样本时捕获额外数据。当一个性能计数器配置为PEBS时，处理器会保存这组具有特定格式的额外数据，这些数据被称为PEBS记录。图 @fig:PEBS_record 展示了Intel Skylake CPU的PEBS记录格式。它包含通用寄存器（例如 `EAX`、`EBX`、`ESP` 等）、`EventingIP`、`数据线性地址` 和 `延迟值` ，以及其他一些字段。PEBS记录的内容布局会因微体系结构的不同而有所差异，请参阅 [@IntelOptimizationManual,卷3B, 第20章--性能监控]。

![PEBS Record Format for 6th Generation, 7th Generation and 8th Generation Intel Core Processor Families. *© Source: [@IntelOptimizationManual, Volume 3B, Chapter 20].* 适用于第六代、第七代和第八代Intel Core处理器系列的PEBS记录格式。*© 来源：[@IntelOptimizationManual, 卷3B, 第20章].*](../../img/pmu-features/PEBS_record.png){#fig:PEBS_record width=100%}

Since Skylake, the PEBS record has been enhanced to collect XMM registers and Last Branch Record (LBR) records. The format has been restructured where fields are grouped into Basic group, Memory group, GPR group, XMM group, and LBR group. Performance profiling tools have the option to select data groups of interest and thus reduce the recording overhead. By default, the PEBS record will only contain the Basic group.

自Skylake体系结构以来，PEBS记录得到了增强，可以收集XMM寄存器和最后分支记录(LBR)。格式已重新调整，字段被分组为基本组、内存组、GPR 组、XMM组和LBR组。性能分析工具可以选择感兴趣的数据组，从而降低记录开销。默认情况下，PEBS记录仅包含基本组。

One of the notable benefits of using PEBS is lower sampling overhead compared to regular interrupt-based sampling. Recall that when the counter overflows, the CPU generates an interrupt to collect one sample. Frequently generating interrupts and having an analysis tool itself capture the program state inside the interrupt service routine is very costly since it involves OS interaction. 

使用PEBS的一个显著优势是，与常规的基于中断的采样相比，其采样开销更低。回想一下，当计数器溢出时，CPU会生成一个中断来采集一个样本。频繁生成中断并让分析工具在中断服务例程中捕获程序状态会非常耗费资源，因为它涉及到操作系统交互。

On the other hand, PEBS keeps a buffer to temporarily store multiple PEBS records. Suppose, we are sampling load instructions using PEBS. When a performance counter is configured for PEBS, an overflow condition in the counter will not trigger an interrupt, instead, it will activate the PEBS mechanism. The mechanism will then trap the next load, capture a new record, and store it in the dedicated PEBS buffer area. The mechanism also takes care of clearing the counter overflow status and reloading the counter with the initial value. Only when the dedicated buffer is full does the processor raise an interrupt and the buffer gets flushed to memory. This mechanism lowers the sampling overhead by triggering fewer interrupts.

另一方面，PEBS会维护一个缓冲区来临时存储多个PEBS记录。假设我们使用PEBS对加载load指令进行采样。当性能计数器配置为PEBS时，计数器溢出不会触发中断，而是会激活PEBS机制。该机制随后会捕获下一个加载指令，捕获一条新记录，并将其存储在专用的PEBS缓冲区区域中。该机制还负责清除计数器溢出状态并将计数器重新加载为初始值。只有当专用缓冲区已满时，处理器才会触发中断并将缓冲区内容刷新到内存中。这种机制通过减少触发中断的次数来降低采样开销。

Linux users can check if PEBS is enabled by executing `dmesg`:

Linux用户可以通过执行 `dmesg` 命令来检查PEBS是否已启用：

```bash
$ dmesg | grep PEBS
[    0.113779] Performance Events: XSAVE Architectural LBR, PEBS fmt4+-baseline,  
AnyThread deprecated, Alderlake Hybrid events, 32-deep LBR, full-width counters, Intel PMU driver.
```

For LBR, Linux perf dumps the entire contents of the LBR stack with every collected sample. So, it is possible to analyze raw LBR dumps collected by Linux perf. However, for PEBS, Linux `perf` doesn't export the raw output as it does for LBR. Instead, it processes PEBS records and extracts only a subset of data depending on a particular need. So, it's not possible to access the collection of raw PEBS records with Linux `perf`. However, Linux `perf` provides some PEBS data processed from raw samples, which can be accessed by `perf report -D`. To dump raw PEBS records, you can use [`pebs-grabber`](https://github.com/andikleen/pmu-tools/tree/master/pebs-grabber)[^1].

对于LBR，Linux perf会在每次采集样本时转储整个LBR堆栈的内容。因此，可以分析Linux perf采集的原始LBR转储。然而，对于PEBS，Linux `perf` 不会像处理LB 那样导出原始输出。相反，它会处理PEBS记录，并根据特定需求仅提取数据子集。因此，无法使用Linux `perf` 访问原始PEBS记录集合。但是，Linux `perf` 提供了一些从原始样本处理后的PEBS数据，可以使用 `perf report -D` 访问这些数据。要转储原始PEBS记录，可以使用 [`pebs-grabber`](https://github.com/andikleen/pmu-tools/tree/master/pebs-grabber)[^1]。

### IBS on AMD Platforms AMD平台上的IBS

Instruction-Based Sampling (IBS) is an AMD64 processor feature that can be used to collect specific metrics related to instruction fetch and instruction execution. The pipeline of an AMD processor consists of two separate phases: a Frontend phase that fetches AMD64 instruction bytes and a Backend phase that executes `ops`. As the phases are logically separated, there are two independent sampling mechanisms: IBS Fetch and IBS Execute. 

基于指令的采样(IBS: Instruction-Based Sampling)是AMD64处理器的一项功能特性，可用于收集与指令获取和指令执行相关的特定指标。AMD处理器的流水线由两个独立的阶段组成：前端阶段负责获取AMD64指令字节，后端阶段负责执行`操作（ops）`。由于这两个阶段在逻辑上是分离的，因此存在两种独立的采样机制：IBS Fetch 和IBS Execute。

- IBS Fetch monitors the Frontend of the pipeline and provides information about ITLB (hit or miss), I-cache (hit or miss), fetch address, fetch latency, and a few other things.
- IBS Execute monitors the Backend of the pipeline and provides information about instruction execution behavior by tracking the execution of a single op. For example, branch (taken or not, predicted or not), and load/store (hit or miss in D-caches and DTLB, linear address, load latency).

- IBS Fetch监控流水线的前端，并提供有关ITLB（命中或未命中）、指令缓存（命中或未命中）、取指地址、取指延迟以及其他一些信息。
- IBS Execute监控流水线的后端，并通过跟踪单个指令的执行情况来提供有关指令执行行为的信息。例如，分支（是否发生、是否预测到分支）以及加载load/存储store（数据缓存和DTLB的命中或未命中、线性地址、加载延迟）。

There are several important differences between PMC and IBS in AMD processors. PMC counters are programmable, whereas IBS acts like fixed counters. IBS counters can only be enabled or disabled for monitoring, they can't be programmed to any selective events. IBS Fetch and Execute counters can be enabled/disabled independently. With PMC, the user has to decide what events to monitor ahead of time. With IBS, a rich set of data is collected for each sampled instruction and then it is up to the user to analyze parts of the data they are interested in. IBS selects and tags an instruction to be monitored and then captures microarchitectural events caused by this instruction during its execution. A more detailed comparison of Intel PEBS and AMD IBS can be found in [@ComparisonPEBSIBS].

AMD处理器中的PMC和IBS之间存在几个重要的区别。PMC计数器是可编程的，而IBS则类似于固定计数器。IBS计数器只能启用或禁用以进行监控，不能针对特定事件进行编程。IBS的取指计数器和执行计数器可以独立启用/禁用。使用PMC时，用户必须预先决定要监控哪些事件。而使用IBS，系统会为每条采样指令收集丰富的数据集，然后由用户自行分析感兴趣的数据部分。IBS会选择并标记要监控的指令，然后在指令执行期间捕获由该指令引起的微体系结构事件。关于Intel PEBS和AMD IBS的更详细比较，请参阅 [@ComparisonPEBSIBS]。

Since IBS is integrated into the processor pipeline and acts as a fixed event counter, the sample collection overhead is minimal. Profilers are required to process the IBS-generated data, which could be huge in size depending upon sampling interval, number of threads configured, whether Fetch/Execute configured, etc. Until Linux kernel version 6.1, IBS always collects samples for all the cores. This limitation causes huge data collection and processing overhead. From Kernel 6.2 onwards, Linux perf supports IBS sample collection only for the configured cores. 

由于IBS集成到处理器流水线中并充当固定事件计数器，因此采样收集的开销极小。分析器需要处理IBS生成的数据，这些数据的大小可能非常庞大，具体取决于采样间隔、配置的线程数、是否配置了取指/执行等因素。在Linux内核版本6.1之前，IBS始终会收集所有核心的样本。此限制会导致巨大的数据收集和处理开销。从内核6.2开始，Linux perf仅支持对已配置的核心进行IBS样本收集。

IBS is supported by Linux perf and the AMD uProf profiler. Here are sample commands to collect IBS Execute and Fetch samples:

Linux perf和A​​MD uProf分析器均支持IBS。以下是收集IBS执行样本和获取样本的示例命令：

```bash
$ perf record -a -e ibs_op/cnt_ctl=1,l3missonly=1/ -- benchmark.exe
$ perf record -a -e ibs_fetch/l3missonly=0/ -- benchmark.exe
$ perf report
```

where `cnt_ctl=0` counts clock cycles, `cnt_ctl=1` counts dispatched ops for an interval period; `l3missonly=1` only keeps the samples that had an L3 miss. These two parameters and a few others are described in more detail in [@AMDUprofManual, Table 25. AMDuProfCLI Collect Command Options]. Note that in both of the commands above, the `-a` option is used to collect IBS samples for all cores, otherwise `perf` would fail to collect samples on Linux kernel 6.1 or older. From version 6.2 onwards, the `-a` option is no longer needed unless you want to collect IBS samples for all cores. The `perf report` command will show samples attributed to functions and source code lines similar to regular PMU events but with added features that we will discuss later. AMD uProf command line tool can generate IBS raw data, which later can be converted to a CSV file for later postprocessing with MS Excel as described in [@AMDUprofManual, Section 7.10 ASCII Dump of IBS Samples].

其中，`cnt_ctl=0` 统计时钟周期数，`cnt_ctl=1` 统计指定时间间隔内已派发的操作数；`l3missonly=1` 仅保留发生L3缓存未命中的样本。这两个参数以及其他一些参数在 [@AMDUprofManual，表25. AMDuProfCLI 收集命令选项] 中有更详细的描述。请注意，在上述两个命令中，`-a` 选项用于收集所有核心的IBS样本，否则在 Linux内核6.1或更早版本上，`perf` 将无法收集样本。从6.2版本开始，除非您想要收集所有核心的IBS样本，否则不再需要 `-a` 选项。`perf report` 命令将显示归因于函数和源代码行的样本，类似于常规的PMU事件，但还包含一些我们稍后将讨论的附加功能。AMD uProf命令行工具可以生成IBS原始数据，之后可以将其转换为CSV文件，以便使用MS Excel进行后续处理，具体操作请参见[@AMDUprofManual，第7.10节--IBS样本的ASCII转储]。

### SPE on Arm Platforms 在Arm平台上的SPE

The Arm Statistical Profiling Extension (SPE) is an architectural feature designed for enhanced instruction execution profiling within Arm CPUs. The SPE feature extension is specified as part of Armv8-A architecture, with support from Arm v8.2 onwards. Arm SPE extension is architecturally optional, which means that Arm processor vendors are not required to implement it. Arm Neoverse cores have supported SPE since Neoverse N1 cores, which were introduced in 2019. 

Arm统计轮廓分析扩展(SPE: Statistical Profiling Extension)是一种体系结构特性，旨在增强Arm CPU 中的指令执行分析。SPE特性扩展是Armv8-A体系结构的一部分，从Arm v8.2开始支持。Arm SPE扩展在体系结构上是可选的，这意味着Arm处理器厂商无需实现它。Arm Neoverse核心自2019年推出的Neoverse N1以来就支持SPE。

Compared to other solutions, SPE is more similar to AMD IBS than it is to Intel PEBS. Similar to IBS, SPE is separate from the general performance monitor counters (PMC), but instead of two flavors of IBS (fetch and execute), there is just a single mechanism.

与其他解决方案相比，SPE与AMD IBS的相似度高于Intel PEBS。与IBS类似，SPE独立于通用性能监视器计数器 (PMC)，但它不像IBS那样提供两种采样方式（取指和执行），而只有一个采样机制。

The SPE sampling process is built in as part of the instruction execution pipeline. Sample collection is still based on a configurable interval, but operations are statistically selected. Each sampled operation generates a sample record, which contains various data about the execution of this operation. SPE record saves the address of the instruction, the virtual and physical address for the data accessed by loads and stores, the source of the data access (cache or DRAM), and the timestamp to correlate with other events in the system. Also, it can give latency of various pipeline stages, such as Issue latency (from dispatch to execution), Translation latency (cycle count for a virtual-to-physical address translation), and Execution latency (latency of load/stores in the functional unit). The whitepaper [@ARMSPE] describes Arm SPE in more detail as well as shows a few optimization examples using it.

SPE采样过程内置于指令执行流水线中。采样仍然基于可配置的时间间隔，但操作的选择是统计性的。每个采样操作都会生成一个采样记录，其中包含有关该操作执行的各种数据。SPE记录保存指令地址、加载和存储操作访问的数据的虚拟地址和物理地址、数据访问来源（缓存或DRAM）以及用于关联系统中其他事件的时间戳。此外，它还可以提供各个流水线阶段的延迟，例如指令发射延迟（从分派调度到执行）、转换延迟（虚拟地址到物理地址转换的周期数）和执行延迟（功能单元中加载/存储操作的延迟）。白皮书 [@ARMSPE] 更详细地描述了Arm SPE，并展示了一些使用它的优化示例。

Similar to Intel PEBS and AMD IBS, Arm SPE helps to reduce the sampling overhead and enables longer collections. In addition to that, it supports postfiltering of sample records, which helps to reduce the memory required for storage. SPE profiling is supported in Linux `perf` and can be used as follows:[^6]

与Intel PEBS和AMD IBS类似，Arm SPE有助于降低采样开销并支持更长的采样时间。此外，它还支持对采样记录进行后过滤，从而有助于减少存储所需的内存。Linux `perf` 支持SPE分析，使用方法如下：[^6]

```bash
$ perf record -e arm_spe_0/<controls>/ -- test_program
$ perf report --stdio
$ spe-parser perf.data -t csv
```

where `<controls>` lets you optionally specify various controls and filters for the collection. `perf report` will give the usual output according to what the user asked for with `<controls>` options. `spe-parser`[^5] is a tool developed by Arm engineers to parse the captured perf record data and save all the SPE records into a CSV file.

`<controls>` 允许您选择性地为集合指定各种控件和过滤器。`perf report` 将根据用户通过 `<controls>` 选项请求的内容给出常规输出。`spe-parser`[^5]是Arm工程师开发的一个工具，用于解析捕获的perf记录数据并将所有SPE记录保存到CSV文件中。

Now that we covered the advanced sampling features, let's discuss how they can be used to improve performance analysis.

现在我们已经了解了高级采样功能，接下来我们将讨论如何利用它们来改进性能分析。

### Precise Events 精确事件

One of the major problems in sampling is pinpointing the exact instruction that caused a particular performance event. As discussed in [@sec:profiling], interrupt-based sampling is based on counting a specific performance event and waiting until it overflows. When an overflow happens, it takes a processor some time to stop the execution and tag the instruction that caused the overflow. This is especially difficult for modern complex out-of-order CPU architectures.

采样的主要问题之一是精确定位导致特定性能事件的指令。正如 [@sec:profiling] 中所述，基于中断的采样基于对特定性能事件进行计数，并等待其溢出。当发生溢出时，处理器需要一些时间来停止执行并标记导致溢出的指令。这对于现代复杂的乱序CPU体系结构来说尤其困难。

It introduces the notion of a skid, which is defined as the distance between the IP (instruction address) that caused the event to the IP where the event is tagged. Skid makes it difficult to discover the instruction causing the performance issue. Consider an application with a large number of cache misses and a hot assembly code that looks like this:

它引入了“滑移距离”（skid）的概念，滑移距离定义为导致事件的指令地址（IP）与事件被标记的指令地址之间的距离。滑移距离使得查找导致性能问题的指令变得困难。考虑一个存在大量缓存未命中且存在如下热汇编代码的应用程序：

```asm
; load1 
; load2
; load3
```

The profiler might tag `load3` as the instruction that causes a large number of cache misses, while in reality, `load1` is the instruction to blame. For high-performance processors, this skid can be hundreds of processor instructions. This usually causes a lot of confusion for performance engineers. Interested readers could learn more about the underlying reasons for such issues on [Intel Developer Zone website](https://software.intel.com/en-us/vtune-help-hardware-event-skid)[^4].

轮廓分析器可能会将 `load3` 指令标记为导致大量缓存未命中的指令，而实际上，罪魁祸首是 `load1` 指令。对于高性能处理器而言，这一滑移距离skid可能涉及数百条处理器指令。这通常会给性能工程师带来诸多困惑。感兴趣的读者可以访问 [Intel Developer Zone website](https://software.intel.com/en-us/vtune-help-hardware-event-skid)[^4] 了解更多关于此类问题的根本原因。

The problem with the skid is mitigated by having the processor itself store the instruction pointer (along with other information). With Intel PEBS, the `EventingIP` field in the PEBS record indicates the instruction that caused the event. This is typically available only for a subset of supported events, called "Precise Events". A complete list of precise events for a specific microarchitecture can be found in [@IntelOptimizationManual, Volume 3B, Chapter 20 Performance Monitoring]. An example of using PEBS precise events to mitigate skid can be found on the [easyperf blog](https://easyperf.net/blog/2018/08/29/Understanding-performance-events-skid).[^2]

通过让处理器自身存储指令指针（以及其他信息），可以缓解这种“缓存未命中”问题。在Intel PEBS中，PEBS记录中的 `EventingIP` 字段指示导致事件的指令。这通常仅适用于部分受支持的事件，称为“精确事件（Precise Events）”。特定微体系结构的精确事件完整列表可在 [@IntelOptimizationManual, 卷3B, 第20章--性能监控] 中找到。关于如何使用PEBS精确事件来缓解滑移距离（skid）问题，请参阅[easyperf博客](https://easyperf.net/blog/2018/08/29/Understanding-performance-events-skid)[^2]。

Listed below are precise events for the Intel Skylake Microarchitecture:

下面列出了Intel Skylake微体系结构的精确事件：

```
INST_RETIRED.*        OTHER_ASSISTS.*    BR_INST_RETIRED.*     BR_MISP_RETIRED.*
FRONTEND_RETIRED.*    HLE_RETIRED.*      RTM_RETIRED.*         MEM_INST_RETIRED.*
MEM_LOAD_RETIRED.*    MEM_LOAD_L3_HIT_RETIRED.*
```

, where `.*` means that all sub-events inside a group can be configured as precise events.

其中 `.*` 表示组内所有子事件都可以配置为精确事件。

With AMD IBS and Arm SPE, all the collected samples are precise by design since the hardware captures the exact instruction address. They both work in a very similar fashion. Whenever an overflow occurs, the mechanism saves the instruction causing the overflow into a dedicated buffer which is then read by the interrupt handler. As the address is preserved, the IBS and SPE sample's instruction attribution is precise.

对于AMD IBS和Arm SPE，由于硬件会捕获精确的指令地址，因此所有收集到的样本都是精确的。它们的工作方式非常相似。每当发生溢出时，该机制会将导致溢出的指令保存到一个专用缓冲区中，然后由中断处理程序读取该缓冲区。由于地址被保留，因此IBS和SPE样本的指令归属是精确的。

Users of Linux `perf` on Intel and AMD platforms must add the `pp` suffix to one of the events listed above to enable precise tagging as shown below. However, on Arm platforms, it has no effect, so users must use the `arm_spe_0` event.

在Intel和AMD平台上使用Linux `perf` 的用户必须将 `pp` 后缀添加到上面列出的某个事件中，才能启用精确标记，如下所示。但是，在Arm平台上，此操作无效，因此用户必须使用 `arm_spe_0` 事件。

```bash
$ perf record -e cycles:pp -- ./a.exe
```

Precise events provide relief for performance engineers as they help to avoid misleading data that often confuses beginners and even senior developers. The TMA methodology heavily relies on precise events to locate the exact line of source code where the inefficient execution takes place.

精确事件为性能工程师提供了便利，因为它们有助于避免误导性数据，这些数据常常会让初学者甚至高级开发人员感到困惑。TMA方法论高度依赖于精确的事件来定位低效执行发生的源代码行。

### Analyzing Memory Accesses 分析内存访问 {#sec:sec_PEBS_DLA}

Memory accesses are a critical factor for the performance of many applications. Both PEBS and IBS enable gathering detailed information about memory accesses in a program. For instance, you can sample loads and collect their target addresses and access latency. Keep in mind, that this does not trace all the stores and loads. Otherwise, the overhead would be too big. Instead, it analyzes only one out of 100,000 accesses or so. You can customize how many samples per second you want. With a large enough collection of samples, it can give an accurate statistical picture.

内存访问是许多应用程序性能的关键因素。PEBS和IBS都能够收集程序中内存访问的详细信息。例如，您可以对加载操作进行采样，并收集其目标地址和访问延迟。请注意，这并非追踪所有存储和加载操作，否则开销会过大。相反，它仅分析大约每10万次访问中的一次。您可以自定义每秒采样次数。通过收集足够多的样本，它可以提供准确的统计信息。

In PEBS, such a feature is called Data Address Profiling (DLA). To provide additional information about sampled loads and stores, it uses the `Data Linear Address` and `Latency Value` fields inside the PEBS facility (see Figure @fig:PEBS_record). If the performance event supports the DLA facility, and DLA is enabled, the processor will dump the memory address and latency of the sampled memory access. You can also filter memory accesses that have latency higher than a certain threshold. This is useful for finding long-latency memory accesses, which can be a performance bottleneck for many applications.

在PEBS中，此功能称为数据地址分析(DLA: Data Address Profiling)。为了提供有关采样加载和存储操作的更多信息，它使用PEBS工具中的`数据线性地址Data Linear Address`和`延迟值Latency Value`字段（参见图 @fig:PEBS_record）。如果性能事件支持DLA功能，并且DLA已启用，处理器将转储采样内存访问的内存地址和延迟。您还可以筛选延迟高于特定阈值的内存访问。这有助于查找长延迟内存访问，而长延迟内存访问可能是许多应用程序的性能瓶颈。

With the IBS Execute and Arm SPE sampling, you can also do an in-depth analysis of memory accesses performed by an application. One approach is to dump collected samples and process them manually. IBS saves the exact linear address, its latency, where the memory location was fetched from (cache or DRAM), and whether it hit or missed in the DTLB. SPE can be used to estimate the latency and bandwidth of the memory subsystem components, estimate memory latencies of individual loads/stores, and more.

借助IBS Execute和Arm SPE采样，您还可以对应用程序执行的内存访问进行深入分析。一种方法是转储收集的样本并手动处理。IBS会保存精确的线性地址、其延迟、内存位置的来源（缓存或DRAM）以及DTLB的命中或未命中情况。SPE可用于估算内存子系统组件的延迟和带宽、估算单个加载/存储操作的内存延迟等等。

One of the most important use cases for these extensions is detecting True and False Sharing, which we will discuss in [@sec:TrueFalseSharing]. The Linux `perf c2c` tool heavily relies on all three mechanisms (PEBS, IBS, and SPE) to find contested memory accesses, which could experience True/False sharing: it matches load/store addresses for different threads and checks if the hit occurs in a cache line modified by other threads.

这些扩展最重要的用例之一是检测真假共享，我们将在 [@sec:TrueFalseSharing] 中讨论。 Linux的 `perf c2c` 工具严重依赖三种机制（PEBS、IBS和SPE）来查找可能存在真/假共享的内存访问冲突：它匹配不同线程的加载/存储地址，并检查匹配是否发生在已被其他线程修改的缓存行中。

[^1]: PEBS grabber tool PEBS抓取工具 - [https://github.com/andikleen/pmu-tools/tree/master/pebs-grabber](https://github.com/andikleen/pmu-tools/tree/master/pebs-grabber). Requires root access.
[^2]: Performance skid 性能事件滑移skid - [https://easyperf.net/blog/2018/08/29/Understanding-performance-events-skid](https://easyperf.net/blog/2018/08/29/Understanding-performance-events-skid)
[^4]: Hardware event skid 硬件事件滑移skid - [https://software.intel.com/en-us/vtune-help-hardware-event-skid](https://software.intel.com/en-us/vtune-help-hardware-event-skid)
[^5]: Arm SPE parser Arm的SPE解析器 - [https://gitlab.arm.com/telemetry-solution/telemetry-solution](https://gitlab.arm.com/telemetry-solution/telemetry-solution)
[^6]: Linux perf driver for `arm_spe` should be installed first (see [https://developer.arm.com/documentation/ka005362/latest/](https://developer.arm.com/documentation/ka005362/latest/)). On Amazon Linux 2 and 2023, the SPE PMU is available by default on Graviton metal instances (see [https://github.com/aws/aws-graviton-getting-started/blob/main/perfrunbook/debug_hw_perf.md](https://github.com/aws/aws-graviton-getting-started/blob/main/perfrunbook/debug_hw_perf.md)) 应首先安装具有 `arm_spe` 的Linux perf驱动程序（参见[https://developer.arm.com/documentation/ka005362/latest/](https://developer.arm.com/documentation/ka005362/latest/))。在Amazon Linux 2和2023上，SPE PMU默认在Graviton裸金属实例上可用（参见 [https://github.com/aws/aws-graviton-getting-started/blob/main/perfrunbook/debug_hw_perf.md](https://github.com/aws/aws-graviton-getting-started/blob/main/perfrunbook/debug_hw_perf.md)）。
