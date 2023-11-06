# Nmap源码解析 16

# `libdnet-stripped/src/ip-cooked.c`

这段代码是一个C语言源文件，它定义了一个名为"ip-cooked.c"的文件。

该文件包含了以下内容：

1. 定义了一个名为"ip-cooked"的函数，但没有对其进行定义。

2. 定义了一个名为"_WIN32"的宏，它表示当前操作系统的平台为Windows。

3. 在#ifdef _WIN32和#else语句中，包含了一些C语言标准库的头文件，如"dnet_winconfig.h"和"config.h"。这些头文件可能是用于在Windows平台上使用某些网络库或配置文件。

4. 包含了一些注释，其中一些指出该文件是作为独立应用程序的一部分使用的，而其他则是描述该文件的作用或源代码的版权信息。


```cpp
/*
 * ip-cooked.c
 *
 * Copyright (c) 2001 Dug Song <dugsong@monkey.org>
 *
 * $Id: ip-cooked.c 547 2005-01-25 21:30:40Z dugsong $
 */

#ifdef _WIN32
#include "dnet_winconfig.h"
#else
#include "config.h"
#endif

#ifndef _WIN32
```

这段代码定义了一个名为 `ip_intf` 的结构体，包含了以太网接口的一些基本信息。

首先，`#include <netinet/in.h>` 和 `#include <unistd.h>` 是用于引入 IPv4 头文件和 Unix 头文件的头标头。

接着，`#include <errno.h>` 和 `#include <stdio.h>` 是用于引入错误信息处理头文件和标准输入输出库的头标头。

然后，`#include <stdlib.h>` 是用于引入标准库头文件的标识头。

接下来，定义了一系列名为 `eth_t`, `name`, `ha`, `pa`, `mtu`, `next` 的变量。其中，`eth_t` 是表示以太网接口类型的结构体变量，`name` 和 `pa` 是用于表示接口名称和物理地址的结构体变量，`mtu` 是用于表示接口的最大传输单元大小的整型变量。

接着，定义了一个名为 `ip_intf` 的结构体变量，并从 `LIST_ENTRY(ip_intf)` 函数中获取了一个下一个指针，用于将所有 IP 地址添加到链表中。

最后，通过 `ip_intf` 结构体变量的指针，添加了一系列 IP 地址到链表中，并将链表的头部指针指向下一个要添加的 IP 地址。


```cpp
#include <netinet/in.h>
#include <unistd.h>
#endif
#include <errno.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#include "dnet.h"
#include "queue.h"

struct ip_intf {
	eth_t			*eth;
	char			 name[INTF_NAME_LEN];
	struct addr		 ha;
	struct addr		 pa;
	int			 mtu;
	LIST_ENTRY(ip_intf)	 next;
};

```



该代码定义了一个名为 ip_handle 的结构体，用于表示网络接口 handle。该结构体包含以下字段：

- arp：一个指针，指向一个 arp_t 类型的数据结构，用于存储链路地址信息。
- intf：一个指针，指向一个 intf_t 类型的数据结构，用于存储输入/输出函数的配置信息。
- route：一个指针，指向一个 route_t 类型的数据结构，用于存储路由信息。
- fd：一个整数，用于存储文件描述符。
- sin：一个用于存储 sockaddr_in 结构体的变量。

该结构体还包含一个名为 ip_intf_list 的链表，用于存储输入/输出函数的配置信息。

该代码中定义了一个名为 _add_ip_intf 的函数，用于在链表的头部添加一个新的输入/输出函数的配置信息。该函数需要接受一个 ip_t 类型的数据结构和一个 void 类型的指针，用于存储输入/输出函数的配置信息。

在该函数中，首先检查输入/输出函数的类型是否为以太网(INTF_TYPE_ETH)，如果是，就继续下一步。然后检查输入/输出函数的地址类型是否为 IP 地址，如果是，就创建一个新的 ip_intf 数据结构，并将其添加到链表的头部。

最后，将创建的 ip_intf 数据结构返回，以便链表中有一个新的输入/输出函数的配置信息。


```cpp
struct ip_handle {
	arp_t			*arp;
	intf_t			*intf;
	route_t			*route;
	int			 fd;
	struct sockaddr_in	 sin;
	
	LIST_HEAD(, ip_intf)	 ip_intf_list;
};

static int
_add_ip_intf(const struct intf_entry *entry, void *arg)
{
	ip_t *ip = (ip_t *)arg;
	struct ip_intf *ipi;

	if (entry->intf_type == INTF_TYPE_ETH &&
	    (entry->intf_flags & INTF_FLAG_UP) != 0 &&
	    entry->intf_mtu >= ETH_LEN_MIN &&
	    entry->intf_addr.addr_type == ADDR_TYPE_IP &&
	    entry->intf_link_addr.addr_type == ADDR_TYPE_ETH) {
		
		if ((ipi = calloc(1, sizeof(*ipi))) == NULL)
			return (-1);
		
		strlcpy(ipi->name, entry->intf_name, sizeof(ipi->name));
		memcpy(&ipi->ha, &entry->intf_link_addr, sizeof(ipi->ha));
		memcpy(&ipi->pa, &entry->intf_addr, sizeof(ipi->pa));
		ipi->mtu = entry->intf_mtu;

		LIST_INSERT_HEAD(&ip->ip_intf_list, ipi, next);
	}
	return (0);
}

```

这段代码定义了一个名为ip_t的指针变量ip，并实现了ip_open函数来初始化ip的各个部分，包括IP地址、ARP地址、网络接口等。

ip_open函数首先检查ip分配是否成功，如果成功，则执行以下操作：

1. 创建一个名为ip的ip_t结构体变量，并将其初始化为NULL，以及一个指向ARP协议的函数的指针。
2. 打开一个TCP套接字，并将其与IP地址一起加入链表中。
3. 将链表中所有节点的IP地址存储在一个ip_intf_list数组中，并使用intf_loop函数将每个IP地址加入链表中。
4. 如果没有使用intf_loop函数，则函数将返回，否则将一直循环，直到链表中所有节点都被加入。
5. 如果所有节点都加入后仍然返回，则说明初始化失败，此时ip_t *ip将不再指向有效的内存，需要进行释放。

ip_t结构体中包含以下成员：

- ip：存储当前IP地址的指针。
- arp：存储当前IP地址的ARP地址的指针。
- intf：存储当前网络接口的指针。
- route：存储当前路由的指针。
- fd：存储当前socket的文件描述符。
- sin：存储当前IP地址的sin部分。

ip_open函数在调用其内部函数时，使用了const函数，以避免对传入参数的修改。ip_close函数在释放IP地址内存时，同样使用了const函数，以避免对传入参数的修改。


```cpp
ip_t *
ip_open(void)
{
	ip_t *ip;

	if ((ip = calloc(1, sizeof(*ip))) != NULL) {
		ip->fd = -1;
		
		if ((ip->arp = arp_open()) == NULL ||
		    (ip->intf = intf_open()) == NULL ||
		    (ip->route = route_open()) == NULL)
			return (ip_close(ip));
		
		if ((ip->fd = socket(AF_INET, SOCK_DGRAM, 0)) < 0)
			return (ip_close(ip));

		memset(&ip->sin, 0, sizeof(ip->sin));
		ip->sin.sin_family = AF_INET;
		ip->sin.sin_port = htons(666);
		
		LIST_INIT(&ip->ip_intf_list);

		if (intf_loop(ip->intf, _add_ip_intf, ip) != 0)
			return (ip_close(ip));
	}
	return (ip);
}

```

这段代码是一个名为`_lookup_ip_intf`的函数，它接收一个IP地址（`ip`）和一个目标IP地址（`dst`）。它的作用是查找一个给定IP地址的网际接口（`ip_t`）并返回其对应的`ip_intf`结构体指针，如果找不到该接口，则返回`NULL`。

函数内部首先将目标IP地址设置为`dst`，然后创建一个包含`ip_intf`结构体的链表，并将该链表作为参数传递给`connect`函数。如果`connect`函数返回一个负数，说明无法建立连接，函数将返回`NULL`。如果`connect`函数成功，它将返回一个指向链表头结构的指针。

接下来，函数使用`getsockname`函数获取给定IP地址的socket的源地址。如果`getsockname`函数返回一个负数，说明无法获取源地址，函数将返回`NULL`。

函数的`LIST_FOREACH`循环遍历链表中的所有`ip_intf`结构体，检查当前结构体对应的`pa`成员是否等于目标IP地址。如果是，函数将返回该结构体指针。如果当前结构体不是链表中的第一个，函数将删除该结构体并重新添加到链表中。

