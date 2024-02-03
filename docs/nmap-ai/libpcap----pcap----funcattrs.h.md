# `nmap\libpcap\pcap\funcattrs.h`

```cpp
/*
 * 以下是版权声明和许可条款
 * 版权归加利福尼亚大学理事会所有
 * 允许在源代码和二进制形式下进行再发布和使用，需满足以下条件：
 * 1. 源代码的再发布必须保留上述版权声明、条件列表和以下免责声明
 * 2. 二进制形式的再发布必须在文档和/或其他提供的材料中重现上述版权声明、条件列表和以下免责声明
 * 3. 所有提及此软件特性或使用的广告材料必须显示以下声明：
 *    本产品包含由劳伦斯伯克利实验室计算机系统工程组开发的软件
 * 4. 未经特定事先书面许可，不得使用大学或实验室的名称来认可或推广从本软件衍生的产品
 * 
 * 本软件由理事会和贡献者“按原样”提供，不提供任何明示或暗示的担保，包括但不限于对适销性和特定用途的适用性的暗示担保
 * 在任何情况下，理事会或贡献者均不对任何直接、间接、附带、特殊、惩罚性或后果性的损害（包括但不限于替代商品或服务的采购、使用损失、数据或利润的损失或业务中断）承担责任，无论是合同责任、严格责任还是侵权行为（包括疏忽或其他方式）引起的，即使已被告知可能发生此类损害
 */

#ifndef lib_pcap_funcattrs_h
#define lib_pcap_funcattrs_h
#include <pcap/compiler-tests.h>

/*
 * Attributes to apply to functions and their arguments, using various
 * compiler-specific extensions.
 */

/*
 * PCAP_API_DEF must be used when defining *data* exported from
 * libpcap. It can be used when defining *functions* exported
 * from libpcap, but it doesn't have to be used there. It
 * should not be used in declarations in headers.
 *
 * PCAP_API must be used when *declaring* data or functions
 * exported from libpcap; PCAP_API_DEF won't work on all platforms.
 */

#if defined(_WIN32)
  /*
   * For Windows:
   *
   *    when building libpcap:
   *
   *       if we're building it as a DLL, we have to declare API
   *       functions with __declspec(dllexport);
   *
   *       if we're building it as a static library, we don't want
   *       to do so.
   *
   *    when using libpcap:
   *
   *       if we're using the DLL, calls to its functions are a
   *       little more efficient if they're declared with
   *       __declspec(dllimport);
   *
   *       if we're not using the dll, we don't want to declare
   *       them that way.
   *
   * So:
   *
   *    if pcap_EXPORTS is defined, we define PCAP_API_DEF as
   *     __declspec(dllexport);
   *
   *    if PCAP_DLL is defined, we define PCAP_API_DEF as
   *    __declspec(dllimport);
   *
   *    otherwise, we define PCAP_API_DEF as nothing.
   */
  #if defined(pcap_EXPORTS)
    /*
     * We're compiling libpcap as a DLL, so we should export functions
     * in our API.
     */
    #define PCAP_API_DEF    __declspec(dllexport)
  #elif defined(PCAP_DLL)
    /*
     * We're using libpcap as a DLL, so the calls will be a little more
     * efficient if we explicitly import the functions.
     */
    #define PCAP_API_DEF    __declspec(dllimport)
  #else
    /*
     * We're not compiling libpcap as a DLL, and we're not using libpcap
     * as a DLL, so we don't need to do anything special with the
     * functions.
     */
    #define PCAP_API_DEF
  #endif
#endif
    /*
     * 如果我们正在构建 libpcap 作为静态库，或者我们正在使用它作为静态库，
     * 或者我们不能确定我们是否正在使用它作为动态库，所以不要显式导入或导出函数。
     */
    #define PCAP_API_DEF
  #endif
#elif defined(MSDOS)
  /* 如果定义了 MSDOS，则需要特殊处理 */
  #define PCAP_API_DEF
#else /* UN*X */
  #ifdef pcap_EXPORTS
    /*
     * 我们正在将 libpcap 编译为（动态）共享库，因此我们应该导出我们 API 中的函数。
     * 编译器可能未配置为默认情况下从共享库中导出函数，因此我们可能需要显式标记函数为导出。
     */
    #if PCAP_IS_AT_LEAST_GNUC_VERSION(3,4) \
        || PCAP_IS_AT_LEAST_XL_C_VERSION(12,0)
      /*
       * GCC 3.4 及更高版本，或者一些声称与 GCC 3.4 及更高版本兼容的编译器，或者 XL C 13.0 及更高版本，因此我们有
       * __attribute__((visibility()).
       */
      #define PCAP_API_DEF    __attribute__((visibility("default")))
    #elif PCAP_IS_AT_LEAST_SUNC_VERSION(5,5)
      /*
       * Sun C 5.5 及更高版本，因此我们有 __global。
       * （Sun C 5.9 及更高版本也有 __attribute__((visibility())，
       * 但是没有理由偏好于 Sun C。）
       */
      #define PCAP_API_DEF    __global
    #else
      /*
       * 我们没有任何要说的。
       */
      #define PCAP_API_DEF
    #endif
  #else
    /*
     * 我们没有在构建 libpcap。
     */
    #define PCAP_API_DEF
  #endif
#endif /* _WIN32/MSDOS/UN*X */

#define PCAP_API    PCAP_API_DEF extern
/*
 * Definitions to 1) indicate what version of libpcap first had a given
 * API and 2) allow upstream providers whose build environments allow
 * APIs to be designated as "first available in this release" to do so
 * by appropriately defining them.
 *
 * Yes, that's you, Apple. :-)  Please define PCAP_AVAILABLE_MACOS()
 * as necessary to make various APIs "weak exports" to make it easier
 * for software that's distributed in binary form and that uses libpcap
 * to run on multiple macOS versions and use new APIs when available.
 * (Yes, such third-party software exists - Wireshark provides binary
 * packages for macOS, for example.  tcpdump doesn't count, as that's
 * provided by Apple, so each release can come with a version compiled
 * to use the APIs present in that release.)
 *
 * The non-macOS versioning is based on
 *
 *    https://en.wikipedia.org/wiki/Darwin_(operating_system)#Release_history
 *
 * If there are any corrections, please submit it upstream to the
 * libpcap maintainers, preferably as a pull request on
 *
 *    https://github.com/the-tcpdump-group/libpcap
 *
 * We don't define it ourselves because, if you're building and
 * installing libpcap on macOS yourself, the APIs will be available
 * no matter what OS version you're installing it on.
 *
 * For other platforms, we don't define them, leaving it up to
 * others to do so based on their OS versions, if appropriate.
 *
 * We start with libpcap 0.4, as that was the last LBL release, and
 * I've never seen earlier releases.
 */
#ifdef __APPLE__
#include <Availability.h>
/*
 * When building as part of macOS, define this as __API_AVAILABLE(__VA_ARGS__).
 *
 * XXX - if there's some #define to indicate that this is being built
 * as part of the macOS build process, we could make that Just Work.
 */
#define PCAP_AVAILABLE(...)
#define PCAP_AVAILABLE_0_4    PCAP_AVAILABLE(macos(10.0)) /* Did any version of Mac OS X ship with this? */
#define PCAP_AVAILABLE_0_5    PCAP_AVAILABLE(macos(10.0)) /* Did any version of Mac OS X ship with this? */
#define PCAP_AVAILABLE_0_6    PCAP_AVAILABLE(macos(10.1))
#define PCAP_AVAILABLE_0_7    PCAP_AVAILABLE(macos(10.4))
#define PCAP_AVAILABLE_0_8    PCAP_AVAILABLE(macos(10.4))
#define PCAP_AVAILABLE_0_9    PCAP_AVAILABLE(macos(10.5), ios(1.0))
#define PCAP_AVAILABLE_1_0    PCAP_AVAILABLE(macos(10.6), ios(4.0))
/* #define PCAP_AVAILABLE_1_1    no routines added to the API */
#define PCAP_AVAILABLE_1_2    PCAP_AVAILABLE(macos(10.9), ios(6.0))
/* #define PCAP_AVAILABLE_1_3    no routines added to the API */
/* #define PCAP_AVAILABLE_1_4    no routines added to the API */
#define PCAP_AVAILABLE_1_5    PCAP_AVAILABLE(macos(10.10), ios(7.0), watchos(1.0))
/* #define PCAP_AVAILABLE_1_6    no routines added to the API */
#define PCAP_AVAILABLE_1_7    PCAP_AVAILABLE(macos(10.12), ios(10.0), tvos(10.0), watchos(3.0))
#define PCAP_AVAILABLE_1_8    PCAP_AVAILABLE(macos(10.13), ios(11.0), tvos(11.0), watchos(4.0)) /* only Windows adds routines to the API; XXX - what version first had it? */
#define PCAP_AVAILABLE_1_9    PCAP_AVAILABLE(macos(10.13), ios(11.0), tvos(11.0), watchos(4.0))
#define PCAP_AVAILABLE_1_10    /* not in macOS yet */
#define PCAP_AVAILABLE_1_11    /* not released yet, so not in macOS yet */
#else /* __APPLE__ */
#define PCAP_AVAILABLE_0_4
#define PCAP_AVAILABLE_0_5
#define PCAP_AVAILABLE_0_6
#define PCAP_AVAILABLE_0_7
#define PCAP_AVAILABLE_0_8
#define PCAP_AVAILABLE_0_9
#define PCAP_AVAILABLE_1_0
/* #define PCAP_AVAILABLE_1_1    no routines added to the API */
#define PCAP_AVAILABLE_1_2
/* #define PCAP_AVAILABLE_1_3    no routines added to the API */
/* #define PCAP_AVAILABLE_1_4    no routines added to the API */
#define PCAP_AVAILABLE_1_5
/* #define PCAP_AVAILABLE_1_6    no routines added to the API */
#define PCAP_AVAILABLE_1_7
#define PCAP_AVAILABLE_1_8
#define PCAP_AVAILABLE_1_9
#define PCAP_AVAILABLE_1_10
#define PCAP_AVAILABLE_1_11
/*
 * 如果是苹果系统，则结束条件编译
 */
#endif /* __APPLE__ */

/*
 * PCAP_NORETURN，在函数声明之前，表示“此函数永远不会返回”。
 * （必须在函数声明之前使用，例如“extern PCAP_NORETURN func(...)”而不是在函数声明之后使用，因为 MSVC 版本必须在声明之前使用。）
 *
 * PCAP_NORETURN_DEF，在函数*定义*之前，表示“此函数永远不会返回”；它仅用于在任何使用之前定义的静态函数，并且因此没有声明。
 * （MSVC 不支持这一点；我猜“__declspec”中的“decl”表示“declaration”，而 __declspec 与定义不兼容。）
 */
#if __has_attribute(noreturn) \
    || PCAP_IS_AT_LEAST_GNUC_VERSION(2,5) \
    || PCAP_IS_AT_LEAST_SUNC_VERSION(5,9) \
    || PCAP_IS_AT_LEAST_XL_C_VERSION(10,1) \
    || PCAP_IS_AT_LEAST_HP_C_VERSION(6,10)
  /*
   * 支持 __attribute((noreturn)) 的编译器，或者 GCC 2.5 及更高版本，或者一些声称与 GCC 2.5 及更高版本兼容的编译器，或者 Solaris Studio 12 (Sun C 5.9) 及更高版本，或者 IBM XL C 10.1 及更高版本（早期版本的 XL C 是否支持此功能？），或者 HP aCC A.06.10 及更高版本。
   */
  #define PCAP_NORETURN __attribute((noreturn))
  #define PCAP_NORETURN_DEF __attribute((noreturn))
#elif defined(_MSC_VER)
  /*
   * MSVC。
   */
  #define PCAP_NORETURN __declspec(noreturn)
  #define PCAP_NORETURN_DEF
#else
  #define PCAP_NORETURN
  #define PCAP_NORETURN_DEF
#endif

/*
 * PCAP_PRINTFLIKE(x,y)，在函数声明之后，表示“此函数使用 printf 风格的格式化，第 x 个参数是格式字符串，第 y 个参数是格式字符串的第一个参数”。
 */
#if __has_attribute(__format__) \
    || PCAP_IS_AT_LEAST_GNUC_VERSION(2,3) \
    || PCAP_IS_AT_LEAST_XL_C_VERSION(10,1) \
    || PCAP_IS_AT_LEAST_HP_C_VERSION(6,10)
  /*
   * 如果编译器支持，或者是 GCC 2.3 及以后版本，或者是一些声称与 GCC 2.3 及以后版本兼容的编译器，
   * 或者是 IBM XL C 10.1 及以后版本（早期版本的 XL C 是否支持？），
   * 或者是 HP aCC A.06.10 及以后版本。
   */
  #define PCAP_PRINTFLIKE(x,y) __attribute__((__format__(__printf__,x,y)))
#else
  #define PCAP_PRINTFLIKE(x,y)
#endif

/*
 * PCAP_DEPRECATED(func, msg), after a function declaration, marks the
 * function as deprecated.
 *
 * The argument is a string giving the warning message to use if the
 * compiler supports that.
 */
#if __has_attribute(deprecated) \
    || PCAP_IS_AT_LEAST_GNUC_VERSION(4,5) \
    || PCAP_IS_AT_LEAST_SUNC_VERSION(5,13)
  /*
   * Compiler that supports __has_attribute and __attribute__((deprecated)),
   * or GCC 4.5 and later, or Sun/Oracle C 12.4 (Sun C 5.13) and later.
   *
   * Those support __attribute__((deprecated(msg))) (we assume, perhaps
   * incorrectly, that anything that supports __has_attribute() is
   * recent enough to support __attribute__((deprecated(msg)))).
   */
  #define PCAP_DEPRECATED(msg)    __attribute__((deprecated(msg)))
#elif PCAP_IS_AT_LEAST_GNUC_VERSION(3,1)
  /*
   * GCC 3.1 through 4.4.
   *
   * Those support __attribute__((deprecated)) but not
   * __attribute__((deprecated(msg))).
   */
  #define PCAP_DEPRECATED(msg)    __attribute__((deprecated))
#elif defined(_MSC_VER) && !defined(BUILDING_PCAP)
  /*
   * MSVC, and we're not building libpcap itself; it's VS 2015
   * and later, so we have __declspec(deprecated(...)).
   *
   * If we *are* building libpcap, we don't want this, as it'll warn
   * us even if we *define* the function.
   */
  #define PCAP_DEPRECATED(msg)    _declspec(deprecated(msg))
#else
  #define PCAP_DEPRECATED(msg)
#endif

/*
 * For flagging arguments as format strings in MSVC.
 */
#ifdef _MSC_VER
 #include <sal.h>
 #define PCAP_FORMAT_STRING(p) _Printf_format_string_ p
#else
 #define PCAP_FORMAT_STRING(p) p
#endif

#endif /* lib_pcap_funcattrs_h */
```