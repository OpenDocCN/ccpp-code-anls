# Nmap源码解析 7

# `TargetGroup.h`

This text is the official documentation for the Nmap network vulnerability scanner. It explains who is allowed to use Nmap, what the Nmap license terms are, and what the implications are for using Nmap in commercial products.

It also explains the Nmap OEM Edition, which has a more permissive license and special features. Additionally, it provides links to the Npcap software for packet capture and transmission. The Npcap software is under a separate license term that prohibits redistribution without special permission.

Finally, it provides information about the Nmap Public Source License, which allows users to port Nmap to new platforms, fix bugs, and add new features. Users are encouraged to submit their changes as a Github PR or by email to the dev@nmap.org mailing list for possible incorporation into the main distribution.


```cpp

/***************************************************************************
 * TargetGroup.h -- The "TargetGroup" class holds a group of IP addresses, *
 * such as those from a '/16' or '10.*.*.*' specification.  It also has a  *
 * trivial HostGroupState class which handles a bunch of expressions that  *
 * go into TargetGroup classes.                                            *
 *                                                                         *
 ***********************IMPORTANT NMAP LICENSE TERMS************************
 *
 * The Nmap Security Scanner is (C) 1996-2023 Nmap Software LLC ("The Nmap
 * Project"). Nmap is also a registered trademark of the Nmap Project.
 *
 * This program is distributed under the terms of the Nmap Public Source
 * License (NPSL). The exact license text applying to a particular Nmap
 * release or source code control revision is contained in the LICENSE
 * file distributed with that version of Nmap or source code control
 * revision. More Nmap copyright/legal information is available from
 * https://nmap.org/book/man-legal.html, and further information on the
 * NPSL license itself can be found at https://nmap.org/npsl/ . This
 * header summarizes some key points from the Nmap license, but is no
 * substitute for the actual license text.
 *
 * Nmap is generally free for end users to download and use themselves,
 * including commercial use. It is available from https://nmap.org.
 *
 * The Nmap license generally prohibits companies from using and
 * redistributing Nmap in commercial products, but we sell a special Nmap
 * OEM Edition with a more permissive license and special features for
 * this purpose. See https://nmap.org/oem/
 *
 * If you have received a written Nmap license agreement or contract
 * stating terms other than these (such as an Nmap OEM license), you may
 * choose to use and redistribute Nmap under those terms instead.
 *
 * The official Nmap Windows builds include the Npcap software
 * (https://npcap.com) for packet capture and transmission. It is under
 * separate license terms which forbid redistribution without special
 * permission. So the official Nmap Windows builds may not be redistributed
 * without special permission (such as an Nmap OEM license).
 *
 * Source is provided to this software because we believe users have a
 * right to know exactly what a program is going to do before they run it.
 * This also allows you to audit the software for security holes.
 *
 * Source code also allows you to port Nmap to new platforms, fix bugs, and add
 * new features. You are highly encouraged to submit your changes as a Github PR
 * or by email to the dev@nmap.org mailing list for possible incorporation into
 * the main distribution. Unless you specify otherwise, it is understood that
 * you are offering us very broad rights to use your submissions as described in
 * the Nmap Public Source License Contributor Agreement. This is important
 * because we fund the project by selling licenses with various terms, and also
 * because the inability to relicense code has caused devastating problems for
 * other Free Software projects (such as KDE and NASM).
 *
 * The free version of Nmap is distributed in the hope that it will be
 * useful, but WITHOUT ANY WARRANTY; without even the implied warranty of
 * MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. Warranties,
 * indemnification and commercial support are all available through the
 * Npcap OEM program--see https://nmap.org/oem/
 *
 ***************************************************************************/

```

这段代码定义了一个名为 TargetGroup 的类，它是一个网络中的一个目标组。TargetGroup 类包含一个指向 NetBlock 结构的成员变量 netblock，用于存储目标组的网络地址信息。

TargetGroup 类包含三个析构函数，分别用于初始化、清理和重置对象。其中，初始化函数接收一个目标表达式，并将其解析为 IP 地址或者 IP 地址范围，然后将 netblock 初始化为该目标地址对应的 NetBlock。清理函数用于释放目标组对象中已经分配的内存，并将其设置为 NULL。重置函数则是清理函数，但不会释放之前分配的内存，而是将其设置为 NULL。

TargetGroup 类还包括三个函数，用于获取目标组中下一个要分配的地址、获取指定地址对应的目标组是否已经被创建过、以及获取指定地址对应的目标组中的所有未扫描过的地址。

总体来说，这段代码定义了一个 TargetGroup 类，用于表示网络中的目标组，以及其相关的操作。


```cpp
/* $Id$ */

#ifndef TARGETGROUP_H
#define TARGETGROUP_H

#include <list>
#include <cstddef>

class NetBlock;

class TargetGroup {
public:
  NetBlock *netblock;

  TargetGroup() {
    this->netblock = NULL;
  }

  ~TargetGroup();

  /* Initializes (or reinitializes) the object with a new expression,
     such as 192.168.0.0/16 , 10.1.0-5.1-254 , or
     fe80::202:e3ff:fe14:1102 .  The af parameter is AF_INET or
     AF_INET6 Returns 0 for success */
  int parse_expr(const char *target_expr, int af);
  /* Grab the next host from this expression (if any).  Returns 0 and
     fills in ss if successful.  ss must point to a pre-allocated
     sockaddr_storage structure */
  int get_next_host(struct sockaddr_storage *ss, std::size_t *sslen);
  /* Returns true iff the given address is the one that was resolved to create
     this target group; i.e., not one of the addresses derived from it with a
     netmask. */
  bool is_resolved_address(const struct sockaddr_storage *ss) const;
  /* Return a string of the name or address that was resolved for this group. */
  const char *get_resolved_name(void) const;
  /* Return the list of addresses that the name for this group resolved to, but
     which were not scanned, if it came from a name resolution. */
  const std::list<struct sockaddr_storage> &get_unscanned_addrs(void) const;
  /* is the current expression a named host */
  int get_namedhost() const;
};

```

这段代码是一个预处理指令，它 checking 并且必须得保证在预处理器中定义了目标分组头文件名称 TARGETGROUP_H。如果预处理器中没有定义该文件名，该指令将引发编译错误。

简单来说，该代码是为了确保在编译之前，目标分组头文件 TARGETGROUP_H 已经定义，否则引发错误。


```cpp
#endif /* TARGETGROUP_H */


```

# `targets.h`

This is a text segment describing the Nmap software and its licensing. Nmap is a network scanner that allows users to scan for open ports on a target system. The licensing for Nmap is discussed in the segment.

The first part of the text segment explains that Nmap is released under the Nmap license. This license prohibits companies from using the software in commercial products, but allows for the special Nmap OEM edition to be used in commercial products with the license agreement or contract from the Nmap team. The Nmap team can sell an OEM edition with more permissive terms for the purpose of commercial use.

The second part of the text segment explains the terms of the Nmap license. The official Nmap Windows builds include the Npcap software for packet capture and transmission. The Npcap software is under a different license term that prohibits redistribution without special permission.

The third part of the text segment explains the Nmap source code policy. The source code is made available to users because it allows them to know what the software is doing and to audit it for security holes. The source code can also be ported to new platforms, fix bugs, and add new features. The Nmap team encourages users to submit their changes as a Github PR or by email to the dev@nmap.org mailing list for possible incorporation into the main distribution.

The fourth part of the text segment explains the Nmap Public Source License Contributor Agreement. The license allows users to use the software under certain terms, but also gives the Nmap team broad rights to use the software as described in the Nmap Public Source License Contributor Agreement. The Nmap team uses the funding from the sales of licenses to fund the project.

The fifth part of the text segment explains the limitations of the Nmap license. The free version of Nmap is distributed in the hope that it will be useful, but WITHOUT ANY WARRANTY; without even the implied warranty of merchantability or fitness for a particular purpose. Warranties, indemnification, and commercial support are all available through the Npcap OEM program.


```cpp

/***************************************************************************
 * targets.h -- Functions relating to "ping scanning" as well as           *
 * determining the exact IPs to hit based on CIDR and other input formats. *
 *                                                                         *
 ***********************IMPORTANT NMAP LICENSE TERMS************************
 *
 * The Nmap Security Scanner is (C) 1996-2023 Nmap Software LLC ("The Nmap
 * Project"). Nmap is also a registered trademark of the Nmap Project.
 *
 * This program is distributed under the terms of the Nmap Public Source
 * License (NPSL). The exact license text applying to a particular Nmap
 * release or source code control revision is contained in the LICENSE
 * file distributed with that version of Nmap or source code control
 * revision. More Nmap copyright/legal information is available from
 * https://nmap.org/book/man-legal.html, and further information on the
 * NPSL license itself can be found at https://nmap.org/npsl/ . This
 * header summarizes some key points from the Nmap license, but is no
 * substitute for the actual license text.
 *
 * Nmap is generally free for end users to download and use themselves,
 * including commercial use. It is available from https://nmap.org.
 *
 * The Nmap license generally prohibits companies from using and
 * redistributing Nmap in commercial products, but we sell a special Nmap
 * OEM Edition with a more permissive license and special features for
 * this purpose. See https://nmap.org/oem/
 *
 * If you have received a written Nmap license agreement or contract
 * stating terms other than these (such as an Nmap OEM license), you may
 * choose to use and redistribute Nmap under those terms instead.
 *
 * The official Nmap Windows builds include the Npcap software
 * (https://npcap.com) for packet capture and transmission. It is under
 * separate license terms which forbid redistribution without special
 * permission. So the official Nmap Windows builds may not be redistributed
 * without special permission (such as an Nmap OEM license).
 *
 * Source is provided to this software because we believe users have a
 * right to know exactly what a program is going to do before they run it.
 * This also allows you to audit the software for security holes.
 *
 * Source code also allows you to port Nmap to new platforms, fix bugs, and add
 * new features. You are highly encouraged to submit your changes as a Github PR
 * or by email to the dev@nmap.org mailing list for possible incorporation into
 * the main distribution. Unless you specify otherwise, it is understood that
 * you are offering us very broad rights to use your submissions as described in
 * the Nmap Public Source License Contributor Agreement. This is important
 * because we fund the project by selling licenses with various terms, and also
 * because the inability to relicense code has caused devastating problems for
 * other Free Software projects (such as KDE and NASM).
 *
 * The free version of Nmap is distributed in the hope that it will be
 * useful, but WITHOUT ANY WARRANTY; without even the implied warranty of
 * MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. Warranties,
 * indemnification and commercial support are all available through the
 * Npcap OEM program--see https://nmap.org/oem/
 *
 ***************************************************************************/

```

