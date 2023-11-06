# GGML源码解析 28

# `tests/test-vec0.c`

这段代码是一个名为"mul_mat_vec_f32_0"的函数，它执行矩阵向量的乘法操作。

该函数需要两个输入参数：一个float类型的向量src0和src1，以及一个float类型的向量dst，这些向量在矩阵中存储。函数还需要两个整数类型的参数：一个表示矩阵的行数，另一个表示矩阵的列数。

函数中使用了一个未定义的函数mul_mat_vec_f32_0，这个函数需要从头文件<stdio.h>、<assert.h>、<stdlib.h>和<time.h>中包含。这个函数的作用是计算src0和src1矩阵的乘积，并将结果存储到dst中。

函数的具体实现如下：

1. 首先，定义了一个const类型的变量N和M，它们都表示矩阵的最大大小。
2. 定义了一个名为mul_mat_vec_f32_0的函数，该函数接受两个float类型的参数src0和src1，以及一个float类型的参数dst，还需要两个unsigned类型的参数nrows和ncols，表示矩阵的行数和列数。
3. 在函数内部，使用两个for循环来遍历src0和src1矩阵的所有元素。
4. 对于每个元素，先定义一个float类型的变量sum，然后使用乘法分配律将src0[i*ncols + j]和src1[j]相乘，并将结果相加，最后将sum存储到dst中。
5. 循环结束后，dst中的所有元素都被相加，得到了一个float类型的结果，该结果即为矩阵的乘积。


```cpp
#include <stdio.h>
#include <assert.h>
#include <stdlib.h>
#include <time.h>

const int N = 1 << 14;
const int M = 1 << 14;

void mul_mat_vec_f32_0(
    const float * src0,
    const float * src1,
    float * dst,
    unsigned nrows,
    unsigned ncols) {
    for (unsigned i = 0; i < nrows; i++) {
        float sum = 0.0f;
        for (unsigned j = 0; j < ncols; j++) {
            sum += src0[i*ncols + j]*src1[j];
        }
        dst[i] = sum;
    }
}
```

This is a C function definition that includes a float variable named `afloat`, which is a custom alignment-optimized float.

The `__attribute__((__aligned__(32)))` attribute explicitly specifies that the `afloat` variable should be aligned to a 32-byte boundary, meaning that the compiler should ensure that the variable is large enough to hold a 32-byte value.

The `mul_mat_vec_f32_1` function is defined to perform matrix multiplication on 32-bit floating-point numbers, but it takes four arguments: a float vector `src0`, a float vector `src1`, a float tensor `dst` representing the destination of the multiplication, and the number of rows and columns of the `src1` vector.

The function works by iterating over each element of the `src1` vector, and then performs the multiplication using a common-source multiple (CSM) layout, which computes the multiplication and stores the result in the `dst` tensor element at that index.

The loop iterates over each element of the `src1` vector, meaning that the function has access to the first `ncols` columns of the matrix, and each row of the source.


```cpp
#if defined(_MSC_VER)
typedef float __declspec(align(32)) afloat;
#else
typedef float afloat __attribute__((__aligned__(32)));
#endif
void mul_mat_vec_f32_1(
    const afloat *restrict src0,
    const afloat *restrict src1,
    afloat *restrict dst,
    unsigned nrows,
    unsigned ncols) {
    for (unsigned i = 0; i < nrows; i++) {
        const afloat * restrict row = src0 + i*ncols;
        const afloat * restrict col = src1;

        float sum = 0.0f;

        for (unsigned j = 0; j < ncols; j++) {
            sum += *row++ * *col++;
        }

        dst[i] = sum;

        //float sum[8] = {0.0f, 0.0f, 0.0f, 0.0f, 0.0f, 0.0f, 0.0f, 0.0f};

        //for (unsigned j = 0; j < ncols; j += 8) {
        //    sum[0] += row[0]*col[0];
        //    sum[1] += row[1]*col[1];
        //    sum[2] += row[2]*col[2];
        //    sum[3] += row[3]*col[3];
        //    sum[4] += row[4]*col[4];
        //    sum[5] += row[5]*col[5];
        //    sum[6] += row[6]*col[6];
        //    sum[7] += row[7]*col[7];

        //    row += 8;
        //    col += 8;
        //}

        //dst[i] = sum[0] + sum[1] + sum[2] + sum[3] + sum[4] + sum[5] + sum[6] + sum[7];
    }
}

```

这段代码定义了一个名为 `mul_mat_vec_f32_2` 的函数，它的参数包括一个指向矩阵源的一维指针 `src0` 和一个指向矩阵源的另一维指针 `src1`，以及一个指针变量 `dst` 和一个整数变量 `nrows` 和 `ncols`。

函数的主要目的是对两个矩阵的元素进行逐行相乘并存储到目标矩阵中。函数的实现包括以下几个步骤：

1. 定义一个名为 `sum` 的浮点型变量，用于存储每一行的乘积。
2. 定义一个指针变量 `row`，用于指向矩阵 `src1` 的第一行，以及一个指针变量 `col`，用于指向矩阵 `src0` 的第一列。
3. 定义一个循环，该循环用于遍历矩阵 `src0` 和 `src1` 的所有元素。
4. 在循环内部，使用嵌套的循环计算每个元素与对应元素的乘积，并将结果累加到 `sum` 中。
5. 将 `row` 和 `col` 移动到下一个元素。
6. 将计算得到的乘积存储到目标变量 `dst` 的对应位置，并更新指针变量 `d` 指向下一个元素。
7. 最后，循环结束后，将 `dst` 指向的目标变量。

总之，该函数的作用是对两个矩阵的元素进行逐行相乘并存储到目标矩阵中。


```cpp
void mul_mat_vec_f32_2(
    const void * src0,
    const void * src1,
    void * dst,
    unsigned nrows,
    unsigned ncols) {
    void * d = dst;
    for (unsigned i = 0; i < nrows; i++) {
        float sum = 0.0f;

        const char * row = (const char*)src0 + i*ncols*sizeof(float);
        const char * col = (const char*)src1;
        for (unsigned j = 0; j < ncols; j++) {
            sum += (*(float *)row) * (*(float *)col);
            row += sizeof(float);
            col += sizeof(float);
        }
        *(float *)d = sum;
        d = (char*)d + sizeof(float);
    }
}

```

这段代码是一个C语言的函数，用于在矩阵上向量化。函数的输入参数是一个float类型的数组src0和src1，以及一个float类型的数组dst。函数内部使用 aligned_alloc 函数对src0和src1进行内存分配，并使用 mul_mat_vec_f32_f 函数在src0和src1之间执行矩阵向量化。函数内部使用循环对src0和src1进行迭代，并计算循环内部的累加和。

由于该函数没有返回值，因此在实际应用中需要在调用该函数之后，通过合理的操作返回结果。


```cpp
#if defined(_MSC_VER)
void* aligned_alloc(size_t alignment, size_t size) {
    return _aligned_malloc(size, alignment);
}
#endif

int main(int argc, const char ** argv) {
    //float * src0 = malloc(sizeof(float)*N*M);
    //float * src1 = malloc(sizeof(float)*M);
    //float * dst  = malloc(sizeof(float)*N);

    afloat * src0 = (float *)(aligned_alloc(32, sizeof(float)*N*M));
    afloat * src1 = (float *)(aligned_alloc(32, sizeof(float)*M));
    afloat * dst  = (float *)(aligned_alloc(32, sizeof(float)*N));

    for (int i = 0; i < N*M; i++) {
        src0[i] = (afloat)i;
    }

    for (int i = 0; i < M; i++) {
        src1[i] = (afloat)i;
    }

    const int nIter = 10;

    const clock_t start = clock();

    double sum = 0.0f;
    for (int i = 0; i < nIter; i++) {
        //mul_mat_vec_f32_0(src0, src1, dst, N, M);
        mul_mat_vec_f32_1(src0, src1, dst, N, M);
        //mul_mat_vec_f32_2(src0, src1, dst, N, M);
        for (int  i = 0; i < N; i++) {
            sum += dst[i];
        }
    }

    {
        const clock_t end = clock();
        printf("%s: elapsed ticks: %ld\n", __func__, end - start);
    }

    printf("%f\n", sum);

    return 0;
}

```

# `tests/test-vec1.c`

这段代码包括以下几个部分：

1. 标准库头文件：包括 <stdint.h>、<stdio.h>、<assert.h>、<stdlib.h>、<time.h> 和 <math.h>，这些头文件定义了整数类型、输入输出函数以及一些数学函数的用法。

2. 包含文件：包括 <immintrin.h>，这是一部分 optimized for AVX-512 的代码，用于数学运算。

3. 时钟函数：包括 <time.h> 和 <sys/time.h>，用于获取当前时间。

4. 浮点数函数：包括 <math.h>，定义了 mathematical functions for floating-point numbers.

5. 并行计算库：包括 <immintrin.h>，用于在 AVX-512 架构上进行并行计算。

6. 输出设备：包括 <stdout>，用于将计算结果输出到 stdout.

