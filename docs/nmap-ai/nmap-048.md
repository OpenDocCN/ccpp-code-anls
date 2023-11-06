# Nmap源码解析 48

# `libpcap/pcap-airpcap.c`

This is a C++ header file that defines the data structure and constants for a Vertex in the OpenGL application programming interface (APSI). The header file specifies the format of the vertex data as well as the data fields that the vertex object can have.

The `vector<_t>` is a template class that can be used to store a collection of Vertex objects. The `_t` is a placeholder for a type parameter, which can be an indexed parameter or a generic type parameter. The `vector<_t>` is a member of the `std::vector` template class, which is a built-in container in the C++ Standard Template Library (STL).

The `size()` function is a member of the `std::vector` template class, it is used to return the number of elements in the vector.

The `push_back()` is a member function of the vector class, it is used to add a new element of the correct type to the vector.

The `begin()` and `end()` are member functions of the vector class, they are used to access and modify the elements of the vector.

The `clear()` is a member function of the vector class, it is used to remove all elements from the vector.

The `max()` and `min()` are member functions of the std::vector, they are used to return the maximum and minimum elements in the vector, respectively.

The `swap()` is a member function of the std::vector, it is used to exchange the elements of two vectors.

The `reverse()` is a member function of the std::vector, it is used to reverse the elements of the vector.


```cpp
/*
 * Copyright (c) 1999 - 2005 NetGroup, Politecnico di Torino (Italy)
 * Copyright (c) 2005 - 2010 CACE Technologies, Davis (California)
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
 * 3. Neither the name of the Politecnico di Torino, CACE Technologies
 * nor the names of its contributors may be used to endorse or promote
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
 *
 */

```

这段代码是一个简单的C语言编程语句，它包含了一个if语句和两个头文件包含。

if语句检查系统是否支持配置文件。如果不支持，它将包含下面的代码。如果支持，那么以下代码将不会被输出。

```cpp
#include <config.h>
#include "pcap-int.h"
#include "pcap-airpcap.h"
```

第一个头文件 `config.h` 是包含一些全局定义的。第二个头文件 `pcap-int.h` 是包含 `pcap` 函数的定义。第三个头文件 `pcap-airpcap.h` 是包含 `airpcap` 函数的定义。

接下来的 `#include` 语句将 `config.h` 和 `pcap-int.h` 和 `pcap-airpcap.h` 包含的内容加载到程序中。

在 `#define` 语句中，我们定义了两个宏，`AIRPCAP_DEFAULT_USER_BUFFER_SIZE` 和 `AIRPCAP_DEFAULT_KERNEL_BUFFER_SIZE`。它们定义了默认的缓冲区大小，分别对于用户空间和内核空间。

`#ifdef` 和 `#include` 语句是互相对立的，如果 `#ifdef` 存在，那么它将不会被输出，但如果 `#include` 存在，那么 `#ifdef` 将被丢弃。


```cpp
#ifdef HAVE_CONFIG_H
#include <config.h>
#endif

#include "pcap-int.h"

#include <airpcap.h>

#include "pcap-airpcap.h"

/* Default size of the buffer we allocate in userland. */
#define	AIRPCAP_DEFAULT_USER_BUFFER_SIZE 256000

/* Default size of the buffer for the AirPcap adapter. */
#define	AIRPCAP_DEFAULT_KERNEL_BUFFER_SIZE 1000000

```

这段代码定义了一系列函数指针变量，它们分别代表与PAirpcap库有关的操作。具体来说：

1. `airpcap_lib`：指向PAirpcap库的代码段。当程序运行时，它将动态加载这个库，这样即使没有安装过库，也可以正常工作，而且不会要求安装相同的库版本。

2. `AirpcapGetLastErrorHandler`：代表函数，接收一个PAirpcapHandle类型的对象和一个PCHAR类型的指针。这个函数用于获取Airpcap库中的最后一个错误码。

3. `AirpcapGetDeviceListHandler`：代表函数，接收一个PAirpcapDeviceDescription类型的对象和一个PCHAR类型的指针。这个函数用于获取Airpcap库中的设备列表，并将其存储在PAirpcapDeviceDescription类型的变量中。

4. `AirpcapFreeDeviceListHandler`：代表函数，接收一个PAirpcapDeviceDescription类型的对象。这个函数用于释放Airpcap库中的设备列表，以便正确地卸载数据库。

5. `AirpcapOpenHandler`：代表函数，接收一个PCHAR类型的指针和一个PCHAR类型的指针。这个函数用于打开Airpcap库，并返回一个指向新创建的PAirpcapHandle类型的指针。

6. `AirpcapCloseHandler`：代表函数，接收一个PAirpcapHandle类型的对象。这个函数用于关闭Airpcap库，并释放之前分配的内存。

7. `AirpcapSetDeviceMacFlagsHandler`：代表函数，接收一个PAirpcapHandle类型的对象和一个UINT类型的参数。这个函数用于设置Airpcap库中的设备麦克风 flags，并将它们存储在接收的PAirpcapHandle类型的对象中。

8. `AirpcapSetLinkTypeHandler`：代表函数，接收一个PAirpcapHandle类型的对象和一个AirpcapLinkType类型的参数。这个函数用于设置Airpcap库中的设备链接类型，并将它们存储在接收的PAirpcapHandle类型的对象中。


```cpp
//
// We load the AirPcap DLL dynamically, so that the code will
// work whether you have it installed or not, and there don't
// have to be two different versions of the library, one linked
// to the AirPcap library and one not linked to it.
//
static pcap_code_handle_t airpcap_lib;

typedef PCHAR (*AirpcapGetLastErrorHandler)(PAirpcapHandle);
typedef BOOL (*AirpcapGetDeviceListHandler)(PAirpcapDeviceDescription *, PCHAR);
typedef VOID (*AirpcapFreeDeviceListHandler)(PAirpcapDeviceDescription);
typedef PAirpcapHandle (*AirpcapOpenHandler)(PCHAR, PCHAR);
typedef VOID (*AirpcapCloseHandler)(PAirpcapHandle);
typedef BOOL (*AirpcapSetDeviceMacFlagsHandler)(PAirpcapHandle, UINT);
typedef BOOL (*AirpcapSetLinkTypeHandler)(PAirpcapHandle, AirpcapLinkType);
```



这段代码定义了几个名为AirpcapGetLinkType、AirpcapSetKernelBuffer、AirpcapSetFilter、AirpcapSetMinToCopy、AirpcapGetReadEvent、AirpcapRead、AirpcapWrite和AirpcapGetStats的函数指针类型。这些函数用于处理与Airpcap相关的操作，例如获取和设置Airpcap的链接类型、内核缓冲区、过滤器、最小复制数量、读取事件、数据传输和获取统计信息等。

AirpcapGetLastError函数用于获取上一次Airpcap操作中发生的最后错误代码，如果没有错误则返回0。

AirpcapGetDeviceList函数用于获取连接设备的列表，并将其存储在p_AirpcapDeviceList变量中。

AirpcapFreeDeviceList函数用于将连接设备列表中所有设备的列表复制到p_AirpcapNoDeviceList变量中，并使用p_AirpcapSetDeviceMacFlags函数设置设备的Mac操作系统类型标志，以便Airpcap将数据传输限制为本地设备类型。

AirpcapOpen函数用于打开Airpcap设备，并返回一个OpenHandle变量。

AirpcapClose函数用于关闭Airpcap设备，并返回一个CloseHandle变量。

AirpcapSetDeviceMacFlags函数用于设置Airpcap设备类型标志，以便Airpcap可以选择性地传输数据到或从指定的设备。

AirpcapGetReadEvent函数用于设置Airpcap读取事件的发生时间戳，并返回一个ReadEventHandle变量。

