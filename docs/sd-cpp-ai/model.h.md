# `stable-diffusion.cpp\model.h`

```
#ifndef __MODEL_H__
#define __MODEL_H__

#include <functional>  // 包含函数对象相关的头文件
#include <map>  // 包含映射容器相关的头文件
#include <memory>  // 包含智能指针相关的头文件
#include <set>  // 包含集合容器相关的头文件
#include <string>  // 包含字符串相关的头文件
#include <vector>  // 包含向量容器相关的头文件

#include "ggml/ggml-backend.h"  // 包含 ggml-backend.h 头文件
#include "ggml/ggml.h"  // 包含 ggml.h 头文件
#include "json.hpp"  // 包含 json.hpp 头文件
#include "zip.h"  // 包含 zip.h 头文件

enum SDVersion {  // 定义枚举类型 SDVersion
    VERSION_1_x,  // 版本 1.x
    VERSION_2_x,  // 版本 2.x
    VERSION_XL,  // 版本 XL
    VERSION_COUNT,  // 版本计数
};

struct TensorStorage {  // 定义结构体 TensorStorage
    std::string name;  // 名称
    ggml_type type = GGML_TYPE_F32;  // 类型，默认为 GGML_TYPE_F32
    bool is_bf16   = false;  // 是否为 bf16 类型，默认为 false
    int64_t ne[4]  = {1, 1, 1, 1};  // 维度数组，默认为 {1, 1, 1, 1}
    int n_dims     = 0;  // 维度数量，默认为 0

    size_t file_index = 0;  // 文件索引，默认为 0
    int index_in_zip  = -1;  // 在 ZIP 文件中的索引，默认为 -1（大于等于 0 表示存储在 ZIP 文件中）
    size_t offset     = 0;   // 文件中的偏移量，默认为 0

    TensorStorage() = default;  // 默认构造函数

    TensorStorage(const std::string& name, ggml_type type, int64_t* ne, int n_dims, size_t file_index, size_t offset = 0)
        : name(name), type(type), n_dims(n_dims), file_index(file_index), offset(offset) {  // 自定义构造函数
        for (int i = 0; i < n_dims; i++) {
            this->ne[i] = ne[i];
        }
    }

    int64_t nelements() const {  // 计算元素总数
        return ne[0] * ne[1] * ne[2] * ne[3];
    }

    int64_t nbytes() const {  // 计算字节数
        return nelements() * ggml_type_size(type) / ggml_blck_size(type);
    }

    int64_t nbytes_to_read() const {  // 计算需要读取的字节数
        if (is_bf16) {
            return nbytes() / 2;
        } else {
            return nbytes();
        }
    }

    void unsqueeze() {  // 扩展维度
        if (n_dims == 2) {
            n_dims = 4;
            ne[3]  = ne[1];
            ne[2]  = ne[0];
            ne[1]  = 1;
            ne[0]  = 1;
        }
    }
    // 将数据分成 n 个块，返回块的向量
    std::vector<TensorStorage> chunk(size_t n) {
        // 创建存储块的向量
        std::vector<TensorStorage> chunks;
        // 计算每个块的大小
        size_t chunk_size = nbytes_to_read() / n;
        // 反转 ne 数组
        reverse_ne();
        // 遍历 n 次，每次创建一个新的块并添加到向量中
        for (int i = 0; i < n; i++) {
            // 复制当前对象作为新的块
            TensorStorage chunk_i = *this;
            // 更新新块的 ne[0] 和 offset
            chunk_i.ne[0]         = ne[0] / n;
            chunk_i.offset        = offset + i * chunk_size;
            // 反转新块的 ne 数组
            chunk_i.reverse_ne();
            // 将新块添加到向量中
            chunks.push_back(chunk_i);
        }
        // 再次反转 ne 数组
        reverse_ne();
        // 返回存储块的向量
        return chunks;
    }

    // 反转 ne 数组
    void reverse_ne() {
        // 创建新的 ne 数组
        int64_t new_ne[4] = {1, 1, 1, 1};
        // 将 ne 数组反转后的值存储到新数组中
        for (int i = 0; i < n_dims; i++) {
            new_ne[i] = ne[n_dims - 1 - i];
        }
        // 将新数组的值复制回原来的 ne 数组
        for (int i = 0; i < n_dims; i++) {
            ne[i] = new_ne[i];
        }
    }
// 结束类定义
};

// 定义回调函数类型，用于创建新的张量
typedef std::function<bool(const TensorStorage&, ggml_tensor**)> on_new_tensor_cb_t;

// 模型加载器类
class ModelLoader {
protected:
    // 存储文件路径的向量
    std::vector<std::string> file_paths_;
    // 存储张量存储的向量
    std::vector<TensorStorage> tensor_storages;

    // 解析数据 pkl 文件
    bool parse_data_pkl(uint8_t* buffer,
                        size_t buffer_size,
                        zip_t* zip,
                        std::string dir,
                        size_t file_index,
                        const std::string& prefix);

    // 从 gguf 文件初始化模型
    bool init_from_gguf_file(const std::string& file_path, const std::string& prefix = "");
    // 从 safetensors 文件初始化模型
    bool init_from_safetensors_file(const std::string& file_path, const std::string& prefix = "");
    // 从 ckpt 文件初始化模型
    bool init_from_ckpt_file(const std::string& file_path, const std::string& prefix = "");
    // 从 diffusers 文件初始化模型
    bool init_from_diffusers_file(const std::string& file_path, const std::string& prefix = "");

public:
    // 从文件初始化模型
    bool init_from_file(const std::string& file_path, const std::string& prefix = "");
    // 获取 SD 版本
    SDVersion get_sd_version();
    // 获取 SD 权重类型
    ggml_type get_sd_wtype();
    // 加载合并
    std::string load_merges();
    // 加载张量
    bool load_tensors(on_new_tensor_cb_t on_new_tensor_cb, ggml_backend_t backend);
    // 加载张量
    bool load_tensors(std::map<std::string, struct ggml_tensor*>& tensors,
                      ggml_backend_t backend,
                      std::set<std::string> ignore_tensors = {});
    // 保存到 gguf 文件
    bool save_to_gguf_file(const std::string& file_path, ggml_type type);
    // 计算内存大小
    int64_t cal_mem_size(ggml_backend_t backend, ggml_type type = GGML_TYPE_COUNT);
    // 默认析构函数
    ~ModelLoader() = default;
};
// 结束宏定义
#endif  // __MODEL_H__
```