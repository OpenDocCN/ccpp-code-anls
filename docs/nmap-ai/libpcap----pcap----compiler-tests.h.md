# `nmap\libpcap\pcap\compiler-tests.h`

```cpp
/*
 * 设置 C 语言的编辑模式和缩进格式
 */
/* 版权声明，版权归加利福尼亚大学理事会所有 */
/*
 * 允许在源代码和二进制形式下重新分发和使用，需满足以下条件：
 * 1. 源代码的再分发必须保留上述版权声明、条件列表和以下免责声明。
 * 2. 二进制形式的再分发必须在文档和/或其他提供的材料中重现上述版权声明、条件列表和以下免责声明。
 * 3. 所有提及此软件特性或使用的广告材料必须显示以下声明：
 *    本产品包括由劳伦斯伯克利实验室的计算机系统工程组开发的软件。
 * 4. 未经特定事先书面许可，不得使用大学或实验室的名称来认可或推广从本软件衍生的产品。
 */
/* 本软件由理事会和贡献者“按原样”提供，不提供任何明示或暗示的担保，包括但不限于对适销性和特定用途的暗示担保。理事会或贡献者不对任何直接、间接、附带、特殊、惩罚性或后果性损害负责（包括但不限于替代商品或服务的采购、使用损失、数据或利润损失或业务中断），无论是在合同、严格责任或侵权行为（包括疏忽或其他方式）的任何责任理论下，即使已被告知可能发生此类损害的可能性。
 */
/* 如果未定义 lib_pcap_compiler_tests_h，则执行以下代码 */
#ifndef lib_pcap_compiler_tests_h
#define lib_pcap_compiler_tests_h

/*
 * This was introduced by Clang:
 *
 *     https://clang.llvm.org/docs/LanguageExtensions.html#has-attribute
 *
 * in some version (which version?); it has been picked up by GCC 5.0.
 */
#ifndef __has_attribute
  /*
   * It's a macro, so you can check whether it's defined to check
   * whether it's supported.
   *
   * If it's not, define it to always return 0, so that we move on to
   * the fallback checks.
   */
  #define __has_attribute(x) 0
#endif

/*
 * Note that the C90 spec's "6.8.1 Conditional inclusion" and the
 * C99 spec's and C11 spec's "6.10.1 Conditional inclusion" say:
 *
 *    Prior to evaluation, macro invocations in the list of preprocessing
 *    tokens that will become the controlling constant expression are
 *    replaced (except for those macro names modified by the defined unary
 *    operator), just as in normal text.  If the token "defined" is
 *    generated as a result of this replacement process or use of the
 *    "defined" unary operator does not match one of the two specified
 *    forms prior to macro replacement, the behavior is undefined.
 *
 * so you shouldn't use defined() in a #define that's used in #if or
 * #elif.  Some versions of Clang, for example, will warn about this.
 *
 * Instead, we check whether the pre-defined macros for particular
 * compilers are defined and, if not, define the "is this version XXX
 * or a later version of this compiler" macros as 0.
 */

/*
 * Check whether this is GCC major.minor or a later release, or some
 * compiler that claims to be "just like GCC" of that version or a
 * later release.
 */

#if ! defined(__GNUC__)
  /* Not GCC and not "just like GCC" */
  #define PCAP_IS_AT_LEAST_GNUC_VERSION(major, minor) 0
#else
  /* GCC or "just like GCC" */
  #define PCAP_IS_AT_LEAST_GNUC_VERSION(major, minor) \
    (__GNUC__ > (major) || \
     (__GNUC__ == (major) && __GNUC_MINOR__ >= (minor)))
#endif
/*
 * 检查是否为 Clang 的 major.minor 版本或更高版本。
 */
#if !defined(__clang__)
  /* 不是 Clang */
  #define PCAP_IS_AT_LEAST_CLANG_VERSION(major, minor) 0
#else
  /* 是 Clang */
  #define PCAP_IS_AT_LEAST_CLANG_VERSION(major, minor) \
    (__clang_major__ > (major) || \
     (__clang_major__ == (major) && __clang_minor__ >= (minor)))
#endif

/*
 * 检查是否为 Sun C/SunPro C/Oracle Studio 的 major.minor 版本或更高版本。
 *
 * 在 __SUNPRO_C 中的版本号以十六进制BCD编码，最高的十六进制数字是主版本号，接下来的一个或两个十六进制数字是次版本号，最后一个数字是补丁版本号。
 *
 * 它代表的是 *编译器* 版本，而不是产品版本；参见
 *
 *    https://sourceforge.net/p/predef/wiki/Compilers/
 *
 * 了解部分映射，我们假设它会持续到后续的12.x产品发布。
 */
#if ! defined(__SUNPRO_C)
  /* 不是 Sun/Oracle C */
  #define PCAP_IS_AT_LEAST_SUNC_VERSION(major,minor) 0
#else
  /* Sun/Oracle C */
  #define PCAP_SUNPRO_VERSION_TO_BCD(major, minor) \
    (((minor) >= 10) ? \
        (((major) << 12) | (((minor)/10) << 8) | (((minor)%10) << 4)) : \
        (((major) << 8) | ((minor) << 4)))
  #define PCAP_IS_AT_LEAST_SUNC_VERSION(major,minor) \
    (__SUNPRO_C >= PCAP_SUNPRO_VERSION_TO_BCD((major), (minor)))
#endif

/*
 * 检查是否为 IBM XL C 的 major.minor 版本或更高版本。
 *
 * 在 __xlC__ 中的版本号将主版本号放在高8位，次版本号放在低8位。
 * 在 AIX 上，__xlC__ 总是被定义，__ibmxl__ 在 XL C 16.1 中变得被定义。
 * 在 Linux 上，自 XL C 13.1.6 起，默认情况下不再定义 __xlC__，但自 XL C 13.1.1 起定义了 __ibmxl__。
 */
#if ! defined(__xlC__) && ! defined(__ibmxl__)
  /* 不是 XL C */
  #define PCAP_IS_AT_LEAST_XL_C_VERSION(major,minor) 0
#else
  /* XL C */
  #if defined(__ibmxl__)
    /*
     * 如果是后续的 Linux 版本的 XL C 编译器，则使用 __ibmxl_version__ 进行版本测试
     */
    #define PCAP_IS_AT_LEAST_XL_C_VERSION(major, minor) \
    (__ibmxl_version__ > (major) || \
     (__ibmxl_version__ == (major) && __ibmxl_release__ >= (minor)))
    #else /* __ibmxl__ */
    /*
     * 如果 __ibmxl__ 未定义，则使用 __xlC__ 进行版本测试
     */
    #define PCAP_IS_AT_LEAST_XL_C_VERSION(major, minor) \
    (__xlC__ >= (((major) << 8) | (minor)))
    #endif /* __ibmxl__ */
#endif

/*
 * 检查这是否是 HP aC++/HP C 的主要.次要版本或更高版本。
 *
 * 在 __HP_aCC 中的版本号以零填充的十进制BCD编码，去掉"A."，最高的两个十进制数字是主要版本号，接下来的两个十进制数字是次要版本号，最后两个十进制数字是补丁版本号。
 * (去掉"A."，去掉主要和次要版本号之间的点，再加上两位补丁版本号。)
 */

#if ! defined(__HP_aCC)
  /* 不是 HP C */
  #define PCAP_IS_AT_LEAST_HP_C_VERSION(major,minor) 0
#else
  /* 是 HP C */
  #define PCAP_IS_AT_LEAST_HP_C_VERSION(major,minor) \
    (__HP_aCC >= ((major)*10000 + (minor)*100))
#endif

#endif /* lib_pcap_compiler_tests_h */
```