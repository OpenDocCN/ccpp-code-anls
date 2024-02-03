# `xmrig\src\crypto\ghostrider\sph_shavite.c`

```cpp
# 定义了一个宏，用于将常量转换为32位无符号整数
#define C32   SPH_C32

# 导入标准库头文件
#include <stddef.h>
#include <string.h>

# 导入 SHAvite 头文件
#include "sph_shavite.h"

# 如果定义了 __cplusplus 宏，则将 extern "C" 添加到下面的代码块中
#ifdef __cplusplus
extern "C"{
#endif

# 如果定义了 SPH_SMALL_FOOTPRINT 宏，并且未定义 SPH_SMALL_FOOTPRINT_SHAVITE 宏，则将 SPH_SMALL_FOOTPRINT_SHAVITE 宏设置为1
#if SPH_SMALL_FOOTPRINT && !defined SPH_SMALL_FOOTPRINT_SHAVITE
#define SPH_SMALL_FOOTPRINT_SHAVITE   1
#endif

# 如果定义了 _MSC_VER 宏，则禁用警告4146
#ifdef _MSC_VER
#pragma warning (disable: 4146)
#endif

# 定义了一个宏，用于将常量转换为32位无符号整数
#define C32   SPH_C32
/*
 * SHA-3竞赛第二轮发布的参考实现和测试向量是错误的，因为它们使用大端AES表，而内部解码使用小端。
 * 以下代码遵循规范。要将其转换为遵循参考实现的代码（在SHAvite-3网站上称为“BugFix”，于2009年11月23日发布），
 * 请注释掉下面的代码（从“#define AES_BIG_ENDIAN...”到AES_ROUND_NOKEY宏的定义），并用后面被注释掉的版本替换它。
 */

// 定义AES_BIG_ENDIAN为0
#define AES_BIG_ENDIAN   0
// 包含aes_helper.c文件
#include "aes_helper.c"

// 定义初始向量IV224
static const sph_u32 IV224[] = {
    C32(0x6774F31C), C32(0x990AE210), C32(0xC87D4274), C32(0xC9546371),
    C32(0x62B2AEA8), C32(0x4B5801D8), C32(0x1B702860), C32(0x842F3017)
};

// 定义初始向量IV256
static const sph_u32 IV256[] = {
    C32(0x49BB3E47), C32(0x2674860D), C32(0xA8B392AC), C32(0x021AC4E6),
    C32(0x409283CF), C32(0x620E5D86), C32(0x6D929DCB), C32(0x96CC2A8B)
};

// 定义初始向量IV384
static const sph_u32 IV384[] = {
    C32(0x83DF1545), C32(0xF9AAEC13), C32(0xF4803CB0), C32(0x11FE1F47),
    C32(0xDA6CD269), C32(0x4F53FCD7), C32(0x950529A2), C32(0x97908147),
    C32(0xB0A4D7AF), C32(0x2B9132BF), C32(0x226E607D), C32(0x3C0F8D7C),
    C32(0x487B3F0F), C32(0x04363E22), C32(0x0155C99C), C32(0xEC2E20D3)
};

// 定义初始向量IV512
static const sph_u32 IV512[] = {
    C32(0x72FCCDD8), C32(0x79CA4727), C32(0x128A077B), C32(0x40D55AEC),
    C32(0xD1901A06), C32(0x430AE307), C32(0xB29F5CD1), C32(0xDF07FBFC),
    C32(0x8E45D73D), C32(0x681AB538), C32(0xBDE86578), C32(0xDD577E47),
    C32(0xE275EADE), C32(0x502D9FCD), C32(0xB9357178), C32(0x022A4B9A)
};

// 定义AES_ROUND_NOKEY宏，用于执行AES轮函数
#define AES_ROUND_NOKEY(x0, x1, x2, x3)   do { \
        // 临时变量t0、t1、t2、t3分别存储x0、x1、x2、x3的值
        sph_u32 t0 = (x0); \
        sph_u32 t1 = (x1); \
        sph_u32 t2 = (x2); \
        sph_u32 t3 = (x3); \
        // 调用AES_ROUND_NOKEY_LE宏执行AES轮函数
        AES_ROUND_NOKEY_LE(t0, t1, t2, t3, x0, x1, x2, x3); \
    } while (0)
/*
 * This is the code needed to match the "reference implementation" as
 * published on Nov 23rd, 2009, instead of the published specification.
 */

// 定义 AES_BIG_ENDIAN 为 1
#define AES_BIG_ENDIAN   1
// 包含 aes_helper.c 文件
#include "aes_helper.c"

// 定义 IV224 常量数组
static const sph_u32 IV224[] = {
    C32(0xC4C67795), C32(0xC0B1817F), C32(0xEAD88924), C32(0x1ABB1BB0),
    C32(0xE0C29152), C32(0xBDE046BA), C32(0xAEEECF99), C32(0x58D509D8)
};

// 定义 IV256 常量数组
static const sph_u32 IV256[] = {
    C32(0x3EECF551), C32(0xBF10819B), C32(0xE6DC8559), C32(0xF3E23FD5),
    C32(0x431AEC73), C32(0x79E3F731), C32(0x98325F05), C32(0xA92A31F1)
};

// 定义 IV384 常量数组
static const sph_u32 IV384[] = {
    C32(0x71F48510), C32(0xA903A8AC), C32(0xFE3216DD), C32(0x0B2D2AD4),
    C32(0x6672900A), C32(0x41032819), C32(0x15A7D780), C32(0xB3CAB8D9),
    C32(0x34EF4711), C32(0xDE019FE8), C32(0x4D674DC4), C32(0xE056D96B),
    C32(0xA35C016B), C32(0xDD903BA7), C32(0x8C1B09B4), C32(0x2C3E9F25)
};

// 定义 IV512 常量数组
static const sph_u32 IV512[] = {
    C32(0xD5652B63), C32(0x25F1E6EA), C32(0xB18F48FA), C32(0xA1EE3A47),
    C32(0xC8B67B07), C32(0xBDCE48D3), C32(0xE3937B78), C32(0x05DB5186),
    C32(0x613BE326), C32(0xA11FA303), C32(0x90C833D4), C32(0x79CEE316),
    C32(0x1E1AF00F), C32(0x2829B165), C32(0x23B25F80), C32(0x21E11499)
};

// 定义 AES_ROUND_NOKEY 宏
#define AES_ROUND_NOKEY(x0, x1, x2, x3)   do { \
        sph_u32 t0 = (x0); \
        sph_u32 t1 = (x1); \
        sph_u32 t2 = (x2); \
        sph_u32 t3 = (x3); \
        AES_ROUND_NOKEY_BE(t0, t1, t2, t3, x0, x1, x2, x3); \
    } while (0)

// 定义 KEY_EXPAND_ELT 宏
#define KEY_EXPAND_ELT(k0, k1, k2, k3)   do { \
        sph_u32 kt; \
        AES_ROUND_NOKEY(k1, k2, k3, k0); \
        kt = (k0); \
        (k0) = (k1); \
        (k1) = (k2); \
        (k2) = (k3); \
        (k3) = kt; \
    } while (0)

// 如果 SPH_SMALL_FOOTPRINT_SHAVITE 宏被定义，则执行以下代码
#if SPH_SMALL_FOOTPRINT_SHAVITE

/*
 * This function assumes that "msg" is aligned for 32-bit access.
 */
// 定义 c256 函数
static void
c256(sph_shavite_small_context *sc, const void *msg)
{
    // 定义局部变量
    sph_u32 p0, p1, p2, p3, p4, p5, p6, p7;
    sph_u32 rk[144];
    size_t u;
    int r, s;

// 如果 SPH_LITTLE_ENDIAN 宏被定义，则执行以下代码
    # 将msg指向的内存块的内容复制到rk指向的内存块中，复制的长度为64个字节
#else
    # 如果不满足条件，则执行以下代码块
    for (u = 0; u < 16; u += 4) {
        # 从消息中提取4个字节，按照小端序解析成32位整数，并存入rk数组中
        rk[u + 0] = sph_dec32le_aligned(
            (const unsigned char *)msg + (u << 2) +  0);
        rk[u + 1] = sph_dec32le_aligned(
            (const unsigned char *)msg + (u << 2) +  4);
        rk[u + 2] = sph_dec32le_aligned(
            (const unsigned char *)msg + (u << 2) +  8);
        rk[u + 3] = sph_dec32le_aligned(
            (const unsigned char *)msg + (u << 2) + 12);
    }
#endif
    # 将变量u赋值为16
    u = 16;
    # 循环4次，r从0到3
    for (r = 0; r < 4; r ++) {
        # 循环2次，s从0到1
        for (s = 0; s < 2; s ++) {
            # 定义4个变量并赋值
            sph_u32 x0, x1, x2, x3;

            # 根据索引计算x0, x1, x2, x3的值
            x0 = rk[u - 15];
            x1 = rk[u - 14];
            x2 = rk[u - 13];
            x3 = rk[u - 16];
            # 调用AES_ROUND_NOKEY函数
            AES_ROUND_NOKEY(x0, x1, x2, x3);
            # 计算rk[u+0]到rk[u+3]的值
            rk[u + 0] = x0 ^ rk[u - 4];
            rk[u + 1] = x1 ^ rk[u - 3];
            rk[u + 2] = x2 ^ rk[u - 2];
            rk[u + 3] = x3 ^ rk[u - 1];
            # 判断条件，更新rk[16]和rk[17]的值
            if (u == 16) {
                rk[ 16] ^= sc->count0;
                rk[ 17] ^= SPH_T32(~sc->count1);
            } else if (u == 56) {
                rk[ 57] ^= sc->count1;
                rk[ 58] ^= SPH_T32(~sc->count0);
            }
            # 更新u的值
            u += 4;

            # 重复上述操作
            x0 = rk[u - 15];
            x1 = rk[u - 14];
            x2 = rk[u - 13];
            x3 = rk[u - 16];
            AES_ROUND_NOKEY(x0, x1, x2, x3);
            rk[u + 0] = x0 ^ rk[u - 4];
            rk[u + 1] = x1 ^ rk[u - 3];
            rk[u + 2] = x2 ^ rk[u - 2];
            rk[u + 3] = x3 ^ rk[u - 1];
            if (u == 84) {
                rk[ 86] ^= sc->count1;
                rk[ 87] ^= SPH_T32(~sc->count0);
            } else if (u == 124) {
                rk[124] ^= sc->count0;
                rk[127] ^= SPH_T32(~sc->count1);
            }
            u += 4;
        }
        # 循环4次，s从0到3
        for (s = 0; s < 4; s ++) {
            # 计算rk[u+0]到rk[u+3]的值
            rk[u + 0] = rk[u - 16] ^ rk[u - 3];
            rk[u + 1] = rk[u - 15] ^ rk[u - 2];
            rk[u + 2] = rk[u - 14] ^ rk[u - 1];
            rk[u + 3] = rk[u - 13] ^ rk[u - 0];
            # 更新u的值
            u += 4;
        }
    }

    # 获取sc->h数组中的值并赋给p0到p7
    p0 = sc->h[0x0];
    p1 = sc->h[0x1];
    p2 = sc->h[0x2];
    p3 = sc->h[0x3];
    p4 = sc->h[0x4];
    p5 = sc->h[0x5];
    p6 = sc->h[0x6];
    p7 = sc->h[0x7];
    # 初始化u的值为0
    u = 0;
    # 循环6次，每次进行一系列的操作
    for (r = 0; r < 6; r ++) {
        # 定义4个32位的变量
        sph_u32 x0, x1, x2, x3;

        # 对p4、p5、p6、p7分别进行异或操作
        x0 = p4 ^ rk[u ++];
        x1 = p5 ^ rk[u ++];
        x2 = p6 ^ rk[u ++];
        x3 = p7 ^ rk[u ++];
        # 调用AES_ROUND_NOKEY函数
        AES_ROUND_NOKEY(x0, x1, x2, x3);
        # 对x0、x1、x2、x3分别进行异或操作
        x0 ^= rk[u ++];
        x1 ^= rk[u ++];
        x2 ^= rk[u ++];
        x3 ^= rk[u ++];
        # 再次调用AES_ROUND_NOKEY函数
        AES_ROUND_NOKEY(x0, x1, x2, x3);
        # 对x0、x1、x2、x3分别进行异或操作
        x0 ^= rk[u ++];
        x1 ^= rk[u ++];
        x2 ^= rk[u ++];
        x3 ^= rk[u ++];
        # 再次调用AES_ROUND_NOKEY函数
        AES_ROUND_NOKEY(x0, x1, x2, x3);
        # 对p0、p1、p2、p3分别进行异或操作
        p0 ^= x0;
        p1 ^= x1;
        p2 ^= x2;
        p3 ^= x3;

        # 对p0、p1、p2、p3分别进行异或操作
        x0 = p0 ^ rk[u ++];
        x1 = p1 ^ rk[u ++];
        x2 = p2 ^ rk[u ++];
        x3 = p3 ^ rk[u ++];
        # 调用AES_ROUND_NOKEY函数
        AES_ROUND_NOKEY(x0, x1, x2, x3);
        # 对x0、x1、x2、x3分别进行异或操作
        x0 ^= rk[u ++];
        x1 ^= rk[u ++];
        x2 ^= rk[u ++];
        x3 ^= rk[u ++];
        # 再次调用AES_ROUND_NOKEY函数
        AES_ROUND_NOKEY(x0, x1, x2, x3);
        # 对x0、x1、x2、x3分别进行异或操作
        x0 ^= rk[u ++];
        x1 ^= rk[u ++];
        x2 ^= rk[u ++];
        x3 ^= rk[u ++];
        # 再次调用AES_ROUND_NOKEY函数
        AES_ROUND_NOKEY(x0, x1, x2, x3);
        # 对p4、p5、p6、p7分别进行异或操作
        p4 ^= x0;
        p5 ^= x1;
        p6 ^= x2;
        p7 ^= x3;
    }
    # 对sc->h数组中的元素与p0、p1、p2、p3、p4、p5、p6、p7分别进行异或操作
    sc->h[0x0] ^= p0;
    sc->h[0x1] ^= p1;
    sc->h[0x2] ^= p2;
    sc->h[0x3] ^= p3;
    sc->h[0x4] ^= p4;
    sc->h[0x5] ^= p5;
    sc->h[0x6] ^= p6;
    sc->h[0x7] ^= p7;
}
#else

/*
 * This function assumes that "msg" is aligned for 32-bit access.
 */
static void
c256(sph_shavite_small_context *sc, const void *msg)
{
    sph_u32 p0, p1, p2, p3, p4, p5, p6, p7;
    sph_u32 x0, x1, x2, x3;
    sph_u32 rk0, rk1, rk2, rk3, rk4, rk5, rk6, rk7;
    sph_u32 rk8, rk9, rkA, rkB, rkC, rkD, rkE, rkF;

    p0 = sc->h[0x0];  // 从上下文中获取哈希值的第一个部分
    p1 = sc->h[0x1];  // 从上下文中获取哈希值的第二个部分
    p2 = sc->h[0x2];  // 从上下文中获取哈希值的第三个部分
    p3 = sc->h[0x3];  // 从上下文中获取哈希值的第四个部分
    p4 = sc->h[0x4];  // 从上下文中获取哈希值的第五个部分
    p5 = sc->h[0x5];  // 从上下文中获取哈希值的第六个部分
    p6 = sc->h[0x6];  // 从上下文中获取哈希值的第七个部分
    p7 = sc->h[0x7];  // 从上下文中获取哈希值的第八个部分
    /* round 0 */
    rk0 = sph_dec32le_aligned((const unsigned char *)msg +  0);  // 从消息中获取第一个32位块
    x0 = p4 ^ rk0;  // 执行异或操作
    rk1 = sph_dec32le_aligned((const unsigned char *)msg +  4);  // 从消息中获取第二个32位块
    x1 = p5 ^ rk1;  // 执行异或操作
    rk2 = sph_dec32le_aligned((const unsigned char *)msg +  8);  // 从消息中获取第三个32位块
    x2 = p6 ^ rk2;  // 执行异或操作
    rk3 = sph_dec32le_aligned((const unsigned char *)msg + 12);  // 从消息中获取第四个32位块
    x3 = p7 ^ rk3;  // 执行异或操作
    AES_ROUND_NOKEY(x0, x1, x2, x3);  // 执行 AES 轮函数
    rk4 = sph_dec32le_aligned((const unsigned char *)msg + 16);  // 从消息中获取第五个32位块
    x0 ^= rk4;  // 执行异或操作
    rk5 = sph_dec32le_aligned((const unsigned char *)msg + 20);  // 从消息中获取第六个32位块
    x1 ^= rk5;  // 执行异或操作
    rk6 = sph_dec32le_aligned((const unsigned char *)msg + 24);  // 从消息中获取第七个32位块
    x2 ^= rk6;  // 执行异或操作
    rk7 = sph_dec32le_aligned((const unsigned char *)msg + 28);  // 从消息中获取第八个32位块
    x3 ^= rk7;  // 执行异或操作
    AES_ROUND_NOKEY(x0, x1, x2, x3);  // 执行 AES 轮函数
    rk8 = sph_dec32le_aligned((const unsigned char *)msg + 32);  // 从消息中获取第九个32位块
    x0 ^= rk8;  // 执行异或操作
    rk9 = sph_dec32le_aligned((const unsigned char *)msg + 36);  // 从消息中获取第十个32位块
    x1 ^= rk9;  // 执行异或操作
    rkA = sph_dec32le_aligned((const unsigned char *)msg + 40);  // 从消息中获取第十一个32位块
    x2 ^= rkA;  // 执行异或操作
    rkB = sph_dec32le_aligned((const unsigned char *)msg + 44);  // 从消息中获取第十二个32位块
    x3 ^= rkB;  // 执行异或操作
    AES_ROUND_NOKEY(x0, x1, x2, x3);  // 执行 AES 轮函数
    p0 ^= x0;  // 执行异或操作
    p1 ^= x1;  // 执行异或操作
    p2 ^= x2;  // 执行异或操作
    p3 ^= x3;  // 执行异或操作
    /* round 1 */
    rkC = sph_dec32le_aligned((const unsigned char *)msg + 48);  // 从消息中获取第十三个32位块
    x0 = p0 ^ rkC;  // 执行异或操作
    rkD = sph_dec32le_aligned((const unsigned char *)msg + 52);  // 从消息中获取第十四个32位块
    x1 = p1 ^ rkD;  // 执行异或操作
    rkE = sph_dec32le_aligned((const unsigned char *)msg + 56);  // 从消息中获取第十五个32位块
    x2 = p2 ^ rkE;  // 执行异或操作
    rkF = sph_dec32le_aligned((const unsigned char *)msg + 60);  // 从消息中获取第十六个32位块
    x3 = p3 ^ rkF;  // 执行异或操作
    # 执行AES_ROUND_NOKEY函数，对x0, x1, x2, x3进行一轮AES加密操作
    AES_ROUND_NOKEY(x0, x1, x2, x3);
    # 执行KEY_EXPAND_ELT函数，对rk0, rk1, rk2, rk3进行密钥扩展操作
    KEY_EXPAND_ELT(rk0, rk1, rk2, rk3);
    # 对rk0进行异或操作，与rkC和sc->count0异或
    rk0 ^= rkC ^ sc->count0;
    # 对rk1进行异或操作，与rkD和sc->count1的取反异或
    rk1 ^= rkD ^ SPH_T32(~sc->count1);
    # 对rk2进行异或操作，与rkE异或
    rk2 ^= rkE;
    # 对rk3进行异或操作，与rkF异或
    rk3 ^= rkF;
    # 对x0进行异或操作，与rk0异或
    x0 ^= rk0;
    # 对x1进行异或操作，与rk1异或
    x1 ^= rk1;
    # 对x2进行异或操作，与rk2异或
    x2 ^= rk2;
    # 对x3进行异或操作，与rk3异或
    x3 ^= rk3;
    # 执行AES_ROUND_NOKEY函数，对x0, x1, x2, x3进行一轮AES加密操作
    AES_ROUND_NOKEY(x0, x1, x2, x3);
    # 执行KEY_EXPAND_ELT函数，对rk4, rk5, rk6, rk7进行密钥扩展操作
    KEY_EXPAND_ELT(rk4, rk5, rk6, rk7);
    # 对rk4进行异或操作，与rk0异或
    rk4 ^= rk0;
    # 对rk5进行异或操作，与rk1异或
    rk5 ^= rk1;
    # 对rk6进行异或操作，与rk2异或
    rk6 ^= rk2;
    # 对rk7进行异或操作，与rk3异或
    rk7 ^= rk3;
    # 对x0进行异或操作，与rk4异或
    x0 ^= rk4;
    # 对x1进行异或操作，与rk5异或
    x1 ^= rk5;
    # 对x2进行异或操作，与rk6异或
    x2 ^= rk6;
    # 对x3进行异或操作，与rk7异或
    x3 ^= rk7;
    # 执行AES_ROUND_NOKEY函数，对x0, x1, x2, x3进行一轮AES加密操作
    AES_ROUND_NOKEY(x0, x1, x2, x3);
    # 对p4进行异或操作，与x0异或
    p4 ^= x0;
    # 对p5进行异或操作，与x1异或
    p5 ^= x1;
    # 对p6进行异或操作，与x2异或
    p6 ^= x2;
    # 对p7进行异或操作，与x3异或
    p7 ^= x3;
    # round 2
    # 执行KEY_EXPAND_ELT函数，对rk8, rk9, rkA, rkB进行密钥扩展操作
    KEY_EXPAND_ELT(rk8, rk9, rkA, rkB);
    # 对rk8进行异或操作，与rk4异或
    rk8 ^= rk4;
    # 对rk9进行异或操作，与rk5异或
    rk9 ^= rk5;
    # 对rkA进行异或操作，与rk6异或
    rkA ^= rk6;
    # 对rkB进行异或操作，与rk7异或
    rkB ^= rk7;
    # 对x0进行异或操作，与p4异或
    x0 = p4 ^ rk8;
    # 对x1进行异或操作，与p5异或
    x1 = p5 ^ rk9;
    # 对x2进行异或操作，与p6异或
    x2 = p6 ^ rkA;
    # 对x3进行异或操作，与p7异或
    x3 = p7 ^ rkB;
    # 执行AES_ROUND_NOKEY函数，对x0, x1, x2, x3进行一轮AES加密操作
    AES_ROUND_NOKEY(x0, x1, x2, x3);
    # 执行KEY_EXPAND_ELT函数，对rkC, rkD, rkE, rkF进行密钥扩展操作
    KEY_EXPAND_ELT(rkC, rkD, rkE, rkF);
    # 对rkC进行异或操作，与rk8异或
    rkC ^= rk8;
    # 对rkD进行异或操作，与rk9异或
    rkD ^= rk9;
    # 对rkE进行异或操作，与rkA异或
    rkE ^= rkA;
    # 对rkF进行异或操作，与rkB异或
    rkF ^= rkB;
    # 对x0进行异或操作，与rkC异或
    x0 ^= rkC;
    # 对x1进行异或操作，与rkD异或
    x1 ^= rkD;
    # 对x2进行异或操作，与rkE异或
    x2 ^= rkE;
    # 对x3进行异或操作，与rkF异或
    x3 ^= rkF;
    # 执行AES_ROUND_NOKEY函数，对x0, x1, x2, x3进行一轮AES加密操作
    AES_ROUND_NOKEY(x0, x1, x2, x3);
    # 对rk0进行异或操作，与rkD异或
    rk0 ^= rkD;
    # 对x0进行异或操作，与rk0异或
    x0 ^= rk0;
    # 对rk1进行异或操作，与rkE异或
    rk1 ^= rkE;
    # 对x1进行异或操作，与rk1异或
    x1 ^= rk1;
    # 对rk2进行异或操作，与rkF异或
    rk2 ^= rkF;
    # 对x2进行异或操作，与rk2异或
    x2 ^= rk2;
    # 对rk3进行异或操作，与rk0异或
    rk3 ^= rk0;
    # 对x3进行异或操作，与rk3异或
    x3 ^= rk3;
    # 执行AES_ROUND_NOKEY函数，对x0, x1, x2, x3进行一轮AES加密操作
    AES_ROUND_NOKEY(x0, x1, x2, x3);
    # 对p0进行异或操作，与x0异或
    p0 ^= x0;
    # 对p1进行异或操作，与x1异或
    p1 ^= x1;
    # 对p2进行异或操作，与x2异或
    p2 ^= x2;
    # 对p3进行异或操作，与x3异或
    p3 ^= x3;
    # round 3
    # 对rk4进行异或操作，与rk1异或
    rk4 ^= rk1;
    # 对x0进行异或操作，与p0异或
    x0 = p0 ^ rk4;
    # 对rk5进行异或操作，与rk2异或
    rk5 ^= rk2;
    # 对x1进行异或操作，与p1异或
    x1 = p1 ^ rk5;
    # 对rk6进行异或操作，与rk3异或
    rk6 ^= rk3;
    # 对x2进行异或操作，与p2异或
    x2 = p2 ^ rk6;
    # 对rk7进行异或操作，与rk4异或
    rk7 ^= rk4;
    # 对x3进行异或操作，与p3异或
    x3 = p3 ^ rk7;
    # 执行AES_ROUND_NOKEY函数，对x0, x1, x2, x3进行一轮AES加密操作
    AES_ROUND_NOKEY(x0, x1, x2, x3);
    # 对rk8进行异或操作，与rk5异或
    rk8 ^= rk5;
    # 对x0进行异或操作，与rk8异或
    x0 ^= rk8;
    # 对rk9进行异或操作，与rk6异或
    rk9 ^= rk6;
    # 对x1进行异或操作，与rk9异或
    x1 ^= rk9;
    # 对rkA进行异或操作，与rk7异或
    rkA ^= rk7;
    # 对x2进行异或操作，与rkA异或
    x2 ^= rkA;
    # 对rkB进行异或操作，与rk8异或
    rkB ^= rk8;
    # 对x3进行异或操作，与rkB异或
    x3 ^= rkB;
    # 执行AES_ROUND_NOKEY函数，对x0, x1, x2, x3进行一轮AES加密操作
    AES_ROUND_NOKEY(x0, x1, x2, x3);
    # 对rkC进行异或操作，与rk9异或
    rkC ^= rk9;
    # 对x0进行异或操作，与rkC异或
    x0 ^= rkC;
    # 对rkD进行异或操作，与rkA异或
    rkD ^= rkA;
    # 对x1进行异或操作，与rkD异或
    x1 ^= rkD;
    # 对rkE进行异或操作，与rkB异或
    rkE ^= rkB;
    # 对x2进行异或操作，与rkE异或
    x2 ^= rkE;
    # 对rkF进行异或操作，与rkC异或
    rkF ^= rkC;
    # 对x3进行异或操作，与rkF异或
    x3 ^= rkF;
    # 执行AES_ROUND_NOKEY函数，对x0, x1, x2, x3进行一轮AES加密操作
    AES_ROUND_NOKEY(x0, x1, x2, x3);
    # 对p4进行异或操作，与x0异或
    p4 ^= x0;
    # 对p5进行异或操作，与x1异或
    p5 ^= x1;
    # 对p6进行异或操作，与x2异或
    p6 ^= x2;
    # 对p7进行异或操作，与x3异或
    p7 ^= x3;
    # round 4
    # 执行KEY_EXPAND_ELT函数，对rk0, rk1, rk2, rk3进行密钥扩展操作
    KEY_EXPAND_ELT(rk0, rk1, rk2, rk3);
    # 对rk0进行异或操作，与rkC异或
    rk0 ^= rkC;
    # 对rk1进行异或操作，与rkD异或
    rk1 ^= rkD;
    # 对rk2进行异或操作，与rkE异或
    rk2 ^= rkE;
    # 对rk3进行异或操作，与rkF异或
    rk3 ^= rkF;
    # 对x0进行异或操作，与p4异或
    x0 = p4 ^ rk0;
    # 对x1进行异或操作，与p5异或
    x1 = p5 ^ rk1;
    # 对x2进行异或操作，与p6异或
    x2 = p6 ^ rk2;
    # 对x3进行异或操作，与p7异或
    x3 = p7 ^ rk3;
    # 执行AES_ROUND_NOKEY函数，对x0, x1, x2, x3进行一轮AES加密操作
    AES_ROUND_NOKEY(x0, x1, x2, x3);
    # 执行密钥扩展操作，使用 rk4, rk5, rk6, rk7 作为输入
    KEY_EXPAND_ELT(rk4, rk5, rk6, rk7);
    # 对 rk4 进行异或操作
    rk4 ^= rk0;
    # 对 rk5 进行异或操作
    rk5 ^= rk1;
    # 对 rk6 进行异或操作
    rk6 ^= rk2;
    # 对 rk7 进行异或操作
    rk7 ^= rk3;
    # 对 x0 进行异或操作
    x0 ^= rk4;
    # 对 x1 进行异或操作
    x1 ^= rk5;
    # 对 x2 进行异或操作
    x2 ^= rk6;
    # 对 x3 进行异或操作
    x3 ^= rk7;
    # 执行 AES 轮函数操作，不使用密钥
    AES_ROUND_NOKEY(x0, x1, x2, x3);
    # 执行密钥扩展操作，使用 rk8, rk9, rkA, rkB 作为输入
    KEY_EXPAND_ELT(rk8, rk9, rkA, rkB);
    # 对 rk8 进行异或操作
    rk8 ^= rk4;
    # 对 rk9 进行异或操作
    rk9 ^= rk5 ^ sc->count1;
    # 对 rkA 进行异或操作
    rkA ^= rk6 ^ SPH_T32(~sc->count0);
    # 对 rkB 进行异或操作
    rkB ^= rk7;
    # 对 x0 进行异或操作
    x0 ^= rk8;
    # 对 x1 进行异或操作
    x1 ^= rk9;
    # 对 x2 进行异或操作
    x2 ^= rkA;
    # 对 x3 进行异或操作
    x3 ^= rkB;
    # 执行 AES 轮函数操作，不使用密钥
    AES_ROUND_NOKEY(x0, x1, x2, x3);
    # 对 p0 进行异或操作
    p0 ^= x0;
    # 对 p1 进行异或操作
    p1 ^= x1;
    # 对 p2 进行异或操作
    p2 ^= x2;
    # 对 p3 进行异或操作
    p3 ^= x3;
    # 执行密钥扩展操作，使用 rkC, rkD, rkE, rkF 作为输入
    KEY_EXPAND_ELT(rkC, rkD, rkE, rkF);
    # 对 rkC 进行异或操作
    rkC ^= rk8;
    # 对 rkD 进行异或操作
    rkD ^= rk9;
    # 对 rkE 进行异或操作
    rkE ^= rkA;
    # 对 rkF 进行异或操作
    rkF ^= rkB;
    # 对 x0 进行赋值操作
    x0 = p0 ^ rkC;
    # 对 x1 进行赋值操作
    x1 = p1 ^ rkD;
    # 对 x2 进行赋值操作
    x2 = p2 ^ rkE;
    # 对 x3 进行赋值操作
    x3 = p3 ^ rkF;
    # 执行 AES 轮函数操作，不使用密钥
    AES_ROUND_NOKEY(x0, x1, x2, x3);
    # 对 rk0 进行异或操作
    rk0 ^= rkD;
    # 对 x0 进行异或操作
    x0 ^= rk0;
    # 对 rk1 进行异或操作
    rk1 ^= rkE;
    # 对 x1 进行异或操作
    x1 ^= rk1;
    # 对 rk2 进行异或操作
    rk2 ^= rkF;
    # 对 x2 进行异或操作
    x2 ^= rk2;
    # 对 rk3 进行异或操作
    rk3 ^= rk0;
    # 对 x3 进行异或操作
    x3 ^= rk3;
    # 执行 AES 轮函数操作，不使用密钥
    AES_ROUND_NOKEY(x0, x1, x2, x3);
    # 对 rk4 进行异或操作
    rk4 ^= rk1;
    # 对 x0 进行异或操作
    x0 ^= rk4;
    # 对 rk5 进行异或操作
    rk5 ^= rk2;
    # 对 x1 进行异或操作
    x1 ^= rk5;
    # 对 rk6 进行异或操作
    rk6 ^= rk3;
    # 对 x2 进行异或操作
    x2 ^= rk6;
    # 对 rk7 进行异或操作
    rk7 ^= rk4;
    # 对 x3 进行异或操作
    x3 ^= rk7;
    # 执行 AES 轮函数操作，不使用密钥
    AES_ROUND_NOKEY(x0, x1, x2, x3);
    # 对 p4 进行异或操作
    p4 ^= x0;
    # 对 p5 进行异或操作
    p5 ^= x1;
    # 对 p6 进行异或操作
    p6 ^= x2;
    # 对 p7 进行异或操作
    p7 ^= x3;
    # 执行密钥扩展操作，使用 rk8, rk9, rkA, rkB 作为输入
    KEY_EXPAND_ELT(rk8, rk9, rkA, rkB);
    # 对 rk8 进行异或操作
    rk8 ^= rk5;
    # 对 x0 进行赋值操作
    x0 = p4 ^ rk8;
    # 对 rk9 进行异或操作
    rk9 ^= rk6;
    # 对 x1 进行赋值操作
    x1 = p5 ^ rk9;
    # 对 rkA 进行异或操作
    rkA ^= rk7;
    # 对 x2 进行赋值操作
    x2 = p6 ^ rkA;
    # 对 rkB 进行异或操作
    rkB ^= rk8;
    # 对 x3 进行赋值操作
    x3 = p7 ^ rkB;
    # 执行 AES 轮函数操作，不使用密钥
    AES_ROUND_NOKEY(x0, x1, x2, x3);
    # 对 rkC 进行异或操作
    rkC ^= rk9;
    # 对 x0 进行异或操作
    x0 ^= rkC;
    # 对 rkD 进行异或操作
    rkD ^= rkA;
    # 对 x1 进行异或操作
    x1 ^= rkD;
    # 对 rkE 进行异或操作
    rkE ^= rkB;
    # 对 x2 进行异或操作
    x2 ^= rkE;
    # 对 rkF 进行异或操作
    rkF ^= rkC;
    # 对 x3 进行异或操作
    x3 ^= rkF;
    # 执行 AES 轮函数操作，不使用密钥
    AES_ROUND_NOKEY(x0, x1, x2, x3);
    # 执行密钥扩展操作，使用 rk0, rk1, rk2, rk3 作为输入
    KEY_EXPAND_ELT(rk0, rk1, rk2, rk3);
    # 对 rk0 进行异或操作
    rk0 ^= rkC;
    # 对 rk1 进行异或操作
    rk1 ^= rkD;
    # 对 rk2 进行异或操作
    rk2 ^= rkE;
    # 对 rk3 进行异或操作
    rk3 ^= rkF;
    # 对 x0 进行异或操作
    x0 ^= rk0;
    # 对 x1 进行异或操作
    x1 ^= rk1;
    # 对 x2 进行异或操作
    x2 ^= rk2;
    # 对 x3 进行异或操作
    x3 ^= rk3;
    # 执行 AES 轮函数操作，不使用密钥
    AES_ROUND_NOKEY(x0, x1, x2, x3);
    # 对 p0 进行异或操作
    p0 ^= x0;
    # 对 p1 进行异或操作
    p1 ^= x1;
    # 对 p2 进行异或操作
    p2 ^= x2;
    # 对 p3 进行异或操作
    p3 ^= x3;
    # 执行密钥扩展操作，使用 rk4, rk5, rk6, rk7 作为输入
    KEY_EXPAND_ELT(rk4, rk5, rk6, rk7);
    # 对 rk4 进行异或操作
    rk4 ^= rk0;
    # 对 rk5 进行异或操作
    rk5 ^= rk1;
    # 对 rk6 进行异或操作
    rk6 ^= rk2 ^ sc->count1;
    # 对 rk7 进行异或操作
    rk7 ^= rk3 ^ SPH_T32(~sc->count0);
    # 对 x0 进行赋值操作
    x0 = p0 ^ rk4;
    # 对 x1 进行赋值操作
    x1 = p1 ^ rk5;
    # 对 x2 进行赋值操作
    x2 = p2 ^ rk6;
    # 对 x3 进行赋值操作
    x3 = p3 ^ rk7;
    # 执行 AES 轮函数操作，不使用密钥
    AES_ROUND_NOKEY(x0, x1, x2, x3);
    # round 1
    # 扩展密钥
    KEY_EXPAND_ELT(rk8, rk9, rkA, rkB);
    # 异或操作
    rk8 ^= rk4;
    rk9 ^= rk5;
    rkA ^= rk6;
    rkB ^= rk7;
    # 异或操作
    x0 ^= rk8;
    x1 ^= rk9;
    x2 ^= rkA;
    x3 ^= rkB;
    # AES轮函数
    AES_ROUND_NOKEY(x0, x1, x2, x3);
    
    # round 2
    # 扩展密钥
    KEY_EXPAND_ELT(rkC, rkD, rkE, rkF);
    # 异或操作
    rkC ^= rk8;
    rkD ^= rk9;
    rkE ^= rkA;
    rkF ^= rkB;
    # 异或操作
    x0 ^= rkC;
    x1 ^= rkD;
    x2 ^= rkE;
    x3 ^= rkF;
    # AES轮函数
    AES_ROUND_NOKEY(x0, x1, x2, x3);
    
    # round 3
    # 异或操作
    p4 ^= x0;
    p5 ^= x1;
    p6 ^= x2;
    p7 ^= x3;
    # round 4
    # 扩展密钥
    rk0 ^= rkD;
    # 异或操作
    x0 = p4 ^ rk0;
    rk1 ^= rkE;
    # 异或操作
    x1 = p5 ^ rk1;
    rk2 ^= rkF;
    # 异或操作
    x2 = p6 ^ rk2;
    rk3 ^= rk0;
    # 异或操作
    x3 = p7 ^ rk3;
    # AES轮函数
    AES_ROUND_NOKEY(x0, x1, x2, x3);
    
    # round 5
    # 异或操作
    rk4 ^= rk1;
    x0 ^= rk4;
    rk5 ^= rk2;
    x1 ^= rk5;
    rk6 ^= rk3;
    x2 ^= rk6;
    rk7 ^= rk4;
    x3 ^= rk7;
    # AES轮函数
    AES_ROUND_NOKEY(x0, x1, x2, x3);
    
    # round 6
    # 异或操作
    rk8 ^= rk5;
    x0 ^= rk8;
    rk9 ^= rk6;
    x1 ^= rk9;
    rkA ^= rk7;
    x2 ^= rkA;
    rkB ^= rk8;
    x3 ^= rkB;
    # AES轮函数
    AES_ROUND_NOKEY(x0, x1, x2, x3);
    
    # round 7
    # 异或操作
    p0 ^= x0;
    p1 ^= x1;
    p2 ^= x2;
    p3 ^= x3;
    # round 8
    # 扩展密钥
    rkC ^= rk9;
    # 异或操作
    x0 = p0 ^ rkC;
    rkD ^= rkA;
    # 异或操作
    x1 = p1 ^ rkD;
    rkE ^= rkB;
    # 异或操作
    x2 = p2 ^ rkE;
    rkF ^= rkC;
    # 异或操作
    x3 = p3 ^ rkF;
    # AES轮函数
    AES_ROUND_NOKEY(x0, x1, x2, x3);
    
    # round 9
    # 扩展密钥
    KEY_EXPAND_ELT(rk0, rk1, rk2, rk3);
    # 异或操作
    rk0 ^= rkC;
    rk1 ^= rkD;
    rk2 ^= rkE;
    rk3 ^= rkF;
    # 异或操作
    x0 ^= rk0;
    x1 ^= rk1;
    x2 ^= rk2;
    x3 ^= rk3;
    # AES轮函数
    AES_ROUND_NOKEY(x0, x1, x2, x3);
    
    # round 10
    # 扩展密钥
    KEY_EXPAND_ELT(rk4, rk5, rk6, rk7);
    # 异或操作
    rk4 ^= rk0;
    rk5 ^= rk1;
    rk6 ^= rk2;
    rk7 ^= rk3;
    # 异或操作
    x0 ^= rk4;
    x1 ^= rk5;
    x2 ^= rk6;
    x3 ^= rk7;
    # AES轮函数
    AES_ROUND_NOKEY(x0, x1, x2, x3);
    
    # round 11
    # 异或操作
    p4 ^= x0;
    p5 ^= x1;
    p6 ^= x2;
    p7 ^= x3;
    # round 12
    # 扩展密钥
    KEY_EXPAND_ELT(rk8, rk9, rkA, rkB);
    # 异或操作
    rk8 ^= rk4;
    rk9 ^= rk5;
    rkA ^= rk6;
    rkB ^= rk7;
    # 异或操作
    x0 = p4 ^ rk8;
    x1 = p5 ^ rk9;
    x2 = p6 ^ rkA;
    x3 = p7 ^ rkB;
    # AES轮函数
    AES_ROUND_NOKEY(x0, x1, x2, x3);
    
    # round 13
    # 扩展密钥
    KEY_EXPAND_ELT(rkC, rkD, rkE, rkF);
    # 异或操作
    rkC ^= rk8 ^ sc->count0;
    # 对 rkD 和 rk9 进行按位异或操作
    rkD ^= rk9;
    # 对 rkE 和 rkA 进行按位异或操作
    rkE ^= rkA;
    # 对 rkF、rkB 和 sc->count1 的按位取反结果进行按位异或操作
    rkF ^= rkB ^ SPH_T32(~sc->count1);
    # 对 x0 和 rkC 进行按位异或操作
    x0 ^= rkC;
    # 对 x1 和 rkD 进行按位异或操作
    x1 ^= rkD;
    # 对 x2 和 rkE 进行按位异或操作
    x2 ^= rkE;
    # 对 x3 和 rkF 进行按位异或操作
    x3 ^= rkF;
    # 调用 AES_ROUND_NOKEY 函数对 x0、x1、x2、x3 进行处理
    AES_ROUND_NOKEY(x0, x1, x2, x3);
    # 对 rk0 和 rkD 进行按位异或操作
    rk0 ^= rkD;
    # 对 x0 和 rk0 进行按位异或操作
    x0 ^= rk0;
    # 对 rk1 和 rkE 进行按位异或操作
    rk1 ^= rkE;
    # 对 x1 和 rk1 进行按位异或操作
    x1 ^= rk1;
    # 对 rk2 和 rkF 进行按位异或操作
    rk2 ^= rkF;
    # 对 x2 和 rk2 进行按位异或操作
    x2 ^= rk2;
    # 对 rk3 和 rk0 进行按位异或操作
    rk3 ^= rk0;
    # 对 x3 和 rk3 进行按位异或操作
    x3 ^= rk3;
    # 调用 AES_ROUND_NOKEY 函数对 x0、x1、x2、x3 进行处理
    AES_ROUND_NOKEY(x0, x1, x2, x3);
    # 对 p0 和 x0 进行按位异或操作
    p0 ^= x0;
    # 对 p1 和 x1 进行按位异或操作
    p1 ^= x1;
    # 对 p2 和 x2 进行按位异或操作
    p2 ^= x2;
    # 对 p3 和 x3 进行按位异或操作
    p3 ^= x3;
    # round 11
    # 对 rk4 和 rk1 进行按位异或操作
    rk4 ^= rk1;
    # 对 x0 和 p0 进行按位异或操作
    x0 = p0 ^ rk4;
    # 对 rk5 和 rk2 进行按位异或操作
    rk5 ^= rk2;
    # 对 x1 和 p1 进行按位异或操作
    x1 = p1 ^ rk5;
    # 对 rk6 和 rk3 进行按位异或操作
    rk6 ^= rk3;
    # 对 x2 和 p2 进行按位异或操作
    x2 = p2 ^ rk6;
    # 对 rk7 和 rk4 进行按位异或操作
    rk7 ^= rk4;
    # 对 x3 和 p3 进行按位异或操作
    x3 = p3 ^ rk7;
    # 调用 AES_ROUND_NOKEY 函数对 x0、x1、x2、x3 进行处理
    AES_ROUND_NOKEY(x0, x1, x2, x3);
    # 对 rk8 和 rk5 进行按位异或操作
    rk8 ^= rk5;
    # 对 x0 和 rk8 进行按位异或操作
    x0 ^= rk8;
    # 对 rk9 和 rk6 进行按位异或操作
    rk9 ^= rk6;
    # 对 x1 和 rk9 进行按位异或操作
    x1 ^= rk9;
    # 对 rkA 和 rk7 进行按位异或操作
    rkA ^= rk7;
    # 对 x2 和 rkA 进行按位异或操作
    x2 ^= rkA;
    # 对 rkB 和 rk8 进行按位异或操作
    rkB ^= rk8;
    # 对 x3 和 rkB 进行按位异或操作
    x3 ^= rkB;
    # 调用 AES_ROUND_NOKEY 函数对 x0、x1、x2、x3 进行处理
    AES_ROUND_NOKEY(x0, x1, x2, x3);
    # 对 rkC 和 rk9 进行按位异或操作
    rkC ^= rk9;
    # 对 x0 和 rkC 进行按位异或操作
    x0 ^= rkC;
    # 对 rkD 和 rkA 进行按位异或操作
    rkD ^= rkA;
    # 对 x1 和 rkD 进行按位异或操作
    x1 ^= rkD;
    # 对 rkE 和 rkB 进行按位异或操作
    rkE ^= rkB;
    # 对 x2 和 rkE 进行按位异或操作
    x2 ^= rkE;
    # 对 rkF 和 rkC 进行按位异或操作
    rkF ^= rkC;
    # 对 x3 和 rkF 进行按位异或操作
    x3 ^= rkF;
    # 调用 AES_ROUND_NOKEY 函数对 x0、x1、x2、x3 进行处理
    AES_ROUND_NOKEY(x0, x1, x2, x3);
    # 对 p4 和 x0 进行按位异或操作
    p4 ^= x0;
    # 对 p5 和 x1 进行按位异或操作
    p5 ^= x1;
    # 对 p6 和 x2 进行按位异或操作
    p6 ^= x2;
    # 对 p7 和 x3 进行按位异或操作
    p7 ^= x3;
    # 对 sc->h[0x0] 和 p0 进行按位异或操作
    sc->h[0x0] ^= p0;
    # 对 sc->h[0x1] 和 p1 进行按位异或操作
    sc->h[0x1] ^= p1;
    # 对 sc->h[0x2] 和 p2 进行按位异或操作
    sc->h[0x2] ^= p2;
    # 对 sc->h[0x3] 和 p3 进行按位异或操作
    sc->h[0x3] ^= p3;
    # 对 sc->h[0x4] 和 p4 进行按位异或操作
    sc->h[0x4] ^= p4;
    # 对 sc->h[0x5] 和 p5 进行按位异或操作
    sc->h[0x5] ^= p5;
    # 对 sc->h[0x6] 和 p6 进行按位异或操作
    sc->h[0x6] ^= p6;
    # 对 sc->h[0x7] 和 p7 进行按位异或操作
    sc->h[0x7] ^= p7;
#endif

#if SPH_SMALL_FOOTPRINT_SHAVITE

/*
 * This function assumes that "msg" is aligned for 32-bit access.
 */
// 定义一个静态函数c512，接受一个sph_shavite_big_context类型的指针sc和一个void类型的指针msg作为参数
static void
c512(sph_shavite_big_context *sc, const void *msg)
{
    // 定义16个32位无符号整数变量
    sph_u32 p0, p1, p2, p3, p4, p5, p6, p7;
    sph_u32 p8, p9, pA, pB, pC, pD, pE, pF;
    // 定义一个包含448个32位无符号整数的数组
    sph_u32 rk[448];
    // 定义一个size_t类型的变量u
    size_t u;
    // 定义两个int类型的变量r和s
    int r, s;

    // 如果是小端字节序，使用memcpy函数将msg的前128字节拷贝到rk数组中
#if SPH_LITTLE_ENDIAN
    memcpy(rk, msg, 128);
// 如果不是小端字节序
#else
    // 使用循环将msg中的数据按照小端字节序拷贝到rk数组中
    for (u = 0; u < 32; u += 4) {
        rk[u + 0] = sph_dec32le_aligned(
            (const unsigned char *)msg + (u << 2) +  0);
        rk[u + 1] = sph_dec32le_aligned(
            (const unsigned char *)msg + (u << 2) +  4);
        rk[u + 2] = sph_dec32le_aligned(
            (const unsigned char *)msg + (u << 2) +  8);
        rk[u + 3] = sph_dec32le_aligned(
            (const unsigned char *)msg + (u << 2) + 12);
    }
#endif
    // 初始化变量u为32
    u = 32;
    # 无限循环，直到满足条件跳出循环
    for (;;) {
        # 遍历 0 到 3 的值
        for (s = 0; s < 4; s ++) {
            # 定义并初始化变量 x0, x1, x2, x3
            x0 = rk[u - 31];
            x1 = rk[u - 30];
            x2 = rk[u - 29];
            x3 = rk[u - 32];
            # 调用 AES_ROUND_NOKEY 函数对 x0, x1, x2, x3 进行处理
            AES_ROUND_NOKEY(x0, x1, x2, x3);
            # 计算并赋值 rk[u + 0], rk[u + 1], rk[u + 2], rk[u + 3]
            rk[u + 0] = x0 ^ rk[u - 4];
            rk[u + 1] = x1 ^ rk[u - 3];
            rk[u + 2] = x2 ^ rk[u - 2];
            rk[u + 3] = x3 ^ rk[u - 1];
            # 判断 u 的值是否等于 32
            if (u == 32) {
                # 对 rk[32], rk[33], rk[34], rk[35] 进行异或操作
                rk[ 32] ^= sc->count0;
                rk[ 33] ^= sc->count1;
                rk[ 34] ^= sc->count2;
                rk[ 35] ^= SPH_T32(~sc->count3);
            # 判断 u 的值是否等于 440
            } else if (u == 440) {
                # 对 rk[440], rk[441], rk[442], rk[443] 进行异或操作
                rk[440] ^= sc->count1;
                rk[441] ^= sc->count0;
                rk[442] ^= sc->count3;
                rk[443] ^= SPH_T32(~sc->count2);
            }
            # u 增加 4
            u += 4;
    
            # 定义并初始化变量 x0, x1, x2, x3
            x0 = rk[u - 31];
            x1 = rk[u - 30];
            x2 = rk[u - 29];
            x3 = rk[u - 32];
            # 调用 AES_ROUND_NOKEY 函数对 x0, x1, x2, x3 进行处理
            AES_ROUND_NOKEY(x0, x1, x2, x3);
            # 计算并赋值 rk[u + 0], rk[u + 1], rk[u + 2], rk[u + 3]
            rk[u + 0] = x0 ^ rk[u - 4];
            rk[u + 1] = x1 ^ rk[u - 3];
            rk[u + 2] = x2 ^ rk[u - 2];
            rk[u + 3] = x3 ^ rk[u - 1];
            # 判断 u 的值是否等于 164
            if (u == 164) {
                # 对 rk[164], rk[165], rk[166], rk[167] 进行异或操作
                rk[164] ^= sc->count3;
                rk[165] ^= sc->count2;
                rk[166] ^= sc->count1;
                rk[167] ^= SPH_T32(~sc->count0);
            # 判断 u 的值是否等于 316
            } else if (u == 316) {
                # 对 rk[316], rk[317], rk[318], rk[319] 进行异或操作
                rk[316] ^= sc->count2;
                rk[317] ^= sc->count3;
                rk[318] ^= sc->count0;
                rk[319] ^= SPH_T32(~sc->count1);
            }
            # u 增加 4
            u += 4;
        }
        # 判断 u 的值是否等于 448，如果是则跳出循环
        if (u == 448)
            break;
        # 遍历 0 到 7 的值
        for (s = 0; s < 8; s ++) {
            # 计算并赋值 rk[u + 0], rk[u + 1], rk[u + 2], rk[u + 3]
            rk[u + 0] = rk[u - 32] ^ rk[u - 7];
            rk[u + 1] = rk[u - 31] ^ rk[u - 6];
            rk[u + 2] = rk[u - 30] ^ rk[u - 5];
            rk[u + 3] = rk[u - 29] ^ rk[u - 4];
            # u 增加 4
            u += 4;
        }
    }
    
    # 将 sc->h[0x0], sc->h[0x1], sc->h[0x2], sc->h[0x3] 赋值给 p0, p1, p2, p3
    p0 = sc->h[0x0];
    p1 = sc->h[0x1];
    p2 = sc->h[0x2];
    p3 = sc->h[0x3];
    # 从 sc 对象中获取索引为 0x4 的元素，赋值给变量 p4
    p4 = sc->h[0x4];
    # 从 sc 对象中获取索引为 0x5 的元素，赋值给变量 p5
    p5 = sc->h[0x5];
    # 从 sc 对象中获取索引为 0x6 的元素，赋值给变量 p6
    p6 = sc->h[0x6];
    # 从 sc 对象中获取索引为 0x7 的元素，赋值给变量 p7
    p7 = sc->h[0x7];
    # 从 sc 对象中获取索引为 0x8 的元素，赋值给变量 p8
    p8 = sc->h[0x8];
    # 从 sc 对象中获取索引为 0x9 的元素，赋值给变量 p9
    p9 = sc->h[0x9];
    # 从 sc 对象中获取索引为 0xA 的元素，赋值给变量 pA
    pA = sc->h[0xA];
    # 从 sc 对象中获取索引为 0xB 的元素，赋值给变量 pB
    pB = sc->h[0xB];
    # 从 sc 对象中获取索引为 0xC 的元素，赋值给变量 pC
    pC = sc->h[0xC];
    # 从 sc 对象中获取索引为 0xD 的元素，赋值给变量 pD
    pD = sc->h[0xD];
    # 从 sc 对象中获取索引为 0xE 的元素，赋值给变量 pE
    pE = sc->h[0xE];
    # 从 sc 对象中获取索引为 0xF 的元素，赋值给变量 pF
    pF = sc->h[0xF];
    # 初始化变量 u 为 0
    u = 0;
    # 循环变量 r 从 0 到 13
    for (r = 0; r < 14; r ++) {
#define C512_ELT(l0, l1, l2, l3, r0, r1, r2, r3)   do { \  # 定义一个宏，用于执行一系列的操作
        sph_u32 x0, x1, x2, x3; \  # 声明四个32位无符号整数变量
        x0 = r0 ^ rk[u ++]; \  # 对x0进行异或操作
        x1 = r1 ^ rk[u ++]; \  # 对x1进行异或操作
        x2 = r2 ^ rk[u ++]; \  # 对x2进行异或操作
        x3 = r3 ^ rk[u ++]; \  # 对x3进行异或操作
        AES_ROUND_NOKEY(x0, x1, x2, x3); \  # 调用AES_ROUND_NOKEY函数
        x0 ^= rk[u ++]; \  # 对x0进行异或操作
        x1 ^= rk[u ++]; \  # 对x1进行异或操作
        x2 ^= rk[u ++]; \  # 对x2进行异或操作
        x3 ^= rk[u ++]; \  # 对x3进行异或操作
        AES_ROUND_NOKEY(x0, x1, x2, x3); \  # 调用AES_ROUND_NOKEY函数
        x0 ^= rk[u ++]; \  # 对x0进行异或操作
        x1 ^= rk[u ++]; \  # 对x1进行异或操作
        x2 ^= rk[u ++]; \  # 对x2进行异或操作
        x3 ^= rk[u ++]; \  # 对x3进行异或操作
        AES_ROUND_NOKEY(x0, x1, x2, x3); \  # 调用AES_ROUND_NOKEY函数
        x0 ^= rk[u ++]; \  # 对x0进行异或操作
        x1 ^= rk[u ++]; \  # 对x1进行异或操作
        x2 ^= rk[u ++]; \  # 对x2进行异或操作
        x3 ^= rk[u ++]; \  # 对x3进行异或操作
        l0 ^= x0; \  # 对l0进行异或操作
        l1 ^= x1; \  # 对l1进行异或操作
        l2 ^= x2; \  # 对l2进行异或操作
        l3 ^= x3; \  # 对l3进行异或操作
    } while (0)  # 宏定义结束

#define WROT(a, b, c, d)   do { \  # 定义一个宏，用于执行一系列的操作
        sph_u32 t = d; \  # 声明一个32位无符号整数变量，并赋值为d
        d = c; \  # 将c的值赋给d
        c = b; \  # 将b的值赋给c
        b = a; \  # 将a的值赋给b
        a = t; \  # 将t的值赋给a
    } while (0)  # 宏定义结束

        C512_ELT(p0, p1, p2, p3, p4, p5, p6, p7);  # 调用C512_ELT宏
        C512_ELT(p8, p9, pA, pB, pC, pD, pE, pF);  # 调用C512_ELT宏

        WROT(p0, p4, p8, pC);  # 调用WROT宏
        WROT(p1, p5, p9, pD);  # 调用WROT宏
        WROT(p2, p6, pA, pE);  # 调用WROT宏
        WROT(p3, p7, pB, pF);  # 调用WROT宏

#undef C512_ELT  # 取消C512_ELT宏的定义
#undef WROT  # 取消WROT宏的定义
    }  # 结束do-while循环
    sc->h[0x0] ^= p0;  # 对sc->h[0x0]进行异或操作
    sc->h[0x1] ^= p1;  # 对sc->h[0x1]进行异或操作
    sc->h[0x2] ^= p2;  # 对sc->h[0x2]进行异或操作
    sc->h[0x3] ^= p3;  # 对sc->h[0x3]进行异或操作
    sc->h[0x4] ^= p4;  # 对sc->h[0x4]进行异或操作
    sc->h[0x5] ^= p5;  # 对sc->h[0x5]进行异或操作
    sc->h[0x6] ^= p6;  # 对sc->h[0x6]进行异或操作
    sc->h[0x7] ^= p7;  # 对sc->h[0x7]进行异或操作
    sc->h[0x8] ^= p8;  # 对sc->h[0x8]进行异或操作
    sc->h[0x9] ^= p9;  # 对sc->h[0x9]进行异或操作
    sc->h[0xA] ^= pA;  # 对sc->h[0xA]进行异或操作
    sc->h[0xB] ^= pB;  # 对sc->h[0xB]进行异或操作
    sc->h[0xC] ^= pC;  # 对sc->h[0xC]进行异或操作
    sc->h[0xD] ^= pD;  # 对sc->h[0xD]进行异或操作
    sc->h[0xE] ^= pE;  # 对sc->h[0xE]进行异或操作
    sc->h[0xF] ^= pF;  # 对sc->h[0xF]进行异或操作
}

#else

/*
 * This function assumes that "msg" is aligned for 32-bit access.
 */
static void
c512(sph_shavite_big_context *sc, const void *msg)
{
    sph_u32 p0, p1, p2, p3, p4, p5, p6, p7;  # 声明16个32位无符号整数变量
    sph_u32 p8, p9, pA, pB, pC, pD, pE, pF;  # 声明16个32位无符号整数变量
    sph_u32 x0, x1, x2, x3;  # 声明四个32位无符号整数变量
    sph_u32 rk00, rk01, rk02, rk03, rk04, rk05, rk06, rk07;  # 声明8个32位无符号整数变量
    sph_u32 rk08, rk09, rk0A, rk0B, rk0C, rk0D, rk0E, rk0F;  # 声明8个32位无符号整数变量
    sph_u32 rk10, rk11, rk12, rk13, rk14, rk15, rk16, rk17;  # 声明8个32位无符号整数变量
    # 定义8个32位无符号整数变量
    sph_u32 rk18, rk19, rk1A, rk1B, rk1C, rk1D, rk1E, rk1F;
    # 定义一个整型变量r
    int r;

    # 从sc->h数组中获取对应下标的值，赋给p0到pF变量
    p0 = sc->h[0x0];
    p1 = sc->h[0x1];
    p2 = sc->h[0x2];
    p3 = sc->h[0x3];
    p4 = sc->h[0x4];
    p5 = sc->h[0x5];
    p6 = sc->h[0x6];
    p7 = sc->h[0x7];
    p8 = sc->h[0x8];
    p9 = sc->h[0x9];
    pA = sc->h[0xA];
    pB = sc->h[0xB];
    pC = sc->h[0xC];
    pD = sc->h[0xD];
    pE = sc->h[0xE];
    pF = sc->h[0xF];
    # 第0轮
    # 从msg中读取4个字节，转换为32位小端整数，赋给rk00到rk0F变量
    rk00 = sph_dec32le_aligned((const unsigned char *)msg +   0);
    x0 = p4 ^ rk00;
    rk01 = sph_dec32le_aligned((const unsigned char *)msg +   4);
    x1 = p5 ^ rk01;
    rk02 = sph_dec32le_aligned((const unsigned char *)msg +   8);
    x2 = p6 ^ rk02;
    rk03 = sph_dec32le_aligned((const unsigned char *)msg +  12);
    x3 = p7 ^ rk03;
    AES_ROUND_NOKEY(x0, x1, x2, x3);
    rk04 = sph_dec32le_aligned((const unsigned char *)msg +  16);
    x0 ^= rk04;
    rk05 = sph_dec32le_aligned((const unsigned char *)msg +  20);
    x1 ^= rk05;
    rk06 = sph_dec32le_aligned((const unsigned char *)msg +  24);
    x2 ^= rk06;
    rk07 = sph_dec32le_aligned((const unsigned char *)msg +  28);
    x3 ^= rk07;
    AES_ROUND_NOKEY(x0, x1, x2, x3);
    rk08 = sph_dec32le_aligned((const unsigned char *)msg +  32);
    x0 ^= rk08;
    rk09 = sph_dec32le_aligned((const unsigned char *)msg +  36);
    x1 ^= rk09;
    rk0A = sph_dec32le_aligned((const unsigned char *)msg +  40);
    x2 ^= rk0A;
    rk0B = sph_dec32le_aligned((const unsigned char *)msg +  44);
    x3 ^= rk0B;
    AES_ROUND_NOKEY(x0, x1, x2, x3);
    rk0C = sph_dec32le_aligned((const unsigned char *)msg +  48);
    x0 ^= rk0C;
    rk0D = sph_dec32le_aligned((const unsigned char *)msg +  52);
    x1 ^= rk0D;
    rk0E = sph_dec32le_aligned((const unsigned char *)msg +  56);
    x2 ^= rk0E;
    rk0F = sph_dec32le_aligned((const unsigned char *)msg +  60);
    x3 ^= rk0F;
    AES_ROUND_NOKEY(x0, x1, x2, x3);
    p0 ^= x0;
    p1 ^= x1;
    p2 ^= x2;
    p3 ^= x3;
    # 从消息中提取第10个32位数据，并按照小端对齐方式解析
    rk10 = sph_dec32le_aligned((const unsigned char *)msg +  64);
    # 计算 x0
    x0 = pC ^ rk10;
    # 从消息中提取第11个32位数据，并按照小端对齐方式解析
    rk11 = sph_dec32le_aligned((const unsigned char *)msg +  68);
    # 计算 x1
    x1 = pD ^ rk11;
    # 从消息中提取第12个32位数据，并按照小端对齐方式解析
    rk12 = sph_dec32le_aligned((const unsigned char *)msg +  72);
    # 计算 x2
    x2 = pE ^ rk12;
    # 从消息中提取第13个32位数据，并按照小端对齐方式解析
    rk13 = sph_dec32le_aligned((const unsigned char *)msg +  76);
    # 计算 x3
    x3 = pF ^ rk13;
    # 执行 AES_ROUND_NOKEY 操作
    AES_ROUND_NOKEY(x0, x1, x2, x3);
    # 从消息中提取第14个32位数据，并按照小端对齐方式解析
    rk14 = sph_dec32le_aligned((const unsigned char *)msg +  80);
    # 更新 x0
    x0 ^= rk14;
    # 从消息中提取第15个32位数据，并按照小端对齐方式解析
    rk15 = sph_dec32le_aligned((const unsigned char *)msg +  84);
    # 更新 x1
    x1 ^= rk15;
    # 从消息中提取第16个32位数据，并按照小端对齐方式解析
    rk16 = sph_dec32le_aligned((const unsigned char *)msg +  88);
    # 更新 x2
    x2 ^= rk16;
    # 从消息中提取第17个32位数据，并按照小端对齐方式解析
    rk17 = sph_dec32le_aligned((const unsigned char *)msg +  92);
    # 更新 x3
    x3 ^= rk17;
    # 执行 AES_ROUND_NOKEY 操作
    AES_ROUND_NOKEY(x0, x1, x2, x3);
    # 从消息中提取第18个32位数据，并按照小端对齐方式解析
    rk18 = sph_dec32le_aligned((const unsigned char *)msg +  96);
    # 更新 x0
    x0 ^= rk18;
    # 从消息中提取第19个32位数据，并按照小端对齐方式解析
    rk19 = sph_dec32le_aligned((const unsigned char *)msg + 100);
    # 更新 x1
    x1 ^= rk19;
    # 从消息中提取第1A个32位数据，并按照小端对齐方式解析
    rk1A = sph_dec32le_aligned((const unsigned char *)msg + 104);
    # 更新 x2
    x2 ^= rk1A;
    # 从消息中提取第1B个32位数据，并按照小端对齐方式解析
    rk1B = sph_dec32le_aligned((const unsigned char *)msg + 108);
    # 更新 x3
    x3 ^= rk1B;
    # 执行 AES_ROUND_NOKEY 操作
    AES_ROUND_NOKEY(x0, x1, x2, x3);
    # 更新 p8
    p8 ^= x0;
    # 更新 p9
    p9 ^= x1;
    # 更新 pA
    pA ^= x2;
    # 更新 pB
    pB ^= x3;

    }
    /* round 13 */
    # 执行 KEY_EXPAND_ELT 操作
    KEY_EXPAND_ELT(rk00, rk01, rk02, rk03);
    # 更新 rk00
    rk00 ^= rk1C;
    # 更新 rk01
    rk01 ^= rk1D;
    # 更新 rk02
    rk02 ^= rk1E;
    # 更新 rk03
    rk03 ^= rk1F;
    # 计算 x0
    x0 = p0 ^ rk00;
    # 计算 x1
    x1 = p1 ^ rk01;
    # 计算 x2
    x2 = p2 ^ rk02;
    # 计算 x3
    x3 = p3 ^ rk03;
    # 执行 AES_ROUND_NOKEY 操作
    AES_ROUND_NOKEY(x0, x1, x2, x3);
    # 执行 KEY_EXPAND_ELT 操作
    KEY_EXPAND_ELT(rk04, rk05, rk06, rk07);
    # 更新 rk04
    rk04 ^= rk00;
    # 更新 rk05
    rk05 ^= rk01;
    # 更新 rk06
    rk06 ^= rk02;
    # 更新 rk07
    rk07 ^= rk03;
    # 更新 x0
    x0 ^= rk04;
    # 更新 x1
    x1 ^= rk05;
    # 更新 x2
    x2 ^= rk06;
    # 更新 x3
    x3 ^= rk07;
    # 执行 AES_ROUND_NOKEY 函数，对 x0, x1, x2, x3 进行加密
    AES_ROUND_NOKEY(x0, x1, x2, x3);
    # 执行 KEY_EXPAND_ELT 函数，对 rk08, rk09, rk0A, rk0B 进行密钥扩展
    KEY_EXPAND_ELT(rk08, rk09, rk0A, rk0B);
    # 对 rk08, rk09, rk0A, rk0B 进行异或操作
    rk08 ^= rk04;
    rk09 ^= rk05;
    rk0A ^= rk06;
    rk0B ^= rk07;
    # 对 x0, x1, x2, x3 进行异或操作
    x0 ^= rk08;
    x1 ^= rk09;
    x2 ^= rk0A;
    x3 ^= rk0B;
    # 再次执行 AES_ROUND_NOKEY 函数，对 x0, x1, x2, x3 进行加密
    AES_ROUND_NOKEY(x0, x1, x2, x3);
    # 执行 KEY_EXPAND_ELT 函数，对 rk0C, rk0D, rk0E, rk0F 进行密钥扩展
    KEY_EXPAND_ELT(rk0C, rk0D, rk0E, rk0F);
    # 对 rk0C, rk0D, rk0E, rk0F 进行异或操作
    rk0C ^= rk08;
    rk0D ^= rk09;
    rk0E ^= rk0A;
    rk0F ^= rk0B;
    # 对 x0, x1, x2, x3 进行异或操作
    x0 ^= rk0C;
    x1 ^= rk0D;
    x2 ^= rk0E;
    x3 ^= rk0F;
    # 再次执行 AES_ROUND_NOKEY 函数，对 x0, x1, x2, x3 进行加密
    AES_ROUND_NOKEY(x0, x1, x2, x3);
    # 对 pC, pD, pE, pF 进行异或操作
    pC ^= x0;
    pD ^= x1;
    pE ^= x2;
    pF ^= x3;
    # 执行 KEY_EXPAND_ELT 函数，对 rk10, rk11, rk12, rk13 进行密钥扩展
    KEY_EXPAND_ELT(rk10, rk11, rk12, rk13);
    # 对 rk10, rk11, rk12, rk13 进行异或操作
    rk10 ^= rk0C;
    rk11 ^= rk0D;
    rk12 ^= rk0E;
    rk13 ^= rk0F;
    # 对 x0, x1, x2, x3 进行赋值操作
    x0 = p8 ^ rk10;
    x1 = p9 ^ rk11;
    x2 = pA ^ rk12;
    x3 = pB ^ rk13;
    # 再次执行 AES_ROUND_NOKEY 函数，对 x0, x1, x2, x3 进行加密
    AES_ROUND_NOKEY(x0, x1, x2, x3);
    # 执行 KEY_EXPAND_ELT 函数，对 rk14, rk15, rk16, rk17 进行密钥扩展
    KEY_EXPAND_ELT(rk14, rk15, rk16, rk17);
    # 对 rk14, rk15, rk16, rk17 进行异或操作
    rk14 ^= rk10;
    rk15 ^= rk11;
    rk16 ^= rk12;
    rk17 ^= rk13;
    # 对 x0, x1, x2, x3 进行异或操作
    x0 ^= rk14;
    x1 ^= rk15;
    x2 ^= rk16;
    x3 ^= rk17;
    # 再次执行 AES_ROUND_NOKEY 函数，对 x0, x1, x2, x3 进行加密
    AES_ROUND_NOKEY(x0, x1, x2, x3);
    # 执行 KEY_EXPAND_ELT 函数，对 rk18, rk19, rk1A, rk1B 进行密钥扩展
    KEY_EXPAND_ELT(rk18, rk19, rk1A, rk1B);
    # 对 rk18, rk19, rk1A, rk1B 进行异或操作
    rk18 ^= rk14 ^ sc->count1;
    rk19 ^= rk15 ^ sc->count0;
    rk1A ^= rk16 ^ sc->count3;
    rk1B ^= rk17 ^ SPH_T32(~sc->count2);
    # 对 x0, x1, x2, x3 进行异或操作
    x0 ^= rk18;
    x1 ^= rk19;
    x2 ^= rk1A;
    x3 ^= rk1B;
    # 再次执行 AES_ROUND_NOKEY 函数，对 x0, x1, x2, x3 进行加密
    AES_ROUND_NOKEY(x0, x1, x2, x3);
    # 执行 KEY_EXPAND_ELT 函数，对 rk1C, rk1D, rk1E, rk1F 进行密钥扩展
    KEY_EXPAND_ELT(rk1C, rk1D, rk1E, rk1F);
    # 对 rk1C, rk1D, rk1E, rk1F 进行异或操作
    rk1C ^= rk18;
    rk1D ^= rk19;
    rk1E ^= rk1A;
    rk1F ^= rk1B;
    # 对 x0, x1, x2, x3 进行异或操作
    x0 ^= rk1C;
    x1 ^= rk1D;
    x2 ^= rk1E;
    x3 ^= rk1F;
    # 再次执行 AES_ROUND_NOKEY 函数，对 x0, x1, x2, x3 进行加密
    AES_ROUND_NOKEY(x0, x1, x2, x3);
    # 对 p4, p5, p6, p7 进行异或操作
    p4 ^= x0;
    p5 ^= x1;
    p6 ^= x2;
    p7 ^= x3;
    # 对 sc->h 数组进行异或操作
    sc->h[0x0] ^= p8;
    sc->h[0x1] ^= p9;
    sc->h[0x2] ^= pA;
    sc->h[0x3] ^= pB;
    sc->h[0x4] ^= pC;
    sc->h[0x5] ^= pD;
    sc->h[0x6] ^= pE;
    sc->h[0x7] ^= pF;
    sc->h[0x8] ^= p0;
    sc->h[0x9] ^= p1;
    sc->h[0xA] ^= p2;
    sc->h[0xB] ^= p3;
    sc->h[0xC] ^= p4;
    sc->h[0xD] ^= p5;
    sc->h[0xE] ^= p6;
    sc->h[0xF] ^= p7;
#endif
#endif

// 初始化 Shavite 小型哈希算法的上下文，设置初始值
static void
shavite_small_init(sph_shavite_small_context *sc, const sph_u32 *iv)
{
    // 将初始值拷贝到上下文的哈希值数组中
    memcpy(sc->h, iv, sizeof sc->h);
    // 将指针、计数器等初始化为0
    sc->ptr = 0;
    sc->count0 = 0;
    sc->count1 = 0;
}

// Shavite 小型哈希算法的核心函数，处理输入数据
static void
shavite_small_core(sph_shavite_small_context *sc, const void *data, size_t len)
{
    // 定义缓冲区和指针
    unsigned char *buf;
    size_t ptr;

    // 将缓冲区和指针初始化为上下文中的对应值
    buf = sc->buf;
    ptr = sc->ptr;

    // 循环处理输入数据
    while (len > 0) {
        // 计算当前缓冲区剩余空间的长度
        size_t clen = (sizeof sc->buf) - ptr;

        // 如果剩余空间大于输入数据长度，则将整个输入数据拷贝到缓冲区中
        if (clen > len)
            clen = len;
        memcpy(buf + ptr, data, clen);

        // 更新指针、长度等值
        data = (const unsigned char *)data + clen;
        ptr += clen;
        len -= clen;

        // 如果缓冲区已满，则进行一次哈希计算
        if (ptr == sizeof sc->buf) {
            // 更新计数器
            if ((sc->count0 = SPH_T32(sc->count0 + 512)) == 0)
                sc->count1 = SPH_T32(sc->count1 + 1);
            // 调用 c256 函数进行哈希计算
            c256(sc, buf);
            // 重置指针
            ptr = 0;
        }
    }

    // 更新上下文中的指针值
    sc->ptr = ptr;
}

// 关闭 Shavite 小型哈希算法的上下文，输出哈希结果
static void
shavite_small_close(sph_shavite_small_context *sc,
    unsigned ub, unsigned n, void *dst, size_t out_size_w32)
{
    // 定义缓冲区、指针、计数器等变量
    unsigned char *buf;
    size_t ptr, u;
    unsigned z;
    sph_u32 count0, count1;

    // 将缓冲区初始化为上下文中的对应值
    buf = sc->buf;
    // 将指针、计数器等初始化为上下文中的对应值
    ptr = sc->ptr;
    count0 = (sc->count0 += (ptr << 3) + n);
    count1 = sc->count1;
    z = 0x80 >> n;
    z = ((ub & -z) | z) & 0xFF;

    // 根据指针和 n 的值，更新缓冲区中的数据
    if (ptr == 0 && n == 0) {
        buf[0] = 0x80;
        memset(buf + 1, 0, 53);
        sc->count0 = sc->count1 = 0;
    } else if (ptr < 54) {
        buf[ptr ++] = z;
        memset(buf + ptr, 0, 54 - ptr);
    } else {
        buf[ptr ++] = z;
        memset(buf + ptr, 0, 64 - ptr);
        // 调用 c256 函数进行哈希计算
        c256(sc, buf);
        memset(buf, 0, 54);
        sc->count0 = sc->count1 = 0;
    }

    // 将计数器的值和输出大小写入缓冲区
    sph_enc32le(buf + 54, count0);
    sph_enc32le(buf + 58, count1);
    buf[62] = out_size_w32 << 5;
    buf[63] = out_size_w32 >> 3;
    // 调用 c256 函数进行哈希计算
    c256(sc, buf);

    // 将哈希结果写入目标地址
    for (u = 0; u < out_size_w32; u ++)
        sph_enc32le((unsigned char *)dst + (u << 2), sc->h[u]);
}

// 初始化 Shavite 大型哈希算法的上下文，设置初始值
static void
shavite_big_init(sph_shavite_big_context *sc, const sph_u32 *iv)
{
    # 将 iv 数组中的内容复制到 sc->h 数组中，大小为 sc->h 的大小
    memcpy(sc->h, iv, sizeof sc->h);
    # 将 sc->ptr 的值设为 0
    sc->ptr = 0;
    # 将 sc->count0 的值设为 0
    sc->count0 = 0;
    # 将 sc->count1 的值设为 0
    sc->count1 = 0;
    # 将 sc->count2 的值设为 0
    sc->count2 = 0;
    # 将 sc->count3 的值设为 0
    sc->count3 = 0;
# 定义一个静态函数，用于处理shavite_big_context结构体的数据
# 参数sc: shavite_big_context结构体指针
# 参数data: 输入数据指针
# 参数len: 输入数据长度
static void
shavite_big_core(sph_shavite_big_context *sc, const void *data, size_t len)
{
    # 定义一个指向缓冲区的指针
    unsigned char *buf;
    # 定义一个指向缓冲区当前位置的指针
    size_t ptr;

    # 将缓冲区指针指向shavite_big_context结构体的buf成员
    buf = sc->buf;
    # 将ptr指针指向shavite_big_context结构体的ptr成员
    ptr = sc->ptr;
    # 当输入数据长度大于0时，循环处理数据
    while (len > 0) {
        # 定义一个变量clen，表示缓冲区剩余空间的长度
        size_t clen;

        # 计算缓冲区剩余空间的长度
        clen = (sizeof sc->buf) - ptr;
        # 如果剩余空间长度大于输入数据长度，则将clen设置为输入数据长度
        if (clen > len)
            clen = len;
        # 将输入数据拷贝到缓冲区中
        memcpy(buf + ptr, data, clen);
        # 更新输入数据指针和长度
        data = (const unsigned char *)data + clen;
        ptr += clen;
        len -= clen;
        # 如果缓冲区已满，则进行一次处理
        if (ptr == sizeof sc->buf) {
            # 更新count0的值
            if ((sc->count0 = SPH_T32(sc->count0 + 1024)) == 0) {
                # 更新count1的值
                sc->count1 = SPH_T32(sc->count1 + 1);
                # 如果count1的值为0，则更新count2的值
                if (sc->count1 == 0) {
                    sc->count2 = SPH_T32(sc->count2 + 1);
                    # 如果count2的值为0，则更新count3的值
                    if (sc->count2 == 0) {
                        sc->count3 = SPH_T32(
                            sc->count3 + 1);
                    }
                }
            }
            # 对缓冲区进行处理
            c512(sc, buf);
            # 将ptr指针重置为0
            ptr = 0;
        }
    }
    # 更新shavite_big_context结构体的ptr成员
    sc->ptr = ptr;
}

# 定义一个静态函数，用于处理shavite_big_context结构体的数据
# 参数sc: shavite_big_context结构体指针
# 参数ub: 未使用的位数
# 参数n: 未使用的字节数
# 参数dst: 输出数据指针
# 参数out_size_w32: 输出数据长度（以32位字为单位）
static void
shavite_big_close(sph_shavite_big_context *sc,
    unsigned ub, unsigned n, void *dst, size_t out_size_w32)
{
    # 定义一个指向缓冲区的指针
    unsigned char *buf;
    # 定义一个指向缓冲区当前位置的指针
    size_t ptr;
    # 定义一个变量u，表示输出数据长度（以字节为单位）
    size_t u;
    # 定义一个变量z，表示未使用的位数
    unsigned z;
    # 定义四个变量，用于保存count0、count1、count2和count3的值
    sph_u32 count0, count1, count2, count3;

    # 将缓冲区指针指向shavite_big_context结构体的buf成员
    buf = sc->buf;
    # 将ptr指针指向shavite_big_context结构体的ptr成员
    ptr = sc->ptr;
    # 更新count0的值
    count0 = (sc->count0 += (ptr << 3) + n);
    # 将count1的值赋给count1变量
    count1 = sc->count1;
    # 将count2的值赋给count2变量
    count2 = sc->count2;
    # 将count3的值赋给count3变量
    count3 = sc->count3;
    # 计算z的值
    z = 0x80 >> n;
    z = ((ub & -z) | z) & 0xFF;
    # 如果ptr为0且n为0，则将buf的第一个字节设置为0x80，其余字节设置为0，并将count0、count1、count2和count3的值重置为0
    if (ptr == 0 && n == 0) {
        buf[0] = 0x80;
        memset(buf + 1, 0, 109);
        sc->count0 = sc->count1 = sc->count2 = sc->count3 = 0;
    # 如果ptr小于110，则将buf的第ptr个字节设置为z，其余字节设置为0
    } else if (ptr < 110) {
        buf[ptr ++] = z;
        memset(buf + ptr, 0, 110 - ptr);
    # 否则，将buf的第ptr个字节设置为z，其余字节设置为0，并对缓冲区进行处理，将buf的前110个字节设置为0，将count0、count1、count2和count3的值重置为0
    } else {
        buf[ptr ++] = z;
        memset(buf + ptr, 0, 128 - ptr);
        c512(sc, buf);
        memset(buf, 0, 110);
        sc->count0 = sc->count1 = sc->count2 = sc->count3 = 0;
    }
    # 将count0、count1、count2和count3的值分别写入buf的第110、114、118和122个字节中
    sph_enc32le(buf + 110, count0);
    sph_enc32le(buf + 114, count1);
    sph_enc32le(buf + 118, count2);
    sph_enc32le(buf + 122, count3);
    # 将out_size_w32左移5位，并将结果存入buf的第126个位置
    buf[126] = out_size_w32 << 5;
    # 将out_size_w32右移3位，并将结果存入buf的第127个位置
    buf[127] = out_size_w32 >> 3;
    # 使用c512函数对buf进行处理
    c512(sc, buf);
    # 遍历out_size_w32次，将sc->h[u]的值以小端格式写入dst的(u<<2)位置
    for (u = 0; u < out_size_w32; u ++)
        sph_enc32le((unsigned char *)dst + (u << 2), sc->h[u]);
/* see sph_shavite.h */
// 初始化 Shavite224 算法的上下文
void
sph_shavite224_init(void *cc)
{
    // 调用 shavite_small_init 函数，传入上下文和 IV224 常量
    shavite_small_init(cc, IV224);
}

/* see sph_shavite.h */
// 执行 Shavite224 算法的压缩函数
void
sph_shavite224(void *cc, const void *data, size_t len)
{
    // 调用 shavite_small_core 函数，传入上下文、数据和数据长度
    shavite_small_core(cc, data, len);
}

/* see sph_shavite.h */
// 关闭 Shavite224 算法的上下文，并输出结果
void
sph_shavite224_close(void *cc, void *dst)
{
    // 调用 shavite_small_close 函数，传入上下文、0、0、输出结果和 7
    shavite_small_close(cc, 0, 0, dst, 7);
    // 重新初始化上下文，传入 IV224 常量
    shavite_small_init(cc, IV224);
}

/* see sph_shavite.h */
// 添加位并关闭 Shavite224 算法的上下文，并输出结果
void
sph_shavite224_addbits_and_close(void *cc, unsigned ub, unsigned n, void *dst)
{
    // 调用 shavite_small_close 函数，传入上下文、ub、n、输出结果和 7
    shavite_small_close(cc, ub, n, dst, 7);
    // 重新初始化上下文，传入 IV224 常量
    shavite_small_init(cc, IV224);
}

// 以下代码与上述类似，只是针对不同的 Shavite 算法版本，作用相同，不再重复解释
# 关闭 Shavite 大散列对象，将结果写入目标缓冲区
# 参数 cc: Shavite 大散列对象
# 参数 dst: 目标缓冲区
sph_shavite512_close(void *cc, void *dst)
{
    # 调用 shavite_big_close 函数关闭 Shavite 大散列对象，将结果写入目标缓冲区
    shavite_big_close(cc, 0, 0, dst, 16);
    # 调用 shavite_big_init 函数重新初始化 Shavite 大散列对象，使用 IV512 常量
    shavite_big_init(cc, IV512);
}

# 添加位数并关闭 Shavite 大散列对象，将结果写入目标缓冲区
# 参数 cc: Shavite 大散列对象
# 参数 ub: 位数的高 8 位
# 参数 n: 位数的低 24 位
# 参数 dst: 目标缓冲区
void
sph_shavite512_addbits_and_close(void *cc, unsigned ub, unsigned n, void *dst)
{
    # 调用 shavite_big_close 函数关闭 Shavite 大散列对象，将结果写入目标缓冲区
    shavite_big_close(cc, ub, n, dst, 16);
    # 调用 shavite_big_init 函数重新初始化 Shavite 大散列对象，使用 IV512 常量
    shavite_big_init(cc, IV512);
}

#ifdef __cplusplus
}
#endif
```