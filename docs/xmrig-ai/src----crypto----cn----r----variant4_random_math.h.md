# `xmrig\src\crypto\cn\r\variant4_random_math.h`

```cpp
#ifndef VARIANT4_RANDOM_MATH_H
#define VARIANT4_RANDOM_MATH_H

#include <cstring>

#include "base/crypto/Algorithm.h"

extern "C"
{
    #include "crypto/cn/c_blake256.h"
}

// 定义枚举类型 V4_Settings
enum V4_Settings
{
    // 生成具有最小理论延迟的代码 = 45个周期，相当于15次乘法运算
    TOTAL_LATENCY = 15 * 3,
    
    // 总是生成至少60条指令
    NUM_INSTRUCTIONS_MIN = 60,

    // 从不生成超过70条指令（最终的RET指令不计算在内）
    NUM_INSTRUCTIONS_MAX = 70,

    // 用于MUL的可用ALU
    // 现代CPU通常只有1个可以进行乘法运算的ALU
    ALU_COUNT_MUL = 1,

    // 总可用ALU
    // 现代CPU有4个ALU，但我们只使用3个，因为随机数运算与其他主循环代码一起执行
    ALU_COUNT = 3,
};

// 定义枚举类型 V4_InstructionList
enum V4_InstructionList
{
    MUL,    // a*b
    ADD,    // a+b + C, C是一个无符号32位常数
    SUB,    // a-b
    ROR,    // 将"a"向右旋转"b & 31"位
    ROL,    // 将"a"向左旋转"b & 31"位
    XOR,    // a^b
    RET,    // 结束执行
    V4_INSTRUCTION_COUNT = RET,
};

// V4_InstructionDefinition用于从随机数据生成代码
// 每个随机字节序列都是有效的代码
//
// 总共有9个寄存器：
// - 4个变量寄存器
// - 从循环变量初始化的5个常量寄存器
// 这就是为什么dst_index是2位的原因
enum V4_InstructionDefinition
{
    V4_OPCODE_BITS = 3,
    V4_DST_INDEX_BITS = 2,
    V4_SRC_INDEX_BITS = 3,
};

// 定义结构体 V4_Instruction
struct V4_Instruction
{
    uint8_t opcode;
    uint8_t dst_index;
    uint8_t src_index;
    uint32_t C;
};

// 定义宏 FORCEINLINE
#ifndef FORCEINLINE
#ifdef __GNUC__
#define FORCEINLINE __attribute__((always_inline)) inline
#elif _MSC_VER
#define FORCEINLINE __forceinline
#else
#define FORCEINLINE inline
#endif
#endif

// 定义宏 UNREACHABLE_CODE
#ifndef UNREACHABLE_CODE
#ifdef __GNUC__
#define UNREACHABLE_CODE __builtin_unreachable()
#elif _MSC_VER
#define UNREACHABLE_CODE __assume(false)
#else
// 定义未到达的代码
#define UNREACHABLE_CODE
#endif
#endif

// 随机数学解释器的循环完全展开并内联，以实现在 CPU 上达到 100% 的分支预测：
// 每个 switch-case 在 Cryptonight 主循环的每次迭代中都会指向相同的目标
//
// 这是在不使用低级机器码生成的情况下速度最快的方式
template<typename v4_reg>
static void v4_random_math(const struct V4_Instruction* code, v4_reg* r)
{
    enum
    {
        REG_BITS = sizeof(v4_reg) * 8,
    };

#define V4_EXEC(i) \
    { \
        const struct V4_Instruction* op = code + i; \  // 获取指令数组中的指令
        const v4_reg src = r[op->src_index]; \  // 获取源寄存器的值
        v4_reg* dst = r + op->dst_index; \  // 获取目标寄存器的地址
        switch (op->opcode) \  // 根据指令操作码执行相应的操作
        { \
        case MUL: \  // 乘法操作
            *dst *= src; \  // 目标寄存器的值乘以源寄存器的值
            break; \
        case ADD: \  // 加法操作
            *dst += src + op->C; \  // 目标寄存器的值加上源寄存器的值和常数 C
            break; \
        case SUB: \  // 减法操作
            *dst -= src; \  // 目标寄存器的值减去源寄存器的值
            break; \
        case ROR: \  // 右循环移位操作
            { \
                const uint32_t shift = src % REG_BITS; \  // 计算移位数
                *dst = (*dst >> shift) | (*dst << ((REG_BITS - shift) % REG_BITS)); \  // 执行右循环移位操作
            } \
            break; \
        case ROL: \  // 左循环移位操作
            { \
                const uint32_t shift = src % REG_BITS; \  // 计算移位数
                *dst = (*dst << shift) | (*dst >> ((REG_BITS - shift) % REG_BITS)); \  // 执行左循环移位操作
            } \
            break; \
        case XOR: \  // 异或操作
            *dst ^= src; \  // 目标寄存器的值与源寄存器的值进行异或操作
            break; \
        case RET: \  // 返回操作
            return; \  // 返回
        default: \  // 默认情况
            UNREACHABLE_CODE; \  // 未到达的代码
            break; \
        } \
    }

#define V4_EXEC_10(j) \  // 定义执行 10 条指令的宏
    V4_EXEC(j + 0) \  // 执行第 j 条指令
    V4_EXEC(j + 1) \  // 执行第 j+1 条指令
    V4_EXEC(j + 2) \  // 执行第 j+2 条指令
    V4_EXEC(j + 3) \  // 执行第 j+3 条指令
    V4_EXEC(j + 4) \  // 执行第 j+4 条指令
    V4_EXEC(j + 5) \  // 执行第 j+5 条指令
    V4_EXEC(j + 6) \  // 执行第 j+6 条指令
    V4_EXEC(j + 7) \  // 执行第 j+7 条指令
    V4_EXEC(j + 8) \  // 执行第 j+8 条指令
    V4_EXEC(j + 9) \  // 执行第 j+9 条指令

    // 生成的程序可以有 60 条指令加上几条（通常是 2-3 条）指令以达到所需的延迟
    // 我已经检查了所有块高度小于 10,000,000 的情况，这里是程序大小的分布情况:
    // 60      27960
    // 61      105054
    // 62      2452759
    // 63      5115997
    // 64      1022269
    // 65      1109635
    // 66      153145
    // 67      8550
    // 68      4529
    // 69      102

    // 在这里展开70条指令
    V4_EXEC_10(0);        // 指令0-9
    V4_EXEC_10(10);        // 指令10-19
    V4_EXEC_10(20);        // 指令20-29
    V4_EXEC_10(30);        // 指令30-39
    V4_EXEC_10(40);        // 指令40-49
    V4_EXEC_10(50);        // 指令50-59
    V4_EXEC_10(60);        // 指令60-69
// 取消定义 V4_EXEC_10
#undef V4_EXEC_10
// 取消定义 V4_EXEC
#undef V4_EXEC
}

// 如果可用数据不足，生成更多数据
static FORCEINLINE void check_data(size_t* data_index, const size_t bytes_needed, int8_t* data, const size_t data_size)
{
    if (*data_index + bytes_needed > data_size)
    {
        hash_extra_blake(data, data_size, (char*) data);
        *data_index = 0;
    }
}

// 根据给定的延迟和 ALU 限制生成尽可能多的随机数学运算
// "code" 数组必须有空间容纳 NUM_INSTRUCTIONS_MAX+1 条指令
template<xmrig::Algorithm::Id ALGO>
static int v4_random_math_init(struct V4_Instruction* code, const uint64_t height)
{
    // MUL 是 3 个周期，3 路加法和旋转是 2 个周期，SUB/XOR 是 1 个周期
    // 这些延迟与英特尔 CPU 的实际指令延迟匹配，从 Sandy Bridge 开始一直到 Skylake/Coffee Lake
    //
    // AMD Ryzen 也有相同的延迟，除了 1 个周期的 ROR/ROL，所以它会比英特尔 Sandy Bridge 和更新的处理器稍快一些
    // 令人惊讶的是，英特尔 Nehalem 也有 1 个周期的 ROR/ROL，所以它也会比英特尔 Sandy Bridge 和更新的处理器更快
    // AMD Bulldozer 的 MUL 延迟为 4 个周期（比英特尔慢），ROR/ROL 延迟为 1 个周期（比英特尔快），因此平均性能将相同
    // 来源：https://www.agner.org/optimize/instruction_tables.pdf
    const int op_latency[V4_INSTRUCTION_COUNT] = { 3, 2, 1, 2, 2, 1 };

    // 理论 ASIC 实现的指令延迟
    const int asic_op_latency[V4_INSTRUCTION_COUNT] = { 3, 1, 1, 1, 1, 1 };

    // 每个指令的可用 ALU
    const int op_ALUs[V4_INSTRUCTION_COUNT] = { ALU_COUNT_MUL, ALU_COUNT, ALU_COUNT, ALU_COUNT, ALU_COUNT, ALU_COUNT };

    int8_t data[32];
    memset(data, 0, sizeof(data));
    uint64_t tmp = SWAP64LE(height);
    memcpy(data, &tmp, sizeof(uint64_t));
    if (ALGO == xmrig::Algorithm::CN_R)    {
        data[20] = -38;
    }
    // 将 data_index 设置为 data 中最后一个字节的后面，以触发使用 blake 哈希进行完整数据更新
    size_t data_index = sizeof(data);

    int code_size;

    // 存在小概率（1.8%）注册 R8 不会在生成的程序中使用
    // 因此我们跟踪它，并在未使用时再次尝试
    bool r8_used;
    // 循环条件几乎总是为假（约98.15%的概率），因此这个循环大部分时间只执行1次迭代
    // 对于所有块高度 < 10,000,000，它最多执行4次迭代
    }  while (!r8_used || (code_size < NUM_INSTRUCTIONS_MIN) || (code_size > NUM_INSTRUCTIONS_MAX));

    // 在这里保证 NUM_INSTRUCTIONS_MIN <= code_size <= NUM_INSTRUCTIONS_MAX
    // 添加最后一条指令来停止解释器
    code[code_size].opcode = RET;
    code[code_size].dst_index = 0;
    code[code_size].src_index = 0;
    code[code_size].C = 0;

    // 返回代码大小
    return code_size;
}
# 结束一个代码块或函数的定义
#endif
# 结束一个条件编译指令的定义
```