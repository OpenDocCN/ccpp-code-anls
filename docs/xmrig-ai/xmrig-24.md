# xmrig源码解析 24

# `src/3rdparty/hwloc/include/hwloc/nvml.h`

这段代码定义了一个名为 "hwloc_nvml.h" 的头文件，其中定义了一些宏，用于帮助在 hwloc 和 NVIDIA 管理库之间进行交互。这些宏将提供 topology 信息，以便在 NVML 设备上获取。具体来说，这些宏将允许从 hwloc 获取有关 NVML 设备的 topology 信息，并在需要时将这些信息传递给 NVIDIA 管理库。


```cpp
/*
 * Copyright © 2012-2021 Inria.  All rights reserved.
 * See COPYING in top-level directory.
 */

/** \file
 * \brief Macros to help interaction between hwloc and the NVIDIA Management Library.
 *
 * Applications that use both hwloc and the NVIDIA Management Library may want to
 * include this file so as to get topology information for NVML devices.
 */

#ifndef HWLOC_NVML_H
#define HWLOC_NVML_H

```

这段代码是一个C++程序，它包括了几个头文件：hwloc.h，hwloc/autogen/config.h，hwloc/helper.h，和nvml.h。它还包含两个预处理指令：#ifdef和#define，它们用于检查源文件是否定义了某些特定的头文件或函数。

程序的主要作用是定义了一个名为"my_struct"的的结构体，并提供了对它的定义。这个结构体包括两个成员：一个整型变量和一个浮点型成员。

此外，程序还定义了一个名为"my_func"的函数，它接受一个整型和一个浮点型两个参数，并返回一个字符串。这个函数是在头文件hwloc/helper.h中定义的，因此它必须是定义在hwloc.h中的。程序没有对这个函数进行具体的实现，只是声明了它的存在。

最后，程序还包括了两个预处理指令：#ifdef和#define。它们的含义如下：

#ifdef HWLOC_LINUX_SYS
#include "hwloc/linux.h"
#endif

