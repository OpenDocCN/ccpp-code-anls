# `ggml\tests\test-svd0.c`

```
// SVD dimensionality reduction

#include <float.h>  // 包含浮点数常量的定义
#include <stdint.h>  // 包含整数类型的定义
#include <stdio.h>   // 包含标准输入输出函数的定义
#include <assert.h>  // 包含断言宏的定义
#include <stdlib.h>  // 包含动态内存分配函数的定义
#include <string.h>  // 包含字符串处理函数的定义
#include <time.h>    // 包含时间函数的定义
#include <math.h>    // 包含数学函数的定义

#include <sys/time.h>  // 包含系统时间函数的定义

#ifdef GGML_USE_ACCELERATE
#include <Accelerate/Accelerate.h>  // 使用加速框架的定义
#endif

float frand(void) {
    return (float) rand() / (float) RAND_MAX;  // 生成一个随机浮点数
}

//int sgesvd_(char *__jobu, char *__jobvt, __CLPK_integer *__m,
//        __CLPK_integer *__n, __CLPK_real *__a, __CLPK_integer *__lda,
//        __CLPK_real *__s, __CLPK_real *__u, __CLPK_integer *__ldu,
//        __CLPK_real *__vt, __CLPK_integer *__ldvt, __CLPK_real *__work,
//        __CLPK_integer *__lwork,
//        __CLPK_integer *__info)

int main(int argc, const char ** argv) {
    int m = 10;  // 定义矩阵的行数
    int n = 5;   // 定义矩阵的列数

    float * A  = malloc(n * m * sizeof(float));  // 分配内存给矩阵 A
    float * A0 = malloc(n * m * sizeof(float));  // 分配内存给矩阵 A0

    for (int i = 0; i < n; ++i) {  // 遍历矩阵的行
        for (int j = 0; j < m; ++j) {  // 遍历矩阵的列
            A[i * m + j] = (float) (10.0f*(i + 1) + 1.0f * frand());  // 为矩阵赋随机值
            //A[i * m + j] = (float) (10.0f*(i%2 + 1) + 0.1f * frand());
            //if (i == 2) {
            //    A[i * m + j] += 20*frand();
            //}
            if ((i == 1 || i == 3) && j > m/2) {  // 如果满足条件
                A[i * m + j] = -A[i * m + j];  // 将矩阵中的值取反
            }
        }
    }

    // average vector
    //float * M = malloc(m * sizeof(float));

    //{
    //    for (int j = 0; j < m; ++j) {
    //        M[j] = 0.0f;
    //    }
    //    for (int i = 0; i < n; ++i) {
    //        for (int j = 0; j < m; ++j) {
    //            M[j] += A[i * m + j];
    //        }
    //    }
    //    for (int j = 0; j < m; ++j) {
    //        M[j] /= (float) n;
    //    }
    //}

    //// subtract average vector
    //for (int i = 0; i < n; ++i) {
    //    for (int j = 0; j < m; ++j) {
    //        A[i * m + j] -= M[j];
    //    }
    //}

    memcpy(A0, A, n * m * sizeof(float));  // 复制矩阵 A 到 A0

    // print A
    printf("A:\n");  // 打印提示信息
    // 遍历矩阵 A 的每一列，打印每个元素的值
    for (int i = 0; i < n; ++i) {
        printf("col %d : ", i);
        for (int j = 0; j < m; ++j) {
            printf("%9.5f ", A[i * m + j]);
        }
        printf("\n");
    }
    printf("\n");

    // 分配内存给 U、S、V 三个矩阵
    float * U = malloc(n * m * sizeof(float));
    float * S = malloc(n * sizeof(float));
    float * V = malloc(n * n * sizeof(float));

    // 设置矩阵 A、U、V 的维度
    int lda = m;
    int ldu = m;
    int ldvt = n;

    // 计算 SVD 分解所需的工作空间大小
    float work_size;
    int lwork = -1;
    int info = 0;

    sgesvd_("S", "S", &m, &n, A, &lda, S, U, &ldu, V, &ldvt, &work_size, &lwork, &info);

    // 将工作空间大小转换为整数
    lwork = (int) work_size;

    // 分配内存给工作空间
    float * work = malloc(lwork * sizeof(float));

    // 重新进行 SVD 分解，此时传入工作空间
    sgesvd_("S", "S", &m, &n, A, &lda, S, U, &ldu, V, &ldvt, work, &lwork, &info);

    // 打印矩阵 U
    printf("U:\n");
    for (int i = 0; i < n; ++i) {
        printf("col %d : ", i);
        for (int j = 0; j < m; ++j) {
            printf("%9.5f ", U[i * m + j]);
        }
        printf("\n");
    }
    printf("\n");

    // 对 S 进行归一化处理
    {
        double sum = 0.0;
        for (int i = 0; i < n; ++i) {
            sum += S[i];
        }
        sum *= sqrt((double) m);
        for (int i = 0; i < n; ++i) {
            S[i] /= sum;
        }
    }

    // 打印归一化后的 S
    printf("S:\n");
    for (int i = 0; i < n; ++i) {
        printf("- %d = %9.5f\n", i, S[i]);
    }
    printf("\n");

    // 打印矩阵 V
    printf("V:\n");
    for (int i = 0; i < n; ++i) {
        printf("col %d : ", i);
        for (int j = 0; j < n; ++j) {
            printf("%9.5f ", V[i * n + j]);
        }
        printf("\n");
    }
    printf("\n");

    // 打印矩阵 A
    printf("A:\n");
    for (int i = 0; i < n; ++i) {
        printf("col %d : ", i);
        for (int j = 0; j < m; ++j) {
            printf("%9.5f ", A[i * m + j]);
        }
        printf("\n");
    }
    printf("\n");

    // 计算 U 中的奇异向量
    // 对 U 进行归一化处理
    for (int i = 0; i < n; ++i) {
        double sum = 0.0;
        for (int j = 0; j < m; ++j) {
            sum += U[i * m + j] * U[i * m + j];
        }
        sum = sqrt(sum);
        for (int j = 0; j < m; ++j) {
            U[i * m + j] /= sum*sqrt((double) m);
        }
    }

    // 打印 U
    printf("U:\n");
    for (int i = 0; i < n; ++i) {
        printf("col %d : ", i);
        for (int j = 0; j < m; ++j) {
            printf("%9.5f ", U[i * m + j]);
        }
        printf("\n");
    }
    printf("\n");

    // 将 A0 投影到 U 上
    float * A1 = malloc(n * n * sizeof(float));

    for (int i = 0; i < n; ++i) {
        for (int j = 0; j < n; ++j) {
            A1[i * n + j] = 0.0f;
            for (int k = 0; k < m; ++k) {
                A1[i * n + j] += A0[i * m + k] * U[j * m + k];
            }
        }
    }

    // 打印 A1
    printf("A1:\n");
    for (int i = 0; i < n; ++i) {
        printf("col %d : ", i);
        for (int j = 0; j < n; ++j) {
            printf("%9.5f ", A1[i * n + j]);
        }
        printf("\n");
    }
    printf("\n");

    // 返回 0
    return 0;
# 闭合前面的函数定义
```