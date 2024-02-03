# `nmap\libpcap\grammar.h`

```cpp
/* 一个由 GNU Bison 3.0.4 生成的 Bison 解析器 */

/* 用于在 C 语言中创建类似 Yacc 的解析器的 Bison 接口
   版权所有 (C) 1984, 1989-1990, 2000-2015 自由软件基金会

   本程序是自由软件：您可以重新发布或修改它
   根据自由软件基金会发布的 GNU 通用公共许可证的条款，版本为 3 或
   （根据您的选择）任何以后的版本。

   本程序是希望它有用的，但没有任何担保；甚至没有暗示的担保。
   有关更多详细信息，请参见 GNU 通用公共许可证。

   您应该已经收到了 GNU 通用公共许可证的副本
   与本程序一起。如果没有，请参见 <http://www.gnu.org/licenses/>。 */

/* 作为特殊例外，您可以创建一个包含
   Bison 解析器骨架的部分或全部内容，并分发该作品
   根据您的选择的条款，只要该作品本身不是
   使用骨架或其修改版本作为解析器生成器的解析器生成器。
   或者，如果您修改或重新分发
   解析器骨架本身，您可以（根据您的选择）删除此
   特殊例外，这将导致骨架和生成的
   Bison 输出文件将根据 GNU 通用公共许可证进行许可，不包括此特殊例外。

   此特殊例外是由自由软件基金会在
   Bison 2.2 版本中添加的。 */

#ifndef YY_PCAP_GRAMMAR_H_INCLUDED
# define YY_PCAP_GRAMMAR_H_INCLUDED
/* 调试跟踪。 */
#ifndef YYDEBUG
# define YYDEBUG 0
#endif
#if YYDEBUG
extern int pcap_debug;
#endif

/* 标记类型。 */
#ifndef YYTOKENTYPE
# define YYTOKENTYPE
  枚举 yytokentype
  {
    DST = 258,
    SRC = 259,
    HOST = 260,
    GATEWAY = 261,
    NET = 262,
    NETMASK = 263,
    PORT = 264,
    PORTRANGE = 265,
    LESS = 266,
    GREATER = 267,
    PROTO = 268,
    PROTOCHAIN = 269,
    CBYTE = 270,  # 定义常量 CBYTE 为 270
    ARP = 271,  # 定义常量 ARP 为 271
    RARP = 272,  # 定义常量 RARP 为 272
    IP = 273,  # 定义常量 IP 为 273
    SCTP = 274,  # 定义常量 SCTP 为 274
    TCP = 275,  # 定义常量 TCP 为 275
    UDP = 276,  # 定义常量 UDP 为 276
    ICMP = 277,  # 定义常量 ICMP 为 277
    IGMP = 278,  # 定义常量 IGMP 为 278
    IGRP = 279,  # 定义常量 IGRP 为 279
    PIM = 280,  # 定义常量 PIM 为 280
    VRRP = 281,  # 定义常量 VRRP 为 281
    CARP = 282,  # 定义常量 CARP 为 282
    ATALK = 283,  # 定义常量 ATALK 为 283
    AARP = 284,  # 定义常量 AARP 为 284
    DECNET = 285,  # 定义常量 DECNET 为 285
    LAT = 286,  # 定义常量 LAT 为 286
    SCA = 287,  # 定义常量 SCA 为 287
    MOPRC = 288,  # 定义常量 MOPRC 为 288
    MOPDL = 289,  # 定义常量 MOPDL 为 289
    TK_BROADCAST = 290,  # 定义常量 TK_BROADCAST 为 290
    TK_MULTICAST = 291,  # 定义常量 TK_MULTICAST 为 291
    NUM = 292,  # 定义常量 NUM 为 292
    INBOUND = 293,  # 定义常量 INBOUND 为 293
    OUTBOUND = 294,  # 定义常量 OUTBOUND 为 294
    IFINDEX = 295,  # 定义常量 IFINDEX 为 295
    PF_IFNAME = 296,  # 定义常量 PF_IFNAME 为 296
    PF_RSET = 297,  # 定义常量 PF_RSET 为 297
    PF_RNR = 298,  # 定义常量 PF_RNR 为 298
    PF_SRNR = 299,  # 定义常量 PF_SRNR 为 299
    PF_REASON = 300,  # 定义常量 PF_REASON 为 300
    PF_ACTION = 301,  # 定义常量 PF_ACTION 为 301
    TYPE = 302,  # 定义常量 TYPE 为 302
    SUBTYPE = 303,  # 定义常量 SUBTYPE 为 303
    DIR = 304,  # 定义常量 DIR 为 304
    ADDR1 = 305,  # 定义常量 ADDR1 为 305
    ADDR2 = 306,  # 定义常量 ADDR2 为 306
    ADDR3 = 307,  # 定义常量 ADDR3 为 307
    ADDR4 = 308,  # 定义常量 ADDR4 为 308
    RA = 309,  # 定义常量 RA 为 309
    TA = 310,  # 定义常量 TA 为 310
    LINK = 311,  # 定义常量 LINK 为 311
    GEQ = 312,  # 定义常量 GEQ 为 312
    LEQ = 313,  # 定义常量 LEQ 为 313
    NEQ = 314,  # 定义常量 NEQ 为 314
    ID = 315,  # 定义常量 ID 为 315
    EID = 316,  # 定义常量 EID 为 316
    HID = 317,  # 定义常量 HID 为 317
    HID6 = 318,  # 定义常量 HID6 为 318
    AID = 319,  # 定义常量 AID 为 319
    LSH = 320,  # 定义常量 LSH 为 320
    RSH = 321,  # 定义常量 RSH 为 321
    LEN = 322,  # 定义常量 LEN 为 322
    IPV6 = 323,  # 定义常量 IPV6 为 323
    ICMPV6 = 324,  # 定义常量 ICMPV6 为 324
    AH = 325,  # 定义常量 AH 为 325
    ESP = 326,  # 定义常量 ESP 为 326
    VLAN = 327,  # 定义常量 VLAN 为 327
    MPLS = 328,  # 定义常量 MPLS 为 328
    PPPOED = 329,  # 定义常量 PPPOED 为 329
    PPPOES = 330,  # 定义常量 PPPOES 为 330
    GENEVE = 331,  # 定义常量 GENEVE 为 331
    ISO = 332,  # 定义常量 ISO 为 332
    ESIS = 333,  # 定义常量 ESIS 为 333
    CLNP = 334,  # 定义常量 CLNP 为 334
    ISIS = 335,  # 定义常量 ISIS 为 335
    L1 = 336,  # 定义常量 L1 为 336
    L2 = 337,  # 定义常量 L2 为 337
    IIH = 338,  # 定义常量 IIH 为 338
    LSP = 339,  # 定义常量 LSP 为 339
    SNP = 340,  # 定义常量 SNP 为 340
    CSNP = 341,  # 定义常量 CSNP 为 341
    PSNP = 342,  # 定义常量 PSNP 为 342
    STP = 343,  # 定义常量 STP 为 343
    IPX = 344,  # 定义常量 IPX 为 344
    NETBEUI = 345,  # 定义常量 NETBEUI 为 345
    LANE = 346,  # 定义常量 LANE 为 346
    LLC = 347,  # 定义常量 LLC 为 347
    METAC = 348,  # 定义常量 METAC 为 348
    BCC = 349,  # 定义常量 BCC 为 349
    SC = 350,  # 定义常量 SC 为 350
    ILMIC = 351,  # 定义常量 ILMIC 为 351
    OAMF4EC = 352,  # 定义常量 OAMF4EC 为 352
    OAMF4SC = 353,  # 定义常量 OAMF4SC 为 353
    OAM = 354,  # 定义常量 OAM 为 354
    OAMF4 = 355,  # 定义常量 OAMF4 为 355
    CONNECTMSG = 356,  # 定义常量 CONNECTMSG 为 356
    METACONNECT = 357,  # 定义常量 METACONNECT 为 357
    VPI = 358,  # 定义常量 VPI 为 358
    VCI = 359,  # 定义常量 VCI 为 359
    RADIO = 360,  # 定义常量 RADIO 为 360
    FISU = 361,  # 定义常量 FISU 为 361
    LSSU = 362,  # 定义常量 LSSU 为 362
    MSU = 363,  # 定义常量 MSU 为 363
    HFISU = 364,  # 定义常量 HFISU 为 364
    HLSSU = 365,  # 定义常量 HLSSU 为 365
    HMSU = 366,  # 定义常量 HMSU 为 366
    SIO = 367,  # 定义常量 SIO 为 367
    OPC = 368,  # 定义常量 OPC 为 368
    DPC = 369,  # 定义常量 DPC 为 369
    SLS = 370,  # 定义常量 SLS 为 370
    HSIO = 371,  # 定义常量 HSIO 为 371
    HOPC = 372,  # 定义常量 HOPC 为 372
    HDPC = 373,  # 定义常量 HDPC 为 373
    HSLS = 374,  # 定义常量 HSLS 为 374
    LEX_ERROR = 375,  # 定义常量 LEX_ERROR 为 375
    OR = 376,  # 定义常量 OR 为 376
    AND = 377,  # 定义常量 AND 为 377
    UMINUS = 378  # 定义常量 UMINUS 为 378
  };
#endif

/* 值类型 */
#if ! defined YYSTYPE && ! defined YYSTYPE_IS_DECLARED

union YYSTYPE
{
#line 349 "grammar.y" /* yacc.c:1909  */

    int i;  // 整数类型
    bpf_u_int32 h;  // 32位无符号整数类型
    char *s;  // 字符串类型
    struct stmt *stmt;  // 指向结构体 stmt 的指针
    struct arth *a;  // 指向结构体 arth 的指针
    struct {
        struct qual q;  // 包含结构体 qual 的结构体
        int atmfieldtype;  // 整数类型
        int mtp3fieldtype;  // 整数类型
        struct block *b;  // 指向结构体 block 的指针
    } blk;
    struct block *rblk;  // 指向结构体 block 的指针

#line 193 "grammar.h" /* yacc.c:1909  */
};

typedef union YYSTYPE YYSTYPE;  // 声明 YYSTYPE 为 union YYSTYPE 类型
# define YYSTYPE_IS_TRIVIAL 1  // 定义 YYSTYPE_IS_TRIVIAL 为 1
# define YYSTYPE_IS_DECLARED 1  // 定义 YYSTYPE_IS_DECLARED 为 1
#endif

int pcap_parse (void *yyscanner, compiler_state_t *cstate);  // 声明函数 pcap_parse，接受 yyscanner 和 cstate 作为参数

#endif /* !YY_PCAP_GRAMMAR_H_INCLUDED  */
```