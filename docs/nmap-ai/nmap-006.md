# Nmap源码解析 6

# `services.h`

This is a text-based network tool, Nmap, that allows users to discover the service 


```cpp

/***************************************************************************
 * services.h -- Various functions relating to reading the nmap-services   *
 * file and port <-> service mapping                                       *
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

这段代码定义了一个名为`services_h`的函数。从函数声明来看，它接受了两个参数：`const char *mask`和`u8 *porttbl`，以及一个未定义的参数`int range_type`。

该函数的作用是将从给定掩码中选择的所有端口映射到`porttbl`数组中的端口上。掩码可以使用`const char *`类型存储，例如`const char *mask = "127.0.0.1/8";`。

具体来说，函数内部首先定义了一个名为`nservent`的结构体，它包含一个`const char *`类型的`s_name`、一个`const char *`类型的`s_proto`和一个`u16`类型的`s_port`成员。

然后，函数定义了一个名为`addportsfromservmask`的函数，它接受两个参数：掩码和`porttbl`数组。函数内部使用`u16`类型存储`porttbl`数组中的端口号，然后使用位运算将掩码与`porttbl`数组中的端口号进行按位与运算，得到一个包含所有与掩码相同的端口的数组。最后，函数将这个数组赋值给`nservent`结构体中的`s_port`成员，并将结果返回。

由于该函数没有定义返回类型，因此它实际上是一个`void`类型的函数，也就是说它不会输出任何值。


```cpp
/* $Id$ */


#ifndef SERVICES_H
#define SERVICES_H

#include "nbase.h"

struct nservent {
  const char *s_name;
  const char *s_proto;
  u16 s_port;
};

int addportsfromservmask(const char *mask, u8 *porttbl, int range_type);
```

这是一个 C 语言代码，定义了两个函数，以及一个名为 free_services 的函数指针。

1. struct nservent *nmap_getservbyport(u16 port, u16 proto) 的作用是获取绑定到指定端口的服务器结构体指针，返回值为一个指向 struct nservent 类型的指针。

2. gettoppts(double level, const char *portlist, struct scan_lists *ports, const char *exclude_list = NULL) 的作用是读取从给定端口列表中的指定端口开始的高层次的 PORTS 链，并将该链存储在 ports 结构体中。如果给定的 exclude_list 存在，则排除给定列表中的端口。最后，将 ports 结构体中的端口复制到给定的 scan_lists 链中。

3. free_services() 是一个未定义的函数，没有具体的实现。


```cpp
const struct nservent *nmap_getservbyport(u16 port, u16 proto);
void gettoppts(double level, const char *portlist, struct scan_lists * ports, const char *exclude_list = NULL);

void free_services();

#endif


```

# `service_scan.h`

This is a text that provides information about the Nmap software and its licensing. Nmap is a network scanner that is designed to be a simple, ASCII-based tool for discovering what hosts on a network are黑洞， open ports, and running services. Nmap is released under the Nmap Public Source License, which allows users to use, modify, and redistribute Nmap, but does not allow for commercial use or distribution. The Nmap license also prohibits companies from using Nmap in commercial products, except for the Nmap OEM Edition, which has more permissive terms. The text also explains that if you have received an Nmap license agreement or contract with other terms, you may choose to use and redistribute Nmap under those terms instead. The official Nmap Windows builds include the Npcap software for packet capture and transmission, which is under a separate license term that prohibits redistribution without special permission. The text also notes that the source code for Nmap is available for audit and that users are encouraged to submit changes as a Github PR or by email to the dev@nmap.org mailing list.



```cpp

/***************************************************************************
 * service_scan.h -- Routines used for service fingerprinting to determine *
 * what application-level protocol is listening on a given port            *
 * (e.g. snmp, http, ftp, smtp, etc.)                                      *
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

这段代码定义了一个名为 "service_scan_h" 的头文件，其作用是定义了一个名为 "service_scan" 的函数。

具体来说，这个头文件包含了一个头文件 "portlist.h" 和 "scan_lists.h"，这两个头文件可能定义了一些通用的函数和数据结构，而这个头文件中定义的函数主要是与字符串匹配有关的内容。

此外，头文件中还包含了一个名为 "pcre2_code_unit_width" 的定义，它表示一个pcre2库中代码单元的最大宽度，这个定义可能是在其他头文件或者库中使用的，用于定义代码单元宽度的概念。

最后，头文件中定义了一个名为 "service_scan" 的函数，它接受一个字符串参数，对这个字符串进行匹配，如果匹配成功，则返回匹配的匹配项数，否则返回0。函数的具体实现可能是在其他源文件或者数据结构中定义的，这个头文件只是定义了函数的接口，其他实现可能由其他源文件或者库来完成。


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

```



这是一段C代码，定义了一些常量，包括DEFAULT_SERVICEWAITMS、DEFAULT_TCPWRAPPEDMS、DEFAULT_CONNECT_TIMEOUT和MAXFALLBACKS。这些常量定义了在Nmap服务探针文件中允许查找的服务等待时间、最大兼容性尝试次数、连接超时时间和允许的最大兼容性尝试次数。

MatchDetails结构体定义了一个匹配细节，包括服务名称、匹配行号、产品、版本、信息以及更多来自匹配的信息。这个结构体是用于表示在Nmap服务探针文件中发现的匹配细节。


```cpp
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

```

这段代码定义了一个名为ServiceProbeMatch的类，其作用是读取一个Nmap服务探针文件中的匹配文本，并进行比较，如果匹配成功则返回相应的服务名称、服务版本号和是否为软匹配等信息，否则返回MatchDetails结构中的const结构体。

具体来说，代码中包括以下几个函数：

1. InitMatch函数：接收一个匹配文本和匹配文本中的最后一个字符'@'，并初始化一个MatchDetails结构体中的成员变量。这个函数主要用于处理文件头部的信息，包括指定软件名称和版本号等。

2. testMatch函数：接收一个Nmap服务探针文件中的匹配文本和匹配文本中的最后一个字符'@'，并返回一个MatchDetails结构体中的const结构体。这个函数主要用于比较两个匹配文本是否相同或者相似。

3.~ServiceProbeMatch函数：实现了一个虚函数，这个函数在对象被删除时被调用，可用于清理资源。

4. InitMatch函数的具体实现没有给出，因此无法判断其具体的作用。


```cpp
/**********************  CLASSES     ***********************************/

class ServiceProbeMatch {
 public:
  ServiceProbeMatch();
  ~ServiceProbeMatch();

// match text from the nmap-service-probes file.  This must be called
// before you try and do anything with this match.  This function
// should be passed the whole line starting with "match" or
// "softmatch" in nmap-service-probes.  The line number that the text
// is provided so that it can be reported in error messages.  This
// function will abort the program if there is a syntax problem.
  void InitMatch(const char *matchtext, int lineno);

