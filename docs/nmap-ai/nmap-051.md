# Nmap源码解析 51

# `libpcap/pcap-dag.c`

这段代码是一个用于实现 packet capture（数据包捕获）接口的函数，特别是在 Endace DAG（数据生成器）卡上。作者为 Richard Littin 和 Sean Irvine，他们曾就读于 Realtwo.com 公司。目前，作者 Jesper Peterson 和 Koryn Grant 和 Stephen Donnelly 负责管理此代码。

这段代码的作用是，当系统支持 configure-profile 参数时，从 Endace DAG 卡上接收数据包，并捕获到系统内存中。当配置过程完成时，将捕获的数据发送到配置文件中。如果当前系统不支持 configure-profile，那么此函数将不予考虑。


```cpp
/*
 * pcap-dag.c: Packet capture interface for Endace DAG cards.
 *
 * Authors: Richard Littin, Sean Irvine ({richard,sean}@reeltwo.com)
 * Modifications: Jesper Peterson
 *                Koryn Grant
 *                Stephen Donnelly <stephen.donnelly@endace.com>
 */

#ifdef HAVE_CONFIG_H
#include <config.h>
#endif

#include <sys/param.h>			/* optionally get BSD define */

```

这段代码是一个网络编程中的的工具函数，它包括了标准库头文件（stdlib.h，string.h，errno.h）以及一些网络接口头文件（如netinet/in.h，sys/mman.h，sys/socket.h，sys/types.h，<net/if.h>）。

首先，它引入了pcap-int库头文件，这个库可能是一个用于网络数据捕获和分析的库。

接下来，它定义了一个名为“mbuf”的结构体，可能用于在数据套接字上发送数据。

然后，它定义了一个名为“rtentry”的结构体，可能用于存储路由表条目的相关数据。

接着，它引入了netinet/in.h头文件，然后使用sys/mman.h和sys/socket.h库，实现了对网络接口的访问。

然后，它通过获取当前套接字地址和端口号，创建了一个套接字对象。

接下来，它使用rtentry结构体中存储的路由表条目，将数据发送到该套接字。

最后，它检查套接字是否可用，如果不可用，则输出错误信息，并返回一个errno值。


```cpp
#include <stdlib.h>
#include <string.h>
#include <errno.h>

#include "pcap-int.h"

#include <netinet/in.h>
#include <sys/mman.h>
#include <sys/socket.h>
#include <sys/types.h>
#include <unistd.h>

struct mbuf;		/* Squelch compiler warnings on some platforms for */
struct rtentry;		/* declarations in <net/if.h> */
#include <net/if.h>

```

这段代码的作用是引入了DAGNew库、DAGAPI库和DAGPCI库，并定义了DAG设备的名称规则。还引入了PCAP-DAG库，用于在数据包传输过程中捕获和分析DAG数据包。

具体来说，这段代码定义了DAG设备的命名规则，包括设备名称、设备号、端口号、协议类型和数据流类型。其中，设备号和端口号范围在0到DAG_MAX_BOARDS和DAG_STREAM_MAX之间。协议类型和数据流类型可以从0到DAG_MAX_BOARDS-1和DAG_STREAM_MAX-1。

此外，还定义了DAG设备的一些常量，如DAG_MAX_BOARDS、DAG_STREAM_MAX和DAG_MIN_BOARDS，用于在代码中进行参考。


```cpp
#include "dagnew.h"
#include "dagapi.h"
#include "dagpci.h"
#include "dag_config_api.h"

#include "pcap-dag.h"

/*
 * DAG devices have names beginning with "dag", followed by a number
 * from 0 to DAG_MAX_BOARDS, then optionally a colon and a stream number
 * from 0 to DAG_STREAM_MAX.
 */
#ifndef DAG_MAX_BOARDS
#define DAG_MAX_BOARDS 32
#endif


```

这段代码定义了一系列ERF（Encapsulation Resistance Format）类型，用于标识不同种类的通信协议。ERF类型是一种二进制数据结构，描述了数据如何在不同协议之间传输。每种ERF类型都有自己的编码和名称。

这段代码的作用是定义了四种ERF类型：AAL5、MC_HDLC、MC_RAW和MC_ATM。这些类型将在编译时根据预设的定义进行编译，并在程序运行时可以被使用。例如，在设计和实现通信协议时，可以使用这些类型来定义数据结构、协议头和错误代码等元素。


```cpp
#ifndef ERF_TYPE_AAL5
#define ERF_TYPE_AAL5               4
#endif

#ifndef ERF_TYPE_MC_HDLC
#define ERF_TYPE_MC_HDLC            5
#endif

#ifndef ERF_TYPE_MC_RAW
#define ERF_TYPE_MC_RAW             6
#endif

#ifndef ERF_TYPE_MC_ATM
#define ERF_TYPE_MC_ATM             7
#endif

```

这段代码定义了ErfTypeMC中的几个成员变量，包括：

1. `ERF_TYPE_MC_RAW_CHANNEL`：定义了一个名为`ERF_TYPE_MC_RAW_CHANNEL`的成员变量，值为8。
2. `ERF_TYPE_MC_AAL5`：定义了一个名为`ERF_TYPE_MC_AAL5`的成员变量，值为9。
3. `ERF_TYPE_COLOR_HDLC_POS`：定义了一个名为`ERF_TYPE_COLOR_HDLC_POS`的成员变量，值为10。
4. `ERF_TYPE_COLOR_ETH`：定义了一个名为`ERF_TYPE_COLOR_ETH`的成员变量，值为11。

这些成员变量在ErfTypeMC结构体中定义了ErfTypeMC的完整功能。


```cpp
#ifndef ERF_TYPE_MC_RAW_CHANNEL
#define ERF_TYPE_MC_RAW_CHANNEL     8
#endif

#ifndef ERF_TYPE_MC_AAL5
#define ERF_TYPE_MC_AAL5            9
#endif

#ifndef ERF_TYPE_COLOR_HDLC_POS
#define ERF_TYPE_COLOR_HDLC_POS     10
#endif

#ifndef ERF_TYPE_COLOR_ETH
#define ERF_TYPE_COLOR_ETH          11
#endif

```

这段代码定义了一些ERF（Enhanced Recursive Falling转置）类型的枚举类型，用于说明输入输出（input/output，in/out）数据的传输类型。下面是每个枚举类型的说明：

```cpp
#ifndef ERF_TYPE_MC_AAL2
#define ERF_TYPE_MC_AAL2            12
#endif

#ifndef ERF_TYPE_IP_COUNTER
#define ERF_TYPE_IP_COUNTER         13
#endif

#ifndef ERF_TYPE_TCP_FLOW_COUNTER
#define ERF_TYPE_TCP_FLOW_COUNTER   14
#endif

#ifndef ERF_TYPE_DSM_COLOR_HDLC_POS
#define ERF_TYPE_DSM_COLOR_HDLC_POS 15
#endif
```

这里的`#ifndef`和`#define`是预处理指令，用于防止重复定义。当定义了某个ERF类型后，该类型的枚举类型将不再被定义。

每个ERF类型都被赋了一个唯一的数字，这些数字用于标识数据传输的类型。例如，`ERF_TYPE_MC_AAL2`的值为12，表示它是支持输入输出数据的类型。


```cpp
#ifndef ERF_TYPE_MC_AAL2
#define ERF_TYPE_MC_AAL2            12
#endif

#ifndef ERF_TYPE_IP_COUNTER
#define ERF_TYPE_IP_COUNTER         13
#endif

#ifndef ERF_TYPE_TCP_FLOW_COUNTER
#define ERF_TYPE_TCP_FLOW_COUNTER   14
#endif

#ifndef ERF_TYPE_DSM_COLOR_HDLC_POS
#define ERF_TYPE_DSM_COLOR_HDLC_POS 15
#endif

```

这段代码定义了一系列ERF（Enhanced Recovery Framework）类型，用于描述无线通信中的信息适配和链路级规格。下面是每个ERF类型的简要描述：

1. DSM-COLOR-ETH：用于描述具有Color模式（RGB和CM）的以太坊以太桥。
2. ERF-TYPE-COLOR-MC-HDLC-POS：用于描述支持Color模式中的一种（MC或HDLC）的链路级规程（POS）。
3. ERF-TYPE-AAL2：用于描述支持Color模式中的另一种（AAL2）的无线接入层（AAL）。
4. ERF-TYPE-COLOR-HASH-POS：用于描述支持Color模式中的的一种（哈希模式）的链路级规程（POS）。


```cpp
#ifndef ERF_TYPE_DSM_COLOR_ETH
#define ERF_TYPE_DSM_COLOR_ETH      16
#endif

#ifndef ERF_TYPE_COLOR_MC_HDLC_POS
#define ERF_TYPE_COLOR_MC_HDLC_POS  17
#endif

#ifndef ERF_TYPE_AAL2
#define ERF_TYPE_AAL2               18
#endif

#ifndef ERF_TYPE_COLOR_HASH_POS
#define ERF_TYPE_COLOR_HASH_POS     19
#endif

```

这段代码定义了一系列ERF（Entity-Relationship Framework）类型枚举，包括ERF类型颜色哈希以太坊（ERF_TYPE_COLOR_HASH_ETH），ERF类型无限波浪（ERF_TYPE_INFINIBAND），ERF类型IPv4（ERF_TYPE_IPV4），ERF类型IPv6（ERF_TYPE_IPV6）。其中，ERF类型颜色哈希以太坊（ERF_TYPE_COLOR_HASH_ETH）枚举的值为20，表示该类型为颜色哈希以太坊。


