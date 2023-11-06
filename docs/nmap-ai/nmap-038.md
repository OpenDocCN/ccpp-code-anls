# Nmap源码解析 38

# `libnetutil/ICMPv6Header.h`

This is a text file that provides information about the Nmap software and its licensing. Nmap is a network scanner that allows users to scan for active hosts, open ports, and network services. The file indicates that Nmap is released under the Nmap Public Source License, but also includes a special Nmap OEM Edition license with more permissive terms. The text file also explains the licensing terms for the official Nmap Windows builds, which include the Npcap software for packet capture and transmission. The file warns users that the official Nmap Windows builds may not be redistributed without special permission, as the Npcap software has separate license terms that prohibit redistribution.


```cpp
/***************************************************************************
 * ICMPv6Header.h -- The ICMPv6Header Class represents an ICMP version 6   *
 * packet. It contains methods to set any header field. In general, these  *
 * methods do error checkings and byte order conversion.                   *
 *                                                                         *
 ***********************IMPORTANT NMAP LICENSE TERMS************************
 *
 * The Nmap Security Scanner is (C) 1996-2023 Nmap Software LLC ("The Nmap
 * Project"). Nmap is also a registered trademark of the Nmap Project.
 *
 * This program is distributed under the terms of the Nmap Public Source
 * License (NPSL). The exact license text applying to a particular Nmap
 * release or source code control revision is contained in the LICENSE
 * file distributed with that version of Nmap or source code control
 * revision. More Nmap copyright/legal information is available from
 * https://nmap.org/book/man-legal.html, and further information on the
 * NPSL license itself can be found at https://nmap.org/npsl/ . This
 * header summarizes some key points from the Nmap license, but is no
 * substitute for the actual license text.
 *
 * Nmap is generally free for end users to download and use themselves,
 * including commercial use. It is available from https://nmap.org.
 *
 * The Nmap license generally prohibits companies from using and
 * redistributing Nmap in commercial products, but we sell a special Nmap
 * OEM Edition with a more permissive license and special features for
 * this purpose. See https://nmap.org/oem/
 *
 * If you have received a written Nmap license agreement or contract
 * stating terms other than these (such as an Nmap OEM license), you may
 * choose to use and redistribute Nmap under those terms instead.
 *
 * The official Nmap Windows builds include the Npcap software
 * (https://npcap.com) for packet capture and transmission. It is under
 * separate license terms which forbid redistribution without special
 * permission. So the official Nmap Windows builds may not be redistributed
 * without special permission (such as an Nmap OEM license).
 *
 * Source is provided to this software because we believe users have a
 * right to know exactly what a program is going to do before they run it.
 * This also allows you to audit the software for security holes.
 *
 * Source code also allows you to port Nmap to new platforms, fix bugs, and add
 * new features. You are highly encouraged to submit your changes as a Github PR
 * or by email to the dev@nmap.org mailing list for possible incorporation into
 * the main distribution. Unless you specify otherwise, it is understood that
 * you are offering us very broad rights to use your submissions as described in
 * the Nmap Public Source License Contributor Agreement. This is important
 * because we fund the project by selling licenses with various terms, and also
 * because the inability to relicense code has caused devastating problems for
 * other Free Software projects (such as KDE and NASM).
 *
 * The free version of Nmap is distributed in the hope that it will be
 * useful, but WITHOUT ANY WARRANTY; without even the implied warranty of
 * MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. Warranties,
 * indemnification and commercial support are all available through the
 * Npcap OEM program--see https://nmap.org/oem/
 *
 ***************************************************************************/
```

This is a function definition for a function called `send_router_solicitation`. It appears to be a part of the `libipv6` library in the IPv6 protocol.

The function takes an initial IPv6 packet as input, and then modifies it to create a new packet with a defined type of ICMPv6 message. If the initial packet is a router solicitation message, the function will use the modified packet to send a router solicitation message back to the sender of the initial packet.

The function contains several helper functions for setting the various fields of the IPv6 header, such as the packet's length, the source and destination addresses, and the ICMPv6 type. It also includes a helper function for setting the IPv6 options in the header and options for the options.

It is not visible from the outside where this function is defined and how it is used, but it appears to be an essential part of the `libipv6` library in the IPv6 protocol.


```cpp
/* This code was originally part of the Nping tool.                        */

#ifndef ICMPv6HEADER_H
#define ICMPv6HEADER_H 1

#include "ICMPHeader.h"

/******************************************************************************/
/*               IMPORTANT INFORMATION ON HOW TO USE THIS CLASS.              */
/******************************************************************************/
/* This class represents an ICMPv6 messages. ICMPv6 messages may be of
 * different types. Each type has its own header and possibly a variable
 * length data field. Information messages have an "invoking packet" field
 * which is the IP packet that triggered the emission of the ICMPv6 message.
 * Other messages may contain a "data" field, like echo requests an replies.
 * Some others may contain ICMPv6 Options.
 *
 * So the thing is, that this class only represents fixed-length ICMPv6
 * headers and does NOT offer storage for ANY variable-length field. This
 * fields may be added to the ICMPv6 header using instances of the RawData
 * class the ICMPv6Option class or even the IPv6Header class (in those cases
 * where a whole packet is appendend to the ICMPv6 message).
 *
 * So, how does this work? Let's look at some examples.
 *
 * 1. Imagine we need to build an ICMP echo request message that includes some
 *    arbitrary data to be echoed. We could do the following:
 *
 *    u8 final_packet[1024];         <-- Buffer to store the resulting packet
 *    u32 final_packet_len=0;        <-- Length of the resulting packet
 *    ICMPv6Header header;           <-- The ICMPv6 fixed-length part
 *    RawData data;                  <-- The data to append to the echo message
 *
 *    header.setType(ICMPv6_ECHO);   <-- Set ICMPv6 type to "Echo request"
 *    data.store("1234567890");      <-- Store data we need to send.
 *    header.setNextElement(&data);  <-- Tell ICMPv6Header what's after it
 *    header.setSum();               <-- Compute the checksum
 *
 *    final_packet_len=header.dumpToBinaryBuffer(fina_packet, 1024);
 *    send_packet(final_packet, final_packet_len)
 *
 * 2. If we are sending a parameter problem message and we need to include the
 *    invoking datagram, we can call setNextElement() passing an IPv6Header
 *    pointer.
 *
 *    u8 final_packet[1024];         <-- Buffer to store the resulting packet
 *    u32 final_packet_len=0;        <-- Length of the resulting packet
 *    ICMPv6Header header;           <-- The ICMPv6 fixed-length part
 *    IPv6Header ipv6;               <-- The IPv6 packet that triggered ICMPv6
 *
 *    header.setType(ICMPv6_PARAMPROB); <-- Set ICMPv6 type to "Param Problem"
 *    header.setNextElement(&ipv6);  <-- Tell ICMPv6Header what's after it
 *    header.setSum();               <-- Compute the checksum
 *
 *    Note that here we don't show how the ipv6 object is set.
 *
 * 3. If we are sending a router solicitation message, we'll call
 *    setNextElement() passing an IPv6Options Pointer.
 *
 *    u8 final_packet[1024];         <-- Buffer to store the resulting packet
 *    u32 final_packet_len=0;        <-- Length of the resulting packet
 *    ICMPv6Header header;           <-- The ICMPv6 fixed-length part
 *    IPv6Options opts1;              <-- IPv6 options
 *    IPv6Options opts2;              <-- IPv6 options
 *    IPv6Options opts3;              <-- IPv6 options
 *
 *    header.setType(ICMPv6_ROUTERSOLICIT); <-- Set ICMPv6 type
 *
 *    opts1.setXXXX();   <-- Set up the options
 *    .
 *    .
 *    .
 *    opts3.setYYYY();
 *
 *    opts2.setNextElement(&opts3);  <-- Link the options
 *    opts1.setNextElement(&opts2);
 *    header.setNextElement(&opts1);
 *    header.setNextElement(&ipv6);  <-- Link the first option to the ICMPv6
 *    header.setSum();               <-- Compute the checksum
 *
 *    And so on...
 *
 */


```

这段代码定义了一系列 ICMP（Internet Control Message Protocol）类型和码，用于在IPv6分组头中传输信息。以下是这些类型的简要说明：