  // If the buf (of length buflen) match the regex in this
  // ServiceProbeMatch, returns the details of the match (service
  // name, version number if applicable, and whether this is a "soft"
  // match.  If the buf doesn't match, the serviceName field in the
  // structure will be NULL.  The MatchDetails returned is only valid
  // until the next time this function is called.  The only exception
  // is that the serviceName field can be saved throughout program
  // execution.  If no version matched, that field will be NULL.
  const struct MatchDetails *testMatch(const u8 *buf, int buflen);
```

这段代码是一个C语言编写的函数，它通过使用正则表达式（Regular Expression，简称regex）在给定的匹配字符串中查找并返回匹配的服务名称（servicename）。通过使用静态成员变量deflineno，可以确保即使该函数多次被调用，它的行为也不会发生改变。

该函数包含以下几个部分：

1. 获取匹配字符串的服务名称：
```cppc
const char *getName() const { return servicename; }
```
2.获取定义该匹配字符串的行号：
```cppc
int getLineNo() const { return deflineno; }
```
3. 初始化函数，以便在使用函数时可以初始化并检查它们：
```cppc
private:
 int deflineno; // The line number where this match is defined.
 bool isInitialized; // Has InitMatch yet been called?
 const char *servicename;
 char *matchstr; // Regular expression text
 pcre2_code *regex_compiled;
 pcre2_match_data *match_data;
 pcre2_match_context *match_context;
 bool matchops_ignorecase;
 bool matchops_dotall;
 bool isSoft; // is this a soft match? ("softmatch" keyword in nmap-service-probes)
 // If any of these 3 are non-NULL, a product, version, or template
 // string was given to deduce the application/version info via
 // substring matches.
 char *product_template;
 char *version_template;
 char *info_template;
 // More templates:
 char *hostname_template;
 char *ostype_template;
 char *devicetype_template;
 std::vector<char *> cpe_templates;
```
4. 通过substring匹配，在匹配字符串中查找匹配的服务名称：
```cppc
int search_service(const char *matchstr, const char *servicename)
{
   int pos = 0;
   int i = 0;
   while (i < strlen(matchstr) && pos < servicename.size())
   {
       int j = i - 1;
       if (servicename[i] != '\0')
           j++;
       if (j == servicename.size() - 1 && matchstr[i+j] == '\0')
           return i+j;
       else if (servicename[i+j+1] == '=')
           return i+j+1;
       else
           i++;
   }
   return -1;
}
```
5. 如果匹配字符串中存在匹配的服务名称，则返回匹配的行号：
```cppc
int getLineNo() const { return deflineno; }
```
6. 通过静态成员变量deflineno确保匹配字符串在该函数被调用之前已经被初始化：
```cppc
private:
 int deflineno; // The line number where this match is defined.
 bool isInitialized; // Has InitMatch yet been called?
```
7. 如果要查找与该匹配字符串匹配的服务名称，可以调用函数：
```cppc
int main()
{
   const char *servicename = "MyService";
   int deflineno = 5;
   int result = search_service(servicename.size() - 1, servicename);
   if (result == -1)
       std::cout << "Service name: " << servicename << std::endl;
   else
       std::cout << "Line number: " << deflineno << std::endl;
   return 0;
}
```
8. 最后，函数使用一些辅助函数：
```cppc
// Additional templates for more options like "hostname_template", "
//   "version_template", "info_template" etc.
// 
// https://github.com/舌锋叉夫/re2/blob
// It's a到底Re2搜索正则表达式的全面解析，涵盖，工作正常。
// https://github.com/j我要大豆/re2
// 再比如，想通过re2解析。


```
// Returns the service name this matches
  const char *getName() const { return servicename; }
  // The Line number where this match string was defined.  Returns
  // -1 if unknown.
  int getLineNo() const { return deflineno; }
 private:
  int deflineno; // The line number where this match is defined.
  bool isInitialized; // Has InitMatch yet been called?
  const char *servicename;
  char *matchstr; // Regular expression text
  pcre2_code *regex_compiled;
  pcre2_match_data *match_data;
  pcre2_match_context *match_context;
  bool matchops_ignorecase;
  bool matchops_dotall;
  bool isSoft; // is this a soft match? ("softmatch" keyword in nmap-service-probes)
  // If any of these 3 are non-NULL, a product, version, or template
  // string was given to deduce the application/version info via
  // substring matches.
  char *product_template;
  char *version_template;
  char *info_template;
  // More templates:
  char *hostname_template;
  char *ostype_template;
  char *devicetype_template;
  std::vector<char *> cpe_templates;
```cpp

这段代码定义了一个名为MatchDetails的结构体，其中包含了一些用于测试匹配的详细信息。MatchDetails结构体中包含六个版本相关的模板，以及用于获取产品版本号的字符串。

该函数使用了六个模板来计算产品版本号，如果产品名称长度小于给出的模板长度，则在相应的模板中查找空模板并返回0。如果产品名称长度等于或大于给出的模板长度，则函数将根据模板计算产品版本号，并将结果存储在MatchDetails结构体中，然后返回该结构体。

该函数可以被用来在测试中测试MatchDetails结构体中的产品版本号是否正确。例如，可以在主函数中调用getProductVersion函数，然后使用所得的匹配数据进行比较测试。


```
// Details to fill out and return for testMatch() calls
  struct MatchDetails MD_return;

  // Use the six version templates and the match data included here
  // to put the version info into the given strings, (as long as the sizes
  // are sufficient).  Returns zero for success.  If no template is available
  // for a string, that string will have zero length after the function
  // call (assuming the corresponding length passed in is at least 1)
  int getVersionStr(const u8 *subject, size_t subjectlen,
                  char *product, size_t productlen,
                  char *version, size_t versionlen, char *info, size_t infolen,
                  char *hostname, size_t hostnamelen, char *ostype, size_t ostypelen,
                  char *devicetype, size_t devicetypelen,
                  char *cpe_a, size_t cpe_alen,
                  char *cpe_h, size_t cpe_hlen,
                  char *cpe_o, size_t cpe_olen) const;
};


```cpp

This is a C function that appears to be part of the Nmap "ServiceProbe" implementation. It is used to add a match entry to a ServiceProbe buffer. The match is added with the specified `match` string and the corresponding line number in the `lineno` provided. The function has the following signature:
```
void addMatch(const char *match, int lineno);
```cpp
It takes a single parameter, `match`, which is the string representing the match, and `lineno`, which is the line number in the `linenumber` provided.

The function has the following additional parameters:
```
int lineno;
```cpp
This line number is requested when the function fails to parse the `match` string and will be passed as the `lineno` parameter.
```
const u8 *probestring;
```cpp
This parameter is the ProbeBestRing of the ServiceProbe. It is used to return the details of the nth match.
```
int probestringlen;
```cpp
This parameter is the length of the ProbeBestRing. It is used to return the details of the nth match.
```
std::vector<u16> probableports;
```cpp
This parameter is a vector of u16 ports that match the ServiceProbe.
```
std::vector<u16> probablesslports;
```cpp
This parameter is a vector of u16 ports that match the ServiceProbe with the lowest number of matches.
```
int rarity;
```cpp
This parameter is the rarity of the ServiceProbe.
```
std::vector<const char *> detectedServices;
```cpp
This parameter is a vector of ServiceProbe detected services.
```
int probeprotocol;
```cpp
This parameter is the protocol of the ServiceProbe.
```
std::vector<ServiceProbeMatch *> matches;
```cpp
This parameter is a vector of ServiceProbe Match objects. It may return `NULL` if there are no match lines at all in this probe.
```
void setPortVector(std::vector<u16> *portv, const char *portstr, int lineno);
```cpp
This is a helper function to set the `portVector` of a `std::vector<u16>` and specify the port string and the line number in the `lineno`.
```
const char *probename;
```cpp
This parameter is the name of the ServiceProbe. It is used to return the details of the nth match.
```
const u8 *probestring;
```cpp
This parameter is the ProbeBestRing of the ServiceProbe. It is used to return the details of the nth match.
```
int lineno;
```cpp
This parameter is the line number in the `linenumber` provided. It is used to return the details of the nth match.
```


```cpp
class ServiceProbe {
 public:
  ServiceProbe();
  ~ServiceProbe();
  const char *getName() const { return probename; }
  // Returns true if this is the "null" probe, meaning it sends no probe and
  // only listens for a banner.  Only TCP services have this.
  bool isNullProbe() const { return (probestringlen == 0); }
// Amount of time to wait after a connection succeeds (or packet sent) for a responses.
  int totalwaitms;
  // If the connection succeeds but closes before this time, it's tcpwrapped.
  int tcpwrappedms;

  // Parses the "probe " line in the nmap-service-probes file.  Pass the rest of the line
  // after "probe ".  The format better be:
  // [TCP|UDP] [probename] "probetext"
  // the lineno is requested because this function will bail with an error
  // (giving the line number) if it fails to parse the string.
  void setProbeDetails(char *pd, int lineno);

