# `xmrig\src\backend\opencl\runners\OclCnRunner.cpp`

```cpp
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
 * 商品性或适用于特定目的。有关更多详细信息，请参见
 * GNU通用公共许可证。
 *
 * 您应该已经收到了GNU通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include <stdexcept>


#include "backend/opencl/runners/OclCnRunner.h"
#include "backend/opencl/kernels/Cn0Kernel.h"
#include "backend/opencl/kernels/Cn1Kernel.h"
#include "backend/opencl/kernels/Cn2Kernel.h"
#include "backend/opencl/kernels/CnBranchKernel.h"
#include "backend/opencl/OclLaunchData.h"
#include "backend/opencl/runners/tools/OclCnR.h"
#include "backend/opencl/wrappers/OclLib.h"
#include "base/io/log/Log.h"
#include "base/net/stratum/Job.h"
#include "crypto/cn/CnAlgo.h"


xmrig::OclCnRunner::OclCnRunner(size_t index, const OclLaunchData &data) : OclBaseRunner(index, data)
{
    // 根据索引和数据创建 OclCnRunner 对象
    uint32_t stridedIndex = data.thread.stridedIndex();
    const auto f = m_algorithm.family();

    // 如果设备的供应商ID是NVIDIA，则将 stridedIndex 设置为0
    if (data.device.vendorId() == OCL_VENDOR_NVIDIA) {
        stridedIndex = 0;
    }
    // 如果 stridedIndex 为1，并且算法家族是 CN_PICO 或 CN_FEMTO 或（算法家族是 CN 且算法基础是 CN_2）
    // 则将 stridedIndex 设置为2
    else if (stridedIndex == 1 && (f == Algorithm::CN_PICO || f == Algorithm::CN_FEMTO || (f == Algorithm::CN && m_algorithm.base() == Algorithm::CN_2))) {
        stridedIndex = 2;
    }

    // 将选项添加到选项字符串中，包括迭代次数
    m_options += " -DITERATIONS="           + std::to_string(CnAlgo<>::iterations(m_algorithm)) + "U";
    # 将算法的掩码转换为字符串，并添加到编译选项中
    m_options += " -DMASK="                 + std::to_string(CnAlgo<>::mask(m_algorithm)) + "U";
    # 将线程工作大小转换为字符串，并添加到编译选项中
    m_options += " -DWORKSIZE="             + std::to_string(data.thread.worksize()) + "U";
    # 将跨步索引转换为字符串，并添加到编译选项中
    m_options += " -DSTRIDED_INDEX="        + std::to_string(stridedIndex) + "U";
    # 将内存块指数转换为字符串，并添加到编译选项中
    m_options += " -DMEM_CHUNK_EXPONENT="   + std::to_string(1U << data.thread.memChunk()) + "U";
    # 将算法的 L3 缓存大小转换为字符串，并添加到编译选项中
    m_options += " -DMEMORY="               + std::to_string(m_algorithm.l3()) + "LU";
    # 将算法的 ID 转换为字符串，并添加到编译选项中
    m_options += " -DALGO="                 + std::to_string(m_algorithm.id());
    # 将算法的基础值转换为字符串，并添加到编译选项中
    m_options += " -DALGO_BASE="            + std::to_string(m_algorithm.base());
    # 将算法的族转换为字符串，并添加到编译选项中
    m_options += " -DALGO_FAMILY="          + std::to_string(m_algorithm.family());
    # 将线程展开因子转换为字符串，并添加到编译选项中
    m_options += " -DCN_UNROLL="            + std::to_string(data.thread.unrollFactor());
# OclCnRunner 类的析构函数
xmrig::OclCnRunner::~OclCnRunner()
{
    # 释放 m_cn0 指针指向的内存
    delete m_cn0;
    # 释放 m_cn1 指针指向的内存
    delete m_cn1;
    # 释放 m_cn2 指针指向的内存
    delete m_cn2;

    # 释放 m_scratchpads 指针指向的内存
    OclLib::release(m_scratchpads);
    # 释放 m_states 指针指向的内存
    OclLib::release(m_states);

    # 遍历 BRANCH_MAX，释放 m_branchKernels[i] 指针指向的内存
    for (size_t i = 0; i < BRANCH_MAX; ++i) {
        delete m_branchKernels[i];
        # 释放 m_branches[i] 指针指向的内存
        OclLib::release(m_branches[i]);
    }

    # 如果 m_algorithm 是 CN_R 算法，则释放 m_cnr 指针指向的内存，并清除 OclCnR 类的状态
    if (m_algorithm == Algorithm::CN_R) {
        OclLib::release(m_cnr);
        OclCnR::clear();
    }
}

# 返回缓冲区大小
size_t xmrig::OclCnRunner::bufferSize() const
{
    return OclBaseRunner::bufferSize() +
           align(m_algorithm.l3() * m_intensity) +
           align(200 * m_intensity) +
           (align(sizeof(cl_uint) * (m_intensity + 2)) * BRANCH_MAX);
}

# 运行函数，计算哈希值
void xmrig::OclCnRunner::run(uint32_t nonce, uint32_t *hashOutput)
{
    # 静态常量 zero 赋值为 0
    static const cl_uint zero = 0;

    # 计算工作组大小和全局线程数
    const size_t w_size = data().thread.worksize();
    const size_t g_thd  = ((m_intensity + w_size - 1U) / w_size) * w_size;

    # 断言全局线程数是工作组大小的整数倍
    assert(g_thd % w_size == 0);

    # 将 m_branches[i] 缓冲区的前 m_intensity 个元素写入 zero
    for (size_t i = 0; i < BRANCH_MAX; ++i) {
        enqueueWriteBuffer(m_branches[i], CL_FALSE, sizeof(cl_uint) * m_intensity, sizeof(cl_uint), &zero);
    }

    # 将 m_output 缓冲区的前 0xFF 个元素写入 zero
    enqueueWriteBuffer(m_output, CL_FALSE, sizeof(cl_uint) * 0xFF, sizeof(cl_uint), &zero);

    # 调用 m_cn0、m_cn1、m_cn2 和 m_branchKernels 的 enqueue 函数
    m_cn0->enqueue(m_queue, nonce, g_thd);
    m_cn1->enqueue(m_queue, nonce, g_thd, w_size);
    m_cn2->enqueue(m_queue, nonce, g_thd);

    for (auto kernel : m_branchKernels) {
        kernel->enqueue(m_queue, nonce, g_thd, w_size);
    }

    # 完成哈希计算，将结果写入 hashOutput
    finalize(hashOutput);
}

# 设置作业和数据块
void xmrig::OclCnRunner::set(const Job &job, uint8_t *blob)
{
    # 如果作业大小超过最大值，则抛出长度错误异常
    if (job.size() > (Job::kMaxBlobSize - 4)) {
        throw std::length_error("job size too big");
    }

    # 计算输入长度
    const int inlen = static_cast<int>(job.size() + 136 - (job.size() % 136));

    # 在 blob 中设置特定值
    blob[job.size()] = 0x01;
    memset(blob + job.size() + 1, 0, inlen - job.size() - 1);
    blob[inlen - 1] |= 0x80;

    # 将数据写入 m_input 缓冲区
    enqueueWriteBuffer(m_input, CL_TRUE, 0, inlen, blob);

    # 设置 m_cn0 的参数
    m_cn0->setArg(1, sizeof(int), &inlen);
}
    # 如果算法为CN_R并且高度不等于作业的高度
    if (m_algorithm == Algorithm::CN_R && m_height != job.height()) {
        # 删除m_cn1对象
        delete m_cn1;

        # 更新高度为作业的高度
        m_height     = job.height();
        # 获取对应高度的OclCnR程序
        auto program = OclCnR::get(*this, m_height);
        # 创建新的Cn1Kernel对象
        m_cn1        = new Cn1Kernel(program, m_height);
        # 设置Cn1Kernel对象的参数
        m_cn1->setArgs(m_input, m_scratchpads, m_states, m_intensity);

        # 如果m_cnr不等于program
        if (m_cnr != program) {
            # 释放m_cnr
            OclLib::release(m_cnr);
            # 保留program到m_cnr
            m_cnr = OclLib::retain(program);
        }
    }

    # 遍历m_branchKernels中的每个kernel
    for (auto kernel : m_branchKernels) {
        # 设置kernel的目标为作业的目标
        kernel->setTarget(job.target());
    }
# 构建 OclCnRunner 类的方法
void xmrig::OclCnRunner::build()
{
    # 调用父类的 build 方法
    OclBaseRunner::build();

    # 创建 Cn0Kernel 对象，并设置参数
    m_cn0 = new Cn0Kernel(m_program);
    m_cn0->setArgs(m_input, 0, m_scratchpads, m_states, m_intensity);

    # 创建 Cn2Kernel 对象，并设置参数
    m_cn2 = new Cn2Kernel(m_program);
    m_cn2->setArgs(m_scratchpads, m_states, m_branches, m_intensity);

    # 如果算法不是 CN_R，则创建 Cn1Kernel 对象，并设置参数
    if (m_algorithm != Algorithm::CN_R) {
        m_cn1 = new Cn1Kernel(m_program);
        m_cn1->setArgs(m_input, m_scratchpads, m_states, m_intensity);
    }

    # 循环创建 CnBranchKernel 对象，并设置参数
    for (size_t i = 0; i < BRANCH_MAX; ++i) {
        auto kernel = new CnBranchKernel(i, m_program);
        kernel->setArgs(m_states, m_branches[i], m_output, m_intensity);

        m_branchKernels[i] = kernel;
    }
}

# 初始化 OclCnRunner 类的方法
void xmrig::OclCnRunner::init()
{
    # 调用父类的 init 方法
    OclBaseRunner::init();

    # 创建用于存储数据的缓冲区
    m_scratchpads = createSubBuffer(CL_MEM_READ_WRITE, m_algorithm.l3() * m_intensity);
    m_states      = createSubBuffer(CL_MEM_READ_WRITE, 200 * m_intensity);

    # 循环创建用于存储数据的缓冲区
    for (size_t i = 0; i < BRANCH_MAX; ++i) {
        m_branches[i] = createSubBuffer(CL_MEM_READ_WRITE, sizeof(cl_uint) * (m_intensity + 2));
    }
}
```