# Nmap源码解析 53

# `libpcap/pcap-dos.c`

这段代码是一个名为 `pcap-dos.c` 的 C 语言源文件，它属于 DOS-libpcap 项目。这个文件被编译成了 DOS/DOSX，用于实现对 PKTDRVR、NDIS2 和 32 位 pmode 网络驱动的接口。

具体来说，这段代码提供的功能包括：

1. 定义了一些头文件：`<stdio.h>`、`<stdlib.h>`、`<string.h>` 和 `<signal.h>`，用于提供标准输入输出、标准库函数以及字符串和信号相关的定义。

2. 引入了一些标准库：`<fcntl.h>` 和 `<limits.h>`，用于提供文件 I/O 操作和数值相关的定义。

3. 实现了一个名为 `pcap_dos_write` 的函数，用于向指定文件输出 DNS 查询结果。这个函数接收一个 `struct sockaddr_in` 类型的结构体，其中包含目标服务器 IP 地址和端口号。

4. 实现了一个名为 `pcap_dos_read` 的函数，用于从指定文件中读取 DNS 查询结果。这个函数同样接收一个 `struct sockaddr_in` 类型的结构体，其中包含目标服务器 IP 地址和端口号。

由于这段代码与信号处理或者网络相关，因此没有提供更多的信息。


```cpp
/*
 *  This file is part of DOS-libpcap
 *  Ported to DOS/DOSX by G. Vanem <gvanem@yahoo.no>
 *
 *  pcap-dos.c: Interface to PKTDRVR, NDIS2 and 32-bit pmode
 *              network drivers.
 */

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <signal.h>
#include <float.h>
#include <fcntl.h>
#include <limits.h> /* for INT_MAX */
```

这段代码的作用是定义了一个名为 "msdos.h" 的头文件，它包含了使用32位驱动程序所需的所有头文件和函数。它定义了一些与 MSDOS 系统相关的头文件和函数，例如：32位 MSDOS 设备驱动程序，PCI 驱动程序，BIOS32 驱动程序，32位设备驱动程序的 32 位和 64 位版本的模块，以及 32位和 64位 AS/MAX 类型。

这个头文件是用于在32位 MSDOS 系统上支持32位和 64位设备驱动程序的必要依赖项。


```cpp
#include <io.h>

#if defined(USE_32BIT_DRIVERS)
  #include "msdos/pm_drvr/pmdrvr.h"
  #include "msdos/pm_drvr/pci.h"
  #include "msdos/pm_drvr/bios32.h"
  #include "msdos/pm_drvr/module.h"
  #include "msdos/pm_drvr/3c501.h"
  #include "msdos/pm_drvr/3c503.h"
  #include "msdos/pm_drvr/3c509.h"
  #include "msdos/pm_drvr/3c59x.h"
  #include "msdos/pm_drvr/3c515.h"
  #include "msdos/pm_drvr/3c90x.h"
  #include "msdos/pm_drvr/3c575_cb.h"
  #include "msdos/pm_drvr/ne.h"
  #include "msdos/pm_drvr/wd.h"
  #include "msdos/pm_drvr/accton.h"
  #include "msdos/pm_drvr/cs89x0.h"
  #include "msdos/pm_drvr/rtl8139.h"
  #include "msdos/pm_drvr/ne2k-pci.h"
```



该代码是一个用于 Linux 系统的 pcap-related 头文件。它包含了一些用于 ndis2-packet 数据报驱动程序的头部定义。

首先，它定义了一个名为 "USE_NDIS2" 的宏，如果该宏定义为 true，则包含以下代码：

```cpp
#include "msdos/ndis2.h"
```

这将包含ndis2库头文件，该库用于管理 NDIS 套接字。

然后，它定义了一个名为 "NDIS2" 的宏，如果该宏定义为 true，则包含以下代码：

```cpp
#include "msdos/ndis2.h"
```

这将包含ndis2库头文件，该库用于管理 NDIS 套接字。

接下来，该代码定义了一个名为 "pcap" 的宏，如果该宏定义为 true，则包含以下代码：

```cpp
#include "pcap.h"
```

这将包含用于 pcap 头文件的定义。

然后，该代码定义了一个名为 "pcap-dos" 的宏，如果该宏定义为 true，则包含以下代码：

```cpp
#include "pcap-dos.h"
```

这将包含用于 pcap-dos 头文件的定义。

接下来，该代码定义了一个名为 "pcap-int" 的宏，如果该宏定义为 true，则包含以下代码：

```cpp
#include "pcap-int.h"
```

这将包含用于 pcap-int 头文件的定义。

然后，该代码定义了一个名为 "pktdrvr" 的宏，如果该宏定义为 true，则包含以下代码：

```cpp
#include "msdos/pktdrvr.h"
```

这将包含用于 pktdrvr 头文件的定义。

最后，该代码通过 #ifdef 语句检查是否启用了 NDIS2，如果是，那么就在前面包含上述头文件。否则，就会直接包含 pcap 头文件。

总结起来，该代码定义了一些用于 Linux 系统的 pcap 相关的头文件，其中一些是必要的库头文件，其他的是用于ndis2-packet 数据报驱动程序的头部定义。


```cpp
#endif

#include "pcap.h"
#include "pcap-dos.h"
#include "pcap-int.h"
#include "msdos/pktdrvr.h"

#ifdef USE_NDIS2
#include "msdos/ndis2.h"
#endif

#include <arpa/inet.h>
#include <net/if.h>
#include <net/if_arp.h>
#include <net/if_ether.h>
```

这段代码是一个 Linux 系统调用，它定义了一些用于无线网络卡(RTL8139)的代码。

该代码的作用是实现了一个简单的链路层协议，允许通过它传输数据到无线网络卡。以下是主要步骤：

1. 引入必要的头文件和定义常量。
2. 如果定义了使用32位处理器，那么定义了一个FLUSHK()函数，用于在发送数据之前清空队列。
3. 定义了一个NDIS_NEXT_DEV常量，用于存储下一个开发板的设备号。
4. 定义了一个名为pktq_init的函数，用于初始化接收队列，包括设置接收队列的大小、数组和当前队列元素。
5. 定义了一个名为pktq_check的函数，用于检查队列是否为空或已满，并返回状态。
6. 定义了一个名为pktq_inc_out的函数，用于将元素添加到输出队列中。
7. 定义了一个名为pktq_in_index的函数，用于获取当前正在接收数据的元素在数组中的索引。
8. 定义了一个名为pktq_clear的函数，用于清除接收队列。
9. 定义了一个名为pktq_in_elem的函数，用于返回正在等待接收的元素。
10. 定义了一个名为pktq_out_elem的函数，用于返回正在发送的元素。

这些函数的具体实现可能因不同的硬件和驱动程序而有所不同。


```cpp
#include <net/if_packe.h>
#include <tcp.h>

#if defined(USE_32BIT_DRIVERS)
  #define FLUSHK()       do { _printk_safe = 1; _printk_flush(); } while (0)
  #define NDIS_NEXT_DEV  &rtl8139_dev

  static char *rx_pool = NULL;
  static void init_32bit (void);

  static int  pktq_init     (struct rx_ringbuf *q, int size, int num, char *pool);
  static int  pktq_check    (struct rx_ringbuf *q);
  static int  pktq_inc_out  (struct rx_ringbuf *q);
  static int  pktq_in_index (struct rx_ringbuf *q) LOCKED_FUNC;
  static void pktq_clear    (struct rx_ringbuf *q) LOCKED_FUNC;

  static struct rx_elem *pktq_in_elem  (struct rx_ringbuf *q) LOCKED_FUNC;
  static struct rx_elem *pktq_out_elem (struct rx_ringbuf *q);

```

这段代码是一个C语言代码片段，它定义了一些宏和内部变量。接下来我会逐步解释这段代码的作用。

首先，我们看到了一个带else的#define定义。这个else子句表示，如果当前代码点不是第一个定义，那么后面的定义都不会被执行。

FLUSHK()是一个内部函数，它的作用是返回一个void类型的0，或者说是返回一个空指针。这个函数在代码的后面部分被调用，并在函数内部进行了定义。

NDIS_NEXT_DEV是一个宏定义，它定义了一个名为NDIS_NEXT_DEV的内部变量。这个变量的作用是在系统启动时，从指定的设备（NDIS）自动获取IP地址，并将其设置为设备的默认网关。

接下来，我们看到了一些内部变量的定义。这些变量将在代码的后续部分被使用。

WATT_DO_EXIT是一个内部函数，它的作用是执行WATT32软件的卸载操作。这个函数将在系统启动时执行，并在函数内部被定义。

WATT_IS_INIT是一个内部函数，它的作用是检查WATT32软件是否处于初始化状态。这个函数将在系统启动时执行，并在函数内部被定义。

W32_DYNAMIC_HOST是一个内部变量，它的作用是表示当前系统的动态主机。这个变量将在系统启动时被设置为默认值。

W32_DO_MASK_REQUEST是一个内部函数，它的作用是设置软件接口上发送数据包时的目标MASK地址。这个函数将在系统启动时被定义。

_W32_USR_POST_INIT是一个内部函数，它的作用是执行WATT32软件的用户初始化操作。这个函数将在系统启动时执行，并在函数内部被定义。

最后，我们看到了一些函数指针的定义。这些函数指针将在系统启动时被绑定到相应的函数上，从而允许我们在系统启动时执行相应的操作。

总之，这段代码定义了一些内部变量和函数，用于初始化WATT32软件的硬件和设置系统启动参数。这些函数和变量将在系统启动时被执行，并在函数内部被定义。


```cpp
#else
  #define FLUSHK()      ((void)0)
  #define NDIS_NEXT_DEV  NULL
#endif

/*
 * Internal variables/functions in Watt-32
 */
extern WORD  _pktdevclass;
extern BOOL  _eth_is_init;
extern int   _w32_dynamic_host;
extern int   _watt_do_exit;
extern int   _watt_is_init;
extern int   _w32__bootp_on, _w32__dhcp_on, _w32__rarp_on, _w32__do_mask_req;
extern void (*_w32_usr_post_init) (void);
```

这段代码定义了三个外部函数和三个外部变量，以及一个静态变量。具体解释如下：

1. `extern void (*_w32_print_hook)()` 是一个指向函数指针的函数指针，它定义了一个内部函数 `_w32_print_hook()`，但这个函数指针本身没有实际的内容，它需要在被调用的函数中实现具体的内容。由于这个函数指针没有被使用，因此它的实现内容未知。

2. `extern void dbug_write (const char *filename)` 是一个函数，它接受一个字符串参数 `filename`，并将其写入到指定文件中。由于这个函数没有定义具体的内容，因此它的实现内容未知。

3. `extern int pkt_get_mtu (void)` 是一个函数，它返回一个整数 `pkt_get_mtu()`，它用于获取数据帧的最大传输单元(MTU)长度。由于这个函数没有定义具体的内容，因此它的实现内容未知。

4. `static int ref_count = 0;` 是一个静态变量，它用于记录数据帧的引用计数。每当数据帧被引用时，计数器 `ref_count` 会增加 1，并将其值递增。

5. `static u_long mac_count    = 0;` 是一个静态变量，它用于记录网络接口的 MAC 地址数量。

6. `static u_long filter_count = 0;` 是一个静态变量，它用于记录数据过滤器的数量。

7. `static volatile BOOL exc_occured = 0;` 是一个静态变量，它用于记录数据帧是否已经被捕获到。

8. `static struct device *handle_to_device [20];` 是一个静态变量，它用于存储数据帧的目标设备号。

9. `static int  pcap_activate_dos (pcap_t *p)` 是一个内部函数，它接受一个指向 `pcap_t` 结构数的指针 `p`，并将其激活。它的实现内容未知。

由于这段代码比较短小，而且其中大多数函数和变量都是未定义的，因此具体的实现内容无法确定。


```cpp
extern void (*_w32_print_hook)();

extern void dbug_write (const char *);  /* Watt-32 lib, pcdbug.c */
extern int  pkt_get_mtu (void);

static int ref_count = 0;

static u_long mac_count    = 0;
static u_long filter_count = 0;

static volatile BOOL exc_occured = 0;

static struct device *handle_to_device [20];

static int  pcap_activate_dos (pcap_t *p);
```



这段代码是一个名为 `pcap_read_dos` 的函数，它是 `pcap` 控制器的其中一个回调函数。它的作用是读取数据包中的 DNS 报文。

具体来说，这个函数接收一个 `pcap` 指向、计数器 `cnt` 和一个 `pcap_handler` 类型的函数指针，以及一个数据缓冲区 `data`。它将遍历数据缓冲区中的所有数据包，对于每个数据包，调用传递给它的 `callback` 函数，将数据包中的 DNS 报文传递给它。

这里值得注意的是，它使用了 `ndis_probe` 函数来检测是否支持网络接口。如果检测到支持网络接口，它将尝试使用 `pkt_probe` 函数读取数据包。

另外，它还定义了一个名为 `pcap_cleanup_dos` 的函数，用于释放由 `pcap_read_dos` 函数分配的资源。


```cpp
static int  pcap_read_dos (pcap_t *p, int cnt, pcap_handler callback,
                           u_char *data);
static void pcap_cleanup_dos (pcap_t *p);
static int  pcap_stats_dos (pcap_t *p, struct pcap_stat *ps);
static int  pcap_sendpacket_dos (pcap_t *p, const void *buf, size_t len);
static int  pcap_setfilter_dos (pcap_t *p, struct bpf_program *fp);

static int  ndis_probe (struct device *dev);
static int  pkt_probe  (struct device *dev);

static void close_driver (void);
static int  init_watt32 (struct pcap *pcap, const char *dev_name, char *err_buf);
static int  first_init (const char *name, char *ebuf, int promisc);

static void watt32_recv_hook (u_char *dummy, const struct pcap_pkthdr *pcap,
                              const u_char *buf);

```

这段代码定义了两个结构体，ndis_dev和pkts_dev,ndis_dev表示一个设备，pkts_dev表示一个传输设备(如网卡)。

ndis_dev的结构体定义了ndis设备的名称、制造商、序列号、以及NDIS驱动程序的下一个设备类型。它还定义了一个probe成员，用于支持设备的探测和配置。

pkts_dev的结构体定义了pkts设备的名称、制造商、序列号以及它所依赖的ndis设备的类型。它还定义了一个probe成员，用于支持传输设备的探测和配置。

这两个结构体都包含了ndis和pkts_device类型，它们在代码中使用，可以用来控制设备的操作。


