# Nmap源码解析 63

# `libpcap/pcap-snoop.c`

这段代码是一个C语言的函数声明，它定义了一些名为“test”的函数。这些函数没有注释，但是根据版权声明，它们的目的是允许用户自由地使用、修改和分发这些函数，前提是在这个过程中包含版权通知和使用条款。

这段代码属于令牌库（token library）或模板库（template library）的范畴，因为它们通常提供了一系列可以用于各种编程任务的功能。这些库或模板通常以源代码形式提供，用户需要包含在这个库或模板中的源代码才能在相应的环境中使用它们。


```cpp
/*
 * Copyright (c) 1993, 1994, 1995, 1996, 1997
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

这段代码是一个名为“config.h”的文件头。它通过包括了两个头文件《config.h》和《hw_modules.h》来自定义了一些常量和宏。

具体来说，该代码的作用是检查系统是否支持配置文件，如果支持，则包含一些定义好的宏，方便对外部功能进行调用。如果用户传入了无意义的参数，则不会输出任何信息。


```cpp
#ifdef HAVE_CONFIG_H
#include <config.h>
#endif

#include <sys/param.h>
#include <sys/file.h>
#include <sys/ioctl.h>
#include <sys/socket.h>
#include <sys/time.h>

#include <net/raw.h>
#include <net/if.h>

#include <netinet/in.h>
#include <netinet/in_systm.h>
```

这段代码是一个网络编程的include头文件，它包含了网络接口平台（NI）套件中定义的与网络接口相关的头文件。具体来说：

1. `#include <netinet/ip.h>`：引入了IP头文件，定义了IP协议栈中的`ip`结构体。

2. `#include <netinet/if_ether.h>`：引入了以太网头文件，定义了以太网头中的`ethernet_address`和`ethernet_port`成员变量。

3. `#include <netinet/ip_var.h>`：引入了IP变量定义头文件，定义了`in_`，`out_`，`帽_`和`自治_团体_`成员变量。

4. `#include <netinet/udp.h>`：引入了UDP头文件，定义了`udp`结构体。

5. `#include <netinet/udp_var.h>`：引入了UDP变量定义头文件，定义了`udp_local_port`和`udp_remote_port`成员变量。

6. `#include <netinet/tcp.h>`：引入了TCP头文件，定义了`tcp`结构体。

7. `#include <netinet/tcpip.h>`：引入了TCPIP头文件，定义了`reserved_报文类型`。

8. `#include <errno.h>`：引入了errno头文件，定义了错误号（EN）和错误描述符。

9. `#include <stdio.h>`：引入了stdio头文件，定义了printf函数。

10. `#include <stdlib.h>`：引入了stdlib头文件，定义了memalign函数。

11. `#include <string.h>`：引入了string.h头文件，定义了strlen函数。

12. `#include <unistd.h>`：引入了unistd头文件，定义了<unistd.h>预处理函数。

13. `#include "pcap-int.h"`：引入了pcap-int头文件，定义了`cap_##_internal_function`预处理函数。

14. `#include "pcap-int.h"`：引入了pcap-int头文件，定义了`cap_##_internal_function`预处理函数。


```cpp
#include <netinet/ip.h>
#include <netinet/if_ether.h>
#include <netinet/ip_var.h>
#include <netinet/udp.h>
#include <netinet/udp_var.h>
#include <netinet/tcp.h>
#include <netinet/tcpip.h>

#include <errno.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

#include "pcap-int.h"

```

这段代码的作用是定义了一个名为 pcap_snoop 的结构体，用于在 Snoop 设备上捕捉数据。该结构体包含一个名为 stat 的 pcap_stat 成员，用于统计在 Snoop 设备上捕获到的数据。

该代码还引入了一个名为 pcap_read_snoop 的函数，用于从 Snoop 设备中读取数据。该函数的第一个参数是一个指向 pcap_t 类型的变量指针，用于读取数据；第二个参数是一个整数，用于指定要读取的数据长度；第三个参数是一个名为 callback 的函数指针，用于在接收到数据时执行指定函数；第四个参数是一个指向 u_char 类型数据的指针 user，用于存储在 Snoop 设备上读取的数据。

该代码还包含一个前缀 #ifdef 男 Have_os_protpo_H，用于检查是否定义了与 Snoop 设备有关的头文件。如果定义了该头文件，则不会执行前缀中的任何代码。


```cpp
#ifdef HAVE_OS_PROTO_H
#include "os-proto.h"
#endif

/*
 * Private data for capturing on snoop devices.
 */
struct pcap_snoop {
	struct pcap_stat stat;
};

static int
pcap_read_snoop(pcap_t *p, int cnt, pcap_handler callback, u_char *user)
{
	struct pcap_snoop *psn = p->priv;
	int cc;
	register struct snoopheader *sh;
	register u_int datalen;
	register u_int caplen;
	register u_char *cp;

```



This is a function definition for `pcap_add_rsni_t` which adds a function call to `snprintf` to print a Snoop packet to a given NSP victim. Here's a breakdown of what this function does:

1. It initializes the input parameters `p` and `sh`.
2. It reads the length of the data from the `p->buffer` pointer and the data length from the `sh->snoop_packetlen` field.
3. It checks if the data length is shorter than the expected data length and adds 64K to it if it is.
4. It creates a `struct pcap_pkthdr` to hold the timestamp and len of the packet.
5. It creates a pointer `cp` to hold the start of the packet.
6. It uses the `snprintf` function to print the packet to the `user`-pointer.
7. It returns 0 if the packet was successfully printed, or returns an error if an error occurred.

This function definition adds a functionality to the `pcap_open` function, allowing to add a custom function call to `snprintf` for custom processing of Snoop packets.


```cpp
again:
	/*
	 * Has "pcap_breakloop()" been called?
	 */
	if (p->break_loop) {
		/*
		 * Yes - clear the flag that indicates that it
		 * has, and return -2 to indicate that we were
		 * told to break out of the loop.
		 */
		p->break_loop = 0;
		return (-2);
	}
	cc = read(p->fd, (char *)p->buffer, p->bufsize);
	if (cc < 0) {
		/* Don't choke when we get ptraced */
		switch (errno) {

		case EINTR:
			goto again;

		case EWOULDBLOCK:
			return (0);			/* XXX */
		}
		pcap_fmt_errmsg_for_errno(p->errbuf, sizeof(p->errbuf),
		    errno, "read");
		return (-1);
	}
	sh = (struct snoopheader *)p->buffer;
	datalen = sh->snoop_packetlen;

	/*
	 * XXX - Sigh, snoop_packetlen is a 16 bit quantity.  If we
	 * got a short length, but read a full sized snoop pakcet,
	 * assume we overflowed and add back the 64K...
	 */
	if (cc == (p->snapshot + sizeof(struct snoopheader)) &&
	    (datalen < p->snapshot))
		datalen += (64 * 1024);

	caplen = (datalen < p->snapshot) ? datalen : p->snapshot;
	cp = (u_char *)(sh + 1) + p->offset;		/* XXX */

	/*
	 * XXX unfortunately snoop loopback isn't exactly like
	 * BSD's.  The address family is encoded in the first 2
	 * bytes rather than the first 4 bytes!  Luckily the last
	 * two snoop loopback bytes are zeroed.
	 */
	if (p->linktype == DLT_NULL && *((short *)(cp + 2)) == 0) {
		u_int *uip = (u_int *)cp;
		*uip >>= 16;
	}

	if (p->fcode.bf_insns == NULL ||
	    pcap_filter(p->fcode.bf_insns, cp, datalen, caplen)) {
		struct pcap_pkthdr h;
		++psn->stat.ps_recv;
		h.ts.tv_sec = sh->snoop_timestamp.tv_sec;
		h.ts.tv_usec = sh->snoop_timestamp.tv_usec;
		h.len = datalen;
		h.caplen = caplen;
		(*callback)(user, &h, cp);
		return (1);
	}
	return (0);
}

```

这段代码是一个名为 `pcap_inject_snoop` 的函数，属于 `pcap_main` 函数堆。它接受一个 `pcap_t` 类型的数据指针 `p`、一个 `const void *` 类型的数据缓冲区 `buf` 和一个 `int` 类型的数据大小 `size`。

函数的作用是向网络数据包中注入一个 SNOOP（Snoop/N汗/Nosc）事件，以便记录数据包的传输信息。通过 `write` 函数将数据缓冲区 `buf` 中的数据写入到 `pcap_t` 类型的数据指针 `p` 的数据文件 `fd` 中，并返回写入结果。如果写入失败，函数会使用 `pcap_fmt_errmsg_for_errno` 函数错误地打印错误消息，并返回一个负值。

从函数的实现来看，它似乎在尝试创建一个数据包并将其发送到网络中，以便记录数据包的传输信息。但是，由于函数没有明确的返回值，因此很难确定它是否成功创建了数据包。


```cpp
static int
pcap_inject_snoop(pcap_t *p, const void *buf, int size)
{
	int ret;

	/*
	 * XXX - libnet overwrites the source address with what I
	 * presume is the interface's address; is that required?
	 */
	ret = write(p->fd, buf, size);
	if (ret == -1) {
		pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
		    errno, "send");
		return (-1);
	}
	return (ret);
}

```

这段代码是一个名为 "pcap_stats_snoop" 的函数，属于 "pcap-以太网数据包抓取器"（pcap-ethernet-data-包抓取器）的 "rawstats" 结构体成员。它负责将统计数据从 "rawstats" 结构体中提取并返回，用于 "pcap_stats_snapshot" 函数。

函数接收两个参数：一个指向 "pcap_t" 结构体的指针变量 "p"，另一个是一个指向 "struct pcap_stat" 结构体的指针变量 "ps"。

函数首先创建了一个名为 "psn" 的 "struct pcap_snoop" 结构体变量，它与 "pcap_t" 结构体中的 "priv" 成员相关联。接着，它创建了一个指向 "rawstats" 结构体的大指针 "rs"，并将其初始化为一个包含所有统计数据的零字节。

接下来，函数调用 "ioctl" 函数，传递 "pcap_fd" 文件描述符和 "SIOCRAWSTATS" 命令，将 "rawstats" 结构体中的统计数据提取并存储到 "rs" 指向的内存区域。

然后，函数计算 "psn->stat.ps_drop" 字段，它将 "rs->rs_snoop.ss_ifdrops"、"rs->rs_snoop.ss_sbdrops" 和 "rs->rs_drain.ds_ifdrops"、"rs->rs_drain.ds_sbdrops" 的值相加，得到 "psn->stat.ps_drop" 字段的值。

最后，函数将提取的统计数据存储到 "ps" 指向的内存区域，并返回 0。


```cpp
static int
pcap_stats_snoop(pcap_t *p, struct pcap_stat *ps)
{
	struct pcap_snoop *psn = p->priv;
	register struct rawstats *rs;
	struct rawstats rawstats;

	rs = &rawstats;
	memset(rs, 0, sizeof(*rs));
	if (ioctl(p->fd, SIOCRAWSTATS, (char *)rs) < 0) {
		pcap_fmt_errmsg_for_errno(p->errbuf, sizeof(p->errbuf),
		    errno, "SIOCRAWSTATS");
		return (-1);
	}

	/*
	 * "ifdrops" are those dropped by the network interface
	 * due to resource shortages or hardware errors.
	 *
	 * "sbdrops" are those dropped due to socket buffer limits.
	 *
	 * As filter is done in userland, "sbdrops" counts packets
	 * regardless of whether they would've passed the filter.
	 *
	 * XXX - does this count *all* Snoop or Drain sockets,
	 * rather than just this socket?  If not, why does it have
	 * both Snoop and Drain statistics?
	 */
	psn->stat.ps_drop =
	    rs->rs_snoop.ss_ifdrops + rs->rs_snoop.ss_sbdrops +
	    rs->rs_drain.ds_ifdrops + rs->rs_drain.ds_sbdrops;

	/*
	 * "ps_recv" counts only packets that passed the filter.
	 * As filtering is done in userland, this does not include
	 * packets dropped because we ran out of buffer space.
	 */
	*ps = psn->stat;
	return (0);
}

```

The code appears to be setting up the Point-to-Point (PPP) protocol on an Ethernet interface, and it is using a specific type of over-the-line Ethernet device.

It first sets thell_hdrlen to the appropriate length based on the first line of the device options passed in by the user (device name "ppp" or "ppp<linktype>").

Then, it checks the device name to see if it is "ppp" or if it contains a "linktype" value, and sets the linktype accordingly.

Finally, it sets the snapshot parameter based on the user specified value and an if statement.

It is important to note that the code is using the "PCAP\_ERRBUF\_SIZE" without the need for () .


