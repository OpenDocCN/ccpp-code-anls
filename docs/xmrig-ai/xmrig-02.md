# xmrig源码解析 2

# `src/3rdparty/argon2.h`

这段代码是一个 XMRig 钱包的 LecPay 付款接口的 C 语言实现。XMRig 是一个 XMPP 协议的库，支持多种协议的电子邮件和消息传递，而 LecPay 是 XMPP 协议的一个子协议，允许用户使用 XMPP 库进行在线支付。

具体来说，这段代码实现了以下功能：

1. 定义了程序需要使用的类和函数，包括从 XMPP 协议中抽象出来的抽象票据类型、地址、认证信息以及支付信息等。
2. 实现了一个 LecPay 钱包的接口，包括初始化钱包、支付、查询钱包余额等基本功能。
3. 实现了 LecPay 钱包的付款接口，允许用户选择不同的付款方式，包括选择购买 TYPES 0 和 1 的货币，设置 BTC 钱包的 ID，以及设置输出的付款信息。
4. 导入了一些必要的库，包括字符串池库和一些其他人名。


```cpp
/* XMRig
 * Copyright 2010      Jeff Garzik <jgarzik@pobox.com>
 * Copyright 2012-2014 pooler      <pooler@litecoinpool.org>
 * Copyright 2014      Lucas Jones <https://github.com/lucasjones>
 * Copyright 2014-2016 Wolf9466    <https://github.com/OhGodAPet>
 * Copyright 2016      Jay D Dee   <jayddee246@gmail.com>
 * Copyright 2017-2018 XMR-Stak    <https://github.com/fireice-uk>, <https://github.com/psychocrypt>
 * Copyright 2018-2019 SChernykh   <https://github.com/SChernykh>
 * Copyright 2016-2019 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   This program is free software: you can redistribute it and/or modify
 *   it under the terms of the GNU General Public License as published by
 *   the Free Software Foundation, either version 3 of the License, or
 *   (at your option) any later version.
 *
 *   This program is distributed in the hope that it will be useful,
 *   but WITHOUT ANY WARRANTY; without even the implied warranty of
 *   MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the
 *   GNU General Public License for more details.
 *
 *   You should have received a copy of the GNU General Public License
 *   along with this program. If not, see <http://www.gnu.org/licenses/>.
 */


```

这段代码是一个头文件#include "3rdparty/argon2/include/argon2.h"的定义。它告诉编译器在编译之前包含argon2.h文件，从而使编译器能够识别该文件中定义的符号。

具体来说，该代码将定义一个名为"XMRIG_3RDPARTY_ARGON2_H"的头文件，其中包含一个以#include "3rdparty/argon2/include/argon2.h"开头的函数声明。这个函数声明中定义了一个名为"XMRIG_3RDPARTY_ARGON2"的函数，它是一个公共函数，可以被任何需要使用它的地方调用。


```cpp
#ifndef XMRIG_3RDPARTY_ARGON2_H
#define XMRIG_3RDPARTY_ARGON2_H


#include "3rdparty/argon2/include/argon2.h"


#endif /* XMRIG_3RDPARTY_ARGON2_H */

```

# `src/3rdparty/cl.h`

该代码是一个XMRig库的源代码。XMRig是一个跨平台的VR开发框架，支持Web和移动平台，它允许您创建VR应用程序，并支持开源社区。该代码中定义了一些函数，变量和常量。以下是对该代码的一些解释：

1. `XMRig`：这是一个名为XMRig的跨平台VR开发框架，允许创建VR应用程序，并支持开源社区。

2. `Copyright 2010...`：这是该代码的版权信息，显示了作者的姓名和公司。

3. `...`：省略了部分行，可能是为了让代码更易于阅读和理解。

4. `XMRig`：这是该代码的一个别名，它是作者为自己项目创建的一个品牌名称。

5. `Copyright 2012-2014...`：这是该代码的版权信息，显示了作者的姓名和公司。

6. `...`：省略了部分行，可能是为了让代码更易于阅读和理解。

7. `Copyright 2014...`：这是该代码的版权信息，显示了作者的姓名和公司。

8. `...`：省略了部分行，可能是为了让代码更易于阅读和理解。

9. `...`：这是该代码的版权信息，显示了作者的姓名和公司。

10. `Copyright 2016-2019...`：这是该代码的版权信息，显示了作者的姓名和公司。

11. `...`：省略了部分行，可能是为了让代码更易于阅读和理解。

12. `...`：这是该代码的版权信息，显示了作者的姓名和公司。

13. `Copyright 2016...`：这是该代码的版权信息，显示了作者的姓名和公司。

14. `...`：省略了部分行，可能是为了让代码更易于阅读和理解。

15. `...`：这是该代码的版权信息，显示了作者的姓名和公司。

16. `...`：这是该代码的版权信息，显示了作者的姓名和公司。

17. `...`：这是该代码的版权信息，显示了作者的姓名和公司。

18. `...`：这是该代码的版权信息，显示了作者的姓名和公司。

19. `...`：这是该代码的版权信息，显示了作者的姓名和公司。

20. `...`：这是该代码的版权信息，显示了作者的姓名和公司。

21. `...`：这是该代码的版权信息，显示了作者的姓名和公司。

22. `...`：这是该代码的版权信息，显示了作者的姓名和公司。

23. `...`：这是该代码的版权信息，显示了作者的姓名和公司。

24. `...`：这是该代码的版权信息，显示了作者的姓名和公司。

25. `...`：这是该代码的版权信息，显示了作者的姓名和公司。

26. `...`：这是该代码的版权信息，显示了作者的姓名和公司。

27. `...`：这是该代码的版权信息，显示了作者的姓名和公司。

28. `...`：这是该代码的版权信息，显示了作者的姓名和公司。

29. `...`：这是该代码的版权信息，显示了作者的姓名和公司。

30. `...`：这是该代码的版权信息，显示了作者的姓名和公司。


```cpp
/* XMRig
 * Copyright 2010      Jeff Garzik <jgarzik@pobox.com>
 * Copyright 2012-2014 pooler      <pooler@litecoinpool.org>
 * Copyright 2014      Lucas Jones <https://github.com/lucasjones>
 * Copyright 2014-2016 Wolf9466    <https://github.com/OhGodAPet>
 * Copyright 2016      Jay D Dee   <jayddee246@gmail.com>
 * Copyright 2017-2018 XMR-Stak    <https://github.com/fireice-uk>, <https://github.com/psychocrypt>
 * Copyright 2018-2019 SChernykh   <https://github.com/SChernykh>
 * Copyright 2016-2019 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   This program is free software: you can redistribute it and/or modify
 *   it under the terms of the GNU General Public License as published by
 *   the Free Software Foundation, either version 3 of the License, or
 *   (at your option) any later version.
 *
 *   This program is distributed in the hope that it will be useful,
 *   but WITHOUT ANY WARRANTY; without even the implied warranty of
 *   MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the
 *   GNU General Public License for more details.
 *
 *   You should have received a copy of the GNU General Public License
 *   along with this program. If not, see <http://www.gnu.org/licenses/>.
 */

```

这段代码定义了一个头文件 "XMRIG_CL_H"，其中定义了一些关于OpenCL客户端库头文件的字符串和预处理指令。

具体来说，该代码会先检查是否定义了APL信用，如果是，就会包含<OpenCL/cl.h>头文件，否则就会包含 "3rdparty/CL/cl.h" 头文件。然后在头部文件中声明了一个名为 "XMRIG_CL_H" 的头文件，定义了一些字符串和预处理指令，最后在包含XMRIG_CL_H的头文件外面加入了一个#include指示符，表示这些预处理指令要在编译时引入。

因此，该代码的作用是定义了一个名为 "XMRIG_CL_H" 的头文件，其中包含OpenCL客户端库头文件以及定义了一些字符串和预处理指令。


```cpp
#ifndef XMRIG_CL_H
#define XMRIG_CL_H


#if defined(__APPLE__)
#   include <OpenCL/cl.h>
#else
#   include "3rdparty/CL/cl.h"
#endif


#endif /* XMRIG_CL_H */

```

# `src/3rdparty/adl/adl_defines.h`

这段代码是一个C语言的源代码，它定义了一个名为"example_license.h"的函数。这个函数的作用是输出一段文本，具体内容如下：
```cppobjectivec
#include <stdio.h>

void example_license()
{
   printf("Copyright (c) 2016 Advanced Micro Devices, Inc. All rights reserved.\n");
   printf("MIT LICENSE: Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the 'Software').\n");
   printf("THE SOFTWARE IS PROVIDED 'AS IS', WITHOUT WARRANTY OF ANY KIND, EXPRESS OR INDIRECT我只招， DISTRIBUTION OR LIKEwise出售拷贝，\n");
   printf("在有限的责任下提供。\n");
   printf("\n");
}
```
函数首先引入了两个头文件：`stdio.h` 和 `example_license.h`，接着定义了一个名为 `example_license` 的函数。在这个函数中，它使用了 `printf` 函数来输出它所包含的内容。

具体来说，函数首先包含了两个 `printf` 函数，分别传递了一个字符串和两个字符串作为参数。第一个 `printf` 函数输出了一行字符串，其中包含了一行 'Copyright' 和 'example_license' 的字符串，以及一行 '软件著作权声明：本文腿部有限的责任下提供，在使用或复制和/或发布任何形式的软件时，您必须承担下列限制：' 的字符串。第二个 `printf` 函数则包含了一个多行字符串，其中包含了一行 '软件提供者保留权利' 和 '软件在此免费使用，前提是您按照以下限制使用：' 的字符串。

通过 `example_license` 函数的输出，这段代码的作用是告诉用户它所包含的软件是遵循某种许可证协议的，并且可以在各种方式下免费使用，但使用或复制软件本身是需要有限的责任的。


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

这段代码是一个C++头文件，名为adl_defines.h。它包含由ADL（软件描述语言）编写的所有定义，这些定义旨在为所有平台提供对软件的功能和效用。这些定义包括ADL错误代码、ADLDisplayInfo结构的枚举以及最大限制。这些定义被包含在ADL SDK中，因此在使用ADL SDK时，这些定义是必需的。


```cpp
// IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
// FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
// AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
// LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
// OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
// SOFTWARE.

/// \file adl_defines.h
/// \brief Contains all definitions exposed by ADL for \ALL platforms.\n <b>Included in ADL SDK</b>
///
/// This file contains all definitions used by ADL.
/// The ADL definitions include the following:
/// \li ADL error codes
/// \li Enumerations for the ADLDisplayInfo structure
/// \li Maximum limits
```

这段代码定义了一个名为"ADL\_DEFINES\_H\_"的文件，其中包含了一些定义。这些定义可能是在一个名为"ADL"的项目的头文件中定义的。这些定义可能包括一些常量、数据类型、函数、类等等。具体的作用和用途需要根据上下文来确定。


```cpp
///

#ifndef ADL_DEFINES_H_
#define ADL_DEFINES_H_

/// \defgroup DEFINES Constants and Definitions
// @{

/// \defgroup define_misc Miscellaneous Constant Definitions
// @{

/// \name General Definitions
// @{

/// Defines ADL_TRUE
```

这段代码定义了一系列头文件，其中包含了一些常量和定义，用于定义ADL编程语言的辅助函数和宏。

ADL_TRUE定义了一个常量，表示ADL编程语言是否为真，值为1。ADL_FALSE定义了一个常量，表示ADL编程语言是否为假，值为0。

ADL_MAX_CHAR定义了一个宏，表示ADL编程语言支持的最大字符数，值为4096。ADL_MAX_PATH定义了一个宏，表示ADL编程语言支持的最大路径长度，值为256。ADL_MAX_ADAPTERS定义了一个宏，表示ADL编程语言支持的最大适配器数量，值为250。ADL_MAX_DISPLAY定义了一个宏，表示ADL编程语言支持的最大显示数量，值为150。ADL_MAX_DEVICENAME定义了一个宏，表示ADL编程语言支持的最大设备名称长度，值为32。

最后，定义了一系列用于所有适配器的常量和变量，用于定义ADL编程语言的辅助函数和宏。


```cpp
#define ADL_TRUE    1
/// Defines ADL_FALSE
#define ADL_FALSE        0

/// Defines the maximum string length
#define ADL_MAX_CHAR                                    4096
/// Defines the maximum string length
#define ADL_MAX_PATH                                    256
/// Defines the maximum number of supported adapters
#define ADL_MAX_ADAPTERS                               250
/// Defines the maxumum number of supported displays
#define ADL_MAX_DISPLAYS                                150
/// Defines the maxumum string length for device name
#define ADL_MAX_DEVICENAME                                32
/// Defines for all adapters
```

这段代码定义了一些与自定义DDC（Direct-To-Cable）适配器相关的API选项。通过使用宏定义，可以简短地定义这些选项，而不需要使用长命名法。以下是这段代码的作用：

1. 通过定义宏，声明了两个常量：ADL_ADAPTER_INDEX_ALL和ADL_MAIN_API_OPTION_NONE。
2. 通过定义宏，声明了一个名为iOption的参数。
3. 通过定义宏，定义了ADL_DDC_OPTION_SWITCHDDC2，表示在DDC线2上切换DCC（Direct-To-Cable）模式。
4. 通过定义宏，定义了ADL_DDC_OPTION_RESTORECOMMAND，表示将命令保存到注册表中的唯一键，对应于参数iCommandIndex。
5. 通过定义宏，定义了ADL_DDC_OPTION_COMBOWRITERead，表示组合写入DDC块的命令。
6. 通过定义宏，定义了-1，表示ADL_ADAPTER_INDEX_ALL的相反数，作为无定义的常量。


```cpp
#define ADL_ADAPTER_INDEX_ALL                            -1
///    Defines APIs with iOption none
#define ADL_MAIN_API_OPTION_NONE                        0
// @}

/// \name Definitions for iOption parameter used by
/// ADL_Display_DDCBlockAccess_Get()
// @{

/// Switch to DDC line 2 before sending the command to the display.
#define ADL_DDC_OPTION_SWITCHDDC2              0x00000001
/// Save command in the registry under a unique key, corresponding to parameter \b iCommandIndex
#define ADL_DDC_OPTION_RESTORECOMMAND 0x00000002
/// Combine write-read DDC block access command.
#define ADL_DDC_OPTION_COMBOWRITEREAD 0x00000010
```

这段代码定义了一个名为 ADL_DDC_OPTION_SENDTOIMMEDIATEDEVICE 的宏，其值为 0x00000020。该宏的含义是：

如果设置了 ADL_DDC_OPTION_SENDTOIMMEDIATEDEVICE，则 DDC 命令将发送到第一个分支的设备；如果未设置该选项，则 DDC 命令将发送到分支的末端设备。这里的 "immediate device" 指的是与图形卡直接连接的设备。

接着，定义了两个名为 ADL_DL_I2C_ACTIONREAD 和 ADL_DL_I2C_ACTIONWRITE 的宏，它们的含义是：

ADL_DL_I2C_ACTIONREAD 表示读取 I2C 设备的状态
ADL_DL_I2C_ACTIONWRITE 表示向 I2C 设备写入数据

最后，定义了一个名为 ADL_DDC_OPTION_SENDTOIMMEDIATEDEVICE 的宏，其值为 0x00000020。


```cpp
/// Direct DDC access to the immediate device connected to graphics card.
/// MST with this option set: DDC command is sent to first branch.
/// MST with this option not set: DDC command is sent to the end node sink device.
#define ADL_DDC_OPTION_SENDTOIMMEDIATEDEVICE 0x00000020
// @}

/// \name Values for
/// ADLI2C.iAction used with ADL_Display_WriteAndReadI2C()
// @{

#define ADL_DL_I2C_ACTIONREAD                                    0x00000001
#define ADL_DL_I2C_ACTIONWRITE                                0x00000002
#define ADL_DL_I2C_ACTIONREAD_REPEATEDSTART    0x00000003
// @}


```

这段代码定义了一个名为“define_adl_results”的函数组，其中定义了多个常量，用于表示ADL函数的不同结果。这些常量分别使用了预定义的编号，可以在程序中根据需要进行查阅。

具体来说，这个函数组定义了四种状态的代码：ADL_OK_WAIT（等待结果），ADL_OK_RESTART（重新启动），ADL_OK但需要模式更改（需要更改模式），ADL_OK_WARNING（发出警告），以及ADL_OK（函数完成成功）。这些代码根据函数执行的不同，返回一个数字，用于表示ADL函数的状态。


```cpp
// @}        //Misc

/// \defgroup define_adl_results Result Codes
/// This group of definitions are the various results returned by all ADL functions \n
// @{
/// All OK, but need to wait
#define ADL_OK_WAIT                4
/// All OK, but need restart
#define ADL_OK_RESTART                3
/// All OK but need mode change
#define ADL_OK_MODE_CHANGE            2
/// All OK, but with warning
#define ADL_OK_WARNING                1
/// ADL function completed successfully
#define ADL_OK                    0
```

这段代码定义了一些错误常量，用于表示ADL（高级糖尿病治疗）框架的错误情况。这些错误常量用于在程序中报告不同类型的ADL错误。

具体来说，这些错误常量包括：
- ADL_ERR：表示ADL初始化失败。
- ADL_ERR_NOT_INIT：表示至少有一个Escape调用来初始化ADL时失败。
- ADL_ERR_INVALID_PARAM：表示传递给ADL的参数不正确。
- ADL_ERR_INVALID_PARAM_SIZE：表示传递给ADL的参数参数大小不正确。
- ADL_ERR_INVALID_ADL_IDX：表示传递给ADL的IDL不正确。
- ADL_ERR_INVALID_CONTROLLER_IDX：表示传递给ADL的控制器IDL不正确。
- ADL_ERR_INVALID_DIPLAY_IDX：表示传递给ADL的DIPLAY IDX不正确。

当程序遇到这些错误时，会输出相应的错误信息，并可能导致程序崩溃。


```cpp
/// Generic Error. Most likely one or more of the Escape calls to the driver failed!
#define ADL_ERR                    -1
/// ADL not initialized
#define ADL_ERR_NOT_INIT            -2
/// One of the parameter passed is invalid
#define ADL_ERR_INVALID_PARAM            -3
/// One of the parameter size is invalid
#define ADL_ERR_INVALID_PARAM_SIZE        -4
/// Invalid ADL index passed
#define ADL_ERR_INVALID_ADL_IDX            -5
/// Invalid controller index passed
#define ADL_ERR_INVALID_CONTROLLER_IDX        -6
/// Invalid display index passed
#define ADL_ERR_INVALID_DIPLAY_IDX        -7
/// Function  not supported by the driver
```

这是一段C语言代码，定义了一系列错误码。这些错误码用于在代码中表明发生了哪些错误。

具体来说，这些错误码包括：

-8 ADL_ERR_NOT_SUPPORTED：表示该函数或变量没有被定义支持。
-9 ADL_ERR_NULL_POINTER：表示尝试访问一个空指针。
-10 ADL_ERR_DISABLED_ADAPTER：表示尝试使用一个被禁用的适配器。
-11 ADL_ERR_INVALID_CALLBACK：表示尝试使用无效的回调函数。
-12 ADL_ERR_RESOURCE_CONFLICT：表示资源冲突，例如在尝试访问一个已经被其他请求占用的资源时。
-20 ADL_ERR_SET_INCOMPLETE：表示设置某些值时，该操作没有完成。
-21 ADL_ERR_NO_XDISPLAY：表示在 Linux 桌面上找不到 Linux XDisplay。

这些错误码都可以通过在代码中使用#define 定义为常量来避免在代码中频繁出现。例如：
```cpp
#define ADL_ERR_NOT_SUPPORTED -8
#define ADL_ERR_NULL_POINTER   -9
#define ADL_ERR_DISABLED_ADAPTER -10
#define ADL_ERR_INVALID_CALLBACK -11
#define ADL_ERR_RESOURCE_CONFLICT  -12
#define ADL_ERR_SET_INCOMPLETE   -20
#define ADL_ERR_NO_XDISPLAY    -21
```
这样，在代码中只需要使用这些常量名称，而不需要包含对应的错误码。


```cpp
#define ADL_ERR_NOT_SUPPORTED            -8
/// Null Pointer error
#define ADL_ERR_NULL_POINTER            -9
/// Call can't be made due to disabled adapter
#define ADL_ERR_DISABLED_ADAPTER        -10
/// Invalid Callback
#define ADL_ERR_INVALID_CALLBACK            -11
/// Display Resource conflict
#define ADL_ERR_RESOURCE_CONFLICT                -12
//Failed to update some of the values. Can be returned by set request that include multiple values if not all values were successfully committed.
#define ADL_ERR_SET_INCOMPLETE                 -20
/// There's no Linux XDisplay in Linux Console environment
#define ADL_ERR_NO_XDISPLAY                    -21

// @}
```

这段代码定义了一个名为“define_display_type”的组，其中包括了多种不同的显示类型，如 Monitor、CRT、Television、LCD 和 Digital Flat Panel。这些显示类型都被定义为位图显示类型，使用了一系列常量来标识它们，包括 #define ADL_DT_MONITOR 0、#define ADL_DT_TELEVISION 1、#define ADL_DT_LCD_PANEL 2 和 #define ADL_DT_DIGITAL_FLAT_PANEL 3。

该代码还定义了一个名为“Display Type”的组，其中包括了多种不同的显示类型，如 Monitor、CRT 和 Digital Flat Panel。这些显示类型都被定义为位图显示类型，使用了一系列常量来标识它们，包括 #define ADL_DT_MONITOR 0、#define ADL_DT_CRT 1 和 #define ADL_DT_LCD_PANEL 2。

最后，该代码还定义了一个名为“componment_video_display_type”的组，其中包括了多种不同的显示类型，如 Component Video 和 Digital Flat Panel。这些显示类型都被定义为位图显示类型，使用了一系列常量来标识它们，包括 #define ADL_DT_COMPONENT_VIDEO 0 和 #define ADL_DT_DIGITAL_FLAT_PANEL 2。


```cpp
/// </A>

/// \defgroup define_display_type Display Type
/// Define Monitor/CRT display type
// @{
/// Define Monitor display type
#define ADL_DT_MONITOR                  0
/// Define TV display type
#define ADL_DT_TELEVISION                    1
/// Define LCD display type
#define ADL_DT_LCD_PANEL                       2
/// Define DFP display type
#define ADL_DT_DIGITAL_FLAT_PANEL        3
/// Define Componment Video display type
#define ADL_DT_COMPONENT_VIDEO               4
```

这段代码定义了一个名为 Projector 的显示类型，对应的显示连接类型包括显示输出类型，如文本、图像、视频和模拟输出等。其中，ADL_DT_PROJECTOR定义了未知显示输出类型，ADL_DOT_UNKNOWN。接着，定义了一系列显示输出类型，如复合视频输出类型为 ADL_DOT_COMPOSITE，模拟视频输出类型为 ADL_DOT_ANALOG，等等。最后，还定义了数字显示输出类型，如 SVideo 输出类型为 ADL_DOT_SVIDEO。


```cpp
/// Define Projector display type
#define ADL_DT_PROJECTOR                       5
// @}

/// \defgroup define_display_connection_type Display Connection Type
// @{
/// Define unknown display output type
#define ADL_DOT_UNKNOWN                0
/// Define composite display output type
#define ADL_DOT_COMPOSITE            1
/// Define SVideo display output type
#define ADL_DOT_SVIDEO                2
/// Define analog display output type
#define ADL_DOT_ANALOG                3
/// Define digital display output type
```

这段代码定义了一个名为 ADL_DOT_DIGITAL 的头文件，其值为 4。接着定义了一个名为 Display 的头文件，其中包含了一系列与颜色相关的定义，如颜色类型、颜色来源、颜色亮度、颜色对比度和颜色饱和度等。这些定义用于在代码中描述颜色，以便程序员可以使用它们来调整程序的外观。

在 Display 头文件中，定义了五个与颜色相关的宏，分别使用了数字编号 0 到 4。这些宏定义了与颜色相关的四个字段，分别是：

```cpp
#define ADL_DISPLAY_COLOR_BRIGHTNESS    (1 << 0)
#define ADL_DISPLAY_COLOR_CONTRAST    (1 << 1)
#define ADL_DISPLAY_COLOR_SATURATION    (1 << 2)
#define ADL_DISPLAY_COLOR_HUE        (1 << 3)
```

这些宏分别表示了颜色亮度、对比度、饱和度和色相。

接下来的代码定义了一个名为 ADL_DISPLAY_COLOR_TEMPERATURE 的宏，但其值为基于 EDID（电子差分）的温度源。

```cpp
/// Color Temperature Source is EDID
#define ADL_DISPLAY_COLOR_TEMPERATURE_SOURCE_EDID    (1 << 5)
```

接下来的代码定义了一个名为 ADL_DISPLAY_COLOR_TEMPERATURE 的宏，但其值为基于用户设置的温度源。

```cpp
/// Color Temperature Source is User
```

这里定义了两个与上面定义的宏同名的符号，分别表示基于 EDID 和基于用户设置的温度源。


```cpp
#define ADL_DOT_DIGITAL                4
// @}

/// \defgroup define_color_type Display Color Type and Source
/// Define  Display Color Type and Source
// @{
#define ADL_DISPLAY_COLOR_BRIGHTNESS    (1 << 0)
#define ADL_DISPLAY_COLOR_CONTRAST    (1 << 1)
#define ADL_DISPLAY_COLOR_SATURATION    (1 << 2)
#define ADL_DISPLAY_COLOR_HUE        (1 << 3)
#define ADL_DISPLAY_COLOR_TEMPERATURE    (1 << 4)

/// Color Temperature Source is EDID
#define ADL_DISPLAY_COLOR_TEMPERATURE_SOURCE_EDID    (1 << 5)
/// Color Temperature Source is User
```

这段代码定义了一个名为 `ADL_DISPLAY_COLOR_TEMPERATURE_SOURCE_USER` 的宏，其值为 `(1 << 6)`。接下来，定义了一个名为 `ADL_DISPLAY_ADJUST_OVERSCAN` 的宏，其值为 `(1 << 0)`；定义了一个名为 `ADL_DISPLAY_ADJUST_VERT_POS` 的宏，其值为 `(1 << 1)`；定义了一个名为 `ADL_DISPLAY_ADJUST_HOR_POS` 的宏，其值为 `(1 << 2)`；定义了一个名为 `ADL_DISPLAY_ADJUST_VERT_SIZE` 的宏，其值为 `(1 << 3)`；定义了一个名为 `ADL_DISPLAY_ADJUST_HOR_SIZE` 的宏，其值为 `(1 << 4)`；定义了一个名为 `ADL_DISPLAY_ADJUST_SIZEPOS` 的宏，其值为 `(ADL_DISPLAY_ADJUST_VERT_POS | ADL_DISPLAY_ADJUST_HOR_POS | ADL_DISPLAY_ADJUST_VERT_SIZE | ADL_DISPLAY_ADJUST_HOR_SIZE)`；定义了一个名为 `ADL_DISPLAY_CUSTOMMODES` 的宏，其值为 `(1 << 5)`；定义了一个名为 `ADL_DISPLAY_ADJUST_UNDERSCAN` 的宏，其值为 `(1 << 6)`。


```cpp
#define ADL_DISPLAY_COLOR_TEMPERATURE_SOURCE_USER    (1 << 6)
// @}

/// \defgroup define_adjustment_capabilities Display Adjustment Capabilities
/// Display adjustment capabilities values.  Returned by ADL_Display_AdjustCaps_Get
// @{
#define ADL_DISPLAY_ADJUST_OVERSCAN        (1 << 0)
#define ADL_DISPLAY_ADJUST_VERT_POS        (1 << 1)
#define ADL_DISPLAY_ADJUST_HOR_POS        (1 << 2)
#define ADL_DISPLAY_ADJUST_VERT_SIZE        (1 << 3)
#define ADL_DISPLAY_ADJUST_HOR_SIZE        (1 << 4)
#define ADL_DISPLAY_ADJUST_SIZEPOS        (ADL_DISPLAY_ADJUST_VERT_POS | ADL_DISPLAY_ADJUST_HOR_POS | ADL_DISPLAY_ADJUST_VERT_SIZE | ADL_DISPLAY_ADJUST_HOR_SIZE)
#define ADL_DISPLAY_CUSTOMMODES            (1<<5)
#define ADL_DISPLAY_ADJUST_UNDERSCAN        (1<<6)
// @}

```

这段代码定义了一系列与ADL（自动调整亮度）显示功能相关的头文件和常量。主要作用是定义了ADL显示功能的支持，包括高动态范围（HDR）显示。在未来的RandR（Rendering over Distance）版本中，这个功能可能会得到更好的支持。

具体来说，这段代码主要作用于定义了以下桌面配置文件标志：

1. ADL_DESKTOPCONFIG_UNKNOWN：未知桌面配置。
2. ADL_DESKTOPCONFIG_SINGLE：单屏模式。
3. ADL_DESKTOPCONFIG_CLONE：克隆模式。
4. ADL_DESKTOPCONFIG_BIGDESK_H：大显示器（Big Desktop）水平模式。
5. ADL_DESKTOPCONFIG_BIGDESK_V：大显示器（Big Desktop）垂直模式。

然后，通过将这些常量组合，可以实现对ADL显示功能的支持。例如，通过将ADL_DESKTOPCONFIG_BIGDESK_H和ADL_DESKTOPCONFIG_BIGDESK_V同时设置为1，可以获得大显示器水平/垂直模式。


```cpp
///Down-scale support
#define ADL_DISPLAY_CAPS_DOWNSCALE        (1 << 0)

/// Sharpness support
#define ADL_DISPLAY_CAPS_SHARPNESS      (1 << 0)

/// \defgroup define_desktop_config Desktop Configuration Flags
/// These flags are used by ADL_DesktopConfig_xxx
/// \deprecated This API has been deprecated because it was only used for RandR 1.1 (Red Hat 5.x) distributions which is now not supported.
// @{
#define ADL_DESKTOPCONFIG_UNKNOWN    0          /* UNKNOWN desktop config   */
#define ADL_DESKTOPCONFIG_SINGLE     (1 <<  0)    /* Single                   */
#define ADL_DESKTOPCONFIG_CLONE      (1 <<  2)    /* Clone                    */
#define ADL_DESKTOPCONFIG_BIGDESK_H  (1 <<  4)    /* Big Desktop Horizontal   */
#define ADL_DESKTOPCONFIG_BIGDESK_V  (1 <<  5)    /* Big Desktop Vertical     */
```

这段代码定义了一系列常量，用于描述桌面配置中的大显示配置。常量名称为ADL_DESKTOPCONFIG_BIGDESK_HR和ADL_DESKTOPCONFIG_BIGDESK_VR，分别表示大显示配置和高分辨率模式。接下来是其他一些常量，描述了在不同大小的显示上如何配置桌面。

例如，ADL_DESKTOPCONFIG_RANDR12定义了RandR 1.2的多屏显示模式。最后，定义了一个名为ADL_MAX_DISPLAY_NAME的常量，表示ADL中最大可用的显示名称长度。

整个定义的目的是在代码中为特定硬件配置提供明确的说明，使得开发人员更容易理解代码的意图。


```cpp
#define ADL_DESKTOPCONFIG_BIGDESK_HR (1 <<  6)    /* Big Desktop Reverse Horz */
#define ADL_DESKTOPCONFIG_BIGDESK_VR (1 <<  7)    /* Big Desktop Reverse Vert */
#define ADL_DESKTOPCONFIG_RANDR12    (1 <<  8)    /* RandR 1.2 Multi-display */
// @}

/// needed for ADLDDCInfo structure
#define ADL_MAX_DISPLAY_NAME                                256

/// \defgroup define_edid_flags Values for ulDDCInfoFlag
/// defines for ulDDCInfoFlag EDID flag
// @{
#define ADL_DISPLAYDDCINFOEX_FLAG_PROJECTORDEVICE       (1 << 0)
#define ADL_DISPLAYDDCINFOEX_FLAG_EDIDEXTENSION         (1 << 1)
#define ADL_DISPLAYDDCINFOEX_FLAG_DIGITALDEVICE         (1 << 2)
#define ADL_DISPLAYDDCINFOEX_FLAG_HDMIAUDIODEVICE       (1 << 3)
```

这段代码定义了一系列常量，用于指定ADL(Advanced Display Language)中显示器连接器的类型。

ADL_DISPLAYDDCINFOEX_FLAG_SUPPORTS_AI表示是否支持AI(人工智能)显卡驱动程序。
ADL_DISPLAYDDCINFOEX_FLAG_SUPPORT_xvYCC601和ADL_DISPLAYDDCINFOEX_FLAG_SUPPORT_xvYCC709分别表示是否支持XVGA和XGA显卡驱动程序。

定义了一系列ADL_DISPLAYCONDEVICE类型，包括ADL_DISPLAY_CONTYPE_UNKNOWN、ADL_DISPLAY_CONTYPE_VGA、ADL_DISPLAY_CONTYPE_DVI_D、ADL_DISPLAY_CONTYPE_DVI_I、ADL_DISPLAY_CONTYPE_ATICVDONGLE_NTSC、ADL_DISPLAY_CONTYPE_ATICVDONGLE_JPN、ADL_DISPLAY_CONTYPE_ATICVDONGLE_NONI2C_JPN等等。

最后，这些定义用于定义ADL_DISPLAYDDCINFOEX结构体，该结构体用于在ADL中描述显示器连接器。


```cpp
#define ADL_DISPLAYDDCINFOEX_FLAG_SUPPORTS_AI           (1 << 4)
#define ADL_DISPLAYDDCINFOEX_FLAG_SUPPORT_xvYCC601      (1 << 5)
#define ADL_DISPLAYDDCINFOEX_FLAG_SUPPORT_xvYCC709      (1 << 6)
// @}

/// \defgroup define_displayinfo_connector Display Connector Type
/// defines for ADLDisplayInfo.iDisplayConnector
// @{
#define ADL_DISPLAY_CONTYPE_UNKNOWN                 0
#define ADL_DISPLAY_CONTYPE_VGA                     1
#define ADL_DISPLAY_CONTYPE_DVI_D                   2
#define ADL_DISPLAY_CONTYPE_DVI_I                   3
#define ADL_DISPLAY_CONTYPE_ATICVDONGLE_NTSC        4
#define ADL_DISPLAY_CONTYPE_ATICVDONGLE_JPN         5
#define ADL_DISPLAY_CONTYPE_ATICVDONGLE_NONI2C_JPN  6
```

这段代码定义了一系列关于显示适配器（ADL）的显示连接器（DISPLAY CONTEXT）类型的常量，涵盖了不同的显示适配器类型，包括：

1. ATOMICVDONGLE Non-I2C：这种类型的显示适配器用于在计算机内部进行数据传输，通常用于服务器和数据存储设备。
2. PROPRIETARY：这种类型的显示适配器可以通过主机控制器进行识别，并可以支持多种设备。
3. HDMI Type A：这种类型的显示适配器支持High Definition Multimedia Interface（HDMI）规范，用于在电视和计算机之间传输音频和视频信号。
4. HDMI Type B：这种类型的显示适配器同样支持HDMI规范，但比Type A的画质稍低。
5. SVIDEO：这种类型的显示适配器用于在计算机和电视之间传输SVGA格式的视频数据。
6. COMPOSITE：这种类型的显示适配器支持复合视频模式，通常用于在电视中播放计算机视频。
7. RCA Three Component Component：这种类型的显示适配器通过三个 Component（即：红、白和黑）连接到扬声器或显示器。
8. DISPLAYPORT：这种类型的显示适配器支持DISPLAYPORT规范，用于在计算机和显示器之间传输音频和视频信号。
9. EDP：这种类型的显示适配器支持Open Editing Device（EDP）规范，用于在电视中编辑音频和视频信号。
10. WIRELESS DISPLAY：这种类型的显示适配器支持无线显示技术，通常用于移动设备与显示设备之间的连接。


```cpp
#define ADL_DISPLAY_CONTYPE_ATICVDONGLE_NONI2C_NTSC 7
#define ADL_DISPLAY_CONTYPE_PROPRIETARY                8
#define ADL_DISPLAY_CONTYPE_HDMI_TYPE_A             10
#define ADL_DISPLAY_CONTYPE_HDMI_TYPE_B             11
#define ADL_DISPLAY_CONTYPE_SVIDEO                   12
#define ADL_DISPLAY_CONTYPE_COMPOSITE               13
#define ADL_DISPLAY_CONTYPE_RCA_3COMPONENT          14
#define ADL_DISPLAY_CONTYPE_DISPLAYPORT             15
#define ADL_DISPLAY_CONTYPE_EDP                     16
#define ADL_DISPLAY_CONTYPE_WIRELESSDISPLAY         17
// @}

/// TV Capabilities and Standards
/// \defgroup define_tv_caps TV Capabilities and Standards
/// \deprecated Dropping support for TV displays
```

这段代码定义了一系列的 `#define` 预处理指令，用于定义不同的电视标准。

具体来说，这些预处理指令定义了如下内容：

- `ADL_TV_STANDARDS`  定义为 `(1 << 0)`。这意味着标准 NTSC 模式(也称为日本标准)将被支持。
- `ADL_TV_SCART`  定义为 `(1 << 1)`。这意味着支持南李孝式制(也称为 SCART)的电视标准将被支持。

此外，还定义了一些其他预处理指令，如 `ADL_STANDARD_NTSC_M`、`ADL_STANDARD_NTSC_JPN` 等，用于定义 NTSC 标准。这些指令定义了 PAL 和 NTSC 标准下的 color space，以及可以被编程的扫描行数量等。


```cpp
// @{
#define ADL_TV_STANDARDS            (1 << 0)
#define ADL_TV_SCART                (1 << 1)

/// TV Standards Definitions
#define ADL_STANDARD_NTSC_M        (1 << 0)
#define ADL_STANDARD_NTSC_JPN        (1 << 1)
#define ADL_STANDARD_NTSC_N        (1 << 2)
#define ADL_STANDARD_PAL_B        (1 << 3)
#define ADL_STANDARD_PAL_COMB_N        (1 << 4)
#define ADL_STANDARD_PAL_D        (1 << 5)
#define ADL_STANDARD_PAL_G        (1 << 6)
#define ADL_STANDARD_PAL_H        (1 << 7)
#define ADL_STANDARD_PAL_I        (1 << 8)
#define ADL_STANDARD_PAL_K        (1 << 9)
```

