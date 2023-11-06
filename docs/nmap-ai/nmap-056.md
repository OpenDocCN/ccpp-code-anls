# Nmap源码解析 56

# `libpcap/pcap-libdlpi.c`

这段代码是一个C语言的函数声明，它定义了一个名为`extern`的函数。这个函数声明中包含了一些说明，让我们来看看它的作用。

首先，这个函数声明中用`extern`关键字定义了一个函数。这个关键字告诉编译器这个函数是外部函数，不是内部函数，也就是这个函数是给外部库或程序使用的。

接下来，我们看到这个函数声明中有一些参数。这些参数是在函数内部使用的，它们或者是外部函数的参数，或者是传递给函数的参数。

然后，我们看到这个函数声明中有一些备注。这些备注告诉编译器如何解释函数的实现。在这里，编译器需要根据这些备注来理解函数的行为。

最后，这个函数声明中还有一些特殊的说明。这些说明告诉编译器如何处理函数的贡献者信息。在这里，编译器需要把这些信息存储在函数的外部文档中，以告知其他人如何使用这个函数。


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
 *
 * This code contributed by Sagun Shakya (sagun.shakya@sun.com)
 */
```

这段代码定义了一些用于DLPI（动态量捕获）的套接字函数，用于在SunOS 5.11操作系统上进行数据包捕获。

首先，它检查了系统是否支持配置文件，如果支持，它就包含了配置文件中的特定头文件。

接下来，代码中定义了一系列套接字函数，包括从系统类型、时间戳和缓冲区大小计算所需的缓冲区长度，以及从DLPI库中分配内存的能力。

总的来说，这段代码定义了一些用于DLPI的函数，以便在SunOS 5.11上进行数据包捕获。


```cpp
/*
 * Packet capture routines for DLPI using libdlpi under SunOS 5.11.
 */

#ifdef HAVE_CONFIG_H
#include <config.h>
#endif

#include <sys/types.h>
#include <sys/time.h>
#include <sys/bufmod.h>
#include <sys/stream.h>
#include <libdlpi.h>
#include <errno.h>
#include <memory.h>
```

这段代码是一个用于在 Linux 系统的 pcap 套件中处理 libdlpi 数据包的开源项目。下面是这段代码的一些关键部分，以及它的作用：

1. 引入相关库文件：pcap-int.h，用于提供关于 libcap 的定义；stdio.h，用于输出信息；stdlib.h，用于提供格式化输入输出的函数；string.h，用于提供字符串操作的函数。
2. 引入 libdlpi 库：在 libdlpi 库中，dlpisupts.h 是主要的文件，该文件中定义了各种 libdlpi 函数，如 dlppromiscon，用于将数据包中的 libdlpi 数据转换为用户可读的名称。
3. libdlpi 函数：dlpromiscon函数用于根据数据包中的 libdlpi 数据创建一个名称，并返回前缀 libdlpi 名称的长度。pcap_read_libdlpi函数用于将 libdlpi 数据包的名称解析为长度为 16 的字符数组，并输出给 pcap 特效。pcap_inject_libdlpi函数用于将 libdlpi 数据包的名称插入到 pcap 特效的，名字前缀为 libdlpi。pcap_libdlpi_err函数用于输出错误信息，并定义了错误代码。pcap_cleanup_libdlpi函数用于输出 libdlpi 数据包清理的函数。
4. 自定义函数：pcap_promiscon，用于实现将 libdlpi 数据转换为用户可读的名称。


```cpp
#include <stropts.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#include "pcap-int.h"
#include "dlpisubs.h"

/* Forwards. */
static int dlpromiscon(pcap_t *, bpf_u_int32);
static int pcap_read_libdlpi(pcap_t *, int, pcap_handler, u_char *);
static int pcap_inject_libdlpi(pcap_t *, const void *, int);
static void pcap_libdlpi_err(const char *, const char *, int, char *);
static void pcap_cleanup_libdlpi(pcap_t *);

```

这段代码定义了一个名为list_interfaces的函数，它返回一个名为lnl的整数类型，表示系统上可用的所有网络接口。

list_interfaces()函数接受两个参数：一个指向constchar *类型的字符串，用于指定接口名称，另一个参数是一个void类型的void指针，用于指定链表节点。函数返回lnl表示成功返回的接口数量。

函数内部首先定义了一个名为linknamelist的链表节点结构体，它包含一个名为linkname的char类型的链表节点，以及一个lnl_next的指向linknamelist的指针。

接着定义了一个名为linkwalk的链表节点结构体，它包含一个名为lw_list的linknamelist类型的链表节点，以及一个lw_err的整数类型的变量，用于记录链表节点错误信息。

最后在函数内部使用这些定义的struct结构体变量，对系统上的所有网络接口进行遍历，将可用的接口数量保存到lnl中，并返回lnl作为函数结果。


```cpp
/*
 * list_interfaces() will list all the network links that are
 * available on a system.
 */
static boolean_t list_interfaces(const char *, void *);

typedef struct linknamelist {
	char	linkname[DLPI_LINKNAME_MAX];
	struct linknamelist *lnl_next;
} linknamelist_t;

typedef struct linkwalk {
	linknamelist_t	*lw_list;
	int		lw_err;
} linkwalk_t;

```

这段代码定义了一个名为`list_interfaces`的函数，用于链式遍历。该函数接受一个`const char *`类型的参数`linkname`，以及一个`void *`类型的参数`arg`。

函数的作用是：
1. 如果`linkname`为空，则返回错误信息。
2. 如果`calloc`函数返回的内存不足，则返回错误信息。
3. 创建一个`linknamelist_t`类型的变量`entry`，并将其指向`calloc`函数返回的内存位置。
4. 使用`pcap_strlcpy`函数将`linkname`参数与`linkname`常量进行复制。
5. 如果`lwp`指向的内存位置为`NULL`，则将其与`entry`的`lnl_next`字段进行连接，并将`entry`的`lnl_next`字段设置为`lwp`。
6. 如果`lwp`指向的内存位置不为`NULL`，则将其与`entry`的`lnl_next`字段进行连接，并将`entry`的`lnl_next`字段设置为`lwp`。
7. 返回`B_FALSE`，表示函数成功执行。


```cpp
/*
 * The caller of this function should free the memory allocated
 * for each linknamelist_t "entry" allocated.
 */
static boolean_t
list_interfaces(const char *linkname, void *arg)
{
	linkwalk_t	*lwp = arg;
	linknamelist_t	*entry;

	if ((entry = calloc(1, sizeof(linknamelist_t))) == NULL) {
		lwp->lw_err = ENOMEM;
		return (B_TRUE);
	}
	(void) pcap_strlcpy(entry->linkname, linkname, DLPI_LINKNAME_MAX);

	if (lwp->lw_list == NULL) {
		lwp->lw_list = entry;
	} else {
		entry->lnl_next = lwp->lw_list;
		lwp->lw_list = entry;
	}

	return (B_FALSE);
}

```

This function appears to check the status of a Linux data driver and, if there are any errors or the driver is not found, it returns an error code.

The function first checks if the device type of the data driver is not a stream device and, if it is not, the function returns an error code.

The function then configures the data buffer for the device, and if there are any errors in the configuration process, the function returns an error code.

The function then sets the read operation of the data driver to use the stream device's native read function, and if there are any errors in this operation, the function returns an error code.

The function also sets the install filter operation to the default implementation, and if there are any errors in this operation, the function returns an error code.

The function then sets the data link operation to the default implementation, and if there are any errors in this operation, the function returns an error code.

The function sets the statistics operation to the stream device's native statistics implementation, and if there are any errors in this operation, the function returns an error code.

The function then calls the cleanup operation of the data driver, and if there are any errors in this operation, the function returns an error code.

If the data driver is not found or there are any errors in the configuration process, the function returns an error code.


```cpp
static int
pcap_activate_libdlpi(pcap_t *p)
{
	struct pcap_dlpi *pd = p->priv;
	int status = 0;
	int retv;
	dlpi_handle_t dh;
	dlpi_info_t dlinfo;

	/*
	 * Enable Solaris raw and passive DLPI extensions;
	 * dlpi_open() will not fail if the underlying link does not support
	 * passive mode. See dlpi(7P) for details.
	 */
	retv = dlpi_open(p->opt.device, &dh, DLPI_RAW|DLPI_PASSIVE);
	if (retv != DLPI_SUCCESS) {
		if (retv == DLPI_ELINKNAMEINVAL || retv == DLPI_ENOLINK) {
			/*
			 * There's nothing more to say, so clear the
			 * error message.
			 */
			status = PCAP_ERROR_NO_SUCH_DEVICE;
			p->errbuf[0] = '\0';
		} else if (retv == DL_SYSERR &&
		    (errno == EPERM || errno == EACCES)) {
			status = PCAP_ERROR_PERM_DENIED;
			snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
			    "Attempt to open DLPI device failed with %s - root privilege may be required",
			    (errno == EPERM) ? "EPERM" : "EACCES");
		} else {
			status = PCAP_ERROR;
			pcap_libdlpi_err(p->opt.device, "dlpi_open", retv,
			    p->errbuf);
		}
		return (status);
	}
	pd->dlpi_hd = dh;

	if (p->opt.rfmon) {
		/*
		 * This device exists, but we don't support monitor mode
		 * any platforms that support DLPI.
		 */
		status = PCAP_ERROR_RFMON_NOTSUP;
		goto bad;
	}

	/* Bind with DLPI_ANY_SAP. */
	if ((retv = dlpi_bind(pd->dlpi_hd, DLPI_ANY_SAP, 0)) != DLPI_SUCCESS) {
		status = PCAP_ERROR;
		pcap_libdlpi_err(p->opt.device, "dlpi_bind", retv, p->errbuf);
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

	/* Enable promiscuous mode. */
	if (p->opt.promisc) {
		retv = dlpromiscon(p, DL_PROMISC_PHYS);
		if (retv < 0) {
			/*
			 * "You don't have permission to capture on
			 * this device" and "you don't have permission
			 * to capture in promiscuous mode on this
			 * device" are different; let the user know,
			 * so if they can't get permission to
			 * capture in promiscuous mode, they can at
			 * least try to capture in non-promiscuous
			 * mode.
			 *
			 * XXX - you might have to capture in
			 * promiscuous mode to see outgoing packets.
			 */
			if (retv == PCAP_ERROR_PERM_DENIED)
				status = PCAP_ERROR_PROMISC_PERM_DENIED;
			else
				status = retv;
			goto bad;
		}
	} else {
		/* Try to enable multicast. */
		retv = dlpromiscon(p, DL_PROMISC_MULTI);
		if (retv < 0) {
			status = retv;
			goto bad;
		}
	}

	/* Try to enable SAP promiscuity. */
	retv = dlpromiscon(p, DL_PROMISC_SAP);
	if (retv < 0) {
		/*
		 * Not fatal, since the DL_PROMISC_PHYS mode worked.
		 * Report it as a warning, however.
		 */
		if (p->opt.promisc)
			status = PCAP_WARNING;
		else {
			status = retv;
			goto bad;
		}
	}

	/* Determine link type.  */
	if ((retv = dlpi_info(pd->dlpi_hd, &dlinfo, 0)) != DLPI_SUCCESS) {
		status = PCAP_ERROR;
		pcap_libdlpi_err(p->opt.device, "dlpi_info", retv, p->errbuf);
		goto bad;
	}

	if (pcap_process_mactype(p, dlinfo.di_mactype) != 0) {
		status = PCAP_ERROR;
		goto bad;
	}

	p->fd = dlpi_fd(pd->dlpi_hd);

	/* Push and configure bufmod. */
	if (pcap_conf_bufmod(p, p->snapshot) != 0) {
		status = PCAP_ERROR;
		goto bad;
	}

	/*
	 * Flush the read side.
	 */
	if (ioctl(p->fd, I_FLUSH, FLUSHR) != 0) {
		status = PCAP_ERROR;
		pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
		    errno, "FLUSHR");
		goto bad;
	}

	/* Allocate data buffer. */
	if (pcap_alloc_databuf(p) != 0) {
		status = PCAP_ERROR;
		goto bad;
	}

	/*
	 * "p->fd" is a FD for a STREAMS device, so "select()" and
	 * "poll()" should work on it.
	 */
	p->selectable_fd = p->fd;

	p->read_op = pcap_read_libdlpi;
	p->inject_op = pcap_inject_libdlpi;
	p->setfilter_op = install_bpf_program;	/* No kernel filtering */
	p->setdirection_op = NULL;	/* Not implemented */
	p->set_datalink_op = NULL;	/* Can't change data link type */
	p->getnonblock_op = pcap_getnonblock_fd;
	p->setnonblock_op = pcap_setnonblock_fd;
	p->stats_op = pcap_stats_dlpi;
	p->cleanup_op = pcap_cleanup_libdlpi;

	return (status);
```

This is a function definition for `dlpi_promiscon` which takes a `pcap_t` pointer and an `bpf_u_int32` argument for


```cpp
bad:
	pcap_cleanup_libdlpi(p);
	return (status);
}

#define STRINGIFY(n)	#n

static int
dlpromiscon(pcap_t *p, bpf_u_int32 level)
{
	struct pcap_dlpi *pd = p->priv;
	int retv;
	int err;

	retv = dlpi_promiscon(pd->dlpi_hd, level);
	if (retv != DLPI_SUCCESS) {
		if (retv == DL_SYSERR &&
		    (errno == EPERM || errno == EACCES)) {
			if (level == DL_PROMISC_PHYS) {
				err = PCAP_ERROR_PROMISC_PERM_DENIED;
				snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
				    "Attempt to set promiscuous mode failed with %s - root privilege may be required",
				    (errno == EPERM) ? "EPERM" : "EACCES");
			} else {
				err = PCAP_ERROR_PERM_DENIED;
				snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
				    "Attempt to set %s mode failed with %s - root privilege may be required",
				    (level == DL_PROMISC_MULTI) ? "multicast" : "SAP promiscuous",
				    (errno == EPERM) ? "EPERM" : "EACCES");
			}
		} else {
			err = PCAP_ERROR;
			pcap_libdlpi_err(p->opt.device,
			    "dlpi_promiscon" STRINGIFY(level),
			    retv, p->errbuf);
		}
		return (err);
	}
	return (0);
}

```

这段代码定义了两个函数：is_dlpi_interface() 和 get_if_flags()。它们的主要作用是检查给定的字符串是否指代一个DLPI设备。

is_dlpi_interface()函数接收一个字符串参数，首先通过调用dlpi_walk()函数检查该字符串是否代表一个DLPI设备。如果不代表DLPI设备，函数返回0。如果代表DLPI设备，函数将返回1。

get_if_flags()函数与is_dlpi_interface()函数类似，但使用了一位菊花字符串（该字符串以"."为分隔符，而不是字符串中的"*"）和三个整数参数： flags、errbuf 和 0。该函数的作用是获取设备接口的标志，并将其设置为所需的标志。如果设备接口是套接字，则该函数将尝试从dladm命令获得连接/断开状态，并将结果设置为所需的标志。如果设备接口不是套接字，则函数将返回0。


```cpp
/*
 * Presumably everything returned by dlpi_walk() is a DLPI device,
 * so there's no work to be done here to check whether name refers
 * to a DLPI device.
 */
static int
is_dlpi_interface(const char *name _U_)
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
	 * XXX - on Solaris, can we do what the dladm command does,
	 * i.e. get a connected/disconnected indication from a kstat?
	 * (Note that you can also get the link speed, and possibly
	 * other information, from a kstat as well.)
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

The 'pcap_platform_finddevs()' function is intended to find additional network links present in the system. It does so by first getting a list of all regular interface devices using 'pcap_findalldevs_interfaces()' and then using a loopback 'dlpi_walk()' to find all DLPI devices in the current zone.

The function returns an error number on failure, or 0 on success. If an error occurs during the walk operation, the function will return -1 and告之错误信息。


```cpp
/*
 * In Solaris, the "standard" mechanism" i.e SIOCGLIFCONF will only find
 * network links that are plumbed and are up. dlpi_walk(3DLPI) will find
 * additional network links present in the system.
 */
int
pcap_platform_finddevs(pcap_if_list_t *devlistp, char *errbuf)
{
	int retv = 0;

	linknamelist_t	*entry, *next;
	linkwalk_t	lw = {NULL, 0};
	int		save_errno;

	/*
	 * Get the list of regular interfaces first.
	 */
	if (pcap_findalldevs_interfaces(devlistp, errbuf,
	    is_dlpi_interface, get_if_flags) == -1)
		return (-1);	/* failure */

	/* dlpi_walk() for loopback will be added here. */

	/*
	 * Find all DLPI devices in the current zone.
	 *
	 * XXX - will pcap_findalldevs_interfaces() find any devices
	 * outside the current zone?  If not, the only reason to call
	 * it would be to get the interface addresses.
	 */
	dlpi_walk(list_interfaces, &lw, 0);

	if (lw.lw_err != 0) {
		pcap_fmt_errmsg_for_errno(errbuf, PCAP_ERRBUF_SIZE,
		    lw.lw_err, "dlpi_walk");
		retv = -1;
		goto done;
	}

	/* Add linkname if it does not exist on the list. */
	for (entry = lw.lw_list; entry != NULL; entry = entry->lnl_next) {
		/*
		 * If it isn't already in the list of devices, try to
		 * add it.
		 */
		if (find_or_add_dev(devlistp, entry->linkname, 0, get_if_flags,
		    NULL, errbuf) == NULL)
			retv = -1;
	}
```

这段代码的作用是读取DLPI（数据平面接口） handle 中接收到的数据。它首先获取当前遍历的 entry 对象的下一个对象（next），然后免费释放 entry 对象。这样，当遍历结束时，会免费释放整个数组。接着，将保存的错误码（save_errno）和当前遍历的 entry 对象的下一个对象（next）保存到 done 变量中，最后返回 done。


```cpp
done:
	save_errno = errno;
	for (entry = lw.lw_list; entry != NULL; entry = next) {
		next = entry->lnl_next;
		free(entry);
	}
	errno = save_errno;

	return (retv);
}

/*
 * Read data received on DLPI handle. Returns -2 if told to terminate, else
 * returns the number of packets read.
 */
```

This function is a part of the Linux kernel's packet Capture library, specifically the Winock representative. It is used to receive data from the network in the packet form and pass it to the handler specified by the `callback` parameter.

The function takes in a pointer to a `pcap_t` structure, a counter `count` indicating the number of packets to receive, a pointer to a function `callback` to handle the received packets, and a pointer to a user-provided buffer `user` of any size.

The function first initializes the pointer to the current `pcap_t` structure with `p->priv` to obtain the private members of the structure.

Then it sets the `break_loop` flag to `0` to clear the flag that indicates the loop and returns immediately if it has been told to break out of the loop.

The function reads data from the network in blocks and processes each packet according to the specified `callback`. It waits until there is no more data to read and returns an error code if an error occurs.


```cpp
static int
pcap_read_libdlpi(pcap_t *p, int count, pcap_handler callback, u_char *user)
{
	struct pcap_dlpi *pd = p->priv;
	int len;
	u_char *bufp;
	size_t msglen;
	int retv;

	len = p->cc;
	if (len != 0) {
		bufp = p->bp;
		goto process_pkts;
	}
	do {
		/* Has "pcap_breakloop()" been called? */
		if (p->break_loop) {
			/*
			 * Yes - clear the flag that indicates that it has,
			 * and return -2 to indicate that we were told to
			 * break out of the loop.
			 */
			p->break_loop = 0;
			return (-2);
		}

		msglen = p->bufsize;
		bufp = (u_char *)p->buffer + p->offset;

		retv = dlpi_recv(pd->dlpi_hd, NULL, NULL, bufp,
		    &msglen, -1, NULL);
		if (retv != DLPI_SUCCESS) {
			/*
			 * This is most likely a call to terminate out of the
			 * loop. So, do not return an error message, instead
			 * check if "pcap_breakloop()" has been called above.
			 */
			if (retv == DL_SYSERR && errno == EINTR) {
				len = 0;
				continue;
			}
			pcap_libdlpi_err(dlpi_linkname(pd->dlpi_hd),
			    "dlpi_recv", retv, p->errbuf);
			return (-1);
		}
		len = msglen;
	} while (len == 0);

```



这两段代码是用于在 Wireshark 中的 pcap 协议中处理 pkt 包的。pcap 是一种网络协议的数据包抓取器，允许用户捕获网络流量。pcap_pkts() 函数用于返回捕获到的数据包的数量，而 process_pkts() 函数则处理这些数据包。

具体来说，process_pkts() 函数接受一个指向 pcap 对象和一个回调函数，以及发送者和接收者用户名和计数器。它通过调用 pcap_process_pkts() 函数来处理数据包，并将处理结果返回到回调函数中。pcap_process_pkts() 函数接受一个指向 pcap 对象和接收数据包的指针、计数器和数据缓冲区。它通过在接收数据包的基础上发送数据，使得 pcap 能够捕获数据流并识别包中的特定信息。

static int
pcap_inject_libdlpi(pcap_t *p, const void *buf, int size)
{
	struct pcap_dlpi *pd = p->priv;
	int retv;

	retv = dlpi_send(pd->dlpi_hd, NULL, 0, buf, size, NULL);
	if (retv != DLPI_SUCCESS) {
		pcap_libdlpi_err(dlpi_linkname(pd->dlpi_hd), "dlpi_send", retv,
		    p->errbuf);
		return (-1);
	}
	/*
	 * dlpi_send(3DLPI) does not provide a way to return the number of
	 * bytes sent on the wire. Based on the fact that DLPI_SUCCESS was
	 * returned we are assuming 'size' bytes were sent.
	 */
	return (size);
}

在这两段代码中，我们定义了一个名为 pcap_inject_libdlpi() 的函数，用于将数据包通过 libdlpi 发送到数据源。该函数需要一个数据缓冲区和一个计数器作为参数。首先，我们将数据缓冲区复制到一个名为 pd 的结构中，然后使用 dlpi_send() 函数将其发送到 libdlpi 实例的第二个发送者。如果发送成功，函数将返回发送的数量，否则会通过 pcap_libdlpi_err() 函数返回一个错误信息。


```cpp
process_pkts:
	return (pcap_process_pkts(p, callback, user, count, bufp, len));
}

static int
pcap_inject_libdlpi(pcap_t *p, const void *buf, int size)
{
	struct pcap_dlpi *pd = p->priv;
	int retv;

	retv = dlpi_send(pd->dlpi_hd, NULL, 0, buf, size, NULL);
	if (retv != DLPI_SUCCESS) {
		pcap_libdlpi_err(dlpi_linkname(pd->dlpi_hd), "dlpi_send", retv,
		    p->errbuf);
		return (-1);
	}
	/*
	 * dlpi_send(3DLPI) does not provide a way to return the number of
	 * bytes sent on the wire. Based on the fact that DLPI_SUCCESS was
	 * returned we are assuming 'size' bytes were sent.
	 */
	return (size);
}

```



这段代码定义了一个名为 `pcap_cleanup_libdlpi` 的函数，属于 `pcap_t` 类的成员函数。

该函数的主要作用是关闭 DLPI(数据链路层协议栈) handle，并且清理与 DLPI 相关的资源。具体实现包括以下几个步骤：

1. 检查是否已经打开了 DLPI handle。如果是，则调用 `dlpi_close` 函数关闭 handle，并将 handle 设置为 `NULL`，同时将文件描述符设置为 `-1`，表示已经关闭了文件。

2. 调用 `pcap_cleanup_live_common` 函数，该函数是用于清理现场分会的函数，但在这个函数中也被用作 `pcap_t` 的成员函数。

3. 调用 `dlpi_init` 函数，该函数是用于初始化 DLPI 的函数，但是该函数没有具体的作用，因此可以被视为一个空操作，不会产生任何影响。

4. 该函数的返回点是 `0`。


```cpp
/*
 * Close dlpi handle.
 */
static void
pcap_cleanup_libdlpi(pcap_t *p)
{
	struct pcap_dlpi *pd = p->priv;

	if (pd->dlpi_hd != NULL) {
		dlpi_close(pd->dlpi_hd);
		pd->dlpi_hd = NULL;
		p->fd = -1;
	}
	pcap_cleanup_live_common(p);
}

```

这段代码定义了两个函数，用于在 libpcap 库中处理错误信息。

第一个函数 `pcap_libdlpi_err` 接收三个参数：`const char *linkname`，`const char *func` 和一个字符数组 `errbuf`，它通过 `dlpi_strerror` 函数将 `err` 错误消息转换为一个字符串，然后通过 `snprintf` 函数将该字符串填充到 `errbuf` 数组中。最后，该函数返回一个指向 `pcap_t` 结构体的指针，用于表示成功创建并返回一个 `pcap_t` 类型的数据。

第二个函数 `pcap_create_interface` 接收两个参数：`const char *device` 和一个字符数组 `ebuf`，它用于创建一个 `pcap_t` 类型的数据，该数据将连接到指定的网络接口。如果该接口不存在，该函数将返回一个 `NULL` 类型的指针。

函数实现中，首先通过 `PCAP_CREATE_COMMON` 函数创建一个字符数组，并将其指针存储到一个 `pcap_t` 类型的变量中。然后通过调用 `pcap_activate_libdlpi` 函数将该接口启用，以便允许创建数据包。最后，将创建好的 `pcap_t` 类型的数据返回给调用者。


```cpp
/*
 * Write error message to buffer.
 */
static void
pcap_libdlpi_err(const char *linkname, const char *func, int err, char *errbuf)
{
	snprintf(errbuf, PCAP_ERRBUF_SIZE, "libpcap: %s failed on %s: %s",
	    func, linkname, dlpi_strerror(err));
}

pcap_t *
pcap_create_interface(const char *device _U_, char *ebuf)
{
	pcap_t *p;

	p = PCAP_CREATE_COMMON(ebuf, struct pcap_dlpi);
	if (p == NULL)
		return (NULL);

	p->activate_op = pcap_activate_libdlpi;
	return (p);
}

```

该代码定义了一个名为 `pcap_lib_version` 的函数，它返回的是一个指向字符串常量 `PCAP_VERSION_STRING` 的指针。

通过调用 `pcap_lib_version()` 函数，可以获取到 Libpcap 的版本信息，从而在需要时进行版本查询或与其他开发人员进行沟通。


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

# `libpcap/pcap-linux.c`

This is a simple Java program that generates a RandomizedHashMap by洗牌 (srand) a臭鼬鼠 (c恭喜)hash table.

This program is using the Azure.M自强力。他们为计算机提供各种用途。这个程序是使用Java编程语言。

该程序的主要想法是通过不断洗牌c恭喜算法，来创建一个随机化的哈希表。

这一想法是通过哈希表来实现的。

请注意，哈希表是一种数据结构，它不允许数据的存储顺序。

随机化哈希表的原理是，哈希表的每个关键字的唯一ID是通过对键进行哈希计算得出的。

在这个特定程序中，使用c恭喜算法作为哈希表的洗牌函数。

总体而言，这个程序是一个很好的例子，展示了Java编程语言中哈希表的实现。


```cpp
/*
 *  pcap-linux.c: Packet capture interface to the Linux kernel
 *
 *  Copyright (c) 2000 Torsten Landschoff <torsten@debian.org>
 *		       Sebastian Krahmer  <krahmer@cs.uni-potsdam.de>
 *
 *  License: BSD
 *
 *  Redistribution and use in source and binary forms, with or without
 *  modification, are permitted provided that the following conditions
 *  are met:
 *
 *  1. Redistributions of source code must retain the above copyright
 *     notice, this list of conditions and the following disclaimer.
 *  2. Redistributions in binary form must reproduce the above copyright
 *     notice, this list of conditions and the following disclaimer in
 *     the documentation and/or other materials provided with the
 *     distribution.
 *  3. The names of the authors may not be used to endorse or promote
 *     products derived from this software without specific prior
 *     written permission.
 *
 *  THIS SOFTWARE IS PROVIDED ``AS IS'' AND WITHOUT ANY EXPRESS OR
 *  IMPLIED WARRANTIES, INCLUDING, WITHOUT LIMITATION, THE IMPLIED
 *  WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE.
 *
 *  Modifications:     Added PACKET_MMAP support
 *                     Paolo Abeni <paolo.abeni@email.it>
 *                     Added TPACKET_V3 support
 *                     Gabor Tatarka <gabor.tatarka@ericsson.com>
 *
 *                     based on previous works of:
 *                     Simon Patarin <patarin@cs.unibo.it>
 *                     Phil Wood <cpw@lanl.gov>
 *
 * Monitor-mode support for mac80211 includes code taken from the iw
 * command; the copyright notice for that code is
 *
 * Copyright (c) 2007, 2008	Johannes Berg
 * Copyright (c) 2007		Andy Lutomirski
 * Copyright (c) 2007		Mike Kershaw
 * Copyright (c) 2008		Gábor Stefanik
 *
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
 * THIS SOFTWARE IS PROVIDED BY THE AUTHOR ``AS IS'' AND ANY EXPRESS OR
 * IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE IMPLIED WARRANTIES
 * OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE ARE DISCLAIMED.
 * IN NO EVENT SHALL THE AUTHOR BE LIABLE FOR ANY DIRECT, INDIRECT,
 * INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING,
 * BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES;
 * LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED
 * AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY,
 * OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY
 * OUT OF THE USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF
 * SUCH DAMAGE.
 */


```

这段代码是一个C语言编译器的预处理指令，它定义了一些符号常量。

具体来说，这段代码定义了以下符号常量：

1. `_GNU_SOURCE`：表示该代码位于GNU源代码的源文件中。
2. `HAVE_CONFIG_H`：表示该文件在操作系统支持的环境中是否包含`config.h`头文件，如果包含则可以使用该文件。
3. `errno.h`：表示包含错误数字的`errno.h`头文件。
4. `stdio.h`：包含输入/输出操作的头文件。
5. `stdlib.h`：包含通用的库函数的头文件。
6. `unistd.h`：包含标准的输入/输出操作的头文件。
7. `fcntl.h`：包含文件输入/输出的头文件。
8. `string.h`：包含字符串操作的头文件。
9. `limits.h`：包含一些用于处理数字限制的头文件。
10. `sys/stat.h`：包含文件系统统计信息的头文件。
11. `sys/socket.h`：包含套接字编程的头文件。

此外，还有一条注释：
```cpp
// 定义了一些通用的宏
#define _GNU_SOURCE
#ifdef HAVE_CONFIG_H
#include <config.h>
#endif
```
这条注释解释了`_GNU_SOURCE`的含义，同时也提醒了开发者在使用任何C或C++标准库函数时，需要根据操作系统是否支持这些函数来进行编译。


```cpp
#define _GNU_SOURCE

#ifdef HAVE_CONFIG_H
#include <config.h>
#endif

#include <errno.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <fcntl.h>
#include <string.h>
#include <limits.h>
#include <sys/stat.h>
#include <sys/socket.h>
```



该代码是一个网络协议栈的示例，用于在Linux系统上实现网络协议的栈。以下是代码的作用说明：

1. `#include <sys/ioctl.h>`：用于引入系统调用IOCTL的函数声明。该函数用于在Linux系统中进行与设备相关的I/O操作。

2. `#include <sys/utsname.h>`：用于引入系统调用UTSNAME的函数声明。该函数用于获取系统串并将其转换为相应的设备名称。

3. `#include <sys/mman.h>`：用于引入系统调用MMAP的函数声明。该函数用于管理内存并支持读写随机访问。

4. `#include <linux/if.h>`：用于引入Linux系统调用IF的函数声明。该函数用于管理网络接口，包括配置、监听和丢弃数据包等。

5. `#include <linux/if_packet.h>`：用于引入Linux系统调用IF_PACKET的函数声明。该函数用于处理网络数据包，包括解包、构建数据包和发送数据包等。

6. `#include <linux/ethtool.h>`：用于引入Linux系统调用ethtool的函数声明。该函数用于管理网络接口的统计信息，包括发送和接收数据包的数量、延迟和吞吐量等。

7. `#include <netinet/in.h>`：用于引入Linux系统调用INET的函数声明。该函数用于管理网络接口的IP地址和其他网络接口信息。

8. `#include <linux/if_arp.h>`：用于引入Linux系统调用IF_ARP的函数声明。该函数用于管理网络接口的ARP高速缓存，包括添加、查询和删除ARP高速缓存项等。

9. `#include <poll.h>`：用于引入Linux系统调用poll的函数声明。该函数用于管理输入/输出套接字，包括检测事件的发生和通知相关函数等。

10. `#include <dirent.h>`：用于引入Linux系统调用dirent的函数声明。该函数用于获取系统当前的工作目录。

11. `#include <sys/eventfd.h>`：用于引入Linux系统调用eventfd的函数声明。该函数用于管理文件描述符，包括创建、读写和关闭文件描述符等。

该代码是一组用于在Linux系统上实现网络协议的函数和头文件。通过这些函数和头文件，可以实现对网络接口和网络套接字的配置和管理，以及处理网络数据包和ARP高速缓存等信息。


```cpp
#include <sys/ioctl.h>
#include <sys/utsname.h>
#include <sys/mman.h>
#include <linux/if.h>
#include <linux/if_packet.h>
#include <linux/sockios.h>
#include <linux/ethtool.h>
#include <netinet/in.h>
#include <linux/if_ether.h>
#include <linux/if_arp.h>
#include <poll.h>
#include <dirent.h>
#include <sys/eventfd.h>

#include "pcap-int.h"
```

这段代码是一个网络数据包抓取程序，用于从网络流量中捕获符合TPACKET_V2规范的数据包。它需要TPACKET_V2支持，如果您的内核版本较低，则需要使用2.6.27或更高版本。

以下是此代码的作用：

1. 包含pcap库中的几个头文件，用于定义TPACKET_V2规范的结构体和函数。
2. 引入diag-control库，用于提供诊断控制功能。
3. 定义了一个名为TPACKET_HDRLEN的宏，用于检查是否支持TPACKET_V2规范。
4. 引入can_socketcan库，用于与网络上的CAN总线通信。
5. 使用TPACKET_V2规范中定义的结构体，从网络流量中捕获数据包。
6. 支持选项TPACKET_FILE，允许用户将整个网络流量文件下载到内存中，并在程序运行时按文件名索引。
7. 如果TPACKET_V2规范不支持，则输出错误信息并退出程序。
8. 在程序结束时，将初始化诊断控制，以便在需要时提供帮助。


```cpp
#include "pcap/sll.h"
#include "pcap/vlan.h"
#include "pcap/can_socketcan.h"

#include "diag-control.h"

/*
 * We require TPACKET_V2 support.
 */
#ifndef TPACKET2_HDRLEN
#error "Libpcap will only work if TPACKET_V2 is supported; you must build for a 2.6.27 or later kernel"
#endif

/* check for memory mapped access avaibility. We assume every needed
 * struct is defined if the macro TPACKET_HDRLEN is defined, because it
 * uses many ring related structs and macros */
```

这段代码定义了一系列定义，其中部分定义了``__attribute__((c納)))`标记，表示这是一个元数据定义，会按照元数据定义的规则输出其作用域下的代码。

具体来说，这段代码定义了一个名为``TPACKET3_HDRLEN``的标识，如果这个标识存在，就定义了一系列和原子相关的定义，包括``__atomic_load_n``和``__atomic_store_n``函数，它们的作用是在程序运行时对缓冲区中的数据进行原子操作。

如果`TPACKET3_HDRLEN`标识不存在，那么不会输出任何与原子相关的定义。

另外，由于这段代码使用了`#ifdef`和`#ifndef`的组合，因此它的作用域只在定义它的源文件所在的目录下被考虑。


```cpp
#ifdef TPACKET3_HDRLEN
# define HAVE_TPACKET3
#endif /* TPACKET3_HDRLEN */

/*
 * Not all compilers that are used to compile code to run on Linux have
 * these builtins.  For example, older versions of GCC don't, and at
 * least some people are doing cross-builds for MIPS with older versions
 * of GCC.
 */
#ifndef HAVE___ATOMIC_LOAD_N
#define __atomic_load_n(ptr, memory_model)		(*(ptr))
#endif
#ifndef HAVE___ATOMIC_STORE_N
#define __atomic_store_n(ptr, val, memory_model)	*(ptr) = (val)
```

这段代码定义了几个与网络数据包相关的宏，以及一个名为`packet_mmap`的函数。函数的实现如下：

```cppc
#define packet_mmap_acquire(pkt)                      \
   (__atomic_load_n(&pkt->tp_status, __ATOMIC_ACQUIRE) != TP_STATUS_KERNEL)      \
   (__atomic_load_n(&pkt->tp_status, __ATOMIC_ACQUIRE) == TP_STATUS_KERNEL)
#define packet_mmap_release(pkt)                      \
   (__atomic_store_n(&pkt->tp_status, __ATOMIC_RELEASE))      \
   (__atomic_store_n(&pkt->tp_status, __ATOMIC_ACQUIRE, __ATOMIC_RELEASE))
#define packet_mmap_v3_acquire(pkt)                  \
   (__atomic_load_n(&pkt->hdr.bh1.block_status, __ATOMIC_ACQUIRE) != TP_STATUS_KERNEL)      \
   (__atomic_load_n(&pkt->hdr.bh1.block_status, __ATOMIC_ACQUIRE) == TP_STATUS_KERNEL)
#define packet_mmap_v3_release(pkt)                  \
   (__atomic_store_n(&pkt->hdr.bh1.block_status, __ATOMIC_RELEASE))      \
   (__atomic_store_n(&pkt->hdr.bh1.block_status, __ATOMIC_ACQUIRE, __ATOMIC_RELEASE))
```

首先，我们需要了解`TP_STATUS_KERNEL`和`TP_STATUS_USER`的区别。在Linux内核中，有用的数据包类型使用`TP_STATUS_KERNEL`，而用户数据包类型则使用`TP_STATUS_USER`。因此，我们可以将`TP_STATUS_KERNEL`作为条件，以获得对数据包类型的访问权限。

`packet_mmap`函数的作用是，在`TP_STATUS_KERNEL`条件下，获取一个数据包的结构指针（也就是`pkt`）。然后，我们定义了两个与释放相关的宏，分别对应`packet_mmap_acquire`和`packet_mmap_release`函数。

另外，我们还包括了`packet_mmap_v3_acquire`和`packet_mmap_v3_release`函数。这两个函数分别用于在`TP_STATUS_USER`条件下获取和释放数据包的块状状态。

最后，我们需要包含`<linux/types.h>`和`<linux/filter.h>`头文件。


```cpp
#endif

#define packet_mmap_acquire(pkt) \
	(__atomic_load_n(&pkt->tp_status, __ATOMIC_ACQUIRE) != TP_STATUS_KERNEL)
#define packet_mmap_release(pkt) \
	(__atomic_store_n(&pkt->tp_status, TP_STATUS_KERNEL, __ATOMIC_RELEASE))
#define packet_mmap_v3_acquire(pkt) \
	(__atomic_load_n(&pkt->hdr.bh1.block_status, __ATOMIC_ACQUIRE) != TP_STATUS_KERNEL)
#define packet_mmap_v3_release(pkt) \
	(__atomic_store_n(&pkt->hdr.bh1.block_status, TP_STATUS_KERNEL, __ATOMIC_RELEASE))

#include <linux/types.h>
#include <linux/filter.h>

#ifdef HAVE_LINUX_NET_TSTAMP_H
```

这段代码的作用是检查设备是否为 bonding设备，并检查设备是否支持无线网络（NL80211）规范。

首先，代码从头文件中包含了两个头文件：`linux/net_tstamp.h` 和 `linux/if_bonding.h`。这两个头文件分别定义了`net_tstamp`类型和`if_bonding`函数，与网络时间戳和 bonds 相关。