```cpp
/* XXX can't disable promiscuous */
static int
pcap_activate_snoop(pcap_t *p)
{
	int fd;
	struct sockaddr_raw sr;
	struct snoopfilter sf;
	u_int v;
	int ll_hdrlen;
	int snooplen;
	struct ifreq ifr;

	fd = socket(PF_RAW, SOCK_RAW, RAWPROTO_SNOOP);
	if (fd < 0) {
		pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
		    errno, "snoop socket");
		goto bad;
	}
	p->fd = fd;
	memset(&sr, 0, sizeof(sr));
	sr.sr_family = AF_RAW;
	(void)strncpy(sr.sr_ifname, p->opt.device, sizeof(sr.sr_ifname));
	if (bind(fd, (struct sockaddr *)&sr, sizeof(sr))) {
		/*
		 * XXX - there's probably a particular bind error that
		 * means "there's no such device" and a particular bind
		 * error that means "that device doesn't support snoop";
		 * they might be the same error, if they both end up
		 * meaning "snoop doesn't know about that device".
		 */
		pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
		    errno, "snoop bind");
		goto bad;
	}
	memset(&sf, 0, sizeof(sf));
	if (ioctl(fd, SIOCADDSNOOP, &sf) < 0) {
		pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
		    errno, "SIOCADDSNOOP");
		goto bad;
	}
	if (p->opt.buffer_size != 0)
		v = p->opt.buffer_size;
	else
		v = 64 * 1024;	/* default to 64K buffer size */
	(void)setsockopt(fd, SOL_SOCKET, SO_RCVBUF, (char *)&v, sizeof(v));
	/*
	 * XXX hack - map device name to link layer type
	 */
	if (strncmp("et", p->opt.device, 2) == 0 ||	/* Challenge 10 Mbit */
	    strncmp("ec", p->opt.device, 2) == 0 ||	/* Indigo/Indy 10 Mbit,
							   O2 10/100 */
	    strncmp("ef", p->opt.device, 2) == 0 ||	/* O200/2000 10/100 Mbit */
	    strncmp("eg", p->opt.device, 2) == 0 ||	/* Octane/O2xxx/O3xxx Gigabit */
	    strncmp("gfe", p->opt.device, 3) == 0 ||	/* GIO 100 Mbit */
	    strncmp("fxp", p->opt.device, 3) == 0 ||	/* Challenge VME Enet */
	    strncmp("ep", p->opt.device, 2) == 0 ||	/* Challenge 8x10 Mbit EPLEX */
	    strncmp("vfe", p->opt.device, 3) == 0 ||	/* Challenge VME 100Mbit */
	    strncmp("fa", p->opt.device, 2) == 0 ||
	    strncmp("qaa", p->opt.device, 3) == 0 ||
	    strncmp("cip", p->opt.device, 3) == 0 ||
	    strncmp("el", p->opt.device, 2) == 0) {
		p->linktype = DLT_EN10MB;
		p->offset = RAW_HDRPAD(sizeof(struct ether_header));
		ll_hdrlen = sizeof(struct ether_header);
		/*
		 * This is (presumably) a real Ethernet capture; give it a
		 * link-layer-type list with DLT_EN10MB and DLT_DOCSIS, so
		 * that an application can let you choose it, in case you're
		 * capturing DOCSIS traffic that a Cisco Cable Modem
		 * Termination System is putting out onto an Ethernet (it
		 * doesn't put an Ethernet header onto the wire, it puts raw
		 * DOCSIS frames out on the wire inside the low-level
		 * Ethernet framing).
		 *
		 * XXX - are there any sorts of "fake Ethernet" that have
		 * Ethernet link-layer headers but that *shouldn't offer
		 * DLT_DOCSIS as a Cisco CMTS won't put traffic onto it
		 * or get traffic bridged onto it?  "el" is for ATM LANE
		 * Ethernet devices, so that might be the case for them;
		 * the same applies for "qaa" classical IP devices.  If
		 * "fa" devices are for FORE SPANS, that'd apply to them
		 * as well; what are "cip" devices - some other ATM
		 * Classical IP devices?
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
	} else if (strncmp("ipg", p->opt.device, 3) == 0 ||
		   strncmp("rns", p->opt.device, 3) == 0 ||	/* O2/200/2000 FDDI */
		   strncmp("xpi", p->opt.device, 3) == 0) {
		p->linktype = DLT_FDDI;
		p->offset = 3;				/* XXX yeah? */
		ll_hdrlen = 13;
	} else if (strncmp("ppp", p->opt.device, 3) == 0) {
		p->linktype = DLT_RAW;
		ll_hdrlen = 0;	/* DLT_RAW meaning "no PPP header, just the IP packet"? */
	} else if (strncmp("qfa", p->opt.device, 3) == 0) {
		p->linktype = DLT_IP_OVER_FC;
		ll_hdrlen = 24;
	} else if (strncmp("pl", p->opt.device, 2) == 0) {
		p->linktype = DLT_RAW;
		ll_hdrlen = 0;	/* Cray UNICOS/mp pseudo link */
	} else if (strncmp("lo", p->opt.device, 2) == 0) {
		p->linktype = DLT_NULL;
		ll_hdrlen = 4;
	} else {
		snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
		    "snoop: unknown physical layer type");
		goto bad;
	}

	if (p->opt.rfmon) {
		/*
		 * No monitor mode on Irix (no Wi-Fi devices on
		 * hardware supported by Irix).
		 */
		return (PCAP_ERROR_RFMON_NOTSUP);
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

```

这段代码的作用是设置 Interface 设备文件描述符时指定的设备类型及其MTU（最大传输单元）。它首先通过调用 `strncpy` 函数从设备文件描述符中读取设备名称，然后使用 `ioctl` 函数获取设备描述符的 MTU，并将其存储到 `ifr.ifr_name` 字段中。如果 MTU 设置失败，则会输出错误信息，并跳转到 `bad` 标签。

对于选项（a），由于设备文件描述符的 MTU 大小未知，因此无法确定 `ifr.ifr_name` 是否已正确设置；对于选项（b），由于 `ifr_mtu` 在某些版本的 `IRIX` 中未定义，因此需要定义；对于选项（c），由于 `ifr_metric` 是 `ifr_ifru` 结构中的一个成员，因此 `ifr.ifr_name` 和 `ifr.ifr_metric` 可以使用相同的值。


```cpp
#ifdef SIOCGIFMTU
	/*
	 * XXX - IRIX appears to give you an error if you try to set the
	 * capture length to be greater than the MTU, so let's try to get
	 * the MTU first and, if that succeeds, trim the snap length
	 * to be no greater than the MTU.
	 */
	(void)strncpy(ifr.ifr_name, p->opt.device, sizeof(ifr.ifr_name));
	if (ioctl(fd, SIOCGIFMTU, (char *)&ifr) < 0) {
		pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
		    errno, "SIOCGIFMTU");
		goto bad;
	}
	/*
	 * OK, we got it.
	 *
	 * XXX - some versions of IRIX 6.5 define "ifr_mtu" and have an
	 * "ifru_metric" member of the "ifr_ifru" union in an "ifreq"
	 * structure, others don't.
	 *
	 * I've no idea what's going on, so, if "ifr_mtu" isn't defined,
	 * we define it as "ifr_metric", as using that field appears to
	 * work on the versions that lack "ifr_mtu" (and, on those that
	 * don't lack it, "ifru_metric" and "ifru_mtu" are both "int"
	 * members of the "ifr_ifru" union, which suggests that they
	 * may be interchangeable in this case).
	 */
```

This is a code snippet for setting up a Socket to act as a Data Link.

It appears to be using the "pcap" library to implement the data link, and it is using the "setfilter\_op" to install a BPF (Built-in Function) program to filter the data passed through the socket.

It also appears to be using the "set\_datalink\_op" and "getnonblock\_op" options with the "pcap\_setnonblock\_fd" and "pcap\_getnonblock\_fd" functions, but it is not clear what these options do.

It seems that the library is using "setfilter\_op" with the "pcap\_read\_snoop" and "pcap\_inject\_snoop" options to filter the data passed through the socket, but it is not clear what is being filtered.

It is also using the "pcap\_fmt\_errmsg\_for\_errno" function to convert any errors to a string and display it in the "errbuf" array.

I would recommend reviewing the code more closely to fully understand how it is functioning.


```cpp
#ifndef ifr_mtu
#define ifr_mtu	ifr_metric
#endif
	if (p->snapshot > ifr.ifr_mtu + ll_hdrlen)
		p->snapshot = ifr.ifr_mtu + ll_hdrlen;
#endif

	/*
	 * The argument to SIOCSNOOPLEN is the number of link-layer
	 * payload bytes to capture - it doesn't count link-layer
	 * header bytes.
	 */
	snooplen = p->snapshot - ll_hdrlen;
	if (snooplen < 0)
		snooplen = 0;
	if (ioctl(fd, SIOCSNOOPLEN, &snooplen) < 0) {
		pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
		    errno, "SIOCSNOOPLEN");
		goto bad;
	}
	v = 1;
	if (ioctl(fd, SIOCSNOOPING, &v) < 0) {
		pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
		    errno, "SIOCSNOOPING");
		goto bad;
	}

	p->bufsize = 4096;				/* XXX */
	p->buffer = malloc(p->bufsize);
	if (p->buffer == NULL) {
		pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
		    errno, "malloc");
		goto bad;
	}

	/*
	 * "p->fd" is a socket, so "select()" should work on it.
	 */
	p->selectable_fd = p->fd;

	p->read_op = pcap_read_snoop;
	p->inject_op = pcap_inject_snoop;
	p->setfilter_op = install_bpf_program;	/* no kernel filtering */
	p->setdirection_op = NULL;	/* Not implemented. */
	p->set_datalink_op = NULL;	/* can't change data link type */
	p->getnonblock_op = pcap_getnonblock_fd;
	p->setnonblock_op = pcap_setnonblock_fd;
	p->stats_op = pcap_stats_snoop;

	return (0);
 bad:
	pcap_cleanup_live_common(p);
	return (PCAP_ERROR);
}

```

这段代码定义了一个名为`pcap_t`的指针变量`p`，以及一个名为`pcap_create_interface`的函数。

`pcap_t`是一个指向`pcap_结构体`的指针，`pcap_create_interface`函数则用于创建一个`pcap_t`类型的变量。

该函数的参数包括一个指向字符串的`device`参数，和一个字符指针`ebuf`。函数首先使用`PCAP_CREATE_COMMON`函数将`ebuf`和一些辅助函数打包成一个`pcap_snoop`结构体，然后将该结构体的指针赋给`p`变量，从而将`pcap_t`指针与`pcap_snoop`绑定。

如果`device`参数所指的设备不支持任何 snoop(也就是不支持数据包 sniffing)，则函数将返回一个`NULL`指针。否则，函数返回一个指向`pcap_t`变量的`pcap_t`指针。


```cpp
pcap_t *
pcap_create_interface(const char *device _U_, char *ebuf)
{
	pcap_t *p;

	p = PCAP_CREATE_COMMON(ebuf, struct pcap_snoop);
	if (p == NULL)
		return (NULL);

	p->activate_op = pcap_activate_snoop;
	return (p);
}

/*
 * XXX - there's probably a particular bind error that means "that device
 * doesn't support snoop"; if so, we should try a bind and use that.
 */
```



这段代码定义了两个静态函数：

1. `static int can_be_bound(const char *name _U_)`：该函数用于检查给定的字符串是否被绑定到任何物理设备(如内存或磁盘)。函数返回一个整数，表示给定字符串是否可以被绑定。

2. `static int get_if_flags(const char *name _U_, bpf_u_int32 *flags _U_, char *errbuf _U_)`：该函数用于获取给定字符串的 if 标志位。函数参数包括两个参数，第一个参数是一个字符串指针和一个 int 类型的变量，表示 if 标志位，第二个参数是一个字符数组，用于存储 if 标志位的值。函数返回一个 int 类型的变量，表示 if 标志位的值。

函数 `can_be_bound` 的作用是检查给定字符串是否可以被绑定到任何物理设备。它简单地返回一个整数，表示字符串是否可以被绑定。

函数 `get_if_flags` 的作用是获取给定字符串的 if 标志位。它接受一个字符串指针、一个 int 类型的参数和一个字符数组，用于存储 if 标志位的值。函数返回一个 int 类型的变量，表示 if 标志位的值。

函数内部没有做任何实际的工作，因为它们只是定义了一些函数，没有实现任何具体的逻辑。


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
	 * Nothing we can do.
	 * XXX - is there a way to find out whether an adapter has
	 * something plugged into it?
	 */
	return (0);
}

