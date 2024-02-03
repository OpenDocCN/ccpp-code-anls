# `nmap\service_scan.h`

```cpp
/* $Id$ */

#ifndef SERVICE_SCAN_H
#define SERVICE_SCAN_H

#include "portlist.h"
#include "scan_lists.h"

#include <vector>

#define PCRE2_CODE_UNIT_WIDTH 8
#include <pcre2.h>

#undef NDEBUG
#include <assert.h>

/**********************  DEFINES/ENUMS ***********************************/
#define DEFAULT_SERVICEWAITMS 5000
#define DEFAULT_TCPWRAPPEDMS 2000   // connections closed after this timeout are not considered "tcpwrapped"
#define DEFAULT_CONNECT_TIMEOUT 5000
#define DEFAULT_CONNECT_SSL_TIMEOUT 8000  // includes connect() + ssl negotiation
#define MAXFALLBACKS 20 /* How many comma separated fallbacks are allowed in the service-probes file? */

/**********************  STRUCTURES  ***********************************/

// This is returned when we find a match
struct MatchDetails {
// Rather the match is a "soft" service only match, where we should
// continue to look for a better match.
  bool isSoft;

  // The service that was matched (Or NULL) zero-terminated.
  const char *serviceName;

  // The line number of this match in nmap-service-probes.
  int lineno;

  // The product/version/info for the service that was matched (Or NULL)
  // zero-terminated.
  const char *product;
  const char *version;
  const char *info;

  // More information from a match. Zero-terminated strings or NULL.
  const char *hostname;
  const char *ostype;
  const char *devicetype;

  // CPE identifiers for application, OS, and hardware type.
  const char *cpe_a;
  const char *cpe_o;
  const char *cpe_h;
};

/**********************  CLASSES     ***********************************/

class ServiceProbeMatch {
 public:
  ServiceProbeMatch();
  ~ServiceProbeMatch();

// match text from the nmap-service-probes file.  This must be called
// before you try and do anything with this match.  This function
// should be passed the whole line starting with "match" or
// "softmatch" in nmap-service-probes.  The line number that the text
// 初始化匹配函数，用于报告错误消息，如果存在语法问题，则会中止程序
void InitMatch(const char *matchtext, int lineno);

// 如果buf（长度为buflen）与ServiceProbeMatch中的正则表达式匹配，则返回匹配的详细信息（服务名称，版本号（如果适用），以及是否为“软”匹配）。
// 如果buf不匹配，则结构中的serviceName字段将为NULL。返回的MatchDetails仅在下一次调用此函数之前有效。
// 唯一的例外是serviceName字段可以在整个程序执行期间保存。如果没有匹配版本，则该字段将为NULL。
const struct MatchDetails *testMatch(const u8 *buf, int buflen);

// 返回匹配的服务名称
const char *getName() const { return servicename; }

// 定义此匹配字符串的行号。如果未知，则返回-1。
int getLineNo() const { return deflineno; }

private:
int deflineno; // 定义此匹配的行号
bool isInitialized; // InitMatch是否已被调用？
const char *servicename;
char *matchstr; // 正则表达式文本
pcre2_code *regex_compiled;
pcre2_match_data *match_data;
pcre2_match_context *match_context;
bool matchops_ignorecase;
bool matchops_dotall;
bool isSoft; // 这是一个软匹配吗？（在nmap-service-probes中的“softmatch”关键字）
// 如果这3个中的任何一个不为NULL，则通过子字符串匹配给出了产品、版本或模板字符串，以推断应用程序/版本信息。
char *product_template;
char *version_template;
char *info_template;
// 更多模板：
char *hostname_template;
char *ostype_template;
char *devicetype_template;
std::vector<char *> cpe_templates;
// 为 testMatch() 调用填写并返回的匹配细节
struct MatchDetails MD_return;

// 使用六个版本模板和此处包含的匹配数据，将版本信息放入给定的字符串中（只要大小足够）。
// 返回零表示成功。如果没有字符串的模板可用，那么在函数调用后该字符串将具有零长度
int getVersionStr(const u8 *subject, size_t subjectlen,
                  char *product, size_t productlen,
                  char *version, size_t versionlen, char *info, size_t infolen,
                  char *hostname, size_t hostnamelen, char *ostype, size_t ostypelen,
                  char *devicetype, size_t devicetypelen,
                  char *cpe_a, size_t cpe_alen,
                  char *cpe_h, size_t cpe_hlen,
                  char *cpe_o, size_t cpe_olen) const;
};

class ServiceProbe {
public:
  // 构造函数
  ServiceProbe();
  // 析构函数
  ~ServiceProbe();
  // 返回探测器名称
  const char *getName() const { return probename; }
  // 如果这是“null”探测器，则返回 true，表示它不发送探测，只监听横幅。只有 TCP 服务有这个特性
  bool isNullProbe() const { return (probestringlen == 0); }
// 等待连接成功（或发送数据包）后等待响应的总时间
  int totalwaitms;
  // 如果连接成功但在此时间内关闭，则为 tcpwrapped
  int tcpwrappedms;

  // 解析 nmap-service-probes 文件中的 "probe " 行。传递 "probe " 后的其余部分
  // 格式应该是：
  // [TCP|UDP] [probename] "probetext"
  // 请求 lineno 是因为如果无法解析字符串，此函数将报错（给出行号）
  void setProbeDetails(char *pd, int lineno);

  // 获取原始二进制形式的探测字符串和长度。字符串将以 NUL 结尾，但字符串中可能还有其他 \0，因此终止仅用于便于在调试情况下打印 ASCII 探测。
  const u8 *getProbeString(int *stringlen) const { *stringlen = probestringlen; return probestring; }
  void setProbeString(const u8 *ps, int stringlen);

  /* 协议是 IPPROTO_TCP 和 IPPROTO_UDP */
  u8 getProbeProtocol() const {
    assert(probeprotocol == IPPROTO_TCP || probeprotocol == IPPROTO_UDP);
};

class AllProbes {
// 定义一个公共类 AllProbes
public:
  AllProbes();
  ~AllProbes();
  // 尝试在 AllProbes 类中查找具有给定名称和协议的探针。如果找不到请求协议的匹配项，它将尝试在任何协议上查找匹配项。它可以返回空探针。
  ServiceProbe *getProbeByName(const char *name, int proto) const;
  // 所有探针的向量，除了 nullProbe
  std::vector<ServiceProbe *> probes;
  // 无探针文本 - 只是等待横幅
  ServiceProbe *nullProbe;

  // 在调用此函数之前，回退存在于每个探针的回退字符串字段中，作为未解析的逗号分隔字符串。此函数使用有序列表的指针填充每个探针的回退数组，以便提高效率并处理诸如 NULL 探针和后续探针的奇怪情况。此函数还释放所有的回退字符串。
  void compileFallbacks();

  // 检查指定端口和协议是否被排除
  int isExcluded(unsigned short port, int proto) const;
  bool excluded_seen;
  struct scan_lists excludedports;

  // 初始化服务扫描
  static AllProbes *service_scan_init(void);
  // 释放服务扫描
  static void service_scan_free(void);
  // 检查排除端口
  static int check_excluded_port(unsigned short port, int proto);
protected:
  static AllProbes *global_AP;
};

/**********************  PROTOTYPES  ***********************************/

// 将给定的 nmap-service-probes 文件解析为 AP 类。不能将其设置为静态，因为我有外部维护工具（servicematch）使用它
void parse_nmap_service_probe_file(AllProbes *AP, const char *filename);

// 对指定目标的所有开放端口执行服务指纹扫描
int service_scan(std::vector<Target *> &Targets);

#endif /* SERVICE_SCAN_H */
```