1. ICMPv6_UNREACH：表示目的无法到达（destination unreachable）。这个类型由 IANA（Internet Engineering Task Force）定义，可以在其编号为 RFC 2463 和 RFC 4443 的 RFC  documents 中找到完整列表。
2. ICMPv6_UNREACH_NO_ROUTE：表示无法到达目的地，但条目已经存在（No route to destination）。这个类型由 IANA 定义，并且其含义与 ICMPv6_UNREACH 相同。
3. ICMPv6_UNREACH_PROHIBITED：表示通信管理器禁止发送数据包到这个地址（Communication administratively prohibited）。这个类型由 IANA 定义，其含义与 ICMPv6_UNREACH 相同。
4. ICMPv6_UNREACH_BEYOND_SCOPE：表示源地址不在活动地址范围内（Beyond scope of source address）。这个类型由 IANA 定义，其含义与 ICMPv6_UNREACH 相同。
5. ICMPv6_UNREACH_ADDR_UNREACH：表示目标地址不可达（Address unreachable）。这个类型由 IANA 定义，其含义与 ICMPv6_UNREACH 相同。


```cpp
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
```

This is a list of valid 6-to-8 port ICMP (Internet Control Message Protocol) values that could be used in ingress and egress policies for IPv6 (Internet Set-up Protocol version 6) networks.

The first three values (ICMPv6_UNREACH_REJECT_ROUTE, ICMPv6_PKTTOOBIG, and ICMPv6_TIMXCEED) are defined in RFC4443 and represent examples of how an ingress or egress policy might reject a traffic packet. The next three values (ICMPv6_TIMXCEED, ICMPv6_TIMXCEED_HOP_EXCEEDED, and ICMPv6_TIMXCEED_REASS_EXCEEDED) are also defined in RFC4443 and represent examples of how an ingress or egress policy might time out. The last two values (ICMPv6_PARAMPROB, and ICMPv6_PARAMPROB_FIELD, ICMPv6_PARAMPROB_NEXT_HDR, ICMPv6_PARAMPROB_OPTION) are defined in RFC2463 and RFC4443 and represent examples of how an ingress or egress policy might verify or validate parameters passed in.


```cpp
#define     ICMPv6_UNREACH_PORT_UNREACH    (4)   /*  --> Port unreachable */
#define     ICMPv6_UNREACH_SRC_ADDR_FAILED (5)   /*  --> Source address failed ingress/egress policy [RFC4443] */
#define     ICMPv6_UNREACH_REJECT_ROUTE    (6)   /*  --> Reject route to destination  [RFC4443] */
#define ICMPv6_PKTTOOBIG                    2    /* Packet too big  [RFC 2463, 4443] */
#define ICMPv6_TIMXCEED                     3    /* Time exceeded  [RFC 2463, 4443] */
#define     ICMPv6_TIMXCEED_HOP_EXCEEDED   (0)   /*  --> Hop limit exceeded in transit */
#define     ICMPv6_TIMXCEED_REASS_EXCEEDED (1)   /*  --> Fragment reassembly time exceeded */
#define ICMPv6_PARAMPROB                    4    /* Parameter problem  [RFC 2463, 4443] */
#define     ICMPv6_PARAMPROB_FIELD         (0)   /*  --> Erroneous header field encountered */
#define     ICMPv6_PARAMPROB_NEXT_HDR      (1)   /*  --> Unrecognized Next Header type encountered */
#define     ICMPv6_PARAMPROB_OPTION        (2)   /*  --> Unrecognized IPv6 option encountered */
#define ICMPv6_ECHO                        128   /* Echo request  [RFC 2463, 4443] */
#define ICMPv6_ECHOREPLY                   129   /* Echo reply  [RFC 2463, 4443] */
#define ICMPv6_GRPMEMBQUERY                130   /* Group Membership Query  [RFC 2710] */
#define ICMPv6_GRPMEMBREP                  131   /* Group Membership Report  [RFC 2710] */
```

This is a list of 22 sub-integers in the Internet Control Message Protocol (ICMPv6) that are defined in RFC 2461. These sub-integers define various features and values that can be included in ICMPv6 messages.


```cpp
#define ICMPv6_GRPMEMBRED                  132   /* Group Membership Reduction  [RFC 2710] */
#define ICMPv6_ROUTERSOLICIT               133   /* Router Solicitation  [RFC 2461] */
#define ICMPv6_ROUTERADVERT                134   /* Router Advertisement  [RFC 2461] */
#define ICMPv6_NGHBRSOLICIT                135   /* Neighbor Solicitation  [RFC 2461] */
#define ICMPv6_NGHBRADVERT                 136   /* Neighbor Advertisement  [RFC 2461] */
#define ICMPv6_REDIRECT                    137   /* Redirect  [RFC 2461] */
#define ICMPv6_RTRRENUM                    138   /* Router Renumbering  [RFC 2894] */
#define     ICMPv6_RTRRENUM_COMMAND        (0)   /*  --> Router Renumbering Command */
#define     ICMPv6_RTRRENUM_RESULT         (1)   /*  --> Router Renumbering Result */
#define     ICMPv6_RTRRENUM_SEQ_RESET      (255) /* Sequence Number Reset */
#define ICMPv6_NODEINFOQUERY               139   /* ICMP Node Information Query  [RFC 4620] */
#define     ICMPv6_NODEINFOQUERY_IPv6ADDR  (0)   /*  --> The Data field contains an IPv6 address */
#define     ICMPv6_NODEINFOQUERY_NAME      (1)   /*  --> The Data field contains a name */
#define     ICMPv6_NODEINFOQUERY_IPv4ADDR  (2)   /*  --> The Data field contains an IPv4 address */
#define ICMPv6_NODEINFORESP                140   /* ICMP Node Information Response  [RFC 4620] */
```

QType of the Query is Unknown
IMCPv6_INVNGHBRSOLICIT  141
IMCPv6_INVNGHBRADVERT    142
ICMPv6_MLDV2                143
ICMPv6_AGENTDISCOVREQ   144
ICMPv6_AGENTDISCOVREPLY 145
ICMPv6_MOBPREFIXSOLICIT   146
ICMPv6_MOBPREFIXADVERT   147
ICMPv6_CERTPATHSOLICIT 148
ICMPv6_MRDADVERT        151
ICMPv6_MRDSOLICIT       152

These are all defined in RFC 3122 and RFC 4286. They are used in various IPv6 unicast and multicast traffic reports.


```cpp
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
```

这段代码定义了两个头文件，ICMPv6_MRDTERMINATE和ICMPv6_FMIPV6，以及一个名为NI的节点信息参数。

ICMPv6_MRDTERMINATE定义了Mrduv6协议中的一个术语，表示主动选择终止发送该术语消息的节点。

ICMPv6_FMIPV6定义了用于在IPv6链路上发送 FMIPv6 消息的函数。

NI是一个包含一系列查询类型的头文件，用于描述N node（节点）信息。这些查询类型包括Noop、Unused、Nodename、Ipv4Addrs和N叶节点等。

此外，还定义了一个NI的NONCE_LEN，表示在请求和响应中发送确认消息的长度。


```cpp
#define ICMPv6_MRDTERMINATE                153   /* MRD, Multicast Router Termination  [RFC 4286] */
#define ICMPv6_FMIPV6                      154   /* FMIPv6 messages  [RFC 5568] */

/* Node Information parameters */
/* -> Query types */
#define NI_QTYPE_NOOP      0
#define NI_QTYPE_UNUSED    1
#define NI_QTYPE_NODENAME  2
#define NI_QTYPE_NODEADDRS 3
#define NI_QTYPE_IPv4ADDRS 4
/* -> Misc */
#define NI_NONCE_LEN 8

/* Nping ICMPv6Header Class internal definitions */
#define ICMPv6_COMMON_HEADER_LEN    4
```

Yes, you are correct. The values you've listed are the minimum header lengths for various Internet Control Message Protocol (ICMPv6) messages, including the Echo (ICMPv6)|Echo-A (ICMPv6)|AWS (ICMPv6)|WIL (ICMPv6)|REDIRECT (ICMPv6)|RTR (ICMPv6)|TIMXCEED (ICMPv6)|ECHO (ICMPv6)|ECHOREPLY (ICMPv6)|ROUTERSOLICIT (ICMPv6)|ROUTERADVERT (ICMPv6)|NGHBRSOLICIT (ICMPv6)|NGHBRADVERT (ICMPv6)|REDIRECT (ICMPv6)|RTRRENENUM (ICMPv6)|NODEINFO (ICMPv6)|MLD (ICMPv6) messages.

The header lengths are defined in the ICMPv6 protocol specification, which is ISO 86359.


```cpp
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
```


ICMPv6Header is a class in the Internet Control Message Protocol version 6 that is used to hold information about a network interface. It is used to describe the attributes of a network interface, such as its address and options. The ICMPv6Header class includes methods to set and get various fields of the header, such as the protocol type, source and destination addresses, segment numbers, and timers. It also includes methods to query for additional information about the interface, such as the maximum delay and the nonce value.