接下来，代码中通过`#ifdef`和`#define`语句检查设备是否支持 NL80211 规范。如果设备支持 NL80211，那么代码将包含 `<linux/nl80211.h>` 和 `<netlink/genl/genl.h>` 头文件，这些头文件包含了 NL80211 规范的相关接口和函数。

最后，代码中使用 `netlink_parse_print(struct genl_info *genl_info, struct netlink_device *netlink);` 函数来创建一个 `netlink_device` 结构体，并使用 `netlink_write(struct genl_info *genl_info, struct netlink_device *netlink, netlink_open_cmds);` 函数将 NL80211 规范的配置信息写入到 `netlink_device` 结构体中，并使用 `netlink_start(struct genl_info *genl_info, struct netlink_device *netlink);` 函数启动 NL80211 设备。

整个代码的作用是检查设备是否支持 NL80211 规范，并配置 NL80211 设备以创建一个 bonding 设备。


```cpp
#include <linux/net_tstamp.h>
#endif

/*
 * For checking whether a device is a bonding device.
 */
#include <linux/if_bonding.h>

/*
 * Got libnl?
 */
#ifdef HAVE_LIBNL
#include <linux/nl80211.h>

#include <netlink/genl/genl.h>
```

这段代码是一个网络链路库（netlink）的头文件，它定义了数据链路层（以太网层）的相关结构和函数。以下是这段代码的主要作用：

1. 引入netlink头文件：netlink是Linux系统中的网络链路库，提供了用于网络编程的API，通过引入netlink头文件，可以使程序在编写网络编程代码时具有兼容性。

2. 引入socklen_t类型：socklen_t表示一个无单位的整数类型，它用于表示套接字（socket）的长度。在网络编程中，获取和设置socklen_t类型的变量可以方便地实现对套接字属性的访问。

3. 定义MAX_LINKHEADER_SIZE：这是一个定义，描述了数据链路层头部最大尺寸。这个值应该大于所有实际数据包中的MTU（最大传输单元）。

4. 包含NL_MAX_COUNT、NL_MAX_ORDINAL、NL_MAX_VIRT、NL_MAX_PENDING和NL_MAX_FIRST_CW等函数：它们定义了数据链路层头部中可用属性的取值范围。这些函数在后续的数据链路层编程中可以用来设置头部信息，例如，set_attr、set_net_attr等函数需要使用NL_MAX_FIRST_CW属性的值来确保数据包按正确的顺序发送。

5. 包含一个netlink_msg结构体：netlink_msg定义了数据链路层数据包的结构。在实际应用中，可以使用这个结构体来创建和处理数据包。

6. 使用set_attr函数设置NL_MAX_COUNT：set_attr函数可以设置NL_MAX_COUNT的值，用于指定可以支持的最大数据包数量。这个值应该根据实际系统中的网络带宽和实际情况进行调整，以实现数据包的正确发送和接收。

7. 使用set_net_attr函数设置NL_MAX_ORDINAL：set_net_attr函数可以设置NL_MAX_ORDINAL的值，用于指定可以支持的最大数据包顺序。这个值应该根据实际系统中的网络带宽和实际情况进行调整，以实现数据包的正确发送和接收。

8. 使用netlink的data链路层初始化函数：netlink的data链路层初始化函数定义了一系列用于设置数据链路层属性的函数，例如set_attribute、set_traffic_class、set_volume_aggregation等。这些函数可以帮助程序快速初始化数据链路层，使得数据包能够正常发送和接收。


```cpp
#include <netlink/genl/family.h>
#include <netlink/genl/ctrl.h>
#include <netlink/msg.h>
#include <netlink/attr.h>
#endif /* HAVE_LIBNL */

#ifndef HAVE_SOCKLEN_T
typedef int		socklen_t;
#endif

#define MAX_LINKHEADER_SIZE	256

/*
 * When capturing on all interfaces we use this as the buffer size.
 * Should be bigger then all MTUs that occur in real life.
 * 64kB should be enough for now.
 */
```

这段代码是一个头文件，定义了一个名为BIGGER_THAN_ALL_MTUS的结构体，其作用是定义了在Linux PF_PACKET打口中使用的私有数据。

具体来说，这个结构体包含以下字段：

1. sysfs_dropped：记录由/sys/class/net/{if_name}/statistics/rx_{missed,fifo}_errors packets reported dropped by /sys/class/net/{if_name}。
2. stat：包含struct pcap_stat类型，包含了接收到的数据包统计信息。
3. device：设备名称。
4. filter_in_userland：表示是否过滤进用户空间。
5. blocks_to_filter_in_userland：表示过滤器卷进入用户空间的数量。
6. must_do_on_close：在关闭时需要执行的操作。
7. timeout：缓冲区满后超时的时间。
8. cooked：表示使用SOCK_DGRAM而不是SOCK_RAW。
9. ifindex：表示我们绑定的接口索引。
10. lo_ifindex：表示回环设备的接口索引。
11. netdown：表示是否检测到NETDOWN信号，如果没有检测到，则表示继续尝试。
12. oldmode：表示在从监视模式切换回正常模式时需要执行的模式。
13. mac80211_device：mac80211监测设备的内存-mapped区域指针。
14. mmapbuf：指向内存-mapped区域的指针。
15. mmapbuflen：内存-mapped区域的大小。
16. vlan_offset：表示在插入VLAN标记时偏移量。
17. tp_version：表示接收到的TPCP协议版本。
18. tp_hdrlen：表示TPCP头部长度。
19. oneshot_buffer：用于复制接收到的数据包的缓冲区。
20. poll_timeout：表示在轮询中超时的时间。

这个头文件定义了结构体，用于在Linux PF_PACKET打口中使用，它可能是用于在接收数据包时进行过滤、统计、管理等操作。


```cpp
#define BIGGER_THAN_ALL_MTUS	(64*1024)

/*
 * Private data for capturing on Linux PF_PACKET sockets.
 */
struct pcap_linux {
	long long sysfs_dropped; /* packets reported dropped by /sys/class/net/{if_name}/statistics/rx_{missed,fifo}_errors */
	struct pcap_stat stat;

	char	*device;	/* device name */
	int	filter_in_userland; /* must filter in userland */
	int	blocks_to_filter_in_userland;
	int	must_do_on_close; /* stuff we must do when we close */
	int	timeout;	/* timeout for buffering */
	int	cooked;		/* using SOCK_DGRAM rather than SOCK_RAW */
	int	ifindex;	/* interface index of device we're bound to */
	int	lo_ifindex;	/* interface index of the loopback device */
	int	netdown;	/* we got an ENETDOWN and haven't resolved it */
	bpf_u_int32 oldmode;	/* mode to restore when turning monitor mode off */
	char	*mondevice;	/* mac80211 monitor device we created */
	u_char	*mmapbuf;	/* memory-mapped region pointer */
	size_t	mmapbuflen;	/* size of region */
	int	vlan_offset;	/* offset at which to insert vlan tags; if -1, don't insert */
	u_int	tp_version;	/* version of tpacket_hdr for mmaped ring */
	u_int	tp_hdrlen;	/* hdrlen of tpacket_hdr for mmaped ring */
	u_char	*oneshot_buffer; /* buffer for copy of packet */
	int	poll_timeout;	/* timeout to use in poll() */
```

这段代码是一个 Linux 系统调用中的函数头，定义了 TPACKET_V3 数据包处理函数需要的参数和变量。以下是代码的主要部分：

```cppc
#ifdef HAVE_TPACKET3
	unsigned char *current_packet; /* Current packet within the TPACKET_V3 block. Move to next block if NULL. */
	int packets_left; /* Unhandled packets left within the block from previous call to pcap_read_linux_mmap_v3 in case of TPACKET_V3. */
#endif
	int poll_breakloop_fd; /* fd to an eventfd to break from blocking operations */
```

首先，我们声明了两个变量 `current_packet` 和 `packets_left`。`current_packet` 是一个指针，用于保存当前正在处理的数据包。如果当前没有数据包，`packets_left` 变量用于记录没有处理完的包的数量。

接着，我们定义了一个 `poll_breakloop_fd` 变量，用于在等待下一个数据包时，通知操作系统事件循环，以便进行异步操作。

最后，我们需要包含头文件。由于我们没有具体的数据包处理函数，因此这个头文件不会被使用。


```cpp
#ifdef HAVE_TPACKET3
	unsigned char *current_packet; /* Current packet within the TPACKET_V3 block. Move to next block if NULL. */
	int packets_left; /* Unhandled packets left within the block from previous call to pcap_read_linux_mmap_v3 in case of TPACKET_V3. */
#endif
	int poll_breakloop_fd; /* fd to an eventfd to break from blocking operations */
};

/*
 * Stuff to do when we close.
 */
#define MUST_CLEAR_RFMON	0x00000001	/* clear rfmon (monitor) mode */
#define MUST_DELETE_MONIF	0x00000002	/* delete monitor-mode interface */

/*
 * Prototypes for internal functions and methods.
 */
```

该代码定义了多个名为thdr的结构体，其成员变量如下：

```cpp
struct tpacket2_hdr {
	int vw u1;
	u32 vw u2;
	u32 vw u3;
	u32 vw u4;
	u32 frame_length;
	u32 data_offset;
	u32 crc;
	u32 data_count;
	u32 wire_order;
	u32 message_type;
	u32 field1;
	u32 field2;
	u32 field3;
	u32 field4;
};
```

`get_if_flags`函数的作用是接收一个字符串和一个32位的整数，并将它们转换为`struct tpacket2_hdr`结构体，然后将结构体中的各个成员赋值，最后将它们作为参数返回。

`is_wifi`函数的作用是接收一个字符串，判断它是否包含"wifi"，如果是，则返回1，否则返回0。

`map_arphrd_to_dlt`函数的作用是将ARPHRD映射为DLLT格式，然后将DLLT文件保存到`/tmp/dllt_arphrd.bin`文件中，最后将文件名返回。

`pcap_activate_linux`函数的作用是在Linux系统上激活pcap抓包工具，然后将其返回。

`setup_socket`函数的作用是在Linux系统上创建一个套接字，然后将其返回。

`setup_mmapped`函数的作用是在Linux系统上创建一个mmapped文件，然后将其返回，最后将文件描述符返回。

`pcap_can_set_rfmon_linux`函数的作用是在Linux系统上设置CAN RFMON选项，然后将其返回。

`pcap_inject_linux`函数的作用是在Linux系统上向目标pcap实例注入数据，然后将其返回。

`pcap_stats_linux`函数的作用是在Linux系统上打印pcap统计信息，然后将其返回。

`pcap_setfilter_linux`函数的作用是在Linux系统上设置pcap过滤器，然后将其返回。

`pcap_setdirection_linux`函数的作用是在Linux系统上设置pcap方向，然后将其返回。

`pcap_set_datalink_linux`函数的作用是在Linux系统上设置数据链路，然后将其返回。

`pcap_cleanup_linux`函数的作用是在Linux系统上清理pcap实例，然后将其返回。


```cpp
static int get_if_flags(const char *, bpf_u_int32 *, char *);
static int is_wifi(const char *);
static void map_arphrd_to_dlt(pcap_t *, int, const char *, int);
static int pcap_activate_linux(pcap_t *);
static int setup_socket(pcap_t *, int);
static int setup_mmapped(pcap_t *, int *);
static int pcap_can_set_rfmon_linux(pcap_t *);
static int pcap_inject_linux(pcap_t *, const void *, int);
static int pcap_stats_linux(pcap_t *, struct pcap_stat *);
static int pcap_setfilter_linux(pcap_t *, struct bpf_program *);
static int pcap_setdirection_linux(pcap_t *, pcap_direction_t);
static int pcap_set_datalink_linux(pcap_t *, int);
static void pcap_cleanup_linux(pcap_t *);

union thdr {
	struct tpacket2_hdr		*h2;
```

这段代码定义了一个名为TPACKET3的结构体，其中包含一个指向名为h3的struct tpacket_block_desc类型的指针。在函数声明中，有一个预处理指令#ifdef HAVE_TPACKET3，它会先判断这个预处理指令是否已经定义过。如果没有定义过，那么它会让编译器自动定义这个预处理指令。

接着，代码中定义了一个名为raw的结构体，其中包含一个u_char类型的指针。这个结构体可能是在后续的代码中被用处过的，不过目前没有用处。

代码后面，定义了一个函数名为destroy_ring，它接受一个pcap_t类型的指针和一个offset，表示想要在链路中删除的帧的偏移量。这个函数可能是在链路中删除帧时使用的。

接着，定义了一个函数名为create_ring，它也接受一个pcap_t类型的指针和一个offset，表示想要创建一个新的链路。如果这个操作成功，它将返回一个表示状态的int类型的值。如果失败，它将返回-1。

接下来，定义了一个函数名为prepare_tpacket_socket，它接受一个pcap_t类型的指针和一个offset，表示想要为链路创建一个新的帧socket。如果这个操作成功，它将返回一个表示状态的int类型的值。如果失败，它将返回-1。

最后，定义了一个函数名为pcap_read_linux_mmap_v2和一个函数名为pcap_read_linux_mmap_v3，它们都接受一个pcap_t类型的指针和一个int和一个pcap_handler类型的指针，表示在链路中读取数据。这些函数使用了pcap_read_linux_mmap_v2和pcap_read_linux_mmap_v3函数，根据TPACKET3预处理指令的选择，会在内存中读取数据。


```cpp
#ifdef HAVE_TPACKET3
	struct tpacket_block_desc	*h3;
#endif
	u_char				*raw;
};

#define RING_GET_FRAME_AT(h, offset) (((u_char **)h->buffer)[(offset)])
#define RING_GET_CURRENT_FRAME(h) RING_GET_FRAME_AT(h, h->offset)

static void destroy_ring(pcap_t *handle);
static int create_ring(pcap_t *handle, int *status);
static int prepare_tpacket_socket(pcap_t *handle);
static int pcap_read_linux_mmap_v2(pcap_t *, int, pcap_handler , u_char *);
#ifdef HAVE_TPACKET3
static int pcap_read_linux_mmap_v3(pcap_t *, int, pcap_handler , u_char *);
```

This is a function definition for hotfile\_linux, which takes a user string, a pcap packet header, and a packet data as input. It appears to handle packets with a VLAN tag and also sets the flag tp\_status\_vlan\_valid to indicate the presence of a VLAN.


```cpp
#endif
static int pcap_setnonblock_linux(pcap_t *p, int nonblock);
static int pcap_getnonblock_linux(pcap_t *p);
static void pcap_oneshot_linux(u_char *user, const struct pcap_pkthdr *h,
    const u_char *bytes);

/*
 * In pre-3.0 kernels, the tp_vlan_tci field is set to whatever the
 * vlan_tci field in the skbuff is.  0 can either mean "not on a VLAN"
 * or "on VLAN 0".  There is no flag set in the tp_status field to
 * distinguish between them.
 *
 * In 3.0 and later kernels, if there's a VLAN tag present, the tp_vlan_tci
 * field is set to the VLAN tag, and the TP_STATUS_VLAN_VALID flag is set
 * in the tp_status field, otherwise the tp_vlan_tci field is set to 0 and
 * the TP_STATUS_VLAN_VALID flag isn't set in the tp_status field.
 *
 * With a pre-3.0 kernel, we cannot distinguish between packets with no
 * VLAN tag and packets on VLAN 0, so we will mishandle some packets, and
 * there's nothing we can do about that.
 *
 * So, on those systems, which never set the TP_STATUS_VLAN_VALID flag, we
 * continue the behavior of earlier libpcaps, wherein we treated packets
 * with a VLAN tag of 0 as being packets without a VLAN tag rather than packets
 * on VLAN 0.  We do this by treating packets with a tp_vlan_tci of 0 and
 * with the TP_STATUS_VLAN_VALID flag not set in tp_status as not having
 * VLAN tags.  This does the right thing on 3.0 and later kernels, and
 * continues the old unfixably-imperfect behavior on pre-3.0 kernels.
 *
 * If TP_STATUS_VLAN_VALID isn't defined, we test it as the 0x10 bit; it
 * has that value in 3.0 and later kernels.
 */
```

这段代码是检查TP（传输协议）状态中的VLAN（虚拟局域网）是否有效的定义。它可以分为两个部分。

首先，定义了一个名为VLAN_VALID的宏，它的作用是检查TP中的VLAN是否有效。这个宏的实现依赖于另一个名为TP_STATUS_VLAN_VALID的定义。

TP_STATUS_VLAN_VALID定义了一个名为TP_STATUS_VLAN_VALID的宏，它的作用是检查当前传输协议栈中是否有VLAN。如果当前系统支持VLAN，那么这个宏定义的值为真；否则，值为假。这个宏在后面的代码中被用来检查VLAN的有效性。

VLAN_VALID定义了一个名为VLAN_VALID的宏，它的作用与TP_STATUS_VLAN_VALID定义的作用相同。这个宏定义了一个名为VLAN_VALID的函数，它的第一个参数是头指针，第二个参数是硬件虚拟地址。这个函数的作用是检查当前的硬件虚拟地址是否与VLAN的有效性相关。如果VLAN有效，那么函数返回true；否则，函数返回false。这个函数在后面的代码中被用来检查VLAN的有效性。

综上所述，这段代码的作用是定义了一个名为VLAN_VALID的函数来检查TP中的VLAN是否有效，并且在当前系统上测试VLAN的有效性，如果系统支持VLAN，则返回true，否则返回false。


```cpp
#ifdef TP_STATUS_VLAN_VALID
  #define VLAN_VALID(hdr, hv)	((hv)->tp_vlan_tci != 0 || ((hdr)->tp_status & TP_STATUS_VLAN_VALID))
#else
  /*
   * This is being compiled on a system that lacks TP_STATUS_VLAN_VALID,
   * so we testwith the value it has in the 3.0 and later kernels, so
   * we can test it if we're running on a system that has it.  (If we're
   * running on a system that doesn't have it, it won't be set in the
   * tp_status field, so the tests of it will always fail; that means
   * we behave the way we did before we introduced this macro.)
   */
  #define VLAN_VALID(hdr, hv)	((hv)->tp_vlan_tci != 0 || ((hdr)->tp_status & 0x10))
#endif

#ifdef TP_STATUS_VLAN_TPID_VALID
```

这段代码定义了一个名为 VLAN_TPID 的函数，用于检查 VLAN 是否与 VLAN 控制台（hv）连接成功。如果连接成功，函数返回 VLAN 控制台的 VLAN TPID，否则返回 ETH_P_8021Q。

此外，代码中定义了一个名为 netdown_timeout 的结构体，用于在未检测到接口时强制关闭网络以防止 "interface disappeared" 事件的发生。netdown_timeout 结构体包含一个宏名为 0，表示在 1000 毫秒后关闭网络；一个名为 1 的宏，表示在 500 毫秒后关闭网络；一个名为 1000 的宏，表示强制关闭网络以防止 "interface disappeared" 事件的发生。


```cpp
# define VLAN_TPID(hdr, hv)	(((hv)->tp_vlan_tpid || ((hdr)->tp_status & TP_STATUS_VLAN_TPID_VALID)) ? (hv)->tp_vlan_tpid : ETH_P_8021Q)
#else
# define VLAN_TPID(hdr, hv)	ETH_P_8021Q
#endif

/*
 * Required select timeout if we're polling for an "interface disappeared"
 * indication - 1 millisecond.
 */
static const struct timeval netdown_timeout = {
	0, 1000		/* 1000 microseconds = 1 millisecond */
};

/*
 * Wrap some ioctl calls
 */
```

这段代码是一个用于网络接口编程的函数，实现了接口获取、MTU获取和AT类型设置等功能。主要部分如下：

1. 定义了4个静态函数，共用一个iface_get_id函数，函数接收两个整型参数：一个fd，表示网络接口的文件描述符；另一个是const char *device，表示设备名称。函数返回int类型的id，表示该接口的ID。

2. 同样，定义了4个静态函数，共用一个iface_get_mtu函数，函数同样接收fd和device参数，并返回int类型的mtu，表示该接口的最大传输单元。

3. 还定义了4个静态函数，共用一个iface_get_arptype函数，函数接收fd和device参数，并返回int类型的at类型，表示该接口的类型。

4. 最后，定义了4个静态函数，共用一个iface_bind函数，函数接收fd、ifIndex和ebuf参数，设置接口的监听模式，并调用iface_get_ts_types、iface_get_offload等函数，实现接口的绑定和获取信息的功能。

5. 有一个静态函数enter_rfmon_mode，用于进入接收射频广播模式，接受两个参数：pcap_t结构体指针和一个socket文件描述符，用于接收RFB广播包。函数使用rfmon_init函数初始化输入设备，并将输入设备设置为读写模式。

6. 有一个静态函数iface_get_ts_types，用于获取接口支持的数据类型数。函数接收device参数，返回int类型的ts_types，表示该接口支持的数据类型数。

7. 有一个静态函数iface_get_offload，用于获取接口的带宽利用率。函数没有参数，直接返回一个整型表示带宽利用率。


```cpp
static int	iface_get_id(int fd, const char *device, char *ebuf);
static int	iface_get_mtu(int fd, const char *device, char *ebuf);
static int	iface_get_arptype(int fd, const char *device, char *ebuf);
static int	iface_bind(int fd, int ifindex, char *ebuf, int protocol);
static int	enter_rfmon_mode(pcap_t *handle, int sock_fd,
    const char *device);
static int	iface_get_ts_types(const char *device, pcap_t *handle,
    char *ebuf);
static int	iface_get_offload(pcap_t *handle);

static int	fix_program(pcap_t *handle, struct sock_fprog *fcode);
static int	fix_offset(pcap_t *handle, struct bpf_insn *p);
static int	set_kernel_filter(pcap_t *handle, struct sock_fprog *fcode);
static int	reset_kernel_filter(pcap_t *handle);

```

This code appears to be a Linux utility that provides a DSA interface for a network interface. It appears to be part of a larger software包 or framework.

The `pcap_create_interface` function appears to be the main function for creating a network interface that is in line with the specified device. It takes a device name as an argument and an optional error buffer as a second argument. It creates a new `pcap_t` object, which is a representation of the network interface, and initializes some of the fields of the object.

The function first checks if the interface already exists, and if it does, it attempts to activate it. If it succeeds, it sets the `activate_op` field of the object to `pcap_activate_linux`, and sets the `can_set_rfmon_op` field to `pcap_can_set_rfmon_linux`, indicating that the interface supports filtering based on射频monitor signals.

Next, the function calls the `iface_get_ts_types` function to determine the timestamp types that the interface supports. If this function returns an error, the object is closed and the function returns NULL.

Finally, the function claims that the interface supports microsecond and nanosecond time stamps, and creates a variable to hold the timestamp precision list. If the allocation fails, the function returns NULL.

Overall, this function appears to be a key part of a network interface software package, and is used to create a Linux interface for a network device.


```cpp
static struct sock_filter	total_insn
	= BPF_STMT(BPF_RET | BPF_K, 0);
static struct sock_fprog	total_fcode
	= { 1, &total_insn };

static int	iface_dsa_get_proto_info(const char *device, pcap_t *handle);

pcap_t *
pcap_create_interface(const char *device, char *ebuf)
{
	pcap_t *handle;

	handle = PCAP_CREATE_COMMON(ebuf, struct pcap_linux);
	if (handle == NULL)
		return NULL;

	handle->activate_op = pcap_activate_linux;
	handle->can_set_rfmon_op = pcap_can_set_rfmon_linux;

	/*
	 * See what time stamp types we support.
	 */
	if (iface_get_ts_types(device, handle, ebuf) == -1) {
		pcap_close(handle);
		return NULL;
	}

	/*
	 * We claim that we support microsecond and nanosecond time
	 * stamps.
	 *
	 * XXX - with adapter-supplied time stamps, can we choose
	 * microsecond or nanosecond time stamps on arbitrary
	 * adapters?
	 */
	handle->tstamp_precision_list = malloc(2 * sizeof(u_int));
	if (handle->tstamp_precision_list == NULL) {
		pcap_fmt_errmsg_for_errno(ebuf, PCAP_ERRBUF_SIZE,
		    errno, "malloc");
		pcap_close(handle);
		return NULL;
	}
	handle->tstamp_precision_list[0] = PCAP_TSTAMP_PRECISION_MICRO;
	handle->tstamp_precision_list[1] = PCAP_TSTAMP_PRECISION_NANO;
	handle->tstamp_precision_count = 2;

	struct pcap_linux *handlep = handle->priv;
	handlep->poll_breakloop_fd = eventfd(0, EFD_NONBLOCK);

	return handle;
}

```

This is the contents of the `/etc/airmon-ng.conf` file. It defines a script that allows you to create, configure, and monitor 802.11 wireless devices. The script searches for devices with the name "monN" starting from "mon0", and if that doesn't exist, it chooses "mon0" as the monitor device name. The script uses the `/sys/class/net/<if_name>/phy80211` links to connect the devices to the same physical layer (PHY), and if a file `/sys/class/ieee80211/<phydev_name>/add_iface` exists, it writes the `mondev_name` to that file and then sleeps for 1 second before doing so. If the PHY is not found, the script will configure the device into monitor mode and write the `mondev_name` to the file `/sys/class/ieee80211/<phydev_name>/add_iface` with no newline and a 1-second sleep.


```cpp
#ifdef HAVE_LIBNL
/*
 * If interface {if_name} is a mac80211 driver, the file
 * /sys/class/net/{if_name}/phy80211 is a symlink to
 * /sys/class/ieee80211/{phydev_name}, for some {phydev_name}.
 *
 * On Fedora 9, with a 2.6.26.3-29 kernel, my Zydas stick, at
 * least, has a "wmaster0" device and a "wlan0" device; the
 * latter is the one with the IP address.  Both show up in
 * "tcpdump -D" output.  Capturing on the wmaster0 device
 * captures with 802.11 headers.
 *
 * airmon-ng searches through /sys/class/net for devices named
 * monN, starting with mon0; as soon as one *doesn't* exist,
 * it chooses that as the monitor device name.  If the "iw"
 * command exists, it does
 *
 *    iw dev {if_name} interface add {monif_name} type monitor
 *
 * where {monif_name} is the monitor device.  It then (sigh) sleeps
 * .1 second, and then configures the device up.  Otherwise, if
 * /sys/class/ieee80211/{phydev_name}/add_iface is a file, it writes
 * {mondev_name}, without a newline, to that file, and again (sigh)
 * sleeps .1 second, and then iwconfig's that device into monitor
 * mode and configures it up.  Otherwise, you can't do monitor mode.
 *
 * All these devices are "glued" together by having the
 * /sys/class/net/{if_name}/phy80211 links pointing to the same
 * place, so, given a wmaster, wlan, or mon device, you can
 * find the other devices by looking for devices with
 * the same phy80211 link.
 *
 * To turn monitor mode off, delete the monitor interface,
 * either with
 *
 *    iw dev {monif_name} interface del
 *
 * or by sending {monif_name}, with no NL, down
 * /sys/class/ieee80211/{phydev_name}/remove_iface
 *
 * Note: if you try to create a monitor device named "monN", and
 * there's already a "monN" device, it fails, as least with
 * the netlink interface (which is what iw uses), with a return
 * value of -ENFILE.  (Return values are negative errnos.)  We
 * could probably use that to find an unused device.
 *
 * Yes, you can have multiple monitor devices for a given
 * physical device.
 */

```

这段代码的作用是判断一个设备是否为苹果80211设备，如果是，则返回该设备的物理设备路径，如果不是，则返回0。在函数内部，首先通过asprintf函数将一个字符串类型的设备路径参数device赋值给一个字符数组路径str，然后使用readlink函数读取该路径str到phydev_path参数的结束，即phydev_max_pathlen-1个字符。如果函数内部成功执行此操作，则说明该设备是一个苹果80211设备，否则返回0。函数内部还使用了errno变量来获取错误信息，如果错误，则通过errbuf参数返回一个错误信息字符串，并free(pathstr)和free(pathstr)来释放路径str和无用内存。


```cpp
/*
 * Is this a mac80211 device?  If so, fill in the physical device path and
 * return 1; if not, return 0.  On an error, fill in handle->errbuf and
 * return PCAP_ERROR.
 */
static int
get_mac80211_phydev(pcap_t *handle, const char *device, char *phydev_path,
    size_t phydev_max_pathlen)
{
	char *pathstr;
	ssize_t bytes_read;

	/*
	 * Generate the path string for the symlink to the physical device.
	 */
	if (asprintf(&pathstr, "/sys/class/net/%s/phy80211", device) == -1) {
		snprintf(handle->errbuf, PCAP_ERRBUF_SIZE,
		    "%s: Can't generate path name string for /sys/class/net device",
		    device);
		return PCAP_ERROR;
	}
	bytes_read = readlink(pathstr, phydev_path, phydev_max_pathlen);
	if (bytes_read == -1) {
		if (errno == ENOENT || errno == EINVAL) {
			/*
			 * Doesn't exist, or not a symlink; assume that
			 * means it's not a mac80211 device.
			 */
			free(pathstr);
			return 0;
		}
		pcap_fmt_errmsg_for_errno(handle->errbuf, PCAP_ERRBUF_SIZE,
		    errno, "%s: Can't readlink %s", device, pathstr);
		free(pathstr);
		return PCAP_ERROR;
	}
	free(pathstr);
	phydev_path[bytes_read] = '\0';
	return 1;
}

```

这段代码定义了一个名为 nl80211_state 的结构体，用于表示 NL80211 无线网络适配器的状态信息。

在 nl80211_init 函数中，首先通过 nl_socket_alloc 函数分配一个 NL80211 网络套接字。接着使用 genl_connect 函数尝试连接到远程 NL80211 设备。如果连接成功，使用 genl_ctrl_alloc_cache 函数分配一个 NL80211 设备对应的 NL80211 高速缓存。最后，使用 genl_ctrl_search_by_name 函数查找 NL80211 高速缓存中是否有名为 "nl80211" 的设备。如果找到了，将 nl80211 设备的信息存储到 state 结构体中，否则输出错误信息并跳转到 out_handle_destroy 函数。


```cpp
struct nl80211_state {
	struct nl_sock *nl_sock;
	struct nl_cache *nl_cache;
	struct genl_family *nl80211;
};

static int
nl80211_init(pcap_t *handle, struct nl80211_state *state, const char *device)
{
	int err;

	state->nl_sock = nl_socket_alloc();
	if (!state->nl_sock) {
		snprintf(handle->errbuf, PCAP_ERRBUF_SIZE,
		    "%s: failed to allocate netlink handle", device);
		return PCAP_ERROR;
	}

	if (genl_connect(state->nl_sock)) {
		snprintf(handle->errbuf, PCAP_ERRBUF_SIZE,
		    "%s: failed to connect to generic netlink", device);
		goto out_handle_destroy;
	}

	err = genl_ctrl_alloc_cache(state->nl_sock, &state->nl_cache);
	if (err < 0) {
		snprintf(handle->errbuf, PCAP_ERRBUF_SIZE,
		    "%s: failed to allocate generic netlink cache: %s",
		    device, nl_geterror(-err));
		goto out_handle_destroy;
	}

	state->nl80211 = genl_ctrl_search_by_name(state->nl_cache, "nl80211");
	if (!state->nl80211) {
		snprintf(handle->errbuf, PCAP_ERRBUF_SIZE,
		    "%s: nl80211 not found", device);
		goto out_cache_free;
	}

	return 0;

```



这是一段 C 语言代码，属于 Linux 网络适配器驱动程序中的一个函数。其作用是实现nl80211_cleanup函数。

nl80211_cleanup函数是用来完成nl80211无线网络适配器的清理工作，包括释放相关资源。具体来说，该函数会首先通过nl_cache_free函数释放nl80211_cache缓存，然后通过nl_socket_free函数释放nl80211_sock套接字，最后返回一个错误码。

该函数的实现可以被看作是清理nl80211无线网络适配器的一些资源，以便于恢复设备到出厂默认状态。


```cpp
out_cache_free:
	nl_cache_free(state->nl_cache);
out_handle_destroy:
	nl_socket_free(state->nl_sock);
	return PCAP_ERROR;
}

static void
nl80211_cleanup(struct nl80211_state *state)
{
	genl_family_put(state->nl80211);
	nl_cache_free(state->nl_cache);
	nl_socket_free(state->nl_sock);
}

```

这段代码是一个名为“del_mon_if”的函数，属于“pcap”和“nl80211”目录。它的作用是监控设备（设备ID）并启用或禁用NH硬件接口（如果存在）。以下是它的实现细节：

1. 函数参数：
  - pcap_t：指向捕获器（网络数据包收集器）的句柄。
  - sock_fd：目标套接字（网络数据包输出socket的文件描述符）。
  - struct nl80211_state：NL80211无线网络状态结构，包含用于NL80211接口的配置和统计信息。
  - 设备ID（字符串）：要监控的设备。
  - 目标设备ID：NL80211接口的目标ID（也称为IFINDEX）。

2. 函数实现：
  - 如果NL80211接口不存在，函数返回PCAP_ERROR。
  - 创建一个struct pcap_linux类型的变量handlep，该变量与输入handle句柄相关联。
  - 获取当前套接字对应的设备ID，并将其存储在ifindex变量中。
  - 创建一个struct nl_msg结构体，用于存储NL80211网络数据包。
  - nlmsg_alloc函数用于分配内存空间，如果分配失败，函数将使用handle错误缓冲区中的错误消息。
  - 使用nlmsg_genl函数生成一个NL80211网络数据包，其中包含设备ID、SO_PEER和NL80211_CMD_NEW_INTERFACE标志。
  - 将生成的NL80211网络数据包的发送者在ifindex变量中设置为设备ID，并将其发送到目标套接字。
  - NLA_PUT_U32函数将NL80211接口的IFINDEX值存储到nlmsg结构体中。

3. 函数输出：
  - 如果没有错误，函数返回PCAP_SUCCESS。


```cpp
static int
del_mon_if(pcap_t *handle, int sock_fd, struct nl80211_state *state,
    const char *device, const char *mondevice);

static int
add_mon_if(pcap_t *handle, int sock_fd, struct nl80211_state *state,
    const char *device, const char *mondevice)
{
	struct pcap_linux *handlep = handle->priv;
	int ifindex;
	struct nl_msg *msg;
	int err;

	ifindex = iface_get_id(sock_fd, device, handle->errbuf);
	if (ifindex == -1)
		return PCAP_ERROR;

	msg = nlmsg_alloc();
	if (!msg) {
		snprintf(handle->errbuf, PCAP_ERRBUF_SIZE,
		    "%s: failed to allocate netlink msg", device);
		return PCAP_ERROR;
	}

	genlmsg_put(msg, 0, 0, genl_family_get_id(state->nl80211), 0,
		    0, NL80211_CMD_NEW_INTERFACE, 0);
	NLA_PUT_U32(msg, NL80211_ATTR_IFINDEX, ifindex);
```

This is a function that sets up a NLM device and adds a new interface using the `nl_device_add()` function. It returns an error code on success or failure.

The function takes several parameters:

* `device`: The name of the device to be added to the NLM.
* `mondevice`: The name of the monitor device to be associated with the NLM device.
* `nl_geterror()`: A function that returns the error associated with the `nl_device_add()` call.
* `errno`: An optional error number from `nl_geterror()` that can be used to display a more complete error message.
* `sock_fd`: The file descriptor of the new interface.
* `state`: An optional pointer to the NLM state structure.
* `device`: The device to be added to the NLM.
* `mondevice`: The name of the monitor device to be associated with the NLM device.
* `nl_device_add()`: The function that does the actual device addition.
* `errbuf`: The buffer that will be used to store the error message.

The function first checks for the device to be selected successfully. If the device cannot be found, an error message is displayed and the function returns an error code. If the device is selected successfully, the function sets the `mondevice` parameter to the new device name and returns 1.


```cpp
DIAG_OFF_NARROWING
	NLA_PUT_STRING(msg, NL80211_ATTR_IFNAME, mondevice);
DIAG_ON_NARROWING
	NLA_PUT_U32(msg, NL80211_ATTR_IFTYPE, NL80211_IFTYPE_MONITOR);

	err = nl_send_auto_complete(state->nl_sock, msg);
	if (err < 0) {
		if (err == -NLE_FAILURE) {
			/*
			 * Device not available; our caller should just
			 * keep trying.  (libnl 2.x maps ENFILE to
			 * NLE_FAILURE; it can also map other errors
			 * to that, but there's not much we can do
			 * about that.)
			 */
			nlmsg_free(msg);
			return 0;
		} else {
			/*
			 * Real failure, not just "that device is not
			 * available.
			 */
			snprintf(handle->errbuf, PCAP_ERRBUF_SIZE,
			    "%s: nl_send_auto_complete failed adding %s interface: %s",
			    device, mondevice, nl_geterror(-err));
			nlmsg_free(msg);
			return PCAP_ERROR;
		}
	}
	err = nl_wait_for_ack(state->nl_sock);
	if (err < 0) {
		if (err == -NLE_FAILURE) {
			/*
			 * Device not available; our caller should just
			 * keep trying.  (libnl 2.x maps ENFILE to
			 * NLE_FAILURE; it can also map other errors
			 * to that, but there's not much we can do
			 * about that.)
			 */
			nlmsg_free(msg);
			return 0;
		} else {
			/*
			 * Real failure, not just "that device is not
			 * available.
			 */
			snprintf(handle->errbuf, PCAP_ERRBUF_SIZE,
			    "%s: nl_wait_for_ack failed adding %s interface: %s",
			    device, mondevice, nl_geterror(-err));
			nlmsg_free(msg);
			return PCAP_ERROR;
		}
	}

	/*
	 * Success.
	 */
	nlmsg_free(msg);

	/*
	 * Try to remember the monitor device.
	 */
	handlep->mondevice = strdup(mondevice);
	if (handlep->mondevice == NULL) {
		pcap_fmt_errmsg_for_errno(handle->errbuf, PCAP_ERRBUF_SIZE,
		    errno, "strdup");
		/*
		 * Get rid of the monitor device.
		 */
		del_mon_if(handle, sock_fd, state, device, mondevice);
		return PCAP_ERROR;
	}
	return 1;

```

device 和 mondevice 是指网络接口设备和媒体设备。这个问题要求我们获取当前网络接口设备（device）和媒体设备（mondevice）的索引号。

首先，通过 iface_get_id 函数获取网络接口设备（device）的索引号。如果该函数返回 -1，说明设备不存在，抛出错误。

然后，使用 nlmsg_alloc 函数获取媒体设备（mondevice）的媒体类型信息，如果该函数返回 -1，说明设备不存在，抛出错误。

接着，使用 nl_msg 结构体中的 genl_family_get_id 函数获取设备的 ID，并使用 nl_send_auto_complete 函数删除该设备，如果函数成功，返回 0，否则返回 -2，使用 nl_wait_for_ack 函数等待删除操作完成，如果函数成功，返回 0，否则返回 -2。

最后，如果所有函数都成功，返回 1，否则返回 PCAP_ERROR。


