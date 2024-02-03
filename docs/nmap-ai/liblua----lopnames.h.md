# `nmap\liblua\lopnames.h`

```cpp
/*
** $Id: lopnames.h $
** Opcode names
** See Copyright Notice in lua.h
*/

#if !defined(lopnames_h)
#define lopnames_h

#include <stddef.h>


/* ORDER OP */

// 定义包含所有操作码名称的静态常量字符串数组
static const char *const opnames[] = {
  "MOVE",         // 移动操作
  "LOADI",        // 加载整数操作
  "LOADF",        // 加载浮点数操作
  "LOADK",        // 加载常量操作
  "LOADKX",       // 加载常量扩展操作
  "LOADFALSE",    // 加载假值操作
  "LFALSESKIP",   // 跳过假值操作
  "LOADTRUE",     // 加载真值操作
  "LOADNIL",      // 加载空值操作
  "GETUPVAL",     // 获取上值操作
  "SETUPVAL",     // 设置上值操作
  "GETTABUP",     // 获取上表操作
  "GETTABLE",     // 获取表操作
  "GETI",         // 获取索引操作
  "GETFIELD",     // 获取字段操作
  "SETTABUP",     // 设置上表操作
  "SETTABLE",     // 设置表操作
  "SETI",         // 设置索引操作
  "SETFIELD",     // 设置字段操作
  "NEWTABLE",     // 新建表操作
  "SELF",         // 自身操作
  "ADDI",         // 整数加法操作
  "ADDK",         // 常量加法操作
  "SUBK",         // 常量减法操作
  "MULK",         // 常量乘法操作
  "MODK",         // 常量取模操作
  "POWK",         // 常量幂运算操作
  "DIVK",         // 常量除法操作
  "IDIVK",        // 常量整除操作
  "BANDK",        // 常量按位与操作
  "BORK",         // 常量按位或操作
  "BXORK",        // 常量按位异或操作
  "SHRI",         // 右移操作
  "SHLI",         // 左移操作
  "ADD",          // 加法操作
  "SUB",          // 减法操作
  "MUL",          // 乘法操作
  "MOD",          // 取模操作
  "POW",          // 幂运算操作
  "DIV",          // 除法操作
  "IDIV",         // 整除操作
  "BAND",         // 按位与操作
  "BOR",          // 按位或操作
  "BXOR",         // 按位异或操作
  "SHL",          // 左移操作
  "SHR",          // 右移操作
  "MMBIN",        // 二元元方法操作
  "MMBINI",       // 二元元方法整数操作
  "MMBINK",       // 二元元方法常量操作
  "UNM",          // 取负操作
  "BNOT",         // 按位取反操作
  "NOT",          // 逻辑非操作
  "LEN",          // 长度操作
  "CONCAT",       // 连接操作
  "CLOSE",        // 关闭操作
  "TBC",          // 尾调用操作
  "JMP",          // 跳转操作
  "EQ",           // 等于操作
  "LT",           // 小于操作
  "LE",           // 小于等于操作
  "EQK",          // 常量等于操作
  "EQI",          // 整数等于操作
  "LTI",          // 整数小于操作
  "LEI",          // 整数小于等于操作
  "GTI",          // 整数大于操作
  "GEI",          // 整数大于等于操作
  "TEST",         // 测试操作
  "TESTSET",      // 测试并设置操作
  "CALL",         // 调用操作
  "TAILCALL",     // 尾调用操作
  "RETURN",       // 返回操作
  "RETURN0",      // 返回0操作
  "RETURN1",      // 返回1操作
  "FORLOOP",      // 循环操作
  "FORPREP",      // 准备循环操作
  "TFORPREP",     // 泛型for循环准备操作
  "TFORCALL",     // 泛型for循环调用操作
  "TFORLOOP",     // 泛型for循环操作
  "SETLIST",      // 设置列表操作
  "CLOSURE",      // 闭包操作
  "VARARG",       // 可变参数操作
  "VARARGPREP",   // 可变参数准备操作
  "EXTRAARG",     // 额外参数操作
  NULL            // 结束标记
};

#endif
```