The ICMPv6Header class is defined in the `icmp6_80214_ification.h` header file, which is part of the IPv6 framework. It is defined as follows:
```cpp
#include "ip6.h"

struct ICMPv6Header {
   u8                  valid : 1;
   u8                  hard_to_validate : 1;
   u8                  address_string_view : 1;
   u8                  protocol_type : 5;
   u8                  source_address : 3;
   u8                  destination_address : 3;
   u8                  segment_number : 3;
   u8                  time_sent : 1;
   u8                  time_expired : 1;
   u8                  retransmit_count : 1;
   u8                  proxy_segment : 1;
   u8                  encapsulation_compression : 1;
   u8                  anti_spoof : 1;
   u8                  rfc3591_action : 1;
   u8                 
   u8                  ...
};
```
The `valid` field is a boolean value that indicates whether the header is valid. The `hard_to_validate` field is a boolean value that indicates whether the header must be validated by the receiver. The `address_string_view` field is a boolean value that indicates whether the source or destination addresses are valid. The `protocol_type` field is a 16-bit field that specifies the protocol type of the message. The `source_address` and `destination_address` fields are 32-bit fields that specify the source and destination addresses of the message, respectively. The `segment_number` field is a 32-bit field that specifies the segment number of the message. The `time_sent` and `time_expired` fields are 16-bit fields that specify whether the message was sent as an explicit request or as a response. The `retransmit_count` field is a 16-bit field that specifies the number of times the message has been sent as a response. The `proxy_segment` field is a 16-bit field that specifies whether the message is being sent through a proxy. The `encapsulation_compression` field is a 16-bit field that specifies whether the message is compressed using thecompression field. The `anti_spoof` field is a 16-bit field that specifies whether the message is being sent to prevent spoofing of the source address. The `rfc3591_action` field is a 32-bit field that specifies the action to be taken in case of a message that does not meet the requirements of the RFC 3591. The `...` field includes additional fields that are not defined here.

The ICMPv6Header class is used to describe the attributes of a network interface in the IPv6 framework. It is defined


```cpp
/* This must the MAX() of all values defined above*/
#define ICMPv6_MAX_MESSAGE_BODY     (ICMPv6_REDIRECT_LEN-ICMPv6_COMMON_HEADER_LEN)



/* Node Information flag bitmaks */
#define ICMPv6_NI_FLAG_T    0x01
#define ICMPv6_NI_FLAG_A    0x02
#define ICMPv6_NI_FLAG_C    0x04
#define ICMPv6_NI_FLAG_L    0x08
#define ICMPv6_NI_FLAG_G    0x10
#define ICMPv6_NI_FLAG_S    0x20

class ICMPv6Header : public ICMPHeader {

        /**********************************************************************/
        /* COMMON ICMPv6 packet HEADER                                        */
        /**********************************************************************/
        /* +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
           |     Type      |     Code      |          Checksum             |
           +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
           |                                                               |
           +                         Message Body                          +
           |                                                               | */
        struct nping_icmpv6_hdr{
            u8 type;
            u8 code;
            u16 checksum;
            u8 data[ICMPv6_MAX_MESSAGE_BODY];
        }__attribute__((__packed__));
        typedef struct nping_icmpv6_hdr nping_icmpv6_hdr_t;


        /**********************************************************************/
        /* ICMPv6 MESSAGE SPECIFIC HEADERS                                    */
        /**********************************************************************/

        /* Destination Unreachable Message
          +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
          |     Type      |     Code      |          Checksum             |
          +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
          |                             Unused                            |
          +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
          |                    As much of invoking packet                 |
          +                as possible without the ICMPv6 packet          +
          |                exceeding the minimum IPv6 MTU [IPv6]          | */
        struct dest_unreach_msg{
            u32 unused;
            //u8 invoking_pkt[?];
        }__attribute__((__packed__));
        typedef struct dest_unreach_msg dest_unreach_msg_t;


        /* Packet Too Big Message
          +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
          |     Type      |     Code      |          Checksum             |
          +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
          |                             MTU                               |
          +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
          |                    As much of invoking packet                 |
          +               as possible without the ICMPv6 packet           +
          |               exceeding the minimum IPv6 MTU [IPv6]           | */
        struct pkt_too_big_msg{
            u32 mtu;
            //u8 invoking_pkt[?];
        }__attribute__((__packed__));
        typedef struct pkt_too_big_msg pkt_too_big_msg_t;


        /* Time Exceeded Message
          +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
          |     Type      |     Code      |          Checksum             |
          +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
          |                             Unused                            |
          +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
          |                    As much of invoking packet                 |
          +               as possible without the ICMPv6 packet           +
          |               exceeding the minimum IPv6 MTU [IPv6]           | */
        struct time_exceeded_msg{
            u32 unused;
            //u8 invoking_pkt[?];
        }__attribute__((__packed__));
        typedef struct time_exceeded_msg time_exceeded_msg_t;


        /* Parameter Problem Message
          +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
          |     Type      |     Code      |          Checksum             |
          +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
          |                            Pointer                            |
          +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
          |                    As much of invoking packet                 |
          +               as possible without the ICMPv6 packet           +
          |               exceeding the minimum IPv6 MTU [IPv6]           | */
        struct parameter_problem_msg{
            u32 pointer;
            //u8 invoking_pkt[?];
        }__attribute__((__packed__));
        typedef struct parameter_problem_msg parameter_problem_msg_t;


        /* Echo Request/Response Messages
          +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
          |     Type      |     Code      |          Checksum             |
          +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
          |           Identifier          |        Sequence Number        |
          +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
          |     Data ...
          +-+-+-+-+-                                                        */
        struct echo_msg{
            u16 id;
            u16 seq;
            //u8 data[?];
        }__attribute__((__packed__));
        typedef struct echo_msg echo_msg_t;

        /* Router Advertisement Message
          +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
          |     Type      |     Code      |          Checksum             |
          +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
          | Cur Hop Limit |M|O|H|Prf|P|R|R|       Router Lifetime         |
          +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
          |                         Reachable Time                        |
          +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
          |                          Retrans Timer                        |
          +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
          |   Options ...
          +-+-+-+-+-+-+-+-+-+-+-+-                                          */
        struct router_advert_msg{
            u8 current_hop_limit;
            u8 autoconfig_flags; /* See RFC 5175 */
            u16 router_lifetime;
            u32 reachable_time;
            u32 retransmission_timer;
            //u8 icmpv6_options[?];
        }__attribute__((__packed__));
        typedef struct router_advert_msg router_advert_msg_t;


        /* Router Solicitation Message
          +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
          |     Type      |     Code      |          Checksum             |
          +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
          |                            Reserved                           |
          +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
          |   Options ...
          +-+-+-+-+-+-+-+-+-+-+-+-                                          */
        struct router_solicit_msg{
            u32 reserved;
            //u8 icmpv6_options[?];
        }__attribute__((__packed__));
        typedef struct router_solicit_msg router_solicit_msg_t;


        /* Neighbor Advertisement Message
          +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
          |     Type      |     Code      |          Checksum             |
          +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
          |R|S|O|                     Reserved                            |
          +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
          |                                                               |
          +                                                               +
          |                                                               |
          +                       Target Address                          +
          |                                                               |
          +                                                               +
          |                                                               |
          +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
          |   Options ...
          +-+-+-+-+-+-+-+-+-+-+-+-                                          */
        struct neighbor_advert_msg{
            u8 flags;
            u8 reserved[3];
            u8 target_address[16];
            //u8 icmpv6_options[?];
        }__attribute__((__packed__));
        typedef struct neighbor_advert_msg neighbor_advert_msg_t;


        /* Neighbor Solicitation Message
          +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
          |     Type      |     Code      |          Checksum             |
          +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
          |                           Reserved                            |
          +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
          |                                                               |
          +                                                               +
          |                                                               |
          +                       Target Address                          +
          |                                                               |
          +                                                               +
          |                                                               |
          +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
          |   Options ...
          +-+-+-+-+-+-+-+-+-+-+-+- */
        struct neighbor_solicit_msg{
            u32 reserved;
            u8 target_address[16];
            //u8 icmpv6_options[?];
        }__attribute__((__packed__));
        typedef struct neighbor_solicit_msg neighbor_solicit_msg_t;


        /* Redirect Message
          +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
          |     Type      |     Code      |          Checksum             |
          +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
          |                           Reserved                            |
          +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
          |                                                               |
          +                                                               +
          |                                                               |
          +                       Target Address                          +
          |                                                               |
          +                                                               +
          |                                                               |
          +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
          |                                                               |
          +                                                               +
          |                                                               |
          +                     Destination Address                       +
          |                                                               |
          +                                                               +
          |                                                               |
          +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
          |   Options ...
          +-+-+-+-+-+-+-+-+-+-+-+-                                          */
        struct redirect_msg{
            u32 reserved;
            u8 target_address[16];
            u8 destination_address[16];
            //u8 icmpv6_options[?];
        }__attribute__((__packed__));
        typedef struct redirect_msg redirect_msg_t;


        /* Router Renumbering Header
          +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
          |     Type      |     Code      |            Checksum           |
          +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
          |                        SequenceNumber                         |
          +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
          | SegmentNumber |     Flags     |            MaxDelay           |
          +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
          |                           reserved                            |
          +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
          |                                                               |
          /                       RR Message Body                         /
          |                                                               |
          +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ */
        struct router_renumbering_msg{
            u32 seq;
            u8 segment_number;
            u8 flags;
            u16 max_delay;
            u32 reserved;
            //u8 rr_msg_body[?];
        }__attribute__((__packed__));
        typedef struct router_renumbering_msg router_renumbering_msg_t;


         /* Node Information Queries
           0                   1                   2                   3
           0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
          +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
          |     Type      |     Code      |           Checksum            |
          +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
          |             Qtype             |       unused      |G|S|L|C|A|T|
          +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
          |                                                               |
          +                             Nonce                             +
          |                                                               |
          +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
          |                                                               |
          /                             Data                              /
          |                                                               |
          +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ */
        struct nodeinfo_msg{
            u16 qtype;
            u16 flags;
            u64 nonce;
            //u8 data[?];
        }__attribute__((__packed__));
        typedef struct nodeinfo_msg nodeinfo_msg_t;


        /* Multicast Listener Discovery
          0                   1                   2                   3
          0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
         +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
         |     Type      |     Code      |          Checksum             |
         +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
         |     Maximum Response Delay    |          Reserved             |
         +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
         |                                                               |
         +                                                               +
         |                                                               |
         +                       Multicast Address                       +
         |                                                               |
         +                                                               +
         |                                                               |
         +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ */
        struct mld_msg{
            u16 max_response_delay;
            u16 reserved;
            u8 mcast_address[16];
        }__attribute__((__packed__));
        typedef struct mld_msg mld_msg_t;


        nping_icmpv6_hdr_t h;

        /* Helper pointers */
        dest_unreach_msg_t       *h_du;
        pkt_too_big_msg_t        *h_ptb;
        time_exceeded_msg_t      *h_te;
        parameter_problem_msg_t  *h_pp;
        echo_msg_t               *h_e;
        router_advert_msg_t      *h_ra;
        router_solicit_msg_t     *h_rs;
        neighbor_advert_msg_t    *h_na;
        neighbor_solicit_msg_t   *h_ns;
        redirect_msg_t           *h_r;
        router_renumbering_msg_t *h_rr;
        nodeinfo_msg_t           *h_ni;
        mld_msg_t                *h_mld;

    public:
        ICMPv6Header();
        ~ICMPv6Header();
        void reset();
        u8 *getBufferPointer();
        int storeRecvData(const u8 *buf, size_t len);
        int protocol_id() const;
        int validate();
        int print(FILE *output, int detail) const;

        /* ICMP Type */
        int setType(u8 val);
        u8 getType() const;
        bool validateType();
        bool validateType(u8 val);

        /* Code */
        int setCode(u8 c);
        u8 getCode() const;
        bool validateCode();
        bool validateCode(u8 type, u8 code);

        /* Checksum */
        int setSum();
        int setSum(u16 s);
        int setSumRandom();
        u16 getSum() const;

        int setReserved(u32 val);
        u32 getReserved() const;
        int setUnused(u32 val);
        u32 getUnused() const;

        int setFlags(u8 val);
        u8 getFlags() const;

        int setMTU(u32 mtu);
        u32 getMTU() const;

        /* Parameter problem */
        int setPointer(u32 val);
        u32 getPointer() const;

        /* Echo */
        int setIdentifier(u16 val);
        u16 getIdentifier() const;
        int setSequence(u16 val);
        int setSequence(u32 val);
        u32 getSequence() const;

        /* Router Advertisement */
        int setCurrentHopLimit(u8 val);
        u8 getCurrentHopLimit() const;

        int setRouterLifetime(u16 val);
        u16 getRouterLifetime() const;

        int setReachableTime(u32 val);
        u32 getReachableTime() const;

        int setRetransmissionTimer(u32 val);
        u32 getRetransmissionTimer() const;

        int setTargetAddress(struct in6_addr addr);
        struct in6_addr getTargetAddress() const;

        int setDestinationAddress(struct in6_addr addr);
        struct in6_addr getDestinationAddress() const;

        int setSegmentNumber(u8 val);
        u8 getSegmentNumber() const;

        int setMaxDelay(u16 val);
        u16 getMaxDelay() const;

        /* Node Information Queries */
        int setQtype(u16 val);
        u16 getQtype() const;
        int setNodeInfoFlags(u16 val);
        u16 getNodeInfoFlags() const;
        int  setG(bool flag_value=true);
        bool getG() const;
        int  setS(bool flag_value=true);
        bool getS() const;
        int  setL(bool flag_value=true);
        bool getL() const;
        int  setC(bool flag_value=true);
        bool getC() const;
        int  setA(bool flag_value=true);
        bool getA() const;
        int  setT(bool flag_value=true);
        bool getT() const;
        int setNonce(u64 nonce_value);
        int setNonce(const u8 *nonce);
        u64 getNonce() const;

        /* Multicast Listener Discovery */
        int setMulticastAddress(struct in6_addr addr);
        struct in6_addr getMulticastAddress() const;

        /* Misc */
        int getHeaderLengthFromType(u8 type) const;
        bool isError() const;
        const char *type2string(int type, int code) const;

}; /* End of class ICMPv6Header */

```