```cpp
nla_put_failure:
	snprintf(handle->errbuf, PCAP_ERRBUF_SIZE,
	    "%s: nl_put failed adding %s interface",
	    device, mondevice);
	nlmsg_free(msg);
	return PCAP_ERROR;
}

static int
del_mon_if(pcap_t *handle, int sock_fd, struct nl80211_state *state,
    const char *device, const char *mondevice)
{
	int ifindex;
	struct nl_msg *msg;
	int err;

	ifindex = iface_get_id(sock_fd, mondevice, handle->errbuf);
	if (ifindex == -1)
		return PCAP_ERROR;

	msg = nlmsg_alloc();
	if (!msg) {
		snprintf(handle->errbuf, PCAP_ERRBUF_SIZE,
		    "%s: failed to allocate netlink msg", device);
		return PCAP_ERROR;
	}

	genlmsg_put(msg, 0, 0, genl_family_get_id(state->nl80211), 0,
		    0, NL80211_CMD_DEL_INTERFACE, 0);
	NLA_PUT_U32(msg, NL80211_ATTR_IFINDEX, ifindex);

	err = nl_send_auto_complete(state->nl_sock, msg);
	if (err < 0) {
		snprintf(handle->errbuf, PCAP_ERRBUF_SIZE,
		    "%s: nl_send_auto_complete failed deleting %s interface: %s",
		    device, mondevice, nl_geterror(-err));
		nlmsg_free(msg);
		return PCAP_ERROR;
	}
	err = nl_wait_for_ack(state->nl_sock);
	if (err < 0) {
		snprintf(handle->errbuf, PCAP_ERRBUF_SIZE,
		    "%s: nl_wait_for_ack failed adding %s interface: %s",
		    device, mondevice, nl_geterror(-err));
		nlmsg_free(msg);
		return PCAP_ERROR;
	}

	/*
	 * Success.
	 */
	nlmsg_free(msg);
	return 1;

```

这段代码是来自nl_put函数中的两个函数，功能是打印设备驱动程序(device)和模块(mondevice)的错误信息，并返回PCAP_ERROR。

具体来说，代码首先通过`snprintf`函数创建一个输出缓冲区(errbuf)，该缓冲区用于存储设备驱动程序和模块返回的错误信息。然后，代码使用`nlmsg_free`函数释放之前分配的`nl_msg`结构体，该结构体包含设备失败时需要打印的消息。

接下来，代码使用嵌套的`if`语句检查设备是否支持`nl_put`函数。如果不支持，则将`protocol`设置为`ETH_P_ALL`，否则将`protocol`设置为设备支持的协议类型。

最后，代码通过`htons`函数将`protocol`转换为整数类型，并将其返回。


```cpp
nla_put_failure:
	snprintf(handle->errbuf, PCAP_ERRBUF_SIZE,
	    "%s: nl_put failed deleting %s interface",
	    device, mondevice);
	nlmsg_free(msg);
	return PCAP_ERROR;
}
#endif /* HAVE_LIBNL */

static int pcap_protocol(pcap_t *handle)
{
	int protocol;

	protocol = handle->opt.protocol;
	if (protocol == 0)
		protocol = ETH_P_ALL;

	return htons(protocol);
}

```

这段代码是一个用于在 Linux 系统上设置can模块的RFMon信号的静态函数。它接受一个can协议头类型的指针参数，返回值是一个整数。以下是函数的实现细节：

1. 检查输入的handle对象是否指向任何设备，如果是，则直接返回0，因为监测模式无法应用于"any"设备。
2. 如果handle对象指向的设备不是"any"，则执行以下操作：
3. 检查给定的设备是否为"can0"。如果是，则执行以下操作：
   a. 设置can0设备的数据路径。
   b. 设置设备的中断为NL_FIRQ。
   c. 尝试配置can0以监测RFMon信号。如果配置成功，则返回1，否则返回0。
   d. 在函数内部，如果配置失败，则输出错误并返回-1。
   e. 如果配置成功，则继续执行下面的操作。
   f. 在设置NL_FIRQ后，尝试使用can0的滤波器。如果配置成功，则返回1，否则返回0。
   g. 由于已经配置为监测RFMon信号，因此不推荐使用can0的硬件中断，以免影响系统性能。
4. 如果配置步骤a-f成功，则返回1。

总之，该函数的主要作用是设置Linux系统上can模块的RFMon信号，以便在can0设备上实现对RFMon信号的监测。它根据设备类型和配置尝试不同的操作，并记录失败的情况。


```cpp
static int
pcap_can_set_rfmon_linux(pcap_t *handle)
{
#ifdef HAVE_LIBNL
	char phydev_path[PATH_MAX+1];
	int ret;
#endif

	if (strcmp(handle->opt.device, "any") == 0) {
		/*
		 * Monitor mode makes no sense on the "any" device.
		 */
		return 0;
	}

```

这段代码是一个C语言函数，名为“get_mac80211_phydev”。它实现了对一个mac80211设备的判断，判断设备是否支持monitor模式。

代码首先通过#ifdef Crisis产生一个条件判断语句，如果HAVE_LIBNL环境变量存在，则执行下面的代码；否则跳过。如果HAVE_LIBNL环境变量存在，则执行get_mac80211_phydev函数。如果函数执行成功，则返回值为1；如果执行失败，则返回值为-1。

get_mac80211_phydev函数的实现如下：
```cpp
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <sys/types.h>
#include <sys/ioctl.h>
#include <linux/input.h>

#define MAX_BUF_LEN 256

int get_mac80211_phydev(int handle, const char *device, const char *path, int *ret) {
   int fd = open(path, O_RDWR | O_NOCTTY | O_NDELAY);
   if (fd < 0) {
       *ret = -1;
       return -1;
   }

   char buf[MAX_BUF_LEN];
   int len = 0;
   while ((int)read(fd, buf, MAX_BUF_LEN) > 0) {
       char device_desc[32];
       sprintf(device_desc, "%s", buf[0]);
       if (strstr(device_desc, "0x%02x") == NULL) {
           device_desc[31] = '\0';
       }
       else {
           device_desc[31] = '\0';
       }
       len++;
   }

   close(fd);
   return (int)strtoupper(device_desc);
}
```
该函数通过输入输出以二进制字符串形式读取设备描述符，并判断其是否为mac80211设备。


```cpp
#ifdef HAVE_LIBNL
	/*
	 * Bleah.  There doesn't seem to be a way to ask a mac80211
	 * device, through libnl, whether it supports monitor mode;
	 * we'll just check whether the device appears to be a
	 * mac80211 device and, if so, assume the device supports
	 * monitor mode.
	 */
	ret = get_mac80211_phydev(handle, handle->opt.device, phydev_path,
	    PATH_MAX);
	if (ret < 0)
		return ret;	/* error */
	if (ret == 1)
		return 1;	/* mac80211 device */
#endif

	return 0;
}

```

这段代码的作用是获取一个网络接口的统计信息，其中包括未正确发送的数据包数量。与/proc/net/dev这个统计信息相比，这个函数避免了计数软件掉包，但可能未实现，或者实现方式与期望的不一样。

函数接受两个参数：一个接口名称（/sys/class/net/{if_name}）和一个统计信息（/proc/net/dev中的"rx_{missed,fifo}"）。函数内部先通过snprintf函数将接口名称和统计信息拼接成一个字符串，然后使用open函数打开该接口的统计文件，使用O_RDONLY参数以只读方式打开。如果打开成功，函数读取文件中的字节并关闭文件，然后将文件中的字节转换成long long类型并返回。如果函数在执行过程中遇到错误，如read函数遇到错误，则返回0。


```cpp
/*
 * Grabs the number of missed packets by the interface from
 * /sys/class/net/{if_name}/statistics/rx_{missed,fifo}_errors.
 *
 * Compared to /proc/net/dev this avoids counting software drops,
 * but may be unimplemented and just return 0.
 * The author has found no straigthforward way to check for support.
 */
static long long int
linux_get_stat(const char * if_name, const char * stat) {
	ssize_t bytes_read;
	int fd;
	char buffer[PATH_MAX];

	snprintf(buffer, sizeof(buffer), "/sys/class/net/%s/statistics/%s", if_name, stat);
	fd = open(buffer, O_RDONLY);
	if (fd == -1)
		return 0;

	bytes_read = read(fd, buffer, sizeof(buffer) - 1);
	close(fd);
	if (bytes_read == -1)
		return 0;
	buffer[bytes_read] = '\0';

	return strtoll(buffer, NULL, 10);
}

```

这段代码是一个用于监控Linux接口的工具，主要作用是统计Linux接口的数据。代码中定义了一个名为linux_if_drops的静态函数，该函数接受一个字符串参数if_name，用于指定要统计的接口名称。函数内部首先通过linux_get_stat函数获取接口 misses 和fifo 错误数量，然后将它们与之前的missed相加，得到一个long long类型的结果。函数的作用是提供一个统一的接口，让用户可以方便地查看多个接口的数据，并在关闭接口时进行统计。


```cpp
static long long int
linux_if_drops(const char * if_name)
{
	long long int missed = linux_get_stat(if_name, "rx_missed_errors");
	long long int fifo = linux_get_stat(if_name, "rx_fifo_errors");
	return missed + fifo;
}


/*
 * Monitor mode is kind of interesting because we have to reset the
 * interface before exiting. The problem can't really be solved without
 * some daemon taking care of managing usage counts.  If we put the
 * interface into monitor mode, we set a flag indicating that we must
 * take it out of that mode when the interface is closed, and, when
 * closing the interface, if that flag is set we take it out of monitor
 * mode.
 */

```

这段代码是针对 Linux 系统的 pcap 库中的 pcap_cleanup_linux 函数。

该函数的目的是在 pcap 库关闭时执行一些清理操作，包括：

1. 如果 pcap_t 实例的 must_do_on_close 属性为非零，则执行一些必须要在关闭时执行的操作。

2. 如果 pcap 库使用 libnl80211 库，则创建一个 nl80211 状态对象并执行 nl80211_init 函数，设置状态对象的 nlstate。然后使用 nl80211_cleanup 函数清除状态对象并删除 monitor 接口。

3. 如果 nl80211 库不能用于使用 libnl80211，则在关闭时执行一些必须要在关闭时执行的操作，如打印错误消息。

pcap_cleanup_linux 函数是 pcap 库中的一个重要函数，用于在关闭时执行一些必要的清理操作，确保 pcap 库关闭时干净地关闭。


```cpp
static void	pcap_cleanup_linux( pcap_t *handle )
{
	struct pcap_linux *handlep = handle->priv;
#ifdef HAVE_LIBNL
	struct nl80211_state nlstate;
	int ret;
#endif /* HAVE_LIBNL */

	if (handlep->must_do_on_close != 0) {
		/*
		 * There's something we have to do when closing this
		 * pcap_t.
		 */
#ifdef HAVE_LIBNL
		if (handlep->must_do_on_close & MUST_DELETE_MONIF) {
			ret = nl80211_init(handle, &nlstate, handlep->device);
			if (ret >= 0) {
				ret = del_mon_if(handle, handle->fd, &nlstate,
				    handlep->device, handlep->mondevice);
				nl80211_cleanup(&nlstate);
			}
			if (ret < 0) {
				fprintf(stderr,
				    "Can't delete monitor interface %s (%s).\n"
				    "Please delete manually.\n",
				    handlep->mondevice, handle->errbuf);
			}
		}
```

这段代码的作用是释放PCAP实例 handle 所代表的网络套接字的相关资源。handle 是一个 PCAP 实例的句柄，包含了 PCAP 实例的相关信息和处理函数。在释放 handle 句柄时，需要将其从所有与之相关的 PCAP 实例中移除，并确保所有相关的资源都被正确地释放。

具体来说，代码中执行以下操作：

1. 从 PCAP 实例列表中移除 handle 句柄。

2. 如果 handle 句柄已经被创建了一个套接字，则销毁该套接字。

3. 如果 handle 句柄中包含一个 one_shot 缓冲区，则释放该缓冲区。

4. 如果 handle 句柄中包含一个 mon_device 指针，则释放该指针。

5. 如果 handle 句柄中包含一个 device 指针，则释放该指针。

6. 如果 handle 句柄中包含一个 poll_breakloop_fd 指针，则关闭该指针。

7. 如果 handle 句柄中包含一个已经被使用的 fd 指针，则关闭该指针。

8. 使用 pcap_cleanup_live_common 函数清洗 PCAP 套接字。


```cpp
#endif /* HAVE_LIBNL */

		/*
		 * Take this pcap out of the list of pcaps for which we
		 * have to take the interface out of some mode.
		 */
		pcap_remove_from_pcaps_to_close(handle);
	}

	if (handle->fd != -1) {
		/*
		 * Destroy the ring buffer (assuming we've set it up),
		 * and unmap it if it's mapped.
		 */
		destroy_ring(handle);
	}

	if (handlep->oneshot_buffer != NULL) {
		free(handlep->oneshot_buffer);
		handlep->oneshot_buffer = NULL;
	}

	if (handlep->mondevice != NULL) {
		free(handlep->mondevice);
		handlep->mondevice = NULL;
	}
	if (handlep->device != NULL) {
		free(handlep->device);
		handlep->device = NULL;
	}

	if (handlep->poll_breakloop_fd != -1) {
		close(handlep->poll_breakloop_fd);
		handlep->poll_breakloop_fd = -1;
	}
	pcap_cleanup_live_common(handle);
}

```

这段代码是一个名为`has_broken_tpacket_v3`的函数，它用于检查系统是否支持TPACKET_V3版本。这个函数的实现基于以下条件判断：

1. 如果系统包含TPACKET_V3版本，则不需要做进一步的处理。
2. 如果系统不包含TPACKET_V3版本或者不支持版本3.19及以上的版本，则认为TPACKET_V3存在严重的错误或特性问题。

该函数的实现比较简单，主要通过检查两个条件来判断TPACKET_V3是否支持版本3.19及以上的版本。如果系统不支持TPACKET_V3或者不支持版本3.19及以上的版本，则返回1，表示存在问题。如果系统包含TPACKET_V3版本，则不做进一步的处理，避免不必要的警告和错误。


```cpp
#ifdef HAVE_TPACKET3
/*
 * Some versions of TPACKET_V3 have annoying bugs/misfeatures
 * around which we have to work.  Determine if we have those
 * problems or not.
 * 3.19 is the first release with a fixed version of
 * TPACKET_V3.  We treat anything before that as
 * not having a fixed version; that may really mean
 * it has *no* version.
 */
static int has_broken_tpacket_v3(void)
{
	struct utsname utsname;
	const char *release;
	long major, minor;
	int matches, verlen;

	/* No version information, assume broken. */
	if (uname(&utsname) == -1)
		return 1;
	release = utsname.release;

	/* A malformed version, ditto. */
	matches = sscanf(release, "%ld.%ld%n", &major, &minor, &verlen);
	if (matches != 2)
		return 1;
	if (release[verlen] != '.' && release[verlen] != '\0')
		return 1;

	/* OK, a fixed version. */
	if (major > 3 || (major == 3 && minor >= 19))
		return 0;

	/* Too old :( */
	return 1;
}
```

这段代码是一个静态函数，名为`set_poll_timeout()`，它用于设置poll()函数的超时时间。

超时时间（timeout）是poll()函数用于等待数据包的一个时间限制。在代码中，首先检查`handlep`参数是否已经设置了一个超时时间，如果已设置则跳过这一步。如果没有设置，则会根据`handlep->timeout`的值来设置超时时间。

接下来，代码会判断两个条件。第一个条件是`handlep->tp_version`是否为TPACKET_V3，如果是，则需要设置一个`handlep->poll_timeout`，以防止阻塞太久。第二个条件是`has_broken_tpacket_v3()`是否为真，如果是，则可以更快地设置超时时间。

总的来说，这段代码的作用是设置poll()函数的超时时间，使其不会阻塞太久。如果TPACKET_V3版本较老，或者在某些情况下无法设置长时监视，则会发生阻塞。


```cpp
#endif

/*
 * Set the timeout to be used in poll() with memory-mapped packet capture.
 */
static void
set_poll_timeout(struct pcap_linux *handlep)
{
#ifdef HAVE_TPACKET3
	int broken_tpacket_v3 = has_broken_tpacket_v3();
#endif
	if (handlep->timeout == 0) {
#ifdef HAVE_TPACKET3
		/*
		 * XXX - due to a set of (mis)features in the TPACKET_V3
		 * kernel code prior to the 3.19 kernel, blocking forever
		 * with a TPACKET_V3 socket can, if few packets are
		 * arriving and passing the socket filter, cause most
		 * packets to be dropped.  See libpcap issue #335 for the
		 * full painful story.
		 *
		 * The workaround is to have poll() time out very quickly,
		 * so we grab the frames handed to us, and return them to
		 * the kernel, ASAP.
		 */
		if (handlep->tp_version == TPACKET_V3 && broken_tpacket_v3)
			handlep->poll_timeout = 1;	/* don't block for very long */
		else
```

这段代码是针对一个网络套接字（handlep）进行轮询（poll）调用的。以下是对代码的解释：

首先，在函数头部声明了一个名为handlep的结构体，其中包含一个名为poll_timeout的整型变量和一个名为timeout的整型变量。

接着，根据handlep->timeout和handlep->tp_version的值，进行以下分支：

1. 如果handlep->timeout为0，且handlep->tp_version不等于TPACKET_V3，则执行以下语句：

	```cpp
		handlep->poll_timeout = -1;
	```

	这个语句的作用是让handlep永远阻塞，不再接收数据包，直到接收到TPACKET_V3数据包为止。这样，当TPACKET_V3套接字真正支持轮询时，我们也不必担心由于时间过长而导致的阻塞问题。

2. 如果handlep->timeout大于0，且handlep->tp_version等于TPACKET_V3，则执行以下语句：

	```cpp
		handlep->poll_timeout = -1;
	```

	这个语句与上面的分支1的作用是相同的，只是TPACKET_V3已经支持轮询，轮询不会导致阻塞。

3. 如果handlep->timeout大于0，且handlep->tp_version不等于TPACKET_V3，则执行以下语句：

	```cpp
		handlep->poll_timeout = handlep->timeout;
	```

	这个语句表示在非阻塞模式下，轮询请求将阻塞调用，但不会阻塞当前套接字。handlep->poll_timeout将设置为当前套接字的发送超时时间（handlep->timeout）。

4. 如果handlep->timeout为0，且handlep->tp_version不等于TPACKET_V3，则执行以下语句：

	```cpp
		handlep->poll_timeout = 0;
	```

	这个语句表示在非阻塞模式下，轮询请求将阻塞调用，但不会阻塞当前套接字。handlep->poll_timeout将设置为0，意味着它将无限期地等待数据包，直到它接收到TPACKET_V3数据包。


```cpp
#endif
			handlep->poll_timeout = -1;	/* block forever */
	} else if (handlep->timeout > 0) {
#ifdef HAVE_TPACKET3
		/*
		 * For TPACKET_V3, the timeout is handled by the kernel,
		 * so block forever; that way, we don't get extra timeouts.
		 * Don't do that if we have a broken TPACKET_V3, though.
		 */
		if (handlep->tp_version == TPACKET_V3 && !broken_tpacket_v3)
			handlep->poll_timeout = -1;	/* block forever, let TPACKET_V3 wake us up */
		else
#endif
			handlep->poll_timeout = handlep->timeout;	/* block for that amount of time */
	} else {
		/*
		 * Non-blocking mode; we call poll() to pick up error
		 * indications, but we don't want it to wait for
		 * anything.
		 */
		handlep->poll_timeout = 0;
	}
}

```

这段代码是一个用于 Linux 操作系统中的 pcap 工具链的函数，它的作用是设置捕获数据包中的 VLAN 标签的偏移量。

具体来说，函数中首先调用 pcap_breakloop_common 函数来完成一些必要的初始化操作。然后，它定义了一个名为 handlep 的结构体变量，该变量引用了一个名为 handle 的 pcap 句柄，以便后续操作。

接下来，代码创建一个名为 value 的 64 字节整型变量，并将其初始化为 1。然后，代码通过 handlep->poll_breakloop_fd 变量的一个指向文件描述文件的指针来调用另外一個函数，并将其值（即 value）写入到文件中。这个函数的实现略过不提。

总的来说，这段代码的主要目的是设置 VLAN 标签的偏移量，以便正确地应用 VLAN 标签。如果 VLAN 标签的偏移量设置不正确，可能会导致数据包无法正确处理，从而影响网络的正常运行。


```cpp
static void pcap_breakloop_linux(pcap_t *handle)
{
	pcap_breakloop_common(handle);
	struct pcap_linux *handlep = handle->priv;

	uint64_t value = 1;
	/* XXX - what if this fails? */
	if (handlep->poll_breakloop_fd != -1)
		(void)write(handlep->poll_breakloop_fd, &value, sizeof(value));
}

/*
 * Set the offset at which to insert VLAN tags.
 * That should be the offset of the type field.
 */
```

这段代码是一个用于设置VLAN偏移量的函数，它接受一个链接类型为DLT_EN10MB的pcap结构体指针作为参数，然后对传入的handle对象进行处理。

在这些处理中，首先根据传入的链接类型确定vlan偏移量，然后将其设置为相应的值。对于DLT_LINUX_SLL，vlan偏移量是在DLT_LINUX_SLL头部的最后2个字节内设置，对于DLT_EN10MB和DLT_LINUX，vlan偏移量是在MAC地址和destination MAC之间偏移2个字节。如果传入的链接类型无法匹配，函数将返回-1，表示无法处理。


```cpp
static void
set_vlan_offset(pcap_t *handle)
{
	struct pcap_linux *handlep = handle->priv;

	switch (handle->linktype) {

	case DLT_EN10MB:
		/*
		 * The type field is after the destination and source
		 * MAC address.
		 */
		handlep->vlan_offset = 2 * ETH_ALEN;
		break;

	case DLT_LINUX_SLL:
		/*
		 * The type field is in the last 2 bytes of the
		 * DLT_LINUX_SLL header.
		 */
		handlep->vlan_offset = SLL_HDR_LEN - 2;
		break;

	default:
		handlep->vlan_offset = -1; /* unknown */
		break;
	}
}

```

	case TPACKET_V1:
		handle->read_op = pcap_read_linux_mmap_v1;
		break;

	case NO_ROOT:
		handle->root_graph_op = pcap_root_graph_no_bootstrap;
		break;

	case ROOT_GRAPH_WITH_DEBUG:
		handle->root_graph_op = pcap_root_graph_with_debug;
		break;

	case ROOT_GRAPH:
		handle->root_graph_op = pcap_root_graph;
		break;

	case MAX_TRACE:
		handle->max_trace_len = MAX_TRACE_LENGTH;
		break;

	case MIN_TRACE:
		handle->min_trace_len = MIN_TRACE_LENGTH;
		break;
	}

	handle->injector_freq = 1000;
	handle->pcap_ Checksum16_Limit = 0;
	handle->pcap_ Checksum16_Action = clear;
	handle->pcap_ Checksum32_Limit = 0;
	handle->pcap_ Checksum32_Action = clear;

	/*
	 * Perform anmmated write
	 * operations to make the
	 * data we Map all these f端，非
	 * directly accessible to the user.
	 */
	if (handle->user_mmap == -1) {
		handle->err_msg = "无法初始化用户内存映射表。";
		goto fail;
	}

	jmp_exc();

	fail:
		ppcap_error_print(ppcap_ctx, "ppcap_core_setup_error");
		fprintf(stderr, "成功是： %d\n", status);
		PCA_print_error(handlep);
		clear_ppcap_ctx(handlep);
		exit(1);
		break;
}


```cpp
/*
 *  Get a handle for a live capture from the given device. You can
 *  pass NULL as device to get all packages (without link level
 *  information of course). If you pass 1 as promisc the interface
 *  will be set to promiscuous mode (XXX: I think this usage should
 *  be deprecated and functions be added to select that later allow
 *  modification of that values -- Torsten).
 */
static int
pcap_activate_linux(pcap_t *handle)
{
	struct pcap_linux *handlep = handle->priv;
	const char	*device;
	int		is_any_device;
	struct ifreq	ifr;
	int		status = 0;
	int		status2 = 0;
	int		ret;

	device = handle->opt.device;

	/*
	 * Make sure the name we were handed will fit into the ioctls we
	 * might perform on the device; if not, return a "No such device"
	 * indication, as the Linux kernel shouldn't support creating
	 * a device whose name won't fit into those ioctls.
	 *
	 * "Will fit" means "will fit, complete with a null terminator",
	 * so if the length, which does *not* include the null terminator,
	 * is greater than *or equal to* the size of the field into which
	 * we'll be copying it, that won't fit.
	 */
	if (strlen(device) >= sizeof(ifr.ifr_name)) {
		/*
		 * There's nothing more to say, so clear the error
		 * message.
		 */
		handle->errbuf[0] = '\0';
		status = PCAP_ERROR_NO_SUCH_DEVICE;
		goto fail;
	}

	/*
	 * Turn a negative snapshot value (invalid), a snapshot value of
	 * 0 (unspecified), or a value bigger than the normal maximum
	 * value, into the maximum allowed value.
	 *
	 * If some application really *needs* a bigger snapshot
	 * length, we should just increase MAXIMUM_SNAPLEN.
	 */
	if (handle->snapshot <= 0 || handle->snapshot > MAXIMUM_SNAPLEN)
		handle->snapshot = MAXIMUM_SNAPLEN;

	handlep->device	= strdup(device);
	if (handlep->device == NULL) {
		pcap_fmt_errmsg_for_errno(handle->errbuf, PCAP_ERRBUF_SIZE,
		    errno, "strdup");
		status = PCAP_ERROR;
		goto fail;
	}

	/*
	 * The "any" device is a special device which causes us not
	 * to bind to a particular device and thus to look at all
	 * devices.
	 */
	is_any_device = (strcmp(device, "any") == 0);
	if (is_any_device) {
		if (handle->opt.promisc) {
			handle->opt.promisc = 0;
			/* Just a warning. */
			snprintf(handle->errbuf, PCAP_ERRBUF_SIZE,
			    "Promiscuous mode not supported on the \"any\" device");
			status = PCAP_WARNING_PROMISC_NOTSUP;
		}
	}

	/* copy timeout value */
	handlep->timeout = handle->opt.timeout;

	/*
	 * If we're in promiscuous mode, then we probably want
	 * to see when the interface drops packets too, so get an
	 * initial count from
	 * /sys/class/net/{if_name}/statistics/rx_{missed,fifo}_errors
	 */
	if (handle->opt.promisc)
		handlep->sysfs_dropped = linux_if_drops(handlep->device);

	/*
	 * If the "any" device is specified, try to open a SOCK_DGRAM.
	 * Otherwise, open a SOCK_RAW.
	 */
	ret = setup_socket(handle, is_any_device);
	if (ret < 0) {
		/*
		 * Fatal error; the return value is the error code,
		 * and handle->errbuf has been set to an appropriate
		 * error message.
		 */
		status = ret;
		goto fail;
	}
	/*
	 * Success.
	 * Try to set up memory-mapped access.
	 */
	ret = setup_mmapped(handle, &status);
	if (ret == -1) {
		/*
		 * We failed to set up to use it, or the
		 * kernel supports it, but we failed to
		 * enable it.  status has been set to the
		 * error status to return and, if it's
		 * PCAP_ERROR, handle->errbuf contains
		 * the error message.
		 */
		goto fail;
	}

	/*
	 * We succeeded.  status has been set to the status to return,
	 * which might be 0, or might be a PCAP_WARNING_ value.
	 */
	/*
	 * Now that we have activated the mmap ring, we can
	 * set the correct protocol.
	 */
	if ((status2 = iface_bind(handle->fd, handlep->ifindex,
	    handle->errbuf, pcap_protocol(handle))) != 0) {
		status = status2;
		goto fail;
	}

	handle->inject_op = pcap_inject_linux;
	handle->setfilter_op = pcap_setfilter_linux;
	handle->setdirection_op = pcap_setdirection_linux;
	handle->set_datalink_op = pcap_set_datalink_linux;
	handle->setnonblock_op = pcap_setnonblock_linux;
	handle->getnonblock_op = pcap_getnonblock_linux;
	handle->cleanup_op = pcap_cleanup_linux;
	handle->stats_op = pcap_stats_linux;
	handle->breakloop_op = pcap_breakloop_linux;

	switch (handlep->tp_version) {

	case TPACKET_V2:
		handle->read_op = pcap_read_linux_mmap_v2;
		break;
```

这段代码是一个C语言程序，它使用了Linux系统中的pcap库来处理网络数据包。程序主要实现了以下功能：

1. 如果系统支持TPACKET_V3头，则使用pcap_read_linux_mmap_v3函数读取数据包。
2. 如果系统不支持TPACKET_V3头，则设置handle->read_op为pcap_read_linear_mmap，然后使用pcap_read函数读取数据包。
3. 设置handle->oneshot_callback为pcap_oneshot_linux函数，用于设置零缝窗口。
4. 设置handle->selectable_fd为当前套接字的文件描述符，以便能够在此文件 descriptor上发送数据包。
5. 如果出现错误，使用pcap_cleanup_linux函数清除pcap实例，然后返回之前设置的错误码。

总的来说，这段代码实现了处理网络数据包的基本功能，可以根据不同头信息选择不同的数据包读取方式，并支持零缝窗口设置。


```cpp
#ifdef HAVE_TPACKET3
	case TPACKET_V3:
		handle->read_op = pcap_read_linux_mmap_v3;
		break;
#endif
	}
	handle->oneshot_callback = pcap_oneshot_linux;
	handle->selectable_fd = handle->fd;

	return status;

fail:
	pcap_cleanup_linux(handle);
	return status;
}

```

这段代码定义了一个名为 `pcap_set_datalink_linux()` 的函数，属于 `pcap_t` 类的成员函数。函数接受两个参数，一个是 `pcap_t` 类型的指针变量 `handle`，另一个是整数类型的变量 `dlt`。函数的作用是设置数据链路层（datalink）的链路类型为 `dlt`，然后更新插入 VLAN 标签的偏移量。

具体来说，函数内部首先调用 `set_vlan_offset()` 函数，该函数会根据新的链路类型更新 `handle` 指向的链路中的 VLAN 标签的偏移量。然后，函数返回一个整数，表示函数操作成功。


```cpp
static int
pcap_set_datalink_linux(pcap_t *handle, int dlt)
{
	handle->linktype = dlt;

	/*
	 * Update the offset at which to insert VLAN tags for the
	 * new link-layer type.
	 */
	set_vlan_offset(handle);

	return 0;
}

/*
 * linux_check_direction()
 *
 * Do checks based on packet direction.
 */
```

This function is used to check the direction of a packet流进或流出。 It takes a pcap handle and a struct sockaddr\_ll containing the source and destination IPv4 addresses and port numbers.

The function first checks if the packet is going out or in. If it's going out, it checks if it's from the local loopback device (index 0). If it's not from the loopback device, the function returns 0.

If the packet is going out, the function checks if it's an outgoing packet. If it is, the function checks if it's a Linux Can (ARPHRD\_CAN) frame. If it's not, the function checks if the user wants incoming packets. If the user wants incoming packets, the function returns 0. If the user wants outgoing packets but the packet is not explicitly set to be an outgoing packet, the function returns 0.

If the packet is going in, the function checks if it's an incoming packet. If the user wants outgoing packets, the function returns 0. If the user wants incoming packets, the function returns 0.

Note that if the packet is from the loopback device and the user doesn't want incoming packets, the function returns 0. If the user wants incoming packets but the packet is not explicitly set to be an incoming packet, the function returns 0.


```cpp
static inline int
linux_check_direction(const pcap_t *handle, const struct sockaddr_ll *sll)
{
	struct pcap_linux	*handlep = handle->priv;

	if (sll->sll_pkttype == PACKET_OUTGOING) {
		/*
		 * Outgoing packet.
		 * If this is from the loopback device, reject it;
		 * we'll see the packet as an incoming packet as well,
		 * and we don't want to see it twice.
		 */
		if (sll->sll_ifindex == handlep->lo_ifindex)
			return 0;

		/*
		 * If this is an outgoing CAN or CAN FD frame, and
		 * the user doesn't only want outgoing packets,
		 * reject it; CAN devices and drivers, and the CAN
		 * stack, always arrange to loop back transmitted
		 * packets, so they also appear as incoming packets.
		 * We don't want duplicate packets, and we can't
		 * easily distinguish packets looped back by the CAN
		 * layer than those received by the CAN layer, so we
		 * eliminate this packet instead.
		 *
		 * We check whether this is a CAN or CAN FD frame
		 * by checking whether the device's hardware type
		 * is ARPHRD_CAN.
		 */
		if (sll->sll_hatype == ARPHRD_CAN &&
		     handle->direction != PCAP_D_OUT)
			return 0;

		/*
		 * If the user only wants incoming packets, reject it.
		 */
		if (handle->direction == PCAP_D_IN)
			return 0;
	} else {
		/*
		 * Incoming packet.
		 * If the user only wants outgoing packets, reject it.
		 */
		if (handle->direction == PCAP_D_OUT)
			return 0;
	}
	return 1;
}

```

这段代码的作用是检查设备绑定的套接字是否仍然存在。它通过询问套接字绑定到的套接字地址，并检查 ifindex 是否为 -1。如果是，那么表示设备已经消失，函数返回 0。否则，函数返回 1。

具体来说，代码首先检查 handlep->ifindex 是否为 -1。如果是，说明套接字尚未绑定，函数返回 1。然后代码尝试通过调用 getsockname 函数获取套接字地址，并检查返回值是否为 -1。如果是，说明获取失败，函数错误地报告错误并返回 -1。如果获取成功，代码将检查 ifaceindex 是否为 -1。如果是，表示设备已经消失，函数返回 0。否则，函数返回 1。


```cpp
/*
 * Check whether the device to which the pcap_t is bound still exists.
 * We do so by asking what address the socket is bound to, and checking
 * whether the ifindex in the address is -1, meaning "that device is gone",
 * or some other value, meaning "that device still exists".
 */
static int
device_still_exists(pcap_t *handle)
{
	struct pcap_linux *handlep = handle->priv;
	struct sockaddr_ll addr;
	socklen_t addr_len;

	/*
	 * If handlep->ifindex is -1, the socket isn't bound, meaning
	 * we're capturing on the "any" device; that device never
	 * disappears.  (It should also never be configured down, so
	 * we shouldn't even get here, but let's make sure.)
	 */
	if (handlep->ifindex == -1)
		return (1);	/* it's still here */

	/*
	 * OK, now try to get the address for the socket.
	 */
	addr_len = sizeof (addr);
	if (getsockname(handle->fd, (struct sockaddr *) &addr, &addr_len) == -1) {
		/*
		 * Error - report an error and return -1.
		 */
		pcap_fmt_errmsg_for_errno(handle->errbuf, PCAP_ERRBUF_SIZE,
		    errno, "getsockname failed");
		return (-1);
	}
	if (addr.sll_ifindex == -1) {
		/*
		 * This means the device went away.
		 */
		return (0);
	}

	/*
	 * The device presumably just went down.
	 */
	return (1);
}

```

该函数的作用是向名为"any"的设备发送数据。首先，它将一个指向"any"设备的pcap句柄和发送数据的缓冲区作为参数传入。然后，它尝试使用`send`函数将数据发送到设备，并返回一个integer类型的错误码。如果发送成功，则返回send的返回值；如果发送失败，则返回一个包含错误信息的错误码。


```cpp
static int
pcap_inject_linux(pcap_t *handle, const void *buf, int size)
{
	struct pcap_linux *handlep = handle->priv;
	int ret;

	if (handlep->ifindex == -1) {
		/*
		 * We don't support sending on the "any" device.
		 */
		pcap_strlcpy(handle->errbuf,
		    "Sending packets isn't supported on the \"any\" device",
		    PCAP_ERRBUF_SIZE);
		return (-1);
	}

	if (handlep->cooked) {
		/*
		 * We don't support sending on cooked-mode sockets.
		 *
		 * XXX - how do you send on a bound cooked-mode
		 * socket?
		 * Is a "sendto()" required there?
		 */
		pcap_strlcpy(handle->errbuf,
		    "Sending packets isn't supported in cooked mode",
		    PCAP_ERRBUF_SIZE);
		return (-1);
	}

	ret = (int)send(handle->fd, buf, size, 0);
	if (ret == -1) {
		pcap_fmt_errmsg_for_errno(handle->errbuf, PCAP_ERRBUF_SIZE,
		    errno, "send");
		return (-1);
	}
	return (ret);
}

```

这段代码的作用是获取给定数据包捕获器的统计信息，并将其存储在stats结构中。它适用于Linux系统，并在其中的socket使用TPACKET_V2协议。

代码首先通过指向结构体内部指针handlep来获取数据包捕获器句柄。接下来，它定义了一个名为handlep->stats的整型变量，用于存储统计信息。

接下来，代码使用handle->priv成员指针来获取线程句柄所关联的Linux数据包捕获器。然后，它遍历并跳过TPACKET_V2结构体中的某些成员，因为对于使用TPACKET_V2的socket，这些成员可能不会被填充，并且对于使用handle->priv指针所关联的socket，只需要知道数据包的大小。

接下来，代码定义了一个名为kstats的结构体，其中包含TPACKET_V3结构体，用于获取统计信息。最后，代码将kstats中的统计信息复制到stats结构中，并返回统计信息计数器中的值。


```cpp
/*
 *  Get the statistics for the given packet capture handle.
 */
static int
pcap_stats_linux(pcap_t *handle, struct pcap_stat *stats)
{
	struct pcap_linux *handlep = handle->priv;
#ifdef HAVE_TPACKET3
	/*
	 * For sockets using TPACKET_V2, the extra stuff at the end
	 * of a struct tpacket_stats_v3 will not be filled in, and
	 * we don't look at it so this is OK even for those sockets.
	 * In addition, the PF_PACKET socket code in the kernel only
	 * uses the length parameter to compute how much data to
	 * copy out and to indicate how much data was copied out, so
	 * it's OK to base it on the size of a struct tpacket_stats.
	 *
	 * XXX - it's probably OK, in fact, to just use a
	 * struct tpacket_stats for V3 sockets, as we don't
	 * care about the tp_freeze_q_cnt stat.
	 */
	struct tpacket_stats_v3 kstats;
```

This is a function that returns the statistics of packets received from a packet filter over a network socket. The function takes two arguments: the packet filter handle and the return buffer for the statistics.

The function first initializes the statistics for received packets. If the handle is not a valid packet filter, the function returns an error.

The function then gets the statistics for the packet filter. The statistics include the number of packets received and the number of packets dropped because there wasn't enough free space in the packet filter.

The statistics are then returned in an integer value.

If the function can't return the statistics because the handle is not a valid packet filter, the function returns the error status of the调用 to getsockopt.


