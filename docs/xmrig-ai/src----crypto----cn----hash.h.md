# `xmrig\src\crypto\cn\hash.h`

```cpp
# 防止头文件被重复包含
#pragma once

# 定义无符号字符类型 BitSequence
typedef unsigned char BitSequence;
# 定义无符号长长整型数据类型 DataLength
typedef unsigned long long DataLength;
# 定义枚举类型 HashReturn，包括 SUCCESS、FAIL、BAD_HASHLEN 三种可能取值
typedef enum {SUCCESS = 0, FAIL = 1, BAD_HASHLEN = 2} HashReturn;
```