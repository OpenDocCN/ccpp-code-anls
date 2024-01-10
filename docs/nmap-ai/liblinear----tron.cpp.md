# `nmap\liblinear\tron.cpp`

```
#include <math.h>  // 包含数学函数库
#include <stdio.h>  // 包含标准输入输出库
#include <string.h>  // 包含字符串处理库
#include <stdarg.h>  // 包含可变参数列表库
#include "tron.h"  // 包含自定义的头文件 tron.h

#ifndef min
template <class T> static inline T min(T x,T y) { return (x<y)?x:y; }  // 定义一个模板函数，返回两个数中较小的一个
#endif

#ifndef max
template <class T> static inline T max(T x,T y) { return (x>y)?x:y; }  // 定义一个模板函数，返回两个数中较大的一个
#endif

#ifdef __cplusplus
extern "C" {
#endif

extern double dnrm2_(int *, double *, int *);  // 声明一个外部的函数 dnrm2_
extern double ddot_(int *, double *, int *, double *, int *);  // 声明一个外部的函数 ddot_
extern int daxpy_(int *, double *, double *, int *, double *, int *);  // 声明一个外部的函数 daxpy_
extern int dscal_(int *, double *, double *, int *);  // 声明一个外部的函数 dscal_

#ifdef __cplusplus
}
#endif

static void default_print(const char *buf)  // 定义一个静态函数 default_print，用于输出字符串到标准输出
{
    fputs(buf,stdout);
    fflush(stdout);
}

void TRON::info(const char *fmt,...)  // 定义一个成员函数 info，用于格式化输出信息
{
    char buf[BUFSIZ];
    va_list ap;
    va_start(ap,fmt);
    vsprintf(buf,fmt,ap);
    va_end(ap);
    (*tron_print_string)(buf);  // 调用 tron_print_string 函数指针，输出格式化后的信息
}

TRON::TRON(const function *fun_obj, double eps, int max_iter)  // 定义构造函数 TRON，初始化成员变量
{
    this->fun_obj=const_cast<function *>(fun_obj);
    this->eps=eps;
    this->max_iter=max_iter;
    tron_print_string = default_print;  // 将 tron_print_string 函数指针指向 default_print 函数
}

TRON::~TRON()  // 定义析构函数 TRON
{
}

void TRON::tron(double *w)  // 定义成员函数 tron，用于执行特定的优化算法
{
    // Parameters for updating the iterates.
    double eta0 = 1e-4, eta1 = 0.25, eta2 = 0.75;  // 定义更新迭代的参数

    // Parameters for updating the trust region size delta.
    double sigma1 = 0.25, sigma2 = 0.5, sigma3 = 4;  // 定义更新信赖域大小的参数

    int n = fun_obj->get_nr_variable();  // 获取变量的数量
    int i, cg_iter;  // 定义循环变量
    double delta, snorm, one=1.0;  // 定义变量
    double alpha, f, fnew, prered, actred, gs;  // 定义变量
    int search = 1, iter = 1, inc = 1;  // 定义变量
    double *s = new double[n];  // 动态分配数组内存
    double *r = new double[n];  // 动态分配数组内存
    double *w_new = new double[n];  // 动态分配数组内存
    double *g = new double[n];  // 动态分配数组内存

    for (i=0; i<n; i++)  // 循环初始化数组 w
        w[i] = 0;

    f = fun_obj->fun(w);  // 计算目标函数值
    fun_obj->grad(w, g);  // 计算目标函数的梯度
    delta = dnrm2_(&n, g, &inc);  // 计算 g 的二范数
    double gnorm1 = delta;  // 保存 g 的二范数
    double gnorm = gnorm1;  // 保存 g 的二范数

    if (gnorm <= eps*gnorm1)  // 判断是否满足终止条件
        search = 0;

    iter = 1;  // 初始化迭代次数

    while (iter <= max_iter && search)  // 迭代循环
    }

    delete[] g;  // 释放数组内存
    delete[] r;  // 释放数组内存
    delete[] w_new;  // 释放数组内存
    delete[] s;  // 释放数组内存
}
int TRON::trcg(double delta, double *g, double *s, double *r)
{
    int i, inc = 1;  // 定义变量 i 和 inc，设置 inc 的值为 1
    int n = fun_obj->get_nr_variable();  // 获取变量的维度
    double one = 1;  // 定义变量 one，设置其值为 1
    double *d = new double[n];  // 创建长度为 n 的 double 类型数组
    double *Hd = new double[n];  // 创建长度为 n 的 double 类型数组
    double rTr, rnewTrnew, alpha, beta, cgtol;  // 定义变量 rTr, rnewTrnew, alpha, beta, cgtol

    for (i=0; i<n; i++)  // 循环 n 次
    {
        s[i] = 0;  // 将 s 数组的第 i 个元素设置为 0
        r[i] = -g[i];  // 将 r 数组的第 i 个元素设置为 -g 数组的第 i 个元素
        d[i] = r[i];  // 将 d 数组的第 i 个元素设置为 r 数组的第 i 个元素
    }
    cgtol = 0.1*dnrm2_(&n, g, &inc);  // 计算 cgtol 的值

    int cg_iter = 0;  // 定义变量 cg_iter，设置其值为 0
    rTr = ddot_(&n, r, &inc, r, &inc);  // 计算 rTr 的值
    while (1)  // 进入循环
    {
        if (dnrm2_(&n, r, &inc) <= cgtol)  // 如果满足条件
            break;  // 退出循环
        cg_iter++;  // cg_iter 加 1
        fun_obj->Hv(d, Hd);  // 调用 Hv 函数

        alpha = rTr/ddot_(&n, d, &inc, Hd, &inc);  // 计算 alpha 的值
        daxpy_(&n, &alpha, d, &inc, s, &inc);  // 计算 s 的值
        if (dnrm2_(&n, s, &inc) > delta)  // 如果满足条件
        {
            info("cg reaches trust region boundary\n");  // 输出信息
            alpha = -alpha;  // 设置 alpha 的值
            daxpy_(&n, &alpha, d, &inc, s, &inc);  // 计算 s 的值

            double std = ddot_(&n, s, &inc, d, &inc);  // 计算 std 的值
            double sts = ddot_(&n, s, &inc, s, &inc);  // 计算 sts 的值
            double dtd = ddot_(&n, d, &inc, d, &inc);  // 计算 dtd 的值
            double dsq = delta*delta;  // 计算 dsq 的值
            double rad = sqrt(std*std + dtd*(dsq-sts));  // 计算 rad 的值
            if (std >= 0)  // 如果满足条件
                alpha = (dsq - sts)/(std + rad);  // 设置 alpha 的值
            else
                alpha = (rad - std)/dtd;  // 设置 alpha 的值
            daxpy_(&n, &alpha, d, &inc, s, &inc);  // 计算 s 的值
            alpha = -alpha;  // 设置 alpha 的值
            daxpy_(&n, &alpha, Hd, &inc, r, &inc);  // 计算 r 的值
            break;  // 退出循环
        }
        alpha = -alpha;  // 设置 alpha 的值
        daxpy_(&n, &alpha, Hd, &inc, r, &inc);  // 计算 r 的值
        rnewTrnew = ddot_(&n, r, &inc, r, &inc);  // 计算 rnewTrnew 的值
        beta = rnewTrnew/rTr;  // 计算 beta 的值
        dscal_(&n, &beta, d, &inc);  // 计算 d 的值
        daxpy_(&n, &one, r, &inc, d, &inc);  // 计算 d 的值
        rTr = rnewTrnew;  // 设置 rTr 的值
    }

    delete[] d;  // 释放内存
    delete[] Hd;  // 释放内存

    return(cg_iter);  // 返回 cg_iter 的值
}

double TRON::norm_inf(int n, double *x)
{
    double dmax = fabs(x[0]);  // 设置 dmax 的值
    for (int i=1; i<n; i++)  // 循环 n-1 次
        if (fabs(x[i]) >= dmax)  // 如果满足条件
            dmax = fabs(x[i]);  // 设置 dmax 的值
    return(dmax);  // 返回 dmax 的值
}

void TRON::set_print_string(void (*print_string) (const char *buf))
{
    # 将print_string函数的引用赋值给tron_print_string变量
    tron_print_string = print_string;
# 闭合前面的函数定义
```