## Inlining Functions 内联函数

If you're one of those developers who frequently looks into assembly code, you have probably seen `CALL`, `PUSH`, `POP`, and `RET` instructions. In x86 ISA, `CALL` and `RET` instructions are used to call and return from a function. `PUSH` and `POP` instructions are used to save a register value on the stack and restore it.

如果您是经常查看汇编代码的开发人员之一，您可能见过 `CALL`、`PUSH`、`POP` 和 `RET` 指令。在x86指令集体系结构中，`CALL` 和 `RET` 指令用于调用函数和从函数返回。`PUSH` 和 `POP` 指令用于将寄存器值保存到栈中，并在需要时将其恢复。

The nuances of a function call are described by the *calling convention*: how arguments are passed and in what order, how the result is returned, which registers the called function must preserve, and how the work is split between the caller and the callee. Based on a calling convention, when a caller makes a function call, it expects that some registers will hold the same values after the callee returns. Thus, if a callee needs to change one of the registers that should be preserved, it needs to save (`PUSH`) and restore (`POP`) them before returning to the caller. A series of `PUSH` instructions is called a *prologue*, and a series of `POP` instructions is called an *epilogue*.

函数调用的细微差别由*调用约定（calling convention）*来描述：参数的传递方式和顺序、结果的返回方式、被调用函数必须保留的寄存器，以及调用者和被调用者之间的工作分配。根据调用约定，当调用者进行函数调用时，它期望某些寄存器在被调用者返回后保持相同的值。因此，如果被调用者需要更改某个应该保留的寄存器，则需要在返回给调用者之前保存（`PUSH`）和恢复（`POP`）这些寄存器的值。一系列 `PUSH` 指令称为*序言prologue*，一系列 `POP` 指令称为*尾声epilogue*。

When a function is small, the overhead of calling a function (prologue and epilogue) can be very pronounced. This overhead can be eliminated by inlining a function body into the place where it is called. Function inlining is a process of replacing a call to function `foo` with the code for `foo` specialized with the actual arguments of the call. Inlining is one of the most important compiler optimizations. Not only because it eliminates the overhead of calling a function, but also because it enables other optimizations. This happens because when a compiler inlines a function, the scope of compiler analysis widens to a much larger chunk of code. However, there are disadvantages as well: inlining can potentially increase code size and compile time.[^20]

当函数很小时，调用函数（序言和尾声）的开销可能非常显著。通过将函数体内联到调用位置，可以消除这种开销。函数内联是指将对函数 `foo` 的调用替换为使用实际参数对 `foo` 进行特例化的代码。内联是编译器最重要的优化之一。它不仅消除了函数调用的开销，而且还支持其他优化。这是因为当编译器内联函数时，编译器分析的范围会扩展到更大的代码块。然而，内联也有缺点：它可能会增加代码大小和编译时间[^20]。

The primary mechanism for function inlining in many compilers relies on a cost model. For example, in the LLVM compiler, function inlining is based on computing a cost for each function *call site*. A call site is a place in the code where a function is called. The cost of inlining a function call is based on the number and type of instructions in that function. Inlining happens if the cost is less than a threshold, which is usually a fixed number; however, it can be varied under certain circumstances.[^21] In addition to the generic cost model, many heuristics can overwrite cost model decisions in some cases. For instance: 

许多编译器中函数内联的主要机制依赖于成本模型。例如，在LLVM编译器中，函数内联基于计算每个函数*调用点（call site）*的成本。调用点是指代码中调用函数的位置。函数调用的内联成本取决于该函数中指令的数量和类型。当成本低于某个阈值时，就会进行内联，该阈值通常是一个固定值；但在某些情况下，它可以改变。[^21] 除了通用的成本模型之外，许多启发式方法在某些情况下可以覆盖成本模型的决策。例如：

* Tiny functions (wrappers) are almost always inlined.
* Functions with a single call site are preferred candidates for inlining.
* Large functions usually are not inlined as they bloat the code of the caller function.

* 小型函数（包装函数wrappers）几乎总是被内联。
* 只有一个调用点的函数是内联的首选。
* 大型函数通常不会被内联，因为它们会使调用函数的代码变得臃肿。

Also, there are situations when inlining is problematic:

此外，在某些情况下，内联会存在问题：

