# Nmap源码解析 62

# `libpcap/pcap-snf.c`

这段代码的作用是检查系统是否支持配置文件。如果系统支持配置文件，那么它将从`/etc/config.h`文件中包含一个名为`config.h`的定义。否则，它将包含一个名为`config.h`的定义，与`/etc/config.h`文件中的内容相同。

对于Unix系统，它还检查`_WIN32`操作系统是否支持本地化查询字符串。如果是，那么它将从`/usr/share/locale/LC_MEDIA_DIR`目录中查找本地化查询字符串，否则将覆盖该目录。

对于Linux系统，它使用`sys/param.h`函数来获取`/etc/config.h`文件中定义的配置参数。

对于Windows系统，它使用`<netinet/in.h>` header文件来获取本地化查询字符串。

总的来说，这段代码的作用是定义了一个可移植的配置文件，以便在系统启动时从启动时配置文件开始获取配置信息。


```cpp
#ifdef HAVE_CONFIG_H
#include <config.h>
#endif

#ifndef _WIN32
#include <sys/param.h>
#endif /* !_WIN32 */

#include <stdlib.h>
#include <string.h>
#include <errno.h>
#include <limits.h> /* for INT_MAX */

#ifndef _WIN32
#include <netinet/in.h>
```

这段代码是一个用于在Linux系统上捕捉和处理数据包的Python程序。它包含以下几个部分：

1. 引入相关头文件：首先，它引入了`/sys/mman.h`、`/sys/socket.h`和`<strics/snf.h>`头文件，这些头文件定义了与操作系统虚拟内存和网络套接字相关的接口，以及用于处理数据包的标准函数。

2. 检查SNF支持：由于程序需要使用SNF（SystemN武术）设备，它需要知道SNF是否支持`inject`函数。这个函数可以在原始数据包中注入额外的数据，以改变数据包的发送方向。

3. 包含必要的头文件：程序还包含了`<unistd.h>`头文件，这是用于使用C库函数的必需品。

4. 导入pcap库：程序导入了`pcap-int.h`和`pcap-snf.h`头文件。这两个头文件包含了pcap库所需的函数，以及用于处理SNF数据的函数原型。

5. 处理数据：程序定义了一系列私有函数，这些函数将捕获的数据包传递给pcap库进行处理。这些函数包括：`parse_pcap_header`、`parse_ Ethernet_header`、`parse_IP_header`、`parse_TCP_header`、`get_ packets`和`free_pcap_header`。

6. 初始化pcap库：程序还定义了一个名为`init_pcap`的函数，该函数用于初始化pcap库。这个函数包括一些辅助函数，如`open_随便你想要叫什么的文件`、`pcap_错的_不是_脏的_打开`和`pcap_get_contiguous_elements`。

7. 启动异步SNF捕获：程序定义了一个名为`sniff`的函数，用于在SNF设备上捕获数据包。这个函数创建一个新的SNF数据包，并在数据包上执行TCP或UDP注入。这使得我们能够捕获到SNF设备上未知的数据流量。

8. 循环运行：程序定义了一个名为`main`的函数，该函数包含程序的主要执行逻辑。在循环中，程序调用`sniff`函数来捕获数据包，并将捕获到的数据包打印出来。

9. 支持选项：程序还定义了一个名为`options`的函数，该函数用于设置或清空选定的选项。其中包括`-n`选项，用于指定SNF设备的名称，以及`-t`选项，用于指定SNF数据包的保留时间。

10. 包含一些常见的头文件：最后，程序还包含了一些常见的头文件，如`<sys/types.h>`和`<unistd.h>`，这些头文件定义了基本的数据类型和标准库函数。


```cpp
#include <sys/mman.h>
#include <sys/socket.h>
#include <sys/types.h>
#include <unistd.h>
#endif /* !_WIN32 */

#include <snf.h>
#if SNF_VERSION_API >= 0x0003
#define SNF_HAVE_INJECT_API
#endif

#include "pcap-int.h"
#include "pcap-snf.h"

/*
 * Private data for capturing on SNF devices.
 */
```



这段代码定义了一个名为 `pcap_snf` 的结构体，包含了以下成员：

- `snf_handle_t` 是一个指向 `snf_handle` 函数的指针，该函数用于创建和管理 `snf_ring` 结构体的数据链路。
- `snf_ring_t` 是一个指向 `snf_ring` 结构体的指针，该结构体定义了数据链路的构成部分，如 `sc`、`udp` 或 `udp64` 等。
- `snf_inject_t` 是一个指向 `snf_inject` 函数的指针，如果 `inject` 参数被使用，则该函数用于将数据包注入到数据链路中。
- `int` 类型的 `snf_timeout` 表示数据包注入超时的时间，单位为秒。
- `int` 类型的 `snf_boardnum` 表示将数据包注入到哪个 `snf_board` 上，其中 `snf_board` 是一个定义在 `pcap_snf_t` 结构体中的结构体，用于指定数据包发送的目标。

`pcap_snf_t` 是另一个名为 `pcap_snf` 的结构体，其定义了 `snf_handle_t` 和 `snf_ring_t` 成员，但没有定义 `snf_inject_t` 和 `snf_timeout` 成员。

因此，这个结构体定义了一个用于创建和管理数据链路的投资驱动网络模块(`pcap_snf`)的数据链路部分，并支持将数据包通过 `snf_inject` 函数注入到数据链路中。


```cpp
struct pcap_snf {
	snf_handle_t snf_handle; /* opaque device handle */
	snf_ring_t   snf_ring;   /* opaque device ring handle */
#ifdef SNF_HAVE_INJECT_API
	snf_inject_t snf_inj;    /* inject handle, if inject is used */
#endif
	int          snf_timeout;
	int          snf_boardnum;
};

static int
snf_set_datalink(pcap_t *p, int dlt)
{
	p->linktype = dlt;
	return (0);
}

```

这段代码是一个用于计算Pcap统计信息的函数，名为“snf_pcap_stats”。它接受两个参数：pcap_t类型的数据名为p，指向pcap_stat类型的结构名为ps。

函数内部首先定义了一个名为stats的结构体，该结构体包含一个指向snf_ring_stats类型的指针snfps，以及一些与统计信息相关的变量。

接着函数内部使用if语句判断前一个函数是否成功执行，成功执行后执行函数内部的代码，即输出ps结构体的值。具体来说，函数先尝试从snfps的snf_ring中获取统计信息，如果获取失败，则输出错误信息；否则，函数计算出ps结构体中对应统计变量的值，并输出ps结构体。


```cpp
static int
snf_pcap_stats(pcap_t *p, struct pcap_stat *ps)
{
	struct snf_ring_stats stats;
	struct pcap_snf *snfps = p->priv;
	int rc;

	if ((rc = snf_ring_getstats(snfps->snf_ring, &stats))) {
		pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
		    rc, "snf_get_stats");
		return -1;
	}
	ps->ps_recv = stats.ring_pkt_recv + stats.ring_pkt_overflow;
	ps->ps_drop = stats.ring_pkt_overflow;
	ps->ps_ifdrop = stats.nic_pkt_overflow + stats.nic_pkt_bad;
	return 0;
}

```

这段代码是一个用于清理OpenWrt中snf（Simple Network Forwarding Library）模块的函数，当它被调用时，会执行以下操作：

1. 检查模块是否支持注入API，如果支持，则调用`snf_inject_close`函数关闭注入的API。
2. 关闭ps模块使用的链表（pcap链表）。
3. 关闭ps模块本身。
4. 调用`pcap_cleanup_live_common`函数，清除当前链表中的数据，释放链表内存，并关闭链表头指针。

总的作用是使SNF模块处于干净的状态，释放之前分配的资源。


```cpp
static void
snf_platform_cleanup(pcap_t *p)
{
	struct pcap_snf *ps = p->priv;

#ifdef SNF_HAVE_INJECT_API
	if (ps->snf_inj)
		snf_inject_close(ps->snf_inj);
#endif
	snf_ring_close(ps->snf_ring);
	snf_close(ps->snf_handle);
	pcap_cleanup_live_common(p);
}

static int
```