#ifdef __cplusplus
extern "C" {
#endif

这两个预处理指令分别检查两个不同的预处理标识符（#ifdef和#define）是否被定义。如果它们都被定义了，那么程序将会编译包括这个头文件或者函数声明的区域，否则将不会编译。


```cpp
#include "hwloc.h"
#include "hwloc/autogen/config.h"
#include "hwloc/helper.h"
#ifdef HWLOC_LINUX_SYS
#include "hwloc/linux.h"
#endif

#include <nvml.h>


#ifdef __cplusplus
extern "C" {
#endif


```

这段代码定义了一个名为`hwlocality_nvml`的函数组，用于与NVIDIA管理库（NVML）进行互操作。这个函数组包括两个函数：`get_device_topology`和`get_device_topology_os`。下面是这两个函数的详细解释：

1. `get_device_topology`函数：

这个函数的作用是获取一个设备（Device）的CPU集合。它通过`device_get_topology`函数获取，该函数会返回一个指向`hwlocality_device_topology_t`结构的指针。这个结构包含设备的物理位置、设备ID和CPU集合等信息。然后，通过一些NVML的封装函数（如`hwlocality_nvml_device_get_device_properties`和`hwlocality_nvml_device_topology_get_device_topology`）获取设备的位置和CPU集合，并将它们存储在`set_device_topology`函数中，以便在需要时进行调用。

1. `get_device_topology_os`函数：

这个函数的作用是获取设备（Device）的CPU集合，与`get_device_topology`函数不同的是，它使用了操作系统（OS）对象（如`pthread_self_so_()`函数）来获取device的CPU集合。这个函数将返回一个`unsigned long long`类型的设备ID，表示设备的CPU集合。这个函数在`hwlocality_nvml_get_device_osdev_by_index`函数中用于获取指定设备的CPU集合，同时也需要在topology中包含I/O设备检测，以确保在获取device的CPU集合时可以正确地获取到相关信息。

总结：

`hwlocality_nvml`函数组为研究NVIDIA管理库（NVML）提供了获取设备（Device）的topology信息的方法。这个函数组包含两个函数，`get_device_topology`用于获取device的CPU集合，而`get_device_topology_os`则需要通过操作系统对象来获取device的CPU集合。


```cpp
/** \defgroup hwlocality_nvml Interoperability with the NVIDIA Management Library
 *
 * This interface offers ways to retrieve topology information about
 * devices managed by the NVIDIA Management Library (NVML).
 *
 * @{
 */

/** \brief Get the CPU set of processors that are physically
 * close to NVML device \p device.
 *
 * Store in \p set the CPU-set describing the locality of the NVML device \p device.
 *
 * Topology \p topology and device \p device must match the local machine.
 * I/O devices detection and the NVML component are not needed in the topology.
 *
 * The function only returns the locality of the device.
 * If more information about the device is needed, OS objects should
 * be used instead, see hwloc_nvml_get_device_osdev()
 * and hwloc_nvml_get_device_osdev_by_index().
 *
 * This function is currently only implemented in a meaningful way for
 * Linux; other systems will simply get a full cpuset.
 */
```

这段代码的作用是获取一个硬件设备（如CPU套）的所有可用的CPU核心集合，并将它们注册到指定的CPU套中。在Linux系统上，通过使用sysfs机制获取本地CPU列表。

具体来说，代码首先检查要查询的系统是否为Linux，如果是，则使用sysfs机制获取设备local_cpus的路径。接着，代码使用nvmlDeviceGetPciInfo函数获取要查询的设备的PCI信息，并使用sprintf函数将PCI信息转换为字符串，以便在Linux系统上使用sysfs函数获取local_cpus。然后，代码使用hwloc_linux_read_path_as_cpumask函数将字符串路径的CPU套复制到给定的CPU套中。最后，如果需要，代码使用hwloc_bitmap_iszero函数将不在CPU套中的CPU套清空。


```cpp
static __hwloc_inline int
hwloc_nvml_get_device_cpuset(hwloc_topology_t topology __hwloc_attribute_unused,
			     nvmlDevice_t device, hwloc_cpuset_t set)
{
#ifdef HWLOC_LINUX_SYS
  /* If we're on Linux, use the sysfs mechanism to get the local cpus */
#define HWLOC_NVML_DEVICE_SYSFS_PATH_MAX 128
  char path[HWLOC_NVML_DEVICE_SYSFS_PATH_MAX];
  nvmlReturn_t nvres;
  nvmlPciInfo_t pci;

  if (!hwloc_topology_is_thissystem(topology)) {
    errno = EINVAL;
    return -1;
  }

  nvres = nvmlDeviceGetPciInfo(device, &pci);
  if (NVML_SUCCESS != nvres) {
    errno = EINVAL;
    return -1;
  }

  sprintf(path, "/sys/bus/pci/devices/%04x:%02x:%02x.0/local_cpus", pci.domain, pci.bus, pci.device);
  if (hwloc_linux_read_path_as_cpumask(path, set) < 0
      || hwloc_bitmap_iszero(set))
    hwloc_bitmap_copy(set, hwloc_topology_get_complete_cpuset(topology));
```

这段代码是一个C语言函数，名为`get_nvml_device()`。其作用是获取一个NVML设备对象的操作系统设备对象，该设备对象的索引由参数`idx`指定。如果没有找到相应的设备对象，函数返回`NULL`。

具体来说，这段代码首先定义了一个`hwloc_bitmap_copy()`函数，用于将一个硬件设备设置的`hwloc_topology_get_complete_cpuset()`函数返回的完整CPU集复制到一个`hwloc_bitmap`硬件设备设置中。接着，定义了一个内联函数`get_nvml_device()`，该函数首先尝试从当前机器的`hwloc_topology`中查找NVML设备，如果找不到相应的设备，则返回`NULL`。如果`topology`是一个有效的`hwloc_topology`，则`get_nvml_device()`将尝试获取一个NVML设备对象，并返回相应的设备对象。

如果`topology`的`I/O_device_detection`和`NVML_COMPONENT`设置被激活，则`get_nvml_device()`将尝试从`hwloc_device_set_device_data()`函数中获取与设备索引`idx`相对应的`hwloc_device_object()`，如果该函数返回的设备对象不存在，则返回`NULL`。


```cpp
#else
  /* Non-Linux systems simply get a full cpuset */
  hwloc_bitmap_copy(set, hwloc_topology_get_complete_cpuset(topology));
#endif
  return 0;
}

/** \brief Get the hwloc OS device object corresponding to the
 * NVML device whose index is \p idx.
 *
 * \return The hwloc OS device object describing the NVML device whose index is \p idx.
 * \return \c NULL if none could be found.
 *
 * The topology \p topology does not necessarily have to match the current
 * machine. For instance the topology may be an XML import of a remote host.
 * I/O devices detection and the NVML component must be enabled in the topology.
 *
 * \note The corresponding PCI device object can be obtained by looking
 * at the OS device parent object (unless PCI devices are filtered out).
 */
```

这段代码定义了一个名为 `hwloc_nvml_get_device_osdev_by_index` 的函数，它接受一个 `hwloc_topology_t` 的输入参数和一个 `unsigned int` 类型的索引。函数内部使用 `hwloc_get_next_osdev` 函数来查找与给定索引的 `osdev`，并且在找到相关设备时，使用设备属性中的 `osdev.type` 来判断是否为 NVML 设备。如果设备类型为 NVML 设备并且设备名称与 `"nvml"` 字符串的 ASCII 字符数量相等，那么函数将返回该设备对象。否则，函数返回 `NULL`。

该函数的作用是获取对应于给定 NVML 设备的 hwloc OS 设备对象。它通过 `hwloc_nvml_get_device_cpuset` 函数来查找与给定索引的 NVML 设备，并在找到相关设备时返回该设备对象。如果设备不是 NVML 设备，或者它在 `hwloc_nvml_get_device_cpuset` 函数中无法找到，那么函数将返回 `NULL`。


```cpp
static __hwloc_inline hwloc_obj_t
hwloc_nvml_get_device_osdev_by_index(hwloc_topology_t topology, unsigned idx)
{
	hwloc_obj_t osdev = NULL;
	while ((osdev = hwloc_get_next_osdev(topology, osdev)) != NULL) {
                if (HWLOC_OBJ_OSDEV_GPU == osdev->attr->osdev.type
                    && osdev->name
		    && !strncmp("nvml", osdev->name, 4)
		    && atoi(osdev->name + 4) == (int) idx)
                        return osdev;
        }
        return NULL;
}

/** \brief Get the hwloc OS device object corresponding to NVML device \p device.
 *
 * \return The hwloc OS device object that describes the given NVML device \p device.
 * \return \c NULL if none could be found.
 *
 * Topology \p topology and device \p device must match the local machine.
 * I/O devices detection and the NVML component must be enabled in the topology.
 * If not, the locality of the object may still be found using
 * hwloc_nvml_get_device_cpuset().
 *
 * \note The corresponding hwloc PCI device may be found by looking
 * at the result parent pointer (unless PCI devices are filtered out).
 */
```

This function appears to retrieve the NVMe device OSDev for a given NVMe device in a HWLOC topology. It does this by first checking if the topology is for a supported system, and then it retrieves the device information using the `nvmlDeviceGetPciInfo` and `nvmlDeviceGetUUID` functions. If the function fails to retrieve the device information, it returns NULL.

The function takes in two arguments: `hwloc_topology_t` and `nvmlDevice_t` as input, and returns an instance of `hwloc_obj_t`.


```cpp
static __hwloc_inline hwloc_obj_t
hwloc_nvml_get_device_osdev(hwloc_topology_t topology, nvmlDevice_t device)
{
	hwloc_obj_t osdev;
	nvmlReturn_t nvres;
	nvmlPciInfo_t pci;
	char uuid[64];

	if (!hwloc_topology_is_thissystem(topology)) {
		errno = EINVAL;
		return NULL;
	}

	nvres = nvmlDeviceGetPciInfo(device, &pci);
	if (NVML_SUCCESS != nvres)
		return NULL;

	nvres = nvmlDeviceGetUUID(device, uuid, sizeof(uuid));
	if (NVML_SUCCESS != nvres)
		uuid[0] = '\0';

	osdev = NULL;
	while ((osdev = hwloc_get_next_osdev(topology, osdev)) != NULL) {
		hwloc_obj_t pcidev = osdev->parent;
		const char *info;

		if (strncmp(osdev->name, "nvml", 4))
			continue;

		if (pcidev
		    && pcidev->type == HWLOC_OBJ_PCI_DEVICE
		    && pcidev->attr->pcidev.domain == pci.domain
		    && pcidev->attr->pcidev.bus == pci.bus
		    && pcidev->attr->pcidev.dev == pci.device
		    && pcidev->attr->pcidev.func == 0)
			return osdev;

		info = hwloc_obj_get_info_by_name(osdev, "NVIDIAUUID");
		if (info && !strcmp(info, uuid))
			return osdev;
	}

	return NULL;
}

```

这段代码是一个C语言中的一个头文件，包含了两个预处理指令。

第一个预处理指令是 `#ifdef __cplusplus`，它判断当前编译器是否支持C++语言的新特性。如果支持，那么编译器会编译 `__cplusplus` 后面的代码，将其作为可执行文件的一部分。如果不支持，则不会生成这个可执行文件。

第二个预处理指令是 `#include <stdio.h>`，它告诉编译器在编译之前包含标准输入输出库（stdio.h）。

综合来看，这段代码的作用是生成一个名为 `test.c` 的可执行文件，其中包含一个以 `__cplusplus` 开头的预处理指令。这个可执行文件是在支持C++语言的新特性时生成的。


```cpp
/** @} */


#ifdef __cplusplus
} /* extern "C" */
#endif


#endif /* HWLOC_NVML_H */

```

# `src/3rdparty/hwloc/include/hwloc/opencl.h`

这段代码定义了一个名为"hwloc_opencl.h"的文件，其中包含了使用hwloc和OpenCL进行交互的宏。

具体来说，这些宏提供了一些方便函数，帮助开发人员更好地与硬件抽象层(hwloc)和OpenCL进行交互。这些函数可以提供有关连接的设备，设备状态和设备属性等信息。

由于hwloc和OpenCL都可以在操作系统上提供更高层次的抽象，因此这些宏可以帮助开发人员更轻松地与OpenCL设备进行交互，而不必直接与硬件交互。


```cpp
/*
 * Copyright © 2012-2021 Inria.  All rights reserved.
 * Copyright © 2013, 2018 Université Bordeaux.  All right reserved.
 * See COPYING in top-level directory.
 */

/** \file
 * \brief Macros to help interaction between hwloc and the OpenCL interface.
 *
 * Applications that use both hwloc and OpenCL may want to
 * include this file so as to get topology information for OpenCL devices.
 */

#ifndef HWLOC_OPENCL_H
#define HWLOC_OPENCL_H

```



这段代码是一个C语言程序，它包括了几个头文件和一些必要的库。

其中包括：

- hwloc.h：这是硬件位置(hwloc)的库，提供了与硬件布局和布局相关的方法和数据结构。
- autogen.h：这是autogenerated的库，允许用户自定义程序的编译器和头文件。
- configuration.h：这是配置库的头部文件，定义了配置文件的语法和相关函数。
- helper.h：这是辅助函数的头部文件，定义了一些通用的函数，如strings,math和printf。
- linux：这是操作系统特定头文件的头部文件，定义了与Linux操作系统相关的函数和数据结构。
- opencl：这是OpenCL头文件的头部文件，定义了与OpenCL操作系统相关的函数和数据结构。

注意到在代码中使用了一些由其他库定义的函数和数据结构。这意味着程序需要链接到这些库，才能正常工作。


```cpp
#include "hwloc.h"
#include "hwloc/autogen/config.h"
#include "hwloc/helper.h"
#ifdef HWLOC_LINUX_SYS
#include "hwloc/linux.h"
#endif

#ifdef __APPLE__
#include <OpenCL/cl.h>
#else
#include <CL/cl.h>
#endif

#include <stdio.h>


```

这段代码定义了一个头文件#ifdef __cplusplus，其中包含了一个extern "C"声明，这意味着它是一个C语言编写的文件，但其中的某些函数和变量是C语言可移植的。

接下来是另一个头文件#endif，它声明了前面#ifdef __cplusplus中的所有函数和变量都使用这个头文件中的定义。

在函数体中，定义了一个名为HWLOC_CL_DEVICE_TOPOLOGY_AMD的宏，它的值为0x4037。它告诉编译器在编译时检查代码时，如果设备不是支持Cl客人大附中四层，或者设备不支持Cl几何，该宏将抛出错误。

接下来定义了一个名为raw的联合体类型，其中包含一个cl_uint类型的type和一个cl_uint类型的data数组，还有一个cl_char类型的unused和bus和device和function成员。

然后定义了一个名为pcie的联合体类型，其中包含一个cl_uint类型的type，一个cl_char类型的unused数组，一个cl_char类型的bus，一个cl_char类型的device和一个cl_char类型的function。

最后，给这个文件起了一个名为__attribute__((__norerange__, __n亲身奉行__))的名字，这样它就可以被声明为C语言或C语言扩展的函数或变量了。


```cpp
#ifdef __cplusplus
extern "C" {
#endif


/* OpenCL extensions aren't always shipped with default headers, and
 * they don't always reflect what the installed implementations support.
 * Try everything and let the implementation return errors when non supported.
 */
/* Copyright (c) 2008-2018 The Khronos Group Inc. */

/* needs "cl_amd_device_attribute_query" device extension, but not strictly required for clGetDeviceInfo() */
#define HWLOC_CL_DEVICE_TOPOLOGY_AMD 0x4037
typedef union {
    struct { cl_uint type; cl_uint data[5]; } raw;
    struct { cl_uint type; cl_char unused[17]; cl_char bus; cl_char device; cl_char function; } pcie;
} hwloc_cl_device_topology_amd;
```

这段代码定义了一系列常量，用于描述NVIDIA OpenCL设备的拓扑结构和属性。

首先，定义了一个名为`HWLOC_CL_DEVICE_TOPOLOGY_TYPE_PCIE_AMD`的常量，值为1，表示仅当设备是PCIe NVIDIA设备时才需要使用这个拓扑类型。

接着，定义了三个常量，分别表示NVIDIA OpenCL设备的PCI总线ID、PCI槽ID和内存区域ID。这些常量的值分别为0x4008、0x4009和0x400A。

最后，定义了一个名为`HWLOC_CL_DEVICE_PCI_DOMAIN_ID_NV`的常量，表示仅当设备是PCIe NVIDIA设备时才需要使用这个拓扑结构。

这个代码定义了一系列常量，用于描述NVIDIA OpenCL设备的拓扑结构和属性，仅在PCIe NVIDIA设备上有效。


```cpp
#define HWLOC_CL_DEVICE_TOPOLOGY_TYPE_PCIE_AMD 1

/* needs "cl_nv_device_attribute_query" device extension, but not strictly required for clGetDeviceInfo() */
#define HWLOC_CL_DEVICE_PCI_BUS_ID_NV 0x4008
#define HWLOC_CL_DEVICE_PCI_SLOT_ID_NV 0x4009
#define HWLOC_CL_DEVICE_PCI_DOMAIN_ID_NV 0x400A


/** \defgroup hwlocality_opencl Interoperability with OpenCL
 *
 * This interface offers ways to retrieve topology information about
 * OpenCL devices.
 *
 * Only AMD and NVIDIA OpenCL implementations currently offer useful locality
 * information about their devices.
 *
 * @{
 */

```

It looks like you are trying to provide a solution for a problem that does not seem to exist. The code you provided is not even close to being a valid answer to the problem you are trying to solve.

If you have a legitimate question or problem that you are trying to solve, please provide more information and code snippets so that I can better understand the problem and help you.


```cpp
/** \brief Return the domain, bus and device IDs of the OpenCL device \p device.
 *
 * Device \p device must match the local machine.
 */
static __hwloc_inline int
hwloc_opencl_get_device_pci_busid(cl_device_id device,
                               unsigned *domain, unsigned *bus, unsigned *dev, unsigned *func)
{
	hwloc_cl_device_topology_amd amdtopo;
	cl_uint nvbus, nvslot, nvdomain;
	cl_int clret;

	clret = clGetDeviceInfo(device, HWLOC_CL_DEVICE_TOPOLOGY_AMD, sizeof(amdtopo), &amdtopo, NULL);
	if (CL_SUCCESS == clret
	    && HWLOC_CL_DEVICE_TOPOLOGY_TYPE_PCIE_AMD == amdtopo.raw.type) {
		*domain = 0; /* can't do anything better */
		/* cl_device_topology_amd stores bus ID in cl_char, dont convert those signed char directly to unsigned int */
		*bus = (unsigned) (unsigned char) amdtopo.pcie.bus;
		*dev = (unsigned) (unsigned char) amdtopo.pcie.device;
		*func = (unsigned) (unsigned char) amdtopo.pcie.function;
		return 0;
	}

	clret = clGetDeviceInfo(device, HWLOC_CL_DEVICE_PCI_BUS_ID_NV, sizeof(nvbus), &nvbus, NULL);
	if (CL_SUCCESS == clret) {
		clret = clGetDeviceInfo(device, HWLOC_CL_DEVICE_PCI_SLOT_ID_NV, sizeof(nvslot), &nvslot, NULL);
		if (CL_SUCCESS == clret) {
			clret = clGetDeviceInfo(device, HWLOC_CL_DEVICE_PCI_DOMAIN_ID_NV, sizeof(nvdomain), &nvdomain, NULL);
			if (CL_SUCCESS == clret) { /* available since CUDA 10.2 */
				*domain = nvdomain;
			} else {
				*domain = 0;
			}
			*bus = nvbus & 0xff;
			/* non-documented but used in many other projects */
			*dev = nvslot >> 3;
			*func = nvslot & 0x7;
			return 0;
		}
	}

	return -1;
}

```

这段代码是一个 C 语言函数，名为 `hwloc_opencl_get_device_cpu_set()`，它的作用是获取一个物理上接近 OpenCL 设备的 CPU 集合，并将结果存储在 `device.CPUSet` 指向的 CPU 集上。

该函数的输入参数包括：

- `device`：描述 OpenCL 设备的数据结构，通常是由 hwloc_opencl_get_device() 函数返回的。这个函数需要指定设备的驱动程序和根句柄。
- `device_os_device`：描述 OpenCL 设备的操作系统设备数据，这个数据可以在 hwloc_opencl_get_device_osdev() 函数中获取。如果没有这个值，函数将无法获取物理 OpenCL 设备。

该函数的输出是一个字符串，描述了物理 OpenCL 设备的 CPU 集合，它将存储在 `device.CPUSet` 指向的内存位置上。

该函数的作用是为了解决在某些场景中，由于 OpenCL 设备与主机硬件之间的差异，需要使用操作系统提供的设备信息，来获取 OpenCL 设备的物理位置。在这种情况下，函数将返回设备的 CPU 集合，以供后续使用。


```cpp
/** \brief Get the CPU set of processors that are physically
 * close to OpenCL device \p device.
 *
 * Store in \p set the CPU-set describing the locality of the OpenCL device \p device.
 *
 * Topology \p topology and device \p device must match the local machine.
 * I/O devices detection and the OpenCL component are not needed in the topology.
 *
 * The function only returns the locality of the device.
 * If more information about the device is needed, OS objects should
 * be used instead, see hwloc_opencl_get_device_osdev()
 * and hwloc_opencl_get_device_osdev_by_index().
 *
 * This function is currently only implemented in a meaningful way for
 * Linux with the AMD or NVIDIA OpenCL implementation; other systems will simply
 * get a full cpuset.
 */
```

这段代码定义了一个名为 `hwloc_opencl_get_device_cpuset` 的函数，它是 `hwloc_topology_opencl_device_淺写` 的别名。函数接受三个参数：

1. `topology`：指向 `hwloc_topology_t` 类型的整数，用于在创建 `hwloc_device_t` 实例时指定底层硬件平台。如果不存在该硬件平台，函数将返回 -1。
2. `device`：指向 `cl_device_id` 类型的整数，用于指定 OpenGL 设备。如果不存在该设备，函数将返回。
3. `set`：指向 `hwloc_cpuset_t` 类型的整数，用于存储在系统中可用的所有 CPU 核心的 ID。

函数首先检查底层硬件平台是否存在，如果不存在，函数将返回 -1。然后，函数尝试使用 `hwloc_topology_get_device_pci_busid` 函数获取设备 PCI 总线 ID，如果失败，函数将使用 `hwloc_topology_get_complete_cpuset` 函数获取系统的所有 CPU 核心。接下来，函数使用 `sprintf` 函数将路径字符串初始化为 `/sys/bus/pci/devices/<device_pci_domain>/local_cpus`，其中 `<device_pci_domain>` 是设备 PCI 总线 ID。最后，函数使用 `hwloc_linux_read_path_as_cpumask` 函数将路径作为掩码，并从系统中读取所有可用 CPU 核心的 ID，如果成功，函数将返回完整的设置。如果路径存在，但所有核心都未被占用，函数将使用 `hwloc_topology_get_complete_cpuset` 函数获取系统的所有 CPU 核心。


```cpp
static __hwloc_inline int
hwloc_opencl_get_device_cpuset(hwloc_topology_t topology __hwloc_attribute_unused,
			       cl_device_id device __hwloc_attribute_unused,
			       hwloc_cpuset_t set)
{
#if (defined HWLOC_LINUX_SYS)
	/* If we're on Linux, try AMD/NVIDIA extensions + the sysfs mechanism to get the local cpus */
#define HWLOC_OPENCL_DEVICE_SYSFS_PATH_MAX 128
	char path[HWLOC_OPENCL_DEVICE_SYSFS_PATH_MAX];
	unsigned pcidomain, pcibus, pcidev, pcifunc;

	if (!hwloc_topology_is_thissystem(topology)) {
		errno = EINVAL;
		return -1;
	}

	if (hwloc_opencl_get_device_pci_busid(device, &pcidomain, &pcibus, &pcidev, &pcifunc) < 0) {
		hwloc_bitmap_copy(set, hwloc_topology_get_complete_cpuset(topology));
		return 0;
	}

	sprintf(path, "/sys/bus/pci/devices/%04x:%02x:%02x.%01x/local_cpus", pcidomain, pcibus, pcidev, pcifunc);
	if (hwloc_linux_read_path_as_cpumask(path, set) < 0
	    || hwloc_bitmap_iszero(set))
		hwloc_bitmap_copy(set, hwloc_topology_get_complete_cpuset(topology));
```

这段代码是一个 C 语言函数，名为 `get_opencl_device`。它用于获取一个 OpenCL 设备的设备对象。

函数的作用是：

1. 如果系统是 Linux，则使用 `hwloc_topology_get_complete_cpuset` 函数获取完整的 CPU 集，并将其作为输入参数传递给 `hwloc_bitmap_copy` 函数。这将复制整个 CPU 集，并将其存储在 `set` 指向的内存位置。
2. 如果系统是 non-Linux 系统，则使用 `hwloc_topology_get_complete_cpuset` 函数获取完整的 CPU 集，并将其作为输入参数传递给 `hwloc_bitmap_copy` 函数。对于 non-Linux 系统，该函数将直接返回一个空指针，因为这些系统通常不支持 OpenCL。

函数的返回值是一个整数，表示是否成功获取了 OpenCL 设备对象。如果成功，则返回 0；如果失败，则返回其他值。


```cpp
#else
	/* Non-Linux systems simply get a full cpuset */
	hwloc_bitmap_copy(set, hwloc_topology_get_complete_cpuset(topology));
#endif
  return 0;
}

/** \brief Get the hwloc OS device object corresponding to the
 * OpenCL device for the given indexes.
 *
 * \return The hwloc OS device object describing the OpenCL device
 * whose platform index is \p platform_index,
 * and whose device index within this platform if \p device_index.
 * \return \c NULL if there is none.
 *
 * The topology \p topology does not necessarily have to match the current
 * machine. For instance the topology may be an XML import of a remote host.
 * I/O devices detection and the OpenCL component must be enabled in the topology.
 *
 * \note The corresponding PCI device object can be obtained by looking
 * at the OS device parent object (unless PCI devices are filtered out).
 */
```

这段代码定义了一个名为 `hwloc_opencl_get_device_osdev_by_index` 的函数，属于 `hwloc_loc` 系列的函数。函数接受两个参数，一个是 `topology`，表示顶层的 HWLOC 结构体，另一个是 `device_index`，表示要查找的设备的索引。函数内部使用 `hwloc_topology_get_device_osdev` 函数获取设备的 OSDEV，然后通过 while 循环遍历所有 OSDEV。在循环中，首先检查当前 OSDEV 的类型是否为 `opencl` 并且是否有 `opencl` 开发者的名称。如果是，并且当前 OSDEV 的设备索引与 `topology` 中的设备索引相同，则返回当前 OSDEV。否则，返回 `NULL`。


```cpp
static __hwloc_inline hwloc_obj_t
hwloc_opencl_get_device_osdev_by_index(hwloc_topology_t topology,
				       unsigned platform_index, unsigned device_index)
{
	unsigned x = (unsigned) -1, y = (unsigned) -1;
	hwloc_obj_t osdev = NULL;
	while ((osdev = hwloc_get_next_osdev(topology, osdev)) != NULL) {
		if (HWLOC_OBJ_OSDEV_COPROC == osdev->attr->osdev.type
                    && osdev->name
		    && sscanf(osdev->name, "opencl%ud%u", &x, &y) == 2
		    && platform_index == x && device_index == y)
                        return osdev;
        }
        return NULL;
}

```

这段代码是一个名为`hwloc_opencl_get_device_osdev_by_index`的函数，它用于获取一个OpenCL设备的hwloc OS设备对象，对应于给定的OpenCL设备。如果没有找到相应的设备对象，函数将返回`NULL`。

该函数仅在支持相关OpenCL扩展的AMD和NVIDIA OpenCL设备上有效。在确定平台和设备索引后，应优先使用`hwloc_opencl_get_device_osdev_by_index()`函数，而不是使用此函数。

函数需要传递两个参数：`topology`和`device`，它们分别表示OpenCL设备的顶部和设备。函数返回一个指向OpenCL设备的hwloc OS设备对象的指针，或者`NULL`，如果无法找到相应的设备对象。

函数的一个附加限制是，它只能在支持相关OpenCL扩展的AMD和NVIDIA OpenCL设备上有效。如果在使用此函数的系统上，PCI设备被过滤了，则无法使用此函数。


```cpp
/** \brief Get the hwloc OS device object corresponding to OpenCL device \p deviceX.
 *
 * \return The hwloc OS device object corresponding to the given OpenCL device \p device.
 * \return \c NULL if none could be found, for instance
 * if required OpenCL attributes are not available.
 *
 * This function currently only works on AMD and NVIDIA OpenCL devices that support
 * relevant OpenCL extensions. hwloc_opencl_get_device_osdev_by_index()
 * should be preferred whenever possible, i.e. when platform and device index
 * are known.
 *
 * Topology \p topology and device \p device must match the local machine.
 * I/O devices detection and the OpenCL component must be enabled in the topology.
 * If not, the locality of the object may still be found using
 * hwloc_opencl_get_device_cpuset().
 *
 * \note This function cannot work if PCI devices are filtered out.
 *
 * \note The corresponding hwloc PCI device may be found by looking
 * at the result parent pointer (unless PCI devices are filtered out).
 */
```

这段代码定义了一个名为`hwloc_opencl_get_device_osdev`的函数，属于`hwloc_opencl_ex`库。它的作用是获取一个设备为opencl的hwloc_device_t对象，并返回该对象的`hwloc_obj_t`引用。函数的实现如下：

```cppc
static __hwloc_inline hwloc_obj_t
hwloc_opencl_get_device_osdev(hwloc_topology_t topology __hwloc_attribute_unused,
			      cl_device_id device __hwloc_attribute_unused)
{
	hwloc_obj_t osdev;
	unsigned pcidomain, pcibus, pcidev, pcifunc;

	if (hwloc_opencl_get_device_pci_busid(device, &pcidomain, &pcibus, &pcidev, &pcifunc) < 0) {
		errno = EINVAL;
		return NULL;
	}

	osdev = NULL;
	while ((osdev = hwloc_get_next_osdev(topology, osdev)) != NULL) {
		hwloc_obj_t pcidev = osdev->parent;
		if (strncmp(osdev->name, "opencl", 6))
			continue;
		if (pcidev
		    && pcidev->type == HWLOC_OBJ_PCI_DEVICE
		    && pcidev->attr->pcidev.domain == pcidomain
		    && pcidev->attr->pcidev.bus == pcibus
		    && pcidev->attr->pcidev.dev == pcidevice
		    && pcidev->attr->pcidev.func == pcifunc)
			return osdev;
		/* if PCI are filtered out, we need a info attr to match on */
	}

	return NULL;
}
```

函数首先定义了一个`hwloc_opencl_get_device_osdev`函数，它的参数包括两个`hwloc_topology_t`类型的参数，分别表示要查询的topology和目标设备的device。函数在内部定义了一个`hwloc_obj_t`类型的变量`osdev`，用于存储查询得到的device对象。

函数的实现主要分为两个步骤：

1. 检查设备是否为opencl，如果是，则直接返回`osdev`，否则继续查询下一个osdev。
2. 对于每个osdev，检查设备是否为PCI设备，如果是，并且其PCI domain与目标设备的PCI domain相同，则返回`osdev`。如果PCI domain被过滤出去了，函数会在返回前添加一个info attribute，用于告知hwloc进一步处理这个PCI device。

如果函数在查询过程中遇到困难，会返回`NULL`。


```cpp
static __hwloc_inline hwloc_obj_t
hwloc_opencl_get_device_osdev(hwloc_topology_t topology __hwloc_attribute_unused,
			      cl_device_id device __hwloc_attribute_unused)
{
	hwloc_obj_t osdev;
	unsigned pcidomain, pcibus, pcidevice, pcifunc;

	if (hwloc_opencl_get_device_pci_busid(device, &pcidomain, &pcibus, &pcidevice, &pcifunc) < 0) {
		errno = EINVAL;
		return NULL;
	}

	osdev = NULL;
	while ((osdev = hwloc_get_next_osdev(topology, osdev)) != NULL) {
		hwloc_obj_t pcidev = osdev->parent;
		if (strncmp(osdev->name, "opencl", 6))
			continue;
		if (pcidev
		    && pcidev->type == HWLOC_OBJ_PCI_DEVICE
		    && pcidev->attr->pcidev.domain == pcidomain
		    && pcidev->attr->pcidev.bus == pcibus
		    && pcidev->attr->pcidev.dev == pcidevice
		    && pcidev->attr->pcidev.func == pcifunc)
			return osdev;
		/* if PCI are filtered out, we need a info attr to match on */
	}

	return NULL;
}

```

这段代码是一个C语言的预处理指令，它主要用于编译预处理。具体来说，它包括以下几个部分：

1. `#ifdef __cplusplus`：这是一个条件编译语句，表示当编译器的版本支持C++语言时，编译这些预处理指令。这个指令告诉编译器，后面的代码块将使用C++语言的预处理定义。

2. `/* extern "C" */`：这是一个预处理指令，表示告诉编译器，后面的代码块将使用C语言的预处理定义。这个指令告诉编译器，它可以在编译过程中定义外部变量。

3. `#elif defined(__GNUC__) || defined(__APPLE__)`：这是一个条件编译语句，表示当编译器的版本不支持C++语言时，编译这些预处理指令。这个指令告诉编译器，它支持GNUC编译器和APPLE操作系统。

4. `#elif defined(__attribute__(`编译时特性`))`：这是一个条件编译语句，表示当编译器的版本支持编译时特性时，编译这些预处理指令。这个指令告诉编译器，它支持编译时特性。

5. `#include "hwloc_init.h"`：这是一个预处理指令，表示告诉编译器，它将使用hwloc库的初始化函数。这个库可能是一个用于管理硬件资源和初始化程序的库。

6. `#ifdef __cplusplus`：这是一个条件编译语句，表示当编译器的版本支持C++语言时，编译这些预处理指令。这个指令告诉编译器，后面的代码块将使用C++语言的预处理定义。

7. `#elif defined(__GNUC__) || defined(__APPLE__)`：这是一个条件编译语句，表示当编译器的版本不支持C++语言时，编译这些预处理指令。这个指令告诉编译器，它支持GNUC编译器和APPLE操作系统。

8. `#elif defined(__attribute__(`编译时特性`))`：这是一个条件编译语句，表示当编译器的版本支持编译时特性时，编译这些预处理指令。这个指令告诉编译器，它支持编译时特性。

9. `#include "hwloc_api.h"`：这是一个预处理指令，表示告诉编译器，它将使用hwloc库的API函数。这个库可能是一个用于管理硬件资源和初始化程序的库。


```cpp
/** @} */


#ifdef __cplusplus
} /* extern "C" */
#endif


#endif /* HWLOC_OPENCL_H */

```

# `src/3rdparty/hwloc/include/hwloc/openfabrics-verbs.h`

这段代码定义了一系列 macro，用于帮助在 hwloc 和 OpenFabrics 之间进行交互。这些宏可帮助开发人员了解他们正在使用的硬件资源，并获取关于 OpenFabrics 硬件的拓扑信息。以下是代码的主要部分：

```cpp
/**
* Copyright © 2009 CNRS
* Copyright © 2009-2021 Inria.  All rights reserved.
* Copyright © 2009-2010 Université Bordeaux
* Copyright © 2009-2011 Cisco Systems, Inc.  All rights reserved.
* See COPYING in top-level directory.
*/

#define HWLOC_FUNCTION_INFO "hwloc_function_info"
#define HWLOC_FUNCTION_LABEL "hwloc_function_label"
#define HWLOC_FUNCTION_DESCRIPTION "hwloc_function_description"

#define OCP_VERSION "opc_version"
#define OCP_BUILD_INFO "opc_build_info"
#define OCP_EXECUTABLE "opc_executable"

#define MAX_HWLOC_INSTances "max_hwloc_instances"
#define MAX_INFOBROWS "max_infobrows"
#define MAX_OPENFAT_SLOTS "max_openfatslots"
```

这些宏包括：

- `HWLOC_FUNCTION_INFO`：用于表示 hwloc 函数信息的宏。
- `HWLOC_FUNCTION_LABEL`：用于表示 hwloc 函数 label 的宏。
- `HWLOC_FUNCTION_DESCRIPTION`：用于表示 hwloc 函数描述信息的宏。
- `OCP_VERSION`：表示 OpenFabrics 操作平台（InfiniBand，等）的版本。
- `OCP_BUILD_INFO`：表示 OpenFabrics 操作平台版本的身份信息。
- `OCP_EXECUTABLE`：表示 OpenFabrics 操作平台可执行文件的名称。
- `MAX_HWLOC_INSTances`：表示支持的最大 hwloc 实例数。
- `MAX_INFOBROWS`：表示支持的最大的 infibrows 行数。
- `MAX_OPENFAT_SLOTS`：表示支持的最大的 OpenFabrics 槽位数。

这些宏的具体含义可以在编译时或运行时通过将它们与相应的函数名称联系起来来获得更具体的输出。例如，当使用 OpenFabrics 时，可以通过调用 `hwloc_create_instance` 函数来获取特定函数的 information，例如：
```cppperl
hwloc_info *hwloc_info_create(uint32_t instance);
```
然后，可以使用 `hwloc_info_get_all` 函数获取有关所有实例的信息：
```cppperl
hwloc_info *hwloc_info_get_all(void);
```
请注意，上述代码中的 `hwloc_create_instance`，`hwloc_info_get_all` 等函数在实现时可能会使用不同的函数名称或宏定义。


```cpp
/*
 * Copyright © 2009 CNRS
 * Copyright © 2009-2021 Inria.  All rights reserved.
 * Copyright © 2009-2010 Université Bordeaux
 * Copyright © 2009-2011 Cisco Systems, Inc.  All rights reserved.
 * See COPYING in top-level directory.
 */

/** \file
 * \brief Macros to help interaction between hwloc and OpenFabrics
 * verbs.
 *
 * Applications that use both hwloc and OpenFabrics verbs may want to
 * include this file so as to get topology information for OpenFabrics
 * hardware (InfiniBand, etc).
 *
 */

```

这段代码定义了一个头文件 "hwloc-opensupport-verbs.h"，其中包含了 HWLOC 库的一些定义。

首先，它导入了 HWLOC 库的头文件 "hwloc.h"，以及 "hwloc/autogen/config.h"。这两个头文件可能定义了一些全局函数和变量，但在这里我们并不关心，因为我们只关注 Verbs 函数。

然后，它通过 #ifdef 预处理指令检查是否支持在 Linux 系统上使用 HWLOC 库。如果是，那么它就会包含 HWLOC 库中定义的 Verbs 函数。

接下来，它引入了 Infiniband 库中的 Verbs 函数头文件 "infiniband/verbs.h"。

最后，它通过 #ifdef 预处理指令定义了一个名为 "my_verbs_handler" 的函数，该函数可能接受一些参数，然后使用 Verbs 函数执行操作。但在这里，我们并不关心这个函数，因为我们只关注头文件中的定义。


```cpp
#ifndef HWLOC_OPENFABRICS_VERBS_H
#define HWLOC_OPENFABRICS_VERBS_H

#include "hwloc.h"
#include "hwloc/autogen/config.h"
#ifdef HWLOC_LINUX_SYS
#include "hwloc/linux.h"
#endif

#include <infiniband/verbs.h>


#ifdef __cplusplus
extern "C" {
#endif


```

这段代码定义了一个名为 `hwlocality_openfabrics` 的函数组，用于与 OpenFabrics 进行互操作。函数组包括以下函数：

1. `__all__` 函数：声明该函数组中所有函数的 `__all__` 修饰符，使得这些函数可以被调用。
2. `hwlocality_openfabrics` 函数：该函数组中唯一的函数，用于获取与设备 `p ibdev` 物理位置接近的 CPU 设置。该函数将返回一个描述设备位置的 CPU 设置。
3. `hwlocality_ibv_get_device_osdev` 函数：该函数用于获取设备 `ibdev` 的操作系统设备对象。如果需要更多的信息，可以调用此函数。
4. `hwlocality_ibv_get_device_osdev_by_name` 函数：该函数与 `hwlocality_ibv_get_device_osdev` 类似，但允许通过设备名称获取设备对象。


```cpp
/** \defgroup hwlocality_openfabrics Interoperability with OpenFabrics
 *
 * This interface offers ways to retrieve topology information about
 * OpenFabrics devices (InfiniBand, Omni-Path, usNIC, etc).
 *
 * @{
 */

/** \brief Get the CPU set of processors that are physically
 * close to device \p ibdev.
 *
 * Store in \p set the CPU-set describing the locality of the OpenFabrics
 * device \p ibdev (InfiniBand, etc).
 *
 * Topology \p topology and device \p ibdev must match the local machine.
 * I/O devices detection is not needed in the topology.
 *
 * The function only returns the locality of the device.
 * If more information about the device is needed, OS objects should
 * be used instead, see hwloc_ibv_get_device_osdev()
 * and hwloc_ibv_get_device_osdev_by_name().
 *
 * This function is currently only implemented in a meaningful way for
 * Linux; other systems will simply get a full cpuset.
 */
```

这段代码定义了一个名为 `hwloc_ibv_get_device_cpuset` 的函数，属于 `hwloc_topology_t` 系列的函数。这个函数的主要作用是获取指定 `hwloc_topology_t` 结构中 `ibdev` 引用的设备的 `local_cpuset`，并将获取到的 `local_cpuset` 存储到给定的 `set` 中。

函数实现中首先检查当前的 `topology` 是否为 Linux 系统，如果是，则使用 `sprintf` 函数从 `/sys/class/infiniband/%s/device/local_cpus` 路径中获取当前设备的 `local_cpuset`，其中 `%s` 是 `ibdev` 结构中的 `device_name` 成员。接着，使用 `hwloc_linux_read_path_as_cpumask` 函数将 `path` 字符串中的所有字符作为 CPU 组，并将结果存储到 `set` 中。最后，如果 `set` 中的任何元素都被设置为 0，则将 `ibdev` 中的 `local_cpuset` 复制到 `set` 中。

总的来说，这个函数的作用是获取一个设备的 `local_cpuset`，并将其存储到给定的 `hwloc_topology_t` 结构中的 `set` 中。


```cpp
static __hwloc_inline int
hwloc_ibv_get_device_cpuset(hwloc_topology_t topology __hwloc_attribute_unused,
			    struct ibv_device *ibdev, hwloc_cpuset_t set)
{
#ifdef HWLOC_LINUX_SYS
  /* If we're on Linux, use the verbs-provided sysfs mechanism to
     get the local cpus */
#define HWLOC_OPENFABRICS_VERBS_SYSFS_PATH_MAX 128
  char path[HWLOC_OPENFABRICS_VERBS_SYSFS_PATH_MAX];

  if (!hwloc_topology_is_thissystem(topology)) {
    errno = EINVAL;
    return -1;
  }

  sprintf(path, "/sys/class/infiniband/%s/device/local_cpus",
	  ibv_get_device_name(ibdev));
  if (hwloc_linux_read_path_as_cpumask(path, set) < 0
      || hwloc_bitmap_iszero(set))
    hwloc_bitmap_copy(set, hwloc_topology_get_complete_cpuset(topology));
```

这段代码是一个 if-else 语句，它会根据操作系统环境和硬件布局自动执行不同的代码。如果没有满足某种情况，它会返回一个 NULL 值。以下是代码的作用：

1. 对于 Linux 系统，它会执行以下操作：

```cpp
hwloc_bitmap_copy(set, hwloc_topology_get_complete_cpuset(topology));
```

这个操作会将 `topology` 对应的 CPU 集复制到一个名为 `set` 的硬件设备集中。

2. 对于其他操作系统（如 Windows、macOS 等），它会执行以下操作：

```cppphp
int i;

for (i = 0; i < hwloc_device_count(); i++) {
   if (hwloc_device_get_name(i, hwloc_device_class_name) == pibname) {
       break;
   }
}

if (i == hwloc_device_count()) {
   return NULL;
}
```

这个操作会遍历所有设备，查找与 `ibname` 相匹配的设备。如果找到了设备，它就会返回一个指向该设备的指针。如果没有找到设备，它将返回一个 NULL 值。

注意：`topology` 参数在代码中没有被明确定义，但根据上下文可以推测它是描述当前硬件设备的拓扑结构。


```cpp
#else
  /* Non-Linux systems simply get a full cpuset */
  hwloc_bitmap_copy(set, hwloc_topology_get_complete_cpuset(topology));
#endif
  return 0;
}

/** \brief Get the hwloc OS device object corresponding to the OpenFabrics
 * device named \p ibname.
 *
 * \return The hwloc OS device object describing the OpenFabrics device
 * (InfiniBand, Omni-Path, usNIC, etc) whose name is \p ibname
 * (mlx5_0, hfi1_0, usnic_0, qib0, etc).
 * \return \c NULL if none could be found.
 *
 * The name \p ibname is usually obtained from ibv_get_device_name().
 *
 * The topology \p topology does not necessarily have to match the current
 * machine. For instance the topology may be an XML import of a remote host.
 * I/O devices detection must be enabled in the topology.
 *
 * \note The corresponding PCI device object can be obtained by looking
 * at the OS device parent object.
 */
```

这段代码定义了一个名为`hwloc_ibv_get_device_osdev_by_name`的函数，它的作用是获取一个硬件设备对象，该对象与OpenFabrics设备（例如InfiniBand设备）的操作系统设备名称相匹配。函数首先定义了一个名为`osdev`的零初始化的对象，然后使用`hwloc_get_next_osdev`函数遍历Topology对象中的所有OS设备对象，检查每个设备对象是否是一个I/O设备，如果是且设备名称与`ibname`参数相匹配，则返回该设备对象。如果在该Topology中找不到匹配的设备对象，则返回`NULL`。函数使用了Infiniband I/O检测设置，该设置允许在Topology对象中检测到I/O设备。如果检测设置不正确，则可以使用`hwloc_ibv_get_device_cpuset`函数查找与设备相关的CPU集。函数还指出，可以通过获取OS设备的父对象来获取相应的PCI设备对象。


```cpp
static __hwloc_inline hwloc_obj_t
hwloc_ibv_get_device_osdev_by_name(hwloc_topology_t topology,
				   const char *ibname)
{
	hwloc_obj_t osdev = NULL;
	while ((osdev = hwloc_get_next_osdev(topology, osdev)) != NULL) {
		if (HWLOC_OBJ_OSDEV_OPENFABRICS == osdev->attr->osdev.type
		    && osdev->name && !strcmp(ibname, osdev->name))
			return osdev;
	}
	return NULL;
}

/** \brief Get the hwloc OS device object corresponding to the OpenFabrics
 * device \p ibdev.
 *
 * \return The hwloc OS device object describing the OpenFabrics
 * device \p ibdev (InfiniBand, etc).
 * \return \c NULL if none could be found.
 *
 * Topology \p topology and device \p ibdev must match the local machine.
 * I/O devices detection must be enabled in the topology.
 * If not, the locality of the object may still be found using
 * hwloc_ibv_get_device_cpuset().
 *
 * \note The corresponding PCI device object can be obtained by looking
 * at the OS device parent object.
 */
```

这段代码定义了一个名为 `hwloc_ibv_get_device_osdev` 的函数，属于 `hwloc_ibv_get_device_osdev.h` 文件。函数接受两个参数：`topology` 和 `ibdev`，分别表示要查询的硬件层和 `ibdev` 结构体。函数返回一个指向 `hwloc_obj_t` 类型对象的指针，该对象存储了 `ibdev` 结构体中 `device_osdev` 成员的实现。

函数首先检查所查询的硬件层是否属于当前系统。如果是，函数将返回 `NULL`，否则会输出一个错误并返回 `EINVAL`。

函数的实现主要取决于 `hwloc_topology_is_thissystem()` 函数，它用于检查当前硬件层是否属于当前系统。如果当前硬件层不属于当前系统，函数将无法正常工作，可能会抛出错误并返回 `EINVAL`。


```cpp
static __hwloc_inline hwloc_obj_t
hwloc_ibv_get_device_osdev(hwloc_topology_t topology,
			   struct ibv_device *ibdev)
{
	if (!hwloc_topology_is_thissystem(topology)) {
		errno = EINVAL;
		return NULL;
	}
	return hwloc_ibv_get_device_osdev_by_name(topology, ibv_get_device_name(ibdev));
}

/** @} */


#ifdef __cplusplus
} /* extern "C" */
```

这是一个C语言代码片段，其中包含两个头文件出口。这个代码片段的作用是定义了两个头文件，在编译时会检查这两个头文件是否已经被定义，如果已经被定义，则编译器会报错，如果没有定义，则可以继续编译。

具体来说，这个代码片段定义了两个名为`hwloc_opens纤维`和`hwloc_opens刚性纤维`的头文件。这两个头文件可能会被用于定义或者包含`hwloc_open`函数，用于在`hwloc`库中打开或者关闭硬件资源。


```cpp
#endif


#endif /* HWLOC_OPENFABRICS_VERBS_H */

```

# `src/3rdparty/hwloc/include/hwloc/plugins.h`

这段代码定义了一个名为`hwloc_plugins_h`的文件，它是一个C++类，继承自`hwloc_backend`结构体。

这个文件的作用是定义一个公共接口，用于构建硬件抽象层(HWL)插件。`hwloc_backend`是一个指向底层硬件驱动程序的指针，而`hwloc_plugins_h`类则是在这些底层硬件驱动程序和用户程序之间进行抽象的接口。

这个文件中定义了一个名为`struct hwloc_backend`的类型，它是一个包含两个整数的结构体，第一个数为`int`，表示自定义的硬件设备ID，第二个数为`int`，表示硬件设备操作的版本号。

此外，这个文件中还定义了一个名为`hwloc_plugins_init()`的函数，它接受一个`hwloc_backend`指针和一个`hwloc_channel_t`类型的参数。这个函数用于初始化`hwloc_plugins_h`类的实例，创建一个`hwloc_channel_t`类型的数据结构，并设置其硬驱ID和设备操作版本号为1。

最后，这个文件中还定义了一个`hwloc_plugins_append()`函数，它接受一个`hwloc_plugins_t`类型的参数，这个函数将新的插件添加到`hwloc_channel_t`类型的数据结构中，并返回它的指针。


```cpp
/*
 * Copyright © 2013-2022 Inria.  All rights reserved.
 * Copyright © 2016 Cisco Systems, Inc.  All rights reserved.
 * See COPYING in top-level directory.
 */

#ifndef HWLOC_PLUGINS_H
#define HWLOC_PLUGINS_H

/** \file
 * \brief Public interface for building hwloc plugins.
 */

struct hwloc_backend;

```

这段代码是一个C语言的代码，定义了一个名为`hwlocality_disc_components`的组，其中包括一些头文件和函数。这个组定义了一些组件，用于在HWLOC寻找硬件组件。

首先，代码中定义了一个`#include "hwloc.h"`，这意味着在HWLOC库中定义了`hwloc.h`的头文件。

然后，代码中定义了一个`#ifdef HWLOC_INSIDE_PLUGIN`，这意味着在插件加载时检查HWLOC是否在内部加载。如果`HWLOC_INSIDE_PLUGIN`为真，那么下面的`#ifdef HWLOC_HAVE_LTDL`将不会被编译，因为LTDL库中已经定义了这些头文件。否则，`#include <ltdl.h>`将在编译时包含LTDL库。

接下来，定义了一系列结构体和函数，这些结构体和函数将在发现组件时使用。

最后，代码中定义了一个名为`__hwloc_disc_component_bootstrap`的函数，用于初始化HWLOC库中的组件。


```cpp
#include "hwloc.h"

#ifdef HWLOC_INSIDE_PLUGIN
/* needed for hwloc_plugin_check_namespace() */
#ifdef HWLOC_HAVE_LTDL
#include <ltdl.h>
#else
#include <dlfcn.h>
#endif
#endif



/** \defgroup hwlocality_disc_components Components and Plugins: Discovery components
 *
 * \note These structures and functions may change when ::HWLOC_COMPONENT_ABI is modified.
 *
 * @{
 */

```

This is a description of the `hwloc_disc_component` struct, which represents a component in the HWLOC Disc topology system.

This component has an enabled state, which is determined by the `enabled_by_default` field. If this field is not set, the component will be disabled unless it is explicitly requested.

The `priority` field indicates the priority of the component. The priority is used to determine which components are displayed in the topology when multiple components with the same name are present.

The `instantiate` function is used to create a backend from the component. It takes as parameters the topology, the component, a set of excluded phases, and data1, data2, and data3 (which is optional).

The `next` field is used internally to list the components in the priority queue.


```cpp
/** \brief Discovery component structure
 *
 * This is the major kind of components, taking care of the discovery.
 * They are registered by generic components, either statically-built or as plugins.
 */
struct hwloc_disc_component {
  /** \brief Name.
   * If this component is built as a plugin, this name does not have to match the plugin filename.
   */
  const char *name;

  /** \brief Discovery phases performed by this component.
   * OR'ed set of ::hwloc_disc_phase_t
   */
  unsigned phases;

  /** \brief Component phases to exclude, as an OR'ed set of ::hwloc_disc_phase_t.
   *
   * For a GLOBAL component, this usually includes all other phases (\c ~UL).
   *
   * Other components only exclude types that may bring conflicting
   * topology information. MISC components should likely not be excluded
   * since they usually bring non-primary additional information.
   */
  unsigned excluded_phases;

  /** \brief Instantiate callback to create a backend from the component.
   * Parameters data1, data2, data3 are NULL except for components
   * that have special enabling routines such as hwloc_topology_set_xml(). */
  struct hwloc_backend * (*instantiate)(struct hwloc_topology *topology, struct hwloc_disc_component *component, unsigned excluded_phases, const void *data1, const void *data2, const void *data3);

  /** \brief Component priority.
   * Used to sort topology->components, higher priority first.
   * Also used to decide between two components with the same name.
   *
   * Usual values are
   * 50 for native OS (or platform) components,
   * 45 for x86,
   * 40 for no-OS fallback,
   * 30 for global components (xml, synthetic),
   * 20 for pci,
   * 10 for other misc components (opencl etc.).
   */
  unsigned priority;

  /** \brief Enabled by default.
   * If unset, if will be disabled unless explicitly requested.
   */
  unsigned enabled_by_default;

  /** \private Used internally to list components by priority on topology->components
   * (the component structure is usually read-only,
   *  the core copies it before using this field for queueing)
   */
  struct hwloc_disc_component * next;
};

```

`hwloc_disc_phase_e` is an XML-defined XMLPeripheralTopologyDiscovery macro that discovers various types of hardware components, such as CPUs, memory, I/O devices, and more.

It has several phases that it can enter, including:

- `HWLOC_DISC_PHASE_CPU`: This phase discovers the CPU and related devices.
- `HWLOC_DISC_PHASE_MEMORY`: This phase discovers the memory and related devices.
- `HWLOC_DISC_PHASE_PCI`: This phase discovers the PCI devices and related devices.
- `HWLOC_DISC_PHASE_IO`: This phase discovers the I/O devices and related devices.
- `HWLOC_DISC_PHASE_ANNOTATE`: This phase annotates any previously discovered objects with additional information.
- `HWLOC_DISC_PHASE_TWEAK`: This phase tweaks the topology to make it ready for use.

These phases can be entered into orderly or random order, and can also be used to modify the discovered objects.



```cpp
/** @} */




/** \defgroup hwlocality_disc_backends Components and Plugins: Discovery backends
 *
 * \note These structures and functions may change when ::HWLOC_COMPONENT_ABI is modified.
 *
 * @{
 */

/** \brief Discovery phase */
typedef enum hwloc_disc_phase_e {
  /** \brief xml or synthetic, platform-specific components such as bgq.
   * Discovers everything including CPU, memory, I/O and everything else.
   * A component with a Global phase usually excludes all other phases.
   * \hideinitializer */
  HWLOC_DISC_PHASE_GLOBAL = (1U<<0),

  /** \brief CPU discovery.
   * \hideinitializer */
  HWLOC_DISC_PHASE_CPU = (1U<<1),

  /** \brief Attach memory to existing CPU objects.
   * \hideinitializer */
  HWLOC_DISC_PHASE_MEMORY = (1U<<2),

  /** \brief Attach PCI devices and bridges to existing CPU objects.
   * \hideinitializer */
  HWLOC_DISC_PHASE_PCI = (1U<<3),

  /** \brief I/O discovery that requires PCI devices (OS devices such as OpenCL, CUDA, etc.).
   * \hideinitializer */
  HWLOC_DISC_PHASE_IO = (1U<<4),

  /** \brief Misc objects that gets added below anything else.
   * \hideinitializer */
  HWLOC_DISC_PHASE_MISC = (1U<<5),

  /** \brief Annotating existing objects, adding distances, etc.
   * \hideinitializer */
  HWLOC_DISC_PHASE_ANNOTATE = (1U<<6),

  /** \brief Final tweaks to a ready-to-use topology.
   * This phase runs once the topology is loaded, before it is returned to the topology.
   * Hence it may only use the main hwloc API for modifying the topology,
   * for instance by restricting it, adding info attributes, etc.
   * \hideinitializer */
  HWLOC_DISC_PHASE_TWEAK = (1U<<7)
} hwloc_disc_phase_t;

```

这段代码定义了一个名为 `hwloc_disc_status` 的枚举类型，用于表示在硬件定位过程中所处的状态。枚举类型中包含三个成员变量：`phase`、`excluded_phases` 和 `flags`。

`phase` 变量表示当前的定位阶段，它必须与组件 phase 字段中的一个匹配。

`excluded_phases` 变量表示动态地被排除的阶段数量。如果一个组件在发现过程中决定某些阶段不再需要，则可以动态地将其从 `excluded_phases` 中添加或从其中删除。

`flags` 是一个位掩，用于组合 `excluded_phases` 和 `GOT_ALLOWED_RESOURCES` 标志。`excluded_phases` 和 `GOT_ALLOWED_RESOURCES` 标志都设置了时，`flags` 的二进制表示将设置为 1。

这个结构体可以用来在程序中记录和设置硬件定位阶段的 state，以及动态地将其排除在外。


```cpp
/** \brief Discovery status flags */
enum hwloc_disc_status_flag_e {
  /** \brief The sets of allowed resources were already retrieved \hideinitializer */
  HWLOC_DISC_STATUS_FLAG_GOT_ALLOWED_RESOURCES = (1UL<<1)
};

/** \brief Discovery status structure
 *
 * Used by the core and backends to inform about what has been/is being done
 * during the discovery process.
 */
struct hwloc_disc_status {
  /** \brief The current discovery phase that is performed.
   * Must match one of the phases in the component phases field.
   */
  hwloc_disc_phase_t phase;

  /** \brief Dynamically excluded phases.
   * If a component decides during discovery that some phases are no longer needed.
   */
  unsigned excluded_phases;

  /** \brief OR'ed set of hwloc_disc_status_flag_e */
  unsigned long flags;
};

```

这段代码定义了一个名为" Discovery backend structure"的函数，用于描述发现组件实例的逻辑。当组件启用时，它的实例化器会创建一个后端。在这里，hwloc_backend_alloc()函数用于设置所有组件可变字段为默认值，以便在启用后端时进行初始化。

该函数的主要作用是为组件创建一个后端实例，并为组件的实例化器初始化所有字段为默认值。在大多数情况下，后端还假设topology.is_thissystem标志设置，因为它们与底层操作系统通信。但是，在某些情况下，后端可能不需要topology.is_thissystem标志，例如通过使用excluded by xml或synthetic backends，或者通过使用环境变量在改变Linux fsroot或x86 cpuid路径时。

函数中还包含一个名为" most backends"的指针，它指出大多数后端会做什么。大多数后端会假设topology.is_thissystem标志设置，因为它们与底层操作系统通信。但是，在某些情况下，后端可能不需要topology.is_thissystem标志，例如通过使用excluded by xml或synthetic backends，或者通过使用环境变量在改变Linux fsroot或x86 cpuid路径时。


```cpp
/** \brief Discovery backend structure
 *
 * A backend is the instantiation of a discovery component.
 * When a component gets enabled for a topology,
 * its instantiate() callback creates a backend.
 *
 * hwloc_backend_alloc() initializes all fields to default values
 * that the component may change (except "component" and "next")
 * before enabling the backend with hwloc_backend_enable().
 *
 * Most backends assume that the topology is_thissystem flag is
 * set because they talk to the underlying operating system.
 * However they may still be used in topologies without the
 * is_thissystem flag for debugging reasons.
 * In practice, they are usually auto-disabled in such cases
 * (excluded by xml or synthetic backends, or by environment
 *  variables when changing the Linux fsroot or the x86 cpuid path).
 */
```

This is a C language definition of an `hwloc_backend` struct that represents the hardware location of a Linux backend. The `hwloc_backend` struct has several fields, including `envvar_forced`, `next`, `phases`, `flags`, `private_data`, `disable`, and `discover`. The `envvar_forced` field indicates whether the backend was forced through an environment variable or not. The `next` field is a pointer to the next backend in the topology. The `phases` field is an array of `hwloc_disc_phase_t` structures representing the discovered phases of the backend. The `flags` field is an unsigned long that can be used to store additional information about the backend. The `private_data` field is a pointer to the private data of the backend, such as a device tree node. The `disable` function is a callback for freeing the private data. The `discover` function is a callback for discovering new backends. The `get_pci_busid_cpuset` function is a callback for retrieving the locality of a PCI object.



```cpp
struct hwloc_backend {
  /** \private Reserved for the core, set by hwloc_backend_alloc() */
  struct hwloc_disc_component * component;
  /** \private Reserved for the core, set by hwloc_backend_enable() */
  struct hwloc_topology * topology;
  /** \private Reserved for the core. Set to 1 if forced through envvar, 0 otherwise. */
  int envvar_forced;
  /** \private Reserved for the core. Used internally to list backends topology->backends. */
  struct hwloc_backend * next;

  /** \brief Discovery phases performed by this component, possibly without some of them if excluded by other components.
   * OR'ed set of ::hwloc_disc_phase_t
   */
  unsigned phases;

  /** \brief Backend flags, currently always 0. */
  unsigned long flags;

  /** \brief Backend-specific 'is_thissystem' property.
   * Set to 0 if the backend disables the thissystem flag for this topology
   * (e.g. loading from xml or synthetic string,
   *  or using a different fsroot on Linux, or a x86 CPUID dump).
   * Set to -1 if the backend doesn't care (default).
   */
  int is_thissystem;

  /** \brief Backend private data, or NULL if none. */
  void * private_data;
  /** \brief Callback for freeing the private_data.
   * May be NULL.
   */
  void (*disable)(struct hwloc_backend *backend);

  /** \brief Main discovery callback.
   * returns -1 on error, either because it couldn't add its objects ot the existing topology,
   * or because of an actual discovery/gathering failure.
   * May be NULL.
   */
  int (*discover)(struct hwloc_backend *backend, struct hwloc_disc_status *status);

  /** \brief Callback to retrieve the locality of a PCI object.
   * Called by the PCI core when attaching PCI hierarchy to CPU objects.
   * May be NULL.
   */
  int (*get_pci_busid_cpuset)(struct hwloc_backend *backend, struct hwloc_pcidev_attr_s *busid, hwloc_bitmap_t cpuset);
};

```

这段代码定义了两个函数：`hwloc_backend_alloc` 和 `hwloc_backend_enable`。它们的作用如下：

`hwloc_backend_alloc`函数用于分配一个后端结构，设置良好的默认值，初始化后端组件和拓扑结构等。它的实现在调用者修改需要分配的后端组件后，调用`hwloc_backend_enable`函数。

`hwloc_backend_enable`函数用于启用 previously分配和设置的后端组件。它接收一个后端组件结构作为参数，并返回一个表示成功或失败的结果。如果成功启用后端组件，该函数将被返回，否则它将返回一个错误代码。调用者可以在启用后端组件之后修改 `hwloc_topology` 和 `hwloc_disc_component` 结构。


```cpp
/** \brief Allocate a backend structure, set good default values, initialize backend->component and topology, etc.
 * The caller will then modify whatever needed, and call hwloc_backend_enable().
 */
HWLOC_DECLSPEC struct hwloc_backend * hwloc_backend_alloc(struct hwloc_topology *topology, struct hwloc_disc_component *component);

/** \brief Enable a previously allocated and setup backend. */
HWLOC_DECLSPEC int hwloc_backend_enable(struct hwloc_backend *backend);

/** @} */




/** \defgroup hwlocality_generic_components Components and Plugins: Generic components
 *
 * \note These structures and functions may change when ::HWLOC_COMPONENT_ABI is modified.
 *
 * @{
 */

```

这是一颗用于定义通用组件结构的C++类，名为`hwloc_component`。该类结构可以用于静态组件定义的`static-components.h`文件或动态加载作为一个插件。

该类包含以下成员：
- `abi`：组件ABI版本。
- `init`：组件A您就可以初始化的回调。当组件作为一个插件加载时，这个回调会被调用。如果组件使用`ltdl`加载自己的插件，那么这个回调在`init`和`finalize`过程中被调用，以避免与`hwloc`的`ltdl`使用冲突。
- `finalize`：组件作为一个插件卸载时的回调。当组件作为一个插件卸载时，这个回调会被调用。如果组件使用`ltdl`加载自己的插件，那么这个回调在`init`和`finalize`过程中被调用，以避免与`hwloc`的`ltdl`使用冲突。
- `type`：组件类型。
- `flags`：组件的标志，目前未知。
- `data`：组件数据，指向一个`hwloc_disc_component`或`hwloc_xml_component`的结构体。

该类的实例化可以通过以下方式：
```cppphp
hwloc_component component;
component.init(0);
```
这个实例化会初始化组件，并且在组件注册后调用`finalize`。


```cpp
/** \brief Generic component type */
typedef enum hwloc_component_type_e {
  /** \brief The data field must point to a struct hwloc_disc_component. */
  HWLOC_COMPONENT_TYPE_DISC,

  /** \brief The data field must point to a struct hwloc_xml_component. */
  HWLOC_COMPONENT_TYPE_XML
} hwloc_component_type_t;

/** \brief Generic component structure
 *
 * Generic components structure, either statically listed by configure in static-components.h
 * or dynamically loaded as a plugin.
 */
struct hwloc_component {
  /** \brief Component ABI version, set to ::HWLOC_COMPONENT_ABI */
  unsigned abi;

  /** \brief Process-wide component initialization callback.
   *
   * This optional callback is called when the component is registered
   * to the hwloc core (after loading the plugin).
   *
   * When the component is built as a plugin, this callback
   * should call hwloc_check_plugin_namespace()
   * and return an negative error code on error.
   *
   * \p flags is always 0 for now.
   *
   * \return 0 on success, or a negative code on error.
   *
   * \note If the component uses ltdl for loading its own plugins,
   * it should load/unload them only in init() and finalize(),
   * to avoid race conditions with hwloc's use of ltdl.
   */
  int (*init)(unsigned long flags);

  /** \brief Process-wide component termination callback.
   *
   * This optional callback is called after unregistering the component
   * from the hwloc core (before unloading the plugin).
   *
   * \p flags is always 0 for now.
   *
   * \note If the component uses ltdl for loading its own plugins,
   * it should load/unload them only in init() and finalize(),
   * to avoid race conditions with hwloc's use of ltdl.
   */
  void (*finalize)(unsigned long flags);

  /** \brief Component type */
  hwloc_component_type_t type;

  /** \brief Component flags, unused for now */
  unsigned long flags;

  /** \brief Component data, pointing to a struct hwloc_disc_component or struct hwloc_xml_component. */
  void * data;
};

```

这段代码定义了一个名为`hwlocality_components_core_funcs`的组，其中包含了一些用于组件和插件的核心函数。

具体来说，这个组包含以下函数：

- `hwlocality_show_critical_error()`：检查给定的错误消息是否隐藏。如果这个函数返回小于2，那么调用者应该打印关键的错误消息，例如不正确的hw topo信息或失败的CUDA初始化。如果这个函数返回2，那么调用者应该打印非关键的错误消息，例如未能初始化CUDA。

- `hwlocality_show_all_errors()`：检查给定的错误消息是否隐藏。如果这个函数返回0，那么调用者应该打印所有的错误消息，包括关键的和不关键的。如果这个函数返回1，那么调用者应该打印关键的错误消息，而忽略所有的其他错误消息。如果这个函数在环境中设置了`HWLOC_HIDE_ERRORS`，那么这个函数将按照设置值打印错误消息，而不是隐藏它们。

- `hwlocality_show_critical_only()`：检查给定的错误消息是否隐藏，并且仅显示关键的错误消息，而不是所有或部分的错误消息。

- `hwlocality_show_all_critical_only()`：与`hwlocality_show_critical_only()`相反，这个函数显示所有的错误消息，包括关键的和不关键的。


```cpp
/** @} */




/** \defgroup hwlocality_components_core_funcs Components and Plugins: Core functions to be used by components
 *
 * \note These structures and functions may change when ::HWLOC_COMPONENT_ABI is modified.
 *
 * @{
 */

/** \brief Check whether error messages are hidden.
 *
 * Callers should print critical error messages
 * (e.g. invalid hw topo info, invalid config)
 * only if this function returns strictly less than 2.
 *
 * Callers should print non-critical error messages
 * (e.g. failure to initialize CUDA)
 * if this function returns 0.
 *
 * This function return 1 by default (show critical only),
 * 0 in lstopo (show all),
 * or anything set in HWLOC_HIDE_ERRORS in the environment.
 *
 * Use macros HWLOC_SHOW_CRITICAL_ERRORS() and HWLOC_SHOW_ALL_ERRORS()
 * for clarity.
 */
```

这段代码定义了一个名为hwloc_hide_errors的函数，其作用是返回一个整数，表示在给定的位置上是否可以隐藏错误输出。通过定义两个宏，HWLOC_SHOW_CRITICAL_ERRORS和HWLOC_SHOW_ALL_ERRORS，可以更容易地理解这段代码的作用。

HWLOC_SHOW_CRITICAL_ERRORS定义了一个函数，返回值为hwloc_hide_errors，表示如果该函数的返回值小于2，则表示可以隐藏 critical 错误。HWLOC_SHOW_ALL_ERRORS定义了另一个函数，返回值也为hwloc_hide_errors，表示如果该函数的返回值等于0，则表示可以隐藏所有错误。

这两段代码定义了一个名为hwloc_hide_errors的函数，用于在给定的位置上是否可以隐藏错误输出。通过定义两个宏，可以更容易地理解该函数的作用。


```cpp
HWLOC_DECLSPEC int hwloc_hide_errors(void);

#define HWLOC_SHOW_CRITICAL_ERRORS() (hwloc_hide_errors() < 2)
#define HWLOC_SHOW_ALL_ERRORS() (hwloc_hide_errors() == 0)

/** \brief Add an object to the topology.
 *
 * Insert new object \p obj in the topology starting under existing object \p root
 * (if \c NULL, the topology root object is used).
 *
 * It is sorted along the tree of other objects according to the inclusion of
 * cpusets, to eventually be added as a child of the smallest object including
 * this object.
 *
 * If the cpuset is empty, the type of the object (and maybe some attributes)
 * must be enough to find where to insert the object. This is especially true
 * for NUMA nodes with memory and no CPUs.
 *
 * The given object should not have children.
 *
 * This shall only be called before levels are built.
 *
 * The caller should check whether the object type is filtered-out before calling this function.
 *
 * The topology cpuset/nodesets will be enlarged to include the object sets.
 *
 * \p reason is a unique string identifying where and why this insertion call was performed
 * (it will be displayed in case of internal insertion error).
 *
 * Returns the object on success.
 * Returns NULL and frees obj on error.
 * Returns another object and frees obj if it was merged with an identical pre-existing object.
 */
```

这段代码定义了一个名为 `hwloc__insert_object_by_cpuset` 的函数，属于 `hwloc_topology` 结构体的派生类 `hwloc_obj_t`。

这个函数的主要作用是插入选项在指定的 `topology` 中，对象类型可以是 `hwloc_obj_t`，`hwloc_device_t`，`hwloc_user_data_t`， `hwloc_resources_t`， `hwloc_proc_t`， `hwloc_subtree_t`， `hwloc_register_t`， `hwloc_device_t` 或者 `hwloc_system_device_t`。

函数的第一个参数是一个指向 `topology` 的 `hwloc_topology` 类型指针，第二个参数是一个指向 `root` 的 `hwloc_obj_t` 类型指针，第三个参数是要插入的对象 `obj` 的类型指针，第四个参数是插入的原因，传递给下标函数使用。

通过调用 `hwloc__insert_object_by_cpuset`，可以将指定的对象插入到指定的 topology 中的某个位置，可以用于在 xml 文件中插入选项，也可以用于在系统设备中插入对象。当用于 "normal" 类型的子节点时，由于 cpuset 被完全忽略，因此应当将对象插入到正确的位置以确保 children 按正确的顺序插入，并且 children 的cpuset 不要相互干预。


```cpp
HWLOC_DECLSPEC hwloc_obj_t
hwloc__insert_object_by_cpuset(struct hwloc_topology *topology, hwloc_obj_t root,
                               hwloc_obj_t obj, const char *reason);

/** \brief Insert an object somewhere in the topology.
 *
 * It is added as the last child of the given parent.
 * The cpuset is completely ignored, so strange objects such as I/O devices should
 * preferably be inserted with this.
 *
 * When used for "normal" children with cpusets (when importing from XML
 * when duplicating a topology), the caller should make sure that:
 * - children are inserted in order,
 * - children cpusets do not intersect.
 *
 * The given object may have normal, I/O or Misc children, as long as they are in order as well.
 * These children must have valid parent and next_sibling pointers.
 *
 * The caller should check whether the object type is filtered-out before calling this function.
 */
```

这两行代码定义了一个名为 `hwloc_insert_object_by_parent` 的函数，属于 `hwloc_topology` 类的成员函数。函数接受一个 `hwloc_topology` 类型的指针 `topology`，一个 `hwloc_obj_t` 类型的指针 `parent`，和一个 `hwloc_obj_t` 类型的参数 `obj`。

这两行代码的主要作用是插入一个新对象到指定的 topology 中的某个 parent 对象中。如果指定的 parent 对象中包含新对象，函数将尝试在 topology 中查找与该 parent 对象相同的根节点，如果找到了，函数将忽略插入的子节点，否则将插入新对象。

这两行代码还定义了一个名为 `hwloc_alloc_setup_object` 的函数，属于 `hwloc_topology_device` 类的成员函数。函数接受一个 `hwloc_topology_device` 类型的指针 `topology`，一个 `hwloc_obj_type_t` 类型的参数 `type`，和一个 `unsigned long long` 类型的参数 `os_index`。

这两行代码的主要作用是分配并设置指定类型的对象，如果指定的对象类型未知或者与 topology 中的其他对象不匹配，函数将使用默认的 HWLOC_UNKNOWN_INDEX。当函数完成时，将调用插入新对象时定义的 `hwloc_insert_object_by_parent` 函数。


```cpp
HWLOC_DECLSPEC void hwloc_insert_object_by_parent(struct hwloc_topology *topology, hwloc_obj_t parent, hwloc_obj_t obj);

/** \brief Allocate and initialize an object of the given type and physical index.
 *
 * If \p os_index is unknown or irrelevant, use \c HWLOC_UNKNOWN_INDEX.
 */
HWLOC_DECLSPEC hwloc_obj_t hwloc_alloc_setup_object(hwloc_topology_t topology, hwloc_obj_type_t type, unsigned os_index);

/** \brief Setup object cpusets/nodesets by OR'ing its children.
 *
 * Used when adding an object late in the topology.
 * Will update the new object by OR'ing all its new children sets.
 *
 * Used when PCI backend adds a hostbridge parent, when distances
 * add a new Group, etc.
 */
```

这两段代码定义了两个函数，名为`hwloc_obj_add_children_sets`和`hwloc_topology_reconnect`。

`hwloc_obj_add_children_sets`函数接收一个`hwloc_obj_t`类型的参数，并返回一个整数。它表示在topology中添加孩子的设置。可以理解为目的函数在拓扑结构中增加或删除物体。

`hwloc_topology_reconnect`函数也接收一个`hwloc_topology_t`类型的参数和一个`unsigned long`类型的参数`flags`，但不使用它。它表示在topology中重新连接物体和级别。可以理解为目的函数在拓扑结构中重新连接物体和级别。

这两段代码定义了两个函数，用于在topology中管理物体和级别。第一个函数用于添加物体，第二个函数用于在topology中重新连接物体和级别。


```cpp
HWLOC_DECLSPEC int hwloc_obj_add_children_sets(hwloc_obj_t obj);

/** \brief Request a reconnection of children and levels in the topology.
 *
 * May be used by backends during discovery if they need arrays or lists
 * of object within levels or children to be fully connected.
 *
 * \p flags is currently unused, must 0.
 */
HWLOC_DECLSPEC int hwloc_topology_reconnect(hwloc_topology_t topology, unsigned long flags __hwloc_attribute_unused);

/** \brief Make sure that plugins can lookup core symbols.
 *
 * This is a sanity check to avoid lazy-lookup failures when libhwloc
 * is loaded within a plugin, and later tries to load its own plugins.
 * This may fail (and abort the program) if libhwloc symbols are in a
 * private namespace.
 *
 * \return 0 on success.
 * \return -1 if the plugin cannot be successfully loaded. The caller
 * plugin init() callback should return a negative error code as well.
 *
 * Plugins should call this function in their init() callback to avoid
 * later crashes if lazy symbol resolution is used by the upper layer that
 * loaded hwloc (e.g. OpenCL implementations using dlopen with RTLD_LAZY).
 *
 * \note The build system must define HWLOC_INSIDE_PLUGIN if and only if
 * building the caller as a plugin.
 *
 * \note This function should remain inline so plugins can call it even
 * when they cannot find libhwloc symbols.
 */
```

这段代码是一个C语言函数，名为`hwloc_plugin_check_namespace`，属于`hwloc_plugin_export`家族。它的作用是检查给定的插件名称和符号是否与定义在`hwloc_plugin_export`中的同名字符相等，如果相等则返回0，否则返回非0。

具体实现包括以下几步：

1. 定义了一个名为`hwloc_plugin_check_namespace`的函数，它接收两个参数：`pluginname`和`symbol`，这两个参数分别传递给函数。

2. 在函数内部，首先定义了一个名为`sym`的变量，用于存储当前要检查的符号名称。

3. 然后判断给定的插件名称和符号是否与定义在`hwloc_plugin_export`中的同名字符相等。这里使用了`hwloc_inside_plugin`这个预处理指令，它的作用是在编译时判断插件是否处于内部源代码中。如果没有这个指令，就需要手动判断。

4. 如果插件是内部源代码，则执行以下操作：

  a. 查找给定的符号名称在`hwloc_plugin_export`中的定义。

  b. 如果定义成功，则将`sym`指向符号名称，`handle`指向符号链囤。

  c. 否则，创建一个新的符号链囤，并将`handle`指向它，`sym`指向符号名称。

  d. 最后，返回`0`表示成功检查，返回`1`表示失败检查。

5. 如果插件不是内部源代码，则按照上述类似的方式执行检查，但使用`dt`（`dll`和`tlb`）函数而不是`dl`。


```cpp
static __hwloc_inline int
hwloc_plugin_check_namespace(const char *pluginname __hwloc_attribute_unused, const char *symbol __hwloc_attribute_unused)
{
#ifdef HWLOC_INSIDE_PLUGIN
  void *sym;
#ifdef HWLOC_HAVE_LTDL
  lt_dlhandle handle = lt_dlopen(NULL);
#else
  void *handle = dlopen(NULL, RTLD_NOW|RTLD_LOCAL);
#endif
  if (!handle)
    /* cannot check, assume things will work */
    return 0;
#ifdef HWLOC_HAVE_LTDL
  sym = lt_dlsym(handle, symbol);
  lt_dlclose(handle);
```

这段代码是一个C语言中的一个函数，其功能是检查当前插件是否在运行时定义了`__attribute__((no_arn))`注解，并且在插件运行时无法通过`dlsym()`函数找到定义的符号`symbol`，如果是，则输出一条错误信息，并返回-1。

具体来说，代码首先通过`dlsym()`函数获取当前插件的符号`symbol`，然后使用`dlclose()`函数关闭打开的通道。接着，代码检查是否可以通过`dlsym()`函数找到符号`symbol`，如果是，则执行以下操作：

1. 如果当前插件已经定义了`__attribute__((no_arn))`注解，并且当前插件运行时可以看到该注解，则使用`fprintf()`函数输出一条错误信息，其中`%s`表示当前插件的名称，`%s'`表示当前插件定义的符号名称，`disabling`表示当前插件正在尝试禁用符号。注意，`%s`和`%s'`是占位符，实际输出时需要根据当前插件的名称和符号进行替换。

2. 如果当前插件运行时无法通过`dlsym()`函数找到符号`symbol`，则执行以下操作：

  a. 如果当前插件已经定义了`__attribute__((no_arn))`注解，并且当前插件运行时可以看到该注解，则使用`static`关键字定义一个名为`verboseenv_checked`的静态变量，其值为0。

  b. 如果当前插件运行时无法通过`dlsym()`函数找到符号`symbol`，则使用`const char *`和`atoi()`函数获取当前环境中的`HWLOC_PLUGINS_VERBOSE`变量，并输出一个错误信息。其中，`%s`表示当前插件的名称，`%s'`表示当前插件定义的符号名称，`disabling`表示当前插件正在尝试禁用符号。注意，`%s`和`%s'`是占位符，实际输出时需要根据当前插件的名称和符号进行替换。


```cpp
#else
  sym = dlsym(handle, symbol);
  dlclose(handle);
#endif
  if (!sym) {
    static int verboseenv_checked = 0;
    static int verboseenv_value = 0;
    if (!verboseenv_checked) {
      const char *verboseenv = getenv("HWLOC_PLUGINS_VERBOSE");
      verboseenv_value = verboseenv ? atoi(verboseenv) : 0;
      verboseenv_checked = 1;
    }
    if (verboseenv_value)
      fprintf(stderr, "Plugin `%s' disabling itself because it cannot find the `%s' core symbol.\n",
	      pluginname, symbol);
    return -1;
  }