AirpcapRead函数用于读取Airpcap设备中的数据，并将其存储在p_AirpcapReadBuffer变量中。

AirpcapWrite函数用于向Airpcap设备写入数据，并将其存储在p_AirpcapWriteBuffer变量中。

AirpcapGetStats函数用于获取Airpcap设备的统计信息，并将其存储在p_AirpcapStats变量中。


```cpp
typedef BOOL (*AirpcapGetLinkTypeHandler)(PAirpcapHandle, PAirpcapLinkType);
typedef BOOL (*AirpcapSetKernelBufferHandler)(PAirpcapHandle, UINT);
typedef BOOL (*AirpcapSetFilterHandler)(PAirpcapHandle, PVOID, UINT);
typedef BOOL (*AirpcapSetMinToCopyHandler)(PAirpcapHandle, UINT);
typedef BOOL (*AirpcapGetReadEventHandler)(PAirpcapHandle, HANDLE *);
typedef BOOL (*AirpcapReadHandler)(PAirpcapHandle, PBYTE, UINT, PUINT);
typedef BOOL (*AirpcapWriteHandler)(PAirpcapHandle, PCHAR, ULONG);
typedef BOOL (*AirpcapGetStatsHandler)(PAirpcapHandle, PAirpcapStats);

static AirpcapGetLastErrorHandler p_AirpcapGetLastError;
static AirpcapGetDeviceListHandler p_AirpcapGetDeviceList;
static AirpcapFreeDeviceListHandler p_AirpcapFreeDeviceList;
static AirpcapOpenHandler p_AirpcapOpen;
static AirpcapCloseHandler p_AirpcapClose;
static AirpcapSetDeviceMacFlagsHandler p_AirpcapSetDeviceMacFlags;
```

这段代码定义了多个静态变量，它们都是指向同名的函数指针类型，这些函数指针都指向类AirpcapLinkTypeHandler的函数。这些函数指针用于处理与Airpcap Link Type相关的事务。

具体来说，p_AirpcapSetLinkType是一个函数指针，指向名为SetLinkType的函数；p_AirpcapGetLinkType是一个函数指针，指向名为GetLinkType的函数；p_AirpcapSetKernelBuffer是一个函数指针，指向名为SetKernelBuffer的函数；p_AirpcapSetFilter是一个函数指针，指向名为SetFilter的函数；p_AirpcapSetMinToCopy是一个函数指针，指向名为SetMinToCopy的函数；p_AirpcapGetReadEvent是一个函数指针，指向名为GetReadEvent的函数；p_AirpcapRead是一个函数指针，指向名为Read的函数；p_AirpcapWrite是一个函数指针，指向名为Write的函数；p_AirpcapGetStats是一个函数指针，指向名为GetStats的函数。


```cpp
static AirpcapSetLinkTypeHandler p_AirpcapSetLinkType;
static AirpcapGetLinkTypeHandler p_AirpcapGetLinkType;
static AirpcapSetKernelBufferHandler p_AirpcapSetKernelBuffer;
static AirpcapSetFilterHandler p_AirpcapSetFilter;
static AirpcapSetMinToCopyHandler p_AirpcapSetMinToCopy;
static AirpcapGetReadEventHandler p_AirpcapGetReadEvent;
static AirpcapReadHandler p_AirpcapRead;
static AirpcapWriteHandler p_AirpcapWrite;
static AirpcapGetStatsHandler p_AirpcapGetStats;

typedef enum LONG
{
	AIRPCAP_API_UNLOADED = 0,
	AIRPCAP_API_LOADED,
	AIRPCAP_API_CANNOT_LOAD,
	AIRPCAP_API_LOADING
} AIRPCAP_API_LOAD_STATUS;

```

This function appears to be part of the AirPcap API for working with network devices. It appears to be used to check if certain functions in the AirPcap library are properly defined and initialized, and to set the current status of the AirPcap library accordingly.

The function takes several arguments:

- `airpcap_lib`: a pointer to the AirPcap library
- `pcap_find_function`: a function used to look for a particular function in the AirPcap library.
- `p_AirpcapGetStats`: a function used to get the current statistics for the AirPcap library.
- `pcap_write`: a function used to write data to the network device

It appears to do the following:

1. Make sure the AirPcap library and the functions for getting statistics and writing data are properly defined and initialized.
2. If the library is not found or the functions are not defined, it sets the current status to `AIRPCAP_API_NOT_FOUND`.
3. If the library and functions are defined, it sets the current status to `AIRPCAP_API_LOADED`.

It is currently working on AIRPCAP\_API\_NOT\_FOUND


