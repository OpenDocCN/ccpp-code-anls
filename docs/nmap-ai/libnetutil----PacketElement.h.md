# `nmap\libnetutil\PacketElement.h`

```cpp
/* 这段代码最初是 Nping 工具的一部分。*/

#ifndef PACKETELEMENT_H
#define PACKETELEMENT_H  1

#include "nbase.h"
#include "netutil.h"

#define HEADER_TYPE_IPv6_HOPOPT   0  /* IPv6 Hop-by-Hop Option                */
#define HEADER_TYPE_ICMPv4        1  /* ICMP Internet Control Message         */
#define HEADER_TYPE_IGMP          2  /* IGMP Internet Group Management        */
#define HEADER_TYPE_IPv4          4  /* IPv4 IPv4 encapsulation               */
#define HEADER_TYPE_TCP           6  /* TCP Transmission Control              */
#define HEADER_TYPE_EGP           8  /* EGP Exterior Gateway Protocol         */
#define HEADER_TYPE_UDP           17 /* UDP User Datagram                     */
#define HEADER_TYPE_IPv6          41 /* IPv6 IPv6 encapsulation               */
#define HEADER_TYPE_IPv6_ROUTE    43 /* IPv6-Route Routing Header for IPv6    */
#define HEADER_TYPE_IPv6_FRAG     44 /* IPv6-Frag Fragment Header for IPv6    */
#define HEADER_TYPE_GRE           47 /* GRE General Routing Encapsulation     */
#define HEADER_TYPE_ESP           50 /* ESP Encap Security Payload            */
#define HEADER_TYPE_AH            51 /* AH Authentication Header              */
#define HEADER_TYPE_ICMPv6        58 /* IPv6-ICMP ICMP for IPv6               */
#define HEADER_TYPE_IPv6_NONXT    59 /* IPv6-NoNxt No Next Header for IPv6    */
#define HEADER_TYPE_IPv6_OPTS     60 /* IPv6-Opts IPv6 Destination Options    */
#define HEADER_TYPE_EIGRP         88 /* EIGRP                                 */
#define HEADER_TYPE_ETHERNET      97 /* Ethernet                              */
#define HEADER_TYPE_L2TP         115 /* L2TP Layer Two Tunneling Protocol     */
#define HEADER_TYPE_SCTP         132 /* SCTP Stream Control Transmission P.   */
#define HEADER_TYPE_IPv6_MOBILE  135 /* Mobility Header                       */
#define HEADER_TYPE_MPLS_IN_IP   137 /* MPLS-in-IP                            */
# 定义不同类型的数据包头部类型
#define HEADER_TYPE_ARP         2054 /* ARP Address Resolution Protocol       */
#define HEADER_TYPE_ICMPv6_OPTION 9997 /* ICMPv6 option                       */
#define HEADER_TYPE_NEP         9998 /* Nping Echo Protocol                   */
#define HEADER_TYPE_RAW_DATA    9999 /* Raw unknown data                      */

# 定义不同的打印详细程度
#define PRINT_DETAIL_LOW   1
#define PRINT_DETAIL_MED   2
#define PRINT_DETAIL_HIGH  3

# 默认打印详细程度为低
#define DEFAULT_PRINT_DETAIL (PRINT_DETAIL_LOW)
# 默认打印描述符为标准输出
#define DEFAULT_PRINT_DESCRIPTOR stdout

# 定义数据包元素类
class PacketElement {

  protected:

    int length;
    PacketElement *next;    /**< Next PacketElement (next proto header)      */
    PacketElement *prev;    /**< Prev PacketElement (previous proto header)  */

  public:

    # 构造函数
    PacketElement();

    # 析构函数
    virtual ~PacketElement(){

    } /* End of PacketElement destructor */

    /** This function MUST be overwritten on ANY class that inherits from
      *  this one. Otherwise getBinaryBuffer will fail */
    # 获取数据包元素的缓冲区指针
    virtual u8 * getBufferPointer(){
        netutil_fatal("getBufferPointer(): Attempting to use superclass PacketElement method.\n");
        return NULL;
     } /* End of getBufferPointer() */

    /** Returns a buffer that contains the header of the packet + all the
     *  lower level headers and payload. Returned buffer should be ok to be
     *  passes to a send() call to be transferred trough a socket.
     *  @return a pointer to a free()able buffer that contains packet's binary
     *  data.
     *  @warning If there are linked elements, their getBinaryBuffer() method
     *  will be called recursively and the buffers that they return WILL be
     *  free()d as soon as we copy the data in our own allocated buffer.
     *  @warning Calls to this method may not ve very efficient since they
     *  always involved a few malloc()s and free()s. If you want efficiency
     *  use dumpToBinaryBuffer(); */
    # 返回一个指向二进制缓冲区的指针
    virtual u8 * getBinaryBuffer(){
      u8 *ourbuff=NULL;
      u8 *othersbuff=NULL;
      u8 *totalbuff=NULL;
      long otherslen=0;

      /* 获取自己的缓冲区地址 */
      if ( (ourbuff=getBufferPointer()) == NULL ){
          netutil_fatal("getBinaryBuffer(): Couldn't get own data pointer\n");
      }
      if( next != NULL ){ /* 存在其他数据包元素 */
        othersbuff = next->getBinaryBuffer();
        otherslen=next->getLen();
        totalbuff=(u8 *)safe_zalloc(otherslen + length);
        memcpy(totalbuff, ourbuff, length);
        memcpy(totalbuff+length, othersbuff, otherslen);
        free(othersbuff);
      }else{
           totalbuff=(u8 *)safe_zalloc(length);
           memcpy(totalbuff, ourbuff, length);
      }
      return totalbuff;
    } /* End of getBinaryBuffer() */


    # 将数据转储到二进制缓冲区
    virtual int dumpToBinaryBuffer(u8* dst, int maxlen){
      u8 *ourbuff=NULL;
      long ourlength=0;
      /* 获取自己的缓冲区地址和长度 */
      if ( (ourbuff=getBufferPointer()) == NULL ||  (ourlength=this->length) < 0 )
            netutil_fatal("getBinaryBuffer(): Couldn't get own data pointer\n");
      /* 复制我们的部分缓冲区 */
      if ( maxlen < ourlength )
            netutil_fatal("getBinaryBuffer(): Packet exceeds maximum length %d\n", maxlen);
      memcpy( dst, ourbuff, ourlength);
       /* 如果还有更多元素，告诉它们复制它们的部分 */
       if( next!= NULL ){
            next->dumpToBinaryBuffer(dst+ourlength, maxlen-ourlength);
       }
       return this->getLen();
    } /* End of dumpToBinaryBuffer() */


    /** 与前一个函数相同，但它将返回缓冲区的长度存储在提供的 int 指针指向的内存中。 */
    virtual u8 * getBinaryBuffer(int *len){
      u8 *buff = getBinaryBuffer();
      if( len != NULL )
         *len = getLen();
      return buff;
    } /* End of getBinaryBuffer() */
    /** 返回此 PacketElement 的长度 + 所有与其相邻的 PacketElements 的长度（通过 "next" 属性链接）。例如，如果我们有一个 IPv4Header p1，链接到一个 TCPHeader p2，表示一个没有选项的简单 TCP SYN，调用 p1.getLen() 将返回 20（没有选项的 IP 头）+ 20（没有选项的 TCP 头）= 40 字节。 */
    int getLen() const {
        /* 如果我们有其他链接的数据包元素，获取其长度 */
        if (next!=NULL)
            return length + next->getLen();
        else
            return length;
    } /* getLen() 结束 */


    /** 返回与此对象链接的下一个 PacketElement 的地址 */
    virtual PacketElement *getNextElement() const {
      return next;
    } /* getNextElement() 结束 */


    /** 将当前对象与协议链中的下一个标头链接起来。注意，此方法还将下一个元素与此元素链接起来，调用 setPrevElement()。 */
    virtual int setNextElement(PacketElement *n){
      next=n;
      if(next!=NULL)
          next->setPrevElement(this);
      return OP_SUCCESS;
    } /* setNextElement() 结束 */


    /** 使用提供的指针值设置属性 prev。
     *  @warning 提供的指针必须指向一个 PacketElement 对象或继承自它的对象。 */
    virtual int setPrevElement(PacketElement *n){
      this->prev=n;
      return OP_SUCCESS;
    } /* setPrevElement() 结束 */
    /** 返回链接到当前 PacketElement 的上一个 PacketElement 的地址。
     *  @warning 在许多情况下，这个函数会返回 NULL，因为很可能使用这个类的用户没有在两个方向上链接 PacketElements。
     *  通常情况下，会将 IPHeader 对象的 "next" 属性设置为其后跟随的 TCPHeader，但不会反过来设置。
     */
    virtual PacketElement *getPrevElement(){
      return prev;
    } /* End of getPrevElement() */

    /** 任何继承自 PacketElement 的类都应该重写这个方法。它应该打印对象的内容，然后调用 this->next->print()，如果 this->next!=NULL。
     */
    virtual int print(FILE *output, int detail) const {
        if(this->next!=NULL)
            this->next->print(output, detail);
        return OP_SUCCESS;
    } /* End of printf() */

    /** 打印对象的内容，使用默认的打印描述符和默认的打印细节。
     */
    virtual int print() const {
        return print(DEFAULT_PRINT_DESCRIPTOR, DEFAULT_PRINT_DETAIL);
    }

    /** 打印对象的内容，使用默认的打印描述符和给定的打印细节。
     */
    virtual int print(int detail) const {
        return print(DEFAULT_PRINT_DESCRIPTOR, detail);
    }

    /** 打印分隔符到输出流，使用给定的打印细节。
     */
    virtual void print_separator(FILE *output, int detail) const {
        fprintf(output, " ");
    }

    /* 返回对象表示的协议类型。所有子类都必须重写这个方法。
     */
    virtual int protocol_id() const = 0;
};
#endif



// 结束 C++ 的头文件宏定义
```