函数首先创建一个名为`ipi`的结构体，用于存储当前链表中的第一个`ip_intf`结构体。然后，函数遍历链表中的所有`ip_intf`结构体，检查它们对应的`pa`成员是否等于目标IP地址。如果是，函数将返回该结构体指针。如果当前结构体不是链表中的第一个，函数将删除该结构体并重新添加到链表中。


```cpp
static struct ip_intf *
_lookup_ip_intf(ip_t *ip, ip_addr_t dst)
{
	struct ip_intf *ipi;
	int n;

	ip->sin.sin_addr.s_addr = dst;
	n = sizeof(ip->sin);
	
	if (connect(ip->fd, (struct sockaddr *)&ip->sin, n) < 0)
		return (NULL);

	if (getsockname(ip->fd, (struct sockaddr *)&ip->sin, &n) < 0)
		return (NULL);

	LIST_FOREACH(ipi, &ip->ip_intf_list, next) {
		if (ipi->pa.addr_ip == ip->sin.sin_addr.s_addr) {
			if (ipi->eth == NULL) {
				if ((ipi->eth = eth_open(ipi->name)) == NULL)
					return (NULL);
			}
			if (ipi != LIST_FIRST(&ip->ip_intf_list)) {
				LIST_REMOVE(ipi, next);
				LIST_INSERT_HEAD(&ip->ip_intf_list, ipi, next);
			}
			return (ipi);
		}
	}
	return (NULL);
}

```

这段代码是一个名为 `_request_arp` 的函数，属于 `struct ip_intf` 和 `struct addr` 类型的结构体。它的作用是向目标 IP 地址发送一个 ARP 请求。

以下是具体的解释：

1. 函数开始时，定义了一个名为 `frame` 的 21 字节缓冲区，用于存放发送给目标设备的 ARP 请求数据。
2. 接下来，定义了一个名为 `ipi` 的 `struct ip_intf` 类型的变量，以及一个名为 `dst` 的 `struct addr` 类型的变量，用于存储目标设备的 IP 地址。
3. 在 `_request_arp` 函数内部，调用了 `eth_pack_hdr` 函数和 `arp_pack_hdr_ethip` 函数。
4. `eth_pack_hdr` 函数接收一个以太网数据包头，将其发送到 `ipi->eth` 接口。这里的数据包类型是 `ETH_ADDR_BROADCAST`，表示发送到所有网络接口。
5. `arp_pack_hdr_ethip` 函数接收一个 ARP 请求数据包，将其发送到 `ipi->pa.addr_ip` 和 `dst->addr_ip` 的 IP 地址。这里的数据包类型是 `ARP_OP_REQUEST`，表示发送一个 ARP 请求。
6. 调用 `eth_send` 函数，将 `frame` 中的数据包发送到 `ipi->eth` 接口。

由于 `arp_pack_hdr_ethip` 函数中需要传递一个 `ARP_HDR` 数据包头，而 `eth_pack_hdr` 函数需要知道要发送的数据包类型，因此需要在 `arp_pack_hdr_ethip` 函数和 `eth_pack_hdr` 函数之间添加一个中转步骤，将数据包转换为正确的格式，以便可以发送。


```cpp
static void
_request_arp(struct ip_intf *ipi, struct addr *dst)
{
	u_char frame[ETH_HDR_LEN + ARP_HDR_LEN + ARP_ETHIP_LEN];

	eth_pack_hdr(frame, ETH_ADDR_BROADCAST, ipi->ha.addr_eth,
	    ETH_TYPE_ARP);
	arp_pack_hdr_ethip(frame + ETH_HDR_LEN, ARP_OP_REQUEST,
	    ipi->ha.addr_eth, ipi->pa.addr_ip, ETH_ADDR_BROADCAST,
	    dst->addr_ip);

	eth_send(ipi->eth, frame, sizeof(frame));
}

ssize_t
```

这段代码是一个名为`ipi_process_arg`的函数，它是ARPigra库中的一个函数，负责处理输入数据中的IP数据。

首先，该函数接收一个IP头和一个ARP消息，并将IP头中的地址转换为ARP消息中的地址。然后，该函数检查输入的ARP消息是否小于目标ARP消息，如果是，则函数将返回IP头中的数据，否则将返回ARP消息中的数据。

在该函数中，首先定义了一个IP头变量`iph`，以及一个IP数据变量`ip_data`。然后，定义了一个变量`start`和一个变量`end`，用于存储IP数据和ARP消息中的地址。

接下来，函数使用一个循环从IP数据中读取数据，并计算IP头的校验和。然后，将IP头中的数据存储在`ip_data`中，并更新IP头的`ip_len`字段。最后，将IP头中的数据发送到以太网，如果发送失败，则函数返回-1。

总的来说，该函数的作用是将输入的ARP消息转换为IP数据，以便在以太网中传输。


```cpp
ip_send(ip_t *ip, const void *buf, size_t len)
{
	struct ip_hdr *iph;
	struct ip_intf *ipi;
	struct arp_entry arpent;
	struct route_entry rtent;
	u_char frame[ETH_LEN_MAX];
	int i, usec;

	iph = (struct ip_hdr *)buf;
	
	if ((ipi = _lookup_ip_intf(ip, iph->ip_dst)) == NULL) {
		errno = EHOSTUNREACH;
		return (-1);
	}
	arpent.arp_pa.addr_type = ADDR_TYPE_IP;
	arpent.arp_pa.addr_bits = IP_ADDR_BITS;
	arpent.arp_pa.addr_ip = iph->ip_dst;
	memcpy(&rtent.route_dst, &arpent.arp_pa, sizeof(rtent.route_dst));

	for (i = 0, usec = 10; i < 3; i++, usec *= 100) {
		if (arp_get(ip->arp, &arpent) == 0)
			break;
		
		if (route_get(ip->route, &rtent) == 0 &&
		    rtent.route_gw.addr_ip != ipi->pa.addr_ip) {
			memcpy(&arpent.arp_pa, &rtent.route_gw,
			    sizeof(arpent.arp_pa));
			if (arp_get(ip->arp, &arpent) == 0)
				break;
		}
		_request_arp(ipi, &arpent.arp_pa);

		usleep(usec);
	}
	if (i == 3)
		memset(&arpent.arp_ha.addr_eth, 0xff, ETH_ADDR_LEN);
	
	eth_pack_hdr(frame, arpent.arp_ha.addr_eth,
	    ipi->ha.addr_eth, ETH_TYPE_IP);

	if (len > ipi->mtu) {
		u_char *p, *start, *end, *ip_data;
		int ip_hl, fraglen;
		
		ip_hl = iph->ip_hl << 2;
		fraglen = ipi->mtu - ip_hl;

		iph = (struct ip_hdr *)(frame + ETH_HDR_LEN);
		memcpy(iph, buf, ip_hl);
		ip_data = (u_char *)iph + ip_hl;

		start = (u_char *)buf + ip_hl;
		end = (u_char *)buf + len;
		
		for (p = start; p < end; ) {
			memcpy(ip_data, p, fraglen);
			
			iph->ip_len = htons(ip_hl + fraglen);
			iph->ip_off = htons(((p + fraglen < end) ? IP_MF : 0) |
			    ((p - start) >> 3));

			ip_checksum(iph, ip_hl + fraglen);

			i = ETH_HDR_LEN + ip_hl + fraglen;
			if (eth_send(ipi->eth, frame, i) != i)
				return (-1);
			p += fraglen;
			if (end - p < fraglen)
				fraglen = end - p;
		}
		return (len);
	}
	memcpy(frame + ETH_HDR_LEN, buf, len);
	i = ETH_HDR_LEN + len;
	if (eth_send(ipi->eth, frame, i) != i)
		return (-1);
	
	return (len);
}

```

这段代码是一个用于关闭IP地址为ip的结构的指针ip_t的函数。它实现了一系列关闭操作，如关闭以太网接口eth、关闭套接字fd、关闭路由协议route、关闭桥接协议intf以及关闭ARP协议arp。

具体来说，代码首先检查ip是否为空。如果是，则执行以下操作：

1. 从ip_intf列表中查找ipi，并逐个遍历ipi所指向的以太网接口。
2. 如果找到了以太网接口，则调用eth_close函数关闭它。
3. 如果关闭成功，则执行以下操作：

a. 如果ip->fd大于0，则调用close函数关闭它。