这段代码定义了一个名为 Target 的类，它是一个实现了类似 MySQL 的关系数据库引擎的目标驱动程序。这个类定义了一个 Target 类，用于存储目标的数据。这个类的实例化需要一个主机GroupState 类的实例作为参数，然后通过构造函数来设置主机GroupState 实例的一些参数。

HostGroupState 类用于管理主机批次队列。它包含一个 defer_buffer 列表，用于存储之前已经返回但是不能立即使用的目标。这些目标在 defer 方法中等待移动到目标队列的末尾。这个类的实例也包含一个 current_batch_sz 变量，用于跟踪主机batch 数组的大小，以及一个 next_batch_no 变量，用于跟踪下一次将如何获取目标。

整段代码的作用是定义了一个 Target 类，用于实现主机批次队列的管理，这个类的实例化需要一个 HostGroupState 类的实例作为参数，然后通过构造函数来设置 HostGroupState 实例的一些参数。


```cpp
/* $Id$ */

#ifndef TARGETS_H
#define TARGETS_H

#include "TargetGroup.h"
#include <list>
#include <nbase.h>
class Target;

class HostGroupState {
public:
  /* The maximum number of entries we want to allow storing in defer_buffer. */
  static const unsigned int DEFER_LIMIT = 64;

  HostGroupState(int lookahead, int randomize, int argc, const char *argv[]);
  ~HostGroupState();
  Target **hostbatch;

  /* The defer_buffer is a place to store targets that have previously been
     returned but that can't be used right now. They wait in defer_buffer until
     HostGroupState::undefer is called, at which point they all move to the end
     of the undeferred list. HostGroupState::next_target always pulls from the
     undeferred list before returning anything new. */
  std::list<Target *> defer_buffer;
  std::list<Target *> undeferred;

  int argc;
  const char **argv;
  int max_batch_sz; /* The size of the hostbatch[] array */
  int current_batch_sz; /* The number of VALID members of hostbatch[] */
  int next_batch_no; /* The index of the next hostbatch[] member to be given
                        back to the user */
  int randomize; /* Whether each batch should be "shuffled" prior to the ping
                    scan (they will also be out of order when given back one
                    at a time to the client program */
  TargetGroup current_group; /* For batch chunking -- targets in queue */

  /* Returns true iff the defer buffer is not yet full. */
  bool defer(Target *t);
  void undefer();
  const char *next_expression();
  Target *next_target();
};

```

这段代码定义了一个名为 Target 的结构体，用于传递用于主机发现的目标主机组的状态信息。它还实现了一些用于处理排除列表的函数，并定义了一个用于输出排除列表的例行。

具体来说，这段代码主要实现了以下功能：

1.定义了一个 Target 结构体，其中包含用于主机发现的目标主机组的状态信息。

2.实现了一个名为nexthost的函数，用于获取下一个主机，并在排除列表中查找它。它接受一个 HostGroupState 类型的参数 hs 和一个排除列表的指针 exclude_group，以及用于发现目标的主机列表的指针 ports 和某种类型的ping类型。

3.实现了一个名为 load_exclude_file 和 load_exclude_string 的函数，它们分别用于加载排除列表文件和字符串。

4.实现了一个名为 dumpExclude 的函数，用于将排除列表内容输出到 stdout。

5.实现了一个名为 returnhost 的函数，用于在 nexthost 返回主机失败时返回默认的主机。

6.实现了一个名为 target_needs_new_hostgroup 的函数，用于检查是否需要创建新的主机组。

7.最后，在 nexthost.c 中定义了一个名为 targets 的变量，用于保存目标主机组。

综上所述，这段代码的主要作用是实现了一个用于主机发现的目标主机组及其相关功能。


```cpp
/* ports is used to pass information about what ports to use for host discovery */
Target *nexthost(HostGroupState *hs, struct addrset *exclude_group,
                 const struct scan_lists *ports, int pingtype);
int load_exclude_file(struct addrset *exclude_group, FILE *fp);
int load_exclude_string(struct addrset *exclude_group, const char *s);
/* a debugging routine to dump an exclude list to stdout. */
int dumpExclude(const struct addrset *exclude_group);
/* Returns the last host obtained by nexthost.  It will be given again the next
   time you call nexthost(). */
void returnhost(HostGroupState *hs);


bool target_needs_new_hostgroup(Target **targets, int targets_sz, const Target *target);

#endif /* TARGETS_H */


```

# `tcpip.h`

This text is a part of the Nmap software license agreement. Nmap is a network scanner that allows users to scan networks and detect various information about the hosts that are running the scan.

The text discusses the limitations and restrictions of using Nmap in commercial products. It states that companies that have received Nmap licenses are prohibited from using the software in such products, except for a special Nmap OEM edition.

The text also mentions the licensing options for Nmap. If a company wants to use Nmap in a commercial product, they must first obtain an Nmap OEM license, which allows them to use the software for commercial purposes. However, this license does not give them the right to redistribute the software.

The text also states that if a company has received an Nmap license agreement or contract that includes terms other than those of the Nmap Public Source License, they can use and redistribute Nmap under those terms. However, if the software is redistributed, the company must have the special Nmap OEM Edition license and obtain special permission from Nmap.

The text also mentions the Npcap software that is included in the official Nmap Windows builds. This software is for packet capture and transmission and has separate license terms that prohibit redistribution without special permission.

Finally, the text advises users to check the Nmap Public Source License to know exactly what they are allowed to do with the software and to audit it for security holes. The text also encourages users to submit their changes as a Github PR or by email to the dev@nmap.org mailing list for possible incorporation into the main distribution.


```cpp

/***************************************************************************
 * tcpip.h -- Various functions relating to low level TCP/IP handling,     *
 * including sending raw packets, routing, printing packets, reading from  *
 * libpcap, etc.                                                           *
 *                                                                         *
 ***********************IMPORTANT NMAP LICENSE TERMS************************
 *
 * The Nmap Security Scanner is (C) 1996-2023 Nmap Software LLC ("The Nmap
 * Project"). Nmap is also a registered trademark of the Nmap Project.
 *
 * This program is distributed under the terms of the Nmap Public Source
 * License (NPSL). The exact license text applying to a particular Nmap
 * release or source code control revision is contained in the LICENSE
 * file distributed with that version of Nmap or source code control
 * revision. More Nmap copyright/legal information is available from
 * https://nmap.org/book/man-legal.html, and further information on the
 * NPSL license itself can be found at https://nmap.org/npsl/ . This
 * header summarizes some key points from the Nmap license, but is no
 * substitute for the actual license text.
 *
 * Nmap is generally free for end users to download and use themselves,
 * including commercial use. It is available from https://nmap.org.
 *
 * The Nmap license generally prohibits companies from using and
 * redistributing Nmap in commercial products, but we sell a special Nmap
 * OEM Edition with a more permissive license and special features for
 * this purpose. See https://nmap.org/oem/
 *
 * If you have received a written Nmap license agreement or contract
 * stating terms other than these (such as an Nmap OEM license), you may
 * choose to use and redistribute Nmap under those terms instead.
 *
 * The official Nmap Windows builds include the Npcap software
 * (https://npcap.com) for packet capture and transmission. It is under
 * separate license terms which forbid redistribution without special
 * permission. So the official Nmap Windows builds may not be redistributed
 * without special permission (such as an Nmap OEM license).
 *
 * Source is provided to this software because we believe users have a
 * right to know exactly what a program is going to do before they run it.
 * This also allows you to audit the software for security holes.
 *
 * Source code also allows you to port Nmap to new platforms, fix bugs, and add
 * new features. You are highly encouraged to submit your changes as a Github PR
 * or by email to the dev@nmap.org mailing list for possible incorporation into
 * the main distribution. Unless you specify otherwise, it is understood that
 * you are offering us very broad rights to use your submissions as described in
 * the Nmap Public Source License Contributor Agreement. This is important
 * because we fund the project by selling licenses with various terms, and also
 * because the inability to relicense code has caused devastating problems for
 * other Free Software projects (such as KDE and NASM).
 *
 * The free version of Nmap is distributed in the hope that it will be
 * useful, but WITHOUT ANY WARRANTY; without even the implied warranty of
 * MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. Warranties,
 * indemnification and commercial support are all available through the
 * Npcap OEM program--see https://nmap.org/oem/
 *
 ***************************************************************************/

```

这段代码定义了一个名为"TCPIP_H"的标识，表示这是一个关于TCP/IP头文件的标准定义。它接着定义了一些头文件和变量，然后引入了<pcap.h>头文件，以便使用pcap函数。接下来它定义了一个名为Target的类，并定义了一个名为"目标"的变量，其类型未被定义。最后，它定义了一个名为"TCPIP_StreamType"的枚举类型，其值为0。


```cpp
/* $Id$ */


#ifndef TCPIP_H
#define TCPIP_H

#include "nbase.h"

#include <pcap.h>

class Target;

#ifndef INET_ADDRSTRLEN
#define INET_ADDRSTRLEN 16
#endif

```

这段代码定义了一个名为 ` PacketTrace` 的类，它包含了一个名为 `trace` 的函数。这个函数接受一个 `pdirection` 类型参数，表示数据包发送方向为 `SENT` 或 `RCVD`。

如果 `packet` 参数是一个 IPv4 数据报，函数会打印出该数据报的内容。如果 `packet` 参数是一个指向 IPv4 数据报的指针，并且 `packet` 参数的类型是 `SENT` 或 `RCVD`，则函数会打印出数据报的摘要信息，包括数据部分的长度 `len` 和时间戳 `now`（可选）。

函数的实现比较简单，直接根据输入参数的类型来判断数据包的发送方向，然后调用一个名为 `trace` 的函数来打印数据包的内容。如果 `now` 参数被传递，可以避免调用 `gettimeofday()` 函数，提高函数的效率。


```cpp
int nmap_raw_socket();

/* Used for tracing all packets sent or received (eg the
   --packet-trace option) */

class PacketTrace {
 public:
  static const int SENT=1; /* These two values must not be changed */
  static const int RCVD=2;
  typedef int pdirection;
  /* Takes an IP PACKET and prints it if packet tracing is enabled.
     'packet' must point to the IPv4 header. The direction must be
     PacketTrace::SENT or PacketTrace::RCVD .  Optional 'now' argument
     makes this function slightly more efficient by avoiding a gettimeofday()
     call. */
  static void trace(pdirection pdir, const u8 *packet, u32 len,
                    struct timeval *now=NULL);
```

这段代码定义了两个函数：traceConnect和traceArp。这两个函数在尝试连接（有或无协议跟踪）时，如果启用了包跟踪，则在尝试的连接中添加跟踪条目。

frame是一个ARP包（包括以太网头）的结构体，其中包含一个指向14字节以太网头的心跳（也可以是数据分片）。变量len表示ARP包的长度。

