# `xmrig\src\crypto\cn\c_groestl.c`

```
/* hash.c     April 2012
 * Groestl ANSI C code optimised for 32-bit machines
 * Author: Thomas Krinninger
 *
 *  This work is based on the implementation of
 *          Soeren S. Thomsen and Krystian Matusiewicz
 *
 *
 */

#include "c_groestl.h"  // 包含自定义的头文件 c_groestl.h
#include "groestl_tables.h"  // 包含自定义的头文件 groestl_tables.h

#define P_TYPE 0  // 定义宏 P_TYPE 为 0
#define Q_TYPE 1  // 定义宏 Q_TYPE 为 1

const uint8_t shift_Values[2][8] = {{0,1,2,3,4,5,6,7},{1,3,5,7,0,2,4,6}};  // 定义二维数组 shift_Values，初始化两个子数组

const uint8_t indices_cyclic[15] = {0,1,2,3,4,5,6,7,0,1,2,3,4,5,6};  // 定义长度为 15 的数组 indices_cyclic，初始化其中的元素

#define ROTATE_COLUMN_DOWN(v1, v2, amount_bytes, temp_var) {temp_var = (v1<<(8*amount_bytes))|(v2>>(8*(4-amount_bytes))); \  // 定义宏 ROTATE_COLUMN_DOWN，实现列的向下旋转
                                                            v2 = (v2<<(8*amount_bytes))|(v1>>(8*(4-amount_bytes))); \  // 实现列的向下旋转
                                                            v1 = temp_var;}  // 实现列的向下旋转
#define COLUMN(x,y,i,c0,c1,c2,c3,c4,c5,c6,c7,tv1,tv2,tu,tl,t)                \
   tu = T[2*(uint32_t)x[4*c0+0]];                \  # 从 T 表中获取指定位置的值，存入 tu
   tl = T[2*(uint32_t)x[4*c0+0]+1];            \  # 从 T 表中获取指定位置的值，存入 tl
   tv1 = T[2*(uint32_t)x[4*c1+1]];            \  # 从 T 表中获取指定位置的值，存入 tv1
   tv2 = T[2*(uint32_t)x[4*c1+1]+1];            \  # 从 T 表中获取指定位置的值，存入 tv2
   ROTATE_COLUMN_DOWN(tv1,tv2,1,t)    \  # 调用 ROTATE_COLUMN_DOWN 函数，对 tv1 和 tv2 进行向下旋转操作
   tu ^= tv1;                        \  # 对 tu 进行异或操作
   tl ^= tv2;                        \  # 对 tl 进行异或操作
   tv1 = T[2*(uint32_t)x[4*c2+2]];            \  # 从 T 表中获取指定位置的值，存入 tv1
   tv2 = T[2*(uint32_t)x[4*c2+2]+1];            \  # 从 T 表中获取指定位置的值，存入 tv2
   ROTATE_COLUMN_DOWN(tv1,tv2,2,t)    \  # 调用 ROTATE_COLUMN_DOWN 函数，对 tv1 和 tv2 进行向下旋转操作
   tu ^= tv1;                        \  # 对 tu 进行异或操作
   tl ^= tv2;                       \  # 对 tl 进行异或操作
   tv1 = T[2*(uint32_t)x[4*c3+3]];            \  # 从 T 表中获取指定位置的值，存入 tv1
   tv2 = T[2*(uint32_t)x[4*c3+3]+1];            \  # 从 T
// 定义一个静态函数，用于计算 RND512P 的一轮操作
static void RND512P(uint8_t *x, uint32_t *y, uint32_t r) {
  uint32_t temp_v1, temp_v2, temp_upper_value, temp_lower_value, temp;
  uint32_t* x32 = (uint32_t*)x;
  // 对 x32 数组中的元素进行异或操作
  x32[ 0] ^= 0x00000000^r;
  x32[ 2] ^= 0x00000010^r;
  x32[ 4] ^= 0x00000020^r;
  x32[ 6] ^= 0x00000030^r;
  x32[ 8] ^= 0x00000040^r;
  x32[10] ^= 0x00000050^r;
  x32[12] ^= 0x00000060^r;
  x32[14] ^= 0x00000070^r;
  // 调用 COLUMN 函数进行一系列操作
  COLUMN(x,y, 0,  0,  2,  4,  6,  9, 11, 13, 15, temp_v1, temp_v2, temp_upper_value, temp_lower_value, temp);
  COLUMN(x,y, 2,  2,  4,  6,  8, 11, 13, 15,  1, temp_v1, temp_v2, temp_upper_value, temp_lower_value, temp);
  COLUMN(x,y, 4,  4,  6,  8, 10, 13, 15,  1,  3, temp_v1, temp_v2, temp_upper_value, temp_lower_value, temp);
  COLUMN(x,y, 6,  6,  8, 10, 12, 15,  1,  3,  5, temp_v1, temp_v2, temp_upper_value, temp_lower_value, temp);
  COLUMN(x,y, 8,  8, 10, 12, 14,  1,  3,  5,  7, temp_v1, temp_v2, temp_upper_value, temp_lower_value, temp);
  COLUMN(x,y,10, 10, 12, 14,  0,  3,  5,  7,  9, temp_v1, temp_v2, temp_upper_value, temp_lower_value, temp);
  COLUMN(x,y,12, 12, 14,  0,  2,  5,  7,  9, 11, temp_v1, temp_v2, temp_upper_value, temp_lower_value, temp);
  COLUMN(x,y,14, 14,  0,  2,  4,  7,  9, 11, 13, temp_v1, temp_v2, temp_upper_value, temp_lower_value, temp);
}
static void RND512Q(uint8_t *x, uint32_t *y, uint32_t r) {
  uint32_t temp_v1, temp_v2, temp_upper_value, temp_lower_value, temp;
  uint32_t* x32 = (uint32_t*)x;
  x32[ 0] = ~x32[ 0];  # 对x32[0]按位取反
  x32[ 1] ^= 0xffffffff^r;  # 对x32[1]进行异或操作
  x32[ 2] = ~x32[ 2];  # 对x32[2]按位取反
  x32[ 3] ^= 0xefffffff^r;  # 对x32[3]进行异或操作
  x32[ 4] = ~x32[ 4];  # 对x32[4]按位取反
  x32[ 5] ^= 0xdfffffff^r;  # 对x32[5]进行异或操作
  x32[ 6] = ~x32[ 6];  # 对x32[6]按位取反
  x32[ 7] ^= 0xcfffffff^r;  # 对x32[7]进行异或操作
  x32[ 8] = ~x32[ 8];  # 对x32[8]按位取反
  x32[ 9] ^= 0xbfffffff^r;  # 对x32[9]进行异或操作
  x32[10] = ~x32[10];  # 对x32[10]按位取反
  x32[11] ^= 0xafffffff^r;  # 对x32[11]进行异或操作
  x32[12] = ~x32[12];  # 对x32[12]按位取反
  x32[13] ^= 0x9fffffff^r;  # 对x32[13]进行异或操作
  x32[14] = ~x32[14];  # 对x32[14]按位取反
  x32[15] ^= 0x8fffffff^r;  # 对x32[15]进行异或操作
  COLUMN(x,y, 0,  2,  6, 10, 14,  1,  5,  9, 13, temp_v1, temp_v2, temp_upper_value, temp_lower_value, temp);  # 调用COLUMN函数
  COLUMN(x,y, 2,  4,  8, 12,  0,  3,  7, 11, 15, temp_v1, temp_v2, temp_upper_value, temp_lower_value, temp);  # 调用COLUMN函数
  COLUMN(x,y, 4,  6, 10, 14,  2,  5,  9, 13,  1, temp_v1, temp_v2, temp_upper_value, temp_lower_value, temp);  # 调用COLUMN函数
  COLUMN(x,y, 6,  8, 12,  0,  4,  7, 11, 15,  3, temp_v1, temp_v2, temp_upper_value, temp_lower_value, temp);  # 调用COLUMN函数
  COLUMN(x,y, 8, 10, 14,  2,  6,  9, 13,  1,  5, temp_v1, temp_v2, temp_upper_value, temp_lower_value, temp);  # 调用COLUMN函数
  COLUMN(x,y,10, 12,  0,  4,  8, 11, 15,  3,  7, temp_v1, temp_v2, temp_upper_value, temp_lower_value, temp);  # 调用COLUMN函数
  COLUMN(x,y,12, 14,  2,  6, 10, 13,  1,  5,  9, temp_v1, temp_v2, temp_upper_value, temp_lower_value, temp);  # 调用COLUMN函数
  COLUMN(x,y,14,  0,  4,  8, 12, 15,  3,  7, 11, temp_v1, temp_v2, temp_upper_value, temp_lower_value, temp);  # 调用COLUMN函数
}

/* compute compression function (short variants) */
static void F512(uint32_t *h, const uint32_t *m) {
  int i;
  uint32_t Ptmp[2*COLS512];
  uint32_t Qtmp[2*COLS512];
  uint32_t y[2*COLS512];
  uint32_t z[2*COLS512];

  for (i = 0; i < 2*COLS512; i++) {
    z[i] = m[i];  # 将m数组的值赋给z数组
  /* 计算 Ptmp[i] = h[i]^m[i] */
  Ptmp[i] = h[i]^m[i];
  }

  /* 计算 Q(m) */
  RND512Q((uint8_t*)z, y, 0x00000000);
  RND512Q((uint8_t*)y, z, 0x01000000);
  RND512Q((uint8_t*)z, y, 0x02000000);
  RND512Q((uint8_t*)y, z, 0x03000000);
  RND512Q((uint8_t*)z, y, 0x04000000);
  RND512Q((uint8_t*)y, z, 0x05000000);
  RND512Q((uint8_t*)z, y, 0x06000000);
  RND512Q((uint8_t*)y, z, 0x07000000);
  RND512Q((uint8_t*)z, y, 0x08000000);
  RND512Q((uint8_t*)y, Qtmp, 0x09000000);

  /* 计算 P(h+m) */
  RND512P((uint8_t*)Ptmp, y, 0x00000000);
  RND512P((uint8_t*)y, z, 0x00000001);
  RND512P((uint8_t*)z, y, 0x00000002);
  RND512P((uint8_t*)y, z, 0x00000003);
  RND512P((uint8_t*)z, y, 0x00000004);
  RND512P((uint8_t*)y, z, 0x00000005);
  RND512P((uint8_t*)z, y, 0x00000006);
  RND512P((uint8_t*)y, z, 0x00000007);
  RND512P((uint8_t*)z, y, 0x00000008);
  RND512P((uint8_t*)y, Ptmp, 0x00000009);

  /* 计算 P(h+m) + Q(m) + h */
  for (i = 0; i < 2*COLS512; i++) {
    h[i] ^= Ptmp[i]^Qtmp[i];
  }
/* digest up to msglen bytes of input (full blocks only) */
static void Transform(groestlHashState *ctx,
           const uint8_t *input,
           int msglen) {

  /* digest message, one block at a time */
  for (; msglen >= SIZE512;
       msglen -= SIZE512, input += SIZE512) {
    F512(ctx->chaining,(uint32_t*)input);

    /* increment block counter */
    ctx->block_counter1++;
    if (ctx->block_counter1 == 0) ctx->block_counter2++;
  }
}

/* given state h, do h <- P(h)+h */
static void OutputTransformation(groestlHashState *ctx) {
  int j;
  uint32_t temp[2*COLS512];
  uint32_t y[2*COLS512];
  uint32_t z[2*COLS512];

  // 复制当前状态到临时数组
  for (j = 0; j < 2*COLS512; j++) {
    temp[j] = ctx->chaining[j];
  }
  // 进行一系列的置换操作
  RND512P((uint8_t*)temp, y, 0x00000000);
  RND512P((uint8_t*)y, z, 0x00000001);
  RND512P((uint8_t*)z, y, 0x00000002);
  RND512P((uint8_t*)y, z, 0x00000003);
  RND512P((uint8_t*)z, y, 0x00000004);
  RND512P((uint8_t*)y, z, 0x00000005);
  RND512P((uint8_t*)z, y, 0x00000006);
  RND512P((uint8_t*)y, z, 0x00000007);
  RND512P((uint8_t*)z, y, 0x00000008);
  RND512P((uint8_t*)y, temp, 0x00000009);
  // 将临时数组的值与当前状态进行异或操作
  for (j = 0; j < 2*COLS512; j++) {
    ctx->chaining[j] ^= temp[j];
  }                                    
}

/* initialise context */
static void Init(groestlHashState* ctx) {
  int i = 0;
  /* allocate memory for state and data buffer */

  // 初始化状态数组
  for(;i<(SIZE512/sizeof(uint32_t));i++)
  {
    ctx->chaining[i] = 0;
  }

  /* set initial value */
  // 设置初始值
  ctx->chaining[2*COLS512-1] = u32BIG((uint32_t)HASH_BIT_LEN);

  /* set other variables */
  // 设置其他变量的初始值
  ctx->buf_ptr = 0;
  ctx->block_counter1 = 0;
  ctx->block_counter2 = 0;
  ctx->bits_in_last_byte = 0;
}

/* update state with databitlen bits of input */
static void Update(groestlHashState* ctx,
          const BitSequence* input,
          DataLength databitlen) {
  int index = 0;  // 初始化索引为0
  int msglen = (int)(databitlen/8);  // 计算消息的字节数
  int rem = (int)(databitlen%8);  // 计算消息的余数

  /* if the buffer contains data that has not yet been digested, first
     add data to buffer until full */
  if (ctx->buf_ptr) {  // 如果缓冲区中包含尚未被消化的数据
    while (ctx->buf_ptr < SIZE512 && index < msglen) {  // 当缓冲区未满且消息未处理完时
      ctx->buffer[(int)ctx->buf_ptr++] = input[index++];  // 将输入数据添加到缓冲区
    }
    if (ctx->buf_ptr < SIZE512) {  // 如果缓冲区仍未满
      /* buffer still not full, return */
      if (rem) {  // 如果还有余数
    ctx->bits_in_last_byte = rem;  // 更新最后一个字节的位数
    ctx->buffer[(int)ctx->buf_ptr++] = input[index];  // 将余数添加到缓冲区
      }
      return;  // 返回
    }

    /* digest buffer */
    ctx->buf_ptr = 0;  // 重置缓冲区指针
    Transform(ctx, ctx->buffer, SIZE512);  // 处理缓冲区数据
  }

  /* digest bulk of message */
  Transform(ctx, input+index, msglen-index);  // 处理剩余的消息数据
  index += ((msglen-index)/SIZE512)*SIZE512;  // 更新索引

  /* store remaining data in buffer */
  while (index < msglen) {  // 当索引小于消息长度时
    ctx->buffer[(int)ctx->buf_ptr++] = input[index++];  // 将剩余的数据存储到缓冲区
  }

  /* if non-integral number of bytes have been supplied, store
     remaining bits in last byte, together with information about
     number of bits */
  if (rem) {  // 如果还有余数
    ctx->bits_in_last_byte = rem;  // 更新最后一个字节的位数
    ctx->buffer[(int)ctx->buf_ptr++] = input[index];  // 将余数添加到缓冲区
  }
}

#define BILB ctx->bits_in_last_byte  // 定义宏 BILB 为最后一个字节的位数

/* finalise: process remaining data (including padding), perform
   output transformation, and write hash result to 'output' */
static void Final(groestlHashState* ctx,
         BitSequence* output) {
  int i, j = 0, hashbytelen = HASH_BIT_LEN/8;  // 初始化变量

  uint8_t *s = (BitSequence*)ctx->chaining;  // 定义指针变量 s 指向 ctx->chaining

  /* pad with '1'-bit and first few '0'-bits */
  if (BILB) {  // 如果最后一个字节还有位数
    ctx->buffer[(int)ctx->buf_ptr-1] &= ((1<<BILB)-1)<<(8-BILB);  // 对最后一个字节进行位操作
    ctx->buffer[(int)ctx->buf_ptr-1] ^= 0x1<<(7-BILB);  // 对最后一个字节进行位操作
    BILB = 0;  // 重置最后一个字节的位数
  }
  else ctx->buffer[(int)ctx->buf_ptr++] = 0x80;  // 否则在缓冲区中添加 0x80

  /* pad with '0'-bits */
  if (ctx->buf_ptr > SIZE512-LENGTHFIELDLEN) {  // 如果缓冲区大于等于 SIZE512-LENGTHFIELDLEN
    /* padding requires two blocks */
    // 在缓冲区指针小于512时，将缓冲区中的值设为0
    while (ctx->buf_ptr < SIZE512) {
      ctx->buffer[(int)ctx->buf_ptr++] = 0;
    }
    /* 对第一个填充块进行摘要 */
    Transform(ctx, ctx->buffer, SIZE512);
    // 重置缓冲区指针
    ctx->buf_ptr = 0;
  }
  // 在缓冲区指针小于512-LENGTHFIELDLEN时，将缓冲区中的值设为0
  while (ctx->buf_ptr < SIZE512-LENGTHFIELDLEN) {
    ctx->buffer[(int)ctx->buf_ptr++] = 0;
  }

  /* 长度填充 */
  ctx->block_counter1++;
  // 如果block_counter1等于0，则block_counter2加1
  if (ctx->block_counter1 == 0) ctx->block_counter2++;
  // 重置缓冲区指针为512
  ctx->buf_ptr = SIZE512;

  // 在缓冲区指针大于512-(int)sizeof(uint32_t)时，将缓冲区中的值设为block_counter1的低8位
  while (ctx->buf_ptr > SIZE512-(int)sizeof(uint32_t)) {
    ctx->buffer[(int)--ctx->buf_ptr] = (uint8_t)ctx->block_counter1;
    ctx->block_counter1 >>= 8;
  }
  // 在缓冲区指针大于512-LENGTHFIELDLEN时，将缓冲区中的值设为block_counter2的低8位
  while (ctx->buf_ptr > SIZE512-LENGTHFIELDLEN) {
    ctx->buffer[(int)--ctx->buf_ptr] = (uint8_t)ctx->block_counter2;
    ctx->block_counter2 >>= 8;
  }
  /* 对最终填充块进行摘要 */
  Transform(ctx, ctx->buffer, SIZE512);
  /* 执行输出转换 */
  OutputTransformation(ctx);

  /* 将哈希结果存储在输出中 */
  for (i = SIZE512-hashbytelen; i < SIZE512; i++,j++) {
    output[j] = s[i];
  }

  /* 将相关变量清零并释放内存 */
  for (i = 0; i < COLS512; i++) {
    ctx->chaining[i] = 0;
  }
  for (i = 0; i < SIZE512; i++) {
    ctx->buffer[i] = 0;
  }
/* hash bit sequence */
// 定义一个函数，用于对比特序列进行哈希处理

void groestl(const BitSequence* data,
        DataLength databitlen,
        BitSequence* hashval) {

  groestlHashState context;

  /* initialise */
  // 初始化哈希状态
    Init(&context);


  /* process message */
  // 处理消息，对消息进行哈希计算
  Update(&context, data, databitlen);

  /* finalise */
  // 完成哈希计算，得到最终的哈希值
  Final(&context, hashval);
}
/*
static int crypto_hash(unsigned char *out,
        const unsigned char *in,
        unsigned long long len)
{
  groestl(in, 8*len, out);
  return 0;
}

*/
```