这段代码定义了一系列宏，用于定义ADL标准视频模式下的不同模式。这些模式包括：PAL K1、PAL L、PAL M、PAL N、PAL SECAM K、PAL SECAM K1和PAL SECAM L。这些宏的值都为1，表示对应的模式是存在的。

接下来的代码定义了一个名为“ADL\_CUSTOMIZEDMODEFLAG”的整型变量，它包含了一个位掩，用于指示哪些模式支持。这个位掩的值为1，表示支持所有定义的模式。

最后，定义了一系列用于定义视频模式下的定制模式的函数，如\_DECLARE\_VIDEO\_CUSTOM\_MODE()，用于将定义的模式与整型变量进行绑定，使得在定义的视频模式中，这些模式可以被安全地使用。


```cpp
#define ADL_STANDARD_PAL_K1        (1 << 10)
#define ADL_STANDARD_PAL_L        (1 << 11)
#define ADL_STANDARD_PAL_M        (1 << 12)
#define ADL_STANDARD_PAL_N        (1 << 13)
#define ADL_STANDARD_PAL_SECAM_D    (1 << 14)
#define ADL_STANDARD_PAL_SECAM_K    (1 << 15)
#define ADL_STANDARD_PAL_SECAM_K1    (1 << 16)
#define ADL_STANDARD_PAL_SECAM_L    (1 << 17)
// @}


/// \defgroup define_video_custom_mode Video Custom Mode flags
/// Component Video Custom Mode flags.  This is used by the iFlags parameter in ADLCustomMode
// @{
#define ADL_CUSTOMIZEDMODEFLAG_MODESUPPORTED    (1 << 0)
```

这段代码定义了一个名为 ADL_CUSTOMIZEDMODEFLAG 的头文件，其中包含了一系列常量定义。这些常量定义了 DDC（数字条件驱动器）信息列中的标识符，包括：

* ADL_CUSTOMIZEDMODEFLAG_NOTDELETETABLE：表示是否删除指定的数据设备，值为 1 时删除，值为 0 时保留。
* ADL_CUSTOMIZEDMODEFLAG_INSERTBYDRIVER：表示通过驱动程序插插入数据设备，值为 1 时通过驱动程序，值为 0 时在系统内部插入。
* ADL_CUSTOMIZEDMODEFLAG_INTERLACED：表示是否启用基于内存的数据设备，值为 1 时启用，值为 0 时禁用。
* ADL_CUSTOMIZEDMODEFLAG_BASEMODE：表示数据设备的基准模式，值为 0 时设置为默认基准模式，值为 1 时设置为具有特定基准模式的数据设备。

此外，还定义了 ADL_DISPLAYDDCINFOEX_FLAG，用于表示在 DDC 信息列中关于显示器的相关信息。


```cpp
#define ADL_CUSTOMIZEDMODEFLAG_NOTDELETETABLE    (1 << 1)
#define ADL_CUSTOMIZEDMODEFLAG_INSERTBYDRIVER    (1 << 2)
#define ADL_CUSTOMIZEDMODEFLAG_INTERLACED    (1 << 3)
#define ADL_CUSTOMIZEDMODEFLAG_BASEMODE        (1 << 4)
// @}

/// \defgroup define_ddcinfoflag Values used for DDCInfoFlag
/// ulDDCInfoFlag field values used by the ADLDDCInfo structure
// @{
#define ADL_DISPLAYDDCINFOEX_FLAG_PROJECTORDEVICE    (1 << 0)
#define ADL_DISPLAYDDCINFOEX_FLAG_EDIDEXTENSION        (1 << 1)
#define ADL_DISPLAYDDCINFOEX_FLAG_DIGITALDEVICE        (1 << 2)
#define ADL_DISPLAYDDCINFOEX_FLAG_HDMIAUDIODEVICE    (1 << 3)
#define ADL_DISPLAYDDCINFOEX_FLAG_SUPPORTS_AI        (1 << 4)
#define ADL_DISPLAYDDCINFOEX_FLAG_SUPPORT_xvYCC601    (1 << 5)
```

这段代码定义了一个宏，名为 ADL_DISPLAYDDCINFOEX_FLAG_SUPPORT_xvYCC709，值为 (1 << 6)。

这个宏告诉我们在定义 ADL_DISPLAY_CONTYPE_ATICVDONGLE_JP 和 ADL_DISPLAY_CONTYPE_ATICVDONGLE_NONI2C 时，使用或忽略 D1、D2、D3 和 D4 成员变量。具体来说，如果定义了 ADL_DISPLAY_CV_DONGLE_D1，那么在定义 ADL_DISPLAY_CONTYPE_ATICVDONGLE_JP 时，必须也定义 ADL_DISPLAY_CV_DONGLE_D1。类似地，如果定义了 ADL_DISPLAY_CV_DONGLE_D2 或 ADL_DISPLAY_CV_DONGLE_D3，那么在定义 ADL_DISPLAY_CONTYPE_ATICVDONGLE_JP 或 ADL_DISPLAY_CONTYPE_ATICVDONGLE_NONI2C 时，必须也定义 ADL_DISPLAY_CV_DONGLE_D2 或 ADL_DISPLAY_CV_DONGLE_D3。

定义 ADL_DISPLAY_CV_DONGLE_D5 的作用是告诉编译器在定义 ADL_DISPLAY_CONTYPE_ATICVDONGLE_NONI2C 时，使用或忽略 D1、D2、D3 和 D4 成员变量。


```cpp
#define ADL_DISPLAYDDCINFOEX_FLAG_SUPPORT_xvYCC709    (1 << 6)
// @}

/// \defgroup define_cv_dongle Values used by ADL_CV_DongleSettings_xxx
/// The following is applicable to ADL_DISPLAY_CONTYPE_ATICVDONGLE_JP and ADL_DISPLAY_CONTYPE_ATICVDONGLE_NONI2C_D only
/// \deprecated Dropping support for Component Video displays
// @{
#define ADL_DISPLAY_CV_DONGLE_D1          (1 << 0)
#define ADL_DISPLAY_CV_DONGLE_D2          (1 << 1)
#define ADL_DISPLAY_CV_DONGLE_D3          (1 << 2)
#define ADL_DISPLAY_CV_DONGLE_D4          (1 << 3)
#define ADL_DISPLAY_CV_DONGLE_D5          (1 << 4)

/// The following is applicable to ADL_DISPLAY_CONTYPE_ATICVDONGLE_NA and ADL_DISPLAY_CONTYPE_ATICVDONGLE_NONI2C only

```

这段代码定义了一系列宏，用于定义和控制显示CV光束的方向和亮度。具体来说，定义了以下几组宏：

```cpp
#define ADL_DISPLAY_CV_DONGLE_480I        (1 << 0)
#define ADL_DISPLAY_CV_DONGLE_480P        (1 << 1)
#define ADL_DISPLAY_CV_DONGLE_540P        (1 << 2)
#define ADL_DISPLAY_CV_DONGLE_720P        (1 << 3)
#define ADL_DISPLAY_CV_DONGLE_1080I       (1 << 4)
#define ADL_DISPLAY_CV_DONGLE_1080P       (1 << 5)
#define ADL_DISPLAY_CV_DONGLE_16_9        (1 << 6)
#define ADL_DISPLAY_CV_DONGLE_720P50      (1 << 7)
#define ADL_DISPLAY_CV_DONGLE_1080I25     (1 << 8)
#define ADL_DISPLAY_CV_DONGLE_576I25      (1 << 9)
#define ADL_DISPLAY_CV_DONGLE_576P50      (1 << 10)
#define ADL_DISPLAY_CV_DONGLE_1080P24      (1 << 11)
#define ADL_DISPLAY_CV_DONGLE_1080P25      (1 << 12)
#define ADL_DISPLAY_CV_DONGLE_1080P30      (1 << 13)
#define ADL_DISPLAY_CV_DONGLE_1080P50      (1 << 14)
```

这些宏用于控制CV光束的方向和亮度。通过设置不同的组合，可以实现不同的投影效果。例如，通过将`ADL_DISPLAY_CV_DONGLE_480I`和`ADL_DISPLAY_CV_DONGLE_540P`设置为1，可以设置为480英寸的分辨率，同时使用480针的显示效果。


```cpp
#define ADL_DISPLAY_CV_DONGLE_480I        (1 << 0)
#define ADL_DISPLAY_CV_DONGLE_480P        (1 << 1)
#define ADL_DISPLAY_CV_DONGLE_540P        (1 << 2)
#define ADL_DISPLAY_CV_DONGLE_720P        (1 << 3)
#define ADL_DISPLAY_CV_DONGLE_1080I       (1 << 4)
#define ADL_DISPLAY_CV_DONGLE_1080P       (1 << 5)
#define ADL_DISPLAY_CV_DONGLE_16_9        (1 << 6)
#define ADL_DISPLAY_CV_DONGLE_720P50      (1 << 7)
#define ADL_DISPLAY_CV_DONGLE_1080I25     (1 << 8)
#define ADL_DISPLAY_CV_DONGLE_576I25      (1 << 9)
#define ADL_DISPLAY_CV_DONGLE_576P50      (1 << 10)
#define ADL_DISPLAY_CV_DONGLE_1080P24      (1 << 11)
#define ADL_DISPLAY_CV_DONGLE_1080P25      (1 << 12)
#define ADL_DISPLAY_CV_DONGLE_1080P30      (1 << 13)
#define ADL_DISPLAY_CV_DONGLE_1080P50      (1 << 14)
```

这段代码定义了一系列与显示模式相关的宏，用于在Robotics supportive环境（ROS）中设置力的格式。主要作用是定义了各种力的显示格式，以便在ROS中使用，从而使代码更易于理解和维护。


```cpp
// @}

/// \defgroup define_formats_ovr    Formats Override Settings
/// Display force modes flags
// @{
///
#define ADL_DISPLAY_FORMAT_FORCE_720P        0x00000001
#define ADL_DISPLAY_FORMAT_FORCE_1080I        0x00000002
#define ADL_DISPLAY_FORMAT_FORCE_1080P        0x00000004
#define ADL_DISPLAY_FORMAT_FORCE_720P50        0x00000008
#define ADL_DISPLAY_FORMAT_FORCE_1080I25    0x00000010
#define ADL_DISPLAY_FORMAT_FORCE_576I25        0x00000020
#define ADL_DISPLAY_FORMAT_FORCE_576P50        0x00000040
#define ADL_DISPLAY_FORMAT_FORCE_1080P24    0x00000080
#define ADL_DISPLAY_FORMAT_FORCE_1080P25    0x00000100
```

这是一组C语言代码，定义了ADL（高级display驱动程序）中与显示模式相关的标志。这些标志用于控制显示的分辨率、帧率、连接性以及模式选择等方面。以下是这些标志的简要解释：

1. ADL_DISPLAY_FORMAT_FORCE_1080P30：表示1080P分辨率、30Hz刷新率和HDRCP（高动态范围内容计划）模式。
2. ADL_DISPLAY_FORMAT_FORCE_1080P50：表示1080P分辨率、50Hz刷新率和HDRCP模式。
3. ADL_DISPLAY_FORMAT_CVDONGLEOVERIDE：表示在CD音箱上以最佳质量显示1080P分辨率、30Hz刷新率和HDRCP模式。
4. ADL_DISPLAY_FORMAT_CVMODEUNDERSCAN：表示在计算机上以最小化画质显示1080P分辨率、30Hz刷新率和HDRCP模式。
5. ADL_DISPLAY_FORMAT_FORCECONNECT_SUPPORTED：表示连接性支持，允许在计算机上使用外部显示器。
6. ADL_DISPLAY_FORMAT_RESTRICT_FORMAT_SELECTION：表示仅允许在某些计算机配置中选择显示模式。
7. ADL_DISPLAY_FORMAT_SETASPECRATIO：表示根据设置的显示分辨率自动调整亮度和对比度以获得最佳视觉效果。
8. ADL_DISPLAY_FORMAT_FORCEMODES：表示强制使用某种显示模式，而不是让用户手动选择。
9. ADL_DISPLAY_FORMAT_LCDRTCCOEFF：表示低对比度模式，在这种模式下，灯丝亮度占据大部分像素，以降低功耗。

OD5（Advanced overDisplay5）是ADL的一个驱动程序，用于在电视和其他显示设备上提供更加逼真的HDR（高动态范围）体验。这些定义为了一系列与显示相关的标志，用于控制在不同设备上如何显示1080P分辨率、帧率和HDRCP内容。


```cpp
#define ADL_DISPLAY_FORMAT_FORCE_1080P30    0x00000200
#define ADL_DISPLAY_FORMAT_FORCE_1080P50    0x00000400

///< Below are \b EXTENDED display mode flags

#define ADL_DISPLAY_FORMAT_CVDONGLEOVERIDE  0x00000001
#define ADL_DISPLAY_FORMAT_CVMODEUNDERSCAN  0x00000002
#define ADL_DISPLAY_FORMAT_FORCECONNECT_SUPPORTED  0x00000004
#define ADL_DISPLAY_FORMAT_RESTRICT_FORMAT_SELECTION 0x00000008
#define ADL_DISPLAY_FORMAT_SETASPECRATIO 0x00000010
#define ADL_DISPLAY_FORMAT_FORCEMODES    0x00000020
#define ADL_DISPLAY_FORMAT_LCDRTCCOEFF   0x00000040
// @}

/// Defines used by OD5
```

这段代码定义了一系列的`#define`，用于定义不同种类的总线（Bustype）类型。这些定义包括PCI（Platform Compute Input）总线、AGP（Advanced Graphics Port）总线、PCIe（Peripheral Component Interconnect Express）总线、PCIe GEN2（Peripheral Component Interconnect Express 2nd Generation）总线、PCIe GEN3（Peripheral Component Interconnect Express 3rd Generation）总线和PCIe GEN4（Peripheral Component Interconnect Express 4th Generation）总线。

此外，还定义了一个名为`ADL_PM_PARAM_DONT_CHANGE`的常量，其值为0。这个常量在后面的代码中被频繁使用，没有进一步的解释。


```cpp
#define ADL_PM_PARAM_DONT_CHANGE    0

/// The following defines Bus types
// @{
#define ADL_BUSTYPE_PCI           0       /* PCI bus                          */
#define ADL_BUSTYPE_AGP           1       /* AGP bus                          */
#define ADL_BUSTYPE_PCIE          2       /* PCI Express bus                  */
#define ADL_BUSTYPE_PCIE_GEN2     3       /* PCI Express 2nd generation bus   */
#define ADL_BUSTYPE_PCIE_GEN3     4       /* PCI Express 3rd generation bus   */
#define ADL_BUSTYPE_PCIE_GEN4     5       /* PCI Express 4th generation bus   */
// @}

/// \defgroup define_ws_caps    Workstation Capabilities
/// Workstation values
// @{

```

这段代码定义了一系列与立体声（Stereo）输出相关的预设常量，用于判断工作站卡是否支持立体声输出。这些预设常量通过组合不同的常量位来设置或清除工作站卡的立体声输出设置。以下是每个预设常量的含义：

```cpp
#define ADL_STEREO_SUPPORTED        (1 << 2)  // 工作区卡是否支持立体声输出？
#define ADL_STEREO_BLUE_LINE        (1 << 3)  // 是否支持通过 "蓝线" 的立体声输出？
#define ADL_STEREO_OFF                0          // 关闭立体声输出
#define ADL_STEREO_ACTIVE             (1 << 1)  // 工作区卡支持立体声输出
#define ADL_STEREO_HORIZONTAL        (1 << 30) // 支持使用水平 interpolate 的立体声输出
#define ADL_STEREO_VERTICAL          (1 << 31) // 支持使用垂直 interpolate 的立体声输出
#define ADL_STEREO_PASSIVE              (1 << 6)  // 是否使用被动式的立体声输出？
#define ADL_STEREO_AUTO_HORIZONTAL    (1 << 30) // 自动使用水平 interpolate 的立体声输出
#define ADL_STEREO_AUTO_VERTICAL    (1 << 31) // 自动使用垂直 interpolate 的立体声输出
```

此外，还有一系列其他预设常量，如：

```cpp
#define ADL_STEREO_BIDI             (1 << 24) // 支持使用 BIDI 立体声输出？
#define ADL_STEREO_RGB             (1 << 25) // 支持使用 RGB 立体声输出？
#define ADL_STEREO_5P           (1 << 30) // 支持使用 5P 电缆连接立体声？
#define ADL_STEREO_7P           (1 << 31) // 支持使用 7P 电缆连接立体声？
```

这些预设常量用于根据工作区卡的具体硬件情况，判断是否支持使用立体声输出，并设置相应的预设常量。


```cpp
/// This value indicates that the workstation card supports active stereo though stereo output connector
#define ADL_STEREO_SUPPORTED        (1 << 2)
/// This value indicates that the workstation card supports active stereo via "blue-line"
#define ADL_STEREO_BLUE_LINE        (1 << 3)
/// This value is used to turn off stereo mode.
#define ADL_STEREO_OFF                0
/// This value indicates that the workstation card supports active stereo.  This is also used to set the stereo mode to active though the stereo output connector
#define ADL_STEREO_ACTIVE             (1 << 1)
/// This value indicates that the workstation card supports auto-stereo monitors with horizontal interleave. This is also used to set the stereo mode to use the auto-stereo monitor with horizontal interleave
#define ADL_STEREO_AUTO_HORIZONTAL    (1 << 30)
/// This value indicates that the workstation card supports auto-stereo monitors with vertical interleave. This is also used to set the stereo mode to use the auto-stereo monitor with vertical interleave
#define ADL_STEREO_AUTO_VERTICAL    (1 << 31)
/// This value indicates that the workstation card supports passive stereo, ie. non stereo sync
#define ADL_STEREO_PASSIVE              (1 << 6)
/// This value indicates that the workstation card supports auto-stereo monitors with vertical interleave. This is also used to set the stereo mode to use the auto-stereo monitor with vertical interleave
```

这段代码定义了一系列常量，用于表示工作站是否支持立体声和局域网音频。以下是每个常量的含义和作用：

```cpp
#define ADL_STEREO_PASSIVE_HORIZ 1 << 7,  // 静音型立体声，工作范围为00000000~00001277
#define ADL_STEREO_PASSIVE_VERT   1 << 8,  // 支持垂直 interleave，工作范围为00000000~00001277
#define ADL_STEREO_AUTO_SAMSUNG  1 << 11,  // 支持三星的自动立体声，工作范围为00000000~00001277
#define ADL_STEREO_AUTO_TSL      1 << 12,  // 支持 Tridility 的自动立体声，工作范围为00000000~00001277
#define ADL_DEEPBITDEPTH_10BPP_SUPPORTED 1 << 5,  // 支持10位深度的 DeepBitDepth，工作范围为00000000~00001277
#define ADL_8BIT_GREYSCALE_SUPPORTED 1 << 9,  // 支持 8 位灰度，工作范围为00000000~00001277
#define ADL_CUSTOM_TIMING_SUPPORTED 1 << 10,  // 支持自定义时钟设置，工作范围为00000000~00001277
```

这些常量的值用于控制工作站是否支持立体声和局域网音频。常量中的数字表示支持的功能。如果数字的二进制表示中包含 1，则表示支持指定的功能。如果数字的二进制表示中包含 0，则表示不支持指定的功能。


```cpp
#define ADL_STEREO_PASSIVE_HORIZ        (1 << 7)
/// This value indicates that the workstation card supports auto-stereo monitors with vertical interleave. This is also used to set the stereo mode to use the auto-stereo monitor with vertical interleave
#define ADL_STEREO_PASSIVE_VERT         (1 << 8)
/// This value indicates that the workstation card supports auto-stereo monitors with Samsung.
#define ADL_STEREO_AUTO_SAMSUNG        (1 << 11)
/// This value indicates that the workstation card supports auto-stereo monitors with Tridility.
#define ADL_STEREO_AUTO_TSL         (1 << 12)
/// This value indicates that the workstation card supports DeepBitDepth (10 bpp)
#define ADL_DEEPBITDEPTH_10BPP_SUPPORTED   (1 << 5)

/// This value indicates that the workstation supports 8-Bit Grayscale
#define ADL_8BIT_GREYSCALE_SUPPORTED   (1 << 9)
/// This value indicates that the workstation supports CUSTOM TIMING
#define ADL_CUSTOM_TIMING_SUPPORTED   (1 << 10)

```

这段代码定义了一系列关于负载均衡设置的定义，包括三种状态：

1. ADL_WORKSTATION_LOADBALANCING_SUPPORTED：表示支持负载均衡，开启了应用程序中的负载均衡功能。
2. ADL_WORKSTATION_LOADBALANCING_AVAILABLE：表示支持负载均衡，但是否开启了应用程序中的负载均衡功能取决于配置。
3. ADL_WORKSTATION_LOADBALANCING_DISABLED：表示不支持负载均衡，也就是关闭了应用程序中的负载均衡功能。
4. ADL_WORKSTATION_LOADBALANCING_ENABLED：表示支持负载均衡，开启了应用程序中的负载均衡功能。
5. ```cppC
// @}
```：这是一个C语言注释，表示以上定义都已完成。


```cpp
/// Load balancing is supported.
#define ADL_WORKSTATION_LOADBALANCING_SUPPORTED         0x00000001
/// Load balancing is available.
#define ADL_WORKSTATION_LOADBALANCING_AVAILABLE         0x00000002

/// Load balancing is disabled.
#define ADL_WORKSTATION_LOADBALANCING_DISABLED          0x00000000
/// Load balancing is Enabled.
#define ADL_WORKSTATION_LOADBALANCING_ENABLED           0x00000001



// @}

/// \defgroup define_adapterspeed speed setting from the adapter
```

这段代码定义了一些常量，用于控制 ASIC 的运行速度。三个常量分别为：

```cpp
#define ADL_CONTEXT_SPEED_UNFORCED       0        /* default asic running speed */
#define ADL_CONTEXT_SPEED_FORCEHIGH       1        /* asic running speed is forced to high */
#define ADL_CONTEXT_SPEED_FORCELOW       2        /* asic running speed is forced to low */
```

以及：

```cpp
#define ADL_ADAPTER_SPEEDCAPS_SUPPORTED   1 << 0    /* change asic running speed setting is supported */
```

接着定义了一个名为 `ADL_GLSYNC_PORT_UNKNOWN` 的常量，其值为 0。接着定义了一个名为 `ADL_GLSYNC_PORT_BNC` 的常量，其值为 1。

这个代码的作用是定义了一些常量，用于控制 ASIC 的运行速度，以及定义了一个名为 `ADL_ADAPTER_SPEEDCAPS_SUPPORTED` 的常量，表示 GL-Sync 是否支持更改 ASIC 的运行速度设置。


```cpp
// @{
#define ADL_CONTEXT_SPEED_UNFORCED        0        /* default asic running speed */
#define ADL_CONTEXT_SPEED_FORCEHIGH        1        /* asic running speed is forced to high */
#define ADL_CONTEXT_SPEED_FORCELOW        2        /* asic running speed is forced to low */

#define ADL_ADAPTER_SPEEDCAPS_SUPPORTED        (1 << 0)    /* change asic running speed setting is supported */
// @}

/// \defgroup define_glsync Genlock related values
/// GL-Sync port types (unique values)
// @{
/// Unknown port of GL-Sync module
#define ADL_GLSYNC_PORT_UNKNOWN        0
/// BNC port of of GL-Sync module
#define ADL_GLSYNC_PORT_BNC            1
```

这段代码定义了一个名为ADL_GLSYNC_PORT_RJ45PORT1和ADL_GLSYNC_PORT_RJ45PORT2的宏，它们定义了GL-Sync模块的RJ45接口的端口号。同时，定义了一个名为ADL_GLSYNC_CONFIGMASK_NONE的常量，表示ADLGLSyncGenlockConfig中的所有成员都是有效的，以及一个名为ADL_GLSYNC_CONFIGMASK_SIGNALSOURCE的宏，表示ADLGLSyncGenlockConfig.lSignalSource成员是有效的。此外，还定义了一个名为ADL_GLSYNC_CONFIGMASK_SYNCFIELD的宏，表示ADLGLSyncGenlockConfig.iSyncField成员是有效的，以及一个名为ADL_GLSYNC_CONFIGMASK_SAMPLERATE的宏，表示ADLGLSyncGenlockConfig.iSampleRate成员是有效的。


```cpp
/// RJ45(1) port of of GL-Sync module
#define ADL_GLSYNC_PORT_RJ45PORT1    2
/// RJ45(2) port of of GL-Sync module
#define ADL_GLSYNC_PORT_RJ45PORT2    3

// GL-Sync Genlock settings mask (bit-vector)

/// None of the ADLGLSyncGenlockConfig members are valid
#define ADL_GLSYNC_CONFIGMASK_NONE                0
/// The ADLGLSyncGenlockConfig.lSignalSource member is valid
#define ADL_GLSYNC_CONFIGMASK_SIGNALSOURCE        (1 << 0)
/// The ADLGLSyncGenlockConfig.iSyncField member is valid
#define ADL_GLSYNC_CONFIGMASK_SYNCFIELD            (1 << 1)
/// The ADLGLSyncGenlockConfig.iSampleRate member is valid
#define ADL_GLSYNC_CONFIGMASK_SAMPLERATE        (1 << 2)
```

这段代码定义了一个名为 ADLGLSyncGenlockConfig 的结构体，它用于配置 ADLGL（自适应同步）同步 Genlock。Genlock 是一种同步锁，用于在图形渲染管线的合成过程中，确保所有状态都一致，从而获得更好的性能和更好的用户体验。

以下是定义在结构体中的成员：

1. lSyncDelay：同步延迟时间，以同步锁的最小延迟为准。
2. iTriggerEdge：触发边（包括开始、结束和属性）。
3. iScanRateCoeff：扫描率系数，用于计算扫描线速度。
4. lFramelockCntlVector：Framelock 控制器向量，用于控制 Genlock 的锁定状态。

其中：
- 第 3 和第 4 成员是可选的，并不是必须的。
- 第 6 成员用于指定 Genlock 是否启用，如果启用，则第 1 成员（lSyncDelay）表示同步锁的最小延迟；如果禁用，则第 1 成员为 0。

这段代码定义了一个用于配置 ADLGL 同步锁的 Bit-vector，通过这些成员可以设置 Genlock 的延迟、触发边、扫描率系数和锁定状态。


```cpp
/// The ADLGLSyncGenlockConfig.lSyncDelay member is valid
#define ADL_GLSYNC_CONFIGMASK_SYNCDELAY            (1 << 3)
/// The ADLGLSyncGenlockConfig.iTriggerEdge member is valid
#define ADL_GLSYNC_CONFIGMASK_TRIGGEREDGE        (1 << 4)
/// The ADLGLSyncGenlockConfig.iScanRateCoeff member is valid
#define ADL_GLSYNC_CONFIGMASK_SCANRATECOEFF        (1 << 5)
/// The ADLGLSyncGenlockConfig.lFramelockCntlVector member is valid
#define ADL_GLSYNC_CONFIGMASK_FRAMELOCKCNTL        (1 << 6)


// GL-Sync Framelock control mask (bit-vector)

/// Framelock is disabled
#define ADL_GLSYNC_FRAMELOCKCNTL_NONE            0
/// Framelock is enabled
```

这段代码定义了一系列宏，用于控制GL-Sync框架中的片元锁定计数器（framelock counters）的启用和禁用。片元锁定计数器是一个用于确保在多线程场景中，每个线程片元（GL-Sync）的锁定状态（lock status）的计数器。通过同步计数器，可以确保在多线程环境下，各个线程之间的同步进度保持一致。

具体来说，这段代码定义了以下几个宏：

1. `ADL_GLSYNC_FRAMELOCKCNTL_ENABLE`：设置片元锁定计数器为当前状态（enabled）。
2. `ADL_GLSYNC_FRAMELOCKCNTL_DISABLE`：禁用片元锁定计数器。
3. `ADL_GLSYNC_FRAMELOCKCNTL_SWAP_COUNTER_RESET`：设置片元锁定计数器为已设置但尚未确认（reset）。
4. `ADL_GLSYNC_FRAMELOCKCNTL_SWAP_COUNTER_ACK`：设置片元锁定计数器为已完成（ack）。
5. `ADL_GLSYNC_FRAMELOCKCNTL_VERSION_KMD`：设置KMD（GL-Sync管理器）版本。
6. `ADL_GLSYNC_FRAMELOCKCNTL_STATE_ENABLE`：设置片元锁定计数器为启用状态（enabled）。
7. `ADL_GLSYNC_FRAMELOCKCNTL_STATE_KMD`：设置片元锁定计数器为KMD（同步）状态。
8. `ADL_GLSYNC_COUNTER_SWAP`：设置同步计数器为片元锁定计数器（framelock counters）类型。

同步计数器通过GL-Sync框架的片元锁定计数器实现。通过设置这些宏，可以对片元锁定计数器进行启用、禁用、设置为已设置但尚未确认或已完成。同步计数器还可以设置版本和状态，以帮助开发人员了解片元锁定的状态。


```cpp
#define ADL_GLSYNC_FRAMELOCKCNTL_ENABLE            ( 1 << 0)

#define ADL_GLSYNC_FRAMELOCKCNTL_DISABLE        ( 1 << 1)
#define ADL_GLSYNC_FRAMELOCKCNTL_SWAP_COUNTER_RESET    ( 1 << 2)
#define ADL_GLSYNC_FRAMELOCKCNTL_SWAP_COUNTER_ACK    ( 1 << 3)
#define ADL_GLSYNC_FRAMELOCKCNTL_VERSION_KMD    (1 << 4)

#define ADL_GLSYNC_FRAMELOCKCNTL_STATE_ENABLE        ( 1 << 0)
#define ADL_GLSYNC_FRAMELOCKCNTL_STATE_KMD        (1 << 4)

// GL-Sync Framelock counters mask (bit-vector)
#define ADL_GLSYNC_COUNTER_SWAP                ( 1 << 0 )

// GL-Sync Signal Sources (unique values)

```

这段代码定义了一系列GL-Sync信号源（gl-sync source）的定义，用于描述在不同GL-Sync端口上发生的同步信号。其中，定义的信号源类型有四种，分别为：

1. ADL_GLSYNC_SIGNALSOURCE_UNDEFINED：表示未定义的信号源。
2. ADL_GLSYNC_SIGNALSOURCE_FREERUN：表示免费的、可选择的信号源。
3. ADL_GLSYNC_SIGNALSOURCE_BNCPORT：表示BNC（BNC）同步信号源。
4. ADL_GLSYNC_SIGNALSOURCE_RJ45PORT1：表示RJ45（1）同步信号源。
5. ADL_GLSYNC_SIGNALSOURCE_RJ45PORT2：表示RJ45（2）同步信号源。

这里的GL-Sync信号源是用于同步数据传输的信号源，用于在数据传输过程中保证数据传输的同步性。


```cpp
/// GL-Sync signal source is undefined
#define ADL_GLSYNC_SIGNALSOURCE_UNDEFINED    0x00000100
/// GL-Sync signal source is Free Run
#define ADL_GLSYNC_SIGNALSOURCE_FREERUN      0x00000101
/// GL-Sync signal source is the BNC GL-Sync port
#define ADL_GLSYNC_SIGNALSOURCE_BNCPORT      0x00000102
/// GL-Sync signal source is the RJ45(1) GL-Sync port
#define ADL_GLSYNC_SIGNALSOURCE_RJ45PORT1    0x00000103
/// GL-Sync signal source is the RJ45(2) GL-Sync port
#define ADL_GLSYNC_SIGNALSOURCE_RJ45PORT2    0x00000104


// GL-Sync Signal Types (unique values)

/// GL-Sync signal type is unknown
```

这段代码定义了一系列GL-Sync信号类型枚举类型，包括ADL_GLSYNC_SIGNALTYPE_UNDEFINED，表示未定义的信号类型；ADL_GLSYNC_SIGNALTYPE_480I，表示GL-Sync信号类型为480I；ADL_GLSYNC_SIGNALTYPE_576I，表示GL-Sync信号类型为576I；ADL_GLSYNC_SIGNALTYPE_480P，表示GL-Sync信号类型为480P；ADL_GLSYNC_SIGNALTYPE_576P，表示GL-Sync信号类型为576P；ADL_GLSYNC_SIGNALTYPE_720P，表示GL-Sync信号类型为720P；ADL_GLSYNC_SIGNALTYPE_1080P，表示GL-Sync信号类型为1080P；ADL_GLSYNC_SIGNALTYPE_1080I，表示GL-Sync信号类型为1080I。这些信号类型用于在代码中描述GL-Sync信号类型，以便开发人员更好地理解代码。


```cpp
#define ADL_GLSYNC_SIGNALTYPE_UNDEFINED      0
/// GL-Sync signal type is 480I
#define ADL_GLSYNC_SIGNALTYPE_480I           1
/// GL-Sync signal type is 576I
#define ADL_GLSYNC_SIGNALTYPE_576I           2
/// GL-Sync signal type is 480P
#define ADL_GLSYNC_SIGNALTYPE_480P           3
/// GL-Sync signal type is 576P
#define ADL_GLSYNC_SIGNALTYPE_576P           4
/// GL-Sync signal type is 720P
#define ADL_GLSYNC_SIGNALTYPE_720P           5
/// GL-Sync signal type is 1080P
#define ADL_GLSYNC_SIGNALTYPE_1080P          6
/// GL-Sync signal type is 1080I
#define ADL_GLSYNC_SIGNALTYPE_1080I          7
```

这段代码定义了三种GL-Sync信号类型，分别是SDI、TTL和ANALOG，同时还定义了三种GL-Sync同步字段选项，分别为UNDEFINED、BOTH和1。

具体来说，ADL_GLSYNC_SIGNALTYPE_SDI表示SDI信号类型，ADL_GLSYNC_SIGNALTYPE_TTL表示TTL信号类型，ADL_GLSYNC_SIGNALTYPE_ANALOG表示ANALOG信号类型。而GL-Sync同步字段选项则可以用于控制同步信号的显示和记录。

在同步字段选项中，ADL_GLSYNC_SYNCFIELD_UNDEFINED表示同步字段选项为未定义，ADL_GLSYNC_SYNCFIELD_BOTH表示同步字段选项为BOTH，ADL_GLSYNC_SYNCFIELD_1表示同步字段选项为1。


```cpp
/// GL-Sync signal type is SDI
#define ADL_GLSYNC_SIGNALTYPE_SDI            8
/// GL-Sync signal type is TTL
#define ADL_GLSYNC_SIGNALTYPE_TTL            9
/// GL_Sync signal type is Analog
#define ADL_GLSYNC_SIGNALTYPE_ANALOG        10

// GL-Sync Sync Field options (unique values)

///GL-Sync sync field option is undefined
#define ADL_GLSYNC_SYNCFIELD_UNDEFINED        0
///GL-Sync sync field option is Sync to Field 1 (used for Interlaced signal types)
#define ADL_GLSYNC_SYNCFIELD_BOTH            1
///GL-Sync sync field option is Sync to Both fields (used for Interlaced signal types)
#define ADL_GLSYNC_SYNCFIELD_1                2


```

这段代码定义了GL-Sync触发边缘的选项，包括：

- 不存在（0）
- 上升沿（1）
- 下降沿（2）
- 上下都存在（3）

接着，定义了GL-Sync扫描率系数和多倍器的选项，包括：

- 不存在（0）

这里的选项是用来定义GL-Sync触发边缘的。通过设置这些选项的值，可以控制GL-Sync在渲染管线中的传输延迟。例如，设置ADL_GLSYNC_TRIGGEREDGE_UNDEFINED为1可以避免在某些情况下不必要的延迟。


```cpp
// GL-Sync trigger edge options (unique values)

/// GL-Sync trigger edge is undefined
#define ADL_GLSYNC_TRIGGEREDGE_UNDEFINED     0
/// GL-Sync trigger edge is the rising edge
#define ADL_GLSYNC_TRIGGEREDGE_RISING        1
/// GL-Sync trigger edge is the falling edge
#define ADL_GLSYNC_TRIGGEREDGE_FALLING       2
/// GL-Sync trigger edge is both the rising and the falling edge
#define ADL_GLSYNC_TRIGGEREDGE_BOTH          3


// GL-Sync scan rate coefficient/multiplier options (unique values)

/// GL-Sync scan rate coefficient/multiplier is undefined
```

这段代码定义了一系列常量，它们用于表示GL-Sync扫描率系数/ multiplier的值。其中，最后一个常量表示了一个分压多路复用(PVM)系数，将5个不同的GL-Sync模式信号的信号值进行加权求和，得到一个5:2的系数。

具体来说，这些常量的值如下：

- 第1行定义了一个名为ADL_GLSYNC_SCANRATECOEFF_UNDEFINED的常量，它的值为0。
- 第2行定义了四个名为ADL_GLSYNC_SCANRATECOEFF_x5、ADL_GLSYNC_SCANRATECOEFF_x4、ADL_GLSYNC_SCANRATECOEFF_x3和ADL_GLSYNC_SCANRATECOEFF_x5_DIV_2的常量，它们的值分别为1、2、3和4。
- 第3行定义了一个名为ADL_GLSYNC_SCANRATECOEFF_x2的常量，它的值为5。
- 第4行定义了一个名为ADL_GLSYNC_SCANRATECOEFF_x3_DIV_2的常量，它的值为6。
- 第5行定义了一个名为ADL_GLSYNC_SCANRATECOEFF_x5_DIV_4的常量，它的值为7。

这些常量被用于定义一个扫描模式(scan mode)，该扫描模式用于在NVIDIA的GPU上对GL-Sync信号进行加权求和，以获得更好的性能。


```cpp
#define ADL_GLSYNC_SCANRATECOEFF_UNDEFINED   0
/// GL-Sync scan rate coefficient/multiplier is 5
#define ADL_GLSYNC_SCANRATECOEFF_x5          1
/// GL-Sync scan rate coefficient/multiplier is 4
#define ADL_GLSYNC_SCANRATECOEFF_x4          2
/// GL-Sync scan rate coefficient/multiplier is 3
#define ADL_GLSYNC_SCANRATECOEFF_x3          3
/// GL-Sync scan rate coefficient/multiplier is 5:2 (SMPTE)
#define ADL_GLSYNC_SCANRATECOEFF_x5_DIV_2    4
/// GL-Sync scan rate coefficient/multiplier is 2
#define ADL_GLSYNC_SCANRATECOEFF_x2          5
/// GL-Sync scan rate coefficient/multiplier is 3 : 2
#define ADL_GLSYNC_SCANRATECOEFF_x3_DIV_2    6
/// GL-Sync scan rate coefficient/multiplier is 5 : 4
#define ADL_GLSYNC_SCANRATECOEFF_x5_DIV_4    7
```

