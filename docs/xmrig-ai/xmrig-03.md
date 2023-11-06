# xmrig源码解析 3

# `src/3rdparty/adl/adl_sdk.h`

这段代码是一个C语言的程序，它将包含在名为"advanced_mci.c"的文件中。该程序是针对名为"advanced_mci.h"的头文件的源代码。

该程序的作用是为特定的硬件芯片提供驱动程序，该芯片是Advanced Micro Devices, Inc.（Advanced Micro Devices）的产品。它被授权为在软件方面不受任何限制地使用、复制、修改、分发的软件，包括在其免费的情况下用于商业目的。

程序的主要部分包括以下几个部分：

1. 一个版权声明，指出该软件的版权由Advanced Micro Devices, Inc.保留，禁止从该软件的任何部分或全部中直接或间接地复制或传输。

2. 一个MIT许可证，允许用户自由地使用、复制、修改、分发的软件，但限制在软件的原始文档中，并允许用户在软件的自由复制之上分配漫游许可。

3. 一个包含多个头文件的包含声明，这些头文件定义了程序需要使用的一些通用函数和数据结构。

4. 一个名为"advanced_mci.c"的源文件，其中包含程序的主要实现。

5. 一些函数声明和定义，指出程序需要使用的一些函数和变量。

6. 其他一些指令和注释，描述了程序的功能和使用说明。


```cpp
//
// Copyright (c) 2016 Advanced Micro Devices, Inc. All rights reserved.
//
// MIT LICENSE:
// Permission is hereby granted, free of charge, to any person obtaining a copy
// of this software and associated documentation files (the "Software"), to deal
// in the Software without restriction, including without limitation the rights
// to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
// copies of the Software, and to permit persons to whom the Software is
// furnished to do so, subject to the following conditions:
//
// The above copyright notice and this permission notice shall be included in
// all copies or substantial portions of the Software.
//
// THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
```

这段代码是一个头文件，定义了一个名为 "adl_sdk.h" 的头文件。这个头文件中包含一个内存分配回调函数的定义，定义了在内存分配时需要调用的函数和数据结构。这个头文件被包含在 "ADL SDK" 中，意味着它是一个 ADL SDK 中的库头文件。

具体来说，这个头文件中定义了一个名为 "Callback" 的结构体，它包含一个名为 "void*" 的指针，用于存储分配的内存的地址。这个结构体还包含一个名为 "int32" 的整型变量 "errorCode" 和一个名为 "int64" 的整型变量 "allocationSize"。

这个头文件中还定义了一个名为 "MemoryAllocationCallback" 的函数，它的参数是一个 "void*" 类型的指针，用于存储分配的内存的地址，以及一个 "int32" 类型的变量 "errorCode" 和一个 "int64" 类型的变量 "allocationSize"。这个函数使用了 "Callback" 结构体中的 "void*" 指针参数，以及 "int32" 类型的变量 "errorCode" 来存储在分配的内存中发生错误时返回的错误码。

此外，这个头文件中还定义了一个名为 "ADL_SDK_MEMORY_ALLOC_CALLBACK" 的函数，它是一个在 "Callback" 函数中调用的函数。这个函数将在分配内存时执行，并且它接受一个 "void*" 类型的指针和一个 "int32" 类型的整型变量 "allocationSize"。


```cpp
// IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
// FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
// AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
// LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
// OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
// SOFTWARE.

/// \file adl_sdk.h
/// \brief Contains the definition of the Memory Allocation Callback.\n <b>Included in ADL SDK</b>
///
/// \n\n
/// This file contains the definition of the Memory Allocation Callback.\n
/// It also includes definitions of the respective structures and constants.\n
/// <b> This is the only header file to be included in a C/C++ project using ADL </b>

```

这段代码定义了一个头文件叫“adl_sdk_h_”，接着定义了一个函数指针变量“__stdcall”，这个类型表示这个函数可以被stdcall函数调用。然后定义了一个名为“ADL_SDK_H_”的函数，但这个函数并没有定义函数体，因为在调用这个函数时，需要传递一个函数指针，所以还需要定义一个名为“ADL_MAIN_MALLOC_CALLBACK”的函数指针变量来存储函数的实现。接着定义了一个名为“void”类型的类型，用于声明一个函数，这个类型的函数有一个实参“int”，用于表示分配的内存大小。最后，在函数声明前面加了一个“#if defined (LINUX)”的注释，表示这个函数只能在Linux系统上编译通过。


```cpp
#ifndef ADL_SDK_H_
#define ADL_SDK_H_

#include "adl_structures.h"

#if defined (LINUX)
#define __stdcall
#endif /* (LINUX) */

/// Memory Allocation Call back
typedef void* ( __stdcall *ADL_MAIN_MALLOC_CALLBACK )( int );


#endif /* ADL_SDK_H_ */

```

# `src/3rdparty/adl/adl_structures.h`

这段代码是一个C语言的函数声明，它定义了一个名为"software\_file"的函数。函数声明包含一个通配符，这意味着该函数可以接受任何类型的文件名。

进一步分析表明，该函数的主要目的是用于在命令行中读取用户输入的文件名，并将其返回给函数调用者。它还包含一个版权声明和MIT许可证，以确保软件的版权得到保护，并且允许用户自由地使用、修改、复制、分发和分发软件。


```cpp
//
// Copyright (c) 2016 Advanced Micro Devices, Inc. All rights reserved.
//
// MIT LICENSE:
// Permission is hereby granted, free of charge, to any person obtaining a copy
// of this software and associated documentation files (the "Software"), to deal
// in the Software without restriction, including without limitation the rights
// to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
// copies of the Software, and to permit persons to whom the Software is
// furnished to do so, subject to the following conditions:
//
// The above copyright notice and this permission notice shall be included in
// all copies or substantial portions of the Software.
//
// THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
```

这段代码定义了一个名为 "adl\_structures.h" 的头文件，其中包含在 AMD Display Library (ADL) 的公共接口中使用的数据结构声明。这些数据结构用于实现支持和常见功能，如输入输出、内存管理、多线程/事件驱动等。这个头文件是 ADL 库的规范，定义了在 ADL 库中使用这些数据结构的标准接口，以便开发人员可以更轻松地使用这些功能。

这个头文件中定义了一系列数据结构，如整型、浮点型、字符型、指针、数组、结构体、联合体等，可以用于实现不同功能。例如，整型数据结构可以用于表示帧缓冲区的位置、大小、是否可用等；浮点型数据结构可以用于表示像素的顏色值；字符型数据结构可以用于表示文本字符串中的每个字符；指针数据结构可以用于表示数据在内存中的位置和大小等。

这个头文件中定义的数据结构可以被用于实现 ADL 库的各种功能，如显示设备管理、图形渲染、输入输入输出等。通过使用这些数据结构，开发人员可以更轻松地使用 ADL 库提供的功能，而不需要深入了解 ADL 库的内部实现。


```cpp
// IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
// FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
// AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
// LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
// OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
// SOFTWARE.

/// \file adl_structures.h
///\brief This file contains the structure declarations that are used by the public ADL interfaces for \ALL platforms.\n <b>Included in ADL SDK</b>
///
/// All data structures used in AMD Display Library (ADL) public interfaces should be defined in this header file.
///

#ifndef ADL_STRUCTURES_H_
#define ADL_STRUCTURES_H_

```

这段代码定义了一个名为AdapterInfo的结构体，它包含了关于显卡适配器的信息。这个结构体可以用来存储显卡的设置，也可以用来调用显卡的接口，以获取或设置各种显卡设置。

注意，这个结构体中有一个名为iSize的成员，它是一个整型变量，用于存储结构体本身的大小。


```cpp
#include "adl_defines.h"

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about the graphics adapter.
///
/// This structure is used to store various information about the graphics adapter.  This
/// information can be returned to the user. Alternatively, it can be used to access various driver calls to set
/// or fetch various settings upon the user's request.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct AdapterInfo
{
/// \ALL_STRUCT_MEM

/// Size of the structure.
    int iSize;
```

这段代码定义了一个名为 "ADLindexHandle" 的结构体，用于表示一个 ADL（Advanced Link-layer）设备索引。这个结构体包含了一些与 ADL 设备相关的整数变量，如设备 ID、总线 ID、设备号、功能号和厂商 ID 等。

这个结构体可以用来在代码中引用一个 ADL 设备，以便访问它上面的资源，比如输入输出、存储设备等。在代码中，可以像访问普通变量一样来引用这个结构体，它会根据结构体中包含的变量值来执行相应的操作。

例如，假设有一个名为 "myADLDevice" 的 ADL 设备，它的 ID 是 "001"，总线 ID 是 "002"，设备号是 "003"，功能号是 "004"，厂商 ID 是 "005"。那么，可以使用以下代码来访问这个设备：
```cpp
// 初始化
int iAdapterIndex = 0;
char strUDID[ADL_MAX_PATH];
int iBusNumber = 0;
int iDeviceNumber = 0;
int iFunctionNumber = 0;
int iVendorID = 0;

// 获取设备名称
char strAdapterName[ADL_MAX_PATH];
char strDisplayName[ADL_MAX_PATH];

// 初始化
strAdapterName[ADL_MAX_PATH] = '\\';
strDisplayName[ADL_MAX_PATH] = strADLDevice;
```
在这个示例中，变量 "myADLDevice" 被初始化为 "001"。然后，通过一系列循环和字符串拼接，得到了设备名称，并将其存储到了 "strAdapterName" 变量中，同时也存储到了 "strDisplayName" 变量中。

接下来，可以使用这个设备名称来访问设备上面的资源，例如通过输入输出、存储设备等。


```cpp
/// The ADL index handle. One GPU may be associated with one or two index handles
    int iAdapterIndex;
/// The unique device ID associated with this adapter.
    char strUDID[ADL_MAX_PATH];
/// The BUS number associated with this adapter.
    int iBusNumber;
/// The driver number associated with this adapter.
    int iDeviceNumber;
/// The function number.
    int iFunctionNumber;
/// The vendor ID associated with this adapter.
    int iVendorID;
/// Adapter name.
    char strAdapterName[ADL_MAX_PATH];
/// Display name. For example, "\\\\Display0" for Windows or ":0:0" for Linux.
    char strDisplayName[ADL_MAX_PATH];
```

这段代码定义了一个名为“iPresent”的整型变量，并进行了初始化。接下来，代码根据定义的平台（在注释中使用了_WIN32 和 _WIN64）来检查是否支持 PNP（Physical Network Programming）并定义了相应的变量。

具体来说，这段代码实现了以下功能：

1. 如果定义的平台支持 PNP，则定义了变量 iExist 和 strDriverPath，strDriverPathExt 和 PNPString，其中 iExist 表示 Image Capture and Device Driver 的存在与否，strDriverPath 和 strDriverPathExt 用于存储 PNP 应用程序注册表路径和文件路径，PNPString 用于存储 PNP 应用程序的显示名称。
2. 如果定义的平台不支持 PNP，则没有定义这些变量，不过仍然初始化了整型变量 iPresent。
3. 通过逻辑适应器（if defined）来显示或not存在，根据定义的平台，iPresent 的值会在以下情况下设置为1：
	* 在 Windows 平台上，当定义平台为 _WIN32 时。
	* 在 Windows 10 1803 或更高版本时，硬件设备驱动程序支持 PNP。

总之，这段代码定义了一些变量，并根据定义的平台来检查是否支持 PNP，从而实现了 Image Capture and Device Driver 的注册和卸载。


```cpp
/// Present or not; 1 if present and 0 if not present.It the logical adapter is present, the display name such as \\\\.\\Display1 can be found from OS
    int iPresent;

#if defined (_WIN32) || defined (_WIN64)
/// \WIN_STRUCT_MEM

/// Exist or not; 1 is exist and 0 is not present.
    int iExist;
/// Driver registry path.
    char strDriverPath[ADL_MAX_PATH];
/// Driver registry path Ext for.
    char strDriverPathExt[ADL_MAX_PATH];
/// PNP string from Windows.
    char strPNPString[ADL_MAX_PATH];
/// It is generated from EnumDisplayDevices.
    int iOSDisplayIndex;
```

这段代码是一个C语言的include文件，它包含了一些定义和宏。这里简要解释一下每个部分的含义：

1. `#ifdef (_WIN32) || (_WIN64)`：这是一个条件编译语句，它判断当前系统是否为Windows32位或者Windows64位。如果为Windows32位，则执行`#include <windows.h>`，否则执行`#include <stdio.h>`。

2. `#if defined(LINUX)`：这是一个判断语句，它判断当前系统是否为Linux。如果是，那么下面所有的代码都将执行。

3. `/// <封装，声明名为iXScreenNum的内部整数类型>`：这是一个声明，它声明了一个名为iXScreenNum的内部整数类型，用于表示内部X屏幕的编号。

4. `/// <封装，声明名为iDrvIndex的内部整数类型>`：这是一个声明，它声明了一个名为iDrvIndex的内部整数类型，用于表示硬件设备驱动程序的索引。

5. `/// <封装，声明名为strXScreenConfigName的内部字符串类型>`：这是一个声明，它声明了一个名为strXScreenConfigName的内部字符串类型，用于存储设备的配置文件名。这个配置文件名使用`strXScreenConfigName`。

6. `#endif`：这是另一个条件编译语句，它用于将所有定义过的部分排除，只有当当前系统不是Linux时才执行。

7. `#include "adapterinfo.h"`：这是引入adapterinfo.h文件的语句，它将定义了所有与AdapterInfo结构体相关的成员。

8. `#include "x視野信息.h"`：这是引入x视野信息.h文件的语句，它将定义了所有与X屏幕信息相关的成员。

总结：这段代码定义了一个名为AdapterInfo的设备驱动程序结构体，包含了一个内部整数类型iXScreenNum和一个内部整数类型iDrvIndex，以及一个内部字符串类型strXScreenConfigName。它还声明了一个名为AdaptationInfo的辅助结构体，用于提供与硬件抽象层相关的信息。这个设备驱动程序结构体可能是在Windows操作系统上开发的，因为它依赖于操作系统特定的函数和头文件。


```cpp
#endif /* (_WIN32) || (_WIN64) */

#if defined (LINUX)
/// \LNX_STRUCT_MEM

/// Internal X screen number from GPUMapInfo (DEPRICATED use XScreenInfo)
    int iXScreenNum;
/// Internal driver index from GPUMapInfo
    int iDrvIndex;
/// \deprecated Internal x config file screen identifier name. Use XScreenInfo instead.
    char strXScreenConfigName[ADL_MAX_PATH];

#endif /* (LINUX) */
} AdapterInfo, *LPAdapterInfo;

```

这段代码定义了一个名为 XScreenInfo 的结构体，用于存储与 Linux X 屏幕相关的信息。这个结构体包含两个成员变量：iXScreenNum 和 strXScreenConfigName。iXScreenNum 表示内部 X 屏幕的编号，而 strXScreenConfigName 则存储了与屏幕配置相关的配置文件名称。

在说明这个结构体时，特别提到了应该使用这个结构体代替 iXScreenNum 和 strXScreenConfigName，因为它们已经被 deprecated（过时）。

此外，在代码的最后，特别提到了 Linux，这意味着这个代码是在 Linux 系统上运行的。


```cpp
/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about the Linux X screen information.
///
/// This structure is used to store the current screen number and xorg.conf ID name assoicated with an adapter index.
/// This structure is updated during ADL_Main_Control_Refresh or ADL_ScreenInfo_Update.
/// Note:  This structure should be used in place of iXScreenNum and strXScreenConfigName in AdapterInfo as they will be
/// deprecated.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
#if defined (LINUX)
typedef struct XScreenInfo
{
/// Internal X screen number from GPUMapInfo.
    int iXScreenNum;
/// Internal x config file screen identifier name.
    char strXScreenConfigName[ADL_MAX_PATH];
} XScreenInfo, *LPXScreenInfo;
```

这段代码是一个 C 语言中的预处理指令，它包含在一个头文件中。它的作用是检查定义结构体 ADLMemoryInfo 的版本是否与 Linux 系统上 ASIC（Application Specific Integrated Circuit）内存相关的支持版本一致。

具体来说，如果 ASIC 内存支持的版本与预定义的结构体不符合，那么预处理指令会编译时失败，从而拒绝编译。如果版本一致，则允许编译。这个预处理指令的作用是确保代码在不同的 ASIC 内存支持版本下能够正常编译。


```cpp
#endif /* (LINUX) */


/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about the ASIC memory.
///
/// This structure is used to store various information about the ASIC memory.  This
/// information can be returned to the user.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLMemoryInfo
{
/// Memory size in bytes.
    long long iMemorySize;
/// Memory type in string.
    char strMemoryType[ADL_MAX_PATH];
```

这段代码定义了一个名为 ADLMemoryInfo 的结构体，该结构体包含有关内存带宽的信息。该结构体定义了 long 类型的大 iMemoryBandwidth 成员，用于表示内存带宽，以 Mbytes/s 为单位。

同时，该代码定义了一个名为 ADLMemoryRequired 的结构体，该结构体定义了内存的详细信息，包括内存大小、使用的内存类型以及用于该内存类型的显示功能。

该代码的作用是定义结构体，以便在程序中能够使用这些结构体，以便根据需要的内存带宽信息来配置 ADL 适配器。


```cpp
/// Memory bandwidth in Mbytes/s.
    long long iMemoryBandwidth;
} ADLMemoryInfo, *LPADLMemoryInfo;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about memory required by type
///
/// This structure is returned by ADL_Adapter_ConfigMemory_Get, which given a desktop and display configuration
/// will return the Memory used.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLMemoryRequired
{
    long long iMemoryReq;        /// Memory in bytes required
    int iType;                    /// Type of Memory \ref define_adl_validmemoryrequiredfields
    int iDisplayFeatureValue;   /// Display features \ref define_adl_visiblememoryfeatures that are using this type of memory
} ADLMemoryRequired, *LPADLMemoryRequired;

```

这段代码定义了一个名为 ADLMemoryDisplayFeatures 的结构体，用于表示与显示相关的信息。该结构体包含两个成员变量：iDisplayIndex 和 iDisplayFeatureValue。iDisplayIndex 表示显示的索引，而 iDisplayFeatureValue 则表示显示正在使用的功能。

结构体中的定义参考了定义的 define_adl_visiblememoryfeatures，该函数返回了与显示相关的信息。

该结构体用于某个名为 ADL_Adapter_ConfigMemory_Get 的函数，该函数根据给定的桌面和显示配置返回内存使用情况。通过传入结构体，可以更方便地获取与显示相关的信息。


```cpp
/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about the features associated with a display
///
/// This structure is a parameter to ADL_Adapter_ConfigMemory_Get, which given a desktop and display configuration
/// will return the Memory used.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLMemoryDisplayFeatures
{
    int iDisplayIndex;            /// ADL Display index
    int iDisplayFeatureValue;    /// features that the display is using \ref define_adl_visiblememoryfeatures
} ADLMemoryDisplayFeatures, *LPADLMemoryDisplayFeatures;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing DDC information.
```

这段代码定义了一个名为ADLDDCInfo的结构体，用于存储各种DDC信息，以便在需要时返回给用户。该结构体包含四个成员变量：ulSize表示结构体的大小，ulSupportsDDC表示是否支持DDC，ulManufacturerID表示显示设备的制造商ID，ulProductID表示显示设备的product ID。

注意，该结构体所有成员变量都被定义为无符号整数类型，这意味着它们的值可以覆盖任何无符号整数类型的值。另外，ulSupportsDDC和ulManufacturerID成员变量在结构体中被声明为unsigned int类型，这意味着它们的值可以表示0到255之间的任何整数。


```cpp
///
/// This structure is used to store various DDC information that can be returned to the user.
/// Note that all fields of type int are actually defined as unsigned int types within the driver.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLDDCInfo
{
/// Size of the structure
    int  ulSize;
/// Indicates whether the attached display supports DDC. If this field is zero on return, no other DDC information fields will be used.
    int  ulSupportsDDC;
/// Returns the manufacturer ID of the display device. Should be zeroed if this information is not available.
    int  ulManufacturerID;
/// Returns the product ID of the display device. Should be zeroed if this information is not available.
    int  ulProductID;
```

这段代码定义了一个名为 ADLDDCInfo 的结构体，用于返回显示设备的各种信息。如果没有可用的信息，则该结构体为空。

以下是结构体中可用的所有成员：

1. cDisplayName：用于存储显示设备的名称，最大长度为 ADL_MAX_DISPLAY_NAME。

2. ulMaxHResolution：用于存储最大水平分辨率，如果没有可用的信息，则该成员为 0。

3. ulMaxVResolution：用于存储最大垂直分辨率，如果没有可用的信息，则该成员为 0。

4. ulMaxRefresh：用于存储最大刷新率，如果没有可用的信息，则该成员为 0。

5. ulPTMCx：用于存储显示设备 preferred timing mode 的水平分辨率。

6. ulPTMCy：用于存储显示设备 preferred timing mode 的垂直分辨率。

7. ulPTMRefreshRate：用于存储显示设备 preferred timing mode 的刷新率。

8. ulDDCInfoFlag：用于存储是否有可用的 DDC（Direct-X Descriptor控制器）信息。

此外，还有两个指针成员，用于指向 ADLDDCInfo 类型的结构体，以便在需要时进行复制或修改。


```cpp
/// Returns the name of the display device. Should be zeroed if this information is not available.
    char cDisplayName[ADL_MAX_DISPLAY_NAME];
/// Returns the maximum Horizontal supported resolution. Should be zeroed if this information is not available.
    int  ulMaxHResolution;
/// Returns the maximum Vertical supported resolution. Should be zeroed if this information is not available.
    int  ulMaxVResolution;
/// Returns the maximum supported refresh rate. Should be zeroed if this information is not available.
    int  ulMaxRefresh;
/// Returns the display device preferred timing mode's horizontal resolution.
    int  ulPTMCx;
/// Returns the display device preferred timing mode's vertical resolution.
    int  ulPTMCy;
/// Returns the display device preferred timing mode's refresh rate.
    int  ulPTMRefreshRate;
/// Return EDID flags.
    int  ulDDCInfoFlag;
} ADLDDCInfo, *LPADLDDCInfo;

```

这段代码定义了一个名为ADLDDCInfo2的结构体，用于存储与DDC（显示数据传输）相关的信息。该结构体包含两个整型成员变量：ulSize和ulSupportsDDC，分别表示信息结构体的大小和显示是否支持DDC。在函数声明中，还声明了一个名为ulManufacturerID的整型成员变量，用于存储显示设备制造商的ID。如果该成员变量在函数返回时为零，则不会使用任何与显示设备制造商相关的信息。


```cpp
/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing DDC information.
///
/// This structure is used to store various DDC information that can be returned to the user.
/// Note that all fields of type int are actually defined as unsigned int types within the driver.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLDDCInfo2
{
/// Size of the structure
    int  ulSize;
/// Indicates whether the attached display supports DDC. If this field is zero on return, no other DDC
/// information fields will be used.
    int  ulSupportsDDC;
/// Returns the manufacturer ID of the display device. Should be zeroed if this information is not available.
    int  ulManufacturerID;
```

这段代码定义了五个整型变量：ulProductID、cDisplayName、ulMaxHResolution、ulMaxVResolution和ulMaxRefresh，它们分别代表显示设备的ID、显示设备的名称、最大水平分辨率、最大垂直分辨率和支持的最大刷新率。如果这些信息不可用，则它们的值会被设置为零。


```cpp
/// Returns the product ID of the display device. Should be zeroed if this information is not available.
    int  ulProductID;
/// Returns the name of the display device. Should be zeroed if this information is not available.
    char cDisplayName[ADL_MAX_DISPLAY_NAME];
/// Returns the maximum Horizontal supported resolution. Should be zeroed if this information is not available.
    int  ulMaxHResolution;
/// Returns the maximum Vertical supported resolution. Should be zeroed if this information is not available.
    int  ulMaxVResolution;
/// Returns the maximum supported refresh rate. Should be zeroed if this information is not available.
    int  ulMaxRefresh;
/// Returns the display device preferred timing mode's horizontal resolution.
    int  ulPTMCx;
/// Returns the display device preferred timing mode's vertical resolution.
    int  ulPTMCy;
/// Returns the display device preferred timing mode's refresh rate.
    int  ulPTMRefreshRate;
```

这段代码定义了EDID属性的值，其中包含了显示是否支持 packed pixel以及显示支持的像素格式等属性。

具体来说，代码定义了以下EDID属性：

- ulDDCInfoFlag: 用于指示是否支持显示驱动器，也就是是否支持 packed pixel。值为0时，表示不支持 packed pixel；值为1时，表示支持 packed pixel。
- bPackedPixelSupported: 用于指示显示是否支持packed pixel。值为0时，表示不支持packed pixel；值为1时，表示支持packed pixel。
- iPanelPixelFormat: 用于指示显示支持的packed pixel格式。根据定义_ddcinfo_pixelformats结构体，可以得到该属性的值。
- ulSerialID: 用于指示EDID序列号。
- ulMinLuminanceData: 用于指示最小显示器亮度。
- ulAvgLuminanceData: 用于指示平均显示器亮度。
- ulMaxLuminanceData: 用于指示最大显示器亮度。


```cpp
/// Return EDID flags.
    int  ulDDCInfoFlag;
/// Returns 1 if the display supported packed pixel, 0 otherwise
    int bPackedPixelSupported;
/// Returns the Pixel formats the display supports \ref define_ddcinfo_pixelformats
    int iPanelPixelFormat;
/// Return EDID serial ID.
    int  ulSerialID;
/// Return minimum monitor luminance data
    int ulMinLuminanceData;
/// Return average monitor luminance data
    int ulAvgLuminanceData;
/// Return maximum monitor luminance data
    int ulMaxLuminanceData;

```

这段代码定义了一个位图数组，用于表示支持的不同传输函数和颜色空间。这个数组可能有以下作用：

1. 存储功能支持的所有传输函数。
2. 存储颜色空间识别码。
3. 计算显示红色色度（X轴和Y轴）的百分比。
4. 计算显示绿色和蓝色色度的百分比。
5. 显示红色、绿色和蓝色色度的百分比。


```cpp
/// Bit vector of supported transfer functions \ref define_source_content_TF
    int iSupportedTransferFunction;

/// Bit vector of supported color spaces \ref define_source_content_CS
    int iSupportedColorSpace;

/// Display Red Chromaticity X coordinate multiplied by 10000
    int iNativeDisplayChromaticityRedX;
/// Display Red Chromaticity Y coordinate multiplied by 10000
    int iNativeDisplayChromaticityRedY;
/// Display Green Chromaticity X coordinate multiplied by 10000
    int iNativeDisplayChromaticityGreenX;
/// Display Green Chromaticity Y coordinate multiplied by 10000
    int iNativeDisplayChromaticityGreenY;
/// Display Blue Chromaticity X coordinate multiplied by 10000
    int iNativeDisplayChromaticityBlueX;
```

这段代码定义了五个整型变量：iNativeDisplayChromaticityBlueY、iNativeDisplayChromaticityWhitePointX、iNativeDisplayChromaticityWhitePointY、iDiffuseScreenReflectance、iSpecularScreenReflectance。它们的作用是：

1. iNativeDisplayChromaticityBlueY：表示蓝色版本的 chromaticity，即在颜色空间中，亮度（Brightness）和对比度（Contrast）之比。
2. iNativeDisplayChromaticityWhitePointX：表示白色部分的坐标，用于计算屏幕的亮度。
3. iNativeDisplayChromaticityWhitePointY：表示白色部分的坐标，用于计算屏幕的对比度。
4. iDiffuseScreenReflectance：表示扩散屏幕的反射率，范围为 0-1，表示为 0.01 的单位。
5. iSpecularScreenReflectance：表示定向屏幕的反射率，范围为 0-1，表示为 0.01 的单位。
6. iSupportedHDR：表示支持 HDR 技术的比特向量。
7. iFreesyncFlags：表示 FreeSync 标志，当该值为 1 时，表示操作系统支持 FreeSync 技术，可以在更高帧率下运行游戏，但游戏本身不支持该技术。


```cpp
/// Display Blue Chromaticity Y coordinate multiplied by 10000
    int iNativeDisplayChromaticityBlueY;
/// Display White Point X coordinate multiplied by 10000
    int iNativeDisplayChromaticityWhitePointX;
/// Display White Point Y coordinate multiplied by 10000
    int iNativeDisplayChromaticityWhitePointY;
/// Display diffuse screen reflectance 0-1 (100%) in units of 0.01
    int iDiffuseScreenReflectance;
/// Display specular screen reflectance 0-1 (100%) in units of 0.01
    int iSpecularScreenReflectance;
/// Bit vector of supported color spaces \ref define_HDR_support
    int iSupportedHDR;
/// Bit vector for freesync flags
    int iFreesyncFlags;

```

这段代码定义了一个名为ADLDDCInfo2的结构体，该结构体包含了关于显示控制器（例如电视、显示器） gamma 设置的信息。接下来，我们将逐步解释这个结构体的各个成员：

1. ulMinLuminanceNoDimmingData：这是一个整型变量，用于存储最小亮度值，在没有调光数据的情况下。

2. ulMaxBacklightMaxLuminanceData：这是一个整型变量，用于存储最大背光亮度值。

3. ulMinBacklightMaxLuminanceData：这是一个整型变量，用于存储最小背光亮度值。

4. ulMaxBacklightMinLuminanceData：这是一个整型变量，用于存储最大背光亮度值。

5. ulMinBacklightMinLuminanceData：这是一个整型变量，用于存储最小背光亮度值。

6. iReserved：这是一个保留的4个整型变量，后续可能被用于将来。

7. LPADLDDCInfo2：这是一个指向名为LPADLDDCInfo2的函数指针。这个函数指针可能用于在代码中使用，但不会在代码中实际执行。

总之，这段代码定义了一个用于存储显示控制器 gamma 设置信息的结构体。这些设置信息对显示设备的亮度有着重要影响。


```cpp
/// Return minimum monitor luminance without dimming data
    int ulMinLuminanceNoDimmingData;

    int ulMaxBacklightMaxLuminanceData;
    int ulMinBacklightMaxLuminanceData;
    int ulMaxBacklightMinLuminanceData;
    int ulMinBacklightMinLuminanceData;

    // Reserved for future use
    int iReserved[4];
} ADLDDCInfo2, *LPADLDDCInfo2;


/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information controller Gamma settings.
```

这段代码定义了一个名为ADLGamma的结构体，用于存储红色、绿色和蓝色通道的伽马设置。这个结构体可以被用来返回ADL（运动捕捉和运动处理引擎）中的伽马设置，也可以被用来设置控制器自身的伽马设置。伽马设置的值可以分别从红色、绿色和蓝色通道的属性中获取。


```cpp
///
/// This structure is used to store the red, green and blue color channel information for the.
/// controller gamma setting. This information is returned by ADL, and it can also be used to
/// set the controller gamma setting.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLGamma
{
/// Red color channel gamma value.
    float fRed;
/// Green color channel gamma value.
    float fGreen;
/// Blue color channel gamma value.
    float fBlue;
} ADLGamma, *LPADLGamma;

```

这段代码定义了一个名为ADLCustomMode的结构体，用于存储组件视频的定制模式信息。这个结构体包含四个成员变量：iFlags、iModeWidth、iModeHeight和iBaseModeWidth。

iFlags成员变量表示组件视频的定制模式 flags，iModeWidth和iModeHeight成员变量表示组件在屏幕上的尺寸，iBaseModeWidth成员变量表示在组件渲染时使用的模式宽度。这些信息都是用来告诉硬件如何在屏幕上正确地显示组件的。


```cpp
/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about component video custom modes.
///
/// This structure is used to store the component video custom mode.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLCustomMode
{
/// Custom mode flags.  They are returned by the ADL driver.
    int iFlags;
/// Custom mode width.
    int iModeWidth;
/// Custom mode height.
    int iModeHeight;
/// Custom mode base width.
    int iBaseModeWidth;
```

这段代码定义了一个名为ADLCustomMode的结构体，其中包含两个整型变量iBaseModeHeight和iRefreshRate。

ADLCustomMode结构体定义了ClockInfo结构体，该结构体包含以下成员：