```cpp
/*
 * These are the device we always support
 */
static struct device ndis_dev = {
              "ndis",
              "NDIS2 LanManager",
              0,
              0,0,0,0,0,0,
              NDIS_NEXT_DEV,  /* NULL or a 32-bit device */
              ndis_probe
            };

static struct device pkt_dev = {
              "pkt",
              "Packet-Driver",
              0,
              0,0,0,0,0,0,
              &ndis_dev,
              pkt_probe
            };

```

这段代码定义了一个名为 `get_device` 的函数，用于获取一个整套设备(device)的句柄(handle)。函数的第一个参数是一个整套设备文件描述符(fd)，用于指定要查找的设备。

首先，函数检查传入的fd是否小于或等于零，如果是，则返回 NULL，表明设备不存在。接着，函数检查传入的fd是否大于 `sizeof(handle_to_device)/sizeof(handle_to_device[0])`，如果是，则也返回 NULL。这里，`handle_to_device` 是一个数组，用于存储 handle 到 device 的映射。如果fd不在这个数组中，函数将返回 NULL。

如果fd在 `handle_to_device` 数组中，函数将返回 handle_to_device数组的第二个元素，即 device 的句柄。

该函数仅在 `pcap_dos` 结构体中声明了一个名为 `wait_proc` 的函数指针，但并没有定义 `pcap_dos` 结构体。


```cpp
static struct device *get_device (int fd)
{
  if (fd <= 0 || fd >= sizeof(handle_to_device)/sizeof(handle_to_device[0]))
     return (NULL);
  return handle_to_device [fd-1];
}

/*
 * Private data for capturing on MS-DOS.
 */
struct pcap_dos {
	void (*wait_proc)(void); /* call proc while waiting */
	struct pcap_stat stat;
};

```

这段代码定义了一个名为`pcap_create_interface`的函数，它接受两个参数：一个表示网络接口名称的`const char`类型和一个指向字符数组的`char`类型。它的作用是创建一个名为`device_name`的设备，并返回一个指向该设备的`pcap_t`类型的指针。

具体来说，函数首先通过调用`PCAP_CREATE_COMMON`函数来创建一个名为`device_name`的设备对象。这个函数接受两个参数：一个是要分配给设备的内存区域，另一个是一个指向结构体`struct pcap_dos`的指针。如果分配内存失败，函数将返回一个`NULL`类型的指针。

接下来，函数通过给设备对象的`activate_op`字段设置为`pcap_activate_dos`，来开启设备对象的Live Capture功能。这样，当调用`pcap_write`函数时，就可以开始捕获网络数据包了。

最后，函数返回一个指向设备对象的`pcap_t`类型的指针。这个指针可以用来对设备对象进行操作，例如捕获数据包、设置捕获过滤规则等。


```cpp
pcap_t *pcap_create_interface (const char *device _U_, char *ebuf)
{
	pcap_t *p;

	p = PCAP_CREATE_COMMON(ebuf, struct pcap_dos);
	if (p == NULL)
		return (NULL);

	p->activate_op = pcap_activate_dos;
	return (p);
}

/*
 * Open MAC-driver with name 'device_name' for live capture of
 * network packets.
 */
```



This is a code snippet for a Linux kernel module that creates a new packet capture device. The device is assigned the net device name specified by the `pcap->opt.device` parameter, and the driver is started with the `PCAP_OPEN` option.

The `PCAP_DESCRIPTION` macro is used to construct a a description of the captured packets. The fifth argument of this macro sets the maximum transmission unit (MTU) of the packets, while the fourth argument is a pointer to an array of `SNPROBBS` structures, each representing a packet's attributes.

It appears that there is a bug in this code, as the `snprintf` function is causing a segmentation fault (SF) on the last call to `pcap_setfilter_dos`. This is because the `PCAP_SETFILTER_DOS` function is using the `setfilter_index` function, which has a 0-based index and a `PCAP_FILTER_INDEX` parameter, which specifies the index of the filter to apply. However, the `PCAP_SETFILTER_DOS` function is using the `index` parameter instead, which is a non-integer type.

A fix to this bug would be to update the code to correctly handle the `PCAP_SETFILTER_DOS` function, either by using the `setfilter_index` function as the `PCAP_SETFILTER_DOS` function, or by updating the code to correctly handle the `PCAP_FILTER_INDEX` parameter.


```cpp
static int pcap_activate_dos (pcap_t *pcap)
{
  if (pcap->opt.rfmon) {
    /*
     * No monitor mode on DOS.
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
  if (pcap->snapshot <= 0 || pcap->snapshot > MAXIMUM_SNAPLEN)
    pcap->snapshot = MAXIMUM_SNAPLEN;

  if (pcap->snapshot < ETH_MIN+8)
      pcap->snapshot = ETH_MIN+8;

  if (pcap->snapshot > ETH_MAX)   /* silently accept and truncate large MTUs */
      pcap->snapshot = ETH_MAX;

  pcap->linktype          = DLT_EN10MB;  /* !! */
  pcap->cleanup_op        = pcap_cleanup_dos;
  pcap->read_op           = pcap_read_dos;
  pcap->stats_op          = pcap_stats_dos;
  pcap->inject_op         = pcap_sendpacket_dos;
  pcap->setfilter_op      = pcap_setfilter_dos;
  pcap->setdirection_op   = NULL;  /* Not implemented.*/
  pcap->fd                = ++ref_count;

  pcap->bufsize = ETH_MAX+100;     /* add some margin */
  pcap->buffer = calloc (pcap->bufsize, 1);

  if (pcap->fd == 1)  /* first time we're called */
  {
    if (!init_watt32(pcap, pcap->opt.device, pcap->errbuf) ||
        !first_init(pcap->opt.device, pcap->errbuf, pcap->opt.promisc))
    {
      /* XXX - free pcap->buffer? */
      return (PCAP_ERROR);
    }
    atexit (close_driver);
  }
  else if (stricmp(active_dev->name,pcap->opt.device))
  {
    snprintf (pcap->errbuf, PCAP_ERRBUF_SIZE,
                   "Cannot use different devices simultaneously "
                   "(`%s' vs. `%s')", active_dev->name, pcap->opt.device);
    /* XXX - free pcap->buffer? */
    return (PCAP_ERROR);
  }
  handle_to_device [pcap->fd-1] = active_dev;
  return (0);
}

```

首先，我们需要明确这个问题的背景和上下文。这个问题似乎与Linux内核中的无线网络模块（如watt32_recv_hook和pcap_recv_hook）以及时间戳（time_of_day）函数有关。然而，从问题描述中我们无法得出确切的结论。因此，我们需要根据所给信息进行推断。

根据问题描述，我们可以得出以下信息：

1. 在接收到一个数据包时，会执行FLUSHK()函数，这意味着我们已经成功接收到了一个数据包。
2. pcap.len（数据包长度）和pcap.caplen（数据包和接收者缓冲区长度）之和应该等于rx_len（接收者缓冲区长度），否则会导致错误。
3. 在处理接收到的数据包时，我们需要执行以下操作：
a. 使用pcap.ts（接收者 timestamp，当前时间戳）计算时间戳。
b. 根据callback函数，我们需要向其传递数据和接收者缓冲区。
c. 如果callback函数包含time_of_day函数，那么它将计算当前时间与当前时间戳之间的时差。
d. 如果我们需要让循环重新开始，我们需要调用pcap_breakloop()函数。

根据这些信息，我们可以得出以下推断：

1. 如果当前接收者缓冲区为空，那么我们仍然需要执行时间戳计算，即使没有数据包已接收。这是因为我们需要确保在循环中我们有足够的时间来接收数据包。
2. 如果我们需要让循环重新开始，我们可以调用pcap_breakloop()函数。但是，在实际情况下，这似乎不是我们通常需要执行的操作。
3. 由于我们无法确定是否需要执行time_of_day函数，因此我们不能确保当收到数据包时，就立即执行该函数。

综上所述，我们无法从所给信息中得出确切的结论。然而，我们可以推断出一些可能需要考虑的因素。


```cpp
/*
 * Poll the receiver queue and call the pcap callback-handler
 * with the packet.
 */
