# `nmap\libnetutil\DestOptsHeader.h`

```cpp
/* This code was originally part of the Nping tool.                        */

#ifndef __DESTOPTS_HEADER_H__
#define __DESTOPTS_HEADER_H__ 1

#include "HopByHopHeader.h"

class DestOptsHeader : public HopByHopHeader {

    private:
     /* +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
        |  Next Header  |  Hdr Ext Len  |                               |
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+                               +
        |                                                               |
        .                                                               .
        .                            Options                            .
        .                                                               .
        |                                                               |
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ */
        // Implemented in HopByHopHeader.h
    public:
        // 默认构造函数
        DestOptsHeader();
        // 析构函数
        ~DestOptsHeader();
        // 打印函数，输出到指定文件，可以选择输出详细信息
        int print(FILE *output, int detail) const;
        // 返回协议 ID
        int protocol_id() const;

}; /* End of class DestOptsHeader */

#endif
```