# `nmap\libpcap\msdos\pktdrvr.c`

```
/*
 *  File.........: pktdrvr.c
 *
 *  Responsible..: Gisle Vanem,  giva@bgnett.no
 *
 *  Created......: 26.Sept 1995
 *
 *  Description..: Packet-driver interface for 16/32-bit C :
 *                 Borland C/C++ 3.0+ small/large model
 *                 Watcom C/C++ 11+, DOS4GW flat model
 *                 Metaware HighC 3.1+ and PharLap 386|DosX
 *                 GNU C/C++ 2.7+ and djgpp 2.x extender
 *
 *  References...: PC/TCP Packet driver Specification. rev 1.09
 *                 FTP Software Inc.
 *
 */

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <dos.h>

#include "pcap-dos.h"
#include "pcap-int.h"
#include "msdos/pktdrvr.h"

#if (DOSX)
#define NUM_RX_BUF  32      /* # of buffers in Rx FIFO queue */
#else
#define NUM_RX_BUF  10
#endif

#define DIM(x)   (sizeof((x)) / sizeof(x[0]))  // 定义一个宏，用于计算数组的元素个数
#define PUTS(s)  do {                                           \
                   if (!pktInfo.quiet)                          \
                      pktInfo.error ?                           \
                        printf ("%s: %s\n", s, pktInfo.error) : \
                        printf ("%s\n", pktInfo.error = s);     \
                 } while (0)  // 定义一个宏，用于输出错误信息

#if defined(__HIGHC__)
  extern UINT _mwenv;

#elif defined(__DJGPP__)
  #include <stddef.h>
  #include <dpmi.h>
  #include <go32.h>
  #include <pc.h>
  #include <sys/farptr.h>

#elif defined(__WATCOMC__)
  #include <i86.h>
  #include <stddef.h>
  extern char _Extender;

#else
  extern void far PktReceiver (void);
#endif
#if (DOSX & (DJGPP|DOS4GW))
  #include <sys/pack_on.h>

  // 定义 DPMI_regs 结构体，用于保存 DPMI 寄存器的信息
  struct DPMI_regs {
         DWORD  r_di;
         DWORD  r_si;
         DWORD  r_bp;
         DWORD  reserved;
         DWORD  r_bx;
         DWORD  r_dx;
         DWORD  r_cx;
         DWORD  r_ax;
         WORD   r_flags;
         WORD   r_es, r_ds, r_fs, r_gs;
         WORD   r_ip, r_cs, r_sp, r_ss;
       };

  /* Data located in a real-mode segment. This becomes far at runtime
   */
  // 定义 PktRealStub 结构体，用于保存实模式段中的数据
  typedef struct  {          /* must match data/code in pkt_rx1.s */
          WORD       _rxOutOfs;
          WORD       _rxInOfs;
          DWORD      _pktDrop;
          BYTE       _pktTemp [20];
          TX_ELEMENT _pktTxBuf[1];
          RX_ELEMENT _pktRxBuf[NUM_RX_BUF];
          WORD       _dummy[2];        /* screenSeg,newInOffset */
          BYTE       _fanChars[4];
          WORD       _fanIndex;
          BYTE       _PktReceiver[15]; /* starts on a paragraph (16byte) */
        } PktRealStub;
  #include <sys/pack_off.h>

  // 定义 real_stub_array 数组，用于保存生成的操作码数组
  static BYTE real_stub_array [] = {
         #include "pkt_stub.inc"       /* generated opcode array */
       };

  // 定义一系列宏，用于获取 PktRealStub 结构体中各个成员的偏移量
  #define rxOutOfs      offsetof (PktRealStub,_rxOutOfs)
  #define rxInOfs       offsetof (PktRealStub,_rxInOfs)
  #define PktReceiver   offsetof (PktRealStub,_PktReceiver [para_skip])
  #define pktDrop       offsetof (PktRealStub,_pktDrop)
  #define pktTemp       offsetof (PktRealStub,_pktTemp)
  #define pktTxBuf      offsetof (PktRealStub,_pktTxBuf)
  #define FIRST_RX_BUF  offsetof (PktRealStub,_pktRxBuf [0])
  #define LAST_RX_BUF   offsetof (PktRealStub,_pktRxBuf [NUM_RX_BUF-1])
#else
  # 外部变量声明，表示接收数据包的偏移量
  extern WORD       rxOutOfs;    /* offsets into pktRxBuf FIFO queue   */
  extern WORD       rxInOfs;
  extern DWORD      pktDrop;     /* # packets dropped in PktReceiver() */
  extern BYTE       pktRxEnd;    /* marks the end of r-mode code/data  */

  extern RX_ELEMENT pktRxBuf [NUM_RX_BUF];       /* PktDrvr Rx buffers */
  extern TX_ELEMENT pktTxBuf;                    /* PktDrvr Tx buffer  */
  extern char       pktTemp[20];                 /* PktDrvr temp area  */

  #define FIRST_RX_BUF (WORD) &pktRxBuf [0]
  #define LAST_RX_BUF  (WORD) &pktRxBuf [NUM_RX_BUF-1]
#endif


#ifdef __BORLANDC__           /* Use Borland's inline functions */
  #define memcpy  __memcpy__
  #define memcmp  __memcmp__
  #define memset  __memset__
#endif


#if (DOSX & PHARLAP)
  # 外部函数声明，表示数据包接收函数
  extern void PktReceiver (void);     /* in pkt_rx0.asm */
  # 静态函数声明，表示实模式下的内存拷贝操作
  static int  RealCopy    (ULONG, ULONG, REALPTR*, FARPTR*, USHORT*);

  #undef  FP_SEG
  #undef  FP_OFF
  # 宏定义，用于获取指针的段地址和偏移地址
  #define FP_OFF(x)     ((WORD)(x))
  #define FP_SEG(x)     ((WORD)(realBase >> 16))
  # 宏定义，用于将段地址和偏移地址转换为实模式下的线性地址
  #define DOS_ADDR(s,o) (((DWORD)(s) << 16) + (WORD)(o))
  # 宏定义，用于将寄存器名转换为扩展寄存器名
  #define r_ax          eax
  #define r_bx          ebx
  #define r_dx          edx
  #define r_cx          ecx
  #define r_si          esi
  #define r_di          edi
  #define r_ds          ds
  #define r_es          es
  # 局部变量声明，表示保护模式下的指针和实模式下的指针
  LOCAL FARPTR          protBase;
  LOCAL REALPTR         realBase;
  LOCAL WORD            realSeg;   /* DOS para-address of allocated area */
  LOCAL SWI_REGS        reg;
  # 静态变量声明，表示实模式下的接收数据包的偏移量指针
  static WORD _far *rxOutOfsFp, *rxInOfsFp;
#elif (DOSX & DJGPP)
  # 如果是在 DJGPP 环境下
  static _go32_dpmi_seginfo rm_mem;
  static __dpmi_regs        reg;
  static DWORD              realBase;
  static int                para_skip = 0;

  #define DOS_ADDR(s,o)     (((WORD)(s) << 4) + (o))  # 定义宏，用于计算实模式地址
  #define r_ax              x.ax  # 定义宏，简化对寄存器的访问
  #define r_bx              x.bx
  #define r_dx              x.dx
  #define r_cx              x.cx
  #define r_si              x.si
  #define r_di              x.di
  #define r_ds              x.ds
  #define r_es              x.es

#elif (DOSX & DOS4GW)
  # 如果是在 DOS4GW 环境下
  LOCAL struct DPMI_regs    reg;
  LOCAL WORD                rm_base_seg, rm_base_sel;
  LOCAL DWORD               realBase;
  LOCAL int                 para_skip = 0;

  LOCAL DWORD dpmi_get_real_vector (int intr);  # 定义本地函数，用于获取实模式中断向量
  LOCAL WORD  dpmi_real_malloc     (int size, WORD *selector);  # 定义本地函数，用于在实模式中分配内存
  LOCAL void  dpmi_real_free       (WORD selector);  # 定义本地函数，用于在实模式中释放内存
  #define DOS_ADDR(s,o) (((DWORD)(s) << 4) + (WORD)(o))  # 定义宏，用于计算实模式地址

#else              /* real-mode Borland etc. */
  # 如果是在实模式 Borland 等环境下
  static struct  {
         WORD r_ax, r_bx, r_cx, r_dx, r_bp;
         WORD r_si, r_di, r_ds, r_es, r_flags;
       } reg;
#endif

#ifdef __HIGHC__
  #pragma Alias (pktDrop,    "_pktDrop")  # 定义别名，用于指定函数的外部名称
  #pragma Alias (pktRxBuf,   "_pktRxBuf")
  #pragma Alias (pktTxBuf,   "_pktTxBuf")
  #pragma Alias (pktTemp,    "_pktTemp")
  #pragma Alias (rxOutOfs,   "_rxOutOfs")
  #pragma Alias (rxInOfs,    "_rxInOfs")
  #pragma Alias (pktRxEnd,   "_pktRxEnd")
  #pragma Alias (PktReceiver,"_PktReceiver")
#endif

PUBLIC PKT_STAT    pktStat;    /* statistics for packets    */  # 公共变量，用于存储数据包的统计信息
PUBLIC PKT_INFO    pktInfo;    /* packet-driver information */  # 公共变量，用于存储数据包驱动程序的信息

PUBLIC PKT_RX_MODE receiveMode  = PDRX_DIRECT;  # 公共变量，用于存储数据包接收模式，默认为直接接收
PUBLIC ETHER       myAddress    = {   0,  0,  0,  0,  0,  0 };  # 公共变量，用于存储本机的以太网地址
PUBLIC ETHER       ethBroadcast = { 255,255,255,255,255,255 };  # 公共变量，用于存储广播地址
# 定义一个结构体，用于存储内部统计信息
LOCAL  struct {             /* internal statistics */
       DWORD  tooSmall;     /* size < ETH_MIN */
       DWORD  tooLarge;     /* size > ETH_MAX */
       DWORD  badSync;      /* count_1 != count_2 */
       DWORD  wrongHandle;  /* upcall to wrong handle */
     } intStat;

/***************************************************************************/

# 根据错误号获取错误信息字符串
PUBLIC const char *PktGetErrorStr (int errNum)
{
  # 定义错误信息字符串数组
  static const char *errStr[] = {
                    "",
                    "Invalid handle number",
                    "No interfaces of specified class found",
                    "No interfaces of specified type found",
                    "No interfaces of specified number found",
                    "Bad packet type specified",
                    "Interface does not support multicast",
                    "Packet driver cannot terminate",
                    "Invalid receiver mode specified",
                    "Insufficient memory space",
                    "Type previously accessed, and not released",
                    "Command out of range, or not implemented",
                    "Cannot send packet (usually hardware error)",
                    "Cannot change hardware address ( > 1 handle open)",
                    "Hardware address has bad length or format",
                    "Cannot reset interface (more than 1 handle open)",
                    "Bad Check-sum",
                    "Bad size",
                    "Bad sync" ,
                    "Source hit"
                  };

  # 如果错误号超出范围，则返回未知驱动程序错误
  if (errNum < 0 || errNum >= DIM(errStr))
     return ("Unknown driver error.");
  # 返回对应错误号的错误信息字符串
  return (errStr [errNum]);
}

/**************************************************************************/

# 根据接口类别获取类名
PUBLIC const char *PktGetClassName (WORD class)
{
  # 根据接口类别进行判断
  switch (class)
  {
    case PD_ETHER:
         return ("DIX-Ether");
    case PD_PRONET10:
         return ("ProNET-10");
    case PD_IEEE8025:
         return ("IEEE 802.5");
    case PD_OMNINET:
         return ("OmniNet");
    # 如果协议类型是苹果Talk，则返回"AppleTalk"
    case PD_APPLETALK:
         return ("AppleTalk");
    # 如果协议类型是串行线路接口协议（SLIP），则返回"SLIP"
    case PD_SLIP:
         return ("SLIP");
    # 如果协议类型是StartLAN，则返回"StartLAN"
    case PD_STARTLAN:
         return ("StartLAN");
    # 如果协议类型是ArcNet，则返回"ArcNet"
    case PD_ARCNET:
         return ("ArcNet");
    # 如果协议类型是AX.25，则返回"AX.25"
    case PD_AX25:
         return ("AX.25");
    # 如果协议类型是KISS，则返回"KISS"
    case PD_KISS:
         return ("KISS");
    # 如果协议类型是带有802.2头部的IEEE 802.3，则返回"IEEE 802.3 w/802.2 hdr"
    case PD_IEEE8023_2:
         return ("IEEE 802.3 w/802.2 hdr");
    # 如果协议类型是带有802.2头部的FDDI，则返回"FDDI w/802.2 hdr"
    case PD_FDDI8022:
         return ("FDDI w/802.2 hdr");
    # 如果协议类型是X.25，则返回"X.25"
    case PD_X25:
         return ("X.25");
    # 如果协议类型是LANstar，则返回"LANstar"
    case PD_LANstar:
         return ("LANstar");
    # 如果协议类型是PPP，则返回"PPP"
    case PD_PPP:
         return ("PPP");
    # 如果协议类型未知，则返回"unknown"
    default:
         return ("unknown");
  }
  # 返回接收模式对应的字符串描述
  PUBLIC char const *PktRXmodeStr (PKT_RX_MODE mode)
  {
    # 定义接收模式对应的字符串数组
    static const char *modeStr [] = {
                  "Receiver turned off",
                  "Receive only directly addressed packets",
                  "Receive direct & broadcast packets",
                  "Receive direct,broadcast and limited multicast packets",
                  "Receive direct,broadcast and all multicast packets",
                  "Receive all packets (promiscuouos mode)"
                };

    # 如果接收模式超出字符串数组的范围，则返回 "??" 字符串
    if (mode > DIM(modeStr))
       return ("??");
    # 返回对应接收模式的字符串描述
    return (modeStr [mode-1]);
  }

  # 判断是否有数据包中断
  LOCAL __inline BOOL PktInterrupt (void)
  {
    BOOL okay;

    # 根据不同的操作系统和环境执行不同的中断处理
    # DOSX & PHARLAP
    # DOSX & DJGPP
    # DOSX & DOS4GW
    # 其他情况
    if (DOSX & PHARLAP)
    {
      _dx_real_int ((UINT)pktInfo.intr, &reg);
      okay = ((reg.flags & 1) == 0);  /* OK if carry clear */
    }
    elif (DOSX & DJGPP)
    {
      __dpmi_int ((int)pktInfo.intr, &reg);
      okay = ((reg.x.flags & 1) == 0);
    }
    elif (DOSX & DOS4GW)
    {
      union  REGS  r;
      struct SREGS s;

      memset (&r, 0, sizeof(r));
      segread (&s);
      r.w.ax  = 0x300;
      r.x.ebx = pktInfo.intr;
      r.w.cx  = 0;
      s.es    = FP_SEG (&reg);
      r.x.edi = FP_OFF (&reg);
      reg.r_flags = 0;
      reg.r_ss = reg.r_sp = 0;     /* DPMI host provides stack */

      int386x (0x31, &r, &r, &s);
      okay = (!r.w.cflag);
    }
    else
    {
      reg.r_flags = 0;
      intr (pktInfo.intr, (struct REGPACK*)&reg);
      okay = ((reg.r_flags & 1) == 0);
    }

    # 如果中断处理正常，则清空错误信息，否则获取错误字符串描述
    if (okay)
         pktInfo.error = NULL;
    else pktInfo.error = PktGetErrorStr (reg.r_dx >> 8);
    return (okay);
  }

  # 搜索中断号为 0x60 到 0x80 的数据包驱动程序
  PUBLIC BOOL PktSearchDriver (void)
  {
    BYTE intr  = 0x20;
    BOOL found = FALSE;

    # 循环搜索中断号为 0x60 到 0xFF 的数据包驱动程序
    while (!found && intr < 0xFF)
    {
    // 声明一个长度为12的静态字符数组，用于存储"PKT DRVR"字符串
    static char str[12];                 /* 3 + strlen("PKT DRVR") */
    // 声明一个长度为9的静态字符数组，并初始化为"PKT DRVR"，占据偏移量为3的位置
    static char pktStr[9] = "PKT DRVR";  /* ASCIIZ string at ofs 3 */
    // 声明一个DWORD类型的变量rp，用于在中断程序中使用
    DWORD  rp;                           /* in interrupt  routine  */
#if (DOSX & PHARLAP)
    # 如果操作系统是DOSX并且使用PHARLAP内存管理器
    _dx_rmiv_get (intr, &rp);
    // 获取实模式中断向量
    ReadRealMem (&str, (REALPTR)rp, sizeof(str));

#elif (DOSX & DJGPP)
    # 如果操作系统是DOSX并且使用DJGPP编译器
    __dpmi_raddr realAdr;
    // 获取实模式中断向量
    __dpmi_get_real_mode_interrupt_vector (intr, &realAdr);
    rp = (realAdr.segment << 4) + realAdr.offset16;
    // 从实模式内存中读取数据
    dosmemget (rp, sizeof(str), &str);

#elif (DOSX & DOS4GW)
    # 如果操作系统是DOSX并且使用DOS4GW扩展器
    rp = dpmi_get_real_vector (intr);
    // 从实模式内存中复制数据
    memcpy (&str, (void*)rp, sizeof(str));

#else
    # 如果操作系统不符合以上条件
    _fmemcpy (&str, getvect(intr), sizeof(str));
#endif

    // 比较内存块，判断是否找到指定的数据
    found = memcmp (&str[3],&pktStr,sizeof(pktStr)) == 0;
    // 增加中断号
    intr++;
  }
  // 设置pktInfo的中断属性
  pktInfo.intr = (found ? intr-1 : 0);
  // 返回是否找到指定数据
  return (found);
}


/**************************************************************************/

static BOOL PktSetAccess (void)
{
  // 设置寄存器的值
  reg.r_ax = 0x0200 + pktInfo.class;
  reg.r_bx = 0xFFFF;
  reg.r_dx = 0;
  reg.r_cx = 0;

#if (DOSX & PHARLAP)
  reg.ds  = 0;
  reg.esi = 0;
  reg.es  = RP_SEG (realBase);
  reg.edi = (WORD) &PktReceiver;

#elif (DOSX & DJGPP)
  reg.x.ds = 0;
  reg.x.si = 0;
  reg.x.es = rm_mem.rm_segment;
  reg.x.di = PktReceiver;

#elif (DOSX & DOS4GW)
  reg.r_ds = 0;
  reg.r_si = 0;
  reg.r_es = rm_base_seg;
  reg.r_di = PktReceiver;

#else
  reg.r_ds = 0;
  reg.r_si = 0;
  reg.r_es = FP_SEG (&PktReceiver);
  reg.r_di = FP_OFF (&PktReceiver);
#endif

  // 如果PktInterrupt返回false，则返回false
  if (!PktInterrupt())
     return (FALSE);

  // 设置pktInfo的句柄属性
  pktInfo.handle = reg.r_ax;
  return (TRUE);
}

/**************************************************************************/

PUBLIC BOOL PktReleaseHandle (WORD handle)
{
  // 设置寄存器的值
  reg.r_ax = 0x0300;
  reg.r_bx = handle;
  // 调用PktInterrupt函数
  return PktInterrupt();
}

/**************************************************************************/

PUBLIC BOOL PktTransmit (const void *eth, int len)
{
  // 如果长度大于最大以太网传输单元
  if (len > ETH_MTU)
     return (FALSE);

  reg.r_ax = 0x0400;             /* Function 4, send pkt */
  reg.r_cx = len;                /* total size of frame  */
#if (DOSX & DJGPP)
  // 如果是 DJGPP 环境，则将数据从内存复制到实模式内存
  dosmemput (eth, len, realBase+pktTxBuf);
  // 设置寄存器 DS 为实模式内存段地址
  reg.x.ds = rm_mem.rm_segment;  /* DOS data segment and */
  // 设置寄存器 SI 为数据偏移地址
  reg.x.si = pktTxBuf;           /* DOS offset to buffer */

#elif (DOSX & DOS4GW)
  // 如果是 DOS4GW 环境，则使用内存拷贝函数将数据从内存复制到实模式内存
  memcpy ((void*)(realBase+pktTxBuf), eth, len);
  // 设置寄存器 DS 为实模式内存段地址
  reg.r_ds = rm_base_seg;
  // 设置寄存器 SI 为数据偏移地址
  reg.r_si = pktTxBuf;

#elif (DOSX & PHARLAP)
  // 如果是 PHARLAP 环境，则使用内存拷贝函数将数据从内存复制到实模式内存
  memcpy (&pktTxBuf, eth, len);
  // 设置寄存器 DS 为实模式内存段地址
  reg.r_ds = FP_SEG (&pktTxBuf);
  // 设置寄存器 SI 为数据偏移地址
  reg.r_si = FP_OFF (&pktTxBuf);

#else
  // 如果是其他环境，则设置寄存器 DS 为以太网数据的段地址
  reg.r_ds = FP_SEG (eth);
  // 设置寄存器 SI 为以太网数据的偏移地址
  reg.r_si = FP_OFF (eth);
#endif

  // 调用 PktInterrupt 函数
  return PktInterrupt();
}

/**************************************************************************/

#if (DOSX & (DJGPP|DOS4GW))
// 定义一个内联函数 CheckElement，根据不同的环境选择不同的参数类型
LOCAL __inline BOOL CheckElement (RX_ELEMENT *rx)
#else
// 定义一个内联函数 CheckElement，根据不同的环境选择不同的参数类型
LOCAL __inline BOOL CheckElement (RX_ELEMENT _far *rx)
#endif
{
  WORD count_1, count_2;

  /*
   * We got an upcall to the same RMCB with wrong handle.
   * This can happen if we failed to release handle at program exit
   */
  // 如果接收到相同 RMCB 的错误句柄，则进行处理
  if (rx->handle != pktInfo.handle)
  {
    // 设置错误信息
    pktInfo.error = "Wrong handle";
    // 错误统计加一
    intStat.wrongHandle++;
    // 释放句柄
    PktReleaseHandle (rx->handle);
    return (FALSE);
  }
  // 获取两个计数值
  count_1 = rx->firstCount;
  count_2 = rx->secondCount;

  // 如果两个计数值不相等，则设置错误信息，错误统计加一
  if (count_1 != count_2)
  {
    pktInfo.error = "Bad sync";
    intStat.badSync++;
    return (FALSE);
  }
  // 如果计数值大于最大以太网数据长度，则设置错误信息，错误统计加一
  if (count_1 > ETH_MAX)
  {
    pktInfo.error = "Large esize";
    intStat.tooLarge++;
    return (FALSE);
  }
#if 0
  // 如果计数值小于最小以太网数据长度，则设置错误信息，错误统计加一
  if (count_1 < ETH_MIN)
  {
    pktInfo.error = "Small esize";
    intStat.tooSmall++;
    return (FALSE);
  }
#endif
  // 返回真值
  return (TRUE);
}

/**************************************************************************/

PUBLIC BOOL PktTerminHandle (WORD handle)
{
  // 设置寄存器 AX 为 0x0500
  reg.r_ax = 0x0500;
  // 设置寄存器 BX 为句柄值
  reg.r_bx = handle;
  // 调用 PktInterrupt 函数
  return PktInterrupt();
}

/**************************************************************************/

PUBLIC BOOL PktResetInterface (WORD handle)
{
  // 设置寄存器 AX 为 0x0700
  reg.r_ax = 0x0700;
  // 设置寄存器 BX 为句柄值
  reg.r_bx = handle;
  // 调用 PktInterrupt 函数
  return PktInterrupt();
}

/**************************************************************************
# 设置数据包接收模式
PUBLIC BOOL PktSetReceiverMode (PKT_RX_MODE mode):
    # 如果数据包类别是 PD_SLIP 或 PD_PPP，则返回 TRUE
    if (pktInfo.class == PD_SLIP || pktInfo.class == PD_PPP)
        return (TRUE);
    # 设置寄存器 AX 的值为 0x1400
    reg.r_ax = 0x1400;
    # 设置寄存器 BX 的值为 pktInfo.handle
    reg.r_bx = pktInfo.handle;
    # 设置寄存器 CX 的值为 mode 的低 16 位
    reg.r_cx = (WORD)mode;
    # 如果 PktInterrupt() 返回 FALSE，则返回 FALSE
    if (!PktInterrupt())
        return (FALSE);
    # 将接收模式设置为 mode
    receiveMode = mode;
    # 返回 TRUE
    return (TRUE);

/**************************************************************************/

# 获取数据包接收模式
PUBLIC BOOL PktGetReceiverMode (PKT_RX_MODE *mode):
    # 设置寄存器 AX 的值为 0x1500
    reg.r_ax = 0x1500;
    # 设置寄存器 BX 的值为 pktInfo.handle
    reg.r_bx = pktInfo.handle;
    # 如果 PktInterrupt() 返回 FALSE，则返回 FALSE
    if (!PktInterrupt())
        return (FALSE);
    # 将 mode 指向的地址的值设置为寄存器 AX 的值
    *mode = reg.r_ax;
    # 返回 TRUE
    return (TRUE);

/**************************************************************************/

# 获取统计信息
static PKT_STAT initialStat;         /* statistics at startup */
static BOOL     resetStat = FALSE;   /* statistics reset ? */
PUBLIC BOOL PktGetStatistics (WORD handle):
    # 设置寄存器 AX 的值为 0x1800
    reg.r_ax = 0x1800;
    # 设置寄存器 BX 的值为 handle
    reg.r_bx = handle;
    # 如果 PktInterrupt() 返回 FALSE，则返回 FALSE
    if (!PktInterrupt())
        return (FALSE);
    # 根据不同的宏定义，从不同的内存地址读取数据包统计信息
    # 并将其存储到 pktStat 中
    # DOSX & PHARLAP
    ReadRealMem (&pktStat, DOS_ADDR(reg.ds,reg.esi), sizeof(pktStat));
    # DOSX & DJGPP
    dosmemget (DOS_ADDR(reg.x.ds,reg.x.si), sizeof(pktStat), &pktStat);
    # DOSX & DOS4GW
    memcpy (&pktStat, (void*)DOS_ADDR(reg.r_ds,reg.r_si), sizeof(pktStat));
    # 默认情况
    _fmemcpy (&pktStat, MK_FP(reg.r_ds,reg.r_si), sizeof(pktStat));
    # 返回 TRUE
    return (TRUE);

/**************************************************************************/

# 获取会话统计信息
PUBLIC BOOL PktSessStatistics (WORD handle):
    # 获取数据包统计信息，如果失败则返回 FALSE
    if (!PktGetStatistics(pktInfo.handle))
        return (FALSE);
    # 如果 resetStat 为真，则重置统计信息
    if (resetStat)
    {
        pktStat.inPackets  -= initialStat.inPackets;
        pktStat.outPackets -= initialStat.outPackets;
        pktStat.inBytes    -= initialStat.inBytes;
        pktStat.outBytes   -= initialStat.outBytes;
        pktStat.inErrors   -= initialStat.inErrors;
        pktStat.outErrors  -= initialStat.outErrors;
        pktStat.outErrors  -= initialStat.outErrors;
        pktStat.lost       -= initialStat.lost;
    }
    # 返回 TRUE
    return (TRUE);
# 重置指定句柄的统计信息
PUBLIC BOOL PktResetStatistics (WORD handle)
{
  # 如果无法获取指定句柄的统计信息，则返回 FALSE
  if (!PktGetStatistics(pktInfo.handle))
     return (FALSE);

  # 将 pktStat 的内容复制到 initialStat 中，长度为 initialStat 的大小
  memcpy (&initialStat, &pktStat, sizeof(initialStat));
  # 将 resetStat 设置为 TRUE
  resetStat = TRUE;
  # 返回 TRUE
  return (TRUE);
}

/**************************************************************************/

# 获取地址信息
PUBLIC BOOL PktGetAddress (ETHER *addr)
{
  # 设置寄存器 AX 的值为 0x0600
  reg.r_ax = 0x0600;
  # 设置寄存器 BX 的值为 pktInfo.handle
  reg.r_bx = pktInfo.handle;
  # 设置寄存器 CX 的值为 addr 的大小
  reg.r_cx = sizeof (*addr);

  # 根据不同的操作系统，设置不同的 ES 和 DI 寄存器的值
  # 根据不同的操作系统，执行不同的内存读取操作
  if (!PktInterrupt())
     return (FALSE);

  # 根据不同的操作系统，执行不同的内存拷贝操作
  return (TRUE);
}

/**************************************************************************/

# 设置地址信息
PUBLIC BOOL PktSetAddress (const ETHER *addr)
{
  # 根据不同的操作系统，执行不同的内存拷贝操作，将 addr 复制到 pktTemp 中
  # 设置寄存器 AX 的值为 0x1900
  reg.r_ax = 0x1900;
  # 设置寄存器 CX 的值为 addr 的大小
  reg.r_cx = sizeof (*addr);      # address length       

  # 根据不同的操作系统，设置不同的 ES 和 DI 寄存器的值
  return PktInterrupt();
}

/**************************************************************************
# 获取驱动程序信息的函数
PUBLIC BOOL PktGetDriverInfo (void):
    # 初始化主版本号和次版本号为0
    pktInfo.majVer = 0;
    pktInfo.minVer = 0;
    # 将pktInfo.name数组清零
    memset (&pktInfo.name, 0, sizeof(pktInfo.name));
    # 设置寄存器r_ax的值为0x01FF
    reg.r_ax = 0x01FF;
    reg.r_bx = 0;
    # 调用PktInterrupt函数，如果返回值为假，则返回FALSE
    if (!PktInterrupt())
        return (FALSE);
    # 将pktInfo.number设置为reg.r_cx与0xFF的与操作结果
    pktInfo.number = reg.r_cx & 0xFF;
    # 将pktInfo.class设置为reg.r_cx右移8位的结果
    pktInfo.class  = reg.r_cx >> 8;
    # 将pktInfo.majVer设置为reg.r_bx的值
    pktInfo.majVer = reg.r_bx;  // !!
    # 将pktInfo.funcs设置为reg.r_ax与0xFF的与操作结果
    pktInfo.funcs  = reg.r_ax & 0xFF;
    # 将pktInfo.type设置为reg.r_dx与0xFF的与操作结果
    pktInfo.type   = reg.r_dx & 0xFF;
    # 根据不同的宏定义条件编译不同的代码块
    # 如果DOSX & PHARLAP为真，则调用ReadRealMem函数
    # 如果DOSX & DJGPP为真，则调用dosmemget函数
    # 如果DOSX & DOS4GW为真，则调用memcpy函数
    # 否则调用_fmemcpy函数
    return (TRUE);

# 获取驱动程序参数的函数
PUBLIC BOOL PktGetDriverParam (void):
    # 设置寄存器r_ax的值为0x0A00
    reg.r_ax = 0x0A00;
    # 调用PktInterrupt函数，如果返回值为假，则返回FALSE
    if (!PktInterrupt())
        return (FALSE);
    # 根据不同的宏定义条件编译不同的代码块
    # 如果DOSX & PHARLAP为真，则调用ReadRealMem函数
    # 如果DOSX & DJGPP为真，则调用dosmemget函数
    # 如果DOSX & DOS4GW为真，则调用memcpy函数
    # 否则调用_fmemcpy函数
    return (TRUE);

# 如果DOSX & PHARLAP为真，则定义PktReceive函数
PUBLIC int PktReceive (BYTE *buf, int max):
    # 获取rxInOfsFp指向的值，赋给inOfs
    WORD inOfs  = *rxInOfsFp;
    # 获取rxOutOfsFp指向的值，赋给outOfs
    WORD outOfs = *rxOutOfsFp;
    # 如果outOfs不等于inOfs，则执行以下代码
    {
      // 定义指向 RX_ELEMENT 结构的指针 head，指向偏移量为 outOfs 的内存位置
      RX_ELEMENT _far *head = (RX_ELEMENT _far*)(protBase+outOfs);
      // 定义变量 size 和 len，初始化 len 为 max
      int size, len = max;
    
      // 如果 head 指向的元素通过 CheckElement 函数检查合法
      if (CheckElement(head))
      {
        // 将 size 设置为 head->firstCount 和 sizeof(RX_ELEMENT) 中较小的值
        size = min (head->firstCount, sizeof(RX_ELEMENT));
        // 将 len 设置为 size 和 max 中较小的值
        len  = min (size, max);
        // 将 head->destin 的数据拷贝到 buf 中，拷贝长度为 len
        _fmemcpy (buf, &head->destin, len);
      }
      else
        // 否则将 size 设置为 -1
        size = -1;
    
      // 将 outOfs 增加 sizeof(RX_ELEMENT) 的大小
      outOfs += sizeof (RX_ELEMENT);
      // 如果 outOfs 大于 LAST_RX_BUF，则将其设置为 FIRST_RX_BUF
      if (outOfs > LAST_RX_BUF)
          outOfs = FIRST_RX_BUF;
      // 将 rxOutOfsFp 指向的值设置为 outOfs
      *rxOutOfsFp = outOfs;
      // 返回 size 的值
      return (size);
    }
    // 如果不满足上述条件，则返回 0
    return (0);
    }
    
    // 定义一个名为 PktQueueBusy 的函数，参数为 busy
    PUBLIC void PktQueueBusy (BOOL busy)
    {
      // 根据 busy 的值更新 rxOutOfsFp 指向的值
      *rxOutOfsFp = busy ? (*rxInOfsFp + sizeof(RX_ELEMENT)) : *rxInOfsFp;
      // 如果 rxOutOfsFp 指向的值大于 LAST_RX_BUF，则将其设置为 FIRST_RX_BUF
      if (*rxOutOfsFp > LAST_RX_BUF)
          *rxOutOfsFp = FIRST_RX_BUF;
      // 将 pktDrop 的值设置为 0
      *(DWORD _far*)(protBase + (WORD)&pktDrop) = 0;
    }
    
    // 定义一个名为 PktBuffersUsed 的函数，无参数
    PUBLIC WORD PktBuffersUsed (void)
    {
      // 获取 rxInOfsFp 和 rxOutOfsFp 指向的值
      WORD inOfs  = *rxInOfsFp;
      WORD outOfs = *rxOutOfsFp;
    
      // 如果 inOfs 大于等于 outOfs，则返回它们之间的元素个数
      if (inOfs >= outOfs)
         return (inOfs - outOfs) / sizeof(RX_ELEMENT);
      // 否则返回 NUM_RX_BUF 减去它们之间的元素个数
      return (NUM_RX_BUF - (outOfs - inOfs) / sizeof(RX_ELEMENT));
    }
    
    // 定义一个名为 PktRxDropped 的函数，无参数
    PUBLIC DWORD PktRxDropped (void)
    {
      // 返回 pktDrop 的值
      return (*(DWORD _far*)(protBase + (WORD)&pktDrop));
    }
  # 如果操作系统是 DOSX 且编译器是 DJGPP，则定义 PktReceive 函数
  PUBLIC int PktReceive (BYTE *buf, int max)
  {
    # 从内存中获取接收缓冲区的偏移量
    WORD ofs = _farpeekw (_dos_ds, realBase+rxOutOfs);

    # 如果接收缓冲区中有数据
    if (ofs != _farpeekw (_dos_ds, realBase+rxInOfs))
    {
      # 定义并初始化 RX_ELEMENT 结构
      RX_ELEMENT head;
      int  size, len = max;

      # 从接收缓冲区中读取数据
      head.firstCount  = _farpeekw (_dos_ds, realBase+ofs);
      head.secondCount = _farpeekw (_dos_ds, realBase+ofs+2);
      head.handle      = _farpeekw (_dos_ds, realBase+ofs+4);

      # 检查接收到的数据是否有效
      if (CheckElement(&head))
      {
        size = min (head.firstCount, sizeof(RX_ELEMENT));
        len  = min (size, max);
        dosmemget (realBase+ofs+6, len, buf);
      }
      else
        size = -1;

      # 更新接收缓冲区的偏移量
      ofs += sizeof (RX_ELEMENT);
      if (ofs > LAST_RX_BUF)
           _farpokew (_dos_ds, realBase+rxOutOfs, FIRST_RX_BUF);
      else _farpokew (_dos_ds, realBase+rxOutOfs, ofs);
      return (size);
    }
    return (0);
  }

  # 定义 PktQueueBusy 函数，用于设置接收队列的忙碌状态
  PUBLIC void PktQueueBusy (BOOL busy)
  {
    WORD ofs;

    # 禁用中断
    disable();
    # 获取接收缓冲区的偏移量
    ofs = _farpeekw (_dos_ds, realBase+rxInOfs);
    # 如果队列忙碌，则更新偏移量
    if (busy)
       ofs += sizeof (RX_ELEMENT);

    # 更新接收缓冲区的偏移量
    if (ofs > LAST_RX_BUF)
         _farpokew (_dos_ds, realBase+rxOutOfs, FIRST_RX_BUF);
    else _farpokew (_dos_ds, realBase+rxOutOfs, ofs);
    # 重置丢包计数
    _farpokel (_dos_ds, realBase+pktDrop, 0UL);
    # 启用中断
    enable();
  }

  # 定义 PktBuffersUsed 函数，用于获取已使用的接收缓冲区数量
  PUBLIC WORD PktBuffersUsed (void)
  {
    WORD inOfs, outOfs;

    # 禁用中断
    disable();
    # 获取接收缓冲区的输入和输出偏移量
    inOfs  = _farpeekw (_dos_ds, realBase+rxInOfs);
    outOfs = _farpeekw (_dos_ds, realBase+rxOutOfs);
    # 启用中断
    enable();
    # 计算已使用的接收缓冲区数量
    if (inOfs >= outOfs)
       return (inOfs - outOfs) / sizeof(RX_ELEMENT);
    return (NUM_RX_BUF - (outOfs - inOfs) / sizeof(RX_ELEMENT));
  }

  # 定义 PktRxDropped 函数，用于获取丢包数量
  PUBLIC DWORD PktRxDropped (void)
  {
    return _farpeekl (_dos_ds, realBase+pktDrop);
  }

#elif (DOSX & DOS4GW)
  # 如果操作系统是 DOSX 且编译器是 DOS4GW，则定义 PktReceive 函数
  PUBLIC int PktReceive (BYTE *buf, int max)
  {
    # 从内存中获取接收缓冲区的偏移量
    WORD ofs = *(WORD*) (realBase+rxOutOfs);

    # 如果接收缓冲区中有数据
    if (ofs != *(WORD*) (realBase+rxInOfs))
    {
      // 定义一个名为 head 的 RX_ELEMENT 结构体变量
      RX_ELEMENT head;
      // 定义整型变量 size，并初始化为 max；定义整型变量 len，并初始化为 max
      int  size, len = max;

      // 从 realBase+ofs 处读取两个字节，赋值给 head 结构体的 firstCount 成员
      head.firstCount  = *(WORD*) (realBase+ofs);
      // 从 realBase+ofs+2 处读取两个字节，赋值给 head 结构体的 secondCount 成员
      head.secondCount = *(WORD*) (realBase+ofs+2);
      // 从 realBase+ofs+4 处读取两个字节，赋值给 head 结构体的 handle 成员
      head.handle      = *(WORD*) (realBase+ofs+4);

      // 调用 CheckElement 函数检查 head 结构体
      if (CheckElement(&head))
      {
        // 将 head.firstCount 和 sizeof(RX_ELEMENT) 中较小的值赋给 size
        size = min (head.firstCount, sizeof(RX_ELEMENT));
        // 将 size 和 max 中较小的值赋给 len
        len  = min (size, max);
        // 从 realBase+ofs+6 处复制长度为 len 的数据到 buf 中
        memcpy (buf, (const void*)(realBase+ofs+6), len);
      }
      else
        // 如果 CheckElement 函数返回 false，则将 size 赋值为 -1
        size = -1;

      // 将 ofs 增加 RX_ELEMENT 结构体的大小
      ofs += sizeof (RX_ELEMENT);
      // 如果 ofs 大于 LAST_RX_BUF，则将 FIRST_RX_BUF 的值写入 realBase+rxOutOfs 处
      if (ofs > LAST_RX_BUF)
           *(WORD*) (realBase+rxOutOfs) = FIRST_RX_BUF;
      else // 否则将 ofs 的值写入 realBase+rxOutOfs 处
           *(WORD*) (realBase+rxOutOfs) = ofs;
      // 返回 size 的值
      return (size);
    }
    // 返回 0
    return (0);
  }

  // 定义一个名为 PktQueueBusy 的函数，参数为 busy
  PUBLIC void PktQueueBusy (BOOL busy)
  {
    // 定义一个名为 ofs 的 WORD 类型变量
    WORD ofs;

    // 禁用中断
    _disable();
    // 从 realBase+rxInOfs 处读取两个字节，赋值给 ofs
    ofs = *(WORD*) (realBase+rxInOfs);
    // 如果 busy 为真，则将 ofs 增加 RX_ELEMENT 结构体的大小
    if (busy)
       ofs += sizeof (RX_ELEMENT);

    // 如果 ofs 大于 LAST_RX_BUF，则将 FIRST_RX_BUF 的值写入 realBase+rxOutOfs 处
    if (ofs > LAST_RX_BUF)
         *(WORD*) (realBase+rxOutOfs) = FIRST_RX_BUF;
    else // 否则将 ofs 的值写入 realBase+rxOutOfs 处
         *(WORD*) (realBase+rxOutOfs) = ofs;
    // 将 0UL 写入 realBase+pktDrop 处
    *(DWORD*) (realBase+pktDrop) = 0UL;
    // 启用中断
    _enable();
  }

  // 定义一个名为 PktBuffersUsed 的函数，无参数
  PUBLIC WORD PktBuffersUsed (void)
  {
    // 定义两个名为 inOfs 和 outOfs 的 WORD 类型变量
    WORD inOfs, outOfs;

    // 禁用中断
    _disable();
    // 从 realBase+rxInOfs 处读取两个字节，赋值给 inOfs
    inOfs  = *(WORD*) (realBase+rxInOfs);
    // 从 realBase+rxOutOfs 处读取两个字节，赋值给 outOfs
    outOfs = *(WORD*) (realBase+rxOutOfs);
    // 启用中断
    _enable();
    // 如果 inOfs 大于等于 outOfs，则返回它们之间 RX_ELEMENT 结构体个数
    if (inOfs >= outOfs)
       return (inOfs - outOfs) / sizeof(RX_ELEMENT);
    // 否则返回总共的 RX_ELEMENT 结构体个数
    return (NUM_RX_BUF - (outOfs - inOfs) / sizeof(RX_ELEMENT));
  }

  // 定义一个名为 PktRxDropped 的函数，无参数
  PUBLIC DWORD PktRxDropped (void)
  {
    // 返回 realBase+pktDrop 处的值
    return *(DWORD*) (realBase+pktDrop);
  }
#else     /* real-mode small/large model */

  // 接收数据包的函数
  PUBLIC int PktReceive (BYTE *buf, int max)
  {
    // 如果接收缓冲区中有数据
    if (rxOutOfs != rxInOfs)
    {
      // 获取指向接收缓冲区头部的指针
      RX_ELEMENT far *head = (RX_ELEMENT far*) MK_FP (_DS,rxOutOfs);
      int  size, len = max;

      // 检查接收缓冲区头部元素是否有效
      if (CheckElement(head))
      {
        // 获取头部元素的数据大小
        size = min (head->firstCount, sizeof(RX_ELEMENT));
        // 计算实际拷贝的数据长度
        len  = min (size, max);
        // 将数据拷贝到指定缓冲区
        _fmemcpy (buf, &head->destin, len);
      }
      else
        size = -1;

      // 更新接收缓冲区的指针位置
      rxOutOfs += sizeof (RX_ELEMENT);
      if (rxOutOfs > LAST_RX_BUF)
          rxOutOfs = FIRST_RX_BUF;
      return (size);
    }
    // 如果接收缓冲区中没有数据，则返回 0
    return (0);
  }

  // 设置接收队列的忙状态
  PUBLIC void PktQueueBusy (BOOL busy)
  {
    // 根据忙状态更新接收缓冲区的指针位置
    rxOutOfs = busy ? (rxInOfs + sizeof(RX_ELEMENT)) : rxInOfs;
    if (rxOutOfs > LAST_RX_BUF)
        rxOutOfs = FIRST_RX_BUF;
    // 重置数据包丢弃计数
    pktDrop = 0L;
  }

  // 获取已使用的接收缓冲区数量
  PUBLIC WORD PktBuffersUsed (void)
  {
    WORD inOfs  = rxInOfs;
    WORD outOfs = rxOutOfs;

    // 计算已使用的接收缓冲区数量
    if (inOfs >= outOfs)
       return ((inOfs - outOfs) / sizeof(RX_ELEMENT));
    return (NUM_RX_BUF - (outOfs - inOfs) / sizeof(RX_ELEMENT));
  }

  // 获取丢弃的数据包数量
  PUBLIC DWORD PktRxDropped (void)
  {
    return (pktDrop);
  }
#endif

/**************************************************************************/

// 释放内存的函数
LOCAL __inline void PktFreeMem (void)
{
  // 根据不同的环境释放内存
#if (DOSX & PHARLAP)
  if (realSeg)
  {
    _dx_real_free (realSeg);
    realSeg = 0;
  }
#elif (DOSX & DJGPP)
  if (rm_mem.rm_segment)
  {
    unsigned ofs;  /* clear the DOS-mem to prevent further upcalls */

    // 清空 DOS 内存，防止进一步的调用
    for (ofs = 0; ofs < 16 * rm_mem.size / 4; ofs += 4)
       _farpokel (_dos_ds, realBase + ofs, 0);
    _go32_dpmi_free_dos_memory (&rm_mem);
    rm_mem.rm_segment = 0;
  }
#elif (DOSX & DOS4GW)
  if (rm_base_sel)
  {
    dpmi_real_free (rm_base_sel);
    rm_base_sel = 0;
  }
#endif
}

/**************************************************************************/

// 退出驱动程序的函数
PUBLIC BOOL PktExitDriver (void)
{
  // 如果数据包句柄存在
  if (pktInfo.handle)
  {
    // 如果无法恢复接收模式，则输出错误信息
    if (!PktSetReceiverMode(PDRX_BROADCAST))
       PUTS ("Error restoring receiver mode.");
    # 如果释放数据包处理句柄失败，则输出错误信息
    if (!PktReleaseHandle(pktInfo.handle))
       PUTS ("Error releasing PKT-DRVR handle.");

    # 释放内存
    PktFreeMem();
    # 重置数据包处理句柄为0
    pktInfo.handle = 0;
  }

  # 如果调试级别大于等于1，则输出内部统计信息
  if (pcap_pkt_debug >= 1)
     printf ("Internal stats: too-small %lu, too-large %lu, bad-sync %lu, "
             "wrong-handle %lu\n",
             intStat.tooSmall, intStat.tooLarge,
             intStat.badSync, intStat.wrongHandle);
  # 返回TRUE
  return (TRUE);
# 结束 dump_pkt_stub 函数
}

# 如果是 DOSX 并且是 DJGPP 或 DOS4GW，则定义 dump_pkt_stub 函数
static void dump_pkt_stub (void)
{
  int i;

  # 打印 PktReceiver 和 pkt_stub[PktReceiver] 的数值
  fprintf (stderr, "PktReceiver %lu, pkt_stub[PktReceiver] =\n",
           PktReceiver);
  # 遍历 real_stub_array 数组的前 15 个元素，并打印它们的数值
  for (i = 0; i < 15; i++)
      fprintf (stderr, "%02X, ", real_stub_array[i+PktReceiver]);
  # 打印换行符
  fputs ("\n", stderr);
}
#endif

/*
 * 前端初始化程序
 */
PUBLIC BOOL PktInitDriver (PKT_RX_MODE mode)
{
  PKT_RX_MODE rxMode;
  BOOL   writeInfo = (pcap_pkt_debug >= 3);

  # 设置 pktInfo.quiet 的值
  pktInfo.quiet = (pcap_pkt_debug < 3);

  # 如果是 Pharlap DOS extender 并且是用 __HIGHC__ 编译
  if (DOSX & PHARLAP) && defined(__HIGHC__)
  {
    # 打印错误信息并返回 FALSE
    if (_mwenv != 2)
    {
      fprintf (stderr, "Only Pharlap DOS extender supported.\n");
      return (FALSE);
    }
  }

  # 如果是 Pharlap DOS extender 并且是用 __WATCOMC__ 编译
  if (DOSX & PHARLAP) && defined(__WATCOMC__)
  {
    # 打印错误信息并返回 FALSE
    if (_Extender != 1)
    {
      fprintf (stderr, "Only DOS4GW style extenders supported.\n");
      return (FALSE);
    }
  }

  # 如果没有找到 Packet driver，则打印信息，释放内存并返回 FALSE
  if (!PktSearchDriver())
  {
    PUTS ("Packet driver not found.");
    PktFreeMem();
    return (FALSE);
  }

  # 如果无法获取 Packet driver 信息，则打印信息，释放内存并返回 FALSE
  if (!PktGetDriverInfo())
  {
    PUTS ("Error getting pkt-drvr information.");
    PktFreeMem();
    return (FALSE);
  }

  # 如果是 Pharlap DOS extender
  if (DOSX & PHARLAP)
  {
    # 复制内存并设置 rxOutOfsFp 和 rxInOfsFp 的值
    if (RealCopy((ULONG)&rxOutOfs, (ULONG)&pktRxEnd,
               &realBase, &protBase, (USHORT*)&realSeg))
    {
      rxOutOfsFp  = (WORD _far *) (protBase + (WORD) &rxOutOfs);
      rxInOfsFp   = (WORD _far *) (protBase + (WORD) &rxInOfs);
      *rxOutOfsFp = FIRST_RX_BUF;
      *rxInOfsFp  = FIRST_RX_BUF;
    }
    else
    {
      # 打印错误信息并返回 FALSE
      PUTS ("Cannot allocate real-mode stub.");
      return (FALSE);
    }
  }
  # 如果是 DJGPP 或 DOS4GW
  elif (DOSX & (DJGPP|DOS4GW))
  {
    # 如果 real_stub_array 数组的大小超过 0xFFFF，则打印错误信息并返回 FALSE
    if (sizeof(real_stub_array) > 0xFFFF)
    {
      fprintf (stderr, "`real_stub_array[]' too big.\n");
      return (FALSE);
    }
    # 如果是 DJGPP
    if (DOSX & DJGPP)
    {
      # 分配内存并设置 rm_mem.size 的值
      rm_mem.size = (sizeof(real_stub_array) + 15) / 16;

      # 如果分配内存失败或者 rm_mem.rm_offset 不等于 0，则打印错误信息
      if (_go32_dpmi_allocate_dos_memory(&rm_mem) || rm_mem.rm_offset != 0)
      {
        PUTS ("real-mode init failed.");
    # 返回一个布尔值，表示条件为假
    return (FALSE);
  }
  # 计算实模式基地址
  realBase = (rm_mem.rm_segment << 4);
  # 将 real_stub_array 的内容复制到实模式基地址处
  dosmemput (&real_stub_array, sizeof(real_stub_array), realBase);
  # 将 FIRST_RX_BUF 的值写入实模式基地址加上 rxOutOfs 处
  _farpokel (_dos_ds, realBase+rxOutOfs, FIRST_RX_BUF);
  # 将 FIRST_RX_BUF 的值写入实模式基地址加上 rxInOfs 处
  _farpokel (_dos_ds, realBase+rxInOfs,  FIRST_RX_BUF);
  # 如果是 DOSX 和 DOS4GW 环境
  rm_base_seg = dpmi_real_malloc (sizeof(real_stub_array), &rm_base_sel);
  # 如果分配内存失败
  if (!rm_base_seg)
  {
    PUTS ("real-mode init failed.");
    return (FALSE);
  }
  # 计算实模式基地址
  realBase = (rm_base_seg << 4);
  # 将 real_stub_array 的内容复制到实模式基地址处
  memcpy ((void*)realBase, &real_stub_array, sizeof(real_stub_array));
  # 设置 rxOutOfs 和 rxInOfs 的值
  *(WORD*) (realBase+rxOutOfs) = FIRST_RX_BUF;
  *(WORD*) (realBase+rxInOfs)  = FIRST_RX_BUF;

  # 查找 real_stub_array 中的指令
  {
    int pushf = PktReceiver;
    # 循环查找 pushf 和 cli 指令
    while (real_stub_array[pushf++] != 0x9C &&    /* pushf */
           real_stub_array[pushf]   != 0xFA)      /* cli   */
    {
      # 如果循环次数超过 16 次
      if (++para_skip > 16)
      {
        # 输出错误信息
        fprintf (stderr, "Something wrong with `pkt_stub.inc'.\n");
        para_skip = 0;
        # 输出 real_stub_array 的内容
        dump_pkt_stub();
        return (FALSE);
      }
    }
    # 如果 real_stub_array[] 的偏移不是 0xB800
    if (*(WORD*)(real_stub_array + offsetof(PktRealStub,_dummy)) != 0xB800)
    {
      # 输出错误信息
      fprintf (stderr, "`real_stub_array[]' is misaligned.\n");
      return (FALSE);
    }
  }

  # 如果 pcap_pkt_debug 大于 2
  if (pcap_pkt_debug > 2)
      # 输出 real_stub_array 的内容
      dump_pkt_stub();

#else
  # 设置 rxOutOfs 和 rxInOfs 的值
  rxOutOfs = FIRST_RX_BUF;
  rxInOfs  = FIRST_RX_BUF;
#endif

  # 设置 pkt-drvr 的访问权限
  if (!PktSetAccess())
  {
    PUTS ("Error setting pkt-drvr access.");
    # 释放内存
    PktFreeMem();
    return (FALSE);
  }

  # 获取适配器地址
  if (!PktGetAddress(&myAddress))
  {
    PUTS ("Error fetching adapter address.");
    # 释放内存
    PktFreeMem();
    return (FALSE);
  }

  # 设置接收模式
  if (!PktSetReceiverMode(mode))
  {
    PUTS ("Error setting receiver mode.");
    # 释放内存
    PktFreeMem();
    return (FALSE);
  }

  # 获取接收模式
  if (!PktGetReceiverMode(&rxMode))
  {
    PUTS ("Error getting receiver mode.");
    # 释放内存
    PktFreeMem();
    # 返回 FALSE
    return (FALSE);
  }

  # 如果需要写入信息，则打印以下信息
  if (writeInfo)
     printf ("Pkt-driver information:\n"
             "  Version  : %d.%d\n"
             "  Name     : %.15s\n"
             "  Class    : %u (%s)\n"
             "  Type     : %u\n"
             "  Number   : %u\n"
             "  Funcs    : %u\n"
             "  Intr     : %Xh\n"
             "  Handle   : %u\n"
             "  Extended : %s\n"
             "  Hi-perf  : %s\n"
             "  RX mode  : %s\n"
             "  Eth-addr : %02X:%02X:%02X:%02X:%02X:%02X\n",

             pktInfo.majVer, pktInfo.minVer, pktInfo.name,
             pktInfo.class,  PktGetClassName(pktInfo.class),
             pktInfo.type,   pktInfo.number,
             pktInfo.funcs,  pktInfo.intr,   pktInfo.handle,
             pktInfo.funcs == 2 || pktInfo.funcs == 6 ? "Yes" : "No",
             pktInfo.funcs == 5 || pktInfo.funcs == 6 ? "Yes" : "No",
             PktRXmodeStr(rxMode),
             myAddress[0], myAddress[1], myAddress[2],
             myAddress[3], myAddress[4], myAddress[5]);
#if defined(DEBUG) && (DOSX & PHARLAP)
  // 如果定义了 DEBUG 并且 DOSX & PHARLAP 为真，则执行以下代码块
  if (writeInfo)
  {
    // 如果 writeInfo 为真，则执行以下代码块
    DWORD    rAdr = realBase + (WORD)&PktReceiver;
    // 计算 rAdr 的数值
    unsigned sel, ofs;

    printf ("\nReceiver at   %04X:%04X\n", RP_SEG(rAdr),    RP_OFF(rAdr));
    // 打印 Receiver 的地址
    printf ("Realbase    = %04X:%04X\n",   RP_SEG(realBase),RP_OFF(realBase));
    // 打印 Realbase 的地址

    sel = _FP_SEG (protBase);
    ofs = _FP_OFF (protBase);
    printf ("Protbase    = %04X:%08X\n", sel,ofs);
    // 打印 Protbase 的地址
    printf ("RealSeg     = %04X\n", realSeg);
    // 打印 RealSeg 的地址

    sel = _FP_SEG (rxOutOfsFp);
    ofs = _FP_OFF (rxOutOfsFp);
    printf ("rxOutOfsFp  = %04X:%08X\n", sel,ofs);
    // 打印 rxOutOfsFp 的地址

    sel = _FP_SEG (rxInOfsFp);
    ofs = _FP_OFF (rxInOfsFp);
    printf ("rxInOfsFp   = %04X:%08X\n", sel,ofs);
    // 打印 rxInOfsFp 的地址

    printf ("Ready: *rxOutOfsFp = %04X *rxInOfsFp = %04X\n",
            *rxOutOfsFp, *rxInOfsFp);
    // 打印 Ready 的信息

    PktQueueBusy (TRUE);
    printf ("Busy:  *rxOutOfsFp = %04X *rxInOfsFp = %04X\n",
            *rxOutOfsFp, *rxInOfsFp);
    // 打印 Busy 的信息
  }
#endif

  memset (&pktStat, 0, sizeof(pktStat));  /* clear statistics */
  // 将 pktStat 清零，清空统计信息
  PktQueueBusy (TRUE);
  // 设置 PktQueueBusy 为真
  return (TRUE);
  // 返回真值
}