```cpp
static AIRPCAP_API_LOAD_STATUS	airpcap_load_status;

/*
 * NOTE: this function should be called by the pcap functions that can
 *       theoretically deal with the AirPcap library for the first time,
 *       namely listing the adapters and creating a pcap_t for an adapter.
 *       All the other ones (activate, close, read, write, set parameters)
 *       work on a pcap_t for an AirPcap device, meaning we've already
 *       created the pcap_t and thus have loaded the functions, so we do
 *       not need to call this function.
 */
static AIRPCAP_API_LOAD_STATUS
load_airpcap_functions(void)
{
	AIRPCAP_API_LOAD_STATUS current_status;

	/*
	 * We don't use a mutex because there's no place that
	 * we can guarantee we'll be called before any threads
	 * other than the main thread exists.  (For example,
	 * this might be a static library, so we can't arrange
	 * to be called by DllMain(), and there's no guarantee
	 * that the application called pcap_init() - which is
	 * supposed to be called only from one thread - so
	 * we can't arrange to be called from it.)
	 *
	 * If nobody's tried to load it yet, mark it as
	 * loading; in any case, return the status before
	 * we modified it.
	 */
	current_status = InterlockedCompareExchange((LONG *)&airpcap_load_status,
	    AIRPCAP_API_LOADING, AIRPCAP_API_UNLOADED);

	/*
	 * If the status was AIRPCAP_API_UNLOADED, we've set it
	 * to AIRPCAP_API_LOADING, because we're going to be
	 * the ones to load the library but current_status is
	 * AIRPCAP_API_UNLOADED.
	 *
	 * if it was AIRPCAP_API_LOADING, meaning somebody else
	 * was trying to load it, spin until they finish and
	 * set the status to a value reflecting whether they
	 * succeeded.
	 */
	while (current_status == AIRPCAP_API_LOADING) {
		current_status = InterlockedCompareExchange((LONG*)&airpcap_load_status,
		    AIRPCAP_API_LOADING, AIRPCAP_API_LOADING);
		Sleep(10);
	}

	/*
	 * At this point, current_status is either:
	 *
	 *	AIRPCAP_API_LOADED, in which case another thread
	 *	loaded the library, so we're done;
	 *
	 *	AIRPCAP_API_CANNOT_LOAD, in which another thread
	 *	tried and failed to load the library, so we're
	 *	done - we won't try it ourselves;
	 *
	 *	AIRPCAP_API_LOADING, in which case *we're* the
	 *	ones loading it, and should now try to do so.
	 */
	if (current_status == AIRPCAP_API_LOADED)
		return AIRPCAP_API_LOADED;

	if (current_status == AIRPCAP_API_CANNOT_LOAD)
		return AIRPCAP_API_CANNOT_LOAD;

	/*
	 * Start out assuming we can't load it.
	 */
	current_status = AIRPCAP_API_CANNOT_LOAD;

	airpcap_lib = pcap_load_code("airpcap.dll");
	if (airpcap_lib != NULL) {
		/*
		 * OK, we've loaded the library; now try to find the
		 * functions we need in it.
		 */
		p_AirpcapGetLastError = (AirpcapGetLastErrorHandler) pcap_find_function(airpcap_lib, "AirpcapGetLastError");
		p_AirpcapGetDeviceList = (AirpcapGetDeviceListHandler) pcap_find_function(airpcap_lib, "AirpcapGetDeviceList");
		p_AirpcapFreeDeviceList = (AirpcapFreeDeviceListHandler) pcap_find_function(airpcap_lib, "AirpcapFreeDeviceList");
		p_AirpcapOpen = (AirpcapOpenHandler) pcap_find_function(airpcap_lib, "AirpcapOpen");
		p_AirpcapClose = (AirpcapCloseHandler) pcap_find_function(airpcap_lib, "AirpcapClose");
		p_AirpcapSetDeviceMacFlags = (AirpcapSetDeviceMacFlagsHandler) pcap_find_function(airpcap_lib, "AirpcapSetDeviceMacFlags");
		p_AirpcapSetLinkType = (AirpcapSetLinkTypeHandler) pcap_find_function(airpcap_lib, "AirpcapSetLinkType");
		p_AirpcapGetLinkType = (AirpcapGetLinkTypeHandler) pcap_find_function(airpcap_lib, "AirpcapGetLinkType");
		p_AirpcapSetKernelBuffer = (AirpcapSetKernelBufferHandler) pcap_find_function(airpcap_lib, "AirpcapSetKernelBuffer");
		p_AirpcapSetFilter = (AirpcapSetFilterHandler) pcap_find_function(airpcap_lib, "AirpcapSetFilter");
		p_AirpcapSetMinToCopy = (AirpcapSetMinToCopyHandler) pcap_find_function(airpcap_lib, "AirpcapSetMinToCopy");
		p_AirpcapGetReadEvent = (AirpcapGetReadEventHandler) pcap_find_function(airpcap_lib, "AirpcapGetReadEvent");
		p_AirpcapRead = (AirpcapReadHandler) pcap_find_function(airpcap_lib, "AirpcapRead");
		p_AirpcapWrite = (AirpcapWriteHandler) pcap_find_function(airpcap_lib, "AirpcapWrite");
		p_AirpcapGetStats = (AirpcapGetStatsHandler) pcap_find_function(airpcap_lib, "AirpcapGetStats");

		//
		// Make sure that we found everything
		//
		if (p_AirpcapGetLastError != NULL &&
		    p_AirpcapGetDeviceList != NULL &&
		    p_AirpcapFreeDeviceList != NULL &&
		    p_AirpcapOpen != NULL &&
		    p_AirpcapClose != NULL &&
		    p_AirpcapSetDeviceMacFlags != NULL &&
		    p_AirpcapSetLinkType != NULL &&
		    p_AirpcapGetLinkType != NULL &&
		    p_AirpcapSetKernelBuffer != NULL &&
		    p_AirpcapSetFilter != NULL &&
		    p_AirpcapSetMinToCopy != NULL &&
		    p_AirpcapGetReadEvent != NULL &&
		    p_AirpcapRead != NULL &&
		    p_AirpcapWrite != NULL &&
		    p_AirpcapGetStats != NULL) {
			/*
			 * We have all we need.
			 */
			current_status = AIRPCAP_API_LOADED;
		}
	}

	if (current_status != AIRPCAP_API_LOADED) {
		/*
		 * We failed; if we found the DLL, close the
		 * handle for it.
		 */
		if (airpcap_lib != NULL) {
			FreeLibrary(airpcap_lib);
			airpcap_lib = NULL;
		}
	}

	/*
	 * Now set the status appropriately - and atomically.
	 */
	InterlockedExchange((LONG *)&airpcap_load_status, current_status);

	return current_status;
}

```



This is a Java class that implements the `landefender` algorithm for AirPcap, which is a networking packet sniffing tool.

The `AirPcap` class has several methods for setting up the filter, including `setfilter()`, which takes a `pcap_t` object and a `struct bpf_program` structure.

The `setfilter()` method reads the current AirPcap filter program from the `adapter` field of the `pcap_t` object, and then installs it in the kernel using the `install_bpf_program()` function.

If the `install_bpf_program()` function fails, the method returns an error. If the filter program is successfully installed, the method sets the `filtering_in_kernel` field of the `pcap_airpcap` object to `1`, indicating that the filter is currently in the kernel.

Finally, the method discards any previously-received packets and returns zero.


```cpp
/*
 * Private data for capturing on AirPcap devices.
 */
struct pcap_airpcap {
	PAirpcapHandle adapter;
	int filtering_in_kernel;
	int nonblock;
	int read_timeout;
	HANDLE read_event;
	struct pcap_stat stat;
};

static int
airpcap_setfilter(pcap_t *p, struct bpf_program *fp)
{
	struct pcap_airpcap *pa = p->priv;

	if (!p_AirpcapSetFilter(pa->adapter, fp->bf_insns,
	    fp->bf_len * sizeof(struct bpf_insn))) {
		/*
		 * Kernel filter not installed.
		 *
		 * XXX - we don't know whether this failed because:
		 *
		 *  the kernel rejected the filter program as invalid,
		 *  in which case we should fall back on userland
		 *  filtering;
		 *
		 *  the kernel rejected the filter program as too big,
		 *  in which case we should again fall back on
		 *  userland filtering;
		 *
		 *  there was some other problem, in which case we
		 *  should probably report an error;
		 *
		 * So we just fall back on userland filtering in
		 * all cases.
		 */

		/*
		 * install_bpf_program() validates the program.
		 *
		 * XXX - what if we already have a filter in the kernel?
		 */
		if (install_bpf_program(p, fp) < 0)
			return (-1);
		pa->filtering_in_kernel = 0;	/* filtering in userland */
		return (0);
	}

	/*
	 * It worked.
	 */
	pa->filtering_in_kernel = 1;	/* filtering in the kernel */

	/*
	 * Discard any previously-received packets, as they might have
	 * passed whatever filter was formerly in effect, but might
	 * not pass this filter (BIOCSETF discards packets buffered
	 * in the kernel, so you can lose packets in any case).
	 */
	p->cc = 0;
	return (0);
}

```

这段代码是一个名为`airpcap_set_datalink`的函数，属于`airpcap`库，用于设置数据链路跟踪（DLT）的类型。

具体来说，函数接收一个`pcap_t`类型的数据链表指针`p`和数字参数`dlt`，然后执行以下操作：

1. 获取一个`struct pcap_airpcap`类型的指针`pa`，该指针属于数据链表的内部结构。
2. 定义一个`AirpcapLinkType`类型的变量`type`，用于表示数据链路的类型。
3. 根据`dlt`参数的值选择`AirpcapLinkType`类型，例如：
	* 如果`dlt`为`DLT_IEEE802_11_RADIO`，则执行以下操作：
		1. 将`type`设置为`AIRPCAP_LT_802_11_PLUS_RADIO`。
		2. 不会执行其他操作。
	* 如果`dlt`为`DLT_PPI`，则执行以下操作：
		1. 将`type`设置为`AIRPCAP_LT_802_11_PLUS_PPI`。
		2. 不会执行其他操作。
	* 如果`dlt`为`DLT_IEEE802_11`，则执行以下操作：
		1. 将`type`设置为`AIRPCAP_LT_802_11`。
		2. 不会执行其他操作。
	* 如果`dlt`的值不匹配任何一个预定义的类型，则执行以下操作：
		1. 返回一个错误码。
		2. 调用链表的`errbuf`成员，使用`snprintf`函数打印错误信息，例如：
			"AirpcapSetLinkType() failed: %s"，然后返回-1。

函数的返回值表示一个`int`类型的值，根据`dlt`参数的值执行不同操作，返回相应的结果。