```cpp
#ifndef ERF_TYPE_COLOR_HASH_ETH
#define ERF_TYPE_COLOR_HASH_ETH     20
#endif

#ifndef ERF_TYPE_INFINIBAND
#define ERF_TYPE_INFINIBAND         21
#endif

#ifndef ERF_TYPE_IPV4
#define ERF_TYPE_IPV4               22
#endif

#ifndef ERF_TYPE_IPV6
#define ERF_TYPE_IPV6               23
#endif

```

这段代码定义了四个ERF（Entity-Relationship Diagram）类型：ERF_TYPE_RAW_LINK、ERF_TYPE_INFINIBAND_LINK、ERF_TYPE_META和ERF_TYPE_PAD。这些枚举类型分别使用了不同的数字，ERF_TYPE_RAW_LINK对应24，ERF_TYPE_INFINIBAND_LINK对应25，ERF_TYPE_META对应27，ERF_TYPE_PAD对应48。

这个代码的作用是定义了ERF类型的枚举，用于描述ERF（实体关系图）的数据结构。在ERF中，使用数字来表示不同的类型，例如ERF_TYPE_RAW_LINK表示原始类型的链接，ERF_TYPE_INFINIBAND_LINK表示异步事务类型的链接，以此类推。

这些枚举类型可以被用于编写ERF模型的代码，使程序更加易于理解和维护。例如，可以使用ERF_TYPE_META类型来表示元数据，ERF_TYPE_PAD类型来表示增强型代理。


```cpp
#ifndef ERF_TYPE_RAW_LINK
#define ERF_TYPE_RAW_LINK           24
#endif

#ifndef ERF_TYPE_INFINIBAND_LINK
#define ERF_TYPE_INFINIBAND_LINK    25
#endif

#ifndef ERF_TYPE_META
#define ERF_TYPE_META               27
#endif

#ifndef ERF_TYPE_PAD
#define ERF_TYPE_PAD                48
#endif

```

这段代码定义了两个头文件：ATM_CELL_SIZE和ATM_HDR_SIZE，以及一个常量MTP2_SENT_OFFSET，MTP2_ANNEX_A_USED_OFFSET，MTP2_LINK_NUMBER_OFFSET和MTP2_HDR_LEN。

ATM_CELL_SIZE定义了ATM单元格的大小，即一个ATM单元格包含52个字节。

ATM_HDR_SIZE定义了ATM头部的大小，即一个ATM头部包含4个字节。

MTP2_SENT_OFFSET定义了一个用于MTP2消息发送的偏移量，它的值为1。

MTP2_ANNEX_A_USED_OFFSET定义了一个用于MTP2消息使用的标志位，它的值为0或1。

MTP2_LINK_NUMBER_OFFSET定义了一个用于表示MPL标签的偏移量，它的值为2。

MTP2_HDR_LEN定义了一个用于表示MTP2头部长度的常量，它的值为4。

因此，这些定义可以在程序中被用来控制ATM单元格的大小和位置，以及MTP2消息的发送和头部信息的结构。


```cpp
#define ATM_CELL_SIZE		52
#define ATM_HDR_SIZE		4

/*
 * A header containing additional MTP information.
 */
#define MTP2_SENT_OFFSET		0	/* 1 byte */
#define MTP2_ANNEX_A_USED_OFFSET	1	/* 1 byte */
#define MTP2_LINK_NUMBER_OFFSET		2	/* 2 bytes */
#define MTP2_HDR_LEN			4	/* length of the header */

#define MTP2_ANNEX_A_NOT_USED      0
#define MTP2_ANNEX_A_USED          1
#define MTP2_ANNEX_A_USED_UNKNOWN  2

```

这段代码定义了一个名为“sunatm_hdr”的结构体，用于表示ATM（Asynchronous Transport Module）数据报中的头部信息。这个结构体包括一个无符号的“flags”字段，用于表示数据报的类型和传输类型；一个无符号的“vpi”字段，用于表示数据报的VPI（Virtual Program Integrity）标识；一个无符号的“vci”字段，用于表示数据报的VCI（Virtual Connection Identifier）标识。

接下来定义了一个名为“pcap_dag”的结构体，用于表示DAG（Data Adaptation Group）数据报的头部信息。这个结构体包括一个无符号的“stat”字段，用于表示数据报统计信息；一个指向内存的“dag_mem_bottom”字指针，用于表示DAG数据报的当前内存bottom指针；一个指向内存的“dag_mem_top”字指针，用于表示DAG数据报的当前内存top指针；一个无符号的“dag_fcs_bits”字段，表示数据报的FCS（Frame Container Serialization）校验比特数；一个无符号的“dag_flags”字段，表示数据报的标志；一个无符号的“dag_stream”字段，表示数据报的流；一个无符号的“dag_timeout”字段，表示数据报超时时间；一个指向整型对象的“dag_ref”指针，用于表示DAG数据报的参考对象；一个无符号的“dag_root”字段，表示DAG数据报的根组件；一个无符号的“drop_attr”字段，用于表示DAG数据报的 drop 属性；一个包含两个无符号时间的“required_select_timeout”字段，用于表示数据报的 Select 定时器超时时间。最后定义了一个名为“pcap_dag_print_stats”的函数，用于打印DAG数据报的统计信息。


```cpp
/* SunATM pseudo header */
struct sunatm_hdr {
	unsigned char	flags;		/* destination and traffic type */
	unsigned char	vpi;		/* VPI */
	unsigned short	vci;		/* VCI */
};

/*
 * Private data for capturing on DAG devices.
 */
struct pcap_dag {
	struct pcap_stat stat;
	u_char	*dag_mem_bottom;	/* DAG card current memory bottom pointer */
	u_char	*dag_mem_top;	/* DAG card current memory top pointer */
	int	dag_fcs_bits;	/* Number of checksum bits from link layer */
	int	dag_flags;	/* Flags */
	int	dag_stream;	/* DAG stream number */
	int	dag_timeout;	/* timeout specified to pcap_open_live.
				 * Same as in linux above, introduce
				 * generally? */
	dag_card_ref_t dag_ref; /* DAG Configuration/Status API card reference */
	dag_component_t dag_root;	/* DAG CSAPI Root component */
	attr_uuid_t drop_attr;  /* DAG Stream Drop Attribute handle, if available */
	struct timeval required_select_timeout;
				/* Timeout caller must use in event loops */
};

```

这段代码定义了一个名为 `pcap_dag_node` 的结构体，它表示数据链路层数据包中的数据分片。该结构体包含三个成员：

1. `next`：指向下一个分片的数据链路层数据包的指针。
2. `p`：指向该分片数据链路层数据包的指针。
3. `pid`：该分片数据链路层数据包的发送者进程ID。

该结构体定义了一个名为 `pcap_dags` 的指向数据链路层数据包分片的指针数组。

该代码还定义了一个名为 `atexit_handler_installed` 的整数变量，用于检查是否已经安装了 atexit handler。如果没有安装，则该函数不会被调用。

该代码定义了一个名为 `endian_test_word` 的整数变量，用于表示二进制数据中的高字节。

该代码还定义了一个名为 `TempPkt` 的整数数组，用于存储数据链路层数据包中的数据。


```cpp
typedef struct pcap_dag_node {
	struct pcap_dag_node *next;
	pcap_t *p;
	pid_t pid;
} pcap_dag_node_t;

static pcap_dag_node_t *pcap_dags = NULL;
static int atexit_handler_installed = 0;
static const unsigned short endian_test_word = 0x0100;

#define IS_BIGENDIAN() (*((unsigned char *)&endian_test_word))

#define MAX_DAG_PACKET 65536

static unsigned char TempPkt[MAX_DAG_PACKET];

```



这段代码定义了一系列用于 DAG（数据链路应用程序）的函数和宏。它们的功能是：

1. 为没有 DAG 大型数据流 API 的编译器定义了函数和宏。
2. 实现了一些与 DAG 相关的函数，包括：
  a. dag_attach_stream64 - 用于将数据流连接到数据链路适配器。
  b. dag_get_stream_poll64 - 用于从数据链路中检索数据包。
  c. dag_set_stream_poll64 - 用于设置数据链路中数据包的拉取（poll）。
  d. dag_size_t - 定义了一个名为 dag_size_t 的枚举类型，用于表示 DAG 数据链路的最大数据包大小。
3. 实现了 dag_stats、dag_set_datalink 和 dag_get_datalink 函数。这些函数用于计算和设置 DAG 统计信息，如发送和接收数据包的数量。
4. 实现了 delete_pcap_dag 函数。这个函数释放了一个 DAG 数据链路适配器，确保所有当前连接的数据流都已断开。


```cpp
#ifndef HAVE_DAG_LARGE_STREAMS_API
#define dag_attach_stream64(a, b, c, d) dag_attach_stream(a, b, c, d)
#define dag_get_stream_poll64(a, b, c, d, e) dag_get_stream_poll(a, b, c, d, e)
#define dag_set_stream_poll64(a, b, c, d, e) dag_set_stream_poll(a, b, c, d, e)
#define dag_size_t uint32_t
#endif

static int dag_stats(pcap_t *p, struct pcap_stat *ps);
static int dag_set_datalink(pcap_t *p, int dlt);
static int dag_get_datalink(pcap_t *p);
static int dag_setnonblock(pcap_t *p, int nonblock);

static void
delete_pcap_dag(const pcap_t *p)
{
	pcap_dag_node_t *curr = NULL, *prev = NULL;

	for (prev = NULL, curr = pcap_dags; curr != NULL && curr->p != p; prev = curr, curr = curr->next) {
		/* empty */
	}

	if (curr != NULL && curr->p == p) {
		if (prev != NULL) {
			prev->next = curr->next;
		} else {
			pcap_dags = curr->next;
		}
	}
}

```

