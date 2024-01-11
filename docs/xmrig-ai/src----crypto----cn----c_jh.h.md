# `xmrig\src\crypto\cn\c_jh.h`

```
/*This program gives the 64-bit optimized bitslice implementation of JH using ANSI C
   该程序使用 ANSI C 提供了 JH 的 64 位优化比特切片实现

   --------------------------------
   Performance
   性能

   Microprocessor: Intel CORE 2 processor (Core 2 Duo Mobile T6600 2.2GHz)
   微处理器：Intel CORE 2 处理器（Core 2 Duo Mobile T6600 2.2GHz）
   Operating System: 64-bit Ubuntu 10.04 (Linux kernel 2.6.32-22-generic)
   操作系统：64 位 Ubuntu 10.04（Linux 内核 2.6.32-22-generic）
   Speed for long message:
   长消息的速度：
   1) 45.8 cycles/byte   compiler: Intel C++ Compiler 11.1   compilation option: icc -O2
   1) 45.8 循环/字节   编译器：Intel C++ 编译器 11.1   编译选项：icc -O2
   2) 56.8 cycles/byte   compiler: gcc 4.4.3                 compilation option: gcc -O3
   2) 56.8 循环/字节   编译器：gcc 4.4.3                 编译选项：gcc -O3

   --------------------------------
   Last Modified: January 16, 2011
   最后修改日期：2011 年 1 月 16 日
*/
#pragma once
// 只包含一次该头文件

#include "hash.h"
// 包含名为 "hash.h" 的头文件

HashReturn jh_hash(int hashbitlen, const BitSequence *data, DataLength databitlen, BitSequence *hashval);
// 声明函数 jh_hash，接受哈希位长度、数据、数据位长度和哈希值作为参数，返回 HashReturn 类型的值
```