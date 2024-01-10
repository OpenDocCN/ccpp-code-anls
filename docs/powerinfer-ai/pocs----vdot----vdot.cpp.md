# `PowerInfer\pocs\vdot\vdot.cpp`

```
#include <cstdio>
#include <vector>
#include <random>
#include <chrono>
#include <cstdlib>
#include <cmath>
#include <cassert>
#include <cstring>
#include <array>

#include <ggml.h>

#if defined(_MSC_VER)
#pragma warning(disable: 4244 4267) // possible loss of data
#endif

constexpr int kVecSize = 1 << 18;

static float drawFromGaussianPdf(std::mt19937& rndm) {
    // 定义常量，用于高斯分布采样
    constexpr double kScale = 1./(1. + std::mt19937::max());
    constexpr double kTwoPiTimesScale = 6.28318530717958647692*kScale;
    static float lastX;
    static bool haveX = false;
    // 如果已经有采样值，则直接返回上次的值
    if (haveX) { haveX = false; return lastX; }
    // 生成高斯分布的随机数
    auto r = sqrt(-2*log(1 - kScale*rndm()));
    auto phi = kTwoPiTimesScale * rndm();
    lastX = r*sin(phi);
    haveX = true;
    return r*cos(phi);
}

static void fillRandomGaussianFloats(std::vector<float>& values, std::mt19937& rndm, float mean = 0) {
    // 用高斯分布填充给定的浮点数向量
    for (auto& v : values) v = mean + drawFromGaussianPdf(rndm);
}

// Copy-pasted from ggml.c
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

// Copy-pasted from ggml.c
#define QK8_0 32
typedef struct {
    float   d;          // delta
    int8_t  qs[QK8_0];  // quants
} block_q8_0;
static_assert(sizeof(block_q8_0) == sizeof(float) + QK8_0, "wrong q8_0 block size/padding");

// "Scalar" dot product between the quantized vector x and float vector y
inline double dot(int n, const block_q4_0* x, const float* y) {
    // 定义常量数组，用于计算点积
    const static float kValues[16] = {-8.f, -7.f, -6.f, -5.f, -4.f, -3.f, -2.f, -1.f, 0.f, 1.f, 2.f, 3.f, 4.f, 5.f, 6.f, 7.f};
    constexpr uint32_t kMask1 = 0x0f0f0f0f;
    # 定义两个32位无符号整数变量
    uint32_t u1, u2;
    # 将u1的地址转换为指向常量uint8_t类型的指针
    auto q1 = (const uint8_t*)&u1;
    # 将u2的地址转换为指向常量uint8_t类型的指针
    auto q2 = (const uint8_t*)&u2;
    # 定义一个双精度浮点数变量sum，并初始化为0
    double sum = 0;
    # 循环，i从0到n-1
    for (int i=0; i<n; ++i) {
        # 获取结构体x中的d成员赋值给浮点数变量d
        float d = x->d;
        # 将结构体x中的qs成员转换为指向常量uint32_t类型的指针赋值给u
        auto u = (const uint32_t*)x->qs;
        # 定义一个浮点数变量s，并初始化为0
        float s = 0;
        # 内层循环，k从0到3
        for (int k=0; k<4; ++k) {
            # 将u[k]与kMask1进行按位与操作后的结果赋值给u1
            u1 = u[k] & kMask1;
            # 将u[k]右移4位后与kMask1进行按位与操作后的结果赋值给u2
            u2 = (u[k] >> 4) & kMask1;
            # 计算s的值
            s += y[0]*kValues[q1[0]] + y[1]*kValues[q2[0]] +
                 y[2]*kValues[q1[1]] + y[3]*kValues[q2[1]] +
                 y[4]*kValues[q1[2]] + y[5]*kValues[q2[2]] +
                 y[6]*kValues[q1[3]] + y[7]*kValues[q2[3]];
            # y指针向后移动8个位置
            y += 8;
        }
        # 计算sum的值
        sum += s*d;
        # 结构体指针x向后移动一个位置
        ++x;
    }
    # 返回sum的值
    return sum;
// Alternative version of the above. Faster on my Mac (~45 us vs ~55 us per dot product),
// but about the same on X86_64 (Ryzen 7950X CPU).
// 上面的另一种版本。在我的 Mac 上更快（每个点积约 45 微秒 vs 约 55 微秒），
// 但在 X86_64（Ryzen 7950X CPU）上大致相同。

inline double dot3(int n, const block_q4_0* x, const float* y) {
    // 计算点积
    double sum = 0;
    for (int i=0; i<n; ++i) {
        // 获取块中的数据和索引
        float d = x->d;
        auto q = x->qs;
        float s = 0;
        for (int k=0; k<4; ++k) {
            // 计算点积的每一项并累加
            s += y[0]*kValues[q[0]].first + y[1]*kValues[q[0]].second +
                 y[2]*kValues[q[1]].first + y[3]*kValues[q[1]].second +
                 y[4]*kValues[q[2]].first + y[5]*kValues[q[2]].second +
                 y[6]*kValues[q[3]].first + y[7]*kValues[q[3]].second;
            y += 8; q += 4;
        }
        sum += s*d;
        ++x;
    }
    return sum;
}

inline double dot41(int n, const block_q4_1* x, const float* y) {
    // 预定义的常量数组
    const static float kValues[16] = {0.f, 1.f, 2.f, 3.f, 4.f, 5.f, 6.f, 7.f, 8.f, 9.f, 10.f, 11.f, 12.f, 13.f, 14.f, 15.f};
    // 预定义的位掩码
    constexpr uint32_t kMask1 = 0x0f0f0f0f;
    uint32_t u1, u2;
    auto q1 = (const uint8_t*)&u1;
    auto q2 = (const uint8_t*)&u2;
    double sum = 0;
    for (int i=0; i<n; ++i) {
        auto u = (const uint32_t*)x->qs;
        float s = 0, s1 = 0;
        for (int k=0; k<4; ++k) {
            // 使用位运算和预定义的常量数组计算点积的每一项并累加
            u1 = u[k] & kMask1;
            u2 = (u[k] >> 4) & kMask1;
            s += y[0]*kValues[q1[0]] + y[1]*kValues[q2[0]] +
                 y[2]*kValues[q1[1]] + y[3]*kValues[q2[1]] +
                 y[4]*kValues[q1[2]] + y[5]*kValues[q2[2]] +
                 y[6]*kValues[q1[3]] + y[7]*kValues[q2[3]];
            s1 += y[0] + y[1] + y[2] + y[3] + y[4] + y[5] + y[6] + y[7];
            y += 8;
        }
        sum += s*x->d + s1*x->m;
        ++x;
    }
    return sum;
}

// Copy-pasted from ggml.c
// 从 ggml.c 中复制粘贴过来的
// 对输入数组进行量化处理，存储到输出数组中
static void quantize_row_q8_0_reference(const float *x, block_q8_0 *y, int k) {
    // 断言输入数组长度是 QK8_0 的整数倍
    assert(k % QK8_0 == 0);
    const int nb = k / QK8_0;
    // 循环遍历 nb 次
    for (int i = 0; i < nb; i++) {
        float amax = 0.0f; // 绝对值最大值

        // 内部循环遍历 QK8_0 次
        for (int l = 0; l < QK8_0; l++) {
            // 获取 x 数组中的值
            const float v = x[i*QK8_0 + l];
            // 更新绝对值最大值
            amax = std::max(amax, fabsf(v));
        }

        // 计算比例因子
        const float d = amax / ((1 << 7) - 1);
        const float id = d ? 1.0f/d : 0.0f;

        // 将比例因子赋值给 y[i].d
        y[i].d = d;

        // 再次遍历 QK8_0 次
        for (int l = 0; l < QK8_0; ++l) {
            // 计算量化后的值并赋给 y[i].qs[l]
            const float   v  = x[i*QK8_0 + l]*id;
            y[i].qs[l] = roundf(v);
        }
    }
}

// 从 ggml.c 中复制粘贴而来
static void dot_q4_q8(const int n, float* s, const void* vx, const void* vy) {
    // 计算块的数量
    const int nb = n / QK8_0;
    // 将 vx 强制转换为 block_q4_0 类型的指针
    const block_q4_0* x = (const block_q4_0*)vx;
    // 将 vy 强制转换为 block_q8_0 类型的指针
    const block_q8_0* y = (const block_q8_0*)vy;
    // 初始化浮点数 sumf
    float sumf = 0;
    // 遍历块的数量
    for (int i = 0; i < nb; i++) {
        // 获取 x[i] 的 d 值
        const float d0 = x[i].d;
        // 获取 y[i] 的 d 值
        const float d1 = y[i].d;

        // 获取 x[i] 的 qs 指针
        const uint8_t * p0 = x[i].qs;
        // 获取 y[i] 的 qs 指针
        const  int8_t * p1 = y[i].qs;

        // 初始化整数 sumi
        int sumi = 0;
        // 遍历 QK8_0/2 次
        for (int j = 0; j < QK8_0/2; j++) {
            // 获取 p0[j] 的值
            const uint8_t v0 = p0[j];

            // 计算 i0 和 i1
            const int i0 = (int8_t) (v0 & 0xf) - 8;
            const int i1 = (int8_t) (v0 >> 4)  - 8;

            // 获取 p1[2*j + 0] 和 p1[2*j + 1] 的值
            const int i2 = p1[2*j + 0];
            const int i3 = p1[2*j + 1];

            // 计算 sumi
            sumi += i0*i2 + i1*i3;
        }
        // 计算 sumf
        sumf += d0*d1*sumi;
    }
    // 将 sumf 赋值给 s
    *s = sumf;
}

int main(int argc, char** argv) {

    // 获取循环次数
    int nloop = argc > 1 ? atoi(argv[1]) : 10;
    // 获取是否使用标量实现
    bool scalar = argc > 2 ? atoi(argv[2]) : false;
    // 获取是否使用 Q4_1
    bool useQ4_1 = argc > 3 ? atoi(argv[3]) : false;

    // 如果同时使用标量实现和 Q4_1，则输出错误信息并返回
    if (scalar && useQ4_1) {
        printf("It is not possible to use Q4_1 quantization and scalar implementations\n");
        return 1;
    }

    // 初始化随机数生成器
    std::mt19937 rndm(1234);

    // 初始化两个长度为 kVecSize 的浮点数向量 x1 和 y1
    std::vector<float> x1(kVecSize), y1(kVecSize);
    // 计算 n4 和 n8
    int n4 = useQ4_1 ? kVecSize / QK4_1 : kVecSize / QK4_0; n4 = 64*((n4 + 63)/64);
    int n8 = kVecSize / QK8_0; n8 = 64*((n8 + 63)/64);

    // 获取类型特性
    auto funcs = useQ4_1 ? ggml_internal_get_type_traits(GGML_TYPE_Q4_1) : ggml_internal_get_type_traits(GGML_TYPE_Q4_0);

    // 初始化长度为 n4 或 n8 的块向量
    std::vector<block_q4_0> q40;
    std::vector<block_q4_1> q41;
    if (useQ4_1) q41.resize(n4);
    else q40.resize(n4);
    std::vector<block_q8_0> q8(n8);
    // 初始化长度为 16 的整数向量 H
    std::vector<int64_t> H(16, 0);
    // 初始化浮点数 sumt, sumt2, maxt, sumqt, sumqt2, maxqt, sum, sumq, exactSum
    double sumt = 0, sumt2 = 0, maxt = 0;
    double sumqt = 0, sumqt2 = 0, maxqt = 0;
    double sum = 0, sumq = 0, exactSum = 0;
    }

    // 报告时间（以及点积的平均值，以防编译器产生优化）
    // 将 sum 和 sumq 分别除以循环次数，得到平均值
    sum /= nloop; sumq /= nloop;
    // 将 exactSum 除以循环次数，得到平均值
    exactSum /= nloop;
    // 打印精确结果
    printf("Exact result: <dot> = %g\n",exactSum);
    // 打印 sum 和 sumq 的平均值
    printf("<dot> = %g, %g\n",sum,sumq);
    // 将 sumt 和 sumt2 分别除以循环次数，得到平均值，并计算方差
    sumt /= nloop; sumt2 /= nloop; sumt2 -= sumt*sumt;
    // 如果方差大于0，计算标准差
    if (sumt2 > 0) sumt2 = sqrt(sumt2);
    // 打印时间和最大时间
    printf("time = %g +/- %g us. maxt = %g us\n",sumt,sumt2,maxt);
    // 将 sumqt 和 sumqt2 分别除以循环次数，得到平均值，并计算方差
    sumqt /= nloop; sumqt2 /= nloop; sumqt2 -= sumqt*sumqt;
    // 如果方差大于0，计算标准差
    if (sumqt2 > 0) sumqt2 = sqrt(sumqt2);
    // 打印时间和最大时间
    printf("timeq = %g +/- %g us. maxt = %g us\n",sumqt,sumqt2,maxqt);
    // 返回 0
    return 0;
# 闭合前面的函数定义
```