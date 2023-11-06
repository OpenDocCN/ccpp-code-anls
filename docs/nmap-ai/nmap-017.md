# Nmap源码解析 17

# `libdnet-stripped/src/route-bsd.c`

这段代码是一个 C 语言程序，名为 "route-bsd.c"，它定义了一个和系统相关的函数。

具体来说，它实现了以下功能：

1. 引入了 "config.h" 头文件，用于定义一些常量和宏。
2. 引入了 "sys/param.h"，用于定义系统参数。
3. 引入了 "sys/types.h"，用于定义系统types。
4. 引入了 "sys/socket.h"，用于定义套接字操作。
5. 定义了一个名为 "sin_port" 的函数，使用了 "sys/types.h" 中 "int" 类型，表示支持 TCP 和 UDP 协议。
6. 定义了一个名为 "sin_set" 的函数，使用了 "sys/types.h" 中 "int" 类型，表示设置函数。
7. 通过组合上述函数，实现了监听指定端口的函数 "main"。

这个程序的作用是实现一个简单的服务器程序，可以监听指定端口并接收客户端发送的数据，根据客户端发送的数据的不同协议，调用不同的函数实现客户端和服务器的通信。


```cpp
/*
 * route-bsd.c
 *
 * Copyright (c) 2001 Dug Song <dugsong@monkey.org>
 * Copyright (c) 1999 Masaki Hirabaru <masaki@merit.edu>
 * 
 * $Id: route-bsd.c 555 2005-02-10 05:18:38Z dugsong $
 */

#include "config.h"

#include <sys/param.h>
#include <sys/types.h>
#include <sys/socket.h>
#ifdef HAVE_SYS_SYSCTL_H
```

这段代码是一个C语言程序，它包括了系统调用相关的头文件，声明了一些预定义变量，以及定义了一些函数。

具体来说，这个程序的作用是获取Linux系统当前的网络接口IP地址和子网掩码信息，并将它们存储在用户定义的变量中。它使用了`<sysctl.h>`头文件中的系统调用接口，通过调用`<sys/sysctl.h>`函数中的`get资本主义园`系统调用方法，获取了当前系统的网络接口信息。

具体实现过程如下：

1. 首先，定义了一个名为`ip_adr_len`的预定义变量，它的值为`IP_ADDR_LEN`，这是由于`<ip.h>`头文件中定义了`IP_ADDR_LEN`常量的含义，它表示IP地址的长度，它的值为`4`。

2. 接着，定义了一系列函数，包括`get_ip_address`函数和`get_ip_info`函数。其中，`get_ip_address`函数通过调用`<sys/sysctl.h>`函数中的`get资本主义园`系统调用方法，获取了当前系统的网络接口信息，并将其存储在`ip_adr`变量中。`get_ip_info`函数则实现了`<sys/stropts.h>`头文件中定义的`stropts`函数，获取用户定义的选项，并将它们存储在`options`变量中。

3. 最后，通过判断系统是否支持`< streams_route >`头文件中的定义，如果支持，则执行`get_ip_address`函数和`get_ip_info`函数，否则跳过它们。

总结起来，这段代码的作用是获取Linux系统当前的网络接口IP地址和子网掩码信息，并将它们存储在用户定义的变量中。


```cpp
#include <sys/sysctl.h>
#endif
#ifdef HAVE_STREAMS_MIB2
#include <sys/stream.h>
#include <sys/tihdr.h>
#include <sys/tiuser.h>
#include <inet/common.h>
#include <inet/mib2.h>
#include <inet/ip.h>
#undef IP_ADDR_LEN
#include <stropts.h>
#elif defined(HAVE_STREAMS_ROUTE)
#include <sys/stream.h>
#include <sys/stropts.h>
#endif
```

这段代码的作用是定义了一个名为"route_t"的枚举类型，枚举类型包括两个成员，分别是一个指向结构体类型的指针和一个无符号字符串。同时，该代码包含了一个名为"oroute_t"的定义，但是并没有定义该指针类型。

通过检查#ifdefprimeKernInfo是否为真，如果为真，则表明系统支持获取内核信息。接着，该代码通过引入<sys/kinfo.h>头文件，引入了<net/route.h>和<netinet/in.h>头文件，然后通过<net/if.h>和<netinet/in.h>头文件，实现了对网络接口的配置和获取。

接着，该代码通过函数<lambda>形式，给route_t类型的对象赋值，其中lambda是一个用户自定义的函数，该函数接收两个int类型的参数，分别表示要查找的路由表起始和终止的AS号。函数内部的代码通过cget知识产权获取系统的核苷酸缓冲区的句柄，并输出当前系统使用的核苷酸缓冲区的起始和终止的AS号，然后将获取到的AS号与当前route_t类型的成员进行比较，如果当前route_t类型的成员指向的AS号与系统当前使用的AS号相同，则将当前AS号与当前route_t类型的成员进行替换，替换后的AS号作为当前route_t类型的成员的值返回。

最后，该代码定义了一个无符号字符串"route_path"，用于存储系统的路由信息输出路径，该字符串在后续的代码中被用来输出系统的路由信息输出路径。


```cpp
#ifdef HAVE_GETKERNINFO
#include <sys/kinfo.h>
#endif

#define route_t	oroute_t	/* XXX - unixware */
#include <net/route.h>
#undef route_t
#include <net/if.h>
#include <netinet/in.h>

#include <errno.h>
#include <fcntl.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
```

这段代码的作用是包含头文件 "dnet.h"，并在其中引入了一个名为 "ROUNDUP" 的宏定义。

宏定义定义了一个名为 "a" 的变量，并使用了 "RT_ROUNDUP" 函数来获取一个整数变量的round up值。如果 "a" 的值是整数，那么 "ROUNDUP" 宏会将其转换为至少64位整数；否则，它会按照 "NETBSD" 定义的规则将 "a" 的值向上取整。这个规则是将 "a" 的值保留为4字节，然后将其转换为至少64位整数。

这里的作用是告诉读者，"ROUNDUP" 宏是一个可以用来将一个整数变量向上取整的函数，但需要保证该函数的定义在特定的操作系统上或者特定硬件上才能正确工作。


```cpp
#include <unistd.h>

#include "dnet.h"

#if defined(RT_ROUNDUP) && defined(__NetBSD__)
/* NetBSD defines this macro rounding to 64-bit boundaries.
   http://fxr.watson.org/fxr/ident?v=NETBSD;i=RT_ROUNDUP */
#define ROUNDUP(a) RT_ROUNDUP(a)
#else
/* Unix Network Programming, 3rd edition says that sockaddr structures in
   rt_msghdr should be padded so their addresses start on a multiple of
   sizeof(u_long). But on 64-bit Mac OS X 10.6 at least, this is false. Apple's
   netstat code uses 4-byte padding, not 8-byte. This is relevant for IPv6
   addresses, for which sa_len == 28.
   http://www.opensource.apple.com/source/network_cmds/network_cmds-329.2.2/netstat.tproj/route.c */
```

这段代码定义了一系列宏，用于定义在不同操作系统对 `#ifdef` 和 `#else` 条件下的行为。以下是每个宏的解释：

```cpp
#ifdef __APPLE__
#define RT_MSGHDR_ALIGNMENT sizeof(uint32_t)
#else
#define RT_MSGHDR_ALIGNMENT sizeof(unsigned long)
#endif
```

这两个宏定义了 `RT_MSGHDR_ALIGNMENT` 为 `sizeof(uint32_t)` 或 `sizeof(unsigned long)`，以定义在 `__APPLE__` 和 `__linux__` 目标平台上宏定义的偏移量。

```cpp
#define ROUNDUP(a) \
	((a) > 0 ? (1 + (((a) - 1) | (RT_MSGHDR_ALIGNMENT - 1))) : RT_MSGHDR_ALIGNMENT)
```

这个宏定义了一个名为 `ROUNDUP` 的函数，用于向上取整并计算一个 `a` 变量的值。函数的参数是一个整数类型的变量 `a`，它的作用域是 `__APPLE__` 和 `__linux__`。函数返回值是将 `a` 中的值右移一位并加上 `RT_MSGHDR_ALIGNMENT - 1`。

```cpp
#ifdef HAVE_SOCKADDR_SA_LEN
#define NEXTSA(s) \
	((struct sockaddr *)((u_char *)(s) + ROUNDUP((s)->sa_len)));
#else
#define NEXTSA(s) \
	((struct sockaddr *)((u_char *)(s) + ROUNDUP(sizeof(*(s)))));
```

这两个宏定义了一个名为 `NEXTSA` 的函数，用于获取一个 `sockaddr_sa_len` 类型的 SLA 或 TSA 结构的下一个 `sockaddr` 指针。函数的第一个参数是一个 `sockaddr_sa_len` 类型的变量 `s`，函数的第二个参数留空。函数的作用是计算 `sizeof(s)` 并将其作为 `sockaddr_sa_len` 类型的变量，然后将其复制到 `next_sa` 变量中。

```cpp
int main() {
   int x = 42;
   int y = ROUNDUP(x);
   int z;
   if (__APPLE__) {
       z = NEXTSA(y);
   } else {
       z = NEXTSA(y);
   }
   printf("%d\n", z);
   return 0;
}
```

这个 `main` 函数定义了一个整数类型的变量 `x` 并将其值设为 42，然后定义了一个名为 `ROUNDUP` 的函数并将其定义为 `x` 的向上取整函数。接下来，定义了一个名为 `NEXTSA` 的函数，用于获取一个 `sockaddr_sa_len` 类型的 SLA 或 TSA 结构的下一个 `sockaddr` 指针。在 `if` 语句中，使用 `NEXTSA` 函数获取 `y` 的下一个 `sockaddr` 指针，如果 `__APPLE__` 条件为真，则使用 `NEXTSA` 函数获取 `z` 的下一个 `sockaddr` 指针。最后，打印出 `z` 的值并输出结果。


```cpp
#ifdef __APPLE__
#define RT_MSGHDR_ALIGNMENT sizeof(uint32_t)
#else
#define RT_MSGHDR_ALIGNMENT sizeof(unsigned long)
#endif
#define ROUNDUP(a) \
	((a) > 0 ? (1 + (((a) - 1) | (RT_MSGHDR_ALIGNMENT - 1))) : RT_MSGHDR_ALIGNMENT)
#endif

#ifdef HAVE_SOCKADDR_SA_LEN
#define NEXTSA(s) \
	((struct sockaddr *)((u_char *)(s) + ROUNDUP((s)->sa_len)))
#else
#define NEXTSA(s) \
	((struct sockaddr *)((u_char *)(s) + ROUNDUP(sizeof(*(s)))))
```

这段代码是一个结构体声明，定义了一个名为 "route_handle" 的结构体类型，其成员包括一个整型变量 "fd" 和一个整型变量 "seq"，以及一个声明，但并未定义。

此代码中， defining a new struct type "route_handle", which contains two members of type int and one member of type "int" and one member of type "int".

This code defines a new struct type called "route_handle" which has two member variables of type int and one member variable of type "int".


