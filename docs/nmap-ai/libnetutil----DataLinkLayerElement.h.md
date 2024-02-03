# `nmap\libnetutil\DataLinkLayerElement.h`

```cpp
/* This code was originally part of the Nping tool.                        */
// 这段代码最初是 Nping 工具的一部分。

#ifndef DATALINKLAYERELEMENT_H
#define DATALINKLAYERELEMENT_H   1
// 如果 DATALINKLAYERELEMENT_H 未定义，则定义为 1

#include "PacketElement.h"
// 包含 PacketElement.h 文件

class DataLinkLayerElement : public PacketElement {
};
// 定义 DataLinkLayerElement 类，继承自 PacketElement 类

#endif
```