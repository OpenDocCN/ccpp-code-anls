# `nmap\libpcap\diag-control.h`

```cpp
/*
 * 设置 C 语言的编辑模式和缩进格式
 */
/* -*- Mode: c; tab-width: 8; indent-tabs-mode: 1; c-basic-offset: 8; -*- */
/*
 * 版权声明
 */
/*
 * 版权声明，版权所有
 */
/*
 * 在源代码和二进制形式下的重新分发和修改的许可条件
 */
/*
 * 1. 源代码的重新分发必须保留上述版权声明、条件列表和以下免责声明。
 */
/*
 * 2. 二进制形式的重新分发必须在文档和/或其他提供的材料中复制上述版权声明、条件列表和以下免责声明。
 */
/*
 * 3. 所有提及此软件特性或使用的广告材料必须显示以下声明：
 *    本产品包括由劳伦斯伯克利实验室的计算机系统工程组开发的软件。
 */
/*
 * 4. 未经特定事先书面许可，不得使用大学或实验室的名称来认可或推广从本软件衍生的产品。
 */
/*
 * 免责声明
 */
/*
 * 本软件由版权所有者和贡献者“按原样”提供，不提供任何明示或暗示的担保，包括但不限于对适销性和特定用途的暗示担保。在任何情况下，无论是在合同、严格责任或侵权（包括疏忽或其他方式）的情况下，都不对使用本软件造成的任何直接、间接、附带、特殊、惩罚性或后果性损害承担责任，即使已被告知可能发生此类损害的可能性。
 */
/*
 * 防止重复包含
 */
#ifndef _diag_control_h
#define _diag_control_h
/*
 * 包含 pcap/compiler-tests.h 头文件
 */
#include "pcap/compiler-tests.h"

/*
 * 如果编译器是 Clang 2.8 或者 GCC 4.6 及以上版本，则支持将 pragma 放入 #define 中
 * 我们只在有支持的编译器上使用它；下面的代码和 #define 控制是否使用该代码
 */
#if PCAP_IS_AT_LEAST_CLANG_VERSION(2,8) || PCAP_IS_AT_LEAST_GNUC_VERSION(4,6)
  #define PCAP_DO_PRAGMA(x) _Pragma (#x)
#endif

/*
 * 抑制 "switch 语句中的枚举值没有明确处理" 警告
 * 我们可能需要在多个不同的 Windows SDK 上构建，因此可能无法在所有 SDK 上包含所有枚举值在 switch 中，
 * 因为它们不一定在所有 SDK 上定义，并且与 #define 不同，没有简单的方法来测试给定枚举是否具有给定值。
 * 它可以通过配置脚本或 CMake 测试来完成。
 */
#if defined(_MSC_VER)
  #define DIAG_OFF_ENUM_SWITCH \
    __pragma(warning(push)) \
    __pragma(warning(disable:4061))
  #define DIAG_ON_ENUM_SWITCH \
    __pragma(warning(pop))
#else
  #define DIAG_OFF_ENUM_SWITCH
  #define DIAG_ON_ENUM_SWITCH
#endif

/*
 * 抑制 "switch 语句只有默认情况" 警告
 * 在 bpf_filter.c 中有一个 switch 语句，只在 Linux 上有额外的情况
 */
#if defined(_MSC_VER)
  #define DIAG_OFF_DEFAULT_ONLY_SWITCH \
    __pragma(warning(push)) \
    __pragma(warning(disable:4065))
  #define DIAG_ON_DEFAULT_ONLY_SWITCH \
    __pragma(warning(pop))
#else
  #define DIAG_OFF_DEFAULT_ONLY_SWITCH
  #define DIAG_ON_DEFAULT_ONLY_SWITCH
#endif

/*
 * 抑制 Flex、narrowing 和弃用警告
 */
#if PCAP_IS_AT_LEAST_CLANG_VERSION(2,8)
  /*
   * 如果是 Clang 2.8 或更高版本，则可以使用 "clang diagnostic ignored -Wxxx" 和 "clang diagnostic push/pop"。
   *
   * 抑制 -Wdocumentation 警告；GCC 不支持 -Wdocumentation，至少根据 GCC 7.3 文档是这样的。显然，Flex 生成的代码会引起至少某些版本的 Clang 的 -Wdocumentation 警告。
   *
   * (这可能是 clang-cl，它定义了 _MSC_VER，因此在测试 _MSC_VER 之前要测试这个。)
   */
  #define DIAG_OFF_FLEX \
    PCAP_DO_PRAGMA(clang diagnostic push) \
    PCAP_DO_PRAGMA(clang diagnostic ignored "-Wsign-compare") \
    PCAP_DO_PRAGMA(clang diagnostic ignored "-Wdocumentation") \
    PCAP_DO_PRAGMA(clang diagnostic ignored "-Wshorten-64-to-32") \
    PCAP_DO_PRAGMA(clang diagnostic ignored "-Wmissing-noreturn") \
    PCAP_DO_PRAGMA(clang diagnostic ignored "-Wunused-parameter") \
    PCAP_DO_PRAGMA(clang diagnostic ignored "-Wunreachable-code")
  #define DIAG_ON_FLEX \
    PCAP_DO_PRAGMA(clang diagnostic pop)

  /*
   * 抑制 Clang 给出的唯一的窄化警告。
   */
  #define DIAG_OFF_NARROWING \
    PCAP_DO_PRAGMA(clang diagnostic push) \
    PCAP_DO_PRAGMA(clang diagnostic ignored "-Wshorten-64-to-32")

  #define DIAG_ON_NARROWING \
    PCAP_DO_PRAGMA(clang diagnostic pop)

  /*
   * 抑制弃用警告。
   */
  #define DIAG_OFF_DEPRECATION \
    PCAP_DO_PRAGMA(clang diagnostic push) \
    PCAP_DO_PRAGMA(clang diagnostic ignored "-Wdeprecated-declarations")
  #define DIAG_ON_DEPRECATION \
    PCAP_DO_PRAGMA(clang diagnostic pop)
  #define DIAG_OFF_FORMAT_TRUNCATION
  #define DIAG_ON_FORMAT_TRUNCATION
#elif defined(_MSC_VER)
  /*
   * 这是 Microsoft Visual Studio；我们可以使用 __pragma(warning(disable:XXXX)) 和 __pragma(warning(push/pop))。
   *
   * 抑制有符号与无符号比较、窄化和不可达代码警告。
   */
  #define DIAG_OFF_FLEX \
    __pragma(warning(push)) \
    # 禁用特定警告编号 4127
    __pragma(warning(disable:4127)) \
    # 禁用特定警告编号 4242
    __pragma(warning(disable:4242)) \
    # 禁用特定警告编号 4244
    __pragma(warning(disable:4244)) \
    # 禁用特定警告编号 4702
    __pragma(warning(disable:4702))
  # 定义宏，用于在代码中开启 FLEX 诊断
  #define DIAG_ON_FLEX \
    __pragma(warning(pop))

  /*
   * 禁用窄化警告
   */
  #define DIAG_OFF_NARROWING \
    # 开启警告推送
    __pragma(warning(push)) \
    # 禁用特定警告编号 4242
    __pragma(warning(disable:4242)) \
    # 禁用特定警告编号 4311
    __pragma(warning(disable:4311))
  # 定义宏，用于在代码中开启窄化诊断
  #define DIAG_ON_NARROWING \
    __pragma(warning(pop))

  /*
   * 禁用废弃警告
   */
  #define DIAG_OFF_DEPRECATION \
    # 开启警告推送
    __pragma(warning(push)) \
    # 禁用特定警告编号 4996
    __pragma(warning(disable:4996))
  # 定义宏，用于在代码中开启废弃诊断
  #define DIAG_ON_DEPRECATION \
    __pragma(warning(pop))
  # 定义宏，用于在代码中禁用格式截断警告
  #define DIAG_OFF_FORMAT_TRUNCATION
  # 定义宏，用于在代码中开启格式截断诊断
  #define DIAG_ON_FORMAT_TRUNCATION
#elif PCAP_IS_AT_LEAST_GNUC_VERSION(4,6)
  /*
   * 如果是 GCC 4.6 或更高版本，或者是声称是这个版本的编译器。
   * 我们可以使用 "GCC diagnostic ignored -Wxxx"（在 4.2 版本中引入）和 "GCC diagnostic push/pop"（在 4.6 版本中引入）。
   */
  #define DIAG_OFF_FLEX \
    PCAP_DO_PRAGMA(GCC diagnostic push) \
    PCAP_DO_PRAGMA(GCC diagnostic ignored "-Wsign-compare") \
    PCAP_DO_PRAGMA(GCC diagnostic ignored "-Wunused-parameter") \
    PCAP_DO_PRAGMA(GCC diagnostic ignored "-Wunreachable-code")
  #define DIAG_ON_FLEX \
    PCAP_DO_PRAGMA(GCC diagnostic pop)

  /*
   * 当前 GCC 不会发出任何窄化警告。
   */
  #define DIAG_OFF_NARROWING
  #define DIAG_ON_NARROWING

  /*
   * 抑制弃用警告。
   */
  #define DIAG_OFF_DEPRECATION \
    PCAP_DO_PRAGMA(GCC diagnostic push) \
    PCAP_DO_PRAGMA(GCC diagnostic ignored "-Wdeprecated-declarations")
  #define DIAG_ON_DEPRECATION \
    PCAP_DO_PRAGMA(GCC diagnostic pop)

  /*
   * 抑制 format-truncation= 警告。
   * GCC 7.1 引入了此警告选项。较早版本（至少是某个特定的 GCC 4.6.4）将请求视为警告。
   */
  #if PCAP_IS_AT_LEAST_GNUC_VERSION(7,1)
    #define DIAG_OFF_FORMAT_TRUNCATION \
      PCAP_DO_PRAGMA(GCC diagnostic push) \
      PCAP_DO_PRAGMA(GCC diagnostic ignored "-Wformat-truncation=")
    #define DIAG_ON_FORMAT_TRUNCATION \
      PCAP_DO_PRAGMA(GCC diagnostic pop)
  #else
   #define DIAG_OFF_FORMAT_TRUNCATION
   #define DIAG_ON_FORMAT_TRUNCATION
  #endif
#else
  /*
   * 不是 Visual Studio，也不是 Clang 2.8 或更高版本，也不是 GCC 4.6 或更高版本，或者是声称是这个版本的编译器；我们不知道可以做什么。
   */
  #define DIAG_OFF_FLEX
  #define DIAG_ON_FLEX
  #define DIAG_OFF_NARROWING
  #define DIAG_ON_NARROWING
  #define DIAG_OFF_DEPRECATION
  #define DIAG_ON_DEPRECATION
  #define DIAG_OFF_FORMAT_TRUNCATION
  #define DIAG_ON_FORMAT_TRUNCATION
#endif
#ifdef YYBYACC
  /*
   * 如果使用 Berkeley YACC
   * 它会在 grammar.h 中生成 yylval 的全局声明，即使它被告知生成纯解析器，意味着它没有任何全局变量。Bison 不会这样做。
   * 这会导致警告，因为解析器中的局部声明会遮蔽全局声明。
   * 因此，如果编译器对此发出警告，我们就关闭 -Wshadow 警告。
   * 此外，生成的代码可能会有包含不可达代码的函数，因此我们要禁止关于这些的警告。
   */
  #if PCAP_IS_AT_LEAST_CLANG_VERSION(2,8)
    /*
     * 这是 Clang 2.8 或更高版本（包括 clang-cl，所以在 _MSC_VER 之前测试这个）；我们可以使用 "clang diagnostic ignored -Wxxx"。
     */
    #define DIAG_OFF_BISON_BYACC \
      PCAP_DO_PRAGMA(clang diagnostic ignored "-Wshadow") \
      PCAP_DO_PRAGMA(clang diagnostic ignored "-Wunreachable-code")
  #elif defined(_MSC_VER)
    /*
     * 这是 Microsoft Visual Studio；我们可以使用 __pragma(warning(disable:XXXX))。
     */
    #define DIAG_OFF_BISON_BYACC \
      __pragma(warning(disable:4702))
  #elif PCAP_IS_AT_LEAST_GNUC_VERSION(4,6)
    /*
     * 这是 GCC 4.6 或更高版本，或者声称是那个版本的编译器。
     * 我们可以使用 "GCC diagnostic ignored -Wxxx"（在 4.2 中引入，但在 4.6 之前可能实际上并不工作得很好）。
     */
    #define DIAG_OFF_BISON_BYACC \
      PCAP_DO_PRAGMA(GCC diagnostic ignored "-Wshadow") \
      PCAP_DO_PRAGMA(GCC diagnostic ignored "-Wunreachable-code")
  #else
    /*
     * 既不是 Clang 2.8 或更高版本，也不是 GCC 4.6 或更高版本，或者声称是那个版本的编译器；我们不知道可以做什么。
     */
    #define DIAG_OFF_BISON_BYACC
  #endif
#else
  /*
   * 如果不是以上条件，则执行以下代码
   * Bison.
   *
   * 生成的代码可能包含具有无法到达的代码的函数和只有默认情况的开关，因此抑制有关这些情况的警告。
   */
  #if PCAP_IS_AT_LEAST_CLANG_VERSION(2,8)
    /*
     * 这是 Clang 2.8 或更高版本（包括 clang-cl），因此在 _MSC_VER 之前测试这一点；我们可以使用 "clang diagnostic ignored -Wxxx"。
     */
    #define DIAG_OFF_BISON_BYACC \
      PCAP_DO_PRAGMA(clang diagnostic ignored "-Wunreachable-code")
  #elif defined(_MSC_VER)
    /*
     * 这是 Microsoft Visual Studio；我们可以使用 __pragma(warning(disable:XXXX))。
     *
     * 抑制一些 /Wall 警告。
     */
    #define DIAG_OFF_BISON_BYACC \
      __pragma(warning(disable:4065)) \
      __pragma(warning(disable:4127)) \
      __pragma(warning(disable:4242)) \
      __pragma(warning(disable:4244)) \
      __pragma(warning(disable:4702)
  #elif PCAP_IS_AT_LEAST_GNUC_VERSION(4,6)
    /*
     * 这是 GCC 4.6 或更高版本，或者声称是那样的编译器。
     * 我们可以使用 "GCC diagnostic ignored -Wxxx"（在 4.2 中引入，但在 4.6 之前实际上可能不太好用）。
     */
    #define DIAG_OFF_BISON_BYACC \
      PCAP_DO_PRAGMA(GCC diagnostic ignored "-Wunreachable-code")
  #else
    /*
     * 既不是 Clang 2.8 或更高版本，也不是 GCC 4.6 或更高版本，或者声称是那样的编译器；我们不知道可以做什么。
     */
    #define DIAG_OFF_BISON_BYACC
  #endif
#endif

/*
 * GCC 在 AIX 上需要这个用于 longjmp()。
 */
#if PCAP_IS_AT_LEAST_GNUC_VERSION(5,1)
  /*
   * 请注意，此内建函数的效果不仅仅是消除警告！GCC 信任它足够让进程在控制流到达内建函数时崩溃（在相同上下文中的无限空循环也会消除警告并以不同的方式破坏进程）。
   * 因此，请务必非常小心地使用这个。
   */
  #define PCAP_UNREACHABLE __builtin_unreachable();
#else
  #define PCAP_UNREACHABLE
#endif

#endif /* _diag_control_h */
```