这段代码定义了一系列GL-Sync扫描速率系数/乘以因子，用于控制扫描速率的 multiplier。其中第一个定义的系数为1，表示使用默认值。接下来的四个定义的系数从4开始，依次递增，分别表示在 SMPTE 模式下的扫描速率系数。最后一个定义的系数为15，表示在 GL-Sync 模式下的扫描速率系数。


```cpp
/// GL-Sync scan rate coefficient/multiplier is 1 (default)
#define ADL_GLSYNC_SCANRATECOEFF_x1          8
/// GL-Sync scan rate coefficient/multiplier is 4 : 5
#define ADL_GLSYNC_SCANRATECOEFF_x4_DIV_5    9
/// GL-Sync scan rate coefficient/multiplier is 2 : 3
#define ADL_GLSYNC_SCANRATECOEFF_x2_DIV_3    10
/// GL-Sync scan rate coefficient/multiplier is 1 : 2
#define ADL_GLSYNC_SCANRATECOEFF_x1_DIV_2    11
/// GL-Sync scan rate coefficient/multiplier is 2 : 5 (SMPTE)
#define ADL_GLSYNC_SCANRATECOEFF_x2_DIV_5    12
/// GL-Sync scan rate coefficient/multiplier is 1 : 3
#define ADL_GLSYNC_SCANRATECOEFF_x1_DIV_3    13
/// GL-Sync scan rate coefficient/multiplier is 1 : 4
#define ADL_GLSYNC_SCANRATECOEFF_x1_DIV_4    14
/// GL-Sync scan rate coefficient/multiplier is 1 : 5
```

这段代码定义了一个名为ADL_GLSYNC_SCANRATECOEFF_x1_DIV_5的宏，其值为15。这个宏被用来定义GL-Sync端口的状态。

GL-Sync是NVIDIA公司开发的一种技术，可以让开发者更轻松地将PCIe和GPU的带宽最大化，从而获得更好的性能。GL-Sync通过定义了一系列的宏，来描述GL-Sync端口的状态，包括连接、输入、输出等。

ADL_GLSYNC_PORTSTATE_UNDEFINED表示GL-Sync端口没有连接到任何设备，ADL_GLSYNC_PORTSTATE_NOCABLE表示GL-Sync端口无法提供有效的数据传输，ADL_GLSYNC_PORTSTATE_IDLE表示GL-Sync端口处于空闲状态，ADL_GLSYNC_PORTSTATE_INPUT表示GL-Sync端口存在输入信号，ADL_GLSYNC_PORTSTATE_OUTPUT表示GL-Sync端口存在输出信号。

ADL_GLSYNC_SCANRATECOEFF_x1_DIV_5是一个无符号整型常量，其值为15。这个常量被用来计算GL-Sync端口的扫描速率系数，用于计算GL-Sync端口的数据传输速度。


```cpp
#define ADL_GLSYNC_SCANRATECOEFF_x1_DIV_5    15


// GL-Sync port (signal presence) states (unique values)

/// GL-Sync port state is undefined
#define ADL_GLSYNC_PORTSTATE_UNDEFINED       0
/// GL-Sync port is not connected
#define ADL_GLSYNC_PORTSTATE_NOCABLE         1
/// GL-Sync port is Idle
#define ADL_GLSYNC_PORTSTATE_IDLE            2
/// GL-Sync port has an Input signal
#define ADL_GLSYNC_PORTSTATE_INPUT           3
/// GL-Sync port is Output
#define ADL_GLSYNC_PORTSTATE_OUTPUT          4


```

这段代码定义了一个同步LED的类型及其索引，用于在ADL_Workstation_GLSyncPortState_Get函数中返回一个同步LED的数组。这个数组包含了BNC口和RJ45口中的一个LED，分别用0和1来表示。此外，还定义了GL-Sync LED的颜色类型及其索引。GL-Sync LED指的是在GL-Sync技术下，需要同步更新的LED，其颜色可以是未定义的或者随机的。


```cpp
// GL-Sync LED types (used index within ADL_Workstation_GLSyncPortState_Get returned ppGlSyncLEDs array) (unique values)

/// Index into the ADL_Workstation_GLSyncPortState_Get returned ppGlSyncLEDs array for the one LED of the BNC port
#define ADL_GLSYNC_LEDTYPE_BNC               0
/// Index into the ADL_Workstation_GLSyncPortState_Get returned ppGlSyncLEDs array for the Left LED of the RJ45(1) or RJ45(2) port
#define ADL_GLSYNC_LEDTYPE_RJ45_LEFT         0
/// Index into the ADL_Workstation_GLSyncPortState_Get returned ppGlSyncLEDs array for the Right LED of the RJ45(1) or RJ45(2) port
#define ADL_GLSYNC_LEDTYPE_RJ45_RIGHT        1


// GL-Sync LED colors (unique values)

/// GL-Sync LED undefined color
#define ADL_GLSYNC_LEDCOLOR_UNDEFINED        0
/// GL-Sync LED is unlit
```

这段代码定义了一系列用于配置GL-Sync串口（仅支持1个GL-Sync端口）的预定义常量。通过这些常量，可以指定GL-Sync端口的IDLE状态、启用和禁用。

具体来说，以下代码主要实现了以下功能：

1.定义了4个预定义常量，分别代表黄色、红色、绿色和闪烁的绿色LED。

2.通过宏定义为每个预定义常量指定唯一的GL-Sync ID。例如，#define ADL_GLSYNC_LEDCOLOR_NOLIGHT 1代表无LED颜色，#define ADL_GLSYNC_LEDCOLOR_YELLOW 2代表黄色LED，以此类推。

3.通过宏定义为每个预定义常量指定唯一的IDL状态。例如，#define ADL_GLSYNC_PORTCNTL_NONE 0x00000000代表禁止操作，#define ADL_GLSYNC_PORTCNTL_ENABLE 0x00000001代表启用操作，#define ADL_GLSYNC_PORTCNTL_DISABLE 0x00000010代表禁用操作。

4.通过宏定义为每个预定义常量指定对应的LED颜色。例如，#define ADL_GLSYNC_LEDCOLOR_NOLIGHT 1代表无LED颜色，#define ADL_GLSYNC_LEDCOLOR_YELLOW 2代表黄色LED，#define ADL_GLSYNC_LEDCOLOR_RED 4代表红色LED，#define ADL_GLSYNC_LEDCOLOR_GREEN 8代表绿色LED，#define ADL_GLSYNC_LEDCOLOR_FLASH_GREEN 5代表闪烁的绿色LED。


```cpp
#define ADL_GLSYNC_LEDCOLOR_NOLIGHT          1
/// GL-Sync LED is yellow
#define ADL_GLSYNC_LEDCOLOR_YELLOW           2
/// GL-Sync LED is red
#define ADL_GLSYNC_LEDCOLOR_RED              3
/// GL-Sync LED is green
#define ADL_GLSYNC_LEDCOLOR_GREEN            4
/// GL-Sync LED is flashing green
#define ADL_GLSYNC_LEDCOLOR_FLASH_GREEN      5


// GL-Sync Port Control (refers one GL-Sync Port) (unique values)

/// Used to configure the RJ54(1) or RJ42(2) port of GL-Sync is as Idle
#define ADL_GLSYNC_PORTCNTL_NONE             0x00000000
```

这段代码定义了一系列与GL-Sync模式控制和显示相关的宏和常量。

首先定义了两个名为ADL_GLSYNC_PORTCNTL_OUTPUT和ADL_GLSYNC_MODECNTL_NONE的宏，它们都使用了0x00000001作为其默认值。第一个宏表示为GL-Sync显示器配置输出，而第二个宏表示为GL-Sync显示器设置为非同步模式。

接着定义了一系列与显示控制相关的位域常量。其中，ADL_GLSYNC_MODECNTL_GENLOCK和ADL_GLSYNC_MODECNTL_TIMINGSERVER定义了用于配置GL-Sync显示器为同步或非同步的位域，而ADL_GLSYNC_MODECNTL_NONE则表示不希望显示器进行同步。

最后，定义了一个名为ADL_GLSYNC_MDSSEG 的宏，用于设置 GL-Sync 显示器的显示模式。


```cpp
/// Used to configure the RJ54(1) or RJ42(2) port of GL-Sync is as Output
#define ADL_GLSYNC_PORTCNTL_OUTPUT           0x00000001


// GL-Sync Mode Control (refers one Display/Controller) (bitfields)

/// Used to configure the display to use internal timing (not genlocked)
#define ADL_GLSYNC_MODECNTL_NONE             0x00000000
/// Bitfield used to configure the display as genlocked (either as Timing Client or as Timing Server)
#define ADL_GLSYNC_MODECNTL_GENLOCK          0x00000001
/// Bitfield used to configure the display as Timing Server
#define ADL_GLSYNC_MODECNTL_TIMINGSERVER     0x00000002

// GL-Sync Mode Status
/// Display is currently not genlocked
```

这是一个定义头文件，其中包含了一些关于GLSYNC模块的常量，包括与显示器有关的模式控制。

1. `#define ADL_GLSYNC_MODECNTL_STATUS_NONE`：定义了一个名为`ADL_GLSYNC_MODECNTL_STATUS_NONE`的常量，值为0x00000000。这个常量表示GLSYNC模块当前没有任何异步操作，也就是处于初始化状态。
2. `#define ADL_GLSYNC_MODECNTL_STATUS_GENLOCK`：定义了一个名为`ADL_GLSYNC_MODECNTL_STATUS_GENLOCK`的常量，值为0x00000001。这个常量表示GLSYNC模块当前正在等待显示器并准备进行预先同步，但并不需要进行同步。
3. `#define ADL_GLSYNC_MODECNTL_STATUS_SETMODE_REQUIRED`：定义了一个名为`ADL_GLSYNC_MODECNTL_STATUS_SETMODE_REQUIRED`的常量，值为0x00000002。这个常量表示GLSYNC模块当前需要设置显示器以进行同步，但并不需要进行预先同步。
4. `#define ADL_GLSYNC_MODECNTL_STATUS_GENLOCK_ALLOWED`：定义了一个名为`ADL_GLSYNC_MODECNTL_STATUS_GENLOCK_ALLOWED`的常量，值为0x00000004。这个常量表示GLSYNC模块可以进行预先同步，并且已经准备好进行同步。
5. `#define ADL_MAX_GLSYNC_PORTS`：定义了一个名为`ADL_MAX_GLSYNC_PORTS`的常量，值为8。这个常量表示GLSYNC模块最多可以支持的最大显示器端口数。
6. `#define ADL_MAX_GLSYNC_PORT_LEDS`：定义了一个名为`ADL_MAX_GLSYNC_PORT_LEDS`的常量，值为8。这个常量表示每个最大端口连接的LED数量。
7. `/// \defgroup define_crossfirestate CrossfireX state of a particular adapter CrossfireX combination`：定义了一个名为`/// define_crossfirestate`的函数，它的作用是定义一个名为`CrossfireX`的命名，用于指定GLSYNC模块当前的异步操作状态，包括显存同步、预先同步和已同步。


```cpp
#define ADL_GLSYNC_MODECNTL_STATUS_NONE         0x00000000
/// Display is currently genlocked
#define ADL_GLSYNC_MODECNTL_STATUS_GENLOCK   0x00000001
/// Display requires a mode switch
#define ADL_GLSYNC_MODECNTL_STATUS_SETMODE_REQUIRED 0x00000002
/// Display is capable of being genlocked
#define ADL_GLSYNC_MODECNTL_STATUS_GENLOCK_ALLOWED 0x00000004

#define ADL_MAX_GLSYNC_PORTS                            8
#define ADL_MAX_GLSYNC_PORT_LEDS                        8

// @}

/// \defgroup define_crossfirestate CrossfireX state of a particular adapter CrossfireX combination
// @{
```

This is a list of possible configurations for the ADL (Advanced Linux调优器) CrossfireX，鸿蒙，即X核心针对Linux的跨平台优化。这些配置项涵盖了 CrossfireX 的各个方面，包括性能、功能和安全。其中，ADL_XFIREX_STATE_3DACTIVE表示三维客户端处于活动状态，即 CrossfireX 安全地启用。其他配置项分别表示三维客户端的显示、是否允许设置主要视图、是否允许显示内存、是否允许 pipes downgraded、是否通知内存当前处于 downgraded、是否通知 pipes 当前处于 downgraded、是否允许 CrossfireX 活动、是否允许设置等。


```cpp
#define ADL_XFIREX_STATE_NOINTERCONNECT            ( 1 << 0 )    /* Dongle / cable is missing */
#define ADL_XFIREX_STATE_DOWNGRADEPIPES            ( 1 << 1 )    /* CrossfireX can be enabled if pipes are downgraded */
#define ADL_XFIREX_STATE_DOWNGRADEMEM            ( 1 << 2 )    /* CrossfireX cannot be enabled unless mem downgraded */
#define ADL_XFIREX_STATE_REVERSERECOMMENDED        ( 1 << 3 )    /* Card reversal recommended, CrossfireX cannot be enabled. */
#define ADL_XFIREX_STATE_3DACTIVE            ( 1 << 4 )    /* 3D client is active - CrossfireX cannot be safely enabled */
#define ADL_XFIREX_STATE_MASTERONSLAVE            ( 1 << 5 )    /* Dongle is OK but master is on slave */
#define ADL_XFIREX_STATE_NODISPLAYCONNECT        ( 1 << 6 )    /* No (valid) display connected to master card. */
#define ADL_XFIREX_STATE_NOPRIMARYVIEW            ( 1 << 7 )    /* CrossfireX is enabled but master is not current primary device */
#define ADL_XFIREX_STATE_DOWNGRADEVISMEM        ( 1 << 8 )    /* CrossfireX cannot be enabled unless visible mem downgraded */
#define ADL_XFIREX_STATE_LESSTHAN8LANE_MASTER        ( 1 << 9 )     /* CrossfireX can be enabled however performance not optimal due to <8 lanes */
#define ADL_XFIREX_STATE_LESSTHAN8LANE_SLAVE        ( 1 << 10 )    /* CrossfireX can be enabled however performance not optimal due to <8 lanes */
#define ADL_XFIREX_STATE_PEERTOPEERFAILED        ( 1 << 11 )    /* CrossfireX cannot be enabled due to failed peer to peer test */
#define ADL_XFIREX_STATE_MEMISDOWNGRADED        ( 1 << 16 )    /* Notification that memory is currently downgraded */
#define ADL_XFIREX_STATE_PIPESDOWNGRADED        ( 1 << 17 )    /* Notification that pipes are currently downgraded */
#define ADL_XFIREX_STATE_XFIREXACTIVE            ( 1 << 18 )    /* CrossfireX is enabled on current device */
```

This is a definition of the XFIREX state field in the ADL (Advanced Drive Library) of AMD driver version 22.0. This field is used to indicate the status of various features related to CrossfireX, such as whether it is enabled and if any reboot or reconfiguration is required.

The state field is defined as a combination of bit fields, each of which corresponds to a specific feature. The bits in each field correspond to the possible values for that feature. For example, the first bit corresponds to the Adobe Document Cloud (ADL) bank, the second bit corresponds to the dual display feature, and so on.

The fields in the state field are defined as follows:

* ADL Bank: 1 bit，免疫力为1时，允许在全部可用ADL银行的内存 downgrade。此选项在0.11版本中没有明确的定义。
* CF Remote Desktop: 2 bits，远程桌面允许使用十字架fire 0.11版本已经定义了这种特性。
* CF P2P (Peer-to-Peer): 2 bits，允许P2P（对对）技术的触摸板。
* CF EAP (802.11b): 1 bit，支持802.11b。
* CF PowerSave (省电): 1 bit，当远程桌面连接处于活动状态时，进入省电模式。
* CF Unified Remote Composition: 1 bit，支持基于Unified Remote的远程桌面。
* CF CrossfireX Enabled: 1 bit，当催化剂驱动程序已确定并配置为支持十字架fire。
* CF XSP (eXtended Server Processor): 1 bit，当催化剂驱动程序已确定并配置为支持eXtended Server Processor。
* CF ECC Memory Support: 1 bit，支持在催化剂驱动程序上配置内存在催化剂的ECC内存。
* CF Reset AutoReboot: 1 bit，自动重新启动系统，以完成某些过程。
* CF DLLUnlocked: 1 bit，当隔离驱动程序加载时，系统是否锁定，即使电源已关闭。
* CF CCC woa 2.0: 1 bit, 2.0的 CCC（高保真度电源管理器）支持。
* CF Drv HANDLE xxxx: 4 bits，在特定的平台驱动程序上，为了解决待测项目提供的硬件ID。

每一项特征都有其自己的内存位，这些内存位可以用作计算其他状态的依据。例如，如果 A 选项的值为 1，那么这个项目已经处于可以升级的 ATT 驱动器中。如果 B 选项的值为 1，那么这个项目可以使用 ADL 并启用 CrossfireX。


```cpp
#define ADL_XFIREX_STATE_VISMEMISDOWNGRADED        ( 1 << 19 )    /* Notification that visible FB memory is currently downgraded */
#define ADL_XFIREX_STATE_INVALIDINTERCONNECTION        ( 1 << 20 )    /* Cannot support current inter-connection configuration */
#define ADL_XFIREX_STATE_NONP2PMODE            ( 1 << 21 )    /* CrossfireX will only work with clients supporting non P2P mode */
#define ADL_XFIREX_STATE_DOWNGRADEMEMBANKS        ( 1 << 22 )    /* CrossfireX cannot be enabled unless memory banks downgraded */
#define ADL_XFIREX_STATE_MEMBANKSDOWNGRADED        ( 1 << 23 )    /* Notification that memory banks are currently downgraded */
#define ADL_XFIREX_STATE_DUALDISPLAYSALLOWED        ( 1 << 24 )    /* Extended desktop or clone mode is allowed. */
#define ADL_XFIREX_STATE_P2P_APERTURE_MAPPING        ( 1 << 25 )    /* P2P mapping was through peer aperture */
#define ADL_XFIREX_STATE_P2PFLUSH_REQUIRED        ADL_XFIREX_STATE_P2P_APERTURE_MAPPING    /* For back compatible */
#define ADL_XFIREX_STATE_XSP_CONNECTED            ( 1 << 26 )    /* There is CrossfireX side port connection between GPUs */
#define ADL_XFIREX_STATE_ENABLE_CF_REBOOT_REQUIRED    ( 1 << 27 )    /* System needs a reboot bofore enable CrossfireX */
#define ADL_XFIREX_STATE_DISABLE_CF_REBOOT_REQUIRED    ( 1 << 28 )    /* System needs a reboot after disable CrossfireX */
#define ADL_XFIREX_STATE_DRV_HANDLE_DOWNGRADE_KEY    ( 1 << 29 )    /* Indicate base driver handles the downgrade key updating */
#define ADL_XFIREX_STATE_CF_RECONFIG_REQUIRED        ( 1 << 30 )    /* CrossfireX need to be reconfigured by CCC because of a LDA chain broken */
#define ADL_XFIREX_STATE_ERRORGETTINGSTATUS        ( 1 << 31 )    /* Could not obtain current status */
// @}

```

这段代码定义了一个名为 "ADL\_DISPLAY\_PIXELFORMAT" 的枚举类型，用于指定数字显示中像素格式。这个枚举类型包含了一些具体的枚举值，如 "ADL\_DISPLAY\_PIXELFORMAT\_UNKNOWN"、"ADL\_DISPLAY\_PIXELFORMAT\_RGB"、"ADL\_DISPLAY\_PIXELFORMAT\_YCRCB444"、"ADL\_DISPLAY\_PIXELFORMAT\_YCRCB422" 和 "ADL\_DISPLAY\_PIXELFORMAT\_RGB\_LIMITED\_RANGE"、"ADL\_DISPLAY\_PIXELFORMAT\_RGB\_FULL\_RANGE" 和 "ADL\_DISPLAY\_PIXELFORMAT\_YCRCB420"。"UNKNOWN" 的值为 0，表示未知。而其他值的值为 1，表示分别支持 RGB、YCRCB444、YCRCB422 和 RGB\_LIMITED\_RANGE 四种不同的范围。


```cpp
///////////////////////////////////////////////////////////////////////////
// ADL_DISPLAY_ADJUSTMENT_PIXELFORMAT adjustment values
// (bit-vector)
///////////////////////////////////////////////////////////////////////////
/// \defgroup define_pixel_formats Pixel Formats values
/// This group defines the various Pixel Formats that a particular digital display can support. \n
/// Since a display can support multiple formats, these values can be bit-or'ed to indicate the various formats \n
// @{
#define ADL_DISPLAY_PIXELFORMAT_UNKNOWN             0
#define ADL_DISPLAY_PIXELFORMAT_RGB                       (1 << 0)
#define ADL_DISPLAY_PIXELFORMAT_YCRCB444                  (1 << 1)    //Limited range
#define ADL_DISPLAY_PIXELFORMAT_YCRCB422                 (1 << 2)    //Limited range
#define ADL_DISPLAY_PIXELFORMAT_RGB_LIMITED_RANGE      (1 << 3)
#define ADL_DISPLAY_PIXELFORMAT_RGB_FULL_RANGE    ADL_DISPLAY_PIXELFORMAT_RGB  //Full range
#define ADL_DISPLAY_PIXELFORMAT_YCRCB420              (1 << 4)
```

这段代码定义了一个名为 `ADL_DL_DISPLAYCONFIG_CONTYPE_UNKNOWN` 的枚举类型，它表示连接器的类型为未知。接着，定义了 7 个枚举类型，每个枚举类型对应一个连接器类型，分别为：

* `ADL_DL_DISPLAYCONFIG_CONTYPE_CV_NONI2C_JP`：表示非晶质电容式触摸屏连接器（日本）
* `ADL_DL_DISPLAYCONFIG_CONTYPE_CV_JPN`：表示晶质电容式触摸屏连接器（日本）
* `ADL_DL_DISPLAYCONFIG_CONTYPE_CV_NA`：表示非晶质电容式触摸屏连接器（未知）
* `ADL_DL_DISPLAYCONFIG_CONTYPE_CV_NONI2C_NA`：表示晶质电容式触摸屏连接器（非晶质电容式触摸屏连接器）
* `ADL_DL_DISPLAYCONFIG_CONTYPE_VGA`：表示VGA连接器
* `ADL_DL_DISPLAYCONFIG_CONTYPE_DVI_D`：表示DVI-D连接器
* `ADL_DL_DISPLAYCONFIG_CONTYPE_DVI_I`：表示DVI-I连接器
* `ADL_DL_DISPLAYCONFIG_CONTYPE_HDMI_TYPE_A`：表示HDMI Type A连接器
* `ADL_DL_DISPLAYCONFIG_CONTYPE_HDMI_TYPE_B`：表示HDMI Type B连接器

最后，还有一条定义了 `ADL_DL_DISPLAYCONFIG_CONTYPE_UNKNOWN` 的定义：`0`。


```cpp
// @}

/// \defgroup define_contype Connector Type Values
/// ADLDisplayConfig.ulConnectorType defines
// @{
#define ADL_DL_DISPLAYCONFIG_CONTYPE_UNKNOWN      0
#define ADL_DL_DISPLAYCONFIG_CONTYPE_CV_NONI2C_JP 1
#define ADL_DL_DISPLAYCONFIG_CONTYPE_CV_JPN       2
#define ADL_DL_DISPLAYCONFIG_CONTYPE_CV_NA        3
#define ADL_DL_DISPLAYCONFIG_CONTYPE_CV_NONI2C_NA 4
#define ADL_DL_DISPLAYCONFIG_CONTYPE_VGA          5
#define ADL_DL_DISPLAYCONFIG_CONTYPE_DVI_D        6
#define ADL_DL_DISPLAYCONFIG_CONTYPE_DVI_I        7
#define ADL_DL_DISPLAYCONFIG_CONTYPE_HDMI_TYPE_A  8
#define ADL_DL_DISPLAYCONFIG_CONTYPE_HDMI_TYPE_B  9
```

这段代码定义了一个名为ADL_DL_DISPLAYCONFIG_CONTYPE_DISPLAYPORT的预处理指令。该指令定义了一个位掩，用于指定DisplayInfoMask的值。DisplayInfoMask是一个32位的二进制掩码，用于指示ADL Display连接到显示器时需要显示的信息。

具体来说，该指令定义了以下5个位掩：

- ADL_DISPLAY_DISPLAYINFO_DISPLAYCONNECTED：表示显示器是否与计算机相连。如果这个位为1，那么显示器将不再输出计算机的图形数据，而只输出计算机自身的图形数据。
- ADL_DISPLAY_DISPLAYINFO_DISPLAYMAPPED：表示是否将计算机的图形数据映射到显示器的表面。如果这个位为1，那么显示器将使用计算机的图形数据来绘制图形，而不是将显示器的表面映射为计算机的图形数据。
- ADL_DISPLAY_DISPLAYINFO_NONLOCAL：表示是否支持非本地显示。如果这个位为1，那么计算机可以使用非本地显示器来显示图形数据。
- ADL_DISPLAY_DISPLAYINFO_FORCIBLESUPPORTED：表示是否支持显示器驱动程序。如果这个位为1，那么计算机可以使用支持显示器驱动程序的显示器来显示图形数据。
- ADL_DISPLAY_DISPLAYINFO_GENLOCKSUPPORTED：表示是否支持生成锁。如果这个位为1，那么计算机可以使用支持生成锁的显示器来显示图形数据。


```cpp
#define ADL_DL_DISPLAYCONFIG_CONTYPE_DISPLAYPORT  10
// @}

///////////////////////////////////////////////////////////////////////////
// ADL_DISPLAY_DISPLAYINFO_ Definitions
// for ADLDisplayInfo.iDisplayInfoMask and ADLDisplayInfo.iDisplayInfoValue
// (bit-vector)
///////////////////////////////////////////////////////////////////////////
/// \defgroup define_displayinfomask Display Info Mask Values
// @{
#define ADL_DISPLAY_DISPLAYINFO_DISPLAYCONNECTED            0x00000001
#define ADL_DISPLAY_DISPLAYINFO_DISPLAYMAPPED                0x00000002
#define ADL_DISPLAY_DISPLAYINFO_NONLOCAL                    0x00000004
#define ADL_DISPLAY_DISPLAYINFO_FORCIBLESUPPORTED            0x00000008
#define ADL_DISPLAY_DISPLAYINFO_GENLOCKSUPPORTED            0x00000010
```

这段代码定义了一系列与显示器驱动程序相关的常量，用于定义显示器的行为。具体来说，这些常量定义了在哪些情况下可以使用特定功能，以及在不同情况下的行为。

ADL_DISPLAY_DISPLAYINFO_MULTIVPU_SUPPORTED:0x00000020，定义了多核显卡的支持。
ADL_DISPLAY_DISPLAYINFO_LDA_DISPLAY:0x00000040，定义了显示器驱动程序以查找显示器驱动程序的ID。
ADL_DISPLAY_DISPLAYINFO_MODETIMING_OVERRIDESUPPORTED:0x00000080，定义了支持修改显示器设置的ID。

ADL_DISPLAY_DISPLAYINFO_MANNER_SUPPORTED_SINGLE:0x00000100，定义了一个单一显示器设置，可以将其设置为物理显示器或虚拟显示器。
ADL_DISPLAY_DISPLAYINFO_MANNER_SUPPORTED_CLONE:0x00000200，定义了一个克隆的单一显示器设置，可以将其设置为物理显示器或虚拟显示器。

ADL_DISPLAY_DISPLAYINFO_MANNER_SUPPORTED_2VSTRETCH:0x00000400，定义了支持2个视频线香的设置。
ADL_DISPLAY_DISPLAYINFO_MANNER_SUPPORTED_2HSTRETCH:0x00000800，定义了支持2个高电压电平的设置。
ADL_DISPLAY_DISPLAYINFO_MANNER_SUPPORTED_EXTENDED:0x00001000，定义了一个延长设置，可以将其设置为物理显示器或虚拟显示器。

ADL_DISPLAY_DISPLAYINFO_MANNER_SUPPORTED_NSTRETCH1GPU:0x00010000，定义了一个NSTRETCH1GPU设置，用于设置能够使用NVIDIA G-Sync技术来支持的GPU。
ADL_DISPLAY_DISPLAYINFO_MANNER_SUPPORTED_NSTRETCHNGPU:0x00020000，定义了一个NSTRETCHNGPU设置，用于设置能够使用NVIDIA G-Sync技术来支持的GPU。


```cpp
#define ADL_DISPLAY_DISPLAYINFO_MULTIVPU_SUPPORTED            0x00000020
#define ADL_DISPLAY_DISPLAYINFO_LDA_DISPLAY                    0x00000040
#define ADL_DISPLAY_DISPLAYINFO_MODETIMING_OVERRIDESSUPPORTED            0x00000080

#define ADL_DISPLAY_DISPLAYINFO_MANNER_SUPPORTED_SINGLE            0x00000100
#define ADL_DISPLAY_DISPLAYINFO_MANNER_SUPPORTED_CLONE            0x00000200

/// Legacy support for XP
#define ADL_DISPLAY_DISPLAYINFO_MANNER_SUPPORTED_2VSTRETCH        0x00000400
#define ADL_DISPLAY_DISPLAYINFO_MANNER_SUPPORTED_2HSTRETCH        0x00000800
#define ADL_DISPLAY_DISPLAYINFO_MANNER_SUPPORTED_EXTENDED        0x00001000

/// More support manners
#define ADL_DISPLAY_DISPLAYINFO_MANNER_SUPPORTED_NSTRETCH1GPU    0x00010000
#define ADL_DISPLAY_DISPLAYINFO_MANNER_SUPPORTED_NSTRETCHNGPU    0x00020000
```

这段代码定义了一些关于ADL（自定义显示库）的显示器设置标志。

首先，定义了两个预定义的IDL（自定义显示库）宏：
```cpp
#define ADL_DISPLAY_DISPLAYINFO_MANNER_SUPPORTED_RESERVED2        0x00040000
#define ADL_DISPLAY_DISPLAYINFO_MANNER_SUPPORTED_RESERVED3        0x00080000
```
这两个宏表示显示器支持的方式，第一个表示支持的消息显示方式，第二个表示支持的对象标识符。

接着，定义了一个名为ADL_DISPLAY_DISPLAYINFO_SHOWTYPE_PROJECTOR的常量：
```cpp
#define ADL_DISPLAY_DISPLAYINFO_SHOWTYPE_PROJECTOR                0x00100000
```
这个常量表示为投影仪显示类型设置的ID。

最后，定义了一个名为ADL_ADAPTER_DISPLAY_MANNER_SUPPORTED的私用接口：
```cpp
// @}

// Adapter Display Manner Supported Values

public interface ADL_ADAPTER_DISPLAY_MANNER_SUPPORTED {
   bit-vector ShowType;
   bit-vector DisplayInfo;
   // ...
}
```
这个接口声明了一个位图，表示支持的消息显示类型，以及支持的消息显示的相关信息。


```cpp
#define ADL_DISPLAY_DISPLAYINFO_MANNER_SUPPORTED_RESERVED2        0x00040000
#define ADL_DISPLAY_DISPLAYINFO_MANNER_SUPPORTED_RESERVED3        0x00080000

/// Projector display type
#define ADL_DISPLAY_DISPLAYINFO_SHOWTYPE_PROJECTOR                0x00100000

// @}


///////////////////////////////////////////////////////////////////////////
// ADL_ADAPTER_DISPLAY_MANNER_SUPPORTED_ Definitions
// for ADLAdapterDisplayCap of ADL_Adapter_Display_Cap()
// (bit-vector)
///////////////////////////////////////////////////////////////////////////
/// \defgroup define_adaptermanner Adapter Manner Support Values
```

这段代码定义了一系列与适配器显示模式相关的 macros，用于设置或获取某些适配器的显示模式支持。这些 macros 的值由 0 或 2 开头，表示不同的显示模式支持。其中，最后一个 macro 定义了 “Legacy support for XP”，表示兼容 Windows XP 的显示模式支持。这些 macros 可以被用来设置或获取 ADL 适配器的显示模式，包括单击、复制、扩展等。


```cpp
// @{
#define ADL_ADAPTER_DISPLAYCAP_MANNER_SUPPORTED_NOTACTIVE        0x00000001
#define ADL_ADAPTER_DISPLAYCAP_MANNER_SUPPORTED_SINGLE            0x00000002
#define ADL_ADAPTER_DISPLAYCAP_MANNER_SUPPORTED_CLONE            0x00000004
#define ADL_ADAPTER_DISPLAYCAP_MANNER_SUPPORTED_NSTRETCH1GPU    0x00000008
#define ADL_ADAPTER_DISPLAYCAP_MANNER_SUPPORTED_NSTRETCHNGPU    0x00000010

/// Legacy support for XP
#define ADL_ADAPTER_DISPLAYCAP_MANNER_SUPPORTED_2VSTRETCH        0x00000020
#define ADL_ADAPTER_DISPLAYCAP_MANNER_SUPPORTED_2HSTRETCH        0x00000040
#define ADL_ADAPTER_DISPLAYCAP_MANNER_SUPPORTED_EXTENDED        0x00000080

#define ADL_ADAPTER_DISPLAYCAP_PREFERDISPLAY_SUPPORTED            0x00000100
#define ADL_ADAPTER_DISPLAYCAP_BEZEL_SUPPORTED                    0x00000200


```

这是一段定义了ADL Display Map Manner类型的代码。这个类型定义了四种不同的显示模式，分别为Active、Singleton、Clone和Reserved。每种显示模式都有对应的位向量表示，可以在操作系统提供的字体中使用。这个定义被用于在图形用户界面中显示ADL Display Map的不同显示模式。


```cpp
///////////////////////////////////////////////////////////////////////////
// ADL_DISPLAY_DISPLAYMAP_MANNER_ Definitions
// for ADLDisplayMap.iDisplayMapMask and ADLDisplayMap.iDisplayMapValue
// (bit-vector)
///////////////////////////////////////////////////////////////////////////
#define ADL_DISPLAY_DISPLAYMAP_MANNER_RESERVED            0x00000001
#define ADL_DISPLAY_DISPLAYMAP_MANNER_NOTACTIVE            0x00000002
#define ADL_DISPLAY_DISPLAYMAP_MANNER_SINGLE            0x00000004
#define ADL_DISPLAY_DISPLAYMAP_MANNER_CLONE                0x00000008
#define ADL_DISPLAY_DISPLAYMAP_MANNER_RESERVED1            0x00000010  // Removed NSTRETCH
#define ADL_DISPLAY_DISPLAYMAP_MANNER_HSTRETCH            0x00000020
#define ADL_DISPLAY_DISPLAYMAP_MANNER_VSTRETCH            0x00000040
#define ADL_DISPLAY_DISPLAYMAP_MANNER_VLD                0x00000080

// @}

```

这段代码定义了两个头文件：ADL_DISPLAY_DISPLAYMAP_OPTION_和ADL_DISPLAY_DISPLAYTARGET_。它们用于定义显示map选项和显示目标优先级。

ADL_DISPLAY_DISPLAYMAP_OPTION_定义了一个位图，用于表示屏幕上每个像素的显示map选项。这个位图的第一个元素是一个标志，表示屏幕上是否可以使用显示map选项。如果这个标志为1，那么就表示可以使用显示map选项，否则就不能使用。

ADL_DISPLAY_DISPLAYTARGET_定义了一个位图，用于表示每个显示目标的优先级。这个位图的第一个元素是一个标志，表示这个显示目标是否是优先的。如果是优先的，那么就表示这个显示目标可以优先显示，否则就不能优先显示。

这两个定义是用于函数ADL_Display_DisplayMapConfig_Get的，用于在屏幕上显示一个显示map，并在优先级较高的显示目标上优先显示显示map。


```cpp
///////////////////////////////////////////////////////////////////////////
// ADL_DISPLAY_DISPLAYMAP_OPTION_ Definitions
// for iOption in function ADL_Display_DisplayMapConfig_Get
// (bit-vector)
///////////////////////////////////////////////////////////////////////////
#define ADL_DISPLAY_DISPLAYMAP_OPTION_GPUINFO            0x00000001

///////////////////////////////////////////////////////////////////////////
// ADL_DISPLAY_DISPLAYTARGET_ Definitions
// for ADLDisplayTarget.iDisplayTargetMask and ADLDisplayTarget.iDisplayTargetValue
// (bit-vector)
///////////////////////////////////////////////////////////////////////////
#define ADL_DISPLAY_DISPLAYTARGET_PREFERRED            0x00000001

///////////////////////////////////////////////////////////////////////////
```

这段代码定义了一个名为ADL_DISPLAY_POSSIBLEMAPRESULT_VALID的定义，它包含三个枚举类型ADLPossibleMapResult.iPossibleMapResultMask、ADLPossibleMapResult.iPossibleMapResultValue和ADLPossibleMapResult.iPossibleMapResultBits，它们分别表示以二进制形式表示的地图结果掩码、地图结果值和地图结果位。

接下来，定义了一个名为ADL_DISPLAY_MODE_的定义，包含三个枚举类型ADLMode.iModeMask、ADLMode.iModeValue和ADLMode.iModeFlag，它们分别表示显示模式掩码、显示模式值和显示模式标志。

最后，通过宏定义将ADL_DISPLAY_POSSIBLEMAPRESULT_VALID、ADL_DISPLAY_POSSIBLEMAPRESULT_BEZELSUPPORTED和ADL_DISPLAY_POSSIBLEMAPRESULT_OVERLAPSUPPORTED定义为整数常量0x00000001、0x00000002和0x00000004。


