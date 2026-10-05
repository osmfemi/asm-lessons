**FFmpeg 汇编语言第三课**

下面解释更多术语，并简要回顾一下历史。

**指令集（Instruction Sets）**

你可能在上一课中已经看到讨论了 SSE2，这是一组 SIMD 指令集。当新一代 CPU 发布时，可能会带来新的指令，有时还会有更大的寄存器。x86 指令集的历史非常复杂，以下是简化版本（实际还有更多子类别）：

* MMX - 1997 年推出，Intel 处理器中的第一个 SIMD，64 位寄存器，具有历史意义
* SSE (Streaming SIMD Extensions) - 1999 年推出，128 位寄存器
* SSE2 - 2000 年推出，许多新指令
* SSE3 - 2004 年推出，第一批水平指令
* SSSE3 (Supplemental SSE3) - 2006 年推出，新指令，但最重要的是 pshufb 洗牌指令，可以说是视频处理中最重要的指令
* SSE4 - 2008 年推出，许多新指令，包括打包的最小值和最大值。
* AVX - 2011 年推出，256 位寄存器（仅浮点）和新的三操作数语法
* AVX2 - 2013 年推出，用于整数指令的 256 位寄存器
* AVX512 - 2017 年推出，512 位寄存器，新的操作掩码功能。由于使用新指令时 CPU 会降频，当时在 FFmpeg 中的使用有限。具有 vpermb 的全 512 位洗牌（排列）。
* AVX512ICL - 2019 年推出，不再有主频降低。
* AVX10 - 即将推出