```cpp
#else /* HAVE_TPACKET3 */
	struct tpacket_stats kstats;
#endif /* HAVE_TPACKET3 */
	socklen_t len = sizeof (struct tpacket_stats);

	long long if_dropped = 0;

	/*
	 * To fill in ps_ifdrop, we parse
	 * /sys/class/net/{if_name}/statistics/rx_{missed,fifo}_errors
	 * for the numbers
	 */
	if (handle->opt.promisc)
	{
		/*
		 * XXX - is there any reason to do this by remembering
		 * the last counts value, subtracting it from the
		 * current counts value, and adding that to stat.ps_ifdrop,
		 * maintaining stat.ps_ifdrop as a count, rather than just
		 * saving the *initial* counts value and setting
		 * stat.ps_ifdrop to the difference between the current
		 * value and the initial value?
		 *
		 * One reason might be to handle the count wrapping
		 * around, on platforms where the count is 32 bits
		 * and where you might get more than 2^32 dropped
		 * packets; is there any other reason?
		 *
		 * (We maintain the count as a long long int so that,
		 * if the kernel maintains the counts as 64-bit even
		 * on 32-bit platforms, we can handle the real count.
		 *
		 * Unfortunately, we can't report 64-bit counts; we
		 * need a better API for reporting statistics, such as
		 * one that reports them in a style similar to the
		 * pcapng Interface Statistics Block, so that 1) the
		 * counts are 64-bit, 2) it's easier to add new statistics
		 * without breaking the ABI, and 3) it's easier to
		 * indicate to a caller that wants one particular
		 * statistic that it's not available by just not supplying
		 * it.)
		 */
		if_dropped = handlep->sysfs_dropped;
		handlep->sysfs_dropped = linux_if_drops(handlep->device);
		handlep->stat.ps_ifdrop += (u_int)(handlep->sysfs_dropped - if_dropped);
	}

	/*
	 * Try to get the packet counts from the kernel.
	 */
	if (getsockopt(handle->fd, SOL_PACKET, PACKET_STATISTICS,
			&kstats, &len) > -1) {
		/*
		 * "ps_recv" counts only packets that *passed* the
		 * filter, not packets that didn't pass the filter.
		 * This includes packets later dropped because we
		 * ran out of buffer space.
		 *
		 * "ps_drop" counts packets dropped because we ran
		 * out of buffer space.  It doesn't count packets
		 * dropped by the interface driver.  It counts only
		 * packets that passed the filter.
		 *
		 * See above for ps_ifdrop.
		 *
		 * Both statistics include packets not yet read from
		 * the kernel by libpcap, and thus not yet seen by
		 * the application.
		 *
		 * In "linux/net/packet/af_packet.c", at least in 2.6.27
		 * through 5.6 kernels, "tp_packets" is incremented for
		 * every packet that passes the packet filter *and* is
		 * successfully copied to the ring buffer; "tp_drops" is
		 * incremented for every packet dropped because there's
		 * not enough free space in the ring buffer.
		 *
		 * When the statistics are returned for a PACKET_STATISTICS
		 * "getsockopt()" call, "tp_drops" is added to "tp_packets",
		 * so that "tp_packets" counts all packets handed to
		 * the PF_PACKET socket, including packets dropped because
		 * there wasn't room on the socket buffer - but not
		 * including packets that didn't pass the filter.
		 *
		 * In the BSD BPF, the count of received packets is
		 * incremented for every packet handed to BPF, regardless
		 * of whether it passed the filter.
		 *
		 * We can't make "pcap_stats()" work the same on both
		 * platforms, but the best approximation is to return
		 * "tp_packets" as the count of packets and "tp_drops"
		 * as the count of drops.
		 *
		 * Keep a running total because each call to
		 *    getsockopt(handle->fd, SOL_PACKET, PACKET_STATISTICS, ....
		 * resets the counters to zero.
		 */
		handlep->stat.ps_recv += kstats.tp_packets;
		handlep->stat.ps_drop += kstats.tp_drops;
		*stats = handlep->stat;
		return 0;
	}

	pcap_fmt_errmsg_for_errno(handle->errbuf, PCAP_ERRBUF_SIZE, errno,
	    "failed to get statistics from socket");
	return -1;
}

```

这段代码定义了一个名为 `any_descr` 的常量字符数组，用于描述 "any" 设备。

接着定义了一个名为 `can_be_bound` 的函数，该函数接收一个表示网络接口名称的参数，并返回一个指示符，表示是否可以将 `PF_PACKET` 类型的套接字绑定到该接口上。函数的实现比较简单，直接返回一个布尔值，表示设备的名称是否以 "any" 开头。

最后，没有做其他事情，直接结束。


```cpp
/*
 * Description string for the "any" device.
 */
static const char any_descr[] = "Pseudo-device that captures on all interfaces";

/*
 * A PF_PACKET socket can be bound to any network interface.
 */
static int
can_be_bound(const char *name _U_)
{
	return (1);
}

/*
 * Get a socket to use with various interface ioctls.
 */
```

It looks like you are trying to add a new line to the previous code, but you are not providing any valid code to add.

If you are trying to add a newline to the following code, I would suggest that you do not do so. This code is already very long, and adding another line would only cause the remaining lines to be indented, which would make the code more difficult to read.

If you have a specific question or request, please provide the code that you would like me to review, and I will do my best to assist you.


```cpp
static int
get_if_ioctl_socket(void)
{
	int fd;

	/*
	 * This is a bit ugly.
	 *
	 * There isn't a socket type that's guaranteed to work.
	 *
	 * AF_NETLINK will work *if* you have Netlink configured into the
	 * kernel (can it be configured out if you have any networking
	 * support at all?) *and* if you're running a sufficiently recent
	 * kernel, but not all the kernels we support are sufficiently
	 * recent - that feature was introduced in Linux 4.6.
	 *
	 * AF_UNIX will work *if* you have UNIX-domain sockets configured
	 * into the kernel and *if* you're not on a system that doesn't
	 * allow them - some SELinux systems don't allow you create them.
	 * Most systems probably have them configured in, but not all systems
	 * have them configured in and allow them to be created.
	 *
	 * AF_INET will work *if* you have IPv4 configured into the kernel,
	 * but, apparently, some systems have network adapters but have
	 * kernels without IPv4 support.
	 *
	 * AF_INET6 will work *if* you have IPv6 configured into the
	 * kernel, but if you don't have AF_INET, you might not have
	 * AF_INET6, either (that is, independently on its own grounds).
	 *
	 * AF_PACKET would work, except that some of these calls should
	 * work even if you *don't* have capture permission (you should be
	 * able to enumerate interfaces and get information about them
	 * without capture permission; you shouldn't get a failure until
	 * you try pcap_activate()).  (If you don't allow programs to
	 * get as much information as possible about interfaces if you
	 * don't have permission to capture, you run the risk of users
	 * asking "why isn't it showing XXX" - or, worse, if you don't
	 * show interfaces *at all* if you don't have permission to
	 * capture on them, "why do no interfaces show up?" - when the
	 * real problem is a permissions problem.  Error reports of that
	 * type require a lot more back-and-forth to debug, as evidenced
	 * by many Wireshark bugs/mailing list questions/Q&A questions.)
	 *
	 * So:
	 *
	 * we first try an AF_NETLINK socket, where "try" includes
	 * "try to do a device ioctl on it", as, in the future, once
	 * pre-4.6 kernels are sufficiently rare, that will probably
	 * be the mechanism most likely to work;
	 *
	 * if that fails, we try an AF_UNIX socket, as that's less
	 * likely to be configured out on a networking-capable system
	 * than is IP;
	 *
	 * if that fails, we try an AF_INET6 socket;
	 *
	 * if that fails, we try an AF_INET socket.
	 */
	fd = socket(AF_NETLINK, SOCK_RAW, NETLINK_GENERIC);
	if (fd != -1) {
		/*
		 * OK, let's make sure we can do an SIOCGIFNAME
		 * ioctl.
		 */
		struct ifreq ifr;

		memset(&ifr, 0, sizeof(ifr));
		if (ioctl(fd, SIOCGIFNAME, &ifr) == 0 ||
		    errno != EOPNOTSUPP) {
			/*
			 * It succeeded, or failed for some reason
			 * other than "netlink sockets don't support
			 * device ioctls".  Go with the AF_NETLINK
			 * socket.
			 */
			return (fd);
		}

		/*
		 * OK, that didn't work, so it's as bad as "netlink
		 * sockets aren't available".  Close the socket and
		 * drive on.
		 */
		close(fd);
	}

	/*
	 * Now try an AF_UNIX socket.
	 */
	fd = socket(AF_UNIX, SOCK_RAW, 0);
	if (fd != -1) {
		/*
		 * OK, we got it!
		 */
		return (fd);
	}

	/*
	 * Now try an AF_INET6 socket.
	 */
	fd = socket(AF_INET6, SOCK_DGRAM, 0);
	if (fd != -1) {
		return (fd);
	}

	/*
	 * Now try an AF_INET socket.
	 *
	 * XXX - if that fails, is there anything else we should try?
	 * AF_CAN, for embedded systems in vehicles, in case they're
	 * built without Internet protocol support?  Any other socket
	 * types popular in non-Internet embedded systems?
	 */
	return (socket(AF_INET, SOCK_DGRAM, 0));
}

```

It looks like the code you provided is trying to retrieve the device type from a network class device. Specifically, it is trying to read the device type from the file "/sys/class/net/*name*/type".

If the code is able to read the device type from the file, it will return the value "arptype" which can then be used to determine the type of network class device.

It is important to note that the device type returned by the code may not always match the actual device type. Additionally, the code may not handle cases where the device type cannot be found in the file or if the file is not readable.


```cpp
/*
 * Get additional flags for a device, using SIOCGIFMEDIA.
 */
static int
get_if_flags(const char *name, bpf_u_int32 *flags, char *errbuf)
{
	int sock;
	FILE *fh;
	unsigned int arptype;
	struct ifreq ifr;
	struct ethtool_value info;

	if (*flags & PCAP_IF_LOOPBACK) {
		/*
		 * Loopback devices aren't wireless, and "connected"/
		 * "disconnected" doesn't apply to them.
		 */
		*flags |= PCAP_IF_CONNECTION_STATUS_NOT_APPLICABLE;
		return 0;
	}

	sock = get_if_ioctl_socket();
	if (sock == -1) {
		pcap_fmt_errmsg_for_errno(errbuf, PCAP_ERRBUF_SIZE, errno,
		    "Can't create socket to get ethtool information for %s",
		    name);
		return -1;
	}

	/*
	 * OK, what type of network is this?
	 * In particular, is it wired or wireless?
	 */
	if (is_wifi(name)) {
		/*
		 * Wi-Fi, hence wireless.
		 */
		*flags |= PCAP_IF_WIRELESS;
	} else {
		/*
		 * OK, what does /sys/class/net/{if_name}/type contain?
		 * (We don't use that for Wi-Fi, as it'll report
		 * "Ethernet", i.e. ARPHRD_ETHER, for non-monitor-
		 * mode devices.)
		 */
		char *pathstr;

		if (asprintf(&pathstr, "/sys/class/net/%s/type", name) == -1) {
			snprintf(errbuf, PCAP_ERRBUF_SIZE,
			    "%s: Can't generate path name string for /sys/class/net device",
			    name);
			close(sock);
			return -1;
		}
		fh = fopen(pathstr, "r");
		if (fh != NULL) {
			if (fscanf(fh, "%u", &arptype) == 1) {
				/*
				 * OK, we got an ARPHRD_ type; what is it?
				 */
				switch (arptype) {

				case ARPHRD_LOOPBACK:
					/*
					 * These are types to which
					 * "connected" and "disconnected"
					 * don't apply, so don't bother
					 * asking about it.
					 *
					 * XXX - add other types?
					 */
					close(sock);
					fclose(fh);
					free(pathstr);
					return 0;

				case ARPHRD_IRDA:
				case ARPHRD_IEEE80211:
				case ARPHRD_IEEE80211_PRISM:
				case ARPHRD_IEEE80211_RADIOTAP:
```

这段代码是一个条件编译语句，用于根据特定的标识符（ARPHRD_IEEE802154、ARPHRD_IEEE802154_MONITOR、ARPHRD_6LOWPAN）来执行不同的代码块。

具体来说，当定义了ARPHRD_IEEE802154标识符时，会执行以ARPHRD_IEEE802154开头的代码块；如果定义了ARPHRD_IEEE802154_MONITOR标识符，则会执行以ARPHRD_IEEE802154_MONITOR开头的代码块；如果定义了ARPHRD_6LOWPAN标识符，则会执行以ARPHRD_6LOWPAN开头的代码块。

在代码块内部，会定义一个名为ARPHRD_6LOWPAN的标识符，它的值为0。然后，通过计算flags变量来判断当前所处的无线网络类型，如果ARPHRD_6LOWPAN标识符已经被定义，则设置flags的值为1，从而执行以ARPHRD_6LOWPAN开头的代码块。

最后，代码还会关闭fh文件，释放pathstr指向的内存。


```cpp
#ifdef ARPHRD_IEEE802154
				case ARPHRD_IEEE802154:
#endif
#ifdef ARPHRD_IEEE802154_MONITOR
				case ARPHRD_IEEE802154_MONITOR:
#endif
#ifdef ARPHRD_6LOWPAN
				case ARPHRD_6LOWPAN:
#endif
					/*
					 * Various wireless types.
					 */
					*flags |= PCAP_IF_WIRELESS;
					break;
				}
			}
			fclose(fh);
		}
		free(pathstr);
	}

```

This is a Linux system call function that checks the status of an Ethernet interface using the ethtool command. The function takes two arguments: the network interface ID (a compile-time constant) and a pointer to a structure containing information about the interface (a pointer to the `if_device` structure).

The function first checks if the specified interface is supported by the operating system. If it is not supported, the function returns an error code. If the interface is supported, the function returns the `mean_value` field of the `if_device` structure, which contains the statistical mean of the number of bytes received and sent on the interface.

If the interface goes down, the function returns an error code. If the interface is not being used, the function returns an error code and the save error return value.

If the interface is connected, the function sets the `mean_value` field of the `if_device` structure to the `mean_value` field of the interface's statistics.

The function also handles the case where the interface does not exist. If the user tries to activate the interface, the function returns an error code. If the user tries to get statistics for the interface, the function returns an error code if the interface is not supported.


```cpp
#ifdef ETHTOOL_GLINK
	memset(&ifr, 0, sizeof(ifr));
	pcap_strlcpy(ifr.ifr_name, name, sizeof(ifr.ifr_name));
	info.cmd = ETHTOOL_GLINK;
	/*
	 * XXX - while Valgrind handles SIOCETHTOOL and knows that
	 * the ETHTOOL_GLINK command sets the .data member of the
	 * structure, Memory Sanitizer doesn't yet do so:
	 *
	 *    https://bugs.llvm.org/show_bug.cgi?id=45814
	 *
	 * For now, we zero it out to squelch warnings; if the bug
	 * in question is fixed, we can remove this.
	 */
	info.data = 0;
	ifr.ifr_data = (caddr_t)&info;
	if (ioctl(sock, SIOCETHTOOL, &ifr) == -1) {
		int save_errno = errno;

		switch (save_errno) {

		case EOPNOTSUPP:
		case EINVAL:
			/*
			 * OK, this OS version or driver doesn't support
			 * asking for this information.
			 * XXX - distinguish between "this doesn't
			 * support ethtool at all because it's not
			 * that type of device" vs. "this doesn't
			 * support ethtool even though it's that
			 * type of device", and return "unknown".
			 */
			*flags |= PCAP_IF_CONNECTION_STATUS_NOT_APPLICABLE;
			close(sock);
			return 0;

		case ENODEV:
			/*
			 * OK, no such device.
			 * The user will find that out when they try to
			 * activate the device; just say "OK" and
			 * don't set anything.
			 */
			close(sock);
			return 0;

		default:
			/*
			 * Other error.
			 */
			pcap_fmt_errmsg_for_errno(errbuf, PCAP_ERRBUF_SIZE,
			    save_errno,
			    "%s: SIOCETHTOOL(ETHTOOL_GLINK) ioctl failed",
			    name);
			close(sock);
			return -1;
		}
	}

	/*
	 * Is it connected?
	 */
	if (info.data) {
		/*
		 * It's connected.
		 */
		*flags |= PCAP_IF_CONNECTION_STATUS_CONNECTED;
	} else {
		/*
		 * It's disconnected.
		 */
		*flags |= PCAP_IF_CONNECTION_STATUS_DISCONNECTED;
	}
```

这段代码是一个用于 Linux 系统的 pcap-疑难解答工具的函数。它主要实现了两个功能：

1. 关闭当前套接字（sock）。
2. 返回抽象层（abstract层）接口（device）的个数（count）。

该函数的作用是先获取系统中所有正常工作的网络接口（any device），然后将 "any" 设备加入接口列表。如果执行成功，则返回抽象层接口的数量；如果失败，则返回 -1。


```cpp
#endif

	close(sock);
	return 0;
}

int
pcap_platform_finddevs(pcap_if_list_t *devlistp, char *errbuf)
{
	/*
	 * Get the list of regular interfaces first.
	 */
	if (pcap_findalldevs_interfaces(devlistp, errbuf, can_be_bound,
	    get_if_flags) == -1)
		return (-1);	/* failure */

	/*
	 * Add the "any" device.
	 * As it refers to all network devices, not to any particular
	 * network device, the notion of "connected" vs. "disconnected"
	 * doesn't apply.
	 */
	if (add_dev(devlistp, "any",
	    PCAP_IF_UP|PCAP_IF_RUNNING|PCAP_IF_CONNECTION_STATUS_NOT_APPLICABLE,
	    any_descr, errbuf) == NULL)
		return (-1);

	return (0);
}

```

这段代码定义了一个名为 `pcap_setdirection_linux` 的函数，属于 `pcap_per_type_t` 类的成员函数。

该函数的作用是设置输入/输出设备方向，接收者可以选择IN、OUT或两者。

函数接受一个 `pcap_t` 类型的输入参数 `handle`，一个 `pcap_direction_t` 类型的参数 `d`，然后将 `handle->direction` 设置为参数 `d`，并返回 0。

通过调用该函数，可以将输入/输出设备设置为特定的方向，例如，将输入设备设置为 OUT，将输出设备设置为 IN。


```cpp
/*
 * Set direction flag: Which packets do we accept on a forwarding
 * single device? IN, OUT or both?
 */
static int
pcap_setdirection_linux(pcap_t *handle, pcap_direction_t d)
{
	/*
	 * It's guaranteed, at this point, that d is a valid
	 * direction value.
	 */
	handle->direction = d;
	return 0;
}

```

这段代码是一个名为 `is_wifi` 的函数，它的作用是检查给定的设备是否支持无线网络。它接受一个字符串参数 `device`，然后返回一个整数。

具体来说，代码首先创建一个名为 `pathstr` 的字符数组，然后使用 `asprintf` 函数将 `device` 参数转换为一个路径名，接着在 `/sys/class/net/<device>/wireless` 目录下查找是否存在。如果找到了该目录，就表示存在一个无线网络接口，函数返回 1；如果查找失败，或者找到的文件不是有效的文件，就返回 0。

如果找到了无线网络接口，代码会尝试使用 `stat` 函数获取该接口的 `stat` 结构体，并从其中获取文件的描述符。然后，代码会释放之前创建的 `pathstr` 字符数组，并返回 1，表示成功检查了无线网络接口。


```cpp
static int
is_wifi(const char *device)
{
	char *pathstr;
	struct stat statb;

	/*
	 * See if there's a sysfs wireless directory for it.
	 * If so, it's a wireless interface.
	 */
	if (asprintf(&pathstr, "/sys/class/net/%s/wireless", device) == -1) {
		/*
		 * Just give up here.
		 */
		return 0;
	}
	if (stat(pathstr, &statb) == 0) {
		free(pathstr);
		return 1;
	}
	free(pathstr);

	return 0;
}

```

这段代码的主要作用是设置系统链路类型（handle->linktype）和链路头部长度（handle->offset），以便在以ARP包形式捕获数据包时，使数据包在4字节边界上对齐，并确保在捕获过程中正确地链接到链路层头部。

首先，它检查cooked_ok是否为非零值。如果是，它将尝试使用DLT_LINUX_SLL设置系统链路类型为-1，以便在 cooked模式下捕获数据包；否则，它将无法使用cooked模式，因此必须选择一些可以在raw模式下工作的链路类型，或者导致错误。

如果无法正确设置系统链路类型，代码将设置handle->linktype为-1，handle->offset为0，以便在适当的链路层头部长度下对齐数据包。


```cpp
/*
 *  Linux uses the ARP hardware type to identify the type of an
 *  interface. pcap uses the DLT_xxx constants for this. This
 *  function takes a pointer to a "pcap_t", and an ARPHRD_xxx
 *  constant, as arguments, and sets "handle->linktype" to the
 *  appropriate DLT_XXX constant and sets "handle->offset" to
 *  the appropriate value (to make "handle->offset" plus link-layer
 *  header length be a multiple of 4, so that the link-layer payload
 *  will be aligned on a 4-byte boundary when capturing packets).
 *  (If the offset isn't set here, it'll be 0; add code as appropriate
 *  for cases where it shouldn't be 0.)
 *
 *  If "cooked_ok" is non-zero, we can use DLT_LINUX_SLL and capture
 *  in cooked mode; otherwise, we can't use cooked mode, so we have
 *  to pick some type that works in raw mode, or fail.
 *
 *  Sets the link type to -1 if unable to map the type.
 */
```

It looks like you are trying to configure a network device. The command you provided, `iface_dsa_get_proto_info()`


```cpp
static void map_arphrd_to_dlt(pcap_t *handle, int arptype,
			      const char *device, int cooked_ok)
{
	static const char cdma_rmnet[] = "cdma_rmnet";

	switch (arptype) {

	case ARPHRD_ETHER:
		/*
		 * For various annoying reasons having to do with DHCP
		 * software, some versions of Android give the mobile-
		 * phone-network interface an ARPHRD_ value of
		 * ARPHRD_ETHER, even though the packets supplied by
		 * that interface have no link-layer header, and begin
		 * with an IP header, so that the ARPHRD_ value should
		 * be ARPHRD_NONE.
		 *
		 * Detect those devices by checking the device name, and
		 * use DLT_RAW for them.
		 */
		if (strncmp(device, cdma_rmnet, sizeof cdma_rmnet - 1) == 0) {
			handle->linktype = DLT_RAW;
			return;
		}

		/*
		 * Is this a real Ethernet device?  If so, give it a
		 * link-layer-type list with DLT_EN10MB and DLT_DOCSIS, so
		 * that an application can let you choose it, in case you're
		 * capturing DOCSIS traffic that a Cisco Cable Modem
		 * Termination System is putting out onto an Ethernet (it
		 * doesn't put an Ethernet header onto the wire, it puts raw
		 * DOCSIS frames out on the wire inside the low-level
		 * Ethernet framing).
		 *
		 * XXX - are there any other sorts of "fake Ethernet" that
		 * have ARPHRD_ETHER but that shouldn't offer DLT_DOCSIS as
		 * a Cisco CMTS won't put traffic onto it or get traffic
		 * bridged onto it?  ISDN is handled in "setup_socket()",
		 * as we fall back on cooked mode there, and we use
		 * is_wifi() to check for 802.11 devices; are there any
		 * others?
		 */
		if (!is_wifi(device)) {
			int ret;

			/*
			 * This is not a Wi-Fi device but it could be
			 * a DSA master/management network device.
			 */
			ret = iface_dsa_get_proto_info(device, handle);
			if (ret < 0)
				return;

			if (ret == 1) {
				/*
				 * This is a DSA master/management network
				 * device linktype is already set by
				 * iface_dsa_get_proto_info() set an
				 * appropriate offset here.
				 */
				handle->offset = 2;
				break;
			}

			/*
			 * It's not a Wi-Fi device; offer DOCSIS.
			 */
			handle->dlt_list = (u_int *) malloc(sizeof(u_int) * 2);
			/*
			 * If that fails, just leave the list empty.
			 */
			if (handle->dlt_list != NULL) {
				handle->dlt_list[0] = DLT_EN10MB;
				handle->dlt_list[1] = DLT_DOCSIS;
				handle->dlt_count = 2;
			}
		}
		/* FALLTHROUGH */

	case ARPHRD_METRICOM:
	case ARPHRD_LOOPBACK:
		handle->linktype = DLT_EN10MB;
		handle->offset = 2;
		break;

	case ARPHRD_EETHER:
		handle->linktype = DLT_EN3MB;
		break;

	case ARPHRD_AX25:
		handle->linktype = DLT_AX25_KISS;
		break;

	case ARPHRD_PRONET:
		handle->linktype = DLT_PRONET;
		break;

	case ARPHRD_CHAOS:
		handle->linktype = DLT_CHAOS;
		break;
```

这段代码是一个C语言代码片段，它定义了一个名为“ARPHRD_CAN”的宏，其值为280。接着，它定义了两个名为“ARPHRD_IEEE802_TR”和“ARPHRD_IEEE802”的宏，分别对应于IEEE802.3和IEEE802.3u标准。

接下来的代码是一个C语言注释，它告诉编译器不要输出这些宏的定义。

最后，代码定义了一个名为“handle”的结构体，它包含一个名为“linktype”的整型成员和一个名为“offset”的整型成员。

整个代码片段的作用是定义了一些常量和宏，用于设置一个网络适配器的链接类型和偏移量。这些常量和宏将被链接到指定的目标文件中，最终在应用程序中使用。


```cpp
#ifndef ARPHRD_CAN
#define ARPHRD_CAN 280
#endif
	case ARPHRD_CAN:
		handle->linktype = DLT_CAN_SOCKETCAN;
		break;

#ifndef ARPHRD_IEEE802_TR
#define ARPHRD_IEEE802_TR 800	/* From Linux 2.4 */
#endif
	case ARPHRD_IEEE802_TR:
	case ARPHRD_IEEE802:
		handle->linktype = DLT_IEEE802;
		handle->offset = 2;
		break;

	case ARPHRD_ARCNET:
		handle->linktype = DLT_ARCNET_LINUX;
		break;

```

This is a description of the LLC (Link-layer Privacy) header and how it is related to the ARPHRD_ type of header in IPv6 packets. The LLC header is also described in terms of its relationship to the SNAP header.

The LLC header is a part of the IPv6 header that is used to specify the link-layer address of a packet. It is added to the packet after the payloads of the packet have been processed.

The ARPHRD_ type is a type of header that is used in IPv6 packets to specify the payloads of the packet. It is used to indicate that the payloads of the packet are not part of the header, but rather contain other types of data that should be processed at the link layer.

VC Based Multiplexing is a method of multiplexing in which different virtual circuits carry different network layer protocols. This is different from the way in which the LLC header is used in IPv6 packets.

There are different LLC header types specified in the RFC 1483, which include the LLC type header, the PFFX-MHL header, and the EVPHR header. However, there is no ioctl available to get the encapsulation type, which makes it difficult for programs to detect whether a packet is LLC-encapsulated or not.

The LLC header is also related to the socket ioctl. The ARPHRD_ type is a more general type of header that can be used for multiplexing in IPv6 packets, and it can include the LLC header. Therefore, there is no need to use the ARPHRD_ type to detect if the packet is LLC-encapsulated or not.

If the LLC header is present in the packet, the handle->linktype will be set to the value specified in the RFC 1483, which is DLT\_LINUX\_SLL. Otherwise, the handle->linktype will be set to -1.

In summary, the LLC header is a part of the IPv6 header that is used to specify the link-layer address of a packet. It is related to the ARPHRD\_ type header and the SNAP header. It is present in the packet after the payloads of the packet have been processed, and it is not necessary to use the ARPHRD\_ type to detect if the packet is LLC-encapsulated or not.


```cpp
#ifndef ARPHRD_FDDI	/* From Linux 2.2.13 */
#define ARPHRD_FDDI	774
#endif
	case ARPHRD_FDDI:
		handle->linktype = DLT_FDDI;
		handle->offset = 3;
		break;

#ifndef ARPHRD_ATM  /* FIXME: How to #include this? */
#define ARPHRD_ATM 19
#endif
	case ARPHRD_ATM:
		/*
		 * The Classical IP implementation in ATM for Linux
		 * supports both what RFC 1483 calls "LLC Encapsulation",
		 * in which each packet has an LLC header, possibly
		 * with a SNAP header as well, prepended to it, and
		 * what RFC 1483 calls "VC Based Multiplexing", in which
		 * different virtual circuits carry different network
		 * layer protocols, and no header is prepended to packets.
		 *
		 * They both have an ARPHRD_ type of ARPHRD_ATM, so
		 * you can't use the ARPHRD_ type to find out whether
		 * captured packets will have an LLC header, and,
		 * while there's a socket ioctl to *set* the encapsulation
		 * type, there's no ioctl to *get* the encapsulation type.
		 *
		 * This means that
		 *
		 *	programs that dissect Linux Classical IP frames
		 *	would have to check for an LLC header and,
		 *	depending on whether they see one or not, dissect
		 *	the frame as LLC-encapsulated or as raw IP (I
		 *	don't know whether there's any traffic other than
		 *	IP that would show up on the socket, or whether
		 *	there's any support for IPv6 in the Linux
		 *	Classical IP code);
		 *
		 *	filter expressions would have to compile into
		 *	code that checks for an LLC header and does
		 *	the right thing.
		 *
		 * Both of those are a nuisance - and, at least on systems
		 * that support PF_PACKET sockets, we don't have to put
		 * up with those nuisances; instead, we can just capture
		 * in cooked mode.  That's what we'll do, if we can.
		 * Otherwise, we'll just fail.
		 */
		if (cooked_ok)
			handle->linktype = DLT_LINUX_SLL;
		else
			handle->linktype = -1;
		break;

```

这段代码定义了一个名为 ARPHRD_IEEE80211 的宏，其值为 801。接下来，定义了两个名为 ARPHRD_IEEE80211_PRISM 和 ARPHRD_IEEE80211 的宏，分别对应于 PRISM 和 IEEE80211 接口。最后，在 handle 结构体中，通过 break 语句确定当前处于哪个宏下的 handle。

简单来说，这段代码定义了两个接口头，用于处理 airmax 和 slope 协议的 802.11 接口。在具体实现时，需要通过 airmax 或者 slope 接口将数据包发送出去，而 handle 结构体中定义的宏将决定使用哪个接口。


```cpp
#ifndef ARPHRD_IEEE80211  /* From Linux 2.4.6 */
#define ARPHRD_IEEE80211 801
#endif
	case ARPHRD_IEEE80211:
		handle->linktype = DLT_IEEE802_11;
		break;

#ifndef ARPHRD_IEEE80211_PRISM  /* From Linux 2.4.18 */
#define ARPHRD_IEEE80211_PRISM 802
#endif
	case ARPHRD_IEEE80211_PRISM:
		handle->linktype = DLT_PRISM_HEADER;
		break;

#ifndef ARPHRD_IEEE80211_RADIOTAP /* new */
```

The code you provided is responsible for handling the link-layer headers of the random link-layer protocol when capturing data on ISDN devices. It determines the type of link-layer header that was provided by the device, and maps it to the appropriate DLT\_ type. If the device provides a raw IP packet with no link-layer header, it is mapped to a new DLT\_I4L\_IP type that has only an Ethernet packet type as a link-layer header. If the device is of the "isdnN" type, it is mapped to DLT\_RAW, and if it is of the "isdY" type, it is mapped to a new DLT\_I4L\_IP type. The handle\_isdn\_n\_ipx


```cpp
#define ARPHRD_IEEE80211_RADIOTAP 803
#endif
	case ARPHRD_IEEE80211_RADIOTAP:
		handle->linktype = DLT_IEEE802_11_RADIO;
		break;

	case ARPHRD_PPP:
		/*
		 * Some PPP code in the kernel supplies no link-layer
		 * header whatsoever to PF_PACKET sockets; other PPP
		 * code supplies PPP link-layer headers ("syncppp.c");
		 * some PPP code might supply random link-layer
		 * headers (PPP over ISDN - there's code in Ethereal,
		 * for example, to cope with PPP-over-ISDN captures
		 * with which the Ethereal developers have had to cope,
		 * heuristically trying to determine which of the
		 * oddball link-layer headers particular packets have).
		 *
		 * As such, we just punt, and run all PPP interfaces
		 * in cooked mode, if we can; otherwise, we just treat
		 * it as DLT_RAW, for now - if somebody needs to capture,
		 * on a 2.0[.x] kernel, on PPP devices that supply a
		 * link-layer header, they'll have to add code here to
		 * map to the appropriate DLT_ type (possibly adding a
		 * new DLT_ type, if necessary).
		 */
		if (cooked_ok)
			handle->linktype = DLT_LINUX_SLL;
		else {
			/*
			 * XXX - handle ISDN types here?  We can't fall
			 * back on cooked sockets, so we'd have to
			 * figure out from the device name what type of
			 * link-layer encapsulation it's using, and map
			 * that to an appropriate DLT_ value, meaning
			 * we'd map "isdnN" devices to DLT_RAW (they
			 * supply raw IP packets with no link-layer
			 * header) and "isdY" devices to a new DLT_I4L_IP
			 * type that has only an Ethernet packet type as
			 * a link-layer header.
			 *
			 * But sometimes we seem to get random crap
			 * in the link-layer header when capturing on
			 * ISDN devices....
			 */
			handle->linktype = DLT_RAW;
		}
		break;

```

这段代码是一个C语言的预处理指令，用于定义某些符号常量的含义。它定义了四个不同的ARPHRD_CISCO和ARPHRD_TUNNEL枚举类型，每种类型对应不同的文件描述符和链路类型。

ARPHRD_CISCO表示使用HDLC协议的ARPHRD设备，ARPHRD_TUNNEL表示使用某个特定隧道协议的ARPHRD设备。在某些情况下，ARPHRD_SIT也可以定义，但其含义是“from Linux 2.2.13”，即表示这是一个旧的Linux发行版。

当程序编译时，会检查其输入源文件是否定义了ARPHRD_CISCO和ARPHRD_TUNNEL枚举类型。如果是，则使用定义的链路类型和处理设备类型。如果不是，则根据定义的枚举类型选择正确的链路类型和处理设备类型。

例如，如果ARPHRD_CISCO定义为ARPHRD_TUNNEL，则不会创建任何链路，而ARPHRD_SIT定义则不会创建任何文件描述符。


```cpp
#ifndef ARPHRD_CISCO
#define ARPHRD_CISCO 513 /* previously ARPHRD_HDLC */
#endif
	case ARPHRD_CISCO:
		handle->linktype = DLT_C_HDLC;
		break;

	/* Not sure if this is correct for all tunnels, but it
	 * works for CIPE */
	case ARPHRD_TUNNEL:
#ifndef ARPHRD_SIT
#define ARPHRD_SIT 776	/* From Linux 2.2.13 */
#endif
	case ARPHRD_SIT:
	case ARPHRD_CSLIP:
	case ARPHRD_SLIP6:
	case ARPHRD_CSLIP6:
	case ARPHRD_ADAPT:
	case ARPHRD_SLIP:
```

这段代码是一个C语言代码片段，定义了两个头文件，并在其中使用了预处理器指令#ifdef和#define。通过#ifdef和#define指令，可以定义某些标识符并在需要时进行定义，从而可以减少代码的冗长。

进一步分析可以发现，在#ifdef ARPHRD_DLCI之前，使用的是#define ARPHRD_DLCI 15。这里定义了一个名为ARPHRD_DLCI的常量，值为15。

在case ARPHRD_DLCI：处，代码使用了一个条件语句。通过在条件中使用了#ifdef，可以确定如果当前系统支持DLT_LINUX_SLL，则执行ARPHRD_DLCI中的代码，否则执行handle->linktype = DLT_RAW；中的代码。

综合起来，这段代码的作用是定义了两个头文件，并根据系统是否支持DLT_LINUX_SLL来定义其中的常量和条件语句。


```cpp
#ifndef ARPHRD_RAWHDLC
#define ARPHRD_RAWHDLC 518
#endif
	case ARPHRD_RAWHDLC:
#ifndef ARPHRD_DLCI
#define ARPHRD_DLCI 15
#endif
	case ARPHRD_DLCI:
		/*
		 * XXX - should some of those be mapped to DLT_LINUX_SLL
		 * instead?  Should we just map all of them to DLT_LINUX_SLL?
		 */
		handle->linktype = DLT_RAW;
		break;

```

这段代码定义了一个名为 ARPHRD_FRAD 的头文件，并在其中定义了三种不同的链接类型：DLT_FRELAY、DLT_LTALK 和 DLT_IP_OVER_FC。这些链接类型分别表示了不同类型的网络链路，如无线局域网（DLT_FRELAY）和局域网（DLT_LTALK）链路，以及支持 RFC 4338-style IP-over-FC 的链路。

当程序运行时，首先会检查其是否定义了 ARPHRD_FRAD 头文件。如果是，就会执行其中的代码。如果不是，则会按照特定的规则从默认行为开始执行。

在 case ARPHRD_FRAD: 这一段代码中，如果当前的链接类型是 ARPHRD_FRAD 中定义的链路类型，就会执行其中的代码，并将 handle->linktype 设置为相应的链路类型。

在 case ARPHRD_LOCALTLK: 这一段代码中，如果当前的链接类型是 ARPHRD_LOCALTLK 中定义的链路类型，就会执行其中的代码，并将 handle->linktype 设置为相应的链路类型。

在 case 18: 这一段代码中，如果当前的链接类型是 ARPHRD_FRAD 中定义的链路类型，就会执行其中的代码，并将 handle->linktype 设置为 DLT_IP_OVER_FC，这个值在 RFC 4338 中定义为 18。如果当前的链接类型不是 ARPHRD_FRAD 中定义的链路类型，就会执行 case ARPHRD_FRAD 和 case ARPHRD_LOCALTLK 中的代码，根据不同的链路类型选择相应的链路类型。


```cpp
#ifndef ARPHRD_FRAD
#define ARPHRD_FRAD 770
#endif
	case ARPHRD_FRAD:
		handle->linktype = DLT_FRELAY;
		break;

	case ARPHRD_LOCALTLK:
		handle->linktype = DLT_LTALK;
		break;

	case 18:
		/*
		 * RFC 4338 defines an encapsulation for IP and ARP
		 * packets that's compatible with the RFC 2625
		 * encapsulation, but that uses a different ARP
		 * hardware type and hardware addresses.  That
		 * ARP hardware type is 18; Linux doesn't define
		 * any ARPHRD_ value as 18, but if it ever officially
		 * supports RFC 4338-style IP-over-FC, it should define
		 * one.
		 *
		 * For now, we map it to DLT_IP_OVER_FC, in the hopes
		 * that this will encourage its use in the future,
		 * should Linux ever officially support RFC 4338-style
		 * IP-over-FC.
		 */
		handle->linktype = DLT_IP_OVER_FC;
		break;

```

It sounds like Christian Svensson is trying to create a mapping from ARPHRD_FC values to DLT_FC_2 and DLT_FC_2\_WITH\_FRAME\_DELIMS for raw Fibre Channel frames. Currently, it appears that there are no network drivers that use these specific ARPHRD\_FC values for IP-over-FC. The ARPHRD\_FC types seem to exist for some reason, but it's not clear what they're intended for or how they're used. Christian is currently mapping these values to DLT\_FC\_2, but he also provides an option of DLT\_FC\_2\_WITH\_FRAME\_DELIMS and DLT\_IP\_OVER\_FC in case there's some old driver out there that uses one of those types for IP-over-FC on which someone wants to capture packets. It's not currently clear what he wants to do with the ARPHRD\_FC values, but it seems like he's trying to find a way to use them for


```cpp
#ifndef ARPHRD_FCPP
#define ARPHRD_FCPP	784
#endif
	case ARPHRD_FCPP:
#ifndef ARPHRD_FCAL
#define ARPHRD_FCAL	785
#endif
	case ARPHRD_FCAL:
#ifndef ARPHRD_FCPL
#define ARPHRD_FCPL	786
#endif
	case ARPHRD_FCPL:
#ifndef ARPHRD_FCFABRIC
#define ARPHRD_FCFABRIC	787
#endif
	case ARPHRD_FCFABRIC:
		/*
		 * Back in 2002, Donald Lee at Cray wanted a DLT_ for
		 * IP-over-FC:
		 *
		 *	https://www.mail-archive.com/tcpdump-workers@sandelman.ottawa.on.ca/msg01043.html
		 *
		 * and one was assigned.
		 *
		 * In a later private discussion (spun off from a message
		 * on the ethereal-users list) on how to get that DLT_
		 * value in libpcap on Linux, I ended up deciding that
		 * the best thing to do would be to have him tweak the
		 * driver to set the ARPHRD_ value to some ARPHRD_FCxx
		 * type, and map all those types to DLT_IP_OVER_FC:
		 *
		 *	I've checked into the libpcap and tcpdump CVS tree
		 *	support for DLT_IP_OVER_FC.  In order to use that,
		 *	you'd have to modify your modified driver to return
		 *	one of the ARPHRD_FCxxx types, in "fcLINUXfcp.c" -
		 *	change it to set "dev->type" to ARPHRD_FCFABRIC, for
		 *	example (the exact value doesn't matter, it can be
		 *	any of ARPHRD_FCPP, ARPHRD_FCAL, ARPHRD_FCPL, or
		 *	ARPHRD_FCFABRIC).
		 *
		 * 11 years later, Christian Svensson wanted to map
		 * various ARPHRD_ values to DLT_FC_2 and
		 * DLT_FC_2_WITH_FRAME_DELIMS for raw Fibre Channel
		 * frames:
		 *
		 *	https://github.com/mcr/libpcap/pull/29
		 *
		 * There doesn't seem to be any network drivers that uses
		 * any of the ARPHRD_FC* values for IP-over-FC, and
		 * it's not exactly clear what the "Dummy types for non
		 * ARP hardware" are supposed to mean (link-layer
		 * header type?  Physical network type?), so it's
		 * not exactly clear why the ARPHRD_FC* types exist
		 * in the first place.
		 *
		 * For now, we map them to DLT_FC_2, and provide an
		 * option of DLT_FC_2_WITH_FRAME_DELIMS, as well as
		 * DLT_IP_OVER_FC just in case there's some old
		 * driver out there that uses one of those types for
		 * IP-over-FC on which somebody wants to capture
		 * packets.
		 */
		handle->dlt_list = (u_int *) malloc(sizeof(u_int) * 3);
		/*
		 * If that fails, just leave the list empty.
		 */
		if (handle->dlt_list != NULL) {
			handle->dlt_list[0] = DLT_FC_2;
			handle->dlt_list[1] = DLT_FC_2_WITH_FRAME_DELIMS;
			handle->dlt_list[2] = DLT_IP_OVER_FC;
			handle->dlt_count = 3;
		}
		handle->linktype = DLT_FC_2;
		break;

```

