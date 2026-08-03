## Code Instrumentation 代码插桩 {#sec:secInstrumentation}

Probably the first approach for doing performance analysis ever invented is *code instrumentation*. It is a technique that inserts extra code into a program to collect specific runtime information. [@lst:CodeInstrumentation] shows the simplest example of inserting a `printf` statement at the beginning of a function to indicate if this function is called. After that, you run the program and count the number of times you see "foo is called" in the output. Perhaps every programmer in the world did this at some point in their career at least once.

代码插桩可能是最早用于性能分析的方法之一。它是一种在程序中插入额外代码以收集特定运行时信息的技术。[@lst:CodeInstrumentation] 展示了一个最简单的示例：在函数开头插入一个 `printf` 语句，以指示该函数是否被调用。之后，运行程序并统计输出中出现“foo is called”的次数。或许世界上每个程序员在其职业生涯中都至少做过一次这样的操作。

Listing: Code Instrumentation
代码列表：代码插桩

~~~~ {#lst:CodeInstrumentation .cpp}
int foo(int x) {
+ printf("foo is called\n");
 // function body...
}
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The plus sign at the beginning of a line means that this line was added and is not present in the original code. In general, instrumentation code is not meant to be pushed into the codebase; rather, it's for collecting the needed data and later can be deleted.

行首的加号表示该行是后添加的，并非原始代码的一部分。一般来说，插桩代码不应该被推送到代码库中；相反，它的作用是收集所需数据，之后可以删除。

A more interesting example of code instrumentation is presented in [@lst:CodeInstrumentationHistogram]. In this made-up code example, the function `findObject` searches for the coordinates of an object with some properties `p` on a map. All objects are guaranteed to eventually be located. The function `getNewCoords` returns new coordinates within a bigger area that is provided as an argument. The function `findObj` returns the confidence level of locating the right object with the current coordinates `c`. If it is an exact match, we stop the search loop and return the coordinates. If the confidence is above the `threshold`, we call `zoomIn` to find a more precise location of the object. Otherwise, we get the new coordinates within the `searchArea` to try our search next time.

[@lst:CodeInstrumentationHistogram] 中展示了一个更有趣的插桩代码示例。在这个虚构的代码示例中，函数 `findObject` 在地图上搜索具有某些属性 `p` 的对象的坐标。所有对象最终都能被找到。函数 `getNewCoords` 返回一个更大的区域内的新坐标，该区域作为参数提供。函数 `findObj` 返回使用当前坐标 `c` 定位到正确对象的置信度。如果完全匹配，则停止搜索循环并返回坐标。如果置信度高于 `threshold`，则调用 `zoomIn` 来找到对象的更精确位置。否则，获取 `searchArea` 内的新坐标，以便下次尝试搜索。

The instrumentation code consists of two classes: `histogram` and `incrementor`. The former keeps track of whatever variable values we are interested in and frequencies of their occurrence and then prints the histogram *after* the program finishes. The latter is just a helper class for pushing values into the `histogram` object. It is simple and can be adjusted to your specific needs quickly.[^3]

插桩代码包含两个类：`histogram` 和 `incrementor`。前者跟踪我们感兴趣的变量值及其出现频率，并在程序运行结束后打印直方图。后者只是一个辅助类，用于将值添加到 `histogram` 对象中。它很简单，可以快速根据您的具体需求进行调整。[^3]

Listing: Code Instrumentation
代码列表：代码插桩

~~~~ {#lst:CodeInstrumentationHistogram .cpp}
+ struct histogram {
+   std::map<uint32_t, std::map<uint32_t, uint64_t>> hist;
+   ~histogram() {
+     for (auto& tripCount : hist)
+       for (auto& zoomCount : tripCount.second)
+         std::cout << "[" << tripCount.first << "]["
+                   << zoomCount.first << "] :  "
+                   << zoomCount.second << "\n";
+   }
+ };
+ histogram h;

+ struct incrementor {
+   uint32_t tripCount = 0;
+   uint32_t zoomCount = 0;
+   ~incrementor() {
+ 	   h.hist[tripCount][zoomCount]++;
+   }
+ };

Coords findObject(const ObjParams& p, Coords searchArea) {
+ incrementor inc;
  Coords c = getNewCoords(searchArea);
  while (true) {
+   inc.tripCount++;
    float match = findObj(p, c);
    if (exactMatch(match))
      return c;
    if (match > threshold) {
      searchArea = zoomIn(searchArea, c);
+     inc.zoomCount++;
    } else {
      c = getNewCoords(searchArea);
    }
  }
  return c;
}
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

In this hypothetical scenario, we added instrumentation to know how frequently we `zoomIn` before we find an object. The variable `inc.tripCount` counts the number of iterations the loop runs before it exits, and the variable `inc.zoomCount` counts how many times we reduce the search area (call to `zoomIn`). We always expect `inc.zoomCount` to be less or equal to `inc.tripCount`. 

在这个假设场景中，我们添加了检测代码，以了解在找到目标对象之前，我们调用 `zoomIn` 函数的频率。变量 `inc.tripCount` 统计循环在退出前执行的迭代次数，而变量 `inc.zoomCount` 统计缩小搜索范围（调用 `zoomIn` 函数）的次数。我们始终期望 `inc.zoomCount` 小于或等于 `inc.tripCount`。

The `findObject` function is called many times with various inputs. Here is a possible output we may observe after running the instrumented program:

`findObject` 函数会被多次调用，并传入不同的输入。以下是运行添加检测代码的程序后可能观察到的输出：

```
// [tripCount][zoomCount]: occurences
[7][6]:  2
[7][5]:  6
[7][4]:  20
[7][3]:  156
[7][2]:  967
[7][1]:  3685
[7][0]:  251004
[6][5]:  2
[6][4]:  7
[6][3]:  39
[6][2]:  300
[6][1]:  1235
[6][0]:  91731
[5][4]:  9
[5][3]:  32
[5][2]:  160
[5][1]:  764
[5][0]:  34142
...
```

The first number in the square bracket is the trip count of the loop, and the second is the number of `zoomIn`s we made within the same loop. The number after the column sign is the number of occurrences of that particular combination of the numbers. For example, two times we observed 7 loop iterations and 6 `zoomIn`s. 251004 times the loop ran 7 iterations and no `zoomIn`s, and so on. You can then plot the data for better visualization, or employ some other statistical methods, but the main point we can make is that `zoomIn`s are not frequent. 

方括号中的第一个数字是循环的次数，第二个数字是同一循环中执行的 `zoomIn` 操作次数。列号后面的数字表示该特定数字组合出现的次数。例如，我们两次观察到循环迭代7次，执行了6次 `zoomIn` 操作。循环迭代7次但没有执行 `zoomIn` 操作的情况出现了251004次，以此类推。您可以绘制数据图以更好地可视化数据，或使用其他统计方法，但我们主要想说明的是，`zoomIn` 操作并不频繁。

The total number of calls to `findObject` is approximately 400k; we can calculate it by summing up all the buckets in the histogram. If we sum up all the buckets with a non-zero `zoomCount`, we get approximately 10k; this is the number of times the `zoomIn` function was called. So, for every `zoomIn` call, we make 40 calls of the `findObject` function.

对 `findObject` 函数的总调用次数约为40万次；我们可以通过对直方图中的所有桶求和来计算。如果我们对所有 `zoomCount` 值非零的桶求和，则得到约1万次；这就是 `zoomIn` 函数的调用次数。因此，每次调用 `zoomIn` 函数时，我们都会调用40次 `findObject` 函数。

Later chapters of this book contain many examples of how such information can be used for optimizations. In our case, we conclude that `findObj` often fails to find the object. It means that the next iteration of the loop will try to find the object using new coordinates but still within the same search area. Knowing that, we could attempt a number of optimizations: 1) run multiple searches in parallel, and synchronize if any of them succeeded; 2) precompute certain things for the current search region, thus eliminating repetitive work inside `findObj`; 3) write a software pipeline that calls `getNewCoords` to generate the next set of required coordinates and prefetch the corresponding map locations from memory. Part 2 of this book looks more deeply into some of these techniques.

