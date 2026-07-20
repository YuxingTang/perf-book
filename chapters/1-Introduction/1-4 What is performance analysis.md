## What Is Performance Analysis? 什么是性能分析？

Have you ever found yourself debating with a coworker about the performance of a certain piece of code? Then you probably know how hard it is to predict which code is going to work the best. With so many moving parts inside modern processors, even small tweaks to code can trigger noticeable performance changes. Relying on intuition when optimizing an application typically results in random "fixes" without real performance impact.

你是否曾与同事争论过某段代码的性能？如果是，那你大概知道预测哪段代码运行效果最佳有多么困难。现代处理器内部组件众多，即使是代码的微小改动也能带来显著的性能变化。仅仅依靠直觉来优化应用程序通常会导致一些随机的“修复”，而这些修复实际上并没有真正提升性能。

Inexperienced developers sometimes make changes in their code and claim it *should* run faster. One such example is replacing `i++` (post-increment) with `++i` (pre-increment) all over the code base (assuming that the previous value of `i` is not used). In the general case, this change will make no difference to the generated code: every decent optimizing compiler will recognize that the previous value of `i` is not used and will eliminate redundant copies anyway. The first piece of advice in this book is: don't solely rely on your intuition. *Always measure.*

经验不足的开发者有时会修改代码，然后声称它*应该*运行得更快。例如，他们会在整个代码库中将 `i++`（后置递增）全部替换为 `++i`（前置递增）（假设 `i` 的先前值没有被使用）。通常情况下，这种更改对生成的代码没有任何影响：任何优秀的优化编译器都会识别出 `i` 的先前值没有被使用，并会自动删除冗余的复制。本书的第一条建议就是：不要仅仅依赖你的直觉。 *务必始终进行测量。*

Many micro-optimization tricks that circulate around the world were valid in the past, but current compilers have already learned them. Additionally, some people tend to overuse legacy bit-twiddling tricks. One such example is the XOR swap idiom.[^2] In reality, simple `std::swap` produces equivalent or faster code. Such accidental changes likely won’t improve the performance of an application. Finding the right place to tune should be the result of careful performance analysis, not intuition or guessing.

许多流传于世的微优化技巧在过去或许有效，但当前的编译器早已识别并修复了它们。此外，有些人倾向于过度使用一些过时的位操作技巧。例如，习惯的XOR交换算法使用[^2]。实际上，简单的 `std::swap` 就能生成等效甚至更快的代码。这种随意的改动不太可能提升应用程序的性能。找到合适的优化点应该是经过仔细的性能分析的结果，而不是凭直觉或猜测。

Performance analysis is a process of collecting information about how a program executes and interpreting it to find optimization opportunities. Any change that ends up being made in the source code of a program should be driven by analyzing and interpreting collected data. We will show you how to use performance analysis techniques to discover optimization opportunities even in a large and unfamiliar codebase. There are many performance analysis methodologies. Depending on the problem, some will be more efficient than others. With experience, you will develop your own strategies about when to use each approach.

性能分析是一个收集程序执行信息并进行解读以发现优化机会的过程。任何对程序源代码的修改都应该基于对收集数据的分析和解读。我们将向您展示如何使用性能分析技术，即使在庞大且陌生的代码库中也能发现优化机会。性能分析方法有很多种。根据问题的不同，有些方法会比其他方法更有效。随着经验的积累，您将逐渐形成自己的策略，了解何时使用哪种方法。

[^2]: XOR-based swap idiom 基于异或运算的交换算法 - [https://en.wikipedia.org/wiki/XOR_swap_algorithm](https://en.wikipedia.org/wiki/XOR_swap_algorithm)