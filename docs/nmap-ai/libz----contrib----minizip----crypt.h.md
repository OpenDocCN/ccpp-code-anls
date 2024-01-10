# `nmap\libz\contrib\minizip\crypt.h`

```
/* crypt.h -- base code for crypt/uncrypt ZIPfile
   版本信息和版权声明
   Version 1.01e, February 12th, 2005
   版权声明
   Copyright (C) 1998-2005 Gilles Vollant
   代码来源说明
   This code is a modified version of crypting code in Infozip distribution
   密码部分的代码来源说明
   The encryption/decryption parts of this source code (as opposed to the
   non-echoing password parts) were originally written in Europe.  The
   whole source package can be freely distributed, including from the USA.
   (Prior to January 2000, re-export from the US was a violation of US law.)
   加密/解密部分的代码来源说明
   This encryption code is a direct transcription of the algorithm from
   Roger Schlafly, described by Phil Katz in the file appnote.txt.  This
   file (appnote.txt) is distributed with the PKZIP program (even in the
   version without encryption capabilities).
   加密算法来源说明
   If you don't need crypting in your application, just define symbols
   NOCRYPT and NOUNCRYPT.
   如果不需要加密，可以定义 NOCRYPT 和 NOUNCRYPT 符号
   This code support the "Traditional PKWARE Encryption".
   代码支持传统的 PKWARE 加密
   The new AES encryption added on Zip format by Winzip (see the page
   http://www.winzip.com/aes_info.htm ) and PKWare PKZip 5.x Strong
   Encryption is not supported.
   不支持 Winzip 添加的新的 AES 加密和 PKWare PKZip 5.x 强加密
*/

#define CRC32(c, b) ((*(pcrc_32_tab+(((int)(c) ^ (b)) & 0xff))) ^ ((c) >> 8))
CRC32 宏定义

/***********************************************************************
 * Return the next byte in the pseudo-random sequence
 */
static int decrypt_byte(unsigned long* pkeys, const z_crc_t* pcrc_32_tab)
{
    unsigned temp;  /* POTENTIAL BUG:  temp*(temp^1) may overflow in an
                     * unpredictable manner on 16-bit systems; not a problem
                     * with any known compiler so far, though */
    返回伪随机序列中的下一个字节
    (void)pcrc_32_tab;
    temp = ((unsigned)(*(pkeys+2)) & 0xffff) | 2;
    return (int)(((temp * (temp ^ 1)) >> 8) & 0xff);
}

/***********************************************************************
 * Update the encryption keys with the next byte of plain text
 */
static int update_keys(unsigned long* pkeys,const z_crc_t* pcrc_32_tab,int c)
{
    (*(pkeys+0)) = CRC32((*(pkeys+0)), c);
    更新加密密钥，使用下一个明文字节
    # 将指针 pkeys 指向的值加上指针 pkeys 指向的值与 0xff 的按位与结果
    (*(pkeys+1)) += (*(pkeys+0)) & 0xff;
    # 将指针 pkeys 指向的值乘以 134775813L 再加上 1
    (*(pkeys+1)) = (*(pkeys+1)) * 134775813L + 1;
    # 定义一个局部变量 keyshift，其值为指针 pkeys 指向的值右移 24 位后的结果
    {
      register int keyshift = (int)((*(pkeys+1)) >> 24);
      # 将指针 pkeys 指向的值作为参数传入 CRC32 函数，并将返回值赋给指针 pkeys+2 指向的位置
      (*(pkeys+2)) = CRC32((*(pkeys+2)), keyshift);
    }
    # 返回变量 c 的值
    return c;
}
/***********************************************************************
 * 根据给定的密码初始化加密密钥和随机头部
 */
static void init_keys(const char* passwd,unsigned long* pkeys,const z_crc_t* pcrc_32_tab)
{
    // 初始化加密密钥
    *(pkeys+0) = 305419896L;
    *(pkeys+1) = 591751049L;
    *(pkeys+2) = 878082192L;
    // 遍历密码，更新密钥
    while (*passwd != '\0') {
        update_keys(pkeys,pcrc_32_tab,(int)*passwd);
        passwd++;
    }
}

// 解密函数
#define zdecode(pkeys,pcrc_32_tab,c) \
    (update_keys(pkeys,pcrc_32_tab,c ^= decrypt_byte(pkeys,pcrc_32_tab)))

// 加密函数
#define zencode(pkeys,pcrc_32_tab,c,t) \
    (t=decrypt_byte(pkeys,pcrc_32_tab), update_keys(pkeys,pcrc_32_tab,c), (Byte)t^(c))

#ifdef INCLUDECRYPTINGCODE_IFCRYPTALLOWED

// 随机头部长度
#define RAND_HEAD_LEN  12
   /* "last resort" source for second part of crypt seed pattern */
#  ifndef ZCR_SEED2
#    define ZCR_SEED2 3141592654UL      /* use PI as default pattern */
#  endif

// 加密头部
static unsigned crypthead(const char* passwd,       /* password string */
                          unsigned char* buf,       /* where to write header */
                          int bufSize,
                          unsigned long* pkeys,
                          const z_crc_t* pcrc_32_tab,
                          unsigned long crcForCrypting)
{
    unsigned n;                  /* index in random header */
    int t;                       /* temporary */
    int c;                       /* random byte */
    unsigned char header[RAND_HEAD_LEN-2]; /* random header */
    static unsigned calls = 0;   /* ensure different random header each time */

    // 如果缓冲区大小小于随机头部长度，则返回
    if (bufSize<RAND_HEAD_LEN)
      return 0;

    /* First generate RAND_HEAD_LEN-2 random bytes. We encrypt the
     * output of rand() to get less predictability, since rand() is
     * often poorly implemented.
     */
    // 第一次调用，生成随机头部
    if (++calls == 1)
    {
        srand((unsigned)(time(NULL) ^ ZCR_SEED2));
    }
    // 初始化密钥
    init_keys(passwd, pkeys, pcrc_32_tab);
    // 生成随机头部
    for (n = 0; n < RAND_HEAD_LEN-2; n++)
    {
        # 生成一个随机数，右移 7 位并取低 8 位，得到一个 0 到 255 之间的随机数
        c = (rand() >> 7) & 0xff;
        # 使用 zencode 函数对随机数进行加密，并将结果存入 header 数组
        header[n] = (unsigned char)zencode(pkeys, pcrc_32_tab, c, t);
    }
    /* Encrypt random header (last two bytes is high word of crc) */
    # 使用密码初始化密钥
    init_keys(passwd, pkeys, pcrc_32_tab);
    # 遍历 header 数组，对每个元素进行加密，并存入 buf 数组
    for (n = 0; n < RAND_HEAD_LEN-2; n++)
    {
        buf[n] = (unsigned char)zencode(pkeys, pcrc_32_tab, header[n], t);
    }
    # 将 crcForCrypting 的高 16 位进行加密，并存入 buf 数组
    buf[n++] = (unsigned char)zencode(pkeys, pcrc_32_tab, (int)(crcForCrypting >> 16) & 0xff, t);
    # 将 crcForCrypting 的高 24 位进行加密，并存入 buf 数组
    buf[n++] = (unsigned char)zencode(pkeys, pcrc_32_tab, (int)(crcForCrypting >> 24) & 0xff, t);
    # 返回加密后的数据长度
    return n;
}
# 结束一个代码块或函数的定义
#endif
# 结束一个条件编译指令的定义
```