这两段代码是用于设置或取消 pcap 包捕获 Non-Blocking 模式的时间限制。

`snf_getnonblock(pcap_t *p)`函数返回一个指向 pcap 结构体中的 `ps` 成员的指针，并且根据设置的 non-blocking 模式的时间限制来返回一个布尔值(0 或 1)。

`snf_setnonblock(pcap_t *p, int nonblock)`函数接受一个 pcap 结构体和一个非阻塞模式选项设置(非阻塞模式时间限制)作为参数。函数首先检查非阻塞模式是否设置，如果是，则将 `ps->snf_timeout` 设置为 0。否则，它检查是否有设置超时时间(如果有的话)，如果是，则将 `ps->snf_timeout` 设置为超时时间。否则，它将 `ps->snf_timeout` 设置为设置的非阻塞模式时间限制。最后，函数返回 0(成功)或 1(失败)。


```cpp
snf_getnonblock(pcap_t *p)
{
	struct pcap_snf *ps = p->priv;

	return (ps->snf_timeout == 0);
}

static int
snf_setnonblock(pcap_t *p, int nonblock)
{
	struct pcap_snf *ps = p->priv;

	if (nonblock)
		ps->snf_timeout = 0;
	else {
		if (p->opt.timeout <= 0)
			ps->snf_timeout = -1; /* forever */
		else
			ps->snf_timeout = p->opt.timeout;
	}
	return (0);
}

```

这段代码定义了一个名为 `snf_timestamp_to_timeval` 的函数，它的输入参数包括一个 `ts_nanosec` 类型的整数表示时间戳，一个 `tstamp_precision` 类型的整数表示时间戳精度，函数的作用是将给定的时间戳 `ts_nanosec` 转换为时间值 `timeval` 并返回。

函数内部首先定义了一个名为 `zero_timeval` 的结构体，包含两个成员：一个代表没有时间的 `zero_timeval`，另一个是保留的原子类型 `const static struct timeval zero_timeval`。

接着定义了一个 `if` 语句，判断给定的 `ts_nanosec` 是否为零，如果是，则直接返回 `zero_timeval`，否则开始计算时间值。

接下来定义了一个 `tv_nsec` 变量，用于存储时间戳 `ts_nanosec` 除以 `_NSEC_PER_SEC` 得到的数值，如果没有这个值，则需要计算并初始化。

然后判断时间戳精度 `tstamp_precision`，如果 `tstamp_precision` 等于 `PCAP_TSTAMP_PRECISION_NANO`，则直接将 `ts_nanosec` 作为时间值输入，否则需要将 `ts_nanosec` 除以 1000 并取整，再作为时间值输入。

最后，函数返回结构体 `struct timeval` 类型的变量 `tv`，包含 `ts_nanosec` 和 `tv_usec` 两个成员。


```cpp
#define _NSEC_PER_SEC 1000000000

static inline
struct timeval
snf_timestamp_to_timeval(const int64_t ts_nanosec, const int tstamp_precision)
{
	struct timeval tv;
	long tv_nsec;
        const static struct timeval zero_timeval;

        if (ts_nanosec == 0)
                return zero_timeval;

	tv.tv_sec = ts_nanosec / _NSEC_PER_SEC;
	tv_nsec = (ts_nanosec % _NSEC_PER_SEC);

	/* libpcap expects tv_usec to be nanos if using nanosecond precision. */
	if (tstamp_precision == PCAP_TSTAMP_PRECISION_NANO)
		tv.tv_usec = tv_nsec;
	else
		tv.tv_usec = tv_nsec / 1000;

	return tv;
}

```



This is a function definition for `pcap_read_pkts()` which is a part of the `libpcap` library in C.

This function reads packets from the given input stream and returns the number of packets successfully read.

It uses a combination of `snf_ring_recv()` to receive packets from the input stream and `pcap_fmt_errmsg_for_errno()` to handle errors.

The function reads packets in the order they were sent and uses a variable `pcaplen` to keep track of the number of bytes in each packet.

It also uses a variable `errbuf` to store the error-related information in case of an error occurs during the read operation.

The function returns `n` which is the number of packets successfully read.

It is important to note that the function may block during the read operation if the input stream is a BufferedStream and the number of bytes to be read exceeds the available number of bytes in the buffer.


```cpp
static int
snf_read(pcap_t *p, int cnt, pcap_handler callback, u_char *user)
{
	struct pcap_snf *ps = p->priv;
	struct pcap_pkthdr hdr;
	int i, flags, err, caplen, n;
	struct snf_recv_req req;
	int nonblock, timeout;

	if (!p)
		return -1;

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

	n = 0;
	timeout = ps->snf_timeout;
	while (n < cnt) {
		/*
		 * Has "pcap_breakloop()" been called?
		 */
		if (p->break_loop) {
			if (n == 0) {
				p->break_loop = 0;
				return (-2);
			} else {
				return (n);
			}
		}

		err = snf_ring_recv(ps->snf_ring, timeout, &req);

		if (err) {
			if (err == EBUSY || err == EAGAIN) {
				return (n);
			}
			else if (err == EINTR) {
				timeout = 0;
				continue;
			}
			else {
				pcap_fmt_errmsg_for_errno(p->errbuf,
				    PCAP_ERRBUF_SIZE, err, "snf_read");
				return -1;
			}
		}

		caplen = req.length;
		if (caplen > p->snapshot)
			caplen = p->snapshot;

		if ((p->fcode.bf_insns == NULL) ||
		     pcap_filter(p->fcode.bf_insns, req.pkt_addr, req.length, caplen)) {
			hdr.ts = snf_timestamp_to_timeval(req.timestamp, p->opt.tstamp_precision);
			hdr.caplen = caplen;
			hdr.len = req.length;
			callback(user, &hdr, req.pkt_addr);
			n++;
		}

		/* After one successful packet is received, we won't block
		* again for that timeout. */
		if (timeout != 0)
			timeout = 0;
	}
	return (n);
}

```

这段代码是一个用于注入数据到网络套接字（socket）的函数。它接受一个pcap_t类型的数据指针（pcap_t *p），一个指向数据缓冲区的指针（const void *buf），以及一个数据缓冲区的大小（int size）。

函数首先检查ps指向的套接字是否已经打开，如果不是，则函数调用snf_inject_open函数打开套接字。如果打开成功，函数调用snf_inject_send函数将数据缓冲区中的数据发送到套接字中。

如果发送失败，函数将返回失败的状态码，或者从数据缓冲区中读取数据并返回。

函数中定义了一个静态函数，名为snf_inject，可以在程序中中被调用。


```cpp
static int
snf_inject(pcap_t *p, const void *buf _U_, int size _U_)
{
#ifdef SNF_HAVE_INJECT_API
	struct pcap_snf *ps = p->priv;
	int rc;
	if (ps->snf_inj == NULL) {
		rc = snf_inject_open(ps->snf_boardnum, 0, &ps->snf_inj);
		if (rc) {
			pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
			    rc, "snf_inject_open");
			return (-1);
		}
	}

	rc = snf_inject_send(ps->snf_inj, -1, 0, buf, size);
	if (!rc) {
		return (size);
	}
	else {
		pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
		    rc, "snf_inject_send");
		return (-1);
	}
```

It looks like there's an error in the code, but it's not clear where the error is. The error message is being generated by `pcap_fmt_errmsg_for_errno()`, but it's not clear what went wrong. It could be an error with the `snf_open()` function, a problem with the `getenv()` call, a `snf_ring_open_id()` failure, a `snf_start()` error, or an issue with the `select()` and `poll()` functions. Without more information, it's difficult to say for sure where the error is occurring.