static int
pcap_read_one (pcap_t *p, pcap_handler callback, u_char *data)
{
  struct pcap_dos *pd = p->priv;
  struct pcap_pkthdr pcap;
  struct timeval     now, expiry = { 0,0 };
  int    rx_len = 0;

  if (p->opt.timeout > 0)
  {
    gettimeofday2 (&now, NULL);
    expiry.tv_usec = now.tv_usec + 1000UL * p->opt.timeout;
    expiry.tv_sec  = now.tv_sec;
    while (expiry.tv_usec >= 1000000L)
    {
      expiry.tv_usec -= 1000000L;
      expiry.tv_sec++;
    }
  }

  while (!exc_occured)
  {
    volatile struct device *dev; /* might be reset by sig_handler */

    dev = get_device (p->fd);
    if (!dev)
       break;

    PCAP_ASSERT (dev->copy_rx_buf || dev->peek_rx_buf);
    FLUSHK();

    /* If driver has a zero-copy receive facility, peek at the queue,
     * filter it, do the callback and release the buffer.
     */
    if (dev->peek_rx_buf)
    {
      PCAP_ASSERT (dev->release_rx_buf);
      rx_len = (*dev->peek_rx_buf) (&p->buffer);
    }
    else
    {
      rx_len = (*dev->copy_rx_buf) (p->buffer, p->snapshot);
    }

    if (rx_len > 0)  /* got a packet */
    {
      mac_count++;

      FLUSHK();

      pcap.caplen = min (rx_len, p->snapshot);
      pcap.len    = rx_len;

      if (callback &&
          (!p->fcode.bf_insns || pcap_filter(p->fcode.bf_insns, p->buffer, pcap.len, pcap.caplen)))
      {
        filter_count++;

        /* Fix-me!! Should be time of arrival. Not time of
         * capture.
         */
        gettimeofday2 (&pcap.ts, NULL);
        (*callback) (data, &pcap, p->buffer);
      }

      if (dev->release_rx_buf)
        (*dev->release_rx_buf) (p->buffer);

      if (pcap_pkt_debug > 0)
      {
        if (callback == watt32_recv_hook)
             dbug_write ("pcap_recv_hook\n");
        else dbug_write ("pcap_read_op\n");
      }
      FLUSHK();
      return (1);
    }

    /* Has "pcap_breakloop()" been called?
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

    /* If not to wait for a packet or pcap_cleanup_dos() called from
     * e.g. SIGINT handler, exit loop now.
     */
    if (p->opt.timeout <= 0 || (volatile int)p->fd <= 0)
       break;

    gettimeofday2 (&now, NULL);

    if (timercmp(&now, &expiry, >))
       break;

```

这段代码是一个嵌入式系统的驱动程序，主要作用是处理网络数据包接收过程中的错误情况。

具体来说，代码包括以下几个部分：

1. `#ifndef DJGPP` 是限速保护函数，防止程序过快地烧毁芯片。
2. `kbhit()` 是一个真正的 CPU 开销函数，用于模拟 CPU 繁忙的情况，以触发限速保护。
3. `if (pd->wait_proc)` 是一个条件分支，判断是否调用 PD 的 `wait_proc()` 函数。
4. `(*pd->wait_proc)()` 调用 PD 的 `wait_proc()` 函数，该函数可能会阻塞 PD 的等待过程，以防止挂起。
5. `if (rx_len < 0)` 是一个条件分支，判断是否接收到了错误的网络数据包。如果是，代码执行以下操作：
  1. `pd->stat.ps_drop` 会递增，表示出现了一次丢包事件。
  2. 如果使用的是 32 位驱动器，代码会打印出 `pkt-err` 错误信息。
  3. 返回一个负的值，表示发生了错误，以便系统调用接口能够正常返回。

6. `return (0);` 表示如果前面的条件为真，则返回 0，否则返回一个负的值。


```cpp
#ifndef DJGPP
    kbhit();    /* a real CPU hog */
#endif

    if (pd->wait_proc)
      (*pd->wait_proc)();     /* call yield func */
  }

  if (rx_len < 0)            /* receive error */
  {
    pd->stat.ps_drop++;
#ifdef USE_32BIT_DRIVERS
    if (pcap_pkt_debug > 1)
       printk ("pkt-err %s\n", pktInfo.error);
#endif
    return (-1);
  }
  return (0);
}

```

这段代码是一个用于从PCAP文件中读取DOS攻击包的函数。函数名为`pcap_read_dos`，它接受一个PCAP句柄`p`，表示读取到的数据包数据，以及一个计数器`cnt`，表示当前处理的数据包数量。函数也接受一个DOS攻击数据缓冲区`data`。

函数的主要作用是读取PCAP文件中的DOS攻击数据包。函数首先检查计数器`cnt`是否已经达到了最大值，如果是，则将计数器`cnt`设置为`INT_MAX`，表示已经处理完了所有的数据包，然后函数跳出循环。

函数内部，首先判断输入的文件描述符`p->fd`是否为0，如果是，则表示文件已经被关闭或者没有打开，会导致函数调用失败或者无法读取数据包，此时函数返回一个负数。

如果`p->fd`仍然打开并且可以读取数据包，则函数调用`pcap_read_one`函数将第一个数据包读取到缓冲区`data`，此时`num`计数器会增加1。如果`p->fd`仍然打开并且可以读取数据包，但是已经读取到了字符`'\0'`，此时`num`计数器会减1，因为字符结束标记会被当作一个数据包处理。

函数内部还调用了一个`_w32_os_yield`函数，这个函数是在Windows 95/NT下防止SIGINT中断的函数。它的作用是暂停当前进程的执行，并让系统产生一个SIGINT中断，使得计算机不会响应其他程序的键盘输入，从而避免恶意攻击者利用键盘输入来发送更多的数据包。


```cpp
static int
pcap_read_dos (pcap_t *p, int cnt, pcap_handler callback, u_char *data)
{
  int rc, num = 0;

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
  if (PACKET_COUNT_IS_UNLIMITED(cnt))
    cnt = INT_MAX;

  while (num <= cnt)
  {
    if (p->fd <= 0)
       return (-1);
    rc = pcap_read_one (p, callback, data);
    if (rc > 0)
       num++;
    if (rc < 0)
       break;
    _w32_os_yield();  /* allow SIGINT generation, yield to Win95/NT */
  }
  return (num);
}

```

这段代码是一个名为 pcap_stats_dos 的函数，用于返回网络统计信息。它接受一个 pcap_t 类型的数据结构和一个结构体数组 ps，其中 ps 是一个包含网络统计信息的结构体。

函数首先检查是否有一个网络设备，如果没有，就返回一个错误信息。然后，它检查是否有一种统计信息可用，如果是，就获取统计信息并将其存储到 ps 结构中。最后，函数检查是否有一个统计信息可用，如果是，就将其存储到 ps 结构中。函数的返回值是 0，因为在 pcap_stats_dos 函数中，所有统计信息都已正确存储。


```cpp
/*
 * Return network statistics
 */
static int pcap_stats_dos (pcap_t *p, struct pcap_stat *ps)
{
  struct net_device_stats *stats;
  struct pcap_dos         *pd;
  struct device           *dev = p ? get_device(p->fd) : NULL;

  if (!dev)
  {
    strcpy (p->errbuf, "illegal pcap handle");
    return (-1);
  }

  if (!dev->get_stats || (stats = (*dev->get_stats)(dev)) == NULL)
  {
    strcpy (p->errbuf, "device statistics not available");
    return (-1);
  }

  FLUSHK();

  pd = p->priv;
  pd->stat.ps_recv   = stats->rx_packets;
  pd->stat.ps_drop  += stats->rx_missed_errors;
  pd->stat.ps_ifdrop = stats->rx_dropped +  /* queue full */
                         stats->rx_errors;    /* HW errors */
  if (ps)
     *ps = pd->stat;

  return (0);
}

```

这段代码是一个用于获取网络和设备统计信息的函数，它接收一个名为 pcap 的 pcap 结构体和一个名为 se 的结构体。

首先，它通过调用 pcap 结构体中的函数 get_device() 获取设备对象，如果该函数返回 NULL，则将 errbuf 设置为错误消息。然后，它检查设备对象是否存在，并且设备是否具有详细的统计信息。如果不存在或设备不具备详细统计信息，则将 errbuf 设置为错误消息并返回 -1。

接下来，如果设备名称为 "pktdrvr"，则函数将 errbuf 设置为错误消息，因为该设备没有详细的统计信息。然后，它调用设备对象的一个名为 get_stats() 的函数，并将从该函数中返回的统计信息复制到 se 结构体中。最后，它返回 0，表示函数成功执行并返回了详细的数据统计信息。


```cpp
/*
 * Return detailed network/device statistics.
 * May be called after 'dev->close' is called.
 */
int pcap_stats_ex (pcap_t *p, struct pcap_stat_ex *se)
{
  struct device *dev = p ? get_device (p->fd) : NULL;

  if (!dev || !dev->get_stats)
  {
    pcap_strlcpy (p->errbuf, "detailed device statistics not available",
             PCAP_ERRBUF_SIZE);
    return (-1);
  }

  if (!strnicmp(dev->name,"pkt",3))
  {
    pcap_strlcpy (p->errbuf, "pktdrvr doesn't have detailed statistics",
             PCAP_ERRBUF_SIZE);
    return (-1);
  }
  memcpy (se, (*dev->get_stats)(dev), sizeof(*se));
  return (0);
}

```

这段代码定义了一个名为 `pcap_setfilter_dos` 的函数，其作用是 simply 存储 `pcap_read_dos()` 回调函数的过滤代码。

函数的参数包括两个：`pcap_t` 类型的指针 `p` 和一个 `bpf_program` 类型的结构体 `fp`，其中 `fp` 是一个用于指定 `pcap_read_dos()` 回调函数的代码。

函数首先检查 `p` 是否为空，如果是，则返回一个负数，表示无法设置过滤代码。否则，函数将 `fp` 的值存储到 `p` 的 `fcode` 字段中，然后返回 0。

由于 `pcap_setfilter_dos()` 函数没有对输入参数 `fp` 做任何验证，因此它可能会被滥用，例如将恶意代码作为合法数据发送到设备。为了确保安全，您应该在实际应用中对其进行适当的验证和过滤。


```cpp
/*
 * Simply store the filter-code for the pcap_read_dos() callback
 * Some day the filter-code could be handed down to the active
 * device (pkt_rx1.s or 32-bit device interrupt handler).
 */
static int pcap_setfilter_dos (pcap_t *p, struct bpf_program *fp)
{
  if (!p)
     return (-1);
  p->fcode = *fp;
  return (0);
}

/*
 * Return # of packets received in pcap_read_dos()
 */
```

以下是这两段代码的作用及其它相关代码的示例：

1. 第一个函数 `pcap_mac_packets` 的作用是返回通过 `pcap_read_mac` 函数过滤后的数据包数量。`mac_count` 代表通过 `pcap_read_mac` 函数接收到的数据包中 MAC 头部信息（即 MAC 地址）的数量。

2. 第二个函数 `pcap_filter_packets` 的作用是返回经过 `pcap_filter_all` 函数处理后的数据包数量。`filter_count` 代表经过 `pcap_filter_all` 函数处理后剩余的数据包数量。这些数据包仍然包含了数据，但它们已经不再包含 MAC 头部信息，因此它们的 MAC 数（即 `mac_count`）将为 0。

3. `pcap_read_mac` 函数的作用是读取一个 MAC 数据包的 MAC 地址。它需要传给一个 `void` 类型的函数指针，这个指针就是上面定义的 `mac_count`。函数的实现如下：
```cppc
int pcap_read_mac(void *mac_data, u_int *mac_len, u_int8_t **frame_index);
```
4. `pcap_filter_all` 函数的作用是处理所有的数据包，无论它们是否经过 `pcap_filter_mac` 函数处理。`filter_count` 代表所有经过 `pcap_filter_all` 函数处理过的数据包数量。函数的实现如下：
```cppc
int pcap_filter_all(void);
```
5. `pcap_write_资本符`（即 `void`）是一个无实际功能的函数指针，它在 `pcap_write` 函数中使用。这个指针在 `pcap_write_vectors` 和 `pcap_write_any` 函数中使用，但并不影响它们的正常工作。


```cpp
u_long pcap_mac_packets (void)
{
  return (mac_count);
}

/*
 * Return # of packets passed through filter in pcap_read_dos()
 */
u_long pcap_filter_packets (void)
{
  return (filter_count);
}

/*
 * Close pcap device. Not called for offline captures.
 */
```



这段代码是用于 pcap-lib 库中的 pcap_cleanup_dos 函数，用于在 pcap 数据包 capture 成功完成后进行清理操作。

具体来说，代码会执行以下操作：

1. 如果当前函数是在捕获成功后执行的，则执行以下操作：

  a. 获取当前 pcap 数据包 capture 的驱动设备文件描述符 (fd)。

  b. 获取当前 pcap 数据包 capture 的统计信息，并打印出来。如果统计信息小于 0，则表示没有捕获到数据包，需要忽略。

  c. 获取当前数据包的设备驱动程序指针 (pd)。

  d. 如果当前设备驱动程序指针为 NULL，则将设备统计信息设置为零，表示没有设备接收到数据包。

  e. 获取当前数据包的统计信息，并将其保存到 pcap_stats() 函数的 ps_drop 成员中。

  f. 如果当前数据包的设备驱动程序指针为当前 pcap 数据包 capture 的驱动设备文件描述符 (fd)，则关闭驱动程序。

  如果当前函数是在捕获失败后执行的，则执行以下操作：

  a. 关闭当前数据包的驱动程序指针 (pd)。

  b. 关闭当前数据包的统计信息，将其设置为 0。

  c. 如果当前数据包的设备驱动程序指针为当前 pcap 数据包 capture 的驱动设备文件描述符 (fd)，则不需要执行其他操作，直接返回。

此外，代码中还有一些注释，指出该函数在 pcap_lib 库中的作用是“在 pcap 数据包 capture 成功完成后执行一系列清理操作”，以及说明该函数需要传递一个 pcap_t 类型的参数。


```cpp
static void pcap_cleanup_dos (pcap_t *p)
{
  struct pcap_dos *pd;

  if (!exc_occured)
  {
    pd = p->priv;
    if (pcap_stats(p,NULL) < 0)
       pd->stat.ps_drop = 0;
    if (!get_device(p->fd))
       return;

    handle_to_device [p->fd-1] = NULL;
    p->fd = 0;
    if (ref_count > 0)
        ref_count--;
    if (ref_count > 0)
       return;
  }
  close_driver();
  /* XXX - call pcap_cleanup_live_common? */
}

```

这段代码是一个用于获取数据包传输接口名称的函数，它接收一个字符型缓冲区（`ebuf`）。如果缓冲区中包含接口名称，则返回该接口的名称；否则返回一个空字符串（`NULL`）。

以下是代码的功能解释：

1. 定义函数：首先定义一个名为 `pcap_lookupdev` 的函数，参数为 `ebuf`。
2. 判断类型：在函数声明前使用 `static` 修饰符，说明该函数仅在定义时可见，从而避免在运行时产生不可预测的行为。
3. 初始化32位系统：调用 `init_32bit()` 函数，初始化32位系统。
4. 循环遍历：使用一个循环，从 `dev_base` 开始，遍历所有可探测的设备。对于每个设备，首先获取其驱动程序，然后执行设备探测操作。
5. 判断设备：如果设备探测成功，说明设备支持数据包传输，获取设备名称并将其存储在 `probed_dev` 指向的内存中，然后返回该设备的名称。
6. 空字符串：如果所有设备都没有探测到，将字符串 `"No driver found"` 存储在 `ebuf` 中，返回 `NULL`。

最后，函数调用者需要在调用时传入一个字符型缓冲区，用于保存接口名称，函数返回该缓冲区。


```cpp
/*
 * Return the name of the 1st network interface,
 * or NULL if none can be found.
 */
char *pcap_lookupdev (char *ebuf)
{
  struct device *dev;

#ifdef USE_32BIT_DRIVERS
  init_32bit();
#endif

  for (dev = (struct device*)dev_base; dev; dev = dev->next)
  {
    PCAP_ASSERT (dev->probe);

    if ((*dev->probe)(dev))
    {
      FLUSHK();
      probed_dev = (struct device*) dev; /* remember last probed device */
      return (char*) dev->name;
    }
  }

  if (ebuf)
     strcpy (ebuf, "No driver found");
  return (NULL);
}

```

这段代码的作用是获取设备（通过 `device` 参数）的本地网络地址和子网掩码，并将它们存储在 `localnet` 和 `netmask` 变量中。

首先，它检查 `_watt_is_init` 是否为真。如果是，那么它会尝试调用 `pcap_open_offline` 和 `pcap_activate`，但如果它们没有成功，它会将错误信息存储在 `errbuf` 变量中并返回 -1。

如果 `_watt_is_init` 为假，它将执行以下操作：

1. 初始化 `sin_mask` 为 0x1。
2. 获取目标网络地址。
3. 如果目标网络地址是 0，那么它将被转换为 IN_CLASSA_NET 类型。否则，它将被转换为 IN_CLASB_NET 类型。
4. 如果目标网络地址是 IN_CLASSC_NET，那么它将被转换为 IN_CLASSC_NET。否则，它会错误地输出并返回 -1。
5. 将 `localnet` 和 `netmask` 转换为无符号整数并存储到 `localnet` 和 `netmask` 变量中。
6. 如果 `device` 参数被正确传递，该代码将返回 0。


```cpp
/*
 * Gets localnet & netmask from Watt-32.
 */
int pcap_lookupnet (const char *device, bpf_u_int32 *localnet,
                    bpf_u_int32 *netmask, char *errbuf)
{
  DWORD mask, net;

  if (!_watt_is_init)
  {
    strcpy (errbuf, "pcap_open_offline() or pcap_activate() must be "
                    "called first");
    return (-1);
  }

  mask  = _w32_sin_mask;
  net = my_ip_addr & mask;
  if (net == 0)
  {
    if (IN_CLASSA(*netmask))
       net = IN_CLASSA_NET;
    else if (IN_CLASSB(*netmask))
       net = IN_CLASSB_NET;
    else if (IN_CLASSC(*netmask))
       net = IN_CLASSC_NET;
    else
    {
      snprintf (errbuf, PCAP_ERRBUF_SIZE, "inet class for 0x%lx unknown", mask);
      return (-1);
    }
  }
  *localnet = htonl (net);
  *netmask = htonl (mask);

  ARGSUSED (device);
  return (0);
}

```



这段代码的作用是获取在当前操作系统中可用的网卡接口列表，并将它们的名称存储在一个pcap_if_list_t结构中，如果错误则返回-1，否则返回0。

该代码的基本思路是遍历一个名为dev的设备树节点，对于每个设备节点，首先检查其是否可用（即probe是否成功），如果是，则执行关闭操作（close），然后进一步检查当前设备是否与网络连接，如果是，则设置PCAP_IF_CONNECTION_STATUS_CONNECTED或PCAP_IF_CONNECTION_STATUS_DISCONNECTED，最后如果新设备被找到，设置found为1。

如果当前设备不可用，则可能会返回-1，需要在if_list_p结构中设置errbuf参数来处理错误情况。


```cpp
/*
 * Get a list of all interfaces that are present and that we probe okay.
 * Returns -1 on error, 0 otherwise.
 * The list may be NULL empty if no interfaces were up and could be opened.
 */
int pcap_platform_finddevs  (pcap_if_list_t *devlistp, char *errbuf)
{
  struct device     *dev;
  pcap_if_t *curdev;
#if 0   /* Pkt drivers should have no addresses */
  struct sockaddr_in sa_ll_1, sa_ll_2;
  struct sockaddr   *addr, *netmask, *broadaddr, *dstaddr;
#endif
  int       ret = 0;
  int       found = 0;

  for (dev = (struct device*)dev_base; dev; dev = dev->next)
  {
    PCAP_ASSERT (dev->probe);

    if (!(*dev->probe)(dev))
       continue;

    PCAP_ASSERT (dev->close);  /* set by probe routine */
    FLUSHK();
    (*dev->close) (dev);

    /*
     * XXX - find out whether it's up or running?  Does that apply here?
     * Can we find out if anything's plugged into the adapter, if it's
     * a wired device, and set PCAP_IF_CONNECTION_STATUS_CONNECTED
     * or PCAP_IF_CONNECTION_STATUS_DISCONNECTED?
     */
    if ((curdev = add_dev(devlistp, dev->name, 0,
                dev->long_name, errbuf)) == NULL)
    {
      ret = -1;
      break;
    }
    found = 1;
```

这段代码是一个用于将IP地址解析为套接字地址的示例。它针对Linux系统进行测试。

代码首先检查条件0是否为真。如果是，那么将以下变量初始化为0，以便在后续操作中进行初始化：

- `sa_ll_1` 是一个用于存储 IP 地址的内存区域，是一个 16 字节的结构体，包含以下字段：
   - `sin_family`:IP 地址使用的协议类型，目前为 AF_INET，即 Internet 协议类型。
   - `sin_addr`:IP 地址的源地址。
   - `netmask`:IP 地址的子网掩码，用于将 IP 地址划分为网络地址和主机地址部分。

- `sa_ll_2` 是一个用于存储 IP 地址的内存区域，也是一个 16 字节的结构体，包含以下字段：
   - `sin_addr`:IP 地址的目标地址。
   - `broadaddr`:IP 地址的广播地址，用于将 IP 地址分配给多个接口。

- `addr`：存储 IP 地址的指针，是一个指向 struct sockaddr 的指针。
- `netmask`：存储 IP 地址的子网掩码的指针，是一个指向 struct sockaddr 的指针。
- `dstaddr`：存储目标IP地址的指针，是一个指向 struct sockaddr 的指针。

接下来，代码通过调用 `add_addr_to_dev` 函数将 IP 地址添加到设备树中。如果成功，将返回 0。如果失败，将返回一个负数。

以下是代码的伪代码：

```cpp
#if 0
   // 初始化 IP 地址
   memset(&sa_ll_1, 0, sizeof(sa_ll_1));
   memset(&sa_ll_2, 0, sizeof(sa_ll_2));
   sa_ll_1.sin_family = AF_INET;
   sa_ll_2.sin_family = AF_INET;

   // 解析 IP 地址
   addr = (struct sockaddr*) &sa_ll_1;
   netmask = (struct sockaddr*) &sa_ll_1;
   dstaddr = (struct sockaddr*) &sa_ll_2;
   broadaddr = (struct sockaddr*) &sa_ll_2;
   memset(&sa_ll_2.sin_addr, 0xFF, sizeof(sa_ll_2.sin_addr));

   // 添加 IP 地址到设备树中
   if (add_addr_to_dev(curdev, addr, sizeof(*addr),
                       netmask, sizeof(*netmask),
                       broadaddr, sizeof(*broadaddr),
                       dstaddr, sizeof(*dstaddr), errbuf) < 0)
   {
     ret = -1;
   }

   // 输出结果
   printf("添加 IP 地址 %s to device %s\n",
          inet_ntoa(addr.sin_addr), devname(curdev));
   // 输出错误信息
   printf("Error adding IP address: %s\n", errbuf);
   return 0;
#endif
```

请注意，这只是一个简单的示例代码，没有对错误处理做出足够的检查。在实际的应用程序中，需要更加完善的错误处理和安全性措施。


```cpp
#if 0   /* Pkt drivers should have no addresses */
    memset (&sa_ll_1, 0, sizeof(sa_ll_1));
    memset (&sa_ll_2, 0, sizeof(sa_ll_2));
    sa_ll_1.sin_family = AF_INET;
    sa_ll_2.sin_family = AF_INET;

    addr      = (struct sockaddr*) &sa_ll_1;
    netmask   = (struct sockaddr*) &sa_ll_1;
    dstaddr   = (struct sockaddr*) &sa_ll_1;
    broadaddr = (struct sockaddr*) &sa_ll_2;
    memset (&sa_ll_2.sin_addr, 0xFF, sizeof(sa_ll_2.sin_addr));

    if (add_addr_to_dev(curdev, addr, sizeof(*addr),
                        netmask, sizeof(*netmask),
                        broadaddr, sizeof(*broadaddr),
                        dstaddr, sizeof(*dstaddr), errbuf) < 0)
    {
      ret = -1;
      break;
    }
```

这段代码是一个 C 语言函数，它用于在程序中进行调试输出。主要作用是当程序在调试时，遇到一些期望不会发生的错误或者未找到的一些依赖项时，能够输出一条消息，以便开发人员进行调试和排查问题。

具体来说，代码包含以下几个部分：

1. 宏定义：#endif

这个宏定义在函数体中会被编译器忽略，因为它仅仅是一个注释。但是，在代码中这样定义可以让代码更易于阅读和理解。

2. 函数参数：
 - ret：整数类型，代表上文中提到的 "ret == 0 && !found" 这一逻辑判断的结果。
 - file：字符串类型，代表当前文件名。
 - line：整数类型，代表当前代码行数。

3. 函数体：
 - 如果 ret 的值为 0 并且 !found 条件为真，那么会执行下面的语句：
   - strcpy(errbuf, "No drivers found");
   - fprintf(stderr, "%s (%u): Assertion \"No drivers found\" failed", file, line);
   - close_driver();
   - _exit(-1);

  这里，首先会调用一个名为 "close_driver()" 的函数，该函数的作用是关闭网络接口，以便在程序运行结束时能够安全地关闭网络连接。然后，输出一条消息，表示在调试过程中遇到了 "No drivers found"，并关闭了驱动程序。最后，函数返回 -1，以便操作系统能够正确地关闭程序。

4. 函数实现：

在函数体中，首先判断 ret 的值是否为 0 以及 !found 是否为真。如果是，那么会执行下面的语句：

 - strcpy(errbuf, "No drivers found");
 - fprintf(stderr, "%s (%u): Assertion \"No drivers found\" failed", file, line);
 - close_driver();

这里，首先调用 "strcpy()" 函数，将 "errbuf" 变量复制到 "errbuf" 常量中。然后，调用 "fprintf()" 函数，将 "errbuf" 中的内容输出到 "stderr" 文件中，并指定了 "%s" 格式控制字符串，表示输出 "No drivers found" 字符串。最后，调用 "close_driver()" 函数，关闭网络接口。

这段代码的主要作用是在程序调试时，输出一些重要信息，以帮助开发人员进行问题排查。


```cpp
#endif
  }

  if (ret == 0 && !found)
     strcpy (errbuf, "No drivers found");

  return (ret);
}

/*
 * pcap_assert() is mainly used for debugging
 */
void pcap_assert (const char *what, const char *file, unsigned line)
{
  FLUSHK();
  fprintf (stderr, "%s (%u): Assertion \"%s\" failed\n",
           file, line, what);
  close_driver();
  _exit (-1);
}

```

这段代码定义了一个名为 `pcap_offline_read()` 的函数，它接受一个 `pcap_t` 类型的数据指针、一个函数指针 `yield` 和一个整数 `wait`。

函数的作用是设置 `pcap_set_wait()` 函数中的参数，用于在 `pcap_offline_read()` 函数中等待并打印数据包。

具体来说，这段代码首先检查传入的 `pcap_t` 是否为空，如果是，则定义一个名为 `pd` 的 `struct pcap_dos` 类型的变量，并将 `pd->wait_proc` 指向 `yield` 函数，将 `pcap_set_wait()` 函数中的 `wait` 参数设置为 `wait`。

如果 `pcap_t` 指针不为空，则执行以下操作：

1. 如果 `p` 指针为 `NULL`，则执行以下操作：

  1.1. 获取 `pcap_t` 指针指向的 `pcap_filter` 对象的 `data_offset` 和 `data_len` 成员。

  1.2. 如果 `pcap_filter` 对象没有被创建，则创建一个新的 `pcap_filter` 对象，设置其 `data_offset` 和 `data_len` 为 `0`，并将 `pcap_set_wait()` 函数中的 `yield` 函数作为其回调函数。

  1.3. 将 `pcap_t` 指针指向新的 `pcap_filter` 对象，并将 `pcap_set_wait()` 函数中的 `wait` 参数设置为 `wait`。

2. 如果 `pcap_t` 指针不为 `NULL`，则执行以下操作：

  2.1. 获取 `pcap_t` 指针指向的 `pcap_filter` 对象的 `data_offset` 和 `data_len` 成员。

  2.2. 如果 `pcap_filter` 对象已经被创建，则执行以下操作：

     2.2.1. 如果 `pcap_t` 指针指向的 `pcap_filter` 对象的 `data_offset` 为 `0`，则执行以下操作：

        2.2.1.1. 打印数据包。

        2.2.1.2. 将 `pcap_set_wait()` 函数中的 `yield` 函数作为其回调函数。

        2.2.1.3. 将 `pcap_t` 指针指向新的 `pcap_filter` 对象，并将 `pcap_set_wait()` 函数中的 `wait` 参数设置为 `wait`。

        2.2.1.4. 循环等待数据包的到达，直到 `wait` 参数指定的时间超时。

        2.2.1.5. 如果 `pcap_filter` 对象没有被创建，则创建一个新的 `pcap_filter` 对象，设置其 `data_offset` 和 `data_len` 为 `0`，并将 `pcap_set_wait()` 函数中的 `yield` 函数作为其回调函数。

        2.2.1.6. 将 `pcap_t` 指针指向新的 `pcap_filter` 对象，并将 `pcap_set_wait()` 函数中的 `wait` 参数设置为 `wait`。


```cpp
/*
 * For pcap_offline_read(): wait and yield between printing packets
 * to simulate the pace packets where actually recorded.
 */
void pcap_set_wait (pcap_t *p, void (*yield)(void), int wait)
{
  if (p)
  {
    struct pcap_dos *pd = p->priv;

    pd->wait_proc  = yield;
    p->opt.timeout = wait;
  }
}

```

这里是一个设备探测的例子，`pcap_lookupdev()` 函数用于查找设备驱动程序返回的设备名称。如果设备已经被探测过，则直接跳过这部分代码。如果设备没有被探测过，则执行一系列操作来检测设备，并将其用于 `active_dev` 变量中，以便在之后的数据链路层操作中使用。

以下是 `pcap_lookupdev()` 函数的实现：
```cppc
int pcap_lookupdev(const char *dev_name, const char *interface, int *success_retval)
{
   int retval;
   int dev;
   int i;

   retval = 0;

   for (i = 0; i < PCAP_MAX_DEVices; i++)
   {
       if (strcmp(dev_name, PCAP_DEVICE_NAMES[i]) == 0)
       {
           retval = 1;
           break;
       }
       if (PCAP_ADDR_NAMES[i] && strcmp(PCAP_ADDR_NAMES[i], interface) == 0)
       {
           retval = 1;
           break;
       }
   }

   if (retval)
   {
       dev = i;
       if (PCAP_IS_UP(dev))
       {
           retval = 1;
       }
   }

   if (*success_retval)
   {
       retval = 1;
   }

   return retval;
}
```
该函数将给定的设备名称与接口名称进行比较，以查找设备驱动程序。如果找到了匹配的设备，则返回 1，否则返回 0。如果设备已经被探测过，则跳过这部分代码。


```cpp
/*
 * Initialise a named network device.
 */
static struct device *
open_driver (const char *dev_name, char *ebuf, int promisc)
{
  struct device *dev;

  for (dev = (struct device*)dev_base; dev; dev = dev->next)
  {
    PCAP_ASSERT (dev->name);

    if (strcmp (dev_name,dev->name))
       continue;

    if (!probed_dev)   /* user didn't call pcap_lookupdev() first */
    {
      PCAP_ASSERT (dev->probe);

      if (!(*dev->probe)(dev))    /* call the xx_probe() function */
      {
        snprintf (ebuf, PCAP_ERRBUF_SIZE, "failed to detect device `%s'", dev_name);
        return (NULL);
      }
      probed_dev = dev;  /* device is probed okay and may be used */
    }
    else if (dev != probed_dev)
    {
      goto not_probed;
    }

    FLUSHK();

    /* Select what traffic to receive
     */
    if (promisc)
         dev->flags |=  (IFF_ALLMULTI | IFF_PROMISC);
    else dev->flags &= ~(IFF_ALLMULTI | IFF_PROMISC);

    PCAP_ASSERT (dev->open);

    if (!(*dev->open)(dev))
    {
      snprintf (ebuf, PCAP_ERRBUF_SIZE, "failed to activate device `%s'", dev_name);
      if (pktInfo.error && !strncmp(dev->name,"pkt",3))
      {
        strcat (ebuf, ": ");
        strcat (ebuf, pktInfo.error);
      }
      return (NULL);
    }

    /* Some devices need this to operate in promiscuous mode
     */
    if (promisc && dev->set_multicast_list)
       (*dev->set_multicast_list) (dev);

    active_dev = dev;   /* remember our active device */
    break;
  }

  /* 'dev_name' not matched in 'dev_base' list.
   */
  if (!dev)
  {
    snprintf (ebuf, PCAP_ERRBUF_SIZE, "device `%s' not supported", dev_name);
    return (NULL);
  }