```cpp
// ADL_DISPLAY_POSSIBLEMAPRESULT_VALID Definitions
// for ADLPossibleMapResult.iPossibleMapResultMask and ADLPossibleMapResult.iPossibleMapResultValue
// (bit-vector)
///////////////////////////////////////////////////////////////////////////
#define ADL_DISPLAY_POSSIBLEMAPRESULT_VALID                0x00000001
#define ADL_DISPLAY_POSSIBLEMAPRESULT_BEZELSUPPORTED    0x00000002
#define ADL_DISPLAY_POSSIBLEMAPRESULT_OVERLAPSUPPORTED    0x00000004

///////////////////////////////////////////////////////////////////////////
// ADL_DISPLAY_MODE_ Definitions
// for ADLMode.iModeMask, ADLMode.iModeValue, and ADLMode.iModeFlag
// (bit-vector)
///////////////////////////////////////////////////////////////////////////
/// \defgroup define_displaymode Display Mode Values
// @{
```



这段代码定义了一系列常量，用于指定ATL()库中关于显示模式的定义。下面是每个常量的解释：

- ADL_DISPLAY_MODE_COLOURFORMAT_565：表示以565色度模式在屏幕上显示4:3:2的图像。
- ADL_DISPLAY_MODE_COLOURFORMAT_8888：表示以8888色度模式在屏幕上显示4:3:2的图像。
- ADL_DISPLAY_MODE_ORIENTATION_SUPPORTED_000：表示支持0度旋转。
- ADL_DISPLAY_MODE_ORIENTATION_SUPPORTED_090：表示支持90度旋转。
- ADL_DISPLAY_MODE_ORIENTATION_SUPPORTED_180：表示支持180度旋转。
- ADL_DISPLAY_MODE_ORIENTATION_SUPPORTED_270：表示支持270度旋转。
- ADL_DISPLAY_MODE_REFRESHMENT_ROUNDED：表示在显示器上以圆形的方式刷新显示，类似于ATL6401中的 mode=0x20000001。
- ADL_DISPLAY_MODE_REFRESHMENT_ONLY：表示仅在显示器上以垂直方向刷新显示，类似于ATL6401中的 mode=0x00000080。
- ADL_DISPLAY_MODE_PROGRESSIVE_FLAG：表示启用渐变和动画效果。
- ADL_DISPLAY_MODE_INTERLACED_FLAG：表示使屏幕上的颜色更加平滑过渡。


```cpp
#define ADL_DISPLAY_MODE_COLOURFORMAT_565                0x00000001
#define ADL_DISPLAY_MODE_COLOURFORMAT_8888                0x00000002
#define ADL_DISPLAY_MODE_ORIENTATION_SUPPORTED_000        0x00000004
#define ADL_DISPLAY_MODE_ORIENTATION_SUPPORTED_090        0x00000008
#define ADL_DISPLAY_MODE_ORIENTATION_SUPPORTED_180        0x00000010
#define ADL_DISPLAY_MODE_ORIENTATION_SUPPORTED_270        0x00000020
#define ADL_DISPLAY_MODE_REFRESHRATE_ROUNDED            0x00000040
#define ADL_DISPLAY_MODE_REFRESHRATE_ONLY                0x00000080

#define ADL_DISPLAY_MODE_PROGRESSIVE_FLAG    0
#define ADL_DISPLAY_MODE_INTERLACED_FLAG    2
// @}

///////////////////////////////////////////////////////////////////////////
// ADL_OSMODEINFO Definitions
```

这段代码定义了一系列的`ADL_OSMODEINFOXPOS_DEFAULT`到`ADL_OSMODEINFOYRES_DEFAULT`的常量，用于表示操作系统模式的信息。这些常量用于在应用程序中设置或获取操作系统模式的信息，例如分辨率、颜色深度等。

具体来说，这些常量的值表示了操作系统的默认设置，当应用程序需要更改这些设置时，可以通过调用这些常量中的一个或多个来获取或设置相应的值。


```cpp
///////////////////////////////////////////////////////////////////////////
/// \defgroup define_osmode OS Mode Values
// @{
#define ADL_OSMODEINFOXPOS_DEFAULT                -640
#define ADL_OSMODEINFOYPOS_DEFAULT                0
#define ADL_OSMODEINFOXRES_DEFAULT                640
#define ADL_OSMODEINFOYRES_DEFAULT                480
#define ADL_OSMODEINFOXRES_DEFAULT800            800
#define ADL_OSMODEINFOYRES_DEFAULT600            600
#define ADL_OSMODEINFOREFRESHRATE_DEFAULT        60
#define ADL_OSMODEINFOCOLOURDEPTH_DEFAULT        8
#define ADL_OSMODEINFOCOLOURDEPTH_DEFAULT16        16
#define ADL_OSMODEINFOCOLOURDEPTH_DEFAULT24        24
#define ADL_OSMODEINFOCOLOURDEPTH_DEFAULT32        32
#define ADL_OSMODEINFOORIENTATION_DEFAULT        0
```

这段代码定义了一个名为ADL_OSMODEINFOORIENTATION_DEFAULT_WIN7的定义，其作用是设置显示配置中旋转变换的强制方向为顺时针方向，即从左至右。接着定义了一个名为ADL_OSMODEFLAG_DEFAULT的定义，其作用是设置ADLOSMODEINFOORIENTATION_DEFAULT_WIN7为0，即不强制执行顺时针方向的显示配置旋转变换。最后，通过ADLThreadingModel enumeration定义了ADL threading behavior，包括ADL_THREADING_UNLOCKED和ADL_THREADING_LOCKED两个枚举类型，分别表示ADL不会和ADL库一起执行顺时针方向的显示配置旋转变换，以及允许ADL库在多线程情况下进入ADL API。


```cpp
#define ADL_OSMODEINFOORIENTATION_DEFAULT_WIN7    DISPLAYCONFIG_ROTATION_FORCE_UINT32
#define ADL_OSMODEFLAG_DEFAULT                    0
// @}


///////////////////////////////////////////////////////////////////////////
// ADLThreadingModel Enumeration
///////////////////////////////////////////////////////////////////////////
/// \defgroup thread_model
/// Used with \ref ADL_Main_ControlX2_Create and \ref ADL2_Main_ControlX2_Create to specify how ADL handles API calls when executed by multiple threads concurrently.
/// \brief Declares ADL threading behavior.
// @{
typedef enum ADLThreadingModel
{
    ADL_THREADING_UNLOCKED    = 0, /*!< Default behavior. ADL will not enforce serialization of ADL API executions by multiple threads.  Multiple threads will be allowed to enter to ADL at the same time. Note that ADL library is not guaranteed to be thread-safe. Client that calls ADL_Main_Control_Create have to provide its own mechanism for ADL calls serialization. */
    ADL_THREADING_LOCKED     /*!< ADL will enforce serialization of ADL API when called by multiple threads.  Only single thread will be allowed to enter ADL API at the time. This option makes ADL calls thread-safe. You shouldn't use this option if ADL calls will be executed on Linux on x-server rendering thread. It can cause the application to hung.  */
}ADLThreadingModel;

```

这段代码定义了一个名为 ADLPurposeCode 的枚举类型，用于表示各种对设备的访问目的码。

这个枚举类型包括以下几个枚举常量：

- ADL_PURPOSECODE_NORMAL    = 0，表示访问目的为普通设备
- ADL_PURPOSECODE_HIDE_MODE_SWITCH，表示访问目的为隐藏设备，且模式选择/取消模式切换
- ADL_PURPOSECODE_MODE_SWITCH，表示访问目的为普通设备
- ADL_PURPOSECODE_ATTATCH_DEVICE，表示设备 attach
- ADL_PURPOSECODE_DETACH_DEVICE，表示设备 detach
- ADL_PURPOSECODE_SETPRIMARY_DEVICE，表示设置主要设备
- ADL_PURPOSECODE_GDI_ROTATION，表示设备在平面内旋转
- ADL_PURPOSECODE_BITFIX_ROTATION，表示设备在平面内进行有限旋转，使用 bitFIX 计数器

每种访问目的码都有一个枚举类型成员，以便根据需要选择适当的访问模式。


```cpp
// @}
///////////////////////////////////////////////////////////////////////////
// ADLPurposeCode Enumeration
///////////////////////////////////////////////////////////////////////////
enum ADLPurposeCode
{
    ADL_PURPOSECODE_NORMAL    = 0,
    ADL_PURPOSECODE_HIDE_MODE_SWITCH,
    ADL_PURPOSECODE_MODE_SWITCH,
    ADL_PURPOSECODE_ATTATCH_DEVICE,
    ADL_PURPOSECODE_DETACH_DEVICE,
    ADL_PURPOSECODE_SETPRIMARY_DEVICE,
    ADL_PURPOSECODE_GDI_ROTATION,
    ADL_PURPOSECODE_ATI_ROTATION
};
```

这段代码定义了一个名为 ADLAngle 的枚举类型，包括四个枚举值：ADL_ANGLE_LANDSCAPE、ADL_ANGLE_ROTATERIGHT、ADL_ANGLE_ROTATE180 和 ADL_ANGLE_ROTATELEFT。每个枚举值都有一个对应的数字值，分别为 0、90、180 和 270。

另外，还定义了一个名为 ADLOrientationDataType 的枚举类型，包括两个枚举值：ADL_ORIENTATIONTYPE_OSDATATYPE 和 ADL_ORIENTATIONTYPE_NONOSDATATYPE。每个枚举值都有一个对应的数字值，分别为 0 和 1。


```cpp
///////////////////////////////////////////////////////////////////////////
// ADLAngle Enumeration
///////////////////////////////////////////////////////////////////////////
enum ADLAngle
{
    ADL_ANGLE_LANDSCAPE = 0,
    ADL_ANGLE_ROTATERIGHT = 90,
    ADL_ANGLE_ROTATE180 = 180,
    ADL_ANGLE_ROTATELEFT = 270,
};

///////////////////////////////////////////////////////////////////////////
// ADLOrientationDataType Enumeration
///////////////////////////////////////////////////////////////////////////
enum ADLOrientationDataType
{
    ADL_ORIENTATIONTYPE_OSDATATYPE,
    ADL_ORIENTATIONTYPE_NONOSDATATYPE
};

```

这段代码定义了两个枚举类型ADLPanningMode和ADLLARGEDESKTOPTYPE，用于表示不同的音频设备参数。

ADLPanningMode枚举类型表示音频设备是否进行规划，它有三个枚举值：0表示不规划，1表示至少一个设备进行规划，2表示允许规划。

ADLLARGEDESKTOPTYPE枚举类型表示桌面的大小，它有三个枚举值：0表示普通桌面，1表示缩水桌面，2表示大型桌面。

这两个枚举类型都是从同一个较低级别枚举类型开始的，也就是说，如果你想从最小级别开始使用某个枚举类型，只需要写一个ADLPanningMode或ADLLARGEDESKTOPTYPEconst类型的变量，比如说：
```cppc
enum ADLPanningMode {
   ADL_PANNINGMODE_NO_PANNING,
   ADL_PANNINGMODE_AT_LEAST_ONE_NO_PANNING,
   ADL_PANNINGMODE_ALLOW_PANNING
};

enum ADLLARGEDESKTOPTYPE {
   ADL_LARGEDESKTOPTYPE_NORMALDESKTOP,
   ADL_LARGEDESKTOPTYPE_PSEUDOLARGEDESKTOP,
   ADL_LARGEDESKTOPTYPE_VERYLARGEDESKTOP
};
```
然后，你可以使用这些枚举类型变量来设置音频设备和桌面大小的值。


```cpp
///////////////////////////////////////////////////////////////////////////
// ADLPanningMode Enumeration
///////////////////////////////////////////////////////////////////////////
enum ADLPanningMode
{
    ADL_PANNINGMODE_NO_PANNING = 0,
    ADL_PANNINGMODE_AT_LEAST_ONE_NO_PANNING = 1,
    ADL_PANNINGMODE_ALLOW_PANNING = 2,
};

///////////////////////////////////////////////////////////////////////////
// ADLLARGEDESKTOPTYPE Enumeration
///////////////////////////////////////////////////////////////////////////
enum ADLLARGEDESKTOPTYPE
{
    ADL_LARGEDESKTOPTYPE_NORMALDESKTOP = 0,
    ADL_LARGEDESKTOPTYPE_PSEUDOLARGEDESKTOP = 1,
    ADL_LARGEDESKTOPTYPE_VERYLARGEDESKTOP = 2
};

```

这段代码定义了一个名为ADLPlatForm的枚举类型，它有两个值：GRAPHICS_PLATFORM_DESKTOP和GRAPHICS_PLATFORM_MOBILE。这两个枚举类型都代表graphics platform(图形平台)，可以用于控制应用程序的图形设置，比如分辨率、帧率等。

另外，还定义了一个名为ADLGraphicCoreGeneration的枚举类型，它也有两个值：ADL_GRAPHIC_CORE_GENERATION_UNDEFINED和ADL_GRAPHIC_CORE_GENERATION_PRE_GCN。ADL_GRAPHIC_CORE_GENERATION_UNDEFINED表示graphic core generation(图形核心生成)的未知值，ADL_GRAPHIC_CORE_GENERATION_PRE_GCN表示graphic core generation(图形核心生成)的预GCN(预先GCN),ADL_GRAPHIC_CORE_GENERATION_GCN表示graphic core generation(图形核心生成)的GCN(通用图形核心生成),ADL_GRAPHIC_CORE_GENERATION_RDNA表示graphic core generation(图形核心生成)的RDNA(随机DNA)。

最后，还定义了一个名为ADLPlatFormGraphicCore generation的枚举类型，它与ADLPlatForm枚举类型相关联，用于控制graphics platform(图形平台)中graphic core generation(图形核心生成)的值。


```cpp
///////////////////////////////////////////////////////////////////////////
// ADLPlatform Enumeration
///////////////////////////////////////////////////////////////////////////
enum ADLPlatForm
{
    GRAPHICS_PLATFORM_DESKTOP  = 0,
    GRAPHICS_PLATFORM_MOBILE   = 1
};

///////////////////////////////////////////////////////////////////////////
// ADLGraphicCoreGeneration Enumeration
///////////////////////////////////////////////////////////////////////////
enum ADLGraphicCoreGeneration
{
    ADL_GRAPHIC_CORE_GENERATION_UNDEFINED                   = 0,
    ADL_GRAPHIC_CORE_GENERATION_PRE_GCN                     = 1,
    ADL_GRAPHIC_CORE_GENERATION_GCN                         = 2,
    ADL_GRAPHIC_CORE_GENERATION_RDNA                        = 3
};

```

这段代码定义了ADL_Display_WriteAndReadI2CRev_Get函数的参数和返回值。

ADL_Display_WriteAndReadI2CRev_Get函数是用于在I2C总线上进行读写操作的函数。它使用了三个预定义的常量，分别为ADL_I2C_MAJOR_API_REV、ADL_I2C_MINOR_DEFAULT_API_REV和ADL_I2C_MINOR_OEM_API_REV。这些常量定义了I2C座位的API版本号。

然后，定义了四个用于表示I2C命令的常量，分别为ADL_DL_I2C_LINE_OEM、ADL_DL_I2C_LINE_OD_CONTROL、ADL_DL_I2C_LINE_OEM2和ADL_DL_I2C_LINE_OEM3。这些常量定义了I2C线的命令，包括写入、读取和执行控制命令。

最后，还定义了一个名为ADL_DL_I2C_LINE_OEM4和ADL_DL_I2C_LINE_OEM5的常量，它们分别表示I2C线的OEM4和OEM5命令。

这些定义在函数中使用，以便在实现I2C通信时根据需要进行正确的配置。


```cpp
// Other Definitions for internal use

// Values for ADL_Display_WriteAndReadI2CRev_Get()

#define ADL_I2C_MAJOR_API_REV           0x00000001
#define ADL_I2C_MINOR_DEFAULT_API_REV   0x00000000
#define ADL_I2C_MINOR_OEM_API_REV       0x00000001

// Values for ADL_Display_WriteAndReadI2C()
#define ADL_DL_I2C_LINE_OEM                0x00000001
#define ADL_DL_I2C_LINE_OD_CONTROL         0x00000002
#define ADL_DL_I2C_LINE_OEM2               0x00000003
#define ADL_DL_I2C_LINE_OEM3               0x00000004
#define ADL_DL_I2C_LINE_OEM4               0x00000005
#define ADL_DL_I2C_LINE_OEM5               0x00000006
```

这段代码定义了一系列常量，用于定义I2C通信中的数据缓冲区大小、最大数据传输速度、最大地址长度和最大偏移长度等参数。

具体来说，定义了I2C数据缓冲区最大大小为0x00000040，也就是4字节。接着定义了最大数据传输速度为0x0000000C，也就是0字节。然后定义了最大地址长度为0x00000006，也就是6个字节。最后定义了最大偏移长度为0x00000004，也就是4个字节。

此外，还定义了一系列常量来控制ADL显示属性。例如，定义了ADL_DL_DISPLAYPROPERTY_TYPE_UNKNOWN为0时，表示不支持任何显示属性；定义了ADL_DL_DISPLAYPROPERTY_TYPE_EXPANSIONMODE为1时，表示支持扩展扫描范围；定义了ADL_DL_DISPLAYPROPERTY_TYPE_USEUNDERSCANSCALING为2时，表示使用 underscale 扫描范围。最后，定义了ADL_DL_DISPLAYPROPERTY_TYPE_ITCFLAGENABLE为9时，表示支持ITC处理。


```cpp
#define ADL_DL_I2C_LINE_OEM6               0x00000007

// Max size of I2C data buffer
#define ADL_DL_I2C_MAXDATASIZE             0x00000040
#define ADL_DL_I2C_MAXWRITEDATASIZE        0x0000000C
#define ADL_DL_I2C_MAXADDRESSLENGTH        0x00000006
#define ADL_DL_I2C_MAXOFFSETLENGTH         0x00000004


/// Values for ADLDisplayProperty.iPropertyType
#define ADL_DL_DISPLAYPROPERTY_TYPE_UNKNOWN              0
#define ADL_DL_DISPLAYPROPERTY_TYPE_EXPANSIONMODE        1
#define ADL_DL_DISPLAYPROPERTY_TYPE_USEUNDERSCANSCALING     2
/// Enables ITC processing for HDMI panels that are capable of the feature
#define ADL_DL_DISPLAYPROPERTY_TYPE_ITCFLAGENABLE        9
```

这段代码定义了两个头文件，一个是`ADL_DL_DISPLAYPROPERTY_TYPE_DOWNSCALE`，另一个是`ADL_DL_DISPLAYPROPERTY_TYPE_INTEGER_SCALING`。这两个头文件定义了两个整型变量，分别对应于`ADL_DL_DISPLAYPROPERTY_TYPE_DOWNSCALE`和`ADL_DL_DISPLAYPROPERTY_TYPE_INTEGER_SCALING`。它们的值都被定义为`11`和`12`。

接下来是两个内联函数，分别定义了四个整型变量，它们的值也被定义为`11`、`12`、`2`和`8`。这些变量分别对应于`ADL_DL_DISPLAYCONTENT_TYPE_GRAPHICS`、`ADL_DL_DISPLAYCONTENT_TYPE_PHOTO`、`ADL_DL_DISPLAYCONTENT_TYPE_CINEMA`和`ADL_DL_DISPLAYCONTENT_TYPE_GAME`。

然后是另一个内联函数，定义了一个整型变量，它的值被定义为`11`。这个整型变量没有定义对应的定义，但它的值被用作前面定义的整型变量的别名，即`ADL_DL_DISPLAYPROPERTY_TYPE_DOWNSCALE`。

最后，定义了一个名为`ADL_DL_DISPLAYPROPERTY_TYPE_INTEGER_SCALING`的宏，它的值为`12`。


```cpp
#define ADL_DL_DISPLAYPROPERTY_TYPE_DOWNSCALE            11
#define ADL_DL_DISPLAYPROPERTY_TYPE_INTEGER_SCALING      12


/// Values for ADLDisplayContent.iContentType
/// Certain HDMI panels that support ITC have support for a feature such that, the display on the panel
/// can be adjusted to optimize the view of the content being displayed, depending on the type of content.
#define ADL_DL_DISPLAYCONTENT_TYPE_GRAPHICS        1
#define ADL_DL_DISPLAYCONTENT_TYPE_PHOTO        2
#define ADL_DL_DISPLAYCONTENT_TYPE_CINEMA        4
#define ADL_DL_DISPLAYCONTENT_TYPE_GAME            8



//values for ADLDisplayProperty.iExpansionMode
```

这段代码定义了一系列与显示属性相关的外部扩展模式，包括：

1. ADL_DL_DISPLAYPROPERTY_EXPANSIONMODE_CENTER：中心扩展模式，即图像在屏幕上居中显示。
2. ADL_DL_DISPLAYPROPERTY_EXPANSIONMODE_FULLSCREEN：全屏幕扩展模式，即图像在屏幕上占据整个屏幕。
3. ADL_DL_DISPLAYPROPERTY_EXPANSIONMODE_ASPECTRATIO：应用程序比例扩展模式，即根据设备的可用空间在水平和垂直方向上调整图像大小。

这些外部扩展模式可以用于在支持外部扩展的显示属性中设置自定义的显示模式。


```cpp
#define ADL_DL_DISPLAYPROPERTY_EXPANSIONMODE_CENTER        0
#define ADL_DL_DISPLAYPROPERTY_EXPANSIONMODE_FULLSCREEN    1
#define ADL_DL_DISPLAYPROPERTY_EXPANSIONMODE_ASPECTRATIO   2


///\defgroup define_dither_states Dithering options
// @{
/// Dithering disabled.
#define ADL_DL_DISPLAY_DITHER_DISABLED              0
/// Use default driver settings for dithering. Note that the default setting could be dithering disabled.
#define ADL_DL_DISPLAY_DITHER_DRIVER_DEFAULT        1
/// Temporal dithering to 6 bpc. Note that if the input is 12 bits, the two least significant bits will be truncated.
#define ADL_DL_DISPLAY_DITHER_FM6                   2
/// Temporal dithering to 8 bpc.
#define ADL_DL_DISPLAY_DITHER_FM8                   3
```

这段代码定义了一系列关于Temporal dithering（时间差分）的常量，用于在不同位深（bpc，每个像素的比特数）下对图像进行差分。这里需要注意的是，如果输入图像的位数是12位，则两个最不相关的比特将被截断。

具体来说，代码定义了以下七种空间差分模式：

1. Temporal dithering to 10 bpc.（10位深度差分）
2. Spatial dithering to 6 bpc.（6位空间差分，随机数生成器在每一帧重置，因此对于相同像素的输入，输出将始终相同）
3. Spatial dithering to 8 bpc.（8位空间差分，随机数生成器在每一帧重置，因此对于相同像素的输入，输出将始终相同）
4. Spatial dithering to 10 bpc.（10位空间差分，随机数生成器在每一帧重置，因此对于相同像素的输入，输出将始终相同）
5. Spatial dithering to 6 bpc.（6位空间差分，随机数生成器在每一帧重置，因此对于相同像素的输入，输出将始终相同，且两个最不相关的比特将被截断）
6. Spatial dithering to 8 bpc.（8位空间差分，随机数生成器在每一帧重置，因此对于相同像素的输入，输出将始终相同，且两个最不相关的比特将被截断）
7. Spatial dithering to 10 bpc.（10位空间差分，随机数生成器在每一帧重置，因此对于相同像素的输入，输出将始终相同，且两个最不相关的比特将被截断）

最后还定义了一个常量，表示如何截断输入图像的最低位。


```cpp
/// Temporal dithering to 10 bpc.
#define ADL_DL_DISPLAY_DITHER_FM10                  4
/// Spatial dithering to 6 bpc. Note that if the input is 12 bits, the two least significant bits will be truncated.
#define ADL_DL_DISPLAY_DITHER_DITH6                 5
/// Spatial dithering to 8 bpc.
#define ADL_DL_DISPLAY_DITHER_DITH8                 6
/// Spatial dithering to 10 bpc.
#define ADL_DL_DISPLAY_DITHER_DITH10                7
/// Spatial dithering to 6 bpc. Random number generators are reset every frame, so the same input value of a certain pixel will always be dithered to the same output value. Note that if the input is 12 bits, the two least significant bits will be truncated.
#define ADL_DL_DISPLAY_DITHER_DITH6_NO_FRAME_RAND   8
/// Spatial dithering to 8 bpc. Random number generators are reset every frame, so the same input value of a certain pixel will always be dithered to the same output value.
#define ADL_DL_DISPLAY_DITHER_DITH8_NO_FRAME_RAND   9
/// Spatial dithering to 10 bpc. Random number generators are reset every frame, so the same input value of a certain pixel will always be dithered to the same output value.
#define ADL_DL_DISPLAY_DITHER_DITH10_NO_FRAME_RAND  10
/// Truncation to 6 bpc.
```

这是一段C/C++代码，定义了一系列常量，用于定义数据保留（data retention）策略，以提高显示驱动程序（ADL）的性能。这些常量定义了如何对数据进行截断，以便在需要时减少数据传输量，从而提高显示驱动程序的效率。

具体来说，这些常量定义了以下数据保留策略：

1. 截断至8位二进制数据。
2. 截断至10位二进制数据。
3. 截断至10位二进制数据，然后进行空间下采样（spatial dithering）。
4. 截断至10位二进制数据，然后进行空间下采样，并采用8位二进制数据。
5. 截断至10位二进制数据，然后进行空间下采样，并采用6位二进制数据。
6. 截断至10位二进制数据，然后进行时间下采样（temporal dithering）。
7. 截断至10位二进制数据，然后进行时间下采样，并采用8位二进制数据。
8. 截断至10位二进制数据，然后进行时间下采样，并采用6位二进制数据。

这些策略可以通过编译器进行展开，生成相应的数据保留函数，以在显示驱动程序中执行。


```cpp
#define ADL_DL_DISPLAY_DITHER_TRUN6                 11
/// Truncation to 8 bpc.
#define ADL_DL_DISPLAY_DITHER_TRUN8                 12
/// Truncation to 10 bpc.
#define ADL_DL_DISPLAY_DITHER_TRUN10                13
/// Truncation to 10 bpc followed by spatial dithering to 8 bpc.
#define ADL_DL_DISPLAY_DITHER_TRUN10_DITH8          14
/// Truncation to 10 bpc followed by spatial dithering to 6 bpc.
#define ADL_DL_DISPLAY_DITHER_TRUN10_DITH6          15
/// Truncation to 10 bpc followed by temporal dithering to 8 bpc.
#define ADL_DL_DISPLAY_DITHER_TRUN10_FM8            16
/// Truncation to 10 bpc followed by temporal dithering to 6 bpc.
#define ADL_DL_DISPLAY_DITHER_TRUN10_FM6            17
/// Truncation to 10 bpc followed by spatial dithering to 8 bpc and temporal dithering to 6 bpc.
#define ADL_DL_DISPLAY_DITHER_TRUN10_DITH8_FM6      18
```

这段代码定义了一系列空间和时间上的平滑（dithering）操作，用于对浮点数数据进行平滑处理。平滑操作包括：水平平滑（spatial dithering）、垂直平滑（temporal dithering）和8位精度平滑（truncation）。

具体来说，这段代码定义了以下操作：

1. 水平平滑：将10位二进制数据（8个字节）进行平滑处理，以达到8位精度。
2. 垂直平滑：将10位二进制数据（8个字节）进行平滑处理，以达到8位精度。
3. 8位精度平滑：将8位二进制数据（6个字节）进行平滑处理，以达到8位精度。

然后，通过宏定义将这些操作与操作的优先级关联起来，以便根据需要组合使用：

```cpp
// Spatial dithering to 10 bpc followed by temporal dithering to 8 bpc.
#define ADL_DL_DISPLAY_DITHER_DITH10_FM8            19
// Spatial dithering to 10 bpc followed by temporal dithering to 6 bpc.
#define ADL_DL_DISPLAY_DITHER_DITH10_FM6            20
// Truncation to 8 bpc followed by spatial dithering to 6 bpc.
#define ADL_DL_DISPLAY_DITHER_TRUN8_DITH6           21
// Truncation to 8 bpc followed by temporal dithering to 6 bpc.
#define ADL_DL_DISPLAY_DITHER_TRUN8_FM6             22
// Spatial dithering to 8 bpc followed by temporal dithering to 6 bpc.
#define ADL_DL_DISPLAY_DITHER_DITH8_FM6             23
// ADL_DL_DISPLAY_DITHER_LAST
```

因此，这段代码定义了一系列操作，可以用来在浮点数数据中进行水平、垂直和8位精度平滑。用户可以根据需要组合这些操作，以获得所需的数据平滑效果。


```cpp
/// Spatial dithering to 10 bpc followed by temporal dithering to 8 bpc.
#define ADL_DL_DISPLAY_DITHER_DITH10_FM8            19
/// Spatial dithering to 10 bpc followed by temporal dithering to 6 bpc.
#define ADL_DL_DISPLAY_DITHER_DITH10_FM6            20
/// Truncation to 8 bpc followed by spatial dithering to 6 bpc.
#define ADL_DL_DISPLAY_DITHER_TRUN8_DITH6           21
/// Truncation to 8 bpc followed by temporal dithering to 6 bpc.
#define ADL_DL_DISPLAY_DITHER_TRUN8_FM6             22
/// Spatial dithering to 8 bpc followed by temporal dithering to 6 bpc.
#define ADL_DL_DISPLAY_DITHER_DITH8_FM6             23
#define ADL_DL_DISPLAY_DITHER_LAST                  ADL_DL_DISPLAY_DITHER_DITH8_FM6
// @}


/// Display Get Cached EDID flag
```

这段代码定义了一些常量，包括：

1. ADL_MAX_EDIDDATA_SIZE，表示增强型ID（增强型数据）的最大数据大小，单位为字节（UCHAR）。
2. ADL_MAX_OVERSEIDDATA_SIZE，表示覆盖型ID（覆盖型数据）的最大数据大小，单位为字节（UCHAR）。
3. ADL_MAX_EDID_EXTENSION_BLOCKS，表示自定义ID（自定义数据）的最大数据大小，单位为字节（UCHAR）。
4. ADL_DL_CONTROLLER_OVERLAY_ALPHA，表示控制器覆盖层α的亮度（alpha），取值范围为0-255。
5. ADL_DL_CONTROLLER_OVERLAY_ALPHAPERPIX，表示控制器覆盖层α的每像素颜色值，范围为0-4294967295。

此外，还定义了几个宏，包括：

1. ADL_DL_DISPLAY_DATA_PACKET__INFO_PACKET_RESET，表示设置IDL显示数据packet的初始化（reset）状态，其值为0。
2. ADL_DL_DISPLAY_DATA_PACKET__INFO_PACKET_SET，表示设置IDL显示数据packet的设置状态，其值为1。
3. ADL_DL_DISPLAY_DATA_PACKET__INFO_PACKET_SCAN，表示扫描IDL显示数据packet的参数范围，其值为0-3。

另外，还定义了一个名为“ADL_DL_DISPLAY_DATA_PACKET__TYPE__AVI”的类型定义，表示IDL显示数据packet的类型为AVI（Application Viewable Image）。定义了一个名为“ADL_DL_DISPLAY_DATA_PACKET__TYPE__GAMMUT”的类型定义，表示IDL显示数据packet的类型为GAMMUT（GAMMUT颜）。


```cpp
#define ADL_MAX_EDIDDATA_SIZE              256 // number of UCHAR
#define ADL_MAX_OVERRIDEEDID_SIZE          512 // number of UCHAR
#define ADL_MAX_EDID_EXTENSION_BLOCKS      3

#define ADL_DL_CONTROLLER_OVERLAY_ALPHA         0
#define ADL_DL_CONTROLLER_OVERLAY_ALPHAPERPIX   1

#define ADL_DL_DISPLAY_DATA_PACKET__INFO_PACKET_RESET      0x00000000
#define ADL_DL_DISPLAY_DATA_PACKET__INFO_PACKET_SET        0x00000001
#define ADL_DL_DISPLAY_DATA_PACKET__INFO_PACKET_SCAN       0x00000002

///\defgroup define_display_packet Display Data Packet Types
// @{
#define ADL_DL_DISPLAY_DATA_PACKET__TYPE__AVI              0x00000001
#define ADL_DL_DISPLAY_DATA_PACKET__TYPE__GAMMUT           0x00000002
```

这段代码定义了一个名为 ADL_DL_DISPLAY_DATA_PACKET__TYPE__ 的头文件，用于定义显示数据 packets 的类型。

具体来说，这个头文件定义了两个整型变量 ADL_DL_DISPLAY_DATA_PACKET__TYPE__VENDORINFO 和 ADL_DL_DISPLAY_DATA_PACKET__TYPE__HDR，分别表示数据包的供应商信息和头部信息。

另外，头文件中还定义了一个名为 ADL_DL_DISPLAY_DATA_PACKET__TYPE__SPD 的整型变量，表示数据包的扫描平面数据传输（SPD）类型。

另外，通过宏定义，定义了几个与显示数据相关的常量，如 ADL_GAMUT_MATRIX_SD 和 ADL_GAMUT_MATRIX_HD，分别表示 SD 和 HD 矩阵的类型。

最后，通过定义了一些与显示数据相关的标志，如 ADL_DL_CLOCKINFO_FLAG_FULLSCREEN3DONLY 和 ADL_DL_CLOCKINFO_FLAG_VPURECOVERYREDUCED，用于控制是否使用完整的屏幕显示和垂直独立校正。


```cpp
#define ADL_DL_DISPLAY_DATA_PACKET__TYPE__VENDORINFO       0x00000004
#define ADL_DL_DISPLAY_DATA_PACKET__TYPE__HDR              0x00000008
#define ADL_DL_DISPLAY_DATA_PACKET__TYPE__SPD              0x00000010
// @}

// matrix types
#define ADL_GAMUT_MATRIX_SD         1   // SD matrix i.e. BT601
#define ADL_GAMUT_MATRIX_HD         2   // HD matrix i.e. BT709

///\defgroup define_clockinfo_flags Clock flags
/// Used by ADLAdapterODClockInfo.iFlag
// @{
#define ADL_DL_CLOCKINFO_FLAG_FULLSCREEN3DONLY         0x00000001
#define ADL_DL_CLOCKINFO_FLAG_ALWAYSFULLSCREEN3D       0x00000002
#define ADL_DL_CLOCKINFO_FLAG_VPURECOVERYREDUCED       0x00000004
```

这段代码定义了一个名为 ADL_DL_CLOCKINFO_FLAG_THERMALPROTECTION 的头文件，并包含了一个常量 0x00000008。这个头文件定义了一些与 GPU 相关的常量和枚举类型，如 GPU 类型、GPU 集成/分离、以及可能的操作结果等。这些常量和枚举类型用于在代码中标识和描述 GPU 的状态和操作结果。

接下来的几行代码定义了支持的 GPU 类型和相应的枚举类型。其中，ADL_DL_POWERXPRESS_GPU_INTEGRATED 和 ADL_DL_POWERXPRESS_GPU_DISCRETE 分别定义了 GPU 类型为 ADL_DL_POWERXPRESS_GPU_INTEGRATED 和 ADL_DL_POWERXPRESS_GPU_DISCRETE。

接下来的几行代码定义了可能的操作结果类型，包括 ADL_DL_POWERXPRESS_SWITCH_RESULT_STARTED、ADL_DL_POWERXPRESS_SWITCH_RESULT_DECLINED、ADL_DL_POWERXPRESS_SWITCH_RESULT_ALREADY 和 ADL_DL_POWERXPRESS_SWITCH_RESULT_DEFERRED 等。这些枚举类型用于描述 GPU 操作结果的状态，例如是否可以启动、关闭、已准备好或被 deferred 等。

最后，定义了一个名为 ADL_DL_CLOCKINFO_FLAG_THERMALPROTECTION 的头文件，其中包含了一个名为 0x00000008 的常量，这个常量定义了一个名为 ADL_DL_THEMETRIC_LABEL 的枚举类型，这个枚举类型定义了如何显示温度保护的标签。


```cpp
#define ADL_DL_CLOCKINFO_FLAG_THERMALPROTECTION        0x00000008
// @}

// Supported GPUs
// ADL_Display_PowerXpressActiveGPU_Get()
#define ADL_DL_POWERXPRESS_GPU_INTEGRATED        1
#define ADL_DL_POWERXPRESS_GPU_DISCRETE            2

// Possible values for lpOperationResult
// ADL_Display_PowerXpressActiveGPU_Get()
#define ADL_DL_POWERXPRESS_SWITCH_RESULT_STARTED         1 // Switch procedure has been started - Windows platform only
#define ADL_DL_POWERXPRESS_SWITCH_RESULT_DECLINED        2 // Switch procedure cannot be started - All platforms
#define ADL_DL_POWERXPRESS_SWITCH_RESULT_ALREADY         3 // System already has required status  - All platforms
#define ADL_DL_POWERXPRESS_SWITCH_RESULT_DEFERRED        5  // Switch was deferred and requires an X restart - Linux platform only

```

这段代码定义了一个名为"PowerXpress support version"的常量，其值为2.0。接下来，定义了一个名为"ADL_DL_POWERXPRESS_VERSION_MAJOR"的宏，其含义是当前PowerXpress支持版本的 majory 值(即2)，再定义了一个名为"ADL_DL_POWERXPRESS_VERSION_MINOR"的宏，其含义是当前PowerXpress支持版本的 minory 值(即0)。然后，定义了一个名为"ADL_DL_POWERXPRESS_VERSION"的宏，其含义是 majory 和 minory 值的中间值，使用了位运算。

接下来，定义了两个宏，分别名为"ADLThermalControllerInfo.iThermalControllerDomain"和"ADLThermalControllerInfo.iFlags"，用于指定PowerXpress的thermal controller的domain和flags。然后，给这两个宏分别赋予了不同的值，域的值为0，表示不支持；域的值为1，表示支持，同时 fenctrl的值为2，表示允许。


```cpp
// PowerXpress support version
// ADL_Display_PowerXpressVersion_Get()
#define ADL_DL_POWERXPRESS_VERSION_MAJOR            2    // Current PowerXpress support version 2.0
#define ADL_DL_POWERXPRESS_VERSION_MINOR            0

#define ADL_DL_POWERXPRESS_VERSION    (((ADL_DL_POWERXPRESS_VERSION_MAJOR) << 16) | ADL_DL_POWERXPRESS_VERSION_MINOR)

//values for ADLThermalControllerInfo.iThermalControllerDomain
#define ADL_DL_THERMAL_DOMAIN_OTHER      0
#define ADL_DL_THERMAL_DOMAIN_GPU        1

//values for ADLThermalControllerInfo.iFlags
#define ADL_DL_THERMAL_FLAG_INTERRUPT    1
#define ADL_DL_THERMAL_FLAG_FANCONTROL   2

```