7. 内存分配：包括 <stdlib.h>，用于 malloc 和 free 内存。

8. 计算矩阵：包括 <immintrin.h>，定义了一个库用于在 AVX-512 架构上执行矩阵计算。

9. 生成随机数：包括 <time.h>，使用 <rand> 和 <time.h> 函数生成随机数。

10. 判断是否为素数：包括 <stdbool.h> 和 <math.h>，定义了一个库用于检测一个数是否为素数。

11. 计算斐波那契数列：包括 <math.h>，定义了一个库用于计算斐波那契数列。

12. 计算阶乘：包括 <math.h>，定义了一个库用于计算一个数的阶乘。

13. 判断矩阵是否满秩：包括 <immintrin.h>，定义了一个库用于检查矩阵是否满秩。

14. 输出矩阵：包括 <stdout>，用于将矩阵输出到 stdout。

15. 计算旋转矩阵：包括 <immintrin.h>，定义了一个库用于执行矩阵的旋转。


```cpp
#include <stdint.h>
#include <stdio.h>
#include <assert.h>
#include <stdlib.h>
#include <time.h>
#include <math.h>

#include <sys/time.h>

#include <immintrin.h>

const int N = 1 << 14;
const int M = 768;

//
```

这段代码是一个计算两个矩阵的乘积并存储在第二个矩阵中的函数。矩阵乘法是在两个数组之间进行的，所以第一个参数是第一个矩阵的每个元素与第二个矩阵的每个元素相乘的结果。第二个参数是一个存储矩阵结果的指针，第三个参数是矩阵的行数和列数。

具体来说，函数接收两个32位浮点数矩阵src0和src1，然后在这些矩阵的第一行和第一列上分别执行乘法运算。对于每个行，函数使用一个float变量sum来保存当前行中所有元素相乘的结果，然后将这个float值存储到相应的dst数组元素中。最后，函数返回dst数组，它包含了两个矩阵的乘积。


```cpp
// naive implementation
//

void mul_mat_vec_f32_0(
    const float * restrict src0,
    const float * restrict src1,
    float * dst,
    int nrows,
    int ncols) {
    for (int i = 0; i < nrows; i++) {
        float sum = 0.0f;
        for (int j = 0; j < ncols; j++) {
            sum += src0[i*ncols + j]*src1[j];
        }
        dst[i] = sum;
    }
}

```

这段代码定义了一个名为 reduce_vector8_0 的函数，其输入参数是一个 8 32 位浮点数向量 v。该函数使用软件并行计算技术(SIMD)将 v 向量中的所有元素相加，并返回结果。其具体实现如下：

1. 从 v 向量的第一个元素开始，提取出 8 位宽的 floating-point 向量，并将其保存到 v1 中。
2. 从 v 向量的第二个元素开始，提取出 8 位宽的 floating-point 向量，并将其保存到 v2 中。
3. 从 v1 和 v2 中提取共 8 位的 floating-point 向量，并将其与 v 中的元素逐个相加，得到一个新的 8 位宽的 floating-point 向量。
4. 使用 Shuffle 算法对新的 8 位宽向量进行排序，具体参数为 0x4e。
5. 使用 Convolution 算法对新的 8 位宽向量进行模糊处理，参数为 0x11。
6. 使用求值操作，将 v 向量中的最后一个元素作为结果，并将其转换成浮点数类型。

该函数的作用是计算一个 8 位宽的 floating-point 向量 v 中的所有元素之和，并返回结果。由于其使用了 SIMD 并行计算技术，因此其性能非常的出色，尤其适用于大规模数据的计算。


```cpp
//
// SIMD with 8 32-bit floats
//

float reduce_vector8_0(__m256 v) {
    __m128 v1 = _mm256_extractf128_ps(v, 0);
    __m128 v2 = _mm256_extractf128_ps(v, 1);
    __m128 v3 = _mm_add_ps(v1, v2);
    __m128 v4 = _mm_shuffle_ps(v3, v3, 0x4e);
    __m128 v5 = _mm_add_ps(v3, v4);
    __m128 v6 = _mm_shuffle_ps(v5, v5, 0x11);
    __m128 v7 = _mm_add_ps(v5, v6);
    return _mm_cvtss_f32(v7);
}

```

这段代码是一个使用AVX（向量化计算加速）实现的矩阵乘法函数。主要目的是对两个二维矩阵（src0和src1）的元素进行乘法运算，并将结果存储到dest（destination）数组中。

在这段代码中，我们首先通过`const int ncols8 = ncols & ~7;`这一行来获取一个8位宽的矩阵，然后通过一系列循环来对每个元素进行操作：

1. 对src0数组的每个元素，我们将创建一个8位宽的临时变量sum，并使用_mm256_setzero_ps和_mm256_loadu_ps函数来初始化它。

2. 对于src1数组的每个元素，我们也将创建一个8位宽的临时变量sum，并使用_mm256_setzero_ps和_mm256_loadu_ps函数来初始化它。

3. 我们将src0数组的每个元素与src1数组的每个元素相乘，并将结果存储到sum中。这可以通过以下代码实现：

```cpp
   for (int i = 0; i < nrows; i++) {
       __m256 sum = _mm256_setzero_ps();
       for (int j = 0; j < ncols8; j += 8) {
           __m256 a = _mm256_loadu_ps(src0 + i*ncols + j);
           __m256 b = _mm256_loadu_ps(src1 + j);
           __m256 c = _mm256_mul_ps(a, b);
           sum = _mm256_add_ps(sum, c);
       }
       dst[i] = reduce_vector8_0(sum);
   }
```

4. 最后，我们通过另一个循环来将src0数组的元素与src1数组的元素相乘，并将结果存储到dst数组中：

```cpp
   for (int i = 0; i < nrows; i++) {
       dst[i] += src0[i*ncols + ncols8] * src1[i];
   }
```

在这段代码中，我们使用AVX指令将矩阵乘法运算分散为一系列更小的操作，从而避免了在内存中传输数据，提高了执行效率。


```cpp
// vectorized implementation using AVX
void mul_mat_vec_f32_1(
    const float * restrict src0,
    const float * restrict src1,
    float * dst,
    int nrows,
    int ncols) {

    const int ncols8 = ncols & ~7;

    for (int i = 0; i < nrows; i++) {
        __m256 sum = _mm256_setzero_ps();
        for (int j = 0; j < ncols8; j += 8) {
            __m256 a = _mm256_loadu_ps(src0 + i*ncols + j);
            __m256 b = _mm256_loadu_ps(src1 + j);
            __m256 c = _mm256_mul_ps(a, b);
            sum = _mm256_add_ps(sum, c);
        }
        dst[i] = reduce_vector8_0(sum);

        for (int j = ncols8; j < ncols; j++) {
            dst[i] += src0[i*ncols + j]*src1[j];
        }
    }
}

```

这段代码是一个名为 `mul_mat_vec_f32_2` 的函数，其作用是实现两个矩阵的乘积。

函数接受两个同方向 32 阶矩阵 `src0` 和 `src1`，以及一个同方向 32 阶矩阵 `dst`，以及矩阵的行数 `nrows` 和列数 `ncols`。

函数内部先定义了三个整数变量 `i`、`sum0` 和 `sum1`，以及三个整数变量 `ncols32`。

接下来，函数体内部进行了循环操作，共执行了 `nrows` 次，每次循环内部先定义了一个名为 `sum0` 的宏，一个名为 `sum1` 的宏和一个名为 `sum2` 的宏和一个名为 `sum3` 的宏。

接着，在循环内部，定义了四个变量 `a0`、`a1`、`a2` 和 `a3`，分别来自 `src0` 中的第 `i` 行，`src1` 中的第 `j` 行。

然后，定义了四个变量 `b0`、`b1`、`b2` 和 `b3`，分别来自 `src1` 中的第 `i` 行，`src1` 中的第 `j` 行。

接下来，在循环内部，进行了 32 次取余操作，每次取余操作都将余数存储到了变量 `a` 中，并计算了变量 `b` 中余数为 `31 - j` 时的值。

最后，循环结束后，将 `dst` 中的元素值逐步乘以 `a` 和 `b` 中的元素值，并将结果存储到了 `dst` 中。


```cpp
void mul_mat_vec_f32_2(
    const float * restrict src0,
    const float * restrict src1,
    float * dst,
    int nrows,
    int ncols) {

    const int ncols32 = ncols & ~31;

    for (int i = 0; i < nrows; i++) {
        __m256 sum0 = _mm256_setzero_ps();
        __m256 sum1 = _mm256_setzero_ps();
        __m256 sum2 = _mm256_setzero_ps();
        __m256 sum3 = _mm256_setzero_ps();

        const float * restrict src0_row = src0 + i*ncols;
        for (int j = 0; j < ncols32; j += 32) {
            __m256 a0 = _mm256_loadu_ps(src0_row + j + 0);
            __m256 a1 = _mm256_loadu_ps(src0_row + j + 8);
            __m256 a2 = _mm256_loadu_ps(src0_row + j + 16);
            __m256 a3 = _mm256_loadu_ps(src0_row + j + 24);
            __m256 b0 = _mm256_loadu_ps(src1 + j + 0);
            __m256 b1 = _mm256_loadu_ps(src1 + j + 8);
            __m256 b2 = _mm256_loadu_ps(src1 + j + 16);
            __m256 b3 = _mm256_loadu_ps(src1 + j + 24);
```

