# Nmap源码解析 59

# `libpcap/pcap-null.c`

这段代码是一个C语言的函数声明，它定义了一些名为“test”的函数。这些函数没有定义参数，并且没有返回类型。

函数声明中包含的星号（*）表示这个函数是一个多态函数，它可以接受不同类型的输入参数。多态函数有助于在编写代码时更少地写代码，同时也能使程序更易于维护和扩展。

具体的函数实现可能由其他代码片段完成，这些代码片段可以包含在使用这个函数的源文件中。


```cpp
/*
 * Copyright (c) 1994, 1995, 1996
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

这段代码是一个用于创建 PCAP 文件的设备配置头文件。它包含了一些定义和声明，以及一些函数，用于在 PCAP 文件中声明一些必要的头文件和变量。

首先，定义了一个名为 "#ifdef HAVE_CONFIG_H" 的预处理指令，如果该指令在编译时 evaluates为真，则包含 PCAP 配置头文件，否则不包含。

然后，定义了一个名为 "pcap_create_interface" 的函数，用于创建一个新的 PCAP 实例。该函数需要从用户提供的设备名称和 PCAP 配置头文件中提取出网络接口名称，并使用 pcap_strlcpy 函数将设备名称和 PCAP 配置头文件中的字符串复制到内存中，最后返回新的 PCAP 实例。

接下来，定义了一个名为 "nosup" 的静态字符数组，用于在 PCAP 配置头文件中声明一个默认的错误消息，该消息将使用 PCAP 的默认错误处理程序进行处理。

最后，定义了一个名为 "PCAP_INET_ADDRESSES" 的静态字符串，用于声明 PCAP 配置头文件中声明的 PCAP 网络接口地址数组。


```cpp
#ifdef HAVE_CONFIG_H
#include <config.h>
#endif

#include <string.h>

#include "pcap-int.h"

static char nosup[] = "live packet capture not supported on this system";

pcap_t *
pcap_create_interface(const char *device _U_, char *ebuf)
{
	(void)pcap_strlcpy(ebuf, nosup, PCAP_ERRBUF_SIZE);
	return (NULL);
}

```



这两段代码是用于网络协议栈中的pcap库中的函数。pcap是一个用于捕获和分析网络数据包的库，而这两段代码用于在平台找到可用的接口并在其中创建一个pcap设备链。

`int pcap_platform_finddevs(pcap_if_list_t *devlistp, char *errbuf)`函数用于返回可用于pcap设备的接口列表。如果没有可用的接口，该函数将返回0。然后将返回0作为errbuf的值，以便用户可以知道发生了什么。

`int pcap_lookupnet(const char *device, bpf_u_int32 *netp, bpf_u_int32 *maskp, char *errbuf)`函数用于在给定的设备上查找网络接口。它将返回一个负数，表示操作失败。如果成功，它将使用pcap库的`nosup`选项将errbuf填充为错误信息。然后将errbuf的值作为给定的netp和maskp的值，以便用户可以知道发生了什么。


```cpp
int
pcap_platform_finddevs(pcap_if_list_t *devlistp _U_, char *errbuf _U_)
{
	/*
	 * There are no interfaces on which we can capture.
	 */
	return (0);
}

#ifdef _WIN32
int
pcap_lookupnet(const char *device _U_, bpf_u_int32 *netp _U_,
    bpf_u_int32 *maskp _U_, char *errbuf)
{
	(void)pcap_strlcpy(errbuf, nosup, PCAP_ERRBUF_SIZE);
	return (-1);
}
```

这段代码是一个C语言代码，其中包括一个头文件和函数。

头文件名为“pcap_lib_version”，定义了一个名为“pcap_lib_version”的函数，函数返回值为“PCAP_VERSION_STRING”。

函数的作用是获取libpcap的版本字符串，然后将其存储在名为“pcap_lib_version”的变量中，并返回该变量的值。这个函数对于使用libpcap的程序来说可能是必要的，以便它们能够正确地使用对应的版本。


```cpp
#endif

/*
 * Libpcap version string.
 */
const char *
pcap_lib_version(void)
{
	return (PCAP_VERSION_STRING);
}

```

# `libpcap/pcap-pf.c`

这段代码定义了一个名为` packet_filter_subroutines.c`的函数，它是`tcpdump`软件中的一员，用于创建和提取TCP数据包。

具体来说，这个函数允许用户创建一个 packet_filter_subroutines 类型的函数指针，该函数指针将作为 `tcpdump` 命令行工具的一部分，用于创建和提取 TCP 数据包。它包含一个函数原型，定义了函数需要参数和返回值，以及允许的参数类型。


```cpp
/*
 * Copyright (c) 1990, 1991, 1992, 1993, 1994, 1995, 1996
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
 * packet filter subroutines for tcpdump
 *	Extraction/creation by Jeffrey Mogul, DECWRL
 */

```

这段代码包括一个 preprocessor 指令和一系列头文件。它的作用是检查系统是否支持配置文件，如果支持，则包含一些网络相关的头文件和函数。如果不支持，则不包含任何头文件。

具体来说，首先包含一个名为 "config.h" 的头文件，它可能定义了一些全局变量或者仅仅是一个简单的配置文件。接着包含一个名为 "sys/types.h" 的头文件，它可能定义了一些与系统类型相关的头文件。接着包含一个名为 "sys/time.h" 的头文件，它可能定义了一些与系统时间相关的头文件。包含一个名为 "sys/timeb.h" 的头文件，它可能定义了一些与系统时间相关的头文件。接着包含一个名为 "sys/socket.h" 的头文件，它可能定义了一些与系统套接字相关的头文件。包含一个名为 "sys/file.h" 的头文件，它可能定义了一些与系统文件相关的头文件。接着包含一个名为 "sys/ioctl.h" 的头文件，它可能定义了一些与系统 I/O 相关的头文件。

此外，接下来还包括一些与网络相关的头文件，如 "net/pfilt.h"。最后，在函数声明之前，通过 "include" 函数引入了一些其他头文件，如 "linux/if.h"。


```cpp
#ifdef HAVE_CONFIG_H
#include <config.h>
#endif

#include <sys/types.h>
#include <sys/time.h>
#include <sys/timeb.h>
#include <sys/socket.h>
#include <sys/file.h>
#include <sys/ioctl.h>
#include <net/pfilt.h>

struct mbuf;
struct rtentry;
#include <net/if.h>

```



该代码是一个网络头文件，它定义了一系列头文件，包括 `netinet/in.h`、`netinet/in_systm.h`、`netinet/ip.h`、`netinet/if_ether.h`、`netinet/ip_var.h`、`netinet/udp.h`、`netinet/udp_var.h`、`netinet/tcp.h` 和 `netinet/tcpip.h`。

这些头文件定义了网络中的各种数据结构和协议，包括 TCP 和 UDP 协议，以及 IPv4 和 IPv6 头。`include <netdb.h>` 和 `include <stdio.h>` 是在网络头文件中引入的函数，用于获取和设置 IP 地址和端口号。

具体来说，该代码的作用是定义了 `INET_LOOP` 函数。这个函数是一个指针，它指向一个 IP 地址和端口号的联合体。`include <netinet/in.h>` 和 `include <netinet/in_systm.h>` 允许函数使用 `INET_LOOP` 函数，`include <netinet/ip.h>` 和 `include <netinet/if_ether.h>` 允许函数使用 IPv4 或 IPv6 头。

然后，该代码定义了一系列函数，包括 `EXIT_FAILURE`、`INET_LOOP_INSECURE`、`INET_LOOP_CONNECT`、`DISCOVER_LOOPBACK_ADDRESS` 和 `DISCOVER_LOOPBACK_PORT`。这些函数会在 IP 地址和端口号上执行一些操作，包括设置 IP 地址、检测回环地址、设置连接参数、发现回环地址和端口号等。

最后，该代码还定义了一个名为 `main` 的函数，它接受一个参数 `argc` 和 `argv`。这个函数会在命令行上打印一些信息，包括发现到的回环地址和端口号。


```cpp
#include <netinet/in.h>
#include <netinet/in_systm.h>
#include <netinet/ip.h>
#include <netinet/if_ether.h>
#include <netinet/ip_var.h>
#include <netinet/udp.h>
#include <netinet/udp_var.h>
#include <netinet/tcp.h>
#include <netinet/tcpip.h>

