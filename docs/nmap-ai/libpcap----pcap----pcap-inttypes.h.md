# `nmap\libpcap\pcap\pcap-inttypes.h`

```cpp
/*
 * 版权声明，声明了代码的版权归属和使用条件
 * 1. 源代码的再发布必须保留版权声明、条件列表和免责声明
 * 2. 二进制形式的再发布必须在文档和/或其他提供的材料中重现版权声明、条件列表和免责声明
 * 3. 不得使用 Politecnico di Torino 的名称或其贡献者的名称来认可或推广从本软件衍生的产品，除非有特定的事先书面许可
 *
 * 本软件由版权所有者和贡献者提供"按原样"，不提供任何明示或暗示的担保，包括但不限于对适销性和特定用途的适用性的暗示担保。在任何情况下，无论是在合同、严格责任还是侵权行为（包括疏忽或其他情况）下，版权所有者或贡献者均不对任何直接、间接、附带、特殊、惩罚性或后果性损害（包括但不限于替代商品或服务的采购、使用、数据或利润的损失或业务中断）负责，即使已被告知可能发生此类损害的可能性。
 */
#ifndef pcap_pcap_inttypes_h
#define pcap_pcap_inttypes_h
/*
 * 如果我们使用 Visual Studio 编译，确保 C99 整数类型被定义，无论如何。
 *
 * XXX - 验证我们在 UN*X 上至少有 C99 支持？
 *
 * MinGW 或各种 DOS 工具链呢？我们目前假设那里有足够的 C99 支持。
 */
#if defined(_MSC_VER)
  /*
   * 编译器是 MSVC。
   */
  #if _MSC_VER >= 1800
    /*
     * VS 2013 或更新版本；我们有 <inttypes.h>。
     */
    #include <inttypes.h>
  #else
    /*
     * 早期的 VS；我们必须自己定义这些东西。
     * 我们不支持使用早期版本的 VS 构建 libpcap，但 Npcap 的 SDK 必须支持使用早期版本的 VS 构建应用程序，
     * 因此我们通过自己定义这些类型来解决这个问题，因为一些文件使用它们。
     */
    typedef unsigned char uint8_t;
    typedef signed char int8_t;
    typedef unsigned short uint16_t;
    typedef signed short int16_t;
    typedef unsigned int uint32_t;
    typedef signed int int32_t;
    #ifdef _MSC_EXTENSIONS
      typedef unsigned _int64 uint64_t;
      typedef _int64 int64_t;
    #else /* _MSC_EXTENSIONS */
      typedef unsigned long long uint64_t;
      typedef long long int64_t;
    #endif
  #endif
#else /* defined(_MSC_VER) */
  /*
   * 如果不是使用 Visual Studio 编译器。
   * 包含 <inttypes.h> 以获取整数类型和 PRi[doux]64 值的定义。
   *
   * 如果编译器是 MinGW，我们假设有 <inttypes.h> - 并且支持格式化打印函数中的 %zu。
   *
   * 如果目标是 UN*X，我们假设有一个 C99 或更新的开发环境，因此有 <inttypes.h> - 并且支持格式化打印函数中的 %zu。
   *
   * 如果目标是 MS-DOS，我们假设有 <inttypes.h> - 并且支持格式化打印函数中的 %zu。
   *
   * 即，假设我们有 <inttypes.h> 并且它足够。
   */

  /*
   * XXX - 如何确保其他编译器和支持库具有足够的 C99 支持？
   */

  #include <inttypes.h>
#endif /* defined(_MSC_VER) */

#endif /* pcap/pcap-inttypes.h */
```