```



这两段代码都是用C语言编写的，用于处理网络协议头部信息 packet。

第一段代码 `pcap_platform_finddevs` 是一个用于查找平台设备列表的函数，接受一个指向 `pcap_if_list_t` 类型的指针变量 `devlistp` 和一个字符指针 `errbuf`，用于存储错误信息。函数使用 `pcap_findalldevs_interfaces` 函数来返回平台设备列表中可用的接口列表，并将结果存储在 `errbuf` 指向的内存区域中。该函数使用 `can_be_bound` 和 `get_if_flags` 参数来过滤出可用的接口。

第二段代码 `pcap_lib_version` 是一个用于返回 libpcap 版本字符串的函数。它使用 `PCAP_VERSION_STRING` 函数来获取当前 libpcap 版本的名称和版本号，并将结果存储在 `const char *` 类型的变量中。

总的来说，这两段代码都是 libpcap 库中的函数，用于获取网络协议头部信息 packet。


```cpp
int
pcap_platform_finddevs(pcap_if_list_t *devlistp, char *errbuf)
{
	return (pcap_findalldevs_interfaces(devlistp, errbuf, can_be_bound,
	    get_if_flags));
}

/*
 * Libpcap version string.
 */
const char *
pcap_lib_version(void)
{
	return (PCAP_VERSION_STRING);
}

```

# `libpcap/pcap-tc.c`

这段代码是一个C语言的函数声明，定义了一个名为“盘古”的函数。盘古函数声明中包含了一些版权声明和限制，说明该函数可以自由地被分发和使用，但不能对原始代码进行修改。同时，盘古函数的使用需要满足一些条件，例如保留版权通知、包含版权声明的二进制文件等等。


```cpp
/*
 * Copyright (c) 2008 CACE Technologies, Davis (California)
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
 * 3. Neither the name of CACE Technologies nor the names of its
 * contributors may be used to endorse or promote products derived from
 * this software without specific prior written permission.
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
 *
 */

```

这段代码是一个简单的预处理指令，用于检查系统是否支持配置文件(config.h)。如果系统支持配置文件，那么就会包含在包含头文件中。如果不支持，则会包含一个名为"NO_CONFIG_H"的定义。

具体来说，首先包含一个名为"config.h"的文件，如果文件存在，则包含其中的所有内容。如果不存在，则定义一个名为"NO_CONFIG_H"的常量，将其值为真，表示不包含"config.h"。

接下来，包含pcap.h和pcap-int.h头文件，这些文件是用于网络数据采集和处理的头文件。

然后，包含"pcap-tc.h"头文件，这个头文件可能是一个用于网络测试和调试的头部文件。

接着，包含一个名为"malloc.h"的头部文件，这个文件用于定义内存管理函数。

然后，包含一个名为"memory.h"的头部文件，这个文件用于定义一些与内存相关的函数。

接下来，包含一个名为"string.h"的头部文件，这个文件包含一些字符串操作函数。

然后，包含一个名为"errno.h"的头部文件，这个文件包含错误码的定义。

最后，定义一个名为"NO_CONFIG_H"，将其值为真，表示不包含"config.h"。


```cpp
#ifdef HAVE_CONFIG_H
#include <config.h>
#endif

#include <pcap.h>
#include <pcap-int.h>

#include "pcap-tc.h"

#include <malloc.h>
#include <memory.h>
#include <string.h>
#include <errno.h>

#ifdef _WIN32
```

这段代码是一个通用的函数指针类型，定义了不同版本的函数，用于在程序中查询和设置TC设备的特性。

具体来说，这些函数指针类型包括：

- `TcFcnQueryPortList`：用于查询设备中的端口列表，并返回端口数量和端口号；
- `TcFcnFreePortList`：用于释放设备中的端口列表，将所有端口列表返回给调用者；
- `TcFcnStatusGetString`：用于获取设备当前的状态，并将其返回，以便用户查看设备状态；
- `TcFcnPortGetName`：用于获取设备的名称，并将其返回；
- `TcFcnPortGetDescription`：用于获取设备的描述，并将其返回；
- `TcFcnInstanceOpenByName`：用于打开设备实例，并返回实例的内存指针；
- `TcFcnInstanceClose`：用于关闭设备实例，并将其状态设置为 close;
- `TcFcnInstanceSetFeature`：用于设置设备的特性，并将其状态设置为 successful;
- `TcFcnInstanceQueryFeature`：用于查询设备的特性，并返回特定特性的值。

每个函数指针都使用 `TC_CALLCONV` 函数进行调用，其中 `TcFcnQueryPortList` 和 `TcFcnFreePortList` 是通过 `TC_CALLCONV *TcFcnQueryPortList` 和 `TC_CALLCONV *TcFcnFreePortList` 函数定义的。


```cpp
#include <tchar.h>
#endif

typedef TC_STATUS	(TC_CALLCONV *TcFcnQueryPortList)			(PTC_PORT *ppPorts, PULONG pLength);
typedef TC_STATUS	(TC_CALLCONV *TcFcnFreePortList)			(TC_PORT *pPorts);

typedef PCHAR		(TC_CALLCONV *TcFcnStatusGetString)			(TC_STATUS status);

typedef PCHAR		(TC_CALLCONV *TcFcnPortGetName)				(TC_PORT port);
typedef PCHAR		(TC_CALLCONV *TcFcnPortGetDescription)		(TC_PORT port);

typedef TC_STATUS	(TC_CALLCONV *TcFcnInstanceOpenByName)		(PCHAR name, PTC_INSTANCE pInstance);
typedef TC_STATUS	(TC_CALLCONV *TcFcnInstanceClose)			(TC_INSTANCE instance);
typedef TC_STATUS	(TC_CALLCONV *TcFcnInstanceSetFeature)		(TC_INSTANCE instance, ULONG feature, ULONG value);
typedef TC_STATUS	(TC_CALLCONV *TcFcnInstanceQueryFeature)	(TC_INSTANCE instance, ULONG feature, PULONG pValue);
```

This is a list of function pointers for the `TC_CALLCONV` interface, which is used to manage the transmission and reception of packets in a technology known as IEEE 802.11. The function pointers are organized into different categories, including `TC_API_UNLOADED`, `TC_API_LOADED`, `TC_API_CANNOT_LOAD`, and `TC_API_LOADING`.

The first category includes functions that should be implemented by the user, such as `TcFcnInstanceTransmitPackets`, `TcFcnInstanceQueryStatistics`, and `TcFcnPacketsBufferCreate`, `TcFcnPacketsBufferDestroy`, and `TcFcnPacketsBufferQueryNextPacket`.

The second category includes functions from the `TcFcn统计`接口，包括 `TcFcn统计摧毁`、`TcFcn统计更新`、`TcFcn统计查询值`。

The third category是函数指针，描述了如何处理 `TC_PACKETS_BUFFER` 类，包括 `TcFcn Packets 缓冲区创建`、`TcFcn Packets 缓冲区摧毁` 和 `TcFcn Packets 缓冲区获取下一个数据包`。

The fourth category描述了如何处理 `PTC_STATISTICS` 类，包括 `TcFcn 统计摧毁` 和 `TcFcn 统计更新`。

The fifth category是关于如何处理 `ulong` 类型的数据，包括 `TcFcn统计摧毁` 和 `TcFcn统计更新`。

The sixth category似乎没有列出具体的函数，只是列出了一个枚举类型，包括 `TC_API_UNLOADED`、`TC_API_LOADED`、`TC_API_CANNOT_LOAD` 和 `TC_API_LOADING`。


```cpp
typedef TC_STATUS	(TC_CALLCONV *TcFcnInstanceReceivePackets)	(TC_INSTANCE instance, PTC_PACKETS_BUFFER pBuffer);
typedef HANDLE		(TC_CALLCONV *TcFcnInstanceGetReceiveWaitHandle) (TC_INSTANCE instance);
typedef TC_STATUS	(TC_CALLCONV *TcFcnInstanceTransmitPackets)	(TC_INSTANCE instance, TC_PACKETS_BUFFER pBuffer);
typedef TC_STATUS	(TC_CALLCONV *TcFcnInstanceQueryStatistics)	(TC_INSTANCE instance, PTC_STATISTICS pStatistics);

typedef TC_STATUS	(TC_CALLCONV *TcFcnPacketsBufferCreate)		(ULONG size, PTC_PACKETS_BUFFER pBuffer);
typedef VOID		(TC_CALLCONV *TcFcnPacketsBufferDestroy)	(TC_PACKETS_BUFFER buffer);
typedef TC_STATUS	(TC_CALLCONV *TcFcnPacketsBufferQueryNextPacket)(TC_PACKETS_BUFFER buffer, PTC_PACKET_HEADER pHeader, PVOID *ppData);
typedef TC_STATUS	(TC_CALLCONV *TcFcnPacketsBufferCommitNextPacket)(TC_PACKETS_BUFFER buffer, PTC_PACKET_HEADER pHeader, PVOID pData);

typedef VOID		(TC_CALLCONV *TcFcnStatisticsDestroy)		(TC_STATISTICS statistics);
typedef TC_STATUS	(TC_CALLCONV *TcFcnStatisticsUpdate)		(TC_STATISTICS statistics);
typedef TC_STATUS	(TC_CALLCONV *TcFcnStatisticsQueryValue)	(TC_STATISTICS statistics, ULONG counterId, PULONGLONG pValue);

typedef enum LONG
{
	TC_API_UNLOADED = 0,
	TC_API_LOADED,
	TC_API_CANNOT_LOAD,
	TC_API_LOADING
}
	TC_API_LOAD_STATUS;


```

这是一个定义的结构体 `TC_FUNCTIONS`，用于表示函数指针变量。

该结构体包含以下成员：

- `TC_API_LOAD_STATUS`：表示函数指针是否已经加载完成。
- `hTcApiDllHandle`：指向函数指针所指 DLL 文件的句柄。
- `QueryPortList`：是一个函数指针，用于查询函数指针所指向的 DLL 中所有可用的函数的端点列表。
- `FreePortList`：是一个函数指针，用于释放端点列表，使函数指针所指向的 DLL 中不存在该端点列表。
- `StatusGetString`：是一个函数指针，用于获取函数指针所指向的 DLL 中所有函数的状态字符串表示。
- `PortGetName`：是一个函数指针，用于获取函数指针所指向的 DLL 中函数的名称。
- `PortGetDescription`：是一个函数指针，用于获取函数指针所指向的 DLL 中函数的描述。
- `InstanceOpenByName`：是一个函数指针，用于根据函数名称打开函数指针所指向的 DLL 文件。
- `InstanceClose`：是一个函数指针，用于关闭函数指针所指向的 DLL 文件。
- `InstanceSetFeature`：是一个函数指针，用于设置函数指针所指向的 DLL 中的功能特征。
- `InstanceQueryFeature`：是一个函数指针，用于查询函数指针所指向的 DLL 中是否有指定的功能特征。
- `InstanceReceivePackets`：是一个函数指针，用于接收函数指针所指向的 DLL 中的数据包。


```cpp
typedef struct _TC_FUNCTIONS
{
	TC_API_LOAD_STATUS			LoadStatus;
#ifdef _WIN32
	HMODULE						hTcApiDllHandle;
#endif
	TcFcnQueryPortList			QueryPortList;
	TcFcnFreePortList			FreePortList;
	TcFcnStatusGetString		StatusGetString;

	TcFcnPortGetName			PortGetName;
	TcFcnPortGetDescription		PortGetDescription;

	TcFcnInstanceOpenByName		InstanceOpenByName;
	TcFcnInstanceClose			InstanceClose;
	TcFcnInstanceSetFeature		InstanceSetFeature;
	TcFcnInstanceQueryFeature	InstanceQueryFeature;
	TcFcnInstanceReceivePackets	InstanceReceivePackets;
```

这段代码是一个 C++ 的 header 文件，其中包含了一些函数声明，定义了一些 macros，然后定义了一些变量。

```cppc++```
```cppc++```


```cpp
#ifdef _WIN32
	TcFcnInstanceGetReceiveWaitHandle InstanceGetReceiveWaitHandle;
#endif
	TcFcnInstanceTransmitPackets InstanceTransmitPackets;
	TcFcnInstanceQueryStatistics InstanceQueryStatistics;

	TcFcnPacketsBufferCreate	PacketsBufferCreate;
	TcFcnPacketsBufferDestroy	PacketsBufferDestroy;
	TcFcnPacketsBufferQueryNextPacket	PacketsBufferQueryNextPacket;
	TcFcnPacketsBufferCommitNextPacket  PacketsBufferCommitNextPacket;

	TcFcnStatisticsDestroy		StatisticsDestroy;
	TcFcnStatisticsUpdate		StatisticsUpdate;
	TcFcnStatisticsQueryValue	StatisticsQueryValue;
}
	TC_FUNCTIONS;

