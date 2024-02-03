# `nmap\libpcap\ftmacros.h`

```cpp
/*
 * 版权声明，版权归加利福尼亚大学理事会所有
 * 允许在源代码和二进制形式下进行再发布和使用，无论是否经过修改
 * 需满足以下条件：
 * 1. 源代码的再发布必须保留上述版权声明、条件列表和以下免责声明
 * 2. 二进制形式的再发布必须在文档和/或其他提供的材料中重现上述版权声明、条件列表和以下免责声明
 * 3. 所有提及此软件特性或使用的广告材料必须显示以下声明：
 *    本产品包含由劳伦斯伯克利实验室计算机系统工程组开发的软件
 * 4. 未经特定事先书面许可，不得使用大学或实验室的名称来认可或推广从本软件衍生的产品
 *
 * 本软件由理事会和贡献者“按原样”提供，不提供任何明示或暗示的担保，包括但不限于对适销性和特定用途的暗示担保
 * 在任何情况下，理事会或贡献者均不对任何直接、间接、附带、特殊、惩罚性或后果性的损害承担责任
 * 包括但不限于替代商品或服务的采购、使用、数据或利润的损失，或业务中断
 * 无论是合同责任、严格责任还是侵权行为（包括疏忽或其他方式）引起的任何责任，即使已被告知可能发生此类损害
 */
#ifndef ftmacros_h
#define    ftmacros_h
/*
 * 定义一些特性测试宏，以确保我们想要声明的所有内容都被声明。
 *
 * 在一些 UNIX 系统上，我们需要强制声明 strtok_r()。
 * 我们*不*想定义 _POSIX_C_SOURCE，因为这往往会使我们使用的非 POSIX API 不可用。
 * XXX - 是否没有一种可移植的方法来说“请尽可能污染命名空间”？
 */
#if defined(sun) || defined(__sun)
  /*
   * 在 Solaris 上，Clang 自动定义 __EXTENSIONS__。
   */
  #ifndef __EXTENSIONS__
    #define __EXTENSIONS__
  #endif

  /*
   * 我们还需要定义 _XPG4_2 以获得
   * 使用 recvmsg() 的 Single UNIX Specification 版本。
   */
  #define _XPG4_2
#elif defined(_hpux) || defined(hpux) || defined(__hpux)
  #define _REENTRANT

  /*
   * 我们需要这个来获得使用 socklen_t 的套接字函数的版本。只有在它没有被定义的情况下定义它，这样我们就不会得到重定义警告。
   */
  #ifndef _XOPEN_SOURCE_EXTENDED
    #define _XOPEN_SOURCE_EXTENDED
  #endif

  /*
   * XXX - GCC 的 PA-RISC 选项列表让人觉得构建使用特定版本的 UNIX API/ABI 的代码很复杂：
   *
   *    https://gcc.gnu.org/onlinedocs/gcc/HPPA-Options.html
   *
   * 请参阅 -munix 标志的描述。
   *
   * 我们可能希望 libpcap 能够与为任何 UNIX 标准构建的程序一起工作。我不确定这是否可能，如果可能的话，它需要做什么样的工作。
   *
   * 这可能还需要我们使用特殊标志构建库以允许它与线程化代码一起使用，至少在使用 HP 的 C 编译器时；希望这样做不会使它*不*能与*非*线程化代码一起工作。
   */
#else
  /*
   * 打开 _GNU_SOURCE 以获取 GNU libc 提供的所有内容，
   * 包括 asprintf()，如果我们使用的是 GNU libc。
   *
   * 不幸的是，它提供的一件事是一个不符合 POSIX 标准的 strerror_r()，
   * 但我们在 pcap_fmt_errmsg_for_errno() 中处理这个问题。
   *
   * 我们不将其限制为，例如，Linux 和 Cygwin，因为这可能是 GNU/HURD 或 Debian 的 kFreeBSD 之一
   * 操作系统（"GNU/FreeBSD"）。
   */
  #define _GNU_SOURCE

  /*
   * 我们同时打开 _DEFAULT_SOURCE 和 _BSD_SOURCE 来尝试获取 BSD 的 u_XXX 类型，比如 u_int 和 u_short 的定义。
   * 我们首先定义 _DEFAULT_SOURCE，这样 GNU libc 的新版本就不会抱怨 _BSD_SOURCE 被弃用；
   * 我们仍然必须定义 _BSD_SOURCE 来处理不支持 _DEFAULT_SOURCE 的旧版本的 GNU libc。
   *
   * 但是，如果它已经被定义了，就不要再定义它，这样我们就不会得到一个重新定义的警告，如果它被定义为，例如，1。
   */
  #ifndef _DEFAULT_SOURCE
    #define _DEFAULT_SOURCE
  #endif
  /* 如果 _BSD_SOURCE 已经被定义为例如 1，避免重新定义它 */
  #ifndef _BSD_SOURCE
    #define _BSD_SOURCE
  #endif
#endif

#endif
```