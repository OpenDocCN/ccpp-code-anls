# `nmap\services.h`

```cpp
// 防止头文件重复包含
#ifndef SERVICES_H
#define SERVICES_H

// 包含自定义的基本网络库头文件
#include "nbase.h"

// 定义网络服务信息结构体
struct nservent {
  const char *s_name;  // 服务名称
  const char *s_proto;  // 协议名称
  u16 s_port;  // 端口号
};

// 根据服务掩码添加端口到端口表
int addportsfromservmask(const char *mask, u8 *porttbl, int range_type);

// 根据端口和协议获取服务信息
const struct nservent *nmap_getservbyport(u16 port, u16 proto);

// 根据给定的级别和端口列表获取顶级端口
void gettoppts(double level, const char *portlist, struct scan_lists * ports, const char *exclude_list = NULL);

// 释放服务信息
void free_services();

// 结束头文件的条件编译
#endif
```