- ulHighCoreClock：OD5（Output Device 5）的基带时钟（High Core Clock）的值。
- ulHighMemoryClock：OD5的内存时钟（High Memory Clock）的值。
- ulHighVddc：OD5的VCC时钟（High VDDc）的值。
- ulCoreMin：OD5芯片的最低核心时钟（Core Min）的值。
- ulCoreMax：OD5芯片的最高核心时钟（Core Max）的值。
- ulMemoryMin：OD5芯片的最低内存时钟（Memory Min）的值。
- ulMemoryMax：OD5芯片的最高内存时钟（Memory Max）的值。
- ulActivityPercent：OD5芯片的活动百分比。
- ulCurrentCoreClock：OD5芯片的当前核心时钟（Current Core Clock）的值。
- ulCurrentMemoryClock：OD5芯片的当前内存时钟（Current Memory Clock）的值。
- ulReserved：保留，用于填充字节。

这个ClockInfo结构体可能用于从OD5设备获取时钟信息，并用于控制OD5设备的工作状态和时间基准。


```cpp
/// Custom mode base height.
    int iBaseModeHeight;
/// Custom mode refresh rate.
    int iRefreshRate;
} ADLCustomMode, *LPADLCustomMode;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing Clock information for OD5 calls.
///
/// This structure is used to retrieve clock information for OD5 calls.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLGetClocksOUT
{
    long ulHighCoreClock;
    long ulHighMemoryClock;
    long ulHighVddc;
    long ulCoreMin;
    long ulCoreMax;
    long ulMemoryMin;
    long ulMemoryMax;
    long ulActivityPercent;
    long ulCurrentCoreClock;
    long ulCurrentMemoryClock;
    long ulReserved;
} ADLGetClocksOUT;

```

这是一段C语言代码，定义了一个名为ADLDisplayConfig的结构体。这个结构体用于存储高清电视（HDTV）的信息，以便在显示调用时进行检索。

该代码的结构体包含了以下字段：

1. ulSize：结构体的大小。
2. ulConnectorType：高清电视连接器的类型。
3. ulDeviceData：设备的高清数据。
4. ulOverriddenDeviceData：是否重置设备的高清数据。

这个结构体可以被用于从数据库或其他数据源获取高清电视信息，并在显示调用时使用。


```cpp
/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing HDTV information for display calls.
///
/// This structure is used to retrieve HDTV information information for display calls.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLDisplayConfig
{
/// Size of the structure
  long ulSize;
/// HDTV connector type.
  long ulConnectorType;
/// HDTV capabilities.
  long ulDeviceData;
/// Overridden HDTV capabilities.
  long ulOverridedDeviceData;
```

这段代码定义了一个名为ADLDisplayConfig的结构体，它包含了关于显示设备的信息，如显示器索引、类型、名称、连接状态、映射的适配器编号和控制器编号等。这个结构体可以用来存储显示设备的相关信息，并且可以被用来访问各种display driver的调用，以设置或获取各种与显示设备相关的设置。

ADLDisplayConfig结构体中的ulReserved字段是一个long类型的保留字段，它用于保留显示设备信息中的某些字段，以避免在使用时发生类型转换错误。


```cpp
/// Reserved field
  long ulReserved;
} ADLDisplayConfig;


/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about the display device.
///
/// This structure is used to store display device information
/// such as display index, type, name, connection status, mapped adapter and controller indexes,
/// whether or not multiple VPUs are supported, local display connections or not (through Lasso), etc.
/// This information can be returned to the user. Alternatively, it can be used to access various driver calls to set
/// or fetch various display device related settings upon the user's request.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
```

这段代码定义了一个名为 ADLDisplayID 的结构体，它用于表示一个Adapter 的显示信息。

这个结构体包含两个整型成员：

1. iDisplayLogicalIndex：这个数字表示属于这个Adapter 的逻辑显示位置，从0开始。对于每个Adapter，这个数字都是唯一的。
2. iDisplayPhysicalIndex：这个数字表示属于这个Adapter 的物理显示位置。对于每个Adapter，这个数字可以从0开始，但是不能与iDisplayLogicalIndex冲突。
3. iDisplayLogicalAdapterIndex：这个整型成员表示属于这个Display的逻辑Adapter的物理位置。如果这个Display有两个逻辑位置，那么这个成员的值就是逻辑位置的下标，从0开始。

此外，还有一条宏定义：
```cpp
typedef struct ADLDisplayID {
   int iDisplayLogicalIndex;
   int iDisplayPhysicalIndex;
   int iDisplayLogicalAdapterIndex;
} ADLDisplayID;
```
这个宏定义将上述结构体成员映射为了变量名。


```cpp
typedef struct ADLDisplayID
{
/// The logical display index belonging to this adapter.
    int iDisplayLogicalIndex;

///\brief The physical display index.
/// For example, display index 2 from adapter 2 can be used by current adapter 1.\n
/// So current adapter may enumerate this adapter as logical display 7 but the physical display
/// index is still 2.
    int iDisplayPhysicalIndex;

/// The persistent logical adapter index for the display.
    int iDisplayLogicalAdapterIndex;

///\brief The persistent physical adapter index for the display.
```

这段代码定义了一个名为ADLDisplayInfo的结构体类型，该类型包含关于显示设备的各种信息。其中包括当前显示适配器（如果存在）的物理ID和显示器名称。

进一步地，该代码定义了一个名为ADLDisplayID的整型变量和一个名为LPADLDisplayID的指针变量。这两个变量都指向同一个结构体类型的变量，但它们的索引值不同。

最后，该代码在定义了ADLDisplayInfo类型之后，没有对它进行任何操作，因此它不会输出任何信息。


```cpp
/// It can be the current adapter or a non-local adapter. \n
/// If this adapter index is different than the current adapter,
/// the Display Non Local flag is set inside DisplayInfoValue.
    int iDisplayPhysicalAdapterIndex;
} ADLDisplayID, *LPADLDisplayID;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about the display device.
///
/// This structure is used to store various information about the display device.  This
/// information can be returned to the user, or used to access various driver calls to set
/// or fetch various display-device-related settings upon the user's request
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLDisplayInfo
{
```

这段代码定义了一个名为 DisplayID 的结构体，其中包括了控制器索引、显示名称、制造商名称和显示类型等信息。

具体来说，这段代码定义了以下成员变量：

1. 一个名为 displayID 的 ADL 显示 ID 结构体变量。
2. 一个名为 iDisplayControllerIndex 的整型变量，用于存储与显示器之间的控制器索引。
3. 一个名为 strDisplayName 的字符型变量，用于存储显示器的名称。
4. 一个名为 strDisplayManufacturerName 的字符型变量，用于存储显示器的制造商名称。
5. 一个名为 iDisplayType 的整型变量，用于存储显示器的类型，例如 CRT、TV、CV、DPP 等。

此外，还有一段 deprecated 的注释，表示该代码将在未来的版本中不再使用。


```cpp
/// The DisplayID structure
    ADLDisplayID displayID;

///\deprecated The controller index to which the display is mapped.\n Will not be used in the future\n
    int  iDisplayControllerIndex;

/// The display's EDID name.
    char strDisplayName[ADL_MAX_PATH];

/// The display's manufacturer name.
    char strDisplayManufacturerName[ADL_MAX_PATH];

/// The Display type. For example: CRT, TV, CV, DFP.
    int  iDisplayType;

```

这段代码定义了一个名为ADLDisplayInfo的结构体，用于描述视频设备的显示输出和连接器类型等信息。

```cpp
/// The display output type. For example: HDMI, SVIDEO, COMPONMNET VIDEO.
   int  iDisplayOutputType;

/// The connector type for the device.
   int  iDisplayConnector;

///\brief The bit mask identifies the number of bits ADLDisplayInfo is currently using. \n
/// It will be the sum all the bit definitions in ADL_DISPLAY_DISPLAYINFO_xxx.
   int  iDisplayInfoMask;

/// The bit mask identifies the display status. \ref define_displayinfomask
   int  iDisplayInfoValue;
```

ADLDisplayInfo结构体包含三个成员变量：iDisplayOutputType，iDisplayConnector和iDisplayInfoMask。iDisplayOutputType表示显示设备的输出类型，如HDMI、SVIDEO或COMPONMNET VIDEO。iDisplayConnector用于标识连接器类型。iDisplayInfoMask是一个位图，用于标识ADL_DISPLAY_DISPLAYINFO_xxx中的信息。iDisplayInfoValue用于指示ADL_DISPLAY_DISPLAYINFO_xxx中显示设备当前使用的信息。

这段代码定义了一个名为ADLDisplayInfo的结构体，它可以用来描述一个视频设备的显示输出和连接器类型等信息。这个结构体可以在应用程序中用来获取和设置一个ADLDisplayInfo对象的值，从而实现在线视频播放等操作。


```cpp
/// The display output type. For example: HDMI, SVIDEO, COMPONMNET VIDEO.
    int  iDisplayOutputType;

/// The connector type for the device.
    int  iDisplayConnector;

///\brief The bit mask identifies the number of bits ADLDisplayInfo is currently using. \n
/// It will be the sum all the bit definitions in ADL_DISPLAY_DISPLAYINFO_xxx.
    int  iDisplayInfoMask;

/// The bit mask identifies the display status. \ref define_displayinfomask
    int  iDisplayInfoValue;
} ADLDisplayInfo, *LPADLDisplayInfo;

/////////////////////////////////////////////////////////////////////////////////////////////
```

这段代码定义了一个名为 ADLDisplayDPMSTInfo 的结构体，用于存储显示端口设备的相关信息。该结构体包含以下成员：

1. 显示 ID（ADLDisplayID）
2. 总带宽可用量（iTotalAvailableBandwidthInMpbs）
3. 分配给该显示的带宽（iAllocatedBandwidthInMbps）
4. 来自 DAL DpMstSinkInfo 的信息（strGlobalUniqueIdentifier）
5. 相对地址信息（rad）
6. 相对地址，地址方案从源端开始（rad）
7. 物理DP端口 ID（iPhysicalConnectorID）

该结构体用于在应用程序中获取和设置显示端口设备的各种信息，以便在用户请求时访问相应的设备驱动程序调用。


```cpp
///\brief Structure containing information about the display port MST device.
///
/// This structure is used to store various MST information about the display port device.  This
/// information can be returned to the user, or used to access various driver calls to
/// fetch various display-device-related settings upon the user's request
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLDisplayDPMSTInfo
{
    /// The ADLDisplayID structure
    ADLDisplayID displayID;

    /// total bandwidth available on the DP connector
    int    iTotalAvailableBandwidthInMpbs;
    /// bandwidth allocated to this display
    int    iAllocatedBandwidthInMbps;

    // info from DAL DpMstSinkInfo
    /// string identifier for the display
    char    strGlobalUniqueIdentifier[ADL_MAX_PATH];

    /// The link count of relative address, rad[0] upto rad[linkCount] are valid
    int        radLinkCount;
    /// The physical connector ID, used to identify the physical DP port
    int        iPhysicalConnectorID;

    /// Relative address, address scheme starts from source side
    char    rad[ADL_MAX_RAD_LINK_COUNT];
} ADLDisplayDPMSTInfo, *LPADLDisplayDPMSTInfo;

```

这段代码定义了一个名为ADLDisplayMode的结构体，用于表示每个控制器使用的显示模式。该结构体包含以下成员：

* iPelsHeight：垂直分辨率（以像素为单位）。
* iPelsWidth：水平分辨率（以像素为单位）。
* iBitsPerPel：每个 pel的比特数。
* iDisplayFrequency：刷新率（以Hz为单位）。

这个结构体可以用来定义用于每个控制器的显示模式，可以根据需要设置不同的成员。


```cpp
/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing the display mode definition used per controller.
///
/// This structure is used to store the display mode definition used per controller.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLDisplayMode
{
/// Vertical resolution (in pixels).
   int  iPelsHeight;
/// Horizontal resolution (in pixels).
   int  iPelsWidth;
/// Color depth.
   int  iBitsPerPel;
/// Refresh rate.
   int  iDisplayFrequency;
} ADLDisplayMode;

```

这是一段C语言代码，定义了一个名为ADLDetailedTiming的结构体。这个结构体包含详细的定时参数，可以用来描述时序执行的标准。

具体来说，这个结构体包含以下几个成员：

1. iSize：结构体的尺寸，占用的字节数。
2. sTimingFlags：时序标志，用于指定是否启用时序执行，例如，如果这个结构体用于存储硬件寄存器的时序参数，那么sTimingFlags的值应该为2，以启用它们。
3. sHTotal：时序的总宽度，即在时序执行期间，所有连续执行的指令的宽度之和。
4. sHDisplay：用于显示时序执行宽度的宽度，它的值在2到7之间，具体取决于sHTotal的值。

这个结构体可以被用于存储描述时序执行的标准，例如，在操作系统中，可以使用这个结构体来描述硬件设备（如处理器、内存控制器等）的时序参数。


```cpp
/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing detailed timing parameters.
///
/// This structure is used to store the detailed timing parameters.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLDetailedTiming
{
/// Size of the structure.
     int   iSize;
/// Timing flags. \ref define_detailed_timing_flags
     short sTimingFlags;
/// Total width (columns).
     short sHTotal;
/// Displayed width.
     short sHDisplay;
```

这段代码定义了几个短类型变量，用于表示水平同步信号的偏移量、宽度、总高度、显示高度、垂直同步信号的偏移量、同步启动时间和同步宽度、像素时钟值以及扫描方向控制。这些变量将在游戏中的像素渲染过程中使用，对于水平或垂直同步信号的计算和设置具有重要影响。


```cpp
/// Horizontal sync signal offset.
     short sHSyncStart;
/// Horizontal sync signal width.
     short sHSyncWidth;
/// Total height (rows).
     short sVTotal;
/// Displayed height.
     short sVDisplay;
/// Vertical sync signal offset.
     short sVSyncStart;
/// Vertical sync signal width.
     short sVSyncWidth;
/// Pixel clock value.
     short sPixelClock;
/// Overscan right.
     short sHOverscanRight;
```

这是一个定义了一个名为ADLDetailedTiming的结构体的头文件，它包含了一些与游戏中的时钟相关的计时器。

具体来说，这个结构体包含以下成员：

1. sHOverscanLeft：低8位（0-255）的计数器，用于记录玩家在屏幕上超过8秒（或超过30帧）的次数。这个计数器是负数，表示屏幕上没有超过8秒的玩家次数。
2. sVOverscanBottom：与sHOverscanLeft类似的计数器，用于记录玩家在屏幕上下载超过255帧（或超过50帧）的次数。这个计数器也是负数，表示屏幕上下载没有超过255帧的玩家次数。
3. sVOverscanTop：与sHOverscanBottom类似的计数器，用于记录玩家在屏幕上垂直超过255帧（或超过50帧）的次数。这个计数器也是负数，表示屏幕上没有超过255帧且垂直没有超过50帧的玩家次数。
4. sOverscan8B：用于记录玩家在屏幕上超过8B（8位二进制计数器）的次数。这个计数器是负数，表示屏幕上没有超过8B的玩家次数。
5. sOverscanGR：用于记录玩家在屏幕上超过GR（高位二进制计数器）的次数。这个计数器是负数，表示屏幕上没有超过GR的玩家次数。

这个结构体在游戏开发中可能会被用来记录游戏中的时钟信息，例如帧率、垂直和水平的超过计数器等。


```cpp
/// Overscan left.
     short sHOverscanLeft;
/// Overscan bottom.
     short sVOverscanBottom;
/// Overscan top.
     short sVOverscanTop;
     short sOverscan8B;
     short sOverscanGR;
} ADLDetailedTiming;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing display mode information.
///
/// This structure is used to store the display mode information.
/// \nosubgrouping
```

这段代码定义了一个名为ADLDisplayModeInfo的结构体，表示动画 domains（ADL）的显示模式信息。这个结构体包含了以下几个成员：

1. iTimingStandard：当前显示模式的定时标准，包括基准时钟（refresh rate）和刷新频率因子（possible standard）。
2. iPossibleStandard：当前显示模式可能的定时标准。
3. iRefreshRate：刷新率因子，即显示模式每秒刷新的像素数。
4. iPelsWidth：每行像素的数量。
5. iPelsHeight：每列像素的数量。
6. sDetailedTiming：包含更详细的时间管理器的成员。

这个结构体用于在应用程序中存储和操作ADL的显示模式信息，包括定时标准、可能的定时标准和刷新率等。


```cpp
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLDisplayModeInfo
{
/// Timing standard of the current mode. \ref define_modetiming_standard
  int  iTimingStandard;
/// Applicable timing standards for the current mode.
  int  iPossibleStandard;
/// Refresh rate factor.
  int  iRefreshRate;
/// Num of pixels in a row.
  int  iPelsWidth;
/// Num of pixels in a column.
  int  iPelsHeight;
/// Detailed timing parameters.
  ADLDetailedTiming  sDetailedTiming;
} ADLDisplayModeInfo;

```

这段代码定义了一个名为ADLDisplayProperty的结构体，用于存储当前适配器的显示属性。该结构体包含了四个成员变量：iSize、iPropertyType、iExpansionMode和iSupport。其中，iSize表示结构体的大小，iPropertyType用于指定显示属性的类型，iExpansionMode用于指定是否支持显示属性，iSupport用于记录当前显示属性是否被支持。这个结构体可以被用来设置、获取或修改当前适配器的显示属性。


```cpp
/////////////////////////////////////////////////////////////////////////////////////////////
/// \brief Structure containing information about display property.
///
/// This structure is used to store the display property for the current adapter.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLDisplayProperty
{
/// Must be set to sizeof the structure
  int iSize;
/// Must be set to \ref ADL_DL_DISPLAYPROPERTY_TYPE_EXPANSIONMODE or \ref ADL_DL_DISPLAYPROPERTY_TYPE_USEUNDERSCANSCALING
  int iPropertyType;
/// Get or Set \ref ADL_DL_DISPLAYPROPERTY_EXPANSIONMODE_CENTER or \ref ADL_DL_DISPLAYPROPERTY_EXPANSIONMODE_FULLSCREEN or \ref ADL_DL_DISPLAYPROPERTY_EXPANSIONMODE_ASPECTRATIO or \ref ADL_DL_DISPLAYPROPERTY_TYPE_ITCFLAGENABLE
  int iExpansionMode;
/// Display Property supported? 1: Supported, 0: Not supported
  int iSupport;
```

这段代码定义了一个名为ADLClockInfo的结构体，用于存储与当前适配器（例如CPU和内存）的时钟信息。具体来说，这个结构体包含两个整型成员iCurrent和iDefault，分别用于存储当前的时钟值和默认值。

接下来，定义了一个名为ADLDisplayProperty的接口，该接口声明了一个名为DisplayProperty的函数，用于显示当前值和默认值。这个函数没有定义任何参数，因此它的作用是接收一个ADLClockInfo类型的参数，然后显示与该参数相关的信息。

最后，定义了一个名为ADLDisplayProperty的类，该类实现了ADLDisplayProperty接口，包含了一个名为DisplayProperty的函数，它的作用与ADLDisplayProperty接口中的函数一样，即接收一个ADLClockInfo类型的参数，并显示与该参数相关的信息。


```cpp
/// Display Property current value
  int iCurrent;
/// Display Property Default value
  int iDefault;
} ADLDisplayProperty;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about Clock.
///
/// This structure is used to store the clock information for the current adapter
/// such as core clock and memory clock info.
///\nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLClockInfo
{
```

这段代码定义了一个名为ADLI2C的结构体，用于存储I2C相关信息。这个结构体包含两个整型成员iCoreClock和iMemoryClock，分别表示I2C主控器和从控器的时钟频率。同时，这个结构体还包含一个指针LPADLClockInfo，用于存储I2C从设备的相关信息。

ADLClockInfo和LPADLClockInfo是两个指针类型，ADLClockInfo是指向ADLClockInfo结构体的指针，LPADLClockInfo是指向LPADLClockInfo结构体的指针。

接下来，该代码会定义一个名为CoreClock和MemoryClock的整型变量，分别表示系统中Core和Memory模块的时钟频率，并且将这两个变量的值初始化为10000。

最后，该代码会定义一个名为ADLI2C的函数，用于存储I2C相关信息。这个函数的实现中，通过判断从设备是否连接到I2C总线上，来设置iMemoryClock的值，并且通过iCoreClock和iMemoryClock来设置ADLClockInfo结构体中相关成员的值。


```cpp
/// Core clock in 10 KHz.
    int iCoreClock;
/// Memory clock in 10 KHz.
    int iMemoryClock;
} ADLClockInfo, *LPADLClockInfo;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about I2C.
///
/// This structure is used to store the I2C information for the current adapter.
/// This structure is used by the ADL_Display_WriteAndReadI2C() function.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLI2C
{
```

以上代码定义了一个名为ADLI2C的结构体，表示I2C slave设备。以下是结构体定义的一些关键成员的说明：

1. iSize：I2C slave设备的尺寸，这是一个整数类型的变量，但具体的数据类型并未定义。
2. iLine：I2C slave设备的通信线，这是一个整数类型的变量，但具体的数据类型并未定义。
3. iAddress：7-bit I2C slave设备地址，这是一个整数类型的变量，但具体的数据类型并未定义。
4. iOffset：I2C slave设备从设备地址中偏移的 data 的大小，这是一个整数类型的变量，但具体的数据类型并未定义。
5. iAction：I2C slave device 操作的动作，可以是ADL_DL_I2C_ACTIONREAD、ADL_DL_I2C_ACTIONWRITE或ADL_DL_I2C_ACTIONREAD_REPEATEDSTART之一，这是一个整数类型的变量，但具体的数据类型并未定义。
6. iSpeed：I2C clock speed in KHz，这是一个整数类型的变量，但具体的数据类型并未定义。
7. iDataSize：I2C bus 接收或发送的数据字节数，这是一个整数类型的变量，但具体的数据类型并未定义。
8. pcData：指向在 I2C bus 上发送或接收数据的字符数组的指针，这是一个字符类型变量，但具体的数据类型并未定义。

除了以上关键成员之外，上述代码还可以被视为定义了 ADLI2C 结构体，该结构体用于表示 I2C slave device 的信息。


```cpp
/// Size of the structure
    int iSize;
/// Numerical value representing hardware I2C.
    int iLine;
/// The 7-bit I2C slave device address, shifted one bit to the left.
    int iAddress;
/// The offset of the data from the address.
    int iOffset;
/// Read from or write to slave device. \ref ADL_DL_I2C_ACTIONREAD or \ref ADL_DL_I2C_ACTIONWRITE or \ref ADL_DL_I2C_ACTIONREAD_REPEATEDSTART
    int iAction;
/// I2C clock speed in KHz.
    int iSpeed;
/// A numerical value representing the number of bytes to be sent or received on the I2C bus.
    int iDataSize;
/// Address of the characters which are to be sent or received on the I2C bus.
    char *pcData;
} ADLI2C;

```

这是一段C语言代码，定义了一个名为“ADLDisplayEDIDData”的结构体。这个结构体包含了一些关于EDID（Extended Descriptor Interface）数据的成员。

具体来说，这个结构体包含以下成员：

1. 成员iSize表示结构体的大小。
2. 成员iFlag是一个布尔值，用于指示结构体是否可见。如果没有这个成员，那么结构体将不可见。
3. 成员iEDIDSize表示用于存储EDID数据的结构体的大小。

这个结构体是用于存储和操作EDID数据的。它被用于ADL_Display_EdidData_Get()和ADL_Display_EdidData_Set()函数。在ADL_Display_EdidData_Get()函数中，这个结构体被用来获取当前适配器可用的最大EDID数据长度。在ADL_Display_EdidData_Set()函数中，这个结构体被用来设置适配器的EDID数据，以最小化可能出现的EDID头部错误。


```cpp
/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about EDID data.
///
/// This structure is used to store the information about EDID data for the adapter.
/// This structure is used by the ADL_Display_EdidData_Get() and ADL_Display_EdidData_Set() functions.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLDisplayEDIDData
{
/// Size of the structure
  int iSize;
/// Set to 0
  int iFlag;
  /// Size of cEDIDData. Set by ADL_Display_EdidData_Get() upon return
  int iEDIDSize;
```

这段代码定义了一个名为ADLDisplayEDIDData的结构体，用于存储控制器过山调整器的输入信息。该结构体包含以下成员：

1. iBlockIndex：表示控制器过山调整器中的数据块。
2. cEDIDData：一个字符数组，用于存储输入数据。
3. iReserved：一个包含4个整数的数组，用于保留空间。

接下来，该结构体被用于以下几个函数：

- ADL_Display_ControllerOverlayAdjustmentCaps_Get：从该函数来看，这个结构体用于存储控制器过山调整器的输入数据，并返回一个名为“c”的指针，该指针指向一个存储在iBlockIndex中的整数。
- ADL_Display_ControllerOverlayAdjustmentData_Get：从该函数来看，这个结构体用于存储控制器过山调整器的输入数据，并返回一个名为“c”的指针，该指针指向一个存储在iReserved[0]中的整数。
- ADL_Display_ControllerOverlayAdjustmentData_Set：从该函数来看，这个结构体用于存储控制器过山调整器的输入数据，并接受一个名为“c”的指针，该指针指向一个存储在iReserved[0]中的整数。


```cpp
/// 0, 1 or 2. If set to 3 or above an error ADL_ERR_INVALID_PARAM is generated
  int iBlockIndex;
/// EDID data
  char cEDIDData[ADL_MAX_EDIDDATA_SIZE];
/// Reserved
  int iReserved[4];
}ADLDisplayEDIDData;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about input of controller overlay adjustment.
///
/// This structure is used to store the information about input of controller overlay adjustment for the adapter.
/// This structure is used by the ADL_Display_ControllerOverlayAdjustmentCaps_Get, ADL_Display_ControllerOverlayAdjustmentData_Get, and
/// ADL_Display_ControllerOverlayAdjustmentData_Set() functions.
/// \nosubgrouping
```

这段代码定义了一个名为ADLControllerOverlayInput的结构体，用于表示VROverlay的调整信息。该结构体包含以下成员：

1. iSize：该成员表示ADLControllerOverlayInput结构体的大小，对于每个结构体，需要将这个大小乘以2，因为每个元素都乘以2。
2. iOverlayAdjust：该成员表示用于控制VROverlay的偏移量，它应该被设置为结构体的大小。
3. iValue：该成员表示VROverlay的亮度值，范围为0-255。
4. iReserved：该成员是一个保留字段，保留下来可能用于未来的扩展。

ADLControllerOverlayInput结构体描述了一个用于控制VROverlay的参数，包括偏移量、亮度值等。这些参数可以用于更新VROverlay的效果，从而实现更好的用户体验。


```cpp
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLControllerOverlayInput
{
/// Should be set to the sizeof the structure
  int  iSize;
///\ref ADL_DL_CONTROLLER_OVERLAY_ALPHA or \ref ADL_DL_CONTROLLER_OVERLAY_ALPHAPERPIX
  int  iOverlayAdjust;
/// Data.
  int  iValue;
/// Should be 0.
  int  iReserved;
} ADLControllerOverlayInput;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about overlay adjustment.
```

这段代码定义了一个名为ADLAdjustmentinfo的结构体，用于存储外设适配器的调整信息。该结构体包含四个成员变量：iDefault、iMin、iMax和iStep，分别表示默认值、最小值、最大值和步长。

该结构体被用于ADLControllerOverlayInfo函数，可能用于控制其他函数的输出。

该代码可能被用于操作系统、嵌入式系统或移动设备等领域的开发，与外设调整和控制相关。


```cpp
///
/// This structure is used to store the information about overlay adjustment for the adapter.
/// This structure is used by the ADLControllerOverlayInfo() function.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLAdjustmentinfo
{
/// Default value
  int iDefault;
/// Minimum value
  int iMin;
/// Maximum Value
  int iMax;
/// Step value
  int iStep;
} ADLAdjustmentinfo;

```

这段代码定义了一个名为ADLControllerOverlayInfo的结构体，用于存储控制器过场信息。该结构体包含以下成员：

1. iSize：成员变量，表示该结构体的大小，这里设置为24。
2. sOverlayInfo：成员变量，指向一个名为ADLAdjustmentinfo的结构体，用于存储过场信息。
3. iReserved：成员变量，是一个32位的整数区域，现在被赋值为0。

具体来说，该结构体用于存储ADL显示控制器（controller）的过场信息。这些信息包括控制器的一些调整（adjustment）信息，以及与调整相关的控制（control）信息。过场信息可以帮助调整和控制这些信息，以便正确显示控制器的状态。


```cpp
/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about controller overlay information.
///
/// This structure is used to store information about controller overlay info for the adapter.
/// This structure is used by the ADL_Display_ControllerOverlayAdjustmentCaps_Get() function.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLControllerOverlayInfo
{
/// Should be set to the sizeof the structure
  int                    iSize;
/// Data.
  ADLAdjustmentinfo        sOverlayInfo;
/// Should be 0.
  int                    iReserved[3];
} ADLControllerOverlayInfo;

```

这段代码定义了一个名为ADLGLSyncModuleID的结构体，用于包含GL-Sync模块的信息。

ADLGLSyncModuleID包含三个成员变量：

1. iModuleID：一个唯一的GL-Sync模块ID；
2. iGlSyncGPUPort：GL-Sync GPU port索引，用于传递给ADLGLSyncGenlockConfig.lSignalSource和ADLGlSyncPortControl.lSignalSource；
3. iFWBootSectorVersion：GL-Sync模块固件版本，用于传递给ADLGLSyncBootSectorConfig.lBootSectorUser自定义函数。

ADLGLSyncModuleID的作用是用于在代码中引用GL-Sync模块的信息，以便在编写与GL-Sync相关的函数时进行初始化和配置。


```cpp
/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing GL-Sync module information.
///
/// This structure is used to retrieve GL-Sync module information for
/// Workstation Framelock/Genlock.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLGLSyncModuleID
{
/// Unique GL-Sync module ID.
    int        iModuleID;
/// GL-Sync GPU port index (to be passed into ADLGLSyncGenlockConfig.lSignalSource and ADLGlSyncPortControl.lSignalSource).
    int        iGlSyncGPUPort;
/// GL-Sync module firmware version of Boot Sector.
    int        iFWBootSectorVersion;
```

这段代码定义了一个名为ADLGLSyncModuleID的结构体，该结构体用于表示GL-Sync模块的固件版本。其中，iFWUserSectorVersion成员变量用于存储与GL-Sync模块相关的用户区域固件版本号。

ADLGLSyncModuleID结构体的定义为：
```cppc
ADLGLSyncModuleID : public std::bitset<32>
{
   int iFWUserSectorVersion;             // 固件版本号
};
```

ADLGLSyncModuleID结构体的成员变量包括：

1. iFWUserSectorVersion：一个32位的整数，用于存储与GL-Sync模块相关的用户区域固件版本号。该成员变量存储在ADLGLSyncModuleID结构体的定义中。
2. iPortType：一个32位的整数，用于表示GL-Sync模块的端口类型。该成员变量存储在ADLGLSyncModuleID结构体的定义中。
```cpparduino
   int        iPortType;             // 端口类型
   ///_authline______
   int        iChannelCount;            // 通道数量
   int        iActivity;               // 活动状态
   bool        bBlocked;              // 是否被屏蔽
   bool        bEnabled;              // 是否启用
   int        iComment;            // 注释
   int        iConfig;              // 配置
   int        iCounted;            // 计数
   int        iCount;              // 计数器
   bool        bDetected;            // 是否已检测到信号
   int        iDem比你；            // 工作模式
   int        iDemCount;            // 检测计数
   int        iEnabledCoreClock;    // 使能核心时钟的频率
   int        iEqualize;            // 是否同步
   int        iEquality;            // 是否相等
   int        iInactive;            // 是否处于激活状态
   int        iInactivity;            // 进入非活动状态的计数
   int        iPriority;            // 优先级
   int        iPort向量；            // 端口向量
   int        iSearchSpace high;   // Search空间（高字节）
   int        iSearchSpace low;   // Search空间（低字节）
   int        iSink;              // Sink端口
   int        iSinkIdx;           // Sink端口的索引
   int        iSpeeds;            // 速度
   int        iSpeedsLow;          // 低速度
   int        iSpeedsHigh;          // 高速度
   int        iSYNC;              // SYNC信号
   int        iT迪姆板；            // 定时器
   int        iTInterval;          // 定时器中断时间
   int        iTRequest;            // 定时器请求时间
   int        iUART0;            // UART0
   int        iUART0Xfer;        // UART0的传输缓冲区
   int        iUART0校验和；        // UART0的校验和
   int        iUSB;              // USB
   int        iUSB免税；          // USB免税
```