这段代码的作用是实现向量点积（点积）操作。点积是一种数学运算，用于计算两个向量之间的相似程度。

在__FMA__这个预处理指令中，我们定义了三个变量sum0、sum1和sum2，以及一个未定义的变量dst。然后进行了一系列的向量加法和乘法，最终得到了一个32位的结果dst。

__FMA__指令的功能是计算两个16字节的向量a0和b0的点积，并输出结果。这里我们定义了三个变量sum0、sum1和sum2，用于记录向量加法或乘法的结果。如果__FMA__指令定义成功，我们就会使用这些变量计算点积。

否则，我们需要使用一些数学技巧来计算点积。具体来说，我们需要对向量a0和b0进行一些数学变换，然后将这些变换的结果相加，最后输出结果dst。

以下是详细的计算过程：

1. 初始化变量sum0、sum1和sum2为0，变量dst为0。

2. 如果__FMA__指令定义成功，我们按照以下步骤计算点积：

a. 使用_mm256_fmadd_ps函数，将向量a0和b0相加，并将结果存储在变量sum0中。

b. 使用_mm256_fmadd_ps函数，将向量a1和b1相加，并将结果存储在变量sum1中。

c. 使用_mm256_fmadd_ps函数，将向量a2和b2相加，并将结果存储在变量sum2中。

d. 将变量sum0、sum1和sum2的值存储到变量dst中。

3. 如果__FMA__指令未定义成功，我们需要使用一些数学技巧来计算点积。具体来说，我们需要对向量a0和b0进行一些数学变换，然后将这些变换的结果相加，最后输出结果dst。

首先，我们需要将向量a0和b0的值存储到变量src0和src1中。然后，我们使用一些数学变换来计算变换后的向量值。

然后，我们再次使用_mm256_fmadd_ps函数，将向量src0和src1相加，并将结果存储在变量sum3中。

最后，我们使用_mm256_add_ps函数，将变量sum3的值存储到变量dst中。


```cpp
#if defined(__FMA__)
            sum0 = _mm256_fmadd_ps(a0, b0, sum0);
            sum1 = _mm256_fmadd_ps(a1, b1, sum1);
            sum2 = _mm256_fmadd_ps(a2, b2, sum2);
            sum3 = _mm256_fmadd_ps(a3, b3, sum3);
#else
            sum0 = _mm256_add_ps(_mm256_mul_ps(a0, b0), sum0);
            sum1 = _mm256_add_ps(_mm256_mul_ps(a1, b1), sum1);
            sum2 = _mm256_add_ps(_mm256_mul_ps(a2, b2), sum2);
            sum3 = _mm256_add_ps(_mm256_mul_ps(a3, b3), sum3);
#endif
        }
        dst[i] = reduce_vector8_0(_mm256_add_ps(_mm256_add_ps(sum0, sum1), _mm256_add_ps(sum2, sum3)));

        for (int j = ncols32; j < ncols; j++) {
            dst[i] += src0[i*ncols + j]*src1[j];
        }
    }
}

```

这段代码定义了一个名为fp32_from_bits的函数，接受一个32位无符号浮点数的参数w。

函数实现：首先，根据定义的操作系统或硬件环境，选择不同的实现方式。如果定义的是OpenGL版本，则直接使用as_float函数。如果定义的是CUDA版本，则使用__uint_as_float函数，将w转换为无符号整数并将其转换为浮点数。如果定义的是INTEL_COMPILER，则使用_castu32_f32函数，将w转换为无符号整数并将其转换为浮点数。如果定义的是_MSC_VER，且定义了ARM或ARM64架构，则使用_CopyFloatFromInt32函数，将w转换为无符号整数并将其转换为浮点数。否则，创建一个名为fp32的联合体类型，将w的位作为as_bits，将w的值作为as_value，然后返回as_value。

这段代码的作用是实现了一个将32位无符号浮点数转换为浮点数的函数，根据不同的操作系统或硬件环境选择不同的实现方式。


```cpp
//
// SIMD with 8 16-bit floats
//

static inline float fp32_from_bits(uint32_t w) {
#if defined(__OPENCL_VERSION__)
    return as_float(w);
#elif defined(__CUDA_ARCH__)
    return __uint_as_float((unsigned int) w);
#elif defined(__INTEL_COMPILER)
    return _castu32_f32(w);
#elif defined(_MSC_VER) && (defined(_M_ARM) || defined(_M_ARM64))
    return _CopyFloatFromInt32((__int32) w);
#else
    union {
        uint32_t as_bits;
        float as_value;
    } fp32 = { w };
    return fp32.as_value;
```

这段代码是一个C语言代码中的一个预处理指令，它通过将一个float类型的变量f转换为整数类型来将其用于某些逻辑操作。以下是代码的作用解释：

1. 如果定义了`__OPENCL_VERSION__`，则使用as_uint函数将float类型转换为整数类型。
2. 如果定义了`__CUDA_ARCH__`，则使用__float_as_uint函数将float类型转换为整数类型。
3. 如果定义了`__INTEL_COMPILER`，则使用_castf32_u32函数将float类型转换为整数类型。
4. 如果定义了`_MSC_VER`，并且定义了`_M_ARM`或者`_M_ARM64`，则使用_CopyInt32FromFloat函数将float类型转换为整数类型。
5. 如果定义了`#elif defined(__INTEL_COMPILER)`，则使用_castf32_u32函数将float类型转换为整数类型。
6. 如果定义了`#elif defined(__OPENCL_VERSION__)`，则使用as_uint函数将float类型转换为整数类型。
7. 如果定义了`#elif defined(__CUDA_ARCH__)`，则使用__float_as_uint函数将float类型转换为整数类型。
8. 如果定义了`#elif defined(__INTEL_COMPILER)`，则使用_castf32_u32函数将float类型转换为整数类型。
9. 如果定义了`#elif defined(__MSC_VER)`，并且定义了`_M_ARM`或者`_M_ARM64`，则使用_CopyInt32FromFloat函数将float类型转换为整数类型。
10. 如果定义了`#else`，则使用一个名为`fp32`的float类型，其中`as_value`为float类型，`as_bits`为32位整数类型。
11. 使用`fp32_to_bits`函数将float类型的变量f转换为32位整数类型的变量。


```cpp
#endif
}

static inline uint32_t fp32_to_bits(float f) {
#if defined(__OPENCL_VERSION__)
	return as_uint(f);
#elif defined(__CUDA_ARCH__)
	return (uint32_t) __float_as_uint(f);
#elif defined(__INTEL_COMPILER)
	return _castf32_u32(f);
#elif defined(_MSC_VER) && (defined(_M_ARM) || defined(_M_ARM64))
	return (uint32_t) _CopyInt32FromFloat(f);
#else
	union {
		float as_value;
		uint32_t as_bits;
	} fp32 = { f };
	return fp32.as_bits;
```

This is a description of a mathematical operation involving an exponentiation function. The operation takes a single-precision floating-point number as input and produces a single-precision floating-point number as output.

The input number is first adjusted by subtracting the maximum value of 0x1F and adding the maximum value of 0xE0, resulting in an adjusted value in the range [0, 0x1F]. The adjusted value is then used as the input to the exponentiation function.

The exponentiation function produces the final result by raising the input number to the power specified in the Exponent field of a泉式寄存器。 The final result is a single-precision floating-point number that represents the original input number.

The exp\_offset variable is determined by subtracting 0x1F from 0xE0 and then shifting the result to the least significant byte of the resulting value. This offset is applied to the exponentiation function so that it takes effect only on the most significant bit of the input number.