这段代码定义了一个名为 `ADLFanSpeedInfo` 的结构体，其中包括了控制风扇速度的多个值，如风扇速度、百分比速度读取与写入、RPM 读取与写入等。

接着，定义了一个名为 `ADLFanSpeedValue` 的结构体，其中包括了用于表示风扇速度类型的值，如百分比速度读取、RPM 读取等。

接着，通过 `#define` 定义了一些常量，用于指定 `ADL_DL_FANCTRL_SUPPORTS_PERCENT_READ`、`ADL_DL_FANCTRL_SUPPORTS_PERCENT_WRITE`、`ADL_DL_FANCTRL_SUPPORTS_RPM_READ` 和 `ADL_DL_FANCTRL_SUPPORTS_RPM_WRITE`。

然后，通过 `#define` 定义了一些常量，用于指定 `ADL_DL_FANCTRL_SPEED_TYPE_PERCENT` 和 `ADL_DL_FANCTRL_SPEED_TYPE_RPM`。

接下来，通过 `#define` 定义了一个名为 `ADL_DL_FANCTRL_FLAG_USER_DEFINED_SPEED` 的常量，用于指定用户自定义风扇速度。


```cpp
///\defgroup define_fanctrl Fan speed cotrol
/// Values for ADLFanSpeedInfo.iFlags
// @{
#define ADL_DL_FANCTRL_SUPPORTS_PERCENT_READ     1
#define ADL_DL_FANCTRL_SUPPORTS_PERCENT_WRITE    2
#define ADL_DL_FANCTRL_SUPPORTS_RPM_READ         4
#define ADL_DL_FANCTRL_SUPPORTS_RPM_WRITE        8
// @}

//values for ADLFanSpeedValue.iSpeedType
#define ADL_DL_FANCTRL_SPEED_TYPE_PERCENT    1
#define ADL_DL_FANCTRL_SPEED_TYPE_RPM        2

//values for ADLFanSpeedValue.iFlags
#define ADL_DL_FANCTRL_FLAG_USER_DEFINED_SPEED   1

```

这段代码定义了一系列用于MVPU（多单元平行处理单元）的接口定义，包括定义了MVPU适配器的编号，以及定义了与ASIC（应用服务器芯片）相关的ADLMVPU状态机状态。

具体来说，这段代码定义了以下内容：

1. 定义了ADL_DL_MAX_MVPU_ADAPTERS常量，表示可以支持的最大MVPU适配器数量。
2. 定义了MVPU_ADAPTER_0、MVPU_ADAPTER_1和MVPU_ADAPTER_2常量，分别表示ASIC型号为0、1和2的设备。
3. 定义了MVPU_ADAPTER_3常量，表示ASIC型号为3的设备。
4. 定义了ADL_DL_MAX_REGISTRY_PATH常量，表示程序可以访问的最大的注册表路径大小。
5. 定义了ADLMVPUStatus结构体，其中iStatus成员包括ASIC型号0、1和2的MVPU状态机状态，分别为Off、On和Off。
6. 定义了ASIC家族相关的定义，包括定义了不同ASIC型号的适配器类型。


```cpp
// MVPU interfaces
#define ADL_DL_MAX_MVPU_ADAPTERS   4
#define MVPU_ADAPTER_0          0x00000001
#define MVPU_ADAPTER_1          0x00000002
#define MVPU_ADAPTER_2          0x00000004
#define MVPU_ADAPTER_3          0x00000008
#define ADL_DL_MAX_REGISTRY_PATH   256

//values for ADLMVPUStatus.iStatus
#define ADL_DL_MVPU_STATUS_OFF   0
#define ADL_DL_MVPU_STATUS_ON    1

// values for ASIC family
///\defgroup define_Asic_type Detailed asic types
/// Defines for Adapter ASIC family type
```

这段代码定义了一系列的ASIC（加速软件接口）计时器标记，用于配置操作系统中的计时器，为这些标记定义了对应的ASIC类型。这些标记通过插入到`adl_asic_timing_flags`结构体中，进而表示了ASIC计时器的计时 flags，具体含义如下：

1. `ADL_ASIC_UNDEFINED`：表示这是一个尚未定义的ASIC计时器。
2. `ADL_ASIC_DISCRETE`：表示这是一个标记为DISCRETE的ASIC计时器，不会再进行深入配置。
3. `ADL_ASIC_INTEGRATED`：表示这是一个已集成到系统中的ASIC计时器，不会再被设置为独立模式。
4. `ADL_ASIC_FIREGL`：表示这是一个标记为FIREGL的ASIC计时器，启用GL驱动程序。
5. `ADL_ASIC_FIREMV`：表示这是一个标记为FIREMV的ASIC计时器，启用CPU虚拟机监控。
6. `ADL_ASIC_XGP`：表示这是一个标记为XGP的ASIC计时器，启用硬件虚拟机监控。
7. `ADL_ASIC_FUSION`：表示这是一个标记为FUSION的ASIC计时器，启用融合技术。
8. `ADL_ASIC_FIRESTREAM`：表示这是一个标记为FIRESTREAM的ASIC计时器，启用硬件浮动寄存器。
9. `ADL_ASIC_EMBEDDED`：表示这是一个标记为EMBEDDED的ASIC计时器，它禁止在虚拟机中运行。

这些标记的组合情况可以用来控制系统中计时器的运行模式，包括定时器、缓冲区等。


```cpp
// @{
#define ADL_ASIC_UNDEFINED    0
#define ADL_ASIC_DISCRETE    (1 << 0)
#define ADL_ASIC_INTEGRATED    (1 << 1)
#define ADL_ASIC_FIREGL        (1 << 2)
#define ADL_ASIC_FIREMV        (1 << 3)
#define ADL_ASIC_XGP        (1 << 4)
#define ADL_ASIC_FUSION        (1 << 5)
#define ADL_ASIC_FIRESTREAM (1 << 6)
#define ADL_ASIC_EMBEDDED   (1 << 7)
// @}

///\defgroup define_detailed_timing_flags Detailed Timimg Flags
/// Defines for ADLDetailedTiming.sTimingFlags field
// @{
```

这段代码定义了一系列的 timing flag，用于控制显示模式下的 timing 标准。这些 flag 用于设置垂直和水平同步的 polarity（正负），以及双重扫描（仅在 Interlanced 模式下使用）。

具体来说，当设置为 Interlanced 模式时，将设置 `ADL_DL_TIMINGFLAG_H_SYNC_POLARITY` 和 `ADL_DL_TIMINGFLAG_V_SYNC_POLARITY` 为 1，设置 `ADL_DL_TIMINGFLAG_DOUBLE_SCAN` 为 0；当设置为 Progressive 模式时，将设置 `ADL_DL_TIMINGFLAG_H_SYNC_POLARITY` 和 `ADL_DL_TIMINGFLAG_V_SYNC_POLARITY` 为 0，设置 `ADL_DL_TIMINGFLAG_DOUBLE_SCAN` 为 1。

此外，`ADL_DL_TIMINGFLAG_INTERLACED` 和 `ADL_DL_TIMINGFLAG_H_SYNC_POLARITY` 和 `ADL_DL_TIMINGFLAG_V_SYNC_POLARITY` 的作用与 `ADL_DL_TIMINGFLAG_H_SYNC_POLARITY` 和 `ADL_DL_TIMINGFLAG_V_SYNC_POLARITY` 类似，但只在 Interlanced 模式下有效。


```cpp
#define ADL_DL_TIMINGFLAG_DOUBLE_SCAN              0x0001
//sTimingFlags is set when the mode is INTERLACED, if not PROGRESSIVE
#define ADL_DL_TIMINGFLAG_INTERLACED               0x0002
//sTimingFlags is set when the Horizontal Sync is POSITIVE, if not NEGATIVE
#define ADL_DL_TIMINGFLAG_H_SYNC_POLARITY          0x0004
//sTimingFlags is set when the Vertical Sync is POSITIVE, if not NEGATIVE
#define ADL_DL_TIMINGFLAG_V_SYNC_POLARITY          0x0008
// @}

///\defgroup define_modetiming_standard Timing Standards
/// Defines for ADLDisplayModeInfo.iTimingStandard field
// @{
#define ADL_DL_MODETIMING_STANDARD_CVT             0x00000001 // CVT Standard
#define ADL_DL_MODETIMING_STANDARD_GTF             0x00000002 // GFT Standard
#define ADL_DL_MODETIMING_STANDARD_DMT             0x00000004 // DMT Standard
```

这段代码定义了一系列与ADL驱动器相关的头文件和常量，具体解释如下：

1. `#define ADL_DL_MODETIMING_STANDARD_CUSTOM`：定义了一个名为`ADL_DL_MODETIMING_STANDARD_CUSTOM`的定义，其值为0x00000008。这意味着这是一个自定义的ADLDL模型标准，其常量名称为`ADL_DL_MODETIMING_STANDARD_CUSTOM`，类型为0x00000008。

2. `#define ADL_DL_MODETIMING_STANDARD_DRIVER_DEFAULT`：定义了一个名为`ADL_DL_MODETIMING_STANDARD_DRIVER_DEFAULT`的定义，其值为0x00000010。这意味着这是一个ADLDL驱动器的默认设置，其常量名称为`ADL_DL_MODETIMING_STANDARD_DRIVER_DEFAULT`，类型为0x00000010。

3. `#define ADL_DL_MODETIMING_STANDARD_CVT_RB`：定义了一个名为`ADL_DL_MODETIMING_STANDARD_CVT_RB`的定义，其值为0x00000020。这意味着这是一个CVT-RB标准，其常量名称为`ADL_DL_MODETIMING_STANDARD_CVT_RB`，类型为0x00000020。

4. `// @}`：这是一个自定义的结束标记，用于标记ADLDL头文件中定义的常量。


```cpp
#define ADL_DL_MODETIMING_STANDARD_CUSTOM          0x00000008 // User-defined standard
#define ADL_DL_MODETIMING_STANDARD_DRIVER_DEFAULT  0x00000010 // Remove Mode from overriden list
#define ADL_DL_MODETIMING_STANDARD_CVT_RB           0x00000020 // CVT-RB Standard
// @}

// \defgroup define_xserverinfo driver x-server info
/// These flags are used by ADL_XServerInfo_Get()
// @

/// Xinerama is active in the x-server, Xinerama extension may report it to be active but it
/// may not be active in x-server
#define ADL_XSERVERINFO_XINERAMAACTIVE            (1<<0)

/// RandR 1.2 is supported by driver, RandR extension may report version 1.2
/// but driver may not support it
```

这段代码定义了一个名为ADL_XSERVERINFO_RANDR12SUPPORTED的宏，其含义是（1<<1）。接着，定义了一个名为ADL_CONTROLLERINDEX_0和ADL_CONTROLLERINDEX_1的宏，分别表示控制器和LL显示器的控制器索引。然后，定义了ADL_DISPLAY_SLSGRID_ORIENTATION_000、ADL_DISPLAY_SLSGRID_ORIENTATION_090、ADL_DISPLAY_SLSGRID_ORIENTATION_180和ADL_DISPLAY_SLSGRID_ORIENTATION_270四个宏，用于表示显示器在四个不同方向上的 orientation，分别为000、090、180和270。最后，定义了一个名为ADL_DISPLAY_SLSGRID_CAP_OPTION_RELATIVETO_LANDSCAPE的宏，用于表示显示器在陆地面积选项中的优先级，分别为0001、0002、0004和0008。


```cpp
#define ADL_XSERVERINFO_RANDR12SUPPORTED          (1<<1)
// @


///\defgroup define_eyefinity_constants Eyefinity Definitions
// @{

#define ADL_CONTROLLERVECTOR_0        1    // ADL_CONTROLLERINDEX_0 = 0, (1 << ADL_CONTROLLERINDEX_0)
#define ADL_CONTROLLERVECTOR_1        2    // ADL_CONTROLLERINDEX_1 = 1, (1 << ADL_CONTROLLERINDEX_1)

#define ADL_DISPLAY_SLSGRID_ORIENTATION_000        0x00000001
#define ADL_DISPLAY_SLSGRID_ORIENTATION_090        0x00000002
#define ADL_DISPLAY_SLSGRID_ORIENTATION_180        0x00000004
#define ADL_DISPLAY_SLSGRID_ORIENTATION_270        0x00000008
#define ADL_DISPLAY_SLSGRID_CAP_OPTION_RELATIVETO_LANDSCAPE     0x00000001
```

这段代码定义了一系列定义，涉及到SLS(Secure Local Server)格式的显示器设置选项。具体来说，代码定义了以下选项：

- ADL_DISPLAY_SLSGRID_CAP_OPTION_RELATIVETO_CURRENTANGLE:0x00000002，表示相对当前角度的显示器范围选项。
- ADL_DISPLAY_SLSGRID_PORTAIT_MODE:0x00000004，表示是否以可变模式查看显示屏。
- ADL_DISPLAY_SLSGRID_KEEPTARGETROTATION:0x00000080，表示是否尝试保持目标旋转。
- ADL_DISPLAY_SLSGRID_SAMEMODESLS_SUPPORT:0x00000010，表示是否支持samemode。
- ADL_DISPLAY_SLSGRID_MIXMODESLS_SUPPORT:0x00000020，表示是否支持mixmode。
- ADL_DISPLAY_SLSGRID_DISPLAYROTATION_SUPPORT:0x00000040，表示是否支持显示器旋转。
- ADL_DISPLAY_SLSGRID_DESKTOPROTATION_SUPPORT:0x00000080，表示是否支持桌面旋转。
- ADL_DISPLAY_SLSMAP_SLSLAYOUTMODE_FIT:0x0100，表示SLS的输出模式设置为FIT。
- ADL_DISPLAY_SLSMAP_SLSLAYOUTMODE_FILL:0x0200，表示SLS的输出模式设置为FILL。
- ADL_DISPLAY_SLSMAP_SLSLAYOUTMODE_EXPAND:0x0400，表示SLS的输出模式设置为EXPAND。
- ADL_DISPLAY_SLSMAP_IS_SLS:0x1000，表示SLS是否工作。

这些选项通常用于控制计算机桌面中特定显示器的设置。例如，如果计算机配置了一块SLS显示器，并且希望使用某种特定的显示器设置，则可以使用以下选项：

```cpp
ADL_DISPLAY_SLSGRID_CAP_OPTION_RELATIVETO_CURRENTANGLE 0x00000002
ADL_DISPLAY_SLSGRID_PORTAIT_MODE                         0x00000004
ADL_DISPLAY_SLSGRID_KEEPTARGETROTATION                  0x00000080
ADL_DISPLAY_SLSGRID_SAMEMODESLS_SUPPORT          0x00000010
ADL_DISPLAY_SLSGRID_MIXMODESLS_SUPPORT          0x00000020
ADL_DISPLAY_SLSGRID_DISPLAYROTATION_SUPPORT    0x00000040
ADL_DISPLAY_SLSGRID_DESKTOPROTATION_SUPPORT    0x00000080
ADL_DISPLAY_SLSMAP_SLSLAYOUTMODE_FIT            0x0100
ADL_DISPLAY_SLSMAP_SLSLAYOUTMODE_FILL            0x0200
ADL_DISPLAY_SLSMAP_SLSLAYOUTMODE_EXPAND        0x0400
ADL_DISPLAY_SLSMAP_IS_SLS               0x1000
```

这些选项中的一些可以使用适当的软件进行理解和修改，以获得更好的显示效果。


```cpp
#define ADL_DISPLAY_SLSGRID_CAP_OPTION_RELATIVETO_CURRENTANGLE     0x00000002
#define ADL_DISPLAY_SLSGRID_PORTAIT_MODE                         0x00000004
#define ADL_DISPLAY_SLSGRID_KEEPTARGETROTATION                  0x00000080

#define ADL_DISPLAY_SLSGRID_SAMEMODESLS_SUPPORT        0x00000010
#define ADL_DISPLAY_SLSGRID_MIXMODESLS_SUPPORT        0x00000020
#define ADL_DISPLAY_SLSGRID_DISPLAYROTATION_SUPPORT    0x00000040
#define ADL_DISPLAY_SLSGRID_DESKTOPROTATION_SUPPORT    0x00000080


#define ADL_DISPLAY_SLSMAP_SLSLAYOUTMODE_FIT        0x0100
#define ADL_DISPLAY_SLSMAP_SLSLAYOUTMODE_FILL       0x0200
#define ADL_DISPLAY_SLSMAP_SLSLAYOUTMODE_EXPAND     0x0400

#define ADL_DISPLAY_SLSMAP_IS_SLS        0x1000
```

这段代码定义了一些与SLSMAP相关的定义，包括SLSMAP的构建、克隆、显示模式配置以及SLSMAP的配置。通过这些定义，开发人员可以使用这些选项来控制SLSMAP的行为。

具体来说，以下定义定义了SLSMAP构建选项：
```cpp
#define ADL_DISPLAY_SLSMAP_IS_SLSBUILDER 0x2000
```
```cpp
#define ADL_DISPLAY_SLSMAP_IS_CLONEVT     0x4000
```
```cpp
这些选项可通过奇偶校验位来判断。
```
#define ADL_DISPLAY_SLSMAPCONFIG_GET_OPTION_RELATIVETO_LANDSCAPE         0x00000001
```cpp
```
#define ADL_DISPLAY_SLSMAPCONFIG_GET_OPTION_RELATIVETO_CURRENTANGLE     0x00000002
```cpp
这些选项允许用户在SLSMAP的配置文件中获取与LANDSCAPE和CURRENTANGLE相关的选项。
```
#define ADL_DISPLAY_SLSMAPCONFIG_CREATE_OPTION_RELATIVETO_LANDSCAPE         0x00000001
```cpp
```
#define ADL_DISPLAY_SLSMAPCONFIG_CREATE_OPTION_RELATIVETO_CURRENTANGLE     0x00000002
```cpp
这些选项允许用户创建自己的SLSMAP配置文件，并将其保存为SLSMAP的配置文件。
```
#define ADL_DISPLAY_SLSMAPCONFIG_REARRANGE_OPTION_RELATIVETO_LANDSCAPE     0x00000001
```cpp
```
#define ADL_DISPLAY_SLSMAPCONFIG_REARRANGE_OPTION_RELATIVETO_CURRENTANGLE     0x00000002
```cpp
这些选项允许用户在SLSMAP的配置文件中重新排列选项，以便更好地组织他们的配置文件。
```
#define ADL_SLS_SAMEMODESLS_SUPPORT         0x0001
```cpp
```
#define ADL_SLS_MIXMODESLS_SUPPORT          0x0002
```cpp
```
#define ADL_SLS_DISPLAYROTATIONSLS_SUPPORT  0x0004
```cpp
这些选项定义了SLSMAP支持的显示模式，包括ON、OFF、以及混合模式。


```
#define ADL_DISPLAY_SLSMAP_IS_SLSBUILDER 0x2000
#define ADL_DISPLAY_SLSMAP_IS_CLONEVT     0x4000

#define ADL_DISPLAY_SLSMAPCONFIG_GET_OPTION_RELATIVETO_LANDSCAPE         0x00000001
#define ADL_DISPLAY_SLSMAPCONFIG_GET_OPTION_RELATIVETO_CURRENTANGLE     0x00000002

#define ADL_DISPLAY_SLSMAPCONFIG_CREATE_OPTION_RELATIVETO_LANDSCAPE         0x00000001
#define ADL_DISPLAY_SLSMAPCONFIG_CREATE_OPTION_RELATIVETO_CURRENTANGLE     0x00000002

#define ADL_DISPLAY_SLSMAPCONFIG_REARRANGE_OPTION_RELATIVETO_LANDSCAPE     0x00000001
#define ADL_DISPLAY_SLSMAPCONFIG_REARRANGE_OPTION_RELATIVETO_CURRENTANGLE     0x00000002

#define ADL_SLS_SAMEMODESLS_SUPPORT         0x0001
#define ADL_SLS_MIXMODESLS_SUPPORT          0x0002
#define ADL_SLS_DISPLAYROTATIONSLS_SUPPORT  0x0004
```cpp

这段代码定义了一系列常量，用于定义SLS(Surface level结构)中与桌面旋转和配置相关的标志位。

具体来说，这些常量定义了以下标志位：

- ADL_SLS_DESKTOPROTATIONSLS_SUPPORT：表示是否支持桌面旋转。如果设置为0，则不支持。
- ADL_SLS_TARGETS_INVALID：表示当前激活的目标是否有效。如果设置为0，则表示所有目标都无效。
- ADL_SLS_MODES_INVALID：表示当前设备是否支持SLS。如果设置为0，则表示所有设备都不支持SLS。
- ADL_SLS_ROTATIONS_INVALID：表示当前设备是否支持旋转。如果设置为0，则表示所有设备都不支持旋转。
- ADL_SLS_POSITIONS_INVALID：表示当前设备是否支持位置。如果设置为0，则表示所有设备都不支持位置。
- ADL_SLS_LAYOUTMODE_INVALID：表示当前设备是否支持布局模式。如果设置为0，则表示所有设备都不支持布局模式。
- ADL_DISPLAY_SLSDISPLAYOFFSET_VALID：表示当前设备是否支持显示偏移量。如果设置为0，则表示所有设备都不支持显示偏移量。
- ADL_DISPLAY_SLSGRID_RELATIVETO_LANDSCAPE：表示显示网格相对于用户空间的位置。如果设置为0，则表示所有显示都位于屏幕的中心位置。如果设置为1，则表示显示网格相对于用户空间的位置相对于屏幕底部。
- ADL_DISPLAY_SLSGRID_RELATIVETO_CURRENTANGLE：表示当前设备是否支持当前角度。如果设置为0，则表示所有设备都不支持当前角度。如果设置为1，则表示当前设备支持当前角度。

这些常量的作用是定义SLS中的相关标志位，用于控制设备是否支持桌面旋转、布局模式、位置等相关功能。


```
#define ADL_SLS_DESKTOPROTATIONSLS_SUPPORT  0x0008

#define ADL_SLS_TARGETS_INVALID     0x0001
#define ADL_SLS_MODES_INVALID       0x0002
#define ADL_SLS_ROTATIONS_INVALID   0x0004
#define ADL_SLS_POSITIONS_INVALID   0x0008
#define ADL_SLS_LAYOUTMODE_INVALID  0x0010

#define ADL_DISPLAY_SLSDISPLAYOFFSET_VALID        0x0002

#define ADL_DISPLAY_SLSGRID_RELATIVETO_LANDSCAPE         0x00000010
#define ADL_DISPLAY_SLSGRID_RELATIVETO_CURRENTANGLE     0x00000020


/// The bit mask identifies displays is currently in bezel mode.
```cpp

这段代码定义了一系列用于SLS（Smart Link Succession）技术的定义，主要用于显示器ADL（Advanced Display Link）的设置。下面是每个定义的作用：

1. `#define ADL_DISPLAY_SLSMAP_BEZELMODE`：定义了一个名为`ADL_DISPLAY_SLSMAP_BEZELMODE`的宏，表示从给定的显示列表中选择Bezel（轮播）显示模式。

2. `#define ADL_DISPLAY_SLSMAP_DISPLAYARRANGED`：定义了一个名为`ADL_DISPLAY_SLSMAP_DISPLAYARRANGED`的宏，表示显示列表是否处于排列状态。

3. `#define ADL_DISPLAY_SLSMAP_CURRENTCONFIG`：定义了一个名为`ADL_DISPLAY_SLSMAP_CURRENTCONFIG`的宏，表示当前适配器是否正在使用指定的显示列表。

4. `#define ADL_DISPLAY_SLSMAPINDEXLIST_OPTION_ACTIVE`：定义了一个名为`ADL_DISPLAY_SLSMAPINDEXLIST_OPTION_ACTIVE`的宏，表示SLS映射列表是否处于活动状态。

5. `#define ADL_DISPLAY_BEZELOFFSET_STEPBYSTEPSET`：定义了一个名为`ADL_DISPLAY_BEZELOFFSET_STEPBYSTEPSET`的宏，表示从Bezel显示模式中减小Bezel偏移量的步长。

6. `#define ADL_DISPLAY_BEZELOFFSET_COMMIT`：定义了一个名为`ADL_DISPLAY_BEZELOFFSET_COMMIT`的宏，表示Bezel显示模式中减小Bezel偏移量是否已经完成。

7. `typedef enum _SLS_ImageCropType {`：定义了一个名为`SLS_ImageCropType`的枚举类型，包含了 Fit、Fill 和 Expand 等图像裁剪类型。

8. `Fit`：定义了一个名为`Fit`的枚举类型，表示图像裁剪类型为Fit。

9. `Fill`：定义了一个名为`Fill`的枚举类型，表示图像裁剪类型为Fill。

10. `Expand`：定义了一个名为`Expand`的枚举类型，表示图像裁剪类型为Expand。


```
#define ADL_DISPLAY_SLSMAP_BEZELMODE            0x00000010
/// The bit mask identifies displays from this map is arranged.
#define ADL_DISPLAY_SLSMAP_DISPLAYARRANGED        0x00000002
/// The bit mask identifies this map is currently in used for the current adapter.
#define ADL_DISPLAY_SLSMAP_CURRENTCONFIG        0x00000004

///For onlay active SLS  map info
#define ADL_DISPLAY_SLSMAPINDEXLIST_OPTION_ACTIVE        0x00000001

///For Bezel
#define ADL_DISPLAY_BEZELOFFSET_STEPBYSTEPSET            0x00000004
#define ADL_DISPLAY_BEZELOFFSET_COMMIT                    0x00000008

typedef enum _SLS_ImageCropType {
    Fit = 1,
    Fill = 2,
    Expand = 3
}SLS_ImageCropType;


```cpp

这段代码定义了一个枚举类型`_DceSettingsType`和`_DpLinkRate`，用于描述`Dce`和`dp`设置。

`_DceSettingsType`枚举类型定义了四种设置：`DceSetting_HdmiLq`、`DceSetting_DpSettings`、`DceSetting_Protection`。

`_DpLinkRate`枚举类型定义了五种设置：`DPLinkRate_Unknown`、`DPLinkRate_RBR`、`DPLinkRate_HBR`、`DPLinkRate_HBR2`、`DPLinkRate_HBR3`。

然后，在接下来的文本中，没有对这两个枚举类型进行使用，因此没有对`_DceSettingsType`和`_DpLinkRate`进行设置。


```
typedef enum _DceSettingsType {
    DceSetting_HdmiLq,
    DceSetting_DpSettings,
    DceSetting_Protection
} DceSettingsType;

typedef enum _DpLinkRate {
    DPLinkRate_Unknown,
    DPLinkRate_RBR,
    DPLinkRate_HBR,
    DPLinkRate_HBR2,
    DPLinkRate_HBR3
} DpLinkRate;

// @}

```cpp

这段代码定义了一系列常量，用于定义PowerXpress中的ADLPXConfigCaps和ADLPXSchemeRange中的各种选项。

具体来说，定义了以下选项：
- ADL_PX_CONFIGCAPS_SPLASHSCREEN_SUPPORT：表示是否支持使用SPLASHSCREEN模式
- ADL_PX_CONFIGCAPS_CF_SUPPORT：表示是否支持CF模式
- ADL_PX_CONFIGCAPS_MUXLESS：表示是否支持混合负载模式
- ADL_PX_CONFIGCAPS_PROFILE_COMPLIANT：表示是否支持PowerXpress支持的配置文件规范
- ADL_PX_CONFIGCAPS_NON_AMD_DRIVEN_DISPLAY：表示是否支持非AMD驱动的显示
- ADL_PX_CONFIGCAPS_FIXED_SUPPORT：表示是否支持固定模式的PowerXpress连接
- ADL_PX_CONFIGCAPS_DYNAMIC_SUPPORT：表示是否支持动态模式的PowerXpress连接
- ADL_PX_CONFIGCAPS_HIDE_AUTO_SWITCH：表示是否支持隐藏自动开关


```
///\defgroup define_powerxpress_constants PowerXpress Definitions
/// @{

/// The bit mask identifies PX caps for ADLPXConfigCaps.iPXConfigCapMask and ADLPXConfigCaps.iPXConfigCapValue
#define    ADL_PX_CONFIGCAPS_SPLASHSCREEN_SUPPORT        0x0001
#define    ADL_PX_CONFIGCAPS_CF_SUPPORT                0x0002
#define    ADL_PX_CONFIGCAPS_MUXLESS                    0x0004
#define    ADL_PX_CONFIGCAPS_PROFILE_COMPLIANT            0x0008
#define    ADL_PX_CONFIGCAPS_NON_AMD_DRIVEN_DISPLAYS    0x0010
#define ADL_PX_CONFIGCAPS_FIXED_SUPPORT             0x0020
#define ADL_PX_CONFIGCAPS_DYNAMIC_SUPPORT           0x0040
#define ADL_PX_CONFIGCAPS_HIDE_AUTO_SWITCH            0x0080

/// The bit mask identifies PX schemes for ADLPXSchemeRange
#define ADL_PX_SCHEMEMASK_FIXED                        0x0001
```cpp

这段代码定义了一个名为ADL_PX_SCHEMEMASK_DYNAMIC的宏，它的值为0x0002。接着定义了一个名为ADL_PX_SCHEME的枚举类型，它包含三个成员：ADL_PX_SCHEME_INVALID、ADL_PX_SCHEME_FIXED和ADL_PX_SCHEME_DYNAMIC，分别对应枚举类型中的0、1和2。最后，定义了一个名为PXScheme的枚举类型，包含四个成员：PX_SCHEME_INVALID、PX_SCHEME_FIXED、PX_SCHEME_DYNAMIC和PX_SCHEME_TELEPORT。没有定义任何函数或变量，可能是在声明某些变量或函数时被用来定义某些常量。


```
#define ADL_PX_SCHEMEMASK_DYNAMIC                    0x0002

/// PX Schemes
typedef enum _ADLPXScheme
{
    ADL_PX_SCHEME_INVALID   = 0,
    ADL_PX_SCHEME_FIXED     = ADL_PX_SCHEMEMASK_FIXED,
    ADL_PX_SCHEME_DYNAMIC   = ADL_PX_SCHEMEMASK_DYNAMIC
}ADLPXScheme;

/// Just keep the old definitions for compatibility, need to be removed later
typedef enum PXScheme
{
    PX_SCHEME_INVALID   = 0,
    PX_SCHEME_FIXED     = 1,
    PX_SCHEME_DYNAMIC   = 2
} PXScheme;


```cpp

这段代码定义了一个名为 "ApplicationListType" 的枚举类型，其包含六个枚举值：ADL_PX40_MRU、ADL_PX40_MISSED、ADL_PX40_DISCRETE、ADL_PX40_INTEGRATED、ADL_MMD_PROFILED 和 ADL_PX40_TOTAL。这些枚举值分别表示了应用程序的配置文件中包含的属性。

在代码的下一行，定义了一个名为 "ApplicationProfile" 的枚举类型，包含六个枚举值：ADL_APP_PROFILE_FILENAME_LENGTH、ADL_APP_PROFILE_TIMESTAMP_LENGTH、ADL_APP_PROFILE_VERSION_LENGTH 和 ADL_APP_PROFILE_PROPERTY_LENGTH。这些枚举值用于表示应用程序配置文件中不同属性的长度。

然后，定义了一个名为 "ApplicationList" 的结构体，包含一个指向 "ApplicationProfile" 类型的指针和一个表示枚举类型 "ApplicationListType" 的成员。这个结构体用于表示应用程序配置文件中的配置信息。

最后，定义了一个名为 "ApplicationListProfile" 的函数，用于将 "ApplicationProfile" 和 "ApplicationList" 结构体拼接在一起，并将结果保存到应用程序配置文件中。


```
/// @}

///\defgroup define_appprofiles For Application Profiles
/// @{

#define ADL_APP_PROFILE_FILENAME_LENGTH        256
#define ADL_APP_PROFILE_TIMESTAMP_LENGTH    32
#define ADL_APP_PROFILE_VERSION_LENGTH        32
#define ADL_APP_PROFILE_PROPERTY_LENGTH        64

enum ApplicationListType
{
    ADL_PX40_MRU,
    ADL_PX40_MISSED,
    ADL_PX40_DISCRETE,
    ADL_PX40_INTEGRATED,
    ADL_MMD_PROFILED,
    ADL_PX40_TOTAL
};

```cpp

这段代码定义了一个枚举类型ADLProfilePropertyType，该枚举类型包括以下六个成员：

- ADL_PROFILEPROPERTY_TYPE_BINARY：二进制数据类型，表示为实现DP12串口通信的设备。
- ADL_PROFILEPROPERTY_TYPE_BOOLEAN：布尔类型，表示DP12串口通信的设备。
- ADL_PROFILEPROPERTY_TYPE_DWORD：无符号32位整型，表示DP12串口通信的设备。
- ADL_PROFILEPROPERTY_TYPE_QWORD：有符号8位整型，表示DP12串口通信的设备。
- ADL_PROFILEPROPERTY_TYPE_ENUMERATED：表示一个或多个连续的枚举类型，用于明确DP12串口通信的设备。
- ADL_PROFILEPROPERTY_TYPE_STRING：表示字符串类型的设备，用于DP12串口通信的设备。

这个枚举类型定义了DP12串口通信设备的多种属性，使得开发人员可以使用单一的接口来访问它们的各种属性。


```
typedef enum _ADLProfilePropertyType
{
    ADL_PROFILEPROPERTY_TYPE_BINARY        = 0,
    ADL_PROFILEPROPERTY_TYPE_BOOLEAN,
    ADL_PROFILEPROPERTY_TYPE_DWORD,
    ADL_PROFILEPROPERTY_TYPE_QWORD,
    ADL_PROFILEPROPERTY_TYPE_ENUMERATED,
    ADL_PROFILEPROPERTY_TYPE_STRING
}ADLProfilePropertyType;


/// @}

///\defgroup define_dp12 For Display Port 1.2
/// @{

```cpp

这段代码定义了一个名为ADL的命名约定，用于描述最大相对地址链接数。其中，ADL_MAX_RAD_LINK_COUNT表示最大相对地址链接数，即多个不同的GAMUT状态可以链接的最大数量。

接下来，定义了一个名为ADL_GAMUT_REFERENCE_SOURCE的标志，用于描述GAMUT状态是否与源GAMUT相关联。还定义了一个名为ADL_CUSTOM_WHITE_POINT的标志，用于描述源GAMUT的坐标是否为白色点。最后，定义了一个名为ADL_CUSTOM_GAMUT的标志，用于描述如何从结构体ADLGamutData中读取GAMUT状态。

这些标志用于描述GAMUT状态与源或目标相关的信息，以及如何从源结构体中读取信息。这些信息可以用于实现类似于在不同GAMUT之间传输数据的功能。


```
/// Maximum Relative Address Link
#define ADL_MAX_RAD_LINK_COUNT    15

/// @}

///\defgroup defines_gamutspace Driver Supported Gamut Space
/// @{

/// The flags desribes that gamut is related to source or to destination and to overlay or to graphics
#define ADL_GAMUT_REFERENCE_SOURCE       (1 << 0)
#define ADL_GAMUT_GAMUT_VIDEO_CONTENT    (1 << 1)

/// The flags are used to describe the source of gamut and how read information from struct ADLGamutData
#define ADL_CUSTOM_WHITE_POINT           (1 << 0)
#define ADL_CUSTOM_GAMUT                 (1 << 1)
```cpp

这段代码定义了一系列的macro定义，用于定义ADL gamut中的空间。这些macro定义包括了一些标志位，例如空间是否为预定义空间，以及是否支持应用控制等。通过将这些标志位组合起来，可以定义出不同的ADL gamut空间。

具体来说，这段代码定义了以下几个macro：

1. ADL_GAMUT_REMAP_ONLY：表示是否重新映射ADL gamut空间。
2. ADL_GAMUT_SPACE_CCIR_709：表示空间为CCIR 709（ Color Comp面对这种709标准的显示器）。
3. ADL_GAMUT_SPACE_CCIR_601：表示空间为CCIR 601（ Color Compồ这一点601标准的显示器）。
4. ADL_GAMUT_SPACE_ADOBE_RGB：表示空间为Adobe RGB色空间。
5. ADL_GAMUT_SPACE_CIE_RGB：表示空间为CIE RGB色空间。
6. ADL_GAMUT_SPACE_CUSTOM：表示是否为预定义空间，可以通过组合多个定义的标志位来定义一个新的空间。
7. ADL_WHITE_POINT_5000K：表示预定义的白色点，用于计算其他颜色。
8. ADL_WHITE_POINT_6500K：表示预定义的白色点，用于计算其他颜色。

通过这些macro定义，可以组合出不同的ADL gamut空间，以满足不同的应用需求。


```
#define ADL_GAMUT_REMAP_ONLY             (1 << 2)

/// The define means the predefined gamut values  .
///Driver uses to find entry in the table and apply appropriate gamut space.
#define ADL_GAMUT_SPACE_CCIR_709     (1 << 0)
#define ADL_GAMUT_SPACE_CCIR_601     (1 << 1)
#define ADL_GAMUT_SPACE_ADOBE_RGB    (1 << 2)
#define ADL_GAMUT_SPACE_CIE_RGB      (1 << 3)
#define ADL_GAMUT_SPACE_CUSTOM       (1 << 4)
#define ADL_GAMUT_SPACE_CCIR_2020    (1 << 5)
#define ADL_GAMUT_SPACE_APPCTRL      (1 << 6)

/// Predefine white point values are structed similar to gamut .
#define ADL_WHITE_POINT_5000K       (1 << 0)
#define ADL_WHITE_POINT_6500K       (1 << 1)
```cpp

这段代码定义了一系列的宏，用于定义图像增强中的Gamut和White Point坐标，以及Regamma系数。

首先，定义了三个Macro，分别是：

```
#define ADL_WHITE_POINT_7500K       (1 << 2)
#define ADL_WHITE_POINT_9300K       (1 << 3)
#define ADL_WHITE_POINT_CUSTOM      (1 << 4)
```cpp