因此，ADLGLSyncModuleID结构体用于表示GL-Sync模块的固件版本和与之相关的用户区域固件版本号。


```cpp
/// GL-Sync module firmware version of User Sector.
    int        iFWUserSectorVersion;
} ADLGLSyncModuleID , *LPADLGLSyncModuleID;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing GL-Sync ports capabilities.
///
/// This structure is used to retrieve hardware capabilities for the ports of the GL-Sync module
/// for Workstation Framelock/Genlock (such as port type and number of associated LEDs).
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLGLSyncPortCaps
{
/// Port type. Bitfield of ADL_GLSYNC_PORTTYPE_*  \ref define_glsync
    int        iPortType;
```

这段代码定义了一个名为ADLGLSyncPortCaps的结构体，该结构体用于表示GL-Sync模块中与该端口相关的LED数量。

```cppc```
首先定义了一个名为ADLGLSyncPortCaps的结构体，其中包含一个名为iNumOfLEDs的整型变量。
```cpppython
int        iNumOfLEDs;
```
接着定义了一个名为ADLGLSyncPortCaps的指针变量，该指针变量名为LPADLGLSyncPortCaps。
```cppscss
ADLGLSyncPortCaps *LPADLGLSyncPortCaps;
```
最后，定义了一个名为ADLGLSyncGenlockConfig的结构体，其中包含用于表示GL-Sync Genlock设置的所有字段。
```cppjava
typedef struct ADLGLSyncGenlockConfig
{
   int        iValidMask;
} ADLGLSyncGenlockConfig;
```
此结构体定义了一个名为iValidMask的整型变量，用于指定在Genlock设置中有效的字段数量。
```cppjava
int        iValidMask;
```
请注意，该代码中没有为该结构体定义任何函数或成员变量的定义，因此其具体的实现将取决于使用场景。


```cpp
/// Number of LEDs associated for this port.
    int        iNumOfLEDs;
}ADLGLSyncPortCaps, *LPADLGLSyncPortCaps;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing GL-Sync Genlock settings.
///
/// This structure is used to get and set genlock settings for the GPU ports of the GL-Sync module
/// for Workstation Framelock/Genlock.\n
/// \see define_glsync
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLGLSyncGenlockConfig
{
/// Specifies what fields in this structure are valid \ref define_glsync
    int        iValidMask;
```

这段代码定义了一个名为ADLGLSyncGenlockConfig的结构体，用于生成同步信号。以下是该结构体中各个成员变量的说明：

* iSyncDelay：同步信号延迟（毫秒）。
* iFramelockCntlVector：用于控制同步信号的帧级锁定计数器。其中，ADL_GLSYNC_FRAMELOCKCNTL_和__EXTRACT_FRAMELOCK_宏定义了该结构体。
* iSignalSource：同步信号的来源。可以是GL_Sync GPU端口索引或ADL_GLSYNC_SIGNALSOURCE_。
* iSampleRate：采样率，用于在同步信号中进行采样。
* iSyncField：用于控制是否进行同步采样。只有ADL_GLSYNC_SYNCFIELD_1和__EXTRACT_SYNCFIELD_宏定义了该结构体。
* iTriggerEdge：同步信号触发边界的值。ADL_GLSYNC_TRIGGEREDGE_和__EXTRACT_TRIGGEREDGE_宏定义了该结构体。
* iScanRateCoeff：扫描率乘数，用于在同步信号中进行采样。ADL_GLSYNC_SCANRATECOEFF_和__EXTRACT_SCANRATECOEFF_宏定义了该结构体。


```cpp
/// Delay (ms) generating a sync signal.
    int        iSyncDelay;
/// Vector of framelock control bits. Bitfield of ADL_GLSYNC_FRAMELOCKCNTL_* \ref define_glsync
    int        iFramelockCntlVector;
/// Source of the sync signal. Either GL_Sync GPU Port index or ADL_GLSYNC_SIGNALSOURCE_* \ref define_glsync
    int        iSignalSource;
/// Use sampled sync signal. A value of 0 specifies no sampling.
    int        iSampleRate;
/// For interlaced sync signals, the value can be ADL_GLSYNC_SYNCFIELD_1 or *_BOTH \ref define_glsync
    int        iSyncField;
/// The signal edge that should trigger synchronization. ADL_GLSYNC_TRIGGEREDGE_* \ref define_glsync
    int        iTriggerEdge;
/// Scan rate multiplier applied to the sync signal. ADL_GLSYNC_SCANRATECOEFF_* \ref define_glsync
    int        iScanRateCoeff;
}ADLGLSyncGenlockConfig, *LPADLGLSyncGenlockConfig;

```

这是一段C语言代码，定义了一个名为ADLGlSyncPortInfo的结构体，用于获取工作站框架lock（ADL_GLSYNC_PORT_*）中GL-Sync端口（BNC或RJ45）的状态信息。

具体来说，这段代码实现了以下功能：

1. 定义了一个名为ADLGlSyncPortInfo的结构体，其中包含以下成员：
 - iPortType：GL-Sync端口的类型（ADL_GLSYNC_PORT_*）。
 - iNumOfLEDs：这个端口包含的LED数量，也可以通过ADLGLSyncPortCaps成员获取。
 - iPortState：GL-Sync端口的状态，可以通过ADL_GLSYNC_PORTSTATE_*成员获取。

2. 通过引入头文件"ADLGlSyncPort.h"，实现了与ADL-GLSYNC_PORT_实现对齐。

3. 通过定义结构体ADLGlSyncPortInfo，实现了对GL-Sync端口信息的获取和存储。


```cpp
/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing GL-Sync port information.
///
/// This structure is used to get status of the GL-Sync ports (BNC or RJ45s)
/// for Workstation Framelock/Genlock.
/// \see define_glsync
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLGlSyncPortInfo
{
/// Type of GL-Sync port (ADL_GLSYNC_PORT_*).
    int        iPortType;
/// The number of LEDs for this port. It's also filled within ADLGLSyncPortCaps.
    int        iNumOfLEDs;
/// Port state ADL_GLSYNC_PORTSTATE_*  \ref define_glsync
    int        iPortState;
```

这段代码定义了一个名为ADLGlSyncPortInfo的结构体，用于表示GL-Sync（仅限RJ45）端口的控制设置。这个结构体包含三个整型成员：iFrequency、iSignalType和iSignalSource。

iFrequency成员表示端口的垂直刷新率（以毫秒为单位的垂直刷新率），这里定义了一个值为60000，表示60Hz的刷新率。

iSignalType成员用于表示GL-Sync信号类型，这里定义了一个值为GL_Sync GPU Port指数，或者ADL_GLSYNC_SIGNALSOURCE_义的指针。

iSignalSource成员用于表示GL-Sync信号的来源，这里可以是GL_Sync GPU Port索引，或者ADL_GLSYNC_SIGNALSOURCE_义的指针。


```cpp
/// Scanned frequency for this port (vertical refresh rate in milliHz; 60000 means 60 Hz).
    int        iFrequency;
/// Used for ADL_GLSYNC_PORT_BNC. It is ADL_GLSYNC_SIGNALTYPE_*   \ref define_glsync
    int        iSignalType;
/// Used for ADL_GLSYNC_PORT_RJ45PORT*. It is GL_Sync GPU Port index or ADL_GLSYNC_SIGNALSOURCE_*.  \ref define_glsync
    int        iSignalSource;

} ADLGlSyncPortInfo, *LPADLGlSyncPortInfo;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing GL-Sync port control settings.
///
/// This structure is used to configure the GL-Sync ports (RJ45s only)
/// for Workstation Framelock/Genlock.
/// \see define_glsync
```

这段代码定义了一个名为 ADLGlSyncPortControl 的结构体，用于表示 ADL GLSync 控制器的相关信息。该结构体包含以下成员：

* iPortType：表示要控制的 ADL_GLSYNC_PORT_RJ45PORT1 或 ADL_GLSYNC_PORT_RJ45PORT2 的端口类型。
* iControlVector：表示端口控制数据，包括一系列用于控制端口操作的寄存器或引脚编号。
* iSignalSource：表示同步信号的来源，可以是 GL_Sync GPU 端口的索引，也可以是 ADL_GLSYNC_SIGNALSOURCE_*。

ADLGlSyncPortControl 结构体用于定义一个 ADL GLSync 控制器，该控制器可以用于控制图形设备的状态，如开启、关闭、设置其工作状态等。通过这个结构体，开发人员可以使用 GLFW（GNU 并行窗口系统）等库函数来与 GL 同步模式进行交互。


```cpp
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLGlSyncPortControl
{
/// Port to control ADL_GLSYNC_PORT_RJ45PORT1 or ADL_GLSYNC_PORT_RJ45PORT2   \ref define_glsync
    int        iPortType;
/// Port control data ADL_GLSYNC_PORTCNTL_*   \ref define_glsync
    int        iControlVector;
/// Source of the sync signal. Either GL_Sync GPU Port index or ADL_GLSYNC_SIGNALSOURCE_*   \ref define_glsync
    int        iSignalSource;
} ADLGlSyncPortControl;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing GL-Sync mode of a display.
///
```

这段代码定义了一个名为ADLGlSyncMode的结构体，用于获取和设置连接到GL-Sync模块的显示的同步模式设置。它包括一个控制矢量（ADL_GLSYNC_MODECNTL_结构的指针）和一个状态矢量（ADL_GLSYNC_MODECNTL_结构的指针），用于表示同步模式和状态。此外，它还包含一个指向GL-Sync连接器的索引的指针。

ADLGlSyncMode结构体可以被用来获取和设置ADL_GLSYNC_MODECNTL_结构中定义的同步模式和状态的值。同步模式有0到31，分别表示默认模式、工作模式、S0PR模式和S1PR模式。同步状态有0到15，分别表示同步或异步。GL-Sync模块可以用来控制由Workstation Framework引起的FNL和Genlock displays的同步。


```cpp
/// This structure is used to get and set GL-Sync mode settings for a display connected to
/// an adapter attached to a GL-Sync module for Workstation Framelock/Genlock.
/// \see define_glsync
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLGlSyncMode
{
/// Mode control vector. Bitfield of ADL_GLSYNC_MODECNTL_*   \ref define_glsync
    int        iControlVector;
/// Mode status vector. Bitfield of ADL_GLSYNC_MODECNTL_STATUS_*   \ref define_glsync
    int        iStatusVector;
/// Index of GL-Sync connector used to genlock the display/controller.
    int        iGLSyncConnectorIndex;
} ADLGlSyncMode, *LPADLGlSyncMode;

```

这是一段定义了一个名为ADLGlSyncMode2的结构体的C代码。这个结构体包含了一个用于控制GL-Sync模式设置的指针和一个用于获取或设置GL-Sync模式设置的状态指针。

ADLGlSyncMode2结构体定义了三个成员变量：iControlVector，iStatusVector和iGLSyncConnectorIndex。其中，iControlVector表示控制模式，iStatusVector表示状态模式，iGLSyncConnectorIndex表示用于 genlock 的 GL-Sync 连接器的索引号。

这个结构体可以用于在应用程序中获取和设置ADL-Sync模式设置，以便在Workstation Framelock和Genlock上连接的显示设备。可以使用函数define_glsync和get_glsync_mode_ex来获取和设置ADL-Sync模式设置。


```cpp
/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing GL-Sync mode of a display.
///
/// This structure is used to get and set GL-Sync mode settings for a display connected to
/// an adapter attached to a GL-Sync module for Workstation Framelock/Genlock.
/// \see define_glsync
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLGlSyncMode2
{
/// Mode control vector. Bitfield of ADL_GLSYNC_MODECNTL_*   \ref define_glsync
    int        iControlVector;
/// Mode status vector. Bitfield of ADL_GLSYNC_MODECNTL_STATUS_*   \ref define_glsync
    int        iStatusVector;
/// Index of GL-Sync connector used to genlock the display/controller.
    int        iGLSyncConnectorIndex;
```

这段代码定义了一个名为ADLInfoPacket的结构体，用于存储显示的数据包信息。这个结构体包含三个字节，分别记录了显示的起始行、结束行和行数，以及是否需要刷新显示。

在接下来的代码中，定义了一个名为ADLGlSyncMode2和LPADLGlSyncMode2的指针类型，这些指针类型用于存储ADLGlSyncMode2结构体。然后，代码定义了一个名为iDisplayIndex的整型变量，用于存储显示的目标索引。

接下来的代码是一个名为ADLDisplayDataPacket的函数，这个函数接收一个指向ADLInfoPacket的指针，然后将其解析为DisplayDataPacket结构体，最后将其赋值给iDisplayIndex。

这段代码的作用是定义了一个用于获取和设置显示数据包信息的ADLInfoPacket结构体，以及一个用于获取和设置ADLGlSyncMode2类型的指针，最终将ADLInfoPacket结构体中的数据传递给iDisplayIndex，以获取或设置显示的状态。


```cpp
/// Index of the display to which this GLSync applies to.
    int        iDisplayIndex;
} ADLGlSyncMode2, *LPADLGlSyncMode2;


/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing the packet info of a display.
///
/// This structure is used to get and set the packet information of a display.
/// This structure is used by ADLDisplayDataPacket.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct  ADLInfoPacket
{
    char hb0;
    char hb1;
    char hb2;
```

这段代码定义了一个名为ADLInfoPacket的结构体，用于存储AVI（Advanced Video Interface，高级视频接口）数据包的info。这个结构体包含两个字节，其中第一个字节是一个名为bPB3_ITC的位，表示是否传送ITC（Integrated Test Case，集成测试案例）数据包；第二个字节是一个未知的字段，用于存放其他的info。

接下来的两行代码定义了一个名为sb的28字节字符数组，用于存储ADLInfoPacket结构体。这里并没有对sb进行初始化，所以它实际上是一个空字符数组。

最后，该代码定义了一个名为ADLInfoPacket的函数，接受一个ADLInfoPacket结构的参数。这个函数的作用是读取并设置ADLInfoPacket结构体中的bPB3_ITC字段，然后执行其他操作，比如生成或接收AVI数据包等。但具体执行哪些操作，该函数没有定义。


```cpp
/// sb0~sb27
    char sb[28];
}ADLInfoPacket;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing the AVI packet info of a display.
///
/// This structure is used to get and set AVI the packet info of a display.
/// This structure is used by ADLDisplayDataPacket.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLAVIInfoPacket  //Valid user defined data/
{
/// byte 3, bit 7
   char bPB3_ITC;
```

这段代码定义了一个名为ADLODClockSetting的结构体，用于表示Overdrive clock setting。这个结构体包含一个名为bPB5的char型成员和一个未被定义的char型成员，用于存储设置。

Overdrive clock setting是指Overdrive可以根据用户驾驶员的驾驶习惯和车辆特性进行个性化的定时，以提供更准确的辅助驾驶体验。这个结构体用于存储这个设置。

接下来的代码定义了一个名为ADLAVIInfoPacket的类，这个类也被用于存储Overdrive clock setting。但是，这个类的成员是公开的，因此可以访问它的成员变量。


```cpp
/// byte 5, bit [7:4].
   char bPB5;
}ADLAVIInfoPacket;

// Overdrive clock setting structure definition.

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing the Overdrive clock setting.
///
/// This structure is used to get the Overdrive clock setting.
/// This structure is used by ADLAdapterODClockInfo.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLODClockSetting
{
```



这段代码定义了一个名为ADLODClockSetting的类，用于控制默认时间设置。这个类包含多个整型变量，分别表示最大、最小、请求的时钟设置，以及设置时需要增加的步长。

代码的主要作用是提供一个可配置的时钟设置，可以根据需要设置不同的时间，从而实现灵活的时间设置。用户可以根据自己的需要，调整这些变量的值，来设置偏好钟设置。


```cpp
/// Deafult clock
    int iDefaultClock;
/// Current clock
    int iCurrentClock;
/// Maximum clcok
    int iMaxClock;
/// Minimum clock
    int iMinClock;
/// Requested clcock
    int iRequestedClock;
/// Step
    int iStepClock;
} ADLODClockSetting;

/////////////////////////////////////////////////////////////////////////////////////////////
```

这是一个定义了一个名为 ADLAdapterODClockInfo 的结构体，它包含了从 Overdrive 获取时钟信息所需的所有成员。这个结构体被用于函数 ADL_Display_ODClockInfo_Get，该函数可以获取时钟信息以进行显示。

具体来说，这个结构体包含以下成员：

1. 成员 iSize：表示该结构体的大小。
2. 成员 iFlags：定义了设置或清除 ADLAdapterODClockInfo 标志的方案。
3. 成员 sMemoryClock：定义了内存时钟的设置，包括启用和时钟频率等。
4. 成员 sEngineClock：定义了发动机时钟的设置，包括启用和时钟频率等。

通过这个结构体，可以方便地获取和设置 Overdrive 钟信息，从而进行更灵活的操作。


```cpp
///\brief Structure containing the Overdrive clock information.
///
/// This structure is used to get the Overdrive clock information.
/// This structure is used by the ADL_Display_ODClockInfo_Get() function.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLAdapterODClockInfo
{
/// Size of the structure
    int iSize;
/// Flag \ref define_clockinfo_flags
    int iFlags;
/// Memory Clock
    ADLODClockSetting sMemoryClock;
/// Engine Clock
    ADLODClockSetting sEngineClock;
} ADLAdapterODClockInfo;

```

这是一个定义了一个名为 ADLAdapterODClockConfig 的结构体的函数。这个结构体包含了一个关于 Overdrive 钟配置的配置。这个结构体是用来设置 Overdrive 钟配置的，也被 ADL_Display_ODClockConfig_Set() 函数使用。

ADLAdapterODClockConfig 结构体定义了三个成员变量：

1. iSize：表示该结构体的大小。
2. iFlags：定义了几个标志，用于指示是否要显示时钟信息。
3. iMemoryClock：表示存储器时钟的 ID。

通过这个结构体，可以设置 Overdrive 钟的配置，包括是否要显示时钟信息和时钟的 ID。


```cpp
/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing the Overdrive clock configuration.
///
/// This structure is used to set the Overdrive clock configuration.
/// This structure is used by the ADL_Display_ODClockConfig_Set() function.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLAdapterODClockConfig
{
/// Size of the structure
  int iSize;
/// Flag \ref define_clockinfo_flags
  int iFlags;
/// Memory Clock
  int iMemoryClock;
```

这段代码定义了一个名为ADLPActivity的结构体，用于存储与当前电源管理相关的活动信息。这个结构体是由ADL_PM_CurrentActivity_Get()函数使用的，用于获取与电源管理相关的活动信息。

ADLPActivity结构体包含一个名为iSize的整数，以及一个用于存储当前电源管理活动信息的数组。这个数组可以用来传递给ADL_PM_CurrentActivity_Get()函数，用于获取与电源管理相关的活动信息。

电源管理相关的活动信息包括用电器的状态、用电器的优先级、以及用电器正在执行的任务等。这些信息可以根据实际需要进行扩展和修改，以适应不同的应用场景。


```cpp
/// Engine Clock
  int iEngineClock;
} ADLAdapterODClockConfig;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about current power management related activity.
///
/// This structure is used to store information about current power management related activity.
/// This structure (Overdrive 5 interfaces) is used by the ADL_PM_CurrentActivity_Get() function.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLPMActivity
{
/// Must be set to the size of the structure
    int iSize;
```

这段代码定义了多个整型变量，用于表示不同的系统时钟和性能指标。下面是每个变量的作用：

1. iEngineClock：当前引擎时钟，用于驱动系统的实时部分。
2. iMemoryClock：当前内存时钟，用于驱动系统的虚拟部分。
3. iVddc：当前可编程模拟器供电电压，用于驱动系统的外设。
4. iActivityPercent：当前GPU利用率，用于表示正在运行的GPU任务占总任务的百分比。
5. iCurrentPerformanceLevel：当前性能级别指数，用于根据用户设置的性能级别调整系统的频率和电压。
6. iCurrentBusSpeed：当前PCIe总线速度，用于驱动系统的外设。
7. iCurrentBusLanes：当前PCIe总线通道数，用于配置系统外设的通道数。
8. iMaximumBusLanes：最大PCIe总线通道数，用于设置系统外设的通道数。


```cpp
/// Current engine clock.
    int iEngineClock;
/// Current memory clock.
    int iMemoryClock;
/// Current core voltage.
    int iVddc;
/// GPU utilization.
    int iActivityPercent;
/// Performance level index.
    int iCurrentPerformanceLevel;
/// Current PCIE bus speed.
    int iCurrentBusSpeed;
/// Number of PCIE bus lanes.
    int iCurrentBusLanes;
/// Maximum number of PCIE bus lanes.
    int iMaximumBusLanes;
```

这段代码定义了一个名为ADLThermalControllerInfo的结构体类型，用于存储有关热管理控制器的信息。这个结构体被用于ADL_PM_ThermalDevices_Enum，用于描述热管理设备。

具体来说，这个结构体包含一个名为iSize的整数，用于描述结构体本身的尺寸。


```cpp
/// Reserved for future purposes.
    int iReserved;
} ADLPMActivity;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about thermal controller.
///
/// This structure is used to store information about thermal controller.
/// This structure is used by ADL_PM_ThermalDevices_Enum.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLThermalControllerInfo
{
/// Must be set to the size of the structure
  int iSize;
```

这是一个C++结构体，名为ADLThermalControllerInfo。它用于存储有关热控制器温度的信息。这个结构体被用于ADL_PM_Temperature_Get()函数。

具体来说，这个结构体包含以下成员：

1. int型成员iThermalDomain，表示热控制器域，可能的值为ADL_DL_THERMAL_DOMAIN_Other或ADL_DL_THERMAL_DOMAIN_GPU，用于指定热控制器要测量的温度域。
2. int型成员iDomainIndex，用于指定热控制器域的唯一索引，用于在系统中查找与给定域相对应的寄存器或驱动程序。
3. int型成员iFlags，表示热控制器的工作标志，可能的值为ADL_DL_THERMAL_FLAG_INTERRUPT或ADL_DL_THERMAL_FLAG_FANCONTROL，用于指示是否允许中断和风扇控制。

这个结构体用于存储ADL_PM_TEMPERATURE_GET()函数需要用到的信息，以便在需要时进行解码和转换。


```cpp
/// Possible valies: \ref ADL_DL_THERMAL_DOMAIN_OTHER or \ref ADL_DL_THERMAL_DOMAIN_GPU.
  int iThermalDomain;
///    GPU 0, 1, etc.
  int iDomainIndex;
/// Possible valies: \ref ADL_DL_THERMAL_FLAG_INTERRUPT or \ref ADL_DL_THERMAL_FLAG_FANCONTROL
  int iFlags;
} ADLThermalControllerInfo;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about thermal controller temperature.
///
/// This structure is used to store information about thermal controller temperature.
/// This structure is used by the ADL_PM_Temperature_Get() function.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
```

这段代码定义了一个名为 ADLTemperature 的结构体，其中包含两个成员变量：iSize 和 iTemperature。

ADLTemperature 结构体是 thermal controller fan speed 的信息结构体，它被用于存储 thermal controller fan speed 的信息。这个结构体被用于函数 ADL_PM_FanSpeedInfo_Get()，可以被用来设置和获取 thermal controller 的 fan speed。


```cpp
typedef struct ADLTemperature
{
/// Must be set to the size of the structure
  int iSize;
/// Temperature in millidegrees Celsius.
  int iTemperature;
} ADLTemperature;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about thermal controller fan speed.
///
/// This structure is used to store information about thermal controller fan speed.
/// This structure is used by the ADL_PM_FanSpeedInfo_Get() function.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
```

这段代码定义了一个名为 ADLFanSpeedInfo 的结构体类型。这个结构体包含了一些与 fan 速度控制相关的参数。

ADLFanSpeedInfo 的定义包括：

1. iSize：表示这个结构体的大小，必须被设置为结构体大小的值。

2. iFlags：定义了一些与 fan 控制相关的标志，比如是否开启某个 flag。

3. iMinPercent：表示一个最小可能的 fan 速度百分比。

4. iMaxPercent：表示一个最大可能的 fan 速度百分比。

5. iMinRPM：表示一个最小可能的 fan 速度 RPM。

6. iMaxRPM：表示一个最大可能的 fan 速度 RPM。

这个结构体定义了 fan 速度控制的相关参数，用于控制 ADL 程序中的 fan 速度。


```cpp
typedef struct ADLFanSpeedInfo
{
/// Must be set to the size of the structure
  int iSize;
/// \ref define_fanctrl
  int iFlags;
/// Minimum possible fan speed value in percents.
  int iMinPercent;
/// Maximum possible fan speed value in percents.
  int iMaxPercent;
/// Minimum possible fan speed value in RPM.
  int iMinRPM;
/// Maximum possible fan speed value in RPM.
  int iMaxRPM;
} ADLFanSpeedInfo;

```

这段代码定义了一个名为ADLFanSpeedValue的结构体，用于存储由热控器报告的扇速信息。该结构体包含三个成员：iSize、iSpeedType和iFanSpeed，分别表示扇速信息的大小、速度类型和扇速值。

该结构体是ADL_Overdrive5_FanSpeed_Get()和ADL_Overdrive5_FanSpeed_Set()函数的参数，用于存储扇速信息。其中，iSize表示扇速信息的大小，iSpeedType表示速度类型，iFanSpeed表示扇速值。通过这两个函数，可以设置或获取扇速信息，从而实现对扇速的控制。


```cpp
/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about fan speed reported by thermal controller.
///
/// This structure is used to store information about fan speed reported by thermal controller.
/// This structure is used by the ADL_Overdrive5_FanSpeed_Get() and ADL_Overdrive5_FanSpeed_Set() functions.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLFanSpeedValue
{
/// Must be set to the size of the structure
  int iSize;
/// Possible valies: \ref ADL_DL_FANCTRL_SPEED_TYPE_PERCENT or \ref ADL_DL_FANCTRL_SPEED_TYPE_RPM
  int iSpeedType;
/// Fan speed value
  int iFanSpeed;
```

这段代码定义了一个名为ADLFanSpeedValue的结构体类型，用于存储ADLOD参数范围的信息。这个结构体包含一个名为iFlags的整型变量，用于表示当前限速标志，以及一个名为iMin的整型变量，表示最小速度参数。

ADLODParameterRange结构体是用于存储最小速度参数的，它包含两个成员变量：iMin和iFlags。其中，iMin成员变量表示最小速度参数，iFlags成员变量表示限速标志。这两个成员变量都是整型变量，可以用作int类型的成员变量。


```cpp
/// The only flag for now is: \ref ADL_DL_FANCTRL_FLAG_USER_DEFINED_SPEED
  int iFlags;
} ADLFanSpeedValue;

////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing the range of Overdrive parameter.
///
/// This structure is used to store information about the range of Overdrive parameter.
/// This structure is used by ADLODParameters.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLODParameterRange
{
/// Minimum parameter value.
  int iMin;
```

这段代码定义了一个名为ADLODParameterRange的结构体，该结构体包含了关于Overdrive参数的信息。这个结构体被用于存储ADL_Overdrive5_ODParameters_Get()函数的输入参数。

具体来说，这个结构体包含两个成员变量：iMax和iStep。iMax成员变量表示Overdrive参数的最大值，iStep成员变量表示Overdrive参数的步长或者变化量。通过这两个成员变量的设置，可以调节Overdrive参数的取值范围。

在函数内部，ADLODParameterRange结构体被用来接收参数，而不是存储它们。因此，这个结构体仅在函数内部使用，而不会在程序中产生任何后代。


```cpp
/// Maximum parameter value.
  int iMax;
/// Parameter step value.
  int iStep;
} ADLODParameterRange;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about Overdrive parameters.
///
/// This structure is used to store information about Overdrive parameters.
/// This structure is used by the ADL_Overdrive5_ODParameters_Get() function.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLODParameters
{
```

这段代码定义了一个名为ADLODParameters的结构体，表示GPU的性能水平参数。这些参数用于测量GPU的性能和活动水平。以下是每个参数的作用：

1. iSize：表示ADLOD结构体的最大尺寸，即结构体可以包含的最大结构体数据。

2. iNumberOfPerformanceLevels：表示ADLOD支持的最大性能级别数量。

3. iActivityReportingSupported：表示GPU是否支持将性能信息报告给操作系统或用户。

4. iDiscretePerformanceLevels：表示ADLOD是否支持离散的性能级别或性能范围。

5. iReserved：保留空间，用于未来的使用。

6. sEngineClock：表示GPU的引擎时钟范围，即GPU可以运行的最大时钟频率。

7. sMemoryClock：表示GPU的内存时钟范围，即GPU可以读取和写入的最大时钟频率。

8. sVddc：表示GPU的电压范围，即GPU所需的最低电压。

这些参数可以用于控制和配置ADLOD的性能和活动水平。例如，要设置ADLOD支持离散的性能级别，可以将其iDiscretePerformanceLevels参数设置为1，然后将其sEngineClock和sMemoryClock参数设置为适当的值。要查看ADLOD的当前性能水平，可以使用GetADLODRequestedPerformanceCount函数从ADLOD结构体中检索有关其活动水平的计数。


```cpp
/// Must be set to the size of the structure
  int iSize;
/// Number of standard performance states.
  int iNumberOfPerformanceLevels;
/// Indicates whether the GPU is capable to measure its activity.
  int iActivityReportingSupported;
/// Indicates whether the GPU supports discrete performance levels or performance range.
  int iDiscretePerformanceLevels;
/// Reserved for future use.
  int iReserved;
/// Engine clock range.
  ADLODParameterRange sEngineClock;
/// Memory clock range.
  ADLODParameterRange sMemoryClock;
/// Core voltage range.
  ADLODParameterRange sVddc;
} ADLODParameters;

```

这段代码定义了一个名为ADLODPerformanceLevel的结构体，它包含以下信息：

* 发动机时钟：iEngineClock
* 内存时钟：iMemoryClock
* 核心电压：iVddc

这个结构体是用来存储汽车ADAS系统中的Overdrive level（驾驶模式）信息的。在Overdrive level的不同阶段，比如IDLE（空闲）、ECON（经济模式）、BOOST（动力模式）和TEAS（换挡模式）下，ADAS系统可能需要不同的性能设置。ADLODPerformanceLevel结构体中包含了存储这些不同阶段所需的核心电压、时钟和电压的值。

这个结构体很可能是从参考文献或其他公开资料中获得的，而不是自己编写的。


```cpp
/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about Overdrive level.
///
/// This structure is used to store information about Overdrive level.
/// This structure is used by ADLODPerformanceLevels.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLODPerformanceLevel
{
/// Engine clock.
  int iEngineClock;
/// Memory clock.
  int iMemoryClock;
/// Core voltage.
  int iVddc;
} ADLODPerformanceLevel;

```

这段代码定义了一个名为ADLODPerformanceLevels的结构体，用于存储Overdrive性能级别信息。该结构体包含一个整型变量iSize，表示该结构体的大小，以及一个整型变量iReserved，用于保留空间。此外，该结构体还包含一个一维数组ADLODPerformanceLevel，该数组用于存储Overdrive性能级别信息。

ADLODPerformanceLevels结构体中包含一个名为aLevels的成员变量，该成员变量是一个ADLODPerformanceLevel数组，用于存储Overdrive性能级别信息。该数组包含一个整型变量i，用于表示每个性能级别的状态，例如正常、高性能、经济等。每个性能级别还可以包含多个其他成员变量，例如PerformanceLevelDirection、PerformanceLevelEvaluationDuration等，用于提供更详细的性能级别信息。

