# `nmap\libpcap\missing\getopt.h`

```cpp
/*
 * 如果平台没有提供 getopt() 函数，我们提供的 getopt() 函数的头文件
 */
extern char *optarg;            /* getopt(3) 外部变量 */
extern int optind, opterr, optreset, optopt;

extern int getopt(int nargc, char * const *nargv, const char *ostr);
*/ 
```