  // obtains the probe string (in raw binary form) and the length.  The string will be
  // NUL-terminated, but there may be other \0 in the string, so the termination is only
  // done for ease of printing ASCII probes in debugging cases.
  const u8 *getProbeString(int *stringlen) const { *stringlen = probestringlen; return probestring; }
  void setProbeString(const u8 *ps, int stringlen);

  /* Protocols are IPPROTO_TCP and IPPROTO_UDP */
  u8 getProbeProtocol() const {
    assert(probeprotocol == IPPROTO_TCP || probeprotocol == IPPROTO_UDP);
    return probeprotocol;
  }
  void setProbeProtocol(u8 protocol) { probeprotocol = protocol; }

  // Takes a string as given in the 'ports '/'sslports ' line of
  // nmap-service-probes.  Pass in the list from the appropriate
  // line.  For 'sslports', tunnel should be specified as
  // SERVICE_TUNNEL_SSL.  Otherwise use SERVICE_TUNNEL_NONE.  The line
  // number is requested because this function will bail with an error
  // (giving the line number) if it fails to parse the string.  Ports
  // are a comma separated list of ports and ranges
  // (e.g. 53,80,6000-6010).
  void setProbablePorts(enum service_tunnel_type tunnel,
                        const char *portstr, int lineno);

  /* Returns true if the passed in port is on the list of probable
     ports for this probe and tunnel type.  Use a tunnel of
     SERVICE_TUNNEL_SSL or SERVICE_TUNNEL_NONE as appropriate */
  bool portIsProbable(enum service_tunnel_type tunnel, u16 portno) const;
  // Returns true if the passed in service name is among those that can
  // be detected by the matches in this probe;
  bool serviceIsPossible(const char *sname) const;

  // Takes a string following a Rarity directive in the probes file.
  // The string should contain a single integer between 1 and 9. The
  // default rarity is 5. This function will bail if the string is invalid.
  void setRarity(const char *portstr, int lineno);

  // Simply returns the rarity of this probe
  int getRarity() const { return rarity; }

  // Takes a match line in a probe description and adds it to the
  // list of matches for this probe.  This function should be passed
  // the whole line starting with "match" or "softmatch" in
  // nmap-service-probes.  The line number is requested because this
  // function will bail with an error (giving the line number) if it
  // fails to parse the string.
  void addMatch(const char *match, int lineno);

  // If the buf (of length buflen) matches one of the regexes in this
  // ServiceProbe, returns the details of the nth match (service name,
  // version number if applicable, and whether this is a "soft" match.
  // If the buf doesn't match, the serviceName field in the structure
  // will be NULL.  The MatchDetails returned is only valid until the
  // next time this function is called.  The only exception is that the
  // serviceName field can be saved throughout program execution.  If
  // no version matched, that field will be NULL. This function may
  // return NULL if there are no match lines at all in this probe.
  const struct MatchDetails *testMatch(const u8 *buf, int buflen, int n);

  char *fallbackStr;
  ServiceProbe *fallbacks[MAXFALLBACKS+1];
  std::vector<u16>::const_iterator probablePortsBegin() const {return probableports.begin();}
  std::vector<u16>::const_iterator probablePortsEnd() const {return probableports.end();}
  bool notForPayload;

 private:
  void setPortVector(std::vector<u16> *portv, const char *portstr,
                                 int lineno);
  const char *probename;

  const u8 *probestring;
  int probestringlen;
  std::vector<u16> probableports;
  std::vector<u16> probablesslports;
  int rarity;
  std::vector<const char *> detectedServices;
  int probeprotocol;
  std::vector<ServiceProbeMatch *> matches; // first-ever use of STL in Nmap!
};

```

这是一个名为 `AllProbes` 的类，它用于对服务进行扫描。该类包括一些用于查找特定名称和协议的服务探针，以及一个指向 `nullProbe` 的指针。下面是类的详细说明：

1. `AllProbes()`：构造函数。初始化时创建一个空服务探针数组。
2. `~AllProbes()`：析构函数。释放服务探针数组和 `nullProbe`。
3. `getProbeByName(const char *name, int proto) const`：尝试查找给定名称和协议的服务探针。如果找不到匹配的探针，将返回 NULL。
4. `nullProbe`：一个指向 `ServiceProbe` 的指针，用于表示服务探针为空。
5. `probes`：一个代表所有服务探针的向量。
6. `ServiceProbe *nullProbe`：一个指向 `nullProbe` 的指针。
7. `void compileFallbacks()`：在所有服务探针之前，将每个探针的 `fallbackStr` 字段填充为给定名称和协议的探针指针。
8. `int isExcluded(unsigned short port, int proto) const`：检查给定端口和服务协议是否被排除。
9. `bool excluded_seen`：一个标志，指示是否已经检测到被排除的服务。
10. `struct scan_lists excludedports`：一个结构体，用于存储已被排除的端口扫描列表。
11. `static AllProbes *service_scan_init(void)`：返回一个指向 `AllProbes` 的指针，用于在服务中执行扫描。
12. `static void service_scan_free(void)`：释放 `service_scan_init`。
13. `static int check_excluded_port(unsigned short port, int proto)`：检查给定端口和服务协议是否被排除。

该类的方法和成员函数用于在服务中执行扫描，并在给定名称和协议的服务探针之间查找符合条件的服务探针。该类还可以通过 `compileFallbacks()` 函数在服务探针中查找指定的端口和协议，以及通过 `excluded_seen` 标志记录哪些服务探针已被排除。


```cpp
class AllProbes {
public:
  AllProbes();
  ~AllProbes();
  // Tries to find the probe in this AllProbes class which have the
  // given name and protocol. If no match is found for the requested
  // protocol it will try to find matches on any protocol.
  // It can return the NULL probe.
  ServiceProbe *getProbeByName(const char *name, int proto) const;
  std::vector<ServiceProbe *> probes; // All the probes except nullProbe
  ServiceProbe *nullProbe; // No probe text - just waiting for banner

  // Before this function is called, the fallbacks exist as unparsed
  // comma-separated strings in the fallbackStr field of each probe.
  // This function fills out the fallbacks array in each probe with
  // an ordered list of pointers to which probes to try. This is both for
  // efficiency and to deal with odd cases like the NULL probe and falling
  // back to probes later in the file. This function also free()s all the
  // fallbackStrs.
  void compileFallbacks();

  int isExcluded(unsigned short port, int proto) const;
  bool excluded_seen;
  struct scan_lists excludedports;