```cpp
#endif
}

/*
 * Convert a 16-bit floating-point number in IEEE half-precision format, in bit representation, to
 * a 32-bit floating-point number in IEEE single-precision format.
 *
 * @note The implementation relies on IEEE-like (no assumption about rounding mode and no operations on denormals)
 * floating-point operations and bitcasts between integer and floating-point variables.
 */
static inline float fp16_ieee_to_fp32_value(uint16_t h) {
    /*
     * Extend the half-precision floating-point number to 32 bits and shift to the upper part of the 32-bit word:
     *      +---+-----+------------+-------------------+
     *      | S |EEEEE|MM MMMM MMMM|0000 0000 0000 0000|
     *      +---+-----+------------+-------------------+
     * Bits  31  26-30    16-25            0-15
     *
     * S - sign bit, E - bits of the biased exponent, M - bits of the mantissa, 0 - zero bits.
     */
    const uint32_t w = (uint32_t) h << 16;
    /*
     * Extract the sign of the input number into the high bit of the 32-bit word:
     *
     *      +---+----------------------------------+
     *      | S |0000000 00000000 00000000 00000000|
     *      +---+----------------------------------+
     * Bits  31                 0-31
     */
    const uint32_t sign = w & UINT32_C(0x80000000);
    /*
     * Extract mantissa and biased exponent of the input number into the high bits of the 32-bit word:
     *
     *      +-----+------------+---------------------+
     *      |EEEEE|MM MMMM MMMM|0 0000 0000 0000 0000|
     *      +-----+------------+---------------------+
     * Bits  27-31    17-26            0-16
     */
    const uint32_t two_w = w + w;

    /*
     * Shift mantissa and exponent into bits 23-28 and bits 13-22 so they become mantissa and exponent
     * of a single-precision floating-point number:
     *
     *       S|Exponent |          Mantissa
     *      +-+---+-----+------------+----------------+
     *      |0|000|EEEEE|MM MMMM MMMM|0 0000 0000 0000|
     *      +-+---+-----+------------+----------------+
     * Bits   | 23-31   |           0-22
     *
     * Next, there are some adjustments to the exponent:
     * - The exponent needs to be corrected by the difference in exponent bias between single-precision and half-precision
     *   formats (0x7F - 0xF = 0x70)
     * - Inf and NaN values in the inputs should become Inf and NaN values after conversion to the single-precision number.
     *   Therefore, if the biased exponent of the half-precision input was 0x1F (max possible value), the biased exponent
     *   of the single-precision output must be 0xFF (max possible value). We do this correction in two steps:
     *   - First, we adjust the exponent by (0xFF - 0x1F) = 0xE0 (see exp_offset below) rather than by 0x70 suggested
     *     by the difference in the exponent bias (see above).
     *   - Then we multiply the single-precision result of exponent adjustment by 2**(-112) to reverse the effect of
     *     exponent adjustment by 0xE0 less the necessary exponent adjustment by 0x70 due to difference in exponent bias.
     *     The floating-point multiplication hardware would ensure than Inf and NaN would retain their value on at least
     *     partially IEEE754-compliant implementations.
     *
     * Note that the above operations do not handle denormal inputs (where biased exponent == 0). However, they also do not
     * operate on denormal inputs, and do not produce denormal results.
     */
    const uint32_t exp_offset = UINT32_C(0xE0) << 23;
```

This is a function that performs a conversion of a denormalized half-precision number to a single-precision number. The function takes an input number in binary format, normalizes it by dividing it by 2**23, and then adjusts the bias to account for the input number's exponent.

The function has two variants of operation:

1.  The first variant, which is the denormalization operation, converts the input number to a denormalized number by subtracting 0.5 from it, and then returns the resulting denormalized number.
2.  The second variant, which is the normalization operation, first converts the input number to a normalized number by dividing it by 2**23, and then adjusts the bias to account for the input number's exponent. This is done by first choosing the lower limit of the input number's exponent as the magic cutoff and using this value to determine the sign of the normalized value. If the input number's exponent is lower than the magic cutoff, the input number is considered to be a denormal number and the function returns the denormalized number. Otherwise, the function returns the normalized number.

The function uses a variable `denormalized_value` to store the result of the normalization operation, as well as a variable `magic_bias` to store the bias for the normalized value. The variable `magic_mask` stores the magic cutoff for the exponent as a 32-bit unsigned integer.


```cpp
#if defined(__STDC_VERSION__) && (__STDC_VERSION__ >= 199901L) || defined(__GNUC__) && !defined(__STRICT_ANSI__)
    const float exp_scale = 0x1.0p-112f;
#else
    const float exp_scale = fp32_from_bits(UINT32_C(0x7800000));
#endif
    const float normalized_value = fp32_from_bits((two_w >> 4) + exp_offset) * exp_scale;

    /*
     * Convert denormalized half-precision inputs into single-precision results (always normalized).
     * Zero inputs are also handled here.
     *
     * In a denormalized number the biased exponent is zero, and mantissa has on-zero bits.
     * First, we shift mantissa into bits 0-9 of the 32-bit word.
     *
     *                  zeros           |  mantissa
     *      +---------------------------+------------+
     *      |0000 0000 0000 0000 0000 00|MM MMMM MMMM|
     *      +---------------------------+------------+
     * Bits             10-31                0-9
     *
     * Now, remember that denormalized half-precision numbers are represented as:
     *    FP16 = mantissa * 2**(-24).
     * The trick is to construct a normalized single-precision number with the same mantissa and thehalf-precision input
     * and with an exponent which would scale the corresponding mantissa bits to 2**(-24).
     * A normalized single-precision floating-point number is represented as:
     *    FP32 = (1 + mantissa * 2**(-23)) * 2**(exponent - 127)
     * Therefore, when the biased exponent is 126, a unit change in the mantissa of the input denormalized half-precision
     * number causes a change of the constructud single-precision number by 2**(-24), i.e. the same ammount.
     *
     * The last step is to adjust the bias of the constructed single-precision number. When the input half-precision number
     * is zero, the constructed single-precision number has the value of
     *    FP32 = 1 * 2**(126 - 127) = 2**(-1) = 0.5
     * Therefore, we need to subtract 0.5 from the constructed single-precision number to get the numerical equivalent of
     * the input half-precision number.
     */
    const uint32_t magic_mask = UINT32_C(126) << 23;
    const float magic_bias = 0.5f;
    const float denormalized_value = fp32_from_bits((two_w >> 17) | magic_mask) - magic_bias;

    /*
     * - Choose either results of conversion of input as a normalized number, or as a denormalized number, depending on the
     *   input exponent. The variable two_w contains input exponent in bits 27-31, therefore if its smaller than 2**27, the
     *   input is either a denormal number, or zero.
     * - Combine the result of conversion of exponent and mantissa with the sign of the input number.
     */
    const uint32_t denormalized_cutoff = UINT32_C(1) << 27;
    const uint32_t result = sign |
        (two_w < denormalized_cutoff ? fp32_to_bits(denormalized_value) : fp32_to_bits(normalized_value));
    return fp32_from_bits(result);
}

```

This is a function definition for a floating-point number conversion function from afloat fp to an int int_t. The function takes an fp32_t data type argument and returns an int_t value.

The function starts by calculating a baseline value for the float scale, which represents 0.125 fp to 0.0. The function then determines the scale to zero value, which represents 0.0 to 0.0. If the `FP_DOUBLE` data type is specified, the function uses the lower 32 bits for the baseline calculation and the upper 56 bits for the scale calculation.

The function then determines the sign of the fp value and adds 0x16 to it. This gives the correct sign bit for the scale calculation. Next, the function determines the number of significant bits for the fp value and extracts them. Finally, the function returns the sign-extended value, which includes the sign bit and all significant bits, wrapped around from 0 to UINT16_C(0x7E00) if it is a positive fp value or wrapped around from 0 to UINT16_C(0x7E00) if it is a negative fp value.


```cpp
/*
 * Convert a 32-bit floating-point number in IEEE single-precision format to a 16-bit floating-point number in
 * IEEE half-precision format, in bit representation.
 *
 * @note The implementation relies on IEEE-like (no assumption about rounding mode and no operations on denormals)
 * floating-point operations and bitcasts between integer and floating-point variables.
 */
static inline uint16_t fp16_ieee_from_fp32_value(float f) {
#if defined(__STDC_VERSION__) && (__STDC_VERSION__ >= 199901L) || defined(__GNUC__) && !defined(__STRICT_ANSI__)
    const float scale_to_inf = 0x1.0p+112f;
    const float scale_to_zero = 0x1.0p-110f;
#else
    const float scale_to_inf = fp32_from_bits(UINT32_C(0x77800000));
    const float scale_to_zero = fp32_from_bits(UINT32_C(0x08800000));
#endif
    float base = (fabsf(f) * scale_to_inf) * scale_to_zero;

    const uint32_t w = fp32_to_bits(f);
    const uint32_t shl1_w = w + w;
    const uint32_t sign = w & UINT32_C(0x80000000);
    uint32_t bias = shl1_w & UINT32_C(0xFF000000);
    if (bias < UINT32_C(0x71000000)) {
        bias = UINT32_C(0x71000000);
    }

    base = fp32_from_bits((bias >> 1) + UINT32_C(0x07800000)) + base;
    const uint32_t bits = fp32_to_bits(base);
    const uint32_t exp_bits = (bits >> 13) & UINT32_C(0x00007C00);
    const uint32_t mantissa_bits = bits & UINT32_C(0x00000FFF);
    const uint32_t nonsign = exp_bits + mantissa_bits;
    return (sign >> 16) | (shl1_w > UINT32_C(0xFF000000) ? UINT16_C(0x7E00) : nonsign);
}

```

这段代码是一个matmul类型的函数，名为"mul_mat_vec_f16_0"，其主要作用是实现两个矩阵的乘积。

该函数接收三个参数：

