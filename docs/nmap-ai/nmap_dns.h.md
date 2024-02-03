# `nmap\nmap_dns.h`

```cpp
#ifndef NMAP_DNS_H
#define NMAP_DNS_H

class Target;

#include <nbase.h>

#include <string>
#include <list>

#include <algorithm>
#include <sstream>

#define DNS_LABEL_MAX_LENGTH 63  // 定义 DNS 标签的最大长度
#define DNS_NAME_MAX_LENGTH 255   // 定义 DNS 名称的最大长度

namespace DNS
{

#define DNS_CHECK_ACCUMLATE(accumulator, tmp, exp) \  // 定义一个宏，用于检查累积值
  do { tmp = exp; if(tmp < 1) return 0 ; accumulator += tmp;} while(0)

#define DNS_CHECK_UPPER_BOUND(accumulator, max)\  // 定义一个宏，用于检查累积值是否超过上限
  do { if(accumulator > max) return 0; } while(0)

#define DNS_HAS_FLAG(v,flag) ((v&flag)==flag)  // 定义一个宏，用于检查是否存在特定标志位

#define DNS_HAS_ERR(v, err) ((v&DNS::ERR_ALL)==err)  // 定义一个宏，用于检查是否存在特定错误标志位

typedef enum
{
  ID = 0,  // 偏移量为0的字段为标识符
  FLAGS_OFFSET = 2,  // 偏移量为2的字段为标志
  QDCOUNT = 4,  // 偏移量为4的字段为问题计数
  ANCOUNT = 6,  // 偏移量为6的字段为回答计数
  NSCOUNT = 8,  // 偏移量为8的字段为授权机构计数
  ARCOUNT = 10,  // 偏移量为10的字段为附加记录计数
  DATA = 12  // 偏移量为12的字段为数据
} HEADER_OFFSET;

typedef enum {
  ERR_ALL = 0x0007,  // 所有错误标志位的组合
  CHECKING_DISABLED = 0x0010,  // 检查禁用
  AUTHENTICATED_DATA = 0x0020,  // 认证数据
  ZERO = 0x0070,  // 零
  RECURSION_AVAILABLE = 0x0080,  // 递归可用
  RECURSION_DESIRED = 0x0100,  // 期望递归
  TRUNCATED = 0x0200,  // 截断
  AUTHORITATIVE_ANSWER = 0x0400,  // 授权回答
  OP_STANDARD_QUERY = 0x0000,  // 标准查询操作
  OP_INVERSE_QUERY = 0x0800, // 反向查询操作，已在 RFC 3425 中废弃
  OP_SERVER_STATUS = 0x1000,  // 服务器状态操作
  RESPONSE = 0x8000  // 响应
} FLAGS;

typedef enum {
  ERR_NO = 0x0000,  // 无错误
  ERR_FORMAT = 0x0001,  // 格式错误
  ERR_SERVFAIL = 0x0002,  // 服务器失败
  ERR_NAME = 0x0003,  // 名称错误
  ERR_NOT_IMPLEMENTED = 0x0004,  // 未实现
  ERR_REFUSED = 0x0005,  // 拒绝
} ERRORS;

typedef enum {
  A = 1,  // A 记录类型
  CNAME = 5,  // CNAME 记录类型
  PTR = 12,  // PTR 记录类型
  AAAA = 28,  // AAAA 记录类型
} RECORD_TYPE;

typedef enum {
  CLASS_IN = 1  // IN 类型
} RECORD_CLASS;

const u8 COMPRESSED_NAME = 0xc0;  // 压缩名称的标志位

#define C_IPV4_PTR_DOMAIN ".in-addr.arpa"  // IPv4 PTR 域名
#define C_IPV6_PTR_DOMAIN ".ip6.arpa"  // IPv6 PTR 域名
const std::string IPV4_PTR_DOMAIN = C_IPV4_PTR_DOMAIN;  // 存储 IPv4 PTR 域名的字符串
const std::string IPV6_PTR_DOMAIN = C_IPV6_PTR_DOMAIN;  // 存储 IPv6 PTR 域名的字符串

class Factory
{
// 定义一个静态成员变量，用于记录递增的 ID
static u16 progressiveId;
// 将 IP 转换为 PTR 记录
static bool ipToPtr(const sockaddr_storage &ip, std::string &ptr);
// 将 PTR 记录转换为 IP
static bool ptrToIp(const std::string &ptr, sockaddr_storage &ip);
// 构建简单的 DNS 请求
static size_t buildSimpleRequest(const std::string &name, RECORD_TYPE rt, u8 *buf, size_t maxlen);
// 构建反向 DNS 请求
static size_t buildReverseRequest(const sockaddr_storage &ip, u8 *buf, size_t maxlen);
// 将无符号短整型数值放入缓冲区
static size_t putUnsignedShort(u16 num, u8 *buf, size_t offset, size_t maxlen);
// 将域名放入缓冲区
static size_t putDomainName(const std::string &name, u8 *buf, size_t offset, size_t maxlen);
// 从缓冲区解析出无符号短整型数值
static size_t parseUnsignedShort(u16 &num, const u8 *buf, size_t offset, size_t maxlen);
// 从缓冲区解析出无符号整型数值
static size_t parseUnsignedInt(u32 &num, const u8 *buf, size_t offset, size_t maxlen);
// 从缓冲区解析出域名
static size_t parseDomainName(std::string &name, const u8 *buf, size_t offset, size_t maxlen);



// 定义一个抽象基类 Record
class Record
{
public:
  // 纯虚函数，用于克隆 Record 对象
  virtual Record * clone() = 0;
  // 虚析构函数
  virtual ~Record() {}
  // 纯虚函数，从缓冲区解析出记录
  virtual size_t parseFromBuffer(const u8 *buf, size_t offset, size_t maxlen) = 0;
};

// 派生类 A_Record，表示 A 记录
class A_Record : public Record
{
public:
  // 存储 IP 地址的结构
  sockaddr_storage value;
  // 克隆 A_Record 对象
  Record * clone() { return new A_Record(*this); }
  // 虚析构函数
  ~A_Record() {}
  // 从缓冲区解析出 A 记录
  size_t parseFromBuffer(const u8 *buf, size_t offset, size_t maxlen);
};

// 派生类 PTR_Record，表示 PTR 记录
class PTR_Record : public Record
{
public:
  // 存储 PTR 记录的值
  std::string value;
  // 克隆 PTR_Record 对象
  Record * clone() { return new PTR_Record(*this); }
  // 虚析构函数
  ~PTR_Record() {}
  // 从缓冲区解析出 PTR 记录
  size_t parseFromBuffer(const u8 *buf, size_t offset, size_t maxlen)
  {
    return Factory::parseDomainName(value, buf, offset, maxlen);
  }
};

// 派生类 CNAME_Record，表示 CNAME 记录
class CNAME_Record : public Record
{
public:
  // 存储 CNAME 记录的值
  std::string value;
  // 克隆 CNAME_Record 对象
  Record * clone() { return new CNAME_Record(*this); }
  // 虚析构函数
  ~CNAME_Record() {}
  // 从缓冲区解析出 CNAME 记录
  size_t parseFromBuffer(const u8 *buf, size_t offset, size_t maxlen)
  {
    return Factory::parseDomainName(value, buf, offset, maxlen);
  }
};

// 定义一个 Query 类，表示 DNS 查询
class Query
{
public:
  // 查询的域名
  std::string name;
  // 记录类型
  u16 record_type;
  // 记录类别
  u16 record_class;

  // 从缓冲区解析出查询
  size_t parseFromBuffer(const u8 *buf, size_t offset, size_t maxlen);
};

// 定义一个 Answer 类，表示 DNS 回答
class Answer
{
// 定义一个公共类 Answer
public:
  // 默认构造函数，初始化 record 为 NULL
  Answer() : record(NULL) {}
  // 拷贝构造函数，深拷贝对象 c 的属性
  Answer(const Answer &c) : name(c.name), record_type(c.record_type),
    record_class(c.record_class), ttl(c.ttl), length(c.length),
    record(c.record->clone()) {}
  // 析构函数，释放 record 指针指向的内存
  ~Answer() { delete record; }

  // 对象属性
  std::string name;
  u16 record_type;
  u16 record_class;
  u32 ttl;
  u16 length;
  Record * record;

  // 从缓冲区解析数据，返回消耗的字节数
  size_t parseFromBuffer(const u8 *buf, size_t offset, size_t maxlen);
  // 赋值运算符重载
  Answer& operator=(const Answer &r);
};

// 定义一个公共类 Packet
class Packet
{
public:
  // 默认构造函数，初始化 id 和 flags 为 0
  Packet() : id(0), flags(0) {}
  // 析构函数
  ~Packet() {}

  // 添加标志位
  void addFlags(FLAGS fl){ flags |= fl; }
  // 移除标志位
  void removeFlags(FLAGS fl){ flags &= ~fl; }
  // 重置标志位
  void resetFlags() { flags = 0; }
  // 从缓冲区解析数据，返回消耗的字节数
  size_t parseFromBuffer(const u8 *buf, size_t maxlen);

  // 对象属性
  u16 id;
  u16 flags;
  std::list<Query> queries;
  std::list<Answer> answers;
};

// 声明一个函数 nmap_mass_rdns，接受 Target 类型的指针数组和整数参数
void nmap_mass_rdns(Target ** targets, int num_targets);

// 声明一个函数 get_dns_servers，返回一个字符串列表
std::list<std::string> get_dns_servers();

// 结束头文件的声明
#endif
```