b. 如果ip->route不为空，则调用route_close函数关闭它。

c. 如果ip->intf不为空，则调用intf_close函数关闭它。

d. 如果ip->arp不为空，则调用arp_close函数关闭它。

4. 最后，如果所有关闭操作都成功，则返回一个空指针。

如果ip不是空，则会执行以下操作：

1. 从ip_intf列表中查找ipi，并逐个遍历ipi所指向的以太网接口。
2. 如果找到了以太网接口，则调用eth_close函数关闭它。
3. 如果关闭成功，则执行以下操作：

a. 如果ip->fd大于0，则调用close函数关闭它。

b. 如果ip->route不为空，则调用route_close函数关闭它。

c. 如果ip->intf不为空，则调用intf_close函数关闭它。

d. 如果ip->arp不为空，则调用arp_close函数关闭它。
4. 如果关闭操作中有任何一个失败，则会返回一个空指针。


```cpp
ip_t *
ip_close(ip_t *ip)
{
	struct ip_intf *ipi, *nxt;

	if (ip != NULL) {
		for (ipi = LIST_FIRST(&ip->ip_intf_list);
		    ipi != LIST_END(&ip->ip_intf_list); ipi = nxt) {
			nxt = LIST_NEXT(ipi, next);
			if (ipi->eth != NULL)
				eth_close(ipi->eth);
			free(ipi);
		}
		if (ip->fd >= 0)
			close(ip->fd);
		if (ip->route != NULL)
			route_close(ip->route);
		if (ip->intf != NULL)
			intf_close(ip->intf);
		if (ip->arp != NULL)
			arp_close(ip->arp);
		free(ip);
	}
	return (NULL);
}

```

# `libdnet-stripped/src/ip-util.c`

这段代码是一个C语言的函数，名为“ip-util.c”。它可能是用于管理IP地址的工具函数。但是，出于某种原因，该函数没有定义完整。所以，我们无法了解具体的功能。建议您提供更多上下文信息，以便我们更好地解释该函数。


```cpp
/*
 * ip-util.c
 *
 * Copyright (c) 2002 Dug Song <dugsong@monkey.org>
 *
 * $Id: ip-util.c 595 2005-02-17 02:55:56Z dugsong $
 */

#ifdef _WIN32
#include "dnet_winconfig.h"
#else
#include "config.h"
#endif

#include <errno.h>
```

这段代码是一个网络协议头，其中包含了头部长度为4的值。通过引入头文件stdlib.h和string.h，我们可以使用这些头文件中的函数和宏。

接下来定义了一个名为_crc32c的函数，该函数接受一个长度为len的字节数组buf，并返回一个无符号long类型的值。函数内部使用了CRC32C函数，这个函数实现了CRC-32C算法。函数实现了一个循环，对传入的每个字节进行CRC32C计算，然后将结果异或一个校验位，最后得到结果。

接下来在头文件中引入了两个头文件：dnet.h和crc32ct.h，它们可能包含了一些网络协议头和CRC32C算法的实现。

总的来说，这段代码是一个网络协议头，其中包含了一个CRC-32C算法的实现。


```cpp
#include <stdlib.h>
#include <string.h>

#include "dnet.h"
#include "crc32ct.h"

/* CRC-32C (Castagnoli). Public domain. */
static unsigned long
_crc32c(unsigned char *buf, int len)
{
	int i;
	unsigned long crc32 = ~0L;
	unsigned long result;
	unsigned char byte0, byte1, byte2, byte3;

	for (i = 0; i < len; i++) {
		CRC32C(crc32, buf[i]);
	}

	result = ~crc32;

	byte0 =  result        & 0xff;
	byte1 = (result >>  8) & 0xff;
	byte2 = (result >> 16) & 0xff;
	byte3 = (result >> 24) & 0xff;
	crc32 = ((byte0 << 24) | (byte1 << 16) | (byte2 <<  8) | byte3);
	return crc32;
}

```

This is a function that modifies the contents of a TCP header and an IP header before sending them over a network. It takes in a TCP header pointer (ip), a raw IP header pointer (buf), and an optional buffer (opt).

The function first calculates the length of the optional buffer by subtracting the length of the IP header from the length of the TCP header, and then appends any additional data to the end of the TCP header to accommodate the IP header.

Next, the function checks that the length of the optional buffer is not too large and sets the padding accordingly. If the buffer is too long, an error is returned.

Finally, the function extracts the IP header from the TCP header and sets the IP protocol htons and the IP length to handle endianness. The function returns the length of the modified TCP header.


```cpp
ssize_t
ip_add_option(void *buf, size_t len, int proto,
    const void *optbuf, size_t optlen)
{
	struct ip_hdr *ip;
	struct tcp_hdr *tcp = NULL;
	u_char *p;
	int hl, datalen, padlen;
	
	if (proto != IP_PROTO_IP && proto != IP_PROTO_TCP) {
		errno = EINVAL;
		return (-1);
	}
	ip = (struct ip_hdr *)buf;
	hl = ip->ip_hl << 2;
	p = (u_char *)buf + hl;
	
	if (proto == IP_PROTO_TCP) {
		tcp = (struct tcp_hdr *)p;
		hl = tcp->th_off << 2;
		p = (u_char *)tcp + hl;
	}
	datalen = (int) (ntohs(ip->ip_len) - (p - (u_char *)buf));
	
	/* Compute padding to next word boundary. */
	if ((padlen = 4 - (optlen % 4)) == 4)
		padlen = 0;

	/* XXX - IP_HDR_LEN_MAX == TCP_HDR_LEN_MAX */
	if (hl + optlen + padlen > IP_HDR_LEN_MAX ||
	    ntohs(ip->ip_len) + optlen + padlen > len) {
		errno = EINVAL;
		return (-1);
	}
	/* XXX - IP_OPT_TYPEONLY() == TCP_OPT_TYPEONLY */
	if (IP_OPT_TYPEONLY(((struct ip_opt *)optbuf)->opt_type))
		optlen = 1;
	
	/* Shift any existing data. */
	if (datalen) {
		memmove(p + optlen + padlen, p, datalen);
	}
	/* XXX - IP_OPT_NOP == TCP_OPT_NOP */
	if (padlen) {
		memset(p, IP_OPT_NOP, padlen);
		p += padlen;
	}
	memmove(p, optbuf, optlen);
	p += optlen;
	optlen += padlen;
	
	if (proto == IP_PROTO_IP)
		ip->ip_hl = (int) ((p - (u_char *)ip) >> 2);
	else if (proto == IP_PROTO_TCP)
		tcp->th_off = (int) ((p - (u_char *)tcp) >> 2);

	ip->ip_len = htons((u_short) (ntohs(ip->ip_len) + optlen));
	
	return (ssize_t)(optlen);
}

```

This code is a Linux kernel module that implements the dynamic allocation of IP addresses. It is written in C and assumes that it is called `ip_free` by calling system calls.

The code defines several functions for working with IP addresses, including:

* `ip_init -- initialize the IP address cache`
* `ip_end -- end the IP address cache`
* `ip_init_noatof -- initialize the IP address cache and set the address to the default global address`
* `ip_end_noatof -- end the IP address cache and remove the default address`
* `ip_lookup -- look up the IP address in the IP address cache`
* `ip_add -- add a new IP address to the IP address cache`
* `ip_del -- delete the IP address from the IP address cache`
* `ip_get -- get the IP address from the IP address cache`
* `ip_set -- set the IP address from the IP address cache`
* `ip_v4l -- convert IPv4 address to aino-compatible format`
* `ip_v6l -- convert IPv6 address to aino-compatible format`

The `ip_init` function initializes the IP address cache by creating a queue of IP addresses and setting the current IP address to the default global address.

The `ip_end` function ends the IP address cache by returning control to the calling system call.

The `ip_init_noatof` function initializes the IP address cache and sets the current IP address to the default global address.

The `ip_end_noatof` function ends the IP address cache and removes the default address.

The `ip_lookup` function looks up the IP address in the IP address cache and returns it, if found.

The `ip_add` function adds a new IP address to the IP address cache.

The `ip_del` function deletes the IP address from the IP address cache.

The `ip_get` function gets the IP address from the IP address cache and returns it.

The `ip_set` function sets the IP address from the IP address cache and returns a success or failure status.

The `ip_v4l` function converts an IPv4 address to aino-compatible format.

The `ip_v6l` function converts an IPv6 address to aino-compatible format.