* A recursive function cannot be inlined into itself unless it's a tail-recursive function (see next section). Also, if the depth of recursion is usually small, it's possible to partially inline a recursive function, i.e., inline a body of a recursive function to itself a couple of times, and then leave a recursive call as before. This may eliminate the overhead of a function call if the recursive call depth is usually small.
* 递归函数不能内联到自身，除非它是尾递归函数（参见下一节）。此外，如果递归深度通常较小，可以对递归函数进行部分内联，即对递归函数体进行几次内联，然后保留之前的递归调用。如果递归调用深度通常较小，这可以消除函数调用的开销。
* A function that is referred to through a pointer can be inlined in place of a direct call but the function has to remain in the binary, i.e., it cannot be fully inlined and eliminated. The same is true for functions with external linkage.
* 通过指针引用的函数可以代替直接调用进行内联，但该函数必须保留在二进制文件中，也就是说，它不能被完全内联并消除。对于外部链接的函数也是如此。

As I wrote earlier, compilers tend to use a cost model approach when deciding about inlining a function, which typically works well in practice. In general, it is a good strategy to rely on the compiler for making all the inlining decisions and adjusting if needed. The cost model cannot account for every possible situation, which leaves room for improvement. Sometimes compilers require special hints from the developer. One way to find potential candidates for inlining in a program is by looking at the profiling data, and in particular, how hot is the prologue and the epilogue of the function. [@lst:FuncInlining] is an example of a function profile with a prologue and epilogue consuming `~50%` of the function time:

正如我之前提到的，编译器在决定是否内联函数时倾向于使用成本模型方法，这在实践中通常效果良好。一般来说，依靠编译器做出所有内联决策并在必要时进行调整是一个不错的策略。成本模型无法涵盖所有​​可能的情况，因此仍有改进的空间。有时，编译器需要开发者提供一些特殊提示。寻找程序中潜在内联候选函数的一种方法是查看性能分析数据，尤其要关注函数的序言和尾声的执行效率。[@lst:FuncInlining] 就是一个函数性能分析示例，其序言和尾声消耗了约50%的函数执行时间：

Listing: A profile of function `foo` which has a hot prologue and epilogue
代码列表：函数 `foo` 的轮廓性能分析，其序言和尾声执行很热（大量的 `push` 和 `pop`）

