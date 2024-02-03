# `xmrig\src\crypto\randomx\blake2\avx2\blake2b-load-avx2.h`

```cpp
#ifndef BLAKE2_AVX2_BLAKE2B_LOAD_AVX2_H
#define BLAKE2_AVX2_BLAKE2B_LOAD_AVX2_H

#define BLAKE2B_LOAD_MSG_0_1(b0) do { \  # 定义宏，将m0和m1中的数据按照特定规则组合到b0中
  t0 = _mm256_unpacklo_epi64(m0, m1); \  # 使用AVX2指令将m0和m1中的数据按照特定规则解包到t0中
  t1 = _mm256_unpacklo_epi64(m2, m3); \  # 使用AVX2指令将m2和m3中的数据按照特定规则解包到t1中
  b0 = _mm256_blend_epi32(t0, t1, 0xF0); \  # 使用AVX2指令将t0和t1中的数据按照特定规则混合到b0中
} while(0)  # 宏定义结束

#define BLAKE2B_LOAD_MSG_0_2(b0) \  # 定义宏，将m0和m1中的数据按照特定规则组合到b0中
do { \  # 宏定义开始
t0 = _mm256_unpackhi_epi64(m0, m1);\  # 使用AVX2指令将m0和m1中的数据按照特定规则解包到t0中
t1 = _mm256_unpackhi_epi64(m2, m3);\  # 使用AVX2指令将m2和m3中的数据按照特定规则解包到t1中
b0 = _mm256_blend_epi32(t0, t1, 0xF0);\  # 使用AVX2指令将t0和t1中的数据按照特定规则混合到b0中
} while(0)  # 宏定义结束

#define BLAKE2B_LOAD_MSG_0_3(b0) \  # 定义宏，将m7和m4中的数据按照特定规则组合到b0中
do { \  # 宏定义开始
t0 = _mm256_unpacklo_epi64(m7, m4);\  # 使用AVX2指令将m7和m4中的数据按照特定规则解包到t0中
t1 = _mm256_unpacklo_epi64(m5, m6);\  # 使用AVX2指令将m5和m6中的数据按照特定规则解包到t1中
b0 = _mm256_blend_epi32(t0, t1, 0xF0);\  # 使用AVX2指令将t0和t1中的数据按照特定规则混合到b0中
} while(0)  # 宏定义结束

#define BLAKE2B_LOAD_MSG_0_4(b0) \  # 定义宏，将m7和m4中的数据按照特定规则组合到b0中
do { \  # 宏定义开始
t0 = _mm256_unpackhi_epi64(m7, m4);\  # 使用AVX2指令将m7和m4中的数据按照特定规则解包到t0中
t1 = _mm256_unpackhi_epi64(m5, m6);\  # 使用AVX2指令将m5和m6中的数据按照特定规则解包到t1中
b0 = _mm256_blend_epi32(t0, t1, 0xF0);\  # 使用AVX2指令将t0和t1中的数据按照特定规则混合到b0中
} while(0)  # 宏定义结束

#define BLAKE2B_LOAD_MSG_1_1(b0) \  # 定义宏，将m7和m2中的数据按照特定规则组合到b0中
do { \  # 宏定义开始
t0 = _mm256_unpacklo_epi64(m7, m2);\  # 使用AVX2指令将m7和m2中的数据按照特定规则解包到t0中
t1 = _mm256_unpackhi_epi64(m4, m6);\  # 使用AVX2指令将m4和m6中的数据按照特定规则解包到t1中
b0 = _mm256_blend_epi32(t0, t1, 0xF0);\  # 使用AVX2指令将t0和t1中的数据按照特定规则混合到b0中
} while(0)  # 宏定义结束

#define BLAKE2B_LOAD_MSG_1_2(b0) \  # 定义宏，将m5和m4中的数据按照特定规则组合到b0中
do { \  # 宏定义开始
t0 = _mm256_unpacklo_epi64(m5, m4);\  # 使用AVX2指令将m5和m4中的数据按照特定规则解包到t0中
t1 = _mm256_alignr_epi8(m3, m7, 8);\  # 使用AVX2指令将m3和m7中的数据按照特定规则对齐到t1中
b0 = _mm256_blend_epi32(t0, t1, 0xF0);\  # 使用AVX2指令将t0和t1中的数据按照特定规则混合到b0中
} while(0)  # 宏定义结束

#define BLAKE2B_LOAD_MSG_1_3(b0) \  # 定义宏，将m2和m0中的数据按照特定规则组合到b0中
do { \  # 宏定义开始
t0 = _mm256_unpackhi_epi64(m2, m0);\  # 使用AVX2指令将m2和m0中的数据按照特定规则解包到t0中
t1 = _mm256_blend_epi32(m5, m0, 0x33);\  # 使用AVX2指令将m5和m0中的数据按照特定规则混合到t1中
b0 = _mm256_blend_epi32(t0, t1, 0xF0);\  # 使用AVX2指令将t0和t1中的数据按照特定规则混合到b0中
} while(0)  # 宏定义结束

#define BLAKE2B_LOAD_MSG_1_4(b0) \  # 定义宏，将m6和m1中的数据按照特定规则组合到b0中
do { \  # 宏定义开始
t0 = _mm256_alignr_epi8(m6, m1, 8);\  # 使用AVX2指令将m6和m1中的数据按照特定规则对齐到t0中
t1 = _mm256_blend_epi32(m3, m1, 0x33);\  # 使用AVX2指令将m3和m1中的数据按照特定规则混合到t1中
b0 = _mm256_blend_epi32(t0, t1, 0xF0);\  # 使用AVX2指令将t0和t1中的数据按照特定规则混合到b0中
} while(0)  # 宏定义结束

#define BLAKE2B_LOAD_MSG_2_1(b0) \  # 定义宏，将m6和m5中的数据按照特定规则组合到b0中
do { \  # 宏定义开始
t0 = _mm256_alignr_epi8(m6, m5, 8);\  # 使用AVX2指令将m6和m5中的数据按照特定规则对齐到t0中
t1 = _mm256_unpackhi_epi64(m2, m7);\  # 使用AVX2指令将m2和m7中的数据按照特定规则解包到t1中
b0 = _mm256_blend_epi32(t0, t1, 0xF0);\  # 使用AVX2指令将t0和t1中的数据按照特定规则混合到b0中
} while(0)  # 宏定义结束

#define BLAKE2B_LOAD_MSG_2_2(b0) \  # 定义宏，将m4和m0中的数据按照特定规则组合到b0中
do { \  # 宏定义开始
t0 = _mm256_unpacklo_epi64(m4, m0);\  # 使用AVX2指令将m4和m0中的数据按照特定规则解包到t0中
t1 = _mm256_blend_epi32(m6, m1, 0x33);\  # 使用AVX2指令将m6和m1中的数据按照特定规则混合到t1中
b0 = _mm256_blend_epi32(t0, t1, 0xF0);\  # 使用AVX2指令将t0和t1中的数据按照特定规则混合到b0中
} while(0)  # 宏定义结束

#define BLAKE2B_LOAD_MSG_2_3(b0) \  # 定义宏，将m5和m4中的数据按照特定规则组合到b0中
do { \  # 宏定义开始
t0 = _mm256_alignr_epi8(m5, m4, 8);\  # 使用AVX2指令将m5和m4中的数据按照特定规则对齐到t0中
t1 = _mm256_unpackhi_epi64(m1, m3);\  # 使用AVX2指令将m1和m3中的数据按照特定规则解包到t1中
b0 = _mm256_blend_epi32(t0, t1, 0xF0);\  # 使用AVX2指令将t0和t1中的数据按照特定规则混合到b0中
} while(0)  # 宏定义结束

#define BLAKE2B_LOAD_MSG_2_4(b0) \  # 定义宏，将m7和m3中的数据按照特定规则组合到b0中
do { \  # 宏定义开始
#define BLAKE2B_LOAD_MSG_3_1(b0) \
do { \
t0 = _mm256_unpackhi_epi64(m3, m1);\
t1 = _mm256_unpackhi_epi64(m6, m5);\
b0 = _mm256_blend_epi32(t0, t1, 0xF0);\
} while(0)

# 定义宏 BLAKE2B_LOAD_MSG_3_1，用于加载消息数据
# 使用 AVX2 指令集中的 _mm256_unpackhi_epi64 函数对 m3 和 m1 进行高位拆分
# 使用 AVX2 指令集中的 _mm256_unpackhi_epi64 函数对 m6 和 m5 进行高位拆分
# 使用 AVX2 指令集中的 _mm256_blend_epi32 函数将 t0 和 t1 进行混合，掩码为 0xF0
# 将混合结果赋值给 b0
# 结束宏定义

#define BLAKE2B_LOAD_MSG_3_2(b0) \
do { \
t0 = _mm256_unpackhi_epi64(m4, m0);\
t1 = _mm256_unpacklo_epi64(m6, m7);\
b0 = _mm256_blend_epi32(t0, t1, 0xF0);\
} while(0)

# 定义宏 BLAKE2B_LOAD_MSG_3_2，用于加载消息数据
# 使用 AVX2 指令集中的 _mm256_unpackhi_epi64 函数对 m4 和 m0 进行高位拆分
# 使用 AVX2 指令集中的 _mm256_unpacklo_epi64 函数对 m6 和 m7 进行低位拆分
# 使用 AVX2 指令集中的 _mm256_blend_epi32 函数将 t0 和 t1 进行混合，掩码为 0xF0
# 将混合结果赋值给 b0
# 结束宏定义

... （后续宏定义类似，用于加载不同位置的消息数据）
# 定义一个宏，用于加载消息的第5个和第4个字节块
# 将m4和m6的高64位解包到t0中
# 将m7和m2的低64位对齐到t1中
# 使用t0和t1混合生成b0
# 宏定义结束
#define BLAKE2B_LOAD_MSG_5_4(b0) \
do { \
t0 = _mm256_unpackhi_epi64(m4, m6);\
t1 = _mm256_alignr_epi8(m7, m2, 8);\
b0 = _mm256_blend_epi32(t0, t1, 0xF0);\
} while(0)

# 定义一个宏，用于加载消息的第6个和第1个字节块
# 将m0和m6的低64位混合到t0中
# 将m7和m2的低64位解包到t1中
# 使用t0和t1混合生成b0
# 宏定义结束
#define BLAKE2B_LOAD_MSG_6_1(b0) \
do { \
t0 = _mm256_blend_epi32(m0, m6, 0x33);\
t1 = _mm256_unpacklo_epi64(m7, m2);\
b0 = _mm256_blend_epi32(t0, t1, 0xF0);\
} while(0)

# 定义一个宏，用于加载消息的第6个和第2个字节块
# 将m2和m7的高64位解包到t0中
# 将m5和m6的低64位对齐到t1中
# 使用t0和t1混合生成b0
# 宏定义结束
#define BLAKE2B_LOAD_MSG_6_2(b0) \
do { \
t0 = _mm256_unpackhi_epi64(m2, m7);\
t1 = _mm256_alignr_epi8(m5, m6, 8);\
b0 = _mm256_blend_epi32(t0, t1, 0xF0);\
} while(0)

# 定义一个宏，用于加载消息的第6个和第3个字节块
# 将m4和m0的低64位解包到t0中
# 将m4和m3的低64位混合到t1中
# 使用t0和t1混合生成b0
# 宏定义结束
#define BLAKE2B_LOAD_MSG_6_3(b0) \
do { \
t0 = _mm256_unpacklo_epi64(m4, m0);\
t1 = _mm256_blend_epi32(m4, m3, 0x33);\
b0 = _mm256_blend_epi32(t0, t1, 0xF0);\
} while(0)

# 定义一个宏，用于加载消息的第6个和第4个字节块
# 将m5和m3的高64位解包到t0中
# 将m1的第1个和第0个字节块混合到t1中
# 使用t0和t1混合生成b0
# 宏定义结束
#define BLAKE2B_LOAD_MSG_6_4(b0) \
do { \
t0 = _mm256_unpackhi_epi64(m5, m3);\
t1 = _mm256_shuffle_epi32(m1, _MM_SHUFFLE(1,0,3,2));\
b0 = _mm256_blend_epi32(t0, t1, 0xF0);\
} while(0)

# 定义一个宏，用于加载消息的第7个和第1个字节块
# 将m6和m3的高64位解包到t0中
# 将m1和m6的低64位混合到t1中
# 使用t0和t1混合生成b0
# 宏定义结束
#define BLAKE2B_LOAD_MSG_7_1(b0) \
do { \
t0 = _mm256_unpackhi_epi64(m6, m3);\
t1 = _mm256_blend_epi32(m1, m6, 0x33);\
b0 = _mm256_blend_epi32(t0, t1, 0xF0);\
} while(0)

# 定义一个宏，用于加载消息的第7个和第2个字节块
# 将m7和m5的低64位对齐到t0中
# 将m0和m4的高64位解包到t1中
# 使用t0和t1混合生成b0
# 宏定义结束
#define BLAKE2B_LOAD_MSG_7_2(b0) \
do { \
t0 = _mm256_alignr_epi8(m7, m5, 8);\
t1 = _mm256_unpackhi_epi64(m0, m4);\
b0 = _mm256_blend_epi32(t0, t1, 0xF0);\
} while(0)

# 定义一个宏，用于加载消息的第7个和第3个字节块
# 将m2和m1的低64位混合到t0中
# 将m4和m7的低64位对齐到t1中
# 使用t0和t1混合生成b0
# 宏定义结束
#define BLAKE2B_LOAD_MSG_7_3(b0) \
do { \
t0 = _mm256_blend_epi32(m2, m1, 0x33);\
t1 = _mm256_alignr_epi8(m4, m7, 8);\
b0 = _mm256_blend_epi32(t0, t1, 0xF0);\
} while(0)

# 定义一个宏，用于加载消息的第7个和第4个字节块
# 将m5和m0的低64位解包到t0中
# 将m2和m3的低64位解包到t1中
# 使用t0和t1混合生成b0
# 宏定义结束
#define BLAKE2B_LOAD_MSG_7_4(b0) \
do { \
t0 = _mm256_unpacklo_epi64(m5, m0);\
t1 = _mm256_unpacklo_epi64(m2, m3);\
b0 = _mm256_blend_epi32(t0, t1, 0xF0);\
} while(0)

# 定义一个宏，用于加载消息的第8个和第1个字节块
# 将m3和m7的低64位解包到t0中
# 将m0和m5的高64位对齐到t1中
# 使用t0和t1混合生成b0
# 宏定义结束
#define BLAKE2B_LOAD_MSG_8_1(b0) \
do { \
t0 = _mm256_unpacklo_epi64(m3, m7);\
t1 = _mm256_alignr_epi8(m0, m5, 8);\
b0 = _mm256_blend_epi32(t0, t1, 0xF0);\
} while(0)

# 定义一个宏，用于加载消息的第8个和第2个字节块
# 将m7和m4的高64位解包到t0中
# 将m4和m1的低64位对齐到t1中
# 使用t0和t1混合生成b0
# 宏定义结束
#define BLAKE2B_LOAD_MSG_8_2(b0) \
do { \
t0 = _mm256_unpackhi_epi64(m7, m4);\
t1 = _mm256_alignr_epi8(m4, m1, 8);\
b0 = _mm256_blend_epi32(t0, t1, 0xF0);\
} while(0)

# 定义一个宏，用于加载消息的第8个和第3个字节块
# 将m5和m6的低64位解包到t0中
# 将m6和m0的高64位解包到t1中
# 使用t0和t1混合生成b0
# 宏定义结束
#define BLAKE2B_LOAD_MSG_8_3(b0) \
do { \
t0 = _mm256_unpacklo_epi64(m5, m6);\
t1 = _mm256_unpackhi_epi64(m6, m0);\
#define BLAKE2B_LOAD_MSG_8_4(b0) \  # 定义宏，加载消息块中的数据到寄存器中
do { \  # 执行宏定义的操作
t0 = _mm256_alignr_epi8(m1, m2, 8);\  # 将m1和m2寄存器中的数据按字节对齐，存储到t0寄存器中
t1 = _mm256_alignr_epi8(m2, m3, 8);\  # 将m2和m3寄存器中的数据按字节对齐，存储到t1寄存器中
b0 = _mm256_blend_epi32(t0, t1, 0xF0);\  # 将t0和t1寄存器中的数据按指定掩码混合，存储到b0寄存器中
} while(0)  # 结束宏定义的操作
// 定义宏 BLAKE2B_LOAD_MSG_11_4，参数为 b0
#define BLAKE2B_LOAD_MSG_11_4(b0) \
// 开始 do-while 循环
do { \
// 使用 m6 和 m1 进行字节对齐，得到 t0
t0 = _mm256_alignr_epi8(m6, m1, 8);\
// 使用 m3 和 m1 进行混合操作，得到 t1
t1 = _mm256_blend_epi32(m3, m1, 0x33);\
// 使用 t0 和 t1 进行混合操作，得到 b0
b0 = _mm256_blend_epi32(t0, t1, 0xF0);\
// 结束 do-while 循环
} while(0)
// 结束宏定义
#endif
```