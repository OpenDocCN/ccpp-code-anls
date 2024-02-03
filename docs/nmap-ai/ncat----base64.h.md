# `nmap\ncat\base64.h`

```cpp
# 定义条件编译指令，如果未定义 NCAT_BASE64_H，则执行下面的代码
#ifndef NCAT_BASE64_H
# 定义 NCAT_BASE64_H，避免重复包含
#define NCAT_BASE64_H
# 声明一个函数 b64enc，接受一个无符号字符指针和一个整数作为参数，返回一个字符指针
char *b64enc(const unsigned char *data, int len);
# 结束条件编译指令
#endif
```