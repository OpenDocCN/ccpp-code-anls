# `xmrig\src\backend\opencl\wrappers\OclKernel.h`

```
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
 * 本程序是自由软件：您可以重新发布和/或修改
 * 根据 GNU 通用公共许可证的条款发布
 * 由自由软件基金会发布，无论许可证的版本是 3 还是
 * （在您的选择下）任何以后的版本。
 *
 * 本程序是希望它有用
 * 但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的保证。更多细节请参见
 * GNU 通用公共许可证。
 *
 * 您应该已经收到了 GNU 通用公共许可证的副本
 * 与本程序一起。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_OCLKERNEL_H
#define XMRIG_OCLKERNEL_H


#include "base/tools/Object.h"
#include "base/tools/String.h"


using cl_command_queue  = struct _cl_command_queue *;
using cl_kernel         = struct _cl_kernel *;
using cl_mem            = struct _cl_mem *;
using cl_program        = struct _cl_program *;


namespace xmrig {


class OclKernel
{
public:
    XMRIG_DISABLE_COPY_MOVE_DEFAULT(OclKernel)

    // 构造函数，接受 OpenCL 程序和内核名称作为参数
    OclKernel(cl_program program, const char *name);
    // 析构函数
    virtual ~OclKernel();

    // 检查内核是否有效
    inline bool isValid() const         { return m_kernel != nullptr; }
    // 返回内核对象
    inline cl_kernel kernel() const     { return m_kernel; }
    // 返回内核名称
    inline const String &name() const   { return m_name; }
    # 将一个 NDRange 内核添加到命令队列中以执行
    void enqueueNDRange(cl_command_queue queue, uint32_t work_dim, const size_t *global_work_offset, const size_t *global_work_size, const size_t *local_work_size);
    
    # 为内核函数设置参数
    void setArg(uint32_t index, size_t size, const void *value);
# 声明私有成员变量，用于存储 OpenCL 内核对象，初始化为 nullptr
cl_kernel m_kernel = nullptr;
# 声明私有成员变量，用于存储内核名称，类型为 String
const String m_name;
# 结束命名空间声明
} // namespace xmrig
# 结束头文件声明
#endif /* XMRIG_OCLKERNEL_H */
```