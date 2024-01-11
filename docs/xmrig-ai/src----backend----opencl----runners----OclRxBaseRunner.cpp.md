# `xmrig\src\backend\opencl\runners\OclRxBaseRunner.cpp`

```
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
 * 本程序是自由软件：您可以重新发布或修改
 * 根据 GNU 通用公共许可证的条款，由
 * 自由软件基金会发布的版本 3 或
 * （根据您的选择）任何以后的版本。
 *
 * 本程序是希望它有用，
 * 但没有任何保证；甚至没有暗示的保证。
 * 有关更多详细信息，请参见
 * GNU 通用公共许可证。
 *
 * 您应该已经收到了 GNU 通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include "backend/opencl/runners/OclRxBaseRunner.h"
#include "backend/opencl/kernels/rx/Blake2bHashRegistersKernel.h"
#include "backend/opencl/kernels/rx/Blake2bInitialHashKernel.h"
#include "backend/opencl/kernels/rx/FillAesKernel.h"
#include "backend/opencl/kernels/rx/FindSharesKernel.h"
#include "backend/opencl/kernels/rx/HashAesKernel.h"
#include "backend/opencl/OclLaunchData.h"
#include "backend/opencl/runners/tools/OclSharedState.h"
#include "backend/opencl/wrappers/OclLib.h"
#include "base/net/stratum/Job.h"
#include "crypto/rx/Rx.h"
#include "crypto/rx/RxAlgo.h"
#include "crypto/rx/RxDataset.h"

// 创建 OclRxBaseRunner 类的构造函数，传入索引和 OclLaunchData 对象
xmrig::OclRxBaseRunner::OclRxBaseRunner(size_t index, const OclLaunchData &data) : OclBaseRunner(index, data)
{
    // 根据数据线程的工作大小进行切换
    # 如果数据为2、4、8、16，则将工作大小设置为数据中的线程工作大小
    case 2:
    case 4:
    case 8:
    case 16:
        m_worksize = data.thread.worksize();
        break;

    # 如果数据不在上述范围内，默认将工作大小设置为8
    default:
        m_worksize = 8;
    }

    # 如果设备类型为Vega_10、Vega_20或Raven，则将GCN版本设置为14
    if (data.device.type() == OclDevice::Vega_10 || data.device.type() == OclDevice::Vega_20 || data.device.type() == OclDevice::Raven) {
        m_gcn_version = 14;
    }

    # 如果设备类型为Navi_10、Navi_12、Navi_14或Navi_21，则将GCN版本设置为15
    if (data.device.type() == OclDevice::Navi_10 || data.device.type() == OclDevice::Navi_12 || data.device.type() == OclDevice::Navi_14 || data.device.type() == OclDevice::Navi_21) {
        m_gcn_version = 15;
    }

    # 将选项字符串添加算法ID、工作大小和GCN版本
    m_options += " -DALGO="             + std::to_string(RxAlgo::id(m_algorithm));
    m_options += " -DWORKERS_PER_HASH=" + std::to_string(m_worksize);
    m_options += " -DGCN_VERSION="      + std::to_string(m_gcn_version);
# 析构函数，释放对象占用的内存
xmrig::OclRxBaseRunner::~OclRxBaseRunner()
{
    # 释放对象占用的内存
    delete m_fillAes1Rx4_scratchpad;
    delete m_fillAes4Rx4_entropy;
    delete m_hashAes1Rx4;
    delete m_blake2b_initial_hash;
    delete m_blake2b_hash_registers_32;
    delete m_blake2b_hash_registers_64;
    delete m_find_shares;

    # 释放 OpenCL 资源
    OclLib::release(m_entropy);
    OclLib::release(m_hashes);
    OclLib::release(m_rounding);
    OclLib::release(m_scratchpads);
    OclLib::release(m_dataset);
}

# 运行函数，执行算法
void xmrig::OclRxBaseRunner::run(uint32_t nonce, uint32_t *hashOutput)
{
    # 定义常量 zero 为 0
    static const uint32_t zero = 0;

    # 设置初始哈希的 nonce
    m_blake2b_initial_hash->setNonce(nonce);
    m_find_shares->setNonce(nonce);

    # 将 zero 写入输出缓冲区
    enqueueWriteBuffer(m_output, CL_FALSE, sizeof(cl_uint) * 0xFF, sizeof(uint32_t), &zero);

    # 执行初始哈希和填充 AES1Rx4 临时缓冲区的操作
    m_blake2b_initial_hash->enqueue(m_queue, m_intensity);
    m_fillAes1Rx4_scratchpad->enqueue(m_queue, m_intensity);

    # 获取程序数量
    const uint32_t programCount = RxAlgo::programCount(m_algorithm);

    # 循环执行算法
    for (uint32_t i = 0; i < programCount; ++i) {
        # 填充 AES4Rx4 临时缓冲区的操作
        m_fillAes4Rx4_entropy->enqueue(m_queue, m_intensity);

        # 执行算法
        execute(i);

        # 判断是否为最后一个程序，执行相应的操作
        if (i == programCount - 1) {
            m_hashAes1Rx4->enqueue(m_queue, m_intensity);
            m_blake2b_hash_registers_32->enqueue(m_queue, m_intensity);
        }
        else {
            m_blake2b_hash_registers_64->enqueue(m_queue, m_intensity);
        }
    }

    # 执行找到共享的操作
    m_find_shares->enqueue(m_queue, m_intensity);

    # 完成算法，获取哈希输出
    finalize(hashOutput);

    # 完成队列中的所有操作
    OclLib::finish(m_queue);
}

# 设置函数，设置作业和数据块
void xmrig::OclRxBaseRunner::set(const Job &job, uint8_t *blob)
{
    # 如果数据不在主机上，并且种子不同，则更新种子和数据集
    if (!data().thread.isDatasetHost() && m_seed != job.seed()) {
        m_seed = job.seed();

        # 获取数据集并写入数据集缓冲区
        auto dataset = Rx::dataset(job, 0);
        enqueueWriteBuffer(m_dataset, CL_TRUE, 0, RxDataset::maxSize(), dataset->raw());
    }

    # 如果作业大小小于最大数据块大小，则填充零
    if (job.size() < Job::kMaxBlobSize) {
        memset(blob + job.size(), 0, Job::kMaxBlobSize - job.size());
    }

    # 将数据块写入输入缓冲区
    enqueueWriteBuffer(m_input, CL_TRUE, 0, Job::kMaxBlobSize, blob);
}
    # 设置初始哈希值的大小为作业的大小
    m_blake2b_initial_hash->setBlobSize(job.size());
    # 设置找到的份额的目标值为作业的目标值
    m_find_shares->setTarget(job.target());
# 返回缓冲区大小，包括基类的缓冲区大小和额外的内存对齐
size_t xmrig::OclRxBaseRunner::bufferSize() const
{
    return OclBaseRunner::bufferSize() +  # 调用基类的 bufferSize 方法获取基类的缓冲区大小
           align((m_algorithm.l3() + 64) * m_intensity) +  # 对 (m_algorithm.l3() + 64) * m_intensity 进行内存对齐
           align(64 * m_intensity) +  # 对 64 * m_intensity 进行内存对齐
           align((128 + 2560) * m_intensity) +  # 对 (128 + 2560) * m_intensity 进行内存对齐
           align(sizeof(uint32_t) * m_intensity);  # 对 sizeof(uint32_t) * m_intensity 进行内存对齐
}

# 构建方法，初始化各种内核对象
void xmrig::OclRxBaseRunner::build()
{
    OclBaseRunner::build();  # 调用基类的 build 方法

    const uint32_t rx_version = RxAlgo::version(m_algorithm);  # 获取算法版本号

    m_fillAes1Rx4_scratchpad = new FillAesKernel(m_program, "fillAes1Rx4_scratchpad");  # 创建 FillAesKernel 对象
    m_fillAes1Rx4_scratchpad->setArgs(m_hashes, m_scratchpads, m_intensity, rx_version);  # 设置参数

    m_fillAes4Rx4_entropy = new FillAesKernel(m_program, "fillAes4Rx4_entropy");  # 创建 FillAesKernel 对象
    m_fillAes4Rx4_entropy->setArgs(m_hashes, m_entropy, m_intensity, rx_version);  # 设置参数

    m_hashAes1Rx4 = new HashAesKernel(m_program);  # 创建 HashAesKernel 对象

    m_blake2b_initial_hash = new Blake2bInitialHashKernel(m_program);  # 创建 Blake2bInitialHashKernel 对象
    m_blake2b_initial_hash->setArgs(m_hashes, m_input);  # 设置参数

    m_blake2b_hash_registers_32 = new Blake2bHashRegistersKernel(m_program, "blake2b_hash_registers_32");  # 创建 Blake2bHashRegistersKernel 对象
    m_blake2b_hash_registers_64 = new Blake2bHashRegistersKernel(m_program, "blake2b_hash_registers_64");  # 创建 Blake2bHashRegistersKernel 对象

    m_find_shares = new FindSharesKernel(m_program);  # 创建 FindSharesKernel 对象
    m_find_shares->setArgs(m_hashes, m_output);  # 设置参数
}

# 初始化方法，初始化各种缓冲区和数据集
void xmrig::OclRxBaseRunner::init()
{
    OclBaseRunner::init();  # 调用基类的 init 方法

    m_scratchpads = createSubBuffer(CL_MEM_READ_WRITE | CL_MEM_HOST_NO_ACCESS, (m_algorithm.l3() + 64) * m_intensity);  # 创建缓冲区
    m_hashes      = createSubBuffer(CL_MEM_READ_WRITE | CL_MEM_HOST_NO_ACCESS, 64 * m_intensity);  # 创建缓冲区
    m_entropy     = createSubBuffer(CL_MEM_READ_WRITE | CL_MEM_HOST_NO_ACCESS, (128 + 2560) * m_intensity);  # 创建缓冲区
    m_rounding    = createSubBuffer(CL_MEM_READ_WRITE | CL_MEM_HOST_NO_ACCESS, sizeof(uint32_t) * m_intensity);  # 创建缓冲区
    m_dataset     = OclSharedState::get(data().device.index()).dataset();  # 获取数据集
}
```