这段代码是一个C Shell脚本，用于定义和配置ARPHRD（Advanced P bush转发送器）设备。它包含了一个头文件（.h文件）和一个函数（.c文件）。

以下是这段代码的作用：

1. 定义了一个名为ARPHRD_IRDA的常量，值为783。

2. 在头文件中，定义了一个名为ARPHRD_LAPD的常量，用于存储设备的标识符（ID）。

3. 在函数内部，对ARPHRD_IRDA常量进行操作，设置了设备的链接类型为DLT_LINUX_IRDA，并使用“Linux-cooked”模式保存接收到的数据包的方向。

4. 在函数内部，使用handlep结构体，成员变量cooked表示数据包是否已经被处理。

5. 在函数内部，定义了一个名为handle的常量，用于存储当前处理的ARPHRD设备句柄。

6. 在函数内部，使用handle常量，创建一个ARPHRD设备句柄。

7. 在函数内部，通过handle常量，启用ARPHRD设备的复位。

8. 在函数内部，通过handle.cooked变量，报告数据包是否已经被正确处理。

9. 在函数内部，报告ARPHRD设备的ID，以便将来进行识别。


```cpp
#ifndef ARPHRD_IRDA
#define ARPHRD_IRDA	783
#endif
	case ARPHRD_IRDA:
		/* Don't expect IP packet out of this interfaces... */
		handle->linktype = DLT_LINUX_IRDA;
		/* We need to save packet direction for IrDA decoding,
		 * so let's use "Linux-cooked" mode. Jean II
		 *
		 * XXX - this is handled in setup_socket(). */
		/* handlep->cooked = 1; */
		break;

	/* ARPHRD_LAPD is unofficial and randomly allocated, if reallocation
	 * is needed, please report it to <daniele@orlandi.com> */
```

这段代码是一个C语言代码，定义了两个头文件，分别是`ARPHRD_LAPD`和`ARPHRD_NONE`。这两个头文件分别定义了ARPHRD设备的两个不同链路层协议：LAPD和None。

`ARPHRD_LAPD`头文件定义了一种链路层协议，使用了IP协议头中的LAPD协议，因此可以处理IP数据包。

`ARPHRD_NONE`头文件定义了一种链路层协议，使用了IP协议头中的RAW协议，因此只能处理IP数据包，没有任何链层头部。

通过组合这两个头文件，可以定义ARPHRD设备对应的链路层协议。在`ARPHRD_LAPD`中，使用了LAPD协议，可以处理IP数据包；在`ARPHRD_NONE`中，使用了RAW协议，只能处理IP数据包，没有任何链层头部。


```cpp
#ifndef ARPHRD_LAPD
#define ARPHRD_LAPD	8445
#endif
	case ARPHRD_LAPD:
		/* Don't expect IP packet out of this interfaces... */
		handle->linktype = DLT_LINUX_LAPD;
		break;

#ifndef ARPHRD_NONE
#define ARPHRD_NONE	0xFFFE
#endif
	case ARPHRD_NONE:
		/*
		 * No link-layer header; packets are just IP
		 * packets, so use DLT_RAW.
		 */
		handle->linktype = DLT_RAW;
		break;

```

这段代码是一个C语言代码片段，它定义了一个名为ARPHRD_IEEE802154的常量，并定义了一个名为ARPHRD_NETLINK的常量。这两个常量在代码中都被定义了，但它们的含义并没有在代码中详细说明。

ARPHRD_IEEE802154和ARPHRD_NETLINK常量分别定义了两种不同的链路类型，ARPHRD_IEEE802154定义了DLT_IEEE802_15_4_NOFCS，ARPHRD_NETLINK定义了DLT_NETLINK。

在这段代码中，还定义了一个名为handle的变量，handle是一个指向链路句柄的指针。handle变量在代码中没有被定义为const或volatile，因此它可能是一个动态变量，也可能是一个静态变量。

在代码的最后，handle->linktype被定义为DLT_IEEE802_15_4_NOFCS，这意味着该链路句柄正在使用IEEE802.15.4规范定义的链路类型，并且该链路类型支持NoFCS编码。


```cpp
#ifndef ARPHRD_IEEE802154
#define ARPHRD_IEEE802154      804
#endif
       case ARPHRD_IEEE802154:
               handle->linktype =  DLT_IEEE802_15_4_NOFCS;
               break;

#ifndef ARPHRD_NETLINK
#define ARPHRD_NETLINK	824
#endif
	case ARPHRD_NETLINK:
		handle->linktype = DLT_NETLINK;
		/*
		 * We need to use cooked mode, so that in sll_protocol we
		 * pick up the netlink protocol type such as NETLINK_ROUTE,
		 * NETLINK_GENERIC, NETLINK_FIB_LOOKUP, etc.
		 *
		 * XXX - this is handled in setup_socket().
		 */
		/* handlep->cooked = 1; */
		break;

```

这段代码是用于设置数据链路表（DLT）中链路类型的函数。

首先，通过#define ARPHRD_VSOCKMON为826定义了一个名为ARPHRD_VSOCKMON的预处理指令。如果预处理指令被定义了，则编译时会将其替换为实际的常量826。

接着，在handle->linktype变量上进行判断。如果当前处理的链路类型是ARPHRD_VSOCK，则将handle->linktype设置为DLT_VSOCK，否则将handle->linktype设置为-1。

最后，定义了一个名为set_dlt_list_cooked的函数，该函数用于设置数据链路表中的链路类型。函数内部首先检查是否已经定义了ARPHRD_VSOCKMON预处理指令，如果已经定义了，则编译时会将其替换为实际的常量826。然后，如果当前处理的链路类型不是ARPHRD_VSOCK，则将数据链路表中的链路类型设置为DLT_LINUX_SLL或DLT_LINUX_SLL2，并设置 handle->dlt_count 为2（表示当前处理链路类型有两种情况）。

总的来说，这段代码就是用于设置数据链路表中的链路类型的函数。


```cpp
#ifndef ARPHRD_VSOCKMON
#define ARPHRD_VSOCKMON	826
#endif
	case ARPHRD_VSOCKMON:
		handle->linktype = DLT_VSOCK;
		break;

	default:
		handle->linktype = -1;
		break;
	}
}

static void
set_dlt_list_cooked(pcap_t *handle)
{
	/*
	 * Support both DLT_LINUX_SLL and DLT_LINUX_SLL2.
	 */
	handle->dlt_list = (u_int *) malloc(sizeof(u_int) * 2);

	/*
	 * If that failed, just leave the list empty.
	 */
	if (handle->dlt_list != NULL) {
		handle->dlt_list[0] = DLT_LINUX_SLL;
		handle->dlt_list[1] = DLT_LINUX_SLL2;
		handle->dlt_count = 2;
	}
}

```

这段代码是一个用于设置 up a PF_PACKET socket 的函数，通过这个 socket 进行数据传输。它接受一个 pcap_t 类型的数据处理句柄（handle），一个表示设备类型（device）的选项参数（opt.device），并且返回 0 表示成功，一个 PCAP_ERROR_ 错误码表示失败。

首先，它通过调用 pcap_linux 结构的 handlep 成员，获取到 handle 的私有的 pcap_linux 成员，然后设置 device 参数。

接下来，它尝试创建一个套接字（sock_fd）并设置 ARPA 类型（arptype）。然后，它通过 call 有氧 库函数 pcap_valset 获取系统当前网络带宽（val）并检查是否可以创建套接字。

接下来，它设置一个名为 mr 的 packet_mreq 结构体，并设置其岭（egress）和事务（trance）超时时间，以便使用 skf 库函数设置相应的参数。

最后，它使用 defined（in this case SO_BPF_EXTENSIONS）和 defined（in this case SKF_AD_VLAN_TAG_PRESENT）函数来设置一些选项（options），然后通过 call pcap_add 函数在数据处理句柄上添加套接字，并返回 0 表示成功。如果设置失败，它将返回 PCAP_ERROR_，并将错误代码（err）保存到 val 中。


```cpp
/*
 * Try to set up a PF_PACKET socket.
 * Returns 0 on success and a PCAP_ERROR_ value on failure.
 */
static int
setup_socket(pcap_t *handle, int is_any_device)
{
	struct pcap_linux *handlep = handle->priv;
	const char		*device = handle->opt.device;
	int			status = 0;
	int			sock_fd, arptype;
	int			val;
	int			err = 0;
	struct packet_mreq	mr;
#if defined(SO_BPF_EXTENSIONS) && defined(SKF_AD_VLAN_TAG_PRESENT)
	int			bpf_extensions;
	socklen_t		len = sizeof(bpf_extensions);
```

Thank you for the detailed response. It looks like there may have been a bug in the code, and I appreciate the patience and expertise you have shown in trying to help me.

Based on your description, it seems that the issue is with the setsockopt() function, which is causing the PCAP\_ERROR. This function is called to set the timeout for the SO\_TIMESTAMPNS option, and is likely the function that is causing the issue.

I'm not sure if there is a specific error message being displayed when calling setsockopt() that could indicate the cause, but I can suggest a few things that might help.

First, it looks like the function may be trying to set the timeout to negativeinfty, which is not allowed. You can try calling setsockopt() with a timeout of 0 instead.

Another possible solution to this issue might be to check the source of the setsockopt() function. The code is using the "setlocale()" function to set thelocale, but it is unclear from the code if this function is working as expected. If this function is not working correctly, it could cause the setsockopt() call to fail and result in an error.

Finally, it may be helpful to review the code for any other potential issues that may be causing the PCAP\_ERROR. I'm sorry that I'm not able to provide more specific guidance on this right now, but I will do my best to assist you further as needed.


```cpp
#endif

	/*
	 * Open a socket with protocol family packet. If cooked is true,
	 * we open a SOCK_DGRAM socket for the cooked interface, otherwise
	 * we open a SOCK_RAW socket for the raw interface.
	 *
	 * The protocol is set to 0.  This means we will receive no
	 * packets until we "bind" the socket with a non-zero
	 * protocol.  This allows us to setup the ring buffers without
	 * dropping any packets.
	 */
	sock_fd = is_any_device ?
		socket(PF_PACKET, SOCK_DGRAM, 0) :
		socket(PF_PACKET, SOCK_RAW, 0);

	if (sock_fd == -1) {
		if (errno == EPERM || errno == EACCES) {
			/*
			 * You don't have permission to open the
			 * socket.
			 */
			status = PCAP_ERROR_PERM_DENIED;
			snprintf(handle->errbuf, PCAP_ERRBUF_SIZE,
			    "Attempt to create packet socket failed - CAP_NET_RAW may be required");
		} else {
			/*
			 * Other error.
			 */
			status = PCAP_ERROR;
		}
		pcap_fmt_errmsg_for_errno(handle->errbuf, PCAP_ERRBUF_SIZE,
		    errno, "socket");
		return status;
	}

	/*
	 * Get the interface index of the loopback device.
	 * If the attempt fails, don't fail, just set the
	 * "handlep->lo_ifindex" to -1.
	 *
	 * XXX - can there be more than one device that loops
	 * packets back, i.e. devices other than "lo"?  If so,
	 * we'd need to find them all, and have an array of
	 * indices for them, and check all of them in
	 * "pcap_read_packet()".
	 */
	handlep->lo_ifindex = iface_get_id(sock_fd, "lo", handle->errbuf);

	/*
	 * Default value for offset to align link-layer payload
	 * on a 4-byte boundary.
	 */
	handle->offset	 = 0;

	/*
	 * What kind of frames do we have to deal with? Fall back
	 * to cooked mode if we have an unknown interface type
	 * or a type we know doesn't work well in raw mode.
	 */
	if (!is_any_device) {
		/* Assume for now we don't need cooked mode. */
		handlep->cooked = 0;

		if (handle->opt.rfmon) {
			/*
			 * We were asked to turn on monitor mode.
			 * Do so before we get the link-layer type,
			 * because entering monitor mode could change
			 * the link-layer type.
			 */
			err = enter_rfmon_mode(handle, sock_fd, device);
			if (err < 0) {
				/* Hard failure */
				close(sock_fd);
				return err;
			}
			if (err == 0) {
				/*
				 * Nothing worked for turning monitor mode
				 * on.
				 */
				close(sock_fd);
				return PCAP_ERROR_RFMON_NOTSUP;
			}

			/*
			 * Either monitor mode has been turned on for
			 * the device, or we've been given a different
			 * device to open for monitor mode.  If we've
			 * been given a different device, use it.
			 */
			if (handlep->mondevice != NULL)
				device = handlep->mondevice;
		}
		arptype	= iface_get_arptype(sock_fd, device, handle->errbuf);
		if (arptype < 0) {
			close(sock_fd);
			return arptype;
		}
		map_arphrd_to_dlt(handle, arptype, device, 1);
		if (handle->linktype == -1 ||
		    handle->linktype == DLT_LINUX_SLL ||
		    handle->linktype == DLT_LINUX_IRDA ||
		    handle->linktype == DLT_LINUX_LAPD ||
		    handle->linktype == DLT_NETLINK ||
		    (handle->linktype == DLT_EN10MB &&
		     (strncmp("isdn", device, 4) == 0 ||
		      strncmp("isdY", device, 4) == 0))) {
			/*
			 * Unknown interface type (-1), or a
			 * device we explicitly chose to run
			 * in cooked mode (e.g., PPP devices),
			 * or an ISDN device (whose link-layer
			 * type we can only determine by using
			 * APIs that may be different on different
			 * kernels) - reopen in cooked mode.
			 *
			 * If the type is unknown, return a warning;
			 * map_arphrd_to_dlt() has already set the
			 * warning message.
			 */
			if (close(sock_fd) == -1) {
				pcap_fmt_errmsg_for_errno(handle->errbuf,
				    PCAP_ERRBUF_SIZE, errno, "close");
				return PCAP_ERROR;
			}
			sock_fd = socket(PF_PACKET, SOCK_DGRAM, 0);
			if (sock_fd < 0) {
				/*
				 * Fatal error.  We treat this as
				 * a generic error; we already know
				 * that we were able to open a
				 * PF_PACKET/SOCK_RAW socket, so
				 * any failure is a "this shouldn't
				 * happen" case.
				 */
				pcap_fmt_errmsg_for_errno(handle->errbuf,
				    PCAP_ERRBUF_SIZE, errno, "socket");
				return PCAP_ERROR;
			}
			handlep->cooked = 1;

			/*
			 * Get rid of any link-layer type list
			 * we allocated - this only supports cooked
			 * capture.
			 */
			if (handle->dlt_list != NULL) {
				free(handle->dlt_list);
				handle->dlt_list = NULL;
				handle->dlt_count = 0;
				set_dlt_list_cooked(handle);
			}

			if (handle->linktype == -1) {
				/*
				 * Warn that we're falling back on
				 * cooked mode; we may want to
				 * update "map_arphrd_to_dlt()"
				 * to handle the new type.
				 */
				snprintf(handle->errbuf, PCAP_ERRBUF_SIZE,
					"arptype %d not "
					"supported by libpcap - "
					"falling back to cooked "
					"socket",
					arptype);
			}

			/*
			 * IrDA capture is not a real "cooked" capture,
			 * it's IrLAP frames, not IP packets.  The
			 * same applies to LAPD capture.
			 */
			if (handle->linktype != DLT_LINUX_IRDA &&
			    handle->linktype != DLT_LINUX_LAPD &&
			    handle->linktype != DLT_NETLINK)
				handle->linktype = DLT_LINUX_SLL;
			if (handle->linktype == -1) {
				snprintf(handle->errbuf, PCAP_ERRBUF_SIZE,
				    "unknown arptype %d, defaulting to cooked mode",
				    arptype);
				status = PCAP_WARNING;
			}
		}

		handlep->ifindex = iface_get_id(sock_fd, device,
		    handle->errbuf);
		if (handlep->ifindex == -1) {
			close(sock_fd);
			return PCAP_ERROR;
		}

		if ((err = iface_bind(sock_fd, handlep->ifindex,
		    handle->errbuf, 0)) != 0) {
			close(sock_fd);
			return err;
		}
	} else {
		/*
		 * The "any" device.
		 */
		if (handle->opt.rfmon) {
			/*
			 * It doesn't support monitor mode.
			 */
			close(sock_fd);
			return PCAP_ERROR_RFMON_NOTSUP;
		}

		/*
		 * It uses cooked mode.
		 */
		handlep->cooked = 1;
		handle->linktype = DLT_LINUX_SLL;
		handle->dlt_list = NULL;
		handle->dlt_count = 0;
		set_dlt_list_cooked(handle);

		/*
		 * We're not bound to a device.
		 * For now, we're using this as an indication
		 * that we can't transmit; stop doing that only
		 * if we figure out how to transmit in cooked
		 * mode.
		 */
		handlep->ifindex = -1;
	}

	/*
	 * Select promiscuous mode on if "promisc" is set.
	 *
	 * Do not turn allmulti mode on if we don't select
	 * promiscuous mode - on some devices (e.g., Orinoco
	 * wireless interfaces), allmulti mode isn't supported
	 * and the driver implements it by turning promiscuous
	 * mode on, and that screws up the operation of the
	 * card as a normal networking interface, and on no
	 * other platform I know of does starting a non-
	 * promiscuous capture affect which multicast packets
	 * are received by the interface.
	 */

	/*
	 * Hmm, how can we set promiscuous mode on all interfaces?
	 * I am not sure if that is possible at all.  For now, we
	 * silently ignore attempts to turn promiscuous mode on
	 * for the "any" device (so you don't have to explicitly
	 * disable it in programs such as tcpdump).
	 */

	if (!is_any_device && handle->opt.promisc) {
		memset(&mr, 0, sizeof(mr));
		mr.mr_ifindex = handlep->ifindex;
		mr.mr_type    = PACKET_MR_PROMISC;
		if (setsockopt(sock_fd, SOL_PACKET, PACKET_ADD_MEMBERSHIP,
		    &mr, sizeof(mr)) == -1) {
			pcap_fmt_errmsg_for_errno(handle->errbuf,
			    PCAP_ERRBUF_SIZE, errno, "setsockopt (PACKET_ADD_MEMBERSHIP)");
			close(sock_fd);
			return PCAP_ERROR;
		}
	}

	/*
	 * Enable auxiliary data and reserve room for reconstructing
	 * VLAN headers.
	 *
	 * XXX - is enabling auxiliary data necessary, now that we
	 * only support memory-mapped capture?  The kernel's memory-mapped
	 * capture code doesn't seem to check whether auxiliary data
	 * is enabled, it seems to provide it whether it is or not.
	 */
	val = 1;
	if (setsockopt(sock_fd, SOL_PACKET, PACKET_AUXDATA, &val,
		       sizeof(val)) == -1 && errno != ENOPROTOOPT) {
		pcap_fmt_errmsg_for_errno(handle->errbuf, PCAP_ERRBUF_SIZE,
		    errno, "setsockopt (PACKET_AUXDATA)");
		close(sock_fd);
		return PCAP_ERROR;
	}
	handle->offset += VLAN_TAG_LEN;

	/*
	 * If we're in cooked mode, make the snapshot length
	 * large enough to hold a "cooked mode" header plus
	 * 1 byte of packet data (so we don't pass a byte
	 * count of 0 to "recvfrom()").
	 * XXX - we don't know whether this will be DLT_LINUX_SLL
	 * or DLT_LINUX_SLL2, so make sure it's big enough for
	 * a DLT_LINUX_SLL2 "cooked mode" header; a snapshot length
	 * that small is silly anyway.
	 */
	if (handlep->cooked) {
		if (handle->snapshot < SLL2_HDR_LEN + 1)
			handle->snapshot = SLL2_HDR_LEN + 1;
	}
	handle->bufsize = handle->snapshot;

	/*
	 * Set the offset at which to insert VLAN tags.
	 */
	set_vlan_offset(handle);

	if (handle->opt.tstamp_precision == PCAP_TSTAMP_PRECISION_NANO) {
		int nsec_tstamps = 1;

		if (setsockopt(sock_fd, SOL_SOCKET, SO_TIMESTAMPNS, &nsec_tstamps, sizeof(nsec_tstamps)) < 0) {
			snprintf(handle->errbuf, PCAP_ERRBUF_SIZE, "setsockopt: unable to set SO_TIMESTAMPNS");
			close(sock_fd);
			return PCAP_ERROR;
		}
	}

	/*
	 * We've succeeded. Save the socket FD in the pcap structure.
	 */
	handle->fd = sock_fd;

```

这段代码的作用是检查两个条件是否都为真，然后执行相应的操作。如果两个条件都为真，则执行一些特殊的代码处理VLAN检查，否则不做处理。

具体来说，首先检查是否定义了`SO_BPF_EXTENSIONS`和`SKF_AD_VLAN_TAG_PRESENT`，如果两个条件都定义了，那么开始检查`BPF_EXTENSIONS`是否定义，如果有定义，那么尝试从`skb`中读取`SO_BPF_EXTENSIONS`，如果成功，那么将`BPF_EXTENSIONS`设置为`1`，并从`skb`中读取`SKF_AD_VLAN_TAG_PRESENT`，如果成功，那么设置`handle->bpf_codegen_flags`为`BPF_SPECIAL_VLAN_HANDLING`，意味着我们可能需要生成特殊的代码处理VLAN检查。

如果在读取`BPF_EXTENSIONS`时失败，那么根据上面是否定义了`SKF_AD_VLAN_TAG_PRESENT`来决定是否生成特殊代码。如果两个条件中有一个不满足，那么不做处理，继续执行原来的逻辑。


```cpp
#if defined(SO_BPF_EXTENSIONS) && defined(SKF_AD_VLAN_TAG_PRESENT)
	/*
	 * Can we generate special code for VLAN checks?
	 * (XXX - what if we need the special code but it's not supported
	 * by the OS?  Is that possible?)
	 */
	if (getsockopt(sock_fd, SOL_SOCKET, SO_BPF_EXTENSIONS,
	    &bpf_extensions, &len) == 0) {
		if (bpf_extensions >= SKF_AD_VLAN_TAG_PRESENT) {
			/*
			 * Yes, we can.  Request that we do so.
			 */
			handle->bpf_codegen_flags |= BPF_SPECIAL_VLAN_HANDLING;
		}
	}
```

这段代码的作用是检查两个条件，并根据检查结果返回一个整数。首先，它检查是否定义了`SO_BPF_EXTENSIONS`和`SKF_AD_VLAN_TAG_PRESENT`。如果这两个条件都定义了，那么它将返回一个整数。否则，它将返回一个负数。


```cpp
#endif /* defined(SO_BPF_EXTENSIONS) && defined(SKF_AD_VLAN_TAG_PRESENT) */

	return status;
}

/*
 * Attempt to setup memory-mapped access.
 *
 * On success, returns 1, and sets *status to 0 if there are no warnings
 * or to a PCAP_WARNING_ code if there is a warning.
 *
 * On error, returns -1, and sets *status to the appropriate error code;
 * if that is PCAP_ERROR, sets handle->errbuf to the appropriate message.
 */
static int
```

This is a function definition for `pcap_linux_create_滤波器` handling the setting up of a Linux kernel capture driver. It outlines the steps required to create a handle for the滤波器， allocate a oneshot buffer, set up the receive ring, and prepare the filter for use.

The function takes a `handle` parameter, which is the internal handle for the capture driver, and a pointer to the current status of the filter.

The function first checks for any errors and sets the status to `PCAP_ERROR` if an error occurs. If the initialization is successful, the function allocates a oneshot buffer and sets its size to the default value for the receive ring.

The function then sets the timeout for the oneshot buffer and creates the receive ring using the `create_ring` function.

Finally, the function returns 1 if the filter was set up successfully, or -1 if an error occurred.


```cpp
setup_mmapped(pcap_t *handle, int *status)
{
	struct pcap_linux *handlep = handle->priv;
	int ret;

	/*
	 * Attempt to allocate a buffer to hold the contents of one
	 * packet, for use by the oneshot callback.
	 */
	handlep->oneshot_buffer = malloc(handle->snapshot);
	if (handlep->oneshot_buffer == NULL) {
		pcap_fmt_errmsg_for_errno(handle->errbuf, PCAP_ERRBUF_SIZE,
		    errno, "can't allocate oneshot buffer");
		*status = PCAP_ERROR;
		return -1;
	}

	if (handle->opt.buffer_size == 0) {
		/* by default request 2M for the ring buffer */
		handle->opt.buffer_size = 2*1024*1024;
	}
	ret = prepare_tpacket_socket(handle);
	if (ret == -1) {
		free(handlep->oneshot_buffer);
		handlep->oneshot_buffer = NULL;
		*status = PCAP_ERROR;
		return ret;
	}
	ret = create_ring(handle, status);
	if (ret == -1) {
		/*
		 * Error attempting to enable memory-mapped capture;
		 * fail.  create_ring() has set *status.
		 */
		free(handlep->oneshot_buffer);
		handlep->oneshot_buffer = NULL;
		return -1;
	}

	/*
	 * Success.  *status has been set either to 0 if there are no
	 * warnings or to a PCAP_WARNING_ value if there is a warning.
	 *
	 * handle->offset is used to get the current position into the rx ring.
	 * handle->cc is used to store the ring size.
	 */

	/*
	 * Set the timeout to use in poll() before returning.
	 */
	set_poll_timeout(handlep);

	return 1;
}

```

This is a Linux kernel module that implements TCPACK packet接收确认机制。函数接收一个TPACKET结构体，其中包含接收到的数据。首先检查版本号是否支持当前内核版本，如果是，则返回一个integer类型的errno。如果是，则返回errno，否则执行错误处理。如果errno为EINVAL，则表示当前版本不支持该版本，需要返回错误信息。在错误处理过程中，会根据不同的错误情况返回不同的错误信息，或者直接返回-1。在设置TCPACKET结构体时，会对当前版本进行设置。


```cpp
/*
 * Attempt to set the socket to the specified version of the memory-mapped
 * header.
 *
 * Return 0 if we succeed; return 1 if we fail because that version isn't
 * supported; return -1 on any other error, and set handle->errbuf.
 */
static int
init_tpacket(pcap_t *handle, int version, const char *version_str)
{
	struct pcap_linux *handlep = handle->priv;
	int val = version;
	socklen_t len = sizeof(val);

	/*
	 * Probe whether kernel supports the specified TPACKET version;
	 * this also gets the length of the header for that version.
	 *
	 * This socket option was introduced in 2.6.27, which was
	 * also the first release with TPACKET_V2 support.
	 */
	if (getsockopt(handle->fd, SOL_PACKET, PACKET_HDRLEN, &val, &len) < 0) {
		if (errno == EINVAL) {
			/*
			 * EINVAL means this specific version of TPACKET
			 * is not supported. Tell the caller they can try
			 * with a different one; if they've run out of
			 * others to try, let them set the error message
			 * appropriately.
			 */
			return 1;
		}

		/*
		 * All other errors are fatal.
		 */
		if (errno == ENOPROTOOPT) {
			/*
			 * PACKET_HDRLEN isn't supported, which means
			 * that memory-mapped capture isn't supported.
			 * Indicate that in the message.
			 */
			snprintf(handle->errbuf, PCAP_ERRBUF_SIZE,
			    "Kernel doesn't support memory-mapped capture; a 2.6.27 or later 2.x kernel is required, with CONFIG_PACKET_MMAP specified for 2.x kernels");
		} else {
			/*
			 * Some unexpected error.
			 */
			pcap_fmt_errmsg_for_errno(handle->errbuf, PCAP_ERRBUF_SIZE,
			    errno, "can't get %s header len on packet socket",
			    version_str);
		}
		return -1;
	}
	handlep->tp_hdrlen = val;

	val = version;
	if (setsockopt(handle->fd, SOL_PACKET, PACKET_VERSION, &val,
			   sizeof(val)) < 0) {
		pcap_fmt_errmsg_for_errno(handle->errbuf, PCAP_ERRBUF_SIZE,
		    errno, "can't activate %s on packet socket", version_str);
		return -1;
	}
	handlep->tp_version = version;

	return 0;
}

```

这段代码的作用是尝试将 socket 设置为 memory-mapped 头中版本 3，如果设置失败，则尝试使用版本 2，并返回相应的结果。具体实现包括以下几个步骤：

1. 如果用户没有指定立即模式（使用 tpacket_v3），则尝试使用版本 3，并尝试初始化 tpacket_v3。如果初始化成功，则返回 0。
2. 如果初始化失败，或者用户指定了立即模式，则尝试使用版本 2，并返回相应的错误码。
3. 如果既不成功也不失败，则返回 -1。


```cpp
/*
 * Attempt to set the socket to version 3 of the memory-mapped header and,
 * if that fails because version 3 isn't supported, attempt to fall
 * back to version 2.  If version 2 isn't supported, just fail.
 *
 * Return 0 if we succeed and -1 on any other error, and set handle->errbuf.
 */
static int
prepare_tpacket_socket(pcap_t *handle)
{
	int ret;

#ifdef HAVE_TPACKET3
	/*
	 * Try setting the version to TPACKET_V3.
	 *
	 * The only mode in which buffering is done on PF_PACKET
	 * sockets, so that packets might not be delivered
	 * immediately, is TPACKET_V3 mode.
	 *
	 * The buffering cannot be disabled in that mode, so
	 * if the user has requested immediate mode, we don't
	 * use TPACKET_V3.
	 */
	if (!handle->opt.immediate) {
		ret = init_tpacket(handle, TPACKET_V3, "TPACKET_V3");
		if (ret == 0) {
			/*
			 * Success.
			 */
			return 0;
		}
		if (ret == -1) {
			/*
			 * We failed for some reason other than "the
			 * kernel doesn't support TPACKET_V3".
			 */
			return -1;
		}

		/*
		 * This means it returned 1, which means "the kernel
		 * doesn't support TPACKET_V3"; try TPACKET_V2.
		 */
	}
```

这段代码是一个用于设置TPACKET_V2协议版本的函数，主要作用是尝试设置一个名为handle的套接字的TPACKET_V2版本。这个函数会根据设置的版本号（TPACKET_V2或TPACKET_V2_WITH_FLAGS）来返回一个整数ret，如果设置成功则返回0，否则返回一个负数。具体实现包括以下几步：

1. 检查给定的handle是否为有效的套接字。
2. 如果handle有效的套接字，设置套接字的TPACKET_V2版本，并尝试设置。
3. 如果设置TPACKET_V2版本成功，返回0。
4. 如果设置TPACKET_V2版本失败，根据不同的错误情况返回不同的错误信息。
5. 设置错误信息。


```cpp
#endif /* HAVE_TPACKET3 */

	/*
	 * Try setting the version to TPACKET_V2.
	 */
	ret = init_tpacket(handle, TPACKET_V2, "TPACKET_V2");
	if (ret == 0) {
		/*
		 * Success.
		 */
		return 0;
	}

	if (ret == 1) {
		/*
		 * OK, the kernel supports memory-mapped capture, but
		 * not TPACKET_V2.  Set the error message appropriately.
		 */
		snprintf(handle->errbuf, PCAP_ERRBUF_SIZE,
		    "Kernel doesn't support TPACKET_V2; a 2.6.27 or later kernel is required");
	}

	/*
	 * We failed.
	 */
	return -1;
}

```

这段代码定义了一个名为MAX的宏，它的参数为a和b，表示如果a大于b，则返回a，否则返回b。这个宏被用于定义一个名为create_ring的函数。

create_ring函数的作用是设置内存映射访问，允许程序在数据包接收和发送期间修改数据。它接受一个pcap_t类型的指针和一个表示状态的整数作为参数。函数内部使用handle变量，这个指针应该是一个已经存在的pcap_t结构体实例。

函数首先定义了一个名为handlep的struct指针，并将它存储在handle变量中。然后定义了一个名为frames_per_block的整数变量，用于表示每个数据包可以被分为多少个数据块。接下来，函数内部使用j从handle的off_色块中读取取值，并使用i计算出每个数据包的长度。

if (i <= frames_per_block) {
   *status = 0;
   handlep->frames += i;
   handlep->off_色块[i] = j;
   return i;
} else {
   *status = -1;
   handlep->errbuf = pcap_error_msg(handle, "create_ring");
   return -1;
}

总之，这段代码定义了一个MAX宏，用于定义一个名为create_ring的函数，这个函数接受两个参数，一个pcap_t指针和一个表示状态的整数。如果函数成功执行，它将返回1，并将状态设置为0；如果函数失败，它将返回-1，并将错误信息存储到handle的errbuf中。


```cpp
#define MAX(a,b) ((a)>(b)?(a):(b))

/*
 * Attempt to set up memory-mapped access.
 *
 * On success, returns 1, and sets *status to 0 if there are no warnings
 * or to a PCAP_WARNING_ code if there is a warning.
 *
 * On error, returns -1, and sets *status to the appropriate error code;
 * if that is PCAP_ERROR, sets handle->errbuf to the appropriate message.
 */
static int
create_ring(pcap_t *handle, int *status)
{
	struct pcap_linux *handlep = handle->priv;
	unsigned i, j, frames_per_block;
```

It looks like you are trying to optimize the packet receive buffer by making sure it is large enough to hold the maximum number of bytes that can be sent in a single packet, while also allowing for some additional buffer space for comfort.

The current implementation uses the TPACKET\_ALIGN macro to ensure that the buffer is large enough to hold the entire packet, but it only accounts for the receive portion of the packet. It does not take into account the send portion, which may cause the buffer to be too small.

Additionally, it seems that you are using the frame\_size and request.tp\_frame\_nr variables to store the receive and send buffer size and the rounding off of the buffer size, respectively. These variables seem to be the correct size to store the entire packet receive buffer.

Overall, your optimization strategy seems to be a good approach, but it may be worth considering adding a check to ensure that the receive buffer is large enough to hold the entire packet, even if the receive portion is too small.


```cpp
#ifdef HAVE_TPACKET3
	/*
	 * For sockets using TPACKET_V2, the extra stuff at the end of a
	 * struct tpacket_req3 will be ignored, so this is OK even for
	 * those sockets.
	 */
	struct tpacket_req3 req;
#else
	struct tpacket_req req;
#endif
	socklen_t len;
	unsigned int sk_type, tp_reserve, maclen, tp_hdrlen, netoff, macoff;
	unsigned int frame_size;

	/*
	 * Start out assuming no warnings or errors.
	 */
	*status = 0;

	/*
	 * Reserve space for VLAN tag reconstruction.
	 */
	tp_reserve = VLAN_TAG_LEN;

	/*
	 * If we're capturing in cooked mode, reserve space for
	 * a DLT_LINUX_SLL2 header; we don't know yet whether
	 * we'll be using DLT_LINUX_SLL or DLT_LINUX_SLL2, as
	 * that can be changed on an open device, so we reserve
	 * space for the larger of the two.
	 *
	 * XXX - we assume that the kernel is still adding
	 * 16 bytes of extra space, so we subtract 16 from
	 * SLL2_HDR_LEN to get the additional space needed.
	 * (Are they doing that for DLT_LINUX_SLL, the link-
	 * layer header for which is 16 bytes?)
	 *
	 * XXX - should we use TPACKET_ALIGN(SLL2_HDR_LEN - 16)?
	 */
	if (handlep->cooked)
		tp_reserve += SLL2_HDR_LEN - 16;

	/*
	 * Try to request that amount of reserve space.
	 * This must be done before creating the ring buffer.
	 */
	len = sizeof(tp_reserve);
	if (setsockopt(handle->fd, SOL_PACKET, PACKET_RESERVE,
	    &tp_reserve, len) < 0) {
		pcap_fmt_errmsg_for_errno(handle->errbuf,
		    PCAP_ERRBUF_SIZE, errno,
		    "setsockopt (PACKET_RESERVE)");
		*status = PCAP_ERROR;
		return -1;
	}

	switch (handlep->tp_version) {

	case TPACKET_V2:
		/* Note that with large snapshot length (say 256K, which is
		 * the default for recent versions of tcpdump, Wireshark,
		 * TShark, dumpcap or 64K, the value that "-s 0" has given for
		 * a long time with tcpdump), if we use the snapshot
		 * length to calculate the frame length, only a few frames
		 * will be available in the ring even with pretty
		 * large ring size (and a lot of memory will be unused).
		 *
		 * Ideally, we should choose a frame length based on the
		 * minimum of the specified snapshot length and the maximum
		 * packet size.  That's not as easy as it sounds; consider,
		 * for example, an 802.11 interface in monitor mode, where
		 * the frame would include a radiotap header, where the
		 * maximum radiotap header length is device-dependent.
		 *
		 * So, for now, we just do this for Ethernet devices, where
		 * there's no metadata header, and the link-layer header is
		 * fixed length.  We can get the maximum packet size by
		 * adding 18, the Ethernet header length plus the CRC length
		 * (just in case we happen to get the CRC in the packet), to
		 * the MTU of the interface; we fetch the MTU in the hopes
		 * that it reflects support for jumbo frames.  (Even if the
		 * interface is just being used for passive snooping, the
		 * driver might set the size of buffers in the receive ring
		 * based on the MTU, so that the MTU limits the maximum size
		 * of packets that we can receive.)
		 *
		 * If segmentation/fragmentation or receive offload are
		 * enabled, we can get reassembled/aggregated packets larger
		 * than MTU, but bounded to 65535 plus the Ethernet overhead,
		 * due to kernel and protocol constraints */
		frame_size = handle->snapshot;
		if (handle->linktype == DLT_EN10MB) {
			unsigned int max_frame_len;
			int mtu;
			int offload;

			mtu = iface_get_mtu(handle->fd, handle->opt.device,
			    handle->errbuf);
			if (mtu == -1) {
				*status = PCAP_ERROR;
				return -1;
			}
			offload = iface_get_offload(handle);
			if (offload == -1) {
				*status = PCAP_ERROR;
				return -1;
			}
			if (offload)
				max_frame_len = MAX(mtu, 65535);
			else
				max_frame_len = mtu;
			max_frame_len += 18;

			if (frame_size > max_frame_len)
				frame_size = max_frame_len;
		}

		/* NOTE: calculus matching those in tpacket_rcv()
		 * in linux-2.6/net/packet/af_packet.c
		 */
		len = sizeof(sk_type);
		if (getsockopt(handle->fd, SOL_SOCKET, SO_TYPE, &sk_type,
		    &len) < 0) {
			pcap_fmt_errmsg_for_errno(handle->errbuf,
			    PCAP_ERRBUF_SIZE, errno, "getsockopt (SO_TYPE)");
			*status = PCAP_ERROR;
			return -1;
		}
		maclen = (sk_type == SOCK_DGRAM) ? 0 : MAX_LINKHEADER_SIZE;
			/* XXX: in the kernel maclen is calculated from
			 * LL_ALLOCATED_SPACE(dev) and vnet_hdr.hdr_len
			 * in:  packet_snd()           in linux-2.6/net/packet/af_packet.c
			 * then packet_alloc_skb()     in linux-2.6/net/packet/af_packet.c
			 * then sock_alloc_send_pskb() in linux-2.6/net/core/sock.c
			 * but I see no way to get those sizes in userspace,
			 * like for instance with an ifreq ioctl();
			 * the best thing I've found so far is MAX_HEADER in
			 * the kernel part of linux-2.6/include/linux/netdevice.h
			 * which goes up to 128+48=176; since pcap-linux.c
			 * defines a MAX_LINKHEADER_SIZE of 256 which is
			 * greater than that, let's use it.. maybe is it even
			 * large enough to directly replace macoff..
			 */
		tp_hdrlen = TPACKET_ALIGN(handlep->tp_hdrlen) + sizeof(struct sockaddr_ll) ;
		netoff = TPACKET_ALIGN(tp_hdrlen + (maclen < 16 ? 16 : maclen)) + tp_reserve;
			/* NOTE: AFAICS tp_reserve may break the TPACKET_ALIGN
			 * of netoff, which contradicts
			 * linux-2.6/Documentation/networking/packet_mmap.txt
			 * documenting that:
			 * "- Gap, chosen so that packet data (Start+tp_net)
			 * aligns to TPACKET_ALIGNMENT=16"
			 */
			/* NOTE: in linux-2.6/include/linux/skbuff.h:
			 * "CPUs often take a performance hit
			 *  when accessing unaligned memory locations"
			 */
		macoff = netoff - maclen;
		req.tp_frame_size = TPACKET_ALIGN(macoff + frame_size);
		/*
		 * Round the buffer size up to a multiple of the
		 * frame size (rather than rounding down, which
		 * would give a buffer smaller than our caller asked
		 * for, and possibly give zero frames if the requested
		 * buffer size is too small for one frame).
		 */
		req.tp_frame_nr = (handle->opt.buffer_size + req.tp_frame_size - 1)/req.tp_frame_size;
		break;

```

