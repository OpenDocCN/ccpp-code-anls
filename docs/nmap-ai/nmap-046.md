# Nmap源码解析 46

# `libpcap/ieee80211.h`

This is a description of the Internet Engineering Task Force (IETF) standard RFC 80211. This standard defines the Redistributions of藏有可执行代码的软件（RDS）许可，其中包括源代码和二进制形式的分布式许可。

根据RFC 80211，分布式许可必须满足以下条件：

1. 分布式许可方必须包含一份《软件许可协议》或其他官方认可的合同，其中应包含上述条件以及声明和保证。
2. 在二进制形式中，许可方必须包含上述条件以及声明和保证。
3. 允许在遵循特定许可协议的情况下，对使用或修改软件的方式进行限制。

分布式许可允许在特定情况下对源代码和二进制形式的分布式许可。


```cpp
/*-
 * Copyright (c) 2001 Atsushi Onoe
 * Copyright (c) 2002-2005 Sam Leffler, Errno Consulting
 * All rights reserved.
 *
 * Redistribution and use in source and binary forms, with or without
 * modification, are permitted provided that the following conditions
 * are met:
 * 1. Redistributions of source code must retain the above copyright
 *    notice, this list of conditions and the following disclaimer.
 * 2. Redistributions in binary form must reproduce the above copyright
 *    notice, this list of conditions and the following disclaimer in the
 *    documentation and/or other materials provided with the distribution.
 * 3. The name of the author may not be used to endorse or promote products
 *    derived from this software without specific prior written permission.
 *
 * Alternatively, this software may be distributed under the terms of the
 * GNU General Public License ("GPL") version 2 as published by the Free
 * Software Foundation.
 *
 * THIS SOFTWARE IS PROVIDED BY THE AUTHOR ``AS IS'' AND ANY EXPRESS OR
 * IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE IMPLIED WARRANTIES
 * OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE ARE DISCLAIMED.
 * IN NO EVENT SHALL THE AUTHOR BE LIABLE FOR ANY DIRECT, INDIRECT,
 * INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT
 * NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS OF USE,
 * DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY
 * THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT
 * (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE OF
 * THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.
 *
 * $FreeBSD: src/sys/net80211/ieee80211.h,v 1.10 2005/07/22 16:55:27 sam Exp $
 */
```

这段代码定义了一个头文件，名为_NET80211_IEEE80211_H_，接下来定义了一系列802.11协议定义。

具体来说，定义了IEEE80211_FC0_VERSION_MASK、IEEE80211_FC0_VERSION_SHIFT、IEEE80211_FC0_VERSION_0、IEEE80211_FC0_TYPE_MASK、IEEE80211_FC0_TYPE_SHIFT、IEEE80211_FC0_TYPE_MGT和IEEE80211_FC0_TYPE_CTL。其中，IEEE80211_FC0_VERSION_MASK表示802.11协议的最低版本，IEEE80211_FC0_TYPE_MASK表示802.11协议的类型字段，可以用来标识数据帧的类型。


```cpp
#ifndef _NET80211_IEEE80211_H_
#define _NET80211_IEEE80211_H_

/*
 * 802.11 protocol definitions.
 */

#define	IEEE80211_FC0_VERSION_MASK		0x03
#define	IEEE80211_FC0_VERSION_SHIFT		0
#define	IEEE80211_FC0_VERSION_0			0x00
#define	IEEE80211_FC0_TYPE_MASK			0x0c
#define	IEEE80211_FC0_TYPE_SHIFT		2
#define	IEEE80211_FC0_TYPE_MGT			0x00
#define	IEEE80211_FC0_TYPE_CTL			0x04
#define	IEEE80211_FC0_TYPE_DATA			0x08

```

这段代码定义了一系列宏定义，用于标识IEEE80211无线网络中的数据帧的帧类型（subtype）。这些宏定义通过IEEE80211_FC0_SUBTYPE_MASK和IEEE80211_FC0_SUBTYPE_SHIFT来设置。其中，IEEE80211_FC0_SUBTYPE_MASK是一个32位的二进制数，用于标识数据帧帧类型的位数。IEEE80211_FC0_SUBTYPE_SHIFT是一个4位的二进制数，用于将MASK中的位数向左移动4位，以便使用IEEE80211_FC0_SUBTYPE_MASK进行计算。

根据IEEE80211_FC0_SUBTYPE_MASK，数据帧的帧类型可以分为以下几种：

- IEEE80211_FC0_SUBTYPE_PROBE_REQ：用于发送代理请求（association request）的数据帧。
- IEEE80211_FC0_SUBTYPE_PROBE_RESP：用于发送代理响应（association response）的数据帧。
- IEEE80211_FC0_SUBTYPE_BEACON：用于发送Beacon（广播）数据帧。
- IEEE80211_FC0_SUBTYPE_ASSOC_REQ：用于标识成员设备（associated device）的请求。
- IEEE80211_FC0_SUBTYPE_ASSOC_RESP：用于标识成员设备（associated device）的响应。
- IEEE80211_FC0_SUBTYPE_REASSOC_REQ：用于标识重新加入组（reassociation request）的数据帧。
- IEEE80211_FC0_SUBTYPE_REASSOC_RESP：用于标识重新加入组（reassociation response）的数据帧。
- IEEE80211_FC0_SUBTYPE_PROBE_REQ：用于发送自适应波束赋值（adaptive beacon）的数据帧。
- IEEE80211_FC0_SUBTYPE_PROBE_RESP：用于发送自适应波束赋值（adaptive beacon）的响应。

这些宏定义根据数据帧的帧类型进行匹配，然后通过IEEE80211_FC0_SUBTYPE_MASK进行计算，以确定所需调用的函数。这些函数可以在芯片的驱动程序中找到，用于执行相应的操作。


```cpp
#define	IEEE80211_FC0_SUBTYPE_MASK		0xf0
#define	IEEE80211_FC0_SUBTYPE_SHIFT		4
/* for TYPE_MGT */
#define	IEEE80211_FC0_SUBTYPE_ASSOC_REQ		0x00
#define	IEEE80211_FC0_SUBTYPE_ASSOC_RESP	0x10
#define	IEEE80211_FC0_SUBTYPE_REASSOC_REQ	0x20
#define	IEEE80211_FC0_SUBTYPE_REASSOC_RESP	0x30
#define	IEEE80211_FC0_SUBTYPE_PROBE_REQ		0x40
#define	IEEE80211_FC0_SUBTYPE_PROBE_RESP	0x50
#define	IEEE80211_FC0_SUBTYPE_BEACON		0x80
#define	IEEE80211_FC0_SUBTYPE_ATIM		0x90
#define	IEEE80211_FC0_SUBTYPE_DISASSOC		0xa0
#define	IEEE80211_FC0_SUBTYPE_AUTH		0xb0
#define	IEEE80211_FC0_SUBTYPE_DEAUTH		0xc0
/* for TYPE_CTL */
```

这段代码定义了一系列宏定义，它们用于定义IEEE80211无线网络中的数据类型。这些宏定义用于在头文件中使用，而不是在源代码中使用。

具体来说，这些宏定义定义了以下数据类型：

- IEEE80211_FC0_SUBTYPE_PS_POLL： poll请求
- IEEE80211_FC0_SUBTYPE_RTS： 随机接入请求
- IEEE80211_FC0_SUBTYPE_CTS： 请求传输
- IEEE80211_FC0_SUBTYPE_ACK： 确认
- IEEE80211_FC0_SUBTYPE_CF_END： 数据帧结束
- IEEE80211_FC0_SUBTYPE_CF_END_ACK： 数据帧结束并确认
- IEEE80211_FC0_SUBTYPE_DATA： 数据
- IEEE80211_FC0_SUBTYPE_CF_ACK： 确认请求
- IEEE80211_FC0_SUBTYPE_CF_POLL： 轮询
- IEEE80211_FC0_SUBTYPE_CF_ACPL： 活动证书计数
- IEEE80211_FC0_SUBTYPE_NODATA： 无数据
- IEEE80211_FC0_SUBTYPE_NODATA_CF_ACK： 无数据并确认
- IEEE80211_FC0_SUBTYPE_NODATA_CF_POLL： 无数据并轮询
- IEEE80211_FC0_SUBTYPE_NODATA_CF_ACPL： 无数据并活动证书计数


```cpp
#define	IEEE80211_FC0_SUBTYPE_PS_POLL		0xa0
#define	IEEE80211_FC0_SUBTYPE_RTS		0xb0
#define	IEEE80211_FC0_SUBTYPE_CTS		0xc0
#define	IEEE80211_FC0_SUBTYPE_ACK		0xd0
#define	IEEE80211_FC0_SUBTYPE_CF_END		0xe0
#define	IEEE80211_FC0_SUBTYPE_CF_END_ACK	0xf0
/* for TYPE_DATA (bit combination) */
#define	IEEE80211_FC0_SUBTYPE_DATA		0x00
#define	IEEE80211_FC0_SUBTYPE_CF_ACK		0x10
#define	IEEE80211_FC0_SUBTYPE_CF_POLL		0x20
#define	IEEE80211_FC0_SUBTYPE_CF_ACPL		0x30
#define	IEEE80211_FC0_SUBTYPE_NODATA		0x40
#define	IEEE80211_FC0_SUBTYPE_NODATA_CF_ACK	0x50
#define	IEEE80211_FC0_SUBTYPE_NODATA_CF_POLL	0x60
#define	IEEE80211_FC0_SUBTYPE_NODATA_CF_ACPL	0x70
```

这段代码定义了一系列宏，用于定义IEEE 802.11协议中的QoS(服务质量)类型。

具体来说，这段代码定义了以下几个常量：

- IEEE80211_FC0_SUBTYPE_QOS:QoS类型的下标，值为0x80，表示支持服务质量级别(QoS level)0x80。
- IEEE80211_FC0_SUBTYPE_QOS_NULL:QoS类型的下标，值为0xc0，表示不支持任何QoS的QoS类型。
- IEEE80211_FC1_DIR_MASK:QoS类型驱动器方向位掩码(Dir mask)，用于指示数据帧中数据帧方向位(SSID)和接收者(Rx)如何设置。
- IEEE80211_FC1_DIR_NODS:QoS类型驱动器方向位(Dir bit)，用于指示数据帧中是否发送设备方向位(TSID)。
- IEEE80211_FC1_DIR_TODS:QoS类型驱动器方向位(Dir bit)，用于指示接收者是否收到数据帧方向位(RXTSID)。
- IEEE80211_FC1_DIR_FROMDS:QoS类型源地址，用于指示数据帧中发送者设备的方向位(TSID)。
- IEEE80211_FC1_DIR_DSTODS:QoS类型目标地址，用于指示数据帧中接收者设备的方向位(RXTSID)。
- IEEE80211_FC1_MORE_FRAG:QoS类型标志，表示是否是 fragment(分片)数据帧。
- IEEE80211_FC1_RETRY:QoS类型标志，表示是否尝试重新发送丢失的数据帧。
- IEEE80211_FC1_PWR_MGT:QoS类型标志，表示是否使用功率管理(Power Management)机制。
- IEEE80211_FC1_MORE_DATA:QoS类型标志，表示是否包含更多数据(如扩展铭文或扩展信息)。
- IEEE80211_FC1_WEP:QoS类型标志，表示是否支持WEP(无线扩展认证)机制。
- IEEE80211_FC1_ORDER:QoS类型标志，表示是否支持ORDER(ORDERED)机制。