  static AllProbes *service_scan_init(void);
  static void service_scan_free(void);
  static int check_excluded_port(unsigned short port, int proto);
```



这段代码是一个C++程序，定义了一个名为`AllProbes`的类，以及两个静态函数`parse_nmap_service_probe_file`和`service_scan`。

`parse_nmap_service_probe_file`函数的作用是解析给定的nmap-service-probes文件，并将其转换为`AllProbes`类的静态成员变量`global_AP`。

`service_scan`函数的作用是执行一个服务指纹扫描，该扫描将针对指定目标的所有开放端口进行。


```cpp
protected:
  static AllProbes *global_AP;
};

/**********************  PROTOTYPES  ***********************************/

/* Parses the given nmap-service-probes file into the AP class Must
   NOT be made static because I have external maintenance tools
   (servicematch) which use this */
void parse_nmap_service_probe_file(AllProbes *AP, const char *filename);

/* Execute a service fingerprinting scan against all open ports of the
   Targets specified. */
int service_scan(std::vector<Target *> &Targets);

```

这是一个预处理指令，它检查一个文件名(“SERVICE_SCAN_H.c”)是否在该目录下存在。如果文件名不存在，那么这段代码将编译失败，不会产生任何执行代码。

如果没有预处理指令，编译器会默认在源代码文件前添加一些预处理指令，比如 #include 和 #define。这些指令有助于程序在编译之前对某些符号进行定义，或者告诉编译器如何引用其他源文件。然而，在某些 cases 下，预处理指令可能被省略，因此需要手动添加。


```cpp
#endif /* SERVICE_SCAN_H */


```

# `string_pool.h`

This is a text document that is part of the Nmap software. It explains the licensing terms for Nmap, which is a network tool for testing and analyzing network traffic.

The text mentions that Nmap is released under several different licenses, including an Nmap license, an Nmap OEM license, and the Npcap software license. The Nmap license and the Nmap OEM license have more permissive terms than the Npcap license, and allow companies to use and redistribute Nmap in commercial products.

The text also notes that if a company has received an Nmap license agreement or contract with other terms, they may choose to use and redistribute Nmap under those terms instead. However, the Npcap license and the Nmap OEM license have stricter terms that prohibit redistribution without special permission.

The text then goes on to explain the licensing terms for the source code for Nmap. The source code is released under the Github license, which allows companies to use, modify, and redistribute the source code as long as they follow the terms of the Github license.

Finally, the text emphasizes that the source code for Nmap is provided "as-is" and that users have the right to know what the software does before running it. It also mentions that the software may not be redistributed without special permission, but users are encouraged to submit changes to the dev@nmap.org mailing list for possible incorporation into the main distribution.


```cpp
/***************************************************************************
 * string_pool.h -- String interning for memory optimization               *
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

这段代码定义了一个名为“String Pool”的头部文件，它提供了对字符串的管理和操作。

在文件头部分，定义了一个名为“STRING_POOL_H”的标识符。接下来的部分定义了一系列函数，用于管理字符串池，其中最重要的是string_pool_insert函数。

string_pool_insert函数的作用是，在第一次调用时给定的字符串将分配内存并存储在静态池中。之后，每次调用该函数时，它将返回一个指向存储在静态池中的字符串的指针，而不是再次在内存中分配相同字符串的内存。

string_pool_sprintf函数是一个format函数，用于将格式化字符串中的字符串和格式化参数注入到字符串池中，然后使用sprintf函数来格式化输出字符串。这个函数可以接受一个格式化字符串，一个或多个参数，以及多个格式化参数。

总的来说，这段代码定义了一个字符串池，用于在程序运行过程中管理使用频率较低的字符串，以避免多次内存分配和释放。通过使用string_pool_insert函数可以在字符串池中插入新的字符串，而通过使用string_pool_sprintf函数可以方便地将格式化字符串注入到字符串池中，并使用sprintf函数来格式化输出字符串。


```cpp
#ifndef STRING_POOL_H
#define STRING_POOL_H

/* The OS database consists of many small strings, many of which appear
   thousands of times. It pays to allocate memory only once for each unique
   string, and have all references point at the one allocated value. */

/* Store a string uniquely. The first time this function is called with a
   certain string, it allocates memory and stores a copy of the string in a
   static pool. Thereafter it will return a pointer to the saved string instead
   of allocating memory for an identical one. */
const char *string_pool_insert(const char *s);

/* Format a string with sprintf and insert it with string_pool_insert. */
const char *string_pool_sprintf(const char *fmt, ...);

```

这三段代码都是用于实现字符串池的工具函数。

第一段代码定义了一个名为`string_pool_substr`的函数，它接收两个参数，一个是字符串`s`，另一个是字符串`t`，然后返回在`s`和`t`之间的子字符串。这个子字符串不包括字符串的左右空格，但包括左边的空格。

第二段代码定义了一个名为`string_pool_substr_strip`的函数，它与`string_pool_substr`类似，但去除左右边的空格。

第三段代码定义了一个名为`string_pool_strip_word`的函数，它接收一个参数`s`，表示要找到的单词的起始位置，然后直到遇到左边的空格为止，返回该单词。如果只有左边的空格，函数返回`NULL`。


```cpp
/* Store the string in the interval [s,t) */
const char *string_pool_substr(const char *s, const char *t);

/* Store the string in the interval [s,t), stripping whitespace from both ends. */
const char *string_pool_substr_strip(const char *s, const char *t);

/* Skip over whitespace to find the beginning of a word, then read until the
   next whitespace character. Returns NULL if only whitespace is found. */
const char *string_pool_strip_word(const char *s, const char *end);

#endif // STRING_POOL_H

```

# `struct_ip.h`

这段代码是一个 C 语言库中的定义，定义了一个名为 "ip" 的结构体成员。这个结构体成员包含了如下的成员：

```cpp
struct ip {
   struct	ip_firstfour ip_ff;

