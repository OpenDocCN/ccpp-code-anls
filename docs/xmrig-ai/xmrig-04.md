# xmrig源码解析 4

# Argon2 [![Build Status](https://travis-ci.org/WOnder93/argon2.svg?branch=master)](https://travis-ci.org/WOnder93/argon2)
A multi-arch library implementing the Argon2 password hashing algorithm.

This project is based on the [original source code](https://github.com/P-H-C/phc-winner-argon2) by the Argon2 authors. The goal of this project is to provide efficient Argon2 implementations for various HW architectures (x86, SSE, ARM, PowerPC, ...).

For the x86_64 architecture, the library implements a simple CPU dispatch which automatically selects the best implementation based on CPU flags and quick benchmarks.

# Building
## Using GNU autotools

To prepare the build environment, run:
```cppbash
autoreconf -i
./configure
```

After that, just run `make` to build the library.

### Running tests
After configuring the build environment, run `make check` to run the tests.

### Architecture options
You can specify the target architecture by passing the `--host=...` flag to `./configure`.

Supported architectures:
 * `x86_64` &ndash; 64-bit x86 architecture
 * `generic` &ndash; use generic C impementation

## Using CMake

To prepare the build environment, run:
```cppbash
cmake -DCMAKE_BUILD_TYPE=Release .
```

Then you can run `make` to build the library.

## Using QMake/Qt Creator
A [QMake](http://doc.qt.io/qt-4.8/qmake-manual.html) project is also available in the `qmake` directory. You can open it in the [Qt Creator IDE](http://wiki.qt.io/Category:Tools::QtCreator) or build it from terminal:
```cppbash
cd qmake
# see table below for the list of possible ARCH and CONFIG values
qmake ARCH=... CONFIG+=...
make
```

### Architecture options
For QMake builds you can configure support for different architectures. Use the `ARCH` variable to choose the architecture and the `CONFIG` variable to set additional options.

Supported architectures:
 * `x86_64` &ndash; 64-bit x86 architecture
   * QMake config flags:
     * `USE_SSE2` &ndash; use SSE2 instructions
     * `USE_SSSE3` &ndash; use SSSE3 instructions
     * `USE_XOP` &ndash; use XOP instructions
     * `USE_AVX2` &ndash; use AVX2 instructions
     * `USE_AVX512F` &ndash; use AVX-512F instructions
 * `generic` &ndash; use generic C impementation


# `src/3rdparty/argon2/arch/generic/lib/argon2-arch.c`

这段代码是一个用于实现 Argon2 授权的示例程序。它包含了一个 `argon2_instance_t` 类型的实例，该实例实现了 Argon2 授权的代码。此外，它还包含了一个 `fill_segment_64` 函数和一个 `fill_segment_default` 函数。

* `argon2_instance_t` 是一个用于存储 Argon2 授权数据的结构体。它包含了多个域，包括 `device_id`、`public_key` 和 `private_key` 等。
* `fill_segment_64` 函数是一个用于填写 Argon2 授权段（Segment）的函数。它接收一个 `x` 表示要填充的 segments 数，以及一个 `n` 表示要填充的每个 segments 的大小。返回值是一个填满 `x` 个 segments 的二进制数。
* `fill_segment_default` 函数与 `fill_segment_64` 类似，但使用了 `default` 签名来指定默认填充策略。它接收一个 `instance` 表示 Argon2 实例的指针，以及一个 `position` 表示要填充的 segments 位置。返回值与 `fill_segment_64` 函数相同，但结果可能会有所不同。

该代码的作用是帮助您了解 Argon2 授权的基本结构和函数，以便您更好地理解 Argon2 的授权过程。


```cpp
#include <stdint.h>
#include <string.h>
#include <stdlib.h>

#include "impl-select.h"

#define rotr64(x, n) (((x) >> (n)) | ((x) << (64 - (n))))

#include "argon2-template-64.h"

void fill_segment_default(const argon2_instance_t *instance,
                          argon2_position_t position)
{
    fill_segment_64(instance, position);
}

```

这段代码定义了一个名为 "argon2_get_impl_list" 的函数，其功能是清空一个名为 "list" 的整型指针变量。

具体来说，函数接收一个指向整型指针 "list" 的参数，然后通过一个简单的赋值语句，将 "list" 的值初始化为0。这样，您就可以在需要的时候方便地访问 "list" 所指向的内存区域，而不会受到该内存区域内容的干扰。


```cpp
void argon2_get_impl_list(argon2_impl_list *list)
{
    list->count = 0;
}

```

# `src/3rdparty/argon2/arch/x86_64/lib/argon2-arch.c`

这段代码是一个C语言编译器，它实现了罗姆津算法（Rumgiest的多线程64位计数器）。该算法可以用于记录64位计数器中的每一位，通过计算得出结果。

具体来说，这段代码实现了一个名为“impl-select.h”的文件，其中包含了罗姆津算法的相关实现。主要包括以下几个部分：

1. 包含头文件：stdint.h、string.h、stdlib.h，这些是C语言标准库中的头文件。

2. 引入函数和数据：来自“argon2-sse2.h”、“argon2-ssse3.h”、“argon2-xop.h”、“argon2-avx2.h”和“argon2-avx512f.h”，这些函数和数据与SSE2和SSS3指令有关。

3. 引入“impl-select.h”文件中的函数，包括：rotr64、__expecti、__相等、__destruct。

4. 通过宏定义实现了几个辅助函数： rotr64 和 __destruct，分别用于计算旋转后的64位计数器和释放已经占用的内存空间。

5. 通过罗姆津算法实现了64位计数器的功能。该算法的主要思想是将64位计数器的每个位选择一个不同的颜色（0表示红色，1表示绿色），然后通过异或和移位操作实现计数器的功能。通过计算不同颜色的位的和，可以得到计数器当前的值。


```cpp
#include <stdint.h>
#include <string.h>
#include <stdlib.h>

#include "impl-select.h"

#include "argon2-sse2.h"
#include "argon2-ssse3.h"
#include "argon2-xop.h"
#include "argon2-avx2.h"
#include "argon2-avx512f.h"

/* NOTE: there is no portable intrinsic for 64-bit rotate, but any
 * sane compiler should be able to compile this into a ROR instruction: */
#define rotr64(x, n) ((x) >> (n)) | ((x) << (64 - (n)))

```

这段代码定义了两个函数：fill_segment_default和argon2_get_impl_list。

fill_segment_default函数的实现如下：
```cppc
void fill_segment_default(const argon2_instance_t *instance,
                         argon2_position_t position)
{
   fill_segment_64(instance, position);
}
```
该函数的作用是调用fill_segment_64函数，并传入一个argon2_instance_t类型的实例和argon2_position_t类型的位置参数。fill_segment_64函数的具体实现如下：
```cppc
void fill_segment_64(const argon2_instance_t *instance,
                         argon2_position_t position)
{
   // 在这里实现 fill_segment_64函数
}
```

argon2_get_impl_list函数的实现如下：
```cppc
void argon2_get_impl_list(argon2_impl_list *list)
{
   static const argon2_impl IMPLS[] = {
       { "x86_64",     NULL,                     fill_segment_default },
       { "SSE2",       xmrig_ar2_check_sse2,     xmrig_ar2_fill_segment_sse2 },
       { "SSSE3",      xmrig_ar2_check_ssse3,    xmrig_ar2_fill_segment_ssse3 },
       { "XOP",        xmrig_ar2_check_xop,      xmrig_ar2_fill_segment_xop },
       { "AVX2",       xmrig_ar2_check_avx2,     xmrig_ar2_fill_segment_avx2 },
       { "AVX-512F",   xmrig_ar2_check_avx512f,  xmrig_ar2_fill_segment_avx512f },
   };

   list->count = sizeof(IMPLS) / sizeof(IMPLS[0]);
   list->entries = IMPLS;
}
```
该函数的作用是返回一个argon2_impl_list类型的结构体，其中包含可用的impl列表。该函数首先定义了一个静态的IMPLS数组，然后使用sizeof函数获取IMPLS的数量，最后将IMPLS数组复制到argon2_impl_list类型的结构体中。

注意，上述代码中，对于不同的操作系统和硬件，fill_segment_64和argon2_get_impl_list的具体实现可能会有所不同。


```cpp
#include "argon2-template-64.h"

void fill_segment_default(const argon2_instance_t *instance,
                          argon2_position_t position)
{
    fill_segment_64(instance, position);
}

void argon2_get_impl_list(argon2_impl_list *list)
{
    static const argon2_impl IMPLS[] = {
        { "x86_64",     NULL,                     fill_segment_default },
        { "SSE2",       xmrig_ar2_check_sse2,     xmrig_ar2_fill_segment_sse2 },
        { "SSSE3",      xmrig_ar2_check_ssse3,    xmrig_ar2_fill_segment_ssse3 },
        { "XOP",        xmrig_ar2_check_xop,      xmrig_ar2_fill_segment_xop },
        { "AVX2",       xmrig_ar2_check_avx2,     xmrig_ar2_fill_segment_avx2 },
        { "AVX-512F",   xmrig_ar2_check_avx512f,  xmrig_ar2_fill_segment_avx512f },
    };

    list->count = sizeof(IMPLS) / sizeof(IMPLS[0]);
    list->entries = IMPLS;
}

```

# `src/3rdparty/argon2/arch/x86_64/lib/argon2-avx2.c`

这段代码包括以下几个部分：

1. `#include "argon2-avx2.h"`：引入了"argon2-avx2.h"头文件，可能是一个用于AVX2指令集的库。
2. `#ifdef HAVE_AVX2`：通过`#ifdef`判断当前编译器是否支持AVX2指令集。如果不支持，将跳过这一行。
3. `#include <string.h>`：引入了`<string.h>`头文件，可能是用于处理字符串类型的数据。
4. `#ifdef __GNUC__`：通过`#ifdef`判断当前编译器是否为GNUC。如果是，则包含`<x86intrin.h>`头文件。
5. `#else`：如果不是GNUC，则包含`<intrin.h>`头文件。
6. `#define r16`：定义了一个名为`r16`的宏，可能是用于定义AVX2指令集中的寄存器或寄存器组。
7. `(_mm256_setr_epi8(2, 3, 4, 5, 6, 7, 0, 1, 10, 11, 12, 13, 14, 15, 8, 9, 18, 19, 20, 21, 22, 23, 16, 17, 26, 27, 28, 29, 30, 31, 24, 25))`：通过`_mm256_setr_epi8`函数将8个字节的数据装入AVX2寄存器组中。这个函数可能接受一个8字节的输入，并将其存储到指定的寄存器中。

综合来看，这段代码可能是一个用于在支持AVX2的GCC编译器中编译ARM代码的工具链，或者是一个用于在支持AVX2的操作系统中运行ARM程序的工具链。


```cpp
#include "argon2-avx2.h"

#ifdef HAVE_AVX2
#include <string.h>

#ifdef __GNUC__
#   include <x86intrin.h>
#else
#   include <intrin.h>
#endif

#define r16 (_mm256_setr_epi8( \
     2,  3,  4,  5,  6,  7,  0,  1, \
    10, 11, 12, 13, 14, 15,  8,  9, \
    18, 19, 20, 21, 22, 23, 16, 17, \
    26, 27, 28, 29, 30, 31, 24, 25))

```

这段代码定义了三个宏：`r24`、`ror64_16` 和 `ror64_32`。它们都接受一个参数 `x`，并返回一个整数类型的值。

`r24`宏的实现是将 `x` 的8位二进制表示中的数字从右向左数6位，然后左移6位，最后对结果进行下标指定的操作。

`ror64_16`宏的实现与`r24`类似，只是对于8位二进制表示，对64位二进制表示进行操作。

`ror64_32`宏的实现是将 `x` 的32位二进制表示中的数字从右向左数32位，然后左移32位，最后对结果进行下标指定的操作。

`f`函数的实现是将 `x` 和 `y` 的二进制表示分别相乘，然后将两个结果的64位二进制表示进行按位异或操作，最后对结果进行左移64位操作得到的结果。


```cpp
#define r24 (_mm256_setr_epi8( \
     3,  4,  5,  6,  7,  0,  1,  2, \
    11, 12, 13, 14, 15,  8,  9, 10, \
    19, 20, 21, 22, 23, 16, 17, 18, \
    27, 28, 29, 30, 31, 24, 25, 26))

#define ror64_16(x) _mm256_shuffle_epi8((x), r16)
#define ror64_24(x) _mm256_shuffle_epi8((x), r24)
#define ror64_32(x) _mm256_shuffle_epi32((x), _MM_SHUFFLE(2, 3, 0, 1))
#define ror64_63(x) \
    _mm256_xor_si256(_mm256_srli_epi64((x), 63), _mm256_add_epi64((x), (x)))

static __m256i f(__m256i x, __m256i y)
{
    __m256i z = _mm256_mul_epu32(x, y);
    return _mm256_add_epi64(_mm256_add_epi64(x, y), _mm256_add_epi64(z, z));
}

```

这段代码定义了一个名为`G1`的宏，其含义为：

```cpp
G1(A0, B0, C0, D0, A1, B1, C1, D1)
```

展开来看，这个宏中定义了一系列数学函数，包括`f`、`_mm256_`和`_mm256_xor_si256`等。这些函数的具体实现可能需要在调用时具体指定，现在仅提供定义。

宏定义中包含了一系列语句，具体解释如下：

1. `do { ...; }`：是一个do-while循环，会在被调用函数体中依次执行循环体中的语句。

2. `A0 = f(A0, B0);`：是一个函数调用，`f`函数的具体实现未定义，需要在使用时具体指定。

3. `A1 = f(A1, B1);`：同上，第二个函数调用。

4. `D0 = _mm256_xor_si256(D0, A0);`：是一个_mm256_xor_si256函数，根据其第二个参数和第三个参数，对`D0`进行异或操作并存储结果，`A0`是第一个输入参数。

5. `D1 = _mm256_xor_si256(D1, A1);`：同上，第二个输入参数。

6. `D0 = ror64_32(D0);`：是一个`randint_ip`函数，根据其第三个参数，生成一个0到64之间的随机数，对`D0`进行按位异或操作并存储结果。

7. `D1 = randint_ip(D1, A0);`：同上，第二个输入参数。

8. `C0 = f(C0, D0);`：是一个未定义的函数，需要在使用时具体指定。

9. `C1 = f(C1, D1);`：同上，第二个函数调用。

10. `B0 = _mm256_xor_si256(B0, C0);`：是一个_mm256_xor_si256函数，根据其第二个输入参数和第三个输入参数，对`B0`进行异或操作并存储结果，`C0`是第一个输入参数。

11. `B1 = _mm256_xor_si256(B1, C1);`：同上，第二个输入参数。

12. `...`：省略了后续的语句。


```cpp
#define G1(A0, B0, C0, D0, A1, B1, C1, D1) \
    do { \
        A0 = f(A0, B0); \
        A1 = f(A1, B1); \
\
        D0 = _mm256_xor_si256(D0, A0); \
        D1 = _mm256_xor_si256(D1, A1); \
\
        D0 = ror64_32(D0); \
        D1 = ror64_32(D1); \
\
        C0 = f(C0, D0); \
        C1 = f(C1, D1); \
\
        B0 = _mm256_xor_si256(B0, C0); \
        B1 = _mm256_xor_si256(B1, C1); \
```

这段代码定义了一个名为G2的函数，其参数包括A0、B0、C0、D0、A1、B1、C1和D1。该函数使用了共8个线程，以并行执行的方式计算多个浮点数的和。

每次执行循环时，函数首先定义了一个名为B0的变量，并使用ror64_24将其初始化为0。然后，定义了一个名为B1的变量，并使用同样的方法将其初始化为0。

这两个变量将在循环中作为参数传递给函数，并在循环体内被多次使用。函数使用do-while循环来重复执行这些操作，直到满足一个条件，即0。

通过这种方式，该代码段可以在多核处理器上并行执行，从而提高计算浮点数的能力。


```cpp
\
        B0 = ror64_24(B0); \
        B1 = ror64_24(B1); \
    } while ((void)0, 0)

#define G2(A0, B0, C0, D0, A1, B1, C1, D1) \
    do { \
        A0 = f(A0, B0); \
        A1 = f(A1, B1); \
\
        D0 = _mm256_xor_si256(D0, A0); \
        D1 = _mm256_xor_si256(D1, A1); \
\
        D0 = ror64_16(D0); \
        D1 = ror64_16(D1); \
```

这段代码的主要目的是实现矩阵的横向一体化，即将一个矩阵的每一行与另一矩阵的每一列分别相乘，并将结果存储到一个新的矩阵中。

代码首先定义了一个名为DIAGONALIZE1的函数，该函数接受六个参数A0、B0、C0、D0、A1和B1，以及一个循环变量COLLATE。

在函数内部，首先定义了一个名为B0的变量，该变量使用_mm256_xor_si256函数对其每个元素与输入矩阵的对应元素相异或，并使用_MM_SHUFFLE函数对四个元素进行行序和列序的划分。

接下来定义了一个名为B1的变量，使用与上面函数类似的方法对其每个元素进行计算。

然后，定义了一个名为B0的变量，使用_mm256_permute4x64_epi64函数对其每个元素进行T形连接，并使用_MM_SHUFFLE函数对其每个元素进行行序和列序的划分。

最后，定义了一个名为B1的变量，与上面函数类似，对其每个元素进行计算。

整段代码使用了一个无限循环，在每次迭代时，将上面计算得到的B0和B1作为新矩阵的一部分，并输出新的矩阵。


```cpp
\
        C0 = f(C0, D0); \
        C1 = f(C1, D1); \
\
        B0 = _mm256_xor_si256(B0, C0); \
        B1 = _mm256_xor_si256(B1, C1); \
\
        B0 = ror64_63(B0); \
        B1 = ror64_63(B1); \
    } while ((void)0, 0)

#define DIAGONALIZE1(A0, B0, C0, D0, A1, B1, C1, D1) \
    do { \
        B0 = _mm256_permute4x64_epi64(B0, _MM_SHUFFLE(0, 3, 2, 1)); \
        B1 = _mm256_permute4x64_epi64(B1, _MM_SHUFFLE(0, 3, 2, 1)); \
```

这段代码的主要作用是实现矩阵的奇异化操作。奇异化操作可以将一个矩阵的每个元素与另一个矩阵的某个元素相乘，并返回一个新的矩阵，使得新矩阵中除了第一行外，其他行都是零。

具体来说，代码中首先定义了一个名为“UNDIAGONALIZE1”的函数，该函数接受一个5x5的矩阵A，一个5x5的矩阵B，一个5x5的矩阵C，以及一个5x5的矩阵D。这些矩阵都是输入参数。

在函数内部，首先进行了一系列的循环操作，每次循环将输入矩阵的A或B元素与一个3x3的矩阵进行异或运算，并保存结果到变量D中。接着，在每次循环内部，又对输入矩阵的C或D元素与一个4x4的矩阵进行异或运算，并保存结果到变量A中。最后，每次循环内部还有一条语句，用于输出奇异化后的结果，并使用0来表示行列不存在的元素。

因此，这段代码的主要作用是实现一个将一个5x5矩阵的每个元素与另一个5x5矩阵中元素相乘，并返回一个新的5x5矩阵的奇异化操作。


```cpp
\
        C0 = _mm256_permute4x64_epi64(C0, _MM_SHUFFLE(1, 0, 3, 2)); \
        C1 = _mm256_permute4x64_epi64(C1, _MM_SHUFFLE(1, 0, 3, 2)); \
\
        D0 = _mm256_permute4x64_epi64(D0, _MM_SHUFFLE(2, 1, 0, 3)); \
        D1 = _mm256_permute4x64_epi64(D1, _MM_SHUFFLE(2, 1, 0, 3)); \
    } while ((void)0, 0)

#define UNDIAGONALIZE1(A0, B0, C0, D0, A1, B1, C1, D1) \
    do { \
        B0 = _mm256_permute4x64_epi64(B0, _MM_SHUFFLE(2, 1, 0, 3)); \
        B1 = _mm256_permute4x64_epi64(B1, _MM_SHUFFLE(2, 1, 0, 3)); \
\
        C0 = _mm256_permute4x64_epi64(C0, _MM_SHUFFLE(1, 0, 3, 2)); \
        C1 = _mm256_permute4x64_epi64(C1, _MM_SHUFFLE(1, 0, 3, 2)); \
```

这段代码的主要目的是对一个4x4矩阵进行diagonal化处理，即将矩阵的边上下的向量进行拼接。

具体来说，代码中首先定义了一个名为DIAGONALIZE2的函数，它接受一个5x4的矩阵A0、B0、C0、D0以及4个分别表示向左、向右、向上、向下偏移的参数A1、B1、C1、D1。

然后，在do-while循环中，代码依次对A0、B0、C0、D0、A1、B1、C1、D1每个元素进行以下操作：

1. 使用_MM_SHUFFLE函数对每个元素进行升序排序。
2. 使用_mm256_permute4x64_epi64函数对升序排序后的元素进行右旋转换，得到一个新的4x4矩阵。
3. 对新的4x4矩阵进行左转，即将矩阵的边上下向右进行交换。

这样，就得到了一个完整的4x4矩阵，每个元素都是原始矩阵中对应元素经过diagonal化处理后得到的结果。


```cpp
\
        D0 = _mm256_permute4x64_epi64(D0, _MM_SHUFFLE(0, 3, 2, 1)); \
        D1 = _mm256_permute4x64_epi64(D1, _MM_SHUFFLE(0, 3, 2, 1)); \
    } while ((void)0, 0)

#define DIAGONALIZE2(A0, B0, C0, D0, A1, B1, C1, D1) \
    do { \
        __m256i tmp1, tmp2; \
        tmp1 = _mm256_blend_epi32(B0, B1, 0xCC); \
        tmp2 = _mm256_blend_epi32(B0, B1, 0x33); \
        B1 = _mm256_permute4x64_epi64(tmp1, _MM_SHUFFLE(2,3,0,1)); \
        B0 = _mm256_permute4x64_epi64(tmp2, _MM_SHUFFLE(2,3,0,1)); \
\
        tmp1 = C0; \
        C0 = C1; \
        C1 = tmp1; \
```

这段代码的主要目的是实现向量加法，并对其结果进行向量叠乘。具体实现过程如下：

1. 将两个输入向量 D0 和 D1 进行向量加法，并将其存储在变量 tmp1 中。
2. 将 tmp1 向左移动 4 位，并在末尾添加两个 0，形成一个新的向量 B0。
3. 对 B0 执行向量取反操作，并将其存储在变量 D0 中。
4. 输出一个空语句，即不执行任何操作。
5. 进入一个无限循环，每次将 D0 和 D1 作为输入向量，并输出它们的向量加法和结果。
6. 在循环内部，定义了一个名为 UNDIAGONALIZE2 的宏，该宏实现了一个简单的向量加法操作。

由于输出语句被省略了，因此无法直接查看输出结果。


```cpp
\
        tmp1 = _mm256_blend_epi32(D0, D1, 0xCC); \
        tmp2 = _mm256_blend_epi32(D0, D1, 0x33); \
        D0 = _mm256_permute4x64_epi64(tmp1, _MM_SHUFFLE(2,3,0,1)); \
        D1 = _mm256_permute4x64_epi64(tmp2, _MM_SHUFFLE(2,3,0,1)); \
    } while ((void)0, 0)

#define UNDIAGONALIZE2(A0, B0, C0, D0, A1, B1, C1, D1) \
    do { \
        __m256i tmp1, tmp2; \
        tmp1 = _mm256_blend_epi32(B0, B1, 0xCC); \
        tmp2 = _mm256_blend_epi32(B0, B1, 0x33); \
        B0 = _mm256_permute4x64_epi64(tmp1, _MM_SHUFFLE(2,3,0,1)); \
        B1 = _mm256_permute4x64_epi64(tmp2, _MM_SHUFFLE(2,3,0,1)); \
\
        tmp1 = C0; \
        C0 = C1; \
        C1 = tmp1; \
```

这段代码实现了一个Blake2-519算法中的函数，称为BLAKE2_ROUND1。

该函数的主要作用是执行多项式运算，并输出结果。具体来说，该函数接受64位无符号整数A、B、C、D和A1、B1、C1、D1，并使用多项式运算将它们组合成128位无符号整数G1、G2、DIAGONALIZE1，然后将多项式结果相乘，并输出结果。

以下是具体实现过程：

1. 定义宏定义

```cpp
#define BLAKE2_ROUND1(A0, B0, C0, D0, A1, B1, C1, D1) \
   do { \
       G1(A0, B0, C0, D0, A1, B1, C1, D1); \
       G2(A0, B0, C0, D0, A1, B1, C1, D1); \
       DIAGONALIZE1(A0, B0, C0, D0, A1, B1, C1, D1); \
       G1(A0, B0, C0, D0, A1, B1, C1, D1); \
       G2(A0, B0, C0, D0, A1, B1, C1, D1); \
   } while ((void)0, 0)
```

2. 函数体

函数体中包含了一个do-while循环，循环变量为void类型，表示该函数可以重复执行，且不会产生任何输出。

在do-while循环内部，使用嵌套的宏定义执行多项式运算。

```cpp
       tmp1 = _mm256_blend_epi32(D0, D1, 0xCC); \
       tmp2 = _mm256_blend_epi32(D0, D1, 0x33); \
       D1 = _mm256_permute4x64_epi64(tmp1, _MM_SHUFFLE(2,3,0,1)); \
       D0 = _mm256_permute4x64_epi64(tmp2, _MM_SHUFFLE(2,3,0,1)); \
   } while ((void)0, 0)
```

首先，定义了两个 temporary variable(tmp1和tmp2)，然后使用了这两个临时变量调用了_mm256_blend_epi32函数，将D0和D1两个32位无符号整数进行了混合运算，并将结果存储到了tmp1中。

然后，又使用了_mm256_blend_epi32函数，将D0和D1两个32位无符号整数进行了混合运算，并将结果存储到了tmp2中。

接下来，定义了一个permute4x64函数，它的功能是输出一个4x64的输出，这个函数的输入参数是一个混合运算的结果，即tmp1，和一些permute的参数，这些参数确定了多项式的具体形状，这个函数会将多项式按照指定的permute函数进行排序，并输出排序后的结果。这里使用了 _MM_SHUFFLE函数来执行高级 Permutations。

最后，将D1的值使用_mm256_permute4x64函数进行多次调用，使用不同的permute函数来对D0和D1进行排序，并最终将结果存储到D1中，从而完成了多项式的运算。

3. 输出结果

由于该函数没有输出 anything，因此最终结果也未进行输出。


```cpp
\
        tmp1 = _mm256_blend_epi32(D0, D1, 0xCC); \
        tmp2 = _mm256_blend_epi32(D0, D1, 0x33); \
        D1 = _mm256_permute4x64_epi64(tmp1, _MM_SHUFFLE(2,3,0,1)); \
        D0 = _mm256_permute4x64_epi64(tmp2, _MM_SHUFFLE(2,3,0,1)); \
    } while ((void)0, 0)

#define BLAKE2_ROUND1(A0, B0, C0, D0, A1, B1, C1, D1) \
    do { \
        G1(A0, B0, C0, D0, A1, B1, C1, D1); \
        G2(A0, B0, C0, D0, A1, B1, C1, D1); \
\
        DIAGONALIZE1(A0, B0, C0, D0, A1, B1, C1, D1); \
\
        G1(A0, B0, C0, D0, A1, B1, C1, D1); \
        G2(A0, B0, C0, D0, A1, B1, C1, D1); \
```

这段代码是一个用于图像去噪的代码，通过一系列的循环操作，对输入的图像进行处理，以达到降噪的效果。

具体来说，代码中定义了一个名为 BLAKE2_ROUND2 的函数，它接受一个二维输入参数矩阵 A0、A1、B0、B1、C0、C1、D0 和 D1，函数的主要作用是对这个矩阵进行处理，以达到去噪的效果。

在 BLAKE2_ROUND2 的函数内部，首先定义了一个 do-while 循环，只要输入不为 0，就一直执行循环体内的代码。在循环体内，定义了一系列的函数 G1、G2 和 DIAGONALIZE2，它们的作用是对输入的图像进行一些处理，如水平翻转、垂直翻转、双边处理等。

具体来说，G1 和 G2 函数分别对图像的第一行和第一列进行处理，每次处理完后，将处理结果保存回原来的输入中。而 DIAGONALIZE2 函数则是对图像进行翻转，将图像上下颠倒。

在处理完所有的输入图像后，再次将结果保存回原来的输入中，这样处理一次输入图像后，输入图像的噪声就会被去除，从而达到去噪的效果。


```cpp
\
        UNDIAGONALIZE1(A0, B0, C0, D0, A1, B1, C1, D1); \
    } while ((void)0, 0)

#define BLAKE2_ROUND2(A0, A1, B0, B1, C0, C1, D0, D1) \
    do { \
        G1(A0, B0, C0, D0, A1, B1, C1, D1); \
        G2(A0, B0, C0, D0, A1, B1, C1, D1); \
\
        DIAGONALIZE2(A0, B0, C0, D0, A1, B1, C1, D1); \
\
        G1(A0, B0, C0, D0, A1, B1, C1, D1); \
        G2(A0, B0, C0, D0, A1, B1, C1, D1); \
\
        UNDIAGONALIZE2(A0, B0, C0, D0, A1, B1, C1, D1); \
    } while ((void)0, 0)

```

This code appears to be a function that configures the ABI (Application Binary Interface) of a software product. It appears to be doing this by setting the values of a few constants and variables within the block and setting the results to the next block.

The code is using the `__m256i` data type to store 8-byte integer values. It is using the `_mm256_` prefix to use the mixed-precision arithmetic operations provided by the hardware support library, such as `_mm256_xor_si256` and `_mm256_storeu_si256`.

The code is setting up a block of 8-byte values, which is specified by the `ARGON2_HWORDS_IN_BLOCK` macro. This block is initialized with values that are derived from the input data, and then modified using the functions `_mm256_xor_si256` and `_mm256_storeu_si256`.

The code is also using the `BLAKE2_ROUND1` and `BLAKE2_ROUND2` functions to perform roundedAxisB spline interpolation on the input data, which is starting from the point `s[8 * i + 0]` and increasing to the point `s[8 * i + 7]`.

Finally, the code is setting up a loop that performs an aggregation of the results of the aggregation, which is stored in the array `block_XY`.


```cpp
enum {
    ARGON2_HWORDS_IN_BLOCK = ARGON2_OWORDS_IN_BLOCK / 2,
};

static void fill_block(__m256i *s, const block *ref_block, block *next_block,
                       int with_xor)
{
    __m256i block_XY[ARGON2_HWORDS_IN_BLOCK];
    unsigned int i;

    if (with_xor) {
        for (i = 0; i < ARGON2_HWORDS_IN_BLOCK; i++) {
            s[i] =_mm256_xor_si256(
                s[i], _mm256_loadu_si256((const __m256i *)ref_block->v + i));
            block_XY[i] = _mm256_xor_si256(
                s[i], _mm256_loadu_si256((const __m256i *)next_block->v + i));
        }

    } else {
        for (i = 0; i < ARGON2_HWORDS_IN_BLOCK; i++) {
            block_XY[i] = s[i] =_mm256_xor_si256(
                s[i], _mm256_loadu_si256((const __m256i *)ref_block->v + i));
        }
    }

    for (i = 0; i < 4; ++i) {
        BLAKE2_ROUND1(
            s[8 * i + 0], s[8 * i + 1], s[8 * i + 2], s[8 * i + 3],
            s[8 * i + 4], s[8 * i + 5], s[8 * i + 6], s[8 * i + 7]);
    }

    for (i = 0; i < 4; ++i) {
        BLAKE2_ROUND2(
            s[4 * 0 + i], s[4 * 1 + i], s[4 * 2 + i], s[4 * 3 + i],
            s[4 * 4 + i], s[4 * 5 + i], s[4 * 6 + i], s[4 * 7 + i]);
    }

    for (i = 0; i < ARGON2_HWORDS_IN_BLOCK; i++) {
        s[i] = _mm256_xor_si256(s[i], block_XY[i]);
        _mm256_storeu_si256((__m256i *)next_block->v + i, s[i]);
    }
}

```

该函数的作用是打印出二维方格阵中从起始地址到最后一个地址的元素。它接受两个指针变量 block 和 input_block 作为输入，并打印出从起始地址 (0) 到结束地址 (ARGON2_HWORDS_IN_BLOCK - 1) 的元素。

具体来说，该函数首先定义了两个大小为 ARGON2_HWORDS_IN_BLOCK 的零块（即在二维方格阵的起始位置为 0，结束位置为 ARGON2_HWORDS_IN_BLOCK-1 的范围内创建两个大小相等的空闲块）。

然后，该函数计算从起始地址 (0) 到结束地址 (ARGON2_HWORDS_IN_BLOCK-1) 的元素个数，并将结果存储在输入块中。为了实现这个目标，该函数使用了两个循环，一个从起始地址 (0) 到结束地址 (ARGON2_HWORDS_IN_BLOCK-1) 循环，另一个从起始地址 (0) 循环到结束地址 (ARGON2_HWORDS_IN_BLOCK-1)。

在每次循环中，该函数都会根据输入块的元素来填充零块。对于每次循环，该函数都会从起始地址 (0) 开始打印元素，直到元素个数达到输入块的大小（即 ARGON2_HWORDS_IN_BLOCK）。然后，它会从起始地址 (0) 开始循环，直到结束地址 (ARGON2_HWORDS_IN_BLOCK-1) 结束。这样，就创建了一个从起始地址 (0) 到结束地址 (ARGON2_HWORDS_IN_BLOCK) 的二维方格阵，并打印出了该矩阵中所有元素。


```cpp
static void next_addresses(block *address_block, block *input_block)
{
    /*Temporary zero-initialized blocks*/
    __m256i zero_block[ARGON2_HWORDS_IN_BLOCK];
    __m256i zero2_block[ARGON2_HWORDS_IN_BLOCK];

    memset(zero_block, 0, sizeof(zero_block));
    memset(zero2_block, 0, sizeof(zero2_block));

    /*Increasing index counter*/
    input_block->v[6]++;

    /*First iteration of G*/
    fill_block(zero_block, input_block, address_block, 0);

    /*Second iteration of G*/
    fill_block(zero2_block, address_block, address_block, 0);
}

```

This code snippet is part of a larger program that似乎处理一些图像数据， and it starts by reading data from a specified file and computing the index of a reference block.

The `fill_block` function is called here, and it takes three arguments: the block to fill (in this case, ref_block), the current block being filled (in this case, curr_block), and a flag indicating whether to perform an all-zero or all-one copy.

The code also defines some constants and variables at the beginning, such as `ARGON2_ADDRESSES_IN_BLOCK`, `ARGON2_VERSION_10`, `instance->memory`, `instance->lane_length`, `ref_index`, `position.pass`, `position.slice`, `position.index`, `ref_block`, and `curr_block`.

The code then enters a loop that reads data from the specified file and calculates the index of the reference block based on the `ARGON2_ADDRESSES_IN_BLOCK` flag.

If `ARGON2_ADDRESSES_IN_BLOCK` is `1`, the code reads the address of the next block from `instance->memory` and uses the `address_block.v` array to get a pseudo-random value within the block. It then compares the pseudo-random value to the `pseudo_rand` variable to determine which block to read from.

If `ARGON2_ADDRESSES_IN_BLOCK` is `0`, the code reads the address of the first block from `instance->memory` and uses it to get the index of the block.

The code then calculates the lane of the reference block and uses the `ref_lane` variable to determine which lane to read from.

The code reads the data from the specified block and stores it in the current block. If `ARGON2_VERSION_10` is `1`, the code performs an all-one copy of the data. If `0` is `position.pass` or `ARGON2_VERSION_10` is `2`, the code overwrites the data in the current block.

Finally, the code returns control to the `fill_block` function.


```cpp
void xmrig_ar2_fill_segment_avx2(const argon2_instance_t *instance, argon2_position_t position)
{
    block *ref_block = NULL, *curr_block = NULL;
    block address_block, input_block;
    uint64_t pseudo_rand, ref_index, ref_lane;
    uint32_t prev_offset, curr_offset;
    uint32_t starting_index, i;
    __m256i state[ARGON2_HWORDS_IN_BLOCK];
    int data_independent_addressing;

    if (instance == NULL) {
        return;
    }

    data_independent_addressing = (instance->type == Argon2_i) ||
            (instance->type == Argon2_id && (position.pass == 0) &&
             (position.slice < ARGON2_SYNC_POINTS / 2));

    if (data_independent_addressing) {
        init_block_value(&input_block, 0);

        input_block.v[0] = position.pass;
        input_block.v[1] = position.lane;
        input_block.v[2] = position.slice;
        input_block.v[3] = instance->memory_blocks;
        input_block.v[4] = instance->passes;
        input_block.v[5] = instance->type;
    }

    starting_index = 0;

    if ((0 == position.pass) && (0 == position.slice)) {
        starting_index = 2; /* we have already generated the first two blocks */

        /* Don't forget to generate the first block of addresses: */
        if (data_independent_addressing) {
            next_addresses(&address_block, &input_block);
        }
    }

    /* Offset of the current block */
    curr_offset = position.lane * instance->lane_length +
                  position.slice * instance->segment_length + starting_index;

    if (0 == curr_offset % instance->lane_length) {
        /* Last block in this lane */
        prev_offset = curr_offset + instance->lane_length - 1;
    } else {
        /* Previous block */
        prev_offset = curr_offset - 1;
    }

    memcpy(state, ((instance->memory + prev_offset)->v), ARGON2_BLOCK_SIZE);

    for (i = starting_index; i < instance->segment_length;
         ++i, ++curr_offset, ++prev_offset) {
        /*1.1 Rotating prev_offset if needed */
        if (curr_offset % instance->lane_length == 1) {
            prev_offset = curr_offset - 1;
        }

        /* 1.2 Computing the index of the reference block */
        /* 1.2.1 Taking pseudo-random value from the previous block */
        if (data_independent_addressing) {
            if (i % ARGON2_ADDRESSES_IN_BLOCK == 0) {
                next_addresses(&address_block, &input_block);
            }
            pseudo_rand = address_block.v[i % ARGON2_ADDRESSES_IN_BLOCK];
        } else {
            pseudo_rand = instance->memory[prev_offset].v[0];
        }

        /* 1.2.2 Computing the lane of the reference block */
        ref_lane = ((pseudo_rand >> 32)) % instance->lanes;

        if ((position.pass == 0) && (position.slice == 0)) {
            /* Can not reference other lanes yet */
            ref_lane = position.lane;
        }

        /* 1.2.3 Computing the number of possible reference block within the
         * lane.
         */
        position.index = i;
        ref_index = xmrig_ar2_index_alpha(instance, &position, pseudo_rand & 0xFFFFFFFF, ref_lane == position.lane);

        /* 2 Creating a new block */
        ref_block =
            instance->memory + instance->lane_length * ref_lane + ref_index;
        curr_block = instance->memory + curr_offset;

        /* version 1.2.1 and earlier: overwrite, not XOR */
        if (0 == position.pass || ARGON2_VERSION_10 == instance->version) {
            fill_block(state, ref_block, curr_block, 0);
        } else {
            fill_block(state, ref_block, curr_block, 1);
        }
    }
}


```

这段代码定义了一个名为“extern”的函数，声明了一个整型变量“int cpu_flags_has_avx2(void)”，但并没有给这个函数赋值。接着定义了一个名为“int xmrig_ar2_check_avx2(void)”的整型函数，并且给这个函数赋了一个名为“extern”的符号，暗示了函数实现中可能需要使用函数外部的“extern”函数。

在函数体内部，有两个函数声明，一个名为“int xmrig_ar2_check_avx2(void)”，另一个在文件末尾以“#endif”标示，但同样没有实现。

另外，还有一段注释，指出“extern”函数在函数体内部可能需要使用函数外部的“extern”函数，但并没有具体说明需要使用哪个函数。


```cpp
extern int cpu_flags_has_avx2(void);
int xmrig_ar2_check_avx2(void) { return cpu_flags_has_avx2(); }

#else

void xmrig_ar2_fill_segment_avx2(const argon2_instance_t *instance, argon2_position_t position) {}
int xmrig_ar2_check_avx2(void) { return 0; }

#endif

```

# `src/3rdparty/argon2/arch/x86_64/lib/argon2-avx512f.c`

这段代码包括以下几个部分：

1. 引入 Argon2 AVEX-512 F 类型定义：#include "argon2-avx512f.h"
2. 定义宏：#ifdef HAVE_AVX512F逞使用 Avax-512F 定义为真时，会包含以下宏定义：
```cpp
#   include <stdint.h>
#   include <string.h>
```
3. 定义函数：function prototype 如下：
```cpp
static __m512i f(__m512i x, __m512i y)
```
4. 实现 f 函数：在函数体内部，使用异步内存模型 (mm512) 通过调用 _mm512_种子 (功能是执行軟體描述的函数，而非为软件描述) 来实现。
```cpp
   __m512i z = _mm512_mul_epu32(x, y);
   return _mm512_add_epi64(_mm512_add_epi64(x, y), _mm512_add_epi64(z, z));
```
但是，需要注意的是，这段代码缺少部分必要的功能，还无法正常工作。


```cpp
#include "argon2-avx512f.h"

#ifdef HAVE_AVX512F
#include <stdint.h>
#include <string.h>

#ifdef __GNUC__
#   include <x86intrin.h>
#else
#   include <intrin.h>
#endif

#define ror64(x, n) _mm512_ror_epi64((x), (n))

static __m512i f(__m512i x, __m512i y)
{
    __m512i z = _mm512_mul_epu32(x, y);
    return _mm512_add_epi64(_mm512_add_epi64(x, y), _mm512_add_epi64(z, z));
}

```

这段代码定义了一个名为`G1`的宏，其含义是将形参`A0`、`B0`、`C0`、`D0`、`A1`、`B1`、`C1`、`D1`传递给函数`f`，并计算出返回值。

具体来说，宏定义中的`do-while`循环会多次调用函数`f`，每次传入不同的形参。在每次调用中，形参`A0`、`A1`、`D0`、`D1`、`C0`、`C1`、`B0`、`B1`会被传递给函数`f`，并在函数内部进行计算。

计算结果被存储在以`__银色文本`为前缀的`f`函数返回值中，最终形成一个完整的矩阵，如下所示：

```cpp
G1(A0, B0, C0, D0, A1, B1, C1, D1) = {A0=f(A0, B0), A1=f(A1, B1), D0=_mm512_xor_si512(D0, A0), D1=_mm512_xor_si512(D1, A1), C0=f(C0, D0), C1=f(C1, D1), B0=_mm512_xor_si512(B0, C0), B1=_mm512_xor_si512(B1, C1), A}
```


```cpp
#define G1(A0, B0, C0, D0, A1, B1, C1, D1) \
    do { \
        A0 = f(A0, B0); \
        A1 = f(A1, B1); \
\
        D0 = _mm512_xor_si512(D0, A0); \
        D1 = _mm512_xor_si512(D1, A1); \
\
        D0 = ror64(D0, 32); \
        D1 = ror64(D1, 32); \
\
        C0 = f(C0, D0); \
        C1 = f(C1, D1); \
\
        B0 = _mm512_xor_si512(B0, C0); \
        B1 = _mm512_xor_si512(B1, C1); \
```

这段代码定义了一个名为 G2 的函数，其参数包括 A0、B0、C0、D0、A1、B1 和 C1。函数内部执行以下操作：

1. 对输入参数进行右移 24 位，并乘以 ror64 函数。
2. 使用 do-while 循环，只要输入不等于 0，就一直执行循环体内的操作。
3. 在循环体内执行以下操作：
a. 使用 f 函数，将输入参数 A0 和 B0 作为参数传入，并得到一个浮点数结果 A1。
b. 使用 _mm512_ 函数，对输入参数 A1 和 B1 进行异或操作，并得到一个 64 位的二进制数 D0 和 D1。
c. 使用 ror64 函数，将 D0 和 D1 进行右移 16 位，并得到一个新的 64 位的二进制数 D0' 和 D1'。
d. 使用 ror64 函数，将 D0' 和 D1' 进行右移 16 位，并得到一个新的 64位的二进制数 D0'' 和 D1''。

该代码的主要目的是计算并存储一组浮点数中的平方根。通过将输入参数进行一系列的异或和右移运算，最终得到了一组完全平方的浮点数，即正数的平方根。


```cpp
\
        B0 = ror64(B0, 24); \
        B1 = ror64(B1, 24); \
    } while ((void)0, 0)

#define G2(A0, B0, C0, D0, A1, B1, C1, D1) \
    do { \
        A0 = f(A0, B0); \
        A1 = f(A1, B1); \
\
        D0 = _mm512_xor_si512(D0, A0); \
        D1 = _mm512_xor_si512(D1, A1); \
\
        D0 = ror64(D0, 16); \
        D1 = ror64(D1, 16); \
```

这段代码的主要目的是实现矩阵的行向量垂直化。具体来说，该代码执行以下操作：

1. 将输入的四个矩阵（A0、B0、C0、D0）传递给函数f，并存储在输出矩阵A1和B1中。
2. 对A1和B1矩阵执行异或操作，并存储在输出矩阵B0和B1中。
3. 对B0和B1矩阵执行异或操作，并存储在输出矩阵B0和B1中。
4. 在函数内部，使用 while 循环，只要输入矩阵中有元素，则执行上述操作。
5. 使用 _MM_SHUFFLE() 函数对输入矩阵进行混合排序，以便在执行异或操作时正确计算矩阵的行向量。


```cpp
\
        C0 = f(C0, D0); \
        C1 = f(C1, D1); \
\
        B0 = _mm512_xor_si512(B0, C0); \
        B1 = _mm512_xor_si512(B1, C1); \
\
        B0 = ror64(B0, 63); \
        B1 = ror64(B1, 63); \
    } while ((void)0, 0)

#define DIAGONALIZE(A0, B0, C0, D0, A1, B1, C1, D1) \
    do { \
        B0 = _mm512_permutex_epi64(B0, _MM_SHUFFLE(0, 3, 2, 1)); \
        B1 = _mm512_permutex_epi64(B1, _MM_SHUFFLE(0, 3, 2, 1)); \
```

这段代码定义了一个名为`UNDIAGONALIZE`的函数，其作用是实现二进制位宽向量数据的对齐和排序。其实现主要依赖于两个硬件特性：内存中的位宽向量数据和软件实现的向量中断。

具体来说，这段代码首先定义了两个输入参数：`A0`、`B0`、`C0`、`D0`和`A1`、`B1`、`C1`、`D1`，它们都是64位的内存空间，用于存储二进制位宽向量数据。接着，函数体中使用了一系列的`_mm512_permutex_epi64`函数，这些函数的作用是创建一个互斥锁并保证线程安全。这些锁被用于同时对多个输入参数进行互斥访问，以保证在排序过程中不会发生竞争条件和数据损坏等问题。

在函数体中，首先执行了一系列的`_MM_SHUFFLE`预处理指令，对输入参数进行了升序排序。接着，通过调用`_mm512_permutex_epi64`函数，在对址模式下实现了对输入参数的互斥访问。这个互斥访问的目的是保证在排序过程中只有一个线程在执行，以避免出现多个线程同时操作而导致的错误。

总的来说，这段代码定义了一个用于对二进制位宽向量数据进行排序和统一的函数，可以有效地提高程序的性能和稳定性。


```cpp
\
        C0 = _mm512_permutex_epi64(C0, _MM_SHUFFLE(1, 0, 3, 2)); \
        C1 = _mm512_permutex_epi64(C1, _MM_SHUFFLE(1, 0, 3, 2)); \
\
        D0 = _mm512_permutex_epi64(D0, _MM_SHUFFLE(2, 1, 0, 3)); \
        D1 = _mm512_permutex_epi64(D1, _MM_SHUFFLE(2, 1, 0, 3)); \
    } while ((void)0, 0)

#define UNDIAGONALIZE(A0, B0, C0, D0, A1, B1, C1, D1) \
    do { \
        B0 = _mm512_permutex_epi64(B0, _MM_SHUFFLE(2, 1, 0, 3)); \
        B1 = _mm512_permutex_epi64(B1, _MM_SHUFFLE(2, 1, 0, 3)); \
\
        C0 = _mm512_permutex_epi64(C0, _MM_SHUFFLE(1, 0, 3, 2)); \
        C1 = _mm512_permutex_epi64(C1, _MM_SHUFFLE(1, 0, 3, 2)); \
```

这段代码是一个用于实现BLAKE2算法的指令集。它定义了一个名为BLAKE2_ROUND的函数，以及一个名为BLAKE2_ROUND的宏定义。

在D0变量中，使用了一个名为_MM_SHUFFLE的SIMD指令，将输入的六个4字节整数进行了调换，其输出也被存储到了D0中。

在D1变量中，再次使用了一个_MM_SHUFFLE指令，对D0的输出进行了调换，其输出也被存储到了D1中。

两个D0和D1变量中数值应该互不相同，但在这里它们的值看起来是相同的，这可能是因为_MM_SHUFFLE指令在执行时对输入数据进行了“乱序”处理，可能导致两个输入值被编码顺序不当，从而使它们看起来像是相同的。

宏定义部分定义了一个名为BLAKE2_ROUND的函数，其输入参数为六个4字节整数，分别为A0、B0、C0、D0、A1和B1、C1和D1。函数的主要作用是执行BLAKE2算法中的“ round”过程，其具体实现方式可能因具体使用的BLAKE2实现而有所不同。但通常，round过程包括输入数据的左右移、十六进制转义、轴瓦变换、循环和最终结果的输出等步骤。

此外，该代码中定义了一个名为BLAKE2_ROUND的宏定义。该宏定义了六个名为G1、G2、DIAGONALIZE和UNDIAGONALIZE的函数，它们的具体实现方式也是BLAKE2算法的实现部分。


```cpp
\
        D0 = _mm512_permutex_epi64(D0, _MM_SHUFFLE(0, 3, 2, 1)); \
        D1 = _mm512_permutex_epi64(D1, _MM_SHUFFLE(0, 3, 2, 1)); \
    } while ((void)0, 0)

#define BLAKE2_ROUND(A0, B0, C0, D0, A1, B1, C1, D1) \
    do { \
        G1(A0, B0, C0, D0, A1, B1, C1, D1); \
        G2(A0, B0, C0, D0, A1, B1, C1, D1); \
\
        DIAGONALIZE(A0, B0, C0, D0, A1, B1, C1, D1); \
\
        G1(A0, B0, C0, D0, A1, B1, C1, D1); \
        G2(A0, B0, C0, D0, A1, B1, C1, D1); \
\
        UNDIAGONALIZE(A0, B0, C0, D0, A1, B1, C1, D1); \
    } while ((void)0, 0)

```

这两行代码定义了两个宏，SWAP_HALVES和SWAP_QUARTERS。宏的作用是在编译时对数据进行交换。

宏定义的具体实现如下：

1. SWAP_HALVES macro
这一行代码定义了一个名为SWAP_HALVES的宏，其作用是交换A0和A1两个整数。实现方式是先定义一个__m512i类型的变量t0和t1，然后执行两个内存重排__MM_SHUFFLE函数，将A0和A1的值与给定的内存进行交换，最后将A0和A1的值赋给t0和t1。最终的结果是A0和A1的值被交换了。

2. SWAP_QUARTERS macro
这一行代码定义了一个名为SWAP_QUARTERS的宏，其作用与SWAP_HALVES类似，但同时交换了A0和A1中的4个字节（即2个quad）。实现方式与SWAP_HALVES类似，但需要使用_mm512_permutexvar_epi64宏来获取CPU对内存的读写权限，以便在交换数据之前获取到最新的A0和A1值，从而保证在交换过程中，先读取或写入内存的值未被修改。最终的结果是A0和A1的值都被交换了。


```cpp
#define SWAP_HALVES(A0, A1) \
    do { \
        __m512i t0, t1; \
        t0 = _mm512_shuffle_i64x2(A0, A1, _MM_SHUFFLE(1, 0, 1, 0)); \
        t1 = _mm512_shuffle_i64x2(A0, A1, _MM_SHUFFLE(3, 2, 3, 2)); \
        A0 = t0; \
        A1 = t1; \
    } while((void)0, 0)

#define SWAP_QUARTERS(A0, A1) \
    do { \
        SWAP_HALVES(A0, A1); \
        A0 = _mm512_permutexvar_epi64(_mm512_setr_epi64(0, 1, 4, 5, 2, 3, 6, 7), A0); \
        A1 = _mm512_permutexvar_epi64(_mm512_setr_epi64(0, 1, 4, 5, 2, 3, 6, 7), A1); \
    } while((void)0, 0)

```



这两组宏定义涉及到汇编语言中的内存操作，具体解释如下：

1. UNSWAP_QUARTERS宏定义：

该宏定义了两个整数变量A0和A1，以及一个do-while循环。在循环中，使用_mm512_permutexvar_epi64宏实现了一组内存操作，其中包括两次permutexvar_epi64的调用和一次SWAP_HALVES函数调用。

两次permutexvar_epi64的调用用于实现并发读操作，即在读取A0和A1数据的同时，保证在同一时间只读取一次。由于该代码针对的是Quarters结构，因此两次permutexvar_epi64的实现可以看做是两个独立的读操作，其实际作用与一个赋值操作无关。

一次SWAP_HALVES函数用于半归约，即将A0和A1的低32位交换为A0和A1的高32位，实现半归约操作。

2. BLAKE2_ROUND1宏定义：

该宏定义了五个整数变量A0、B0、C0、D0和A1，以及一个do-while循环。在循环中，使用_mm512_permutexvar_epi64和_mm512_swapvar_epi64宏实现了一组内存操作，其中包括两次permutexvar_epi64的调用和三次SWAP_HALVES函数调用。

两次permutexvar_epi64的调用用于实现并发读操作，即在读取A0、B0、C0、D0和A1数据的同时，保证在同一时间只读取一次。

三次SWAP_HALVES函数用于半归约，即将A0和A1的低32位交换为A0和A1的高32位，实现半归约操作。


```cpp
#define UNSWAP_QUARTERS(A0, A1) \
    do { \
        A0 = _mm512_permutexvar_epi64(_mm512_setr_epi64(0, 1, 4, 5, 2, 3, 6, 7), A0); \
        A1 = _mm512_permutexvar_epi64(_mm512_setr_epi64(0, 1, 4, 5, 2, 3, 6, 7), A1); \
        SWAP_HALVES(A0, A1); \
    } while((void)0, 0)

#define BLAKE2_ROUND1(A0, C0, B0, D0, A1, C1, B1, D1) \
    do { \
        SWAP_HALVES(A0, B0); \
        SWAP_HALVES(C0, D0); \
        SWAP_HALVES(A1, B1); \
        SWAP_HALVES(C1, D1); \
        BLAKE2_ROUND(A0, B0, C0, D0, A1, B1, C1, D1); \
        SWAP_HALVES(A0, B0); \
        SWAP_HALVES(C0, D0); \
        SWAP_HALVES(A1, B1); \
        SWAP_HALVES(C1, D1); \
    } while ((void)0, 0)

```

这段代码定义了一个名为BLAKE2_ROUND2的函数，其参数为7个整数，分别为A0、A1、B0、B1、C0、C1、D0、D1。函数内部执行以下操作：首先，通过do-while循环体，对A0、A1、B0、B1、C0、C1、D0、D1进行了一系列的交换操作，使得这些参数的值进行了循环、随机化。其次，执行了BLAKE2_ROUND函数，对A0、B0、C0、D0、A1、B1、C1、D1进行了一系列的BLAKE2_ROUND函数，这些函数会使得参数的值产生更多的变化，但是这些变化是随机的，并不是可预测的。最后，通过do-while循环体，再次对A0、A1、B0、B1、C0、C1、D0、D1进行了一系列的交换操作，使得这些参数的值重新进行了循环、随机化。


```cpp
#define BLAKE2_ROUND2(A0, A1, B0, B1, C0, C1, D0, D1) \
    do { \
        SWAP_QUARTERS(A0, A1); \
        SWAP_QUARTERS(B0, B1); \
        SWAP_QUARTERS(C0, C1); \
        SWAP_QUARTERS(D0, D1); \
        BLAKE2_ROUND(A0, B0, C0, D0, A1, B1, C1, D1); \
        UNSWAP_QUARTERS(A0, A1); \
        UNSWAP_QUARTERS(B0, B1); \
        UNSWAP_QUARTERS(C0, C1); \
        UNSWAP_QUARTERS(D0, D1); \
    } while ((void)0, 0)

enum {
    ARGON2_VECS_IN_BLOCK = ARGON2_OWORDS_IN_BLOCK / 4,
};

```

This code appears to be a function that performs a simple arithmetic operation on a block of data (represented as a vector of 8-byte integers) and optionally performs a rounding operation using the Blake256 hashing algorithm.

The function takes a block of data (represented as a vector of 8-byte integers) as input, and performs the following operations:

1. Fills in any missing values in the input block with the value 0 (since the missing values are not explicitly specified, this is done to avoid any issues with 0-based indexing).
2. performs a simple arithmetic operation (addition) on the input data.
3. Performs a rounding operation using the Blake256 hashing algorithm. The rounding operation takes 2 arguments (the first is the data to be rounded, the second is the index of the data to be rounded)
4. fills in any missing values in the output block with the value 0 (since the missing values are not explicitly specified, this is done to avoid any issues with 0-based indexing).
5. Optionally, it performs a second rounding operation using the Blake256 hashing algorithm.

It appears that the function also performs some additional operations that are not strictly necessary for the rounding operation, but are included to improve the performance of the algorithm.


```cpp
static void fill_block(__m512i *s, const block *ref_block, block *next_block,
                       int with_xor)
{
    __m512i block_XY[ARGON2_VECS_IN_BLOCK];
    unsigned int i;

    if (with_xor) {
        for (i = 0; i < ARGON2_VECS_IN_BLOCK; i++) {
            s[i] =_mm512_xor_si512(
                s[i], _mm512_loadu_si512((const __m512i *)ref_block->v + i));
            block_XY[i] = _mm512_xor_si512(
                s[i], _mm512_loadu_si512((const __m512i *)next_block->v + i));
        }

    } else {
        for (i = 0; i < ARGON2_VECS_IN_BLOCK; i++) {
            block_XY[i] = s[i] =_mm512_xor_si512(
                s[i], _mm512_loadu_si512((const __m512i *)ref_block->v + i));
        }
    }

    for (i = 0; i < 2; ++i) {
        BLAKE2_ROUND1(
            s[8 * i + 0], s[8 * i + 1], s[8 * i + 2], s[8 * i + 3],
            s[8 * i + 4], s[8 * i + 5], s[8 * i + 6], s[8 * i + 7]);
    }

    for (i = 0; i < 2; ++i) {
        BLAKE2_ROUND2(
            s[2 * 0 + i], s[2 * 1 + i], s[2 * 2 + i], s[2 * 3 + i],
            s[2 * 4 + i], s[2 * 5 + i], s[2 * 6 + i], s[2 * 7 + i]);
    }

    for (i = 0; i < ARGON2_VECS_IN_BLOCK; i++) {
        s[i] = _mm512_xor_si512(s[i], block_XY[i]);
        _mm512_storeu_si512((__m512i *)next_block->v + i, s[i]);
    }
}

```

这段代码是一个静态函数，名为`next_addresses`，其作用是辅助输入的`input_block`和输出的`address_block`之间的数据传输。

函数内部定义了两个512字节的整型数组`zero_block`和`zero2_block`，都被初始化为零。然后，函数输入的`input_block`的`v`字段被递增。

接下来，函数实现了一个`fill_block`函数，接受一个空块`zero_block`和一个输入块`input_block`，以及一个输出块`address_block`。函数首先在输入块的`v`字段上进行循环，将输入块中的数据复制到输出块的对应字段上。然后，函数在输出块的对应字段上递增输入块的索引。

该函数的作用是用于在输入块和输出块之间传递数据，实现数据辅助传输。


```cpp
static void next_addresses(block *address_block, block *input_block)
{
    /*Temporary zero-initialized blocks*/
    __m512i zero_block[ARGON2_VECS_IN_BLOCK];
    __m512i zero2_block[ARGON2_VECS_IN_BLOCK];

    memset(zero_block, 0, sizeof(zero_block));
    memset(zero2_block, 0, sizeof(zero2_block));

    /*Increasing index counter*/
    input_block->v[6]++;

    /*First iteration of G*/
    fill_block(zero_block, input_block, address_block, 0);

    /*Second iteration of G*/
    fill_block(zero2_block, address_block, address_block, 0);
}

```

This code snippet is part of a larger program that似乎处理了某种硬件设备的初始化和运行。在这里，我主要解释了代码中涉及的几个概念。

首先，这段代码涉及到缓存行。缓存行被分为若干个块（block）。每个块包含一个或多个内存区域，这些内存区域可能与主存（main memory）相连。由于这段代码在处理硬件设备，所以它在使用缓存行来暂存和操作数据。

接下来，代码定义了一个名为`prev_offset`的变量。这是一个整数，用于保存上一次准备好的块的偏移量（offset）。这个变量将在之后的代码中被用来计算一些数据。

在下一行，代码检查当前遍历的块是否位于数据独立的缓存行中。如果是，那么它将随机从上一行的缓存行中读取一个地址。否则，它将从主存中随机读取一个地址。

接下来，代码计算参考块的位置和引用块的引用方式。如果代码的`data_independent_addressing`选项为真（默认情况下），那么它将使用基于ARGON2地址的伪随机数生成器来选择下一个块的位置。否则，它将使用链表的方式，将每个块的地址存储在一个数组中，并从数组的第一个元素开始生成伪随机数。

接下来，代码将根据参考块的位置和引用方式，计算出引用该块的引用块的索引。引用块的索引将用于访问块的附加数据，如CRC（循环冗余校验）值等。

最后，代码创建一个新的块并将其分配给准备好的引用块。新创建的块将被分配到指定的缓存行中，新块的偏移量将设置为与上一行中准备好的块的偏移量之和。

总之，这段代码的主要目的是在硬件设备准备就绪后，使用缓存行来暂存和操作数据，并计算出参考块和引用块的位置和引用方式。


```cpp
void xmrig_ar2_fill_segment_avx512f(const argon2_instance_t *instance, argon2_position_t position)
{
    block *ref_block = NULL, *curr_block = NULL;
    block address_block, input_block;
    uint64_t pseudo_rand, ref_index, ref_lane;
    uint32_t prev_offset, curr_offset;
    uint32_t starting_index, i;
    __m512i state[ARGON2_VECS_IN_BLOCK];
    int data_independent_addressing;

    if (instance == NULL) {
        return;
    }

    data_independent_addressing = (instance->type == Argon2_i) ||
            (instance->type == Argon2_id && (position.pass == 0) &&
             (position.slice < ARGON2_SYNC_POINTS / 2));

    if (data_independent_addressing) {
        init_block_value(&input_block, 0);

        input_block.v[0] = position.pass;
        input_block.v[1] = position.lane;
        input_block.v[2] = position.slice;
        input_block.v[3] = instance->memory_blocks;
        input_block.v[4] = instance->passes;
        input_block.v[5] = instance->type;
    }

    starting_index = 0;

    if ((0 == position.pass) && (0 == position.slice)) {
        starting_index = 2; /* we have already generated the first two blocks */

        /* Don't forget to generate the first block of addresses: */
        if (data_independent_addressing) {
            next_addresses(&address_block, &input_block);
        }
    }

    /* Offset of the current block */
    curr_offset = position.lane * instance->lane_length +
                  position.slice * instance->segment_length + starting_index;

    if (0 == curr_offset % instance->lane_length) {
        /* Last block in this lane */
        prev_offset = curr_offset + instance->lane_length - 1;
    } else {
        /* Previous block */
        prev_offset = curr_offset - 1;
    }

    memcpy(state, ((instance->memory + prev_offset)->v), ARGON2_BLOCK_SIZE);

    for (i = starting_index; i < instance->segment_length;
         ++i, ++curr_offset, ++prev_offset) {
        /*1.1 Rotating prev_offset if needed */
        if (curr_offset % instance->lane_length == 1) {
            prev_offset = curr_offset - 1;
        }

        /* 1.2 Computing the index of the reference block */
        /* 1.2.1 Taking pseudo-random value from the previous block */
        if (data_independent_addressing) {
            if (i % ARGON2_ADDRESSES_IN_BLOCK == 0) {
                next_addresses(&address_block, &input_block);
            }
            pseudo_rand = address_block.v[i % ARGON2_ADDRESSES_IN_BLOCK];
        } else {
            pseudo_rand = instance->memory[prev_offset].v[0];
        }

        /* 1.2.2 Computing the lane of the reference block */
        ref_lane = ((pseudo_rand >> 32)) % instance->lanes;

        if ((position.pass == 0) && (position.slice == 0)) {
            /* Can not reference other lanes yet */
            ref_lane = position.lane;
        }

        /* 1.2.3 Computing the number of possible reference block within the
         * lane.
         */
        position.index = i;
        ref_index = xmrig_ar2_index_alpha(instance, &position, pseudo_rand & 0xFFFFFFFF, ref_lane == position.lane);

        /* 2 Creating a new block */
        ref_block =
            instance->memory + instance->lane_length * ref_lane + ref_index;
        curr_block = instance->memory + curr_offset;

        /* version 1.2.1 and earlier: overwrite, not XOR */
        if (0 == position.pass || ARGON2_VERSION_10 == instance->version) {
            fill_block(state, ref_block, curr_block, 0);
        } else {
            fill_block(state, ref_block, curr_block, 1);
        }
    }
}

```

这段代码定义了一个名为“extern int cpu_flags_has_avx512f(void);”的函数，该函数返回一个整数类型的变量。接下来定义了一个名为“int xmrig_ar2_check_avx512f(void)”的函数，该函数也返回一个整数类型的变量。它的作用是判断是否支持AVX-512F指令。最后，在if语句中，当满足“#else”这个条件时，会先调用“xmrig_ar2_check_avx512f()”这个函数，如果这个函数返回0，那么就会执行“xmrig_ar2_fill_segment_avx512f()”这个函数，该函数会在AVX-512F模式下，对传入的起始位置和结束位置进行数据填充。


```cpp
extern int cpu_flags_has_avx512f(void);
int xmrig_ar2_check_avx512f(void) { return cpu_flags_has_avx512f(); }

#else

void xmrig_ar2_fill_segment_avx512f(const argon2_instance_t *instance, argon2_position_t position) {}
int xmrig_ar2_check_avx512f(void) { return 0; }

#endif

```

# `src/3rdparty/argon2/arch/x86_64/lib/argon2-sse2.c`

这段代码是一个C语言的编译器扩展，其中包含了两个函数，ror64_16和ror64_24。它们被用来实现64位代码的向量化(或者称为二进制编码)。

在代码中，首先通过#ifdef宏检查是否支持SSE2。如果不支持，则通过#elif宏检查是否支持(__GNUC__)，如果是，则包含在#include <x86intrin.h>中。否则，包含在#include <intrin.h>中。这里使用了环境变量 HAS_SSE2，以便在编译时检查。

接下来定义了两个函数，ror64_16和ror64_24。这两个函数都接受一个16字节的整数x作为参数。函数实现中，通过调用_mm_shuffle和_mm_sli宏来计算x的16位和64位二进制表示。函数实现中，使用了_mm_shuffle和_mm_sli宏，它们可以对一个16位无符号整数进行向量化。函数实现中，还使用了一个辅助函数_mm_xor_si128，它对两个16位无符号整数进行异或并存储到一个新的16位无符号整数中。

最后，通过宏定义将这两个函数分别宏名为ror64_16和ror64_24，以便在需要时进行使用。


```cpp
#include "argon2-sse2.h"

#ifdef HAVE_SSE2
#ifdef __GNUC__
#   include <x86intrin.h>
#else
#   include <intrin.h>
#endif

#define ror64_16(x) \
    _mm_shufflehi_epi16( \
        _mm_shufflelo_epi16((x), _MM_SHUFFLE(0, 3, 2, 1)), \
        _MM_SHUFFLE(0, 3, 2, 1))
#define ror64_24(x) \
    _mm_xor_si128(_mm_srli_epi64((x), 24), _mm_slli_epi64((x), 40))
```

这段代码定义了两个宏，分别是ror64_32和ror64_63。它们定义了两个整数函数，功能是通过异或操作和加法操作，将两个整数x和y计算得到一个新的整数z，并返回这个新的整数。

这两个函数是通过将x和y的平方通过移位运算得到一个无符号整数，然后对这个无符号整数进行异或操作，最后对这个结果进行加法操作得到一个有符号整数。这个加法操作使用了xadc和yadc寄存器，实现了向量加法的功能。


```cpp
#define ror64_32(x) _mm_shuffle_epi32((x), _MM_SHUFFLE(2, 3, 0, 1))
#define ror64_63(x) \
    _mm_xor_si128(_mm_srli_epi64((x), 63), _mm_add_epi64((x), (x)))

static __m128i f(__m128i x, __m128i y)
{
    __m128i z = _mm_mul_epu32(x, y);
    return _mm_add_epi64(_mm_add_epi64(x, y), _mm_add_epi64(z, z));
}

#define G1(A0, B0, C0, D0, A1, B1, C1, D1) \
    do { \
        A0 = f(A0, B0); \
        A1 = f(A1, B1); \
\
        D0 = _mm_xor_si128(D0, A0); \
        D1 = _mm_xor_si128(D1, A1); \
```

这段代码实现了一个循环，其中的变量A0、B0、C0、D0、A1、B1、C1和D1都被循环使用。通过调用函数f()来执行每一次循环，并将每一次循环的输出存储到变量G2的成员变量中。

具体来说，代码首先定义了一个名为G2的宏，其含义为：

```cpp
G2(A0, B0, C0, D0, A1, B1, C1, D1)
```

这个宏调用了函数f()中的A0、B0、C0、D0、A1、B1、C1和D1参数。通过这些参数，函数f()执行了一系列的计算，并将结果存储到了G2宏的成员变量中。

循环条件为((void)0, 0)，这表示只要G2宏的输出为真，循环就会继续执行，直到条件不成立为止。


```cpp
\
        D0 = ror64_32(D0); \
        D1 = ror64_32(D1); \
\
        C0 = f(C0, D0); \
        C1 = f(C1, D1); \
\
        B0 = _mm_xor_si128(B0, C0); \
        B1 = _mm_xor_si128(B1, C1); \
\
        B0 = ror64_24(B0); \
        B1 = ror64_24(B1); \
    } while ((void)0, 0)

#define G2(A0, B0, C0, D0, A1, B1, C1, D1) \
    do { \
        A0 = f(A0, B0); \
        A1 = f(A1, B1); \
```

这段代码的主要作用是实现一个循环，对输入的8个整数进行异或运算，并输出结果。

具体来说，代码首先定义了两个整型变量D0和D1，以及两个整型变量C0和C1。这些变量作为输入，参与了异或运算。接着，代码实现了一个f函数，接受4个整型参数，分别传入C0、D0、D1和B0，返回对应的f值。

接下来，代码进入了一个while循环，只要输入不为0，就一直执行循环体内的代码。循环体内的代码首先将输入的8个整数异或在一起，形成一个8位二进制数B0。然后，代码将B0的6位二进制数转换成64位整数，并输出结果。接着，代码将B1的6位二进制数转换成64位整数，并与B0进行异或运算，形成一个新的64位整数B1。

最后，代码再次将B1的6位二进制数转换成64位整数，并输出结果。


```cpp
\
        D0 = _mm_xor_si128(D0, A0); \
        D1 = _mm_xor_si128(D1, A1); \
\
        D0 = ror64_16(D0); \
        D1 = ror64_16(D1); \
\
        C0 = f(C0, D0); \
        C1 = f(C1, D1); \
\
        B0 = _mm_xor_si128(B0, C0); \
        B1 = _mm_xor_si128(B1, C1); \
\
        B0 = ror64_63(B0); \
        B1 = ror64_63(B1); \
    } while ((void)0, 0)

```

这两行代码定义了两个名为DIAGONALIZE和UNDIAGONALIZE的宏，它们都接受输入参数A、B、C、D、A1、B1、C1和D1以及输出参数A0、B0、C0、D0、A1、B1、C1和D1。

宏DIAGONALIZE的作用是水平翻转四元数A0、B0、C0、D0,A1、B1、C1、D1。具体来说，宏DIAGONALIZE通过两次循环，每次将一个字节数组中的四个元素(即低字节和三个高字节)存储到A、B、C、D四个变量中，然后将A、B、C、D四个变量与一个二进制数t0和t1进行并运算，得到一个新的字节数组，将这个新字节数组存储到A1、B1、C1、D1四个变量中。这样，宏DIAGONALIZE就水平翻转了四元数A0、B0、C0、D0、A1、B1、C1和D1。

宏UNDIAGONALIZE的作用是垂直翻转四元数A0、B0、C0、D0、A1、B1、C1和D1，具体来说，宏UNDIAGONALIZE通过两次循环，每次将一个字节数组中的四个元素(即高字节和三个低字节)存储到A、B、C、D四个变量中，然后将A、B、C、D四个变量与一个二进制数t0和t1进行并运算，得到一个新的字节数组，将这个新字节数组存储到A1、B1、C1、D1四个变量中。这样，宏UNDIAGONALIZE就垂直翻转了四元数A0、B0、C0、D0、A1、B1、C1和D1。


```cpp
#define DIAGONALIZE(A0, B0, C0, D0, A1, B1, C1, D1) \
    do { \
        __m128i t0 = D0; \
        __m128i t1 = B0; \
        D0 = _mm_unpackhi_epi64(D1, _mm_unpacklo_epi64(t0, t0)); \
        D1 = _mm_unpackhi_epi64(t0, _mm_unpacklo_epi64(D1, D1)); \
        B0 = _mm_unpackhi_epi64(B0, _mm_unpacklo_epi64(B1, B1)); \
        B1 = _mm_unpackhi_epi64(B1, _mm_unpacklo_epi64(t1, t1)); \
    } while ((void)0, 0)

#define UNDIAGONALIZE(A0, B0, C0, D0, A1, B1, C1, D1) \
    do { \
        __m128i t0 = B0; \
        __m128i t1 = D0; \
        B0 = _mm_unpackhi_epi64(B1, _mm_unpacklo_epi64(B0, B0)); \
        B1 = _mm_unpackhi_epi64(t0, _mm_unpacklo_epi64(B1, B1)); \
        D0 = _mm_unpackhi_epi64(D0, _mm_unpacklo_epi64(D1, D1)); \
        D1 = _mm_unpackhi_epi64(D1, _mm_unpacklo_epi64(t1, t1)); \
    } while ((void)0, 0)

```

这段代码定义了一个名为BLAKE2_ROUND的宏，其含义是对于输入的A、B、C、D四个参数，通过先进行G1、G2、DIAGONALIZE和UNDIAGONALIZE来计算出最终结果，其中G1、G2、DIAGONALIZE和UNDIAGONALIZE分别代表4个不同的计算步骤。

具体来说，宏定义中的代码会首先定义4个变量A0、A1、B0和B1，以及4个变量C0、C1、D0和D1，然后进行4次计算，每次先执行G1、G2、DIAGONALIZE和UNDIAGONALIZE中的任意一个，然后再进行一次自定义的计算，最后将计算结果赋值给A、B、C、D四个变量。

由于宏定义中包含的4个函数G1、G2、DIAGONALIZE和UNDIAGONALIZE均没有参数，因此可以推测出这些函数的具体实现是通过参数或者局部变量来实现的，但是具体如何实现并不影响宏定义的作用。


```cpp
#define BLAKE2_ROUND(A0, A1, B0, B1, C0, C1, D0, D1) \
    do { \
        G1(A0, B0, C0, D0, A1, B1, C1, D1); \
        G2(A0, B0, C0, D0, A1, B1, C1, D1); \
\
        DIAGONALIZE(A0, B0, C0, D0, A1, B1, C1, D1); \
\
        G1(A0, B0, C1, D0, A1, B1, C0, D1); \
        G2(A0, B0, C1, D0, A1, B1, C0, D1); \
\
        UNDIAGONALIZE(A0, B0, C0, D0, A1, B1, C1, D1); \
    } while ((void)0, 0)

#include "argon2-template-128.h"

```

这段代码定义了一个名为“xmrig_ar2_fill_segment_sse2”的函数，其作用是调用一个名为“fill_segment_128”的函数，但这个函数没有定义。 

此外，还定义了一个名为“xmrig_ar2_check_sse2”的函数，其作用是返回一个名为“int”的整数类型。 

这两个函数都与SSE2硬件加速有关。根据函数原型，第一个函数接收一个“argon2_instance_t”类型的实例和一个“argon2_position_t”类型的位置参数，然后调用“fill_segment_128”函数，但这个函数没有定义，因此其行为是未定义的。 

第二个函数名为“xmrig_ar2_check_sse2”，返回一个整数类型的值。根据函数原型，其返回值是“int”类型的，而且根据定义，返回值为0，表示当前处理器不支持SSE2。


```cpp
void xmrig_ar2_fill_segment_sse2(const argon2_instance_t *instance, argon2_position_t position)
{
    fill_segment_128(instance, position);
}

extern int cpu_flags_has_sse2(void);
int xmrig_ar2_check_sse2(void) { return cpu_flags_has_sse2(); }

#else

void xmrig_ar2_fill_segment_sse2(const argon2_instance_t *instance, argon2_position_t position) {}
int xmrig_ar2_check_sse2(void) { return 0; }

#endif

```