#include <errno.h>
#include <netdb.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
```

这段代码的作用是定义了一个名为 `PCAP_DONT_INCLUDE_PCAP_BPF_H` 的宏，它通过 `#define` 告诉编译器不要包含 `pcap/bpf.h` 头文件。然后它引入了 `net/bpf.h` 头文件，这是 Linux 内核中的网络 BPF 接口，允许程序通过 BPF 实现网络流量控制等功能。接着它引入了 `pcap-int.h` 头文件，它定义了一些用于打印网络数据包信息的基本函数。最后，它检查了 `HAVE_OS_PROTO_H` 宏是否被定义，如果没有，它将引入 `os-proto.h` 头文件，它是一个用于打印操作系统系统消息的库头文件。


```cpp
#include <unistd.h>

/*
 * Make "pcap.h" not include "pcap/bpf.h"; we are going to include the
 * native OS version, as we need various BPF ioctls from it.
 */
#define PCAP_DONT_INCLUDE_PCAP_BPF_H
#include <net/bpf.h>

#include "pcap-int.h"

#ifdef HAVE_OS_PROTO_H
#include "os-proto.h"
#endif

```

这段代码定义了一个名为 `pcap_pf` 的结构体，用于表示pcap文件的过滤器。这个结构体包含以下几个私有成员：

- `filtering_in_kernel`：一个整数，表示是否在内核中进行过滤。
- `TotPkts`：一个 `u_long` 类型的整数，表示已经接受过滤的包总数。
- `TotAccepted`：一个 `u_long` 类型的整数，表示已经成功接受过滤的包总数。
- `TotDrops`：一个 `u_long` 类型的整数，表示已经掉落的包总数。
- `TotMissed`：一个 `u_long` 类型的整数，表示在过滤过程中未成功接受但已经传送的包的数量。
- `OrigMissed`：一个 `u_long` 类型的整数，表示在过滤之前未成功接受但已经传送的包的数量。

该代码中还定义了一个常量 `PCAP_FDDIPAD`，表示在输出到FDDI设备时对数据进行填充以使其在边界处对齐。


```cpp
/*
 * FDDI packets are padded to make everything line up on a nice boundary.
 */
#define       PCAP_FDDIPAD 3

/*
 * Private data for capturing on Ultrix and DEC OSF/1^WDigital UNIX^W^W
 * Tru64 UNIX packetfilter devices.
 */
struct pcap_pf {
	int	filtering_in_kernel; /* using kernel filter */
	u_long	TotPkts;	/* can't oflow for 79 hrs on ether */
	u_long	TotAccepted;	/* count accepted by filter */
	u_long	TotDrops;	/* count of dropped packets */
	long	TotMissed;	/* missed by i/f during this run */
	long	OrigMissed;	/* missed by i/f before this run */
};

```

This is a function definition for `ip_filter_push`. This function appears to be part of the Linux kernel's network stack, and it is used to enforce rules on packets that are allowed to be processed by the kernel's networking subsystem.

The function takes in a `PC` structure that contains information about the current packet and its input. The structure contains fields such as `pc` (which contains information about the current packet), `sp` (which contains information about the packet that is being processed), and `en` (which contains information about the environment in which the packet is being processed).

The function then performs several checks and transformations on the packet, such as calculating the number of bytes that have been processed by the kernel's networking subsystem so far (`inc`), correctly accounting for the number of packets that have been processed by the kernel so far (`n`), and updating the `TotPkts`, `TotDrops`, and `TotMissed` fields in the `PC` structure.

Finally, the function checks whether the packet passed through the kernel's networking subsystem and, if it did, passes it to the callback function for further processing. If the packet did not pass through the kernel's networking subsystem, the function returns 0 to indicate that the packet should not have been processed.


```cpp
static int pcap_setfilter_pf(pcap_t *, struct bpf_program *);

/*
 * BUFSPACE is the size in bytes of the packet read buffer.  Most tcpdump
 * applications aren't going to need more than 200 bytes of packet header
 * and the read shouldn't return more packets than packetfilter's internal
 * queue limit (bounded at 256).
 */
#define BUFSPACE (200 * 256)

static int
pcap_read_pf(pcap_t *pc, int cnt, pcap_handler callback, u_char *user)
{
	struct pcap_pf *pf = pc->priv;
	register u_char *p, *bp;
	register int cc, n, buflen, inc;
	register struct enstamp *sp;
	struct enstamp stamp;
	register u_int pad;

 again:
	cc = pc->cc;
	if (cc == 0) {
		cc = read(pc->fd, (char *)pc->buffer + pc->offset, pc->bufsize);
		if (cc < 0) {
			if (errno == EWOULDBLOCK)
				return (0);
			if (errno == EINVAL &&
			    lseek(pc->fd, 0L, SEEK_CUR) + pc->bufsize < 0) {
				/*
				 * Due to a kernel bug, after 2^31 bytes,
				 * the kernel file offset overflows and
				 * read fails with EINVAL. The lseek()
				 * to 0 will fix things.
				 */
				(void)lseek(pc->fd, 0L, SEEK_SET);
				goto again;
			}
			pcap_fmt_errmsg_for_errno(pc->errbuf,
			    sizeof(pc->errbuf), errno, "pf read");
			return (-1);
		}
		bp = (u_char *)pc->buffer + pc->offset;
	} else
		bp = pc->bp;
	/*
	 * Loop through each packet.
	 *
	 * This assumes that a single buffer of packets will have
	 * <= INT_MAX packets, so the packet count doesn't overflow.
	 */
	n = 0;
	pad = pc->fddipad;
	while (cc > 0) {
		/*
		 * Has "pcap_breakloop()" been called?
		 * If so, return immediately - if we haven't read any
		 * packets, clear the flag and return -2 to indicate
		 * that we were told to break out of the loop, otherwise
		 * leave the flag set, so that the *next* call will break
		 * out of the loop without having read any packets, and
		 * return the number of packets we've processed so far.
		 */
		if (pc->break_loop) {
			if (n == 0) {
				pc->break_loop = 0;
				return (-2);
			} else {
				pc->cc = cc;
				pc->bp = bp;
				return (n);
			}
		}
		if (cc < sizeof(*sp)) {
			snprintf(pc->errbuf, sizeof(pc->errbuf),
			    "pf short read (%d)", cc);
			return (-1);
		}
		if ((long)bp & 3) {
			sp = &stamp;
			memcpy((char *)sp, (char *)bp, sizeof(*sp));
		} else
			sp = (struct enstamp *)bp;
		if (sp->ens_stamplen != sizeof(*sp)) {
			snprintf(pc->errbuf, sizeof(pc->errbuf),
			    "pf short stamplen (%d)",
			    sp->ens_stamplen);
			return (-1);
		}

		p = bp + sp->ens_stamplen;
		buflen = sp->ens_count;
		if (buflen > pc->snapshot)
			buflen = pc->snapshot;

		/* Calculate inc before possible pad update */
		inc = ENALIGN(buflen + sp->ens_stamplen);
		cc -= inc;
		bp += inc;
		pf->TotPkts++;
		pf->TotDrops += sp->ens_dropped;
		pf->TotMissed = sp->ens_ifoverflows;
		if (pf->OrigMissed < 0)
			pf->OrigMissed = pf->TotMissed;

		/*
		 * Short-circuit evaluation: if using BPF filter
		 * in kernel, no need to do it now - we already know
		 * the packet passed the filter.
		 *
		 * Note: the filter code was generated assuming
		 * that pc->fddipad was the amount of padding
		 * before the header, as that's what's required
		 * in the kernel, so we run the filter before
		 * skipping that padding.
		 */
		if (pf->filtering_in_kernel ||
		    pcap_filter(pc->fcode.bf_insns, p, sp->ens_count, buflen)) {
			struct pcap_pkthdr h;
			pf->TotAccepted++;
			h.ts = sp->ens_tstamp;
			h.len = sp->ens_count - pad;
			p += pad;
			buflen -= pad;
			h.caplen = buflen;
			(*callback)(user, &h, p);
			if (++n >= cnt && !PACKET_COUNT_IS_UNLIMITED(cnt)) {
				pc->cc = cc;
				pc->bp = bp;
				return (n);
			}
		}
	}
	pc->cc = 0;
	return (n);
}

```

这段代码是一个用于 `pcap_inject_pf` 函数，其作用是将 `buf` 字节码中的数据发送到 `pcap_t` 结构中的 `p` 指针所表示的设备文件中，并返回成功的返回值。

具体来说，代码中首先定义了一个名为 `pcap_inject_pf` 的函数，它接受两个参数：一个指向 `pcap_t` 结构对象的指针 `p`，以及一个指向 `buf` 字节码的指针 `buf` 和一个表示 `buf` 字节码的大小的整数 `size`。

函数实现中，首先调用 `write` 函数将 `buf` 字节码中的数据发送到 `pcap_t` 结构中的 `p` 指针所表示的设备文件中，这个函数的参数是 `p` 指针和 `size` 大小。然后，代码使用 `if` 语句检查 `write` 函数的返回值是否为成功，如果是，就执行下面的代码块。否则，首先从 `pcap_t` 结构中的错误变量 `errbuf` 中获取错误信息，然后使用 `errno` 获取错误码，并输出错误信息。最后，返回负一，表示函数失败。

