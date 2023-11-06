# Nmap源码解析 10

# `libdnet-stripped/include/dnet/icmpv6.h`

这段代码定义了一个名为"icmpv6.h"的 header文件，属于ICMPv6头部定义的范畴。该文件主要定义了ICMPv6的一些基本概念和头部长度。

具体来说，该文件定义了ICMPv6头部的结构体类型，以及IPv6地址和ICMPv6地址的表示方式。此外，还定义了ICMPv6头部长度为4字节。

由于该文件并未定义全局变量或函数，因此无法对其进行直接修改或使用。


```cpp
/*
 * icmpv6.h
 *
 * ICMPv6.
 * RFC 4443
 *
 * $Id: $
 */

#ifndef DNET_ICMPV6_H
#define DNET_ICMPV6_H

#define ICMPV6_HDR_LEN	4	/* base ICMPv6 header length */

#ifndef __GNUC__
```

这段代码定义了一个头文件，名为`__attribute__`，用于定义扩展功能属性。

具体来说，第一个 `#ifndef` 是声明，表示如果这个头文件已经被定义过，则不需要再次定义。如果这个头文件没有被定义过，则需要定义。

接下来是 `#define` 函数，它接受一个参数 `x`，使用 `__attribute__` 进行定义。如果定义成功，则返回 `x` 经过处理的定义形式，否则返回 `x` 的原始形式。

然后是 `# pragma pack(1)` 声明，告诉编译器使用该头文件定义的 `__attribute__` 函数的第一个参数必须是一个整数。

最后是定义一个名为`icmpv6_hdr`的结构体，包含三个成员：`icmpv6_type`、`icmpv6_code` 和 `icmpv6_cksum`，分别表示 IPv6 消息的类型、代码和校验和。


```cpp
#ifndef __attribute__
# define __attribute__(x)
#endif
# pragma pack(1)
#endif

/*
 * ICMPv6 header
 */
struct icmpv6_hdr {
	uint8_t		icmpv6_type;	/* type of message, see below */
	uint8_t		icmpv6_code;	/* type sub code */
	uint16_t	icmpv6_cksum;	/* ones complement cksum of struct */
};

```

这段代码定义了一个头文件，名为“icmpv6_parameters”，其中包含了一些用于表示IPv6数据包状态的枚举类型和编码类型。

以下是定义的各个枚举类型和编码类型：

* ICMPV6_CODE_NONE：表示没有数据包状态的编码类型。
* ICMPV6_UNREACH：表示无法到达目的地的编码类型。可以细分为四种情况：
	+ ICMPV6_UNREACH_NOROUTE：表示无法通过任何网络路由到达目的地。
	+ ICMPV6_UNREACH_PROHIB：表示管理员禁止到达目的地。
	+ ICMPV6_UNREACH_SCOPE：表示目标地址不在源地址范围内。
	+ ICMPV6_UNREACH_ADDR：表示目标地址不可达。
* ICMPV6_UNREACH_PORT：表示目标端口不可达。
* ICMPV6_UNREACH_FILTER_PROHIB：表示源点过滤和防火墙策略禁止到达目的地。
* ICMPV6_UNREACH_REJECT_ROUTE：表示无法到达目的地，但可以尝试重新路由。
* ICMPV6_TIMEXCEED：表示在传输过程中超时，其中包含两种情况：
	+ ICMPV6_TIMEXCEED_INTRANS：表示在传输过程中由于网络拥塞等原因导致超时。
	+ ICMPV6_TIMEXCEED：表示在传输过程中总时超时。

这些枚举类型和编码类型可以用于生成适当的错误代码，以便在IPv6数据包处理过程中出现问题时进行诊断和错误处理。


```cpp
/*
 * Types (icmpv6_type) and codes (icmpv6_code) -
 * http://www.iana.org/assignments/icmpv6-parameters
 */
#define		ICMPV6_CODE_NONE	0		/* for types without codes */
#define ICMPV6_UNREACH		1		/* dest unreachable, codes: */
#define		ICMPV6_UNREACH_NOROUTE		0	/* no route to dest */
#define		ICMPV6_UNREACH_PROHIB		1	/* admin prohibited */
#define		ICMPV6_UNREACH_SCOPE		2	/* beyond scope of source address */
#define		ICMPV6_UNREACH_ADDR		3	/* address unreach */
#define		ICMPV6_UNREACH_PORT		4	/* port unreach */
#define		ICMPV6_UNREACH_FILTER_PROHIB	5	/* src failed ingress/egress policy */
#define		ICMPV6_UNREACH_REJECT_ROUTE	6	/* reject route */
#define ICMPV6_TIMEXCEED	3		/* time exceeded, code: */
#define		ICMPV6_TIMEXCEED_INTRANS	0	/* hop limit exceeded in transit */
```

这段代码定义了一系列头文件和常量，用于标识和处理ICMPV6协议中的错误信息。

具体来说，这些头文件定义了ICMPV6_TIMEXCEED_REASS定义了 fragmetn reassembly time exceeded 的状态码，ICMPV6_PARAMPROBLEM定义了参数问题，包括错误的头信息和编码，ICMPV6_PARAMPROBLEM_FIELD定义了一个错误的头信息，ICMPV6_NEIGHBOR_SOLICITATION和ICMPV6_NEIGHBOR_ADVERTISEMENT定义了邻居发现类型，分别表示主动和被动的邻居发现，ICMPV6_INFOTYPE定义了输入字段类型，用于标识数据包的协议类型，最后，定义了几个常量，如ICMPV6_ECHO和ICMPV6_ECHOREPLY，用于标识回送请求和回送响应。

这些定义用于在ICMPV6协议中处理错误信息，例如ICMPV6_TIMEXCEED_REASS状态码，用于指示是否可以重新使用一个IPv6分片，如果分片重新组装时间超过了设定的阈值。ICMPV6_PARAMPROBLEM用于标识参数问题，例如无效的参数头信息和编码，ICMPV6_NEIGHBOR_SOLICITATION和ICMPV6_NEighbor_ADVERTISEMENT用于标识邻居发现类型，最后，ICMPV6_INFOTYPE用于标识输入字段类型。


```cpp
#define		ICMPV6_TIMEXCEED_REASS		1	/* fragmetn reassembly time exceeded */
#define ICMPV6_PARAMPROBLEM	4		/* parameter problem, code: */
#define 	ICMPV6_PARAMPROBLEM_FIELD	0	/* erroneous header field encountered */
#define 	ICMPV6_PARAMPROBLEM_NEXTHEADER	1	/* unrecognized Next Header type encountered */
#define 	ICMPV6_PARAMPROBLEM_OPTION	2	/* unrecognized IPv6 option encountered */
#define ICMPV6_ECHO		128		/* echo request */
#define ICMPV6_ECHOREPLY	129		/* echo reply */
/*
 * Neighbor discovery types (RFC 4861)
 */
#define	ICMPV6_NEIGHBOR_SOLICITATION	135
#define	ICMPV6_NEIGHBOR_ADVERTISEMENT	136

#define	ICMPV6_INFOTYPE(type) (((type) & 0x80) != 0)

```

这两段代码定义了两个结构体：icmpv6_msg_echo和icmpv6_msg_nd。

icmpv6_msg_echo结构体定义了一个用于发送ICMPv6 Echo请求（包括数据）的报文。该结构体包含以下字段：

- icmpv6_id：设置为0x12345678的16位标识符。
- icmpv6_seq：设置为当前心跳周期中返回数据包的序列号。
- icmpv6_data：可选的数据部分，以字节数组形式包含。

icmpv6_msg_nd结构体定义了一个用于发送ICMPv6邻居请求（包括广告）的报文。该结构体包含以下字段：

- icmpv6_flags：设置为0x01020304的32位标识符，用于标识邻居请求或广告。
- icmpv6_target：设置为IPv6地址000011112222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222222


```cpp
/*
 * Echo message data
 */
struct icmpv6_msg_echo {
	uint16_t	icmpv6_id;
	uint16_t	icmpv6_seq;
	uint8_t		icmpv6_data __flexarr;	/* optional data */
};

/* Neighbor solicitation or advertisement (single hardcoded option).
   RFC 4861, sections 4.3 and 4.4. */
struct icmpv6_msg_nd {
	uint32_t	icmpv6_flags;
	ip6_addr_t	icmpv6_target;
	uint8_t		icmpv6_option_type;
	uint8_t		icmpv6_option_length;
	eth_addr_t	icmpv6_mac;
};

```

这段代码定义了一个名为 `icmpv6_msg` 的枚举类型，包含两个成员变量 `icmpv6_msg_echo` 和 `icmpv6_msg_nd`，分别对应着 ICMPV6_ECHO 和 ICMPV6_NEIGHBOR 消息类型。

`union` 关键字表示该枚举类型中所有成员变量都共用同一个体。

`struct` 关键字定义了 `icmpv6_msg` 枚举类型的每个成员变量，包括 `icmpv6_msg_echo` 和 `icmpv6_msg_nd`。

`do-while` 语句用于循环执行代码块内的语句，这里是在定义 `icmpv6_msg_echo` 和 `icmpv6_msg_nd` 成员变量时设置它们对应的成员变量的值。

`struct icmpv6_hdr *icmpv6_pack_p = (struct icmpv6_hdr *)(hdr);` 表示将 `hdr` 所指向的结构体类型的指针赋值给 `icmpv6_pack_p`，使得 `icmpv6_pack_p` 指向了 `hdr` 的下一个克隆体。

`icmpv6_pack_hdr(hdr, type, code)` 在 `icmpv6_pack_p` 的定义中，将 `hdr` 所指的消息头部信息作为参数传递给 `icmpv6_pack_hdr` 函数，这个函数接收三个参数：`hdr` 的类型、代码和校验和。

函数定义的代码作为 `icmpv6_pack_p` 的成员变量，被存储在一个 `icmpv6_hdr *` 类型的变量中。

由于 `#ifndef` 和 `#pragma pack()` 都不属于平常编译器会解析的预处理指令，因此这段代码不会输出 `hdr` 的值，也不会创建克隆体。


```cpp
/*
 * ICMPv6 message union
 */
union icmpv6_msg {
	struct icmpv6_msg_echo	   echo;	/* ICMPV6_ECHO{REPLY} */
	struct icmpv6_msg_nd	   nd;		/* ICMPV6_NEIGHBOR_{SOLICITATION,ADVERTISEMENT} */
};

#ifndef __GNUC__
# pragma pack()
#endif

#define icmpv6_pack_hdr(hdr, type, code) do {				\
	struct icmpv6_hdr *icmpv6_pack_p = (struct icmpv6_hdr *)(hdr);	\
	icmpv6_pack_p->icmpv6_type = type; icmpv6_pack_p->icmpv6_code = code;	\
} while (0)

```

这两段代码定义了两个函数，一个是 `icmpv6_pack_hdr_echo`，另一个是 `icmpv6_pack_hdr_ns_mac`。它们的作用是参与 Internet 控制消息协议（ICMPv6）的封装。封装的主要目的是将数据包封装为 ICMPV6 消息，并在数据包中添加一些头部信息。

1. `icmpv6_pack_hdr_echo`

该函数的实现主要涉及以下几个步骤：

a. 从 `hdr` 字面量开始，找到 `ICMPV6_HDR_LEN` 字段的位置，然后将 `ICMPV6_HDR_LEN` 长度加上，得到一个 `ICMPv6_msg_echo` 结构体 pointer，即 `echo_pack_p`。

b. 将 `echo_pack_p` 所指向的结构体中的 `icmpv6_id` 和 `icmpv6_seq` 成员赋值，分别为 `id` 和 `seq`。

c. 通过 `memmove` 函数将 `data` 字段中的数据复制到 `echo_pack_p->icmpv6_data` 指向的内存区域。然后，这个区域的长度（由 `len` 参数指定）将被复制到 `echo_pack_p`。

d. 该函数没有返回值，因为它只是一个辅助函数，不返回数据。

1. `icmpv6_pack_hdr_ns_mac`

与 `icmpv6_pack_hdr_echo` 类似，该函数的实现主要涉及以下几个步骤：

a. 从 `hdr` 字面量开始，找到 `ICMPV6_HDR_NS_MAC_LEN` 字段的位置，然后将 `ICMPV6_HDR_NS_MAC_LEN` 长度加上，得到一个 `ICMPv6_msg_nd` 结构体 pointer，即 `nd_pack_p`。

b. 将 `nd_pack_p` 所指向的结构体中的 `icmpv6_flags` 成员赋值为 0。

c. 通过 `memmove` 函数将 `target` 字段中的 IP6 地址复制到 `nd_pack_p->icmpv6_target`。

d. 通过 `memmove` 函数将 `srcmac` 字段中的以太网 MAC 地址复制到 `nd_pack_p->icmpv6_mac`。

e. 该函数没有返回值，因为它只是一个辅助函数，不返回数据。