```

这段代码是一个C语言中定义的函数，属于hwlocality_components_filtering库。该库的作用是过滤出给定的组件或插件，使得在hwlocality_混编（hwlocality_混编）中，组件或插件可以被正确地配置和激活。

具体来说，该函数在`hwlocality_inside_plugins`结构体中检查定义的组件或插件，如果定义成功则返回0，否则返回一个非0值，以便在混编时可以报告错误。同时，该函数也可以作为库函数被调用以实现更高级别的组件或插件过滤。


```cpp
#endif /* HWLOC_INSIDE_PLUGIN */
  return 0;
}

/** @} */




/** \defgroup hwlocality_components_filtering Components and Plugins: Filtering objects
 *
 * \note These structures and functions may change when ::HWLOC_COMPONENT_ABI is modified.
 *
 * @{
 */

```

该代码是一个名为 `hwloc_filter_check_pcidev_subtype_important` 的函数，它用于检查给定的 PCI 设备类 ID 是否重要。函数的返回值是一个整数，1 如果设备类 ID 重要，0 否则。

函数的作用是判断给定的 PCI 设备类 ID 是否属于重要的类别，然后返回相应的值。重要的类别包括：

- PCI_BASE_CLASS_DISPLAY：显示类设备，如显示卡
- PCI_BASE_CLASS_NETWORK：网络类设备，如以太网适配器
- PCI_BASE_CLASS_STORAGE：存储类设备，如硬盘驱动器
- PCI_BASE_CLASS_PROCESSOR：处理器类设备，如 CPU
- PCI_CLASS_SERIAL_FIBER：串行类设备，如串口控制器
- PCI_CLASS_SERIAL_INFINIBAND：串行类设备，如 SATA 控制器
- PCI_CLASS_MEMORY_CXL：内存类设备，如芯片组
- PCI_BASE_CLASS_BRIDGE：桥接类设备，如网络桥接器
- PCI_CLASS_PROCESSOR：处理器类设备，如 GPU

具体的，函数首先将设备类 ID 按位异或，然后与重要的类别进行比较，如果设备类 ID 属于重要的类别，函数返回 1，否则返回 0。


```cpp
/** \brief Check whether the given PCI device classid is important.
 *
 * \return 1 if important, 0 otherwise.
 */