```cpp
static int
airpcap_set_datalink(pcap_t *p, int dlt)
{
	struct pcap_airpcap *pa = p->priv;
	AirpcapLinkType type;

	switch (dlt) {

	case DLT_IEEE802_11_RADIO:
		type = AIRPCAP_LT_802_11_PLUS_RADIO;
		break;

	case DLT_PPI:
		type = AIRPCAP_LT_802_11_PLUS_PPI;
		break;

	case DLT_IEEE802_11:
		type = AIRPCAP_LT_802_11;
		break;

	default:
		/* This can't happen; just return. */
		return (0);
	}
	if (!p_AirpcapSetLinkType(pa->adapter, type)) {
		snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
		    "AirpcapSetLinkType() failed: %s",
		    p_AirpcapGetLastError(pa->adapter));
		return (-1);
	}
	p->linktype = dlt;
	return (0);
}

```

这两段代码是用于设置空气净化器（airpcap）设备的非阻塞状态的函数。

“airpcap_getnonblock(pcap_t *p)”函数返回设备的非阻塞状态，如果设备处于非阻塞模式，则返回设备的非阻塞状态的值；如果设备已经连接，则返回0。

“airpcap_setnonblock(pcap_t *p, int nonblock)”函数用于设置设备的非阻塞状态。如果设备的非阻塞状态为1，则设置新的超时时间（timeout）为设备的读取超时时间（read_timeout），并将设备设置为非阻塞模式。如果设备的非阻塞状态为0，则将设备的读取超时时间设置为新的超时时间，并将设备设置为阻塞模式。

在这段代码中，通过使用pcap_t结构体中的priv成员，访问了设备的私有成员函数，从而实现了对设备非阻塞状态的设置。


```cpp
static int
airpcap_getnonblock(pcap_t *p)
{
	struct pcap_airpcap *pa = p->priv;

	return (pa->nonblock);
}

static int
airpcap_setnonblock(pcap_t *p, int nonblock)
{
	struct pcap_airpcap *pa = p->priv;
	int newtimeout;

	if (nonblock) {
		/*
		 * Set the packet buffer timeout to -1 for non-blocking
		 * mode.
		 */
		newtimeout = -1;
	} else {
		/*
		 * Restore the timeout set when the device was opened.
		 * (Note that this may be -1, in which case we're not
		 * really leaving non-blocking mode.  However, although
		 * the timeout argument to pcap_set_timeout() and
		 * pcap_open_live() is an int, you're not supposed to
		 * supply a negative value, so that "shouldn't happen".)
		 */
		newtimeout = p->opt.timeout;
	}
	pa->read_timeout = newtimeout;
	pa->nonblock = (newtimeout == -1);
	return (0);
}

```

这段代码是一个名为“airpcap_stats”的函数，属于“pcap”文件夹。它的作用是获取一个名为“pa”的结构体中包含的“adapter”成员的统计信息，然后将统计信息存储到“ps”结构体中。

函数内部首先定义了一个名为“tas”的结构体，然后使用“p_AirpcapGetStats”函数尝试从“pa”的“adapter”成员中获取统计信息，如果获取失败则输出错误信息并返回-1，否则将获取到的统计信息存储到“ps”结构体中。

函数的具体实现可以分为以下几个步骤：

1. 定义一个名为“tas”的结构体，用于存储统计信息。
2. 定义一个名为“ps”的结构体，用于存储统计信息。
3. 定义一个名为“pa”的结构体，用于存储当前网络适配器的成员变量。
4. 调用“p_AirpcapGetStats”函数，获取统计信息。
5. 如果获取成功，将统计信息存储到“ps”结构体中。
6. 返回0，表示函数成功并返回成功值。


```cpp
static int
airpcap_stats(pcap_t *p, struct pcap_stat *ps)
{
	struct pcap_airpcap *pa = p->priv;
	AirpcapStats tas;

	/*
	 * Try to get statistics.
	 */
	if (!p_AirpcapGetStats(pa->adapter, &tas)) {
		snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
		    "AirpcapGetStats() failed: %s",
		    p_AirpcapGetLastError(pa->adapter));
		return (-1);
	}

	ps->ps_drop = tas.Drops;
	ps->ps_recv = tas.Recvs;
	ps->ps_ifdrop = tas.IfDrops;

	return (0);
}

```

这段代码是一个用于获取操作系统统计信息的Win32本地函数。它与从用户空间中传递过来的pcap_stat参数相比，更安全可靠。

在这段注释中，开发者强调这种方法比使用变量更安全，因为变量的内容可能会因为变得太大而使程序崩溃或者在分配内存时出现错误。同时，这种方法可以写入由操作系统分配给该变量的内存区域。

该函数的使用方式取决于需要获取哪些统计信息，可以通过函数来实现。由于代码中没有提供具体的函数原型，因此无法提供具体的实现方式。


```cpp
/*
 * Win32-only routine for getting statistics.
 *
 * This way is definitely safer than passing the pcap_stat * from the userland.
 * In fact, there could happen than the user allocates a variable which is not
 * big enough for the new structure, and the library will write in a zone
 * which is not allocated to this variable.
 *
 * In this way, we're pretty sure we are writing on memory allocated to this
 * variable.
 *
 * XXX - but this is the wrong way to handle statistics.  Instead, we should
 * have an API that returns data in a form like the Options section of a
 * pcapng Interface Statistics Block:
 *
 *    https://xml2rfc.tools.ietf.org/cgi-bin/xml2rfc.cgi?url=https://raw.githubusercontent.com/pcapng/pcapng/master/draft-tuexen-opsawg-pcapng.xml&modeAsFormat=html/ascii&type=ascii#rfc.section.4.6
 *
 * which would let us add new statistics straightforwardly and indicate which
 * statistics we are and are *not* providing, rather than having to provide
 * possibly-bogus values for statistics we can't provide.
 */
```

这段代码是一个名为“airpcap_stats_ex”的函数，属于“pcap”类别，用于从给定的网卡（pcap_t）中获取统计信息，并将结果存储在“pcap_stat”结构中。该函数的实现如下：

1. 初始化函数参数：定义了一个指向“struct pcap_stat”类型的指针变量“pcap_stat_size”，以及一个指向“int”类型的指针变量“pcap_stat”。
2. 从“p_AirpcapGetStats”函数中获取统计信息：尝试从给定的网卡中获取统计信息，并将结果存储在“tas”结构中。
3. 将统计信息存储到结构体中：将“tas”结构中的统计信息（如“Recvs”、“Drops”和“IfDrops”）存储到“pcap_stat”结构中，其中“ps_recv”表示接收的数据量，“ps_drop”表示丢失的数据量，“ps_ifdrop”表示接口丢弃的数据量。
4. 在函数头部添加一些注释：在函数头部添加了一些注释，用于说明函数的用途以及实现原理。


```cpp
static struct pcap_stat *
airpcap_stats_ex(pcap_t *p, int *pcap_stat_size)
{
	struct pcap_airpcap *pa = p->priv;
	AirpcapStats tas;

	*pcap_stat_size = sizeof (p->stat);

	/*
	 * Try to get statistics.
	 */
	if (!p_AirpcapGetStats(pa->adapter, &tas)) {
		snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
		    "AirpcapGetStats() failed: %s",
		    p_AirpcapGetLastError(pa->adapter));
		return (NULL);
	}

	p->stat.ps_recv = tas.Recvs;
	p->stat.ps_drop = tas.Drops;
	p->stat.ps_ifdrop = tas.IfDrops;
	/*
	 * Just in case this is ever compiled for a target other than
	 * Windows, which is extremely unlikely at best.
	 */
```

这段代码是一个名为`airpcap_setbuff`的函数，属于`airpcap`结构的子结构。它的作用是设置一个操作系统级别的数据缓冲区的维度。以下是该函数的逐步解释：

