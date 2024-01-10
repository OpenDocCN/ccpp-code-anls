# `nmap\idle_scan.h`

```
/* $Id$ */

// 防止头文件重复包含
#ifndef IDLE_SCAN_H
#define IDLE_SCAN_H

// 包含必要的头文件
#include <nbase.h>

// 声明 Target 类
class Target;

// 处理未收到端口开放的扫描类型（这些扫描在 pos_scan 中）。Super_scan 包括 FIN/XMAS/NULL/Maimon/UDP 和 IP Proto 等扫描
void idle_scan(Target *target, u16 *portarray, int numports,
               char *proxy, const struct scan_lists *ports);

// 结束头文件的防止重复包含
#endif /* IDLE_SCAN_H */
```