值得注意的是，指令集既可以被添加到 CPU 中，也可以被移除。例如，AVX512 在第 12 代 Intel CPU 中被有争议地[移除](https://www.igorslab.de/en/intel-deactivated-avx-512-on-alder-lake-but-fully-questionable-interpretation-of-efficiency-news-editorial/)了。正因如此，FFmpeg 会进行运行时 CPU 检测，检测当前 CPU 支持的功能。

正如你在作业中看到的，函数指针默认指向 C 实现，并在运行时被特定指令集的变体替换。这意味着检测只做一次，之后无需重复。这与许多专有应用形成鲜明对比——后者硬编码特定指令集，导致功能完好的计算机被淘汰。运行时检测还允许动态开启/关闭优化函数，这是开源的一大优势。

像 FFmpeg 这样的程序在全球数十亿台设备上运行，其中一些可能非常古老。FFmpeg 技术上甚至支持仅有 SSE 的机器——那些已有 25 年历史的机器！好在 x86inc.asm 能够在你使用了某个指令集中不存在的指令时发出警告。

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

对于像 FFmpeg 这样拥有数十亿用户的应用来说，即使 0.1% 也意味着大量用户，一旦出问题就会产生大量错误报告。FFmpeg 拥有完善的测试基础设施，通过 [FATE 测试套件](https://fate.ffmpeg.org/?query=subarch:x86_64%2F%2F) 测试 CPU/操作系统/编译器的各种组合。每次提交都会在数百台机器上运行，以确保没有任何东西被破坏。

Intel 在这里提供了详细的指令集手册：[https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html)

搜索 PDF 可能比较麻烦，这里有一个非官方的网页版替代方案：[https://www.felixcloutier.com/x86/](https://www.felixcloutier.com/x86/)

这里还有一个 SIMD 指令的可视化展示：
[https://www.officedaytime.com/simd512e/](https://www.officedaytime.com/simd512e/)

x86 汇编的一大挑战在于找到适合需求的指令。有时候指令可以用于其最初设计意图之外的场景。

**指针偏移技巧（Pointer offset trickery）**

回到第一课中的原始函数，但在 C 函数中添加一个 width 参数。

这里使用 ptrdiff_t 而不是 int 作为 width 变量，以确保 64 位参数的高 32 位为零。如果直接在函数签名中传递 int 类型的 width，然后将其作为四字用于指针算术（即使用 `widthq`），寄存器的高 32 位可能包含任意值。虽然可以用 `movsxd` 进行符号扩展来解决（也可参见 x86inc.asm 中的宏 `movsxdifnidn`），但使用 ptrdiff_t 是更简单的做法。

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

逐步分析如下：

```assembly
   add srcq, widthq
   add src2q, widthq
   neg widthq
```

width 被加到每个指针上，使它们指向缓冲区的末尾，然后 width 被取反。

```assembly
    movu  m0, [srcq+widthq]
    movu  m1, [src2q+widthq]
```

加载时 widthq 为负值。因此第一次迭代时 [srcq+widthq] 指向 srcq 的原始地址，即缓冲区的开头。

```assembly
    add   widthq, mmsize
    jl .loop
```

mmsize 被加到负的 widthq 上，使其逐渐趋近于零。循环条件是 jl（小于零则跳转）。这个技巧让 widthq 同时充当指针偏移量**和**循环计数器，节省了一条 cmp 指令。它还允许在多次加载和存储中复用该偏移量，并在需要时使用偏移量的倍数（作业中会用到这一点）。

**对齐（Alignment）**

在之前的所有例子中，一直使用 movu 来回避对齐话题。许多 CPU 在数据对齐时（即内存地址能被 SIMD 寄存器大小整除时）可以更快地加载和存储数据。在 FFmpeg 中，尽可能使用 mova 进行对齐的加载和存储。

在 FFmpeg 中，av_malloc 能够在堆上分配对齐的内存，DECLARE_ALIGNED C 预处理器指令可以在栈上声明对齐的内存。如果对未对齐的地址使用 mova，会导致段错误（segmentation fault）并使程序崩溃。确保对齐值与 SIMD 寄存器大小匹配也很重要：xmm 为 16，ymm 为 32，zmm 为 64。

以下是如何将 RODATA 段的开头对齐到 64 字节：

```assembly
SECTION_RODATA 64
```

注意这只是对齐 RODATA 的开头。可能需要填充字节以确保下一个标签保持在 64 字节边界上。

**范围扩展（Range expansion）**

之前回避的另一个话题是溢出。例如，当一个字节的值在加法或乘法后超过 255 时就会发生溢出。有时需要比字节更大的中间值（如字）来完成运算，或者需要将数据保留在更大的中间尺寸中。

对于无符号字节，这就是 punpcklbw（packed unpack low bytes to words，即打包解包低位字节到字）和 punpckhbw（packed unpack high bytes to words，即打包解包高位字节到字）派上用场的地方。

来看 punpcklbw 是如何工作的。Intel 手册中的 SSE2 版本的语法如下：

| PUNPCKLBW xmm1, xmm2/m128 |
| :---- |

这意味着它的源操作数（右侧）可以是一个 xmm 寄存器或内存地址（m128 表示具有标准 [base + scale*index + disp] 语法的内存地址），目标操作数是一个 xmm 寄存器。

上面的 officedaytime.com 网站有一个很好的图表展示了发生了什么：

![What is this](image1.png)

你可以看到字节分别从两个寄存器的低半部分交错排列。但这与范围扩展有什么关系？如果 src 寄存器全为零，就会把 dst 中的字节与零交错——这就是所谓的*零扩展*（zero extension），因为字节是无符号的。punpckhbw 可用于对高位字节做同样的操作。

这是一个展示如何做到这一点的片段：

```assembly
pxor      m2, m2 ; 将 m2 清零

movu      m0, [srcq]
movu      m1, m0 ; 将 m0 复制到 m1
punpcklbw m0, m2
punpckhbw m1, m2
```

```m0``` 和 ```m1``` 现在包含零扩展为字的原始字节。在下一课中，你将看到 AVX 中的三操作数指令如何使第二个 movu 变得不必要。

**符号扩展（Sign extension）**

有符号数据稍微复杂一些。要对有符号整数进行范围扩展，需要使用[符号扩展](https://en.wikipedia.org/wiki/Sign_extension)（sign extension）。它用符号位填充高位（MSB）。例如：int8_t 中的 -2 是 0b11111110。将其符号扩展为 int16_t 时，MSB 的 1 被重复，得到 0b1111111111111110。

```pcmpgtb```（packed compare greater than byte，即打包比较大于字节）可用于符号扩展。通过比较（0 > 字节），如果字节为负，目标字节的所有位被设为 1；否则设为 0。然后用 punpckX 执行符号扩展：负字节对应 0b11111111，非负字节对应 0x00000000。将原始字节与 pcmpgtb 的结果交错，即完成了到字的符号扩展。

```assembly
pxor      m2, m2 ; 将 m2 清零

movu      m0, [srcq]
movu      m1, m0 ; 将 m0 复制到 m1

pcmpgtb   m2, m0
punpcklbw m0, m2
punpckhbw m1, m2
```

正如你所看到的，与无符号情况相比，多了一条指令。

**打包（Packing）**

packuswb（pack unsigned word to byte，即打包无符号字到字节）和 packsswb 可以将字转换为字节。它们将两个包含字的 SIMD 寄存器合并为一个包含字节的 SIMD 寄存器。注意，如果值超出字节范围，会被饱和（saturated，即截断到最大值）。

**洗牌（Shuffles）**

洗牌（Shuffles），也称为排列（permutes），可以说是视频处理中最重要的指令。SSSE3 中的 pshufb（packed shuffle bytes，即打包洗牌字节）是其中最重要的变体。

对于每个字节，源字节的值被用作目标寄存器的索引；如果 MSB 被设置，则目标字节被清零。这类似于以下 C 代码（不过在 SIMD 中 16 次迭代是并行发生的）：

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

注意这里用 -1 作为洗牌索引来清零输出字节，以便于阅读：-1 作为字节是 0b11111111（二进制补码），因此 MSB（0x80）被设置。