static __hwloc_inline int
hwloc_filter_check_pcidev_subtype_important(unsigned classid)
{
  unsigned baseclass = classid >> 8;
  return (baseclass == 0x03 /* PCI_BASE_CLASS_DISPLAY */
	  || baseclass == 0x02 /* PCI_BASE_CLASS_NETWORK */
	  || baseclass == 0x01 /* PCI_BASE_CLASS_STORAGE */
	  || baseclass == 0x00 /* Unclassified, for Atos/Bull BXI */
	  || baseclass == 0x0b /* PCI_BASE_CLASS_PROCESSOR */
	  || classid == 0x0c04 /* PCI_CLASS_SERIAL_FIBER */
	  || classid == 0x0c06 /* PCI_CLASS_SERIAL_INFINIBAND */
          || classid == 0x0502 /* PCI_CLASS_MEMORY_CXL */
          || baseclass == 0x06 /* PCI_BASE_CLASS_BRIDGE with non-PCI downstream. the core will drop the useless ones later */
	  || baseclass == 0x12 /* Processing Accelerators */);
}

```

这两段代码定义了一个名为 `hwloc_filter_check_osdev_subtype_important` 的函数，其功能是检查给定的操作系统设备子类型是否重要。具体来说，如果给定的子类型不是 HWLOC_OBJ_OSDEV_DMA，那么函数返回 1，否则返回 0。

在实际应用中，这个函数可以用来对输入或输出设备进行筛选，将不重要的设备从输入或输出队列中排除。请注意，这个函数不能用于 I/O 设备，因为 I/O 设备已经被预设为只能被过滤。


```cpp
/** \brief Check whether the given OS device subtype is important.
 *
 * \return 1 if important, 0 otherwise.
 */