1. "src0"和"src1"是指向同一行不同列的16位整数的指针，作为输入数据。
2. "dst"是一个3D浮点数数组，用于存储结果数据，大小为nrows行和ncols列。
3. nrows和ncols参数分别表示输入和输出矩阵的行列数。

函数内部实现的具体步骤如下：

1. 计算输入两个矩阵的每一行的并，使用_mm256_setzero_ps初始化为0，并保存到sum中。
2. 计算两个输入矩阵的每一行的8个分量的乘积，分别使用_mm256_cvtph_ps将两个8位整数的指针转换为浮点数，然后进行乘法运算，并将结果保存到对应的元素中。这里使用了SSE（Simplified Sequential Executing）下的向量点运算。
3. 将计算得到的8个分量的和保存到dst数组的对应行中。

最终函数返回dst数组，即乘积的结果数据。


```cpp
void mul_mat_vec_f16_0(
    const uint16_t * src0,
    const uint16_t * src1,
             float * dst,
    int nrows,
    int ncols) {

    const int ncols8 = ncols & ~7;

    for (int i = 0; i < nrows; i++) {
        __m256 sum = _mm256_setzero_ps();

        const uint16_t * src0_row = src0 + i * ncols;
        for (int j = 0; j < ncols8; j += 8) {
            __m256 a = _mm256_cvtph_ps(_mm_loadu_si128((__m128i*)(src0_row + j)));
            __m256 b = _mm256_cvtph_ps(_mm_loadu_si128((__m128i*)(src1 + j)));
```

This function multiplies a matrix of f16 floating-point numbers by a vector of f16 floating-point numbers and stores the result in a dense vector of f16 floating-point numbers.

The input matrices `src0` and `src1` are passed through the function multiplied by 16 to divide each element by 8. The resulting multiplication is performed by summing the products of each element of the input matrix by the corresponding element of the weight matrix.

The function uses SIMD instructions to perform the multiplication across multiple elements in parallel, which allows for a significant speedup compared to a simple forward or backward multiplication.


```cpp
#if defined(__FMA__)
            sum = _mm256_fmadd_ps(a, b, sum);
#else
            sum = _mm256_add_ps(_mm256_mul_ps(a, b), sum);
#endif
        }
        dst[i] = reduce_vector8_0(sum);

        for (int j = ncols8; j < ncols; j++) {
            dst[i] += fp16_ieee_to_fp32_value(src0_row[j]) * fp16_ieee_to_fp32_value(src1[j]);
        }
    }
}

void mul_mat_vec_f16_1(
    const uint16_t * src0,
    const uint16_t * src1,
             float * dst,
    int nrows,
    int ncols) {

    const int ncols16 = ncols & ~15;

    for (int i = 0; i < nrows; i++) {
        __m256 sum0 = _mm256_setzero_ps();
        __m256 sum1 = _mm256_setzero_ps();

        const uint16_t * src0_row = src0 + i * ncols;
        for (int j = 0; j < ncols16; j += 16) {
            __m256 a0 = _mm256_cvtph_ps(_mm_loadu_si128((__m128i*)(src0_row + j + 0)));
            __m256 a1 = _mm256_cvtph_ps(_mm_loadu_si128((__m128i*)(src0_row + j + 8)));
            __m256 b0 = _mm256_cvtph_ps(_mm_loadu_si128((__m128i*)(src1 + j)));
            __m256 b1 = _mm256_cvtph_ps(_mm_loadu_si128((__m128i*)(src1 + j + 8)));
```

这段代码的作用是实现向量加法操作，其中使用到了FPAA贲拿冯·法。具体来说，代码 checks whether __FMA__ 是否被定义，如果是，则执行功能 1，即对两个向量 a0 和 b0 进行加法操作，并求和得到结果 sum0，同时使用 _mm256_fmadd_ps 函数实现。如果不是，则执行功能 2，即对两个向量 a0 和 b0 分别执行加法操作，并求和得到结果 sum0，同时使用 _mm256_add_ps 函数实现。最后，将两个结果相加，得到向量 dst[i] 的值。


```cpp
#if defined(__FMA__)
            sum0 = _mm256_fmadd_ps(a0, b0, sum0);
            sum1 = _mm256_fmadd_ps(a1, b1, sum1);
#else
            sum0 = _mm256_add_ps(_mm256_mul_ps(a0, b0), sum0);
            sum1 = _mm256_add_ps(_mm256_mul_ps(a1, b1), sum1);
#endif
        }
        dst[i] = reduce_vector8_0(sum0) + reduce_vector8_0(sum1);

        for (int j = ncols16; j < ncols; j++) {
            dst[i] += fp16_ieee_to_fp32_value(src0_row[j]) * fp16_ieee_to_fp32_value(src1[j]);
        }
    }
}

```

It looks like you are trying to compute the sum of a set of row-major integer sources. Each source is a 32-element row-major integer array, and the order of the elements in each row is important.

To compute the sum, you can use a loop to iterate over each element of the first source, and then compute the sum using row-major indexing.

Here's some sample code to illustrate how this might work:
```cpp
#include <immintrin.h>

void row_major_sum(int sources[], int nrows, int ncols) {
   int sum[256];
   int i, j;

   for (i = 0; i < nrows; i++) {
       sum[i] = 0;
       for (j = 0; j < ncols; j++) {
           sum[i] += sources[i * ncols + j];
       }
   }

   for (i = 0; i < nrows; i++) {
       sum[i] = 0;
       for (j = 0; j < ncols; j++) {
           sum[i] += sources[i * ncols + j];
       }
   }

   // row-major sum
   for (i = 0; i < nrows; i++) {
       sum[i] = _mm_sub_ps(sum[i], sum[i + 8 / 2], _mm_set_ps(0));
       sum[i] = _mm_sub_ps(sum[i], sum[i + 16 / 2], _mm_set_ps(0));
   }
}

int main() {
   int sources[10] = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};
   int nrows = 5;
   int ncols = 32;

   row_major_sum(sources, nrows, ncols);

   printf("%zu\n", sources[0]);

   return 0;
}
```
This code will output `1`, which is the sum of the first source.

I hope this helps! Let me know if


```cpp
void mul_mat_vec_f16_2(
    const uint16_t * src0,
    const uint16_t * src1,
             float * dst,
    int nrows,
    int ncols) {

    const int ncols32 = ncols & ~31;

    for (int i = 0; i < nrows; i++) {
        __m256 sum0 = _mm256_setzero_ps();
        __m256 sum1 = _mm256_setzero_ps();
        __m256 sum2 = _mm256_setzero_ps();
        __m256 sum3 = _mm256_setzero_ps();

        const uint16_t * src0_row = src0 + i * ncols;
        for (int j = 0; j < ncols32; j += 32) {
            __m256 a0 = _mm256_cvtph_ps(_mm_loadu_si128((__m128i*)(src0_row + j + 0)));
            __m256 a1 = _mm256_cvtph_ps(_mm_loadu_si128((__m128i*)(src0_row + j + 8)));
            __m256 a2 = _mm256_cvtph_ps(_mm_loadu_si128((__m128i*)(src0_row + j + 16)));
            __m256 a3 = _mm256_cvtph_ps(_mm_loadu_si128((__m128i*)(src0_row + j + 24)));
            __m256 b0 = _mm256_cvtph_ps(_mm_loadu_si128((__m128i*)(src1 + j)));
            __m256 b1 = _mm256_cvtph_ps(_mm_loadu_si128((__m128i*)(src1 + j + 8)));
            __m256 b2 = _mm256_cvtph_ps(_mm_loadu_si128((__m128i*)(src1 + j + 16)));
            __m256 b3 = _mm256_cvtph_ps(_mm_loadu_si128((__m128i*)(src1 + j + 24)));
```

这段代码是一个矩阵运算的库函数，主要作用是对给定的三行输入矩阵a和b，输出结果矩阵dst。

具体来说，代码会对于每一行i，对输入的sum0、sum1、sum2和sum3进行求和运算，并将结果加入dst的对应行中。对于每一行，代码还会对于该行所有的元素，将输入的fp16_ieee_to_fp32_value函数计算得到的结果，与该行的对应元素相乘，再将结果加入dst的对应行中。

最终，dst的值就是对输入的三行矩阵a和b计算得到的结果。


```cpp
#if defined(__FMA__)
            sum0 = _mm256_fmadd_ps(a0, b0, sum0);
            sum1 = _mm256_fmadd_ps(a1, b1, sum1);
            sum2 = _mm256_fmadd_ps(a2, b2, sum2);
            sum3 = _mm256_fmadd_ps(a3, b3, sum3);
#else
            sum0 = _mm256_add_ps(_mm256_mul_ps(a0, b0), sum0);
            sum1 = _mm256_add_ps(_mm256_mul_ps(a1, b1), sum1);
            sum2 = _mm256_add_ps(_mm256_mul_ps(a2, b2), sum2);
            sum3 = _mm256_add_ps(_mm256_mul_ps(a3, b3), sum3);
#endif
        }
        dst[i] = reduce_vector8_0(sum0) + reduce_vector8_0(sum1) + reduce_vector8_0(sum2) + reduce_vector8_0(sum3);

        for (int j = ncols32; j < ncols; j++) {
            dst[i] += fp16_ieee_to_fp32_value(src0_row[j]) * fp16_ieee_to_fp32_value(src1[j]);
        }
    }
}

```

