# `nmap\liblinear\blas\daxpy.c`

```
#include "blas.h"

int daxpy_(int *n, double *sa, double *sx, int *incx, double *sy,
           int *incy)
{
  long int i, m, ix, iy, nn, iincx, iincy;
  register double ssa;

  /* constant times a vector plus a vector.   
     uses unrolled loop for increments equal to one.   
     jack dongarra, linpack, 3/11/78.   
     modified 12/3/93, array(1) declarations changed to array(*) */

  /* Dereference inputs */
  nn = *n;  // 将指针 n 指向的值赋给 nn
  ssa = *sa;  // 将指针 sa 指向的值赋给 ssa
  iincx = *incx;  // 将指针 incx 指向的值赋给 iincx
  iincy = *incy;  // 将指针 incy 指向的值赋给 iincy

  if( nn > 0 && ssa != 0.0 )  // 如果 nn 大于 0 并且 ssa 不等于 0
  {
    if (iincx == 1 && iincy == 1) /* code for both increments equal to 1 */  // 如果增量都等于 1
    {
      m = nn-3;  // 计算循环次数
      for (i = 0; i < m; i += 4)  // 循环，每次增加 4
      {
        sy[i] += ssa * sx[i];  // 计算结果并赋值
        sy[i+1] += ssa * sx[i+1];  // 计算结果并赋值
        sy[i+2] += ssa * sx[i+2];  // 计算结果并赋值
        sy[i+3] += ssa * sx[i+3];  // 计算结果并赋值
      }
      for ( ; i < nn; ++i) /* clean-up loop */  // 清理剩余的循环
        sy[i] += ssa * sx[i];  // 计算结果并赋值
    }
    else /* code for unequal increments or equal increments not equal to 1 */  // 如果增量不相等或者增量不等于 1
    {
      ix = iincx >= 0 ? 0 : (1 - nn) * iincx;  // 计算起始位置
      iy = iincy >= 0 ? 0 : (1 - nn) * iincy;  // 计算起始位置
      for (i = 0; i < nn; i++)  // 循环
      {
        sy[iy] += ssa * sx[ix];  // 计算结果并赋值
        ix += iincx;  // 更新索引
        iy += iincy;  // 更新索引
      }
    }
  }

  return 0;  // 返回结果
} /* daxpy_ */
```