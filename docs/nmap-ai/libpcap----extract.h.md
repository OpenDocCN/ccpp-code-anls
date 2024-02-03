# `nmap\libpcap\extract.h`

```cpp
# 版权声明，版权归加利福尼亚大学所有
# 允许在源代码和二进制形式下进行再发布和使用，但需要保留版权声明和许可条款
# 如果包含二进制代码的发布包括文档或其他材料，则需要包含版权声明和许可条款
# 所有提及此软件特性或使用的广告材料都需要包含以下声明："本产品包含由加利福尼亚大学、劳伦斯伯克利实验室及其贡献者开发的软件"
# 未经事先书面许可，不得使用加利福尼亚大学或其贡献者的名称来认可或推广从本软件衍生的产品
# 本软件按原样提供，不提供任何明示或暗示的担保，包括但不限于对适销性和特定用途的暗示担保
#ifndef _WIN32
#include <arpa/inet.h>
#endif
#include <pcap/pcap-inttypes.h>
#include <pcap/compiler-tests.h>
#include "portability.h"
/*
 * 如果我们有支持 __attribute__ 的 GCC 或 Clang 版本，可以使用它来表示
 * “如果我们正在使用无符号行为消毒构建，就不要在这个函数中抱怨未定义的行为”，
 * 我们使用这个属性标记这些函数 - 我们*知道*它在 C 标准中是未定义的，
 * 但我们*也*知道它在我们目标的 ISA 和我们使用的编译器中做我们想要的事情。
 *
 * 对于 GCC 4.9.0 及更高版本，我们使用 __attribute__((no_sanitize_undefined))；
 * 5.0 之前的 GCC 没有 __has_attribute，我不确定 GCC 或 Clang 哪个首先支持
 * __attribute__((no_sanitize(XXX))。
 *
 * 对于 Clang，我们使用 __has_attribute 检查 __attribute__((no_sanitize(XXX))，
 * 因为有一些 Clang 版本支持 __attribute__((no_sanitize("undefined"))，
 * 但不支持 __attribute__((no_sanitize_undefined))。
 *
 * 我们在这里定义它，而不是在 funcattrs.h 中，因为我们只希望它在这里使用，
 * 我们不希望它被广泛使用。（任何打印机都会得到这个定义，但这至少应该
 * 让人们更难找到。）
 */
#if defined(__GNUC__) && ((__GNUC__ * 100 + __GNUC_MINOR__) >= 409)
#define UNALIGNED_OK    __attribute__((no_sanitize_undefined))
#elif __has_attribute(no_sanitize)
#define UNALIGNED_OK    __attribute__((no_sanitize("undefined")))
#else
#define UNALIGNED_OK
#endif

#if (defined(__i386__) || defined(_M_IX86) || defined(__X86__) || defined(__x86_64__) || defined(_M_X64)) || \
    (defined(__m68k__) && (!defined(__mc68000__) && !defined(__mc68010__))) || \
    (defined(__ppc__) || defined(__ppc64__) || defined(_M_PPC) || defined(_ARCH_PPC) || defined(_ARCH_PPC64)) || \
    (defined(__s390__) || defined(__s390x__) || defined(__zarch__))
# 定义一个内联函数，用于从给定地址提取大端序的无符号16位整数
UNALIGNED_OK static inline uint16_t
EXTRACT_BE_U_2(const void *p)
{
    # 将指针强制转换为无符号16位整数指针，然后通过它获取数据并进行网络字节序转换
    return ((uint16_t)ntohs(*(const uint16_t *)(p)));
}

# 定义一个内联函数，用于从给定地址提取大端序的有符号16位整数
UNALIGNED_OK static inline int16_t
EXTRACT_BE_S_2(const void *p)
{
    # 将指针强制转换为有符号16位整数指针，然后通过它获取数据并进行网络字节序转换
    return ((int16_t)ntohs(*(const int16_t *)(p)));
}

# 定义一个内联函数，用于从给定地址提取大端序的无符号32位整数
UNALIGNED_OK static inline uint32_t
EXTRACT_BE_U_4(const void *p)
{
    # 将指针强制转换为无符号32位整数指针，然后通过它获取数据并进行网络字节序转换
    return ((uint32_t)ntohl(*(const uint32_t *)(p)));
}

# 定义一个内联函数，用于从给定地址提取大端序的有符号32位整数
UNALIGNED_OK static inline int32_t
EXTRACT_BE_S_4(const void *p)
{
    # 将指针强制转换为有符号32位整数指针，然后通过它获取数据并进行网络字节序转换
    return ((int32_t)ntohl(*(const int32_t *)(p)));
}

# 定义一个内联函数，用于从给定地址提取大端序的无符号64位整数
UNALIGNED_OK static inline uint64_t
EXTRACT_BE_U_8(const void *p)
{
    # 将指针强制转换为无符号32位整数指针，然后通过它获取数据并进行网络字节序转换，最后将两个32位整数合并成64位整数
    return ((uint64_t)(((uint64_t)ntohl(*((const uint32_t *)(p) + 0))) << 32 |
        ((uint64_t)ntohl(*((const uint32_t *)(p) + 1))) << 0));

}

# 定义一个内联函数，用于从给定地址提取大端序的有符号64位整数
UNALIGNED_OK static inline int64_t
EXTRACT_BE_S_8(const void *p)
{
    # 将指针强制转换为无符号32位整数指针，然后通过它获取数据并进行网络字节序转换，最后将两个32位整数合并成64位整数
    return ((int64_t)(((int64_t)ntohl(*((const uint32_t *)(p) + 0))) << 32 |
        ((uint64_t)ntohl(*((const uint32_t *)(p) + 1))) << 0));

}
#elif PCAP_IS_AT_LEAST_GNUC_VERSION(2,0) && \
    (defined(__alpha) || defined(__alpha__) || \
     defined(__mips) || defined(__mips__))
typedef struct {
    uint16_t    val;
} __attribute__((packed)) unaligned_uint16_t;

typedef struct {
    int16_t        val;
} __attribute__((packed)) unaligned_int16_t;

typedef struct {
    uint32_t    val;
} __attribute__((packed)) unaligned_uint32_t;

typedef struct {
    int32_t        val;
} __attribute__((packed)) unaligned_int32_t;

# 定义一个内联函数，用于从给定地址提取大端序的无符号16位整数
UNALIGNED_OK static inline uint16_t
EXTRACT_BE_U_2(const void *p)
{
    # 将指针 p 强制转换为 const unaligned_uint16_t* 类型，然后取其 val 成员
    # 使用 ntohs 函数将网络字节顺序转换为主机字节顺序
    # 再将结果强制转换为 uint16_t 类型，并返回
    return ((uint16_t)ntohs(((const unaligned_uint16_t *)(p))->val));
# 定义一个静态内联函数，用于从给定地址提取大端序的 16 位有符号整数
UNALIGNED_OK static inline int16_t
EXTRACT_BE_S_2(const void *p)
{
    return ((int16_t)ntohs(((const unaligned_int16_t *)(p))->val));
}

# 定义一个静态内联函数，用于从给定地址提取大端序的 32 位无符号整数
UNALIGNED_OK static inline uint32_t
EXTRACT_BE_U_4(const void *p)
{
    return ((uint32_t)ntohl(((const unaligned_uint32_t *)(p))->val));
}

# 定义一个静态内联函数，用于从给定地址提取大端序的 32 位有符号整数
UNALIGNED_OK static inline int32_t
EXTRACT_BE_S_4(const void *p)
{
    return ((int32_t)ntohl(((const unaligned_int32_t *)(p))->val));
}

# 定义一个静态内联函数，用于从给定地址提取大端序的 64 位无符号整数
UNALIGNED_OK static inline uint64_t
EXTRACT_BE_U_8(const void *p)
{
    return ((uint64_t)(((uint64_t)ntohl(((const unaligned_uint32_t *)(p) + 0)->val)) << 32 |
        ((uint64_t)ntohl(((const unaligned_uint32_t *)(p) + 1)->val)) << 0));
}

# 定义一个静态内联函数，用于从给定地址提取大端序的 64 位有符号整数
UNALIGNED_OK static inline int64_t
EXTRACT_BE_S_8(const void *p)
{
    return ((int64_t)(((uint64_t)ntohl(((const unaligned_uint32_t *)(p) + 0)->val)) << 32 |
        ((uint64_t)ntohl(((const unaligned_uint32_t *)(p) + 1)->val)) << 0));
}
#else
# 如果不支持非对齐加载，则使用下面的宏定义进行加载
# 定义一个宏，用于从给定地址提取大端序的 16 位无符号整数
#define EXTRACT_BE_U_2(p) \
    ((uint16_t)(((uint16_t)(*((const uint8_t *)(p) + 0)) << 8) | \
                ((uint16_t)(*((const uint8_t *)(p) + 1)) << 0)))
# 定义一个宏，用于从给定地址提取大端序的 16 位有符号整数
#define EXTRACT_BE_S_2(p) \
    # 将指针 p 指向的内存中的两个字节按大端序解析为有符号 16 位整数
    ((int16_t)(((uint16_t)(*((const uint8_t *)(p) + 0)) << 8) | \
               ((uint16_t)(*((const uint8_t *)(p) + 1)) << 0)))
#define EXTRACT_BE_U_4(p) \
    // 从指针 p 指向的地址开始，提取4字节的大端序列无符号整数
    ((uint32_t)(((uint32_t)(*((const uint8_t *)(p) + 0)) << 24) | \
                ((uint32_t)(*((const uint8_t *)(p) + 1)) << 16) | \
                ((uint32_t)(*((const uint8_t *)(p) + 2)) << 8) | \
                ((uint32_t)(*((const uint8_t *)(p) + 3)) << 0)))
#define EXTRACT_BE_S_4(p) \
    // 从指针 p 指向的地址开始，提取4字节的大端序列有符号整数
    ((int32_t)(((uint32_t)(*((const uint8_t *)(p) + 0)) << 24) | \
               ((uint32_t)(*((const uint8_t *)(p) + 1)) << 16) | \
               ((uint32_t)(*((const uint8_t *)(p) + 2)) << 8) | \
               ((uint32_t)(*((const uint8_t *)(p) + 3)) << 0)))
#define EXTRACT_BE_U_8(p) \
    // 从指针 p 指向的地址开始，提取8字节的大端序列无符号整数
    ((uint64_t)(((uint64_t)(*((const uint8_t *)(p) + 0)) << 56) | \
                ((uint64_t)(*((const uint8_t *)(p) + 1)) << 48) | \
                ((uint64_t)(*((const uint8_t *)(p) + 2)) << 40) | \
                ((uint64_t)(*((const uint8_t *)(p) + 3)) << 32) | \
                ((uint64_t)(*((const uint8_t *)(p) + 4)) << 24) | \
                ((uint64_t)(*((const uint8_t *)(p) + 5)) << 16) | \
                ((uint64_t)(*((const uint8_t *)(p) + 6)) << 8) | \
                ((uint64_t)(*((const uint8_t *)(p) + 7)) << 0)))
#define EXTRACT_BE_S_8(p) \
    // 从指针 p 指向的地址开始，提取8字节的大端序列有符号整数
    ((int64_t)(((uint64_t)(*((const uint8_t *)(p) + 0)) << 56) | \
               ((uint64_t)(*((const uint8_t *)(p) + 1)) << 48) | \
               ((uint64_t)(*((const uint8_t *)(p) + 2)) << 40) | \
               ((uint64_t)(*((const uint8_t *)(p) + 3)) << 32) | \
               ((uint64_t)(*((const uint8_t *)(p) + 4)) << 24) | \
               ((uint64_t)(*((const uint8_t *)(p) + 5)) << 16) | \
               ((uint64_t)(*((const uint8_t *)(p) + 6)) << 8) | \
               ((uint64_t)(*((const uint8_t *)(p) + 7)) << 0)))

/*
 * 从网络字节序提取 IPv4 地址，并以主机字节序提供结果。
 */
#define EXTRACT_IPV4_TO_HOST_ORDER(p) \
    # 将指针 p 指向的内存中的 4 个字节按大端序解析为一个 32 位的无符号整数
    ((uint32_t)(((uint32_t)(*((const uint8_t *)(p) + 0)) << 24) | \  # 将第一个字节左移 24 位
                ((uint32_t)(*((const uint8_t *)(p) + 1)) << 16) | \  # 将第二个字节左移 16 位
                ((uint32_t)(*((const uint8_t *)(p) + 2)) << 8) | \   # 将第三个字节左移 8 位
                ((uint32_t)(*((const uint8_t *)(p) + 3)) << 0)))      # 将第四个字节左移 0 位
#endif /* unaligned access checks */

/*
 * Non-power-of-2 sizes.
 */

// 从指针 p 指向的地址开始，提取3字节的大端无符号整数
#define EXTRACT_BE_U_3(p) \
    ((uint32_t)(((uint32_t)(*((const uint8_t *)(p) + 0)) << 16) | \
                ((uint32_t)(*((const uint8_t *)(p) + 1)) << 8) | \
                ((uint32_t)(*((const uint8_t *)(p) + 2)) << 0)))

// 从指针 p 指向的地址开始，提取3字节的大端有符号整数
#define EXTRACT_BE_S_3(p) \
    (((*((const uint8_t *)(p) + 0)) & 0x80) ? \
      ((int32_t)(((uint32_t)(*((const uint8_t *)(p) + 0)) << 16) | \
                 ((uint32_t)(*((const uint8_t *)(p) + 1)) << 8) | \
                 ((uint32_t)(*((const uint8_t *)(p) + 2)) << 0))) : \
      ((int32_t)(0xFF000000U | \
                 ((uint32_t)(*((const uint8_t *)(p) + 0)) << 16) | \
                 ((uint32_t)(*((const uint8_t *)(p) + 1)) << 8) | \
                 ((uint32_t)(*((const uint8_t *)(p) + 2)) << 0))))

// 从指针 p 指向的地址开始，提取5字节的大端无符号整数
#define EXTRACT_BE_U_5(p) \
    ((uint64_t)(((uint64_t)(*((const uint8_t *)(p) + 0)) << 32) | \
                ((uint64_t)(*((const uint8_t *)(p) + 1)) << 24) | \
                ((uint64_t)(*((const uint8_t *)(p) + 2)) << 16) | \
                ((uint64_t)(*((const uint8_t *)(p) + 3)) << 8) | \
                ((uint64_t)(*((const uint8_t *)(p) + 4)) << 0)))

// 从指针 p 指向的地址开始，提取5字节的大端有符号整数
#define EXTRACT_BE_S_5(p) \
    # 检查指针 p 所指向的内存中第一个字节的最高位是否为 1
    (((const uint8_t *)(p) + 0)) & 0x80) ? \
        # 如果最高位为 1，则将后续 5 个字节的数据按大端序解析为 64 位有符号整数
        ((int64_t)(((uint64_t)(*((const uint8_t *)(p) + 0)) << 32) | \
                   ((uint64_t)(*((const uint8_t *)(p) + 1)) << 24) | \
                   ((uint64_t)(*((const uint8_t *)(p) + 2)) << 16) | \
                   ((uint64_t)(*((const uint8_t *)(p) + 3)) << 8) | \
                   ((uint64_t)(*((const uint8_t *)(p) + 4)) << 0))) : \
        # 如果最高位不为 1，则将后续 5 个字节的数据按大端序解析为 64 位有符号整数，并将最高 32 位补 1
        ((int64_t)(INT64_T_CONSTANT(0xFFFFFF0000000000U) | \
                   ((uint64_t)(*((const uint8_t *)(p) + 0)) << 32) | \
                   ((uint64_t)(*((const uint8_t *)(p) + 1)) << 24) | \
                   ((uint64_t)(*((const uint8_t *)(p) + 2)) << 16) | \
                   ((uint64_t)(*((const uint8_t *)(p) + 3)) << 8) | \
                   ((uint64_t)(*((const uint8_t *)(p) + 4)) << 0)))
// 定义一个宏，用于从大端序的6个字节中提取出一个无符号64位整数
#define EXTRACT_BE_U_6(p) \
    ((uint64_t)(((uint64_t)(*((const uint8_t *)(p) + 0)) << 40) | \
                ((uint64_t)(*((const uint8_t *)(p) + 1)) << 32) | \
                ((uint64_t)(*((const uint8_t *)(p) + 2)) << 24) | \
                ((uint64_t)(*((const uint8_t *)(p) + 3)) << 16) | \
                ((uint64_t)(*((const uint8_t *)(p) + 4)) << 8) | \
                ((uint64_t)(*((const uint8_t *)(p) + 5)) << 0)))

// 定义一个宏，用于从大端序的6个字节中提取出一个有符号64位整数
#define EXTRACT_BE_S_6(p) \
    (((*((const uint8_t *)(p) + 0)) & 0x80) ? \
       ((int64_t)(((uint64_t)(*((const uint8_t *)(p) + 0)) << 40) | \
                  ((uint64_t)(*((const uint8_t *)(p) + 1)) << 32) | \
                  ((uint64_t)(*((const uint8_t *)(p) + 2)) << 24) | \
                  ((uint64_t)(*((const uint8_t *)(p) + 3)) << 16) | \
                  ((uint64_t)(*((const uint8_t *)(p) + 4)) << 8) | \
                  ((uint64_t)(*((const uint8_t *)(p) + 5)) << 0))) : \
      ((int64_t)(INT64_T_CONSTANT(0xFFFFFFFF00000000U) | \
                  ((uint64_t)(*((const uint8_t *)(p) + 0)) << 40) | \
                  ((uint64_t)(*((const uint8_t *)(p) + 1)) << 32) | \
                  ((uint64_t)(*((const uint8_t *)(p) + 2)) << 24) | \
                  ((uint64_t)(*((const uint8_t *)(p) + 3)) << 16) | \
                  ((uint64_t)(*((const uint8_t *)(p) + 4)) << 8) | \
                  ((uint64_t)(*((const uint8_t *)(p) + 5)) << 0))))

// 定义一个宏，用于从大端序的7个字节中提取出一个无符号64位整数
#define EXTRACT_BE_U_7(p) \
    ((uint64_t)(((uint64_t)(*((const uint8_t *)(p) + 0)) << 48) | \
                ((uint64_t)(*((const uint8_t *)(p) + 1)) << 40) | \
                ((uint64_t)(*((const uint8_t *)(p) + 2)) << 32) | \
                ((uint64_t)(*((const uint8_t *)(p) + 3)) << 24) | \
                ((uint64_t)(*((const uint8_t *)(p) + 4)) << 16) | \
                ((uint64_t)(*((const uint8_t *)(p) + 5)) << 8) | \
                ((uint64_t)(*((const uint8_t *)(p) + 6)) << 0)))

// 定义一个宏，用于从大端序的7个字节中提取出一个有符号64位整数
#define EXTRACT_BE_S_7(p) \
    # 检查指针 p 指向的内存中第一个字节的最高位是否为 1
    (((const uint8_t *)(p) + 0)) & 0x80) ? \
        # 如果最高位为 1，则将后续 7 个字节的数据按大端序解析为 64 位有符号整数
        ((int64_t)(((uint64_t)(*((const uint8_t *)(p) + 0)) << 48) | \
                   ((uint64_t)(*((const uint8_t *)(p) + 1)) << 40) | \
                   ((uint64_t)(*((const uint8_t *)(p) + 2)) << 32) | \
                   ((uint64_t)(*((const uint8_t *)(p) + 3)) << 24) | \
                   ((uint64_t)(*((const uint8_t *)(p) + 4)) << 16) | \
                   ((uint64_t)(*((const uint8_t *)(p) + 5)) << 8) | \
                   ((uint64_t)(*((const uint8_t *)(p) + 6)) << 0))) : \
        # 如果最高位不为 1，则将后续 7 个字节的数据按大端序解析为 64 位有符号整数
        ((int64_t)(INT64_T_CONSTANT(0xFFFFFFFFFF000000U) | \
                   ((uint64_t)(*((const uint8_t *)(p) + 0)) << 48) | \
                   ((uint64_t)(*((const uint8_t *)(p) + 1)) << 40) | \
                   ((uint64_t)(*((const uint8_t *)(p) + 2)) << 32) | \
                   ((uint64_t)(*((const uint8_t *)(p) + 3)) << 24) | \
                   ((uint64_t)(*((const uint8_t *)(p) + 4)) << 16) | \
                   ((uint64_t)(*((const uint8_t *)(p) + 5)) << 8) | \
                   ((uint64_t)(*((const uint8_t *)(p) + 6)) << 0)))
/*
 * 宏用于提取可能未对齐的小端整数值。
 * XXX - 在支持未对齐加载的小端机器上执行加载操作？
 */
#define EXTRACT_LE_U_2(p) \ 
    ((uint16_t)(((uint16_t)(*((const uint8_t *)(p) + 1)) << 8) | \  // 从指针p指向的地址中提取一个16位的无符号整数值
                ((uint16_t)(*((const uint8_t *)(p) + 0)) << 0)))  // 从指针p指向的地址中提取一个16位的无符号整数值
#define EXTRACT_LE_S_2(p) \ 
    ((int16_t)(((uint16_t)(*((const uint8_t *)(p) + 1)) << 8) | \  // 从指针p指向的地址中提取一个16位的有符号整数值
               ((uint16_t)(*((const uint8_t *)(p) + 0)) << 0)))  // 从指针p指向的地址中提取一个16位的有符号整数值
#define EXTRACT_LE_U_4(p) \ 
    ((uint32_t)(((uint32_t)(*((const uint8_t *)(p) + 3)) << 24) | \  // 从指针p指向的地址中提取一个32位的无符号整数值
                ((uint32_t)(*((const uint8_t *)(p) + 2)) << 16) | \  // 从指针p指向的地址中提取一个32位的无符号整数值
                ((uint32_t)(*((const uint8_t *)(p) + 1)) << 8) | \   // 从指针p指向的地址中提取一个32位的无符号整数值
                ((uint32_t)(*((const uint8_t *)(p) + 0)) << 0)))  // 从指针p指向的地址中提取一个32位的无符号整数值
#define EXTRACT_LE_S_4(p) \ 
    ((int32_t)(((uint32_t)(*((const uint8_t *)(p) + 3)) << 24) | \  // 从指针p指向的地址中提取一个32位的有符号整数值
               ((uint32_t)(*((const uint8_t *)(p) + 2)) << 16) | \  // 从指针p指向的地址中提取一个32位的有符号整数值
               ((uint32_t)(*((const uint8_t *)(p) + 1)) << 8) | \   // 从指针p指向的地址中提取一个32位的有符号整数值
               ((uint32_t)(*((const uint8_t *)(p) + 0)) << 0)))  // 从指针p指向的地址中提取一个32位的有符号整数值
#define EXTRACT_LE_U_3(p) \ 
    ((uint32_t)(((uint32_t)(*((const uint8_t *)(p) + 2)) << 16) | \  // 从指针p指向的地址中提取一个24位的无符号整数值
                ((uint32_t)(*((const uint8_t *)(p) + 1)) << 8) | \   // 从指针p指向的地址中提取一个24位的无符号整数值
                ((uint32_t)(*((const uint8_t *)(p) + 0)) << 0)))  // 从指针p指向的地址中提取一个24位的无符号整数值
#define EXTRACT_LE_S_3(p) \ 
    ((int32_t)(((uint32_t)(*((const uint8_t *)(p) + 2)) << 16) | \  // 从指针p指向的地址中提取一个24位的有符号整数值
               ((uint32_t)(*((const uint8_t *)(p) + 1)) << 8) | \   // 从指针p指向的地址中提取一个24位的有符号整数值
               ((uint32_t)(*((const uint8_t *)(p) + 0)) << 0)))  // 从指针p指向的地址中提取一个24位的有符号整数值
#define EXTRACT_LE_U_8(p) \  // 从指针p指向的地址中提取一个64位的无符号整数值
    # 将指针 p 指向的内存中的8个字节的数据按照大端序解析成一个64位的整数
    ((uint64_t)(((uint64_t)(*((const uint8_t *)(p) + 7)) << 56) | \
                ((uint64_t)(*((const uint8_t *)(p) + 6)) << 48) | \
                ((uint64_t)(*((const uint8_t *)(p) + 5)) << 40) | \
                ((uint64_t)(*((const uint8_t *)(p) + 4)) << 32) | \
                ((uint64_t)(*((const uint8_t *)(p) + 3)) << 24) | \
                ((uint64_t)(*((const uint8_t *)(p) + 2)) << 16) | \
                ((uint64_t)(*((const uint8_t *)(p) + 1)) << 8) | \
                ((uint64_t)(*((const uint8_t *)(p) + 0)) << 0)))
# 定义一个宏，用于提取小端序的8字节数据并转换为int64_t类型
#define EXTRACT_LE_S_8(p) \
    ((int64_t)(((uint64_t)(*((const uint8_t *)(p) + 7)) << 56) | \  # 将第8个字节左移56位，并转换为uint64_t类型
               ((uint64_t)(*((const uint8_t *)(p) + 6)) << 48) | \  # 将第7个字节左移48位，并转换为uint64_t类型
               ((uint64_t)(*((const uint8_t *)(p) + 5)) << 40) | \  # 将第6个字节左移40位，并转换为uint64_t类型
               ((uint64_t)(*((const uint8_t *)(p) + 4)) << 32) | \  # 将第5个字节左移32位，并转换为uint64_t类型
               ((uint64_t)(*((const uint8_t *)(p) + 3)) << 24) | \  # 将第4个字节左移24位，并转换为uint64_t类型
               ((uint64_t)(*((const uint8_t *)(p) + 2)) << 16) | \  # 将第3个字节左移16位，并转换为uint64_t类型
               ((uint64_t)(*((const uint8_t *)(p) + 1)) << 8) | \   # 将第2个字节左移8位，并转换为uint64_t类型
               ((uint64_t)(*((const uint8_t *)(p) + 0)) << 0)))     # 将第1个字节左移0位，并转换为uint64_t类型
```