1. 首先，定义一个名为`_WIN32`的预处理指令，后面会详细解释。
2. 在预处理指令部分，定义了一个名为`p->stat`的局部变量，用于保存修改后的统计信息。
3. 使用`#ifdef`和`#endif`定义了一个名为`_WIN32`的预处理指令，使得某些编译器可以省略掉该指令。如果不包含`_WIN32`预处理指令，则会输出一条错误信息。
4. 调用一个名为`tas.Capt`的函数，并将其返回值赋值给`p->stat.ps_capt`。
5. 如果`tas.Capt`函数执行失败，则会使用`p_AirpcapSetKernelBuffer`函数失败，并输出一条错误信息。如果成功，则返回0。
6. 最终返回设置缓冲区维度`dim`的返回值。


```cpp
#ifdef _WIN32
	p->stat.ps_capt = tas.Capt;
#endif
	return (&p->stat);
}

/* Set the dimension of the kernel-level capture buffer */
static int
airpcap_setbuff(pcap_t *p, int dim)
{
	struct pcap_airpcap *pa = p->priv;

	if (!p_AirpcapSetKernelBuffer(pa->adapter, dim)) {
		snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
		    "AirpcapSetKernelBuffer() failed: %s",
		    p_AirpcapGetLastError(pa->adapter));
		return (-1);
	}
	return (0);
}

```



这段代码定义了两个名为 `airpcap_setmode()` 和 `airpcap_setmintocopy()` 的函数，属于 AirPcap 驱动程序的API函数。

`airpcap_setmode()` 函数用于设置 airpcap 驱动程序的工作模式。函数的第一个参数 `p` 代表 airpcap 数据包 capture 实例，第二个参数 `mode` 表示要设置的工作模式，可以取的模式包括：MODE_CAPT、MODE_FCP、MODE_ICMP。如果设置的工作模式不正确，函数会输出一个错误信息，并返回 -1。

`airpcap_setmintocopy()` 函数用于设置 airpcap 驱动程序最小数据量，即在释放读取调用时最小允许的数据量。函数的第一个参数 `p` 代表 airpcap 数据包 capture 实例，第二个参数 `size` 表示要设置的最小数据量。函数会尝试调用 `p_AirpcapSetMinToCopy()` 函数，如果失败，将输出错误信息并返回 -1。


```cpp
/* Set the driver working mode */
static int
airpcap_setmode(pcap_t *p, int mode)
{
	 if (mode != MODE_CAPT) {
		snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
		    "Only MODE_CAPT is supported on an AirPcap adapter");
		return (-1);
	 }
	 return (0);
}

/*set the minimum amount of data that will release a read call*/
static int
airpcap_setmintocopy(pcap_t *p, int size)
{
	struct pcap_airpcap *pa = p->priv;

	if (!p_AirpcapSetMinToCopy(pa->adapter, size)) {
		snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
		    "AirpcapSetMinToCopy() failed: %s",
		    p_AirpcapGetLastError(pa->adapter));
		return (-1);
	}
	return (0);
}

```



这段代码是一个名为`airpcap_getevent`的函数，属于`airpcap`类的成员函数。它的作用是返回一个`pcap_t`类型的指针，该指针指向一个`struct pcap_airpcap`类型的数据结构。

具体来说，该函数的实现以下两步：

1. 首先，它创建了一个名为`pa`的`struct pcap_airpcap`类型的数据结构，并将它作为实参传递给一个名为`p`的参数。

2. 接着，该函数返回`pa->read_event`，即`pa`指向的`struct pcap_airpcap`中包含`read_event`成员的指针。由于`read_event`是`struct pcap_airpcap`的一个成员函数，因此该函数的作用就是获取`pa`指向的`struct pcap_airpcap`中的`read_event`函数返回的值，然后将其返回。

另外，该函数还有一个名为`airpcap_oid_get_request`的静态函数，它的作用是获取一个`bpf_u_int32`类型的对象，该对象的值为`0`。它的实现类似于一个`PCAP_ERROR`类型的函数，返回值为`PCAP_ERROR`。由于题目中未给出具体的错误情况，因此这里不做详细解释。


```cpp
static HANDLE
airpcap_getevent(pcap_t *p)
{
	struct pcap_airpcap *pa = p->priv;

	return (pa->read_event);
}

static int
airpcap_oid_get_request(pcap_t *p, bpf_u_int32 oid _U_, void *data _U_,
    size_t *lenp _U_)
{
	snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
	    "Getting OID values is not supported on an AirPcap adapter");
	return (PCAP_ERROR);
}

```

这是一个用C语言编写的名为"airpcap_oid_set_request.h"的头文件，属于airpcap库。这个头文件描述了两个函数：airpcap_oid_set_request和airpcap_sendqueue_transmit。

1. airpcap_oid_set_request函数：

这个函数的作用是设置OID（Object Identifier）的值。OID是dpdk中的一个数据类型，用于标识设备、协议头或其他实体。这个函数接受3个参数：p是一个指向airpcap实例的指针，oid是一个8位的无符号整数，data是一个指向数据缓冲区的指针，lenp是一个指向lenp变量（用于返回数据长度）的指针。

函数首先检查传入的oid值是否可以在airpcap适配器上设置，然后创建一个错误字符串，最后返回PCAP_ERROR（错误代码）。

2. airpcap_sendqueue_transmit函数：

这个函数的作用是尝试将队列中的数据发送到airpcap适配器上。它接受3个参数：p是一个指向airpcap实例的指针，queue是一个指向pcap_send_queue结构体的指针（用于存储数据发送队列），sync是一个同步标志，表示是否同步发送数据。

函数首先创建一个错误字符串，然后尝试将queue中的数据正确发送。如果发送失败，函数将返回0。


```cpp
static int
airpcap_oid_set_request(pcap_t *p, bpf_u_int32 oid _U_, const void *data _U_,
    size_t *lenp _U_)
{
	snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
	    "Setting OID values is not supported on an AirPcap adapter");
	return (PCAP_ERROR);
}

static u_int
airpcap_sendqueue_transmit(pcap_t *p, pcap_send_queue *queue _U_, int sync _U_)
{
	snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
	    "Cannot queue packets for transmission on an AirPcap adapter");
	return (0);
}

```

这段代码是一个用于设置用户数据缓冲区的函数，其功能是将一个用户提供的缓冲区申请下来，并将其赋值给一个名为 p 的指针变量，同时将该缓冲区的长度赋值给一个名为 size 的整型变量。

函数的实现中，首先对传入的大小参数进行检查，如果大小为 0 或负数，则输出一个错误信息并返回 -1，这表明函数的输入参数必须是正整数。

接着，如果缓冲区申请成功，则执行以下操作：

1. 在内存中为新分配一个大小为传入参数的大小的字符数组。
2. 将新分配的缓冲区赋值给 p->buffer，同时将 p->bufsize 变量赋值为传入参数的大小。
3. 返回 0，表明函数成功执行并返回 0，否则输出一个错误信息并返回 -1。


```cpp
static int
airpcap_setuserbuffer(pcap_t *p, int size)
{
	unsigned char *new_buff;

	if (size <= 0) {
		/* Bogus parameter */
		snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
		    "Error: invalid size %d",size);
		return (-1);
	}

	/* Allocate the buffer */
	new_buff = (unsigned char *)malloc(sizeof(char)*size);

	if (!new_buff) {
		snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
		    "Error: not enough memory");
		return (-1);
	}

	free(p->buffer);

	p->buffer = new_buff;
	p->bufsize = size;

	return (0);
}

```



这段代码定义了两个名为 `airpcap_live_dump` 和 `airpcap_live_dump_ended` 的函数，属于 `airpcap_live_dump` 函数内部。