首先，将输入的DST矩阵按照列进行合并，即：

`dst`  `dst[i][(i+8)%8]`  `dst[(i+8)%8][(i+16)%8]`  `dst[(i+16)%8][(i+24)%8]`

然后，对于每一行，从左往右扫描列中的元素，计算出与当前元素相邻的前32个元素的和，并将它们存储到`sum0`、`sum1`、`sum2`、`sum3`中。

注意，这里计算的是8位的元素，所以需要将列宽（即8）转换为32位，并在计算过程中对齐到32位。

最后，将`sum0`、`sum1`、`sum2`、`sum3`分别与前32位乘以8（即`dst[i][0]`、`dst[i][1]`、`dst[i][2]`、`dst[i][3]`）并相加，得到最终的`dst[i][(i+8)%8]`。

完整实现如下：


```cpp
void mul_mat_vec_f16_3(
    const uint16_t * src0,
    const    float * src1,
             float * dst,
    int nrows,
    int ncols) {

    const int ncols32 = ncols & ~31;

    for (int i = 0; i < nrows; i++) {
        __m256 sum0 = _mm256_setzero_ps();
        __m256 sum1 = _mm256_setzero_ps();
        __m256 sum2 = _mm256_setzero_ps();
        __m256 sum3 = _mm256_setzero_ps();

        const uint16_t * src0_row = src0 + i * ncols;
        for (int j = 0; j < ncols32; j += 32) {
            __m256 a0 = _mm256_cvtph_ps(_mm_loadu_si128((__m128i*)(src0_row + j + 0)));
            __m256 a1 = _mm256_cvtph_ps(_mm_loadu_si128((__m128i*)(src0_row + j + 8)));
            __m256 a2 = _mm256_cvtph_ps(_mm_loadu_si128((__m128i*)(src0_row + j + 16)));
            __m256 a3 = _mm256_cvtph_ps(_mm_loadu_si128((__m128i*)(src0_row + j + 24)));
            __m256 b0 = _mm256_loadu_ps(src1 + j);
            __m256 b1 = _mm256_loadu_ps(src1 + j + 8);
            __m256 b2 = _mm256_loadu_ps(src1 + j + 16);
            __m256 b3 = _mm256_loadu_ps(src1 + j + 24);
```

这段代码是一个用于对给定的向量进行矩阵加法和向量减法的FFT算法的实现。以下是对代码中关键部分的解释：

```cppc
#if defined(__FMA__)
```

这是一个条件判断语句，只有在定义了`__FMA__`时才会执行下面的代码。如果定义了`__FMA__`，那么后面的代码将计算三个向量的和，并将其存储在`dst`数组中。

```cppc
           sum0 = _mm256_fmadd_ps(a0, b0, sum0);
           sum1 = _mm256_fmadd_ps(a1, b1, sum1);
           sum2 = _mm256_fmadd_ps(a2, b2, sum2);
           sum3 = _mm256_fmadd_ps(a3, b3, sum3);
```

这段代码计算三个输入向量的和，并将其存储在`sum0`，`sum1`和`sum2`中。`_mm256_fmadd_ps`是一个FFT算法的数学函数，用于执行矩阵加法和计算。`a0`，`b0`，`a1`，`b1`和`a2`是输入向量的索引，`sum0`，`sum1`和`sum2`是计算得到的和。

```cppc
#else
```

这是一个否则语句，如果没有定义`__FMA__`，那么这段代码将会计算每个输入向量的和，并将它们存储在`sum0`，`sum1`和`sum2`中。

```cppc
           sum0 = _mm256_add_ps(_mm256_mul_ps(a0, b0), sum0);
           sum1 = _mm256_add_ps(_mm256_mul_ps(a1, b1), sum1);
           sum2 = _mm256_add_ps(_mm256_mul_ps(a2, b2), sum2);
           sum3 = _mm256_add_ps(_mm256_mul_ps(a3, b3), sum3);
```

这段代码计算每个输入向量的和，并将其存储在`sum0`，`sum1`和`sum2`中。`_mm256_add_ps`和`_mm256_mul_ps`是FFT算法的数学函数，用于执行矩阵加法和计算。`a0`，`b0`，`a1`，`b1`和`a2`是输入向量的索引，`sum0`，`sum1`和`sum2`是计算得到的和。

```cppc
       }
       dst[i] = reduce_vector8_0(sum0) + reduce_vector8_0(sum1) + reduce_vector8_0(sum2) + reduce_vector8_0(sum3);
```

这段代码将计算得到的和存储在`dst`数组中。`_mm256_fmadd_ps`用于执行矩阵加法，`_mm256_fmadd_ps`用于执行矩阵加法，`reduce_vector8_0`用于将输入向量转化为8位整数类型的向量。

```cppc
       for (int j = ncols32; j < ncols; j++) {
           dst[i] += fp16_ieee_to_fp32_value(src0_row[j]) * fp16_ieee_to_fp32_value(src1[j]);
       }
```

这段代码使用`fp16_ieee_to_fp32_value`函数将输入向量中的浮点数转换为32位整数，然后将其乘以输入向量中的浮点数。`src0_row[j]`和`src1[j]`是输入向量的索引，`fp16_ieee_to_fp32_value`用于将浮点数转换为整数。

```cppc
       }
```

这是一个结束标签，表明这段代码已经结束。


```cpp
#if defined(__FMA__)
            sum0 = _mm256_fmadd_ps(a0, b0, sum0);
            sum1 = _mm256_fmadd_ps(a1, b1, sum1);
            sum2 = _mm256_fmadd_ps(a2, b2, sum2);
            sum3 = _mm256_fmadd_ps(a3, b3, sum3);
#else
            sum0 = _mm256_add_ps(_mm256_mul_ps(a0, b0), sum0);
            sum1 = _mm256_add_ps(_mm256_mul_ps(a1, b1), sum1);
            sum2 = _mm256_add_ps(_mm256_mul_ps(a2, b2), sum2);
            sum3 = _mm256_add_ps(_mm256_mul_ps(a3, b3), sum3);
#endif
        }
        dst[i] = reduce_vector8_0(sum0) + reduce_vector8_0(sum1) + reduce_vector8_0(sum2) + reduce_vector8_0(sum3);

        for (int j = ncols32; j < ncols; j++) {
            dst[i] += fp16_ieee_to_fp32_value(src0_row[j]) * fp16_ieee_to_fp32_value(src1[j]);
        }
    }
}

```

This is a C++ function named "add_matrix_vector_with_method" that performs a matrix multiplication operation on two one-dimensional vectors and adds the result to a specified destination vector. The function supports three different matrix vector multiplication methods and performs these operations using either 64 single-precision floating-point values for the f32 data type, 16 single-precision floating-point values for the f16 data type, or 32 single-precision floating-point values for the f16 data type.

The function takes as input a source vector (src0 and src1) and a destination vector (dst) of the same data type as src0 and src1, as well as a number of rows (N) and a number of columns (M) for the matrix multiplication operation. The function also takes as input an optional destination vector specifier (method), which is an array of methods to use for the matrix vector multiplication operation, and a source vector specifier (method), which is an array of methods to use for the matrix vector multiplication operation.

The function first sets the destination vector to zero and then performs the matrix multiplication based on the specified method. If the specified method is 0, the function performs a multiplication using the default method, which adds the elements of src0 and src1 to the destination vector. If the specified method is 1 or 2, the function performs a multiplication using the specified method, which adds the elements of src0 and src1 to the destination vector using the specified number of multiplications. If the specified method is 3, the function performs a multiplication using the specified method for the first vector but does not perform any additional multiplications for the second vector. If the specified method is 4 or 5, the function performs a multiplication using the specified method for the first vector and adds the elements of the second vector to the destination vector using the specified number of multiplications. If the specified method is 6, the function performs a multiplication using the specified method for the first vector and adds the elements of the second vector to the destination vector using the specified number of multiplications.

The function then adds the elements of the destination vector to the specified source vector and returns the result.

The function uses the clock() function to measure the elapsed time for the matrix multiplication operation, and also uses the get_time_us() function to measure the elapsed time in microseconds. The function prints the elapsed time for the matrix multiplication operation in both its original and microseconds versions.


