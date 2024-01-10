# `nmap\libdnet-stripped\src\addr-util.c`

```
/*
 * addr-util.c
 *
 * 版权所有 2002 Dug Song <dugsong@monkey.org>
 *
 * $Id: addr-util.c 539 2005-01-23 07:36:54Z dugsong $
 */

#ifdef _WIN32
#include "dnet_winconfig.h"
#else
#include "config.h"
#endif

#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#include "dnet.h"

// 将八进制转换为十进制的字符串数组
static const char *octet2dec[] = {
    "0", "1", "2", "3", "4", "5", "6", "7", "8", "9", "10", "11", "12",
    "13", "14", "15", "16", "17", "18", "19", "20", "21", "22", "23",
    "24", "25", "26", "27", "28", "29", "30", "31", "32", "33", "34",
    "35", "36", "37", "38", "39", "40", "41", "42", "43", "44", "45",
    "46", "47", "48", "49", "50", "51", "52", "53", "54", "55", "56",
    "57", "58", "59", "60", "61", "62", "63", "64", "65", "66", "67",
    "68", "69", "70", "71", "72", "73", "74", "75", "76", "77", "78",
    "79", "80", "81", "82", "83", "84", "85", "86", "87", "88", "89",
    "90", "91", "92", "93", "94", "95", "96", "97", "98", "99", "100",
    "101", "102", "103", "104", "105", "106", "107", "108", "109",
    "110", "111", "112", "113", "114", "115", "116", "117", "118",
    "119", "120", "121", "122", "123", "124", "125", "126", "127",
    "128", "129", "130", "131", "132", "133", "134", "135", "136",
    "137", "138", "139", "140", "141", "142", "143", "144", "145",
    "146", "147", "148", "149", "150", "151", "152", "153", "154",
    "155", "156", "157", "158", "159", "160", "161", "162", "163",
    "164", "165", "166", "167", "168", "169", "170", "171", "172",
    "173", "174", "175", "176", "177", "178", "179", "180", "181",
    "182", "183", "184", "185", "186", "187", "188", "189", "190",
    "191", "192", "193", "194", "195", "196", "197", "198", "199",
    "200", "201", "202", "203", "204", "205", "206", "207", "208",
    "209", "210", "211", "212", "213", "214", "215", "216", "217",
    "218", "219", "220", "221", "222", "223", "224", "225", "226",
    "227", "228", "229", "230", "231", "232", "233", "234", "235",
    # 这是一系列字符串，可能是用于某种数据处理或者配置的值
    # 没有上下文，无法确定具体用途
# 定义一个静态的十六进制字符串数组，用于将十进制数转换为十六进制字符串
static const char *octet2hex[] = {
    # 从00到ff的十六进制字符串表示，共256个元素
    "00", "01", "02", "03", "04", "05", "06", "07", "08", "09", "0a",
    "0b", "0c", "0d", "0e", "0f", "10", "11", "12", "13", "14", "15",
    # ... 省略部分元素 ...
    "fe", "ff"
};

# 将以太网地址转换为字符串表示
char *
eth_ntop(const eth_addr_t *eth, char *dst, size_t len)
{
    const char *x;
    char *p = dst;
    int i;
    
    # 如果目标字符串长度不足以存放转换后的字符串，返回空指针
    if (len < 18)
        return (NULL);
    
    # 遍历以太网地址的每个字节
    for (i = 0; i < ETH_ADDR_LEN; i++) {
        # 将每个字节转换为十六进制字符串表示，存入目标字符串中
        for (x = octet2hex[eth->data[i]]; (*p = *x) != '\0'; x++, p++)
            ;
        # 在每个字节的十六进制字符串表示后面添加冒号
        *p++ = ':';
    }
    # 将字符串 p 的最后一个字符设置为字符串结束符'\0'
    p[-1] = '\0';
    
    # 返回 dst 变量的值
    return (dst);
}

char *
eth_ntoa(const eth_addr_t *eth)
{
    struct addr a; // 定义一个地址结构体a
    
    addr_pack(&a, ADDR_TYPE_ETH, ETH_ADDR_BITS, eth->data, ETH_ADDR_LEN); // 调用addr_pack函数，将以太网地址打包到地址结构体a中
    return (addr_ntoa(&a)); // 调用addr_ntoa函数，将地址结构体a转换成字符串并返回
}

int
eth_pton(const char *p, eth_addr_t *eth)
{
    char *ep; // 定义一个字符指针ep
    long l; // 定义一个长整型变量l
    int i; // 定义一个整型变量i
    
    for (i = 0; i < ETH_ADDR_LEN; i++) { // 循环遍历以太网地址的每个字节
        l = strtol(p, &ep, 16); // 将字符串p转换成长整型数，以16进制表示
        if (ep == p || l < 0 || l > 0xff || (i < ETH_ADDR_LEN - 1 && *ep != ':')) // 判断转换是否成功以及转换后的值是否在合法范围内
            break; // 如果不合法则跳出循环
        eth->data[i] = (u_char)l; // 将转换后的值存入以太网地址结构体中
        p = ep + 1; // 更新字符串指针p的位置
    }
    return ((i == ETH_ADDR_LEN && *ep == '\0') ? 0 : -1); // 返回转换结果
}

char *
ip_ntop(const ip_addr_t *ip, char *dst, size_t len)
{
    const char *d; // 定义一个常量字符指针d
    char *p = dst; // 定义一个字符指针p，指向目标字符串
    u_char *data = (u_char *)ip; // 定义一个无符号字符指针data，指向IP地址结构体
    int i; // 定义一个整型变量i
    
    if (len < 16) // 判断目标字符串长度是否足够
        return (NULL); // 如果不够则返回空指针
    
    for (i = 0; i < IP_ADDR_LEN; i++) { // 循环遍历IP地址的每个字节
        for (d = octet2dec[data[i]]; (*p = *d) != '\0'; d++, p++) // 将每个字节转换成十进制表示并存入目标字符串
            ;
        *p++ = '.'; // 在目标字符串中加入点号
    }
    p[-1] = '\0'; // 将目标字符串最后一个字符改为结束符
    return (dst); // 返回转换后的IP地址字符串
}

char *
ip_ntoa(const ip_addr_t *ip)
{
    struct addr a; // 定义一个地址结构体a
    
    addr_pack(&a, ADDR_TYPE_IP, IP_ADDR_BITS, ip, IP_ADDR_LEN); // 调用addr_pack函数，将IP地址打包到地址结构体a中
    return (addr_ntoa(&a)); // 调用addr_ntoa函数，将地址结构体a转换成字符串并返回
}

int
ip_pton(const char *p, ip_addr_t *ip)
{
    u_char *data = (u_char *)ip; // 定义一个无符号字符指针data，指向IP地址结构体
    char *ep; // 定义一个字符指针ep
    long l; // 定义一个长整型变量l
    int i; // 定义一个整型变量i

    for (i = 0; i < IP_ADDR_LEN; i++) { // 循环遍历IP地址的每个字节
        l = strtol(p, &ep, 10); // 将字符串p转换成长整型数，以10进制表示
        if (ep == p || l < 0 || l > 0xff || (i < IP_ADDR_LEN - 1 && *ep != '.')) // 判断转换是否成功以及转换后的值是否在合法范围内
            break; // 如果不合法则跳出循环
        data[i] = (u_char)l; // 将转换后的值存入IP地址结构体中
        p = ep + 1; // 更新字符串指针p的位置
    }
    return ((i == IP_ADDR_LEN && *ep == '\0') ? 0 : -1); // 返回转换结果
}

char *
ip6_ntop(const ip6_addr_t *ip6, char *dst, size_t len)
{
    uint16_t data[IP6_ADDR_LEN / 2]; // 定义一个16位无符号整型数组data
    struct { int base, len; } best, cur; // 定义两个结构体变量best和cur
    char *p = dst; // 定义一个字符指针p，指向目标字符串
    int i; // 定义一个整型变量i

    cur.len = best.len = 0; // 初始化两个结构体变量的长度为0
    
    if (len < 46) // 判断目标字符串长度是否足够
        return (NULL); // 如果不够则返回空指针
    
    /* Copy into 16-bit array. */
    for (i = 0; i < IP6_ADDR_LEN / 2; i++) { // 循环遍历IPV6地址的每个16位块
        data[i] = ip6->data[2 * i] << 8; // 将IPV6地址的每个字节合并成16位块
        data[i] |= ip6->data[2 * i + 1];
    }
    
    best.base = cur.base = -1; // 初始化两个结构体变量的基址为-1
    /*
     * 从Vixie的inet_pton6()中借鉴的算法
     */
    for (i = 0; i < IP6_ADDR_LEN; i += 2) {
        // 如果数据的一半为0
        if (data[i / 2] == 0) {
            // 如果当前段的起始位置为-1
            if (cur.base == -1) {
                cur.base = i;
                cur.len = 0;
            } else
                cur.len += 2;
        } else {
            // 如果当前段的起始位置不为-1
            if (cur.base != -1) {
                // 如果最佳段的起始位置为-1或者当前段长度大于最佳段长度
                if (best.base == -1 || cur.len > best.len)
                    best = cur;
                cur.base = -1;
            }
        }
    }
    // 如果当前段的起始位置不为-1并且（最佳段的起始位置为-1或者当前段长度大于最佳段长度）
    if (cur.base != -1 && (best.base == -1 || cur.len > best.len))
        best = cur;
    // 如果最佳段的起始位置不为-1并且最佳段长度小于2
    if (best.base != -1 && best.len < 2)
        best.base = -1;
    // 如果最佳段的起始位置为0
    if (best.base == 0)
        *p++ = ':';

    for (i = 0; i < IP6_ADDR_LEN; i += 2) {
        // 如果i等于最佳段的起始位置
        if (i == best.base) {
            *p++ = ':';
            i += best.len;
        } else if (i == 12 && best.base == 0 &&
            (best.len == 10 || (best.len == 8 &&
            data[5] == 0xffff))) {
            // 如果ip_ntop()函数返回NULL
            if (ip_ntop((ip_addr_t *)&data[6], p,
                len - (p - dst)) == NULL)
                return (NULL);
            return (dst);
        } else p += sprintf(p, "%x:", data[i / 2]);
    }
    // 如果最佳段的起始位置加2加最佳段长度等于IP6_ADDR_LEN
    if (best.base + 2 + best.len == IP6_ADDR_LEN) {
        *p = '\0';
    } else
        p[-1] = '\0';

    return (dst);
# 将 IPv6 地址转换为字符串表示形式
char *
ip6_ntoa(const ip6_addr_t *ip6)
{
    struct addr a;
    # 将 IPv6 地址打包成地址结构
    addr_pack(&a, ADDR_TYPE_IP6, IP6_ADDR_BITS, ip6->data, IP6_ADDR_LEN);
    # 返回地址结构的字符串表示形式
    return (addr_ntoa(&a));
}

# 将字符串表示的 IPv6 地址转换为二进制形式
int
ip6_pton(const char *p, ip6_addr_t *ip6)
{
    uint16_t data[8], *u = (uint16_t *)ip6->data;
    int i, j, n, z = -1;
    char *ep;
    long l;
    
    # 如果字符串以冒号开头，则跳过
    if (*p == ':')
        p++;
    
    # 循环处理字符串中的每个部分
    for (n = 0; n < 8; n++) {
        # 将字符串转换为长整型数值
        l = strtol(p, &ep, 16);
        
        # 如果无法转换或者遇到特殊情况，则返回错误
        if (ep == p) {
            if (ep[0] == ':' && z == -1) {
                z = n;
                p++;
            } else if (ep[0] == '\0') {
                break;
            } else {
                return (-1);
            }
        } else if (ep[0] == '.' && n <= 6) {
            # 如果遇到 IPv4 形式的地址，则调用 ip_pton 函数进行处理
            if (ip_pton(p, (ip_addr_t *)(data + n)) < 0)
                return (-1);
            n += 2;
            ep = ""; /* XXX */
            break;
        } else if (l >= 0 && l <= 0xffff) {
            # 将数值转换为网络字节序，并存入数组中
            data[n] = htons((uint16_t)l);

            if (ep[0] == '\0') {
                n++;
                break;
            } else if (ep[0] != ':' || ep[1] == '\0')
                return (-1);

            p = ep + 1;
        } else
            return (-1);
    }
    # 检查是否处理完所有部分，并且没有多余的字符
    if (n == 0 || *ep != '\0' || (z == -1 && n != 8))
        return (-1);
    
    # 将处理好的数据存入 IPv6 地址结构中
    for (i = 0; i < z; i++) {
        u[i] = data[i];
    }
    while (i < 8 - (n - z - 1)) {
        u[i++] = 0;
    }
    for (j = z + 1; i < 8; i++, j++) {
        u[i] = data[j];
    }
    return (0);
}
```