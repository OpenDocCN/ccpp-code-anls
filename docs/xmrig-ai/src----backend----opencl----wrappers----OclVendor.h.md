# `xmrig\src\backend\opencl\wrappers\OclVendor.h`

```cpp
/*
 * XMRig
 * 版权所有 2010      Jeff Garzik <jgarzik@pobox.com>
 * 版权所有 2012-2014 pooler      <pooler@litecoinpool.org>
 * 版权所有 2014      Lucas Jones <https://github.com/lucasjones>
 * 版权所有 2014-2016 Wolf9466    <https://github.com/OhGodAPet>
 * 版权所有 2016      Jay D Dee   <jayddee246@gmail.com>
 * 版权所有 2017-2018 XMR-Stak    <https://github.com/fireice-uk>, <https://github.com/psychocrypt>
 * 版权所有 2018-2019 SChernykh   <https://github.com/SChernykh>
 * 版权所有 2016-2019 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改
 *   根据 GNU 通用公共许可证的条款发布，由
 *   自由软件基金会发布的版本 3 或
 *   （根据您的选择）任何以后的版本。
 *
 *   本程序是希望它有用，
 *   但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。更多细节请参见
 *   GNU 通用公共许可证。
 *
 *   您应该已经收到了 GNU 通用公共许可证的副本
 *   与本程序一起。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_OCLVENDOR_H
#define XMRIG_OCLVENDOR_H

// 命名空间 xmrig
namespace xmrig {

// 枚举类型 OclVendor，表示 OpenCL 厂商
enum OclVendor : unsigned {
    OCL_VENDOR_UNKNOWN,  // 未知厂商
    OCL_VENDOR_AMD,      // AMD 厂商
    OCL_VENDOR_NVIDIA,   // NVIDIA 厂商
    OCL_VENDOR_INTEL,    // Intel 厂商
    OCL_VENDOR_APPLE     // Apple 厂商
};

} // 命名空间 xmrig

#endif /* XMRIG_OCLVENDOR_H */
```