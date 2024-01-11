# `xmrig\src\crypto\ghostrider\sph_groestl.c`

```
/* $Id: groestl.c 260 2011-07-21 01:02:38Z tp $ */
/* 
 * Groestl implementation.
 * Groestl 算法的实现
 *
 * ==========================(LICENSE BEGIN)============================
 * 授权许可声明开始
 *
 * Copyright (c) 2007-2010  Projet RNRT SAPHIR
 * 版权声明
 *
 * Permission is hereby granted, free of charge, to any person obtaining
 * a copy of this software and associated documentation files (the
 * "Software"), to deal in the Software without restriction, including
 * without limitation the rights to use, copy, modify, merge, publish,
 * distribute, sublicense, and/or sell copies of the Software, and to
 * permit persons to whom the Software is furnished to do so, subject to
 * the following conditions:
 * 授权许可声明内容
 *
 * The above copyright notice and this permission notice shall be
 * included in all copies or substantial portions of the Software.
 * 上述版权声明和授权声明应包含在所有副本或实质部分的软件中
 *
 * THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND,
 * EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF
 * MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT.
 * IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY
 * CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT,
 * TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE
 * SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
 * 软件提供的是"按现状"的，不提供任何形式的明示或暗示的担保，包括但不限于对适销性、特定用途的适用性和非侵权性的担保。在任何情况下，作者或版权持有人均不对任何索赔、损害或其他责任负责，无论是合同诉讼、侵权行为还是其他情况，无论是由于软件或使用或其他交易引起的、与之相关的或与之无关的
 *
 * ===========================(LICENSE END)=============================
 * 授权许可声明结束
 *
 * @author   Thomas Pornin <thomas.pornin@cryptolog.com>
 * 作者信息
 */

#include <stddef.h>
#include <string.h>

#include "sph_groestl.h"

#ifdef __cplusplus
extern "C"{
#endif

#if SPH_SMALL_FOOTPRINT && !defined SPH_SMALL_FOOTPRINT_GROESTL
#define SPH_SMALL_FOOTPRINT_GROESTL   1
#endif

/*
 * Apparently, the 32-bit-only version is not faster than the 64-bit
 * version unless using the "small footprint" code on a 32-bit machine.
 * 显然，仅支持32位的版本在32位机器上除非使用"small footprint"代码，否则不会比64位版本更快
 */
#if !defined SPH_GROESTL_64
#if SPH_SMALL_FOOTPRINT_GROESTL && !SPH_64_TRUE
#define SPH_GROESTL_64   0
#else
#define SPH_GROESTL_64   1
#endif
#endif

#if !SPH_64
#undef SPH_GROESTL_64
#endif
#ifdef _MSC_VER
#pragma warning (disable: 4146)
#endif



/*
 * The internal representation may use either big-endian or
 * little-endian. Using the platform default representation speeds up
 * encoding and decoding between bytes and the matrix columns.
 */



#undef USE_LE
#if SPH_GROESTL_LITTLE_ENDIAN
#define USE_LE   1
#elif SPH_GROESTL_BIG_ENDIAN
#define USE_LE   0
#elif SPH_LITTLE_ENDIAN
#define USE_LE   1
#endif



#if USE_LE



#define C32e(x)     ((SPH_C32(x) >> 24) \
                    | ((SPH_C32(x) >>  8) & SPH_C32(0x0000FF00)) \
                    | ((SPH_C32(x) <<  8) & SPH_C32(0x00FF0000)) \
                    | ((SPH_C32(x) << 24) & SPH_C32(0xFF000000)))
#define dec32e_aligned   sph_dec32le_aligned
#define enc32e           sph_enc32le
#define B32_0(x)    ((x) & 0xFF)
#define B32_1(x)    (((x) >> 8) & 0xFF)
#define B32_2(x)    (((x) >> 16) & 0xFF)
#define B32_3(x)    ((x) >> 24)



#define R32u(u, d)   SPH_T32(((u) << 16) | ((d) >> 16))
#define R32d(u, d)   SPH_T32(((u) >> 16) | ((d) << 16))



#define PC32up(j, r)   ((sph_u32)((j) + (r)))
#define PC32dn(j, r)   0
#define QC32up(j, r)   SPH_C32(0xFFFFFFFF)
#define QC32dn(j, r)   (((sph_u32)(r) << 24) ^ SPH_T32(~((sph_u32)(j) << 24)))



#if SPH_64
#define C64e(x)     ((SPH_C64(x) >> 56) \
                    | ((SPH_C64(x) >> 40) & SPH_C64(0x000000000000FF00)) \
                    | ((SPH_C64(x) >> 24) & SPH_C64(0x0000000000FF0000)) \
                    | ((SPH_C64(x) >>  8) & SPH_C64(0x00000000FF000000)) \
                    | ((SPH_C64(x) <<  8) & SPH_C64(0x000000FF00000000)) \
                    | ((SPH_C64(x) << 24) & SPH_C64(0x0000FF0000000000)) \
                    | ((SPH_C64(x) << 40) & SPH_C64(0x00FF000000000000)) \
                    | ((SPH_C64(x) << 56) & SPH_C64(0xFF00000000000000)))
#define dec64e_aligned   sph_dec64le_aligned
#define enc64e           sph_enc64le
#define B64_0(x)    ((x) & 0xFF)
#define B64_1(x)    (((x) >> 8) & 0xFF)
#define B64_2(x)    (((x) >> 16) & 0xFF)
#define B64_3(x)    (((x) >> 24) & 0xFF)  # 定义宏，用于提取64位数x的第3个字节
#define B64_4(x)    (((x) >> 32) & 0xFF)  # 定义宏，用于提取64位数x的第4个字节
#define B64_5(x)    (((x) >> 40) & 0xFF)  # 定义宏，用于提取64位数x的第5个字节
#define B64_6(x)    (((x) >> 48) & 0xFF)  # 定义宏，用于提取64位数x的第6个字节
#define B64_7(x)    ((x) >> 56)           # 定义宏，用于提取64位数x的第7个字节
#define R64         SPH_ROTL64             # 定义宏，用于对64位数进行循环左移
#define PC64(j, r)  ((sph_u64)((j) + (r)))  # 定义宏，用于对64位数进行加法运算
#define QC64(j, r)  (((sph_u64)(r) << 56) ^ SPH_T64(~((sph_u64)(j) << 56)))  # 定义宏，用于对64位数进行异或运算

#define C32e(x)     SPH_C32(x)  # 定义宏，用于将32位数x转换为大端字节序
#define dec32e_aligned   sph_dec32be_aligned  # 定义宏，用于从对齐的大端字节序中解码32位数
#define enc32e           sph_enc32be  # 定义宏，用于将32位数编码为大端字节序
#define B32_0(x)    ((x) >> 24)  # 定义宏，用于提取32位数x的第0个字节
#define B32_1(x)    (((x) >> 16) & 0xFF)  # 定义宏，用于提取32位数x的第1个字节
#define B32_2(x)    (((x) >> 8) & 0xFF)   # 定义宏，用于提取32位数x的第2个字节
#define B32_3(x)    ((x) & 0xFF)          # 定义宏，用于提取32位数x的第3个字节
#define R32u(u, d)   SPH_T32(((u) >> 16) | ((d) << 16))  # 定义宏，用于对32位数进行无符号右移和无符号左移
#define R32d(u, d)   SPH_T32(((u) << 16) | ((d) >> 16))  # 定义宏，用于对32位数进行无符号左移和无符号右移
#define PC32up(j, r)   ((sph_u32)((j) + (r)) << 24)  # 定义宏，用于对32位数进行加法运算和左移
#define PC32dn(j, r)   0  # 定义宏，用于对32位数进行加法运算和右移
#define QC32up(j, r)   SPH_C32(0xFFFFFFFF)  # 定义宏，用于对32位数进行异或运算
#define QC32dn(j, r)   ((sph_u32)(r) ^ SPH_T32(~(sph_u32)(j)))  # 定义宏，用于对32位数进行异或运算
#if SPH_64
#define C64e(x)     SPH_C64(x)  # 如果SPH_64为真，则定义宏，用于将64位数x转换为大端字节序
#define dec64e_aligned   sph_dec64be_aligned  # 如果SPH_64为真，则定义宏，用于从对齐的大端字节序中解码64位数
#define enc64e           sph_enc64be  # 如果SPH_64为真，则定义宏，用于将64位数编码为大端字节序
#define B64_0(x)    ((x) >> 56)  # 如果SPH_64为真，则定义宏，用于提取64位数x的第0个字节
#define B64_1(x)    (((x) >> 48) & 0xFF)  # 如果SPH_64为真，则定义宏，用于提取64位数x的第1个字节
#define B64_2(x)    (((x) >> 40) & 0xFF)  # 如果SPH_64为真，则定义宏，用于提取64位数x的第2个字节
#define B64_3(x)    (((x) >> 32) & 0xFF)  # 如果SPH_64为真，则定义宏，用于提取64位数x的第3个字节
#define B64_4(x)    (((x) >> 24) & 0xFF)  # 如果SPH_64为真，则定义宏，用于提取64位数x的第4个字节
#define B64_5(x)    (((x) >> 16) & 0xFF)  # 如果SPH_64为真，则定义宏，用于提取64位数x的第5个字节
#define B64_6(x)    (((x) >> 8) & 0xFF)   # 如果SPH_64为真，则定义宏，用于提取64位数x的第6个字节
#define B64_7(x)    ((x) & 0xFF)          # 如果SPH_64为真，则定义宏，用于提取64位数x的第7个字节
#define R64         SPH_ROTR64             # 如果SPH_64为真，则定义宏，用于对64位数进行循环右移
#define PC64(j, r)  ((sph_u64)((j) + (r)) << 56)  # 如果SPH_64为真，则定义宏，用于对64位数进行加法运算和左移
#define QC64(j, r)  ((sph_u64)(r) ^ SPH_T64(~(sph_u64)(j)))  # 如果SPH_64为真，则定义宏，用于对64位数进行异或运算
#endif

#endif

#if SPH_GROESTL_64

static const sph_u64 T0[] = {
    C64e(0xc632f4a5f497a5c6), C64e(0xf86f978497eb84f8),
    C64e(0xee5eb099b0c799ee), C64e(0xf67a8c8d8cf78df6),
    C64e(0xffe8170d17e50dff), C64e(0xd60adcbddcb7bdd6),
    C64e(0xde16c8b1c8a7b1de), C64e(0x916dfc54fc395491),
    C64e(0x6090f050f0c05060), C64e(0x0207050305040302),
    C64e(0xce2ee0a9e087a9ce), C64e(0x56d1877d87ac7d56),
    C64e(0xe7cc2b192bd519e7), C64e(0xb513a662a67162b5),
    # 以下是一系列的函数调用，每个函数调用都传入一个十六进制数作为参数
    # 这些函数调用可能是用于数据加密或者其他算法操作
    C64e(0x4d7c31e6319ae64d), C64e(0xec59b59ab5c39aec),
    C64e(0x8f40cf45cf05458f), C64e(0x1fa3bc9dbc3e9d1f),
    C64e(0x8949c040c0094089), C64e(0xfa68928792ef87fa),
    C64e(0xefd03f153fc515ef), C64e(0xb29426eb267febb2),
    C64e(0x8ece40c94007c98e), C64e(0xfbe61d0b1ded0bfb),
    C64e(0x416e2fec2f82ec41), C64e(0xb31aa967a97d67b3),
    C64e(0x5f431cfd1cbefd5f), C64e(0x456025ea258aea45),
    C64e(0x23f9dabfda46bf23), C64e(0x535102f702a6f753),
    C64e(0xe445a196a1d396e4), C64e(0x9b76ed5bed2d5b9b),
    C64e(0x75285dc25deac275), C64e(0xe1c5241c24d91ce1),
    C64e(0x3dd4e9aee97aae3d), C64e(0x4cf2be6abe986a4c),
    C64e(0x6c82ee5aeed85a6c), C64e(0x7ebdc341c3fc417e),
    C64e(0xf5f3060206f102f5), C64e(0x8352d14fd11d4f83),
    C64e(0x688ce45ce4d05c68), C64e(0x515607f407a2f451),
    C64e(0xd18d5c345cb934d1), C64e(0xf9e1180818e908f9),
    C64e(0xe24cae93aedf93e2), C64e(0xab3e9573954d73ab),
    C64e(0x6297f553f5c45362), C64e(0x2a6b413f41543f2a),
    C64e(0x081c140c14100c08), C64e(0x9563f652f6315295),
    C64e(0x46e9af65af8c6546), C64e(0x9d7fe25ee2215e9d),
    C64e(0x3048782878602830), C64e(0x37cff8a1f86ea137),
    C64e(0x0a1b110f11140f0a), C64e(0x2febc4b5c45eb52f),
    C64e(0x0e151b091b1c090e), C64e(0x247e5a365a483624),
    C64e(0x1badb69bb6369b1b), C64e(0xdf98473d47a53ddf),
    C64e(0xcda76a266a8126cd), C64e(0x4ef5bb69bb9c694e),
    C64e(0x7f334ccd4cfecd7f), C64e(0xea50ba9fbacf9fea),
    C64e(0x123f2d1b2d241b12), C64e(0x1da4b99eb93a9e1d),
    C64e(0x58c49c749cb07458), C64e(0x3446722e72682e34),
    C64e(0x3641772d776c2d36), C64e(0xdc11cdb2cda3b2dc),
    C64e(0xb49d29ee2973eeb4), C64e(0x5b4d16fb16b6fb5b),
    C64e(0xa4a501f60153f6a4), C64e(0x76a1d74dd7ec4d76),
    C64e(0xb714a361a37561b7), C64e(0x7d3449ce49face7d),
    C64e(0x52df8d7b8da47b52), C64e(0xdd9f423e42a13edd),
    C64e(0x5ecd937193bc715e), C64e(0x13b1a297a2269713),
    C64e(0xa6a204f50457f5a6), C64e(0xb901b868b86968b9),
    C64e(0x0000000000000000), C64e(0xc1b5742c74992cc1),
    # 以下是一系列的函数调用，每个函数调用都传入一个十六进制数作为参数
    # 这些函数调用可能是对输入数据进行加密、解密或其他处理
    # 由于没有上下文，无法确定这些函数的具体作用
    # 需要查看函数定义或者相关文档才能确定其作用
    C64e(0x40e0a060a0806040), C64e(0xe3c2211f21dd1fe3),
    C64e(0x793a43c843f2c879), C64e(0xb69a2ced2c77edb6),
    C64e(0xd40dd9bed9b3bed4), C64e(0x8d47ca46ca01468d),
    C64e(0x671770d970ced967), C64e(0x72afdd4bdde44b72),
    C64e(0x94ed79de7933de94), C64e(0x98ff67d4672bd498),
    C64e(0xb09323e8237be8b0), C64e(0x855bde4ade114a85),
    C64e(0xbb06bd6bbd6d6bbb), C64e(0xc5bb7e2a7e912ac5),
    C64e(0x4f7b34e5349ee54f), C64e(0xedd73a163ac116ed),
    C64e(0x86d254c55417c586), C64e(0x9af862d7622fd79a),
    C64e(0x6699ff55ffcc5566), C64e(0x11b6a794a7229411),
    C64e(0x8ac04acf4a0fcf8a), C64e(0xe9d9301030c910e9),
    C64e(0x040e0a060a080604), C64e(0xfe66988198e781fe),
    C64e(0xa0ab0bf00b5bf0a0), C64e(0x78b4cc44ccf04478),
    C64e(0x25f0d5bad54aba25), C64e(0x4b753ee33e96e34b),
    C64e(0xa2ac0ef30e5ff3a2), C64e(0x5d4419fe19bafe5d),
    C64e(0x80db5bc05b1bc080), C64e(0x0580858a850a8a05),
    C64e(0x3fd3ecadec7ead3f), C64e(0x21fedfbcdf42bc21),
    C64e(0x70a8d848d8e04870), C64e(0xf1fd0c040cf904f1),
    C64e(0x63197adf7ac6df63), C64e(0x772f58c158eec177),
    C64e(0xaf309f759f4575af), C64e(0x42e7a563a5846342),
    C64e(0x2070503050403020), C64e(0xe5cb2e1a2ed11ae5),
    C64e(0xfdef120e12e10efd), C64e(0xbf08b76db7656dbf),
    C64e(0x8155d44cd4194c81), C64e(0x18243c143c301418),
    C64e(0x26795f355f4c3526), C64e(0xc3b2712f719d2fc3),
    C64e(0xbe8638e13867e1be), C64e(0x35c8fda2fd6aa235),
    C64e(0x88c74fcc4f0bcc88), C64e(0x2e654b394b5c392e),
    C64e(0x936af957f93d5793), C64e(0x55580df20daaf255),
    C64e(0xfc619d829de382fc), C64e(0x7ab3c947c9f4477a),
    C64e(0xc827efacef8bacc8), C64e(0xba8832e7326fe7ba),
    C64e(0x324f7d2b7d642b32), C64e(0xe642a495a4d795e6),
    C64e(0xc03bfba0fb9ba0c0), C64e(0x19aab398b3329819),
    C64e(0x9ef668d16827d19e), C64e(0xa322817f815d7fa3),
    C64e(0x44eeaa66aa886644), C64e(0x54d6827e82a87e54),
    C64e(0x3bdde6abe676ab3b), C64e(0x0b959e839e16830b),
    C64e(0x8cc945ca4503ca8c), C64e(0xc7bc7b297b9529c7),
    # 以下是一系列经过 C64e 函数处理的十六进制数值，具体含义需要查看 C64e 函数的定义
    C64e(0x6b056ed36ed6d36b), C64e(0x286c443c44503c28),
    C64e(0xa72c8b798b5579a7), C64e(0xbc813de23d63e2bc),
    C64e(0x1631271d272c1d16), C64e(0xad379a769a4176ad),
    C64e(0xdb964d3b4dad3bdb), C64e(0x649efa56fac85664),
    C64e(0x74a6d24ed2e84e74), C64e(0x1436221e22281e14),
    C64e(0x92e476db763fdb92), C64e(0x0c121e0a1e180a0c),
    C64e(0x48fcb46cb4906c48), C64e(0xb88f37e4376be4b8),
    C64e(0x9f78e75de7255d9f), C64e(0xbd0fb26eb2616ebd),
    C64e(0x43692aef2a86ef43), C64e(0xc435f1a6f193a6c4),
    C64e(0x39dae3a8e372a839), C64e(0x31c6f7a4f762a431),
    C64e(0xd38a593759bd37d3), C64e(0xf274868b86ff8bf2),
    C64e(0xd583563256b132d5), C64e(0x8b4ec543c50d438b),
    C64e(0x6e85eb59ebdc596e), C64e(0xda18c2b7c2afb7da),
    C64e(0x018e8f8c8f028c01), C64e(0xb11dac64ac7964b1),
    C64e(0x9cf16dd26d23d29c), C64e(0x49723be03b92e049),
    C64e(0xd81fc7b4c7abb4d8), C64e(0xacb915fa1543faac),
    C64e(0xf3fa090709fd07f3), C64e(0xcfa06f256f8525cf),
    C64e(0xca20eaafea8fafca), C64e(0xf47d898e89f38ef4),
    C64e(0x476720e9208ee947), C64e(0x1038281828201810),
    C64e(0x6f0b64d564ded56f), C64e(0xf073838883fb88f0),
    C64e(0x4afbb16fb1946f4a), C64e(0x5cca967296b8725c),
    C64e(0x38546c246c702438), C64e(0x575f08f108aef157),
    C64e(0x732152c752e6c773), C64e(0x9764f351f3355197),
    C64e(0xcbae6523658d23cb), C64e(0xa125847c84597ca1),
    C64e(0xe857bf9cbfcb9ce8), C64e(0x3e5d6321637c213e),
    C64e(0x96ea7cdd7c37dd96), C64e(0x611e7fdc7fc2dc61),
    C64e(0x0d9c9186911a860d), C64e(0x0f9b9485941e850f),
    C64e(0xe04bab90abdb90e0), C64e(0x7cbac642c6f8427c),
    C64e(0x712657c457e2c471), C64e(0xcc29e5aae583aacc),
    C64e(0x90e373d8733bd890), C64e(0x06090f050f0c0506),
    C64e(0xf7f4030103f501f7), C64e(0x1c2a36123638121c),
    C64e(0xc23cfea3fe9fa3c2), C64e(0x6a8be15fe1d45f6a),
    C64e(0xaebe10f91047f9ae), C64e(0x69026bd06bd2d069),
    C64e(0x17bfa891a82e9117), C64e(0x9971e858e8295899),
    C64e(0x3a5369276974273a), C64e(0x27f7d0b9d04eb927),
    # 这些代码看起来是一系列函数调用，但是缺少上下文，无法确定具体作用
    # 需要更多的代码上下文才能正确解释这些函数调用的作用
# 如果不是小型足迹的 Groestl 算法，则执行以下代码
#if !SPH_SMALL_FOOTPRINT_GROESTL

# 定义 T1 常量数组，包含一系列 64 位无符号整数
static const sph_u64 T1[] = {
    C64e(0xc6c632f4a5f497a5), C64e(0xf8f86f978497eb84),
    C64e(0xeeee5eb099b0c799), C64e(0xf6f67a8c8d8cf78d),
    C64e(0xffffe8170d17e50d), C64e(0xd6d60adcbddcb7bd),
    C64e(0xdede16c8b1c8a7b1), C64e(0x91916dfc54fc3954),
    C64e(0x606090f050f0c050), C64e(0x0202070503050403),
    C64e(0xcece2ee0a9e087a9), C64e(0x5656d1877d87ac7d),
    C64e(0xe7e7cc2b192bd519), C64e(0xb5b513a662a67162),
    C64e(0x4d4d7c31e6319ae6), C64e(0xecec59b59ab5c39a),
    C64e(0x8f8f40cf45cf0545), C64e(0x1f1fa3bc9dbc3e9d),
    C64e(0x898949c040c00940), C64e(0xfafa68928792ef87),
    C64e(0xefefd03f153fc515), C64e(0xb2b29426eb267feb),
    C64e(0x8e8ece40c94007c9), C64e(0xfbfbe61d0b1ded0b),
    C64e(0x41416e2fec2f82ec), C64e(0xb3b31aa967a97d67),
    C64e(0x5f5f431cfd1cbefd), C64e(0x45456025ea258aea),
    C64e(0x2323f9dabfda46bf), C64e(0x53535102f702a6f7),
    C64e(0xe4e445a196a1d396), C64e(0x9b9b76ed5bed2d5b),
    C64e(0x7575285dc25deac2), C64e(0xe1e1c5241c24d91c),
    C64e(0x3d3dd4e9aee97aae), C64e(0x4c4cf2be6abe986a),
    C64e(0x6c6c82ee5aeed85a), C64e(0x7e7ebdc341c3fc41),
    C64e(0xf5f5f3060206f102), C64e(0x838352d14fd11d4f),
    C64e(0x68688ce45ce4d05c), C64e(0x51515607f407a2f4),
    C64e(0xd1d18d5c345cb934), C64e(0xf9f9e1180818e908),
    C64e(0xe2e24cae93aedf93), C64e(0xabab3e9573954d73),
    C64e(0x626297f553f5c453), C64e(0x2a2a6b413f41543f),
    C64e(0x08081c140c14100c), C64e(0x959563f652f63152),
    C64e(0x4646e9af65af8c65), C64e(0x9d9d7fe25ee2215e),
    C64e(0x3030487828786028), C64e(0x3737cff8a1f86ea1),
    C64e(0x0a0a1b110f11140f), C64e(0x2f2febc4b5c45eb5),
    C64e(0x0e0e151b091b1c09), C64e(0x24247e5a365a4836),
    C64e(0x1b1badb69bb6369b), C64e(0xdfdf98473d47a53d),
    C64e(0xcdcda76a266a8126), C64e(0x4e4ef5bb69bb9c69),
    C64e(0x7f7f334ccd4cfecd), C64e(0xeaea50ba9fbacf9f),
    C64e(0x12123f2d1b2d241b), C64e(0x1d1da4b99eb93a9e),
    C64e(0x5858c49c749cb074), C64e(0x343446722e72682e),
    # 以下是一系列经过 C64e 函数加密的数据，每个数据都是 8 个字节
    # 这些数据可能是密钥、向量或者其他加密相关的数据
    C64e(0x363641772d776c2d), C64e(0xdcdc11cdb2cda3b2),
    C64e(0xb4b49d29ee2973ee), C64e(0x5b5b4d16fb16b6fb),
    C64e(0xa4a4a501f60153f6), C64e(0x7676a1d74dd7ec4d),
    C64e(0xb7b714a361a37561), C64e(0x7d7d3449ce49face),
    C64e(0x5252df8d7b8da47b), C64e(0xdddd9f423e42a13e),
    C64e(0x5e5ecd937193bc71), C64e(0x1313b1a297a22697),
    C64e(0xa6a6a204f50457f5), C64e(0xb9b901b868b86968),
    C64e(0x0000000000000000), C64e(0xc1c1b5742c74992c),
    C64e(0x4040e0a060a08060), C64e(0xe3e3c2211f21dd1f),
    C64e(0x79793a43c843f2c8), C64e(0xb6b69a2ced2c77ed),
    C64e(0xd4d40dd9bed9b3be), C64e(0x8d8d47ca46ca0146),
    C64e(0x67671770d970ced9), C64e(0x7272afdd4bdde44b),
    C64e(0x9494ed79de7933de), C64e(0x9898ff67d4672bd4),
    C64e(0xb0b09323e8237be8), C64e(0x85855bde4ade114a),
    C64e(0xbbbb06bd6bbd6d6b), C64e(0xc5c5bb7e2a7e912a),
    C64e(0x4f4f7b34e5349ee5), C64e(0xededd73a163ac116),
    C64e(0x8686d254c55417c5), C64e(0x9a9af862d7622fd7),
    C64e(0x666699ff55ffcc55), C64e(0x1111b6a794a72294),
    C64e(0x8a8ac04acf4a0fcf), C64e(0xe9e9d9301030c910),
    C64e(0x04040e0a060a0806), C64e(0xfefe66988198e781),
    C64e(0xa0a0ab0bf00b5bf0), C64e(0x7878b4cc44ccf044),
    C64e(0x2525f0d5bad54aba), C64e(0x4b4b753ee33e96e3),
    C64e(0xa2a2ac0ef30e5ff3), C64e(0x5d5d4419fe19bafe),
    C64e(0x8080db5bc05b1bc0), C64e(0x050580858a850a8a),
    C64e(0x3f3fd3ecadec7ead), C64e(0x2121fedfbcdf42bc),
    C64e(0x7070a8d848d8e048), C64e(0xf1f1fd0c040cf904),
    C64e(0x6363197adf7ac6df), C64e(0x77772f58c158eec1),
    C64e(0xafaf309f759f4575), C64e(0x4242e7a563a58463),
    C64e(0x2020705030504030), C64e(0xe5e5cb2e1a2ed11a),
    C64e(0xfdfdef120e12e10e), C64e(0xbfbf08b76db7656d),
    C64e(0x818155d44cd4194c), C64e(0x1818243c143c3014),
    C64e(0x2626795f355f4c35), C64e(0xc3c3b2712f719d2f),
    C64e(0xbebe8638e13867e1), C64e(0x3535c8fda2fd6aa2),
    C64e(0x8888c74fcc4f0bcc), C64e(0x2e2e654b394b5c39),
    C64e(0x93936af957f93d57), C64e(0x5555580df20daaf2),
    # 使用C64e函数对给定的十六进制数进行解密
    C64e(0xfcfc619d829de382), C64e(0x7a7ab3c947c9f447),
    C64e(0xc8c827efacef8bac), C64e(0xbaba8832e7326fe7),
    C64e(0x32324f7d2b7d642b), C64e(0xe6e642a495a4d795),
    C64e(0xc0c03bfba0fb9ba0), C64e(0x1919aab398b33298),
    C64e(0x9e9ef668d16827d1), C64e(0xa3a322817f815d7f),
    C64e(0x4444eeaa66aa8866), C64e(0x5454d6827e82a87e),
    C64e(0x3b3bdde6abe676ab), C64e(0x0b0b959e839e1683),
    C64e(0x8c8cc945ca4503ca), C64e(0xc7c7bc7b297b9529),
    C64e(0x6b6b056ed36ed6d3), C64e(0x28286c443c44503c),
    C64e(0xa7a72c8b798b5579), C64e(0xbcbc813de23d63e2),
    C64e(0x161631271d272c1d), C64e(0xadad379a769a4176),
    C64e(0xdbdb964d3b4dad3b), C64e(0x64649efa56fac856),
    C64e(0x7474a6d24ed2e84e), C64e(0x141436221e22281e),
    C64e(0x9292e476db763fdb), C64e(0x0c0c121e0a1e180a),
    C64e(0x4848fcb46cb4906c), C64e(0xb8b88f37e4376be4),
    C64e(0x9f9f78e75de7255d), C64e(0xbdbd0fb26eb2616e),
    C64e(0x4343692aef2a86ef), C64e(0xc4c435f1a6f193a6),
    C64e(0x3939dae3a8e372a8), C64e(0x3131c6f7a4f762a4),
    C64e(0xd3d38a593759bd37), C64e(0xf2f274868b86ff8b),
    C64e(0xd5d583563256b132), C64e(0x8b8b4ec543c50d43),
    C64e(0x6e6e85eb59ebdc59), C64e(0xdada18c2b7c2afb7),
    C64e(0x01018e8f8c8f028c), C64e(0xb1b11dac64ac7964),
    C64e(0x9c9cf16dd26d23d2), C64e(0x4949723be03b92e0),
    C64e(0xd8d81fc7b4c7abb4), C64e(0xacacb915fa1543fa),
    C64e(0xf3f3fa090709fd07), C64e(0xcfcfa06f256f8525),
    C64e(0xcaca20eaafea8faf), C64e(0xf4f47d898e89f38e),
    C64e(0x47476720e9208ee9), C64e(0x1010382818282018),
    C64e(0x6f6f0b64d564ded5), C64e(0xf0f073838883fb88),
    C64e(0x4a4afbb16fb1946f), C64e(0x5c5cca967296b872),
    C64e(0x3838546c246c7024), C64e(0x57575f08f108aef1),
    C64e(0x73732152c752e6c7), C64e(0x979764f351f33551),
    C64e(0xcbcbae6523658d23), C64e(0xa1a125847c84597c),
    C64e(0xe8e857bf9cbfcb9c), C64e(0x3e3e5d6321637c21),
    C64e(0x9696ea7cdd7c37dd), C64e(0x61611e7fdc7fc2dc),
    C64e(0x0d0d9c9186911a86), C64e(0x0f0f9b9485941e85),
    # 使用C64e函数对给定的十六进制数进行加密
    C64e(0xe0e04bab90abdb90), C64e(0x7c7cbac642c6f842),
    C64e(0x71712657c457e2c4), C64e(0xcccc29e5aae583aa),
    C64e(0x9090e373d8733bd8), C64e(0x0606090f050f0c05),
    C64e(0xf7f7f4030103f501), C64e(0x1c1c2a3612363812),
    C64e(0xc2c23cfea3fe9fa3), C64e(0x6a6a8be15fe1d45f),
    C64e(0xaeaebe10f91047f9), C64e(0x6969026bd06bd2d0),
    C64e(0x1717bfa891a82e91), C64e(0x999971e858e82958),
    C64e(0x3a3a536927697427), C64e(0x2727f7d0b9d04eb9),
    C64e(0xd9d991483848a938), C64e(0xebebde351335cd13),
    C64e(0x2b2be5ceb3ce56b3), C64e(0x2222775533554433),
    C64e(0xd2d204d6bbd6bfbb), C64e(0xa9a9399070904970),
    C64e(0x0707878089800e89), C64e(0x3333c1f2a7f266a7),
    C64e(0x2d2decc1b6c15ab6), C64e(0x3c3c5a6622667822),
    C64e(0x1515b8ad92ad2a92), C64e(0xc9c9a96020608920),
    C64e(0x87875cdb49db1549), C64e(0xaaaab01aff1a4fff),
    C64e(0x5050d8887888a078), C64e(0xa5a52b8e7a8e517a),
    C64e(0x0303898a8f8a068f), C64e(0x59594a13f813b2f8),
    C64e(0x0909929b809b1280), C64e(0x1a1a233917393417),
    C64e(0x65651075da75cada), C64e(0xd7d784533153b531),
    C64e(0x8484d551c65113c6), C64e(0xd0d003d3b8d3bbb8),
    C64e(0x8282dc5ec35e1fc3), C64e(0x2929e2cbb0cb52b0),
    C64e(0x5a5ac3997799b477), C64e(0x1e1e2d3311333c11),
    C64e(0x7b7b3d46cb46f6cb), C64e(0xa8a8b71ffc1f4bfc),
    C64e(0x6d6d0c61d661dad6), C64e(0x2c2c624e3a4e583a)
# 定义一个静态的64位无符号整数数组T2
static const sph_u64 T2[] = {
    # 定义64位无符号整数，并初始化
    C64e(0xa5c6c632f4a5f497), C64e(0x84f8f86f978497eb),
    C64e(0x99eeee5eb099b0c7), C64e(0x8df6f67a8c8d8cf7),
    C64e(0x0dffffe8170d17e5), C64e(0xbdd6d60adcbddcb7),
    C64e(0xb1dede16c8b1c8a7), C64e(0x5491916dfc54fc39),
    C64e(0x50606090f050f0c0), C64e(0x0302020705030504),
    C64e(0xa9cece2ee0a9e087), C64e(0x7d5656d1877d87ac),
    C64e(0x19e7e7cc2b192bd5), C64e(0x62b5b513a662a671),
    C64e(0xe64d4d7c31e6319a), C64e(0x9aecec59b59ab5c3),
    C64e(0x458f8f40cf45cf05), C64e(0x9d1f1fa3bc9dbc3e),
    C64e(0x40898949c040c009), C64e(0x87fafa68928792ef),
    C64e(0x15efefd03f153fc5), C64e(0xebb2b29426eb267f),
    C64e(0xc98e8ece40c94007), C64e(0x0bfbfbe61d0b1ded),
    C64e(0xec41416e2fec2f82), C64e(0x67b3b31aa967a97d),
    C64e(0xfd5f5f431cfd1cbe), C64e(0xea45456025ea258a),
    C64e(0xbf2323f9dabfda46), C64e(0xf753535102f702a6),
    C64e(0x96e4e445a196a1d3), C64e(0x5b9b9b76ed5bed2d),
    C64e(0xc27575285dc25dea), C64e(0x1ce1e1c5241c24d9),
    C64e(0xae3d3dd4e9aee97a), C64e(0x6a4c4cf2be6abe98),
    C64e(0x5a6c6c82ee5aeed8), C64e(0x417e7ebdc341c3fc),
    C64e(0x02f5f5f3060206f1), C64e(0x4f838352d14fd11d),
    C64e(0x5c68688ce45ce4d0), C64e(0xf451515607f407a2),
    C64e(0x34d1d18d5c345cb9), C64e(0x08f9f9e1180818e9),
    C64e(0x93e2e24cae93aedf), C64e(0x73abab3e9573954d),
    C64e(0x53626297f553f5c4), C64e(0x3f2a2a6b413f4154),
    C64e(0x0c08081c140c1410), C64e(0x52959563f652f631),
    C64e(0x654646e9af65af8c), C64e(0x5e9d9d7fe25ee221),
    C64e(0x2830304878287860), C64e(0xa13737cff8a1f86e),
    C64e(0x0f0a0a1b110f1114), C64e(0xb52f2febc4b5c45e),
    C64e(0x090e0e151b091b1c), C64e(0x3624247e5a365a48),
    C64e(0x9b1b1badb69bb636), C64e(0x3ddfdf98473d47a5),
    C64e(0x26cdcda76a266a81), C64e(0x694e4ef5bb69bb9c),
    C64e(0xcd7f7f334ccd4cfe), C64e(0x9feaea50ba9fbacf),
    C64e(0x1b12123f2d1b2d24), C64e(0x9e1d1da4b99eb93a),
    C64e(0x745858c49c749cb0), C64e(0x2e343446722e7268),
    C64e(0x2d363641772d776c), C64e(0xb2dcdc11cdb2cda3),
    # 以下是一系列经过 C64e 函数处理的十六进制数值
    C64e(0xeeb4b49d29ee2973), C64e(0xfb5b5b4d16fb16b6),
    C64e(0xf6a4a4a501f60153), C64e(0x4d7676a1d74dd7ec),
    C64e(0x61b7b714a361a375), C64e(0xce7d7d3449ce49fa),
    C64e(0x7b5252df8d7b8da4), C64e(0x3edddd9f423e42a1),
    C64e(0x715e5ecd937193bc), C64e(0x971313b1a297a226),
    C64e(0xf5a6a6a204f50457), C64e(0x68b9b901b868b869),
    C64e(0x0000000000000000), C64e(0x2cc1c1b5742c7499),
    C64e(0x604040e0a060a080), C64e(0x1fe3e3c2211f21dd),
    C64e(0xc879793a43c843f2), C64e(0xedb6b69a2ced2c77),
    C64e(0xbed4d40dd9bed9b3), C64e(0x468d8d47ca46ca01),
    C64e(0xd967671770d970ce), C64e(0x4b7272afdd4bdde4),
    C64e(0xde9494ed79de7933), C64e(0xd49898ff67d4672b),
    C64e(0xe8b0b09323e8237b), C64e(0x4a85855bde4ade11),
    C64e(0x6bbbbb06bd6bbd6d), C64e(0x2ac5c5bb7e2a7e91),
    C64e(0xe54f4f7b34e5349e), C64e(0x16ededd73a163ac1),
    C64e(0xc58686d254c55417), C64e(0xd79a9af862d7622f),
    C64e(0x55666699ff55ffcc), C64e(0x941111b6a794a722),
    C64e(0xcf8a8ac04acf4a0f), C64e(0x10e9e9d9301030c9),
    C64e(0x0604040e0a060a08), C64e(0x81fefe66988198e7),
    C64e(0xf0a0a0ab0bf00b5b), C64e(0x447878b4cc44ccf0),
    C64e(0xba2525f0d5bad54a), C64e(0xe34b4b753ee33e96),
    C64e(0xf3a2a2ac0ef30e5f), C64e(0xfe5d5d4419fe19ba),
    C64e(0xc08080db5bc05b1b), C64e(0x8a050580858a850a),
    C64e(0xad3f3fd3ecadec7e), C64e(0xbc2121fedfbcdf42),
    C64e(0x487070a8d848d8e0), C64e(0x04f1f1fd0c040cf9),
    C64e(0xdf6363197adf7ac6), C64e(0xc177772f58c158ee),
    C64e(0x75afaf309f759f45), C64e(0x634242e7a563a584),
    C64e(0x3020207050305040), C64e(0x1ae5e5cb2e1a2ed1),
    C64e(0x0efdfdef120e12e1), C64e(0x6dbfbf08b76db765),
    C64e(0x4c818155d44cd419), C64e(0x141818243c143c30),
    C64e(0x352626795f355f4c), C64e(0x2fc3c3b2712f719d),
    C64e(0xe1bebe8638e13867), C64e(0xa23535c8fda2fd6a),
    C64e(0xcc8888c74fcc4f0b), C64e(0x392e2e654b394b5c),
    C64e(0x5793936af957f93d), C64e(0xf25555580df20daa),
    C64e(0x82fcfc619d829de3), C64e(0x477a7ab3c947c9f4),
    # 使用C64e函数对给定的十六进制数进行加密
    C64e(0xacc8c827efacef8b), 
    C64e(0xe7baba8832e7326f),
    C64e(0x2b32324f7d2b7d64), 
    C64e(0x95e6e642a495a4d7),
    C64e(0xa0c0c03bfba0fb9b), 
    C64e(0x981919aab398b332),
    C64e(0xd19e9ef668d16827), 
    C64e(0x7fa3a322817f815d),
    C64e(0x664444eeaa66aa88), 
    C64e(0x7e5454d6827e82a8),
    C64e(0xab3b3bdde6abe676), 
    C64e(0x830b0b959e839e16),
    C64e(0xca8c8cc945ca4503), 
    C64e(0x29c7c7bc7b297b95),
    C64e(0xd36b6b056ed36ed6), 
    C64e(0x3c28286c443c4450),
    C64e(0x79a7a72c8b798b55), 
    C64e(0xe2bcbc813de23d63),
    C64e(0x1d161631271d272c), 
    C64e(0x76adad379a769a41),
    C64e(0x3bdbdb964d3b4dad), 
    C64e(0x5664649efa56fac8),
    C64e(0x4e7474a6d24ed2e8), 
    C64e(0x1e141436221e2228),
    C64e(0xdb9292e476db763f), 
    C64e(0x0a0c0c121e0a1e18),
    C64e(0x6c4848fcb46cb490), 
    C64e(0xe4b8b88f37e4376b),
    C64e(0x5d9f9f78e75de725), 
    C64e(0x6ebdbd0fb26eb261),
    C64e(0xef4343692aef2a86), 
    C64e(0xa6c4c435f1a6f193),
    C64e(0xa83939dae3a8e372), 
    C64e(0xa43131c6f7a4f762),
    C64e(0x37d3d38a593759bd), 
    C64e(0x8bf2f274868b86ff),
    C64e(0x32d5d583563256b1), 
    C64e(0x438b8b4ec543c50d),
    C64e(0x596e6e85eb59ebdc), 
    C64e(0xb7dada18c2b7c2af),
    C64e(0x8c01018e8f8c8f02), 
    C64e(0x64b1b11dac64ac79),
    C64e(0xd29c9cf16dd26d23), 
    C64e(0xe04949723be03b92),
    C64e(0xb4d8d81fc7b4c7ab), 
    C64e(0xfaacacb915fa1543),
    C64e(0x07f3f3fa090709fd), 
    C64e(0x25cfcfa06f256f85),
    C64e(0xafcaca20eaafea8f), 
    C64e(0x8ef4f47d898e89f3),
    C64e(0xe947476720e9208e), 
    C64e(0x1810103828182820),
    C64e(0xd56f6f0b64d564de), 
    C64e(0x88f0f073838883fb),
    C64e(0x6f4a4afbb16fb194), 
    C64e(0x725c5cca967296b8),
    C64e(0x243838546c246c70), 
    C64e(0xf157575f08f108ae),
    C64e(0xc773732152c752e6), 
    C64e(0x51979764f351f335),
    C64e(0x23cbcbae6523658d), 
    C64e(0x7ca1a125847c8459),
    C64e(0x9ce8e857bf9cbfcb), 
    C64e(0x213e3e5d6321637c),
    C64e(0xdd9696ea7cdd7c37), 
    C64e(0xdc61611e7fdc7fc2),
    C64e(0x860d0d9c9186911a), 
    C64e(0x850f0f9b9485941e),
    C64e(0x90e0e04bab90abdb), 
    C64e(0x427c7cbac642c6f8),
    # 这些代码看起来是一系列的函数调用，但是缺少上下文，无法确定具体的作用和含义
    # 需要更多的上下文信息才能添加合适的注释
# 定义静态常量数组 T3
static const sph_u64 T3[] = {
    # 定义一系列64位常量，并用C64e宏进行转换
    C64e(0x97a5c6c632f4a5f4), C64e(0xeb84f8f86f978497),
    C64e(0xc799eeee5eb099b0), C64e(0xf78df6f67a8c8d8c),
    C64e(0xe50dffffe8170d17), C64e(0xb7bdd6d60adcbddc),
    C64e(0xa7b1dede16c8b1c8), C64e(0x395491916dfc54fc),
    C64e(0xc050606090f050f0), C64e(0x0403020207050305),
    C64e(0x87a9cece2ee0a9e0), C64e(0xac7d5656d1877d87),
    C64e(0xd519e7e7cc2b192b), C64e(0x7162b5b513a662a6),
    C64e(0x9ae64d4d7c31e631), C64e(0xc39aecec59b59ab5),
    C64e(0x05458f8f40cf45cf), C64e(0x3e9d1f1fa3bc9dbc),
    C64e(0x0940898949c040c0), C64e(0xef87fafa68928792),
    C64e(0xc515efefd03f153f), C64e(0x7febb2b29426eb26),
    C64e(0x07c98e8ece40c940), C64e(0xed0bfbfbe61d0b1d),
    C64e(0x82ec41416e2fec2f), C64e(0x7d67b3b31aa967a9),
    C64e(0xbefd5f5f431cfd1c), C64e(0x8aea45456025ea25),
    C64e(0x46bf2323f9dabfda), C64e(0xa6f753535102f702),
    C64e(0xd396e4e445a196a1), C64e(0x2d5b9b9b76ed5bed),
    C64e(0xeac27575285dc25d), C64e(0xd91ce1e1c5241c24),
    C64e(0x7aae3d3dd4e9aee9), C64e(0x986a4c4cf2be6abe),
    C64e(0xd85a6c6c82ee5aee), C64e(0xfc417e7ebdc341c3),
    C64e(0xf102f5f5f3060206), C64e(0x1d4f838352d14fd1),
    C64e(0xd05c68688ce45ce4), C64e(0xa2f451515607f407),
    C64e(0xb934d1d18d5c345c), C64e(0xe908f9f9e1180818),
    C64e(0xdf93e2e24cae93ae), C64e(0x4d73abab3e957395),
    C64e(0xc453626297f553f5), C64e(0x543f2a2a6b413f41),
    C64e(0x100c08081c140c14), C64e(0x3152959563f652f6),
    C64e(0x8c654646e9af65af), C64e(0x215e9d9d7fe25ee2),
    C64e(0x6028303048782878), C64e(0x6ea13737cff8a1f8),
    C64e(0x140f0a0a1b110f11), C64e(0x5eb52f2febc4b5c4),
    C64e(0x1c090e0e151b091b), C64e(0x483624247e5a365a),
    C64e(0x369b1b1badb69bb6), C64e(0xa53ddfdf98473d47),
    C64e(0x8126cdcda76a266a), C64e(0x9c694e4ef5bb69bb),
    C64e(0xfecd7f7f334ccd4c), C64e(0xcf9feaea50ba9fba),
    C64e(0x241b12123f2d1b2d), C64e(0x3a9e1d1da4b99eb9),
    C64e(0xb0745858c49c749c), C64e(0x682e343446722e72),
    C64e(0x6c2d363641772d77), C64e(0xa3b2dcdc11cdb2cd),
    # 使用C64e函数对给定的十六进制数进行加密
    C64e(0x73eeb4b49d29ee29), C64e(0xb6fb5b5b4d16fb16),
    C64e(0x53f6a4a4a501f601), C64e(0xec4d7676a1d74dd7),
    C64e(0x7561b7b714a361a3), C64e(0xface7d7d3449ce49),
    C64e(0xa47b5252df8d7b8d), C64e(0xa13edddd9f423e42),
    C64e(0xbc715e5ecd937193), C64e(0x26971313b1a297a2),
    C64e(0x57f5a6a6a204f504), C64e(0x6968b9b901b868b8),
    C64e(0x0000000000000000), C64e(0x992cc1c1b5742c74),
    C64e(0x80604040e0a060a0), C64e(0xdd1fe3e3c2211f21),
    C64e(0xf2c879793a43c843), C64e(0x77edb6b69a2ced2c),
    C64e(0xb3bed4d40dd9bed9), C64e(0x01468d8d47ca46ca),
    C64e(0xced967671770d970), C64e(0xe44b7272afdd4bdd),
    C64e(0x33de9494ed79de79), C64e(0x2bd49898ff67d467),
    C64e(0x7be8b0b09323e823), C64e(0x114a85855bde4ade),
    C64e(0x6d6bbbbb06bd6bbd), C64e(0x912ac5c5bb7e2a7e),
    C64e(0x9ee54f4f7b34e534), C64e(0xc116ededd73a163a),
    C64e(0x17c58686d254c554), C64e(0x2fd79a9af862d762),
    C64e(0xcc55666699ff55ff), C64e(0x22941111b6a794a7),
    C64e(0x0fcf8a8ac04acf4a), C64e(0xc910e9e9d9301030),
    C64e(0x080604040e0a060a), C64e(0xe781fefe66988198),
    C64e(0x5bf0a0a0ab0bf00b), C64e(0xf0447878b4cc44cc),
    C64e(0x4aba2525f0d5bad5), C64e(0x96e34b4b753ee33e),
    C64e(0x5ff3a2a2ac0ef30e), C64e(0xbafe5d5d4419fe19),
    C64e(0x1bc08080db5bc05b), C64e(0x0a8a050580858a85),
    C64e(0x7ead3f3fd3ecadec), C64e(0x42bc2121fedfbcdf),
    C64e(0xe0487070a8d848d8), C64e(0xf904f1f1fd0c040c),
    C64e(0xc6df6363197adf7a), C64e(0xeec177772f58c158),
    C64e(0x4575afaf309f759f), C64e(0x84634242e7a563a5),
    C64e(0x4030202070503050), C64e(0xd11ae5e5cb2e1a2e),
    C64e(0xe10efdfdef120e12), C64e(0x656dbfbf08b76db7),
    C64e(0x194c818155d44cd4), C64e(0x30141818243c143c),
    C64e(0x4c352626795f355f), C64e(0x9d2fc3c3b2712f71),
    C64e(0x67e1bebe8638e138), C64e(0x6aa23535c8fda2fd),
    C64e(0x0bcc8888c74fcc4f), C64e(0x5c392e2e654b394b),
    C64e(0x3d5793936af957f9), C64e(0xaaf25555580df20d),
    C64e(0xe382fcfc619d829d), C64e(0xf4477a7ab3c947c9),
    # 使用C64e函数对0x8bacc8c827efacef进行加密
    C64e(0x8bacc8c827efacef), 
    # 使用C64e函数对0x6fe7baba8832e732进行加密
    C64e(0x6fe7baba8832e732),
    # 使用C64e函数对0x642b32324f7d2b7d进行加密
    C64e(0x642b32324f7d2b7d), 
    # 使用C64e函数对0xd795e6e642a495a4进行加密
    C64e(0xd795e6e642a495a4),
    # 使用C64e函数对0x9ba0c0c03bfba0fb进行加密
    C64e(0x9ba0c0c03bfba0fb), 
    # 使用C64e函数对0x32981919aab398b3进行加密
    C64e(0x32981919aab398b3),
    # 使用C64e函数对0x27d19e9ef668d168进行加密
    C64e(0x27d19e9ef668d168), 
    # 使用C64e函数对0x5d7fa3a322817f81进行加密
    C64e(0x5d7fa3a322817f81),
    # 使用C64e函数对0x88664444eeaa66aa进行加密
    C64e(0x88664444eeaa66aa), 
    # 使用C64e函数对0xa87e5454d6827e82进行加密
    C64e(0xa87e5454d6827e82),
    # 使用C64e函数对0x76ab3b3bdde6abe6进行加密
    C64e(0x76ab3b3bdde6abe6), 
    # 使用C64e函数对0x16830b0b959e839e进行加密
    C64e(0x16830b0b959e839e),
    # 使用C64e函数对0x03ca8c8cc945ca45进行加密
    C64e(0x03ca8c8cc945ca45), 
    # 使用C64e函数对0x9529c7c7bc7b297b进行加密
    C64e(0x9529c7c7bc7b297b),
    # 使用C64e函数对0xd6d36b6b056ed36e进行加密
    C64e(0xd6d36b6b056ed36e), 
    # 使用C64e函数对0x503c28286c443c44进行加密
    C64e(0x503c28286c443c44),
    # 使用C64e函数对0x5579a7a72c8b798b进行加密
    C64e(0x5579a7a72c8b798b), 
    # 使用C64e函数对0x63e2bcbc813de23d进行加密
    C64e(0x63e2bcbc813de23d),
    # 使用C64e函数对0x2c1d161631271d27进行加密
    C64e(0x2c1d161631271d27), 
    # 使用C64e函数对0x4176adad379a769a进行加密
    C64e(0x4176adad379a769a),
    # 使用C64e函数对0xad3bdbdb964d3b4d进行加密
    C64e(0xad3bdbdb964d3b4d), 
    # 使用C64e函数对0xc85664649efa56fa进行加密
    C64e(0xc85664649efa56fa),
    # 使用C64e函数对0xe84e7474a6d24ed2进行加密
    C64e(0xe84e7474a6d24ed2), 
    # 使用C64e函数对0x281e141436221e22进行加密
    C64e(0x281e141436221e22),
    # 使用C64e函数对0x3fdb9292e476db76进行加密
    C64e(0x3fdb9292e476db76), 
    # 使用C64e函数对0x180a0c0c121e0a1e进行加密
    C64e(0x180a0c0c121e0a1e),
    # 使用C64e函数对0x906c4848fcb46cb4进行加密
    C64e(0x906c4848fcb46cb4), 
    # 使用C64e函数对0x6be4b8b88f37e437进行加密
    C64e(0x6be4b8b88f37e437),
    # 使用C64e函数对0x255d9f9f78e75de7进行加密
    C64e(0x255d9f9f78e75de7), 
    # 使用C64e函数对0x616ebdbd0fb26eb2进行加密
    C64e(0x616ebdbd0fb26eb2),
    # 使用C64e函数对0x86ef4343692aef2a进行加密
    C64e(0x86ef4343692aef2a), 
    # 使用C64e函数对0x93a6c4c435f1a6f1进行加密
    C64e(0x93a6c4c435f1a6f1),
    # 使用C64e函数对0x72a83939dae3a8e3进行加密
    C64e(0x72a83939dae3a8e3), 
    # 使用C64e函数对0x62a43131c6f7a4f7进行加密
    C64e(0x62a43131c6f7a4f7),
    # 使用C64e函数对0xbd37d3d38a593759进行加密
    C64e(0xbd37d3d38a593759), 
    # 使用C64e函数对0xff8bf2f274868b86进行加密
    C64e(0xff8bf2f274868b86),
    # 使用C64e函数对0xb132d5d583563256进行加密
    C64e(0xb132d5d583563256), 
    # 使用C64e函数对0x0d438b8b4ec543c5进行加密
    C64e(0x0d438b8b4ec543c5),
    # 使用C64e函数对0xdc596e6e85eb59eb进行加密
    C64e(0xdc596e6e85eb59eb), 
    # 使用C64e函数对0xafb7dada18c2b7c2进行加密
    C64e(0xafb7dada18c2b7c2),
    # 使用C64e函数对0x028c01018e8f8c8f进行加密
    C64e(0x028c01018e8f8c8f), 
    # 使用C64e函数对0x7964b1b11dac64ac进行加密
    C64e(0x7964b1b11dac64ac),
    # 使用C64e函数对0x23d29c9cf16dd26d进行加密
    C64e(0x23d29c9cf16dd26d), 
    # 使用C64e函数对0x92e04949723be03b进行加密
    C64e(0x92e04949723be03b),
    # 使用C64e函数对0xabb4d8d81fc7b4c7进行加密
    C64e(0xabb4d8d81fc7b4c7), 
    # 使用C64e函数对0x43faacacb915fa15进行加密
    C64e(0x43faacacb915fa15),
    # 使用C64e函数对0xfd07f3f3fa090709进行加密
    C64e(0xfd07f3f3fa090709), 
    # 使用C64e函数对0x8525cfcfa06f256f进行加密
    C64e(0x8525cfcfa06f256f),
    # 使用C64e函数对0x8fafcaca20eaafea进行加密
    C64e(0x8fafcaca20eaafea), 
    # 使用C64e函数对0xf38ef4f47d898e89进行加密
    C64e(0xf38ef4f47d898e89),
    # 使用C64e函数对0x8ee947476720e920进行加密
    C64e(0x8ee947476720e920), 
    # 使用C64e函数对0x2018101038281828进行加密
    C64e(0x2018101038281828),
    # 使用C64e函数对0xded56f6f0b64d564进行加密
    C64e(0xded56f6f0b64d564), 
    # 使用C64e函数对0xfb88f0f073838883进行加密
    C64e(0xfb88f0f073838883),
    # 使用C64e函数对0x946f4a4afbb16fb1进行加密
    C64e(0x946f4a4afbb16fb1), 
    # 使用C64e函数对0xb8725c5cca967296进行加密
    C64e(0xb8725c5cca967296),
    # 使用C64e函数对0x70243838546c246c进行加密
    C64e(0x70243838546c246c), 
    # 使用C64e函数对0xaef157575f08f108进行加密
    C64e(0xaef157575f08f108),
    # 使用C64e函数对0xe6c773732152c752进行加密
    C64e(0xe6c773732152c752), 
    # 使用C64e函数对0x3551979764f351f3进行加密
    C64e(0x3551979764f351f3),
    # 使用C64e函数对0x8d23cbcbae652365进行加密
    C64e(0x8d23cbcbae652365), 
    # 使用C64e函数对0x597ca1a125847c84进行加密
    C64e(0x597ca1a125847c84),
    # 使用C64e函数对0xcb9ce8e857bf9cbf进行加密
    C64e(0xcb9ce8e857bf9cbf), 
    # 使用C64e函数对0x7c213e3e5d632163进行加密
    C64e(0x7c213e3e5d632163),
    # 使用C64e函数对0x37dd9696ea7cdd7c进行加密
    C64e(0x37dd9696ea7cdd7c), 
    # 使用C64e函数对0xc2dc61611e7fdc7f进行加密
    C64e(0xc2dc61611e7fdc7f),
    # 使用C64e函数对0x1a860d0d9c918691进行加密
    C64e(0x1a860d0d9c918691), 
    # 使用C64e函数对0x1e850f0f9b948594进行加密
    C64e(0x1e850f0f9b948594),
    # 使用C64e函数对0xdb90e0e04bab90ab进行加密
    C64e(0xdb90e0e04bab90ab), 
    # 使用C64e函数对0xf8427c7cbac642c6进行加密
    C64e(0xf8427c7cbac642c6),
    # 使用C64e函数对0xe2c471712657c457进行加密
    C64e(0xe2c471712657c457), 
    # 使用C64e函数对0x83aacccc29e5aae5进行加密
    C64e(0x83aacccc29e5aae5),
    # 使用C64e函数对0x3bd89090e373d873进行加密
    C64e(0x3bd89090e373d873), 
    # 使用C64e函数对0x0c050606090f050f进行加密
    C64e(0x0c050606090f050f),
    # 使用C64e函数对0xf501f7f7f4030103进行加密
    C64e(0xf501f7f7f4030103), 
    # 使用C64e函数对0x38121c1c2a361236进行加密
    C64e(0x38121c1c2a361236),
    # 使用C64e函数对0x9fa3c2c23cfea3fe进行加密
    C64e(0x9fa3c2c23cfea3fe), 
    # 使用C64e函数对0xd45f6a6a8be15fe1进行加密
    C64e(0xd45f6a6a8be15fe1),
    # 使用C64e函数对0x47f9aeaebe10f910进行加密
    C64e(0x47f9aeaebe10f910), 
    # 使用C64e函数对0xd2d06969026bd06b进行加密
    C64e(0xd2d06969026bd06b),
    # 使用C64e函数对0x2e911717bfa891a8进行加密
    C64e(0x2e911717bfa891a8), 
    # 使用C64e函数对0x2958999971e858e8进行加密
    C64e(0x2958999971e858e8),
    # 使用C64e函数对0x74273a3a53692769进行加密
    C64e(0x74273a3a53692769), 
    # 使用C64e函数对0x4eb92727f7d0b9d0进行加密
    C64e(0x4eb92727f7d0b9d0),
    # 使用C64e函数对0xa938d9d991483848进行加密
    C64e(0xa938d9d991483848), 
    # 使用C64e函数对0xcd13ebebde351335进行加密
    C64e(0xcd13ebebde351335),
    # 使用C64e函数对0x56b32b2be5ceb3ce进行加密
    C64e(0x56b32b2be5ceb3ce), 
    # 使用C64e函数对0x4433222277553355进行加密
    C64e(0x4433222277553355),
    # 使用C64e函数对0xbfbbd2d204d6bbd6进行加密
    C64e(0xbfbbd2d204d6bbd6), 
    # 使用C64e函数对0x4970a9a939907090进行加密
    C64e(0x4970a9a939907090),
    # 使用C64e函数对0x0e89070787808980进行加密
    C64e(0x0e89070787808980), 
    # 使用C64e函数对0x66a73333c1f2a7f2进行加密
    C64e(0x66a73333c1f2a7f2),
    # 使用C64e函数对0x5ab62d2decc1b6c1进行加密
    C64e(0x5ab62d2decc1b6c1), 
    # 使用C64e函数对0x78223c3c5a662266进行加密
    C64e(0x78223c3c5a662266),
    # 使用C64e函数对0x2a921515b8ad92ad进行加密
    C64e(0x2a921515b8ad92ad), 
    # 使用C64e函数对0x8920c9c9a9602060进行加密
    C64e(0x8920c9c9a9602060),
    # 使用C64e函数对0x154987875cdb49db进行加密
    C64e(0x154987875cdb49db), 
    # 使用C64e函数对0x4fffaaaab01aff1a进行加密
    C64e(0x4fffaaaab01aff1a),
    # 使用C64e函数对0xa0785050d8887888进行加密
    C64e(0xa0785050d8887888), 
    # 使用C64e函数对0x517aa5a52b8e7a8e进行加密
    C64e(0x517aa5a52b8e7a8e),
    # 使用C64e函数对0x068f0303898a8f8a进行加密
    C64e(0x068f0303898a8f8a), 
    # 使用C64e函数对0xb2f859594a13f813进行加密
    C64e(0xb2f859594a13f813),
    # 使用C64e函数对0x12800909929b809b进行加密
    C64e(0x12800909929b809b), 
    # 使用C64e函数对0x34171a1a23391739进行加密
    C64e(0x34171a1a23391739),
    # 使用C64e函数对0xcada65651075da75进行加密
    C64e(0xcada65651075da75), 
    # 使用C64e函数对0xb531d7d784533153进行加密
    C64e(0xb531d7d784533153),
    # 使用C64e函数对0x13c68484d551c651进行加密
    C64e(0x13c68484d551c651), 
    # 使用C64e函数对0xbbb8d0d003d3b8d3进行加密
    C64e(0xbbb8d0d003d3b8d3),
    # 使用C64e函数对0x1fc38282dc5ec35e进行加密
    C64e(0x1fc38282dc5ec35e), 
    # 使用C64e函数对0x52b02929e2cbb0cb进行加密
    C64e(0x52b02929e2cbb0cb),
    # 使用C64e函数对0xb4775a5ac3997799进行加密
    C64e(0xb4775a5ac3997799), 
    # 使用C64e函数对0x3c111e1e2d331133进行加密
    C64e(0x3c111e1e2d331133),
    # 使用C64e函数对0xf6cb7b7b3d46cb46进行加密
    C64e(0xf6cb7b7b3d46cb46), 
    # 使用C64e函数对0x4bfca8a8b71ffc1f进行加密
    C64e(0x4bfca8a8b71ffc1f),
    # 使用C64e函数对0xdad66d6d0c61d661进行加密
    C64e(0xdad66d6d0c61d661), 
    # 使用C64e函数对0x583a2c2c624e3a4e进行加密
    C64e(0x583a2c2c624e3a4e)
// 定义一个静态的64位无符号整数数组T4
static const sph_u64 T4[] = {
    // 以下为数组元素的赋值
    C64e(0xf497a5c6c632f4a5), C64e(0x97eb84f8f86f9784),
    C64e(0xb0c799eeee5eb099), C64e(0x8cf78df6f67a8c8d),
    C64e(0x17e50dffffe8170d), C64e(0xdcb7bdd6d60adcbd),
    C64e(0xc8a7b1dede16c8b1), C64e(0xfc395491916dfc54),
    C64e(0xf0c050606090f050), C64e(0x0504030202070503),
    C64e(0xe087a9cece2ee0a9), C64e(0x87ac7d5656d1877d),
    C64e(0x2bd519e7e7cc2b19), C64e(0xa67162b5b513a662),
    C64e(0x319ae64d4d7c31e6), C64e(0xb5c39aecec59b59a),
    C64e(0xcf05458f8f40cf45), C64e(0xbc3e9d1f1fa3bc9d),
    C64e(0xc00940898949c040), C64e(0x92ef87fafa689287),
    C64e(0x3fc515efefd03f15), C64e(0x267febb2b29426eb),
    C64e(0x4007c98e8ece40c9), C64e(0x1ded0bfbfbe61d0b),
    C64e(0x2f82ec41416e2fec), C64e(0xa97d67b3b31aa967),
    C64e(0x1cbefd5f5f431cfd), C64e(0x258aea45456025ea),
    C64e(0xda46bf2323f9dabf), C64e(0x02a6f753535102f7),
    C64e(0xa1d396e4e445a196), C64e(0xed2d5b9b9b76ed5b),
    C64e(0x5deac27575285dc2), C64e(0x24d91ce1e1c5241c),
    C64e(0xe97aae3d3dd4e9ae), C64e(0xbe986a4c4cf2be6a),
    C64e(0xeed85a6c6c82ee5a), C64e(0xc3fc417e7ebdc341),
    C64e(0x06f102f5f5f30602), C64e(0xd11d4f838352d14f),
    C64e(0xe4d05c68688ce45c), C64e(0x07a2f451515607f4),
    C64e(0x5cb934d1d18d5c34), C64e(0x18e908f9f9e11808),
    C64e(0xaedf93e2e24cae93), C64e(0x954d73abab3e9573),
    C64e(0xf5c453626297f553), C64e(0x41543f2a2a6b413f),
    C64e(0x14100c08081c140c), C64e(0xf63152959563f652),
    C64e(0xaf8c654646e9af65), C64e(0xe2215e9d9d7fe25e),
    C64e(0x7860283030487828), C64e(0xf86ea13737cff8a1),
    C64e(0x11140f0a0a1b110f), C64e(0xc45eb52f2febc4b5),
    C64e(0x1b1c090e0e151b09), C64e(0x5a483624247e5a36),
    C64e(0xb6369b1b1badb69b), C64e(0x47a53ddfdf98473d),
    C64e(0x6a8126cdcda76a26), C64e(0xbb9c694e4ef5bb69),
    C64e(0x4cfecd7f7f334ccd), C64e(0xbacf9feaea50ba9f),
    C64e(0x2d241b12123f2d1b), C64e(0xb93a9e1d1da4b99e),
    C64e(0x9cb0745858c49c74), C64e(0x72682e343446722e),
    // 以下为数组的结尾
};
#endif
    # 使用C64e函数对给定的十六进制数进行解密
    C64e(0x776c2d363641772d), C64e(0xcda3b2dcdc11cdb2),
    C64e(0x2973eeb4b49d29ee), C64e(0x16b6fb5b5b4d16fb),
    C64e(0x0153f6a4a4a501f6), C64e(0xd7ec4d7676a1d74d),
    C64e(0xa37561b7b714a361), C64e(0x49face7d7d3449ce),
    C64e(0x8da47b5252df8d7b), C64e(0x42a13edddd9f423e),
    C64e(0x93bc715e5ecd9371), C64e(0xa226971313b1a297),
    C64e(0x0457f5a6a6a204f5), C64e(0xb86968b9b901b868),
    C64e(0x0000000000000000), C64e(0x74992cc1c1b5742c),
    C64e(0xa080604040e0a060), C64e(0x21dd1fe3e3c2211f),
    C64e(0x43f2c879793a43c8), C64e(0x2c77edb6b69a2ced),
    C64e(0xd9b3bed4d40dd9be), C64e(0xca01468d8d47ca46),
    C64e(0x70ced967671770d9), C64e(0xdde44b7272afdd4b),
    C64e(0x7933de9494ed79de), C64e(0x672bd49898ff67d4),
    C64e(0x237be8b0b09323e8), C64e(0xde114a85855bde4a),
    C64e(0xbd6d6bbbbb06bd6b), C64e(0x7e912ac5c5bb7e2a),
    C64e(0x349ee54f4f7b34e5), C64e(0x3ac116ededd73a16),
    C64e(0x5417c58686d254c5), C64e(0x622fd79a9af862d7),
    C64e(0xffcc55666699ff55), C64e(0xa722941111b6a794),
    C64e(0x4a0fcf8a8ac04acf), C64e(0x30c910e9e9d93010),
    C64e(0x0a080604040e0a06), C64e(0x98e781fefe669881),
    C64e(0x0b5bf0a0a0ab0bf0), C64e(0xccf0447878b4cc44),
    C64e(0xd54aba2525f0d5ba), C64e(0x3e96e34b4b753ee3),
    C64e(0x0e5ff3a2a2ac0ef3), C64e(0x19bafe5d5d4419fe),
    C64e(0x5b1bc08080db5bc0), C64e(0x850a8a050580858a),
    C64e(0xec7ead3f3fd3ecad), C64e(0xdf42bc2121fedfbc),
    C64e(0xd8e0487070a8d848), C64e(0x0cf904f1f1fd0c04),
    C64e(0x7ac6df6363197adf), C64e(0x58eec177772f58c1),
    C64e(0x9f4575afaf309f75), C64e(0xa584634242e7a563),
    C64e(0x5040302020705030), C64e(0x2ed11ae5e5cb2e1a),
    C64e(0x12e10efdfdef120e), C64e(0xb7656dbfbf08b76d),
    C64e(0xd4194c818155d44c), C64e(0x3c30141818243c14),
    C64e(0x5f4c352626795f35), C64e(0x719d2fc3c3b2712f),
    C64e(0x3867e1bebe8638e1), C64e(0xfd6aa23535c8fda2),
    C64e(0x4f0bcc8888c74fcc), C64e(0x4b5c392e2e654b39),
    C64e(0xf93d5793936af957), C64e(0x0daaf25555580df2),
    # 使用C64e函数对0x9de382fcfc619d82进行加密
    C64e(0x9de382fcfc619d82), 
    # 使用C64e函数对0xc9f4477a7ab3c947进行加密
    C64e(0xc9f4477a7ab3c947),
    # 使用C64e函数对0xef8bacc8c827efac进行加密
    C64e(0xef8bacc8c827efac), 
    # 使用C64e函数对0x326fe7baba8832e7进行加密
    C64e(0x326fe7baba8832e7),
    # 使用C64e函数对0x7d642b32324f7d2b进行加密
    C64e(0x7d642b32324f7d2b), 
    # 使用C64e函数对0xa4d795e6e642a495进行加密
    C64e(0xa4d795e6e642a495),
    # 使用C64e函数对0xfb9ba0c0c03bfba0进行加密
    C64e(0xfb9ba0c0c03bfba0), 
    # 使用C64e函数对0xb332981919aab398进行加密
    C64e(0xb332981919aab398),
    # 使用C64e函数对0x6827d19e9ef668d1进行加密
    C64e(0x6827d19e9ef668d1), 
    # 使用C64e函数对0x815d7fa3a322817f进行加密
    C64e(0x815d7fa3a322817f),
    # 使用C64e函数对0xaa88664444eeaa66进行加密
    C64e(0xaa88664444eeaa66), 
    # 使用C64e函数对0x82a87e5454d6827e进行加密
    C64e(0x82a87e5454d6827e),
    # 使用C64e函数对0xe676ab3b3bdde6ab进行加密
    C64e(0xe676ab3b3bdde6ab), 
    # 使用C64e函数对0x9e16830b0b959e83进行加密
    C64e(0x9e16830b0b959e83),
    # 使用C64e函数对0x4503ca8c8cc945ca进行加密
    C64e(0x4503ca8c8cc945ca), 
    # 使用C64e函数对0x7b9529c7c7bc7b29进行加密
    C64e(0x7b9529c7c7bc7b29),
    # 使用C64e函数对0x6ed6d36b6b056ed3进行加密
    C64e(0x6ed6d36b6b056ed3), 
    # 使用C64e函数对0x44503c28286c443c进行加密
    C64e(0x44503c28286c443c),
    # 使用C64e函数对0x8b5579a7a72c8b79进行加密
    C64e(0x8b5579a7a72c8b79), 
    # 使用C64e函数对0x3d63e2bcbc813de2进行加密
    C64e(0x3d63e2bcbc813de2),
    # 使用C64e函数对0x272c1d161631271d进行加密
    C64e(0x272c1d161631271d), 
    # 使用C64e函数对0x9a4176adad379a76进行加密
    C64e(0x9a4176adad379a76),
    # 使用C64e函数对0x4dad3bdbdb964d3b进行加密
    C64e(0x4dad3bdbdb964d3b), 
    # 使用C64e函数对0xfac85664649efa56进行加密
    C64e(0xfac85664649efa56),
    # 使用C64e函数对0xd2e84e7474a6d24e进行加密
    C64e(0xd2e84e7474a6d24e), 
    # 使用C64e函数对0x22281e141436221e进行加密
    C64e(0x22281e141436221e),
    # 使用C64e函数对0x763fdb9292e476db进行加密
    C64e(0x763fdb9292e476db), 
    # 使用C64e函数对0x1e180a0c0c121e0a进行加密
    C64e(0x1e180a0c0c121e0a),
    # 使用C64e函数对0xb4906c4848fcb46c进行加密
    C64e(0xb4906c4848fcb46c), 
    # 使用C64e函数对0x376be4b8b88f37e4进行加密
    C64e(0x376be4b8b88f37e4),
    # 使用C64e函数对0xe7255d9f9f78e75d进行加密
    C64e(0xe7255d9f9f78e75d), 
    # 使用C64e函数对0xb2616ebdbd0fb26e进行加密
    C64e(0xb2616ebdbd0fb26e),
    # 使用C64e函数对0x2a86ef4343692aef进行加密
    C64e(0x2a86ef4343692aef), 
    # 使用C64e函数对0xf193a6c4c435f1a6进行加密
    C64e(0xf193a6c4c435f1a6),
    # 使用C64e函数对0xe372a83939dae3a8进行加密
    C64e(0xe372a83939dae3a8), 
    # 使用C64e函数对0xf762a43131c6f7a4进行加密
    C64e(0xf762a43131c6f7a4),
    # 使用C64e函数对0x59bd37d3d38a5937进行加密
    C64e(0x59bd37d3d38a5937), 
    # 使用C64e函数对0x86ff8bf2f274868b进行加密
    C64e(0x86ff8bf2f274868b),
    # 使用C64e函数对0x56b132d5d5835632进行加密
    C64e(0x56b132d5d5835632), 
    # 使用C64e函数对0xc50d438b8b4ec543进行加密
    C64e(0xc50d438b8b4ec543),
    # 使用C64e函数对0xebdc596e6e85eb59进行加密
    C64e(0xebdc596e6e85eb59), 
    # 使用C64e函数对0xc2afb7dada18c2b7进行加密
    C64e(0xc2afb7dada18c2b7),
    # 使用C64e函数对0x8f028c01018e8f8c进行加密
    C64e(0x8f028c01018e8f8c), 
    # 使用C64e函数对0xac7964b1b11dac64进行加密
    C64e(0xac7964b1b11dac64),
    # 使用C64e函数对0x6d23d29c9cf16dd2进行加密
    C64e(0x6d23d29c9cf16dd2), 
    # 使用C64e函数对0x3b92e04949723be0进行加密
    C64e(0x3b92e04949723be0),
    # 使用C64e函数对0xc7abb4d8d81fc7b4进行加密
    C64e(0xc7abb4d8d81fc7b4), 
    # 使用C64e函数对0x1543faacacb915fa进行加密
    C64e(0x1543faacacb915fa),
    # 使用C64e函数对0x09fd07f3f3fa0907进行加密
    C64e(0x09fd07f3f3fa0907), 
    # 使用C64e函数对0x6f8525cfcfa06f25进行加密
    C64e(0x6f8525cfcfa06f25),
    # 使用C64e函数对0xea8fafcaca20eaaf进行加密
    C64e(0xea8fafcaca20eaaf), 
    # 使用C64e函数对0x89f38ef4f47d898e进行加密
    C64e(0x89f38ef4f47d898e),
    # 使用C64e函数对0x208ee947476720e9进行加密
    C64e(0x208ee947476720e9), 
    # 使用C64e函数对0x2820181010382818进行加密
    C64e(0x2820181010382818),
    # 使用C64e函数对0x64ded56f6f0b64d5进行加密
    C64e(0x64ded56f6f0b64d5), 
    # 使用C64e函数对0x83fb88f0f0738388进行加密
    C64e(0x83fb88f0f0738388),
    # 使用C64e函数对0xb1946f4a4afbb16f进行加密
    C64e(0xb1946f4a4afbb16f), 
    # 使用C64e函数对0x96b8725c5cca9672进行加密
    C64e(0x96b8725c5cca9672),
    # 使用C64e函数对0x6c70243838546c24进行加密
    C64e(0x6c70243838546c24), 
    # 使用C64e函数对0x08aef157575f08f1进行加密
    C64e(0x08aef157575f08f1),
    # 使用C64e函数对0x52e6c773732152c7进行加密
    C64e(0x52e6c773732152c7), 
    # 使用C64e函数对0xf33551979764f351进行加密
    C64e(0xf33551979764f351),
    # 使用C64e函数对0x658d23cbcbae6523进行加密
    C64e(0x658d23cbcbae6523), 
    # 使用C64e函数对0x84597ca1a125847c进行加密
    C64e(0x84597ca1a125847c),
    # 使用C64e函数对0xbfcb9ce8e857bf9c进行加密
    C64e(0xbfcb9ce8e857bf9c), 
    # 使用C64e函数对0x637c213e3e5d6321进行加密
    C64e(0x637c213e3e5d6321),
    # 使用C64e函数对0x7c37dd9696ea7cdd进行加密
    C64e(0x7c37dd9696ea7cdd), 
    # 使用C64e函数对0x7fc2dc61611e7fdc进行加密
    C64e(0x7fc2dc61611e7fdc),
    # 使用C64e函数对0x911a860d0d9c9186进行加密
    C64e(0x911a860d0d9c9186), 
    # 使用C64e函数对0x941e850f0f9b9485进行加密
    C64e(0x941e850f0f9b9485),
    # 以C64e函数对十六进制数进行加密
    C64e(0xabdb90e0e04bab90), C64e(0xc6f8427c7cbac642),
    C64e(0x57e2c471712657c4), C64e(0xe583aacccc29e5aa),
    C64e(0x733bd89090e373d8), C64e(0x0f0c050606090f05),
    C64e(0x03f501f7f7f40301), C64e(0x3638121c1c2a3612),
    C64e(0xfe9fa3c2c23cfea3), C64e(0xe1d45f6a6a8be15f),
    C64e(0x1047f9aeaebe10f9), C64e(0x6bd2d06969026bd0),
    C64e(0xa82e911717bfa891), C64e(0xe82958999971e858),
    C64e(0x6974273a3a536927), C64e(0xd04eb92727f7d0b9),
    C64e(0x48a938d9d9914838), C64e(0x35cd13ebebde3513),
    C64e(0xce56b32b2be5ceb3), C64e(0x5544332222775533),
    C64e(0xd6bfbbd2d204d6bb), C64e(0x904970a9a9399070),
    C64e(0x800e890707878089), C64e(0xf266a73333c1f2a7),
    C64e(0xc15ab62d2decc1b6), C64e(0x6678223c3c5a6622),
    C64e(0xad2a921515b8ad92), C64e(0x608920c9c9a96020),
    C64e(0xdb154987875cdb49), C64e(0x1a4fffaaaab01aff),
    C64e(0x88a0785050d88878), C64e(0x8e517aa5a52b8e7a),
    C64e(0x8a068f0303898a8f), C64e(0x13b2f859594a13f8),
    C64e(0x9b12800909929b80), C64e(0x3934171a1a233917),
    C64e(0x75cada65651075da), C64e(0x53b531d7d7845331),
    C64e(0x5113c68484d551c6), C64e(0xd3bbb8d0d003d3b8),
    C64e(0x5e1fc38282dc5ec3), C64e(0xcb52b02929e2cbb0),
    C64e(0x99b4775a5ac39977), C64e(0x333c111e1e2d3311),
    C64e(0x46f6cb7b7b3d46cb), C64e(0x1f4bfca8a8b71ffc),
    C64e(0x61dad66d6d0c61d6), C64e(0x4e583a2c2c624e3a)
};
# 如果不是小型足迹的 Groestl 算法，则执行以下代码

# 定义 T5 常量数组，包含一系列 64 位无符号整数
static const sph_u64 T5[] = {
    C64e(0xa5f497a5c6c632f4), C64e(0x8497eb84f8f86f97),
    C64e(0x99b0c799eeee5eb0), C64e(0x8d8cf78df6f67a8c),
    C64e(0x0d17e50dffffe817), C64e(0xbddcb7bdd6d60adc),
    C64e(0xb1c8a7b1dede16c8), C64e(0x54fc395491916dfc),
    C64e(0x50f0c050606090f0), C64e(0x0305040302020705),
    C64e(0xa9e087a9cece2ee0), C64e(0x7d87ac7d5656d187),
    C64e(0x192bd519e7e7cc2b), C64e(0x62a67162b5b513a6),
    C64e(0xe6319ae64d4d7c31), C64e(0x9ab5c39aecec59b5),
    C64e(0x45cf05458f8f40cf), C64e(0x9dbc3e9d1f1fa3bc),
    C64e(0x40c00940898949c0), C64e(0x8792ef87fafa6892),
    C64e(0x153fc515efefd03f), C64e(0xeb267febb2b29426),
    C64e(0xc94007c98e8ece40), C64e(0x0b1ded0bfbfbe61d),
    C64e(0xec2f82ec41416e2f), C64e(0x67a97d67b3b31aa9),
    C64e(0xfd1cbefd5f5f431c), C64e(0xea258aea45456025),
    C64e(0xbfda46bf2323f9da), C64e(0xf702a6f753535102),
    C64e(0x96a1d396e4e445a1), C64e(0x5bed2d5b9b9b76ed),
    C64e(0xc25deac27575285d), C64e(0x1c24d91ce1e1c524),
    C64e(0xaee97aae3d3dd4e9), C64e(0x6abe986a4c4cf2be),
    C64e(0x5aeed85a6c6c82ee), C64e(0x41c3fc417e7ebdc3),
    C64e(0x0206f102f5f5f306), C64e(0x4fd11d4f838352d1),
    C64e(0x5ce4d05c68688ce4), C64e(0xf407a2f451515607),
    C64e(0x345cb934d1d18d5c), C64e(0x0818e908f9f9e118),
    C64e(0x93aedf93e2e24cae), C64e(0x73954d73abab3e95),
    C64e(0x53f5c453626297f5), C64e(0x3f41543f2a2a6b41),
    C64e(0x0c14100c08081c14), C64e(0x52f63152959563f6),
    C64e(0x65af8c654646e9af), C64e(0x5ee2215e9d9d7fe2),
    C64e(0x2878602830304878), C64e(0xa1f86ea13737cff8),
    C64e(0x0f11140f0a0a1b11), C64e(0xb5c45eb52f2febc4),
    C64e(0x091b1c090e0e151b), C64e(0x365a483624247e5a),
    C64e(0x9bb6369b1b1badb6), C64e(0x3d47a53ddfdf9847),
    C64e(0x266a8126cdcda76a), C64e(0x69bb9c694e4ef5bb),
    C64e(0xcd4cfecd7f7f334c), C64e(0x9fbacf9feaea50ba),
    C64e(0x1b2d241b12123f2d), C64e(0x9eb93a9e1d1da4b9),
    C64e(0x749cb0745858c49c), C64e(0x2e72682e34344672),
    # 以下是一系列经过 C64e 函数处理的十六进制数值，每个数值都代表一个字节
    # 这些数值可能是密钥、向量或者其他加密算法中使用的参数
    C64e(0x2d776c2d36364177), C64e(0xb2cda3b2dcdc11cd),
    C64e(0xee2973eeb4b49d29), C64e(0xfb16b6fb5b5b4d16),
    C64e(0xf60153f6a4a4a501), C64e(0x4dd7ec4d7676a1d7),
    C64e(0x61a37561b7b714a3), C64e(0xce49face7d7d3449),
    C64e(0x7b8da47b5252df8d), C64e(0x3e42a13edddd9f42),
    C64e(0x7193bc715e5ecd93), C64e(0x97a226971313b1a2),
    C64e(0xf50457f5a6a6a204), C64e(0x68b86968b9b901b8),
    C64e(0x0000000000000000), C64e(0x2c74992cc1c1b574),
    C64e(0x60a080604040e0a0), C64e(0x1f21dd1fe3e3c221),
    C64e(0xc843f2c879793a43), C64e(0xed2c77edb6b69a2c),
    C64e(0xbed9b3bed4d40dd9), C64e(0x46ca01468d8d47ca),
    C64e(0xd970ced967671770), C64e(0x4bdde44b7272afdd),
    C64e(0xde7933de9494ed79), C64e(0xd4672bd49898ff67),
    C64e(0xe8237be8b0b09323), C64e(0x4ade114a85855bde),
    C64e(0x6bbd6d6bbbbb06bd), C64e(0x2a7e912ac5c5bb7e),
    C64e(0xe5349ee54f4f7b34), C64e(0x163ac116ededd73a),
    C64e(0xc55417c58686d254), C64e(0xd7622fd79a9af862),
    C64e(0x55ffcc55666699ff), C64e(0x94a722941111b6a7),
    C64e(0xcf4a0fcf8a8ac04a), C64e(0x1030c910e9e9d930),
    C64e(0x060a080604040e0a), C64e(0x8198e781fefe6698),
    C64e(0xf00b5bf0a0a0ab0b), C64e(0x44ccf0447878b4cc),
    C64e(0xbad54aba2525f0d5), C64e(0xe33e96e34b4b753e),
    C64e(0xf30e5ff3a2a2ac0e), C64e(0xfe19bafe5d5d4419),
    C64e(0xc05b1bc08080db5b), C64e(0x8a850a8a05058085),
    C64e(0xadec7ead3f3fd3ec), C64e(0xbcdf42bc2121fedf),
    C64e(0x48d8e0487070a8d8), C64e(0x040cf904f1f1fd0c),
    C64e(0xdf7ac6df6363197a), C64e(0xc158eec177772f58),
    C64e(0x759f4575afaf309f), C64e(0x63a584634242e7a5),
    C64e(0x3050403020207050), C64e(0x1a2ed11ae5e5cb2e),
    C64e(0x0e12e10efdfdef12), C64e(0x6db7656dbfbf08b7),
    C64e(0x4cd4194c818155d4), C64e(0x143c30141818243c),
    C64e(0x355f4c352626795f), C64e(0x2f719d2fc3c3b271),
    C64e(0xe13867e1bebe8638), C64e(0xa2fd6aa23535c8fd),
    C64e(0xcc4f0bcc8888c74f), C64e(0x394b5c392e2e654b),
    C64e(0x57f93d5793936af9), C64e(0xf20daaf25555580d),
    # 这些代码看起来是一系列的函数调用，每个函数调用都使用了一个十六进制数作为参数
    # 由于缺少上下文，无法确定这些函数的具体作用，需要查看函数定义或者上下文来解释这些函数调用的含义
    # 如果这些函数调用是加密相关的，那么这些十六进制数可能是密钥或者其他加密参数
    # 如果这些函数调用是哈希相关的，那么这些十六进制数可能是待哈希的数据
    # 总之，需要更多的上下文信息才能准确解释这些函数调用的含义
    # 使用C64e函数对0x90abdb90e0e04bab进行加密
    C64e(0x90abdb90e0e04bab), 
    # 使用C64e函数对0x42c6f8427c7cbac6进行加密
    C64e(0x42c6f8427c7cbac6),
    # 使用C64e函数对0xc457e2c471712657进行加密
    C64e(0xc457e2c471712657), 
    # 使用C64e函数对0xaae583aacccc29e5进行加密
    C64e(0xaae583aacccc29e5),
    # 使用C64e函数对0xd8733bd89090e373进行加密
    C64e(0xd8733bd89090e373), 
    # 使用C64e函数对0x050f0c050606090f进行加密
    C64e(0x050f0c050606090f),
    # 使用C64e函数对0x0103f501f7f7f403进行加密
    C64e(0x0103f501f7f7f403), 
    # 使用C64e函数对0x123638121c1c2a36进行加密
    C64e(0x123638121c1c2a36),
    # 使用C64e函数对0xa3fe9fa3c2c23cfe进行加密
    C64e(0xa3fe9fa3c2c23cfe), 
    # 使用C64e函数对0x5fe1d45f6a6a8be1进行加密
    C64e(0x5fe1d45f6a6a8be1),
    # 使用C64e函数对0xf91047f9aeaebe10进行加密
    C64e(0xf91047f9aeaebe10), 
    # 使用C64e函数对0xd06bd2d06969026b进行加密
    C64e(0xd06bd2d06969026b),
    # 使用C64e函数对0x91a82e911717bfa8进行加密
    C64e(0x91a82e911717bfa8), 
    # 使用C64e函数对0x58e82958999971e8进行加密
    C64e(0x58e82958999971e8),
    # 使用C64e函数对0x276974273a3a5369进行加密
    C64e(0x276974273a3a5369), 
    # 使用C64e函数对0xb9d04eb92727f7d0进行加密
    C64e(0xb9d04eb92727f7d0),
    # 使用C64e函数对0x3848a938d9d99148进行加密
    C64e(0x3848a938d9d99148), 
    # 使用C64e函数对0x1335cd13ebebde35进行加密
    C64e(0x1335cd13ebebde35),
    # 使用C64e函数对0xb3ce56b32b2be5ce进行加密
    C64e(0xb3ce56b32b2be5ce), 
    # 使用C64e函数对0x3355443322227755进行加密
    C64e(0x3355443322227755),
    # 使用C64e函数对0xbbd6bfbbd2d204d6进行加密
    C64e(0xbbd6bfbbd2d204d6), 
    # 使用C64e函数对0x70904970a9a93990进行加密
    C64e(0x70904970a9a93990),
    # 使用C64e函数对0x89800e8907078780进行加密
    C64e(0x89800e8907078780), 
    # 使用C64e函数对0xa7f266a73333c1f2进行加密
    C64e(0xa7f266a73333c1f2),
    # 使用C64e函数对0xb6c15ab62d2decc1进行加密
    C64e(0xb6c15ab62d2decc1), 
    # 使用C64e函数对0x226678223c3c5a66进行加密
    C64e(0x226678223c3c5a66),
    # 使用C64e函数对0x92ad2a921515b8ad进行加密
    C64e(0x92ad2a921515b8ad), 
    # 使用C64e函数对0x20608920c9c9a960进行加密
    C64e(0x20608920c9c9a960),
    # 使用C64e函数对0x49db154987875cdb进行加密
    C64e(0x49db154987875cdb), 
    # 使用C64e函数对0xff1a4fffaaaab01a进行加密
    C64e(0xff1a4fffaaaab01a),
    # 使用C64e函数对0x7888a0785050d888进行加密
    C64e(0x7888a0785050d888), 
    # 使用C64e函数对0x7a8e517aa5a52b8e进行加密
    C64e(0x7a8e517aa5a52b8e),
    # 使用C64e函数对0x8f8a068f0303898a进行加密
    C64e(0x8f8a068f0303898a), 
    # 使用C64e函数对0xf813b2f859594a13进行加密
    C64e(0xf813b2f859594a13),
    # 使用C64e函数对0x809b12800909929b进行加密
    C64e(0x809b12800909929b), 
    # 使用C64e函数对0x173934171a1a2339进行加密
    C64e(0x173934171a1a2339),
    # 使用C64e函数对0xda75cada65651075进行加密
    C64e(0xda75cada65651075), 
    # 使用C64e函数对0x3153b531d7d78453进行加密
    C64e(0x3153b531d7d78453),
    # 使用C64e函数对0xc65113c68484d551进行加密
    C64e(0xc65113c68484d551), 
    # 使用C64e函数对0xb8d3bbb8d0d003d3进行加密
    C64e(0xb8d3bbb8d0d003d3),
    # 使用C64e函数对0xc35e1fc38282dc5e进行加密
    C64e(0xc35e1fc38282dc5e), 
    # 使用C64e函数对0xb0cb52b02929e2cb进行加密
    C64e(0xb0cb52b02929e2cb),
    # 使用C64e函数对0x7799b4775a5ac399进行加密
    C64e(0x7799b4775a5ac399), 
    # 使用C64e函数对0x11333c111e1e2d33进行加密
    C64e(0x11333c111e1e2d33),
    # 使用C64e函数对0xcb46f6cb7b7b3d46进行加密
    C64e(0xcb46f6cb7b7b3d46), 
    # 使用C64e函数对0xfc1f4bfca8a8b71f进行加密
    C64e(0xfc1f4bfca8a8b71f),
    # 使用C64e函数对0xd661dad66d6d0c61进行加密
    C64e(0xd661dad66d6d0c61), 
    # 使用C64e函数对0x3a4e583a2c2c624e进行加密
    C64e(0x3a4e583a2c2c624e)
# 定义一个静态的64位无符号整数数组T6
static const sph_u64 T6[] = {
    # 定义数组的元素，每个元素都是一个64位的无符号整数
    C64e(0xf4a5f497a5c6c632), C64e(0x978497eb84f8f86f),
    C64e(0xb099b0c799eeee5e), C64e(0x8c8d8cf78df6f67a),
    C64e(0x170d17e50dffffe8), C64e(0xdcbddcb7bdd6d60a),
    C64e(0xc8b1c8a7b1dede16), C64e(0xfc54fc395491916d),
    C64e(0xf050f0c050606090), C64e(0x0503050403020207),
    C64e(0xe0a9e087a9cece2e), C64e(0x877d87ac7d5656d1),
    C64e(0x2b192bd519e7e7cc), C64e(0xa662a67162b5b513),
    C64e(0x31e6319ae64d4d7c), C64e(0xb59ab5c39aecec59),
    C64e(0xcf45cf05458f8f40), C64e(0xbc9dbc3e9d1f1fa3),
    C64e(0xc040c00940898949), C64e(0x928792ef87fafa68),
    C64e(0x3f153fc515efefd0), C64e(0x26eb267febb2b294),
    C64e(0x40c94007c98e8ece), C64e(0x1d0b1ded0bfbfbe6),
    C64e(0x2fec2f82ec41416e), C64e(0xa967a97d67b3b31a),
    C64e(0x1cfd1cbefd5f5f43), C64e(0x25ea258aea454560),
    C64e(0xdabfda46bf2323f9), C64e(0x02f702a6f7535351),
    C64e(0xa196a1d396e4e445), C64e(0xed5bed2d5b9b9b76),
    C64e(0x5dc25deac2757528), C64e(0x241c24d91ce1e1c5),
    C64e(0xe9aee97aae3d3dd4), C64e(0xbe6abe986a4c4cf2),
    C64e(0xee5aeed85a6c6c82), C64e(0xc341c3fc417e7ebd),
    C64e(0x060206f102f5f5f3), C64e(0xd14fd11d4f838352),
    C64e(0xe45ce4d05c68688c), C64e(0x07f407a2f4515156),
    C64e(0x5c345cb934d1d18d), C64e(0x180818e908f9f9e1),
    C64e(0xae93aedf93e2e24c), C64e(0x9573954d73abab3e),
    C64e(0xf553f5c453626297), C64e(0x413f41543f2a2a6b),
    C64e(0x140c14100c08081c), C64e(0xf652f63152959563),
    C64e(0xaf65af8c654646e9), C64e(0xe25ee2215e9d9d7f),
    C64e(0x7828786028303048), C64e(0xf8a1f86ea13737cf),
    C64e(0x110f11140f0a0a1b), C64e(0xc4b5c45eb52f2feb),
    C64e(0x1b091b1c090e0e15), C64e(0x5a365a483624247e),
    C64e(0xb69bb6369b1b1bad), C64e(0x473d47a53ddfdf98),
    C64e(0x6a266a8126cdcda7), C64e(0xbb69bb9c694e4ef5),
    C64e(0x4ccd4cfecd7f7f33), C64e(0xba9fbacf9feaea50),
    C64e(0x2d1b2d241b12123f), C64e(0xb99eb93a9e1d1da4),
    C64e(0x9c749cb0745858c4), C64e(0x722e72682e343446),
    C64e(0x772d776c2d363641), C64e(0xcdb2cda3b2dcdc11),
    # ...
}
    # 这些是一系列的函数调用，每个函数调用都使用了一个参数，参数是十六进制数
    C64e(0x29ee2973eeb4b49d), C64e(0x16fb16b6fb5b5b4d),
    C64e(0x01f60153f6a4a4a5), C64e(0xd74dd7ec4d7676a1),
    C64e(0xa361a37561b7b714), C64e(0x49ce49face7d7d34),
    C64e(0x8d7b8da47b5252df), C64e(0x423e42a13edddd9f),
    C64e(0x937193bc715e5ecd), C64e(0xa297a226971313b1),
    C64e(0x04f50457f5a6a6a2), C64e(0xb868b86968b9b901),
    C64e(0x0000000000000000), C64e(0x742c74992cc1c1b5),
    C64e(0xa060a080604040e0), C64e(0x211f21dd1fe3e3c2),
    C64e(0x43c843f2c879793a), C64e(0x2ced2c77edb6b69a),
    C64e(0xd9bed9b3bed4d40d), C64e(0xca46ca01468d8d47),
    C64e(0x70d970ced9676717), C64e(0xdd4bdde44b7272af),
    C64e(0x79de7933de9494ed), C64e(0x67d4672bd49898ff),
    C64e(0x23e8237be8b0b093), C64e(0xde4ade114a85855b),
    C64e(0xbd6bbd6d6bbbbb06), C64e(0x7e2a7e912ac5c5bb),
    C64e(0x34e5349ee54f4f7b), C64e(0x3a163ac116ededd7),
    C64e(0x54c55417c58686d2), C64e(0x62d7622fd79a9af8),
    C64e(0xff55ffcc55666699), C64e(0xa794a722941111b6),
    C64e(0x4acf4a0fcf8a8ac0), C64e(0x301030c910e9e9d9),
    C64e(0x0a060a080604040e), C64e(0x988198e781fefe66),
    C64e(0x0bf00b5bf0a0a0ab), C64e(0xcc44ccf0447878b4),
    C64e(0xd5bad54aba2525f0), C64e(0x3ee33e96e34b4b75),
    C64e(0x0ef30e5ff3a2a2ac), C64e(0x19fe19bafe5d5d44),
    C64e(0x5bc05b1bc08080db), C64e(0x858a850a8a050580),
    C64e(0xecadec7ead3f3fd3), C64e(0xdfbcdf42bc2121fe),
    C64e(0xd848d8e0487070a8), C64e(0x0c040cf904f1f1fd),
    C64e(0x7adf7ac6df636319), C64e(0x58c158eec177772f),
    C64e(0x9f759f4575afaf30), C64e(0xa563a584634242e7),
    C64e(0x5030504030202070), C64e(0x2e1a2ed11ae5e5cb),
    C64e(0x120e12e10efdfdef), C64e(0xb76db7656dbfbf08),
    C64e(0xd44cd4194c818155), C64e(0x3c143c3014181824),
    C64e(0x5f355f4c35262679), C64e(0x712f719d2fc3c3b2),
    C64e(0x38e13867e1bebe86), C64e(0xfda2fd6aa23535c8),
    C64e(0x4fcc4f0bcc8888c7), C64e(0x4b394b5c392e2e65),
    C64e(0xf957f93d5793936a), C64e(0x0df20daaf2555558),
    C64e(0x9d829de382fcfc61), C64e(0xc947c9f4477a7ab3),
    # 定义一个长字符串，包含一系列的十六进制数
    C64e(0xefacef8bacc8c827), C64e(0x32e7326fe7baba88),
    C64e(0x7d2b7d642b32324f), C64e(0xa495a4d795e6e642),
    C64e(0xfba0fb9ba0c0c03b), C64e(0xb398b332981919aa),
    C64e(0x68d16827d19e9ef6), C64e(0x817f815d7fa3a322),
    C64e(0xaa66aa88664444ee), C64e(0x827e82a87e5454d6),
    C64e(0xe6abe676ab3b3bdd), C64e(0x9e839e16830b0b95),
    C64e(0x45ca4503ca8c8cc9), C64e(0x7b297b9529c7c7bc),
    C64e(0x6ed36ed6d36b6b05), C64e(0x443c44503c28286c),
    C64e(0x8b798b5579a7a72c), C64e(0x3de23d63e2bcbc81),
    C64e(0x271d272c1d161631), C64e(0x9a769a4176adad37),
    C64e(0x4d3b4dad3bdbdb96), C64e(0xfa56fac85664649e),
    C64e(0xd24ed2e84e7474a6), C64e(0x221e22281e141436),
    C64e(0x76db763fdb9292e4), C64e(0x1e0a1e180a0c0c12),
    C64e(0xb46cb4906c4848fc), C64e(0x37e4376be4b8b88f),
    C64e(0xe75de7255d9f9f78), C64e(0xb26eb2616ebdbd0f),
    C64e(0x2aef2a86ef434369), C64e(0xf1a6f193a6c4c435),
    C64e(0xe3a8e372a83939da), C64e(0xf7a4f762a43131c6),
    C64e(0x593759bd37d3d38a), C64e(0x868b86ff8bf2f274),
    C64e(0x563256b132d5d583), C64e(0xc543c50d438b8b4e),
    C64e(0xeb59ebdc596e6e85), C64e(0xc2b7c2afb7dada18),
    C64e(0x8f8c8f028c01018e), C64e(0xac64ac7964b1b11d),
    C64e(0x6dd26d23d29c9cf1), C64e(0x3be03b92e0494972),
    C64e(0xc7b4c7abb4d8d81f), C64e(0x15fa1543faacacb9),
    C64e(0x090709fd07f3f3fa), C64e(0x6f256f8525cfcfa0),
    C64e(0xeaafea8fafcaca20), C64e(0x898e89f38ef4f47d),
    C64e(0x20e9208ee9474767), C64e(0x2818282018101038),
    C64e(0x64d564ded56f6f0b), C64e(0x838883fb88f0f073),
    C64e(0xb16fb1946f4a4afb), C64e(0x967296b8725c5cca),
    C64e(0x6c246c7024383854), C64e(0x08f108aef157575f),
    C64e(0x52c752e6c7737321), C64e(0xf351f33551979764),
    C64e(0x6523658d23cbcbae), C64e(0x847c84597ca1a125),
    C64e(0xbf9cbfcb9ce8e857), C64e(0x6321637c213e3e5d),
    C64e(0x7cdd7c37dd9696ea), C64e(0x7fdc7fc2dc61611e),
    C64e(0x9186911a860d0d9c), C64e(0x9485941e850f0f9b),
    C64e(0xab90abdb90e0e04b), C64e(0xc642c6f8427c7cba),
    
    
    注释：这段代码定义了一个长字符串，其中包含了一系列的十六进制数。
    # 以下是一系列的十六进制数值，表示一组数据
    C64e(0x57c457e2c4717126), C64e(0xe5aae583aacccc29),
    C64e(0x73d8733bd89090e3), C64e(0x0f050f0c05060609),
    C64e(0x030103f501f7f7f4), C64e(0x36123638121c1c2a),
    C64e(0xfea3fe9fa3c2c23c), C64e(0xe15fe1d45f6a6a8b),
    C64e(0x10f91047f9aeaebe), C64e(0x6bd06bd2d0696902),
    C64e(0xa891a82e911717bf), C64e(0xe858e82958999971),
    C64e(0x69276974273a3a53), C64e(0xd0b9d04eb92727f7),
    C64e(0x483848a938d9d991), C64e(0x351335cd13ebebde),
    C64e(0xceb3ce56b32b2be5), C64e(0x5533554433222277),
    C64e(0xd6bbd6bfbbd2d204), C64e(0x9070904970a9a939),
    C64e(0x8089800e89070787), C64e(0xf2a7f266a73333c1),
    C64e(0xc1b6c15ab62d2dec), C64e(0x66226678223c3c5a),
    C64e(0xad92ad2a921515b8), C64e(0x6020608920c9c9a9),
    C64e(0xdb49db154987875c), C64e(0x1aff1a4fffaaaab0),
    C64e(0x887888a0785050d8), C64e(0x8e7a8e517aa5a52b),
    C64e(0x8a8f8a068f030389), C64e(0x13f813b2f859594a),
    C64e(0x9b809b1280090992), C64e(0x39173934171a1a23),
    C64e(0x75da75cada656510), C64e(0x533153b531d7d784),
    C64e(0x51c65113c68484d5), C64e(0xd3b8d3bbb8d0d003),
    C64e(0x5ec35e1fc38282dc), C64e(0xcbb0cb52b02929e2),
    C64e(0x997799b4775a5ac3), C64e(0x3311333c111e1e2d),
    C64e(0x46cb46f6cb7b7b3d), C64e(0x1ffc1f4bfca8a8b7),
    C64e(0x61d661dad66d6d0c), C64e(0x4e3a4e583a2c2c62)
# 定义一个静态常量数组 T7，包含一系列 64 位的无符号整数
static const sph_u64 T7[] = {
    # 每个元素都是一个 64 位的无符号整数，使用 C64e 宏定义
    C64e(0x32f4a5f497a5c6c6), C64e(0x6f978497eb84f8f8),
    C64e(0x5eb099b0c799eeee), C64e(0x7a8c8d8cf78df6f6),
    C64e(0xe8170d17e50dffff), C64e(0x0adcbddcb7bdd6d6),
    C64e(0x16c8b1c8a7b1dede), C64e(0x6dfc54fc39549191),
    C64e(0x90f050f0c0506060), C64e(0x0705030504030202),
    C64e(0x2ee0a9e087a9cece), C64e(0xd1877d87ac7d5656),
    C64e(0xcc2b192bd519e7e7), C64e(0x13a662a67162b5b5),
    C64e(0x7c31e6319ae64d4d), C64e(0x59b59ab5c39aecec),
    C64e(0x40cf45cf05458f8f), C64e(0xa3bc9dbc3e9d1f1f),
    C64e(0x49c040c009408989), C64e(0x68928792ef87fafa),
    C64e(0xd03f153fc515efef), C64e(0x9426eb267febb2b2),
    C64e(0xce40c94007c98e8e), C64e(0xe61d0b1ded0bfbfb),
    C64e(0x6e2fec2f82ec4141), C64e(0x1aa967a97d67b3b3),
    C64e(0x431cfd1cbefd5f5f), C64e(0x6025ea258aea4545),
    C64e(0xf9dabfda46bf2323), C64e(0x5102f702a6f75353),
    C64e(0x45a196a1d396e4e4), C64e(0x76ed5bed2d5b9b9b),
    C64e(0x285dc25deac27575), C64e(0xc5241c24d91ce1e1),
    C64e(0xd4e9aee97aae3d3d), C64e(0xf2be6abe986a4c4c),
    C64e(0x82ee5aeed85a6c6c), C64e(0xbdc341c3fc417e7e),
    C64e(0xf3060206f102f5f5), C64e(0x52d14fd11d4f8383),
    C64e(0x8ce45ce4d05c6868), C64e(0x5607f407a2f45151),
    C64e(0x8d5c345cb934d1d1), C64e(0xe1180818e908f9f9),
    C64e(0x4cae93aedf93e2e2), C64e(0x3e9573954d73abab),
    C64e(0x97f553f5c4536262), C64e(0x6b413f41543f2a2a),
    C64e(0x1c140c14100c0808), C64e(0x63f652f631529595),
    C64e(0xe9af65af8c654646), C64e(0x7fe25ee2215e9d9d),
    C64e(0x4878287860283030), C64e(0xcff8a1f86ea13737),
    C64e(0x1b110f11140f0a0a), C64e(0xebc4b5c45eb52f2f),
    C64e(0x151b091b1c090e0e), C64e(0x7e5a365a48362424),
    C64e(0xadb69bb6369b1b1b), C64e(0x98473d47a53ddfdf),
    C64e(0xa76a266a8126cdcd), C64e(0xf5bb69bb9c694e4e),
    C64e(0x334ccd4cfecd7f7f), C64e(0x50ba9fbacf9feaea),
    C64e(0x3f2d1b2d241b1212), C64e(0xa4b99eb93a9e1d1d),
    C64e(0xc49c749cb0745858), C64e(0x46722e72682e3434),
    C64e(0x41772d776c2d3636), C64e(0x11cdb2cda3b2dcdc),
    # ...
}
    # 以下是一系列经过 C64e 函数处理的十六进制数值
    C64e(0x9d29ee2973eeb4b4), C64e(0x4d16fb16b6fb5b5b),
    C64e(0xa501f60153f6a4a4), C64e(0xa1d74dd7ec4d7676),
    C64e(0x14a361a37561b7b7), C64e(0x3449ce49face7d7d),
    C64e(0xdf8d7b8da47b5252), C64e(0x9f423e42a13edddd),
    C64e(0xcd937193bc715e5e), C64e(0xb1a297a226971313),
    C64e(0xa204f50457f5a6a6), C64e(0x01b868b86968b9b9),
    C64e(0x0000000000000000), C64e(0xb5742c74992cc1c1),
    C64e(0xe0a060a080604040), C64e(0xc2211f21dd1fe3e3),
    C64e(0x3a43c843f2c87979), C64e(0x9a2ced2c77edb6b6),
    C64e(0x0dd9bed9b3bed4d4), C64e(0x47ca46ca01468d8d),
    C64e(0x1770d970ced96767), C64e(0xafdd4bdde44b7272),
    C64e(0xed79de7933de9494), C64e(0xff67d4672bd49898),
    C64e(0x9323e8237be8b0b0), C64e(0x5bde4ade114a8585),
    C64e(0x06bd6bbd6d6bbbbb), C64e(0xbb7e2a7e912ac5c5),
    C64e(0x7b34e5349ee54f4f), C64e(0xd73a163ac116eded),
    C64e(0xd254c55417c58686), C64e(0xf862d7622fd79a9a),
    C64e(0x99ff55ffcc556666), C64e(0xb6a794a722941111),
    C64e(0xc04acf4a0fcf8a8a), C64e(0xd9301030c910e9e9),
    C64e(0x0e0a060a08060404), C64e(0x66988198e781fefe),
    C64e(0xab0bf00b5bf0a0a0), C64e(0xb4cc44ccf0447878),
    C64e(0xf0d5bad54aba2525), C64e(0x753ee33e96e34b4b),
    C64e(0xac0ef30e5ff3a2a2), C64e(0x4419fe19bafe5d5d),
    C64e(0xdb5bc05b1bc08080), C64e(0x80858a850a8a0505),
    C64e(0xd3ecadec7ead3f3f), C64e(0xfedfbcdf42bc2121),
    C64e(0xa8d848d8e0487070), C64e(0xfd0c040cf904f1f1),
    C64e(0x197adf7ac6df6363), C64e(0x2f58c158eec17777),
    C64e(0x309f759f4575afaf), C64e(0xe7a563a584634242),
    C64e(0x7050305040302020), C64e(0xcb2e1a2ed11ae5e5),
    C64e(0xef120e12e10efdfd), C64e(0x08b76db7656dbfbf),
    C64e(0x55d44cd4194c8181), C64e(0x243c143c30141818),
    C64e(0x795f355f4c352626), C64e(0xb2712f719d2fc3c3),
    C64e(0x8638e13867e1bebe), C64e(0xc8fda2fd6aa23535),
    C64e(0xc74fcc4f0bcc8888), C64e(0x654b394b5c392e2e),
    C64e(0x6af957f93d579393), C64e(0x580df20daaf25555),
    C64e(0x619d829de382fcfc), C64e(0xb3c947c9f4477a7a),
    # 定义一个长字符串，包含一系列的十六进制数
    C64e(0x27efacef8bacc8c8), C64e(0x8832e7326fe7baba),
    C64e(0x4f7d2b7d642b3232), C64e(0x42a495a4d795e6e6),
    C64e(0x3bfba0fb9ba0c0c0), C64e(0xaab398b332981919),
    C64e(0xf668d16827d19e9e), C64e(0x22817f815d7fa3a3),
    C64e(0xeeaa66aa88664444), C64e(0xd6827e82a87e5454),
    C64e(0xdde6abe676ab3b3b), C64e(0x959e839e16830b0b),
    C64e(0xc945ca4503ca8c8c), C64e(0xbc7b297b9529c7c7),
    C64e(0x056ed36ed6d36b6b), C64e(0x6c443c44503c2828),
    C64e(0x2c8b798b5579a7a7), C64e(0x813de23d63e2bcbc),
    C64e(0x31271d272c1d1616), C64e(0x379a769a4176adad),
    C64e(0x964d3b4dad3bdbdb), C64e(0x9efa56fac8566464),
    C64e(0xa6d24ed2e84e7474), C64e(0x36221e22281e1414),
    C64e(0xe476db763fdb9292), C64e(0x121e0a1e180a0c0c),
    C64e(0xfcb46cb4906c4848), C64e(0x8f37e4376be4b8b8),
    C64e(0x78e75de7255d9f9f), C64e(0x0fb26eb2616ebdbd),
    C64e(0x692aef2a86ef4343), C64e(0x35f1a6f193a6c4c4),
    C64e(0xdae3a8e372a83939), C64e(0xc6f7a4f762a43131),
    C64e(0x8a593759bd37d3d3), C64e(0x74868b86ff8bf2f2),
    C64e(0x83563256b132d5d5), C64e(0x4ec543c50d438b8b),
    C64e(0x85eb59ebdc596e6e), C64e(0x18c2b7c2afb7dada),
    C64e(0x8e8f8c8f028c0101), C64e(0x1dac64ac7964b1b1),
    C64e(0xf16dd26d23d29c9c), C64e(0x723be03b92e04949),
    C64e(0x1fc7b4c7abb4d8d8), C64e(0xb915fa1543faacac),
    C64e(0xfa090709fd07f3f3), C64e(0xa06f256f8525cfcf),
    C64e(0x20eaafea8fafcaca), C64e(0x7d898e89f38ef4f4),
    C64e(0x6720e9208ee94747), C64e(0x3828182820181010),
    C64e(0x0b64d564ded56f6f), C64e(0x73838883fb88f0f0),
    C64e(0xfbb16fb1946f4a4a), C64e(0xca967296b8725c5c),
    C64e(0x546c246c70243838), C64e(0x5f08f108aef15757),
    C64e(0x2152c752e6c77373), C64e(0x64f351f335519797),
    C64e(0xae6523658d23cbcb), C64e(0x25847c84597ca1a1),
    C64e(0x57bf9cbfcb9ce8e8), C64e(0x5d6321637c213e3e),
    C64e(0xea7cdd7c37dd9696), C64e(0x1e7fdc7fc2dc6161),
    C64e(0x9c9186911a860d0d), C64e(0x9b9485941e850f0f),
    C64e(0x4bab90abdb90e0e0), C64e(0xbac642c6f8427c7c),
    # 以下是一系列经过C64e加密的十六进制数字
    C64e(0x2657c457e2c47171), C64e(0x29e5aae583aacccc),
    C64e(0xe373d8733bd89090), C64e(0x090f050f0c050606),
    C64e(0xf4030103f501f7f7), C64e(0x2a36123638121c1c),
    C64e(0x3cfea3fe9fa3c2c2), C64e(0x8be15fe1d45f6a6a),
    C64e(0xbe10f91047f9aeae), C64e(0x026bd06bd2d06969),
    C64e(0xbfa891a82e911717), C64e(0x71e858e829589999),
    C64e(0x5369276974273a3a), C64e(0xf7d0b9d04eb92727),
    C64e(0x91483848a938d9d9), C64e(0xde351335cd13ebeb),
    C64e(0xe5ceb3ce56b32b2b), C64e(0x7755335544332222),
    C64e(0x04d6bbd6bfbbd2d2), C64e(0x399070904970a9a9),
    C64e(0x878089800e890707), C64e(0xc1f2a7f266a73333),
    C64e(0xecc1b6c15ab62d2d), C64e(0x5a66226678223c3c),
    C64e(0xb8ad92ad2a921515), C64e(0xa96020608920c9c9),
    C64e(0x5cdb49db15498787), C64e(0xb01aff1a4fffaaaa),
    C64e(0xd8887888a0785050), C64e(0x2b8e7a8e517aa5a5),
    C64e(0x898a8f8a068f0303), C64e(0x4a13f813b2f85959),
    C64e(0x929b809b12800909), C64e(0x2339173934171a1a),
    C64e(0x1075da75cada6565), C64e(0x84533153b531d7d7),
    C64e(0xd551c65113c68484), C64e(0x03d3b8d3bbb8d0d0),
    C64e(0xdc5ec35e1fc38282), C64e(0xe2cbb0cb52b02929),
    C64e(0xc3997799b4775a5a), C64e(0x2d3311333c111e1e),
    C64e(0x3d46cb46f6cb7b7b), C64e(0xb71ffc1f4bfca8a8),
    C64e(0x0c61d661dad66d6d), C64e(0x624e3a4e583a2c2c)
// 定义一个宏，用于声明一个小状态
#define DECL_STATE_SMALL \
    sph_u64 H[8];

// 定义一个宏，用于从小状态中读取状态
#define READ_STATE_SMALL(sc)   do { \
        memcpy(H, (sc)->state.wide, sizeof H); \
    } while (0)

// 定义一个宏，用于将小状态写入状态
#define WRITE_STATE_SMALL(sc)   do { \
        memcpy((sc)->state.wide, H, sizeof H); \
    } while (0)

// 如果定义了 SPH_SMALL_FOOTPRINT_GROESTL 宏
#if SPH_SMALL_FOOTPRINT_GROESTL

// 定义一个宏，用于进行 RSTT 变换
#define RSTT(d, a, b0, b1, b2, b3, b4, b5, b6, b7)   do { \
        t[d] = T0[B64_0(a[b0])] \
            ^ R64(T0[B64_1(a[b1])],  8) \
            ^ R64(T0[B64_2(a[b2])], 16) \
            ^ R64(T0[B64_3(a[b3])], 24) \
            ^ T4[B64_4(a[b4])] \
            ^ R64(T4[B64_5(a[b5])],  8) \
            ^ R64(T4[B64_6(a[b6])], 16) \
            ^ R64(T4[B64_7(a[b7])], 24); \
    } while (0)

// 如果没有定义 SPH_SMALL_FOOTPRINT_GROESTL 宏
#else

// 定义一个宏，用于进行 RSTT 变换
#define RSTT(d, a, b0, b1, b2, b3, b4, b5, b6, b7)   do { \
        t[d] = T0[B64_0(a[b0])] \
            ^ T1[B64_1(a[b1])] \
            ^ T2[B64_2(a[b2])] \
            ^ T3[B64_3(a[b3])] \
            ^ T4[B64_4(a[b4])] \
            ^ T5[B64_5(a[b5])] \
            ^ T6[B64_6(a[b6])] \
            ^ T7[B64_7(a[b7])]; \
    } while (0)

#endif

// 定义一个宏，用于进行小轮变换
#define ROUND_SMALL_P(a, r)   do { \
        sph_u64 t[8]; \
        a[0] ^= PC64(0x00, r); \
        a[1] ^= PC64(0x10, r); \
        a[2] ^= PC64(0x20, r); \
        a[3] ^= PC64(0x30, r); \
        a[4] ^= PC64(0x40, r); \
        a[5] ^= PC64(0x50, r); \
        a[6] ^= PC64(0x60, r); \
        a[7] ^= PC64(0x70, r); \
        RSTT(0, a, 0, 1, 2, 3, 4, 5, 6, 7); \
        RSTT(1, a, 1, 2, 3, 4, 5, 6, 7, 0); \
        RSTT(2, a, 2, 3, 4, 5, 6, 7, 0, 1); \
        RSTT(3, a, 3, 4, 5, 6, 7, 0, 1, 2); \
        RSTT(4, a, 4, 5, 6, 7, 0, 1, 2, 3); \
        RSTT(5, a, 5, 6, 7, 0, 1, 2, 3, 4); \
        RSTT(6, a, 6, 7, 0, 1, 2, 3, 4, 5); \
        RSTT(7, a, 7, 0, 1, 2, 3, 4, 5, 6); \
        a[0] = t[0]; \
        a[1] = t[1]; \
        a[2] = t[2]; \
        a[3] = t[3]; \
        a[4] = t[4]; \
        a[5] = t[5]; \
        a[6] = t[6]; \
        a[7] = t[7]; \
    } while (0)
#define ROUND_SMALL_Q(a, r)   do { \  # 定义一个宏，用于对a进行一系列操作，r为轮数
        sph_u64 t[8]; \  # 声明一个包含8个sph_u64类型元素的数组t
        a[0] ^= QC64(0x00, r); \  # 对a[0]进行异或操作
        a[1] ^= QC64(0x10, r); \  # 对a[1]进行异或操作
        a[2] ^= QC64(0x20, r); \  # 对a[2]进行异或操作
        a[3] ^= QC64(0x30, r); \  # 对a[3]进行异或操作
        a[4] ^= QC64(0x40, r); \  # 对a[4]进行异或操作
        a[5] ^= QC64(0x50, r); \  # 对a[5]进行异或操作
        a[6] ^= QC64(0x60, r); \  # 对a[6]进行异或操作
        a[7] ^= QC64(0x70, r); \  # 对a[7]进行异或操作
        RSTT(0, a, 1, 3, 5, 7, 0, 2, 4, 6); \  # 调用RSTT函数
        RSTT(1, a, 2, 4, 6, 0, 1, 3, 5, 7); \  # 调用RSTT函数
        RSTT(2, a, 3, 5, 7, 1, 2, 4, 6, 0); \  # 调用RSTT函数
        RSTT(3, a, 4, 6, 0, 2, 3, 5, 7, 1); \  # 调用RSTT函数
        RSTT(4, a, 5, 7, 1, 3, 4, 6, 0, 2); \  # 调用RSTT函数
        RSTT(5, a, 6, 0, 2, 4, 5, 7, 1, 3); \  # 调用RSTT函数
        RSTT(6, a, 7, 1, 3, 5, 6, 0, 2, 4); \  # 调用RSTT函数
        RSTT(7, a, 0, 2, 4, 6, 7, 1, 3, 5); \  # 调用RSTT函数
        a[0] = t[0]; \  # 将t[0]的值赋给a[0]
        a[1] = t[1]; \  # 将t[1]的值赋给a[1]
        a[2] = t[2]; \  # 将t[2]的值赋给a[2]
        a[3] = t[3]; \  # 将t[3]的值赋给a[3]
        a[4] = t[4]; \  # 将t[4]的值赋给a[4]
        a[5] = t[5]; \  # 将t[5]的值赋给a[5]
        a[6] = t[6]; \  # 将t[6]的值赋给a[6]
        a[7] = t[7]; \  # 将t[7]的值赋给a[7]
    } while (0)  # do-while循环，确保宏的完整性

#if SPH_SMALL_FOOTPRINT_GROESTL  # 如果定义了SPH_SMALL_FOOTPRINT_GROESTL
#define PERM_SMALL_P(a)   do { \  # 定义一个宏，用于对a进行一系列操作
        int r; \  # 声明一个整型变量r
        for (r = 0; r < 10; r ++) \  # 循环10次
            ROUND_SMALL_P(a, r); \  # 调用ROUND_SMALL_P宏
    } while (0)  # do-while循环，确保宏的完整性

#define PERM_SMALL_Q(a)   do { \  # 定义一个宏，用于对a进行一系列操作
        int r; \  # 声明一个整型变量r
        for (r = 0; r < 10; r ++) \  # 循环10次
            ROUND_SMALL_Q(a, r); \  # 调用ROUND_SMALL_Q宏
    } while (0)  # do-while循环，确保宏的完整性

#else  # 如果没有定义SPH_SMALL_FOOTPRINT_GROESTL
/*
 * Apparently, unrolling more than that confuses GCC, resulting in
 * lower performance, even though L1 cache would be no problem.
 */
#define PERM_SMALL_P(a)   do { \  # 定义一个宏，用于对a进行一系列操作
        int r; \  # 声明一个整型变量r
        for (r = 0; r < 10; r += 2) { \  # 循环5次
            ROUND_SMALL_P(a, r + 0); \  # 调用ROUND_SMALL_P宏
            ROUND_SMALL_P(a, r + 1); \  # 调用ROUND_SMALL_P宏
        } \  # 结束for循环
    } while (0)  # do-while循环，确保宏的完整性

#define PERM_SMALL_Q(a)   do { \  # 定义一个宏，用于对a进行一系列操作
        int r; \  # 声明一个整型变量r
        for (r = 0; r < 10; r += 2) { \  # 循环5次
            ROUND_SMALL_Q(a, r + 0); \  # 调用ROUND_SMALL_Q宏
            ROUND_SMALL_Q(a, r + 1); \  # 调用ROUND_SMALL_Q宏
        } \  # 结束for循环
    } while (0)  # do-while循环，确保宏的完整性

#endif
# 定义一个宏，用于压缩小块数据
#define COMPRESS_SMALL   do { \
        # 定义两个数组，用于存储数据
        sph_u64 g[8], m[8]; \
        # 定义一个变量，用于循环
        size_t u; \
        # 遍历数组，将数据解码并存储到数组中
        for (u = 0; u < 8; u ++) { \
            m[u] = dec64e_aligned(buf + (u << 3)); \
            # 将解码后的数据与预定义的常量进行异或操作
            g[u] = m[u] ^ H[u]; \
        } \
        # 对数组进行一系列置换操作
        PERM_SMALL_P(g); \
        PERM_SMALL_Q(m); \
        # 将置换后的数组与原数组进行异或操作
        for (u = 0; u < 8; u ++) \
            H[u] ^= g[u] ^ m[u]; \
    } while (0)

# 定义一个宏，用于最后处理小块数据
#define FINAL_SMALL   do { \
        # 定义一个数组，用于存储数据
        sph_u64 x[8]; \
        # 定义一个变量，用于循环
        size_t u; \
        # 将原数组的数据复制到新数组中
        memcpy(x, H, sizeof x); \
        # 对新数组进行一系列置换操作
        PERM_SMALL_P(x); \
        # 将置换后的新数组与原数组进行异或操作
        for (u = 0; u < 8; u ++) \
            H[u] ^= x[u]; \
    } while (0)

# 定义一个宏，用于声明大块数据的状态
#define DECL_STATE_BIG \
    # 定义一个数组，用于存储数据
    sph_u64 H[16];

# 定义一个宏，用于读取大块数据的状态
#define READ_STATE_BIG(sc)   do { \
        # 将状态数组的数据复制到指定的数组中
        memcpy(H, (sc)->state.wide, sizeof H); \
    } while (0)

# 定义一个宏，用于写入大块数据的状态
#define WRITE_STATE_BIG(sc)   do { \
        # 将指定的数组的数据复制到状态数组中
        memcpy((sc)->state.wide, H, sizeof H); \
    } while (0)

# 如果定义了 SPH_SMALL_FOOTPRINT_GROESTL 宏
#if SPH_SMALL_FOOTPRINT_GROESTL

# 定义一个宏，用于进行一系列置换操作
#define RBTT(d, a, b0, b1, b2, b3, b4, b5, b6, b7)   do { \
        # 将指定的数据进行一系列置换操作，并存储到指定的数组中
        t[d] = T0[B64_0(a[b0])] \
            ^ R64(T0[B64_1(a[b1])],  8) \
            ^ R64(T0[B64_2(a[b2])], 16) \
            ^ R64(T0[B64_3(a[b3])], 24) \
            ^ T4[B64_4(a[b4])] \
            ^ R64(T4[B64_5(a[b5])],  8) \
            ^ R64(T4[B64_6(a[b6])], 16) \
            ^ R64(T4[B64_7(a[b7])], 24); \
    } while (0)

# 否则
#else

# 定义一个宏，用于进行一系列置换操作
#define RBTT(d, a, b0, b1, b2, b3, b4, b5, b6, b7)   do { \
        # 将指定的数据进行一系列置换操作，并存储到指定的数组中
        t[d] = T0[B64_0(a[b0])] \
            ^ T1[B64_1(a[b1])] \
            ^ T2[B64_2(a[b2])] \
            ^ T3[B64_3(a[b3])] \
            ^ T4[B64_4(a[b4])] \
            ^ T5[B64_5(a[b5])] \
            ^ T6[B64_6(a[b6])] \
            ^ T7[B64_7(a[b7])]; \
    } while (0)

#endif

# 如果定义了 SPH_SMALL_FOOTPRINT_GROESTL 宏
#if SPH_SMALL_FOOTPRINT_GROESTL

# 继续定义
// 定义一个宏，用于对大端序的数据进行一轮置换
#define ROUND_BIG_P(a, r)   do { \
        // 定义一个临时数组，用于存储中间结果
        sph_u64 t[16]; \
        // 定义一个变量 u，用于循环计数
        size_t u; \
        // 对输入数组 a 的不同位置进行异或操作，使用宏 PC64 进行位移和异或
        a[0x0] ^= PC64(0x00, r); \
        a[0x1] ^= PC64(0x10, r); \
        a[0x2] ^= PC64(0x20, r); \
        a[0x3] ^= PC64(0x30, r); \
        a[0x4] ^= PC64(0x40, r); \
        a[0x5] ^= PC64(0x50, r); \
        a[0x6] ^= PC64(0x60, r); \
        a[0x7] ^= PC64(0x70, r); \
        a[0x8] ^= PC64(0x80, r); \
        a[0x9] ^= PC64(0x90, r); \
        a[0xA] ^= PC64(0xA0, r); \
        a[0xB] ^= PC64(0xB0, r); \
        a[0xC] ^= PC64(0xC0, r); \
        a[0xD] ^= PC64(0xD0, r); \
        a[0xE] ^= PC64(0xE0, r); \
        a[0xF] ^= PC64(0xF0, r); \
        // 循环遍历数组，对每个 4 个元素进行一轮置换
        for (u = 0; u < 16; u += 4) { \
            // 使用宏 RBTT 进行一轮置换
            RBTT(u + 0, a, u + 0, (u + 1) & 0xF, \
                (u + 2) & 0xF, (u + 3) & 0xF, (u + 4) & 0xF, \
                (u + 5) & 0xF, (u + 6) & 0xF, (u + 11) & 0xF); \
            RBTT(u + 1, a, u + 1, (u + 2) & 0xF, \
                (u + 3) & 0xF, (u + 4) & 0xF, (u + 5) & 0xF, \
                (u + 6) & 0xF, (u + 7) & 0xF, (u + 12) & 0xF); \
            RBTT(u + 2, a, u + 2, (u + 3) & 0xF, \
                (u + 4) & 0xF, (u + 5) & 0xF, (u + 6) & 0xF, \
                (u + 7) & 0xF, (u + 8) & 0xF, (u + 13) & 0xF); \
            RBTT(u + 3, a, u + 3, (u + 4) & 0xF, \
                (u + 5) & 0xF, (u + 6) & 0xF, (u + 7) & 0xF, \
                (u + 8) & 0xF, (u + 9) & 0xF, (u + 14) & 0xF); \
        } \
        // 将临时数组 t 的内容拷贝回输入数组 a
        memcpy(a, t, sizeof t); \
    } while (0)
#define ROUND_BIG_Q(a, r)   do { \  # 定义一个宏，用于执行大端模式下的Q轮函数，参数为a和r
        sph_u64 t[16]; \  # 声明一个长度为16的sph_u64类型数组t
        size_t u; \  # 声明一个size_t类型的变量u
        a[0x0] ^= QC64(0x00, r); \  # 对a[0x0]进行异或操作
        a[0x1] ^= QC64(0x10, r); \  # 对a[0x1]进行异或操作
        a[0x2] ^= QC64(0x20, r); \  # 对a[0x2]进行异或操作
        a[0x3] ^= QC64(0x30, r); \  # 对a[0x3]进行异或操作
        a[0x4] ^= QC64(0x40, r); \  # 对a[0x4]进行异或操作
        a[0x5] ^= QC64(0x50, r); \  # 对a[0x5]进行异或操作
        a[0x6] ^= QC64(0x60, r); \  # 对a[0x6]进行异或操作
        a[0x7] ^= QC64(0x70, r); \  # 对a[0x7]进行异或操作
        a[0x8] ^= QC64(0x80, r); \  # 对a[0x8]进行异或操作
        a[0x9] ^= QC64(0x90, r); \  # 对a[0x9]进行异或操作
        a[0xA] ^= QC64(0xA0, r); \  # 对a[0xA]进行异或操作
        a[0xB] ^= QC64(0xB0, r); \  # 对a[0xB]进行异或操作
        a[0xC] ^= QC64(0xC0, r); \  # 对a[0xC]进行异或操作
        a[0xD] ^= QC64(0xD0, r); \  # 对a[0xD]进行异或操作
        a[0xE] ^= QC64(0xE0, r); \  # 对a[0xE]进行异或操作
        a[0xF] ^= QC64(0xF0, r); \  # 对a[0xF]进行异或操作
        for (u = 0; u < 16; u += 4) { \  # 循环，每次增加4
            RBTT(u + 0, a, (u + 1) & 0xF, (u + 3) & 0xF, \  # 调用RBTT函数
                (u + 5) & 0xF, (u + 11) & 0xF, (u + 0) & 0xF, \  # 调用RBTT函数
                (u + 2) & 0xF, (u + 4) & 0xF, (u + 6) & 0xF); \  # 调用RBTT函数
            RBTT(u + 1, a, (u + 2) & 0xF, (u + 4) & 0xF, \  # 调用RBTT函数
                (u + 6) & 0xF, (u + 12) & 0xF, (u + 1) & 0xF, \  # 调用RBTT函数
                (u + 3) & 0xF, (u + 5) & 0xF, (u + 7) & 0xF); \  # 调用RBTT函数
            RBTT(u + 2, a, (u + 3) & 0xF, (u + 5) & 0xF, \  # 调用RBTT函数
                (u + 7) & 0xF, (u + 13) & 0xF, (u + 2) & 0xF, \  # 调用RBTT函数
                (u + 4) & 0xF, (u + 6) & 0xF, (u + 8) & 0xF); \  # 调用RBTT函数
            RBTT(u + 3, a, (u + 4) & 0xF, (u + 6) & 0xF, \  # 调用RBTT函数
                (u + 8) & 0xF, (u + 14) & 0xF, (u + 3) & 0xF, \  # 调用RBTT函数
                (u + 5) & 0xF, (u + 7) & 0xF, (u + 9) & 0xF); \  # 调用RBTT函数
        } \  # 循环结束
        memcpy(a, t, sizeof t); \  # 将t的内容拷贝到a中
    } while (0)  # 循环结束

#else  # 如果不满足条件，则执行下面的代码

    } while (0)  # 循环结束

    } while (0)  # 循环结束

#endif  # 条件编译结束

#define PERM_BIG_P(a)   do { \  # 定义一个宏，用于执行大端模式下的P置换函数，参数为a
        int r; \  # 声明一个int类型的变量r
        for (r = 0; r < 14; r += 2) { \  # 循环，每次增加2
            ROUND_BIG_P(a, r + 0); \  # 调用ROUND_BIG_P函数
            ROUND_BIG_P(a, r + 1); \  # 调用ROUND_BIG_P函数
        } \  # 循环结束
    } while (0)  # 循环结束

#define PERM_BIG_Q(a)   do { \  # 定义一个宏，用于执行大端模式下的Q置换函数，参数为a
        int r; \  # 声明一个int类型的变量r
        for (r = 0; r < 14; r += 2) { \  # 循环，每次增加2
            ROUND_BIG_Q(a, r + 0); \  # 调用ROUND_BIG_Q函数
            ROUND_BIG_Q(a, r + 1); \  # 调用ROUND_BIG_Q函数
        } \  # 循环结束
    } while (0)  # 循环结束

/* obsolete  # 注释
#if SPH_SMALL_FOOTPRINT_GROESTL  # 如果满足条件，则执行下面的代码
#define COMPRESS_BIG   do { \
        sph_u64 g[16], m[16], *ya; \  # 定义64位整数数组g和m，以及指向64位整数数组的指针ya
        const sph_u64 *yc; \  # 定义指向常量64位整数数组的指针yc
        size_t u; \  # 定义变量u，表示大小
        int i; \  # 定义变量i，表示整数
        for (u = 0; u < 16; u ++) { \  # 循环16次
            m[u] = dec64e_aligned(buf + (u << 3)); \  # 将buf中的数据解码为64位整数，并存储在m数组中
            g[u] = m[u] ^ H[u]; \  # 将m数组中的数据与H数组中的数据进行异或运算，并存储在g数组中
        } \
        ya = g; \  # 将g数组的地址赋值给ya指针
        yc = CP; \  # 将CP的地址赋值给yc指针
        for (i = 0; i < 2; i ++) { \  # 循环2次
            PERM_BIG(ya, yc); \  # 调用PERM_BIG函数，传入ya指针和yc指针作为参数
            ya = m; \  # 将m数组的地址赋值给ya指针
            yc = CQ; \  # 将CQ的地址赋值给yc指针
        } \
        for (u = 0; u < 16; u ++) { \  # 循环16次
            H[u] ^= g[u] ^ m[u]; \  # 将H数组中的数据与g数组中的数据和m数组中的数据进行异或运算，并存储在H数组中
        } \
    } while (0)  # do-while循环，循环体为0



#else



#define COMPRESS_BIG   do { \
        sph_u64 g[16], m[16]; \  # 定义64位整数数组g和m
        size_t u; \  # 定义变量u，表示大小
        for (u = 0; u < 16; u ++) { \  # 循环16次
            m[u] = dec64e_aligned(buf + (u << 3)); \  # 将buf中的数据解码为64位整数，并存储在m数组中
            g[u] = m[u] ^ H[u]; \  # 将m数组中的数据与H数组中的数据进行异或运算，并存储在g数组中
        } \
        PERM_BIG_P(g); \  # 调用PERM_BIG_P函数，传入g数组作为参数
        PERM_BIG_Q(m); \  # 调用PERM_BIG_Q函数，传入m数组作为参数
        for (u = 0; u < 16; u ++) { \  # 循环16次
            H[u] ^= g[u] ^ m[u]; \  # 将H数组中的数据与g数组中的数据和m数组中的数据进行异或运算，并存储在H数组中
        } \
    } while (0)  # do-while循环，循环体为0



/* obsolete
#endif



#define FINAL_BIG   do { \
        sph_u64 x[16]; \  # 定义64位整数数组x
        size_t u; \  # 定义变量u，表示大小
        memcpy(x, H, sizeof x); \  # 将H数组中的数据复制到x数组中
        PERM_BIG_P(x); \  # 调用PERM_BIG_P函数，传入x数组作为参数
        for (u = 0; u < 16; u ++) \  # 循环16次
            H[u] ^= x[u]; \  # 将H数组中的数据与x数组中的数据进行异或运算，并存储在H数组中
    } while (0)  # do-while循环，循环体为0



#else

static const sph_u32 T0up[] = {  # 定义静态的32位整数数组T0up
    C32e(0xc632f4a5), C32e(0xf86f9784), C32e(0xee5eb099), C32e(0xf67a8c8d),
    C32e(0xffe8170d), C32e(0xd60adcbd), C32e(0xde16c8b1), C32e(0x916dfc54),
    C32e(0x6090f050), C32e(0x02070503), C32e(0xce2ee0a9), C32e(0x56d1877d),
    C32e(0xe7cc2b19), C32e(0xb513a662), C32e(0x4d7c31e6), C32e(0xec59b59a),
    C32e(0x8f40cf45), C32e(0x1fa3bc9d), C32e(0x8949c040), C32e(0xfa689287),
    C32e(0xefd03f15), C32e(0xb29426eb), C32e(0x8ece40c9), C32e(0xfbe61d0b),
    C32e(0x416e2fec), C32e(0xb31aa967), C32e(0x5f431cfd), C32e(0x456025ea),
    C32e(0x23f9dabf), C32e(0x535102f7), C32e(0xe445a196), C32e(0x9b76ed5b),
    C32e(0x75285dc2), C32e(0xe1c5241c), C32e(0x3dd4e9ae), C32e(0x4cf2be6a),
    C32e(0x6c82ee5a), C32e(0x7ebdc341), C32e(0xf5f30602), C32e(0x8352d14f),
    # 以下是一系列的函数调用，每个函数调用都使用了C32e函数对参数进行处理
    # 这些函数调用可能是在进行数据处理或者加密操作
    C32e(0x688ce45c), C32e(0x515607f4), C32e(0xd18d5c34), C32e(0xf9e11808),
    C32e(0xe24cae93), C32e(0xab3e9573), C32e(0x6297f553), C32e(0x2a6b413f),
    C32e(0x081c140c), C32e(0x9563f652), C32e(0x46e9af65), C32e(0x9d7fe25e),
    C32e(0x30487828), C32e(0x37cff8a1), C32e(0x0a1b110f), C32e(0x2febc4b5),
    C32e(0x0e151b09), C32e(0x247e5a36), C32e(0x1badb69b), C32e(0xdf98473d),
    C32e(0xcda76a26), C32e(0x4ef5bb69), C32e(0x7f334ccd), C32e(0xea50ba9f),
    C32e(0x123f2d1b), C32e(0x1da4b99e), C32e(0x58c49c74), C32e(0x3446722e),
    C32e(0x3641772d), C32e(0xdc11cdb2), C32e(0xb49d29ee), C32e(0x5b4d16fb),
    C32e(0xa4a501f6), C32e(0x76a1d74d), C32e(0xb714a361), C32e(0x7d3449ce),
    C32e(0x52df8d7b), C32e(0xdd9f423e), C32e(0x5ecd9371), C32e(0x13b1a297),
    C32e(0xa6a204f5), C32e(0xb901b868), C32e(0x00000000), C32e(0xc1b5742c),
    C32e(0x40e0a060), C32e(0xe3c2211f), C32e(0x793a43c8), C32e(0xb69a2ced),
    C32e(0xd40dd9be), C32e(0x8d47ca46), C32e(0x671770d9), C32e(0x72afdd4b),
    C32e(0x94ed79de), C32e(0x98ff67d4), C32e(0xb09323e8), C32e(0x855bde4a),
    C32e(0xbb06bd6b), C32e(0xc5bb7e2a), C32e(0x4f7b34e5), C32e(0xedd73a16),
    C32e(0x86d254c5), C32e(0x9af862d7), C32e(0x6699ff55), C32e(0x11b6a794),
    C32e(0x8ac04acf), C32e(0xe9d93010), C32e(0x040e0a06), C32e(0xfe669881),
    C32e(0xa0ab0bf0), C32e(0x78b4cc44), C32e(0x25f0d5ba), C32e(0x4b753ee3),
    C32e(0xa2ac0ef3), C32e(0x5d4419fe), C32e(0x80db5bc0), C32e(0x0580858a),
    C32e(0x3fd3ecad), C32e(0x21fedfbc), C32e(0x70a8d848), C32e(0xf1fd0c04),
    C32e(0x63197adf), C32e(0x772f58c1), C32e(0xaf309f75), C32e(0x42e7a563),
    C32e(0x20705030), C32e(0xe5cb2e1a), C32e(0xfdef120e), C32e(0xbf08b76d),
    C32e(0x8155d44c), C32e(0x18243c14), C32e(0x26795f35), C32e(0xc3b2712f),
    C32e(0xbe8638e1), C32e(0x35c8fda2), C32e(0x88c74fcc), C32e(0x2e654b39),
    C32e(0x936af957), C32e(0x55580df2), C32e(0xfc619d82), C32e(0x7ab3c947),
    C32e(0xc827efac), C32e(0xba8832e7), C32e(0x324f7d2b), C32e(0xe642a495),
    # 这些代码看起来像是一系列函数调用，但是缺少上下文，无法确定具体作用
    # 需要更多上下文信息才能正确解释这些代码的作用
    # 这是一系列的函数调用，返回一组常量值
    C32e(0x82dc5ec3), C32e(0x29e2cbb0), C32e(0x5ac39977), C32e(0x1e2d3311),
    C32e(0x7b3d46cb), C32e(0xa8b71ffc), C32e(0x6d0c61d6), C32e(0x2c624e3a)
# 定义一个静态的无符号32位整数数组T0dn
static const sph_u32 T0dn[] = {
    # 定义一系列常量，用于后续的计算
    C32e(0xf497a5c6), C32e(0x97eb84f8), C32e(0xb0c799ee), C32e(0x8cf78df6),
    C32e(0x17e50dff), C32e(0xdcb7bdd6), C32e(0xc8a7b1de), C32e(0xfc395491),
    C32e(0xf0c05060), C32e(0x05040302), C32e(0xe087a9ce), C32e(0x87ac7d56),
    C32e(0x2bd519e7), C32e(0xa67162b5), C32e(0x319ae64d), C32e(0xb5c39aec),
    C32e(0xcf05458f), C32e(0xbc3e9d1f), C32e(0xc0094089), C32e(0x92ef87fa),
    C32e(0x3fc515ef), C32e(0x267febb2), C32e(0x4007c98e), C32e(0x1ded0bfb),
    C32e(0x2f82ec41), C32e(0xa97d67b3), C32e(0x1cbefd5f), C32e(0x258aea45),
    C32e(0xda46bf23), C32e(0x02a6f753), C32e(0xa1d396e4), C32e(0xed2d5b9b),
    C32e(0x5deac275), C32e(0x24d91ce1), C32e(0xe97aae3d), C32e(0xbe986a4c),
    C32e(0xeed85a6c), C32e(0xc3fc417e), C32e(0x06f102f5), C32e(0xd11d4f83),
    C32e(0xe4d05c68), C32e(0x07a2f451), C32e(0x5cb934d1), C32e(0x18e908f9),
    C32e(0xaedf93e2), C32e(0x954d73ab), C32e(0xf5c45362), C32e(0x41543f2a),
    C32e(0x14100c08), C32e(0xf6315295), C32e(0xaf8c6546), C32e(0xe2215e9d),
    C32e(0x78602830), C32e(0xf86ea137), C32e(0x11140f0a), C32e(0xc45eb52f),
    C32e(0x1b1c090e), C32e(0x5a483624), C32e(0xb6369b1b), C32e(0x47a53ddf),
    C32e(0x6a8126cd), C32e(0xbb9c694e), C32e(0x4cfecd7f), C32e(0xbacf9fea),
    C32e(0x2d241b12), C32e(0xb93a9e1d), C32e(0x9cb07458), C32e(0x72682e34),
    C32e(0x776c2d36), C32e(0xcda3b2dc), C32e(0x2973eeb4), C32e(0x16b6fb5b),
    C32e(0x0153f6a4), C32e(0xd7ec4d76), C32e(0xa37561b7), C32e(0x49face7d),
    C32e(0x8da47b52), C32e(0x42a13edd), C32e(0x93bc715e), C32e(0xa2269713),
    C32e(0x0457f5a6), C32e(0xb86968b9), C32e(0x00000000), C32e(0x74992cc1),
    C32e(0xa0806040), C32e(0x21dd1fe3), C32e(0x43f2c879), C32e(0x2c77edb6),
    C32e(0xd9b3bed4), C32e(0xca01468d), C32e(0x70ced967), C32e(0xdde44b72),
    C32e(0x7933de94), C32e(0x672bd498), C32e(0x237be8b0), C32e(0xde114a85),
    C32e(0xbd6d6bbb), C32e(0x7e912ac5), C32e(0x349ee54f), C32e(0x3ac116ed),
    # ...
    # 定义一个长列表，包含一系列的常量值
    C32e(0x5417c586), C32e(0x622fd79a), C32e(0xffcc5566), C32e(0xa7229411),
    C32e(0x4a0fcf8a), C32e(0x30c910e9), C32e(0x0a080604), C32e(0x98e781fe),
    C32e(0x0b5bf0a0), C32e(0xccf04478), C32e(0xd54aba25), C32e(0x3e96e34b),
    C32e(0x0e5ff3a2), C32e(0x19bafe5d), C32e(0x5b1bc080), C32e(0x850a8a05),
    C32e(0xec7ead3f), C32e(0xdf42bc21), C32e(0xd8e04870), C32e(0x0cf904f1),
    C32e(0x7ac6df63), C32e(0x58eec177), C32e(0x9f4575af), C32e(0xa5846342),
    C32e(0x50403020), C32e(0x2ed11ae5), C32e(0x12e10efd), C32e(0xb7656dbf),
    C32e(0xd4194c81), C32e(0x3c301418), C32e(0x5f4c3526), C32e(0x719d2fc3),
    C32e(0x3867e1be), C32e(0xfd6aa235), C32e(0x4f0bcc88), C32e(0x4b5c392e),
    C32e(0xf93d5793), C32e(0x0daaf255), C32e(0x9de382fc), C32e(0xc9f4477a),
    C32e(0xef8bacc8), C32e(0x326fe7ba), C32e(0x7d642b32), C32e(0xa4d795e6),
    C32e(0xfb9ba0c0), C32e(0xb3329819), C32e(0x6827d19e), C32e(0x815d7fa3),
    C32e(0xaa886644), C32e(0x82a87e54), C32e(0xe676ab3b), C32e(0x9e16830b),
    C32e(0x4503ca8c), C32e(0x7b9529c7), C32e(0x6ed6d36b), C32e(0x44503c28),
    C32e(0x8b5579a7), C32e(0x3d63e2bc), C32e(0x272c1d16), C32e(0x9a4176ad),
    C32e(0x4dad3bdb), C32e(0xfac85664), C32e(0xd2e84e74), C32e(0x22281e14),
    C32e(0x763fdb92), C32e(0x1e180a0c), C32e(0xb4906c48), C32e(0x376be4b8),
    C32e(0xe7255d9f), C32e(0xb2616ebd), C32e(0x2a86ef43), C32e(0xf193a6c4),
    C32e(0xe372a839), C32e(0xf762a431), C32e(0x59bd37d3), C32e(0x86ff8bf2),
    C32e(0x56b132d5), C32e(0xc50d438b), C32e(0xebdc596e), C32e(0xc2afb7da),
    C32e(0x8f028c01), C32e(0xac7964b1), C32e(0x6d23d29c), C32e(0x3b92e049),
    C32e(0xc7abb4d8), C32e(0x1543faac), C32e(0x09fd07f3), C32e(0x6f8525cf),
    C32e(0xea8fafca), C32e(0x89f38ef4), C32e(0x208ee947), C32e(0x28201810),
    C32e(0x64ded56f), C32e(0x83fb88f0), C32e(0xb1946f4a), C32e(0x96b8725c),
    C32e(0x6c702438), C32e(0x08aef157), C32e(0x52e6c773), C32e(0xf3355197),
    C32e(0x658d23cb), C32e(0x84597ca1), C32e(0xbfcb9ce8), C32e(0x637c213e),
    # 以下是一系列调用函数 C32e() 的结果，每个函数调用返回一个值
    C32e(0x7c37dd96), C32e(0x7fc2dc61), C32e(0x911a860d), C32e(0x941e850f),
    C32e(0xabdb90e0), C32e(0xc6f8427c), C32e(0x57e2c471), C32e(0xe583aacc),
    C32e(0x733bd890), C32e(0x0f0c0506), C32e(0x03f501f7), C32e(0x3638121c),
    C32e(0xfe9fa3c2), C32e(0xe1d45f6a), C32e(0x1047f9ae), C32e(0x6bd2d069),
    C32e(0xa82e9117), C32e(0xe8295899), C32e(0x6974273a), C32e(0xd04eb927),
    C32e(0x48a938d9), C32e(0x35cd13eb), C32e(0xce56b32b), C32e(0x55443322),
    C32e(0xd6bfbbd2), C32e(0x904970a9), C32e(0x800e8907), C32e(0xf266a733),
    C32e(0xc15ab62d), C32e(0x6678223c), C32e(0xad2a9215), C32e(0x608920c9),
    C32e(0xdb154987), C32e(0x1a4fffaa), C32e(0x88a07850), C32e(0x8e517aa5),
    C32e(0x8a068f03), C32e(0x13b2f859), C32e(0x9b128009), C32e(0x3934171a),
    C32e(0x75cada65), C32e(0x53b531d7), C32e(0x5113c684), C32e(0xd3bbb8d0),
    C32e(0x5e1fc382), C32e(0xcb52b029), C32e(0x99b4775a), C32e(0x333c111e),
    C32e(0x46f6cb7b), C32e(0x1f4bfca8), C32e(0x61dad66d), C32e(0x4e583a2c)
# 定义一个静态的无符号32位整数数组T1up
static const sph_u32 T1up[] = {
    # 定义一系列的32位无符号整数常量，并将其赋值给数组T1up的元素
    C32e(0xc6c632f4), C32e(0xf8f86f97), C32e(0xeeee5eb0), C32e(0xf6f67a8c),
    C32e(0xffffe817), C32e(0xd6d60adc), C32e(0xdede16c8), C32e(0x91916dfc),
    C32e(0x606090f0), C32e(0x02020705), C32e(0xcece2ee0), C32e(0x5656d187),
    C32e(0xe7e7cc2b), C32e(0xb5b513a6), C32e(0x4d4d7c31), C32e(0xecec59b5),
    C32e(0x8f8f40cf), C32e(0x1f1fa3bc), C32e(0x898949c0), C32e(0xfafa6892),
    C32e(0xefefd03f), C32e(0xb2b29426), C32e(0x8e8ece40), C32e(0xfbfbe61d),
    C32e(0x41416e2f), C32e(0xb3b31aa9), C32e(0x5f5f431c), C32e(0x45456025),
    C32e(0x2323f9da), C32e(0x53535102), C32e(0xe4e445a1), C32e(0x9b9b76ed),
    C32e(0x7575285d), C32e(0xe1e1c524), C32e(0x3d3dd4e9), C32e(0x4c4cf2be),
    C32e(0x6c6c82ee), C32e(0x7e7ebdc3), C32e(0xf5f5f306), C32e(0x838352d1),
    C32e(0x68688ce4), C32e(0x51515607), C32e(0xd1d18d5c), C32e(0xf9f9e118),
    C32e(0xe2e24cae), C32e(0xabab3e95), C32e(0x626297f5), C32e(0x2a2a6b41),
    C32e(0x08081c14), C32e(0x959563f6), C32e(0x4646e9af), C32e(0x9d9d7fe2),
    C32e(0x30304878), C32e(0x3737cff8), C32e(0x0a0a1b11), C32e(0x2f2febc4),
    C32e(0x0e0e151b), C32e(0x24247e5a), C32e(0x1b1badb6), C32e(0xdfdf9847),
    C32e(0xcdcda76a), C32e(0x4e4ef5bb), C32e(0x7f7f334c), C32e(0xeaea50ba),
    C32e(0x12123f2d), C32e(0x1d1da4b9), C32e(0x5858c49c), C32e(0x34344672),
    C32e(0x36364177), C32e(0xdcdc11cd), C32e(0xb4b49d29), C32e(0x5b5b4d16),
    C32e(0xa4a4a501), C32e(0x7676a1d7), C32e(0xb7b714a3), C32e(0x7d7d3449),
    C32e(0x5252df8d), C32e(0xdddd9f42), C32e(0x5e5ecd93), C32e(0x1313b1a2),
    C32e(0xa6a6a204), C32e(0xb9b901b8), C32e(0x00000000), C32e(0xc1c1b574),
    C32e(0x4040e0a0), C32e(0xe3e3c221), C32e(0x79793a43), C32e(0xb6b69a2c),
    C32e(0xd4d40dd9), C32e(0x8d8d47ca), C32e(0x67671770), C32e(0x7272afdd),
    C32e(0x9494ed79), C32e(0x9898ff67), C32e(0xb0b09323), C32e(0x85855bde),
    C32e(0xbbbb06bd), C32e(0xc5c5bb7e), C32e(0x4f4f7b34), C32e(0xededd73a),
    # 定义一个包含多个常量的列表，这些常量是通过调用C32e函数得到的
    C32e(0x8686d254), C32e(0x9a9af862), C32e(0x666699ff), C32e(0x1111b6a7),
    C32e(0x8a8ac04a), C32e(0xe9e9d930), C32e(0x04040e0a), C32e(0xfefe6698),
    C32e(0xa0a0ab0b), C32e(0x7878b4cc), C32e(0x2525f0d5), C32e(0x4b4b753e),
    C32e(0xa2a2ac0e), C32e(0x5d5d4419), C32e(0x8080db5b), C32e(0x05058085),
    C32e(0x3f3fd3ec), C32e(0x2121fedf), C32e(0x7070a8d8), C32e(0xf1f1fd0c),
    C32e(0x6363197a), C32e(0x77772f58), C32e(0xafaf309f), C32e(0x4242e7a5),
    C32e(0x20207050), C32e(0xe5e5cb2e), C32e(0xfdfdef12), C32e(0xbfbf08b7),
    C32e(0x818155d4), C32e(0x1818243c), C32e(0x2626795f), C32e(0xc3c3b271),
    C32e(0xbebe8638), C32e(0x3535c8fd), C32e(0x8888c74f), C32e(0x2e2e654b),
    C32e(0x93936af9), C32e(0x5555580d), C32e(0xfcfc619d), C32e(0x7a7ab3c9),
    C32e(0xc8c827ef), C32e(0xbaba8832), C32e(0x32324f7d), C32e(0xe6e642a4),
    C32e(0xc0c03bfb), C32e(0x1919aab3), C32e(0x9e9ef668), C32e(0xa3a32281),
    C32e(0x4444eeaa), C32e(0x5454d682), C32e(0x3b3bdde6), C32e(0x0b0b959e),
    C32e(0x8c8cc945), C32e(0xc7c7bc7b), C32e(0x6b6b056e), C32e(0x28286c44),
    C32e(0xa7a72c8b), C32e(0xbcbc813d), C32e(0x16163127), C32e(0xadad379a),
    C32e(0xdbdb964d), C32e(0x64649efa), C32e(0x7474a6d2), C32e(0x14143622),
    C32e(0x9292e476), C32e(0x0c0c121e), C32e(0x4848fcb4), C32e(0xb8b88f37),
    C32e(0x9f9f78e7), C32e(0xbdbd0fb2), C32e(0x4343692a), C32e(0xc4c435f1),
    C32e(0x3939dae3), C32e(0x3131c6f7), C32e(0xd3d38a59), C32e(0xf2f27486),
    C32e(0xd5d58356), C32e(0x8b8b4ec5), C32e(0x6e6e85eb), C32e(0xdada18c2),
    C32e(0x01018e8f), C32e(0xb1b11dac), C32e(0x9c9cf16d), C32e(0x4949723b),
    C32e(0xd8d81fc7), C32e(0xacacb915), C32e(0xf3f3fa09), C32e(0xcfcfa06f),
    C32e(0xcaca20ea), C32e(0xf4f47d89), C32e(0x47476720), C32e(0x10103828),
    C32e(0x6f6f0b64), C32e(0xf0f07383), C32e(0x4a4afbb1), C32e(0x5c5cca96),
    C32e(0x3838546c), C32e(0x57575f08), C32e(0x73732152), C32e(0x979764f3),
    C32e(0xcbcbae65), C32e(0xa1a12584), C32e(0xe8e857bf), C32e(0x3e3e5d63),
    # 定义一系列常量，这些常量是通过调用C32e函数得到的结果
    C32e(0x9696ea7c), C32e(0x61611e7f), C32e(0x0d0d9c91), C32e(0x0f0f9b94),
    C32e(0xe0e04bab), C32e(0x7c7cbac6), C32e(0x71712657), C32e(0xcccc29e5),
    C32e(0x9090e373), C32e(0x0606090f), C32e(0xf7f7f403), C32e(0x1c1c2a36),
    C32e(0xc2c23cfe), C32e(0x6a6a8be1), C32e(0xaeaebe10), C32e(0x6969026b),
    C32e(0x1717bfa8), C32e(0x999971e8), C32e(0x3a3a5369), C32e(0x2727f7d0),
    C32e(0xd9d99148), C32e(0xebebde35), C32e(0x2b2be5ce), C32e(0x22227755),
    C32e(0xd2d204d6), C32e(0xa9a93990), C32e(0x07078780), C32e(0x3333c1f2),
    C32e(0x2d2decc1), C32e(0x3c3c5a66), C32e(0x1515b8ad), C32e(0xc9c9a960),
    C32e(0x87875cdb), C32e(0xaaaab01a), C32e(0x5050d888), C32e(0xa5a52b8e),
    C32e(0x0303898a), C32e(0x59594a13), C32e(0x0909929b), C32e(0x1a1a2339),
    C32e(0x65651075), C32e(0xd7d78453), C32e(0x8484d551), C32e(0xd0d003d3),
    C32e(0x8282dc5e), C32e(0x2929e2cb), C32e(0x5a5ac399), C32e(0x1e1e2d33),
    C32e(0x7b7b3d46), C32e(0xa8a8b71f), C32e(0x6d6d0c61), C32e(0x2c2c624e)
# 定义静态常量数组 T1dn
static const sph_u32 T1dn[] = {
    # 定义一系列32位无符号整数常量，并用C32e宏进行转换
    C32e(0xa5f497a5), C32e(0x8497eb84), C32e(0x99b0c799), C32e(0x8d8cf78d),
    C32e(0x0d17e50d), C32e(0xbddcb7bd), C32e(0xb1c8a7b1), C32e(0x54fc3954),
    C32e(0x50f0c050), C32e(0x03050403), C32e(0xa9e087a9), C32e(0x7d87ac7d),
    C32e(0x192bd519), C32e(0x62a67162), C32e(0xe6319ae6), C32e(0x9ab5c39a),
    C32e(0x45cf0545), C32e(0x9dbc3e9d), C32e(0x40c00940), C32e(0x8792ef87),
    C32e(0x153fc515), C32e(0xeb267feb), C32e(0xc94007c9), C32e(0x0b1ded0b),
    C32e(0xec2f82ec), C32e(0x67a97d67), C32e(0xfd1cbefd), C32e(0xea258aea),
    C32e(0xbfda46bf), C32e(0xf702a6f7), C32e(0x96a1d396), C32e(0x5bed2d5b),
    C32e(0xc25deac2), C32e(0x1c24d91c), C32e(0xaee97aae), C32e(0x6abe986a),
    C32e(0x5aeed85a), C32e(0x41c3fc41), C32e(0x0206f102), C32e(0x4fd11d4f),
    C32e(0x5ce4d05c), C32e(0xf407a2f4), C32e(0x345cb934), C32e(0x0818e908),
    C32e(0x93aedf93), C32e(0x73954d73), C32e(0x53f5c453), C32e(0x3f41543f),
    C32e(0x0c14100c), C32e(0x52f63152), C32e(0x65af8c65), C32e(0x5ee2215e),
    C32e(0x28786028), C32e(0xa1f86ea1), C32e(0x0f11140f), C32e(0xb5c45eb5),
    C32e(0x091b1c09), C32e(0x365a4836), C32e(0x9bb6369b), C32e(0x3d47a53d),
    C32e(0x266a8126), C32e(0x69bb9c69), C32e(0xcd4cfecd), C32e(0x9fbacf9f),
    C32e(0x1b2d241b), C32e(0x9eb93a9e), C32e(0x749cb074), C32e(0x2e72682e),
    C32e(0x2d776c2d), C32e(0xb2cda3b2), C32e(0xee2973ee), C32e(0xfb16b6fb),
    C32e(0xf60153f6), C32e(0x4dd7ec4d), C32e(0x61a37561), C32e(0xce49face),
    C32e(0x7b8da47b), C32e(0x3e42a13e), C32e(0x7193bc71), C32e(0x97a22697),
    C32e(0xf50457f5), C32e(0x68b86968), C32e(0x00000000), C32e(0x2c74992c),
    C32e(0x60a08060), C32e(0x1f21dd1f), C32e(0xc843f2c8), C32e(0xed2c77ed),
    C32e(0xbed9b3be), C32e(0x46ca0146), C32e(0xd970ced9), C32e(0x4bdde44b),
    C32e(0xde7933de), C32e(0xd4672bd4), C32e(0xe8237be8), C32e(0x4ade114a),
    C32e(0x6bbd6d6b), C32e(0x2a7e912a), C32e(0xe5349ee5), C32e(0x163ac116),
    # 以下是一系列的函数调用，每个函数调用都是对常量进行处理并返回结果
    C32e(0xc55417c5), C32e(0xd7622fd7), C32e(0x55ffcc55), C32e(0x94a72294),
    C32e(0xcf4a0fcf), C32e(0x1030c910), C32e(0x060a0806), C32e(0x8198e781),
    C32e(0xf00b5bf0), C32e(0x44ccf044), C32e(0xbad54aba), C32e(0xe33e96e3),
    C32e(0xf30e5ff3), C32e(0xfe19bafe), C32e(0xc05b1bc0), C32e(0x8a850a8a),
    C32e(0xadec7ead), C32e(0xbcdf42bc), C32e(0x48d8e048), C32e(0x040cf904),
    C32e(0xdf7ac6df), C32e(0xc158eec1), C32e(0x759f4575), C32e(0x63a58463),
    C32e(0x30504030), C32e(0x1a2ed11a), C32e(0x0e12e10e), C32e(0x6db7656d),
    C32e(0x4cd4194c), C32e(0x143c3014), C32e(0x355f4c35), C32e(0x2f719d2f),
    C32e(0xe13867e1), C32e(0xa2fd6aa2), C32e(0xcc4f0bcc), C32e(0x394b5c39),
    C32e(0x57f93d57), C32e(0xf20daaf2), C32e(0x829de382), C32e(0x47c9f447),
    C32e(0xacef8bac), C32e(0xe7326fe7), C32e(0x2b7d642b), C32e(0x95a4d795),
    C32e(0xa0fb9ba0), C32e(0x98b33298), C32e(0xd16827d1), C32e(0x7f815d7f),
    C32e(0x66aa8866), C32e(0x7e82a87e), C32e(0xabe676ab), C32e(0x839e1683),
    C32e(0xca4503ca), C32e(0x297b9529), C32e(0xd36ed6d3), C32e(0x3c44503c),
    C32e(0x798b5579), C32e(0xe23d63e2), C32e(0x1d272c1d), C32e(0x769a4176),
    C32e(0x3b4dad3b), C32e(0x56fac856), C32e(0x4ed2e84e), C32e(0x1e22281e),
    C32e(0xdb763fdb), C32e(0x0a1e180a), C32e(0x6cb4906c), C32e(0xe4376be4),
    C32e(0x5de7255d), C32e(0x6eb2616e), C32e(0xef2a86ef), C32e(0xa6f193a6),
    C32e(0xa8e372a8), C32e(0xa4f762a4), C32e(0x3759bd37), C32e(0x8b86ff8b),
    C32e(0x3256b132), C32e(0x43c50d43), C32e(0x59ebdc59), C32e(0xb7c2afb7),
    C32e(0x8c8f028c), C32e(0x64ac7964), C32e(0xd26d23d2), C32e(0xe03b92e0),
    C32e(0xb4c7abb4), C32e(0xfa1543fa), C32e(0x0709fd07), C32e(0x256f8525),
    C32e(0xafea8faf), C32e(0x8e89f38e), C32e(0xe9208ee9), C32e(0x18282018),
    C32e(0xd564ded5), C32e(0x8883fb88), C32e(0x6fb1946f), C32e(0x7296b872),
    C32e(0x246c7024), C32e(0xf108aef1), C32e(0xc752e6c7), C32e(0x51f33551),
    C32e(0x23658d23), C32e(0x7c84597c), C32e(0x9cbfcb9c), C32e(0x21637c21),
    # 以下是一系列经过C32e函数处理的十六进制数值
    C32e(0xdd7c37dd), C32e(0xdc7fc2dc), C32e(0x86911a86), C32e(0x85941e85),
    C32e(0x90abdb90), C32e(0x42c6f842), C32e(0xc457e2c4), C32e(0xaae583aa),
    C32e(0xd8733bd8), C32e(0x050f0c05), C32e(0x0103f501), C32e(0x12363812),
    C32e(0xa3fe9fa3), C32e(0x5fe1d45f), C32e(0xf91047f9), C32e(0xd06bd2d0),
    C32e(0x91a82e91), C32e(0x58e82958), C32e(0x27697427), C32e(0xb9d04eb9),
    C32e(0x3848a938), C32e(0x1335cd13), C32e(0xb3ce56b3), C32e(0x33554433),
    C32e(0xbbd6bfbb), C32e(0x70904970), C32e(0x89800e89), C32e(0xa7f266a7),
    C32e(0xb6c15ab6), C32e(0x22667822), C32e(0x92ad2a92), C32e(0x20608920),
    C32e(0x49db1549), C32e(0xff1a4fff), C32e(0x7888a078), C32e(0x7a8e517a),
    C32e(0x8f8a068f), C32e(0xf813b2f8), C32e(0x809b1280), C32e(0x17393417),
    C32e(0xda75cada), C32e(0x3153b531), C32e(0xc65113c6), C32e(0xb8d3bbb8),
    C32e(0xc35e1fc3), C32e(0xb0cb52b0), C32e(0x7799b477), C32e(0x11333c11),
    C32e(0xcb46f6cb), C32e(0xfc1f4bfc), C32e(0xd661dad6), C32e(0x3a4e583a)
// 定义静态常量数组 T2up
static const sph_u32 T2up[] = {
    // 以下为数组元素的数值初始化
    C32e(0xa5c6c632), C32e(0x84f8f86f), C32e(0x99eeee5e), C32e(0x8df6f67a),
    C32e(0x0dffffe8), C32e(0xbdd6d60a), C32e(0xb1dede16), C32e(0x5491916d),
    C32e(0x50606090), C32e(0x03020207), C32e(0xa9cece2e), C32e(0x7d5656d1),
    C32e(0x19e7e7cc), C32e(0x62b5b513), C32e(0xe64d4d7c), C32e(0x9aecec59),
    C32e(0x458f8f40), C32e(0x9d1f1fa3), C32e(0x40898949), C32e(0x87fafa68),
    C32e(0x15efefd0), C32e(0xebb2b294), C32e(0xc98e8ece), C32e(0x0bfbfbe6),
    C32e(0xec41416e), C32e(0x67b3b31a), C32e(0xfd5f5f43), C32e(0xea454560),
    C32e(0xbf2323f9), C32e(0xf7535351), C32e(0x96e4e445), C32e(0x5b9b9b76),
    C32e(0xc2757528), C32e(0x1ce1e1c5), C32e(0xae3d3dd4), C32e(0x6a4c4cf2),
    C32e(0x5a6c6c82), C32e(0x417e7ebd), C32e(0x02f5f5f3), C32e(0x4f838352),
    C32e(0x5c68688c), C32e(0xf4515156), C32e(0x34d1d18d), C32e(0x08f9f9e1),
    C32e(0x93e2e24c), C32e(0x73abab3e), C32e(0x53626297), C32e(0x3f2a2a6b),
    C32e(0x0c08081c), C32e(0x52959563), C32e(0x654646e9), C32e(0x5e9d9d7f),
    C32e(0x28303048), C32e(0xa13737cf), C32e(0x0f0a0a1b), C32e(0xb52f2feb),
    C32e(0x090e0e15), C32e(0x3624247e), C32e(0x9b1b1bad), C32e(0x3ddfdf98),
    C32e(0x26cdcda7), C32e(0x694e4ef5), C32e(0xcd7f7f33), C32e(0x9feaea50),
    C32e(0x1b12123f), C32e(0x9e1d1da4), C32e(0x745858c4), C32e(0x2e343446),
    C32e(0x2d363641), C32e(0xb2dcdc11), C32e(0xeeb4b49d), C32e(0xfb5b5b4d),
    C32e(0xf6a4a4a5), C32e(0x4d7676a1), C32e(0x61b7b714), C32e(0xce7d7d34),
    C32e(0x7b5252df), C32e(0x3edddd9f), C32e(0x715e5ecd), C32e(0x971313b1),
    C32e(0xf5a6a6a2), C32e(0x68b9b901), C32e(0x00000000), C32e(0x2cc1c1b5),
    C32e(0x604040e0), C32e(0x1fe3e3c2), C32e(0xc879793a), C32e(0xedb6b69a),
    C32e(0xbed4d40d), C32e(0x468d8d47), C32e(0xd9676717), C32e(0x4b7272af),
    C32e(0xde9494ed), C32e(0xd49898ff), C32e(0xe8b0b093), C32e(0x4a85855b),
    C32e(0x6bbbbb06), C32e(0x2ac5c5bb), C32e(0xe54f4f7b), C32e(0x16ededd7),
    // 数组元素初始化结束
    # 以下是一系列的常量值，采用C32e函数进行转换
    C32e(0xc58686d2), C32e(0xd79a9af8), C32e(0x55666699), C32e(0x941111b6),
    C32e(0xcf8a8ac0), C32e(0x10e9e9d9), C32e(0x0604040e), C32e(0x81fefe66),
    C32e(0xf0a0a0ab), C32e(0x447878b4), C32e(0xba2525f0), C32e(0xe34b4b75),
    C32e(0xf3a2a2ac), C32e(0xfe5d5d44), C32e(0xc08080db), C32e(0x8a050580),
    C32e(0xad3f3fd3), C32e(0xbc2121fe), C32e(0x487070a8), C32e(0x04f1f1fd),
    C32e(0xdf636319), C32e(0xc177772f), C32e(0x75afaf30), C32e(0x634242e7),
    C32e(0x30202070), C32e(0x1ae5e5cb), C32e(0x0efdfdef), C32e(0x6dbfbf08),
    C32e(0x4c818155), C32e(0x14181824), C32e(0x35262679), C32e(0x2fc3c3b2),
    C32e(0xe1bebe86), C32e(0xa23535c8), C32e(0xcc8888c7), C32e(0x392e2e65),
    C32e(0x5793936a), C32e(0xf2555558), C32e(0x82fcfc61), C32e(0x477a7ab3),
    C32e(0xacc8c827), C32e(0xe7baba88), C32e(0x2b32324f), C32e(0x95e6e642),
    C32e(0xa0c0c03b), C32e(0x981919aa), C32e(0xd19e9ef6), C32e(0x7fa3a322),
    C32e(0x664444ee), C32e(0x7e5454d6), C32e(0xab3b3bdd), C32e(0x830b0b95),
    C32e(0xca8c8cc9), C32e(0x29c7c7bc), C32e(0xd36b6b05), C32e(0x3c28286c),
    C32e(0x79a7a72c), C32e(0xe2bcbc81), C32e(0x1d161631), C32e(0x76adad37),
    C32e(0x3bdbdb96), C32e(0x5664649e), C32e(0x4e7474a6), C32e(0x1e141436),
    C32e(0xdb9292e4), C32e(0x0a0c0c12), C32e(0x6c4848fc), C32e(0xe4b8b88f),
    C32e(0x5d9f9f78), C32e(0x6ebdbd0f), C32e(0xef434369), C32e(0xa6c4c435),
    C32e(0xa83939da), C32e(0xa43131c6), C32e(0x37d3d38a), C32e(0x8bf2f274),
    C32e(0x32d5d583), C32e(0x438b8b4e), C32e(0x596e6e85), C32e(0xb7dada18),
    C32e(0x8c01018e), C32e(0x64b1b11d), C32e(0xd29c9cf1), C32e(0xe0494972),
    C32e(0xb4d8d81f), C32e(0xfaacacb9), C32e(0x07f3f3fa), C32e(0x25cfcfa0),
    C32e(0xafcaca20), C32e(0x8ef4f47d), C32e(0xe9474767), C32e(0x18101038),
    C32e(0xd56f6f0b), C32e(0x88f0f073), C32e(0x6f4a4afb), C32e(0x725c5cca),
    C32e(0x24383854), C32e(0xf157575f), C32e(0xc7737321), C32e(0x51979764),
    C32e(0x23cbcbae), C32e(0x7ca1a125), C32e(0x9ce8e857), C32e(0x213e3e5d),
    # 以下是一系列调用函数 C32e() 的结果，每个函数调用都返回一个 32 位的值
    C32e(0xdd9696ea), C32e(0xdc61611e), C32e(0x860d0d9c), C32e(0x850f0f9b),
    C32e(0x90e0e04b), C32e(0x427c7cba), C32e(0xc4717126), C32e(0xaacccc29),
    C32e(0xd89090e3), C32e(0x05060609), C32e(0x01f7f7f4), C32e(0x121c1c2a),
    C32e(0xa3c2c23c), C32e(0x5f6a6a8b), C32e(0xf9aeaebe), C32e(0xd0696902),
    C32e(0x911717bf), C32e(0x58999971), C32e(0x273a3a53), C32e(0xb92727f7),
    C32e(0x38d9d991), C32e(0x13ebebde), C32e(0xb32b2be5), C32e(0x33222277),
    C32e(0xbbd2d204), C32e(0x70a9a939), C32e(0x89070787), C32e(0xa73333c1),
    C32e(0xb62d2dec), C32e(0x223c3c5a), C32e(0x921515b8), C32e(0x20c9c9a9),
    C32e(0x4987875c), C32e(0xffaaaab0), C32e(0x785050d8), C32e(0x7aa5a52b),
    C32e(0x8f030389), C32e(0xf859594a), C32e(0x80090992), C32e(0x171a1a23),
    C32e(0xda656510), C32e(0x31d7d784), C32e(0xc68484d5), C32e(0xb8d0d003),
    C32e(0xc38282dc), C32e(0xb02929e2), C32e(0x775a5ac3), C32e(0x111e1e2d),
    C32e(0xcb7b7b3d), C32e(0xfca8a8b7), C32e(0xd66d6d0c), C32e(0x3a2c2c62)
# 定义静态常量数组 T2dn
static const sph_u32 T2dn[] = {
    # 定义 32 位常量，使用 C32e 宏进行转换
    C32e(0xf4a5f497), C32e(0x978497eb), C32e(0xb099b0c7), C32e(0x8c8d8cf7),
    C32e(0x170d17e5), C32e(0xdcbddcb7), C32e(0xc8b1c8a7), C32e(0xfc54fc39),
    C32e(0xf050f0c0), C32e(0x05030504), C32e(0xe0a9e087), C32e(0x877d87ac),
    C32e(0x2b192bd5), C32e(0xa662a671), C32e(0x31e6319a), C32e(0xb59ab5c3),
    C32e(0xcf45cf05), C32e(0xbc9dbc3e), C32e(0xc040c009), C32e(0x928792ef),
    C32e(0x3f153fc5), C32e(0x26eb267f), C32e(0x40c94007), C32e(0x1d0b1ded),
    C32e(0x2fec2f82), C32e(0xa967a97d), C32e(0x1cfd1cbe), C32e(0x25ea258a),
    C32e(0xdabfda46), C32e(0x02f702a6), C32e(0xa196a1d3), C32e(0xed5bed2d),
    C32e(0x5dc25dea), C32e(0x241c24d9), C32e(0xe9aee97a), C32e(0xbe6abe98),
    C32e(0xee5aeed8), C32e(0xc341c3fc), C32e(0x060206f1), C32e(0xd14fd11d),
    C32e(0xe45ce4d0), C32e(0x07f407a2), C32e(0x5c345cb9), C32e(0x180818e9),
    C32e(0xae93aedf), C32e(0x9573954d), C32e(0xf553f5c4), C32e(0x413f4154),
    C32e(0x140c1410), C32e(0xf652f631), C32e(0xaf65af8c), C32e(0xe25ee221),
    C32e(0x78287860), C32e(0xf8a1f86e), C32e(0x110f1114), C32e(0xc4b5c45e),
    C32e(0x1b091b1c), C32e(0x5a365a48), C32e(0xb69bb636), C32e(0x473d47a5),
    C32e(0x6a266a81), C32e(0xbb69bb9c), C32e(0x4ccd4cfe), C32e(0xba9fbacf),
    C32e(0x2d1b2d24), C32e(0xb99eb93a), C32e(0x9c749cb0), C32e(0x722e7268),
    C32e(0x772d776c), C32e(0xcdb2cda3), C32e(0x29ee2973), C32e(0x16fb16b6),
    C32e(0x01f60153), C32e(0xd74dd7ec), C32e(0xa361a375), C32e(0x49ce49fa),
    C32e(0x8d7b8da4), C32e(0x423e42a1), C32e(0x937193bc), C32e(0xa297a226),
    C32e(0x04f50457), C32e(0xb868b869), C32e(0x00000000), C32e(0x742c7499),
    C32e(0xa060a080), C32e(0x211f21dd), C32e(0x43c843f2), C32e(0x2ced2c77),
    C32e(0xd9bed9b3), C32e(0xca46ca01), C32e(0x70d970ce), C32e(0xdd4bdde4),
    C32e(0x79de7933), C32e(0x67d4672b), C32e(0x23e8237b), C32e(0xde4ade11),
    C32e(0xbd6bbd6d), C32e(0x7e2a7e91), C32e(0x34e5349e), C32e(0x3a163ac1),
    # 定义一个包含一系列常量的列表
    C32e(0x54c55417), C32e(0x62d7622f), C32e(0xff55ffcc), C32e(0xa794a722),
    C32e(0x4acf4a0f), C32e(0x301030c9), C32e(0x0a060a08), C32e(0x988198e7),
    C32e(0x0bf00b5b), C32e(0xcc44ccf0), C32e(0xd5bad54a), C32e(0x3ee33e96),
    C32e(0x0ef30e5f), C32e(0x19fe19ba), C32e(0x5bc05b1b), C32e(0x858a850a),
    C32e(0xecadec7e), C32e(0xdfbcdf42), C32e(0xd848d8e0), C32e(0x0c040cf9),
    C32e(0x7adf7ac6), C32e(0x58c158ee), C32e(0x9f759f45), C32e(0xa563a584),
    C32e(0x50305040), C32e(0x2e1a2ed1), C32e(0x120e12e1), C32e(0xb76db765),
    C32e(0xd44cd419), C32e(0x3c143c30), C32e(0x5f355f4c), C32e(0x712f719d),
    C32e(0x38e13867), C32e(0xfda2fd6a), C32e(0x4fcc4f0b), C32e(0x4b394b5c),
    C32e(0xf957f93d), C32e(0x0df20daa), C32e(0x9d829de3), C32e(0xc947c9f4),
    C32e(0xefacef8b), C32e(0x32e7326f), C32e(0x7d2b7d64), C32e(0xa495a4d7),
    C32e(0xfba0fb9b), C32e(0xb398b332), C32e(0x68d16827), C32e(0x817f815d),
    C32e(0xaa66aa88), C32e(0x827e82a8), C32e(0xe6abe676), C32e(0x9e839e16),
    C32e(0x45ca4503), C32e(0x7b297b95), C32e(0x6ed36ed6), C32e(0x443c4450),
    C32e(0x8b798b55), C32e(0x3de23d63), C32e(0x271d272c), C32e(0x9a769a41),
    C32e(0x4d3b4dad), C32e(0xfa56fac8), C32e(0xd24ed2e8), C32e(0x221e2228),
    C32e(0x76db763f), C32e(0x1e0a1e18), C32e(0xb46cb490), C32e(0x37e4376b),
    C32e(0xe75de725), C32e(0xb26eb261), C32e(0x2aef2a86), C32e(0xf1a6f193),
    C32e(0xe3a8e372), C32e(0xf7a4f762), C32e(0x593759bd), C32e(0x868b86ff),
    C32e(0x563256b1), C32e(0xc543c50d), C32e(0xeb59ebdc), C32e(0xc2b7c2af),
    C32e(0x8f8c8f02), C32e(0xac64ac79), C32e(0x6dd26d23), C32e(0x3be03b92),
    C32e(0xc7b4c7ab), C32e(0x15fa1543), C32e(0x090709fd), C32e(0x6f256f85),
    C32e(0xeaafea8f), C32e(0x898e89f3), C32e(0x20e9208e), C32e(0x28182820),
    C32e(0x64d564de), C32e(0x838883fb), C32e(0xb16fb194), C32e(0x967296b8),
    C32e(0x6c246c70), C32e(0x08f108ae), C32e(0x52c752e6), C32e(0xf351f335),
    C32e(0x6523658d), C32e(0x847c8459), C32e(0xbf9cbfcb), C32e(0x6321637c),
    # 以下是一系列调用函数 C32e() 的语句，每个函数调用都返回一个特定的值
    # 这些值可能是哈希值、加密值或其他特定的数值
    # 每个函数调用的参数都是一个十六进制数
    # 这些值可能用于密码学、哈希算法或其他加密相关的操作
// 定义一个静态的无符号32位整数数组T3up
static const sph_u32 T3up[] = {
    // 定义一系列32位的常量值
    C32e(0x97a5c6c6), C32e(0xeb84f8f8), C32e(0xc799eeee), C32e(0xf78df6f6),
    C32e(0xe50dffff), C32e(0xb7bdd6d6), C32e(0xa7b1dede), C32e(0x39549191),
    C32e(0xc0506060), C32e(0x04030202), C32e(0x87a9cece), C32e(0xac7d5656),
    C32e(0xd519e7e7), C32e(0x7162b5b5), C32e(0x9ae64d4d), C32e(0xc39aecec),
    C32e(0x05458f8f), C32e(0x3e9d1f1f), C32e(0x09408989), C32e(0xef87fafa),
    C32e(0xc515efef), C32e(0x7febb2b2), C32e(0x07c98e8e), C32e(0xed0bfbfb),
    C32e(0x82ec4141), C32e(0x7d67b3b3), C32e(0xbefd5f5f), C32e(0x8aea4545),
    C32e(0x46bf2323), C32e(0xa6f75353), C32e(0xd396e4e4), C32e(0x2d5b9b9b),
    C32e(0xeac27575), C32e(0xd91ce1e1), C32e(0x7aae3d3d), C32e(0x986a4c4c),
    C32e(0xd85a6c6c), C32e(0xfc417e7e), C32e(0xf102f5f5), C32e(0x1d4f8383),
    C32e(0xd05c6868), C32e(0xa2f45151), C32e(0xb934d1d1), C32e(0xe908f9f9),
    C32e(0xdf93e2e2), C32e(0x4d73abab), C32e(0xc4536262), C32e(0x543f2a2a),
    C32e(0x100c0808), C32e(0x31529595), C32e(0x8c654646), C32e(0x215e9d9d),
    C32e(0x60283030), C32e(0x6ea13737), C32e(0x140f0a0a), C32e(0x5eb52f2f),
    C32e(0x1c090e0e), C32e(0x48362424), C32e(0x369b1b1b), C32e(0xa53ddfdf),
    C32e(0x8126cdcd), C32e(0x9c694e4e), C32e(0xfecd7f7f), C32e(0xcf9feaea),
    C32e(0x241b1212), C32e(0x3a9e1d1d), C32e(0xb0745858), C32e(0x682e3434),
    C32e(0x6c2d3636), C32e(0xa3b2dcdc), C32e(0x73eeb4b4), C32e(0xb6fb5b5b),
    C32e(0x53f6a4a4), C32e(0xec4d7676), C32e(0x7561b7b7), C32e(0xface7d7d),
    C32e(0xa47b5252), C32e(0xa13edddd), C32e(0xbc715e5e), C32e(0x26971313),
    C32e(0x57f5a6a6), C32e(0x6968b9b9), C32e(0x00000000), C32e(0x992cc1c1),
    C32e(0x80604040), C32e(0xdd1fe3e3), C32e(0xf2c87979), C32e(0x77edb6b6),
    C32e(0xb3bed4d4), C32e(0x01468d8d), C32e(0xced96767), C32e(0xe44b7272),
    C32e(0x33de9494), C32e(0x2bd49898), C32e(0x7be8b0b0), C32e(0x114a8585),
    C32e(0x6d6bbbbb), C32e(0x912ac5c5), C32e(0x9ee54f4f), C32e(0xc116eded),
    # 以下是一系列的函数调用，每个函数调用都使用了一个十六进制数作为参数
    # 这些函数调用可能是对数据进行加密、解密或其他处理
    # 每个函数调用的具体作用需要根据函数定义来确定
    C32e(0x17c58686), C32e(0x2fd79a9a), C32e(0xcc556666), C32e(0x22941111),
    C32e(0x0fcf8a8a), C32e(0xc910e9e9), C32e(0x08060404), C32e(0xe781fefe),
    C32e(0x5bf0a0a0), C32e(0xf0447878), C32e(0x4aba2525), C32e(0x96e34b4b),
    C32e(0x5ff3a2a2), C32e(0xbafe5d5d), C32e(0x1bc08080), C32e(0x0a8a0505),
    C32e(0x7ead3f3f), C32e(0x42bc2121), C32e(0xe0487070), C32e(0xf904f1f1),
    C32e(0xc6df6363), C32e(0xeec17777), C32e(0x4575afaf), C32e(0x84634242),
    C32e(0x40302020), C32e(0xd11ae5e5), C32e(0xe10efdfd), C32e(0x656dbfbf),
    C32e(0x194c8181), C32e(0x30141818), C32e(0x4c352626), C32e(0x9d2fc3c3),
    C32e(0x67e1bebe), C32e(0x6aa23535), C32e(0x0bcc8888), C32e(0x5c392e2e),
    C32e(0x3d579393), C32e(0xaaf25555), C32e(0xe382fcfc), C32e(0xf4477a7a),
    C32e(0x8bacc8c8), C32e(0x6fe7baba), C32e(0x642b3232), C32e(0xd795e6e6),
    C32e(0x9ba0c0c0), C32e(0x32981919), C32e(0x27d19e9e), C32e(0x5d7fa3a3),
    C32e(0x88664444), C32e(0xa87e5454), C32e(0x76ab3b3b), C32e(0x16830b0b),
    C32e(0x03ca8c8c), C32e(0x9529c7c7), C32e(0xd6d36b6b), C32e(0x503c2828),
    C32e(0x5579a7a7), C32e(0x63e2bcbc), C32e(0x2c1d1616), C32e(0x4176adad),
    C32e(0xad3bdbdb), C32e(0xc8566464), C32e(0xe84e7474), C32e(0x281e1414),
    C32e(0x3fdb9292), C32e(0x180a0c0c), C32e(0x906c4848), C32e(0x6be4b8b8),
    C32e(0x255d9f9f), C32e(0x616ebdbd), C32e(0x86ef4343), C32e(0x93a6c4c4),
    C32e(0x72a83939), C32e(0x62a43131), C32e(0xbd37d3d3), C32e(0xff8bf2f2),
    C32e(0xb132d5d5), C32e(0x0d438b8b), C32e(0xdc596e6e), C32e(0xafb7dada),
    C32e(0x028c0101), C32e(0x7964b1b1), C32e(0x23d29c9c), C32e(0x92e04949),
    C32e(0xabb4d8d8), C32e(0x43faacac), C32e(0xfd07f3f3), C32e(0x8525cfcf),
    C32e(0x8fafcaca), C32e(0xf38ef4f4), C32e(0x8ee94747), C32e(0x20181010),
    C32e(0xded56f6f), C32e(0xfb88f0f0), C32e(0x946f4a4a), C32e(0xb8725c5c),
    C32e(0x70243838), C32e(0xaef15757), C32e(0xe6c77373), C32e(0x35519797),
    C32e(0x8d23cbcb), C32e(0x597ca1a1), C32e(0xcb9ce8e8), C32e(0x7c213e3e),
    # 定义一个包含一系列常量的列表
    C32e(0x37dd9696), C32e(0xc2dc6161), C32e(0x1a860d0d), C32e(0x1e850f0f),
    C32e(0xdb90e0e0), C32e(0xf8427c7c), C32e(0xe2c47171), C32e(0x83aacccc),
    C32e(0x3bd89090), C32e(0x0c050606), C32e(0xf501f7f7), C32e(0x38121c1c),
    C32e(0x9fa3c2c2), C32e(0xd45f6a6a), C32e(0x47f9aeae), C32e(0xd2d06969),
    C32e(0x2e911717), C32e(0x29589999), C32e(0x74273a3a), C32e(0x4eb92727),
    C32e(0xa938d9d9), C32e(0xcd13ebeb), C32e(0x56b32b2b), C32e(0x44332222),
    C32e(0xbfbbd2d2), C32e(0x4970a9a9), C32e(0x0e890707), C32e(0x66a73333),
    C32e(0x5ab62d2d), C32e(0x78223c3c), C32e(0x2a921515), C32e(0x8920c9c9),
    C32e(0x15498787), C32e(0x4fffaaaa), C32e(0xa0785050), C32e(0x517aa5a5),
    C32e(0x068f0303), C32e(0xb2f85959), C32e(0x12800909), C32e(0x34171a1a),
    C32e(0xcada6565), C32e(0xb531d7d7), C32e(0x13c68484), C32e(0xbbb8d0d0),
    C32e(0x1fc38282), C32e(0x52b02929), C32e(0xb4775a5a), C32e(0x3c111e1e),
    C32e(0xf6cb7b7b), C32e(0x4bfca8a8), C32e(0xdad66d6d), C32e(0x583a2c2c)
# 定义一个静态常量数组 T3dn，包含一系列32位无符号整数
static const sph_u32 T3dn[] = {
    # 将十六进制数转换为32位无符号整数，并添加到数组中
    C32e(0x32f4a5f4), C32e(0x6f978497), C32e(0x5eb099b0), C32e(0x7a8c8d8c),
    C32e(0xe8170d17), C32e(0x0adcbddc), C32e(0x16c8b1c8), C32e(0x6dfc54fc),
    C32e(0x90f050f0), C32e(0x07050305), C32e(0x2ee0a9e0), C32e(0xd1877d87),
    C32e(0xcc2b192b), C32e(0x13a662a6), C32e(0x7c31e631), C32e(0x59b59ab5),
    C32e(0x40cf45cf), C32e(0xa3bc9dbc), C32e(0x49c040c0), C32e(0x68928792),
    C32e(0xd03f153f), C32e(0x9426eb26), C32e(0xce40c940), C32e(0xe61d0b1d),
    C32e(0x6e2fec2f), C32e(0x1aa967a9), C32e(0x431cfd1c), C32e(0x6025ea25),
    C32e(0xf9dabfda), C32e(0x5102f702), C32e(0x45a196a1), C32e(0x76ed5bed),
    C32e(0x285dc25d), C32e(0xc5241c24), C32e(0xd4e9aee9), C32e(0xf2be6abe),
    C32e(0x82ee5aee), C32e(0xbdc341c3), C32e(0xf3060206), C32e(0x52d14fd1),
    C32e(0x8ce45ce4), C32e(0x5607f407), C32e(0x8d5c345c), C32e(0xe1180818),
    C32e(0x4cae93ae), C32e(0x3e957395), C32e(0x97f553f5), C32e(0x6b413f41),
    C32e(0x1c140c14), C32e(0x63f652f6), C32e(0xe9af65af), C32e(0x7fe25ee2),
    C32e(0x48782878), C32e(0xcff8a1f8), C32e(0x1b110f11), C32e(0xebc4b5c4),
    C32e(0x151b091b), C32e(0x7e5a365a), C32e(0xadb69bb6), C32e(0x98473d47),
    C32e(0xa76a266a), C32e(0xf5bb69bb), C32e(0x334ccd4c), C32e(0x50ba9fba),
    C32e(0x3f2d1b2d), C32e(0xa4b99eb9), C32e(0xc49c749c), C32e(0x46722e72),
    C32e(0x41772d77), C32e(0x11cdb2cd), C32e(0x9d29ee29), C32e(0x4d16fb16),
    C32e(0xa501f601), C32e(0xa1d74dd7), C32e(0x14a361a3), C32e(0x3449ce49),
    C32e(0xdf8d7b8d), C32e(0x9f423e42), C32e(0xcd937193), C32e(0xb1a297a2),
    C32e(0xa204f504), C32e(0x01b868b8), C32e(0x00000000), C32e(0xb5742c74),
    C32e(0xe0a060a0), C32e(0xc2211f21), C32e(0x3a43c843), C32e(0x9a2ced2c),
    C32e(0x0dd9bed9), C32e(0x47ca46ca), C32e(0x1770d970), C32e(0xafdd4bdd),
    C32e(0xed79de79), C32e(0xff67d467), C32e(0x9323e823), C32e(0x5bde4ade),
    C32e(0x06bd6bbd), C32e(0xbb7e2a7e), C32e(0x7b34e534), C32e(0xd73a163a),
    # ...
};
    # 以下是一系列的函数调用，每个函数调用的参数是一个十六进制数
    C32e(0xd254c554), C32e(0xf862d762), C32e(0x99ff55ff), C32e(0xb6a794a7),
    C32e(0xc04acf4a), C32e(0xd9301030), C32e(0x0e0a060a), C32e(0x66988198),
    C32e(0xab0bf00b), C32e(0xb4cc44cc), C32e(0xf0d5bad5), C32e(0x753ee33e),
    C32e(0xac0ef30e), C32e(0x4419fe19), C32e(0xdb5bc05b), C32e(0x80858a85),
    C32e(0xd3ecadec), C32e(0xfedfbcdf), C32e(0xa8d848d8), C32e(0xfd0c040c),
    C32e(0x197adf7a), C32e(0x2f58c158), C32e(0x309f759f), C32e(0xe7a563a5),
    C32e(0x70503050), C32e(0xcb2e1a2e), C32e(0xef120e12), C32e(0x08b76db7),
    C32e(0x55d44cd4), C32e(0x243c143c), C32e(0x795f355f), C32e(0xb2712f71),
    C32e(0x8638e138), C32e(0xc8fda2fd), C32e(0xc74fcc4f), C32e(0x654b394b),
    C32e(0x6af957f9), C32e(0x580df20d), C32e(0x619d829d), C32e(0xb3c947c9),
    C32e(0x27efacef), C32e(0x8832e732), C32e(0x4f7d2b7d), C32e(0x42a495a4),
    C32e(0x3bfba0fb), C32e(0xaab398b3), C32e(0xf668d168), C32e(0x22817f81),
    C32e(0xeeaa66aa), C32e(0xd6827e82), C32e(0xdde6abe6), C32e(0x959e839e),
    C32e(0xc945ca45), C32e(0xbc7b297b), C32e(0x056ed36e), C32e(0x6c443c44),
    C32e(0x2c8b798b), C32e(0x813de23d), C32e(0x31271d27), C32e(0x379a769a),
    C32e(0x964d3b4d), C32e(0x9efa56fa), C32e(0xa6d24ed2), C32e(0x36221e22),
    C32e(0xe476db76), C32e(0x121e0a1e), C32e(0xfcb46cb4), C32e(0x8f37e437),
    C32e(0x78e75de7), C32e(0x0fb26eb2), C32e(0x692aef2a), C32e(0x35f1a6f1),
    C32e(0xdae3a8e3), C32e(0xc6f7a4f7), C32e(0x8a593759), C32e(0x74868b86),
    C32e(0x83563256), C32e(0x4ec543c5), C32e(0x85eb59eb), C32e(0x18c2b7c2),
    C32e(0x8e8f8c8f), C32e(0x1dac64ac), C32e(0xf16dd26d), C32e(0x723be03b),
    C32e(0x1fc7b4c7), C32e(0xb915fa15), C32e(0xfa090709), C32e(0xa06f256f),
    C32e(0x20eaafea), C32e(0x7d898e89), C32e(0x6720e920), C32e(0x38281828),
    C32e(0x0b64d564), C32e(0x73838883), C32e(0xfbb16fb1), C32e(0xca967296),
    C32e(0x546c246c), C32e(0x5f08f108), C32e(0x2152c752), C32e(0x64f351f3),
    C32e(0xae652365), C32e(0x25847c84), C32e(0x57bf9cbf), C32e(0x5d632163),
    # 定义一系列经过C32e函数处理的常量值
    C32e(0xea7cdd7c), C32e(0x1e7fdc7f), C32e(0x9c918691), C32e(0x9b948594),
    C32e(0x4bab90ab), C32e(0xbac642c6), C32e(0x2657c457), C32e(0x29e5aae5),
    C32e(0xe373d873), C32e(0x090f050f), C32e(0xf4030103), C32e(0x2a361236),
    C32e(0x3cfea3fe), C32e(0x8be15fe1), C32e(0xbe10f910), C32e(0x026bd06b),
    C32e(0xbfa891a8), C32e(0x71e858e8), C32e(0x53692769), C32e(0xf7d0b9d0),
    C32e(0x91483848), C32e(0xde351335), C32e(0xe5ceb3ce), C32e(0x77553355),
    C32e(0x04d6bbd6), C32e(0x39907090), C32e(0x87808980), C32e(0xc1f2a7f2),
    C32e(0xecc1b6c1), C32e(0x5a662266), C32e(0xb8ad92ad), C32e(0xa9602060),
    C32e(0x5cdb49db), C32e(0xb01aff1a), C32e(0xd8887888), C32e(0x2b8e7a8e),
    C32e(0x898a8f8a), C32e(0x4a13f813), C32e(0x929b809b), C32e(0x23391739),
    C32e(0x1075da75), C32e(0x84533153), C32e(0xd551c651), C32e(0x03d3b8d3),
    C32e(0xdc5ec35e), C32e(0xe2cbb0cb), C32e(0xc3997799), C32e(0x2d331133),
    C32e(0x3d46cb46), C32e(0xb71ffc1f), C32e(0x0c61d661), C32e(0x624e3a4e)
# 定义一个宏，用于声明一个包含16个32位无符号整数的数组H
#define DECL_STATE_SMALL \
    sph_u32 H[16];

# 定义一个宏，用于将状态sc的narrow字段的内容复制到数组H中
#define READ_STATE_SMALL(sc)   do { \
        memcpy(H, (sc)->state.narrow, sizeof H); \
    } while (0)

# 定义一个宏，用于将数组H的内容复制到状态sc的narrow字段中
#define WRITE_STATE_SMALL(sc)   do { \
        memcpy((sc)->state.narrow, H, sizeof H); \
    } while (0)

# 定义两个宏，用于将两个标识符连接成一个新的标识符
#define XCAT(x, y)    XCAT_(x, y)
#define XCAT_(x, y)   x ## y

# 定义一个宏，用于重置t数组的值，根据给定的索引和a数组的值进行计算
#define RSTT(d0, d1, a, b0, b1, b2, b3, b4, b5, b6, b7)   do { \
        t[d0] = T0up[B32_0(a[b0])] \
            ^ T1up[B32_1(a[b1])] \
            ^ T2up[B32_2(a[b2])] \
            ^ T3up[B32_3(a[b3])] \
            ^ T0dn[B32_0(a[b4])] \
            ^ T1dn[B32_1(a[b5])] \
            ^ T2dn[B32_2(a[b6])] \
            ^ T3dn[B32_3(a[b7])]; \
        t[d1] = T0dn[B32_0(a[b0])] \
            ^ T1dn[B32_1(a[b1])] \
            ^ T2dn[B32_2(a[b2])] \
            ^ T3dn[B32_3(a[b3])] \
            ^ T0up[B32_0(a[b4])] \
            ^ T1up[B32_1(a[b5])] \
            ^ T2up[B32_2(a[b6])] \
            ^ T3up[B32_3(a[b7])]; \
    } while (0)
# 定义一个宏函数，用于对输入的数组a进行一系列操作
#define ROUND_SMALL_P(a, r)   do { \
        # 定义一个临时数组t，用于存储中间结果
        sph_u32 t[16]; \
        # 对数组a的每个元素进行异或操作
        a[0x0] ^= PC32up(0x00, r); \
        a[0x1] ^= PC32dn(0x00, r); \
        a[0x2] ^= PC32up(0x10, r); \
        a[0x3] ^= PC32dn(0x10, r); \
        a[0x4] ^= PC32up(0x20, r); \
        a[0x5] ^= PC32dn(0x20, r); \
        a[0x6] ^= PC32up(0x30, r); \
        a[0x7] ^= PC32dn(0x30, r); \
        a[0x8] ^= PC32up(0x40, r); \
        a[0x9] ^= PC32dn(0x40, r); \
        a[0xA] ^= PC32up(0x50, r); \
        a[0xB] ^= PC32dn(0x50, r); \
        a[0xC] ^= PC32up(0x60, r); \
        a[0xD] ^= PC32dn(0x60, r); \
        a[0xE] ^= PC32up(0x70, r); \
        a[0xF] ^= PC32dn(0x70, r); \
        # 调用RSTT宏函数对数组a进行一系列操作
        RSTT(0x0, 0x1, a, 0x0, 0x2, 0x4, 0x6, 0x9, 0xB, 0xD, 0xF); \
        RSTT(0x2, 0x3, a, 0x2, 0x4, 0x6, 0x8, 0xB, 0xD, 0xF, 0x1); \
        RSTT(0x4, 0x5, a, 0x4, 0x6, 0x8, 0xA, 0xD, 0xF, 0x1, 0x3); \
        RSTT(0x6, 0x7, a, 0x6, 0x8, 0xA, 0xC, 0xF, 0x1, 0x3, 0x5); \
        RSTT(0x8, 0x9, a, 0x8, 0xA, 0xC, 0xE, 0x1, 0x3, 0x5, 0x7); \
        RSTT(0xA, 0xB, a, 0xA, 0xC, 0xE, 0x0, 0x3, 0x5, 0x7, 0x9); \
        RSTT(0xC, 0xD, a, 0xC, 0xE, 0x0, 0x2, 0x5, 0x7, 0x9, 0xB); \
        RSTT(0xE, 0xF, a, 0xE, 0x0, 0x2, 0x4, 0x7, 0x9, 0xB, 0xD); \
        # 将临时数组t的内容复制回数组a
        memcpy(a, t, sizeof t); \
    } while (0)
# 定义一个宏函数ROUND_SMALL_Q，用于对给定的数组a进行一轮小型置换操作
#define ROUND_SMALL_Q(a, r)   do { \
        # 定义一个临时数组t，用于存储中间结果
        sph_u32 t[16]; \
        # 对数组a的每个元素进行异或操作，异或的值是通过宏函数QC32up和QC32dn计算得到的
        a[0x0] ^= QC32up(0x00, r); \
        a[0x1] ^= QC32dn(0x00, r); \
        a[0x2] ^= QC32up(0x10, r); \
        a[0x3] ^= QC32dn(0x10, r); \
        a[0x4] ^= QC32up(0x20, r); \
        a[0x5] ^= QC32dn(0x20, r); \
        a[0x6] ^= QC32up(0x30, r); \
        a[0x7] ^= QC32dn(0x30, r); \
        a[0x8] ^= QC32up(0x40, r); \
        a[0x9] ^= QC32dn(0x40, r); \
        a[0xA] ^= QC32up(0x50, r); \
        a[0xB] ^= QC32dn(0x50, r); \
        a[0xC] ^= QC32up(0x60, r); \
        a[0xD] ^= QC32dn(0x60, r); \
        a[0xE] ^= QC32up(0x70, r); \
        a[0xF] ^= QC32dn(0x70, r); \
        # 调用宏函数RSTT对数组a进行一轮置换操作
        RSTT(0x0, 0x1, a, 0x2, 0x6, 0xA, 0xE, 0x1, 0x5, 0x9, 0xD); \
        RSTT(0x2, 0x3, a, 0x4, 0x8, 0xC, 0x0, 0x3, 0x7, 0xB, 0xF); \
        RSTT(0x4, 0x5, a, 0x6, 0xA, 0xE, 0x2, 0x5, 0x9, 0xD, 0x1); \
        RSTT(0x6, 0x7, a, 0x8, 0xC, 0x0, 0x4, 0x7, 0xB, 0xF, 0x3); \
        RSTT(0x8, 0x9, a, 0xA, 0xE, 0x2, 0x6, 0x9, 0xD, 0x1, 0x5); \
        RSTT(0xA, 0xB, a, 0xC, 0x0, 0x4, 0x8, 0xB, 0xF, 0x3, 0x7); \
        RSTT(0xC, 0xD, a, 0xE, 0x2, 0x6, 0xA, 0xD, 0x1, 0x5, 0x9); \
        RSTT(0xE, 0xF, a, 0x0, 0x4, 0x8, 0xC, 0xF, 0x3, 0x7, 0xB); \
        # 将临时数组t的内容复制回数组a
        memcpy(a, t, sizeof t); \
    } while (0)

#if SPH_SMALL_FOOTPRINT_GROESTL

#define PERM_SMALL_P(a)   do { \
        # 定义一个变量r，用于循环计数
        int r; \
        # 对数组a进行10轮小型置换操作
        for (r = 0; r < 10; r ++) \
            ROUND_SMALL_P(a, r); \
    } while (0)

#define PERM_SMALL_Q(a)   do { \
        # 定义一个变量r，用于循环计数
        int r; \
        # 对数组a进行10轮小型置换操作
        for (r = 0; r < 10; r ++) \
            ROUND_SMALL_Q(a, r); \
    } while (0)

#else

#define PERM_SMALL_P(a)   do { \
        # 定义一个变量r，用于循环计数
        int r; \
        # 对数组a进行5轮小型置换操作
        for (r = 0; r < 10; r += 2) { \
            ROUND_SMALL_P(a, r + 0); \
            ROUND_SMALL_P(a, r + 1); \
        } \
    } while (0)

#define PERM_SMALL_Q(a)   do { \
        # 定义一个变量r，用于循环计数
        int r; \
        # 对数组a进行5轮小型置换操作
        for (r = 0; r < 10; r += 2) { \
            ROUND_SMALL_Q(a, r + 0); \
            ROUND_SMALL_Q(a, r + 1); \
        } \
    } while (0)

#endif
# 定义压缩函数，对小块数据进行压缩
#define COMPRESS_SMALL   do { \
        # 定义局部变量，包括两个数组和一个整型变量
        sph_u32 g[16], m[16]; \
        size_t u; \
        # 遍历16个元素，将buf中的数据解码为m数组，同时与H数组进行异或操作得到g数组
        for (u = 0; u < 16; u ++) { \
            m[u] = dec32e_aligned(buf + (u << 2)); \
            g[u] = m[u] ^ H[u]; \
        } \
        # 对g数组进行置换操作
        PERM_SMALL_P(g); \
        # 对m数组进行置换操作
        PERM_SMALL_Q(m); \
        # 将g数组、m数组和H数组进行异或操作
        for (u = 0; u < 16; u ++) \
            H[u] ^= g[u] ^ m[u]; \
    } while (0)

# 定义最终处理函数，对小块数据进行最终处理
#define FINAL_SMALL   do { \
        # 定义局部变量，包括一个数组和一个整型变量
        sph_u32 x[16]; \
        size_t u; \
        # 将H数组的内容复制到x数组
        memcpy(x, H, sizeof x); \
        # 对x数组进行置换操作
        PERM_SMALL_P(x); \
        # 将x数组和H数组进行异或操作
        for (u = 0; u < 16; u ++) \
            H[u] ^= x[u]; \
    } while (0)

# 定义大状态声明，包括一个32个元素的数组
#define DECL_STATE_BIG \
    sph_u32 H[32];

# 定义读取大状态的宏，将状态sc中的内容复制到H数组
#define READ_STATE_BIG(sc)   do { \
        memcpy(H, (sc)->state.narrow, sizeof H); \
    } while (0)

# 定义写入大状态的宏，将H数组的内容复制到状态sc中
#define WRITE_STATE_BIG(sc)   do { \
        memcpy((sc)->state.narrow, H, sizeof H); \
    } while (0)

# 如果使用小内存占用的Groestl算法
#if SPH_SMALL_FOOTPRINT_GROESTL

# 定义RBTT宏，对数据进行一系列的操作
#define RBTT(d0, d1, a, b0, b1, b2, b3, b4, b5, b6, b7)   do { \
        # 定义局部变量，包括8个整型变量
        sph_u32 fu2 = T0up[B32_2(a[b2])]; \
        sph_u32 fd2 = T0dn[B32_2(a[b2])]; \
        sph_u32 fu3 = T1up[B32_3(a[b3])]; \
        sph_u32 fd3 = T1dn[B32_3(a[b3])]; \
        sph_u32 fu6 = T0up[B32_2(a[b6])]; \
        sph_u32 fd6 = T0dn[B32_2(a[b6])]; \
        sph_u32 fu7 = T1up[B32_3(a[b7])]; \
        sph_u32 fd7 = T1dn[B32_3(a[b7])]; \
        # 对数据进行一系列的异或和置换操作，得到t数组
        t[d0] = T0up[B32_0(a[b0])] \
            ^ T1up[B32_1(a[b1])] \
            ^ R32u(fu2, fd2) \
            ^ R32u(fu3, fd3) \
            ^ T0dn[B32_0(a[b4])] \
            ^ T1dn[B32_1(a[b5])] \
            ^ R32d(fu6, fd6) \
            ^ R32d(fu7, fd7); \
        t[d1] = T0dn[B32_0(a[b0])] \
            ^ T1dn[B32_1(a[b1])] \
            ^ R32d(fu2, fd2) \
            ^ R32d(fu3, fd3) \
            ^ T0up[B32_0(a[b4])] \
            ^ T1up[B32_1(a[b5])] \
            ^ R32u(fu6, fd6) \
            ^ R32u(fu7, fd7); \
    } while (0)

#else
#define RBTT(d0, d1, a, b0, b1, b2, b3, b4, b5, b6, b7)   do { \
        t[d0] = T0up[B32_0(a[b0])] \
            ^ T1up[B32_1(a[b1])] \
            ^ T2up[B32_2(a[b2])] \
            ^ T3up[B32_3(a[b3])] \
            ^ T0dn[B32_0(a[b4])] \
            ^ T1dn[B32_1(a[b5])] \
            ^ T2dn[B32_2(a[b6])] \
            ^ T3dn[B32_3(a[b7])]; \
        t[d1] = T0dn[B32_0(a[b0])] \
            ^ T1dn[B32_1(a[b1])] \
            ^ T2dn[B32_2(a[b2])] \
            ^ T3dn[B32_3(a[b3])] \
            ^ T0up[B32_0(a[b4])] \
            ^ T1up[B32_1(a[b5])] \
            ^ T2up[B32_2(a[b6])] \
            ^ T3up[B32_3(a[b7])]; \
    } while (0)

这是一个宏定义，用于定义一个函数。函数的作用是根据给定的参数，计算并返回一个结果。


#endif

这是一个条件编译的结束标志。


#if SPH_SMALL_FOOTPRINT_GROESTL

    } while (0)

    } while (0)

#else

    } while (0)

    } while (0)

#endif

这是一个条件编译的开始标志，根据条件`SPH_SMALL_FOOTPRINT_GROESTL`的值，决定编译哪一部分代码。


#if SPH_SMALL_FOOTPRINT_GROESTL

#define PERM_BIG_P(a)   do { \
        int r; \
        for (r = 0; r < 14; r ++) \
            ROUND_BIG_P(a, r); \
    } while (0)

#define PERM_BIG_Q(a)   do { \
        int r; \
        for (r = 0; r < 14; r ++) \
            ROUND_BIG_Q(a, r); \
    } while (0)

#else

#define PERM_BIG_P(a)   do { \
        int r; \
        for (r = 0; r < 14; r += 2) { \
            ROUND_BIG_P(a, r + 0); \
            ROUND_BIG_P(a, r + 1); \
        } \
    } while (0)

#define PERM_BIG_Q(a)   do { \
        int r; \
        for (r = 0; r < 14; r += 2) { \
            ROUND_BIG_Q(a, r + 0); \
            ROUND_BIG_Q(a, r + 1); \
        } \
    } while (0)

#endif

这是一个条件编译的开始标志，根据条件`SPH_SMALL_FOOTPRINT_GROESTL`的值，定义不同的宏。


#define COMPRESS_BIG   do { \
        sph_u32 g[32], m[32]; \
        size_t u; \
        for (u = 0; u < 32; u ++) { \
            m[u] = dec32e_aligned(buf + (u << 2)); \
            g[u] = m[u] ^ H[u]; \
        } \
        PERM_BIG_P(g); \
        PERM_BIG_Q(m); \
        for (u = 0; u < 32; u ++) \
            H[u] ^= g[u] ^ m[u]; \
    } while (0)

这是一个宏定义，用于定义一个函数。函数的作用是根据给定的参数，计算并返回一个结果。
#define FINAL_BIG   do { \  # 定义一个宏，用于执行一系列操作
        sph_u32 x[32]; \  # 声明一个包含32个sph_u32类型元素的数组x
        size_t u; \  # 声明一个变量u，用于循环计数
        memcpy(x, H, sizeof x); \  # 将数组H的内容复制到数组x中
        PERM_BIG_P(x); \  # 调用PERM_BIG_P函数对数组x进行处理
        for (u = 0; u < 32; u ++) \  # 循环遍历数组x
            H[u] ^= x[u]; \  # 对数组H的每个元素与数组x对应元素进行异或操作
    } while (0)  # 宏定义结束

#endif  # 结束条件编译指令

static void  # 声明一个静态函数
groestl_small_init(sph_groestl_small_context *sc, unsigned out_size)  # 函数名及参数列表
{
    size_t u;  # 声明一个变量u，用于循环计数

    sc->ptr = 0;  # 将结构体sc的ptr成员赋值为0
#if SPH_GROESTL_64  # 如果SPH_GROESTL_64宏被定义
    for (u = 0; u < 7; u ++)  # 循环遍历7次
        sc->state.wide[u] = 0;  # 将结构体sc的state.wide数组的每个元素赋值为0
#if USE_LE  # 如果USE_LE宏被定义
    sc->state.wide[7] = ((sph_u64)(out_size & 0xFF) << 56)  # 将结构体sc的state.wide数组的第7个元素赋值为指定的位操作结果
        | ((sph_u64)(out_size & 0xFF00) << 40);  # 将结构体sc的state.wide数组的第7个元素赋值为指定的位操作结果
#else  # 如果USE_LE宏未被定义
    sc->state.wide[7] = (sph_u64)out_size;  # 将结构体sc的state.wide数组的第7个元素赋值为out_size的值
#endif  # 结束条件编译指令
#else  # 如果SPH_GROESTL_64宏未被定义
    for (u = 0; u < 15; u ++)  # 循环遍历15次
        sc->state.narrow[u] = 0;  # 将结构体sc的state.narrow数组的每个元素赋值为0
#if USE_LE  # 如果USE_LE宏被定义
    sc->state.narrow[15] = ((sph_u32)(out_size & 0xFF) << 24)  # 将结构体sc的state.narrow数组的第15个元素赋值为指定的位操作结果
        | ((sph_u32)(out_size & 0xFF00) << 8);  # 将结构体sc的state.narrow数组的第15个元素赋值为指定的位操作结果
#else  # 如果USE_LE宏未被定义
    sc->state.narrow[15] = (sph_u32)out_size;  # 将结构体sc的state.narrow数组的第15个元素赋值为out_size的值
#endif  # 结束条件编译指令
#endif  # 结束条件编译指令
#if SPH_64  # 如果SPH_64宏被定义
    sc->count = 0;  # 将结构体sc的count成员赋值为0
#else  # 如果SPH_64宏未被定义
    sc->count_high = 0;  # 将结构体sc的count_high成员赋值为0
    sc->count_low = 0;  # 将结构体sc的count_low成员赋值为0
#endif  # 结束条件编译指令
}

static void  # 声明一个静态函数
groestl_small_core(sph_groestl_small_context *sc, const void *data, size_t len)  # 函数名及参数列表
{
    unsigned char *buf;  # 声明一个指向无符号字符的指针buf
    size_t ptr;  # 声明一个变量ptr，用于记录当前位置
    DECL_STATE_SMALL  # 声明一个状态变量

    buf = sc->buf;  # 将结构体sc的buf成员赋值给buf
    ptr = sc->ptr;  # 将结构体sc的ptr成员赋值给ptr
    if (len < (sizeof sc->buf) - ptr) {  # 如果数据长度小于buf剩余空间
        memcpy(buf + ptr, data, len);  # 将数据复制到buf中
        ptr += len;  # 更新ptr的值
        sc->ptr = ptr;  # 将ptr的值赋回结构体sc的ptr成员
        return;  # 返回
    }

    READ_STATE_SMALL(sc);  # 读取状态
    while (len > 0) {  # 循环直到数据长度为0
        size_t clen;  # 声明一个变量clen，用于记录当前数据块长度

        clen = (sizeof sc->buf) - ptr;  # 计算当前数据块长度
        if (clen > len)  # 如果当前数据块长度大于剩余数据长度
            clen = len;  # 更新当前数据块长度
        memcpy(buf + ptr, data, clen);  # 将数据复制到buf中
        ptr += clen;  # 更新ptr的值
        data = (const unsigned char *)data + clen;  # 更新数据指针
        len -= clen;  # 更新剩余数据长度
        if (ptr == sizeof sc->buf) {  # 如果buf已满
            COMPRESS_SMALL;  # 执行压缩操作
#if SPH_64  # 如果SPH_64宏被定义
            sc->count ++;  # count加1
#else  # 如果SPH_64宏未被定义
            if ((sc->count_low = SPH_T32(sc->count_low + 1)) == 0)  # 如果count_low加1后为0
                sc->count_high = SPH_T32(sc->count_high + 1);  # count_high加1
#endif  # 结束条件编译指令
            ptr = 0;  # 重置ptr
        }
    }
    WRITE_STATE_SMALL(sc);  # 写入状态
    sc->ptr = ptr;  # 更新结构体sc的ptr成员
}

static void  # 声明一个静态函数
groestl_small_close(sph_groestl_small_context *sc,  # 函数名及参数列表
    # 函数签名，指定函数的参数和返回值类型
    # unsigned: 无符号整数类型
    # ub: 无符号整数类型的参数
    # n: 无符号整数类型的参数
    # void *dst: 指向任意类型的指针
    # size_t: 无符号整数类型，用于表示内存大小
    # out_len: size_t类型的参数
{
    // 定义一个包含 72 个无符号字符的数组
    unsigned char pad[72];
    // 定义 size_t 类型的变量 u, ptr, pad_len
    size_t u, ptr, pad_len;
    // 如果定义了 SPH_64，则定义 sph_u64 类型的变量 count，否则定义 sph_u32 类型的变量 count_high, count_low
#if SPH_64
    sph_u64 count;
#else
    sph_u32 count_high, count_low;
#endif
    // 定义一个无符号整数 z
    unsigned z;
    // 声明一个名为 DECL_STATE_SMALL 的宏
    DECL_STATE_SMALL

    // 将 sc 指针指向的值赋给 ptr
    ptr = sc->ptr;
    // 计算 z 的值
    z = 0x80 >> n;
    // 计算 pad[0] 的值
    pad[0] = ((ub & -z) | z) & 0xFF;
    // 如果 ptr 小于 56
    if (ptr < 56) {
        // 计算 pad_len 的值
        pad_len = 64 - ptr;
        // 如果定义了 SPH_64，则计算 count 的值，否则计算 count_high, count_low 的值
#if SPH_64
        count = SPH_T64(sc->count + 1);
#else
        count_low = SPH_T32(sc->count_low + 1);
        count_high = SPH_T32(sc->count_high);
        if (count_low == 0)
            count_high = SPH_T32(count_high + 1);
#endif
    } else {
        // 计算 pad_len 的值
        pad_len = 128 - ptr;
        // 如果定义了 SPH_64，则计算 count 的值，否则计算 count_high, count_low 的值
#if SPH_64
        count = SPH_T64(sc->count + 2);
#else
        count_low = SPH_T32(sc->count_low + 2);
        count_high = SPH_T32(sc->count_high);
        if (count_low <= 1)
            count_high = SPH_T32(count_high + 1);
#endif
    }
    // 将 pad + 1 到 pad + pad_len - 9 的值设置为 0
    memset(pad + 1, 0, pad_len - 9);
    // 如果定义了 SPH_64，则将 count 的值编码为大端格式写入 pad + pad_len - 8，否则将 count_high, count_low 的值编码为大端格式写入 pad + pad_len - 8, pad + pad_len - 4
#if SPH_64
    sph_enc64be(pad + pad_len - 8, count);
#else
    sph_enc64be(pad + pad_len - 8, count_high);
    sph_enc64be(pad + pad_len - 4, count_low);
#endif
    // 调用 groestl_small_core 函数
    groestl_small_core(sc, pad, pad_len);
    // 读取状态
    READ_STATE_SMALL(sc);
    // 调用 FINAL_SMALL 宏
    FINAL_SMALL;
    // 如果定义了 SPH_GROESTL_64，则循环执行 enc64e 操作，否则循环执行 enc32e 操作
#if SPH_GROESTL_64
    for (u = 0; u < 4; u ++)
        enc64e(pad + (u << 3), H[u + 4]);
#else
    for (u = 0; u < 8; u ++)
        enc32e(pad + (u << 2), H[u + 8]);
#endif
    // 将 pad + 32 - out_len 到 pad + 32 的值复制到 dst
    memcpy(dst, pad + 32 - out_len, out_len);
    // 调用 groestl_small_init 函数
    groestl_small_init(sc, (unsigned)out_len << 3);
}

// 定义一个名为 groestl_big_init 的静态函数，参数为 sph_groestl_big_context 类型的指针 sc 和 unsigned 类型的 out_size
static void
groestl_big_init(sph_groestl_big_context *sc, unsigned out_size)
{
    // 定义 size_t 类型的变量 u
    size_t u;

    // 将 sc->ptr 的值设置为 0
    sc->ptr = 0;
    // 如果定义了 SPH_GROESTL_64，则循环执行赋值操作，否则循环执行赋值操作
#if SPH_GROESTL_64
    for (u = 0; u < 15; u ++)
        sc->state.wide[u] = 0;
#if USE_LE
    sc->state.wide[15] = ((sph_u64)(out_size & 0xFF) << 56)
        | ((sph_u64)(out_size & 0xFF00) << 40);
#else
    sc->state.wide[15] = (sph_u64)out_size;
#endif
#else
    for (u = 0; u < 31; u ++)
        sc->state.narrow[u] = 0;
#if USE_LE
    sc->state.narrow[31] = ((sph_u32)(out_size & 0xFF) << 24)
        | ((sph_u32)(out_size & 0xFF00) << 8);
#else
    sc->state.narrow[31] = (sph_u32)out_size;
#endif
#endif
}
#if SPH_64
    # 如果定义了 SPH_64 宏，则将 sc->count 设置为 0
    sc->count = 0;
#else
    # 如果未定义 SPH_64 宏，则将 sc->count_high 和 sc->count_low 设置为 0
    sc->count_high = 0;
    sc->count_low = 0;
#endif
}

static void
groestl_big_core(sph_groestl_big_context *sc, const void *data, size_t len)
{
    # 定义指向 unsigned char 类型的指针 buf 和 size_t 类型的变量 ptr
    unsigned char *buf;
    size_t ptr;
    # 声明 STATE_BIG 宏

    # 将 sc->buf 赋值给 buf
    buf = sc->buf;
    # 将 sc->ptr 赋值给 ptr
    ptr = sc->ptr;
    # 如果 len 小于 (sizeof sc->buf) - ptr，则将 data 的内容复制到 buf + ptr 处，并更新 ptr 的值
    if (len < (sizeof sc->buf) - ptr) {
        memcpy(buf + ptr, data, len);
        ptr += len;
        sc->ptr = ptr;
        return;
    }

    # 读取 STATE_BIG 的值
    READ_STATE_BIG(sc);
    # 当 len 大于 0 时，执行循环
    while (len > 0) {
        # 定义 size_t 类型的变量 clen
        size_t clen;

        # 将 (sizeof sc->buf) - ptr 的值赋给 clen
        clen = (sizeof sc->buf) - ptr;
        # 如果 clen 大于 len，则将 len 的值赋给 clen
        if (clen > len)
            clen = len;
        # 将 data 的内容复制到 buf + ptr 处，并更新 ptr 的值
        memcpy(buf + ptr, data, clen);
        ptr += clen;
        # 将 data 强制转换为 const unsigned char 类型的指针，并更新其值
        data = (const unsigned char *)data + clen;
        # 更新 len 的值
        len -= clen;
        # 如果 ptr 的值等于 sizeof sc->buf，则执行 COMPRESS_BIG 宏，并根据 SPH_64 宏的定义更新 sc->count 的值
        if (ptr == sizeof sc->buf) {
            COMPRESS_BIG;
#if SPH_64
            sc->count ++;
#else
            if ((sc->count_low = SPH_T32(sc->count_low + 1)) == 0)
                sc->count_high = SPH_T32(sc->count_high + 1);
#endif
            ptr = 0;
        }
    }
    # 写入 STATE_BIG 的值
    WRITE_STATE_BIG(sc);
    # 更新 sc->ptr 的值
    sc->ptr = ptr;
}

static void
groestl_big_close(sph_groestl_big_context *sc,
    unsigned ub, unsigned n, void *dst, size_t out_len)
{
    # 定义 unsigned char 类型的数组 pad 和 size_t 类型的变量 ptr、pad_len、u
    unsigned char pad[136];
    size_t ptr, pad_len, u;
    # 根据 SPH_64 宏的定义，定义 sph_u64 或 sph_u32 类型的变量 count 或 count_high、count_low
#if SPH_64
    sph_u64 count;
#else
    sph_u32 count_high, count_low;
#endif
    # 定义 unsigned 类型的变量 z
    unsigned z;
    # 声明 STATE_BIG 宏

    # 将 sc->ptr 的值赋给 ptr
    ptr = sc->ptr;
    # 将 0x80 右移 n 位的结果赋给 z
    z = 0x80 >> n;
    # 将 ((ub & -z) | z) & 0xFF 的结果赋给 pad[0]
    pad[0] = ((ub & -z) | z) & 0xFF;
    # 如果 ptr 小于 120，则将 pad_len 的值设置为 128 - ptr，并根据 SPH_64 宏的定义更新 count 的值
    if (ptr < 120) {
        pad_len = 128 - ptr;
#if SPH_64
        count = SPH_T64(sc->count + 1);
#else
        count_low = SPH_T32(sc->count_low + 1);
        count_high = SPH_T32(sc->count_high);
        if (count_low == 0)
            count_high = SPH_T32(count_high + 1);
#endif
    } else {
        # 如果 ptr 大于等于 120，则将 pad_len 的值设置为 256 - ptr，并根据 SPH_64 宏的定义更新 count 的值
        pad_len = 256 - ptr;
#if SPH_64
        count = SPH_T64(sc->count + 2);
#else
        count_low = SPH_T32(sc->count_low + 2);
        count_high = SPH_T32(sc->count_high);
        if (count_low <= 1)
            count_high = SPH_T32(count_high + 1);
#endif
    }
    # 使用指定的值填充内存区域，从第二个元素开始，填充长度为 pad_len - 9 的区域
    memset(pad + 1, 0, pad_len - 9);
#if SPH_64
    # 如果定义了 SPH_64 宏，则执行以下代码块
    # 将 count 的值以大端字节序写入 pad + pad_len - 8 的位置
#else
    # 如果未定义 SPH_64 宏，则执行以下代码块
    # 将 count_high 的值以大端字节序写入 pad + pad_len - 8 的位置
    # 将 count_low 的值以大端字节序写入 pad + pad_len - 4 的位置
#endif
    # 调用 groestl_big_core 函数，对 sc、pad、pad_len 进行处理
    groestl_big_core(sc, pad, pad_len);
    # 读取 sc 的状态
    READ_STATE_BIG(sc);
    # 调用 FINAL_BIG 宏，对 sc 进行处理
    FINAL_BIG;
#if SPH_GROESTL_64
    # 如果定义了 SPH_GROESTL_64 宏，则执行以下代码块
    # 将 H[u + 8] 的值以大端字节序写入 pad + (u << 3) 的位置
#else
    # 如果未定义 SPH_GROESTL_64 宏，则执行以下代码块
    # 将 H[u + 16] 的值以大端字节序写入 pad + (u << 2) 的位置
#endif
    # 将 pad + 64 - out_len 的内容复制到 dst 的位置，长度为 out_len
    memcpy(dst, pad + 64 - out_len, out_len);
    # 调用 groestl_big_init 函数，对 sc 进行初始化，参数为 (unsigned)out_len << 3
    groestl_big_init(sc, (unsigned)out_len << 3);
}

/* see sph_groestl.h */
# sph_groestl.h 文件中的函数注释
void
sph_groestl224_init(void *cc)
{
    # 调用 groestl_small_init 函数，对 cc 进行初始化，参数为 224
    groestl_small_init(cc, 224);
}

/* see sph_groestl.h */
# sph_groestl.h 文件中的函数注释
void
sph_groestl224(void *cc, const void *data, size_t len)
{
    # 调用 groestl_small_core 函数，对 cc、data、len 进行处理
    groestl_small_core(cc, data, len);
}

/* see sph_groestl.h */
# sph_groestl.h 文件中的函数注释
void
sph_groestl224_close(void *cc, void *dst)
{
    # 调用 groestl_small_close 函数，对 cc 进行处理，参数为 0, 0, dst, 28
    groestl_small_close(cc, 0, 0, dst, 28);
}

/* see sph_groestl.h */
# sph_groestl.h 文件中的函数注释
void
sph_groestl224_addbits_and_close(void *cc, unsigned ub, unsigned n, void *dst)
{
    # 调用 groestl_small_close 函数，对 cc 进行处理，参数为 ub, n, dst, 28
    groestl_small_close(cc, ub, n, dst, 28);
}

/* see sph_groestl.h */
# sph_groestl.h 文件中的函数注释
void
sph_groestl256_init(void *cc)
{
    # 调用 groestl_small_init 函数，对 cc 进行初始化，参数为 256
    groestl_small_init(cc, 256);
}

/* see sph_groestl.h */
# sph_groestl.h 文件中的函数注释
void
sph_groestl256(void *cc, const void *data, size_t len)
{
    # 调用 groestl_small_core 函数，对 cc、data、len 进行处理
    groestl_small_core(cc, data, len);
}

/* see sph_groestl.h */
# sph_groestl.h 文件中的函数注释
void
sph_groestl256_close(void *cc, void *dst)
{
    # 调用 groestl_small_close 函数，对 cc 进行处理，参数为 0, 0, dst, 32
    groestl_small_close(cc, 0, 0, dst, 32);
}

/* see sph_groestl.h */
# sph_groestl.h 文件中的函数注释
void
sph_groestl256_addbits_and_close(void *cc, unsigned ub, unsigned n, void *dst)
{
    # 调用 groestl_small_close 函数，对 cc 进行处理，参数为 ub, n, dst, 32
    groestl_small_close(cc, ub, n, dst, 32);
}

/* see sph_groestl.h */
# sph_groestl.h 文件中的函数注释
void
sph_groestl384_init(void *cc)
{
    # 调用 groestl_big_init 函数，对 cc 进行初始化，参数为 384
    groestl_big_init(cc, 384);
}

/* see sph_groestl.h */
# sph_groestl.h 文件中的函数注释
void
sph_groestl384(void *cc, const void *data, size_t len)
{
    # 调用 groestl_big_core 函数，对 cc、data、len 进行处理
    groestl_big_core(cc, data, len);
}

/* see sph_groestl.h */
# sph_groestl.h 文件中的函数注释
void
sph_groestl384_close(void *cc, void *dst)
{
    # 调用 groestl_big_close 函数，对 cc 进行处理，参数为 0, 0, dst, 48
    groestl_big_close(cc, 0, 0, dst, 48);
}

/* see sph_groestl.h */
# sph_groestl.h 文件中的函数注释
void
sph_groestl384_addbits_and_close(void *cc, unsigned ub, unsigned n, void *dst)
{
    # 调用 groestl_big_close 函数，对 cc 进行处理，参数为 ub, n, dst, 48
    groestl_big_close(cc, ub, n, dst, 48);
}
    # 调用 groestl_big_close 函数，传入参数 cc, ub, n, dst, 48
    groestl_big_close(cc, ub, n, dst, 48);
/* see sph_groestl.h */
// 初始化 Groestl512 算法的上下文
void
sph_groestl512_init(void *cc)
{
    // 调用 groestl_big_init 函数，初始化 Groestl512 算法的上下文
    groestl_big_init(cc, 512);
}

/* see sph_groestl.h */
// 执行 Groestl512 算法的核心操作
void
sph_groestl512(void *cc, const void *data, size_t len)
{
    // 调用 groestl_big_core 函数，执行 Groestl512 算法的核心操作
    groestl_big_core(cc, data, len);
}

/* see sph_groestl.h */
// 关闭 Groestl512 算法的上下文，并输出结果
void
sph_groestl512_close(void *cc, void *dst)
{
    // 调用 groestl_big_close 函数，关闭 Groestl512 算法的上下文，并输出结果
    groestl_big_close(cc, 0, 0, dst, 64);
}

/* see sph_groestl.h */
// 添加额外的比特位到 Groestl512 算法的上下文中，并关闭上下文并输出结果
void
sph_groestl512_addbits_and_close(void *cc, unsigned ub, unsigned n, void *dst)
{
    // 调用 groestl_big_close 函数，添加额外的比特位到 Groestl512 算法的上下文中，并关闭上下文并输出结果
    groestl_big_close(cc, ub, n, dst, 64);
}

#ifdef __cplusplus
}

#endif
```