这段代码的作用是执行有隙等距关机（ graceful shutdown）操作，释放动态内存，并关闭 DAG 设备的文件描述符。通过调用 dag_stop_stream、dag_detach_stream 和一些辅助函数，实现了上述目的。

具体来说，这段代码以下几个步骤实现了 graceful shutdown：

1. 调用 dag_stop_stream，通知用户数据正在按字节顺序从文件中读取，并尝试停止数据读取。如果遇到错误，打印错误信息并返回。
2. 调用 dag_detach_stream，通知用户数据正在按字节顺序从文件中写入，并尝试停止数据写入。如果遇到错误，打印错误信息并返回。
3. 如果 pcap_t 结构中仍然存在动态内存，调用 dag_config_dispose 函数，释放这些内存，并打印错误信息。然后将 p->fd 设置为 -1，以确保不会进一步释放资源。最后，删除 pcap_dag 结构，并调用 pcap_cleanup_live_common 函数，清除系统内存并完成关机操作。

通过这些步骤，代码可以有效地执行 graceful shutdown，并确保在关闭设备时，所有打开的文件描述符都得到正确处理。


```cpp
/*
 * Performs a graceful shutdown of the DAG card, frees dynamic memory held
 * in the pcap_t structure, and closes the file descriptor for the DAG card.
 */

static void
dag_platform_cleanup(pcap_t *p)
{
	struct pcap_dag *pd = p->priv;

	if(dag_stop_stream(p->fd, pd->dag_stream) < 0)
		fprintf(stderr,"dag_stop_stream: %s\n", strerror(errno));

	if(dag_detach_stream(p->fd, pd->dag_stream) < 0)
		fprintf(stderr,"dag_detach_stream: %s\n", strerror(errno));

	if(pd->dag_ref != NULL) {
		dag_config_dispose(pd->dag_ref);
		/*
		 * Note: we don't need to call close(p->fd) or
		 * dag_close(p->fd), as dag_config_dispose(pd->dag_ref)
		 * does this.
		 *
		 * Set p->fd to -1 to make sure that's not done.
		 */
		p->fd = -1;
		pd->dag_ref = NULL;
	}
	delete_pcap_dag(p);
	pcap_cleanup_live_common(p);
}

```

这段代码是一个用于处理 Pcap 数据链路层出错的函数。它的主要作用是在程序结束时对系统进行清理，并确保在程序结束时所有创建的数据链路层节点都被正确地释放。

具体来说，这段代码执行以下操作：

1. 首先，它进入一个无限循环，直到系统创建了新的数据链路层节点。

2. 对于每个数据链路层节点，它检查该节点是否是当前正在运行的进程。如果是，则调用一个名为 "dag_platform_cleanup" 的函数，确保在程序结束时清理完所有资源。如果不是，则调用 "delete_pcap_dag" 函数，将该节点从内存中删除。

3. 在循环结束后，如果仍然存在创建的数据链路层节点，函数将返回 0，否则返回 -1，表示出错。


```cpp
static void
atexit_handler(void)
{
	while (pcap_dags != NULL) {
		if (pcap_dags->pid == getpid()) {
			if (pcap_dags->p != NULL)
				dag_platform_cleanup(pcap_dags->p);
		} else {
			delete_pcap_dag(pcap_dags->p);
		}
	}
}

static int
new_pcap_dag(pcap_t *p)
{
	pcap_dag_node_t *node = NULL;

	if ((node = malloc(sizeof(pcap_dag_node_t))) == NULL) {
		return -1;
	}

	if (!atexit_handler_installed) {
		atexit(atexit_handler);
		atexit_handler_installed = 1;
	}

	node->next = pcap_dags;
	node->p = p;
	node->pid = getpid();

	pcap_dags = node;

	return 0;
}

```

这段代码是一个名为 "dag_erf_ext_header_count" 的函数，它的作用是统计 DAG 文件中的扩展头数量。

函数接受两个参数：一个指向 DAG 文件的指针 erf 和一个整数 len，表示 DAG 文件的长度。函数内部首先进行了几个基本检查，如果 erf 为 NULL 或者 len 长度小于 16 字节，函数返回 0，表示没有扩展头。

接着，函数判断了一下 DAG 文件中是否有扩展头。具体来说，首先检查 erf 数组中是否包含 8 个或以上的字节，如果是，则函数返回 0，表示没有扩展头。然后，函数每次循环 over the extension headers，检查 hdr\_type 字段是否包含 8 个或以上的字节，如果是，就说明有一个扩展头，函数就返回 hdr\_num，表示这个扩展头是由 DAG 文件中的哪个元素组成的。循环内部，函数还进行了一些基本检查，比如检查 len 是否小于扩展头的长度，以及检查 hdr\_num 是否比扩展头的长度多 1。

最后，函数返回 hdr\_num，表示 DAG 文件中扩展头的数量。


```cpp
static unsigned int
dag_erf_ext_header_count(const uint8_t *erf, size_t len)
{
	uint32_t hdr_num = 0;
	uint8_t  hdr_type;

	/* basic sanity checks */
	if ( erf == NULL )
		return 0;
	if ( len < 16 )
		return 0;

	/* check if we have any extension headers */
	if ( (erf[8] & 0x80) == 0x00 )
		return 0;

	/* loop over the extension headers */
	do {

		/* sanity check we have enough bytes */
		if ( len < (24 + (hdr_num * 8)) )
			return hdr_num;

		/* get the header type */
		hdr_type = erf[(16 + (hdr_num * 8))];
		hdr_num++;

	} while ( hdr_type & 0x80 );

	return hdr_num;
}

```

This code appears to be processing the contents of a packet on an Ethernet frame. It checks whether the packet is a color MacroHeader (CMH) frame, or an Ethernet frame that may or may not have a MacroHeader, and skips over any expansion headers.

The code first checks whether the packet's linktype is set to a value that doesn't indicate the presence of a MacroHeader, such as DLT_CHDLC or DLT_LAPD. If the linktype is not one of those values, the code continues to the next step, where it checks the order of the packet fields.

If the order isDLT_FRELAY)

If the order is DLT_CHDLC)

If the order is not DLT_FRELAY) or not DLT_CHDLC), the code skips over the first 8 bytes of the packet and then calculates the capacity of the packet based on the remaining length and the size of the MacroHeader (assuming it has a fixed size and is not a variable-length header).

The code then skips over any further expansion headers and calculates the length of the MacroHeader by subtracting the size of the first 8 bytes of the packet from the remaining length of the packet, and adjusting the length of the MacroHeader based on the size of the FCS (Extended Field Capture) section of the packet.

Finally, the code checks the position of the MacroHeader in the packet and jumps to the corresponding position in the data plane (dp) to continue processing the packet.