```cpp
uint64_t get_time_us(void) {
    struct timeval tv;
    gettimeofday(&tv, NULL);
    return tv.tv_sec * 1000000 + tv.tv_usec;
}

int main(int argc, const char ** argv) {
    float * src0 = malloc(sizeof(float)*N*M);
    float * src1 = malloc(sizeof(float)*M);
    float * dst  = malloc(sizeof(float)*N);

    //float * src0 = (float *)(aligned_alloc(64, sizeof(float)*N*M));
    //float * src1 = (float *)(aligned_alloc(64, sizeof(float)*M));
    //float * dst  = (float *)(aligned_alloc(64, sizeof(float)*N));

    for (int i = 0; i < N*M; i++) {
        src0[i] = rand() / (float)RAND_MAX;
    }

    for (int i = 0; i < M; i++) {
        src1[i] = rand() / (float)RAND_MAX;
    }

    // convert src0 and src1 to __fp16
    uint16_t * src0_fp16 = (uint16_t *)(malloc(sizeof(uint16_t)*N*M));
    uint16_t * src1_fp16 = (uint16_t *)(malloc(sizeof(uint16_t)*M));
    //uint16_t * src0_fp16 = (uint16_t *)(aligned_alloc(64, sizeof(uint16_t)*N*M));
    //uint16_t * src1_fp16 = (uint16_t *)(aligned_alloc(64, sizeof(uint16_t)*M));

    {
        const uint64_t t_start = get_time_us();

        for (int i = 0; i < N*M; i++) {
            src0_fp16[i] = fp16_ieee_from_fp32_value(src0[i]);
            //printf("%f %f\n", src0[i], fp16_ieee_to_fp32_value(src0_fp16[i]));
            //assert(!isnan(fp16_ieee_to_fp32_value(src0_fp16[i])));
        }

        for (int i = 0; i < M; i++) {
            src1_fp16[i] = fp16_ieee_from_fp32_value(src1[i]);
        }

        const uint64_t t_end = get_time_us();
        printf("convert time: %f ms\n", (t_end - t_start) / 1000.0);
    }

    for (int i = 0; i < 16; ++i) {
        printf("%f %f\n", src0[i], fp16_ieee_to_fp32_value(src0_fp16[i]));
    }

    int method = 0;
    if (argc > 1) {
        method = atoi(argv[1]);
    }

    const int nIter = 1000;

    const clock_t start = clock();
    const uint64_t start_us = get_time_us();

    double iM = 1.0/M;
    double sum = 0.0f;
    for (int i = 0; i < nIter; i++) {
        if (method == 0) {
            mul_mat_vec_f32_0(src0, src1, dst, N, M);
        }

        if (method == 1) {
            mul_mat_vec_f32_1(src0, src1, dst, N, M);
        }

        if (method == 2) {
            mul_mat_vec_f32_2(src0, src1, dst, N, M);
        }

        if (method == 3) {
            mul_mat_vec_f16_0(src0_fp16, src1_fp16, dst, N, M);
        }

        if (method == 4) {
            mul_mat_vec_f16_1(src0_fp16, src1_fp16, dst, N, M);
        }

        if (method == 5) {
            mul_mat_vec_f16_2(src0_fp16, src1_fp16, dst, N, M);
        }

        if (method == 6) {
            mul_mat_vec_f16_3(src0_fp16, src1, dst, N, M);
        }
    }

    for (int i = 0; i < N; i++) {
        sum += dst[i]*iM;
    }

    {
        const clock_t end = clock();
        const uint64_t end_us = get_time_us();
        printf("%s: elapsed ticks: %ld\n", __func__, end - start);
        printf("%s: elapsed us: %ld\n", __func__, end_us - start_us);
    }

    printf("%f\n", sum);

    free(src0);
    free(src1);
    free(dst);

    free(src0_fp16);
    free(src1_fp16);

    return 0;
}

```

# `tests/test-vec2.c`

这段代码包括以下几个部分：

1. `#include <stdint.h>` 和 `#include <stdio.h>`：这两个头文件是 C 语言标准库中的标准输入输出函数，用于提供输入输出操作的相关接口。

2. `#include <assert.h>`：这个头文件用于在程序运行时进行输入输出调试，以帮助开发人员快速发现和修复程序中的错误。

3. `#include <stdlib.h>`：这个头文件包含标准库中的常用函数，例如字符串处理、内存管理、随机数生成等。

4. `#include <time.h>` 和 `#include <math.h>`：这两个头文件用于提供时间支持和数学计算功能，例如 `time.h` 包含日期和时间的输入输出操作，`math.h` 包含数学计算的支持。

5. `#include <sys/time.h>`：这个头文件提供从操作系统获取当前时间的函数，并支持获取当前时间的计时秒数。

6. `#include <arm_neon.h>`：这个头文件提供从arm微控制器中获取特定寄存器值的函数，它的作用是获取一个32位机型的arm微控制器，通过这个头文件获取寄存器值来实现对微控制器的操作。

7. `const int N = 1 << 12;` 和 `const int M = 1 << 12;`：这两个定义用于定义一个大小为128x8的矩阵，并输出其值，在后续的代码中可能被用来作为输入数据。

8. `const int NUM_PINS = 26;`：这个定义用于定义微控制器上的总针数，可能被用来定义总的输入输出点数。

9. `void pinsToScreen(int pins, int val, int *sw_x, int *sw_y, int *len);`：这个函数将一个int类型的硬件值转换成一个32位软件值，并输出到屏幕上的指定硬件针（sw_x和sw_y），并返回这个int类型变量。

10. `void dfs(int node, int type, int *in, int *out, int len);`：这个函数是深度优先搜索算法的实现，给定一个有向无环图，它会在这个图上以某种步长进行深度优先搜索，并输出这个图的拓扑结构。

11. `void display_matrix(int matrix[N][N], int len);`：这个函数用于显示一个大小为128x8的矩阵，可能作为后续代码的输入数据。


```cpp
#include <stdint.h>
#include <stdio.h>
#include <assert.h>
#include <stdlib.h>
#include <time.h>
#include <math.h>

#include <sys/time.h>

#include <arm_neon.h>

const int N = 1 << 12;
const int M = 1 << 12;

//
```



这段代码定义了一个名为 `mul_mat_vec_f32_0` 的函数，它的输入参数包括一个二维数组 `src0`、一个二维数组 `src1`、一个三维数组 `dst`、数组长度 `nrows` 和列组长度 `ncols`。函数的作用是对输入的 `src0` 和 `src1` 数组进行垂直 multiplication，并将结果存储到 `dst` 数组中。

函数的具体实现可以分为以下几个步骤：

1. 初始化 `sum` 变量为 0。
2. 遍历 `src0` 和 `src1` 数组的每个元素，计算出每个元素的值并累加到 `sum` 中。
3. 将 `sum` 赋值给 `dst` 数组的对应元素。

由于 `src0` 和 `src1` 数组是垂直的，因此函数的实现是线性的，时间复杂度为 $O(nN)$，其中 $n$ 是 `src0` 和 `src1` 数组的长度，$N$ 是 `dst` 数组的长度。


```cpp
// naive implementation
//

void mul_mat_vec_f32_0(
    const float * restrict src0,
    const float * restrict src1,
    float * dst,
    int nrows,
    int ncols) {
    for (int i = 0; i < nrows; i++) {
        float sum = 0.0f;
        for (int j = 0; j < ncols; j++) {
            sum += src0[i*ncols + j]*src1[j];
        }
        dst[i] = sum;
    }
}

```

This is a function that performs a matrix multiplication operation on two 16-bit matrices `src1` and `src2`, and optionally a scaling operation (multiplication by a constant factor) on these matrices. The scaling operation is defined as a function `src2_scaled`, which applies the same scaling operation to `src2` as it does to `src1`:
```cpp
src2_scaled(src2) = src2;
src2_scaled(src1) = src1;
src2_scaled(const int64 x) = src2;
src2_scaled(const float64 x) = src2;
```
The function takes two arguments: `src1` and `src2`, which are 16-bit matrices of column width `ncols`, and optionally a scaling factor `scale`, which is a 16-bit signed integer. The scaling factor is applied element-wise to both `src1` and `src2`, so that the final result has the same scaling factor for all elements.

The function performs a point-to-point matrix multiplication of `src1` and `src2` by means of a function that computes the element-wise product of the corresponding elements of these matrices. This is done by computing the sum of the element-wise product of the corresponding elements of `src1` and `src2`, and dividing that by the element-wise product of the column widths of the matrices.

The function also performs a loop-based multiplication of `src1` by the scaling factor, if specified. This is done by computing the element-wise product of the corresponding elements of `src1` and `scale`, and dividing that by the element-wise product of the column widths of `src1`.

The function returns the final result as a 16-bit signed integer `dst`.


