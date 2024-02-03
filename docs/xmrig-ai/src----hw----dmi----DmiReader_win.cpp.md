# `xmrig\src\hw\dmi\DmiReader_win.cpp`

```cpp
/*
 * XMRig
 * 版权所有 (c) 2002-2006 Hugo Weber  <address@hidden>
 * 版权所有 (c) 2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以自由地重新分发和/或修改
 *   根据GNU通用公共许可证的条款，发布的版本为3或
 *   （根据您的选择）以后的版本。
 *
 *   本程序是基于希望它有用的目的分发的，
 *   但没有任何担保；甚至没有适销性或特定用途的隐含担保。
 *   有关更多详细信息，请参阅GNU通用公共许可证。
 *
 *   如果没有收到GNU通用公共许可证的副本
 *   请参阅<http://www.gnu.org/licenses/>。
 */


#include "hw/dmi/DmiReader.h"
#include "hw/dmi/DmiTools.h"


#include <windows.h>


namespace xmrig {


/*
 * 使用GetSystemFirmwareTable API获取SMBIOS表所需的结构体。
 */
struct RawSMBIOSData {
    uint8_t    Used20CallingMethod;
    uint8_t    SMBIOSMajorVersion;
    uint8_t    SMBIOSMinorVersion;
    uint8_t    DmiRevision;
    uint32_t Length;
    uint8_t    SMBIOSTableData[];
};


} // namespace xmrig


bool xmrig::DmiReader::read()
{
    constexpr uint32_t RSMB = 0x52534D42;

    // 获取SMBIOS表的大小
    const uint32_t size = GetSystemFirmwareTable(RSMB, 0, nullptr, 0);
    // 分配内存来存储SMBIOS表数据
    auto smb            = reinterpret_cast<RawSMBIOSData *>(HeapAlloc(GetProcessHeap(), HEAP_ZERO_MEMORY, size));

    // 如果内存分配失败，则返回false
    if (!smb) {
        return false;
    }

    // 获取SMBIOS表数据
    if (GetSystemFirmwareTable(RSMB, 0, smb, size) != size) {
        // 释放内存
        HeapFree(GetProcessHeap(), 0, smb);

        return false;
    }

    // 计算SMBIOS版本号
    m_version = (smb->SMBIOSMajorVersion << 16) + (smb->SMBIOSMinorVersion << 8) + smb->DmiRevision;
    // 记录SMBIOS表的长度
    m_size    = smb->Length;

    // 解码SMBIOS表数据，并在完成后释放内存
    return decode(smb->SMBIOSTableData, [smb]() {
        HeapFree(GetProcessHeap(), 0, smb);
    });
}
```