```cpp
#define	IEEE80211_FC0_SUBTYPE_QOS		0x80
#define	IEEE80211_FC0_SUBTYPE_QOS_NULL		0xc0

#define	IEEE80211_FC1_DIR_MASK			0x03
#define	IEEE80211_FC1_DIR_NODS			0x00	/* STA->STA */
#define	IEEE80211_FC1_DIR_TODS			0x01	/* STA->AP  */
#define	IEEE80211_FC1_DIR_FROMDS		0x02	/* AP ->STA */
#define	IEEE80211_FC1_DIR_DSTODS		0x03	/* AP ->AP  */

#define	IEEE80211_FC1_MORE_FRAG			0x04
#define	IEEE80211_FC1_RETRY			0x08
#define	IEEE80211_FC1_PWR_MGT			0x10
#define	IEEE80211_FC1_MORE_DATA			0x20
#define	IEEE80211_FC1_WEP			0x40
#define	IEEE80211_FC1_ORDER			0x80

```

这段代码定义了一系列IEEE80211无线标准中的定义，包括SEQ_FRAG_MASK,SEQ_SEQ_MASK,SEQ_SEQ_SHIFT,IEEE80211_NWID_LEN,QOS_TXOP,QOS_ACKPOLICY,QOS_ACKPOLICY_S,QOS_ESOP,QOS_ESOP_S,QOS_TID等。其中，IEEE80211_SEQ_FRAG_MASK表示SEQ_FRAG分片的长度，SEQ_SEQ_MASK是SEQ_SEQ序列的长度，SEQ_SEQ_SHIFT是序列的偏移量，IEEE80211_NWID_LEN是无线网络ID的长度，QOS_TXOP是服务质量保证(QoS)的传输偏移量，QOS_ACKPOLICY和QOS_ACKPOLICY_S是服务质量保证的发送和接收策略，IEEE80211_QOS_ESOP和IEEE80211_QOS_ESOP_S是扩展序列设置(ESP)中的扩展序列类型，IEEE80211_QOS_TID是事务ID。这些定义用于定义IEEE80211无线网络的标准，以实现更好的无线网络性能和可靠性。


```cpp
#define	IEEE80211_SEQ_FRAG_MASK			0x000f
#define	IEEE80211_SEQ_FRAG_SHIFT		0
#define	IEEE80211_SEQ_SEQ_MASK			0xfff0
#define	IEEE80211_SEQ_SEQ_SHIFT			4

#define	IEEE80211_NWID_LEN			32

#define	IEEE80211_QOS_TXOP			0x00ff
/* bit 8 is reserved */
#define	IEEE80211_QOS_ACKPOLICY			0x60
#define	IEEE80211_QOS_ACKPOLICY_S		5
#define	IEEE80211_QOS_ESOP			0x10
#define	IEEE80211_QOS_ESOP_S			4
#define	IEEE80211_QOS_TID			0x0f

```

这两行代码定义了IEEE80211无线网络的MGT（Message Type Body）和CTL（Message Type Header）subtypes。

MGT subtypes包含了IEEE80211无线网络中的数据帧类型，包括AssocRequest、AssocResponse、ProbeRequest、ProbeResponse、Reserved#6到Reserved#15。

CTL subtypes包含了IEEE80211无线网络中的控制帧类型，包括Reserved#0到Reserved#15。

通过定义这些subtypes，可以使得代码在编译时更易于理解和调试。在代码中，这些名称已经包含了它们的含义，所以不需要在代码中再次解释它们的含义。


```cpp
#define IEEE80211_MGT_SUBTYPE_NAMES {			\
	"assoc-req",		"assoc-resp",		\
	"reassoc-req",		"reassoc-resp",		\
	"probe-req",		"probe-resp",		\
	"reserved#6",		"reserved#7",		\
	"beacon",		"atim",			\
	"disassoc",		"auth",			\
	"deauth",		"reserved#13",		\
	"reserved#14",		"reserved#15"		\
}

#define IEEE80211_CTL_SUBTYPE_NAMES {			\
	"reserved#0",		"reserved#1",		\
	"reserved#2",		"reserved#3",		\
	"reserved#3",		"reserved#5",		\
	"reserved#6",		"reserved#7",		\
	"reserved#8",		"reserved#9",		\
	"ps-poll",		"rts",			\
	"cts",			"ack",			\
	"cf-end",		"cf-end-ack"		\
}

```

这段代码定义了一个名为 IEEE80211_DATA_SUBTYPE_NAMES 的宏，它定义了数据子类型的名称。这个宏是预处理指令，它在编译时就会被替换为它定义的值。

接下来的部分定义了一个名为 IEEE80211_TYPE_NAMES 的枚举，它定义了 IEEE80211 规范中的实体名称。这个枚举同样也是预处理指令，它在编译时就会被替换为它定义的值。

最后，没有其他代码，所以这个头文件可以被包含在源代码中，也可以被独立地使用。


```cpp
#define IEEE80211_DATA_SUBTYPE_NAMES {			\
	"data",			"data-cf-ack",		\
	"data-cf-poll",		"data-cf-ack-poll",	\
	"null",			"cf-ack",		\
	"cf-poll",		"cf-ack-poll",		\
	"qos-data",		"qos-data-cf-ack",	\
	"qos-data-cf-poll",	"qos-data-cf-ack-poll",	\
	"qos",			"reserved#13",		\
	"qos-cf-poll",		"qos-cf-ack-poll"	\
}

#define IEEE80211_TYPE_NAMES	{ "mgt", "ctl", "data", "reserved#4" }

#endif /* _NET80211_IEEE80211_H_ */

```

# libpcap installation notes
Libpcap can be built either with the configure script and `make`, or
with CMake and any build system supported by CMake.

To build libpcap with the configure script and `make`:

* Run `./configure` (a shell script).  The configure script will
determine your system attributes and generate an appropriate `Makefile`
from `Makefile.in`.  The configure script has a number of options to
control the configuration of libpcap; `./configure --help`` will show
them.

* Next, run `make`.  If everything goes well, you can
`su` to root and run `make install`.  However, you need not install
libpcap if you just want to build tcpdump; just make sure the tcpdump
and libpcap directory trees have the same parent directory.

To build libpcap with CMake and the build system of your choice, from
the command line:

* Create a build directory into which CMake will put the build files it
generates; CMake does not work as well with builds done in the source
code directory as does the configure script.  The build directory may be
created as a subdirectory of the source directory or as a directory
outside the source directory.

* Change to the build directory and run CMake with the path from the
build directory to the source directory as an argument.  The `-G` flag
can be used to select the CMake "generator" appropriate for the build
system you're using; various `-D` flags can be used to control the
configuration of libpcap.

* Run the build tool.  If everything goes well, you can `su` to root and
run the build tool with the `install` target.  Building tcpdump from a
libpcap in a build directory is not supported.

An `uninstall` target is supported with both `./configure` and CMake.

***DO NOT*** run the build as root; there is no need to do so, running
anything as root that doesn't need to be run as root increases the risk
of damaging your system, and running the build as root will put files in
the build directory that are owned by root and that probably cannot be
overwritten, removed, or replaced except by root, which could cause
permission errors in subsequent builds.

If configure says:

    configure: warning: cannot determine packet capture interface
    configure: warning: (see INSTALL.md file for more info)

or CMake says:

    cannot determine packet capture interface

    (see the INSTALL.md file for more info)

then your system either does not support packet capture or your system
does support packet capture but libpcap does not support that
particular type. (If you have HP-UX, see below.) If your system uses a
packet capture not supported by libpcap, please send us patches; don't
forget to include an autoconf fragment suitable for use in
`configure.ac`.

It is possible to override the default packet capture type with the
`--with-pcap`` option to `./configure` or the `-DPCAP_TYPE` option to
CMake, although the circumstances where this works are limited.  One
possible reason to do that would be to force a supported packet capture
type in the case where the configure or CMake scripts fails to detect
it.

You will need a C99 compiler to build libpcap. The configure script
will abort if your compiler is not C99 compliant. If this happens, use
the generally available GNU C compiler (GCC) or Clang.

