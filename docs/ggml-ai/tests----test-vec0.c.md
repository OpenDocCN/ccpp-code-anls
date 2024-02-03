# `ggml\tests\test-vec0.c`

```cpp
#include <stdio.h>  // 包含标准输入输出库
#include <assert.h>  // 包含断言库
#include <stdlib.h>  // 包含标准库
#include <time.h>  // 包含时间库

const int N = 1 << 14;  // 定义常量 N，值为 2 的 14 次方
const int M = 1 << 14;  // 定义常量 M，值为 2 的 14 次方

void mul_mat_vec_f32_0(  // 定义函数 mul_mat_vec_f32_0，实现矩阵和向量的乘法
    const float * src0,  // 参数 src0，指向浮点数的指针，用于存储矩阵数据
    const float * src1,  // 参数 src1，指向浮点数的指针，用于存储向量数据
    float * dst,  // 参数 dst，指向浮点数的指针，用于存储结果向量数据
    unsigned nrows,  // 参数 nrows，无符号整型，表示矩阵的行数
    unsigned ncols) {  // 参数 ncols，无符号整型，表示矩阵的列数
    for (unsigned i = 0; i < nrows; i++) {  // 循环遍历矩阵的行
        float sum = 0.0f;  // 初始化 sum 为 0.0
        for (unsigned j = 0; j < ncols; j++) {  // 循环遍历矩阵的列
            sum += src0[i*ncols + j]*src1[j];  // 计算矩阵和向量的乘积并累加到 sum
        }
        dst[i] = sum;  // 将结果存储到 dst 中
    }
}
#if defined(_MSC_VER)
typedef float __declspec(align(32)) afloat;  // 如果是 MSC 编译器，定义 afloat 类型，要求 32 字节对齐
#else
typedef float afloat __attribute__((__aligned__(32)));  // 如果不是 MSC 编译器，定义 afloat 类型，要求 32 字节对齐
#endif
void mul_mat_vec_f32_1(  // 定义函数 mul_mat_vec_f32_1，实现矩阵和向量的乘法
    const afloat *restrict src0,  // 参数 src0，指向 afloat 类型的指针，用于存储矩阵数据，要求对指针的限定
    const afloat *restrict src1,  // 参数 src1，指向 afloat 类型的指针，用于存储向量数据，要求对指针的限定
    afloat *restrict dst,  // 参数 dst，指向 afloat 类型的指针，用于存储结果向量数据，要求对指针的限定
    unsigned nrows,  // 参数 nrows，无符号整型，表示矩阵的行数
    unsigned ncols) {  // 参数 ncols，无符号整型，表示矩阵的列数
    for (unsigned i = 0; i < nrows; i++) {  // 循环遍历矩阵的行
        const afloat * restrict row = src0 + i*ncols;  // 定义指向矩阵行的指针，要求对指针的限定
        const afloat * restrict col = src1;  // 定义指向向量的指针，要求对指针的限定

        float sum = 0.0f;  // 初始化 sum 为 0.0

        for (unsigned j = 0; j < ncols; j++) {  // 循环遍历矩阵的列
            sum += *row++ * *col++;  // 计算矩阵和向量的乘积并累加到 sum
        }

        dst[i] = sum;  // 将结果存储到 dst 中
    }
}

void mul_mat_vec_f32_2(  // 定义函数 mul_mat_vec_f32_2，实现矩阵和向量的乘法
    const void * src0,  // 参数 src0，指向 void 类型的指针，用于存储矩阵数据
    const void * src1,  // 参数 src1，指向 void 类型的指针，用于存储向量数据
    void * dst,  // 参数 dst，指向 void 类型的指针，用于存储结果向量数据
    unsigned nrows,  // 参数 nrows，无符号整型，表示矩阵的行数
    unsigned ncols) {  // 参数 ncols，无符号整型，表示矩阵的列数
    void * d = dst;  // 定义指向结果向量的指针
    # 遍历行
    for (unsigned i = 0; i < nrows; i++) {
        # 初始化行和列的和
        float sum = 0.0f;

        # 获取当前行的指针
        const char * row = (const char*)src0 + i*ncols*sizeof(float);
        # 获取列的指针
        const char * col = (const char*)src1;
        # 遍历列
        for (unsigned j = 0; j < ncols; j++) {
            # 计算当前行和列的乘积并累加到和中
            sum += (*(float *)row) * (*(float *)col);
            # 移动到下一个元素
            row += sizeof(float);
            col += sizeof(float);
        }
        # 将和存储到目标数组中
        *(float *)d = sum;
        # 移动到下一个位置
        d = (char*)d + sizeof(float);
    }
// 如果是在 Windows 平台下编译，定义了一些特定的宏，就使用 _aligned_malloc 函数来分配内存
#if defined(_MSC_VER) || defined(__MINGW32__) || defined(__MINGW64__)
void* aligned_alloc(size_t alignment, size_t size) {
    return _aligned_malloc(size, alignment);
}
#endif

// 主函数
int main(int argc, const char ** argv) {
    // 用 malloc 函数分配内存，但是没有对齐
    //float * src0 = malloc(sizeof(float)*N*M);
    //float * src1 = malloc(sizeof(float)*M);
    //float * dst  = malloc(sizeof(float)*N);

    // 使用 aligned_alloc 函数分配内存，并且对齐到 32 字节
    afloat * src0 = (float *)(aligned_alloc(32, sizeof(float)*N*M));
    afloat * src1 = (float *)(aligned_alloc(32, sizeof(float)*M));
    afloat * dst  = (float *)(aligned_alloc(32, sizeof(float)*N));

    // 初始化 src0 数组
    for (int i = 0; i < N*M; i++) {
        src0[i] = (afloat)i;
    }

    // 初始化 src1 数组
    for (int i = 0; i < M; i++) {
        src1[i] = (afloat)i;
    }

    // 迭代次数
    const int nIter = 10;

    // 记录开始时间
    const clock_t start = clock();

    // 计算结果的总和
    double sum = 0.0f;
    for (int i = 0; i < nIter; i++) {
        // 调用函数进行矩阵乘向量运算
        //mul_mat_vec_f32_0(src0, src1, dst, N, M);
        mul_mat_vec_f32_1(src0, src1, dst, N, M);
        //mul_mat_vec_f32_2(src0, src1, dst, N, M);
        // 计算结果数组的总和
        for (int  i = 0; i < N; i++) {
            sum += dst[i];
        }
    }

    {
        // 记录结束时间
        const clock_t end = clock();
        // 输出函数执行时间
        printf("%s: elapsed ticks: %ld\n", __func__, end - start);
    }

    // 输出结果的总和
    printf("%f\n", sum);

    return 0;
}
```