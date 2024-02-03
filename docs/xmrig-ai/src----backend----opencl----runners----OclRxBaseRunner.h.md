# `xmrig\src\backend\opencl\runners\OclRxBaseRunner.h`

```cpp
/*
 * XMRig
 * 版权所有 2010      Jeff Garzik <jgarzik@pobox.com>
 * 版权所有 2012-2014 pooler      <pooler@litecoinpool.org>
 * 版权所有 2014      Lucas Jones <https://github.com/lucasjones>
 * 版权所有 2014-2016 Wolf9466    <https://github.com/OhGodAPet>
 * 版权所有 2016      Jay D Dee   <jayddee246@gmail.com>
 * 版权所有 2017-2018 XMR-Stak    <https://github.com/fireice-uk>, <https://github.com/psychocrypt>
 * 版权所有 2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有 2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布和/或修改
 * 根据 GNU 通用公共许可证的条款发布，由
 * 自由软件基金会发布的版本 3 或
 * （在您的选择下）任何以后的版本。
 *
 * 本程序是希望它有用
 * 但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的保证。请参阅
 * GNU 通用公共许可证以获取更多详细信息。
 *
 * 您应该已经收到了 GNU 通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_OCLRXBASERUNNER_H
#define XMRIG_OCLRXBASERUNNER_H


#include "backend/opencl/runners/OclBaseRunner.h"
#include "base/tools/Buffer.h"


namespace xmrig {


class Blake2bHashRegistersKernel;
class Blake2bInitialHashKernel;
class FillAesKernel;
class FindSharesKernel;
class HashAesKernel;


class OclRxBaseRunner : public OclBaseRunner
{
public:
    XMRIG_DISABLE_COPY_MOVE_DEFAULT(OclRxBaseRunner)

    OclRxBaseRunner(size_t index, const OclLaunchData &data);
    ~OclRxBaseRunner() override;

protected:
    size_t bufferSize() const override;
    void build() override;
    void init() override;
    void run(uint32_t nonce, uint32_t *hashOutput) override;
    void set(const Job &job, uint8_t *blob) override;

protected:
    # 定义一个纯虚函数，子类需要实现该函数
    virtual void execute(uint32_t iteration) = 0;
    
    # 定义指向 Blake2bHashRegistersKernel 对象的指针，初始化为 nullptr
    Blake2bHashRegistersKernel *m_blake2b_hash_registers_32 = nullptr;
    Blake2bHashRegistersKernel *m_blake2b_hash_registers_64 = nullptr;
    Blake2bInitialHashKernel *m_blake2b_initial_hash        = nullptr;
    
    # 定义一个 Buffer 对象 m_seed
    
    # 定义 OpenCL 内存对象，分别用于存储 dataset、entropy、hashes、rounding、scratchpads
    cl_mem m_dataset                                        = nullptr;
    cl_mem m_entropy                                        = nullptr;
    cl_mem m_hashes                                         = nullptr;
    cl_mem m_rounding                                       = nullptr;
    cl_mem m_scratchpads                                    = nullptr;
    
    # 定义指向 FillAesKernel 对象的指针，分别用于填充 scratchpad 和 entropy
    FillAesKernel *m_fillAes1Rx4_scratchpad                 = nullptr;
    FillAesKernel *m_fillAes4Rx4_entropy                    = nullptr;
    
    # 定义指向 FindSharesKernel 对象的指针
    FindSharesKernel *m_find_shares                         = nullptr;
    
    # 定义指向 HashAesKernel 对象的指针，用于哈希操作
    HashAesKernel *m_hashAes1Rx4                            = nullptr;
    
    # 定义两个 uint32_t 类型的变量，分别用于存储 gcn_version 和 worksize 的值
    uint32_t m_gcn_version                                  = 12;
    uint32_t m_worksize                                     = 8;
}; 
// 结束了 xmrig 命名空间的定义

} /* namespace xmrig */
// 回到 xmrig 命名空间

#endif // XMRIG_OCLRXBASERUNNER_H
// 结束了 XMRIG_OCLRXBASERUNNER_H 的条件编译
```