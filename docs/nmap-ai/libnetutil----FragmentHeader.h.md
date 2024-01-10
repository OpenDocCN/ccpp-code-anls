# `nmap\libnetutil\FragmentHeader.h`

```
/* This code was originally part of the Nping tool.                        */
// 定义了一个宏，用于防止头文件被重复包含
#ifndef __FRAGMENT_HEADER_H__
#define __FRAGMENT_HEADER_H__ 1

// 包含 IPv6ExtensionHeader.h 头文件
#include "IPv6ExtensionHeader.h"

// 定义分片头部的长度
#define FRAGMENT_HEADER_LEN 8

// 定义了一个名为 FragmentHeader 的类，继承自 IPv6ExtensionHeader 类
class FragmentHeader : public IPv6ExtensionHeader {

    private:

     /* +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
        |  Next Header  |   Reserved    |      Fragment Offset    |Res|M|
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
        |                         Identification                        |
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ */
        // 定义了一个名为 nping_ipv6_ext_fragment_hdr 的结构体
        struct nping_ipv6_ext_fragment_hdr{
            u8 nh; // Next Header 字段
            u8 res1; // Reserved 字段
            u8 off_res_flag[2]; // Fragment Offset 和 Res/M 字段
            u32 id; // Identification 字段
        }__attribute__((__packed__));
        typedef struct nping_ipv6_ext_fragment_hdr nping_ipv6_ext_fragment_hdr_t;

        // 声明了一个 nping_ipv6_ext_fragment_hdr_t 类型的变量 h
        nping_ipv6_ext_fragment_hdr_t h;

    public:
        // 构造函数
        FragmentHeader();
        // 析构函数
        ~FragmentHeader();
        // 重置方法
        void reset();
        // 获取缓冲区指针方法
        u8 *getBufferPointer();
        // 存储接收数据方法
        int storeRecvData(const u8 *buf, size_t len);
        // 获取协议 ID 方法
        int protocol_id() const;
        // 验证方法
        int validate();
        // 打印方法
        int print(FILE *output, int detail) const;

        /* Protocol specific methods */
        // 设置 Next Header 字段方法
        int setNextHeader(u8 val);
        // 获取 Next Header 字段方法
        u8 getNextHeader();

        // 设置 Fragment Offset 字段方法
        int setOffset(u16 val);
        // 获取 Fragment Offset 字段方法
        u16 getOffset();

        // 设置 M 标志位方法
        int setM(bool m_flag);
        // 获取 M 标志位方法
        bool getM();

        // 设置 Identification 字段方法
        int setIdentification(u32 val);
        // 获取 Identification 字段方法
        u32 getIdentification();


}; /* End of class FragmentHeader */

#endif
```