```cpp
void
ip_checksum(void *buf, size_t len)
{
	struct ip_hdr *ip;
	int hl, off, sum;

	if (len < IP_HDR_LEN)
		return;
	
	ip = (struct ip_hdr *)buf;
	hl = ip->ip_hl << 2;
	ip->ip_sum = 0;
	sum = ip_cksum_add(ip, hl, 0);
	ip->ip_sum = ip_cksum_carry(sum);

	off = htons(ip->ip_off);
	
	if ((off & IP_OFFMASK) != 0 || (off & IP_MF) != 0)
		return;
	
	len -= hl;
	
	if (ip->ip_p == IP_PROTO_TCP) {
		struct tcp_hdr *tcp = (struct tcp_hdr *)((u_char *)ip + hl);
		
		if (len >= TCP_HDR_LEN) {
			tcp->th_sum = 0;
			sum = ip_cksum_add(tcp, len, 0) +
			    htons((u_short)(ip->ip_p + len));
			sum = ip_cksum_add(&ip->ip_src, 8, sum);
			tcp->th_sum = ip_cksum_carry(sum);
		}
	} else if (ip->ip_p == IP_PROTO_UDP) {
		struct udp_hdr *udp = (struct udp_hdr *)((u_char *)ip + hl);

		if (len >= UDP_HDR_LEN) {
			udp->uh_sum = 0;
			sum = ip_cksum_add(udp, len, 0) +
			    htons((u_short)(ip->ip_p + len));
			sum = ip_cksum_add(&ip->ip_src, 8, sum);
			udp->uh_sum = ip_cksum_carry(sum);
			if (!udp->uh_sum)
				udp->uh_sum = 0xffff;	/* RFC 768 */
		}
	} else if (ip->ip_p == IP_PROTO_SCTP) {
		struct sctp_hdr *sctp = (struct sctp_hdr *)((u_char *)ip + hl);

		if (len >= SCTP_HDR_LEN) {
			sctp->sh_sum = 0;
			sctp->sh_sum = htonl(_crc32c((u_char *)sctp, len));
		}
	} else if (ip->ip_p == IP_PROTO_ICMP || ip->ip_p == IP_PROTO_IGMP) {
		struct icmp_hdr *icmp = (struct icmp_hdr *)((u_char *)ip + hl);
		
		if (len >= ICMP_HDR_LEN) {
			icmp->icmp_cksum = 0;
			sum = ip_cksum_add(icmp, len, 0);
			icmp->icmp_cksum = ip_cksum_carry(sum);
		}
	}
}

```

这段代码是一个名为`ip_cksum_add`的函数，它的作用是计算一个IP数据包的 checksum。

数据包的长度（len）首先被除以2，得到一个整数`sn`，然后根据 Duff 的设备算法，在16个不同的状态下进行循环，对每个状态计算一个 16 字节的转义字符（即 ASCII 码），然后将这些转义字符的 ASCII 码值加到 checksum 上。

不过，如果数据包长度不是16的整数倍，那么最后的 checsum 还会加上数据包长度（不包括末尾的16个字节）的 ASCII 码。


```cpp
int
ip_cksum_add(const void *buf, size_t len, int cksum)
{
	uint16_t *sp = (uint16_t *)buf;
	int n, sn;
	
	sn = (int) len / 2;
	n = (sn + 15) / 16;

	/* XXX - unroll loop using Duff's device. */
	switch (sn % 16) {
	case 0:	do {
		cksum += *sp++;
	case 15:
		cksum += *sp++;
	case 14:
		cksum += *sp++;
	case 13:
		cksum += *sp++;
	case 12:
		cksum += *sp++;
	case 11:
		cksum += *sp++;
	case 10:
		cksum += *sp++;
	case 9:
		cksum += *sp++;
	case 8:
		cksum += *sp++;
	case 7:
		cksum += *sp++;
	case 6:
		cksum += *sp++;
	case 5:
		cksum += *sp++;
	case 4:
		cksum += *sp++;
	case 3:
		cksum += *sp++;
	case 2:
		cksum += *sp++;
	case 1:
		cksum += *sp++;
		} while (--n > 0);
	}
	if (len & 1)
		cksum += htons(*(u_char *)sp << 8);

	return (cksum);
}

```

# `libdnet-stripped/src/ip-win32.c`

这段代码是一个C语言源文件，它定义了一个名为“ip-win32.c”的文件。这个文件的作用是包含一个名为“ip-win32.h”的头文件，并且在其中的某个位置定义了一个名为“ip_win32”的函数。

通过检查编译器是否支持“_WIN32”头文件，代码可以知道是否需要包含“dnet_winconfig.h”头文件。否则，使用“config.h”头文件作为替代。

接着，代码通过包含“ws2tcpip.h”头文件，表明要使用Windows的IP协议套件。

最后，代码定义了一个名为“ip_win32”的函数，但没有对其进行定义和使用，因此在代码中没有对其进行定义的函数体。


```cpp
/*
 * ip-win32.c
 *
 * Copyright (c) 2002 Dug Song <dugsong@monkey.org>
 *
 * $Id: ip-win32.c 547 2005-01-25 21:30:40Z dugsong $
 */

#ifdef _WIN32
#include "dnet_winconfig.h"
#else
#include "config.h"
#endif

#include <ws2tcpip.h>

```

这段代码是一个用于创建IP头结构的函数。它接收一个IP地址作为参数，然后使用WSAStartup函数初始化套接字，创建IP套接字，并设置套接字以监听IP请求。接下来，它创建一个IP头结构变量ip，并设置其参数。最后，它通过ip_close函数返回该IP头结构变量。


```cpp
#include <errno.h>
#include <stdlib.h>

#include "dnet.h"

struct ip_handle {
	WSADATA			wsdata;
	SOCKET			fd;
	struct sockaddr_in	sin;
};

ip_t *
ip_open(void)
{
	BOOL on;
	ip_t *ip;

	if ((ip = calloc(1, sizeof(*ip))) != NULL) {
		if (WSAStartup(MAKEWORD(2, 2), &ip->wsdata) != 0) {
			free(ip);
			return (NULL);
		}
		if ((ip->fd = socket(AF_INET, SOCK_RAW, IPPROTO_RAW)) ==
		    INVALID_SOCKET)
			return (ip_close(ip));
		
		on = TRUE;
		if (setsockopt(ip->fd, IPPROTO_IP, IP_HDRINCL,
			(const char *)&on, sizeof(on)) == SOCKET_ERROR) {
			SetLastError(ERROR_NETWORK_ACCESS_DENIED);
			return (ip_close(ip));
		}
		ip->sin.sin_family = AF_INET;
		ip->sin.sin_port = htons(666);
	}
	return (ip);
}

```

这段代码是一个用于向IP套接字发送数据的小头函数。具体来说，它将一个IP套接字指针ip和一个字符缓冲区buf中的数据长度len传递给ip_send函数。ip_send函数的作用是将buf中的数据发送到ip的IP地址，然后返回目标IP套接字到len字节长度的数据被发送的返回值。如果发送失败，函数将返回-1。

ip_send函数的结构如下：

```cpp
ssize_t ip_send(ip_t *ip, const void *buf, size_t len)
```

参数说明：

- ip_t：目标IP套接字指针，用于将数据发送到目标IP地址。
- const void *buf：目标字符缓冲区，包含要发送的数据。
- size_t len：目标字符缓冲区中的数据长度。

函数实现：

1. 将buf中的数据复制到ip的IP地址中。
2. 使用sendto函数将数据发送到ip的IP地址，并取得返回值。
3. 如果发送成功，将返回目标IP套接字到len字节长度的数据被发送的返回值。
4. 如果发送失败，返回-1。


```cpp
ssize_t
ip_send(ip_t *ip, const void *buf, size_t len)
{
	struct ip_hdr *hdr = (struct ip_hdr *)buf;
	
	ip->sin.sin_addr.s_addr = hdr->ip_src;
	
	if ((len = sendto(ip->fd, (const char *)buf, (int)len, 0,
	    (struct sockaddr *)&ip->sin, sizeof(ip->sin))) != SOCKET_ERROR)
		return (ssize_t)(len);
	
	return (-1);
}

ip_t *
```

这段代码是一个用于关闭套接字（socket）的函数，它属于一个名为ip_close的函数。函数接受一个整型指针变量ip，用于存储输入输出套接字（socket）的地址。函数首先检查ip是否为空，如果是，则执行WSACleanup函数清空套接字。然后，它检查ip->fd是否为无效套接字，如果是，则调用closesocket函数关闭套接字并返回。最后，如果ip和ip->fd均有效，函数释放套接字内存并返回一个空指针。


