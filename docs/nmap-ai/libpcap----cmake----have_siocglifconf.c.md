# `nmap\libpcap\cmake\have_siocglifconf.c`

```cpp
# 包含系统调用的头文件
#include <sys/ioctl.h>
#include <sys/socket.h>
#include <sys/sockio.h>
# 主函数
int main() {
    # 使用 ioctl 函数获取网络接口配置信息
    ioctl(0, SIOCGLIFCONF, (char *)0);
}
```