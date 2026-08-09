## Replace Branches with Arithmetic 用算术运算替换分支

In some scenarios, branches can be replaced with arithmetic. The code in [@lst:LookupBranches] can also be rewritten using a simple arithmetic formula, as shown in [@lst:ArithmeticBranches]. Notice, that for this code, the Clang-17 compiler replaced expensive division with much cheaper multiplication and right shift operations.

在某些情况下，分支可以用算术运算替换。如 [@lst:ArithmeticBranches] 所示，[@lst:LookupBranches] 中的代码也可以使用简单的算术公式重写。请注意，对于这段代码，Clang-17编译器用更高效的乘法和右移操作替换了开销较大的除法运算。

Listing: Replacing branches with arithmetic.
代码列表：用算术运算替换分支。

~~~~ {#lst:ArithmeticBranches .cpp}
int8_t mapToBucket(unsigned v) {             │    mov al, -1
  constexpr unsigned BucketRangeMax = 50;    │    cmp edi, 49
  if (v < BucketRangeMax)                    │    ja .exit
    return v / 10;                           │    movzx eax, dil
  return -1;                                 │    imul eax, eax, 205
}                                            │    shr eax, 11
                                             │  .exit:
                                             │    ret
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

As of the year 2024, compilers are usually unable to find these shortcuts on their own, so it is up to the programmer to do it manually. If you can find a way to replace a frequently mispredicted branch with arithmetic, you will likely see a performance improvement.

截至2024年，编译器通常无法自行找到这些快捷方式，因此需要程序员手动操作。如果您能找到一种方法，用算术运算替换经常预测错误的跳转语句，则很可能会看到性能提升。