本书后续章节包含许多示例，说明如何利用此类信息进行优化。在我们的例子中，我们得出结论：`findObj` 函数经常找不到目标对象。这意味着循环的下一次迭代将尝试使用新的坐标在同一搜索区域内查找目标对象。基于此，我们可以尝试以下几种优化方法：1）并行运行多个搜索，并在其中任何一个搜索成功时进行同步；2）预先计算当前搜索区域的某些信息，从而消除 `findObj` 函数内部的重复工作；3）编写一个软件流水线，调用 `getNewCoords` 函数生成下一组所需的坐标，并从内存中预取相应的地图位置。本书第二部分将更深入地探讨其中的一些技术。

Code instrumentation provides very detailed information when you need specific knowledge about the execution of a program. It allows us to track any information about every variable in a program. Using such a method often yields the best insight when optimizing big pieces of code because you can use a top-down approach (instrumenting the main function and then drilling down to its callees) to better understand the behavior of an application. Code instrumentation enables developers to observe the architecture and flow of an application. This technique is especially helpful for someone working with an unfamiliar codebase.

代码插桩可以在您需要了解程序执行的具体信息时提供非常详细的信息。它允许我们跟踪程序中每个变量的任何信息。在优化大型代码时，使用这种方法通常能获得最佳洞察，因为您可以采用自顶向下的方法（先对主函数进行插桩，然后逐级深入到其调用者）来更好地理解应用程序的行为。代码插桩使开发人员能够观察应用程序的架构和流程。对于处理不熟悉的代码库的人来说，这项技术尤其有用。

