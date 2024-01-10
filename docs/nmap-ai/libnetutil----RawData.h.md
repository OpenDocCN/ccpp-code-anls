# `nmap\libnetutil\RawData.h`

```
/* This code was originally part of the Nping tool.                        */
// 定义了一个名为RAWDATA_H的宏，用于避免重复包含同一个头文件
#ifndef RAWDATA_H
#define RAWDATA_H  1

// 包含ApplicationLayerElement.h头文件
#include "ApplicationLayerElement.h"

// 定义RawData类，继承自ApplicationLayerElement类
class RawData : public ApplicationLayerElement {

  private:
    u8 *data; // 声明一个名为data的u8指针

   public:
        RawData(); // 构造函数
        ~RawData(); // 析构函数
        void reset(); // 重置函数
        u8 *getBufferPointer(); // 获取缓冲区指针函数
        int storeRecvData(const u8 *buf, size_t len); // 存储接收到的数据函数
        int protocol_id() const; // 协议ID函数
        int validate(); // 验证函数
        int print(FILE *output, int detail) const; // 打印函数

        u8 *getBufferPointer(int *mylen); // 获取缓冲区指针函数
        int store(const u8 *buf, size_t len); // 存储函数
        int store(const char *str); // 存储函数

};

#endif
```