这些Macro用于定义图像增强中的White Point坐标，其中，7500K和9300K是固定的，而CUSTOM则是一个可配置的Macro，可以根据需要进行自定义。

接着，定义了一个名为ADL_GAMUT_WHITEPOINT_DIVIDER的Macro，用于定义Gamut的系数。

```
#define ADL_GAMUT_WHITEPOINT_DIVIDER           10000
```cpp

然后，定义了三个Regamma系数：

```
#define ADL_REGAMMA_COEFFICIENT_A0_DIVIDER       10000000
#define ADL_REGAMMA_COEFFICIENT_A1A2A3_DIVIDER   1000
```cpp

其中，A0系数是描述了白点是否来自预设的EDID值，而A1，A2，A3系数则是描述了Gamut的系数，它们的使用需要用到上面定义的ADL_GAMUT_WHITEPOINT_DIVIDER。

接下来，定义了一个名为ADL_EDID_REGAMMA_COEFFICIENTS的Macro，用于描述系数是否来自预设的EDID值。

```
#define ADL_EDID_REGAMMA_COEFFICIENTS          (1 << 0)
```cpp

最后，定义了一个名为ADL_WHITE_POINT_CUSTOM的Macro，用于描述White Point坐标是否来自用户自定义的值。

综上所述，这段代码定义了一系列的Macro，用于定义图像增强中的Gamut和White Point坐标，以及Regamma系数，并且可以根据需要进行自定义。


```
#define ADL_WHITE_POINT_7500K       (1 << 2)
#define ADL_WHITE_POINT_9300K       (1 << 3)
#define ADL_WHITE_POINT_CUSTOM      (1 << 4)

///gamut and white point coordinates are from 0.0 -1.0 and divider is used to find the real value .
/// X float = X int /divider
#define ADL_GAMUT_WHITEPOINT_DIVIDER           10000

///gamma a0 coefficient uses the following divider:
#define ADL_REGAMMA_COEFFICIENT_A0_DIVIDER       10000000
///gamma a1 ,a2,a3 coefficients use the following divider:
#define ADL_REGAMMA_COEFFICIENT_A1A2A3_DIVIDER   1000

///describes whether the coefficients are from EDID or custom user values.
#define ADL_EDID_REGAMMA_COEFFICIENTS          (1 << 0)
```cpp

这段代码定义了一个名为 ADLRegamma 的结构体，用于控制软件中图像处理过程中的伽马值。它包括多个宏，每个宏都指定了一个或多个 gamma 值，这些值用于控制是否应用 de gamma 校正或 regamma 校正。

具体来说，这段代码定义了以下几个全局变量：

- ADL_USE_GAMMA_RAMP：如果设置了这个宏，那么驱动程序将应用 gamma ramp，否则将应用 regamma。
- ADL_APPLY_DEGAMMA：如果设置了这个宏，那么将应用 de gamma 校正。
- ADL_EDID_REGAMMA_PREDEFINED_SRGB：如果设置了这个宏，那么将应用 PQ 伽马曲线，但是这个应用是有限制的，最大允许 nits 为 2084。
- ADL_EDID_REGAMMA_PREDEFINED_PQ：如果设置了这个宏，那么将应用 PQ 伽马曲线，但是这个应用是有限制的，最大允许 nits 为 2084，同时使用 2084nits 模式。
- ADL_EDID_REGAMMA_PREDEFINED_36：如果设置了这个宏，那么将应用 3.6 伽马曲线。
- ADL_EDID_REGAMMA_PREDEFINED_BT709：如果设置了这个宏，那么将应用 BT709 伽马曲线。

此外，还有一两个宏，它们定义了 gamma 值为 1 时需要做的事情：

- ADL_USE_GAMMA_RAMP：如果设置了这个宏，那么将禁用 gamma ramp，并将 de gamma 和 regamma 应用限制设置为 1。
- ADL_APPLY_DEGAMMA：如果设置了这个宏，那么将禁用 gamma ramp，并将 de gamma 和 regamma 应用限制设置为 1。


```
///Used for struct ADLRegamma. Feature if set use gamma ramp, if missing use regamma coefficents
#define ADL_USE_GAMMA_RAMP                     (1 << 4)
///Used for struct ADLRegamma. If the gamma ramp flag is used then the driver could apply de gamma corretion to the supplied curve and this depends on this flag
#define ADL_APPLY_DEGAMMA                      (1 << 5)
///specifies that standard SRGB gamma should be applied
#define ADL_EDID_REGAMMA_PREDEFINED_SRGB       (1 << 1)
///specifies that PQ gamma curve should be applied
#define ADL_EDID_REGAMMA_PREDEFINED_PQ         (1 << 2)
///specifies that PQ gamma curve should be applied, lower max nits
#define ADL_EDID_REGAMMA_PREDEFINED_PQ_2084_INTERIM (1 << 3)
///specifies that 3.6 gamma should be applied
#define ADL_EDID_REGAMMA_PREDEFINED_36         (1 << 6)
///specifies that BT709 gama should be applied
#define ADL_EDID_REGAMMA_PREDEFINED_BT709      (1 << 7)
///specifies that regamma should be disabled, and application controls regamma content (of the whole screen)
```cpp

这段代码定义了一个名为ADL_EDID_REGAMMA_PREDEFINED_APPCTRL的宏，其值为(1 << 8)。这个宏的含义是，它定义了属于IPanelPixelFormat的结构体的一些常量，包括ADL_DISPLAY_DDCINFO_PIXEL_FORMAT_RGB656、ADL_DISPLAY_DDCINFO_PIXEL_FORMAT_RGB666、ADL_DISPLAY_DDCINFO_PIXEL_FORMAT_RGB888、ADL_DISPLAY_DDCINFO_PIXEL_FORMAT_RGB101010和ADL_DISPLAY_DDCINFO_PIXEL_FORMAT_RGB161616。

这些常量的目的是定义了像素格式的输出，用于在APPCTRL字段中设置与DDCSupport相关的图像数据，以在DDCSupport功能正常工作时产生。这些常量在下面的说明中详细说明：

```
#define ADL_DISPLAY_DDCINFO_PIXEL_FORMAT_RGB656
```cpp

这个常量的含义是，它定义了一个名为RGB656的像素格式，用于在DDCSupport功能正常工作时产生。

```
#define ADL_DISPLAY_DDCINFO_PIXEL_FORMAT_RGB666
```cpp

这个常量的含义是，它定义了一个名为RGB666的像素格式，用于在DDCSupport功能正常工作时产生。

```
#define ADL_DISPLAY_DDCINFO_PIXEL_FORMAT_RGB888
```cpp

这个常量的含义是，它定义了一个名为RGB888的像素格式，用于在DDCSupport功能正常工作时产生。

```
#define ADL_DISPLAY_DDCINFO_PIXEL_FORMAT_RGB101010
```cpp

这个常量的含义是，它定义了一个名为RGB101010的像素格式，用于在DDCSupport功能正常工作时产生。

```
#define ADL_DISPLAY_DDCINFO_PIXEL_FORMAT_RGB161616
```cpp

这个常量的含义是，它定义了一个名为RGB161616的像素格式，用于在DDCSupport功能正常工作时产生。

```
#define ADL_DISPLAY_DDCINFO_PIXEL_FORMAT_RGB_RESERVED1
```cpp

这个常量的含义是，它定义了一个名为RGB_RESERVED1的像素格式，用于在DDCSupport功能正常工作时产生。

```
#define ADL_DISPLAY_DDCINFO_PIXEL_FORMAT_RGB_RESERVED2
```cpp

这个常量的含义是，它定义了一个名为RGB_RESERVED2的像素格式，用于在DDCSupport功能正常工作时产生。

```
#define ADL_DISPLAY_DDCINFO_PIXEL_FORMAT_RGB_RESERVED3
```cpp

这个常量的含义是，它定义了一个名为RGB_RESERVED3的像素格式，用于在DDCSupport功能正常工作时产生。
```


```cpp
#define ADL_EDID_REGAMMA_PREDEFINED_APPCTRL    (1 << 8)

/// @}

/// \defgroup define_ddcinfo_pixelformats DDCInfo Pixel Formats
/// @{
/// defines for iPanelPixelFormat  in struct ADLDDCInfo2
#define ADL_DISPLAY_DDCINFO_PIXEL_FORMAT_RGB656                       0x00000001L
#define ADL_DISPLAY_DDCINFO_PIXEL_FORMAT_RGB666                       0x00000002L
#define ADL_DISPLAY_DDCINFO_PIXEL_FORMAT_RGB888                       0x00000004L
#define ADL_DISPLAY_DDCINFO_PIXEL_FORMAT_RGB101010                    0x00000008L
#define ADL_DISPLAY_DDCINFO_PIXEL_FORMAT_RGB161616                    0x00000010L
#define ADL_DISPLAY_DDCINFO_PIXEL_FORMAT_RGB_RESERVED1                0x00000020L
#define ADL_DISPLAY_DDCINFO_PIXEL_FORMAT_RGB_RESERVED2                0x00000040L
#define ADL_DISPLAY_DDCINFO_PIXEL_FORMAT_RGB_RESERVED3                0x00000080L
```

```cppcss
#define ADL_DISPLAY_DDCINFO_PIXEL_FORMAT_XRGB_BIAS101010                0x00000100L
#define ADL_DISPLAY_DDCINFO_PIXEL_FORMAT_YCBCR444_8BPCC               0x00000200L
#define ADL_DISPLAY_DDCINFO_PIXEL_FORMAT_YCBCR444_10BPCC              0x00000400L
#define ADL_DISPLAY_DDCINFO_PIXEL_FORMAT_YCBCR444_12BPCC              0x00000800L
#define ADL_DISPLAY_DDCINFO_PIXEL_FORMAT_YCBCR422_8BPCC               0x00001000L
#define ADL_DISPLAY_DDCINFO_PIXEL_FORMAT_YCBCR422_10BPCC              0x00010000L
#define ADL_DISPLAY_DDCINFO_PIXEL_FORMAT_YCBCR422_12BPCC              0x00020000L
#define ADL_DISPLAY_DDCINFO_PIXEL_FORMAT_YCBCR420_8BPCC               0x00040000L
#define ADL_DISPLAY_DDCINFO_PIXEL_FORMAT_YCBCR420_10BPCC              0x00080000L
#define ADL_DISPLAY_DDCINFO_PIXEL_FORMAT_YCBCR420_12BPCC              0x00100000L
#define ADL_DISPLAY_DDCINFO_PIXEL_FORMAT_YCBCR420_8BPCC               0x00200000L
#define ADL_DISPLAY_DDCINFO_PIXEL_FORMAT_YCBCR420_10BPCC              0x00400000L
#define ADL_DISPLAY_DDCINFO_PIXEL_FORMAT_YCBCR420_12BPCC              0x00800000L
#define ADL_DISPLAY_DDCINFO_PIXEL_FORMAT_YCBCR420_8BPCC               0x01000000L
#define ADL_DISPLAY_DDCINFO_PIXEL_FORMAT_YCBCR420_10BPCC              0x02000000L
#define ADL_DISPLAY_DDCINFO_PIXEL_FORMAT_YCBCR420_12BPCC              0x04000000L
#define ADL_DISPLAY_DDCINFO_PIXEL_FORMAT_YCBCR420_8BPCC               0x08000000L
#define ADL_DISPLAY_DDCINFO_PIXEL_FORMAT_YCBCR420_10BPCC              0x10000000L
#define ADL_DISPLAY_DDCINFO_PIXEL_FORMAT_YCBCR420_12BPCC              0x20000000L
#define ADL_DISPLAY_DDCINFO_PIXEL_FORMAT_YCBCR420_8BPCC               0x40000000L
#define ADL_DISPLAY_DDCINFO_PIXEL_FORMAT_YCBCR420_10BPCC              0x80000000L
#define ADL_DISPLAY_DDCINFO_PIXEL_FORMAT_YCBCR420_12BPCC              0x10000000L
#define ADL_DISPLAY_DDCINFO_PIXEL_FORMAT_YCBCR420_8BPCC               0x20000000L
#define ADL_DISPLAY_DDCINFO_PIXEL_FORMAT_YCBCR420_10BPCC              0x40000000L
#define ADL_DISPLAY_DDCINFO_PIXEL_FORMAT_YCBCR420_12BPCC              0x80000000L
#define ADL_DISPLAY_DDCINFO_PIXEL_FORMAT_YCBCR420_8BPCC               0x80800000L
#define ADL_DISPLAY_DDCINFO_PIXEL_FORMAT_YCBCR420_10BPCC              0x10100000L
#define ADL_DISPLAY_DDCINFO_PIXEL_FORMAT_YCBCR420_12BPCC              0x20200000L
#define ADL_DISPLAY_DDCINFO_PIXEL_FORMAT_YCBCR420_8BPCC               0x40400000L
#define ADL_DISPLAY_DDCINFO_PIXEL_FORMAT_YCBCR420_10BPCC              0x80800000L
#define ADL_DISPLAY_DDCINFO_PIXEL_FORMAT_YCBCR420_12BPCC              0x10100000L
#define ADL_DISPLAY_DDCINFO_PIXEL_FORMAT_YCBCR420_8BPCC               0x20200000L
#define ADL_DISPLAY_DDCINFO_PIXEL_FORMAT_YCBCR420_10BPCC              0x40400000L
#define ADL_DISPLAY_DDCINFO_PIXEL_FORMAT_YCBCR4


```
#define ADL_DISPLAY_DDCINFO_PIXEL_FORMAT_XRGB_BIAS101010              0x00000100L
#define ADL_DISPLAY_DDCINFO_PIXEL_FORMAT_YCBCR444_8BPCC               0x00000200L
#define ADL_DISPLAY_DDCINFO_PIXEL_FORMAT_YCBCR444_10BPCC              0x00000400L
#define ADL_DISPLAY_DDCINFO_PIXEL_FORMAT_YCBCR444_12BPCC              0x00000800L
#define ADL_DISPLAY_DDCINFO_PIXEL_FORMAT_YCBCR422_8BPCC               0x00001000L
#define ADL_DISPLAY_DDCINFO_PIXEL_FORMAT_YCBCR422_10BPCC              0x00002000L
#define ADL_DISPLAY_DDCINFO_PIXEL_FORMAT_YCBCR422_12BPCC              0x00004000L
#define ADL_DISPLAY_DDCINFO_PIXEL_FORMAT_YCBCR420_8BPCC               0x00008000L
#define ADL_DISPLAY_DDCINFO_PIXEL_FORMAT_YCBCR420_10BPCC              0x00010000L
#define ADL_DISPLAY_DDCINFO_PIXEL_FORMAT_YCBCR420_12BPCC              0x00020000L
/// @}

/// \defgroup define_source_content_TF ADLSourceContentAttributes transfer functions (gamma)
/// @{
/// defines for iTransferFunction in ADLSourceContentAttributes
```cpp



这段代码定义了一个名为 `ADL_TF_` 的命名体，其中定义了几个常量，分别代表不同的图像色彩空间和参数。

具体来说，这些常量定义了以下内容：

- `ADL_TF_sRGB`：代表sRGB色彩空间，是一种国际标准化的色彩空间，适合用于数字图像和视频编码中。
- `ADL_TF_BT709`：代表BT.709色彩空间，也称为sRGB色彩空间，是一种广泛使用的色彩空间，适合用于电视和电影屏幕的颜色还原。
- `ADL_TF_PQ2084`：代表PQ2084色彩空间，这是一种类似于BT.709的色彩空间，但具有更宽的颜色 gamut，更适合于色彩饱和度和色彩偏差的修复。
- `ADL_TF_PQ2084_INTERIM`：代表PQ2084-Interim色彩空间，是一种暂时的、实验性的色彩空间，用于在某些BT.709修复方案中使用。
- `ADL_TF_LINEAR_0_1`：代表线性0-1色彩空间，适合于一些色彩准确度和颜色表现力的要求。
- `ADL_TF_LINEAR_0_125`：代表线性0-125色彩空间，适合于更多的色彩表现力和色彩准确度要求。
- `ADL_TF_DOLBYVISION`：代表DolbyVision色彩空间，适合于Dolby Vision高动态范围电视信号的传输和接收。
- `ADL_TF_GAMMA_22`：代表Gamma 2.2色彩空间，适合于一些高动态范围电视信号的传输和接收，如HDR电视信号。

此外，还有一组定义了上述常量的别名，分别使用了 `ADL_CS_` 和 `ADL_CS_BT601` 作为前缀，方便用户在使用时快速查阅。


```
#define ADL_TF_sRGB                0x0001      ///< sRGB
#define ADL_TF_BT709            0x0002      ///< BT.709
#define ADL_TF_PQ2084            0x0004      ///< PQ2084
#define ADL_TF_PQ2084_INTERIM    0x0008        ///< PQ2084-Interim
#define ADL_TF_LINEAR_0_1        0x0010      ///< Linear 0 - 1
#define ADL_TF_LINEAR_0_125        0x0020      ///< Linear 0 - 125
#define ADL_TF_DOLBYVISION        0x0040      ///< DolbyVision
#define ADL_TF_GAMMA_22         0x0080      ///< Plain 2.2 gamma curve
/// @}

/// \defgroup define_source_content_CS ADLSourceContentAttributes color spaces
/// @{
/// defines for iColorSpace in ADLSourceContentAttributes
#define ADL_CS_sRGB                0x0001      ///< sRGB
#define ADL_CS_BT601             0x0002      ///< BT.601
```cpp

这是一个C++代码中的定义，为了一些与高清（HDR）相关的选项。这些定义属于名为“ADL”的头文件。

具体来说，这些定义包括：

- “ADL_CS_BT709”：表示这是BT.709格式的定义。
- “ADL_CS_BT2020”：表示这是BT.2020格式的定义。
- “ADL_CS_ADOBE”：表示这是Adobe RGB格式的定义。
- “ADL_CS_P3”：表示这是DCI-P3格式的定义。
- “ADL_CS_scRGB_MS_REF”：表示这是scRGB（MS参考）格式的定义。
- “ADL_CS_DISPLAY_NATIVE”：表示这是显示为原生（非压缩）的定义。
- “ADL_CS_APP_CONTROL”：表示这是应用程序控制的定义。
- “ADL_CS_DOLBYVISION”：表示这是DolbyVision格式的定义。

这里定义了一些与高清（HDR）相关的选项，用于在ADL格式的数据中使用它们。


```
#define ADL_CS_BT709            0x0004      ///< BT.709
#define ADL_CS_BT2020            0x0008      ///< BT.2020
#define ADL_CS_ADOBE            0x0010      ///< Adobe RGB
#define ADL_CS_P3                0x0020      ///< DCI-P3
#define ADL_CS_scRGB_MS_REF        0x0040      ///< scRGB (MS Reference)
#define ADL_CS_DISPLAY_NATIVE    0x0080      ///< Display Native
#define ADL_CS_APP_CONTROL         0x0100      ///< Application Controlled
#define ADL_CS_DOLBYVISION      0x0200      ///< DolbyVision
/// @}

/// \defgroup define_HDR_support ADLDDCInfo2 HDR support options
/// @{
/// defines for iSupportedHDR in ADLDDCInfo2
#define ADL_HDR_CEA861_3        0x0001      ///< HDR10/CEA861.3 HDR supported
#define ADL_HDR_DOLBYVISION        0x0002      ///< DolbyVision HDR supported
```cpp

这段代码定义了一个名为 ADL_HDR_FREESYNC_HDR 的头文件，定义了 FreeSync HDR 支持的一些标志，并输出了这些标志的值。

具体来说，这段代码定义了两个头文件：ADLSourceContentAttributes 和 ADLDDCInfo2。在 ADLSourceContentAttributes 中，定义了一个名为 iFlags 的标志，它包含一个名为 ADL_SCA_LOCAL_DIMMING_DISABLE 的标志，它的值为 0x0001，表示禁止本地 dimming。在 ADLHDR 中，定义了一个名为 ADL_HDR_FREESYNC_BACKLIGHT_SUPPORT 的标志，它的值为 0x0001，表示支持全局 backlight 控制。在 iFreesyncFlags 中，定义了一个名为 ADL_HDR_FREESYNC_LOCAL_DIMMING 的标志，它的值为 0x0002，表示支持本地 dimming。

这些定义在头文件中定义了一些变量，并在代码中进行了定义和输出，以便在程序中使用这些标志。


```
#define ADL_HDR_FREESYNC_HDR    0x0004      ///< FreeSync HDR supported
/// @}

/// \defgroup define_FreesyncFlags ADLDDCInfo2 Freesync HDR flags
/// @{
/// defines for iFreesyncFlags in ADLDDCInfo2
#define ADL_HDR_FREESYNC_BACKLIGHT_SUPPORT           0x0001      ///< Global backlight control supported
#define ADL_HDR_FREESYNC_LOCAL_DIMMING               0x0002      ///< Local dimming supported
/// @}

/// \defgroup define_source_content_flags ADLSourceContentAttributes flags
/// @{
/// defines for iFlags in ADLSourceContentAttributes
#define ADL_SCA_LOCAL_DIMMING_DISABLE    0x0001      ///< Disable local dimming
/// @}

```cpp

这段代码定义了一个名为 "Deep Bit Depth" 的 ADL_Workstation_DeepBitDepth_Set 函数和名为 "ADL_DEEPBITDEPTH_FORCEOFF" 的 ADL_Workstation_DeepBitDepth_Get 函数。

通过分析代码可以发现，这两个函数都与 Deep Bit depth 状态有关。其中，"ADL_DEEPBITDEPTH_10BPP_AUTO" 表示当 Deep Bit 深度状态为 10 位每像素时，无论显示是否支持 10 位每像素，该函数都将生效；而 "ADL_DEEPBITDEPTH_10BPP_FORCEON" 表示当 Deep Bit 深度状态为 10 位每像素时，如果显示不支持 10 位每像素，该函数将强制将 Deep Bit 深度状态设为 1。


```
/// \defgroup define_dbd_state Deep Bit Depth
/// @{

/// defines for ADL_Workstation_DeepBitDepth_Get and  ADL_Workstation_DeepBitDepth_Set functions
// This value indicates that the deep bit depth state is forced off
#define ADL_DEEPBITDEPTH_FORCEOFF     0
/// This value indicates that the deep bit depth state  is set to auto, the driver will automatically enable the
/// appropriate deep bit depth state depending on what connected display supports.
#define ADL_DEEPBITDEPTH_10BPP_AUTO     1
/// This value indicates that the deep bit depth state  is forced on to 10 bits per pixel, this is regardless if the display
/// supports 10 bpp.
#define ADL_DEEPBITDEPTH_10BPP_FORCEON     2

/// defines for ADLAdapterConfigMemory of ADL_Adapter_ConfigMemory_Get
/// If this bit is set, it indicates that the Deep Bit Depth pixel is set on the display
```cpp

这段代码定义了一个名为ADL_ADAPTER_CONFIGMEMORY的类，下面是这个类的定义：

```
#define ADL_ADAPTER_CONFIGMEMORY_DBD            (1 << 0)
#define ADL_ADAPTER_CONFIGMEMORY_ROTATE            (1 << 1)
#define ADL_ADAPTER_CONFIGMEMORY_STEREO_PASSIVE    (1 << 2)
#define ADL_ADAPTER_CONFIGMEMORY_STEREO_ACTIVE    (1 << 3)
#define ADL_ADAPTER_CONFIGMEMORY_TAREFREEVSYNC    (1 << 4)
#define ADL_ADAPTER_CONFIGMEMORY_ENHANCEDVSYNC    (1 << 4)
#define ADL_ADAPTER_CONFIGMEMORY_TEARFREEVSYNC    (1 << 4)
```cpp

通过这个类，可以设置或查找ADL适配器的内存类型，包括DBD，ROTATE，STEREO_PASSIVE，STEREO_ACTIVE，TAREFREEVSYNC和ENHANCEDVSYNC。这些设置表示不同的功能，如显示是否水平旋转，是否设置被动立体声，是否设置活动立体声，是否设置无撕裂显示器，是否启用优化垂直同步等。


```
#define ADL_ADAPTER_CONFIGMEMORY_DBD            (1 << 0)
/// If this bit is set, it indicates that the display is rotated (90, 180 or 270)
#define ADL_ADAPTER_CONFIGMEMORY_ROTATE            (1 << 1)
/// If this bit is set, it indicates that passive stereo is set on the display
#define ADL_ADAPTER_CONFIGMEMORY_STEREO_PASSIVE    (1 << 2)
/// If this bit is set, it indicates that the active stereo is set on the display
#define ADL_ADAPTER_CONFIGMEMORY_STEREO_ACTIVE    (1 << 3)
/// If this bit is set, it indicates that the tear free vsync is set on the display
#define ADL_ADAPTER_CONFIGMEMORY_ENHANCEDVSYNC    (1 << 4)
#define ADL_ADAPTER_CONFIGMEMORY_TEARFREEVSYNC    (1 << 4)
/// @}

/// \defgroup define_adl_validmemoryrequiredfields Memory Type
/// @{

```cpp

这段代码定义了一个名为 ADLMemoryRequired 的结构体，用于定义可见和不可见的内存类型。其中，#define 表示这是一个别名，为宏定义，而不是普通的变量定义。

ADL_MEMORYREQTYPE_VISIBLE 表示可见内存，当设置为 1 时，表示当前的 ADLMemoryRequired 结构体中的所有成员都是可见内存，也就是说，可以被 GPU 访问的显存。

ADL_MEMORYREQTYPE_INVISIBLE 表示不可见的内存，当设置为 1 时，表示当前的 ADLMemoryRequired 结构体中的所有成员都是不可见的内存，不可以被 GPU 访问的显存。

ADL_MEMORYREQTYPE_GPURESERVEDVISIBLE 表示可以保留一定比例的 GPU 显存用于其他分配，当设置为 1 时，表示当前的 ADLMemoryRequired 结构体中的所有成员都可以从 GPU 获得，但是不能保证每个成员都可以访问。

最后，定义了一系列用于表示内存是否为可见或不可见的常量，以及用于标识是否为可见内存的标志。


```
///  This group defines memory types in ADLMemoryRequired struct \n
/// Indicates that this is the visible memory
#define ADL_MEMORYREQTYPE_VISIBLE                (1 << 0)
/// Indicates that this is the invisible memory.
#define ADL_MEMORYREQTYPE_INVISIBLE                (1 << 1)
/// Indicates that this is amount of visible memory per GPU that should be reserved for all other allocations.
#define ADL_MEMORYREQTYPE_GPURESERVEDVISIBLE    (1 << 2)
/// @}

/// \defgroup define_adapter_tear_free_status
/// Used in ADL_Adapter_TEAR_FREE_Set and ADL_Adapter_TFD_Get functions to indicate the tear free
/// desktop status.
/// @{
/// Tear free desktop is enabled.
#define ADL_ADAPTER_TEAR_FREE_ON                1
```cpp

这段代码定义了一个名为 ADL_ADAPTER_CROSSDISPLAYPLATFORM 的枚举类型，并定义了几个枚举值，分别表示跨显示平台信息。这些枚举值包括：

- 1. CROSSDISPLAY：表示支持跨显示平台信息。
- 0：表示不支持跨显示平台信息。

此外，还有一段注释。


```
/// Tear free desktop can't be enabled due to a lack of graphic adapter memory.
#define ADL_ADAPTER_TEAR_FREE_NOTENOUGHMEM        -1
/// Tear free desktop can't be enabled due to quad buffer stereo being enabled.
#define ADL_ADAPTER_TEAR_FREE_OFF_ERR_QUADBUFFERSTEREO    -2
/// Tear free desktop can't be enabled due to MGPU-SLS being enabled.
#define ADL_ADAPTER_TEAR_FREE_OFF_ERR_MGPUSLD    -3
/// Tear free desktop is disabled.
#define ADL_ADAPTER_TEAR_FREE_OFF                0
/// @}

/// \defgroup define_adapter_crossdisplay_platforminfo
/// Used in ADL_Adapter_CrossDisplayPlatformInfo_Get function to indicate the Crossdisplay platform info.
/// @{
/// CROSSDISPLAY platform.
#define ADL_CROSSDISPLAY_PLATFORM                    (1 << 0)
```cpp

这段代码定义了一个名为“CROSSDISPLAY”的平台，用于填充等候区列表框（lasso station）和等候区列表框（docking station）。它通过使用#define语句定义了两个平台，分别为ADL_CROSSDISPLAY_PLATFORM_LASSO和ADL_CROSSDISPLAY_PLATFORM_DOCKSTATION，同时还定义了一个名为ADL_CROSSDISPLAY_OPTION_NONE的枚举类型，用于表示跨显示选项。

ADL_CROSSDISPLAY_OPTION_NONE表示如果正在运行3D应用程序，则不会执行跨显示选项切换，直接返回ADL_OK_WAIT。否则，它会执行跨显示选项切换，即使3D应用程序正在运行。

ADL_CROSSDISPLAY_OPTION_FORCESWITCH表示强制执行跨显示选项切换，即使3D应用程序正在运行。

这段代码定义了一个用于表示跨显示选项的枚举类型，以及用于设置是否要执行跨显示选项切换的函数ADL_Adapter_CrossdisplayInfoX2_Set。函数的第一个实参是一个整数，表示当前正在运行的3D应用程序的状况，如果没有3D应用程序运行，则返回ADL_OK_WAIT。如果3D应用程序正在运行，但跨显示选项还没有设置，则返回ADL_CROSSDISPLAY_OPTION_FORCESWITCH。


```
/// CROSSDISPLAY platform for Lasso station.
#define ADL_CROSSDISPLAY_PLATFORM_LASSO                (1 << 1)
/// CROSSDISPLAY platform for docking station.
#define ADL_CROSSDISPLAY_PLATFORM_DOCKSTATION        (1 << 2)
/// @}

/// \defgroup define_adapter_crossdisplay_option
/// Used in ADL_Adapter_CrossdisplayInfoX2_Set function to indicate cross display options.
/// @{
/// Checking if 3D application is runnning. If yes, not to do switch, return ADL_OK_WAIT; otherwise do switch.
#define ADL_CROSSDISPLAY_OPTION_NONE            0
/// Force switching without checking for running 3D applications
#define ADL_CROSSDISPLAY_OPTION_FORCESWITCH        (1 << 0)
/// @}

```cpp

这段代码定义了一个名为 `ADL_ADAPTERCONFIGSTATE` 的类，该类包含了一些与适配器相关的状态定义。这些定义用于 `ADL_Adapter_ConfigureState_Get` 函数，用于告诉开发人员适配器的功能和状态。

代码中定义了三个枚举类型：`ADL_ADAPTERCONFIGSTATE_HEADLESS`，`ADL_ADAPTERCONFIGSTATE_REQUISITE_RENDER` 和 `ADL_ADAPTERCONFIGSTATE_ANCILLARY_RENDER`，它们分别表示 adapter 是否配置为可配置的主渲染功能、是否配置为辅助渲染功能，以及是否支持散射聚集。

此外，还定义了一个枚举类型 `ADL_ADAPTERCONFIGSTATE_SCATTERGATHER`，用于指示是否启用散射聚集功能。

最后，定义了一个名为 `ADL_ADAPTERCONFIGSTATE_INIT` 的函数，该函数的作用未知，但可以在后续的代码中被调用以设置适配器的初始状态。


```
/// \defgroup define_adapter_states Adapter Capabilities
/// These defines the capabilities supported by an adapter. It is used by \ref ADL_Adapter_ConfigureState_Get
/// @{
/// Indicates that the adapter is headless (i.e. no displays can be connected to it)
#define ADL_ADAPTERCONFIGSTATE_HEADLESS ( 1 << 2 )
/// Indicates that the adapter is configured to define the main rendering capabilities. For example, adapters
/// in Crossfire(TM) configuration, this bit would only be set on the adapter driving the display(s).
#define ADL_ADAPTERCONFIGSTATE_REQUISITE_RENDER ( 1 << 0 )
/// Indicates that the adapter is configured to be used to unload some of the rendering work for a particular
/// requisite rendering adapter. For eample, for adapters in a Crossfire configuration, this bit would be set
/// on all adapters that are currently not driving the display(s)
#define ADL_ADAPTERCONFIGSTATE_ANCILLARY_RENDER ( 1 << 1 )
/// Indicates that scatter gather feature enabled on the adapter
#define ADL_ADAPTERCONFIGSTATE_SCATTERGATHER ( 1 << 4 )
/// @}

```cpp

这段代码定义了一个名为"define_controllermode_ulModifiers"的组，其中包括了三个定义：

1. "ADL_CONTROLLERMODE_CM_MODIFIER_VIEW_POSITION"定义了一个位掩，表示是否更改了视图位置。
2. "ADL_CONTROLLERMODE_CM_MODIFIER_VIEW_PANLOCK"定义了一个位掩，表示是否更改了视图平锁。
3. "ADL_CONTROLLERMODE_CM_MODIFIER_VIEW_SIZE"定义了一个位掩，表示是否更改了视图大小。

这些定义是由"Mirabilis"组定义的，用于实现Mirabilis功能。


```
/// \defgroup define_controllermode_ulModifiers
/// These defines the detailed actions supported by set viewport. It is used by \ref ADL_Display_ViewPort_Set
/// @{
/// Indicate that the viewport set will change the view position
#define ADL_CONTROLLERMODE_CM_MODIFIER_VIEW_POSITION       0x00000001
/// Indicate that the viewport set will change the view PanLock
#define ADL_CONTROLLERMODE_CM_MODIFIER_VIEW_PANLOCK        0x00000002
/// Indicate that the viewport set will change the view size
#define ADL_CONTROLLERMODE_CM_MODIFIER_VIEW_SIZE           0x00000008
/// @}

/// \defgroup defines for Mirabilis
/// These defines are used for the Mirabilis feature
/// @{
///
```cpp

这段代码定义了一个名为ADLMultiChannelSplitStateFlag的枚举类型，用于表示音频采样率的状态。这个枚举类型有以下几个成员：

* ADLMultiChannelSplit_Unitialized：表示所有音频样本率都已启用。
* ADLMultiChannelSplit_Disabled：表示所有音频样本率都已禁用。
* ADLMultiChannelSplit_Enabled：表示至少有一个音频样本率已启用。
* ADLMultiChannelSplit_SaveProfile：表示已启用且具有配置文件保存功能的音频样本率。

这个枚举类型可以用于选择合适的音频采样率，并可以用于根据需要动态地启用或禁用音频采样率。


```
/// Indicates the maximum number of audio sample rates
#define ADL_MAX_AUDIO_SAMPLE_RATE_COUNT                    16
/// @}

///////////////////////////////////////////////////////////////////////////
// ADLMultiChannelSplitStateFlag Enumeration
///////////////////////////////////////////////////////////////////////////
enum ADLMultiChannelSplitStateFlag
{
    ADLMultiChannelSplit_Unitialized = 0,
    ADLMultiChannelSplit_Disabled    = 1,
    ADLMultiChannelSplit_Enabled     = 2,
    ADLMultiChannelSplit_SaveProfile = 3
};

```cpp

这段代码定义了一个名为 ADLSampleRate 的枚举类型，包含了 8 个枚举值，分别为 ADLSampleRate_32KHz、ADLSampleRate_44P1KHz、ADLSampleRate_48KHz、ADLSampleRate_88P2KHz、ADLSampleRate_96KHz、ADLSampleRate_176P4KHz、ADLSampleRate_192KHz 和 ADLSampleRate_384KHz。这些枚举值可以用于表示数据采样率，其中每个枚举值都有一个唯一的数字标识符。

这个枚举类型可能用于描述数字音频或其他数字信号的采样率，可以帮助程序正确地读取和写入采样率。例如，如果正在开发一个数字音频播放器，可以使用 ADLSampleRate 枚举类型来表示不同的采样率，并为每个采样率指定正确的编码器设置。


```
///////////////////////////////////////////////////////////////////////////
// ADLSampleRate Enumeration
///////////////////////////////////////////////////////////////////////////
enum ADLSampleRate
{
    ADLSampleRate_32KHz =0,
    ADLSampleRate_44P1KHz,
    ADLSampleRate_48KHz,
    ADLSampleRate_88P2KHz,
    ADLSampleRate_96KHz,
    ADLSampleRate_176P4KHz,
    ADLSampleRate_192KHz,
    ADLSampleRate_384KHz, //DP1.2
    ADLSampleRate_768KHz, //DP1.2
    ADLSampleRate_Undefined
};

```cpp

这段代码定义了一个名为 `ADL_OD6_CAPABILITY_GROUP` 的命名组，该命名组定义了 Overdrive 6 驱动程序支持的功能。这组定义了如何控制时钟、内存和图形活动的。通过这个命名组，用户可以知道是否支持特定的功能，例如 SVI2 电压控制、OD6+ 的百分比调整等功能。这个命名组在定义了一些标识符之后，通过 `#define` 预处理指令进行定义，因此在实际代码中，这个命名组名不会被输出。


```
/// \defgroup define_overdrive6_capabilities
/// These defines the capabilities supported by Overdrive 6. It is used by \ref ADL_Overdrive6_Capabilities_Get
// @{
/// Indicate that core (engine) clock can be changed.
#define ADL_OD6_CAPABILITY_SCLK_CUSTOMIZATION               0x00000001
/// Indicate that memory clock can be changed.
#define ADL_OD6_CAPABILITY_MCLK_CUSTOMIZATION               0x00000002
/// Indicate that graphics activity reporting is supported.
#define ADL_OD6_CAPABILITY_GPU_ACTIVITY_MONITOR             0x00000004
/// Indicate that power limit can be customized.
#define ADL_OD6_CAPABILITY_POWER_CONTROL                    0x00000008
/// Indicate that SVI2 Voltage Control is supported.
#define ADL_OD6_CAPABILITY_VOLTAGE_CONTROL                  0x00000010
/// Indicate that OD6+ percentage adjustment is supported.
#define ADL_OD6_CAPABILITY_PERCENT_ADJUSTMENT               0x00000020
```cpp

