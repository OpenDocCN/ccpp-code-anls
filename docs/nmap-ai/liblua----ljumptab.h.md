# `nmap\liblua\ljumptab.h`

```cpp
// 取消宏定义 vmdispatch
#undef vmdispatch
// 取消宏定义 vmcase
#undef vmcase
// 取消宏定义 vmbreak
#undef vmbreak

// 定义宏 vmdispatch，跳转到 disptab[x] 对应的位置
#define vmdispatch(x)     goto *disptab[x];

// 定义宏 vmcase，创建标签 L_##l
#define vmcase(l)     L_##l:

// 定义宏 vmbreak，执行 vmfetch() 后跳转到 GET_OPCODE(i) 对应的位置
#define vmbreak        vmfetch(); vmdispatch(GET_OPCODE(i));

// 创建包含指令对应位置的跳转表
static const void *const disptab[NUM_OPCODES] = {
    // ...（省略部分内容）
    &&L_OP_VARARG,
    &&L_OP_VARARGPREP,
    &&L_OP_EXTRAARG
};
```