```



该代码定义了四个名为 TcCreatePcapIfFromPort、TcSetDatalink、TcGetNonBlock 和 TcSetNonBlock 的函数，以及四个名为 TcCleanup、TcInject、TcRead 和 TcStats 的函数。这些函数用于管理 Linux TCP/IP 数据包 capture 链路的操作。

具体来说，这些函数实现了一个简单的数据包 capture 链路的创建和配置，包括以下步骤：

1. 创建一个指向链路根的指针
2. 设置数据链路传输模式为 bpf
3. 获取当前链路接收的等待句柄
4. 设置数据包发送超时时间为 100 毫秒
5. 返回数据包接收超时时间

这些函数的具体实现可能还需要根据具体环境进行适当的调整。


```cpp
static pcap_if_t* TcCreatePcapIfFromPort(TC_PORT port);
static int TcSetDatalink(pcap_t *p, int dlt);
static int TcGetNonBlock(pcap_t *p);
static int TcSetNonBlock(pcap_t *p, int nonblock);
static void TcCleanup(pcap_t *p);
static int TcInject(pcap_t *p, const void *buf, int size);
static int TcRead(pcap_t *p, int cnt, pcap_handler callback, u_char *user);
static int TcStats(pcap_t *p, struct pcap_stat *ps);
#ifdef _WIN32
static struct pcap_stat *TcStatsEx(pcap_t *p, int *pcap_stat_size);
static int TcSetBuff(pcap_t *p, int dim);
static int TcSetMode(pcap_t *p, int mode);
static int TcSetMinToCopy(pcap_t *p, int size);
static HANDLE TcGetReceiveWaitHandle(pcap_t *p);
static int TcOidGetRequest(pcap_t *p, bpf_u_int32 oid, void *data, size_t *lenp);
```



该代码是一个名为TcOidSetRequest的函数，属于include.h文件。其作用是设置传输设备（如无线网络卡）的oid参数，以支持按需上下文（NEST）模式。具体地，该函数接收一个pcap_t类型的数据包指针、一个const类型的oid参数、一个const类型的数据缓冲区以及一个size_t类型的输出长度指针。函数将这些参数用于设置传输设备的oid参数，并返回一个int类型的结果表示传输成功或失败。

函数内部定义了一系列函数，包括TC_FUNCTIONS结构体，用于声明函数的实现。这些函数具体实现了从传输设备中读取数据包、向传输设备发送数据包等功能。

另外，还定义了一个名为TcSendqueueTransmit的函数，它的作用是将数据包从发送队列中发送出去。函数接受一个pcap_t类型的数据包指针、一个pcap_send_queue类型的数据包发送队列以及一个int类型的同步标志。函数将这些参数用于设置发送队列的行为，并返回一个int类型的结果表示发送成功或失败。

另外，还定义了一个名为TcSetUserBuffer的函数，它的作用是设置用户数据缓冲区。函数接受一个pcap_t类型的数据包指针和一个char类型的数据缓冲区，并返回一个int类型的结果表示设置成功或失败。

最后，还定义了一个名为TcLiveDump的函数，它的作用是向文件中写入数据包，以进行实模式保存。函数接受一个pcap_t类型的数据包指针、一个char类型的数据文件名和一个int类型的最大数据包数量以及一个int类型的最大数据包数量。函数将这些参数用于将数据包写入文件，并返回一个int类型的结果表示成功或失败。


```cpp
static int TcOidSetRequest(pcap_t *p, bpf_u_int32 oid, const void *data, size_t *lenp);
static u_int TcSendqueueTransmit(pcap_t *p, pcap_send_queue *queue, int sync);
static int TcSetUserBuffer(pcap_t *p, int size);
static int TcLiveDump(pcap_t *p, char *filename, int maxsize, int maxpacks);
static int TcLiveDumpEnded(pcap_t *p, int sync);
static PAirpcapHandle TcGetAirPcapHandle(pcap_t *p);
#endif

#ifdef _WIN32
TC_FUNCTIONS g_TcFunctions =
{
	TC_API_UNLOADED, /* LoadStatus */
	NULL,  /* hTcApiDllHandle */
	NULL,  /* QueryPortList */
	NULL,  /* FreePortList */
	NULL,  /* StatusGetString */
	NULL,  /* PortGetName */
	NULL,  /* PortGetDescription */
	NULL,  /* InstanceOpenByName */
	NULL,  /* InstanceClose */
	NULL,  /* InstanceSetFeature */
	NULL,  /* InstanceQueryFeature */
	NULL,  /* InstanceReceivePackets */
	NULL,  /* InstanceGetReceiveWaitHandle */
	NULL,  /* InstanceTransmitPackets */
	NULL,  /* InstanceQueryStatistics */
	NULL,  /* PacketsBufferCreate */
	NULL,  /* PacketsBufferDestroy */
	NULL,  /* PacketsBufferQueryNextPacket */
	NULL,  /* PacketsBufferCommitNextPacket */
	NULL,  /* StatisticsDestroy */
	NULL,  /* StatisticsUpdate */
	NULL  /* StatisticsQueryValue */
};
```

这段代码是一个C函数指针数组，定义了多个TC函数及其返回值类型。

这个数组定义的函数都是用“#else”注释开头的，表示这些函数不是从标准库函数中定义的，而是由程序员自己定义的。具体来说，这些函数可能包括：

- TcQueryPortList：返回一个TC_QUERY_PORT_LIST类型的变量，表示查询当前系统中可用的物理端口列表。
- TcStatusGetString：返回一个TC_STATUS_GET_STRING类型的变量，表示获取当前TC实例的状态字符串。
- TcPortGetName：返回一个TC_PORT_GET_NAME类型的变量，表示获取指定端口的名。
- TcPortGetDescription：返回一个TC_PORT_GET_DESCRIPTION类型的变量，表示获取指定端口的描述字符串。
- TcInstanceOpenByName：返回一个TC_INSTANCE_OPEN_BY_NAME类型的变量，表示通过名称打开一个TC实例。
- TcInstanceClose：返回一个TC_INSTANCE_CLOSE类型的变量，表示关闭一个TC实例。
- TcInstanceSetFeature：返回一个TC_INSTANCE_SET_FEATURE类型的变量，表示设置一个TC实例的一个功能。
- TcInstanceQueryFeature：返回一个TC_INSTANCE_QUERY_FEATURE类型的变量，表示查询一个TC实例的一个功能。
- TcInstanceReceivePackets：返回一个TC_INSTANCE_RECEIVE_PACKETS类型的变量，表示接收数据包。

函数值类型说明：

- TcQueryPortList、TcStatusGetString、TcPortGetName、TcPortGetDescription、TcInstanceOpenByName、TcInstanceClose、TcInstanceSetFeature、TcInstanceQueryFeature、TcInstanceReceivePackets:
返回值类型为TC_QUERY_PORT_LIST、TC_STATUS_GET_STRING、TC_PORT_GET_NAME、TC_PORT_GET_DESCRIPTION、TC_INSTANCE_OPEN_BY_NAME、TC_INSTANCE_CLOSE、TC_INSTANCE_SET_FEATURE、TC_INSTANCE_QUERY_FEATURE、TC_INSTANCE_RECEIVE_PACKETS:
其中TC_QUERY_PORT_LIST表示查询当前系统中可用的物理端口列表，TC_PORT_GET_NAME表示获取指定端口的名，TC_PORT_GET_DESCRIPTION表示获取指定端口的描述字符串，TC_INSTANCE_OPEN_BY_NAME表示通过名称打开一个TC实例，TC_INSTANCE_CLOSE表示关闭一个TC实例，TC_INSTANCE_SET_FEATURE表示设置一个TC实例的一个功能，TC_INSTANCE_QUERY_FEATURE表示查询一个TC实例的一个功能，TC_INSTANCE_RECEIVE_PACKETS表示接收数据包。

- TcInstanceGetReceiveWaitHandle:
返回值类型为TC_INSTANCE_GET_RECEIVE_WAIT_HANDLE类型的变量，表示获取当前TC实例接收等待句柄。


```cpp
#else
TC_FUNCTIONS g_TcFunctions =
{
	TC_API_LOADED, /* LoadStatus */
	TcQueryPortList,
	TcFreePortList,
	TcStatusGetString,
	TcPortGetName,
	TcPortGetDescription,
	TcInstanceOpenByName,
	TcInstanceClose,
	TcInstanceSetFeature,
	TcInstanceQueryFeature,
	TcInstanceReceivePackets,
#ifdef _WIN32
	TcInstanceGetReceiveWaitHandle,
```

这段代码定义了一个名为"MAX_TC_PACKET_SIZE"的宏，其值为9500。这个宏可能会在代码中有一些重要的作用，但我不确定具体是用来做什么的。


```cpp
#endif
	TcInstanceTransmitPackets,
	TcInstanceQueryStatistics,
	TcPacketsBufferCreate,
	TcPacketsBufferDestroy,
	TcPacketsBufferQueryNextPacket,
	TcPacketsBufferCommitNextPacket,
	TcStatisticsDestroy,
	TcStatisticsUpdate,
	TcStatisticsQueryValue,
};
#endif

#define MAX_TC_PACKET_SIZE	9500

```

这段代码定义了两个头文件：PPI_PACKET_HEADER 和 PPI_FIELD_HEADER，用于定义 PPI（照片打印机接口）数据包装结构的头部和字段。

PPI_PACKET_HEADER 是数据包装结构的头部，定义了 PPI 版本的号（PPH_PH_FLAG_PADDING）和使用 PPI 协议的版本号（PPH_PH_VERSION）。同时，它还定义了 PPI 数据包装结构的几个字段，如 PphVersion、PphFlags 和 PphLength，以及 PpiDlt，用于指示数据包的类型。

PPI_FIELD_HEADER 是数据包装结构的字段头，定义了 PPI 数据包装结构的字段类型，如 PfhType 和 PfhLength，以及它们的字段长度。

这些头文件定义了 PPI 数据包装结构，用于在从设备传输到计算机时传输数据。


```cpp
#pragma pack(push, 1)

#define PPH_PH_FLAG_PADDING	((UCHAR)0x01)
#define PPH_PH_VERSION		((UCHAR)0x00)

typedef struct _PPI_PACKET_HEADER
{
	UCHAR	PphVersion;
	UCHAR	PphFlags;
	USHORT	PphLength;
	ULONG	PphDlt;
}
	PPI_PACKET_HEADER, *PPPI_PACKET_HEADER;

typedef struct _PPI_FIELD_HEADER
{
	USHORT PfhType;
	USHORT PfhLength;
}
	PPI_FIELD_HEADER, *PPPI_FIELD_HEADER;


```

这段代码定义了一些属于PPI（总线协议）定义的常量和类型。

1. `#define PPI_FIELD_TYPE_AGGREGATION_EXTENSION ((UCHAR)0x08)`定义了一个宏，名为`PPI_FIELD_TYPE_AGGREGATION_EXTENSION`，它的值为`0x08`。`(UCHAR)0x08`表示这是一个8位的无符号整数，等于802.3AT类型。

2. `typedef struct _PPI_FIELD_AGGREGATION_EXTENSION`定义了一个结构体，名为`PPI_FIELD_AGGREGATION_EXTENSION`，它的成员包括`InterfaceId`和`Flags`。

3. `PPI_FIELD_AGGREGATION_EXTENSION`，全名为`PPI_FIELD_AGGREGATION_EXTENSION`，是一个结构体，它的成员包括`Flags`和`Errors`。`Flags`是一个32位的无符号整数，用于标记802.3AT类型字段是否支持保留字段。`Errors`是一个32位的无符号整数，用于标记802.3AT类型字段中包含的错误计数器。

4. `PPI_FIELD_TYPE_802_3_EXTENSION`定义了一个宏，名为`PPI_FIELD_TYPE_802_3_EXTENSION`，它的值为`0x09`。

5. `PPI_FLD_802_3_EXT_FLAG_FCS_PRESENT`定义了一个宏，名为`PPI_FLD_802_3_EXT_FLAG_FCS_PRESENT`，它的值为`0x00000001`。

6. `typedef struct _PPI_FIELD_802_3_EXTENSION`定义了一个结构体，名为`PPI_FIELD_802_3_EXTENSION`，它的成员包括`Flags`和`Errors`。`Flags`是一个32位的无符号整数，用于标记802.3AT类型字段是否支持保留字段。`Errors`是一个32位的无符号整数，用于标记802.3AT类型字段中包含的错误计数器。