这段代码是一个 preprocessed 预处理指令，其作用是在源代码文件（此处假设为 "example.c"）被编译之前，检查是否定义了头文件 "example.h"。

具体地，当源代码文件 "example.c" 被编译时，如果 "example.h" 已经被定义过，那么编译器会忽略该头文件。如果 "example.h"没有被定义过，则编译器会报错，指出该头文件不存在，从而提醒开发者在代码中添加 "example.h"。

因此，该代码的作用是确保在编译之前定义了头文件，以避免编译错误。


```cpp
#endif

```

# `libnetutil/ICMPv6Option.h`

This is a text that provides information about the Nmap software and its licensing. Nmap is a network scanner that allows users to scan networks and identify potential problems. The licensing for Nmap is discussed in the text.

The text explains that the official Nmap Windows builds include the Npcap software for packet capture and transmission. The Npcap software is under a different license term that forks Nmap's source code. The text also mentions that Nmap's free version is distributed with a warranty that does not cover commercial use.

The text also provides information about the Nmap OEM Edition. This edition of Nmap includes commercial features and has a more permissive license term than the free version. The text encourages companies to use the Nmap OEM Edition for commercial purposes.

Furthermore, the text explains that if a company has received an Nmap license agreement or contract with other terms, they may use and redistribute Nmap under those terms. However, the text also warns that the official Nmap Windows builds, the Npcap software, and the source code are under separate license terms that prohibit redistribution without special permission.