static __hwloc_inline int
hwloc_filter_check_osdev_subtype_important(hwloc_obj_osdev_type_t subtype)
{
  return (subtype != HWLOC_OBJ_OSDEV_DMA);
}

/** \brief Check whether a non-I/O object type should be filtered-out.
 *
 * Cannot be used for I/O objects.
 *
 * \return 1 if the object type should be kept, 0 otherwise.
 */
```

这两段代码是用来判断一个对象（在 hwloc_topology_t 和 hwloc_obj_t 分别表示）是否应该被保留，其中第一段代码是在函数 hwloc_filter_check_keep_object_type 中实现的，第二段代码是在函数 hwloc_filter_check_keep_object 中实现的。

在 hwloc_topology_t 和 hwloc_obj_t 中，使用的是 hwloc_topology_get_type_filter 函数获取类型过滤器，然后使用 if 语句判断当前对象类型是否属于需要保留的类型，如果是，则返回 1，否则返回 0。

在 hwloc_filter_check_keep_object_type 中，对类型为 hwloc_type_filter_keeper_none 和 hwloc_type_filter_keeper_important 的函数进行了自定义，逻辑与上面类似。

在 hwloc_filter_check_keep_object 中，对一个给定的对象进行保留判断，首先获取其类型，然后使用 hwloc_topology_get_type_filter 获取类型过滤器，接着判断类型是否属于需要保留的类型（如 hwloc_obj_pci_device 和 hwloc_obj_os_device ），如果是，则返回 1，否则返回 0。


```cpp
static __hwloc_inline int
hwloc_filter_check_keep_object_type(hwloc_topology_t topology, hwloc_obj_type_t type)
{
  enum hwloc_type_filter_e filter = HWLOC_TYPE_FILTER_KEEP_NONE;
  hwloc_topology_get_type_filter(topology, type, &filter);
  assert(filter != HWLOC_TYPE_FILTER_KEEP_IMPORTANT); /* IMPORTANT only used for I/O */
  return filter == HWLOC_TYPE_FILTER_KEEP_NONE ? 0 : 1;
}