```cpp
ip_close(ip_t *ip)
{
	if (ip != NULL) {
		WSACleanup();
		if (ip->fd != INVALID_SOCKET)
			closesocket(ip->fd);
		free(ip);
	}
	return (NULL);
}

```

# `libdnet-stripped/src/ip.c`

这段代码是一个C语言的程序，它定义了一个名为"ip.c"的文件。程序的作用是实现网络接口（socket）的基本功能，包括绑定IP地址和端口号、监听网络连接、接收数据等。以下是程序的一些重要部分：

1. 引入网络头文件：ip.h
2. 引入IP地址配置文件：config.h
3. 定义INET_RESOLVED_ADDRESSES constant，值为127.0.0.1，表示IPv4广播地址
4. 函数err_if_ <错误判断函数，从config.h中导入
5. 函数sock_create <socket创建函数>
6. 函数sock_bind <socket绑定函数>
7. 函数sock_listen <socket监听函数>
8. 函数sock_accept <socket接受函数>
9. 函数send_msg <发送消息函数，从stdio.h中导入>
10. 函数recv_msg <接收消息函数，从stdio.h中导入>
11. 函数close <关闭socket函数>

这段代码主要用于实现网络接口的基本操作，包括绑定IP地址和端口号、监听网络连接、接收数据等。这些操作通过调用socket的相关函数来完成。通过这些函数，程序可以实现与客户端的通信，并在网络上接收和发送数据。


```cpp
/*
 * ip.c
 *
 * Copyright (c) 2000 Dug Song <dugsong@monkey.org>
 *
 * $Id: ip.c 547 2005-01-25 21:30:40Z dugsong $
 */

#include "config.h"

#include <netinet/in.h>

#include <errno.h>
#include <stdio.h>
#include <stdlib.h>
```

这段代码的主要作用是创建一个名为ip_handle的结构体数组，并输出该数组的地址，以便在程序运行时动态分配内存。

具体来说，代码中首先引入了头文件string.h和unistd.h，然后定义了一个名为ip_handle的结构体，该结构体包含一个fd成员变量，表示用于存储IP地址的套接字描述符(socket描述符)。

接着，代码中定义了一个名为ip_open的函数，该函数用于创建一个空的ip_handle结构体并返回其地址。函数的主要实现过程如下：

1. 如果静态分配内存失败，就返回 NULL，表示函数内部存在错误。
2. 如果创建IP套接字失败，就返回之前创建的ip_handle结构体的地址，表示函数内部存在错误。
3. 创建一个新的ip_handle结构体，并将其fd成员变量设置为套接字的描述符。
4. 返回新创建的ip_handle结构体的地址，以便在程序运行时动态分配内存。

由于ip_handle结构体中只包含一个fd成员变量，因此在程序运行时，只需要创建一个空的ip_handle结构体，就可以正确地使用动态内存分配。


```cpp
#include <string.h>
#include <unistd.h>

#include "dnet.h"

struct ip_handle {
	int	fd;
};

ip_t *
ip_open(void)
{
	ip_t *i;
	int n;
	socklen_t len;

	if ((i = calloc(1, sizeof(*i))) == NULL)
		return (NULL);

	if ((i->fd = socket(AF_INET, SOCK_RAW, IPPROTO_RAW)) < 0)
		return (ip_close(i));
```

这段代码的作用是检查服务器套接字（i.fd）是否支持IPv4头部包含选项（IPPROTO_IP）以及是否支持SO_SNDBUF选项。如果服务器套接字不支持IPv4头部包含选项，或者不支持SO_SNDBUF选项，则会关闭服务器套接字并返回错误代码。如果服务器套接字支持IPv4头部包含选项，则会尝试使用SO_SNDBUF选项将套接字缓冲区中的所有数据发送出去。如果尝试发送的数据长度小于服务器套接字缓冲区中设置的最大数据缓冲区长度（128个字节），则会发生写入超时错误，从而关闭服务器套接字并返回错误代码。


```cpp
#ifdef IP_HDRINCL
	n = 1;
	if (setsockopt(i->fd, IPPROTO_IP, IP_HDRINCL, &n, sizeof(n)) < 0)
		return (ip_close(i));
#endif
#ifdef SO_SNDBUF
	len = sizeof(n);
	if (getsockopt(i->fd, SOL_SOCKET, SO_SNDBUF, &n, &len) < 0)
		return (ip_close(i));

	for (n += 128; n < 1048576; n += 128) {
		if (setsockopt(i->fd, SOL_SOCKET, SO_SNDBUF, &n, len) < 0) {
			if (errno == ENOBUFS)
				break;
			return (ip_close(i));
		}
	}
```

这段代码是一个用于发送IP数据包的函数。它实现了两个条件判断，一个是在服务器套接字上发送数据，另一个是在所有连接的套接字上发送数据。

具体来说，代码首先定义了一个变量n，其值为1。接着判断调用setsockopt()函数是否成功，如果失败，就返回。如果成功，说明当前套接字支持发送IP数据包，就设置n的值为1。然后调用i的fd套接字，并返回一个整数，表示ip_send()函数的返回值。

接下来是ip_send()函数体，首先将一个IP数据报的header和实际要发送的数据组合在一起，形成一个ip_hdr结构。然后将这个ip_hdr结构的一个指针赋值给一个struct sockaddr_in类型的变量sin，用于保存发送数据包的源地址和端口号。

最后，调用ip_send()函数，并将已经准备好的ip_hdr和sin结构体作为参数传入，将其返回值作为整数返回。


```cpp
#endif
#ifdef SO_BROADCAST
	n = 1;
	if (setsockopt(i->fd, SOL_SOCKET, SO_BROADCAST, &n, sizeof(n)) < 0)
		return (ip_close(i));
#endif
	return (i);
}

ssize_t
ip_send(ip_t *i, const void *buf, size_t len)
{
	struct ip_hdr *ip;
	struct sockaddr_in sin;

	ip = (struct ip_hdr *)buf;

	memset(&sin, 0, sizeof(sin));
```

这段代码是一个 C 语言程序，它用于在 Linux 系统中实现发送数据的功能。现在我们来逐步解释这段代码的作用：

1. 首先，定义了一些符号常量：HAVE_SOCKADDR_SA_LEN、HAVE_RAWIP_HOST_OFFLEN。这两个符号常量分别表示是否有套接字地址（socket）和是否有 raw IP 地址（IPv4）支持。

2. 然后，定义了两个整型变量 sin 和 ip。这两个变量分别表示源 IP 地址和目标 IP 地址。

3. 在这两个变量中，通过系统调用函数 sin() 获取了目标 IP 地址的套接字地址（socket）长度，然后将其存储到了 sin.sin_len 变量中。

4. 通过系统调用函数 ip_set_dst() 设置源 IP 地址，然后使用 sendto() 函数将其发送到目标 IP 地址。这段代码使用了套接字（socket）发送数据的功能。

5. 最后，通过系统调用函数 ntohs() 和 htons() 函数实现了将 IP 数据包发送长度和偏移量转换为数值类型。

6. 在函数的末尾，通过调用 sendto() 函数并使用瓦片甲烷回调函数，实现了将数据发送到目标 IP 地址的功能。

7. 最后，返回了函数的参数 len，作为数据包发送成功后返回的结果。


```cpp
#ifdef HAVE_SOCKADDR_SA_LEN       
	sin.sin_len = sizeof(sin);
#endif
	sin.sin_family = AF_INET;
	sin.sin_addr.s_addr = ip->ip_dst;
	
#ifdef HAVE_RAWIP_HOST_OFFLEN
	ip->ip_len = ntohs(ip->ip_len);
	ip->ip_off = ntohs(ip->ip_off);

	len = sendto(i->fd, buf, len, 0,
	    (struct sockaddr *)&sin, sizeof(sin));
	
	ip->ip_len = htons(ip->ip_len);
	ip->ip_off = htons(ip->ip_off);

	return (len);
```

这段代码是一个 C 语言的函数，它定义了两个函数，一个是 `ip_open`，另一个是 `ip_close`。它们的功能如下：

1. `ip_open`：用于打开一个 TCP 套接字（ip_t 类型的数据结构）。它接受一个 `ip_t` 类型的数据结构作为参数，然后执行以下操作：

  a. 如果 `i` 所指数据结构中包含一个已知的 TCP 套接字，则直接返回。

  b. 如果 `i` 所指数据结构中包含一个 TCP 套接字但尚未打开，则创建一个新的 TCP 套接字并返回。

  c. 如果 `i` 所指数据结构中包含一个尚未打开的 TCP 套接字，则创建一个新的 TCP 套接字并返回。

