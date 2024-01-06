# `PowerInfer\pocs\vdot\vdot.cpp`

```
#include <cstdio> // 包含标准输入输出头文件
#include <vector> // 包含向量头文件
#include <random> // 包含随机数生成器头文件
#include <chrono> // 包含时间头文件
#include <cstdlib> // 包含标准库头文件
#include <cmath> // 包含数学函数头文件
#include <cassert> // 包含断言头文件
#include <cstring> // 包含字符串操作头文件
#include <array> // 包含数组头文件

#include <ggml.h> // 包含 ggml 库头文件

#if defined(_MSC_VER)
#pragma warning(disable: 4244 4267) // 禁止特定警告
#endif

constexpr int kVecSize = 1 << 18; // 定义常量 kVecSize 为 2 的 18 次方

static float drawFromGaussianPdf(std::mt19937& rndm) { // 定义静态函数 drawFromGaussianPdf，参数为随机数生成器的引用
    constexpr double kScale = 1./(1. + std::mt19937::max()); // 定义常量 kScale 为 1/(1 + 随机数生成器的最大值)
// 计算常量 kTwoPiTimesScale，表示 2π 乘以比例尺
constexpr double kTwoPiTimesScale = 6.28318530717958647692*kScale;
// 静态变量，用于存储上一次的 X 值
static float lastX;
// 静态变量，表示是否已经有 X 值
static bool haveX = false;
// 如果已经有 X 值，则返回上一次的 X 值
if (haveX) { haveX = false; return lastX; }
// 计算服从高斯分布的随机数
auto r = sqrt(-2*log(1 - kScale*rndm()));
// 计算随机角度
auto phi = kTwoPiTimesScale * rndm();
// 计算 X 值
lastX = r*sin(phi);
// 标记已经有 X 值
haveX = true;
// 返回服从高斯分布的随机数
return r*cos(phi);
}

// 填充给定的浮点数向量 values，使其服从高斯分布
static void fillRandomGaussianFloats(std::vector<float>& values, std::mt19937& rndm, float mean = 0) {
    // 遍历向量中的每个元素，使其服从高斯分布
    for (auto& v : values) v = mean + drawFromGaussianPdf(rndm);
}

// 从 ggml.c 中复制粘贴的代码
// 定义常量 QK4_0，表示 32
#define QK4_0 32
// 定义结构体，包含 delta 和 nibbles/quants 数组
typedef struct {
    float   d;          // delta
    uint8_t qs[QK4_0 / 2];  // nibbles / quants
// 定义一个名为 block_q4_0 的结构体
} block_q4_0;
// 使用 static_assert 断言结构体 block_q4_0 的大小等于 float 类型的大小加上 QK4_0 的一半，如果不相等则输出错误信息
static_assert(sizeof(block_q4_0) == sizeof(float) + QK4_0 / 2, "wrong q4_0 block size/padding");

// 定义一个名为 block_q4_1 的结构体
#define QK4_1 32
typedef struct {
    float   d;          // delta
    float   m;          // min
    uint8_t qs[QK4_1 / 2];  // nibbles / quants
} block_q4_1;
// 使用 static_assert 断言结构体 block_q4_1 的大小等于 float 类型的大小乘以 2 加上 QK4_1 的一半，如果不相等则输出错误信息
static_assert(sizeof(block_q4_1) == sizeof(float) * 2 + QK4_1 / 2, "wrong q4_1 block size/padding");

// 从 ggml.c 中复制粘贴的代码
// 定义一个名为 block_q8_0 的结构体
#define QK8_0 32
typedef struct {
    float   d;          // delta
    int8_t  qs[QK8_0];  // quants
} block_q8_0;
// 使用 static_assert 断言结构体 block_q8_0 的大小等于 float 类型的大小加上 QK8_0，如果不相等则输出错误信息
static_assert(sizeof(block_q8_0) == sizeof(float) + QK8_0, "wrong q8_0 block size/padding");

// "Scalar" dot product between the quantized vector x and float vector y
// 计算向量 x 和 y 的点积，返回结果
inline double dot(int n, const block_q4_0* x, const float* y) {
    // 静态数组，存储常量数值
    const static float kValues[16] = {-8.f, -7.f, -6.f, -5.f, -4.f, -3.f, -2.f, -1.f, 0.f, 1.f, 2.f, 3.f, 4.f, 5.f, 6.f, 7.f};
    // 常量，用于按位与操作
    constexpr uint32_t kMask1 = 0x0f0f0f0f;
    // 定义两个 32 位无符号整数
    uint32_t u1, u2;
    // 定义两个指向 u1 和 u2 的指针
    auto q1 = (const uint8_t*)&u1;
    auto q2 = (const uint8_t*)&u2;
    // 初始化点积的和
    double sum = 0;
    // 遍历向量 x 的元素
    for (int i=0; i<n; ++i) {
        // 获取 x 的值
        float d = x->d;
        // 获取 x 的四个无符号整数
        auto u = (const uint32_t*)x->qs;
        // 初始化临时变量 s
        float s = 0;
        // 遍历四个无符号整数
        for (int k=0; k<4; ++k) {
            // 将无符号整数按位与操作，并存储到 u1 和 u2 中
            u1 = u[k] & kMask1;
            u2 = (u[k] >> 4) & kMask1;
            // 计算点积的部分和
            s += y[0]*kValues[q1[0]] + y[1]*kValues[q2[0]] +
                 y[2]*kValues[q1[1]] + y[3]*kValues[q2[1]] +
                 y[4]*kValues[q1[2]] + y[5]*kValues[q2[2]] +
                 y[6]*kValues[q1[3]] + y[7]*kValues[q2[3]];
            // 更新向量 y 的指针位置
            y += 8;
        }
// 计算两个数组的点积
sum += s*d;
// x 自增
++x;
}
// 上述函数的另一种版本，在我的 Mac 上更快（每个点积约 45 微秒 vs 约 55 微秒），
// 但在 X86_64（Ryzen 7950X CPU）上大致相同。
inline double dot3(int n, const block_q4_0* x, const float* y) {
    // 静态数组，包含 256 个浮点数对
    const static std::pair<float,float> kValues[256] = {
        // 数组初始化
这段代码是一系列的坐标点，每个坐标点由两个浮点数组成，表示 x 和 y 坐标。
    // 定义一个二维数组，存储了一组浮点数坐标
    { 0.f,  7.f}, { 1.f,  7.f}, { 2.f,  7.f}, { 3.f,  7.f}, { 4.f,  7.f}, { 5.f,  7.f}, { 6.f,  7.f}, { 7.f,  7.f}
};
// 定义一个双精度浮点数变量 sum，并初始化为 0
double sum = 0;
// 循环遍历 n 次
for (int i=0; i<n; ++i) {
    // 获取 x 指针指向的结构体中的 d 值
    float d = x->d;
    // 获取 x 指针指向的结构体中的 qs 数组
    auto q = x->qs;
    // 定义一个浮点数变量 s，并初始化为 0
    float s = 0;
    // 内层循环，遍历 4 次
    for (int k=0; k<4; ++k) {
        // 根据给定的公式计算 s 的值
        s += y[0]*kValues[q[0]].first + y[1]*kValues[q[0]].second +
             y[2]*kValues[q[1]].first + y[3]*kValues[q[1]].second +
             y[4]*kValues[q[2]].first + y[5]*kValues[q[2]].second +
             y[6]*kValues[q[3]].first + y[7]*kValues[q[3]].second;
        // 更新 y 和 q 的指向位置
        y += 8; q += 4;
    }
    // 将 s 乘以 d 并加到 sum 上
    sum += s*d;
    // 更新 x 的指向位置
    ++x;
}
// 返回最终的 sum 值
return sum;
// 计算 n 个 block_q4_1 结构体数组 x 和 float 数组 y 的点积
inline double dot41(int n, const block_q4_1* x, const float* y) {
    // 静态数组，包含16个float值
    const static float kValues[16] = {0.f, 1.f, 2.f, 3.f, 4.f, 5.f, 6.f, 7.f, 8.f, 9.f, 10.f, 11.f, 12.f, 13.f, 14.f, 15.f};
    // 用于掩码的常量
    constexpr uint32_t kMask1 = 0x0f0f0f0f;
    // 两个32位整数
    uint32_t u1, u2;
    // 将 u1 和 u2 分别解释为 uint8_t 类型的数组
    auto q1 = (const uint8_t*)&u1;
    auto q2 = (const uint8_t*)&u2;
    // 结果的累加和
    double sum = 0;
    // 遍历 block_q4_1 结构体数组
    for (int i=0; i<n; ++i) {
        // 将 block_q4_1 结构体数组中的四个32位整数解释为 uint32_t 类型的数组
        auto u = (const uint32_t*)x->qs;
        // 临时变量 s 和 s1
        float s = 0, s1 = 0;
        // 遍历每个32位整数
        for (int k=0; k<4; ++k) {
            // 将每个32位整数按位与 kMask1，得到 u1 和 u2
            u1 = u[k] & kMask1;
            u2 = (u[k] >> 4) & kMask1;
            // 计算点积
            s += y[0]*kValues[q1[0]] + y[1]*kValues[q2[0]] +
                 y[2]*kValues[q1[1]] + y[3]*kValues[q2[1]] +
                 y[4]*kValues[q1[2]] + y[5]*kValues[q2[2]] +
                 y[6]*kValues[q1[3]] + y[7]*kValues[q2[3]];
            // 计算 y 的和
            s1 += y[0] + y[1] + y[2] + y[3] + y[4] + y[5] + y[6] + y[7];
            // 指针移动到下一个8个元素
            y += 8;
        }
        // 计算累加和，sum += s*x->d + s1*x->m
        sum += s*x->d + s1*x->m;
        // 指针 x 向后移动一位
        ++x;
    }
    // 返回累加和
    return sum;
}

// 从 ggml.c 中复制粘贴而来
// 对输入数组 x 进行量化，结果存入数组 y，k 为数组 x 的长度
static void quantize_row_q8_0_reference(const float *x, block_q8_0 *y, int k) {
    // 确保 k 是 QK8_0 的整数倍
    assert(k % QK8_0 == 0);
    // 计算数组 x 的长度 k 被 QK8_0 整除后的商
    const int nb = k / QK8_0;

    // 遍历数组 x，每次处理 QK8_0 个元素
    for (int i = 0; i < nb; i++) {
        float amax = 0.0f; // 绝对值最大值

        // 遍历 QK8_0 个元素，找出绝对值最大的元素
        for (int l = 0; l < QK8_0; l++) {
            const float v = x[i*QK8_0 + l];
            amax = std::max(amax, fabsf(v));
        }

        // 计算量化因子 d
        const float d = amax / ((1 << 7) - 1);
        // 计算倒数，如果 d 不为 0 则为 1/d，否则为 0
        const float id = d ? 1.0f/d : 0.0f;

        // 将计算得到的倒数赋值给 y[i].d
        y[i].d = d;

        // 遍历 QK8_0 次，计算 x[i*QK8_0 + l] 乘以 id 的值，并四舍五入后赋值给 y[i].qs[l]
        for (int l = 0; l < QK8_0; ++l) {
            const float   v  = x[i*QK8_0 + l]*id;
            y[i].qs[l] = roundf(v);
        }
    }
}

// 从 ggml.c 中复制粘贴过来的函数
static void dot_q4_q8(const int n, float* s, const void* vx, const void* vy) {
    // 计算 nb，即 n 除以 QK8_0 的商
    const int nb = n / QK8_0;
    // 将 vx 和 vy 强制转换为 block_q4_0* 和 block_q8_0* 类型
    const block_q4_0* x = (const block_q4_0*)vx;
    const block_q8_0* y = (const block_q8_0*)vy;
    // 初始化 sumf 为 0
    float sumf = 0;
    // 遍历 nb 次
    for (int i = 0; i < nb; i++) {
        // 将 x[i].d 和 y[i].d 分别赋值给 d0 和 d1
        const float d0 = x[i].d;
        const float d1 = y[i].d;
// 从结构体数组 x 中获取第 i 个元素的 qs 字段的地址
const uint8_t * p0 = x[i].qs;
// 从结构体数组 y 中获取第 i 个元素的 qs 字段的地址
const int8_t * p1 = y[i].qs;

// 初始化 sumi 变量
int sumi = 0;
// 遍历循环，j 从 0 到 QK8_0/2
for (int j = 0; j < QK8_0/2; j++) {
    // 从 p0 中获取第 j 个元素的值
    const uint8_t v0 = p0[j];

    // 从 v0 中获取低 4 位，转换成有符号整数并减去 8
    const int i0 = (int8_t) (v0 & 0xf) - 8;
    // 从 v0 中获取高 4 位，转换成有符号整数并减去 8
    const int i1 = (int8_t) (v0 >> 4)  - 8;

    // 从 p1 中获取第 2*j + 0 个元素的值
    const int i2 = p1[2*j + 0];
    // 从 p1 中获取第 2*j + 1 个元素的值
    const int i3 = p1[2*j + 1];

    // 计算 i0*i2 + i1*i3 的结果并累加到 sumi 变量
    sumi += i0*i2 + i1*i3;
}
// 计算 d0*d1*sumi 的结果并累加到 sumf 变量
sumf += d0*d1*sumi;
// 将 sumf 的值赋给指针 s 所指向的变量
*s = sumf;
// 主函数，接受命令行参数
int main(int argc, char** argv) {

    // 根据命令行参数确定循环次数，默认为10
    int nloop = argc > 1 ? atoi(argv[1]) : 10;
    // 根据命令行参数确定是否使用标量，默认为false
    bool scalar = argc > 2 ? atoi(argv[2]) : false;
    // 根据命令行参数确定是否使用Q4_1，默认为false
    bool useQ4_1 = argc > 3 ? atoi(argv[3]) : false;

    // 如果同时使用标量和Q4_1，则输出错误信息并返回
    if (scalar && useQ4_1) {
        printf("It is not possible to use Q4_1 quantization and scalar implementations\n");
        return 1;
    }

    // 初始化随机数生成器
    std::mt19937 rndm(1234);

    // 创建两个长度为kVecSize的浮点数向量
    std::vector<float> x1(kVecSize), y1(kVecSize);
    // 根据是否使用Q4_1确定n4的值
    int n4 = useQ4_1 ? kVecSize / QK4_1 : kVecSize / QK4_0; n4 = 64*((n4 + 63)/64);
    // 根据默认值确定n8的值
    int n8 = kVecSize / QK8_0; n8 = 64*((n8 + 63)/64);

    // 根据是否使用Q4_1选择相应的函数
    auto funcs = useQ4_1 ? ggml_internal_get_type_traits(GGML_TYPE_Q4_1) : ggml_internal_get_type_traits(GGML_TYPE_Q4_0);

    // 创建一个空的 block_q4_0 类型的向量 q40
    std::vector<block_q4_0> q40;
    // 创建一个空的 block_q4_1 类型的向量 q41
    std::vector<block_q4_1> q41;
    // 如果 useQ4_1 为真，则将 q41 的大小调整为 n4，否则将 q40 的大小调整为 n4
    if (useQ4_1) q41.resize(n4);
    else q40.resize(n4);
    // 创建一个大小为 n8 的 block_q8_0 类型的向量 q8
    std::vector<block_q8_0> q8(n8);
    // 创建一个大小为 16 的 int64_t 类型的向量 H，初始值都为 0
    std::vector<int64_t> H(16, 0);
    // 初始化一些变量
    double sumt = 0, sumt2 = 0, maxt = 0;
    double sumqt = 0, sumqt2 = 0, maxqt = 0;
    double sum = 0, sumq = 0, exactSum = 0;
    // 循环 nloop 次
    for (int iloop=0; iloop<nloop; ++iloop) {

        // 用随机数填充向量 x1
        fillRandomGaussianFloats(x1, rndm);

        // 用随机数填充向量 y1
        fillRandomGaussianFloats(y1, rndm);

        // 计算精确的点积
        for (int k=0; k<kVecSize; ++k) exactSum += x1[k]*y1[k];
        // 对 x 进行量化
        // 注意，我们不将此计入时间，因为在实际应用中，我们已经有了量化模型的权重。
        if (useQ4_1) {
            funcs.from_float(x1.data(), q41.data(), kVecSize);
        } else {
            funcs.from_float(x1.data(), q40.data(), kVecSize);
        }

        // 现在使用上面的“标量”版本来测量点积所需的时间
        auto t1 = std::chrono::high_resolution_clock::now();
        if (useQ4_1) sum += dot41(kVecSize / QK4_1, q41.data(), y1.data());
        else sum += dot(kVecSize / QK4_0, q40.data(), y1.data());
        auto t2 = std::chrono::high_resolution_clock::now();
        auto t = 1e-3*std::chrono::duration_cast<std::chrono::nanoseconds>(t2-t1).count();
        sumt += t; sumt2 += t*t; maxt = std::max(maxt, t);

        // 现在测量量化 y 和使用量化后的 y 执行点积所需的时间
        t1 = std::chrono::high_resolution_clock::now();
        float result;
        // 如果是标量操作
        if (scalar) {
            // 对 y1 进行量化为 q8，使用参考函数
            quantize_row_q8_0_reference(y1.data(), q8.data(), kVecSize);
            // 计算 q40 和 q8 的点积，结果存储在 result 中
            dot_q4_q8(kVecSize, &result, q40.data(), q8.data());
        }
        // 如果不是标量操作
        else {
            // 获取 vec_dot 函数的类型特征
            auto vdot = ggml_internal_get_type_traits(funcs.vec_dot_type);
            // 将 y1 和 q8 转换为浮点数，存储在 vdot 中
            vdot.from_float(y1.data(), q8.data(), kVecSize);
            // 如果使用 Q4_1，则调用 funcs.vec_dot 函数计算点积
            if (useQ4_1) funcs.vec_dot(kVecSize, &result, q41.data(), q8.data());
            // 否则调用 funcs.vec_dot 函数计算点积
            else funcs.vec_dot(kVecSize, &result, q40.data(), q8.data());
        }
        // 将 result 加到 sumq 中
        sumq += result;
        // 记录当前时间
        t2 = std::chrono::high_resolution_clock::now();
        // 计算时间间隔
        t = 1e-3*std::chrono::duration_cast<std::chrono::nanoseconds>(t2-t1).count();
        // 将 t 加到 sumqt 中，t 的平方加到 sumqt2 中，更新 maxqt
        sumqt += t; sumqt2 += t*t; maxqt = std::max(maxqt, t);
    }

    // 报告时间（以及点积的平均值，以防编译器优化掉函数调用）
    sum /= nloop; sumq /= nloop;
# 计算平均值
exactSum /= nloop;
# 打印精确结果
printf("Exact result: <dot> = %g\n",exactSum);
# 打印结果和结果的平方
printf("<dot> = %g, %g\n",sum,sumq);
# 计算时间的平均值和平方的平均值，并计算标准差
sumt /= nloop; sumt2 /= nloop; sumt2 -= sumt*sumt;
# 如果标准差大于0，则计算平方根
if (sumt2 > 0) sumt2 = sqrt(sumt2);
# 打印时间和标准差
printf("time = %g +/- %g us. maxt = %g us\n",sumt,sumt2,maxt);
# 计算时间的平均值和平方的平均值，并计算标准差
sumqt /= nloop; sumqt2 /= nloop; sumqt2 -= sumqt*sumqt;
# 如果标准差大于0，则计算平方根
if (sumqt2 > 0) sumqt2 = sqrt(sumqt2);
# 打印时间和标准差
printf("timeq = %g +/- %g us. maxt = %g us\n",sumqt,sumqt2,maxqt);
# 返回0表示程序执行成功
return 0;
```