这段代码定义了一个名为 `ADL_OD6_CAPABILITY_ETHERMAL_LIMIT_UNLOCK` 的符号，其值为 0x00000040，表示是否支持热限开锁。接着定义了一个名为 `ADL_OD6_CAPABILITY_FANSPEED_IN_RPM` 的符号，其值为 0x00000080，表示是否支持显示风扇速度以 RPM 为单位的选项。最后定义了一个名为 `ADL_OD6_SUPPORTEDSTATE_PERFORMANCE` 的符号，其值为 0x00000001，表示在性能模式下是否支持超速驾驶。另外，还有一个名为 `ADL_OD6_SUPPORTEDSTATE_POWER_SAVING` 的符号，其值为 0x00000002，表示是否支持省电驾驶。


```
/// Indicate that Thermal Limit Unlock is supported.
#define ADL_OD6_CAPABILITY_THERMAL_LIMIT_UNLOCK             0x00000040
///Indicate that Fan speed needs to be displayed in RPM
#define ADL_OD6_CAPABILITY_FANSPEED_IN_RPM                    0x00000080
// @}

/// \defgroup define_overdrive6_supported_states
/// These defines the power states supported by Overdrive 6. It is used by \ref ADL_Overdrive6_Capabilities_Get
// @{
/// Indicate that overdrive is supported in the performance state.  This is currently the only state supported.
#define ADL_OD6_SUPPORTEDSTATE_PERFORMANCE                  0x00000001
/// Do not use.  Reserved for future use.
#define ADL_OD6_SUPPORTEDSTATE_POWER_SAVING                 0x00000002
// @}

```cpp

这是一个C语言定义的名为“define_overdrive6_getstateinfo”的定义组，用于定义了从ADL_Overdrive6状态机中获取性能状态的相关信息。

具体来说，这段代码定义了以下7个宏：

1. ADL_OD6_GETSTATEINFO_DEFAULT_PERFORMANCE：表示性能状态的默认时钟，由0x00000001表示。如果没有给出具体的值，那么时钟不会生效。
2. ADL_OD6_GETSTATEINFO_DEFAULT_POWER_SAVING：表示在性能状态下可以省下的能量消耗，由0x00000002表示。同样，如果没有给出具体的值，那么该宏也不会生效。
3. ADL_OD6_GETSTATEINFO_CURRENT：表示当前状态下的时钟，与ADL_OD6_GETSTATEINFO_CUSTOM_PERFORMANCE相同，仅在支持性能状态时生效。
4. ADL_OD6_GETSTATEINFO_CUSTOM_PERFORMANCE：表示通过Overdrive 6获取的时钟，仅在支持性能状态时生效，如果没有给出具体的值，那么该宏的值与ADL_OD6_GETSTATEINFO_DEFAULT_PERFORMANCE相同。
5. ADL_OD6_GETSTATEINFO_CUSTOM_POWER_SAVING：表示通过Overdrive 6获取的与ADL_OD6_GETSTATEINFO_DEFAULT_POWER_SAVING相同的值，仅在支持性能状态时生效。

由于该代码定义了一些宏，因此在实际使用中，只需要关注定义组中给出的这些宏即可，而不需要关心其具体实现。


```
/// \defgroup define_overdrive6_getstateinfo
/// These defines the power states to get information about. It is used by \ref ADL_Overdrive6_StateInfo_Get
// @{
/// Get default clocks for the performance state.
#define ADL_OD6_GETSTATEINFO_DEFAULT_PERFORMANCE            0x00000001
/// Do not use.  Reserved for future use.
#define ADL_OD6_GETSTATEINFO_DEFAULT_POWER_SAVING           0x00000002
/// Get clocks for current state.  Currently this is the same as \ref ADL_OD6_GETSTATEINFO_CUSTOM_PERFORMANCE
/// since only performance state is supported.
#define ADL_OD6_GETSTATEINFO_CURRENT                        0x00000003
/// Get the modified clocks (if any) for the performance state.  If clocks were not modified
/// through Overdrive 6, then this will return the same clocks as \ref ADL_OD6_GETSTATEINFO_DEFAULT_PERFORMANCE.
#define ADL_OD6_GETSTATEINFO_CUSTOM_PERFORMANCE             0x00000004
/// Do not use.  Reserved for future use.
#define ADL_OD6_GETSTATEINFO_CUSTOM_POWER_SAVING            0x00000005
```cpp

这段代码定义了一个名为 `ADL_OD6_STATE_PERFORMANCE` 的宏，它的值为 0x00000001。同时，定义了一个名为 `ADL_OD6_SETSTATE_PERFORMANCE` 的宏，其值为 0x00000002。这两个宏都被定义为 `define_overdrive6_getstate` 和 `define_overdrive6_getmaxclockadjust` 函数的成员。

根据定义，`ADL_OD6_STATE_PERFORMANCE` 表示性能状态，而 `ADL_OD6_SETSTATE_PERFORMANCE` 表示设置自定义时钟的功耗状态。这两个宏分别对应了 `ADL_Overdrive6_StateEx_Get` 和 `ADL_Overdrive6_MaxClockAdjust_Get` 函数的输入参数。


```
// @}

/// \defgroup define_overdrive6_getstate and define_overdrive6_getmaxclockadjust
/// These defines the power states to get information about. It is used by \ref ADL_Overdrive6_StateEx_Get and \ref ADL_Overdrive6_MaxClockAdjust_Get
// @{
/// Get default clocks for the performance state.  Only performance state is currently supported.
#define ADL_OD6_STATE_PERFORMANCE            0x00000001
// @}

/// \defgroup define_overdrive6_setstate
/// These define which power state to set customized clocks on. It is used by \ref ADL_Overdrive6_State_Set
// @{
/// Set customized clocks for the performance state.
#define ADL_OD6_SETSTATE_PERFORMANCE                        0x00000001
/// Do not use.  Reserved for future use.
```cpp

这段代码定义了一个名为ADL_OD6_SETSTATE_POWER_SAVING的常量，其值为0x00000002。这个常量用于定义ADL_Overdrive6_ThermalController_Caps中的Capabilities结构体。

Capabilities结构体定义了GPU thermal controller的功能，包括是否支持GPU fan speed control、是否支持读取 fan speed percentage、是否支持设置 fan speed percentage为指定百分比、是否支持读取 fan speed RPM等。通过定义这些常量，可以确保在代码中使用这些功能时，不同版本的API都能正常工作。


```
#define ADL_OD6_SETSTATE_POWER_SAVING                       0x00000002
// @}

/// \defgroup define_overdrive6_thermalcontroller_caps
/// These defines the capabilities of the GPU thermal controller. It is used by \ref ADL_Overdrive6_ThermalController_Caps
// @{
/// GPU thermal controller is supported.
#define ADL_OD6_TCCAPS_THERMAL_CONTROLLER                   0x00000001
/// GPU fan speed control is supported.
#define ADL_OD6_TCCAPS_FANSPEED_CONTROL                     0x00000002
/// Fan speed percentage can be read.
#define ADL_OD6_TCCAPS_FANSPEED_PERCENT_READ                0x00000100
/// Fan speed can be set by specifying a percentage value.
#define ADL_OD6_TCCAPS_FANSPEED_PERCENT_WRITE               0x00000200
/// Fan speed RPM (revolutions-per-minute) can be read.
```cpp

这段代码定义了一个名为ADL_OD6_TCCAPS_FANSPEED_RPM的宏，包含两个子定义：

1. 0x00000400中的ADL_OD6_TCCAPS_FANSPEED_RPM_READ，定义了一个以RPM为单位的浮点数，表示风扇的转速。这个值可以被用来设置风扇的转速。

2. 0x00000800中的ADL_OD6_TCCAPS_FANSPEED_RPM_WRITE，定义了一个以百分比为单位的浮点数，表示风扇的转速。这个值可以被用来设置风扇的转速，并且可以通过用户输入来修改。

此外，还有一组定义了ADL_OD6_FANSPEED_TYPE_PERCENT和ADL_OD6_FANSPEED_USER_DEFINED，用于报告风扇的转速类型和是否可以定制风扇转速。


```
#define ADL_OD6_TCCAPS_FANSPEED_RPM_READ                    0x00000400
/// Fan speed can be set by specifying an RPM value.
#define ADL_OD6_TCCAPS_FANSPEED_RPM_WRITE                   0x00000800
// @}

/// \defgroup define_overdrive6_fanspeed_type
/// These defines the fan speed type being reported. It is used by \ref ADL_Overdrive6_FanSpeed_Get
// @{
/// Fan speed reported in percentage.
#define ADL_OD6_FANSPEED_TYPE_PERCENT                       0x00000001
/// Fan speed reported in RPM.
#define ADL_OD6_FANSPEED_TYPE_RPM                           0x00000002
/// Fan speed has been customized by the user, and fan is not running in automatic mode.
#define ADL_OD6_FANSPEED_USER_DEFINED                       0x00000100
// @}

```cpp

这段代码定义了一个名为 ADLODNControlType 的枚举类型，共包括四个成员：Current、Default、Auto 和 Manual。这些成员用于定义 OverdriveN 驱动程序中的 EventCounter 类型，以便在实现对 OverdriveN 设备控制时进行使用。

具体来说，这段代码定义了一个名为 ADLODNControlType 的枚举类型，它用于定义 OverdriveN 设备控制过程中的不同控制类型。在 OverdriveN 的用户手册中，这个枚举类型被描述为“用于定义 ADL 驱动程序中 OverdriveN 设备控制过程中的控制类型”。

ADLODNControlType 枚举类型包括四个成员：Current、Default、Auto 和 Manual。其中，Current 表示“当前”，Default 表示“默认”，Auto 表示“自动”，Manual 表示“手动”。这些成员可以用于实现对 OverdriveN 设备的控制，以及对设备状态的监控和报警。


```
/// \defgroup define_overdrive_EventCounter_type
/// These defines the EventCounter type being reported. It is used by \ref ADL2_OverdriveN_CountOfEvents_Get ,can be used on older OD version supported ASICs also.
// @{
#define ADL_ODN_EVENTCOUNTER_THERMAL        0
#define ADL_ODN_EVENTCOUNTER_VPURECOVERY    1
// @}

///////////////////////////////////////////////////////////////////////////
// ADLODNControlType Enumeration
///////////////////////////////////////////////////////////////////////////
enum ADLODNControlType
{
    ODNControlType_Current = 0,
    ODNControlType_Default,
    ODNControlType_Auto,
    ODNControlType_Manual
};

```cpp



This is a bit field that represents a control over various features of an optical distribution network (ODN). It is used to enable or disable advanced features of the ODN, such as load balancing, power balancing, andPerformanceTuning.

The bit field is divided into several sections, each of which corresponds to a specific feature. The first section, `ADL_ODN_SCLK_DPM`, controls the enablement of point-to-point (SCLK) DPM (data power margin) for the君子 (Gentleman) transmission mode. The second section, `ADL_ODN_MCLK_DPM`, controls the enablement of point-to-point (MCLK) DPM for the君子传输 mode.

The third section, `ADL_ODN_SCLK_VDD`, controls the enablement of binary variable node (BVN) transmission for the SCLK (single-carrier) mode. The fourth section, `ADL_ODN_MCLK_VDD`, controls the enablement of BVN transmission for the MCLK (multi-carrier) mode.

The fifth section, `ADL_ODN_FAN_SPEED_MIN`, sets the minimum fan speed of the air-cooled adapters for the network devices. The sixth section, `ADL_ODN_FAN_SPEED_TARGET`, sets the target fan speed of the air-cooled adapters.

The seventh section, `ADL_ODN_ACOUSTIC_LIMIT_SCLK`, limits the amount of unstable latency that the ODN can tolerate. The eighth section, `ADL_ODN_TEMPERATURE_FAN_MAX`, sets the maximum temperature that the air-cooled adapters can dissipate. The ninth section, `ADL_ODN_TEMPERATURE_SYSTEM`, sets the temperature that the system temperature can accept. The tenth section, `ADL_ODN_POWER_LIMIT`, sets the maximum power consumption that the ODN can allow.

The eleventh section, `ADL_ODN_SCLK_AUTO_LIMIT`, enables the auto-limiting of SCLK output power. The twelfth section, `ADL_ODN_MCLK_AUTO_LIMIT`, enables the auto-limiting of MCLK output power. The thirteenth section, `ADL_ODN_SCLK_DPM_MASK_ENABLE`, enables the enablement of SCLK DPM (data power margin) for the君子传输 mode.

The fourteenth section, `ADL_ODN_MCLK_DPM_MASK_ENABLE`, enables the enablement of MCLK DPM for the君子传输 mode. The fifteenth section, `ADL_ODN_MCLK_UNDERCLOCK_ENABLE`, enables the under-clocking of MCLK for the君子传输 mode.

The sixteenth section, `ADL_ODN_SCLK_DPM_THROTTLE_NOTIFY`, sets the threshold for the SCLK DPM over-limit notification. The seventeenth section, `ADL_ODN_POWER_UTILIZATION`, enables the power utilization monitoring. The eighteenth section, `ADL_ODN_PERF_TUNING_SLIDER`, sets the performance tuning slider. The nineteenth section, `ADL_ODN_REMOVE_WATTMAN_PAGE`, enables the removal of Wattman pages.

The external link, `ADL_ODN_SPECIFICATION_CITATION_NUMBER`, is a reference to the referenced specifications.


```
enum ADLODNDPMMaskType
{
     ADL_ODN_DPM_CLOCK               = 1 << 0,
     ADL_ODN_DPM_VDDC                = 1 << 1,
     ADL_ODN_DPM_MASK                = 1 << 2,
};

//ODN features Bits for ADLODNCapabilitiesX2
enum ADLODNFeatureControl
{
     ADL_ODN_SCLK_DPM                = 1 << 0,
     ADL_ODN_MCLK_DPM                = 1 << 1,
     ADL_ODN_SCLK_VDD                = 1 << 2,
     ADL_ODN_MCLK_VDD                = 1 << 3,
     ADL_ODN_FAN_SPEED_MIN           = 1 << 4,
     ADL_ODN_FAN_SPEED_TARGET        = 1 << 5,
     ADL_ODN_ACOUSTIC_LIMIT_SCLK     = 1 << 6,
     ADL_ODN_TEMPERATURE_FAN_MAX     = 1 << 7,
     ADL_ODN_TEMPERATURE_SYSTEM      = 1 << 8,
     ADL_ODN_POWER_LIMIT             = 1 << 9,
     ADL_ODN_SCLK_AUTO_LIMIT             = 1 << 10,
     ADL_ODN_MCLK_AUTO_LIMIT             = 1 << 11,
     ADL_ODN_SCLK_DPM_MASK_ENABLE        = 1 << 12,
     ADL_ODN_MCLK_DPM_MASK_ENABLE        = 1 << 13,
     ADL_ODN_MCLK_UNDERCLOCK_ENABLE      = 1 << 14,
     ADL_ODN_SCLK_DPM_THROTTLE_NOTIFY    = 1 << 15,
     ADL_ODN_POWER_UTILIZATION           = 1 << 16,
     ADL_ODN_PERF_TUNING_SLIDER          = 1 << 17,
     ADL_ODN_REMOVE_WATTMAN_PAGE         = 1 << 31 // Internal Only
};

```cpp

根据提供的代码，该示例似乎是针对一种名为ADLODN的设备，其支持自动分配OC内存。在代码中，设置名为ADLODN_ODN_EXT_FEATURE_FAN_CURVE的扩展功能ID，以启用基于 fan curve的自动分配内存功能。然而，需要注意的是，此功能仅在特定设备上可用，并且可能会受到支持的最高内存带宽限制的限制。


```
//If any new feature is added, PPLIB only needs to add ext feature ID and Item ID(Seeting ID). These IDs should match the drive defined in CWDDEPM.h
enum ADLODNExtFeatureControl
{
	ADL_ODN_EXT_FEATURE_MEMORY_TIMING_TUNE = 1 << 0,
	ADL_ODN_EXT_FEATURE_FAN_ZERO_RPM_CONTROL = 1 << 1,
	ADL_ODN_EXT_FEATURE_AUTO_UV_ENGINE = 1 << 2,   //Auto under voltage
	ADL_ODN_EXT_FEATURE_AUTO_OC_ENGINE = 1 << 3,   //Auto OC Enine
	ADL_ODN_EXT_FEATURE_AUTO_OC_MEMORY = 1 << 4,   //Auto OC memory
	ADL_ODN_EXT_FEATURE_FAN_CURVE = 1 << 5    //Fan curve

};

//If any new feature is added, PPLIB only needs to add ext feature ID and Item ID(Seeting ID).These IDs should match the drive defined in CWDDEPM.h
enum ADLODNExtSettingId
{
	ADL_ODN_PARAMETER_AC_TIMING = 0,
	ADL_ODN_PARAMETER_FAN_ZERO_RPM_CONTROL,
	ADL_ODN_PARAMETER_AUTO_UV_ENGINE,
	ADL_ODN_PARAMETER_AUTO_OC_ENGINE,
	ADL_ODN_PARAMETER_AUTO_OC_MEMORY,
	ADL_ODN_PARAMETER_FAN_CURVE_TEMPERATURE_1,
	ADL_ODN_PARAMETER_FAN_CURVE_SPEED_1,
	ADL_ODN_PARAMETER_FAN_CURVE_TEMPERATURE_2,
	ADL_ODN_PARAMETER_FAN_CURVE_SPEED_2,
	ADL_ODN_PARAMETER_FAN_CURVE_TEMPERATURE_3,
	ADL_ODN_PARAMETER_FAN_CURVE_SPEED_3,
	ADL_ODN_PARAMETER_FAN_CURVE_TEMPERATURE_4,
	ADL_ODN_PARAMETER_FAN_CURVE_SPEED_4,
	ADL_ODN_PARAMETER_FAN_CURVE_TEMPERATURE_5,
	ADL_ODN_PARAMETER_FAN_CURVE_SPEED_5,
    ADL_ODN_POWERGAUGE,
	ODN_COUNT

} ;

```cpp

这段代码定义了一个名为 ADLOD8FeatureControl 的枚举类型，用于说明 ADI 芯片的功能。该枚举类型包括了以下 16 个标志位：

- 1. ADL_OD8_GFXCLK_LIMITS：控制显卡时钟频率的极限值。
- 2. ADL_OD8_GFXCLK_CURVE：控制显卡时钟频率的曲线。
- 3. ADL_OD8_UCLK_MAX：控制 CPU 输出时钟频率的最大值。
- 4. ADL_OD8_POWER_LIMIT：控制电源输出的电压限制。
- 5. ADL_OD8_ACOUSTIC_LIMIT_SCLK：限制 SCLK 信号的时钟频率。
- 6. ADL_OD8_FAN_SPEED_MIN：控制风扇的最小转速。
- 7. ADL_OD8_TEMPERATURE_FAN：控制风扇的目标温度。
- 8. ADL_OD8_TEMPERATURE_SYSTEM：控制操作系统的最高温度。
- 9. ADL_OD8_MEMORY_TIMING_TUNE：控制内存的时钟频率调度。
- 10. ADL_OD8_FAN_ZERO_RPM_CONTROL：控制风扇的零转矩控制。
- 11. ADL_OD8_AUTO_UV_ENGINE：自动调整 UV 引擎的时钟频率。
- 12. ADL_OD8_AUTO_OC_ENGINE：自动调整时钟频率的引擎。
- 13. ADL_OD8_AUTO_OC_MEMORY：自动调整时钟频率的内存。
- 14. ADL_OD8_FAN_CURVE：控制风扇的曲线。
- 15. ADL_OD8_WS_AUTO_FAN_ACOUSTIC_LIMIT：设置自动风扇控制器的工作模式。
- 16. ADL_OD8_POWER_GAUGE：显示电源的功率限制。


```
//OD8 Capability features bits
enum ADLOD8FeatureControl
{
    ADL_OD8_GFXCLK_LIMITS = 1 << 0,
    ADL_OD8_GFXCLK_CURVE = 1 << 1,
    ADL_OD8_UCLK_MAX = 1 << 2,
    ADL_OD8_POWER_LIMIT = 1 << 3,
    ADL_OD8_ACOUSTIC_LIMIT_SCLK = 1 << 4,   //FanMaximumRpm
    ADL_OD8_FAN_SPEED_MIN = 1 << 5,   //FanMinimumPwm
    ADL_OD8_TEMPERATURE_FAN = 1 << 6,   //FanTargetTemperature
    ADL_OD8_TEMPERATURE_SYSTEM = 1 << 7,    //MaxOpTemp
    ADL_OD8_MEMORY_TIMING_TUNE = 1 << 8,
    ADL_OD8_FAN_ZERO_RPM_CONTROL = 1 << 9 ,
	ADL_OD8_AUTO_UV_ENGINE = 1 << 10,  //Auto under voltage
	ADL_OD8_AUTO_OC_ENGINE = 1 << 11,  //Auto overclock engine
	ADL_OD8_AUTO_OC_MEMORY = 1 << 12,  //Auto overclock memory
	ADL_OD8_FAN_CURVE = 1 << 13,   //Fan curve
	ADL_OD8_WS_AUTO_FAN_ACOUSTIC_LIMIT = 1 << 14, //Workstation Manual Fan controller
    ADL_OD8_POWER_GAUGE = 1 << 15 //Power Gauge
};


```cpp

ADLOD8SettingId can be used to identify settings for the Adobe Cloud也就即时义云装载设置。这些设置包括音频设备设置，如LCLK，OD8和UCLK的时钟电压百分比，频率和振荡。同时也可以设置风扇驱动器的最小速度和最大速度以及自动调节，以最小化处理器温度并提高游戏的性能。此外，还可以设置内存限制和自动电压调节以获得更好的电源利用率和稳定性。还可以设置振荡和速度以获得最佳的音频质量。


```
typedef enum _ADLOD8SettingId
{
	OD8_GFXCLK_FMAX = 0,
	OD8_GFXCLK_FMIN,
	OD8_GFXCLK_FREQ1,
	OD8_GFXCLK_VOLTAGE1,
	OD8_GFXCLK_FREQ2,
	OD8_GFXCLK_VOLTAGE2,
	OD8_GFXCLK_FREQ3,
	OD8_GFXCLK_VOLTAGE3,
	OD8_UCLK_FMAX,
	OD8_POWER_PERCENTAGE,
	OD8_FAN_MIN_SPEED,
	OD8_FAN_ACOUSTIC_LIMIT,
	OD8_FAN_TARGET_TEMP,
	OD8_OPERATING_TEMP_MAX,
	OD8_AC_TIMING,
	OD8_FAN_ZERORPM_CONTROL,
	OD8_AUTO_UV_ENGINE_CONTROL,
	OD8_AUTO_OC_ENGINE_CONTROL,
	OD8_AUTO_OC_MEMORY_CONTROL,
	OD8_FAN_CURVE_TEMPERATURE_1,
	OD8_FAN_CURVE_SPEED_1,
	OD8_FAN_CURVE_TEMPERATURE_2,
	OD8_FAN_CURVE_SPEED_2,
	OD8_FAN_CURVE_TEMPERATURE_3,
	OD8_FAN_CURVE_SPEED_3,
	OD8_FAN_CURVE_TEMPERATURE_4,
	OD8_FAN_CURVE_SPEED_4,
	OD8_FAN_CURVE_TEMPERATURE_5,
	OD8_FAN_CURVE_SPEED_5,
	OD8_WS_FAN_AUTO_FAN_ACOUSTIC_LIMIT,
    OD8_POWER_GAUGE, //Starting from this is new features with new capabilities and new interface for limits.
	OD8_COUNT
} ADLOD8SettingId;


```cpp

`ADLSensorType` is a variable that is used to store information about a particular sensor. The ADLSensorType field is defined as follows:

```c
ADLSensorType:
   LogType         = 0,
   FanExc         = 1,
   BlkExc         = 2,
   Pmlog         = 3,
   SocPower        = 4,
   SocCurrent      = 5,
   FanRpm         = 6,
   FanPercentage    = 7,
   SocVoltage     = 8,
   SocPowerConsumed = 9,
   SmartPowershift = 10,
   MaxSensors      = 11
```cpp

It contains a combination of 11 fields that provide information about a particular sensor.


```
//Define Performance Metrics Log max sensors number
#define ADL_PMLOG_MAX_SENSORS  256

typedef enum _ADLSensorType
{
	SENSOR_MAXTYPES = 0,
	PMLOG_CLK_GFXCLK = 1,
	PMLOG_CLK_MEMCLK = 2,
	PMLOG_CLK_SOCCLK = 3,
	PMLOG_CLK_UVDCLK1 = 4,
	PMLOG_CLK_UVDCLK2 = 5,
	PMLOG_CLK_VCECLK = 6,
	PMLOG_CLK_VCNCLK = 7,
	PMLOG_TEMPERATURE_EDGE = 8,
	PMLOG_TEMPERATURE_MEM = 9,
	PMLOG_TEMPERATURE_VRVDDC = 10,
	PMLOG_TEMPERATURE_VRMVDD = 11,
	PMLOG_TEMPERATURE_LIQUID = 12,
	PMLOG_TEMPERATURE_PLX = 13,
	PMLOG_FAN_RPM = 14,
	PMLOG_FAN_PERCENTAGE = 15,
	PMLOG_SOC_VOLTAGE = 16,
	PMLOG_SOC_POWER = 17,
	PMLOG_SOC_CURRENT = 18,
	PMLOG_INFO_ACTIVITY_GFX = 19,
	PMLOG_INFO_ACTIVITY_MEM = 20,
	PMLOG_GFX_VOLTAGE = 21,
	PMLOG_MEM_VOLTAGE = 22,
	PMLOG_ASIC_POWER = 23,
	PMLOG_TEMPERATURE_VRSOC = 24,
	PMLOG_TEMPERATURE_VRMVDD0 = 25,
	PMLOG_TEMPERATURE_VRMVDD1 = 26,
	PMLOG_TEMPERATURE_HOTSPOT = 27,
        PMLOG_TEMPERATURE_GFX = 28,
        PMLOG_TEMPERATURE_SOC = 29,
        PMLOG_GFX_POWER = 30,
        PMLOG_GFX_CURRENT = 31,
        PMLOG_TEMPERATURE_CPU = 32,
        PMLOG_CPU_POWER = 33,
        PMLOG_CLK_CPUCLK = 34,
        PMLOG_THROTTLER_STATUS = 35,
        PMLOG_CLK_VCN1CLK1 = 36,
        PMLOG_CLK_VCN1CLK2 = 37,
        PMLOG_SMART_POWERSHIFT_CPU = 38,
        PMLOG_SMART_POWERSHIFT_DGPU = 39,
	PMLOG_MAX_SENSORS_REAL
} ADLSensorType;


```cpp

这是一道比较典型的权值匹配题目，需要将输入的输入权值映射到输出的一组权值中。我们可以将输入权值和输出的一组权值存储在一个二维数组中，然后使用匹配算法来找到匹配关系。

具体实现可以参考下面的代码：

```
#include <stdio.h>
#include <stdlib.h>

#define MAX_权值 10000000
#define MAX_输入 10000000

int main()
{
   int i, j;
   int input[MAX_输入], output[MAX_权值];

   while(i < MAX_输入)
   {
       input[i] = i;
       output[i] = i;
       i++;
   }

   while(i < MAX_输入)
   {
       i--;
       for(j = 0; j < MAX_权值； j++)
       {
           if(input[i] == j)
           {
               output[i] = j;
               break;
           }
       }
       i++;
   }

   printf("输入权值： \n");
   for(i = 0; i < MAX_input; i++)
   {
       printf("%d ", input[i]);
   }
   printf("\n");

   printf("输出权值： \n");
   for(i = 0; i < MAX_权值； i++)
   {
       printf("%d ", output[i]);
   }
   printf("\n");

   return 0;
}
```cpp

程序首先定义了一个常量 MAX_权值 和 MAX_输入，用来表示输入和输出的最大值。接着定义了一个 ADL_PMLOG_SENSORS 数组，用于存储输入输出的一组权值。

在主程序中，首先将输入的权值初始化为输入的端口号，然后将输出的一组权值也初始化为输出端口号。接下来就可以像数据匹配一样，将输入的权值和输出的一组权值进行匹配，然后输出匹配结果。


```
//Throttle Status
typedef enum _ADL_THROTTLE_NOTIFICATION
{
	ADL_PMLOG_THROTTLE_POWER = 1 << 0,
	ADL_PMLOG_THROTTLE_THERMAL = 1 << 1,
	ADL_PMLOG_THROTTLE_CURRENT = 1 << 2,
} ADL_THROTTLE_NOTIFICATION;

typedef enum _ADL_PMLOG_SENSORS
{
	ADL_SENSOR_MAXTYPES = 0,
	ADL_PMLOG_CLK_GFXCLK = 1,
	ADL_PMLOG_CLK_MEMCLK = 2,
	ADL_PMLOG_CLK_SOCCLK = 3,
	ADL_PMLOG_CLK_UVDCLK1 = 4,
	ADL_PMLOG_CLK_UVDCLK2 = 5,
	ADL_PMLOG_CLK_VCECLK = 6,
	ADL_PMLOG_CLK_VCNCLK = 7,
	ADL_PMLOG_TEMPERATURE_EDGE = 8,
	ADL_PMLOG_TEMPERATURE_MEM = 9,
	ADL_PMLOG_TEMPERATURE_VRVDDC = 10,
	ADL_PMLOG_TEMPERATURE_VRMVDD = 11,
	ADL_PMLOG_TEMPERATURE_LIQUID = 12,
	ADL_PMLOG_TEMPERATURE_PLX = 13,
	ADL_PMLOG_FAN_RPM = 14,
	ADL_PMLOG_FAN_PERCENTAGE = 15,
	ADL_PMLOG_SOC_VOLTAGE = 16,
	ADL_PMLOG_SOC_POWER = 17,
	ADL_PMLOG_SOC_CURRENT = 18,
	ADL_PMLOG_INFO_ACTIVITY_GFX = 19,
	ADL_PMLOG_INFO_ACTIVITY_MEM = 20,
	ADL_PMLOG_GFX_VOLTAGE = 21,
	ADL_PMLOG_MEM_VOLTAGE = 22,
	ADL_PMLOG_ASIC_POWER = 23,
	ADL_PMLOG_TEMPERATURE_VRSOC = 24,
	ADL_PMLOG_TEMPERATURE_VRMVDD0 = 25,
	ADL_PMLOG_TEMPERATURE_VRMVDD1 = 26,
	ADL_PMLOG_TEMPERATURE_HOTSPOT = 27,
	ADL_PMLOG_TEMPERATURE_GFX = 28,
	ADL_PMLOG_TEMPERATURE_SOC = 29,
	ADL_PMLOG_GFX_POWER = 30,
	ADL_PMLOG_GFX_CURRENT = 31,
	ADL_PMLOG_TEMPERATURE_CPU = 32,
	ADL_PMLOG_CPU_POWER = 33,
	ADL_PMLOG_CLK_CPUCLK = 34,
	ADL_PMLOG_THROTTLER_STATUS = 35,
    ADL_PMLOG_CLK_VCN1CLK1 = 36,
    ADL_PMLOG_CLK_VCN1CLK2 = 37,
    ADL_PMLOG_SMART_POWERSHIFT_CPU	= 38,
    ADL_PMLOG_SMART_POWERSHIFT_DGPU = 39
} ADL_PMLOG_SENSORS;

```cpp

这段代码定义了一个名为“define_ecc_mode_states”的组，其中包括了三种ECC（Error Correction Code）模式，分别定义为0、2和3。这些模式用于控制ADL（Advanced Data Encoding）功能的工作状态。

接着定义了一个名为“define_board_layout_flags”的组，其中包括了两个定义，分别表示ADL板布局中插槽的有效数量。

最后，在定义了一些常量和宏定义后，未在当前函数中使用，但可能需要在其他源文件中使用。


```
/// \defgroup define_ecc_mode_states
/// These defines the ECC(Error Correction Code) state. It is used by \ref ADL_Workstation_ECC_Get,ADL_Workstation_ECC_Set
// @{
/// Error Correction is OFF.
#define ECC_MODE_OFF 0
/// Error Correction is ECCV2.
#define ECC_MODE_ON 2
/// Error Correction is HBM.
#define ECC_MODE_HBM 3
// @}

/// \defgroup define_board_layout_flags
/// These defines are the board layout flags state which indicates what are the valid properties of \ref ADLBoardLayoutInfo . It is used by \ref ADL_Adapter_BoardLayout_Get
// @{
/// Indicates the number of slots is valid.
```cpp

这段代码定义了一系列常量，用于标识板卡上的可用插槽数量、插槽尺寸以及连接器偏移。通过这些常量，可以保证板卡的设计是可行的，并且在编程过程中可以避免对这些硬件参数的误用或不当设置。

具体来说，定义的第一个常量 ADL_BLAYOUT_VALID_NUMBER_OF_SLOTS 表示板卡上可用的插槽数量，它由两个字节的长度和宽度组成，因此每个插槽可以被视为一个长度为2字节，宽度为1字节的二进制向量。第二个常量 ADL_BLAYOUT_VALID_SLOT_SIZES 表示每个插槽的最大尺寸，它由两个字节的长度和宽度组成，因此每个插槽的最大尺寸可以被视为一个长度为2字节，宽度为1字节的二进制向量。第三个常量 ADL_BLAYOUT_VALID_CONNECTOR_OFFSETS 表示每个连接器偏移的最大值，它由两个字节组成，表示为4字节。第四个常量 ADL_BLAYOUT_VALID_CONNECTOR_LENGTHS 表示每个连接器长度的最大值，它由两个字节组成，表示为8字节。

通过定义这些常量，开发人员可以使用它们来确保板卡的设计符合要求，并且在编程过程中可以避免对硬件参数的误用或不当设置。例如，在编写板卡驱动程序时，可以使用这些常量来判断板卡上可用的插槽数量是否符合预期，或者在使用板卡进行通信时，可以使用这些常量来确保连接器偏移不会导致连接问题。


```
#define ADL_BLAYOUT_VALID_NUMBER_OF_SLOTS 0x1
/// Indicates the slot sizes are valid. Size of the slot consists of the length and width.
#define ADL_BLAYOUT_VALID_SLOT_SIZES 0x2
/// Indicates the connector offsets are valid.
#define ADL_BLAYOUT_VALID_CONNECTOR_OFFSETS 0x4
/// Indicates the connector lengths is valid.
#define ADL_BLAYOUT_VALID_CONNECTOR_LENGTHS 0x8
// @}

/// \defgroup define_max_constants
/// These defines are the maximum value constants.
// @{
/// Indicates the Maximum supported slots on board.
#define ADL_ADAPTER_MAX_SLOTS 4
/// Indicates the Maximum supported connectors on slot.
```cpp

这段代码定义了一系列关于连接器最大连接数量、最大相对地址链接数量、最大EDID数据块大小、错误记录最大数量以及最大电源状态数量等技术常量的定义。它们用于指示适配器最大支持连接类型，并用于根据这些连接类型对ADL_Adapter_SupportedConnections_Get函数进行检查。以下是这些定义的详细解释：

1. `#define ADL_ADAPTER_MAX_CONNECTORS 10`：表示适配器最大支持10个连接器。
2. `/// Indicates the Maximum supported properties of connection`：这个注释说明了这个定义的含义。
3. `#define ADL_MAX_CONNECTION_TYPES 32`：表示适配器最大支持32种连接类型。
4. `/// Indicates the Maximum relative address link count.`：这个注释说明了这个定义的含义。
5. `#define ADL_MAX_RELATIVE_ADDRESS_LINK_COUNT 15`：表示适配器最大支持15个相对地址链接。
6. `/// Indicates the Maximum size of EDID data block size`：这个注释说明了这个定义的含义。
7. `/// Indicates the Maximum count of Error Records.`：这个注释说明了这个定义的含义。
8. `ADL_MAX_ERROR_RECORDS_COUNT  256`：表示适配器最大支持256个错误记录。
9. `/// Indicates the maximum number of power states supported`：这个注释说明了这个定义的含义。
10. `#define ADL_MAX_POWER_POLICY    6`：表示适配器最大支持6个电源状态。

总的来说，这段代码定义了一系列关于连接器最大连接数量、最大相对地址链接数量、最大EDID数据块大小、错误记录最大数量以及最大电源状态数量等技术常量的定义。这些定义用于指示适配器最大支持连接类型，并用于根据这些连接类型对ADL_Adapter_SupportedConnections_Get函数进行检查。


```
#define ADL_ADAPTER_MAX_CONNECTORS 10
/// Indicates the Maximum supported properties of connection
#define ADL_MAX_CONNECTION_TYPES 32
/// Indicates the Maximum relative address link count.
#define ADL_MAX_RELATIVE_ADDRESS_LINK_COUNT 15
/// Indicates the Maximum size of EDID data block size
#define ADL_MAX_DISPLAY_EDID_DATA_SIZE 1024
/// Indicates the Maximum count of Error Records.
#define ADL_MAX_ERROR_RECORDS_COUNT  256
/// Indicates the maximum number of power states supported
#define ADL_MAX_POWER_POLICY    6
// @}

/// \defgroup define_connection_types
/// These defines are the connection types constants which indicates  what are the valid connection type of given connector. It is used by \ref ADL_Adapter_SupportedConnections_Get
```cpp

这是一个头文件，定义了一系列标识各种连接类型的常量。其中包括：

* `ADL_CONNECTION_TYPE_VGA`：表示VGA连接类型的有效。
* `ADL_CONNECTION_TYPE_DVI`：表示DVI连接类型的有效。
* `ADL_CONNECTION_TYPE_DVI_SL`：表示DVI-SL连接类型的有效。
* `ADL_CONNECTION_TYPE_HDMI`：表示HDMI连接类型的有效。
* `ADL_CONNECTION_TYPE_DISPLAY_PORT`：表示显示端口连接类型的有效。
* `ADL_CONNECTION_TYPE_ACTIVE_DONGLE_DP_DVI_SL`：表示主动Dongle DP-> DVI(单链路)连接类型的有效。
* `ADL_CONNECTION_TYPE_ACTIVE_DONGLE_DP_DVI_DL`：表示主动Dongle DP-> DVI(双链路)连接类型的有效。