总的来说，该代码定义了一个用于存储Overdrive性能级别信息的结构体，该结构体包含一个用于存储Overdrive性能级别信息的数组，以及一个用于表示Overdrive性能级别信息的整型变量。这些信息可以被用于函数ADL_Overdrive5_ODPerformanceLevels_Get()和ADL_Overdrive5_ODPerformanceLevels_Set()。


```cpp
/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about Overdrive performance levels.
///
/// This structure is used to store information about Overdrive performance levels.
/// This structure is used by the ADL_Overdrive5_ODPerformanceLevels_Get() and ADL_Overdrive5_ODPerformanceLevels_Set() functions.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLODPerformanceLevels
{
/// Must be set to sizeof( \ref ADLODPerformanceLevels ) + sizeof( \ref ADLODPerformanceLevel ) * (ADLODParameters.iNumberOfPerformanceLevels - 1)
  int iSize;
  int iReserved;
/// Array of performance state descriptors. Must have ADLODParameters.iNumberOfPerformanceLevels elements.
  ADLODPerformanceLevel aLevels [1];
} ADLODPerformanceLevels;

```

这是一个定义了一个名为ADLCrossfireComb的结构体的函数。这个结构体包含一个 adapter 数组，其中包含该组合中电缆适配器的数量，以及一个链表，其中包含该组合中每个适配器的索引。

这个结构体是用来存储信息，以便于ADL_Adapter_Crossfire_Caps()，ADL_Adapter_Crossfire_Get()和ADL_Adapter_Crossfire_Set()函数的使用。


```cpp
/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about the proper CrossfireX chains combinations.
///
/// This structure is used to store information about the CrossfireX chains combination for a particular adapter.
/// This structure is used by the ADL_Adapter_Crossfire_Caps(), ADL_Adapter_Crossfire_Get(), and ADL_Adapter_Crossfire_Set() functions.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLCrossfireComb
{
/// Number of adapters in this combination.
  int iNumLinkAdapter;
/// A list of ADL indexes of the linked adapters in this combination.
  int iAdaptLink[3];
} ADLCrossfireComb;

```

这段代码定义了一个名为ADLCrossfireInfo的结构体，用于存储 CrossfireX 状态和错误信息。该结构体被用于 ADL_Adapter_Crossfire_Get() 函数。

具体来说，该结构体包含以下成员：

- iErrorCode：当前错误代码；
- iState：当前状态；
- iSupported：是否支持 CrossfireX。

此外，还有一个名为ADL_TRUE和ADL_FALSE的常量，用于表示 CrossfireX 是否支持该组合。


```cpp
/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing CrossfireX state and error information.
///
/// This structure is used to store state and error information about a particular adapter CrossfireX combination.
/// This structure is used by the ADL_Adapter_Crossfire_Get() function.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLCrossfireInfo
{
/// Current error code of this CrossfireX combination.
  int iErrorCode;
/// Current \ref define_crossfirestate
  int iState;
/// If CrossfireX is supported by this combination. The value is either \ref ADL_TRUE or \ref ADL_FALSE.
  int iSupported;
} ADLCrossfireInfo;

```

这段代码定义了一个名为 ADLBiosInfo 的结构体，用于存储关于芯片集的各种信息。这个结构体包含四个成员变量：strPartNumber、strVersion、strDate，分别用于存储芯片的型号、版本和生产日期。这个结构体是一个 pointer，名为 LPADLBiosInfo，用于指针一个 ADLBiosInfo 类型的变量。

这个代码的作用是定义一个名为 ADLBiosInfo 的结构体，用于存储芯片集的型号、版本和生产日期等信息，并将其作为 void 类型别名 LPADLBiosInfo 类型别名。这个结构体可以被用来存储和操作与芯片集相关的信息。


```cpp
/////////////////////////////////////////////////////////////////////////////////////////////
/// \brief Structure containing information about the BIOS.
///
/// This structure is used to store various information about the Chipset.  This
/// information can be returned to the user.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLBiosInfo
{
    char strPartNumber[ADL_MAX_PATH];    ///< Part number.
    char strVersion[ADL_MAX_PATH];        ///< Version number.
    char strDate[ADL_MAX_PATH];        ///< BIOS date in yyyy/mm/dd hh:mm format.
} ADLBiosInfo, *LPADLBiosInfo;


```

这是一个定义了一个名为ADLAdapterLocation的结构体的定义。这个结构体用于存储有关适配器位置的信息。这个结构体被用于ADLMVPUStatus函数。

具体来说，这个结构体包含三个成员变量：iBus、iDevice和iFunction，分别表示一个8位的PCI总线号、一个5位的设备号和一个3位的函数号。这些成员变量用于表示适配器的位置。


```cpp
/////////////////////////////////////////////////////////////////////////////////////////////
/// \brief Structure containing information about adapter location.
///
/// This structure is used to store information about adapter location.
/// This structure is used by ADLMVPUStatus.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLAdapterLocation
{
/// PCI Bus number : 8 bits
    int iBus;
/// Device number : 5 bits
    int iDevice;
/// Function number : 3 bits
    int iFunction;
} ADLAdapterLocation,ADLBdf;

```

这段代码定义了一个名为ADLVersionsInfo的结构体，用于存储软件版本信息，包括驱动程序版本和催化剂版本，以及一个指向最新安装的Catalyst驱动程序的XML文件的链接。

ADLVersionsInfo类型的变量可以被声明为：
```cpp
ADLVersionsInfo myInstance;
```
或者：
```cpp
ADLVersionsInfo myStaticInstance;
```
该结构体包含三个字符型变量：
```cpp
char strDriverVer[ADL_MAX_PATH];
char strCatalystVersion[ADL_MAX_PATH];
char strCatalystWebLink[ADL_MAX_PATH];
```
每个变量都包含如下内容：
```cpp
strDriverVer = "8.71-100128n-094835E-ATI";
strCatalystVersion = "10.1";
strCatalystWebLink = "http://www.amd.com/us/driverxml";
```
注意，该代码中还包含一个名为ADL_MAX_PATH的常量，用于确保在定义变量时对驱动程序和XML文件的路径进行限制，该常量应该被定义为：
```cpp
#define ADL_MAX_PATH 2048
```
该常量的作用是在定义变量时对驱动程序和XML文件的路径进行限制，确保路径不会过长，从而避免程序崩溃或产生不可预测的行为。


```cpp
/////////////////////////////////////////////////////////////////////////////////////////////
/// \brief Structure containing version information
///
/// This structure is used to store software version information, description of the display device and a web link to the latest installed Catalyst drivers.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLVersionsInfo
{
    /// Driver Release (Packaging) Version (e.g. 8.71-100128n-094835E-ATI)
    char strDriverVer[ADL_MAX_PATH];
    /// Catalyst Version(e.g. "10.1").
    char strCatalystVersion[ADL_MAX_PATH];
    /// Web link to an XML file with information about the latest AMD drivers and locations (e.g. "http://www.amd.com/us/driverxml" )
    char strCatalystWebLink[ADL_MAX_PATH];

} ADLVersionsInfo, *LPADLVersionsInfo;

```

这段代码定义了一个名为ADLVersionsInfoX2的结构体，用于存储软件版本信息、显示设备的描述以及最新的Catalyst驱动程序的网站链接。

ADLVersionsInfoX2中定义了5个字符串类型的成员变量，分别代表了Catalyst和Crimson版本号，以及一个包含最新AMD驱动程序信息XML文件的链接。这些成员变量都使用了最大路径长度，意味着它们可以支持最大2^32个字符的字符串。

这个结构体可以被用于在程序中查找和设置ADL软件的版本信息，例如在安装、卸载或更新Catalyst驱动程序时。


```cpp
/////////////////////////////////////////////////////////////////////////////////////////////
/// \brief Structure containing version information
///
/// This structure is used to store software version information, description of the display device and a web link to the latest installed Catalyst drivers.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLVersionsInfoX2
{
    /// Driver Release (Packaging) Version (e.g. "16.20.1035-160621a-303814C")
    char strDriverVer[ADL_MAX_PATH];
    /// Catalyst Version(e.g. "15.8").
    char strCatalystVersion[ADL_MAX_PATH];
    /// Crimson Version(e.g. "16.6.2").
    char strCrimsonVersion[ADL_MAX_PATH];
    /// Web link to an XML file with information about the latest AMD drivers and locations (e.g. "http://support.amd.com/drivers/xml/driver_09_us.xml" )
    char strCatalystWebLink[ADL_MAX_PATH];

} ADLVersionsInfoX2, *LPADLVersionsInfoX2;

```

这是一段定义了一个名为 ADLMVPUCaps 的结构体的代码。这个结构体用于存储有关 MultiVPU 功能的信息。它被用于函数 ADL_Display_MVPUCaps_Get，该函数负责获取 MultiVPU 功能的信息。

结构体包括三个成员：

1. iSize：必须为 sizeof( ADLMVPUCaps )。这是一个整数类型，用于存储 ADLMVPUCaps 的结构体大小。

2. iAdapterCount：表示连接到 MultiVPU 的视频适配器数量。

3. iPossibleMVPUMasters：用于存储所有可能的 MultiVPU 主人（MVPU Master）的位设置。这个结构体被用于判断 MultiVPU 是否支持主人模式。在主人模式下，多个显卡可以作为 MultiVPU 的主人，从而实现更高的性能。


```cpp
/////////////////////////////////////////////////////////////////////////////////////////////
/// \brief Structure containing information about MultiVPU capabilities.
///
/// This structure is used to store information about MultiVPU capabilities.
/// This structure is used by the ADL_Display_MVPUCaps_Get() function.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLMVPUCaps
{
/// Must be set to sizeof( ADLMVPUCaps ).
  int iSize;
/// Number of adapters.
  int iAdapterCount;
/// Bits set for all possible MVPU masters. \ref MVPU_ADAPTER_0 .. \ref MVPU_ADAPTER_3
  int iPossibleMVPUMasters;
```

这段代码定义了一个名为ADLMVPUCaps的结构体，用于存储所有可能的多媒体虚拟演示环境（MVPU） slave的信息。其中包括：

1. int型变量iPossibleMVPUSlaves，用于存储可能的MVPU slave数量。
2. 字符型变量cAdapterPath，用于存储每个适配器的路径。该结构体数组长度为ADL_DL_MAX_MVPU_ADAPTERS，即MAX_MVPU_ADAPTERS（可能的最大MVPU适配器数量）。

接下来的代码定义了一个名为ADLMVPUStatus的函数，该函数根据iPossibleMVPUSlaves和cAdapterPath变量来获取MVPU的状态，并将其存储在ADLMVPUStatus结构体中。

这里需要注意的是，ADLMVPUCaps结构体中未定义所有可能的MVPU slave信息。


```cpp
/// Bits set for all possible MVPU slaves. \ref MVPU_ADAPTER_0 .. \ref MVPU_ADAPTER_3
  int iPossibleMVPUSlaves;
/// Registry path for each adapter.
  char cAdapterPath[ADL_DL_MAX_MVPU_ADAPTERS][ADL_DL_MAX_REGISTRY_PATH];
} ADLMVPUCaps;

/////////////////////////////////////////////////////////////////////////////////////////////
/// \brief Structure containing information about MultiVPU status.
///
/// This structure is used to store information about MultiVPU status.
/// Ths structure is used by the ADL_Display_MVPUStatus_Get() function.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLMVPUStatus
{
```

这段代码定义了一个名为ADLMVPUStatus的结构体，用于表示一个多路复用电源（MVPU）的状态。以下是该结构体包含的一些成员变量和成员函数的说明：

```cppc
// 成员变量
```

* `iSize`：表示MVPU状态结构体的大小。
* `iActiveAdapterCount`：表示正在激活的MVPU适配器数量。
* `iStatus`：表示MVPU的状态。
* `aAdapterLocation`：表示每个MVPU适配器的PCI总线号和设备号，以及它们在MVPU中的位置。
```cpp
// 成员函数
```

* `ADL_DL_MAX_MVPU_ADAPTERS`：定义了一个名为`ADL_DL_MAX_MVPU_ADAPTERS`的常量，用于表示MVPU最大可用的适配器数量。
```cpp
// 其他函数
```

由于这段代码中没有定义任何函数，因此无法进一步了解该代码的实际作用。建议参考相关文档和上下文，以更好地理解该代码。


```cpp
/// Must be set to sizeof( ADLMVPUStatus ).
  int iSize;
/// Number of active adapters.
  int iActiveAdapterCount;
/// MVPU status.
  int iStatus;
/// PCI Bus/Device/Function for each active adapter participating in MVPU.
  ADLAdapterLocation aAdapterLocation[ADL_DL_MAX_MVPU_ADAPTERS];
} ADLMVPUStatus;

// Displays Manager structures

///////////////////////////////////////////////////////////////////////////
/// \brief Structure containing information about the activatable source.
///
```

这段代码定义了一个名为 ADLActivatableSource 的结构体，用于存储可激活的源信息。该结构体包含五个成员变量：iAdapterIndex、iNumActivatableSources、iActivatableSourceMask、iActivatableSourceValue 和 iNumActivatableSources。

ADLActivatableSource 结构体可以用来将激活的可激活源信息返回给用户。iAdapterIndex 成员变量表示持久化逻辑适配器的索引，iNumActivatableSources 成员变量表示可激活的源的数目，iActivatableSourceMask 成员变量是一个 bitmask，用于标识激活源的数目，iActivatableSourceValue 和 iNumActivatableSources 成员变量则分别表示激活源的值和数目。

该结构体还包含两个保护成员：iActivatableSourceValue 和 iActivatableSourceValue，它们都被设置为 0，意味着目前这些源没有被激活。此外，该结构体还包含一个名为 iActivatableSource 的指针成员，用于将该结构体作为参数传递给其他函数。


```cpp
/// This structure is used to store activatable source information
/// This information can be returned to the user.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLActivatableSource
{
    /// The Persistent logical Adapter Index.
    int iAdapterIndex;
    /// The number of Activatable Sources.
    int iNumActivatableSources;
    /// The bit mask identifies the number of bits ActivatableSourceValue is using. (Not currnetly used)
    int iActivatableSourceMask;
    /// The bit mask identifies the status.  (Not currnetly used)
    int iActivatableSourceValue;
} ADLActivatableSource, *LPADLActivatableSource;

```

这是一段定义了一个名为ADLMode的结构体的头文件，用于存储当前显示适配器的显示模式。该结构体包含以下成员：

1. int iAdapterIndex：表示适配器的索引，即从0开始标识各个硬件设备。
2. ADLDisplayID：显示器的ID，通常在配置操作系统时指定。

这个结构体可以用来在应用程序中获取当前显示适配器的显示模式，例如设置广告位和清除它们。通过获取适配器的索引，可以知道要配置哪个硬件设备来显示广告位。


```cpp
/////////////////////////////////////////////////////////////////////////////////////////////
/// \brief Structure containing information about display mode.
///
/// This structure is used to store the display mode for the current adapter
/// such as X, Y positions, screen resolutions, orientation,
/// color depth, refresh rate, progressive or interlace mode, etc.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////

typedef struct ADLMode
{
/// Adapter index.
    int iAdapterIndex;
/// Display IDs.
    ADLDisplayID displayID;
```

这段代码定义了五个整型变量和一个浮点型变量，它们可能代表屏幕的位置、分辨率、颜色深度、刷新率和方向。此外，还有一个布尔型变量 iModeFlag，用于指示是否处于 Vista 模式。

具体来说，iXPos 和 iYPos 变量可能用于记录屏幕在水平或垂直方向上的位置。iXRes 和 iYRes 变量可能用于记录屏幕的分辨率。iColourDepth 变量可能用于记录屏幕颜色深度的选项，例如 16 或 32。fRefreshRate 变量可能用于记录屏幕刷新率。iOrientation 变量可能用于记录屏幕的方向，例如 0 到 180 之间的角度。最后，iModeFlag 变量可能用于指示是否处于 Vista 模式，其中 True 表示 progressive mode，而 False 表示 interlaced mode。


```cpp
/// Screen position X coordinate.
    int iXPos;
/// Screen position Y coordinate.
    int iYPos;
/// Screen resolution Width.
    int iXRes;
/// Screen resolution Height.
    int iYRes;
/// Screen Color Depth. E.g., 16, 32.
    int iColourDepth;
/// Screen refresh rate. Could be fractional E.g. 59.97
    float fRefreshRate;
/// Screen orientation. E.g., 0, 90, 180, 270.
    int iOrientation;
/// Vista mode flag indicating Progressive or Interlaced mode.
    int iModeFlag;
```

这段代码定义了一个名为ADLMode的结构体，用于存储显示目标信息。该结构体包含三个成员：iModeMask、iModeValue和ADLDisplayTarget。

具体来说，ADLMode结构体中的iModeMask成员是一个整数，用于标识当前显示模式中使用的比特位掩码，它由所有定义在define_displaymode中的比特位掩码之和得到。iModeValue成员也是一个整数，用于标识当前显示模式的状态，它与display_status成员 variable相同，该variable由定义在define_display_status成员中。

ADLDisplayTarget结构体是ADLMode结构体的别名，用于存储具体的显示目标信息。该结构体包含五个成员：displayID、iDisplayMapIndex、iDisplayTargetMask、iDisplayTargetValue和ADLDisplayTarget，其中，displayID成员用于标识显示目标，iDisplayMapIndex成员用于标识显示目标在显示地图中的索引，iDisplayTargetMask成员用于标识显示目标当前使用的比特位掩码，iDisplayTargetValue成员用于标识显示目标的状态。


```cpp
/// The bit mask identifying the number of bits this Mode is currently using. It is the sum of all the bit definitions defined in \ref define_displaymode
    int iModeMask;
/// The bit mask identifying the display status. The detailed definition is in  \ref define_displaymode
    int iModeValue;
} ADLMode, *LPADLMode;


/////////////////////////////////////////////////////////////////////////////////////////////
/// \brief Structure containing information about display target information.
///
/// This structure is used to store the display target information.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLDisplayTarget
{
    /// The Display ID.
    ADLDisplayID displayID;

    /// The display map index identify this manner and the desktop surface.
    int iDisplayMapIndex;

    /// The bit mask identifies the number of bits DisplayTarget is currently using. It is the sum of all the bit definitions defined in \ref ADL_DISPLAY_DISPLAYTARGET_PREFERRED.
    int  iDisplayTargetMask;

    /// The bit mask identifies the display status. The detailed definition is in \ref ADL_DISPLAY_DISPLAYTARGET_PREFERRED.
    int  iDisplayTargetValue;

} ADLDisplayTarget, *LPADLDisplayTarget;


```

这段代码定义了一个名为ADLBezelTransientMode的结构体，用于存储显示SLS bezel Mode信息。该结构体包含以下字段：

- iAdapterIndex：适配器索引，用于与设备连接的数据结构。
- iSLSMapIndex：SLS映射索引，用于在SLS映射表中查找与适配器相匹配的SLS映射。
- iSLSModeIndex：SLS显示模式索引，用于在SLS模式表中查找与适配器相匹配的SLS显示模式。
- displayMode：显示SLS bezel操作模式，可以是0表示正常显示，1表示重置显示，2表示防止显示等。
- iNumBezelOffset：显示SLS bezel偏移的数量。
- iFirstBezelOffsetArrayIndex：第一个SLS显示模式对应的自然模式阵列中的偏移量索引。
- iSLSBezelTransientModeMask：用于标识当前结构体使用的是哪些位。
- iSLSBezelTransientModeValue：显示SLS bezel的初始值，可以是0或1。

这个结构体可以用来在显示SLS bezel时记录相关信息，如SLS映射、显示模式、偏移量等，以便在代码中更方便地使用和操作。


```cpp
/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about the display SLS bezel Mode information.
///
/// This structure is used to store the display SLS bezel Mode information.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct tagADLBezelTransientMode
{
    /// Adapter Index
    int iAdapterIndex;

    /// SLS Map Index
    int iSLSMapIndex;

    /// The mode index
    int iSLSModeIndex;

    /// The mode
    ADLMode displayMode;

    /// The number of bezel offsets belongs to this map
    int  iNumBezelOffset;

    /// The first bezel offset array index in the native mode array
    int  iFirstBezelOffsetArrayIndex;

    /// The bit mask identifies the bits this structure is currently using. It will be the total OR of all the bit definitions.
    int  iSLSBezelTransientModeMask;

    /// The bit mask identifies the display status. The detail definition is defined below.
    int  iSLSBezelTransientModeValue;

} ADLBezelTransientMode, *LPADLBezelTransientMode;


```

这段代码定义了一个名为ADLAdapterDisplayCap的结构体，用于存储适配器显示模式的信息。这个结构体包含三个成员变量：iAdapterIndex，iAdapterDisplayCapMask和iAdapterDisplayCapValue。

iAdapterIndex成员变量表示一个持久化逻辑适配器的索引，用于标识要控制的显示模式。

iAdapterDisplayCapMask成员变量是一个用于标识ADL_ADAPTER_DISPLAYCAP_XXX位位的位掩码，它告诉编译器如何使用成员变量的值。

iAdapterDisplayCapValue成员变量包含ADL_ADAPTER_DISPLAYCAP_XXX中定义的各个位位的值，用于指示当前显示模式。

这个结构体可以被用来返回给用户，也可以被用来调用各种涉及显示设备的相关 driver 调用，以便根据用户的请求获取各种显示设备的相关显示模式设置。


```cpp
/////////////////////////////////////////////////////////////////////////////////////////////
/// \brief Structure containing information about the adapter display manner.
///
/// This structure is used to store adapter display manner information
/// This information can be returned to the user. Alternatively, it can be used to access various driver calls to
/// fetch various display device related display manner settings upon the user's request.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLAdapterDisplayCap
{
    /// The Persistent logical Adapter Index.
    int iAdapterIndex;
    /// The bit mask identifies the number of bits AdapterDisplayCap is currently using. Sum all the bits defined in ADL_ADAPTER_DISPLAYCAP_XXX
    int  iAdapterDisplayCapMask;
    /// The bit mask identifies the status. Refer to ADL_ADAPTER_DISPLAYCAP_XXX
    int  iAdapterDisplayCapValue;
} ADLAdapterDisplayCap, *LPADLAdapterDisplayCap;


```

这是一段C++代码，定义了一个名为ADLDisplayMap的结构体，用于存储显示映射信息。该结构体包含显示模式（0表示无显示，1表示水平显示，2表示垂直显示）以及显示数据，如显示顺序、行号和列号。

该代码的作用是定义一个名为ADLDisplayMap的结构体，用于存储操作系统中各种显示模式下的显示映射数据。该结构体成员包括：

1. iDisplayMapIndex：当前显示模式所对应的桌面索引，从0开始。
2. displayMode：当前显示模式，可以是0（无显示）或1（水平显示）或2（垂直显示）。
3. displayData：用于存储显示数据，包括显示顺序、行号和列号。

这个结构体可以被用来在应用程序中获取和设置显示映射信息，例如在设置界面中选择不同的显示模式时，可以从该结构体中获取当前显示模式和相应的显示数据，然后将这些信息用于绘制或更新显示。


```cpp
/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about display mapping.
///
/// This structure is used to store the display mapping data such as display manner.
/// For displays with horizontal or vertical stretch manner,
/// this structure also stores the display order, display row, and column data.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLDisplayMap
{
/// The current display map index. It is the OS desktop index. For example, if the OS index 1 is showing clone mode, the display map will be 1.
    int iDisplayMapIndex;

/// The Display Mode for the current map
    ADLMode displayMode;

```

这段代码定义了一个名为ADLDisplayMap的类，其中包含以下成员变量：

1. iNumDisplayTarget：整型变量，用于表示地图中显示目标的数量。
2. iFirstDisplayTargetArrayIndex：整型变量，用于表示目标数组中第一个显示目标的位置。
3. iDisplayMapMask：整型变量，表示显示地图当前使用的位掩码，是所有定义在ADL_DISPLAY_DISPLAYMAP_MANNER_xxx中的位掩码之和。
4. iDisplayMapValue：整型变量，表示显示地图当前的显示状态，是ADL_DISPLAY_DISPLAYMAP_MANNER_xxx中的一个枚举类型，取值为0或1。

总之，这段代码定义了一个ADLDisplayMap类型的数据结构，用于表示地图，其中包含地图中显示目标的数量、目标数组中显示目标的位置、显示地图当前使用的位掩码和显示地图当前的显示状态。


```cpp
/// The number of display targets belongs to this map\n
    int iNumDisplayTarget;

/// The first target array index in the Target array\n
    int iFirstDisplayTargetArrayIndex;

/// The bit mask identifies the number of bits DisplayMap is currently using. It is the sum of all the bit definitions defined in ADL_DISPLAY_DISPLAYMAP_MANNER_xxx.
     int  iDisplayMapMask;

///The bit mask identifies the display status. The detailed definition is in ADL_DISPLAY_DISPLAYMAP_MANNER_xxx.
    int  iDisplayMapValue;

} ADLDisplayMap, *LPADLDisplayMap;


```

这段代码定义了一个名为ADLPossibleMap的结构体，用于存储关于显示设备（可能映射）的信息。这个结构体包含以下成员：

1. iIndex：当前的可能映射索引。
2. iAdapterIndex：标识要验证的GPU的适配器ID。
3. iNumDisplayMap：用于验证的GPU上显示Map的数量。
4. displayMap：指向存储显示Map的指针。
5. iNumDisplayTarget：用于验证的显示Map的目标数量。
6. displayTarget：指向存储显示目标（目标）的指针。

这个结构体可以用来在GPU之间传输信息，以及在不同GPU上验证显示设备。


```cpp
/////////////////////////////////////////////////////////////////////////////////////////////
/// \brief Structure containing information about the display device possible map for one GPU
///
/// This structure is used to store the display device possible map
/// This information can be returned to the user.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLPossibleMap
{
    /// The current PossibleMap index. Each PossibleMap is assigned an index
    int iIndex;
    /// The adapter index identifying the GPU for which to validate these Maps & Targets
    int iAdapterIndex;
    /// Number of display Maps for this GPU to be validated
    int iNumDisplayMap;
    /// The display Maps list to validate
    ADLDisplayMap* displayMap;
    /// the number of display Targets for these display Maps
    int iNumDisplayTarget;
    /// The display Targets list for these display Maps to be validated.
    ADLDisplayTarget* displayTarget;
} ADLPossibleMap, *LPADLPossibleMap;


```

这是一个定义了一个名为 ADLPossibleMapping 的结构体，用于存储当前显示设备（如显示器）的可能映射信息。该结构体包含三个成员变量：iDisplayIndex、iDisplayControllerIndex 和 iDisplayMannerSupported。

iDisplayIndex 表示显示设备的索引，每个显示器都有一个唯一的索引。iDisplayControllerIndex 表示控制器（控制器组件）的索引，该控制器负责与显示器进行通信。iDisplayMannerSupported 表示是否支持显示器的手动操作（例如，轮幅控制）。

这个结构体是一个模板类，可以被用来定义一个或多个相同结构的变量。在实际应用中，您需要根据自己的需求定义自己的 ADLPossibleMapping 结构体。


```cpp
/////////////////////////////////////////////////////////////////////////////////////////////
/// \brief Structure containing information about display possible mapping.
///
/// This structure is used to store the display possible mapping's controller index for the current display.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLPossibleMapping
{
    int iDisplayIndex;                ///< The display index. Each display is assigned an index.
    int iDisplayControllerIndex;    ///< The controller index to which display is mapped.
    int iDisplayMannerSupported;    ///< The supported display manner.
} ADLPossibleMapping, *LPADLPossibleMapping;

/////////////////////////////////////////////////////////////////////////////////////////////
/// \brief Structure containing information about the validated display device possible map result.
```

这段代码定义了一个名为ADLPossibleMapResult的结构体，用于存储已经被验证过的显示设备可能的地图结果。这些结果可以返回给用户。该结构体定义了两个整型变量：iIndex和iPossibleMapResultMask，用于表示当前的显示地图索引和可能的地图结果掩码。此外，该结构体还定义了一个名为iPossibleMapResultValue的整型变量，用于表示可能的地图结果的具体值。

ADLPossibleMapResult结构体可以被用于在应用程序中存储和访问显示设备的地图结果。在某些情况下，用户可能需要了解当前显示设备正在使用的地图结果。因此，该结构体提供了一个方便的方式来存储和获取这些信息。


```cpp
///
/// This structure is used to store the validated display device possible map result
/// This information can be returned to the user.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLPossibleMapResult
{
    /// The current display map index. It is the OS Desktop index. For example, OS Index 1 showing clone mode. The Display Map will be 1.
    int iIndex;
    // The bit mask identifies the number of bits   PossibleMapResult is currently using. It will be the sum all the bit definitions defined in ADL_DISPLAY_POSSIBLEMAPRESULT_VALID.
    int iPossibleMapResultMask;
    /// The bit mask identifies the possible map result. The detail definition is defined in ADL_DISPLAY_POSSIBLEMAPRESULT_XXX.
    int iPossibleMapResultValue;
} ADLPossibleMapResult, *LPADLPossibleMapResult;

```

这是一个关于ADLSLSGrid结构体的定义，它用于存储显示SLS网格信息。这个结构体包含三个成员： adapter索引、grid索引和grid行。

注意：我没有对这段代码进行进一步的解析，因为它可能受到不同编程语言或库的影响。如果你需要更详细的信息，请提供更多关于这个代码的上下文和背景。


```cpp
/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about the display SLS Grid information.
///
/// This structure is used to store the display SLS Grid information.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLSLSGrid
{
/// The Adapter index.
    int iAdapterIndex;

/// The grid index.
    int  iSLSGridIndex;

/// The grid row.
    int  iSLSGridRow;

```

这是一个C++语言的代码，定义了一个名为ADLSLSGrid的结构体，用于表示显示SLS Map的信息。这个结构体包含三个成员变量：iSLSGridColumn、iSLSGridMask和iSLSGridValue。

iSLSGridColumn表示当前显示SLS Map列的数量，从0到255。

iSLSGridMask是一个整数，用于标识显示SLS Map currently使用的比特数，所有定义在ADL_DISPLAY_SLSGRID_ORIENTATION_XXX中的比特数之和。

iSLSGridValue是一个整数，用于标识显示SLS Map的状态，它参考了ADL_DISPLAY_SLSGRID_ORIENTATION_XXX中定义的显示状态。

该结构体成员函数可以用来设置和获取ADLSLSGrid结构体中的成员变量，以访问和修改显示SLS Map的信息。


```cpp
/// The grid column.
    int  iSLSGridColumn;

/// The grid bit mask identifies the number of bits DisplayMap is currently using. Sum of all bits defined in ADL_DISPLAY_SLSGRID_ORIENTATION_XXX
    int  iSLSGridMask;

/// The grid bit value identifies the display status. Refer to ADL_DISPLAY_SLSGRID_ORIENTATION_XXX
    int  iSLSGridValue;

} ADLSLSGrid, *LPADLSLSGrid;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about the display SLS Map information.
///
/// This structure is used to store the display SLS Map information.
```

这段代码定义了一个名为 ADLSLSMap 的结构体，用于表示联想无线设备管理器（ADL）和软件（SLS）地图的属性。该结构体包含多个整型变量，表示地图的各个属性，包括 adapterIndex、SLSMapIndex、grid、iSurfaceMapIndex、iOrientation、iNumSLSTarget、iFirstSLSTargetArrayIndex、iNumNativeMode、iFirstNativeModeArrayIndex、iNumBezelMode 和 iFirstBezelModeArrayIndex。

ADLSLSMap 的定义包括了一系列用于设置和获取 ADL 和 JSOM 的属性的变量。例如，iAdapterIndex 变量表示适配器索引，iSLSMapIndex 变量表示当前的显示图（OS）索引，grid 变量表示当前的网格，iSurfaceMapIndex 变量表示显示的表面（克隆模式）的索引，iOrientation 变量表示屏幕的旋转角度，iNumSLSTarget 和 iNumNativeMode 变量表示显示的目标和原生模式的数量，iFirstSLSTargetArrayIndex 和 iFirstNativeModeArrayIndex 变量表示显示目标的首地址和首元素，iNumBezelMode 和 iNumBezelModeArrayIndex 变量表示 Bezel 模式的数量和首元素，iFirstBezelModeArrayIndex 和 iFirstBezelModeArrayIndex 变量表示 Bezel 首元素的数组和索引，iNumBezelOffset 和 iNumBezelOffsetArrayIndex 变量表示 Bezel 首元素的偏移量和首元素数组，iSLSMapMask 和 iSLSMapValue 变量表示显示地图的位图和状态。