```cpp
#define icmpv6_pack_hdr_echo(hdr, type, code, id, seq, data, len) do {	\
	struct icmpv6_msg_echo *echo_pack_p = (struct icmpv6_msg_echo *)\
		((uint8_t *)(hdr) + ICMPV6_HDR_LEN);			\
	icmpv6_pack_hdr(hdr, type, code);				\
	echo_pack_p->icmpv6_id = htons(id);				\
	echo_pack_p->icmpv6_seq = htons(seq);				\
	memmove(echo_pack_p->icmpv6_data, data, len);			\
} while (0)

#define icmpv6_pack_hdr_ns_mac(hdr, targetip, srcmac) do {		\
	struct icmpv6_msg_nd *nd_pack_p = (struct icmpv6_msg_nd *)	\
		((uint8_t *)(hdr) + ICMPV6_HDR_LEN);			\
	icmpv6_pack_hdr(hdr, ICMPV6_NEIGHBOR_SOLICITATION, 0);		\
	nd_pack_p->icmpv6_flags = 0;					\
	memmove(&nd_pack_p->icmpv6_target, &(targetip), IP6_ADDR_LEN);	\
	nd_pack_p->icmpv6_option_type = 1;				\
	nd_pack_p->icmpv6_option_length = 1;				\
	memmove(&nd_pack_p->icmpv6_mac, &(srcmac), ETH_ADDR_LEN);	\
} while (0)

```

这段代码是一个头文件声明，表示这是一个自定义的 C++ 头文件，它可能是一个 C++ 程序或库的一部分。这个头文件包含一个预处理指令 #define，它告诉编译器在编译之前需要定义的一些符号，包括 "DNET" 和 "ICMPV6"，同时也告诉编译器不要忘记 include 当前头文件。

因此，这段代码的作用是定义了一些预处理指令，以便在编译之前定义这些符号，并为程序或库提供更易读性。


```cpp
#endif /* DNET_ICMPV6_H */

```

# `libdnet-stripped/include/dnet/intf.h`

这段代码定义了一个名为"intf.h"的文件头目，同时在文件中引入了名为"intf.c"的文件。因此，这段代码的作用是定义了一个名为"intf"的接口，并实现了对它的定义。

具体来说，这个接口提供了一些与网络接口相关的操作，包括创建、绑定、监听和取消绑定等。这些操作的具体实现可能由"intf.c"文件来完成。


```cpp
/*
 * intf.c
 *
 * Network interface operations.
 *
 * Copyright (c) 2000 Dug Song <dugsong@monkey.org>
 *
 * $Id: intf.h 478 2004-01-13 07:41:09Z dugsong $
 */

#ifndef DNET_INTF_H
#define DNET_INTF_H

/*
 * Interface entry
 */
```

这段代码定义了一个名为INTF_NAME_LEN的宏，其值为16。接下来定义了一个名为intf_entry的struct类型的数据结构，该结构体包含了以下字段：

- intf_len：记录接口名称的长度。
- intf_name：记录接口名称的内存缓冲区。
- intf_index：记录接口的索引，可以是本地或远程。
- intf_type：记录接口类型，可以是读写或读写/写。
- intf_flags：记录接口的标志，可以是ASYNC或GENERIC_GROUP。
- intf_mtu：记录接口的最大传输单元（MTU）。
- intf_addr：记录接口的地址。
- intf_dst_addr：记录接口的目标地址。
- intf_link_addr：记录链路层地址。
- intf_alias_num：记录接口的别名数量。
- intf_alias_addrs：记录接口别名地址的内存缓冲区。

这个结构体可以用来表示一个网络接口，通过它可以对网络接口进行定义，描述它的属性，如MTU、地址等信息。


```cpp
#define INTF_NAME_LEN	16

struct intf_entry {
	u_int		intf_len;		    /* length of entry */
	char		intf_name[INTF_NAME_LEN];   /* interface name */
	u_int		intf_index;		    /* interface index (r/o) */
	u_short		intf_type;		    /* interface type (r/o) */
	u_short		intf_flags;		    /* interface flags */
	u_int		intf_mtu;		    /* interface MTU */
	struct addr	intf_addr;		    /* interface address */
	struct addr	intf_dst_addr;		    /* point-to-point dst */
	struct addr	intf_link_addr;		    /* link-layer address */
	u_int		intf_alias_num;		    /* number of aliases */
	struct addr	intf_alias_addrs __flexarr; /* array of aliases */
};

```

这段代码定义了一系列接口类型，用于描述不同网络接口的特性。它们定义了一种接口类型枚举，每个枚举都有一个唯一的数字表示。这些数字通常与IEEE 802.11标准中的协议相关。

以下是这些接口类型的简要描述：

- INTF_TYPE_OTHER：其他类型的接口，可能包括尚未定义的接口。
- INTF_TYPE_ETH：以太网接口。
- INTF_TYPE_TOKENRING：令牌环接口。
- INTF_TYPE_FDDI：光纤数据接口。
- INTF_TYPE_PPP：点对点协议（如PPP）接口。
- INTF_TYPE_LOOPBACK：硬件环回接口。
- INTF_TYPE_SLIP：串行线接口（如串口）。
- INTF_TYPE_TUN：虚拟/内部连接（如SSTP）接口。

这些枚举定义了接口类型的枚举，使得代码在编译时可以更轻松地理解和维护。接口类型枚举便于将特定功能与实现细节分开，从而提高代码的可读性。


```cpp
/*
 * MIB-II interface types - http://www.iana.org/assignments/ianaiftype-mib
 */
#define INTF_TYPE_OTHER		1	/* other */
#define INTF_TYPE_ETH		6	/* Ethernet */
#define INTF_TYPE_TOKENRING	9	/* Token Ring */
#define INTF_TYPE_FDDI		15	/* FDDI */
#define INTF_TYPE_PPP		23	/* Point-to-Point Protocol */
#define INTF_TYPE_LOOPBACK	24	/* software loopback */
#define INTF_TYPE_SLIP		28	/* Serial Line Interface Protocol */
#define INTF_TYPE_TUN		53	/* proprietary virtual/internal */

/*
 * Interface flags
 */
```

这段代码定义了一系列标记，用于标识不同种类的网络接口。

#define INTF_FLAG_UP		0x01	/* enable interface */
#define INTF_FLAG_LOOPBACK	0x02	/* is a loopback net (r/o) */
#define INTF_FLAG_POINTOPOINT	0x04	/* point-to-point link (r/o) */
#define INTF_FLAG_NOARP		0x08	/* disable ARP */
#define INTF_FLAG_BROADCAST	0x10	/* supports broadcast (r/o) */
#define INTF_FLAG_MULTICAST	0x20	/* supports multicast (r/o) */

这些标记用于描述接口的类型，例如 `INTF_FLAG_UP` 表示支持网络接口，`INTF_FLAG_LOOPBACK` 表示该接口是一个循环接口。

另外，还定义了一个名为 `intf_t` 的结构体，用于表示网络接口的信息。

typedef struct intf_handle intf_t;

这个 `intf_t` 类型的变量可以用来操作 `intf_handle` 类型的变量，例如 `intf_open()` 函数接受一个 `intf_t` 类型的参数，用于开启接口。

```cpp
intf_t	*intf_open(void);
```

`intf_get()` 函数用于获取接口的当前状态，并返回一个 `intf_entry` 类型的指针。

```cpp
int	 intf_get(intf_t *i, struct intf_entry *entry);
```

`intf_get_index()` 函数用于获取一个接口中当前标记的索引，标记可以是上面定义的一系列标记。

```cpp
int	 intf_get_index(intf_t *intf, struct intf_entry *entry, int af, unsigned int index);
```

注意，`intf_open()`、`intf_get()` 和 `intf_get_index()` 函数的实现并未在代码中给出，这些函数的具体实现可能因操作系统和硬件的不同而有所不同。


```cpp
#define INTF_FLAG_UP		0x01	/* enable interface */
#define INTF_FLAG_LOOPBACK	0x02	/* is a loopback net (r/o) */
#define INTF_FLAG_POINTOPOINT	0x04	/* point-to-point link (r/o) */
#define INTF_FLAG_NOARP		0x08	/* disable ARP */
#define INTF_FLAG_BROADCAST	0x10	/* supports broadcast (r/o) */
#define INTF_FLAG_MULTICAST	0x20	/* supports multicast (r/o) */

typedef struct intf_handle intf_t;

typedef int (*intf_handler)(const struct intf_entry *entry, void *arg);

__BEGIN_DECLS
intf_t	*intf_open(void);
int	 intf_get(intf_t *i, struct intf_entry *entry);
int	 intf_get_index(intf_t *intf, struct intf_entry *entry, int af, unsigned int index);
```

以下是代码的作用：

这是一组用于从网络数据包中获取不同网络接口设备（如网卡）的设备的名称的功能库。实现了三个函数：get_src，get_dst，get_pcap_devname。

get_src函数接收一个intf_t类型的指针，一个struct intf_entry类型的结构体和一个struct addr类型的指针作为参数。函数返回一个int类型的变量，表示设备名称。

get_dst函数与get_src函数类似，但输入参数为intf_t类型的指针，结构体为struct intf_entry，输出参数为struct addr类型的指针。函数返回一个int类型的变量，表示设备名称。

get_pcap_devname函数接收一个const char *类型的整网数据包设备名称和一个char类型的指针作为输入参数。函数返回一个包含设备名称和设备描述符的char类型指针。函数使用了一个内部函数intf_get_pcap_devname_cached，如果缓存中存在设备名称，则直接返回缓存中的设备名称，否则实现一个缓存函数。

intf_set函数接收一个intf_t类型的指针和一个intf_handler类型的函数作为输入参数。函数内部使用了一个内部函数intf_set，接收一个const struct intf_entry *类型的指针和一个int类型的整数作为输入参数。

intf_loop函数接收一个intf_t类型的指针，一个intf_handler类型的函数和一个void类型的指针作为输入参数。函数内部使用了一个内部函数intf_loop，接收一个intf_t类型的指针和一个int类型的整数作为输入参数。


```cpp
int	 intf_get_src(intf_t *i, struct intf_entry *entry, struct addr *src);
int	 intf_get_dst(intf_t *i, struct intf_entry *entry, struct addr *dst);
int	 intf_get_pcap_devname(const char *intf_name, char *pcapdev, int pcapdevlen);
int	 intf_get_pcap_devname_cached(const char *intf_name, char *pcapdev, int pcapdevlen, int refresh);
int	 intf_set(intf_t *i, const struct intf_entry *entry);
int	 intf_loop(intf_t *i, intf_handler callback, void *arg);
intf_t	*intf_close(intf_t *i);
__END_DECLS

#endif /* DNET_INTF_H */

```

# `libdnet-stripped/include/dnet/ip.h`

这段代码定义了一个名为"ip.h"的头文件，它包含了关于IPv4协议的一些定义和声明。以下是它的主要部分：

```cpp
#ifndef DNET_IP_H
#define DNET_IP_H
```

这是一个预处理指令，它告诉编译器在编译之前需要包含的标识符。它表示为"DNET_IP_H"。

```cpp
#define IP_ADDR_LEN	4		/* IP address length */
#define IP_ADDR_BITS	32		/* IP address bits */
```

这两个定义定义了IPv4地址的长度和位数。IPv4地址是一个32位的二进制字符串，它由四个8位二进制数组成，每个8位二进制数表示一个比特位。IPv4地址的长度必须是4字节，这意味着它最多可以表示2^4-1（即15个）不同的地址。

```cpp
#define MAX_IP_C是人迹可至的最大IP地址
```

这个定义是一个IP地址常量，它定义了一个常量"MAX_IP_C"，表示一个人迹可至的最大IP地址。这个常量通常在网络设计中使用，以确保网络能够支持足够大的地址空间。

```cpp
#define IP_FILTER_SIZE	15		/* Filter size */
```

这个定义是一个IP地址常量，它表示了一个常量"IP_FILTER_SIZE"，表示一个用于IPv4地址过滤的掩码长度。IPv4地址过滤可以帮助防止网络攻击，因为它可以限制一个网络中的IP地址的数量。

```cpp
IP_FILTER_SIZE参数表示每个IPv4地址可以拥有的滤镜数量。因此，它表示一个15位的二进制数，它最多可以表示2^15-1（即32167）不同的滤镜。
```

```cpp
static int ntohs(int n);
```

这个函数是一个将IPv4地址转换为8位字节序列的函数。它的参数是一个int类型的整数，表示IPv4地址。函数返回一个int类型的整数，表示将IPv4地址转换为字节序列后的值。

```cpp
static int ntohs(int n)
{
   return (n >> 8) + (n & 0xff);
}
```

这个函数将一个32位的IPv4地址转换为字节序列，并返回它。它通过先将IPv4地址的32位二进制数转换为最高位，然后将其转换为字节序列。这是因为IPv4地址是4字节长度的，因此它的最高位必须是0或255之一。