```
// @{
/// Indicates the VGA connection type is valid.
#define ADL_CONNECTION_TYPE_VGA 0
/// Indicates the DVI_I connection type is valid.
#define ADL_CONNECTION_TYPE_DVI 1
/// Indicates the DVI_SL connection type is valid.
#define ADL_CONNECTION_TYPE_DVI_SL 2
/// Indicates the HDMI connection type is valid.
#define ADL_CONNECTION_TYPE_HDMI 3
/// Indicates the DISPLAY PORT connection type is valid.
#define ADL_CONNECTION_TYPE_DISPLAY_PORT 4
/// Indicates the Active dongle DP->DVI(single link) connection type is valid.
#define ADL_CONNECTION_TYPE_ACTIVE_DONGLE_DP_DVI_SL 5
/// Indicates the Active dongle DP->DVI(double link) connection type is valid.
#define ADL_CONNECTION_TYPE_ACTIVE_DONGLE_DP_DVI_DL 6
```cpp



这段代码定义了DP和HDMI连接器的有效连接类型，用于指示不同设备是否支持DP或HDMI连接。其中，有效的连接类型有四种，分别为：

- #define ADL_CONNECTION_TYPE_ACTIVE_DONGLE_DP_HDMI 7：表示主动笔记本电脑的DP或HDMI连接是有效的。
- #define ADL_CONNECTION_TYPE_ACTIVE_DONGLE_DP_VGA 8：表示主动笔记本电脑的DP或VGA连接是有效的。
- #define ADL_CONNECTION_TYPE_PASSIVE_DONGLE_DP_HDMI 9：表示被动笔记本电脑的DP或HDMI连接是有效的。
- #define ADL_CONNECTION_TYPE_PASSIVE_DONGLE_DP_DVI 10：表示被动笔记本电脑的DP或DVI连接是有效的。
- #define ADL_CONNECTION_TYPE_MST 11：表示服务器或变换器上的DP或HDMI连接是有效的。
- #define ADL_CONNECTION_TYPE_VIRTUAL 13：表示虚拟机上的DP或HDMI连接是有效的。

此外，#define后面还定义了一些宏，用于生成位mask。


```
/// Indicates the Active dongle DP->HDMI connection type is valid.
#define ADL_CONNECTION_TYPE_ACTIVE_DONGLE_DP_HDMI 7
/// Indicates the Active dongle DP->VGA connection type is valid.
#define ADL_CONNECTION_TYPE_ACTIVE_DONGLE_DP_VGA 8
/// Indicates the Passive dongle DP->HDMI connection type is valid.
#define ADL_CONNECTION_TYPE_PASSIVE_DONGLE_DP_HDMI 9
/// Indicates the Active dongle DP->VGA connection type is valid.
#define ADL_CONNECTION_TYPE_PASSIVE_DONGLE_DP_DVI 10
/// Indicates the MST type is valid.
#define ADL_CONNECTION_TYPE_MST 11
/// Indicates the active dongle, all types.
#define ADL_CONNECTION_TYPE_ACTIVE_DONGLE          12
/// Indicates the Virtual Connection Type.
#define ADL_CONNECTION_TYPE_VIRTUAL    13
/// Macros for generating bitmask from index.
```cpp

这段代码定义了一个名为ADL_CONNECTION_BITMAST_FROM_INDEX的宏，表示从索引index开始，将所有定义连接属性的位（bit）设置为1。

接下来的代码定义了一系列的连接属性，包括：

* 指示比特率（bitrate）是否有效，定义为0x1；
* 指示通道数量（number of lanes）是否有效，定义为0x2；
* 指示3D补偿是否有效，定义为0x4；
* 指示输出带宽是否有效，定义为0x8；
* 指示颜色深度是否有效，定义为0x16。

这些定义是由一个名为ADL_Adapter_SupportedConnections_Get的函数使用的，它们可以用来检查给定的连接类型支持哪些属性。


```
#define ADL_CONNECTION_BITMAST_FROM_INDEX(index) (1 << index)
// @}

/// \defgroup define_connection_properties
/// These defines are the connection properties which indicates what are the valid properties of given connection type. It is used by \ref ADL_Adapter_SupportedConnections_Get
// @{
/// Indicates the property Bitrate is valid.
#define ADL_CONNECTION_PROPERTY_BITRATE 0x1
/// Indicates the property number of lanes is valid.
#define ADL_CONNECTION_PROPERTY_NUMBER_OF_LANES 0x2
/// Indicates the property 3D caps is valid.
#define ADL_CONNECTION_PROPERTY_3DCAPS  0x4
/// Indicates the property output bandwidth is valid.
#define ADL_CONNECTION_PROPERTY_OUTPUT_BANDWIDTH 0x8
/// Indicates the property colordepth is valid.
```cpp

这段代码定义了一系列与 lane count（即车道数量）相关的常量，用于在 DP（数据平面）和其他相关领域中使用。其中，ADL_CONNECTION_PROPERTY_COLORDEPTH定义了一个名为“ADL_CONNECTION_PROPERTY_COLORDEPTH”的宏，其值为0x10。

接下来的部分定义了一系列常量，用于表示车道数量的不同状态。这些常量用于在 DP 和其他相关领域中进行数据交换和处理。例如，ADL_LANECOUNT_UNKNOWN表示未知（或未配置）车道数量；ADL_LANECOUNT_ONE表示只有一个车道；ADL_LANECOUNT_TWO表示两个车道；ADL_LANECOUNT_FOUR表示四个车道；ADL_LANECOUNT_EIGHT表示八个车道。


```
#define ADL_CONNECTION_PROPERTY_COLORDEPTH  0x10
// @}

/// \defgroup define_lanecount_constants
/// These defines are the Lane count constants which will be used in DP & etc.
// @{
/// Indicates if lane count is unknown
#define ADL_LANECOUNT_UNKNOWN 0
/// Indicates if lane count is 1
#define ADL_LANECOUNT_ONE 1
/// Indicates if lane count is 2
#define ADL_LANECOUNT_TWO 2
/// Indicates if lane count is 4
#define ADL_LANECOUNT_FOUR 4
/// Indicates if lane count is 8
```cpp

这段代码定义了一些用于定义自动驾驶列数（lane count）和链速常量（link rate constant）的宏。

首先，定义了一个名为 ADL_LANECOUNT_EIGHT 的宏，它的值为 eight，表示自动驾驶列数为八个。这个宏可以被用来定义自动驾驶系统中的 lane count 变量。

接着，定义了一个名为 ADL_LANECOUNT_DEF 的宏，它的值为 ADL_LANECOUNT_FOUR，表示自动驾驶列数的默认值为四个。这个宏也可以被用来定义自动驾驶系统中的 default_lane_count 变量。

然后，定义了一个名为 ADL_LINK_BITRATE_UNKNOWN 的宏，它的值为 0，表示链接速率未知。这个宏可以被用来检查链接速率是否为未知值。

接着，定义了一个名为 ADL_LINK_BITRATE_1_62_GHZ 的宏，它的值为 0x06，表示链速为 1.62Ghz。这个宏可以被用来定义自动驾驶系统中的 link_speed_1_62GHZ 变量。

接着，定义了一个名为 ADL_LINK_BITRATE_2_7_GHZ 的宏，它的值为 0x0A，表示链速为 2.7Ghz。这个宏可以被用来定义自动驾驶系统中的 link_speed_2_7GHZ 变量。

最后，定义了一个名为 ADL_LINK_BITRATE_5_4_GHZ 的宏，它的值为 0x0E，表示链速为 5.4Ghz。这个宏可以被用来定义自动驾驶系统中的 link_speed_5_4GHZ 变量。


```
#define ADL_LANECOUNT_EIGHT 8
/// Indicates default value of lane count
#define ADL_LANECOUNT_DEF ADL_LANECOUNT_FOUR
// @}

/// \defgroup define_linkrate_constants
/// These defines are the link rate constants which will be used in DP & etc.
// @{
/// Indicates if link rate is unknown
#define ADL_LINK_BITRATE_UNKNOWN 0
/// Indicates if link rate is 1.62Ghz
#define ADL_LINK_BITRATE_1_62_GHZ 0x06
/// Indicates if link rate is 2.7Ghz
#define ADL_LINK_BITRATE_2_7_GHZ 0x0A
/// Indicates if link rate is 5.4Ghz
```cpp

这段代码定义了一系列常量，用于指定DP（像素）颜色深度的值。以下是每个常量的说明：

```
#define ADL_LINK_BITRATE_5_4_GHZ 0x14   // 表示链接速率是否为5.4Ghz
#define ADL_LINK_BITRATE_8_1_GHZ 0x1E   // 表示默认链接速率设置
#define ADL_LINK_BITRATE_DEF ADL_LINK_BITRATE_2_7_GHZ  // 表示缺省链接速率设置
```cpp

```
/// Indicates if link rate is 8.1Ghz
#define ADL_CONNPROP_S3D_ALTERNATE_TO_FRAME_PACK            0x00000001
```cpp

```
/// \defgroup define_colordepth_constants
/// These defines are the color depth constants which will be used in DP & etc.
// @{
#define ADL_CONNPROP_S3D_ALTERNATE_TO_FRAME_PACK            0x00000001
// @}
```cpp

其中，`ADL_LINK_BITRATE_5_4_GHZ`表示链接速率为5.4Ghz，`ADL_LINK_BITRATE_8_1_GHZ`表示默认的链接速率设置，`ADL_LINK_BITRATE_DEF`表示缺省的链接速率设置。同时，`ADL_CONNPROP_S3D_ALTERNATE_TO_FRAME_PACK`表示如果当前设置的链接速率与`ADL_CONNPROP_S3D_ALTERNATE_TO_FRAME_PACK`中指定的值相同，则使用该指定值，否则使用缺省值。


```
#define ADL_LINK_BITRATE_5_4_GHZ 0x14

/// Indicates if link rate is 8.1Ghz
#define ADL_LINK_BITRATE_8_1_GHZ 0x1E
/// Indicates default value of link rate
#define ADL_LINK_BITRATE_DEF ADL_LINK_BITRATE_2_7_GHZ
// @}

/// \defgroup define_colordepth_constants
/// These defines are the color depth constants which will be used in DP & etc.
// @{
#define ADL_CONNPROP_S3D_ALTERNATE_TO_FRAME_PACK            0x00000001
// @}


```cpp

这段代码定义了一系列颜色深度常量，用于在数据结构绘图中表明数据结构的颜色深度。这些常量包括颜色深度的枚举类型，每个类型都有四个整数成员，分别表示颜色深度是否为0、666、888或101010。通过将这些常量成员存储为整数，可以方便地在程序中根据需要选择和设置颜色深度。


```
/// \defgroup define_colordepth_constants
/// These defines are the color depth constants which will be used in DP & etc.
// @{
/// Indicates if color depth is unknown
#define ADL_COLORDEPTH_UNKNOWN 0
/// Indicates if color depth is 666
#define ADL_COLORDEPTH_666 1
/// Indicates if color depth is 888
#define ADL_COLORDEPTH_888 2
/// Indicates if color depth is 101010
#define ADL_COLORDEPTH_101010 3
/// Indicates if color depth is 121212
#define ADL_COLORDEPTH_121212 4
/// Indicates if color depth is 141414
#define ADL_COLORDEPTH_141414 5
```cpp

这段代码定义了一系列定义，包括颜色深度相关的定义和 emulation 状态相关的定义。

具体来说，ADL_COLORDEPTH_161616 是一个预定义的常量，表示颜色深度是否为 16 位。如果颜色深度为 16 位，则使用 ADL_COLORDEPTH_888 作为默认值。

ADL_EMUL_STATUS_REAL_DEVICE_CONNECTED，ADL_EMUL_STATUS_EMULATED_DEVICE_PRESENT 和 ADL_EMUL_STATUS_EMULATED_DEVICE_USED 定义了 emulation 状态的三个状态。这些定义的值可以从 0x1 到 0x3 取不同的值，表示 emulation 设备是否与现实设备连接，是否已经连接，以及是否正在使用 emulation 设备。


```
/// Indicates if color depth is 161616
#define ADL_COLORDEPTH_161616 6
/// Indicates default value of color depth
#define ADL_COLOR_DEPTH_DEF ADL_COLORDEPTH_888
// @}


/// \defgroup define_emulation_status
/// These defines are the status of emulation
// @{
/// Indicates if real device is connected.
#define ADL_EMUL_STATUS_REAL_DEVICE_CONNECTED 0x1
/// Indicates if emulated device is presented.
#define ADL_EMUL_STATUS_EMULATED_DEVICE_PRESENT 0x2
/// Indicates if emulated device is used.
```cpp

这段代码定义了一些定义，主要是关于模拟模式的。接下来，我会逐步解释这些定义以及相关的逻辑。

首先，有以下三个定义：

```
#define ADL_EMUL_STATUS_EMULATED_DEVICE_USED  0x4
#define ADL_EMUL_STATUS_LAST_ACTIVE_DEVICE_USED 0x8
#define ADL_EMUL_MODE_OFF 0
#define ADL_EMUL_MODE_ON_CONNECTED 1
#define ADL_EMUL_MODE_ON_DISCONNECTED 2
```cpp

第一个定义是关于模拟模式的状态定义，包括关闭、连接和断开。其中，`ADL_EMUL_STATUS_EMULATED_DEVICE_USED`表示当处于开启的模拟模式时，使用的老设备号，这里的设备可以是虚拟设备或者硬件设备。

第二个定义定义了最后活跃设备的使用状态，分为三种情况：

1. 当处于关闭的模拟模式时，使用的是最后连接的设备。
2. 当处于连接的模拟模式时，最后活跃设备是指显示器，连接成功后，这个设备就是最后的活跃设备。
3. 当处于断开的模拟模式时，最后活跃设备是指虚拟设备，断开和连接成功后，这个设备就是最后的活跃设备。

接下来是关于模拟模式的模式定义：

```
/// Indicates if no emulation is used
#define ADL_EMUL_MODE_OFF 0
/// Indicates if emulation is used when display connected
#define ADL_EMUL_MODE_ON_CONNECTED 1
/// Indicates if emulation is used when display dis connected
#define ADL_EMUL_MODE_ON_DISCONNECTED 2
```cpp

第三组定义定义了模拟模式的选项，包括关闭、连接显示器并连接显示器、以及显示器断开。这里显示器的连接状态与模拟模式的选项没有关系，它们是两个独立的定义。

综上所述，这段代码定义了一些关于模拟模式的定义，包括模拟模式的状态定义、最后活跃设备的定义，以及模拟模式的选项定义。


```
#define ADL_EMUL_STATUS_EMULATED_DEVICE_USED  0x4
/// In case when last active real/emulated device used (when persistence is enabled but no emulation enforced then persistence will use last connected/emulated device).
#define ADL_EMUL_STATUS_LAST_ACTIVE_DEVICE_USED 0x8
// @}

/// \defgroup define_emulation_mode
/// These defines are the modes of emulation
// @{
/// Indicates if no emulation is used
#define ADL_EMUL_MODE_OFF 0
/// Indicates if emulation is used when display connected
#define ADL_EMUL_MODE_ON_CONNECTED 1
/// Indicates if emulation is used when display dis connected
#define ADL_EMUL_MODE_ON_DISCONNECTED 2
/// Indicates if emulation is used always
```cpp

这段代码定义了一个名为 ADL_EMUL_MODE_ALWAYS 的宏，其值为 3。接着定义了一个名为 ADL_QUERY_REAL_DATA 的定义，表示查询真实设备的数据；定义了一个名为 ADL_QUERY_EMULATED_DATA 的定义，表示查询虚拟数据；定义了一个名为 ADL_QUERY_CURRENT_DATA 的定义，表示查询当前正在使用的数据。最后，定义了一个名为 ADL_PERSISTENCE_STATE 的组名，但其中没有定义任何成员。


```
#define ADL_EMUL_MODE_ALWAYS 3
// @}

/// \defgroup define_emulation_query
/// These defines are the modes of emulation
// @{
/// Indicates Data from real device
#define ADL_QUERY_REAL_DATA 0
/// Indicates Emulated data
#define ADL_QUERY_EMULATED_DATA 1
/// Indicates Data currently in use
#define ADL_QUERY_CURRENT_DATA 2
// @}

/// \defgroup define_persistence_state
```cpp

这段代码定义了几个关于 persistence状态的定义，包括ADL_EDID_PERSISTANCE_DISABLED、ADL_EDID_PERSISTANCE_ENABLED和ADL_CONNECTOR_TYPE_UNKNOWN。

ADL_EDID_PERSISTANCE_DISABLED表示可持久存储是否禁用，值为0时禁用，值为1时启用。

ADL_CONNECTOR_TYPE_VGA表示是否连接到VGA适配器，值为0时表示未连接，值为1时连接。


```
/// These defines are the states of persistence
// @{
/// Indicates persistence is disabled
#define ADL_EDID_PERSISTANCE_DISABLED 0
/// Indicates persistence is enabled
#define ADL_EDID_PERSISTANCE_ENABLED 1
// @}

/// \defgroup define_connector_types Connector Type
/// defines for ADLConnectorInfo.iType
// @{
/// Indicates unknown Connector type
#define ADL_CONNECTOR_TYPE_UNKNOWN                 0
/// Indicates VGA Connector type
#define ADL_CONNECTOR_TYPE_VGA                     1
```cpp

这段代码定义了几个常量，用于标识不同的DVI连接器类型。

具体来说，这些常量定义了以下连接器类型：

- DVI-D连接器类型：#define ADL_CONNECTOR_TYPE_DVI_D 2
- DVI-I连接器类型：#define ADL_CONNECTOR_TYPE_DVI_I 3
- Active Dongle-NA连接器类型：#define ADL_CONNECTOR_TYPE_ATICVDONGLE_NA 4
- Active Dongle-JP连接器类型：#define ADL_CONNECTOR_TYPE_ATICVDONGLE_JP 5
- Active Dongle-NONI2C连接器类型：#define ADL_CONNECTOR_TYPE_ATICVDONGLE_NONI2C 6
- Active Dongle-NONI2C-D连接器类型：#define ADL_CONNECTOR_TYPE_ATICVDONGLE_NONI2C_D 7
- HDMI类型A连接器类型：#define ADL_CONNECTOR_TYPE_HDMI_TYPE_A 8
- HDMI类型B连接器类型：#define ADL_CONNECTOR_TYPE_HDMI_TYPE_B


```
/// Indicates DVI-D Connector type
#define ADL_CONNECTOR_TYPE_DVI_D                   2
/// Indicates DVI-I Connector type
#define ADL_CONNECTOR_TYPE_DVI_I                   3
/// Indicates Active Dongle-NA Connector type
#define ADL_CONNECTOR_TYPE_ATICVDONGLE_NA          4
/// Indicates Active Dongle-JP Connector type
#define ADL_CONNECTOR_TYPE_ATICVDONGLE_JP          5
/// Indicates Active Dongle-NONI2C Connector type
#define ADL_CONNECTOR_TYPE_ATICVDONGLE_NONI2C      6
/// Indicates Active Dongle-NONI2C-D Connector type
#define ADL_CONNECTOR_TYPE_ATICVDONGLE_NONI2C_D    7
/// Indicates HDMI-Type A Connector type
#define ADL_CONNECTOR_TYPE_HDMI_TYPE_A             8
/// Indicates HDMI-Type B Connector type
```cpp

这段代码定义了一个命名常量，用于标识显示器连接器类型。其中，ADL_CONNECTOR_TYPE_HDMI_TYPE_B表示为HDMI接口的显示器连接器类型，ADL_CONNECTOR_TYPE_DISPLAYPORT表示为显示 port接口的连接器类型，ADL_CONNECTOR_TYPE_EDP表示为EDP（Enhanced Definition Power）接口的连接器类型，ADL_CONNECTOR_TYPE_MINI_DISPLAYPORT表示为MiniDP（Minimal Display Port）接口的连接器类型，ADL_CONNECTOR_TYPE_VIRTUAL表示为虚拟连接器类型。

另外，还定义了一个名为ADL_CONNECTOR_TYPE_DISPLAYPORT的枚举类型，用于指定具体的显示器连接器类型。该枚举类型定义了四种可能的显示器连接器类型，分别为0到3。其中，ADL_CONNECTOR_TYPE_DISPLAYPORT_TYPE_B表示为标准显示器连接器（VGA、SVG、HDMI等）的显示器连接器类型，ADL_CONNECTOR_TYPE_DISPLAYPORT_TYPE_C表示为电视连接器（ Component or Dual Link）的显示器连接器类型，ADL_CONNECTOR_TYPE_DISPLAYPORT_TYPE_D表示为游戏主机（如Xbox One、PlayStation 4等）的显示器连接器类型，ADL_CONNECTOR_TYPE_DISPLAYPORT_TYPE_E表示为桌面电脑（如Windows 8、Windows 10等）的显示器连接器类型。


```
#define ADL_CONNECTOR_TYPE_HDMI_TYPE_B             9
/// Indicates Display port Connector type
#define ADL_CONNECTOR_TYPE_DISPLAYPORT             10
/// Indicates EDP Connector type
#define ADL_CONNECTOR_TYPE_EDP                     11
/// Indicates MiniDP Connector type
#define ADL_CONNECTOR_TYPE_MINI_DISPLAYPORT        12
/// Indicates Virtual Connector type
#define ADL_CONNECTOR_TYPE_VIRTUAL                   13
// @}

/// \defgroup define_freesync_usecase
/// These defines are to specify use cases in which FreeSync should be enabled
/// They are a bit mask. To specify FreeSync for more than one use case, the input value
/// should be set to include multiple bits set
```cpp

这段代码定义了一系列与FreeSync功能相关的头文件和常量，用于描述在不同使用场景下是否支持FreeSync。

具体来说，这些头文件定义了以下三个标志：

1. `ADL_FREESYNC_USECASE_STATIC`：表示静态屏幕使用场景下是否支持FreeSync。如果设置为`0x1`，则表示支持，否则表示不支持。
2. `ADL_FREESYNC_USECASE_VIDEO`：表示视频使用场景下是否支持FreeSync。如果设置为`0x2`，则表示支持，否则表示不支持。
3. `ADL_FREESYNC_USECASE_GAMING`：表示游戏使用场景下是否支持FreeSync。如果设置为`0x4`，则表示支持，否则表示不支持。

此外，还有一系列常量，用于描述上述标志，如`ADL_FREESYNC_CAP_SUPPORTED`等，用于描述FreeSync是否连接到支持GPU，或者静态屏幕和视频是否共享同一个显示器等。


```
// @{
/// Indicates FreeSync is enabled for Static Screen case
#define ADL_FREESYNC_USECASE_STATIC                 0x1
/// Indicates FreeSync is enabled for Video use case
#define ADL_FREESYNC_USECASE_VIDEO                  0x2
/// Indicates FreeSync is enabled for Gaming use case
#define ADL_FREESYNC_USECASE_GAMING                 0x4
// @}

/// \defgroup define_freesync_caps
/// These defines are used to retrieve FreeSync display capabilities.
/// GPU support flag also indicates whether the display is
/// connected to a GPU that actually supports FreeSync
// @{
#define ADL_FREESYNC_CAP_SUPPORTED                      (1 << 0)
```cpp

这段代码定义了一系列头文件，用于定义与显示相关的高清显示（ADL_FREESYNC_CAP）功能。这些头文件定义了多个位（0, 1, 2, 3, 4, 5, 6），用于指示是否支持某些高清晰度显示功能。这些位可以用来设置或清除ADL_FREESYNC_CAP。

具体来说，这些头文件定义了以下符号：

```
#define ADL_FREESYNC_CAP_GPUSUPPORTED                   (1 << 1)
#define ADL_FREESYNC_CAP_DISPLAYSUPPORTED               (1 << 2)
#define ADL_FREESYNC_CAP_CURRENTMODESUPPORTED           (1 << 3)
#define ADL_FREESYNC_CAP_NOCFXORCFXSUPPORTED            (1 << 4)
#define ADL_FREESYNC_CAP_NOGENLOCKORGENLOCKSUPPORTED    (1 << 5)
#define ADL_FREESYNC_CAP_BORDERLESSWINDOWSUPPORTED      (1 << 6)
```cpp

其中，每个符号的意义如下：

- `ADL_FREESYNC_CAP_GPUSUPPORTED`：指示支持GPUsense同步输出吗？，是1，否为0。
- `ADL_FREESYNC_CAP_DISPLAYSUPPORTED`：指示是否支持显示同步，是1，否为0。
- `ADL_FREESYNC_CAP_CURRENTMODESUPPORTED`：指示是否支持当前模式同步，是1，否为0。
- `ADL_FREESYNC_CAP_NOCFXORCFXSUPPORTED`：指示是否支持FX Or CFX输出，是1，否为0。
- `ADL_FREESYNC_CAP_NOGENLOCKORGENLOCKSUPPORTED`：指示是否支持Genlock或Organo lock，是1，否为0。
- `ADL_FREESYNC_CAP_BORDERLESSWINDOWSUPPORTED`：指示是否支持无边界窗口，是1，否为0。

另外，还定义了两个常量：

```
#define ADL_MST_COMMANDLINE_PATH_MSG               0x1
#define ADL_MST_COMMANDLINE_BROADCAST                  0x2
```cpp

这些常量分别表示了命令行路径和广播发送的MSG的编号。


```
#define ADL_FREESYNC_CAP_GPUSUPPORTED                   (1 << 1)
#define ADL_FREESYNC_CAP_DISPLAYSUPPORTED               (1 << 2)
#define ADL_FREESYNC_CAP_CURRENTMODESUPPORTED           (1 << 3)
#define ADL_FREESYNC_CAP_NOCFXORCFXSUPPORTED            (1 << 4)
#define ADL_FREESYNC_CAP_NOGENLOCKORGENLOCKSUPPORTED    (1 << 5)
#define ADL_FREESYNC_CAP_BORDERLESSWINDOWSUPPORTED      (1 << 6)
// @}


/// \defgroup define_MST_CommandLine_execute
// @{
/// Indicates the MST command line for branch message if the bit is set. Otherwise, it is display message
#define ADL_MST_COMMANDLINE_PATH_MSG                 0x1
/// Indicates the MST command line to send message in broadcast way it the bit is set
#define ADL_MST_COMMANDLINE_BROADCAST                  0x2

```cpp

这段代码定义了一个名为 `ADL_CROSSGPU_DISPLAY_CLONE_AMD` 的枚举类型，以及一个名为 `ADL_CROSSGPU_DISPLAY_CLONE` 的枚举类型。这两个枚举类型都表示跨GPU复制，但第一个枚举类型`ADL_CROSSGPUDISPLAYCLONE_AMD_WITH_NONAMD`表示使用NVIDIA的跨GPU复制，而第二个枚举类型`ADL_CROSSGPUDISPLAYCLONE`表示使用AMD的跨GPU复制。

此外，还定义了一个名为 `ADL_CROSSGPUDISPLAYCLONE_AMD_WITH_NONAMD` 的枚举类型，它表示在AMD的跨GPU复制中，同时使用NVIDIA的设备。这个枚举类型定义了一个名为 `ADL_CROSSGPU_DISPLAYCLONE_NVIDIA` 的枚举类型，它表示在NVIDIA的跨GPU复制中，与上述枚举类型相同。


```
// @}


/// \defgroup define_Adapter_CloneTypes_Get
// @{
/// Indicates there is crossGPU clone with non-AMD dispalys
#define ADL_CROSSGPUDISPLAYCLONE_AMD_WITH_NONAMD                 0x1
/// Indicates there is crossGPU clone
#define ADL_CROSSGPUDISPLAYCLONE                  0x2

// @}

/// \defgroup define_D3DKMT_HANDLE
// @{
/// Handle can be used to create Device Handle when using CreateDevice()
```cpp

这段代码定义了一个名为ADL_D3DKMT_HANDLE的未定义类型，可能用于在应用程序中保存与D3DKMT相关的数据。

接下来定义了一个名为ADL_RAS_ERROR_INJECTION_MODE的枚举类型，用于表示D3DKMT中错误注入的模式，包括ADL_RAS_ERROR_INJECTION_MODE_SINGLE和ADL_RAS_ERROR_INJECTION_MODE_MULTIPLE两种取值。

最后，没有添加任何其他说明，因此该代码可能是一个无效的C或C++代码片段。


```
typedef unsigned int ADL_D3DKMT_HANDLE;
// @}


// End Bracket for Constants and Definitions. Add new groups ABOVE this line!

// @}


typedef enum _ADL_RAS_ERROR_INJECTION_MODE
{
	ADL_RAS_ERROR_INJECTION_MODE_SINGLE = 1,
	ADL_RAS_ERROR_INJECTION_MODE_MULTIPLE = 2
}ADL_RAS_ERROR_INJECTION_MODE;


```cpp

这段代码定义了一个枚举类型ADL_RAS_BLOCK_ID，包含了11个枚举常量，每个枚举常量都有一个唯一的ID，如下所示：

ADL_RAS_BLOCK_ID_UMC = 0
ADL_RAS_BLOCK_ID_SDMA = 1
ADL_RAS_BLOCK_ID_GFX_HUB = 2
ADL_RAS_BLOCK_ID_MMHUB = 3
ADL_RAS_BLOCK_ID_ATHUB = 4
ADL_RAS_BLOCK_ID_PCIE_BIF = 5
ADL_RAS_BLOCK_ID_HDP = 6
ADL_RAS_BLOCK_ID_XGMI_WAFL = 7
ADL_RAS_BLOCK_ID_DF = 8
ADL_RAS_BLOCK_ID_SMN = 9
ADL_RAS_BLOCK_ID_SEM = 10
ADL_RAS_BLOCK_ID_MP0 = 11

ADL_RAS_BLOCK_ID可以用于定义与枚举相关的行为，例如确定芯片的扇区类型或用于生成校准配置。


```
typedef enum _ADL_RAS_BLOCK_ID
{
	ADL_RAS_BLOCK_ID_UMC = 0,
	ADL_RAS_BLOCK_ID_SDMA,
	ADL_RAS_BLOCK_ID_GFX_HUB,
	ADL_RAS_BLOCK_ID_MMHUB,
	ADL_RAS_BLOCK_ID_ATHUB,
	ADL_RAS_BLOCK_ID_PCIE_BIF,
	ADL_RAS_BLOCK_ID_HDP,
	ADL_RAS_BLOCK_ID_XGMI_WAFL,
	ADL_RAS_BLOCK_ID_DF,
	ADL_RAS_BLOCK_ID_SMN,
	ADL_RAS_BLOCK_ID_SEM,
	ADL_RAS_BLOCK_ID_MP0,
	ADL_RAS_BLOCK_ID_MP1,
	ADL_RAS_BLOCK_ID_FUSE
}ADL_RAS_BLOCK_ID;

```cpp

ADL_RAS__UMC_HBM = 0,
ADL_RAS__UMC_SRAM = 1,
ADL_MEM_SUB_BLOCK_ID = {0,1},
ADL_RAS_ERROR_TYPE = {ADL_RAS_ERROR__NONE, ADL_RAS_ERROR__PARITY, ADL_RAS_ERROR__SINGLE_CORRECTABLE, ADL_RAS_ERROR__PARITY_SINGLE_CORRECTABLE, ADL_RAS_ERROR__MULTI_UNCORRECTABLE, ADL_RAS_ERROR__PARITY_MULTI_UNCORRECTABLE, ADL_RAS_ERROR__SINGLE_CORRECTABLE_MULTI_UNCORRECTABLE, ADL_RAS_ERROR__PARITY_SINGLE_CORRECTABLE_MULTI_UNCORRECTABLE, ADL_RAS_ERROR__SINGLE_CORRECTABLE_MULTI_UNCORRECTABLE, ADL_RAS_ERROR__SINGLE_CORRECTABLE_MULTI_UNCORRECTABLE, ADL_RAS_ERROR__MULTI_UNCORRECTABLE, ADL_RAS_ERROR__SINGLE_CORRECTABLE_MULTI_UNCORRECTABLE, ADL_RAS_ERROR__SINGLE_CORRECTABLE_MULTI_UNCORRECTABLE, ADL_RAS_ERROR__SINGLE_CORRECTABLE_MULTI_UNCORRECTABLE, ADL_RAS_ERROR__SINGLE_CORRECTABLE_MULTI_UNCORRECTABLE, ADL_RAS_ERROR__SINGLE_CORRECTABLE_MULTI_UNCORRECTABLE, ADL_RAS_ERROR__SINGLE_CORRECTABLE_MULTI_UNCORRECTABLE, ADL_RAS_ERROR__SINGLE_CORRECTABLE_MULTI_UNCORRECTABLE, ADL_RAS_ERROR__SINGLE_CORRECTABLE_MULTI_UNCORRECTABLE, ADL_RAS_ERROR__MULTI_UNCORRECTABLE}


```
typedef enum _ADL_MEM_SUB_BLOCK_ID
{
	ADL_RAS__UMC_HBM = 0,
	ADL_RAS__UMC_SRAM = 1
}ADL_MEM_SUB_BLOCK_ID;

typedef enum  _ADL_RAS_ERROR_TYPE
{
	ADL_RAS_ERROR__NONE = 0,
	ADL_RAS_ERROR__PARITY = 1,
	ADL_RAS_ERROR__SINGLE_CORRECTABLE = 2,
	ADL_RAS_ERROR__PARITY_SINGLE_CORRECTABLE = 3,
	ADL_RAS_ERROR__MULTI_UNCORRECTABLE = 4,
	ADL_RAS_ERROR__PARITY_MULTI_UNCORRECTABLE = 5,
	ADL_RAS_ERROR__SINGLE_CORRECTABLE_MULTI_UNCORRECTABLE = 6,
	ADL_RAS_ERROR__PARITY_SINGLE_CORRECTABLE_MULTI_UNCORRECTABLE = 7,
	ADL_RAS_ERROR__POISON = 8,
	ADL_RAS_ERROR__PARITY_POISON = 9,
	ADL_RAS_ERROR__SINGLE_CORRECTABLE_POISON = 10,
	ADL_RAS_ERROR__PARITY_SINGLE_CORRECTABLE_POISON = 11,
	ADL_RAS_ERROR__MULTI_UNCORRECTABLE_POISON = 12,
	ADL_RAS_ERROR__PARITY_MULTI_UNCORRECTABLE_POISON = 13,
	ADL_RAS_ERROR__SINGLE_CORRECTABLE_MULTI_UNCORRECTABLE_POISON = 14,
	ADL_RAS_ERROR__PARITY_SINGLE_CORRECTABLE_MULTI_UNCORRECTABLE_POISON = 15
}ADL_RAS_ERROR_TYPE;

```cpp

这段代码定义了一个名为 ADL_RAS_INJECTION_METHOD 的枚举类型，它定义了不同类型的注入错误。

下一个定义了一个名为 ADL_DRIVER_EVENT_TYPE 的枚举类型，它定义了不同的驱动器事件类型。

下一个段落定义了常量 ADL_RAS_INJECTION_METHOD，它包含四个枚举类型成员：ADL_RAS_ERROR__UMC_METH_COHERENT，ADL_RAS_ERROR__UMC_METH_SINGLE_SHOT，ADL_RAS_ERROR__UMC_METH_PERSISTENT 和 ADL_RAS_ERROR__UMC_METH_PERSISTENT_DISABLE。

这些成员表示了不同类型的注入错误，每种类型的注入错误都有两个不同的函数可以调用，分别是：异步注入错误处理函数和常规注入错误处理函数。


```
typedef enum _ADL_RAS_INJECTION_METHOD
{
	ADL_RAS_ERROR__UMC_METH_COHERENT = 0,
	ADL_RAS_ERROR__UMC_METH_SINGLE_SHOT = 1,
	ADL_RAS_ERROR__UMC_METH_PERSISTENT = 2,
	ADL_RAS_ERROR__UMC_METH_PERSISTENT_DISABLE = 3
}ADL_RAS_INJECTION_METHOD;

// Driver event types
typedef enum _ADL_DRIVER_EVENT_TYPE
{
	ADL_EVENT_ID_AUTO_FEATURE_COMPLETED = 30,
	ADL_EVENT_ID_FEATURE_AVAILABILITY = 31,

} ADL_DRIVER_EVENT_TYPE;


```cpp

这段代码定义了一个名为 ADL_UIFEATURES_GROUP 的枚举类型，该类型包含了 11 个枚举常量，每个枚举常量都有一个数字值，分别为 ADL_UIFEATURES_GROUP_DVR、ADL_UIFEATURES_GROUP_TURBOSYNC、ADL_UIFEATURES_GROUP_FRAMEMETRICSMONITOR、ADL_UIFEATURES_GROUP_FRTC、ADL_UIFEATURES_GROUP_XVISION、ADL_UIFEATURES_GROUP_BLOCKCHAIN 和 ADL_UIFEATURES_GROUP_GAMEINTELLIGENCE。这些枚举常量的作用是用于指定 UIA 组件中属于哪个功能域。


```
//UIFeature Ids
typedef enum _ADL_UIFEATURES_GROUP
{
	ADL_UIFEATURES_GROUP_DVR = 0,
	ADL_UIFEATURES_GROUP_TURBOSYNC = 1,
	ADL_UIFEATURES_GROUP_FRAMEMETRICSMONITOR = 2,
	ADL_UIFEATURES_GROUP_FRTC = 3,
	ADL_UIFEATURES_GROUP_XVISION = 4,
	ADL_UIFEATURES_GROUP_BLOCKCHAIN = 5,
	ADL_UIFEATURES_GROUP_GAMEINTELLIGENCE = 6,
	ADL_UIFEATURES_GROUP_CHILL = 7,
	ADL_UIFEATURES_GROUP_DELAG = 8,
	ADL_UIFEATURES_GROUP_BOOST = 9,
	ADL_UIFEATURES_GROUP_USU = 10,
	ADL_UIFEATURES_GROUP_XGMI = 11
}ADL_UIFEATURES_GROUP;

```cpp

这段代码是一个 preprocess 指令，用于告诉 C 编译器在编译之前需要做的一些事情。

具体来说，这个 preprocess 指令定义了一个名为 "ADL_DEFINES_H_" 的头文件，其中包含了一些定义。这些定义可能会在源代码中中被其他源代码引用，因此在编译之前需要确定它们的存在，并定义它们。

由于这个 preprocess 指令是在一个预先定义好的头文件中出现的，因此编译器将会读取这个头文件，并检查其中定义是否已经被定义过。如果头文件中定义已经被定义过，那么编译器将会继续处理源代码，否则将会报错。

这个 preprocess 指令的作用是定义了一个头文件，其中包含了一些定义，用于在编译之前确定这些定义的存在，并定义它们。


```
#endif /* ADL_DEFINES_H_ */



```