如果 `write` 函数执行成功，函数将返回 0。


```cpp
static int
pcap_inject_pf(pcap_t *p, const void *buf, int size)
{
	int ret;

	ret = write(p->fd, buf, size);
	if (ret == -1) {
		pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
		    errno, "send");
		return (-1);
	}
	return (ret);
}

static int
```

This function appears to be part of the `pcap` packet capture framework in C. It appears to be handling the counting of packets received from the network, as well as the counting of packets dropped by the input queue (either by the kernel or by the application).

The function takes two parameters: `p` and `ps`. `p` is a pointer to a `cap_t` structure containing information about the packet capture process, while `ps` is a pointer to a `struct pcap_stat` structure containing statistical information about the packets received and processed by thecapture process.

The function first sets the value of `ps_recv` to the total number of packets received and processed by the application, `ps_drop` to the total number of packets dropped by the input queue, and `ps_ifdrop` to the total number of packets missed by the input queue.

It then sets the value of `ps_recv` to the total number of packets received and processed by the kernel, `ps_drop` to the total number of packets dropped by the kernel, and `ps_ifdrop` to zero.

It appears that the function is intended to provide a balanced view of the number of packets received and processed by the application versus those dropped by the input queue, regardless of whether the packets were allowed to pass through the kernel filter or not.


```cpp
pcap_stats_pf(pcap_t *p, struct pcap_stat *ps)
{
	struct pcap_pf *pf = p->priv;

	/*
	 * If packet filtering is being done in the kernel:
	 *
	 *	"ps_recv" counts only packets that passed the filter.
	 *	This does not include packets dropped because we
	 *	ran out of buffer space.  (XXX - perhaps it should,
	 *	by adding "ps_drop" to "ps_recv", for compatibility
	 *	with some other platforms.  On the other hand, on
	 *	some platforms "ps_recv" counts only packets that
	 *	passed the filter, and on others it counts packets
	 *	that didn't pass the filter....)
	 *
	 *	"ps_drop" counts packets that passed the kernel filter
	 *	(if any) but were dropped because the input queue was
	 *	full.
	 *
	 *	"ps_ifdrop" counts packets dropped by the network
	 *	interface (regardless of whether they would have passed
	 *	the input filter, of course).
	 *
	 * If packet filtering is not being done in the kernel:
	 *
	 *	"ps_recv" counts only packets that passed the filter.
	 *
	 *	"ps_drop" counts packets that were dropped because the
	 *	input queue was full, regardless of whether they passed
	 *	the userland filter.
	 *
	 *	"ps_ifdrop" counts packets dropped by the network
	 *	interface (regardless of whether they would have passed
	 *	the input filter, of course).
	 *
	 * These statistics don't include packets not yet read from
	 * the kernel by libpcap, but they may include packets not
	 * yet read from libpcap by the application.
	 */
	ps->ps_recv = pf->TotAccepted;
	ps->ps_drop = pf->TotDrops;
	ps->ps_ifdrop = pf->TotMissed - pf->OrigMissed;
	return (0);
}

```

DLT_DOCSIS
========

The above code snippet defines a constant called `DLT_DOCSIS` which is used to define the index of a PCAP implementation.


```cpp
/*
 * We include the OS's <net/bpf.h>, not our "pcap/bpf.h", so we probably
 * don't get DLT_DOCSIS defined.
 */
#ifndef DLT_DOCSIS
#define DLT_DOCSIS	143
#endif

static int
pcap_activate_pf(pcap_t *p)
{
	struct pcap_pf *pf = p->priv;
	short enmode;
	int backlog = -1;	/* request the most */
	struct enfilter Filter;
	struct endevp devparams;
	int err;

	/*
	 * Initially try a read/write open (to allow the inject
	 * method to work).  If that fails due to permission
	 * issues, fall back to read-only.  This allows a
	 * non-root user to be granted specific access to pcap
	 * capabilities via file permissions.
	 *
	 * XXX - we should have an API that has a flag that
	 * controls whether to open read-only or read-write,
	 * so that denial of permission to send (or inability
	 * to send, if sending packets isn't supported on
	 * the device in question) can be indicated at open
	 * time.
	 *
	 * XXX - we assume here that "pfopen()" does not, in fact, modify
	 * its argument, even though it takes a "char *" rather than a
	 * "const char *" as its first argument.  That appears to be
	 * the case, at least on Digital UNIX 4.0.
	 *
	 * XXX - is there an error that means "no such device"?  Is
	 * there one that means "that device doesn't support pf"?
	 */
	p->fd = pfopen(p->opt.device, O_RDWR);
	if (p->fd == -1 && errno == EACCES)
		p->fd = pfopen(p->opt.device, O_RDONLY);
	if (p->fd < 0) {
		if (errno == EACCES) {
			snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
			    "pf open: %s: Permission denied\n"
```

这段代码是 perl 语言中的一个函数，其作用是处理 packetfilter 函数的错误。函数的输入参数是一个名为 p 的 PCAP 结构体，该结构体包含了网络数据包过滤器的一些选项，如 device、action、errno 等。

函数首先检查设备是否已配置正确，如果设备不正确，函数将返回 PCAP_ERROR_PERM_DENIED，然后尝试使用 errno 为 4（PCAP_ERRNO_MAX_ACTION）的函数 pcap_fmt_errno_for_errno 打印错误信息，并返回 PCAP_ERROR。如果仍然正确，函数将尝试将 snapshot 值设置为一个更大的值（不超过 MAXIMUM_SNAPLEN），然后尝试调用 ioctl 函数 0x8 (enmode) 设置 PCAP 结构体中的 enmode。如果该函数调用失败，函数将返回 PCAP_ERROR，并从 bad 标签处跳转到错误处理程序。

最后，函数还处理了原始数据包 missed 属性的设置，将其设置为 -1。


```cpp
"your system may not be properly configured; see the packetfilter(4) man page",
			    p->opt.device);
			err = PCAP_ERROR_PERM_DENIED;
		} else {
			pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
			    errno, "pf open: %s", p->opt.device);
			err = PCAP_ERROR;
		}
		goto bad;
	}

	/*
	 * Turn a negative snapshot value (invalid), a snapshot value of
	 * 0 (unspecified), or a value bigger than the normal maximum
	 * value, into the maximum allowed value.
	 *
	 * If some application really *needs* a bigger snapshot
	 * length, we should just increase MAXIMUM_SNAPLEN.
	 */
	if (p->snapshot <= 0 || p->snapshot > MAXIMUM_SNAPLEN)
		p->snapshot = MAXIMUM_SNAPLEN;

	pf->OrigMissed = -1;
	enmode = ENTSTAMP|ENNONEXCL;
	if (!p->opt.immediate)
		enmode |= ENBATCH;
	if (p->opt.promisc)
		enmode |= ENPROMISC;
	if (ioctl(p->fd, EIOCMBIS, (caddr_t)&enmode) < 0) {
		pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
		    errno, "EIOCMBIS");
		err = PCAP_ERROR;
		goto bad;
	}
```

这段代码的作用是设置网络接口设备（如以太网卡）的参数，以便能够自定义接收数据包。以下是代码的要点解释：

1. 首先通过#ifdef预处理指令，检查当前编译环境是否支持`ENCOPYALL`模式。如果不支持，则执行尝试设置该模式以及设置缓冲区描述符的代码。

2. 如果设置缓冲区描述符的尝试失败，则执行设置后端缓冲区的代码。这里只是简单的输出一个错误信息，并不会对系统产生任何影响。

3. 如果设置缓冲区描述符成功，则执行设置后端缓冲区的代码。

4. 接下来，使用ioctl函数设置网络接口设备的参数。如果设置失败，将打印错误信息并返回错误码。这里使用的是PCAP_ERROR默认错误码，会在错误信息中提供错误详细信息。

5. 最后，使用to_ascii函数将获得的网络接口类型字符串打印到errno变量中，以便于使用调试工具进行调试。


```cpp
#ifdef	ENCOPYALL
	/* Try to set COPYALL mode so that we see packets to ourself */
	enmode = ENCOPYALL;
	(void)ioctl(p->fd, EIOCMBIS, (caddr_t)&enmode);/* OK if this fails */
#endif
	/* set the backlog */
	if (ioctl(p->fd, EIOCSETW, (caddr_t)&backlog) < 0) {
		pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
		    errno, "EIOCSETW");
		err = PCAP_ERROR;
		goto bad;
	}
	/* discover interface type */
	if (ioctl(p->fd, EIOCDEVP, (caddr_t)&devparams) < 0) {
		pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
		    errno, "EIOCDEVP");
		err = PCAP_ERROR;
		goto bad;
	}
	/* HACK: to compile prior to Ultrix 4.2 */
```