```cpp
/*
 *  Read at most max_packets from the capture stream and call the callback
 *  for each of them. Returns the number of packets handled, -1 if an
 *  error occurred, or -2 if we were told to break out of the loop.
 */
static int
dag_read(pcap_t *p, int cnt, pcap_handler callback, u_char *user)
{
	struct pcap_dag *pd = p->priv;
	unsigned int processed = 0;
	unsigned int nonblocking = pd->dag_flags & DAGF_NONBLOCK;
	unsigned int num_ext_hdr = 0;
	unsigned int ticks_per_second;

	/* Get the next bufferful of packets (if necessary). */
	while (pd->dag_mem_top - pd->dag_mem_bottom < dag_record_size) {

		/*
		 * Has "pcap_breakloop()" been called?
		 */
		if (p->break_loop) {
			/*
			 * Yes - clear the flag that indicates that
			 * it has, and return -2 to indicate that
			 * we were told to break out of the loop.
			 */
			p->break_loop = 0;
			return -2;
		}

		/* dag_advance_stream() will block (unless nonblock is called)
		 * until 64kB of data has accumulated.
		 * If to_ms is set, it will timeout before 64kB has accumulated.
		 * We wait for 64kB because processing a few packets at a time
		 * can cause problems at high packet rates (>200kpps) due
		 * to inefficiencies.
		 * This does mean if to_ms is not specified the capture may 'hang'
		 * for long periods if the data rate is extremely slow (<64kB/sec)
		 * If non-block is specified it will return immediately. The user
		 * is then responsible for efficiency.
		 */
		if ( NULL == (pd->dag_mem_top = dag_advance_stream(p->fd, pd->dag_stream, &(pd->dag_mem_bottom))) ) {
		     return -1;
		}

		if (nonblocking && (pd->dag_mem_top - pd->dag_mem_bottom < dag_record_size))
		{
			/* Pcap is configured to process only available packets, and there aren't any, return immediately. */
			return 0;
		}

		if(!nonblocking &&
		   pd->dag_timeout &&
		   (pd->dag_mem_top - pd->dag_mem_bottom < dag_record_size))
		{
			/* Blocking mode, but timeout set and no data has arrived, return anyway.*/
			return 0;
		}

	}

	/*
	 * Process the packets.
	 *
	 * This assumes that a single buffer of packets will have
	 * <= INT_MAX packets, so the packet count doesn't overflow.
	 */
	while (pd->dag_mem_top - pd->dag_mem_bottom >= dag_record_size) {

		unsigned short packet_len = 0;
		int caplen = 0;
		struct pcap_pkthdr	pcap_header;

		dag_record_t *header = (dag_record_t *)(pd->dag_mem_bottom);

		u_char *dp = ((u_char *)header); /* + dag_record_size; */
		unsigned short rlen;

		/*
		 * Has "pcap_breakloop()" been called?
		 */
		if (p->break_loop) {
			/*
			 * Yes - clear the flag that indicates that
			 * it has, and return -2 to indicate that
			 * we were told to break out of the loop.
			 */
			p->break_loop = 0;
			return -2;
		}

		rlen = ntohs(header->rlen);
		if (rlen < dag_record_size)
		{
			pcap_strlcpy(p->errbuf, "dag_read: record too small",
			    PCAP_ERRBUF_SIZE);
			return -1;
		}
		pd->dag_mem_bottom += rlen;

		/* Count lost packets. */
		switch((header->type & 0x7f)) {
			/* in these types the color value overwrites the lctr */
		case ERF_TYPE_COLOR_HDLC_POS:
		case ERF_TYPE_COLOR_ETH:
		case ERF_TYPE_DSM_COLOR_HDLC_POS:
		case ERF_TYPE_DSM_COLOR_ETH:
		case ERF_TYPE_COLOR_MC_HDLC_POS:
		case ERF_TYPE_COLOR_HASH_ETH:
		case ERF_TYPE_COLOR_HASH_POS:
			break;

		default:
			if ( (pd->drop_attr == kNullAttributeUuid) && (header->lctr) ) {
				pd->stat.ps_drop += ntohs(header->lctr);
			}
		}

		if ((header->type & 0x7f) == ERF_TYPE_PAD) {
			continue;
		}

		num_ext_hdr = dag_erf_ext_header_count(dp, rlen);

		/* ERF encapsulation */
		/* The Extensible Record Format is not dropped for this kind of encapsulation,
		 * and will be handled as a pseudo header by the decoding application.
		 * The information carried in the ERF header and in the optional subheader (if present)
		 * could be merged with the libpcap information, to offer a better decoding.
		 * The packet length is
		 * o the length of the packet on the link (header->wlen),
		 * o plus the length of the ERF header (dag_record_size), as the length of the
		 *   pseudo header will be adjusted during the decoding,
		 * o plus the length of the optional subheader (if present).
		 *
		 * The capture length is header.rlen and the byte stuffing for alignment will be dropped
		 * if the capture length is greater than the packet length.
		 */
		if (p->linktype == DLT_ERF) {
			packet_len = ntohs(header->wlen) + dag_record_size;
			caplen = rlen;
			switch ((header->type & 0x7f)) {
			case ERF_TYPE_MC_AAL5:
			case ERF_TYPE_MC_ATM:
			case ERF_TYPE_MC_HDLC:
			case ERF_TYPE_MC_RAW_CHANNEL:
			case ERF_TYPE_MC_RAW:
			case ERF_TYPE_MC_AAL2:
			case ERF_TYPE_COLOR_MC_HDLC_POS:
				packet_len += 4; /* MC header */
				break;

			case ERF_TYPE_COLOR_HASH_ETH:
			case ERF_TYPE_DSM_COLOR_ETH:
			case ERF_TYPE_COLOR_ETH:
			case ERF_TYPE_ETH:
				packet_len += 2; /* ETH header */
				break;
			} /* switch type */

			/* Include ERF extension headers */
			packet_len += (8 * num_ext_hdr);

			if (caplen > packet_len) {
				caplen = packet_len;
			}
		} else {
			/* Other kind of encapsulation according to the header Type */

			/* Skip over generic ERF header */
			dp += dag_record_size;
			/* Skip over extension headers */
			dp += 8 * num_ext_hdr;

			switch((header->type & 0x7f)) {
			case ERF_TYPE_ATM:
			case ERF_TYPE_AAL5:
				if ((header->type & 0x7f) == ERF_TYPE_AAL5) {
					packet_len = ntohs(header->wlen);
					caplen = rlen - dag_record_size;
				}
			case ERF_TYPE_MC_ATM:
				if ((header->type & 0x7f) == ERF_TYPE_MC_ATM) {
					caplen = packet_len = ATM_CELL_SIZE;
					dp+=4;
				}
			case ERF_TYPE_MC_AAL5:
				if ((header->type & 0x7f) == ERF_TYPE_MC_AAL5) {
					packet_len = ntohs(header->wlen);
					caplen = rlen - dag_record_size - 4;
					dp+=4;
				}
				/* Skip over extension headers */
				caplen -= (8 * num_ext_hdr);

				if ((header->type & 0x7f) == ERF_TYPE_ATM) {
					caplen = packet_len = ATM_CELL_SIZE;
				}
				if (p->linktype == DLT_SUNATM) {
					struct sunatm_hdr *sunatm = (struct sunatm_hdr *)dp;
					unsigned long rawatm;

					rawatm = ntohl(*((unsigned long *)dp));
					sunatm->vci = htons((rawatm >>  4) & 0xffff);
					sunatm->vpi = (rawatm >> 20) & 0x00ff;
					sunatm->flags = ((header->flags.iface & 1) ? 0x80 : 0x00) |
						((sunatm->vpi == 0 && sunatm->vci == htons(5)) ? 6 :
						 ((sunatm->vpi == 0 && sunatm->vci == htons(16)) ? 5 :
						  ((dp[ATM_HDR_SIZE] == 0xaa &&
						    dp[ATM_HDR_SIZE+1] == 0xaa &&
						    dp[ATM_HDR_SIZE+2] == 0x03) ? 2 : 1)));

				} else if (p->linktype == DLT_ATM_RFC1483) {
					packet_len -= ATM_HDR_SIZE;
					caplen -= ATM_HDR_SIZE;
					dp += ATM_HDR_SIZE;
				} else
					continue;
				break;

			case ERF_TYPE_COLOR_HASH_ETH:
			case ERF_TYPE_DSM_COLOR_ETH:
			case ERF_TYPE_COLOR_ETH:
			case ERF_TYPE_ETH:
				if ((p->linktype != DLT_EN10MB) &&
				    (p->linktype != DLT_DOCSIS))
					continue;
				packet_len = ntohs(header->wlen);
				packet_len -= (pd->dag_fcs_bits >> 3);
				caplen = rlen - dag_record_size - 2;
				/* Skip over extension headers */
				caplen -= (8 * num_ext_hdr);
				if (caplen > packet_len) {
					caplen = packet_len;
				}
				dp += 2;
				break;

			case ERF_TYPE_COLOR_HASH_POS:
			case ERF_TYPE_DSM_COLOR_HDLC_POS:
			case ERF_TYPE_COLOR_HDLC_POS:
			case ERF_TYPE_HDLC_POS:
				if ((p->linktype != DLT_CHDLC) &&
				    (p->linktype != DLT_PPP_SERIAL) &&
				    (p->linktype != DLT_FRELAY))
					continue;
				packet_len = ntohs(header->wlen);
				packet_len -= (pd->dag_fcs_bits >> 3);
				caplen = rlen - dag_record_size;
				/* Skip over extension headers */
				caplen -= (8 * num_ext_hdr);
				if (caplen > packet_len) {
					caplen = packet_len;
				}
				break;

			case ERF_TYPE_COLOR_MC_HDLC_POS:
			case ERF_TYPE_MC_HDLC:
				if ((p->linktype != DLT_CHDLC) &&
				    (p->linktype != DLT_PPP_SERIAL) &&
				    (p->linktype != DLT_FRELAY) &&
				    (p->linktype != DLT_MTP2) &&
				    (p->linktype != DLT_MTP2_WITH_PHDR) &&
				    (p->linktype != DLT_LAPD))
					continue;
				packet_len = ntohs(header->wlen);
				packet_len -= (pd->dag_fcs_bits >> 3);
				caplen = rlen - dag_record_size - 4;
				/* Skip over extension headers */
				caplen -= (8 * num_ext_hdr);
				if (caplen > packet_len) {
					caplen = packet_len;
				}
				/* jump the MC_HDLC_HEADER */
				dp += 4;
```

这段代码是一个名为`process_pkt_ts`的函数，属于`pd`数据平面链表的结点处理函数。它用于处理单个数据包，即从链表中取出一个数据包，对其进行处理，然后将结果返回给调用者。以下是这个函数的主要步骤：

1. 根据传入的`p`指针，解析出`ts`时间戳的精度，然后根据精度计算出每秒的时钟中断数`ticks_per_second`。
2. 根据`ts`和`ticks_per_second`计算出时间戳`ts`对应的微秒级时间`ms`级时间，然后将`ts`和`ms`级时间加上`caplen`，得到`pd_header.ts`。
3. 创建一个新的`pd_header`，其中包含处理后的数据包信息和计数器。
4. 根据数据包的长度和当前计数器，填充`pd_header`中的数据。
5. 如果当前计数器已达到用户设定的限制，则返回已处理的数据包数，否则继续处理下一个数据包。
6. 如果已经处理完所有数据包，函数结束并返回已处理的数据包数。

这个函数主要用于处理单个数据包，对数据包进行统计和处理，然后将结果返回给调用者。对于每个数据包，它需要根据传入的时间戳精度计算出每秒的时钟中断数，并根据这个值来处理数据包。对于不同的数据包，可能需要计算出不同的时钟中断数。


