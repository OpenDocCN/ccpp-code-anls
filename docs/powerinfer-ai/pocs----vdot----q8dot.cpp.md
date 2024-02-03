# `PowerInfer\pocs\vdot\q8dot.cpp`

```cpp
#include <cstdio>
#include <type_traits>
#include <vector>
#include <random>
#include <chrono>
#include <cstdlib>
#include <cmath>
#include <cassert>
#include <cstring>
#include <array>
#include <type_traits>

#include <ggml.h>

constexpr int kVecSize = 1 << 16;

// 从 ggml.c 中复制粘贴过来的代码
#define QK4_0 32
typedef struct {
    float   d;          // delta
    uint8_t qs[QK4_0 / 2];  // nibbles / quants
} block_q4_0;
static_assert(sizeof(block_q4_0) == sizeof(float) + QK4_0 / 2, "wrong q4_0 block size/padding");

#define QK4_1 32
typedef struct {
    float   d;          // delta
    float   m;          // min
    uint8_t qs[QK4_1 / 2];  // nibbles / quants
} block_q4_1;
static_assert(sizeof(block_q4_1) == sizeof(float) * 2 + QK4_1 / 2, "wrong q4_1 block size/padding");

// 从 ggml.c 中复制粘贴过来的代码
#define QK8_0 32
typedef struct {
    float   d;          // delta
    float   s;          // d * sum(qs[i])
    int8_t  qs[QK8_0];  // quants
} block_q8_0;
static_assert(sizeof(block_q8_0) == 2*sizeof(float) + QK8_0, "wrong q8_0 block size/padding");

static_assert(QK4_1 == QK8_0, "QK4_1 and QK8_0 must be the same");
static_assert(QK4_0 == QK8_0, "QK4_0 and QK8_0 must be the same");

template <typename T>
static void fillQ4blocks(std::vector<T>& blocks, std::mt19937& rndm) {
    for (auto& b : blocks) {
        b.d = 1;  // 设置 delta 为 1
        for (int i=0; i<QK4_1/2; ++i) {
            uint8_t v1 = rndm() >> 28;  // 生成随机数并右移 28 位
            uint8_t v2 = rndm() >> 28;  // 生成随机数并右移 28 位
            b.qs[i] = v1 | (v2 << 4);  // 将两个随机数合并成一个字节并赋值给 qs 数组
        }
    }
}

static void fillQ80blocks(std::vector<block_q8_0>& blocks, std::mt19937& rndm) {
    for (auto& b : blocks) {
        b.d = 1;  // 设置 delta 为 1
        int sum = 0;
        for (int i=0; i<QK8_0; ++i) {
            b.qs[i] = (rndm() >> 24) - 128;  // 生成随机数并右移 24 位，然后减去 128 并赋值给 qs 数组
            sum += b.qs[i];  // 计算 qs 数组的和
        }
        b.s = b.d * sum;  // 计算 s 值
    }
}

static float simpleDot(const block_q4_0& x, const block_q8_0& y) {
    int s1 = 0; //, s2 = 0;  // 初始化 s1 为 0
    # 使用循环遍历数组，每次增加2
    for (int i=0; i<QK4_1/2; i+=2) {
        # 从数组中取出第i个元素的低4位
        int v1 = x.qs[i+0] & 0xf;
        # 从数组中取出第i个元素的高4位
        int v2 = x.qs[i+0] >> 4;
        # 从数组中取出第i+1个元素的低4位
        int v3 = x.qs[i+1] & 0xf;
        # 从数组中取出第i+1个元素的高4位
        int v4 = x.qs[i+1] >> 4;
        # 计算j的值
        int j = 2*i;
        # 计算s1的值
        s1 += v1*y.qs[j] + v2*y.qs[j+1] + v3*y.qs[j+2] + v4*y.qs[j+3];
        //s2 += y.qs[j] + y.qs[j+1] + y.qs[j+2] + y.qs[j+3];
    }
    # 返回计算结果
    return y.d * x.d * s1 - 8 * x.d * y.s;
    //return y.d * x.d * (s1 - 8 * s2);
// 计算两个结构体的简单点积
static float simpleDot(const block_q4_1& x, const block_q8_0& y) {
    int s1 = 0; // 初始化第一个求和变量
    for (int i=0; i<QK4_1/2; i+=2) { // 循环遍历结构体中的数据
        int v1 = x.qs[i+0] & 0xf; // 获取结构体中的数据并进行位运算
        int v2 = x.qs[i+0] >> 4; // 获取结构体中的数据并进行位运算
        int v3 = x.qs[i+1] & 0xf; // 获取结构体中的数据并进行位运算
        int v4 = x.qs[i+1] >> 4; // 获取结构体中的数据并进行位运算
        int j = 2*i; // 计算索引值
        s1 += v1*y.qs[j] + v2*y.qs[j+1] + v3*y.qs[j+2] + v4*y.qs[j+3]; // 计算点积
    }
    return y.d * x.d * s1 + y.s * x.m; // 返回点积的加权和
}

// 定义一个结构体用于存储统计数据
struct Stat {
    double sum = 0, sumt = 0, sumt2 = 0, maxt = 0; // 初始化统计数据
    int nloop = 0; // 初始化循环次数
    void addResult(double s, double t) { // 添加统计结果
        sum += s; // 累加s
        sumt += t; sumt2 += t*t; maxt = std::max(maxt, t); // 累加t，t的平方，更新最大值
        ++nloop; // 循环次数加1
    }
    void reportResult(const char* title) const { // 报告统计结果
        if (nloop < 1) { // 如果循环次数小于1
            printf("%s(%s): no result\n",__func__,title); // 输出无结果
            return;
        }
        printf("============ %s\n",title); // 输出标题
        printf("<dot> = %g\n",sum/nloop); // 输出点积的平均值
        auto t = sumt/nloop, dt = sumt2/nloop - t*t; // 计算时间的平均值和方差
        if (dt > 0) dt = sqrt(dt); // 如果方差大于0，则计算标准差
        printf("<time> = %g +/- %g us. Max. time = %g us.\n",t,dt,maxt); // 输出时间的平均值、标准差和最大值
    }
}

// 主函数
int main(int argc, char** argv) {
    int nloop = argc > 1 ? atoi(argv[1]) : 10; // 获取循环次数
    int type  = argc > 2 ? atoi(argv[2]) : 1; // 获取类型

    std::mt19937 rndm(1234); // 初始化随机数生成器

    std::vector<block_q4_1> x41; // 定义存储结构体的向量
    std::vector<block_q4_0> x40; // 定义存储结构体的向量
    std::vector<block_q8_0> y(kVecSize); // 定义存储结构体的向量
    if (type == 0) x40.resize(kVecSize); // 根据类型调整向量大小
    else {
        x41.resize(kVecSize); // 根据类型调整向量大小
        for (auto& b : x41) b.m = 1; // 初始化结构体中的数据
    }

    auto ggml_type = type == 0 ? GGML_TYPE_Q4_0 : GGML_TYPE_Q4_1; // 根据类型选择不同的类型

    auto funcs = ggml_internal_get_type_traits(ggml_type); // 获取类型特性

    Stat simple, ggml; // 定义两个统计对象
    // 循环执行 nloop 次
    for (int iloop=0; iloop<nloop; ++iloop) {

        // 根据 type 的值选择填充 x40 或 x41
        if (type == 0) fillQ4blocks(x40, rndm);
        else fillQ4blocks(x41, rndm);
        // 填充 y
        fillQ80blocks(y, rndm);

        // 记录当前时间
        auto t1 = std::chrono::high_resolution_clock::now();
        double s = 0;
        // 根据 type 的值选择计算简单点积
        if (type == 0) for (int i=0; i<kVecSize; ++i) s += simpleDot(x40[i], y[i]);
        else for (int i=0; i<kVecSize; ++i) s += simpleDot(x41[i], y[i]);
        // 记录结束时间
        auto t2 = std::chrono::high_resolution_clock::now();
        // 计算时间间隔
        auto t = 1e-3*std::chrono::duration_cast<std::chrono::nanoseconds>(t2-t1).count();
        // 如果循环次数大于3，则将结果添加到 simple 对象中
        if (iloop > 3) simple.addResult(s, t);

        // 重新记录当前时间
        t1 = std::chrono::high_resolution_clock::now();
        float fs;
        // 根据 type 的值选择调用 vec_dot 函数
        if (type == 0) funcs.vec_dot(kVecSize * QK4_1, &fs, x40.data(), y.data());
        else funcs.vec_dot(kVecSize * QK4_1, &fs, x41.data(), y.data());
        // 重新记录结束时间
        t2 = std::chrono::high_resolution_clock::now();
        // 重新计算时间间隔
        t = 1e-3*std::chrono::duration_cast<std::chrono::nanoseconds>(t2-t1).count();
        // 如果循环次数大于3，则将结果添加到 ggml 对象中
        if (iloop > 3) ggml.addResult(fs, t);

    }

    // 报告时间和点积的平均值，以防止编译器在发现结果未被使用后进行优化
    simple.reportResult("Simple");
    ggml.reportResult("ggml");
    // 返回 0
    return 0;
# 闭合前面的函数定义
```