这段代码定义了一个名为ENDT_FDDI的常量，其值为4。接下来，定义了一个switch语句，根据devparams.end_dev_type变量来选择DLT_EN10MB或DLT_FDDI。

如果是ENDT_10MB，则定义p->linktype为DLT_EN10MB,p->offset为2，并且如果没有分配内存空间成功，则p->dlt_list将保持为空。然后定义p->dlt_list包含两个成员，一个是DLT_EN10MB，另一个是DLT_DOCSIS,p->dlt_count设置为2。这样，应用程序就可以根据需要选择10MB或FDDI链路类型。

如果是ENDT_FDDI，则定义p->linktype为DLT_FDDI，即直接使用FDDI作为链路类型。


```cpp
#ifndef	ENDT_FDDI
#define	ENDT_FDDI	4
#endif
	switch (devparams.end_dev_type) {

	case ENDT_10MB:
		p->linktype = DLT_EN10MB;
		p->offset = 2;
		/*
		 * This is (presumably) a real Ethernet capture; give it a
		 * link-layer-type list with DLT_EN10MB and DLT_DOCSIS, so
		 * that an application can let you choose it, in case you're
		 * capturing DOCSIS traffic that a Cisco Cable Modem
		 * Termination System is putting out onto an Ethernet (it
		 * doesn't put an Ethernet header onto the wire, it puts raw
		 * DOCSIS frames out on the wire inside the low-level
		 * Ethernet framing).
		 */
		p->dlt_list = (u_int *) malloc(sizeof(u_int) * 2);
		/*
		 * If that fails, just leave the list empty.
		 */
		if (p->dlt_list != NULL) {
			p->dlt_list[0] = DLT_EN10MB;
			p->dlt_list[1] = DLT_DOCSIS;
			p->dlt_count = 2;
		}
		break;

	case ENDT_FDDI:
		p->linktype = DLT_FDDI;
		break;

```

这段代码是一个条件编译语句，用于根据不同的条件选择不同的链接类型。

具体来说，当条件为#ifdef ENDT_SLIP时，代码会执行其中的case语句，即p->linktype = DLT_SLIP;，然后跳出整个条件语句。

当条件为#ifdef ENDT_PP的时候，代码会执行其中的case语句，即p->linktype = DLT_PP;，然后跳出整个条件语句。

当条件为#ifdef ENDT_LOOPBACK的时候，代码会执行其中的case语句，即p->linktype = DLT_EN10MB;，然后设置offset为2，并跳出整个条件语句。不过，这个分支并不影响前面的判断，因此仍然会执行p->linktype = DLT_SLIP;。

总之，这段代码的作用是选择适当的链接类型，并在需要时设置其偏移量。


```cpp
#ifdef ENDT_SLIP
	case ENDT_SLIP:
		p->linktype = DLT_SLIP;
		break;
#endif

#ifdef ENDT_PPP
	case ENDT_PPP:
		p->linktype = DLT_PPP;
		break;
#endif

#ifdef ENDT_LOOPBACK
	case ENDT_LOOPBACK:
		/*
		 * It appears to use Ethernet framing, at least on
		 * Digital UNIX 4.0.
		 */
		p->linktype = DLT_EN10MB;
		p->offset = 2;
		break;
```

This is a function definition for the `pcap_t` packet filter device driver in C. It appears to handle the configuration and initialization of the device, as well as the management of its associated file descriptor (such as the network socket).

The function first checks if the device has an associated timeout in its configuration options. If it does, it sets up a timeout for the `select()` and `poll()` system calls, which are used for "selecting" and "polling" packets.

The function then configures the `pcap_read_pf`, `pcap_inject_pf`, and `pcap_setfilter_pf` system calls for reading, injecting, and filtering packets, respectively. It also sets the `pcap_fd` system call to point to the file descriptor associated with the device.

Finally, the function sets up a function pointer to `pcap_stats_pf`, which is used to calculate and print statistics about the packets processed by the device.

Note that the `PCAP_ERROR` error code is generated if the device configuration fails (e.g. if the `malloc()` call fails).


```cpp
#endif

#ifdef ENDT_TRN
	case ENDT_TRN:
		p->linktype = DLT_IEEE802;
		break;
#endif

	default:
		/*
		 * XXX - what about ENDT_IEEE802?  The pfilt.h header
		 * file calls this "IEEE 802 networks (non-Ethernet)",
		 * but that doesn't specify a specific link layer type;
		 * it could be 802.4, or 802.5 (except that 802.5 is
		 * ENDT_TRN), or 802.6, or 802.11, or....  That's why
		 * DLT_IEEE802 was hijacked to mean Token Ring in various
		 * BSDs, and why we went along with that hijacking.
		 *
		 * XXX - what about ENDT_HDLC and ENDT_NULL?
		 * Presumably, as ENDT_OTHER is just "Miscellaneous
		 * framing", there's not much we can do, as that
		 * doesn't specify a particular type of header.
		 */
		snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
		    "unknown data-link type %u", devparams.end_dev_type);
		err = PCAP_ERROR;
		goto bad;
	}
	/* set truncation */
	if (p->linktype == DLT_FDDI) {
		p->fddipad = PCAP_FDDIPAD;

		/* packetfilter includes the padding in the snapshot */
		p->snapshot += PCAP_FDDIPAD;
	} else
		p->fddipad = 0;
	if (ioctl(p->fd, EIOCTRUNCATE, (caddr_t)&p->snapshot) < 0) {
		pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
		    errno, "EIOCTRUNCATE");
		err = PCAP_ERROR;
		goto bad;
	}
	/* accept all packets */
	memset(&Filter, 0, sizeof(Filter));
	Filter.enf_Priority = 37;	/* anything > 2 */
	Filter.enf_FilterLen = 0;	/* means "always true" */
	if (ioctl(p->fd, EIOCSETF, (caddr_t)&Filter) < 0) {
		pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
		    errno, "EIOCSETF");
		err = PCAP_ERROR;
		goto bad;
	}

	if (p->opt.timeout != 0) {
		struct timeval timeout;
		timeout.tv_sec = p->opt.timeout / 1000;
		timeout.tv_usec = (p->opt.timeout * 1000) % 1000000;
		if (ioctl(p->fd, EIOCSRTIMEOUT, (caddr_t)&timeout) < 0) {
			pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
			    errno, "EIOCSRTIMEOUT");
			err = PCAP_ERROR;
			goto bad;
		}
	}

	p->bufsize = BUFSPACE;
	p->buffer = malloc(p->bufsize + p->offset);
	if (p->buffer == NULL) {
		pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
		    errno, "malloc");
		err = PCAP_ERROR;
		goto bad;
	}

	/*
	 * "select()" and "poll()" work on packetfilter devices.
	 */
	p->selectable_fd = p->fd;

	p->read_op = pcap_read_pf;
	p->inject_op = pcap_inject_pf;
	p->setfilter_op = pcap_setfilter_pf;
	p->setdirection_op = NULL;	/* Not implemented. */
	p->set_datalink_op = NULL;	/* can't change data link type */
	p->getnonblock_op = pcap_getnonblock_fd;
	p->setnonblock_op = pcap_setnonblock_fd;
	p->stats_op = pcap_stats_pf;

	return (0);
 bad:
	pcap_cleanup_live_common(p);
	return (err);
}

```

这段代码定义了一个名为pcap_t的指针变量，其作用是用于管理PCAP数据包抓取程序的创建和配置。

pcap_create_interface函数用于创建一个PCAP数据包抓取程序，该函数的第一个参数是一个字符串，表示要使用的网络接口的名称。第二个参数是一个字符数组，用于存储该接口的电子数据包缓冲区，用于存储从网络接口捕获的数据包。

函数首先创建一个名为p的PCAP数据包抓取程序句柄，然后设置activate_op为pcap_activate_pf函数的地址，以便在调用pcap_activate_pf函数时自动设置pf协议的配置。

最后，函数返回一个指向pcap数据包抓取程序句柄的指针，如果函数成功创建并返回，否则返回一个指向 NULL的指针。


```cpp
pcap_t *
pcap_create_interface(const char *device _U_, char *ebuf)
{
	pcap_t *p;

	p = PCAP_CREATE_COMMON(ebuf, struct pcap_pf);
	if (p == NULL)
		return (NULL);

	p->activate_op = pcap_activate_pf;
	return (p);
}

/*
 * XXX - is there an error from pfopen() that means "no such device"?
 * Is there one that means "that device doesn't support pf"?
 */
```

