# `nmap\liblinear\blas\blasp.h`

```cpp
#ifdef F2C_COMPAT
// 如果定义了 F2C_COMPAT，则使用以下函数原型
void cdotc_(fcomplex *dotval, int *n, fcomplex *cx, int *incx,
            fcomplex *cy, int *incy);
void cdotu_(fcomplex *dotval, int *n, fcomplex *cx, int *incx,
            fcomplex *cy, int *incy);
double sasum_(int *n, float *sx, int *incx);
double scasum_(int *n, fcomplex *cx, int *incx);
double scnrm2_(int *n, fcomplex *x, int *incx);
double sdot_(int *n, float *sx, int *incx, float *sy, int *incy);
double snrm2_(int *n, float *x, int *incx);
void zdotc_(dcomplex *dotval, int *n, dcomplex *cx, int *incx,
            dcomplex *cy, int *incy);
void zdotu_(dcomplex *dotval, int *n, dcomplex *cx, int *incx,
            dcomplex *cy, int *incy);
#else
// 如果未定义 F2C_COMPAT，则使用以下函数原型
fcomplex cdotc_(int *n, fcomplex *cx, int *incx, fcomplex *cy, int *incy);
fcomplex cdotu_(int *n, fcomplex *cx, int *incx, fcomplex *cy, int *incy);
float sasum_(int *n, float *sx, int *incx);
float scasum_(int *n, fcomplex *cx, int *incx);
float scnrm2_(int *n, fcomplex *x, int *incx);
float sdot_(int *n, float *sx, int *incx, float *sy, int *incy);
float snrm2_(int *n, float *x, int *incx);
dcomplex zdotc_(int *n, dcomplex *cx, int *incx, dcomplex *cy, int *incy);
dcomplex zdotu_(int *n, dcomplex *cx, int *incx, dcomplex *cy, int *incy);
#endif
// 剩余的函数按字母顺序列出
int caxpy_(int *n, fcomplex *ca, fcomplex *cx, int *incx, fcomplex *cy,
           int *incy);
int ccopy_(int *n, fcomplex *cx, int *incx, fcomplex *cy, int *incy);
int cgbmv_(char *trans, int *m, int *n, int *kl, int *ku,
           fcomplex *alpha, fcomplex *a, int *lda, fcomplex *x, int *incx,
           fcomplex *beta, fcomplex *y, int *incy);
# 执行复杂矩阵乘法
int cgemm_(char *transa, char *transb, int *m, int *n, int *k,
           fcomplex *alpha, fcomplex *a, int *lda, fcomplex *b, int *ldb,
           fcomplex *beta, fcomplex *c, int *ldc);

# 执行复杂矩阵向量乘法
int cgemv_(char *trans, int *m, int *n, fcomplex *alpha, fcomplex *a,
           int *lda, fcomplex *x, int *incx, fcomplex *beta, fcomplex *y,
           int *incy);

# 执行复杂矩阵向量外积
int cgerc_(int *m, int *n, fcomplex *alpha, fcomplex *x, int *incx,
           fcomplex *y, int *incy, fcomplex *a, int *lda);

# 执行复杂矩阵向量外积
int cgeru_(int *m, int *n, fcomplex *alpha, fcomplex *x, int *incx,
           fcomplex *y, int *incy, fcomplex *a, int *lda);

# 执行复杂埃尔米特矩阵向量乘法
int chbmv_(char *uplo, int *n, int *k, fcomplex *alpha, fcomplex *a,
           int *lda, fcomplex *x, int *incx, fcomplex *beta, fcomplex *y,
           int *incy);

# 执行复杂埃尔米特矩阵乘法
int chemm_(char *side, char *uplo, int *m, int *n, fcomplex *alpha,
           fcomplex *a, int *lda, fcomplex *b, int *ldb, fcomplex *beta,
           fcomplex *c, int *ldc);

# 执行复杂埃尔米特矩阵向量乘法
int chemv_(char *uplo, int *n, fcomplex *alpha, fcomplex *a, int *lda,
           fcomplex *x, int *incx, fcomplex *beta, fcomplex *y, int *incy);

# 执行复杂埃尔米特矩阵向量外积
int cher_(char *uplo, int *n, float *alpha, fcomplex *x, int *incx,
          fcomplex *a, int *lda);

# 执行复杂埃尔米特矩阵向量外积
int cher2_(char *uplo, int *n, fcomplex *alpha, fcomplex *x, int *incx,
           fcomplex *y, int *incy, fcomplex *a, int *lda);

# 执行复杂埃尔米特矩阵乘法或转置矩阵乘法
int cher2k_(char *uplo, char *trans, int *n, int *k, fcomplex *alpha,
            fcomplex *a, int *lda, fcomplex *b, int *ldb, float *beta,
            fcomplex *c, int *ldc);

# 执行复杂埃尔米特矩阵乘法或转置矩阵乘法
int cherk_(char *uplo, char *trans, int *n, int *k, float *alpha,
           fcomplex *a, int *lda, float *beta, fcomplex *c, int *ldc);

# 执行复杂埃尔米特矩阵向量乘法
int chpmv_(char *uplo, int *n, fcomplex *alpha, fcomplex *ap, fcomplex *x,
           int *incx, fcomplex *beta, fcomplex *y, int *incy);

# 执行复杂埃尔米特矩阵向量外积
int chpr_(char *uplo, int *n, float *alpha, fcomplex *x, int *incx,
          fcomplex *ap);
# 计算复数矩阵和向量的乘积，并将结果加到另一个向量上
int chpr2_(char *uplo, int *n, fcomplex *alpha, fcomplex *x, int *incx,
           fcomplex *y, int *incy, fcomplex *ap);

# 计算 Givens 变换
int crotg_(fcomplex *ca, fcomplex *cb, float *c, fcomplex *s);

# 将复数向量乘以一个复数，并将结果存储在另一个向量中
int cscal_(int *n, fcomplex *ca, fcomplex *cx, int *incx);

# 将复数向量乘以一个实数，并将结果存储在另一个向量中
int csscal_(int *n, float *sa, fcomplex *cx, int *incx);

# 交换两个复数向量的元素
int cswap_(int *n, fcomplex *cx, int *incx, fcomplex *cy, int *incy);

# 计算矩阵乘积和另一个矩阵的乘积之和
int csymm_(char *side, char *uplo, int *m, int *n, fcomplex *alpha,
           fcomplex *a, int *lda, fcomplex *b, int *ldb, fcomplex *beta,
           fcomplex *c, int *ldc);

# 计算矩阵乘积和另一个矩阵的转置乘积之和
int csyr2k_(char *uplo, char *trans, int *n, int *k, fcomplex *alpha,
            fcomplex *a, int *lda, fcomplex *b, int *ldb, fcomplex *beta,
            fcomplex *c, int *ldc);

# 计算矩阵乘积和另一个矩阵的转置乘积之和
int csyrk_(char *uplo, char *trans, int *n, int *k, fcomplex *alpha,
           fcomplex *a, int *lda, fcomplex *beta, fcomplex *c, int *ldc);

# 使用三角矩阵和向量进行矩阵-向量乘法
int ctbmv_(char *uplo, char *trans, char *diag, int *n, int *k,
           fcomplex *a, int *lda, fcomplex *x, int *incx);

# 使用三角矩阵和向量解决线性方程组
int ctbsv_(char *uplo, char *trans, char *diag, int *n, int *k,
           fcomplex *a, int *lda, fcomplex *x, int *incx);

# 使用三角矩阵和向量进行矩阵-向量乘法
int ctpmv_(char *uplo, char *trans, char *diag, int *n, fcomplex *ap,
           fcomplex *x, int *incx);

# 使用三角矩阵和向量解决线性方程组
int ctpsv_(char *uplo, char *trans, char *diag, int *n, fcomplex *ap,
           fcomplex *x, int *incx);

# 使用三角矩阵和矩阵进行矩阵乘法
int ctrmm_(char *side, char *uplo, char *transa, char *diag, int *m,
           int *n, fcomplex *alpha, fcomplex *a, int *lda, fcomplex *b,
           int *ldb);

# 使用三角矩阵和向量解决线性方程组
int ctrmv_(char *uplo, char *trans, char *diag, int *n, fcomplex *a,
           int *lda, fcomplex *x, int *incx);

# 使用三角矩阵和矩阵解决线性方程组
int ctrsm_(char *side, char *uplo, char *transa, char *diag, int *m,
           int *n, fcomplex *alpha, fcomplex *a, int *lda, fcomplex *b,
           int *ldb);

# 使用三角矩阵和向量解决线性方程组
int ctrsv_(char *uplo, char *trans, char *diag, int *n, fcomplex *a,
           int *lda, fcomplex *x, int *incx);

# 计算两个向量的线性组合
int daxpy_(int *n, double *sa, double *sx, int *incx, double *sy,
           int *incy);
# 复制数组
int dcopy_(int *n, double *sx, int *incx, double *sy, int *incy);

# 带广义矩阵向量乘法
int dgbmv_(char *trans, int *m, int *n, int *kl, int *ku,
           double *alpha, double *a, int *lda, double *x, int *incx,
           double *beta, double *y, int *incy);

# 带一般矩阵乘法
int dgemm_(char *transa, char *transb, int *m, int *n, int *k,
           double *alpha, double *a, int *lda, double *b, int *ldb,
           double *beta, double *c, int *ldc);

# 带一般矩阵向量乘法
int dgemv_(char *trans, int *m, int *n, double *alpha, double *a,
           int *lda, double *x, int *incx, double *beta, double *y, 
           int *incy);

# 一般矩阵的外积
int dger_(int *m, int *n, double *alpha, double *x, int *incx,
          double *y, int *incy, double *a, int *lda);

# 旋转向量
int drot_(int *n, double *sx, int *incx, double *sy, int *incy,
          double *c, double *s);

# 生成 Givens 旋转因子
int drotg_(double *sa, double *sb, double *c, double *s);

# 带对称矩阵向量乘法
int dsbmv_(char *uplo, int *n, int *k, double *alpha, double *a,
           int *lda, double *x, int *incx, double *beta, double *y, 
           int *incy);

# 向量的缩放
int dscal_(int *n, double *sa, double *sx, int *incx);

# 带对称矩阵向量乘法
int dspmv_(char *uplo, int *n, double *alpha, double *ap, double *x,
           int *incx, double *beta, double *y, int *incy);

# 带对称矩阵的外积
int dspr_(char *uplo, int *n, double *alpha, double *x, int *incx,
          double *ap);

# 带对称矩阵的外积
int dspr2_(char *uplo, int *n, double *alpha, double *x, int *incx,
           double *y, int *incy, double *ap);

# 交换向量
int dswap_(int *n, double *sx, int *incx, double *sy, int *incy);

# 带对称矩阵矩阵乘法
int dsymm_(char *side, char *uplo, int *m, int *n, double *alpha,
           double *a, int *lda, double *b, int *ldb, double *beta,
           double *c, int *ldc);

# 带对称矩阵向量乘法
int dsymv_(char *uplo, int *n, double *alpha, double *a, int *lda,
           double *x, int *incx, double *beta, double *y, int *incy);

# 带对称矩阵的外积
int dsyr_(char *uplo, int *n, double *alpha, double *x, int *incx,
          double *a, int *lda);
# 计算双精度实数向量的外积，并将结果累加到双精度实数矩阵中
int dsyr2_(char *uplo, int *n, double *alpha, double *x, int *incx,
           double *y, int *incy, double *a, int *lda);

# 计算双精度实数矩阵的乘积，并将结果累加到另一个双精度实数矩阵中
int dsyr2k_(char *uplo, char *trans, int *n, int *k, double *alpha,
            double *a, int *lda, double *b, int *ldb, double *beta,
            double *c, int *ldc);

# 计算双精度实数矩阵的乘积，并将结果累加到另一个双精度实数矩阵中
int dsyrk_(char *uplo, char *trans, int *n, int *k, double *alpha,
           double *a, int *lda, double *beta, double *c, int *ldc);

# 计算双精度实数矩阵与双精度实数向量的乘积，并将结果存储在另一个双精度实数向量中
int dtbmv_(char *uplo, char *trans, char *diag, int *n, int *k,
           double *a, int *lda, double *x, int *incx);

# 解双精度实数三角矩阵与双精度实数向量的线性方程组
int dtbsv_(char *uplo, char *trans, char *diag, int *n, int *k,
           double *a, int *lda, double *x, int *incx);

# 计算三角双精度实数矩阵与双精度实数向量的乘积，并将结果存储在另一个双精度实数向量中
int dtpmv_(char *uplo, char *trans, char *diag, int *n, double *ap,
           double *x, int *incx);

# 解三角双精度实数矩阵与双精度实数向量的线性方程组
int dtpsv_(char *uplo, char *trans, char *diag, int *n, double *ap,
           double *x, int *incx);

# 计算三角双精度实数矩阵与双精度实数矩阵的乘积
int dtrmm_(char *side, char *uplo, char *transa, char *diag, int *m,
           int *n, double *alpha, double *a, int *lda, double *b, 
           int *ldb);

# 计算三角双精度实数矩阵与双精度实数向量的乘积，并将结果存储在另一个双精度实数向量中
int dtrmv_(char *uplo, char *trans, char *diag, int *n, double *a,
           int *lda, double *x, int *incx);

# 解三角双精度实数矩阵与双精度实数矩阵的线性方程组
int dtrsm_(char *side, char *uplo, char *transa, char *diag, int *m,
           int *n, double *alpha, double *a, int *lda, double *b, 
           int *ldb);

# 解三角双精度实数矩阵与双精度实数向量的线性方程组
int dtrsv_(char *uplo, char *trans, char *diag, int *n, double *a,
           int *lda, double *x, int *incx);

# 计算单精度实数向量的线性组合
int saxpy_(int *n, float *sa, float *sx, int *incx, float *sy, int *incy);

# 复制单精度实数向量
int scopy_(int *n, float *sx, int *incx, float *sy, int *incy);

# 计算一般带状单精度实数矩阵与单精度实数向量的乘积，并将结果累加到另一个单精度实数向量中
int sgbmv_(char *trans, int *m, int *n, int *kl, int *ku,
           float *alpha, float *a, int *lda, float *x, int *incx,
           float *beta, float *y, int *incy);

# 计算单精度实数矩阵的乘积
int sgemm_(char *transa, char *transb, int *m, int *n, int *k,
           float *alpha, float *a, int *lda, float *b, int *ldb,
           float *beta, float *c, int *ldc);
# 执行单精度矩阵-向量乘法
int sgemv_(char *trans, int *m, int *n, float *alpha, float *a,
           int *lda, float *x, int *incx, float *beta, float *y, 
           int *incy);

# 执行单精度矩阵-向量外积
int sger_(int *m, int *n, float *alpha, float *x, int *incx,
          float *y, int *incy, float *a, int *lda);

# 执行单精度向量旋转
int srot_(int *n, float *sx, int *incx, float *sy, int *incy,
          float *c, float *s);

# 执行单精度向量旋转
int srotg_(float *sa, float *sb, float *c, float *s);

# 执行单精度带对称矩阵向量乘法
int ssbmv_(char *uplo, int *n, int *k, float *alpha, float *a,
           int *lda, float *x, int *incx, float *beta, float *y, 
           int *incy);

# 执行单精度向量缩放
int sscal_(int *n, float *sa, float *sx, int *incx);

# 执行单精度带对称矩阵向量乘法
int sspmv_(char *uplo, int *n, float *alpha, float *ap, float *x,
           int *incx, float *beta, float *y, int *incy);

# 执行单精度对称矩阵向量外积
int sspr_(char *uplo, int *n, float *alpha, float *x, int *incx,
          float *ap);

# 执行单精度对称矩阵向量外积
int sspr2_(char *uplo, int *n, float *alpha, float *x, int *incx,
           float *y, int *incy, float *ap);

# 执行单精度向量交换
int sswap_(int *n, float *sx, int *incx, float *sy, int *incy);

# 执行单精度矩阵乘法
int ssymm_(char *side, char *uplo, int *m, int *n, float *alpha,
           float *a, int *lda, float *b, int *ldb, float *beta,
           float *c, int *ldc);

# 执行单精度对称矩阵向量乘法
int ssymv_(char *uplo, int *n, float *alpha, float *a, int *lda,
           float *x, int *incx, float *beta, float *y, int *incy);

# 执行单精度对称矩阵向量外积
int ssyr_(char *uplo, int *n, float *alpha, float *x, int *incx,
          float *a, int *lda);

# 执行单精度对称矩阵向量外积
int ssyr2_(char *uplo, int *n, float *alpha, float *x, int *incx,
           float *y, int *incy, float *a, int *lda);

# 执行单精度对称矩阵乘法
int ssyr2k_(char *uplo, char *trans, int *n, int *k, float *alpha,
            float *a, int *lda, float *b, int *ldb, float *beta,
            float *c, int *ldc);

# 执行单精度对称矩阵乘法
int ssyrk_(char *uplo, char *trans, int *n, int *k, float *alpha,
           float *a, int *lda, float *beta, float *c, int *ldc);

# 执行单精度带三角矩阵向量乘法
int stbmv_(char *uplo, char *trans, char *diag, int *n, int *k,
           float *a, int *lda, float *x, int *incx);
# 解决稠密矩阵方程 Ax = b，A 为上（'U'）或下（'L'）三角矩阵
int stbsv_(char *uplo, char *trans, char *diag, int *n, int *k,
           float *a, int *lda, float *x, int *incx);

# 矩阵-向量乘积，A 为上（'U'）或下（'L'）三角矩阵
int stpmv_(char *uplo, char *trans, char *diag, int *n, float *ap,
           float *x, int *incx);

# 解决稠密矩阵方程 Ax = b，A 为上（'U'）或下（'L'）三角矩阵
int stpsv_(char *uplo, char *trans, char *diag, int *n, float *ap,
           float *x, int *incx);

# 矩阵乘法，A 为上（'U'）或下（'L'）三角矩阵
int strmm_(char *side, char *uplo, char *transa, char *diag, int *m,
           int *n, float *alpha, float *a, int *lda, float *b, 
           int *ldb);

# 矩阵-向量乘积，A 为上（'U'）或下（'L'）三角矩阵
int strmv_(char *uplo, char *trans, char *diag, int *n, float *a,
           int *lda, float *x, int *incx);

# 解决稠密矩阵方程 Ax = b，A 为上（'U'）或下（'L'）三角矩阵
int strsm_(char *side, char *uplo, char *transa, char *diag, int *m,
           int *n, float *alpha, float *a, int *lda, float *b, 
           int *ldb);

# 解决稠密矩阵方程 Ax = b，A 为上（'U'）或下（'L'）三角矩阵
int strsv_(char *uplo, char *trans, char *diag, int *n, float *a,
           int *lda, float *x, int *incx);

# 向量加法，y = alpha * x + y
int zaxpy_(int *n, dcomplex *ca, dcomplex *cx, int *incx, dcomplex *cy,
           int *incy);

# 向量复制，y = x
int zcopy_(int *n, dcomplex *cx, int *incx, dcomplex *cy, int *incy);

# 向量缩放，x = sa * x
int zdscal_(int *n, double *sa, dcomplex *cx, int *incx);

# 稠密矩阵-向量乘积，y = alpha * A * x + beta * y
int zgbmv_(char *trans, int *m, int *n, int *kl, int *ku,
           dcomplex *alpha, dcomplex *a, int *lda, dcomplex *x, int *incx,
           dcomplex *beta, dcomplex *y, int *incy);

# 矩阵乘法，C = alpha * A * B + beta * C
int zgemm_(char *transa, char *transb, int *m, int *n, int *k,
           dcomplex *alpha, dcomplex *a, int *lda, dcomplex *b, int *ldb,
           dcomplex *beta, dcomplex *c, int *ldc);

# 矩阵-向量乘积，y = alpha * A * x + beta * y
int zgemv_(char *trans, int *m, int *n, dcomplex *alpha, dcomplex *a,
           int *lda, dcomplex *x, int *incx, dcomplex *beta, dcomplex *y,
           int *incy);

# 稠密矩阵-向量乘积，A = alpha * x * y^T + A
int zgerc_(int *m, int *n, dcomplex *alpha, dcomplex *x, int *incx,
           dcomplex *y, int *incy, dcomplex *a, int *lda);

# 稠密矩阵-向量乘积，A = alpha * x * y^T + A
int zgeru_(int *m, int *n, dcomplex *alpha, dcomplex *x, int *incx,
           dcomplex *y, int *incy, dcomplex *a, int *lda);
# 计算 Hermitian 矩阵向量乘积
int zhbmv_(char *uplo, int *n, int *k, dcomplex *alpha, dcomplex *a,
           int *lda, dcomplex *x, int *incx, dcomplex *beta, dcomplex *y,
           int *incy);

# 计算 Hermitian 矩阵乘法
int zhemm_(char *side, char *uplo, int *m, int *n, dcomplex *alpha,
           dcomplex *a, int *lda, dcomplex *b, int *ldb, dcomplex *beta,
           dcomplex *c, int *ldc);

# 计算 Hermitian 矩阵向量乘积
int zhemv_(char *uplo, int *n, dcomplex *alpha, dcomplex *a, int *lda,
           dcomplex *x, int *incx, dcomplex *beta, dcomplex *y, int *incy);

# 计算 Hermitian 矩阵向量乘积
int zher_(char *uplo, int *n, double *alpha, dcomplex *x, int *incx,
          dcomplex *a, int *lda);

# 计算 Hermitian 矩阵向量乘积并加到另一个向量上
int zher2_(char *uplo, int *n, dcomplex *alpha, dcomplex *x, int *incx,
           dcomplex *y, int *incy, dcomplex *a, int *lda);

# 计算 Hermitian 矩阵乘积
int zher2k_(char *uplo, char *trans, int *n, int *k, dcomplex *alpha,
            dcomplex *a, int *lda, dcomplex *b, int *ldb, double *beta,
            dcomplex *c, int *ldc);

# 计算 Hermitian 矩阵乘积
int zherk_(char *uplo, char *trans, int *n, int *k, double *alpha,
           dcomplex *a, int *lda, double *beta, dcomplex *c, int *ldc);

# 计算 Hermitian 矩阵向量乘积
int zhpmv_(char *uplo, int *n, dcomplex *alpha, dcomplex *ap, dcomplex *x,
           int *incx, dcomplex *beta, dcomplex *y, int *incy);

# 计算 Hermitian 矩阵向量乘积并进行 Hermitian 更新
int zhpr_(char *uplo, int *n, double *alpha, dcomplex *x, int *incx,
          dcomplex *ap);

# 计算 Hermitian 矩阵向量乘积并进行 Hermitian 更新
int zhpr2_(char *uplo, int *n, dcomplex *alpha, dcomplex *x, int *incx,
           dcomplex *y, int *incy, dcomplex *ap);

# 生成 Givens 变换
int zrotg_(dcomplex *ca, dcomplex *cb, double *c, dcomplex *s);

# 对向量进行复数标量乘法
int zscal_(int *n, dcomplex *ca, dcomplex *cx, int *incx);

# 交换两个向量
int zswap_(int *n, dcomplex *cx, int *incx, dcomplex *cy, int *incy);

# 计算对称矩阵乘积
int zsymm_(char *side, char *uplo, int *m, int *n, dcomplex *alpha,
           dcomplex *a, int *lda, dcomplex *b, int *ldb, dcomplex *beta,
           dcomplex *c, int *ldc);

# 计算对称矩阵乘积并进行 Hermitian 更新
int zsyr2k_(char *uplo, char *trans, int *n, int *k, dcomplex *alpha,
            dcomplex *a, int *lda, dcomplex *b, int *ldb, dcomplex *beta,
            dcomplex *c, int *ldc);
# 计算复数矩阵乘积并将结果加到复数矩阵中
int zsyrk_(char *uplo, char *trans, int *n, int *k, dcomplex *alpha,
           dcomplex *a, int *lda, dcomplex *beta, dcomplex *c, int *ldc);

# 用复数矩阵的乘积来更新复数向量
int ztbmv_(char *uplo, char *trans, char *diag, int *n, int *k,
           dcomplex *a, int *lda, dcomplex *x, int *incx);

# 用复数矩阵的乘积来解复数向量
int ztbsv_(char *uplo, char *trans, char *diag, int *n, int *k,
           dcomplex *a, int *lda, dcomplex *x, int *incx);

# 用压缩的复数矩阵的乘积来更新复数向量
int ztpmv_(char *uplo, char *trans, char *diag, int *n, dcomplex *ap,
           dcomplex *x, int *incx);

# 用压缩的复数矩阵的乘积来解复数向量
int ztpsv_(char *uplo, char *trans, char *diag, int *n, dcomplex *ap,
           dcomplex *x, int *incx);

# 计算复数矩阵的乘积并将结果加到复数矩阵中
int ztrmm_(char *side, char *uplo, char *transa, char *diag, int *m,
           int *n, dcomplex *alpha, dcomplex *a, int *lda, dcomplex *b,
           int *ldb);

# 用复数矩阵的乘积来更新复数向量
int ztrmv_(char *uplo, char *trans, char *diag, int *n, dcomplex *a,
           int *lda, dcomplex *x, int *incx);

# 用复数矩阵的乘积来解复数向量
int ztrsm_(char *side, char *uplo, char *transa, char *diag, int *m,
           int *n, dcomplex *alpha, dcomplex *a, int *lda, dcomplex *b,
           int *ldb);

# 用复数矩阵的乘积来解复数向量
int ztrsv_(char *uplo, char *trans, char *diag, int *n, dcomplex *a,
           int *lda, dcomplex *x, int *incx);
```