# `xmrig\src\crypto\randomx\virtual_memory.cpp`

```cpp
/*
版权所有 (c) 2018-2019, tevador <tevador@gmail.com>

保留所有权利。

在源代码和二进制形式下，无论是否修改，只要满足以下条件，就可以重新分发和使用：
    * 必须保留上述版权声明、此条件列表和以下免责声明。
    * 在二进制形式下，必须在文档和/或其他提供的材料中复制上述版权声明、此条件列表和以下免责声明。
    * 未经特定事先书面许可，不得使用版权持有人的名称或贡献者的名称来认可或推广从本软件派生的产品。

本软件由版权持有人和贡献者“按原样”提供，不提供任何明示或暗示的担保，
包括但不限于对适销性和特定用途的适用性的暗示担保。
在任何情况下，版权持有人或贡献者均不对任何直接、间接、偶然、特殊、惩罚性或后果性损害
（包括但不限于替代商品或服务的采购、使用、数据或利润损失，或业务中断）承担任何责任，
无论是合同责任、严格责任还是侵权行为(包括疏忽或其他)的任何理论，即使事先已被告知此类损害的可能性。
*/

#include <stdexcept>


#include "crypto/common/VirtualMemory.h"
#include "crypto/randomx/virtual_memory.hpp"

// 分配可执行内存的函数，参数为字节数和是否使用大页
void* allocExecutableMemory(std::size_t bytes, bool hugePages) {
    // 调用VirtualMemory类的allocateExecutableMemory函数分配可执行内存
    void *mem = xmrig::VirtualMemory::allocateExecutableMemory(bytes, hugePages);
    // 如果分配失败，则抛出运行时错误
    if (mem == nullptr) {
        throw std::runtime_error("Failed to allocate executable memory");
    }

    // 返回分配的内存指针
    return mem;
}

// 分配大页内存的函数，参数为字节数
void* allocLargePagesMemory(std::size_t bytes) {
    # 使用xmrig::VirtualMemory::allocateLargePagesMemory函数分配一块大页内存，并将其地址赋给指针变量mem
    void *mem = xmrig::VirtualMemory::allocateLargePagesMemory(bytes);
    # 如果分配内存失败，抛出std::runtime_error异常，提示内存分配失败
    if (mem == nullptr) {
        throw std::runtime_error("Failed to allocate large pages memory");
    }
    # 返回分配的内存地址
    return mem;
# 释放分页内存的函数
void freePagedMemory(void* ptr, std::size_t bytes) {
    # 调用 VirtualMemory 类的 freeLargePagesMemory 方法释放大页内存
    xmrig::VirtualMemory::freeLargePagesMemory(ptr, bytes);
}
```