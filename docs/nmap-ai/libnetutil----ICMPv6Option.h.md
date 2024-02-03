# `nmap\libnetutil\ICMPv6Option.h`

```cpp
/* This code was originally part of the Nping tool.                        */
// 定义了 ICMPv6Option 类，继承自 NetworkLayerElement 类

#ifndef __ICMPv6OPTION_H__
#define __ICMPv6OPTION_H__ 1
// 如果未定义 __ICMPv6OPTION_H__，则定义为 1

#include "NetworkLayerElement.h"
// 包含 NetworkLayerElement.h 文件

/* Packet header diagrams included in this file have been taken from the
 * following IETF RFC documents: RFC 2461, RFC 2894 */
// 本文件中包含的数据包头图表取自以下 IETF RFC 文档：RFC 2461，RFC 2894

/* The following codes have been defined by IANA. A complete list may be found
 * at http://www.iana.org/assignments/icmpv6-parameters */
// 以下代码由 IANA 定义。完整列表可在 http://www.iana.org/assignments/icmpv6-parameters 找到

/* ICMPv6 Option Types */
#define ICMPv6_OPTION_SRC_LINK_ADDR 1
#define ICMPv6_OPTION_TGT_LINK_ADDR 2
#define ICMPv6_OPTION_PREFIX_INFO   3
#define ICMPv6_OPTION_REDIR_HDR     4
#define ICMPv6_OPTION_MTU           5
// ICMPv6 选项类型

/* Nping ICMPv6Options Class internal definitions */
#define ICMPv6_OPTION_COMMON_HEADER_LEN    2
#define ICMPv6_OPTION_MIN_HEADER_LEN       8
#define ICMPv6_OPTION_SRC_LINK_ADDR_LEN    (ICMPv6_OPTION_COMMON_HEADER_LEN+6)
#define ICMPv6_OPTION_TGT_LINK_ADDR_LEN    (ICMPv6_OPTION_COMMON_HEADER_LEN+6)
#define ICMPv6_OPTION_PREFIX_INFO_LEN      (ICMPv6_OPTION_COMMON_HEADER_LEN+30)
#define ICMPv6_OPTION_REDIR_HDR_LEN        (ICMPv6_OPTION_COMMON_HEADER_LEN+6)
#define ICMPv6_OPTION_MTU_LEN              (ICMPv6_OPTION_COMMON_HEADER_LEN+6)
// Nping ICMPv6Options 类的内部定义

/* This must the MAX() of all values defined above*/
#define ICMPv6_OPTION_MAX_MESSAGE_BODY     (ICMPv6_OPTION_PREFIX_INFO_LEN-ICMPv6_OPTION_COMMON_HEADER_LEN)
// 这必须是上述所有值的最大值

#define ICMPv6_OPTION_LINK_ADDRESS_LEN 6
// ICMPv6 选项链接地址长度为 6

class ICMPv6Option : public NetworkLayerElement {
// 定义 ICMPv6Option 类，继承自 NetworkLayerElement 类
    # 默认构造函数，无参数
    public:
        ICMPv6Option();
    # 析构函数，无参数
        ~ICMPv6Option();
    # 重置对象状态
        void reset();
    # 获取缓冲区指针
        u8 *getBufferPointer();
    # 存储接收到的数据
        int storeRecvData(const u8 *buf, size_t len);
    # 获取协议 ID
        int protocol_id() const;

    # 设置类型字段的数值
        int setType(u8 val);
    # 获取类型字段的数值
        u8 getType();
    # 验证类型字段的数值是否有效
        bool validateType(u8 val);

    # 设置长度字段的数值
        int setLength(u8 val);
    # 获取长度字段的数值
        u8 getLength();

    # 设置链路地址字段的数值
        int setLinkAddress(u8* val);
    # 获取链路地址字段的数值
        u8 *getLinkAddress();

    # 设置前缀长度字段的数值
        int setPrefixLength(u8 val);
    # 获取前缀长度字段的数值
        u8 getPrefixLength();

    # 设置标志字段的数值
        int setFlags(u8 val);
    # 获取标志字段的数值
        u8 getFlags();

    # 设置有效生存期字段的数值
        int setValidLifetime(u32 val);
    # 获取有效生存期字段的数值
        u32 getValidLifetime();

    # 设置首选生存期字段的数值
        int setPreferredLifetime(u32 val);
    # 获取首选生存期字段的数值
        u32 getPreferredLifetime();

    # 设置前缀字段的数值
        int setPrefix(u8 *val);
    # 获取前缀字段的数值
        u8 *getPrefix();

    # 设置最大传输单元字段的数值
        int setMTU(u32 val);
    # 获取最大传输单元字段的数值
        u32 getMTU();

    # 根据类型字段的数值获取头部长度
        int getHeaderLengthFromType(u8 type);
}; /* End of class ICMPv6Option */
#endif

这段代码是 C++ 的注释，用于标记 ICMPv6Option 类的结束。同时，`#endif` 用于结束条件编译指令，表示结束 ICMPv6Option 类的定义。
```