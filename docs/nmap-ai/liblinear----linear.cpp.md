# `nmap\liblinear\linear.cpp`

```cpp
#include <math.h> // 包含数学函数库
#include <stdio.h> // 包含标准输入输出函数库
#include <stdlib.h> // 包含标准库函数库
#include <string.h> // 包含字符串处理函数库
#include <stdarg.h> // 包含可变参数函数库
#include "linear.h" // 包含自定义的 linear.h 头文件
#include "tron.h" // 包含自定义的 tron.h 头文件
typedef signed char schar; // 定义有符号字符类型 schar
template <class T> static inline void swap(T& x, T& y) { T t=x; x=y; y=t; } // 定义模板函数 swap，用于交换两个变量的值
#ifndef min
template <class T> static inline T min(T x,T y) { return (x<y)?x:y; } // 定义模板函数 min，返回两个值中较小的值
#endif
#ifndef max
template <class T> static inline T max(T x,T y) { return (x>y)?x:y; } // 定义模板函数 max，返回两个值中较大的值
#endif
template <class S, class T> static inline void clone(T*& dst, S* src, int n)
{   
    dst = new T[n]; // 分配内存给目标数组
    memcpy((void *)dst,(void *)src,sizeof(T)*n); // 复制源数组的内容到目标数组
}
#define Malloc(type,n) (type *)malloc((n)*sizeof(type)) // 定义宏 Malloc，用于分配内存
#define INF HUGE_VAL // 定义宏 INF，表示无穷大

static void print_string_stdout(const char *s)
{
    fputs(s,stdout); // 将字符串 s 输出到标准输出
    fflush(stdout); // 刷新标准输出流
}

static void (*liblinear_print_string) (const char *) = &print_string_stdout; // 定义函数指针 liblinear_print_string，指向 print_string_stdout 函数

#if 1
static void info(const char *fmt,...) // 定义静态函数 info，用于格式化输出信息
{
    char buf[BUFSIZ]; // 定义字符数组 buf，用于存储格式化后的信息
    va_list ap; // 定义可变参数列表
    va_start(ap,fmt); // 初始化可变参数列表
    vsprintf(buf,fmt,ap); // 格式化输出信息到 buf
    va_end(ap); // 结束可变参数列表
    (*liblinear_print_string)(buf); // 调用 liblinear_print_string 函数指针，输出格式化后的信息
}
#else
static void info(const char *fmt,...) {} // 如果条件不成立，则定义空的 info 函数
#endif

class l2r_lr_fun : public function // 定义 l2r_lr_fun 类，继承自 function 类
{
public:
    l2r_lr_fun(const problem *prob, double Cp, double Cn); // 构造函数
    ~l2r_lr_fun(); // 析构函数

    double fun(double *w); // 计算目标函数值
    void grad(double *w, double *g); // 计算目标函数的梯度
    void Hv(double *s, double *Hs); // 计算 Hessian 矩阵与向量的乘积

    int get_nr_variable(void); // 获取变量的数量

private:
    void Xv(double *v, double *Xv); // 计算 X 与向量 v 的乘积
    void XTv(double *v, double *XTv); // 计算 X 转置与向量 v 的乘积

    double *C; // 惩罚参数
    double *z; // 中间变量
    double *D; // 中间变量
    const problem *prob; // 指向 problem 类型的指针
};

l2r_lr_fun::l2r_lr_fun(const problem *prob, double Cp, double Cn) // l2r_lr_fun 类的构造函数
{
    int i;
    int l=prob->l; // 获取问题的样本数量
    int *y=prob->y; // 获取问题的标签

    this->prob = prob; // 将 prob 赋值给成员变量 prob

    z = new double[l]; // 分配内存给 z 数组
    D = new double[l]; // 分配内存给 D 数组
    C = new double[l]; // 分配内存给 C 数组

    for (i=0; i<l; i++) // 遍历样本
    {
        if (y[i] == 1) // 如果标签为 1
            C[i] = Cp; // 则将惩罚参数设置为 Cp
        else
            C[i] = Cn; // 否则将惩罚参数设置为 Cn
    }
}

l2r_lr_fun::~l2r_lr_fun() // l2r_lr_fun 类的析构函数
{
    delete[] z; // 释放 z 数组的内存
    delete[] D; // 释放 D 数组的内存
    delete[] C; // 释放 C 数组的内存
}


double l2r_lr_fun::fun(double *w) // 计算目标函数值
{
    int i;
    double f=0; // 初始化目标函数值为 0
    int *y=prob->y; // 获取问题的标签
    int l=prob->l; // 获取问题的样本数量
    # 获取变量的数量
    int w_size=get_nr_variable();
    
    # 计算向量 w 与向量 z 的内积，结果保存在向量 Xv 中
    Xv(w, z);
    
    # 遍历每个样本
    for(i=0;i<l;i++)
    {
        # 计算 y[i]*z[i]
        double yz = y[i]*z[i];
        
        # 根据 yz 的正负情况计算损失函数
        if (yz >= 0)
            f += C[i]*log(1 + exp(-yz));
        else
            f += C[i]*(-yz+log(1 + exp(yz)));
    }
    
    # 将损失函数值乘以 2
    f = 2*f;
    
    # 计算正则化项
    for(i=0;i<w_size;i++)
        f += w[i]*w[i];
    
    # 将结果除以 2
    f /= 2.0;
    
    # 返回最终的损失函数值
    return(f);
}

void l2r_lr_fun::grad(double *w, double *g)
{
    int i;
    int *y=prob->y; // 获取问题中的标签数组
    int l=prob->l; // 获取问题中的样本数量
    int w_size=get_nr_variable(); // 获取变量的数量

    for(i=0;i<l;i++)
    {
        z[i] = 1/(1 + exp(-y[i]*z[i])); // 计算 z[i] 的值
        D[i] = z[i]*(1-z[i]); // 计算 D[i] 的值
        z[i] = C[i]*(z[i]-1)*y[i]; // 计算 z[i] 的值
    }
    XTv(z, g); // 调用 XTv 函数

    for(i=0;i<w_size;i++)
        g[i] = w[i] + g[i]; // 计算梯度
}

int l2r_lr_fun::get_nr_variable(void)
{
    return prob->n; // 返回问题中的变量数量
}

void l2r_lr_fun::Hv(double *s, double *Hs)
{
    int i;
    int l=prob->l; // 获取问题中的样本数量
    int w_size=get_nr_variable(); // 获取变量的数量
    double *wa = new double[l]; // 创建长度为 l 的动态数组

    Xv(s, wa); // 调用 Xv 函数
    for(i=0;i<l;i++)
        wa[i] = C[i]*D[i]*wa[i]; // 计算 wa[i] 的值

    XTv(wa, Hs); // 调用 XTv 函数
    for(i=0;i<w_size;i++)
        Hs[i] = s[i] + Hs[i]; // 计算 Hs[i] 的值
    delete[] wa; // 释放动态数组内存
}

void l2r_lr_fun::Xv(double *v, double *Xv)
{
    int i;
    int l=prob->l; // 获取问题中的样本数量
    feature_node **x=prob->x; // 获取问题中的特征节点数组

    for(i=0;i<l;i++)
    {
        feature_node *s=x[i]; // 获取特征节点数组中的第 i 个节点
        Xv[i]=0; // 初始化 Xv[i] 的值
        while(s->index!=-1)
        {
            Xv[i]+=v[s->index-1]*s->value; // 计算 Xv[i] 的值
            s++; // 移动到下一个特征节点
        }
    }
}

void l2r_lr_fun::XTv(double *v, double *XTv)
{
    int i;
    int l=prob->l; // 获取问题中的样本数量
    int w_size=get_nr_variable(); // 获取变量的数量
    feature_node **x=prob->x; // 获取问题中的特征节点数组

    for(i=0;i<w_size;i++)
        XTv[i]=0; // 初始化 XTv[i] 的值
    for(i=0;i<l;i++)
    {
        feature_node *s=x[i]; // 获取特征节点数组中的第 i 个节点
        while(s->index!=-1)
        {
            XTv[s->index-1]+=v[i]*s->value; // 计算 XTv[s->index-1] 的值
            s++; // 移动到下一个特征节点
        }
    }
}

class l2r_l2_svc_fun : public function
{
public:
    l2r_l2_svc_fun(const problem *prob, double Cp, double Cn);
    ~l2r_l2_svc_fun();

    double fun(double *w);
    void grad(double *w, double *g);
    void Hv(double *s, double *Hs);

    int get_nr_variable(void);

private:
    void Xv(double *v, double *Xv);
    void subXv(double *v, double *Xv);
    void subXTv(double *v, double *XTv);

    double *C;
    double *z;
    double *D;
    int *I;
    int sizeI;
    const problem *prob;
};

l2r_l2_svc_fun::l2r_l2_svc_fun(const problem *prob, double Cp, double Cn)
{
    int i;
    int l=prob->l; // 获取问题中的样本数量
    int *y=prob->y; // 获取问题中的标签数组
    # 将参数 prob 赋值给当前对象的 prob 属性
    this->prob = prob;

    # 创建一个长度为 l 的 double 类型数组，并将其地址赋给 z 指针
    z = new double[l];
    # 创建一个长度为 l 的 double 类型数组，并将其地址赋给 D 指针
    D = new double[l];
    # 创建一个长度为 l 的 double 类型数组，并将其地址赋给 C 指针
    C = new double[l];
    # 创建一个长度为 l 的 int 类型数组，并将其地址赋给 I 指针
    I = new int[l];

    # 遍历数组 y
    for (i=0; i<l; i++)
    {
        # 如果 y[i] 的值为 1，则将 Cp 赋值给 C[i]
        if (y[i] == 1)
            C[i] = Cp;
        # 否则将 Cn 赋值给 C[i]
        else
            C[i] = Cn;
    }
# l2r_l2_svc_fun 类的析构函数，用于释放动态分配的内存
l2r_l2_svc_fun::~l2r_l2_svc_fun()
{
    # 释放 z 数组的内存
    delete[] z;
    # 释放 D 数组的内存
    delete[] D;
    # 释放 C 数组的内存
    delete[] C;
    # 释放 I 数组的内存
    delete[] I;
}

# l2r_l2_svc_fun 类的 fun 方法，用于计算目标函数的值
double l2r_l2_svc_fun::fun(double *w)
{
    int i;
    double f=0;
    int *y=prob->y;
    int l=prob->l;
    int w_size=get_nr_variable();

    # 计算 Xv(w, z) 的值
    Xv(w, z);
    for(i=0;i<l;i++)
    {
        # 计算 z[i] = y[i]*z[i]
        z[i] = y[i]*z[i];
        double d = 1-z[i];
        if (d > 0)
            # 计算目标函数的值
            f += C[i]*d*d;
    }
    f = 2*f;
    for(i=0;i<w_size;i++)
        # 计算目标函数的值
        f += w[i]*w[i];
    f /= 2.0;

    return(f);
}

# l2r_l2_svc_fun 类的 grad 方法，用于计算目标函数的梯度
void l2r_l2_svc_fun::grad(double *w, double *g)
{
    int i;
    int *y=prob->y;
    int l=prob->l;
    int w_size=get_nr_variable();

    sizeI = 0;
    for (i=0;i<l;i++)
        if (z[i] < 1)
        {
            z[sizeI] = C[i]*y[i]*(z[i]-1);
            I[sizeI] = i;
            sizeI++;
        }
    # 计算 subXTv(z, g) 的值
    subXTv(z, g);

    for(i=0;i<w_size;i++)
        # 计算目标函数的梯度
        g[i] = w[i] + 2*g[i];
}

# l2r_l2_svc_fun 类的 get_nr_variable 方法，用于获取变量的数量
int l2r_l2_svc_fun::get_nr_variable(void)
{
    return prob->n;
}

# l2r_l2_svc_fun 类的 Hv 方法，用于计算 Hessian 矩阵与向量的乘积
void l2r_l2_svc_fun::Hv(double *s, double *Hs)
{
    int i;
    int l=prob->l;
    int w_size=get_nr_variable();
    double *wa = new double[l];

    # 计算 subXv(s, wa) 的值
    subXv(s, wa);
    for(i=0;i<sizeI;i++)
        wa[i] = C[I[i]]*wa[i];

    # 计算 subXTv(wa, Hs) 的值
    subXTv(wa, Hs);
    for(i=0;i<w_size;i++)
        # 计算 Hessian 矩阵与向量的乘积
        Hs[i] = s[i] + 2*Hs[i];
    delete[] wa;
}

# l2r_l2_svc_fun 类的 Xv 方法，用于计算特征向量与向量的乘积
void l2r_l2_svc_fun::Xv(double *v, double *Xv)
{
    int i;
    int l=prob->l;
    feature_node **x=prob->x;

    for(i=0;i<l;i++)
    {
        feature_node *s=x[i];
        Xv[i]=0;
        while(s->index!=-1)
        {
            Xv[i]+=v[s->index-1]*s->value;
            s++;
        }
    }
}

# l2r_l2_svc_fun 类的 subXv 方法，用于计算子集特征向量与向量的乘积
void l2r_l2_svc_fun::subXv(double *v, double *Xv)
{
    int i;
    feature_node **x=prob->x;

    for(i=0;i<sizeI;i++)
    {
        feature_node *s=x[I[i]];
        Xv[i]=0;
        while(s->index!=-1)
        {
            Xv[i]+=v[s->index-1]*s->value;
            s++;
        }
    }
}

# l2r_l2_svc_fun 类的 subXTv 方法，用于计算子集特征向量的转置与向量的乘积
void l2r_l2_svc_fun::subXTv(double *v, double *XTv)
{
    int i;
    int w_size=get_nr_variable();
    feature_node **x=prob->x;
    # 初始化数组 XTv 的每个元素为 0
    for(i=0;i<w_size;i++)
        XTv[i]=0;
    # 遍历数组 sizeI
    for(i=0;i<sizeI;i++)
    {
        # 获取特征节点指针
        feature_node *s=x[I[i]];
        # 当特征节点的索引不为 -1 时，执行循环
        while(s->index!=-1)
        {
            # 更新 XTv 数组中的值
            XTv[s->index-1]+=v[i]*s->value;
            # 指针移动到下一个特征节点
            s++;
        }
    }
// 结束类定义
}

// 定义一个多类支持向量机的坐标下降算法，由Crammer和Singer提出
//
//  min_{\alpha}  0.5 \sum_m ||w_m(\alpha)||^2 + \sum_i \sum_m e^m_i alpha^m_i
//    s.t.     \alpha^m_i <= C^m_i \forall m,i , \sum_m \alpha^m_i=0 \forall i
// 
//  其中 e^m_i = 0 如果 y_i  = m,
//        e^m_i = 1 如果 y_i != m,
//  C^m_i = C 如果 m  = y_i, 
//  C^m_i = 0 如果 m != y_i, 
//  而 w_m(\alpha) = \sum_i \alpha^m_i x_i 
//
// 给定: 
// x, y, C
// eps 是停止容差
//
// 解将被放入 w 中
//
// 参见LIBLINEAR论文的附录，Fan等人(2008)

#define GETI(i) (prob->y[i])
// 为了支持实例的权重，使用GETI(i) (i)

class Solver_MCSVM_CS
{
    public:
        Solver_MCSVM_CS(const problem *prob, int nr_class, double *C, double eps=0.1, int max_iter=100000);
        ~Solver_MCSVM_CS();
        void Solve(double *w);
    private:
        void solve_sub_problem(double A_i, int yi, double C_yi, int active_i, double *alpha_new);
        bool be_shrunk(int i, int m, int yi, double alpha_i, double minG);
        double *B, *C, *G;
        int w_size, l;
        int nr_class;
        int max_iter;
        double eps;
        const problem *prob;
};

Solver_MCSVM_CS::Solver_MCSVM_CS(const problem *prob, int nr_class, double *weighted_C, double eps, int max_iter)
{
    this->w_size = prob->n;
    this->l = prob->l;
    this->nr_class = nr_class;
    this->eps = eps;
    this->max_iter = max_iter;
    this->prob = prob;
    this->B = new double[nr_class];
    this->G = new double[nr_class];
    this->C = weighted_C;
}

Solver_MCSVM_CS::~Solver_MCSVM_CS()
{
    delete[] B;
    delete[] G;
}

int compare_double(const void *a, const void *b)
{
    if(*(double *)a > *(double *)b)
        return -1;
    if(*(double *)a < *(double *)b)
        return 1;
    return 0;
}

void Solver_MCSVM_CS::solve_sub_problem(double A_i, int yi, double C_yi, int active_i, double *alpha_new)
{
    int r;
    double *D;

    clone(D, B, active_i);
    # 如果当前的索引 yi 小于活跃索引 active_i，则更新 D[yi] 的值
    if(yi < active_i)
        D[yi] += A_i*C_yi;
    # 对数组 D 进行快速排序，按照 double 类型的大小进行比较
    qsort(D, active_i, sizeof(double), compare_double);

    # 计算 beta 的值
    double beta = D[0] - A_i*C_yi;
    # 遍历数组 D，找到满足条件 beta < r*D[r] 的最小 r 值
    for(r=1;r<active_i && beta<r*D[r];r++)
        beta += D[r];

    # 对 beta 进行计算
    beta /= r;
    # 遍历数组 alpha_new，根据条件计算每个元素的值
    for(r=0;r<active_i;r++)
    {
        # 如果 r 等于 yi，则计算 alpha_new[r] 的值
        if(r == yi)
            alpha_new[r] = min(C_yi, (beta-B[r])/A_i);
        # 否则计算 alpha_new[r] 的值
        else
            alpha_new[r] = min((double)0, (beta - B[r])/A_i);
    }
    # 释放数组 D 的内存空间
    delete[] D;
}

bool Solver_MCSVM_CS::be_shrunk(int i, int m, int yi, double alpha_i, double minG)
{
    // 初始化边界值
    double bound = 0;
    // 如果 m 等于 yi
    if(m == yi)
        // 获取 C[GETI(i)] 的值
        bound = C[GETI(i)];
    // 如果 alpha_i 等于 bound 并且 G[m] 小于 minG
    if(alpha_i == bound && G[m] < minG)
        // 返回 true
        return true;
    // 返回 false
    return false;
}

void Solver_MCSVM_CS::Solve(double *w)
{
    int i, m, s;
    int iter = 0;
    // 分配内存空间
    double *alpha =  new double[l*nr_class];
    double *alpha_new = new double[nr_class];
    int *index = new int[l];
    double *QD = new double[l];
    int *d_ind = new int[nr_class];
    double *d_val = new double[nr_class];
    int *alpha_index = new int[nr_class*l];
    int *y_index = new int[l];
    int active_size = l;
    int *active_size_i = new int[l];
    // 计算 eps_shrink 的值
    double eps_shrink = max(10.0*eps, 1.0); // stopping tolerance for shrinking
    bool start_from_all = true;
    // 初始化
    for(i=0;i<l*nr_class;i++)
        alpha[i] = 0;
    for(i=0;i<w_size*nr_class;i++)
        w[i] = 0; 
    for(i=0;i<l;i++)
    {
        for(m=0;m<nr_class;m++)
            alpha_index[i*nr_class+m] = m;
        feature_node *xi = prob->x[i];
        QD[i] = 0;
        while(xi->index != -1)
        {
            QD[i] += (xi->value)*(xi->value);
            xi++;
        }
        active_size_i[i] = nr_class;
        y_index[i] = prob->y[i];
        index[i] = i;
    }

    while(iter < max_iter) 
    }

    // 输出优化完成的信息
    info("\noptimization finished, #iter = %d\n",iter);
    // 如果迭代次数超过最大迭代次数，输出警告信息
    if (iter >= max_iter)
        info("\nWARNING: reaching max number of iterations\n");

    // 计算目标函数值
    double v = 0;
    int nSV = 0;
    for(i=0;i<w_size*nr_class;i++)
        v += w[i]*w[i];
    v = 0.5*v;
    for(i=0;i<l*nr_class;i++)
    {
        v += alpha[i];
        if(fabs(alpha[i]) > 0)
            nSV++;
    }
    for(i=0;i<l;i++)
        v -= alpha[i*nr_class+prob->y[i]];
    // 输出目标函数值和支持向量个数
    info("Objective value = %lf\n",v);
    info("nSV = %d\n",nSV);

    // 释放内存空间
    delete [] alpha;
    delete [] alpha_new;
    delete [] index;
    delete [] QD;
    delete [] d_ind;
    delete [] d_val;
}
    # 释放动态分配的 alpha_index 数组的内存
    delete [] alpha_index;
    # 释放动态分配的 y_index 数组的内存
    delete [] y_index;
    # 释放动态分配的 active_size_i 数组的内存
    delete [] active_size_i;
// 一个坐标下降算法，用于解决L1-loss和L2-loss SVM对偶问题
//
//  min_\alpha  0.5(\alpha^T (Q + D)\alpha) - e^T \alpha,
//    s.t.      0 <= alpha_i <= upper_bound_i,
// 
//  其中 Qij = yi yj xi^T xj，而 D 是一个对角矩阵 
//
// 在L1-SVM情况下:
//         如果 y_i = 1，那么 upper_bound_i = Cp
//         如果 y_i = -1，那么 upper_bound_i = Cn
//         D_ii = 0
// 在L2-SVM情况下:
//         upper_bound_i = INF
//         如果 y_i = 1，那么 D_ii = 1/(2*Cp)
//         如果 y_i = -1，那么 D_ii = 1/(2*Cn)
//
// 给定: 
// x, y, Cp, Cn
// eps 是停止容差
//
// 解将被放入 w 中
// 
// 参见Hsieh等人的ICML 2008中的算法3
#undef GETI
#define GETI(i) (y[i]+1)
// 为了支持实例的权重，使用GETI(i) (i)

static void solve_l2r_l1l2_svc(
    const problem *prob, double *w, double eps, 
    double Cp, double Cn, int solver_type)
{
    int l = prob->l; // 实例数
    int w_size = prob->n; // 特征数
    int i, s, iter = 0; // 迭代次数
    double C, d, G; // 参数
    double *QD = new double[l]; // 存储 Q + D 的数组
    int max_iter = 1000; // 最大迭代次数
    int *index = new int[l]; // 索引数组
    double *alpha = new double[l]; // 存储 alpha 的数组
    schar *y = new schar[l]; // 存储 y 的数组
    int active_size = l; // 活跃集大小

    // PG: 投影梯度，用于收缩和停止
    double PG;
    double PGmax_old = INF;
    double PGmin_old = -INF;
    double PGmax_new, PGmin_new;

    // 默认的 solver_type: L2R_L2LOSS_SVC_DUAL
    double diag[3] = {0.5/Cn, 0, 0.5/Cp}; // 对角线数组
    double upper_bound[3] = {INF, 0, INF}; // 上界数组
    if(solver_type == L2R_L1LOSS_SVC_DUAL)
    {
        diag[0] = 0;
        diag[2] = 0;
        upper_bound[0] = Cn;
        upper_bound[2] = Cp;
    }

    for(i=0; i<w_size; i++)
        w[i] = 0; // 初始化 w
    for(i=0; i<l; i++)
    {
        // 初始化 alpha 数组的值为 0
        alpha[i] = 0;
        // 如果样本的标签值大于 0，则将 y[i] 赋值为 +1
        if(prob->y[i] > 0)
        {
            y[i] = +1; 
        }
        // 否则将 y[i] 赋值为 -1
        else
        {
            y[i] = -1;
        }
        // 计算 QD[i] 的值，其中包括对角线元素的值
        QD[i] = diag[GETI(i)];

        // 获取第 i 个样本的特征向量
        feature_node *xi = prob->x[i];
        // 计算 QD[i] 的值，包括特征向量的值
        while (xi->index != -1)
        {
            QD[i] += (xi->value)*(xi->value);
            xi++;
        }
        // 将索引 i 存储到 index 数组中
        index[i] = i;
    }

    // 迭代更新权重 w
    while (iter < max_iter)
    }

    // 输出优化完成的信息，包括迭代次数
    info("\noptimization finished, #iter = %d\n",iter);
    // 如果达到最大迭代次数，则输出警告信息
    if (iter >= max_iter)
        info("\nWARNING: reaching max number of iterations\nUsing -s 2 may be faster (also see FAQ)\n\n");

    // 计算目标函数的值
    double v = 0;
    int nSV = 0;
    for(i=0; i<w_size; i++)
        v += w[i]*w[i];
    for(i=0; i<l; i++)
    {
        v += alpha[i]*(alpha[i]*diag[GETI(i)] - 2);
        if(alpha[i] > 0)
            ++nSV;
    }
    // 输出目标函数的值和支持向量的数量
    info("Objective value = %lf\n",v/2);
    info("nSV = %d\n",nSV);

    // 释放动态分配的内存
    delete [] QD;
    delete [] alpha;
    delete [] y;
    delete [] index;
}
// 用于支持实例权重，使用 GETI(i) (i)

// 用于 L2 正则化逻辑回归问题的对偶坐标下降算法
// 最小化目标函数：0.5(\alpha^T Q \alpha) + \sum \alpha_i log (\alpha_i) + (upper_bound_i - alpha_i) log (upper_bound_i - alpha_i)
// 约束条件：0 <= alpha_i <= upper_bound_i
// 其中 Qij = yi yj xi^T xj
// upper_bound_i = Cp (y_i = 1)
// upper_bound_i = Cn (y_i = -1)
// 给定：x, y, Cp, Cn
// eps 是停止容忍度
// 解将被放入 w 中
// 参见 Yu 等人的 MLJ 2010 中的算法 5

#undef GETI
#define GETI(i) (y[i]+1)
// 用于支持实例权重，使用 GETI(i) (i)

void solve_l2r_lr_dual(const problem *prob, double *w, double eps, double Cp, double Cn)
{
    int l = prob->l; // 实例数
    int w_size = prob->n; // 特征数
    int i, s, iter = 0; // 迭代次数
    double *xTx = new double[l]; // 存储 xTx
    int max_iter = 1000; // 最大迭代次数
    int *index = new int[l]; // 存储索引
    double *alpha = new double[2*l]; // 存储 alpha 和 C - alpha
    schar *y = new schar[l]; // 存储 y
    int max_inner_iter = 100; // 内部牛顿迭代次数
    double innereps = 1e-2; // 内部容忍度
    double innereps_min = min(1e-8, eps); // 内部最小容忍度
    double upper_bound[3] = {Cn, 0, Cp}; // 上界

    for(i=0; i<w_size; i++)
        w[i] = 0; // 初始化 w
    for(i=0; i<l; i++)
    {
        if(prob->y[i] > 0)
        {
            y[i] = +1; // 设置 y
        }
        else
        {
            y[i] = -1; // 设置 y
        }
        alpha[2*i] = min(0.001*upper_bound[GETI(i)], 1e-8); // 设置 alpha
        alpha[2*i+1] = upper_bound[GETI(i)] - alpha[2*i]; // 设置 alpha

        xTx[i] = 0; // 初始化 xTx
        feature_node *xi = prob->x[i]; // 获取特征节点
        while (xi->index != -1)
        {
            xTx[i] += (xi->value)*(xi->value); // 计算 xTx
            w[xi->index-1] += y[i]*alpha[2*i]*xi->value; // 更新 w
            xi++; // 下一个特征节点
        }
        index[i] = i; // 设置索引
    }

    while (iter < max_iter)
    {
        // 迭代更新 alpha
    }

    info("\noptimization finished, #iter = %d\n",iter); // 输出迭代次数
    if (iter >= max_iter)
        info("\nWARNING: reaching max number of iterations\nUsing -s 0 may be faster (also see FAQ)\n\n"); // 警告：达到最大迭代次数

    // 计算目标函数值
}
    # 初始化变量 v 为 0
    double v = 0;
    # 遍历数组 w，计算 w[i] 的平方和
    for(i=0; i<w_size; i++)
        v += w[i] * w[i];
    # 将结果乘以 0.5
    v *= 0.5;
    # 遍历数组 alpha，计算目标函数的值
    for(i=0; i<l; i++)
        v += alpha[2*i] * log(alpha[2*i]) + alpha[2*i+1] * log(alpha[2*i+1]) 
            - upper_bound[GETI(i)] * log(upper_bound[GETI(i)]);
    # 打印目标函数的值
    info("Objective value = %lf\n", v);

    # 释放动态分配的内存
    delete [] xTx;
    delete [] alpha;
    delete [] y;
    delete [] index;
// 关闭大括号，结束 solve_l1r_l2_svc 函数的定义
}

// 一个坐标下降算法，用于 L1 正则化的 L2 损失支持向量分类
//
//  min_w \sum |wj| + C \sum max(0, 1-yi w^T xi)^2,
//
// 给定： 
// x, y, Cp, Cn
// eps 是停止容差
//
// 解将被放入 w 中
//
// 参见 Yuan et al. (2010) 和 LIBLINEAR 论文的附录，Fan et al. (2008)

// 定义 GETI 宏，用于支持实例的权重
#undef GETI
#define GETI(i) (y[i]+1)
// 要支持实例的权重，使用 GETI(i) (i)

// 定义 solve_l1r_l2_svc 函数，接受问题 prob_col、权重向量 w、停止容差 eps、正类别惩罚参数 Cp、负类别惩罚参数 Cn
static void solve_l1r_l2_svc(
    problem *prob_col, double *w, double eps, 
    double Cp, double Cn)
{
    // 获取问题 prob_col 的实例数
    int l = prob_col->l;
    // 获取问题 prob_col 的特征数
    int w_size = prob_col->n;
    int j, s, iter = 0;
    // 最大迭代次数
    int max_iter = 1000;
    // 活跃特征数
    int active_size = w_size;
    // 最大线搜索次数
    int max_num_linesearch = 20;

    // 正则化参数
    double sigma = 0.01;
    double d, G_loss, G, H;
    double Gmax_old = INF;
    double Gmax_new, Gnorm1_new;
    double Gnorm1_init;
    double d_old, d_diff;
    double loss_old, loss_new;
    double appxcond, cond;

    // 创建大小为 w_size 的整型数组 index
    int *index = new int[w_size];
    // 创建大小为 l 的有符号字符数组 y
    schar *y = new schar[l];
    // 创建大小为 l 的双精度浮点数数组 b，b = 1-ywTx
    double *b = new double[l];
    // 创建大小为 w_size 的双精度浮点数数组 xj_sq
    double *xj_sq = new double[w_size];
    // 创建指向特征节点的指针 x
    feature_node *x;

    // 创建大小为 3 的双精度浮点数数组 C，包含 Cn, 0, Cp
    double C[3] = {Cn,0,Cp};

    // 初始化 b、y、w、index、xj_sq
    for(j=0; j<l; j++)
    {
        b[j] = 1;
        if(prob_col->y[j] > 0)
            y[j] = 1;
        else
            y[j] = -1;
    }
    for(j=0; j<w_size; j++)
    {
        w[j] = 0;
        index[j] = j;
        xj_sq[j] = 0;
        x = prob_col->x[j];
        while(x->index != -1)
        {
            int ind = x->index-1;
            double val = x->value;
            x->value *= y[ind]; // x->value 存储 yi*xij
            xj_sq[j] += C[GETI(ind)]*val*val;
            x++;
        }
    }

    // 迭代直到达到最大迭代次数
    while(iter < max_iter)
    }

    // 输出优化完成信息，包括迭代次数
    info("\noptimization finished, #iter = %d\n", iter);
    // 如果达到最大迭代次数，输出警告信息
    if(iter >= max_iter)
        info("\nWARNING: reaching max number of iterations\n");

    // 计算目标值

    double v = 0;
    int nnz = 0;
    for(j=0; j<w_size; j++)
    {
        // 从prob_col->x数组中获取元素赋值给x
        x = prob_col->x[j];
        // 当x的索引不为-1时，执行循环
        while(x->index != -1)
        {
            // 将x->value恢复为原值乘以prob_col->y[x->index-1]
            x->value *= prob_col->y[x->index-1]; // restore x->value
            // 指针x向后移动
            x++;
        }
        // 如果w[j]不等于0
        if(w[j] != 0)
        {
            // 将w[j]的绝对值加到v上
            v += fabs(w[j]);
            // 非零元素计数加一
            nnz++;
        }
    }
    // 遍历数组b，如果b[j]大于0，则将C[GETI(j)]*b[j]*b[j]加到v上
    for(j=0; j<l; j++)
        if(b[j] > 0)
            v += C[GETI(j)]*b[j]*b[j];
    
    // 输出目标值
    info("Objective value = %lf\n", v);
    // 输出非零元素个数和特征数
    info("#nonzeros/#features = %d/%d\n", nnz, w_size);
    
    // 释放内存
    delete [] index;
    delete [] y;
    delete [] b;
    delete [] xj_sq;
    }
// 定义宏GETI(i)，用于获取y[i]的值
#define GETI(i) (y[i]+1)
// 支持实例权重，使用GETI(i) (i)

// 解决L1正则化逻辑回归问题的坐标下降算法
//
//  min_w \sum |wj| + C \sum log(1+exp(-yi w^T xi)),
//
// 给定： 
// x, y, Cp, Cn
// eps是停止容差
//
// 解将放在w中
//
// 参见Yuan等人(2011)和LIBLINEAR论文附录，Fan等人(2008)

// 取消GETI
#undef GETI
#define GETI(i) (y[i]+1)
// 支持实例权重，使用GETI(i) (i)

static void solve_l1r_lr(
    const problem *prob_col, double *w, double eps, 
    double Cp, double Cn)
{
    int l = prob_col->l; // 实例数
    int w_size = prob_col->n; // 特征数
    int j, s, newton_iter=0, iter=0; // 初始化变量
    int max_newton_iter = 100; // 最大牛顿迭代次数
    int max_iter = 1000; // 最大迭代次数
    int max_num_linesearch = 20; // 最大线搜索次数
    int active_size; // 活跃集大小
    int QP_active_size; // QP活跃集大小

    double nu = 1e-12; // nu值
    double inner_eps = 1; // 内部容差
    double sigma = 0.01; // sigma值
    double w_norm=0, w_norm_new; // w的范数
    double z, G, H; // z, G, H值
    double Gnorm1_init; // G的初始范数
    double Gmax_old = INF; // 旧的Gmax值
    double Gmax_new, Gnorm1_new; // 新的Gmax值，新的Gnorm1值
    double QP_Gmax_old = INF; // QP旧的Gmax值
    double QP_Gmax_new, QP_Gnorm1_new; // QP新的Gmax值，新的Gnorm1值
    double delta, negsum_xTd, cond; // delta值，负xTd的和，条件

    int *index = new int[w_size]; // 索引数组
    schar *y = new schar[l]; // 类别数组
    double *Hdiag = new double[w_size]; // H对角线数组
    double *Grad = new double[w_size]; // 梯度数组
    double *wpd = new double[w_size]; // wpd数组
    double *xjneg_sum = new double[w_size]; // xjneg_sum数组
    double *xTd = new double[l]; // xTd数组
    double *exp_wTx = new double[l]; // exp_wTx数组
    double *exp_wTx_new = new double[l]; // exp_wTx_new数组
    double *tau = new double[l]; // tau数组
    double *D = new double[l]; // D数组
    feature_node *x; // 特征节点

    double C[3] = {Cn,0,Cp}; // C数组

    for(j=0; j<l; j++)
    {
        if(prob_col->y[j] > 0)
            y[j] = 1;
        else
            y[j] = -1;

        // 假设初始w为0
        exp_wTx[j] = 1;
        tau[j] = C[GETI(j)]*0.5;
        D[j] = C[GETI(j)]*0.25;
    }
    for(j=0; j<w_size; j++)
    {
        // 初始化变量 w[j] 为 0
        w[j] = 0;
        // 将 w[j] 赋值给 wpd[j]
        wpd[j] = w[j];
        // 将 j 赋值给 index[j]
        index[j] = j;
        // 初始化 xjneg_sum[j] 为 0
        xjneg_sum[j] = 0;
        // 获取 prob_col->x[j] 的值赋给 x
        x = prob_col->x[j];
        // 遍历 x->index 直到遇到 -1
        while(x->index != -1)
        {
            // 获取 x->index-1 的值赋给 ind
            int ind = x->index-1;
            // 如果 y[ind] 等于 -1，则执行下面的语句
            if(y[ind] == -1)
                // 计算 xjneg_sum[j] 的值
                xjneg_sum[j] += C[GETI(ind)]*x->value;
            // x 指针向后移动
            x++;
        }
    }

    // 当 newton_iter 小于 max_newton_iter 时执行循环
    while(newton_iter < max_newton_iter)
    }

    // 输出调试信息
    info("=========================\n");
    // 输出优化完成的信息和迭代次数
    info("optimization finished, #iter = %d\n", newton_iter);
    // 如果迭代次数超过最大迭代次数，则输出警告信息
    if(newton_iter >= max_newton_iter)
        info("WARNING: reaching max number of iterations\n");

    // 计算目标值
    double v = 0;
    int nnz = 0;
    // 遍历 w 数组
    for(j=0; j<w_size; j++)
        // 如果 w[j] 不等于 0，则执行下面的语句
        if(w[j] != 0)
        {
            // 计算 v 的值
            v += fabs(w[j]);
            // 统计非零元素的个数
            nnz++;
        }
    // 遍历 y 数组
    for(j=0; j<l; j++)
        // 如果 y[j] 等于 1，则执行下面的语句
        if(y[j] == 1)
            // 计算 v 的值
            v += C[GETI(j)]*log(1+1/exp_wTx[j]);
        else
            // 计算 v 的值
            v += C[GETI(j)]*log(1+exp_wTx[j]);

    // 输出目标值
    info("Objective value = %lf\n", v);
    // 输出非零元素个数和特征数
    info("#nonzeros/#features = %d/%d\n", nnz, w_size);

    // 释放内存
    delete [] index;
    delete [] y;
    delete [] Hdiag;
    delete [] Grad;
    delete [] wpd;
    delete [] xjneg_sum;
    delete [] xTd;
    delete [] exp_wTx;
    delete [] exp_wTx_new;
    delete [] tau;
    delete [] D;
}
// 将矩阵 X 从行格式转置为列格式
static void transpose(const problem *prob, feature_node **x_space_ret, problem *prob_col)
{
    int i;
    int l = prob->l; // 获取样本数
    int n = prob->n; // 获取特征数
    int nnz = 0; // 非零元素的数量
    int *col_ptr = new int[n+1]; // 列指针数组，用于存储每一列的起始位置
    feature_node *x_space; // 特征节点数组
    prob_col->l = l; // 设置prob_col的样本数
    prob_col->n = n; // 设置prob_col的特征数
    prob_col->y = new int[l]; // 分配l个int大小的内存，用于存储样本标签
    prob_col->x = new feature_node*[n]; // 分配n个feature_node*大小的内存，用于存储特征节点数组

    for(i=0; i<l; i++)
        prob_col->y[i] = prob->y[i]; // 将prob的样本标签复制给prob_col

    for(i=0; i<n+1; i++)
        col_ptr[i] = 0; // 初始化列指针数组为0
    for(i=0; i<l; i++)
    {
        feature_node *x = prob->x[i]; // 获取第i个样本的特征节点数组
        while(x->index != -1) // 当特征节点的索引不为-1时
        {
            nnz++; // 非零元素数量加1
            col_ptr[x->index]++; // 对应列指针加1
            x++; // 指向下一个特征节点
        }
    }
    for(i=1; i<n+1; i++)
        col_ptr[i] += col_ptr[i-1] + 1; // 更新列指针数组，使其存储每一列的结束位置

    x_space = new feature_node[nnz+n]; // 分配nnz+n个feature_node大小的内存，用于存储转置后的特征节点数组
    for(i=0; i<n; i++)
        prob_col->x[i] = &x_space[col_ptr[i]]; // 设置prob_col的特征节点数组的起始位置

    for(i=0; i<l; i++)
    {
        feature_node *x = prob->x[i]; // 获取第i个样本的特征节点数组
        while(x->index != -1) // 当特征节点的索引不为-1时
        {
            int ind = x->index-1; // 获取特征节点的索引
            x_space[col_ptr[ind]].index = i+1; // 设置转置后的特征节点的索引，索引从1开始
            x_space[col_ptr[ind]].value = x->value; // 设置转置后的特征节点的值
            col_ptr[ind]++; // 对应列指针加1
            x++; // 指向下一个特征节点
        }
    }
    for(i=0; i<n; i++)
        x_space[col_ptr[i]].index = -1; // 设置转置后的特征节点的结束标志

    *x_space_ret = x_space; // 将转置后的特征节点数组赋值给x_space_ret

    delete [] col_ptr; // 释放内存
}

// label: label name, start: begin of each class, count: #data of classes, perm: indices to the original data
// perm, length l, must be allocated before calling this subroutine
static void group_classes(const problem *prob, int *nr_class_ret, int **label_ret, int **start_ret, int **count_ret, int *perm)
{
    int l = prob->l; // 获取样本数
    int max_nr_class = 16; // 最大类别数
    int nr_class = 0; // 类别数
    int *label = Malloc(int,max_nr_class); // 分配max_nr_class个int大小的内存，用于存储类别标签
    int *count = Malloc(int,max_nr_class); // 分配max_nr_class个int大小的内存，用于存储每个类别的样本数
    int *data_label = Malloc(int,l); // 分配l个int大小的内存，用于存储样本的类别标签
    int i;

    for(i=0;i<l;i++)
    {
        // 获取当前样本的标签
        int this_label = prob->y[i];
        int j;
        // 遍历所有类别
        for(j=0;j<nr_class;j++)
        {
            // 如果当前标签等于类别标签
            if(this_label == label[j])
            {
                // 对应类别计数加一
                ++count[j];
                // 退出循环
                break;
            }
        }
        // 将当前样本的类别标签索引存入数据标签数组
        data_label[i] = j;
        // 如果当前类别标签不在已有类别中
        if(j == nr_class)
        {
            // 如果已有类别数等于最大类别数
            if(nr_class == max_nr_class)
            {
                // 扩大最大类别数
                max_nr_class *= 2;
                // 重新分配内存
                label = (int *)realloc(label,max_nr_class*sizeof(int));
                count = (int *)realloc(count,max_nr_class*sizeof(int));
            }
            // 添加新的类别标签和计数
            label[nr_class] = this_label;
            count[nr_class] = 1;
            // 类别数加一
            ++nr_class;
        }
    }

    // 分配内存并初始化 start 数组
    int *start = Malloc(int,nr_class);
    start[0] = 0;
    for(i=1;i<nr_class;i++)
        start[i] = start[i-1]+count[i-1];
    // 根据数据标签重新排列样本
    for(i=0;i<l;i++)
    {
        perm[start[data_label[i]]] = i;
        ++start[data_label[i]];
    }
    // 重新初始化 start 数组
    start[0] = 0;
    for(i=1;i<nr_class;i++)
        start[i] = start[i-1]+count[i-1];

    // 返回类别数、类别标签、start 数组和计数数组
    *nr_class_ret = nr_class;
    *label_ret = label;
    *start_ret = start;
    *count_ret = count;
    // 释放数据标签内存
    free(data_label);
    // 训练单个样本，参数包括问题、参数、权重、正类别惩罚因子、负类别惩罚因子
static void train_one(const problem *prob, const parameter *param, double *w, double Cp, double Cn)
{
    // 设置精度
    double eps=param->eps;
    // 初始化正类别和负类别样本数量
    int pos = 0;
    int neg = 0;
    // 遍历问题中的样本标签，统计正类别和负类别样本数量
    for(int i=0;i<prob->l;i++)
        if(prob->y[i]==+1)
            pos++;
    neg = prob->l - pos;

    // 初始化函数对象
    function *fun_obj=NULL;
    // 根据参数中的求解器类型选择相应的函数对象
    switch(param->solver_type)
    {
        case L2R_LR:  # 如果是 L2R_LR 情况
        {
            fun_obj=new l2r_lr_fun(prob, Cp, Cn);  # 创建 L2R_LR 的目标函数对象
            TRON tron_obj(fun_obj, eps*min(pos,neg)/prob->l);  # 创建 TRON 优化器对象
            tron_obj.set_print_string(liblinear_print_string);  # 设置优化器的输出函数
            tron_obj.tron(w);  # 使用优化器进行优化
            delete fun_obj;  # 删除目标函数对象
            break;  # 结束当前情况
        }
        case L2R_L2LOSS_SVC:  # 如果是 L2R_L2LOSS_SVC 情况
        {
            fun_obj=new l2r_l2_svc_fun(prob, Cp, Cn);  # 创建 L2R_L2LOSS_SVC 的目标函数对象
            TRON tron_obj(fun_obj, eps*min(pos,neg)/prob->l);  # 创建 TRON 优化器对象
            tron_obj.set_print_string(liblinear_print_string);  # 设置优化器的输出函数
            tron_obj.tron(w);  # 使用优化器进行优化
            delete fun_obj;  # 删除目标函数对象
            break;  # 结束当前情况
        }
        case L2R_L2LOSS_SVC_DUAL:  # 如果是 L2R_L2LOSS_SVC_DUAL 情况
            solve_l2r_l1l2_svc(prob, w, eps, Cp, Cn, L2R_L2LOSS_SVC_DUAL);  # 使用特定函数解决问题
            break;  # 结束当前情况
        case L2R_L1LOSS_SVC_DUAL:  # 如果是 L2R_L1LOSS_SVC_DUAL 情况
            solve_l2r_l1l2_svc(prob, w, eps, Cp, Cn, L2R_L1LOSS_SVC_DUAL);  # 使用特定函数解决问题
            break;  # 结束当前情况
        case L1R_L2LOSS_SVC:  # 如果是 L1R_L2LOSS_SVC 情况
        {
            problem prob_col;  # 创建问题的列视图
            feature_node *x_space = NULL;  # 创建特征节点空间
            transpose(prob, &x_space ,&prob_col);  # 转置问题
            solve_l1r_l2_svc(&prob_col, w, eps*min(pos,neg)/prob->l, Cp, Cn);  # 使用特定函数解决问题
            delete [] prob_col.y;  # 删除问题的列视图标签
            delete [] prob_col.x;  # 删除问题的列视图特征
            delete [] x_space;  # 删除特征节点空间
            break;  # 结束当前情况
        }
        case L1R_LR:  # 如果是 L1R_LR 情况
        {
            problem prob_col;  # 创建问题的列视图
            feature_node *x_space = NULL;  # 创建特征节点空间
            transpose(prob, &x_space ,&prob_col);  # 转置问题
            solve_l1r_lr(&prob_col, w, eps*min(pos,neg)/prob->l, Cp, Cn);  # 使用特定函数解决问题
            delete [] prob_col.y;  # 删除问题的列视图标签
            delete [] prob_col.x;  # 删除问题的列视图特征
            delete [] x_space;  # 删除特征节点空间
            break;  # 结束当前情况
        }
        case L2R_LR_DUAL:  # 如果是 L2R_LR_DUAL 情况
            solve_l2r_lr_dual(prob, w, eps, Cp, Cn);  # 使用特定函数解决问题
            break;  # 结束当前情况
        default:  # 如果是其他情况
            fprintf(stderr, "Error: unknown solver_type\n");  # 输出错误信息
            break;  # 结束当前情况
    }
}
//
// Interface functions
//
// 训练函数，根据给定问题和参数返回模型
model* train(const problem *prob, const parameter *param)
{
    int i,j;
    int l = prob->l;  // 获取问题的样本数
    int n = prob->n;  // 获取问题的特征数
    int w_size = prob->n;  // 获取问题的特征数
    model *model_ = Malloc(model,1);  // 分配模型内存空间

    if(prob->bias>=0)
        model_->nr_feature=n-1;  // 如果存在偏置项，特征数减一
    else
        model_->nr_feature=n;  // 否则特征数不变
    model_->param = *param;  // 设置模型参数
    model_->bias = prob->bias;  // 设置模型偏置项

    int nr_class;
    int *label = NULL;
    int *start = NULL;
    int *count = NULL;
    int *perm = Malloc(int,l);  // 分配内存空间

    // group training data of the same class
    // 将相同类别的训练数据分组
    group_classes(prob,&nr_class,&label,&start,&count,perm);

    model_->nr_class=nr_class;  // 设置模型类别数
    model_->label = Malloc(int,nr_class);  // 分配内存空间
    for(i=0;i<nr_class;i++)
        model_->label[i] = label[i];  // 设置模型标签

    // calculate weighted C
    // 计算加权的 C
    double *weighted_C = Malloc(double, nr_class);  // 分配内存空间
    for(i=0;i<nr_class;i++)
        weighted_C[i] = param->C;  // 初始化加权 C
    for(i=0;i<param->nr_weight;i++)
    {
        for(j=0;j<nr_class;j++)
            if(param->weight_label[i] == label[j])
                break;
        if(j == nr_class)
            fprintf(stderr,"WARNING: class label %d specified in weight is not found\n", param->weight_label[i]);  // 输出警告信息
        else
            weighted_C[j] *= param->weight[i];  // 计算加权 C
    }

    // constructing the subproblem
    // 构建子问题
    feature_node **x = Malloc(feature_node *,l);  // 分配内存空间
    for(i=0;i<l;i++)
        x[i] = prob->x[perm[i]];  // 设置子问题特征

    int k;
    problem sub_prob;
    sub_prob.l = l;  // 设置子问题样本数
    sub_prob.n = n;  // 设置子问题特征数
    sub_prob.x = Malloc(feature_node *,sub_prob.l);  // 分配内存空间
    sub_prob.y = Malloc(int,sub_prob.l);  // 分配内存空间

    for(k=0; k<sub_prob.l; k++)
        sub_prob.x[k] = x[k];  // 设置子问题特征

    // multi-class svm by Crammer and Singer
    // 通过 Crammer 和 Singer 的多类别支持向量机
    if(param->solver_type == MCSVM_CS)
    {
        model_->w=Malloc(double, n*nr_class);  // 分配内存空间
        for(i=0;i<nr_class;i++)
            for(j=start[i];j<start[i]+count[i];j++)
                sub_prob.y[j] = i;  // 设置子问题标签
        Solver_MCSVM_CS Solver(&sub_prob, nr_class, weighted_C, param->eps);  // 初始化多类别支持向量机求解器
        Solver.Solve(model_->w);  // 求解模型权重
    }
    else
    {
        // 如果类别数为2
        if(nr_class == 2)
        {
            // 为模型的权重分配内存
            model_->w=Malloc(double, w_size);

            // 计算第一个类别的结束索引
            int e0 = start[0]+count[0];
            k=0;
            // 为第一个类别的样本标签赋值为+1
            for(; k<e0; k++)
                sub_prob.y[k] = +1;
            // 为第二个类别的样本标签赋值为-1
            for(; k<sub_prob.l; k++)
                sub_prob.y[k] = -1;

            // 训练二分类模型
            train_one(&sub_prob, param, &model_->w[0], weighted_C[0], weighted_C[1]);
        }
        // 如果类别数大于2
        else
        {
            // 为模型的权重分配内存
            model_->w=Malloc(double, w_size*nr_class);
            // 为临时权重向量分配内存
            double *w=Malloc(double, w_size);
            for(i=0;i<nr_class;i++)
            {
                // 获取当前类别的起始索引和结束索引
                int si = start[i];
                int ei = si+count[i];

                k=0;
                // 为当前类别之前的样本标签赋值为-1
                for(; k<si; k++)
                    sub_prob.y[k] = -1;
                // 为当前类别的样本标签赋值为+1
                for(; k<ei; k++)
                    sub_prob.y[k] = +1;
                // 为当前类别之后的样本标签赋值为-1
                for(; k<sub_prob.l; k++)
                    sub_prob.y[k] = -1;

                // 训练多分类模型
                train_one(&sub_prob, param, w, weighted_C[i], param->C);

                // 将临时权重向量复制到模型的权重中
                for(int j=0;j<w_size;j++)
                    model_->w[j*nr_class+i] = w[j];
            }
            // 释放临时权重向量的内存
            free(w);
        }

    }

    // 释放内存
    free(x);
    free(label);
    free(start);
    free(count);
    free(perm);
    free(sub_prob.x);
    free(sub_prob.y);
    free(weighted_C);
    // 返回训练好的模型
    return model_;
}

This is the end of the function "cross_validation".


void cross_validation(const problem *prob, const parameter *param, int nr_fold, int *target)
{

This is the beginning of the function "cross_validation" which takes in a problem, a parameter, the number of folds, and a target array as input.


    int i;
    int *fold_start = Malloc(int,nr_fold+1);
    int l = prob->l;
    int *perm = Malloc(int,l);

Declare variables "i", "fold_start" as an array of integers, "l" as the number of instances in the problem, and "perm" as an array of integers.


    for(i=0;i<l;i++) perm[i]=i;
    for(i=0;i<l;i++)
    {
        int j = i+rand()%(l-i);
        swap(perm[i],perm[j]);
    }

Initialize the "perm" array with indices from 0 to l-1 and shuffle the indices randomly.


    for(i=0;i<=nr_fold;i++)
        fold_start[i]=i*l/nr_fold;

Calculate the starting index for each fold and store it in the "fold_start" array.


    for(i=0;i<nr_fold;i++)
    {
        int begin = fold_start[i];
        int end = fold_start[i+1];
        int j,k;
        struct problem subprob;

        subprob.bias = prob->bias;
        subprob.n = prob->n;
        subprob.l = l-(end-begin);
        subprob.x = Malloc(struct feature_node*,subprob.l);
        subprob.y = Malloc(int,subprob.l);

Iterate through each fold and create a subproblem by setting its bias, number of features, number of instances, and allocating memory for its feature nodes and labels.


        k=0;
        for(j=0;j<begin;j++)
        {
            subprob.x[k] = prob->x[perm[j]];
            subprob.y[k] = prob->y[perm[j]];
            ++k;
        }
        for(j=end;j<l;j++)
        {
            subprob.x[k] = prob->x[perm[j]];
            subprob.y[k] = prob->y[perm[j]];
            ++k;
        }

Populate the subproblem with the appropriate feature nodes and labels based on the current fold.


        struct model *submodel = train(&subprob,param);
        for(j=begin;j<end;j++)
            target[perm[j]] = predict(submodel,prob->x[perm[j]]);
        free_and_destroy_model(&submodel);
        free(subprob.x);
        free(subprob.y);
    }

Train a model using the subproblem, make predictions for the instances in the current fold, free the memory allocated for the submodel, feature nodes, and labels.


    free(fold_start);
    free(perm);
}

Free the memory allocated for the "fold_start" and "perm" arrays.


int predict_values(const struct model *model_, const struct feature_node *x, double *dec_values)
{
    int idx;
    int n;
    if(model_->bias>=0)
        n=model_->nr_feature+1;
    else
        n=model_->nr_feature;
    double *w=model_->w;
    int nr_class=model_->nr_class;
    int i;
    int nr_w;
    if(nr_class==2 && model_->param.solver_type != MCSVM_CS)
        nr_w = 1;
    else
        nr_w = nr_class;

This is the beginning of the function "predict_values" which takes in a model, a feature node, and an array for storing decision values.


    const feature_node *lx=x;
    for(i=0;i<nr_w;i++)
        dec_values[i] = 0;
    for(; (idx=lx->index)!=-1; lx++)

Initialize the decision values array and iterate through the feature nodes of the input instance.
    {
        // 如果测试数据的维度可能超过训练数据的维度
        if(idx<=n)
            // 对于每个权重向量，将测试数据与对应的特征向量相乘并加到决策值上
            for(i=0;i<nr_w;i++)
                dec_values[i] += w[(idx-1)*nr_w+i]*lx->value;
    }

    // 如果类别数为2
    if(nr_class==2)
        // 返回第一个类别或第二个类别，取决于决策值的正负
        return (dec_values[0]>0)?model_->label[0]:model_->label[1];
    else
    {
        // 初始化决策值最大的类别索引为0
        int dec_max_idx = 0;
        // 遍历每个类别的决策值
        for(i=1;i<nr_class;i++)
        {
            // 如果当前类别的决策值大于最大决策值的类别的决策值
            if(dec_values[i] > dec_values[dec_max_idx])
                // 更新最大决策值的类别索引
                dec_max_idx = i;
        }
        // 返回具有最大决策值的类别
        return model_->label[dec_max_idx];
    }
}

int predict(const model *model_, const feature_node *x)
{
    // 为决策值分配内存
    double *dec_values = Malloc(double, model_->nr_class);
    // 预测标签
    int label=predict_values(model_, x, dec_values);
    // 释放决策值内存
    free(dec_values);
    // 返回预测标签
    return label;
}

int predict_probability(const struct model *model_, const struct feature_node *x, double* prob_estimates)
{
    // 检查模型是否支持概率估计
    if(check_probability_model(model_))
    {
        int i;
        int nr_class=model_->nr_class;
        int nr_w;
        // 根据类别数量确定权重数量
        if(nr_class==2)
            nr_w = 1;
        else
            nr_w = nr_class;

        // 预测标签并计算概率估计
        int label=predict_values(model_, x, prob_estimates);
        for(i=0;i<nr_w;i++)
            prob_estimates[i]=1/(1+exp(-prob_estimates[i]));

        // 对于二分类，计算第二类别的概率估计
        if(nr_class==2) // for binary classification
            prob_estimates[1]=1.-prob_estimates[0];
        else
        {
            double sum=0;
            // 计算概率估计的总和
            for(i=0; i<nr_class; i++)
                sum+=prob_estimates[i];

            // 归一化概率估计
            for(i=0; i<nr_class; i++)
                prob_estimates[i]=prob_estimates[i]/sum;
        }

        // 返回预测标签
        return label;        
    }
    else
        return 0;
}

static const char *solver_type_table[]=
{
    "L2R_LR", "L2R_L2LOSS_SVC_DUAL", "L2R_L2LOSS_SVC", "L2R_L1LOSS_SVC_DUAL", "MCSVM_CS",
    "L1R_L2LOSS_SVC", "L1R_LR", "L2R_LR_DUAL", NULL
};

int save_model(const char *model_file_name, const struct model *model_)
{
    int i;
    int nr_feature=model_->nr_feature;
    int n;
    const parameter& param = model_->param;

    // 根据偏置值确定特征数量
    if(model_->bias>=0)
        n=nr_feature+1;
    else
        n=nr_feature;
    int w_size = n;
    // 打开模型文件
    FILE *fp = fopen(model_file_name,"w");
    if(fp==NULL) return -1;

    int nr_w;
    // 根据类别数量和解法类型确定权重数量
    if(model_->nr_class==2 && model_->param.solver_type != MCSVM_CS)
        nr_w=1;
    else
        nr_w=model_->nr_class;

    // 写入解法类型和类别数量到模型文件
    fprintf(fp, "solver_type %s\n", solver_type_table[param.solver_type]);
    fprintf(fp, "nr_class %d\n", model_->nr_class);
    fprintf(fp, "label");
    # 遍历模型的类别数量，将每个类别的标签写入文件
    for(i=0; i<model_->nr_class; i++)
        fprintf(fp, " %d", model_->label[i]);
    # 写入换行符
    fprintf(fp, "\n");

    # 写入特征数量信息
    fprintf(fp, "nr_feature %d\n", nr_feature);

    # 写入偏置值
    fprintf(fp, "bias %.16g\n", model_->bias);

    # 写入权重信息
    fprintf(fp, "w\n");
    # 遍历权重数组，将每个权重值写入文件
    for(i=0; i<w_size; i++)
    {
        int j;
        for(j=0; j<nr_w; j++)
            fprintf(fp, "%.16g ", model_->w[i*nr_w+j]);
        # 写入换行符
        fprintf(fp, "\n");
    }

    # 检查文件写入错误或关闭文件失败，返回-1；否则返回0
    if (ferror(fp) != 0 || fclose(fp) != 0) return -1;
    else return 0;
# 加载模型的函数，根据给定的模型文件名
struct model *load_model(const char *model_file_name)
{
    # 以只读方式打开模型文件
    FILE *fp = fopen(model_file_name,"r");
    # 如果文件指针为空，返回空指针
    if(fp==NULL) return NULL;

    int i;
    int nr_feature;
    int n;
    int nr_class;
    double bias;
    # 分配内存给模型结构体指针
    model *model_ = Malloc(model,1);
    # 获取模型参数的引用
    parameter& param = model_->param;

    # 初始化模型的标签为NULL
    model_->label = NULL;

    char cmd[81];
    # 读取模型文件中的内容
    while(1)
    {
        # 读取模型文件中的命令
        fscanf(fp,"%80s",cmd);
        # 如果命令是solver_type
        if(strcmp(cmd,"solver_type")==0)
        {
            # 读取solver_type的值
            fscanf(fp,"%80s",cmd);
            int i;
            # 遍历solver_type_table数组
            for(i=0;solver_type_table[i];i++)
            {
                # 如果找到对应的solver_type值，设置param.solver_type为i
                if(strcmp(solver_type_table[i],cmd)==0)
                {
                    param.solver_type=i;
                    break;
                }
            }
            # 如果solver_type_table[i]为空，打印错误信息，释放内存并返回空指针
            if(solver_type_table[i] == NULL)
            {
                fprintf(stderr,"unknown solver type.\n");
                free(model_->label);
                free(model_);
                return NULL;
            }
        }
        # 如果命令是nr_class
        else if(strcmp(cmd,"nr_class")==0)
        {
            # 读取nr_class的值
            fscanf(fp,"%d",&nr_class);
            model_->nr_class=nr_class;
        }
        # 如果命令是nr_feature
        else if(strcmp(cmd,"nr_feature")==0)
        {
            # 读取nr_feature的值
            fscanf(fp,"%d",&nr_feature);
            model_->nr_feature=nr_feature;
        }
        # 如果命令是bias
        else if(strcmp(cmd,"bias")==0)
        {
            # 读取bias的值
            fscanf(fp,"%lf",&bias);
            model_->bias=bias;
        }
        # 如果命令是w，跳出循环
        else if(strcmp(cmd,"w")==0)
        {
            break;
        }
        # 如果命令是label
        else if(strcmp(cmd,"label")==0)
        {
            # 读取nr_class个标签值
            int nr_class = model_->nr_class;
            model_->label = Malloc(int,nr_class);
            for(int i=0;i<nr_class;i++)
                fscanf(fp,"%d",&model_->label[i]);
        }
        # 如果命令是未知的，打印错误信息，释放内存并返回空指针
        else
        {
            fprintf(stderr,"unknown text in model file: [%s]\n",cmd);
            free(model_);
            return NULL;
        }
    }

    # 设置nr_feature为模型的nr_feature
    nr_feature=model_->nr_feature;
    # 如果模型的bias大于等于0，n为nr_feature加1，否则n为nr_feature
    if(model_->bias>=0)
        n=nr_feature+1;
    else
        n=nr_feature;
    # 设置w_size为n
    int w_size = n;
    # 定义整型变量 nr_w
    int nr_w;
    # 如果类别数为2且解决器类型不是MCSVM_CS，则 nr_w 等于1，否则 nr_w 等于类别数
    if(nr_class==2 && param.solver_type != MCSVM_CS)
        nr_w = 1;
    else
        nr_w = nr_class;

    # 为模型的权重分配内存空间
    model_->w=Malloc(double, w_size*nr_w);
    # 遍历权重数组
    for(i=0; i<w_size; i++)
    {
        # 定义整型变量 j
        int j;
        # 遍历 nr_w，读取文件中的权重数据并存入模型的权重数组中
        for(j=0; j<nr_w; j++)
            fscanf(fp, "%lf ", &model_->w[i*nr_w+j]);
        # 读取换行符
        fscanf(fp, "\n");
    }
    # 如果文件读取出错或关闭文件失败，则返回空指针
    if (ferror(fp) != 0 || fclose(fp) != 0) return NULL;

    # 返回模型
    return model_;
# 获取模型中特征的数量
int get_nr_feature(const model *model_)
{
    return model_->nr_feature;
}

# 获取模型中类别的数量
int get_nr_class(const model *model_)
{
    return model_->nr_class;
}

# 获取模型中的标签
void get_labels(const model *model_, int* label)
{
    if (model_->label != NULL)
        for(int i=0;i<model_->nr_class;i++)
            label[i] = model_->label[i];
}

# 释放模型内容
void free_model_content(struct model *model_ptr)
{
    if(model_ptr->w != NULL)
        free(model_ptr->w);
    if(model_ptr->label != NULL)
        free(model_ptr->label);
}

# 释放并销毁模型
void free_and_destroy_model(struct model **model_ptr_ptr)
{
    struct model *model_ptr = *model_ptr_ptr;
    if(model_ptr != NULL)
    {
        free_model_content(model_ptr);
        free(model_ptr);
    }
}

# 销毁参数
void destroy_param(parameter* param)
{
    if(param->weight_label != NULL)
        free(param->weight_label);
    if(param->weight != NULL)
        free(param->weight);
}

# 检查参数是否合法
const char *check_parameter(const problem *prob, const parameter *param)
{
    if(param->eps <= 0)
        return "eps <= 0";

    if(param->C <= 0)
        return "C <= 0";

    if(param->solver_type != L2R_LR
        && param->solver_type != L2R_L2LOSS_SVC_DUAL
        && param->solver_type != L2R_L2LOSS_SVC
        && param->solver_type != L2R_L1LOSS_SVC_DUAL
        && param->solver_type != MCSVM_CS
        && param->solver_type != L1R_L2LOSS_SVC
        && param->solver_type != L1R_LR
        && param->solver_type != L2R_LR_DUAL)
        return "unknown solver type";

    return NULL;
}

# 检查模型是否是概率模型
int check_probability_model(const struct model *model_)
{
    return (model_->param.solver_type==L2R_LR ||
            model_->param.solver_type==L2R_LR_DUAL ||
            model_->param.solver_type==L1R_LR);
}

# 设置打印字符串的函数
void set_print_string_function(void (*print_func)(const char*))
{
    if (print_func == NULL) 
        liblinear_print_string = &print_string_stdout;
    else
        liblinear_print_string = print_func;
}
```