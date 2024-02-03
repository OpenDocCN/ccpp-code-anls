# `xmrig\src\backend\opencl\runners\OclRxJitRunner.cpp`

```cpp
/*
 * XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改它
 * 根据由自由软件基金会发布的GNU通用公共许可证的条款，
 * 无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 * 本程序是希望它有用的分发，
 * 但没有任何保证；甚至没有暗示的保证
 * 商品性或适用于特定目的。更多细节请参阅
 * GNU通用公共许可证。
 *
 * 您应该已经收到了GNU通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include <stdexcept>


#include "backend/opencl/runners/OclRxJitRunner.h"
#include "backend/opencl/cl/rx/randomx_run_gfx803.h"
#include "backend/opencl/cl/rx/randomx_run_gfx900.h"
#include "backend/opencl/cl/rx/randomx_run_gfx1010.h"
#include "backend/opencl/kernels/rx/Blake2bHashRegistersKernel.h"
#include "backend/opencl/kernels/rx/HashAesKernel.h"
#include "backend/opencl/kernels/rx/RxJitKernel.h"
#include "backend/opencl/kernels/rx/RxRunKernel.h"
#include "backend/opencl/OclLaunchData.h"
#include "backend/opencl/wrappers/OclLib.h"
#include "backend/opencl/wrappers/OclError.h"


// 创建OclRxJitRunner类的构造函数，传入索引和OclLaunchData对象
xmrig::OclRxJitRunner::OclRxJitRunner(size_t index, const OclLaunchData &data) : OclRxBaseRunner(index, data)
{
}


// OclRxJitRunner类的析构函数
xmrig::OclRxJitRunner::~OclRxJitRunner()
{
    // 释放m_randomx_jit指针指向的内存
    delete m_randomx_jit;
    // 释放m_randomx_run指针指向的内存
    delete m_randomx_run;

    // 释放m_asmProgram指针指向的内存
    OclLib::release(m_asmProgram);
    // 释放m_intermediate_programs指针指向的内存
    OclLib::release(m_intermediate_programs);
    // 释放m_programs指针指向的内存
    OclLib::release(m_programs);
    // 释放m_registers指针指向的内存
    OclLib::release(m_registers);
}


// 返回OclRxJitRunner类的缓冲区大小
size_t xmrig::OclRxJitRunner::bufferSize() const
{
    // 返回OclRxBaseRunner类的缓冲区大小加上256 * m_intensity的对齐大小，5120 * m_intensity的对齐大小，10048 * m_intensity的对齐大小
    return OclRxBaseRunner::bufferSize() + align(256 * m_intensity) + align(5120 * m_intensity) + align(10048 * m_intensity);
}
// 构建函数，继承自OclRxBaseRunner的build函数
void xmrig::OclRxJitRunner::build()
{
    // 调用OclRxBaseRunner的build函数
    OclRxBaseRunner::build();

    // 设置m_hashAes1Rx4的参数
    m_hashAes1Rx4->setArgs(m_scratchpads, m_registers, 256, m_intensity);
    // 设置m_blake2b_hash_registers_32的参数
    m_blake2b_hash_registers_32->setArgs(m_hashes, m_registers, 256);
    // 设置m_blake2b_hash_registers_64的参数
    m_blake2b_hash_registers_64->setArgs(m_hashes, m_registers, 256);

    // 创建RxJitKernel对象，并设置参数
    m_randomx_jit = new RxJitKernel(m_program);
    m_randomx_jit->setArgs(m_entropy, m_registers, m_intermediate_programs, m_programs, m_intensity, m_rounding);

    // 如果无法加载汇编程序，则抛出运行时错误
    if (!loadAsmProgram()) {
        throw std::runtime_error(OclError::toString(CL_INVALID_PROGRAM));
    }

    // 创建RxRunKernel对象，并设置参数
    m_randomx_run = new RxRunKernel(m_asmProgram);
    m_randomx_run->setArgs(m_dataset, m_scratchpads, m_registers, m_rounding, m_programs, m_intensity, m_algorithm);
}


// 执行函数，执行随机X JIT
void xmrig::OclRxJitRunner::execute(uint32_t iteration)
{
    // 将m_randomx_jit加入队列，设置强度和迭代次数
    m_randomx_jit->enqueue(m_queue, m_intensity, iteration);

    // 完成队列中的任务
    OclLib::finish(m_queue);

    // 将m_randomx_run加入队列，设置强度和轮数
    m_randomx_run->enqueue(m_queue, m_intensity, (m_gcn_version == 15) ? 32 : 64);
}


// 初始化函数，初始化寄存器和程序
void xmrig::OclRxJitRunner::init()
{
    // 调用OclRxBaseRunner的init函数
    OclRxBaseRunner::init();

    // 创建256 * m_intensity大小的寄存器子缓冲区
    m_registers = createSubBuffer(CL_MEM_READ_WRITE | CL_MEM_HOST_NO_ACCESS, 256 * m_intensity);
    // 创建5120 * m_intensity大小的中间程序子缓冲区
    m_intermediate_programs = createSubBuffer(CL_MEM_READ_WRITE | CL_MEM_HOST_NO_ACCESS, 5120 * m_intensity);
    // 创建10048 * m_intensity大小的程序子缓冲区
    m_programs = createSubBuffer(CL_MEM_READ_WRITE | CL_MEM_HOST_NO_ACCESS, 10048 * m_intensity);
}


// 加载汇编程序函数
bool xmrig::OclRxJitRunner::loadAsmProgram()
{
    // 从编译好的OpenCL代码中读取ELF头部标志
    uint32_t elf_header_flags = 0;
    const uint32_t elf_header_flags_offset = 0x30;

    // 获取程序的二进制大小
    size_t bin_size = 0;
    if (OclLib::getProgramInfo(m_program, CL_PROGRAM_BINARY_SIZES, sizeof(bin_size), &bin_size) != CL_SUCCESS) {
        return false;
    }

    // 创建二进制数据的vector
    std::vector<char> binary_data(bin_size);
    // 创建一个指针数组，其中包含指向二进制数据的指针
    char* tmp[1] = { binary_data.data() };
    // 获取程序信息，如果失败则返回false
    if (OclLib::getProgramInfo(m_program, CL_PROGRAM_BINARIES, sizeof(char*), tmp) != CL_SUCCESS) {
        return false;
    }

    // 如果二进制数据的大小大于等于 elf_header_flags_offset 加上一个 uint32_t 的大小
    if (bin_size >= elf_header_flags_offset + sizeof(uint32_t)) {
        // 从二进制数据中解释出 elf_header_flags
        elf_header_flags = *reinterpret_cast<uint32_t*>((binary_data.data() + elf_header_flags_offset));
    }

    // 初始化变量
    size_t len              = 0;
    unsigned char *binary   = nullptr;

    // 根据 m_gcn_version 的值选择不同的预编译二进制数据和大小
    switch (m_gcn_version) {
    case 14:
        len = randomx_run_gfx900_bin_size;
        binary = randomx_run_gfx900_bin;
        break;
    case 15:
        len = randomx_run_gfx1010_bin_size;
        binary = randomx_run_gfx1010_bin;
        break;
    default:
        len = randomx_run_gfx803_bin_size;
        binary = randomx_run_gfx803_bin;
        break;
    }

    // 如果 elf_header_flags 不为0，则在预编译二进制数据中设置正确的内部设备ID
    if (elf_header_flags) {
        *reinterpret_cast<uint32_t*>(binary + elf_header_flags_offset) = elf_header_flags;
    }

    // 初始化变量
    cl_int status   = 0;
    cl_int ret      = 0;
    cl_device_id device = data().device.id();

    // 使用预编译二进制数据创建程序
    m_asmProgram = OclLib::createProgramWithBinary(ctx(), 1, &device, &len, (const unsigned char**) &binary, &status, &ret);
    // 如果创建程序失败则返回false
    if (ret != CL_SUCCESS) {
        return false;
    }

    // 构建程序，如果成功则返回true
    return OclLib::buildProgram(m_asmProgram, 1, &device) == CL_SUCCESS;
# 闭合前面的函数定义
```