```cpp
/*
 * ip.h
 *
 * Internet Protocol (RFC 791).
 *
 * Copyright (c) 2000 Dug Song <dugsong@monkey.org>
 *
 * $Id: ip.h 594 2005-02-16 22:02:45Z dugsong $
 */

#ifndef DNET_IP_H
#define DNET_IP_H

#define IP_ADDR_LEN	4		/* IP address length */
#define IP_ADDR_BITS	32		/* IP address bits */

```

这段代码定义了一些常量，用于定义IP头和IP选项的数据结构。主要作用是定义IP头部和IP选项的长度以及最大长度。

具体来说，以下代码定义了：

1. IP头部（HDR）的最大长度为20字节（IP_HDR_LEN），其中包含了20个字节的基本IP头部信息。
2. IP选项的最大长度为40字节（IP_OPT_LEN_MAX），其中包含了40个字节的基本IP选项信息。
3. IP头部最大长度为（IP_HDR_LEN + IP_OPT_LEN_MAX），即20字节（IP_HDR_LEN）和40字节（IP_OPT_LEN_MAX）的总和，最大长度为65535字节（IP_LEN_MAX）。
4. IP头部最小长度为IP头部最大长度减去IP选项最大长度，即20字节（IP_HDR_LEN）减去40字节（IP_OPT_LEN_MAX），得到的最小长度为IP_HDR_LEN的最小值，即IP_HDR_LEN的最小值为65535字节（IP_LEN_MIN）。
5. 定义了一个名为ip_addr_t的类型，用于表示IP地址，其最大长度为65535字节（IP_LEN_MAX），最小长度为65535字节减去20字节（IP_HDR_LEN），即65515字节（IP_HDR_MIN）。


```cpp
#define IP_HDR_LEN	20		/* base IP header length */
#define IP_OPT_LEN	2		/* base IP option length */
#define IP_OPT_LEN_MAX	40
#define IP_HDR_LEN_MAX	(IP_HDR_LEN + IP_OPT_LEN_MAX)

#define IP_LEN_MAX	65535
#define IP_LEN_MIN	IP_HDR_LEN

typedef uint32_t	ip_addr_t;

#ifndef __GNUC__
#ifndef __attribute__
# define __attribute__(x)
#endif
# pragma pack(1)
```

这段代码是一个IP头结构体，定义了IP头的一些基本字段。它包括了IP版本的字段，IP头长度，ID和 fragmentation offset，TTL和校验和字段，以及源地址和目标地址。

IP头的作用是定义IP数据包的结构，作为数据传输的基础。它定义了数据包如何在网络中传输，以及如何确保数据的可靠性和正确性。IP头通常包括源地址和目标地址，以便路由器能够将数据包从源地址传输到目标地址。此外，IP头还定义了数据包的类型（如TCP或UDP），以便应用程序能够正确地发送或接收数据包。


```cpp
#endif

/*
 * IP header, without options
 */
struct ip_hdr {
#if DNET_BYTESEX == DNET_BIG_ENDIAN
	uint8_t		ip_v:4,		/* version */
			ip_hl:4;	/* header length (incl any options) */
#elif DNET_BYTESEX == DNET_LIL_ENDIAN
	uint8_t		ip_hl:4,
			ip_v:4;
#else
# error "need to include <dnet.h>"	
#endif
	uint8_t		ip_tos;		/* type of service */
	uint16_t	ip_len;		/* total length (incl header) */
	uint16_t	ip_id;		/* identification */
	uint16_t	ip_off;		/* fragment offset and flags */
	uint8_t		ip_ttl;		/* time to live */
	uint8_t		ip_p;		/* protocol */
	uint16_t	ip_sum;		/* checksum */
	ip_addr_t	ip_src;		/* source address */
	ip_addr_t	ip_dst;		/* destination address */
};

```

这段代码定义了一系列IP选项中的服务类型(IP_TOS_DEFAULT到IP_TOS_ETHER)，以及这些服务类型的默认值。它们是根据RFC 1349定义的，描述了IP在不同类型的数据传输中的优先级。这些选项包括：

- IP_TOS_DEFAULT：默认值，表示高优先级的服务，适用于对数据传输的最低延迟要求较高的情况。
- IP_TOS_LOWDELAY：低延迟(LOWDELAY)服务，适用于对数据传输的最低延迟要求较低的情况。
- IP_TOS_THROUGHPUT：高吞吐量(THROWTHROW)服务，适用于对数据传输的带宽要求较高的情况。
- IP_TOS_RELIABILITY：高可靠性(RELIABILITY)服务，适用于需要确保数据传输可靠性较高的应用，例如金融交易等。
- IP_TOS_LOWCOST：低费用(LOWCOST)服务，适用于需要尽可能以最低成本提供服务的情况，例如互联网电子邮件传输等。
- IP_TOS_ECT:ECN-capable transport(ECT)服务，表示支持增强型传输模式(ECN)，适用于需要支持网络增强型传输的用途。
- IP_TOS_CE: congestion experienced(CE)服务，表示曾经经历过的拥塞，适用于需要检测和缓解网络拥塞的应用，例如实时视频等。

IP选项的具体含义如下：

- IP_TOS_DEFAULT:000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000


```cpp
/*
 * Type of service (ip_tos), RFC 1349 ("obsoleted by RFC 2474")
 */
#define IP_TOS_DEFAULT		0x00	/* default */
#define IP_TOS_LOWDELAY		0x10	/* low delay */
#define IP_TOS_THROUGHPUT	0x08	/* high throughput */
#define IP_TOS_RELIABILITY	0x04	/* high reliability */
#define IP_TOS_LOWCOST		0x02	/* low monetary cost - XXX */
#define IP_TOS_ECT		0x02	/* ECN-capable transport */
#define IP_TOS_CE		0x01	/* congestion experienced */

/*
 * IP precedence (high 3 bits of ip_tos), hopefully unused
 */
#define IP_TOS_PREC_ROUTINE		0x00
```

这段代码定义了一系列IP TOS（Transmission Control Indication）Preemption flags，用于指导操作系统如何处理IP数据包。这些 flags 提供了对 IP数据包的 Fragmentation 处理。下面是每个定义的预先设置：

1. IP_TOS_PREC_PRIORITY：这是最低优先级，用于确保所有数据包优先级最高。
2. IP_TOS_PREC_IMMEDIATE：可以重置IP TOS 字段，但不会更改优先级。
3. IP_TOS_PREC_FLASH：设置或清除 IP TOS 字段，但不会更改优先级。
4. IP_TOS_PREC_FLASHOVERFLOW：设置或清除 IP TOS 字段，但不会更改优先级。
5. IP_TOS_PREC_CRITIC_ECP：设置或清除 IP TOS 字段，但不会更改优先级。
6. IP_TOS_PREC_INTERNETCONTROL：设置或清除 IP TOS 字段，但不会更改优先级。
7. IP_TOS_PREC_NETCONTROL：设置或清除 IP TOS 字段，但不会更改优先级。
8. IP_RF：设置或清除 Fragmentation flags。
9. IP_DF：设置或清除 Don't fragment flags。
10. IP_MF：设置或清除 More fragments flags（ last frag）。
11. IP_OFFMASK：设置或清除 Fragmentation offset mask。


```cpp
#define IP_TOS_PREC_PRIORITY		0x20
#define IP_TOS_PREC_IMMEDIATE		0x40
#define IP_TOS_PREC_FLASH		0x60
#define IP_TOS_PREC_FLASHOVERRIDE	0x80
#define IP_TOS_PREC_CRITIC_ECP		0xa0
#define IP_TOS_PREC_INTERNETCONTROL	0xc0
#define IP_TOS_PREC_NETCONTROL		0xe0

/*
 * Fragmentation flags (ip_off)
 */
#define IP_RF		0x8000		/* reserved */
#define IP_DF		0x4000		/* don't fragment */
#define IP_MF		0x2000		/* more fragments (not last frag) */
#define IP_OFFMASK	0x1fff		/* mask for fragment offset */

```

这段代码定义了一些用于 Internet 协议（IP）的标准常量，以及定义了如何使用这些常量。

首先定义了两个整数常量：IP_TTL_DEFAULT 和 IP_TTL_MAX。IP_TTL_DEFAULT 表示IP生存时间（TTL）的默认值，在 IPv6 中，通常这是一个固定的值，为 64 秒。IP_TTL_MAX 表示IP生存时间（TTL）的最大值。

接下来定义了四个头目：IP_PROTO_IP、IP_PROTO_HOPOPTS、IP_PROTO_ICMP 和 IP_PROTO_IGMP。这些头目用于标识协议类型。IP_PROTO_IP 是 IPv6 的头目，用于标识使用 IPv6 协议通过 IP 协议（IPv6）传输数据。IP_PROTO_HOPOPTS 是 IPv6 头目，它定义了在 IPv6 头部的开销。IP_PROTO_ICMP 是 IPv6 的头目，用于处理互联网控制消息协议（ICMP）。IP_PROTO_IGMP 是 IPv6 的头目，用于处理 Internet 组管理协议（IGMP）。

最后，没有定义任何函数或变量，所以这些头目没有相应的实现。


```cpp
/*
 * Time-to-live (ip_ttl), seconds
 */
#define IP_TTL_DEFAULT	64		/* default ttl, RFC 1122, RFC 1340 */
#define IP_TTL_MAX	255		/* maximum ttl */

/*
 * Protocol (ip_p) - http://www.iana.org/assignments/protocol-numbers
 */
#define	IP_PROTO_IP		0		/* dummy for IP */
#define IP_PROTO_HOPOPTS	IP_PROTO_IP	/* IPv6 hop-by-hop options */
#define	IP_PROTO_ICMP		1		/* ICMP */
#define	IP_PROTO_IGMP		2		/* IGMP */
#define IP_PROTO_GGP		3		/* gateway-gateway protocol */
#define	IP_PROTO_IPIP		4		/* IP in IP */
```

这段代码定义了一系列头文件中的协议类型，包括TCP、CBT、EGP、IGP、BBNRCC、NVP、PUP、ARGUS、EMCON、XNET和CHAOS。这些头文件用于定义网络协议的协议类型，以便编译器和读者能够更好地理解代码中涉及的协议。


```cpp
#define IP_PROTO_ST		5		/* ST datagram mode */
#define	IP_PROTO_TCP		6		/* TCP */
#define IP_PROTO_CBT		7		/* CBT */
#define	IP_PROTO_EGP		8		/* exterior gateway protocol */
#define IP_PROTO_IGP		9		/* interior gateway protocol */
#define IP_PROTO_BBNRCC		10		/* BBN RCC monitoring */
#define IP_PROTO_NVP		11		/* Network Voice Protocol */
#define	IP_PROTO_PUP		12		/* PARC universal packet */
#define IP_PROTO_ARGUS		13		/* ARGUS */
#define IP_PROTO_EMCON		14		/* EMCON */
#define IP_PROTO_XNET		15		/* Cross Net Debugger */
#define IP_PROTO_CHAOS		16		/* Chaos */
#define	IP_PROTO_UDP		17		/* UDP */
#define IP_PROTO_MUX		18		/* multiplexing */
#define IP_PROTO_DCNMEAS	19		/* DCN measurement */
```



该代码定义了一系列的际通协议类型ID，包括主机监控协议(IP_PROTO_HMP)、数据包收发测量协议(IP_PROTO_PRM)、Xerox NS IDP(IP_PROTO_IDP)等。这些定义在网络协议的规范中定义，用于定义网络协议的协议头和协议字段，以便在不同系统之间进行通信。


```cpp
#define IP_PROTO_HMP		20		/* Host Monitoring Protocol */
#define IP_PROTO_PRM		21		/* Packet Radio Measurement */
#define	IP_PROTO_IDP		22		/* Xerox NS IDP */
#define IP_PROTO_TRUNK1		23		/* Trunk-1 */
#define IP_PROTO_TRUNK2		24		/* Trunk-2 */
#define IP_PROTO_LEAF1		25		/* Leaf-1 */
#define IP_PROTO_LEAF2		26		/* Leaf-2 */
#define IP_PROTO_RDP		27		/* "Reliable Datagram" proto */
#define IP_PROTO_IRTP		28		/* Inet Reliable Transaction */
#define	IP_PROTO_TP		29 		/* ISO TP class 4 */
#define IP_PROTO_NETBLT		30		/* Bulk Data Transfer */
#define IP_PROTO_MFPNSP		31		/* MFE Network Services */
#define IP_PROTO_MERITINP	32		/* Merit Internodal Protocol */
#define IP_PROTO_SEP		33		/* Sequential Exchange proto */
#define IP_PROTO_3PC		34		/* Third Party Connect proto */
```

这段代码定义了一系列IP协议的唯一标识符（ID），用于标识网络协议。这些协议包括：IP协议ID为35的Interdomain Policy Route、IP协议ID为36的Xpress Transfer Protocol、IP协议ID为37的Datagram Delivery Protocol和控制消息传输协议、IP协议ID为38的TP++ Transport Protocol、IP协议ID为40的IL Transport Protocol、IP协议ID为41的IPv6、IP协议ID为42的Source Demand Routing、IP协议ID为43的IPv6路由头、IP协议ID为44的Source Demand Routing、IP协议ID为46的Reservation Protocol、IP协议ID为47的Mobile Host Routing和IP协议ID为48的ENA。