```

这段代码有两个主要的作用：

1. 初始化 MAC 驱动程序：在 MAC 驱动程序初始化时，首先检查是否已经对设备进行过检测。如果没有对设备进行过检测，那么就会输出一条错误消息，然后返回 NULL。如果已经对设备进行过检测，那么就返回设备的引用。

2. 关闭 MAC 驱动程序：在 MAC 驱动程序关闭时，使用循环遍历所有 handle_to_device 结构，对每个设备进行关闭操作，然后清空 active_dev 变量，使得下一次循环时不再遍历已关闭的设备。最后，关闭设备并清空输入输出缓冲区。


```cpp
not_probed:
  if (!probed_dev)
  {
    snprintf (ebuf, PCAP_ERRBUF_SIZE, "device `%s' not probed", dev_name);
    return (NULL);
  }
  return (dev);
}

/*
 * Deinitialise MAC driver.
 * Set receive mode back to default mode.
 */
static void close_driver (void)
{
  /* !!todo: loop over all 'handle_to_device[]' ? */
  struct device *dev = active_dev;

  if (dev && dev->close)
  {
    (*dev->close) (dev);
    FLUSHK();
  }

  active_dev = NULL;

```

这段代码是一个C语言代码片段，它通过条件判断来设置某些信号的处理函数。具体来说：

1.第一行，通过使用#ifdefUSE_32BIT_DRIVERS，当32位驱动器被使用时，执行以下操作：

```cpp
#elif defined(__DJGPP__)
```

2.如果定义了USE_32BIT_DRIVERS，则执行以下操作：

```cpp
if (rx_pool)
{
   k_free (rx_pool);
   rx_pool = NULL;
}
```

清除32位无线驱动程序所创建的内存区域，并将其设置为空。

3. 如果未定义USE_32BIT_DRIVERS，则执行以下操作：

```cpp
if (dev)
   pcibios_exit();
```

执行一些普遍的应用程序终止，如PCIBIOS设置，设置屏幕和打印机等。

4. 第二行，使用了一个名为setup_signals的函数，它的参数是一个整数类型的函数指针，用于设置某些信号的处理函数。具体来说：

```cpp
static void setup_signals (void (*handler)(int))
{
 signal (SIGSEGV,handler);
 signal (SIGILL, handler);
 signal (SIGFPE, handler);
}
```

设置三个信号的处理函数为：

- SIGSEGV：SIG_NONE
- SIGILL：SIG_DFL
- SIGFPE：SIG_Warning

5. 第三行，是一个未命名的函数，它定义了一个名为handler的函数指针，用于设置信号终止信号函数的参数类型。具体来说：

```cpp
static void setup_signals (void (*handler)(int))
{
 signal (SIGSEGV,handler);
 signal (SIGILL, handler);
 signal (SIGFPE, handler);
}
```

6.第四行，是一个使用宏定义的函数，用于在程序启动时设置信号终止函数。具体来说：

```cpp
#ifdef __DJGPP__
static void setup_signals (void (*handler)(int))
{
 signal (SIGSEGV,handler);
 signal (SIGILL, handler);
 signal (SIGFPE, handler);
}
```

7.第五行，是一个用于初始化信号终止函数的函数。具体来说：

```cpp
static void setup_signals (void (*handler)(int))
{
 signal (SIGSEGV,handler);
 signal (SIGILL, handler);
 signal (SIGFPE, handler);
 --signal (SIGSEGV,handler);
 --signal (SIGILL, handler);
 --signal (SIGFPE, handler);
}
```

8.最后一行，是未命名的函数，用于设置信息，打印"Build：x.x.x，Target：x.x.x，Running：N/A"的值。具体来说：

```cpp
printf ("Build：%x.%x.%x\n", Build, Target, Runtime);
```

总结：该代码的作用是设置一系列信号的处理函数和终止函数，以使程序能够正确处理由32位硬件和/或调试器产生的信号。同时，初始化信号终止函数，以确保程序在重新编译和重新加载时，能够正确地初始化信号终止函数。


```cpp
#ifdef USE_32BIT_DRIVERS
  if (rx_pool)
  {
    k_free (rx_pool);
    rx_pool = NULL;
  }
  if (dev)
     pcibios_exit();
#endif
}