`airpcap_live_dump` 函数的作用是输出错误信息并返回状态码。它接收一个指向 `pcap_t` 结构的指针 `p`、一个字符指针 `filename` 和两个整数 `maxsize` 和 `maxpacks`，表示要 dump 的数据包最大数量。函数先创建一个字符数组 `p->errbuf`，用于存储错误信息，然后使用 `snprintf` 函数将错误信息存储到字符数组中。最后，函数返回状态码 `-1`，表示失败并抛出异常。

`airpcap_live_dump_ended` 函数的作用是输出错误信息并结束函数。它与 `airpcap_live_dump` 函数不同，它没有参数。函数同样创建了一个字符数组 `p->errbuf`，用于存储错误信息，并使用 `snprintf` 函数将错误信息存储到字符数组中。最后，函数也返回状态码 `-1`，表示失败并抛出异常。


```cpp
static int
airpcap_live_dump(pcap_t *p, char *filename _U_, int maxsize _U_,
    int maxpacks _U_)
{
	snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
	    "AirPcap adapters don't support live dump");
	return (-1);
}

static int
airpcap_live_dump_ended(pcap_t *p, int sync _U_)
{
	snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
	    "AirPcap adapters don't support live dump");
	return (-1);
}

```

} while (cc < p_AirpcapRead(pa->adapter, (PBYTE)p->buffer,
		p->bufsize, &bytes_read)) {
	int i, j;

	datap = (u_char *)p->buffer + cc;

	for (i = 0; i < bytes_read; i++) {
		datap[i] = *datap[i];
	}

	int n = cc - i;

	if (n < 0) {
		n = 0;
		datap[i] = 0;
	}

	if (!p_AirpcapRead(pa->adapter, (PBYTE)datap,
			n, &bytes_read)) {
		snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
			"AirpcapRead() failed: %s",
			p_AirpcapGetLastError(pa->adapter));
			return (-1);
		}
	}

	cc = bytes_read;

	for (j = 0; j < n; j++) {
		int k;

		if (datap[j] == 0) {
			if (cc < p_AirpcapRead(pa->adapter,
					(PBYTE)datap + j,
					n - 1, &bytes_read)) {
					snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
						"AirpcapRead() failed: %s",
						p_AirpcapGetLastError(pa->adapter));
						return (-1);
					}

					cc = bytes_read;
				}
			} else {
					break;
				}
		} else {
				int k;

				datap[j] = 0;

				if (cc < p_AirpcapRead(pa->adapter,
						(PBYTE)datap + j,
						n - 1, &bytes_read)) {
						snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
							"AirpcapRead() failed: %s",
							p_AirpcapGetLastError(pa->adapter));
							return (-1);
						}

					cc = bytes_read;
				}
			}
		}
	}

	return (0);
}


```cpp
static PAirpcapHandle
airpcap_get_airpcap_handle(pcap_t *p)
{
	struct pcap_airpcap *pa = p->priv;

	return (pa->adapter);
}

static int
airpcap_read(pcap_t *p, int cnt, pcap_handler callback, u_char *user)
{
	struct pcap_airpcap *pa = p->priv;
	int cc;
	int n;
	register u_char *bp, *ep;
	UINT bytes_read;
	u_char *datap;

	cc = p->cc;
	if (cc == 0) {
		/*
		 * Has "pcap_breakloop()" been called?
		 */
		if (p->break_loop) {
			/*
			 * Yes - clear the flag that indicates that it
			 * has, and return PCAP_ERROR_BREAK to indicate
			 * that we were told to break out of the loop.
			 */
			p->break_loop = 0;
			return (PCAP_ERROR_BREAK);
		}

		//
		// If we're not in non-blocking mode, wait for data to
		// arrive.
		//
		if (pa->read_timeout != -1) {
			WaitForSingleObject(pa->read_event,
			    (pa->read_timeout ==0 )? INFINITE: pa->read_timeout);
		}

		//
		// Read the data.
		// p_AirpcapRead doesn't block.
		//
		if (!p_AirpcapRead(pa->adapter, (PBYTE)p->buffer,
		    p->bufsize, &bytes_read)) {
			snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
			    "AirpcapRead() failed: %s",
			    p_AirpcapGetLastError(pa->adapter));
			return (-1);
		}
		cc = bytes_read;
		bp = (u_char *)p->buffer;
	} else
		bp = p->bp;

	/*
	 * Loop through each packet.
	 *
	 * This assumes that a single buffer of packets will have
	 * <= INT_MAX packets, so the packet count doesn't overflow.
	 */
```

This function appears to evaluate the Short circuitry of a packet in a network, using the information in the packet and the information about the interface. It does so by evaluating the packet for certain fields, such as the Community, the Flags, and the Fragment, and by using the information about the interface to determine if the packet is allowed to be sent or if it needs to be discarded.

It appears that the function starts by evaluating the packet for the purposes of Short circuitry. If the packet is allowed to proceed, it is passed to the callback function, which evaluates the packet and returns an error code or a result. If the packet is not allowed to proceed, the function returns an error code.


```cpp
#define bhp ((AirpcapBpfHeader *)bp)
	n = 0;
	ep = bp + cc;
	for (;;) {
		register u_int caplen, hdrlen;

		/*
		 * Has "pcap_breakloop()" been called?
		 * If so, return immediately - if we haven't read any
		 * packets, clear the flag and return PCAP_ERROR_BREAK
		 * to indicate that we were told to break out of the loop,
		 * otherwise leave the flag set, so that the *next* call
		 * will break out of the loop without having read any
		 * packets, and return the number of packets we've
		 * processed so far.
		 */
		if (p->break_loop) {
			if (n == 0) {
				p->break_loop = 0;
				return (PCAP_ERROR_BREAK);
			} else {
				p->bp = bp;
				p->cc = (int) (ep - bp);
				return (n);
			}
		}
		if (bp >= ep)
			break;

		caplen = bhp->Caplen;
		hdrlen = bhp->Hdrlen;
		datap = bp + hdrlen;
		/*
		 * Short-circuit evaluation: if using BPF filter
		 * in the AirPcap adapter, no need to do it now -
		 * we already know the packet passed the filter.
		 */
		if (pa->filtering_in_kernel ||
		    p->fcode.bf_insns == NULL ||
		    pcap_filter(p->fcode.bf_insns, datap, bhp->Originallen, caplen)) {
			struct pcap_pkthdr pkthdr;

			pkthdr.ts.tv_sec = bhp->TsSec;
			pkthdr.ts.tv_usec = bhp->TsUsec;
			pkthdr.caplen = caplen;
			pkthdr.len = bhp->Originallen;
			(*callback)(user, &pkthdr, datap);
			bp += AIRPCAP_WORDALIGN(caplen + hdrlen);
			if (++n >= cnt && !PACKET_COUNT_IS_UNLIMITED(cnt)) {
				p->bp = bp;
				p->cc = (int)(ep - bp);
				return (n);
			}
		} else {
			/*
			 * Skip this packet.
			 */
			bp += AIRPCAP_WORDALIGN(caplen + hdrlen);
		}
	}
```

这段代码是一个名为"airpcap_inject"的函数，属于"pcap_t"类的成员函数。它的作用是接收一个"pcap_airpcap"结构体类型的参数(指针变量pa)，并返回从"p_AirpcapWrite"函数中发送的数据字节数。

具体来说，代码首先定义了一个名为"bhp"的未定义变量，并将其值设为0。接着，定义了一个名为"p->cc"的局部变量，并将其值设为从"p_AirpcapWrite"函数中返回的数据字节数。最后，函数本身返回这个局部变量的值，但需要满足一个限制条件：如果"p_AirpcapWrite"函数失败，则需要将"p->errbuf"中的错误字符串打印出来，并返回-1。