connectrc是用于获取尝试连接的返回代码的整数。如果返回代码为-1，函数将获取errno并将其作为connect_errno传递给函数调用者。

timeval结构体用于存储当前时间，如果启用包跟踪，则函数可以使用timeval来避免使用gettimeofday()函数。

static void traceConnect(u8 proto, const struct sockaddr *sock,
                          int socklen, int connectrc, int connect_errno,
                          const struct timeval *now);
static void traceArp(pirection pdir, const u8 *frame, u32 len,
                                   struct timeval *now);
static void traceND(pirection pdir, const u8 *frame, u32 len,
                                   struct timeval *now);

上述函数是在Linux系统中的网络接口上跟踪数据包传输情况时需要用到的。


```cpp
/* Adds a trace entry when a connect() is attempted if packet tracing
   is enabled.  Pass IPPROTO_TCP or IPPROTO_UDP as the protocol.  The
   sock may be a sockaddr_in or sockaddr_in6.  The return code of
   connect is passed in connectrc.  If the return code is -1, get the
   errno and pass that as connect_errno. */
  static void traceConnect(u8 proto, const struct sockaddr *sock,
                           int socklen, int connectrc, int connect_errno,
                           const struct timeval *now);
  /* Takes an ARP PACKET (including ethernet header) and prints it if
     packet tracing is enabled.  'frame' must point to the 14-byte
     ethernet header (e.g. starting with destination addr). The
     direction must be PacketTrace::SENT or PacketTrace::RCVD .
     Optional 'now' argument makes this function slightly more
     efficient by avoiding a gettimeofday() call. */
  static void traceArp(pdirection pdir, const u8 *frame, u32 len,
                                    struct timeval *now);
  static void traceND(pdirection pdir, const u8 *frame, u32 len,
                                    struct timeval *now);
};

```

这段代码定义了一个名为 PacketCounter 的类，用于统计数据包的发送和接收情况。它包括四个成员变量：sendPackets、sendBytes、recvPackets 和 recvBytes，分别表示数据包发送的数量、字节数和接收的数量、字节数。

在构造函数中，该类成员变量被初始化为 0。

该类使用了 Windows API，其中定义了 IPPROTO_IGMP 枚举类型，表示 IGMP（Internet Protocol for Buring Accounting Message Encapsulation）协议版本 2。因此，该代码在支持 IGMP 协议的 Windows 系统中运行。


```cpp
class PacketCounter {
 public:
  PacketCounter() : sendPackets(0), sendBytes(0), recvPackets(0), recvBytes(0) {}
#if WIN32
  unsigned __int64
#else
  unsigned long long
#endif
          sendPackets, sendBytes, recvPackets, recvBytes;
};


/* Some systems might not have this */
#ifndef IPPROTO_IGMP
#define IPPROTO_IGMP 2
```

这段代码定义了两个函数，以及一个名为其他地方的头文件。接下来，我会逐步解释每个函数的作用。

1. `inet_socktop` 函数：

该函数接收一个 `sockaddr_storage` 类型的结构体，包含一个IP地址。函数的作用是将IP地址转换为 IPv4 或 IPv6 格式的字符串。由于返回的是一个静态缓冲区，这个函数只能在调用一次之后保持其有效性。你可以将其视为一个安全性的妥协，因为缓冲区只能在使用时进行一次初始化。

2. `resolve_all` 函数：

该函数接收一个字符串 `hostname` 和一个整数 `pf`。函数的作用是尝试将 `hostname` 解析为 IPv4 或 IPv6 格式的字符串，并返回一个指向 `addrinfo` 结构体的指针数组。如果解析失败或无法完成解析，函数将返回 NULL。这个函数允许你尝试解析一个主机名并获取其 IPv4 或 IPv6 地址。

这里 `resolve_all` 函数的作用是将一个字符串解析为 IPv4 或 IPv6 地址。它通过调用 `getaddrinfo` 函数获取目标主机名对应的 IPv4 或 IPv6 地址。返回的地址存储在一个名为 `res` 的指针中。`getaddrinfo` 函数在传递给 `resolve_all` 函数的第二个参数中指定要解析的 IP 类型（IPv4 或 IPv6）。你可以通过调用 `resolve_all` 函数多次来获取多个 IPv4 或 IPv6 地址。


```cpp
#endif

/* Prototypes */
/* Converts an IP address given in a sockaddr_storage to an IPv4 or
   IPv6 IP address string.  Since a static buffer is returned, this is
   not thread-safe and can only be used once in calls like printf()
*/
const char *inet_socktop(const struct sockaddr_storage *ss);

/* Tries to resolve the given name (or literal IP) into a sockaddr
   structure. This function calls getaddrinfo and returns the same
   addrinfo linked list that getaddrinfo produces. Returns NULL for any
   error or failure to resolve. */
struct addrinfo *resolve_all(const char *hostname, int pf);

```

这段代码是一个用于查找目标地址（dst）到目标地址（dst）的路由信息的函数。它实现了通过发送IP数据包并确定到达目标地址的源地址和接口，来实现查找路由信息的功能。如果找不到路由，函数将返回false并设置rnfo为undefined；如果找到了路由，函数将返回true，并将rnfo填充为所有路由详细信息。

具体来说，这段代码首先通过从struct sockaddr_storage类型的指针dst中获取目标地址，然后使用send_ip_packet函数发送一个预构建的IPv4或IPv6数据包。接着，代码使用struct eth_nfo类型的指针eth和struct sockaddr_storage类型的指针dst，将IP数据包的头部信息组装成一个新的IPv4或IPv6数据包，并将它发送到目标地址。如果成功发送数据包，函数将返回true，并将rnfo填充为包含所有路由详细信息。


```cpp
/* Takes a destination address (dst) and tries to determine the
   source address and interface necessary to route to this address.
   If no route is found, false is returned and rnfo is undefined.  If
   a route is found, true is returned and rnfo is filled in with all
   of the routing details.  This function takes into account -S and -e
   options set by user (o.spoofsource, o.device) */
int nmap_route_dst(const struct sockaddr_storage *dst, struct route_nfo *rnfo);

/* Send a pre-built IPv4 or IPv6 packet */
int send_ip_packet(int sd, const struct eth_nfo *eth,
  const struct sockaddr_storage *dst,
  const u8 *packet, unsigned int packetlen);

/* Builds an IP packet (including an IP header) by packing the fields
   with the given information.  It allocates a new buffer to store the
   packet contents, and then returns that buffer.  The packet is not
   actually sent by this function.  Caller must delete the buffer when
   finished with the packet.  The packet length is returned in
   packetlen, which must be a valid int pointer. */
```

这段代码定义了两个名为build_ip_raw和build_ipv6_raw的函数，用于构建IP数据包。

build_ip_raw函数的输入参数包括源IP地址[source]、目标IP地址[victim]、协议类型(protocol)和高时间戳(ttl)。函数返回一个IP数据包缓冲区(buffer)，其中包含构建好的IP头部。

build_ipv6_raw函数的输入参数包括源IPv6地址[source]、目标IPv6地址[victim]、隧道类型(the type of the tunnel)和拥塞控制(the congestion control)标志。函数返回一个IPv6数据包缓冲区(buffer)，其中包含构建好的IP头部。

这两个函数都是通过解包IP数据包头信息来构建IP数据包的。它们将根据提供的信息分配一个新的IP数据包缓冲区，其中包含构建好的IP头部。然后函数将返回这个缓冲区，以便用户在需要时重新使用它。


```cpp
u8 *build_ip_raw(const struct in_addr *source, const struct in_addr *victim,
                 u8 proto,
                 int ttl, u16 ipid, u8 tos, bool df,
                 const u8* ipopt, int ipoptlen,
                 const char *data, u16 datalen,
                 u32 *packetlen);

u8 *build_ipv6_raw(const struct in6_addr *source,
                   const struct in6_addr *victim, u8 tc, u32 flowlabel,
                   u8 nextheader, int hoplimit,
                   const char *data, u16 datalen, u32 *outpacketlen);

/* Builds a TCP packet (including an IP header) by packing the fields
   with the given information.  It allocates a new buffer to store the
   packet contents, and then returns that buffer.  The packet is not
   actually sent by this function.  Caller must delete the buffer when
   finished with the packet.  The packet length is returned in
   packetlen, which must be a valid int pointer. */
```

这两段代码都定义了一个名为`build_tcp_raw`的函数，该函数接受四个参数：

1. `const struct in_addr *source`：源IP地址的指针。
2. `const struct in_addr *victim`：目标IP地址的指针。
3. `int ttl`：IP时间戳（TTL）字段，以秒为单位。
4. `u16 ipid`：发送方IPID，用于标识发送方。
5. `u8 tos`：目标操作系统类型的字段，以4字节为单位。
6. `bool df`：禁止数据分片（DF）的标志，用于设置IP分片。
7. `const u8* ipopt`：目标IP选项，以4字节为单位。
8. `int ipoptlen`：目标IP选项的长度，以4字节为单位。
9. `u16 sport`：目标端口号。
10. `u16 dport`：目标数据端口号。
11. `u32 seq`：序列号字段，用于标识数据包中的每个分片。
12. `u32 ack`：确认号字段，用于标识接收到的数据分片。
13. `u8 reserved`：保留字段，保留。
14. `u8 flags`：标志字段，用于标识数据包中的分片类型。
15. `u16 window`：窗口字段，用于标识窗口大小。
16. `u16 urp`：URP字段，用于标识是否使用URP。
17. `const u8 *tcpopt`：目标TCP选项的指针。
18. `int tcpoptlen`：目标TCP选项的长度，以4字节为单位。
19. `const char *data`：目标数据。
20. `u16 datalen`：目标数据的长度，以字节为单位。
21. `u32 *packetlen`：用于标识数据包的长度。

这两段代码的目的是定义两个函数，都接受IPv6或IPv4地址，以及一些选项和标志，以帮助构建TCP数据包。第一个函数`build_tcp_raw`接受IPv6地址，第二个函数`build_tcp_raw_ipv6`接受IPv4地址。这两个函数都可以接受多个选项，包括IP选项、Sport、Dport、Window和URP等，以及数据和数据长度。通过使用这些选项，可以确保数据包正确发送并到达目标地址。


```cpp
u8 *build_tcp_raw(const struct in_addr *source, const struct in_addr *victim,
                  int ttl, u16 ipid, u8 tos, bool df,
                  const u8* ipopt, int ipoptlen,
                  u16 sport, u16 dport,
                  u32 seq, u32 ack, u8 reserved, u8 flags, u16 window, u16 urp,
                  const u8 *options, int optlen,
                  const char *data, u16 datalen,
                  u32 *packetlen);

u8 *build_tcp_raw_ipv6(const struct in6_addr *source,
                       const struct in6_addr *victim, u8 tc, u32 flowlabel,
                       u8 hoplimit, u16 sport, u16 dport, u32 seq, u32 ack,
                       u8 reserved, u8 flags, u16 window, u16 urp,
                       const u8 *tcpopt, int tcpoptlen, const char *data,
                       u16 datalen, u32 *packetlen);

```