#ifdef __DJGPP__
static void setup_signals (void (*handler)(int))
{
  signal (SIGSEGV,handler);
  signal (SIGILL, handler);
  signal (SIGFPE, handler);
}

```

这段代码是一个C语言函数，名为`exc_handler`，用于处理信号`SIGSEGV`、`SIGILL`和`SIGFPE`。

当操作系统接收到`SIGSEGV`或`SIGILL`信号时，该函数将被调用。在这些信号的产生过程中，硬件可能会产生一个异常，该函数通过调用`irq_eoi_cmd`函数来向操作系统发送Eoi信号以终止异常。`_printk_safe`函数被调用，可以将`stderr`输出设置为`1`，这样就可以输出信息，即使`stderr`是`0`，以便用户了解哪些错误发生了。

当`SIGFPE`信号产生时，该函数将被调用。在这种情况下，该函数首先调用`_fpreset()`函数，然后调用`fputs()`函数将`SIGFPE`的信号名称输出到`stderr`中。`fpreset()`函数是一个辅助函数，可以在不抛出异常的情况下设置某些事情的默认值。`fputs()`函数用于将文本输出到`stderr`中。

当`SIGSEGV`或`SIGILL`信号产生时，函数返回值为`0`。当`SIGFPE`信号产生时，函数返回值为`1`。

该函数也可以传递参数，例如`int active_dev`和`int sig`，用于记录操作系统产生的信号类型和信号编号。


```cpp
static void exc_handler (int sig)
{
#ifdef USE_32BIT_DRIVERS
  if (active_dev->irq > 0)    /* excludes IRQ 0 */
  {
    disable_irq (active_dev->irq);
    irq_eoi_cmd (active_dev->irq);
    _printk_safe = 1;
  }
#endif

  switch (sig)
  {
    case SIGSEGV:
         fputs ("Catching SIGSEGV.\n", stderr);
         break;
    case SIGILL:
         fputs ("Catching SIGILL.\n", stderr);
         break;
    case SIGFPE:
         _fpreset();
         fputs ("Catching SIGFPE.\n", stderr);
         break;
    default:
         fprintf (stderr, "Catching signal %d.\n", sig);
  }
  exc_occured = 1;
  close_driver();
}
```

这段代码是一个名为“first_init”的函数，它是 capreceive 库的一部分。该函数的作用是打开 pcap 设备，以便第一个调用 pcap_activate() 的客户端。

具体来说，该函数接收一个名为“name”的参数，一个指向字符数组的指针“ebuf”，和一个表示是否为 Promiscuous（ promote）的整数“promisc”。然后，它将根据传入的参数创建一个名为“dev”的结构体变量。

接下来，该函数使用“k_calloc”函数从系统内存中分配足够的“receive_buf_size”字节数组，并将它存储在“rx_pool”变量中。如果分配失败，函数将返回一个字符串“Not enough memory (Rx pool)”，或者返回 0。

最后，该函数使用“rx_pool”变量将数据缓冲区“ebuf”初始化为“rx_pool”中的数据，并使用“pcap_activate”函数将数据缓冲区中的数据传递给“pcap_send”函数。


```cpp
#endif  /* __DJGPP__ */


/*
 * Open the pcap device for the first client calling pcap_activate()
 */