```cpp
#define IP_PROTO_IDPR		35		/* Interdomain Policy Route */
#define IP_PROTO_XTP		36		/* Xpress Transfer Protocol */
#define IP_PROTO_DDP		37		/* Datagram Delivery Proto */
#define IP_PROTO_CMTP		38		/* IDPR Ctrl Message Trans */
#define IP_PROTO_TPPP		39		/* TP++ Transport Protocol */
#define IP_PROTO_IL		40		/* IL Transport Protocol */
#define IP_PROTO_IPV6		41		/* IPv6 */
#define IP_PROTO_SDRP		42		/* Source Demand Routing */
#define IP_PROTO_ROUTING	43		/* IPv6 routing header */
#define IP_PROTO_FRAGMENT	44		/* IPv6 fragmentation header */
#define IP_PROTO_RSVP		46		/* Reservation protocol */
#define	IP_PROTO_GRE		47		/* General Routing Encap */
#define IP_PROTO_MHRP		48		/* Mobile Host Routing */
#define IP_PROTO_ENA		49		/* ENA */
#define	IP_PROTO_ESP		50		/* Encap Security Payload */
```

这段代码定义了一系列头标，用于标识IP协议的不同部分。这些头标的作用是在编译时检查源代码是否正确，以及编译器能否正确地生成特定的IP头。

具体来说，这些头标定义了以下内容：

- IP协议的AH头标：51
- IP协议的INLSP头标：52
- IP协议的SWIPE头标：53
- IP协议的NARP头标：54
- IP协议的MOBILE头标：55（此头标已被删除，因为IPv6并不支持移动IP）
- IP协议的TLSP头标：56
- IP协议的SKIP头标：57
- IP协议的ICMPV6头标：58
- IP协议的NONE头标：59
- IP协议的DSTOPTS头标：60
- IP协议的ANYHOST头标：61
- IP协议的ANYNET头标：62
- IP协议的EXPAK头标：64
- IP协议的KRYPTOLAN头标：65

每种IP协议可能会使用不同的头标，以便编译器能够正确地识别它们。


```cpp
#define	IP_PROTO_AH		51		/* Authentication Header */
#define IP_PROTO_INLSP		52		/* Integated Net Layer Sec */
#define IP_PROTO_SWIPE		53		/* SWIPE */
#define IP_PROTO_NARP		54		/* NBMA Address Resolution */
#define	IP_PROTO_MOBILE		55		/* Mobile IP, RFC 2004 */
#define IP_PROTO_TLSP		56		/* Transport Layer Security */
#define IP_PROTO_SKIP		57		/* SKIP */
#define IP_PROTO_ICMPV6		58		/* ICMP for IPv6 */
#define IP_PROTO_NONE		59		/* IPv6 no next header */
#define IP_PROTO_DSTOPTS	60		/* IPv6 destination options */
#define IP_PROTO_ANYHOST	61		/* any host internal proto */
#define IP_PROTO_CFTP		62		/* CFTP */
#define IP_PROTO_ANYNET		63		/* any local network */
#define IP_PROTO_EXPAK		64		/* SATNET and Backroom EXPAK */
#define IP_PROTO_KRYPTOLAN	65		/* Kryptolan */
```

这段代码定义了一系列IP协议，包括RVD(Remote Virtual Device)、IPPC(Inet Pluribus Packet Core)、DISTFS(any distributed fs)、SATMON(SATNET Monitoring)、VISA(Visual Information System Architecture)、IPCV(Inet Packet Core Utility)、CPNX(Comp Proto Net Executive)、CPHB(Comp Protocol Heart Beat)、WSN(Wang Span Network)、PVP(Packet Video Protocol)、BRSATMON(Backroom SATNET Monitor)、SUNND(SUN ND Protocol)和WBMON(WIDEBAND Monitoring)。这些协议用于在计算机网络中传输数据。


```cpp
#define IP_PROTO_RVD		66		/* MIT Remote Virtual Disk */
#define IP_PROTO_IPPC		67		/* Inet Pluribus Packet Core */
#define IP_PROTO_DISTFS		68		/* any distributed fs */
#define IP_PROTO_SATMON		69		/* SATNET Monitoring */
#define IP_PROTO_VISA		70		/* VISA Protocol */
#define IP_PROTO_IPCV		71		/* Inet Packet Core Utility */
#define IP_PROTO_CPNX		72		/* Comp Proto Net Executive */
#define IP_PROTO_CPHB		73		/* Comp Protocol Heart Beat */
#define IP_PROTO_WSN		74		/* Wang Span Network */
#define IP_PROTO_PVP		75		/* Packet Video Protocol */
#define IP_PROTO_BRSATMON	76		/* Backroom SATNET Monitor */
#define IP_PROTO_SUNND		77		/* SUN ND Protocol */
#define IP_PROTO_WBMON		78		/* WIDEBAND Monitoring */
#define IP_PROTO_WBEXPAK	79		/* WIDEBAND EXPAK */
#define	IP_PROTO_EON		80		/* ISO CNLP */
```

这段代码定义了一系列IP协议的数据报类型，包括：Versatile Msg Transport（81）、Secure VMTP（82）、VINES（83）、TTP（84）、NSFNET-IGP（85）、Dissimilar Gateway Proto（86）、TCF（87）、EIGRP（88）、OSPF（89）、Sprite RPC Protocol（90）、LARP（91）、Multicast Transport Proto（92）、AX.25 Frames（93）、IP Encap（94）和Mobile Internet Ctrl（95）。

这些数据报类型是在网络协议中传输数据报时使用的。每个IP协议可能会使用不同的数据报类型来实现特定的功能。

例如，TCP数据包使用TCP协议通过网络传输数据，而UDP数据包使用UDP协议通过网络传输数据。在这个例子中，IP协议使用数据报类型来定义网络通信的参数，如协议类型、协议长度、服务类型等。


```cpp
#define IP_PROTO_VMTP		81		/* Versatile Msg Transport*/
#define IP_PROTO_SVMTP		82		/* Secure VMTP */
#define IP_PROTO_VINES		83		/* VINES */
#define IP_PROTO_TTP		84		/* TTP */
#define IP_PROTO_NSFIGP		85		/* NSFNET-IGP */
#define IP_PROTO_DGP		86		/* Dissimilar Gateway Proto */
#define IP_PROTO_TCF		87		/* TCF */
#define IP_PROTO_EIGRP		88		/* EIGRP */
#define IP_PROTO_OSPF		89		/* Open Shortest Path First */
#define IP_PROTO_SPRITERPC	90		/* Sprite RPC Protocol */
#define IP_PROTO_LARP		91		/* Locus Address Resolution */
#define IP_PROTO_MTP		92		/* Multicast Transport Proto */
#define IP_PROTO_AX25		93		/* AX.25 Frames */
#define IP_PROTO_IPIPENCAP	94		/* yet-another IP encap */
#define IP_PROTO_MICP		95		/* Mobile Internet Ctrl */
```

这段代码定义了一系列IP协议数据单元（数据报）所使用的协议类型（协议ID），包括 semaphoreCommSecSecProto（96）、Ethernet in IPv4（97）、encapsulation header（98）、private encryption scheme（99）、GMTP（100）、Ipsilon Flow Mgmt Proto（101）、NeighborIndicationMessage（102）、Protocol Indep Multicast（103）、ARIS（104）、SCPS（105）、QNX（106）、Active Networks（107）和IP Payload Compression（108）。这些数据报协议用于在网络中传输数据包。


```cpp
#define IP_PROTO_SCCSP		96		/* Semaphore Comm Sec Proto */
#define IP_PROTO_ETHERIP	97		/* Ethernet in IPv4 */
#define	IP_PROTO_ENCAP		98		/* encapsulation header */
#define IP_PROTO_ANYENC		99		/* private encryption scheme */
#define IP_PROTO_GMTP		100		/* GMTP */
#define IP_PROTO_IFMP		101		/* Ipsilon Flow Mgmt Proto */
#define IP_PROTO_PNNI		102		/* PNNI over IP */
#define IP_PROTO_PIM		103		/* Protocol Indep Multicast */
#define IP_PROTO_ARIS		104		/* ARIS */
#define IP_PROTO_SCPS		105		/* SCPS */
#define IP_PROTO_QNX		106		/* QNX */
#define IP_PROTO_AN		107		/* Active Networks */
#define IP_PROTO_IPCOMP		108		/* IP Payload Compression */
#define IP_PROTO_SNP		109		/* Sitara Networks Protocol */
#define IP_PROTO_COMPAQPEER	110		/* Compaq Peer Protocol */
```

这段代码定义了一系列IP协议类型，用于在网络中传输数据包。下面是每个协议类型的简要解释：

1. IPXIP：IPX（Intermediate Protocol X Effect）协议是IP数据报传输的一种机制，用于在网络上使用IPX/SPX协议栈。IPXIP协议使用IPX（Intermediate Protocol X Effect）协议栈，并通过IP数据报传输IPX/SPX消息。

2. VRRP：虚拟路由器冗余协议（Virtual Router Redundancy Protocol）用于在网络中实现路由器冗余。通过将多个VRRP路由器连接在一起，可以确保网络中的路由器可以自动检测和接管负载，以避免网络中断。

3. PGM：PGM（Programmable Generated Method）协议是一种可靠的数据传输协议，用于在网络上传输数据。PGM协议使用0-hop协议（0-port protocol）实现数据传输，这是一种无协议的数据传输机制，用于在网络上传输数据。

4. ANY0HOP：0-hop协议（Any-Hop Protocol）是一种数据传输机制，用于在网络上传输数据。它允许数据包通过任何网络层进行传输，而不受网络层的限制。

5. L2TP：二层隧道协议（Layer 2 Tunneling Protocol）是一种在数据链路层（第二层）上实现隧道协议的协议。它允许在网络上通过建立第二层隧道来传输数据，这对于在网络上传输层协议（如IP协议）非常有用。

6. DDX：数据独立设备（Data-Independent Device）协议是一种用于在网络上传输数据独立设备（如IP电话或网络摄像机）的协议。DDX协议使用IP数据报传输数据独立设备（DI）地址和数据。

7. IATP：Interactive Agent Transfer Protocol：用于在网络上传输数据代理（如IP电话或网络摄像机）的协议。IATP协议使用IP数据报传输代理地址和数据。

8. STP：自组网协议（Spanning Tree Protocol）是一种在网络上防止环路的协议。它通过选举一个根桥（Root Bridge）并将其连接到网络中的所有其他桥上，来实现网络环路。STP协议用于在网络上传输数据包。

9. SRP：源点到客户端路径（Source-to-Client Path）协议是一种在网络上传输数据包的协议。它允许源点（如IP电话或网络摄像机）在网络上选择一个客户端（目标点），并传输数据到该客户端。

10. UTI：用户数据报协议（User Data Queue Protocol）：用于在网络上传输用户数据报的协议。它允许用户生成并发送用户数据报，用于在网络上传输数据。

11. SMP：简单消息协议（Simple Message Protocol）是一种在网络上传输数据包的协议。它支持简单的消息传输，通常用于在网络上传输消息。

12. SM：简单消息协议（Simple Message Protocol）是一种在网络上传输数据包的协议。它支持简单的消息传输，通常用于在网络上传输消息。

13. PTP：性能传输协议（Performance Tunneling Protocol）是一种在网络上传输数据包的协议。它用于在网络上传输数据，并提供了一些性能优化，如错误恢复和速度自适应。

14. ISIS：Internet Society Message Authentication Code（Internet Society Message Authentication Code）协议是一种在网络上传输数据包的协议。它使用数字证书（D


```cpp
#define IP_PROTO_IPXIP		111		/* IPX in IP */
#define IP_PROTO_VRRP		112		/* Virtual Router Redundancy */
#define IP_PROTO_PGM		113		/* PGM Reliable Transport */
#define IP_PROTO_ANY0HOP	114		/* 0-hop protocol */
#define IP_PROTO_L2TP		115		/* Layer 2 Tunneling Proto */
#define IP_PROTO_DDX		116		/* D-II Data Exchange (DDX) */
#define IP_PROTO_IATP		117		/* Interactive Agent Xfer */
#define IP_PROTO_STP		118		/* Schedule Transfer Proto */
#define IP_PROTO_SRP		119		/* SpectraLink Radio Proto */
#define IP_PROTO_UTI		120		/* UTI */
#define IP_PROTO_SMP		121		/* Simple Message Protocol */
#define IP_PROTO_SM		122		/* SM */
#define IP_PROTO_PTP		123		/* Performance Transparency */
#define IP_PROTO_ISIS		124		/* ISIS over IPv4 */
#define IP_PROTO_FIRE		125		/* FIRE */
```