```cpp
#else
	pcap_strlcpy(p->errbuf, "Sending packets isn't supported with this snf version",
	    PCAP_ERRBUF_SIZE);
	return (-1);
#endif
}

static int
snf_activate(pcap_t* p)
{
	struct pcap_snf *ps = p->priv;
	char *device = p->opt.device;
	const char *nr = NULL;
	int err;
	int flags = -1, ring_id = -1;

	if (device == NULL) {
		snprintf(p->errbuf, PCAP_ERRBUF_SIZE, "device is NULL");
		return -1;
	}

	/* In Libpcap, we set pshared by default if NUM_RINGS is set to > 1.
	 * Since libpcap isn't thread-safe */
	if ((nr = getenv("SNF_FLAGS")) && *nr)
		flags = strtol(nr, NULL, 0);
	else if ((nr = getenv("SNF_NUM_RINGS")) && *nr && atoi(nr) > 1)
		flags = SNF_F_PSHARED;
	else
		nr = NULL;


        /* Allow pcap_set_buffer_size() to set dataring_size.
         * Default is zero which allows setting from env SNF_DATARING_SIZE.
         * pcap_set_buffer_size() is in bytes while snf_open() accepts values
         * between 0 and 1048576 in Megabytes. Values in this range are
         * mapped to 1MB.
         */
	err = snf_open(ps->snf_boardnum,
			0, /* let SNF API parse SNF_NUM_RINGS, if set */
			NULL, /* default RSS, or use SNF_RSS_FLAGS env */
                        (p->opt.buffer_size > 0 && p->opt.buffer_size < 1048576) ? 1048576 : p->opt.buffer_size, /* default to SNF_DATARING_SIZE from env */
			flags, /* may want pshared */
			&ps->snf_handle);
	if (err != 0) {
		pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
		    err, "snf_open failed");
		return -1;
	}

	if ((nr = getenv("SNF_PCAP_RING_ID")) && *nr) {
		ring_id = (int) strtol(nr, NULL, 0);
	}
	err = snf_ring_open_id(ps->snf_handle, ring_id, &ps->snf_ring);
	if (err != 0) {
		pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
		    err, "snf_ring_open_id(ring=%d) failed", ring_id);
		return -1;
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

	if (p->opt.timeout <= 0)
		ps->snf_timeout = -1;
	else
		ps->snf_timeout = p->opt.timeout;

	err = snf_start(ps->snf_handle);
	if (err != 0) {
		pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
		    err, "snf_start failed");
		return -1;
	}

	/*
	 * "select()" and "poll()" don't work on snf descriptors.
	 */
```

这段代码是一个 C 语言编写的 Preprocessor 头文件，它定义了一系列预处理指令，用于影响程序在编译前的行为。以下是该代码的主要作用：

1. 定义了 `_WIN32` 宏，表示这是一个 Windows 系统上的预处理指令。如果没有定义 `_WIN32` 宏，该预处理指令将不再被识别。
2. 定义了 `p` 指向的下一个可选择文件的描述符（fd）为 -1，即它不可用。
3. 定义了 `p` 的 `linktype` 为 `DLT_EN10MB`，表示这是一个 En10MB 数据链路单元。
4. 定义了 `p` 的 `read_op` 为 `snf_read`，`inject_op` 为 `snf_inject`，`setfilter_op` 为 `install_bpf_program`，`setdirection_op` 为 `NULL`，表示这是一个没有数据链路分配操作的预处理指令。
5. 定义了 `p` 的 `set_datalink_op` 为 `snf_set_datalink`，表示这是一个数据链路分配操作的预处理指令。
6. 定义了 `p` 的 `getnonblock_op` 为 `snf_getnonblock`，表示这是一个非阻塞数据链路分配操作的预处理指令。
7. 定义了 `p` 的 `stats_op` 为 `snf_pcap_stats`，表示这是一个统计数据链路统计信息的预处理指令。
8. 定义了 `p` 的 `cleanup_op` 为 `snf_platform_cleanup`，表示这是一个清理操作的预处理指令。
9. 定义了 `ps` 指向的 `snf_inj` 变量，表示一个指向预处理器函数的指针，用于将 `snf_inj` 函数作为预处理指令。


```cpp
#ifndef _WIN32
	p->selectable_fd = -1;
#endif /* !_WIN32 */
	p->linktype = DLT_EN10MB;
	p->read_op = snf_read;
	p->inject_op = snf_inject;
	p->setfilter_op = install_bpf_program;
	p->setdirection_op = NULL; /* Not implemented.*/
	p->set_datalink_op = snf_set_datalink;
	p->getnonblock_op = snf_getnonblock;
	p->setnonblock_op = snf_setnonblock;
	p->stats_op = snf_pcap_stats;
	p->cleanup_op = snf_platform_cleanup;
#ifdef SNF_HAVE_INJECT_API
	ps->snf_inj = NULL;
```

			/*
			 * Update the status to indicate the device is running
			 * or not.  This is a place to put it in the
			 * "auto" or "man" category.
			 */
			int state;

			state = get_device_status(dev,
			    PCAP_IF_CONNECTION_STATUS_CONNECTED,
			    PCAP_IF_CONNECTION_STATUS_DISCONNECTED,
			    errbuf, sizeof(errbuf));

			switch (state) {
				case PCAP_IF_CONNECTION_STATUS_CONNECTED:
					desc = strdup(
					    "Connected to PCI device with tag "
					    "%s"
					    , dev->desc);
					break;
					case PCAP_IF_CONNECTION_STATUS_DISCONNECTED:
						desc = strdup(
						    "Disconnected from PCI device with tag "
						    "%s"
						    , dev->desc);
						break;
						case PCAP_IF_CONNECTION_STATUS_CHANGING:
							desc = strdup(
							    "Changing PCI device with tag "
								    "%s"
								, dev->desc);
							break;
							default:
							desc = strdup(errbuf);
							break;
				}

				/*
				 * For devices like this, where the tag
				 * is not a PCI device but a serial
				 * device, let's make sure to set the status
				 * to 'man' in case the PCI device is
				 * not being used.
				 */
				if (dev->tag == PCI_TAG_PCI)
					dev->tag = 0;

				/*
				 * PCI devices don't have a tag, so we don't
				 * need to set the status for them.
				 */
				}

				free(dev->desc);
				dev->description = desc;
			}
		}
	}
```cpp

		}
	}

	return 0;
```

		}

		}
	}

	if (errno != 0)
			goto XXX;

		}

		}

		}

		;

		}

		if (dev == NULL) {
			errno = -1;
			goto XXX;
		}

		}

		);

		if (errno != 0)
			goto XXX;

		}

		if (dev == NULL) {
			errno = -1;
			goto XXX;
		}

		}

		);

		if (errno != 0)
			goto XXX;

		}

		if (dev->status == PCAP_IF_CONNECTION_STATUS_CONNECTED)
			str = strdup(
				"Enabled on PCI device, device "
					errbuf[PCAP_IF_CONNECTION_STATUS_MASK]
					" with tag "
						errbuf[PCAP_IF_CONNECTION_STATUS_TAG]
					);
				free(errbuf);
				errno = 0;
				return 0;
			} else if (dev->status == PCAP_IF_CONNECTION_STATUS_DISCONNECTED)
				str = strdup(
					"Disconnected from PCI device with tag "
						errbuf[PCAP_IF_CONNECTION_STATUS_MASK]
						errbuf[PCAP_IF_CONNECTION_STATUS_TAG]
					);
					free(errbuf);
					errno = 0;
					return 0;
			} else if (dev->status == PCAP_IF_CONNECTION_STATUS_CHANGING)
				str = strdup(
					"Changing PCI device with tag "
						errbuf[PCAP_IF_CONNECTION_STATUS_MASK]
						errbuf[PCAP_IF_CONNECTION_STATUS_TAG]
					);
						free(errbuf);
						errno = 0;
						return 0;
			}

			return 0;
		}
	}