```cpp
/***************************************************************************
 * ICMPv6Option.h -- The ICMPv6Option Class represents an ICMP version 6   *
 * option. It contains methods to set any header field. In general, these  *
 * methods do error checkings and byte order conversion.                   *
 *                                                                         *
 ***********************IMPORTANT NMAP LICENSE TERMS************************
 *
 * The Nmap Security Scanner is (C) 1996-2023 Nmap Software LLC ("The Nmap
 * Project"). Nmap is also a registered trademark of the Nmap Project.
 *
 * This program is distributed under the terms of the Nmap Public Source
 * License (NPSL). The exact license text applying to a particular Nmap
 * release or source code control revision is contained in the LICENSE
 * file distributed with that version of Nmap or source code control
 * revision. More Nmap copyright/legal information is available from
 * https://nmap.org/book/man-legal.html, and further information on the
 * NPSL license itself can be found at https://nmap.org/npsl/ . This
 * header summarizes some key points from the Nmap license, but is no
 * substitute for the actual license text.
 *
 * Nmap is generally free for end users to download and use themselves,
 * including commercial use. It is available from https://nmap.org.
 *
 * The Nmap license generally prohibits companies from using and
 * redistributing Nmap in commercial products, but we sell a special Nmap
 * OEM Edition with a more permissive license and special features for
 * this purpose. See https://nmap.org/oem/
 *
 * If you have received a written Nmap license agreement or contract
 * stating terms other than these (such as an Nmap OEM license), you may
 * choose to use and redistribute Nmap under those terms instead.
 *
 * The official Nmap Windows builds include the Npcap software
 * (https://npcap.com) for packet capture and transmission. It is under
 * separate license terms which forbid redistribution without special
 * permission. So the official Nmap Windows builds may not be redistributed
 * without special permission (such as an Nmap OEM license).
 *
 * Source is provided to this software because we believe users have a
 * right to know exactly what a program is going to do before they run it.
 * This also allows you to audit the software for security holes.
 *
 * Source code also allows you to port Nmap to new platforms, fix bugs, and add
 * new features. You are highly encouraged to submit your changes as a Github PR
 * or by email to the dev@nmap.org mailing list for possible incorporation into
 * the main distribution. Unless you specify otherwise, it is understood that
 * you are offering us very broad rights to use your submissions as described in
 * the Nmap Public Source License Contributor Agreement. This is important
 * because we fund the project by selling licenses with various terms, and also
 * because the inability to relicense code has caused devastating problems for
 * other Free Software projects (such as KDE and NASM).
 *
 * The free version of Nmap is distributed in the hope that it will be
 * useful, but WITHOUT ANY WARRANTY; without even the implied warranty of
 * MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. Warranties,
 * indemnification and commercial support are all available through the
 * Npcap OEM program--see https://nmap.org/oem/
 *
 ***************************************************************************/
```

这段代码定义了一个名为“__ICMPv6OPTION_H__”的标识，其值为1。这个标识表明它属于IPv6选项头的一部分。

接着，它引入了来自IETF RFC 2461和RFC 2894的包头图。

然后，定义了几个IPv6选项类型，包括：ICMPv6_OPTION_SRC_LINK_ADDR，用于指定数据包的源IPv6地址。

最后，没有其他操作，而是定义了这些选项类型。


```cpp
/* This code was originally part of the Nping tool.                        */

#ifndef __ICMPv6OPTION_H__
#define __ICMPv6OPTION_H__ 1

#include "NetworkLayerElement.h"

/* Packet header diagrams included in this file have been taken from the
 * following IETF RFC documents: RFC 2461, RFC 2894 */

/* The following codes have been defined by IANA. A complete list may be found
 * at http://www.iana.org/assignments/icmpv6-parameters */

/* ICMPv6 Option Types */
#define ICMPv6_OPTION_SRC_LINK_ADDR 1
```

这段代码定义了一个名为“ICMPv6_OPTION_TGT_LINK_ADDR”的宏，它的值为2；定义了一个名为“ICMPv6_OPTION_PREFIX_INFO”的宏，它的值为3；定义了一个名为“ICMPv6_OPTION_REDIR_HDR”的宏，它的值为4；定义了一个名为“ICMPv6_OPTION_MTU”的宏，它的值为6。

这些宏用于定义一个名为“ICMPv6_OPTION_TUPLE”的枚举类型，它包含了上述定义的四个成员变量。这个枚举类型在后续的代码中被用来表示ICMPv6选项的含义。


```cpp
#define ICMPv6_OPTION_TGT_LINK_ADDR 2
#define ICMPv6_OPTION_PREFIX_INFO   3
#define ICMPv6_OPTION_REDIR_HDR     4
#define ICMPv6_OPTION_MTU           5

/* Nping ICMPv6Options Class internal definitions */
#define ICMPv6_OPTION_COMMON_HEADER_LEN    2
#define ICMPv6_OPTION_MIN_HEADER_LEN       8
#define ICMPv6_OPTION_SRC_LINK_ADDR_LEN    (ICMPv6_OPTION_COMMON_HEADER_LEN+6)
#define ICMPv6_OPTION_TGT_LINK_ADDR_LEN    (ICMPv6_OPTION_COMMON_HEADER_LEN+6)
#define ICMPv6_OPTION_PREFIX_INFO_LEN      (ICMPv6_OPTION_COMMON_HEADER_LEN+30)
#define ICMPv6_OPTION_REDIR_HDR_LEN        (ICMPv6_OPTION_COMMON_HEADER_LEN+6)
#define ICMPv6_OPTION_MTU_LEN              (ICMPv6_OPTION_COMMON_HEADER_LEN+6)
/* This must the MAX() of all values defined above*/
#define ICMPv6_OPTION_MAX_MESSAGE_BODY     (ICMPv6_OPTION_PREFIX_INFO_LEN-ICMPv6_OPTION_COMMON_HEADER_LEN)

```

This is a struct definition for an ICMPv6Option that defines the options to be included in an ICMPv6 packet. It contains the following fields:

* reserved: A 16-bit field that is set to zero by the factory.
* mtu: A 32-bit field that stores the MTU (maximum transmission unit) of the packet.

This struct is used to pack multiple option fields into a single byte buffer, which is then sent in the ICMPv6 packet.

You can


```cpp
#define ICMPv6_OPTION_LINK_ADDRESS_LEN 6

class ICMPv6Option : public NetworkLayerElement {

    private:

        /**********************************************************************/
        /* COMMON ICMPv6 OPTION HEADER                                        */
        /**********************************************************************/

        struct nping_icmpv6_option{
            u8 type;
            u8 length;
            u8 data[ICMPv6_OPTION_MAX_MESSAGE_BODY];
        }__attribute__((__packed__));
        typedef struct nping_icmpv6_option nping_icmpv6_option_t;

        /**********************************************************************/
        /* ICMPv6 OPTION FORMATS                                              */
        /**********************************************************************/
        /* +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
           |     Type      |    Length     |              ...              |
           +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
           ~                              ...                              ~
           +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ */


        /* Source/Target Link-layer Address
          +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
          |     Type      |    Length     |    Link-Layer Address ...
          +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ */
        struct link_addr_option{
            u8 link_addr[6];
        }__attribute__((__packed__));
        typedef struct link_addr_option link_addr_option_t;


        /* Prefix Information
          +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
          |     Type      |    Length     | Prefix Length |L|A| Reserved1 |
          +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
          |                         Valid Lifetime                        |
          +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
          |                       Preferred Lifetime                      |
          +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
          |                           Reserved2                           |
          +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
          |                                                               |
          +                                                               +
          |                                                               |
          +                            Prefix                             +
          |                                                               |
          +                                                               +
          |                                                               |
          +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ */
        struct prefix_info_option{
            u8 prefix_length;
            u8 flags;
            u32 valid_lifetime;
            u32 preferred_lifetime;
            u32 reserved;
            u8 prefix[16];
        }__attribute__((__packed__));
        typedef struct prefix_info_option prefix_info_option_t;


        /* Redirect Header
          +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
          |     Type      |    Length     |            Reserved           |
          +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
          |                           Reserved                            |
          +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
          |                                                               |
          ~                       IP header + data                        ~
          |                                                               |
          +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ */
        struct redirect_option{
            u16 reserved_1;
            u32 reserved_2;
            //u8 invoking_pkt[?];
        }__attribute__((__packed__));
        typedef struct redirect_option redirect_option_t;


        /* MTU
          +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
          |     Type      |    Length     |           Reserved            |
          +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
          |                              MTU                              |
          +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ */
        struct mtu_option{
            u16 reserved;
            u32 mtu;
        }__attribute__((__packed__));
        typedef struct mtu_option mtu_option_t;


        nping_icmpv6_option_t h;

        link_addr_option_t        *h_la;
        prefix_info_option_t      *h_pi;
        redirect_option_t         *h_r;
        mtu_option_t              *h_mtu;

    public:
        ICMPv6Option();
        ~ICMPv6Option();
        void reset();
        u8 *getBufferPointer();
        int storeRecvData(const u8 *buf, size_t len);
        int protocol_id() const;

        int setType(u8 val);
        u8 getType();
        bool validateType(u8 val);

        int setLength(u8 val);
        u8 getLength();

        int setLinkAddress(u8* val);
        u8 *getLinkAddress();

        int setPrefixLength(u8 val);
        u8 getPrefixLength();

        int setFlags(u8 val);
        u8 getFlags();

        int setValidLifetime(u32 val);
        u32 getValidLifetime();

        int setPreferredLifetime(u32 val);
        u32 getPreferredLifetime();

        int setPrefix(u8 *val);
        u8 *getPrefix();

        int setMTU(u32 val);
        u32 getMTU();

        int getHeaderLengthFromType(u8 type);

}; /* End of class ICMPv6Option */

```

