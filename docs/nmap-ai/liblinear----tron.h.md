# `nmap\liblinear\tron.h`

```cpp
#ifndef _TRON_H
#define _TRON_H

class function
{
public:
    // 定义纯虚函数，用于计算函数值
    virtual double fun(double *w) = 0 ;
    // 定义纯虚函数，用于计算函数梯度
    virtual void grad(double *w, double *g) = 0 ;
    // 定义纯虚函数，用于计算 Hessian 矩阵与向量的乘积
    virtual void Hv(double *s, double *Hs) = 0 ;

    // 定义纯虚函数，用于获取变量的数量
    virtual int get_nr_variable(void) = 0 ;
    // 定义虚析构函数
    virtual ~function(void){}
};

class TRON
{
public:
    // 构造函数，初始化 TRON 对象
    TRON(const function *fun_obj, double eps = 0.1, int max_iter = 1000);
    // 析构函数，释放 TRON 对象
    ~TRON();

    // TRON 算法的主要函数，用于优化参数 w
    void tron(double *w);
    // 设置打印信息的函数指针
    void set_print_string(void (*i_print) (const char *buf));

private:
    // 用于计算共轭梯度法的子函数
    int trcg(double delta, double *g, double *s, double *r);
    // 计算向量的无穷范数
    double norm_inf(int n, double *x);

    // 定义精度
    double eps;
    // 定义最大迭代次数
    int max_iter;
    // 定义函数对象指针
    function *fun_obj;
    // 打印信息的函数
    void info(const char *fmt,...);
    // 打印信息的函数指针
    void (*tron_print_string)(const char *buf);
};
#endif
```