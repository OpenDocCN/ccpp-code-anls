# `nmap\libnetutil\NetworkLayerElement.h`

```
// 这段代码最初是 Nping 工具的一部分。

#ifndef NETWORKLAYERELEMENT_H
#define NETWORKLAYERELEMENT_H  1

#include "PacketElement.h"

/// class NetworkLayerElement - 网络层元素类
class NetworkLayerElement : public PacketElement {

  public:
    // 获取地址长度
    virtual u16 getAddressLength() const{
        return 0;
    }

    // 获取源地址
    virtual const u8 *getSourceAddress() const{
        return NULL;
    }

    // 获取目的地址
    virtual const u8 *getDestinationAddress() const{
        return NULL;
    }

    // 设置下一个头部
    virtual int setNextHeader(u8 val){
        return 0;
    }

    // 获取下一个头部
    virtual u8 getNextHeader() const{
        return 0;
    }
};

#endif
```