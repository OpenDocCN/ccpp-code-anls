# `nmap\liblinear\blas\blas.h`

```cpp
/* blas.h  --  C header file for BLAS                         Ver 1.0 */
/* Jesse Bennett                                       March 23, 2000 */

/**  barf  [ba:rf]  2.  "He suggested using FORTRAN, and everybody barfed."

    - From The Shogakukan DICTIONARY OF NEW ENGLISH (Second edition) */

#ifndef BLAS_INCLUDE
#define BLAS_INCLUDE

/* Data types specific to BLAS implementation */
typedef struct { float r, i; } fcomplex;  // 定义复数结构体，包含实部和虚部
typedef struct { double r, i; } dcomplex;  // 定义双精度复数结构体，包含实部和虚部
typedef int blasbool;  // 定义 BLAS 布尔类型

#include "blasp.h"    /* Prototypes for all BLAS functions */  // 包含 BLAS 函数的原型声明

#define FALSE 0  // 定义 FALSE 常量为 0
#define TRUE  1  // 定义 TRUE 常量为 1

/* Macro functions */
#define MIN(a,b) ((a) <= (b) ? (a) : (b))  // 定义取最小值的宏函数
#define MAX(a,b) ((a) >= (b) ? (a) : (b))  // 定义取最大值的宏函数

#endif
```