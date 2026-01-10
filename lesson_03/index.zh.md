**FFmpeg 汇编语言第三课**

让我们解释更多的术语，并给你上一堂简短的历史课。

**指令集**

你可能已经在上一课中看到我们讨论了 SSE2，这是一套 SIMD 指令集。当新一代 CPU 发布时，它可能带有新的指令，有时会有更大的寄存器尺寸。x86 指令集的历史非常复杂，所以这是一个简化的历史（还有更多的子类）：

* MMX - 1997 年推出，Intel 处理器中的第一个 SIMD，64 位寄存器，具有历史意义
* SSE (Streaming SIMD Extensions) - 1999 年推出，128 位寄存器
* SSE2 - 2000 年推出，许多新指令
* SSE3 - 2004 年推出，第一批水平指令
* SSSE3 (Supplemental SSE3) - 2006 年推出，新指令，但最重要的是 pshufb 洗牌指令，可以说是视频处理中最重要的指令
* SSE4 - 2008 年推出，许多新指令，包括打包的最小值和最大值。
* AVX - 2011 年推出，256 位寄存器（仅浮点）和新的三操作数语法
* AVX2 - 2013 年推出，用于整数指令的 256 位寄存器
* AVX512 - 2017 年推出，512 位寄存器，新的操作掩码功能。由于使用新指令时 CPU 频率会降低，当时在 FFmpeg 中的使用有限。具有 vpermb 的全 512 位洗牌（排列）。
* AVX512ICL - 2019 年推出，不再有主频降低。
* AVX10 - 即将推出

