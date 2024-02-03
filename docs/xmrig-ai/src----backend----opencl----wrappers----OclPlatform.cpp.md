# `xmrig\src\backend\opencl\wrappers\OclPlatform.cpp`

```cpp
// 包含一系列版权声明
// 包含 GNU 通用公共许可证的条款
// 定义了一些头文件

#include "backend/opencl/wrappers/OclPlatform.h"
#include "3rdparty/rapidjson/document.h"
#include "backend/opencl/wrappers/OclLib.h"

// 定义了一个返回 OclPlatform 对象的函数
std::vector<xmrig::OclPlatform> xmrig::OclPlatform::get()
{
    // 获取 OpenCL 平台的 ID
    const std::vector<cl_platform_id> platforms = OclLib::getPlatformIDs();
    // 创建一个 OclPlatform 对象的向量
    std::vector<OclPlatform> out;
    // 如果没有平台，返回空向量
    if (platforms.empty()) {
        return out;
    }

    // 预留足够的空间
    out.reserve(platforms.size());

    // 遍历平台，创建 OclPlatform 对象并添加到向量中
    for (size_t i = 0; i < platforms.size(); i++) {
        out.emplace_back(i, platforms[i]);
    }

    // 返回 OclPlatform 对象的向量
    return out;
}

// 打印平台数量
void xmrig::OclPlatform::print()
{
    // 获取平台对象
    const auto platforms = OclPlatform::get();
    // 打印平台数量
    printf("%-28s%zu\n\n", "Number of platforms:", platforms.size());
}
    // 遍历 platforms 容器中的每个 platform 对象
    for (const auto &platform : platforms) {
        // 打印 platform 的索引
        printf("  %-26s%zu\n",  "Index:",       platform.index());
        // 打印 platform 的配置文件
        printf("  %-26s%s\n",   "Profile:",     platform.profile().data());
        // 打印 platform 的版本
        printf("  %-26s%s\n",   "Version:",     platform.version().data());
        // 打印 platform 的名称
        printf("  %-26s%s\n",   "Name:",        platform.name().data());
        // 打印 platform 的供应商
        printf("  %-26s%s\n",   "Vendor:",      platform.vendor().data());
        // 打印 platform 的扩展
        printf("  %-26s%s\n\n", "Extensions:",  platform.extensions().data());
    }
}

rapidjson::Value xmrig::OclPlatform::toJSON(rapidjson::Document &doc) const
{
    using namespace rapidjson;
    auto &allocator = doc.GetAllocator();

    // 如果平台无效，则返回空值
    if (!isValid()) {
        return Value(kNullType);
    }

    // 创建一个 JSON 对象
    Value out(kObjectType);
    // 添加平台索引到 JSON 对象
    out.AddMember("index",      static_cast<uint64_t>(index()), allocator);
    // 添加平台配置信息到 JSON 对象
    out.AddMember("profile",    profile().toJSON(doc), allocator);
    // 添加平台版本信息到 JSON 对象
    out.AddMember("version",    version().toJSON(doc), allocator);
    // 添加平台名称信息到 JSON 对象
    out.AddMember("name",       name().toJSON(doc), allocator);
    // 添加平台供应商信息到 JSON 对象
    out.AddMember("vendor",     vendor().toJSON(doc), allocator);
    // 添加平台扩展信息到 JSON 对象
    out.AddMember("extensions", extensions().toJSON(doc), allocator);

    return out;
}


std::vector<xmrig::OclDevice> xmrig::OclPlatform::devices() const
{
    // 创建一个空的设备列表
    std::vector<OclDevice> out;
    // 如果平台无效，则返回空的设备列表
    if (!isValid()) {
        return out;
    }

    // 获取平台上的 GPU 设备数量
    cl_uint num_devices = 0;
    OclLib::getDeviceIDs(id(), CL_DEVICE_TYPE_GPU, 0, nullptr, &num_devices);
    // 如果没有 GPU 设备，则返回空的设备列表
    if (num_devices == 0) {
        return out;
    }

    // 预留空间以容纳设备列表
    out.reserve(num_devices);
    // 获取平台上的 GPU 设备列表
    std::vector<cl_device_id> devices(num_devices);
    OclLib::getDeviceIDs(id(), CL_DEVICE_TYPE_GPU, num_devices, devices.data(), nullptr);

    // 遍历设备列表，创建 OclDevice 对象并添加到设备列表中
    for (size_t i = 0; i < devices.size(); ++i) {
        out.emplace_back(i, devices[i], id());
    }

    return out;
}


xmrig::String xmrig::OclPlatform::extensions() const
{
    // 获取平台的扩展信息
    return OclLib::getString(id(), CL_PLATFORM_EXTENSIONS);
}


xmrig::String xmrig::OclPlatform::name() const
{
    // 获取平台的名称
    return OclLib::getString(id(), CL_PLATFORM_NAME);
}


xmrig::String xmrig::OclPlatform::profile() const
{
    // 获取平台的配置信息
    return OclLib::getString(id(), CL_PLATFORM_PROFILE);
}


xmrig::String xmrig::OclPlatform::vendor() const
{
    // 获取平台的供应商信息
    return OclLib::getString(id(), CL_PLATFORM_VENDOR);
}


xmrig::String xmrig::OclPlatform::version() const
{
    // 获取平台的版本信息
    return OclLib::getString(id(), CL_PLATFORM_VERSION);
}
```