2. `ip_close`：用于关闭一个已经打开的 TCP 套接字。它接受一个 `ip_t` 类型的数据结构作为参数，然后执行以下操作：

  a. 如果 `i` 所指数据结构中包含一个已知的 TCP 套接字，则先调用 `close` 函数关闭该套接字，然后将 `i` 所指数据结构指向 NULL，释放内存。

  b. 如果 `i` 所指数据结构中包含一个已知的 TCP 套接字但尚未打开，则调用 `close` 函数关闭该套接字。

  c. 如果 `i` 所指数据结构中包含一个尚未打开的 TCP 套接字，则创建一个新的 TCP 套接字并调用 `close` 函数关闭该套接字。

注意：`ip_open` 和 `ip_close` 函数的作用域均为函数内部，不会对外部程序产生影响。


```cpp
#else
	return (sendto(i->fd, buf, len, 0,
	    (struct sockaddr *)&sin, sizeof(sin)));
#endif
}

ip_t *
ip_close(ip_t *i)
{
	if (i != NULL) {
		if (i->fd >= 0)
			close(i->fd);
		free(i);
	}
	return (NULL);
}

```

# `libdnet-stripped/src/ip6.c`

这段代码是一个用于在Linux系统上实现IPv6协议的C代码。它定义了一个名为ip6.c的文件，其中定义了一些函数和定义了一些变量。以下是该代码的作用说明：

1. 定义了一个名为ip6.c的文件，说明这是一个C语言源文件。

2. 定义了几个函数，包括ip6_init、ip6_connect、ip6_send、ip6_get、ip6_put等。这些函数用于IPv6协议的初始化、连接、数据传输等功能。

3. 定义了一些变量，包括ip6_hWnd、ip6_sock、ip6_ip、ip6_port等。这些变量将在函数中用来操作IPv6协议套接字。

4. 在函数ip6_init中，初始化IPv6协议套接字，包括设置IPv6协议头、安全性和接口等。

5. 在函数ip6_connect中，建立IPv6协议的连接，包括发送SYN包、接收确认应答等。

6. 在函数ip6_send和ip6_get中，分别发送和接收IPv6协议的数据包，可以用于发送数据、接收数据包、获取发送的数据等。

7. 在函数ip6_put中，接收并解析IPv6协议的数据包，可以用于接收数据、解析数据等。


```cpp
/*
 * ip6.c
 *
 * Copyright (c) 2002 Dug Song <dugsong@monkey.org>
 *
 * $Id: ip6.c 539 2005-01-23 07:36:54Z dugsong $
 */

#ifdef _WIN32
#include "dnet_winconfig.h"
#else
#include "config.h"
#endif

#include "dnet.h"

```

This is a function definition for `ip_proxy_send_ipv6` which takes a packet pointer `p` and a checksum `sum` as input.

It checks the type of the input `nxt` and performs the appropriate checks or calculations.

If `nxt` is of the IP protocol and the `p` packet contains an IPv6 header, it extracts the source and destination Unicast Indicators and the checksum, and sends the packet with the checksum.

If `nxt` is of the ICMP protocol or if `nxt` is of the IGMP protocol, it checks the checksum, and if it's not equal to 0, it updates the checksum with the calculated value.


```cpp
#define IP6_IS_EXT(n)	\
	((n) == IP_PROTO_HOPOPTS || (n) == IP_PROTO_DSTOPTS || \
	 (n) == IP_PROTO_ROUTING || (n) == IP_PROTO_FRAGMENT)

void
ip6_checksum(void *buf, size_t len)
{
	struct ip6_hdr *ip6 = (struct ip6_hdr *)buf;
	struct ip6_ext_hdr *ext;
	u_char *p, nxt;
	int i, sum;
	
	nxt = ip6->ip6_nxt;
	
	for (i = IP6_HDR_LEN; IP6_IS_EXT(nxt); i += (ext->ext_len + 1) << 3) {
		if (i >= (int)len) return;
		ext = (struct ip6_ext_hdr *)((u_char *)buf + i);
		nxt = ext->ext_nxt;
	}
	p = (u_char *)buf + i;
	len -= i;
	
	if (nxt == IP_PROTO_TCP) {
		struct tcp_hdr *tcp = (struct tcp_hdr *)p;
		
		if (len >= TCP_HDR_LEN) {
			tcp->th_sum = 0;
			sum = ip_cksum_add(tcp, len, 0) + htons(nxt + (u_short)len);
			sum = ip_cksum_add(&ip6->ip6_src, 32, sum);
			tcp->th_sum = ip_cksum_carry(sum);
		}
	} else if (nxt == IP_PROTO_UDP) {
		struct udp_hdr *udp = (struct udp_hdr *)p;

		if (len >= UDP_HDR_LEN) {
			udp->uh_sum = 0;
			sum = ip_cksum_add(udp, len, 0) + htons(nxt + (u_short)len);
			sum = ip_cksum_add(&ip6->ip6_src, 32, sum);
			if ((udp->uh_sum = ip_cksum_carry(sum)) == 0)
				udp->uh_sum = 0xffff;
		}
	} else if (nxt == IP_PROTO_ICMPV6) {
		struct icmp_hdr *icmp = (struct icmp_hdr *)p;

		if (len >= ICMP_HDR_LEN) {
			icmp->icmp_cksum = 0;
			sum = ip_cksum_add(icmp, len, 0) + htons(nxt + (u_short)len);
			sum = ip_cksum_add(&ip6->ip6_src, 32, sum);
			icmp->icmp_cksum = ip_cksum_carry(sum);
		}		
	} else if (nxt == IP_PROTO_ICMP || nxt == IP_PROTO_IGMP) {
		struct icmp_hdr *icmp = (struct icmp_hdr *)p;
		
		if (len >= ICMP_HDR_LEN) {
			icmp->icmp_cksum = 0;
			sum = ip_cksum_add(icmp, len, 0);
			icmp->icmp_cksum = ip_cksum_carry(sum);
		}
	}
}

```

# `libdnet-stripped/src/memcmp.c`

Berkeley is a university located in California, and it is the subject of this software license. Chris Torek has made this software available under the following conditions:

1. Redistributions of this software must retain the above copyright notice, this list of conditions and the following disclaimer.
2. Redistributions in binary form must reproduce the above copyright notice, this list of conditions and the following disclaimer in the documentation and/or other materials provided with the distribution.
3. All advertising materials mentioning features or use of this software must display the following acknowledgement:
```cppmakefile
This product includes software developed by the University of California, Berkeley and its contributors.
```
4. The name of the University or its contributors may not be used to endorse or promote products derived from this software without specific prior written permission.

This software is provided "AS IS" and any express or implied warranties, including merchantability and fitness for a particular purpose, are disclaimed. In no event shall the regents or contributors be liable for any direct, indirect, incidental, special, exhaustive, or consequential damages (including, but not limited to, procurement of substitute goods or services; loss of use, data, or profits; or business interruption) how ever caused and on any theory of liability, whether in contract, strict liability, or tort (including negligence or otherwise) arising out of the use of this software, even if advised of the possibility of such damage.


```cpp
/*-
 * Copyright (c) 1990 The Regents of the University of California.
 * All rights reserved.
 *
 * This code is derived from software contributed to Berkeley by
 * Chris Torek.
 *
 * Redistribution and use in source and binary forms, with or without
 * modification, are permitted provided that the following conditions
 * are met:
 * 1. Redistributions of source code must retain the above copyright
 *    notice, this list of conditions and the following disclaimer.
 * 2. Redistributions in binary form must reproduce the above copyright
 *    notice, this list of conditions and the following disclaimer in the
 *    documentation and/or other materials provided with the distribution.
 * 3. All advertising materials mentioning features or use of this software
 *    must display the following acknowledgement:
 *	This product includes software developed by the University of
 *	California, Berkeley and its contributors.
 * 4. Neither the name of the University nor the names of its contributors
 *    may be used to endorse or promote products derived from this software
 *    without specific prior written permission.
 *
 * THIS SOFTWARE IS PROVIDED BY THE REGENTS AND CONTRIBUTORS ``AS IS'' AND
 * ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE
 * IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE
 * ARE DISCLAIMED.  IN NO EVENT SHALL THE REGENTS OR CONTRIBUTORS BE LIABLE
 * FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL
 * DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS
 * OR SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION)
 * HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT
 * LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY
 * OUT OF THE USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF
 * SUCH DAMAGE.
 */

```

