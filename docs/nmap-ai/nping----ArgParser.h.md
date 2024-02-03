# `nmap\nping\ArgParser.h`

```cpp
#ifndef NPING_ARGPARSER_H
#define NPING_ARGPARSER_H

class ArgParser {

  public:
    // 构造函数
    ArgParser();
    // 析构函数
    ~ArgParser();
    // 解析命令行参数
    int parseArguments(int argc, char *argv[]);

    // 打印版本信息
    void printVersion(void);
    // 打印用法信息
    void printUsage(void);
    // 解析广告条目
    int parseAdvertEntry(char *str, struct in_addr *addr, u32 *pref);
    // 将字符串转换为 ICMP 类型
    int atoICMPType(char *opt, u8 *type);
    // 将字符串转换为 ICMP 代码
    int atoICMPCode(char *opt, u8 *code);
    // 将字符串转换为 ARP 操作码
    int atoARPOpCode(char *opt, u16 *code);
    // 将字符串转换为以太网类型
    int atoEtherType(char *opt, u16 *type);
    // 解析 ICMP 时间戳
    int parseICMPTimestamp(char *optarg, u32 *dst);

}; /* End of class ArgParser*/
#endif // NPING_ARGPARSER_H
```