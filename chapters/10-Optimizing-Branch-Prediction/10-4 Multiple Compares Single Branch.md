## Multiple Tests Single Branch 单个分支多重测试 {#sec:MultipleCmpSingleBranch}

The last technique that we discuss in this chapter aims at minimizing the dynamic number of branch instructions by combining multiple tests. The main idea here is to avoid executing a branch for every element of a large array. Instead, the goal is to perform multiple tests simultaneously, which primarily involves using SIMD instructions. The result of multiple tests is a vector mask that can be converted into a byte mask, which often can be processed with a single branch instruction. This enables us to eliminate many branch instructions as you will see shortly. You may encounter this technique being used in SIMD implementations of various algorithms such as JSON/HTML parsing, media codecs, and others.

本章讨论的最后一种技术旨在通过组合多个测试来最小化动态分支指令的数量。其主要思想是避免对大型数组中的每个元素都执行分支操作。相反，目标是同时执行多个测试，这主要涉及使用SIMD指令。多个测试的结果是一个向量掩码，可以将其转换为字节掩码，而字节掩码通常可以用单个分支指令处理。正如你稍后将看到的，这使我们能够消除许多分支指令。您可能会在各种算法的SIMD实现中遇到这种技术，例如：JSON/HTML解析、媒体编解码器等。

[@lst:LongestLineNaive] shows a function that finds the longest line in an input string by testing one character at a time. We go through the input string and search for end-of-line (`eol`) characters (`\n`, 0x0A in ASCII). For every found `eol` character we check if the current line is the longest, and reset the length of the current line to zero. This code will execute one branch instruction for every character.[^1]

[@lst:LongestLineNaive] 展示了一个函数，它通过一次测试一个字符来查找输入字符串中的最长行。我们遍历输入字符串并搜索行尾字符（`eol`，也就是`\n`，ASCII码为0x0A）。对于每个找到的 `eol` 字符，我们检查当前行是否为最长行，如果是，则将当前行的长度重置为零。这段代码会为每个字符执行一条分支指令。[^1]

Listing: Find the longest line (one character at a time).
代码列表：寻找最长的行（一次1个字符）