/** \brief Check whether the given object should be filtered-out.
 *
 * \return 1 if the object type should be kept, 0 otherwise.
 */
static __hwloc_inline int
hwloc_filter_check_keep_object(hwloc_topology_t topology, hwloc_obj_t obj)
{
  hwloc_obj_type_t type = obj->type;
  enum hwloc_type_filter_e filter = HWLOC_TYPE_FILTER_KEEP_NONE;
  hwloc_topology_get_type_filter(topology, type, &filter);
  if (filter == HWLOC_TYPE_FILTER_KEEP_NONE)
    return 0;
  if (filter == HWLOC_TYPE_FILTER_KEEP_IMPORTANT) {
    if (type == HWLOC_OBJ_PCI_DEVICE)
      return hwloc_filter_check_pcidev_subtype_important(obj->attr->pcidev.class_id);
    if (type == HWLOC_OBJ_OS_DEVICE)
      return hwloc_filter_check_osdev_subtype_important(obj->attr->osdev.type);
  }
  return 1;
}

```

这段代码是一个名为 `hwlocality_components_pcidiscovery` 的函数，属于 PCI 发现组件。它包含了一些辅助函数，用于在 PCI 配置空间缓冲区中查找指定特性的偏移量。这个函数接受一个 256 字节的参数，其中包含一个 config space，这个 space 中已知的情况需要设置为 0xff，表示不知道该特性存在于哪个位置。函数返回一个指向特性偏移量的指针。


```cpp
/** @} */




