# `nmap\libnetutil\HopByHopHeader.h`

```
#ifndef __HOP_BY_HOP_HEADER_H__
#define __HOP_BY_HOP_HEADER_H__ 1

// 如果未定义__HOP_BY_HOP_HEADER_H__，则定义__HOP_BY_HOP_HEADER_H__为1，表示该头文件已被包含


#include "IPv6ExtensionHeader.h"

// 包含名为"IPv6ExtensionHeader.h"的头文件


#define HOP_BY_HOP_MAX_OPTIONS_LEN 256*8
#define HOPBYHOP_MIN_HEADER_LEN 8
#define HOPBYHOP_MAX_HEADER_LEN (HOPBYHOP_MIN_HEADER_LEN + HOP_BY_HOP_MAX_OPTIONS_LEN)
#define HOPBYHOP_MAX_OPTION_LEN 256

// 定义常量HOP_BY_HOP_MAX_OPTIONS_LEN为256*8，HOPBYHOP_MIN_HEADER_LEN为8，HOPBYHOP_MAX_HEADER_LEN为HOPBYHOP_MIN_HEADER_LEN + HOP_BY_HOP_MAX_OPTIONS_LEN，HOPBYHOP_MAX_OPTION_LEN为256


class HopByHopHeader : public IPv6ExtensionHeader {

// 定义名为HopByHopHeader的类，继承自IPv6ExtensionHeader类


protected:

// 以下内容为受保护的成员


struct nping_ipv6_ext_hopbyhop_hdr{
    u8 nh;
    u8 len;
    u8 options[HOP_BY_HOP_MAX_OPTIONS_LEN];
}__attribute__((__packed__));
typedef struct nping_ipv6_ext_hopbyhop_hdr nping_ipv6_ext_hopbyhop_hdr_t;

// 定义名为nping_ipv6_ext_hopbyhop_hdr的结构体，包含nh、len和options三个成员，使用__attribute__((__packed__))进行紧凑排列，定义名为nping_ipv6_ext_hopbyhop_hdr_t的结构体类型


struct nping_ipv6_ext_hopbyhop_opt{
    u8 type;
    u8 len;
    u8 data[HOPBYHOP_MAX_OPTION_LEN];
}__attribute__((__packed__));
typedef struct nping_ipv6_ext_hopbyhop_opt nping_ipv6_ext_hopbyhop_opt_t;

// 定义名为nping_ipv6_ext_hopbyhop_opt的结构体，包含type、len和data三个成员，使用__attribute__((__packed__))进行紧凑排列，定义名为nping_ipv6_ext_hopbyhop_opt_t的结构体类型


nping_ipv6_ext_hopbyhop_hdr_t h;
u8 *curr_option;

// 声明名为h的nping_ipv6_ext_hopbyhop_hdr_t类型的变量，声明名为curr_option的u8类型指针变量
    # 公共成员函数声明部分

    # 默认构造函数
    HopByHopHeader();

    # 析构函数
    ~HopByHopHeader();

    # 重置对象状态
    void reset();

    # 获取缓冲区指针
    u8 *getBufferPointer();

    # 存储接收到的数据
    int storeRecvData(const u8 *buf, size_t len);

    # 获取协议 ID
    int protocol_id() const;

    # 验证数据有效性
    int validate();

    # 打印信息到文件
    int print(FILE *output, int detail) const;

    # 协议特定方法

    # 设置下一个头部
    int setNextHeader(u8 val);

    # 获取下一个头部
    u8 getNextHeader();

    # 添加选项
    int addOption(u8 type, u8 len, const u8 *data);

    # 添加填充
    int addPadding();
}; /* End of class HopByHopHeader */
#endif

这段代码是C++中的注释，用于标记HopByHopHeader类的结束，并结束条件编译指令。
```