/*
 * DPMI functions only for Watcom + DOS4GW extenders
 */
#if (DOSX & DOS4GW)
LOCAL DWORD dpmi_get_real_vector (int intr)
{
  // 定义一个本地函数 dpmi_get_real_vector，参数为中断号
  union REGS r;

  r.x.eax = 0x200;
  r.x.ebx = (DWORD) intr;
  int386 (0x31, &r, &r);
  return ((r.w.cx << 4) + r.w.dx);
}

LOCAL WORD dpmi_real_malloc (int size, WORD *selector)
{
  // 定义一个本地函数 dpmi_real_malloc，参数为内存大小和选择器
  union REGS r;

  r.x.eax = 0x0100;             /* DPMI allocate DOS memory */
  r.x.ebx = (size + 15) / 16;   /* Number of paragraphs requested */
  int386 (0x31, &r, &r);
  if (r.w.cflag & 1)
     return (0);

  *selector = r.w.dx;
  return (r.w.ax);              /* Return segment address */
}

LOCAL void dpmi_real_free (WORD selector)
{
  // 定义一个本地函数 dpmi_real_free，参数为选择器
  union REGS r;

  r.x.eax = 0x101;              /* DPMI free DOS memory */
  r.x.ebx = selector;           /* Selector to free */
  int386 (0x31, &r, &r);
}
#endif