```cpp
#ifdef DLT_MTP2_WITH_PHDR
				if (p->linktype == DLT_MTP2_WITH_PHDR) {
					/* Add the MTP2 Pseudo Header */
					caplen += MTP2_HDR_LEN;
					packet_len += MTP2_HDR_LEN;

					TempPkt[MTP2_SENT_OFFSET] = 0;
					TempPkt[MTP2_ANNEX_A_USED_OFFSET] = MTP2_ANNEX_A_USED_UNKNOWN;
					*(TempPkt+MTP2_LINK_NUMBER_OFFSET) = ((header->rec.mc_hdlc.mc_header>>16)&0x01);
					*(TempPkt+MTP2_LINK_NUMBER_OFFSET+1) = ((header->rec.mc_hdlc.mc_header>>24)&0xff);
					memcpy(TempPkt+MTP2_HDR_LEN, dp, caplen);
					dp = TempPkt;
				}
#endif
				break;

			case ERF_TYPE_IPV4:
				if ((p->linktype != DLT_RAW) &&
				    (p->linktype != DLT_IPV4))
					continue;
				packet_len = ntohs(header->wlen);
				caplen = rlen - dag_record_size;
				/* Skip over extension headers */
				caplen -= (8 * num_ext_hdr);
				if (caplen > packet_len) {
					caplen = packet_len;
				}
				break;

			case ERF_TYPE_IPV6:
				if ((p->linktype != DLT_RAW) &&
				    (p->linktype != DLT_IPV6))
					continue;
				packet_len = ntohs(header->wlen);
				caplen = rlen - dag_record_size;
				/* Skip over extension headers */
				caplen -= (8 * num_ext_hdr);
				if (caplen > packet_len) {
					caplen = packet_len;
				}
				break;

			/* These types have no matching 'native' DLT, but can be used with DLT_ERF above */
			case ERF_TYPE_MC_RAW:
			case ERF_TYPE_MC_RAW_CHANNEL:
			case ERF_TYPE_IP_COUNTER:
			case ERF_TYPE_TCP_FLOW_COUNTER:
			case ERF_TYPE_INFINIBAND:
			case ERF_TYPE_RAW_LINK:
			case ERF_TYPE_INFINIBAND_LINK:
			default:
				/* Unhandled ERF type.
				 * Ignore rather than generating error
				 */
				continue;
			} /* switch type */

		} /* ERF encapsulation */

		if (caplen > p->snapshot)
			caplen = p->snapshot;

		/* Run the packet filter if there is one. */
		if ((p->fcode.bf_insns == NULL) || pcap_filter(p->fcode.bf_insns, dp, packet_len, caplen)) {

			/* convert between timestamp formats */
			register unsigned long long ts;

			if (IS_BIGENDIAN()) {
				ts = SWAPLL(header->ts);
			} else {
				ts = header->ts;
			}

			switch (p->opt.tstamp_precision) {
			case PCAP_TSTAMP_PRECISION_NANO:
				ticks_per_second = 1000000000;
				break;
			case PCAP_TSTAMP_PRECISION_MICRO:
			default:
				ticks_per_second = 1000000;
				break;

			}
			pcap_header.ts.tv_sec = ts >> 32;
			ts = (ts & 0xffffffffULL) * ticks_per_second;
			ts += 0x80000000; /* rounding */
			pcap_header.ts.tv_usec = ts >> 32;
			if (pcap_header.ts.tv_usec >= ticks_per_second) {
				pcap_header.ts.tv_usec -= ticks_per_second;
				pcap_header.ts.tv_sec++;
			}

			/* Fill in our own header data */
			pcap_header.caplen = caplen;
			pcap_header.len = packet_len;

			/* Count the packet. */
			pd->stat.ps_recv++;

			/* Call the user supplied callback function */
			callback(user, &pcap_header, dp);

			/* Only count packets that pass the filter, for consistency with standard Linux behaviour. */
			processed++;
			if (processed == cnt && !PACKET_COUNT_IS_UNLIMITED(cnt))
			{
				/* Reached the user-specified limit. */
				return cnt;
			}
		}
	}

	return processed;
}

```

这段代码定义了一个名为 "dag_inject" 的函数，属于 "pcap_main" 系列的函数。该函数的作用是：

1. 如果传入了空字符串 ("");
2. 如果 DAG 设备支持活包捕捉，设置捕获到的数据包大小 ("size" 参数) 并返回 0；
3. 如果 DAG 设备不支持活包捕捉，设置错误字符串 ("errbuf" 参数) 并返回 -1。
4. 如果需要支持每流统计信息，需要通过设置适当的 DAG 工具来实现。
5. 在 DAG 设备初始化之后，设置 API 抽样的参数。
6. 在函数中使用了 pcap_strlcpy 函数，将错误字符串 ("errbuf") 复制到 "errbuf" 变量中，该函数的作用是将输入的参数进行字符串转义，并将其拼接到一个新的字符数组中。


```cpp
static int
dag_inject(pcap_t *p, const void *buf _U_, int size _U_)
{
	pcap_strlcpy(p->errbuf, "Sending packets isn't supported on DAG cards",
	    PCAP_ERRBUF_SIZE);
	return (-1);
}

/*
 *  Get a handle for a live capture from the given DAG device.  Passing a NULL
 *  device will result in a failure.  The promisc flag is ignored because DAG
 *  cards are always promiscuous.  The to_ms parameter is used in setting the
 *  API polling parameters.
 *
 *  snaplen is now also ignored, until we get per-stream slen support. Set
 *  slen with appropriate DAG tool BEFORE pcap_activate().
 *
 *  See also pcap(3).
 */
```

It looks like you are trying to provide additional functionality to the Linux kernel's� 名为 "dag\_set


```cpp
static int dag_activate(pcap_t* p)
{
	struct pcap_dag *pd = p->priv;
	char *s;
	int n;
	daginf_t* daginf;
	char * newDev = NULL;
	char * device = p->opt.device;
	int ret;
	dag_size_t mindata;
	struct timeval maxwait;
	struct timeval poll;

	if (device == NULL) {
		snprintf(p->errbuf, PCAP_ERRBUF_SIZE, "device is NULL");
		return PCAP_ERROR;
	}

	/* Initialize some components of the pcap structure. */
	newDev = (char *)malloc(strlen(device) + 16);
	if (newDev == NULL) {
		ret = PCAP_ERROR;
		pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
		    errno, "Can't allocate string for device name");
		goto fail;
	}

	/* Parse input name to get dag device and stream number if provided */
	if (dag_parse_name(device, newDev, strlen(device) + 16, &pd->dag_stream) < 0) {
		/*
		 * XXX - it'd be nice if this indicated what was wrong
		 * with the name.  Does this reliably set errno?
		 * Should this return PCAP_ERROR_NO_SUCH_DEVICE in some
		 * cases?
		 */
		ret = PCAP_ERROR;
		pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
		    errno, "dag_parse_name");
		goto fail;
	}
	device = newDev;

	if (pd->dag_stream%2) {
		ret = PCAP_ERROR;
		snprintf(p->errbuf, PCAP_ERRBUF_SIZE, "dag_parse_name: tx (even numbered) streams not supported for capture");
		goto fail;
	}

	/* setup device parameters */
	if((pd->dag_ref = dag_config_init((char *)device)) == NULL) {
		/*
		 * XXX - does this reliably set errno?
		 */
		if (errno == ENOENT) {
			/*
			 * There's nothing more to say, so clear
			 * the error message.
			 */
			ret = PCAP_ERROR_NO_SUCH_DEVICE;
			p->errbuf[0] = '\0';
		} else if (errno == EPERM || errno == EACCES) {
			ret = PCAP_ERROR_PERM_DENIED;
			snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
			    "Attempt to open %s failed with %s - additional privileges may be required",
			    device, (errno == EPERM) ? "EPERM" : "EACCES");
		} else {
			ret = PCAP_ERROR;
			pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
			    errno, "dag_config_init %s", device);
		}
		goto fail;
	}

	if((p->fd = dag_config_get_card_fd(pd->dag_ref)) < 0) {
		/*
		 * XXX - does this reliably set errno?
		 */
		ret = PCAP_ERROR;
		pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
		    errno, "dag_config_get_card_fd %s", device);
		goto failclose;
	}

	/* Open requested stream. Can fail if already locked or on error */
	if (dag_attach_stream64(p->fd, pd->dag_stream, 0, 0) < 0) {
		ret = PCAP_ERROR;
		pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
		    errno, "dag_attach_stream");
		goto failclose;
	}

	/* Try to find Stream Drop attribute */
	pd->drop_attr = kNullAttributeUuid;
	pd->dag_root = dag_config_get_root_component(pd->dag_ref);
	if ( dag_component_get_subcomponent(pd->dag_root, kComponentStreamFeatures, 0) )
	{
		pd->drop_attr = dag_config_get_indexed_attribute_uuid(pd->dag_ref, kUint32AttributeStreamDropCount, pd->dag_stream/2);
	}

	/* Set up default poll parameters for stream
	 * Can be overridden by pcap_set_nonblock()
	 */
	if (dag_get_stream_poll64(p->fd, pd->dag_stream,
				&mindata, &maxwait, &poll) < 0) {
		ret = PCAP_ERROR;
		pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
		    errno, "dag_get_stream_poll");
		goto faildetach;
	}

	/* Use the poll time as the required select timeout for callers
	 * who are using select()/etc. in an event loop waiting for
	 * packets to arrive.
	 */
	pd->required_select_timeout = poll;
	p->required_select_timeout = &pd->required_select_timeout;

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

	if (p->opt.immediate) {
		/* Call callback immediately.
		 * XXX - is this the right way to p this?
		 */
		mindata = 0;
	} else {
		/* Amount of data to collect in Bytes before calling callbacks.
		 * Important for efficiency, but can introduce latency
		 * at low packet rates if to_ms not set!
		 */
		mindata = 65536;
	}

	/* Obey opt.timeout (was to_ms) if supplied. This is a good idea!
	 * Recommend 10-100ms. Calls will time out even if no data arrived.
	 */
	maxwait.tv_sec = p->opt.timeout/1000;
	maxwait.tv_usec = (p->opt.timeout%1000) * 1000;

	if (dag_set_stream_poll64(p->fd, pd->dag_stream,
				mindata, &maxwait, &poll) < 0) {
		ret = PCAP_ERROR;
		pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
		    errno, "dag_set_stream_poll");
		goto faildetach;
	}

        /* XXX Not calling dag_configure() to set slen; this is unsafe in
	 * multi-stream environments as the gpp config is global.
         * Once the firmware provides 'per-stream slen' this can be supported
	 * again via the Config API without side-effects */
```