static int first_init (const char *name, char *ebuf, int promisc)
{
  struct device *dev;

#ifdef USE_32BIT_DRIVERS
  rx_pool = k_calloc (RECEIVE_BUF_SIZE, RECEIVE_QUEUE_SIZE);
  if (!rx_pool)
  {
    strcpy (ebuf, "Not enough memory (Rx pool)");
    return (0);
  }
```

这段代码是一个嵌入式系统的应用程序，主要负责初始化硬件设备和处理异常信号。

具体来说，代码中包含以下几个部分：

1. `#ifdef __DJGPP__` 是一个预处理指令，表示如果这个文件是作为 DJGPP 应用程序的子模块，那么需要执行预处理指令 `setup_signals`。这个预处理指令可能是在编译时或者链接时动态选择的。

2. `#ifdef USE_32BIT_DRIVERS` 是一个条件编译指令，表示如果这个应用程序需要使用 32 位处理器，那么需要执行 `init_32bit` 函数。这个条件编译指令使用了 `ifdef` 和 `__敢修__` 这两个指示符，它们允许在不同的编译选项下编译应用程序。

3. `#include <sys/ioctl.h>` 是一个头文件，它包含了一个名为 `open_driver` 的函数，这个函数用于初始化设备驱动程序。

4. `int main()` 是这个应用程序的主函数，它负责打开设备驱动程序，并处理从设备中接收到的数据。如果设备初始化失败，程序将会打印一些错误信息并退出。


```cpp
#endif

#ifdef __DJGPP__
  setup_signals (exc_handler);
#endif

#ifdef USE_32BIT_DRIVERS
  init_32bit();
#endif

  dev = open_driver (name, ebuf, promisc);
  if (!dev)
  {
#ifdef USE_32BIT_DRIVERS
    k_free (rx_pool);
    rx_pool = NULL;
```

这段代码是一个条件编译语句，用于检查两个条件是否成立。如果两个条件中有一个不成立，程序将跳转到标记后的代码块。

具体来说，这段代码的作用如下：

1. 如果定义了`__DJGPP__`，那么会执行`setup_signals`函数，这个函数可能是用于设置信号处理程序的一些参数设置，但是具体是什么并没有给出明确的说明。
2. 如果定义了`USE_32BIT_DRIVERS`，那么会执行下列代码：
a. 如果这是一个16位驱动器，那么将使用32位设备。
b. 如果这个驱动器不是16位"pkctl/ndis"，那么需要初始化一个近似于32位设备的环形缓冲区。
3. 如果定义了`__DJGPP__`和`USE_32BIT_DRIVERS`，那么将执行以下代码：
a. 如果当前处理器是64位，那么执行`setup_signals`函数。
b. 如果当前处理器是32位，那么执行以下代码：
i. 如果定义了`__DJGPP__`和`USE_32BIT_DRIVERS`，那么将跳转到标记后的代码块，否则执行`setup_signals`函数。
ii. 如果设置了`__DJGPP__`，那么执行`setup_signals`函数，并设置SIG_DFL信号。
iii. 如果设置了`USE_32BIT_DRIVERS`，那么根据当前设备的驱动类型来执行相应的初始化操作。
iv. 如果设备是一个16位"pkctl/ndis"，那么将使用32位设备。
v. 如果设备不是一个16位"pkctl/ndis"，那么需要初始化一个近似于32位设备的环形缓冲区。


```cpp
#endif

#ifdef __DJGPP__
    setup_signals (SIG_DFL);
#endif
    return (0);
  }

#ifdef USE_32BIT_DRIVERS
  /*
   * If driver is NOT a 16-bit "pkt/ndis" driver (having a 'copy_rx_buf'
   * set in it's probe handler), initialise near-memory ring-buffer for
   * the 32-bit device.
   */
  if (dev->copy_rx_buf == NULL)
  {
    dev->get_rx_buf     = get_rxbuf;
    dev->peek_rx_buf    = peek_rxbuf;
    dev->release_rx_buf = release_rxbuf;
    pktq_init (&dev->queue, RECEIVE_BUF_SIZE, RECEIVE_QUEUE_SIZE, rx_pool);
  }
```

这段代码是一个C语言程序，它主要作用是定义和实现一个32位硬件设备驱动程序的初始化函数。下面是具体的解释：

1. `#ifdef USE_32BIT_DRIVERS` 是条预处理指令，用于检查当前系统是否支持32位硬件设备驱动程序，如果是，则下面的代码会被执行。

2. `static void init_32bit (void)` 是定义了一个名为`init_32bit`的函数，它会在当前系统启动时执行。

3. `static int init_pci = 0` 是一个静态变量，用于记录初始化PCI的尝试次数。它的作用是在32位硬件设备驱动程序每次启动时，尝试初始化PCI，以确保系统正确支持32位硬件设备。

4. `if (!_printk_file)` 和 `_printk_init (64*1024, NULL)` 这两行代码用于在启动时打印系统启动的日志。`_printk_file`是一个环境变量，用于告诉系统中是否有可读取的日志文件。如果没有，则会执行这两行代码，将系统启动日志写入到`/var/log/sys.log`文件中。

5. `if (!init_pci)` 和 `(void)pci_init()`这两行代码用于初始化32位硬件设备驱动程序的PCI驱动部分。在初始化PCI驱动之前，需要先初始化系统中的PCI设备，以确保系统支持32位硬件设备驱动程序。这两行代码会尝试调用`pci_init()`函数，如果初始化失败，则会返回初始化失败的结果。

6. `init_pci = 1` 是一条语句，用于给静态变量`init_pci`赋值1。它的作用是在初始化PCI驱动时，确保系统会尝试调用`pci_init()`函数，即使初始化已经在之前的尝试中失败了。


```cpp
#endif
  return (1);
}

#ifdef USE_32BIT_DRIVERS
static void init_32bit (void)
{
  static int init_pci = 0;

  if (!_printk_file)
     _printk_init (64*1024, NULL); /* calls atexit(printk_exit) */

  if (!init_pci)
     (void)pci_init();             /* init BIOS32+PCI interface */
  init_pci = 1;
}
```



这段代码是一个Watt-32无线网络抓包工具中的声明，定义了一些函数用于使用Watt-32与pcap合作。具体来说，这些函数包括：

1. `watt32_recv_hook()`，是一个无线网络接收 hook 函数。该函数接收由`watt32_type`定义的 mac 地址处的数据，并将其复制到 `rxbuf` 数组中。同时，该函数根据接收到的数据中的 ether_type 字段来设置 `etype` 变量。这个函数在传输过程中被用于检测并记录无线网络中的 traffic。

2. `etype` 变量是一个 `WORD` 类型的变量，用于记录当前收到数据中的 ether_type 字段的值，可以是 0、2 或 8。这个变量被用于在接收数据时识别不同的数据类型。

3. `pcap_save` 是一个 `pcap_t` 类型的变量，用于记录当前捕获的无线网络数据。这个变量被用于在接收数据时保存数据到文件中，以便在以后的数据分析工作中使用。

4. `static void watt32_recv_hook (u_char *dummy, const struct pcap_pkthdr *pcap,
                                   const u_char *buf)` 函数声明，但不含实现。这个函数接收一个 `u_char *` 类型的参数 `buf`，用于存储无线网络中的数据。该函数在接收数据时执行，将 `buf` 中的数据复制到 `rxbuf` 数组中，并设置 `etype` 变量。然后，该函数将 `rxbuf` 和 `etype` 作为参数传递给 `watt32_type` 的 `recv()` 函数，以便将其执行。


```cpp
#endif


/*
 * Hook functions for using Watt-32 together with pcap
 */
static char rxbuf [ETH_MAX+100]; /* rx-buffer with some margin */
static WORD etype;
static pcap_t pcap_save;

static void watt32_recv_hook (u_char *dummy, const struct pcap_pkthdr *pcap,
                              const u_char *buf)
{
  /* Fix me: assumes Ethernet II only */
  struct ether_header *ep = (struct ether_header*) buf;

  memcpy (rxbuf, buf, pcap->caplen);
  etype = ep->ether_type;
  ARGSUSED (dummy);
}

```

这段代码是一个用于Watt-32包捕获的函数。它通过pcap_recv_hook函数来收集从网络设备接收到的数据包，从而绕过操作系统内置的_eth_arrived()函数。

首先，判断WATTCP_VER是否大于或等于0x0224，如果是，则执行pcap_recv_hook函数。

pcap_recv_hook函数接受一个WORD类型的参数，用于表示数据包的类型，函数内部将etype设置为该类型，然后返回一个指向数据包数据的指针。

函数内部使用pcap_read_dos函数读取一个1KB大小的数据包，该函数将数据包的头部和数据分开，其中头部包含数据包的类型、长度、服务类型等信息，而数据部分则是实际的数据。

接着，判断返回值是否为0。如果是，说明读取成功，将etype设置为数据包的类型，并将地址指向数据包数据的起始位置。如果返回值为负数，则说明发生了错误，将etype设置为NULL，并将地址设置为NULL。

最后，函数将返回etype的指针，指向数据包类型的定义，etype即为数据包的类型，如802.11协议中的802.11类型。


```cpp
#if (WATTCP_VER >= 0x0224)
/*
 * This function is used by Watt-32 to poll for a packet.
 * i.e. it's set to bypass _eth_arrived()
 */
static void *pcap_recv_hook (WORD *type)
{
  int len = pcap_read_dos (&pcap_save, 1, watt32_recv_hook, NULL);

  if (len < 0)
     return (NULL);

  *type = etype;
  return (void*) &rxbuf;
}

```

这段代码是一个Watt-32驱动程序中的函数，它被称为_eth_xmit_hook。它的作用是在网络接口发送数据时执行。函数接收一个数据缓冲区和一个数据长度，然后执行下面的操作：

1. 如果pcap_pkt_debug的值为1，那么函数内部将打印一条消息，消息将包含当前正在发送的数据包的奇偶性。
2. 如果正在运行的设备啟用了发送数据，并且啟用了dbg_init()函数，那么函数将在发送数据时执行。
3. 如果pcap_pkt_debug的值为0，那么函数不会执行1或2，但是仍然会将rc设置为数据长度，以便后续处理。
4. 函数返回一个整數，如果rc为0，则表示成功地将数据包发送出去，否则将返回一个负的整数，表示出现了一些错误。

这段代码的作用是用于在Watt-32驱动程序中，将数据包发送到网络接口，并在发送数据时执行一些调试信息，以便开发人员进行调试。


```cpp
/*
 * This function is called by Watt-32 (via _eth_xmit_hook).
 * If dbug_init() was called, we should trace packets sent.
 */
static int pcap_xmit_hook (const void *buf, unsigned len)
{
  int rc = 0;

  if (pcap_pkt_debug > 0)
     dbug_write ("pcap_xmit_hook: ");

  if (active_dev && active_dev->xmit)
     if ((*active_dev->xmit) (active_dev, buf, len) > 0)
        rc = len;

  if (pcap_pkt_debug > 0)
     dbug_write (rc ? "ok\n" : "fail\n");
  return (rc);
}
```

这段代码是一个用于 pcap-ng 库的函数，名为 `pcap_sendpacket_dos`。它实现了在发送数据时，将数据发送到指定的设备。

具体来说，这段代码在 `pcap_t` 类型的数据包结构中实现了 `send` 函数。该函数接收一个 `buf` 指针和一个 `len` 参数，用于存储数据包的实际长度。首先，它通过调用 `get_device` 函数获取到设备对象，如果设备对象为 `NULL`，则表示没有设备可以发送数据，函数将返回负值。否则，函数调用 `xmit` 函数将数据包发送到指定的设备，如果设备对象 `NULL`，则表示无法发送数据，函数将返回负值。

在实际应用中，如果需要使用 BOOTP、DHCP 或 RARP 等协议，我们需要阻止 Watt-32 等工具使用它们发送数据。这段代码实现了这个目的，从而保证了数据的安全性。


```cpp
#endif

static int pcap_sendpacket_dos (pcap_t *p, const void *buf, size_t len)
{
  struct device *dev = p ? get_device(p->fd) : NULL;

  if (!dev || !dev->xmit)
     return (-1);
  return (*dev->xmit) (dev, buf, len);
}

/*
 * This function is called by Watt-32 in tcp_post_init().
 * We should prevent Watt-32 from using BOOTP/DHCP/RARP etc.
 */
```



This code defines two function pointers: `prev_post_hook` and `null_print`.

`prev_post_hook` is a function pointer that points to a function of type `void` which is called in response to a hook. The hook is a function of type `void` that is executed before or after the current function is executed.

`null_print` is a function that prints the value of `NULL`. This function is typically used to disable printing for other functions in the program.

The code also includes some settings for the Watt-32 operating system that are intended to configure the system to work correctly. These settings include disabling the `_w32__dhcp_on` and `_w32__rarp_on` hooks, setting the `_w32__do_mask_req` to 0, and setting the `_w32_dynamic_host` field to 0.


```cpp
static void (*prev_post_hook) (void);

static void pcap_init_hook (void)
{
  _w32__bootp_on = _w32__dhcp_on = _w32__rarp_on = 0;
  _w32__do_mask_req = 0;
  _w32_dynamic_host = 0;
  if (prev_post_hook)
    (*prev_post_hook)();
}

/*
 * Suppress PRINT message from Watt-32's sock_init()
 */
static void null_print (void) {}

```

It looks like you're trying to initialize the Watt-32 TCP/IP stack for a network on an Ethernet interface. You are using a 32-bit driver and have defined a pktdrvr, but it appears that it is not being initialized.

初始化过程包括设置一些选项和设置一些保护哈希。然后检查是否有可用的 pktdrvr，如果没有，则根据给定的 IP 地址和子网掩码计算出 my_ip_addr 和 _w32_sin_mask。

接下来，您需要实现一些函数，以便在函数中执行必要的设置和初始化。这些函数包括：

1. `_watt_do_exit()` 函数：如果函数返回 0，则表示 Watt-32 已经成功初始化。
2. `_watt_is_init()` 函数：用于在 Watt-32 初始化时检查是否已成功初始化。
3. `_w32_usr_post_init()` 函数：在 pktdrvr 初始化失败时执行的函数。
4. `snprintf()` 函数：用于在 `err_buf` 中打印错误信息。


```cpp
/*
 * To use features of Watt-32 (netdb functions and socket etc.)
 * we must call sock_init(). But we set various hooks to prevent
 * using normal PKTDRVR functions in pcpkt.c. This should hopefully
 * make Watt-32 and pcap co-operate.
 */
static int init_watt32 (struct pcap *pcap, const char *dev_name, char *err_buf)
{
  char *env;
  int   rc, MTU, has_ip_addr;
  int   using_pktdrv = 1;

  /* If user called sock_init() first, we need to reinit in
   * order to open debug/trace-file properly
   */
  if (_watt_is_init)
     sock_exit();

  env = getenv ("PCAP_TRACE");
  if (env && atoi(env) > 0 &&
      pcap_pkt_debug < 0)   /* if not already set */
  {
    dbug_init();
    pcap_pkt_debug = atoi (env);
  }

  _watt_do_exit      = 0;    /* prevent sock_init() calling exit() */
  prev_post_hook     = _w32_usr_post_init;
  _w32_usr_post_init = pcap_init_hook;
  _w32_print_hook    = null_print;

  if (dev_name && strncmp(dev_name,"pkt",3))
     using_pktdrv = FALSE;

  rc = sock_init();
  has_ip_addr = (rc != 8);  /* IP-address assignment failed */

  /* if pcap is using a 32-bit driver w/o a pktdrvr loaded, we
   * just pretend Watt-32 is initialised okay.
   *
   * !! fix-me: The Watt-32 config isn't done if no pktdrvr
   *            was found. In that case my_ip_addr + sin_mask
   *            have default values. Should be taken from another
   *            ini-file/environment in any case (ref. tcpdump.ini)
   */
  _watt_is_init = 1;

  if (!using_pktdrv || !has_ip_addr)  /* for now .... */
  {
    static const char myip[] = "192.168.0.1";
    static const char mask[] = "255.255.255.0";

    printf ("Just guessing, using IP %s and netmask %s\n", myip, mask);
    my_ip_addr    = aton (myip);
    _w32_sin_mask = aton (mask);
  }
  else if (rc && using_pktdrv)
  {
    snprintf (err_buf, PCAP_ERRBUF_SIZE, "sock_init() failed, code %d", rc);
    return (0);
  }

  /* Set recv-hook for peeking in _eth_arrived().
   */
```

这段代码是一个if语句，它检查了一个名为WATTCP_VER的预先定义的常量是否大于或等于0x0224。如果是，那么它将在函数内部修改两个名为_eth_recv_hook和_eth_xmit_hook的pcaprecv和pcapxmit hook，使得它们在函数重新返回给用户之前能够继续执行。这个修改后的函数可以在网络应用程序中，当WATTCP协议支持0x0224或更高版本时，捕获和发送网络数据包。

此外，函数内部还执行了一些其他的操作，如释放由函数pkt_init()分配的pkt-drvr句柄，根据当前是否使用pktdrv来判断是否已经初始化以太网接口，并使用_eth_get_hwtype()函数获取硬件类型和驱动程序类型。


```cpp
#if (WATTCP_VER >= 0x0224)
  _eth_recv_hook = pcap_recv_hook;
  _eth_xmit_hook = pcap_xmit_hook;
#endif

  /* Free the pkt-drvr handle allocated in pkt_init().
   * The above hooks should thus use the handle reopened in open_driver()
   */
  if (using_pktdrv)
  {
    _eth_release();
/*  _eth_is_init = 1; */  /* hack to get Rx/Tx-hooks in Watt-32 working */
  }

  memcpy (&pcap_save, pcap, sizeof(pcap_save));
  MTU = pkt_get_mtu();
  pcap_save.fcode.bf_insns = NULL;
  pcap_save.linktype       = _eth_get_hwtype (NULL, NULL);
  pcap_save.snapshot       = MTU > 0 ? MTU : ETH_MAX; /* assume 1514 */

  /* prevent use of resolve() and resolve_ip()
   */
  last_nameserver = 0;
  return (1);
}

```

It appears that this is a list of 32 device debug addresses with their corresponding names and ARG_ATOI/ARG_IFACE0/ARG_IFACE1/ARG_IFACE2/ARG_IFACE3 arguments.

Each device entry has a "DEBUG" field which is a pointer to a debug handler function.

The ARG_ATOI argument is a mode that specifies that this is an ATOI device, which has the same arguments as an Intel device.

The ARG_IFACE0, ARG_IFACE1, and ARG_IFACE2 fields are used for the IFACE0, IFACE1, and IFACE2 arguments, respectively. These arguments are passed to the device-specific handler function, which should implement the correct functionality.

The ARG_IFACE3 argument is used for the IFACE3 argument, which is a mode that specifies the device's address space layout.


```cpp
int EISA_bus = 0;  /* Where is natural place for this? */

/*
 * Application config hooks to set various driver parameters.
 */

static const struct config_table debug_tab[] = {
            { "PKT.DEBUG",       ARG_ATOI,   &pcap_pkt_debug    },
            { "PKT.VECTOR",      ARG_ATOX_W, NULL               },
            { "NDIS.DEBUG",      ARG_ATOI,   NULL               },
#ifdef USE_32BIT_DRIVERS
            { "3C503.DEBUG",     ARG_ATOI,   &ei_debug          },
            { "3C503.IO_BASE",   ARG_ATOX_W, &el2_dev.base_addr },
            { "3C503.MEMORY",    ARG_ATOX_W, &el2_dev.mem_start },
            { "3C503.IRQ",       ARG_ATOI,   &el2_dev.irq       },
            { "3C505.DEBUG",     ARG_ATOI,   NULL               },
            { "3C505.BASE",      ARG_ATOX_W, NULL               },
            { "3C507.DEBUG",     ARG_ATOI,   NULL               },
            { "3C509.DEBUG",     ARG_ATOI,   &el3_debug         },
            { "3C509.ILOOP",     ARG_ATOI,   &el3_max_loop      },
            { "3C529.DEBUG",     ARG_ATOI,   NULL               },
            { "3C575.DEBUG",     ARG_ATOI,   &debug_3c575       },
            { "3C59X.DEBUG",     ARG_ATOI,   &vortex_debug      },
            { "3C59X.IFACE0",    ARG_ATOI,   &vortex_options[0] },
            { "3C59X.IFACE1",    ARG_ATOI,   &vortex_options[1] },
            { "3C59X.IFACE2",    ARG_ATOI,   &vortex_options[2] },
            { "3C59X.IFACE3",    ARG_ATOI,   &vortex_options[3] },
            { "3C90X.DEBUG",     ARG_ATOX_W, &tc90xbc_debug     },
            { "ACCT.DEBUG",      ARG_ATOI,   &ethpk_debug       },
            { "CS89.DEBUG",      ARG_ATOI,   &cs89_debug        },
            { "RTL8139.DEBUG",   ARG_ATOI,   &rtl8139_debug     },
        /*  { "RTL8139.FDUPLEX", ARG_ATOI,   &rtl8139_options   }, */
            { "SMC.DEBUG",       ARG_ATOI,   &ei_debug          },
        /*  { "E100.DEBUG",      ARG_ATOI,   &e100_debug        }, */
            { "PCI.DEBUG",       ARG_ATOI,   &pci_debug         },
            { "BIOS32.DEBUG",    ARG_ATOI,   &bios32_debug      },
            { "IRQ.DEBUG",       ARG_ATOI,   &irq_debug         },
            { "TIMER.IRQ",       ARG_ATOI,   &timer_irq         },
```

这是一个用于链表存储设备名称和对应关系的代码。具体来说，它定义了一个名为 pcap_config_hook 的函数，用于在应用程序的配置处理中使用 Watt-32 的 config-table 函数。函数接收两个参数，一个是关键词（通常是一个字符串），另一个是参数值。函数内部使用 parse_config_table 函数从 config-table 中解析出相应的配置项，并返回其结果。最后，函数没有返回值，因此需要在调用它的函数中使用它。


```cpp
#endif
            { NULL }
          };

/*
 * pcap_config_hook() is an extension to application's config
 * handling. Uses Watt-32's config-table function.
 */
int pcap_config_hook (const char *keyword, const char *value)
{
  return parse_config_table (debug_tab, NULL, keyword, value);
}

/*
 * Linked list of supported devices
 */
```



该代码定义了两个指向device结构体的指针active_dev和probed_dev，以及一个指向device结构体的指针dev_base。

active_dev和probed_dev变量都初始化为 NULL，表示尚未开启或探测的设备。

dev_base是一个device结构体数组，保存了所有已知网络设备的列表。

pkt_pkt_debug是一个整数变量，用于记录已经打印的pkt设备相关的调试信息。如果该变量大于1，那么将会使用fprintf函数在屏幕上打印出一条调试信息。

pkt_close函数是用于关闭已经开启的网络设备的功能。该函数返回一个布尔值，如果设备关闭成功，返回TRUE，否则返回FALSE。

在该函数中，首先使用PktExitDriver函数来关闭网络设备，然后打印一条调试信息。接着，释放dev设备结构的内存，使得dev设备结构体不再被使用。


```cpp
struct device       *active_dev = NULL;      /* the device we have opened */
struct device       *probed_dev = NULL;      /* the device we have probed */
const struct device *dev_base   = &pkt_dev;  /* list of network devices */

/*
 * PKTDRVR device functions
 */
int pcap_pkt_debug = -1;

static void pkt_close (struct device *dev)
{
  BOOL okay = PktExitDriver();

  if (pcap_pkt_debug > 1)
     fprintf (stderr, "pkt_close(): %d\n", okay);

  if (dev->priv)
     free (dev->priv);
  dev->priv = NULL;
}

```

这段代码定义了一个名为 pkt_open 的函数，它接收一个设备指针作为参数。函数的作用是初始化一个接收套接字（RX）的驱动程序。以下是函数的步骤：

1. 检查设备指针的标志位是否为 IFF_PROMISC。如果是，将模式设置为 PDRX_ALL_PACKETS。
2. 否则，将模式设置为 PDRX_BROADCAST。
3. 如果初始化驱动程序失败，返回 0。
4. 调用 PktInitDriver 函数，设置接收套接字的驱动程序 mode。
5. 调用 PktResetStatistics 函数，清除接收套接字的统计信息。
6. 调用 PktQueueBusy 函数，初始化接收套接字队列为空。
7. 返回 1，表示函数成功完成。


```cpp
static int pkt_open (struct device *dev)
{
  PKT_RX_MODE mode;

  if (dev->flags & IFF_PROMISC)
       mode = PDRX_ALL_PACKETS;
  else mode = PDRX_BROADCAST;

  if (!PktInitDriver(mode))
     return (0);

  PktResetStatistics (pktInfo.handle);
  PktQueueBusy (FALSE);
  return (1);
}

```

这段代码定义了一个名为 pkt_xmit 的静态函数，属于 device 设备结构的设备。

函数接受三个参数：dev 指针，用于访问设备；buf 是一个字符数组，用于传输的数据；len 是一个整数，表示传输数据的长度。

函数首先检查是否启用了 pcap_pkt_debug 打印调试信息，如果没有启用，则输出一条调试信息。然后，函数调用 PktTransmit 函数将数据传输到网络接口。

函数的一个可选实现是：dbg_write("pcap_xmit: "); debug_write (stderr, buf, len); 这条输出消息会包含到 pcap_pkt_debug 打印调试信息的行。

函数返回一个整数，表示成功传输数据的结果。如果函数返回负数，则表示在传输数据过程中发生了一个错误，并为统计变量加一。


```cpp
static int pkt_xmit (struct device *dev, const void *buf, int len)
{
  struct net_device_stats *stats = (struct net_device_stats*) dev->priv;

  if (pcap_pkt_debug > 0)
     dbug_write ("pcap_xmit\n");

  if (!PktTransmit(buf,len))
  {
    stats->tx_errors++;
    return (0);
  }
  return (len);
}

```



这段代码定义了两个函数，分别是 `pkt_stats` 和 `pkt_probe`。

1. `pkt_stats` 函数接收一个 `struct device *dev` 类型的参数，并返回一个指向 `struct net_device_stats` 类型的指针。函数的主要作用是获取网络设备统计信息，包括收到的 packets 数量、错误的数量和丢失的 packets 数量。函数首先检查是否传递了统计信息或者统计信息是否有效，如果失败则返回 NULL，否则返回指向统计信息的指针。

2. `pkt_probe` 函数也接收一个 `struct device *dev` 类型的参数，并返回一个表示是否已注册到 `kernel` 中的整数。函数的主要作用是在网络设备被插入到系统后，将其 `probe` 函数注册到系统中，以便在驱动程序需要时动态地获取统计信息。函数首先检查是否已经注册了驱动程序，如果没有，则返回 0，否则返回 1，表示成功将驱动程序注册到系统中。

`pkt_probe` 函数的具体实现比较简单，主要通过调用 `pkt_open`、`pkt_xmit` 和 `pkt_stats` 函数来注册和获取统计信息，并将这些信息存储在一个 `struct net_device_stats` 类型的结构体中，最终返回这个结构体。


```cpp
static void *pkt_stats (struct device *dev)
{
  struct net_device_stats *stats = (struct net_device_stats*) dev->priv;

  if (!stats || !PktSessStatistics(pktInfo.handle))
     return (NULL);

  stats->rx_packets       = pktStat.inPackets;
  stats->rx_errors        = pktStat.lost;
  stats->rx_missed_errors = PktRxDropped();
  return (stats);
}

static int pkt_probe (struct device *dev)
{
  if (!PktSearchDriver())
     return (0);

  dev->open           = pkt_open;
  dev->xmit           = pkt_xmit;
  dev->close          = pkt_close;
  dev->get_stats      = pkt_stats;
  dev->copy_rx_buf    = PktReceive;  /* farmem peek and copy routine */
  dev->get_rx_buf     = NULL;
  dev->peek_rx_buf    = NULL;
  dev->release_rx_buf = NULL;
  dev->priv           = calloc (sizeof(struct net_device_stats), 1);
  if (!dev->priv)
     return (0);
  return (1);
}

```

这段代码定义了NDIS设备的函数，包括ndis_close和ndis_open。ndis_close函数在设备关闭时执行，ndis_open函数在设备打开时执行。

ndis_close函数通过调用NdisShutdown()函数来关闭设备。NdisShutdown()函数用于关闭NDIS设备的硬件和软件资源，包括关闭设备上连接的硬件设备驱动程序和NDIS服务程序。

ndis_open函数检查设备是否为promisc设备，如果是，则执行ndis_close函数。否则，ndis_open函数会在设备打开时执行，设备将一直处于打开状态。

ndis_open函数的实现主要涉及到两个步骤：首先检查设备是否为promisc设备，如果是，则调用ndis_close函数关闭设备。否则，在设备打开时执行ndis_close函数。


```cpp
/*
 * NDIS device functions
 */
static void ndis_close (struct device *dev)
{
#ifdef USE_NDIS2
  NdisShutdown();
#endif
  ARGSUSED (dev);
}

static int ndis_open (struct device *dev)
{
  int promisc = (dev->flags & IFF_PROMISC);

```

这段代码是一个NDIS (Network Device INFORMATION) 驱动程序。它通过NDIS库支持网络设备统计信息，如收发数据包数和错误统计。以下是代码的功能和用途：

1. 首先，定义了一个预处理函数NDIS_IF_CONFIG，如果当前没有配置NDIS，则返回0；否则返回1。
2. 如果已经配置了NDIS，则检查设备是否支持NDIS。如果不支持，返回0；否则返回1。
3. 如果当前已经配置了NDIS，那么定义了一个名为ndis_stats的函数，它接受一个设备指针作为参数。
4. 在ndis_stats函数中，定义了一个静态的统计信息结构体net_device_stats，用于存储统计数据。
5. 在ndis_stats函数中，调用了一个名为promisc的参数。推测promisc是一个NDIS滤波器，用于指定允许NDIS统计的数据包类型。
6. 然后，创建并初始化一个名为stats的net_device_stats结构体，用于存储统计数据。
7. 最后，将stats作为ndis_stats函数的返回值，以便驱动程序可以调用ndis_stats函数来获取统计信息。


```cpp
#ifdef USE_NDIS2
  if (!NdisInit(promisc))
     return (0);
  return (1);
#else
  ARGSUSED (promisc);
  return (0);
#endif
}

static void *ndis_stats (struct device *dev)
{
  static struct net_device_stats stats;

  /* to-do */
  ARGSUSED (dev);
  return (&stats);
}

```

这段代码是一个设备驱动程序，主要是用来初始化网络接口设备(NDIS)。

具体来说，代码中定义了一个名为ndis_probe的函数，它接收一个设备指针作为参数。如果网络接口设备无法打开，函数返回0。

函数内部先判断是否使用了NDIS2库，如果是，则执行ndis_open函数打开网络接口设备。如果不是NDIS2库，则需要手动初始化NDIS，具体实现包括设置网络接口设备的编号、开启NDIS、关闭NDIS等操作。

在函数内部，还定义了一系列与NDIS相关的函数，包括ndis_open、ndis_close、ndis_stats等。这些函数用于获取和设置网络接口设备的统计信息。

另外，函数内部还定义了一个copy_rx_buf函数，用于从接收模式(RXMODE)切换到发送模式(TRXMODE)，但该函数并未实现具体的操作，只是声明了一个空指针，需要由用户自己来实现。

最后，函数返回0表示初始化成功，否则返回-1表示初始化失败。


```cpp
static int ndis_probe (struct device *dev)
{
#ifdef USE_NDIS2
  if (!NdisOpen())
     return (0);
#endif

  dev->open           = ndis_open;
  dev->xmit           = NULL;
  dev->close          = ndis_close;
  dev->get_stats      = ndis_stats;
  dev->copy_rx_buf    = NULL;       /* to-do */
  dev->get_rx_buf     = NULL;       /* upcall is from rmode driver */
  dev->peek_rx_buf    = NULL;
  dev->release_rx_buf = NULL;
  return (0);
}

```

这段代码的作用是搜索和探测支持32位（pmode）pcap设备的设备。它定义了两个名为el2_dev和el3_dev的结构体变量，并使用它们定义了两个设备的名称和初始化信息。然后，它枚举了当前系统中可用的设备，如果使用的是32位驱动程序，那么就监听pmode的设备。最后，它返回了一个device类型的变量，用于跟踪下一个可用设备的lpcap成员。


```cpp
/*
 * Search & probe for supported 32-bit (pmode) pcap devices
 */
#if defined(USE_32BIT_DRIVERS)

struct device el2_dev LOCKED_VAR = {
              "3c503",
              "EtherLink II",
              0,
              0,0,0,0,0,0,
              NULL,
              el2_probe
            };

struct device el3_dev LOCKED_VAR = {
              "3c509",
              "EtherLink III",
              0,
              0,0,0,0,0,0,
              &el2_dev,
              el3_probe
            };

```

这两段代码定义了两个结构体变量：`tc515_dev` 和 `tc59_dev`，它们都被赋名为 `LOCKED_VAR`。

每个结构体变量包含以下字段：

- `"3c515"`：一个字符串，表示设备的制造商名称。
- `EtherLink PCI`：一个字符串，表示设备使用的协议。
- `0`：一个整数，表示设备配置为最高速度。
- `0`：一个整数，表示设备配置为全双工模式。
- `0`：一个整数，表示设备配置为无速度模式。
- `0`：一个指向 `EtherLink Dev` 结构的指针，设备接口的句柄。
- `tc515_probe`：一个指向 `EtherLink Dev` 结构的指针，设备接口的实现。
- `tc59_probe`：一个指向 `EtherLink Dev` 结构的指针，设备接口的实现。

这两段代码的作用是定义了两个结构体变量，用于表示网络接口卡的信息。第一个结构体变量 `tc515_dev` 可能是一个网络接口卡的制造商，第二个结构体变量 `tc59_dev` 可能是一个网络接口卡的制造商，但它们的值是未知的。


```cpp
struct device tc515_dev LOCKED_VAR = {
              "3c515",
              "EtherLink PCI",
              0,
              0,0,0,0,0,0,
              &el3_dev,
              tc515_probe
            };

struct device tc59_dev LOCKED_VAR = {
              "3c59x",
              "EtherLink PCI",
              0,
              0,0,0,0,0,0,
              &tc515_dev,
              tc59x_probe
            };

```

这段代码定义了两个结构体变量：tc90xbc_dev和wd_dev。它们都包含一个名为"device"的成员变量，以及一个名为"locked_var"的成员变量。

tc90xbc_dev成员的值使用了字符串格式化，将"3c90x"和"EtherLink 90X"分别赋值给"3c90x"和"EtherLink 90X"成员变量。

wd_dev成员的值使用了整型格式化，将"wd"赋值给"wd"成员变量，将0和0分别赋值给"0"成员变量。

两个结构体变量的成员变量都包含一个名为"device_probe"的成员变量，它们的值都是一个指向device类型变量的指针。这表明它们都指向同一个外设，即90x乙太网芯片的probe版本。


```cpp
struct device tc90xbc_dev LOCKED_VAR = {
              "3c90x",
              "EtherLink 90X",
              0,
              0,0,0,0,0,0,
              &tc59_dev,
              tc90xbc_probe
            };

struct device wd_dev LOCKED_VAR = {
              "wd",
              "Westen Digital",
              0,
              0,0,0,0,0,0,
              &tc90xbc_dev,
              wd_probe
            };

```

这两段代码定义了两个结构体变量：dev和acct。

dev结构体变量定义了一个名为"device"，结构体变量中包含以下字段：
- "ne" 是一个字符串，表示设备的名称。
- "NEx000" 是一个字符串，表示设备的 MAC 地址。
- 0 到 0 表示这是一个0，即这个字段可以被初始化为0。
- 0 到 0 表示这是一个0，即这个字段可以被初始化为0。
- 0 到 0 表示这是一个0，即这个字段可以被初始化为0。
- 指向一个名为wd_dev的设备的指针。
- ne_probe是一个设备指针变量，这个变量存储了device.ne的引用。

acct结构体变量定义了一个名为"acct"，结构体变量中包含以下字段：
- "acct" 是一个字符串，表示设备的名称。
- "Accton EtherPocket" 是一个字符串，表示设备的 IP 地址。
- 0 到 0 表示这是一个0，即这个字段可以被初始化为0。
- 0 到 0 表示这是一个0，即这个字段可以被初始化为0。
- 0 到 0 表示这是一个0，即这个字段可以被初始化为0。
- 指向一个名为ne_dev的设备的指针。
- ethpk_probe是一个设备指针变量，这个变量存储了device.acct的引用。


```cpp
struct device ne_dev LOCKED_VAR = {
              "ne",
              "NEx000",
              0,
              0,0,0,0,0,0,
              &wd_dev,
              ne_probe
            };

struct device acct_dev LOCKED_VAR = {
              "acct",
              "Accton EtherPocket",
              0,
              0,0,0,0,0,0,
              &ne_dev,
              ethpk_probe
            };

```

这两段代码定义了两个结构体变量，分别是 "device" 和 "rtl8139_dev"，它们都被定义为 "LOCKED_VAR" 类型。

"device" 结构体变量定义了一个关于无线网络适配器的设备信息，包括制造商、芯片和配置等。这个结构体变量可能用于记录多个与无线网络相关的设备，如无线网卡、无线集线器等。

"rtl8139_dev" 结构体变量定义了一个关于无线网络适配器的设备信息，包括制造商、芯片和配置等。这个结构体变量也可能用于记录多个与无线网络相关的设备，如无线网卡、无线集线器等。

在这两个结构体变量中，都包含一个名为 "&acct\_dev" 的成员，它是一个指向无线网络适配器（acct\_dev）的指针。另外，这两个结构体变量还包含一个名为 "cs89x0\_probe" 的成员，它是一个用于配置无线网络适配器探针（probe）的整数。


```cpp
struct device cs89_dev LOCKED_VAR = {
              "cs89",
              "Crystal Semiconductor",
              0,
              0,0,0,0,0,0,
              &acct_dev,
              cs89x0_probe
            };

struct device rtl8139_dev LOCKED_VAR = {
              "rtl8139",
              "RealTek PCI",
              0,
              0,0,0,0,0,0,
              &cs89_dev,
              rtl8139_probe     /* dev->probe routine */
            };

```

这段代码定义了一个名为 "peek_rxbuf" 的函数，该函数通过轮询的方式获取队列中的元素。该函数接收一个指向 "buf" 指针的参数。

函数首先定义了两个指向 "rx_elem" 结构体的指针 "tail" 和 "head"，用于跟踪队列的起始元素和结束元素。

接着，函数在内置循环中进行以下操作：

1. 调用 "pktq_check" 函数，检查队列中是否有数据。
2. 如果队列中有数据，获取队列头元素（即第一个元素）和队列尾元素（即最后一个元素）。
3. 将取出的元素从队列中删除。
4. 将取出的元素复制到 "buf" 指向的内存位置。
5. 如果取出的元素是队列的末尾元素，将 "buf" 指针指向 NULL，并返回取出的元素个数（即队列长度）。

函数的实现使用了进程通信中的轮询机制，即不断从队列中取出元素，直到队列为空或取出所有元素为止。该函数可以用于实时传输系统等应用中，通过轮询获取网络数据中的有效信息。


```cpp
/*
 * Dequeue routine is called by polling.
 * NOTE: the queue-element is not copied, only a pointer is
 * returned at '*buf'
 */
int peek_rxbuf (BYTE **buf)
{
  struct rx_elem *tail, *head;

  PCAP_ASSERT (pktq_check (&active_dev->queue));

  DISABLE();
  tail = pktq_out_elem (&active_dev->queue);
  head = pktq_in_elem (&active_dev->queue);
  ENABLE();

  if (head != tail)
  {
    PCAP_ASSERT (tail->size < active_dev->queue.elem_size-4-2);

    *buf = &tail->data[0];
    return (tail->size);
  }
  *buf = NULL;
  return (0);
}

```

这段代码定义了一个名为`release_rxbuf`的函数，它接收一个字节数组`buf`作为参数。函数内部使用了`PCAP`和`pktq_x`函数，因此我们可以推测出函数的作用是释放一个正在接收数据的套接字队列，将`buf`中的数据传送给队列，并将套接字队列的数量加一。

进一步分析，我们可以看到函数使用了`BYTE`类型，这表明它接收的是一字节数组。而`ARGSUSED`函数则表示该函数在使用参数`buf`时需要确保该参数不为空。因此，我们可以得出结论：该函数的作用是释放一个正在接收数据的套接字队列，将`buf`中的数据传送给队列，并将套接字队列的数量加一。


```cpp
/*
 * Release buffer we peeked at above.
 */
int release_rxbuf (BYTE *buf)
{
#ifndef NDEBUG
  struct rx_elem *tail = pktq_out_elem (&active_dev->queue);

  PCAP_ASSERT (&tail->data[0] == buf);
#else
  ARGSUSED (buf);
#endif
  pktq_inc_out (&active_dev->queue);
  return (1);
}

```

这段代码是一个名为`get_rxbuf()`的函数，它在受保护的代码中从设备中断（IRQ）处理程序中调用，用于请求一个缓冲区。在这里，我们可以得到以下解答：

1. 函数的作用：该函数检查传入的缓冲区长度是否符合期望的值，如果不符合，则返回一个空指针。

2. 输入参数：函数有两个输入参数，一个是整数类型，表示要请求的缓冲区长度；另一个是整数类型，表示中断传输的帧的最大长度。

3. 函数内部操作：

  a. 首先检查传入的缓冲区长度是否小于或等于`ETH_MIN`和`ETH_MAX`中的最小值和最大值，否则返回一个空指针。

  b. 从`active_dev`设备的`queue`成员中读取中断传输的帧的下一个分片的位置，即`pktq_in_index()`函数返回的值。

  c. 通过`active_dev`设备的状态判断，如果`active_dev`为发送设备，则需要调用`device_send_frame()`函数；如果`active_dev`为接收设备，则需要调用`device_receive_frame()`函数。

  d. 在调用`device_send_frame()`函数和`device_receive_frame()`函数时，通过写入和中断号作为缓冲区字符串的前缀，构造出一个32KB的栈，用于保存中断传输的帧。

4. 输出：函数没有输出任何内容。


```cpp
/*
 * get_rxbuf() routine (in locked code) is called from IRQ handler
 * to request a buffer. Interrupts are disabled and we have a 32kB stack.
 */
BYTE *get_rxbuf (int len)
{
  int idx;

  if (len < ETH_MIN || len > ETH_MAX)
     return (NULL);

  idx = pktq_in_index (&active_dev->queue);

#ifdef DEBUG
  {
    static int fan_idx LOCKED_VAR = 0;
    writew ("-\\|/"[fan_idx++] | (15 << 8),      /* white on black colour */
            0xB8000 + 2*79);  /* upper-right corner, 80-col colour screen */
    fan_idx &= 3;
  }
```

这段代码是一个C语言函数，它的作用是检查传入的`active_dev->queue`是否为活动设备中的输出队列。如果是，则遍历该队列中的所有元素并返回第一个元素。如果不是，则执行以下操作：

1. 将`active_dev->queue`中的元素个数存储在变量`len`中。
2. 将`active_dev->queue.out_index`设置为传入的`idx`。
3. 返回`active_dev->queue.in_index`对应的元素。
4. 如果`idx`不等于`active_dev->queue.out_index`，则执行以下操作：
a. 通过`pktq_in_elem`函数将输入队列中的元素一个一个地加入`active_dev->queue`中。
b. 执行`pktq_clear`函数清除`active_dev->queue`中的所有元素。
c. 执行`return`语句返回`active_dev->queue.in_index`对应的元素。

该函数的作用是检查要返回的元素是否来自输入队列，如果不是，则执行清除输入队列并返回。注意，该函数没有实现删除最老的元素的操作，这需要您自己实现。


```cpp
/* writew (idx + '0' + 0x0F00, 0xB8000 + 2*78); */
#endif

  if (idx != active_dev->queue.out_index)
  {
    struct rx_elem *head = pktq_in_elem (&active_dev->queue);

    head->size = len;
    active_dev->queue.in_index = idx;
    return (&head->data[0]);
  }

  /* !!to-do: drop 25% of the oldest element
   */
  pktq_clear (&active_dev->queue);
  return (NULL);
}

```

这段代码定义了一个名为 `pktq_check` 的函数，该函数接收一个 `struct rx_ringbuf` 类型的参数 `q`。

这个函数首先检查参数 `q` 是否为 `NULL`，如果是，则返回 `0`。然后，它检查 `q` 是否已满，如果是，则返回 `-1`，表示已满无法接收新数据。如果 `q` 不为 `NULL` 且 `q->num_elem` 和 `q->buf_start` 都为真，则表示有数据要接收，函数会返回 `0`，表示成功接收数据。

由于函数内没有对输入参数 `q` 进行任何实际的输入或输出操作，因此函数在实际应用中不会对数据产生影响，只是返回一个表示接收状态的值。


```cpp
/*
 *  Simple ring-buffer queue handler for reception of packets
 *  from network driver.
 */
#define PKTQ_MARKER  0xDEADBEEF

static int pktq_check (struct rx_ringbuf *q)
{
#ifndef NDEBUG
  int   i;
  char *buf;
#endif

  if (!q || !q->num_elem || !q->buf_start)
     return (0);

```

这段代码定义了一个名为`pktq_init`的函数，属于`struct rx_ringbuf`类型的用户数据结构。该函数初始化一个`struct rx_elem`类型的 ringbuf，并将其大小设置为要发送的数据元素数量乘以每个数据元素的大小。函数的实现包括以下几个步骤：

1. 将 ringbuf 的起始地址设为传入的`pool`指针，以便在接收数据时从该地址开始接收数据。

2. 创建一个 `num_elem` 类型的变量 `i`，用于遍历 data elements。

3. 创建一个 `DWORD` 类型的变量 `buf`，用于存储每个 data element。同时，使用 `DWORD` 类型的指针 `stride` 来计算每个数据元素的大小。

4. 遍历每个数据元素，计算出该数据元素在 ringbuf 中的偏移量，如果该偏移量不等于标记 `PKTQ_MARKER`，就说明数据元素已经到达 ringbuf 的末尾，可以返回一个错误码。

5. 在遍历过程中，将每个数据元素的偏移量加一个大小的 4，以确保数据元素在 ringbuf 中正确排列。

6. 循环结束后，将 ringbuf 的起始地址恢复为传入的 `pool` 指针，并返回初始化成功。


```cpp
#ifndef NDEBUG
  buf = q->buf_start;

  for (i = 0; i < q->num_elem; i++)
  {
    buf += q->elem_size;
    if (*(DWORD*)(buf - sizeof(DWORD)) != PKTQ_MARKER)
       return (0);
  }
#endif
  return (1);
}

static int pktq_init (struct rx_ringbuf *q, int size, int num, char *pool)
{
  int i;

  q->elem_size = size;
  q->num_elem  = num;
  q->buf_start = pool;
  q->in_index  = 0;
  q->out_index = 0;

  PCAP_ASSERT (size >= sizeof(struct rx_elem) + sizeof(DWORD));
  PCAP_ASSERT (num);
  PCAP_ASSERT (pool);

  for (i = 0; i < num; i++)
  {
```

这段代码是一个 C 语言函数，名为 `increment_queue`。它对一个名为 `out_index` 的队列进行递增，并将队尾元素的索引增加 1，同时将队列的头指针（即队列元素的下一个元素）指向新的队尾元素。

在这段代码中，首先定义了一个名为 `elem` 的结构体变量，它是一个指向 `struct rx_elem` 类型的指针。接着，代码中有一个条件语句 `if 0`，该条件语句始终为真（即不输出 anything），但是为了解决代码可读性，这里暂时保留。

接下来，在 `if 0` 的注释下，定义了一个名为 `size` 的整型变量，并将其加 1，然后将结果存储为 `out_index` 变量的索引。

最后，代码调用了函数 `increment_queue`，并将其返回值设置为 1，表明函数成功执行。


```cpp
#if 0
    struct rx_elem *elem = (struct rx_elem*) pool;

    /* assert dword aligned elements
     */
    PCAP_ASSERT (((unsigned)(&elem->data[0]) & 3) == 0);
#endif
    pool += size;
    *(DWORD*) (pool - sizeof(DWORD)) = PKTQ_MARKER;
  }
  return (1);
}

/*
 * Increment the queue 'out_index' (tail).
 * Check for wraps.
 */
```

这两段代码定义了一个名为 pktq_inc_out 的函数和一个名为 pktq_in_index 的函数。它们都是属于一个名为 pktq_ringbuf 的结构体。

pktq_inc_out 函数的作用是在 ringbuf 中增加一个元素，并将该元素的出队索引设置为当前元素的索引加 1。如果当前元素的数量已经达到了 ringbuf 的最大元素数，那么出队索引将被重置为 0。函数的返回值是增加的元素的索引。

pktq_in_index 函数的作用是返回队列中当前元素的索引，检查是否发生了环。如果当前元素的索引超出了队列元素的数量，那么索引将被重置为 0。函数的返回值是当前元素的索引，如果没有发生环。

整个函数的作用是提供一个队列，允许用户在发送数据之前增加队列元素，并返回下一个要发送的元素的索引，同时检查队列是否发生了环。


```cpp
static int pktq_inc_out (struct rx_ringbuf *q)
{
  q->out_index++;
  if (q->out_index >= q->num_elem)
      q->out_index = 0;
  return (q->out_index);
}

/*
 * Return the queue's next 'in_index' (head).
 * Check for wraps.
 */
static int pktq_in_index (struct rx_ringbuf *q)
{
  volatile int index = q->in_index + 1;

  if (index >= q->num_elem)
      index = 0;
  return (index);
}

```

这两函数的作用是返回射线形循环缓冲器（rx_ringbuf）中的队头元素（elem）和队尾元素（elem）的内存地址。

队头元素（elem）是射线形缓冲器中未被发送出去的元素，也就是队头的内容。通过这两函数可以获取到队头元素的位置，从而实现获取队头元素的功能。

队尾元素（elem）是射线形缓冲器中已发送出去但还未被接收到的元素，也就是队尾的内容。通过这两函数可以获取到队尾元素的位置，从而实现获取队尾元素的功能。


```cpp
/*
 * Return the queue's head-buffer.
 */
static struct rx_elem *pktq_in_elem (struct rx_ringbuf *q)
{
  return (struct rx_elem*) (q->buf_start + (q->elem_size * q->in_index));
}

/*
 * Return the queue's tail-buffer.
 */
static struct rx_elem *pktq_out_elem (struct rx_ringbuf *q)
{
  return (struct rx_elem*) (q->buf_start + (q->elem_size * q->out_index));
}

```

这段代码定义了一个名为 `pktq_clear` 的函数，属于 `rx_ringbuf` 结构体的成员函数。其作用是清空队列中的数据，可以通过设置 `q->in_index` 和 `q->out_index` 来完成。

另外，定义了一个名为 `extern` 的符号，但需要在调用时使用 `__IOPORT_H` 和 `__DMA_H`，这意味着它是一个导出函数，用于在名为 `mem_align_sort_loop` 的函数中使用。


```cpp
/*
 * Clear the queue ring-buffer by setting head=tail.
 */
static void pktq_clear (struct rx_ringbuf *q)
{
  q->in_index = q->out_index;
}

/*
 * Symbols that must be linkable for "gcc -O0"
 */
#undef __IOPORT_H
#undef __DMA_H

#define extern
```

这段代码是一个定义，定义了一个名为`__inline__`的宏，其含义是允许在使用这个定义的地方使用该定义，从而使代码更易于阅读和维护。

接下来的两个头文件包含定义中定义的宏的前缀，这两个头文件分别是"msdos/pm_drvr/ioport.h"和"msdos/pm_drvr/dma.h"。

接下来是一个双等号，表示该行为块内定义的符号常量。该常量名称为"USE_32BIT_DRIVERS"，用于控制是否使用32位驱动程序。如果使用32位驱动程序，则将该常量设置为`1`；否则，该常量设置为`0`。

然后是定义了一个名为"pcap_lib_version"的函数，该函数返回pcap库的版本字符串。

最后是另外两个函数，分别包含在滥用可移植代码指南(libpcap)中，用于在必要的情况下输出调试信息。


```cpp
#define __inline__

#include "msdos/pm_drvr/ioport.h"
#include "msdos/pm_drvr/dma.h"

#endif /* USE_32BIT_DRIVERS */

/*
 * Libpcap version string.
 */
const char *
pcap_lib_version(void)
{
  return ("DOS-" PCAP_VERSION_STRING);
}

```