这段代码定义了两个静态函数：can_be_bound() 和 get_if_flags()。

can_be_bound()函数接受一个字符串参数name，并返回一个整数。函数的作用是判断给定的字符串是否为循环设备（如无线充电器、路由器等）。如果字符串是一个循环设备的名称，函数将返回0；否则，函数将返回1。

get_if_flags()函数接受一个字符串参数name和一个整数参数flags，并返回一个字符数组errbuf。函数的作用是获取给定字符串的if flags，并根据if flags的值判断给定设备是否支持循环回程。如果if flags中包含PCAP_IF_LOOPBACK，则表示该设备支持循环回程，函数将返回0；否则，函数将返回一个指向字符数组的指针errbuf，其中errbuf包含一个错误消息。


```cpp
static int
can_be_bound(const char *name _U_)
{
	return (1);
}

static int
get_if_flags(const char *name _U_, bpf_u_int32 *flags _U_, char *errbuf _U_)
{
	/*
	 * Nothing we can do other than mark loopback devices as "the
	 * connected/disconnected status doesn't apply".
	 *
	 * XXX - is there a way to find out whether an adapter has
	 * something plugged into it?
	 */
	if (*flags & PCAP_IF_LOOPBACK) {
		/*
		 * Loopback devices aren't wireless, and "connected"/
		 * "disconnected" doesn't apply to them.
		 */
		*flags |= PCAP_IF_CONNECTION_STATUS_NOT_APPLICABLE;
		return (0);
	}
	return (0);
}

```

This is a function that implements theBPF (Bare-Metal Filter) mechanism for filtering TCP traffic using a kernel-level BPF filter. TheBPF filter allows you to configure the kernel to only process packets that match a given filter, which can improve the performance of your application by reducing the amount of traffic that needs to be processed by the kernel.

The function first sets up a BPF filter using the `BPF_CONFIG()` function, but it doesn't actually install it into the kernel just yet. It then sets the `filtering_in_kernel` flag to 1 to indicate that the filter is currently installed in the kernel, and returns 0 to indicate that the installation was successful.

If the installation was successful, the function then discards any previously-received packets and sets the `cc` flag to 0. This will cause the filter to be applied by the application, but the filter won't be installed in the kernel yet.

If the installation was not successful, the function logs a message and returns -1.

Once the filter is installed in the kernel, the function returns 0 to indicate that the installation was successful.

The function also logs a message indicating that the filter is being applied in the user process, rather than the kernel process.

Note: This function should be used in conjunction with the `tcpdump` command, as it will only work with certain versions of the `tcpdump` tool.


```cpp
int
pcap_platform_finddevs(pcap_if_list_t *devlistp, char *errbuf)
{
	return (pcap_findalldevs_interfaces(devlistp, errbuf, can_be_bound,
	    get_if_flags));
}

static int
pcap_setfilter_pf(pcap_t *p, struct bpf_program *fp)
{
	struct pcap_pf *pf = p->priv;
	struct bpf_version bv;

	/*
	 * See if BIOCVERSION works.  If not, we assume the kernel doesn't
	 * support BPF-style filters (it's not documented in the bpf(7)
	 * or packetfiler(7) man pages, but the code used to fail if
	 * BIOCSETF worked but BIOCVERSION didn't, and I've seen it do
	 * kernel filtering in DU 4.0, so presumably BIOCVERSION works
	 * there, at least).
	 */
	if (ioctl(p->fd, BIOCVERSION, (caddr_t)&bv) >= 0) {
		/*
		 * OK, we have the version of the BPF interpreter;
		 * is it the same major version as us, and the same
		 * or better minor version?
		 */
		if (bv.bv_major == BPF_MAJOR_VERSION &&
		    bv.bv_minor >= BPF_MINOR_VERSION) {
			/*
			 * Yes.  Try to install the filter.
			 */
			if (ioctl(p->fd, BIOCSETF, (caddr_t)fp) < 0) {
				pcap_fmt_errmsg_for_errno(p->errbuf,
				    sizeof(p->errbuf), errno, "BIOCSETF");
				return (-1);
			}

			/*
			 * OK, that succeeded.  We're doing filtering in
			 * the kernel.  (We assume we don't have a
			 * userland filter installed - that'd require
			 * a previous version check to have failed but
			 * this one to succeed.)
			 *
			 * XXX - this message should be supplied to the
			 * application as a warning of some sort,
			 * except that if it's a GUI application, it's
			 * not clear that it should be displayed in
			 * a window to annoy the user.
			 */
			fprintf(stderr, "tcpdump: Using kernel BPF filter\n");
			pf->filtering_in_kernel = 1;

			/*
			 * Discard any previously-received packets,
			 * as they might have passed whatever filter
			 * was formerly in effect, but might not pass
			 * this filter (BIOCSETF discards packets buffered
			 * in the kernel, so you can lose packets in any
			 * case).
			 */
			p->cc = 0;
			return (0);
		}

		/*
		 * We can't use the kernel's BPF interpreter; don't give
		 * up, just log a message and be inefficient.
		 *
		 * XXX - this should really be supplied to the application
		 * as a warning of some sort.
		 */
		fprintf(stderr,
	    "tcpdump: Requires BPF language %d.%d or higher; kernel is %d.%d\n",
		    BPF_MAJOR_VERSION, BPF_MINOR_VERSION,
		    bv.bv_major, bv.bv_minor);
	}

	/*
	 * We couldn't do filtering in the kernel; do it in userland.
	 */
	if (install_bpf_program(p, fp) < 0)
		return (-1);

	/*
	 * XXX - this message should be supplied by the application as
	 * a warning of some sort.
	 */
	fprintf(stderr, "tcpdump: Filtering in user process\n");
	pf->filtering_in_kernel = 0;
	return (0);
}

```

这段代码定义了一个名为`pcap_lib_version`的函数，它的作用是返回一个指向`PCAP_VERSION_STRING`字符串的指针。

在函数体内部，没有对`PCAP_VERSION_STRING`进行实际操作，它只是一个字符串常量，其值为`"0.9.4"`。因此，这段代码的作用是返回一个与`PCAP_VERSION_STRING`相关的字符串，以便其他函数或程序使用。


```cpp
/*
 * Libpcap version string.
 */
const char *
pcap_lib_version(void)
{
	return (PCAP_VERSION_STRING);
}

```

# `libpcap/pcap-rdmasniff.c`

这段代码是一个C语言的函数声明，它定义了一个名为"compute_mean"的函数。函数没有参数并且返回一个double类型的值，它的实现将返回一个算术平均值。

在这段注释中，说明了这段代码的版权、许可证和使用限制。开发者允许对这段代码进行修改或分发，但必须遵守以下两个条件：

1. 对源代码的分发必须包含上述版权声明、此列表条件以及一个说明，说明这段代码可以如何使用，如何得到授权，以及如何宣传或推荐使用此代码；
2. 对二进制形式的分发必须包含上述版权声明、此列表条件以及一个说明，说明这段代码在二进制形式下的使用。开发者保留在分发中使用此代码的权利，前提是必须遵守上述限制。

最后，作者声明在任何情况下，这段代码中的错误或缺陷不会影响到任何人，包括在作者或贡献者的情况下。


```cpp
/*
 * Copyright (c) 2017 Pure Storage, Inc.
 * All rights reserved.
 *
 * Redistribution and use in source and binary forms, with or without
 * modification, are permitted provided that the following conditions
 * are met:
 *
 * 1. Redistributions of source code must retain the above copyright
 * notice, this list of conditions and the following disclaimer.
 * 2. Redistributions in binary form must reproduce the above copyright
 * notice, this list of conditions and the following disclaimer in the
 * documentation and/or other materials provided with the distribution.
 * 3. The name of the author may not be used to endorse or promote
 * products derived from this software without specific prior written
 * permission.
 *
 * THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS
 * "AS IS" AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT
 * LIMITED TO, THE IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR
 * A PARTICULAR PURPOSE ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT
 * OWNER OR CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL,
 * SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT
 * LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS OF USE,
 * DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY
 * THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT
 * (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE
 * OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.
 */

```

这段代码是一个C语言程序，它包含一个头文件和两个函数指针变量。我们需要逐步解释这段代码的作用，以便更好地理解它的功能。

1. 首先，这是一个包含宏定义的头文件，它的名称为"config.h"。这里我们无法直接查看该文件的内容，因此无法了解它具体定义了哪些宏。

2. 接下来，定义了一个名为"ibv_flow_attr_sniffer"的函数指针。这是一个全局函数，因为它没有被定义为静态函数或私有函数。

