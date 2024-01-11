# `xmrig\src\backend\opencl\runners\OclKawPowRunner.cpp`

```
/*
 * XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改它
 * 根据由自由软件基金会发布的GNU通用公共许可证的条款
 * 其中包括许可证的第3版或
 * （根据您的选择）任何以后的版本。
 *
 * 本程序是希望它有用的
 * 但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的保证。请参阅
 * GNU通用公共许可证以获取更多详细信息。
 *
 * 您应该已经收到了GNU通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include <stdexcept>


#include "backend/opencl/runners/OclKawPowRunner.h"  // 引入OclKawPowRunner头文件
#include "backend/common/Tags.h"  // 引入Tags头文件
#include "3rdparty/libethash/ethash_internal.h"  // 引入ethash_internal头文件
#include "backend/opencl/kernels/kawpow/KawPow_CalculateDAGKernel.h"  // 引入KawPow_CalculateDAGKernel头文件
#include "backend/opencl/OclLaunchData.h"  // 引入OclLaunchData头文件
#include "backend/opencl/runners/tools/OclKawPow.h"  // 引入OclKawPow头文件
#include "backend/opencl/wrappers/OclError.h"  // 引入OclError头文件
#include "backend/opencl/wrappers/OclLib.h"  // 引入OclLib头文件
#include "base/io/log/Log.h"  // 引入Log头文件
#include "base/io/log/Log.h"  // 引入Log头文件
#include "base/io/log/Tags.h"  // 引入Tags头文件
#include "base/net/stratum/Job.h"  // 引入Job头文件
#include "base/tools/Chrono.h"  // 引入Chrono头文件
#include "crypto/common/VirtualMemory.h"  // 引入VirtualMemory头文件
#include "crypto/kawpow/KPHash.h"  // 引入KPHash头文件


namespace xmrig {

// 定义常量BLOB_SIZE为40
constexpr size_t BLOB_SIZE = 40;

// 构造函数，初始化OclKawPowRunner对象
OclKawPowRunner::OclKawPowRunner(size_t index, const OclLaunchData &data) : OclBaseRunner(index, data)
{
    // 根据工作大小设置工作组大小
    switch (data.thread.worksize())
    {
    case 64:
    case 128:
    case 256:
    case 512:
        m_workGroupSize = data.thread.worksize();
        break;
    }

    // 如果设备的供应商ID为OCL_VENDOR_NVIDIA，则设置选项和DAG工作组大小
    if (data.device.vendorId() == OclVendor::OCL_VENDOR_NVIDIA) {
        m_options += " -DPLATFORM=OPENCL_PLATFORM_NVIDIA";
        m_dagWorkGroupSize = 32;
    }
}
# OclKawPowRunner 类的析构函数，用于释放资源
OclKawPowRunner::~OclKawPowRunner()
{
    # 释放光线缓存资源
    OclLib::release(m_lightCache);
    # 释放有向无环图资源
    OclLib::release(m_dag);

    # 删除计算有向无环图的内核
    delete m_calculateDagKernel;

    # 释放控制队列资源
    OclLib::release(m_controlQueue);
    # 释放停止标志资源
    OclLib::release(m_stop);

    # 清除 OclKawPow 类的状态
    OclKawPow::clear();
}

# OclKawPowRunner 类的运行函数，用于执行计算
void OclKawPowRunner::run(uint32_t nonce, uint32_t *hashOutput)
{
    # 设置本地工作组大小
    const size_t local_work_size = m_workGroupSize;
    # 设置全局工作偏移量
    const size_t global_work_offset = nonce;
    # 设置全局工作大小
    const size_t global_work_size = m_intensity - (m_intensity % m_workGroupSize);

    # 将 m_blob 写入输入缓冲区
    enqueueWriteBuffer(m_input, CL_FALSE, 0, BLOB_SIZE, m_blob);

    # 将 zero 数组写入输出缓冲区和停止缓冲区
    const uint32_t zero[2] = {};
    enqueueWriteBuffer(m_output, CL_FALSE, 0, sizeof(uint32_t), zero);
    enqueueWriteBuffer(m_stop, CL_FALSE, 0, sizeof(uint32_t) * 2, zero);

    # 重置跳过的哈希数量
    m_skippedHashes = 0;

    # 调用 OpenCL 的 NDRangeKernel 函数执行搜索内核
    const cl_int ret = OclLib::enqueueNDRangeKernel(m_queue, m_searchKernel, 1, &global_work_offset, &global_work_size, &local_work_size, 0, nullptr, nullptr);
    if (ret != CL_SUCCESS) {
        # 如果执行出错，记录错误信息并抛出异常
        LOG_ERR("%s" RED(" error ") RED_BOLD("%s") RED(" when calling ") RED_BOLD("clEnqueueNDRangeKernel") RED(" for kernel ") RED_BOLD("progpow_search"),
            ocl_tag(), OclError::toString(ret));

        throw std::runtime_error(OclError::toString(ret));
    }

    # 从停止缓冲区读取 stop 数组
    uint32_t stop[2] = {};
    enqueueReadBuffer(m_stop, CL_FALSE, 0, sizeof(stop), stop);

    # 从输出缓冲区读取 output 数组
    uint32_t output[16] = {};
    enqueueReadBuffer(m_output, CL_TRUE, 0, sizeof(output), output);

    # 计算跳过的哈希数量
    m_skippedHashes = stop[1] * m_workGroupSize;

    # 如果输出的第一个元素大于 15，则将其设置为 15
    if (output[0] > 15) {
        output[0] = 15;
    }

    # 将哈希输出的第一个元素设置为 output[0]，并复制剩余元素
    hashOutput[0xFF] = output[0];
    memcpy(hashOutput, output + 1, output[0] * sizeof(uint32_t));
}

# 设置 OclKawPowRunner 类的状态和参数
void OclKawPowRunner::set(const Job &job, uint8_t *blob)
{
    # 将作业的高度转换为 uint32_t 类型，并获取搜索内核
    m_blockHeight = static_cast<uint32_t>(job.height());
    m_searchKernel = OclKawPow::get(*this, m_blockHeight, m_workGroupSize);

    # 计算当前高度对应的纪元
    const uint32_t epoch = m_blockHeight / KPHash::EPOCH_LENGTH;

    # 计算数据集的大小
    const uint64_t dag_size = KPCache::dag_size(epoch);
}
    # 如果 DAG 大小大于当前 DAG 容量
    if (dag_size > m_dagCapacity) {
        # 释放当前 DAG 对象
        OclLib::release(m_dag);

        # 将 DAG 容量设置为大于等于 DAG 大小的最小的 16MB 的倍数
        m_dagCapacity = VirtualMemory::align(dag_size, 16 * 1024 * 1024);
        # 创建新的 DAG 对象
        m_dag = OclLib::createBuffer(m_ctx, CL_MEM_READ_WRITE, m_dagCapacity);
    }

    # 如果当前 epoch 不等于传入的 epoch
    if (epoch != m_epoch) {
        # 更新当前 epoch
        m_epoch = epoch;

        # 使用互斥锁保护共享资源
        {
            std::lock_guard<std::mutex> lock(KPCache::s_cacheMutex);

            # 初始化缓存
            KPCache::s_cache.init(epoch);

            # 如果缓存大小大于当前轻缓存容量
            if (KPCache::s_cache.size() > m_lightCacheCapacity) {
                # 释放当前轻缓存对象
                OclLib::release(m_lightCache);

                # 将轻缓存容量设置为大于等于缓存大小的最小的倍数
                m_lightCacheCapacity = VirtualMemory::align(KPCache::s_cache.size());
                # 创建新的轻缓存对象
                m_lightCache = OclLib::createBuffer(m_ctx, CL_MEM_READ_ONLY, m_lightCacheCapacity);
            }

            # 更新轻缓存大小
            m_lightCacheSize = KPCache::s_cache.size();
            # 将缓存数据写入轻缓存对象
            enqueueWriteBuffer(m_lightCache, CL_TRUE, 0, m_lightCacheSize, KPCache::s_cache.data());
        }

        # 记录开始计算 DAG 的时间
        const uint64_t start_ms = Chrono::steadyMSecs();

        # 计算 DAG 的字数
        const uint32_t dag_words = dag_size / sizeof(node);
        # 设置计算 DAG 的内核参数
        m_calculateDagKernel->setArgs(0, m_lightCache, m_dag, dag_words, m_lightCacheSize / sizeof(node));

        # 定义每次计算的 DAG 字数
        constexpr uint32_t N = 1 << 18;

        # 循环计算 DAG
        for (uint32_t start = 0; start < dag_words; start += N) {
            m_calculateDagKernel->setArg(0, sizeof(start), &start);
            m_calculateDagKernel->enqueue(m_queue, N, m_dagWorkGroupSize);
        }

        # 等待计算完成
        OclLib::finish(m_queue);

        # 输出 DAG 计算所花费的时间
        LOG_INFO("%s " YELLOW("KawPow") " DAG for epoch " WHITE_BOLD("%u") " calculated " BLACK_BOLD("(%" PRIu64 "ms)"), Tags::opencl(), epoch, Chrono::steadyMSecs() - start_ms);
    }

    # 获取目标值和 hack_false
    const uint64_t target = job.target();
    const uint32_t hack_false = 0;

    # 设置搜索内核的参数
    OclLib::setKernelArg(m_searchKernel, 0, sizeof(cl_mem), &m_dag);
    OclLib::setKernelArg(m_searchKernel, 1, sizeof(cl_mem), &m_input);
    OclLib::setKernelArg(m_searchKernel, 2, sizeof(target), &target);
    OclLib::setKernelArg(m_searchKernel, 3, sizeof(hack_false), &hack_false);
    # 设置搜索内核的第四个参数为输出内存对象的大小和地址
    OclLib::setKernelArg(m_searchKernel, 4, sizeof(cl_mem), &m_output);
    # 设置搜索内核的第五个参数为停止内存对象的大小和地址
    OclLib::setKernelArg(m_searchKernel, 5, sizeof(cl_mem), &m_stop);

    # 将输入数据写入输入内存对象
    m_blob = blob;
    enqueueWriteBuffer(m_input, CL_TRUE, 0, BLOB_SIZE, m_blob);
// 通知作业提前结束的函数
void OclKawPowRunner::jobEarlyNotification(const Job&)
{
    // 定义一个常量值为1的32位整数
    const uint32_t one = 1;
    // 将常量值写入到m_stop缓冲区中
    const cl_int ret = OclLib::enqueueWriteBuffer(m_controlQueue, m_stop, CL_TRUE, 0, sizeof(one), &one, 0, nullptr, nullptr);
    // 如果写入操作不成功，则抛出运行时错误
    if (ret != CL_SUCCESS) {
        throw std::runtime_error(OclError::toString(ret));
    }
}

// 构建函数
void xmrig::OclKawPowRunner::build()
{
    // 调用基类的build函数
    OclBaseRunner::build();
    // 创建KawPow_CalculateDAGKernel对象并赋值给m_calculateDagKernel
    m_calculateDagKernel = new KawPow_CalculateDAGKernel(m_program);
}

// 初始化函数
void xmrig::OclKawPowRunner::init()
{
    // 调用基类的init函数
    OclBaseRunner::init();
    // 创建一个命令队列并赋值给m_controlQueue
    m_controlQueue = OclLib::createCommandQueue(m_ctx, data().device.id());
    // 创建一个大小为2个32位整数的只读缓冲区并赋值给m_stop
    m_stop = OclLib::createBuffer(m_ctx, CL_MEM_READ_ONLY, sizeof(uint32_t) * 2);
}
// 命名空间结束
} // namespace xmrig
```