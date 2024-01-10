# `nmap\liblinear\blas\dscal.c`

```
#include "blas.h"

int dscal_(int *n, double *sa, double *sx, int *incx)
{
  long int i, m, nincx, nn, iincx;
  double ssa;

  /* scales a vector by a constant.   
     uses unrolled loops for increment equal to 1.   
     jack dongarra, linpack, 3/11/78.   
     modified 3/93 to return if incx .le. 0.   
     modified 12/3/93, array(1) declarations changed to array(*) */

  /* Dereference inputs */
  nn = *n;  // 将指针 n 指向的值赋给 nn
  iincx = *incx;  // 将指针 incx 指向的值赋给 iincx
  ssa = *sa;  // 将指针 sa 指向的值赋给 ssa

  if (nn > 0 && iincx > 0)  // 如果 nn 和 iincx 都大于 0
  {
    if (iincx == 1) /* code for increment equal to 1 */  // 如果增量等于 1
    {
      m = nn-4;  // 计算循环次数
      for (i = 0; i < m; i += 5)  // 循环，每次增加 5
      {
        sx[i] = ssa * sx[i];  // 对数组进行乘法操作
        sx[i+1] = ssa * sx[i+1];  // 对数组进行乘法操作
        sx[i+2] = ssa * sx[i+2];  // 对数组进行乘法操作
        sx[i+3] = ssa * sx[i+3];  // 对数组进行乘法操作
        sx[i+4] = ssa * sx[i+4];  // 对数组进行乘法操作
      }
      for ( ; i < nn; ++i) /* clean-up loop */  // 清理剩余的循环
        sx[i] = ssa * sx[i];  // 对数组进行乘法操作
    }
    else /* code for increment not equal to 1 */  // 如果增量不等于 1
    {
      nincx = nn * iincx;  // 计算循环次数
      for (i = 0; i < nincx; i += iincx)  // 循环，每次增加 iincx
        sx[i] = ssa * sx[i];  // 对数组进行乘法操作
    }
  }

  return 0;  // 返回 0
} /* dscal_ */
```