这段代码是一个C语言代码，它检查两个参数s1和s2是不是相等，并且检查它们的长度是否大于0。如果是，那么它执行下面的操作：将p1和p2指向内存中的两个连续位置，然后逐个比较它们的值。如果它们的值不同，那么它们之间的差值被返回。最后，如果n不等于0，那么它返回这个差值。

该代码与Linux系统中的`memcmp`函数非常相似。这个函数比较两个字符串是否相等，如果不相等，就返回它们的差值。这个函数是在`libc`库中定义的，因此代码中包含`#if defined(LIBC_SCCS) && !defined(lint)`来检查该函数是否定义。如果没有定义`lint`函数，那么就会输出`$OpenBSD: memcmp.c,v 1.2 1996/08/19 08:34:05 tholo Exp $`错误。

该代码与`memcmp`函数不同的是，它使用了C语言的内存地址和参数，而不是Unix中的`memcmp`函数的参数。这是因为`memcmp`函数需要两个字符串参数，并且需要在内存中进行比较，而该代码中的`memcmp`函数不需要这些参数，并且只需要在代码中进行比较。


```cpp
#if defined(LIBC_SCCS) && !defined(lint)
static char *rcsid = "$OpenBSD: memcmp.c,v 1.2 1996/08/19 08:34:05 tholo Exp $";
#endif /* LIBC_SCCS and not lint */

#include <string.h>

/*
 * Compare memory regions.
 */
int
memcmp(s1, s2, n)
	const void *s1, *s2;
	size_t n;
{
	if (n != 0) {
		register const unsigned char *p1 = s1, *p2 = s2;

		do {
			if (*p1++ != *p2++)
				return (*--p1 - *--p2);
		} while (--n != 0);
	}
	return (0);
}

```

# `libdnet-stripped/src/rand.c`

这段代码是一个 C 语言源文件，它定义了一个名为 "rand.c" 的文件。函数名称隐式地定义了 "rand" 函数。接下来是定义了一些通用的宏，以及一些函数原型。

_WIN32 是一个 preprocessor 标志，这意味着代码中使用的是 Windows 平台的相关库。这意味着代码中定义的函数是可移植的。然而，这不是代码的主要部分，因此我们无法确定这里的代码具体是为了在 Windows 系统上做什么而设计的。


```cpp
/*
 * rand.c
 *
 * Pseudorandom number generation, based on OpenBSD arc4random().
 *
 * Copyright (c) 2000 Dug Song <dugsong@monkey.org>
 * Copyright (c) 1996 David Mazieres <dm@lcs.mit.edu>
 *
 * $Id: rand.c 587 2005-02-15 06:37:07Z dugsong $
 */

#include "config.h"

#ifdef _WIN32
/* XXX */
```

这段代码的作用是定义了一个名为"_WIN32_WINNT"的函数指针变量，它的值为(_WIN32_WINNT_WIN7)，然后定义了一个名为"_bcrypt"的函数。接着通过 pragma 命令声明该函数使用 bcrypt.lib 库。然后通过 define 定义了一个名为"inline"的函数，它的定义中使用了 __inline 修饰，意味着该函数只需要在编译时进行展开，而不需要生成可执行文件。接下来通过 else 引用了系统提供的头文件 <sys/types.h> 和 <sys/time.h>，以及 <unistd.h>。最后通过 fcntl.h 头文件引入了用于文件 I/O 的头文件 <fcntl.h>。整个函数体内部包含了对 bcrypt.lib 库的引入和使用。


```cpp
# undef _WIN32_WINNT
# define _WIN32_WINNT _WIN32_WINNT_WIN7
# include <bcrypt.h>
# pragma comment(lib, "bcrypt.lib")
# define inline __inline
#else
# include <sys/types.h>
# include <sys/time.h>
# include <unistd.h>
#endif
#include <fcntl.h>
#include <stdlib.h>
#include <string.h>

#include "dnet.h"

```



这段代码定义了一个名为 `rand_handle` 的结构体，其中包含以下成员：

- `i`：一个 8 位整数，用于保存当前循环的索引。
- `j`：一个 8 位整数，用于保存当前循环的步长。
- `s`：一个 256 字节的数组，用于保存当前循环的随机数。
- `tmp`：一个指向一个 8 位整数的指针，用于保存生成的随机数。
- `tmplen`：一个存储生成的随机数长度差的整数，用于保存生成的随机数的长度。

该结构体的定义了一个名为 `rand_init` 的函数，该函数接收一个 `rand_t` 类型的参数，用于初始化随机数生成器。函数内部先定义了一个包含 256 个元素的数组 `s`，并将其赋值为 0。然后将 `i` 赋值为 0,`j` 赋值为 0。最后，函数使用内部循环来遍历数组 `s`，并将每个元素的值赋为当前循环的步长，即 `i` 和 `j`。

另外，还定义了一个名为 `rand_generate` 的函数，用于生成随机数。该函数接收一个 `rand_t` 类型的参数，用于生成随机数。函数内部先定义了一个包含 256 个元素的数组 `s`，并将其赋值为 0。然后使用 `rand()` 函数生成一个介于 0 到 `RAND_MAX` 之间的随机数，并将其存储到数组 `s` 中。接着，函数使用内部循环来遍历数组 `s`，并从数组中随机选择一个元素，存储到 `tmp` 指向的数组中。最后，函数计算并存储生成的随机数的长度差，即 `tmplen - rand()`。

该结构体和函数可以一起用于生成随机的数据，例如用于游戏中的随机数、密码学等等。


```cpp
struct rand_handle {
	uint8_t		 i;
	uint8_t		 j;
	uint8_t		 s[256];
	u_char		*tmp;
	int		 tmplen;
};

static inline void
rand_init(rand_t *rand)
{
	int i;
	
	for (i = 0; i < 256; i++)
		rand->s[i] = i;
	rand->i = rand->j = 0;
}

```

这段代码定义了一个名为`rand_addrandom`的函数，属于`static inline`类型，其作用是生成一个指定长度的随机数，并将结果存储在`buf`指向的内存区域中。

函数接收3个参数：一个指向`rand_t`类型对象的指针、一个`u_char`类型指针以及一个整数参数`len`。函数内部使用变量`rand_i`和`rand_j`来跟踪随机数生成器内部的当前状态，`rand_s`是一个包含32个随机整数的数组，用于生成随机数。

函数的主要思想是使用循环来遍历生成器中的所有元素，然后将当前状态的随机数与传入的`buf[i % len]`进行按位与运算，并将结果存回原来的位置。此外，rand_i和rand_j还自增1，以便在下一次循环时能够访问不同的位置。

最后，函数将`rand_j`的值赋给`rand_i`，并将`rand_i`的值赋为`rand_j`，这样就保证了按位与操作的结果不会产生不可预测的副作用。


```cpp
static inline void
rand_addrandom(rand_t *rand, u_char *buf, int len)
{
	int i;
	uint8_t si;
	
	rand->i--;
	for (i = 0; i < 256; i++) {
		rand->i = (rand->i + 1);
		si = rand->s[rand->i];
		rand->j = (rand->j + si + buf[i % len]);
		rand->s[rand->i] = rand->s[rand->j];
		rand->s[rand->j] = si;
	}
	rand->j = rand->i;
}

```

这段代码的作用是生成一个随机数种子，并使用该种子来生成一个伪随机数随机数。它适用于在不需要使用系统预设随机数种子的情况下生成随机数。

具体来说，代码首先定义了一个名为 rand_t 的指针变量 r，以及一个 u_char 类型的字节数组 seed，大小为 256。然后，代码使用 if 语句检查是否成功使用 BCryptGenRandom 函数生成随机数种子。如果该函数成功，则直接返回一个随机数种子句柄。否则，代码会尝试打开 /dev/arandom 和 /dev/urandom 设备文件，并从它们中读取随机数种子。如果这两个操作都成功，则读取一定数量的随机数字节，然后关闭设备文件并获取当前时间戳。最后，代码将生成的随机数种子保存到 seed 数组中，并从它中随机选择一个随机数，用于生成伪随机数随机数。


