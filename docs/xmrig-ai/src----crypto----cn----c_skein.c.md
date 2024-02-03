# `xmrig\src\crypto\cn\c_skein.c`

```cpp
/***********************************************************************
**
** Implementation of the Skein hash function.
**
** Source code author: Doug Whiting, 2008.
**
** This algorithm and source code is released to the public domain.
**
************************************************************************/

#define  SKEIN_PORT_CODE /* instantiate any code in skein_port.h */

#include <stddef.h>                          /* get size_t definition */
#include <string.h>      /* get the memcpy/memset functions */
#include "c_skein.h"       /* get the Skein API definitions   */

#ifndef SKEIN_512_NIST_MAX_HASHBITS
#define SKEIN_512_NIST_MAX_HASHBITS (512)
#endif

#define  SKEIN_MODIFIER_WORDS  ( 2)          /* number of modifier (tweak) words */

#define  SKEIN_512_STATE_WORDS ( 8)
#define  SKEIN_MAX_STATE_WORDS (16)

#define  SKEIN_512_STATE_BYTES ( 8*SKEIN_512_STATE_WORDS)
#define  SKEIN_512_STATE_BITS  (64*SKEIN_512_STATE_WORDS)
#define  SKEIN_512_BLOCK_BYTES ( 8*SKEIN_512_STATE_WORDS)

#define SKEIN_RND_SPECIAL       (1000u)
#define SKEIN_RND_KEY_INITIAL   (SKEIN_RND_SPECIAL+0u)
#define SKEIN_RND_KEY_INJECT    (SKEIN_RND_SPECIAL+1u)
#define SKEIN_RND_FEED_FWD      (SKEIN_RND_SPECIAL+2u)

typedef struct
{
  size_t  hashBitLen;                      /* size of hash result, in bits */
  size_t  bCnt;                            /* current byte count in buffer b[] */
  u64b_t  T[SKEIN_MODIFIER_WORDS];         /* tweak words: T[0]=byte cnt, T[1]=flags */
} Skein_Ctxt_Hdr_t;

typedef struct                               /*  512-bit Skein hash context structure */
{
  Skein_Ctxt_Hdr_t h;                      /* common header context variables */
  u64b_t  X[SKEIN_512_STATE_WORDS];        /* chaining variables */
  u08b_t  b[SKEIN_512_BLOCK_BYTES];        /* partial block buffer (8-byte aligned) */
} Skein_512_Ctxt_t;

/*   Skein APIs for (incremental) "straight hashing" */
static int  Skein_512_Init  (Skein_512_Ctxt_t *ctx, size_t hashBitLen);
# 定义 Skein_512_Update 函数，用于更新 Skein_512_Ctxt_t 上下文和消息数据
static int  Skein_512_Update(Skein_512_Ctxt_t *ctx, const u08b_t *msg, size_t msgByteCnt);
# 定义 Skein_512_Final 函数，用于计算 Skein_512_Ctxt_t 上下文的最终哈希值
static int  Skein_512_Final (Skein_512_Ctxt_t *ctx, u08b_t * hashVal);

# 如果 SKEIN_TREE_HASH 未定义，则定义为 1
#ifndef SKEIN_TREE_HASH
#define SKEIN_TREE_HASH (1)
#endif

/*****************************************************************
** "Internal" Skein definitions
**    -- not needed for sequential hashing API, but will be
**           helpful for other uses of Skein (e.g., tree hash mode).
**    -- included here so that they can be shared between
**           reference and optimized code.
******************************************************************/

# 定义 tweak word T[1] 的位字段起始位置
# 由于它是第二个字，所以偏移量为 64
#define SKEIN_T1_BIT(BIT)       ((BIT) - 64)

# 定义 tweak word T[1] 中的各个位字段的位置
#define SKEIN_T1_POS_TREE_LVL   SKEIN_T1_BIT(112)       # 位 112..118: 哈希树的级别
#define SKEIN_T1_POS_BIT_PAD    SKEIN_T1_BIT(119)       # 位 119: 部分最终输入字节
#define SKEIN_T1_POS_BLK_TYPE   SKEIN_T1_BIT(120)       # 位 120..125: 类型字段
#define SKEIN_T1_POS_FIRST      SKEIN_T1_BIT(126)       # 位 126: 第一个块标志
#define SKEIN_T1_POS_FINAL      SKEIN_T1_BIT(127)       # 位 127: 最终块标志

# 定义 tweak word T[1] 的标志位
#define SKEIN_T1_FLAG_FIRST     (((u64b_t)  1 ) << SKEIN_T1_POS_FIRST)
#define SKEIN_T1_FLAG_FINAL     (((u64b_t)  1 ) << SKEIN_T1_POS_FINAL)
#define SKEIN_T1_FLAG_BIT_PAD   (((u64b_t)  1 ) << SKEIN_T1_POS_BIT_PAD)

# 定义 tweak word T[1] 的哈希树级别位字段掩码
#define SKEIN_T1_TREE_LVL_MASK  (((u64b_t)0x7F) << SKEIN_T1_POS_TREE_LVL)
#define SKEIN_T1_TREE_LEVEL(n)  (((u64b_t) (n)) << SKEIN_T1_POS_TREE_LVL)

# 定义 tweak word T[1] 的块类型字段
#define SKEIN_BLK_TYPE_KEY      ( 0)                    # 密钥，用于 MAC 和 KDF
#define SKEIN_BLK_TYPE_CFG      ( 4)                    # 配置块
# 定义 SKEIN 块类型的常量，用于标识不同类型的块
#define SKEIN_BLK_TYPE_PERS     ( 8)                    /* 个性化字符串 */
#define SKEIN_BLK_TYPE_PK       (12)                    /* 公钥（用于数字签名哈希） */
#define SKEIN_BLK_TYPE_KDF      (16)                    /* KDF 的密钥标识符 */
#define SKEIN_BLK_TYPE_NONCE    (20)                    /* PRNG 的随机数 */
#define SKEIN_BLK_TYPE_MSG      (48)                    /* 消息处理 */
#define SKEIN_BLK_TYPE_OUT      (63)                    /* 输出阶段 */
#define SKEIN_BLK_TYPE_MASK     (63)                    /* 位字段掩码 */

# 定义 SKEIN_T1_BLK_TYPE 宏，用于将块类型转换为对应的值
#define SKEIN_T1_BLK_TYPE(T)   (((u64b_t) (SKEIN_BLK_TYPE_##T)) << SKEIN_T1_POS_BLK_TYPE)
#define SKEIN_T1_BLK_TYPE_KEY   SKEIN_T1_BLK_TYPE(KEY)  /* 用于 MAC 和 KDF 的密钥 */
#define SKEIN_T1_BLK_TYPE_CFG   SKEIN_T1_BLK_TYPE(CFG)  /* 配置块 */
#define SKEIN_T1_BLK_TYPE_PERS  SKEIN_T1_BLK_TYPE(PERS) /* 个性化字符串 */
#define SKEIN_T1_BLK_TYPE_PK    SKEIN_T1_BLK_TYPE(PK)   /* 公钥（用于数字签名哈希） */
#define SKEIN_T1_BLK_TYPE_KDF   SKEIN_T1_BLK_TYPE(KDF)  /* KDF 的密钥标识符 */
#define SKEIN_T1_BLK_TYPE_NONCE SKEIN_T1_BLK_TYPE(NONCE)/* PRNG 的随机数 */
#define SKEIN_T1_BLK_TYPE_MSG   SKEIN_T1_BLK_TYPE(MSG)  /* 消息处理 */
#define SKEIN_T1_BLK_TYPE_OUT   SKEIN_T1_BLK_TYPE(OUT)  /* 输出阶段 */
#define SKEIN_T1_BLK_TYPE_MASK  SKEIN_T1_BLK_TYPE(MASK) /* 位字段掩码 */

# 定义 SKEIN_T1_BLK_TYPE_CFG_FINAL 和 SKEIN_T1_BLK_TYPE_OUT_FINAL 宏，用于表示最终的配置块和输出阶段
#define SKEIN_T1_BLK_TYPE_CFG_FINAL       (SKEIN_T1_BLK_TYPE_CFG | SKEIN_T1_FLAG_FINAL)
#define SKEIN_T1_BLK_TYPE_OUT_FINAL       (SKEIN_T1_BLK_TYPE_OUT | SKEIN_T1_FLAG_FINAL)

# 定义 SKEIN_VERSION 常量，表示 SKEIN 的版本号
#define SKEIN_VERSION           (1)

# 如果未定义 SKEIN_ID_STRING_LE，则定义为 0x33414853，即 "SHA3" 的小端序表示
#ifndef SKEIN_ID_STRING_LE      /* 允许在编译时个性化 */
#define SKEIN_ID_STRING_LE      (0x33414853)            /* "SHA3" (little-endian)*/
#endif

# 定义 SKEIN_MK_64 宏，用于将两个 32 位整数合并成一个 64 位整数
#define SKEIN_MK_64(hi32,lo32)  ((lo32) + (((u64b_t) (hi32)) << 32))
# 定义 SKEIN_SCHEMA_VER 常量，表示 SKEIN 的模式版本
#define SKEIN_SCHEMA_VER        SKEIN_MK_64(SKEIN_VERSION,SKEIN_ID_STRING_LE)
#define SKEIN_KS_PARITY         SKEIN_MK_64(0x1BD11BDA,0xA9FC1A22) 
// 定义 SKEIN_KS_PARITY 为 0x1BD11BDA,0xA9FC1A22 的64位整数

#define SKEIN_CFG_STR_LEN       (4*8) 
// 定义 SKEIN_CFG_STR_LEN 为 32

/* bit field definitions in config block treeInfo word */
// 在配置块 treeInfo 字中定义位字段

#define SKEIN_CFG_TREE_LEAF_SIZE_POS  ( 0) 
// 定义 SKEIN_CFG_TREE_LEAF_SIZE_POS 为 0

#define SKEIN_CFG_TREE_NODE_SIZE_POS  ( 8) 
// 定义 SKEIN_CFG_TREE_NODE_SIZE_POS 为 8

#define SKEIN_CFG_TREE_MAX_LEVEL_POS  (16) 
// 定义 SKEIN_CFG_TREE_MAX_LEVEL_POS 为 16

#define SKEIN_CFG_TREE_LEAF_SIZE_MSK  (((u64b_t) 0xFF) << SKEIN_CFG_TREE_LEAF_SIZE_POS) 
// 定义 SKEIN_CFG_TREE_LEAF_SIZE_MSK 为 (((u64b_t) 0xFF) 左移 SKEIN_CFG_TREE_LEAF_SIZE_POS 位

#define SKEIN_CFG_TREE_NODE_SIZE_MSK  (((u64b_t) 0xFF) << SKEIN_CFG_TREE_NODE_SIZE_POS) 
// 定义 SKEIN_CFG_TREE_NODE_SIZE_MSK 为 (((u64b_t) 0xFF) 左移 SKEIN_CFG_TREE_NODE_SIZE_POS 位

#define SKEIN_CFG_TREE_MAX_LEVEL_MSK  (((u64b_t) 0xFF) << SKEIN_CFG_TREE_MAX_LEVEL_POS) 
// 定义 SKEIN_CFG_TREE_MAX_LEVEL_MSK 为 (((u64b_t) 0xFF) 左移 SKEIN_CFG_TREE_MAX_LEVEL_POS 位

#define SKEIN_CFG_TREE_INFO(leaf,node,maxLvl)                   \
  ( (((u64b_t)(leaf  )) << SKEIN_CFG_TREE_LEAF_SIZE_POS) |    \
  (((u64b_t)(node  )) << SKEIN_CFG_TREE_NODE_SIZE_POS) |    \
  (((u64b_t)(maxLvl)) << SKEIN_CFG_TREE_MAX_LEVEL_POS) ) 
// 定义 SKEIN_CFG_TREE_INFO 宏，根据 leaf, node, maxLvl 构建配置信息

#define SKEIN_CFG_TREE_INFO_SEQUENTIAL SKEIN_CFG_TREE_INFO(0,0,0) /* use as treeInfo in InitExt() call for sequential processing */
// 定义 SKEIN_CFG_TREE_INFO_SEQUENTIAL 为 SKEIN_CFG_TREE_INFO(0,0,0)，用于顺序处理的 InitExt() 调用中作为 treeInfo 使用

/*
**   Skein macros for getting/setting tweak words, etc.
**   These are useful for partial input bytes, hash tree init/update, etc.
**/
// 获取/设置 tweak words 等的 Skein 宏。这些对于部分输入字节、哈希树初始化/更新等非常有用。

#define Skein_Get_Tweak(ctxPtr,TWK_NUM)         ((ctxPtr)->h.T[TWK_NUM]) 
// 获取指定上下文中的指定 Tweak 数

#define Skein_Set_Tweak(ctxPtr,TWK_NUM,tVal)    {(ctxPtr)->h.T[TWK_NUM] = (tVal);} 
// 设置指定上下文中的指定 Tweak 数为给定值

#define Skein_Get_T0(ctxPtr)    Skein_Get_Tweak(ctxPtr,0) 
// 获取指定上下文中的 T0 Tweak 数

#define Skein_Get_T1(ctxPtr)    Skein_Get_Tweak(ctxPtr,1) 
// 获取指定上下文中的 T1 Tweak 数

#define Skein_Set_T0(ctxPtr,T0) Skein_Set_Tweak(ctxPtr,0,T0) 
// 设置指定上下文中的 T0 Tweak 数为给定值

#define Skein_Set_T1(ctxPtr,T1) Skein_Set_Tweak(ctxPtr,1,T1) 
// 设置指定上下文中的 T1 Tweak 数为给定值

/* set both tweak words at once */
// 一次设置两个 Tweak 数

#define Skein_Set_T0_T1(ctxPtr,T0,T1)           \
{                                           \
  Skein_Set_T0(ctxPtr,(T0));                  \
  Skein_Set_T1(ctxPtr,(T1));                  \
}

#define Skein_Set_Type(ctxPtr,BLK_TYPE)         \
  Skein_Set_T1(ctxPtr,SKEIN_T1_BLK_TYPE_##BLK_TYPE) 
// 设置指定上下文中的 T1 Tweak 数为指定块类型的值

/* set up for starting with a new type: h.T[0]=0; h.T[1] = NEW_TYPE; h.bCnt=0; */
// 设置为以新类型开始：h.T[0]=0; h.T[1] = NEW_TYPE; h.bCnt=0;
#define Skein_Start_New_Type(ctxPtr,BLK_TYPE)   \
# 设置 Skein 上下文的 T0 和 T1 值，标记为第一个块类型
Skein_Set_T0_T1(ctxPtr,0,SKEIN_T1_FLAG_FIRST | SKEIN_T1_BLK_TYPE_##BLK_TYPE);
# 将上下文的块计数器重置为 0
(ctxPtr)->h.bCnt=0;

# 清除头部的第一个标志位
#define Skein_Clear_First_Flag(hdr)      { (hdr).T[1] &= ~SKEIN_T1_FLAG_FIRST;       }
# 设置头部的比特填充标志位
#define Skein_Set_Bit_Pad_Flag(hdr)      { (hdr).T[1] |=  SKEIN_T1_FLAG_BIT_PAD;     }
# 设置头部的树级别
#define Skein_Set_Tree_Level(hdr,height) { (hdr).T[1] |= SKEIN_T1_TREE_LEVEL(height);}

# 调试和错误检查的 Skein 内部定义
#define Skein_Show_Block(bits,ctx,X,blkPtr,wPtr,ksEvenPtr,ksOddPtr)
#define Skein_Show_Round(bits,ctx,r,X)
#define Skein_Show_R_Ptr(bits,ctx,r,X_ptr)
#define Skein_Show_Final(bits,ctx,cnt,outPtr)
#define Skein_Show_Key(bits,ctx,key,keyBytes)

# 如果未定义运行时检查，则默认忽略所有断言
#ifndef SKEIN_ERR_CHECK
#define Skein_Assert(x,retCode)
#define Skein_assert(x)
# 如果定义了 SKEIN_ASSERT，则包含 assert.h 头文件，并使用 assert 函数进行断言
#elif   defined(SKEIN_ASSERT)
#include <assert.h>
#define Skein_Assert(x,retCode) assert(x)
#define Skein_assert(x)         assert(x)
# 否则，包含 assert.h 头文件，并根据条件进行断言
#else
#include <assert.h>
# 如果条件不满足，则返回指定的错误码
#define Skein_Assert(x,retCode) { if (!(x)) return retCode; } /*  caller  error */
# 否则，使用 assert 函数进行断言
#define Skein_assert(x)         assert(x)                     /* internal error */
#endif

# Skein 块函数的常量定义（在 Ref 和 Opt 代码中共享）
enum
{
  /* Skein_512 round rotation constants */
  // 定义 Skein_512 轮常量
  R_512_0_0=46, R_512_0_1=36, R_512_0_2=19, R_512_0_3=37,
  R_512_1_0=33, R_512_1_1=27, R_512_1_2=14, R_512_1_3=42,
  R_512_2_0=17, R_512_2_1=49, R_512_2_2=36, R_512_2_3=39,
  R_512_3_0=44, R_512_3_1= 9, R_512_3_2=54, R_512_3_3=56,
  R_512_4_0=39, R_512_4_1=30, R_512_4_2=34, R_512_4_3=24,
  R_512_5_0=13, R_512_5_1=50, R_512_5_2=10, R_512_5_3=17,
  R_512_6_0=25, R_512_6_1=29, R_512_6_2=39, R_512_6_3=43,
  R_512_7_0= 8, R_512_7_1=35, R_512_7_2=56, R_512_7_3=22,
};

#ifndef SKEIN_ROUNDS
#define SKEIN_512_ROUNDS_TOTAL (72)
#else                                        /* allow command-line define in range 8*(5..14)   */
#define SKEIN_512_ROUNDS_TOTAL (8*((((SKEIN_ROUNDS/ 10) + 5) % 10) + 5))
#endif

/*
***************** Pre-computed Skein IVs *******************
**
** NOTE: these values are not "magic" constants, but
** are generated using the Threefish block function.
** They are pre-computed here only for speed; i.e., to
** avoid the need for a Threefish call during Init().
**
** The IV for any fixed hash length may be pre-computed.
** Only the most common values are included here.
**
************************************************************
**/

#define MK_64 SKEIN_MK_64

/* blkSize =  512 bits. hashSize =  256 bits */
// 定义 512 位块大小和 256 位哈希大小
const u64b_t SKEIN_512_IV_256[] =
    {
    MK_64(0xCCD044A1,0x2FDB3E13),
    MK_64(0xE8359030,0x1A79A9EB),
    MK_64(0x55AEA061,0x4F816E6F),
    MK_64(0x2A2767A4,0xAE9B94DB),
    MK_64(0xEC06025E,0x74DD7683),
    MK_64(0xE7A436CD,0xC4746251),
    MK_64(0xC36FBAF9,0x393AD185),
    MK_64(0x3EEDBA18,0x33EDFC13)
    };

#ifndef SKEIN_USE_ASM
#define SKEIN_USE_ASM   (0)                     /* default is all C code (no ASM) */
#endif

#ifndef SKEIN_LOOP
#define SKEIN_LOOP 001                          /* default: unroll 256 and 512, but not 1024 */
#endif

#define BLK_BITS        (WCNT*64)               /* some useful definitions for code here */
#define KW_TWK_BASE     (0)
#define KW_KEY_BASE     (3)
#define ks              (kw + KW_KEY_BASE)  // 定义 ks 为 kw 加上 KW_KEY_BASE
#define ts              (kw + KW_TWK_BASE)  // 定义 ts 为 kw 加上 KW_TWK_BASE

#ifdef SKEIN_DEBUG
#define DebugSaveTweak(ctx) { ctx->h.T[0] = ts[0]; ctx->h.T[1] = ts[1]; }  // 如果定义了 SKEIN_DEBUG，则定义 DebugSaveTweak 宏，将 ts 的值保存到 ctx->h.T[0] 和 ctx->h.T[1] 中
#else
#define DebugSaveTweak(ctx)  // 如果未定义 SKEIN_DEBUG，则空宏
#endif

/*****************************  Skein_512 ******************************/
#if !(SKEIN_USE_ASM & 512)
static void Skein_512_Process_Block(Skein_512_Ctxt_t *ctx,const u08b_t *blkPtr,size_t blkCnt,size_t byteCntAdd)
    { /* do it in C */
    enum
        {
        WCNT = SKEIN_512_STATE_WORDS  // 定义 WCNT 为 SKEIN_512_STATE_WORDS
        };
#undef  RCNT
#define RCNT  (SKEIN_512_ROUNDS_TOTAL/8)  // 定义 RCNT 为 SKEIN_512_ROUNDS_TOTAL 除以 8

#ifdef  SKEIN_LOOP                              /* configure how much to unroll the loop */
#define SKEIN_UNROLL_512 (((SKEIN_LOOP)/10)%10)  // 根据 SKEIN_LOOP 定义 SKEIN_UNROLL_512
#else
#define SKEIN_UNROLL_512 (0)  // 如果未定义 SKEIN_LOOP，则将 SKEIN_UNROLL_512 定义为 0
#endif

#if SKEIN_UNROLL_512
#if (RCNT % SKEIN_UNROLL_512)
#error "Invalid SKEIN_UNROLL_512"               /* sanity check on unroll count */
#endif
    size_t  r;
    u64b_t  kw[WCNT+4+RCNT*2];                  /* key schedule words : chaining vars + tweak + "rotation"*/  // 定义 kw 数组
#else
    u64b_t  kw[WCNT+4];                         /* key schedule words : chaining vars + tweak */  // 定义 kw 数组
#endif
    u64b_t  X0,X1,X2,X3,X4,X5,X6,X7;            /* local copy of vars, for speed */  // 定义本地变量 X0 到 X7
    u64b_t  w [WCNT];                           /* local copy of input block */  // 定义本地变量 w
#ifdef SKEIN_DEBUG
    const u64b_t *Xptr[8];                      /* use for debugging (help compiler put Xn in registers) */  // 定义用于调试的 Xptr 数组
    Xptr[0] = &X0;  Xptr[1] = &X1;  Xptr[2] = &X2;  Xptr[3] = &X3;
    Xptr[4] = &X4;  Xptr[5] = &X5;  Xptr[6] = &X6;  Xptr[7] = &X7;
#endif

    Skein_assert(blkCnt != 0);                  /* never call with blkCnt == 0! */  // 断言 blkCnt 不等于 0
    ts[0] = ctx->h.T[0];  // 将 ctx->h.T[0] 的值赋给 ts[0]
    ts[1] = ctx->h.T[1];  // 将 ctx->h.T[1] 的值赋给 ts[1]
    do  {
        /* 这个实现仅支持 2**64 字节的输入（这里没有进位） */
        ts[0] += byteCntAdd;                    /* 更新处理长度 */

        /* 预先计算此块的密钥调度 */
        ks[0] = ctx->X[0];
        ks[1] = ctx->X[1];
        ks[2] = ctx->X[2];
        ks[3] = ctx->X[3];
        ks[4] = ctx->X[4];
        ks[5] = ctx->X[5];
        ks[6] = ctx->X[6];
        ks[7] = ctx->X[7];
        ks[8] = ks[0] ^ ks[1] ^ ks[2] ^ ks[3] ^
                ks[4] ^ ks[5] ^ ks[6] ^ ks[7] ^ SKEIN_KS_PARITY;

        ts[2] = ts[0] ^ ts[1];

        Skein_Get64_LSB_First(w,blkPtr,WCNT); /* 以小端格式获取输入块 */
        DebugSaveTweak(ctx);
        Skein_Show_Block(BLK_BITS,&ctx->h,ctx->X,blkPtr,w,ks,ts);

        X0   = w[0] + ks[0];                    /* 进行第一次完整的密钥注入 */
        X1   = w[1] + ks[1];
        X2   = w[2] + ks[2];
        X3   = w[3] + ks[3];
        X4   = w[4] + ks[4];
        X5   = w[5] + ks[5] + ts[0];
        X6   = w[6] + ks[6] + ts[1];
        X7   = w[7] + ks[7];

        blkPtr += SKEIN_512_BLOCK_BYTES;

        Skein_Show_R_Ptr(BLK_BITS,&ctx->h,SKEIN_RND_KEY_INITIAL,Xptr);
        /* 运行轮次 */
#define Round512(p0,p1,p2,p3,p4,p5,p6,p7,ROT,rNum)                  \  # 定义一个名为Round512的宏，用于执行一系列操作
    X##p0 += X##p1; X##p1 = RotL_64(X##p1,ROT##_0); X##p1 ^= X##p0; \  # 对X0和X1执行一系列操作
    X##p2 += X##p3; X##p3 = RotL_64(X##p3,ROT##_1); X##p3 ^= X##p2; \  # 对X2和X3执行一系列操作
    X##p4 += X##p5; X##p5 = RotL_64(X##p5,ROT##_2); X##p5 ^= X##p4; \  # 对X4和X5执行一系列操作
    X##p6 += X##p7; X##p7 = RotL_64(X##p7,ROT##_3); X##p7 ^= X##p6; \  # 对X6和X7执行一系列操作

#if SKEIN_UNROLL_512 == 0
#define R512(p0,p1,p2,p3,p4,p5,p6,p7,ROT,rNum)      /* unrolled */  \  # 定义一个名为R512的宏，用于执行一系列操作
    Round512(p0,p1,p2,p3,p4,p5,p6,p7,ROT,rNum)                      \  # 调用Round512宏
    Skein_Show_R_Ptr(BLK_BITS,&ctx->h,rNum,Xptr);                    # 调用Skein_Show_R_Ptr函数

#define I512(R)                                                     \  # 定义一个名为I512的宏，用于执行一系列操作
    X0   += ks[((R)+1) % 9];   /* inject the key schedule value */  \  # 对X0执行一系列操作
    X1   += ks[((R)+2) % 9];                                        \  # 对X1执行一系列操作
    X2   += ks[((R)+3) % 9];                                        \  # 对X2执行一系列操作
    X3   += ks[((R)+4) % 9];                                        \  # 对X3执行一系列操作
    X4   += ks[((R)+5) % 9];                                        \  # 对X4执行一系列操作
    X5   += ks[((R)+6) % 9] + ts[((R)+1) % 3];                      \  # 对X5执行一系列操作
    X6   += ks[((R)+7) % 9] + ts[((R)+2) % 3];                      \  # 对X6执行一系列操作
    X7   += ks[((R)+8) % 9] +     (R)+1;                            \  # 对X7执行一系列操作
    Skein_Show_R_Ptr(BLK_BITS,&ctx->h,SKEIN_RND_KEY_INJECT,Xptr);   # 调用Skein_Show_R_Ptr函数
#else                                       /* looping version */
#define R512(p0,p1,p2,p3,p4,p5,p6,p7,ROT,rNum)                      \  # 定义一个名为R512的宏，用于执行一系列操作
    Round512(p0,p1,p2,p3,p4,p5,p6,p7,ROT,rNum)                      \  # 调用Round512宏
    Skein_Show_R_Ptr(BLK_BITS,&ctx->h,4*(r-1)+rNum,Xptr);           # 调用Skein_Show_R_Ptr函数

#define I512(R)                                                     \  # 定义一个名为I512的宏，用于执行一系列操作
    X0   += ks[r+(R)+0];        /* inject the key schedule value */ \  # 对X0执行一系列操作
    X1   += ks[r+(R)+1];                                            \  # 对X1执行一系列操作
    X2   += ks[r+(R)+2];                                            \  # 对X2执行一系列操作
    X3   += ks[r+(R)+3];                                            \  # 对X3执行一系列操作
    X4   += ks[r+(R)+4];                                            \  # 对X4执行一系列操作
    X5   += ks[r+(R)+5] + ts[r+(R)+0];                              \  # 更新X5的值
    X6   += ks[r+(R)+6] + ts[r+(R)+1];                              \  # 更新X6的值
    X7   += ks[r+(R)+7] +    r+(R)   ;                              \  # 更新X7的值
    ks[r +       (R)+8] = ks[r+(R)-1];  /* rotate key schedule */   \  # 旋转密钥计划
    ts[r +       (R)+2] = ts[r+(R)-1];                              \  # 更新ts数组的值
    Skein_Show_R_Ptr(BLK_BITS,&ctx->h,SKEIN_RND_KEY_INJECT,Xptr);   # 显示调试信息

    for (r=1;r < 2*RCNT;r+=2*SKEIN_UNROLL_512)   /* loop thru it */  # 循环迭代
#endif                         /* end of looped code definitions */
        {
#define R512_8_rounds(R)  /* do 8 full rounds */  \  # 定义一个宏，用于执行8轮完整的加密操作
        R512(0,1,2,3,4,5,6,7,R_512_0,8*(R)+ 1);   \  # 调用R512函数进行加密操作
        R512(2,1,4,7,6,5,0,3,R_512_1,8*(R)+ 2);   \  # 调用R512函数进行加密操作
        R512(4,1,6,3,0,5,2,7,R_512_2,8*(R)+ 3);   \  # 调用R512函数进行加密操作
        R512(6,1,0,7,2,5,4,3,R_512_3,8*(R)+ 4);   \  # 调用R512函数进行加密操作
        I512(2*(R));                              \  # 调用I512函数进行加密操作
        R512(0,1,2,3,4,5,6,7,R_512_4,8*(R)+ 5);   \  # 调用R512函数进行加密操作
        R512(2,1,4,7,6,5,0,3,R_512_5,8*(R)+ 6);   \  # 调用R512函数进行加密操作
        R512(4,1,6,3,0,5,2,7,R_512_6,8*(R)+ 7);   \  # 调用R512函数进行加密操作
        R512(6,1,0,7,2,5,4,3,R_512_7,8*(R)+ 8);   \  # 调用R512函数进行加密操作
        I512(2*(R)+1);        /* and key injection */  \  # 调用I512函数进行密钥注入

        R512_8_rounds( 0);  # 调用宏定义的R512_8_rounds函数进行8轮加密操作

#define R512_Unroll_R(NN) ((SKEIN_UNROLL_512 == 0 && SKEIN_512_ROUNDS_TOTAL/8 > (NN)) || (SKEIN_UNROLL_512 > (NN)))  # 定义一个宏，用于判断是否需要展开循环

  #if   R512_Unroll_R( 1)  # 如果需要展开循环1次
        R512_8_rounds( 1);  # 调用宏定义的R512_8_rounds函数进行8轮加密操作
  #endif
  #if   R512_Unroll_R( 2)  # 如果需要展开循环2次
        R512_8_rounds( 2);  # 调用宏定义的R512_8_rounds函数进行8轮加密操作
  #endif
  #if   R512_Unroll_R( 3)  # 如果需要展开循环3次
        R512_8_rounds( 3);  # 调用宏定义的R512_8_rounds函数进行8轮加密操作
  #endif
  #if   R512_Unroll_R( 4)  # 如果需要展开循环4次
        R512_8_rounds( 4);  # 调用宏定义的R512_8_rounds函数进行8轮加密操作
  #endif
  #if   R512_Unroll_R( 5)  # 如果需要展开循环5次
        R512_8_rounds( 5);  # 调用宏定义的R512_8_rounds函数进行8轮加密操作
  #endif
  #if   R512_Unroll_R( 6)  # 如果需要展开循环6次
        R512_8_rounds( 6);  # 调用宏定义的R512_8_rounds函数进行8轮加密操作
  #endif
  #if   R512_Unroll_R( 7)  # 如果需要展开循环7次
        R512_8_rounds( 7);  # 调用宏定义的R512_8_rounds函数进行8轮加密操作
  #endif
  #if   R512_Unroll_R( 8)  # 如果需要展开循环8次
        R512_8_rounds( 8);  # 调用宏定义的R512_8_rounds函数进行8轮加密操作
  #endif
  #if   R512_Unroll_R( 9)  # 如果需要展开循环9次
        R512_8_rounds( 9);  # 调用宏定义的R512_8_rounds函数进行8轮加密操作
  #endif
  #if   R512_Unroll_R(10)  # 如果需要展开循环10次
        R512_8_rounds(10);  # 调用宏定义的R512_8_rounds函数进行8轮加密操作
  #endif
  #if   R512_Unroll_R(11)  # 如果需要展开循环11次
        R512_8_rounds(11);  # 调用宏定义的R512_8_rounds函数进行8轮加密操作
  #endif
  #if   R512_Unroll_R(12)  # 如果需要展开循环12次
        R512_8_rounds(12);  # 调用宏定义的R512_8_rounds函数进行8轮加密操作
  #endif
  #if   R512_Unroll_R(13)  # 如果需要展开循环13次
        R512_8_rounds(13);  # 调用宏定义的R512_8_rounds函数进行8轮加密操作
  #endif
  #if   R512_Unroll_R(14)  # 如果需要展开循环14次
        R512_8_rounds(14);  # 调用宏定义的R512_8_rounds函数进行8轮加密操作
  #endif
  #if  (SKEIN_UNROLL_512 > 14)  # 如果需要展开循环大于14次
#error  "need more unrolling in Skein_512_Process_Block"
  #endif
        }

        /* do the final "feedforward" xor, update context chaining vars */
        // 执行最终的“前馈”异或操作，更新上下文链变量
        ctx->X[0] = X0 ^ w[0];
        ctx->X[1] = X1 ^ w[1];
        ctx->X[2] = X2 ^ w[2];
        ctx->X[3] = X3 ^ w[3];
        ctx->X[4] = X4 ^ w[4];
        ctx->X[5] = X5 ^ w[5];
        ctx->X[6] = X6 ^ w[6];
        ctx->X[7] = X7 ^ w[7];
        Skein_Show_Round(BLK_BITS,&ctx->h,SKEIN_RND_FEED_FWD,ctx->X);

        ts[1] &= ~SKEIN_T1_FLAG_FIRST;
        }
    while (--blkCnt);
    ctx->h.T[0] = ts[0];
    ctx->h.T[1] = ts[1];
    }
#endif

/*****************************************************************/
/*     512-bit Skein                                             */
/*****************************************************************/

/*++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*/
/* init the context for a straight hashing operation  */
// 为直接哈希操作初始化上下文
static int Skein_512_Init(Skein_512_Ctxt_t *ctx, size_t hashBitLen)
    {
    union
        {
        u08b_t  b[SKEIN_512_STATE_BYTES];
        u64b_t  w[SKEIN_512_STATE_WORDS];
        } cfg;                              /* config block */

    Skein_Assert(hashBitLen > 0,SKEIN_BAD_HASHLEN);
    ctx->h.hashBitLen = hashBitLen;         /* output hash bit count */

    switch (hashBitLen)
        {             /* use pre-computed values, where available */
#ifndef SKEIN_NO_PRECOMP
        case  256: memcpy(ctx->X,SKEIN_512_IV_256,sizeof(ctx->X));  break;
#endif
        default:
            /* 如果没有预先计算的 IV 值，则执行此处代码 */
            /* 构建/处理配置块，类型为 CONFIG（可能是预先计算的） */
            Skein_Start_New_Type(ctx,CFG_FINAL);        /* 设置 tweaks: T0=0; T1=CFG | FINAL */

            cfg.w[0] = Skein_Swap64(SKEIN_SCHEMA_VER);  /* 设置模式和版本 */
            cfg.w[1] = Skein_Swap64(hashBitLen);        /* 哈希结果长度（以比特为单位） */
            cfg.w[2] = Skein_Swap64(SKEIN_CFG_TREE_INFO_SEQUENTIAL);
            memset(&cfg.w[3],0,sizeof(cfg) - 3*sizeof(cfg.w[0])); /* 用零填充配置块 */

            /* 从配置块计算初始链变量 */
            memset(ctx->X,0,sizeof(ctx->X));            /* 将链变量清零 */
            Skein_512_Process_Block(ctx,cfg.b,1,SKEIN_CFG_STR_LEN);
            break;
        }

    /* 现在链变量 ctx->X 已经根据给定的 hashBitLen 初始化。 */
    /* 设置为处理哈希的数据消息部分（默认） */
    Skein_Start_New_Type(ctx,MSG);              /* T0=0, T1= MSG 类型 */

    return SKEIN_SUCCESS;
    }

/*++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*/
/* 处理输入字节 */
static int Skein_512_Update(Skein_512_Ctxt_t *ctx, const u08b_t *msg, size_t msgByteCnt)
    {
    size_t n;

    Skein_Assert(ctx->h.bCnt <= SKEIN_512_BLOCK_BYTES,SKEIN_FAIL);    /* 捕获未初始化的上下文 */

    /* 处理完整的数据块，如果有的话 */
    if (msgByteCnt + ctx->h.bCnt > SKEIN_512_BLOCK_BYTES)
        {
        // 如果消息字节数加上上下文中的缓冲区字节数大于块字节数，则执行以下操作
        if (ctx->h.bCnt)                              /* finish up any buffered message data */
            {
            // 如果上下文中的缓冲区字节数不为零，则执行以下操作，完成任何缓冲的消息数据
            n = SKEIN_512_BLOCK_BYTES - ctx->h.bCnt;  /* # bytes free in buffer b[] */
            // 计算缓冲区 b[] 中的剩余字节数
            if (n)
                {
                // 如果剩余字节数不为零，则执行以下操作
                Skein_assert(n < msgByteCnt);         /* check on our logic here */
                // 断言逻辑正确性
                memcpy(&ctx->b[ctx->h.bCnt],msg,n);
                // 将消息数据复制到缓冲区 b[] 中
                msgByteCnt  -= n;
                // 更新消息字节数
                msg         += n;
                // 更新消息指针
                ctx->h.bCnt += n;
                // 更新上下文中的缓冲区字节数
                }
            Skein_assert(ctx->h.bCnt == SKEIN_512_BLOCK_BYTES);
            // 断言上下文中的缓冲区字节数等于块字节数
            Skein_512_Process_Block(ctx,ctx->b,1,SKEIN_512_BLOCK_BYTES);
            // 处理块数据
            ctx->h.bCnt = 0;
            // 重置上下文中的缓冲区字节数
            }
        /* now process any remaining full blocks, directly from input message data */
        // 现在处理任何剩余的完整块，直接从输入消息数据中处理
        if (msgByteCnt > SKEIN_512_BLOCK_BYTES)
            {
            // 如果消息字节数大于块字节数，则执行以下操作
            n = (msgByteCnt-1) / SKEIN_512_BLOCK_BYTES;   /* number of full blocks to process */
            // 计算要处理的完整块的数量
            Skein_512_Process_Block(ctx,msg,n,SKEIN_512_BLOCK_BYTES);
            // 处理块数据
            msgByteCnt -= n * SKEIN_512_BLOCK_BYTES;
            // 更新消息字节数
            msg        += n * SKEIN_512_BLOCK_BYTES;
            // 更新消息指针
            }
        Skein_assert(ctx->h.bCnt == 0);
        // 断言上下文中的缓冲区字节数为零
        }

    /* copy any remaining source message data bytes into b[] */
    // 将任何剩余的源消息数据字节复制到 b[] 中
    if (msgByteCnt)
        {
        // 如果消息字节数不为零，则执行以下操作
        Skein_assert(msgByteCnt + ctx->h.bCnt <= SKEIN_512_BLOCK_BYTES);
        // 断言消息字节数加上上下文中的缓冲区字节数小于等于块字节数
        memcpy(&ctx->b[ctx->h.bCnt],msg,msgByteCnt);
        // 将消息数据复制到缓冲区 b[] 中
        ctx->h.bCnt += msgByteCnt;
        // 更新上下文中的缓冲区字节数
        }

    return SKEIN_SUCCESS;
    }
/*++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*/
/* finalize the hash computation and output the result */
static int Skein_512_Final(Skein_512_Ctxt_t *ctx, u08b_t *hashVal)
    {
    size_t i,n,byteCnt;  /* 声明变量 i, n, byteCnt 用于循环和计算字节数 */
    u64b_t X[SKEIN_512_STATE_WORDS];  /* 声明数组 X 用于存储状态 */

    Skein_Assert(ctx->h.bCnt <= SKEIN_512_BLOCK_BYTES,SKEIN_FAIL);    /* 检查上下文是否已初始化 */

    ctx->h.T[1] |= SKEIN_T1_FLAG_FINAL;                 /* 将标记设置为最终块 */
    if (ctx->h.bCnt < SKEIN_512_BLOCK_BYTES)            /* 如果需要，对 b[] 进行零填充 */
        memset(&ctx->b[ctx->h.bCnt],0,SKEIN_512_BLOCK_BYTES - ctx->h.bCnt);

    Skein_512_Process_Block(ctx,ctx->b,1,ctx->h.bCnt);  /* 处理最终块 */

    /* 现在输出结果 */
    byteCnt = (ctx->h.hashBitLen + 7) >> 3;             /* 输出字节的总数 */

    /* 在“计数器模式”下运行 Threefish 以生成输出 */
    memset(ctx->b,0,sizeof(ctx->b));  /* 将 b[] 清零，以便它可以保存计数器 */
    memcpy(X,ctx->X,sizeof(X));       /* 保留计数器模式“密钥”的本地副本 */
    for (i=0;i*SKEIN_512_BLOCK_BYTES < byteCnt;i++)
        {
        ((u64b_t *)ctx->b)[0]= Skein_Swap64((u64b_t) i); /* 构建计数器块 */
        Skein_Start_New_Type(ctx,OUT_FINAL);
        Skein_512_Process_Block(ctx,ctx->b,1,sizeof(u64b_t)); /* 运行“计数器模式” */
        n = byteCnt - i*SKEIN_512_BLOCK_BYTES;   /* 剩余要输出的字节数 */
        if (n >= SKEIN_512_BLOCK_BYTES)
            n  = SKEIN_512_BLOCK_BYTES;
        Skein_Put64_LSB_First(hashVal+i*SKEIN_512_BLOCK_BYTES,ctx->X,n);   /* “输出”计数器模式字节 */
        Skein_Show_Final(512,&ctx->h,n,hashVal+i*SKEIN_512_BLOCK_BYTES);
        memcpy(ctx->X,X,sizeof(X));   /* 为下一次恢复计数器模式密钥 */
        }
    return SKEIN_SUCCESS;
    }

#if defined(SKEIN_CODE_SIZE) || defined(SKEIN_PERF)
static size_t Skein_512_API_CodeSize(void)
    {
    # 返回Skein_512_API_CodeSize的地址减去Skein_512_Init的地址，结果为指针之间的偏移量
    return ((u08b_t *) Skein_512_API_CodeSize) -
           ((u08b_t *) Skein_512_Init);
    }
#endif

typedef struct
{
  uint_t  statebits;                      /* 256, 512, or 1024 */  // 定义状态位数，可以是256、512或1024
  union
  {
    Skein_Ctxt_Hdr_t h;                 /* common header "overlay" */
    Skein_512_Ctxt_t ctx_512;
  } u;
}
hashState;

/* "incremental" hashing API */
static SkeinHashReturn Init  (hashState *state, int hashbitlen);  // 初始化哈希状态
static SkeinHashReturn Update(hashState *state, const SkeinBitSequence *data, SkeinDataLength databitlen);  // 更新哈希状态
static SkeinHashReturn Final (hashState *state,       SkeinBitSequence *hashval);  // 完成哈希计算并返回哈希值

/*++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*/
/* select the context size and init the context */
static SkeinHashReturn Init(hashState *state, int hashbitlen)  // 选择上下文大小并初始化上下文
{
    state->statebits = 64*SKEIN_512_STATE_WORDS;  // 设置状态位数
    return Skein_512_Init(&state->u.ctx_512,(size_t) hashbitlen);  // 调用Skein_512_Init函数初始化上下文
}

/*++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*/
/* process data to be hashed */
static SkeinHashReturn Update(hashState *state, const SkeinBitSequence *data, SkeinDataLength databitlen)  // 处理要进行哈希的数据
{
  /* only the final Update() call is allowed do partial bytes, else assert an error */
  Skein_Assert((state->u.h.T[1] & SKEIN_T1_FLAG_BIT_PAD) == 0 || databitlen == 0, SKEIN_FAIL);  // 断言，确保只有最后一次调用Update()可以处理部分字节，否则报错

  Skein_Assert(state->statebits % 256 == 0 && (state->statebits-256) < 1024,SKEIN_FAIL);  // 断言，确保状态位数是256的倍数且小于1024

  if ((databitlen & 7) == 0)  /* partial bytes? */  // 判断是否有部分字节
  {
    return Skein_512_Update(&state->u.ctx_512,data,databitlen >> 3);  // 调用Skein_512_Update函数处理数据
  }
  else
  {   /* handle partial final byte */
    size_t bCnt = (databitlen >> 3) + 1;                  /* number of bytes to handle (nonzero here!) */  // 计算要处理的字节数
    u08b_t b,mask;

    mask = (u08b_t) (1u << (7 - (databitlen & 7)));       /* partial byte bit mask */  // 计算部分字节的位掩码
    b    = (u08b_t) ((data[bCnt-1] & (0-mask)) | mask);   /* apply bit padding on final byte */  // 对最后一个字节应用位填充

    Skein_512_Update(&state->u.ctx_512,data,bCnt-1); /* process all but the final byte    */  // 处理除最后一个字节之外的所有字节
    Skein_512_Update(&state->u.ctx_512,&b  ,  1   ); /* process the (masked) partial byte */  // 处理（掩码）部分字节
    # 设置状态对象的哈希值为 1，用于最后一次调用
    Skein_Set_Bit_Pad_Flag(state->u.h);                    /* set tweak flag for the final call */
    # 返回 SKEIN_SUCCESS，表示函数执行成功
    return SKEIN_SUCCESS;
  }
/*++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*/
/* finalize hash computation and output the result (hashbitlen bits) */
static SkeinHashReturn Final(hashState *state, SkeinBitSequence *hashval)
{
  // 确保状态位数是 256 的倍数且小于 1024，否则返回失败
  Skein_Assert(state->statebits % 256 == 0 && (state->statebits-256) < 1024,FAIL);
  // 调用 Skein_512_Final 完成哈希计算，并输出结果
  return Skein_512_Final(&state->u.ctx_512,hashval);
}

/*++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*/
/* all-in-one hash function */
SkeinHashReturn skein_hash(int hashbitlen, const SkeinBitSequence *data, /* all-in-one call */
                SkeinDataLength databitlen,SkeinBitSequence *hashval)
{
  // 初始化哈希状态
  hashState  state;
  // 调用 Init 函数初始化哈希状态，并返回结果
  SkeinHashReturn r = Init(&state,hashbitlen);
  if (r == SKEIN_SUCCESS)
  { /* these calls do not fail when called properly */
    // 如果初始化成功，则调用 Update 函数更新哈希状态
    r = Update(&state,data,databitlen);
    // 调用 Final 函数完成哈希计算
    Final(&state,hashval);
  }
  // 返回哈希计算结果
  return r;
}

void xmr_skein(const SkeinBitSequence *data, SkeinBitSequence *hashval){
  #define XMR_HASHBITLEN 256
  #define XMR_DATABITLEN 1600

  // 初始化哈希状态
  hashState  state;
  state.statebits = 64*SKEIN_512_STATE_WORDS;

  // 设置哈希位长度并复制初始向量
  state.u.ctx_512.h.hashBitLen = XMR_HASHBITLEN;
  memcpy(state.u.ctx_512.X,SKEIN_512_IV_256,sizeof(state.u.ctx_512.X));
  Skein_512_Ctxt_t* ctx = &(state.u.ctx_512);
  // 开始新的哈希类型
  Skein_Start_New_Type(ctx,MSG);

  // 更新哈希状态
  if ((XMR_DATABITLEN & 7) == 0){  /* partial bytes? */
    // 如果数据位长度是 8 的倍数，则调用 Skein_512_Update 更新哈希状态
    Skein_512_Update(&state.u.ctx_512,data,XMR_DATABITLEN >> 3);
  }else{   /* handle partial final byte */
    // 处理部分字节的情况
    size_t bCnt = (XMR_DATABITLEN >> 3) + 1;                  /* number of bytes to handle (nonzero here!) */
    u08b_t b,mask;

    mask = (u08b_t) (1u << (7 - (XMR_DATABITLEN & 7)));       /* partial byte bit mask */
    b    = (u08b_t) ((data[bCnt-1] & (0-mask)) | mask);   /* apply bit padding on final byte */

    // 处理除最后一个字节外的所有字节
    Skein_512_Update(&state.u.ctx_512,data,bCnt-1); /* process all but the final byte    */
    # 使用Skein_512_Update函数处理（掩码）部分字节
    Skein_512_Update(&state.u.ctx_512,&b  ,  1   ); 
    # 设置tweak标志以进行最终调用
    Skein_Set_Bit_Pad_Flag(state.u.h);                    
  }

  // Finalize
  # 使用Skein_512_Final函数完成哈希计算并将结果存储在hashval中
  Skein_512_Final(&state.u.ctx_512, hashval);
# 闭合前面的函数定义
```