# `xmrig\src\crypto\cn\CryptoNight.h`

```
// 定义 XMRig 的版权声明
/* XMRig
 * Copyright 2010      Jeff Garzik <jgarzik@pobox.com>
 * Copyright 2012-2014 pooler      <pooler@litecoinpool.org>
 * Copyright 2014      Lucas Jones <https://github.com/lucasjones>
 * Copyright 2014-2016 Wolf9466    <https://github.com/OhGodAPet>
 * Copyright 2016      Jay D Dee   <jayddee246@gmail.com>
 * Copyright 2017-2018 XMR-Stak    <https://github.com/fireice-uk>, <https://github.com/psychocrypt>
 * Copyright 2018      Lee Clagett <https://github.com/vtnerd>
 * Copyright 2018-2019 SChernykh   <https://github.com/SChernykh>
 * Copyright 2016-2018 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   This program is free software: you can redistribute it and/or modify
 *   it under the terms of the GNU General Public License as published by
 *   the Free Software Foundation, either version 3 of the License, or
 *   (at your option) any later version.
 *
 *   This program is distributed in the hope that it will be useful,
 *   but WITHOUT ANY WARRANTY; without even the implied warranty of
 *   MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the
 *   GNU General Public License for more details.
 *
 *   You should have received a copy of the GNU General Public License
 *   along with this program. If not, see <http://www.gnu.org/licenses/>.
 */

// 定义 XMRig 的头文件保护
#ifndef XMRIG_CRYPTONIGHT_H
#define XMRIG_CRYPTONIGHT_H

// 包含标准库头文件
#include <stddef.h>
#include <stdint.h>

// 定义 ABI_ATTRIBUTE 宏
#if defined _MSC_VER || defined XMRIG_ARM
#   define ABI_ATTRIBUTE
#else
#   define ABI_ATTRIBUTE __attribute__((ms_abi))
#endif

// 定义 cryptonight_ctx 结构体
struct cryptonight_ctx;
// 定义 cn_mainloop_fun_ms_abi 函数指针类型
typedef void(*cn_mainloop_fun_ms_abi)(cryptonight_ctx**) ABI_ATTRIBUTE;

// 定义 cryptonight_r_data 结构体
struct cryptonight_r_data {
    int algo; // 算法类型
    uint64_t height; // 区块高度

    // 匹配函数，判断算法类型和区块高度是否匹配
    bool match(const int a, const uint64_t h) const { return (a == algo) && (h == height); }
};

// 定义 cryptonight_ctx 结构体
struct cryptonight_ctx {
    alignas(16) uint8_t state[224]; // 224 字节对齐的状态数组
    alignas(16) uint8_t *memory; // 16 字节对齐的内存指针
    const uint32_t* tweak1_table; // tweak1 表指针
    uint64_t tweak1_2; // tweak1_2 变量

    uint8_t unused[24]; // 24 字节的未使用空间
    const uint32_t *saes_table; // saes 表指针

... (继续下面的代码注释)
    // 声明一个名为 cn_mainloop_fun_ms_abi 的变量，类型为 generated_code
    cn_mainloop_fun_ms_abi generated_code;
    // 声明一个名为 cryptonight_r_data 的变量，类型为 generated_code_data
    cryptonight_r_data generated_code_data;

    // 声明一个名为 save_state 的数组，长度为 128，类型为 uint8_t，内存对齐为 16
    alignas(16) uint8_t save_state[128];
    // 声明一个名为 first_half 的布尔变量
    bool first_half;
};
// 结束if条件语句的闭合
#endif /* XMRIG_CRYPTONIGHT_H */
// 结束头文件的条件编译指令
```