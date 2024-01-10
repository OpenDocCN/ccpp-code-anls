# `nmap\mswin32\make-vars.h`

```
# 定义一个名为IGNORE的宏，内容为#include "../nmap.h"
define IGNORE
#include "../nmap.h"
endef

# 定义一个名为EXPORT的宏，参数为_var，将name变量的值加上_var后缀并导出
#define EXPORT(_var) export $(name)##_var:= $(patsubst "%,%,$(patsubst %",%,$(subst " ",,NMAP##_var)))

# 设置name变量的值为NMAP
name = NMAP
# 调用EXPORT宏，将_NAME加上name变量的值NMAP后缀并导出
EXPORT(_NAME)
# 调用EXPORT宏，将_VERSION加上name变量的值NMAP后缀并导出
EXPORT(_VERSION)
# 调用EXPORT宏，将_NUM_VERSION加上name变量的值NMAP后缀并导出
EXPORT(_NUM_VERSION)
# 取消NMAP_NAME的定义
#undef NMAP_NAME
# 包含../../nmap-build/nmap-oem.h文件
#include "../../nmap-build/nmap-oem.h"
# 定义NMAP_OEM_NAME为NMAP_NAME
#define NMAP_OEM_NAME NMAP_NAME
# 调用EXPORT宏，将_OEM_NAME加上name变量的值NMAP后缀并导出
EXPORT(_OEM_NAME)
```