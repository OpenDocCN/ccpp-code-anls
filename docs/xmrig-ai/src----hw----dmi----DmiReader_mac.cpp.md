# `xmrig\src\hw\dmi\DmiReader_mac.cpp`

```
/*
 * XMRig
 * 版权所有 (c) 2002-2006 Hugo Weber  <address@hidden>
 * 版权所有 (c) 2018-2023 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2023 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以根据 GNU 通用公共许可证的条款重新分发或修改它，
 * 该许可证由自由软件基金会发布，无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 * 本程序是希望它有用，但没有任何担保；甚至没有对特定目的的隐含担保。有关更多详细信息，请参见 GNU 通用公共许可证。
 *
 * 您应该已经收到了 GNU 通用公共许可证的副本。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */

#include "hw/dmi/DmiReader.h"
#include "hw/dmi/DmiTools.h"


#include <Carbon/Carbon.h>


namespace xmrig {


// 计算校验和
static int checksum(const uint8_t *buf, size_t len)
{
    uint8_t sum = 0;

    for (size_t a = 0; a < len; a++) {
        sum += buf[a];
    }

    return (sum == 0);
}


// 获取 DMI 表
static uint8_t *dmi_table(uint32_t base, uint32_t &len, io_service_t service)
{
    CFMutableDictionaryRef properties = nullptr;
    if (IORegistryEntryCreateCFProperties(service, &properties, kCFAllocatorDefault, kNilOptions) != kIOReturnSuccess) {
        return nullptr;
    }

    CFDataRef data;
    uint8_t *buf = nullptr;

    if (CFDictionaryGetValueIfPresent(properties, CFSTR("SMBIOS"), (const void **)&data)) {
        assert(len == CFDataGetLength(data));

        len = CFDataGetLength(data);
        buf = reinterpret_cast<uint8_t *>(malloc(len));

        CFDataGetBytes(data, CFRangeMake(0, len), buf);
    }

    CFRelease(properties);

    return buf;
}


// 解码 SMBIOS
static uint8_t *smbios_decode(uint8_t *buf, uint32_t &size, uint32_t &version, io_service_t service)
{
    # 检查条件：buf[0x05]大于0x20，或者checksum函数返回false，或者buf + 0x10与"_DMI_"不相等，或者checksum函数返回false
    if (buf[0x05] > 0x20 || !checksum(buf, buf[0x05]) || memcmp(buf + 0x10, "_DMI_", 5) != 0 || !checksum(buf + 0x10, 0x0F))  {
        # 如果条件满足，则返回空指针
        return nullptr;
    }

    # 计算version的值
    version = ((buf[0x06] << 8) + buf[0x07]) << 8;
    # 获取size的值
    size    = dmi_get<uint16_t>(buf + 0x16);

    # 返回dmi_table对象，参数为dmi_get<uint32_t>(buf + 0x18)，size和service
    return dmi_table(dmi_get<uint32_t>(buf + 0x18), size, service);
} // namespace xmrig

// 读取 DMI 数据
bool xmrig::DmiReader::read()
{
    // 创建一个 Mach 端口
    mach_port_t port;
    IOMasterPort(MACH_PORT_NULL, &port);

    // 获取匹配的 IO 服务
    io_service_t service = IOServiceGetMatchingService(port, IOServiceMatching("AppleSMBIOS"));
    // 如果获取不到服务，则返回 false
    if (service == MACH_PORT_NULL) {
        return false;
    }

    // 从 IO 注册表中获取属性数据
    CFDataRef data = reinterpret_cast<CFDataRef>(IORegistryEntryCreateCFProperty(service, CFSTR("SMBIOS-EPS"), kCFAllocatorDefault, kNilOptions));
    // 如果获取不到数据或者数据长度小于 0x1f，则返回 false
    if (!data || CFDataGetLength(data) < 0x1f) {
        return false;
    }

    // 创建一个缓冲区，并将属性数据拷贝到缓冲区中
    uint8_t buf[0x20]{};
    CFDataGetBytes(data, CFRangeMake(0, sizeof(buf) - 1), buf);
    CFRelease(data);

    // 解码缓冲区中的数据，并返回解码结果
    auto smb      = smbios_decode(buf, m_size, m_version, service);
    // 如果解码结果不为空，则调用 decode 方法解码
    const bool rc = smb ? decode(smb) : false;

    // 释放 IO 服务
    IOObjectRelease(service);

    // 返回解码结果
    return rc;
}
```