You will need either Flex 2.5.31 or later, or a version of Lex
compatible with it (if any exist), to build libpcap.  The configure
script will abort if there isn't any such program; CMake fails if Flex
or Lex cannot be found, but doesn't ensure that it's compatible with
Flex 2.5.31 or later.  If you have an older version of Flex, or don't
have a compatible version of Lex, the current version of Flex is
available [here](https://github.com/westes/flex).

You will need either Bison, Berkeley YACC, or a version of YACC
compatible with them (if any exist), to build libpcap.  The configure
script will abort if there isn't any such program; CMake fails if Bison
or some form of YACC cannot be found, but doesn't ensure that it's
compatible with Bison or Berkeley YACC.  If you don't have any such
program, the current version of Bison can be found
[here](https://ftp.gnu.org/gnu/bison/) and the current version of
Berkeley YACC can be found [here](https://invisible-island.net/byacc/).

Sometimes the stock C compiler does not interact well with Flex and
Bison. The list of problems includes undefined references for alloca(3).
You can get around this by installing GCC.

## Linux specifics
On Linux, libpcap will not work if the kernel does not have the packet
socket option enabled; see [this file](doc/README.linux) for more
information.

## Solaris specifics
If you use the SPARCompiler, you must be careful to not use the
`/usr/ucb/cc` interface. If you do, you will get bogus warnings and
perhaps errors. Either make sure your path has `/opt/SUNWspro/bin`
before `/usr/ucb` or else:

    setenv CC /opt/SUNWspro/bin/cc

before running configure. (You might have to do a `make distclean`
if you already ran `configure` once).

See [this file](doc/README.solaris.md) for more up to date
Solaris-related information.

## HP-UX specifics
If you use HP-UX, you must have at least version 9 and either the
version of `cc` that supports C99 (`cc -AC99`) or else use the GNU C
compiler. You must also buy the optional streams package. If you don't
have:

    /usr/include/sys/dlpi.h
    /usr/include/sys/dlpi_ext.h

then you don't have the streams package. In addition, we believe you
need to install the "9.X LAN and DLPI drivers cumulative" patch
(PHNE_6855) to make the version 9 DLPI work with libpcap.

The DLPI streams package is standard starting with HP-UX 10.

The HP implementation of DLPI is a little bit eccentric. Unlike
Solaris, you must attach `/dev/dlpi` instead of the specific `/dev/*`
network pseudo device entry in order to capture packets. The PPA is
based on the ifnet "index" number. Under HP-UX 9, it is necessary to
read `/dev/kmem` and the kernel symbol file (`/hp-ux`). Under HP-UX 10,
DLPI can provide information for determining the PPA. It does not seem
to be possible to trace the loopback interface. Unlike other DLPI
implementations, PHYS implies MULTI and SAP and you get an error if you
try to enable more than one promiscuous mode at a time.

It is impossible to capture outbound packets on HP-UX 9.  To do so on
HP-UX 10, you will, apparently, need a late "LAN products cumulative
patch" (at one point, it was claimed that this would be PHNE_18173 for
s700/10.20; at another point, it was claimed that the required patches
were PHNE_20892, PHNE_20725 and PHCO_10947, or newer patches), and to do
so on HP-UX 11 you will, apparently, need the latest lancommon/DLPI
patches and the latest driver patch for the interface(s) in use on HP-UX
11 (at one point, it was claimed that patches PHNE_19766, PHNE_19826,
PHNE_20008, and PHNE_20735 did the trick).

Furthermore, on HP-UX 10, you will need to turn on a kernel switch by
doing

	echo 'lanc_outbound_promisc_flag/W 1' | adb -w /stand/vmunix /dev/mem

You would have to arrange that this happens on reboots; the right way to
do that would probably be to put it into an executable script file
`/sbin/init.d/outbound_promisc` and making
`/sbin/rc2.d/S350outbound_promisc` a symbolic link to that script.

Finally, testing shows that there can't be more than one simultaneous
DLPI user per network interface.

See [this file](doc/README.hpux) for more information specific to HP-UX.

## AIX specifics
See [this file](doc/README.aix) for information on installing libpcap and
configuring your system to be able to support libpcap.

## other specifics
If you are trying to do packet capture with a FORE ATM card, you may or
may not be able to. They usually only release their driver in object
code so unless their driver supports packet capture, there's not much
libpcap can do.

If you get an error like:

    tcpdump: recv_ack: bind error 0x???

when using DLPI, look for the DL_ERROR_ACK error return values, usually
in `/usr/include/sys/dlpi.h`, and find the corresponding value.

## Description of files
	CHANGES		    - description of differences between releases
	ChmodBPF/*	    - macOS startup item to set ownership and permissions on /dev/bpf*
	CMakeLists.txt	    - CMake file
	CONTRIBUTING.md	    - guidelines for contributing
	CREDITS		    - people that have helped libpcap along
	INSTALL.md	    - this file
	LICENSE		    - the license under which tcpdump is distributed
	Makefile.in	    - compilation rules (input to the configure script)
	README.md	    - description of distribution
	doc/README.aix	    - notes on using libpcap on AIX
	doc/README.dag	    - notes on using libpcap to capture on Endace DAG devices
	doc/README.hpux	    - notes on using libpcap on HP-UX
	doc/README.linux    - notes on using libpcap on Linux
	doc/README.macos    - notes on using libpcap on macOS
	doc/README.septel   - notes on using libpcap to capture on Intel/Septel devices
	doc/README.sita	    - notes on using libpcap to capture on SITA devices
	doc/README.solaris.md - notes on using libpcap on Solaris
	doc/README.Win32.md - notes on using libpcap on Win32 systems (with Npcap)
	VERSION		    - version of this release
	aclocal.m4	    - autoconf macros
	arcnet.h	    - ARCNET definitions
	atmuni31.h	    - ATM Q.2931 definitions
	bpf_dump.c	    - BPF program printing routines
	bpf_filter.c	    - BPF filtering routines
	bpf_image.c	    - BPF disassembly routine
	config.guess	    - autoconf support
	config.h.in	    - autoconf input
	config.sub	    - autoconf support
	configure	    - configure script (run this first)
	configure.ac	    - configure script source
	dlpisubs.c	    - DLPI-related functions for pcap-dlpi.c and pcap-libdlpi.c
	dlpisubs.h	    - DLPI-related function declarations
	etherent.c	    - /etc/ethers support routines
	ethertype.h	    - Ethernet protocol types and names definitions
	fad-getad.c	    - pcap_findalldevs() for systems with getifaddrs()
	fad-gifc.c	    - pcap_findalldevs() for systems with only SIOCGIFLIST
	fad-glifc.c	    - pcap_findalldevs() for systems with SIOCGLIFCONF
	testprogs/filtertest.c      - test program for BPF compiler
	testprogs/findalldevstest.c - test program for pcap_findalldevs()
	gencode.c	    - BPF code generation routines
	gencode.h	    - BPF code generation definitions
	grammar.y	    - filter string grammar
	ieee80211.h	    - 802.11 definitions
	install-sh	    - BSD style install script
	lbl/os-*.h	    - OS-dependent defines and prototypes
	llc.h		    - 802.2 LLC SAP definitions
	missing/*	    - replacements for missing library functions
	mkdep		    - construct Makefile dependency list
	msdos/*		    - drivers for MS-DOS capture support
	nametoaddr.c	    - hostname to address routines
	nlpid.h		    - OSI network layer protocol identifier definitions
	optimize.c	    - BPF optimization routines
	pcap/bluetooth.h    - public definition of DLT_BLUETOOTH_HCI_H4_WITH_PHDR header
	pcap/bpf.h	    - BPF definitions
	pcap/namedb.h	    - public libpcap name database definitions
	pcap/pcap.h	    - public libpcap definitions
	pcap/sll.h	    - public definitions of DLT_LINUX_SLL and DLT_LINUX_SLL2 headers
	pcap/usb.h	    - public definition of DLT_USB header
	pcap-bpf.c	    - BSD Packet Filter support
	pcap-bpf.h	    - header for backwards compatibility
	pcap-bt-linux.c	    - Bluetooth capture support for Linux
	pcap-bt-linux.h	    - Bluetooth capture support for Linux
	pcap-dag.c	    - Endace DAG device capture support
	pcap-dag.h	    - Endace DAG device capture support
	pcap-dlpi.c	    - Data Link Provider Interface support
	pcap-dos.c	    - MS-DOS capture support
	pcap-dos.h	    - headers for MS-DOS capture support
	pcap-enet.c	    - enet support
	pcap-int.h	    - internal libpcap definitions
	pcap-libdlpi.c	    - Data Link Provider Interface support for systems with libdlpi
	pcap-linux.c	    - Linux packet socket support
	pcap-namedb.h	    - header for backwards compatibility
	pcap-nit.c	    - SunOS Network Interface Tap support
	pcap-npf.c	    - Npcap capture support
	pcap-null.c	    - dummy monitor support (allows offline use of libpcap)
	pcap-pf.c	    - Ultrix and Digital/Tru64 UNIX Packet Filter support
	pcap-septel.c       - Intel/Septel device capture support
	pcap-septel.h       - Intel/Septel device capture support
	pcap-sita.c	    - SITA device capture support
	pcap-sita.h	    - SITA device capture support
	pcap-sita.html	    - SITA device capture documentation
	pcap-snit.c	    - SunOS 4.x STREAMS-based Network Interface Tap support
	pcap-snoop.c	    - IRIX Snoop network monitoring support
	pcap-usb-linux.c    - USB capture support for Linux
	pcap-usb-linux.h    - USB capture support for Linux
	pcap.3pcap	    - manual entry for the library
	pcap.c		    - pcap utility routines
	pcap.h		    - header for backwards compatibility
	pcap_*.3pcap	    - manual entries for library functions
	pcap-filter.manmisc.in   - manual entry for filter syntax
	pcap-linktype.manmisc.in - manual entry for link-layer header types
	ppp.h		    - Point to Point Protocol definitions
	savefile.c	    - offline support
	scanner.l	    - filter string scanner
	sunatmpos.h	    - definitions for SunATM capturing


# `libpcap/llc.h`

这段代码是一个C语言的函数声明，它定义了一些函数，但不会输出具体的函数名称。这些函数可能涉及到文件操作、用户界面、网络编程或其他与操作系统交互的领域。同时，这段代码使用了非常严格的版权声明，以保护相关知识产权。


```cpp
/*
 * Copyright (c) 1993, 1994, 1997
 *	The Regents of the University of California.  All rights reserved.
 *
 * Redistribution and use in source and binary forms, with or without
 * modification, are permitted provided that: (1) source code distributions
 * retain the above copyright notice and this paragraph in its entirety, (2)
 * distributions including binary code include the above copyright notice and
 * this paragraph in its entirety in the documentation or other materials
 * provided with the distribution, and (3) all advertising materials mentioning
 * features or use of this software display the following acknowledgement:
 * ``This product includes software developed by the University of California,
 * Lawrence Berkeley Laboratory and its contributors.'' Neither the name of
 * the University nor the names of its contributors may be used to endorse
 * or promote products derived from this software without specific prior
 * written permission.
 * THIS SOFTWARE IS PROVIDED ``AS IS'' AND WITHOUT ANY EXPRESS OR IMPLIED
 * WARRANTIES, INCLUDING, WITHOUT LIMITATION, THE IMPLIED WARRANTIES OF
 * MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE.
 */

```

这段代码定义了一些与LLC（LLab UML图形）相关的定义。

LLC_U_FMT表示LLC头部中关于单元格的信息。
LLC_GSAP表示LLC头部中关于图形和符号的定义。
LLC_IG表示LLC头部中关于单个或组单位的定义。
LLC_S_FMT表示LLC头部中关于字符串的定义。

LLC_U_POLL表示LLC头部中关于单元格下标操作的定义。
LLC_IS_POLL表示LLC头部中关于POLL操作的定义。
LLC_XID_FI表示LLC头部中关于XID输入输出的定义。

LLC_U_CMD_MASK表示LLC头部中关于命令和命令修饰的定义。
LLC_UI表示LLC头部中关于UI属性的定义。

LLC_CLR_UI表示LLC头部中关于CLR UI属性的定义。


```cpp
/*
 * Definitions for information in the LLC header.
 */

#define	LLC_U_FMT	3
#define	LLC_GSAP	1
#define	LLC_IG	        1 /* Individual / Group */
#define LLC_S_FMT	1

#define	LLC_U_POLL	0x10
#define	LLC_IS_POLL	0x0100
#define	LLC_XID_FI	0x81

#define LLC_U_CMD_MASK	0xef
#define	LLC_UI		0x03
```

这段代码定义了一系列宏定义，它们用于定义LLC命令集中的数据结构。

LLC_UA是一个32位标识，表示LLC命令集中的Unknown字段。
LLC_DISC是一个32位标识，表示LLC命令集中的Discrim符。
LLC_DM是一个32位标识，表示LLC命令集中的Move�/Move史指令。
LLC_SABME是一个32位标识，表示LLC命令集中的SABME操作。
LLC_TEST是一个32位标识，表示LLC命令集中的Test操作。
LLC_XID是一个32位标识，表示LLC命令集中的XID操作。
LLC_FRMR是一个32位标识，表示LLC命令集中的FRMR操作。

LLC_S_CMD_MASK是一个32位标识，表示LLC命令集中的S_CMD_MASK字段。
LLC_RR是一个32位标识，表示LLC命令集中的R代币。
LLC_RNR是一个32位标识，表示LLC命令集中的R组。
LLC_REJ是一个32位标识，表示LLC命令集中的J。

LLC_IS_NR(is)是一个32位标识，表示is是否为NL。
LLC_I_NS(is)是一个32位标识，表示is是否为IN。

LLC_CMD_MASK是一个32位标识，表示LLC命令集中的所有标志。
LLC_LR(is)是一个32位标识，表示is是否为LR。
LLC_LRC(is)是一个32位标识，表示is是否为LRC。
LLC_HDR(is)是一个32位标识，表示is是否为HDR。

LLC_ACTIVITY_MASK是一个32位标识，表示LLC命令集中的活动状态。
LLC_PENDING_ACTIVITY_MASK是一个32位标识，表示LLC命令集中的等待激活状态。
LLC_ACTIVE_ACTIVITY_MASK是一个32位标识，表示LLC命令集中的活动状态。

LLC_PENDING_PAYMENT_MASK是一个32位标识，表示LLC命令集中的等待付款状态。
LLC_PAYMENT_RECEIVED_MASK是一个32位标识，表示LLC命令集中的付款已收到状态。
LLC_PAYMENT_RECEIVED_ACCOUNT_MASK是一个32位标识，表示LLC命令集中的付款已收到，并且该付款的接收方是账户。


```cpp
#define	LLC_UA		0x63
#define	LLC_DISC	0x43
#define	LLC_DM		0x0f
#define	LLC_SABME	0x6f
#define	LLC_TEST	0xe3
#define	LLC_XID		0xaf
#define	LLC_FRMR	0x87

#define LLC_S_CMD_MASK	0x0f
#define	LLC_RR		0x0001
#define	LLC_RNR		0x0005
#define	LLC_REJ		0x0009

#define LLC_IS_NR(is)	(((is) >> 9) & 0x7f)
#define LLC_I_NS(is)	(((is) >> 1) & 0x7f)

```

这段代码定义了一个结构体 `LLCSAP_VALUES`，包含8个成员，分别是一个整型变量和8个布尔型变量。它们用于表示LL驱动程序与SAP之间的通信。值类型字段 `LLCSAP_NULL` 被定义为000000000000000000000000000000000000000，它的值为0；`LLCSAP_GLOBAL` 被定义为 0xff，它的值为255；`LLCSAP_8021B_I` 被定义为0x02，它的值为1；`LLCSAP_8021B_G` 被定义为0x03，它的值为1。


```cpp
/*
 * 802.2 LLC SAP values.
 */

#ifndef LLCSAP_NULL
#define	LLCSAP_NULL		0x00
#endif
#ifndef LLCSAP_GLOBAL
#define	LLCSAP_GLOBAL		0xff
#endif
#ifndef LLCSAP_8021B_I
#define	LLCSAP_8021B_I		0x02
#endif
#ifndef LLCSAP_8021B_G
#define	LLCSAP_8021B_G		0x03
```

这段代码定义了一些头文件，它们定义了与LLCSAP相关的预定义名称（LLCSAP_IP，LLCSAP_PROWAYNM，LLCSAP_8021D，LLCSAP_RS511，LLCSAP_ISO8208）。这些预定义名称将被用于定义LLCSAP类型的其他代码。

具体来说，`#ifdef LLCSAP_IP`定义了当定义LLCSAP_IP时，仅编译预处理指令。`#elif LLCSAP_PROWAYNM`定义了当定义LLCSAP_PROWAYNM时，仅编译预处理指令。以此类推，对于所有LLCSAP_XXXXX预定义名称，如果定义，则仅编译预处理指令。

注意，即使不定义任何LLCSAP类型，这些预定义名称也可能会被定义，因此预处理指令可能会被编译多次。


```cpp
#endif
#ifndef LLCSAP_IP
#define	LLCSAP_IP		0x06
#endif
#ifndef LLCSAP_PROWAYNM
#define	LLCSAP_PROWAYNM		0x0e
#endif
#ifndef LLCSAP_8021D
#define	LLCSAP_8021D		0x42
#endif
#ifndef LLCSAP_RS511
#define	LLCSAP_RS511		0x4e
#endif
#ifndef LLCSAP_ISO8208
#define	LLCSAP_ISO8208		0x7e
```

这段代码定义了四个头文件，它们描述了与LLCSAP库相关的预定义宏。

具体来说，这段代码定义了以下宏：

- LLCSAP_PROWAY：表示当前工作方式为启用。
- LOCLSAP_PROWAY：表示当前工作方式为启用。这里的“LLCSAP”是一个前缀，表示这是LLCSAP库。
- LOCLSAP_SNAP：表示当前工作方式为启用。这里的“LLCSAP”同样是一个前缀，表示这是LLCSAP库。
- LOCLSAP_IPX：表示当前工作方式为启用。这里的“LLCSAP”仍然是一个前缀，表示这是LLCSAP库。
- LLCSAP_NETBEUI：表示当前工作方式为启用。这里的“LLCSAP”仍然是一个前缀，表示这是LLCSAP库。
- LLCSAP_ISONS：表示当前工作方式为启用。这里的“LLCSAP”仍然是一个前缀，表示这是LLCSAP库。这里的“ISONS”表示这是一个由多个LLCSAP库组成的LLCSAP库。

此外，还有一条#ifdef LLCSAP_PROWAY和#define LLCSAP_PROWAY 0x8e，这条代码表示如果当前工作方式为启用，那么宏定义中的LLCSAP_PROWAY将替换为0x8e。


```cpp
#endif
#ifndef LLCSAP_PROWAY
#define	LLCSAP_PROWAY		0x8e
#endif
#ifndef LLCSAP_SNAP
#define	LLCSAP_SNAP		0xaa
#endif
#ifndef LLCSAP_IPX
#define LLCSAP_IPX		0xe0
#endif
#ifndef LLCSAP_NETBEUI
#define LLCSAP_NETBEUI		0xf0
#endif
#ifndef LLCSAP_ISONS
#define	LLCSAP_ISONS		0xfe
```

这段代码是一个预处理指令，它的作用是在源代码文件被编译之前，检查是否已经定义了某些特定的变量或函数。

这里，#endif指令是一个预处理指令，它本身不具有任何实际的语义，但是它会检查一个名为"__defsizeof"的定义是否已经被定义。如果这个定义已经被定义，那么编译器会忽略这个定义，否则会报错。

__defsizeof是一个C预处理指令，它的作用是在编译时计算一个变量或函数的大小。这个指令的作用是在编译时知道某些变量的尺寸，从而避免在运行时出现一些未定义的行为。

总的来说，这段代码的作用是告诉编译器在编译之前检查一个预定义的指令是否已经被定义，以避免编译错误。


```cpp
#endif

```

# `libpcap/nametoaddr.c`

这段代码定义了一个头文件，其中包含一个声明，表示该头文件及其所有者享有版权。这个头文件定义了一种名为“Redistribution and use in source and binary forms, with or without modification”的条件，表示可以对源代码或二进制形式进行分发和修改。同时，该头文件还声明在二进制文件中包含上述版权通知，并且在广告材料中包含相应的说明。最后，该头文件指出该软件由“The University of California, Lawrence Berkeley Laboratory and its contributors”开发，并声明不得以任何方式通过暗示或保证的方式宣传或推广该软件。


```cpp
/*
 * Copyright (c) 1990, 1991, 1992, 1993, 1994, 1995, 1996, 1997, 1998
 *	The Regents of the University of California.  All rights reserved.
 *
 * Redistribution and use in source and binary forms, with or without
 * modification, are permitted provided that: (1) source code distributions
 * retain the above copyright notice and this paragraph in its entirety, (2)
 * distributions including binary code include the above copyright notice and
 * this paragraph in its entirety in the documentation or other materials
 * provided with the distribution, and (3) all advertising materials mentioning
 * features or use of this software display the following acknowledgement:
 * ``This product includes software developed by the University of California,
 * Lawrence Berkeley Laboratory and its contributors.'' Neither the name of
 * the University nor the names of its contributors may be used to endorse
 * or promote products derived from this software without specific prior
 * written permission.
 * THIS SOFTWARE IS PROVIDED ``AS IS'' AND WITHOUT ANY EXPRESS OR IMPLIED
 * WARRANTIES, INCLUDING, WITHOUT LIMITATION, THE IMPLIED WARRANTIES OF
 * MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE.
 *
 * Name to id translation routines used by the scanner.
 * These functions are not time critical.
 */

```

这段代码的作用是检查是否支持配置文件（CONFIG_H）以及是否支持网络库（DECNETLIB）。如果配置文件支持，那么会包含网络库。如果配置文件不支持，则需要包含一些头文件来使程序正确地编译。 

具体来说：

1. 如果当前系统支持config.h配置文件，那么包含以下头文件：<config.h>，<sys/types.h>，<netdnet/dnetdb.h>。

2. 如果当前系统不支持config.h配置文件，那么需要包含以下头文件：<winsock2.h>，<ws2tcpip.h>，<winsock2.h>，<ws2tcpip.h>。

3. 如果当前系统正在使用Windows 2000或更早版本，那么包含INET6头文件。

4. 如果当前系统支持配置文件，但是不支持网络库，那么包含以下头文件：<wspiapi.h>。

5. 如果当前系统不支持配置文件，并且不支持网络库，那么包含以下头文件：<ms738520.h>，<Ws2_32.h>，<Wswpapi.h>。

6. 如果当前系统正在使用Windows 2000或更早版本，但是不支持getaddrinfo函数，那么包含以下头文件：<Wspiapi.h>，<WspiapiGetAddrInfo.linen>。

综上所述，这段代码的作用是检查当前系统是否支持配置文件和网络库，并包含相应的头文件。


```cpp
#ifdef HAVE_CONFIG_H
#include <config.h>
#endif

#ifdef DECNETLIB
#include <sys/types.h>
#include <netdnet/dnetdb.h>
#endif

#ifdef _WIN32
  #include <winsock2.h>
  #include <ws2tcpip.h>

  #ifdef INET6
    /*
     * To quote the MSDN page for getaddrinfo() at
     *
     *    https://msdn.microsoft.com/en-us/library/windows/desktop/ms738520(v=vs.85).aspx
     *
     * "Support for getaddrinfo on Windows 2000 and older versions
     * The getaddrinfo function was added to the Ws2_32.dll on Windows XP and
     * later. To execute an application that uses this function on earlier
     * versions of Windows, then you need to include the Ws2tcpip.h and
     * Wspiapi.h files. When the Wspiapi.h include file is added, the
     * getaddrinfo function is defined to the WspiapiGetAddrInfo inline
     * function in the Wspiapi.h file. At runtime, the WspiapiGetAddrInfo
     * function is implemented in such a way that if the Ws2_32.dll or the
     * Wship6.dll (the file containing getaddrinfo in the IPv6 Technology
     * Preview for Windows 2000) does not include getaddrinfo, then a
     * version of getaddrinfo is implemented inline based on code in the
     * Wspiapi.h header file. This inline code will be used on older Windows
     * platforms that do not natively support the getaddrinfo function."
     *
     * We use getaddrinfo(), so we include Wspiapi.h here.
     */
    #include <wspiapi.h>
  #endif /* INET6 */
```

`ether_hostton` is a function that converts an Ethernet address from host byte order to network byte order. It takes two arguments: a byte order to convert and the destination network address. The function returns the byte order of the network address.

If the `ethereal` header file is included, the function is defined at `ethereal_ether_hostton().`

Note that the function has been deprecated in favor of the `inet_ntop()` function, which is a more convenient and platform-independent alternative for converting between host and network byte order. However, if you are using a platform that does not have the `ethereal` header file, you will need to include the `ethereal_ether_hostton()` function directly.


```cpp
#else /* _WIN32 */
  #include <sys/param.h>
  #include <sys/types.h>
  #include <sys/socket.h>
  #include <sys/time.h>

  #include <netinet/in.h>

  #ifdef HAVE_ETHER_HOSTTON
    #if defined(NET_ETHERNET_H_DECLARES_ETHER_HOSTTON)
      /*
       * OK, just include <net/ethernet.h>.
       */
      #include <net/ethernet.h>
    #elif defined(NETINET_ETHER_H_DECLARES_ETHER_HOSTTON)
      /*
       * OK, just include <netinet/ether.h>
       */
      #include <netinet/ether.h>
    #elif defined(SYS_ETHERNET_H_DECLARES_ETHER_HOSTTON)
      /*
       * OK, just include <sys/ethernet.h>
       */
      #include <sys/ethernet.h>
    #elif defined(ARPA_INET_H_DECLARES_ETHER_HOSTTON)
      /*
       * OK, just include <arpa/inet.h>
       */
      #include <arpa/inet.h>
    #elif defined(NETINET_IF_ETHER_H_DECLARES_ETHER_HOSTTON)
      /*
       * OK, include <netinet/if_ether.h>, after all the other stuff we
       * need to include or define for its benefit.
       */
      #define NEED_NETINET_IF_ETHER_H
    #else
      /*
       * We'll have to declare it ourselves.
       * If <netinet/if_ether.h> defines struct ether_addr, include
       * it.  Otherwise, define it ourselves.
       */
      #ifdef HAVE_STRUCT_ETHER_ADDR
        #define NEED_NETINET_IF_ETHER_H
      #else /* HAVE_STRUCT_ETHER_ADDR */
	struct ether_addr {
		unsigned char ether_addr_octet[6];
	};
      #endif /* HAVE_STRUCT_ETHER_ADDR */
    #endif /* what declares ether_hostton() */

    #ifdef NEED_NETINET_IF_ETHER_H
      #include <net/if.h>	/* Needed on some platforms */
      #include <netinet/in.h>	/* Needed on some platforms */
      #include <netinet/if_ether.h>
    #endif /* NEED_NETINET_IF_ETHER_H */

    #ifndef HAVE_DECL_ETHER_HOSTTON
      /*
       * No header declares it, so declare it ourselves.
       */
      extern int ether_hostton(const char *, struct ether_addr *);
    #endif /* !defined(HAVE_DECL_ETHER_HOSTTON) */
  #endif /* HAVE_ETHER_HOSTTON */

  #include <arpa/inet.h>
  #include <netdb.h>
```

这段代码是一个用于Windows操作系统的预处理指令。它通过包含头文件和函数指针，将定义的一些标准库函数包含到当前源文件中，从而允许我们在源文件中使用这些函数。

具体来说，这段代码的作用是：

1. 包含errno.h、stdlib.h、string.h、stdio.h这五个头文件，这些头文件分别定义了errno、stdlib、string、stdio这五个函数，用于定义错误报告、标准库函数、字符串操作、输入输出等功能。

2. 包含pcap-int.h、diag-control.h、gencode.h、nametoaddr.h这四个头文件，这些头文件定义了pcap、diag-control、gencode、nametoaddr这四个函数，用于实现网络工具的头文件。其中pcap是一个网络数据采集工具，diag-control是一个用于获取系统诊断信息的工具，gencode是一个用于将IP地址转换为MAC地址的函数，nametoaddr是一个用于将IP地址转换为MAC地址的函数。

3. 在源文件中引入了这五个头文件，这样就可以在源文件中使用errno.h、stdlib.h、string.h、stdio.h这五个函数，也可以使用pcap-int.h、diag-control.h、gencode.h、nametoaddr.h这四个函数。

总之，这段代码定义了多个头文件，通过引入它们，使得源文件中可以正常使用errno.h、stdlib.h、string.h、stdio.h、pcap-int.h、diag-control.h、gencode.h、nametoaddr.h这些函数，从而实现了对预处理的工作。


```cpp
#endif /* _WIN32 */

#include <errno.h>
#include <stdlib.h>
#include <string.h>
#include <stdio.h>

#include "pcap-int.h"

#include "diag-control.h"

#include "gencode.h"
#include <pcap/namedb.h>
#include "nametoaddr.h"

```

这段代码定义了两个宏定义：NTOHS 和 NTOHL。这两个宏定义了如何将主机名转换为互联网地址。

NTOHS macro 定义了将主机名转换为网络地址的函数，它的参数为 x。这个函数使用 ntohl 函数将主机名转换为网络地址，并将结果存储回 x。

NTOHL macro 定义了将主机名转换为本地地址的函数，它的参数为 x。这个函数使用 ntohs 函数将主机名转换为本地地址，并将结果存储回 x。

这两颗宏定义是通过预处理指令 #ifdef 和 #endif 分别定义的。当定义这两颗宏时，如果当前系统支持 OS-PROTO，那么就会包含这两个宏定义。

因此，这段代码的作用是定义了如何将主机名转换为互联网地址和本地地址的函数，分别使用 ntohl 和 ntohs 函数实现。


```cpp
#ifdef HAVE_OS_PROTO_H
#include "os-proto.h"
#endif

#ifndef NTOHL
#define NTOHL(x) (x) = ntohl(x)
#define NTOHS(x) (x) = ntohs(x)
#endif

/*
 *  Convert host name to internet address.
 *  Return 0 upon failure.
 *  XXX - not thread-safe; don't use it inside libpcap.
 */
bpf_u_int32 **
```

这段代码是一个用于输出IP地址的函数，它接受一个字符串参数。函数的作用是获取与给定名称相应的IP地址。它主要实现了以下几点：

1. 如果定义在当前头文件中，则直接使用`h_addr`常量作为输出；
2. 如果定义在`pcap_core.h`或`pcap_network_顺变`中，则采用`gethostbyname()`函数获取与给定名称相应的IP地址；
3. 如果定义在`pcap_abstract.h`中，则使用` deprecated_gethostbyname()`函数，该函数已经在Windows平台上被废弃；
4. 如果定义在`projections.h`中，则忽略此警告。


```cpp
pcap_nametoaddr(const char *name)
{
#ifndef h_addr
	static bpf_u_int32 *hlist[2];
#endif
	bpf_u_int32 **p;
	struct hostent *hp;

	/*
	 * gethostbyname() is deprecated on Windows, perhaps because
	 * it's not thread-safe, or because it doesn't support IPv6,
	 * or both.
	 *
	 * We deprecate pcap_nametoaddr() on all platforms because
	 * it's not thread-safe; we supply it for backwards compatibility,
	 * so suppress the deprecation warning.  We could, I guess,
	 * use getaddrinfo() and construct the array ourselves, but
	 * that's probably not worth the effort, as that wouldn't make
	 * this thread-safe - we can't change the API to require that
	 * our caller free the address array, so we still have to reuse
	 * a local array.
	 */
```

这段代码是一个用于在Linux系统上查找IP地址的函数，其主要作用是输出一条DIAG_OFF_DEPRECATION错误消息，并返回一个指向链表中IP地址的指针。

函数的具体实现如下：

1. 首先，函数通过调用`gethostbyname`函数来获取目标IP地址的名称。

2. 如果第一步操作成功，即获取到目标IP地址，那么函数将使用以下两种方式之一来存储该地址：

  a. 存储为链表中的第一个元素，类型为`bpf_u_int32`，表示为`hlist[0] = (bpf_u_int32 *)hp->h_addr;`。

  b. 否则，将遍历整个IP地址列表，并将每个地址的类型存储为`bpf_u_int32`，存储为`NTOHL(hp->h_addr_list[i]);`，其中`i`从链表的第二个元素开始。最后，返回整个IP地址列表的地址，类型为`bpf_u_int32 *hp->h_addr_list`。

3. 如果第一步操作失败，或者获取到的地址不在IP地址列表中，函数将返回`0`。

4. 最后，函数使用链表的方式返回IP地址列表，类型为`bpf_u_int32 **hp->h_addr_list`。


```cpp
DIAG_OFF_DEPRECATION
	if ((hp = gethostbyname(name)) != NULL) {
DIAG_ON_DEPRECATION
#ifndef h_addr
		hlist[0] = (bpf_u_int32 *)hp->h_addr;
		NTOHL(hp->h_addr);
		return hlist;
#else
		for (p = (bpf_u_int32 **)hp->h_addr_list; *p; ++p)
			NTOHL(**p);
		return (bpf_u_int32 **)hp->h_addr_list;
#endif
	}
	else
		return 0;
}

```

这段代码是一个用于在Linux系统中查找IP地址和网络接口的函数。它属于`struct addrinfo`类型，用于存储IP地址和网络接口的元数据。

函数接收一个字符串参数`name`，用于指定要查找的网络接口。函数首先定义了一个`struct addrinfo`变量`hints`，用于存储输入的地址信息。然后，它定义了一个指向`struct addrinfo`变量的指针`res`。

函数调用`getaddrinfo`函数来获取输入字符串`name`所表示的网络接口的地址信息。如果函数调用成功，它将返回`res`指向的地址，否则返回`NULL`。

如果错误，函数将返回`NULL`。


```cpp
struct addrinfo *
pcap_nametoaddrinfo(const char *name)
{
	struct addrinfo hints, *res;
	int error;

	memset(&hints, 0, sizeof(hints));
	hints.ai_family = PF_UNSPEC;
	hints.ai_socktype = SOCK_STREAM;	/*not really*/
	hints.ai_protocol = IPPROTO_TCP;	/*not really*/
	error = getaddrinfo(name, NULL, &hints, &res);
	if (error)
		return NULL;
	else
		return res;
}

```

这段代码的作用是尝试将一个网络名称转换为互联网地址，并在转换失败时返回0。它使用了以下方法来实现这个目的：

1. 首先检查操作系统是否支持从系统名称中获取网络名称。如果不支持，那么代码会直接返回0。
2. 如果操作系统支持从系统名称中获取网络名称，那么代码将尝试读取C:\Windows\System32\drivers\etc\networks文件中的网络名称。如果这个文件包含的是系统默认的网络名称（如127.0.0.1/0），那么转换为互联网地址的操作将会成功。否则，将尝试使用BSD代码中的一种方式（但是这个方法不一定能够成功）读取该文件中的网络名称。
3. 对于Windows 99或更早的版本，它可能无法从system32目录中的网络名称中获取正确答案，因此代码将返回0。


```cpp
/*
 *  Convert net name to internet address.
 *  Return 0 upon failure.
 *  XXX - not guaranteed to be thread-safe!  See below for platforms
 *  on which it is thread-safe and on which it isn't.
 */
#if defined(_WIN32) || defined(__CYGWIN__)
bpf_u_int32
pcap_nametonetaddr(const char *name _U_)
{
	/*
	 * There's no "getnetbyname()" on Windows.
	 *
	 * XXX - I guess we could use the BSD code to read
	 * C:\Windows\System32\drivers\etc/networks, assuming
	 * that's its home on all the versions of Windows
	 * we use, but that file probably just has the loopback
	 * network on 127/24 on 99 44/100% of Windows machines.
	 *
	 * (Heck, these days it probably just has that on 99 44/100%
	 * of *UN*X* machines.)
	 */
	return 0;
}
```

This function appears to be a reentrant implementation of the `getnetbyname()` and `getnetbyname_r()` functions in various Unix-like operating systems. It dynamically allocates a buffer for the return value and returns an error code depending on whether the function completes successfully or an error occurs.

The function uses various error handling mechanisms depending on the specific implementation. For example, the dynamically-allocated buffer is addressed as `buf` with a maximum size of `sizeof(buf)`. If the `getnetbyname()` or `getnetbyname_r()` functions return an error, the function checks for the error and returns `0`.

If the function successfully completes, it attempts to retrieve the network number associated with the given name using the `np` variable. If the function completes successfully, it returns the network number.

If the `getnetbyname()` function returns an error, the `getnetbyname_r()` function is called to retrieve the network number. If the `getnetbyname_r()` function also returns an error, the function dynamically allocates a buffer for the return value and returns the error code.

Overall, the function appears to be well-protected against errors and should work reliably for most use cases. However, since it depends on the specific implementation of the `getnetbyname()` and `getnetbyname_r()` functions, users may need to carefully consider the implications of using this function in their specific environment.


```cpp
#else /* _WIN32 */
bpf_u_int32
pcap_nametonetaddr(const char *name)
{
	/*
	 * UN*X.
	 */
	struct netent *np;
  #if defined(HAVE_LINUX_GETNETBYNAME_R)
	/*
	 * We have Linux's reentrant getnetbyname_r().
	 */
	struct netent result_buf;
	char buf[1024];	/* arbitrary size */
	int h_errnoval;
	int err;

	/*
	 * Apparently, the man page at
	 *
	 *    http://man7.org/linux/man-pages/man3/getnetbyname_r.3.html
	 *
	 * lies when it says
	 *
	 *    If the function call successfully obtains a network record,
	 *    then *result is set pointing to result_buf; otherwise, *result
	 *    is set to NULL.
	 *
	 * and, in fact, at least in some versions of GNU libc, it does
	 * *not* always get set if getnetbyname_r() succeeds.
	 */
	np = NULL;
	err = getnetbyname_r(name, &result_buf, buf, sizeof buf, &np,
	    &h_errnoval);
	if (err != 0) {
		/*
		 * XXX - dynamically allocate the buffer, and make it
		 * bigger if we get ERANGE back?
		 */
		return 0;
	}
  #elif defined(HAVE_SOLARIS_IRIX_GETNETBYNAME_R)
	/*
	 * We have Solaris's and IRIX's reentrant getnetbyname_r().
	 */
	struct netent result_buf;
	char buf[1024];	/* arbitrary size */

	np = getnetbyname_r(name, &result_buf, buf, (int)sizeof buf);
  #elif defined(HAVE_AIX_GETNETBYNAME_R)
	/*
	 * We have AIX's reentrant getnetbyname_r().
	 */
	struct netent result_buf;
	struct netent_data net_data;

	if (getnetbyname_r(name, &result_buf, &net_data) == -1)
		np = NULL;
	else
		np = &result_buf;
  #else
	/*
	 * We don't have any getnetbyname_r(); either we have a
	 * getnetbyname() that uses thread-specific data, in which
	 * case we're thread-safe (sufficiently recent FreeBSD,
	 * sufficiently recent Darwin-based OS, sufficiently recent
	 * HP-UX, sufficiently recent Tru64 UNIX), or we have the
	 * traditional getnetbyname() (everything else, including
	 * current NetBSD and OpenBSD), in which case we're not
	 * thread-safe.
	 */
	np = getnetbyname(name);
  #endif
	if (np != NULL)
		return np->n_net;
	else
		return 0;
}
```

这段代码的作用是转换一个端口号（port）名称到它的端口号（port）和协议类型（proto）。我们假设要转换的端口号只使用 TCP 或 UDP 协议。

具体实现中，首先定义了一个名为 `pcap_nametoport` 的函数，接受一个参数 `name`，表示要转换的端口号名称，以及三个整数参数 `port`、`proto`，分别表示目标端口号、协议类型和回传参数。

接着，通过调用 `gtest_assert_no_error` 函数来检查输入参数，如果没有错误，就执行下面的操作：

1. 获取输入参数的地址信息 `hints`，以及指向结果信息 `res` 的指针。
2. 创建一个指向 `ai` 的指针，用于获取输入地址信息。
3. 如果输入协议支持 IPv4 或 IPv6，就创建一个指向 `in4` 的指针，用于获取输入的 IPv4 或 IPv6 地址信息。
4. 调用 `res->ai_family` 函数，将输入的协议类型（AF）设置为 `AF_INET`，然后将输入的端口号（port）复制到 `res->ai_port` 字段中。
5. 调用 `res->ai_协议` 函数，将输入的协议类型（协议类型）复制到 `res->ai_proto` 字段中。
6. 如果输入协议支持回传参数，就创建一个指向 `in6` 的指针，用于获取输入的 IPv6 地址信息。
7. 调用 `res->ai_info` 函数，将输入的地址信息、协议信息设置为默认值，然后将结果信息 `res` 返回。

最后，如果输入参数存在错误，就调用 `gtest_assert_error` 函数输出错误信息。


```cpp
#endif /* _WIN32 */

/*
 * Convert a port name to its port and protocol numbers.
 * We assume only TCP or UDP.
 * Return 0 upon failure.
 */
int
pcap_nametoport(const char *name, int *port, int *proto)
{
	struct addrinfo hints, *res, *ai;
	int error;
	struct sockaddr_in *in4;
#ifdef INET6
	struct sockaddr_in6 *in6;
```

这段代码是用来设置服务器套接字的。具体来说，它将尝试连接到本地TCP或UDP套接字，并在套接字上绑定一个TCP或UDP套接字。如果连接成功，它将返回套接字的IP地址和端口号。如果连接失败，它将返回错误代码。


```cpp
#endif
	int tcp_port = -1;
	int udp_port = -1;

	/*
	 * We check for both TCP and UDP in case there are
	 * ambiguous entries.
	 */
	memset(&hints, 0, sizeof(hints));
	hints.ai_family = PF_UNSPEC;
	hints.ai_socktype = SOCK_STREAM;
	hints.ai_protocol = IPPROTO_TCP;
	error = getaddrinfo(NULL, name, &hints, &res);
	if (error != 0) {
		if (error != EAI_NONAME &&
		    error != EAI_SERVICE) {
			/*
			 * This is a real error, not just "there's
			 * no such service name".
			 * XXX - this doesn't return an error string.
			 */
			return 0;
		}
	} else {
		/*
		 * OK, we found it.  Did it find anything?
		 */
		for (ai = res; ai != NULL; ai = ai->ai_next) {
			/*
			 * Does it have an address?
			 */
			if (ai->ai_addr != NULL) {
				/*
				 * Yes.  Get a port number; we're done.
				 */
				if (ai->ai_addr->sa_family == AF_INET) {
					in4 = (struct sockaddr_in *)ai->ai_addr;
					tcp_port = ntohs(in4->sin_port);
					break;
				}
```

It looks like you are trying to establish a connection to a server using the IPv6 protocol. You are using the `getaddrinfo()` function to obtain the IPv6 address of the server, but you are also specifying the IPv4 protocol when you are trying to use the `sin_port()` function to get the port number. This is not a valid combination and you are likely to get a bug or an error.

You should fix the above mistake by using the `sin6_port()` function instead of `sin_port()` and make sure you are using the IPv6 protocol.

Also, you should check the return value of the `getaddrinfo()` function to see if it is successful. If it is successful, then you should print a message to indicate that the connection was successful. If it's not successful, you should print a message to indicate the failure.


```cpp
#ifdef INET6
				if (ai->ai_addr->sa_family == AF_INET6) {
					in6 = (struct sockaddr_in6 *)ai->ai_addr;
					tcp_port = ntohs(in6->sin6_port);
					break;
				}
#endif
			}
		}
		freeaddrinfo(res);
	}

	memset(&hints, 0, sizeof(hints));
	hints.ai_family = PF_UNSPEC;
	hints.ai_socktype = SOCK_DGRAM;
	hints.ai_protocol = IPPROTO_UDP;
	error = getaddrinfo(NULL, name, &hints, &res);
	if (error != 0) {
		if (error != EAI_NONAME &&
		    error != EAI_SERVICE) {
			/*
			 * This is a real error, not just "there's
			 * no such service name".
			 * XXX - this doesn't return an error string.
			 */
			return 0;
		}
	} else {
		/*
		 * OK, we found it.  Did it find anything?
		 */
		for (ai = res; ai != NULL; ai = ai->ai_next) {
			/*
			 * Does it have an address?
			 */
			if (ai->ai_addr != NULL) {
				/*
				 * Yes.  Get a port number; we're done.
				 */
				if (ai->ai_addr->sa_family == AF_INET) {
					in4 = (struct sockaddr_in *)ai->ai_addr;
					udp_port = ntohs(in4->sin_port);
					break;
				}
```

这段代码的作用是检查网络接口是否支持IPv6协议，并根据支持的协议修改UDP端口号。

具体来说，代码首先检查输入的地址是否属于INET6协议，如果是，则将in6指向该地址的INET6部分，然后获取该地址的sin6部分表示的UDP端口号，最后将UDP端口号作为第一个参数传递给函数，以判断是否匹配已知的端口号。

如果该地址不属于INET6协议，则执行else语句， 在if语句中添加一条语句，用于在以后可能会遇到同样in6地址的情况下，能够方便的通过更改port来判断ipv6和ipv4哪些协议在做服务，这样在不清楚ipv6和ipv4哪些协议在做服务的时候，也不会迷惑不知道该做哪些事情。

接下来是寻找/etc/services文件，解析/etc/services文件，如果找到了存在就是/etc/services文件中的service name=* whatever，修改该服务的port为tcp_port，如果找到的就是*udp_port=* whatever，udp_port的值就是tcp_port的值，这样在以后通过该服务的时候，就可以通过该服务的port来判断是ipv6还是ipv4。

最后，还有一条if语句，用于判断tcp_port是否为0，如果是，则认为当前系统只支持ipv4协议，*proto=PROTO_UNDEF；否则就认为当前系统支持ipv6协议，*proto=IPPROTO_TCP。


```cpp
#ifdef INET6
				if (ai->ai_addr->sa_family == AF_INET6) {
					in6 = (struct sockaddr_in6 *)ai->ai_addr;
					udp_port = ntohs(in6->sin6_port);
					break;
				}
#endif
			}
		}
		freeaddrinfo(res);
	}

	/*
	 * We need to check /etc/services for ambiguous entries.
	 * If we find an ambiguous entry, and it has the
	 * same port number, change the proto to PROTO_UNDEF
	 * so both TCP and UDP will be checked.
	 */
	if (tcp_port >= 0) {
		*port = tcp_port;
		*proto = IPPROTO_TCP;
		if (udp_port >= 0) {
			if (udp_port == tcp_port)
				*proto = PROTO_UNDEF;
```

这段代码是一个条件判断语句，它会根据两个条件来判断是否执行特定的操作。如果没有满足任何一个条件，则会输出一条警告信息。

具体来说，首先判断是否定义了 `notdef`，如果不定义，则会执行下面的代码块。这个代码块包含一个带参数的函数，会接收一个名为 `name` 的参数，然后输出一条警告信息。如果已经定义了 `notdef`，则直接跳过这个代码块，否则执行下面的代码块。

在执行 `warning` 函数时，会根据传递给它的 `name` 参数来生成警告信息。如果 `name` 是一个没有前缀的变量名，则认为这是一个未定义的名称，因此会输出一条警告信息，警告信息中包含 `name` 变量和未定义的名称。


```cpp
#ifdef notdef
			else
				/* Can't handle ambiguous names that refer
				   to different port numbers. */
				warning("ambiguous port %s in /etc/services",
					name);
#endif
		}
		return 1;
	}
	if (udp_port >= 0) {
		*port = udp_port;
		*proto = IPPROTO_UDP;
		return 1;
	}
```

这段代码是一个条件判断语句，它会根据一个名为 "ultrix" 的定义来判断是否支持 NFS（Network File System）服务。如果 NFS 服务没有被安装，那么它会执行以下操作：

1. 如果名称 "nfs" 与 "nfs" 相等，那么将设备的端口设置为 2049，并将协议类型设置为 undefined（未定义），返回 1。
2. 如果 NFS 服务已经安装，那么不做任何操作，直接返回 0。

这里用到了 C 语言的 `#if defined(某种) || defined(__osf__)` 语法，它会判断某个标识符是否已经被定义过。如果已经定义过，则不做操作，否则执行特定的代码块。


```cpp
#if defined(ultrix) || defined(__osf__)
	/* Special hack in case NFS isn't in /etc/services */
	if (strcmp(name, "nfs") == 0) {
		*port = 2049;
		*proto = PROTO_UNDEF;
		return 1;
	}
#endif
	return 0;
}

/*
 * Convert a string in the form PPP-PPP, where correspond to ports, to
 * a starting and ending port in a port range.
 * Return 0 on failure.
 */
```

该代码是一个名为 `pcap_nametoportrange` 的函数，它用于根据传入的网络名称（`name`）查找相应的端口号（`port1` 和 `port2`）和协议类型（`proto`）。

函数首先通过 `sscanf` 函数将输入的网络名称转换为两个整数，分别存储在 `p1` 和 `p2` 变量中。如果转换失败，函数将返回 0。

接着，函数通过 `strchr` 函数尝试从字符串 `name` 中找到表示端口号的子字符串，如果找到，则将子字符串存储在 `off` 变量中，并将 `name` 字符串的后续字符存储在一个空字符串中。如果 `name` 中找不到表示端口号的子字符串，函数将返回 0。

然后，函数使用 `pcap_namt` 函数根据 `name` 查找相应的端口号，并将获取到的端口号存储在 `port1` 和 `port2` 变量中。如果查找失败，函数将返回 0。

接下来，函数通过 `strcpy` 函数将找到的端口号名（`name`）存储在 `cpy` 变量中，并将 `name` 字符串中的后续字符删除。如果 `name` 字符串越界，`cpy` 变量将不再有效。

最后，函数根据 `port1` 和 `port2` 的值尝试使用 `pcap_ndr` 函数设置正确的协议类型（`proto`）。如果设置失败，函数将返回 `PROTO_UNDEF`。

如果所有参数都被正确传递，函数返回 1，否则返回 0。


```cpp
int
pcap_nametoportrange(const char *name, int *port1, int *port2, int *proto)
{
	u_int p1, p2;
	char *off, *cpy;
	int save_proto;

	if (sscanf(name, "%d-%d", &p1, &p2) != 2) {
		if ((cpy = strdup(name)) == NULL)
			return 0;

		if ((off = strchr(cpy, '-')) == NULL) {
			free(cpy);
			return 0;
		}

		*off = '\0';

		if (pcap_nametoport(cpy, port1, proto) == 0) {
			free(cpy);
			return 0;
		}
		save_proto = *proto;

		if (pcap_nametoport(off + 1, port2, proto) == 0) {
			free(cpy);
			return 0;
		}
		free(cpy);

		if (*proto != save_proto)
			*proto = PROTO_UNDEF;
	} else {
		*port1 = p1;
		*port2 = p2;
		*proto = PROTO_UNDEF;
	}

	return 1;
}

```

It looks like the `getprotobyname_r()` function is not defined in the `<protobyname.h>` header file that is provided with the FreeBSD, NetBSD, OpenBSD, and current AIX versions of the protobuf installation.

It is possible that the header file provides a different function or behavior for `getprotobyname_r()` in some cases. Without more information, it is difficult to determine the exact behavior of the function in your specific case.

If you need to use the `getprotobyname_r()` function in your code, you may need to dynamically allocate a buffer with a larger size using the `malloc()` function, as the current implementation of `getprotobyname_r()` expects an arbitrary size buffer. Alternatively, you may need to modify the behavior of `getprotobyname_r()` to handle thread-specific data or other exceptions that it may encounter.

I would recommend consulting the documentation for the `getprotobyname_r()` function to determine the correct behavior in your specific case.


```cpp
/*
 * XXX - not guaranteed to be thread-safe!  See below for platforms
 * on which it is thread-safe and on which it isn't.
 */
int
pcap_nametoproto(const char *str)
{
	struct protoent *p;
  #if defined(HAVE_LINUX_GETNETBYNAME_R)
	/*
	 * We have Linux's reentrant getprotobyname_r().
	 */
	struct protoent result_buf;
	char buf[1024];	/* arbitrary size */
	int err;

	err = getprotobyname_r(str, &result_buf, buf, sizeof buf, &p);
	if (err != 0) {
		/*
		 * XXX - dynamically allocate the buffer, and make it
		 * bigger if we get ERANGE back?
		 */
		return 0;
	}
  #elif defined(HAVE_SOLARIS_IRIX_GETNETBYNAME_R)
	/*
	 * We have Solaris's and IRIX's reentrant getprotobyname_r().
	 */
	struct protoent result_buf;
	char buf[1024];	/* arbitrary size */

	p = getprotobyname_r(str, &result_buf, buf, (int)sizeof buf);
  #elif defined(HAVE_AIX_GETNETBYNAME_R)
	/*
	 * We have AIX's reentrant getprotobyname_r().
	 */
	struct protoent result_buf;
	struct protoent_data proto_data;

	if (getprotobyname_r(str, &result_buf, &proto_data) == -1)
		p = NULL;
	else
		p = &result_buf;
  #else
	/*
	 * We don't have any getprotobyname_r(); either we have a
	 * getprotobyname() that uses thread-specific data, in which
	 * case we're thread-safe (sufficiently recent FreeBSD,
	 * sufficiently recent Darwin-based OS, sufficiently recent
	 * HP-UX, sufficiently recent Tru64 UNIX, Windows), or we have
	 * the traditional getprotobyname() (everything else, including
	 * current NetBSD and OpenBSD), in which case we're not
	 * thread-safe.
	 */
	p = getprotobyname(str);
  #endif
	if (p != 0)
		return p->p_proto;
	else
		return PROTO_UNDEF;
}

```

这段代码定义了一个名为etproto的结构体，其中包含两个成员变量：一个字符串s和一个字节数组p。然后，它定义了一个名为efromep接口，该接口定义了etheropt类型。接着定义了一个名为ethtop的函数，该函数将一个etheropt类型的数据作为参数并返回。最后，在函数头部声明了ethtop函数，但没有实现该函数的具体行为。


```cpp
#include "ethertype.h"

struct eproto {
	const char *s;
	u_short p;
};

/*
 * Static data base of ether protocol types.
 * tcpdump used to import this, and it's declared as an export on
 * Debian, at least, so make it a public symbol, even though we
 * don't officially export it by declaring it in a header file.
 * (Programs *should* do this themselves, as tcpdump now does.)
 *
 * We declare it here, right before defining it, to squelch any
 * warnings we might get from compilers about the lack of a
 * declaration.
 */
```

这段代码定义了一个名为eproto的结构体数组eproto_db，其中每个结构体包含一个网络协议类型和一个协议名称。这个数组是为了存储一系列网络协议的信息，以便在代码中进行使用。

eproto_db数组中共包含了以下协议类型和协议名称：

- Ethernet类型：AARP,ARP,ATALK,DN,IP,IPv6
-链路层类型：iOS,IPv4,iOS,iOS,iOS,iOS
-数据链路层类型：iOS,IP,iOS,iOS,iOS,iOS

这些协议类型和名称都是苹果公司的iOS操作系统中支持和使用的一种或多种链路层协议。


```cpp
PCAP_API struct eproto eproto_db[];
PCAP_API_DEF struct eproto eproto_db[] = {
	{ "aarp", ETHERTYPE_AARP },
	{ "arp", ETHERTYPE_ARP },
	{ "atalk", ETHERTYPE_ATALK },
	{ "decnet", ETHERTYPE_DN },
	{ "ip", ETHERTYPE_IP },
#ifdef INET6
	{ "ip6", ETHERTYPE_IPV6 },
#endif
	{ "lat", ETHERTYPE_LAT },
	{ "loopback", ETHERTYPE_LOOPBACK },
	{ "mopdl", ETHERTYPE_MOPDL },
	{ "moprc", ETHERTYPE_MOPRC },
	{ "rarp", ETHERTYPE_REVARP },
	{ "sca", ETHERTYPE_SCA },
	{ (char *)0, 0 }
};

```

这段代码定义了一个名为 `int` 的整型变量 `pcap_nametoeproto`，以及一个名为 `const char *s` 的字符型指针变量。

该函数的作用是根据传入的 `s` 字符串，返回相应的 `struct eproto` 结构体指针，其中 `eproto_db` 是一个预定义的结构体，包含一个指向 `struct eproto` 的指针。

函数的具体实现可以分为两个步骤：

1. 遍历 `eproto_db` 结构体数组，对于每个结构体节点，使用 `strcmp` 函数与传入的 `s` 字符串进行比较，如果两个字符串相等，则返回该结构体节点所指的对象，即返回 `eproto_db` 结构体中的下一个节点。
2. 如果遍历完成后仍然没有找到与 `s` 字符串相等的结构体节点，则返回一个名为 `PROTO_UNDEF` 的固定值。

该函数可以在 `llc.h` 头文件中使用，因此它应该是从 `llc` 包中导入的函数。


```cpp
int
pcap_nametoeproto(const char *s)
{
	struct eproto *p = eproto_db;

	while (p->s != 0) {
		if (strcmp(p->s, s) == 0)
			return p->p;
		p += 1;
	}
	return PROTO_UNDEF;
}

#include "llc.h"

```

这段代码定义了一个结构体数组 `llc_db`, 它存储了 LLC 值的标准编码。

接下来定义了一个名为 `pcap_nametollc` 的函数，它接收一个字符串参数 `s`，返回该字符串对应的 LLC 标准编码。

在函数内部， 创建了一个名为 `p` 的指针， 并将其指向数组 `llc_db` 的第一个元素。然后循环遍历 `llc_db` 数组， 如果找到当前字符串 `s`，就返回该字符串所对应的元素， 否则继续遍历下一个元素。

最后， 如果遍历完整个 `llc_db` 数组都没有找到匹配的字符串， 函数返回一个名为 `PROTO_UNDEF` 的保留值。


```cpp
/* Static data base of LLC values. */
static struct eproto llc_db[] = {
	{ "iso", LLCSAP_ISONS },
	{ "stp", LLCSAP_8021D },
	{ "ipx", LLCSAP_IPX },
	{ "netbeui", LLCSAP_NETBEUI },
	{ (char *)0, 0 }
};

int
pcap_nametollc(const char *s)
{
	struct eproto *p = llc_db;

	while (p->s != 0) {
		if (strcmp(p->s, s) == 0)
			return p->p;
		p += 1;
	}
	return PROTO_UNDEF;
}

```

这段代码定义了一个名为`xdtoi`的函数来将一个十六进制数字转换为8位无符号整数。函数接受一个单字符类型的输入参数`c`，并返回一个无符号整数类型的结果。函数实现中，如果输入参数`c`在`'0'`和`'9'`之间，则直接将其转换为8位无符号整数并返回。否则，如果`c`在`'a'`和`'f'`之间，则将其转换为10位无符号整数，并在10的基础上加10，以便与`'a'`和`'f'`对应。如果输入参数`c`不是一个有效的十六进制数字，函数将返回-1。

接下来，定义了一个名为`__pcap_atoin`的函数，该函数将一个32位的无符号整数地址赋给一个指向内存的指针`addr`。函数接受一个字符串类型的输入参数`s`，并将其中的地址解析为32位的无符号整数。函数实现中，将输入参数`s`中的地址作为32位无符号整数，并将其与0合并。然后，将合并后的字符串作为32位无符号整数输入到`addr`指向的内存位置，并将输入的8个字节复制到输入的字符串中。最后，函数返回输入的字符串的长度。

由于这两段代码没有注释，因此不知道它们如何被使用以及如何设置它们的行为。


```cpp
/* Hex digit to 8-bit unsigned integer. */
static inline u_char
xdtoi(u_char c)
{
	if (c >= '0' && c <= '9')
		return (u_char)(c - '0');
	else if (c >= 'a' && c <= 'f')
		return (u_char)(c - 'a' + 10);
	else
		return (u_char)(c - 'A' + 10);
}

int
__pcap_atoin(const char *s, bpf_u_int32 *addr)
{
	u_int n;
	int len;

	*addr = 0;
	len = 0;
	for (;;) {
		n = 0;
		while (*s && *s != '.') {
			if (n > 25) {
				/* The result will be > 255 */
				return -1;
			}
			n = n * 10 + *s++ - '0';
		}
		if (n > 255)
			return -1;
		*addr <<= 8;
		*addr |= n & 0xff;
		len += 8;
		if (*s == '\0')
			return len;
		++s;
	}
	/* NOTREACHED */
}

```

这段代码是一个名为`__pcap_atodn`的函数，它是`pcap`库中的一个用于从给定字符串 `s` 中解析出 `IP` 结构体中的 `source_address` 和 `port` 字段的函数。

具体来说，这段代码的实现包括以下几个步骤：

1. 如果 `s` 字符串无法解析出两个整数，则直接返回 0，表示函数失败。

2. 解析出两个整数 `area` 和 `node`。

3. 将 `area` 整数左移 10 位并在最高位添加符号位，得到一个 32 位二进制数，存入 `addr` 变量中。

4. 将 `node` 整数左移 8 位并在最高位添加符号位，得到一个 16 位二进制数，与 `NODEMASK` 进行按位与运算，得到一个 16 位二进制数，存入 `addr` 变量中。

5. 将上面两个结果按位或操作得到一个 32 位二进制数，存入 `addr` 变量中。

6. 函数返回 32，表示成功解析出 `source_address` 和 `port` 两个字段。


```cpp
int
__pcap_atodn(const char *s, bpf_u_int32 *addr)
{
#define AREASHIFT 10
#define AREAMASK 0176000
#define NODEMASK 01777

	u_int node, area;

	if (sscanf(s, "%d.%d", &area, &node) != 2)
		return(0);

	*addr = (area << AREASHIFT) & AREAMASK;
	*addr |= (node & NODEMASK);

	return(32);
}

```

这段代码定义了一个名为 `pcap_ether_aton` 的函数，它的作用是将一个字符串（以冒号或分号分隔的数字）转换为一个以太网地址。函数接受一个 `const char *` 类型的参数 `s`。

函数内部先定义了一个名为 `ep` 的局部变量，用于存储目的以太网地址；定义了一个名为 `e` 的局部变量，用于存储已解析过的以太网地址。

函数内部使用一个循环从 `s` 开始遍历，对于每个字符，先判断其是否为冒号、分号或破折号，如果是，则将 `s` 所代表的数字加 1。然后将 `d` 的值左移 4 位，再乘以 16 以得到其对应的二进制数字。接着将 `d` 和 `e` 所存储的值进行按位与运算，最后将结果存储在 `ep` 指向的内存位置。

最后，函数返回一个指向已经解析过的以太网地址的指针 `e`。


```cpp
/*
 * Convert 's', which can have the one of the forms:
 *
 *	"xx:xx:xx:xx:xx:xx"
 *	"xx.xx.xx.xx.xx.xx"
 *	"xx-xx-xx-xx-xx-xx"
 *	"xxxx.xxxx.xxxx"
 *	"xxxxxxxxxxxx"
 *
 * (or various mixes of ':', '.', and '-') into a new
 * ethernet address.  Assumes 's' is well formed.
 */
u_char *
pcap_ether_aton(const char *s)
{
	register u_char *ep, *e;
	register u_char d;

	e = ep = (u_char *)malloc(6);
	if (e == NULL)
		return (NULL);

	while (*s) {
		if (*s == ':' || *s == '.' || *s == '-')
			s += 1;
		d = xdtoi(*s++);
		if (PCAP_ISXDIGIT(*s)) {
			d <<= 4;
			d |= xdtoi(*s++);
		}
		*ep++ = d;
	}

	return (e);
}

```

这段代码定义了一个名为 `pcap_ether_hostton` 的函数，它用于将 `name` 参数的以太坊主机名转换为相应的字节数组。

该函数首先检查是否已初始化，如果未初始化，则需要通过 `fopen` 函数将 PCAP EtherType 文件读取到内存中，并将其余的初始化逻辑作为参数传递给 `pcap_next_etherical` 函数。

如果已经初始化，则使用 `pcap_next_etherical` 函数从文件中读取 PCAP EtherType 数据，并将其与给定的 `name` 比较。如果是，则使用 `malloc` 函数在内存中分配一个 6 字节长的字节数组，并将 `ethereal_addr_copy` 函数的第一个参数设置为该 `name`，第二个参数设置为 `NULL`，然后将字节数组中的字节复制到该指针上，最后将结果返回。

如果 `pcap_ether_hostton` 函数在多次调用时仍然无法正确地将 `name` 转换为相应的以太坊主机名，那么它将返回一个 `NULL` 表示错误。


```cpp
#ifndef HAVE_ETHER_HOSTTON
/*
 * Roll our own.
 * XXX - not thread-safe, because pcap_next_etherent() isn't thread-
 * safe!  Needs a mutex or a thread-safe pcap_next_etherent().
 */
u_char *
pcap_ether_hostton(const char *name)
{
	register struct pcap_etherent *ep;
	register u_char *ap;
	static FILE *fp = NULL;
	static int init = 0;

	if (!init) {
		fp = fopen(PCAP_ETHERS_FILE, "r");
		++init;
		if (fp == NULL)
			return (NULL);
	} else if (fp == NULL)
		return (NULL);
	else
		rewind(fp);

	while ((ep = pcap_next_etherent(fp)) != NULL) {
		if (strcmp(ep->name, name) == 0) {
			ap = (u_char *)malloc(6);
			if (ap != NULL) {
				memcpy(ap, ep->addr, 6);
				return (ap);
			}
			break;
		}
	}
	return (NULL);
}
```

这段代码定义了一个名为 `pcap_ether_hostton` 的函数，其作用是使用操作系统提供的函数，将一个串转换为以太网地址。

该函数接收一个字符串参数 `name`，并将其转换为一个字符数组。然后，该函数使用 `ether_hostton` 函数将该字符串转换为一个 `etherether_addr` 类型的数据结构，将其存储在 `a` 数组中。如果 `ether_hostton` 函数成功转换，则将结果存储在 `ap` 数组中，该数组长度为 6。最后，函数返回生成的 `ap` 数组。

由于 `pcap_ether_hostton` 函数中使用了 `malloc` 函数来分配内存，因此在函数实现中需要包含 `malloc` 函数的定义。


```cpp
#else
/*
 * Use the OS-supplied routine.
 * This *should* be thread-safe; the API doesn't have a static buffer.
 */
u_char *
pcap_ether_hostton(const char *name)
{
	register u_char *ap;
	u_char a[6];
	char namebuf[1024];

	/*
	 * In AIX 7.1 and 7.2: int ether_hostton(char *, struct ether_addr *);
	 */
	pcap_strlcpy(namebuf, name, sizeof(namebuf));
	ap = NULL;
	if (ether_hostton(namebuf, (struct ether_addr *)a) == 0) {
		ap = (u_char *)malloc(6);
		if (ap != NULL)
			memcpy((char *)ap, (char *)a, 6);
	}
	return (ap);
}
```

这段代码是一个#ifdef中标记的函数，名字为__pcap_nametodnaddr。它接受一个名为name的参数，并返回一个u_short类型的变量res，其值为name到相应节点对象的地址的解码。

函数的实现主要分两个步骤：

1. 获取一个名为name的节点对象，这可能需要从libxml或其他类似库中获取。
2. 将获取到的节点对象的地址复制到一个u_short类型的变量res中。

getnodebyname函数用于获取给定name的节点对象。如果name无法被找到，函数返回一个null pointer。注意，这个函数需要libxml或其他类似库的支持，因此在使用时需要检查lib是否已经安装。

整段代码的作用是获取给定名称的节点对象的地址并将其复制到res变量中，然后返回一个布尔值，表示函数是否成功执行。需要注意的是，由于没有提供函数的可移植性，因此需要根据实际使用场景进行调整。


```cpp
#endif

/*
 * XXX - not guaranteed to be thread-safe!
 */
int
#ifdef	DECNETLIB
__pcap_nametodnaddr(const char *name, u_short *res)
{
	struct nodeent *getnodebyname();
	struct nodeent *nep;

	nep = getnodebyname(name);
	if (nep == ((struct nodeent *)0))
		return(0);

	memcpy((char *)res, (char *)nep->n_addr, sizeof(unsigned short));
	return(1);
```

这段代码是一个名为`__pcap_nametodnaddr`的函数，它是排斥测试中的一个函数。函数接受两个参数，一个是字符指针`name`，另一个是用户自定义类型`u_short`的指针`res`。函数的作用是在函数开始时返回0，在函数体中执行以下操作：

1. 如果`name`所指向的内存区域以`__pcap_nametodnaddr`为名，那么返回0。
2. 否则，函数体中执行以下操作：

	1. 将`name`所指向的内存区域复制到一个名为`res`的内存区域中。
	2. 由于`name`是一个字符指针，将其解包为字符，并将其存储在`res`所指向的内存区域中。
	3. 由于`res`是一个用户自定义类型，其数据类型为`u_short`，所以将其存储在`res`所指向的内存区域中。
	4. 函数结束时返回0。


```cpp
#else
__pcap_nametodnaddr(const char *name _U_, u_short *res _U_)
{
	return(0);
#endif
}

```