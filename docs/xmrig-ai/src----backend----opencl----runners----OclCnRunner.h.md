# `xmrig\src\backend\opencl\runners\OclCnRunner.h`

```
/* XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改它
 * 根据由自由软件基金会发布的GNU通用公共许可证的条款，
 * 无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 * 本程序是希望它有用的分发，
 * 但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的保证。有关更多详细信息，请参见
 * GNU通用公共许可证。
 *
 * 您应该已经收到了GNU通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_OCLCNRUNNER_H
#define XMRIG_OCLCNRUNNER_H


#include "backend/opencl/runners/OclBaseRunner.h"


namespace xmrig {


class Cn0Kernel;
class Cn1Kernel;
class Cn2Kernel;
class CnBranchKernel;


class OclCnRunner : public OclBaseRunner
{
public:
    XMRIG_DISABLE_COPY_MOVE_DEFAULT(OclCnRunner)

    OclCnRunner(size_t index, const OclLaunchData &data);
    ~OclCnRunner() override;

protected:
    size_t bufferSize() const override;
    void run(uint32_t nonce, uint32_t *hashOutput) override;
    void set(const Job &job, uint8_t *blob) override;
    void build() override;
    void init() override;

private:
    enum Branches : size_t {
        BRANCH_BLAKE_256,
        BRANCH_GROESTL_256,
        BRANCH_JH_256,
        BRANCH_SKEIN_512,
        BRANCH_MAX
    };


    cl_mem m_scratchpads    = nullptr;  // 用于存储OpenCL内存对象的变量，用于存储临时数据
    cl_mem m_states         = nullptr;  // 用于存储OpenCL内存对象的变量，用于存储状态数据
    cl_program m_cnr        = nullptr;  // 用于存储OpenCL程序对象的变量
    Cn0Kernel *m_cn0        = nullptr;  // 用于存储Cn0Kernel对象的指针
    Cn1Kernel *m_cn1        = nullptr;  // 用于存储Cn1Kernel对象的指针
    Cn2Kernel *m_cn2        = nullptr;  // 用于存储Cn2Kernel对象的指针
    uint64_t m_height       = 0;        // 用于存储高度的变量

    std::vector<cl_mem> m_branches                = { nullptr, nullptr, nullptr, nullptr };  // 用于存储OpenCL内存对象的向量，用于存储分支数据
    // 创建一个包含四个指针的向量，初始值都为nullptr
    std::vector<CnBranchKernel *> m_branchKernels = { nullptr, nullptr, nullptr, nullptr };
}; 
// 结束了 xmrig 命名空间的定义

} /* namespace xmrig */
// 声明代码结束了 xmrig 命名空间的定义

#endif // XMRIG_OCLCNRUNNER_H
// 结束了对 XMRIG_OCLCNRUNNER_H 的条件编译声明
```