这段代码定义了一系列IP协议类型的常量，用于标识在网络协议中传输的数据类型。每个常量都对应了一个数字，表示IP协议类型，这些数字的范围是1到255。

以下是定义的每个IP协议类型的含义：

- IP协议类型126表示Combat无线电传输(CRTP)。
- IP协议类型127表示Combat无线电UDP传输(CRUDP)。
- IP协议类型128表示SSCOPMCE(SSCOPMCE)。
- IP协议类型129表示IPLT(IPLT)。
- IP协议类型130表示Secure Packet Shield(SP)。
- IP协议类型131表示Private IP Encap in IP(PIPE)。
- IP协议类型132表示Stream Ctrl Transmission(SCTP)。
- IP协议类型133表示Fibre Channel(FC)。
- IP协议类型134表示RSVPIGN(RSVP-E2E-IGNORE)。

此外，还定义了一个常量IP协议类型255，表示raw IP packets(生肉数据报)。最后，还定义了一个常量IP协议类型MAX，表示所有 reserved IP protocol(保留的IP协议)。


```cpp
#define IP_PROTO_CRTP		126		/* Combat Radio Transport */
#define IP_PROTO_CRUDP		127		/* Combat Radio UDP */
#define IP_PROTO_SSCOPMCE	128		/* SSCOPMCE */
#define IP_PROTO_IPLT		129		/* IPLT */
#define IP_PROTO_SPS		130		/* Secure Packet Shield */
#define IP_PROTO_PIPE		131		/* Private IP Encap in IP */
#define IP_PROTO_SCTP		132		/* Stream Ctrl Transmission */
#define IP_PROTO_FC		133		/* Fibre Channel */
#define IP_PROTO_RSVPIGN	134		/* RSVP-E2E-IGNORE */
#define	IP_PROTO_RAW		255		/* Raw IP packets */
#define IP_PROTO_RESERVED	IP_PROTO_RAW	/* Reserved */
#define	IP_PROTO_MAX		255

/*
 * Option types (opt_type) - http://www.iana.org/assignments/ip-parameters
 */
```

这段代码定义了一系列IP控制选项，包括控制、调试测量、复制、保留等。其中，IP_OPT_CONTROL为控制选项，IP_OPT_DEBMEAS为调试测量选项，IP_OPT_COPY为复制选项，IP_OPT_RESERVED1和IP_OPT_RESERVED2为保留选项。IP_OPT_EOL表示选项列表的结束，IP_OPT_NOP表示不做任何操作，IP_OPT_SEC表示基本安全选项，IP_OPT_LSRR表示宽松源路由选项，IP_OPT_TS表示时间戳选项，IP_OPT_ESEC表示扩展安全选项，IP_OPT_CIPSO表示商业安全选项，IP_OPT_RR表示记录路由选项，IP_OPT_SATID表示Stream ID（已弃用）选项。这些选项可以用于对IP数据包进行控制，以满足一些安全需求。


```cpp
#define IP_OPT_CONTROL		0x00		/* control */
#define IP_OPT_DEBMEAS		0x40		/* debugging & measurement */
#define IP_OPT_COPY		0x80		/* copy into all fragments */
#define IP_OPT_RESERVED1	0x20
#define IP_OPT_RESERVED2	0x60

#define IP_OPT_EOL	  0			/* end of option list */
#define IP_OPT_NOP	  1			/* no operation */
#define IP_OPT_SEC	 (2|IP_OPT_COPY)	/* DoD basic security */
#define IP_OPT_LSRR	 (3|IP_OPT_COPY)	/* loose source route */
#define IP_OPT_TS	 (4|IP_OPT_DEBMEAS)	/* timestamp */
#define IP_OPT_ESEC	 (5|IP_OPT_COPY)	/* DoD extended security */
#define IP_OPT_CIPSO	 (6|IP_OPT_COPY)	/* commercial security */
#define IP_OPT_RR	  7			/* record route */
#define IP_OPT_SATID	 (8|IP_OPT_COPY)	/* stream ID (obsolete) */
```

这段代码定义了一系列选项标志，用于配置IP协议栈中的参数。以下是每个选项标志的简要解释：

1. IP_OPT_SSRR：这是一个严格源路由的选项标志，通常用于IPv4和IPv6网络中。
2. IP_OPT_ZSU：这是一个实验性的测量选项，用于指定是否应该在IPv4和IPv6头中使用Zeroized Sequence Unity (ZSU) 选项。
3. IP_OPT_MTUP：这是一个用于MTU（Message Transfer Unit）测量的选项，通常用于IPv4和IPv6网络中。
4. IP_OPT_MTUR：这是一个用于MTU回复的选项，通常用于IPv4和IPv6网络中。
5. IP_OPT_FINN：这是一个用于经验性流量控制选项的选项标志，包括拥塞控制和流量限制。
6. IP_OPT_VISA：这是一个用于实验性访问控制选项的选项标志，通常用于IPv4和IPv6网络中。
7. IP_OPT_ENCODE：这是一个未知的选项标志，通常用于IPv4和IPv6网络中。
8. IP_OPT_IMITD：这是一个用于IMI（Indication Message Indication Table）选项的选项标志，通常用于IPv4和IPv6网络中。
9. IP_OPT_EIP：这是一个用于RFC 1385扩展IP选项的选项标志，通常用于IPv4和IPv6网络中。
10. IP_OPT_TR：这是一个用于tracerout的选项标志，通常用于IPv4和IPv6网络中。
11. IP_OPT_ADDExt：这是一个用于IPv7扩展地址选项的选项标志，通常用于IPv4和IPv6网络中。
12. IP_OPT_RTRALT：这是一个用于路由器警报（RFC 2113）的选项标志，通常用于IPv4和IPv6网络中。
13. IP_OPT_SDB：这是一个用于指定方向广播选项的选项标志，通常用于IPv4和IPv6网络中。
14. IP_OPT_NSAPA：这是一个用于NSAP（Network Separated Address Protocol）地址选项的选项标志，通常用于IPv4和IPv6网络中。
15. IP_OPT_DPS：这是一个未知的选项标志，通常用于IPv4和IPv6网络中。


```cpp
#define IP_OPT_SSRR	 (9|IP_OPT_COPY)	/* strict source route */
#define IP_OPT_ZSU	 10			/* experimental measurement */
#define IP_OPT_MTUP	 11			/* MTU probe */
#define IP_OPT_MTUR	 12			/* MTU reply */
#define IP_OPT_FINN	(13|IP_OPT_COPY|IP_OPT_DEBMEAS)	/* exp flow control */
#define IP_OPT_VISA	(14|IP_OPT_COPY)	/* exp access control */
#define IP_OPT_ENCODE	 15			/* ??? */
#define IP_OPT_IMITD	(16|IP_OPT_COPY)	/* IMI traffic descriptor */
#define IP_OPT_EIP	(17|IP_OPT_COPY)	/* extended IP, RFC 1385 */
#define IP_OPT_TR	(18|IP_OPT_DEBMEAS)	/* traceroute */
#define IP_OPT_ADDEXT	(19|IP_OPT_COPY)	/* IPv7 ext addr, RFC 1475 */
#define IP_OPT_RTRALT	(20|IP_OPT_COPY)	/* router alert, RFC 2113 */
#define IP_OPT_SDB	(21|IP_OPT_COPY)	/* directed bcast, RFC 1770 */
#define IP_OPT_NSAPA	(22|IP_OPT_COPY)	/* NSAP addresses */
#define IP_OPT_DPS	(23|IP_OPT_COPY)	/* dynamic packet state */
```

这段代码定义了一系列头文件，用于定义IP选项的数据结构体和常量。主要作用是定义IP选项的功能和选项，以便在网络编程中对IP协议进行定义。

具体来说，以下代码定义了以下几个IP选项：

1. IP_OPT_UMP：上游 multicast，将IP_OPT_COPY的第二个参数赋值为24，即 2^24。
2. IP_OPT_MAX：设置 IP_OPT_COPY的最大数量。
3. IP_OPT_COPY：复制 IP 选项，包括复制选项的值。
4. IP_OPT_CLASS：定义 IP 选项的类别，可能的取值包括0、1和2。
5. IP_OPT_NUMBER：定义 IP 选项的数字，可能的取值包括0-255。
6. IP_OPT_TYPEONLY：如果 IP 选项是 EOF（0）或 NOP（255），则仅包含此选项，否则不包含此选项。
7. IP_OPT_DATA_SEC：定义 IP 选项的数据安全选项，包括安全选项的值。

这些选项的具体含义可以在 RFC 791和 3.1 中查阅。


```cpp
#define IP_OPT_UMP	(24|IP_OPT_COPY)	/* upstream multicast */
#define IP_OPT_MAX	 25

#define IP_OPT_COPIED(o)	((o) & 0x80)
#define IP_OPT_CLASS(o)		((o) & 0x60)
#define IP_OPT_NUMBER(o)	((o) & 0x1f)
#define IP_OPT_TYPEONLY(o)	((o) == IP_OPT_EOL || (o) == IP_OPT_NOP)

/*
 * Security option data - RFC 791, 3.1
 */
struct ip_opt_data_sec {
	uint16_t	s;		/* security */
	uint16_t	c;		/* compartments */
	uint16_t	h;		/* handling restrictions */
	uint8_t		tcc[3];		/* transmission control code */
} __attribute__((__packed__));

```

这段代码定义了一系列IP选项的数据类型，用于在网络协议中传输安全信息。下面是对每个选项的简要解释：

#define IP_OPT_SEC_UNCLASS    0x0000    /* unclassified */
#define IP_OPT_SEC_CONFID     0xf135    /* confidential */
#define IP_OPT_SEC_EFTO       0x789a    /* EFTO */
#define IP_OPT_SEC_MMMM     0xbc4d    /* MMMM */
#define IP_OPT_SEC_PROG       0x5e26    /* PROG */
#define IP_OPT_SEC_RESTR     0xaf13    /* restricted */
#define IP_OPT_SEC_SECRET     0xd788    /* secret */
#define IP_OPT_SEC_TOPSECRET  0x6bc5    /* top secret */

这个定义了8个IP选项数据类型，包括路由选项和安全选项。这些选项根据其值被赋予不同的标签，以便网络设备能够正确地处理它们。这些标签由一个十六进制数和一个字符串组成，它们表示了IP选项的类型和类别。

例如，IP选项OPT_SEC_CONFID的值为0xf135，表示这是采用了Fast Progress加密(FPC)的选项。另一个选项IP_OPT_SEC_RESTR的值为0xaf13，表示这是有限区域安全(LRS)选项。

这些标签的格式如下：

标签：0x0000 - 0x7fffff   - 0xf135    - 0x789a    - 0xbc4d    - 0x5e26    - 0xaf13    - 0xd788    - 0x6bc5

前24位是选项类型，接下来的4位是选项数据类型，最后的28位是选项数据。

这些标签用于在IP协议中传输安全信息，以便设备能够正确地识别和处理它们。


```cpp
#define IP_OPT_SEC_UNCLASS	0x0000	/* unclassified */
#define IP_OPT_SEC_CONFID	0xf135	/* confidential */
#define IP_OPT_SEC_EFTO		0x789a	/* EFTO */
#define IP_OPT_SEC_MMMM		0xbc4d	/* MMMM */
#define IP_OPT_SEC_PROG		0x5e26	/* PROG */
#define IP_OPT_SEC_RESTR	0xaf13	/* restricted */
#define IP_OPT_SEC_SECRET	0xd788	/* secret */
#define IP_OPT_SEC_TOPSECRET	0x6bc5	/* top secret */

/*
 * {Loose Source, Record, Strict Source} Route option data - RFC 791, 3.1
 */
struct ip_opt_data_rr {
	uint8_t		ptr;		/* from start of option, >= 4 */
	uint32_t	iplist __flexarr; /* list of IP addresses */
} __attribute__((__packed__));

```

该代码定义了一个名为`ip_opt_data_ts`的结构体，用于表示IP选项数据中的时间戳选项。

该结构体有两个成员变量，一个是指针类型`ptr`，表示从选项开始的位置(或者是`DNET_BYTESEX`中定义的大端或小端字节顺序)，另一个是用于表示IP数量和时间戳的标记位，分别为`oflw`和`flg`。

如果`DNET_BYTESEX`为`DNET_BIG_ENDIAN`，则`flg`表示为`/`字符串中的IP数量，`oflw`表示跳过了多少个IP地址。如果`DNET_BYTESEX`为`DNET_LIL_ENDIAN`，则`flg`表示为`/`字符串中的IP数量，`oflw`表示跳过了多少个IP地址。

另外，该结构体还有一个成员变量`ipts`，表示IP地址 pairs，是一个32位无符号整数。

该结构体使用了`__attribute__((__packed__))`的修饰，意味着它的成员变量按字节对齐，并且每个成员变量都能够被正确地无线缆化( serialized )。