```cpp
#define		PPI_FIELD_TYPE_AGGREGATION_EXTENSION	((UCHAR)0x08)

typedef struct _PPI_FIELD_AGGREGATION_EXTENSION
{
	ULONG		InterfaceId;
}
	PPI_FIELD_AGGREGATION_EXTENSION, *PPPI_FIELD_AGGREGATION_EXTENSION;


#define		PPI_FIELD_TYPE_802_3_EXTENSION			((UCHAR)0x09)

#define PPI_FLD_802_3_EXT_FLAG_FCS_PRESENT			((ULONG)0x00000001)

typedef struct _PPI_FIELD_802_3_EXTENSION
{
	ULONG		Flags;
	ULONG		Errors;
}
	PPI_FIELD_802_3_EXTENSION, *PPPI_FIELD_802_3_EXTENSION;

```

这段代码定义了一个名为 struct PPI_HEADER 的结构体，该结构体包含八个字段：

1. PPI_PACKET_HEADER：表示数据包的头部信息。
2. PPI_FIELD_HEADER：表示数据字段的头部信息。
3. PPI_FIELD_AGGREGATION_EXTENSION：表示聚合数据字段的扩展信息。
4. PPI_FIELD_HEADER：表示点分3字段的头部信息。
5. PPI_FIELD_802_3_EXTENSION：表示802.3标准的扩展字段。

6. PPI_FIELD_802_4_EXTENSION：表示802.4标准的扩展字段。
7. PPI_FIELD_802_5_EXTENSION：表示802.5标准的扩展字段。
8. PPI_FIELD_802_6_EXTENSION：表示802.6标准的扩展字段。

PPI_HEADER 和 PPPI_HEADER 是该结构体的两个指针。

另外，#pragma pack(pop) 是出自 Microsoft 的 pragma 指令，用于移除 pack 指令中的前几个元素，以便编译器能够更方便地生成二进制代码。


```cpp
typedef struct _PPI_HEADER
{
	PPI_PACKET_HEADER PacketHeader;
	PPI_FIELD_HEADER  AggregationFieldHeader;
	PPI_FIELD_AGGREGATION_EXTENSION AggregationField;
	PPI_FIELD_HEADER  Dot3FieldHeader;
	PPI_FIELD_802_3_EXTENSION Dot3Field;
}
	PPI_HEADER, *PPPI_HEADER;
#pragma pack(pop)

#ifdef _WIN32
/*
 * NOTE: this function should be called by the pcap functions that can theoretically
 *       deal with the Tc library for the first time, namely listing the adapters and
 *       opening one. All the other ones (close, read, write, set parameters) work
 *       on an open instance of TC, so we do not care to call this function
 */
```

This function appears to check if the TcApi library is loaded successfully. It does this by checking if the function pointers for each function in the library (e.g. PortGetName, PortGetDescription, InstanceOpenByName, etc.) are NULL, and if the library as a whole is not loaded. If the library is loaded, the function returns the current status of the library (TC_API_LOADED). If not, it returns the value None.


```cpp
TC_API_LOAD_STATUS LoadTcFunctions(void)
{
	TC_API_LOAD_STATUS currentStatus;

	do
	{
		currentStatus = InterlockedCompareExchange((LONG*)&g_TcFunctions.LoadStatus, TC_API_LOADING, TC_API_UNLOADED);

		while(currentStatus == TC_API_LOADING)
		{
			currentStatus = InterlockedCompareExchange((LONG*)&g_TcFunctions.LoadStatus, TC_API_LOADING, TC_API_LOADING);
			Sleep(10);
		}

		/*
		 * at this point we are either in the LOADED state, unloaded state (i.e. we are the ones loading everything)
		 * or in cannot load
		 */
		if(currentStatus  == TC_API_LOADED)
		{
			return TC_API_LOADED;
		}

		if (currentStatus == TC_API_CANNOT_LOAD)
		{
			return TC_API_CANNOT_LOAD;
		}

		currentStatus = TC_API_CANNOT_LOAD;

		g_TcFunctions.hTcApiDllHandle = pcap_load_code("TcApi.dll");
		if (g_TcFunctions.hTcApiDllHandle == NULL)	break;

		g_TcFunctions.QueryPortList			= (TcFcnQueryPortList)			pcap_find_function(g_TcFunctions.hTcApiDllHandle, "TcQueryPortList");
		g_TcFunctions.FreePortList			= (TcFcnFreePortList)			pcap_find_function(g_TcFunctions.hTcApiDllHandle, "TcFreePortList");

		g_TcFunctions.StatusGetString			= (TcFcnStatusGetString)		pcap_find_function(g_TcFunctions.hTcApiDllHandle, "TcStatusGetString");

		g_TcFunctions.PortGetName			= (TcFcnPortGetName)			pcap_find_function(g_TcFunctions.hTcApiDllHandle, "TcPortGetName");
		g_TcFunctions.PortGetDescription		= (TcFcnPortGetDescription)		pcap_find_function(g_TcFunctions.hTcApiDllHandle, "TcPortGetDescription");

		g_TcFunctions.InstanceOpenByName		= (TcFcnInstanceOpenByName)		pcap_find_function(g_TcFunctions.hTcApiDllHandle, "TcInstanceOpenByName");
		g_TcFunctions.InstanceClose			= (TcFcnInstanceClose)			pcap_find_function(g_TcFunctions.hTcApiDllHandle, "TcInstanceClose");
		g_TcFunctions.InstanceSetFeature		= (TcFcnInstanceSetFeature)		pcap_find_function(g_TcFunctions.hTcApiDllHandle, "TcInstanceSetFeature");
		g_TcFunctions.InstanceQueryFeature		= (TcFcnInstanceQueryFeature)	pcap_find_function(g_TcFunctions.hTcApiDllHandle, "TcInstanceQueryFeature");
		g_TcFunctions.InstanceReceivePackets		= (TcFcnInstanceReceivePackets)	pcap_find_function(g_TcFunctions.hTcApiDllHandle, "TcInstanceReceivePackets");
		g_TcFunctions.InstanceGetReceiveWaitHandle	= (TcFcnInstanceGetReceiveWaitHandle)pcap_find_function(g_TcFunctions.hTcApiDllHandle, "TcInstanceGetReceiveWaitHandle");
		g_TcFunctions.InstanceTransmitPackets		= (TcFcnInstanceTransmitPackets)pcap_find_function(g_TcFunctions.hTcApiDllHandle, "TcInstanceTransmitPackets");
		g_TcFunctions.InstanceQueryStatistics		= (TcFcnInstanceQueryStatistics)pcap_find_function(g_TcFunctions.hTcApiDllHandle, "TcInstanceQueryStatistics");

		g_TcFunctions.PacketsBufferCreate		= (TcFcnPacketsBufferCreate)	pcap_find_function(g_TcFunctions.hTcApiDllHandle, "TcPacketsBufferCreate");
		g_TcFunctions.PacketsBufferDestroy		= (TcFcnPacketsBufferDestroy)	pcap_find_function(g_TcFunctions.hTcApiDllHandle, "TcPacketsBufferDestroy");
		g_TcFunctions.PacketsBufferQueryNextPacket	= (TcFcnPacketsBufferQueryNextPacket)pcap_find_function(g_TcFunctions.hTcApiDllHandle, "TcPacketsBufferQueryNextPacket");
		g_TcFunctions.PacketsBufferCommitNextPacket	= (TcFcnPacketsBufferCommitNextPacket)pcap_find_function(g_TcFunctions.hTcApiDllHandle, "TcPacketsBufferCommitNextPacket");

		g_TcFunctions.StatisticsDestroy			= (TcFcnStatisticsDestroy)		pcap_find_function(g_TcFunctions.hTcApiDllHandle, "TcStatisticsDestroy");
		g_TcFunctions.StatisticsUpdate			= (TcFcnStatisticsUpdate)		pcap_find_function(g_TcFunctions.hTcApiDllHandle, "TcStatisticsUpdate");
		g_TcFunctions.StatisticsQueryValue		= (TcFcnStatisticsQueryValue)	pcap_find_function(g_TcFunctions.hTcApiDllHandle, "TcStatisticsQueryValue");

		if (   g_TcFunctions.QueryPortList == NULL
			|| g_TcFunctions.FreePortList == NULL
			|| g_TcFunctions.StatusGetString == NULL
			|| g_TcFunctions.PortGetName == NULL
			|| g_TcFunctions.PortGetDescription == NULL
			|| g_TcFunctions.InstanceOpenByName == NULL
			|| g_TcFunctions.InstanceClose == NULL
			|| g_TcFunctions.InstanceSetFeature	 == NULL
			|| g_TcFunctions.InstanceQueryFeature == NULL
			|| g_TcFunctions.InstanceReceivePackets == NULL
			|| g_TcFunctions.InstanceGetReceiveWaitHandle == NULL
			|| g_TcFunctions.InstanceTransmitPackets == NULL
			|| g_TcFunctions.InstanceQueryStatistics == NULL
			|| g_TcFunctions.PacketsBufferCreate == NULL
			|| g_TcFunctions.PacketsBufferDestroy == NULL
			|| g_TcFunctions.PacketsBufferQueryNextPacket == NULL
			|| g_TcFunctions.PacketsBufferCommitNextPacket == NULL
			|| g_TcFunctions.StatisticsDestroy == NULL
			|| g_TcFunctions.StatisticsUpdate == NULL
			|| g_TcFunctions.StatisticsQueryValue == NULL
		)
		{
			break;
		}

		/*
		 * everything got loaded, yay!!
		 */
		currentStatus = TC_API_LOADED;
	}while(FALSE);

	if (currentStatus != TC_API_LOADED)
	{
		if (g_TcFunctions.hTcApiDllHandle != NULL)
		{
			FreeLibrary(g_TcFunctions.hTcApiDllHandle);
			g_TcFunctions.hTcApiDllHandle = NULL;
		}
	}

	InterlockedExchange((LONG*)&g_TcFunctions.LoadStatus, currentStatus);

	return currentStatus;
}
```

这段代码定义了一个名为"pcap_tc"的结构体，用于在捕捉到TurboCap设备上的数据包时保存接收到的数据。

在代码中，首先定义了一个名为"LoadTcFunctions"的函数，用于返回TC API加载的状态，该函数在函数声明后面被declared asstatic，意味着该函数可以被任何函数调用，并且是内联函数，不需要使用外部函数指针。函数内部没有做任何实际的逻辑处理，只是返回一个表示TC API已加载的常量值，这个常量值通常在应用程序中使用。

接着定义了一个名为"pcap_tc"的结构体，用于保存接收到的数据包，该结构体包含四个成员变量：TcInstance、TcPacketsBuffer、TcAcceptedCount和PpiPacket，分别表示设备实例、数据包缓冲区、接收到的数据包数量和标记的数据包 pointer。

最后，在代码的最后部分，使用宏定义定义了一个名为"TC_INSTANCE"的宏，表示TC设备的实例号，将其与pcap_tc结构体中的TcInstance成员变量进行匹配，并将匹配的结果保存到TcPacketsBuffer和TcAcceptedCount成员变量中。然后，使用一个未命名的u_char类型的变量PpiPacket，将其赋值为匹配到的TcPacketsBuffer中的数据，最后将该匹配到的数据作为PpiPacket的值返回。


```cpp
#else
// static linking
TC_API_LOAD_STATUS LoadTcFunctions(void)
{
	return TC_API_LOADED;
}
#endif

/*
 * Private data for capturing on TurboCap devices.
 */
struct pcap_tc {
	TC_INSTANCE TcInstance;
	TC_PACKETS_BUFFER TcPacketsBuffer;
	ULONG TcAcceptedCount;
	u_char *PpiPacket;
};

```

这段代码是一个用 TcFindAllDevs函数来枚举网络接口卡（device）的代码。它接受一个 pcap_if_list_t 类型的输入参数，一个字符指针 errbuf，和一个字符串指针不需要的。

函数内部首先定义了一个 do-while 循环，在每次循环中执行以下操作：

1.加载 TcFunctions 函数，这个函数在 TcFoundAllDevs函数中被调用，用于初始化负载库和统计当前接口的端口数量。

2.获取配置错误的错误码。

3.遍历接口列表中的所有端口，将每个端口转换为一系列参数，包括将端口转换为 PcapIf 结构，并将其添加到当前的设备列表中。

4.如果当前遍历完所有端口，释放并关闭配置错误的端口列表，设置为 null。

5.do-while 循环结束后，返回结果变量，通常是 0，表示成功枚举完所有网络接口。