   #define ip_v                  ip_ff.ip_fv
   #define ip_hl                ip_ff.ip_fhl
   #define ip_vhl               ip_ff.ip_fvhl
   #define ip_tos                ip_ff.ip_ftos
   #define ip_len                ip_ff.ip_flen
};
```

这个代码的作用是定义了一个名为 "ip" 的结构体，其中包括了 "ip_firstfour" 和一些 defined macro 成员，如 "ip_v"、"ip_hl" 等。这些成员的名称和定义可以在 netinet/ip.h 中找到，但是由于这个 library 是用 AIX 定义的，因此这些定义对于其他文件或者用其他方式使用的人来说都是不可见的。

此外，这个 library 的定义在包含它的源文件后，即使其他文件中定义了 "ip" 这类名称，也是无效的，因为这些定义已经被定义为库中的成员，不能再其他地方被使用。


```cpp
/* The C library on AIX defines the names of various members of struct ip to
   something else in <netinet/ip.h>:

struct ip {
        struct	ip_firstfour ip_ff;
#define	ip_v	ip_ff.ip_fv
#define	ip_hl	ip_ff.ip_fhl
#define	ip_vhl	ip_ff.ip_fvhl
#define	ip_tos	ip_ff.ip_ftos
#define	ip_len	ip_ff.ip_flen

   This breaks code that actually wants to use names like ip_v for its own
   purposes, like struct ip_hdr in libdnet. The AIX definitions will work
   if they are included late in the list of includes, before other code that
   might want to use the above names has already been preprocessed. The
   includes that end up defining struct ip are therefore limited to this
   file, so it can be included in a .cc file after other .h have been
   included. */

```

这段代码定义了一系列类型定义，用于在编译时检查源代码文件是否使用了NetBSD（Net-宝莱坞硅）规范。这些类型定义包括：

```cpp
u_int32_t：32位无符号整数类型
u_int16_t：16位无符号整数类型
u_int8_t：8位无符号整数类型
```

在Linux系统上，这些类型定义通常用于`<inttypes.h>`和`<netinet/ip.h>`头文件中，以正确地使用struct `ip`结构体。

```cpp
#ifdef WIN32
```

这是一个预处理指令，用于告诉编译器在编译之前包含这些定义。如果在源代码文件中已经定义了这些类型定义，则编译器将不再需要包含它们。

```cpp
#ifndef __FAVOR_BSD
#define __FAVOR_BSD
#endif
```

这是另一个预处理指令，用于告诉编译器在编译之前包含这个定义。如果当前源代码文件是在NetBSD规范下使用的，则这个定义是多余的，编译器不会对此产生任何影响。

```cpp
#ifndef __USE_BSD
#define __USE_BSD
#endif
```

这也是一个预处理指令，用于告诉编译器在编译之前包含这个定义。如果当前源代码文件不是在NetBSD规范下使用的，则这个定义将不再被编译器接受，编译器将对此产生错误提示。

```cpp
#ifndef _BSD_SOURCE
#define _BSD_SOURCE
#endif
```

这同样是预处理指令，用于告诉编译器在编译之前包含这个定义。这个指令的作用是告诉编译器，如果当前源代码文件是在NetBSD规范下使用的，但是它不是NetBSD规范的源代码文件，那么这个定义也将不再被编译器接受。


```cpp
#ifdef WIN32
typedef unsigned __int32 u_int32_t;
typedef unsigned __int16 u_int16_t;
typedef unsigned __int8 u_int8_t;
#endif
/* Linux uses these defines in netinet/ip.h to use the correct struct ip */
#ifndef __FAVOR_BSD
#define __FAVOR_BSD
#endif
#ifndef __USE_BSD
#define __USE_BSD
#endif
#ifndef _BSD_SOURCE
#define _BSD_SOURCE
#endif

```

这段代码是为了保证BSDI系统的正确性而编写的一个保护代码。它定义了一个变量 `_IP_VHL`，并将其定义为 `0`。

接下来的几行代码定义了一个宏定义 `#undef _IP_VHL`，它的作用是覆盖 `_IP_VHL` 变量，将其定义为 `0`。

接着，代码开始检查两个条件。第一个条件是 `#ifndef NETINET_IN_SYSTM_H`，如果这个条件为真，那么下一行代码将包含一些网络接口头文件，定义了 `n_long` 类型的变量 `_IP_VHL`。第二个条件是 `#if HAVE_NET_IF_H`，如果这个条件为真，那么下一行代码将包含 `netif.h` 头文件，定义了 `net_if` 类型变量。

如果前两个条件都不满足，那么第三行代码将包含 `ip.h` 头文件，定义了 `ip` 类型变量。这些变量将在定义之后被初始化，并可以被使用在代码中。


```cpp
/* BSDI needs this to insure the correct struct ip */
#undef _IP_VHL

#ifndef NETINET_IN_SYSTM_H  /* This guarding is needed for at least some versions of OpenBSD */
#include <netinet/in_systm.h> /* defines n_long needed for netinet/ip.h */
#define NETINET_IN_SYSTM_H
#endif
#if HAVE_NET_IF_H
#ifndef NET_IF_H /* This guarding is needed for at least some versions of OpenBSD */
#include <net/if.h>
#define NET_IF_H
#endif
#endif
#ifndef NETINET_IP_H  /* This guarding is needed for at least some versions of OpenBSD */
#include <netinet/ip.h>
```

这段代码是一个头文件，它定义了一个名为NETINET_IP_H的宏。如果这个宏被定义了，那么它会在编译时展开为以下代码：
```cpp
#include <net/if_arp.h>
```
然后是网络头文件，它包含了一个名为arp_h的函数。
```cpp
#include <netinet/ip_icmp.h>
```
接下来是关于IP头结构的定义。
```cpp
#define NETINET_IP_H
```
总的来说，这段代码的作用是定义了一个名为NETINET_IP_H的宏，它使用了预处理器指令#include和#define。此外，还包含了一个名为arp_h的函数，它从网络头文件中导入了IP头结构。


```cpp
#define NETINET_IP_H
#endif
#include <net/if_arp.h>

#ifndef WIN32
#include <netinet/ip_icmp.h>
#endif

#ifndef HAVE_STRUCT_IP
#define HAVE_STRUCT_IP

/* From Linux glibc, which apparently borrowed it from
   BSD code.  Slightly modified for portability --fyodor@nmap.org */
/*
 * Structure of an internet header, naked of options.
 */
```

这是一个定义了IPv4头部长度的结构体。它定义了以下字段：

1. `ip_v`：IPv4版本号，使用4位二进制表示法，值为0表示IPv4头部长度，值为1表示IPv6头部长度。
2. `ip_hl`：IPv4头部长度，使用4位二进制表示法，与`ip_v`字段中的值为1一起使用，表示IPv6头部长度。
3. `ip_tos`：IPv4类型 of service，使用2位二进制表示法，表示服务类型，其值从0到1512000000000调整为：
  - 0：特殊用途
  - 1：关键路由器
  - 2：网络地址
  - 3：持久性虚拟专用网络（PvPN）
  - 4：ǔ士
  - 5：src（源）
  - 6：dst（目标）
  - 7：缆线
  - 8：礼品带
  - 9：矿工
  - 10：稀有
  - 11：试用
  - 12：永久
  - 13：类型转换
  - 14：自发 deprecated
  - 15：Markator（標記者）
4. `ip_len`：IPv4头部长度，使用4位二进制表示法，与`ip_v`字段中的值为1一起使用，表示IPv6头部长度。
5. `ip_id`：IPv4标识，使用16位二进制表示法，与IPv4头部长度一起使用，用于标识网络中的设备。
6. `ip_off`：IPv4分片偏移量，使用4位二进制表示法，与IPv6头部长度一起使用，用于标识分片中的偏移量。
7. `ip_rf`：保留字段，用于指示是否为保留分片。
8. `ip_df`：保留字段，用于指示是否为保留分片。


```cpp
struct ip
  {
#if WORDS_BIGENDIAN
    u_int8_t ip_v:4;                    /* version */
    u_int8_t ip_hl:4;                   /* header length */
#else
    u_int8_t ip_hl:4;                   /* header length */
    u_int8_t ip_v:4;                    /* version */
#endif
    u_int8_t ip_tos;                    /* type of service */
    u_short ip_len;                     /* total length */
    u_short ip_id;                      /* identification */
    u_short ip_off;                     /* fragment offset field */
#define IP_RF 0x8000                    /* reserved fragment flag */
#define IP_DF 0x4000                    /* don't fragment flag */
```

这段代码定义了一个结构体，表示IP数据报中的TTL（Time to Live）字段。IP数据报中的TTL字段表示数据报在网络中允许经过的最大路由数量。TTL字段的长度为4字节，它的值越小，表示数据报在网络中可以经过的路由数量就越少。

该结构体中还定义了一个与TTL相关的标志位，名为IP_MF。如果IP_MF为1，则表示数据报使用了fragment（分片）机制。分片机制可以将一个IP数据报分成多个片段，以便在网络上以分片的形式传输。

此外，该结构体中还定义了一个名为ip_ttl的IP数据报TTL字段，一个名为ip_sum的IP数据报 checksum 字段，以及一个名为ip_src的IP数据报 source 地址字段和一个名为ip_dst的IP数据报 destination 地址字段。这些字段分别用于表示IP数据报中的TTL、checksum和 source、destination 地址。


```cpp
#define IP_MF 0x2000                    /* more fragments flag */
#define IP_OFFMASK 0x1fff               /* mask for fragmenting bits */
    u_int8_t ip_ttl;                    /* time to live */
    u_int8_t ip_p;                      /* protocol */
    u_short ip_sum;                     /* checksum */
    struct in_addr ip_src, ip_dst;      /* source and dest address */
  };

#endif /* HAVE_STRUCT_IP */

#ifndef HAVE_STRUCT_ICMP
#define HAVE_STRUCT_ICMP
/* From Linux /usr/include/netinet/ip_icmp.h GLIBC */

/*
 * Internal of an ICMP Router Advertisement
 */
```

这段代码定义了一个名为icmp_ra_addr的结构体，该结构体有两个成员：ira_addr和ira_preference。

icmp_ra_addr结构体是ICMP（Internet Control Message Protocol，互联网报文协议）中的一个结构体，它包含一个32位的IP地址和一个32位的优先级。

icmp是一个结构体，它包含一个8位的icmp类型，一个8位的icmp代码，以及一个16位的icmp校验和。此外，该结构体还包括一个由两个 union 组成的结构体，分别是icmp_idseq和icmp_pmtu。

icmp_idseq结构体包含一个32位的icd_id和一个32位的icd_seq，它们用于表示IP数据报中的ID和序列号。

icmp_pmtu结构体包含一个16位的ipm_void和一个16位的ipm_nextmtu，它们用于表示是否有NEXT-MESSAGE-TRANSPORT-ID和NEXT-MESSAGE-TRANSPORT-LENGTH字段，如果没有这些字段，则表示数据报无法确定下一个头部的长度。

icmp_hun结构体包含一个8位的irt_num_addrs，一个8位的irt_wpa和一个32位的irt_lifetime。

该代码定义的icmp结构体是用于在IP数据报中发送ICMP请求（如RFC895）和响应（如RFC1191）的结构体。它包含一个IP地址和一个优先级，用于标识发送者和接收者。


```cpp
struct icmp_ra_addr
{
  u_int32_t ira_addr;
  u_int32_t ira_preference;
};

struct icmp
{
  u_int8_t  icmp_type;  /* type of message, see below */
  u_int8_t  icmp_code;  /* type sub code */
  u_int16_t icmp_cksum; /* ones complement checksum of struct */
  union
  {
    struct ih_idseq             /* echo datagram */
    {
      u_int16_t icd_id;
      u_int16_t icd_seq;
    } ih_idseq;
    u_int32_t ih_void;

    /* ICMP_UNREACH_NEEDFRAG -- Path MTU Discovery (RFC1191) */
    struct ih_pmtu
    {
      u_int16_t ipm_void;
      u_int16_t ipm_nextmtu;
    } ih_pmtu;

    struct ih_rtradv
    {
      u_int8_t irt_num_addrs;
      u_int8_t irt_wpa;
      u_int16_t irt_lifetime;
    } ih_rtradv;
  } icmp_hun;
  /* Removed icmp_pptr and icmp_gwaddr from union and #defines because they conflict with dnet */
```

这段代码定义了ICMP包类型定义结构体icmp_dun，包括其ID、SEQ、VOID、PMODULE、NEXTMOTEU、ADDRs、WPA、LIFETIME等字段。

具体来说，定义了四个宏定义：icmp_id、icmp_seq、icmp_void、icmp_pmvoid，分别对应ICMP包的ID、SEQ、VOID、PMODULE字段。

接着定义了一个名为icmp_dun的结构体，其字段包括：

- id_ts：包含三个字段，分别表示ID、SEQ、TTIME，用于保存ICMP包的ID、SEQ、TTIME信息。
- id_ip：包含两个字段，分别为ID和SE(IP数据报中的ID字段和数据报类型)，用于保存IP数据报中的ID字段和数据报类型信息。
- id_radv：包含一个字段，表示ICMP包中的RADV类型，例如RED、GRE、或者为空。
- id_mask：用于标识该包是否使用MASK类型，如果是，则包含一个字段，表示该包使用的MASK类型。
- id_data：包含一个字段，用于保存该包中的数据部分，可以是一个IP数据报或者一个MASK类型的数据。如果是MASK类型，则包含一个包含多个字节的IP数据报。

此外，还定义了一个名为icmp_hun的硬件抽象层结构体，可能用于操作系统或者硬件层对接。


```cpp
#define icmp_id         icmp_hun.ih_idseq.icd_id
#define icmp_seq        icmp_hun.ih_idseq.icd_seq
#define icmp_void       icmp_hun.ih_void
#define icmp_pmvoid     icmp_hun.ih_pmtu.ipm_void
#define icmp_nextmtu    icmp_hun.ih_pmtu.ipm_nextmtu
#define icmp_num_addrs  icmp_hun.ih_rtradv.irt_num_addrs
#define icmp_wpa        icmp_hun.ih_rtradv.irt_wpa
#define icmp_lifetime   icmp_hun.ih_rtradv.irt_lifetime
  union
  {
    struct
    {
      u_int32_t its_otime;
      u_int32_t its_rtime;
      u_int32_t its_ttime;
    } id_ts;
    struct
    {
      struct ip idi_ip;
      /* options and then 64 bits of data */
    } id_ip;
    struct icmp_ra_addr id_radv;
    u_int32_t   id_mask;
    u_int8_t    id_data[1];
  } icmp_dun;
```

这段代码定义了一系列用于传递ICMP包统计信息的可重定义函数。其中：

```cpp
#define icmp_otime      icmp_dun.id_ts.its_otime
#define icmp_rtime      icmp_dun.id_ts.its_rtime
#define icmp_ttime      icmp_dun.id_ts.its_ttime
#define icmp_ip         icmp_dun.id_ip.idi_ip
#define icmp_radv       icmp_dun.id_radv
#define icmp_mask       icmp_dun.id_mask
#define icmp_data       icmp_dun.id_data
```

定义的这些函数接受一个名为 `icmp_dun` 的结构体作为参数，然后返回对应于 `id_ts` 和 `id_ip` 的成员的值。这些成员是 `icmp_ts` 和 `icmp_ip` 的成员变量，它们用于跟踪发送和接收ICMP包的时间和IP地址。

这个代码片段可能用于监测和管理网络中的ICMP流量。


```cpp
#define icmp_otime      icmp_dun.id_ts.its_otime
#define icmp_rtime      icmp_dun.id_ts.its_rtime
#define icmp_ttime      icmp_dun.id_ts.its_ttime
#define icmp_ip         icmp_dun.id_ip.idi_ip
#define icmp_radv       icmp_dun.id_radv
#define icmp_mask       icmp_dun.id_mask
#define icmp_data       icmp_dun.id_data
};
#endif /* HAVE_STRUCT_ICMP */


```

# `Target.h`

This is a text document that provides information about the Nmap software and its licensing. Nmap is a network scanner that is used for identifying and analyzing networks, as well as providing information about the ports they are open and the services running on those ports.

The Nmap license is a text-based license that generally prohibits companies from using and redistributing Nmap in commercial products. However, Nmap sells a special Nmap OEM Edition license with a more permissive license and special features.

If you have received an Nmap license with strict terms, you may use and redistribute Nmap under those terms. However, if you have received a Nmap license with more permissive terms, you may use and redistribute Nmap under those terms. The official Nmap Windows builds include the Npcap software for packet capture and transmission. The source code for Nmap is available on GitHub and allows users to port Nmap to new platforms, fix bugs, and add new features.

The Nmap license is under the Nmap Public Source License Contributor Agreement, which is a text-based license that allows users to offer contributions to the project in exchange for certain rights. The free version of Nmap is distributed in the hope that it will be useful, but WITHOUT ANY WARRANTY; without even the implied warranty of MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. Warranties, indemnification, and commercial support are all available through the Npcap OEM program.


```cpp

/***************************************************************************
 * Target.h -- The Target class encapsulates much of the information Nmap  *
 * has about a host.  Results (such as ping, OS scan, etc) are stored in   *
 * this class as they are determined.                                      *
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

这段代码是一个C语言代码片段，定义了一个名为“target.h”的头文件。

该代码包含以下内容：

1. 定义了一个名为“$Id$”的标识，用于标识该代码片段，方便代码的版本管理和调试。

2. 定义了一个名为“TARGET_H”的定义，该定义的内容在下面的“TARGET_H.h”文件中定义。这里定义了一个头文件，因此需要在头文件目录加上一层“/”。

3. 引入了“nbase.h”头文件，可能是用于定义网络基本数据结构。

4. 引入了“libnetutil/netutil.h”头文件，可能是用于网络操作的。

5. 包含了一个名为“nse_main.h”的头文件，可能是用于定义纳西语（NDS）主程序的。

6. 包含了一个名为“portreasons.h”的头文件，可能是用于定义港口不可用性报告的。

7. 包含了一个名为“portlist.h”的头文件，可能是用于定义港口列表的。

8. 定义了一个名为“TARGET_PORT”的变量，但没有具体的定义。

9. 没有其他明显的输出，因此该代码片段可能用于内部处理，而不需要输出到外部文件或作为应用程序的输入或输出。


```cpp
/* $Id$ */

#ifndef TARGET_H
#define TARGET_H

#include "nbase.h"

#include "libnetutil/netutil.h" /* devtype */

#ifndef NOLUA
#include "nse_main.h"
#endif

#include "portreasons.h"
#include "portlist.h"
```



该代码包括多个头文件和类定义，用于实现对OSCan协议的检测和分析。

- `probespec.h` 和 `osscan.h` 包含了一些通用的函数和数据结构，定义了 OSSCan 协议的一些通用概念，如检测、分析、配置等。

- `FingerPrintResults` 是一个类，用于存储 FingerPrint 结果的详细信息，包括扫描到的指纹类型、时间戳等。

- `#include <list>` 和 `#include <string>` 包含了一个标准库的头文件 `<list>` 和 `<string>`，用于支持链表和字符串。

- `#include <vector>` 和 `#include <time.h>` 包含了一个标准库的头文件 `<vector>` 和 `<time.h>`，用于支持向量和时间相关的操作。

- `INET6_ADDRSTRLEN` 定义了一个常量，表示 Internet 6 层地址字符串的长度，通常用于处理 IPv6 地址。

- `osscan_flags` 定义了一个枚举类型 `osscan_flags`，用于表示 OSSCan 协议的检测方式，包括 OS_NOTPERF、OS_PERF 和 OS_PERF_UNREL 等选项。

- `#include "osscan2.h"` 和 `#include "probespec.h"` 包含了一个标准库的头文件 `osscan2.h` 和 `probespec.h`，用于 OSSCan 协议的分析和检测。

- `#include <iostream>` 和 `#include <fstream>` 包含了一个标准库的头文件 `<iostream>` 和 `<fstream>`，用于输入和输出数据。


```cpp
#include "probespec.h"
#include "osscan.h"
#include "osscan2.h"
class FingerPrintResults;

#include <list>
#include <string>
#include <vector>
#include <time.h> /* time_t */

#ifndef INET6_ADDRSTRLEN
#define INET6_ADDRSTRLEN 46
#endif

enum osscan_flags {
        OS_NOTPERF=0, OS_PERF, OS_PERF_UNREL
};

```

该代码定义了一个名为 host_timeout_nfo 的结构体，用于表示主机连接超时的情况。

该结构体包含以下成员：

1. `msecs_used`：表示已使用的毫秒数，用于记录主机连接超时所用的时间。
2. `toclock_running`：表示当前是否正在运行时钟，即时间是否正确。
3. `toclock_start`：表示时钟开始的时间。
4. `host_start` 和 `host_end`：表示主机连接的起始和结束时间，均为 `time_t` 类型。

该结构体定义了 TracerouteHop 结构体，用于在主机之间传递 Traceroute 信息。TracerouteHop 结构体包含以下成员：

1. `tag`：表示目标套接字，用于在传输过程中标识数据包。
2. `timedout`：表示是否超时，如果已经超时，则可以设置为 `false`。
3. `name`：表示显示名称，用于在传输过程中显示目标主机的信息。
4. `addr`：表示目标主机的服务器地址，用于在传输过程中定位目标主机。
5. `ttl`：表示 TTL（生存时间），用于限制数据包在 Traceroute 中的生存时间。
6. `rtt`：表示往返时间，用于计算数据包从发送者到接收者的响应时间。

最后，该代码没有输出任何内容，只是定义了上述结构体和函数，用于在后续代码中使用。


```cpp
struct host_timeout_nfo {
  unsigned long msecs_used; /* How many msecs has this Target used? */
  bool toclock_running; /* Is the clock running right now? */
  struct timeval toclock_start; /* When did the clock start? */
  time_t host_start, host_end; /* The absolute start and end for this host */
};

struct TracerouteHop {
  struct sockaddr_storage tag;
  bool timedout;
  std::string name;
  struct sockaddr_storage addr;
  int ttl;
  float rtt; /* In milliseconds. */

  int display_name(char *buf, size_t len) const {
    if (name.empty())
      return Snprintf(buf, len, "%s", inet_ntop_ez(&addr, sizeof(addr)));
    else
      return Snprintf(buf, len, "%s (%s)", name.c_str(), inet_ntop_ez(&addr, sizeof(addr)));
  }
};

```

This is a Linux kernel header file that defines the select function for network interfaces. Select is a function that allows you to select one of several network interfaces.

The header file defines several functions related to the select function:

* devtype: This type is used to indicate the type of the selected interface. It can be one of several specified devtypes, such as "loopback", "devt\_p2p", or "devt\_other".
* setIfType: This function is used to set the type of the selected interface.
* interface\_type: This variable is used to keep track of the type of the selected interface. It is initialized to the first argument passed to setIfType().
* setTimeOutClock: This function is used to start or stop the timeout clock for the selected interface.
* startTimeOutClock: This function is used to start the timeout clock for the selected interface.
* stopTimeOutClock: This function is used to stop the timeout clock for the selected interface.
* timedOut: This function is used to check if the selected interface is timedout.
* StartTime: This function is used to return the start time of the selected interface.
* EndTime: This function is used to return the end time of the selected interface.
* setMACAddress: This function is used to set the MAC address of the selected interface.
* setSrcMACAddress: This function is used to set the MAC address of the source MAC address of the selected interface.
* setNextHopMACAddress: This function is used to set the MAC address of the next hop for the selected interface.
* getMACAddress: This function is used to return the MAC address of the selected interface.
* getSrcMACAddress: This function is used to return the MAC address of the source MAC address of the selected interface.
* getNextHopMACAddress: This function is used to return the MAC address of the next hop for the selected interface.

The functions defined in this header file are used to set and retrieve the various configuration options of the selected network interface.


```cpp
struct EarlySvcResponse {
  probespec pspec;
  int len;
  u8 data[1];
};

class Target {
 public: /* For now ... TODO: a lot of the data members should be made private */
  Target();
  ~Target();
  /* Returns the address family of the destination address. */
  int af() const;
  /* Fills a sockaddr_storage with the AF_INET or AF_INET6 address
     information of the target.  This is a preferred way to get the
     address since it is portable for IPv6 hosts.  Returns 0 for
     success. ss_len must be provided.  It is not examined, but is set
     to the size of the sockaddr copied in. */
  int TargetSockAddr(struct sockaddr_storage *ss, size_t *ss_len) const;
  const struct sockaddr_storage *TargetSockAddr() const;
  /* Note that it is OK to pass in a sockaddr_in or sockaddr_in6 casted
     to sockaddr_storage */
  void setTargetSockAddr(const struct sockaddr_storage *ss, size_t ss_len);
  // Returns IPv4 target host address or {0} if unavailable.
  struct in_addr v4host() const;
  const struct in_addr *v4hostip() const;
  const struct in6_addr *v6hostip() const;
  /* The source address used to reach the target */
  int SourceSockAddr(struct sockaddr_storage *ss, size_t *ss_len) const;
  const struct sockaddr_storage *SourceSockAddr() const;
  /* Note that it is OK to pass in a sockaddr_in or sockaddr_in6 casted
     to sockaddr_storage */
  void setSourceSockAddr(const struct sockaddr_storage *ss, size_t ss_len);
  struct sockaddr_storage source() const;
  const struct in_addr *v4sourceip() const;
  const struct in6_addr *v6sourceip() const;
  /* The IPv4 or IPv6 literal string for the target host */
  const char *targetipstr() const { return targetipstring; }
  /* The IPv4 or IPv6 literal string for the source address */
  const char *sourceipstr() const { return sourceipstring; }
  /* Give the name from the last setHostName() call, which should be
   the name obtained from reverse-resolution (PTR query) of the IP (v4
   or v6).  If the name has not been set, or was set to NULL, an empty
   string ("") is returned to make printing easier. */
  const char *HostName() const { return hostname? hostname : "";  }
  /* You can set to NULL to erase a name or if it failed to resolve -- or
     just don't call this if it fails to resolve.  The hostname is blown
     away when you setTargetSockAddr(), so make sure you do these in proper
     order
  */
  void setHostName(const char *name);
  /* Generates a printable string consisting of the host's IP
     address and hostname (if available).  Eg "www.insecure.org
     (64.71.184.53)" or "fe80::202:e3ff:fe14:1102".  The name is
     written into the buffer provided, which is also returned.  Results
     that do not fit in buflen will be truncated. */
  const char *NameIP(char *buf, size_t buflen) const;
  /* This next version returns a STATIC buffer -- so no concurrency */
  const char *NameIP() const;

  /* Give the name from the last setTargetName() call, which is the
   name of the target given on the command line if it's a named
   host. */
  const char *TargetName() const { return targetname; }
  /* You can set to NULL to erase a name.  The targetname is blown
     away when you setTargetSockAddr(), so make sure you do these in proper
     order
  */
  void setTargetName(const char *name);

  /* If the host is directly connected on a network, set and retrieve
     that information here.  directlyConnected() will abort if it hasn't
     been set yet.  */
  void setDirectlyConnected(bool connected);
  bool directlyConnected() const;
  int directlyConnectedOrUnset() const; /* 1-directly connected, 0-no, -1-we don't know*/

  /* If the host is NOT directly connected, you can set the next hop
     value here. It is OK to pass in a sockaddr_in or sockaddr_in6
     casted to sockaddr_storage*/
  void setNextHop(const struct sockaddr_storage *next_hop, size_t next_hop_len);
  /* Returns the next hop for sending packets to this host.  Returns true if
     next_hop was filled in.  It might be false, for example, if
     next_hop has never been set */
  bool nextHop(struct sockaddr_storage *next_hop, size_t *next_hop_len) const;

  void setMTU(int devmtu);
  int MTU(void) const;

  /* Sets the interface type to one of:
     devt_ethernet, devt_loopback, devt_p2p, devt_other
   */
  void setIfType(devtype iftype) { interface_type = iftype; }
  /* Returns -1 if it has not yet been set with setIfType() */
  devtype ifType() const { return interface_type; }
  /* Starts the timeout clock for the host running (e.g. you are
     beginning a scan).  If you do not have the current time handy,
     you can pass in NULL.  When done, call stopTimeOutClock (it will
     also automatically be stopped of timedOut() returns true) */
  void startTimeOutClock(const struct timeval *now);
  /* The complement to startTimeOutClock. */
  void stopTimeOutClock(const struct timeval *now);
  /* Is the timeout clock currently running? */
  bool timeOutClockRunning() const { return htn.toclock_running; }
  /* Returns whether the host is timedout.  If the timeoutclock is
     running, counts elapsed time for that.  Pass NULL if you don't have the
     current time handy.  You might as well also pass NULL if the
     clock is not running, as the func won't need the time. */
  bool timedOut(const struct timeval *now) const;
  /* Return time_t for the start and end time of this host */
  time_t StartTime() const { return htn.host_start; }
  time_t EndTime() const { return htn.host_end; }

  /* Takes a 6-byte MAC address */
  int setMACAddress(const u8 *addy);
  int setSrcMACAddress(const u8 *addy);
  int setNextHopMACAddress(const u8 *addy); // this should be the target's own MAC if directlyConnected()

  /* Returns a pointer to 6-byte MAC address, or NULL if none is set */
  const u8 *MACAddress() const;
  const u8 *SrcMACAddress() const;
  const u8 *NextHopMACAddress() const;

```

这段代码的主要作用是设置目标主机的设备名称和完整名称，以便在函数 `deviceName()` 和 `deviceFullName()` 中使用。设备名称包括设备类型和子网掩码，如果这些名称设置为非空，将覆盖已有的存储版本。

函数 `osscanPerformed()` 用于获取操作系统扫描结果，函数 `osscanSetFlag()` 用于设置扫描的标志，例如 `HF_SCAN_RECORD_ARP`。

结构体 `seq_info` 用于存储扫描结果，包括距离（单位：米）和距离计算方法（距离计算方法：距离计算方法可以是 `HF_SCAN_CREATE_TRACKING_Loop`、`HF_SCAN_CREATE_TRACKING_Loop_CIR` 或 `HF_SCAN_TRACKING_Loop`）。

结构体 `FingerPrintResults` 用于存储操作系统扫描结果，包括 `HF_SCAN_RESULT_FIELD_VALUES`。

结构体 `PortList` 用于存储操作系统扫描结果中的端口。

结构体 `EarlySvcResponse` 用于存储操作系统扫描结果，包括 `HF_SCAN_RESULT_FIELD_VALUES`。

结构体 `TracerouteHop` 用于存储追踪路由器输出的结果。

结构体 `struct sockaddr_storage` 用于存储 DNS 查询结果。

该代码的作用是设置目标主机设备的名称和完整名称，以便在函数 `deviceName()` 和 `deviceFullName()` 中使用，并在函数 `osscanPerformed()` 和 `osscanSetFlag()` 中进行设置。


```cpp
/* Set the device names so that they can be returned by deviceName()
   and deviceFullName().  The normal name may not include alias
   qualifier, while the full name may include it (e.g. "eth1:1").  If
   these are non-null, they will overwrite the stored version */
  void setDeviceNames(const char *name, const char *fullname);
  const char *deviceName() const;
  const char *deviceFullName() const;

  int osscanPerformed(void) const;
  void osscanSetFlag(int flag);

  struct seq_info seq;
  int distance;
  enum dist_calc_method distance_calculation_method;
  FingerPrintResults *FPR; /* FP results get by the OS scan system. */
  PortList ports;
  std::vector<EarlySvcResponse *> earlySvcResponses;

  int weird_responses; /* echo responses from other addresses, Ie a network broadcast address */
  int flags; /* HOST_UNKNOWN, HOST_UP, or HOST_DOWN. */
  struct timeout_info to;
  char *hostname; // Null if unable to resolve or unset
  char * targetname; // The name of the target host given on the command line if it is a named host

  struct probespec traceroute_probespec;
  std::list <TracerouteHop> traceroute_hops;

  /* If the address for this target came from a DNS lookup, the list of
     resultant addresses (sometimes there are more than one) that were not scanned. */
  std::list<struct sockaddr_storage> unscanned_addrs;

```

这段代码定义了一个名为`ScriptResults`的类，但并没有定义它的作用域。它包含一个名为`reason`的`state_reason_t`类型的变量，一个名为`pingprobe`的`probespec`类型的变量，以及一个名为`pingprobe_state`的整数类型的变量。

该类的作用是：在网络探测中，当接收到特定探测类型（由`probespec.type`决定）的探测响应时，记录当前的探测状态以及目标或源的IP地址。用于在扫描过程中保留当前的探测类型，以便在扫描结束后，根据需要重新运行探测。


```cpp
#ifndef NOLUA
  ScriptResults scriptResults;
#endif

  state_reason_t reason;

  /* A probe that is known to receive a response. This is used to hold the
     current timing ping probe type during scanning. */
  probespec pingprobe;
  /* The state the port or protocol entered when the response to pingprobe was
     received. */
  int pingprobe_state;

  private:
  void FreeInternal(); // Free memory allocated inside this object
 // Creates a "presentation" formatted string out of the target's IPv4/IPv6 address
  void GenerateTargetIPString();
 // Creates a "presentation" formatted string out of the source IPv4/IPv6 address.
  void GenerateSourceIPString();
  struct sockaddr_storage targetsock, sourcesock, nexthopsock;
  size_t targetsocklen, sourcesocklen, nexthopsocklen;
  int directly_connected; // -1 = unset; 0 = no; 1 = yes
  char targetipstring[INET6_ADDRSTRLEN];
  char sourceipstring[INET6_ADDRSTRLEN];
  mutable char *nameIPBuf; /* for the NameIP(void) function to return */
  u8 MACaddress[6], SrcMACaddress[6], NextHopMACaddress[6];
  bool MACaddress_set, SrcMACaddress_set, NextHopMACaddress_set;
  struct host_timeout_nfo htn;
  devtype interface_type;
  char devname[32];
  char devfullname[32];
  int mtu;
  /* 0 (OS_NOTPERF) if os detection not performed
   * 1 (OS_PERF) if os detection performed
   * 2 (OS_PERF_UNREL) if an unreliable os detection has been performed */
  int osscan_flag;
};

```

这段代码是一个 Preprocessed C 代码，它处理了 #define 定义，对其进行了解析。

具体来说，这段代码的作用是检查定义 TARGET_H 的目标（Target）是否被定义。如果目标已经被定义，那么代码不会做任何事情；否则，代码会输出一条消息，指出目标没有被定义。


```cpp
#endif /* TARGET_H */


```