X:
		if (errno != 0)
			goto XXX;

		}

		if (dev == NULL) {
			errno = -1;
			goto XXX;
		}

		}

		/*
		 * Add the port to the bitmask.
		 */
		if (merge)
			allports |= 1 << ifa->snf_ifa_portnum;
		break;

		end:
			if (dev->status == PCAP_IF_CONNECTION_STATUS_CONNECTED)
				str = strdup(
					"Enabled on PCI device, device "
		


```cpp
#endif
	return 0;
}

#define MAX_DESC_LENGTH 128
int
snf_findalldevs(pcap_if_list_t *devlistp, char *errbuf)
{
	pcap_if_t *dev;
#ifdef _WIN32
	struct sockaddr_in addr;
#endif
	struct snf_ifaddrs *ifaddrs, *ifa;
	char name[MAX_DESC_LENGTH];
	char desc[MAX_DESC_LENGTH];
	int ret, allports = 0, merge = 0;
	const char *nr = NULL;

	if (snf_init(SNF_VERSION_API)) {
		(void)snprintf(errbuf, PCAP_ERRBUF_SIZE,
		    "snf_getifaddrs: snf_init failed");
		return (-1);
	}

	if (snf_getifaddrs(&ifaddrs) || ifaddrs == NULL)
	{
		pcap_fmt_errmsg_for_errno(errbuf, PCAP_ERRBUF_SIZE,
		    errno, "snf_getifaddrs");
		return (-1);
	}
	if ((nr = getenv("SNF_FLAGS")) && *nr) {
		errno = 0;
		merge = strtol(nr, NULL, 0);
		if (errno) {
			(void)snprintf(errbuf, PCAP_ERRBUF_SIZE,
				"snf_getifaddrs: SNF_FLAGS is not a valid number");
			return (-1);
		}
		merge = merge & SNF_F_AGGREGATE_PORTMASK;
	}

	for (ifa = ifaddrs; ifa != NULL; ifa = ifa->snf_ifa_next) {
		/*
		 * Myricom SNF adapter ports may appear as regular
		 * network interfaces, which would already have been
		 * added to the list of adapters by pcap_platform_finddevs()
		 * if this isn't an SNF-only version of libpcap.
		 *
		 * Our create routine intercepts pcap_create() calls for
		 * those interfaces and arranges that they will be
		 * opened using the SNF API instead.
		 *
		 * So if we already have an entry for the device, we
		 * don't add an additional entry for it, we just
		 * update the description for it, if any, to indicate
		 * which snfN device it is.  Otherwise, we add an entry
		 * for it.
		 *
		 * In either case, if SNF_F_AGGREGATE_PORTMASK is set
		 * in SNF_FLAGS, we add this port to the bitmask
		 * of ports, which we use to generate a device
		 * we can use to capture on all ports.
		 *
		 * Generate the description string.  If port aggregation
		 * is set, use 2^{port number} as the unit number,
		 * rather than {port number}.
		 *
		 * XXX - do entries in this list have IP addresses for
		 * the port?  If so, should we add them to the
		 * entry for the device, if they're not already in the
		 * list of IP addresses for the device?
		 */
		(void)snprintf(desc,MAX_DESC_LENGTH,"Myricom %ssnf%d",
			merge ? "Merge Bitmask Port " : "",
			merge ? 1 << ifa->snf_ifa_portnum : ifa->snf_ifa_portnum);
		/*
		 * Add the port to the bitmask.
		 */
		if (merge)
			allports |= 1 << ifa->snf_ifa_portnum;
		/*
		 * See if there's already an entry for the device
		 * with the name ifa->snf_ifa_name.
		 */
		dev = find_dev(devlistp, ifa->snf_ifa_name);
		if (dev != NULL) {
			/*
			 * Yes.  Update its description.
			 */
			char *desc_str;

			desc_str = strdup(desc);
			if (desc_str == NULL) {
				pcap_fmt_errmsg_for_errno(errbuf,
				    PCAP_ERRBUF_SIZE, errno,
				    "snf_findalldevs strdup");
				return -1;
			}
			free(dev->description);
			dev->description = desc_str;
		} else {
			/*
			 * No.  Add an entry for it.
			 *
			 * XXX - is there a notion of "up" or "running",
			 * and can we determine whether something's
			 * plugged into the adapter and set
			 * PCAP_IF_CONNECTION_STATUS_CONNECTED or
			 * PCAP_IF_CONNECTION_STATUS_DISCONNECTED?
			 */
			dev = add_dev(devlistp, ifa->snf_ifa_name, 0, desc,
			    errbuf);
			if (dev == NULL)
				return -1;
```

这段代码是一个用于在Windows操作系统中根据设备名称获取IP地址的代码。它首先通过调用`inet_pton`函数将设备名称转换为IP地址。如果转换成功，将添加到`dev`结构中的`sin_family`成员，并将设备名称设置为输入的设备名称。然后，它尝试将添加的IP地址添加到网络设备驱动程序的`dev`结构中，并返回一个非负值。如果转换或添加失败，将返回一个负值或错误代码。


```cpp
#ifdef _WIN32
			/*
			 * On Windows, fill in IP# from device name
			 */
                        ret = inet_pton(AF_INET, dev->name, &addr.sin_addr);
                        if (ret == 1) {
				/*
				 * Successful conversion of device name
				 * to IPv4 address.
				 */
				addr.sin_family = AF_INET;
				if (add_addr_to_dev(dev, &addr, sizeof(addr),
				    NULL, 0, NULL, 0, NULL, 0, errbuf) == -1)
					return -1;
                        } else if (ret == -1) {
				/*
				 * Error.
				 */
				pcap_fmt_errmsg_for_errno(errbuf,
				    PCAP_ERRBUF_SIZE, errno,
				    "sinf_findalldevs inet_pton");
                                return -1;
                        }
```

这段代码是一个 Linux 系统的设备驱动程序，名为 "myricom merge driver"。它实现了 port aggregation，即将多个串口连接汇聚成一个单独的串口，从而实现更高的数据传输速率和更好的网络适应性。以下是这段代码的主要作用和功能：

1. 定义了一个头文件 "myricom merge driver"，该头文件中定义了一个名为 "snf_freeifaddrs" 的函数。

2. 在头文件中定义了一个名为 "merge" 的宏，其值为 true，表示实现 port aggregation。

3. 如果选择了 true，则在函数中创建一个新的 snfX 设备条目，其中包含所有端口的位掩。

4. 如果选择了 false，则在函数中添加一个名为 "snf%d" 的宏，其中所有端口的位掩，并创建一个新的设备条目，其中包含所有端口的位掩。

5. 如果 port aggregation 被启用，则在函数中添加一个名为 "add_dev" 的函数，用于将一个设备添加到 devlist 中。如果该函数实现成功，则将设备的名称、描述和错误缓冲区作为参数传递给该函数。

6. 如果添加设备失败，则在函数中返回 -1，并打印错误缓冲区中的错误信息。

7. 在 if 语句中检查是否启用了 port aggregation，如果是，则执行以下操作：

  - 创建一个新的条目，其中包含所有端口的位掩。

  - 创建一个新的条目，其中包含所有端口的位掩，并指定连接状态为 true。

  - 尝试将设备添加到 devlist 中，并检查是否成功。

  - 如果成功添加设备，则返回 0，否则返回 -1，并打印错误缓冲区中的错误信息。


```cpp
#endif _WIN32
		}
	}
	snf_freeifaddrs(ifaddrs);
	/*
	 * Create a snfX entry if port aggregation is enabled
	 */
	if (merge) {
		/*
		 * Add a new entry with all ports bitmask
		 */
		(void)snprintf(name,MAX_DESC_LENGTH,"snf%d",allports);
		(void)snprintf(desc,MAX_DESC_LENGTH,"Myricom Merge Bitmask All Ports snf%d",
			allports);
		/*
		 * XXX - is there any notion of "up" and "running" that
		 * would apply to this device, given that it handles
		 * multiple ports?
		 *
		 * Presumably, there's no notion of "connected" vs.
		 * "disconnected", as "is this plugged into a network?"
		 * would be a per-port property.
		 */
		if (add_dev(devlistp, name,
		    PCAP_IF_CONNECTION_STATUS_NOT_APPLICABLE, desc,
		    errbuf) == NULL)
			return (-1);
		/*
		 * XXX - should we give it a list of addresses with all
		 * the addresses for all the ports?
		 */
	}

	return 0;
}

```

This function appears to be a part of a network protocol stack, and it appears to be used to create a pseudo header that includes the network address and a board number for a network interface device.

The function takes a single parameter, a string device name, and returns a pointer to a pointer to a structure representing a pseudo header that includes the network address and board number.

The function first checks whether the device name is recognized by the function's implementation. If the device is not recognized, the function returns NULL.

If the device is recognized, the function creates a pointer to a structure representing a pseudo header, and sets the pointer's activate\_op and snf\_boardnum fields to the functions that should be called to configure the pseudo header.

Finally, the function returns the pointer to the pseudo header structure.

Note: This function should be called from a function that has an完善的 interface for configuring the network address and board number of the interface device. Additionally, the PCAP library is used in the function, which is a library for creating and manipulating network packet captures.


```cpp
pcap_t *
snf_create(const char *device, char *ebuf, int *is_ours)
{
	pcap_t *p;
	int boardnum = -1;
	struct snf_ifaddrs *ifaddrs, *ifa;
	size_t devlen;
	struct pcap_snf *ps;

	if (snf_init(SNF_VERSION_API)) {
		/* Can't initialize the API, so no SNF devices */
		*is_ours = 0;
		return NULL;
	}

	/*
	 * Match a given interface name to our list of interface names, from
	 * which we can obtain the intended board number
	 */
	if (snf_getifaddrs(&ifaddrs) || ifaddrs == NULL) {
		/* Can't get SNF addresses */
		*is_ours = 0;
		return NULL;
	}
	devlen = strlen(device) + 1;
	ifa = ifaddrs;
	while (ifa) {
		if (strncmp(device, ifa->snf_ifa_name, devlen) == 0) {
			boardnum = ifa->snf_ifa_boardnum;
			break;
		}
		ifa = ifa->snf_ifa_next;
	}
	snf_freeifaddrs(ifaddrs);

	if (ifa == NULL) {
		/*
		 * If we can't find the device by name, support the name "snfX"
		 * and "snf10gX" where X is the board number.
		 */
		if (sscanf(device, "snf10g%d", &boardnum) != 1 &&
		    sscanf(device, "snf%d", &boardnum) != 1) {
			/* Nope, not a supported name */
			*is_ours = 0;
			return NULL;
		}
	}

	/* OK, it's probably ours. */
	*is_ours = 1;

	p = PCAP_CREATE_COMMON(ebuf, struct pcap_snf);
	if (p == NULL)
		return NULL;
	ps = p->priv;

	/*
	 * We support microsecond and nanosecond time stamps.
	 */
	p->tstamp_precision_list = malloc(2 * sizeof(u_int));
	if (p->tstamp_precision_list == NULL) {
		pcap_fmt_errmsg_for_errno(ebuf, PCAP_ERRBUF_SIZE, errno,
		    "malloc");
		pcap_close(p);
		return NULL;
	}
	p->tstamp_precision_list[0] = PCAP_TSTAMP_PRECISION_MICRO;
	p->tstamp_precision_list[1] = PCAP_TSTAMP_PRECISION_NANO;
	p->tstamp_precision_count = 2;

	p->activate_op = snf_activate;
	ps->snf_boardnum = boardnum;
	return p;
}

```

这段代码是一个用于检查pcap库是否支持SNF（SuperNOC中间件）设备的工具函数。它包含两个函数：pcap_platform_finddevs和mem_alloc。以下是它们的解释：

1. pcap_platform_finddevs函数：

该函数用于返回在pcap库中支持SNF设备的个数。在函数内部，使用if_list_t结构来存储所有的网络接口。然后使用pcap_if_list_size函数计算if_list_t中元素的个数。最后，通过比较if_list_t数组长度与函数内部定义的整数变量SNF_COUNT，来决定是否支持SNF设备。如果没有找到SNF设备，函数返回0。

2. mem_alloc函数：

该函数用于在内存上分配空间，用于存储设备数据。它需要提供两个参数：errbuf和output_len。errbuf是一个用于存储错误信息的字符数组，而output_len是一个整数，用于指示错误信息中实际可用的字符数。函数返回这个错误信息字符数组以及分配的内存空间大小。

总之，这段代码的作用是检查pcap库是否支持SNF设备，并分配足够的内存空间用于存储设备数据。


```cpp
#ifdef SNF_ONLY
/*
 * This libpcap build supports only SNF cards, not regular network
 * interfaces..
 */

/*
 * There are no regular interfaces, just SNF interfaces.
 */
int
pcap_platform_finddevs(pcap_if_list_t *devlistp, char *errbuf)
{
	return (0);
}

```

这段代码定义了两个函数，旨在创建一个适配器并输出错误信息。以下是代码的详细解释：

1. `pcap_create_interface()`函数：

该函数的作用是尝试创建一个 regular（即串口）接口，并输出错误信息。通过调用 `PCAP_CREATE_INTERFACE()` 函数实现。

函数原型如下：
```cppc
pcap_t *pcap_create_interface(const char *device, char *errbuf);
```
参数说明：
- `device`：接口名称，通常为 "serial0"。
- `errbuf`：错误信息，用于存储 PCAP_ERRBUF_SIZE 后的错误字符串。

函数实现：
```cppc
Snprintf(errbuf, PCAP_ERRBUF_SIZE,
   "This version of libpcap only supports SNF cards");
```
这段代码仅实现了一个简单的错误处理。当尝试创建其他类型的接口（如串口）时，将抛出错误并输出错误信息。

2. `const char *libpcap_version()`函数：

该函数返回 PCAP 库的版本字符串。通过调用 `PCAP_VERSION()` 函数实现。

函数原型如下：
```cppc
const char *PCAP_VERSION();
```

```cppc
const char *PCAP_VERSION()
{
   return version;
}
```
函数实现：
```cppsql
const char *PCAP_VERSION()
{
   static const char *version = "0.9.4";
   return version;
}
```
这段代码仅实现了一个简单的函数，用于返回 PCAP 库的版本字符串。


```cpp
/*
 * Attempts to open a regular interface fail.
 */
pcap_t *
pcap_create_interface(const char *device, char *errbuf)
{
	snprintf(errbuf, PCAP_ERRBUF_SIZE,
	    "This version of libpcap only supports SNF cards");
	return NULL;
}

/*
 * Libpcap version string.
 */
const char *
```

这段代码是一个C语言函数，名为“pcap_lib_version(void)”。它返回当前pcap库版本号，格式为“字符串”。

具体来说，函数内部首先定义了一个名为“PCAP_VERSION_STRING”的常量，它是一个字符串，包含了pcap库的版本信息。然后，函数调用了一个名为“PCAP_VERSION_NUMBER”的函数，该函数获取当前系统时间，将其作为整数返回，再将该整数转换为字符串，与PCAP_VERSION_STRING拼接成完整版本号。最后，函数将返回的版本号（字符串）作为printf函数的第一个参数输出，同时也设置了第二个参数，使得printf函数在输出时会打印引号。


```cpp
pcap_lib_version(void)
{
	return (PCAP_VERSION_STRING " (SNF-only)");
}
#endif

```

# `libpcap/pcap-snit.c`

这段代码是一个C语言的函数，它定义了一个名为“external工具”的函数，并为其定义了一些元数据。

具体来说，这段代码定义了一个名为“external”的函数，参数个数为无限制的。这个函数被声明为“external”函数，意味着这个函数是通过管道（$$<$$）或符号链接（$$>$$）加载到内存中的，而不是在内存中定义的。这个函数的实现将在编译时生成一个包含参数列表、函数名称和函数体字的符号表（.sym）。

此外，这段代码定义了一些元数据，其中包括函数的文件名、作者、版本号、描述等信息。这些元数据将在函数被加载到内存时被初始化，并在函数被调用时被读取和打印。

最后，这段代码还指出这个软件是由“The Regents of the University of California.  All rights reserved.”授权的，允许对软件进行修改和分发，但需要包含原始版权通知和说明，并且在涉及到二进制形式的软件分发时也需要包含原始版权通知和说明。


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
 * Modifications made to accommodate the new SunOS4.0 NIT facility by
 * Micky Liu, micky@cunixc.cc.columbia.edu, Columbia University in May, 1989.
 * This module now handles the STREAMS based NIT.
 */

```

这段代码是一个C语言程序，它主要用于检查系统是否支持某种配置文件(CONFIG_H)。

首先，它通过一个宏定义来检查是否支持CONFIG_H。如果支持，它就会包含宏定义中的所有包含“#include <config.h>”的函数和变量。

否则，它就不会包含这些函数和变量，因为它无法确定系统是否支持CONFIG_H。

接下来，它包含了一些标准输入输出函数，例如`<sys/types.h>`、`<sys/time.h>`、`<sys/timeb.h>`、`<sys/dir.h>`、`<sys/fcntlcom.h>`、`<sys/file.h>`、`<sys/ioctl.h>`、`<sys/stropts.h>`、`<net/if.h>`。

然后，它包含了一些用于获取TCP连接信息的标准函数，例如`<sys/socket.h>`、`<sys/stropts.h>`、`<net/if.h>`。

接着，它包含了一些用于获取IP地址信息的标准函数，例如`<sys/types.h>`、`<sys/ip.h>`。

最后，根据上面提到的信息，我们可以得出结论，即这段代码的作用是检查系统是否支持CONFIG_H，并包含用于获取TCP连接信息和IP地址信息的标准函数。


```cpp
#ifdef HAVE_CONFIG_H
#include <config.h>
#endif

#include <sys/types.h>
#include <sys/time.h>
#include <sys/timeb.h>
#include <sys/dir.h>
#include <sys/fcntlcom.h>
#include <sys/file.h>
#include <sys/ioctl.h>
#include <sys/socket.h>
#include <sys/stropts.h>

#include <net/if.h>
```

这段代码是一个网络协议栈的示例，定义了网络头信息和数据包信息。具体解释如下：

1. `#include <net/nit.h>` 和 `#include <net/nit_if.h>` 定义了网络协议栈和网络接口头文件。`<net/nit.h>` 和 `<net/nit_if.h>` 定义了网络接口，`<net/nit_pf.h>` 和 `<net/nit_buf.h>` 定义了数据包信息。

2. `#include <netinet/in.h>` 和 `#include <netinet/in_systm.h>` 定义了网络头信息和系统头信息。`<netinet/ip.h>` 和 `<netinet/if_ether.h>` 定义了网络头信息。`<netinet/ip_var.h>` 和 `<netinet/udp.h>` 定义了系统头信息。`<netinet/udp_var.h>` 定义了系统头信息。

3. `#include <netinet/tcp.h>` 和 `#include <netinet/tcpip.h>` 定义了传输控制协议(TCP)和传输协议互联网协议(IP)。`<netinet/ip.h>` 和 `<netinet/if_ether.h>` 定义了网络头信息。`<netinet/udp.h>` 和 `<netinet/udp_var.h>` 定义了系统头信息。

4. `#include <string.h>` 和 `#include <stdlib.h>` 定义了字符串处理函数和标准库头文件。

5. `void foo()` 是函数定义，定义了 `foo` 函数，但未定义函数体，因此不能直接使用 `foo` 函数。


```cpp
#include <net/nit.h>
#include <net/nit_if.h>
#include <net/nit_pf.h>
#include <net/nit_buf.h>

#include <netinet/in.h>
#include <netinet/in_systm.h>
#include <netinet/ip.h>
#include <netinet/if_ether.h>
#include <netinet/ip_var.h>
#include <netinet/udp.h>
#include <netinet/udp_var.h>
#include <netinet/tcp.h>
#include <netinet/tcpip.h>

```

这段代码是一个用于 Linux 系统应用程序的网络编程框架，它包括了若干头文件和函数定义。通过引入这个文件，我们可以使用函数函数实现类似系统调用，而不需要关心其原理。这个程序的作用是提供一个简单的框架，让用户能够更方便地在 NIT（网络映射单元）中实现网络编程。以下是这个程序的主要部分：

1. 包含头文件：首先，这个程序包含了系统标准库中的几个头文件，包括 errno.h、stdio.h、string.h 和 unistd.h。这些头文件包含了一些通用的函数定义，对于这个程序的运行是必不可少的。

2. 引入 "pcap-int.h"：这个头文件来源于名为 "pcap-int.h" 的外部源代码文件。这里可能是一个用于实现网络数据捕获的库，它提供了文件读写操作的接口。

3. 引入 "os-proto.h"：同样地，这个头文件来源于 "os-proto.h" 的外部源代码文件。这个头文件可能是一个用于实现操作系统接口的库，它提供了操作系统接口的接口函数。

4. 定义常量：这个程序定义了一些常量，包括 "chunkSize" 和 "nit"，用于设置 NIT 中的数据包缓冲区大小和数据包的缓冲区个数。

5. 函数声明：这个程序中定义了一些函数，用于实现 NIT 中的数据包发送、接收操作。这些函数接受不同类型的数据，包括 char、int 和 void 类型。

6. 包含 "pcap-int.h" 和 "os-proto.h"：为了避免重复，这个程序再次引入了之前定义的头文件。

7. 输出 "OK"，表示程序成功加载并支持 "pcap-int.h" 和 "os-proto.h"：这个输出可以是一个用于显示程序成功信息的字符串，也可以是一个其他程序用来调用这个函数的接口。


```cpp
#include <errno.h>
#include <stdio.h>
#include <string.h>
#include <unistd.h>

#include "pcap-int.h"

#ifdef HAVE_OS_PROTO_H
#include "os-proto.h"
#endif

/*
 * The chunk size for NIT.  This is the amount of buffering
 * done for read calls.
 */
```

这段代码定义了两个头文件，以及一个宏，用于定义NIT中的数据分块大小。

宏CHUNKSIZE表示NIT数据分块的大小，等于2乘以1024字节。

BUFSPACE通过宏定义导入了这个宏，表示一个NIT数据分块所占用的字节数，等于4乘以CHUNKSIZE。

接下来是nit_setflags函数的实现，这个函数是用于在NIT设备上设置或清除标志的。它接收4个整数参数：设备号、功能号、设置标志和清除标志。这个函数的实现比较简单，只是将传递给它的参数进行位运算，然后将结果存储回NIT设备中。

最后定义的结构体pcap_snit用于在NIT设备上捕获数据。它存储了NIT设备的一些统计信息，例如接收和发送的数据包数量，以及分块的起始和结束时间。


```cpp
#define CHUNKSIZE (2*1024)

/*
 * The total buffer space used by NIT.
 */
#define BUFSPACE (4*CHUNKSIZE)

/* Forwards */
static int nit_setflags(int, int, int, char *);

/*
 * Private data for capturing on STREAMS NIT devices.
 */
struct pcap_snit {
	struct pcap_stat stat;
};

```

这段代码是一个静态函数，名为 `pcap_stats_snit`，属于 `pcap_t` 类的实例。它接收两个参数：一个指向 `pcap_snit` 结构体的指针 `ps` 和一个指向 `pcap_stat` 结构体的指针 `psn`。

函数内部首先定义了两个内部变量 `ps_recv` 和 `ps_drop`，它们分别统计 handed 和 dropped packets。这里的统计范围都是不包括因为缓冲区耗尽而导致的 packets。

接下来，通过 `psn->stat` 获取了 `psn` 指向的 `pcap_snit` 实例的统计信息，将其赋值给 `ps` 指向的 `psf` 成员。

最后，函数返回 0，表示成功。


```cpp
static int
pcap_stats_snit(pcap_t *p, struct pcap_stat *ps)
{
	struct pcap_snit *psn = p->priv;

	/*
	 * "ps_recv" counts packets handed to the filter, not packets
	 * that passed the filter.  As filtering is done in userland,
	 * this does not include packets dropped because we ran out
	 * of buffer space.
	 *
	 * "ps_drop" counts packets dropped inside the "/dev/nit"
	 * device because of flow control requirements or resource
	 * exhaustion; it doesn't count packets dropped by the
	 * interface driver, or packets dropped upstream.  As filtering
	 * is done in userland, it counts packets regardless of whether
	 * they would've passed the filter.
	 *
	 * These statistics don't include packets not yet read from the
	 * kernel by libpcap or packets not yet read from libpcap by the
	 * application.
	 */
	*ps = psn->stat;
	return (0);
}

```

This is a code snippet for the Windows Platform that reads data from a network interface device (NID) and sends it to the kernel NIT service. The NID is configured to transmit the data at a specified interval, and this code provides a function for reading the data from the NID and sending it to the kernel NIT service.

The code takes in the interval and the maximum number of bytes to read from the NID. It also reads the data from the NID and sends it to the kernel NIT service.

This code snippet is just a part of the whole program, which should be aware of the whole program's context to work correctly.


```cpp
static int
pcap_read_snit(pcap_t *p, int cnt, pcap_handler callback, u_char *user)
{
	struct pcap_snit *psn = p->priv;
	register int cc, n;
	register u_char *bp, *cp, *ep;
	register struct nit_bufhdr *hdrp;
	register struct nit_iftime *ntp;
	register struct nit_iflen *nlp;
	register struct nit_ifdrops *ndp;
	register int caplen;

	cc = p->cc;
	if (cc == 0) {
		cc = read(p->fd, (char *)p->buffer, p->bufsize);
		if (cc < 0) {
			if (errno == EWOULDBLOCK)
				return (0);
			pcap_fmt_errmsg_for_errno(p->errbuf, sizeof(p->errbuf),
			    errno, "pcap_read");
			return (-1);
		}
		bp = (u_char *)p->buffer;
	} else
		bp = p->bp;

	/*
	 * loop through each snapshot in the chunk
	 *
	 * This assumes that a single buffer of packets will have
	 * <= INT_MAX packets, so the packet count doesn't overflow.
	 */
	n = 0;
	ep = bp + cc;
	while (bp < ep) {
		/*
		 * Has "pcap_breakloop()" been called?
		 * If so, return immediately - if we haven't read any
		 * packets, clear the flag and return -2 to indicate
		 * that we were told to break out of the loop, otherwise
		 * leave the flag set, so that the *next* call will break
		 * out of the loop without having read any packets, and
		 * return the number of packets we've processed so far.
		 */
		if (p->break_loop) {
			if (n == 0) {
				p->break_loop = 0;
				return (-2);
			} else {
				p->bp = bp;
				p->cc = ep - bp;
				return (n);
			}
		}

		++psn->stat.ps_recv;
		cp = bp;

		/* get past NIT buffer  */
		hdrp = (struct nit_bufhdr *)cp;
		cp += sizeof(*hdrp);

		/* get past NIT timer   */
		ntp = (struct nit_iftime *)cp;
		cp += sizeof(*ntp);

		ndp = (struct nit_ifdrops *)cp;
		psn->stat.ps_drop = ndp->nh_drops;
		cp += sizeof *ndp;

		/* get past packet len  */
		nlp = (struct nit_iflen *)cp;
		cp += sizeof(*nlp);

		/* next snapshot        */
		bp += hdrp->nhb_totlen;

		caplen = nlp->nh_pktlen;
		if (caplen > p->snapshot)
			caplen = p->snapshot;

		if (pcap_filter(p->fcode.bf_insns, cp, nlp->nh_pktlen, caplen)) {
			struct pcap_pkthdr h;
			h.ts = ntp->nh_timestamp;
			h.len = nlp->nh_pktlen;
			h.caplen = caplen;
			(*callback)(user, &h, cp);
			if (++n >= cnt && !PACKET_COUNT_IS_UNLIMITED(cnt)) {
				p->cc = ep - bp;
				p->bp = bp;
				return (n);
			}
		}
	}
	p->cc = 0;
	return (n);
}

```

这段代码是一个用于 Linux 系统上 pcap 文件的注入工具函数，其作用是将给定的数据缓冲区（buf）中的数据发送到 pcap 文件的描述符（fd）中，以便在 pcap 文件中读取数据时使用。

具体来说，该函数接收一个 pcap 句柄（pcap_t *p）和一个数据缓冲区（buf），接收数据缓冲区的大小（size）。函数内部首先定义了一个名为 ctl 的结构体，用于存储数据缓冲区中的数据，然后定义了一个名为 data 的结构体，用于存储要发送到 pcap 文件的数据。

接着，函数实现了一个 write 函数的调用，将数据缓冲区中的数据写入到 pcap 文件的描述符中，使用 putmsg 函数将数据传递给 pcap 函数，并获取返回值。如果返回值为负数，则说明在执行写入操作时遇到了错误，函数将返回一个负值表示。如果返回值等于 0，则说明数据写入成功。

最后，函数将结果返回给调用者，如果返回值等于 -1，则说明 pcap 函数在执行写入操作时遇到了错误，需要进行错误处理。


```cpp
static int
pcap_inject_snit(pcap_t *p, const void *buf, int size)
{
	struct strbuf ctl, data;

	/*
	 * XXX - can we just do
	 *
	ret = write(pd->f, buf, size);
	 */
	ctl.len = sizeof(*sa);	/* XXX - what was this? */
	ctl.buf = (char *)sa;
	data.buf = buf;
	data.len = size;
	ret = putmsg(p->fd, &ctl, &data);
	if (ret == -1) {
		pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
		    errno, "send");
		return (-1);
	}
	return (ret);
}

```

It looks like this is a code function for sending a timestamp (NIOCSTIME) to a network interface. The function takes an interface string (specified by the '--interface' option) and an optional timeout (specified by the '--timeout' option).

The function first checks for any existing timestamp on the interface by using the '--promisc' option with the 'NIOCSCHUNK' error code. If an existing timestamp is found, the function displays an error message and returns -1.

If no existing timestamp is found, the function sets the timestamp to the current time (NIOCSTIME) and sets up a timeout to go after it (INFTIM). The timeout is set to the maximum allowed time (INFTIM) in microseconds.

The function then sets the flags for the timestamp to include the timestamp type, the length of the timestamp, and the drop of the timestamp (specified by the '--drop' option).

Finally, the function sends the timestamp to the network interface by using the 'NIOCSFLAGS' error code.

It's worth noting that the exact behavior of this function may be different depending on the specific network interface and the version of the library used.


```cpp
static int
nit_setflags(pcap_t *p)
{
	bpf_u_int32 flags;
	struct strioctl si;
	u_int zero = 0;
	struct timeval timeout;

	if (p->opt.immediate) {
		/*
		 * Set the chunk size to zero, so that chunks get sent
		 * up immediately.
		 */
		si.ic_cmd = NIOCSCHUNK;
		si.ic_len = sizeof(zero);
		si.ic_dp = (char *)&zero;
		if (ioctl(p->fd, I_STR, (char *)&si) < 0) {
			pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
			    errno, "NIOCSCHUNK");
			return (-1);
		}
	}
	si.ic_timout = INFTIM;
	if (p->opt.timeout != 0) {
		timeout.tv_sec = p->opt.timeout / 1000;
		timeout.tv_usec = (p->opt.timeout * 1000) % 1000000;
		si.ic_cmd = NIOCSTIME;
		si.ic_len = sizeof(timeout);
		si.ic_dp = (char *)&timeout;
		if (ioctl(p->fd, I_STR, (char *)&si) < 0) {
			pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
			    errno, "NIOCSTIME");
			return (-1);
		}
	}
	flags = NI_TIMESTAMP | NI_LEN | NI_DROPS;
	if (p->opt.promisc)
		flags |= NI_PROMISC;
	si.ic_cmd = NIOCSFLAGS;
	si.ic_len = sizeof(flags);
	si.ic_dp = (char *)&flags;
	if (ioctl(p->fd, I_STR, (char *)&si) < 0) {
		pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
		    errno, "NIOCSFLAGS");
		return (-1);
	}
	return (0);
}

```

This is a device driver for an Ethernet capture that implements the Linux kernel's Ethernet轉發功能。在这里，我們通過select()，poll()和dlt_list等函數來實現對STREAMS設備的數據流選擇和數據流輸出。此外，我們還实现了pcap_install_dir，pcap_的国际以及pcap_reservehead恶性肿瘤，pcap_subtest等靜態功能。


```cpp
static int
pcap_activate_snit(pcap_t *p)
{
	struct strioctl si;		/* struct for ioctl() */
	struct ifreq ifr;		/* interface request struct */
	int chunksize = CHUNKSIZE;
	int fd;
	static const char dev[] = "/dev/nit";
	int err;

	if (p->opt.rfmon) {
		/*
		 * No monitor mode on SunOS 4.x (no Wi-Fi devices on
		 * hardware supported by SunOS 4.x).
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

	if (p->snapshot < 96)
		/*
		 * NIT requires a snapshot length of at least 96.
		 */
		p->snapshot = 96;

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
	 */
	p->fd = fd = open(dev, O_RDWR);
	if (fd < 0 && errno == EACCES)
		p->fd = fd = open(dev, O_RDONLY);
	if (fd < 0) {
		if (errno == EACCES) {
			err = PCAP_ERROR_PERM_DENIED;
			snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
			    "Attempt to open %s failed with EACCES - root privileges may be required",
			    dev);
		} else {
			err = PCAP_ERROR;
			pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
			    errno, "%s", dev);
		}
		goto bad;
	}

	/* arrange to get discrete messages from the STREAM and use NIT_BUF */
	if (ioctl(fd, I_SRDOPT, (char *)RMSGD) < 0) {
		pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
		    errno, "I_SRDOPT");
		err = PCAP_ERROR;
		goto bad;
	}
	if (ioctl(fd, I_PUSH, "nbuf") < 0) {
		pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
		    errno, "push nbuf");
		err = PCAP_ERROR;
		goto bad;
	}
	/* set the chunksize */
	si.ic_cmd = NIOCSCHUNK;
	si.ic_timout = INFTIM;
	si.ic_len = sizeof(chunksize);
	si.ic_dp = (char *)&chunksize;
	if (ioctl(fd, I_STR, (char *)&si) < 0) {
		pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
		    errno, "NIOCSCHUNK");
		err = PCAP_ERROR;
		goto bad;
	}

	/* request the interface */
	strncpy(ifr.ifr_name, p->opt.device, sizeof(ifr.ifr_name));
	ifr.ifr_name[sizeof(ifr.ifr_name) - 1] = '\0';
	si.ic_cmd = NIOCBIND;
	si.ic_len = sizeof(ifr);
	si.ic_dp = (char *)&ifr;
	if (ioctl(fd, I_STR, (char *)&si) < 0) {
		/*
		 * XXX - is there an error that means "no such device"?
		 * Is there one that means "that device doesn't support
		 * STREAMS NIT"?
		 */
		pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
		    errno, "NIOCBIND: %s", ifr.ifr_name);
		err = PCAP_ERROR;
		goto bad;
	}

	/* set the snapshot length */
	si.ic_cmd = NIOCSSNAP;
	si.ic_len = sizeof(p->snapshot);
	si.ic_dp = (char *)&p->snapshot;
	if (ioctl(fd, I_STR, (char *)&si) < 0) {
		pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
		    errno, "NIOCSSNAP");
		err = PCAP_ERROR;
		goto bad;
	}
	if (nit_setflags(p) < 0) {
		err = PCAP_ERROR;
		goto bad;
	}

	(void)ioctl(fd, I_FLUSH, (char *)FLUSHR);
	/*
	 * NIT supports only ethernets.
	 */
	p->linktype = DLT_EN10MB;

	p->bufsize = BUFSPACE;
	p->buffer = malloc(p->bufsize);
	if (p->buffer == NULL) {
		pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
		    errno, "malloc");
		err = PCAP_ERROR;
		goto bad;
	}

	/*
	 * "p->fd" is an FD for a STREAMS device, so "select()" and
	 * "poll()" should work on it.
	 */
	p->selectable_fd = p->fd;

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

	p->read_op = pcap_read_snit;
	p->inject_op = pcap_inject_snit;
	p->setfilter_op = install_bpf_program;	/* no kernel filtering */
	p->setdirection_op = NULL;	/* Not implemented. */
	p->set_datalink_op = NULL;	/* can't change data link type */
	p->getnonblock_op = pcap_getnonblock_fd;
	p->setnonblock_op = pcap_setnonblock_fd;
	p->stats_op = pcap_stats_snit;

	return (0);
 bad:
	pcap_cleanup_live_common(p);
	return (err);
}

```

这段代码定义了一个名为pcap_t的指针变量，它是一个pcap_t结构体类型的变量。接下来，定义了一个名为pcap_create_interface的函数，该函数接收两个参数：一个表示网络接口名称的const char *device参数和一个字符型指针ebuf。函数内部创建一个名为p的pcap_t对象，并使用PCAP_CREATE_COMMON函数将其分配给一个名为struct pcap_snit的结构体类型的变量。

pcap_create_interface函数的作用是创建一个pcap_t对象，该对象表示当前网络接口的信息，并将其存储在p变量中。pcap_create_interface函数的第二个参数ebuf是一个字符型指针，用于存储接口名称，函数使用PCAP_SET_NAME函数为接口名称设置值，这个值将作为pcap_t对象中的接口名称字段。


```cpp
pcap_t *
pcap_create_interface(const char *device _U_, char *ebuf)
{
	pcap_t *p;

	p = PCAP_CREATE_COMMON(ebuf, struct pcap_snit);
	if (p == NULL)
		return (NULL);

	p->activate_op = pcap_activate_snit;
	return (p);
}

/*
 * XXX - there's probably a NIOCBIND error that means "that device
 * doesn't support NIT"; if so, we should try an NIOCBIND and use that.
 */
```



这段代码定义了两个静态函数，get_if_flags()和can_be_bound()用于判断USB设备是否可以绑定到指定的USB命名空间中。

1. get_if_flags()函数接受一个USB命名空间名称和一个指向包含32个整数的指针，用于存储该命名空间可用的比特位。函数的实现比较简单，直接返回0，表示无法确定该命名空间是否可用。

2. can_be_bound()函数也接受一个USB命名空间名称和一个指向包含32个整数的指针，用于存储该命名空间是否已经被绑定到该设备上。函数的实现比较复杂，首先检查设备是否支持该命名空间，然后检查设备是否已经将该命名空间绑定到自己的USB总线上。如果设备既支持该命名空间，又已经将其绑定到自己的USB总线上，函数返回1，表示设备可以被绑定到该命名空间中。

由于该代码中包含过多的C字符串，无法通过阅读代码了解函数的实际作用。


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



这两段代码定义了两个函数，第一个函数 `pcap_platform_finddevs` 用于在平台设备中查找网络接口，第二个函数是函数指针，用于将调用第二个函数的返回值作为参数传递给第一个函数。

函数 `pcap_findalldevs_interfaces` 的作用是查找平台设备中可用的网络接口。它接受一个 `pcap_if_list_t` 类型的参数 `devlistp`，包含多个 `pcap_if_t` 类型的接口。函数使用 `can_be_bound` 和 `get_if_flags` 参数获取接口是否已经绑定并且获取接口的 flags，然后返回接口的名称。

函数第二个参数 `errbuf` 是一个字符指针，用于存储错误信息。如果函数成功执行并且没有错误，则该指针为 `NULL`。否则，它将包含一个指向错误信息字符串的指针。

函数 `pcap_lib_version` 返回一个指向 `const char *` 类型的指针，该指针指向 libpcap 的版本字符串。函数使用 `PCAP_VERSION_STRING` 函数获取该字符串，然后将其作为指针返回。


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