```cpp
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct    ADLSLSMap
{
    /// The Adapter Index
    int iAdapterIndex;

    /// The current display map index. It is the OS Desktop index. For example, OS Index 1 showing clone mode. The Display Map will be 1.
    int iSLSMapIndex;

    /// Indicate the current grid
    ADLSLSGrid grid;

    /// OS surface index
    int  iSurfaceMapIndex;

     ///  Screen orientation. E.g., 0, 90, 180, 270
     int iOrientation;

    /// The number of display targets belongs to this map
    int  iNumSLSTarget;

    /// The first target array index in the Target array
    int  iFirstSLSTargetArrayIndex;

    /// The number of native modes belongs to this map
    int  iNumNativeMode;

    /// The first native mode array index in the native mode array
    int  iFirstNativeModeArrayIndex;

    /// The number of bezel modes belongs to this map
    int  iNumBezelMode;

    /// The first bezel mode array index in the native mode array
    int  iFirstBezelModeArrayIndex;

    /// The number of bezel offsets belongs to this map
    int  iNumBezelOffset;

    /// The first bezel offset array index in the
    int  iFirstBezelOffsetArrayIndex;

    /// The bit mask identifies the number of bits DisplayMap is currently using. Sum all the bit definitions defined in ADL_DISPLAY_SLSMAP_XXX.
    int  iSLSMapMask;

    /// The bit mask identifies the display map status. Refer to ADL_DISPLAY_SLSMAP_XXX
    int  iSLSMapValue;


} ADLSLSMap, *LPADLSLSMap;

```

这段代码定义了一个名为ADLSLSOffset的结构体，用于存储与显示SLS Offset相关的信息。该结构体包含以下字段：

iAdapterIndex：适配器索引
iSLSMapIndex：当前显示map索引，从0开始
displayID：显示ID
iBezelModeIndex：SLS Bezel Mode Index
iBezelOffsetX：SLS Bezel Offset X
iBezelOffsetY：SLS Bezel Offset Y
iDisplayWidth：显示宽度
iDisplayHeight：显示高度
iBezelOffsetMask：显示SLS Offset的位图
iBezelOffsetValue：显示SLS Offset的值，从0开始。

ADLSLSOffset是该结构体的别名，用于引用内部字段。

该结构体用于在应用程序中设置和获取与显示SLS Offset相关的信息，例如在适配器连接或显示SLS Offset更改时更新显示设备的数据。


```cpp
/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about the display SLS Offset information.
///
/// This structure is used to store the display SLS Offset information.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLSLSOffset
{
    /// The Adapter Index
    int iAdapterIndex;

    /// The current display map index. It is the OS Desktop index. For example, OS Index 1 showing clone mode. The Display Map will be 1.
    int iSLSMapIndex;

    /// The Display ID.
    ADLDisplayID displayID;

    /// SLS Bezel Mode Index
    int iBezelModeIndex;

    /// SLS Bezel Offset X
    int iBezelOffsetX;

    /// SLS Bezel Offset Y
    int iBezelOffsetY;

    /// SLS Display Width
    int iDisplayWidth;

    /// SLS Display Height
    int iDisplayHeight;

    /// The bit mask identifies the number of bits Offset is currently using.
    int iBezelOffsetMask;

    /// The bit mask identifies the display status.
    int  iBezelffsetValue;
} ADLSLSOffset, *LPADLSLSOffset;

```

这段代码定义了一个名为 ADLSLSMode 的结构体，用于存储与显示 SLS 模式信息相关的信息。该结构体包含以下字段：

1. iAdapterIndex：适配器索引。
2. iSLSMapIndex：当前显示模式所使用的显示映射器的索引。
3. iSLSModeIndex：当前显示模式的索引。
4. displayMode：当前显示模式的模拟模式。
5. iSLSNativeModeMask：用于标识当前显示模式是否为模拟模式的位掩码。
6. iSLSNativeModeValue：用于标识当前显示模式是否正在使用的位掩码。

该结构体用于在应用程序中存储和操作显示 SLS 模式信息。通过该结构体，用户可以设置或获取显示 SLS 模式信息，以便更好地控制设备的行为。


```cpp
/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about the display SLS Mode information.
///
/// This structure is used to store the display SLS Mode information.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLSLSMode
{
    /// The Adapter Index
    int iAdapterIndex;

    /// The current display map index. It is the OS Desktop index. For example, OS Index 1 showing clone mode. The Display Map will be 1.
    int iSLSMapIndex;

    /// The mode index
    int iSLSModeIndex;

    /// The mode for this map.
    ADLMode displayMode;

    /// The bit mask identifies the number of bits Mode is currently using.
    int iSLSNativeModeMask;

    /// The bit mask identifies the display status.
    int iSLSNativeModeValue;
} ADLSLSMode, *LPADLSLSMode;




```

这段代码定义了一个名为ADLPossibleSLSMap的结构体，用于存储与显示有关的SLS Map信息。该结构体包含以下成员：

1. iSLSMapIndex：当前显示map的索引，它是一个OS索引，用于标识要显示的显示map。例如，OS索引1表示克隆模式，显示map索引为1。
2. iNumSLSMap：用于存储显示map的数量。
3. lpSLSMap：指向SLSMap的指针，用于存储验证的显示map信息。
4. iNumSLSTarget：用于存储验证的SLSTarget的数量。
5. lpDisplayTarget：指向DisplayTarget的指针，用于存储验证的显示目标信息。

该结构体可以被用于在应用程序中验证显示map和SLS Map信息。


```cpp
/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about the display Possible SLS Map information.
///
/// This structure is used to store the display Possible SLS Map information.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLPossibleSLSMap
{
    /// The current display map index. It is the OS Desktop index.
    /// For example, OS Index 1 showing clone mode. The Display Map will be 1.
    int iSLSMapIndex;

    /// Number of display map to be validated.
    int iNumSLSMap;

    /// The display map list for validation
    ADLSLSMap* lpSLSMap;

    /// the number of display map config to be validated.
    int iNumSLSTarget;

    /// The display target list for validation.
    ADLDisplayTarget* lpDisplayTarget;
} ADLPossibleSLSMap, *LPADLPossibleSLSMap;


```

这段代码定义了一个名为 ADLSLSTarget 的结构体，用于存储 SLS（系统级淋巴系统）目标的信息。这个结构体包含以下字段：

1. iAdapterIndex：逻辑适配器索引。
2. iSLSMapIndex：SLS 地图索引。
3. displayTarget：目标显示名称。
4. iSLSGridPositionX：目标在 SLS 网格中的行数。
5. iSLSGridPositionY：目标在 SLS 网格中的列数。
6. viewSize：每个 SLS 目标的有效视图大小、宽度和旋转角度。
7. iSLSTargetMask：用于表示 iSLSTargetValue 位位的位mask。
8. iSLSTargetValue：表示 iSLSTargetMask 中标记为位 1 的位。

ADLSLSTarget 结构体用于在应用程序和用户界面之间传递 SLS 目标信息，例如在基于 SLS 的环境中，用户可能需要获取不同 SLS 目标的信息，这些信息存储在 ADLSLSTarget 结构体中，然后通过某种方式（如按钮点击）获取并显示这些信息。


```cpp
/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about the SLS targets.
///
/// This structure is used to store the SLS targets information.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLSLSTarget
{
    /// the logic adapter index
    int iAdapterIndex;

    /// The SLS map index
    int iSLSMapIndex;

    /// The target ID
    ADLDisplayTarget displayTarget;

    /// Target postion X in SLS grid
    int iSLSGridPositionX;

    /// Target postion Y in SLS grid
    int iSLSGridPositionY;

    /// The view size width, height and rotation angle per SLS Target
    ADLMode viewSize;

    /// The bit mask identifies the bits in iSLSTargetValue are currently used
    int iSLSTargetMask;

    /// The bit mask identifies status info. It is for function extension purpose
    int iSLSTargetValue;

} ADLSLSTarget, *LPADLSLSTarget;

```

这段代码定义了一个名为ADLBezelOffsetSteppingSize的结构体，用于存储Adapteroffset Stepping Size信息。

ADLBezelOffsetSteppingSize结构体包含以下成员：

* iAdapterIndex：逻辑适配器索引
* iSLSMapIndex：SLS地图索引
* iBezelOffsetSteppingSizeX：贝塞尔X轴 Stepping Size 偏移量
* iBezelOffsetSteppingSizeY：贝塞尔Y轴 Stepping Size 偏移量
* iBezelOffsetSteppingSizeMask：当前 Stepping Size 偏移量的掩码，用于确定该结构体当前正在使用哪些位
* iBezelOffsetSteppingSizeValue：当前 Stepping Size 偏移量的值

该结构体用于在贝塞尔轴向上递增的 Stepping Size，以适应不同设备的对齐需求。在初始化时，iBezelOffsetSteppingSizeX 和 iBezelOffsetSteppingSizeY 都为 0。如果当前设备以横向或纵向对齐，iBezelOffsetSteppingSizeMask 将设置为 0，iBezelOffsetSteppingSizeValue 将设置为 0。否则，iBezelOffsetSteppingSizeMask 将设置为 256，iBezelOffsetSteppingSizeValue 将设置为设备当前对齐模式的垂直和水平的 Stepping Size 偏移量之和。

在设置完结构体后，如果想让 Adapteroffset Stepping Size 生效，则需要将 iBezelOffsetSteppingSize 设置为 1，并将 iBezelOffsetSteppingSizeMask 设置为 Adapteroffset Stepping Size 的掩码。


```cpp
/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about the Adapter offset stepping size.
///
/// This structure is used to store the Adapter offset stepping size information.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLBezelOffsetSteppingSize
{
    /// the logic adapter index
    int iAdapterIndex;

    /// The SLS map index
    int iSLSMapIndex;

    /// Bezel X stepping size offset
    int iBezelOffsetSteppingSizeX;

    /// Bezel Y stepping size offset
    int iBezelOffsetSteppingSizeY;

    /// Identifies the bits this structure is currently using. It will be the total OR of all the bit definitions.
    int iBezelOffsetSteppingSizeMask;

    /// Bit mask identifies the display status.
    int iBezelOffsetSteppingSizeValue;

} ADLBezelOffsetSteppingSize, *LPADLBezelOffsetSteppingSize;

```

这段代码定义了一个名为 ADLSLSOverlappedMode 的结构体，用于存储所有 SLS 模式下重叠设置的信息。该结构体包含两个成员变量：SLSMode 和 iNumSLSTarget，分别表示 SLS 模式和目标显示的数量。第三个成员变量 iFirstTargetArrayIndex 用于引用目标数组中的第一个元素。该结构体可以被用于存储多个 SLS 模式下的重叠设置信息。


```cpp
/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about the overlap offset info for all the displays for each SLS mode.
///
/// This structure is used to store the no. of overlapped modes for each SLS Mode once user finishes the configuration from Overlap Widget
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLSLSOverlappedMode
{
    /// the SLS mode for which the overlap is configured
    ADLMode SLSMode;
    /// the number of target displays in SLS.
    int iNumSLSTarget;
    /// the first target array index in the target array
    int iFirstTargetArrayIndex;
}ADLSLSTargetOverlap, *LPADLSLSTargetOverlap;

```

这段代码定义了一个名为 ADLPXConfigCaps 的结构体，用于存储 PowerExpress 配置项。这个结构体包含两个整型变量 iAdapterIndex 和 iPXConfigCapMask，分别表示适配器的索引和 PowerExpress 配置项的位掩码。另外，它还包含一个整型变量 iPXConfigCapValue，表示 PowerExpress 配置项的值。

ADLPXConfigCaps 这个结构体是整个驱动程序中重要的一个部分，它提供了 PowerExpress 配置项的定义，通过这个结构体，开发人员可以了解 PowerExpress 支持的功能和约束，从而更好地开发和调整驱动程序。


```cpp
/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about driver supported PowerExpress Config Caps
///
/// This structure is used to store the driver supported PowerExpress Config Caps
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLPXConfigCaps
{
    /// The Persistent logical Adapter Index.
    int iAdapterIndex;

    /// The bit mask identifies the number of bits PowerExpress Config Caps is currently using. It is the sum of all the bit definitions defined in ADL_PX_CONFIGCAPS_XXXX /ref define_powerxpress_constants.
    int  iPXConfigCapMask;

    /// The bit mask identifies the PowerExpress Config Caps value. The detailed definition is in ADL_PX_CONFIGCAPS_XXXX /ref define_powerxpress_constants.
    int  iPXConfigCapValue;

} ADLPXConfigCaps, *LPADLPXConfigCaps;

```

这段代码定义了一个名为ADLPDXTYPE的枚举类型，用于表示PX或HG类型。该枚举类型包含四个成员变量，分别为ADL_PX_NONE、ADL_SWITCHABLE_AMDAMD、ADL_HG_AMDAMD和ADL_HG_AMDOther，分别对应枚举类型的四个成员变量。

通过使用这个枚举类型，程序可以在代码中根据输入的值来明确当前处理器架构的类型，有助于代码的可读性和可维护性。例如，在代码中需要根据ADL_PX_NONE枚举类型的值来设置某些编译选项的值，或者需要根据当前处理器架构的类型来选择正确的类型进行操作。


```cpp
/////////////////////////////////////////////////////////////////////////////////////////
///\brief Enum containing PX or HG type
///
/// This enum is used to get PX or hG type
///
/// \nosubgrouping
//////////////////////////////////////////////////////////////////////////////////////////
enum ADLPxType
{
	//Not AMD related PX/HG or not PX or HG at all
	ADL_PX_NONE = 0,
	//A+A PX
	ADL_SWITCHABLE_AMDAMD = 1,
	// A+A HG
	ADL_HG_AMDAMD = 2,
	//A+I PX
	ADL_SWITCHABLE_AMDOTHER = 3,
	//A+I HG
	ADL_HG_AMDOTHER = 4,
};


```

这段代码定义了一个名为ADLApplicationData的结构体，用于存储一个应用程序的基本信息。这个结构体包含以下字段：

1. strPathName：应用程序的路径名称，最大长度为ADL_MAX_PATH。
2. strFileName：应用程序的文件名，最大长度为ADL_APP_PROFILE_FILENAME_LENGTH。
3. strTimeStamp：应用程序的创建时间戳，最大长度为ADL_APP_PROFILE_TIMESTAMP_LENGTH。
4. strVersion：应用程序的版本号，最大长度为ADL_APP_PROFILE_VERSION_LENGTH。

这个结构体可以被用于在应用程序开发过程中收集和存储信息，例如用于配置文件或者记录应用程序版本号等信息。


```cpp
/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about an application
///
/// This structure is used to store basic information of an application
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct _ADLApplicationData
{
    /// Path Name
    char strPathName[ADL_MAX_PATH];
    /// File Name
    char strFileName[ADL_APP_PROFILE_FILENAME_LENGTH];
    /// Creation timestamp
    char strTimeStamp[ADL_APP_PROFILE_TIMESTAMP_LENGTH];
    /// Version
    char strVersion[ADL_APP_PROFILE_VERSION_LENGTH];
}ADLApplicationData;

```

这段代码定义了一个名为 ADLApplicationDataX2 的结构体，用于存储一个应用程序的基本信息。该结构体包含以下成员：

1. 路径名称：一个指向名为 [ADL_MAX_PATH] 的常量路径的 wchar_t 类型的变量。
2. 文件名：一个指向名为 [ADL_APP_PROFILE_FILENAME_LENGTH] 的常量文件名的 wchar_t 类型的变量。
3. 创建时间戳：一个指向名为 [ADL_APP_PROFILE_TIMESTAMP_LENGTH] 的常量时间戳的 wchar_t 类型的变量。
4. 版本：一个指向名为 [ADL_APP_PROFILE_VERSION_LENGTH] 的常量版本的名的 wchar_t 类型的变量。

这个结构体可以用于在应用程序中获取和设置基本信息，例如文件名、路径、时间戳和版本等。


```cpp
/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about an application
///
/// This structure is used to store basic information of an application
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct _ADLApplicationDataX2
{
    /// Path Name
    wchar_t strPathName[ADL_MAX_PATH];
    /// File Name
    wchar_t strFileName[ADL_APP_PROFILE_FILENAME_LENGTH];
    /// Creation timestamp
    wchar_t strTimeStamp[ADL_APP_PROFILE_TIMESTAMP_LENGTH];
    /// Version
    wchar_t strVersion[ADL_APP_PROFILE_VERSION_LENGTH];
}ADLApplicationDataX2;

```

这段代码定义了一个名为 ADLApplicationDataX3 的结构体，用于存储一个应用程序的基本信息，包括应用程序的进程 ID。这个结构体包含以下字段：

1. 路径名称：存储应用程序的完整路径名，包括文件夹和文件名。
2. 文件名：存储应用程序的文件名，仅在进程 ID 不同的文件中可见。
3. 创建时间戳：存储应用程序的创建时间，格式为 年-月-日 时：分：秒， milliseconds。
4. 版本：存储应用程序的版本号，仅在进程 ID 不同的文件中可见。
5. 进程 ID：存储应用程序的进程 ID，仅在进程 ID 不同的文件中可见。

这个结构体可以被用于在应用程序中获取和设置其基本信息，例如在 Windows 系统中使用 Application补丁时，可以使用 Get都得 LLCodeForResource 函数获取应用程序的进程 ID。


```cpp
/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about an application
///
/// This structure is used to store basic information of an application including process id
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct _ADLApplicationDataX3
{
    /// Path Name
    wchar_t strPathName[ADL_MAX_PATH];
    /// File Name
    wchar_t strFileName[ADL_APP_PROFILE_FILENAME_LENGTH];
    /// Creation timestamp
    wchar_t strTimeStamp[ADL_APP_PROFILE_TIMESTAMP_LENGTH];
    /// Version
    wchar_t strVersion[ADL_APP_PROFILE_VERSION_LENGTH];
    //Application Process id
    unsigned int iProcessId;
}ADLApplicationDataX3;

```

这段代码定义了一个名为 `_PropertyRecord` 的结构体类型，它包含一个应用程序配置文件profile中某个属性的相关信息。这个结构体类型包含以下字段：

1. `strName`：应用程序配置文件中该属性的名称，它是一个字符数组，最多可以包含 ADL_APP_PROFILE_PROPERTY_LENGTH 个字符。
2. `eType`：该属性的数据类型，根据这个数据类型可以决定如何存储和处理该属性。
3. `iDataSize`：该属性的数据大小，以字节为单位。
4. `uData`：该属性的原始数据，可以是任何数据类型。

这个结构体类型可以用来在应用程序配置文件中定义和存储某个属性的相关信息，然后在程序中根据需要进行访问和修改。


```cpp
/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information of a property of an application profile
///
/// This structure is used to store property information of an application profile
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct _PropertyRecord
{
    /// Property Name
    char strName [ADL_APP_PROFILE_PROPERTY_LENGTH];
    /// Property Type
    ADLProfilePropertyType eType;
    /// Data Size in bytes
    int iDataSize;
    /// Property Value, can be any data type
    unsigned char uData[1];
}PropertyRecord;

```

这段代码定义了一个名为`_ADLApplicationProfile`的结构体，用于存储应用程序配置文件中的信息。该结构体包含以下成员：

1. `iCount`：表示该结构体中属性的数量。
2. `record`：一个数组，用于存储所有属性的记录。每个记录包含以下字段：
	* ` propertyType`：属性的数据类型，如整型、文本等。
	* ` propertyId`：属性的ID，用于标识该属性。
	* ` propertyValue`：属性的值。

该结构体的定义在`ADLApplicationProfile`的定义中，并可以在应用程序配置文件中使用。


```cpp
/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about an application profile
///
/// This structure is used to store information of an application profile
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct _ADLApplicationProfile
{
    /// Number of properties
    int iCount;
    /// Buffer to store all property records
    PropertyRecord record[1];
}ADLApplicationProfile;


```

这段代码定义了一个名为ADLPowerControlInfo的结构体，用于存储关于OD5 Power Control功能的个人信息。

ADLPowerControlInfo结构体包含三个成员变量：

1. iMinValue：最低值。
2. iMaxValue：最高值。
3. iStepValue：在最小值和最高值之间每次变化的最小步骤。

这个结构体可以被用来描述一个OD5 Power Control feature，例如，可以用于更新某个功能的最低电压值或者最高电压值。


```cpp
/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about an OD5 Power Control feature
///
/// This structure is used to store information of an Power Control feature
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLPowerControlInfo
{
/// Minimum value.
int iMinValue;
/// Maximum value.
int iMaxValue;
/// The minimum change in between minValue and maxValue.
int iStepValue;
 } ADLPowerControlInfo;

```

这段代码定义了一个名为 ADLControllerMode 的结构体，用于存储关于控制器模式的有关信息。该结构体包含以下成员：

iModifiers：表示应用程序将在视口中应用的操作类型。
iViewPositionCx：水平视图的起始位置，单位为像素。
iViewPositionCy：垂直视图的起始位置，单位为像素。
iViewPanLockLeft：水平左 panlock 的起始位置，单位为像素。
iViewPanLockRight：水平右 panlock 的起始位置，单位为像素。
iViewPanLockTop：垂直上 panlock 的起始位置，单位为像素。
iViewPanLockBottom：垂直下 panlock 的起始位置，单位为像素。
iViewResolutionCx：视口的宽度，单位为像素。
iViewResolutionCy：视口的高度，单位为像素。

这个结构体可能用于在某些游戏引擎中管理不同的控制器模式，并根据这些模式应用不同的动作。


```cpp
/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about an controller mode
///
/// This structure is used to store information of an controller mode
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct _ADLControllerMode
{
    /// This falg indicates actions that will be applied by set viewport
    /// The value can be a combination of ADL_CONTROLLERMODE_CM_MODIFIER_VIEW_POSITION,
    /// ADL_CONTROLLERMODE_CM_MODIFIER_VIEW_PANLOCK and ADL_CONTROLLERMODE_CM_MODIFIER_VIEW_SIZE
    int iModifiers;

    /// Horizontal view starting position
    int iViewPositionCx;

    /// Vertical view starting position
    int iViewPositionCy;

    /// Horizontal left panlock position
    int iViewPanLockLeft;

    /// Horizontal right panlock position
    int iViewPanLockRight;

    /// Vertical top panlock position
    int iViewPanLockTop;

    /// Vertical bottom panlock position
    int iViewPanLockBottom;

    /// View resolution in pixels (width)
    int iViewResolutionCx;

    /// View resolution in pixels (hight)
    int iViewResolutionCy;
}ADLControllerMode;

```

这段代码定义了一个名为ADLDisplayIdentifier的结构体，用于存储有关显示的信息。这个结构体包含以下成员：

1. ulDisplayIndex：ADL显示的索引号，从0开始。
2. ulManufacturerId：显示制造商的ID。
3. ulProductId：显示产品的ID。
4. ulSerialNo：显示的串号。

这个结构体可能被用于在应用程序中获取和设置显示的相关信息，例如获取显示制造商和产品ID，以及获取显示的串号。


```cpp
/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about a display
///
/// This structure is used to store information about a display
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLDisplayIdentifier
{
    /// ADL display index
    long ulDisplayIndex;

    /// manufacturer ID of the display
    long ulManufacturerId;

    /// product ID of the display
    long ulProductId;

    /// serial number of the display
    long ulSerialNo;

} ADLDisplayIdentifier;

```

这段代码定义了一个名为 ADLOD6ParameterRange 的结构体，用于存储 Overdrive 6 clock range 的信息。这个结构体包含三个成员变量：iMin、iMax 和 iStep，分别表示时钟范围的最小值、最大值和时间步长。这个结构体可以被用于描述 Overdrive 6 的 clock range，使得程序可以方便地读取和设置这些参数。


```cpp
/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about Overdrive 6 clock range
///
/// This structure is used to store information about Overdrive 6 clock range
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct _ADLOD6ParameterRange
{
    /// The starting value of the clock range
    int     iMin;
    /// The ending value of the clock range
    int     iMax;
    /// The minimum increment between clock values
    int     iStep;

} ADLOD6ParameterRange;

```

这段代码定义了一个名为`ADLOD6Capabilities`的结构体，用于存储Overdrive 6的 Capability 信息。该结构体包含以下字段：

- `iCapabilities`：包含一个表示 OD6 capability  flags 的位图。
- `iSupportedStates`：包含一个表示 OD6 支持状态的位图。当前仅支持性能状态。
- `iNumberOfPerformanceLevels`：表示 OD6 性能级别的数量。
- `sEngineClockRange`：包含工程时钟范围的最小值和最大值。
- `sMemoryClockRange`：包含内存时钟范围的最小值和最大值。
- `iExtValue`：表示未来可用的扩展值。
- `iExtMask`：表示未来可用的扩展掩码。

这个结构体用于在代码中声明和初始化 OD6 的 Capability 信息，包括其工程时钟范围和内存时钟范围，以及支持性能级别等。


```cpp
/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about Overdrive 6 capabilities
///
/// This structure is used to store information about Overdrive 6 capabilities
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct _ADLOD6Capabilities
{
    /// Contains a bitmap of the OD6 capability flags.  Possible values: \ref ADL_OD6_CAPABILITY_SCLK_CUSTOMIZATION,
    /// \ref ADL_OD6_CAPABILITY_MCLK_CUSTOMIZATION, \ref ADL_OD6_CAPABILITY_GPU_ACTIVITY_MONITOR
    int     iCapabilities;
    /// Contains a bitmap indicating the power states
    /// supported by OD6.  Currently only the performance state
    /// is supported. Possible Values: \ref ADL_OD6_SUPPORTEDSTATE_PERFORMANCE
    int     iSupportedStates;
    /// Number of levels. OD6 will always use 2 levels, which describe
    /// the minimum to maximum clock ranges.
    /// The 1st level indicates the minimum clocks, and the 2nd level
    /// indicates the maximum clocks.
    int     iNumberOfPerformanceLevels;
    /// Contains the hard limits of the sclk range.  Overdrive
    /// clocks cannot be set outside this range.
    ADLOD6ParameterRange     sEngineClockRange;
    /// Contains the hard limits of the mclk range.  Overdrive
    /// clocks cannot be set outside this range.
    ADLOD6ParameterRange     sMemoryClockRange;

    /// Value for future extension
    int     iExtValue;
    /// Mask for future extension
    int     iExtMask;

} ADLOD6Capabilities;

```

这段代码定义了一个名为ADLOD6PerformanceLevel的结构体，用于存储Overdrive 6的时钟值。该结构体包含两个成员变量：iEngineClock和iMemoryClock，分别表示发动机和内存的时钟。

这个结构体可以被用来在程序中存储关于Overdrive 6时钟值的信息。通过这个结构体，程序可以更方便地操作Overdrive 6的时钟值，以便更好地了解其运行状况。


```cpp
/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about Overdrive 6 clock values.
///
/// This structure is used to store information about Overdrive 6 clock values.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct _ADLOD6PerformanceLevel
{
    /// Engine (core) clock.
    int iEngineClock;
    /// Memory clock.
    int iMemoryClock;

} ADLOD6PerformanceLevel;

```

这段代码定义了一个名为 ADLOD6StateInfo 的结构体，用于存储 Overdrive 6  clock 的信息。该结构体包含一个名为 iNumberOfPerformanceLevels 的整数，其值为 2，表示 OD6 使用 clock ranges 而不是 discrete performance levels。

结构体内部包含一个名为 iExtValue 的整数，用于表示未来扩展的支持值；一个名为 iExtMask 的整数，用于指定未来扩展的掩码。

此外，结构体内部还包含一个名为 aLevels 的指针，该指针指向一个名为 ADLOD6PerformanceLevel 的数组，该数组包含 OD6  clock 的 levels 数组。

最后，该结构体还包含一个名为 iNumberOfClocks 的整数，用于表示 OD6 clock ranges 的数量。


```cpp
/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about Overdrive 6 clocks.
///
/// This structure is used to store information about Overdrive 6 clocks.  This is a
/// variable-sized structure.  iNumberOfPerformanceLevels indicate how many elements
/// are contained in the aLevels array.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct _ADLOD6StateInfo
{
    /// Number of levels.  OD6 uses clock ranges instead of discrete performance levels.
    /// iNumberOfPerformanceLevels is always 2.  The 1st level indicates the minimum clocks
    /// in the range.  The 2nd level indicates the maximum clocks in the range.
    int     iNumberOfPerformanceLevels;

    /// Value for future extension
    int     iExtValue;
    /// Mask for future extension
    int     iExtMask;

    /// Variable-sized array of levels.
    /// The number of elements in the array is specified by iNumberofPerformanceLevels.
    ADLOD6PerformanceLevel aLevels [1];

} ADLOD6StateInfo;

```

这段代码定义了一个名为 ADLOD6CurrentStatus 的结构体，用于存储 Overdrive 6 的当前性能状态。这个结构体包含以下成员：

* iEngineClock：当前发动机时钟，以 10 KHz 为单位。
* iMemoryClock：当前内存时钟，以 10 KHz 为单位。
* iActivityPercent：当前 GPU 活动百分比，指示 GPU 利用率。
* iCurrentPerformanceLevel：当前 GPU 性能级别，从 0 到 100 的整数。
* iCurrentBusSpeed：当前 PCI-E 总线速度，以 Mbps 为单位。
* iCurrentBusLanes：当前 PCI-E 总线通道数，从 1 到 4 的整数。
* iMaximumBusLanes：最大可能的 PCI-E 总线通道数，从 1 到 4 的整数。
* iExtValue：当前性能状态的扩展值。
* iExtMask：当前性能状态的扩展位掩码。

这个结构体用于在 Overdrive 6 的控制面板和使用说明中获取和设置这些成员。


```cpp
/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about current Overdrive 6 performance status.
///
/// This structure is used to store information about current Overdrive 6 performance status.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct _ADLOD6CurrentStatus
{
    /// Current engine clock in 10 KHz.
    int     iEngineClock;
    /// Current memory clock in 10 KHz.
    int     iMemoryClock;
    /// Current GPU activity in percent.  This
    /// indicates how "busy" the GPU is.
    int     iActivityPercent;
    /// Not used.  Reserved for future use.
    int     iCurrentPerformanceLevel;
    /// Current PCI-E bus speed
    int     iCurrentBusSpeed;
    /// Current PCI-E bus # of lanes
    int     iCurrentBusLanes;
    /// Maximum possible PCI-E bus # of lanes
    int     iMaximumBusLanes;

    /// Value for future extension
    int     iExtValue;
    /// Mask for future extension
    int     iExtMask;

} ADLOD6CurrentStatus;

```

这段代码定义了一个名为 ADLOD6ThermalControllerCaps 的结构体，用于存储 Overdrive 6 型 thermal controller 的功能。该结构体包含一个表示 thermal controller 功能位图的 bitmap，以及一些与 fan speed 相关的参数。

具体来说，该结构体包含以下参数：

- iCapabilities：表示 thermal controller 支持的功能位，包括 TCCAPS_ETH_POWER_OFF、TCCAPS_ETH_POWER_ON、TCCAPS_FANSPEED_CONTROL、TCCAPS_FANSPEED_PERCENT_READ、TCCAPS_FANSPEED_PERCENT_WRITE、TCCAPS_FANSPEED_RPM_READ 和 TCCAPS_FANSPEED_RPM_WRITE。
- iFanMinPercent：表示 fan speed  minimum 控制率，即 fan speed 低于该值时需要采取的控制措施。这个值的范围是 0 到 100，其中 100 表示完全控制 fan speed。
- iFanMaxPercent：表示 fan speed 最大控制率，即 fan speed 高于该值时需要采取的控制措施。这个值的范围是 0 到 100，其中 100 表示完全控制 fan speed。
- iFanMinRPM：表示 minimum fan speed，即单位时间内最低转速。这个值的范围是 0 到 65535，其中 65535 表示 maximum value。
- iFanMaxRPM：表示 maximum fan speed，即单位时间内最高转速。这个值的范围是 0 到 65535，其中 65535 表示 maximum value。