```cpp
#endif

struct route_handle {
	int	fd;
	int	seq;
#ifdef HAVE_STREAMS_MIB2
	int	ip_fd;
#endif
};

#ifdef DEBUG
static void
route_msg_print(struct rt_msghdr *rtm)
{
	printf("v: %d type: 0x%x flags: 0x%x addrs: 0x%x pid: %d seq: %d\n",
	    rtm->rtm_version, rtm->rtm_type, rtm->rtm_flags,
	    rtm->rtm_addrs, rtm->rtm_pid, rtm->rtm_seq);
}
```

This function appears to be a part of a TCP transport layer that handles the delivery of a packet from a client to a server. It takes in a destination address (IP address and port number) and a gateway address (IP address and port number), and returns an error code.

The function first checks if the destination address is valid and if the gateway address is set. If the address is valid and the gateway address is set, the function checks if the packet was received from the client or the server. If the packet was received from the client, the function sets the destination address to the gateway address and adds a netmask to the destination address. If the packet was received from the server, the function sets the destination address to the client address and removes the netmask.

Then, the function checks if the destination address is a valid IP address and port number. If the address is valid and the port number is valid, the function sets the gateway flag and returns an error code. If the address is invalid or the port number is not a valid port number, the function returns an error code.

Finally, the function returns an error code if the packet delivery was successful.


```cpp
#endif

static int
route_msg(route_t *r, int type, char intf_name[INTF_NAME_LEN], struct addr *dst, struct addr *gw)
{
	struct addr net;
	struct rt_msghdr *rtm;
	struct sockaddr *sa;
	u_char buf[BUFSIZ];
	pid_t pid;
	int len;

	memset(buf, 0, sizeof(buf));

	rtm = (struct rt_msghdr *)buf;
	rtm->rtm_version = RTM_VERSION;
	if ((rtm->rtm_type = type) != RTM_DELETE)
		rtm->rtm_flags = RTF_UP;
	rtm->rtm_addrs = RTA_DST;
	rtm->rtm_seq = ++r->seq;

	/* Destination */
	sa = (struct sockaddr *)(rtm + 1);
	if (addr_net(dst, &net) < 0 || addr_ntos(&net, sa) < 0)
		return (-1);
	sa = NEXTSA(sa);

	/* Gateway */
	if (gw != NULL && type != RTM_GET) {
		rtm->rtm_flags |= RTF_GATEWAY;
		rtm->rtm_addrs |= RTA_GATEWAY;
		if (addr_ntos(gw, sa) < 0)
			return (-1);
		sa = NEXTSA(sa);
	}
	/* Netmask */
	if (dst->addr_ip == IP_ADDR_ANY || dst->addr_bits < IP_ADDR_BITS) {
		rtm->rtm_addrs |= RTA_NETMASK;
		if (addr_btos(dst->addr_bits, sa) < 0)
			return (-1);
		sa = NEXTSA(sa);
	} else
		rtm->rtm_flags |= RTF_HOST;
	
	rtm->rtm_msglen = (u_char *)sa - buf;
```

这段代码是一个用于实时路由监控（RTM）的打印函数，适用于 Linux 系统。它会根据用户设置的 PREEMPTION_RETURN 和 RTM 类型设置不同的行为。

首先，我们通过自定义函数 route_msg_print() 来打印信息，如果成功打印，则代表系统可以正常工作，否则打印出错误信息。这个函数在 DEBUG 和Have_STREAMS_ROUTE 这两个宏定义中存在，当宏定义不同时，只会在DEBUG中生效。

另外，我们还有一组条件判断，用于在具有 STREAMS_ROUTE 宏定义的情况下，判断是否可以通过非阻塞 IO 调用（ioctl() 和 write()）获取流数据。如果不能获取，说明设备可能不存在，需要返回一个错误代码。否则，继续读取数据，并分析所得到的信息。

最后一个判断部分，我们通过获取当前进程 ID，并与 RTM 中的 PID 和序列进行匹配，来确保我们所读取的信息是正确的。如果匹配成功，就会检查错误，否则继续等待后续数据。


```cpp
#ifdef DEBUG
	route_msg_print(rtm);
#endif
#ifdef HAVE_STREAMS_ROUTE
	if (ioctl(r->fd, RTSTR_SEND, rtm) < 0)
		return (-1);
#else
	if (write(r->fd, buf, rtm->rtm_msglen) < 0)
		return (-1);

	pid = getpid();
	
	while (type == RTM_GET && (len = read(r->fd, buf, sizeof(buf))) > 0) {
		if (len < (int)sizeof(*rtm)) {
			return (-1);
		}
		if (rtm->rtm_type == type && rtm->rtm_pid == pid &&
		    rtm->rtm_seq == r->seq) {
			if (rtm->rtm_errno) {
				errno = rtm->rtm_errno;
				return (-1);
			}
			break;
		}
	}
```

这段代码是一个 C 语言函数，名为 ifname。它用于检查输入参数 `rtm` 是否为 `RTM_GET` 类型，如果是，则执行以下操作：

1. 如果 `rtm` 包含 `RTM_ADDRS` 字段且其值为 `(RTA_DST|RTA_GATEWAY)`，那么执行以下操作：

  a. 获取 `rtm + 1` 指向的地址。

  b. 遍历该地址对应的 `RTA_DST` 和 `RTA_GATEWAY` 字段。

  c. 如果 `RTA_DST` 和 `RTA_GATEWAY` 都为非空，那么执行以下操作：

    i. 获取下一个 `RTA_DST` 或 `RTA_GATEWAY` 字段对应的 `struct sockaddr *` 类型的指针。

    ii. 获取该 `struct sockaddr *` 类型的指针 `sa`。

    iii. 如果 `addr_ston(sa, gateway)` 返回负数或者 `gateway` 的 `addr_type` 不是 `ADDR_TYPE_IP`，那么执行以下操作：

      a. 返回 `-1` 和错误码 `errno`。

      b. 执行完以上操作后，返回 `0`。

ifname 函数的作用是判断输入参数 `rtm` 是否为 `RTM_GET` 类型，如果是，就执行上述操作，否则返回 `0`。


```cpp
#endif
	if (type == RTM_GET && (rtm->rtm_addrs & (RTA_DST|RTA_GATEWAY)) ==
	    (RTA_DST|RTA_GATEWAY)) {
		sa = (struct sockaddr *)(rtm + 1);
		sa = NEXTSA(sa);
		
		if (addr_ston(sa, gw) < 0 || gw->addr_type != ADDR_TYPE_IP) {
			errno = ESRCH;
			return (-1);
		}

		if (intf_name != NULL) {
			char namebuf[IF_NAMESIZE];

			if (if_indextoname(rtm->rtm_index, namebuf) == NULL) {
				errno = ESRCH;
				return (-1);
			}
			strlcpy(intf_name, namebuf, INTF_NAME_LEN);
		}
	}
	return (0);
}

```

这段代码定义了一个名为route_t的结构体，用于表示网络路由器中的路由信息。

route_open函数用于初始化路由器并返回其内部指针，以便进一步操作。该函数首先检查是否成功申请到内存，如果没有申请成功则返回。然后，它根据所选的操作系统是否支持流式I/O来决定是否继续进行下一步操作。如果支持流式I/O，则分别尝试以打开的方式打开IP_DEV_NAME、IP_DEV_NAME和/dev/route文件以读写，如果其中任何一个操作失败则返回。如果都不支持流式I/O，则尝试打开/dev/route文件以读写，并尝试使用socket函数尝试打开一个套接字。

该函数的作用是初始化路由器并返回其内部指针，以便进一步操作。


```cpp
route_t *
route_open(void)
{
	route_t *r;
	
	if ((r = calloc(1, sizeof(*r))) != NULL) {
		r->fd = -1;
#ifdef HAVE_STREAMS_MIB2
		if ((r->ip_fd = open(IP_DEV_NAME, O_RDWR)) < 0)
			return (route_close(r));
#endif
#ifdef HAVE_STREAMS_ROUTE
		if ((r->fd = open("/dev/route", O_RDWR, 0)) < 0)
#else
		if ((r->fd = socket(PF_ROUTE, SOCK_RAW, AF_INET)) < 0)
```

这两行代码是一个简单的路由器（route_t）函数，主要作用是处理增加一条路由。函数接收一个结构体数组route_entry表示要添加的路由条目，首先通过memcpy函数将该结构体复制到一个名为rtent的临时结构体中。然后，函数调用route_msg函数来向当前路由器（route_t）发送一个通知，通知其添加了一个新的路由条目。如果函数执行成功，则返回0；否则，返回-1。


```cpp
#endif
			return (route_close(r));
	}
	return (r);
}

int
route_add(route_t *r, const struct route_entry *entry)
{
	struct route_entry rtent;
	
	memcpy(&rtent, entry, sizeof(rtent));
	
	if (route_msg(r, RTM_ADD, NULL, &rtent.route_dst, &rtent.route_gw) < 0)
		return (-1);
	
	return (0);
}

```

这段代码是一个名为`route_delete`的函数，其作用是删除一条路线条目。具体来说，该函数接受一个指向路线条目结构体的指针`entry`作为参数，并将其复制到一个名为`rtent`的内部结构体中。然后，函数调用`route_get`函数来获取该路线条目，如果返回值为负数，则说明该条目不存在，函数返回-1。如果`route_get`返回成功，则说明该条目存在，函数接下来会尝试使用`route_msg`函数来通知相关组件，最后返回0表示成功删除该条目。


```cpp
int
route_delete(route_t *r, const struct route_entry *entry)
{
	struct route_entry rtent;
	
	memcpy(&rtent, entry, sizeof(rtent));
	
	if (route_get(r, &rtent) < 0)
		return (-1);
	
	if (route_msg(r, RTM_DELETE, NULL, &rtent.route_dst, &rtent.route_gw) < 0)
		return (-1);
	
	return (0);
}

```

这段代码是一个C语言函数，名为route_get，定义在intf_init.h头文件中。它的作用是获取一个IP路由的下一个跳（如果存在）并将其存储在struct route_entry结构体中，然后将intf_name数组中的第一个元素设置为'\0'，将metric设置为0。

对于第二个if语句，如果路由的消息返回值为负数，则表示在路由表中未找到目标路由，函数将返回-1。否则，函数将初始化intf_name数组和metric，并将intf_name数组的第一个元素设置为'\0'（清除之前的值），然后将其返回值设置为0，表示成功获取下一个跳。

在if defined(HAVE_SYS_SYSCTL_H) || defined(HAVE_STREAMS_ROUTE) || defined(HAVE_GETKERNINFO)的条件下，函数使用宏定义 route_get_gateway，该宏在地址为AF_LINK的下一个跳中查找一个IP地址，并将其存储在struct route_entry结构体中。如果在该位置找到了下一个跳，函数将返回一个指向all-zero地址的指针，否则返回-1。


