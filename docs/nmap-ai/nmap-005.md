# Nmap源码解析 5

# `protocols.h`

This is a text segment describing the Nmap software and its licensing. Nmap is a network scanner that allows users to scan for vulnerabilities, as well as perform other tasks such as analyzing network traffic. The licensing for Nmap is explained, including the Nmap OEM Edition, which has a more permissive license and special features. The Nmap license generally prohibits companies from using and redistributing Nmap in commercial products, but the Nmap OEM Edition allows for this. The official Nmap Windows builds include the Npcap software for packet capture and transmission, which is under a separate license term that prohibits redistribution without special permission. The Nmap source code is available for users to view and modify, and users are encouraged to contribute to the project through a GitHub pull request or by email to the dev@nmap.org mailing list. The Nmap Public Source License Contributor Agreement has specific terms and conditions for using the Nmap software under the free version of the software.



```cpp

/***************************************************************************
 * protocols.h -- Functions relating to the protocol scan and mapping      *
 * between IPproto Number <-> name.                                        *
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

这段代码定义了一个名为`PROTOCOLS_H`的头部文件，其中包含了一些关于协议头信息的内容。

首先，它包括了一个`#include "nbase.h"`的声明，这个头部文件可能是一个网络协议的`nbase.h`头文件，它定义了一些与网络协议相关的头信息。

接着，它又包括了一个`#ifndef PROTOCOLS_H`到`#define PROTOCOLS_H`的注释，这个注释表示这个头部文件的作用是定义一个名为`PROTOCOLS_H`的头部文件，其中包含了一些协议头信息。

在`#include "libnetutil/netutil.h"`中，它引入了`netutil.h`头文件，这个头文件可能是一个网络协议的`libnetutil.h`头文件，它定义了一些与网络协议相关的头信息。

接着，它定义了一个名为`struct nprotoent`的结构体类型，其中包含一个名为`p_name`的属性，它是一个字符串类型，用于存放协议的名称。另一个属性`p_proto`是一个16字节的整型，用于存放协议类型。

最后，该头部文件还包括了一些其他头信息，比如`#define MAX_PROTOCOLS 100`和`#define MAX_NTHREADS 10`，它们用于定义`MAX_PROTOCOLS`和`MAX_NTHREADS`的常量，用于在编译时限制可支持的协议数量和线程数量。


```cpp
/* $Id$ */

#ifndef PROTOCOLS_H
#define PROTOCOLS_H

#include "nbase.h"

#ifndef IPPROTO_SCTP
#include "libnetutil/netutil.h"
#endif

struct nprotoent {
  const char *p_name;
  u16 p_proto;
};

```

这段代码是一个函数，名为 `addprotocolsfromservmask`，它接收一个字符数组 `mask` 和一个字符指针 `porttbl`，然后返回一个指向 `struct nprotoent` 类型的指针，该类型存储了所有网络协议的列表。

首先，该函数检查 `mask` 是否为空字符串，如果是，则表示所有协议都使用相同的端口号，函数返回一个空的 `struct nprotoent` 类型的指针。如果不是，函数接下来定义了一系列函数，用于从 `porttbl` 中获取每个端口号对应的网络协议。

具体来说，函数定义了两个名为 `ipproto2str` 和 `ipproto2str_uc` 的函数，它们都接受一个协议类型 `p`，并将其转换为字符串表示。这些函数根据 `p` 的值确定使用哪种协议，并将其转换为字符串表示。

另外，该函数还定义了一个名为 `MAX_IPPROTOSTRLEN` 的常量，表示最大字符串长度，用于表示 IP 协议前缀。


```cpp
int addprotocolsfromservmask(char *mask, u8 *porttbl);
const struct nprotoent *nmap_getprotbynum(int num);
const struct nprotoent *nmap_getprotbyname(const char *name);

#define MAX_IPPROTOSTRLEN 4
#define IPPROTO2STR(p)		\
  ((p)==IPPROTO_TCP ? "tcp" :	\
   (p)==IPPROTO_UDP ? "udp" :	\
   (p)==IPPROTO_SCTP ? "sctp" :	\
   "n/a")
#define IPPROTO2STR_UC(p)	\
  ((p)==IPPROTO_TCP ? "TCP" :	\
   (p)==IPPROTO_UDP ? "UDP" :	\
   (p)==IPPROTO_SCTP ? "SCTP" :	\
   "N/A")

```

这段代码是一个预处理指令，它的作用是在源代码文件被编译之前，检查其中是否定义了某些特定变量或函数。

具体来说，如果源代码文件中定义了名为"debug"的变量，并且这个变量被声明为"const"，那么这段代码会阻止编译器编译该源文件，因为在编译之前需要确认该变量确实被定义为"const"。

#endif指令是一个预处理指令，用于检查源代码文件中定义的特定变量或函数是否符合某些特定要求。当满足要求时，#endif指令会阻止编译器编译该源文件，因此在实际应用中需要谨慎使用。


```cpp
#endif


```

