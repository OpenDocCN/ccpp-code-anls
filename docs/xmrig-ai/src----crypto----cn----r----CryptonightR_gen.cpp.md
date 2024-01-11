# `xmrig\src\crypto\cn\r\CryptonightR_gen.cpp`

```
/* XMRig
 * 版权所有 2010      Jeff Garzik <jgarzik@pobox.com>
 * 版权所有 2012-2014 pooler      <pooler@litecoinpool.org>
 * 版权所有 2014      Lucas Jones <https://github.com/lucasjones>
 * 版权所有 2014-2016 Wolf9466    <https://github.com/OhGodAPet>
 * 版权所有 2016      Jay D Dee   <jayddee246@gmail.com>
 * 版权所有 2017-2018 XMR-Stak    <https://github.com/fireice-uk>, <https://github.com/psychocrypt>
 * 版权所有 2018      Lee Clagett <https://github.com/vtnerd>
 * 版权所有 2018-2019 SChernykh   <https://github.com/SChernykh>
 * 版权所有 2016-2019 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改
 *   根据 GNU 通用公共许可证的条款，由
 *   自由软件基金会发布的版本 3 或
 *   （根据您的选择）任何更高版本。
 *
 *   本程序是希望它有用，
 *   但没有任何保证；甚至没有暗示的保证
 *   商品性或适用于特定目的。有关更多详细信息，请参见
 *   GNU 通用公共许可证。
 *
 *   您应该已收到 GNU 通用公共许可证的副本
 *   与本程序一起。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */

#include <cstring>
#include "crypto/cn/CryptoNight_monero.h"

// 定义一个指向无返回值函数的指针类型
typedef void(*void_func)();

#include "crypto/cn/asm/CryptonightR_template.h"
#include "crypto/common/Assembly.h"
#include "crypto/common/VirtualMemory.h"

// 添加代码到指定位置
static inline void add_code(uint8_t* &p, void (*p1)(), void (*p2)())
{
    // 计算要添加的代码的大小
    const ptrdiff_t size = reinterpret_cast<const uint8_t*>(p2) - reinterpret_cast<const uint8_t*>(p1);
    // 如果大小大于0
    if (size > 0) {
        // 将代码从 p1 复制到 p，并更新 p 的位置
        memcpy(p, reinterpret_cast<void*>(p1), size);
        p += size;
    }
}

// 添加随机数学运算到指定位置
static inline void add_random_math(uint8_t* &p, const V4_Instruction* code, int code_size, const void_func* instructions, const void_func* instructions_mov, bool is_64_bit, xmrig::Assembly::Id ASM)
{
    // 初始化前一个旋转源为无效值
    uint32_t prev_rot_src = (uint32_t)(-1);

    // 循环遍历指令数组
    for (int i = 0;; ++i) {
        // 获取当前指令
        const V4_Instruction inst = code[i];
        // 如果当前指令是RET，则跳出循环
        if (inst.opcode == RET) {
            break;
        }

        // 根据当前指令的操作码确定新的操作码
        uint8_t opcode = (inst.opcode == MUL) ? inst.opcode : (inst.opcode + 2);
        uint8_t dst_index = inst.dst_index;
        uint8_t src_index = inst.src_index;

        // 提取操作数和操作码，组成新的指令
        const uint32_t a = inst.dst_index;
        const uint32_t b = inst.src_index;
        const uint8_t c = opcode | (dst_index << V4_OPCODE_BITS) | (((src_index == 8) ? dst_index : src_index) << (V4_OPCODE_BITS + V4_DST_INDEX_BITS));

        // 根据当前指令的操作码执行不同的操作
        switch (inst.opcode) {
        case ROR:
        case ROL:
            // 如果当前指令是ROR或ROL，并且旋转源不同于前一个旋转源，则添加MOV指令
            if (b != prev_rot_src) {
                prev_rot_src = b;
                add_code(p, instructions_mov[c], instructions_mov[c + 1]);
            }
            break;
        }

        // 如果当前指令的目的操作数等于前一个旋转源，则将前一个旋转源设置为无效值
        if (a == prev_rot_src) {
            prev_rot_src = (uint32_t)(-1);
        }

        // 获取指令的起始地址
        void_func begin = instructions[c];

        // 如果是AMD Bulldozer架构，并且指令是MUL，并且不是64位模式，则执行特定操作
        if ((ASM == xmrig::Assembly::BULLDOZER) && (inst.opcode == MUL) && !is_64_bit) {
            // AMD Bulldozer在32位模式下32位IMUL的延迟为4，64位IMUL的延迟为6
            // 在32位模式下始终使用32位IMUL - 跳过前缀0x48并将0x49更改为0x41
            uint8_t* prefix = reinterpret_cast<uint8_t*>(begin);

            if (*prefix == 0x49) {
                *(p++) = 0x41;
            }

            begin = reinterpret_cast<void_func>(prefix + 1);
        }

        // 添加指令到代码块中
        add_code(p, begin, instructions[c + 1]);

        // 如果当前指令是ADD，则将常数C写入指令的位置，并根据是否是64位模式设置前一个旋转源为无效值
        if (inst.opcode == ADD) {
            *(uint32_t*)(p - sizeof(uint32_t) - (is_64_bit ? 3 : 0)) = inst.C;
            if (is_64_bit) {
                prev_rot_src = (uint32_t)(-1);
            }
        }
    }
// 编译 V4 指令集的代码，将其转换为机器码
void v4_compile_code(const V4_Instruction* code, int code_size, void* machine_code, xmrig::Assembly ASM)
{
    // 将机器码的起始地址转换为 uint8_t 指针
    uint8_t* p0 = reinterpret_cast<uint8_t*>(machine_code);
    // 创建一个指针 p，指向机器码的起始地址
    uint8_t* p = p0;

    // 添加 CryptonightR 模板的第一部分代码
    add_code(p, CryptonightR_template_part1, CryptonightR_template_part2);
    // 添加随机数学运算的代码
    add_random_math(p, code, code_size, instructions, instructions_mov, false, ASM);
    // 添加 CryptonightR 模板的第二部分代码
    add_code(p, CryptonightR_template_part2, CryptonightR_template_part3);
    // 计算并填充循环偏移量
    *(int*)(p - 4) = static_cast<int>((((const uint8_t*)CryptonightR_template_mainloop) - ((const uint8_t*)CryptonightR_template_part1)) - (p - p0));
    // 添加 CryptonightR 模板的第三部分代码
    add_code(p, CryptonightR_template_part3, CryptonightR_template_end);

    // 刷新指令缓存
    xmrig::VirtualMemory::flushInstructionCache(machine_code, p - p0);
}

// 编译 V4 指令集的双倍代码，将其转换为机器码
void v4_compile_code_double(const V4_Instruction* code, int code_size, void* machine_code, xmrig::Assembly ASM)
{
    // 将机器码的起始地址转换为 uint8_t 指针
    uint8_t* p0 = reinterpret_cast<uint8_t*>(machine_code);
    // 创建一个指针 p，指向机器码的起始地址
    uint8_t* p = p0;

    // 添加 CryptonightR 双倍模板的第一部分代码
    add_code(p, CryptonightR_template_double_part1, CryptonightR_template_double_part2);
    // 添加随机数学运算的代码
    add_random_math(p, code, code_size, instructions, instructions_mov, false, ASM);
    // 添加 CryptonightR 双倍模板的第二部分代码
    add_code(p, CryptonightR_template_double_part2, CryptonightR_template_double_part3);
    // 添加随机数学运算的代码
    add_random_math(p, code, code_size, instructions, instructions_mov, false, ASM);
    // 添加 CryptonightR 双倍模板的第三部分代码
    add_code(p, CryptonightR_template_double_part3, CryptonightR_template_double_part4);
    // 计算并填充循环偏移量
    *(int*)(p - 4) = static_cast<int>((((const uint8_t*)CryptonightR_template_double_mainloop) - ((const uint8_t*)CryptonightR_template_double_part1)) - (p - p0));
    // 添加 CryptonightR 双倍模板的第四部分代码
    add_code(p, CryptonightR_template_double_part4, CryptonightR_template_double_end);

    // 刷新指令缓存
    xmrig::VirtualMemory::flushInstructionCache(machine_code, p - p0);
}

// 编译 V4 指令集的软件 AES 代码，将其转换为机器码
void v4_soft_aes_compile_code(const V4_Instruction* code, int code_size, void* machine_code, xmrig::Assembly ASM)
{
    // 将机器码的起始地址转换为 uint8_t 指针
    uint8_t* p0 = reinterpret_cast<uint8_t*>(machine_code);
    // 创建一个指针 p，指向机器码的起始地址
    uint8_t* p = p0;

    // 添加 CryptonightR 软件 AES 模板的第一部分代码
    add_code(p, CryptonightR_soft_aes_template_part1, CryptonightR_soft_aes_template_part2);
    # 在指定位置插入随机数和数学运算指令
    add_random_math(p, code, code_size, instructions, instructions_mov, false, ASM);
    # 在指定位置插入代码
    add_code(p, CryptonightR_soft_aes_template_part2, CryptonightR_soft_aes_template_part3);
    # 计算并写入偏移量
    *(int*)(p - 4) = static_cast<int>((((const uint8_t*)CryptonightR_soft_aes_template_mainloop) - ((const uint8_t*)CryptonightR_soft_aes_template_part1)) - (p - p0));
    # 在指定位置插入代码
    add_code(p, CryptonightR_soft_aes_template_part3, CryptonightR_soft_aes_template_end);

    # 刷新指令缓存
    xmrig::VirtualMemory::flushInstructionCache(machine_code, p - p0);
# 闭合前面的函数定义
```