```cpp
int
route_get(route_t *r, struct route_entry *entry)
{
	if (route_msg(r, RTM_GET, entry->intf_name, &entry->route_dst, &entry->route_gw) < 0)
		return (-1);
	entry->intf_name[0] = '\0';
	entry->metric = 0;
	
	return (0);
}

#if defined(HAVE_SYS_SYSCTL_H) || defined(HAVE_STREAMS_ROUTE) || defined(HAVE_GETKERNINFO)
/* This wrapper around addr_ston, on failure, checks for a gateway address
 * family of AF_LINK, and if it finds one, stores an all-zero address of the
 * same type as dst. The all-zero address is a convention for same-subnet
 * routing table entries. */
```

这段代码是一个名为 "addr_ston_gateway" 的函数，它的作用是接收一个目的地址(struct addr)和一个源地址(struct sockaddr)，并返回一个整数类型的地址信息(存放在结构体变量 a 中)。

函数的实现包括以下几个步骤：

1. 调用自定义函数 addr_ston，传递目的地址(struct addr)和源地址(struct sockaddr)，并返回其返回值。
2. 如果步骤 1 的函数调用成功，就说明目的地址和源地址是有效的，返回 0。
3. 如果步骤 1 的函数调用失败，就在这里进行错误处理。会检查 sa->sa_family 是否为 AF_LINK，如果是，就说明源地址是链路层地址，需要将 a 结构体中的 addr_type 设置为目的地址的 addr_type，然后返回 0。
4. 步骤 3 的代码块中包含一个 if 语句，判断 sa->sa_family 是否为 AF_LINK。如果是，就说明源地址是链路层地址，执行以下操作：
   a. 清除 a 结构体中的 addr_type。
   b. 将 a 结构体中的 addr_type 设置为目的地址的 addr_type。
   c. 返回 0。
   d. 如果sa->sa_family 不是 AF_LINK，或者上述操作失败，就在这里进行错误处理。

整段代码的作用是接收一个目的地址和源地址，返回一个整数类型的地址信息。如果函数能够正确处理所有的错误情况，并提供有效的地址信息，则函数返回 0；否则，函数返回一个非 0 的错误代码。


```cpp
static int
addr_ston_gateway(const struct addr *dst,
	const struct sockaddr *sa, struct addr *a)
{
	int rc;

	rc = addr_ston(sa, a);
	if (rc == 0)
		return rc;

#ifdef HAVE_NET_IF_DL_H
# ifdef AF_LINK
	if (sa->sa_family == AF_LINK) {
		memset(a, 0, sizeof(*a));
		a->addr_type = dst->addr_type;
		return (0);
	}
```

这段代码是一个 C 语言程序，它实现了一个名为 route_loop 的函数，用于处理通过网络路由器(如 Linux 的 iptables)的路由规则。以下是此代码的主要部分：

1. 定义了一些常量和变量，包括两个用于指向上一个名为 route_t 的结构体变量的整数类型变量 (-1 和 __蓬，分别表示匹配任意路由器和未匹配的路由器)，以及一个名为 callback 的函数指针 (-1)，一个名为 arg 的字符串类型的变量，用于传递给函数一个额外的数据，等等。

2. 定义了一个名为 route_loop 的函数，它接受一个名为 r 的路由器结构体变量，一个名为 callback 的函数指针和一个字符串类型的参数。函数内部使用一些函数指针和变量，包括 sysctl 函数，这个函数用于通过操作系统调用来设置或获取控制台 Linux 用户界面控制器的功能。这个函数的作用是将设置或获取控制台控制器的功能作为参数传递给 route_handler 函数，并将返回值赋给 ret 变量。

3. 在 function 内部，首先定义了一个名为 rt_msghdr 的结构体变量，用于存储一个从路由器收到的数据包的内存头信息。然后，定义了一个名为 route_entry 的结构体变量，用于存储通过路由器规则匹配到的数据包的下一个跳目的地。接着定义了一个字符串类型的变量 buf，它用于存储从路由器收到的数据包的原始负载数据。然后，定义了一个字符串类型的变量 lim，它用于存储数据包缓冲区的结尾位置。接着，定义了一个字符串类型的变量 next，它用于存储数据包下一个跳目的地。

4. 在 function 内部，使用 malloc 函数在堆内存上分配了一块足够大的内存，并将它用于存储从路由器收到的数据包的原始负载数据。然后，使用 sysctl 函数将一些系统控制器的设置作为参数传递给 route_handler 函数。接着，在 function 内部遍历从路由器收到的数据包的下一个跳目的地，并将它们存储在 next 变量中。

5. 在 function 内部，使用一些字符串指针和函数指针，包括 strlen 和 0，使用这些变量计算数据包原始负载数据的长度，并将其存储在 lim 变量中。然后，使用 sysctl 函数将一些系统控制器的设置作为参数传递给 route_handler 函数。接着，在 function 内部使用一些字符串指针和函数指针，包括 strlen 和 0，使用这些变量读取从路由器收到的数据包的原始负载数据，并将其存储在 buf 变量中。

6. 在 function 内部，使用一些字符串指针和函数指针，包括 strlen 和 0，使用这些变量遍历从路由器收到的数据包的下一个跳目的地，并将它们存储在 next 变量中。然后，在 function 内部使用一些函数指针，包括 strlen 和 0，使用这些变量将原始负载数据中从第一个下一跳到第二个下一跳的所有字符串复制到 next 变量中。

7. 在 function 内部，使用一些函数指针和函数指针，包括 strlen 和 0，使用这些变量将从路由器收到的数据包的原始负载数据中从第一个下一跳到第二个下一跳的所有字符串复制到 lim 变量中。然后，在 function 内部使用一些函数指针和函数指针，包括 strlen 和 0，使用这些变量将从路由器收到的数据包的原始负载数据中从第一个下一跳到第二个下一跳的所有字符串复制到 next 变量中。

8. 在 function 内部，使用一些函数指针和函数指针，包括 strlen 和 0，使用这些变量检查从路由器收到的数据包的原始负载数据中是否存在某个字符串，并将其存储在 arg 指向的函数指针中。然后，在 function 内部使用一些函数指针和函数指针，包括 strlen 和 0，使用这些变量使用宏定义将原始负载数据中从第一个下一跳到第二个下一跳的所有字符串复制到 arg 指向的函数指针中。


```cpp
# endif
#endif

	return (-1);
}

int
route_loop(route_t *r, route_handler callback, void *arg)
{
	struct rt_msghdr *rtm;
	struct route_entry entry;
	struct sockaddr *sa;
	char *buf, *lim, *next;
	int ret;
#ifdef HAVE_SYS_SYSCTL_H
	int mib[6] = { CTL_NET, PF_ROUTE, 0, 0 /* XXX */, NET_RT_DUMP, 0 };
	size_t len;
	
	if (sysctl(mib, 6, NULL, &len, NULL, 0) < 0)
		return (-1);

	if (len == 0)
		return (0);
	
	if ((buf = malloc(len)) == NULL)
		return (-1);
	
	if (sysctl(mib, 6, buf, &len, NULL, 0) < 0) {
		free(buf);
		return (-1);
	}
	lim = buf + len;
	next = buf;
```

这段代码的作用是获取 Linux 内核中的一个名为 "HAVE_GETKERNINFO" 的定义，如果这个定义存在，则执行一系列操作，否则输出一个错误。

具体来说，代码首先调用一个名为 "getkerninfo" 的函数，这个函数接受两个参数，第一个参数是一个格式字符串，表示获取的信息类型，第二个参数是两个可选参数，用于指定信息的具体内容。这里的信息类型是 "KINFO_RT_DUMP"，表示获取系统的状态信息，包括 CPU 架构、处理器规格等。

如果这个函数返回 0，说明系统不存在这个信息，代码将输出一个错误并返回。否则，代码将返回一个指向空字符串的指针，表示成功获取到了信息。

接下来，代码分配了一块内存，用于存储获取到的信息，并更新了两个指针，分别是 Lim 和 Next，分别指向当前可用的内存位置和下一个可用的内存位置。


```cpp
#elif defined(HAVE_GETKERNINFO)
	int len = getkerninfo(KINFO_RT_DUMP,0,0,0);

	if (len == 0)
		return (0);

	if ((buf = malloc(len)) == NULL)
		return (-1);

	if (getkerninfo(KINFO_RT_DUMP,buf,&len,0) < 0) {
		free(buf);
		return (-1);
	}
	lim = buf + len;
	next = buf;
```

这段代码的作用是尝试从系统标准输入（通常是键盘或鼠标输入）中读取一条或多条路由信息，并将它们存储在一个名为 "buf" 的缓冲区中。

代码首先定义了两个结构体变量 "giarg" 和 "gp"，分别用于存储获取到的路由信息和下一个路由的等信息。

接着，代码使用 "memset" 函数将 "giarg" 初始化为一个空字符串，然后使用 "ioctl" 函数尝试从系统标准输入中读取一条或多条路由信息，并将读取到的信息存储到 "giarg" 变量中。

如果读取路由信息的过程出现错误，代码将返回一个负的值。

接下来，代码使用 "malloc" 函数在 "buf" 缓冲区中为读取到的路由信息分配足够的内存空间，并将 "giarg" 和 "gp" 指向这两个内存区域。

然后，代码使用 "ioctl" 函数再次尝试从系统标准输入中读取一条或多条路由信息，并将读取到的信息存储到下一个 "buf" 缓冲区中。如果这个过程也出现错误，代码将释放 "buf" 并返回一个负的值。

最后，代码将 "buf" 缓冲区中的内容对齐，以便从下一个 "buf" 缓冲区中直接读取。


```cpp
#else /* HAVE_STREAMS_ROUTE */
	struct rt_giarg giarg, *gp;

	memset(&giarg, 0, sizeof(giarg));
	giarg.gi_op = KINFO_RT_DUMP;

	if (ioctl(r->fd, RTSTR_GETROUTE, &giarg) < 0)
		return (-1);

	if ((buf = malloc(giarg.gi_size)) == NULL)
		return (-1);

	gp = (struct rt_giarg *)buf;
	gp->gi_size = giarg.gi_size;
	gp->gi_op = KINFO_RT_DUMP;
	gp->gi_where = buf;
	gp->gi_arg = RTF_UP | RTF_GATEWAY;

	if (ioctl(r->fd, RTSTR_GETROUTE, buf) < 0) {
		free(buf);
		return (-1);
	}
	lim = buf + gp->gi_size;
	next = buf + sizeof(giarg);
```

This function appears to be implementing the `rtree_lookup` function in FreeBSD. It takes a `struct sockaddr *` pointer to an RTree node and a result address family as arguments.

It first sets up a `struct sockaddr *` pointer called `entry` to point to the current node in the RTree. It then checks if the name of the current node has already been read. If it has, it copies the name to the `intf_name` field of the structure.

The function then checks if the destination address of the current node is the same as the root address or if it is a gateway. If the destination address is the root address or a gateway, the function continues to the next step.

If the destination address is not the root address or a gateway, the function will look for a destination address. It does this by first checking if the destination address is an IPv4 address or an IPv6 address. If it is an IPv4 address, the function will try to get the route to the destination using the `route_dst` field of the RTM node. If the function fails with `addr_stone`, the function will continue to the next step. If the destination address is an IPv6 address, the function will check if the destination address is a loopback address or a global address. If it is not a loopback address or a global address, the function will set the `entry->metric` field to a non-zero value and continue to the next step.