这是一个用于C语言的预处理指令，即`#ifdef`和`#ifndef`。它的作用是在编译之前检查特定是否定义过，如果定义过则不做任何处理，如果还没有定义过则抛出编译错误。

简而言之，这段代码是在告诉编译器某个标识符是否定义过，并在编译之前进行检查。这个标识符是在一个预先定义好的标识符，如果没有定义过，编译器会报错并停止程序的编译。


```cpp
#endif

```

# `libnetutil/ICMPv6RRBody.h`

This is a text-based Nmap license that allows for the use, distribution, and modification of Nmap for non-commercial purposes. The license has certain restrictions and requirements, such as requiring that any resulting commercial product have the Nmap logo and注明 the license terms. The license also allows for the use and distribution of the Npcap software for packet capture and transmission, subject to its own license terms. The Nmap license is permissive and allows for a wide range of activities, while the Npcap license is more restrictive and requires specific授权 for commercial use.


```cpp
/***************************************************************************
 * ICMPv6RRBody.cc -- The ICMPv6RRBody Class represents an ICMP version 6  *
 * Router Renumbering message body. It contains methods to set any header  *
 * field. In general, these  methods do error checkings and byte order     *
 * conversions.                                                            *
 *                                                                         *
 ***********************IMPORTANT NMAP LICENSE TERMS************************
 *
 * The Nmap Security Scanner is (C) 1996-2023 Nmap Software LLC ("The Nmap
 * Project"). Nmap is also a registered trademark of the Nmap Project.
 *
 * This program is distributed under the terms of the Nmap Public Source
 * License (NPSL). The exact license text applying to a particular Nmap
 * release or source code control revision is contained in the LICENSE
 * file distributed with that version of Nmap or source code control
 * revision. More Nmap copyright/legal information is available from
 * https://nmap.org/book/man-legal.html, and further information on the
 * NPSL license itself can be found at https://nmap.org/npsl/ . This
 * header summarizes some key points from the Nmap license, but is no
 * substitute for the actual license text.
 *
 * Nmap is generally free for end users to download and use themselves,
 * including commercial use. It is available from https://nmap.org.
 *
 * The Nmap license generally prohibits companies from using and
 * redistributing Nmap in commercial products, but we sell a special Nmap
 * OEM Edition with a more permissive license and special features for
 * this purpose. See https://nmap.org/oem/
 *
 * If you have received a written Nmap license agreement or contract
 * stating terms other than these (such as an Nmap OEM license), you may
 * choose to use and redistribute Nmap under those terms instead.
 *
 * The official Nmap Windows builds include the Npcap software
 * (https://npcap.com) for packet capture and transmission. It is under
 * separate license terms which forbid redistribution without special
 * permission. So the official Nmap Windows builds may not be redistributed
 * without special permission (such as an Nmap OEM license).
 *
 * Source is provided to this software because we believe users have a
 * right to know exactly what a program is going to do before they run it.
 * This also allows you to audit the software for security holes.
 *
 * Source code also allows you to port Nmap to new platforms, fix bugs, and add
 * new features. You are highly encouraged to submit your changes as a Github PR
 * or by email to the dev@nmap.org mailing list for possible incorporation into
 * the main distribution. Unless you specify otherwise, it is understood that
 * you are offering us very broad rights to use your submissions as described in
 * the Nmap Public Source License Contributor Agreement. This is important
 * because we fund the project by selling licenses with various terms, and also
 * because the inability to relicense code has caused devastating problems for
 * other Free Software projects (such as KDE and NASM).
 *
 * The free version of Nmap is distributed in the hope that it will be
 * useful, but WITHOUT ANY WARRANTY; without even the implied warranty of
 * MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. Warranties,
 * indemnification and commercial support are all available through the
 * Npcap OEM program--see https://nmap.org/oem/
 *
 ***************************************************************************/
```



这段代码定义了一个名为 "ICMPv6RRBody" 的类，属于 Nping 工具。该类的定义在头文件 "ICMPv6RRBODY_H" 中。

在该类的定义中，首先包含了一个名为 "NetworkLayerElement" 的类，然后定义了一系列与 ICMPv6 包中 "RR" 字段相关的常量和宏定义。

具体来说，该类包含以下几个常量：

- ICMPv6_RR_MATCH_PREFIX_LEN：匹配 IPv6 前缀的预长度，用于在 "RR" 字段中匹配前缀。
- ICMPv6_RR_USE_PREFIX_LEN：表示是否使用 "PREFIX" 作为前缀来匹配 IPv6 前缀。
- ICMPv6_RR_RESULT_MSG_LEN：表示 "RR" 字段中包含的结果消息的最大长度。

此外，该类还包含一个名为 "generatePrefix match" 的函数，用于生成 IPv6 前缀并匹配，以及一个名为 "generateMatch" 的函数，用于生成匹配 "PREFIX" 或 "LL" 字段的匹配。


```cpp
/* This code was originally part of the Nping tool.                        */

#ifndef ICMPv6RRBODY_H
#define ICMPv6RRBODY_H 1

#include "NetworkLayerElement.h"

/* Packet header diagrams included in this file have been taken from the
 * following IETF RFC documents: RFC 2894 */

/* Nping ICMPv6RRBody Class internal definitions */
#define ICMPv6_RR_MATCH_PREFIX_LEN 24
#define ICMPv6_RR_USE_PREFIX_LEN   32
#define ICMPv6_RR_RESULT_MSG_LEN   24
/* This must the MAX() of all values defined above*/
```

This is a C++ class that implements the `ICMPv6RRBody` interface for IPv6-RFC8571 ICMP version 6 messages. This class contains the structure of the `rr_result_msg` and `h_r` variables that are used to store the received data and the related metadata, respectively.

The class has several member variables, including the `reserved` field which is a `u8` variable reserved for future use, the `flags` field which is a `u8` variable to store the flags related to the message, the `ordinal` field which is a `u8` variable to store the ordinal number of the message, the `matched_length` field which is a `u32` variable to store the length of the matched prefix, and the `interface_index` field which is a `u32` variable to store the index of the interface to which the message is addressed.

The class also has two pointer variables, `h_mp` and `h_up`, which are pointers to the `rr_match_prefix_t` structs that are used to store the matched prefixes.

The class has two methods, `reset` and `storeRecvData`, which are used to reset and store the received data, respectively.

The `ICMPv6RRBody` class is defined in terms of the `ICMPv6RRBody` header and the `rr_result_msg_t` field, which is defined in the `struct` as follows:
```cpp
struct rr_result_msg_t {
           u8 reserved;
           u8 flags;
           u8 ordinal;
           u8 matched_length;
           u32 interface_index;
           u8 matched_prefix[16];
       } __attribute__((__packed__));
```
This header file is not included in the scope of the `rr_result_msg_t` definition, but it defines the structure of the `rr_result_msg_t` field.


```cpp
#define ICMPv6_RR_MAX_LENGTH (ICMPv6_RR_USE_PREFIX_LEN)
#define ICMPv6_RR_MIN_LENGTH (ICMPv6_RR_MATCH_PREFIX_LEN)


class ICMPv6RRBody : public NetworkLayerElement {

    private:

        /**********************************************************************/
        /* COMMON ICMPv6 OPTION HEADER                                        */
        /**********************************************************************/

        struct nping_icmpv6_rr_body{
            u8 data[ICMPv6_RR_MAX_LENGTH];
        }__attribute__((__packed__));
        typedef struct nping_icmpv6_rr_body nping_icmpv6_rr_body_t;

        /**********************************************************************/
        /* ICMPv6 OPTION FORMATS                                              */
        /**********************************************************************/


        /* Match-Prefix Part

          +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
          |    OpCode     |   OpLength    |    Ordinal    |   MatchLen    |
          +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
          |    MinLen     |    MaxLen     |           reserved            |
          +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
          |                                                               |
          +-                                                             -+
          |                                                               |
          +-                         MatchPrefix                         -+
          |                                                               |
          +-                                                             -+
          |                                                               |
          +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 */
        struct rr_match_prefix{
            u8 op_code;
            u8 op_length;
            u8 ordinal;
            u8 match_length;
            u8 min_length;
            u8 max_length;
            u16 reserved;
            u8 match_prefix[16];
        }__attribute__((__packed__));
        typedef struct rr_match_prefix rr_match_prefix_t;


        /* Use-Prefix Part
          +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
          |    UseLen     |    KeepLen    |   FlagMask    |    RAFlags    |
          +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
          |                        Valid Lifetime                         |
          +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
          |                      Preferred Lifetime                       |
          +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
          |V|P|                         reserved                          |
          +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
          |                                                               |
          +-                                                             -+
          |                                                               |
          +-                          UsePrefix                          -+
          |                                                               |
          +-                                                             -+
          |                                                               |
          +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 */
        struct rr_use_prefix{
            u8 use_len;
            u8 keep_len;
            u8 flag_mask;
            u8 ra_flags;
            u32 valid_lifetime;
            u32 preferred_lifetime;
            u8 flags;
            u8 reserved[3];
            u8 use_prefix[16];
        }__attribute__((__packed__));
        typedef struct rr_use_prefix rr_use_prefix_t;


        /* Result Message

          +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
          |         reserved          |B|F|    Ordinal    |  MatchedLen   |
          +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
          |                         InterfaceIndex                        |
          +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
          |                                                               |
          +-                                                             -+
          |                                                               |
          +-                        MatchedPrefix                        -+
          |                                                               |
          +-                                                             -+
          |                                                               |
          +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ */
        struct rr_result_msg{
            u8 reserved;
            u8 flags;
            u8 ordinal;
            u8 matched_length;
            u32 interface_index;
            u8 matched_prefix[16];
        }__attribute__((__packed__));
        typedef struct rr_result_msg rr_result_msg_t;

        nping_icmpv6_rr_body_t h;

        rr_match_prefix_t *h_mp;
        rr_use_prefix_t   *h_up;
        rr_result_msg_t   *h_r;

    public:
        ICMPv6RRBody();
        ~ICMPv6RRBody();
        void reset();
        u8 *getBufferPointer();
        int storeRecvData(const u8 *buf, size_t len);

}; /* End of class ICMPv6RRBody */

```

