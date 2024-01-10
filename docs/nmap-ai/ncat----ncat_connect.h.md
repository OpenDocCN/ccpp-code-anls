# `nmap\ncat\ncat_connect.h`

```
# 定义宏，用于标识版本信息
/* $Id$ */
# 防止头文件重复包含
#ifndef NCAT_CONNECT_H
#define NCAT_CONNECT_H
# 包含 nsock.h 头文件
#include "nsock.h"
# 声明 ncat_connect 函数，用于处理基于 nsock 的连接
extern int ncat_connect(void);
# 结束条件编译指令
#endif
```