#if defined(DOSX) && (DOSX & PHARLAP)
/*
 * Description:
 *     This routine allocates conventional memory for the specified block
 *     of code (which must be within the first 64K of the protected mode
 *     program segment) and copies the code to it.
 *
 *     The caller should free up the conventional memory block when it
 *     is done with the conventional memory.
 *
 *     NOTE THIS ROUTINE REQUIRES 386|DOS-EXTENDER 3.0 OR LATER.
 *
 * Calling arguments:
 *     start_offs      start of real mode code in program segment
 *     end_offs        1 byte past end of real mode code in program segment
 *     real_basep      returned;  real mode ptr to use as a base for the
 *                        real mode code (eg, to get the real mode FAR
 *                        addr of a function foo(), take
 *                        real_basep + (ULONG) foo).
 *                        This pointer is constructed such that
 *                        offsets within the real mode segment are
 *                        the same as the link-time offsets in the
 *                        protected mode program segment
 *     prot_basep      returned;  prot mode ptr to use as a base for getting
 *                        to the conventional memory, also constructed
 *                        so that adding the prot mode offset of a
 *                        function or variable to the base gets you a
 *                        ptr to the function or variable in the
 *                        conventional memory block.
 *     rmem_adrp       returned;  real mode para addr of allocated
 *                        conventional memory block, to be used to free
 *                        up the conventional memory when done.  DO NOT
 *                        USE THIS TO CONSTRUCT A REAL MODE PTR, USE
 *                        REAL_BASEP INSTEAD SO THAT OFFSETS WORK OUT
 *                        CORRECTLY.
 *
 * Returned values:
 *     0      if error
 *     1      if success
 */
