# `xmrig\src\backend\opencl\wrappers\OclLib.cpp`

```cpp
/* XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改它
 *   根据由自由软件基金会发布的GNU通用公共许可证的条款，版本为3或
 *   （根据您的选择）任何以后的版本。
 *
 *   本程序是希望它有用，
 *   但没有任何保证；甚至没有暗示的保证适用于特定用途。请参阅
 *   GNU通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include <thread>
#include <stdexcept>
#include <uv.h>


#include "backend/opencl/wrappers/OclLib.h"
#include "backend/common/Tags.h"
#include "backend/opencl/wrappers/OclError.h"
#include "base/io/Env.h"
#include "base/io/log/Log.h"


#if defined(OCL_DEBUG_REFERENCE_COUNT)
#   define LOG_REFS(x, ...) xmrig::Log::print(xmrig::Log::WARNING, x, ##__VA_ARGS__)
#endif

// 定义 OpenCL 库的动态链接库对象
static uv_lib_t oclLib;

// 定义错误模板
static const char *kErrorTemplate                    = MAGENTA_BG_BOLD(WHITE_BOLD_S " opencl  ") RED(" error ") RED_BOLD("%s") RED(" when calling ") RED_BOLD("%s");

// 定义 OpenCL 函数名常量
static const char *kBuildProgram                     = "clBuildProgram";
static const char *kCreateBuffer                     = "clCreateBuffer";
static const char *kCreateCommandQueue               = "clCreateCommandQueue";
static const char *kCreateCommandQueueWithProperties = "clCreateCommandQueueWithProperties";
static const char *kCreateContext                    = "clCreateContext";
static const char *kCreateKernel                     = "clCreateKernel";
static const char *kCreateProgramWithBinary          = "clCreateProgramWithBinary";
# 定义字符串常量，表示 OpenCL API 中的不同函数名称
static const char *kCreateProgramWithSource          = "clCreateProgramWithSource";
static const char *kCreateSubBuffer                  = "clCreateSubBuffer";
static const char *kEnqueueNDRangeKernel             = "clEnqueueNDRangeKernel";
static const char *kEnqueueReadBuffer                = "clEnqueueReadBuffer";
static const char *kEnqueueWriteBuffer               = "clEnqueueWriteBuffer";
static const char *kFinish                           = "clFinish";
static const char *kGetCommandQueueInfo              = "clGetCommandQueueInfo";
static const char *kGetContextInfo                   = "clGetContextInfo";
static const char *kGetDeviceIDs                     = "clGetDeviceIDs";
static const char *kGetDeviceInfo                    = "clGetDeviceInfo";
static const char *kGetKernelInfo                    = "clGetKernelInfo";
static const char *kGetMemObjectInfo                 = "clGetMemObjectInfo";
static const char *kGetPlatformIDs                   = "clGetPlatformIDs";
static const char *kGetPlatformInfo                  = "clGetPlatformInfo";
static const char *kGetProgramBuildInfo              = "clGetProgramBuildInfo";
static const char *kGetProgramInfo                   = "clGetProgramInfo";
static const char *kReleaseCommandQueue              = "clReleaseCommandQueue";
static const char *kReleaseContext                   = "clReleaseContext";
static const char *kReleaseDevice                    = "clReleaseDevice";
static const char *kReleaseKernel                    = "clReleaseKernel";
static const char *kReleaseMemObject                 = "clReleaseMemObject";
static const char *kReleaseProgram                   = "clReleaseProgram";
static const char *kRetainMemObject                  = "clRetainMemObject";
static const char *kRetainProgram                    = "clRetainProgram";
static const char *kSetKernelArg                     = "clSetKernelArg";
static const char *kSetMemObjectDestructorCallback   = "clSetMemObjectDestructorCallback";
# 定义常量字符串，表示符号未找到和卸载平台编译器
static const char *kSymbolNotFound                   = "symbol not found";
static const char *kUnloadPlatformCompiler           = "clUnloadPlatformCompiler";


# 如果定义了 OpenCL 2.0 版本
typedef cl_command_queue (CL_API_CALL *createCommandQueueWithProperties_t)(cl_context, cl_device_id, const cl_queue_properties *, cl_int *);
#endif

# 定义函数指针类型，用于创建命令队列、上下文、构建程序、执行内核等操作
typedef cl_command_queue (CL_API_CALL *createCommandQueue_t)(cl_context, cl_device_id, cl_command_queue_properties, cl_int *);
typedef cl_context (CL_API_CALL *createContext_t)(const cl_context_properties *, cl_uint, const cl_device_id *, void (CL_CALLBACK *pfn_notify)(const char *, const void *, size_t, void *), void *, cl_int *);
typedef cl_int (CL_API_CALL *buildProgram_t)(cl_program, cl_uint, const cl_device_id *, const char *, void (CL_CALLBACK *pfn_notify)(cl_program, void *), void *);
typedef cl_int (CL_API_CALL *enqueueNDRangeKernel_t)(cl_command_queue, cl_kernel, cl_uint, const size_t *, const size_t *, const size_t *, cl_uint, const cl_event *, cl_event *);
typedef cl_int (CL_API_CALL *enqueueReadBuffer_t)(cl_command_queue, cl_mem, cl_bool, size_t, size_t, void *, cl_uint, const cl_event *, cl_event *);
typedef cl_int (CL_API_CALL *enqueueWriteBuffer_t)(cl_command_queue, cl_mem, cl_bool, size_t, size_t, const void *, cl_uint, const cl_event *, cl_event *);
typedef cl_int (CL_API_CALL *finish_t)(cl_command_queue);
typedef cl_int (CL_API_CALL *getCommandQueueInfo_t)(cl_command_queue, cl_command_queue_info, size_t, void *, size_t *);
typedef cl_int (CL_API_CALL *getContextInfo_t)(cl_context, cl_context_info, size_t, void *, size_t *);
typedef cl_int (CL_API_CALL *getDeviceIDs_t)(cl_platform_id, cl_device_type, cl_uint, cl_device_id *, cl_uint *);
typedef cl_int (CL_API_CALL *getDeviceInfo_t)(cl_device_id, cl_device_info, size_t, void *, size_t *);
typedef cl_int (CL_API_CALL *getKernelInfo_t)(cl_kernel, cl_kernel_info, size_t, void *, size_t *);
// 定义函数指针类型，用于获取内存对象信息
typedef cl_int (CL_API_CALL *getMemObjectInfo_t)(cl_mem, cl_mem_info, size_t, void *, size_t *);
// 定义函数指针类型，用于获取平台 ID
typedef cl_int (CL_API_CALL *getPlatformIDs_t)(cl_uint, cl_platform_id *, cl_uint *);
// 定义函数指针类型，用于获取平台信息
typedef cl_int (CL_API_CALL *getPlatformInfo_t)(cl_platform_id, cl_platform_info, size_t, void *, size_t *);
// 定义函数指针类型，用于获取程序构建信息
typedef cl_int (CL_API_CALL *getProgramBuildInfo_t)(cl_program, cl_device_id, cl_program_build_info, size_t, void *, size_t *);
// 定义函数指针类型，用于获取程序信息
typedef cl_int (CL_API_CALL *getProgramInfo_t)(cl_program, cl_program_info, size_t, void *, size_t *);
// 定义函数指针类型，用于释放命令队列
typedef cl_int (CL_API_CALL *releaseCommandQueue_t)(cl_command_queue);
// 定义函数指针类型，用于释放上下文
typedef cl_int (CL_API_CALL *releaseContext_t)(cl_context);
// 定义函数指针类型，用于释放设备
typedef cl_int (CL_API_CALL *releaseDevice_t)(cl_device_id device);
// 定义函数指针类型，用于释放内核
typedef cl_int (CL_API_CALL *releaseKernel_t)(cl_kernel);
// 定义函数指针类型，用于释放内存对象
typedef cl_int (CL_API_CALL *releaseMemObject_t)(cl_mem);
// 定义函数指针类型，用于释放程序
typedef cl_int (CL_API_CALL *releaseProgram_t)(cl_program);
// 定义函数指针类型，用于增加内存对象的引用计数
typedef cl_int (CL_API_CALL *retainMemObject_t)(cl_mem);
// 定义函数指针类型，用于增加程序的引用计数
typedef cl_int (CL_API_CALL *retainProgram_t)(cl_program);
// 定义函数指针类型，用于设置内核参数
typedef cl_int (CL_API_CALL *setKernelArg_t)(cl_kernel, cl_uint, size_t, const void *);
// 定义函数指针类型，用于设置内存对象的析构回调函数
typedef cl_int (CL_API_CALL *setMemObjectDestructorCallback_t)(cl_mem, void (CL_CALLBACK *)(cl_mem, void *), void *);
// 定义函数指针类型，用于卸载平台编译器
typedef cl_int (CL_API_CALL *unloadPlatformCompiler_t)(cl_platform_id);
// 定义函数指针类型，用于创建内核
typedef cl_kernel (CL_API_CALL *createKernel_t)(cl_program, const char *, cl_int *);
// 定义函数指针类型，用于创建缓冲区
typedef cl_mem (CL_API_CALL *createBuffer_t)(cl_context, cl_mem_flags, size_t, void *, cl_int *);
// 定义函数指针类型，用于创建子缓冲区
typedef cl_mem (CL_API_CALL *createSubBuffer_t)(cl_mem, cl_mem_flags, cl_buffer_create_type, const void *, cl_int *);
// 定义函数指针类型，用于使用二进制数据创建程序
typedef cl_program (CL_API_CALL *createProgramWithBinary_t)(cl_context, cl_uint, const cl_device_id *, const size_t *, const unsigned char **, cl_int *, cl_int *);
// 定义函数指针类型，用于使用源码创建程序
typedef cl_program (CL_API_CALL *createProgramWithSource_t)(cl_context, cl_uint, const char **, const size_t *, cl_int *);

// 如果定义了 OpenCL 2.0 版本，则声明创建带属性的命令队列的函数指针
#if defined(CL_VERSION_2_0)
static createCommandQueueWithProperties_t pCreateCommandQueueWithProperties = nullptr;
// 定义静态函数指针，用于指向 OpenCL 库中的函数
static buildProgram_t  pBuildProgram                                        = nullptr;
static createBuffer_t pCreateBuffer                                         = nullptr;
static createCommandQueue_t pCreateCommandQueue                             = nullptr;
static createContext_t pCreateContext                                       = nullptr;
static createKernel_t pCreateKernel                                         = nullptr;
static createProgramWithBinary_t pCreateProgramWithBinary                   = nullptr;
static createProgramWithSource_t pCreateProgramWithSource                   = nullptr;
static createSubBuffer_t pCreateSubBuffer                                   = nullptr;
static enqueueNDRangeKernel_t pEnqueueNDRangeKernel                         = nullptr;
static enqueueReadBuffer_t pEnqueueReadBuffer                               = nullptr;
static enqueueWriteBuffer_t pEnqueueWriteBuffer                             = nullptr;
static finish_t pFinish                                                     = nullptr;
static getCommandQueueInfo_t pGetCommandQueueInfo                           = nullptr;
static getContextInfo_t pGetContextInfo                                     = nullptr;
static getDeviceIDs_t pGetDeviceIDs                                         = nullptr;
static getDeviceInfo_t pGetDeviceInfo                                       = nullptr;
static getKernelInfo_t pGetKernelInfo                                       = nullptr;
static getMemObjectInfo_t pGetMemObjectInfo                                 = nullptr;
static getPlatformIDs_t pGetPlatformIDs                                     = nullptr;
static getPlatformInfo_t pGetPlatformInfo                                   = nullptr;
static getProgramBuildInfo_t pGetProgramBuildInfo                           = nullptr;
static getProgramInfo_t pGetProgramInfo                                     = nullptr;
// 定义静态函数指针，用于释放命令队列、上下文、设备、内核、内存对象、程序、保留内存对象、保留程序、设置内核参数、设置内存对象析构回调、卸载平台编译器
static releaseCommandQueue_t pReleaseCommandQueue                           = nullptr;
static releaseContext_t pReleaseContext                                     = nullptr;
static releaseDevice_t pReleaseDevice                                       = nullptr;
static releaseKernel_t pReleaseKernel                                       = nullptr;
static releaseMemObject_t pReleaseMemObject                                 = nullptr;
static releaseProgram_t pReleaseProgram                                     = nullptr;
static retainMemObject_t pRetainMemObject                                   = nullptr;
static retainProgram_t pRetainProgram                                       = nullptr;
static setKernelArg_t pSetKernelArg                                         = nullptr;
static setMemObjectDestructorCallback_t pSetMemObjectDestructorCallback     = nullptr;
static unloadPlatformCompiler_t pUnloadPlatformCompiler                     = nullptr;

// 定义宏，用于动态加载函数指针
#define DLSYM(x) if (uv_dlsym(&oclLib, k##x, reinterpret_cast<void**>(&p##x)) == -1) { throw std::runtime_error(kSymbolNotFound); }

// 命名空间 xmrig
namespace xmrig {

// 初始化静态成员变量
bool OclLib::m_initialized = false;
bool OclLib::m_ready       = false;
String OclLib::m_loader;

// 模板函数，用于获取 OpenCL 字符串信息
template<typename FUNC, typename OBJ, typename PARAM>
static String getOclString(FUNC fn, OBJ obj, PARAM param)
{
    size_t size = 0;
    // 获取字符串信息的大小
    if (fn(obj, param, 0, nullptr, &size) != CL_SUCCESS) {
        return String();
    }

    // 分配内存并获取字符串信息
    char *buf = new char[size]();
    fn(obj, param, size, buf, nullptr);

    return String(buf);
}

} // namespace xmrig

// 初始化函数
bool xmrig::OclLib::init(const char *fileName)
{
    // 如果未初始化，则进行初始化
    if (!m_initialized) {
        // 设置加载器文件名，如果为空则使用默认加载器
        m_loader      = fileName == nullptr ? defaultLoader() : Env::expand(fileName);
        // 使用加载器加载 OpenCL 库，并加载函数指针
        m_ready       = uv_dlopen(m_loader, &oclLib) == 0 && load();
        // 标记已经初始化
        m_initialized = true;
    }

    return m_ready;
}

// 获取最后的错误信息
const char *xmrig::OclLib::lastError()
{
    return uv_dlerror(&oclLib);
}

// 关闭 OpenCL 库
void xmrig::OclLib::close()
{
    uv_dlclose(&oclLib);
}
bool xmrig::OclLib::load()
{
    try {
        // 尝试动态加载 OpenCL 库中的函数
        DLSYM(CreateCommandQueue);
        DLSYM(CreateContext);
        DLSYM(BuildProgram);
        // ... 其他函数
    } catch (std::exception &ex) {
        // 如果加载函数失败，则返回 false
        return false;
    }

    // 如果定义了 OpenCL 2.0 版本，则加载额外的函数
#   if defined(CL_VERSION_2_0)
    uv_dlsym(&oclLib, kCreateCommandQueueWithProperties, reinterpret_cast<void**>(&pCreateCommandQueueWithProperties));
#   endif

    // 加载成功则返回 true
    return true;
}


xmrig::String xmrig::OclLib::defaultLoader()
{
    // 根据不同的操作系统返回默认的 OpenCL 库路径
#   if defined(__APPLE__)
    return "/System/Library/Frameworks/OpenCL.framework/OpenCL";
#   elif defined(_WIN32)
    return "OpenCL.dll";
#   else
    return "libOpenCL.so";
#   endif
}


cl_command_queue xmrig::OclLib::createCommandQueue(cl_context context, cl_device_id device, cl_int *errcode_ret) noexcept
{
    cl_command_queue result = nullptr;

    // 如果定义了 OpenCL 2.0 版本，则执行以下代码
#   if defined(CL_VERSION_2_0)
    # 检查是否存在 pCreateCommandQueueWithProperties 函数
    if (pCreateCommandQueueWithProperties) {
        # 定义命令队列属性数组
        const cl_queue_properties commandQueueProperties[] = { 0, 0, 0 };
        # 调用 pCreateCommandQueueWithProperties 函数创建命令队列
        result = pCreateCommandQueueWithProperties(context, device, commandQueueProperties, errcode_ret);
    }
    # 如果不存在 pCreateCommandQueueWithProperties 函数，则执行以下代码
    else {
#   endif
        // 定义命令队列属性
        const cl_command_queue_properties commandQueueProperties = { 0 };
        // 创建命令队列
        result = pCreateCommandQueue(context, device, commandQueueProperties, errcode_ret);
#   if defined(CL_VERSION_2_0)
    }
#   endif

    // 如果创建命令队列时出现错误
    if (*errcode_ret != CL_SUCCESS) {
        // 记录错误信息
        LOG_ERR(kErrorTemplate, OclError::toString(*errcode_ret), kCreateCommandQueueWithProperties);
        // 返回空指针
        return nullptr;
    }

    // 返回结果
    return result;
}


// 创建命令队列
cl_command_queue xmrig::OclLib::createCommandQueue(cl_context context, cl_device_id device)
{
    cl_int ret = 0;
    // 调用创建命令队列的函数
    cl_command_queue queue = createCommandQueue(context, device, &ret);
    // 如果创建命令队列时出现错误
    if (ret != CL_SUCCESS) {
        // 抛出运行时错误
        throw std::runtime_error(OclError::toString(ret));
    }

    // 返回命令队列
    return queue;
}


// 创建上下文
cl_context xmrig::OclLib::createContext(const cl_context_properties *properties, cl_uint num_devices, const cl_device_id *devices, void (CL_CALLBACK *pfn_notify)(const char *, const void *, size_t, void *), void *user_data, cl_int *errcode_ret)
{
    assert(pCreateContext != nullptr);

    // 调用创建上下文的函数
    auto result = pCreateContext(properties, num_devices, devices, pfn_notify, user_data, errcode_ret);
    // 如果创建上下文时出现错误
    if (*errcode_ret != CL_SUCCESS) {
        // 记录错误信息
        LOG_ERR(kErrorTemplate, OclError::toString(*errcode_ret), kCreateContext);
        // 返回空指针
        return nullptr;
    }

    // 返回结果
    return result;
}


// 创建上下文
cl_context xmrig::OclLib::createContext(const std::vector<cl_device_id> &ids)
{
    cl_int ret = 0;
    // 调用创建上下文的函数
    return createContext(nullptr, static_cast<cl_uint>(ids.size()), ids.data(), nullptr, nullptr, &ret);
}


// 构建程序
cl_int xmrig::OclLib::buildProgram(cl_program program, cl_uint num_devices, const cl_device_id *device_list, const char *options, void (CL_CALLBACK *pfn_notify)(cl_program program, void *user_data), void *user_data) noexcept
{
    assert(pBuildProgram != nullptr);

    // 调用构建程序的函数
    const cl_int ret = pBuildProgram(program, num_devices, device_list, options, pfn_notify, user_data);
    // 如果构建程序时出现错误
    if (ret != CL_SUCCESS) {
        // 记录错误信息
        LOG_ERR(kErrorTemplate, OclError::toString(ret), kBuildProgram);
    }

    // 返回结果
    return ret;
}
// 将 NDRange 内核添加到命令队列中执行
cl_int xmrig::OclLib::enqueueNDRangeKernel(cl_command_queue command_queue, cl_kernel kernel, cl_uint work_dim, const size_t *global_work_offset, const size_t *global_work_size, const size_t *local_work_size, cl_uint num_events_in_wait_list, const cl_event *event_wait_list, cl_event *event) noexcept
{
    // 断言 pEnqueueNDRangeKernel 不为空
    assert(pEnqueueNDRangeKernel != nullptr);

    // 调用 OpenCL 函数 pEnqueueNDRangeKernel
    return pEnqueueNDRangeKernel(command_queue, kernel, work_dim, global_work_offset, global_work_size, local_work_size, num_events_in_wait_list, event_wait_list, event);
}


// 将缓冲区的数据读取到主机内存中
cl_int xmrig::OclLib::enqueueReadBuffer(cl_command_queue command_queue, cl_mem buffer, cl_bool blocking_read, size_t offset, size_t size, void *ptr, cl_uint num_events_in_wait_list, const cl_event *event_wait_list, cl_event *event) noexcept
{
    // 断言 pEnqueueReadBuffer 不为空
    assert(pEnqueueReadBuffer != nullptr);

    // 调用 OpenCL 函数 pEnqueueReadBuffer
    const cl_int ret = pEnqueueReadBuffer(command_queue, buffer, blocking_read, offset, size, ptr, num_events_in_wait_list, event_wait_list, event);
    // 如果返回值不是 CL_SUCCESS，则记录错误日志
    if (ret != CL_SUCCESS) {
        LOG_ERR(kErrorTemplate, OclError::toString(ret), kEnqueueReadBuffer);
    }

    return ret;
}


// 将数据写入缓冲区
cl_int xmrig::OclLib::enqueueWriteBuffer(cl_command_queue command_queue, cl_mem buffer, cl_bool blocking_write, size_t offset, size_t size, const void *ptr, cl_uint num_events_in_wait_list, const cl_event *event_wait_list, cl_event *event) noexcept
{
    // 断言 pEnqueueWriteBuffer 不为空
    assert(pEnqueueWriteBuffer != nullptr);

    // 调用 OpenCL 函数 pEnqueueWriteBuffer
    const cl_int ret = pEnqueueWriteBuffer(command_queue, buffer, blocking_write, offset, size, ptr, num_events_in_wait_list, event_wait_list, event);
    // 如果返回值不是 CL_SUCCESS，则记录错误日志
    if (ret != CL_SUCCESS) {
        LOG_ERR(kErrorTemplate, OclError::toString(ret), kEnqueueWriteBuffer);
    }

    return ret;
}


// 等待命令队列中的所有命令执行完毕
cl_int xmrig::OclLib::finish(cl_command_queue command_queue) noexcept
{
    // 断言 pFinish 不为空
    assert(pFinish != nullptr);

    // 调用 OpenCL 函数 pFinish
    return pFinish(command_queue);
}
# 获取指定命令队列的信息
cl_int xmrig::OclLib::getCommandQueueInfo(cl_command_queue command_queue, cl_command_queue_info param_name, size_t param_value_size, void *param_value, size_t *param_value_size_ret) noexcept
{
    return pGetCommandQueueInfo(command_queue, param_name, param_value_size, param_value, param_value_size_ret);
}


# 获取指定上下文的信息
cl_int xmrig::OclLib::getContextInfo(cl_context context, cl_context_info param_name, size_t param_value_size, void *param_value, size_t *param_value_size_ret) noexcept
{
    return pGetContextInfo(context, param_name, param_value_size, param_value, param_value_size_ret);
}


# 获取指定平台上指定设备类型的设备ID
cl_int xmrig::OclLib::getDeviceIDs(cl_platform_id platform, cl_device_type device_type, cl_uint num_entries, cl_device_id *devices, cl_uint *num_devices) noexcept
{
    assert(pGetDeviceIDs != nullptr);

    return pGetDeviceIDs(platform, device_type, num_entries, devices, num_devices);
}


# 获取指定设备的信息
cl_int xmrig::OclLib::getDeviceInfo(cl_device_id device, cl_device_info param_name, size_t param_value_size, void *param_value, size_t *param_value_size_ret) noexcept
{
    assert(pGetDeviceInfo != nullptr);

    const cl_int ret = pGetDeviceInfo(device, param_name, param_value_size, param_value, param_value_size_ret);
    if (ret != CL_SUCCESS && param_name != CL_DEVICE_BOARD_NAME_AMD) {
        LOG_ERR("Error %s when calling %s, param 0x%04x", OclError::toString(ret), kGetDeviceInfo, param_name);
    }

    return ret;
}


# 获取指定内核的信息
cl_int xmrig::OclLib::getKernelInfo(cl_kernel kernel, cl_kernel_info param_name, size_t param_value_size, void *param_value, size_t *param_value_size_ret) noexcept
{
    return pGetKernelInfo(kernel, param_name, param_value_size, param_value, param_value_size_ret);
}


# 获取指定内存对象的信息
cl_int xmrig::OclLib::getMemObjectInfo(cl_mem memobj, cl_mem_info param_name, size_t param_value_size, void *param_value, size_t *param_value_size_ret) noexcept
{
    return pGetMemObjectInfo(memobj, param_name, param_value_size, param_value, param_value_size_ret);
}
// 获取 OpenCL 平台的 ID
cl_int xmrig::OclLib::getPlatformIDs(cl_uint num_entries, cl_platform_id *platforms, cl_uint *num_platforms)
{
    // 确保 pGetPlatformIDs 函数指针不为空
    assert(pGetPlatformIDs != nullptr);

    // 调用 pGetPlatformIDs 函数获取平台 ID
    return pGetPlatformIDs(num_entries, platforms, num_platforms);
}


// 获取 OpenCL 平台的信息
cl_int xmrig::OclLib::getPlatformInfo(cl_platform_id platform, cl_platform_info param_name, size_t param_value_size, void *param_value, size_t *param_value_size_ret) noexcept
{
    // 确保 pGetPlatformInfo 函数指针不为空
    assert(pGetPlatformInfo != nullptr);

    // 调用 pGetPlatformInfo 函数获取平台信息
    return pGetPlatformInfo(platform, param_name, param_value_size, param_value, param_value_size_ret);
}


// 获取 OpenCL 程序的构建信息
cl_int xmrig::OclLib::getProgramBuildInfo(cl_program program, cl_device_id device, cl_program_build_info param_name, size_t param_value_size, void *param_value, size_t *param_value_size_ret) noexcept
{
    // 确保 pGetProgramBuildInfo 函数指针不为空
    assert(pGetProgramBuildInfo != nullptr);

    // 调用 pGetProgramBuildInfo 函数获取程序构建信息
    const cl_int ret = pGetProgramBuildInfo(program, device, param_name, param_value_size, param_value, param_value_size_ret);
    // 如果返回值不是 CL_SUCCESS，则记录错误信息
    if (ret != CL_SUCCESS) {
        LOG_ERR(kErrorTemplate, OclError::toString(ret), kGetProgramBuildInfo);
    }

    return ret;
}


// 获取 OpenCL 程序的信息
cl_int xmrig::OclLib::getProgramInfo(cl_program program, cl_program_info param_name, size_t param_value_size, void *param_value, size_t *param_value_size_ret)
{
    // 确保 pGetProgramInfo 函数指针不为空
    assert(pGetProgramInfo != nullptr);

    // 调用 pGetProgramInfo 函数获取程序信息
    const cl_int ret = pGetProgramInfo(program, param_name, param_value_size, param_value, param_value_size_ret);
    // 如果返回值不是 CL_SUCCESS，则记录错误信息
    if (ret != CL_SUCCESS) {
        LOG_ERR(kErrorTemplate, OclError::toString(ret), kGetProgramInfo);
    }

    return ret;
}


// 释放 OpenCL 命令队列
cl_int xmrig::OclLib::release(cl_command_queue command_queue) noexcept
{
    // 确保 pReleaseCommandQueue 和 pGetCommandQueueInfo 函数指针不为空
    assert(pReleaseCommandQueue != nullptr);
    assert(pGetCommandQueueInfo != nullptr);

    // 如果命令队列为空，则返回 CL_SUCCESS
    if (command_queue == nullptr) {
        return CL_SUCCESS;
    }

#   if defined(OCL_DEBUG_REFERENCE_COUNT)
    // 输出命令队列的引用计数信息
    LOG_REFS("%p %u ~queue", command_queue, getUint(command_queue, CL_QUEUE_REFERENCE_COUNT));
#   endif

    // 完成命令队列中的所有未完成的任务
    finish(command_queue);

    // 释放命令队列
    cl_int ret = pReleaseCommandQueue(command_queue);
    # 如果返回值不是 CL_SUCCESS，则表示出现错误
    if (ret != CL_SUCCESS) {
        # 记录错误信息到日志中
        LOG_ERR(kErrorTemplate, OclError::toString(ret), kReleaseCommandQueue);
    }
    # 返回结果
    return ret;
# 释放 OpenCL 上下文
cl_int xmrig::OclLib::release(cl_context context) noexcept
{
    # 确保释放上下文的函数指针不为空
    assert(pReleaseContext != nullptr);

    # 如果定义了 OCL_DEBUG_REFERENCE_COUNT，则记录上下文的引用计数
    # LOG_REFS("%p %u ~context", context, getUint(context, CL_CONTEXT_REFERENCE_COUNT));

    # 调用释放上下文的函数指针，并获取返回值
    const cl_int ret = pReleaseContext(context);
    # 如果返回值不是 CL_SUCCESS，则记录错误信息
    if (ret != CL_SUCCESS) {
        LOG_ERR(kErrorTemplate, OclError::toString(ret), kReleaseContext);
    }

    # 返回函数的返回值
    return ret;
}

# 释放 OpenCL 设备
cl_int xmrig::OclLib::release(cl_device_id id) noexcept
{
    # 确保释放设备的函数指针不为空
    assert(pReleaseDevice != nullptr);

    # 如果定义了 OCL_DEBUG_REFERENCE_COUNT，则记录设备的引用计数
    # LOG_REFS("%p %u ~device", id, getUint(id, CL_DEVICE_REFERENCE_COUNT));

    # 调用释放设备的函数指针，并获取返回值
    const cl_int ret = pReleaseDevice(id);
    # 如果返回值不是 CL_SUCCESS，则记录错误信息
    if (ret != CL_SUCCESS) {
        LOG_ERR(kErrorTemplate, OclError::toString(ret), kReleaseDevice);
    }

    # 返回函数的返回值
    return ret;
}

# 释放 OpenCL 内核
cl_int xmrig::OclLib::release(cl_kernel kernel) noexcept
{
    # 确保释放内核的函数指针不为空
    assert(pReleaseKernel != nullptr);

    # 如果内核为空，则返回 CL_SUCCESS
    if (kernel == nullptr) {
        return CL_SUCCESS;
    }

    # 如果定义了 OCL_DEBUG_REFERENCE_COUNT，则记录内核的引用计数
    # LOG_REFS("%p %u ~kernel %s", kernel, getUint(kernel, CL_KERNEL_REFERENCE_COUNT), getString(kernel, CL_KERNEL_FUNCTION_NAME).data());

    # 调用释放内核的函数指针，并获取返回值
    const cl_int ret = pReleaseKernel(kernel);
    # 如果返回值不是 CL_SUCCESS，则记录错误信息
    if (ret != CL_SUCCESS) {
        LOG_ERR(kErrorTemplate, OclError::toString(ret), kReleaseKernel);
    }

    # 返回函数的返回值
    return ret;
}

# 释放 OpenCL 内存对象
cl_int xmrig::OclLib::release(cl_mem mem_obj) noexcept
{
    # 确保释放内存对象的函数指针不为空
    assert(pReleaseMemObject != nullptr);

    # 如果内存对象为空，则返回 CL_SUCCESS
    if (mem_obj == nullptr) {
        return CL_SUCCESS;
    }

    # 如果定义了 OCL_DEBUG_REFERENCE_COUNT，则记录内存对象的引用计数
    # LOG_REFS("%p %u ~mem %zub", mem_obj, getUint(mem_obj, CL_MEM_REFERENCE_COUNT), getUlong(mem_obj, CL_MEM_SIZE));

    # 调用释放内存对象的函数指针，并获取返回值
    const cl_int ret = pReleaseMemObject(mem_obj);
    # 如果返回值不是 CL_SUCCESS，则记录错误信息
    if (ret != CL_SUCCESS) {
        LOG_ERR(kErrorTemplate, OclError::toString(ret), kReleaseMemObject);
    }

    # 返回函数的返回值
    return ret;
}

# 释放 OpenCL 程序
cl_int xmrig::OclLib::release(cl_program program) noexcept
{
    # 确保释放程序的函数指针不为空
    assert(pReleaseProgram != nullptr);
    # 如果程序指针为空，则返回 CL_SUCCESS
    if (program == nullptr) {
        return CL_SUCCESS;
    }
# 如果定义了 OCL_DEBUG_REFERENCE_COUNT，则记录程序的引用计数和内核名称
    LOG_REFS("%p %u ~program %s", program, getUint(program, CL_PROGRAM_REFERENCE_COUNT), getString(program, CL_PROGRAM_KERNEL_NAMES).data());
#   endif

    # 释放程序对象，并返回释放结果
    const cl_int ret = pReleaseProgram(program);
    # 如果释放失败，则记录错误信息
    if (ret != CL_SUCCESS) {
        LOG_ERR(kErrorTemplate, OclError::toString(ret), kReleaseProgram);
    }
    # 返回释放结果
    return ret;
}


# 设置内核参数
cl_int xmrig::OclLib::setKernelArg(cl_kernel kernel, cl_uint arg_index, size_t arg_size, const void *arg_value) noexcept
{
    # 断言内核设置函数不为空
    assert(pSetKernelArg != nullptr);
    # 调用内核设置函数，并返回设置结果
    return pSetKernelArg(kernel, arg_index, arg_size, arg_value);
}


# 卸载平台编译器
cl_int xmrig::OclLib::unloadPlatformCompiler(cl_platform_id platform) noexcept
{
    # 调用卸载平台编译器函数，并返回卸载结果
    return pUnloadPlatformCompiler(platform);
}


# 创建内核
cl_kernel xmrig::OclLib::createKernel(cl_program program, const char *kernel_name, cl_int *errcode_ret) noexcept
{
    # 断言创建内核函数不为空
    assert(pCreateKernel != nullptr);
    # 调用创建内核函数，并返回创建结果
    auto result = pCreateKernel(program, kernel_name, errcode_ret);
    # 如果创建失败，则记录错误信息并返回空指针
    if (*errcode_ret != CL_SUCCESS) {
        LOG_ERR("%s" RED(" error ") RED_BOLD("%s") RED(" when calling ") RED_BOLD("clCreateKernel") RED(" for kernel ") RED_BOLD("%s"),
                ocl_tag(), OclError::toString(*errcode_ret), kernel_name);
        return nullptr;
    }
    # 返回创建结果
    return result;
}


# 重载创建内核函数
cl_kernel xmrig::OclLib::createKernel(cl_program program, const char *kernel_name)
{
    # 调用带错误码返回值的创建内核函数，并处理错误情况
    cl_int ret = 0;
    cl_kernel kernel = createKernel(program, kernel_name, &ret);
    if (ret != CL_SUCCESS) {
        throw std::runtime_error(OclError::toString(ret));
    }
    # 返回创建结果
    return kernel;
}


# 创建缓冲区
cl_mem xmrig::OclLib::createBuffer(cl_context context, cl_mem_flags flags, size_t size, void *host_ptr)
{
    # 调用带错误码返回值的创建缓冲区函数，并处理错误情况
    cl_int ret = 0;
    cl_mem mem = createBuffer(context, flags, size, host_ptr, &ret);
    if (ret != CL_SUCCESS) {
        throw std::runtime_error(OclError::toString(ret));
    }
    # 返回创建结果
    return mem;
}


# 重载创建缓冲区函数
cl_mem xmrig::OclLib::createBuffer(cl_context context, cl_mem_flags flags, size_t size, void *host_ptr, cl_int *errcode_ret) noexcept
{
    # 断言创建缓冲区的指针不为空
    assert(pCreateBuffer != nullptr);

    # 调用创建缓冲区的函数，并将结果保存在result中
    auto result = pCreateBuffer(context, flags, size, host_ptr, errcode_ret);
    # 如果错误码不是CL_SUCCESS，则记录错误信息并返回空指针
    if (*errcode_ret != CL_SUCCESS) {
        LOG_ERR("%s" RED(" error ") RED_BOLD("%s") RED(" when calling ") RED_BOLD("%s") RED(" with buffer size ") RED_BOLD("%zu"),
                ocl_tag(), OclError::toString(*errcode_ret), kCreateBuffer, size);
        return nullptr;
    }

    # 返回创建缓冲区的结果
    return result;
# 创建一个子缓冲区，用于从父缓冲区中创建一个新的缓冲区
cl_mem xmrig::OclLib::createSubBuffer(cl_mem buffer, cl_mem_flags flags, size_t offset, size_t size, cl_int *errcode_ret) noexcept
{
    # 定义一个包含偏移量和大小的缓冲区区域
    const cl_buffer_region region = { offset, size };

    # 调用 OpenCL 函数创建子缓冲区
    auto result = pCreateSubBuffer(buffer, flags, CL_BUFFER_CREATE_TYPE_REGION, &region, errcode_ret);
    # 如果创建子缓冲区失败，则记录错误信息并返回空指针
    if (*errcode_ret != CL_SUCCESS) {
        LOG_ERR("%s" RED(" error ") RED_BOLD("%s") RED(" when calling ") RED_BOLD("%s") RED(" with offset ") RED_BOLD("%zu") RED(" and size ") RED_BOLD("%zu"),
                ocl_tag(), OclError::toString(*errcode_ret), kCreateSubBuffer, offset, size);

        return nullptr;
    }

    # 返回创建的子缓冲区
    return result;
}

# 重载的 createSubBuffer 函数，用于创建子缓冲区并处理错误
cl_mem xmrig::OclLib::createSubBuffer(cl_mem buffer, cl_mem_flags flags, size_t offset, size_t size)
{
    # 调用带错误码返回值的 createSubBuffer 函数
    cl_int ret = 0;
    cl_mem mem = createSubBuffer(buffer, flags, offset, size, &ret);
    # 如果创建子缓冲区失败，则抛出运行时错误
    if (ret != CL_SUCCESS) {
        throw std::runtime_error(OclError::toString(ret));
    }

    # 返回创建的子缓冲区
    return mem;
}

# 保留内存对象，增加其引用计数
cl_mem xmrig::OclLib::retain(cl_mem memobj) noexcept
{
    # 确保 pRetainMemObject 函数不为空
    assert(pRetainMemObject != nullptr);

    # 如果内存对象不为空，则调用保留内存对象的函数
    if (memobj != nullptr) {
        pRetainMemObject(memobj);
    }

    # 返回保留后的内存对象
    return memobj;
}

# 使用二进制数据创建程序对象
cl_program xmrig::OclLib::createProgramWithBinary(cl_context context, cl_uint num_devices, const cl_device_id *device_list, const size_t *lengths, const unsigned char **binaries, cl_int *binary_status, cl_int *errcode_ret) noexcept
{
    # 确保 pCreateProgramWithBinary 函数不为空
    assert(pCreateProgramWithBinary != nullptr);

    # 调用 OpenCL 函数创建程序对象
    auto result = pCreateProgramWithBinary(context, num_devices, device_list, lengths, binaries, binary_status, errcode_ret);
    # 如果创建程序对象失败，则记录错误信息并返回空指针
    if (*errcode_ret != CL_SUCCESS) {
        LOG_ERR(kErrorTemplate, OclError::toString(*errcode_ret), kCreateProgramWithBinary);

        return nullptr;
    }

    # 返回创建的程序对象
    return result;
}

# 使用源码字符串创建程序对象
cl_program xmrig::OclLib::createProgramWithSource(cl_context context, cl_uint count, const char **strings, const size_t *lengths, cl_int *errcode_ret) noexcept
{
    # 确保 pCreateProgramWithSource 函数不为空
    assert(pCreateProgramWithSource != nullptr);
    // 使用给定的上下文、源码字符串数组、字符串长度数组和错误码指针创建程序对象
    auto result = pCreateProgramWithSource(context, count, strings, lengths, errcode_ret);
    // 如果创建程序对象时出现错误
    if (*errcode_ret != CL_SUCCESS) {
        // 记录错误日志
        LOG_ERR(kErrorTemplate, OclError::toString(*errcode_ret), kCreateProgramWithSource);
        // 返回空指针
        return nullptr;
    }
    // 返回创建的程序对象
    return result;
// 保留 OpenCL 程序对象，增加其引用计数
cl_program xmrig::OclLib::retain(cl_program program) noexcept
{
    // 断言保留程序函数指针不为空
    assert(pRetainProgram != nullptr);

    // 如果程序对象不为空，则调用保留程序函数
    if (program != nullptr) {
        pRetainProgram(program);
    }

    // 返回程序对象
    return program;
}

// 获取平台数量
cl_uint xmrig::OclLib::getNumPlatforms() noexcept
{
    // 初始化平台数量和返回值
    cl_uint count   = 0;
    cl_int ret      = 0;

    // 获取平台数量，如果失败则记录错误日志
    if ((ret = OclLib::getPlatformIDs(0, nullptr, &count)) != CL_SUCCESS) {
        LOG_ERR("Error %s when calling clGetPlatformIDs for number of platforms.", OclError::toString(ret));
    }

    // 如果没有找到平台，则记录错误日志
    if (count == 0) {
        LOG_ERR("No OpenCL platform found.");
    }

    // 返回平台数量
    return count;
}

// 获取 cl_uint 类型的参数值
cl_uint xmrig::OclLib::getUint(cl_command_queue command_queue, cl_command_queue_info param_name, cl_uint defaultValue) noexcept
{
    // 获取命令队列信息
    getCommandQueueInfo(command_queue, param_name, sizeof(cl_uint), &defaultValue);

    // 返回参数值
    return defaultValue;
}

// 获取 cl_uint 类型的参数值
cl_uint xmrig::OclLib::getUint(cl_context context, cl_context_info param_name, cl_uint defaultValue) noexcept
{
    // 获取上下文信息
    getContextInfo(context, param_name, sizeof(cl_uint), &defaultValue);

    // 返回参数值
    return defaultValue;
}

// 获取 cl_uint 类型的参数值
cl_uint xmrig::OclLib::getUint(cl_device_id id, cl_device_info param, cl_uint defaultValue) noexcept
{
    // 获取设备信息
    getDeviceInfo(id, param, sizeof(cl_uint), &defaultValue);

    // 返回参数值
    return defaultValue;
}

// 获取 cl_uint 类型的参数值
cl_uint xmrig::OclLib::getUint(cl_kernel kernel, cl_kernel_info  param_name, cl_uint defaultValue) noexcept
{
    // 获取内核信息
    getKernelInfo(kernel, param_name, sizeof(cl_uint), &defaultValue);

    // 返回参数值
    return defaultValue;
}

// 获取 cl_uint 类型的参数值
cl_uint xmrig::OclLib::getUint(cl_mem memobj, cl_mem_info param_name, cl_uint defaultValue) noexcept
{
    // 获取内存对象信息
    getMemObjectInfo(memobj, param_name, sizeof(cl_uint), &defaultValue);

    // 返回参数值
    return defaultValue;
}

// 获取 cl_uint 类型的参数值
cl_uint xmrig::OclLib::getUint(cl_program program, cl_program_info param, cl_uint defaultValue) noexcept
{
    // 获取程序信息
    getProgramInfo(program, param, sizeof(cl_uint), &defaultValue);

    // 返回参数值
    return defaultValue;
}

// 获取 cl_ulong 类型的参数值
cl_ulong xmrig::OclLib::getUlong(cl_device_id id, cl_device_info param, cl_ulong defaultValue) noexcept
{
    # 调用函数获取设备信息，传入设备ID、参数、参数大小和默认值的地址
    getDeviceInfo(id, param, sizeof(cl_ulong), &defaultValue);
    
    # 返回默认值
    return defaultValue;
# 获取指定内存对象的参数信息，如果获取失败则返回默认值
cl_ulong xmrig::OclLib::getUlong(cl_mem memobj, cl_mem_info param_name, cl_ulong defaultValue) noexcept
{
    getMemObjectInfo(memobj, param_name, sizeof(cl_ulong), &defaultValue);
    # 返回获取到的参数值或默认值
    return defaultValue;
}


# 获取所有平台的 ID
std::vector<cl_platform_id> xmrig::OclLib::getPlatformIDs() noexcept
{
    # 获取平台数量
    const uint32_t count = getNumPlatforms();
    # 创建存储平台 ID 的 vector
    std::vector<cl_platform_id> platforms(count);

    # 如果有平台存在，则获取平台 ID
    if (count) {
        getPlatformIDs(count, platforms.data(), nullptr);
    }

    # 返回平台 ID 的 vector
    return platforms;
}


# 获取程序构建日志
xmrig::String xmrig::OclLib::getProgramBuildLog(cl_program program, cl_device_id device) noexcept
{
    # 获取构建日志的大小
    size_t size = 0;
    if (getProgramBuildInfo(program, device, CL_PROGRAM_BUILD_LOG, 0, nullptr, &size) != CL_SUCCESS) {
        return String();
    }

    # 分配内存来存储构建日志
    char* log = nullptr;
    try {
        log = new char[size + 1]();
    }
    catch (...) {
        return String();
    }

    # 获取构建日志
    if (getProgramBuildInfo(program, device, CL_PROGRAM_BUILD_LOG, size, log, nullptr) != CL_SUCCESS) {
        delete [] log;
        return String();
    }

    # 返回构建日志
    return log;
}


# 获取设备信息的字符串形式
xmrig::String xmrig::OclLib::getString(cl_device_id id, cl_device_info param) noexcept
{
    return getOclString(OclLib::getDeviceInfo, id, param);
}


# 获取内核信息的字符串形式
xmrig::String xmrig::OclLib::getString(cl_kernel kernel, cl_kernel_info param_name) noexcept
{
    return getOclString(OclLib::getKernelInfo, kernel, param_name);
}


# 获取平台信息的字符串形式
xmrig::String xmrig::OclLib::getString(cl_platform_id platform, cl_platform_info param_name) noexcept
{
    return getOclString(OclLib::getPlatformInfo, platform, param_name);
}


# 获取程序信息的字符串形式
xmrig::String xmrig::OclLib::getString(cl_program program, cl_program_info param_name) noexcept
{
    return getOclString(OclLib::getProgramInfo, program, param_name);
}
```