此外，该结构体还包含一些用于 future extension 的值和用于 future extension 的 bitmask。


```cpp
/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about Overdrive 6 thermal contoller capabilities
///
/// This structure is used to store information about Overdrive 6 thermal controller capabilities
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct _ADLOD6ThermalControllerCaps
{
    /// Contains a bitmap of thermal controller capability flags. Possible values: \ref ADL_OD6_TCCAPS_THERMAL_CONTROLLER, \ref ADL_OD6_TCCAPS_FANSPEED_CONTROL,
    /// \ref ADL_OD6_TCCAPS_FANSPEED_PERCENT_READ, \ref ADL_OD6_TCCAPS_FANSPEED_PERCENT_WRITE, \ref ADL_OD6_TCCAPS_FANSPEED_RPM_READ, \ref ADL_OD6_TCCAPS_FANSPEED_RPM_WRITE
    int     iCapabilities;
    /// Minimum fan speed expressed as a percentage
    int     iFanMinPercent;
    /// Maximum fan speed expressed as a percentage
    int     iFanMaxPercent;
    /// Minimum fan speed expressed in revolutions-per-minute
    int     iFanMinRPM;
    /// Maximum fan speed expressed in revolutions-per-minute
    int     iFanMaxRPM;

    /// Value for future extension
    int     iExtValue;
    /// Mask for future extension
    int     iExtMask;

} ADLOD6ThermalControllerCaps;

```

这段代码定义了一个名为 ADLOD6FanSpeedInfo 的结构体，用于存储 Overdrive 6  fanspeed 的信息。这个结构体包含三个整型变量：iSpeedType，iFanSpeedPercent 和 iFanSpeedRPM，分别表示 fan speed 的类型、当前 fan speed 在百分比中的值和当前 fan speed 在 RPM 中的值。另外，这个结构体还包括两个用于标记是否使用了 future extension 的整型变量：iExtValue 和 iExtMask。

ADLOD6FanSpeedInfo 的结构体可以在程序中被用来存储有关 Overdrive 6  fanspeed 的信息。例如，当读取或写入 Overdrive 6 控制器时，可能需要使用这个结构体来获取或设置 fanspeed 的不同参数。


```cpp
/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about Overdrive 6 fan speed information
///
/// This structure is used to store information about Overdrive 6 fan speed information
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct _ADLOD6FanSpeedInfo
{
    /// Contains a bitmap of the valid fan speed type flags.  Possible values: \ref ADL_OD6_FANSPEED_TYPE_PERCENT, \ref ADL_OD6_FANSPEED_TYPE_RPM, \ref ADL_OD6_FANSPEED_USER_DEFINED
    int     iSpeedType;
    /// Contains current fan speed in percent (if valid flag exists in iSpeedType)
    int     iFanSpeedPercent;
    /// Contains current fan speed in RPM (if valid flag exists in iSpeedType)
    int        iFanSpeedRPM;

    /// Value for future extension
    int     iExtValue;
    /// Mask for future extension
    int     iExtMask;

} ADLOD6FanSpeedInfo;

```

这是一个定义了一个名为 ADLOD6FanSpeedValue 的结构体，用于存储 Overdrive 6  fanspeed 的信息。这个结构体有两个成员变量，分别是 iSpeedType 和 iFanSpeed，分别表示风扇速度类型（可能的值包括 ADL_OD6_FANSPEED_TYPE_PERCENT 和 ADL_OD6_FANSPEED_TYPE_RPM）和风扇速度值。此外，还有一个名为 iExtValue 的成员变量，表示风扇在未来可能变化的值，以及一个名为 iExtMask 的成员变量，表示判断未来值是否为变化的掩码。

这个结构体可以被用来在程序中读取、设置或检查Overdrive 6 风扇的速度值。


```cpp
/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about Overdrive 6 fan speed value
///
/// This structure is used to store information about Overdrive 6 fan speed value
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct _ADLOD6FanSpeedValue
{
    /// Indicates the units of the fan speed.  Possible values: \ref ADL_OD6_FANSPEED_TYPE_PERCENT, \ref ADL_OD6_FANSPEED_TYPE_RPM
    int     iSpeedType;
    /// Fan speed value (units as indicated above)
    int     iFanSpeed;

    /// Value for future extension
    int     iExtValue;
    /// Mask for future extension
    int     iExtMask;

} ADLOD6FanSpeedValue;

```

这段代码定义了一个名为 ADLOD6PowerControlInfo 的结构体，用于存储 Overdrive 6 PowerControl 的设置信息。该结构体包含以下成员：

iMinValue：设置 PowerControl 调整的最小值；
iMaxValue：设置 PowerControl 调整的最大值；
iStepValue：设置 PowerControl 调整的步长；
iExtValue：设置未来可扩展的最小值；
iExtMask：设置未来可扩展的掩码。

这个结构体用于存储 Overdrive 6 PowerControl 的设置，可以让用户通过调整 PowerTune 功率限制来改变 GPU 的性能。


```cpp
/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about Overdrive 6 PowerControl settings.
///
/// This structure is used to store information about Overdrive 6 PowerControl settings.
/// PowerControl is the feature which allows the performance characteristics of the GPU
/// to be adjusted by changing the PowerTune power limits.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct _ADLOD6PowerControlInfo
{
    /// The minimum PowerControl adjustment value
    int     iMinValue;
    /// The maximum PowerControl adjustment value
    int     iMaxValue;
    /// The minimum difference between PowerControl adjustment values
    int     iStepValue;

    /// Value for future extension
    int     iExtValue;
    /// Mask for future extension
    int     iExtMask;

} ADLOD6PowerControlInfo;

```

这段代码定义了一个名为 ADLOD6VoltageControlInfo 的结构体，用于存储 Overdrive 6 电源控制设置中的信息。该结构体包含以下成员：

- iMinValue：电压控制调整的最小值
- iMaxValue：电压控制调整的最大值
- iStepValue：电压控制调整的最小变化量
- iExtValue：电压控制的未来扩展值
- iExtMask：电压控制的未来扩展掩码

这个结构体可能用于存储与 Overdrive 6 电源控制相关的设置信息，以便用户可以根据需要调整 GPU 的性能。


```cpp
/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about Overdrive 6 PowerControl settings.
///
/// This structure is used to store information about Overdrive 6 PowerControl settings.
/// PowerControl is the feature which allows the performance characteristics of the GPU
/// to be adjusted by changing the PowerTune power limits.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct _ADLOD6VoltageControlInfo
{
    /// The minimum VoltageControl adjustment value
    int     iMinValue;
    /// The maximum VoltageControl adjustment value
    int     iMaxValue;
    /// The minimum difference between VoltageControl adjustment values
    int     iStepValue;

    /// Value for future extension
    int     iExtValue;
    /// Mask for future extension
    int     iExtMask;

} ADLOD6VoltageControlInfo;

```

这段代码定义了一个名为 ADLECCData 的结构体类型，包含了 SEC（Single Error Count）和 DED（Doubt Error Detect）统计数据。

ADLECCData 结构体定义了两个整型变量 iSec 和 iDed，分别表示单错误计数和双错误检测计数。

结构体成员的定义是为了支持和各种编程语言和库的规范，以便于代码可读性、可维护性和跨平台兼容性。


```cpp
////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing ECC statistics namely SEC counts and DED counts
/// Single error count - count of errors that can be corrected
/// Doubt Error Detect -  count of errors that cannot be corrected
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct _ADLECCData
{
    // Single error count - count of errors that can be corrected
    int iSec;
    // Double error detect - count of errors that cannot be corrected
    int iDed;

} ADLECCData;




```

这段代码定义了一个名为ADL_CONTEXT_HANDLE的结构体，用于表示ADL客户端上下文处理程序的句柄。这个句柄是在ADL2_Main_Control_Create函数返回后获得的，客户端需要将这个句柄传递给后续的ADL调用，并在ADL2_Main_Control_Destroy函数返回前销毁这个句柄。

同时，该结构体还包含一个名为ADLDisplayModeX2的成员，用于表示每个控制器使用的显示模式定义。


```cpp
/// \brief Handle to ADL client context.
///
///  ADL clients obtain context handle from initial call to \ref ADL2_Main_Control_Create.
///  Clients have to pass the handle to each subsequent ADL call and finally destroy
///  the context with call to \ref ADL2_Main_Control_Destroy
/// \nosubgrouping
typedef void *ADL_CONTEXT_HANDLE;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing the display mode definition used per controller.
///
/// This structure is used to store the display mode definition used per controller.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLDisplayModeX2
{
```

这是一个定义了Overdrive 6扩展功能的数据结构。该数据结构包含以下字段：

iWidth：水平分辨率（以像素为单位）。
iHeight：垂直分辨率（以行为单位）。
iScanType：扫描类型，分为ADL和Progressive两种类型。
iRefreshRate：刷新率，用于在显示与打印之间同步更新。
iTimingStandard：计时标准，包括OnPush、OnOff和Off。

此代码定义了一个名为ADLDisplayModeX2的结构体，用于表示Overdrive 6扩展的显示模式。通过这个结构体，可以方便地在不同的Overdrive 6扩展之间进行同步更新。


```cpp
/// Horizontal resolution (in pixels).
   int  iWidth;
/// Vertical resolution (in lines).
   int  iHeight;
/// Interlaced/Progressive. The value will be set for Interlaced as ADL_DL_TIMINGFLAG_INTERLACED. If not set it is progressive. Refer define_detailed_timing_flags.
   int  iScanType;
/// Refresh rate.
   int  iRefreshRate;
/// Timing Standard. Refer define_modetiming_standard.
   int  iTimingStandard;
} ADLDisplayModeX2;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about Overdrive 6 extension capabilities
///
```

This is a C++ structure that contains information about the Overdrive 6 extension capabilities. It has several fields, including `iCapabilities`, `iSupportedStates`, `sEngineClockPercent`, `sMemoryClockPercent`, `sPowerControlPercent`, and `iExtValue`.

The `iCapabilities` field contains a bitmap of the OD6 extension capability flags. The possible values for these flags are `ADL_OD6_CAPABILITY_SCLK_CUSTOMIZATION`, `ADL_OD6_CAPABILITY_MCLK_CUSTOMIZATION`, `ADL_OD6_CAPABILITY_GPU_ACTIVITY_MONITOR`, and `ADL_OD6_CAPABILITY_POWER_CONTROL`, `ADL_OD6_CAPABILITY_VOLTAGE_CONTROL`, and `ADL_OD6_CAPABILITY_PERCENT_ADJUSTMENT`.

The `iSupportedStates` field contains the current state of the OD6 extension capabilities. The possible values for this field are `ADL_OD6_SUPPORTEDSTATE_PERFORMANCE`.

The `sEngineClockPercent` and `sMemoryClockPercent` fields contain the limits of the SCLK and MCLK overdrive adjustment ranges, respectively. These limits should not be adjusted outside of the specified range.

The `sPowerControlPercent` field contains the limits of the Power Limit adjustment range. This field is currently only used for performance state, and should not be adjusted outside of its specified range.

The `iExtValue` and `iExtMask` fields are reserved for future expansion of the structure.

Overall, this structure provides a compact representation of the information needed to control OD6 extension capabilities in Overdrive 6.


```cpp
/// This structure is used to store information about Overdrive 6 extension capabilities
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct _ADLOD6CapabilitiesEx
{
    /// Contains a bitmap of the OD6 extension capability flags.  Possible values: \ref ADL_OD6_CAPABILITY_SCLK_CUSTOMIZATION,
    /// \ref ADL_OD6_CAPABILITY_MCLK_CUSTOMIZATION, \ref ADL_OD6_CAPABILITY_GPU_ACTIVITY_MONITOR,
    /// \ref ADL_OD6_CAPABILITY_POWER_CONTROL, \ref ADL_OD6_CAPABILITY_VOLTAGE_CONTROL, \ref ADL_OD6_CAPABILITY_PERCENT_ADJUSTMENT,
    //// \ref ADL_OD6_CAPABILITY_THERMAL_LIMIT_UNLOCK
    int iCapabilities;
    /// The Power states that support clock and power customization.  Only performance state is currently supported.
    /// Possible Values: \ref ADL_OD6_SUPPORTEDSTATE_PERFORMANCE
    int iSupportedStates;
    /// Returns the hard limits of the SCLK overdrive adjustment range.  Overdrive clocks should not be adjusted outside of this range.  The values are specified as +/- percentages.
    ADLOD6ParameterRange sEngineClockPercent;
    /// Returns the hard limits of the MCLK overdrive adjustment range.  Overdrive clocks should not be adjusted outside of this range.  The values are specified as +/- percentages.
    ADLOD6ParameterRange sMemoryClockPercent;
    /// Returns the hard limits of the Power Limit adjustment range.  Power limit should not be adjusted outside this range.  The values are specified as +/- percentages.
    ADLOD6ParameterRange sPowerControlPercent;
    /// Reserved for future expansion of the structure.
    int iExtValue;
    /// Reserved for future expansion of the structure.
    int iExtMask;
} ADLOD6CapabilitiesEx;

```

这是一段C语言代码，定义了一个名为ADLOD6StateEx的结构体，用于存储Overdrive 6扩展状态信息。这个结构体包含以下字段：

1. iEngineClockPercent：当前发动机时钟调整值，以百分比表示。
2. iMemoryClockPercent：当前内存时钟调整值，以百分比表示。
3. iPowerControlPercent：当前电源控制调整值，以百分比表示。
4. iExtValue：保留，用于未来的结构扩展。
5. iExtMask：保留，用于未来的结构扩展。

这个结构体可以被用来在程序中记录和操作Overdrive 6扩展状态信息。


```cpp
/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about Overdrive 6 extension state information
///
/// This structure is used to store information about Overdrive 6 extension state information
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct _ADLOD6StateEx
{
    /// The current engine clock adjustment value, specified as a +/- percent.
    int iEngineClockPercent;
    /// The current memory clock adjustment value, specified as a +/- percent.
    int iMemoryClockPercent;
    /// The current power control adjustment value, specified as a +/- percent.
    int iPowerControlPercent;
    /// Reserved for future expansion of the structure.
    int iExtValue;
    /// Reserved for future expansion of the structure.
    int iExtMask;
} ADLOD6StateEx;

```

这段代码定义了一个名为`ADLOD6MaxClockAdjust`的结构体，用于存储Overdrive 6扩展推荐的最大钟调整值。这个结构体包含以下成员：

1. `iEngineClockMax`：建议的最大引擎钟调整值，以指定的功率限制值为基准。
2. `iMemoryClockMax`：建议的最大内存钟调整值。请注意，这个字段目前是独立的，未来如果需要，我们可以通过比较`iEngineClockMax`和`iMemoryClockMax`的值来决定如何设置内存的最大值。
3. `iExtValue`：建议的扩展最大值，暂时没有使用。
4. `iExtMask`：用于将来扩展结构的保留字段。

这个结构体可以用来定义一个描述Overdrive 6扩展的最大钟调整值的变量或结构体，以便对不同部件的钟调整进行统一管理。


```cpp
/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about Overdrive 6 extension recommended maximum clock adjustment values
///
/// This structure is used to store information about Overdrive 6 extension recommended maximum clock adjustment values
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct _ADLOD6MaxClockAdjust
{
    /// The recommended maximum engine clock adjustment in percent, for the specified power limit value.
    int iEngineClockMax;
    /// The recommended maximum memory clock adjustment in percent, for the specified power limit value.
    /// Currently the memory is independent of the Power Limit setting, so iMemoryClockMax will always return the maximum
    /// possible adjustment value.  This field is here for future enhancement in case we add a dependency between Memory Clock
    /// adjustment and Power Limit setting.
    int iMemoryClockMax;
    /// Reserved for future expansion of the structure.
    int iExtValue;
    /// Reserved for future expansion of the structure.
    int iExtMask;
} ADLOD6MaxClockAdjust;

```

这段代码定义了一个名为ADLConnectorInfo的结构体，用于包含连接器信息，包括连接器的编号、位置、类型等。这个结构体是一个模板类，可以用来定义连接器的各种属性。

ADLConnectorInfo类型的变量可以被用来声明变量、数组或指针，从而可以用来保存连接器信息。例如，可以在一个数组中声明一个ADLConnectorInfo类型的变量，用来存储连接器的各项属性。在声明变量之后，就可以对这个变量进行使用了，例如读取或修改连接器信息。


```cpp
////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing the Connector information
///
/// this structure is used to get the connector information like length, positions & etc.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLConnectorInfo
{
    ///index of the connector(0-based)
    int iConnectorIndex;
    ///used for disply identification/ordering
    int iConnectorId;
    ///index of the slot, 0-based index.
    int iSlotIndex;
    ///Type of the connector. \ref define_connector_types
    int iType;
    ///Position of the connector(in millimeters), from the right side of the slot.
    int iOffset;
    ///Length of the connector(in millimeters).
    int iLength;

} ADLConnectorInfo;

```

这段代码定义了一个名为ADLBracketSlotInfo的结构体，用于表示电路板上的插槽信息。该结构体包含三个成员变量：槽位编号、槽位长度和槽位宽度。

具体来说，该结构体提供了一个用于获取插槽信息的接口，用户可以通过该接口获取到槽位的各种信息，例如槽位的编号、长度和宽度等。该结构体可以被用于开发与电路板设计相关的应用程序。


```cpp
////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing the slot information
///
/// this structure is used to get the slot information like length of the slot, no of connectors on the slot & etc.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLBracketSlotInfo
{
    ///index of the slot, 0-based index.
    int iSlotIndex;
    ///length of the slot(in millimeters).
    int iLength;
    ///width of the slot(in millimeters).
    int iWidth;
} ADLBracketSlotInfo;

```

这是一个定义了名为 ADLMSTRad 的结构体，用于存储 MST 分支信息。该结构体中包含两个成员变量：

1. iLinkNumber：表示链接的编号，用于在分支结构中追踪该链接。
2. rad：一个字符数组，用于存储分支的相对地址，该地址 scheme 是从源 side 开始的。

这个结构体可以用于存储一个分支结构中的信息，以便在需要时进行访问和操作。


```cpp
////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing MST branch information
///
/// this structure is used to store the MST branch information
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLMSTRad
{
    ///depth of the link.
    int iLinkNumber;
    /// Relative address, address scheme starts from source side
    char rad[ADL_MAX_RAD_LINK_COUNT];
} ADLMSTRad;

////////////////////////////////////////////////////////////////////////////////////////////
```

这段代码定义了一个名为 ADLDevicePort 的结构体，用于存储连接到 ADL 的设备端口信息。这个结构体包含一个表示接口连接器的索引（ iConnectorIndex），一个表示 MSF（MST Topology）中相对的 MSA（MST Address）的指针（ aMSTRad ），以及一个用于连接到 MSF 的信号名称（信号名）。

DP（Data Port）连接器连接到 MSF 的时候，可以使用这个结构体中定义的 aMSTRad 成员来获取 MST 的地址。当 MSF 中的连接器不是 DP 连接器时，或者 MSF 中的连接器 RAD 为 0 时，dp 连接器将忽略 MST 地址。


```cpp
///\brief Structure containing port information
///
/// this structure is used to get the display or MST branch information
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLDevicePort
{
    ///index of the connector.
    int iConnectorIndex;
    ///Relative MST address. If MST RAD contains 0 it means DP or Root of the MST topology. For non DP connectors MST RAD is ignored.
    ADLMSTRad aMSTRad;
} ADLDevicePort;

////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing supported connection types and properties
```

这段代码定义了一个名为ADLSupportedConnections的结构体，用于表示连接器支持哪些连接类型以及支持哪些属性。结构体中包含一个名为iSupportedConnections的整数，表示连接器的位串，以及一个名为iSupportedProperties的二维数组，其中每个连接类型对应一个名为iSupportedProperties的整数。

具体来说，这段代码的作用是定义一个结构体，结构体名为ADLSupportedConnections，包含连接器的类型、支持的所有属性和每个连接类型的支持属性。这个结构体可以在程序中被用来检查连接器是否支持某种连接类型，以及在支持连接类型时获取该连接类型的属性。


```cpp
///
/// this structure is used to get the supported connection types and supported properties of given connector
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLSupportedConnections
{
    ///Bit vector of supported connections. Bitmask is defined in constants section. \ref define_connection_types
    int iSupportedConnections;
    ///Array of bitvectors. Each bit vector represents supported properties for one connection type. Index of this array is connection type (bit number in mask).
    int iSupportedProperties[ADL_MAX_CONNECTION_TYPES];
} ADLSupportedConnections;

////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing connection state of the connector
///
```

这段代码定义了一个名为ADLConnectionState的结构体，用于获取给定连接器当前的仿真状态和模式。这个结构体有以下几个成员：

1. iEmulationStatus：表示连接器的仿真状态，是一个整数，每个比特表示不同的状态，具体定义在mask_constants中。
2. iEmulationMode：表示连接器的当前仿真模式，也是一个整数，具体定义在constants中。
3. iDisplayIndex：表示连接器当前显示的ID，如果连接器活动，则此成员会包含显示ID，否则为CWDDEDI_INVALID_DISPLAY_INDEX。

结构体定义后，没有其他操作，所以它的作用是获取连接器的状态和模式，并输出给用户。


```cpp
/// this structure is used to get the current Emulation status and mode of the given connector
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLConnectionState
{
    ///The value is bit vector. Each bit represents status. See masks constants for details. \ref define_emulation_status
    int iEmulationStatus;
    ///It contains information about current emulation mode. See constants for details. \ref define_emulation_mode
    int iEmulationMode;
    ///If connection is active it will contain display id, otherwise CWDDEDI_INVALID_DISPLAY_INDEX
    int iDisplayIndex;
} ADLConnectionState;


////////////////////////////////////////////////////////////////////////////////////////////
```

这段代码定义了一个名为ADLConnectionProperties的结构体，用于表示连接类型的属性信息。这个结构体包含以下几个成员变量：

- iValidProperties：表示实际可用的连接属性，根据连接类型有不同的值。
- iBitrate：表示数据传输速率，可以用于MST分支、DP或DP主动吊挂。
- iNumberOfLanes：表示DP连接中的通道数。
- iColorDepth：表示彩色深度的位数，可以用于一些高端的DP连接。
- iStereo3DCaps：表示3D功能的可用性，可以是用于一些高端DP连接的比特向量。
- iOutputBandwidth：表示输出带宽，可以用于MST分支、DP或DP主动吊挂。


```cpp
///\brief Structure containing connection properties information
///
/// this structure is used to retrieve the properties of connection type
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLConnectionProperties
{
    //Bit vector. Represents actual properties. Supported properties for specific connection type. \ref define_connection_properties
    int iValidProperties;
    //Bitrate(in MHz). Could be used for MST branch, DP or DP active dongle. \ref define_linkrate_constants
    int iBitrate;
    //Number of lanes in DP connection. \ref define_lanecount_constants
    int iNumberOfLanes;
    //Color depth(in bits). \ref define_colordepth_constants
    int iColorDepth;
    //3D capabilities. It could be used for some dongles. For instance: alternate framepack. Value of this property is bit vector.
    int iStereo3DCaps;
    ///Output Bandwidth. Could be used for MST branch, DP or DP Active dongle. \ref define_linkrate_constants
    int iOutputBandwidth;
} ADLConnectionProperties;

```

这段代码定义了一个名为ADLConnectionData的结构体，用于表示与设备的数据连接信息。该结构体包含以下字段：

1. iConnectionType：表示数据连接类型，根据类型可以有不同的值，包括iNumberofPorts和IDataSize。
2. aConnectionProperties：包含连接属性，例如最大端口数、自动连接等。
3. iNumberofPorts：表示连接的端口数。
4. iActiveConnections：表示处于活动状态的连接数量。
5. iDataSize：表示EDID数据块的大小。
6. EdidData：保存EDID数据，包括校验和数据长度等。

这个结构体可以用来在应用程序中检索与设备的数据连接信息，例如在初始化或者关闭设备时检测连接状态的变化等。


```cpp
////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing connection information
///
/// this structure is used to retrieve the data from driver which includes
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLConnectionData
{
    ///Connection type. based on the connection type either iNumberofPorts or IDataSize,EDIDdata is valid, \ref define_connection_types
    int iConnectionType;
    ///Specifies the connection properties.
    ADLConnectionProperties aConnectionProperties;
    ///Number of ports
    int iNumberofPorts;
    ///Number of Active Connections
    int iActiveConnections;
    ///actual size of EDID data block size.
    int iDataSize;
    ///EDID Data
    char EdidData[ADL_MAX_DISPLAY_EDID_DATA_SIZE];
} ADLConnectionData;

```

这是一个定义了一个名为ADLAdapterCapsX2的结构体的头文件。这个结构体用于存储关于控制器模式的信息，包括适配器ID、控制器数量、显示数量、覆盖层数量、GLSyncConnectors数量以及适配器 caps的比特掩码和值。这些信息可能会用于控制器的初始化和操作。


```cpp
/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about an controller mode including Number of Connectors
///
/// This structure is used to store information of an controller mode
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLAdapterCapsX2
{
    /// AdapterID for this adapter
    int iAdapterID;
    /// Number of controllers for this adapter
    int iNumControllers;
    /// Number of displays for this adapter
    int iNumDisplays;
    /// Number of overlays for this adapter
    int iNumOverlays;
    /// Number of GLSyncConnectors
    int iNumOfGLSyncConnectors;
    /// The bit mask identifies the adapter caps
    int iCapsMask;
    /// The bit identifies the adapter caps \ref define_adapter_caps
    int iCapsValue;
    /// Number of Connectors for this adapter
    int iNumConnectors;
}ADLAdapterCapsX2;

```

这段代码定义了一个枚举类型 _ADL_ERROR_RECORD_SEVERITY 和一个联合类型 ADL_ECC_EDC_FLAG，用于表示记录链中错误记录的严重性级别和ECC(Ethernet Current)实体动作 flag。具体作用如下：

1.定义了一个枚举类型 ADL_ERROR_RECORD_SEVERITY，它定义了四种不同的错误记录严重性级别，分别为 ADL_GLOBALLY_UNCORRECTED、ADL_LOCALLY_UNCORRECTED、ADL_DEFFERED 和 ADL_CORRECTED。

2.定义了一个联合类型 ADL_ECC_EDC_FLAG，它定义了一个结构体类型的标志位 set，用于表示是否正在使用 ECC(ECC=128) 访问操作，以及保留位，用于表示更多的 ECC 访问操作。

3.该联合类型 ADL_ECC_EDC_FLAG 被定义为使用 struct 定义的联合类型。这个结构体类型包含两个成员，isEccAccessing 和 reserved,isEccAccessing 是一个 bit 类型，用于表示当前是否正在使用 ECC 访问操作，reserved 是一个 32 位类型，用于表示更多的 ECC 访问操作。

4.上述枚举类型和联合类型可以用于记录链中的错误记录。ADL_ERROR_RECORD_SEVERITY 枚举类型定义了错误记录的四个不同的严重性级别，可以用于记录链中每个 ECC 实体操作的错误记录。ADL_ECC_EDC_FLAG 联合类型可以用于标记 ECC 访问操作是否正在发生，以及记录更多的 ECC 访问操作。


```cpp
typedef enum _ADL_ERROR_RECORD_SEVERITY
{
    ADL_GLOBALLY_UNCORRECTED  = 1,
    ADL_LOCALLY_UNCORRECTED   = 2,
    ADL_DEFFERRED             = 3,
    ADL_CORRECTED             = 4
}ADL_ERROR_RECORD_SEVERITY;

typedef union _ADL_ECC_EDC_FLAG
{
    struct
    {
        unsigned int isEccAccessing        : 1;
        unsigned int reserved              : 31;
    }bits;
    unsigned int u32All;
}ADL_ECC_EDC_FLAG;

```

这是一个C++的代码，定义了一个名为ADLErrorRecord的结构体。该结构体用于存储EDC（Engine Development Command）错误记录。以下是该结构体中各个成员的含义：

1. Severity of error（错误严重性）：表示错误的严重程度，例如：ADL_ERROR_RECORD_SEVERITY_HIGH、ADL_ERROR_RECORD_SEVERITY_MEDIUM或ADL_ERROR_RECORD_SEVERITY_LOW。
2. Is the counter valid?（计数器是否有效）：表示计数器是否已经被正确初始化，如果是，则此值为1，如果不是，则此值为0。
3. Counter value, if valid（计数值，如果有效）：表示错误发生的计数器值，如果错误发生在特定的位置，则此值可能为具体的错误编号，否则为无效的计数器值。
4. Is the location information valid?（位置信息是否有效）：表示错误位置的信息是否正确，例如：1表示记录的物理位置在内存中的位置，而2表示错误发生在CPU的寄存器中。
5. Physical location of error（错误位置）：表示错误发生的物理位置，例如：CU（某个具体的CPU）编号。
6. Time of error record creation（错误记录创建的时间）：例如：时间戳或中断类型，用于追踪错误发生的时间。
7. Padding（填充字节）：用于在记录中填充字节，以保证记录的长度为4的倍数。

ADLErrorRecord结构体用于存储EDC错误记录的相关信息，以便进行记录、处理和追踪。


```cpp
/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about EDC Error Record
///
/// This structure is used to store EDC Error Record
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLErrorRecord
{
    // Severity of error
    ADL_ERROR_RECORD_SEVERITY Severity;

    // Is the counter valid?
    int  countValid;

    // Counter value, if valid
    unsigned int count;

    // Is the location information valid?
    int locationValid;

    // Physical location of error
    unsigned int CU; // CU number on which error occurred, if known
    char StructureName[32]; // e.g. LDS, TCC, etc.

    // Time of error record creation (e.g. time of query, or time of poison interrupt)
    char tiestamp[32];

    unsigned int padding[3];
}ADLErrorRecord;

```

这段代码定义了一个枚举类型 _ADL_EDC_BLOCK_ID 和一个枚举类型 _ADL_ERROR_INJECTION_MODE。

_ADL_EDC_BLOCK_ID 枚举类型定义了 8 个枚举成员，分别对应于 ADL_EDC_BLOCK_ID_SQCIS、ADL_EDC_BLOCK_ID_SQCDS、ADL_EDC_BLOCK_ID_SGPR、ADL_EDC_BLOCK_ID_VGPR、ADL_EDC_BLOCK_ID_LDS、ADL_EDC_BLOCK_ID_GDS 和 ADL_EDC_BLOCK_ID_TCL1。

_ADL_ERROR_INJECTION_MODE 枚举类型定义了 3 个枚举成员，分别为 ADL_ERROR_INJECTION_MODE_SINGLE、ADL_ERROR_INJECTION_MODE_MULTIPLE 和 ADL_ERROR_INJECTION_MODE_ADDRESS。


```cpp
typedef enum _ADL_EDC_BLOCK_ID
{
    ADL_EDC_BLOCK_ID_SQCIS = 1,
    ADL_EDC_BLOCK_ID_SQCDS = 2,
    ADL_EDC_BLOCK_ID_SGPR  = 3,
    ADL_EDC_BLOCK_ID_VGPR  = 4,
    ADL_EDC_BLOCK_ID_LDS   = 5,
    ADL_EDC_BLOCK_ID_GDS   = 6,
    ADL_EDC_BLOCK_ID_TCL1  = 7,
    ADL_EDC_BLOCK_ID_TCL2  = 8
}ADL_EDC_BLOCK_ID;

typedef enum _ADL_ERROR_INJECTION_MODE
{
    ADL_ERROR_INJECTION_MODE_SINGLE      = 1,
    ADL_ERROR_INJECTION_MODE_MULTIPLE    = 2,
    ADL_ERROR_INJECTION_MODE_ADDRESS     = 3
}ADL_ERROR_INJECTION_MODE;

```

这段代码定义了一个名为ADL_ERROR_PATTERN的结构体，表示一个错误模式。这个结构体定义了以下字段：

- `EccInjVector`:16位二进制数，表示ECC(Error-Compare-Compensating)指令的惩罚值。
- `EccInjEn`:9位二进制数，表示ECC指令是否啟用。
- `EccBeatEn`:4位二进制数，表示比較轮是否執行。
- `EccChEn`:4位二进制数，表示讀取/寫入/執行(R/W/E)操作開始/結束。
- `reserved`:31位保留字段，用於保留未來的用途。