这两段代码定义了一个名为`send_tcp_raw`的函数，其作用是构造并发送一个 raw TCP 数据包。函数接受一系列参数，包括传输层头信息（TTL）、数据源 IP 地址、目标 IP 地址、端口号、序列号、确认号、保留字段、窗口和无连接数据传输窗口（WINDOW）等。函数内部使用`eth_nfo`结构体来获取这些信息。

具体来说，这段代码实现以下功能：

1. 根据传入的参数，设置数据包的一些基本信息，包括TTL、数据源IP地址、目标IP地址、端口号、序列号等。

2. 如果需要使用数据传输窗口（WINDOW），则设置数据包的发送窗口和接收窗口。

3. 如果需要发送数据，则构造数据包的IP头部，其中使用`ipopt`参数传递IP选项数据，如果传递的IP选项数据长度超过20个字节，则对数据进行截断。

4. 发送数据包。

5. 如果需要发送带有伪装的数据包，则构造数据包的IP头部，其中使用`ipopt`参数传递IP选项数据，构造出来的数据包将伪装成目标IP地址的发送者。


```cpp
/* Build and send a raw tcp packet.  If TTL is -1, a partially random
   (but likely large enough) one is chosen */
int send_tcp_raw(int sd, const struct eth_nfo *eth,
                  const struct in_addr *source, const struct in_addr *victim,
                  int ttl, bool df,
                  u8* ipopt, int ipoptlen,
                  u16 sport, u16 dport,
                  u32 seq, u32 ack, u8 reserved, u8 flags, u16 window, u16 urp,
                  u8 *options, int optlen,
                  const char *data, u16 datalen);

int send_tcp_raw_decoys(int sd, const struct eth_nfo *eth,
                         const struct in_addr *victim,
                         int ttl, bool df,
                         u8* ipopt, int ipoptlen,
                         u16 sport, u16 dport,
                         u32 seq, u32 ack, u8 reserved, u8 flags, u16 window, u16 urp,
                         u8 *options, int optlen,
                         const char *data, u16 datalen);

```

这段代码定义了两个函数用于构建UDP数据报，一个IPv4版本，一个IPv6版本。IPv4版本主要负责构造IPv4数据报，IPv6版本主要负责构造IPv6数据报。两个函数都接受源地址、目标地址、TTL（Time to Live）和一些IP头部信息作为输入参数。

在函数内部，首先分配一块新的内存缓冲区用于存储数据报内容，然后将构建好的数据报内容复制到缓冲区。接着，设置数据报的一些标志，如DF（Don't fragment）等，并确保数据报长度为有效的32位整数。最后，将缓冲区返回给调用者。

调用者需要在函数使用完毕后，根据需要释放分配的内存。数据报的长度是在函数中计算出来的，需要确保是有效的32位整数。

需要注意的是，这些函数不会真正地将数据报发送出去。为了确保数据报正确构建并传递给下级函数，你应该在需要时手动发送数据报。


```cpp
/* Builds a UDP packet (including an IP header) by packing the fields
   with the given information.  It allocates a new buffer to store the
   packet contents, and then returns that buffer.  The packet is not
   actually sent by this function.  Caller must delete the buffer when
   finished with the packet.  The packet length is returned in
   packetlen, which must be a valid int pointer. */
u8 *build_udp_raw(const struct in_addr *source, const struct in_addr *victim,
       int ttl, u16 ipid, u8 tos, bool df,
                  u8* ipopt, int ipoptlen,
       u16 sport, u16 dport,
       const char *data, u16 datalen,
       u32 *packetlen);

u8 *build_udp_raw_ipv6(const struct in6_addr *source,
                       const struct in6_addr *victim, u8 tc, u32 flowlabel,
                       u8 hoplimit, u16 sport, u16 dport,
                       const char *data, u16 datalen, u32 *packetlen);

```

这两段代码都是用于构建SCTP（Stream Control Transmission Protocol，传输控制传输协议）数据报的函数。

`send_udp_raw`函数接收一个IP数据报，其中包含目标IP地址和端口号，然后使用IP选项（IPv4选项）填充数据报。首先，通过调用`send_udp`函数将数据报发送出去，然后在函数内部使用剩余的选项填充数据报。最后，将填充完数据报的内存区域返回给调用者。

`send_udp_raw_decoys`函数与`send_udp_raw`函数非常相似，只是使用了一些辅助函数。这个函数的参数列表具有以下共同点：

- `eth`：与IP数据报相关的结构体，包含了IP头部中的网络类型字段（如果存在的话）。
- `source`：目标IP地址的INA（Internet Address Allocation Network）表示。
- `victim`：目标IP地址的INA表示。
- `ttl`：IP数据报的生存时间（Time to Live，生存时间），主要用于确认数据报在网络中可以持续存在的时间。
- `ipid`：目标IP地址的下一个字节的索引，用于在数据报中指定目标IP。
- `ipopt`：一个INET_EXPIRATION_TIME字段的输入参数，用于指定数据报中包含保留时间（保留时间）字段的选项。
- `ip`：目标IP地址，用于在数据报中指定目标IP。
- `sport`：数据报的服务类型（服务类型字段，用于标识数据报的服务类型）。
- `dport`：数据报的协议类型（协议类型字段，用于标识数据报使用的协议类型）。
- `data`：数据，在数据报中传输的数据。
- `datalen`：数据报的长度，数据报中数据的长度。

这两个函数的主要区别在于它们的返回值类型。`send_udp_raw`函数返回一个INET_EXPIRATION_TIME类型的变量，而`send_udp_raw_decoys`函数返回一个包含INET_EXPIRATION_TIME类型的变量和另一个INET_EXPIRATION_TIME类型的变量的INET_EXPIRATION_TIME类型的指针。

这两段代码的作用是将IP数据报封装成SCTP数据报并发送。


```cpp
int send_udp_raw(int sd, const struct eth_nfo *eth,
                  struct in_addr *source, const struct in_addr *victim,
                  int ttl, u16 ipid,
                  u8* ipopt, int ipoptlen,
                  u16 sport, u16 dport,
                  const char *data, u16 datalen);

int send_udp_raw_decoys(int sd, const struct eth_nfo *eth,
                         const struct in_addr *victim,
                         int ttl, u16 ipid,
                         u8* ipops, int ip,
                         u16 sport, u16 dport,
                         const char *data, u16 datalen);

/* Builds an SCTP packet (including an IP header) by packing the fields
   with the given information.  It allocates a new buffer to store the
   packet contents, and then returns that buffer.  The packet is not
   actually sent by this function.  Caller must delete the buffer when
   finished with the packet.  The packet length is returned in
   packetlen, which must be a valid int pointer. */
```

这段代码定义了两个函数，分别名为`build_sctp_raw`和`build_sctp_raw_ipv6`。这两个函数的作用是创建一个IPv6数据报文，包括一个IP头部和一个SCTP数据报文头部。通过这两个函数，可以将数据 source 和 victim，TTL（Time to Live）和 IPID（Internet Protocol Identifier）也可以被传递进来。

具体来说，这两个函数都会先检查输入的地址是否是IPv6地址，如果是，则会将数据 source 和 victim，TTL 和 IPID 存储在一个结构体中，然后通过`ipopt`参数传递给`ipoptlen`参数。同时，这两个函数都会判断是否启用数据报文分片，如果是，则需要通过`chunks`参数指定每个数据分片的大小，以及通过`data`参数指定数据的总大小。

通过调用这两个函数，就可以创建一个IPv6数据报文，然后将其发送到目标地址。在函数内部，数据包的实际发送是由调用者负责的。


```cpp
u8 *build_sctp_raw(const struct in_addr *source, const struct in_addr *victim,
                   int ttl, u16 ipid, u8 tos, bool df,
                   u8* ipopt, int ipoptlen,
                   u16 sport, u16 dport,
                   u32 vtag, char *chunks, int chunkslen,
                   const char *data, u16 datalen,
                   u32 *packetlen);

u8 *build_sctp_raw_ipv6(const struct in6_addr *source,
                        const struct in6_addr *victim, u8 tc, u32 flowlabel,
                        u8 hoplimit, u16 sport, u16 dport, u32 vtag,
                        char *chunks, int chunkslen, const char *data, u16 datalen,
                        u32 *packetlen);

/* Builds an ICMP packet (including an IP header) by packing the
   fields with the given information.  It allocates a new buffer to
   store the packet contents, and then returns that buffer.  The
   packet is not actually sent by this function.  Caller must delete
   the buffer when finished with the packet.  The packet length is
   returned in packetlen, which must be a valid int pointer. The
   id/seq will be converted to network byte order (if it differs from
   HBO) */
```

这段代码定义了两个名为build_icmp_raw和build_icmpv6_raw的函数，它们用于构建IPICMP（Internet Control Information Protocol version 6）数据报。这些函数接受一个IP地址数组（in_addr结构体）和目标IP地址数组（in_addr结构体），以及IPID（接口ID）、TTL（生存时间），以及一个IP选项数组（ipopt结构体，ipoptlen为该数组长度）。

通过这两个函数，您可以将源IP地址和目标IP地址以及一些选项封装成一个数据报，这将通过IPICMP协议发送。数据报将包含一个IP头部，其中包含源和目标IP地址，以及TTL和0个或多个选项。

函数build_icmp_raw和build_icmpv6_raw仅在构建数据报时使用，它们不会发送数据报。您需要在调用完这些函数后，通过套接字或其他方式将数据报发送出去。


```cpp
u8 *build_icmp_raw(const struct in_addr *source, const struct in_addr *victim,
                   int ttl, u16 ipid, u8 tos, bool df,
                   u8* ipopt, int ipoptlen,
                   u16 seq, unsigned short id, u8 ptype, u8 pcode,
                   const char *data, u16 datalen, u32 *packetlen);

u8 *build_icmpv6_raw(const struct in6_addr *source,
                     const struct in6_addr *victim, u8 tc, u32 flowlabel,
                     u8 hoplimit, u16 seq, u16 id, u8 ptype, u8 pcode,
                     const char *data, u16 datalen, u32 *packetlen);

/* Builds an IGMP packet (including an IP header) by packing the fields
   with the given information.  It allocates a new buffer to store the
   packet contents, and then returns that buffer.  The packet is not
   actually sent by this function.  Caller must delete the buffer when
   finished with the packet.  The packet length is returned in packetlen,
   which must be a valid int pointer.
 */
```