这段代码是一个条件编译语句，名为 `TPACKET_V3`。如果编译器的配置文件中包含了 `TPACKET_V3`，那么它将执行以下代码。

代码的作用是设置请求的 `tp_frame_size` 和 `tp_frame_nr` 成员变量，以满足请求的 `tp_frame_size` 参数。

具体来说，代码首先定义了一个名为 `TPACKET_V3` 的条件，然后在 `case` 语句中定义了一个名为 `TPACKET_V3:` 的枚举。接着，在 `req.tp_frame_size` 和 `req.tp_frame_nr` 成员变量上进行了修改，以满足 `handle->opt.buffer_size` 的需求。

如果 `handle->opt.buffer_size` 的值大于 `MAXIMUM_SNAPLEN`，则将 `req.tp_frame_size` 设置为 `MAXIMUM_SNAPLEN`，并将 `req.tp_frame_nr` 设置为 `(handle->opt.buffer_size + req.tp_frame_size - 1)/req.tp_frame_size`。否则，将 `req.tp_frame_size` 设置为 `MAXIMUM_SNAPLEN`，并将 `req.tp_frame_nr` 设置为 `((handle->opt.buffer_size + req.tp_frame_size - 1)/req.tp_frame_size)`。

最后，代码以 `break` 语句跳出 `case` 语句，从而停止对 `TPACKET_V3:` 的处理。


```cpp
#ifdef HAVE_TPACKET3
	case TPACKET_V3:
		/* The "frames" for this are actually buffers that
		 * contain multiple variable-sized frames.
		 *
		 * We pick a "frame" size of MAXIMUM_SNAPLEN to leave
		 * enough room for at least one reasonably-sized packet
		 * in the "frame". */
		req.tp_frame_size = MAXIMUM_SNAPLEN;
		/*
		 * Round the buffer size up to a multiple of the
		 * "frame" size (rather than rounding down, which
		 * would give a buffer smaller than our caller asked
		 * for, and possibly give zero "frames" if the requested
		 * buffer size is too small for one "frame").
		 */
		req.tp_frame_nr = (handle->opt.buffer_size + req.tp_frame_size - 1)/req.tp_frame_size;
		break;
```

这段代码是一个判断是否为内部错误的函数，用于处理 TPACKET 包的错误信息。其作用是接收一个 TPACKET 包，根据该包的版本和错误码，输出相应的错误信息，然后返回一个负的错误码。

具体来说，代码首先定义了一个名为 default 的默认处理函数，如果在该函数中处理失败，就执行一系列错误处理措施，并返回一个负的错误码。这些错误处理措施包括计算最小区块大小以及根据区块大小调整函数。

对于TPACKET包的错误处理，代码首先检查该包的版本是否为已知值，如果是，则输出相应的错误信息并返回一个负的错误码。如果不是已知值，代码将尝试使用 SIOCSHWTSTAMP 函数来获取系统时间戳，但如果系统不支持该函数，则不做任何处理。最后，代码将计算得到的区块大小作为最小区块大小，并将该大小作为输入参数传入到计算函数中，以计算最大区块大小。


```cpp
#endif
	default:
		snprintf(handle->errbuf, PCAP_ERRBUF_SIZE,
		    "Internal error: unknown TPACKET_ value %u",
		    handlep->tp_version);
		*status = PCAP_ERROR;
		return -1;
	}

	/* compute the minimum block size that will handle this frame.
	 * The block has to be page size aligned.
	 * The max block size allowed by the kernel is arch-dependent and
	 * it's not explicitly checked here. */
	req.tp_block_size = getpagesize();
	while (req.tp_block_size < req.tp_frame_size)
		req.tp_block_size <<= 1;

	frames_per_block = req.tp_block_size/req.tp_frame_size;

	/*
	 * PACKET_TIMESTAMP was added after linux/net_tstamp.h was,
	 * so we check for PACKET_TIMESTAMP.  We check for
	 * linux/net_tstamp.h just in case a system somehow has
	 * PACKET_TIMESTAMP but not linux/net_tstamp.h; that might
	 * be unnecessary.
	 *
	 * SIOCSHWTSTAMP was introduced in the patch that introduced
	 * linux/net_tstamp.h, so we don't bother checking whether
	 * SIOCSHWTSTAMP is defined (if your Linux system has
	 * linux/net_tstamp.h but doesn't define SIOCSHWTSTAMP, your
	 * Linux system is badly broken).
	 */
```

The code you provided is a C program that sets the timestamps for a PCAP-layer socket. The timestamps are either hardware timestamps that are synchronized with the system clock or raw timestamps that are not synchronized with the system clock.

The program first sets the status of the socket to PCAP_WARNING_TSTAMP_TYPE_NOTSUP, which indicates that the timestamps are not supported. If the status is not set to PCAP_WARNING_TSTAMP_TYPE_NOTSUP, the program will try to set the timesource for the hardware timestamp to PCAP_TSTAMP_ADAPTER and then call the `setsockopt()` function to set the timestamps.

If the timesource is set to PCAP_TSTAMP_ADAPTER, the program will use the adapter to get the hardware timestamp. If the timesource is set to PCAP_TSTAMP_ADAPTER_UNSYNCED, the program will use the adapter to get the raw hardware timestamp.

The program then returns the status of the socket.

It is important to note that the code may not work as intended in all cases, and there may be additional error handling that is not included in the code you provided.


```cpp
#if defined(HAVE_LINUX_NET_TSTAMP_H) && defined(PACKET_TIMESTAMP)
	/*
	 * If we were told to do so, ask the kernel and the driver
	 * to use hardware timestamps.
	 *
	 * Hardware timestamps are only supported with mmapped
	 * captures.
	 */
	if (handle->opt.tstamp_type == PCAP_TSTAMP_ADAPTER ||
	    handle->opt.tstamp_type == PCAP_TSTAMP_ADAPTER_UNSYNCED) {
		struct hwtstamp_config hwconfig;
		struct ifreq ifr;
		int timesource;

		/*
		 * Ask for hardware time stamps on all packets,
		 * including transmitted packets.
		 */
		memset(&hwconfig, 0, sizeof(hwconfig));
		hwconfig.tx_type = HWTSTAMP_TX_ON;
		hwconfig.rx_filter = HWTSTAMP_FILTER_ALL;

		memset(&ifr, 0, sizeof(ifr));
		pcap_strlcpy(ifr.ifr_name, handle->opt.device, sizeof(ifr.ifr_name));
		ifr.ifr_data = (void *)&hwconfig;

		/*
		 * This may require CAP_NET_ADMIN.
		 */
		if (ioctl(handle->fd, SIOCSHWTSTAMP, &ifr) < 0) {
			switch (errno) {

			case EPERM:
				/*
				 * Treat this as an error, as the
				 * user should try to run this
				 * with the appropriate privileges -
				 * and, if they can't, shouldn't
				 * try requesting hardware time stamps.
				 */
				*status = PCAP_ERROR_PERM_DENIED;
				snprintf(handle->errbuf, PCAP_ERRBUF_SIZE,
				    "Attempt to set hardware timestamp failed - CAP_NET_ADMIN may be required");
				return -1;

			case EOPNOTSUPP:
			case ERANGE:
				/*
				 * Treat this as a warning, as the
				 * only way to fix the warning is to
				 * get an adapter that supports hardware
				 * time stamps for *all* packets.
				 * (ERANGE means "we support hardware
				 * time stamps, but for packets matching
				 * that particular filter", so it means
				 * "we don't support hardware time stamps
				 * for all incoming packets" here.)
				 *
				 * We'll just fall back on the standard
				 * host time stamps.
				 */
				*status = PCAP_WARNING_TSTAMP_TYPE_NOTSUP;
				break;

			default:
				pcap_fmt_errmsg_for_errno(handle->errbuf,
				    PCAP_ERRBUF_SIZE, errno,
				    "SIOCSHWTSTAMP failed");
				*status = PCAP_ERROR;
				return -1;
			}
		} else {
			/*
			 * Well, that worked.  Now specify the type of
			 * hardware time stamp we want for this
			 * socket.
			 */
			if (handle->opt.tstamp_type == PCAP_TSTAMP_ADAPTER) {
				/*
				 * Hardware timestamp, synchronized
				 * with the system clock.
				 */
				timesource = SOF_TIMESTAMPING_SYS_HARDWARE;
			} else {
				/*
				 * PCAP_TSTAMP_ADAPTER_UNSYNCED - hardware
				 * timestamp, not synchronized with the
				 * system clock.
				 */
				timesource = SOF_TIMESTAMPING_RAW_HARDWARE;
			}
			if (setsockopt(handle->fd, SOL_PACKET, PACKET_TIMESTAMP,
				(void *)&timesource, sizeof(timesource))) {
				pcap_fmt_errmsg_for_errno(handle->errbuf,
				    PCAP_ERRBUF_SIZE, errno,
				    "can't set PACKET_TIMESTAMP");
				*status = PCAP_ERROR;
				return -1;
			}
		}
	}
```

这段代码是一个网络应用程序，它请求操作系统为它创建一个套接字，以便能够发送数据。它包括了以下主要步骤：

1. 询问操作系统以获取套接字。如果操作系统能够创建套接字，则创建一个接收 ring。
2. 如果操作系统无法创建套接字，则设置 Retry 标志并尝试再次创建套接字。
3. 如果套接字创建成功，则设置一些参数，如请求缓冲区的最大大小、允许超时时间等。
4. 如果套接字支持TPACKET3，则设置一些选项，如允许超时时间、默认超时时间等。
5. 设置套接字的发送和接收 ring。
6. 设置允许的 HTTP 请求时间。


```cpp
#endif /* HAVE_LINUX_NET_TSTAMP_H && PACKET_TIMESTAMP */

	/* ask the kernel to create the ring */
retry:
	req.tp_block_nr = req.tp_frame_nr / frames_per_block;

	/* req.tp_frame_nr is requested to match frames_per_block*req.tp_block_nr */
	req.tp_frame_nr = req.tp_block_nr * frames_per_block;

#ifdef HAVE_TPACKET3
	/* timeout value to retire block - use the configured buffering timeout, or default if <0. */
	if (handlep->timeout > 0) {
		/* Use the user specified timeout as the block timeout */
		req.tp_retire_blk_tov = handlep->timeout;
	} else if (handlep->timeout == 0) {
		/*
		 * In pcap, this means "infinite timeout"; TPACKET_V3
		 * doesn't support that, so just set it to UINT_MAX
		 * milliseconds.  In the TPACKET_V3 loop, if the
		 * timeout is 0, and we haven't yet seen any packets,
		 * and we block and still don't have any packets, we
		 * keep blocking until we do.
		 */
		req.tp_retire_blk_tov = UINT_MAX;
	} else {
		/*
		 * XXX - this is not valid; use 0, meaning "have the
		 * kernel pick a default", for now.
		 */
		req.tp_retire_blk_tov = 0;
	}
	/* private data not used */
	req.tp_sizeof_priv = 0;
	/* Rx ring - feature request bits - none (rxhash will not be filled) */
	req.tp_feature_req_word = 0;
```

This function appears to dynamically allocate an Rx Ring from a network packet stream, using the raw data provided by the packet, and returns the allocated ring. It does this by first allocating a portion of the data from the packet and then using that data to allocate a ring for each frame header pointer in the packet. The handle to the allocated ring is then updated to point to the beginning of the ring.

The function first checks if the allocate_ring function allocation failed, in which case it will return an error and destroy the ring. If the allocation succeeds, it will then fill the ring with the data from the packet, and update the handle to point to the end of the ring. Finally, the function returns 1 to indicate success.


```cpp
#endif

	if (setsockopt(handle->fd, SOL_PACKET, PACKET_RX_RING,
					(void *) &req, sizeof(req))) {
		if ((errno == ENOMEM) && (req.tp_block_nr > 1)) {
			/*
			 * Memory failure; try to reduce the requested ring
			 * size.
			 *
			 * We used to reduce this by half -- do 5% instead.
			 * That may result in more iterations and a longer
			 * startup, but the user will be much happier with
			 * the resulting buffer size.
			 */
			if (req.tp_frame_nr < 20)
				req.tp_frame_nr -= 1;
			else
				req.tp_frame_nr -= req.tp_frame_nr/20;
			goto retry;
		}
		pcap_fmt_errmsg_for_errno(handle->errbuf, PCAP_ERRBUF_SIZE,
		    errno, "can't create rx ring on packet socket");
		*status = PCAP_ERROR;
		return -1;
	}

	/* memory map the rx ring */
	handlep->mmapbuflen = req.tp_block_nr * req.tp_block_size;
	handlep->mmapbuf = mmap(0, handlep->mmapbuflen,
	    PROT_READ|PROT_WRITE, MAP_SHARED, handle->fd, 0);
	if (handlep->mmapbuf == MAP_FAILED) {
		pcap_fmt_errmsg_for_errno(handle->errbuf, PCAP_ERRBUF_SIZE,
		    errno, "can't mmap rx ring");

		/* clear the allocated ring on error*/
		destroy_ring(handle);
		*status = PCAP_ERROR;
		return -1;
	}

	/* allocate a ring for each frame header pointer*/
	handle->cc = req.tp_frame_nr;
	handle->buffer = malloc(handle->cc * sizeof(union thdr *));
	if (!handle->buffer) {
		pcap_fmt_errmsg_for_errno(handle->errbuf, PCAP_ERRBUF_SIZE,
		    errno, "can't allocate ring of frame headers");

		destroy_ring(handle);
		*status = PCAP_ERROR;
		return -1;
	}

	/* fill the header ring with proper frame ptr*/
	handle->offset = 0;
	for (i=0; i<req.tp_block_nr; ++i) {
		u_char *base = &handlep->mmapbuf[i*req.tp_block_size];
		for (j=0; j<frames_per_block; ++j, ++handle->offset) {
			RING_GET_CURRENT_FRAME(handle) = base;
			base += req.tp_frame_size;
		}
	}

	handle->bufsize = req.tp_frame_size;
	handle->offset = 0;
	return 1;
}

```

这段代码是一个用于释放环形数据链路设备（RING）资源的功能。这个函数是在 `pcap_初始化` 函数之后定义的。

具体来说，这个函数接收一个 `pcap_t` 类型的数据链路对象（handle），然后执行以下操作：

1. 告诉操作系统销毁环形数据链路设备。这个函数不检查 `setsockopt` 函数的失败，因为我们可以从错误中恢复，或者在设备建立完成之前就已经设置了它。
2. 如果环形数据链路设备已经映射到内存中，那么函数会释放它。

这段代码仅在 `pcap_初始化` 函数被调用时执行。在 `pcap_终止` 函数中，可能不会调用这个函数，因此这个函数的实现不会在所有情况下都适用。


```cpp
/* free all ring related resources*/
static void
destroy_ring(pcap_t *handle)
{
	struct pcap_linux *handlep = handle->priv;

	/*
	 * Tell the kernel to destroy the ring.
	 * We don't check for setsockopt failure, as 1) we can't recover
	 * from an error and 2) we might not yet have set it up in the
	 * first place.
	 */
	struct tpacket_req req;
	memset(&req, 0, sizeof(req));
	(void)setsockopt(handle->fd, SOL_PACKET, PACKET_RX_RING,
				(void *) &req, sizeof(req));

	/* if ring is mapped, unmap it*/
	if (handlep->mmapbuf) {
		/* do not test for mmap failure, as we can't recover from any error */
		(void)munmap(handlep->mmapbuf, handlep->mmapbuflen);
		handlep->mmapbuf = NULL;
	}
}

```

这段代码定义了一个特殊的一击回调函数，用于在`pcap_next()`和`pcap_next_ex()`中传递给其传递的包数据。这个回调函数在函数内部被嵌套，因此在函数内部使用它时，需要手动复制这个包数据。

这是因为`pcap_next()`和`pcap_next_ex()`在传递包数据给回调函数之后，需要等待回调函数返回。然而，`pcap_read_linux_mmap()`在传递包数据给回调函数之后，需要立即释放这个包数据，否则它认为在环中还有至少一个未处理的包，因此会调用`select()`函数，表明有数据需要处理。因此，在回调函数中，需要复制这个包数据以确保它仍然有效。

需要注意的是，如果捕获使用的是 ring 缓冲，则使用`pcap_next()`或`pcap_next_ex()`将需要复制的包数据数量将大于使用`pcap_loop()`或`pcap_dispatch()`。因此，如果想减少这种复制的次数，可以考虑使用`pcap_loop()`或`pcap_dispatch()`。


```cpp
/*
 * Special one-shot callback, used for pcap_next() and pcap_next_ex(),
 * for Linux mmapped capture.
 *
 * The problem is that pcap_next() and pcap_next_ex() expect the packet
 * data handed to the callback to be valid after the callback returns,
 * but pcap_read_linux_mmap() has to release that packet as soon as
 * the callback returns (otherwise, the kernel thinks there's still
 * at least one unprocessed packet available in the ring, so a select()
 * will immediately return indicating that there's data to process), so,
 * in the callback, we have to make a copy of the packet.
 *
 * Yes, this means that, if the capture is using the ring buffer, using
 * pcap_next() or pcap_next_ex() requires more copies than using
 * pcap_loop() or pcap_dispatch().  If that bothers you, don't use
 * pcap_next() or pcap_next_ex().
 */
```

这两段代码定义了两个名为 `pcap_oneshot_linux` 和 `pcap_getnonblock_linux` 的函数。它们属于名为 `pcap_pkthdr` 的结构体。

这两段代码的主要目的是实现 pcap 库中的 `oneshot` 函数。`oneshot` 函数用于将单个数据包转换为 oneshot 包，其中包含一个数据包的所有内容（包括数据和消息头）。

具体来说，这两段代码实现了以下步骤：

1. 定义了 `pcap_oneshot_linux` 函数，该函数接收一个用户数据指针 `user`，一个 `pcap_pkthdr` 结构体指针 `h`，以及一个字节数组 `bytes`。函数首先将 `user` 和 `h` 传递给 `sp` 变量，然后将 `bytes` 字节数组中的数据复制到 `handlep` 变量所指向的 `struct pcap_linux` 结构体中。最后，将 `handlep->oneshot_buffer` 指向 `bytes`。

2. 定义了 `pcap_getnonblock_linux` 函数，该函数接收一个 `pcap_t` 结构体指针 `handle`。函数首先将 `handlep` 指向的 `struct pcap_linux` 结构体指针传递给 `handlep`，然后使用 `handlep->timeout` 获取一个超时时间（如果设置为非负值，则为 0）。最后，函数返回一个布尔值，表示是否可以继续执行非阻塞操作。

这两段代码的作用是将一个数据包转换为 oneshot 包，并设置一个非阻塞超时时间。这可以帮助在需要时使用 `oneshot` 函数，而不会影响现有的阻塞操作。


```cpp
static void
pcap_oneshot_linux(u_char *user, const struct pcap_pkthdr *h,
    const u_char *bytes)
{
	struct oneshot_userdata *sp = (struct oneshot_userdata *)user;
	pcap_t *handle = sp->pd;
	struct pcap_linux *handlep = handle->priv;

	*sp->hdr = *h;
	memcpy(handlep->oneshot_buffer, bytes, h->caplen);
	*sp->pkt = handlep->oneshot_buffer;
}

static int
pcap_getnonblock_linux(pcap_t *handle)
{
	struct pcap_linux *handlep = handle->priv;

	/* use negative value of timeout to indicate non blocking ops */
	return (handlep->timeout<0);
}

```

This function appears to be a part of the Linux `pcap` packetcapture library. It appears to be setting up non-blocking file descriptors for use with the `pcap_send` function, which sends packets over a network.

The function first sets the file descriptor to non-blocking mode and maps the return value to the corresponding non-blocking mode value. It then checks whether the non-blocking mode parameter was set correctly. If it was, it closes the eventfd that was previously opened for blocking mode.

If non-blocking mode was not set, it opens the eventfd for blocking mode and updates the timeout value for the `pcap_send` function.

Finally, it updates the timeout value for the `poll` function used by `pcap_send` to use in non-blocking mode.


```cpp
static int
pcap_setnonblock_linux(pcap_t *handle, int nonblock)
{
	struct pcap_linux *handlep = handle->priv;

	/*
	 * Set the file descriptor to non-blocking mode, as we use
	 * it for sending packets.
	 */
	if (pcap_setnonblock_fd(handle, nonblock) == -1)
		return -1;

	/*
	 * Map each value to their corresponding negation to
	 * preserve the timeout value provided with pcap_set_timeout.
	 */
	if (nonblock) {
		if (handlep->timeout >= 0) {
			/*
			 * Indicate that we're switching to
			 * non-blocking mode.
			 */
			handlep->timeout = ~handlep->timeout;
		}
		if (handlep->poll_breakloop_fd != -1) {
			/* Close the eventfd; we do not need it in nonblock mode. */
			close(handlep->poll_breakloop_fd);
			handlep->poll_breakloop_fd = -1;
		}
	} else {
		if (handlep->poll_breakloop_fd == -1) {
			/* If we did not have an eventfd, open one now that we are blocking. */
			if ( ( handlep->poll_breakloop_fd = eventfd(0, EFD_NONBLOCK) ) == -1 ) {
				int save_errno = errno;
				snprintf(handle->errbuf, PCAP_ERRBUF_SIZE,
						"Could not open eventfd: %s",
						strerror(errno));
				errno = save_errno;
				return -1;
			}
		}
		if (handlep->timeout < 0) {
			handlep->timeout = ~handlep->timeout;
		}
	}
	/* Update the timeout to use in poll(). */
	set_poll_timeout(handlep);
	return 0;
}

```

这段代码的作用是获取名为 "test" 的环缓冲区帧的 status 字段，该帧的偏移量为 16。它属于 Linux 头文件 <linux/pcap.h>。


```cpp
/*
 * Get the status field of the ring buffer frame at a specified offset.
 */
static inline u_int
pcap_get_ring_frame_status(pcap_t *handle, int offset)
{
	struct pcap_linux *handlep = handle->priv;
	union thdr h;

	h.raw = RING_GET_FRAME_AT(handle, offset);
	switch (handlep->tp_version) {
	case TPACKET_V2:
		return __atomic_load_n(&h.h2->tp_status, __ATOMIC_ACQUIRE);
		break;
#ifdef HAVE_TPACKET3
	case TPACKET_V3:
		return __atomic_load_n(&h.h3->hdr.bh1.block_status, __ATOMIC_ACQUIRE);
		break;
```

This is a function that deals with the handling of network interface configuration flagged for the purpose of enabling or disabling it. It takes in a handle to an ioctl() function, and a pointer to a capture structure.

The function first checks for any errors with the ioctl operation using the errno returned by the ioctl() function. If an error occurs, it checks for the specific error code ENXIO or ENODEV, and in some cases, it returns a return code accordingly. If the ioctl operation is successful, it then checks if the interface is still up and attempts to reset the required select timeout of the interface.

If the interface is still down, the function spinning in a loop doing poll()s that immediately time out if there's no indication on any descriptor. After that, it returns 0 to indicate successful finish.

If the ioctl operation is not successful and the error is not in the expected codes, it returns the last error returned by the ioctl() function.


```cpp
#endif
	}
	/* This should not happen. */
	return 0;
}

/*
 * Block waiting for frames to be available.
 */
static int pcap_wait_for_frames_mmap(pcap_t *handle)
{
	struct pcap_linux *handlep = handle->priv;
	int timeout;
	struct ifreq ifr;
	int ret;
	struct pollfd pollinfo[2];
	int numpollinfo;
	pollinfo[0].fd = handle->fd;
	pollinfo[0].events = POLLIN;
	if ( handlep->poll_breakloop_fd == -1 ) {
		numpollinfo = 1;
		pollinfo[1].revents = 0;
		/*
		 * We set pollinfo[1].revents to zero, even though
		 * numpollinfo = 1 meaning that poll() doesn't see
		 * pollinfo[1], so that we do not have to add a
		 * conditional of numpollinfo > 1 below when we
		 * test pollinfo[1].revents.
		 */
	} else {
		pollinfo[1].fd = handlep->poll_breakloop_fd;
		pollinfo[1].events = POLLIN;
		numpollinfo = 2;
	}

	/*
	 * Keep polling until we either get some packets to read, see
	 * that we got told to break out of the loop, get a fatal error,
	 * or discover that the device went away.
	 *
	 * In non-blocking mode, we must still do one poll() to catch
	 * any pending error indications, but the poll() has a timeout
	 * of 0, so that it doesn't block, and we quit after that one
	 * poll().
	 *
	 * If we've seen an ENETDOWN, it might be the first indication
	 * that the device went away, or it might just be that it was
	 * configured down.  Unfortunately, there's no guarantee that
	 * the device has actually been removed as an interface, because:
	 *
	 * 1) if, as appears to be the case at least some of the time,
	 * the PF_PACKET socket code first gets a NETDEV_DOWN indication
	 * for the device and then gets a NETDEV_UNREGISTER indication
	 * for it, the first indication will cause a wakeup with ENETDOWN
	 * but won't set the packet socket's field for the interface index
	 * to -1, and the second indication won't cause a wakeup (because
	 * the first indication also caused the protocol hook to be
	 * unregistered) but will set the packet socket's field for the
	 * interface index to -1;
	 *
	 * 2) even if just a NETDEV_UNREGISTER indication is registered,
	 * the packet socket's field for the interface index only gets
	 * set to -1 after the wakeup, so there's a small but non-zero
	 * risk that a thread blocked waiting for the wakeup will get
	 * to the "fetch the socket name" code before the interface index
	 * gets set to -1, so it'll get the old interface index.
	 *
	 * Therefore, if we got an ENETDOWN and haven't seen a packet
	 * since then, we assume that we might be waiting for the interface
	 * to disappear, and poll with a timeout to try again in a short
	 * period of time.  If we *do* see a packet, the interface has
	 * come back up again, and is *definitely* still there, so we
	 * don't need to poll.
	 */
	for (;;) {
		/*
		 * Yes, we do this even in non-blocking mode, as it's
		 * the only way to get error indications from a
		 * tpacket socket.
		 *
		 * The timeout is 0 in non-blocking mode, so poll()
		 * returns immediately.
		 */
		timeout = handlep->poll_timeout;

		/*
		 * If we got an ENETDOWN and haven't gotten an indication
		 * that the device has gone away or that the device is up,
		 * we don't yet know for certain whether the device has
		 * gone away or not, do a poll() with a 1-millisecond timeout,
		 * as we have to poll indefinitely for "device went away"
		 * indications until we either get one or see that the
		 * device is up.
		 */
		if (handlep->netdown) {
			if (timeout != 0)
				timeout = 1;
		}
		ret = poll(pollinfo, numpollinfo, timeout);
		if (ret < 0) {
			/*
			 * Error.  If it's not EINTR, report it.
			 */
			if (errno != EINTR) {
				pcap_fmt_errmsg_for_errno(handle->errbuf,
				    PCAP_ERRBUF_SIZE, errno,
				    "can't poll on packet socket");
				return PCAP_ERROR;
			}

			/*
			 * It's EINTR; if we were told to break out of
			 * the loop, do so.
			 */
			if (handle->break_loop) {
				handle->break_loop = 0;
				return PCAP_ERROR_BREAK;
			}
		} else if (ret > 0) {
			/*
			 * OK, some descriptor is ready.
			 * Check the socket descriptor first.
			 *
			 * As I read the Linux man page, pollinfo[0].revents
			 * will either be POLLIN, POLLERR, POLLHUP, or POLLNVAL.
			 */
			if (pollinfo[0].revents == POLLIN) {
				/*
				 * OK, we may have packets to
				 * read.
				 */
				break;
			}
			if (pollinfo[0].revents != 0) {
				/*
				 * There's some indication other than
				 * "you can read on this descriptor" on
				 * the descriptor.
				 */
				if (pollinfo[0].revents & POLLNVAL) {
					snprintf(handle->errbuf,
					    PCAP_ERRBUF_SIZE,
					    "Invalid polling request on packet socket");
					return PCAP_ERROR;
				}
				if (pollinfo[0].revents & (POLLHUP | POLLRDHUP)) {
					snprintf(handle->errbuf,
					    PCAP_ERRBUF_SIZE,
					    "Hangup on packet socket");
					return PCAP_ERROR;
				}
				if (pollinfo[0].revents & POLLERR) {
					/*
					 * Get the error.
					 */
					int err;
					socklen_t errlen;

					errlen = sizeof(err);
					if (getsockopt(handle->fd, SOL_SOCKET,
					    SO_ERROR, &err, &errlen) == -1) {
						/*
						 * The call *itself* returned
						 * an error; make *that*
						 * the error.
						 */
						err = errno;
					}

					/*
					 * OK, we have the error.
					 */
					if (err == ENETDOWN) {
						/*
						 * The device on which we're
						 * capturing went away or the
						 * interface was taken down.
						 *
						 * We don't know for certain
						 * which happened, and the
						 * next poll() may indicate
						 * that there are packets
						 * to be read, so just set
						 * a flag to get us to do
						 * checks later, and set
						 * the required select
						 * timeout to 1 millisecond
						 * so that event loops that
						 * check our socket descriptor
						 * also time out so that
						 * they can call us and we
						 * can do the checks.
						 */
						handlep->netdown = 1;
						handle->required_select_timeout = &netdown_timeout;
					} else if (err == 0) {
						/*
						 * This shouldn't happen, so
						 * report a special indication
						 * that it did.
						 */
						snprintf(handle->errbuf,
						    PCAP_ERRBUF_SIZE,
						    "Error condition on packet socket: Reported error was 0");
						return PCAP_ERROR;
					} else {
						pcap_fmt_errmsg_for_errno(handle->errbuf,
						    PCAP_ERRBUF_SIZE,
						    err,
						    "Error condition on packet socket");
						return PCAP_ERROR;
					}
				}
			}
			/*
			 * Now check the event device.
			 */
			if (pollinfo[1].revents & POLLIN) {
				ssize_t nread;
				uint64_t value;

				/*
				 * This should never fail, but, just
				 * in case....
				 */
				nread = read(handlep->poll_breakloop_fd, &value,
				    sizeof(value));
				if (nread == -1) {
					pcap_fmt_errmsg_for_errno(handle->errbuf,
					    PCAP_ERRBUF_SIZE,
					    errno,
					    "Error reading from event FD");
					return PCAP_ERROR;
				}

				/*
				 * According to the Linux read(2) man
				 * page, read() will transfer at most
				 * 2^31-1 bytes, so the return value is
				 * either -1 or a value between 0
				 * and 2^31-1, so it's non-negative.
				 *
				 * Cast it to size_t to squelch
				 * warnings from the compiler; add this
				 * comment to squelch warnings from
				 * humans reading the code. :-)
				 *
				 * Don't treat an EOF as an error, but
				 * *do* treat a short read as an error;
				 * that "shouldn't happen", but....
				 */
				if (nread != 0 &&
				    (size_t)nread < sizeof(value)) {
					snprintf(handle->errbuf, PCAP_ERRBUF_SIZE,
					    "Short read from event FD: expected %zu, got %zd",
					    sizeof(value), nread);
					return PCAP_ERROR;
				}

				/*
				 * This event gets signaled by a
				 * pcap_breakloop() call; if we were told
				 * to break out of the loop, do so.
				 */
				if (handle->break_loop) {
					handle->break_loop = 0;
					return PCAP_ERROR_BREAK;
				}
			}
		}

		/*
		 * Either:
		 *
		 *   1) we got neither an error from poll() nor any
		 *      readable descriptors, in which case there
		 *      are no packets waiting to read
		 *
		 * or
		 *
		 *   2) We got readable descriptors but the PF_PACKET
		 *      socket wasn't one of them, in which case there
		 *      are no packets waiting to read
		 *
		 * so, if we got an ENETDOWN, we've drained whatever
		 * packets were available to read at the point of the
		 * ENETDOWN.
		 *
		 * So, if we got an ENETDOWN and haven't gotten an indication
		 * that the device has gone away or that the device is up,
		 * we don't yet know for certain whether the device has
		 * gone away or not, check whether the device exists and is
		 * up.
		 */
		if (handlep->netdown) {
			if (!device_still_exists(handle)) {
				/*
				 * The device doesn't exist any more;
				 * report that.
				 *
				 * XXX - we should really return an
				 * appropriate error for that, but
				 * pcap_dispatch() etc. aren't documented
				 * as having error returns other than
				 * PCAP_ERROR or PCAP_ERROR_BREAK.
				 */
				snprintf(handle->errbuf,  PCAP_ERRBUF_SIZE,
				    "The interface disappeared");
				return PCAP_ERROR;
			}

			/*
			 * The device still exists; try to see if it's up.
			 */
			memset(&ifr, 0, sizeof(ifr));
			pcap_strlcpy(ifr.ifr_name, handlep->device,
			    sizeof(ifr.ifr_name));
			if (ioctl(handle->fd, SIOCGIFFLAGS, &ifr) == -1) {
				if (errno == ENXIO || errno == ENODEV) {
					/*
					 * OK, *now* it's gone.
					 *
					 * XXX - see above comment.
					 */
					snprintf(handle->errbuf,
					    PCAP_ERRBUF_SIZE,
					    "The interface disappeared");
					return PCAP_ERROR;
				} else {
					pcap_fmt_errmsg_for_errno(handle->errbuf,
					    PCAP_ERRBUF_SIZE, errno,
					    "%s: Can't get flags",
					    handlep->device);
					return PCAP_ERROR;
				}
			}
			if (ifr.ifr_flags & IFF_UP) {
				/*
				 * It's up, so it definitely still exists.
				 * Cancel the ENETDOWN indication - we
				 * presumably got it due to the interface
				 * going down rather than the device going
				 * away - and revert to "no required select
				 * timeout.
				 */
				handlep->netdown = 0;
				handle->required_select_timeout = NULL;
			}
		}

		/*
		 * If we're in non-blocking mode, just quit now, rather
		 * than spinning in a loop doing poll()s that immediately
		 * time out if there's no indication on any descriptor.
		 */
		if (handlep->poll_timeout == 0)
			break;
	}
	return 0;
}

```



This is a function definition for `bp_push_部门的handle <工程项目， >. It appears to be part of a software framework for managing network interfaces, and it functions with the `push_部门的handle` function.

The `bp_push_部门的handle` function appears to take two arguments: `handlep` and `tp_vlan_tpid`、`tp_vlan_tci`。It先将 `handlep` 和 `tp_vlan_tpid`、`tp_vlan_tci` 的值复制到 `bp` 指向的内存区域，然后将 `VLAN_TAG_LEN` 的字节数移动到 `bp` 和 `handlep` 指向的内存区域。接着，将 `tag` 指向的 `vlan_tag` 的值和 `vlan_offset` 参数添加到 `pcaphdr` 的 `caplen` 和 `len` 字段中，最后将 `pcaphdr` 移入 `bp`。

函数体中包含三个块：

1. 将 `handlep` 和 `tp_vlan_tpid`、`tp_vlan_tci` 的值复制到 `bp` 指向的内存区域：`
```cpp
bp -= VLAN_TAG_LEN;
memmove(bp, bp + VLAN_TAG_LEN, handlep->vlan_offset);
```
2. 将 `VLAN_TAG_LEN` 的字节数移动到 `bp` 和 `handlep` 指向的内存区域：
```cpp
bp += VLAN_TAG_LEN;
```
3. 将 `tag` 指向的 `vlan_tag` 的值和 `vlan_offset` 参数添加到 `pcaphdr` 的 `caplen` 和 `len` 字段中：
```cpp
pcaphdr.caplen += VLAN_TAG_LEN;
pcaphdr.len += VLAN_TAG_LEN;
```
4. 最后，将 `pcaphdr` 移入 `bp`：
```cpp
pcaphdr.caplen = handle->vlan_offset;
pcaphdr.len = 0;
```
该函数体看起来是一个输出函数，它需要两个输入参数 `handlep` 和 `tp_vlan_tpid`、`tp_vlan_tci`，以及两个可选的输入参数 `snapshot` 和 `vlan_offset`。