从代码中可以看出，这个函数主要是用于在"pcap_t"结构体中注入数据包。它将一个"pcap_airpcap"结构体类型的数据作为输入参数，并将其发送给"p_AirpcapWrite"函数。如果函数成功执行，它将返回数据的大小，否则将抛出错误信息。


```cpp
#undef bhp
	p->cc = 0;
	return (n);
}

static int
airpcap_inject(pcap_t *p, const void *buf, int size)
{
	struct pcap_airpcap *pa = p->priv;

	/*
	 * XXX - the second argument to AirpcapWrite() *should* have
	 * been declared as a const pointer - a write function that
	 * stomps on what it writes is *extremely* rude - but such
	 * is life.  We assume it is, in fact, not going to write on
	 * our buffer.
	 */
	if (!p_AirpcapWrite(pa->adapter, (void *)buf, size)) {
		snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
		    "AirpcapWrite() failed: %s",
		    p_AirpcapGetLastError(pa->adapter));
		return (-1);
	}

	/*
	 * We assume it all got sent if "AirpcapWrite()" succeeded.
	 * "pcap_inject()" is expected to return the number of bytes
	 * sent.
	 */
	return (size);
}

```



这段代码定义了两个函数，一个是 `airpcap_cleanup()`，另一个是 `airpcap_breakloop()`。它们都属于 `pcap_t` 结构体，用于管理 airpcap 套接字。

1. `airpcap_cleanup()` 函数的作用是清理 airpcap 套接字，包括关闭任何打开的 airpcap 设备，并释放之前分配的内存。它的参数是一个指向 `pcap_t` 结构的指针 `p`。

2. `airpcap_breakloop()` 函数的作用是中断所有的 airpcap 读取操作，直到读取操作成功为止。它的参数也是一个指向 `pcap_t` 结构的指针 `p`。

这两个函数都有可能抛出异常并返回错误码。需要根据实际需要来选择正确的调用其中一个函数。


```cpp
static void
airpcap_cleanup(pcap_t *p)
{
	struct pcap_airpcap *pa = p->priv;

	if (pa->adapter != NULL) {
		p_AirpcapClose(pa->adapter);
		pa->adapter = NULL;
	}
	pcap_cleanup_live_common(p);
}

static void
airpcap_breakloop(pcap_t *p)
{
	HANDLE read_event;

	pcap_breakloop_common(p);
	struct pcap_airpcap *pa = p->priv;

	/* XXX - what if either of these fail? */
	/*
	 * XXX - will SetEvent() force a wakeup and, if so, will
	 * the AirPcap read code handle that sanely?
	 */
	if (!p_AirpcapGetReadEvent(pa->adapter, &read_event))
		return;
	SetEvent(read_event);
}

```

This appears to be a code snippet for a software-defined radio (SDR) device. It appears to define some constants and pointers that are used to configure the device, such as the supported data link types and the maximum number of filtering operations that can be performed on the device. It also appears to define some functions that are used to perform common tasks, such as reading data from the air interface, injecting data into the air interface, and setting the device's filtering parameters. It looks like this code snippet is intended to be used by a software developer who needs to understand how to use the SDR device and implement their own custom filtering and functionality.


```cpp
static int
airpcap_activate(pcap_t *p)
{
	struct pcap_airpcap *pa = p->priv;
	char *device = p->opt.device;
	char airpcap_errbuf[AIRPCAP_ERRBUF_SIZE];
	BOOL status;
	AirpcapLinkType link_type;

	pa->adapter = p_AirpcapOpen(device, airpcap_errbuf);
	if (pa->adapter == NULL) {
		snprintf(p->errbuf, PCAP_ERRBUF_SIZE, "%s", airpcap_errbuf);
		return (PCAP_ERROR);
	}

	/*
	 * Set monitor mode appropriately.
	 * Always turn off the "ACK frames sent to the card" mode.
	 */
	if (p->opt.rfmon) {
		status = p_AirpcapSetDeviceMacFlags(pa->adapter,
		    AIRPCAP_MF_MONITOR_MODE_ON);
	} else
		status = p_AirpcapSetDeviceMacFlags(pa->adapter,
		    AIRPCAP_MF_ACK_FRAMES_ON);
	if (!status) {
		p_AirpcapClose(pa->adapter);
		snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
		    "AirpcapSetDeviceMacFlags() failed: %s",
		    p_AirpcapGetLastError(pa->adapter));
		return (PCAP_ERROR);
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
	 * If the buffer size wasn't explicitly set, default to
	 * AIRPCAP_DEFAULT_KERNEL_BUFFER_SIZE.
	 */
	if (p->opt.buffer_size == 0)
		p->opt.buffer_size = AIRPCAP_DEFAULT_KERNEL_BUFFER_SIZE;

	if (!p_AirpcapSetKernelBuffer(pa->adapter, p->opt.buffer_size)) {
		snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
		    "AirpcapSetKernelBuffer() failed: %s",
		    p_AirpcapGetLastError(pa->adapter));
		goto bad;
	}

	if(!p_AirpcapGetReadEvent(pa->adapter, &pa->read_event)) {
		snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
		    "AirpcapGetReadEvent() failed: %s",
		    p_AirpcapGetLastError(pa->adapter));
		goto bad;
	}

	/* Set the buffer size */
	p->bufsize = AIRPCAP_DEFAULT_USER_BUFFER_SIZE;
	p->buffer = malloc(p->bufsize);
	if (p->buffer == NULL) {
		pcap_fmt_errmsg_for_errno(p->errbuf, PCAP_ERRBUF_SIZE,
		    errno, "malloc");
		goto bad;
	}

	if (p->opt.immediate) {
		/* Tell the driver to copy the buffer as soon as data arrives. */
		if (!p_AirpcapSetMinToCopy(pa->adapter, 0)) {
			snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
			    "AirpcapSetMinToCopy() failed: %s",
			    p_AirpcapGetLastError(pa->adapter));
			goto bad;
		}
	} else {
		/*
		 * Tell the driver to copy the buffer only if it contains
		 * at least 16K.
		 */
		if (!p_AirpcapSetMinToCopy(pa->adapter, 16000)) {
			snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
			    "AirpcapSetMinToCopy() failed: %s",
			    p_AirpcapGetLastError(pa->adapter));
			goto bad;
		}
	}

	/*
	 * Find out what the default link-layer header type is,
	 * and set p->datalink to that.
	 *
	 * We don't force it to another value because there
	 * might be some programs using WinPcap/Npcap that,
	 * when capturing on AirPcap devices, assume the
	 * default value set with the AirPcap configuration
	 * program is what you get.
	 *
	 * The out-of-the-box default appears to be radiotap.
	 */
	if (!p_AirpcapGetLinkType(pa->adapter, &link_type)) {
		/* That failed. */
		snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
		    "AirpcapGetLinkType() failed: %s",
		    p_AirpcapGetLastError(pa->adapter));
		goto bad;
	}
	switch (link_type) {

	case AIRPCAP_LT_802_11_PLUS_RADIO:
		p->linktype = DLT_IEEE802_11_RADIO;
		break;

	case AIRPCAP_LT_802_11_PLUS_PPI:
		p->linktype = DLT_PPI;
		break;

	case AIRPCAP_LT_802_11:
		p->linktype = DLT_IEEE802_11;
		break;

	case AIRPCAP_LT_UNKNOWN:
	default:
		/* OK, what? */
		snprintf(p->errbuf, PCAP_ERRBUF_SIZE,
		    "AirpcapGetLinkType() returned unknown link type %u",
		    link_type);
		goto bad;
	}

	/*
	 * Now provide a list of all the supported types; we
	 * assume they all work.  We put radiotap at the top,
	 * followed by PPI, followed by "no radio metadata".
	 */
	p->dlt_list = (u_int *) malloc(sizeof(u_int) * 3);
	if (p->dlt_list == NULL)
		goto bad;
	p->dlt_list[0] = DLT_IEEE802_11_RADIO;
	p->dlt_list[1] = DLT_PPI;
	p->dlt_list[2] = DLT_IEEE802_11;
	p->dlt_count = 3;

	p->read_op = airpcap_read;
	p->inject_op = airpcap_inject;
	p->setfilter_op = airpcap_setfilter;
	p->setdirection_op = NULL;	/* Not implemented. */
	p->set_datalink_op = airpcap_set_datalink;
	p->getnonblock_op = airpcap_getnonblock;
	p->setnonblock_op = airpcap_setnonblock;
	p->breakloop_op = airpcap_breakloop;
	p->stats_op = airpcap_stats;
	p->stats_ex_op = airpcap_stats_ex;
	p->setbuff_op = airpcap_setbuff;
	p->setmode_op = airpcap_setmode;
	p->setmintocopy_op = airpcap_setmintocopy;
	p->getevent_op = airpcap_getevent;
	p->oid_get_request_op = airpcap_oid_get_request;
	p->oid_set_request_op = airpcap_oid_set_request;
	p->sendqueue_transmit_op = airpcap_sendqueue_transmit;
	p->setuserbuffer_op = airpcap_setuserbuffer;
	p->live_dump_op = airpcap_live_dump;
	p->live_dump_ended_op = airpcap_live_dump_ended;
	p->get_airpcap_handle_op = airpcap_get_airpcap_handle;
	p->cleanup_op = airpcap_cleanup;

	return (0);
 bad:
	airpcap_cleanup(p);
	return (PCAP_ERROR);
}

```

