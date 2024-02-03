# `xmrig\src\backend\opencl\cl\rx\randomx_constants_graft.h`

```cpp
// 定义数据集基本大小（以字节为单位）。必须是2的幂。
#define RANDOMX_DATASET_BASE_SIZE  2147483648

// 定义数据集额外大小（以字节为单位）。必须能被64整除。
#define RANDOMX_DATASET_EXTRA_SIZE 33554368

// 定义 L3 缓存大小（以字节为单位）。必须是2的幂。
#define RANDOMX_SCRATCHPAD_L3      2097152

// 定义 L2 缓存大小（以字节为单位）。必须是2的幂，并且小于或等于 RANDOMX_SCRATCHPAD_L3。
#define RANDOMX_SCRATCHPAD_L2      262144

// 定义 L1 缓存大小（以字节为单位）。必须是2的幂（最小为64），并且小于或等于 RANDOMX_SCRATCHPAD_L2。
#define RANDOMX_SCRATCHPAD_L1      16384

// 跳转条件掩码大小（以位为单位）。
#define RANDOMX_JUMP_BITS          8

// 跳转条件掩码偏移量（以位为单位）。RANDOMX_JUMP_BITS 和 RANDOMX_JUMP_OFFSET 的总和不能超过16。
#define RANDOMX_JUMP_OFFSET        8

// 整数指令
#define RANDOMX_FREQ_IADD_RS       16
#define RANDOMX_FREQ_IADD_M         7
#define RANDOMX_FREQ_ISUB_R        16
#define RANDOMX_FREQ_ISUB_M         7
#define RANDOMX_FREQ_IMUL_R        16
#define RANDOMX_FREQ_IMUL_M         4
#define RANDOMX_FREQ_IMULH_R        4
#define RANDOMX_FREQ_IMULH_M        1
#define RANDOMX_FREQ_ISMULH_R       4
#define RANDOMX_FREQ_ISMULH_M       1
#define RANDOMX_FREQ_IMUL_RCP       8
#define RANDOMX_FREQ_INEG_R         2
// 定义随机X指令的频率参数
#define RANDOMX_FREQ_IXOR_R        15
#define RANDOMX_FREQ_IXOR_M         5
#define RANDOMX_FREQ_IROR_R         7
#define RANDOMX_FREQ_IROL_R         3
#define RANDOMX_FREQ_ISWAP_R        4

// 定义浮点指令的频率参数
#define RANDOMX_FREQ_FSWAP_R        4
#define RANDOMX_FREQ_FADD_R        16
#define RANDOMX_FREQ_FADD_M         5
#define RANDOMX_FREQ_FSUB_R        16
#define RANDOMX_FREQ_FSUB_M         5
#define RANDOMX_FREQ_FSCAL_R        6
#define RANDOMX_FREQ_FMUL_R        32
#define RANDOMX_FREQ_FDIV_M         4
#define RANDOMX_FREQ_FSQRT_R        6

// 定义控制指令的频率参数
#define RANDOMX_FREQ_CBRANCH       25
#define RANDOMX_FREQ_CFROUND        1

// 定义存储指令的频率参数
#define RANDOMX_FREQ_ISTORE        16

// 定义空操作指令的频率参数
#define RANDOMX_FREQ_NOP            0

// 定义随机X数据集项大小
#define RANDOMX_DATASET_ITEM_SIZE 64

// 定义随机X程序大小
#define RANDOMX_PROGRAM_SIZE 280

// 定义哈希大小
#define HASH_SIZE 64
// 定义熵大小
#define ENTROPY_SIZE (128 + RANDOMX_PROGRAM_SIZE * 8)
// 定义寄存器大小
#define REGISTERS_SIZE 256
// 定义立即数缓冲区大小
#define IMM_BUF_SIZE (RANDOMX_PROGRAM_SIZE * 4 - REGISTERS_SIZE)
// 定义立即数索引计数
#define IMM_INDEX_COUNT ((IMM_BUF_SIZE / 4) - 2)
// 定义虚拟机状态大小
#define VM_STATE_SIZE (REGISTERS_SIZE + IMM_BUF_SIZE + RANDOMX_PROGRAM_SIZE * 4)
// 定义舍入模式
#define ROUNDING_MODE (RANDOMX_FREQ_CFROUND ? -1 : 0)

// 定义L1/L2/L3缓存位
#define LOC_L1 (32 - 14)
#define LOC_L2 (32 - 18)
#define LOC_L3 (32 - 21)
```