~~~~ {#lst:LongestLineNaive .cpp}
uint32_t longestLine(const std::string &str) {
  uint32_t maxLen = 0;
  uint32_t curLen = 0;
  for (auto s : str) {
    if (s == '\n') {
      maxLen = std::max(curLen, maxLen);
      curLen = 0;
    } else {
      curLen++;
    }
  }
  // if no end-of-line in the end
  maxLen = std::max(curLen, maxLen);
  return maxLen;
}
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Consider the alternative implementation shown in [@lst:LongestLineSIMD] that tests eight characters at a time. You will typically see this idea implemented using compiler intrinsics (see [@sec:secIntrinsics]), however, I decided to show a standard C++ code for clarity. This exact case is featured in one of Performance Ninja's lab assignments,[^2] so you can try writing SIMD code yourself. Keep in mind, that the code I'm showing is incomplete as it misses a few corner cases; I provide it just to illustrate the idea.

考虑一下 [@lst:LongestLineSIMD] 中所示的另一种实现方式，它每次测试8个字符。通常情况下，这种思路会使用编译器内部函数来实现（参见 [@sec:secIntrinsics]），但为了清晰起见，我决定展示一段标准的C++代码。“性能忍者Performance Ninja”的一个实验作业[^2]中就包含了这种情况，所以你可以尝试自己编写SIMD代码。请注意，我展示的代码并不完整，因为它缺少一些边界情况；我提供它只是为了说明这个思路。

Listing: Find the longest line (8 characters at a time).
代码列表：寻找最长的行（一次8个字符）

~~~~ {#lst:LongestLineSIMD .cpp .numberLines}
uint32_t longestLine(const std::string &str) {
  uint32_t maxLen = 0;
  const uint64_t eol = 0x0a0a0a0a0a0a0a0a;
  auto *buf = str.data();
  uint32_t lineBeginPos = 0;
  for (uint32_t pos = 0; pos + 7 < str.size(); pos += 8) {
    // Load 8-byte chunk of the input string.
    uint64_t vect = *((const uint64_t*)(buf + pos));
    // Check all characters in this chunk.
    uint8_t mask = compareBytes(vect, eol);
    while (mask) {
      uint16_t eolPos = tzcnt(mask);
      // Compute the length of the current string.
      uint32_t curLen = (pos - lineBeginPos) + eolPos;
      // New line starts with the character after '\n'
      lineBeginPos += curLen + 1;
      // Is this line the longest?
      maxLen = std::max(curLen, maxLen);
      // Shift the mask to check if we have more '\n'
      mask >>= eolPos + 1;
    }
  }
  // process remainder (not shown)
  return maxLen;
}

uint8_t compareBytes(uint64_t a, uint64_t b) {
  // Perform a byte-wise comparison of a and b.
  // Produce a bit mask with the result of comparisons:
  // one if bytes are equal, zero if different.
}

uint8_t tzcnt(uint8_t mask) {
  // Count the number of trailing zero bits in the mask.
}
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

We start by preparing an 8-byte mask filled with `eol` symbols. The inner loop loads eight characters of the input string and performs a byte-wise comparison of these characters with the `eol` mask. Vectors in modern processors contain 16/32/64 bytes, so we can process even more characters simultaneously. The result of the eight comparisons is an 8-bit mask with either 0 or 1 in the corresponding position (see `compareBytes`). For example, when comparing `0x00FF0A000AFFFF00` and `0x0A0A0A0A0A0A0A0A`, we will get `0b00101000` as a result. With x86 and ARM ISAs, the function `compareBytes` can be implemented using two vector instructions.[^4]

我们首先准备一个填充了换行符（`eol`）的8字节掩码。内部循环加载输入字符串的8个字符，并将这些字符与换行符掩码进行逐字节比较。现代处理器中的向量包含16/32/64字节，因此我们可以同时处理更多字符。8次比较的结果是一个8位掩码，其对应位置为0或1（参见 `compareBytes`）。例如，比较 `0x00FF0A000AFFFF00` 和 `0x0A0A0A0A0A0A0A0A`，结果为 `0b00101000`。在x86和ARM指令集体系结构中，可以使用两条向量指令来实现 `compareBytes` 函数[^4]。

If the mask is zero, that means there are no `eol` characters in the current chunk and we can skip it (see line 11). This is a critical optimization that provides large speedups for input strings with long lines. If a mask is not zero, that means there are `eol` characters and we need to find their positions. To do so, we use the `tzcnt` function, which counts the number of trailing zero bits in an 8-bit mask (the position of the rightmost set bit). For example, for the mask `0b00101000`, it will return 3. Most ISAs support implementing the `tzcnt` function with a single instruction.[^3] Line 14 calculates the length of the current line using the result of the `tzcnt` function. We shift right the mask and repeat until there are no set bits in the mask.

如果掩码为零，则表示当前数据块中没有换行符，我们可以跳过它（参见第11行）。这是一项关键的优化，可以显著提升处理长行输入字符串的速度。如果掩码不为零，则表示存在 `eol` 字符，我们需要找到它们的位置。为此，我们使用 `tzcnt` 函数，该函数计算8位掩码中尾随零位的数量（即最右侧置位的位置）。例如，对于掩码`0b00101000`，它将返回3。大多数指令集体系结构(ISA)都支持使用单条指令实现 `tzcnt` 函数[^3]。第14行使用 `tzcnt` 函数的结果计算当前行的长度。我们将掩码右移，并重复此过程，直到掩码中不再有置位为止。

For an input string with a single very long line (best case scenario), the SIMD version will execute eight times fewer branch instructions. However, in the worst-case scenario with zero-length lines (i.e., only `eol` characters in the input string), the original approach is faster. I benchmarked this technique using AVX2 implementation (with chunks of 16 characters) on several different inputs, including textbooks, and source code files. The result was 5--6 times fewer branch instructions and more than 4x better performance when running on Intel Core i7-1260P (12th Gen, Alder Lake).

对于包含单个长行的输入字符串（最佳情况），SIMD版本执行的分支指令数量将减少8倍。然而，在最坏情况下，如果输入字符串中只有零长度的行（即只有 `eol` 字符），则原始方法速度更快。我使用AVX2实现（以16个字符为一组）对这项技术进行了基准测试，测试对象包括教科书和源代码文件等多种不同的输入。结果表明，在Intel Core i7-1260P（第12代，Alder Lake）处理器上运行时，分支指令数量减少了5到6倍，性能提升了4倍以上。

[^1]: Assuming that the compiler will avoid generating branch instructions for `std::max`. 假设编译器会避免为 `std::max` 生成分支指令。
[^2]: Performance Ninja: compiler intrinsics 2  性能忍者课程：编译器内部函数2- [https://github.com/dendibakh/perf-ninja/tree/main/labs/core_bound/compiler_intrinsics_2](https://github.com/dendibakh/perf-ninja/tree/main/labs/core_bound/compiler_intrinsics_2).
[^3]: Although in x86, there is no version of the `TZCNT` instruction that supports 8-bit inputs. 尽管在x86体系结构中，没有支持8位输入的 `TZCNT` 指令版本。
[^4]: For example, with AVX2 (256-bit vectors), you can use `VPCMPEQB` and `VPMOVMSKB` instructions. 例如，对于AVX2（256位向量），你可以使用 `VPCMPEQB` 和 `VPMOVMSKB` 指令。