```cpp
/* handle a single memory mapped packet */
static int pcap_handle_packet_mmap(
		pcap_t *handle,
		pcap_handler callback,
		u_char *user,
		unsigned char *frame,
		unsigned int tp_len,
		unsigned int tp_mac,
		unsigned int tp_snaplen,
		unsigned int tp_sec,
		unsigned int tp_usec,
		int tp_vlan_tci_valid,
		__u16 tp_vlan_tci,
		__u16 tp_vlan_tpid)
{
	struct pcap_linux *handlep = handle->priv;
	unsigned char *bp;
	struct sockaddr_ll *sll;
	struct pcap_pkthdr pcaphdr;
	pcap_can_socketcan_hdr *canhdr;
	unsigned int snaplen = tp_snaplen;
	struct utsname utsname;

	/* perform sanity check on internal offset. */
	if (tp_mac + tp_snaplen > handle->bufsize) {
		/*
		 * Report some system information as a debugging aid.
		 */
		if (uname(&utsname) != -1) {
			snprintf(handle->errbuf, PCAP_ERRBUF_SIZE,
				"corrupted frame on kernel ring mac "
				"offset %u + caplen %u > frame len %d "
				"(kernel %.32s version %s, machine %.16s)",
				tp_mac, tp_snaplen, handle->bufsize,
				utsname.release, utsname.version,
				utsname.machine);
		} else {
			snprintf(handle->errbuf, PCAP_ERRBUF_SIZE,
				"corrupted frame on kernel ring mac "
				"offset %u + caplen %u > frame len %d",
				tp_mac, tp_snaplen, handle->bufsize);
		}
		return -1;
	}

	/* run filter on received packet
	 * If the kernel filtering is enabled we need to run the
	 * filter until all the frames present into the ring
	 * at filter creation time are processed.
	 * In this case, blocks_to_filter_in_userland is used
	 * as a counter for the packet we need to filter.
	 * Note: alternatively it could be possible to stop applying
	 * the filter when the ring became empty, but it can possibly
	 * happen a lot later... */
	bp = frame + tp_mac;

	/* if required build in place the sll header*/
	sll = (void *)(frame + TPACKET_ALIGN(handlep->tp_hdrlen));
	if (handlep->cooked) {
		if (handle->linktype == DLT_LINUX_SLL2) {
			struct sll2_header *hdrp;

			/*
			 * The kernel should have left us with enough
			 * space for an sll header; back up the packet
			 * data pointer into that space, as that'll be
			 * the beginning of the packet we pass to the
			 * callback.
			 */
			bp -= SLL2_HDR_LEN;

			/*
			 * Let's make sure that's past the end of
			 * the tpacket header, i.e. >=
			 * ((u_char *)thdr + TPACKET_HDRLEN), so we
			 * don't step on the header when we construct
			 * the sll header.
			 */
			if (bp < (u_char *)frame +
					   TPACKET_ALIGN(handlep->tp_hdrlen) +
					   sizeof(struct sockaddr_ll)) {
				snprintf(handle->errbuf, PCAP_ERRBUF_SIZE,
					"cooked-mode frame doesn't have room for sll header");
				return -1;
			}

			/*
			 * OK, that worked; construct the sll header.
			 */
			hdrp = (struct sll2_header *)bp;
			hdrp->sll2_protocol = sll->sll_protocol;
			hdrp->sll2_reserved_mbz = 0;
			hdrp->sll2_if_index = htonl(sll->sll_ifindex);
			hdrp->sll2_hatype = htons(sll->sll_hatype);
			hdrp->sll2_pkttype = sll->sll_pkttype;
			hdrp->sll2_halen = sll->sll_halen;
			memcpy(hdrp->sll2_addr, sll->sll_addr, SLL_ADDRLEN);

			snaplen += sizeof(struct sll2_header);
		} else {
			struct sll_header *hdrp;

			/*
			 * The kernel should have left us with enough
			 * space for an sll header; back up the packet
			 * data pointer into that space, as that'll be
			 * the beginning of the packet we pass to the
			 * callback.
			 */
			bp -= SLL_HDR_LEN;

			/*
			 * Let's make sure that's past the end of
			 * the tpacket header, i.e. >=
			 * ((u_char *)thdr + TPACKET_HDRLEN), so we
			 * don't step on the header when we construct
			 * the sll header.
			 */
			if (bp < (u_char *)frame +
					   TPACKET_ALIGN(handlep->tp_hdrlen) +
					   sizeof(struct sockaddr_ll)) {
				snprintf(handle->errbuf, PCAP_ERRBUF_SIZE,
					"cooked-mode frame doesn't have room for sll header");
				return -1;
			}

			/*
			 * OK, that worked; construct the sll header.
			 */
			hdrp = (struct sll_header *)bp;
			hdrp->sll_pkttype = htons(sll->sll_pkttype);
			hdrp->sll_hatype = htons(sll->sll_hatype);
			hdrp->sll_halen = htons(sll->sll_halen);
			memcpy(hdrp->sll_addr, sll->sll_addr, SLL_ADDRLEN);
			hdrp->sll_protocol = sll->sll_protocol;

			snaplen += sizeof(struct sll_header);
		}
	} else {
		/*
		 * If this is a packet from a CAN device, so that
		 * sll->sll_hatype is ARPHRD_CAN, then, as we're
		 * not capturing in cooked mode, its link-layer
		 * type is DLT_CAN_SOCKETCAN.  Fix up the header
		 * provided by the code below us to match what
		 * DLT_CAN_SOCKETCAN is expected to provide.
		 */
		if (sll->sll_hatype == ARPHRD_CAN) {
			/*
			 * DLT_CAN_SOCKETCAN is specified as having the
			 * CAN ID and flags in network byte order, but
			 * capturing on a CAN device provides it in host
			 * byte order.  Convert it to network byte order.
			 */
			canhdr = (pcap_can_socketcan_hdr *)bp;
			canhdr->can_id = htonl(canhdr->can_id);

			/*
			 * In addition, set the CANFD_FDF flag if
			 * the protocol is LINUX_SLL_P_CANFD, as
			 * the protocol field itself isn't in
			 * the packet to indicate that it's a
			 * CAN FD packet.
			 */
			uint16_t protocol = ntohs(sll->sll_protocol);
			if (protocol == LINUX_SLL_P_CANFD) {
				canhdr->fd_flags |= CANFD_FDF;

				/*
				 * Zero out all the unknown bits in
				 * fd_flags and clear the reserved
				 * fields, so that a program reading
				 * this can assume that CANFD_FDF
				 * is set because we set it, not
				 * because some uninitialized crap
				 * was provided in the fd_flags
				 * field.
				 *
				 * (At least some LINKTYPE_CAN_SOCKETCAN
				 * files attached to Wireshark bugs
				 * had uninitialized junk there, so it
				 * does happen.)
				 *
				 * Update this if Linux adds more flag
				 * bits to the fd_flags field or uses
				 * either of the reserved fields for
				 * FD frames.
				 */
				canhdr->fd_flags &= ~(CANFD_FDF|CANFD_ESI|CANFD_BRS);
				canhdr->reserved1 = 0;
				canhdr->reserved2 = 0;
			} else {
				/*
				 * Clear CANFD_FDF if it's set (probably
				 * again meaning that this field is
				 * uninitialized junk).
				 */
				canhdr->fd_flags &= ~CANFD_FDF;
			}
		}
	}

	if (handlep->filter_in_userland && handle->fcode.bf_insns) {
		struct pcap_bpf_aux_data aux_data;

		aux_data.vlan_tag_present = tp_vlan_tci_valid;
		aux_data.vlan_tag = tp_vlan_tci & 0x0fff;

		if (pcap_filter_with_aux_data(handle->fcode.bf_insns,
					      bp,
					      tp_len,
					      snaplen,
					      &aux_data) == 0)
			return 0;
	}

	if (!linux_check_direction(handle, sll))
		return 0;

	/* get required packet info from ring header */
	pcaphdr.ts.tv_sec = tp_sec;
	pcaphdr.ts.tv_usec = tp_usec;
	pcaphdr.caplen = tp_snaplen;
	pcaphdr.len = tp_len;

	/* if required build in place the sll header*/
	if (handlep->cooked) {
		/* update packet len */
		if (handle->linktype == DLT_LINUX_SLL2) {
			pcaphdr.caplen += SLL2_HDR_LEN;
			pcaphdr.len += SLL2_HDR_LEN;
		} else {
			pcaphdr.caplen += SLL_HDR_LEN;
			pcaphdr.len += SLL_HDR_LEN;
		}
	}

	if (tp_vlan_tci_valid &&
		handlep->vlan_offset != -1 &&
		tp_snaplen >= (unsigned int) handlep->vlan_offset)
	{
		struct vlan_tag *tag;

		/*
		 * Move everything in the header, except the type field,
		 * down VLAN_TAG_LEN bytes, to allow us to insert the
		 * VLAN tag between that stuff and the type field.
		 */
		bp -= VLAN_TAG_LEN;
		memmove(bp, bp + VLAN_TAG_LEN, handlep->vlan_offset);

		/*
		 * Now insert the tag.
		 */
		tag = (struct vlan_tag *)(bp + handlep->vlan_offset);
		tag->vlan_tpid = htons(tp_vlan_tpid);
		tag->vlan_tci = htons(tp_vlan_tci);

		/*
		 * Add the tag to the packet lengths.
		 */
		pcaphdr.caplen += VLAN_TAG_LEN;
		pcaphdr.len += VLAN_TAG_LEN;
	}

	/*
	 * The only way to tell the kernel to cut off the
	 * packet at a snapshot length is with a filter program;
	 * if there's no filter program, the kernel won't cut
	 * the packet off.
	 *
	 * Trim the snapshot length to be no longer than the
	 * specified snapshot length.
	 *
	 * XXX - an alternative is to put a filter, consisting
	 * of a "ret <snaplen>" instruction, on the socket
	 * in the activate routine, so that the truncation is
	 * done in the kernel even if nobody specified a filter;
	 * that means that less buffer space is consumed in
	 * the memory-mapped buffer.
	 */
	if (pcaphdr.caplen > (bpf_u_int32)handle->snapshot)
		pcaphdr.caplen = handle->snapshot;

	/* pass the packet to the user */
	callback(user, &pcaphdr, bp);

	return 1;
}

```

This is a function that deals with the filtering and processing of packets at the physical layer (PHY) of a network. It takes in a packet handle and applies various filters and checks before returning the processed packet count to the kernel.

The function first checks if the packet handle is valid and has the required information. If the handle is valid, the function starts processing the packet by counting the number of bytes in the packet, the timestamp, and the VLAN information.

If the packet is valid and the count is less than 100, the function returns the packet count to the kernel. If the packet is not valid or the count is greater than 100, the function returns PCAP_ERROR_INVALID_PACKET.

If the packet is valid but needs to be processed further, the function returns PCAP_ERROR_FILTER_PACKET. This can occur if the packet is not a valid member of the queue and needs to be filtered. The function then proceeds to process the packet, apply the required filters, and return the processed packet count to the kernel.

If the packet is valid and needs to be processed further, the function returns PCAP_ERROR_FILTER_PACKET. This can occur if the packet is not a valid member of the queue and needs to be filtered. The function then proceeds to process the packet, apply the required filters, and return the processed packet count to the kernel.


```cpp
static int
pcap_read_linux_mmap_v2(pcap_t *handle, int max_packets, pcap_handler callback,
		u_char *user)
{
	struct pcap_linux *handlep = handle->priv;
	union thdr h;
	int pkts = 0;
	int ret;

	/* wait for frames availability.*/
	h.raw = RING_GET_CURRENT_FRAME(handle);
	if (!packet_mmap_acquire(h.h2)) {
		/*
		 * The current frame is owned by the kernel; wait for
		 * a frame to be handed to us.
		 */
		ret = pcap_wait_for_frames_mmap(handle);
		if (ret) {
			return ret;
		}
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

	while (pkts < max_packets) {
		/*
		 * Get the current ring buffer frame, and break if
		 * it's still owned by the kernel.
		 */
		h.raw = RING_GET_CURRENT_FRAME(handle);
		if (!packet_mmap_acquire(h.h2))
			break;

		ret = pcap_handle_packet_mmap(
				handle,
				callback,
				user,
				h.raw,
				h.h2->tp_len,
				h.h2->tp_mac,
				h.h2->tp_snaplen,
				h.h2->tp_sec,
				handle->opt.tstamp_precision == PCAP_TSTAMP_PRECISION_NANO ? h.h2->tp_nsec : h.h2->tp_nsec / 1000,
				VLAN_VALID(h.h2, h.h2),
				h.h2->tp_vlan_tci,
				VLAN_TPID(h.h2, h.h2));
		if (ret == 1) {
			pkts++;
		} else if (ret < 0) {
			return ret;
		}

		/*
		 * Hand this block back to the kernel, and, if we're
		 * counting blocks that need to be filtered in userland
		 * after having been filtered by the kernel, count
		 * the one we've just processed.
		 */
		packet_mmap_release(h.h2);
		if (handlep->blocks_to_filter_in_userland > 0) {
			handlep->blocks_to_filter_in_userland--;
			if (handlep->blocks_to_filter_in_userland == 0) {
				/*
				 * No more blocks need to be filtered
				 * in userland.
				 */
				handlep->filter_in_userland = 0;
			}
		}

		/* next block */
		if (++handle->offset >= handle->cc)
			handle->offset = 0;

		/* check for break loop condition*/
		if (handle->break_loop) {
			handle->break_loop = 0;
			return PCAP_ERROR_BREAK;
		}
	}
	return pkts;
}

```

This is a function definition for `handle_pkts()` in the `net_handle.h` header file of the Linux kernel.

This function receives `handlep` as an argument and handles packets that have been received but not yet processed. It does this by first incrementing the `pkts` counter. If the packet is valid and has not been processed yet, the function adds it to the packet queue and returns the number of packets that have been received. If the packet is not valid or has already been processed, the function sets the `handlep->current_packet` to `NULL` and returns the number of packets that have been received.

The function also handles the case where the kernel has not yet filtered any packets for this handle. If the function counts the blocks that need to be filtered in userland, it releases the packet memory and returns the block count. If the function does not need to filter any blocks in userland and the kernel has not yet requested the packet, the function waits for the next packet and loops back to the beginning. If the function reaches the end of the packet queue without processing any packets, it loops back to the beginning.

The function also includes a block loop condition that checks if the handle has received any packets for a certain period of time (default is 3 seconds). If the loop has not been processed within that time, the function breaks out of the loop and returns `PCAP_ERROR_BREAK`.


```cpp
#ifdef HAVE_TPACKET3
static int
pcap_read_linux_mmap_v3(pcap_t *handle, int max_packets, pcap_handler callback,
		u_char *user)
{
	struct pcap_linux *handlep = handle->priv;
	union thdr h;
	int pkts = 0;
	int ret;

again:
	if (handlep->current_packet == NULL) {
		/* wait for frames availability.*/
		h.raw = RING_GET_CURRENT_FRAME(handle);
		if (!packet_mmap_v3_acquire(h.h3)) {
			/*
			 * The current frame is owned by the kernel; wait
			 * for a frame to be handed to us.
			 */
			ret = pcap_wait_for_frames_mmap(handle);
			if (ret) {
				return ret;
			}
		}
	}
	h.raw = RING_GET_CURRENT_FRAME(handle);
	if (!packet_mmap_v3_acquire(h.h3)) {
		if (pkts == 0 && handlep->timeout == 0) {
			/* Block until we see a packet. */
			goto again;
		}
		return pkts;
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

	while (pkts < max_packets) {
		int packets_to_read;

		if (handlep->current_packet == NULL) {
			h.raw = RING_GET_CURRENT_FRAME(handle);
			if (!packet_mmap_v3_acquire(h.h3))
				break;

			handlep->current_packet = h.raw + h.h3->hdr.bh1.offset_to_first_pkt;
			handlep->packets_left = h.h3->hdr.bh1.num_pkts;
		}
		packets_to_read = handlep->packets_left;

		if (packets_to_read > (max_packets - pkts)) {
			/*
			 * There are more packets in the buffer than
			 * the number of packets we have left to
			 * process to get up to the maximum number
			 * of packets to process.  Only process enough
			 * of them to get us up to that maximum.
			 */
			packets_to_read = max_packets - pkts;
		}

		while (packets_to_read-- && !handle->break_loop) {
			struct tpacket3_hdr* tp3_hdr = (struct tpacket3_hdr*) handlep->current_packet;
			ret = pcap_handle_packet_mmap(
					handle,
					callback,
					user,
					handlep->current_packet,
					tp3_hdr->tp_len,
					tp3_hdr->tp_mac,
					tp3_hdr->tp_snaplen,
					tp3_hdr->tp_sec,
					handle->opt.tstamp_precision == PCAP_TSTAMP_PRECISION_NANO ? tp3_hdr->tp_nsec : tp3_hdr->tp_nsec / 1000,
					VLAN_VALID(tp3_hdr, &tp3_hdr->hv1),
					tp3_hdr->hv1.tp_vlan_tci,
					VLAN_TPID(tp3_hdr, &tp3_hdr->hv1));
			if (ret == 1) {
				pkts++;
			} else if (ret < 0) {
				handlep->current_packet = NULL;
				return ret;
			}
			handlep->current_packet += tp3_hdr->tp_next_offset;
			handlep->packets_left--;
		}

		if (handlep->packets_left <= 0) {
			/*
			 * Hand this block back to the kernel, and, if
			 * we're counting blocks that need to be
			 * filtered in userland after having been
			 * filtered by the kernel, count the one we've
			 * just processed.
			 */
			packet_mmap_v3_release(h.h3);
			if (handlep->blocks_to_filter_in_userland > 0) {
				handlep->blocks_to_filter_in_userland--;
				if (handlep->blocks_to_filter_in_userland == 0) {
					/*
					 * No more blocks need to be filtered
					 * in userland.
					 */
					handlep->filter_in_userland = 0;
				}
			}

			/* next block */
			if (++handle->offset >= handle->cc)
				handle->offset = 0;

			handlep->current_packet = NULL;
		}

		/* check for break loop condition*/
		if (handle->break_loop) {
			handle->break_loop = 0;
			return PCAP_ERROR_BREAK;
		}
	}
	if (pkts == 0 && handlep->timeout == 0) {
		/* Block until we see a packet. */
		goto again;
	}
	return pkts;
}
```

这段代码的作用是将指定的BPF代码附加到数据包捕捉设备（pcap）的过滤器中。它属于pcap库，用于在数据包捕捉设备上应用BPF过滤器。

具体来说，这段代码实现以下功能：

1. 如果传入了 NULL 的参数（即没有数据包捕捉设备可供过滤），则返回 -1，表示无法设置过滤器。
2. 如果传入了要处理的过滤器代码，首先将错误信息存储在 pcap_errbuf 中，然后尝试将过滤器安装到数据包捕捉设备上。如果安装失败，将错误信息打印到 pcap_errbuf 中，并返回 -1。
3. 如果安装了用户级别的过滤器，将过滤器从用户空间复制到内核空间，并将 filter_in_userland 设置为 1。
4. 如果安装了 kernel 级别的过滤器，尝试安装到数据包捕捉设备上。如果可以成功安装，将 filter_in_userland 设置为 0，并将过滤器从用户空间复制到内核空间。


```cpp
#endif /* HAVE_TPACKET3 */

/*
 *  Attach the given BPF code to the packet capture device.
 */
static int
pcap_setfilter_linux(pcap_t *handle, struct bpf_program *filter)
{
	struct pcap_linux *handlep;
	struct sock_fprog	fcode;
	int			can_filter_in_kernel;
	int			err = 0;
	int			n, offset;

	if (!handle)
		return -1;
	if (!filter) {
	        pcap_strlcpy(handle->errbuf, "setfilter: No filter specified",
			PCAP_ERRBUF_SIZE);
		return -1;
	}

	handlep = handle->priv;

	/* Make our private copy of the filter */

	if (install_bpf_program(handle, filter) < 0)
		/* install_bpf_program() filled in errbuf */
		return -1;

	/*
	 * Run user level packet filter by default. Will be overridden if
	 * installing a kernel filter succeeds.
	 */
	handlep->filter_in_userland = 1;

	/* Install kernel level filter if possible */

```

It looks like the code is a Linux kernel module that filters packets based on a specified filter. The filter takes place in the user space and has a specific behavior when multiple threads try to add packets to the same ring.

The code first filters out any blocks that have already been filtered and then walks through the ring, counting the number of blocks that have not yet been filtered. If there are no free blocks, it decrements the count by 1 to prevent a race with another thread that might add packets before the filter was changed.

It then sets the count of blocks worth of packets to filter in user space to the total number of blocks in the ring minus the number of free blocks it found. The filter is then turned on in user space.

The code also includes some additional checks to prevent multiple threads from adding packets to the same ring at the same time. If there are multiple threads trying to add packets, the filter will not modify the settings in user space until all of the threads have finished processing their packets.


```cpp
#ifdef USHRT_MAX
	if (handle->fcode.bf_len > USHRT_MAX) {
		/*
		 * fcode.len is an unsigned short for current kernel.
		 * I have yet to see BPF-Code with that much
		 * instructions but still it is possible. So for the
		 * sake of correctness I added this check.
		 */
		fprintf(stderr, "Warning: Filter too complex for kernel\n");
		fcode.len = 0;
		fcode.filter = NULL;
		can_filter_in_kernel = 0;
	} else
#endif /* USHRT_MAX */
	{
		/*
		 * Oh joy, the Linux kernel uses struct sock_fprog instead
		 * of struct bpf_program and of course the length field is
		 * of different size. Pointed out by Sebastian
		 *
		 * Oh, and we also need to fix it up so that all "ret"
		 * instructions with non-zero operands have MAXIMUM_SNAPLEN
		 * as the operand if we're not capturing in memory-mapped
		 * mode, and so that, if we're in cooked mode, all memory-
		 * reference instructions use special magic offsets in
		 * references to the link-layer header and assume that the
		 * link-layer payload begins at 0; "fix_program()" will do
		 * that.
		 */
		switch (fix_program(handle, &fcode)) {

		case -1:
		default:
			/*
			 * Fatal error; just quit.
			 * (The "default" case shouldn't happen; we
			 * return -1 for that reason.)
			 */
			return -1;

		case 0:
			/*
			 * The program performed checks that we can't make
			 * work in the kernel.
			 */
			can_filter_in_kernel = 0;
			break;

		case 1:
			/*
			 * We have a filter that'll work in the kernel.
			 */
			can_filter_in_kernel = 1;
			break;
		}
	}

	/*
	 * NOTE: at this point, we've set both the "len" and "filter"
	 * fields of "fcode".  As of the 2.6.32.4 kernel, at least,
	 * those are the only members of the "sock_fprog" structure,
	 * so we initialize every member of that structure.
	 *
	 * If there is anything in "fcode" that is not initialized,
	 * it is either a field added in a later kernel, or it's
	 * padding.
	 *
	 * If a new field is added, this code needs to be updated
	 * to set it correctly.
	 *
	 * If there are no other fields, then:
	 *
	 *	if the Linux kernel looks at the padding, it's
	 *	buggy;
	 *
	 *	if the Linux kernel doesn't look at the padding,
	 *	then if some tool complains that we're passing
	 *	uninitialized data to the kernel, then the tool
	 *	is buggy and needs to understand that it's just
	 *	padding.
	 */
	if (can_filter_in_kernel) {
		if ((err = set_kernel_filter(handle, &fcode)) == 0)
		{
			/*
			 * Installation succeeded - using kernel filter,
			 * so userland filtering not needed.
			 */
			handlep->filter_in_userland = 0;
		}
		else if (err == -1)	/* Non-fatal error */
		{
			/*
			 * Print a warning if we weren't able to install
			 * the filter for a reason other than "this kernel
			 * isn't configured to support socket filters.
			 */
			if (errno == ENOMEM) {
				/*
				 * Either a kernel memory allocation
				 * failure occurred, or there's too
				 * much "other/option memory" allocated
				 * for this socket.  Suggest that they
				 * increase the "other/option memory"
				 * limit.
				 */
				fprintf(stderr,
				    "Warning: Couldn't allocate kernel memory for filter: try increasing net.core.optmem_max with sysctl\n");
			} else if (errno != ENOPROTOOPT && errno != EOPNOTSUPP) {
				fprintf(stderr,
				    "Warning: Kernel filter failed: %s\n",
					pcap_strerror(errno));
			}
		}
	}

	/*
	 * If we're not using the kernel filter, get rid of any kernel
	 * filter that might've been there before, e.g. because the
	 * previous filter could work in the kernel, or because some other
	 * code attached a filter to the socket by some means other than
	 * calling "pcap_setfilter()".  Otherwise, the kernel filter may
	 * filter out packets that would pass the new userland filter.
	 */
	if (handlep->filter_in_userland) {
		if (reset_kernel_filter(handle) == -1) {
			pcap_fmt_errmsg_for_errno(handle->errbuf,
			    PCAP_ERRBUF_SIZE, errno,
			    "can't remove kernel filter");
			err = -2;	/* fatal error */
		}
	}

	/*
	 * Free up the copy of the filter that was made by "fix_program()".
	 */
	if (fcode.filter != NULL)
		free(fcode.filter);

	if (err == -2)
		/* Fatal error */
		return -1;

	/*
	 * If we're filtering in userland, there's nothing to do;
	 * the new filter will be used for the next packet.
	 */
	if (handlep->filter_in_userland)
		return 0;

	/*
	 * We're filtering in the kernel; the packets present in
	 * all blocks currently in the ring were already filtered
	 * by the old filter, and so will need to be filtered in
	 * userland by the new filter.
	 *
	 * Get an upper bound for the number of such blocks; first,
	 * walk the ring backward and count the free blocks.
	 */
	offset = handle->offset;
	if (--offset < 0)
		offset = handle->cc - 1;
	for (n=0; n < handle->cc; ++n) {
		if (--offset < 0)
			offset = handle->cc - 1;
		if (pcap_get_ring_frame_status(handle, offset) != TP_STATUS_KERNEL)
			break;
	}

	/*
	 * If we found free blocks, decrement the count of free
	 * blocks by 1, just in case we lost a race with another
	 * thread of control that was adding a packet while
	 * we were counting and that had run the filter before
	 * we changed it.
	 *
	 * XXX - could there be more than one block added in
	 * this fashion?
	 *
	 * XXX - is there a way to avoid that race, e.g. somehow
	 * wait for all packets that passed the old filter to
	 * be added to the ring?
	 */
	if (n != 0)
		n--;

	/*
	 * Set the count of blocks worth of packets to filter
	 * in userland to the total number of blocks in the
	 * ring minus the number of free blocks we found, and
	 * turn on userland filtering.  (The count of blocks
	 * worth of packets to filter in userland is guaranteed
	 * not to be zero - n, above, couldn't be set to a
	 * value > handle->cc, and if it were equal to
	 * handle->cc, it wouldn't be zero, and thus would
	 * be decremented to handle->cc - 1.)
	 */
	handlep->blocks_to_filter_in_userland = handle->cc - n;
	handlep->filter_in_userland = 1;

	return 0;
}

```

这段代码定义了一个名为 `iface_get_id` 的函数，它接收一个文件描述符（int fd）、一个设备名称（const char *device）和一个字符数组 `ebuf`，并返回该设备的索引号（int）。

函数首先定义了一个 `ifreq` 类型的变量 `ifr`，用于存储输入信息。然后，用 `pcap_strlcpy` 函数将设备名称存储到 `ifr.ifr_name` 字段中。

接着，函数调用 `ioctl` 函数的 `SIOCGIFINDEX` 接口，获取输入信息，存储到 `ifr.ifr_ifindex` 字段中。

最后，函数使用 `pcap_fmt_errmsg_for_errno` 函数将错误信息格式化并打印到 `ebuf` 数组中，然后返回 -1，表明失败。


```cpp
/*
 *  Return the index of the given device name. Fill ebuf and return
 *  -1 on failure.
 */
static int
iface_get_id(int fd, const char *device, char *ebuf)
{
	struct ifreq	ifr;

	memset(&ifr, 0, sizeof(ifr));
	pcap_strlcpy(ifr.ifr_name, device, sizeof(ifr.ifr_name));

	if (ioctl(fd, SIOCGIFINDEX, &ifr) == -1) {
		pcap_fmt_errmsg_for_errno(ebuf, PCAP_ERRBUF_SIZE,
		    errno, "SIOCGIFINDEX");
		return -1;
	}

	return ifr.ifr_ifindex;
}

```

The code you provided is a Linux kernel function that receives an error message from a network interface and returns a "network down" indication, or a "network up" indication, depending on whether the interface is up or down.

The function first checks if the interface is up by checking the return value of `pcap_gcloud_bind()`. If the return value is an error code, the function formats the error message using `pcap_fmt_errmsg_for_errno()` and returns the error code.

If the interface is down, the function returns `PCAP_ERROR_IFACE_NOT_UP`. If the interface is up but there is an error, the function returns `PCAP_ERROR`.

The function also checks for the error code `ENODEV` for `errno == ENODEV` . In this case, the function will not format the error message and will return `PCAP_ERROR_NO_SUCH_DEVICE`.

The last two lines of the function are for the pending errors, if any. If there are any errors, the function returns `PCAP_ERROR`.


```cpp
/*
 *  Bind the socket associated with FD to the given device.
 *  Return 0 on success or a PCAP_ERROR_ value on a hard error.
 */
static int
iface_bind(int fd, int ifindex, char *ebuf, int protocol)
{
	struct sockaddr_ll	sll;
	int			ret, err;
	socklen_t		errlen = sizeof(err);

	memset(&sll, 0, sizeof(sll));
	sll.sll_family		= AF_PACKET;
	sll.sll_ifindex		= ifindex < 0 ? 0 : ifindex;
	sll.sll_protocol	= protocol;

	if (bind(fd, (struct sockaddr *) &sll, sizeof(sll)) == -1) {
		if (errno == ENETDOWN) {
			/*
			 * Return a "network down" indication, so that
			 * the application can report that rather than
			 * saying we had a mysterious failure and
			 * suggest that they report a problem to the
			 * libpcap developers.
			 */
			return PCAP_ERROR_IFACE_NOT_UP;
		}
		if (errno == ENODEV) {
			/*
			 * There's nothing more to say, so clear the
			 * error message.
			 */
			ebuf[0] = '\0';
			ret = PCAP_ERROR_NO_SUCH_DEVICE;
		} else {
			ret = PCAP_ERROR;
			pcap_fmt_errmsg_for_errno(ebuf, PCAP_ERRBUF_SIZE,
			    errno, "bind");
		}
		return ret;
	}

	/* Any pending errors, e.g., network is down? */

	if (getsockopt(fd, SOL_SOCKET, SO_ERROR, &err, &errlen) == -1) {
		pcap_fmt_errmsg_for_errno(ebuf, PCAP_ERRBUF_SIZE,
		    errno, "getsockopt (SO_ERROR)");
		return PCAP_ERROR;
	}

	if (err == ENETDOWN) {
		/*
		 * Return a "network down" indication, so that
		 * the application can report that rather than
		 * saying we had a mysterious failure and
		 * suggest that they report a problem to the
		 * libpcap developers.
		 */
		return PCAP_ERROR_IFACE_NOT_UP;
	} else if (err > 0) {
		pcap_fmt_errmsg_for_errno(ebuf, PCAP_ERRBUF_SIZE,
		    err, "bind");
		return PCAP_ERROR;
	}

	return 0;
}

```

Please find the solution for the problem at the following link : <https://github.com/username/revert-linux-code/issues/2574>


```cpp
/*
 * Try to enter monitor mode.
 * If we have libnl, try to create a new monitor-mode device and
 * capture on that; otherwise, just say "not supported".
 */
#ifdef HAVE_LIBNL
static int
enter_rfmon_mode(pcap_t *handle, int sock_fd, const char *device)
{
	struct pcap_linux *handlep = handle->priv;
	int ret;
	char phydev_path[PATH_MAX+1];
	struct nl80211_state nlstate;
	struct ifreq ifr;
	u_int n;

	/*
	 * Is this a mac80211 device?
	 */
	ret = get_mac80211_phydev(handle, device, phydev_path, PATH_MAX);
	if (ret < 0)
		return ret;	/* error */
	if (ret == 0)
		return 0;	/* no error, but not mac80211 device */

	/*
	 * XXX - is this already a monN device?
	 * If so, we're done.
	 */

	/*
	 * OK, it's apparently a mac80211 device.
	 * Try to find an unused monN device for it.
	 */
	ret = nl80211_init(handle, &nlstate, device);
	if (ret != 0)
		return ret;
	for (n = 0; n < UINT_MAX; n++) {
		/*
		 * Try mon{n}.
		 */
		char mondevice[3+10+1];	/* mon{UINT_MAX}\0 */

		snprintf(mondevice, sizeof mondevice, "mon%u", n);
		ret = add_mon_if(handle, sock_fd, &nlstate, device, mondevice);
		if (ret == 1) {
			/*
			 * Success.  We don't clean up the libnl state
			 * yet, as we'll be using it later.
			 */
			goto added;
		}
		if (ret < 0) {
			/*
			 * Hard failure.  Just return ret; handle->errbuf
			 * has already been set.
			 */
			nl80211_cleanup(&nlstate);
			return ret;
		}
	}

	snprintf(handle->errbuf, PCAP_ERRBUF_SIZE,
	    "%s: No free monN interfaces", device);
	nl80211_cleanup(&nlstate);
	return PCAP_ERROR;

```

This function appears to be responsible for setting up a network interface (device) and configuring it for use with a PCAP-enabled network driver.

It first gets the device name from the handle of the network interface, and then sets the interface flags to the current flags (FF, not in use, running, up). It then sets the name of the network interface to the handle's device name and sets the flags for the interface.

If there are any errors, it formats the error message and returns the error code. It also cleans up the NL80211 library and returns the updated error code.

Finally, it adds the network interface to the list of PCAP-enabled interfaces to close when the PCAP-enabled network driver is closed.


```cpp
added:

#if 0
	/*
	 * Sleep for .1 seconds.
	 */
	delay.tv_sec = 0;
	delay.tv_nsec = 500000000;
	nanosleep(&delay, NULL);
#endif

	/*
	 * If we haven't already done so, arrange to have
	 * "pcap_close_all()" called when we exit.
	 */
	if (!pcap_do_addexit(handle)) {
		/*
		 * "atexit()" failed; don't put the interface
		 * in rfmon mode, just give up.
		 */
		del_mon_if(handle, sock_fd, &nlstate, device,
		    handlep->mondevice);
		nl80211_cleanup(&nlstate);
		return PCAP_ERROR;
	}

	/*
	 * Now configure the monitor interface up.
	 */
	memset(&ifr, 0, sizeof(ifr));
	pcap_strlcpy(ifr.ifr_name, handlep->mondevice, sizeof(ifr.ifr_name));
	if (ioctl(sock_fd, SIOCGIFFLAGS, &ifr) == -1) {
		pcap_fmt_errmsg_for_errno(handle->errbuf, PCAP_ERRBUF_SIZE,
		    errno, "%s: Can't get flags for %s", device,
		    handlep->mondevice);
		del_mon_if(handle, sock_fd, &nlstate, device,
		    handlep->mondevice);
		nl80211_cleanup(&nlstate);
		return PCAP_ERROR;
	}
	ifr.ifr_flags |= IFF_UP|IFF_RUNNING;
	if (ioctl(sock_fd, SIOCSIFFLAGS, &ifr) == -1) {
		pcap_fmt_errmsg_for_errno(handle->errbuf, PCAP_ERRBUF_SIZE,
		    errno, "%s: Can't set flags for %s", device,
		    handlep->mondevice);
		del_mon_if(handle, sock_fd, &nlstate, device,
		    handlep->mondevice);
		nl80211_cleanup(&nlstate);
		return PCAP_ERROR;
	}

	/*
	 * Success.  Clean up the libnl state.
	 */
	nl80211_cleanup(&nlstate);

	/*
	 * Note that we have to delete the monitor device when we close
	 * the handle.
	 */
	handlep->must_do_on_close |= MUST_DELETE_MONIF;

	/*
	 * Add this to the list of pcaps to close when we exit.
	 */
	pcap_add_to_pcaps_to_close(handle);

	return 1;
}
```

这段代码是一个C语言函数，它实现了对于Linux系统中的SOF_TIMESTAMPING_和PCAP_TSTAMP_格式的数据格式的互相转换。

具体来说，这段代码分为两个部分。第一部分是当系统中有libnl库时，函数enter_rfmon_mode的作用是进入 Monitoring 模式，如果libnl库不存在，函数返回0。第二部分是当系统中有Linux系统中的网络数据包捕获工具链（如tcpdump等）时，函数enter_rfmon_mode的作用是将SOF_TIMESTAMPING_格式的数据转换为PCAP_TSTAMP_格式的数据，以便于使用pcap_parse()函数进行数据包抓取。

这里主要涉及到了两个头文件：<linux/net/timestamp.h>和<linux/timers.h>，同时还需要包含<errno.h>和<sys/types.h>等头文件。


```cpp
#else /* HAVE_LIBNL */
static int
enter_rfmon_mode(pcap_t *handle _U_, int sock_fd _U_, const char *device _U_)
{
	/*
	 * We don't have libnl, so we can't do monitor mode.
	 */
	return 0;
}
#endif /* HAVE_LIBNL */

#if defined(HAVE_LINUX_NET_TSTAMP_H) && defined(PACKET_TIMESTAMP)
/*
 * Map SOF_TIMESTAMPING_ values to PCAP_TSTAMP_ values.
 */
```

这段代码定义了一个名为 `sof_ts_type_map` 的结构体数组，用于存储不同种类的timestamping类型以及相应的pcap timestamp类型。该数组长度为 `3`。

该数组的每个元素都包含两个整型变量，一个表示timestamping类型，一个表示与之相对应的timestamp类型。其中，timestamping类型包括SOF_TIMESTAMPING_SOFTWARE、SOF_TIMESTAMPING_SYS_HARDWARE和SOF_TIMESTAMPING_RAW_HARDWARE三种。

该函数 `iface_set_all_ts_types` 接受一个 `pcap_t` 类型的输入参数 `handle`，以及一个字符型缓冲区 `ebuf`。该函数的作用是设置handle对象中时间戳类型列表和时间戳类型计数。函数的实现包括以下几步：

1. 在handle对象中创建一个大小为 `NUM_SOF_TIMESTAMPING_TYPES` 的字符型缓冲区 `tstamp_type_list`，用于存储所有timestamping类型。
2. 遍历 `NUM_SOF_TIMESTAMPING_TYPES` 次，将每种timestamping类型存储在`tamp_type_list`中的对应位置。
3. 将`tamp_type_list`中所有位置存储的数量存储在handle对象的`tstamp_type_count` 变量中。
4. 返回0，表示设置成功。


```cpp
static const struct {
	int soft_timestamping_val;
	int pcap_tstamp_val;
} sof_ts_type_map[3] = {
	{ SOF_TIMESTAMPING_SOFTWARE, PCAP_TSTAMP_HOST },
	{ SOF_TIMESTAMPING_SYS_HARDWARE, PCAP_TSTAMP_ADAPTER },
	{ SOF_TIMESTAMPING_RAW_HARDWARE, PCAP_TSTAMP_ADAPTER_UNSYNCED }
};
#define NUM_SOF_TIMESTAMPING_TYPES	(sizeof sof_ts_type_map / sizeof sof_ts_type_map[0])

/*
 * Set the list of time stamping types to include all types.
 */
static int
iface_set_all_ts_types(pcap_t *handle, char *ebuf)
{
	u_int i;

	handle->tstamp_type_list = malloc(NUM_SOF_TIMESTAMPING_TYPES * sizeof(u_int));
	if (handle->tstamp_type_list == NULL) {
		pcap_fmt_errmsg_for_errno(ebuf, PCAP_ERRBUF_SIZE,
		    errno, "malloc");
		return -1;
	}
	for (i = 0; i < NUM_SOF_TIMESTAMPING_TYPES; i++)
		handle->tstamp_type_list[i] = sof_ts_type_map[i].pcap_tstamp_val;
	handle->tstamp_type_count = NUM_SOF_TIMESTAMPING_TYPES;
	return 0;
}

```

This is a code snippet for an implementation of the Windows Header Tips and Stamp (WHTSTAMP) filter. It appears to check if the platform supports the filter and, if it does, it may either report all timestamping types or only report a few PTP packets. The code also appears to handle the case where the filter is not supported.

The code defines a single variable, `handle`, which is the handle for the packet filter, and a variable, `info`, which is the information about the packet filter.

The next lines of code initialize the variable, `num_ts_types`, to zero and set the variable, `handle->tstamp_type_list`, to a null pointer.

The next section of the code reads through the information about the packet filter and attempts to determine the number of timestamping types that the platform supports. If the platform supports timestamping, the code reads through the information about the timestamping types and stores the variable, `num_ts_types`, in the `handle->tstamp_type_list` pointer.

If the platform does not support timestamping, the code sets the `handle->tstamp_type_list` pointer to a caller-provided pointer to a NULL pointer.

The last section of the code sets the variable, `handle->tstamp_type_count`, to the value of `num_ts_types`, if the platform supports timestamping, or to a caller-provided pointer to the `handle->tstamp_type_list` pointer if the platform does not support timestamping.

The code also includes a function, `get_timestamping_types`, which appears to return a pointer to a数组 of timestamping types. It is not clear from the provided code how this function is used.