```cpp
/*
 * Timestamp option data - RFC 791, 3.1
 */
struct ip_opt_data_ts {
	uint8_t		ptr;		/* from start of option, >= 5 */
#if DNET_BYTESEX == DNET_BIG_ENDIAN
	uint8_t		oflw:4,		/* number of IPs skipped */
	    		flg:4;		/* address[ / timestamp] flag */
#elif DNET_BYTESEX == DNET_LIL_ENDIAN
	uint8_t		flg:4,
			oflw:4;
#endif
	uint32_t	ipts __flexarr;	/* IP address [/ timestamp] pairs */
} __attribute__((__packed__));

```

这段代码定义了三个IP选项：TS（timestamps only）、TSADDR（IP address / timestamp pairs）、PRESPEC（IP address / zero timestamp pairs）。

这些选项数据保存在一个名为ip_opt_data_tr的结构体中。这个结构体定义了两个成员：id和ohc，分别表示选项的ID和出站跳数；另外两个成员rhc和origip，表示回程跳数和原始发送IP地址。这些成员都是16位的无符号整数。

定义这些选项的目的是在编译时检查是否定义了这些选项，如果没有定义，则编译器会报错，从而确保程序在编译时就能遵循RFC 1393规范。


```cpp
#define IP_OPT_TS_TSONLY	0	/* timestamps only */
#define IP_OPT_TS_TSADDR	1	/* IP address / timestamp pairs */
#define IP_OPT_TS_PRESPEC	3	/* IP address / zero timestamp pairs */

/*
 * Traceroute option data - RFC 1393, 2.2
 */
struct ip_opt_data_tr {
	uint16_t	id;		/* ID number */
	uint16_t	ohc;		/* outbound hop count */
	uint16_t	rhc;		/* return hop count */
	uint32_t	origip;		/* originator IP address */
} __attribute__((__packed__));

/*
 * IP option (following IP header)
 */
```

这段代码定义了一个名为`ip_opt`的结构体，用于表示IP选项数据。该结构体包含一个名为`opt_type`的8位整型变量，表示选项类型；一个名为`opt_len`的8位整型变量，表示选项长度，必须大于等于`IP_OPT_LEN`；一个名为`opt_data`的联合体类型变量，用于存储IP选项数据。

`opt_data`联合体类型变量可以包含以下几种成员：

1. `struct ip_opt_data_sec`类型成员，表示IP选项的设置类型为开启或关闭。
2. `struct ip_opt_data_rr`类型成员，表示IP选项的设置类型为IPv4或IPv6链路层地址。
3. `struct ip_opt_data_ts`类型成员，表示IP选项的设置类型为时间戳。
4. `uint16_t`类型成员，表示IP选项的设置类型为子站地址。
5. `uint16_t`类型成员，表示IP选项的设置类型为多播。
6. `struct ip_opt_data_tr`类型成员，表示IP选项的设置类型为传输。
7. `uint32_t`类型成员，表示IP选项的设置类型为自定义选项。
8. `uint32_t`类型成员，表示IP选项的设置类型为AID。
9. `uint8_t`类型成员，表示IP选项的设置类型为数据8。
10. `uint8_t`类型成员，表示IP选项的设置类型为MAX_OPT_LEN。

此外，还有一个名为`ip_opt_len`的常量，用于指定该结构体变量的最大长度，单位为字节。该常量的默认值为`IP_OPT_LEN`，表示IP选项的最大长度为`IP_OPT_LEN`字节。


```cpp
struct ip_opt {
	uint8_t		opt_type;	/* option type */
	uint8_t		opt_len;	/* option length >= IP_OPT_LEN */
	union ip_opt_data {
		struct ip_opt_data_sec	sec;	   /* IP_OPT_SEC */
		struct ip_opt_data_rr	rr;	   /* IP_OPT_{L,S}RR */
		struct ip_opt_data_ts	ts;	   /* IP_OPT_TS */
		uint16_t		satid;	   /* IP_OPT_SATID */
		uint16_t		mtu;	   /* IP_OPT_MTU{P,R} */
		struct ip_opt_data_tr	tr;	   /* IP_OPT_TR */
		uint32_t		addext[2]; /* IP_OPT_ADDEXT */
		uint16_t		rtralt;    /* IP_OPT_RTRALT */
		uint32_t		sdb[9];    /* IP_OPT_SDB */
		uint8_t			data8[IP_OPT_LEN_MAX - IP_OPT_LEN];
	} opt_data;
} __attribute__((__packed__));

```

这段代码定义了一系列关于IP地址分类的宏，用于对IP地址进行分类。

首先，定义了几个IP地址分类的宏：

```cpp
#define	IP_CLASSA(i)		(((uint32_t)(i) & htonl(0x80000000)) == \
				 htonl(0x00000000))
#define	IP_CLASSA_NET		(htonl(0xff000000))
#define	IP_CLASSA_NSHIFT	24
#define	IP_CLASSA_HOST		(htonl(0x00ffffff))
#define	IP_CLASSA_MAX		128
```

IPv4地址分类枚举，将IP地址分为四个类别：CLASSA、CLASSA_NET、CLASSA_NSHIFT和CLASSA_HOST。其中，CLASSA是最重要的分类，用于将IP地址划分到相应的类别中。

IPv6地址分类枚举，将IPv6地址分为六个类别：CLASSA、CLASSA_NET、CLASSA_NSHIFT、CLASSA_HOST、CLASSA_MASK和CLASSA_MAX。其中，CLASSA是最重要的分类，用于将IPv6地址划分到相应的类别中。

然后，定义了一个名为“IP_CLASSB(i)”的宏，表示IPv6地址CLASSA_HOST类别的分类。

```cpp
#define	IP_CLASSB(i)		(((uint32_t)(i) & htonl(0xc0000000)) == \
				 htonl(0x80000000))
```

这个宏表示IPv6地址CLASSA_HOST类别的分类，它与IPv4中的CLASSA_HOST类别的含义是相同的。


```cpp
#ifndef __GNUC__
# pragma pack()
#endif

/*
 * Classful addressing
 */
#define	IP_CLASSA(i)		(((uint32_t)(i) & htonl(0x80000000)) == \
				 htonl(0x00000000))
#define	IP_CLASSA_NET		(htonl(0xff000000))
#define	IP_CLASSA_NSHIFT	24
#define	IP_CLASSA_HOST		(htonl(0x00ffffff))
#define	IP_CLASSA_MAX		128

#define	IP_CLASSB(i)		(((uint32_t)(i) & htonl(0xc0000000)) == \
				 htonl(0x80000000))
```

这段代码定义了一系列IP 地址类别，用于区分网络地址和主机地址。以下是每个定义的作用：

1. `#define IP_CLASSB_NET`：定义了一个名为IP_CLASSB_NET的常量，代表IPv4地址中的网络部分，通常由子网掩码表示。
2. `#define IP_CLASSB_NSHIFT`：定义了一个名为IP_CLASSB_NSHIFT的常量，代表IPv4地址中的子网掩码位数，用于将IP地址分成网络地址和主机地址两部分。
3. `#define IP_CLASSB_HOST`：定义了一个名为IP_CLASSB_HOST的常量，代表IPv4地址中的主机部分，通常由私有IP地址表示。
4. `#define IP_CLASSB_MAX`：定义了一个名为IP_CLASSB_MAX的常量，表示IPv4地址中网络地址和主机地址的最大值，即2^32-2。

1. `#define IP_CLASSC(i)`：定义了一个名为IP_CLASSC的函数，用于将IPv4地址中的网络部分和子网掩码位数组合成一个二进制数。这个函数将IP地址转换为二进制形式，并在二进制形式中找到网络部分和子网掩码位数，然后将它们组合成一个返回值。
2. `#define IP_CLASSC_NET`：定义了一个名为IP_CLASSC_NET的常量，代表IPv4地址中的网络部分，通常由子网掩码表示。
3. `#define IP_CLASSC_NSHIFT`：定义了一个名为IP_CLASSC_NSHIFT的常量，代表IPv4地址中的子网掩码位数，用于将IP地址分成网络地址和主机地址两部分。
4. `#define IP_CLASSC_HOST`：定义了一个名为IP_CLASSC_HOST的常量，代表IPv4地址中的主机部分，通常由私有IP地址表示。
5. `#define IP_CLASSD(i)`：定义了一个名为IP_CLASSD的函数，用于将IPv4地址中的网络部分和私有IP地址组合成一个二进制数。这个函数将IP地址转换为二进制形式，并在二进制形式中找到网络部分和私有IP地址，然后将它们组合成一个返回值。
6. `#define IP_CLASSD_NET`：定义了一个名为IP_CLASSD_NET的常量，代表IPv4地址中的网络部分，通常由子网掩码表示。
7. `/* These ones aren't really net and host fields, but routing needn't know. */`：这个注释说明了这些选项并不是真正的网络和主机地址，它们在路由器中可能不需要进行区分，但是仍然需要知道IPv4地址的格式。


```cpp
#define	IP_CLASSB_NET		(htonl(0xffff0000))
#define	IP_CLASSB_NSHIFT	16
#define	IP_CLASSB_HOST		(htonl(0x0000ffff))
#define	IP_CLASSB_MAX		65536

#define	IP_CLASSC(i)		(((uint32_t)(i) & htonl(0xe0000000)) == \
				 htonl(0xc0000000))
#define	IP_CLASSC_NET		(htonl(0xffffff00))
#define	IP_CLASSC_NSHIFT	8
#define	IP_CLASSC_HOST		(htonl(0x000000ff))

#define	IP_CLASSD(i)		(((uint32_t)(i) & htonl(0xf0000000)) == \
				 htonl(0xe0000000))
/* These ones aren't really net and host fields, but routing needn't know. */
#define	IP_CLASSD_NET		(htonl(0xf0000000))
```

这段代码定义了一系列IP头，用于定义IPv6头中的选项字段。它们包括：

1. #define IP_CLASSD_NSHIFT 28：定义了一个名为IP_CLASSD_NSHIFT的结构体，它包含一个16位的网络类标识（NSHIFT）。
2. #define IP_CLASSD_HOST (htonl(0x0fffffff))：定义了一个名为IP_CLASSD_HOST的结构体，它包含一个32位的网络主机地址。
3. #define IP_MULTICAST(i) IP_CLASSD(i)：定义了一个名为IP_MULTICAST的结构体，它包含一个IPv6多播地址。
4. #define IP_EXPERIMENTAL(i) (((uint32_t)(i) & htonl(0xf0000000)) == (htonl(0xf0000000) << 1))：定义了一个名为IP_EXPERIMENTAL的结构体，它包含一个IPv6实验性选项字段，根据该字段的值可以判断IPv6是否处于实验性状态。
5. #define IP_BADCLASS(i) (((uint32_t)(i) & htonl(0xf0000000)) == (htonl(0xf0000000) << 1))：定义了一个名为IP_BADCLASS的结构体，它包含一个IPv6 bad class选项字段，根据该字段的值可以判断IPv6是否属于不良类别。
6. #define IP_LOCAL_GROUP(i) (((uint32_t)(i) & htonl(0xffffff00)) == (htonl(0xe0000000) << 1))：定义了一个名为IP_LOCAL_GROUP的结构体，它包含一个IPv6本地组选项字段，根据该字段的值可以判断IPv6是否属于本地组。
7. #define IP_ADDR_ANY (htonl(0x00000000))：定义了一个名为IP_ADDR_ANY的结构体，它包含一个IPv6任何cast地址。
8. #define IP_ADDR_BROADCAST (htonl(0xffffffff))：定义了一个名为IP_ADDR_BROADCAST的结构体，它包含一个IPv6广播地址。


```cpp
#define	IP_CLASSD_NSHIFT	28
#define	IP_CLASSD_HOST		(htonl(0x0fffffff))
#define	IP_MULTICAST(i)		IP_CLASSD(i)

#define	IP_EXPERIMENTAL(i)	(((uint32_t)(i) & htonl(0xf0000000)) == \
				 htonl(0xf0000000))
#define	IP_BADCLASS(i)		(((uint32_t)(i) & htonl(0xf0000000)) == \
				 htonl(0xf0000000))
#define	IP_LOCAL_GROUP(i)	(((uint32_t)(i) & htonl(0xffffff00)) == \
				 htonl(0xe0000000))
/*
 * Reserved addresses
 */
#define IP_ADDR_ANY		(htonl(0x00000000))	/* 0.0.0.0 */
#define IP_ADDR_BROADCAST	(htonl(0xffffffff))	/* 255.255.255.255 */
```

这段代码定义了一系列IP地址的宏定义，包括IP_ADDR_LOOPBACK、IP_ADDR_MCAST_ALL和IP_ADDR_MCAST_LOCAL。这些宏定义用于在代码中使用IP地址，以简化代码的编写。

IP_ADDR_LOOPBACK表示127.0.0.1，即回环地址，通常用于测试网络接口。

