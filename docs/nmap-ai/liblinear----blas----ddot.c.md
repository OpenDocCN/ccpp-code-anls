# `nmap\liblinear\blas\ddot.c`

```
#include "blas.h"

double ddot_(int *n, double *sx, int *incx, double *sy, int *incy)
{
  long int i, m, nn, iincx, iincy;
  double stemp;
  long int ix, iy;

  /* forms the dot product of two vectors.   
     uses unrolled loops for increments equal to one.   
     jack dongarra, linpack, 3/11/78.   
     modified 12/3/93, array(1) declarations changed to array(*) */

  /* Dereference inputs */
  nn = *n;  // 获取向量的长度
  iincx = *incx;  // 获取 x 向量的增量
  iincy = *incy;  // 获取 y 向量的增量

  stemp = 0.0;  // 初始化点积结果为 0
  if (nn > 0)  // 如果向量长度大于 0
  {
    if (iincx == 1 && iincy == 1) /* code for both increments equal to 1 */
    {
      m = nn-4;  // 计算循环展开的次数
      for (i = 0; i < m; i += 5)  // 循环展开，每次计算 5 个元素
        stemp += sx[i] * sy[i] + sx[i+1] * sy[i+1] + sx[i+2] * sy[i+2] +
                 sx[i+3] * sy[i+3] + sx[i+4] * sy[i+4];

      for ( ; i < nn; i++)        /* clean-up loop */
        stemp += sx[i] * sy[i];  // 处理剩余的元素
    }
    else /* code for unequal increments or equal increments not equal to 1 */
    {
      ix = 0;  // 初始化 x 向量的索引
      iy = 0;  // 初始化 y 向量的索引
      if (iincx < 0)
        ix = (1 - nn) * iincx;  // 计算负增量时的起始索引
      if (iincy < 0)
        iy = (1 - nn) * iincy;  // 计算负增量时的起始索引
      for (i = 0; i < nn; i++)  // 遍历向量
      {
        stemp += sx[ix] * sy[iy];  // 计算点积
        ix += iincx;  // 更新 x 向量的索引
        iy += iincy;  // 更新 y 向量的索引
      }
    }
  }

  return stemp;  // 返回点积结果
} /* ddot_ */
```