```cpp
void mul_mat_vec_f16_0(
    const __fp16 * src0,
    const __fp16 * src1,
           float * dst,
    int nrows,
    int ncols) {

    const int n64 = ncols & ~63;

    for (int r = 0; r < nrows; r++) {
        float sumf = 0.0;

        float16x8_t sum0 = vdupq_n_f16(0.0f);
        float16x8_t sum1 = vdupq_n_f16(0.0f);
        float16x8_t sum2 = vdupq_n_f16(0.0f);
        float16x8_t sum3 = vdupq_n_f16(0.0f);
        float16x8_t sum4 = vdupq_n_f16(0.0f);
        float16x8_t sum5 = vdupq_n_f16(0.0f);
        float16x8_t sum6 = vdupq_n_f16(0.0f);
        float16x8_t sum7 = vdupq_n_f16(0.0f);

        float16x8_t x0, x1, x2, x3, x4, x5, x6, x7;
        float16x8_t y0, y1, y2, y3, y4, y5, y6, y7;

        const __fp16 * restrict p0 = src0 + r*ncols;

        for (int i = 0; i < n64; i += 64) {
            x0 = vld1q_f16(p0 + i + 0 );
            x1 = vld1q_f16(p0 + i + 8 );
            x2 = vld1q_f16(p0 + i + 16);
            x3 = vld1q_f16(p0 + i + 24);
            x4 = vld1q_f16(p0 + i + 32);
            x5 = vld1q_f16(p0 + i + 40);
            x6 = vld1q_f16(p0 + i + 48);
            x7 = vld1q_f16(p0 + i + 56);

            y0 = vld1q_f16(src1 + i + 0 );
            y1 = vld1q_f16(src1 + i + 8 );
            y2 = vld1q_f16(src1 + i + 16);
            y3 = vld1q_f16(src1 + i + 24);
            y4 = vld1q_f16(src1 + i + 32);
            y5 = vld1q_f16(src1 + i + 40);
            y6 = vld1q_f16(src1 + i + 48);
            y7 = vld1q_f16(src1 + i + 56);

            sum0 = vfmaq_f16(sum0, x0, y0);
            sum1 = vfmaq_f16(sum1, x1, y1);
            sum2 = vfmaq_f16(sum2, x2, y2);
            sum3 = vfmaq_f16(sum3, x3, y3);
            sum4 = vfmaq_f16(sum4, x4, y4);
            sum5 = vfmaq_f16(sum5, x5, y5);
            sum6 = vfmaq_f16(sum6, x6, y6);
            sum7 = vfmaq_f16(sum7, x7, y7);
        }

        // TODO: F16 - better way to reduce this ?
        float16x8_t sum = vaddq_f16(sum0, sum1);

        sum = vaddq_f16(sum, sum2);
        sum = vaddq_f16(sum, sum3);
        sum = vaddq_f16(sum, sum4);
        sum = vaddq_f16(sum, sum5);
        sum = vaddq_f16(sum, sum6);
        sum = vaddq_f16(sum, sum7);

        sumf += sum[0] + sum[1] + sum[2] + sum[3] + sum[4] + sum[5] + sum[6] + sum[7];

        for (int j = n64; j < n64; j++) {
            sumf += src0[r*ncols + j]*src1[j];
        }

        dst[r] = sumf;
    }
}

```

This is a FFN that performs a simple sum operation on an array of floating-point numbers. The input to the FFN is a 4-element array of floating-point numbers, and the output is also a 4-element array of floating-point numbers, containing the sum of the input numbers.

The FFN first converts the input numbers to 4-element format, then performs a sum operation on the 4-element blocks, and finally converts the result back to a 4-element format.

The code is well-structured and easy to read.


```cpp
void mul_mat_vec_f16_1(
    const __fp16 * src0,
    const __fp16 * src1,
           float * dst,
    int nrows,
    int ncols) {

    const int n32 = ncols & ~31;

    for (int r = 0; r < nrows; r++) {
        float sumf = 0.0;

        float16x8_t sum0 = vdupq_n_f16(0.0f);
        float16x8_t sum1 = vdupq_n_f16(0.0f);
        float16x8_t sum2 = vdupq_n_f16(0.0f);
        float16x8_t sum3 = vdupq_n_f16(0.0f);

        float16x8_t x0, x1, x2, x3;
        float16x8_t y0, y1, y2, y3;

        const __fp16 * restrict p0 = src0 + r*ncols;

        for (int i = 0; i < n32; i += 32) {
            x0 = vld1q_f16(p0 + i + 0 );
            x1 = vld1q_f16(p0 + i + 8 );
            x2 = vld1q_f16(p0 + i + 16);
            x3 = vld1q_f16(p0 + i + 24);

            y0 = vld1q_f16(src1 + i + 0 );
            y1 = vld1q_f16(src1 + i + 8 );
            y2 = vld1q_f16(src1 + i + 16);
            y3 = vld1q_f16(src1 + i + 24);

            sum0 = vfmaq_f16(sum0, x0, y0);
            sum1 = vfmaq_f16(sum1, x1, y1);
            sum2 = vfmaq_f16(sum2, x2, y2);
            sum3 = vfmaq_f16(sum3, x3, y3);
        }

        // reduce sum0..sum3 to sum0
        sum0 = vaddq_f16(sum0, sum1);
        sum2 = vaddq_f16(sum2, sum3);
        sum0 = vaddq_f16(sum0, sum2);

        // load sum0 into 2 float32x4_t
        float32x4_t sum0f32 = vcvt_f32_f16(vget_low_f16(sum0));
        float32x4_t sum1f32 = vcvt_f32_f16(vget_high_f16(sum0));

        // reduce sum0f32 and sum1f32 to sumf
        sum0f32 = vaddq_f32(sum0f32, sum1f32);

        float32x2_t sumf32 = vadd_f32(vget_low_f32(sum0f32), vget_high_f32(sum0f32));
        sumf = vget_lane_f32(sumf32, 0) + vget_lane_f32(sumf32, 1);

        //sumf = sum0[0] + sum0[1] + sum0[2] + sum0[3] + sum0[4] + sum0[5] + sum0[6] + sum0[7];

        for (int j = n32; j < n32; j++) {
            sumf += src0[r*ncols + j]*src1[j];
        }

        dst[r] = sumf;
    }
}

```

This is a C function that performs matrix multiplication on two 16-bit matrices `src0` and `src1` using different indexing modes. It takes command-line arguments for the indexing mode (0 for default, 1 for only the left matrix, and 2 for only the right matrix).

The function uses the multiplication algorithms available in the Eigen library for high-level matrix multiplication:���采用`inner_product`函数 for float16 matrices, and `s为人工计算浮点数`。函数的实现比较简单，以减少出错的可能性。

输出结果是两个矩阵的乘积。

截至调用该函数的毫秒数。


```cpp
uint64_t get_time_us(void) {
    struct timeval tv;
    gettimeofday(&tv, NULL);
    return tv.tv_sec * 1000000 + tv.tv_usec;
}

int main(int argc, const char ** argv) {
    float * src0 = malloc(sizeof(float)*N*M);
    float * src1 = malloc(sizeof(float)*M);
    float * dst  = malloc(sizeof(float)*N);

    //float * src0 = (float *)(aligned_alloc(64, sizeof(float)*N*M));
    //float * src1 = (float *)(aligned_alloc(64, sizeof(float)*M));
    //float * dst  = (float *)(aligned_alloc(64, sizeof(float)*N));

    for (int i = 0; i < N*M; i++) {
        src0[i] = rand() / (float)RAND_MAX;
    }

    for (int i = 0; i < M; i++) {
        src1[i] = rand() / (float)RAND_MAX;
    }

    // convert src0 and src1 to __fp16
    __fp16 * src0_fp16 = (__fp16 *)(malloc(sizeof(__fp16)*N*M));
    __fp16 * src1_fp16 = (__fp16 *)(malloc(sizeof(__fp16)*M));

    {
        const uint64_t t_start = get_time_us();

        for (int i = 0; i < N*M; i++) {
            src0_fp16[i] = src0[i];
            //printf("%f %f\n", src0[i], src0_fp16[i]);
            //assert(!isnan(src0_fp16[i]));
        }

        for (int i = 0; i < M; i++) {
            src1_fp16[i] = src1[i];
        }

        const uint64_t t_end = get_time_us();
        printf("convert time: %f ms\n", (t_end - t_start) / 1000.0);
    }

    for (int i = 0; i < 16; ++i) {
        printf("%f %f\n", src0[i], src0_fp16[i]);
    }

    int method = 0;
    if (argc > 1) {
        method = atoi(argv[1]);
    }

    const int nIter = 1000;

    const clock_t start = clock();
    const uint64_t start_us = get_time_us();

    double iM = 1.0/M;
    double sum = 0.0f;
    for (int i = 0; i < nIter; i++) {
        if (method == 0) {
            mul_mat_vec_f32_0(src0, src1, dst, N, M);
        }

        if (method == 1) {
            mul_mat_vec_f16_0(src0_fp16, src1_fp16, dst, N, M);
        }

        if (method == 2) {
            mul_mat_vec_f16_1(src0_fp16, src1_fp16, dst, N, M);
        }
    }

    for (int i = 0; i < N; i++) {
        sum += dst[i]*iM;
    }

    {
        const clock_t end = clock();
        const uint64_t end_us = get_time_us();
        printf("%s: elapsed ticks: %ld\n",  __func__, end - start);
        printf("%s: elapsed us:    %llu / %f ms\n",  __func__, end_us - start_us, (end_us - start_us) / 1000.0 / nIter);
    }

    printf("%f\n", sum);

    free(src0);
    free(src1);
    free(dst);

    free(src0_fp16);
    free(src1_fp16);

    return 0;
}

```