IP_ADDR_MCAST_ALL表示224.0.0.1，这是一个保留地址，通常用于ARP协议中的目标地址。

IP_ADDR_MCAST_LOCAL表示224.0.0.255，这是一个保留地址，通常用于IPv6协议中的私有地址。

ip_pack_hdr函数的作用是将IP头中的各个字段打包成二进制数据，以便在网络传输过程中进行传输。该函数接收输入的IP头和一个IP数据报头，将数据报头中的源IP地址、目标IP地址、数据长度和时间戳等信息打包成一个IP数据报头，然后将IP数据报头中的源IP地址、目标IP地址和数据报头信息打包成一个IP数据报头，最后将两个IP数据报头合并为一个二进制数据包，以实现IP数据报的传输。


```cpp
#define IP_ADDR_LOOPBACK	(htonl(0x7f000001))	/* 127.0.0.1 */
#define IP_ADDR_MCAST_ALL	(htonl(0xe0000001))	/* 224.0.0.1 */
#define IP_ADDR_MCAST_LOCAL	(htonl(0xe00000ff))	/* 224.0.0.255 */

#define ip_pack_hdr(hdr, tos, len, id, off, ttl, p, src, dst) do {	\
	struct ip_hdr *ip_pack_p = (struct ip_hdr *)(hdr);		\
	ip_pack_p->ip_v = 4; ip_pack_p->ip_hl = 5;			\
	ip_pack_p->ip_tos = tos; ip_pack_p->ip_len = htons(len);	\
 	ip_pack_p->ip_id = htons(id); ip_pack_p->ip_off = htons(off);	\
	ip_pack_p->ip_ttl = ttl; ip_pack_p->ip_p = p;			\
	ip_pack_p->ip_src = src; ip_pack_p->ip_dst = dst;		\
} while (0)

typedef struct ip_handle ip_t;

```

这段代码是一个用于配置IP头中的选项的函数。它定义了以下函数：

* `ip_open()`：打开一个IP头缓冲区，并返回一个指向IP头缓冲区的指针。
* `ip_send()`：将一个IP头缓冲区和目标IP地址作为参数，返回发送该缓冲区中的数据包的长度。
* `ip_close()`：关闭一个IP头缓冲区。
* `ip_ntop()`：将IP地址转换为字符串，并使用目标IP地址作为格式化字符。
* `ip_pton()`：将一个IP地址字符串转换为IP地址。
* `ip_ntoa()`：将IP地址转换为字符串。
* `ip_aton()`：将IP地址字符串转换为IP地址。
* `ip_add_option()`：向IP头缓冲区中添加一个选项。
* `ip_checksum()`：计算IP头缓冲区中的数据包的校验和。
* `ip_cksum_add()`：计算IP头缓冲区中的数据包的校验和，并增加指定的选项的长度。

该函数的实现看起来像是用于在IP头中添加选项的库函数，但实际上它并没有对IP头数据进行任何修改，而只是简单地将IP头缓冲区中的数据作为字符串返回。


```cpp
__BEGIN_DECLS
ip_t	*ip_open(void);
ssize_t	 ip_send(ip_t *i, const void *buf, size_t len);
ip_t	*ip_close(ip_t *i);

char	*ip_ntop(const ip_addr_t *ip, char *dst, size_t len);
int	 ip_pton(const char *src, ip_addr_t *dst);
char	*ip_ntoa(const ip_addr_t *ip);
#define	 ip_aton ip_pton

ssize_t	 ip_add_option(void *buf, size_t len,
	    int proto, const void *optbuf, size_t optlen);
void	 ip_checksum(void *buf, size_t len);

int	 ip_cksum_add(const void *buf, size_t len, int cksum);
```

这段代码是一个定义，定义了一个名为 `ip_cksum_carry` 的函数，它的参数是一个整数类型的变量 `x`。

这个函数的作用是计算 `x` 的哈希值，并返回哈希值。计算哈希值的过程中，使用了异或运算和加法运算，其中异或运算可以快速计算两个二进制数产生序的不同结果，而加法运算则可以保证结果的正确性。

具体来说，函数的实现过程如下：

1. 将 `x` 左移 16 位，并加上本地的最高位(也就是 0xffff)，这样就可以计算出 `x` 的哈希值。
2. 将哈希值向左移动 16 位，同时将最高位清零，这样就可以保证哈希值的正确性。
3. 对哈希值再次进行异或运算，就可以得到最终的哈希值。

由于哈希值是用于快速定位数据记录的，因此它的计算方式需要尽可能简单快速，同时也需要保证结果的正确性。


```cpp
#define	 ip_cksum_carry(x) \
	    (x = (x >> 16) + (x & 0xffff), (~(x + (x >> 16)) & 0xffff))
__END_DECLS

#endif /* DNET_IP_H */

```

# `libdnet-stripped/include/dnet/ip6.h`

这段代码定义了一个名为ip6.h的文件，它是IPv6协议的规范。下面是这份文件的内容：

```cpp
/*
* ip6.h
*
* Internet Protocol, Version 6 (RFC 2460).
*
* Copyright (c) 2002 Dug Song <dugsong@monkey.org>
*
* $Id: ip6.h 486 2004-02-23 10:01:15Z dugsong $
*/

#ifndef DNET_IP6_H
#define DNET_IP6_H

#define IP6_ADDR_LEN	16
#define IP6_ADDR_BITS	128
```

该文件中定义了一些IPv6地址相关的常量和类型，其中包括IPv6地址的长度、IPv6地址的位数、IPv6地址类型的数量等。这些常量和类型定义了IPv6地址的基础，是IPv6协议中的重要组成部分。


```cpp
/*
 * ip6.h
 *
 * Internet Protocol, Version 6 (RFC 2460).
 *
 * Copyright (c) 2002 Dug Song <dugsong@monkey.org>
 *
 * $Id: ip6.h 486 2004-02-23 10:01:15Z dugsong $
 */

#ifndef DNET_IP6_H
#define DNET_IP6_H

#define IP6_ADDR_LEN	16
#define IP6_ADDR_BITS	128

```

这段代码定义了一些宏，包括：

1. IP6_HDR_LEN，表示IPv6头部的长度，为40字节。
2. IP6_LEN_MIN，表示IPv6最小有效负载长度的下限，与IP6_HDR_LEN相同。
3. IP6_LEN_MAX，表示IPv6最大有效负载长度的上限，为65535字节。
4. IP6_MTU_MIN，表示IPv6最小MTU（数据分片的最小长度），为1280字节。

接下来是另外两个宏：

5. #define，表示后面要定义的宏。
6. struct ip6_addr，定义了一个名为ip6_addr的结构体，其中包含一个IPv6地址的数据。

整段代码定义了一个IPv6头，其长度为40字节。然后定义了IPv6的最小有效负载长度和最大有效负载长度，以及最小MTU长度。接着定义了一个名为ip6_addr的结构体，其中包含一个IPv6地址的数据。整段代码的功能是定义了一些宏，用于定义IPv6地址的结构体。


```cpp
#define IP6_HDR_LEN	40		/* IPv6 header length */
#define IP6_LEN_MIN	IP6_HDR_LEN
#define IP6_LEN_MAX	65535		/* non-jumbo payload */

#define IP6_MTU_MIN	1280		/* minimum MTU (1024 + 256) */

typedef struct ip6_addr {
	uint8_t         data[IP6_ADDR_LEN];
} ip6_addr_t;

#ifndef __GNUC__
#ifndef __attribute__
# define __attribute__(x)
#endif
# pragma pack(1)
```

这段代码定义了一个名为`ip6_hdr`的结构体，表示IPv6头部。

IPv6头部包含一个名为`ip6_ctlun`的联合体，它包含一个`ip6_un1`结构体，表示IPv6头部的控制信息。

`ip6_un1`结构体包含以下字段：

- `ip6_un1_flow`：20位的流标识，表示数据包中的流量类型。
- `ip6_un1_plen`：表示数据包的长度，单位为4字节。
- `ip6_un1_nxt`：表示下一个IPv6头部的类型。
- `ip6_un1_hlim`：表示数据包的跳限制，单位为2比特。

`ip6_ctlun`包含上述字段，以及一个名为`ip6_src`的`ip6_addr_t`和一个名为`ip6_dst`的`ip6_addr_t`，分别表示数据包的源IPv6地址和目标IPv6地址。

该结构体定义了IPv6头部，用于在传输层处理IPv6数据包。


```cpp
#endif

/*
 * IPv6 header
 */
struct ip6_hdr {
	union {
		struct ip6_hdr_ctl {
			uint32_t	ip6_un1_flow; /* 20 bits of flow ID */
			uint16_t	ip6_un1_plen; /* payload length */
			uint8_t		ip6_un1_nxt;  /* next header */
			uint8_t		ip6_un1_hlim; /* hop limit */
		} ip6_un1;
		uint8_t	ip6_un2_vfc;	/* 4 bits version, top 4 bits class */
	} ip6_ctlun;
	ip6_addr_t	ip6_src;
	ip6_addr_t	ip6_dst;
} __attribute__((__packed__));

```

这段代码定义了一系列头文件和常量，用于定义IPv6协议的头部信息。

具体来说，这些头文件定义了以下内容：

- ip6_vfc:IPv6控制信息头部中VFC(Virtual Function Client)字段的定义。
- ip6_flow:IPv6控制信息头部中Flow信息字段的定义。
- ip6_plen:IPv6控制信息头部中Plen字段的定义。
- ip6_nxt:IPv6控制信息头部中Nxt字段的定义。IPv6协议栈中的IPv6头中的协议字段为0x01，因此这些字段被称为IPv6头中的协议字段。
- ip6_hlim:IPv6控制信息头部中Hlim字段的定义。

另外，一些头文件中定义了一些常量，如IP6_VERSION和IP6_FLOWINFO_MASK,IP6_FLOWLABEL_MASK等等，用于定义IPv6协议的头部信息。

这些定义在网络编程中非常有用，用于定义IPv6头部信息，以使得程序能够正确解析和处理IPv6数据包。


```cpp
#define ip6_vfc		ip6_ctlun.ip6_un2_vfc
#define ip6_flow	ip6_ctlun.ip6_un1.ip6_un1_flow
#define ip6_plen	ip6_ctlun.ip6_un1.ip6_un1_plen
#define ip6_nxt		ip6_ctlun.ip6_un1.ip6_un1_nxt	/* IP_PROTO_* */
#define ip6_hlim	ip6_ctlun.ip6_un1.ip6_un1_hlim

#define IP6_VERSION		0x60
#define IP6_VERSION_MASK	0xf0		/* ip6_vfc version */

#if DNET_BYTESEX == DNET_BIG_ENDIAN
#define IP6_FLOWINFO_MASK	0x0fffffff	/* ip6_flow info (28 bits) */
#define IP6_FLOWLABEL_MASK	0x000fffff	/* ip6_flow label (20 bits) */
#elif DNET_BYTESEX == DNET_LIL_ENDIAN
#define IP6_FLOWINFO_MASK	0xffffff0f	/* ip6_flow info (28 bits) */
#define IP6_FLOWLABEL_MASK	0xffff0f00	/* ip6_flow label (20 bits) */
```

这段代码定义了一些头文件，用于定义IPv6协议中的 Hop Limit。Hop Limit 是一种防止网络中的路由器发送入过长的数据包的措施，它限制了在单个 IP 包中传输的最大数量。

具体来说，`#define IP6_HLIM_DEFAULT 64`定义了一个默认的 Hop Limit，为 64。`#define IP6_HLIM_MAX 255`定义了一个允许的最大 Hop Limit，为 255。这两个定义告诉我们可以使用这两个值来设置或获取 Hop Limit。

此外，`#define IP6_PROTO_IPV6 IP_PROTO_IPV6`定义了一个名为 IP6_PROTO_IPV6 的头文件。这个头文件中包含了一个字符串，用于指定输入或输出接口的 IP 协议类型。

另外，`#define IP6_PROTO_DSTOPTS IP_PROTO_DSTOPTS`定义了一个名为 IP6_PROTO_DSTOPTS 的头文件。这个头文件中也包含了一个字符串，用于指定输入或输出接口的 DST 协议类型。

`#define IP6_PROTO_ROUTING IP_PROTO_ROUTING`定义了一个名为 IP6_PROTO_ROUTING 的头文件。这个头文件中也包含了一个字符串，用于指定输入或输出接口的 Routing 协议类型。

`#define IP6_PROTO_FRAGMENT IP_PROTO_FRAGMENT`定义了一个名为 IP6_PROTO_FRAGMENT 的头文件。这个头文件中也包含了一个字符串，用于指定输入或输出接口的 Fragment 协议类型。

`#define IP6_PROTO_AH IP_PROTO_AH`定义了一个名为 IP6_PROTO_AH 的头文件。这个头文件中也包含了一个字符串，用于指定输入或输出接口的 Authentication Header 协议类型。