Finally, the function returns the result of the lookup.


```cpp
#endif
	/* This loop assumes that RTA_DST, RTA_GATEWAY, and RTA_NETMASK have the
	 * values, 1, 2, and 4 respectively. Cf. Unix Network Programming,
	 * p. 494, function get_rtaddrs. */
	for (ret = 0; next < lim; next += rtm->rtm_msglen) {
		char namebuf[IF_NAMESIZE];
		sa_family_t sfam;
		rtm = (struct rt_msghdr *)next;
		sa = (struct sockaddr *)(rtm + 1);
		/* peek at address family */
		sfam = sa->sa_family;

		if (if_indextoname(rtm->rtm_index, namebuf) == NULL)
			continue;
		strlcpy(entry.intf_name, namebuf, sizeof(entry.intf_name));

		if ((rtm->rtm_addrs & RTA_DST) == 0)
			/* Need a destination. */
			continue;
		if (addr_ston(sa, &entry.route_dst) < 0)
			continue;

		if ((rtm->rtm_addrs & RTA_GATEWAY) == 0)
			/* Need a gateway. */
			continue;
		sa = NEXTSA(sa);
		if (addr_ston_gateway(&entry.route_dst, sa, &entry.route_gw) < 0)
			continue;
		
		if (entry.route_dst.addr_type != entry.route_gw.addr_type ||
		    (entry.route_dst.addr_type != ADDR_TYPE_IP &&
			entry.route_dst.addr_type != ADDR_TYPE_IP6))
			continue;

		if (rtm->rtm_addrs & RTA_NETMASK) {
			sa = NEXTSA(sa);
			/* FreeBSD for IPv6 uses a different AF for netmasks. Force the same one. */
			sa->sa_family = sfam;
			if (addr_stob(sa, &entry.route_dst.addr_bits) < 0)
				continue;
		}

		entry.metric = 0;

		if ((ret = callback(&entry, arg)) != 0)
			break;
	}
	free(buf);
	
	return (ret);
}
```

This is a function that reads a Linux kernel message from the kernel message buffer, and performs some processing on it.

It takes in a message buffer, which is passed to the read syscall, and a handle to a function that is called in the message. The function signature for the message is specified, and the function is called with the appropriate arguments.

The function reads the message and extracts the packet address information from it. It then checks the destination address of the packet to determine which IP6 address to use for the function call.

The function iterates through the packet and extracts the address information for the destination address, storing it in the `entry.route_dst` structure. It also performs some additional processing on the address information before returning.

If the function returns an error, it returns the error code. If the function completes successfully, it returns 0.


```cpp
#elif defined(HAVE_STREAMS_MIB2)

#ifdef IRE_DEFAULT		/* This means Solaris 5.6 */
/* I'm not sure if they are compatible, though -- masaki */
#define IRE_ROUTE IRE_CACHE
#define IRE_ROUTE_REDIRECT IRE_HOST_REDIRECT
#endif /* IRE_DEFAULT */

int
route_loop(route_t *r, route_handler callback, void *arg)
{
	struct route_entry entry;
	struct strbuf msg;
	struct T_optmgmt_req *tor;
	struct T_optmgmt_ack *toa;
	struct T_error_ack *tea;
	struct opthdr *opt;
	u_char buf[8192];
	int flags, rc, rtable, ret;

	tor = (struct T_optmgmt_req *)buf;
	toa = (struct T_optmgmt_ack *)buf;
	tea = (struct T_error_ack *)buf;

	tor->PRIM_type = T_OPTMGMT_REQ;
	tor->OPT_offset = sizeof(*tor);
	tor->OPT_length = sizeof(*opt);
	tor->MGMT_flags = T_CURRENT;
	
	opt = (struct opthdr *)(tor + 1);
	opt->level = MIB2_IP;
	opt->name = opt->len = 0;
	
	msg.maxlen = sizeof(buf);
	msg.len = sizeof(*tor) + sizeof(*opt);
	msg.buf = buf;
	
	if (putmsg(r->ip_fd, &msg, NULL, 0) < 0)
		return (-1);
	
	opt = (struct opthdr *)(toa + 1);
	msg.maxlen = sizeof(buf);
	
	for (;;) {
		mib2_ipRouteEntry_t *rt, *rtend;

		flags = 0;
		if ((rc = getmsg(r->ip_fd, &msg, NULL, &flags)) < 0)
			return (-1);

		/* See if we're finished. */
		if (rc == 0 &&
		    msg.len >= sizeof(*toa) &&
		    toa->PRIM_type == T_OPTMGMT_ACK &&
		    toa->MGMT_flags == T_SUCCESS && opt->len == 0)
			break;

		if (msg.len >= sizeof(*tea) && tea->PRIM_type == T_ERROR_ACK)
			return (-1);
		
		if (rc != MOREDATA || msg.len < (int)sizeof(*toa) ||
		    toa->PRIM_type != T_OPTMGMT_ACK ||
		    toa->MGMT_flags != T_SUCCESS)
			return (-1);
		
		rtable = (opt->level == MIB2_IP && opt->name == MIB2_IP_21);
		
		msg.maxlen = sizeof(buf) - (sizeof(buf) % sizeof(*rt));
		msg.len = 0;
		flags = 0;
		
		do {
			struct sockaddr_in sin;

			rc = getmsg(r->ip_fd, NULL, &msg, &flags);
			
			if (rc != 0 && rc != MOREDATA)
				return (-1);
			
			if (!rtable)
				continue;
			
			rt = (mib2_ipRouteEntry_t *)msg.buf;
			rtend = (mib2_ipRouteEntry_t *)(msg.buf + msg.len);

			sin.sin_family = AF_INET;

			for ( ; rt < rtend; rt++) {
				if ((rt->ipRouteInfo.re_ire_type &
				    (IRE_BROADCAST|IRE_ROUTE_REDIRECT|
					IRE_LOCAL|IRE_ROUTE)) != 0 ||
				    rt->ipRouteNextHop == IP_ADDR_ANY)
					continue;
				
				entry.intf_name[0] = '\0';

				sin.sin_addr.s_addr = rt->ipRouteNextHop;
				addr_ston((struct sockaddr *)&sin,
				    &entry.route_gw);
				
				sin.sin_addr.s_addr = rt->ipRouteDest;
				addr_ston((struct sockaddr *)&sin,
				    &entry.route_dst);
				
				sin.sin_addr.s_addr = rt->ipRouteMask;
				addr_stob((struct sockaddr *)&sin,
				    &entry.route_dst.addr_bits);

				entry.metric = 0;
				
				if ((ret = callback(&entry, arg)) != 0)
					return (ret);
			}
		} while (rc == MOREDATA);
	}

	tor = (struct T_optmgmt_req *)buf;
	toa = (struct T_optmgmt_ack *)buf;
	tea = (struct T_error_ack *)buf;

	tor->PRIM_type = T_OPTMGMT_REQ;
	tor->OPT_offset = sizeof(*tor);
	tor->OPT_length = sizeof(*opt);
	tor->MGMT_flags = T_CURRENT;
	
	opt = (struct opthdr *)(tor + 1);
	opt->level = MIB2_IP6;
	opt->name = opt->len = 0;
	
	msg.maxlen = sizeof(buf);
	msg.len = sizeof(*tor) + sizeof(*opt);
	msg.buf = buf;
	
	if (putmsg(r->ip_fd, &msg, NULL, 0) < 0)
		return (-1);
	
	opt = (struct opthdr *)(toa + 1);
	msg.maxlen = sizeof(buf);
	
	for (;;) {
		mib2_ipv6RouteEntry_t *rt, *rtend;

		flags = 0;
		if ((rc = getmsg(r->ip_fd, &msg, NULL, &flags)) < 0)
			return (-1);

		/* See if we're finished. */
		if (rc == 0 &&
		    msg.len >= sizeof(*toa) &&
		    toa->PRIM_type == T_OPTMGMT_ACK &&
		    toa->MGMT_flags == T_SUCCESS && opt->len == 0)
			break;

		if (msg.len >= sizeof(*tea) && tea->PRIM_type == T_ERROR_ACK)
			return (-1);
		
		if (rc != MOREDATA || msg.len < (int)sizeof(*toa) ||
		    toa->PRIM_type != T_OPTMGMT_ACK ||
		    toa->MGMT_flags != T_SUCCESS)
			return (-1);
		
		rtable = (opt->level == MIB2_IP6 && opt->name == MIB2_IP6_ROUTE);
		
		msg.maxlen = sizeof(buf) - (sizeof(buf) % sizeof(*rt));
		msg.len = 0;
		flags = 0;
		
		do {
			struct sockaddr_in6 sin6;

			rc = getmsg(r->ip_fd, NULL, &msg, &flags);
			
			if (rc != 0 && rc != MOREDATA)
				return (-1);
			
			if (!rtable)
				continue;
			
			rt = (mib2_ipv6RouteEntry_t *)msg.buf;
			rtend = (mib2_ipv6RouteEntry_t *)(msg.buf + msg.len);

			sin6.sin6_family = AF_INET6;

			for ( ; rt < rtend; rt++) {
				if ((rt->ipv6RouteInfo.re_ire_type &
				    (IRE_BROADCAST|IRE_ROUTE_REDIRECT|
					IRE_LOCAL|IRE_ROUTE)) != 0 ||
				    memcmp(&rt->ipv6RouteNextHop, IP6_ADDR_UNSPEC, IP6_ADDR_LEN) == 0)
					continue;
				
				entry.intf_name[0] = '\0';

				sin6.sin6_addr = rt->ipv6RouteNextHop;
				addr_ston((struct sockaddr *)&sin6,
				    &entry.route_gw);
				
				sin6.sin6_addr = rt->ipv6RouteDest;
				addr_ston((struct sockaddr *)&sin6,
				    &entry.route_dst);
				
				entry.route_dst.addr_bits = rt->ipv6RoutePfxLength;
				
				if ((ret = callback(&entry, arg)) != 0)
					return (ret);
			}
		} while (rc == MOREDATA);
	}
	return (0);
}
```

This is a function definition for a `route_handler` callback that is used to handle the operation of a network route. The function takes a `callback` function pointer, which is executed whenever a route is added, modified, or deleted. The function uses the `_radix_walk` function to perform a recursive traversal of the route's children, and uses the `callback` function to compare the original and new routes.

The function has a few additional parameters:

* `fd`: file descriptor for the input file
* `rnode`: node in the route's children
* `rt`: entry in the route's children
* `sin`: socket address in the destination network
* `entry`: network route entry
* `arg`: void pointer

The function first checks if the input file is a read-only file and then reads the contents of the file into the `rnode` and `rt` variables. It then continues the traversal recursively until it reaches the end of the route's children or reaches a read-only file.

The function uses the `callback` function to compare the original and new routes. If the function returns an error, it will return the error code.

Finally, the function returns the return value of the `_radix_walk` function.