3. 在函数指针"ibv_flow_attr_sniffer"后面，又包含了一个包含多个头文件的引用。这些头文件都与网络数据包处理和分析相关，具体我们无法从代码中得知。

4. 在头文件"pcap-int.h"中，可能包含IBV（Intel钦定版）协议栈的相关函数和宏。

5. 在头文件"pcap-rdmasniff.h"中，可能包含用于处理RDMA（Remote Direct Memory Access，远程直接内存访问）的相关函数和宏。

6. 接下来，定义了一个名为"test_example"的函数。但我们无法得知这个函数的实现，因为它的定义在"ibv_flow_attr_sniffer"内部。

7. 在函数"test_example"后面，又包含了一个包含多个头文件的引用。这些头文件都与测试和调试相关，具体我们无法从代码中得知。

8. 在函数"test_example"内部，定义了一个整型变量"i"。但它的具体值和作用在代码中并未明确说明。

9. 在函数"test_example"内部，定义了一个名为"tcp_socket"的函数指针。这个函数指针可能用于创建和管理TCP套接字，具体我们无法从代码中得知。

10. 在函数"test_example"内部，定义了一个名为"send_packet"的函数。这个函数可能用于发送数据包，具体我们无法从代码中得知。

11. 在函数"send_packet"内部，又包含了一个名为"icmp_option"的函数指针。这个函数指针可能用于设置IPICMP（Internet Control Message Protocol，Internet报文控制协议）选项，具体我们无法从代码中得知。

12. 在函数"send_packet"内部，定义了一个名为"packet"的结构体。这个结构体可能包含用于封装数据包的信息，具体我们无法从代码中得知。

13. 在函数"send_packet"内部，定义了一个名为"parse_options"的函数。这个函数可能用于解析接收到的数据包选项，具体我们无法从代码中得知。

14. 在函数"send_packet"内部，定义了一个名为"send_packet64"的函数。这个函数可能用于发送支持IPv6的IPv6数据包，具体我们无法从代码中得知。

15. 在函数"send_packet64"内部，又包含了一个名为"get_current_time"的函数。这个函数可能用于获取当前时间戳，具体我们无法从代码中得知。

16. 在函数"send_packet64"内部，定义了一个名为"init_socket"的函数。这个函数可能用于初始化TCP套接字，具体我们无法从代码中得知。

17. 在函数"init_socket"内部，又包含了一个名为"set_options"的函数。这个函数可能用于设置TCP套接字的选项，具体我们无法从代码中得知。

18. 在函数"init_socket"内部，又包含了一个名为"set_current_time"的函数。这个函数可能用于设置TCP套接字的时间戳，具体我们无法从代码中得知。

19. 在函数"init_socket"内部，又包含一个名为"send_sendall"的函数。这个函数可能用于发送数据到指定端口，具体我们无法从代码中得知。

20. 在函数"send_sendall"内部，又包含一个名为"send_option"的函数。这个函数可能用于设置IP选项，具体我们无法从代码中得知。

21. 在函数"send_option"内部，又包含一个名为"send_receive"的函数。这个函数可能用于在发送数据


```cpp
#ifdef HAVE_CONFIG_H
#include "config.h"
#endif

#include "pcap-int.h"
#include "pcap-rdmasniff.h"

#include <infiniband/verbs.h>
#include <stdlib.h>
#include <string.h>
#include <limits.h> /* for INT_MAX */
#include <sys/time.h>

#if !defined(IBV_FLOW_ATTR_SNIFFER)
#define IBV_FLOW_ATTR_SNIFFER	3
```

这段代码定义了一个名为 `struct pcap_rdmasniff` 的结构体，用于表示以太网数据包接收者（RDMASNIFF）的相关信息。它用于接收无线网络中的数据包，并将其存储在内存中或通过管道发送到接收者（RDMASNIFF）进行进一步处理。以下是结构体中定义的一些常量：

```cpp
static const int RDMASNIFF_NUM_RECEIVES = 128;
static const int RDMASNIFF_RECEIVE_SIZE = 10000;
```

其中，`RDMASNIFF_NUM_RECEIVES` 是一个整数常量，表示每个接收者（RDMASNIFF）可以接收的最大数据包数量。`RDMASNIFF_RECEIVE_SIZE` 是一个整数常量，表示每个数据包的大小，单位为字节。

```cpp
struct pcap_rdmasniff {
	struct ibv_device *		rdma_device;
	struct ibv_context *		context;
	struct ibv_comp_channel *	channel;
	struct ibv_pd *			pd;
	struct ibv_cq *			cq;
	struct ibv_qp *			qp;
	struct ibv_flow *               flow;
	struct ibv_mr *			mr;
	u_char *			oneshot_buffer;
	unsigned long			port_num;
	int                             cq_event;
	u_int                           packets_recv;
};
```

`struct ibv_device *` 指向一个 `struct ibv_device` 类型的结构体，用于表示接收者（RDMASNIFF）所使用的网络接口。`struct ibv_context *` 指向一个 `struct ibv_context` 类型的结构体，用于表示当前网络接口的状态信息。`struct ibv_comp_channel *` 和 `struct ibv_qp` 类似，用于表示信道的相关信息。`struct ibv_flow` 和 `struct ibv_mr` 则用于表示数据包的传输和接收相关信息。

此外，还包括一些用于接收数据包的指针和变量：

```cpp
	u_char *			oneshot_buffer;
	unsigned long			port_num;
	int                             cq_event;
	u_int                           packets_recv;
```

`oneshot_buffer` 是一个无缓冲区字符数组，用于保存接收到的数据包，每个数据包被封装成一个字符数组。`port_num` 是一个整数，用于标识在所有可用接口中哪一个接收到了数据包。`cq_event` 是一个整数，用于指示队列中数据的队头或队尾事件。`packets_recv` 是一个整数，用于指示已经成功接收到的数据包数量。

总之，这段代码定义了一个 `struct pcap_rdmasniff` 结构体，用于表示以太网数据包接收者（RDMASNIFF）的相关信息，包括接收者所需的硬件和软件资源，以及如何接收和传输数据包。


```cpp
#endif

static const int RDMASNIFF_NUM_RECEIVES = 128;
static const int RDMASNIFF_RECEIVE_SIZE = 10000;

struct pcap_rdmasniff {
	struct ibv_device *		rdma_device;
	struct ibv_context *		context;
	struct ibv_comp_channel *	channel;
	struct ibv_pd *			pd;
	struct ibv_cq *			cq;
	struct ibv_qp *			qp;
	struct ibv_flow *               flow;
	struct ibv_mr *			mr;
	u_char *			oneshot_buffer;
	unsigned long			port_num;
	int                             cq_event;
	u_int                           packets_recv;
};

```



这段代码是一个用于 Linux 操作系统中的 pcap 文件的函数，名为 rdmasc Statistics。函数的作用是统计在 pcap 文件中发生的数据包的接收、丢包和 Interface drop 情况，并返回这些统计信息。以下是函数的详细解释：

1. `rdmasniff_stats()`：这个函数接收一个 pcap 文件 handle 和一个结构体 pcap_stat 变量。函数内部使用结构体变量 `priv` 来访问数据的实际拥有者。函数的主要操作是统计数据包的接收情况。将 `priv->packets_recv` 赋值给 `stat->ps_recv`，将 `priv->drop` 的值初始化为 0，将 `priv->ifdrop` 的值初始化为 0。

2. `rdmasc cleanup()`：这个函数同样接收一个 pcap 文件 handle，不过这个函数是在函数 `rdmasniff_stats()` 函数内部被调用了。函数的主要操作是释放之前分配的内存，并关闭数据包通道、清除统计信息，最后释放 pcap 文件 handle。

3. `rdmasc` 函数的参数：

  - `handle`：一个指向 pcap 文件的指针。
  - `stat`：一个结构体变量，用于存储统计信息。这个结构体包含以下成员：
    - `ps_recv`：数据包接收的数量。
    - `ps_drop`：数据包丢包的数量。
    - `ps_ifdrop`：接口丢包的数量。

4. `rdmasc` 函数的实现主要依赖于提前分配的内存空间，这些内存空间在函数被调用时需要被释放。函数释放这些内存空间的方式是使用 `free` 函数，并使用 `pcap_cleanup_live_common()` 函数来清除统计信息。


```cpp
static int
rdmasniff_stats(pcap_t *handle, struct pcap_stat *stat)
{
	struct pcap_rdmasniff *priv = handle->priv;

	stat->ps_recv = priv->packets_recv;
	stat->ps_drop = 0;
	stat->ps_ifdrop = 0;

	return 0;
}

static void
rdmasniff_cleanup(pcap_t *handle)
{
	struct pcap_rdmasniff *priv = handle->priv;

	ibv_dereg_mr(priv->mr);
	ibv_destroy_flow(priv->flow);
	ibv_destroy_qp(priv->qp);
	ibv_destroy_cq(priv->cq);
	ibv_dealloc_pd(priv->pd);
	ibv_destroy_comp_channel(priv->channel);
	ibv_close_device(priv->context);
	free(priv->oneshot_buffer);

	pcap_cleanup_live_common(handle);
}

```