```cpp
int
TcFindAllDevs(pcap_if_list_t *devlist, char *errbuf)
{
	TC_API_LOAD_STATUS loadStatus;
	ULONG numPorts;
	PTC_PORT pPorts = NULL;
	TC_STATUS status;
	int result = 0;
	pcap_if_t *dev;
	ULONG i;

	do
	{
		loadStatus = LoadTcFunctions();

		if (loadStatus != TC_API_LOADED)
		{
			result = 0;
			break;
		}

		/*
		 * enumerate the ports, and add them to the list
		 */
		status = g_TcFunctions.QueryPortList(&pPorts, &numPorts);

		if (status != TC_SUCCESS)
		{
			result = 0;
			break;
		}

		for (i = 0; i < numPorts; i++)
		{
			/*
			 * transform the port into an entry in the list
			 */
			dev = TcCreatePcapIfFromPort(pPorts[i]);

			if (dev != NULL)
				add_dev(devlist, dev->name, dev->flags, dev->description, errbuf);
		}

		if (numPorts > 0)
		{
			/*
			 * ignore the result here
			 */
			status = g_TcFunctions.FreePortList(pPorts);
		}

	}while(FALSE);

	return result;
}

```

这段代码是一个用C语言编写的 pcap_if_t 类型的函数，它的作用是返回一个指向名为 "TcCreatePcapIfFromPort" 的 pcap_if_t 类型的指针，该函数接收一个参数 "port"，它是一个整数类型的参数，用于指定要在哪个网络接口上创建新的数据包捕获器。

函数内部首先定义了两个字符指针 name 和 description，它们分别用于存储数据包捕获器名称和描述信息。接着定义了一个指向指针 newIf 的变量，该变量将用于存储创建一个新的数据包捕获器。

函数使用 malloc 函数为新创建的数据包捕获器分配内存，并初始化内存，包括设置数据包捕获器名称、描述信息和指针，以及设置其地址为 NULL 和 next 成员为 NULL，同时设置其 flags 成员为 0。

最后，函数返回新创建的数据包捕获器指针，如果分配内存失败或函数内部出现错误，则返回 NULL。


```cpp
static pcap_if_t* TcCreatePcapIfFromPort(TC_PORT port)
{
	CHAR *name;
	CHAR *description;
	pcap_if_t *newIf = NULL;

	newIf = (pcap_if_t*)malloc(sizeof(*newIf));
	if (newIf == NULL)
	{
		return NULL;
	}

	memset(newIf, 0, sizeof(*newIf));

	name = g_TcFunctions.PortGetName(port);
	description = g_TcFunctions.PortGetDescription(port);

	newIf->name = (char*)malloc(strlen(name) + 1);
	if (newIf->name == NULL)
	{
		free(newIf);
		return NULL;
	}

	newIf->description = (char*)malloc(strlen(description) + 1);
	if (newIf->description == NULL)
	{
		free(newIf->name);
		free(newIf);
		return NULL;
	}

	strcpy(newIf->name, name);
	strcpy(newIf->description, description);

	newIf->addresses = NULL;
	newIf->next = NULL;
	newIf->flags = 0;

	return newIf;

}

```

This code segment appears to be part of a larger program that manages the TurboCap protocol栈.

The given code is responsible for enabling and enabling different operations of the TurboCap protocol stack on a received花卉标本上的 TurboCap device.

The first if statement checks if there is any timeout specified for enabling reception. If there isn't, it defaults to the default value of 0. The second if statement checks if the timeout is set. If it is, the if statement checks if the timeout is negative or not and sets the timeout to the given value.

The third if statement checks if the turboCap device is a transmitter. If it is, the enableTransmission function is called. After that, the injectOperation function is set.

The fourth if statement checks if the timeout is set. If it is, the setTimeout function is called. After that, the readOperation function is set.

The following are the helper functions for installing thebpf-program, which is responsible for installing the program on the TurboCap device.

snprintf, AllCaps，砂梨nsprintf，悬崖下面吊金线，俾消毒，污水过滤，污水筛除， install_bpf_program，超大型人品，子乌冬，块状 Brook绝缘体表面涂层，可以在主控制器的 IterateTC，检查约束设置，安装大型时光延长，自主走量化器，四大皆言，DDD5DDD0123456789012345678，自主走量化器，四大皆言，更换基带，基带调谐，画中画，快中快，库库公路，理解了，笑里藏刀，认证成功，验证和设置，写，画外之音，意思，验证，取消，记录，设置，暂停，恢复，继续，读取，编辑，删除，导出，二次打包，刷新，合并，差错，查错，隔离，搜索，设置，快照，打印，邮件，截屏，内存管理，禁止上下文，彩云追风，城市精神，天高地厚，随机数生成器，SQLite,VCS，互联网，美剧，单元测试，切洋葱不眨眼，云雾大雾，隋唐盛世，后浪推雾，主流。


```cpp
static int
TcActivate(pcap_t *p)
{
	struct pcap_tc *pt = p->priv;
	TC_STATUS status;
	ULONG timeout;
	PPPI_HEADER pPpiHeader;

	if (p->opt.rfmon)
	{
		/*
		 * No monitor mode on Tc cards; they're Ethernet
		 * capture adapters.
		 */
		return PCAP_ERROR_RFMON_NOTSUP;
	}

	pt->PpiPacket = malloc(sizeof(PPI_HEADER) + MAX_TC_PACKET_SIZE);

	if (pt->PpiPacket == NULL)
	{
		snprintf(p->errbuf, PCAP_ERRBUF_SIZE, "Error allocating memory");
		return PCAP_ERROR;
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

	/*
	 * Initialize the PPI fixed fields
	 */
	pPpiHeader = (PPPI_HEADER)pt->PpiPacket;
	pPpiHeader->PacketHeader.PphDlt = DLT_EN10MB;
	pPpiHeader->PacketHeader.PphLength = sizeof(PPI_HEADER);
	pPpiHeader->PacketHeader.PphFlags = 0;
	pPpiHeader->PacketHeader.PphVersion = 0;

	pPpiHeader->AggregationFieldHeader.PfhLength = sizeof(PPI_FIELD_AGGREGATION_EXTENSION);
	pPpiHeader->AggregationFieldHeader.PfhType = PPI_FIELD_TYPE_AGGREGATION_EXTENSION;

	pPpiHeader->Dot3FieldHeader.PfhLength = sizeof(PPI_FIELD_802_3_EXTENSION);
	pPpiHeader->Dot3FieldHeader.PfhType = PPI_FIELD_TYPE_802_3_EXTENSION;

	status = g_TcFunctions.InstanceOpenByName(p->opt.device, &pt->TcInstance);

	if (status != TC_SUCCESS)
	{
		/* Adapter detected but we are not able to open it. Return failure. */
		snprintf(p->errbuf, PCAP_ERRBUF_SIZE, "Error opening TurboCap adapter: %s", g_TcFunctions.StatusGetString(status));
		return PCAP_ERROR;
	}

	p->linktype = DLT_EN10MB;
	p->dlt_list = (u_int *) malloc(sizeof(u_int) * 2);
	/*
	 * If that fails, just leave the list empty.
	 */
	if (p->dlt_list != NULL) {
		p->dlt_list[0] = DLT_EN10MB;
		p->dlt_list[1] = DLT_PPI;
		p->dlt_count = 2;
	}

	/*
	 * ignore promiscuous mode
	 * p->opt.promisc
	 */


	/*
	 * ignore all the buffer sizes
	 */

	/*
	 * enable reception
	 */
	status = g_TcFunctions.InstanceSetFeature(pt->TcInstance, TC_INST_FT_RX_STATUS, 1);

	if (status != TC_SUCCESS)
	{
		snprintf(p->errbuf, PCAP_ERRBUF_SIZE,"Error enabling reception on a TurboCap instance: %s", g_TcFunctions.StatusGetString(status));
		goto bad;
	}

	/*
	 * enable transmission
	 */
	status = g_TcFunctions.InstanceSetFeature(pt->TcInstance, TC_INST_FT_TX_STATUS, 1);
	/*
	 * Ignore the error here.
	 */

	p->inject_op = TcInject;
	/*
	 * if the timeout is -1, it means immediate return, no timeout
	 * if the timeout is 0, it means INFINITE
	 */

	if (p->opt.timeout == 0)
	{
		timeout = 0xFFFFFFFF;
	}
	else
	if (p->opt.timeout < 0)
	{
		/*
		 *  we insert a minimal timeout here
		 */
		timeout = 10;
	}
	else
	{
		timeout = p->opt.timeout;
	}

	status = g_TcFunctions.InstanceSetFeature(pt->TcInstance, TC_INST_FT_READ_TIMEOUT, timeout);

	if (status != TC_SUCCESS)
	{
		snprintf(p->errbuf, PCAP_ERRBUF_SIZE,"Error setting the read timeout a TurboCap instance: %s", g_TcFunctions.StatusGetString(status));
		goto bad;
	}

	p->read_op = TcRead;
	p->setfilter_op = install_bpf_program;
	p->setdirection_op = NULL;	/* Not implemented. */
	p->set_datalink_op = TcSetDatalink;
	p->getnonblock_op = TcGetNonBlock;
	p->setnonblock_op = TcSetNonBlock;
	p->stats_op = TcStats;
```

这段代码是一个 C 语言程序，它定义了一系列操作，用于在 Windows 系统上执行各种任务。以下是每个操作的作用：

```cpp
#ifdef _WIN32
   p->stats_ex_op = TcStatsEx;
   p->setbuff_op = TcSetBuff;
   p->setmode_op = TcSetMode;
   p->setmintocopy_op = TcSetMinToCopy;
   p->getevent_op = TcGetReceiveWaitHandle;
   p->oid_get_request_op = TcOidGetRequest;
   p->oid_set_request_op = TcOidSetRequest;
   p->sendqueue_transmit_op = TcSendqueueTransmit;
   p->setuserbuffer_op = TcSetUserBuffer;
   p->live_dump_op = TcLiveDump;
   p->live_dump_ended_op = TcLiveDumpEnded;
   p->get_airpcap_handle_op = TcGetAirPcapHandle;
#else
   p->selectable_fd = -1;
#endif
```

这些操作的具体实现可以参考 Microsoft 的 Windows 头文件，例如 `<windows.h>` 文件。


```cpp
#ifdef _WIN32
	p->stats_ex_op = TcStatsEx;
	p->setbuff_op = TcSetBuff;
	p->setmode_op = TcSetMode;
	p->setmintocopy_op = TcSetMinToCopy;
	p->getevent_op = TcGetReceiveWaitHandle;
	p->oid_get_request_op = TcOidGetRequest;
	p->oid_set_request_op = TcOidSetRequest;
	p->sendqueue_transmit_op = TcSendqueueTransmit;
	p->setuserbuffer_op = TcSetUserBuffer;
	p->live_dump_op = TcLiveDump;
	p->live_dump_ended_op = TcLiveDumpEnded;
	p->get_airpcap_handle_op = TcGetAirPcapHandle;
#else
	p->selectable_fd = -1;
```

This function appears to check if a given port is a TurboCap device and, if it is not, it returns NULL. If it is a TurboCap device, it returns a pointer to a g_TcFunctions.QueryPortList structure containing a list of all the ports on the device.


```cpp
#endif

	p->cleanup_op = TcCleanup;

	return 0;
bad:
	TcCleanup(p);
	return PCAP_ERROR;
}

pcap_t *
TcCreate(const char *device, char *ebuf, int *is_ours)
{
	ULONG numPorts;
	PTC_PORT pPorts = NULL;
	TC_STATUS status;
	int is_tc;
	ULONG i;
	pcap_t *p;

	if (LoadTcFunctions() != TC_API_LOADED)
	{
		/*
		 * XXX - report this as an error rather than as
		 * "not a TurboCap device"?
		 */
		*is_ours = 0;
		return NULL;
	}

	/*
	 * enumerate the ports, and add them to the list
	 */
	status = g_TcFunctions.QueryPortList(&pPorts, &numPorts);

	if (status != TC_SUCCESS)
	{
		/*
		 * XXX - report this as an error rather than as
		 * "not a TurboCap device"?
		 */
		*is_ours = 0;
		return NULL;
	}

	is_tc = FALSE;
	for (i = 0; i < numPorts; i++)
	{
		if (strcmp(g_TcFunctions.PortGetName(pPorts[i]), device) == 0)
		{
			is_tc = TRUE;
			break;
		}
	}

	if (numPorts > 0)
	{
		/*
		 * ignore the result here
		 */
		(void)g_TcFunctions.FreePortList(pPorts);
	}

	if (!is_tc)
	{
		*is_ours = 0;
		return NULL;
	}

	/* OK, it's probably ours. */
	*is_ours = 1;

	p = PCAP_CREATE_COMMON(ebuf, struct pcap_tc);
	if (p == NULL)
		return NULL;

	p->activate_op = TcActivate;
	/*
	 * Set these up front, so that, even if our client tries
	 * to set non-blocking mode before we're activated, or
	 * query the state of non-blocking mode, they get an error,
	 * rather than having the non-blocking mode option set
	 * for use later.
	 */
	p->getnonblock_op = TcGetNonBlock;
	p->setnonblock_op = TcSetNonBlock;
	return p;
}

```

