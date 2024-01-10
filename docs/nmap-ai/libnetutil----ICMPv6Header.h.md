# `nmap\libnetutil\ICMPv6Header.h`

```
/* This code was originally part of the Nping tool.                        */

#ifndef ICMPv6HEADER_H
#define ICMPv6HEADER_H 1

#include "ICMPHeader.h"

/******************************************************************************/
/*               IMPORTANT INFORMATION ON HOW TO USE THIS CLASS.              */
/******************************************************************************/
/* Packet header diagrams included in this file have been taken from the
 * following IETF RFC documents: RFC 4443, RFC 2461, RFC 2894 */

/* ICMP types and codes.
 * The following types and codes have been defined by IANA. A complete list
 * may be found at http://www.iana.org/assignments/icmpv6-parameters
 *
 * Definitions on the first level of indentation are ICMPv6 Types.
 * Definitions on the second level of indentation (values enclosed in
 * parenthesis) are ICMPv6 Codes */
#define ICMPv6_UNREACH                      1    /* Destination unreachable  [RFC 2463, 4443] */
#define     ICMPv6_UNREACH_NO_ROUTE        (0)   /*  --> No route to destination */
#define     ICMPv6_UNREACH_PROHIBITED      (1)   /*  --> Communication administratively prohibited */
#define     ICMPv6_UNREACH_BEYOND_SCOPE    (2)   /*  --> Beyond scope of source address  [RFC4443] */
#define     ICMPv6_UNREACH_ADDR_UNREACH    (3)   /*  --> Address unreachable */
#define     ICMPv6_UNREACH_PORT_UNREACH    (4)   /*  --> Port unreachable */
#define     ICMPv6_UNREACH_SRC_ADDR_FAILED (5)   /*  --> Source address failed ingress/egress policy [RFC4443] */
#define     ICMPv6_UNREACH_REJECT_ROUTE    (6)   /*  --> Reject route to destination  [RFC4443] */
#define ICMPv6_PKTTOOBIG                    2    /* Packet too big  [RFC 2463, 4443] */
#define ICMPv6_TIMXCEED                     3    /* Time exceeded  [RFC 2463, 4443] */
#define     ICMPv6_TIMXCEED_HOP_EXCEEDED   (0)   /*  --> Hop limit exceeded in transit */
#define     ICMPv6_TIMXCEED_REASS_EXCEEDED (1)   /*  --> Fragment reassembly time exceeded */
# 定义 ICMPv6 参数问题类型，数值为 4，参考 RFC 2463, 4443
#define ICMPv6_PARAMPROB                    4    /* Parameter problem  [RFC 2463, 4443] */
# 定义 ICMPv6 参数问题类型中的字段错误，数值为 0
#define     ICMPv6_PARAMPROB_FIELD         (0)   /*  --> Erroneous header field encountered */
# 定义 ICMPv6 参数问题类型中的下一个头部类型错误，数值为 1
#define     ICMPv6_PARAMPROB_NEXT_HDR      (1)   /*  --> Unrecognized Next Header type encountered */
# 定义 ICMPv6 参数问题类型中的 IPv6 选项错误，数值为 2
#define     ICMPv6_PARAMPROB_OPTION        (2)   /*  --> Unrecognized IPv6 option encountered */
# 定义 ICMPv6 回显请求类型，数值为 128，参考 RFC 2463, 4443
#define ICMPv6_ECHO                        128   /* Echo request  [RFC 2463, 4443] */
# 定义 ICMPv6 回显回复类型，数值为 129，参考 RFC 2463, 4443
#define ICMPv6_ECHOREPLY                   129   /* Echo reply  [RFC 2463, 4443] */
# 定义 ICMPv6 组成员查询类型，数值为 130，参考 RFC 2710
#define ICMPv6_GRPMEMBQUERY                130   /* Group Membership Query  [RFC 2710] */
# 定义 ICMPv6 组成员报告类型，数值为 131，参考 RFC 2710
#define ICMPv6_GRPMEMBREP                  131   /* Group Membership Report  [RFC 2710] */
# 定义 ICMPv6 组成员减少类型，数值为 132，参考 RFC 2710
#define ICMPv6_GRPMEMBRED                  132   /* Group Membership Reduction  [RFC 2710] */
# 定义 ICMPv6 路由器请求类型，数值为 133，参考 RFC 2461
#define ICMPv6_ROUTERSOLICIT               133   /* Router Solicitation  [RFC 2461] */
# 定义 ICMPv6 路由器通告类型，数值为 134，参考 RFC 2461
#define ICMPv6_ROUTERADVERT                134   /* Router Advertisement  [RFC 2461] */
# 定义 ICMPv6 邻居请求类型，数值为 135，参考 RFC 2461
#define ICMPv6_NGHBRSOLICIT                135   /* Neighbor Solicitation  [RFC 2461] */
# 定义 ICMPv6 邻居通告类型，数值为 136，参考 RFC 2461
#define ICMPv6_NGHBRADVERT                 136   /* Neighbor Advertisement  [RFC 2461] */
# 定义 ICMPv6 重定向类型，数值为 137，参考 RFC 2461
#define ICMPv6_REDIRECT                    137   /* Redirect  [RFC 2461] */
# 定义 ICMPv6 路由器重新编号类型，数值为 138，参考 RFC 2894
#define ICMPv6_RTRRENUM                    138   /* Router Renumbering  [RFC 2894] */
# 定义 ICMPv6 路由器重新编号类型中的命令，数值为 0
#define     ICMPv6_RTRRENUM_COMMAND        (0)   /*  --> Router Renumbering Command */
# 定义 ICMPv6 路由器重新编号类型中的结果，数值为 1
#define     ICMPv6_RTRRENUM_RESULT         (1)   /*  --> Router Renumbering Result */
# 定义 ICMPv6 路由器重新编号类型中的序列号重置，数值为 255
#define     ICMPv6_RTRRENUM_SEQ_RESET      (255) /* Sequence Number Reset */
# 定义 ICMPv6 节点信息查询类型，数值为 139，参考 RFC 4620
#define ICMPv6_NODEINFOQUERY               139   /* ICMP Node Information Query  [RFC 4620] */
# 定义 ICMPv6 节点信息查询类型中的 IPv6 地址，数值为 0
#define     ICMPv6_NODEINFOQUERY_IPv6ADDR  (0)   /*  --> The Data field contains an IPv6 address */
# 定义 ICMPv6 节点信息查询类型中的名称，数值为 1
#define     ICMPv6_NODEINFOQUERY_NAME      (1)   /*  --> The Data field contains a name */
# 定义 ICMPv6 节点信息查询类型中的 IPv4 地址，数值为 2
#define     ICMPv6_NODEINFOQUERY_IPv4ADDR  (2)   /*  --> The Data field contains an IPv4 address */
# 定义 ICMPv6 协议中的消息类型和对应的编号，以及相关的注释信息
#define ICMPv6_NODEINFORESP                140   /* ICMP Node Information Response  [RFC 4620] */
#define     ICMPv6_NODEINFORESP_SUCCESS    (0)   /*  --> A successful reply.   */
#define     ICMPv6_NODEINFORESP_REFUSED    (1)   /*  --> The Responder refuses to supply the answer */
#define     ICMPv6_NODEINFORESP_UNKNOWN    (2)   /*  --> The Qtype of the Query is unknown */
#define ICMPv6_INVNGHBRSOLICIT             141   /* Inverse Neighbor Discovery Solicitation Message  [RFC 3122] */
#define ICMPv6_INVNGHBRADVERT              142   /* Inverse Neighbor Discovery Advertisement Message  [RFC 3122] */
#define ICMPv6_MLDV2                       143   /* MLDv2 Multicast Listener Report  [RFC 3810] */
#define ICMPv6_AGENTDISCOVREQ              144   /* Home Agent Address Discovery Request Message  [RFC 3775] */
#define ICMPv6_AGENTDISCOVREPLY            145   /* Home Agent Address Discovery Reply Message  [RFC 3775] */
#define ICMPv6_MOBPREFIXSOLICIT            146   /* Mobile Prefix Solicitation  [RFC 3775] */
#define ICMPv6_MOBPREFIXADVERT             147   /* Mobile Prefix Advertisement  [RFC 3775] */
#define ICMPv6_CERTPATHSOLICIT             148   /* Certification Path Solicitation  [RFC 3971] */
#define ICMPv6_CERTPATHADVERT              149   /* Certification Path Advertisement  [RFC 3971] */
#define ICMPv6_EXPMOBILITY                 150   /* Experimental mobility protocols  [RFC 4065] */
#define ICMPv6_MRDADVERT                   151   /* MRD, Multicast Router Advertisement  [RFC 4286] */
#define ICMPv6_MRDSOLICIT                  152   /* MRD, Multicast Router Solicitation  [RFC 4286] */
#define ICMPv6_MRDTERMINATE                153   /* MRD, Multicast Router Termination  [RFC 4286] */
#define ICMPv6_FMIPV6                      154   /* FMIPv6 messages  [RFC 5568] */

/* Node Information parameters */
/* -> Query types */
#define NI_QTYPE_NOOP      0  /* No operation */
#define NI_QTYPE_UNUSED    1  /* Unused */
#define NI_QTYPE_NODENAME  2  /* Node name */
#define NI_QTYPE_NODEADDRS 3  /* Node addresses */
#define NI_QTYPE_IPv4ADDRS 4
/* -> Misc */
#define NI_NONCE_LEN 8

/* Nping ICMPv6Header Class internal definitions */
#define ICMPv6_COMMON_HEADER_LEN    4
#define ICMPv6_MIN_HEADER_LEN       8
#define ICMPv6_UNREACH_LEN          (ICMPv6_COMMON_HEADER_LEN+4)
#define ICMPv6_PKTTOOBIG_LEN        (ICMPv6_COMMON_HEADER_LEN+4)
#define ICMPv6_TIMXCEED_LEN         (ICMPv6_COMMON_HEADER_LEN+4)
#define ICMPv6_PARAMPROB_LEN        (ICMPv6_COMMON_HEADER_LEN+4)
#define ICMPv6_ECHO_LEN             (ICMPv6_COMMON_HEADER_LEN+4)
#define ICMPv6_ECHOREPLY_LEN        (ICMPv6_COMMON_HEADER_LEN+4)
#define ICMPv6_ROUTERSOLICIT_LEN    (ICMPv6_COMMON_HEADER_LEN+4)
#define ICMPv6_ROUTERADVERT_LEN     (ICMPv6_COMMON_HEADER_LEN+12)
#define ICMPv6_NGHBRSOLICIT_LEN     (ICMPv6_COMMON_HEADER_LEN+20)
#define ICMPv6_NGHBRADVERT_LEN      (ICMPv6_COMMON_HEADER_LEN+20)
#define ICMPv6_REDIRECT_LEN         (ICMPv6_COMMON_HEADER_LEN+36)
#define ICMPv6_RTRRENUM_LEN         (ICMPv6_COMMON_HEADER_LEN+12)
#define ICMPv6_NODEINFO_LEN         (ICMPv6_COMMON_HEADER_LEN+12)
#define ICMPv6_MLD_LEN              (ICMPv6_COMMON_HEADER_LEN+20)
/* This must the MAX() of all values defined above*/
#define ICMPv6_MAX_MESSAGE_BODY     (ICMPv6_REDIRECT_LEN-ICMPv6_COMMON_HEADER_LEN)

/* Node Information flag bitmaks */
#define ICMPv6_NI_FLAG_T    0x01
#define ICMPv6_NI_FLAG_A    0x02
#define ICMPv6_NI_FLAG_C    0x04
#define ICMPv6_NI_FLAG_L    0x08
#define ICMPv6_NI_FLAG_G    0x10
#define ICMPv6_NI_FLAG_S    0x20

}; /* End of class ICMPv6Header */

#endif
```