This is a Linux implementation of the `new_pcap_dag()` function, which is used to create a data-link device (DAG) in the network偏移方向上。


```cpp
#if 0
	/* set the card snap length to the specified snaplen parameter */
	/* This is a really bad idea, as different cards have different
	 * valid slen ranges. Should fix in Config API. */
	if (p->snapshot == 0 || p->snapshot > MAX_DAG_SNAPLEN) {
		p->snapshot = MAX_DAG_SNAPLEN;
	} else if (snaplen < MIN_DAG_SNAPLEN) {
		p->snapshot = MIN_DAG_SNAPLEN;
	}
	/* snap len has to be a multiple of 4 */
#endif

	if(dag_start_stream(p->fd, pd->dag_stream) < 0) {
		ret = PCAP_ERROR;
		pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
		    errno, "dag_start_stream %s", device);
		goto faildetach;
	}

	/*
	 * Important! You have to ensure bottom is properly
	 * initialized to zero on startup, it won't give you
	 * a compiler warning if you make this mistake!
	 */
	pd->dag_mem_bottom = 0;
	pd->dag_mem_top = 0;

	/*
	 * Find out how many FCS bits we should strip.
	 * First, query the card to see if it strips the FCS.
	 */
	daginf = dag_info(p->fd);
	if ((0x4200 == daginf->device_code) || (0x4230 == daginf->device_code))	{
		/* DAG 4.2S and 4.23S already strip the FCS.  Stripping the final word again truncates the packet. */
		pd->dag_fcs_bits = 0;

		/* Note that no FCS will be supplied. */
		p->linktype_ext = LT_FCS_DATALINK_EXT(0);
	} else {
		/*
		 * Start out assuming it's 32 bits.
		 */
		pd->dag_fcs_bits = 32;

		/* Allow an environment variable to override. */
		if ((s = getenv("ERF_FCS_BITS")) != NULL) {
			if ((n = atoi(s)) == 0 || n == 16 || n == 32) {
				pd->dag_fcs_bits = n;
			} else {
				ret = PCAP_ERROR;
				snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
					"pcap_activate %s: bad ERF_FCS_BITS value (%d) in environment", device, n);
				goto failstop;
			}
		}

		/*
		 * Did the user request that they not be stripped?
		 */
		if ((s = getenv("ERF_DONT_STRIP_FCS")) != NULL) {
			/* Yes.  Note the number of 16-bit words that will be
			   supplied. */
			p->linktype_ext = LT_FCS_DATALINK_EXT(pd->dag_fcs_bits/16);

			/* And don't strip them. */
			pd->dag_fcs_bits = 0;
		}
	}

	pd->dag_timeout	= p->opt.timeout;

	p->linktype = -1;
	if (dag_get_datalink(p) < 0) {
		ret = PCAP_ERROR;
		goto failstop;
	}

	p->bufsize = 0;

	if (new_pcap_dag(p) < 0) {
		ret = PCAP_ERROR;
		pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
		    errno, "new_pcap_dag %s", device);
		goto failstop;
	}

	/*
	 * "select()" and "poll()" don't work on DAG device descriptors.
	 */
	p->selectable_fd = -1;

	if (newDev != NULL) {
		free((char *)newDev);
	}

	p->read_op = dag_read;
	p->inject_op = dag_inject;
	p->setfilter_op = install_bpf_program;
	p->setdirection_op = NULL; /* Not implemented.*/
	p->set_datalink_op = dag_set_datalink;
	p->getnonblock_op = pcap_getnonblock_fd;
	p->setnonblock_op = dag_setnonblock;
	p->stats_op = dag_stats;
	p->cleanup_op = dag_platform_cleanup;
	pd->stat.ps_drop = 0;
	pd->stat.ps_recv = 0;
	pd->stat.ps_ifdrop = 0;
	return 0;

```

这三段代码都是用于处理网络调试信息中的 DAG (Data and Event Log) 流。在这里，我们看到了三个不同的函数，它们分别用于检查和关闭 DAG 流，以及报告错误和标准输出。

1. dag_stop_stream(p->fd, pd->dag_stream) 是一个用于关闭 DAG 流的函数。它接受两个参数，一个是 DAG 文件的描述符(通常是一个文件名)，另一个是 DAG 文件的描述符(通常是 DAG 文件的下一个事件的时间戳)。该函数返回一个负值，如果关闭 DAG 流操作失败，则使用 stderr 输出错误信息。

2. dag_detach_stream(p->fd, pd->dag_stream) 是一个用于尝试关闭 DAG 流的函数。它与 dag_stop_stream 类似，但它的尝试关闭操作不会对后续的事件产生影响。该函数同样接受两个参数，一个是 DAG 文件的描述符(通常是一个文件名)，另一个是 DAG 文件的描述符(通常是 DAG 文件的下一个事件的时间戳)。该函数返回一个负值，如果尝试关闭 DAG 流操作失败，则使用 stderr 输出错误信息。

3. failclose 是这三个函数的主函数。它负责关闭已经创建的 DAG 文件，并释放相关的资源。它包括以下操作：

  a. 使用 dag_config_dispose(pd->dag_ref) 函数关闭 DAG 根。
  b. 使用 close(p->fd) 函数关闭 DAG 文件描述符。
  c. 设置 p->fd 为 -1，以防止已经在 DAG 文件中创建的文件描述符被更改。
  d. 释放之前创建的 DAG 文件和相关的资源。

这三段代码的作用是确保 DAG 文件可以被安全地关闭，并尽量确保不会影响正在进行的 DAG 操作。


```cpp
failstop:
	if (dag_stop_stream(p->fd, pd->dag_stream) < 0) {
		fprintf(stderr,"dag_stop_stream: %s\n", strerror(errno));
	}

faildetach:
	if (dag_detach_stream(p->fd, pd->dag_stream) < 0)
		fprintf(stderr,"dag_detach_stream: %s\n", strerror(errno));

failclose:
	dag_config_dispose(pd->dag_ref);
	/*
	 * Note: we don't need to call close(p->fd) or dag_close(p->fd),
	 * as dag_config_dispose(pd->dag_ref) does this.
	 *
	 * Set p->fd to -1 to make sure that's not done.
	 */
	p->fd = -1;
	pd->dag_ref = NULL;
	delete_pcap_dag(p);

```

This function appears to be a part of a network protocol library, and it is responsible for checking whether a given DAG (data and event) stream is one of our own or from another remote devices. It takes in a DAG stream, and checks various fields within the stream to determine


```cpp
fail:
	pcap_cleanup_live_common(p);
	if (newDev != NULL) {
		free((char *)newDev);
	}

	return ret;
}

pcap_t *dag_create(const char *device, char *ebuf, int *is_ours)
{
	const char *cp;
	char *cpend;
	long devnum;
	pcap_t *p;
	long stream = 0;

	/* Does this look like a DAG device? */
	cp = strrchr(device, '/');
	if (cp == NULL)
		cp = device;
	/* Does it begin with "dag"? */
	if (strncmp(cp, "dag", 3) != 0) {
		/* Nope, doesn't begin with "dag" */
		*is_ours = 0;
		return NULL;
	}
	/* Yes - is "dag" followed by a number from 0 to DAG_MAX_BOARDS-1 */
	cp += 3;
	devnum = strtol(cp, &cpend, 10);
	if (*cpend == ':') {
		/* Followed by a stream number. */
		stream = strtol(++cpend, &cpend, 10);
	}

	if (cpend == cp || *cpend != '\0') {
		/* Not followed by a number. */
		*is_ours = 0;
		return NULL;
	}

	if (devnum < 0 || devnum >= DAG_MAX_BOARDS) {
		/* Followed by a non-valid number. */
		*is_ours = 0;
		return NULL;
	}

	if (stream <0 || stream >= DAG_STREAM_MAX) {
		/* Followed by a non-valid stream number. */
		*is_ours = 0;
		return NULL;
	}

	/* OK, it's probably ours. */
	*is_ours = 1;

	p = PCAP_CREATE_COMMON(ebuf, struct pcap_dag);
	if (p == NULL)
		return NULL;

	p->activate_op = dag_activate;

	/*
	 * We claim that we support microsecond and nanosecond time
	 * stamps.
	 *
	 * XXX Our native precision is 2^-32s, but libpcap doesn't support
	 * power of two precisions yet. We can convert to either MICRO or NANO.
	 */
	p->tstamp_precision_list = malloc(2 * sizeof(u_int));
	if (p->tstamp_precision_list == NULL) {
		pcap_fmt_errmsg_for_errno(ebuf, PCAP_ERRBUF_SIZE,
		    errno, "malloc");
		pcap_close(p);
		return NULL;
	}
	p->tstamp_precision_list[0] = PCAP_TSTAMP_PRECISION_MICRO;
	p->tstamp_precision_list[1] = PCAP_TSTAMP_PRECISION_NANO;
	p->tstamp_precision_count = 2;
	return p;
}

```