```cpp
#elif defined(HAVE_NET_RADIX_H)
/* XXX - Tru64, others? */
#include <nlist.h>

static int
_kread(int fd, void *addr, void *buf, int len)
{
	if (lseek(fd, (off_t)addr, SEEK_SET) == (off_t)-1L)
		return (-1);
	return (read(fd, buf, len) == len ? 0 : -1);
}

static int
_radix_walk(int fd, struct radix_node *rn, route_handler callback, void *arg)
{
	struct radix_node rnode;
	struct rtentry rt;
	struct sockaddr_in sin;
	struct route_entry entry;
	int ret = 0;
 again:
	_kread(fd, rn, &rnode, sizeof(rnode));
	if (rnode.rn_b < 0) {
		if (!(rnode.rn_flags & RNF_ROOT)) {
			entry.intf_name[0] = '\0';
			_kread(fd, rn, &rt, sizeof(rt));
			_kread(fd, rt_key(&rt), &sin, sizeof(sin));
			addr_ston((struct sockaddr *)&sin, &entry.route_dst);
			if (!(rt.rt_flags & RTF_HOST)) {
				_kread(fd, rt_mask(&rt), &sin, sizeof(sin));
				addr_stob((struct sockaddr *)&sin,
				    &entry.route_dst.addr_bits);
			}
			_kread(fd, rt.rt_gateway, &sin, sizeof(sin));
			addr_ston((struct sockaddr *)&sin, &entry.route_gw);
			entry.metric = 0;
			if ((ret = callback(&entry, arg)) != 0)
				return (ret);
		}
		if ((rn = rnode.rn_dupedkey))
			goto again;
	} else {
		rn = rnode.rn_r;
		if ((ret = _radix_walk(fd, rnode.rn_l, callback, arg)) != 0)
			return (ret);
		if ((ret = _radix_walk(fd, rn, callback, arg)) != 0)
			return (ret);
	}
	return (ret);
}

```

这段代码是一个用于解析IPv4地址的反向链路追踪的工具，它接受一个IPv4地址、一个反向链路追踪函数指针和一个用户数据指针作为参数。函数名为route_loop，它递归地遍历所有反向链路追踪函数，直到找到与传入IPv4地址相匹配的链路。

具体来说，这段代码实现以下步骤：

1. 初始化一个名为route_t的结构体，用于存储链路信息，但没有实际的数据成员。
2. 定义一个名为route_handler的函数，它接收一个链路信息结构体和一个用户数据指针，然后执行反向链路追踪。
3. 定义一个名为radix_node_head的结构体，用于存储链路头信息。
4. 定义一个名为nl的数组，用于存储链路头信息。
5. 打开一个名为/dev/kmem的设备文件，并初始化为只读模式，以便后续的读取操作。
6. 循环读取/dev/kmem文件中的数据，并将其存储到nl数组中。
7. 使用nl数组中的数据构建一个链路头结构体，然后递归地执行反向链路追踪函数。
8. 如果链路信息结构体包含有效的IPv4地址，则执行反向链路追踪函数，并返回0。
9. 关闭/dev/kmem设备文件并返回函数的返回值。

这段代码主要用于从设备文件中读取IPv4地址的反向链路追踪信息，用于追踪链路中IPv4地址的转换或者解析。


```cpp
int
route_loop(route_t *r, route_handler callback, void *arg)
{
	struct radix_node_head *rnh, head;
	struct nlist nl[2];
	int fd, ret = 0;

	memset(nl, 0, sizeof(nl));
	nl[0].n_name = "radix_node_head";
	
	if (knlist(nl) < 0 || nl[0].n_type == 0 ||
	    (fd = open("/dev/kmem", O_RDONLY, 0)) < 0)
		return (-1);
	
	for (_kread(fd, (void *)nl[0].n_value, &rnh, sizeof(rnh));
	    rnh != NULL; rnh = head.rnh_next) {
		_kread(fd, rnh, &head, sizeof(head));
		/* XXX - only IPv4 for now... */
		if (head.rnh_af == AF_INET) {
			if ((ret = _radix_walk(fd, head.rnh_treetop,
				 callback, arg)) != 0)
				break;
		}
	}
	close(fd);
	return (ret);
}
```

这段代码定义了一个名为route_loop的函数，以及一个名为route_close的函数。

route_loop函数的作用是包装了一个名为route_t的变量r，接收一个名为route_handler的函数指针和一个名为arg的void型别。函数内部使用ENOSYS错误码来表示错误情况，并返回-1。

route_close函数的作用是关闭通过ip_fd打开的端口。函数接收一个名为r的route_t类型的变量。

具体来说，route_close函数首先检查r是否为空。如果是，函数将输出一个错误并返回一个空指针。否则，函数首先检查r.ip_fd是否大于等于0，如果是，函数使用close函数关闭r.ip_fd。注意，函数使用了HAVE_STREAMS_MIB2头文件中的函数，表明该函数在Linux系统上有正确的实现。


```cpp
#else
int
route_loop(route_t *r, route_handler callback, void *arg)
{
	errno = ENOSYS;
	return (-1);
}
#endif

route_t *
route_close(route_t *r)
{
	if (r != NULL) {
#ifdef HAVE_STREAMS_MIB2
		if (r->ip_fd >= 0)
			close(r->ip_fd);
```

这段代码是一个 C 语言中的函数，它的作用是处理名为 "r" 的指针变量。

具体来说，这段代码首先检查 "r" 指针所指向的文件描述符是否大于等于 0，如果是，就关闭 "r" 指针所指向的文件并释放 "r" 指针。然后，函数返回一个指向 NULL 的指针。

在这段代码中，首先定义了一个名为 "r" 的指针变量，然后定义了一个名为 "close" 的函数和一个名为 "free" 的函数。接着，在 if 语句中调用 "close" 函数，最后返回一个指向 NULL 的指针。


```cpp
#endif
		if (r->fd >= 0)
			close(r->fd);
		free(r);
	}
	return (NULL);
}

```

# `libdnet-stripped/src/route-hpux.c`

这段代码是一个名为"route-hpux.c"的C文件，它定义了一个与系统相关的路由相关函数。让我们一步一步地解释它的作用。

1. 首先，它引入了一个名为"config.h"的文件。但没有输出或引用它，因此我们无法了解它定义了哪些变量或函数。

2. 它定义了一个名为"route-hpux"的函数。函数名没有给出任何上下文，但通常情况下，这个名字意味着它与路由有关。

3. 在函数体中，它首先包含了一个头文件"config.h"。这个头文件可能定义了一些与配置文件或启动参数相关的变量或函数，但同样，我们无法确定具体是哪些。

4. 接下来，它包含了一个与系统相关的函数。通过包含"sys/types.h"，"sys/ioctl.h"，"sys/mib.h"和"sys/socket.h"，它允许函数与操作系统相关的I/O操作和系统调用。

5. 接下来，是一些与套接字相关的头文件，包括"sys/select.h"。

6. 然后，是函数的一个局部变量，名为"server_fd"。这个变量似乎用于跟踪服务器套接字的大小，但我们无法确定它的确切目的。

7. 接下来，是另一个局部变量，名为"route_len"。我们无法确定这个变量的具体作用，因为我们不知道这个函数与路由表或任何其他系统组件的关系。

8. 然后，是函数的一个循环。通过循环，它接收一个IP地址和端口号作为参数。

9. 在循环内部，它使用系统调用"connect"在服务器套接字上创建一个新套接字并绑定到服务器端口。

10. 接下来，它使用另一个系统调用"bind"将服务器套接字与客户端套接字（即服务器套接字的客户端端口）绑定。

11. 然后，函数创建一个名为"client_fd"的文件描述符，用于接收客户端的输入数据。

12. 最后，它复制一个名为"route_table"的文件到客户端套接字。我们无法确定这个文件的内容，因此我们无法知道函数如何使用它。

总的来说，这段代码定义了一个与系统相关的路由函数。它通过使用操作系统调用和文件描述符等机制，在服务器和客户端之间传输数据，并创建一个或多个套接字，以便在客户端套接字上接收数据并将其复制到服务器套接字中。


```cpp
/*
 * route-hpux.c
 *
 * Copyright (c) 2000 Dug Song <dugsong@monkey.org>
 *
 * $Id: route-hpux.c 483 2004-01-14 04:52:11Z dugsong $
 */

#include "config.h"

#include <sys/types.h>
#include <sys/ioctl.h>
#include <sys/mib.h>
#include <sys/socket.h>

```

这段代码是一个网络编程中的路由函数，它的作用是处理目标主机地址（即目的IP地址）为IPv6地址的入方向（IN）或出方向（OUT）路由。

具体来说，这段代码包括以下几个部分：

1. 引入了net、errno、stdio、stdlib、string、unistd等头文件，这些头文件包含了一些用于网络编程的标准函数和头文件，对代码的运行没有贡献，但可能需要从网络编程库中动态链接。

2. 定义了一个名为ADDR_ICHOST的函数，它接收一个IPv6地址（即目标主机地址）和一个路由表（即下一跳路由表）作为参数。它的作用是判断目标主机地址的地址类型（IPv4或IPv6）以及对应的地址位数，然后从路由表中查找相应的下一跳路由。

3. 在函数内部，对ADDR_ICHOST函数进行了修改，增加了对IPv4地址的处理，使得函数能够处理IPv4地址作为目标主机地址的情况。

4. 在main函数中，定义了一个名为example的函数，它接收一个IPv6地址和一个路由表作为参数。然后，它会根据给定的路由表执行一条入方向（或出方向）路由，并将结果返回。

5. 在example函数中，首先通过socket命令创建了一个IPv6地址，然后构造了一个IPv6地址对，将目的地址（即example函数传递给的IPv6地址）与下一跳路由表（example函数中给出的路由表）进行比较，最后通过系统调用net_route_lookup函数执行入方向或出方向路由，并将结果返回。


```cpp
#include <net/route.h>

#include <errno.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

#include "dnet.h"

#define ADDR_ISHOST(a)	(((a)->addr_type == ADDR_TYPE_IP &&	\
			  (a)->addr_bits == IP_ADDR_BITS) ||	\
			 ((a)->addr_type == ADDR_TYPE_IP6 &&	\
			  (a)->addr_bits == IP6_ADDR_BITS))

```

这段代码定义了一个名为 `route_handle` 的结构体，它包含一个整数类型的成员 `fd`。

接着定义了一个名为 `route_open` 的函数，它接受一个空指针并返回一个指向 `route_t` 类型的指针 `r`。函数的实现包括以下步骤：

1. 检查 `r` 是否为空指针，如果是，则直接返回。
2. 如果 `r` 为空指针，则创建一个新指针 `r`，该指针包含一个整数类型的成员 `fd`，可以使用调用 `socket` 函数并传入 `AF_INET` 参数、`SOCK_DGRAM` 参数和 0 作为协议类型参数来创建一个套接字。
3. 返回新创建的指针 `r`。

整函数 `route_open` 可以用来在应用程序中创建和管理路径名。它创建一个套接字，并将它绑定到本机的 IP 地址，使得可以通过该套接字发送和接收数据包。它返回一个指向结构体 `route_t` 的指针，该结构体定义了用于表示一条路径的信息。