这段代码是一个用于 build_igmp_raw 函数，其作用是将从本地计算机接收到的 IGMP 数据包 IP 包发送到目标计算机。以下是函数的详细解释：

1. 函数接收两个 IPv4 地址指针 source 和 victim，以及一个 IP 包 ID、一个 ToS 和一个数据链路协议类型(DF)，同时接收一个可选的输入数据指针 ipopt，表示数据链路协议类型。函数还接收一个输入数据包长度指针 ipopt_len，表示数据链路协议类型长度。

2. 函数首先判断输入的数据链路协议类型是否为 IGMP。如果是，函数将从本地计算机接收到的每个 IGMP 数据包 IP 查找目标计算机的下一个出口，并将数据包发送到目标计算机。

3. 如果输入的数据链路协议类型不是 IGMP，函数将忽略从本地计算机接收到的每个 IGMP 数据包 IP，并直接将数据包发送到目标计算机。

4. 函数使用 libpcap 获取从本地计算机接收到的每个数据包的读取时间值。如果函数返回 true，则认为获得的读取时间值是有效的，否则认为无效。

5. 函数使用 pcap_print_stats 函数打印数据包接收统计信息。

6. 函数使用 u8 *ipopt 存储输入的数据链路协议类型指针。

7. 函数使用 u8 *ipopt_len 存储输入的数据链路协议类型长度指针。

8. 函数使用 u16 *ipid 存储输入的 IP 包 ID。

9. 函数使用 u8 *tos 存储输入的 ToS。

10. 函数使用 u8 *df 存储输入的数据链路协议类型 DF。

11. 函数使用 u8 *ipopt 存储输入的数据链路协议类型指针。

12. 函数使用 int ipopt_len 存储输入的数据链路协议类型长度。

13. 函数使用 u16 *packetlen 存储输入的数据包长度。

14. 函数将输入的 IP 包发送到目标计算机。


```cpp
u8 *build_igmp_raw(const struct in_addr *source, const struct in_addr *victim,
                   int ttl, u16 ipid, u8 tos, bool df,
                   u8* ipopt, int ipoptlen,
                   u8 ptype, u8 pcode,
                   const char *data, u16 datalen, u32 *packetlen);


// Returns whether the packet receive time value obtained from libpcap
// (and thus by readip_pcap()) should be considered valid.  When
// invalid (Windows and Amiga), readip_pcap returns the time you called it.
bool pcap_recv_timeval_valid();

/* Prints stats from a pcap descriptor (number of received and dropped
   packets). */
void pcap_print_stats(int logt, pcap_t *pd);



```

这段代码定义了两个名为`readtcppacket`和`readudppacket`的函数，以及一个名为`getFinalPacketStats`的函数。它们的作用是帮助程序员更快地调试TCP数据包。

具体来说，`readtcppacket`函数接收一个TCP数据包的内存表示，并返回一个short类型的变量`buf`，该变量包含该数据包的统计信息，如源IP地址、目标IP地址、协议类型等。如果函数在尝试读取数据包时遇到问题，它将返回一个空字符串。

`readudppacket`函数与`readtcppacket`函数类似，但它的第二个参数是相反的，即一个short类型的变量`buf`，用于存储一个short类型的目标MAC地址。如果该函数在尝试读取数据包时遇到问题，它将返回一个空字符串。

`getFinalPacketStats`函数接收一个字符串类型的缓冲区和一个最大长度，用于存储数据包的统计信息。该函数使用一个short类型的变量`buflen`来确保缓冲区足够大以容纳所有必要的信息。如果函数在尝试读取数据包时遇到问题，它将返回一个空字符串。


```cpp
/* A simple function I wrote to help in debugging, shows the important fields
   of a TCP packet*/
int readtcppacket(const u8 *packet, int readdata);
int readudppacket(const u8 *packet, int readdata);

/* Fill buf (up to buflen -- truncate if necessary but always
   terminate) with a short representation of the packet stats.
   Returns buf.  Aborts if there is a problem. */
char *getFinalPacketStats(char *buf, int buflen);

/* This function tries to determine the target's ethernet MAC address
   from a received packet as follows:
   1) If linkhdr is an ethernet header, grab the src mac (otherwise give up)
   2) If overwrite is 0 and a MAC is already set for this target, give up.
   3) If the packet source address is not the target, give up.
   4) Use the routing table to try to determine rather target is
      directly connected to the src host running Nmap.  If it is, set the MAC.

   This function returns 0 if it ends up setting the MAC, nonzero otherwise
```

这两函数一起用于设置目标设备的下一个目标MAC地址。

函数setTargetMACIfAvailable的作用是为目标设备设置下一个的目标MAC地址。它需要一个目标指针、一个链头指针和一个源MAC地址。函数尝试在目标设备上查找下一个目标MAC地址，如果找不到，就尝试使用ARP缓存，如果仍然失败，就会发送一个ARP请求。

函数setTargetNextHopMAC的作用是获取目标设备的下一个目标MAC地址。它需要一个目标接口名称、一个源MAC地址、一个目标MAC地址和一个结构体存储目标设备的多个信息。函数尝试在网络接口上查找下一个目标MAC地址，如果找到，并返回其地址。

这两个函数一起使用，以确保在直接连接的机器数量较少的情况下，能够为每个目标设备设置下一个目标MAC地址。


```cpp
*/

int setTargetMACIfAvailable(Target *target, struct link_header *linkhdr,
                            const struct sockaddr_storage *src, int overwrite);

/* This function ensures that the next hop MAC address for a target is
   filled in.  This address is the target's own MAC if it is directly
   connected, and the next hop mac otherwise.  Returns true if the
   address is set when the function ends, false if not.  This function
   firt checks if it is already set, if not it tries the arp cache,
   and if that fails it sends an ARP request itself.  This should be called
   after an ARP scan if many directly connected machines are involved. */
bool setTargetNextHopMAC(Target *target);

bool getNextHopMAC(const char *iface, const u8 *srcmac, const struct sockaddr_storage *srcss,
                   const struct sockaddr_storage *dstss, u8 *dstmac);

```

这段代码是用于读取TCP数据包中的时间戳选项信息，如果数据包包含时间戳选项，则返回timestamp和echots参数的值，否则返回0。同时，它还接受一个链接头信息作为输入参数，用于在数据包中查找时间戳选项。

具体来说，代码首先定义了两个函数readipv4_pcap和readip_pcap，这两个函数接受一个链接头信息作为输入参数，然后分别实现从数据包中读取IPv4头部和TCP头部，并将读取到的数据存储在一个u8指针中。接着定义了一个常量u8_timestamp_option_is_available，用于判断数据包中是否包含时间戳选项。最后在rcvdtime和linknfo中填入相应的值，并将函数的返回值作为整数返回。


```cpp
/* If rcvdtime is non-null and a packet is returned, rcvd will be
   filled with the time that packet was captured from the wire by
   pcap.  If linknfo is not NULL, lnkinfo->headerlen and
   lnkinfo->header will be filled with the appropriate values. */
const u8 *readipv4_pcap(pcap_t *pd, unsigned int *len, long to_usec,
                    struct timeval *rcvdtime, struct link_header *linknfo, bool validate);

const u8 *readip_pcap(pcap_t *pd, unsigned int *len, long to_usec,
                  struct timeval *rcvdtime, struct link_header *linknfo, bool validate);

/* Examines the given tcp packet and obtains the TCP timestamp option
   information if available.  Note that the CALLER must ensure that
   "tcp" contains a valid header (in particular the th_off must be the
   true packet length and tcp must contain it).  If a valid timestamp
   option is found in the header, nonzero is returned and the
   'timestamp' and 'echots' parameters are filled in with the
   appropriate value (if non-null).  Otherwise 0 is returned and the
   parameters (if non-null) are filled with 0.  Remember that the
   correct way to check for errors is to look at the return value
   since a zero ts or echots could possibly be valid. */
```

这段代码定义了两个函数：`gettcpopt_ts` 和 `max_rcvbuf`。

函数 `gettcpopt_ts` 接收一个指向 `tcp_hdr` 结构体的指针 `tcp`，以及两个指向 `timestamp` 和 `echots` 且均为 `u32` 的指针 `timestamp` 和 `echots`。它将获取一个 socket 的接收缓冲区的最大大小（最多 500K），并将该缓冲区的最大大小设置为 `timestamp` 和 `echots` 的值。

函数 `max_rcvbuf` 接受一个整数 `sd`，它表示 socket descriptor（套口描述符）。函数使用 `recvtime` 函数在 `sd` 上进行一次收发操作，并将结果存储在 `buf` 中。收发操作持续时间为 `seconds`，如果 `timedout` 参数为 `NULL`，则表示操作不会超时，否则表示操作超时，此时返回 `0`。

由于 `recvtime` 函数没有具体的实现，因此无法提供其作用的具体内容。


```cpp
int gettcpopt_ts(const struct tcp_hdr *tcp, u32 *timestamp, u32 *echots);

/* Maximize the receive buffer of a socket descriptor (up to 500K) */
void max_rcvbuf(int sd);

/* Do a receive (recv()) on a socket and stick the results (upt to
   len) into buf .  Give up after 'seconds'.  Returns the number of
   bytes read (or -1 in the case of an error.  It only does one recv
   (it will not keep going until len bytes are read).  If timedout is
   not NULL, it will be set to zero (no timeout occurred) or 1 (it
   did). */
int recvtime(int sd, char *buf, int len, int seconds, int *timedout);

#endif /*TCPIP_H*/


```

# `timing.h`

This text is a part of the Nmap license agreement. Nmap is a network scanner that allows users to scan for and discover network-connected devices, including routers, switches, and hosts. The text prohibits companies from using the Nmap software in commercial products, but allows the use of a special Nmap OEM edition with more permissive terms. The text also states that if a company has received an Nmap license agreement or contract with terms other than those of the Nmap license, they can use and redistribute Nmap under those terms. The text also provides information about the source code and license terms for Nmap.



