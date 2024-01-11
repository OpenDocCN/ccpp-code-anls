# `xmrig\src\backend\cuda\wrappers\CudaDevice.h`

```
// 包含版权声明和许可证信息
// 定义 XMRig_CUDADEVICE_H 宏，用于条件编译
// 包含所需的头文件和命名空间
// 声明结构体和指针类型别名
// 声明 Algorithm 和 CudaThreads 类
// 声明 CudaDevice 类
// 禁用默认构造函数和拷贝构造函数
// 声明移动构造函数
// 声明带参数的构造函数和析构函数
// 声明获取空闲内存大小、全局内存大小、时钟频率、计算能力、内存时钟频率和 SMX 数量的成员函数
    // 生成加密算法的CUDA线程
    void generate(const Algorithm &algorithm, CudaThreads &threads) const;

    // 检查上下文是否有效
    inline bool isValid() const                     { return m_ctx != nullptr; }
    // 返回PCI拓扑结构
    inline const PciTopology &topology() const      { return m_topology; }
    // 返回设备名称
    inline const String &name() const               { return m_name; }
    // 返回设备架构
    inline uint32_t arch() const                    { return (computeCapability(true) * 10) + computeCapability(false); }
    // 返回设备索引
    inline uint32_t index() const                   { return m_index; }
// 如果定义了 XMRIG_FEATURE_NVML，则定义内联函数 nvmlDevice，返回 m_nvmlDevice
inline nvmlDevice_t nvmlDevice() const          { return m_nvmlDevice; }
// 如果定义了 XMRIG_FEATURE_NVML，则定义内联函数 setNvmlDevice，设置 m_nvmlDevice
inline void setNvmlDevice(nvmlDevice_t device)  { m_nvmlDevice = device; }

// 如果定义了 XMRIG_FEATURE_API，则定义函数toJSON，将设备信息转换为 JSON 格式
void toJSON(rapidjson::Value &out, rapidjson::Document &doc) const;

// 禁用赋值运算符重载
CudaDevice &operator=(const CudaDevice &other)  = delete;
CudaDevice &operator=(CudaDevice &&other)       = delete;

// 私有成员变量
private:
    const uint32_t m_index          = 0; // 设备索引，默认为 0
    nvid_ctx *m_ctx                 = nullptr; // 上下文指针，默认为空指针
    PciTopology m_topology; // PCI 拓扑结构
    String m_name; // 设备名称

// 如果定义了 XMRIG_FEATURE_NVML，则定义成员变量 m_nvmlDevice，默认为空指针
#   ifdef XMRIG_FEATURE_NVML
    nvmlDevice_t m_nvmlDevice       = nullptr;
#   endif
};

// 命名空间结束
} // namespace xmrig

// 头文件结束
#endif /* XMRIG_CUDADEVICE_H */
```