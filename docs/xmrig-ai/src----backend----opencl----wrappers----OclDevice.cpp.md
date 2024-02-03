# `xmrig\src\backend\opencl\wrappers\OclDevice.cpp`

```cpp
/* XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改
 *   根据由自由软件基金会发布的GNU通用公共许可证的条款，
 *   许可证的版本为3，或者（根据您的选择）任何以后的版本。
 *
 *   本程序是希望它有用，
 *   但没有任何保证；甚至没有暗示的保证。
 *   有关更多详细信息，请参见GNU通用公共许可证。
 *
 *   您应该已经收到了GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include "backend/opencl/wrappers/OclDevice.h"
#include "3rdparty/rapidjson/document.h"
#include "backend/opencl/OclGenerator.h"
#include "backend/opencl/OclThreads.h"
#include "backend/opencl/wrappers/OclLib.h"
#include "base/io/log/Log.h"


#ifdef XMRIG_FEATURE_ADL
#   include "backend/opencl/wrappers/AdlLib.h"
#endif


#include <algorithm>


// NOLINTNEXTLINE(modernize-use-using)
typedef union
{
    struct { cl_uint type; cl_uint data[5]; } raw;
    struct { cl_uint type; cl_char unused[17]; cl_char bus; cl_char device; cl_char function; } pcie;
} topology_amd;


namespace xmrig {


#ifdef XMRIG_ALGO_RANDOMX
extern bool ocl_generic_rx_generator(const OclDevice &device, const Algorithm &algorithm, OclThreads &threads);
#endif

#ifdef XMRIG_ALGO_KAWPOW
extern bool ocl_generic_kawpow_generator(const OclDevice& device, const Algorithm& algorithm, OclThreads& threads);
#endif

extern bool ocl_vega_cn_generator(const OclDevice &device, const Algorithm &algorithm, OclThreads &threads);
extern bool ocl_generic_cn_generator(const OclDevice &device, const Algorithm &algorithm, OclThreads &threads);


static ocl_gen_config_fun generators[] = {
# 如果定义了 XMRIG_ALGO_RANDOMX，则使用 ocl_generic_rx_generator
    ocl_generic_rx_generator,
# 如果定义了 XMRIG_ALGO_KAWPOW，则使用 ocl_generic_kawpow_generator
    ocl_generic_kawpow_generator,
# 使用 ocl_vega_cn_generator
    ocl_vega_cn_generator,
# 使用 ocl_generic_cn_generator
    ocl_generic_cn_generator
};


# 根据平台供应商和扩展名获取平台供应商 ID
static OclVendor getPlatformVendorId(const String &vendor, const String &extensions)
{
    # 如果扩展名包含 "cl_amd_" 或供应商包含 "Advanced Micro Devices" 或 "AMD"，则返回 OCL_VENDOR_AMD
    if (extensions.contains("cl_amd_") || vendor.contains("Advanced Micro Devices") || vendor.contains("AMD")) {
        return OCL_VENDOR_AMD;
    }
    # 如果扩展名包含 "cl_nv_" 或供应商包含 "NVIDIA"，则返回 OCL_VENDOR_NVIDIA
    if (extensions.contains("cl_nv_") || vendor.contains("NVIDIA")) {
        return OCL_VENDOR_NVIDIA;
    }
    # 如果扩展名包含 "cl_intel_" 或供应商包含 "Intel"，则返回 OCL_VENDOR_INTEL
    if (extensions.contains("cl_intel_") || vendor.contains("Intel")) {
        return OCL_VENDOR_INTEL;
    }
    # 如果扩展名包含 "cl_APPLE_" 或供应商包含 "Apple"，则返回 OCL_VENDOR_APPLE
    if (extensions.contains("cl_APPLE_") || vendor.contains("Apple")) {
        return OCL_VENDOR_APPLE;
    }
    # 否则返回 OCL_VENDOR_UNKNOWN
    return OCL_VENDOR_UNKNOWN;
}


# 根据供应商获取供应商 ID
static OclVendor getVendorId(const String &vendor)
{
    # 如果供应商包含 "Advanced Micro Devices" 或 "AMD"，则返回 OCL_VENDOR_AMD
    if (vendor.contains("Advanced Micro Devices") || vendor.contains("AMD")) {
        return OCL_VENDOR_AMD;
    }
    # 如果供应商包含 "NVIDIA"，则返回 OCL_VENDOR_NVIDIA
    if (vendor.contains("NVIDIA")) {
        return OCL_VENDOR_NVIDIA;
    }
    # 如果供应商包含 "Intel"，则返回 OCL_VENDOR_INTEL
    if (vendor.contains("Intel")) {
        return OCL_VENDOR_INTEL;
    }
    # 如果供应商包含 "Apple"，则返回 OCL_VENDOR_APPLE
    if (vendor.contains("Apple")) {
        return OCL_VENDOR_APPLE;
    }
    # 否则返回 OCL_VENDOR_UNKNOWN
    return OCL_VENDOR_UNKNOWN;
}


# 根据名称和平台供应商 ID 获取设备类型
static OclDevice::Type getType(const String &name, const OclVendor platformVendorId)
    # 如果平台供应商 ID 为 OCL_VENDOR_APPLE
    if (platformVendorId == OCL_VENDOR_APPLE) {
        # 苹果平台：使用产品名称，而不是 gfx# 或代号
        if (name.contains("AMD Radeon")) {
            # 如果名称包含 "AMD Radeon"，并且包含以下任一字符串，则返回相应的设备类型
            if (name.contains(" 450 ") ||
                name.contains(" 455 ") ||
                name.contains(" 460 ")) {
                return OclDevice::Baffin;
            }

            if (name.contains(" 555 ") || name.contains(" 555X ") ||
                name.contains(" 560 ") || name.contains(" 560X ") ||
                name.contains(" 570 ") || name.contains(" 570X ") ||
                name.contains(" 575 ") || name.contains(" 575X ")) {
                return OclDevice::Polaris;
            }

            if (name.contains(" 580 ") || name.contains(" 580X ")) {
                return OclDevice::Ellesmere;
            }

            if (name.contains(" Vega ")) {
                # 如果名称包含 "Vega"，并且包含以下任一字符串，则返回相应的 Vega 设备类型
                if (name.contains(" 48 ") ||
                    name.contains(" 56 ") ||
                    name.contains(" 64 ") ||
                    name.contains(" 64X ")) {
                    return OclDevice::Vega_10;
                }
                if (name.contains(" 16 ") ||
                    name.contains(" 20 ") ||
                    name.contains(" II ")) {
                    return OclDevice::Vega_20;
                }
            }

            if (name.contains(" 5700 ") || name.contains(" W5700X ")) {
                return OclDevice::Navi_10;
            }

            if (name.contains(" 5600 ") || name.contains(" 5600M ")) {
                return OclDevice::Navi_12;
            }

            if (name.contains(" 5300 ") || name.contains(" 5300M ") ||
                name.contains(" 5500 ") || name.contains(" 5500M ")) {
                return OclDevice::Navi_14;
            }

            if (name.contains(" W6800 ") || name.contains(" W6900X ")) {
                return OclDevice::Navi_21;
            }
        }
    }

    # 如果名称为 "gfx900" 或 "gfx901"，则返回 Vega_10 设备类型
    if (name == "gfx900" || name == "gfx901") {
        return OclDevice::Vega_10;
    }
    # 如果设备名为 "gfx902" 或 "gfx903"，返回 OclDevice::Raven
    if (name == "gfx902" || name == "gfx903") {
        return OclDevice::Raven;
    }
    # 如果设备名为 "gfx906" 或 "gfx907"，返回 OclDevice::Vega_20
    if (name == "gfx906" || name == "gfx907") {
        return OclDevice::Vega_20;
    }
    # 如果设备名为 "gfx1010"，返回 OclDevice::Navi_10
    if (name == "gfx1010") {
        return OclDevice::Navi_10;
    }
    # 如果设备名为 "gfx1011"，返回 OclDevice::Navi_12
    if (name == "gfx1011") {
        return OclDevice::Navi_12;
    }
    # 如果设备名为 "gfx1012"，返回 OclDevice::Navi_14
    if (name == "gfx1012") {
        return OclDevice::Navi_14;
    }
    # 如果设备名为 "gfx1030"，返回 OclDevice::Navi_21
    if (name == "gfx1030") {
        return OclDevice::Navi_21;
    }
    # 如果设备名为 "gfx804"，返回 OclDevice::Lexa
    if (name == "gfx804") {
        return OclDevice::Lexa;
    }
    # 如果设备名为 "Baffin"，返回 OclDevice::Baffin
    if (name == "Baffin") {
        return OclDevice::Baffin;
    }
    # 如果设备名包含 "Ellesmere"，返回 OclDevice::Ellesmere
    if (name.contains("Ellesmere")) {
        return OclDevice::Ellesmere;
    }
    # 如果设备名为 "gfx803" 或 包含 "polaris"，返回 OclDevice::Polaris
    if (name == "gfx803" || name.contains("polaris")) {
        return OclDevice::Polaris;
    }
    # 默认情况下返回 OclDevice::Unknown
    return OclDevice::Unknown;
} // namespace xmrig

// 构造函数，初始化 OpenCL 设备对象
xmrig::OclDevice::OclDevice(uint32_t index, cl_device_id id, cl_platform_id platform) :
    m_id(id), // 初始化设备 ID
    m_platform(platform), // 初始化平台 ID
    m_platformVendor(OclLib::getString(platform, CL_PLATFORM_VENDOR)), // 获取平台供应商信息
    m_name(OclLib::getString(id, CL_DEVICE_NAME)), // 获取设备名称
    m_vendor(OclLib::getString(id, CL_DEVICE_VENDOR)), // 获取设备供应商信息
    m_extensions(OclLib::getString(id, CL_DEVICE_EXTENSIONS)), // 获取设备支持的扩展信息
    m_maxMemoryAlloc(OclLib::getUlong(id, CL_DEVICE_MAX_MEM_ALLOC_SIZE)), // 获取设备最大内存分配
    m_globalMemory(OclLib::getUlong(id, CL_DEVICE_GLOBAL_MEM_SIZE)), // 获取设备全局内存大小
    m_computeUnits(OclLib::getUint(id, CL_DEVICE_MAX_COMPUTE_UNITS, 1)), // 获取设备计算单元数量
    m_index(index) // 初始化设备索引
{
    m_vendorId  = getVendorId(m_vendor); // 获取设备供应商 ID
    m_platformVendorId = getPlatformVendorId(m_platformVendor, m_extensions); // 获取平台供应商 ID
    m_type      = getType(m_name, m_platformVendorId); // 获取设备类型

    // 检查是否支持 AMD 设备属性查询
    if (m_extensions.contains("cl_amd_device_attribute_query")) {
        topology_amd topology;

        // 获取 AMD 设备拓扑信息
        if (OclLib::getDeviceInfo(id, CL_DEVICE_TOPOLOGY_AMD, sizeof(topology), &topology, nullptr) == CL_SUCCESS && topology.raw.type == CL_DEVICE_TOPOLOGY_TYPE_PCIE_AMD) {
            m_topology = PciTopology(static_cast<uint32_t>(topology.pcie.bus), static_cast<uint32_t>(topology.pcie.device), static_cast<uint32_t>(topology.pcie.function)); // 设置设备拓扑信息
        }
        m_board = OclLib::getString(id, CL_DEVICE_BOARD_NAME_AMD); // 获取 AMD 设备板卡名称
    }
    // 检查是否支持 NVIDIA 设备属性查询
    else if (m_extensions.contains("cl_nv_device_attribute_query")) {
        cl_uint bus = 0;
        if (OclLib::getDeviceInfo(id, CL_DEVICE_PCI_BUS_ID_NV, sizeof (bus), &bus, nullptr) == CL_SUCCESS) {
            cl_uint slot  = OclLib::getUint(id, CL_DEVICE_PCI_SLOT_ID_NV);
            m_topology = PciTopology(bus, (slot >> 3) & 0xff, slot & 7); // 设置设备拓扑信息
        }
    }
}

// 获取可打印的设备名称
xmrig::String xmrig::OclDevice::printableName() const
{
    const size_t size = m_board.size() + m_name.size() + 64; // 计算缓冲区大小
    char *buf         = new char[size](); // 创建缓冲区

    if (m_board.isNull()) {
        snprintf(buf, size, GREEN_BOLD("%s"), m_name.data()); // 格式化设备名称
    }
}
    # 如果条件不满足，则执行以下代码块
    else {
        # 使用 snprintf 函数将格式化后的字符串存储到 buf 中，包括 m_board 和 m_name 的值
        snprintf(buf, size, GREEN_BOLD("%s") " (" CYAN_BOLD("%s") ")", m_board.data(), m_name.data());
    }
    # 返回 buf
    return buf;
// 返回设备的时钟频率
uint32_t xmrig::OclDevice::clock() const
{
    return OclLib::getUint(id(), CL_DEVICE_MAX_CLOCK_FREQUENCY);
}

// 根据给定算法和线程生成器，生成 OpenCL 线程
void xmrig::OclDevice::generate(const Algorithm &algorithm, OclThreads &threads) const
{
    // 遍历生成器列表，使用设备、算法和线程生成器
    for (auto fn : generators) {
        // 如果生成成功，则返回
        if (fn(*this, algorithm, threads)) {
            return;
        }
    }
}

#ifdef XMRIG_FEATURE_API
// 将设备信息转换为 JSON 格式
void xmrig::OclDevice::toJSON(rapidjson::Value &out, rapidjson::Document &doc) const
{
    using namespace rapidjson;
    auto &allocator = doc.GetAllocator();

    // 添加设备板信息到 JSON 对象
    out.AddMember("board",       board().toJSON(doc), allocator);
    // 添加设备名称信息到 JSON 对象
    out.AddMember("name",        name().toJSON(doc), allocator);
    // 添加设备总线 ID 信息到 JSON 对象
    out.AddMember("bus_id",      topology().toString().toJSON(doc), allocator);
    // 添加设备计算单元数量到 JSON 对象
    out.AddMember("cu",          computeUnits(), allocator);
    // 添加设备全局内存大小到 JSON 对象
    out.AddMember("global_mem",  static_cast<uint64_t>(globalMemSize()), allocator);

#   ifdef XMRIG_FEATURE_ADL
    // 如果 ADL 库准备就绪
    if (AdlLib::isReady()) {
        // 获取设备健康信息
        auto data = AdlLib::health(*this);

        // 创建健康信息的 JSON 对象
        Value health(kObjectType);
        // 添加温度信息到健康信息 JSON 对象
        health.AddMember("temperature", data.temperature, allocator);
        // 添加功耗信息到健康信息 JSON 对象
        health.AddMember("power",       data.power, allocator);
        // 添加时钟频率信息到健康信息 JSON 对象
        health.AddMember("clock",       data.clock, allocator);
        // 添加内存时钟频率信息到健康信息 JSON 对象
        health.AddMember("mem_clock",   data.memClock, allocator);
        // 添加转速信息到健康信息 JSON 对象
        health.AddMember("rpm",         data.rpm, allocator);

        // 添加健康信息到 JSON 对象
        out.AddMember("health", health, allocator);
    }
#   endif
}
#endif
```