```cpp
/*
 * Get a list of time stamp types.
 */
#ifdef ETHTOOL_GET_TS_INFO
static int
iface_get_ts_types(const char *device, pcap_t *handle, char *ebuf)
{
	int fd;
	struct ifreq ifr;
	struct ethtool_ts_info info;
	int num_ts_types;
	u_int i, j;

	/*
	 * This doesn't apply to the "any" device; you can't say "turn on
	 * hardware time stamping for all devices that exist now and arrange
	 * that it be turned on for any device that appears in the future",
	 * and not all devices even necessarily *support* hardware time
	 * stamping, so don't report any time stamp types.
	 */
	if (strcmp(device, "any") == 0) {
		handle->tstamp_type_list = NULL;
		return 0;
	}

	/*
	 * Create a socket from which to fetch time stamping capabilities.
	 */
	fd = get_if_ioctl_socket();
	if (fd < 0) {
		pcap_fmt_errmsg_for_errno(ebuf, PCAP_ERRBUF_SIZE,
		    errno, "socket for SIOCETHTOOL(ETHTOOL_GET_TS_INFO)");
		return -1;
	}

	memset(&ifr, 0, sizeof(ifr));
	pcap_strlcpy(ifr.ifr_name, device, sizeof(ifr.ifr_name));
	memset(&info, 0, sizeof(info));
	info.cmd = ETHTOOL_GET_TS_INFO;
	ifr.ifr_data = (caddr_t)&info;
	if (ioctl(fd, SIOCETHTOOL, &ifr) == -1) {
		int save_errno = errno;

		close(fd);
		switch (save_errno) {

		case EOPNOTSUPP:
		case EINVAL:
			/*
			 * OK, this OS version or driver doesn't support
			 * asking for the time stamping types, so let's
			 * just return all the possible types.
			 */
			if (iface_set_all_ts_types(handle, ebuf) == -1)
				return -1;
			return 0;

		case ENODEV:
			/*
			 * OK, no such device.
			 * The user will find that out when they try to
			 * activate the device; just return an empty
			 * list of time stamp types.
			 */
			handle->tstamp_type_list = NULL;
			return 0;

		default:
			/*
			 * Other error.
			 */
			pcap_fmt_errmsg_for_errno(ebuf, PCAP_ERRBUF_SIZE,
			    save_errno,
			    "%s: SIOCETHTOOL(ETHTOOL_GET_TS_INFO) ioctl failed",
			    device);
			return -1;
		}
	}
	close(fd);

	/*
	 * Do we support hardware time stamping of *all* packets?
	 */
	if (!(info.rx_filters & (1 << HWTSTAMP_FILTER_ALL))) {
		/*
		 * No, so don't report any time stamp types.
		 *
		 * XXX - some devices either don't report
		 * HWTSTAMP_FILTER_ALL when they do support it, or
		 * report HWTSTAMP_FILTER_ALL but map it to only
		 * time stamping a few PTP packets.  See
		 * http://marc.info/?l=linux-netdev&m=146318183529571&w=2
		 *
		 * Maybe that got fixed later.
		 */
		handle->tstamp_type_list = NULL;
		return 0;
	}

	num_ts_types = 0;
	for (i = 0; i < NUM_SOF_TIMESTAMPING_TYPES; i++) {
		if (info.so_timestamping & sof_ts_type_map[i].soft_timestamping_val)
			num_ts_types++;
	}
	if (num_ts_types != 0) {
		handle->tstamp_type_list = malloc(num_ts_types * sizeof(u_int));
		if (handle->tstamp_type_list == NULL) {
			pcap_fmt_errmsg_for_errno(ebuf, PCAP_ERRBUF_SIZE,
			    errno, "malloc");
			return -1;
		}
		for (i = 0, j = 0; i < NUM_SOF_TIMESTAMPING_TYPES; i++) {
			if (info.so_timestamping & sof_ts_type_map[i].soft_timestamping_val) {
				handle->tstamp_type_list[j] = sof_ts_type_map[i].pcap_tstamp_val;
				j++;
			}
		}
		handle->tstamp_type_count = num_ts_types;
	} else
		handle->tstamp_type_list = NULL;

	return 0;
}
```

这段代码是一个iface_get_ts_types函数，它的作用是获取控制台设备（device）类型为"any"的ts（time stamp）类型信息。在函数中，首先检查设备是否为"any"，如果是，则返回离开发区外的 NULL。然后尝试使用iface_set_all_ts_types函数获取设备所有支持的ts类型，如果这个函数失败，则返回-1。最后，根据设备类型来设置ts类型列表。


```cpp
#else /* ETHTOOL_GET_TS_INFO */
static int
iface_get_ts_types(const char *device, pcap_t *handle, char *ebuf)
{
	/*
	 * This doesn't apply to the "any" device; you can't say "turn on
	 * hardware time stamping for all devices that exist now and arrange
	 * that it be turned on for any device that appears in the future",
	 * and not all devices even necessarily *support* hardware time
	 * stamping, so don't report any time stamp types.
	 */
	if (strcmp(device, "any") == 0) {
		handle->tstamp_type_list = NULL;
		return 0;
	}

	/*
	 * We don't have an ioctl to use to ask what's supported,
	 * so say we support everything.
	 */
	if (iface_set_all_ts_types(handle, ebuf) == -1)
		return -1;
	return 0;
}
```

这段代码是用来检查支持哪些时间戳类型输出的。首先，通过条件判断是否定义了`HAVE_LINUX_NET_TSTAMP_H`和`PACKET_TIMESTAMP`，如果是，则执行以下函数：

```cppc
static int
iface_get_ts_types(const char *device, pcap_t *p, char *ebuf)
{
   return 0; // 成功，说明设备支持时间戳类型
}
```

如果不定义上述条件，或者定义了但输出类型不支持时间戳类型，则返回`-1`。

接下来，通过`SIOCETHTOOL`函数检查是否支持某种特定的时间戳类型。我们以`ETHTOOL_GUFO`为例，如果支持，则执行以下操作：

```cppc
static int
iface_get_ts_types(const char *device, pcap_t *p, char *ebuf)
{
   int ret = 0; // 成功，说明设备支持时间戳类型

   if (SIOCETHTOOL(device, &ret, ETHTOOL_GUFO) == 0)
   {
       if (ret == -1)
       {
           tcprintf_error(p->tc, "This is not even supported");
           return -1;
       }
   }

   return ret; // 支持，返回成功返回值
}
```

如果`SIOCETHTOOL`函数返回`-1`，我们继续检查定义是否支持其他时间戳类型。如果没有定义任何条件，或者定义了但输出类型不支持时间戳类型，则输出错误信息并返回`-1`。


```cpp
#endif /* ETHTOOL_GET_TS_INFO */
#else  /* defined(HAVE_LINUX_NET_TSTAMP_H) && defined(PACKET_TIMESTAMP) */
static int
iface_get_ts_types(const char *device _U_, pcap_t *p _U_, char *ebuf _U_)
{
	/*
	 * Nothing to fetch, so it always "succeeds".
	 */
	return 0;
}
#endif /* defined(HAVE_LINUX_NET_TSTAMP_H) && defined(PACKET_TIMESTAMP) */

/*
 * Find out if we have any form of fragmentation/reassembly offloading.
 *
 * We do so using SIOCETHTOOL checking for various types of offloading;
 * if SIOCETHTOOL isn't defined, or we don't have any #defines for any
 * of the types of offloading, there's nothing we can do to check, so
 * we just say "no, we don't".
 *
 * We treat EOPNOTSUPP, EINVAL and, if eperm_ok is true, EPERM as
 * indications that the operation isn't supported.  We do EPERM
 * weirdly because the SIOCETHTOOL code in later kernels 1) doesn't
 * support ETHTOOL_GUFO, 2) also doesn't include it in the list
 * of ethtool operations that don't require CAP_NET_ADMIN privileges,
 * and 3) does the "is this permitted" check before doing the "is
 * this even supported" check, so it fails with "this is not permitted"
 * rather than "this is not even supported".  To work around this
 * annoyance, we only treat EPERM as an error for the first feature,
 * and assume that they all do the same permission checks, so if the
 * first one is allowed all the others are allowed if supported.
 */
```

这段代码是一个用于 Linux 操作系统中的网络设备驱动程序的函数，它的作用是处理通过 `ethtool` 命令发送的配置选项。具体的，如果定义了 `SIOCETHTOOL`，并且 `ETHTOOL_GTSO`，`ETHTOOL_GUFO`，`ETHTOOL_GGSO` 或 `ETHTOOL_GFLAGS`，`ETHTOOL_GGRO` 中的任意一个，函数将返回一个整数，表示 `ethtool` 命令的执行结果。如果命令的参数或选项无效，函数将返回一个负数。


```cpp
#if defined(SIOCETHTOOL) && (defined(ETHTOOL_GTSO) || defined(ETHTOOL_GUFO) || defined(ETHTOOL_GGSO) || defined(ETHTOOL_GFLAGS) || defined(ETHTOOL_GGRO))
static int
iface_ethtool_flag_ioctl(pcap_t *handle, int cmd, const char *cmdname,
    int eperm_ok)
{
	struct ifreq	ifr;
	struct ethtool_value eval;

	memset(&ifr, 0, sizeof(ifr));
	pcap_strlcpy(ifr.ifr_name, handle->opt.device, sizeof(ifr.ifr_name));
	eval.cmd = cmd;
	eval.data = 0;
	ifr.ifr_data = (caddr_t)&eval;
	if (ioctl(handle->fd, SIOCETHTOOL, &ifr) == -1) {
		if (errno == EOPNOTSUPP || errno == EINVAL ||
		    (errno == EPERM && eperm_ok)) {
			/*
			 * OK, let's just return 0, which, in our
			 * case, either means "no, what we're asking
			 * about is not enabled" or "all the flags
			 * are clear (i.e., nothing is enabled)".
			 */
			return 0;
		}
		pcap_fmt_errmsg_for_errno(handle->errbuf, PCAP_ERRBUF_SIZE,
		    errno, "%s: SIOCETHTOOL(%s) ioctl failed",
		    handle->opt.device, cmdname);
		return -1;
	}
	return eval.data;
}

```

这段代码定义了一个名为XXX的结构体，用于存储操作系统中网络设备（如网卡）的offloading信息。

在这段注释中，开发者指出尽管我们需要检查设备是否要进行offloading，但至少我们需要检查设备是否支持具体的offloading类型。

接下来的部分指出，由于需要支持新类型的offloading，但新类型的offloading可能并不总是可以进行检查的，尤其是在使用ETHTOOL_GFEATURES等ETHTOOL命令时。

开发者表示，实际上，我们无法确定设备支持哪些offloading类型，以及在进行offloading时，设备可能会遇到的问题。

因此，该代码的主要目的是让开发人员知道需要检查哪些offloading信息，以及设备在支持新类型的offloading时可能会遇到的问题。


```cpp
/*
 * XXX - it's annoying that we have to check for offloading at all, but,
 * given that we have to, it's still annoying that we have to check for
 * particular types of offloading, especially that shiny new types of
 * offloading may be added - and, worse, may not be checkable with
 * a particular ETHTOOL_ operation; ETHTOOL_GFEATURES would, in
 * theory, give those to you, but the actual flags being used are
 * opaque (defined in a non-uapi header), and there doesn't seem to
 * be any obvious way to ask the kernel what all the offloading flags
 * are - at best, you can ask for a set of strings(!) to get *names*
 * for various flags.  (That whole mechanism appears to have been
 * designed for the sole purpose of letting ethtool report flags
 * by name and set flags by name, with the names having no semantics
 * ethtool understands.)
 */
```

这段代码是一个用于在名为“iface_get_offload”的函数中设置或获取“iface_ethtool_flag_ioctl”函数输出的静态函数。

函数接受一个名为“handle”的“pcap_t”类型的指针，代表一个网络接口的文件句柄。函数内部通过调用“iface_ethtool_flag_ioctl”函数来进行设置或获取操作。通过“handle”指针调用“iface_ethtool_flag_ioctl”函数可以实现以下操作：

1. 设置“iface_ethtool_flag_ioctl”函数的第一个参数“handle”为0，表示设置为“被动模式”，不做任何其他操作。
2. 如果“iface_ethtool_flag_ioctl”函数的返回值为负数，表示设置操作失败，返回-1。
3. 如果“iface_ethtool_flag_ioctl”函数的返回值为正数，表示设置操作成功。具体根据设置的标志类型进行判断，比如“ETHTOOL_GTSO”表示启用GTSO offloading,“ETHTOOL_GGSO”表示启用generic segmentation offloading。根据这些标志，函数会执行相应的操作。

该函数的作用是用于在网络接口文件句柄上设置或获取“iface_ethtool_flag_ioctl”函数的输出，以实现某些网络技术的offloading。


```cpp
static int
iface_get_offload(pcap_t *handle)
{
	int ret;

#ifdef ETHTOOL_GTSO
	ret = iface_ethtool_flag_ioctl(handle, ETHTOOL_GTSO, "ETHTOOL_GTSO", 0);
	if (ret == -1)
		return -1;
	if (ret)
		return 1;	/* TCP segmentation offloading on */
#endif

#ifdef ETHTOOL_GGSO
	/*
	 * XXX - will this cause large unsegmented packets to be
	 * handed to PF_PACKET sockets on transmission?  If not,
	 * this need not be checked.
	 */
	ret = iface_ethtool_flag_ioctl(handle, ETHTOOL_GGSO, "ETHTOOL_GGSO", 0);
	if (ret == -1)
		return -1;
	if (ret)
		return 1;	/* generic segmentation offloading on */
```

这段代码是 Linux 操作系统中的 iface_ethtool_flag_ioctl() 函数的一部分。该函数用于通过用户空间接口（IFP）控制以太网设备的操作。

具体来说，这段代码的作用是检查两个以太网接口是否支持某些选项，并根据检查结果返回不同的状态码。

第一个检查点（ETFTH_GFLAGS）是一个选项，用于指示是否启用大的接收负载。如果启用，该选项将使系统能够向发送者发送大量的接收数据，从而导致接收端存在大量数据，可能需要进行大量的重新组装。

第二个检查点（ETHTOOL_GGRO）是一个选项，用于指示是否启用通用接收负载。如果启用，该选项将通过 iface_ethtool_flag_ioctl() 函数将 iface 设置为“活动”，从而启用接收负载，但需要提醒开发人员需要处理此选项。

在这段代码中，首先通过条件语句检查这两个选项是否都启用。如果两个选项都启用，那么系统将返回一个带有状态码 0 的成功状态码，否则将返回一个带有状态码 -1 的失败状态码。如果只有选项 ETHTOOL_GFLAGS 启用，那么将返回状态码 1，表示系统无法支持大的接收负载。如果只有选项 ETHTOOL_GGRO 启用，那么将返回状态码 1，表示系统无法支持通用接收负载。


```cpp
#endif

#ifdef ETHTOOL_GFLAGS
	ret = iface_ethtool_flag_ioctl(handle, ETHTOOL_GFLAGS, "ETHTOOL_GFLAGS", 0);
	if (ret == -1)
		return -1;
	if (ret & ETH_FLAG_LRO)
		return 1;	/* large receive offloading on */
#endif

#ifdef ETHTOOL_GGRO
	/*
	 * XXX - will this cause large reassembled packets to be
	 * handed to PF_PACKET sockets on receipt?  If not,
	 * this need not be checked.
	 */
	ret = iface_ethtool_flag_ioctl(handle, ETHTOOL_GGRO, "ETHTOOL_GGRO", 0);
	if (ret == -1)
		return -1;
	if (ret)
		return 1;	/* generic (large) receive offloading on */
```

这段代码是一个 iface_ethtool_flag_ioctl() 函数中的一条声明，该函数用于在 Linux kernel 2.6 和更高版本中，通过 `ethtool` 工具打印出 iface_ethtool_flag_ioctl() 函数的返回值，具体作用如下：

1. 在 Linux 2.6 和更高版本中，通过 `ethtool` 工具打印出 `ethtool_gufu` 标志的值，值可以是 0 或 1。

2. 如果当前版本的 Linux 不支持 `ethtool_gufu`，则该函数会尝试使用 `ethtool_gufu` 标志，并尝试打印出 `ethtool_gufu=1` 的值。

3. 如果通过 `ethtool` 工具打印出的返回值为 -1，则表示当前版本不支持 `ethtool_gufu` 标志，函数会返回 -1。

4. 如果通过 `ethtool` 工具打印出的返回值为 1，则表示当前版本支持 `ethtool_gufu` 标志，函数不会执行特别的操作。

5. 如果通过 `ethtool` 工具打印出的返回值为 `ethtool_gufu=1` 且 `ethtool_create_context()` 函数成功创建了 `ethtool_gufu` 上下文，则表示当前版本成功支持 `ethtool_gufu` 标志，函数返回 0。


```cpp
#endif

#ifdef ETHTOOL_GUFO
	/*
	 * Do this one last, as support for it was removed in later
	 * kernels, and it fails with EPERM on those kernels rather
	 * than with EOPNOTSUPP (see explanation in comment for
	 * iface_ethtool_flag_ioctl()).
	 */
	ret = iface_ethtool_flag_ioctl(handle, ETHTOOL_GUFO, "ETHTOOL_GUFO", 1);
	if (ret == -1)
		return -1;
	if (ret)
		return 1;	/* UDP fragmentation offloading on */
#endif

	return 0;
}
```

这段代码是一个用于 Linux network 设备驱动程序的代码。具体来说，它是一个 iface_get_offload() 函数，用于获取网络接口的 offload 设置。

在 iface_get_offload() 函数中，首先定义了一个静态变量 idsa_protos，用于存储 tagging 协议的列表。然后，定义了一个常量 bpf_u_int32，用于表示协议 ID。

接下来，处理 iface_get_offload() 函数的第一个参数，即 pcap_t 类型的 handle 变量，获取其 Offload 成员函数的返回值。如果 handle 指向的设备支持 ethtool ioctls，那么执行 iface_get_offload() 函数体内的代码，以获取 Offload 设置。如果设备不支持 ethtool ioctls，那么直接返回 0。

然后，在 iface_get_offload() 函数体外，定义了一个 dsa_proto 结构体数组，用于存储所有支持 tagging 协议的配置。

最后，在 iface_get_offload() 函数体内，根据协议 ID 遍历 dsa_protos 数组，查找并返回对应的协议配置。


```cpp
#else /* SIOCETHTOOL */
static int
iface_get_offload(pcap_t *handle _U_)
{
	/*
	 * XXX - do we need to get this information if we don't
	 * have the ethtool ioctls?  If so, how do we do that?
	 */
	return 0;
}
#endif /* SIOCETHTOOL */

static struct dsa_proto {
	const char *name;
	bpf_u_int32 linktype;
} dsa_protos[] = {
	/*
	 * None is special and indicates that the interface does not have
	 * any tagging protocol configured, and is therefore a standard
	 * Ethernet interface.
	 */
	{ "none", DLT_EN10MB },
	{ "brcm", DLT_DSA_TAG_BRCM },
	{ "brcm-prepend", DLT_DSA_TAG_BRCM_PREPEND },
	{ "dsa", DLT_DSA_TAG_DSA },
	{ "edsa", DLT_DSA_TAG_EDSA },
};

```

This function appears to parse a file that contains a list of DSA (Data Link Set Accounting) tags, one per line. It returns an error code and a message if the file is not read successfully or if the tags are not valid DSA tags.

The function takes a filename as an argument and opens it. It then reads the contents of the file and loops through each line. For each line, it checks whether it contains a valid DSA tag by comparing it to a known list of DSA tags. If it finds a match, it sets the `linktype` field of the corresponding DSA tag to the correct value and returns 0. If it does not find a match, it returns 1 and formats the error message using `asprintf`.

If the file is not read successfully (e.g. because the file does not exist or the read attempt fails), the function returns 1 and formats the error message using `asprintf`.


```cpp
static int
iface_dsa_get_proto_info(const char *device, pcap_t *handle)
{
	char *pathstr;
	unsigned int i;
	/*
	 * Make this significantly smaller than PCAP_ERRBUF_SIZE;
	 * the tag *shouldn't* have some huge long name, and making
	 * it smaller keeps newer versions of GCC from whining that
	 * the error message if we don't support the tag could
	 * overflow the error message buffer.
	 */
	char buf[128];
	ssize_t r;
	int fd;

	fd = asprintf(&pathstr, "/sys/class/net/%s/dsa/tagging", device);
	if (fd < 0) {
		pcap_fmt_errmsg_for_errno(handle->errbuf, PCAP_ERRBUF_SIZE,
					  fd, "asprintf");
		return PCAP_ERROR;
	}

	fd = open(pathstr, O_RDONLY);
	free(pathstr);
	/*
	 * This is not fatal, kernel >= 4.20 *might* expose this attribute
	 */
	if (fd < 0)
		return 0;

	r = read(fd, buf, sizeof(buf) - 1);
	if (r <= 0) {
		pcap_fmt_errmsg_for_errno(handle->errbuf, PCAP_ERRBUF_SIZE,
					  errno, "read");
		close(fd);
		return PCAP_ERROR;
	}
	close(fd);

	/*
	 * Buffer should be LF terminated.
	 */
	if (buf[r - 1] == '\n')
		r--;
	buf[r] = '\0';

	for (i = 0; i < sizeof(dsa_protos) / sizeof(dsa_protos[0]); i++) {
		if (strlen(dsa_protos[i].name) == (size_t)r &&
		    strcmp(buf, dsa_protos[i].name) == 0) {
			handle->linktype = dsa_protos[i].linktype;
			switch (dsa_protos[i].linktype) {
			case DLT_EN10MB:
				return 0;
			default:
				return 1;
			}
		}
	}

	snprintf(handle->errbuf, PCAP_ERRBUF_SIZE,
		      "unsupported DSA tag: %s", buf);

	return PCAP_ERROR;
}

```

这段代码的作用是查询给定接口的MTU（最大传输单元）。

具体来说，它首先构造一个查询请求的结构体 `ifreq`，并将其发送到系统获取MTU的接口上。如果接口不可用，它将返回一个错误代码。如果成功获取MTU，它将返回该接口的最大MTU。

函数的参数包括：

- `fd`：文件描述符，用于发送查询请求到系统；
- `device`：接口设备名称，用于构造查询请求的结构体；
- `ebuf`：用于存储查询结果的缓冲区，通常是一个 `char` 类型的数组。

返回值：

如果函数成功获取MTU，它将返回接口的MTU；如果接口不可用，它将返回一个错误代码，该代码将在 `pcap_fmt_errmsg_for_errno` 函数中打印错误消息。


```cpp
/*
 *  Query the kernel for the MTU of the given interface.
 */
static int
iface_get_mtu(int fd, const char *device, char *ebuf)
{
	struct ifreq	ifr;

	if (!device)
		return BIGGER_THAN_ALL_MTUS;

	memset(&ifr, 0, sizeof(ifr));
	pcap_strlcpy(ifr.ifr_name, device, sizeof(ifr.ifr_name));

	if (ioctl(fd, SIOCGIFMTU, &ifr) == -1) {
		pcap_fmt_errmsg_for_errno(ebuf, PCAP_ERRBUF_SIZE,
		    errno, "SIOCGIFMTU");
		return -1;
	}

	return ifr.ifr_mtu;
}

```

这段代码的作用是获取给定接口的硬件类型，并将其存储在ebuf指向的char数组中，具体实现如下：

1. 定义了一个名为iface_get_arptype的函数，参数为fd、device和ebuf，函数实现如下：

  - 初始化一个ifreq类型的变量ifr，其中ifr.ifr_name指向device参数所指的设备。
  -调用ioctl函数，使用SIOCGIFHWADDR的系统调用，获取给定设备描述符的硬件类型。
  - 如果获取失败，检查错误代码，并输出错误信息到ebuf数组中。
  - 如果获取成功，返回硬件类型。

2. iface_get_arptype函数的具体实现可以分为以下几个步骤：

  - 初始化ifr变量，使用memset函数将ifr变量清零。
  - 使用pcap_strlcpy函数，将device参数所指的设备字符串复制到ifr.ifr_name指向的内存空间中。
  - 如果ioctl函数的返回值为-1，说明设备不存在，使用iferrno变量获取错误代码，并输出错误信息到ebuf数组中。
  - 如果ioctl函数的返回值不为-1，说明获取成功，使用ifr.ifr_hwaddr.sa_family变量获取硬件类型，并将其存储到ebuf指向的内存空间中。

这段代码的作用是获取给定接口的硬件类型，并输出到ebuf指向的数组中，如果获取失败则输出错误信息。


```cpp
/*
 *  Get the hardware type of the given interface as ARPHRD_xxx constant.
 */
static int
iface_get_arptype(int fd, const char *device, char *ebuf)
{
	struct ifreq	ifr;
	int		ret;

	memset(&ifr, 0, sizeof(ifr));
	pcap_strlcpy(ifr.ifr_name, device, sizeof(ifr.ifr_name));

	if (ioctl(fd, SIOCGIFHWADDR, &ifr) == -1) {
		if (errno == ENODEV) {
			/*
			 * No such device.
			 *
			 * There's nothing more to say, so clear
			 * the error message.
			 */
			ret = PCAP_ERROR_NO_SUCH_DEVICE;
			ebuf[0] = '\0';
		} else {
			ret = PCAP_ERROR;
			pcap_fmt_errmsg_for_errno(ebuf, PCAP_ERRBUF_SIZE,
			    errno, "SIOCGIFHWADDR");
		}
		return ret;
	}

	return ifr.ifr_hwaddr.sa_family;
}

```


This function appears to be a part of the Linux kernel's networking stack, and it is used to filter packets that have been captured by the kernel's network interface (such as ethernet or wlan).

It takes a pointer to a packet structure (`struct packet`) as its input, and returns an error code indicating whether the function was successful in applying a filter to that packet or not.

The function performs the following steps:

1. Allocates memory for a new filter structure (`struct sock_filter`), which will be stored in the same buffer as the original packet structure.
2. Copies the contents of the packet structure into the filter structure.
3. Extracts the code of the packet structure from the buffer, and uses that code to decide which instructions to apply to the packet.
4. Filter the packet according to the rules defined by the `BPF_CLASS` and `BPF_MODE` fields of the packet structure.
5. If the function is unable to apply any filters to the packet, it returns an error code indicating that the `BPF_MSH` filter was not supported.

Note that the function assumes that the packet structure contains code that is valid for the current networking protocol (e.g. Ethernet or ICA套接字).


```cpp
static int
fix_program(pcap_t *handle, struct sock_fprog *fcode)
{
	struct pcap_linux *handlep = handle->priv;
	size_t prog_size;
	register int i;
	register struct bpf_insn *p;
	struct bpf_insn *f;
	int len;

	/*
	 * Make a copy of the filter, and modify that copy if
	 * necessary.
	 */
	prog_size = sizeof(*handle->fcode.bf_insns) * handle->fcode.bf_len;
	len = handle->fcode.bf_len;
	f = (struct bpf_insn *)malloc(prog_size);
	if (f == NULL) {
		pcap_fmt_errmsg_for_errno(handle->errbuf, PCAP_ERRBUF_SIZE,
		    errno, "malloc");
		return -1;
	}
	memcpy(f, handle->fcode.bf_insns, prog_size);
	fcode->len = len;
	fcode->filter = (struct sock_filter *) f;

	for (i = 0; i < len; ++i) {
		p = &f[i];
		/*
		 * What type of instruction is this?
		 */
		switch (BPF_CLASS(p->code)) {

		case BPF_LD:
		case BPF_LDX:
			/*
			 * It's a load instruction; is it loading
			 * from the packet?
			 */
			switch (BPF_MODE(p->code)) {

			case BPF_ABS:
			case BPF_IND:
			case BPF_MSH:
				/*
				 * Yes; are we in cooked mode?
				 */
				if (handlep->cooked) {
					/*
					 * Yes, so we need to fix this
					 * instruction.
					 */
					if (fix_offset(handle, p) < 0) {
						/*
						 * We failed to do so.
						 * Return 0, so our caller
						 * knows to punt to userland.
						 */
						return 0;
					}
				}
				break;
			}
			break;
		}
	}
	return 1;	/* we succeeded */
}

```

This is a Linux kernel function that checks the integrity of a packet based on a偏移字段. The function takes a packet header (p) and an offset as input and returns 0 if the packet is valid or -1 if it's not.

The function first checks the header offset against a threshold of 0, which means that if the header is larger than the header length, the function will return -1. If the header offset is valid, the function then checks if the packet type field is equal to 14, which is a special value that indicates the protocol field. If the protocol field is 14, the function maps the protocol field to the special magic kernel offset, and if it's not, the function maps the protocol field to the header offset.

If the packet type field is not 14, the function checks if the packet is valid according to the rules defined by the kernel. If the packet is valid, the function maps the packet type field to the header offset, and if it's not, the function returns -1.

If the function cannot determine the packet type or the protocol field is invalid, it will return -1.


```cpp
static int
fix_offset(pcap_t *handle, struct bpf_insn *p)
{
	/*
	 * Existing references to auxiliary data shouldn't be adjusted.
	 *
	 * Note that SKF_AD_OFF is negative, but p->k is unsigned, so
	 * we use >= and cast SKF_AD_OFF to unsigned.
	 */
	if (p->k >= (bpf_u_int32)SKF_AD_OFF)
		return 0;
	if (handle->linktype == DLT_LINUX_SLL2) {
		/*
		 * What's the offset?
		 */
		if (p->k >= SLL2_HDR_LEN) {
			/*
			 * It's within the link-layer payload; that starts
			 * at an offset of 0, as far as the kernel packet
			 * filter is concerned, so subtract the length of
			 * the link-layer header.
			 */
			p->k -= SLL2_HDR_LEN;
		} else if (p->k == 0) {
			/*
			 * It's the protocol field; map it to the
			 * special magic kernel offset for that field.
			 */
			p->k = SKF_AD_OFF + SKF_AD_PROTOCOL;
		} else if (p->k == 4) {
			/*
			 * It's the ifindex field; map it to the
			 * special magic kernel offset for that field.
			 */
			p->k = SKF_AD_OFF + SKF_AD_IFINDEX;
		} else if (p->k == 10) {
			/*
			 * It's the packet type field; map it to the
			 * special magic kernel offset for that field.
			 */
			p->k = SKF_AD_OFF + SKF_AD_PKTTYPE;
		} else if ((bpf_int32)(p->k) > 0) {
			/*
			 * It's within the header, but it's not one of
			 * those fields; we can't do that in the kernel,
			 * so punt to userland.
			 */
			return -1;
		}
	} else {
		/*
		 * What's the offset?
		 */
		if (p->k >= SLL_HDR_LEN) {
			/*
			 * It's within the link-layer payload; that starts
			 * at an offset of 0, as far as the kernel packet
			 * filter is concerned, so subtract the length of
			 * the link-layer header.
			 */
			p->k -= SLL_HDR_LEN;
		} else if (p->k == 0) {
			/*
			 * It's the packet type field; map it to the
			 * special magic kernel offset for that field.
			 */
			p->k = SKF_AD_OFF + SKF_AD_PKTTYPE;
		} else if (p->k == 14) {
			/*
			 * It's the protocol field; map it to the
			 * special magic kernel offset for that field.
			 */
			p->k = SKF_AD_OFF + SKF_AD_PROTOCOL;
		} else if ((bpf_int32)(p->k) > 0) {
			/*
			 * It's within the header, but it's not one of
			 * those fields; we can't do that in the kernel,
			 * so punt to userland.
			 */
			return -1;
		}
	}
	return 0;
}

```

This function appears to be responsible for attaching a new filter to a network device. It takes an optional "handle" parameter, which is a `socket` object. The function first checks if the socket is already in binary mode. If it's not, it converts the socket to binary mode and returns a success error. If the socket is already binary mode, it sets the filter index and retrieves the filter code. It then attempts to change the filter mode using the `setsockopt()` function. If the change fails, it checks if the filter was already on in the kernel, and if it was, it restores the filter index. If the filter was never on in the kernel, it reports a failure with a PCAP error.

If the socket is successfully changed to binary mode and the filter index is changed, it attaches the filter. If any errors occur, it returns an error code.


```cpp
static int
set_kernel_filter(pcap_t *handle, struct sock_fprog *fcode)
{
	int total_filter_on = 0;
	int save_mode;
	int ret;
	int save_errno;

	/*
	 * The socket filter code doesn't discard all packets queued
	 * up on the socket when the filter is changed; this means
	 * that packets that don't match the new filter may show up
	 * after the new filter is put onto the socket, if those
	 * packets haven't yet been read.
	 *
	 * This means, for example, that if you do a tcpdump capture
	 * with a filter, the first few packets in the capture might
	 * be packets that wouldn't have passed the filter.
	 *
	 * We therefore discard all packets queued up on the socket
	 * when setting a kernel filter.  (This isn't an issue for
	 * userland filters, as the userland filtering is done after
	 * packets are queued up.)
	 *
	 * To flush those packets, we put the socket in read-only mode,
	 * and read packets from the socket until there are no more to
	 * read.
	 *
	 * In order to keep that from being an infinite loop - i.e.,
	 * to keep more packets from arriving while we're draining
	 * the queue - we put the "total filter", which is a filter
	 * that rejects all packets, onto the socket before draining
	 * the queue.
	 *
	 * This code deliberately ignores any errors, so that you may
	 * get bogus packets if an error occurs, rather than having
	 * the filtering done in userland even if it could have been
	 * done in the kernel.
	 */
	if (setsockopt(handle->fd, SOL_SOCKET, SO_ATTACH_FILTER,
		       &total_fcode, sizeof(total_fcode)) == 0) {
		char drain[1];

		/*
		 * Note that we've put the total filter onto the socket.
		 */
		total_filter_on = 1;

		/*
		 * Save the socket's current mode, and put it in
		 * non-blocking mode; we drain it by reading packets
		 * until we get an error (which is normally a
		 * "nothing more to be read" error).
		 */
		save_mode = fcntl(handle->fd, F_GETFL, 0);
		if (save_mode == -1) {
			pcap_fmt_errmsg_for_errno(handle->errbuf,
			    PCAP_ERRBUF_SIZE, errno,
			    "can't get FD flags when changing filter");
			return -2;
		}
		if (fcntl(handle->fd, F_SETFL, save_mode | O_NONBLOCK) < 0) {
			pcap_fmt_errmsg_for_errno(handle->errbuf,
			    PCAP_ERRBUF_SIZE, errno,
			    "can't set nonblocking mode when changing filter");
			return -2;
		}
		while (recv(handle->fd, &drain, sizeof drain, MSG_TRUNC) >= 0)
			;
		save_errno = errno;
		if (save_errno != EAGAIN) {
			/*
			 * Fatal error.
			 *
			 * If we can't restore the mode or reset the
			 * kernel filter, there's nothing we can do.
			 */
			(void)fcntl(handle->fd, F_SETFL, save_mode);
			(void)reset_kernel_filter(handle);
			pcap_fmt_errmsg_for_errno(handle->errbuf,
			    PCAP_ERRBUF_SIZE, save_errno,
			    "recv failed when changing filter");
			return -2;
		}
		if (fcntl(handle->fd, F_SETFL, save_mode) == -1) {
			pcap_fmt_errmsg_for_errno(handle->errbuf,
			    PCAP_ERRBUF_SIZE, errno,
			    "can't restore FD flags when changing filter");
			return -2;
		}
	}

	/*
	 * Now attach the new filter.
	 */
	ret = setsockopt(handle->fd, SOL_SOCKET, SO_ATTACH_FILTER,
			 fcode, sizeof(*fcode));
	if (ret == -1 && total_filter_on) {
		/*
		 * Well, we couldn't set that filter on the socket,
		 * but we could set the total filter on the socket.
		 *
		 * This could, for example, mean that the filter was
		 * too big to put into the kernel, so we'll have to
		 * filter in userland; in any case, we'll be doing
		 * filtering in userland, so we need to remove the
		 * total filter so we see packets.
		 */
		save_errno = errno;

		/*
		 * If this fails, we're really screwed; we have the
		 * total filter on the socket, and it won't come off.
		 * Report it as a fatal error.
		 */
		if (reset_kernel_filter(handle) == -1) {
			pcap_fmt_errmsg_for_errno(handle->errbuf,
			    PCAP_ERRBUF_SIZE, errno,
			    "can't remove kernel total filter");
			return -2;	/* fatal error */
		}

		errno = save_errno;
	}
	return ret;
}

```

这段代码是一个用于 Linux kernel的网络编程接口，属于 `libpcap` 库的一部分。它的作用是实现了一个名为 `reset_kernel_filter` 的函数，用于设置或取消指定的内核过滤器。

函数接受一个 `pcap_t` 类型的参数，代表了一个网络套接字（socket）。函数首先设置一个名为 `dummy` 的值，用于在设置过滤器时忽略一些错误。

接着，函数调用 `setsockopt` 函数，设置套接字 `handle` 的过滤器。通过 `SOL_SOCKET` 参数指定要设置的协议类型，通过 `SO_DETACH_FILTER` 参数指定要创建的过滤器类型。`dummy` 参数用于在设置过滤器时忽略一个错误的返回值。

函数还检查是否有 `ENOENT` 和 `ENONET` 错误。如果没有找到这两个错误，函数返回一个非负整数，表示过滤器设置成功。如果找到任何一个错误，函数返回一个负数，可能表示过滤器设置失败或者设置了错误的过滤器。


```cpp
static int
reset_kernel_filter(pcap_t *handle)
{
	int ret;
	/*
	 * setsockopt() barfs unless it get a dummy parameter.
	 * valgrind whines unless the value is initialized,
	 * as it has no idea that setsockopt() ignores its
	 * parameter.
	 */
	int dummy = 0;

	ret = setsockopt(handle->fd, SOL_SOCKET, SO_DETACH_FILTER,
				   &dummy, sizeof(dummy));
	/*
	 * Ignore ENOENT - it means "we don't have a filter", so there
	 * was no filter to remove, and there's still no filter.
	 *
	 * Also ignore ENONET, as a lot of kernel versions had a
	 * typo where ENONET, rather than ENOENT, was returned.
	 */
	if (ret == -1 && errno != ENOENT && errno != ENONET)
		return -1;
	return 0;
}

```

这段代码定义了一个名为`pcap_set_protocol_linux`的函数和一个名为`pcap_lib_version`的函数。它们都是属于`libpcap`库的函数。

`pcap_set_protocol_linux`函数的作用是设置`pcap_t`类型的指针变量`p`所指向的套接字（socket）使用的协议类型。它接受一个整数参数`protocol`，并将其设置为`protocol`。

`pcap_lib_version`函数返回`libpcap`版本字符串。它只是简单地返回一个常量，使用时直接使用即可。


```cpp
int
pcap_set_protocol_linux(pcap_t *p, int protocol)
{
	if (pcap_check_activated(p))
		return (PCAP_ERROR_ACTIVATED);
	p->opt.protocol = protocol;
	return (0);
}

/*
 * Libpcap version string.
 */
const char *
pcap_lib_version(void)
{
```

这段代码是一个条件语句，它会根据一个名为HAVE_TPACKET3的定义来决定是否返回一个字符串，该定义可能来自一个名为HAVE_TPACKET3的预定义函数或头文件。如果没有定义HAVE_TPACKET3，该代码将返回字符串" (with TPACKET_V2)"，否则将返回字符串" (with TPACKET_V3) "。

具体来说，如果HAVE_TPACKET3定义存在，那么该代码将返回带有TPACKET_V3的TPACKET3版本，否则将返回带有TPACKET_V2的TPACKET2版本。


```cpp
#if defined(HAVE_TPACKET3)
	return (PCAP_VERSION_STRING " (with TPACKET_V3)");
#else
	return (PCAP_VERSION_STRING " (with TPACKET_V2)");
#endif
}

```