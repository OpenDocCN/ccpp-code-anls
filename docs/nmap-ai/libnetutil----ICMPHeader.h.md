# `nmap\libnetutil\ICMPHeader.h`

```cpp
# 这段代码最初是 Nping 工具的一部分。

# 防止头文件被重复包含
#ifndef __ICMPHEADER_H__
#define __ICMPHEADER_H__  1

# 包含 PacketElement 头文件
#include "PacketElement.h"

# 定义 ICMPHeader 类，继承自 PacketElement 类
class ICMPHeader : public PacketElement {

  public:
    # 虚函数，返回 ICMP 报文类型
    virtual u8 getType() const = 0;

    # 虚函数，设置 ICMP 报文类型
    virtual int setType(u8 val) = 0;

    # 虚函数，返回 ICMP 报文代码
    virtual u8 getCode() const = 0;

    # 虚函数，设置 ICMP 报文代码
    virtual int setCode(u8 val) = 0;

    # 虚函数，返回是否为错误报文
    virtual bool isError() const = 0;
};

# 结束头文件防止重复包含
#endif /* __ICMPHEADER_H__ */
```