值得注意的是，指令集既可以添加到 CPU 中，也可以从中移除。例如，AVX512 在第 12 代 Intel CPU 中被[移除](https://www.igorslab.de/en/intel-deactivated-avx-512-on-alder-lake-but-fully-questionable-interpretation-of-efficiency-news-editorial/)（有争议）。正是因为这个原因，FFmpeg 进行运行时 CPU 检测。FFmpeg 检测其运行所在的 CPU 的功能。

正如你在作业中看到的，函数指针默认是 C 语言的，并被特定的指令集变体替换。这意味着检测只做一次，以后就不需要再做了。这与许多专有应用程序形成鲜明对比，后者硬编码特定的指令集，使得功能完美的计算机被淘汰。这也允许在运行时打开/关闭优化函数。这是开源的一大好处。

像 FFmpeg 这样的程序在世界各地数十亿台设备上使用，其中一些可能非常古老。FFmpeg 技术上支持仅支持 SSE 的机器，这些机器已经有 25 年历史了！值得庆幸的是，如果你使用了特定指令集中不可用的指令，x86inc.asm 能够告诉你。

为了让你了解实际情况，以下是截至 2024 年 11 月来自 [Steam 调查](https://store.steampowered.com/hwsurvey/Steam-Hardware-Software-Survey-Welcome-to-Steam) 的指令集可用性（这显然偏向于游戏玩家）：

| 指令集 | 可用性 |
| :---- | :---- |
| SSE2 | 100% |
| SSE3 | 100% |
| SSSE3 | 99.86% |
| SSE4.1 | 99.80% |
| AVX | 97.39% |
| AVX2 | 94.44% |
| AVX512 (Steam 不区分 AVX512 和 AVX512ICL) | 14.09% |

对于像 FFmpeg 这样拥有数十亿用户的应用程序来说，即使是 0.1% 也是非常大的用户数量，如果出现问题会有大量的错误报告。FFmpeg 拥有广泛的测试基础设施，用于在我们的 [FATE 测试套件](https://fate.ffmpeg.org/?query=subarch:x86_64%2F%2F) 中测试 CPU/操作系统/编译器的各种变体。每一次提交都在数百台机器上运行，以确没有任何东西被破坏。

Intel 在这里提供了详细的指令集手册：[https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html)

在 PDF 中搜索可能很麻烦，所以这里有一个非官方的基于 Web 的替代方案：[https://www.felixcloutier.com/x86/](https://www.felixcloutier.com/x86/)

这里也有一个 SIMD 指令的可视化表示：
[https://www.officedaytime.com/simd512e/](https://www.officedaytime.com/simd512e/)

x86 汇编的部分挑战在于找到适合你需求的正确指令。在某些情况下，指令可以以非最初预期的方式使用。

**指针偏移技巧**

让我们回到第一课中的原始函数，但在 C 函数中添加一个宽度参数。

我们使用 ptrdiff_t 而不是 int 作为宽度变量，以确保 64 位参数的高 32 位为零。如果我们直接在函数签名中传递一个 int 宽度，然后尝试将其作为一个四字用于指针算术（即使用 `widthq`），寄存器的高 32 位可能会被任意值填充。我们可以通过使用 `movsxd` 对宽度进行符号扩展来解决这个问题（也可以参见 x86inc.asm 中的宏 `movsxdifnidn`），但这是一个更简单的方法。

下面的函数包含指针偏移技巧：

```assembly
;static void add_values(uint8_t *src, const uint8_t *src2, ptrdiff_t width)
INIT_XMM sse2
cglobal add_values, 3, 3, 2, src, src2, width
   add srcq, widthq
   add src2q, widthq
   neg widthq

.loop
    movu  m0, [srcq+widthq]
    movu  m1, [src2q+widthq]

    paddb m0, m1

    movu  [srcq+widthq], m0
    add   widthq, mmsize
    jl .loop

    RET
```

让我们一步步来，因为这可能很令人困惑：

```assembly
   add srcq, widthq
   add src2q, widthq
   neg widthq
```

宽度被加到每个指针上，使得每个指针现在都指向要处理的缓冲区的末尾。然后宽度被取反。

```assembly
    movu  m0, [srcq+widthq]
    movu  m1, [src2q+widthq]
```

然后以 widthq 为负值进行加载。所以在第一次迭代中 [srcq+widthq] 指向 srcq 的原始地址，即指回缓冲区的开头。

```assembly
    add   widthq, mmsize
    jl .loop
```

mmsize 被加到负的 widthq 上，使其更接近零。循环条件现在是 jl（如果小于零则跳转）。这个技巧意味着 widthq 同时被用作指针偏移量**和**循环计数器，节省了一条 cmp 指令。它还允许指针偏移量用于多次加载和存储，以及在需要时使用指针偏移量的倍数（在作业中记住这一点）。

**对齐**

在所有的例子中，我们一直使用 movu 来避开对齐的话题。许多 CPU 如果数据是对齐的，即如果内存地址可以被 SIMD 寄存器大小整除，则可以更快地加载和存储数据。在可能的情况下，我们尝试在 FFmpeg 中使用 mova 进行对齐的加载和存储。

在 FFmpeg 中，av_malloc 能够在堆上提供对齐的内存，而 DECLARE_ALIGNED C 预处理器指令可以在栈上提供对齐的内存。如果 mova 用于未对齐的地址，它将导致段错误 (segmentation fault) 并且应用程序将崩溃。确对齐值对应于 SIMD 寄存器大小也很重要，即 xmm 为 16，ymm 为 32，zmm 为 64。

以下是如何将 RODATA 段的开头对齐到 64 字节：

```assembly
SECTION_RODATA 64
```

注意这只是对齐 RODATA 的开头。可能需要填充字节以确保下一个标签保持在 64 字节边界上。

**范围扩展**

我们在之前避开的另一个话题是溢出。这发生在例如字节的值在加法或乘法等操作后超过 255 时。我们可能想要执行一个需要比字节更大的中间值（例如字）的操作，或者可能我们想要将数据保留在那个更大的中间尺寸中。

对于无符号字节，这就是 punpcklbw（打包解包低位字节到字）和 punpckhbw（打包解包高位字节到字）派上用场的地方。

让我们看看 punpcklbw 是如何工作的。Intel 手册中的 SSE2 版本的语法如下：

| PUNPCKLBW xmm1, xmm2/m128 |
| :---- |

这意味着它的源（右侧）可以是一个 xmm 寄存器或一个内存地址（m128 意味着具有标准 [base + scale*index + disp] 语法的内存地址），目标是一个 xmm 寄存器。

上面的 officedaytime.com 网站有一个很好的图表展示了发生了什么：

![What is this](image1.png)

你可以看到字节分别从每个寄存器的下半部分交错排列。但这与范围扩展有什么关系呢？如果 src 寄存器全为零，这将把 dst 中的字节与零交错。这就是所谓的*零扩展*，因为字节是无符号的。punpckhbw 可用于对高位字节做同样的事情。

这是一个展示如何做到这一点的片段：

```assembly
pxor      m2, m2 ; 将 m2 清零

movu      m0, [srcq]
movu      m1, m0 ; 在 m1 中制作 m0 的副本
punpcklbw m0, m2
punpckhbw m1, m2
```

```m0``` 和 ```m1``` 现在包含零扩展为字的原始字节。在下一课中，你将看到 AVX 中的三操作数指令如何使第二个 movu 变得不必要。

**符号扩展**

有符号数据稍微复杂一些。为了范围扩展一个有符号整数，我们需要使用一个称为[符号扩展](https://en.wikipedia.org/wiki/Sign_extension)的过程。这用符号位填充最高有效位 (MSB)。例如：int8_t 中的 -2 是 0b11111110。为了将其符号扩展为 int16_t，重复 1 的 MSB 以变为 0b1111111111111110。

```pcmpgtb```（打包比较大于字节）可用于符号扩展。通过进行比较（0 > 字节），如果字节为负，目标字节中的所有位都设置为 1，否则目标字节中的位设置为 0。punpckX 可以像上面一样用于执行符号扩展。如果字节为负，对应的字节是 0b11111111，否则是 0x00000000。将字节值与 pcmpgtb 的输出交错，结果就是执行了到字的符号扩展。

```assembly
pxor      m2, m2 ; 将 m2 清零

movu      m0, [srcq]
movu      m1, m0 ; 在 m1 中制作 m0 的副本

pcmpgtb   m2, m0
punpcklbw m0, m2
punpckhbw m1, m2
```

正如你所看到的，与无符号情况相比，多了一条指令。

**打包**

packuswb（打包无符号字到字节）和 packsswb 让你从字转到字节。它允许你将两个包含字的 SIMD 寄存器交错成一个包含字节的 SIMD 寄存器。注意如果值超过字节范围，它们将被饱和（即被限制在最大值）。

**洗牌 (Shuffles)**

洗牌，也称为排列 (permutes)，可以说是视频处理中最重要的指令，SSSE3 中可用的 pshufb（打包洗牌字节）是最重要的变体。

对于每个字节，相应的源字节被用作目标寄存器的索引，除非设置了 MSB，此时目标字节被清零。它类似于下面的 C 代码（尽管在 SIMD 中所有 16 次循环迭代并行发生）：

```c
for(int i = 0; i < 16; i++) {
    if(src[i] & 0x80)
        dst[i] = 0;
    else
        dst[i] = dst[src[i]]
}
```
这是一个简单的汇编示例：

```assembly
SECTION_DATA 64

shuffle_mask: db 4, 3, 1, 2, -1, 2, 3, 7, 5, 4, 3, 8, 12, 13, 15, -1

section .text

movu m0, [srcq]
movu m1, [shuffle_mask]
pshufb m0, m1 ; 基于 m1 洗牌 m0
```

注意，为了便于阅读，使用 -1 作为洗牌索引来清零输出字节：-1 作为一个字节是 0b11111111 位域（二进制补码），因此 MSB (0x80) 被设置。
