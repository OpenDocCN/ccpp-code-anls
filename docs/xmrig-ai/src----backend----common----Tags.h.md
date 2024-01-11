# `xmrig\src\backend\common\Tags.h`

```
/*
 * XMRig
 * 版权所有 2010      Jeff Garzik <jgarzik@pobox.com>
 * 版权所有 2012-2014 pooler      <pooler@litecoinpool.org>
 * 版权所有 2014      Lucas Jones <https://github.com/lucasjones>
 * 版权所有 2014-2016 Wolf9466    <https://github.com/OhGodAPet>
 * 版权所有 2016      Jay D Dee   <jayddee246@gmail.com>
 * 版权所有 2017-2018 XMR-Stak    <https://github.com/fireice-uk>, <https://github.com/psychocrypt>
 * 版权所有 2018      Lee Clagett <https://github.com/vtnerd>
 * 版权所有 2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有 2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改
 *   根据 GNU 通用公共许可证的条款发布，由
 *   自由软件基金会发布的版本 3 或
 *   （根据您的选择）任何以后的版本。
 *
 *   本程序是希望它有用：
 *   但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。详细信息请参见
 *   GNU 通用公共许可证。
 *
 *   您应该已经收到 GNU 通用公共许可证的副本
 *   与本程序一起。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_BACKEND_TAGS_H
#define XMRIG_BACKEND_TAGS_H


#include <cstdint>


namespace xmrig {


const char *backend_tag(uint32_t backend);  // 返回指定后端的标签
const char *cpu_tag();  // 返回 CPU 的标签


#ifdef XMRIG_FEATURE_OPENCL
const char *ocl_tag();  // 返回 OpenCL 的标签
#endif


#ifdef XMRIG_FEATURE_CUDA
const char *cuda_tag();  // 返回 CUDA 的标签
#endif


} // namespace xmrig


#endif /* XMRIG_BACKEND_TAGS_H */
```