```cpp
struct route_handle {
	int	fd;
};

route_t *
route_open(void)
{
	route_t *r;

	if ((r = calloc(1, sizeof(*r))) != NULL) {
		if ((r->fd = socket(AF_INET, SOCK_DGRAM, 0)) < 0)
			return (route_close(r));
	}
	return (r);
}

```

这段代码是一个用于 Linux 路由表（route）的函数，名为 route_add。函数接收两个参数：一个指向 route_entry 结构的指针 entry，以及一个指向 struct rtentry 的指针 rt。函数的作用是在路由表中添加一条新的路由，如果添加成功，则返回状态码 0。

函数的具体实现包括以下几个步骤：

1. 定义一个名为 rt 的结构体，用于存储新的路由条目。
2. 定义一个名为 dst 的结构体，用于存储目标网络的地址信息。
3. 如果目标网络是主机（即地址为 0），则直接将 entry->route_dst 作为 rt.rt_dst 成员。否则，执行以下操作：

a. 如果目标网络是网络地址（即地址以 0 结尾），则执行 addr_net 函数并将网络地址转换为二进制格式，然后将其作为 rt.rt_dst 成员。

b. 否则，执行 address_ntos 函数，将从链中获取的地址转换为网络格式，然后将其作为 rt.rt_dst 成员。如果转换失败，函数将返回 -1。

4. 如果函数可以成功调用 system_call 函数，通过调用 ioctl 函数，设置新的路由条目为 rt.rt_dst，则返回 0。


```cpp
int
route_add(route_t *r, const struct route_entry *entry)
{
	struct rtentry rt;
	struct addr dst;
	
	memset(&rt, 0, sizeof(rt));
	rt.rt_flags = RTF_UP | RTF_GATEWAY;

	if (ADDR_ISHOST(&entry->route_dst)) {
		rt.rt_flags |= RTF_HOST;
		memcpy(&dst, &entry->route_dst, sizeof(dst));
	} else
		addr_net(&entry->route_dst, &dst);

	if (addr_ntos(&dst, &rt.rt_dst) < 0 ||
	    addr_ntos(&entry->route_gw, &rt.rt_gateway) < 0 ||
	    addr_btom(entry->route_dst.addr_bits, &rt.rt_subnetmask,
		IP_ADDR_LEN) < 0)
		return (-1);
	
	return (ioctl(r->fd, SIOCADDRT, &rt));
}

```

这段代码是一个名为`route_delete`的函数，其作用是删除一条网络路由。它接受两个参数：一个指向路由表项（route_t）的指针`r`和一个指向路由条目（route_entry）的指针`entry`。

函数首先定义了一个名为`rt`的结构体，用于存储删除操作后的路由表项。然后定义了一个名为`dst`的结构体，用于存储目标地址。

接下来，函数首先检查输入的目标地址是否为主机地址。如果是，函数将`RTF_HOST`路由类型添加到`rt`结构体的`rt_flags`位中，并使用`memcpy`函数将目标地址和`entry->route_dst`的地址复制到`dst`中。如果不是，函数将`RTF_UP`路由类型添加到`rt`结构体的`rt_flags`位中，并尝试将目标地址转换为网络地址。如果转换失败，函数返回负数。

最后，函数调用`ioctl`函数并传递两个参数：`r`和`entry->route_dst`。函数使用`SIOCDELRT`系统调用，并将`rt`结构体作为第一个参数传递，以删除指定路由表项。如果删除成功，函数返回0。


```cpp
int
route_delete(route_t *r, const struct route_entry *entry)
{
	struct rtentry rt;
	struct addr dst;

	memset(&rt, 0, sizeof(rt));
	rt.rt_flags = RTF_UP;
	
	if (ADDR_ISHOST(&entry->route_dst)) {
		rt.rt_flags |= RTF_HOST;
		memcpy(&dst, &entry->route_dst, sizeof(dst));
	} else
		addr_net(&entry->route_dst, &dst);
	
	if (addr_ntos(&dst, &rt.rt_dst) < 0 ||
	    addr_btom(entry->route_dst.addr_bits, &rt.rt_subnetmask,
		IP_ADDR_LEN) < 0)
		return (-1);
	
	return (ioctl(r->fd, SIOCDELRT, &rt));
}

```

这段代码是一个C语言函数，名为`route_get`，它用于获取通过某个IP路由器到达某个目的IP地址的下一跳IP地址。

该函数接收两个参数：一个指向`route_t`结构体的指针`r`，以及一个指向`route_entry`结构体的指针`entry`。函数内部首先定义了一个`struct rtreq`类型的变量`rtr`，用于存储下一跳IP地址的信息。

接着，函数内部创建了一个`struct rtreq`类型的变量`rtr`，其中包含下一跳IP地址的地址字段。然后，根据输入的目标IP地址是否为`IP_ADDR_ANY`来设置默认下一跳IP地址。

接下来，函数使用`memcpy`函数将下一跳IP地址的地址字段复制到`rtr.rtr_destaddr`指向的内存空间中。然后，根据输入的目标IP地址的位数长度，将`rtr.rtr_subnetmask`设置为适当的网络前缀。

接着，函数尝试使用`ioctl`函数请求路由器接口`r->fd`更新其状态，如果该操作失败，则返回错误代码。

最后，函数将`entry->intf_name`和`entry->route_gw.addr_type`、`entry->route_gw.addr_bits`和`entry->route_gw.addr_ip`设置为适当的初始化值，然后返回0表示成功。


```cpp
int
route_get(route_t *r, struct route_entry *entry)
{
	struct rtreq rtr;

	memset(&rtr, 0, sizeof(rtr));

	/* XXX - gross hack for default route */
	if (entry->route_dst.addr_ip == IP_ADDR_ANY) {
		rtr.rtr_destaddr = htonl(0x60060606);
		rtr.rtr_subnetmask = 0xffffffff;
	} else {
		memcpy(&rtr.rtr_destaddr, &entry->route_dst.addr_ip,
		    IP_ADDR_LEN);
		if (entry->route_dst.addr_bits < IP_ADDR_BITS)
			addr_btom(entry->route_dst.addr_bits,
			    &rtr.rtr_subnetmask, IP_ADDR_LEN);
	}
	if (ioctl(r->fd, SIOCGRTENTRY, &rtr) < 0)
		return (-1);

	if (rtr.rtr_gwayaddr == 0) {
		errno = ESRCH;
		return (-1);
	}
	entry->intf_name[0] = '\0';
	entry->route_gw.addr_type = ADDR_TYPE_IP;
	entry->route_gw.addr_bits = IP_ADDR_BITS;
	memcpy(&entry->route_gw.addr_ip, &rtr.rtr_gwayaddr, IP_ADDR_LEN);
	entry->metric = 0;
	
	return (0);
}

```



route_loop 是一个用于遍历路由表并调用 route_handler 函数的函数。它有以下参数：

- r: 一个指向 route_t 结构体的指针，包含了一个完整的二层路由表。
- callback: 一个回调函数，用于处理每个路由条目。该函数的输入参数是一个 struct_ipRouteCb 类型的数据，用于在回调中接收路由条目。
- arg: 一个指向 void 类型的指针，用于在回调中传递给 route_handler 函数的参数。

route_loop 会遍历二层路由表，并对于每个条目，调用 callback 函数来处理该条目。如果 callback 函数返回负数，那么就停止遍历该条目，否则继续遍历。

下面是一个 example 用法：

```cpp
int ret;
route_t *rt;
route_handler_t cb;
void *arg;

ret = route_loop(rt, cb, arg);
if (ret != 0)
   return;

cb = (route_handler_t)arg;
ret = cb(rt, arg);
```

这个 example  会调用 route_handler 函数，并把 arg 传递给该函数。如果 route_handler 函数返回负数，那么退出 route_loop，否则继续遍历。


```cpp
#define MAX_RTENTRIES	256	/* XXX */

int
route_loop(route_t *r, route_handler callback, void *arg)
{
	struct nmparms nm;
	struct route_entry entry;
	mib_ipRouteEnt rtentries[MAX_RTENTRIES];
	int fd, i, n, ret;
	
	if ((fd = open_mib("/dev/ip", O_RDWR, 0 /* XXX */, 0)) < 0)
		return (-1);
	
	nm.objid = ID_ipRouteTable;
	nm.buffer = rtentries;
	n = sizeof(rtentries);
	nm.len = &n;
	
	if (get_mib_info(fd, &nm) < 0) {
		close_mib(fd);
		return (-1);
	}
	close_mib(fd);

	n /= sizeof(*rtentries);
	ret = 0;
	
	for (i = 0; i < n; i++) {
		if (rtentries[i].Type != NMDIRECT &&
		    rtentries[i].Type != NMREMOTE)
			continue;
		
		entry.intf_name[0] = '\0';
		entry.route_dst.addr_type = entry.route_gw.addr_type = ADDR_TYPE_IP;
		entry.route_dst.addr_bits = entry.route_gw.addr_bits = IP_ADDR_BITS;
		entry.route_dst.addr_ip = rtentries[i].Dest;
		addr_mtob(&rtentries[i].Mask, IP_ADDR_LEN,
		    &entry.route_dst.addr_bits);
		entry.route_gw.addr_ip = rtentries[i].NextHop;
		entry.metric = 0;

		if ((ret = callback(&entry, arg)) != 0)
			break;
	}
	return (ret);
}

```

这段代码定义了一个名为route_t的结构体，用于表示网络路由信息。

接下来，定义了一个名为route_close的函数，该函数接收一个route_t类型的指针变量r，用于存储网络路由信息。

函数内部首先检查r是否为空，如果是，则执行以下语句：

if (r != NULL) {

如果r指向的fd通道的值为正数，则关闭该fd并释放r指向的内存，最后返回一个指向空指针的NULL。

}

如果r指向的fd通道的值为0或者函数外没有传入fd参数，则不关闭该fd，并且不释放r指向的内存，此时函数内部返回一个空指针NULL。

route_close函数的作用是关闭通过route_t结构体中的fd指针所表示的文件描述符，并释放route_t结构体指针变量r的内存。


```cpp
route_t *
route_close(route_t *r)
{
	if (r != NULL) {
		if (r->fd >= 0)
			close(r->fd);
		free(r);
	}
	return (NULL);
}

```

# `libdnet-stripped/src/route-linux.c`

这段代码是一个 Linux 路由器的路由引擎，它的主要作用是处理数据包，并将数据包从源地址到目的地址进行传递。

具体来说，这段代码包括以下几个主要部分：

1. 引入了 "route-linux.h" 头文件，这个头文件定义了路程器引擎的一些函数和结构体，这个文件是由 TCP-Routing 项目开发的。
2. 定义了一系列用于处理数据包的函数，包括 "parse-route-header" 和 "route-lookup-npf-packet" 等函数。
3. 初始化了一个 " routes" 结构体，用于存储路由表。
4. 读取一个数据包，并调用 "parse-route-header" 函数来解析出该数据包的路由信息，然后使用 "route-lookup-npf-packet" 函数来查找该数据包的目的地。
5. 如果目的地在路由表中，就使用 "write-packet" 函数将数据包发送出去，否则就执行 "print-packet-error" 函数并输出错误信息。

