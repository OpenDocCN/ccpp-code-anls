# `xmrig\src\backend\opencl\wrappers\OclLib.h`

```cpp
/* XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改它
 *   根据由自由软件基金会发布的GNU通用公共许可证的条款，
 *   无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用的分发，
 *   但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。更多详情请参见
 *   GNU通用公共许可证。
 *
 *   您应该已经收到了GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_OCLLIB_H
#define XMRIG_OCLLIB_H


#include <vector>


#include "3rdparty/cl.h"
#include "base/tools/String.h"

#ifndef CL_DEVICE_TOPOLOGY_AMD
#define CL_DEVICE_TOPOLOGY_AMD 0x4037
#endif
#ifndef CL_DEVICE_BOARD_NAME_AMD
#define CL_DEVICE_BOARD_NAME_AMD 0x4038
#endif
#ifndef CL_DEVICE_TOPOLOGY_TYPE_PCIE_AMD
#define CL_DEVICE_TOPOLOGY_TYPE_PCIE_AMD 1
#endif
#ifndef CL_DEVICE_PCI_BUS_ID_NV
#define CL_DEVICE_PCI_BUS_ID_NV 0x4008
#endif
#ifndef CL_DEVICE_PCI_SLOT_ID_NV
#define CL_DEVICE_PCI_SLOT_ID_NV 0x4009
#endif


namespace xmrig {


class OclLib
{
public:
    static bool init(const char *fileName = nullptr);
    static const char *lastError();
    static void close();

    static inline bool isInitialized()   { return m_initialized; }
    static inline const String &loader() { return m_loader; }

    static cl_command_queue createCommandQueue(cl_context context, cl_device_id device, cl_int *errcode_ret) noexcept;
    static cl_command_queue createCommandQueue(cl_context context, cl_device_id device);
    # 创建 OpenCL 上下文
    static cl_context createContext(const cl_context_properties *properties, cl_uint num_devices, const cl_device_id *devices, void (CL_CALLBACK *pfn_notify)(const char *, const void *, size_t, void *), void *user_data, cl_int *errcode_ret);
    
    # 创建 OpenCL 上下文
    static cl_context createContext(const std::vector<cl_device_id> &ids);
    
    # 构建 OpenCL 程序
    static cl_int buildProgram(cl_program program, cl_uint num_devices, const cl_device_id *device_list, const char *options = nullptr, void (CL_CALLBACK *pfn_notify)(cl_program program, void *user_data) = nullptr, void *user_data = nullptr) noexcept;
    
    # 将内核函数加入到命令队列中进行执行
    static cl_int enqueueNDRangeKernel(cl_command_queue command_queue, cl_kernel kernel, cl_uint work_dim, const size_t *global_work_offset, const size_t *global_work_size, const size_t *local_work_size, cl_uint num_events_in_wait_list, const cl_event *event_wait_list, cl_event *event) noexcept;
    
    # 将缓冲区数据读取到内存中
    static cl_int enqueueReadBuffer(cl_command_queue command_queue, cl_mem buffer, cl_bool blocking_read, size_t offset, size_t size, void *ptr, cl_uint num_events_in_wait_list, const cl_event *event_wait_list, cl_event *event) noexcept;
    
    # 将数据写入缓冲区
    static cl_int enqueueWriteBuffer(cl_command_queue command_queue, cl_mem buffer, cl_bool blocking_write, size_t offset, size_t size, const void *ptr, cl_uint num_events_in_wait_list, const cl_event *event_wait_list, cl_event *event) noexcept;
    
    # 等待命令队列中的任务执行完成
    static cl_int finish(cl_command_queue command_queue) noexcept;
    
    # 获取命令队列的信息
    static cl_int getCommandQueueInfo(cl_command_queue command_queue, cl_command_queue_info param_name, size_t param_value_size, void *param_value, size_t *param_value_size_ret = nullptr) noexcept;
    
    # 获取上下文的信息
    static cl_int getContextInfo(cl_context context, cl_context_info param_name, size_t param_value_size, void *param_value, size_t *param_value_size_ret = nullptr) noexcept;
    
    # 获取设备的 ID
    static cl_int getDeviceIDs(cl_platform_id platform, cl_device_type device_type, cl_uint num_entries, cl_device_id *devices, cl_uint *num_devices) noexcept;
    # 获取设备信息
    static cl_int getDeviceInfo(cl_device_id device, cl_device_info param_name, size_t param_value_size, void *param_value, size_t *param_value_size_ret = nullptr) noexcept;
    
    # 获取内核信息
    static cl_int getKernelInfo(cl_kernel kernel, cl_kernel_info param_name, size_t param_value_size, void *param_value, size_t *param_value_size_ret = nullptr) noexcept;
    
    # 获取内存对象信息
    static cl_int getMemObjectInfo(cl_mem memobj, cl_mem_info param_name, size_t param_value_size, void *param_value, size_t *param_value_size_ret = nullptr) noexcept;
    
    # 获取平台 ID
    static cl_int getPlatformIDs(cl_uint num_entries, cl_platform_id *platforms, cl_uint *num_platforms);
    
    # 获取平台信息
    static cl_int getPlatformInfo(cl_platform_id platform, cl_platform_info param_name, size_t param_value_size, void *param_value, size_t *param_value_size_ret) noexcept;
    
    # 获取程序构建信息
    static cl_int getProgramBuildInfo(cl_program program, cl_device_id device, cl_program_build_info param_name, size_t param_value_size, void *param_value, size_t *param_value_size_ret) noexcept;
    
    # 获取程序信息
    static cl_int getProgramInfo(cl_program program, cl_program_info param_name, size_t param_value_size, void *param_value, size_t *param_value_size_ret = nullptr);
    
    # 释放命令队列
    static cl_int release(cl_command_queue command_queue) noexcept;
    
    # 释放上下文
    static cl_int release(cl_context context) noexcept;
    
    # 释放设备 ID
    static cl_int release(cl_device_id id) noexcept;
    
    # 释放内核
    static cl_int release(cl_kernel kernel) noexcept;
    
    # 释放内存对象
    static cl_int release(cl_mem mem_obj) noexcept;
    
    # 释放程序
    static cl_int release(cl_program program) noexcept;
    
    # 设置内核参数
    static cl_int setKernelArg(cl_kernel kernel, cl_uint arg_index, size_t arg_size, const void *arg_value) noexcept;
    
    # 卸载平台编译器
    static cl_int unloadPlatformCompiler(cl_platform_id platform) noexcept;
    
    # 创建内核
    static cl_kernel createKernel(cl_program program, const char *kernel_name, cl_int *errcode_ret) noexcept;
    
    # 创建内核
    static cl_kernel createKernel(cl_program program, const char *kernel_name);
    
    # 创建缓冲区
    static cl_mem createBuffer(cl_context context, cl_mem_flags flags, size_t size, void *host_ptr = nullptr);
    // 创建一个 OpenCL 内存对象
    static cl_mem createBuffer(cl_context context, cl_mem_flags flags, size_t size, void *host_ptr, cl_int *errcode_ret) noexcept;

    // 创建一个 OpenCL 子缓冲区
    static cl_mem createSubBuffer(cl_mem buffer, cl_mem_flags flags, size_t offset, size_t size, cl_int *errcode_ret) noexcept;

    // 创建一个 OpenCL 子缓冲区
    static cl_mem createSubBuffer(cl_mem buffer, cl_mem_flags flags, size_t offset, size_t size);

    // 保留一个 OpenCL 内存对象
    static cl_mem retain(cl_mem memobj) noexcept;

    // 使用二进制数据创建一个 OpenCL 程序对象
    static cl_program createProgramWithBinary(cl_context context, cl_uint num_devices, const cl_device_id *device_list, const size_t *lengths, const unsigned char **binaries, cl_int *binary_status, cl_int *errcode_ret) noexcept;

    // 使用源码字符串创建一个 OpenCL 程序对象
    static cl_program createProgramWithSource(cl_context context, cl_uint count, const char **strings, const size_t *lengths, cl_int *errcode_ret) noexcept;

    // 保留一个 OpenCL 程序对象
    static cl_program retain(cl_program program) noexcept;

    // 获取平台数量
    static cl_uint getNumPlatforms() noexcept;

    // 获取指定命令队列的无符号整数参数值
    static cl_uint getUint(cl_command_queue command_queue, cl_command_queue_info param_name, cl_uint defaultValue = 0) noexcept;

    // 获取指定上下文的无符号整数参数值
    static cl_uint getUint(cl_context context, cl_context_info param_name, cl_uint defaultValue = 0) noexcept;

    // 获取指定设备的无符号整数参数值
    static cl_uint getUint(cl_device_id id, cl_device_info param, cl_uint defaultValue = 0) noexcept;

    // 获取指定内核的无符号整数参数值
    static cl_uint getUint(cl_kernel kernel, cl_kernel_info  param_name, cl_uint defaultValue = 0) noexcept;

    // 获取指定内存对象的无符号整数参数值
    static cl_uint getUint(cl_mem memobj, cl_mem_info param_name, cl_uint defaultValue = 0) noexcept;

    // 获取指定程序对象的无符号整数参数值
    static cl_uint getUint(cl_program program, cl_program_info param, cl_uint defaultValue = 0) noexcept;

    // 获取指定设备的无符号长整数参数值
    static cl_ulong getUlong(cl_device_id id, cl_device_info param, cl_ulong defaultValue = 0) noexcept;

    // 获取指定内存对象的无符号长整数参数值
    static cl_ulong getUlong(cl_mem memobj, cl_mem_info param_name, cl_ulong defaultValue = 0) noexcept;

    // 获取平台 ID 列表
    static std::vector<cl_platform_id> getPlatformIDs() noexcept;

    // 获取程序构建日志
    static String getProgramBuildLog(cl_program program, cl_device_id device) noexcept;

    // 获取设备信息字符串
    static String getString(cl_device_id id, cl_device_info param) noexcept;
    # 获取 OpenCL 内核对象的信息并以字符串形式返回
    static String getString(cl_kernel kernel, cl_kernel_info param_name) noexcept;
    
    # 获取 OpenCL 平台对象的信息并以字符串形式返回
    static String getString(cl_platform_id platform, cl_platform_info param_name) noexcept;
    
    # 获取 OpenCL 程序对象的信息并以字符串形式返回
    static String getString(cl_program program, cl_program_info param_name) noexcept;
# 声明私有成员和静态成员函数
private:
    # 声明静态成员函数load()，返回布尔类型
    static bool load();
    # 声明静态成员函数defaultLoader()，返回String类型
    static String defaultLoader();

    # 声明静态布尔类型成员变量m_initialized
    static bool m_initialized;
    # 声明静态布尔类型成员变量m_ready
    static bool m_ready;
    # 声明静态String类型成员变量m_loader
    static String m_loader;
};


# 结束xmrig命名空间
} // namespace xmrig

# 结束条件编译指令，关闭XMRIG_OCLLIB_H头文件
#endif /* XMRIG_OCLLIB_H */
```