~~~~ {#lst:FuncInlining .cpp}
Overhead |  Source code & Disassembly
   (%)   |  of function `foo`
--------------------------------------------
    3.77 :  418be0:  push   r15	     # prologue
    4.62 :  418be2:  mov    r15d,0x64
    2.14 :  418be8:  push   r14
    1.34 :  418bea:  mov    r14,rsi
    3.43 :  418bed:  push   r13
    3.08 :  418bef:  mov    r13,rdi
    1.24 :  418bf2:  push   r12
    1.14 :  418bf4:  mov    r12,rcx
    3.08 :  418bf7:  push   rbp
    3.43 :  418bf8:  mov    rbp,rdx
    1.94 :  418bfb:  push   rbx
    0.50 :  418bfc:  sub    rsp,0x8
    ...
    # function body
    ...
    4.17 :  418d43:  add    rsp,0x8  # epilogue
    3.67 :  418d47:  pop    rbx
    0.35 :  418d48:  pop    rbp
    0.94 :  418d49:  pop    r12
    4.72 :  418d4b:  pop    r13
    4.12 :  418d4d:  pop    r14
    0.00 :  418d4f:  pop    r15
    1.59 :  418d51:  ret
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

When you see hot `PUSH` and `POP` instructions, this might be a strong indicator that the time consumed by the prologue and epilogue of the function might be saved if we inline the function. Note that even if the prologue and epilogue are hot, it doesn't necessarily mean it will be profitable to inline the function. Inlining triggers a lot of different changes, so it's hard to predict the outcome. Always measure the performance of the changed code before forcing a compiler to inline a function.

当您看到热门的 `PUSH` 和 `POP` 指令时，这可能强烈表明，如果我们内联该函数，函数的序言和尾声所消耗的时间可能会被节省。请注意，即使序言和尾声是热点代码，也并不一定意味着内联该函数就能带来收益。内联会触发许多不同的变化，因此很难预测结果。在强制编译器内联函数之前，务必先测试修改后代码的性能。

For the GCC and Clang compilers, you can make a hint for inlining `foo` with the help of a C++11 `[[gnu::always_inline]]` attribute as shown in the code example below. With earlier C++ standards you can use `__attribute__((always_inline))`. For the MSVC compiler, you can use the `__forceinline` keyword.

对于GCC和Clang编译器，您可以使用C++11 的 `[[gnu::always_inline]]` 属性来提示内联 `foo`，如下面的代码示例所示。对于早期的C++标准，您可以使用 `__attribute__((always_inline))`。对于MSVC编译器，您可以使用 `__forceinline` 关键字。

```cpp
[[gnu::always_inline]] int foo() {
    // foo body
}
```

### Tail Call Optimization 尾部调用优化

In a tail-recursive function, the recursive call is the last operation performed by the function before it returns its result. A simple example is demonstrated [@lst:TailCall]. In the original code, the `sum` function recursively accumulates integer numbers from 0 to `n`, for example, a call of `sum(5,0)` will yield `5+4+3+2+1`, which gives 15.

在尾递归函数中，递归调用是函数返回结果之前执行的最后一个操作。演示了一个简单的示例 [@lst:TailCall]。在原始代码中，“sum”函数递归地累加从 0 到 `n` 的整数，例如，调用“sum(5,0)”将产生“5+4+3+2+1”，即15。

If we compile the original code without optimizations (`-O0`), compilers will generate assembly code that has a recursive call. This is very inefficient due to the overhead of the function call. Moreover, if you call `sum` with a large `n`, the program will create a large number of stack frames on top of each other. There is a high chance that it will result in a stack overflow since the stack memory is limited.

如果我们在没有优化的情况下编译原始代码（`-O0`），编译器将生成具有递归调用的汇编代码。由于函数调用的开销，这是非常低效的。此外，如果您使用较大的“n”调用“sum”，程序将创建大量彼此叠加的栈帧。由于栈内存有限，很有可能导致堆栈溢出。

When you apply optimizations, e.g., `-O2`, to the example in [@lst:TailCall], compilers will recognize an opportunity for tail call optimization. The transformation will reuse the current stack frame instead of recursively creating new frames. To do so, the compiler flushes the current frame and replaces the `call` instruction with a `jmp` to the beginning of the function. Just like inlining, tail call optimization provides room for further optimizations. So, later, the compiler can apply more transformations to replace the original version with an iterative version shown on the right. For example, GCC 13.2 generates identical machine code for both versions.

当你对 [@lst:TailCall] 中的示例应用优化（例如“-O2”）时，编译器将识别尾调用优化的机会。该转换将重用当前的栈帧，而不是递归地创建新帧。为此，编译器刷新当前帧，并将“call”指令替换为“jmp”到函数开头。就像内联一样，尾调用优化为进一步优化提供了空间。因此，稍后，编译器可以应用更多转换，将原始版本替换为右侧所示的迭代版本。例如，GCC 13.2为两个版本生成相同的机器代码。

Listing: Tail Call Compiler Optimization
代码列表：尾调用编译器优化
		
~~~~ {#lst:TailCall .cpp}
// original code                         // compiler intermediate transformation
int sum(int n, int acc) {                int sum(int n, int acc) {
  if (n == 0) {                            for (int i = n; i > 0; --i) {
    return acc;                    =>        acc += i;
  } else {                                 }
    return sum(n - 1, acc + n);            return acc;
  }                                      }
}
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Like with any compiler optimization, there are cases when it cannot perform the code transformation you want. If you are using the Clang compiler, and you want guaranteed tail call optimizations, you can mark a `return` statement with `__attribute__((musttail))`. This indicates that the compiler must generate a tail call for the program to be correct, even when optimizations are disabled. One example, where it is beneficial is language interpreter loops.[^22] In case of doubt, it is better to use an iterative version instead of tail recursion and leave tail recursion to functional programming languages.

与其他编译器优化一样，有时它无法执行你想要的代码转换。如果你使用的是Clang编译器，并且想要保证尾调用优化，你可以使用 `__attribute__((musttail))` 标记 `return` 语句。这表明即使优化被禁用，编译器也必须生成一个尾调用才能保证程序正确运行。例如，语言解释器循环就非常适合这种做法[^22]。如有疑问，最好使用迭代版本而不是尾递归，并将尾递归留给函数式编程语言。

[^20]: See the article: 参见文章： [https://aras-p.info/blog/2017/10/09/Forced-Inlining-Might-Be-Slow/](https://aras-p.info/blog/2017/10/09/Forced-Inlining-Might-Be-Slow/).
[^21]: For example: 1) when a function declaration has a hint for inlining; 2) when there is profiling data for the function; or 3) when a compiler optimizes for size (`-Os`) rather than performance (`-O2`). 例如：1）当函数声明包含内联提示时；2）当存在函数的轮廓性能分析数据时；或3）当编译器优化的是大小（`-Os`）而不是性能（`-O2`）时。
[^22]: Josh Haberman's blog: motivation for guaranteed tail calls - [https://blog.reverberate.org/2021/04/21/musttail-efficient-interpreters.html](https://blog.reverberate.org/2021/04/21/musttail-efficient-interpreters.html).
