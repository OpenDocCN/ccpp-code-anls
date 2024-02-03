# `nmap\MACLookup.h`

```cpp
/* $Id$ */

#ifndef MACLOOKUP_H
#define MACLOOKUP_H

#include <nbase.h>

/* 根据 MAC 地址的前缀返回注册了该前缀的公司名称。
   如果找不到给定前缀的供应商，或者出现其他错误，则返回 NULL。 */
const char *MACPrefix2Corp(const u8 *prefix);

/* 接受一个字符串，并在表中查找包含该字符串的供应商名称。
   设置mac_data中的初始字节，并返回找到的第一个匹配条目的半字节（nibble）数。
   如果没有条目匹配，则保持mac_data不变并返回false。
   注意，这不是特别高效的方法，如果经常调用，应该重写。 */
int MACCorp2Prefix(const char *vendorstr, u8 *mac_data);

#endif /* MACLOOKUP_H */
```