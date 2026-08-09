## Replace Branches with Lookup 用查找表替换分支

One way to avoid frequently mispredicted branches is to use lookup tables. An example of code when such transformation might be profitable is shown in [@lst:LookupBranches]. As usual, the original version is on the left while the improved version is on the right. Function `mapToBucket` maps values in the `[0-50)` range into corresponding five buckets, and returns `-1` for values that are out of this range. For uniformly distributed values of `v`, we will have an equal probability for `v` to fall into any of the buckets. In the generated assembly for the original version, we will likely see many branches, which could have high misprediction rates. Hopefully, it's possible to rewrite the function `mapToBucket` using a single array lookup, as shown on the right.

避免频繁出现分支预测错误的一种方法是使用查找表。[@lst:LookupBranches] 中展示了一个使用这种转换可能获益的代码示例。和往常一样，左侧是原始版本，右侧是改进版本。函数 `mapToBucket` 将 `[0-50)` 范围内的值映射到相应的五个桶buckets中，对于超出此范围的值返回 `-1`。对于均匀分布的 `v` 值，`v` 落入任何一个桶的概率相等。在原始版本生成的汇编代码中，我们可能会看到许多分支，这些分支的预测错误率可能很高。希望能够像右侧所示那样，使用单个数组查找来重写 `mapToBucket` 函数。

Listing: Replacing branches with lookup tables.
代码列表：用查找表替换分支。

~~~~ {#lst:LookupBranches .cpp}
int8_t mapToBucket(unsigned v) {       int8_t buckets[50] = {
  if      (v < 10) return 0;             0, 0, 0, 0, 0, 0, 0, 0, 0, 0,
  else if (v < 20) return 1;             1, 1, 1, 1, 1, 1, 1, 1, 1, 1,
  else if (v < 30) return 2;      =>     2, 2, 2, 2, 2, 2, 2, 2, 2, 2,
  else if (v < 40) return 3;             3, 3, 3, 3, 3, 3, 3, 3, 3, 3,
  else if (v < 50) return 4;             4, 4, 4, 4, 4, 4, 4, 4, 4, 4 };
  return -1;
}                                      int8_t mapToBucket(unsigned v) {
                                         if (v < (sizeof(buckets) / sizeof(int8_t)))
                                           return buckets[v];
                                         return -1;
                                       }
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

For the improved version of `mapToBucket` on the right, a compiler will likely generate a single branch instruction that guards against out-of-bounds access to the `buckets` array. A typical hot path through this function will execute the untaken branch and one load instruction. The branch will be well-predicted by the CPU branch predictor since we expect most of the input values to fall into the range covered by the `buckets` array. The lookup will also be fast since the `buckets` array is small and likely to be in the L1 D-cache.

对于右侧改进后的 `mapToBucket` 函数，编译器很可能会生成一条分支指令，以防止对 `buckets` 数组进行越界访问。该函数的典型热路径会执行未跳转的分支指令和一条加载指令。由于我们预期大多数输入值都落在 `buckets` 数组覆盖的范围内，因此CPU分支预测器能够准确预测该分支。此外，由于 `buckets` 数组很小且很可能位于L1数据缓存中，因此查找速度也会很快。

If we need to map a bigger range of values, say `[0-1M)`, allocating a very large array is not practical. In this case, we might use interval map data structures that accomplish that goal using much less memory but logarithmic lookup complexity. Readers can find existing implementations of interval map container in [Boost](https://www.boost.org/doc/libs/1_65_0/libs/icl/doc/html/boost/icl/interval_map.html)[^2] and [LLVM](https://llvm.org/doxygen/IntervalMap_8h_source.html)[^3].

如果我们需要映射更大的值范围，例如 `[0-1M)`，分配一个非常大的数组是不切实际的。在这种情况下，我们可以使用区间映射数据结构来实现该目标，它使用更少的内存，但查找复杂度为对数级。读者可以在 [Boost](https://www.boost.org/doc/libs/1_65_0/libs/icl/doc/html/boost/icl/interval_map.html)[^2] 和 [LLVM](https://llvm.org/doxygen/IntervalMap_8h_source.html)[^3] 中找到区间映射容器的现有实现。

[^2]: C++ Boost `interval_map` C++ Boost库的 `interval_map` 函数 - [https://www.boost.org/doc/libs/1_65_0/libs/icl/doc/html/boost/icl/interval_map.html](https://www.boost.org/doc/libs/1_65_0/libs/icl/doc/html/boost/icl/interval_map.html)
[^3]: LLVM's `IntervalMap` LLVM 的 `IntervalMap` 函数 - [https://llvm.org/doxygen/IntervalMap_8h_source.html](https://llvm.org/doxygen/IntervalMap_8h_source.html)