这段代码是一个用于从IPv6数据包中接收并处理数据的函数，属于“rdmasniff”家族。它属于“pcap”库，可以在IPv6数据包接收紧张时起作用。

函数接收一个IPv6数据包，接收线程ID，和64个无符号整数类型的数据。首先，它将接收到一个包含有向连接标识符(RDMASNIFF_RECEIVE_SIZE)和数据分片段的IPv6数据包。然后设置两个指针变量：一个指向IPv6数据包接收方的下一个分片段的指针，一个指向一个结构体，这个结构体定义了数据分片的信息，包括分片偏移量，数据长度，数据类型，以及分片ID等。

接下来，使用一个循环，将设置的RDMASNIFF_RECEIVE_SIZE字节的数据加载到从IPv6数据包接收方缓冲区中的下一个分片。然后使用“ibv_post_recv”函数将数据包接收方缓冲区中的数据发送到IPv6数据包发送方缓冲区中。

最后，使用一个循环，遍历坏块指针，如果数据分片没有出现，说明数据片收齐，就绪接收。


```cpp
static void
rdmasniff_post_recv(pcap_t *handle, uint64_t wr_id)
{
	struct pcap_rdmasniff *priv = handle->priv;
	struct ibv_sge sg_entry;
	struct ibv_recv_wr wr, *bad_wr;

	sg_entry.length = RDMASNIFF_RECEIVE_SIZE;
	sg_entry.addr = (uintptr_t) handle->buffer + RDMASNIFF_RECEIVE_SIZE * wr_id;
	sg_entry.lkey = priv->mr->lkey;

	wr.wr_id = wr_id;
	wr.num_sge = 1;
	wr.sg_list = &sg_entry;
	wr.next = NULL;

	ibv_post_recv(priv->qp, &wr, &bad_wr);
}

```

This is a function for managing the receive operations of a network packet socket. It receives packets from the network, filters out packets with the maximum sequence number, and marks them as received by the application.

The function takes an input parameter `handle`, which is a pointer to the packet socket handle. It is an梅花牌(max_packets) counter, which keeps track of the number of packets that have been received.

The function first checks if the maximum number of packets is reached and sets the max_packets accordingly. It then enters a while loop that continuously filters out incoming packets until the loop is exited.

In each iteration of the loop, the function first checks if the current packet has been successfully received by the application. If it has not, it marks the packet as failed and exits the loop. If the packet was successfully received, it extracts the packet sequence number and checks if it is within the snapshot limit. If it is not, the function marks the packet as received and increments the receive count.

The function also handles the case where the application uses the function to break out of the receive loop, which is an error and returns an error code.

The function returns the number of packets that were successfully received.


```cpp
static int
rdmasniff_read(pcap_t *handle, int max_packets, pcap_handler callback, u_char *user)
{
	struct pcap_rdmasniff *priv = handle->priv;
	struct ibv_cq *ev_cq;
	void *ev_ctx;
	struct ibv_wc wc;
	struct pcap_pkthdr pkth;
	u_char *pktd;
	int count = 0;

	if (!priv->cq_event) {
		while (ibv_get_cq_event(priv->channel, &ev_cq, &ev_ctx) < 0) {
			if (errno != EINTR) {
				return PCAP_ERROR;
			}
			if (handle->break_loop) {
				handle->break_loop = 0;
				return PCAP_ERROR_BREAK;
			}
		}
		ibv_ack_cq_events(priv->cq, 1);
		ibv_req_notify_cq(priv->cq, 0);
		priv->cq_event = 1;
	}

	/*
	 * This can conceivably process more than INT_MAX packets,
	 * which would overflow the packet count, causing it either
	 * to look like a negative number, and thus cause us to
	 * return a value that looks like an error, or overflow
	 * back into positive territory, and thus cause us to
	 * return a too-low count.
	 *
	 * Therefore, if the packet count is unlimited, we clip
	 * it at INT_MAX; this routine is not expected to
	 * process packets indefinitely, so that's not an issue.
	 */
	if (PACKET_COUNT_IS_UNLIMITED(max_packets))
		max_packets = INT_MAX;

	while (count < max_packets) {
		if (ibv_poll_cq(priv->cq, 1, &wc) != 1) {
			priv->cq_event = 0;
			break;
		}

		if (wc.status != IBV_WC_SUCCESS) {
			fprintf(stderr, "failed WC wr_id %" PRIu64 " status %d/%s\n",
				wc.wr_id,
				wc.status, ibv_wc_status_str(wc.status));
			continue;
		}

		pkth.len = wc.byte_len;
		pkth.caplen = min(pkth.len, (u_int)handle->snapshot);
		gettimeofday(&pkth.ts, NULL);

		pktd = (u_char *) handle->buffer + wc.wr_id * RDMASNIFF_RECEIVE_SIZE;

		if (handle->fcode.bf_insns == NULL ||
		    pcap_filter(handle->fcode.bf_insns, pktd, pkth.len, pkth.caplen)) {
			callback(user, &pkth, pktd);
			++priv->packets_recv;
			++count;
		}

		rdmasniff_post_recv(handle, wc.wr_id);

		if (handle->break_loop) {
			handle->break_loop = 0;
			return PCAP_ERROR_BREAK;
		}
	}

	return count;
}

```

This function appears to register a device-specific one-shot memory buffer for use by the `rdmasniff_do_queue` function.  It is using the `rdmasniff_reg_dev` function to register the buffer with the port, but it is not handling the actual registration of the buffer.

The function is using a pointer to an `rdmasniff_device_t` structure to pass a device-specific piece of information to the `rdmasniff_reg_dev` function.

It appears to be setting up a one-shot buffer for the purpose of receiving data through the `rdmasniff_do_queue` function, and it is registering the buffer with the port using the `rdmasniff_reg_dev` function.


