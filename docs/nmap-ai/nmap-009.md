# Nmap源码解析 9

# `libdnet-stripped/include/dnet/addr.h`

这段代码定义了一个名为 addr.h 的头文件，其中包含有关网络地址操作的信息。

定义了两个变量 ADDR_TYPE_NONE 和 ADDR_TYPE_ETH，它们的值分别为 0 和 1。这两个变量用于表示网络地址类型，其中 ADDR_TYPE_NONE 表示当前网络地址没有被设置，而 ADDR_TYPE_ETH 表示当前网络地址是以太网地址。

接着定义了一个名为 POSTRD0 的宏，它的值为 0。

由于在 addr.h 中没有定义变量或函数，因此不会输出源代码。


```cpp
/*
 * addr.h
 *
 * Network address operations.
 *
 * Copyright (c) 2000 Dug Song <dugsong@monkey.org>
 *
 * $Id: addr.h 404 2003-02-27 03:44:55Z dugsong $
 */

#ifndef DNET_ADDR_H
#define DNET_ADDR_H

#define ADDR_TYPE_NONE		0	/* No address set */
#define	ADDR_TYPE_ETH		1	/* Ethernet */
```

这段代码定义了两个宏定义，分别是`ADDR_TYPE_IP`和`ADDR_TYPE_IP6`，它们的值为2和3。接着定义了一个结构体`addr`，其中包含了一个`addr_type`字段和一个`addr_bits`字段。addr_type字段表示IPv4地址类型，addr_bits字段表示IPv4地址位。

接下来，在addr结构体中定义了一个`__eth`成员和一个`__ip`成员和一个`__ip6`成员。这三个成员都是联合成员，每个成员都可以取到它们的成员变量。然后定义了一个`__data8`成员和一个`__data16`成员和一个`__data32`成员。它们的初始值分别为`{0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0,


```cpp
#define	ADDR_TYPE_IP		2	/* Internet Protocol v4 */
#define	ADDR_TYPE_IP6		3	/* Internet Protocol v6 */

struct addr {
	uint16_t		addr_type;
	uint16_t		addr_bits;
	union {
		eth_addr_t	__eth;
		ip_addr_t	__ip;
		ip6_addr_t	__ip6;
		
		uint8_t		__data8[16];
		uint16_t	__data16[8];
		uint32_t	__data32[4];
	} __addr_u;
};
```

这段代码定义了几个宏定义，包括 `addr_eth`, `addr_ip`, `addr_ip6`, `addr_data8`, `addr_data16`, `addr_data32`，以及一个内部函数 `addr_pack`。

这些宏定义用于定义结构体 `addr` 类型的成员变量。具体来说：

- `addr_eth`定义了 `addr` 类型的 `addr_type` 成员变量为 `__addr_u.__eth`，其中 `__addr_u` 是 `<ascii_number>` 类型的别名，`.__eth` 是 `__addr_u` 类型别名，表示 `addr` 类型所在的以太坊地址类型。
- `addr_ip`定义了 `addr` 类型的 `addr_ip` 成员变量为 `__addr_u.__ip`，其中 `__addr_u` 是 `<ascii_number>` 类型的别名，`.__ip` 是 `__addr_u` 类型别名，表示 `addr` 类型所在的 IPv4 地址类型。
- `addr_ip6` 定义了 `addr` 类型的 `addr_ip6` 成员变量为 `__addr_u.__ip6`，其中 `__addr_u` 是 `<ascii_number>` 类型的别名，`.__ip6` 是 `__addr_u` 类型别名，表示 `addr` 类型所在的 IPv6 地址类型。
- `addr_data8` 和 `addr_data16` 定义了 `addr` 类型的 `addr_data8` 和 `addr_data16` 成员变量为 `__addr_u.__data8` 和 `__addr_u.__data16`，其中 `__addr_u` 是 `<ascii_number>` 类型的别名，`.__data8` 和 `.__data16` 是 `__addr_u.__data` 类型别名，表示 `addr` 类型所在的数据链路层地址类型。
- `addr_pack` 是一个内部函数，用于将给定的 `addr` 类型转换为字节序列。该函数的参数包括 `addr`、数据类型、位数和数据长度。函数内部使用 `memmove` 函数将给定的数据复制到 `addr` 类型的 `addr_data` 成员变量中。

另外，该代码中没有定义 `__BEGIN_DECLS`，因此它不是内部函数，也不是变量。


```cpp
#define addr_eth	__addr_u.__eth
#define addr_ip		__addr_u.__ip
#define addr_ip6	__addr_u.__ip6
#define addr_data8	__addr_u.__data8
#define addr_data16	__addr_u.__data16
#define addr_data32	__addr_u.__data32

#define addr_pack(addr, type, bits, data, len) do {	\
	(addr)->addr_type = type;			\
	(addr)->addr_bits = bits;			\
	memmove((addr)->addr_data8, (char *)data, len);	\
} while (0)

__BEGIN_DECLS
int	 addr_cmp(const struct addr *a, const struct addr *b);

```

这段代码是一个用于将IP地址转换为字符串的函数，尤其是将IPv4地址转换为字符串。以下是代码的解释：

```cppc
#include <string.h>
#include <errno.h>
#include <arpa/inet.h>
#include <sys/types.h>

// addr_bcast函数，将目标地址作为参数，返回地址类型为void的函数指针。
int addr_bcast(const struct addr *a, struct addr *b)
{
   return 0;
}

// addr_net函数，将目标地址作为参数，返回地址类型为void的函数指针。
int addr_net(const struct addr *a, struct addr *b)
{
   return 0;
}

// addr_ntop函数，将IPv4地址转换为字符串，指定目标字符串大小为size。
char *addr_ntop(const struct addr *src, char *dst, size_t size)
{
   if (size <= 0) {
       return NULL;
   }

   int len = sizeof(int) > size ? sizeof(int) : size;
   int i = 0;
   while (i < len) {
       int bit = ((struct sockaddr_in *)addr_src)->sin_port & 0xFF;
       dst[i] = '0' + (bit < 8 ? 0x00000000 : bit);
       i++;
   }

   while (i < len - 1) {
       int bit = ((struct sockaddr_in *)addr_src)->sin_port & 0xFF;
       dst[i] = '0' + (bit < 8 ? 0x00000000 : bit);
       i++;
   }

   dst[len - 1] = '\0';
   return dst;
}

// addr_pton函数，将字符串转换为IPv4地址，使用地址类型为const char *的函数指针。
int addr_pton(const char *src, struct addr *dst)
{
   int len = strlen(src);
   if (len < sizeof(int)) {
       return 0;
   }

   int i = 0;
   while (i < len) {
       int bit = (int)str[i] - '0';
       dst->sin_family = 0;
       dst->sin_port = (int)src[i] - 0x10000000;
       dst->sin_addr = addr_aton((const struct sockaddr_in *)&dst->sin_addr, (const struct sockaddr_in *)&dst->sin_port, len - 1);

       if (bit < 0) {
           dst->sin_family = 0;
           dst->sin_port = 0;
           dst->sin_addr = 0;
           break;
       }

       i++;
   }

   return 1;
}

// addr_ntoa函数，将IPv4地址转换为字符串，指定目标字符串大小为size。
char *addr_ntoa(const struct addr *a)
{
   if (size <= 0) {
       return NULL;
   }

   int len = sizeof(int) > size ? sizeof(int) : size;
   char *str = (char )malloc((len + 1) * sizeof(char));

   int i = 0;
   while (i < len) {
       int bit = ((struct sockaddr_in *)addr_a)->sin_port & 0xFF;
       str[i] = '0' + (bit < 8 ? 0x00000000 : bit);
       i++;
   }

   str[len] = '\0';
   return str;
}

// addr_rotate函数，将IPv4地址中的高位8位字节复制到目标地址中，并将源地址和目标地址交换。
void addr_rotate(struct addr *dst, const struct addr *src)
{
   int old_len = dst->sin_len;
   int i, j;
   for (i = 0; i < old_len; i++) {
       for (j = i + 8 - 8; j < old_len; j++) {
           int bit = src->sin_addr[i] & 0xFF;
           int temp = dst->sin_addr[i];
           dst->sin_addr[i] = dst->sin_addr[i + 8 - bit];
           dst->sin_addr[i + 8 - bit] = temp;
           dst->sin_len = dst->sin_len - i;
           if (dst->sin_len < 0) {
               break;
           }
       }
   }
}

// addr_judge函数，将IPv4地址转换为二进制字符串，并输出其中的前addr_len个字节。
void addr_judge(const struct addr *a, char *str, size_t size)
{
   int i, j;
   const char *byte_order = "010102030405060708090a0b0c";

   for (i = 0; i < size; i++) {
       int bit = ((struct sockaddr_in *)a->sin_addr)[i] & 0xFF;
       if (byte_order[i] == 0) {
           str[i] = '1' + (bit < 8 ? 0 : bit);
       } else {
           str[i] = '0' + (bit < 8 ? 0 : bit);
       }
   }

   str[size - 1] = '\0';
}

// addr_join函数，将IPv4地址连接到字符串末尾，并输出其中的前addr_len个字节。
void addr_join(const struct addr *a, char *str, size_t size)
{
   int i, j;
   const char *byte_order = "010102030405060708090a0b0c";

   for (i = 0; i < size; i++) {
       int bit = ((struct sockaddr_in *)a->sin_addr)[i] & 0xFF;
       if (byte_order[i] == 0) {
           str[i] = '1' + (bit < 8 ? 0 : bit);
       } else {
           str[i] = '0' + (bit < 8 ? 0 : bit);
       }
   }

   str[size - 1] = '\0';
}