这段代码的作用是处理数据包，并决定数据包的下一跳目的地，然后将这些信息存储在路由表中，以便下一跳设备可以接收和处理这些数据包。


```cpp
/*
 * route-linux.c
 *
 * Copyright (c) 2000 Dug Song <dugsong@monkey.org>
 *
 * $Id: route-linux.c 619 2006-01-15 07:33:29Z dugsong $
 */

#include "config.h"

#include <sys/types.h>
#include <sys/ioctl.h>
#include <sys/socket.h>
#include <sys/uio.h>

```

这段代码是一个 Linux 系统的 C 语言脚本，它定义了一系列网络相关的头文件，以及一些常量。

其中包括：

1. asm/types.h：定义了 x86 架构下的types，如文件描述符、字符串、和算术类型等。
2. net/if.h：定义了 Interface 结构体，用于表示网络接口信息。
3. netinet/in.h：定义了 inet 结构体，用于表示 IPv4 地址。
4. linux/netlink.h：定义了 Network Link 结构体，用于表示网络链路信息。
5. linux/rtnetlink.h：定义了 Runtime Link 结构体，用于表示操作系统中的网络链路信息。
6. net/route.h：定义了 Route 结构体，用于表示路由信息。
7. ctype.h：定义了CTYPE 头文件，用于支持多字节字符串的输入输出。
8. errno.h：定义了 errno 头文件，用于表示错误信息。
9. stdio.h：定义了 stdio 函数，用于在标准输入和标准输出上输出字符串和变量。
10. stdlib.h：定义了系统调用函数，用于调用 C 语言标准库中的函数。
11. string.h：定义了 str 函数，用于将字符串转换为小写。
12. unistd.h：定义了 unistd 函数，用于调用 C 语言标准库中的函数。


```cpp
#include <asm/types.h>
#include <net/if.h>
#include <netinet/in.h>
#include <linux/netlink.h>
#include <linux/rtnetlink.h>

#include <net/route.h>

#include <ctype.h>
#include <errno.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

```

这段代码定义了两个宏，用于指定网络接口的地址类型。第一个宏 ADDR_ISHOST(a) 判断一个网络接口是否为 IP 地址，如果是，则判断其使用的地址类型是否为 IPv6 地址。第二个宏 PROC_ROUTE_FILE 和 PROC_IPV6_ROUTE_FILE 用于指定用于获取路由信息的文件路径。最后，定义了一个名为 route_handle 的结构体，用于表示网络接口的路由信息。该结构体包含两个成员变量，fd 表示文件描述符，nlfd 表示下一个文件描述符。


```cpp
#include "dnet.h"

#define ADDR_ISHOST(a)	(((a)->addr_type == ADDR_TYPE_IP &&	\
			  (a)->addr_bits == IP_ADDR_BITS) ||	\
			 ((a)->addr_type == ADDR_TYPE_IP6 &&	\
			  (a)->addr_bits == IP6_ADDR_BITS))

#define PROC_ROUTE_FILE		"/proc/net/route"
#define PROC_IPV6_ROUTE_FILE	"/proc/net/ipv6_route"

struct route_handle {
	int	 fd;
	int	 nlfd;
};

```

这段代码定义了一个名为 route_t 的结构体，用于表示网络路由信息。该结构体包含一个名为 r 的指向 route_open 函数的指针，以及一个名为 rc 的布尔变量。

route_open 函数的作用是在 Linux 系统中创建一个路由表项，以便在数据包传输过程中进行转发。它需要进行一系列的系统调用操作来完成这个任务，包括创建套接字、绑定套接字、创建套接字 Buffer 等。具体实现如下：

1. 如果已经分配好了内存空间，就创建一个名为 r 的路由表项，并将其 fd 成员赋值为 -1,nlfd 成员赋值为 -1。

2. 如果创建套接字失败，就返回前一个返回值。

3. 如果创建套接字成功，就创建一个名为 snl 的 socket 结构体，并将其用于绑定到网络接口上。

4. 如果绑定套接字失败，就返回前一个返回值。

5. 调用 bind 函数将 socket 中的 socket 协议的 family 设为 AF_NETLINK，同时将 socket 中的协议的类型设为 SOCK_RAW，以便能够接收和发送数据包。

6. 调用 bind 函数将套接字和 socket 中的 socket 中的 socket 协议的 family 设为 AF_INET，同时将 socket 中的协议的类型设为 SOCK_DGRAM，以便能够接收和发送数据包。

7. 如果调用 bind 函数成功，就返回前一个返回值。


```cpp
route_t *
route_open(void)
{
	struct sockaddr_nl snl;
	route_t *r;

	if ((r = calloc(1, sizeof(*r))) != NULL) {
		r->fd = r->nlfd = -1;
		
		if ((r->fd = socket(AF_INET, SOCK_DGRAM, 0)) < 0)
			return (route_close(r));
		
		if ((r->nlfd = socket(AF_NETLINK, SOCK_RAW,
			 NETLINK_ROUTE)) < 0)
			return (route_close(r));
		
		memset(&snl, 0, sizeof(snl));
		snl.nl_family = AF_NETLINK;
		
		if (bind(r->nlfd, (struct sockaddr *)&snl, sizeof(snl)) < 0)
			return (route_close(r));
	}
	return (r);
}

```

这段代码是一个用于 Linux 路由表（rt）的函数，名为 route_add。其功能是接受一个路由表项（route_entry）作为参数，然后向该路由表项添加一个新的路由条目。

具体来说，这段代码实现以下几个步骤：

1. 初始化一个名为 rt 的结构体，其中包含rt 标志位（RTF_UP | RTF_GATEWAY）以及一个指向路径条目的指针。
2. 如果目的地址（route_entry.route_dst）已经存在于路由表中，则执行以下操作：
   a. 设置 rt 标志位为 RTF_HOST，即将路由更新为 host（主机）类型。
   b. 复制目的地址到结构体 rt 的 rt_dst 成员中。

3. 如果目的地址不正确或者无法访问，则返回一个负的值。具体来说，执行以下操作：
   a. 尝试使用 addr_net 函数将目的地址设置为正确的网络地址。
   b. 使用 addr_ntos 函数将目的地址转换为网络地址。
   c. 如果同时执行 a 和 b 操作后仍然无法访问目的地址，则执行以下操作：
      a. 检查目的地址是否为全零（NULL）字节。如果是，则表示目的地址为无效地址，应该返回一个负的值。
      b. 如果 a 和 b 操作都失败，则执行以下操作：
        c. 检查目的地址是否在网络中。如果目的地址在网络中但仍然无法访问，则执行以下操作：
          d. 更新路由表并返回。

4. 使用 ioctl 函数向路由表发送添加路由条目的请求。这个函数会返回一个负的错误码，如果失败，则表示添加路由条目失败。


```cpp
int
route_add(route_t *r, const struct route_entry *entry)
{
	struct rtentry rt;
	struct addr dst;

	memset(&rt, 0, sizeof(rt));
	rt.rt_flags = RTF_UP | RTF_GATEWAY;

	if (ADDR_ISHOST(&entry->route_dst)) {
		rt.rt_flags |= RTF_HOST;
		memcpy(&dst, &entry->route_dst, sizeof(dst));
	} else
		addr_net(&entry->route_dst, &dst);
	
	if (addr_ntos(&dst, &rt.rt_dst) < 0 ||
	    addr_ntos(&entry->route_gw, &rt.rt_gateway) < 0 ||
	    addr_btos(entry->route_dst.addr_bits, &rt.rt_genmask) < 0)
		return (-1);
	
	return (ioctl(r->fd, SIOCADDRT, &rt));
}

```

这段代码是一个名为`route_delete`的函数，属于`route_t`结构的成员。

它的作用是删除一条路由，并返回一个IOCTL return code。

具体来说，这段代码实现以下几个步骤：

1. 创建一个名为`rt`的`rtentry`结构体，其中包含`rt_flags`和`rt_dst`成员变量。
2. 将`rt_flags`设置为`RTF_UP`，表示当前路由处于可用状态。
3. 如果目标地址是一个网络地址，则将`dst`字段设置为该地址。否则，将`dst`字段设置为`entry->route_dst`的下一个地址。
4. 如果`dst`是一个网络地址，尝试将其转换为`addr_net`函数的返回值，并将其设置为`rt.rt_dst`。
5. 如果`dst`是一个广播地址，或者它的地址作为组播地址发送，尝试将其转换为`addr_btos`函数的返回值，并将其设置为`rt.rt_genmask`。
6. 如果步骤2至5中的任何一步出现错误，返回一个负的IOCTL return code。
7. 最后，通过调用`ioctl`函数`r->fd`，并传递一个指向`rt`的`SIOCDELRT`ioctl格式的参数，成功删除路由，并将结果返回给调用者。


```cpp
int
route_delete(route_t *r, const struct route_entry *entry)
{
	struct rtentry rt;
	struct addr dst;
	
	memset(&rt, 0, sizeof(rt));
	rt.rt_flags = RTF_UP;

	if (ADDR_ISHOST(&entry->route_dst)) {
		rt.rt_flags |= RTF_HOST;
		memcpy(&dst, &entry->route_dst, sizeof(dst));
	} else
		addr_net(&entry->route_dst, &dst);
	
	if (addr_ntos(&dst, &rt.rt_dst) < 0 ||
	    addr_btos(entry->route_dst.addr_bits, &rt.rt_genmask) < 0)
		return (-1);
	
	return (ioctl(r->fd, SIOCDELRT, &rt));
}

```

This is a C function that determines whether a message received from a BGP router is valid and can be processed by the local router. It takes in a single parameter `nmsg` which is a pointer to a BGP `nmsg` structure, containing information about the received message.

The function first checks if the length of the `nmsg` structure is less than the size of an `nmsg` structure, or if the `nlmsg_len` field is negative or greater than the maximum value for an `nlmsg_len` field. If any of these conditions are met, the function returns `-1` to indicate an error.

Next, the function checks if the `nlmsg_type` field is set to `NLMSG_ERROR`, in which case the function returns `-1`. If `nlmsg_type` is not set to `NLMSG_ERROR`, the function then checks if the message is an error message by checking the `nlmsg_len` field against the maximum length of an error message. If the `nlmsg_len` is less than the maximum length, the function returns `0` to indicate that the message is not an error message and can be processed. If the `nlmsg_len` is greater than or equal to the maximum length, the function returns `-1` to indicate an error.