/** \defgroup hwlocality_components_pcidisc Components and Plugins: helpers for PCI discovery
 *
 * \note These structures and functions may change when ::HWLOC_COMPONENT_ABI is modified.
 *
 * @{
 */

/** \brief Return the offset of the given capability in the PCI config space buffer
 *
 * This function requires a 256-bytes config space. Unknown/unavailable bytes should be set to 0xff.
 */
```

这两段代码定义了两个名为`hwloc_pcidisc_find_cap`和`hwloc_pcidisc_find_linkspeed`的函数，以及一个名为`hwloc_obj_type_check_bridge_type`的函数。

`hwloc_pcidisc_find_cap`函数的作用是读取PCI（Peripheral Component Interconnect，外围组件互连）配置空间中PCI_CAP_ID_EXP的位置，然后返回一个表示Linkspeed（链路速度）的整数。这个函数需要20个字节的数据，其中包括两个关键部分：一个长度为20的变量`exp_cap`，用于存储Linkspeed的配置空间部分；另一个长度为4的变量`linkspeed_offset`，用于存储offset部分。

`hwloc_pcidisc_find_linkspeed`函数的作用是返回给定的设备类别的hwloc对象类型。这个函数需要16个字节的配置头部，用于在头部中查找设备类别和配置空间。然后，它根据给定的offset偏移量，返回适当的链路速度类型。

`hwloc_obj_type_check_bridge_type`函数的作用是检查设备类别是否为Bridge，并返回其设备类别的字符串表示。这个函数需要4个字节的数据，用于在头部中查找设备类别。然后，它根据给定的设备类别，返回相应的字符串表示。


```cpp
HWLOC_DECLSPEC unsigned hwloc_pcidisc_find_cap(const unsigned char *config, unsigned cap);

