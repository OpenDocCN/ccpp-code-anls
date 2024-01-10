# `nmap\libnetutil\TransportLayerElement.h`

```
// 这段代码最初是 Nping 工具的一部分。

#ifndef TRANSPORTLAYERELEMENT_H
#define TRANSPORTLAYERELEMENT_H  1

#include "PacketElement.h"

/// class TransportLayerElement - TransportLayerElement 类的定义
class TransportLayerElement : public PacketElement {

  public:

    /* Returns source port. */ // 返回源端口
    virtual u16 getSourcePort() const = 0;

    /* Sets source port. */ // 设置源端口
    virtual int setSourcePort(u16 val) = 0;

    /* Returns destination port. */ // 返回目的端口
    virtual u16 getDestinationPort() const = 0;

    /* Sets destination port. */ // 设置目的端口
    virtual int setDestinationPort(u16 val) = 0;

    /* Sets checksum. */ // 设置校验和
    virtual int setSum(u16 val) = 0;

  protected:
    u16 compute_checksum(); // 计算校验和
};

#endif
```