The code instrumentation technique is heavily used in performance analysis of real-time scenarios, such as video games and embedded development. Some profilers combine instrumentation with other techniques such as tracing or sampling. We will look at one such hybrid profiler called Tracy in [@sec:Tracy].

代码插桩技术广泛应用于实时场景的性能分析，例如：视频游戏和嵌入式开发。一些性能分析器会将插桩与其他技术（例如跟踪或采样）相结合。我们将在[@sec:Tracy]中介绍一款名为Tracy的混合性能分析器。

While code instrumentation is powerful in many cases, it does not provide any information about how code executes from the OS or CPU perspective. For example, it can't give you information about how often the process was scheduled in and out of execution (known by the OS) or how many branch mispredictions occurred (known by the CPU). Instrumented code is a part of an application and has the same privileges as the application itself. It runs in userspace and doesn't have access to the kernel.

虽然代码插桩在很多情况下都很强大，但它无法提供任何关于代码如何从操作系统或CPU角度执行的信息。例如，它无法提供进程被调度进出执行的频率（操作系统知道）或发生了多少次分支预测错误（CPU知道）。被插桩的代码是应用程序的一部分，拥有与应用程序本身相同的权限。它运行在用户空间，无法访问内核。

A more important downside of this technique is that every time something new needs to be instrumented, say another variable, recompilation is required. This can become a burden and increase analysis time. Unfortunately, there are additional downsides. Since you usually care about hot paths in the application, you're instrumenting the things that reside in the performance-critical part of the code. Injecting instrumentation code in a hot path might easily result in a 2x slowdown of the overall benchmark. Remember not to benchmark an instrumented program. By instrumenting the code, you change the behavior of the program, so you might not see the same effects you saw earlier.

这项技术更重要的缺点是，每次需要添加新的代码插桩（例如：添加一个变量）时，都需要重新编译。这会成为一种负担，并增加分析时间。不幸的是，还有其他缺点。由于您通常关注应用程序中的热点路径，因此您插桩的对象是代码中性能关键部分的内容。在热点路径中注入插桩代码很容易导致整体基准测试速度降低2倍。请记住，不要对已插桩的程序进行基准测试。通过插桩代码，你改变了程序的行为，因此可能看不到之前观察到的相同效果。

All of the above increases the time between experiments and consumes more development time, which is why engineers don't manually instrument their code very often these days. However, automated code instrumentation is still widely used by compilers. Compilers are capable of automatically instrumenting an entire program (except third-party libraries) to collect interesting statistics about the execution. The most widely known use cases for automated instrumentation are code coverage analysis and Profile-Guided Optimization (see [@sec:secPGO]).