```cpp
rand_t *
rand_open(void)
{
	rand_t *r;
	u_char seed[256];
#ifdef _WIN32
	if (STATUS_SUCCESS != BCryptGenRandom(NULL, seed, sizeof(seed), BCRYPT_USE_SYSTEM_PREFERRED_RNG))
	  return NULL;
#else
	struct timeval *tv = (struct timeval *)seed;
	int fd;

	if ((fd = open("/dev/arandom", O_RDONLY)) != -1 ||
	    (fd = open("/dev/urandom", O_RDONLY)) != -1) {
		read(fd, seed + sizeof(*tv), sizeof(seed) - sizeof(*tv));
		close(fd);
	}
	gettimeofday(tv, NULL);
```

这段代码是一个 C 语言的函数，名为“rand_getbyte”，属于一个名为“rand”的函数组合。它的作用是生成一个 0 到 255 之间的随机整数，并返回这个随机整数。

函数的实现包括以下几个步骤：

1. 定义了一个名为“r”的变量，并进行了初始化：令“r”指向“malloc()”函数返回的内存区域，如果没有内存区域可用，函数将“malloc()”调用失败，并返回一个空指针。然后，进行了“rand_init()”和“rand_addrandom()”函数的调用，对“r”指向的内存区域进行了初始化和随机化。

2. 定义了一个名为“tmp”的变量，并进行了初始化：令“tmp”为空，然后将“tmp”的值存储为“NULL”。

3. 定义了一个名为“tmplen”的变量，并进行了初始化：令“tmplen”为 0，然后将“tmplen”的值存储为“0”。

4. 返回“tmp”，这个值是一个指向“NULL”的指针，用来表示函数没有成功生成随机整数，或者需要重新生成随机整数。

总的来说，这段代码生成随机整数的过程包括：初始化“r”指向的内存区域、对内存区域进行初始化和随机化、设置一些辅助变量、返回生成的随机整数。


```cpp
#endif
	if ((r = malloc(sizeof(*r))) != NULL) {
		rand_init(r);
		rand_addrandom(r, seed, 128);
		rand_addrandom(r, seed + 128, 128);
		r->tmp = NULL;
		r->tmplen = 0;
	}
	return (r);
}

static uint8_t
rand_getbyte(rand_t *r)
{
	uint8_t si, sj;

	r->i = (r->i + 1);
	si = r->s[r->i];
	r->j = (r->j + si);
	sj = r->s[r->j];
	r->s[r->i] = sj;
	r->s[r->j] = si;
	return (r->s[(si + sj) & 0xff]);
}

```

这两函数是用于生成伪随机整数的。其中，rand_get()函数接受一个rand_t类型的参数，返回一个长度为len的u_char数组，用于存储生成的随机整数。rand_set()函数接受一个rand_t类型的参数，以及一个指向const void类型的指针buf，用于存储生成的随机整数。函数内部使用rand_init()和rand_addrandom()函数来生成随机整数。其中，rand_init()函数初始化rand_t类型的参数r,rand_addrandom()函数用于在指定范围内添加随机数。


```cpp
int
rand_get(rand_t *r, void *buf, size_t len)
{
	u_char *p;
	u_int i;

	for (p = buf, i = 0; i < len; i++) {
		p[i] = rand_getbyte(r);
	}
	return (0);
}

int
rand_set(rand_t *r, const void *buf, size_t len)
{
	rand_init(r);
	rand_addrandom(r, (u_char *)buf, len);
	rand_addrandom(r, (u_char *)buf, len);
	return (0);
}

```

这段代码是一个C语言的函数，定义了两个名为rand_add和rand_uint8的函数，以及一个名为rand_uint16的函数。

rand_add函数接受两个参数：一个指向随机数生成器的指针(rand_t *r)，一个是要生成随机数据的缓冲区(const void *buf)，以及一个字符数组长度(size_t len)。函数首先使用rand_addrandom函数对生成器进行初始化，然后使用该生成器生成指定长度的随机数据，并将结果存储到buf指向的内存区域中。最后，函数返回0表示成功，并返回生成的随机数据的长度。

rand_uint8函数接受一个参数：一个指向随机数生成器的指针(rand_t *r)，它将生成一个0到255之间的随机整数。函数首先使用rand_getbyte函数从生成器中获取一个随机整数，然后将其左移8位，以便将其可以被存储为两个字节的长度。最后，函数返回生成的随机整数。

rand_uint16函数与rand_uint8类似，只是生成了一个16字节的随机整数。函数与rand_uint8相似，只是将rand_getbyte函数生成的随机整数左移8位。最后，函数返回生成的随机整数。


```cpp
int
rand_add(rand_t *r, const void *buf, size_t len)
{
	rand_addrandom(r, (u_char *)buf, len);
	return (0);
}

uint8_t
rand_uint8(rand_t *r)
{
	return (rand_getbyte(r));
}

uint16_t
rand_uint16(rand_t *r)
{
	uint16_t val;

	val = rand_getbyte(r) << 8;
	val |= rand_getbyte(r);
	return (val);
}

```

这两段代码定义了两个名为"rand_uint32"和"rand_shuffle"的函数，用于生成指定格式的随机数。

1. "rand_uint32"函数的作用是生成一个32位的随机数。函数的实参"rand_t r"是一个指向随机数发生器的指针，函数内部使用"rand_getbyte"函数从该发生器中获取随机数，然后进行左移运算，将获取到的随机数合并为32位无符号整数，并将其返回。

2. "rand_shuffle"函数的作用是按照指定的格式的随机数重新排列指定的字节流中的数据，使得数据按照指定的格式的随机数重新排列后，依然保持原来的顺序。函数的实参"rand_t r"是一个指向随机数发生器的指针，函数内部使用"rand_uint32"函数生成指定格式的随机数，然后使用"memcpy"函数将生成的随机数与指定的字节流中的数据进行重排，并将其返回。函数的实参"void *base"是一个指针，指针指向一个32字节的字节流对象，函数内部将生成的随机数与该字节流中的数据进行重排，并将其存储到指定的指针中，最后将指针返回。函数的实参"size_t nmemb"是一个整数类型，表示需要重排的字节数，函数内部将生成的随机数与指定的字节数进行重排，以确保数据的正确性。函数的实参"size_t size"是一个整数类型，表示需要生成的随机数的长度，函数内部使用"rand_getbyte"函数从指定格式的随机数发生器中获取指定长度的随机数，并将其合并为32位无符号整数，并将其存储到指定的位置。


```cpp
uint32_t
rand_uint32(rand_t *r)
{
	uint32_t val;

	val = rand_getbyte(r) << 24;
	val |= rand_getbyte(r) << 16;
	val |= rand_getbyte(r) << 8;
	val |= rand_getbyte(r);
	return (val);
}

int
rand_shuffle(rand_t *r, void *base, size_t nmemb, size_t size)
{
	u_char *save, *src, *dst, *start = (u_char *)base;
	u_int i, j;

	if (nmemb < 2)
		return (0);
	
	if ((u_int)r->tmplen < size) {
		if (r->tmp == NULL) {
			if ((save = malloc(size)) == NULL)
				return (-1);
		} else if ((save = realloc(r->tmp, size)) == NULL)
			return (-1);
		
		r->tmp = save;
		r->tmplen = size;
	} else
		save = r->tmp;
	
	for (i = 0; i < nmemb; i++) {
		if ((j = rand_uint32(r) % (nmemb - 1)) != i) {
			src = start + (size * i);
			dst = start + (size * j);
			memcpy(save, dst, size);
			memcpy(dst, src, size);
			memcpy(src, save, size);
		}
	}
	return (0);
}

```

这段代码定义了一个名为rand_t的指针变量，并且实现了一个名为rand_close的函数。rand_close函数接收一个rand_t类型的指针变量r，并且如果r指向的内存不为空，则先释放r指向的内存，再释放r指向的内存。最后，rand_close函数返回一个指向 NULL的指针。

具体来说，rand_t *r指向的内存如果是空的，则rand_close函数不会做任何操作，相当于直接返回一个NULL的指针。如果r指向的内存不为空，则先释放r指向的内存，这意味着r指向的内存现在可以被自由地使用，比如可以再次赋值或者申请内存空间等。然后再释放r指向的内存，这个内存也不再可以被使用，但是可以被再次申请内存空间。

通过释放r指向的内存，可以确保rand_t类型的指针变量r所指向的内存不会泄漏，也就是不会产生内存泄漏问题。同时，释放r指向的内存也可以确保rand_t类型的指针变量r本身不会被绑定到内存上，从而保证rand_t指针变量的正确性。


```cpp
rand_t *
rand_close(rand_t *r)
{
	if (r != NULL) {
		if (r->tmp != NULL)
			free(r->tmp);
		free(r);
	}
	return (NULL);
}

```