// addr_convert函数，将IPv4地址转换为其他类型的地址，并输出转换后的地址。
void addr_convert(const struct addr *a, struct addr *dst)
{
   int i;
   for (i = 0; i < sizeof(int); i++) {
       int bit = ((struct sockaddr_in *)a->sin_addr)[i] & 0xFF;
       if (bit < 8) {
           dst->sin_len = dst->sin_len - i;
           dst->sin_addr[i] = (int)bit - 0x10;
       } else {
           d


```
int	 addr_bcast(const struct addr *a, struct addr *b);
int	 addr_net(const struct addr *a, struct addr *b);

char	*addr_ntop(const struct addr *src, char *dst, size_t size);
int	 addr_pton(const char *src, struct addr *dst);

char	*addr_ntoa(const struct addr *a);
#define	 addr_aton	addr_pton

int	 addr_ntos(const struct addr *a, struct sockaddr *sa);
int	 addr_ston(const struct sockaddr *sa, struct addr *a);

int	 addr_btos(uint16_t bits, struct sockaddr *sa);
int	 addr_stob(const struct sockaddr *sa, uint16_t *bits);

```cpp

这两函数是实现地址分段机制的配套函数，主要用于将实参传递给形参。通过这两函数实现，可以实现对指针变量进行分段操作，即实参通过地址分段进行传值。

具体来说，`addr_btom`函数接收一个uint16_t类型的带地址的比特数组、一个void类型的掩码指针和一个size_t类型的参数，然后对输入的比特数组按位与掩码计算得到的地址进行大小转化，即将一个uint16_t类型的点分十进制表示的地址转化为一个size_t类型的整数。

`addr_mtob`函数接收一个const void类型的掩码指针、一个size_t类型的参数和一个uint16_t类型的指针，然后将输入的掩码与一个uint16_t类型的点分十进制表示的地址进行与运算，得到一个新的uint16_t类型的点分十进制表示的地址。

这两函数通过位运算实现了地址分段，然后通过大小转化实现了地址的正确获取。


```
int	 addr_btom(uint16_t bits, void *mask, size_t size);
int	 addr_mtob(const void *mask, size_t size, uint16_t *bits);
__END_DECLS

#endif /* DNET_ADDR_H */

```cpp

# `libdnet-stripped/include/dnet/arp.h`

这段代码定义了一个名为"arp.h"的Header，包含了关于地址解析协议(ARP)的定义。

ARP协议是一种将IP地址映射到MAC地址的协议，通过广播查询本地网络上的其他设备来获取目标设备的MAC地址。

代码中定义了一个名为"ARP_HDR_LEN"的常量，表示ARP头部的长度，其值为8字节。

接下来的代码定义了一个名为"arp.h"的Header，包含了更多的定义和声明。这些定义和声明定义了ARP头和ARP请求报文，以及定义了ARP头中的各个字段。


```
/*
 * arp.h
 * 
 * Address Resolution Protocol.
 * RFC 826
 *
 * Copyright (c) 2000 Dug Song <dugsong@monkey.org>
 *
 * $Id: arp.h 416 2003-03-16 17:39:18Z dugsong $
 */

#ifndef DNET_ARP_H
#define DNET_ARP_H

#define ARP_HDR_LEN	8	/* base ARP header length */
```cpp

这段代码定义了一个名为ARP_ETHIP_LEN的宏，它的值为20。这个宏定义了ARP消息的长度。这个定义可以在编译时进行替换，因此不会对程序的运行产生影响。

接下来定义了一个名为arp_hdr的结构体，它存储了一个ARP头。这个头包含了两个8位的硬件地址（base ARP message length）和一个16位的协议地址（format of protocol address）。硬件地址用于存储数据帧的物理地址，协议地址用于存储数据帧的目标地址。

最后，定义了一个名为__attribute__的宏，这个宏可以用于编译时检查，确保使用了正确的语法。如果使用了这个宏，那么这个代码片段就会被编译为arm64-v8a架构的代码，并且不会对程序的运行产生影响。


```
#define ARP_ETHIP_LEN	20	/* base ARP message length */

#ifndef __GNUC__
#ifndef __attribute__
# define __attribute__(x)
#endif
# pragma pack(1)
#endif

/*
 * ARP header
 */
struct arp_hdr {
	uint16_t	ar_hrd;	/* format of hardware address */
	uint16_t	ar_pro;	/* format of protocol address */
	uint8_t		ar_hln;	/* length of hardware address (ETH_ADDR_LEN) */
	uint8_t		ar_pln;	/* length of protocol address (IP_ADDR_LEN) */
	uint16_t	ar_op;	/* operation */
};

```cpp

这段代码定义了一系列ARP（Advanced Random Paper）地址格式的常量，用于描述硬件地址和协议地址。

ARP（Advanced Random Paper）是一种用于在IPv4网络上自动获取MAC地址的协议。ARP地址格式为32位，通常包括一个硬件地址（以太网、令牌总线、AppleTalk等）和一个协议类型字段。

这里定义了以下几种ARP地址格式：

- ARP-HRD-ETH：用于表示通过以太网接口（ETH）获得的数据包，包括硬件地址（以太网地址）和一个协议类型字段（值为0x0001表示以太网）。

- ARP-HRD-IEEE802：用于表示通过IEEE 802标准（如Wi-Fi、蓝牙等）获得的硬件地址，包括硬件地址和一个协议类型字段（值为0x0006表示IEEE 802）。

- ARP-HRD-INFINIBAND：用于表示支持InfinBand（也就是USB扩展总线）的硬件地址，包括硬件地址和一个协议类型字段（值为0x0020表示InfinBand）。

- ARP-HRD-APPLETALK：用于表示支持AppleTalk DDP（数据流行病学协议）的硬件地址，包括硬件地址和一个协议类型字段（值为0x0309表示AppleTalk）。

- ARP-HRD-IEEE80211：用于表示支持IEEE 802.11（Wi-Fi）协议的硬件地址，包括硬件地址和一个协议类型字段（值为0x0321表示IEEE 802.11）。此外，还定义了一个名为ARP-HRD-IEEE80211-PRISM的扩展，用于表示支持IEEE 802.11的硬件地址（值为0x0322表示IEEE 802.11 + prism header）。

- ARP-HRD-IEEE80211_PRISM：与上面定义的ARP-HRD-IEEE80211相同，但使用了PRISM头，用于表示支持IEEE 802.11的硬件地址（值为0x0323表示IEEE 802.11 + radiotap header）。

- ARP-HRD-VOID：表示一个特殊的ARP地址格式，没有实际的使用场景。


```
/*
 * Hardware address format
 */
#define ARP_HRD_ETH 	0x0001	/* ethernet hardware */
#define ARP_HRD_IEEE802	0x0006	/* IEEE 802 hardware */

#define ARP_HRD_INFINIBAND 0x0020 /* InfiniBand */
#define ARP_HRD_APPLETALK 0x0309 /* AppleTalk DDP */
#define ARP_HDR_IEEE80211 0x0321  /* IEEE 802.11 */
#define ARP_HRD_IEEE80211_PRISM 0x0322  /* IEEE 802.11 + prism header */
#define ARP_HRD_IEEE80211_RADIOTAP 0x0323  /* IEEE 802.11 + radiotap header */
#define ARP_HRD_VOID 0xFFFF			/* Void type, nothing is known */

/*
 * Protocol address format
 */
```cpp

这段代码定义了一个名为ARP_PRO_IP的宏，其值为0x0800，表示这是一个IP协议。

接下来定义了几个宏，用于描述ARP操作。其中ARP_OP_REQUEST表示发送ARP请求，ARP_OP_REPLY表示发送ARP回复，ARP_OP_REVREQUEST表示请求解析硬件地址，ARP_OP_REVREPLY表示响应解析协议地址。

接着定义了一个名为arp_ethip的结构体，用于存储ARP以太网和IP的封装。

最后在程序中使用这些宏定义，定义了几个变量，用于表示将要发送的ARP请求和回复，以及将要解析的硬件地址和协议地址。


```
#define ARP_PRO_IP	0x0800	/* IP protocol */

/*
 * ARP operation
 */
#define	ARP_OP_REQUEST		1	/* request to resolve ha given pa */
#define	ARP_OP_REPLY		2	/* response giving hardware address */
#define	ARP_OP_REVREQUEST	3	/* request to resolve pa given ha */
#define	ARP_OP_REVREPLY		4	/* response giving protocol address */

/*
 * Ethernet/IP ARP message
 */
struct arp_ethip {
	uint8_t		ar_sha[ETH_ADDR_LEN];	/* sender hardware address */
	uint8_t		ar_spa[IP_ADDR_LEN];	/* sender protocol address */
	uint8_t		ar_tha[ETH_ADDR_LEN];	/* target hardware address */
	uint8_t		ar_tpa[IP_ADDR_LEN];	/* target protocol address */
};

```cpp

这段代码定义了一个名为`arp_entry`的结构体，用于存储ARP缓存条目。

ARP缓存条目包含两个部分：协议地址（以小写字母h和数字0到255为范围）和硬件地址。

协议地址字段表示ARP协议头中的协议类型，它通常为0x00000001。硬件地址字段包含一个IP地址，用于存储发送ARP请求的硬件地址。

该代码的目的是定义一个函数`arp_pack_hdr_ethip`，该函数将ARP协议头和硬件地址组装成二进制数据，以便在`memmove`函数中进行复制。

这个函数的实现需要以下步骤：

1. 定义`arp_pack_hdr_ethip`函数
2. 定义`struct arp_hdr`，`struct arp_ethip`，`struct arp_entry`和`struct elf_hdr`结构体
3. 初始化`arp_pack_hdr_ethip`函数的参数
4. 在函数体内，将`memmove`函数所需的参数复制到`pack_ethip_h`，`pack_ethip_l`，`pack_ethip_p`，`pack_arp_p`，`pack_arp_pa`和`pack_arp_ha`变量中。
5. 通过`do-while`循环，每次循环接收一个ARP条目
6. 在循环体内，提取出ARP协议头和硬件地址


```
/*
 * ARP cache entry
 */
struct arp_entry {
	struct addr	arp_pa;			/* protocol address */
	struct addr	arp_ha;			/* hardware address */
};

#ifndef __GNUC__
# pragma pack()
#endif

#define arp_pack_hdr_ethip(hdr, op, sha, spa, tha, tpa) do {	\
	struct arp_hdr *pack_arp_p = (struct arp_hdr *)(hdr);	\
	struct arp_ethip *pack_ethip_p = (struct arp_ethip *)	\
		((uint8_t *)(hdr) + ARP_HDR_LEN);		\
	pack_arp_p->ar_hrd = htons(ARP_HRD_ETH);		\
	pack_arp_p->ar_pro = htons(ARP_PRO_IP);			\
	pack_arp_p->ar_hln = ETH_ADDR_LEN;			\
	pack_arp_p->ar_pln = IP_ADDR_LEN;			\
	pack_arp_p->ar_op = htons(op);				\
	memmove(pack_ethip_p->ar_sha, &(sha), ETH_ADDR_LEN);	\
	memmove(pack_ethip_p->ar_spa, &(spa), IP_ADDR_LEN);	\
	memmove(pack_ethip_p->ar_tha, &(tha), ETH_ADDR_LEN);	\
	memmove(pack_ethip_p->ar_tpa, &(tpa), IP_ADDR_LEN);	\
} while (0)

```cpp

这段代码定义了一个名为 "arp_handle" 的结构体类型，称为 "arp_t"，该结构体包含一个指向 "arp_entry" 类型的指针和一个用于 "arp_handler" 类型的指针。

定义了一个名为 "arp_open" 的函数，该函数返回一个指向 "arp_handle" 类型的指针，该函数使用 "void" 类型的参数 "arg" 和 "const struct arp_entry *entry"。"

定义了一个名为 "arp_add" 的函数，该函数接受一个 "arp_handle" 类型的指针，一个 "const struct arp_entry *entry" 和一个 "void" 类型的参数 "arg"。函数返回 "arp_handle" 类型的指针，说明成功添加了一个 ARP 记录到链表中。

定义了一个名为 "arp_delete" 的函数，该函数接受一个 "arp_handle" 类型的指针，一个 "const struct arp_entry *entry" 和一个 "void" 类型的参数 "arg"。函数返回 "int" 类型的指针，说明成功删除了 ARP 记录到链表中的第一个。

定义了一个名为 "arp_get" 的函数，该函数接受一个 "arp_handle" 类型的指针和一个 "struct arp_entry *entry"。函数返回 "const struct arp_entry *entry" 类型的指针，说明成功读取 ARP 记录中的一个值。

定义了一个名为 "arp_loop" 的函数，该函数接受一个 "arp_handle" 类型的指针，一个 "arp_handler" 类型的参数 "callback" 和一个 "void" 类型的参数 "arg"。函数会在链表中循环遍历 "callback" 函数给出的 "arg" 类型的参数，然后将 "arg" 类型的参数与链表中的 "arp_entry" 类型的指针进行比较，如果相等，则执行相应的 "callback" 函数。

定义了一个名为 "arp_close" 的函数，该函数接受一个 "arp_handle" 类型的指针。函数返回 "int" 类型的指针，说明成功关闭链表。


```
typedef struct arp_handle arp_t;

typedef int (*arp_handler)(const struct arp_entry *entry, void *arg);

__BEGIN_DECLS
arp_t	*arp_open(void);
int	 arp_add(arp_t *arp, const struct arp_entry *entry);
int	 arp_delete(arp_t *arp, const struct arp_entry *entry);
int	 arp_get(arp_t *arp, struct arp_entry *entry);
int	 arp_loop(arp_t *arp, arp_handler callback, void *arg);
arp_t	*arp_close(arp_t *arp);
__END_DECLS

#endif /* DNET_ARP_H */

```cpp

# `libdnet-stripped/include/dnet/blob.h`

这段代码定义了一个名为 blob 的结构体，用于表示二进制数据块。该结构体包含五个成员变量：base、off、end、size 和 created。

* base 变量表示二进制数据块的起始地址，通过它可以访问到数据块的结尾地址。
* end 变量表示二进制数据块的结束地址，不包括该地址。
* size 变量表示分配给数据块的内存空间大小，即数据块的大小。
* created 变量表示二进制数据块是否是分配的内存，如果是，则从 0 开始计数。

这个结构体可以被用于在 C 语言中处理二进制数据，比如在内存中分配数据块、读取数据块内容等操作。


```
/*
 * blob.h
 *
 * Binary blob handling.
 *
 * Copyright (c) 2002 Dug Song <dugsong@monkey.org>
 *
 * $Id: blob.h 334 2002-04-05 03:06:44Z dugsong $
 */

#ifndef DNET_BLOB_H
#define DNET_BLOB_H

typedef struct blob {
	u_char		*base;		/* start of data */
	int		 off;		/* offset into data */
	int		 end;		/* end of data */
	int		 size;		/* size of allocation */
} blob_t;

```cpp

这段代码定义了一个名为 blob_t 的结构体，它包含了一些与文件 I/O 相关的操作。接下来分别解释每个函数的作用：

1. blob_new()：用于创建一个新的 blob_t 结构体，这个结构体包含了一些文件 I/O 相关的成员，例如文件指针、数据缓冲区和数据长度等。

2. blob_read()：从指定 blob_t 结构体中读取数据，并存储在由第二个输出参数 buf 指向的起始位置缓冲区中。返回从文件中读取的数据字节数 len。

3. blob_write()：将数据缓冲区中由第二个输入参数 buf 指向的连续数据写入到指定 blob_t 结构体中，从文件中读取的数据字节数 len。

4. blob_seek()：设置从文件中读取或写入数据时的目标位置。函数接受一个 int 类型的参数 off，表示目标位置从文件中开始偏移的偏移量，而第二个参数 whence 是一个 int 类型的参数，表示目标定位类型。当 whence_t. SEEK_SET 时，表示从文件开始偏移，并从文件中开始读取；当 whence_t. SEEK_CUR 是时，表示从文件中开始偏移，并从文件中继续读取。函数返回目标位置偏移量。

5. blob_skip()：从指定 blob_t 结构体中跳转到指定偏移量位置处，并跳过指定偏移量。

6. blob_rewind()：从指定 blob_t 结构体中跳转到指定偏移量位置处，并从头开始读取文件。

7. blob_offset(b)：返回 blob_t 结构体中数据缓冲区的起始位置。

8. blob_left(b)：返回 blob_t 结构体中数据缓冲区的结束位置。

9. blob_index(b)：返回 blob_t 结构体中数据缓冲区中指定偏移量位置处的数据元素指针。

10. blob_rindex(b)：返回 blob_t 结构体中数据缓冲区中指定偏移量位置处的数据元素指针，但只返回前连续的元素。


```
__BEGIN_DECLS
blob_t	*blob_new(void);

int	 blob_read(blob_t *b, void *buf, int len);
int	 blob_write(blob_t *b, const void *buf, int len);

int	 blob_seek(blob_t *b, int off, int whence);
#define  blob_skip(b, l)	blob_seek(b, l, SEEK_CUR)
#define  blob_rewind(b)		blob_seek(b, 0, SEEK_SET)

#define	 blob_offset(b)		((b)->off)
#define	 blob_left(b)		((b)->end - (b)->off)

int	 blob_index(blob_t *b, const void *buf, int len);
int	 blob_rindex(blob_t *b, const void *buf, int len);

```cpp

这两函数是Blob库中的函数，作用分别是Blob的压缩和反压缩。

`blob_pack`函数接受一个Blob对象`b`，一个格式字符串`fmt`和一个`...`参数`va_start`。它的作用是将`fmt`中的格式字符串解析为模板参数，然后将解析得到的结构体或指针赋值给`b`，使得`b`可以按照指定的格式进行输出或输入。具体实现中，`fmt`字符串中的每一个格式字符都是通过 Blob 中的 `%` 符号来对应的，比如 `%d` 对应的是 `int` 类型。

`blob_unpack`函数与`blob_pack`函数正好相反，它的作用是将一个格式字符串中的模板参数值解包赋值给Blob对象`b`中的结构体或指针。

`blob_insert`函数接受一个Blob对象`b`和一个字节缓冲区`buf`，它的作用是在`b`中插入一个新的Blob单元，插入的字节数组是`len`个字节。具体实现中，`buf`字节缓冲区中的字节数组是要插入到`b`的起始位置，`len`表示插入的字节数。

`blob_delete`函数与`blob_insert`函数正好相反，它的作用是从`b`中删除一个指定字节缓冲区`buf`，并返回从`buf`中删除的字节数。

`blob_print`函数输出一个Blob对象`b`中的内容，通过一个格式字符串模板参数`fmt`，并使用指定的`style`参数指定输出的格式风格。

`blob_free`函数返回一个Blob对象`b`所对应的内存空间，这个内存空间可以再次被Blob库中的其他函数使用。

`blob_register_alloc`函数注册Blob库，以便在使用时自动调用相应的函数。它的第一个参数是一个要注册的Blob类型，第二个参数是一个模板Blob库函数指针，第三个参数是一个Blob库函数指针，用于将指定的Blob库函数注册到Blob库中。


```
int	 blob_pack(blob_t *b, const char *fmt, ...);
int	 blob_unpack(blob_t *b, const char *fmt, ...);

int	 blob_insert(blob_t *b, const void *buf, int len);
int	 blob_delete(blob_t *b, void *buf, int len);

int	 blob_print(blob_t *b, char *style, int len);

blob_t	*blob_free(blob_t *b);

int	 blob_register_alloc(size_t size, void *(*bmalloc)(size_t),
	    void (*bfree)(void *), void *(*brealloc)(void *, size_t));
#ifdef va_start
typedef int (*blob_fmt_cb)(int pack, int len, blob_t *b, va_list *arg);

```cpp

这段代码是一个名为“blob_register_pack”的函数，它的作用是注册一种名为“Blob”的数据结构。这个函数接受两个参数，第一个参数是一个字符类型的变量“c”，第二个参数是一个名为“Blobfmt_cb”的函数指针，这个函数指针 later指向了一个以“fmt_”为后缀的函数，这个函数接受一个可变参数“fmt_”。

具体来说，这个函数的主要作用是接受一个“Blob”数据结构以及一个“fmt_”函数指针，并将它们与传入的“c”和“fmt_”参数绑定在一起，以便在以后的调用中正确使用它们。

这个函数与DNET的Blob似乎没有直接关系，它可能是在另一个命名空间中定义的函数。


```
int	 blob_register_pack(char c, blob_fmt_cb fmt_cb);
#endif
__END_DECLS

#endif /* DNET_BLOB_H */

```cpp

# `libdnet-stripped/include/dnet/eth.h`

这段代码是一个头文件，名为“eth.h”，定义了一个名为“Ethernet”的类，包含了Ethernet的一些基本成员变量和函数。

具体来说，该头文件定义了以下成员变量和函数：

- Ethernet类有一个名为“mac_ address”的成员变量，类型为“unsigned char”表示无符号字节数组，用于存储数据帧的发送方MAC地址。
- Ethernet类有一个名为“mac_ len”的成员变量，类型为“unsigned int”表示无符号整数，用于存储数据帧的长度。
- Ethernet类有一个名为“ethereum_ address”的成员变量，类型为“unsigned char”表示无符号字节数组，用于存储以太坊网络中的收发方地址。
- Ethernet类有一个名为“ipv4_ address”的成员变量，类型为“unsigned long”表示无符号长整数，用于存储数据包的源IP地址和目的IP地址。
- Ethernet类有一个名为“eth_ protocol”的成员变量，类型为“unsigned int”表示无符号整数，用于存储数据包的协议类型。
- Ethernet类有一个名为“rx_ packets”的成员变量，类型为“unsigned int”表示无符号整数，用于存储接收到的数据包数量。
- Ethernet类有一个名为“tx_ packets”的成员变量，类型为“unsigned int”表示无符号整数，用于存储发送的数据包数量。

该头文件还定义了一个名为“Ethernet_ craper”的函数，用于将一个IP数据包转换为以太坊地址，并返回其“to_hash”值。


```
/*
 * eth.h
 *
 * Ethernet.
 *
 * Copyright (c) 2000 Dug Song <dugsong@monkey.org>
 *
 * $Id: eth.h 547 2005-01-25 21:30:40Z dugsong $
 */

#ifndef DNET_ETH_H
#define DNET_ETH_H

#define ETH_ADDR_LEN	6
#define ETH_ADDR_BITS	48
```cpp

这段代码定义了几个常量，用于定义以太网协议的帧结构和长度。

#define ETH_TYPE_LEN		2	/* 定义以太网协议类型长度为2 */
#define ETH_CRC_LEN		4	/* 定义以太网协议类型中CRC长度为4 */
#define ETH_HDR_LEN		14	/* 定义以太网协议头部长度为14 */

#define ETH_LEN_MIN		64	/* 定义最小帧长度的最小值 */
#define ETH_LEN_MAX		1518	/* 定义最大帧长度的最大值 */

#define ETH_MTU		(ETH_LEN_MAX - ETH_HDR_LEN - ETH_CRC_LEN)	/* 定义MTU(帧最大长度) */
#define ETH_MIN		(ETH_LEN_MIN - ETH_HDR_LEN - ETH_CRC_LEN)	/* 定义MIN(帧最小长度) */

定义了一个名为“eth_addr”的结构体，用于表示一个以太网地址。

struct eth_hdr {
	eth_addr_t	eth_dst;	/*  destination address */
	eth_addr_t	eth_src;	/* source address */
	uint16_t	eth_type;	/*  payload type */
};

定义了一个名为“struct eth_hdr”的结构体，用于表示以太网协议的头部，其中包含目的MAC地址和源MAC地址，以及设置为正确的数据帧类型。


```
#define ETH_TYPE_LEN	2
#define ETH_CRC_LEN	4
#define ETH_HDR_LEN	14

#define ETH_LEN_MIN	64		/* minimum frame length with CRC */
#define ETH_LEN_MAX	1518		/* maximum frame length with CRC */

#define ETH_MTU		(ETH_LEN_MAX - ETH_HDR_LEN - ETH_CRC_LEN)
#define ETH_MIN		(ETH_LEN_MIN - ETH_HDR_LEN - ETH_CRC_LEN)

typedef struct eth_addr {
	uint8_t		data[ETH_ADDR_LEN];
} eth_addr_t;

struct eth_hdr {
	eth_addr_t	eth_dst;	/* destination address */
	eth_addr_t	eth_src;	/* source address */
	uint16_t	eth_type;	/* payload type */
};

```cpp

这段代码定义了一系列Ethernet数据包类型的常量，用于标识网络层协议中的数据包类型。以下是每个常量的解释：

- ETH_TYPE_PUP：室分(point-to-point)，用于点对点(P2P)网络中的数据包类型。
- ETH_TYPE_IP：互联网协议(IP)，用于在IP网络中传输的数据包类型。
- ETH_TYPE_ARP：地址解析协议(ARP)，用于将IP地址映射为物理地址(MAC)地址。
- ETH_TYPE_REVARP：反向地址解析协议(RARP)，用于从物理地址(MAC)地址映射到IP地址。
- ETH_TYPE_8021Q:IEEE 802.1Q VLAN(虚拟局域网)标记协议，用于在局域网中传输的数据包类型。
- ETH_TYPE_IPV6:Internet协议版本6(IPv6)，用于在IP网络上传输的数据包类型。
- ETH_TYPE_MPLS：多协议标签(MPLS)，用于在网络中传输的数据包类型。
- ETH_TYPE_MPLS_MCAST:MPLS多播标记(MPLS-MCAST)，用于在MPLS网络中传输的数据包类型。
- ETH_TYPE_PPPOEDISC:PPP Over Ethernet Discovery Stage(PPPoE)，用于在PPPoE协议中传输的数据包类型。
- ETH_TYPE_PPPOE:PPP Over Ethernet Session Stage(PPPoE)，用于在PPPoE协议中传输的数据包类型。
- ETH_TYPE_LOOPBACK：接口回环(loopback)，用于在本地测试网络接口的数据包类型。


```
/*
 * Ethernet payload types - http://standards.ieee.org/regauth/ethertype
 */
#define ETH_TYPE_PUP	0x0200		/* PUP protocol */
#define ETH_TYPE_IP	0x0800		/* IP protocol */
#define ETH_TYPE_ARP	0x0806		/* address resolution protocol */
#define ETH_TYPE_REVARP	0x8035		/* reverse addr resolution protocol */
#define ETH_TYPE_8021Q	0x8100		/* IEEE 802.1Q VLAN tagging */
#define ETH_TYPE_IPV6	0x86DD		/* IPv6 protocol */
#define ETH_TYPE_MPLS	0x8847		/* MPLS */
#define ETH_TYPE_MPLS_MCAST	0x8848	/* MPLS Multicast */
#define ETH_TYPE_PPPOEDISC	0x8863	/* PPP Over Ethernet Discovery Stage */
#define ETH_TYPE_PPPOE	0x8864		/* PPP Over Ethernet Session Stage */
#define ETH_TYPE_LOOPBACK	0x9000	/* used to test interfaces */

```cpp

这段代码定义了几个宏，以及一个名为 `eth_pack_hdr` 的函数。接下来我会逐步解释每个部分的含义。

宏定义：
```
#define ETH_IS_MULTICAST(ea)	(*(ea) & 0x01) /* is address mcast/bcast? */
```cpp
这个宏定义了一个名为 `ETH_IS_MULTICAST` 的函数，它的参数 `ea` 是一个 `int` 类型的整数。这个函数的作用是判断给定的 `int` 是否是广播地址。如果是广播地址，函数返回 `true`，否则返回 `false`。

```
#define ETH_ADDR_BROADCAST	"\xff\xff\xff\xff\xff\xff"
```cpp
这个宏定义了一个名为 `ETH_ADDR_BROADCAST` 的字符串，它的值是 "0xff\xff\xff\xff\xff\xff"。这个字符串表示一个广播地址，由六个 `0` 字节的无符号整数组成。

```
#define eth_pack_hdr(h, dst, src, type) do {				\
											\
												\
													\
														\
														\
															\
															\
															\
															\
																\
																\
																\
																\
																\
																\
																\
																\
																\
																	\
																	\
																\
																\
																	\
																	\
																		\
																		\
																	\
																	\
																		\
																		\
																		\
																		\
																		\
																			\
																			\
																			\
																				\
																			\
																				\
																				\
																					\
																					\
																					\
																					\
```cpp


```
#define ETH_IS_MULTICAST(ea)	(*(ea) & 0x01) /* is address mcast/bcast? */

#define ETH_ADDR_BROADCAST	"\xff\xff\xff\xff\xff\xff"

#define eth_pack_hdr(h, dst, src, type) do {			\
	struct eth_hdr *eth_pack_p = (struct eth_hdr *)(h);	\
	memmove(&eth_pack_p->eth_dst, &(dst), ETH_ADDR_LEN);	\
	memmove(&eth_pack_p->eth_src, &(src), ETH_ADDR_LEN);	\
	eth_pack_p->eth_type = htons(type);			\
} while (0)

typedef struct eth_handle eth_t;

__BEGIN_DECLS
eth_t	*eth_open(const char *device);
```cpp

这是一个网络协议栈中的代码，主要实现了以太网（Ethernet）协议的功能。以下是代码的主要作用和功能：

1. `int eth_get(eth_t *e, eth_addr_t *ea);`：返回一个整数类型的`eth_t`指针，表示输入的以太网类型。同时返回一个指向`eth_addr_t`类型的`ea`指针，表示输入的以太地址。
2. `int eth_set(eth_t *e, const eth_addr_t *ea);`：设置输入的以太网类型。输入的以太地址将被转换为`eth_addr_t`结构，并将其赋值给输入的`e`指针。
3. `ssize_t eth_send(eth_t *e, const void *buf, size_t len);`：将输入的`eth_t`指针`e`与一个`const void *`类型的缓冲区`buf`中的内容进行发送。返回传输数据的字节数，单位是字节（`ssize_t`）。
4. `eth_t *eth_close(eth_t *e);`：关闭输入的以太网类型。该函数没有实际实现，只是作为一个虚函数，在需要时被调用。如果没有调用它，函数将不会执行任何操作。
5. `int eth_get_pcap_devname(const char *ifname, char *pcapdev, int pcapdevlen);`：从给定的`const char *`类型的接口名称（`ifname`）中获取相应的硬件设备名称（`pcapdev`）。并将设备名称存储在`pcapdev`指向的内存中，长度为`pcapdevlen`。函数返回设备名称的起始地址。
6. `char *eth_ntop(const eth_addr_t *eth, char *dst, size_t len);`：将输入的以太地址（`eth_addr_t`）转换为字符串，并将其存储在`dst`指向的内存中。输入的数据长度为`len`。函数返回转换后的字符串。
7. `int eth_pton(const char *src, eth_addr_t *dst);`：将输入的源地址（`const char *`）转换为以太网地址，并将其存储在`dst`指向的内存中。函数返回源地址转换后的地址。
8. `char *eth_ntoa(const eth_addr_t *eth);`：将输入的以太地址（`eth_addr_t`）转换为字符串，并将其存储在`strdup`函数的返回位置。函数返回转换后的字符串。

除了以上主要函数外，还有一些辅助函数，例如：

9. `void *eth_举伦(const eth_addr_t *eth);`：对给定的以太地址进行链路层地址转换（LINK层地址转换）。返回转换后的指针。
10. `int eth_v6_pton(const char *ipv6_str, eth_addr_t *dst);`：将输入的六段地址（IPv6地址）转换为以太网地址，并将其存储在`dst`指向的内存中。函数返回源地址转换后的地址。


```
int	 eth_get(eth_t *e, eth_addr_t *ea);
int	 eth_set(eth_t *e, const eth_addr_t *ea);
ssize_t	 eth_send(eth_t *e, const void *buf, size_t len);
eth_t	*eth_close(eth_t *e);

int	 eth_get_pcap_devname(const char *ifname, char *pcapdev, int pcapdevlen);

char	*eth_ntop(const eth_addr_t *eth, char *dst, size_t len);
int	 eth_pton(const char *src, eth_addr_t *dst);
char	*eth_ntoa(const eth_addr_t *eth);
#define	 eth_aton eth_pton
__END_DECLS

#endif /* DNET_ETH_H */

```cpp

# `libdnet-stripped/include/dnet/fw.h`

这段代码定义了一个名为fw_rule的结构体，用于表示网络防火墙规则。该结构体包含以下字段：

1. fw_device：接口名称，最多支持4K个字符。
2. fw_op：操作码，可以是0-7，表示防火墙允许或拒绝连接。
3. fw_dir：方向，0表示入方向，1表示出方向。
4. fw_proto：IP协议，可以是0-49。
5. fw_src：源地址，范围在0-65535之间。
6. fw_dst：目标地址，范围在0-65535之间。
7. fw_sport：源端口范围，范围在0-65535之间。
8. fw_dport：目标端口范围，范围在0-65535之间。
9. fw_抗议：是否允许这个连接，可以是0或1。

这个结构体可以定义一个防火墙规则，通过修改fw_rule的值来设置防火墙的规则。


```
/*
 * fw.h
 *
 * Network firewalling operations.
 *
 * Copyright (c) 2001 Dug Song <dugsong@monkey.org>
 *
 * $Id: fw.h 394 2002-12-14 04:02:36Z dugsong $
 */

#ifndef DNET_FW_H
#define DNET_FW_H

struct fw_rule {
	char		fw_device[INTF_NAME_LEN]; /* interface name */
	uint8_t		fw_op;			  /* operation */
	uint8_t		fw_dir;			  /* direction */
	uint8_t		fw_proto;		  /* IP protocol */
	struct addr	fw_src;			  /* src address / net */
	struct addr	fw_dst;			  /* dst address / net */
	uint16_t	fw_sport[2];		  /* range / ICMP type */
	uint16_t	fw_dport[2];		  /* range / ICMP code */
};

```cpp

这段代码定义了一系列伪命名(Macro Definition)，用于定义输入输出操作的功能。

FW_OP_ALLOW 和 FW_OP_BLOCK 是宏定义，定义了输入输出操作允许和禁止的值，其值分别为 1 和 2。

FW_DIR_IN 和 FW_DIR_OUT 是宏定义，定义了输入输出方向分别为 IN 和 OUT。

fw_pack_rule 是一个函数，是一个用户定义的函数，用于将输入数据包装成一个符合规范的输入数据格式，其中输入数据包括输入的设备、输入的操作系统调用(op)、输入的输入输出方向(dir)、输入输出端口号(p, s, d, sp1, sp2)和输入输出数据长度(dp1, dp2)。该函数接受六个参数，分别为 rule、dev、op、dir、p、s、d、sp1 和 sp2，其中 rule 是已经定义好的一个结构体，包含输入数据的所有字段。函数首先根据输入的 device 和 dev 判断是否支持该设备，然后根据输入的 op 和 dir 判断是否支持该操作和方向，接着根据输入的 p 和 s 判断输入端口是否为该设备输出端口，然后根据输入的 d 和 sp1 判断是否允许数据从该设备输入到其它设备，最后根据输入的 op 和 dp1、dp2 判断是否允许数据从设备输出到该设备。最后，函数将输入数据拷贝到 rule 的 fw_device 字段中，以便后续使用。


```
#define FW_OP_ALLOW	1
#define FW_OP_BLOCK	2

#define FW_DIR_IN	1
#define FW_DIR_OUT	2

#define fw_pack_rule(rule, dev, op, dir, p, s, d, sp1, sp2, dp1, dp2)	\
do {									\
	strlcpy((rule)->fw_device, dev, sizeof((rule)->fw_device));	\
	(rule)->fw_op = op; (rule)->fw_dir = dir;			\
	(rule)->fw_proto = p;						\
	memmove(&(rule)->fw_src, &(s), sizeof((rule)->fw_src));		\
	memmove(&(rule)->fw_dst, &(d), sizeof((rule)->fw_dst));		\
	(rule)->fw_sport[0] = sp1; (rule)->fw_sport[1] = sp2;		\
	(rule)->fw_dport[0] = dp1; (rule)->fw_dport[1] = dp2;		\
} while (0)

```cpp

这段代码定义了一个名为fw_t的结构体，用于表示网络防火墙中的规则。

定义了一个名为fw_handler的函数指针类型，表示一个用于处理网络防火墙规则的函数。

定义了三个函数fw_open,fw_add,fw_delete和fw_loop，用于打开、添加、删除和遍历网络防火墙规则。

函数fw_open接收一个void类型的参数，表示在内存中为该结构体分配内存空间。

函数fw_add接收一个fw_t类型的参数和一个const struct fw_rule类型的参数，表示向网络防火墙规则集中添加一个新规则。

函数fw_delete接收一个fw_t类型的参数和一个const struct fw_rule类型的参数，表示从网络防火墙规则集中删除一个规则。

函数fw_loop接收一个fw_t类型的参数和一个fw_handler类型的参数和一个void类型的参数，表示在给定规则集上循环处理每个规则，并将回调函数作为参数传递给下一个规则。

函数fw_close接收一个fw_t类型的参数，表示在内存中释放网络防火墙规则集所分配的内存空间。

该代码的作用是定义了一个用于网络防火墙的规则集对象，其中包括打开、添加、删除和遍历规则的函数，以及定义了一个规则集的地址作为参数的函数fw_loop。这个代码定义的结构体可以在其他部分中被用来。


```
typedef struct fw_handle fw_t;

typedef int (*fw_handler)(const struct fw_rule *rule, void *arg);

__BEGIN_DECLS
fw_t	*fw_open(void);
int	 fw_add(fw_t *f, const struct fw_rule *rule);
int	 fw_delete(fw_t *f, const struct fw_rule *rule);
int	 fw_loop(fw_t *f, fw_handler callback, void *arg);
fw_t	*fw_close(fw_t *f);
__END_DECLS

#endif /* DNET_FW_H */

```cpp

# `libdnet-stripped/include/dnet/icmp.h`

这段代码定义了一个名为"icmp.h"的 header 文件，它是 Internet 控制消息协议(ICMP)的规范。这个文件定义了一些全局常量和语义，以及包含在 ICMP 消息中的字段。

具体来说，这个文件定义了 ICMP 消息头部的长度为 ICMP_HDR_LEN 字段，定义了一个名为 "icmp_header" 的函数，用于在 ICMP 消息中传输 header 部分，以及一个名为 "icmp_message" 的函数，用于处理 ICMP 消息的完整内容。

这个文件是用于在 Linux 系统中使用 ICMP 协议来实现网络通信的，例如在发送或接收 ICMP 消息时进行处理。


```
/*
 * icmp.h
 *
 * Internet Control Message Protocol.
 * RFC 792, 950, 1256, 1393, 1475, 2002, 2521
 *
 * Copyright (c) 2000 Dug Song <dugsong@monkey.org>
 *
 * $Id: icmp.h 416 2003-03-16 17:39:18Z dugsong $
 */

#ifndef DNET_ICMP_H
#define DNET_ICMP_H

#define ICMP_HDR_LEN	4	/* base ICMP header length */
```cpp

这段代码定义了一个名为"ICMP_LEN_MIN"的宏，表示最小ICMP消息的大小，加上头部。

接下来定义了一个名为"__attribute__"的保留修饰符，如果这个修饰符在头文件中使用，则可以编译时检查其存在性并进行类型检查。

然后定义了一个名为"__attribute__(x)"的保留修饰符，表示可以编译时检查其存在性，并进行类型检查。这里使用了一个参数"x"，表示该修饰符后面的定义。

接下来定义了一个名为"struct icmp_hdr"的结构体，其中包含ICMP消息头中的四个字段：icmp_type、icmp_code、icmp_cksum。

然后定义了一个名为"icmp_len_min"的宏，表示最小ICMP消息的大小，加上头部。

接下来定义了一个名为"__attribute__"的保留修饰符，如果这个修饰符在头文件中使用，则可以编译时检查其存在性并进行类型检查。

然后定义了一个名为"__attribute__(x)"的保留修饰符，表示可以编译时检查其存在性，并进行类型检查。这里使用了一个参数"x"，表示该修饰符后面的定义。

接下来定义了一个名为"icmp_hdr"的结构体，其中包含ICMP消息头中的四个字段：icmp_type、icmp_code、icmp_dest、icmp_ts。

然后定义了一个名为"icmp_header_size"的函数，该函数的参数为结构体指针，表示返回该结构体的大小，单位为字节。

接下来定义了一个名为"compare_icmp_hdr"的函数，该函数比较两个ICMP消息头的大小，返回值为其是否相等，函数的第一个参数为两个ICMP消息头，第二个参数为结构体指针。

最后定义了一个名为"main"的函数，该函数接受一个ICMP消息头作为参数，然后比较该消息头与之前定义的最小ICMP消息大小，如果它们相等，则打印一条消息，否则打印一条消息并输出结构体指针。


```
#define ICMP_LEN_MIN	8	/* minimum ICMP message size, with header */

#ifndef __GNUC__
#ifndef __attribute__
# define __attribute__(x)
#endif
# pragma pack(1)
#endif

/*
 * ICMP header
 */
struct icmp_hdr {
	uint8_t		icmp_type;	/* type of message, see below */
	uint8_t		icmp_code;	/* type sub code */
	uint16_t	icmp_cksum;	/* ones complement cksum of struct */
};

```cpp

这段代码定义了一系列ICMP（Internet Control Message Protocol，互联网报文协议）参数的宏，用于在代码中使用。其中包括：ICMP类型（icmp_type）和代码（icmp_code）。这些参数用于在ICMP协议中进行通信，例如回应请求、路由问题等。以下是定义的各个参数：

1. ICMP_CODE_NONE：表示没有代码的ICMP类型。
2. ICMP_ECHOREPLY：表示echo reply的ICMP类型。
3. ICMP_UNREACH：表示无法到达的目标，代码为3。其中，代码值包括：
  - ICMP_UNREACH_NET：表示目标不可达的网络问题；
  - ICMP_UNREACH_HOST：表示目标不可达的主机问题；
  - ICMP_UNREACH_PROTO：表示目标不可达的协议问题；
  - ICMP_UNREACH_PORT：表示目标不可达的端口问题；
  - ICMP_UNREACH_NEEDFRAG：表示IP分片导致的丢弃问题；
  - ICMP_UNREACH_SRCFAIL：表示源点无法到达的问题；
  - ICMP_UNREACH_HOST_UNKNOWN：表示主机不可知的问题。
4. ICMP_UNREACH_PROTOCOL：表示无法到达的协议问题，代码值为2。
5. ICMP_UNREACH_PORT：表示无法到达的端口问题，代码值为3。
6. ICMP_UNREACH_NEEDFRAG：表示IP分片导致的丢弃问题，代码值为4。
7. ICMP_UNREACH_SRCFAIL：表示源点无法到达的问题，代码值为5。
8. ICMP_UNREACH：表示无法到达的目标，代码值为6。
9. ICMP_UNREACH_NET：表示目标不可达的网络问题，代码值为7。
10. ICMP_UNREACH_HOST：表示目标不可达的主机问题，代码值为8。


```
/*
 * Types (icmp_type) and codes (icmp_code) -
 * http://www.iana.org/assignments/icmp-parameters
 */
#define		ICMP_CODE_NONE		0	/* for types without codes */
#define	ICMP_ECHOREPLY		0		/* echo reply */
#define	ICMP_UNREACH		3		/* dest unreachable, codes: */
#define		ICMP_UNREACH_NET		0	/* bad net */
#define		ICMP_UNREACH_HOST		1	/* bad host */
#define		ICMP_UNREACH_PROTO		2	/* bad protocol */
#define		ICMP_UNREACH_PORT		3	/* bad port */
#define		ICMP_UNREACH_NEEDFRAG		4	/* IP_DF caused drop */
#define		ICMP_UNREACH_SRCFAIL		5	/* src route failed */
#define		ICMP_UNREACH_NET_UNKNOWN	6	/* unknown net */
#define		ICMP_UNREACH_HOST_UNKNOWN	7	/* unknown host */
```cpp

这段代码定义了一系列ICMP（Internet Control Message Protocol，互联网报文协议）报文头，用于在网络中报告和处理各种错误情况。以下是这些定义的作用：

1. #define：定义了一系列预定义的ICMP报文头。例如，当发生丢包、延迟或者网络问题等网络问题时，这些报告头可以在网络设备（如路由器、交换机等）和主机之间传递，以便进行调试和诊断。

2. ICMP_UNREACH_ISOLATED：8，此值为8，表示源主机被 isolation（独立）了。当源主机被isolation时，ICMP在此位置发送数据包，以便报告此问题。

3. ICMP_UNREACH_NET_PROHIB：此报告头用于加密设备的开发人员。当发生网络问题时候，发送数据包报告此问题。

4. ICMP_UNREACH_HOST_PROHIB：此报告头用于报告主机被isolation的情况。当主机被isolation时，ICMP在此位置发送数据包，以便报告此问题。

5. ICMP_UNREACH_TOSNET：此报告头用于报告到网络的ICMP报文因为"网络问题"（如丢包、延迟等）而不能被正确发送的情况。

6. ICMP_UNREACH_TOSHOST：此报告头用于报告到主机的ICMP报文因为"主机问题"（如被isolation、超时等）而不能被正确发送的情况。

7. ICMP_UNREACH_FILTER_PROHIB：此报告头用于报告内容过滤被禁止的情况。当内容过滤被禁止时，ICMP在此位置发送数据包，以便报告此问题。

8. ICMP_UNREACH_HOST_PRECEDENCE：此报告头用于报告主机优先级出错的情况。当主机优先级出错时，ICMP在此位置发送数据包，以便报告此问题。

9. ICMP_UNREACH_PRECEDENCE_CUTOFF：此报告头用于报告优先级切片的截止时间。当优先级切片超过此时间时，ICMP在此位置发送数据包，以便报告此问题。

10. ICMP_SRCQUENCH：此报告头表示数据包丢失，需要重新发送。

11. ICMP_REDIRECT：此报告头表示要执行的下一步操作，如通过更改路由或使用 shorter route。

12. ICMP_REDIRECT_NET：此报告头表示向目标网络执行的操作。

13. ICMP_REDIRECT_HOST：此报告头表示向目标主机执行的操作。

14. ICMP_REDIRECT_TOSNET：此报告头表示目标为网络，但采取的操作（如更改路由）。

15. ICMP_REDIRECT_TOSHOST：此报告头表示目标为主机，但采取的操作（如更改路由）。

16. ICMP_ALTHOSTADDR：此报告头表示设置为备用主机地址的ICMP报文。当主服务器不可用时，发送此数据包以通知备用主机接管服务。


```
#define		ICMP_UNREACH_ISOLATED		8	/* src host isolated */
#define		ICMP_UNREACH_NET_PROHIB		9	/* for crypto devs */
#define		ICMP_UNREACH_HOST_PROHIB	10	/* ditto */
#define		ICMP_UNREACH_TOSNET		11	/* bad tos for net */
#define		ICMP_UNREACH_TOSHOST		12	/* bad tos for host */
#define		ICMP_UNREACH_FILTER_PROHIB	13	/* prohibited access */
#define		ICMP_UNREACH_HOST_PRECEDENCE	14	/* precedence error */
#define		ICMP_UNREACH_PRECEDENCE_CUTOFF	15	/* precedence cutoff */
#define	ICMP_SRCQUENCH		4		/* packet lost, slow down */
#define	ICMP_REDIRECT		5		/* shorter route, codes: */
#define		ICMP_REDIRECT_NET		0	/* for network */
#define		ICMP_REDIRECT_HOST		1	/* for host */
#define		ICMP_REDIRECT_TOSNET		2	/* for tos and net */
#define		ICMP_REDIRECT_TOSHOST		3	/* for tos and host */
#define	ICMP_ALTHOSTADDR	6		/* alternate host address */
```cpp

这段代码是一个 C 语言代码，定义了几个用于 Internet Control Message Protocol（ICMP）的常量。

ICMP-ECHO：这是一个用于测试网络中电子邮件服务器（如 SMTP）是否正常工作的命令，回送客户端发送的邮件到客户端。返回客户端发送给服务器的 ICMP 数据包的 ECHO 消息（包含一个 "Hello" 消息和客户端发送给服务器的数据）。

ICMP-RTRADVERT：这是用于向路由器告知其支持哪些 ICMP 类型，以及这些类型的参数。这个数据包可以向路由器发送一系列 ICMP 数据包，告知其支持的 ICMP 类型和参数，从而使路由器能够更好地处理它们。

ICMP-RTRADVERT_NORMAL：这是正常情况下发送的 ICMP 类型，包含一个包含一系列参数的 ICMP 数据包。

ICMP-RTRADVERT_NOROUTE_COMMON：这是当路由器接收到这个数据包时，应该采取的行动。这个数据包测试的是路由器是否支持指定的路由。

ICMP-RTRSOLICIT：这是用于向路由器发送请求，以获取其支持哪些 ICMP 类型。这个数据包包含一个包含一系列参数的 ICMP 数据包，用于请求路由器返回支持的数据。

ICMP-TIMEXCEED：这是测试网络延迟的命令，可以告诉服务器某个 ICMP 数据包在传输过程中超时了。

ICMP-TIMEXCEED_INTRANS：这是另一个测试网络延迟的命令，可以告诉服务器某个 ICMP 数据包在传输过程中超时了，但这个超时并不会导致数据包丢失。

ICMP-PARAMPROB：这是测试 ICMP 数据包的错误报告的命令，包含一个包含一系列参数的 ICMP 数据包，用于报告错误。

ICMP-PARAMPROB_ERRATPTR：这是当 ICMP 错误报告中的错误 PTR 存在时发送的 ICMP 数据包。

ICMP-PARAMPROB_OPTABSENT：这是当 ICMP 错误报告中的错误 PTR 不存在时发送的 ICMP 数据包。

ICMP-TSTAMP：这是用于在 ICMP 数据包中包含日期和时间戳的命令。

ICMP-TSTAMPREPLY：这是用于在 ICMP 数据包中包含日期和时间戳的回复命令。

ICMP-INFO：这是用于在 ICMP 数据包中包含控制信息的命令。


```
#define	ICMP_ECHO		8		/* echo service */
#define	ICMP_RTRADVERT		9		/* router advertise, codes: */
#define		ICMP_RTRADVERT_NORMAL		0	/* normal */
#define		ICMP_RTRADVERT_NOROUTE_COMMON 16	/* selective routing */
#define	ICMP_RTRSOLICIT		10		/* router solicitation */
#define	ICMP_TIMEXCEED		11		/* time exceeded, code: */
#define		ICMP_TIMEXCEED_INTRANS		0	/* ttl==0 in transit */
#define		ICMP_TIMEXCEED_REASS		1	/* ttl==0 in reass */
#define	ICMP_PARAMPROB		12		/* ip header bad */
#define		ICMP_PARAMPROB_ERRATPTR		0	/* req. opt. absent */
#define		ICMP_PARAMPROB_OPTABSENT	1	/* req. opt. absent */
#define		ICMP_PARAMPROB_LENGTH		2	/* bad length */
#define	ICMP_TSTAMP		13		/* timestamp request */
#define	ICMP_TSTAMPREPLY	14		/* timestamp reply */
#define	ICMP_INFO		15		/* information request */
```cpp

这段代码定义了一系列ICMP（Internet Control Message Protocol）错误码。它们用于在IPv4（互联网协议第四版）和IPv6（互联网协议第六版）网络中传输错误信息。以下是每个错误码的简要解释：

1. ICMP_INFOREPLY：表示信息回复消息。通常用于向客户端发送成功应答。
2. ICMP_MASK：表示地址掩码请求。请求用于请求目标主机提供特定地址的掩码。
3. ICMP_MASKREPLY：表示地址掩码回复。回复用于向客户端发送目标主机提供的地址掩码。
4. ICMP_TRACEROUTE：表示跟踪路由器。用于报告到达目标的路由器的数量。
5. ICMP_DATACONVERR：表示数据转换错误。通常用于报告失败的媒体类型转换操作。
6. ICMP_MOBILE_REDIRECT：表示移动主机重定向。用于通知客户端它需要采取的其他措施，例如重新连接到远程主机或重新启动代理。
7. ICMP_IPV6_WHEREAREYOU：表示IPv6的“你在哪里”。通常用于向客户端发送IPv6的“定位”信息。
8. ICMP_IPV6_IAMHERE：表示IPv6的“我在此处”。通常用于向客户端发送IPv6的“身份”信息。
9. ICMP_MOBILE_REG：表示移动注册。用于通知客户端它需要注册到相应的情域。
10. ICMP_MOBILE_REGREPLY：表示移动注册的回复。用于向客户端发送注册确认。
11. ICMP_DNS：表示域名解析。用于向客户端提供DNS（域名系统）服务器的响应。
12. ICMP_DNSREPLY：表示域名解析的回复。用于向客户端提供DNS服务器的响应。
13. ICMP_SKIP：表示跳过。用于通知客户端它需要跳过当前的分发。
14. ICMP_PHOTURIS：表示照片无关的“非活动”。通常用于报告发生的网络问题与照片无关。
15. ICMP_PHOTURIS_UNKNOWN_INDEX：表示照片无关的“未知活动”。通常用于报告发生的网络问题，但照片解决方案可以解决。


```
#define	ICMP_INFOREPLY		16		/* information reply */
#define	ICMP_MASK		17		/* address mask request */
#define	ICMP_MASKREPLY		18		/* address mask reply */
#define ICMP_TRACEROUTE		30		/* traceroute */
#define ICMP_DATACONVERR	31		/* data conversion error */
#define ICMP_MOBILE_REDIRECT	32		/* mobile host redirect */
#define ICMP_IPV6_WHEREAREYOU	33		/* IPv6 where-are-you */
#define ICMP_IPV6_IAMHERE	34		/* IPv6 i-am-here */
#define ICMP_MOBILE_REG		35		/* mobile registration req */
#define ICMP_MOBILE_REGREPLY	36		/* mobile registration reply */
#define ICMP_DNS		37		/* domain name request */
#define ICMP_DNSREPLY		38		/* domain name reply */
#define ICMP_SKIP		39		/* SKIP */
#define ICMP_PHOTURIS		40		/* Photuris */
#define		ICMP_PHOTURIS_UNKNOWN_INDEX	0	/* unknown sec index */
```cpp

这段代码定义了一系列与ICMP Photuris有关的错误代码。它们用于在ICMP协议中传输照片身份验证和压缩失败的报文。以下是每个错误代码的简要解释：

1. ICMP_PHOTURIS_AUTH_FAILED：表示身份验证失败。例如，尝试登录到一个不存在的服务器时会收到这个错误。
2. ICMP_PHOTURIS_DECOMPRESS_FAILED：表示解压缩失败。这可能是由于网络问题或服务器不支持JPEG或PNG等压缩格式导致的。
3. ICMP_PHOTURIS_DECRYPT_FAILED：表示解密失败。在将照片传输到服务器时，可能发生了JPEG或PNG等加密格式的转换失败。
4. ICMP_PHOTURIS_NEED_AUTHN：表示当前没有身份验证。例如，当尝试连接到一个不安全的服务器时会收到这个错误。
5. ICMP_PHOTURIS_NEED_AUTHZ：表示当前没有授权。例如，当服务器要求提供身份验证时，但是客户端无法提供授权信息时会收到这个错误。

此外，还定义了一个常量ICMP_TYPE_MAX，用于表示ICMP协议中可用的数据类型数量。


```
#define		ICMP_PHOTURIS_AUTH_FAILED	1	/* auth failed */
#define		ICMP_PHOTURIS_DECOMPRESS_FAILED	2	/* decompress failed */
#define		ICMP_PHOTURIS_DECRYPT_FAILED	3	/* decrypt failed */
#define		ICMP_PHOTURIS_NEED_AUTHN	4	/* no authentication */
#define		ICMP_PHOTURIS_NEED_AUTHZ	5	/* no authorization */
#define	ICMP_TYPE_MAX		40

#define	ICMP_INFOTYPE(type)						\
	((type) == ICMP_ECHOREPLY || (type) == ICMP_ECHO ||		\
	(type) == ICMP_RTRADVERT || (type) == ICMP_RTRSOLICIT ||	\
	(type) == ICMP_TSTAMP || (type) == ICMP_TSTAMPREPLY ||		\
	(type) == ICMP_INFO || (type) == ICMP_INFOREPLY ||		\
	(type) == ICMP_MASK || (type) == ICMP_MASKREPLY)

/*
 * Echo message data
 */
```cpp

这段代码定义了一个名为 `icmp_msg_echo` 的结构体，该结构体有三个成员：`icmp_id`、`icmp_seq` 和 `icmp_data`。`icmp_id` 和 `icmp_seq` 是 `icmp_msg_echo` 结构体的基本格式，其中 `icmp_id` 是 8 字节无类域，`icmp_seq` 是 16 字节的序列号。`icmp_data` 是可选的，它的长度没有限制，但是必须是 8 字节。

另外，该结构体还定义了一个名为 `icmp_msg_needfrag` 的结构体，其中包含一个 `icmp_void` 字段，它的值必须是 0。`icmp_msg_needfrag` 结构体还有一个 `icmp_ip` 字段，它是一个 `ip` 头，其长度必须是 24 字节，且必须是下一个路由器的 IP 地址。

总体来说，这两个结构体定义了 ICMP（Internet Control Message Protocol，互联网报文协议）回送请求报文和响应报文的数据结构。`icmp_msg_echo` 结构体包含一个完整的 `icmp_msg_echo` 报文，而 `icmp_msg_needfrag` 结构体则包含一个需要传递给下一个路由器的 `icmp_needfrag` 报文，其中包含一个 `icmp_void` 字段，它的值为 0，说明不需要传递数据。


```
struct icmp_msg_echo {
	uint16_t	icmp_id;
	uint16_t	icmp_seq;
	uint8_t		icmp_data __flexarr;	/* optional data */
};

/*
 * Fragmentation-needed (unreachable) message data
 */
struct icmp_msg_needfrag {
	uint16_t	icmp_void;		/* must be zero */
	uint16_t	icmp_mtu;		/* MTU of next-hop network */
	uint8_t		icmp_ip __flexarr;	/* IP hdr + 8 bytes of pkt */
};

```cpp

该代码定义了一个名为`icmp_msg_quote`的结构体，表示ICMP协议中的一个报文 quote。该结构体包含以下字段：

1. `icmp_void`字段，表示一个8位的ICMP类型，根据字段名称可以猜测这个字段可能表示一个特殊的void类型，但需要进一步确认。
2. `icmp_gwaddr`字段，表示用于发送RFC1256类型路由更新的目标IP地址。
3. `icmp_pptr`字段，表示坏字节字段的指针。
4. `icmp_ip`字段，表示IP头部，包括8个字节的数据和一些IP头部字段的偏移量。

这个结构体被用于定义一个`icmp_msg_rtradvert`结构体，表示用于发送RFC1256类型路由更新的一种ICMP消息类型。这个结构体中包含以下字段：

1. `icmp_num_addrs`字段，表示目标地址的数量，也就是RFC1256中定义的地址/前缀对的数量。
2. `icmp_wpa`字段，表示使用802.1X认证的启用值，如果这个字段为2，则表示使用802.1X身份验证，否则表示使用简单的PAP或CHAP认证。
3. `icmp_lifetime`字段，表示路由的寿命，以秒为单位。
4. `icmp_msg_rtr_data`字段，包含一个指向RFC1256类型路由更新消息的指针，以及一些用于获取路由更新的额外信息，如路由ID、前缀和掩码等。

这个`icmp_msg_rtradvert`结构体被用于在ICMP协议中发送RFC1256类型路由更新消息。


```
/*
 *  Unreachable, source quench, redirect, time exceeded,
 *  parameter problem message data
 */
struct icmp_msg_quote {
	uint32_t	icmp_void;		/* must be zero */
#define icmp_gwaddr	icmp_void		/* router IP address to use */
#define icmp_pptr	icmp_void		/* ptr to bad octet field */
	uint8_t		icmp_ip __flexarr;	/* IP hdr + 8 bytes of pkt */
};

/*
 * Router advertisement message data, RFC 1256
 */
struct icmp_msg_rtradvert {
	uint8_t		icmp_num_addrs;		/* # of address / pref pairs */
	uint8_t		icmp_wpa;		/* words / address == 2 */
	uint16_t	icmp_lifetime;		/* route lifetime in seconds */
	struct icmp_msg_rtr_data {
		uint32_t	icmp_void;
```cpp

这段代码定义了一个名为“icmp_rtr”的结构体，它表示一个IPICMP路由器。该结构体中定义了一个名为“icmp_gwaddr”的成员，它是一个指向IP地址的指针，用于指定路由器的全局地址。此外，该结构体中定义了一个名为“icmp_pref”的成员，它是一个8位无符号整数，用于指定路由器的偏好设置，可以设置为0表示禁用全局地址。

另外，该结构体中还定义了一个名为“icmp_rtr_prefl”的成员，它是一个用于存储预设全局地址的掩码，其值为0x80000000表示禁用全局地址。

最后，该结构体中定义了一个名为“icmp_msg_tstamp”的成员，它用于存储IPICMP消息的时间戳数据，包括标识符、序列号、原始时间戳、接收时间戳和发送时间戳。


```
#define icmp_gwaddr		icmp_void	/* router IP address */
		uint32_t	icmp_pref;	/* router preference (usu 0) */
	} icmp_rtr __flexarr;			/* variable # of routers */
};
#define ICMP_RTR_PREF_NODEFAULT	0x80000000	/* do not use as default gw */

/*
 * Timestamp message data
 */
struct icmp_msg_tstamp {
	uint32_t	icmp_id;		/* identifier */
	uint32_t	icmp_seq;		/* sequence number */
	uint32_t	icmp_ts_orig;		/* originate timestamp */
	uint32_t	icmp_ts_rx;		/* receive timestamp */
	uint32_t	icmp_ts_tx;		/* transmit timestamp */
};

```cpp

这两段代码定义了ICMP消息的不同部分，包括消息ID、序列号、地址掩码和traceroute信息。

1. `icmp_msg_mask`结构体定义了ICMP消息的掩码部分，包括ICMP ID、序列号和地址掩码。这个结构体被用于下面的`icmp_traceroute`结构体中。

2. `icmp_msg_traceroute`结构体定义了ICMP traceroute信息，包括出站 hop count、返回 hop count、链速率和最大传输单元（MTU）。这个结构体也被用于下面的`icmp_traceroute`函数中。


```
/*
 * Address mask message data, RFC 950
 */
struct icmp_msg_mask {
	uint32_t	icmp_id;		/* identifier */
	uint32_t	icmp_seq;		/* sequence number */
	uint32_t	icmp_mask;		/* address mask */
};

/*
 * Traceroute message data, RFC 1393, RFC 1812
 */
struct icmp_msg_traceroute {
	uint16_t	icmp_id;		/* identifier */
	uint16_t	icmp_void;		/* unused */
	uint16_t	icmp_ohc;		/* outbound hop count */
	uint16_t	icmp_rhc;		/* return hop count */
	uint32_t	icmp_speed;		/* link speed, bytes/sec */
	uint32_t	icmp_mtu;		/* MTU in bytes */
};

```cpp

该代码定义了一个名为"icmp_msg_dnsreply"的结构体，用于表示IP Internet Control Message Protocol (ICMP)回复消息数据。这个结构体定义了以下字段：

- icmp_id：ICMP标识符，用于标识请求消息
- icmp_seq：ICMP序列号，用于标识请求消息
- icmp_ttl：IP 时间戳服务生命周期
- icmp_names：变长字符数组，用于保存DNS服务器响应的消息名称

这个结构体可以用于定义ICMP协议的回复消息，以便在应用程序中处理DNS查询请求。


```
/*
 * Domain name reply message data, RFC 1788
 */
struct icmp_msg_dnsreply {
	uint16_t	icmp_id;		/* identifier */
	uint16_t	icmp_seq;		/* sequence number */
	uint32_t	icmp_ttl;		/* time-to-live */
	uint8_t		icmp_names __flexarr;	/* variable number of names */
};

/*
 * Generic identifier, sequence number data
 */
struct icmp_msg_idseq {
	uint16_t	icmp_id;
	uint16_t	icmp_seq;
};

```cpp

这段代码定义了一个名为 `icmp_msg` 的结构体，用于表示ICMP(Internet Control Message Protocol)消息。该结构体包含多个成员，用于表示ICMP消息的不同方面，如类型、参数、时间戳等。每个成员的类型都被定义为一个枚举类型，例如 `ICMP_MSG_ECHO`、`ICMP_MSG_UNREACH` 等。这些枚举类型用于标识不同的ICMP消息类型。

`union icmp_msg` 表示该结构体是一个名为 `icmp_msg` 的联合体，它包含多个成员变量。该联合体使用 `union` 关键字，将多个成员合并成一个单一变量。

`struct icmp_msg_echo` 等成员变量表示了ICMP消息中的回波(echo)类型。例如，`ICMP_MSG_ECHO` 表示回波请求(echo request)，而 `ICMP_MSG_QUOTE` 表示回波响应(echo response)。

`struct icmp_msg_quote` 等成员变量表示了ICMP消息中的计数值。例如，`ICMP_MSG_NONE` 表示未发送数据(none)，而 `ICMP_MSG_QUOTE` 表示发送数据(quote)请求。

`struct icmp_msg_needfrag` 等成员变量表示了ICMP消息中的分片请求。例如，`ICMP_MSG_FRAG_ACK` 表示分片已准备好(fragment acknowledgment)请求，而 `ICMP_MSG_FRAG_REJECT` 表示分片已拒绝(fragment reject)请求。

`struct icmp_msg_quote` 等成员变量表示了ICMP消息中的源地址。例如，`ICMP_MSG_SRC_ADDRESS` 表示源地址(source address)请求，而 `ICMP_MSG_DST_ADDRESS` 表示目标地址(destination address)回复。

`struct icmp_msg_rtrsolicit` 等成员变量表示了ICMP消息中的回送请求。例如，`ICMP_MSG_RTR_SOLICITATE` 表示回送请求(trash message)，而 `ICMP_MSG_RTR_ACK` 表示回送确认(trash acknowledge)。

`struct icmp_msg_quote` 等成员变量表示了ICMP消息中的时间戳。例如，`ICMP_MSG_TIMESTAMP` 表示时间戳请求，而 `ICMP_MSG_ID_QUOTE` 表示时间戳回答。


```
/*
 * ICMP message union
 */
union icmp_msg {
	struct icmp_msg_echo	   echo;	/* ICMP_ECHO{REPLY} */
	struct icmp_msg_quote	   unreach;	/* ICMP_UNREACH */
	struct icmp_msg_needfrag   needfrag;	/* ICMP_UNREACH_NEEDFRAG */
	struct icmp_msg_quote	   srcquench;	/* ICMP_SRCQUENCH */
	struct icmp_msg_quote	   redirect;	/* ICMP_REDIRECT (set to 0) */
	uint32_t		   rtrsolicit;	/* ICMP_RTRSOLICIT */
	struct icmp_msg_rtradvert  rtradvert;	/* ICMP_RTRADVERT */
	struct icmp_msg_quote	   timexceed;	/* ICMP_TIMEXCEED */
	struct icmp_msg_quote	   paramprob;	/* ICMP_PARAMPROB */
	struct icmp_msg_tstamp	   tstamp;	/* ICMP_TSTAMP{REPLY} */
	struct icmp_msg_idseq	   info;	/* ICMP_INFO{REPLY} */
	struct icmp_msg_mask	   mask;	/* ICMP_MASK{REPLY} */
	struct icmp_msg_traceroute traceroute;	/* ICMP_TRACEROUTE */
	struct icmp_msg_idseq	   dns;		/* ICMP_DNS */
	struct icmp_msg_dnsreply   dnsreply;	/* ICMP_DNSREPLY */
};

```cpp

这段代码定义了两个宏，名为 `icmp_pack_hdr` 和 `icmp_pack_hdr_echo`。这两个宏的主要作用是定义了 `icmp_pack_hdr` 函数和 `icmp_pack_hdr_echo` 函数的参数和返回值类型和使用方式。

具体来说，这两个宏定义了一个名为 `icmp_pack_hdr` 的函数，该函数接受一个 `ICMP_HDR` 类型的头指针，以及一个 `ICMP_CODE` 类型的参数 `code`。函数内部先定义了一个名为 `icmp_pack_p` 的指向 `ICMP_HDR` 类型的指针，然后将 `ICMP_TYPE` 字段设置为传入的 `type`，将 `ICMP_CODE` 字段设置为 `code`。最后在循环中进行一些处理，确保循环条件为真。

另一个名为 `icmp_pack_hdr_echo` 的函数，与 `icmp_pack_hdr` 类似，但输出的是 `ICMP_MSG_ECHO` 类型，而不是 `ICMP_MSG_TRACE` 类型。具体来说，该函数接受一个 `ICMP_HDR` 类型的头指针，以及一个 `ICMP_CODE` 类型的参数 `code`。函数内部定义了一个名为 `echo_pack_p` 的指向 `ICMP_MSG_ECHO` 类型的指针，然后将 `ICMP_TYPE` 字段设置为 `ICMP_MSG_ECHO`，将 `ICMP_CODE` 字段设置为 `code`。接下来，将输入的 `ICMP_HDR_LEN` 中的数据复制到输出指针 `echo_pack_p->icmp_data` 中。

总的来说，这段代码定义了两个函数，用于对 `ICMP_HDR` 类型的数据进行包装，通过 `ICMP_PACK_HDR` 和 `ICMP_PACK_HDR_ECHO` 函数来实现。


```
#ifndef __GNUC__
# pragma pack()
#endif

#define icmp_pack_hdr(hdr, type, code) do {				\
	struct icmp_hdr *icmp_pack_p = (struct icmp_hdr *)(hdr);	\
	icmp_pack_p->icmp_type = type; icmp_pack_p->icmp_code = code;	\
} while (0)

#define icmp_pack_hdr_echo(hdr, type, code, id, seq, data, len) do {	\
	struct icmp_msg_echo *echo_pack_p = (struct icmp_msg_echo *)	\
		((uint8_t *)(hdr) + ICMP_HDR_LEN);			\
	icmp_pack_hdr(hdr, type, code);					\
	echo_pack_p->icmp_id = htons(id);				\
	echo_pack_p->icmp_seq = htons(seq);				\
	memmove(echo_pack_p->icmp_data, data, len);			\
} while (0)

```cpp

这两行代码是在定义两个函数，名为 `icmp_pack_hdr_quote` 和 `icmp_pack_hdr_mask`。这两个函数都是用于在IP头中封装 ICMP 协议头。

具体来说，这两行代码会创建一个 `icmp_msg_quote` 结构体，并将其作为参数传递给 `memmove` 函数。`icmp_pack_hdr` 函数用于将 ICMP 头压缩，然后将 `word` 字段设置为 `hdr` 的下一个字段，`pkt` 参数包含 `ICMP_PACKET_TRANSPORT_PORT` 字段，表示数据传输的 IP 头部。

`icmp_pack_hdr_quote` 函数在 `icmp_pack_hdr` 函数的基础上，将 `IERROR_ICMP_ERROR` 字段设置为 `htonl(word)`，然后将 `ICMP_PACKET_TRANSPORT_PORT` 字段的内容复制到 `icmp_msg_quote` 结构体的 `icmp_ip` 字段中。

`icmp_pack_hdr_mask` 函数在 `icmp_pack_hdr` 函数的基础上，将 `IERROR_ICMP_ERROR` 字段设置为 `htonl(word)`，然后将 `IERROR_ICMP_MASK_QUOTE` 字段设置为 `htonl(mask)`。这个字段用于标识错误代码，以便后续代码进行处理。

总的来说，这两行代码定义了两个函数，用于对 ICMP 头进行封装，以支持更多的功能。


```
#define icmp_pack_hdr_quote(hdr, type, code, word, pkt, len) do {	\
	struct icmp_msg_quote *quote_pack_p = (struct icmp_msg_quote *)	\
		((uint8_t *)(hdr) + ICMP_HDR_LEN);			\
	icmp_pack_hdr(hdr, type, code);					\
	quote_pack_p->icmp_void = htonl(word);				\
	memmove(quote_pack_p->icmp_ip, pkt, len);			\
} while (0)

#define icmp_pack_hdr_mask(hdr, type, code, id, seq, mask) do {		\
	struct icmp_msg_mask *mask_pack_p = (struct icmp_msg_mask *)	\
		((uint8_t *)(hdr) + ICMP_HDR_LEN);			\
	icmp_pack_hdr(hdr, type, code);					\
	mask_pack_p->icmp_id = htons(id);				\
	mask_pack_p->icmp_seq = htons(seq);				\
	mask_pack_p->icmp_mask = htonl(mask);				\
} while (0)

```cpp

这段代码定义了一个名为`icmp_pack_hdr_needfrag`的定义，它接受一个结构体类型的参数`frag_pack_p`，该结构体定义了`icmp_msg_needfrag`结构体。

函数的作用是向IP数据报中添加一个ICMP消息头和数据，使得数据能够正确发送。数据发送完毕后，函数退出循环。

这里简要解释一下代码的作用：

1. 定义了一个名为`icmp_pack_hdr_needfrag`的函数，它接受一个结构体类型的参数`frag_pack_p`。
2. 调用`icmp_pack_hdr`函数，将数据发送出去。
3. 将`frag_pack_p`结构体中的`icmp_void`成员设置为`0`，表示这是一个数据包，而不是一个请求。
4. 将`frag_pack_p`结构体中的`icmp_mtu`成员设置为`mtu`的值。
5. 使用`memmove`函数将`pkt`中的数据复制到`frag_pack_p`结构体中的`icmp_ip`成员中。
6. 将`frag_pack_p`结构体中的`icmp_msg_needfrag`成员的`icmp_void`成员设置为`0`，表示这是一个数据包，而不是一个请求。
7. 将`frag_pack_p`结构体中的`icmp_msg_needfrag`成员的`icmp_mtu`成员设置为`mtu`的值。
8. 通过循环调用`icmp_pack_hdr`函数，将数据发送出去，并确保数据发送完毕后退出循环。


```
#define icmp_pack_hdr_needfrag(hdr, type, code, mtu, pkt, len) do {	\
	struct icmp_msg_needfrag *frag_pack_p =				\
	(struct icmp_msg_needfrag *)((uint8_t *)(hdr) + ICMP_HDR_LEN);	\
	icmp_pack_hdr(hdr, type, code);					\
	frag_pack_p->icmp_void = 0;					\
	frag_pack_p->icmp_mtu = htons(mtu);				\
	memmove(frag_pack_p->icmp_ip, pkt, len);			\
} while (0)

#endif /* DNET_ICMP_H */

```