以上所有因素都会增加实验之间的间隔时间，并消耗更多的开发时间，这就是为什么如今工程师很少手动插桩代码的原因。然而，编译器仍然广泛使用自动代码插桩。编译器能够自动插桩整个程序（第三方库除外），以收集有关执行情况的有用统计信息。自动化插桩最广为人知的应用场景是代码覆盖率分析和基于性能分析的优化（参见 [@sec:secPGO]）。

When talking about instrumentation, it's important to mention *binary instrumentation* techniques. The idea behind binary instrumentation is similar but it is done on an already-built executable file rather than on source code. There are two types of binary instrumentation: static (done ahead of time) and dynamic (instrumented code is inserted on-demand as a program executes). The main advantage of dynamic binary instrumentation is that it does not require program recompilation and relinking. Also, with dynamic instrumentation, one can limit the amount of instrumentation to only interesting code regions, instead of instrumenting the entire program.

谈到插桩，必须提及*二进制插桩*技术。二进制插桩的原理与源代码插桩类似，但它是在已编译的可执行文件上进行插桩，而不是在源代码上进行插桩。二进制插桩有两种类型：静态插桩（预先完成）和动态插桩（在程序执行过程中按需插入插桩代码）。动态二进制插桩的主要优势在于无需重新编译和链接程序。此外，使用动态插桩，可以将插桩范围限制在感兴趣的代码区域，而不是对整个程序进行插桩。

Binary instrumentation is very useful in performance analysis and debugging. One of the most popular tools for binary instrumentation is the Intel Pin[^1] tool. Pin intercepts the execution of a program at the occurrence of an interesting event and generates new instrumented code starting at this point in the program. This enables the collection of various runtime information. One of the most popular tools that is built on top of Pin is Intel SDE (Software Development Emulator).[^2] Another well-known binary instrumentation tool is called DynamoRIO.[^4] Here are some of the things you can collect using a binary instrumentation tool:

二进制插桩在性能分析和调试中非常有用。Intel Pin[^1]工具是目前最流行的二进制插桩工具之一。Pin会在程序执行到某个感兴趣的事件时进行拦截，并从该点开始生成新的插桩代码。这使得我们可以收集各种运行时信息。基于 Pin 建的最流行的工具之一是Intel的软件开发模拟器（SDE: Software Development Emulator）[^2]。另一个知名的二进制插桩工具是DynamoRIO[^4]。以下是你可以使用二进制插桩工具收集的一些信息：

* instruction count and function call counts,
* instruction mix analysis,
* intercepting function calls and execution of any instruction in an application,
* memory intensity and footprint (see [@sec:MemoryIntensityFootprint]).

* 指令计数和函数调用计数，
* 指令混合分析，
* 拦截应用程序中的函数调用和任何指令的执行，
* 内存占用和内存占用量（参见 [@sec:MemoryIntensityFootprint]）。

Like code instrumentation, binary instrumentation only instruments user-level code and can be very slow.

与代码插桩类似，二进制插桩仅对用户级代码进行插桩，因此速度可能非常慢。

[^1]: Pin - [https://software.intel.com/en-us/articles/pin-a-dynamic-binary-instrumentation-tool](https://software.intel.com/en-us/articles/pin-a-dynamic-binary-instrumentation-tool)
[^2]: Intel SDE - [https://www.intel.com/content/www/us/en/developer/articles/tool/software-development-emulator.html](https://www.intel.com/content/www/us/en/developer/articles/tool/software-development-emulator.html)
[^3]: I have a slightly more advanced version of this code which I usually copy-paste into whatever project I'm working on, and later delete. 我有一个稍微高级一点的版本，通常会复制粘贴到我正在做的项目中，之后再删除。
[^4]: DynamoRIO - [https://github.com/DynamoRIO/dynamorio](https://github.com/DynamoRIO/dynamorio). It supports Linux and Windows operating systems, and runs on x86 and ARM hardware. 它支持Linux和Windows操作系统，可在x86和ARM硬件上运行。