If the `nlmsg_type` is `NLMSG_ERROR`, the function returns `-1` to indicate that the message is an error message and cannot be processed. If the `nlmsg_type` is not set to `NLMSG_ERROR`, the function then checks if the message is an OIF message by checking the `nlmsg_type` field against the valid OIF message types. If the `nlmsg_type` is not set to an OIF message type, the function returns `0` to indicate that the message is not an OIF message and can be processed. If the `nlmsg_type` is set to an OIF message type, the function copies the `intf_name` field from the `nmsg` structure to a temporary buffer and attempts to get a printable name of the OIF from the `intf_index` field in the `nmsg` structure. If the `intf_index` is not found or the `intf_name` cannot be returned, the function returns `-1` to indicate an error. If the `nlmsg_type` is set to `NLMSG_ERROR`, the function returns `-1` to indicate that the message is an error message and cannot be processed. If the `nlmsg_type` is not set to `NLMSG_ERROR` and the message is not an OIF message, the function returns `0` to indicate that the message is not an error message and can be processed. If the `nlmsg_type` is set to `NLMSG_ERROR`, the function returns `-1` to indicate that the message is an error message and cannot be processed. If the `nlmsg_type` is not set to `NLMSG_ERROR` and the message is an OIF message, the function returns `0` to indicate that the message is not an OIF message and can be processed. If the `nlmsg_type` is set to `NLMSG_ERROR`, the function returns `-1` to indicate that the message is an error message and cannot be processed. If the `nlmsg_type` is not set to `NLMSG_ERROR` and the message is not an error message or an OIF message, the function returns `0` to indicate that the message is not an error message and can be processed. If the `nlmsg_type` is set to `NLMSG_ERROR`, the function returns `-1` to indicate that the message is an error message and cannot be processed. If the `nlmsg_type` is not set to `NLMSG_ERROR` and the message is an OIF message, the function returns `0` to indicate that the message is not an OIF message and can be processed. If the `nlmsg_type` is set to `NLMSG_ERROR`, the function returns `-1` to indicate that the message is an error message and cannot be processed. If the `nlmsg_type` is not set to `NLMSG_ERROR` and


```cpp
int
route_get(route_t *r, struct route_entry *entry)
{
	static int seq;
	struct nlmsghdr *nmsg;
	struct rtmsg *rmsg;
	struct rtattr *rta;
	struct sockaddr_nl snl;
	struct iovec iov;
	struct msghdr msg;
	u_char buf[512];
	int i, af, alen;

	switch (entry->route_dst.addr_type) {
	case ADDR_TYPE_IP:
		af = AF_INET;
		alen = IP_ADDR_LEN;
		break;
	case ADDR_TYPE_IP6:
		af = AF_INET6;
		alen = IP6_ADDR_LEN;
		break;
	default:
		errno = EINVAL;
		return (-1);
	}
	memset(buf, 0, sizeof(buf));

	nmsg = (struct nlmsghdr *)buf;
	nmsg->nlmsg_len = NLMSG_LENGTH(sizeof(*nmsg)) + RTA_LENGTH(alen);
	nmsg->nlmsg_flags = NLM_F_REQUEST;
	nmsg->nlmsg_type = RTM_GETROUTE;
	nmsg->nlmsg_seq = ++seq;

	rmsg = (struct rtmsg *)(nmsg + 1);
	rmsg->rtm_family = af;
	rmsg->rtm_dst_len = entry->route_dst.addr_bits;
	
	rta = RTM_RTA(rmsg);
	rta->rta_type = RTA_DST;
	rta->rta_len = RTA_LENGTH(alen);

	/* XXX - gross hack for default route */
	if (af == AF_INET && entry->route_dst.addr_ip == IP_ADDR_ANY) {
		i = htonl(0x60060606);
		memcpy(RTA_DATA(rta), &i, alen);
	} else
		memcpy(RTA_DATA(rta), entry->route_dst.addr_data8, alen);
	
	memset(&snl, 0, sizeof(snl));
	snl.nl_family = AF_NETLINK;

	iov.iov_base = nmsg;
	iov.iov_len = nmsg->nlmsg_len;
	
	memset(&msg, 0, sizeof(msg));
	msg.msg_name = &snl;
	msg.msg_namelen = sizeof(snl);
	msg.msg_iov = &iov;
	msg.msg_iovlen = 1;
	
	if (sendmsg(r->nlfd, &msg, 0) < 0)
		return (-1);

	iov.iov_base = buf;
	iov.iov_len = sizeof(buf);
	
	if ((i = recvmsg(r->nlfd, &msg, 0)) <= 0)
		return (-1);

	if (nmsg->nlmsg_len < (int)sizeof(*nmsg) || nmsg->nlmsg_len > i ||
	    nmsg->nlmsg_seq != seq) {
		errno = EINVAL;
		return (-1);
	}
	if (nmsg->nlmsg_type == NLMSG_ERROR)
		return (-1);
	
	i -= NLMSG_LENGTH(sizeof(*nmsg));
	
	entry->route_gw.addr_type = ADDR_TYPE_NONE;
	entry->intf_name[0] = '\0';
	for (rta = RTM_RTA(rmsg); RTA_OK(rta, i); rta = RTA_NEXT(rta, i)) {
		if (rta->rta_type == RTA_GATEWAY) {
			entry->route_gw.addr_type = entry->route_dst.addr_type;
			memcpy(entry->route_gw.addr_data8, RTA_DATA(rta), alen);
			entry->route_gw.addr_bits = alen * 8;
		} else if (rta->rta_type == RTA_OIF) {
			char ifbuf[IFNAMSIZ];
			char *p;
			int intf_index;

			intf_index = *(int *) RTA_DATA(rta);
			p = if_indextoname(intf_index, ifbuf);
			if (p == NULL)
				return (-1);
			strlcpy(entry->intf_name, ifbuf, sizeof(entry->intf_name));
		}
	}
	if (entry->route_gw.addr_type == ADDR_TYPE_NONE) {
		errno = ESRCH;
		return (-1);
	}
	
	return (0);
}

```

这段代码是一个工具函数，用于解析 PXML 文件中的邻居信息。它主要实现了以下功能：

1. 读取 PXML 文件中的邻居信息。
2. 解析邻居信息，包括 RTF (RFC 2846) 类型和邻居地址。
3. 计算邻居的距离（单位是字节）。
4. 输出邻居信息。

代码中定义了以下变量：

- `ip_addr`：目标 IP 地址。
- `邻居_ip`：邻居的 IP 地址。
- `邻居_port`：邻居的端口号（80 或 443）。
- `parsed_邻居`：解析后的邻居信息。
- `cmd_line`：命令行参数。
- `filename`：输入的 PXML 文件的路径。
- `iflags`：输入的文件模式（0 表示不读取模式，1 表示只读取模式，2 表示只写入模式，4 表示同时模式）。
- `callback`：回调函数，用于处理邻居解析的参数。
- `fp`：读取 PXML 文件的文件指针。

函数中实现了以下功能：

1. 读取 PXML 文件中的邻居信息。
2. 解析邻居信息，包括 RTF 类型和邻居地址。
3. 计算邻居的距离（单位是字节）。
4. 输出邻居信息。
5. 如果解析邻居信息的过程中出现错误，使用 `ret` 变量返回，此时函数结束。

函数中主要实现了一个 while 循环，用于不断读取文件中的邻居信息，并将其输出。每次循环，先读取一个邻居邻居的 IP 地址和端口号，然后判断读取模式，接着解析获取邻居的 RTF 类型，再计算邻居的距离，最后输出邻居信息。如果解析失败，函数将返回，函数结束。


```cpp
int
route_loop(route_t *r, route_handler callback, void *arg)
{
	FILE *fp;
	struct route_entry entry;
	char buf[BUFSIZ];
	char ifbuf[16];
	int ret = 0;

	if ((fp = fopen(PROC_ROUTE_FILE, "r")) != NULL) {
		int i, iflags, refcnt, use, metric, mss, win, irtt;
		uint32_t mask;
		
		while (fgets(buf, sizeof(buf), fp) != NULL) {
			i = sscanf(buf, "%15s %X %X %X %d %d %d %X %d %d %d\n",
			    ifbuf, &entry.route_dst.addr_ip,
			    &entry.route_gw.addr_ip, &iflags, &refcnt, &use,
			    &metric, &mask, &mss, &win, &irtt);
			
			if (i < 11 || !(iflags & RTF_UP))
				continue;
		
			strlcpy(entry.intf_name, ifbuf, sizeof(entry.intf_name));

			entry.route_dst.addr_type = entry.route_gw.addr_type =
			    ADDR_TYPE_IP;
		
			if (addr_mtob(&mask, IP_ADDR_LEN,
				&entry.route_dst.addr_bits) < 0)
				continue;
			
			entry.route_gw.addr_bits = IP_ADDR_BITS;
			entry.metric = metric;
			
			if ((ret = callback(&entry, arg)) != 0)
				break;
		}
		fclose(fp);
	}
	if (ret == 0 && (fp = fopen(PROC_IPV6_ROUTE_FILE, "r")) != NULL) {
		char s[33], d[8][5], n[8][5];
		int i, iflags, metric;
		u_int slen, dlen;
		
		while (fgets(buf, sizeof(buf), fp) != NULL) {
			i = sscanf(buf, "%04s%04s%04s%04s%04s%04s%04s%04s %02x "
			    "%32s %02x %04s%04s%04s%04s%04s%04s%04s%04s "
			    "%x %*x %*x %x %15s",
			    d[0], d[1], d[2], d[3], d[4], d[5], d[6], d[7],
			    &dlen, s, &slen,
			    n[0], n[1], n[2], n[3], n[4], n[5], n[6], n[7],
			    &metric, &iflags, ifbuf);
			
			if (i < 21 || !(iflags & RTF_UP))
				continue;

			strlcpy(entry.intf_name, ifbuf, sizeof(entry.intf_name));

			snprintf(buf, sizeof(buf), "%s:%s:%s:%s:%s:%s:%s:%s/%d",
			    d[0], d[1], d[2], d[3], d[4], d[5], d[6], d[7],
			    dlen);
			addr_aton(buf, &entry.route_dst);
			snprintf(buf, sizeof(buf), "%s:%s:%s:%s:%s:%s:%s:%s/%d",
			    n[0], n[1], n[2], n[3], n[4], n[5], n[6], n[7],
			    IP6_ADDR_BITS);
			addr_aton(buf, &entry.route_gw);
			entry.metric = metric;
			
			if ((ret = callback(&entry, arg)) != 0)
				break;
		}
		fclose(fp);
	}
	return (ret);
}

```

这段代码定义了一个名为 route_t 的结构体，它用于表示网络路由信息。该结构体包含两个成员变量，分别是一个指向文件描述符（fd）的指针和一个指向长整型数据结构的指针。

接着，函数 route_close() 的实现，该函数接收一个指向 route_t 结构体的指针变量 r。函数首先检查 r 是否为空，如果是，则执行以下操作：关闭 r 指向的文件描述符，关闭 r 指向的长整型数据结构的文件描述符。接着，释放 r 指向的内存。最后，函数返回一个指向 NULL 的指针。

该函数的作用是关闭通过指针变量 r 指向的文件描述符，如果 r 为空，则不会执行任何操作，可以确保在调用者正确使用函数后，不会导致内存泄漏或其他问题。


```cpp
route_t *
route_close(route_t *r)
{
	if (r != NULL) {
		if (r->fd >= 0)
			close(r->fd);
		if (r->nlfd >= 0)
			close(r->nlfd);
		free(r);
	}
	return (NULL);
}

```