# 复制实模式内存到保护模式内存
int RealCopy (ULONG    start_offs,    # 起始偏移量
              ULONG    end_offs,      # 结束偏移量
              REALPTR *real_basep,   # 实模式基地址指针
              FARPTR  *prot_basep,   # 保护模式基地址指针
              USHORT  *rmem_adrp)    # 内存地址指针

{
  ULONG   rm_base;    # 用于访问分配的传统内存的基本实模式段地址
  UCHAR  *source;     # 复制的源指针
  FARPTR  destin;     # 复制的目标指针
  ULONG   len;        # 要复制的字节数
  ULONG   temp;
  USHORT  stemp;

  # 首先检查有效输入
  if (start_offs >= end_offs || end_offs > 0x10000)
     return (FALSE);

  # 将 start_offs 向下舍入到段（16字节）边界，以便我们可以轻松设置实模式指针。将 end_offs 向上舍入，以确保我们分配足够的段
  start_offs &= ~15;
  end_offs = (15 + (end_offs << 4)) >> 4;

  # 为我们的实模式代码分配传统内存。记住将字节计数向上舍入到16字节段大小。我们将其分配在DOS数据缓冲区之上，以便DOS数据缓冲区和应用程序传统内存块都可以调整大小。
  # 首先尝试分配它；如果我们无法获得它，将应用程序内存块缩小到最小值，再次尝试分配内存，然后将应用程序内存块扩大到最大值。
  # （不要尝试缩小DOS数据缓冲区以释放传统内存；这对于使文件I/O运行更慢的可能副作用不是很好。）
  len = ((end_offs - start_offs) + 15) >> 4;
  if (_dx_real_above(len, rmem_adrp, &stemp) != _DOSE_NONE)
  {
    if (_dx_cmem_usage(0, 0, &temp, &temp) != _DOSE_NONE)
       return (FALSE);

    if (_dx_real_above(len, rmem_adrp, &stemp) != _DOSE_NONE)
       *rmem_adrp = 0;

    if (_dx_cmem_usage(0, 1, &temp, &temp) != _DOSE_NONE)
    {
      if (*rmem_adrp != 0)
         _dx_real_free (*rmem_adrp);
      return (FALSE);
    }
    # 如果实模式内存地址指针为0，则返回FALSE
    if (*rmem_adrp == 0)
       return (FALSE);
  }

  /* 构造实模式和保护模式指针以访问分配的内存。
   * 注意，我们知道 start_offs 是按照段（16字节）边界对齐的，因为我们将其向下舍入了。
   *
   * 通过将实模式选择器减去 start_offs，使偏移量正确。
   */
  rm_base = ((ULONG) *rmem_adrp) - (start_offs >> 4);
  RP_SET (*real_basep, 0, rm_base);
  FP_SET (*prot_basep, rm_base << 4, SS_DOSMEM);

  /* 将实模式代码/数据复制到分配的内存中
   */
  source = (UCHAR *) start_offs;
  destin = *prot_basep;
  FP_SET (destin, FP_OFF(*prot_basep) + start_offs, FP_SEL(*prot_basep));
  len = end_offs - start_offs;
  WriteFarMem (destin, source, len);

  return (TRUE);
# 如果定义了 DOSX 并且 DOSX 与 PHARLAP 相与的结果为真，则执行以下代码块
#endif /* DOSX && (DOSX & PHARLAP) */
```