这是一个用于C或C++的代码片段，用来定义一个函数或类的前缀定义，即当某些库或头文件在使用该函数或类时，不需要再次定义它们。通常情况下，这些定义在头文件中使用，因此该代码将确保该函数或类已经被定义，即使在该源文件中没有显式地定义它们。

具体来说，如果源文件A中定义了一个函数或类为func，头文件B中使用了该函数或类，但是在该源文件之前已经定义了该函数或类，则该代码将包含一个前缀定义，类似如下：

```cpp
#include "header.h"

#define func(param1, param2) 
```

其中，header.h是头文件B的名称，该代码将确保该头文件中包含func定义之前的内容。因此，该代码将确保header.h文件中已经定义了func函数，即使在该文件之前没有定义它。


```cpp
#endif

```

# `libnetutil/IPv4Header.h`

This is a text-based Nmap license that has some specific conditions and limitations. It is intended to be included with the official Nmap Windows builds, but it is also Agile's license and should be treated as such. This license allows companies to use the Nmap software for commercial purposes, but they are not allowed to redistribute the Nmap software under any circumstances. If a company wants to use the Nmap software for commercial purposes, they will need to purchase an Nmap OEM license from Agile. The Nmap license also includes a provision that allows users to modify and port the Nmap software to new platforms, as long as they follow the terms of the Nmap Public Source License Contributor Agreement.


```cpp
/***************************************************************************
 * IPv4Header.h -- The IPv4Header Class represents an IPv4 datagram. It    *
 * contains methods to set any header field. In general, these methods do  *
 * error checkings and byte order conversion.                              *
 *                                                                         *
 ***********************IMPORTANT NMAP LICENSE TERMS************************
 *
 * The Nmap Security Scanner is (C) 1996-2023 Nmap Software LLC ("The Nmap
 * Project"). Nmap is also a registered trademark of the Nmap Project.
 *
 * This program is distributed under the terms of the Nmap Public Source
 * License (NPSL). The exact license text applying to a particular Nmap
 * release or source code control revision is contained in the LICENSE
 * file distributed with that version of Nmap or source code control
 * revision. More Nmap copyright/legal information is available from
 * https://nmap.org/book/man-legal.html, and further information on the
 * NPSL license itself can be found at https://nmap.org/npsl/ . This
 * header summarizes some key points from the Nmap license, but is no
 * substitute for the actual license text.
 *
 * Nmap is generally free for end users to download and use themselves,
 * including commercial use. It is available from https://nmap.org.
 *
 * The Nmap license generally prohibits companies from using and
 * redistributing Nmap in commercial products, but we sell a special Nmap
 * OEM Edition with a more permissive license and special features for
 * this purpose. See https://nmap.org/oem/
 *
 * If you have received a written Nmap license agreement or contract
 * stating terms other than these (such as an Nmap OEM license), you may
 * choose to use and redistribute Nmap under those terms instead.
 *
 * The official Nmap Windows builds include the Npcap software
 * (https://npcap.com) for packet capture and transmission. It is under
 * separate license terms which forbid redistribution without special
 * permission. So the official Nmap Windows builds may not be redistributed
 * without special permission (such as an Nmap OEM license).
 *
 * Source is provided to this software because we believe users have a
 * right to know exactly what a program is going to do before they run it.
 * This also allows you to audit the software for security holes.
 *
 * Source code also allows you to port Nmap to new platforms, fix bugs, and add
 * new features. You are highly encouraged to submit your changes as a Github PR
 * or by email to the dev@nmap.org mailing list for possible incorporation into
 * the main distribution. Unless you specify otherwise, it is understood that
 * you are offering us very broad rights to use your submissions as described in
 * the Nmap Public Source License Contributor Agreement. This is important
 * because we fund the project by selling licenses with various terms, and also
 * because the inability to relicense code has caused devastating problems for
 * other Free Software projects (such as KDE and NASM).
 *
 * The free version of Nmap is distributed in the hope that it will be
 * useful, but WITHOUT ANY WARRANTY; without even the implied warranty of
 * MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. Warranties,
 * indemnification and commercial support are all available through the
 * Npcap OEM program--see https://nmap.org/oem/
 *
 ***************************************************************************/
```



这段代码定义了一个头部长度为20的IPv4Header结构体，包含了IPv4头部的标准选项字段和保留字段。其中，

* IP_RF表示保留字段，保留字段的长度为64位，用于标识是否支持保留字段。
* IP_DF表示保留字段，用于标识是否支持分片。
* IP_MF表示更多片段标志，用于标识是否支持多播。
* IP_OFFMASK表示保留字段，用于标识是否支持 fragmentation。
* IP_HEADER_LEN表示IPv4头部标准选项字段的长度，为20字节。
* MAX_IP_OPTIONS_LEN表示IPv4头部保留字段的最大长度，为40字节。

该头部长度为20的IPv4Header结构体包含了IPv4头部标准选项字段和保留字段，用于标识网络层元素的接收和发送。


```cpp
/* This code was originally part of the Nping tool.                        */

#ifndef IPV4HEADER_H
#define IPV4HEADER_H 1

#include "NetworkLayerElement.h"

#define IP_RF 0x8000               /* Reserved fragment flag         */
#define IP_DF 0x4000               /* Don't fragment flag            */
#define IP_MF 0x2000               /* More fragments flag            */
#define IP_OFFMASK 0x1fff          /* Mask for fragmenting bits      */
#define IP_HEADER_LEN 20           /* Length of the standard header  */
#define MAX_IP_OPTIONS_LEN 40      /* Max Length for IP Options      */

/* Default header values */
```

This is a C implementation of an IPv4 header that defines the structure and fields of an IPv4 header.

The header is defined as a struct with 256 fields. Here is a list of the fields and their descriptions:

* Version (4 bits): The header version, 0 for IPv4, 1 for IPv6.
* Internet Header Length (IHL) (4 bits): The header length of the internet header, 0 for IPv4, 1 for IPv6.
* Total Length (16 bits): The total length of the packet, including the header.
* Identification (16 bits): The header Checksum.
* fragmentation offsets (32 bits): The fragmentation offsets, used for fragmentation orders.
* fragmentation order (8 bits): The fragmentation order, 0 for Iterative, 1 for Non-Iterative.
*TTL (8 bits): The Time to Live (TTL) of the packet.
*Protocol (8 bits): The protocol, 0 forIP, 1 forICMP.
*Dst Protocol (8 bits): The destination protocol, 0 forIP, 1 forICMP.
*Dst Address (32 bits): The destination network address.
*Src Address (32 bits): The source network address.
*Source Port (16 bits): The source port number.
*Dest Port (16 bits): The destination port number.
*Udp Source Port (16 bits): The source port number for UDP.
*Udp Dest Port (16 bits): The destination port number for UDP.
*Sql Source Port (8 bits): The source port number for SQL.
*Sql Dest Port (8 bits): The destination port number for SQL.
*Admin Order (8 bits): The Admin Order, used for quality of service (QoS).
*Flags (32 bits): Flags.
*RF (16 bits): Flag to indicate Nagle reduce.
*DF (16 bits): Flag to indicate Nagle分发。
*MF (32 bits): Flag to indicate Marked for fragmentation。
*Frag Offset (32 bits): The fragmentation offset, used for fragmentation orders.
* fragmentation order (8 bits): The fragmentation order, 0 for Iterative, 1 for Non-Iterative。
*TTL (8 bits): The Time to Live (TTL) of the packet。
*Protocol (8 bits): The protocol, 0 forIP, 1 forICMP。
*Dst Protocol (8 bits): The destination protocol, 0 forIP, 1 forICMP。
*Dst Address (32 bits): The destination network address。
*Src Address (32 bits): The source network address。
*Source Port (16 bits): The source port number。
*Dest Port (16 bits): The destination port number。
*Udp Source Port (16 bits): The source port number for UDP。
*Udp Dest Port (16 bits): The destination port number for UDP。
*Sql Source Port (8 bits): The source port number for SQL。
*Sql Dest Port (8 bits): The destination port number for SQL。
*Admin Order (8 bits): The Admin Order, used for quality of service (QoS)。
*Flags (32 bits): Flags。
*RF (16 bits): Flag to indicate Nagle reduce。
*DF (16 bits): Flag to indicate Nagle分发。
*MF (32 bits): Flag to indicate Marked for fragmentation。
*Frag Offset (32 bits): The fragmentation offset, used for fragmentation orders。
* fragmentation order (8 bits): The fragmentation order, 0 for Iterative, 1 for Non-Iterative。
*TTL (8 bits): The Time to Live (TTL) of the packet。
*Protocol (8 bits): The protocol, 0 forIP, 1 forICMP。
*Dst Protocol (8 bits): The destination protocol, 0 forIP, 1 forICMP。
*Dst Address (32 bits): The destination network address。
*Src Address (32 bits): The source network address。
*Source Port (16 bits): The source port number。
*Dest Port (16 bits): The destination port number。
*Udp Source Port (16 bits): The source port number for UDP。
*Udp Dest Port (16 bits): The destination port number for UDP。
*Sql Source Port (8 bits): The source port number for SQL。
*Sql Dest Port (8 bits): The destination port number for SQL。
*Admin Order (8 bits): The Admin Order, used for quality of service (QoS)。
*Flags (32 bits): Flags。
*RF (16 bits): Flag to indicate Nagle reduce。
*DF (16 bits): Flag to indicate Nagle分发。
*MF (32 bits): Flag to indicate Marked for fragmentation。
*Frag Offset (32 bits): The fragmentation offset, used for fragmentation orders。
* fragmentation order (8 bits): The fragmentation order, 0 for Iterative, 1 for Non-Iterative。
*TTL (8 bits): The Time to Live (TTL) of the packet。
*Protocol (8 bits): The protocol, 0 forIP, 1 forICMP。
*Dst Protocol (8 bits): The destination protocol, 0 forIP, 1 forICMP。
*Dst Address (32 bits): The destination network address。
*Src Address (32 bits): The source network address。
*Source Port (16 bits): The source port number。
*Dest Port (16 bits): The destination port number。
*Udp Source Port (16 bits): The source port number for UDP。
*Udp Dest Port (16 bits): The destination port number for UDP。
*Sql Source Port (8 bits): The source port number for SQL。
*Sql Dest Port (8 bits): The destination port number for SQL。
*Admin Order (8 bits): The Admin Order, used for quality of service (QoS)。
*Flags (32 bits): Flags。
*RF (16 bits): Flag to indicate Nagle reduce。
*DF (16 bits): Flag to indicate Nagle分发。
*MF (32 bits): Flag to indicate Marked for fragmentation。
*Frag Offset (32 bits): The fragmentation offset, used for fragmentation orders。
* fragmentation order (8 bits): The fragmentation order, 0 for Iterative, 1 for Non-Iterative。
*TTL (8 bits): The Time to Live (TTL) of the packet。
*


```cpp
#define IPv4_DEFAULT_TOS      0
#define IPv4_DEFAULT_ID       0
#define IPv4_DEFAULT_TTL      64
#define IPv4_DEFAULT_PROTO    6 /* TCP */

class IPv4Header : public NetworkLayerElement {

    private:
        /*
         0                   1                   2                   3
         0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
        |Version|  IHL  |Type of Service|          Total Length         |
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
        |         Identification        |Flags|      Fragment Offset    |
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
        |  Time to Live |    Protocol   |         Header Checksum       |
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
        |                       Source Address                          |
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
        |                    Destination Address                        |
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
        |                    Options                    |    Padding    |
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
        */
        struct nping_ipv4_hdr {
        #if WORDS_BIGENDIAN
            u8 ip_v:4;                     /* Version                        */
            u8 ip_hl:4;                    /* Header length                  */
        #else
            u8 ip_hl:4;                    /* Header length                  */
            u8 ip_v:4;                     /* Version                        */
        #endif
            u8 ip_tos;                     /* Type of service                */
            u16 ip_len;                    /* Total length                   */
            u16 ip_id;                     /* Identification                 */
            u16 ip_off;                    /* Fragment offset field          */
            u8 ip_ttl;                     /* Time to live                   */
            u8 ip_p;                       /* Protocol                       */
            u16 ip_sum;                    /* Checksum                       */
            struct in_addr ip_src;         /* Source IP address              */
            struct in_addr ip_dst;         /* Destination IP address         */
            u8 options[MAX_IP_OPTIONS_LEN];  /* IP Options                   */
        }__attribute__((__packed__));

        typedef struct nping_ipv4_hdr nping_ipv4_hdr_t;

        nping_ipv4_hdr_t h;

        int ipoptlen; /**< Length of IP options */

    public:

        /* Misc */
        IPv4Header();
        ~IPv4Header();
        void reset();
        u8 *getBufferPointer();
        int storeRecvData(const u8 *buf, size_t len);
        int protocol_id() const;
        int validate();
        int print(FILE *output, int detail) const;

        /* IP version */
        int setVersion();
        u8 getVersion() const;

        /* Header Length */
        int setHeaderLength();
        int setHeaderLength(u8 l);
        u8 getHeaderLength() const;

        /* Type of Service */
        int setTOS(u8 v);
        u8 getTOS() const;

        /* Total length of the datagram */
        int setTotalLength();
        int setTotalLength(u16 l);
        u16 getTotalLength() const;

        /* Identification value */
        int setIdentification();
        int setIdentification(u16 i);
        u16 getIdentification() const;

        /* Fragment Offset */
        int setFragOffset();
        int setFragOffset(u16 f);
        u16 getFragOffset() const;

        /* Flags */
        int setRF();
        int unsetRF();
        bool getRF() const;
        int setDF();
        int unsetDF();
        bool getDF() const;
        int setMF();
        int unsetMF();
        bool getMF() const;

        /* Time to live */
        int setTTL();
        int setTTL(u8 t);
        u8 getTTL() const;

        /* Next protocol */
        int setNextProto(u8 p);
        int setNextProto(const char *p);
        u8 getNextProto() const;
        int setNextHeader(u8 val);
        u8 getNextHeader() const;

        /* Checksum */
        int setSum();
        int setSum(u16 s);
        int setSumRandom();
        u16 getSum() const;

        /* Destination IP */
        int setDestinationAddress(u32 d);
        int setDestinationAddress(struct in_addr d);
        const u8 *getDestinationAddress() const;
        struct in_addr getDestinationAddress(struct in_addr *result) const;


        /* Source IP */
        int setSourceAddress(u32 d);
        int setSourceAddress(struct in_addr d);
        const u8 *getSourceAddress() const;
        struct in_addr getSourceAddress(struct in_addr *result) const;

        u16 getAddressLength() const;

        /* IP Options */
        int setOpts(const char *txt);
        int setOpts(u8 *opts_buff,  u32 opts_len);
        const u8 *getOpts() const;
        const u8 *getOpts(int *len) const;
        int printOptions() const;
        const char *getOptionsString() const;

}; /* End of class IPv4Header */

```

这是一个用于预处理单个头文件boilerplate的代码。boilerplate是一个广泛使用的编程技巧，它定义了一个标准输出语句，但只有在包含此标准的头文件中，它才会被定义。

因此，这段代码的作用是检查名为boilerplate的源文件是否在当前目录中。如果是，则确保该文件包含一个标准的输出语句，如果没有，则将包含一个标准的输出语句的代码添加到boilerplate中。


```cpp
#endif

```