这段代码是一个名为 `dag_stats` 的函数，属于 `pcap_main` 类的成员函数。它的作用是获取数据包传输过程中的统计信息，并将统计信息存储到 `struct pcap_stat` 类型的变量 `ps` 中。以下是该函数的详细解释：

1. 函数参数说明：
  - `pcap_t` 指针变量 `p` 指向的下一个数据包。
  - `struct pcap_stat` 类型的变量 `ps` 存储了统计信息的结果。
  
2. 函数实现：
  - 在函数开始时，首先创建一个名为 `pd` 的 `struct pcap_dag` 类型的变量，该变量存储了当前数据包的 `drop_attr` 属性的值。
  - 接着定义了一个 `uint32_t` 类型的变量 `stream_drop`，用于统计数据包在传输过程中被丢弃的数量。
  - 定义了一个 `dag_err_t` 类型的变量 `dag_error`，用于统计数据包传输过程中的错误情况。
  - 如果 `pd->drop_attr` 属性存在，则执行以下操作：
      - 如果 `ps_drop` 统计信息存在，则将 `stream_drop` 的值赋给 `pd->stat.ps_drop`。
      - 否则，将 `dag_error` 错误信息打印到 `pcap_errbuf` 中，以便用户查看。
  - 最后，将 `ps` 统计信息存储到 `ps` 变量中，并返回 0。

3. 函数来源：

该函数属于 `pcap_main` 类的成员函数，因此在 `pcap_main` 类的源文件中定义。


```cpp
static int
dag_stats(pcap_t *p, struct pcap_stat *ps) {
	struct pcap_dag *pd = p->priv;
	uint32_t stream_drop;
	dag_err_t dag_error;

	/*
	 * Packet records received (ps_recv) are counted in dag_read().
	 * Packet records dropped (ps_drop) are read from Stream Drop attribute if present,
	 * otherwise integrate the ERF Header lctr counts (if available) in dag_read().
	 * We are reporting that no records are dropped by the card/driver (ps_ifdrop).
	 */

	if(pd->drop_attr != kNullAttributeUuid) {
		/* Note this counter is cleared at start of capture and will wrap at UINT_MAX.
		 * The application is responsible for polling ps_drop frequently enough
		 * to detect each wrap and integrate total drop with a wider counter */
		if ((dag_error = dag_config_get_uint32_attribute_ex(pd->dag_ref, pd->drop_attr, &stream_drop)) == kDagErrNone) {
			pd->stat.ps_drop = stream_drop;
		} else {
			snprintf(p->errbuf, PCAP_ERRBUF_SIZE, "reading stream drop attribute: %s",
				 dag_config_strerror(dag_error));
			return -1;
		}
	}

	*ps = pd->stat;

	return 0;
}

```

This function appears to be part of the Linux kernel's Drivers昏界面驱动程序，用于初始化 and配置 以太网网卡。它的主要作用是检测连接到网络的设备，并将其信息添加到 `/proc/net/dev` 文件中。以下是该函数的主要步骤：

1. 获取设备驱动程序的设备名称。
2. 检查设备是否已配置为网络接口。
3. 如果设备已配置为网络接口，则检查连接到该接口的网络是否已配置好。
4. 检查设备是否已连接到网络。
5. 如果设备已连接到网络，则尝试卸载设备并重新配置它。
6. 如果设备无法卸载或重新配置，则返回错误并退出。
7. 如果设备成功卸载或重新配置，则返回 0。

以上是该函数的主要步骤，但具体情况可能会有所不同。由于缺乏上下文和具体实现，无法提供更多信息。


```cpp
/*
 * Add all DAG devices.
 */
int
dag_findalldevs(pcap_if_list_t *devlistp, char *errbuf)
{
	char name[12];	/* XXX - pick a size */
	int c;
	char dagname[DAGNAME_BUFSIZE];
	int dagstream;
	int dagfd;
	dag_card_inf_t *inf;
	char *description;
	int stream, rxstreams;

	/* Try all the DAGs 0-DAG_MAX_BOARDS */
	for (c = 0; c < DAG_MAX_BOARDS; c++) {
		snprintf(name, 12, "dag%d", c);
		if (-1 == dag_parse_name(name, dagname, DAGNAME_BUFSIZE, &dagstream))
		{
			(void) snprintf(errbuf, PCAP_ERRBUF_SIZE,
			    "dag: device name %s can't be parsed", name);
			return (-1);
		}
		if ( (dagfd = dag_open(dagname)) >= 0 ) {
			description = NULL;
			if ((inf = dag_pciinfo(dagfd)))
				description = dag_device_name(inf->device_code, 1);
			/*
			 * XXX - is there a way to determine whether
			 * the card is plugged into a network or not?
			 * If so, we should check that and set
			 * PCAP_IF_CONNECTION_STATUS_CONNECTED or
			 * PCAP_IF_CONNECTION_STATUS_DISCONNECTED.
			 *
			 * Also, are there notions of "up" and "running"?
			 */
			if (add_dev(devlistp, name, 0, description, errbuf) == NULL) {
				/*
				 * Failure.
				 */
				return (-1);
			}
			rxstreams = dag_rx_get_stream_count(dagfd);
			for(stream=0;stream<DAG_STREAM_MAX;stream+=2) {
				if (0 == dag_attach_stream64(dagfd, stream, 0, 0)) {
					dag_detach_stream(dagfd, stream);

					snprintf(name,  10, "dag%d:%d", c, stream);
					if (add_dev(devlistp, name, 0, description, errbuf) == NULL) {
						/*
						 * Failure.
						 */
						return (-1);
					}

					rxstreams--;
					if(rxstreams <= 0) {
						break;
					}
				}
			}
			dag_close(dagfd);
		}

	}
	return (0);
}

```

0;

This function appears to regulate the non-blocking mode of a TCP port using the `PCAP` (Point-to-Console Access Protocol) library.

It first sets the non-blocking mode using the `PCAP_SET Nonblocking` function, and then calls the `dag_get_stream_poll64` function to receive data from the port.

If the `dag_get_stream_poll64` function returns an error, the function will return an error code. If it succeeds, it will return a non-blocking mode flag (`DAGF_NONBLOCK`) that can be set on the `pd->dag_flags` field to return the callbacks as non-blocking.

Note that the function also checks if `nonblock` is set, if it's not set it will return `0`, if it is set it will return `1`.

It is important to mention that this function may introduce latency and should be used in a environment where low latency is required.


```cpp
static int
dag_set_datalink(pcap_t *p, int dlt)
{
	p->linktype = dlt;

	return (0);
}

static int
dag_setnonblock(pcap_t *p, int nonblock)
{
	struct pcap_dag *pd = p->priv;
	dag_size_t mindata;
	struct timeval maxwait;
	struct timeval poll;

	/*
	 * Set non-blocking mode on the FD.
	 * XXX - is that necessary?  If not, don't bother calling it,
	 * and have a "dag_getnonblock()" function that looks at
	 * "pd->dag_flags".
	 */
	if (pcap_setnonblock_fd(p, nonblock) < 0)
		return (-1);

	if (dag_get_stream_poll64(p->fd, pd->dag_stream,
				&mindata, &maxwait, &poll) < 0) {
		pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
		    errno, "dag_get_stream_poll");
		return -1;
	}

	/* Amount of data to collect in Bytes before calling callbacks.
	 * Important for efficiency, but can introduce latency
	 * at low packet rates if to_ms not set!
	 */
	if(nonblock)
		mindata = 0;
	else
		mindata = 65536;

	if (dag_set_stream_poll64(p->fd, pd->dag_stream,
				mindata, &maxwait, &poll) < 0) {
		pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
		    errno, "dag_set_stream_poll");
		return -1;
	}

	if (nonblock) {
		pd->dag_flags |= DAGF_NONBLOCK;
	} else {
		pd->dag_flags &= ~DAGF_NONBLOCK;
	}
	return (0);
}

```

这段代码是一个用于获取数据链路层（Datalink）头部信息的函数，它接受一个数据链路层输出包（pcap_t）和一个数据链路层地址转换器（pcap_dag）变量。函数内部先定义了一个长度为256的types数组，然后将数组长度设为256，以便于将所有可能的数据链路层类型罗列出来。接下来，函数使用memset函数将types数组中的所有元素都置为0。然后，函数检查输入的pcap_dag变量是否为空，如果是，则错误地将errno设置为"malloc"。在if语句的后面，函数根据pcap_dag变量的值来设置linktype变量。函数最终返回0，表示成功获取数据链路层头部信息。


```cpp
static int
dag_get_datalink(pcap_t *p)
{
	struct pcap_dag *pd = p->priv;
	int index=0, dlt_index=0;
	uint8_t types[255];

	memset(types, 0, 255);

	if (p->dlt_list == NULL && (p->dlt_list = malloc(255*sizeof(*(p->dlt_list)))) == NULL) {
		pcap_fmt_errmsg_for_errno(p->errbuf, sizeof(p->errbuf),
		    errno, "malloc");
		return (-1);
	}

	p->linktype = 0;

```