`#define IP6_PROTO_ESP IP_PROTO_ESP`定义了一个名为 IP6_PROTO_ESP 的头文件。这个头文件中也包含了一个字符串，用于指定输入或输出接口的 Encapsulation Security Payload 协议类型。

`#define IP6_PROTO_DSTOPTS IP_PROTO_DSTOPTS`定义了一个名为 IP6_PROTO_DSTOPTS 的头文件。这个头文件中也包含了一个字符串，用于指定输入或输出接口的 DST 协议类型。

`#define IP6_PROTO_MAX IP6_PROTO_MAX`定义了一个名为 IP6_PROTO_MAX 的头文件。这个头文件中包含了一个字符串，用于指定输入或输出接口的最大协议类型。


```cpp
#endif

/*
 * Hop limit (ip6_hlim)
 */
#define IP6_HLIM_DEFAULT	64
#define IP6_HLIM_MAX		255

/*
 * Preferred extension header order from RFC 2460, 4.1:
 *
 * IP_PROTO_IPV6, IP_PROTO_HOPOPTS, IP_PROTO_DSTOPTS, IP_PROTO_ROUTING,
 * IP_PROTO_FRAGMENT, IP_PROTO_AH, IP_PROTO_ESP, IP_PROTO_DSTOPTS, IP_PROTO_*
 */

```

这段代码定义了一个名为`ip6_ext_data_routing`的结构体，用于表示IPv6路由头中的路由类型及其相关数据。

该结构体有两个成员变量：`type`和`segleft`，分别表示路由类型和分片偏移量。这两个成员变量都是8位整数类型，并包含一个可变长度的`reserved`字段，用于保留保留的4个字节。

另外，该结构体还定义了一个名为`slmap`的8位对齐的数组，用于表示IPv6地址的松散/严格映射。这个数组最多可以包含64个连续的IPv6地址。

最后，该结构体定义了一个名为`addr`的16字长度的数组，用于表示IPv6地址，最多可以包含23个地址。

该`ip6_ext_data_routing`结构体可以被用于在IPv6路由头中指定路由类型和分片偏移量。它还可以被用于在IPv6协议头中定义IPv6路由的格式。


```cpp
/*
 * Routing header data (IP_PROTO_ROUTING)
 */
struct ip6_ext_data_routing {
	uint8_t  type;			/* routing type */
	uint8_t  segleft;		/* segments left */
	/* followed by routing type specific data */
} __attribute__((__packed__));

struct ip6_ext_data_routing0 {
	uint8_t  type;			/* always zero */
	uint8_t  segleft;		/* segments left */
	uint8_t  reserved;		/* reserved field */
	uint8_t  slmap[3];		/* strict/loose bit map */
	ip6_addr_t  addr[1];		/* up to 23 addresses */
} __attribute__((__packed__));

```

这段代码定义了一个名为ip6_ext_data_fragment的结构体，用于表示IPv6头部的数据分片。

ip6_ext_data_fragment结构体包含两个成员，一个是offset，表示分片数据在IPv6头部数据中的偏移量，二是ident，表示分片数据的标识。

接下来的代码定义了几个宏，用于计算分片数据的偏移量、保留位和标志。

ip6_off_mask是一个32位的宏，表示分片数据的偏移量范围。它由IP6_OFF_MASK、IP6_RESERVED_MASK和IP6_MORE_FRAG标志位计算得出。

ip6_reserved_mask是一个16位的宏，表示保留位。

ip6_more_frag是一个8位的宏，表示是否包含更多的分片数据。

最后，ip6_ext_data_fragment结构体的定义是在一个名为ip6_fragmentation_regions的函数中。这个函数定义了几个宏，用于计算分片数据的偏移量和保留位。


```cpp
/*
 * Fragment header data (IP_PROTO_FRAGMENT)
 */
struct ip6_ext_data_fragment {
	uint16_t  offlg;		/* offset, reserved, and flag */
	uint32_t  ident;		/* identification */
} __attribute__((__packed__));

/*
 * Fragmentation offset, reserved, and flags (offlg)
 */
#if DNET_BYTESEX == DNET_BIG_ENDIAN
#define IP6_OFF_MASK		0xfff8	/* mask out offset from offlg */
#define IP6_RESERVED_MASK	0x0006	/* reserved bits in offlg */
#define IP6_MORE_FRAG		0x0001	/* more-fragments flag */
```

这段代码是一个C语言代码，定义了一些IPv6头部的选项类型。

首先，定义了两个常量：IP6_OFF_MASK和IP6_RESERVED_MASK，它们用于计算IPv6头部的出偏移。

然后，定义了一个IP6_MORE_FRAG常量，它的值为1，表示是否支持多播。

接着，定义了一些IP6选项类型：IP6_OPT_PAD1、IP6_OPT_PADN和IP6_OPT_JUMBO。这些选项类型都有对应的选项字段，用于在IPv6头部中指定特定的选项。这些选项字段都有一个默认值，如果没有指定，则使用默认的选项。

最后，定义了一个IP6_OPT_RTALERT和IP6_OPT_RTALERT_LEN常量。这些选项用于启用或配置IPv6路由器警报，并定义了相应的选项长度。

这些头部选项用于IPv6协议，用于定义IPv6头部中包含的选项，以便正确地组装和解析IPv6数据包。


```cpp
#elif DNET_BYTESEX == DNET_LIL_ENDIAN
#define IP6_OFF_MASK		0xf8ff	/* mask out offset from offlg */
#define IP6_RESERVED_MASK	0x0600	/* reserved bits in offlg */
#define IP6_MORE_FRAG		0x0100	/* more-fragments flag */
#endif

/*
 * Option types, for IP_PROTO_HOPOPTS, IP_PROTO_DSTOPTS headers
 */
#define IP6_OPT_PAD1		0x00	/* 00 0 00000 */
#define IP6_OPT_PADN		0x01	/* 00 0 00001 */
#define IP6_OPT_JUMBO		0xC2	/* 11 0 00010 = 194 */
#define IP6_OPT_JUMBO_LEN	6
#define IP6_OPT_RTALERT		0x05	/* 00 0 00101 */
#define IP6_OPT_RTALERT_LEN	4
```

这段代码定义了一系列IP6选项，用于标记IP6数据报中的选项。这些选项包括MLD（多路监听）消息、RSVP（资源预留）消息、Active Networks消息以及可选的数据类型，如ICMP（互联网控制消息协议）消息等。

具体来说，这段代码定义了以下几个选项：

1. IP6_OPT_RTALERT_MLD：表示数据报中包含MLD（多路监听）消息。
2. IP6_OPT_RTALERT_RSVP：表示数据报中包含RSVP（资源预留）消息。
3. IP6_OPT_RTALERT_ACTNET：表示数据报中包含Active Networks消息。
4. IP6_OPT_LEN_MIN：表示IP6选项的最小长度，为2个字节。

此外，还定义了以下几个常量：

1. IP6_OPT_TYPE(o)：表示数据报中的选项类型，通过将选项类型字段（低8位）与0xC0按位与得到。
2. IP6_OPT_TYPE_SKIP：表示在数据报失败时继续处理，不丢弃数据报。
3. IP6_OPT_TYPE_DISCARD：表示在数据报失败时丢弃数据报，但仅在数据报是单播时才执行。
4. IP6_OPT_TYPE_FORCEICMP：表示在数据报失败时强制发送ICMP消息，即使在数据报不是单播时。
5. IP6_OPT_MUTABLE：表示选项数据可以在传输过程中更改。


```cpp
#define IP6_OPT_RTALERT_MLD	0	/* Datagram contains an MLD message */
#define IP6_OPT_RTALERT_RSVP	1	/* Datagram contains an RSVP message */
#define IP6_OPT_RTALERT_ACTNET	2 	/* contains an Active Networks msg */
#define IP6_OPT_LEN_MIN		2

#define IP6_OPT_TYPE(o)		((o) & 0xC0)	/* high 2 bits of opt_type */
#define IP6_OPT_TYPE_SKIP	0x00	/* continue processing on failure */
#define IP6_OPT_TYPE_DISCARD	0x40	/* discard packet on failure */
#define IP6_OPT_TYPE_FORCEICMP	0x80	/* discard and send ICMP on failure */
#define IP6_OPT_TYPE_ICMP	0xC0	/* ...only if non-multicast dst */

#define IP6_OPT_MUTABLE		0x20	/* option data may change en route */

/*
 * Extension header (chained via {ip6,ext}_nxt, following IPv6 header)
 */
```

这段代码定义了一个名为`ip6_ext_hdr`的结构体，用于表示IPv6头部的扩展头部。

该结构体包含三个部分：

1. `ext_nxt`：一个8位无符号整数，表示下一个扩展头部的位置。
2. `ext_len`：一个8位无符号整数，表示下一个扩展头部的长度，以8字节为单位。
3. `ext_data`：一个联合体类型，包含两个可能的值：

 - `struct ip6_ext_data_routing`：表示Routine Identifier，用于标识数据包的路由信息。
 - `struct ip6_ext_data_fragment`：表示Fragment Identifier，用于标识数据包的片段信息。

另外，该结构体还包含一个`__attribute__((__packed__))`的修饰，表示该结构体中的所有成员都被固化，即不可修改。


```cpp
struct ip6_ext_hdr {
	uint8_t  ext_nxt;	/* next header */
	uint8_t  ext_len;	/* following length in units of 8 octets */
	union {
		struct ip6_ext_data_routing	routing;
		struct ip6_ext_data_fragment	fragment;
	} ext_data;
} __attribute__((__packed__));

#ifndef __GNUC__
# pragma pack()
#endif

/*
 * Reserved addresses
 */
```

这段代码定义了两个IP6地址宏定义，分别是IP6_ADDR_UNSPEC和IP6_ADDR_LOOPBACK。IP6_ADDR_UNSPEC定义了一个64字节的IPv6地址，用于全球唯一标识符（Global Unique Identifier）或者IPv6路由器链路local link。IP6_ADDR_LOOPBACK定义了一个64字节的IPv6地址，用于在链路local link上形成环路，用于IPv6路由器链路local link，这个地址会在RFC8008中定义为永久组播地址。

ip6_pack_hdr()函数，该函数将IPv6头信息按照IPv6地址的格式组装，并返回包含已经组装好的IPv6头信息的内存指针ip6。ip6_pack_hdr()函数接受5个参数，分别为IPv6头信息hdr、IPv6 flow control Fc、IPv6 fragment loop plen、IPv6 next header Nxt和IPv6 header limit hlim，分别表示IPv6头信息、无类别域、片段边界对齐校验、跳过选项和最大长度。


```cpp
#define IP6_ADDR_UNSPEC	\
	"\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00"
#define IP6_ADDR_LOOPBACK \
	"\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x01"

#define ip6_pack_hdr(hdr, fc, fl, plen, nxt, hlim, src, dst) do {	\
	struct ip6_hdr *ip6 = (struct ip6_hdr *)(hdr);			\
	ip6->ip6_flow = htonl(((uint32_t)(fc) << 20) |			\
	    (0x000fffff & (fl)));					\
	ip6->ip6_vfc = (IP6_VERSION | ((fc) >> 4));			\
	ip6->ip6_plen = htons((plen));					\
	ip6->ip6_nxt = (nxt); ip6->ip6_hlim = (hlim);			\
	memmove(&ip6->ip6_src, &(src), IP6_ADDR_LEN);			\
	memmove(&ip6->ip6_dst, &(dst), IP6_ADDR_LEN);			\
} while (0);

```

这段代码是一个 C 语言函数，定义了三个名为 ip6_ntop、ip6_pton 和 ip6_ntoa 的函数，以及一个名为 ip6_checksum 的函数。这些函数的作用如下：

1. ip6_ntop 函数：将一个 IPv6 地址作为参数传递给另一个函数，然后将 IPv6 地址转换为字符串，将结果存储在从第二个参数开始的第一个字符和后续的字符中。
2. ip6_pton 函数：将一个字符串转换为一个 IPv6 地址，然后将 IPv6地址存储在从第一个字符开始的第二个字符和后续的字符中。
3. ip6_ntoa 函数：将一个 IPv6 地址转换为字符串，与 ip6_pton 函数类似，但将 IPv6地址转换为字符串，而不是 IP 地址。
4. ip6_checksum 函数：接收一个 IPv6 数据的缓冲区和一个缓冲区的大小，计算并存储数据和缓冲区的和。

整个函数定义了一个名为 ip6_ntop 的函数，它的参数包括一个 IPv6 地址和一个字符串，返回一个字符串，该字符串包含了将 IPv6 地址转换为字符串的结果。


```cpp
__BEGIN_DECLS
char	*ip6_ntop(const ip6_addr_t *ip6, char *dst, size_t size);
int	 ip6_pton(const char *src, ip6_addr_t *dst);
char	*ip6_ntoa(const ip6_addr_t *ip6);
#define	 ip6_aton ip6_pton

void	 ip6_checksum(void *buf, size_t len);
__END_DECLS

#endif /* DNET_IP6_H */

```