```cpp

/***************************************************************************
 * timing.h -- Functions related to computing scan timing (such as keeping *
 * track of and adjusting smoothed round trip times, statistical           *
 * deviations, timeout values, etc.  Various user options (such as the     *
 * timing policy (-T)) also play a role in these calculations.             *
 *                                                                         *
 ***********************IMPORTANT NMAP LICENSE TERMS************************
 *
 * The Nmap Security Scanner is (C) 1996-2023 Nmap Software LLC ("The Nmap
 * Project"). Nmap is also a registered trademark of the Nmap Project.
 *
 * This program is distributed under the terms of the Nmap Public Source
 * License (NPSL). The exact license text applying to a particular Nmap
 * release or source code control revision is contained in the LICENSE
 * file distributed with that version of Nmap or source code control
 * revision. More Nmap copyright/legal information is available from
 * https://nmap.org/book/man-legal.html, and further information on the
 * NPSL license itself can be found at https://nmap.org/npsl/ . This
 * header summarizes some key points from the Nmap license, but is no
 * substitute for the actual license text.
 *
 * Nmap is generally free for end users to download and use themselves,
 * including commercial use. It is available from https://nmap.org.
 *
 * The Nmap license generally prohibits companies from using and
 * redistributing Nmap in commercial products, but we sell a special Nmap
 * OEM Edition with a more permissive license and special features for
 * this purpose. See https://nmap.org/oem/
 *
 * If you have received a written Nmap license agreement or contract
 * stating terms other than these (such as an Nmap OEM license), you may
 * choose to use and redistribute Nmap under those terms instead.
 *
 * The official Nmap Windows builds include the Npcap software
 * (https://npcap.com) for packet capture and transmission. It is under
 * separate license terms which forbid redistribution without special
 * permission. So the official Nmap Windows builds may not be redistributed
 * without special permission (such as an Nmap OEM license).
 *
 * Source is provided to this software because we believe users have a
 * right to know exactly what a program is going to do before they run it.
 * This also allows you to audit the software for security holes.
 *
 * Source code also allows you to port Nmap to new platforms, fix bugs, and add
 * new features. You are highly encouraged to submit your changes as a Github PR
 * or by email to the dev@nmap.org mailing list for possible incorporation into
 * the main distribution. Unless you specify otherwise, it is understood that
 * you are offering us very broad rights to use your submissions as described in
 * the Nmap Public Source License Contributor Agreement. This is important
 * because we fund the project by selling licenses with various terms, and also
 * because the inability to relicense code has caused devastating problems for
 * other Free Software projects (such as KDE and NASM).
 *
 * The free version of Nmap is distributed in the hope that it will be
 * useful, but WITHOUT ANY WARRANTY; without even the implied warranty of
 * MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. Warranties,
 * indemnification and commercial support are all available through the
 * Npcap OEM program--see https://nmap.org/oem/
 *
 ***************************************************************************/

```

这段代码是一个尼古拉斯网络包（Numpy）中的 timing 函数，用于计算代码运行时的时间。主要作用是定义了一个常量，通过判断系统时间（使用 GETTIMEOFDAY 函数）是否可用，如果可用，则使用系统时间来计时，否则使用从头文件 time.h 中提供的函数。通过这个 timing 函数，可以保证在运行时，程序能够正确记录代码执行的时间，即使在没有显式的时间处理函数的情况下，尼古拉斯网络包仍然会保证代码正确工作。


```cpp
/* $Id$ */

#ifndef NMAP_TIMING_H
#define NMAP_TIMING_H

#if TIME_WITH_SYS_TIME
# include <sys/time.h>
# include <time.h>
#else
# if HAVE_SYS_TIME_H
#  include <sys/time.h>
# else
#  include <time.h>
# endif
#endif

```

这段代码定义了一个名为`struct ultra_timing_vals`的结构体，它用于在网络拥塞控制技术中实现拥塞窗口调节。

该结构体包含以下字段：

* `cwnd`：拥塞窗口，即在往返信令中的平均带宽。
* `ssthresh`：在慢开始模式和拥塞避免模式之间的阈值。当超过此阈值时，将进入拥塞避免模式。
* `num_replies_expected`：我们预计接收到多少个数据报的回复。这个值在每次发送数据报时都会更新。
* `num_replies_received`：我们已经接收到多少个数据报的回复。这个值在每次发送数据报时都会更新。
* `num_updates`：基于拥塞窗口调整的定时器，用于处理数据报传输过程中的时间戳更新。
* `last_drop`：上一次数据报传输过程中的最后一个时间戳调整。

此外，该结构体还包含以下函数：

* `cc_scale`：根据`scan_performance_vars`结构体中的采样性能变量对拥塞窗口进行缩放。
* `ack`：根据`scan_performance_vars`结构体中的采样性能变量对拥塞窗口进行升级。
* `drop`：实现数据报数据丢失掉落。
* `drop_group`：实现数据报数据丢失掉落，但是仅在指定的时间间隔内。

该结构体在网络拥塞控制技术中起到关键作用，它通过根据采样的性能变量来调整拥塞窗口的大小，以提高网络的传输效率。


```cpp
#include <nbase.h> /* u32 */

/* Based on TCP congestion control techniques from RFC2581. */
struct ultra_timing_vals {
  double cwnd; /* Congestion window - in probes */
  int ssthresh; /* The threshold above which mode is changed from slow start
                   to congestion avoidance */
  /* The number of replies we would expect if every probe produced a reply. This
     is almost like the total number of probes sent but it is not incremented
     until a reply is received or a probe times out. This and
     num_replies_received are used to scale congestion window increments. */
  int num_replies_expected;
  /* The number of replies we've received to probes of any type. */
  int num_replies_received;
  /* Number of updates to this timing structure (generally packet receipts). */
  int num_updates;
  /* Last time values were adjusted for a drop (you usually only want
     to adjust again based on probes sent after that adjustment so a
     sudden batch of drops doesn't destroy timing.  Init to now */
  struct timeval last_drop;

  double cc_scale(const struct scan_performance_vars *perf);
  void ack(const struct scan_performance_vars *perf, double scale = 1.0);
  void drop(unsigned in_flight,
    const struct scan_performance_vars *perf, const struct timeval *now);
  void drop_group(unsigned in_flight,
    const struct scan_performance_vars *perf, const struct timeval *now);
};

```



这段代码定义了一个名为`scan_performance_vars`的结构体，用于表示网络性能参数。这个结构体包含了一些初始化变量和一个`init`函数。

具体来说，这个结构体用于在网络拥塞情况下进行参数设置，以确定网络的性能和行为。以下是一些主要的作用：

1. `int low_cwnd;`表示低拥塞窗口的最小值，即在什么情况下网络应该停止发送数据包。

2. `int host_initial_cwnd;`表示主机初始拥塞窗口的最小值，即在什么情况下主机应该停止发送数据包。

3. `int group_initial_cwnd;`表示主机组初始拥塞窗口的最小值，即在什么情况下主机组应该停止发送数据包。

4. `int max_cwnd;`表示可以拥有的最大拥塞窗口。

5. `int slow_incr;`表示在慢开始模式下，每个响应所增加的代理数量。

6. `int ca_incr;`表示在拥塞避免模式下，每个响应所增加的代理数量，与`slow_incr`类似，但是这个值大致相当于`slow_incr`的值。

7. `int cc_scale_max;`表示拥塞窗口增加的最大比例。

8. `int initial_ssthresh;`表示慢开始阈值，即代理开始降低拥塞窗口时的阈值。

9. `double group_drop_cwnd_divisor;`表示主机组掉包除数，用于将所有主机分摊的拥塞窗口。

10. `double group_drop_ssthresh_divisor;`表示用于主机组下陷阈值的除数，用于指定主机组下陷阈值。

11. `double host_drop_ssthresh_divisor;`表示用于主机下陷阈值的除数，用于指定主机下陷阈值。

12. `void init();`表示一个初始化函数，该函数在全局变量被填充后执行，但不会输出任何函数调用。


```cpp
/* These are mainly initializers for ultra_timing_vals. */
struct scan_performance_vars {
  int low_cwnd;  /* The lowest cwnd (congestion window) allowed */
  int host_initial_cwnd; /* Initial congestion window for ind. hosts */
  int group_initial_cwnd; /* Initial congestion window for all hosts as a group */
  int max_cwnd; /* I should never have more than this many probes
                   outstanding */
  int slow_incr; /* How many probes are incremented for each response
                    in slow start mode */
  int ca_incr; /* How many probes are incremented per (roughly) rtt in
                  congestion avoidance mode */
  int cc_scale_max; /* The maximum scaling factor for congestion window
                       increments. */
  int initial_ssthresh;
  double group_drop_cwnd_divisor; /* all-host group cwnd divided by this
                                     value if any packet drop occurs */
  double group_drop_ssthresh_divisor; /* used to drop the group ssthresh when
                                         any drop occurs */
  double host_drop_ssthresh_divisor; /* used to drop the host ssthresh when
                                         any drop occurs */

  /* Do initialization after the global NmapOps table has been filled in. */
  void init();
};

```

这段代码定义了一个名为`timeout_info`的结构体，其中包含三个整型成员变量：`srtt`表示平滑的RTT估计(微秒),`rttvar`表示RTT时间方差，`timeout`表示当前的timeout阈值(微秒)。

`initialize_timeout_info`函数的作用是初始化上述结构体类型的变量。

`adjust_timeouts2`函数接受两个时间戳变量`sent`表示发送时间戳和`received`表示接收时间戳，以及一个`timeout_info`类型的参数`to`。函数使用`send`和`receive`时间戳计算当前的RTT估计，并尝试更新`timeout_info`中的`timeout`成员变量以保证在调用者调用该函数后，`timeout`不会超过设置的最大值。


```cpp
struct timeout_info {
  int srtt; /* Smoothed rtt estimate (microseconds) */
  int rttvar; /* Rout trip time variance */
  int timeout; /* Current timeout threshold (microseconds) */
};

/* Call this function on a newly allocated struct timeout_info to
   initialize the values appropriately */
void initialize_timeout_info(struct timeout_info *to);

/* Same as adjust_timeouts(), except this one allows you to specify
 the receive time too (which could be because it was received a while
 back or it could be for efficiency because the caller already knows
 the current time */
void adjust_timeouts2(const struct timeval *sent,
                      const struct timeval *received,
                      struct timeout_info *to);

```

这段代码定义了一个名为RateMeter的类，用于测量某个数量的当前和寿命平均速率。它有两个主要函数：enforce_scan_delay和adjust_timeouts。

enforce_scan_delay函数用于确保在回送延迟期间不会再次调用自己。它接收一个指向当前时间的timeval作为参数，并在需要时调用。

adjust_timeouts函数用于根据最新的探测所花费的时间来调整超时时间。它接收两个结构体，一个用于存储当前时间，另一个用于存储时间戳。在函数内部，调整时间戳，然后将调整后的时间戳存储回调整时间戳。

RateMeter类有一个const类型的成员current_rate_history，用于设置或清除计时器。它有一个非空的时间戳变量start_tv，用于设置计时的起点，一个停止计时的timeval variable stop_tv，用于设置计时的截止日期，和一个指向当前时间的last_update_tv。它还有一个total变量，用于存储总时间。另外，它有一个isSet函数，用于检查给定的时间是否已经设置。

此外，RateMeter类还实现了一个辅助函数getOverallRate，用于获取当前计时的速率，以及getCurrentRate和getTotal函数，用于获取当前计时的速率和计数值。最后，它还有一个elapsedTime函数，用于计算记录到的延迟时间。