这段代码的作用是检查服务器是否支持某种类型的 DAG 数据流。它通过调用 `dag_get_stream_erf_types` 和 `dag_get_erf_types` 函数来获取一个列表中所有可能的 ERF 类型。如果这些函数返回的错误码不是 success，那么代码会输出错误并返回 -1。如果两个函数都返回成功，那么代码会遍历可能的 ERF 类型，并对每个类型输出一条消息。

这里的作用相当于 ffmpeg 命令行工具中的 `-核对支类型` 选项，它会尝试从指定的文件描述符中读取 DAG 数据流，并检查服务器是否支持指定的 ERF 类型。如果指定的 ERF 类型不存在，或者读取过程中发生错误，代码会输出错误并返回 -1。如果 ERF 类型正确，那么代码会继续读取并处理 DAG 数据流。


```cpp
#ifdef HAVE_DAG_GET_STREAM_ERF_TYPES
	/* Get list of possible ERF types for this card */
	if (dag_get_stream_erf_types(p->fd, pd->dag_stream, types, 255) < 0) {
		pcap_fmt_errmsg_for_errno(p->errbuf, sizeof(p->errbuf),
		    errno, "dag_get_stream_erf_types");
		return (-1);
	}

	while (types[index]) {

#elif defined HAVE_DAG_GET_ERF_TYPES
	/* Get list of possible ERF types for this card */
	if (dag_get_erf_types(p->fd, types, 255) < 0) {
		pcap_fmt_errmsg_for_errno(p->errbuf, sizeof(p->errbuf),
		    errno, "dag_get_erf_types");
		return (-1);
	}

	while (types[index]) {
```

This is a function definition for an `ethernet_role` structure that is used to manage the LinkedIn Learning Taxonomy (LLT) framework in erb毫不犹豫地支持通过携带IPv6数据包。它处理多种数据包类型，并在其中插入LLT标签。在switch类型的数据包中，插入标签后会根据所插入的标签选择相应的DLT类型。

首先，它检查输入的`p`结构中是否有`linktype`成员。如果没有，它将`p->linktype`设置为`DLT_RAW`。然后，它检查`p->dlt_list`是否为`NULL`。如果是，它将`p->dlt_list`中所有`DLT_RAW`和`DLT_IPV4`项都设置为`DLT_RAW`。如果是，则创建一个包含`DLT_RAW`和`DLT_IPV4`的`DLT_LIST`，并将其添加到`p->dlt_list`中。接下来，它检查`p->linktype`是否为`NULL`。如果是，它将`p->linktype`设置为`DLT_RAW`。然后，它根据所插入的标签选择相应的DLT类型，并更新`p->dlt_list`中`DLT_INDEX_`和`DLT_COUNT_`成员的值。最后，它返回`p->linktype`，作为结果。


```cpp
#else
	/* Check the type through a dagapi call. */
	types[index] = dag_linktype(p->fd);

	{
#endif
		switch((types[index] & 0x7f)) {

		case ERF_TYPE_HDLC_POS:
		case ERF_TYPE_COLOR_HDLC_POS:
		case ERF_TYPE_DSM_COLOR_HDLC_POS:
		case ERF_TYPE_COLOR_HASH_POS:

			if (p->dlt_list != NULL) {
				p->dlt_list[dlt_index++] = DLT_CHDLC;
				p->dlt_list[dlt_index++] = DLT_PPP_SERIAL;
				p->dlt_list[dlt_index++] = DLT_FRELAY;
			}
			if(!p->linktype)
				p->linktype = DLT_CHDLC;
			break;

		case ERF_TYPE_ETH:
		case ERF_TYPE_COLOR_ETH:
		case ERF_TYPE_DSM_COLOR_ETH:
		case ERF_TYPE_COLOR_HASH_ETH:
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
			if (p->dlt_list != NULL) {
				p->dlt_list[dlt_index++] = DLT_EN10MB;
				p->dlt_list[dlt_index++] = DLT_DOCSIS;
			}
			if(!p->linktype)
				p->linktype = DLT_EN10MB;
			break;

		case ERF_TYPE_ATM:
		case ERF_TYPE_AAL5:
		case ERF_TYPE_MC_ATM:
		case ERF_TYPE_MC_AAL5:
			if (p->dlt_list != NULL) {
				p->dlt_list[dlt_index++] = DLT_ATM_RFC1483;
				p->dlt_list[dlt_index++] = DLT_SUNATM;
			}
			if(!p->linktype)
				p->linktype = DLT_ATM_RFC1483;
			break;

		case ERF_TYPE_COLOR_MC_HDLC_POS:
		case ERF_TYPE_MC_HDLC:
			if (p->dlt_list != NULL) {
				p->dlt_list[dlt_index++] = DLT_CHDLC;
				p->dlt_list[dlt_index++] = DLT_PPP_SERIAL;
				p->dlt_list[dlt_index++] = DLT_FRELAY;
				p->dlt_list[dlt_index++] = DLT_MTP2;
				p->dlt_list[dlt_index++] = DLT_MTP2_WITH_PHDR;
				p->dlt_list[dlt_index++] = DLT_LAPD;
			}
			if(!p->linktype)
				p->linktype = DLT_CHDLC;
			break;

		case ERF_TYPE_IPV4:
			if (p->dlt_list != NULL) {
				p->dlt_list[dlt_index++] = DLT_RAW;
				p->dlt_list[dlt_index++] = DLT_IPV4;
			}
			if(!p->linktype)
				p->linktype = DLT_RAW;
			break;

		case ERF_TYPE_IPV6:
			if (p->dlt_list != NULL) {
				p->dlt_list[dlt_index++] = DLT_RAW;
				p->dlt_list[dlt_index++] = DLT_IPV6;
			}
			if(!p->linktype)
				p->linktype = DLT_RAW;
			break;

		case ERF_TYPE_LEGACY:
		case ERF_TYPE_MC_RAW:
		case ERF_TYPE_MC_RAW_CHANNEL:
		case ERF_TYPE_IP_COUNTER:
		case ERF_TYPE_TCP_FLOW_COUNTER:
		case ERF_TYPE_INFINIBAND:
		case ERF_TYPE_RAW_LINK:
		case ERF_TYPE_INFINIBAND_LINK:
		case ERF_TYPE_META:
		default:
			/* Libpcap cannot deal with these types yet */
			/* Add no 'native' DLTs, but still covered by DLT_ERF */
			break;

		} /* switch */
		index++;
	}

	p->dlt_list[dlt_index++] = DLT_ERF;

	p->dlt_count = dlt_index;

	if(!p->linktype)
		p->linktype = DLT_ERF;

	return p->linktype;
}

```

这段代码是一个用于指定如何加载名为“dag_only”的库的示例。它通过检查当前系统是否支持名为“DAG”的库来选择是否加载它。如果系统不支持该库，则返回0。

具体来说，该代码包括两个函数：

1. `pcap_platform_finddevs()`函数用于返回如何在系统中查找DAG接口。它有一个参数`devlistp`，它是一个指向`pcap_if_list_t`类型的指针，用于存储接口列表。该函数使用`_U_`参数来表示该函数的实参将是一个无参数的类型。然后，它返回0，表示没有找到DAG接口。

2. `int`定义了一个名为`errbuf`的参数，用于存储在加载库时可能产生的错误字符串。该函数将使用`pcap_get_error_str()`函数来获取错误字符串，并将其存储在`errbuf`中。然后，它将返回0，表示成功加载库，并将错误字符串留空。


```cpp
#ifdef DAG_ONLY
/*
 * This libpcap build supports only DAG cards, not regular network
 * interfaces.
 */

/*
 * There are no regular interfaces, just DAG interfaces.
 */
int
pcap_platform_finddevs(pcap_if_list_t *devlistp _U_, char *errbuf)
{
	return (0);
}

```

pcap_version = "0.9.6";

This code defines two functions within the `libpcap` library:

1. `pcap_create_interface`：该函数尝试打开一个 regular（非 DAG）interface（接口），如果失败，将在错误缓冲区（errbuf）中打印错误信息。然后返回 NULL，表示没有成功创建该接口。

函数原型定义在头文件中：
```cppperl
pcap_t * pcap_create_interface(const char *device, char *errbuf);
```
1. `pcap_version`：该函数返回 libpcap 的版本字符串。在函数内部，没有其他函数或变量与该版本字符串相关联。


```cpp
/*
 * Attempts to open a regular interface fail.
 */
pcap_t *
pcap_create_interface(const char *device, char *errbuf)
{
	snprintf(errbuf, PCAP_ERRBUF_SIZE,
	    "This version of libpcap only supports DAG cards");
	return NULL;
}

/*
 * Libpcap version string.
 */
const char *
```

这段代码是一个 preprocessed header 函数，它的目的是定义一个函数，用于在源代码文件中包含一些额外的信息。

具体来说，这段代码定义了一个名为 "pcap_lib_version" 的函数，它的实现比较简单，直接将 "PCAP_VERSION_STRING" 和 "(DAG-only)" 字符串拼接在一起，然后以 "PCAP_LIB_VERSION" 的形式返回。

这个函数的作用是，在源代码文件中定义 PCAP_LIB_VERSION 函数时，会根据这个函数的实现结果来编译、链接、加载 PCAP 库，从而正确的支持 DAG(数据链路树)协议栈的使用。由于这个函数是在 PCAP 库之前定义的，因此它只能用于 PCAP 库的版本兼容性检查等。


```cpp
pcap_lib_version(void)
{
	return (PCAP_VERSION_STRING " (DAG-only)");
}
#endif

```