# `xmrig\src\base\tools\cryptonote\crypto-ops.h`

```cpp
// 版权声明，版权所有，禁止未经许可的再发布和使用
// 1. 源代码的再发布必须保留版权声明、条件列表和免责声明
// 2. 二进制形式的再发布必须在文档和/或其他提供的材料中复制版权声明、条件列表和免责声明
// 3. 未经特定书面许可，不得使用版权持有者或贡献者的名称来认可或推广从本软件衍生的产品
// 版权持有者和贡献者提供的本软件是"按原样"提供的，不提供任何明示或暗示的担保，包括但不限于适销性和特定用途的暗示担保。在任何情况下，版权持有者或贡献者均不对任何直接、间接、偶发、特殊、惩罚性或后果性损害（包括但不限于替代商品或服务的采购、使用、数据或利润的损失或业务中断）负责，无论是在合同、严格责任或侵权行为（包括疏忽或其他方式）的任何理论下，即使已被告知可能发生此类损害的可能性。
// 本文件的部分内容最初版权归Cryptonote开发人员所有

// 防止头文件重复包含
#pragma once

/* 从 fe.h 头文件中引入类型定义 */
typedef int32_t fe[10];

/* 从 ge.h 头文件中引入结构体定义 */
typedef struct {
  fe X;
  fe Y;
  fe Z;
} ge_p2;

typedef struct {
  fe X;
  fe Y;
  fe Z;
  fe T;
} ge_p3;

typedef struct {
  fe X;
  fe Y;
  fe Z;
  fe T;
} ge_p1p1;

typedef struct {
  fe yplusx;
  fe yminusx;
  fe xy2d;
} ge_precomp;
/* 定义一个结构体，包含四个 fe 类型的字段 */
typedef struct {
  fe YplusX;
  fe YminusX;
  fe Z;
  fe T2d;
} ge_cached;

/* 从 ge_add.c 文件中引入函数 ge_add，该函数接受一个 ge_p1p1 类型的指针，一个 ge_p3 类型的指针和一个 ge_cached 类型的指针作为参数 */
void ge_add(ge_p1p1 *, const ge_p3 *, const ge_cached *);

/* 从 ge_double_scalarmult.c 文件中引入 ge_dsmp 类型的数组 ge_Bi，以及三个函数 ge_dsm_precomp、ge_double_scalarmult_base_vartime 和 ge_triple_scalarmult_base_vartime，它们分别接受 ge_dsmp 类型的指针、unsigned char 类型的指针、ge_p3 类型的指针和 ge_dsmp 类型的参数，以及其他参数 */
typedef ge_cached ge_dsmp[8];
extern const ge_precomp ge_Bi[8];
void ge_dsm_precomp(ge_dsmp r, const ge_p3 *s);
void ge_double_scalarmult_base_vartime(ge_p2 *, const unsigned char *, const ge_p3 *, const unsigned char *);
void ge_triple_scalarmult_base_vartime(ge_p2 *, const unsigned char *, const unsigned char *, const ge_dsmp, const unsigned char *, const ge_dsmp);
void ge_double_scalarmult_base_vartime_p3(ge_p3 *, const unsigned char *, const ge_p3 *, const unsigned char *);

/* 从 ge_frombytes.c 文件中引入 fe 类型的常量 fe_sqrtm1 和 fe_d，以及函数 ge_frombytes_vartime，该函数接受一个 ge_p3 类型的指针和一个 unsigned char 类型的指针作为参数 */
extern const fe fe_sqrtm1;
extern const fe fe_d;
int ge_frombytes_vartime(ge_p3 *, const unsigned char *);

/* 从 ge_p1p1_to_p2.c 文件中引入函数 ge_p1p1_to_p2，该函数接受一个 ge_p2 类型的指针和一个 ge_p1p1 类型的指针作为参数 */
void ge_p1p1_to_p2(ge_p2 *, const ge_p1p1 *);

/* 从 ge_p1p1_to_p3.c 文件中引入函数 ge_p1p1_to_p3，该函数接受一个 ge_p3 类型的指针和一个 ge_p1p1 类型的指针作为参数 */
void ge_p1p1_to_p3(ge_p3 *, const ge_p1p1 *);

/* 从 ge_p2_dbl.c 文件中引入函数 ge_p2_dbl，该函数接受一个 ge_p1p1 类型的指针和一个 ge_p2 类型的指针作为参数 */
void ge_p2_dbl(ge_p1p1 *, const ge_p2 *);

/* 从 ge_p3_to_cached.c 文件中引入 fe 类型的常量 fe_d2，以及函数 ge_p3_to_cached，该函数接受一个 ge_cached 类型的指针和一个 ge_p3 类型的指针作为参数 */
extern const fe fe_d2;
void ge_p3_to_cached(ge_cached *, const ge_p3 *);

/* 从 ge_p3_to_p2.c 文件中引入函数 ge_p3_to_p2，该函数接受一个 ge_p2 类型的指针和一个 ge_p3 类型的指针作为参数 */
void ge_p3_to_p2(ge_p2 *, const ge_p3 *);

/* 从 ge_p3_tobytes.c 文件中引入函数 ge_p3_tobytes，该函数接受一个 unsigned char 类型的指针和一个 ge_p3 类型的指针作为参数 */
void ge_p3_tobytes(unsigned char *, const ge_p3 *);

/* 从 ge_scalarmult_base.c 文件中引入 ge_precomp 类型的数组 ge_base，以及函数 ge_scalarmult_base，该函数接受一个 ge_p3 类型的指针和一个 unsigned char 类型的指针作为参数 */
extern const ge_precomp ge_base[32][8];
void ge_scalarmult_base(ge_p3 *, const unsigned char *);

/* 从 ge_tobytes.c 文件中引入函数 ge_tobytes，该函数接受一个 unsigned char 类型的指针和一个 ge_p2 类型的指针作为参数 */
void ge_tobytes(unsigned char *, const ge_p2 *);

/* 从 sc_reduce.c 文件中引入函数 sc_reduce，该函数接受一个 unsigned char 类型的指针作为参数 */
void sc_reduce(unsigned char *);

/* 新代码 */

/* 定义一个函数 ge_scalarmult，该函数接受一个 ge_p2 类型的指针，一个 unsigned char 类型的指针和一个 ge_p3 类型的指针作为参数 */
void ge_scalarmult(ge_p2 *, const unsigned char *, const ge_p3 *);
/* 定义一个函数 ge_scalarmult_p3，该函数接受一个 ge_p3 类型的指针，一个 unsigned char 类型的指针和一个 ge_p3 类型的指针作为参数 */
void ge_scalarmult_p3(ge_p3 *, const unsigned char *, const ge_p3 *);
/* 定义一个函数 ge_double_scalarmult_precomp_vartime，该函数接受一个 ge_p2 类型的指针，两个 unsigned char 类型的指针，一个 ge_p3 类型的指针和一个 unsigned char 类型的指针作为参数 */
void ge_double_scalarmult_precomp_vartime(ge_p2 *, const unsigned char *, const ge_p3 *, const unsigned char *, const ge_dsmp);
// 用于执行变时间的三倍点乘法预计算
void ge_triple_scalarmult_precomp_vartime(ge_p2 *, const unsigned char *, const ge_dsmp, const unsigned char *, const ge_dsmp, const unsigned char *, const ge_dsmp);

// 用于执行变时间的双倍点乘法预计算
void ge_double_scalarmult_precomp_vartime2(ge_p2 *, const unsigned char *, const ge_dsmp, const unsigned char *, const ge_dsmp);

// 用于执行变时间的双倍点乘法预计算，返回结果为 P3 类型
void ge_double_scalarmult_precomp_vartime2_p3(ge_p3 *, const unsigned char *, const ge_dsmp, const unsigned char *, const ge_dsmp);

// 将 ge_p2 类型的点乘以 8
void ge_mul8(ge_p1p1 *, const ge_p2 *);

// 定义 fe 类型的常量
extern const fe fe_ma2;
extern const fe fe_ma;
extern const fe fe_fffb1;
extern const fe fe_fffb2;
extern const fe fe_fffb3;
extern const fe fe_fffb4;

// 定义 ge_p3 类型的常量
extern const ge_p3 ge_p3_identity;
extern const ge_p3 ge_p3_H;

// 从字节数组解析出 ge_p2 类型的点，执行变时间的操作
void ge_fromfe_frombytes_vartime(ge_p2 *, const unsigned char *);

// 将字节数组置零
void sc_0(unsigned char *);

// 将字节数组模 2^256
void sc_reduce32(unsigned char *);

// 将两个字节数组相加
void sc_add(unsigned char *, const unsigned char *, const unsigned char *);

// 将两个字节数组相减
void sc_sub(unsigned char *, const unsigned char *, const unsigned char *);

// 执行字节数组的乘减操作
void sc_mulsub(unsigned char *, const unsigned char *, const unsigned char *, const unsigned char *);

// 执行字节数组的乘法
void sc_mul(unsigned char *, const unsigned char *, const unsigned char *);

// 执行字节数组的乘加操作
void sc_muladd(unsigned char *s, const unsigned char *a, const unsigned char *b, const unsigned char *c);

// 检查字节数组是否为零
int sc_check(const unsigned char *);

// 检查字节数组是否非零，不进行规范化
int sc_isnonzero(const unsigned char *);

// 内部函数，从字节数组加载 3 个字节的数据
uint64_t load_3(const unsigned char *in);

// 内部函数，从字节数组加载 4 个字节的数据
uint64_t load_4(const unsigned char *in);

// 执行 P3 类型点的减法操作
void ge_sub(ge_p1p1 *r, const ge_p3 *p, const ge_cached *q);

// 执行 fe 类型的加法操作
void fe_add(fe h, const fe f, const fe g);

// 将 fe 类型的数据转换为字节数组
void fe_tobytes(unsigned char *, const fe);

// 执行 fe 类型的求逆操作
void fe_invert(fe out, const fe z);

// 检查 P3 类型的点是否为无穷远点
int ge_p3_is_point_at_infinity(const ge_p3 *p);
```