这段代码定义了一个名为TcSetDatalink的静态函数，其作用是设置接收输入数据链路层（DLT）的类型。

在函数内部，我们首先检查传入的参数p和dlt是否在预定义的DLT值列表中。如果是，我们直接设置p->linktype为新的值，否则不做任何工作。

在函数外部，我们需要确保pcap_set_datalink()函数仍然可以正常工作，即使我们调用了内部函数。因此，我们需要定义一个实现类似功能的函数，内容与内部函数高度相似，但会执行一些额外的工作，以避免pcap_set_datalink()认为我们完全不支持设置链路层头部类型。


```cpp
static int TcSetDatalink(pcap_t *p, int dlt)
{
	/*
	 * We don't have to do any work here; pcap_set_datalink() checks
	 * whether the value is in the list of DLT_ values we
	 * supplied, so we don't have to, and, if it is valid, sets
	 * p->linktype to the new value; we don't have to do anything
	 * in hardware, we just use what's in p->linktype.
	 *
	 * We do have to have a routine, however, so that pcap_set_datalink()
	 * doesn't think we don't support setting the link-layer header
	 * type at all.
	 */
	return 0;
}

```



这段代码是用于管理TurboCap无线模块的函数。TcGetNonBlock和TcSetNonBlock函数用于设置或取消对非阻塞模式的设置。如果设置为非阻塞模式，则会输出一条错误消息并返回-1。TcCleanup函数用于在函数结束时清理TcGetNonBlock和TcSetNonBlock函数所设置的变量。


```cpp
static int TcGetNonBlock(pcap_t *p)
{
	snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
	    "Non-blocking mode isn't supported for TurboCap ports");
	return -1;
}

static int TcSetNonBlock(pcap_t *p, int nonblock)
{
	snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
	    "Non-blocking mode isn't supported for TurboCap ports");
	return -1;
}

static void TcCleanup(pcap_t *p)
{
	struct pcap_tc *pt = p->priv;

	if (pt->TcPacketsBuffer != NULL)
	{
		g_TcFunctions.PacketsBufferDestroy(pt->TcPacketsBuffer);
		pt->TcPacketsBuffer = NULL;
	}
	if (pt->TcInstance != NULL)
	{
		/*
		 * here we do not check for the error values
		 */
		g_TcFunctions.InstanceClose(pt->TcInstance);
		pt->TcInstance = NULL;
	}

	if (pt->PpiPacket != NULL)
	{
		free(pt->PpiPacket);
		pt->PpiPacket = NULL;
	}

	pcap_cleanup_live_common(p);
}

```

This function appears to be part of the `Wireshark` software, which is a network protocol analyzer. It appears to handle the transfer of packets over a network using the `g_TcFunctions` and `g_TcInstance` APIs.

The function takes in a pointer to a `PCAP` structure, which is a packet capture file. The function first creates a new packet buffer by calling `g_TcFunctions.PacketsBufferCreate` with the given buffer size and a pointer to the new buffer. It then packs the packet data into the buffer and commits the buffer to the next packet. Finally, it sends the packet to the `g_TcInstance` using `g_TcFunctions.InstanceTransmitPackets`, and repeats this process for all packets in the packet buffer.

If any errors occur, the function returns an error string and a `SNPRINTF` error code.


```cpp
/* Send a packet to the network */
static int TcInject(pcap_t *p, const void *buf, int size)
{
	struct pcap_tc *pt = p->priv;
	TC_STATUS status;
	TC_PACKETS_BUFFER buffer;
	TC_PACKET_HEADER header;

	if (size >= 0xFFFF)
	{
		snprintf(p->errbuf, PCAP_ERRBUF_SIZE, "send error: the TurboCap API does not support packets larger than 64k");
		return -1;
	}

	status = g_TcFunctions.PacketsBufferCreate(sizeof(TC_PACKET_HEADER) + TC_ALIGN_USHORT_TO_64BIT((USHORT)size), &buffer);

	if (status != TC_SUCCESS)
	{
		snprintf(p->errbuf, PCAP_ERRBUF_SIZE, "send error: TcPacketsBufferCreate failure: %s (%08x)", g_TcFunctions.StatusGetString(status), status);
		return -1;
	}

	/*
	 * we assume that the packet is without the checksum, as common with WinPcap
	 */
	memset(&header, 0, sizeof(header));

	header.Length = (USHORT)size;
	header.CapturedLength = header.Length;

	status = g_TcFunctions.PacketsBufferCommitNextPacket(buffer, &header, (PVOID)buf);

	if (status == TC_SUCCESS)
	{
		status = g_TcFunctions.InstanceTransmitPackets(pt->TcInstance, buffer);

		if (status != TC_SUCCESS)
		{
			snprintf(p->errbuf, PCAP_ERRBUF_SIZE, "send error: TcInstanceTransmitPackets failure: %s (%08x)", g_TcFunctions.StatusGetString(status), status);
		}
	}
	else
	{
		snprintf(p->errbuf, PCAP_ERRBUF_SIZE, "send error: TcPacketsBufferCommitNextPacket failure: %s (%08x)", g_TcFunctions.StatusGetString(status), status);
	}

	g_TcFunctions.PacketsBufferDestroy(buffer);

	if (status != TC_SUCCESS)
	{
		return -1;
	}
	else
	{
		return 0;
	}
}

```

0 65535
65535


```cpp
static int TcRead(pcap_t *p, int cnt, pcap_handler callback, u_char *user)
{
	struct pcap_tc *pt = p->priv;
	TC_STATUS status;
	int n = 0;

	/*
	 * Has "pcap_breakloop()" been called?
	 */
	if (p->break_loop)
	{
		/*
		 * Yes - clear the flag that indicates that it
		 * has, and return -2 to indicate that we were
		 * told to break out of the loop.
		 */
		p->break_loop = 0;
		return -2;
	}

	if (pt->TcPacketsBuffer == NULL)
	{
		status = g_TcFunctions.InstanceReceivePackets(pt->TcInstance, &pt->TcPacketsBuffer);
		if (status != TC_SUCCESS)
		{
			snprintf(p->errbuf, PCAP_ERRBUF_SIZE, "read error, TcInstanceReceivePackets failure: %s (%08x)", g_TcFunctions.StatusGetString(status), status);
			return -1;
		}
	}

	while (TRUE)
	{
		struct pcap_pkthdr hdr;
		TC_PACKET_HEADER tcHeader;
		PVOID data;
		ULONG filterResult;

		/*
		 * Has "pcap_breakloop()" been called?
		 * If so, return immediately - if we haven't read any
		 * packets, clear the flag and return -2 to indicate
		 * that we were told to break out of the loop, otherwise
		 * leave the flag set, so that the *next* call will break
		 * out of the loop without having read any packets, and
		 * return the number of packets we've processed so far.
		 */
		if (p->break_loop)
		{
			if (n == 0)
			{
				p->break_loop = 0;
				return -2;
			}
			else
			{
				return n;
			}
		}

		if (pt->TcPacketsBuffer == NULL)
		{
			break;
		}

		status = g_TcFunctions.PacketsBufferQueryNextPacket(pt->TcPacketsBuffer, &tcHeader, &data);

		if (status == TC_ERROR_END_OF_BUFFER)
		{
			g_TcFunctions.PacketsBufferDestroy(pt->TcPacketsBuffer);
			pt->TcPacketsBuffer = NULL;
			break;
		}

		if (status != TC_SUCCESS)
		{
			snprintf(p->errbuf, PCAP_ERRBUF_SIZE, "read error, TcPacketsBufferQueryNextPacket failure: %s (%08x)", g_TcFunctions.StatusGetString(status), status);
			return -1;
		}

		/* No underlying filtering system. We need to filter on our own */
		if (p->fcode.bf_insns)
		{
			filterResult = pcap_filter(p->fcode.bf_insns, data, tcHeader.Length, tcHeader.CapturedLength);

			if (filterResult == 0)
			{
				continue;
			}

			if (filterResult > tcHeader.CapturedLength)
			{
				filterResult = tcHeader.CapturedLength;
			}
		}
		else
		{
			filterResult = tcHeader.CapturedLength;
		}

		pt->TcAcceptedCount ++;

		hdr.ts.tv_sec = (bpf_u_int32)(tcHeader.Timestamp / (ULONGLONG)(1000  * 1000 * 1000));
		hdr.ts.tv_usec = (bpf_u_int32)((tcHeader.Timestamp % (ULONGLONG)(1000  * 1000 * 1000)) / 1000);

		if (p->linktype == DLT_EN10MB)
		{
			hdr.caplen = filterResult;
			hdr.len = tcHeader.Length;
			(*callback)(user, &hdr, data);
		}
		else
		{
			PPPI_HEADER pPpiHeader = (PPPI_HEADER)pt->PpiPacket;
			PVOID data2 = pPpiHeader + 1;

			pPpiHeader->AggregationField.InterfaceId = TC_PH_FLAGS_RX_PORT_ID(tcHeader.Flags);
			pPpiHeader->Dot3Field.Errors = tcHeader.Errors;
			if (tcHeader.Flags & TC_PH_FLAGS_CHECKSUM)
			{
				pPpiHeader->Dot3Field.Flags = PPI_FLD_802_3_EXT_FLAG_FCS_PRESENT;
			}
			else
			{
				pPpiHeader->Dot3Field.Flags = 0;
			}

			if (filterResult <= MAX_TC_PACKET_SIZE)
			{
				memcpy(data2, data, filterResult);
				hdr.caplen = sizeof(PPI_HEADER) + filterResult;
				hdr.len = sizeof(PPI_HEADER) + tcHeader.Length;
			}
			else
			{
				memcpy(data2, data, MAX_TC_PACKET_SIZE);
				hdr.caplen = sizeof(PPI_HEADER) + MAX_TC_PACKET_SIZE;
				hdr.len = sizeof(PPI_HEADER) + tcHeader.Length;
			}

			(*callback)(user, &hdr, pt->PpiPacket);

		}

		if (++n >= cnt && cnt > 0)
		{
			return n;
		}
	}

	return n;
}

```

This function appears to retrieve statistics about the number of packets received and dropped by the TurboCap network device. It does this by querying the device's TcInstanceQueryStatistics and TcStatisticsQueryValue functions.

The function takes two arguments: a pointer to a network packet capture (PCAP) structure, which will store the packet statistics, and the desired statistical result. The function returns either an error code or a pointer to the retrieved statistics value.

If the query is successful, the function retrieves the statistics value for the TotalRxPackets counter, which includes both the incoming and outgoing packets. If the result is not a valid counter, an error message issnprintf is called with the error code and status information.

If the query is successful, the function retrieves the statistics values for the Instance and DropPacket counters, which are specific to the TurboCap device. These values are stored in the PS_RECV and PS_IFDROP variables in the PCAP structure. If the retrieved value is not a valid counter, the function sets the corresponding PS_RECV and PS_IFDROP values to 0xFFFFFFFFF.

If the query encounters any errors, appropriate error messages aresnprintf and the PCAP structure is not modified.


