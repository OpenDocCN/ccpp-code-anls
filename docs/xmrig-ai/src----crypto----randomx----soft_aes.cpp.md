# `xmrig\src\crypto\randomx\soft_aes.cpp`

```cpp
/*
版权声明
*/
#include "crypto/randomx/soft_aes.h"

// 定义并对齐存储空间，用于 AES 加密的查找表
alignas(64) uint32_t lutEnc0[256];
alignas(64) uint32_t lutEnc1[256];
alignas(64) uint32_t lutEnc2[256];
alignas(64) uint32_t lutEnc3[256];

// 定义并对齐存储空间，用于 AES 解密的查找表
alignas(64) uint32_t lutDec0[256];
alignas(64) uint32_t lutDec1[256];
alignas(64) uint32_t lutDec2[256];
alignas(64) uint32_t lutDec3[256];

// 定义静态函数，用于在有限域 GF(2^8) 上进行乘法运算
static uint32_t mul_gf2(uint32_t b, uint32_t c)
{
    uint32_t s = 0;
    # 使用三个变量 i, j, k 分别初始化为 b, c, 1
    for (uint32_t i = b, j = c, k = 1; (k < 0x100) && j; k <<= 1)
    {
        # 如果 j 的最低位为 1
        if (j & k)
        {
            # 对 s 进行异或操作
            s ^= i;
            # 对 j 进行异或操作
            j ^= k;
        }

        # 左移 i 一位
        i <<= 1;
        # 如果 i 的最高位为 1
        if (i & 0x100)
            # 对 i 进行异或操作
            i ^= (1 << 8) | (1 << 4) | (1 << 3) | (1 << 1) | (1 << 0);
    }

    # 返回结果 s
    return s;
// 定义一个宏，用于将一个8位无符号整数左移指定位数或右移指定位数
#define ROTL8(x,shift) ((uint8_t) ((x) << (shift)) | ((x) >> (8 - (shift))))

// 定义一个结构体，用于初始化SAES
static struct SAESInitializer
{
    // 构造函数，用于初始化SAES
    SAESInitializer()
    {
        // 静态数组，用于存储S盒和S盒的逆变换
        static uint8_t sbox[256];
        static uint8_t sbox_reverse[256];

        // 初始化p和q
        uint8_t p = 1, q = 1;

        // 使用do-while循环生成S盒和S盒的逆变换
        do {
            // 更新p的值
            p = p ^ (p << 1) ^ (p & 0x80 ? 0x1B : 0);

            // 更新q的值
            q ^= q << 1;
            q ^= q << 2;
            q ^= q << 4;
            q ^= (q & 0x80) ? 0x09 : 0;

            // 计算S盒的值并存储到sbox数组中
            const uint8_t value = q ^ ROTL8(q, 1) ^ ROTL8(q, 2) ^ ROTL8(q, 3) ^ ROTL8(q, 4) ^ 0x63;
            sbox[p] = value;
            sbox_reverse[value] = p;
        } while (p != 1);

        // 设置S盒和S盒的逆变换的初始值
        sbox[0] = 0x63;
        sbox_reverse[0x63] = 0;

        // 使用for循环生成加密和解密的查找表
        for (uint32_t i = 0; i < 0x100; ++i)
        {
            // 定义一个联合体，用于将32位整数转换为4个8位整数
            union
            {
                uint32_t w;
                uint8_t p[4];
            };

            // 计算加密查找表的值
            uint32_t s = sbox[i];
            p[0] = mul_gf2(s, 2);
            p[1] = s;
            p[2] = s;
            p[3] = mul_gf2(s, 3);

            lutEnc0[i] = w; w = (w << 8) | (w >> 24);
            lutEnc1[i] = w; w = (w << 8) | (w >> 24);
            lutEnc2[i] = w; w = (w << 8) | (w >> 24);
            lutEnc3[i] = w;

            // 计算解密查找表的值
            s = sbox_reverse[i];
            p[0] = mul_gf2(s, 0xe);
            p[1] = mul_gf2(s, 0x9);
            p[2] = mul_gf2(s, 0xd);
            p[3] = mul_gf2(s, 0xb);

            lutDec0[i] = w; w = (w << 8) | (w >> 24);
            lutDec1[i] = w; w = (w << 8) | (w >> 24);
            lutDec2[i] = w; w = (w << 8) | (w >> 24);
            lutDec3[i] = w;
        }
    }
} aes_initializer;
```