```cpp
/* Adjust our timeout values based on the time the latest probe took for a
   response.  We update our RTT averages, etc. */
void adjust_timeouts(struct timeval sent, struct timeout_info *to);

#define DEFAULT_CURRENT_RATE_HISTORY 5.0

/* Sleeps if necessary to ensure that it isn't called twice within less
   time than o.send_delay.  If it is passed a non-null tv, the POST-SLEEP
   time is recorded in it */
void enforce_scan_delay(struct timeval *tv);

/* This class measures current and lifetime average rates for some quantity. */
class RateMeter {
  public:
    RateMeter(double current_rate_history = DEFAULT_CURRENT_RATE_HISTORY);

    void start(const struct timeval *now = NULL);
    void stop(const struct timeval *now = NULL);
    void update(double amount, const struct timeval *now = NULL);
    double getOverallRate(const struct timeval *now = NULL) const;
    double getCurrentRate(const struct timeval *now = NULL, bool update = true);
    double getTotal(void) const;
    double elapsedTime(const struct timeval *now = NULL) const;

  private:
    /* How many seconds to look back when calculating the "current" rates. */
    double current_rate_history;

    /* When this meter started recording. */
    struct timeval start_tv;
    /* When this meter stopped recording. */
    struct timeval stop_tv;
    /* The last time the current sample rates were updated. */
    struct timeval last_update_tv;

    double total;
    double current_rate;

    static bool isSet(const struct timeval *tv);
};

```

这段代码定义了一个名为 PacketRateMeter 的类，用于测量数据包和字节速率。它实现了 RateMeter 类的特殊ization，用于从给定时间戳的计数值中计算数据包和字节数以及每个时间步的速率。

PacketRateMeter 的构造函数接受一个当前速率历史参数，用于记录从开始时间到当前时间的数据包和字节数。start 函数用于开始计时，stop 函数用于结束计时，update 函数用于更新从开始时间到当前时间的数据包和字节数。

getOverallPacketRate 和 getCurrentPacketRate 函数分别返回数据包速率和当前数据包速率，它们使用相同的时间戳作为输入，并更新基于从开始时间到当前时间的计数值。

getOverallByteRate 和 getCurrentByteRate 函数分别返回字节速率和当前字节速率，它们使用相同的时间戳作为输入，并更新基于从开始时间到当前时间的计数值。

getNumPackets 和 getNumBytes 函数分别返回过去数据包和字节数，它们使用从开始时间到当前时间的计数值来计算。

总之，这段代码提供了一个测量数据包和字节速率的工具，可以对网络数据包和字节数据进行统计和测量。


```cpp
/* A specialization of RateMeter that measures packet and byte rates. */
class PacketRateMeter {
  public:
    PacketRateMeter(double current_rate_history = DEFAULT_CURRENT_RATE_HISTORY);

    void start(const struct timeval *now = NULL);
    void stop(const struct timeval *now = NULL);
    void update(u32 len, const struct timeval *now = NULL);
    double getOverallPacketRate(const struct timeval *now = NULL) const;
    double getCurrentPacketRate(const struct timeval *now = NULL, bool update = true);
    double getOverallByteRate(const struct timeval *now = NULL) const;
    double getCurrentByteRate(const struct timeval *now = NULL, bool update = true);
    unsigned long long getNumPackets(void) const;
    unsigned long long getNumBytes(void) const;

  private:
    RateMeter packet_rate_meter;
    RateMeter byte_rate_meter;
};

```

这段代码定义了一个名为 ScanProgressMeter 的类，该类包含了一些通用的方法，如构造函数、析构函数和可能需要打印定时统计信息的函数。

具体来说，构造函数接受一个字符串参数 stypestr，该参数表示打印在什么时间间隔内，以及打印统计信息的次数和统计信息的详细程度。这些参数可以用于指定打印哪些统计信息，以及何时开始打印它们。

析构函数用来判断是否需要打印最新的统计信息。它检查当前时间是否与之前计算进度时的时间相等，或者当前时间是否已经过期。如果当前时间与之前计算进度的时间不相等，或者当前时间已经过期，那么就调用 mayBePrinted 函数来检查是否需要打印最新的统计信息。

ScanProgressMeter类还包含一个静止时间戳 (time_expires) 和一个打印当前进度信息的函数（printStatsIfNecessary），这些函数在之前的解析中已经定义。


```cpp
class ScanProgressMeter {
 public:
  /* A COPY of stypestr is made and saved for when stats are printed */
  ScanProgressMeter(const char *stypestr);
  ~ScanProgressMeter();
/* Decides whether a timing report is likely to even be
   printed.  There are stringent limitations on how often they are
   printed, as well as the verbosity level that must exist.  So you
   might as well check this before spending much time computing
   progress info.  now can be NULL if caller doesn't have the current
   time handy.  Just because this function returns true does not mean
   that the next printStatsIfNecessary will always print something.
   It depends on whether time estimates have changed, which this func
   doesn't even know about. */
  bool mayBePrinted(const struct timeval *now);

```

这段代码定义了一个名为 printStats 的函数，它的作用是打印一个估计扫描完成时间，前提是 mayBePrinted() 变量为 true，并且似乎合理的打印时间估计。最后，函数返回一个布尔值，表示是否打印了该行。

printStats函数的实现包括两个函数：printStatsIfNecessary 和 printStats。printStatsIfNecessary函数接收两个参数：一个是已经完成工作的百分比（perc_done），另一个是当前时间戳（now），函数返回一个布尔值，表示是否打印了该行。printStats函数是printStatsIfNecessary的函数，用于在完成工作百分比较低时打印时间估计。两个函数的实现基本相同，都是打印一个估计扫描完成时间，并在完成工作百分比较高时，输出一个表示扫描完成时间窗口的估计值，类似于一个消息：扫描即将完成。

此外，endTask函数用于结束当前任务，它接收一个已经完成工作的时间戳（now）和一个额外的信息（additional_info），如果任务结束，函数返回 true，否则返回 false。beginTask 和 endTask 函数在代码中并未实现，可能是后续需要用到的函数。


```cpp
/* Prints an estimate of when this scan will complete.  It only does
   so if mayBePrinted() is true, and it seems reasonable to do so
   because the estimate has changed significantly.  Returns whether
   or not a line was printed.*/
  bool printStatsIfNecessary(double perc_done, const struct timeval *now);

  /* Prints an estimate of when this scan will complete. */
  bool printStats(double perc_done, const struct timeval *now);

  /* Prints that this task is complete. */
  bool endTask(const struct timeval *now, const char *additional_info) { return beginOrEndTask(now, additional_info, false); }

  struct timeval begin; /* When this ScanProgressMeter was instantiated */
 private:
  struct timeval last_print_test; /* Last time printStatsIfNecessary was called */
  struct timeval last_print; /* The most recent time the ETC was printed */
  char *scantypestr;
  struct timeval last_est; /* The latest PRINTED estimate */

  bool beginOrEndTask(const struct timeval *now, const char *additional_info, bool beginning);
};

```

这段代码是一个尼古拉天文台（NMAP）库的包含指针，它指定了全局一个名为“NMAP_TIMING_H”的预处理头文件。这个预处理指令会检查NMAP库是否支持C++11中的[[您的私有签名字符，不会输出任何内容]]（#ifdef...#endif）预处理指令，如果支持，那么将预处理指令后面的代码替换为尼古拉天文台的版本，否则报告错误。


```cpp
#endif /* NMAP_TIMING_H */


```

# `traceroute.h`

This text is a description of the Nmap software and its licensing. Nmap is a network scanner that allows users to scan for and discover devices on a network, as well as provide information about the target. The text explains that Nmap is under a number of licenses, including an Nmap license that allows companies to use it for commercial purposes, and an Nmap OEM license that restricts redistribution of the software in commercial products. The text also explains that the Nmap windows builds include the Npcap software for packet capture and transmission, which is under a different license. The text provides contact information for the Nmap team and encourages users to submit changes to the software to the Npcap developers.



```cpp

/***************************************************************************
 * traceroute.h -- Parallel multi-protocol traceroute feature              *
 *                                                                         *
 ***********************IMPORTANT NMAP LICENSE TERMS************************
 *
 * The Nmap Security Scanner is (C) 1996-2023 Nmap Software LLC ("The Nmap
 * Project"). Nmap is also a registered trademark of the Nmap Project.
 *
 * This program is distributed under the terms of the Nmap Public Source
 * License (NPSL). The exact license text applying to a particular Nmap
 * release or source code control revision is contained in the LICENSE
 * file distributed with that version of Nmap or source code control
 * revision. More Nmap copyright/legal information is available from
 * https://nmap.org/book/man-legal.html, and further information on the
 * NPSL license itself can be found at https://nmap.org/npsl/ . This
 * header summarizes some key points from the Nmap license, but is no
 * substitute for the actual license text.
 *
 * Nmap is generally free for end users to download and use themselves,
 * including commercial use. It is available from https://nmap.org.
 *
 * The Nmap license generally prohibits companies from using and
 * redistributing Nmap in commercial products, but we sell a special Nmap
 * OEM Edition with a more permissive license and special features for
 * this purpose. See https://nmap.org/oem/
 *
 * If you have received a written Nmap license agreement or contract
 * stating terms other than these (such as an Nmap OEM license), you may
 * choose to use and redistribute Nmap under those terms instead.
 *
 * The official Nmap Windows builds include the Npcap software
 * (https://npcap.com) for packet capture and transmission. It is under
 * separate license terms which forbid redistribution without special
 * permission. So the official Nmap Windows builds may not be redistributed
 * without special permission (such as an Nmap OEM license).
 *
 * Source is provided to this software because we believe users have a
 * right to know exactly what a program is going to do before they run it.
 * This also allows you to audit the software for security holes.
 *
 * Source code also allows you to port Nmap to new platforms, fix bugs, and add
 * new features. You are highly encouraged to submit your changes as a Github PR
 * or by email to the dev@nmap.org mailing list for possible incorporation into
 * the main distribution. Unless you specify otherwise, it is understood that
 * you are offering us very broad rights to use your submissions as described in
 * the Nmap Public Source License Contributor Agreement. This is important
 * because we fund the project by selling licenses with various terms, and also
 * because the inability to relicense code has caused devastating problems for
 * other Free Software projects (such as KDE and NASM).
 *
 * The free version of Nmap is distributed in the hope that it will be
 * useful, but WITHOUT ANY WARRANTY; without even the implied warranty of
 * MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. Warranties,
 * indemnification and commercial support are all available through the
 * Npcap OEM program--see https://nmap.org/oem/
 *
 ***************************************************************************/

```

这是一段C++代码，定义了一个名为`traceroute`的函数，它的功能是计算从给定节点（traceroute starting node）到给定节点集合（traceroute starting set）的最短路径。该函数接受一个`std::vector<Target *>`类型的输入，其中`Target`是一个自定义的类，代表一个目标节点，该函数返回最短路径及其相关的信息。