```cpp
static int
TcStats(pcap_t *p, struct pcap_stat *ps)
{
	struct pcap_tc *pt = p->priv;
	TC_STATISTICS statistics;
	TC_STATUS status;
	ULONGLONG counter;
	struct pcap_stat s;

	status = g_TcFunctions.InstanceQueryStatistics(pt->TcInstance, &statistics);

	if (status != TC_SUCCESS)
	{
		snprintf(p->errbuf, PCAP_ERRBUF_SIZE, "TurboCap error in TcInstanceQueryStatistics: %s (%08x)", g_TcFunctions.StatusGetString(status), status);
		return -1;
	}

	memset(&s, 0, sizeof(s));

	status = g_TcFunctions.StatisticsQueryValue(statistics, TC_COUNTER_INSTANCE_TOTAL_RX_PACKETS, &counter);
	if (status != TC_SUCCESS)
	{
		snprintf(p->errbuf, PCAP_ERRBUF_SIZE, "TurboCap error in TcStatisticsQueryValue: %s (%08x)", g_TcFunctions.StatusGetString(status), status);
		return -1;
	}
	if (counter <= (ULONGLONG)0xFFFFFFFF)
	{
		s.ps_recv = (ULONG)counter;
	}
	else
	{
		s.ps_recv = 0xFFFFFFFF;
	}

	status = g_TcFunctions.StatisticsQueryValue(statistics, TC_COUNTER_INSTANCE_RX_DROPPED_PACKETS, &counter);
	if (status != TC_SUCCESS)
	{
		snprintf(p->errbuf, PCAP_ERRBUF_SIZE, "TurboCap error in TcStatisticsQueryValue: %s (%08x)", g_TcFunctions.StatusGetString(status), status);
		return -1;
	}
	if (counter <= (ULONGLONG)0xFFFFFFFF)
	{
		s.ps_ifdrop = (ULONG)counter;
		s.ps_drop = (ULONG)counter;
	}
	else
	{
		s.ps_ifdrop = 0xFFFFFFFF;
		s.ps_drop = 0xFFFFFFFF;
	}

```

This code appears to be a function used to retrieve statistics about the TCO (TurboCpe) system. It includes the following functions:

* g_TcFunctions.StatusGetString: This function retrieves the status of the TcFunctions system and returns it as a string.
* g_TcFunctions.StatisticsQueryValue: This function retrieves statistics values from the TcFunctions system and returns them as a variety of data types depending on the specific query.
* snprintf: This function is a part of the standard C library and is used to convert a variable-length buffer to a string.
* p: This variable appears to be used as an intermediate variable for the function calls.

The function takes a single parameter, `statistics`, which is a pointer to a `TcStatistics` structure. It then queries various statistics about the system, such as the total number of RX packets and the number of dropped packets. If any of the queries fail, it returns an error message and NULL. Otherwise, it populates the `p` struct with the retrieved data and returns it.


```cpp
#if defined(_WIN32) && defined(ENABLE_REMOTE)
	s.ps_capt = pt->TcAcceptedCount;
#endif
	*ps = s;

	return 0;
}


#ifdef _WIN32
static struct pcap_stat *
TcStatsEx(pcap_t *p, int *pcap_stat_size)
{
	struct pcap_tc *pt = p->priv;
	TC_STATISTICS statistics;
	TC_STATUS status;
	ULONGLONG counter;

	*pcap_stat_size = sizeof (p->stat);

	status = g_TcFunctions.InstanceQueryStatistics(pt->TcInstance, &statistics);

	if (status != TC_SUCCESS)
	{
		snprintf(p->errbuf, PCAP_ERRBUF_SIZE, "TurboCap error in TcInstanceQueryStatistics: %s (%08x)", g_TcFunctions.StatusGetString(status), status);
		return NULL;
	}

	memset(&p->stat, 0, sizeof(p->stat));

	status = g_TcFunctions.StatisticsQueryValue(statistics, TC_COUNTER_INSTANCE_TOTAL_RX_PACKETS, &counter);
	if (status != TC_SUCCESS)
	{
		snprintf(p->errbuf, PCAP_ERRBUF_SIZE, "TurboCap error in TcStatisticsQueryValue: %s (%08x)", g_TcFunctions.StatusGetString(status), status);
		return NULL;
	}
	if (counter <= (ULONGLONG)0xFFFFFFFF)
	{
		p->stat.ps_recv = (ULONG)counter;
	}
	else
	{
		p->stat.ps_recv = 0xFFFFFFFF;
	}

	status = g_TcFunctions.StatisticsQueryValue(statistics, TC_COUNTER_INSTANCE_RX_DROPPED_PACKETS, &counter);
	if (status != TC_SUCCESS)
	{
		snprintf(p->errbuf, PCAP_ERRBUF_SIZE, "TurboCap error in TcStatisticsQueryValue: %s (%08x)", g_TcFunctions.StatusGetString(status), status);
		return NULL;
	}
	if (counter <= (ULONGLONG)0xFFFFFFFF)
	{
		p->stat.ps_ifdrop = (ULONG)counter;
		p->stat.ps_drop = (ULONG)counter;
	}
	else
	{
		p->stat.ps_ifdrop = 0xFFFFFFFF;
		p->stat.ps_drop = 0xFFFFFFFF;
	}

```

这段代码是一个条件判断语句，它会根据两个条件判断是否需要做进一步的处理。如果两个条件都为真，那么执行内部的代码，否则不做任何处理。

具体来说，如果定义了 _WIN32 并且定义了 enable_remote，那么会执行 p->stat.ps_capt = pt->TcAcceptedCount; 这个代码，即将远程接受的数据计数到 p->stat.ps_capt 变量中。

如果只定义了 _WIN32 或者只定义了 enable_remote，那么不做任何处理，返回 p 指向的 pcap 结构体。


```cpp
#if defined(_WIN32) && defined(ENABLE_REMOTE)
	p->stat.ps_capt = pt->TcAcceptedCount;
#endif

	return &p->stat;
}

/* Set the dimension of the kernel-level capture buffer */
static int
TcSetBuff(pcap_t *p, int dim)
{
	/*
	 * XXX turbocap has an internal way of managing buffers.
	 * And at the moment it's not configurable, so we just
	 * silently ignore the request to set the buffer.
	 */
	return 0;
}

```

这段代码是tpcap库中的两个函数，分别为TcSetMode和TcSetMinToCopy。

TcSetMode函数的作用是，当调用者传入了无效的模式号后，返回一个负值，并打印错误消息。

TcSetMinToCopy函数的作用是，设置mintocopy成员变量的大小，如果设置的小于0，则会返回一个负值，并打印错误消息。否则，如果设置成功，则返回0。


```cpp
static int
TcSetMode(pcap_t *p, int mode)
{
	if (mode != MODE_CAPT)
	{
		snprintf(p->errbuf, PCAP_ERRBUF_SIZE, "Mode %d not supported by TurboCap devices. TurboCap only supports capture.", mode);
		return -1;
	}

	return 0;
}

static int
TcSetMinToCopy(pcap_t *p, int size)
{
	struct pcap_tc *pt = p->priv;
	TC_STATUS status;

	if (size < 0)
	{
		snprintf(p->errbuf, PCAP_ERRBUF_SIZE, "Mintocopy cannot be less than 0.");
		return -1;
	}

	status = g_TcFunctions.InstanceSetFeature(pt->TcInstance, TC_INST_FT_MINTOCOPY, (ULONG)size);

	if (status != TC_SUCCESS)
	{
		snprintf(p->errbuf, PCAP_ERRBUF_SIZE, "TurboCap error setting the mintocopy: %s (%08x)", g_TcFunctions.StatusGetString(status), status);
	}

	return 0;
}

```



这段代码定义了两个函数，作用如下：

1. TcGetReceiveWaitHandle(pcap_t *p)函数：
该函数返回一个硬件设备接收等待handle，并使用TcFunctions.InstanceGetReceiveWaitHandle函数获取。它将接收输入数据并将其存储在p->priv指向的struct pcap_tc结构中。

2. TcOidGetRequest(pcap_t *p, bpf_u_int32 oid, void *data, size_t *lenp)函数：
该函数尝试从TcDevice对象中获取与传入的oid相等的请求，并返回一个int类型的错误码。如果没有找到匹配的OID，函数将返回PCAP_ERROR。

注意，这两函数均使用g_TcFunctions.InstanceGetReceiveWaitHandle函数获取接收等待handle。


```cpp
static HANDLE
TcGetReceiveWaitHandle(pcap_t *p)
{
	struct pcap_tc *pt = p->priv;

	return g_TcFunctions.InstanceGetReceiveWaitHandle(pt->TcInstance);
}

static int
TcOidGetRequest(pcap_t *p, bpf_u_int32 oid _U_, void *data _U_, size_t *lenp _U_)
{
	snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
	    "An OID get request cannot be performed on a TurboCap device");
	return PCAP_ERROR;
}

```



这两段代码定义了两个函数：`TcOidSetRequest` 和 `TcSendqueueTransmit`。

`TcOidSetRequest`函数的作用是设置Tc设备接收到的OID。它接收三个参数：一个指向 `pcap_t` 结构物的指针 `p`，一个表示OID的 `bpf_u_int32` 类型的数据 `oid`，以及一个指向 `const void *data` 的指针 `data`。它的作用是检查是否有必要执行此操作，并输出一个字符串错误。

`TcSendqueueTransmit`函数的作用是将数据发送到Tc设备中的发送队列中。它接收四个参数：一个指向 `pcap_t` 结构物的指针 `p`，一个指向 `pcap_send_queue` 结构物的指针 `queue` ，一个表示同步操作的整数 `sync` ，以及一个指向 `const void *data` 的指针 `data`。它的作用是检查是否有必要执行此操作，并输出一个字符串错误。

这两个函数都是从 `tcpd` 库中定义的，因此它们应该在所有支持Tc设备的主机上可用，只要主机支持 `tcpd` 库。


```cpp
static int
TcOidSetRequest(pcap_t *p, bpf_u_int32 oid _U_, const void *data _U_,
    size_t *lenp _U_)
{
	snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
	    "An OID set request cannot be performed on a TurboCap device");
	return PCAP_ERROR;
}

static u_int
TcSendqueueTransmit(pcap_t *p, pcap_send_queue *queue _U_, int sync _U_)
{
	snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
	    "Packets cannot be bulk transmitted on a TurboCap device");
	return 0;
}

```



这两段代码定义了两个名为 `TcSetUserBuffer` 和 `TcLiveDump` 的函数，属于一个名为 `TcCap` 的类的成员函数。它们的函数名下面都有说明，其中第一个参数 `p` 是 `pcap_t` 类型的指针，代表需要修改的无线网络芯片的变量；第二个参数 `size` 表示用户缓冲区的最大长度，第三个参数是 `int` 类型，第四个参数是一个字符串，用于输出文件中包含的包的名称。

`TcSetUserBuffer` 函数的作用是设置芯片的参数 `errbuf` 中用户缓冲区的最大长度，并将错误信息字符串输出到芯片的错误缓冲区中。如果需要修改的芯片支持 `PCAP_ERRBUF_SIZE` 类型的参数，则需要将该参数设置为 `PCAP_ERRBUF_SIZE`。

`TcLiveDump` 函数的作用是在芯片的错误缓冲区中读取最大长度内的所有包，并将这些包的字节串输出到指定的文件中。如果需要修改的芯片不支持 `PCAP_ERRBUF_SIZE` 类型的参数，则需要将该函数调用中的最大长度修改为适合该芯片的最大长度。函数的第二个参数是一个字符串，用于指定输出文件的名称。函数的第三个参数是一个整数，表示需要下载的最大包数。


```cpp
static int
TcSetUserBuffer(pcap_t *p, int size _U_)
{
	snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
	    "The user buffer cannot be set on a TurboCap device");
	return -1;
}

static int
TcLiveDump(pcap_t *p, char *filename _U_, int maxsize _U_, int maxpacks _U_)
{
	snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
	    "Live packet dumping cannot be performed on a TurboCap device");
	return -1;
}

```



这段代码定义了两个函数，一个是`TcLiveDumpEnded`，另一个是`TcGetAirPcapHandle`。

`TcLiveDumpEnded`函数接收一个`pcap_t`类型的参数`p`，以及一个同步标志`sync`。函数的作用是输出一个错误信息，并返回一个负值。具体来说，函数通过`snprintf`函数将一个字符串打印到`p->errbuf`数组中，该字符串类似于以下内容：

```cpp
"Live packet dumping cannot be performed on a TurboCap device"
```

由于是在函数内部打印的，因此函数将返回一个负值，表示发生了一个错误。

`TcGetAirPcapHandle`函数返回一个`PAirpcapHandle`类型的指针`h`。函数的作用是获取一个TurboCap设备上的`PAirpcapHandle`。由于该函数没有返回任何值，因此它不会发生任何错误或者返回任何值。

这两个函数是在`tcpdump`库中使用的，用于在TurboCap设备上进行数据包抓取。`TcLiveDumpEnded`函数将错误信息打印到`errbuf`数组中，并返回一个负值，表示发生了一个错误。`TcGetAirPcapHandle`函数用于获取一个TurboCap设备上的`PAirpcapHandle`。


```cpp
static int
TcLiveDumpEnded(pcap_t *p, int sync _U_)
{
	snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
	    "Live packet dumping cannot be performed on a TurboCap device");
	return -1;
}

static PAirpcapHandle
TcGetAirPcapHandle(pcap_t *p _U_)
{
	return NULL;
}
#endif

```