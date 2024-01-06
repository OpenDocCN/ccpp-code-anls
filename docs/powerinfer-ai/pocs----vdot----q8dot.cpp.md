# `PowerInfer\pocs\vdot\q8dot.cpp`

```
// 包含所需的头文件
#include <cstdio>           // C 标准输入输出
#include <type_traits>      // 提供了一些模板类，用于在编译期间进行类型特性判断
#include <vector>           // 提供了动态数组的实现
#include <random>           // 提供了随机数生成器
#include <chrono>           // 提供了时间相关的功能
#include <cstdlib>          // 包含了一些通用的函数和宏
#include <cmath>            // 提供了数学函数
#include <cassert>          // 提供了断言宏
#include <cstring>          // 提供了字符串操作函数
#include <array>            // 提供了固定大小数组的实现
#include <type_traits>      // 提供了一些类型特性判断的模板类

#include <ggml.h>           // 包含了 ggml 库的头文件

constexpr int kVecSize = 1 << 16;   // 定义了一个常量 kVecSize，值为 2^16

// 从 ggml.c 中复制粘贴过来的代码
#define QK4_0 32    // 定义了一个宏 QK4_0，值为 32
typedef struct {
    float   d;      // 定义了一个结构体，包含一个 float 类型的成员变量 d
// 定义一个名为 qs 的长度为 QK4_0 / 2 的 uint8_t 数组，用于存储 nibbles / quants
} block_q4_0;
// 使用 static_assert 断言检查 block_q4_0 结构体的大小是否等于 sizeof(float) + QK4_0 / 2，如果不等则输出错误信息
static_assert(sizeof(block_q4_0) == sizeof(float) + QK4_0 / 2, "wrong q4_0 block size/padding");

// 定义一个名为 qs 的长度为 QK4_1 / 2 的 uint8_t 数组，用于存储 nibbles / quants
} block_q4_1;
// 使用 static_assert 断言检查 block_q4_1 结构体的大小是否等于 sizeof(float) * 2 + QK4_1 / 2，如果不等则输出错误信息
static_assert(sizeof(block_q4_1) == sizeof(float) * 2 + QK4_1 / 2, "wrong q4_1 block size/padding");

// 从 ggml.c 中复制粘贴的代码
// 定义一个名为 qs 的长度为 QK8_0 的 int8_t 数组，用于存储 quants
} block_q8_0;
// 使用 static_assert 断言检查 block_q8_0 结构体的大小是否等于 2*sizeof(float) + QK8_0，如果不等则输出错误信息
static_assert(sizeof(block_q8_0) == 2*sizeof(float) + QK8_0, "wrong q8_0 block size/padding");
// 确保 QK4_1 和 QK8_0 相等，否则触发静态断言
static_assert(QK4_1 == QK8_0, "QK4_1 and QK8_0 must be the same");
// 确保 QK4_0 和 QK8_0 相等，否则触发静态断言
static_assert(QK4_0 == QK8_0, "QK4_0 and QK8_0 must be the same");

// 填充 Q4blocks 的模板函数，使用随机数生成器填充每个块
template <typename T>
static void fillQ4blocks(std::vector<T>& blocks, std::mt19937& rndm) {
    // 遍历每个块
    for (auto& b : blocks) {
        // 设置块的 d 值为 1
        b.d = 1;
        // 遍历块的一半长度，使用随机数填充块的 qs 数组
        for (int i=0; i<QK4_1/2; ++i) {
            uint8_t v1 = rndm() >> 28;
            uint8_t v2 = rndm() >> 28;
            b.qs[i] = v1 | (v2 << 4);
        }
    }
}

// 填充 Q80blocks 的函数，使用随机数生成器填充每个块
static void fillQ80blocks(std::vector<block_q8_0>& blocks, std::mt19937& rndm) {
    // 遍历每个块
    for (auto& b : blocks) {
        // 设置块的 d 值为 1
        b.d = 1;
        // 初始化 sum 变量
        int sum = 0;
// 对数组 b.qs 进行循环赋值，每次赋值都是一个随机数右移24位再减去128
// 计算数组 b.qs 的和
// 计算 b.s 的值
static float simpleDot(const block_q4_0& x, const block_q8_0& y) {
    // 初始化变量 s1 为 0
    // 循环遍历数组 x.qs，每次遍历都对两个元素进行处理
    // 将 x.qs 中的元素按位与运算和右移操作，然后与 y.qs 中的元素相乘并累加到 s1 中
    // 返回计算结果
// 计算简单的点乘结果
static float simpleDot(const block_q4_1& x, const block_q8_0& y) {
    int s1 = 0; // 初始化变量 s1
    for (int i=0; i<QK4_1/2; i+=2) { // 循环遍历数组
        // 提取 x.qs[i+0] 中的低 4 位和高 4 位
        int v1 = x.qs[i+0] & 0xf;
        int v2 = x.qs[i+0] >> 4;
        // 提取 x.qs[i+1] 中的低 4 位和高 4 位
        int v3 = x.qs[i+1] & 0xf;
        int v4 = x.qs[i+1] >> 4;
        int j = 2*i;
        // 计算点乘结果并累加到 s1
        s1 += v1*y.qs[j] + v2*y.qs[j+1] + v3*y.qs[j+2] + v4*y.qs[j+3];
    }
    // 返回点乘结果
    return y.d * x.d * s1 + y.s * x.m;
}

// 定义结构体 Stat
struct Stat {
    double sum = 0, sumt = 0, sumt2 = 0, maxt = 0; // 初始化结构体成员变量
    // 初始化循环次数为0
    int nloop = 0;
    // 添加结果到总和中
    void addResult(double s, double t) {
        sum += s; // 将s加到sum中
        sumt += t; // 将t加到sumt中
        sumt2 += t*t; // 将t的平方加到sumt2中
        maxt = std::max(maxt, t); // 更新maxt为max(maxt, t)
        ++nloop; // 循环次数加1
    }
    // 报告结果
    void reportResult(const char* title) const {
        // 如果循环次数小于1，打印无结果
        if (nloop < 1) {
            printf("%s(%s): no result\n",__func__,title);
            return;
        }
        // 打印标题
        printf("============ %s\n",title);
        // 打印平均值
        printf("<dot> = %g\n",sum/nloop);
        // 计算平均时间和标准差
        auto t = sumt/nloop, dt = sumt2/nloop - t*t;
        if (dt > 0) dt = sqrt(dt); // 如果标准差大于0，计算平方根
        // 打印平均时间和标准差
        printf("<time> = %g +/- %g us. Max. time = %g us.\n",t,dt,maxt);
    }
};
# 主函数，接受命令行参数
int main(int argc, char** argv) {

    # 根据命令行参数确定循环次数，如果没有参数，默认为10
    int nloop = argc > 1 ? atoi(argv[1]) : 10;
    # 根据命令行参数确定类型，如果没有参数，默认为1
    int type  = argc > 2 ? atoi(argv[2]) : 1;

    # 创建随机数生成器对象
    std::mt19937 rndm(1234);

    # 创建不同类型的向量对象
    std::vector<block_q4_1> x41;
    std::vector<block_q4_0> x40;
    std::vector<block_q8_0> y(kVecSize);
    # 根据类型选择调整向量大小
    if (type == 0) x40.resize(kVecSize);
    else {
        x41.resize(kVecSize);
        # 对 x41 中的每个元素进行初始化
        for (auto& b : x41) b.m = 1;
    }

    # 根据类型选择 ggml_type
    auto ggml_type = type == 0 ? GGML_TYPE_Q4_0 : GGML_TYPE_Q4_1;

    # 根据 ggml_type 获取相应的函数
    auto funcs = ggml_internal_get_type_traits(ggml_type);
    // 定义两个对象 simple 和 ggml，类型为 Stat
    Stat simple, ggml;

    // 循环 nloop 次
    for (int iloop=0; iloop<nloop; ++iloop) {

        // 如果 type 等于 0，则调用 fillQ4blocks 函数填充 x40，否则填充 x41
        if (type == 0) fillQ4blocks(x40, rndm);
        else fillQ4blocks(x41, rndm);
        
        // 调用 fillQ80blocks 函数填充 y
        fillQ80blocks(y, rndm);

        // 记录当前时间
        auto t1 = std::chrono::high_resolution_clock::now();
        double s = 0;
        // 如果 type 等于 0，则循环计算 simpleDot(x40[i], y[i]) 的和，否则计算 simpleDot(x41[i], y[i]) 的和
        if (type == 0) for (int i=0; i<kVecSize; ++i) s += simpleDot(x40[i], y[i]);
        else for (int i=0; i<kVecSize; ++i) s += simpleDot(x41[i], y[i]);
        // 记录当前时间
        auto t2 = std::chrono::high_resolution_clock::now();
        // 计算时间间隔
        auto t = 1e-3*std::chrono::duration_cast<std::chrono::nanoseconds>(t2-t1).count();
        // 如果循环次数大于 3，则将结果 s 和时间 t 添加到 simple 对象中
        if (iloop > 3) simple.addResult(s, t);

        // 记录当前时间
        t1 = std::chrono::high_resolution_clock::now();
        float fs;
        // 如果 type 等于 0，则调用 funcs.vec_dot 函数计算 x40 和 y 的点积，否则计算 x41 和 y 的点积
        if (type == 0) funcs.vec_dot(kVecSize * QK4_1, &fs, x40.data(), y.data());
        else funcs.vec_dot(kVecSize * QK4_1, &fs, x41.data(), y.data());
        // 记录结束时间
        t2 = std::chrono::high_resolution_clock::now();
        // 计算时间间隔并转换为秒
        t = 1e-3*std::chrono::duration_cast<std::chrono::nanoseconds>(t2-t1).count();
        // 如果循环次数大于3，则将结果添加到ggml对象中
        if (iloop > 3) ggml.addResult(fs);

    }

    // 报告时间（以及点积的平均值，以防编译器在发现结果未被使用后优化掉函数调用）
    simple.reportResult("Simple");
    ggml.reportResult("ggml");
    // 返回0表示程序正常结束
    return 0;
}
```