函数实现包括两个部分：

1. 函数声明：```cppcpp
int traceroute(std::vector<Target *> &Targets);
```

2. 函数实现：```cppcpp
int traceroute(std::vector<Target *> &Targets) {
   // 在这里实现具体的traceroute算法
   // ...
   return 0;
}
```

这两个部分一起定义了函数`traceroute`的整体行为。函数`traceroute`接收一个`std::vector<Target *>`类型的输入，其中包含目标节点（Target）的引用。函数返回一个整数类型的值，表示从起始节点到给定节点集合的最短路径的长度。

此外，函数还实现了一个名为`traceroute_hop_cache_clear`的函数，它的功能是清除traceroute算法中的缓存。这个函数没有具体的实现，可能是为了在后续计算中初始化缓存。


```cpp
/* $Id$ */

#ifndef NMAP_TRACEROUTE_H
#define NMAP_TRACEROUTE_H

#include <vector>

class Target;

int traceroute(std::vector<Target *> &Targets);

void traceroute_hop_cache_clear();

#endif

```

# `utils.h`

This is a text that provides information about the Nmap open-source software and the Npcap packet capture and transmission software. It explains that the Nmap license prohibits companies from using the software in commercial products, but allows for a special Nmap OEM Edition with a more permissive license. The license for the OEM Edition allows companies to use the software in commercial products, but with the understanding that they will not redistribute the software without special permission. The text also mentions that the official Nmap Windows builds include the Npcap software, which is under a separate license term that prohibits redistribution without special permission. The text explains that the Nmap source code is available for users to view and modify, and suggests that users should submit any changes to the dev@nmap.org mailing list for possible incorporation into the main distribution.



```cpp
/***************************************************************************
 * utils.h -- Various miscellaneous utility functions which defy           *
 * categorization :)                                                       *
 *                                                                         *
 ***********************IMPORTANT NMAP LICENSE TERMS************************
 *
 * The Nmap Security Scanner is (C) 1996-2023 Nmap Software LLC ("The Nmap
 * Project"). Nmap is also a registered trademark of the Nmap Project.
 *
 * This program is distributed under the terms of the Nmap Public Source
 * License (NPSL). The exact license text applying to a particular Nmap
 * release or source code control revision is contained in the LICENSE
 * file distributed with that version of Nmap or source code control
 * revision. More Nmap copyright/legal information is available from
 * https://nmap.org/book/man-legal.html, and further information on the
 * NPSL license itself can be found at https://nmap.org/npsl/ . This
 * header summarizes some key points from the Nmap license, but is no
 * substitute for the actual license text.
 *
 * Nmap is generally free for end users to download and use themselves,
 * including commercial use. It is available from https://nmap.org.
 *
 * The Nmap license generally prohibits companies from using and
 * redistributing Nmap in commercial products, but we sell a special Nmap
 * OEM Edition with a more permissive license and special features for
 * this purpose. See https://nmap.org/oem/
 *
 * If you have received a written Nmap license agreement or contract
 * stating terms other than these (such as an Nmap OEM license), you may
 * choose to use and redistribute Nmap under those terms instead.
 *
 * The official Nmap Windows builds include the Npcap software
 * (https://npcap.com) for packet capture and transmission. It is under
 * separate license terms which forbid redistribution without special
 * permission. So the official Nmap Windows builds may not be redistributed
 * without special permission (such as an Nmap OEM license).
 *
 * Source is provided to this software because we believe users have a
 * right to know exactly what a program is going to do before they run it.
 * This also allows you to audit the software for security holes.
 *
 * Source code also allows you to port Nmap to new platforms, fix bugs, and add
 * new features. You are highly encouraged to submit your changes as a Github PR
 * or by email to the dev@nmap.org mailing list for possible incorporation into
 * the main distribution. Unless you specify otherwise, it is understood that
 * you are offering us very broad rights to use your submissions as described in
 * the Nmap Public Source License Contributor Agreement. This is important
 * because we fund the project by selling licenses with various terms, and also
 * because the inability to relicense code has caused devastating problems for
 * other Free Software projects (such as KDE and NASM).
 *
 * The free version of Nmap is distributed in the hope that it will be
 * useful, but WITHOUT ANY WARRANTY; without even the implied warranty of
 * MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. Warranties,
 * indemnification and commercial support are all available through the
 * Npcap OEM program--see https://nmap.org/oem/
 *
 ***************************************************************************/

```

这段代码定义了一个名为`util_h`的头文件，其中包含了一些通用的函数和宏。

首先，通过`#ifndef`和`#define`来定义了一些条件下的代码。`#ifndef WIN32`表示只有在`WIN32`操作系统上才会编译该代码，否则不会输出任何内容。`#include <sys/mman.h>`和`#include "nbase.h"`包含了一些标准库函数和头文件，与该代码的作用无关。

然后，定义了一些通用的函数和宏。`MOD_DIFF`是一个 arithmetic function，用于计算两个整数之间的差异，取模ulo 2^32。`MAX_VAL`是一个帮助函数，用于将一个整数转换为最接近的十进制数。`__GET_位的掩码`是一个位运算，返回一个掩码，用于将一个二进制数设置为某个位。

最后，没有定义任何函数，所以该头文件也没有任何功能。


```cpp
/* $Id$ */

#ifndef UTILS_H
#define UTILS_H

#ifndef WIN32
#include <sys/mman.h>
#endif

#include "nbase.h"
#include <assert.h>

/* Arithmatic difference modulo 2^32 */
#ifndef MOD_DIFF
#define MOD_DIFF(a,b) ((u32) (MIN((u32)(a) - (u32 ) (b), (u32 )(b) - (u32) (a))))
```

这段代码定义了一些常量和宏，其中最主要的用途是定义了一个带参数的函数 `mod_diff_ushort`，它的功能是计算两个 16 位无符号整数 `a` 和 `b` 的差。

常量 `MAX_PARSE_ARGS` 定义了一个最大参数数，用于对函数进行Integer Underflow Handling。

此外，该代码中还定义了一些宏，包括 `MOD_DIFF_USHORT`，用于定义 `a` 和 `b` 的差，以及 `FALSE` 和 `TRUE`，分别表示 false 和 true。


```cpp
#endif

/* Arithmatic difference modulo 2^16 */
#ifndef MOD_DIFF_USHORT
#define MOD_DIFF_USHORT(a,b) ((MIN((unsigned short)((unsigned short)(a) - (unsigned short ) (b)), (unsigned short) ((unsigned short )(b) - (unsigned short) (a)))))
#endif
#ifndef FALSE
#define FALSE 0
#endif
#ifndef TRUE
#define TRUE 1
#endif

#define MAX_PARSE_ARGS 254 /* +1 for integrity checking + 1 for null term */

```



这段代码是一个C++模板类，名为` wildtest`.

函数`wildtest`的参数为`const char *wild`和`const char *test`，分别表示要测试的密码和测试的输入。

函数体以下是一个没有输出语句的函数，但是通过观察函数体，可以看到它实现了一个“wild”函数，该函数接受两个字符串参数`wild`和`test`，并返回一个打印该“wild”的函数的ID的值。

接着，函数体又定义了一个函数`nmap_hexdump`，该函数接受一个`const unsigned char *cp`和一个`unsigned int length`作为参数，表示要打印的字符串和该字符串的长度。

需要指出的是，该函数没有实现具体的打印操作，只是定义了一个函数体，因此需要由其他函数调用该函数来实际输出字符串。


```cpp
/* Return num if it is between min and max.  Otherwise return min or max
   (whichever is closest to num). */
template<class T> T box(T bmin, T bmax, T bnum) {
  assert(bmin <= bmax);
  if (bnum >= bmax)
    return bmax;
  if (bnum <= bmin)
    return bmin;
  return bnum;
}

int wildtest(const char *wild, const char *test);

void nmap_hexdump(const unsigned char *cp, unsigned int length);

```



1. genfry函数的作用是生成 fair-sized array，即数组长度为elem_sz，元素个数为num_elem，并且每个元素都是大小为elem_sz的整数类型。

2. shortfry函数的作用和genfry函数相似，只是生成的数组长度为num_elem，元素个数为16，每个元素为short类型。

3. chomp函数的作用是截取输入字符串中的前指定长度，并返回其中的前指定长度字符。

4. Send函数的作用是发送消息，其中接收方为套接字，消息长度为len，发送方为发送者sd的整数值。

5. arg_parse函数的作用是解析命令行参数，其中第一个参数为命令名称，第二个参数为参数列表。

6. arg_parse_free函数的作用是释放arg_parse函数传递给它的参数列表。

7. cstring_unescape函数的作用是将输入的字符串中的 escape 转义，并返回处理后的字符串。

8. bintohexstr函数的作用是将输入的字符串中的所有字符转换为十六进制字符串，并返回处理后的字符串。

9. parse_hex_string函数的作用是解析输入的十六进制字符串，并返回其中的字符串。

10. u8 *parse_hex_string函数的作用是解析输入的十六进制字符串，并返回一个u8数组，该数组包含解析后的字符串。


```cpp
void genfry(unsigned char *arr, int elem_sz, int num_elem);
void shortfry(unsigned short *arr, int num_elem);
char *chomp(char *string);

int Send(int sd, const void *msg, size_t len, int flags);

int arg_parse(const char *command, char ***argv);
void arg_parse_free(char **argv);

char *cstring_unescape(char *str, unsigned int *len);

void bintohexstr(char *buf, int buflen, const char *src, int srclen);

u8 *parse_hex_string(const char *str, size_t *outlen);

```



这两行代码是用于从环境中获取配置文件part的函数。

`int cpe_get_part(const char *cpe)` 函数从传入的 `cpe` 参数中获取 part 信息，并返回一个整数。可能用于从设备驱动程序或用户配置文件中读取配置信息。

`char *mmapfile(char *fname, s64 *length, int openflags)` 函数用于在内存中映射一个文件到指定大小并设置访问权限。它将 `fname` 参数是一个已经存在的文件名，`length` 参数是目标映射的大小，`openflags` 参数是一个用于指定文件权限的整数。它可能用于在进程中使用操作系统提供的文件映射功能。

在 `#ifdef WIN32` 注释之前的内容可能是用C语言编写的，而 `#ifdef WIN32` 注释之后的内容则是一段用于输出特定错误代码的代码。它可能用于在 Windows 上处理错误的条件。


```cpp
int cpe_get_part(const char *cpe);

char *mmapfile(char *fname, s64 *length, int openflags);

#ifdef WIN32
int win32_munmap(char *filestr, int filelen);
#endif /* WIN32 */

#endif /* UTILS_H */


```