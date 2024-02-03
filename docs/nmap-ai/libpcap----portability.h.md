# `nmap\libpcap\portability.h`

```cpp
/*
 * 版权声明，版权归加利福尼亚大学理事会所有
 * 禁止在源代码和二进制形式中进行重新分发和修改，除非满足以下条件：
 * 1. 源代码的重新分发必须保留上述版权声明、条件列表和以下免责声明。
 * 2. 二进制形式的重新分发必须在文档和/或其他提供的材料中重现上述版权声明、条件列表和以下免责声明。
 * 3. 所有提及此软件特性或使用的广告材料必须显示以下声明：
 *    本产品包含由劳伦斯伯克利实验室计算机系统工程组开发的软件。
 * 4. 未经特定事先书面许可，不得使用大学或实验室的名称来认可或推广从本软件衍生的产品。
 *
 * 本软件由理事会和贡献者“按原样”提供，不提供任何明示或暗示的担保，包括但不限于对适销性和特定用途的适用性的暗示担保。在任何情况下，理事会或贡献者均不对任何直接、间接、附带、特殊、惩罚性或后果性的损害（包括但不限于替代商品或服务的采购、使用、数据或利润的损失或业务中断）承担责任，无论是在合同、严格责任或侵权行为（包括疏忽或其他方式）的任何理论下，即使已被告知可能发生此类损害。
 */

#ifndef portability_h
#define    portability_h

/*
 * 在 Windows 和 UN*X 之间以及不同版本的 UN*X 之间实现可移植性的辅助函数
 */
#include <stdarg.h>    /* 声明可变参数函数在某些平台上 */

#include "pcap/funcattrs.h"

#ifdef __cplusplus
extern "C" {
#endif

#ifdef HAVE_STRLCAT
  #define pcap_strlcat    strlcat
#else
  #if defined(_MSC_VER) || defined(__MINGW32__)
    /*
     * strncat_s() 至少支持到 Visual Studio 2005；我们需要 Visual Studio 2015 或更高版本。
     */
    #define pcap_strlcat(x, y, z) \
    strncat_s((x), (z), (y), _TRUNCATE)
  #else
    /*
     * 自定义定义。
     */
    extern size_t pcap_strlcat(char * restrict dst, const char * restrict src, size_t dstsize);
  #endif
#endif

#ifdef HAVE_STRLCPY
  #define pcap_strlcpy    strlcpy
#else
  #if defined(_MSC_VER) || defined(__MINGW32__)
    /*
     * strncpy_s() 至少支持到 Visual Studio 2005；我们需要 Visual Studio 2015 或更高版本。
     */
    #define pcap_strlcpy(x, y, z) \
    strncpy_s((x), (z), (y), _TRUNCATE)
  #else
    /*
     * 自定义定义。
     */
    extern size_t pcap_strlcpy(char * restrict dst, const char * restrict src, size_t dstsize);
  #endif
#endif

#ifdef _MSC_VER
  /*
   * 如果 <crtdbg.h> 已经被包含，并且 _DEBUG 被定义，并且 __STDC__ 为零，<crtdbg.h> 将会定义 strdup() 调用 _strdup_dbg()。所以如果它已经被定义，就不要重新定义它。
   */
  #ifndef strdup
  #define strdup    _strdup
  #endif
#endif

/*
 * 我们想要 asprintf()，用于一些情况下我们用它来构造动态分配的可变长度字符串；它存在于一些平台上，但不是所有平台。
 */
#ifdef HAVE_ASPRINTF
#define pcap_asprintf asprintf
#else
extern int pcap_asprintf(char **, PCAP_FORMAT_STRING(const char *), ...)
    PCAP_PRINTFLIKE(2, 3);
#endif

#ifdef HAVE_VASPRINTF
#define pcap_vasprintf vasprintf
#else
extern int pcap_vasprintf(char **, const char *, va_list ap);
#endif

/* 对于 Solaris 11 之前的版本。 */
#ifndef timeradd
#ifndef timeradd
// 定义一个宏，用于计算两个时间结构相加的结果
#define timeradd(a, b, result)                       \
  do {                                               \
    (result)->tv_sec = (a)->tv_sec + (b)->tv_sec;    \
    (result)->tv_usec = (a)->tv_usec + (b)->tv_usec; \
    if ((result)->tv_usec >= 1000000) {              \
      ++(result)->tv_sec;                            \
      (result)->tv_usec -= 1000000;                  \
    }                                                \
  } while (0)
#endif /* timeradd */

#ifndef timersub
// 定义一个宏，用于计算两个时间结构相减的结果
#define timersub(a, b, result)                       \
  do {                                               \
    (result)->tv_sec = (a)->tv_sec - (b)->tv_sec;    \
    (result)->tv_usec = (a)->tv_usec - (b)->tv_usec; \
    if ((result)->tv_usec < 0) {                     \
      --(result)->tv_sec;                            \
      (result)->tv_usec += 1000000;                  \
    }                                                \
  } while (0)
#endif /* timersub */

#ifdef HAVE_STRTOK_R
  // 如果有 strtok_r 函数，则使用它
  #define pcap_strtok_r    strtok_r
#else
  #ifdef _WIN32
    /*
     * Microsoft gives it a different name.
     */
    // 在 Windows 平台上，Microsoft 使用不同的函数名
    #define pcap_strtok_r    strtok_s
  #else
    /*
     * Define it ourselves.
     */
    // 否则自己定义 strtok_r 函数
    extern char *pcap_strtok_r(char *, const char *, char **);
  #endif
#endif /* HAVE_STRTOK_R */

#ifdef _WIN32
  #if !defined(__cplusplus)
    // 在 Windows 平台上，如果不是 C++ 环境，则将 inline 定义为 __inline
    #define inline __inline
  #endif
#endif /* _WIN32 */

#ifdef __cplusplus
}
#endif

#endif
```