此外，還定義了一個名為ADL_ERROR_INJECTION_DATA的结构体，用於表示一個錯誤注入的信息。該結構体包含兩個字段：

- `errorAddress`:long long字段，表示錯誤發生的地址。
- `errorPattern`:ADL_ERROR_PATTERN結構体，包含了上面定義的ADL_ERROR_PATTERN結構体。

該程式定義了一個用於錯誤注入的API，並且使用該API對系統進行測試。錯誤注入的信息被存儲在`errorAddress`和`errorPattern`字段中。如果錯誤發生，這些字段將包含系統錯誤的地址和模式。


```cpp
typedef union _ADL_ERROR_PATTERN
{
    struct
    {
        unsigned long  EccInjVector         :  16;
        unsigned long  EccInjEn             :  9;
        unsigned long  EccBeatEn            :  4;
        unsigned long  EccChEn              :  4;
        unsigned long  reserved             :  31;
    } bits;
    unsigned long long u64Value;
} ADL_ERROR_PATTERN;

typedef struct _ADL_ERROR_INJECTION_DATA
{
    unsigned long long errorAddress;
    ADL_ERROR_PATTERN errorPattern;
}ADL_ERROR_INJECTION_DATA;

```

这是一个C++的结构体定义，名为ADLErrorInjection。这个结构体用于存储EDC（electronic control unit）错误注入的信息。

ADLErrorInjection包含两个成员变量：

1. 类型为ADL_EDC_BLOCK_ID的blockId，表示EDC块的ID。
2. 类型为ADL_ERROR_INJECTION_MODE的errorInjectionMode，表示错误注入的模式，可能为0或1。

此外，该结构体还包含一个名为errorInjectionData的成员变量，但该成员变量并未定义具体的数据类型。


```cpp
/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about EDC Error Injection
///
/// This structure is used to store EDC Error Injection
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLErrorInjection
{
    ADL_EDC_BLOCK_ID blockId;
    ADL_ERROR_INJECTION_MODE errorInjectionMode;
}ADLErrorInjection;

typedef struct ADLErrorInjectionX2
{
    ADL_EDC_BLOCK_ID blockId;
    ADL_ERROR_INJECTION_MODE errorInjectionMode;
    ADL_ERROR_INJECTION_DATA errorInjectionData;
}ADLErrorInjectionX2;

```

这是一个定义了一个名为 ADLFreeSyncCap 的结构体，用于存储显示与 GPU 之间的 FreeSync 功能信息。这个结构体包含两个整型变量 iCaps 和 iMinRefreshRateInMicroHz，分别表示显示支持的 FreeSync 功能 flag 和最小 FreeSync 刷新率。此外，还有一个包含五个保留整型的 iReserved 成员，用于存储将来可能需要填写的保留字段。


```cpp
/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing per display FreeSync capability information.
///
/// This structure is used to store the FreeSync capability of both the display and
/// the GPU the display is connected to.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLFreeSyncCap
{
    /// FreeSync capability flags. \ref define_freesync_caps
    int iCaps;
    /// Reports minimum FreeSync refresh rate supported by the display in micro hertz
    int iMinRefreshRateInMicroHz;
    /// Reports maximum FreeSync refresh rate supported by the display in micro hertz
    int iMaxRefreshRateInMicroHz;
    /// Reserved
    int iReserved[5];
} ADLFreeSyncCap;

```

这段代码定义了一个名为ADLDceSettings的结构体，用于存储显示的显示连接体验设置。该结构体包含两个部分：显示连接设置和保护设置。

显示连接设置部分，定义了HDMI质量检测是否启用，以及DP链接率和总 lanes数等设置。这里的"\nosubgrouping"表示这是一个多包合并结构。

保护设置部分，定义了DP链接保护是否启用，以及一些保护设置，如相对预放大和电压摆动等。

整结构体定义了两个指针变量，iReserved[15]和Settings，iReserved[15]用于保存其他特定用量的数据，而Settings则用于显示连接体验设置的存储。


```cpp
/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing per display Display Connectivty Experience Settings
///
/// This structure is used to store the Display Connectivity Experience settings of a
/// display
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct _ADLDceSettings
{
    DceSettingsType type;                       // Defines which structure is in the union below
    union
    {
        struct
        {
            bool qualityDetectionEnabled;
        } HdmiLq;
        struct
        {
            DpLinkRate linkRate;                // Read-only
            unsigned int numberOfActiveLanes;   // Read-only
            unsigned int numberofTotalLanes;    // Read-only
            int relativePreEmphasis;            // Allowable values are -2 to +2
            int relativeVoltageSwing;           // Allowable values are -2 to +2
            int persistFlag;
        } DpLink;
        struct
        {
            bool linkProtectionEnabled;         // Read-only
        } Protection;
    } Settings;
    int iReserved[15];
} ADLDceSettings;

```

这段代码定义了一个名为ADLGraphicCoreInfo的结构体，用于获取图形核心信息。这个结构体包含以下几个Field：

1. iGCGen：表示图形核心的生成级别，分为GCN和RDNA两种情况。
2. iNumCUs：表示每个CPU能处理的最大线程数，只对GCN有效。
3. iNumWGPs：表示每个WGP能处理的最大线程数，只对RDNA有效。
4. iNumPEsPerCU：表示每个CPU对每个CU能处理的最大线程数，只对GCN有效。
5. iNumPEsPerWGP：表示每个WGP对每个WGP能处理的最大线程数，只对RDNA有效。
6. iNumSIMDs：表示每个SIMD能处理的最大线程数，只对GCN有效。
7. iNumROPs：表示每个ROP能处理的最大线程数，对GCN和RDNA都有效。
8. iReserved：保留空间，后期可能被填充满用。

这个结构体在图形处理器中用于获取图形核心信息，包括CPU、线程、线程以内的WGPs、SIMDs和ROPs等。通过这个结构体，开发人员可以更方便地获取和操作图形核心的信息，从而优化图形应用程序的性能。


```cpp
/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about Graphic Core
///
/// This structure is used to get Graphic Core Info
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLGraphicCoreInfo
{
    /// indicate the graphic core generation
    int iGCGen;

    union
    {
        /// Total number of CUs. Valid for GCN (iGCGen == GCN)
        int iNumCUs;
        /// Total number of WGPs. Valid for RDNA (iGCGen == RDNA)
        int iNumWGPs;
    };

    union
    {
        /// Number of processing elements per CU. Valid for GCN (iGCGen == GCN)
        int iNumPEsPerCU;
        /// Number of processing elements per WGP. Valid for RDNA (iGCGen == RDNA)
        int iNumPEsPerWGP;
    };

    /// Total number of SIMDs. Valid for Pre GCN (iGCGen == Pre-GCN)
    int iNumSIMDs;

    /// Total number of ROPs. Valid for both GCN and Pre GCN
    int iNumROPs;

    /// reserved for future use
    int iReserved[11];
}ADLGraphicCoreInfo;


```

这是一个定义了一个名为`ADLODNParameterRange`的结构体的定义。这个结构体用于存储关于Overdrive N时钟范围的信息。

这个结构体包含以下字段：

* `iMode`：时钟范围的的模式，0表示以1Hz为单位的模式，1表示以50Hz为单位的模式。
* `iMin`：时钟范围的最小值。
* `iMax`：时钟范围的最大值。
* `iStep`：时钟值的最小增量。
* `iDefault`：时钟范围的默认值。

这个结构体在代码中可能被用来定义一个`ADLODNParameterRange`类型的变量，这个变量可以用来设置Overdrive N时钟范围的大小和参数。


```cpp
/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about Overdrive N clock range
///
/// This structure is used to store information about Overdrive N clock range
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct _ADLODNParameterRange
{
    /// The starting value of the clock range
    int     iMode;
    /// The starting value of the clock range
    int     iMin;
    /// The ending value of the clock range
    int     iMax;
    /// The minimum increment between clock values
    int     iStep;
    /// The default clock values
    int     iDefault;

} ADLODNParameterRange;

```

这段代码定义了一个名为 ADLODNCapabilities 的结构体，用于存储关于 Overdrive N 功能的最低到最高时钟范围的信息。这个结构体包含了一些整型变量，如 `iMaximumNumberOfPerformanceLevels`，表示时钟级别的数量，以及每个级别的最高和最低时钟范围。

此外，这个结构体还包含了一些只读的 ADLODNParameterRange 类型的变量，表示每个时钟范围的硬件限制。这些变量包括 `sEngineClockRange`，`sMemoryClockRange`，`svddcRange`，`power` 和 `powerTuneTemperature`，分别表示发动机、内存、电压驱动器和温度调节系统的最高和最低时钟范围。

最后，这个结构体还包括一些只读的 ADLODNParameterRange 类型的变量，表示风扇的温度和速度限制。


```cpp
/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about Overdrive N capabilities
///
/// This structure is used to store information about Overdrive N capabilities
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct _ADLODNCapabilities
{
    /// Number of levels which describe the minimum to maximum clock ranges.
    /// The 1st level indicates the minimum clocks, and the 2nd level
    /// indicates the maximum clocks.
    int     iMaximumNumberOfPerformanceLevels;
    /// Contains the hard limits of the sclk range.  Overdrive
    /// clocks cannot be set outside this range.
    ADLODNParameterRange     sEngineClockRange;
    /// Contains the hard limits of the mclk range.  Overdrive
    /// clocks cannot be set outside this range.
    ADLODNParameterRange     sMemoryClockRange;
    /// Contains the hard limits of the vddc range.  Overdrive
    /// clocks cannot be set outside this range.
    ADLODNParameterRange     svddcRange;
    /// Contains the hard limits of the power range.  Overdrive
    /// clocks cannot be set outside this range.
    ADLODNParameterRange     power;
    /// Contains the hard limits of the power range.  Overdrive
    /// clocks cannot be set outside this range.
    ADLODNParameterRange     powerTuneTemperature;
    /// Contains the hard limits of the Temperature range.  Overdrive
    /// clocks cannot be set outside this range.
    ADLODNParameterRange     fanTemperature;
    /// Contains the hard limits of the Fan range.  Overdrive
    /// clocks cannot be set outside this range.
    ADLODNParameterRange     fanSpeed;
    /// Contains the hard limits of the Fan range.  Overdrive
    /// clocks cannot be set outside this range.
    ADLODNParameterRange     minimumPerformanceClock;
} ADLODNCapabilities;

```

TheADLODNCapabilitiesX2 structure contains the minimum and maximum clock ranges for each performance level.

The structure has a single int iMaximumNumberOfPerformanceLevels, which indicates the number of performance levels that can be set.

Each performance level is represented by an array of three ints: iFlags, iMaximumNumberOfPerformanceLevels, and a bit vector indicating which features are supported.

The performance levels control the range of clock rates that can be used for different functionalities in the ADLODN, such as engine, memory, and power.

For example, the minimum and maximum clock ranges for the engine performance level may be set to 0, while the maximum clock range for the memory performance level may be set to 200.

The structure also contains the hard limits of the different performance levels, such as the sclk, mclk, vddc, and power ranges.

Overall, theADLODNCapabilitiesX2 structure provides a structure for specifying the minimum and maximum clock ranges for each performance level in the ADLODN.



```cpp
/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about Overdrive N capabilities
///
/// This structure is used to store information about Overdrive N capabilities
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct _ADLODNCapabilitiesX2
{
    /// Number of levels which describe the minimum to maximum clock ranges.
    /// The 1st level indicates the minimum clocks, and the 2nd level
    /// indicates the maximum clocks.
    int     iMaximumNumberOfPerformanceLevels;
    /// bit vector, which tells what are the features are supported.
    /// \ref: ADLODNFEATURECONTROL
    int iFlags;
    /// Contains the hard limits of the sclk range.  Overdrive
    /// clocks cannot be set outside this range.
    ADLODNParameterRange     sEngineClockRange;
    /// Contains the hard limits of the mclk range.  Overdrive
    /// clocks cannot be set outside this range.
    ADLODNParameterRange     sMemoryClockRange;
    /// Contains the hard limits of the vddc range.  Overdrive
    /// clocks cannot be set outside this range.
    ADLODNParameterRange     svddcRange;
    /// Contains the hard limits of the power range.  Overdrive
    /// clocks cannot be set outside this range.
    ADLODNParameterRange     power;
    /// Contains the hard limits of the power range.  Overdrive
    /// clocks cannot be set outside this range.
    ADLODNParameterRange     powerTuneTemperature;
    /// Contains the hard limits of the Temperature range.  Overdrive
    /// clocks cannot be set outside this range.
    ADLODNParameterRange     fanTemperature;
    /// Contains the hard limits of the Fan range.  Overdrive
    /// clocks cannot be set outside this range.
    ADLODNParameterRange     fanSpeed;
    /// Contains the hard limits of the Fan range.  Overdrive
    /// clocks cannot be set outside this range.
    ADLODNParameterRange     minimumPerformanceClock;
    /// Contains the hard limits of the throttleNotification
    ADLODNParameterRange throttleNotificaion;
    /// Contains the hard limits of the Auto Systemclock
    ADLODNParameterRange autoSystemClock;
} ADLODNCapabilitiesX2;

```

这是一个定义了一个名为ADLODNPerformanceLevel的结构体的函数。这个结构体包含关于Overdrive level的信息，并用于存储ADLOD Performance Levels。

这个函数的作用是定义一个名为ADLODNPerformanceLevel的结构体，该结构体包含关于Overdrive level的信息。这个结构体将用于ADLOD Performance Levels的存储和操作。


```cpp
/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about Overdrive level.
///
/// This structure is used to store information about Overdrive level.
/// This structure is used by ADLODPerformanceLevels.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLODNPerformanceLevel
{
    /// clock.
    int iClock;
    /// VDCC.
    int iVddc;
    /// enabled
    int iEnabled;
} ADLODNPerformanceLevel;

```

这段代码定义了一个名为 ADLODNPerformanceLevels 的结构体，用于存储 Overdrive N 的性能级别信息。该结构体包含以下成员：

1. iSize：结构体大小。
2. iMode：自动/手动模式，用于控制数据传输模式。
3. iNumberOfPerformanceLevels：性能级别数量。
4. iLevels：性能级别状态描述符的数组，大小为 ADLODPerformanceLevels 的大小，即  sizeof(ADLODPerformanceLevels)*(ADLODParameters.iNumberOfPerformanceLevels - 1)。

此外，还定义了一个名为 ADLODPerformanceLevels 的结构体，用于存储 ADLOD 性能级别信息。

最后，该代码未输出任何函数，但为 ADL_OverdriveN_ODPerformanceLevels_Get() 和 ADL_OverdriveN_ODPerformanceLevels_Set() 函数提供了输入输出接口。


```cpp
/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about Overdrive N performance levels.
///
/// This structure is used to store information about Overdrive performance levels.
/// This structure is used by the ADL_OverdriveN_ODPerformanceLevels_Get() and ADL_OverdriveN_ODPerformanceLevels_Set() functions.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLODNPerformanceLevels
{
    int iSize;
    //Automatic/manual
    int iMode;
    /// Must be set to sizeof( \ref ADLODPerformanceLevels ) + sizeof( \ref ADLODPerformanceLevel ) * (ADLODParameters.iNumberOfPerformanceLevels - 1)
    int iNumberOfPerformanceLevels;
    /// Array of performance state descriptors. Must have ADLODParameters.iNumberOfPerformanceLevels elements.
    ADLODNPerformanceLevel aLevels[1];
} ADLODNPerformanceLevels;

```

这是一个名为 ADLODNFanControl 的结构体，用于存储关于 Overdrive N 风扇控制的信息。这个结构体被用于函数 ADL_OverdriveN_ODPerformanceLevels_Get 和 ADL_OverdriveN_ODPerformanceLevels_Set。

这个结构体包含以下成员：

* iMode：风扇控制模式，可以是 0 或 1。
* iFanControlMode：风扇控制模式，可以是 0 或 1。
* iCurrentFanSpeedMode：当前风扇速度控制模式，可以是 0 或 1。
* iCurrentFanSpeed：当前风扇速度，可以是 0 到 63535 之间的整数。
* iTargetFanSpeed：目标风扇速度，可以是 0 到 63535 之间的整数。
* iTargetTemperature：目标风扇温度，可以是 0 到 63535 之间的整数。
* iMinPerformanceClock：最小性能时钟，表示在多少毫秒内，风扇可以达到目标速度。
* iMinFanLimit：最小风扇速度限制，如果目标风扇速度低于该限制，风扇将停机。

这些成员定义了一个名为 ADLODNFanControl 的结构体，它可以在函数中用于存储和操作 Overdrive N 风扇控制的信息。


```cpp
/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about Overdrive N Fan Speed.
///
/// This structure is used to store information about Overdrive Fan control .
/// This structure is used by the ADL_OverdriveN_ODPerformanceLevels_Get() and ADL_OverdriveN_ODPerformanceLevels_Set() functions.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLODNFanControl
{
    int iMode;
    int iFanControlMode;
    int iCurrentFanSpeedMode;
    int iCurrentFanSpeed;
    int iTargetFanSpeed;
    int iTargetTemperature;
    int iMinPerformanceClock;
    int iMinFanLimit;
} ADLODNFanControl;

```

这段代码定义了一个名为 ADLODNPowerLimitSetting 的结构体，用于存储关于 Overdrive N 电源限制的信息。这个结构体被用于两个函数：ADL_OverdriveN_ODPerformanceLevels_Get 和 ADL_OverdriveN_ODPerformanceLevels_Set。

ADLODNPowerLimitSetting 结构体包含以下成员：

- iMode：电源管理模式，可以是 0 或 1。
- iTDPLimit：TDP 限制，可以是 0 或 4。
- iMaxOperatingTemperature：允许的最高运行温度，单位为摄氏度。

ADLODNPerformanceStatus 结构体包含以下成员：

- iCoreClock：CPU 核心时钟，单位为兆时钟（MHz）。
- iMemoryClock：内存时钟，单位为兆时钟（MHz）。
- iDCEFClock：DCEF 时钟，单位为兆时钟（MHz）。
- iGFXClock：GFX 时钟，单位为兆时钟（MHz）。
- iUVDClock：UVD 时钟，单位为兆时钟（MHz）。
- iVCEClock：VCEC 时钟，单位为兆时钟（MHz）。
- iGPUActivityPercent：GPU 活动百分比，范围为 0 到 100。
- iCurrentCorePerformanceLevel：当前核心性能级别，范围为 0 到 100。
- iCurrentMemoryPerformanceLevel：当前内存性能级别，范围为 0 到 100。
- iCurrentDCEFPerformanceLevel：当前 DCEF 性能级别，范围为 0 到 100。
- iCurrentGFXPerformanceLevel：当前 GFX 性能级别，范围为 0 到 100。
- iUVDPerformanceLevel：当前 UVD 性能级别，范围为 0 到 100。
- iVCEPerformanceLevel：当前 VCE 性能级别，范围为 0 到 100。
- iCurrentBusSpeed：当前总线速度，单位为 GB/s。
- iCurrentBusLanes：当前总线带宽，单位为 GB。
- iMaximumBusLanes：总线带宽上限，单位为 GB。
- iVDDC：默认的电源类型，可以是 0 或 1。
- iVDDCI：电源类型为 0 时使用的电源电流，可以是 0 或 4。


```cpp
/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about Overdrive N power limit.
///
/// This structure is used to store information about Overdrive power limit.
/// This structure is used by the ADL_OverdriveN_ODPerformanceLevels_Get() and ADL_OverdriveN_ODPerformanceLevels_Set() functions.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLODNPowerLimitSetting
{
    int iMode;
    int iTDPLimit;
    int iMaxOperatingTemperature;
} ADLODNPowerLimitSetting;

typedef struct ADLODNPerformanceStatus
{
    int iCoreClock;
    int iMemoryClock;
    int iDCEFClock;
    int iGFXClock;
    int iUVDClock;
    int iVCEClock;
    int iGPUActivityPercent;
    int iCurrentCorePerformanceLevel;
    int iCurrentMemoryPerformanceLevel;
    int iCurrentDCEFPerformanceLevel;
    int iCurrentGFXPerformanceLevel;
    int iUVDPerformanceLevel;
    int iVCEPerformanceLevel;
    int iCurrentBusSpeed;
    int iCurrentBusLanes;
    int iMaximumBusLanes;
    int iVDDC;
    int iVDDCI;
} ADLODNPerformanceStatus;

```

这段代码定义了一个名为ADLODNPerformanceLevelX2的结构体，用于存储Overdrive level的信息。这个结构体包含以下成员：

1. iClock：时钟，用于同步数据传输。
2. iVddc：VDDC，用于驱动器供电。
3. iEnabled：是否启用，用于设置是否要启用VDDC。
4. iControl：控制位，用于控制同步。

这个结构体可以被用于ADLODPerformanceLevels，根据需要，可以对其进行不同的使用。


```cpp
///\brief Structure containing information about Overdrive level.
///
/// This structure is used to store information about Overdrive level.
/// This structure is used by ADLODPerformanceLevels.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLODNPerformanceLevelX2
{
    /// clock.
    int iClock;
    /// VDCC.
    int iVddc;
    /// enabled
    int iEnabled;
    /// MASK
    int iControl;
} ADLODNPerformanceLevelX2;

```

这段代码定义了一个名为ADLODNPerformanceLevelsX2的结构体，用于存储Overdrive N的性能级别信息。该结构体包含以下成员：

1. iSize：存储结构体大小。
2. iMode：用于指定自动或手动模式，其中iSize成员用于确保正确设置。
3. iNumberOfPerformanceLevels：用于存储Overdrive N性能级别数量。该成员必须根据ADLODPerformanceLevels结构体中iNumberOfPerformanceLevels成员的值进行计算。
4. iLevels：存储性能级别状态描述符的数组。这个数组必须与ADLODPerformanceLevels结构体中iNumberOfPerformanceLevels成员的值匹配。

ADLODNPerformanceLevelsX2结构体用于存储Overdrive N的性能级别信息，并将其作为参数传递给ADL_OverdriveN_ODPerformanceLevels_Get()和ADL_OverdriveN_ODPerformanceLevels_Set()函数。这两个函数分别用于获取和设置Overdrive N的性能级别。


```cpp
/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about Overdrive N performance levels.
///
/// This structure is used to store information about Overdrive performance levels.
/// This structure is used by the ADL_OverdriveN_ODPerformanceLevels_Get() and ADL_OverdriveN_ODPerformanceLevels_Set() functions.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLODNPerformanceLevelsX2
{
    int iSize;
    //Automatic/manual
    int iMode;
    /// Must be set to sizeof( \ref ADLODPerformanceLevels ) + sizeof( \ref ADLODPerformanceLevel ) * (ADLODParameters.iNumberOfPerformanceLevels - 1)
    int iNumberOfPerformanceLevels;
    /// Array of performance state descriptors. Must have ADLODParameters.iNumberOfPerformanceLevels elements.
    ADLODNPerformanceLevelX2 aLevels[1];
} ADLODNPerformanceLevelsX2;

```

这段代码定义了一个名为 ADLODNCurrentPowerType 的枚举类型，它表示了 GPU 中的电流消耗类型。枚举类型包括了 ODN GPU 总电流、ODN GPU PPT 电流、ODN GPU Socket 电流和 ODN GPU 芯片电流。

接下来定义了一个名为 ADLODNCurrentPowerParameters 的结构体，用于存储当前 GPU 电流消耗的参数。该结构体包括了一个名为 powerType 的成员，它表示了当前电流消耗类型的枚举类型，以及一个名为 currentPower 的成员，表示了当前的电流大小。


```cpp
typedef enum _ADLODNCurrentPowerType
{
    ODN_GPU_TOTAL_POWER = 0,
    ODN_GPU_PPT_POWER,
    ODN_GPU_SOCKET_POWER,
    ODN_GPU_CHIP_POWER
} ADLODNCurrentPowerType;

// in/out: CWDDEPM_CURRENTPOWERPARAMETERS
typedef struct _ADLODNCurrentPowerParameters
{
    int   size;
    ADLODNCurrentPowerType   powerType;
    int  currentPower;
} ADLODNCurrentPowerParameters;

```

这段代码定义了一个名为 ADLODNExtSingleInitSetting 的结构体，包含五个整型变量：mode、minValue、maxValue、step 和 defaultValue。这个结构体定义了一个 ADLODNExtSingleInitSetting 类型的变量。

接下来，定义了一个名为 ADLOD8SingleInitSetting 的结构体，包含四个整型变量：featureID、minValue、maxValue 和 defaultValue。这个结构体定义了一个 ADLOD8SingleInitSetting 类型的变量。

这两个结构体都使用了 ODN（Open Data Network）Ext range data structure。Ext range data structure 是一种高效的存储数据类型，可以存储连续的值。

这里的作用是定义了两个结构体，ADLODNExtSingleInitSetting 和 ADLOD8SingleInitSetting。这两个结构体可能用于在程序中设置 ADLODNExtSingle 和 ADLOD8Single 数据类型的实例的某些参数。


```cpp
//ODN Ext range data structure
typedef struct _ADLODNExtSingleInitSetting
{
	int mode;
	int minValue;
	int maxValue;
	int step;
	int defaultValue;
} ADLODNExtSingleInitSetting;

//OD8 Ext range data structure
typedef struct _ADLOD8SingleInitSetting
{
    int featureID;
    int minValue;
    int maxValue;
    int defaultValue;
} ADLOD8SingleInitSetting;


```

这是一个定义了一个名为 ADLOD8InitSetting 的结构体，用于存储 Overdrive8 的初始设置信息。该结构体包含两个成员变量，一个是整型变量 count，表示 Overdrive8 初始设置的数量，另一个是 int 类型的 overdrive8Capabilities，表示 Overdrive8 的 Capability 能力。另外，该结构体还包含一个名为 od8SettingTable 的数组，用于存储 Overdrive8 的各个设置。这里的 overdrive8Capabilities 和 od8SettingTable 成员用于存储 Overdrive8 的不同设置。


```cpp
/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about Overdrive8 initial setting
///
/// This structure is used to store information about Overdrive8 initial setting
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct _ADLOD8InitSetting
{
    int count;
    int overdrive8Capabilities;
    ADLOD8SingleInitSetting  od8SettingTable[OD8_COUNT];
} ADLOD8InitSetting;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about Overdrive8 current setting
```

这段代码定义了一个名为 ADLOD8CurrentSetting 的结构体，用于存储 Overdrive8 的当前设置。

该结构体包含两个成员变量：count 和 Od8SettingTable。其中，count 表示设置的数量，而 Od8SettingTable 是一个包含 8 个整型元素的数组，用于存储设置的具体值。这些值在代码中被限定了，不会被允许为空或超出范围。

ADLOD8CurrentSetting 的定义在函数 ADLOD8CurrentSetting_default 的结尾，而函数的实现则覆盖了该结构体的默认实现，即成员变量为默认值。如果需要使用自定义的设置，则需要手动初始化该结构体的成员变量。


```cpp
///
/// This structure is used to store information about Overdrive8 current setting
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct _ADLOD8CurrentSetting
{
    int count;
    int Od8SettingTable[OD8_COUNT];
} ADLOD8CurrentSetting;


/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about Overdrive8 set setting
///
/// This structure is used to store information about Overdrive8 set setting
```

这段代码定义了一个名为 ADLOD8SingleSetSetting 的结构体，用于表示设置的 ADLOD8 分层设置。这个结构体有两个成员变量，一个是整型变量 value，表示设置的值，另一个是整型变量 requested，表示请求的值，其中 0 为默认值，1 为请求的值。此外，还有一个成员变量 reset，表示是否重置设置，其中 0 为不重置，1 为重置。

接下来定义了一个名为 ADLOD8SetSetting 的结构体，用于表示 ADLOD8 分层设置的计数器。这个结构体有一个成员变量 count，表示设置的层数，以及一个名为 od8SettingTable 的成员变量数组，用于存储 ADLOD8 分层设置的具体值。


```cpp
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////

typedef struct _ADLOD8SingleSetSetting
{
    int value;
    int requested;      // 0 - default , 1 - requested
    int reset;          // 0 - do not reset , 1 - reset setting back to default
} ADLOD8SingleSetSetting;


typedef struct _ADLOD8SetSetting
{
    int count;
    ADLOD8SingleSetSetting  od8SettingTable[OD8_COUNT];
} ADLOD8SetSetting;

```

这段代码定义了一个名为 ADLSingleSensorData 的结构体，用于存储关于 Performance Metrics 数据输出的信息。该结构体包含两个成员变量：supported 和 value，分别表示传感器是否支持该数据类型，以及传感器在 Performance Metrics 数据中输出的值。

接下来，定义了一个名为 ADLPMLogDataOutput 的结构体，用于存储关于 Performance Metrics 数据输出日志的信息。该结构体包含一个名为 sensors 的成员变量，该成员变量是一个 ADLSingleSensorData 类型的数组，用于存储 Performance Metrics 数据中的传感器信息。另外，该结构体还包含一个名为 size 的成员变量，表示日志数据输出的大小，以及一个名为 output 的成员变量，表示用于存储日志数据的输出。


```cpp
/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about Performance Metrics data
///
/// This structure is used to store information about Performance Metrics data output
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct _ADLSingleSensorData
{
    int supported;
    int  value;
} ADLSingleSensorData;

typedef struct _ADLPMLogDataOutput
{
    int size;
    ADLSingleSensorData sensors[ADL_PMLOG_MAX_SENSORS];
}ADLPMLogDataOutput;


```

这段代码定义了一个名为ADLPPLogSettings的结构体，用于存储关于PPLog设置的信息。该结构体包含以下成员：

- BreakOnAssert：当程序在assert语句下被调用时，该成员是否启用断电。如果启用，则会发出一个警告，而不是让程序崩溃。

- BreakOnWarn：当程序在warn语句下被调用时，该成员是否启用断电。如果启用，则会发出一个警告，而不是让程序崩溃。

- LogEnabled：是否启用PPLog输出。如果启用，则记录下来。

- LogFieldMask：用于指定在日志中显示哪些字段。

- LogDestinations：用于指定将日志输出到哪些设备或目的地。

- LogSeverityEnabled：是否启用严重性日志输出。

- LogSourceMask：用于指定在日志中包含哪些源。

- PowerProfilingEnabled：是否启用功率分析。

- PowerProfilingTimeInterval：表示功率分析的时间间隔。


```cpp
/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about PPLog settings.
///
/// This structure is used to store information about PPLog settings.
/// This structure is used by the ADL2_PPLogSettings_Set() and ADL2_PPLogSettings_Get() functions.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct ADLPPLogSettings
{
    int BreakOnAssert;
    int BreakOnWarn;
    int LogEnabled;
    int LogFieldMask;
    int LogDestinations;
    int LogSeverityEnabled;
    int LogSourceMask;
    int PowerProfilingEnabled;
    int PowerProfilingTimeInterval;
}ADLPPLogSettings;

```

这段代码定义了一个名为 ADLFPSSettingsOutput 的结构体，它包含与 AC 和 DC 帧每秒相关的信息。这个结构体用于存储 AC 和 DC 帧每秒设置的信息。

该结构体中包含以下成员：

- ulSize：设置为帧每秒数据的长度。
- bACFPSEnabled：指示 AC 状态下的 FPS 监视是否启用，值为 1 或 0。
- bDCFPSEnabled：指示 DC 状态下的 FPS 监视是否启用，值为 1 或 0。
- ulACFPSCurrent：当前 AC 状态下的 FPS 阈值。
- ulDCFPSCurrent：当前 DC 状态下的 FPS 阈值。
- ulACFPSMaximum：AC 状态下的最大 FPS 阈值。
- ulACFPSMinimum：AC 状态下的最小 FPS 阈值。
- ulDCFPSMaximum：DC 状态下的最大 FPS 阈值。
- ulDCFPSMinimum：DC 状态下的最小 FPS 阈值。

这些成员用于保存 AC 和 DC 帧每秒的 FPS 设置信息，以便可以根据需要进行相应的设置或获取。