```cpp
static void
rdmasniff_oneshot(u_char *user, const struct pcap_pkthdr *h, const u_char *bytes)
{
	struct oneshot_userdata *sp = (struct oneshot_userdata *) user;
	pcap_t *handle = sp->pd;
	struct pcap_rdmasniff *priv = handle->priv;

	*sp->hdr = *h;
	memcpy(priv->oneshot_buffer, bytes, h->caplen);
	*sp->pkt = priv->oneshot_buffer;
}

static int
rdmasniff_activate(pcap_t *handle)
{
	struct pcap_rdmasniff *priv = handle->priv;
	struct ibv_qp_init_attr qp_init_attr;
	struct ibv_qp_attr qp_attr;
	struct ibv_flow_attr flow_attr;
	struct ibv_port_attr port_attr;
	int i;

	priv->context = ibv_open_device(priv->rdma_device);
	if (!priv->context) {
		snprintf(handle->errbuf, PCAP_ERRBUF_SIZE,
			      "Failed to open device %s", handle->opt.device);
		goto error;
	}

	priv->pd = ibv_alloc_pd(priv->context);
	if (!priv->pd) {
		snprintf(handle->errbuf, PCAP_ERRBUF_SIZE,
			      "Failed to alloc PD for device %s", handle->opt.device);
		goto error;
	}

	priv->channel = ibv_create_comp_channel(priv->context);
	if (!priv->channel) {
		snprintf(handle->errbuf, PCAP_ERRBUF_SIZE,
			      "Failed to create comp channel for device %s", handle->opt.device);
		goto error;
	}

	priv->cq = ibv_create_cq(priv->context, RDMASNIFF_NUM_RECEIVES,
				 NULL, priv->channel, 0);
	if (!priv->cq) {
		snprintf(handle->errbuf, PCAP_ERRBUF_SIZE,
			      "Failed to create CQ for device %s", handle->opt.device);
		goto error;
	}

	ibv_req_notify_cq(priv->cq, 0);

	memset(&qp_init_attr, 0, sizeof qp_init_attr);
	qp_init_attr.send_cq = qp_init_attr.recv_cq = priv->cq;
	qp_init_attr.cap.max_recv_wr = RDMASNIFF_NUM_RECEIVES;
	qp_init_attr.cap.max_recv_sge = 1;
	qp_init_attr.qp_type = IBV_QPT_RAW_PACKET;
	priv->qp = ibv_create_qp(priv->pd, &qp_init_attr);
	if (!priv->qp) {
		snprintf(handle->errbuf, PCAP_ERRBUF_SIZE,
			      "Failed to create QP for device %s", handle->opt.device);
		goto error;
	}

	memset(&qp_attr, 0, sizeof qp_attr);
	qp_attr.qp_state = IBV_QPS_INIT;
	qp_attr.port_num = priv->port_num;
	if (ibv_modify_qp(priv->qp, &qp_attr, IBV_QP_STATE | IBV_QP_PORT)) {
		snprintf(handle->errbuf, PCAP_ERRBUF_SIZE,
			      "Failed to modify QP to INIT for device %s", handle->opt.device);
		goto error;
	}

	memset(&qp_attr, 0, sizeof qp_attr);
	qp_attr.qp_state = IBV_QPS_RTR;
	if (ibv_modify_qp(priv->qp, &qp_attr, IBV_QP_STATE)) {
		snprintf(handle->errbuf, PCAP_ERRBUF_SIZE,
			      "Failed to modify QP to RTR for device %s", handle->opt.device);
		goto error;
	}

	memset(&flow_attr, 0, sizeof flow_attr);
	flow_attr.type = IBV_FLOW_ATTR_SNIFFER;
	flow_attr.size = sizeof flow_attr;
	flow_attr.port = priv->port_num;
	priv->flow = ibv_create_flow(priv->qp, &flow_attr);
	if (!priv->flow) {
		snprintf(handle->errbuf, PCAP_ERRBUF_SIZE,
			      "Failed to create flow for device %s", handle->opt.device);
		goto error;
	}

	handle->bufsize = RDMASNIFF_NUM_RECEIVES * RDMASNIFF_RECEIVE_SIZE;
	handle->buffer = malloc(handle->bufsize);
	if (!handle->buffer) {
		snprintf(handle->errbuf, PCAP_ERRBUF_SIZE,
			      "Failed to allocate receive buffer for device %s", handle->opt.device);
		goto error;
	}

	priv->oneshot_buffer = malloc(RDMASNIFF_RECEIVE_SIZE);
	if (!priv->oneshot_buffer) {
		snprintf(handle->errbuf, PCAP_ERRBUF_SIZE,
			      "Failed to allocate oneshot buffer for device %s", handle->opt.device);
		goto error;
	}

	priv->mr = ibv_reg_mr(priv->pd, handle->buffer, handle->bufsize, IBV_ACCESS_LOCAL_WRITE);
	if (!priv->mr) {
		snprintf(handle->errbuf, PCAP_ERRBUF_SIZE,
			      "Failed to register MR for device %s", handle->opt.device);
		goto error;
	}


	for (i = 0; i < RDMASNIFF_NUM_RECEIVES; ++i) {
		rdmasniff_post_recv(handle, i);
	}

	if (!ibv_query_port(priv->context, priv->port_num, &port_attr) &&
	    port_attr.link_layer == IBV_LINK_LAYER_INFINIBAND) {
		handle->linktype = DLT_INFINIBAND;
	} else {
		handle->linktype = DLT_EN10MB;
	}

	if (handle->snapshot <= 0 || handle->snapshot > RDMASNIFF_RECEIVE_SIZE)
		handle->snapshot = RDMASNIFF_RECEIVE_SIZE;

	handle->offset = 0;
	handle->read_op = rdmasniff_read;
	handle->stats_op = rdmasniff_stats;
	handle->cleanup_op = rdmasniff_cleanup;
	handle->setfilter_op = install_bpf_program;
	handle->setdirection_op = NULL;
	handle->set_datalink_op = NULL;
	handle->getnonblock_op = pcap_getnonblock_fd;
	handle->setnonblock_op = pcap_setnonblock_fd;
	handle->oneshot_callback = rdmasniff_oneshot;
	handle->selectable_fd = priv->channel->fd;

	return 0;

```

这段代码是一个错误处理程序，它负责处理在IBV网络中出现的各种错误。以下是代码的作用：

1. 如果存在priv->mr，则调用ibv_dereg_mr函数来释放内存。
2. 如果存在priv->flow，则调用ibv_destroy_flow函数来释放内存。
3. 如果存在priv->qp，则调用ibv_destroy_qp函数来释放内存。
4. 如果存在priv->cq，则调用ibv_destroy_cq函数来释放内存。
5. 如果存在priv->channel，则调用ibv_destroy_comp_channel函数来释放内存。
6. 如果存在priv->pd，则调用ibv_dealloc_pd函数来释放内存。
7. 如果存在priv->context，则调用ibv_close_device函数来关闭设备。
8. 如果存在priv->oneshot_buffer，则释放并free内存。

如果以上函数都没有被调用，则返回PCAP_ERROR。


```cpp
error:
	if (priv->mr) {
		ibv_dereg_mr(priv->mr);
	}

	if (priv->flow) {
		ibv_destroy_flow(priv->flow);
	}

	if (priv->qp) {
		ibv_destroy_qp(priv->qp);
	}

	if (priv->cq) {
		ibv_destroy_cq(priv->cq);
	}

	if (priv->channel) {
		ibv_destroy_comp_channel(priv->channel);
	}

	if (priv->pd) {
		ibv_dealloc_pd(priv->pd);
	}

	if (priv->context) {
		ibv_close_device(priv->context);
	}

	if (priv->oneshot_buffer) {
		free(priv->oneshot_buffer);
	}

	return PCAP_ERROR;
}

```

This function appears to be part of a program that is intended to read network packets and determine which devices are producing those packets. It takes two arguments: a pointer to a variable containing network buffers (`ebuf`) and a pointer to an integer containing the status of the packet (`is_ours`). The function appears to check which devices are producing the packets and, if it finds a device that is producing packets with a specific device name, it creates a `PCAP` object and sets the device to be read from.


```cpp
pcap_t *
rdmasniff_create(const char *device, char *ebuf, int *is_ours)
{
	struct pcap_rdmasniff *priv;
	struct ibv_device **dev_list;
	int numdev;
	size_t namelen;
	const char *port;
	unsigned long port_num;
	int i;
	pcap_t *p = NULL;

	*is_ours = 0;

	dev_list = ibv_get_device_list(&numdev);
	if (!dev_list) {
		return NULL;
	}
	if (!numdev) {
		ibv_free_device_list(dev_list);
		return NULL;
	}

	namelen = strlen(device);

	port = strchr(device, ':');
	if (port) {
		port_num = strtoul(port + 1, NULL, 10);
		if (port_num > 0) {
			namelen = port - device;
		} else {
			port_num = 1;
		}
	} else {
		port_num = 1;
	}

	for (i = 0; i < numdev; ++i) {
		if (strlen(dev_list[i]->name) == namelen &&
		    !strncmp(device, dev_list[i]->name, namelen)) {
			*is_ours = 1;

			p = PCAP_CREATE_COMMON(ebuf, struct pcap_rdmasniff);
			if (p) {
				p->activate_op = rdmasniff_activate;
				priv = p->priv;
				priv->rdma_device = dev_list[i];
				priv->port_num = port_num;
			}
			break;
		}
	}

	ibv_free_device_list(dev_list);
	return p;
}

```

这段代码是一个用于解析和打印 RDMA 设备列表的函数。它接受一个指向整型指针变量 devlistp 和一个字符型指针 err_str，然后执行以下操作：

1. 获取一个指向整型指针变量 numdev 的指针。
2. 如果 numdev 为 0，函数返回 0，表示设备列表为空。
3. 遍历设备列表中的每个设备，执行以下操作：
		- 检查设备名称是否为 "RDMA sniffer"，如果是，执行以下操作：
			1. 调用函数 add_dev，传递 devlistp 和设备名称，用于将设备添加到设备列表中。
			2. 如果添加设备成功，将 ret 设置为 0，表示设备成功添加。
			3. 如果添加设备失败，将 ret 设置为 -1，并打印错误字符串 err_str。
			4. 释放设备列表中的内存，使用 ibv_free_device_list 函数。
4. 如果 numdev 为非 0 值，函数返回 ret，表示设备列表中有设备。

函数的作用是打印解析后的 RDMA 设备列表，帮助用户了解连接的设备。


```cpp
int
rdmasniff_findalldevs(pcap_if_list_t *devlistp, char *err_str)
{
	struct ibv_device **dev_list;
	int numdev;
	int i;
	int ret = 0;

	dev_list = ibv_get_device_list(&numdev);
	if (!dev_list) {
		return 0;
	}

	for (i = 0; i < numdev; ++i) {
		/*
		 * XXX - do the notions of "up", "running", or
		 * "connected" apply here?
		 */
		if (!add_dev(devlistp, dev_list[i]->name, 0, "RDMA sniffer", err_str)) {
			ret = -1;
			break;
		}
	}

	ibv_free_device_list(dev_list);
	return ret;
}

```