/** \brief Fill linkspeed by reading the PCI config space where PCI_CAP_ID_EXP is at position offset.
 *
 * Needs 20 bytes of EXP capability block starting at offset in the config space
 * for registers up to link status.
 */
HWLOC_DECLSPEC int hwloc_pcidisc_find_linkspeed(const unsigned char *config, unsigned offset, float *linkspeed);

/** \brief Return the hwloc object type (PCI device or Bridge) for the given class and configuration space.
 *
 * This function requires 16 bytes of common configuration header at the beginning of config.
 */
HWLOC_DECLSPEC hwloc_obj_type_t hwloc_pcidisc_check_bridge_type(unsigned device_class, const unsigned char *config);

```

这两段代码是用于在给定的PCI桥中填充属性的函数。

第一段代码定义了一个名为hwloc_pcidisc_find_bridge_buses的函数，它接受一个PCI配置空间，然后返回一个整数表示找到的桥的 bus 数量。函数需要32个字节的最小公共配置头作为参数，并使用给定的PCI配置空间中的数据查找桥。如果找到的桥的属性无效，函数将返回-1并摧毁/p对象。

第二段代码定义了一个名为hwloc_pcidisc_tree_insert_by_busid的函数，它接受一个PCI树，一个桥对象，然后将桥对象插入到PCI树中。如果树头指向 NULL，则新的桥对象将被插入到树中。函数使用给定的PCI配置空间中的数据查找桥，并使用桥对象的hwloc_obj成员函数进行插入。


```cpp
/** \brief Fills the attributes of the given PCI bridge using the given PCI config space.
 *
 * This function requires 32 bytes of common configuration header at the beginning of config.
 *
 * Returns -1 and destroys /p obj if bridge fields are invalid.
 */
HWLOC_DECLSPEC int hwloc_pcidisc_find_bridge_buses(unsigned domain, unsigned bus, unsigned dev, unsigned func,
						   unsigned *secondary_busp, unsigned *subordinate_busp,
						   const unsigned char *config);

/** \brief Insert a PCI object in the given PCI tree by looking at PCI bus IDs.
 *
 * If \p treep points to \c NULL, the new object is inserted there.
 */
HWLOC_DECLSPEC void hwloc_pcidisc_tree_insert_by_busid(struct hwloc_obj **treep, struct hwloc_obj *obj);

```

这段代码定义了一个名为`hwloc_pcidisc_tree_attach`的函数，属于`hwlocality_components_pcifind`组。它的作用是在给定的树形PCI对象中添加一些额外的桥接，并将其附加到topology中。

该函数的参数有两个：一个`hwloc_topology`结构体，表示要操作的topology；另一个`hwloc_obj`结构体，表示要操作的obj。这两个结构体都包含topology中已识别的PCI对象的引用。

函数实现中，首先通过调用`hwloc_pcidisc_find_by_busid()`函数，来寻找与给定busid相匹配的PCI对象。如果找到了，则使用`hwloc_pcidisc_find_busid_parent()`函数获取其父节点，并将这两个节点连接到topology中，并将它们的parent_树设为根节点。这样，当前节点及其父节点都将被添加到topology中。

函数的实现旨在为其他寻找PCI对象的 backends提供一个通用的接口，即使是在其他backends中，可能需要使用类似`hwloc_pcidisc_find_by_busid()`或`hwloc_pcidisc_find_busid_parent()`这样的函数。


```cpp
/** \brief Add some hostbridges on top of the given tree of PCI objects and attach them to the topology.
 *
 * Other backends may lookup PCI objects or localities (for instance to attach OS devices)
 * by using hwloc_pcidisc_find_by_busid() or hwloc_pcidisc_find_busid_parent().
 */
HWLOC_DECLSPEC int hwloc_pcidisc_tree_attach(struct hwloc_topology *topology, struct hwloc_obj *tree);

/** @} */




/** \defgroup hwlocality_components_pcifind Components and Plugins: finding PCI objects during other discoveries
 *
 * \note These structures and functions may change when ::HWLOC_COMPONENT_ABI is modified.
 *
 * @{
 */

```

这段代码定义了一个名为 `hwloc_pci_find_parent_by_busid` 的函数，它的作用是查找一个PCI总线的设备或其父设备，根据指定的PCI ID。

函数接收一个 `struct hwloc_topology` 类型的上下文，包含一个 `unsigned domain` 成员，表示要查询的PCI域，以及一个 `unsigned bus` 成员，表示要查找的PCI ID。还接收一个 `unsigned dev` 成员，表示要查找的设备的设备号，和一个 `unsigned func` 成员，表示要查找的函数类型。

函数首先使用 `hwloc_topology_is_cpi_domain()` 函数检查传入的 `struct hwloc_topology` 是否包含指定的PCI域。如果是，则调用 `hwloc_pci_find_object()` 函数查找与PCI ID完全匹配的设备。否则，函数将返回具有类似定位的另一个对象，例如一个父桥或本地CPU套件。

函数的实现非常简单，只需要根据传入的参数查询PCI数据库，然后返回匹配的PCI设备或其父设备。


```cpp
/** \brief Find the object or a parent of a PCI bus ID.
 *
 * When attaching a new object (typically an OS device) whose locality
 * is specified by PCI bus ID, this function returns the PCI object
 * to use as a parent for attaching.
 *
 * If the exact PCI device with this bus ID exists, it is returned.
 * Otherwise (for instance if it was filtered out), the function returns
 * another object with similar locality (for instance a parent bridge,
 * or the local CPU Package).
 */
HWLOC_DECLSPEC struct hwloc_obj * hwloc_pci_find_parent_by_busid(struct hwloc_topology *topology, unsigned domain, unsigned bus, unsigned dev, unsigned func);

/** \brief Find the PCI device or bridge matching a PCI bus ID exactly.
 *
 * This is useful for adding specific information about some objects
 * based on their PCI id. When it comes to attaching objects based on
 * PCI locality, hwloc_pci_find_parent_by_busid() should be preferred.
 */
```

这段代码定义了一个名为hwloc_pci_find_by_busid的结构体指针变量hwloc_pci_find_by_busid，以及一个名为hwloc_backend_distances_add_handle_t的类型指针变量。hwloc_pci_find_by_busid函数用于在topology中通过总线ID查找并返回与该总线ID相关的hwloc_obj结构体指针。hwloc_backend_distances_add_handle_t定义了一个新的函数，名为hwloc_backend_distances_add_create，用于创建一个新的hwloc_backend_distances结构体。这个函数接受topology和名称参数，以及一个整数参数kind，表示距离类型的索引，以及一些标志参数。根据kind参数的值，这个函数会根据定义的索引返回相应的hwloc_distances结构体指针或hwloc_backend_distances_add_handle_t指针。


```cpp
HWLOC_DECLSPEC struct hwloc_obj * hwloc_pci_find_by_busid(struct hwloc_topology *topology, unsigned domain, unsigned bus, unsigned dev, unsigned func);

/** \brief Handle to a new distances structure during its addition to the topology. */
typedef void * hwloc_backend_distances_add_handle_t;

/** \brief Create a new empty distances structure.
 *
 * This is identical to hwloc_distances_add_create()
 * but this variant is designed for backend inserting
 * distances during topology discovery.
 */
HWLOC_DECLSPEC hwloc_backend_distances_add_handle_t
hwloc_backend_distances_add_create(hwloc_topology_t topology,
                                   const char *name, unsigned long kind,
                                   unsigned long flags);

```

这段代码定义了一个名为`hwloc_backend_distances_add_values`的函数，它的作用是创建一个新的空距离结构体，然后向其中添加指定数量的对象和值。

具体来说，该函数接受四个参数：

- `topology`：表示顶图类型。
- `handle`：表示距离添加 Handle。
- `nbobjs`：表示要添加的对象数量。
- `objs`：指向要添加对象的指针。
- `values`：指向要添加的值的指针。
- `flags`：表示添加值的标志，具体作用未知。

函数成功执行后，会将创建的距离结构体中的所有元素复制到`handle`所指位置，并不会再将其分配给调用者。


```cpp
/** \brief Specify the objects and values in a new empty distances structure.
 *
 * This is similar to hwloc_distances_add_values()
 * but this variant is designed for backend inserting
 * distances during topology discovery.
 *
 * The only semantical difference is that \p objs and \p values
 * are not duplicated, but directly attached to the topology.
 * On success, these arrays are given to the core and should not
 * ever be freed by the caller anymore.
 */
HWLOC_DECLSPEC int
hwloc_backend_distances_add_values(hwloc_topology_t topology,
                                   hwloc_backend_distances_add_handle_t handle,
                                   unsigned nbobjs, hwloc_obj_t *objs,
                                   hwloc_uint64_t *values,
                                   unsigned long flags);

```

这段代码定义了一个名为 `hwloc_backend_distances_add_commit` 的函数，它的作用是提交一个新的距离结构。这个函数的实现与 `hwloc_distances_add_commit()` 函数相似，但是这个函数是针对后端在拓扑发现过程中插入距离的。

具体来说，这个函数接受两个参数：`topology` 是一个表示拓扑结构的 `hwloc_topology_t` 类型的数据结构，`handle` 是距离结构的 `hwloc_backend_distances_add_handle_t` 类型的数据结构，这个数据结构存储了距离结构的引用。`flags` 是用来设置距离结构的一些标志，比如是否使用二进制编码等。

函数的实现中，首先调用 `hwloc_topology_backend_commit()` 函数，这个函数会将拓扑结构的信息保存到 `handle` 指向的距离结构中，然后将 flags 设置为输入的值。最后，函数返回这个 handle，以便后续操作使用。


```cpp
/** \brief Commit a new distances structure.
 *
 * This is similar to hwloc_distances_add_commit()
 * but this variant is designed for backend inserting
 * distances during topology discovery.
 */
HWLOC_DECLSPEC int
hwloc_backend_distances_add_commit(hwloc_topology_t topology,
                                   hwloc_backend_distances_add_handle_t handle,
                                   unsigned long flags);

/** @} */




```

这段代码是一个头文件声明，表示这是一个自定义的 C 语言源文件，它位于名为 "HWLOC_PLUGINS_H" 的头文件中。这个头文件可以被其他源文件包含，从而允许程序在运行时使用自定义的符号，定义，声明等等。

具体来说，这个头文件中定义了一些与代码生成、语义分析等相关的函数和变量，例如：

- `#ifdef` 和 `#ifndef` 预处理指令，用于检查当前源文件是否已经被定义为该头文件，如果没有被定义，则编译时会生成编译器无法识别的错误。
- `#include` 预处理指令，用于包含该头文件中定义的所有内容。
- `SCRIPT_MEMORY_POLICY` 定义了一个名为 "ScriptMemoryPolicy" 的变量，表示程序在分配和释放内存时采取的政策，例如，它可以防止缓冲区溢出等常见的安全漏洞。
- `ENABLE_C_INCLUDE_GLASS_NONE` 定义了一个名为 "EnableCIncludeGlnElision" 的变量，表示是否启用 C 语言中的 include 和 glasses 注释。
- `SUPPORT_WIN_MPI` 定义了一个名为 "SupportWinMpI" 的变量，表示程序是否支持 Windows 上的 MPI(Message Passing Interface)调用。

这个头文件的作用是定义了一些变量和函数，用于定义某些特定的文本和声明，并允许程序在运行时使用这些定义，以提供更好的代码可读性和可维护性。


```cpp
#endif /* HWLOC_PLUGINS_H */

```