Nmap [![Build Status](https://travis-ci.org/nmap/nmap.svg?branch=master)](https://travis-ci.org/nmap/nmap) [![Language grade: C/C++](https://img.shields.io/lgtm/grade/cpp/g/nmap/nmap.svg?logo=lgtm&logoWidth=18)](https://lgtm.com/projects/g/nmap/nmap/context:cpp) [![Language grade: Python](https://img.shields.io/lgtm/grade/python/g/nmap/nmap.svg?logo=lgtm&logoWidth=18)](https://lgtm.com/projects/g/nmap/nmap/context:python) [![Total alerts](https://img.shields.io/lgtm/alerts/g/nmap/nmap.svg?logo=lgtm&logoWidth=18)](https://lgtm.com/projects/g/nmap/nmap/alerts/)
====

Nmap is released under a custom license, which is based on (but not compatible
with) GPLv2. The Nmap license allows free usage by end users, and we also offer
a commercial license for companies that wish to redistribute Nmap technology
with their products. See [Nmap Copyright and Licensing](https://nmap.org/book/man-legal.html)
for full details.

The latest version of this software as well as binary installers for Windows,
macOS, and Linux (RPM) are available from
[Nmap.org](https://nmap.org/download.html)

Full documentation is also available
[on the Nmap.org website](https://nmap.org/docs.html).

Questions and suggestions may be sent to
[the Nmap-dev mailing list](https://nmap.org/mailman/listinfo/dev).

Installing
----------
Ideally, you should be able to just type:

    ./configure
    make
    make install

For far more in-depth compilation, installation, and removal notes, read the
[Nmap Install Guide](https://nmap.org/book/install.html) on Nmap.org.

Using Nmap
----------
Nmap has a lot of features, but getting started is as easy as running `nmap
scanme.nmap.org`. Running `nmap` without any parameters will give a helpful
list of the most common options, which are discussed in depth in [the man
page](https://nmap.org/book/man.html). Users who prefer a graphical interface
can use the included [Zenmap front-end](https://nmap.org/zenmap/).

Contributing
------------
Information about filing bug reports and contributing to the Nmap project can
be found in the [HACKING](HACKING) and [CONTRIBUTING.md](CONTRIBUTING.md)
files.


# `scan_engine.h`

This is a text that discusses the Nmap software and its licensing. Nmap is a network scanner that can be used to find vulnerabilities in systems, networks, and websites. The licensing for Nmap is a topic of discussion.

The text states that Nmap has a public source license, which allows users to use, modify, and redistribute the software for any purpose, as long as they follow certain guidelines. The license also has provisions that prohibit companies from using the software in commercial products without Nmap's permission.

However, the text also mentions that Nmap sells a special Nmap OEM edition license with more permissive terms. This license allows users to use, modify, and redistribute the software in commercial products, provided they have the special license. The text encourages users to contact Nmap for more information about the availability of this license.

The text also mentions that the official Nmap Windows builds include the Npcap software, which is for packet capture and transmission. The Npcap software has its own license terms that prohibit redistribution without special permission.

Finally, the text emphasizes that the free version of Nmap is distributed with the hope that it will be useful, but WITHOUT ANY WARRANTY; the implied warranty of merchantability, and commercial support are all available through the Npcap OEM program.


```cpp

/***************************************************************************
 * scan_engine.h -- Includes much of the "engine" functions for scanning,  *
 * such as ultra_scan.  It also includes dependent functions such as       *
 * those for collecting SYN/connect scan responses.                        *
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



该代码是一个C语言的程序，定义了一个名为“scan_engine.h”的头文件。从该头文件中可以看出，它导入了两个名为“scan_lists.h”和“probespec.h”的头文件，这两个头文件可能包含一些通用的数据结构和函数。

该程序还导入了来自“timing.h”的头文件，可能用于在扫描过程中记录时间。

在头文件中定义了一个名为“scan_engine”的函数，它接受一个“Packet”类型的参数。函数的作用是解析扫描结果，并将其存储在“scn_list”中，以便后续的分析和处理。

函数首先创建了一个名为“scn_list”的链表，用于存储扫描结果。然后，使用“pcap.h”库中的“parse_pcap_file”函数读取输入的包。函数根据标记的值，将每个包的协议切割成相应的数据，并将数据添加到链表中。

最后，函数还创建了一个“timing”记录器，用于记录扫描过程的时间。在函数的结束部分，将记录器归零并返回。

该程序的作用是定义了一个扫描引擎，可以读取网络数据包，将其解析为相应的数据，并将数据存储在链表中。同时，还记录了扫描过程的时间，以便后续的分析和处理。


```cpp
/* $Id$ */

#ifndef SCAN_ENGINE_H
#define SCAN_ENGINE_H

#include "scan_lists.h"
#include "probespec.h"

#include <dnet.h>

#include "timing.h"

#include <pcap.h>
#include <list>
#include <vector>
```

这段代码定义了一个名为Target的类，用于实现Nmap扫描功能。它包含了一个包含Target类对象的std::vector，一个名为ports的std::vector，包含一个指向扫描列表的结构体，一个名为scantype的整数，和一个名为to的指向timeout_info结构体的指针。

在头文件中，它引入了<set>和<algorithm>头文件。

在类中，它实现了一个名为ultra_scan的函数，该函数接受一个std::vector<Target *>作为第一个参数，一个指向scan_lists的结构体作为第二个参数，一个名为stype的整数作为第三个参数，和一个指向timeout_info结构体的指针作为第四个参数。

函数内部包含以下语句：

1. 引入了<set>和<algorithm>头文件。
2. 定义了一个名为ports的std::vector，用于存储目标列表。
3. 定义了一个名为scantype的整数，用于确定Nmap扫描时需要扫描的类型。
4. 定义了一个名为to的指向timeout_info结构体的指针，用于存储Nmap扫描所需的时间超时信息。
5. 实现了一个ultra_scan函数，用于实现Nmap扫描功能。该函数首先将输入的扫描列表和扫描类型存储到std::vector中。然后，使用算法库中的<algorithm>库中的find_host函数来查找目标列表中与扫描类型匹配的第一个目标，并设置它。接着，使用algorithm库中的<parallel>函数并传入std::vector和find_host函数的返回值，来计算并设置每个目标扫描所需的超时时间。最后，将计算出的扫描列表和扫描类型存储到返回对象中，并使用is_valid函数检查返回对象是否为有效目标列表，使用is_alive函数检查目标是否还能扫描，如果扫描失败或目标不可达，则设置is_alive为真，并将错误信息存储到to对象中。
6. 使用is_alive函数检查目标是否还能扫描，如果扫描失败或目标不可达，则设置is_alive为真，并将错误信息存储到to对象中。


```cpp
#include <set>
#include <algorithm>
class Target;

/* 3rd generation Nmap scanning function.  Handles most Nmap port scan types */
void ultra_scan(std::vector<Target *> &Targets, const struct scan_lists *ports,
                stype scantype, struct timeout_info *to = NULL);

/* Determines an ideal number of hosts to be scanned (port scan, os
   scan, version detection, etc.) in parallel after the ping scan is
   completed.  This is a balance between efficiency (more hosts in
   parallel often reduces scan time per host) and results latency (you
   need to wait for all hosts to finish before Nmap can spit out the
   results).  Memory consumption usually also increases with the
   number of hosts scanned in parallel, though rarely to significant
   levels. */
```



这段代码定义了一个名为 determineScanGroupSize 的函数，其作用是确定扫描组大小。函数的输入参数包括已扫描的主机数量 numHostsScanned 和一个指向 struct scanLists 的指针 ports。

函数实现中，首先定义了一个名为 UltraScanInfo 的结构体，该结构体可能包含了用于扫描的主机信息。接着，函数实现了一个名为 ConnectProbe 的类，该类可能用于在服务器和客户端之间建立连接。

DetermineScanGroupSize函数的具体实现如下：

1. 函数创建一个名为 ultscanhierInfo 的变量，用于存储 UltraScanInfo 结构体。

2. 函数创建一个名为 scanhierList 的指向 struct scanLists 的指针，并将其存储在 UltraScanInfo 结构体中。

3. 函数计算 numGroups 和 numGroupMax，用于确定扫描组的最大大小。

4. 函数根据 numGroups 和 numGroupMax 创建一个名为 scanGroups 的数组，用于存储扫描组信息。

5. 函数将 scanhierList 指向的指针传递给 connectProbe，并在函数内部使用 scanhierList 中的指针来设置连接客户端的端口。

6. 函数返回 scanGroups 的大小。

ConnectProbe类可能包含如下实现：

1. 定义了一个名为 constructor 的构造函数，用于初始化 ConnectProbe 对象。

2. 定义了一个名为 destructor 的 destructor，用于在对象被删除时清理资源。

3. 定义了一个名为 connectServer 的方法，用于连接服务器。该方法可能接受一个端口和一个套接字描述符作为参数。

4. 定义了一个名为 connectClient 的方法，用于连接客户端。该方法可能接受一个端口和一个套接字描述符作为参数。


```cpp
int determineScanGroupSize(int hosts_scanned_so_far,
                           const struct scan_lists *ports);

class UltraScanInfo;

struct ppkt { /* Beginning of ICMP Echo/Timestamp header         */
  u8 type;
  u8 code;
  u16 checksum;
  u16 id;
  u16 seq;
};

class ConnectProbe {
public:
  ConnectProbe();
  ~ConnectProbe();
  int sd; /* Socket descriptor used for connection.  -1 if not valid. */
};

```

此代码定义了三种结构体：IPExtraProbeData_icmp，IPExtraProbeData_tcp和IPExtraProbeData_udp以及一种结构体IPExtraProbeData_sctp。

IPExtraProbeData_icmp结构体包含一个标识符ident。

IPExtraProbeData_tcp结构体包含一个源端口号(sport)和一个序列号(seq)，用于标识数据包的顺序。

IPExtraProbeData_udp结构体包含一个源端口号(sport)，用于标识数据包的顺序。

IPExtraProbeData_sctp结构体包含一个源端口号(sport)，一个虚拟标志位(vtag)，用于标识数据包的顺序。

这些结构体可能用于在网络中发送诊断测试数据包，以便诊断网络设备的工作情况。


```cpp
struct IPExtraProbeData_icmp {
  u16 ident;
};

struct IPExtraProbeData_tcp {
  u16 sport;
  u32 seq; /* host byte order (like the other fields */
};

struct IPExtraProbeData_udp {
  u16 sport;
};

struct IPExtraProbeData_sctp {
  u16 sport;
  u32 vtag;
};

```

这段代码定义了一个名为IPExtraProbeData的结构体，它用于表示IP扩展代理数据。

该结构体包含以下成员：

1. u32 ipid：32位的IPID，以 host byte order 格式存储。
2. union {
 struct IPExtraProbeData_icmp icmp;
 struct IPExtraProbeData_tcp tcp;
 struct IPExtraProbeData_udp udp;
 struct IPExtraProbeData_sctp sctp;
 } pd;
}

3. union _tryno_u：这是一个联合类型，由一个或多个结构体成员组成。这里包含一个名为 fields 的结构体成员，它是一个包含 isPing 和 seqnum 的联合类型。
4. u8 opaque：这是一个未初始化的8位整数类型的成员，它可以被用来存储一个值，但在此处没有明确的使用。

IPExtraProbeData结构体可以用来表示网络探测信息，如 ping 或 scanprobe。通过将 isPing 和 seqnum 设置为1或初始化，可以启用或禁用相应的探测。此外，opaque成员可以被用来存储一个字符串类型的数据，但在此处没有明确的使用。


```cpp
struct IPExtraProbeData {
  u32 ipid; /* host byte order */
  union {
    struct IPExtraProbeData_icmp icmp;
    struct IPExtraProbeData_tcp tcp;
    struct IPExtraProbeData_udp udp;
    struct IPExtraProbeData_sctp sctp;
  } pd;
};

union _tryno_u {
  struct {
  u8 isPing : 1; // Is this a ping, not a scanprobe?
  u8 seqnum : 7; // Sequence number, 0-127
  } fields;
  u8 opaque;
};
```

This is a function prototype for a `ConnectProbe` struct that contains information about a network probe, such as its protocol type (e.g. IPPROTO_TCP, IPPROTO_UDP), the sequence number of this probe, and a flag indicating whether this is a ping or scanprobe. The `ConnectProbe` struct also contains information about the probe's active state, such as whether it has timed out or been retransmitted, as well as the time the last probe was sent if it is a retransmit.


```cpp
typedef union _tryno_u tryno_t;

/* At least for now, I'll just use this like a struct and access
   all the data members directly */
class UltraProbe {
public:
  UltraProbe();
  ~UltraProbe();
  enum UPType { UP_UNSET, UP_IP, UP_CONNECT, UP_ARP, UP_ND } type; /* The type of probe this is */

  /* Sets this UltraProbe as type UP_IP and creates & initializes the
     internal IPProbe.  The relevant probespec is necessary for setIP
     because pspec.type is ambiguous with just the ippacket (e.g. a
     tcp packet could be PS_PROTO or PS_TCP). */
  void setIP(const u8 *ippacket, u32 iplen, const probespec *pspec);
  /* Sets this UltraProbe as type UP_CONNECT, preparing to connect to given
   port number*/
  void setConnect(u16 portno);
  /* Pass an arp packet, including ethernet header. Must be 42bytes */
  void setARP(const u8 *arppkt, u32 arplen);
  void setND(const u8 *ndpkt, u32 ndlen);
  // The 4 accessors below all return in HOST BYTE ORDER
  // source port used if TCP, UDP or SCTP
  u16 sport() const;
  // destination port used if TCP, UDP or SCTP
  u16 dport() const;
  u32 ipid() const {
    return probes.IP.ipid;
  }
  u16 icmpid() const; // ICMP ident if protocol is ICMP
  u32 tcpseq() const; // TCP sequence number if protocol is TCP
  u32 sctpvtag() const; // SCTP vtag if protocol is SCTP
  /* Number, such as IPPROTO_TCP, IPPROTO_UDP, etc. */
  u8 protocol() const {
    return mypspec.proto;
  }
  ConnectProbe *CP() const {
    return probes.CP;  // if type == UP_CONNECT
  }
  // Arpprobe removed because not used.
  //  ArpProbe *AP() { return probes.AP; } // if UP_ARP
  // Returns the protocol number, such as IPPROTO_TCP, or IPPROTO_UDP, by
  // reading the appropriate fields of the probespec.

  /* Get general details about the probe */
  const probespec *pspec() const {
    return &mypspec;
  }

  /* Returns true if the given tryno matches this probe. */
  bool check_tryno(u8 tryno) const {
    return tryno == this->tryno.opaque;
  }

  /* Helper for checking protocol/port match from a packet. */
  bool check_proto_port(u8 proto, u16 sport_or_icmpid, u16 dport) const;

  /* tryno/pingseq, depending on what type of probe this is (ping vs scanprobe) */
  tryno_t tryno; /* Try (retransmission) number of this probe */
  /* If true, probe is considered no longer active due to timeout, but it
     may be kept around a while, just in case a reply comes late */
  bool timedout;
  /* A packet may be timedout for a while before being retransmitted due to
     packet sending rate limitations */
  bool retransmitted;

  struct timeval sent;
  /* Time the previous probe was sent, if this is a retransmit (tryno > 0) */
  struct timeval prevSent;
  bool isPing() const {
    return tryno.fields.isPing;
  }
  u8 get_tryno() const {
    return tryno.fields.seqnum;
  }

```



这段代码是一个C++类，名为“ConnectScanInfo”。

该类包含一个名为“probespec”的私有成员变量，该变量被初始化为通过set*函数填充的IPExtraProbeData类型。

该类还包含一个名为“union”的私有成员，该成员包含一个名为“IP”的成员和一个名为“CP”的成员，还有一个名为“Arp”的成员。

该类还定义了一个名为“ConnectScanInfo”的类成员函数，该函数用于初始化、监控和设置连接扫描器的状态。

该函数名为“watchSD”，用于检查是否正在监视一个socket描述符。

该函数名为“clearSD”，用于从fd_sets和maxValidSD中清除正在监视的socket描述符。

该函数名为“sendOK”，用于尝试获取一个可用的socket描述符，并返回是否成功。

该函数名为“maxValidSD”，用于存储在任何fd_sets中可以使用的最大socket描述符。

该函数名为“fds_read”，用于存储当前正在监视的socket描述符的fd_set。

该函数名为“fds_write”，用于存储当前正在写入的socket描述符的fd_set。

该函数名为“fds_except”，用于存储当前正在监视的socket描述符的fd_set。

该函数名为“numSDs”，用于存储正在被监视的socket描述符的数量。

该函数名为“getSocket”，用于返回正在被监视的socket描述符中包含的第一个socket的编号。


```cpp
private:
  probespec mypspec; /* Filled in by the appropriate set* function */
  union {
    IPExtraProbeData IP;
    ConnectProbe *CP;
    //    ArpProbe *AP;
  } probes;
};

/* Global info for the connect scan */
class ConnectScanInfo {
public:
  ConnectScanInfo();
  ~ConnectScanInfo();

  /* Watch a socket descriptor (add to fd_sets and maxValidSD).  Returns
     true if the SD was absent from the list, false if you tried to
     watch an SD that was already being watched. */
  bool watchSD(int sd);

  /* Clear SD from the fd_sets and maxValidSD.  Returns true if the SD
   was in the list, false if you tried to clear an sd that wasn't
   there in the first place. */
  bool clearSD(int sd);
  /* Try to get a socket that's good for select(). Return true if it worked;
   * false if it didn't. */
  bool sendOK();
  int maxValidSD; /* The maximum socket descriptor in any of the fd_sets */
  fd_set fds_read;
  fd_set fds_write;
  fd_set fds_except;
  int numSDs; /* Number of socket descriptors being watched */
  int getSocket();
```

This is a structure definition for an internal variable in the OpenWrt GSS interface. It contains global timing information for the scan, including the current time without the gettimeofday() function. The variable contains the following fields:

1. `timing`: contains the current time in the GSS era, based on the server's timestamp and the number of probes sent.
2. `t`: contains the timeout information for the scan, including the maximum waiting time and the maximum number of probes to send.
3. `numtargets`: contains the total number of hosts that were scanned, including the number of finished and incomplete hosts.
4. `numprobes`: contains the number of probes that were sent on each host.
5. `probes_sent`: keeps track of the number of probes that were sent.
6. `lastrcvd`: stores the most recently received probe response time.
7. `lastping_sent`: stores the time the most recent ping was sent.
8. `lastping_sent_numprobes`: stores the number of probes that were sent in the last ping.
9. `send_no_earlier_than` and `send_no_later_than`: controls the minimum and maximum rate of sending probes, respectively.
10. `connect_scan_info`: contains information about the scan, including the host to which global pings are sent.
11. `pinghost`: contains the host to which global pings are sent.
12. `last_wait`: stores the time at which the last probe response was received.
13. `probes_sent_at_last_wait`: stores the number of probes that were sent in the last wait.
14. `num_hosts_timedout`: stores the number of hosts that timed out during the scan, or were already timedout.

This variable is used by the GSS implementation to perform the scan and update the global timing information.


```cpp
private:
  int nextSD;
  int maxSocketsAllowed; /* No more than this many sockets may be created @once */
};

class HostScanStats;

/* These are ultra_scan() statistics for the whole group of Targets */
class GroupScanStats {
public:
  struct timeval timeout; /* The time at which we abort the scan */
  /* Most recent host tested for sendability */
  struct sockaddr_storage latestip;
  GroupScanStats(UltraScanInfo *UltraSI);
  ~GroupScanStats();
  void probeSent(unsigned int nbytes);
  /* Returns true if the GLOBAL system says that sending is OK. */
  bool sendOK(struct timeval *when) const;
  /* Total # of probes outstanding (active) for all Hosts */
  int num_probes_active;
  UltraScanInfo *USI; /* The USI which contains this GSS.  Use for at least
                         getting the current time w/o gettimeofday() */
  struct ultra_timing_vals timing;
  struct timeout_info to; /* Group-wide packet rtt/timeout info */
  int numtargets; /* Total # of targets scanned -- includes finished and incomplete hosts */
  int numprobes; /* Number of probes/ports scanned on each host */
  /* The last time waitForResponses finished (initialized to GSS creation time */
  int probes_sent; /* Number of probes sent in total.  This DOES include pings and retransmissions */

  /* The most recently received probe response time -- initialized to scan
     start time. */
  struct timeval lastrcvd;
  /* The time the most recent ping was sent (initialized to scan begin time) */
  struct timeval lastping_sent;
  /* Value of numprobes_sent at lastping_sent time -- to ensure that we don't
     send too many pings when probes are going slowly. */
  int lastping_sent_numprobes;

  /* These two variables control minimum- and maximum-rate sending (--min-rate
     and --max-rate). send_no_earlier_than is for --max-rate and
     send_no_later_than is for --min-rate; they have effect only when the
     respective command-line option is given. An attempt is made to keep the
     sending rate within the interval, however for send_no_later_than it is not
     guaranteed. */
  struct timeval send_no_earlier_than;
  struct timeval send_no_later_than;

  /* The host to which global pings are sent. This is kept updated to be the
     most recent host that was found up. */
  HostScanStats *pinghost;

  struct timeval last_wait;
  int probes_sent_at_last_wait;
  // number of hosts that timed out during scan, or were already timedout
  int num_hosts_timedout;
  ConnectScanInfo *CSI;
};

```

这两段代码定义了两个结构体，分别是send_delay_nfo和rate_limit_detection_nfo。

send_delay_nfo定义了一个延迟函数，这个延迟函数在第一次调用时会被设置为指定的毫秒数，然后用于在每次 probe（即数据包发送尝试）之间延迟一段时间，这个延迟时间可以设置比例来控制延迟时间的减少速度。这个结构体还包含两个变量，goodRespSinceDelayChanged和droppedRespSinceDelayChanged，用于记录在最后一次调用延迟时间之前成功和失败的 probes数量。最后，还有一个变量last_boost，用于记录最后一次调用的时间，用于计算发送第一个数据包的时间。

rate_limit_detection_nfo定义了一个rate_limit_detection_nfo结构体，用于测试网络中的速率限制。它包含两个变量，max_tryno_sent和rld_waiting，分别表示已发送的最大尝试次数和当前正在等待的RLD（Regenerative Logic Device，生成式逻辑设备）时间。另外，还有一个变量rld_waittime，用于在RLD正在等待时可以发送数据的时间。


```cpp
struct send_delay_nfo {
  unsigned int delayms; /* Milliseconds to delay between probes */
  /* The number of successful and dropped probes since the last time the delay
     was changed. The ratio controls when the rate drops. */
  unsigned int goodRespSinceDelayChanged;
  unsigned int droppedRespSinceDelayChanged;
  struct timeval last_boost; /* Most recent time of increase to delayms.  Init to creation time. */
};

/* To test for rate limiting, there is a delay in sending the first packet
   of a certain retransmission number.  These values help track that. */
struct rate_limit_detection_nfo {
  unsigned int max_tryno_sent; /* What is the max tryno we have sent so far (starts at 0) */
  bool rld_waiting; /* Are we currently waiting due to RLD? */
  struct timeval rld_waittime; /* if RLD waiting, when can we send? */
};

```

This is a function defined in the Linux kernel source code that is responsible for managing the pings and trransmits of a host. It controls the number of pings sent to the host and the scan delay for the host, based on the scan type and network conditions.

The function has several parameters:

- lastping\_sent\_numprobes: The number of pings sent by the host in the last 10 seconds.
- lastprobe\_sent: The most recent probe that was sent by the host, including pings. This is used to calculate the scan delay.
- allowedTryno: The maximum number of probes that may be sent before the host is considered out of order. This may be increased or decreased based on the scan type and network conditions.
- mayincrease: A flag that indicates whether the allowedTryno may be increased.
- capped: A boolean indicating whether the number of probes sent should be limited.
- max\_successful\_tryno: The maximum number of probes that have produced useful results.
- ports\_finished: The number of ports of the host that have been determined.
- numprobes\_sent: The number of port probes that have been sent to the host.
- nxtpseq: The next ping sequence number that starts at zero and goes up to 127.

The function also has two subfunctions:

- boostScanDelay: A function that boosts the scan delay for the host by adjusting the values of the scan delay parameter (127 is the maximum allowed scan delay).
- rate\_limit\_detection: A function that sets the rate limit detection threshold for the host based on the number of dropped packets and the scan delay.


```cpp
/* The ultra_scan() statistics that apply to individual target hosts in a
   group */
class HostScanStats {
public:
  Target *target; /* A copy of the Target that these stats refer to. */
  HostScanStats(Target *t, UltraScanInfo *UltraSI);
  ~HostScanStats();
  bool freshPortsLeft() const; /* Returns true if there are ports remaining to probe */
  int numFreshPortsLeft() const; /* Returns the number of ports remaining to probe */
  int next_portidx; /* Index of the next port to probe in the relevant
                       ports array in USI.ports */
  bool sent_arp; /* Has an ARP probe been sent for the target yet? */

  /* massping state. */
  /* The index of the next ACK port in o.ping_ackprobes to probe during ping
     scan. */
  int next_ackportpingidx;
  /* The index of the next SYN port in o.ping_synprobes to probe during ping
     scan. */
  int next_synportpingidx;
  /* The index of the next UDP port in o.ping_udpprobes to probe during ping
     scan. */
  int next_udpportpingidx;
  /* The index of the next SCTP port in o.ping_protoprobes to probe during ping
     scan. */
  int next_sctpportpingidx;
  /* The index of the next IP protocol in o.ping_protoprobes to probe during ping
     scan. */
  int next_protoportpingidx;
  /* Whether we have sent an ICMP echo request. */
  bool sent_icmp_ping;
  /* Whether we have sent an ICMP address mask request. */
  bool sent_icmp_mask;
  /* Whether we have sent an ICMP timestamp request. */
  bool sent_icmp_ts;

  /* Have we warned that we've given up on a port for this host yet? Only one
     port per host is reported. */
  bool retry_capped_warned;

  void probeSent(unsigned int nbytes);

  /* How long I am currently willing to wait for a probe response
     before considering it timed out.  Uses the host values from
     target if they are available, otherwise from gstats.  Results
     returned in MICROseconds.  */
  unsigned long probeTimeout() const;

  /* How long I'll wait until completely giving up on a probe.
     Timedout probes are often marked as such (and sometimes
     considered a drop), but kept in the list juts in case they come
     really late.  But after probeExpireTime(), I don't waste time
     keeping them around. Give in MICROseconds */
  unsigned long probeExpireTime(const UltraProbe *probe) const;
  /* Returns OK if sending a new probe to this host is OK (to avoid
     flooding). If when is non-NULL, fills it with the time that sending
     will be OK assuming no pending probes are resolved by responses
     (call it again if they do).  when will become now if it returns
     true. */
  bool sendOK(struct timeval *when) const;

  /* If there are pending probe timeouts, fills in when with the time of
     the earliest one and returns true.  Otherwise returns false and
     puts now in when. */
  bool nextTimeout(struct timeval *when) const;
  UltraScanInfo *USI; /* The USI which contains this HSS */

  /* Removes a probe from probes_outstanding, adjusts HSS and USS
     active probe stats accordingly, then deletes the probe. */
  void destroyOutstandingProbe(std::list<UltraProbe *>::iterator probeI);

  /* Removes all probes from probes_outstanding using
     destroyOutstandingProbe. This is used in ping scan to quit waiting
     for responses once a host is known to be up. Invalidates iterators
     pointing into probes_outstanding. */
  void destroyAllOutstandingProbes();

  /* Mark an outstanding probe as timedout.  Adjusts stats
     accordingly.  For connect scans, this closes the socket. */
  void markProbeTimedout(std::list<UltraProbe *>::iterator probeI);

  /* New (active) probes are appended to the end of this list.  When a
     host times out, it will be marked as such, but may hang around on
     the list for a while just in case a response comes in.  So use
     num_probes_active to learn how many active (not timed out) probes
     are outstanding.  Probes on the bench (reached the current
     maximum tryno and expired) are not counted in
     probes_outstanding.  */
  std::list<UltraProbe *> probes_outstanding;
  /* The number of probes in probes_outstanding, minus the inactive (timed out) ones */
  unsigned int num_probes_active;
  /* Probes timed out but not yet retransmitted because of congestion
     control limits or because more retransmits may not be
     necessary.  Note that probes on probe_bench are not included
     in this value. */
  unsigned int num_probes_waiting_retransmit;
  unsigned int num_probes_outstanding() const {
    return probes_outstanding.size();
  }

  /* The bench is a stock of probes (compacted into just the
     probespec) that have met the current maximum tryno, and are on
     ice until that tryno increases (so we can retransmit again), or
     solidifies (so we can mark the port firewalled or whatever).  The
     tryno of bench members is bench_tryno.  If the maximum tryno
     increases, everyone on the bench is moved to the retry_stack.
   */
  std::vector<probespec> probe_bench;
  unsigned int bench_tryno; /* # tryno of probes on the bench */
  /* The retry_stack are probespecs that were on the bench but are now
     slated to be retried.  It is kept sorted such that probes with highest
     retry counts are on top, ready to be taken first. */
  std::vector<probespec> retry_stack;
  /* retry_stack_tries MUST BE KEPT IN SYNC WITH retry_stack.
     retry_stack_tries[i] is the number of completed retries for the
     probe in retry_stack[i] */
  std::vector<u8> retry_stack_tries;
  /* tryno of probes on the retry queue */
  /* Moves the given probe from the probes_outstanding list, to
     probe_bench, and decrements num_probes_waiting_retransmit accordingly */
  void moveProbeToBench(std::list<UltraProbe *>::iterator probeI);
  /* Dismiss all probe attempts on bench -- the ports are marked
     'filtered' or whatever is appropriate for having no response */
  void dismissBench();
  /* Move all members of bench to retry_stack for probe retransmission */
  void retransmitBench();

  bool completed() const; /* Whether or not the scan of this Target has completed */
  struct timeval completiontime; /* When this Target completed */

  /* This function provides the proper cwnd and ssthresh to use.  It
     may differ from versions in timing member var because when no
     responses have been received for this host, may look at others in
     the group.  For CHANGING this host's timing, use the timing
     memberval instead. */
  void getTiming(struct ultra_timing_vals *tmng) const;
  struct ultra_timing_vals timing;
  /* The most recently received probe response time -- initialized to scan start time. */
  struct timeval lastrcvd;
  struct timeval lastping_sent; /* The time the most recent ping was sent (initialized to scan begin time) */

  /* Value of numprobes_sent at lastping_sent time -- to ensure that we
     don't send too many pings when probes are going slowly. */
  int lastping_sent_numprobes;
  struct timeval lastprobe_sent; /* Most recent probe send (including pings) by host.  Init to scan begin time. */
  /* gives the maximum try number (try numbers start at zero and
     increments for each retransmission) that may be used, based on
     the scan type, observed network reliability, timing mode, etc.
     This may change during the scan based on network traffic.  If
     capped is not null, it will be filled with true if the tryno is
     at its upper limit.  That often calls for a warning to be issued,
     and marking of remaining timedout ports firewalled or whatever is
     appropriate.  If mayincrease is non-NULL, it is set to whether
     the allowedTryno may increase again.  If it is false, any probes
     which have reached the given limit may be dealt with. */
  unsigned int allowedTryno(bool *capped, bool *mayincrease) const;

  /* Provides the next ping sequence number.  This starts at zero, goes
   up to 127, then wraps around back to 0. */
  u8 nextPingSeq() {
    // Has to fit in 7 bits: tryno.fields.seqnum
    nxtpseq = (nxtpseq + 1) % 0x80;
    return nxtpseq;
  }
  /* This is the highest try number that has produced useful results
     (such as port status change). */
  unsigned int max_successful_tryno;
  int ports_finished; /* The number of ports of this host that have been determined */
  int numprobes_sent; /* Number of port probes (not counting pings, but counting retransmits) sent to this host */
  /* Boost the scan delay for this host, usually because too many packet
     drops were detected. */
  void boostScanDelay();
  struct send_delay_nfo sdn;
  struct rate_limit_detection_nfo rld;

```

这段代码定义了一个名为`ultra_scan_performance_vars`的`struct`类型，其中包括了一些与`ultra_scan`相关的性能设置。

该`struct`包含以下成员：

- `ping_magnifier`：当从目标主机发送一个扫描请求时，此成员记录了成功获得回应的数量。值越大，意味着成功回应的数量越多，因此输入会越少。

- `pingtime`：尝试发送一个扫描请求的时间，以秒为单位。

- `tryno_cap`：允许发送的最大尝试次数。如果超过这个值，发送扫描请求将被视为异常，并可能导致程序崩溃。

该`struct`还包含一个名为`init`的函数，该函数在`ultra_scan_performance_vars`被实例化时执行，初始化该`struct`的成员变量。


```cpp
private:
  u8 nxtpseq; /* the next scanping sequence number to use */
};

/* A few extra performance tuning parameters specific to ultra_scan. */
struct ultra_scan_performance_vars : public scan_performance_vars {
  /* When a successful ping response comes back, it counts as this many
     "normal" responses, because the fact that pings are necessary means
     we aren't getting much input. */
  int ping_magnifier;
  /* Try to send a scanping if no response has been received from a target host
     in this many usecs */
  int pingtime;
  unsigned int tryno_cap; /* The maximum trynumber (starts at zero) allowed */

  void init();
};

```

HostScanStats is a struct that keeps track of the host information for a network scan.

This struct has several member variables:

* incompleteHosts: This is a vector of HostScanStats structs that have an incomplete
	+ IP address. This is useful when we want to know the number of hosts that have
		incomplete information.
* completedHosts: This is a vector of HostScanStats structs that have a completed
	+ IP address. This is useful when we want to know the number of hosts that have
		complete information.
* lastCompletedHostRemoval: This is the timestamp when we last removed a completed host
	+ from the completedHosts vector.
* incompleteHostsCount: This is the number of incomplete host blocks in the
	+ incompleteHosts vector.
* scanMonitor: This is a pointer to a ScanMonitor struct that allows us to log
	+ information about the scan.
* rawsd: This is the raw socket descriptor.
* useRawSockets: This is a boolean value that is used to control whether we should use
	+ raw sockets for sending data over the network.
* ports: This is a vector of port numbers that we want to scan.
* rawSockets: This is a vector of raw socket objects that we want to use for sending
	+ data over the network.
* ethFlags: This is a bit field that is used to configure the Ethernet
	+ flags for the raw sockets.
* icsFlags: This is a bit field that is used to configure the Internet Control
	+ semantics flag for the raw sockets.
* UUID: This is a universally unique identifier (UUID) that is used to identify the
	+ scan.
* scanDuration: This is the duration of the scan in seconds.

This struct also has several member function pointers:

* logScan: This is a function that logs information about the scan.
* logCurrent: This is a function that logs information about the current scan.
* scanCompleted: This is a function that removes completed hosts from the incompleteHosts
	+ vector.
* logRemoveCompleted: This is a function that logs information about the removal of
	+ completed hosts from the incompleteHosts vector.


```cpp
struct HssPredicate {
public:
  int operator() (const HostScanStats *lhs, const HostScanStats *rhs) const;
  static struct sockaddr_storage *ss;
};

class UltraScanInfo {
public:
  UltraScanInfo();
  UltraScanInfo(std::vector<Target *> &Targets, const struct scan_lists *pts, stype scantype) {
    Init(Targets, pts, scantype);
  }
  ~UltraScanInfo();
  /* Must call Init if you create object with default constructor */
  void Init(std::vector<Target *> &Targets, const struct scan_lists *pts, stype scantp);

  unsigned int numProbesPerHost() const;

  /* Consults with the group stats, and the hstats for every
     incomplete hosts to determine whether any probes may be sent.
     Returns true if they can be sent immediately.  If when is non-NULL,
     it is filled with the next possible time that probes can be sent
     (which will be now, if the function returns true */
  bool sendOK(struct timeval *tv) const;
  stype scantype;
  bool tcp_scan; /* scantype is a type of TCP scan */
  bool udp_scan;
  bool sctp_scan; /* scantype is a type of SCTP scan */
  bool prot_scan;
  bool ping_scan; /* Includes trad. ping scan & arp scan */
  bool ping_scan_arp; /* ONLY includes arp ping scan */
  bool ping_scan_nd; /* ONLY includes ND ping scan */
  bool noresp_open_scan; /* Whether no response means a port is open */

  /* massping state. */
  /* If ping_scan is true (unless ping_scan_arp is also true), this is the set
     of ping techniques to use (ICMP, raw ICMP, TCP connect, raw TCP, or raw
     UDP). */
  struct {
    unsigned int rawicmpscan: 1,
      connecttcpscan: 1,
      rawtcpscan: 1,
      rawudpscan: 1,
      rawsctpscan: 1,
      rawprotoscan: 1;
  } ptech;

  bool isRawScan() const;

  struct timeval now; /* Updated after potentially meaningful delays.  This can
                         be used to save a call to gettimeofday() */
  GroupScanStats *gstats;
  struct ultra_scan_performance_vars perf;
  /* A circular buffer of the incompleteHosts.  nextIncompleteHost() gives
     the next one.  The first time it is called, it will give the
     first host in the list.  If incompleteHosts is empty, returns
     NULL. */
  HostScanStats *nextIncompleteHost();
  /* Removes any hosts that have completed their scans from the incompleteHosts
     list, and remove any hosts from completedHosts which have exceeded their
     lifetime.  Returns the number of hosts removed. */
  int removeCompletedHosts();
  /* Find a HostScanStats by its IP address in the incomplete and completed
     lists.  Returns NULL if none are found. */
  HostScanStats *findHost(struct sockaddr_storage *ss) const;

  double getCompletionFraction() const;

  unsigned int numIncompleteHosts() const {
    return incompleteHosts.size();
  }
  /* Call this instead of checking for numIncompleteHosts() == 0 because it
     avoids a potential traversal of the list to find the size. */
  bool incompleteHostsEmpty() const {
    return incompleteHosts.empty();
  }
  bool numIncompleteHostsLessThan(unsigned int n) const;

  unsigned int numInitialHosts() const {
    return numInitialTargets;
  }

  void log_overall_rates(int logt) const;
  void log_current_rates(int logt, bool update = true);

  /* Any function which messes with (removes elements from)
     incompleteHosts may have to manipulate nextI */
  std::multiset<HostScanStats *, HssPredicate> incompleteHosts;
  /* Hosts are moved from incompleteHosts to completedHosts as they are
     completed. We keep them around because sometimes responses come back very
     late, after we consider a host completed. */
  std::multiset<HostScanStats *, HssPredicate> completedHosts;
  /* The last time we went through completedHosts to remove hosts */
  struct timeval lastCompletedHostRemoval;

  ScanProgressMeter *SPM;
  PacketRateMeter send_rate_meter;
  const struct scan_lists *ports;
  int rawsd; /* raw socket descriptor */
  pcap_t *pd;
  eth_t *ethsd;
  u32 seqmask; /* This mask value is used to encode values in sequence
                  numbers.  It is set randomly in UltraScanInfo::Init() */
  u16 base_port;
  const struct sockaddr_storage *SourceSockAddr() const { return &sourceSockAddr; }

```

这段代码定义了一个名为 `numInitialTargets` 的 `unsigned int` 类型的变量，它用于保存一个 `std::multiset<HostScanStats *, HssPredicate>::iterator` 类型的指针，这个指针用于遍历传递给 `uma_source_device_t` 类型的 `nextI` 所指向的 `HostScanStats` 结构体数组中的下一个目标。

接下来代码中定义了一个名为 `sourceSockAddr` 的 `struct sockaddr_storage` 类型的变量，用于保存 `HostScanStats` 结构体中的所有目标的相关信息。

接着，代码中定义了一个名为 `base_port` 的 `unsigned int` 类型的变量，用于保存当前初始化目标时所选择的源端口。为了确保在不同的 `uma_source_device_t` 实例中，初始化的源端口不会发生冲突，代码中通过递增 `base_port` 并将其最大值限制在 16 位以内来达到这个目的。

最后，代码中定义了一个名为 `per_probe_info` 的 `unsigned int` 类型的变量，用于保存当前 `HostScanStats` 结构体中的每个探针的信息。这个变量被定义为 `std::multiset<HostScanStats *, HssPredicate>::iterator` 类型的指针，这个指针用于遍历传递给 `uma_source_device_t` 类型的 `nextI` 所指向的 `HostScanStats` 结构体数组中的下一个目标，并将这个结构体中的所有信息进行编码，以便在目标发生更改时，能够正确地进行事件的跟踪。


```cpp
private:

  unsigned int numInitialTargets;
  std::multiset<HostScanStats *, HssPredicate>::iterator nextI;
  // All targets in an invocation will have the same source address.
  struct sockaddr_storage sourceSockAddr;
  /* We encode per-probe information like the tryno in the source
     port, for protocols that use ports. (Except when o.magic_port_set is
     true--then we honor the requested source port.) The tryno is
     encoded as offsets from base_port, a base source port number (see
     sport_encode and sport_decode). To avoid interpreting a late response from a
     previous invocation of ultra_scan as a response for the same port in the
     current invocation, we increase base_port by a healthy amount designed to be
     greater than any offset likely to be used by a probe, each time ultra_scan is
     run.

     If we don't increase the base port, then there is the risk of something like
     the following happening:
     1. Nmap sends an ICMP echo and a TCP ACK probe to port 80 for host discovery.
     2. Nmap receives an ICMP echo reply and marks the host up.
     3. Nmap sends a TCP SYN probe to port 80 for port scanning.
     4. Nmap finally receives a delayed TCP RST in response to its earlier ACK
     probe, and wrongly marks port 80 as closed. */

  /* Base port must be chosen so that there is room to add an 8-bit value (tryno)
   * without exceeding 16 bits. We increment modulo the largest prime number N
   * such that 33000 + N + 256 < 65536, which ensures no overlapping cycles. */
  // Nearest prime not exceeding 65536 - 256 - 33000:
```

这段代码定义了一个名为“ultra_timing_type”的枚举类型，用于表示计时统计是针对整个群组还是单个主机。其中，枚举类型“TIMING_HOST”表示对单个主机进行计时统计，而“TIMING_GROUP”表示对整个群组进行计时统计。

接下来，代码中定义了一个名为“increment_base_port”的函数，它改变了一个名为“base_port”的静态变量，将其值限定在一个安全的随机端口范围内，以避免与附近的过去或未来的“ultra_scan”调用的冲突。该函数的实现是通过在函数内部生成一个随机整数，将其加到“g_base_port”上，然后将结果限定在33000到33727之间，最后返回修改后的“g_base_port”。

最后，在代码的最后部分，通过枚举类型“TIMING_HOST”和“TIMING_GROUP”，来选择对单个主机还是整个群组进行计时统计，具体实现是通过调用函数“increment_base_port”来实现。


```cpp
#define PRIME_32K 32261
  /* Change base_port to a new number in a safe port range that is unlikely to
     conflict with nearby past or future invocations of ultra_scan. */
  static u16 increment_base_port() {
    static u16 g_base_port = 33000 + get_random_uint() % PRIME_32K;
    g_base_port = 33000 + (g_base_port - 33000 + 256) % PRIME_32K;
    return g_base_port;
  }

};

/* Whether this is storing timing stats for a whole group or an
   individual host */
enum ultra_timing_type { TIMING_HOST, TIMING_GROUP };

```

这段代码是一个名为 `ultrascan_port_probe_update` 的函数，它是 `ultrascan_probe_update` 函数的别名。它接受一个 `UltraScanInfo` 类型的参数 `USI`，一个 `HostScanStats` 类型的参数 `hss`，和一个指向 `UltraProbe` 类型的指针 `probeI`，一个整数参数 `newstate`，一个表示时间戳的 `timeval` 类型的参数 `rcvdtime`，以及一个布尔参数 `adjust_timing_hint`，如果为真，则调整接收时间戳的时间戳调整。

该函数的主要作用是更新 `ultrascan_probe_update` 函数中的 `newstate` 字段，并根据 `adjust_timing_hint` 参数调整接收时间戳的时间戳调整。

由于没有提供 `ultrascan_probe_update` 函数的定义，因此无法提供其具体的行为。


```cpp
const char *pspectype2ascii(int type);

void ultrascan_port_probe_update(UltraScanInfo *USI, HostScanStats *hss,
                                 std::list<UltraProbe *>::iterator probeI,
                                 int newstate, struct timeval *rcvdtime,
                                 bool adjust_timing_hint = true);

void ultrascan_host_probe_update(UltraScanInfo *USI, HostScanStats *hss,
                                        std::list<UltraProbe *>::iterator probeI,
                                        int newstate, struct timeval *rcvdtime,
                                        bool adjust_timing_hint = true);

void ultrascan_ping_update(UltraScanInfo *USI, HostScanStats *hss,
                                  std::list<UltraProbe *>::iterator probeI,
                                  struct timeval *rcvdtime,
                                  bool adjust_timing = true);
```

这是一个预处理指令，它检查当前源文件(SCAN_ENGINE_H.C)是否已经定义。如果文件已经被定义，则编译器会编译任何定义过并未在当前源文件中定义的符号。

简而言之，这个预处理指令告诉编译器不要在当前源文件中定义SCAN_ENGINE_H标准头文件，但如果它已经定义了，则编译器会忽略这些定义，继续编译。


```cpp
#endif /* SCAN_ENGINE_H */


```

# `scan_engine_connect.h`

This text is the official documentation for Nmap, a network scanner that provides information about the scan it has completed. It explains how to obtain a specific version of Nmap with a more permissive license for commercial use. The text also mentions that the official Nmap Windows builds are restricted from redistribution without special permission. Additionally, the text explains the license terms for Nmap and Npcap, and mentions that the Nmap Public Source License Contributor Agreement allows users to modify and redistribute the software, subject to certain conditions.



```cpp

/***************************************************************************
 * scan_engine_connect.h -- includes helper functions for scan_engine.cc   *
 * that are related to port scanning using connect() system call.          *
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

这段代码定义了一个名为 "scan_engine_connect.h" 的头文件，可能属于一个名为 "scan_engine" 的库或框架。

它包括以下类：

1. UltraProbe：可能是一个用于扫描引擎的设备或组件。
2. UltraScanInfo：包含用于扫描引擎的信息。
3. HostScanStats：用于记录扫描引擎的统计数据。
4. sendConnectScanProbe：用于发送连接扫描代理的函数。
5. do_one_select_round：用于在选择扫描轮时执行的函数。

该代码没有输出任何函数或变量，也没有定义任何变量，但使用了函数指针和宏定义。它可能是一个通用的库头部，或者是用于在编译时检查代码中符号定义的模板。


```cpp
/* $Id$ */

#ifndef SCAN_ENGINE_CONNECT_H
#define SCAN_ENGINE_CONNECT_H

#include <nbase.h>
class UltraProbe;
class UltraScanInfo;
class HostScanStats;

UltraProbe *sendConnectScanProbe(UltraScanInfo *USI, HostScanStats *hss,
                                 u16 destport, tryno_t tryno);
bool do_one_select_round(UltraScanInfo *USI, struct timeval *stime);

#endif

```

# `scan_engine_raw.h`

This text is the official documentation for Nmap, which is a network scanner used to discover networks,

it allows users to search for hosts, ports, and services, and to view the output in a

There are different versions of Nmap, including the standard version and an

OEM (Open Source License) version, which has additional features and a more permissive

Each version of Nmap has different licensing terms, and users are reminded to review

the specific terms of the version they are using.


```cpp

/***************************************************************************
 * scan_engine_raw.h -- includes helper functions for scan_engine.cc that *
 * are related to port scanning using raw (IP, Ethernet) packets.          *
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

这段代码定义了一个名为 "scan_engine_raw.h" 的头文件，它可能属于一个名为 "scan_engine" 的软件包。这个头文件中定义了一些类和函数，下面是一些概述：

1. "UltraProbe"，这是一个类，它可能是一个用于扫描网络的设备，可以捕获包并获取它们的源地址、目的地址和协议头。
2. "UltraScanInfo"，这是一个类，它包含有关扫描任务的信息，如扫描的目标、扫描类型和统计数据。
3. "HostScanStats"，这是一个类，它包含有关扫描结果的统计数据，如扫描的目标数量、扫描成功率等。
4. "Target"，这是一个类，它可能是一个用于被扫描的目标设备，如一个服务器或一个客户端。
5. "get_ping_pcap_result"，这是一个函数，它接收一个 UltraScanInfo 类型的对象和一个时间 val 类型的指针，用于获取与目标主机之间的履历。
6. "begin_sniffer"，这是一个函数，它接收一个 UltraScanInfo 类型的对象和一个向量类型的目标，用于开始捕获数据包并将其发送到目标。


```cpp
/* $Id$ */

#ifndef SCAN_ENGINE_RAW_H
#define SCAN_ENGINE_RAW_H

#include <nbase.h>
class UltraProbe;
class UltraScanInfo;
class HostScanStats;
#include <vector>

class Target;

int get_ping_pcap_result(UltraScanInfo *USI, struct timeval *stime);
void begin_sniffer(UltraScanInfo *USI, std::vector<Target *> &Targets);
```

这段代码定义了三个函数指针变量，分别是 sendArpScanProbe、sendNDScanProbe 和 sendIPScanProbe，它们都是用来发送不同类型的 scan Probe 的函数。

具体来说，sendArpScanProbe 函数接受两个参数 UltraScanInfo 和 HostScanStats，并返回一个指向 UltraProbe 的指针，用于发送 ARP 扫描 Probe。

sendNDScanProbe 函数同样接受两个参数 UltraScanInfo 和 HostScanStats，并返回一个指向 UltraProbe 的指针，用于发送 ND 扫描 Probe。

sendIPScanProbe 函数需要传递一个 UltraScanInfo、一个 const 的 probespec 类型的参数，以及一个指向 timeval 的指针。它用于发送 IP 扫描 Probe，并可以获取指定时间范围内所有匹配的 IP 地址。

另外，还定义了三个函数指针 get_arp_result、get_ns_result 和 get_pcap_result，它们分别用于获取 ARP、NS 和 Pcap 扫描结果。


```cpp
UltraProbe *sendArpScanProbe(UltraScanInfo *USI, HostScanStats *hss,
                             tryno_t tryno);
UltraProbe *sendNDScanProbe(UltraScanInfo *USI, HostScanStats *hss,
                            tryno_t tryno);
UltraProbe *sendIPScanProbe(UltraScanInfo *USI, HostScanStats *hss,
                            const probespec *pspec, tryno_t tryno);
bool get_arp_result(UltraScanInfo *USI, struct timeval *stime);
bool get_ns_result(UltraScanInfo *USI, struct timeval *stime);
bool get_pcap_result(UltraScanInfo *USI, struct timeval *stime);

#endif

```

# `scan_lists.h`

This text is a part of the Nmap license agreement. It explains the terms and conditions of the license, including the prohibition of using the software for commercial purposes and the restrictions on redistributing the software. It also explains the exceptions to these restrictions, including the Nmap OEM edition license and the use of the Npcap software for packet capture and transmission. The text also mentions the license's failure to provide any warranty for the free version of Nmap and that it is distributed "WITHOUT ANY WARRANTY" and that it is recommended to use the Npcap OEM program for commercial use.



```cpp
/***************************************************************************
 * scan_lists.h -- Structures and functions for lists of ports to scan and *
 * scan types                                                              *
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

这段代码定义了一个名为“scan_lists.h”的头文件，定义了一些标记，表示扫描哪些端口和协议。通过检查这些标记，可以决定哪些端口和协议应该被扫描。

具体来说，这段代码定义了以下标记：

SCAN_TCP_PORT：表示是否扫描TCP协议的端口。
SCAN_UDP_PORT：表示是否扫描UDP协议的端口。
SCAN_SCTP_PORT：表示是否扫描SCTP协议的端口。
SCAN_PROTOCOLS：表示是否扫描任何协议。

然后，定义了一个名为“scan_lists”的结构体，它包含以下字段：

syn_ping_ports：一个包含多个UDP端口的字段，用于在完成连接性测试时执行。
ack_ping_ports：一个包含多个UDP端口的字段，用于在确认连接性测试时执行。
udp_ping_ports：一个包含多个UDP端口的字段，用于执行UDP端口扫描。
sctp_ping_ports：一个包含多个SCTP端口的字段，用于执行SCTP端口扫描。
proto_ping_ports：一个包含多个协议的端口的字段，用于执行协议扫描。
syn_ping_count：一个整数，用于记录每个端口的syn_ping操作次数。
ack_ping_count：一个整数，用于记录每个端口的ack_ping操作次数。
udp_ping_count：一个整数，用于记录每个端口的udp_ping操作次数。
sctp_ping_count：一个整数，用于记录每个端口的sctp_ping操作次数。
proto_ping_count：一个整数，用于记录每个端口的protocol_scan操作次数。
tcp_ports：一个包含多个TCP端口的字段，用于执行TCP端口扫描。
tcp_count：一个整数，用于记录每个TCP端口的操作次数。
udp_ports：一个包含多个UDP端口的字段，用于执行UDP端口扫描。
udp_count：一个整数，用于记录每个UDP端口的操作次数。
sctp_ports：一个包含多个SCTP端口的字段，用于执行SCTP端口扫描。
sctp_count：一个整数，用于记录每个SCTP端口的操作次数。
protocols：一个包含多个协议名称的字段，用于执行协议扫描。
protoc


```cpp
#ifndef SCAN_LISTS_H
#define SCAN_LISTS_H

/* just flags to indicate whether a particular port number should get tcp
 * scanned, udp scanned, or both
 */
#define SCAN_TCP_PORT	(1 << 0)
#define SCAN_UDP_PORT	(1 << 1)
#define SCAN_SCTP_PORT	(1 << 2)
#define SCAN_PROTOCOLS	(1 << 3)

/* The various kinds of port/protocol scans we can have
 * Each element is to point to an array of port/protocol numbers
 */
struct scan_lists {
        /* The "synprobes" are also used when doing a connect() ping */
        unsigned short *syn_ping_ports;
        unsigned short *ack_ping_ports;
        unsigned short *udp_ping_ports;
        unsigned short *sctp_ping_ports;
        unsigned short *proto_ping_ports;
        int syn_ping_count;
        int ack_ping_count;
        int udp_ping_count;
        int sctp_ping_count;
        int proto_ping_count;
        //the above fields are only used for host discovery
        //the fields below are only used for port scanning
        unsigned short *tcp_ports;
        int tcp_count;
        unsigned short *udp_ports;
        int udp_count;
        unsigned short *sctp_ports;
        int sctp_count;
        unsigned short *prots;
        int prot_count;
};

```

以上代码定义了一个枚举类型 `stype`，该枚举类型定义了 13 个不同的枚举值，分别为 STYPE_UNKNOWN、HOST_DISCOVERY、ACK_SCAN、SYN_SCAN、FIN_SCAN、XMAS_SCAN、UDP_SCAN、CONNECT_SCAN、NULL_SCAN、WINDOW_SCAN、SCTP_INIT_SCAN、SCTP_COOKIE_ECHO_SCAN、MAIMON_SCAN、IPPROT_SCAN、PING_SCAN 和 PING_SCAN_ARP。每个枚举值都有一个默认的枚举值，分别为 STYPE_UNKNOWN_DEFAULT、HOST_DISCOVERY_DEFAULT、ACK_SCAN_DEFAULT、SYN_SCAN_DEFAULT、FIN_SCAN_DEFAULT、XMAS_SCAN_DEFAULT、UDP_SCAN_DEFAULT、CONNECT_SCAN_DEFAULT、NULL_SCAN_DEFAULT、WINDOW_SCAN_DEFAULT、SCTP_INIT_SCAN_DEFAULT、SCTP_COOKIE_ECHO_SCAN_DEFAULT 和 IPPROT_SCAN_DEFAULT。

这个枚举类型定义了 SType 枚举类型，用于在代码中使用枚举常量，以便根据特定的值执行不同的操作。例如，使用 SType.HOST_DISCOVERY 枚举值可以实现主机探索功能，SType.ACK_SCAN 枚举值可以用于确认数据包应答，等等。


```cpp
typedef enum {
  STYPE_UNKNOWN,
  HOST_DISCOVERY,
  ACK_SCAN,
  SYN_SCAN,
  FIN_SCAN,
  XMAS_SCAN,
  UDP_SCAN,
  CONNECT_SCAN,
  NULL_SCAN,
  WINDOW_SCAN,
  SCTP_INIT_SCAN,
  SCTP_COOKIE_ECHO_SCAN,
  MAIMON_SCAN,
  IPPROT_SCAN,
  PING_SCAN,
  PING_SCAN_ARP,
  IDLE_SCAN,
  BOUNCE_SCAN,
  SERVICE_SCAN,
  OS_SCAN,
  SCRIPT_PRE_SCAN,
  SCRIPT_SCAN,
  SCRIPT_POST_SCAN,
  TRACEROUTE,
  PING_SCAN_ND
} stype;

```



这是一段用C语言编写的函数，定义了四个函数，用于获取、修改和删除port扫描列表。

1. `getpts`函数，接收一个const char *类型的expr参数，返回一个struct scan_lists类型的指针。该函数的作用是获取表达式符号左侧的port扫描列表。

2. `getpts_simple`函数，接收一个const char *类型的origexpr参数，以及一个int类型的range_type参数，返回一个int类型的指针。该函数的作用是获取origexpr表达式中port扫描列表的起始位置和结束位置，并将结果存储在list中。

3. `removepts`函数，接收一个const char *类型的expr参数，返回一个struct scan_lists类型的指针。该函数的作用是删除const char *expr参数中所有出现在port扫描列表中的port，并将结果存储在ports参数中。

4. `free_scan_lists`函数，接收一个struct scan_lists类型的指针，用于存储free_scan_lists函数的参数。该函数的作用是free函数的参数ports所指向的内存空间，释放掉已经分配的内存。

该代码可能是在一个port扫描程序中使用的。通过使用这些函数，可以方便地管理port扫描列表，实现不同的port扫描功能。


```cpp
/* port manipulators */
void getpts(const char *expr, struct scan_lists * ports); /* someone stole the name getports()! */
void getpts_simple(const char *origexpr, int range_type,
                   unsigned short **list, int *count);
void removepts(const char *expr, struct scan_lists * ports);
void free_scan_lists(struct scan_lists *ports);

/* general helper functions */
const char *scantype2str(stype scantype);

#endif /* SCAN_LISTS_H */

```