```cpp
/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information related Frames Per Second for AC and DC.
///
/// This structure is used to store information related AC and DC Frames Per Second settings
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct _ADLFPSSettingsOutput
{
    /// size
    int ulSize;
    /// FPS Monitor is enabled in the AC state if 1
    int bACFPSEnabled;
    /// FPS Monitor is enabled in the DC state if 1
    int bDCFPSEnabled;
    /// Current Value of FPS Monitor in AC state
    int ulACFPSCurrent;
    /// Current Value of FPS Monitor in DC state
    int ulDCFPSCurrent;
    /// Maximum FPS Threshold allowed in PPLib for AC
    int ulACFPSMaximum;
    /// Minimum FPS Threshold allowed in PPLib for AC
    int ulACFPSMinimum;
    /// Maximum FPS Threshold allowed in PPLib for DC
    int ulDCFPSMaximum;
    /// Minimum FPS Threshold allowed in PPLib for DC
    int ulDCFPSMinimum;

} ADLFPSSettingsOutput;

```

这段代码定义了一个名为ADLFPSSettingsInput的结构体，用于存储与AC和DC控制器相关的帧每秒设置信息。该结构体包含以下成员：

1. ulSize：表示该结构体的大小。
2. bGlobalSettings：指示是否设置全局FPS（用于CCC控制器）。
3. ulACFPSCurrent：当前设置在AC状态下的FPS计数值。
4. ulDCFPSCurrent：当前设置在DC状态下的FPS计数值。
5. ulReserved：保留字段，用于保留 future expansion。

ADLFPSSettingsInput用于存储与帧每秒相关的设置信息，以供操作系统或其他需要使用此设置的应用程序进行参考。


```cpp
/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information related Frames Per Second for AC and DC.
///
/// This structure is used to store information related AC and DC Frames Per Second settings
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct _ADLFPSSettingsInput
{
    /// size
    int ulSize;
    /// Settings are for Global FPS (used by CCC)
    int bGlobalSettings;
    /// Current Value of FPS Monitor in AC state
    int ulACFPSCurrent;
    /// Current Value of FPS Monitor in DC state
    int ulDCFPSCurrent;
    /// Reserved
    int ulReserved[6];

} ADLFPSSettingsInput;

```

这段代码定义了一个名为 ADLPMLOG_SENSORS 的结构体，用于存储电源管理日志相关的支持信息。该结构体包含一个名为 usSensors 的列表，其中包含由 ADLPMLOG_SENSORS 定义的传感器。还包含一个名为 ulReserved 的数组，其中包含 16 个保留的整数。

这个结构体是一个用于存储电源管理日志支持信息的通用模板，可用于定义其他结构体或函数，只需将相应的值填入即可。


```cpp
/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information related power management logging.
///
/// This structure is used to store support information for power management logging.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
enum { ADL_PMLOG_MAX_SUPPORTED_SENSORS = 256 };

typedef struct _ADLPMLogSupportInfo
{
    /// list of sensors defined by ADL_PMLOG_SENSORS
    unsigned short usSensors[ADL_PMLOG_MAX_SUPPORTED_SENSORS];
    /// Reserved
    int ulReserved[16];

} ADLPMLogSupportInfo;


```

这段代码定义了一个名为 ADLPMLogStartInput 的结构体，用于描述开始电源管理日志所需的输入信息。该结构体包含以下字段：

1. usSensors：一个包含 ADL_PMLOG_SENSORS 定义的传感器列表，每个传感器都有一个对应的 usSensor 字段。
2. ulSampleRate：一个包含 ADL_PMLOG_MAX_SUPPORTED_SENSORS 定义的采样率，每个传感器的采样率都有一个对应的 ulSampleRate 字段。
3. ulReserved：一个包含 15 个保留字段的整数数组，每个保留字段都有一个默认值（通常是 0）。

ADLPMLogStartInput 是 ADL2_Adapter_PMLogStart 的输入参数，它被用于在 PMlogStart 函数中开始电源管理日志的记录。


```cpp
/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information to start power management logging.
///
/// This structure is used as input to ADL2_Adapter_PMLog_Start
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct _ADLPMLogStartInput
{
    /// list of sensors defined by ADL_PMLOG_SENSORS
    unsigned short usSensors[ADL_PMLOG_MAX_SUPPORTED_SENSORS];
    /// Sample rate in milliseconds
    unsigned long ulSampleRate;
    /// Reserved
    int ulReserved[15];

} ADLPMLogStartInput;

```

这段代码定义了一个名为 `ADLPMLogData` 的结构体类型，用于存储 ADL 平台模型中的日志数据。

该结构体定义了六个成员变量：

1. `ulVersion`：一个无符号整数，表示数据记录的版本。
2. `ulActiveSampleRate`：一个无符号整数，表示当前正在使用的样本率。
3. `ulLastUpdated`：一个无符号长整数，表示数据记录的最后更新时间。
4. `ulValues`：一个 2D 数组，表示每个传感器的值。这个数组最多可以支持 ADL 平台模型中所有传感器的最大数量。
5. `ulReserved`：一个无符号整数，保留用于未来的数据记录。

这个结构体类型的定义在为其他一些结构体和函数提供共同基准的前提下进行，所以如果你在定义其他结构体或函数时需要使用 `ADLPMLogData` 的话，你需要确保你已经定义了这些结构体，并且已经给它们提供了初始化，否则就会出现编译错误或运行时错误。


```cpp
typedef struct _ADLPMLogData
{
    /// Structure version
    unsigned int ulVersion;
    /// Current driver sample rate
    unsigned int ulActiveSampleRate;
    /// Timestamp of last update
    unsigned long long ulLastUpdated;
    /// 2D array of senesor and values
    unsigned int ulValues[ADL_PMLOG_MAX_SUPPORTED_SENSORS][2];
    /// Reserved
    unsigned int ulReserved[256];

} ADLPMLogData;

```

这段代码定义了一个名为ADL2_Adapter_PMLog_Start输出的结构体，名为ADL2_Adapter_PMLog_StartOutput。这个结构体用于在电源管理（Power Management）中记录日志信息。在代码中，这个结构体被用来作为ADL2_Adapter_PMLog_Start函数的输出参数。

具体来说，这个结构体包含以下成员：

1. 一个指向内存地址的指针，可以是void类型或unsigned long long类型。这个指针用于存储日志数据，可以是广告、用户配置或其他信息。
2. 保留14个整数的ulReserved成员。这些成员通常是保留的，不应该在代码中使用。
3. 字符串ADL2_Adapter_PMLog_StartOutput_str成员，用于存储函数的输入参数和返回值。

ADL2_Adapter_PMLog_Start函数接受一个ADL2_Adapter_PMLog_StartOutput结构体作为输入，然后将其存储在相应的位置。


```cpp
/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information to start power management logging.
///
/// This structure is returned as output from ADL2_Adapter_PMLog_Start
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct _ADLPMLogStartOutput
{
    /// Pointer to memory address containing logging data
    union
    {
        void* pLoggingAddress;
        unsigned long long ptr_LoggingAddress;
    };
    /// Reserved
    int ulReserved[14];

} ADLPMLogStartOutput;


```

这段代码定义了一个名为ADLRASGetErrorCountsInput的结构体，用于存储与RAS（可能是一种汽车操作系统）获取错误计数器相关的信息。该结构体包含一个名为Reserved的16个字节，但实际使用时可能需要更多。

这个结构体用于在函数或函数中携带错误计数器输出的需求。错误计数器通常包含与系统相关的信息，如蓝牙信号强度、网络连接质量等。


```cpp
/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information related RAS Get Error Counts Information
///
/// This structure is used to store RAS Error Counts Get Input Information
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////

typedef struct _ADLRASGetErrorCountsInput
{
    unsigned int                Reserved[16];
} ADLRASGetErrorCountsInput;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information related RAS Get Error Counts Information
///
```

这段代码定义了一个名为ADLRASGetErrorCountsOutput的结构体，用于存储与RAS（RAS 是一种用于内存访问的硬件设备）有关的错误计数信息。该结构体包含以下成员：

1. CorrectedErrors：已正确计算出来的DRAM和SRAM ECC计数器中的错误数量。
2. UnCorrectedErrors：未正确计算出来的DRAM和SRAM ECC计数器中的错误数量。
3. Reserved：保留的14个字节，用于保留其他需要保留的值。

该结构体的定义在头文件中，然后可以在程序中创建该结构体变量，例如：
```cppc
ADLRASGetErrorCountsOutput myObject;
```
在实际使用中，该结构体可以用于在程序执行过程中统计和获取与RAS有关的错误计数信息。


```cpp
/// This structure is used to store RAS Error Counts Get Output Information
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////


typedef struct _ADLRASGetErrorCountsOutput
{
    unsigned int                CorrectedErrors;    // includes both DRAM and SRAM ECC
    unsigned int                UnCorrectedErrors;  // includes both DRAM and SRAM ECC
    unsigned int                Reserved[14];
} ADLRASGetErrorCountsOutput;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information related RAS Get Error Counts Information
///
```

这段代码定义了一个名为ADLRASGetErrorCounts的结构体，用于存储与RAS（Remote Access Linux）相关的错误计数信息。该结构体有以下成员：

1. 输入成员：一个表示输入信息大小的无符号整数，用于存储原始的输入数据。
2. 输入成员：一个ADLRASGetErrorCountsInput类型的无符号整数，用于接收原始输入并对其进行解码。
3. 输出成员：一个表示输出信息大小的无符号整数，用于存储解码后的输出数据。

结构体中的成员函数在程序运行时被赋值，并在程序执行过程中进行相应的计算。ADLRASGetErrorCounts结构体提供了一种通用的接口，让用户可以方便地获取与RAS相关的错误信息。


```cpp
/// This structure is used to store RAS Error Counts Get Information
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////

typedef struct _ADLRASGetErrorCounts
{
    unsigned int                InputSize;
    ADLRASGetErrorCountsInput   Input;
    unsigned int                OutputSize;
    ADLRASGetErrorCountsOutput  Output;
} ADLRASGetErrorCounts;


/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information related RAS Error Counts Reset Information
```

这段代码定义了一个名为ADLRASResetErrorCountsInput的结构体，用于存储与RAS错误计数器相关的信息。该结构体包含一个名为Reserved的8个字节，其中包含一个保留字段，用于存储输入数据。

进一步，该结构体定义了一个名为ADLRASResetErrorCountsOutput的结构体，用于存储与RAS错误计数器相关的输出信息。

最后，该代码段定义了一个名为ADLRASResetErrorCounts函数，该函数将ADLRASResetErrorCountsInput结构体作为其输入参数，并返回ADLRASResetErrorCountsOutput结构体作为其输出结果。


```cpp
///
/// This structure is used to store RAS Error Counts Reset Input Information
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////


typedef struct _ADLRASResetErrorCountsInput
{
    unsigned int                Reserved[8];
} ADLRASResetErrorCountsInput;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information related RAS Error Counts Reset Information
///
/// This structure is used to store RAS Error Counts Reset Output Information
```

这段代码定义了一个名为`ADLRASResetErrorCountsOutput`的结构体类型，用于存储与RAS恢复相关的错误计数信息。该结构体包含一个`Reserved`数组，其中包含8个整数，这些可能用于保留其他数据。

接下来的两行定义了该结构体类型的成员函数。第一行声明了一个名为`Reserved`的数组，用于存储错误计数器的初始值。第二行定义了一个名为`ADLRASResetErrorCountsOutput`的结构体名称，该名称用于存储该数组。

接下来的代码定义了一个名为`RASErrorCountsResetInfo`的函数指针，该函数指针用于调用该结构体类型的成员函数。该函数指针包含一个指向`ADLRASResetErrorCountsOutput`结构体的指针，以及一个字符串`reserved_error_counts_count`作为参数。

该函数指针在调用时被赋值为`RASErrorCountsResetInfo<reserved_error_counts_count>`，然后通过该指针调用结构体类型的成员函数，传入所需的参数。

最后，该代码还定义了一个名为`error_counts_count`的变量，用于存储错误计数器的原始值。


```cpp
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////


typedef struct _ADLRASResetErrorCountsOutput
{
    unsigned int                Reserved[8];
} ADLRASResetErrorCountsOutput;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information related RAS Error Counts Reset Information
///
/// This structure is used to store RAS Error Counts Reset Information
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////


```

这段代码定义了一个名为 `ADLRASResetErrorCounts` 的结构体类型。这个结构体包含四个成员变量：`InputSize`、`Input`、`OutputSize` 和 `Output`。它的含义是：

- `InputSize` 是输入数据的大小。
- `Input` 是输入数据，它被存储在 `ADLRASResetErrorCountsInput` 类型的变量中。
- `OutputSize` 是输出数据的大小。
- `Output` 是输出数据，它被存储在 `ADLRASResetErrorCountsOutput` 类型的变量中。

这个结构体可以被用来定义一个函数，它的参数可以是整数或指针。例如，定义一个函数，接收一个整数参数，用来统计 RAS 错误注入器的错误计数器。


```cpp
typedef struct _ADLRASResetErrorCounts
{
    unsigned int                    InputSize;
    ADLRASResetErrorCountsInput     Input;
    unsigned int                    OutputSize;
    ADLRASResetErrorCountsOutput    Output;
} ADLRASResetErrorCounts;


/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information related RAS Error Injection information
///
/// This structure is used to store RAS Error Injection input information
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////

```

这是一个定义的结构体，名为ADLRASErrorInjectonInput，其包含了与RAS错误注入相关的信息。

具体来说，这个结构体包括以下成员：

1. Address：输入或输出设备的地址，这是一个无符号长整型。
2. Value：ADL_RAS_INJECTION_METHOD，这是用于指定RAS错误注入的值，包括INSERTION_MODE_FILE和INSERTION_MODE_BUFFER。
3. BlockId：用于标识RAS错误注入操作的块ID。
4. InjectErrorType：表示RAS错误注入错误的类型，包括ADL_RAS_ERROR_TYPE_ACTIVITY和ADL_RAS_ERROR_TYPE_IDLE。
5. SubBlockIndex：用于标识子块的索引，范围为0到该结构体中ADL_MEM_SUB_BLOCK_ID的最大值减1。
6. padding：保留空间，用于在结构体中填充字节数。

这个结构体用于表示ADL错误注入的相关信息，可以用于在应用程序中处理和记录RAS错误注入事件。


```cpp
typedef struct _ADLRASErrorInjectonInput
{
    unsigned long long Address;
    ADL_RAS_INJECTION_METHOD Value;
    ADL_RAS_BLOCK_ID BlockId;
    ADL_RAS_ERROR_TYPE InjectErrorType;
    ADL_MEM_SUB_BLOCK_ID SubBlockIndex;
    unsigned int padding[9];
} ADLRASErrorInjectonInput;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information related RAS Error Injection information
///
/// This structure is used to store RAS Error Injection output information
/// \nosubgrouping
```

这段代码定义了一个名为`ADLRASErrorInjectionOutput`的结构体类型，用于存储与RAS错误注入相关的信息。该结构体包含两个成员变量：`ErrorInjectionStatus`和`padding`。其中，`ErrorInjectionStatus`是一个无符号整数，用于表示RAS错误注入的状态；`padding`是一个15个无符号整数的数组，用于存储额外的信息。


```cpp
////////////////////////////////////////////////////////////////////////////////////////////


typedef struct _ADLRASErrorInjectionOutput
{
    unsigned int ErrorInjectionStatus;
    unsigned int padding[15];
} ADLRASErrorInjectionOutput;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information related RAS Error Injection information
///
/// This structure is used to store RAS Error Injection information
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////


```

这段代码定义了一个名为 ADLRASErrorInjection 的结构体，用于表示一个应用程序中的错误信息。该结构体包含以下成员：

1. 输入大小（InputSize）：表示从输入源接收到的数据量。
2. 输入（Input）：指向 ADLRASErrorInjectonInput 的指针，用于表示输入数据中的错误信息。
3. 输出大小（OutputSize）：表示从应用程序输出的数据量。
4. 输出（Output）：指向 ADLRASErrorInjectionOutput 的指针，用于表示应用程序输出的错误信息。


接下来是另一段代码，定义了一个名为 ADLSGApplicationInfo 的结构体，用于表示一个应用程序的基本信息。该结构体包含以下成员：

1. 应用程序文件名（strFileName）：应用程序文件的名称。
2. 应用程序文件路径（strFilePath）：应用程序文件的路径。
3. 应用程序版本（strVersion）：应用程序的版本号。
4. 运行时间（timeStamp）：应用程序运行的时间戳，以秒为单位。
5. 应用程序是否存在配置文件（iProfileExists）：判断应用程序是否配置了相关设置。如果存在，则设置为 1，否则设置为 0。
6. GPU：用于运行应用程序的 GPU。
7. GPU 亲和力（iGPUAffinity）：指定 GPU 在应用程序中的亲和力。
8. GPU BDF：指定 GPU 的基本故障表（BDF）。


```cpp
typedef struct _ADLRASErrorInjection
{
    unsigned int                           InputSize;
    ADLRASErrorInjectonInput               Input;
    unsigned int                           OutputSize;
    ADLRASErrorInjectionOutput             Output;
} ADLRASErrorInjection;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about an application
///
/// This structure is used to store basic information of a recently ran or currently running application
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct _ADLSGApplicationInfo
{
    /// Application file name
    wchar_t strFileName[ADL_MAX_PATH];
    /// Application file path
    wchar_t strFilePath[ADL_MAX_PATH];
    /// Application version
    wchar_t strVersion[ADL_MAX_PATH];
    /// Timestamp at which application has run
    long long int timeStamp;
    /// Holds whether the applicaition profile exists or not
    unsigned int iProfileExists;
    /// The GPU on which application runs
    unsigned int iGPUAffinity;
    /// The BDF of the GPU on which application runs
    ADLBdf GPUBdf;
} ADLSGApplicationInfo;

```

这段代码定义了一个名为"ADLPreFlipPostProcessingLUTAlgorithm"的枚举类型，其中包含了三种枚举值：ADLPreFlipPostProcessingLUTAlgorithm_Default、ADLPreFlipPostProcessingLUTAlgorithm_Full和ADLPreFlipPostProcessingLUTAlgorithm_Approximation。这些枚举类型用于表示处理来自AC和DC信号的LUT设置。

具体来说，这段代码定义了一个名为"ADLPreFlipPostProcessingLUTAlgorithmInfo"的枚举类型，其中包含了一个名为"ADLPreFlipPostProcessingLUTAlgorithmIndex"的成员，用于表示输入的LUT算法类型。具体来说，枚举类型ADLPreFlipPostProcessingLUTAlgorithm包含了以下几种枚举值：

* ADLPreFlipPostProcessingLUTAlgorithm_Default：默认的LUT算法，用于处理来自AC和DC信号的LUT设置。
* ADLPreFlipPostProcessingLUTAlgorithm_Full：完整的LUT算法，用于处理来自AC和DC信号的LUT设置，此时LUT将被完全加权以确保相同的输出。
* ADLPreFlipPostProcessingLUTAlgorithm_Approximation：近似的LUT算法，用于处理来自AC和DC信号的LUT设置，此时LUT设置将被舍弃以节省存储空间。

此外，该代码定义了一个名为"ADLPreFlipPostProcessingLUTAlgorithmInfoValidLUTIndex"的枚举类型，用于表示有效的LUT索引。该枚举类型使用了ADLPreFlipPostProcessingLUTAlgorithmInfo中的成员，具体初始值由上文的ADLPreFlipPostProcessingLUTAlgorithmIndex成员指定。


```cpp
/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information related Frames Per Second for AC and DC.
///
/// This structure is used to store information related AC and DC Frames Per Second settings
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
enum { ADLPreFlipPostProcessingInfoInvalidLUTIndex = 0xFFFFFFFF };

enum ADLPreFlipPostProcessingLUTAlgorithm
{
    ADLPreFlipPostProcessingLUTAlgorithm_Default = 0,
    ADLPreFlipPostProcessingLUTAlgorithm_Full,
    ADLPreFlipPostProcessingLUTAlgorithm_Approximation
};

```

这段代码定义了一个名为 ADLPreFlipPostProcessingInfo 的结构体，表示在使用 Adobe瞎猫糊弄人的 Dynamo UI 应用时需要的一些信息。这个结构体包含以下几个字段：

- ulSize：表示该结构体的大小，即 16 字节。
- bEnabled：表示是否启用了这个功能，如果为 1，则表示已启用，否则表示未启用。
- ulSelectedLUTIndex：表示当前选中的 LUT（Level 12 Transitions）索引，如果没有选择，则返回 0xFFFFFFF。
- ulSelectedLUTAlgorithm：表示当前选中的 LUT 算法，0x112对应4。
- ulReserved：保留字段，用于保留未来可能的用处，目前未知。

这个结构体可能是用来在 Dynamo UI 中设置 LUT，并返回选定的 LUT 的信息。


```cpp
typedef struct _ADLPreFlipPostProcessingInfo
{
    /// size
    int ulSize;
    /// Current active state
    int bEnabled;
    /// Current selected LUT index.  0xFFFFFFF returned if nothing selected.
    int ulSelectedLUTIndex;
    /// Current selected LUT Algorithm
    int ulSelectedLUTAlgorithm;
    /// Reserved
    int ulReserved[12];

} ADLPreFlipPostProcessingInfo;



```

这段代码定义了一个名为 ADL_ERROR_REASON 的结构体类型，以及一个名为 ADL_DELAG_NOTFICATION_REASON 的结构体类型。这两个结构体类型都包含一个名为 boost 的整型成员和一个名为 chill 的整型成员。

ADL_ERROR_REASON 结构体成员 boost 和 chill 分别表示在按键是否有被按下时，进入提示框中的状态，当 boost 或 chill 被启用时，将处于不同的状态。

ADL_DELAG_NOTFICATION_REASON 结构体成员 HotkeyChanged、GlobalEnableChanged 和 GlobalLimitFPSChanged 分别表示当热键的值被更改时，进入提示框中的状态。当这三个成员中的任意一个被设置时，进入提示框中的状态。


```cpp
typedef struct _ADL_ERROR_REASON
{
	int boost; //ON, when boost is Enabled
	int delag; //ON, when delag is Enabled
	int chill; //ON, when chill is Enabled
}ADL_ERROR_REASON;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about DELAG Settings change reason
///
///  Elements of DELAG settings changed reason.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct _ADL_DELAG_NOTFICATION_REASON
{
	int HotkeyChanged; //Set when Hotkey value is changed
	int GlobalEnableChanged; //Set when Global enable value is changed
	int GlobalLimitFPSChanged; //Set when Global enable value is changed
}ADL_DELAG_NOTFICATION_REASON;

```

这段代码定义了一个名为 ADL_DELAG_SETTINGS 的结构体类型，用于表示 DELAG 设置中的元素。该结构体包含以下成员：

- Hotkey: 热键值
- GlobalEnable: 全局 enable 值
- GlobalLimitFPS: 全局限速 FPS 值
- GlobalLimitFPS_MinLimit: 全局限速 FPS 值的最低限制
- GlobalLimitFPS_MaxLimit: 全局限速 FPS 值的最高限制
- GlobalLimitFPS_Step: 全局限速 FPS 值的小数位数

这个结构体可以用来定义 ADL 设置中的元素，并且在程序中可以通过输入 ADL_DELAG_SETTINGS 类型的值来设置这些 settings。


```cpp
/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about DELAG Settings
///
///  Elements of DELAG settings.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct _ADL_DELAG_SETTINGS
{
	int Hotkey; // Hotkey value
	int GlobalEnable; //Global enable value
	int GlobalLimitFPS; //Global Limit FPS
	int GlobalLimitFPS_MinLimit; //Gloabl Limit FPS slider min limit value
	int GlobalLimitFPS_MaxLimit; //Gloabl Limit FPS slider max limit value
	int GlobalLimitFPS_Step; //Gloabl Limit FPS step  value
}ADL_DELAG_SETTINGS;

```

这段代码定义了一个名为`ADL_BOOST_NOTFICATION_REASON`的结构体，它包含关于BOOST设置的信息。这个结构体定义了三个整型变量：`HotkeyChanged`、`GlobalEnableChanged`和`GlobalMinResChanged`。

这个结构体在函数签名中作为参数传递，然后被用来初始化函数内部的一个整型变量。这个整型变量在整个函数中都被使用，负责根据传入的设置信息，输出一个表示BOOST设置变更原因的提示信息。


```cpp
/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about BOOST Settings change reason
///
///  Elements of BOOST settings changed reason.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct _ADL_BOOST_NOTFICATION_REASON
{
	int HotkeyChanged; //Set when Hotkey value is changed
	int GlobalEnableChanged; //Set when Global enable value is changed
	int GlobalMinResChanged; //Set when Global min resolution value is changed
}ADL_BOOST_NOTFICATION_REASON;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about BOOST Settings
```

这段代码定义了一个名为ADL_BOOST_SETTINGS的结构体，该结构体包含以下成员：

- Hotkey: 热键，即用户定义的快捷键
- GlobalEnable: 全局启用，用于设置是否启用ADL_BOOST设置
- GlobalMinRes: 全球最小分辨率，用于设置窗口最小分辨率
- GlobalMinRes_MinLimit: 全球最小分辨率滑块的最小限制值
- GlobalMinRes_MaxLimit: 全球最小分辨率滑块的最大限制值
- GlobalMinRes_Step: 全球最小分辨率滑块的步长值

该结构体可能用于在程序中设置ADL_BOOST功能，例如通过引用ADL_BOOST_SETTINGS来设置热键，设置全局启用，设置全球最小分辨率等。


```cpp
///
///  Elements of BOOST settings.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct _ADL_BOOST_SETTINGS
{
	int Hotkey; // Hotkey value
	int GlobalEnable; //Global enable value
	int GlobalMinRes; //Gloabl Min Resolution value
	int GlobalMinRes_MinLimit; //Gloabl Min Resolution slider min limit value
	int GlobalMinRes_MaxLimit; //Gloabl Min Resolution slider max limit value
	int GlobalMinRes_Step; //Gloabl Min Resolution step  value
}ADL_BOOST_SETTINGS;

/////////////////////////////////////////////////////////////////////////////////////////////
```

这段代码定义了一个名为`ADL_RIS_NOTFICATION_REASON`的结构体，它包含了关于RIS设置更改原因的信息。

该结构体有两个成员变量：`GlobalEnableChanged`和`GlobalSharpeningDegreeChanged`。这些成员变量分别表示全局 enable 设置和全局 sharpening 度设置是否发生了更改。如果任何一个成员变量被设置为`1`，那么就可以认为 RIS 设置已经发生了更改，并需要执行相应的 action。

该结构体的定义在`/\!\堔/\brief`声明之前，说明该结构体在程序中的作用是定义一个二进制字段，用于表示 RIS 设置更改的原因。

接下来的代码定义了一个名为`RIS_SETTINGS_CHANGE_REASON_MAP`的数组，它包含了一个枚举类型`RIS_SETTINGS_CHANGE_REASON_MAP_VALUES`，它定义了该数组元素的每一个成员变量。

例如，`RIS_SETTINGS_CHANGE_REASON_MAP_VALUES[0]`表示全局 enable 设置，它的值为`ADL_RIS_NOTFICATION_REASON_GLOBAL_ENABLE_CHANGED`，如果这个成员变量被设置为`1`，那么就表示全局 enable 设置已经发生了更改。


```cpp
///\brief Structure containing information about RIS Settings change reason
///
///  Elements of RIS settings changed reason.
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct _ADL_RIS_NOTFICATION_REASON
{
	unsigned int GlobalEnableChanged; //Set when Global enable value is changed
	unsigned int GlobalSharpeningDegreeChanged; //Set when Global sharpening Degree value is changed
}ADL_RIS_NOTFICATION_REASON;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about RIS Settings
///
///  Elements of RIS settings.
```

这段代码定义了一个名为 ADL_RIS_SETTINGS 的结构体，用于表示 CHILL 设置的更改原因。该结构体包含以下字段：

- GlobalEnable：设置是否启用全局 CHILL 设置。
- GlobalSharpeningDegree：设置全局 sharpening（平滑度）的值。
- GlobalSharpeningDegree_MinLimit：全局 sharpening 的最小限制值。
- GlobalSharpeningDegree_MaxLimit：全局 sharpening 的最大限制值。
- GlobalSharpeningDegree_Step：全局 sharpening 的步长值。

这个结构体在代码中可能被用来接收和处理 CHILL 设置的更改信息。


```cpp
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct _ADL_RIS_SETTINGS
{
	int GlobalEnable; //Global enable value
	int GlobalSharpeningDegree; //Global sharpening value
	int GlobalSharpeningDegree_MinLimit; //Gloabl sharpening slider min limit value
	int GlobalSharpeningDegree_MaxLimit; //Gloabl sharpening slider max limit value
	int GlobalSharpeningDegree_Step; //Gloabl sharpening step  value
}ADL_RIS_SETTINGS;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about CHILL Settings change reason
///
///  Elements of Chiil settings changed reason.
```

这段代码定义了一个名为 `ADL_CHILL_NOTFICATION_REASON` 的结构体，用于表示 CHILL 设置的信息。这个结构体包含以下成员：

* `HotkeyChanged`：表示热键更改事件的发生时间，当热键的值发生改变时，这个成员会被设置为 1。
* `GlobalEnableChanged`：表示全局 enable 设置的发生时间，当全局 enable 设置的值发生改变时，这个成员会被设置为 1。
* `GlobalMinFPSChanged`：表示全局最小 FPS 设置的发生时间，当全局最小 FPS 设置的值发生改变时，这个成员会被设置为 1。
* `GlobalMaxFPSChanged`：表示全局最大 FPS 设置的发生时间，当全局最大 FPS 设置的值发生改变时，这个成员会被设置为 1。

这个结构体被用于定义一个名为 `ADL_CHILL_NOTFICATION_EVENT` 的函数，这个函数用于处理 CHILL 设置信息。当 CHILL 设置信息发生变化时，这个函数会被调用，并且可以通过它的参数来访问设置信息的变化时间。


```cpp
/// \nosubgrouping
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct _ADL_CHILL_NOTFICATION_REASON
{
	int HotkeyChanged; //Set when Hotkey value is changed
	int GlobalEnableChanged; //Set when Global enable value is changed
	int GlobalMinFPSChanged; //Set when Global min FPS value is changed
	int GlobalMaxFPSChanged; //Set when Global max FPS value is changed
}ADL_CHILL_NOTFICATION_REASON;

/////////////////////////////////////////////////////////////////////////////////////////////
///\brief Structure containing information about CHILL Settings
///
///  Elements of Chiil settings.
/// \nosubgrouping
```

这段代码定义了一个名为ADL_CHILL_SETTINGS的结构体，该结构体用于表示自定义 chill 设置。这个结构体定义了以下参数：

* Hotkey: 热键，用于调用这个函数
* GlobalEnable: 全局启用，用于设置是否启用这个热键
* GlobalMinFPS: 全球最小 FPS，用于设置是否以低于这个 FPS 运行
* GlobalMaxFPS: 全球最大 FPS，用于设置是否以高于这个 FPS 运行
* GlobalFPS_MinLimit: 全球 FPS 滑动器最小限制，用于设置 FPS 滑动器是否能够低于这个值
* GlobalFPS_MaxLimit: 全球 FPS 滑动器最大限制，用于设置 FPS 滑动器是否能够高于这个值
* GlobalFPS_Step: 全球 FPS 滑动器步长，用于设置 FPS 滑动器每秒变化的步长

这个函数的作用是定义了一个自定义的 chill 设置结构体，用于设置如何使用热键启动某些函数。


```cpp
////////////////////////////////////////////////////////////////////////////////////////////
typedef struct _ADL_CHILL_SETTINGS
{
	int Hotkey; // Hotkey value
	int GlobalEnable; //Global enable value
	int GlobalMinFPS; //Global Min FPS value
	int GlobalMaxFPS; //Global Max FPS value
	int GlobalFPS_MinLimit; //Gloabl FPS slider min limit value
	int GlobalFPS_MaxLimit; //Gloabl FPS slider max limit value
	int GlobalFPS_Step; //Gloabl FPS Slider step  value
}ADL_CHILL_SETTINGS;
#endif /* ADL_STRUCTURES_H_ */

```