# `nmap\libpcap\pcap\socket.h`

```
/*
 * 设置 C 语言的编辑模式为 c
 * 设置 tab 宽度为 8
 * 设置缩进模式为 1
 * 设置基本缩进为 8
 */
/* 
 * 版权声明，版权归加利福尼亚大学理事会所有
 * 允许在源代码和二进制形式下进行再发布和使用，需满足以下条件：
 * 1. 源代码的再发布必须保留上述版权声明、条件列表和以下免责声明
 * 2. 二进制形式的再发布必须在文档和/或其他提供的材料中重现上述版权声明、条件列表和以下免责声明
 * 3. 所有提及此软件特性或使用的广告材料必须显示以下声明：
 *    本产品包含由劳伦斯伯克利实验室计算机系统工程组开发的软件
 * 4. 未经特定事先书面许可，不得使用大学或实验室的名称来认可或推广从本软件衍生的产品
 * 
 * 本软件由理事会和贡献者“按原样”提供，不提供任何明示或暗示的担保，包括但不限于对适销性和特定用途的暗示担保
 * 无论在任何情况下，理事会或贡献者均不对任何直接、间接、附带、特殊、惩罚性或后果性的损害（包括但不限于替代商品或服务的采购、使用、数据或利润的损失或业务中断）承担责任
 * 无论是在合同、严格责任或侵权行为（包括疏忽或其他方式）的任何责任理论下，即使已被告知可能发生此类损害，也不承担任何责任
 */
#ifndef lib_pcap_socket_h
#define lib_pcap_socket_h
/*
 * 一些不同平台上套接字的细微差异。
 * 我们包括在 UN*X 和 Windows 上需要用于 Internet 协议套接字访问的套接字。
 */
#ifdef _WIN32
  /* 在 MingW32 下需要 windef.h 来定义 winsock2.h 中使用的定义 */
  #ifdef __MINGW32__
    #include <windef.h>
  #endif
  #include <winsock2.h>
  #include <ws2tcpip.h>

  /*
   * Winsock 没有这个 POSIX 类型；它用于 struct timeval 的 tv_usec 值。
   */
  typedef long suseconds_t;
#else /* _WIN32 */
  #include <sys/types.h>
  #include <sys/socket.h>
  #include <netdb.h>        /* 用于 struct addrinfo/getaddrinfo() */
  #include <netinet/in.h>    /* 在 BSD 中用于 sockaddr_in */
  #include <arpa/inet.h>

  /*!
   * \brief 在 Winsock 中，套接字句柄的类型是 SOCKET；在 UN*X 中，它是
   * 文件描述符，因此是有符号整数。
   * 我们在 UN*X 上将 SOCKET 定义为有符号整数，以便它可以
   * 在两个平台上使用。
   */
  #ifndef SOCKET
    #define SOCKET int
  #endif

  /*!
   * \brief 在 Winsock 中，如果 socket() 失败，错误返回是 INVALID_SOCKET；
   * 在 UN*X 中，是 -1。
   * 我们在 UN*X 上将 INVALID_SOCKET 定义为 -1，以便它可以在
   * 两个平台上使用。
   */
  #ifndef INVALID_SOCKET
    #define INVALID_SOCKET -1
  #endif
#endif /* _WIN32 */

#endif /* lib_pcap_socket_h */
```