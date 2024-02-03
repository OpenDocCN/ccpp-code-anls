# `nmap\nping\NpingTarget.h`

```cpp
// 防止头文件重复包含
#ifndef NPINGTARGET_H
#define NPINGTARGET_H

// 包含必要的头文件
#include "nping.h"
#include "common.h"
#include "../libnetutil/netutil.h"

// 定义 INET6_ADDRSTRLEN 如果未定义
#ifndef INET6_ADDRSTRLEN
#define INET6_ADDRSTRLEN 46
#endif

// 定义 NpingTarget 类
class NpingTarget {

  private:

    char devname[32];       /**< Net interface normal name                   */
    char devfullname[32];   /**< Net interface full name                     */
    devtype dev_type;       /**< Type of network interface                   */
    int directly_connected; /**< -1 = unset; 0 = no; 1 = yes                 */  
    int distance;           /**< Distance to target in hops                  */
    int addressfamily;      /**< Address family:  AF_INET or AF_INET6        */
    char *nameIPBuf;        /**< for the NameIP(void) function to return     */
    char *hostname;         /**< Resolved host name                          */
    int namedhost;          /**< =1 is named host; =0 is an IP; =-1 unset    */
    char *targetname;       /**< Name of the target host given on the        *
                             *   command line if it is a named host          */

    // 目标地址和长度
    struct sockaddr_storage targetsock;   
    size_t targetsocklen;

    // 源地址和长度
    struct sockaddr_storage sourcesock;   
    size_t sourcesocklen;

    // 伪造的源地址和长度
    struct sockaddr_storage spoofedsrcsock; 
    size_t spoofedsrcsocklen;
    bool spoofedsrc_set;

    // 下一跳地址和长度
    struct sockaddr_storage nexthopsock;  
    size_t nexthopsocklen;

    // 目标 IP 地址字符串和设置标志
    char targetipstring[INET6_ADDRSTRLEN];
    bool targetipstring_set;

    // 目标 MAC 地址和设置标志
    u8 MACaddress[6];         
    bool MACaddress_set;

    // 源 MAC 地址和设置标志
    u8 SrcMACaddress[6];      
    bool SrcMACaddress_set;

    // 下一跳 MAC 地址和设置标志
    u8 NextHopMACaddress[6];  
    bool NextHopMACaddress_set;
    /* ICMP 标识字段，应该只初始化一次，因此我们将所有 ICMP 探测发送到相同的目标，使用相同的 ID（就像 ping 实用程序一样） */
    u16 icmp_id;

    /* ICMP 序列字段，应该初始化为 1，然后每次通过 obtainICMPSequence() 请求其值时递增 */
    u16 icmp_seq;

    /* 私有方法 */
    void Initialize();
    void FreeInternal();
    void generateIPString();

  public:

    NpingTarget();
    ~NpingTarget();
    void Recycle();

    /* 目标 IP 地址 */
    int getTargetSockAddr(struct sockaddr_storage *ss, size_t *ss_len);
    int setTargetSockAddr(struct sockaddr_storage *ss, size_t ss_len);
    struct in_addr getIPv4Address();
    const struct in_addr *getIPv4Address_aux();
    struct in6_addr getIPv6Address();
    const struct in6_addr *getIPv6Address_aux();
    u8 *getIPv6Address_u8();

    /* 用于到达目标的源地址 */
    int getSourceSockAddr(struct sockaddr_storage *ss, size_t *ss_len);
    int setSourceSockAddr(struct sockaddr_storage *ss, size_t ss_len);
    int getSpoofedSourceSockAddr(struct sockaddr_storage *ss, size_t *ss_len);
    int setSpoofedSourceSockAddr(struct sockaddr_storage *ss, size_t ss_len);
    bool spoofingSourceAddress();
    struct in_addr getIPv4SourceAddress();  
    const struct in_addr *getIPv4SourceAddress_aux();
    struct in_addr getIPv4SpoofedSourceAddress();  
    const struct in_addr *getIPv4SpoofedSourceAddress_aux();
    struct in6_addr getIPv6SourceAddress();  
    const struct in6_addr *getIPv6SourceAddress_aux();
    u8 *getIPv6SourceAddress_u8();

    /* 关于主机接近性的信息 */
    void setDirectlyConnected(bool connected);
    bool isDirectlyConnected();
    int isDirectlyConnectedOrUnset();

    /* 下一跳 */
    void setNextHop(struct sockaddr_storage *next_hop, size_t next_hop_len);  
    # 获取下一跳地址和长度
    bool getNextHop(struct sockaddr_storage *next_hop, size_t *next_hop_len);
    # 设置下一跳 MAC 地址
    int setNextHopMACAddress(const u8 *addy);
    # 获取下一跳 MAC 地址
    const u8 *getNextHopMACAddress();

    # 设置 MAC 地址（在目标直接连接时使用）
    int setMACAddress(const u8 *addy);
    # 获取 MAC 地址
    const u8 *getMACAddress();
    # 确定下一跳 MAC 地址
    bool determineNextHopMACAddress();
    # 确定目标 MAC 地址
    bool determineTargetMACAddress();

    # 设置源 MAC 地址
    int setSrcMACAddress(const u8 *addy);
    # 获取源 MAC 地址
    const u8 *getSrcMACAddress();

    # 设置用于此目标的网络设备名称
    void setDeviceNames(const char *name, const char *fullname);
    # 获取设备名称
    const char *getDeviceName();
    # 获取设备完整名称
    const char *getDeviceFullName();
    # 设置设备类型
    int setDeviceType(devtype type);
    # 获取设备类型
    devtype getDeviceType();

    # 获取解析后的主机名
    const char *getResolvedHostName();
    # 设置解析后的主机名
    void setResolvedHostName(char *name);

    # 获取从命令行提供的目标名称
    const char *getSuppliedHostName();
    # 设置从命令行提供的目标名称
    int setSuppliedHostName(char *name);
    # 设置命名主机
    int setNamedHost(bool val);
    # 是否为命名主机
    bool isNamedHost();

    # 可打印的字符串
    const char *getTargetIPstr();
    const char *getNameAndIP(char *buf, size_t buflen);
    const char *getNameAndIP();
    const char *getSourceIPStr();
    const char *getSpoofedSourceIPStr();
    const char *getNextHopIPStr();
    const char *getMACStr(u8 *mac);
    const char *getTargetMACStr();
    const char *getSourceMACStr();
    const char *getNextHopMACStr(); 

    # ICMP 相关方法
    u16 obtainICMPSequence();
    u16 getICMPIdentifier();

    # 其他
    void printTargetDetails();
/* STATS***********************************************************************/
// 定义最大发送探测信息条目数
#define MAX_SENTPROBEINFO_ENTRIES 10

// 定义数据包统计结构体
typedef struct pkt_stat{
    int proto; // 协议类型
    u16 tcp_port; // TCP端口号
    u16 icmp_id; // ICMP标识符
    u16 icmp_seq; // ICMP序列号
    struct timeval sent; // 发送时间
    struct timeval recv; // 接收时间
}pktstat_t;

// 存储发送探测信息的数组
pktstat_t sentprobes[MAX_SENTPROBEINFO_ENTRIES];
int current_stat; // 当前统计信息
int total_stats; // 总统计信息

unsigned long int sent_total; // 总发送数
unsigned long int recv_total; // 总接收数
unsigned long int max_rtt; // 最大往返时间
bool max_rtt_set; // 最大往返时间是否设置
unsigned long int min_rtt; // 最小往返时间
bool min_rtt_set; // 最小往返时间是否设置
unsigned long int avg_rtt; // 平均往返时间
bool avg_rtt_set; // 平均往返时间是否设置

// 设置TCP接收探测
int setProbeRecvTCP(u16 sport, u16 dport);
// 设置TCP发送探测
int setProbeSentTCP(u16 sport, u16 dport);
// 设置UDP接收探测
int setProbeRecvUDP(u16 sport, u16 dport);
// 设置UDP发送探测
int setProbeSentUDP(u16 sport, u16 dport);
// 设置ICMP发送探测
int setProbeSentICMP(u16 id, u16 seq);
// 设置ICMP接收探测
int setProbeRecvICMP(u16 id, u16 seq);
// 设置ARP发送探测
int setProbeSentARP();
// 设置ARP接收探测
int setProbeRecvARP();
// 更新往返时间
int updateRTTs(unsigned long int diff);
// 打印统计信息
int printStats();
// 打印计数
void printCounts();
// 打印往返时间
void printRTTs();
/* STATS***********************************************************************/

};

#endif /* NPINGTARGET_H */
```