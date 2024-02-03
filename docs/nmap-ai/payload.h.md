# `nmap\payload.h`

```cpp
/* $Id$ */

#ifndef PAYLOAD_H
#define PAYLOAD_H

#include "service_scan.h"

// 定义最大负载数量的限制，使用u8类型进行索引和检索，并且需要一次性发送所有负载，不能过多
#define MAX_PAYLOADS_PER_PORT 0xff

// 获取UDP目的端口的负载数据，返回负载数据指针，设置长度，并指定索引
const u8 *get_udp_payload(u16 dport, int *length, u8 index);
// 获取UDP目的端口的负载数量
u8 udp_payload_count(u16 dport);
// 根据UDP目的端口和缓冲区内容，返回匹配详情结构体指针
const struct MatchDetails *payload_service_match(u16 dport, const u8 *buf, int buflen);
// 初始化负载
void init_payloads(void);
// 释放负载
void free_payloads(void);

#endif /* PAYLOAD_H */
```