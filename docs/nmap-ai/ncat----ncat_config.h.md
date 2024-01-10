# `nmap\ncat\ncat_config.h`

```
# 如果存在 config.h 文件，则包含 config.h 文件
#include "config.h"
# 如果不存在 config.h 文件，但是 WIN32 宏定义存在，则包含 config_win.h 文件
#elif WIN32
#include "config_win.h"
# 如果既不存在 config.h 文件，也不存在 WIN32 宏定义，则抛出错误信息
#else
#error "No config.h, and not WIN32"
# 结束条件编译指令
#endif
```