这段代码定义了一个名为`airpcap_can_set_rfmon`的函数，其作用是返回一个表示`pcap_t`结构体是否支持无线网络监控（Monitor）的整数。

接下来定义了一个名为`device_is_airpcap`的函数，该函数接受一个设备名称和一个字符数组，然后判断该设备是否为无线网络监控设备。其逻辑是首先检查设备是否以"/dev/airpcap"的格式开始，如果是，则设备是一台无线网络监控设备，返回1；否则，不是无线网络监控设备，返回0。

该代码的背景是 Linux 操作系统中，`pcap`库用于网络数据包抓取，而`airpcap`是一个支持无线网络监控的库，可以在捕捉无线网络数据包的同时，显示捕获的详细信息。


```cpp
/*
 * Monitor mode is supported.
 */
static int
airpcap_can_set_rfmon(pcap_t *p)
{
	return (1);
}

int
device_is_airpcap(const char *device, char *ebuf)
{
	static const char airpcap_prefix[] = "\\\\.\\airpcap";

	/*
	 * We don't determine this by calling AirpcapGetDeviceList()
	 * and looking at the list, as that appears to be a costly
	 * operation.
	 *
	 * Instead, we just check whether it begins with "\\.\airpcap".
	 */
	if (strncmp(device, airpcap_prefix, sizeof airpcap_prefix - 1) == 0) {
		/*
		 * Yes, it's an AirPcap device.
		 */
		return (1);
	}

	/*
	 * No, it's not an AirPcap device.
	 */
	return (0);
}

```

这段代码定义了一个名为`PCAP_CREATE_COMMON`的函数，它接收三个参数：

1. `ebuf`：这是一个字符指针，指向一个`struct pcap_airpcap`结构体。这个结构体可能用于存储一些关于设备的信息。
2. `is_ours`：这是一个整数变量，用于存储设备是否为AirPcap。
3. `p`：这是一个指向`pcap_t`类型的指针，用于存储创建的新的`pcap_t`结构体。

函数的作用是判断设备是否为AirPcap，如果是，则执行以下操作：

1. 如果设备已经被加载了AirPcap库，则直接返回。
2. 如果设备不是AirPcap，则检查设备是否支持AirPcap，如果是，则执行以下操作：

	1. 检查`is_ours`是否为0，如果是，则认为设备不是AirPcap，返回一个指向`NULL`的指针。
2. 如果`is_ours`为1，则执行以下操作：

		1. 尝试调用`load_airpcap_functions()`函数来加载AirPcap库。如果函数成功加载库，则`PCAP_CREATE_COMMON`函数可以正常返回。

这段代码的作用是用于在Linux系统上创建一个新的`PCAP_CREATE_COMMON`函数，该函数用于创建一个新的`pcap_t`结构体，以便后续的操作。


```cpp
pcap_t *
airpcap_create(const char *device, char *ebuf, int *is_ours)
{
	int ret;
	pcap_t *p;

	/*
	 * This can be called before we've tried loading the library,
	 * so do so if we haven't already tried to do so.
	 */
	if (load_airpcap_functions() != AIRPCAP_API_LOADED) {
		/*
		 * We assume this means that we don't have the AirPcap
		 * software installed, which probably means we don't
		 * have an AirPcap device.
		 *
		 * Don't treat that as an error.
		 */
		*is_ours = 0;
		return (NULL);
	}

	/*
	 * Is this an AirPcap device?
	 */
	ret = device_is_airpcap(device, ebuf);
	if (ret == 0) {
		/* No. */
		*is_ours = 0;
		return (NULL);
	}

	/*
	 * Yes.
	 */
	*is_ours = 1;
	p = PCAP_CREATE_COMMON(ebuf, struct pcap_airpcap);
	if (p == NULL)
		return (NULL);

	p->activate_op = airpcap_activate;
	p->can_set_rfmon_op = airpcap_can_set_rfmon;
	return (p);
}

```

这段代码是一个名为`airpcap_findalldevs`的函数，它的作用是检查系统上可用的所有AirPcap设备并返回它们的数量。函数的实现中，首先调用`load_airpcap_functions`函数，如果这个函数成功加载了AirPcap库，那么就可以开始使用AirPcap设备了。接下来，调用`p_AirpcapGetDeviceList`函数获取所有可用的AirPcap设备，并将它们存储在一个`AirpcapDeviceDescription`结构体中。然后，遍历所有的设备，调用`add_dev`函数将每个设备添加到`devlistp`中，如果某个设备无法添加或者添加失败，会在错误日志中记录信息并返回-1。最后，释放使用过的设备列表并返回0。


```cpp
/*
 * Add all AirPcap devices.
 */
int
airpcap_findalldevs(pcap_if_list_t *devlistp, char *errbuf)
{
	AirpcapDeviceDescription *airpcap_devices, *airpcap_device;
	char airpcap_errbuf[AIRPCAP_ERRBUF_SIZE];

	/*
	 * This can be called before we've tried loading the library,
	 * so do so if we haven't already tried to do so.
	 */
	if (load_airpcap_functions() != AIRPCAP_API_LOADED) {
		/*
		 * XXX - unless the error is "no such DLL", report this
		 * as an error rather than as "no AirPcap devices"?
		 */
		return (0);
	}

	if (!p_AirpcapGetDeviceList(&airpcap_devices, airpcap_errbuf)) {
		snprintf(errbuf, PCAP_ERRBUF_SIZE,
		    "AirpcapGetDeviceList() failed: %s", airpcap_errbuf);
		return (-1);
	}

	for (airpcap_device = airpcap_devices; airpcap_device != NULL;
	    airpcap_device = airpcap_device->next) {
		if (add_dev(devlistp, airpcap_device->Name, 0,
		    airpcap_device->Description, errbuf) == NULL) {
			/*
			 * Failure.
			 */
			p_AirpcapFreeDeviceList(airpcap_devices);
			return (-1);
		}
	}
	p_AirpcapFreeDeviceList(airpcap_devices);
	return (0);
}

```