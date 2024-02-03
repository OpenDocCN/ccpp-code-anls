# `chatglm.cpp\chatglm.cpp`

```cpp
// 包含 chatglm.h 头文件
#include "chatglm.h"
// 包含一些标准库头文件
#include <algorithm>
#include <codecvt>
#include <cstring>
#include <fcntl.h>
#include <fstream>
#include <functional>
#include <iomanip>
#include <iostream>
#include <locale>
#include <numeric>
#include <random>
#include <regex>
#include <string>
#include <sys/stat.h>
#include <thread>

// 根据条件编译包含一些特定的头文件
#ifdef __has_include
#if __has_include(<unistd.h>)
#include <unistd.h>
#if defined(_POSIX_MAPPED_FILES)
#include <sys/mman.h>
#endif
#if defined(_POSIX_MEMLOCK_RANGE)
#include <sys/resource.h>
#endif
#endif
#endif

// 如果是 Windows 系统，定义一些宏和包含一些头文件
#if defined(_WIN32)
#define WIN32_LEAN_AND_MEAN
#ifndef NOMINMAX
#define NOMINMAX
#endif
#include <io.h>
#include <stdio.h>
#include <windows.h>
#endif

// 如果定义了 GGML_USE_CUBLAS，包含 ggml-cuda.h 头文件
#ifdef GGML_USE_CUBLAS
#include <ggml-cuda.h>
#endif

// chatglm 命名空间
namespace chatglm {

// 将 tensor 的形状转换为字符串
static std::string shape_to_string(ggml_tensor *tensor) {
    std::ostringstream oss;
    oss << '[';
    for (int i = tensor->n_dims - 1; i >= 0; i--) {
        oss << tensor->ne[i] << (i > 0 ? ", " : "");
    }
    oss << ']';
    return oss.str();
}

// 将 tensor 的步幅转换为字符串
static std::string strides_to_string(ggml_tensor *tensor) {
    std::ostringstream oss;
    oss << '[';
    for (int i = tensor->n_dims - 1; i >= 0; i--) {
        oss << tensor->nb[i] << (i > 0 ? ", " : "");
    }
    oss << ']';
    return oss.str();
}

// 将 ggml_tensor 转换为字符串，可选择是否包含数据
std::string to_string(ggml_tensor *tensor, bool with_data) {
    std::ostringstream oss;
    oss << "ggml_tensor(";
    // 如果需要输出数据
    if (with_data) {
        // 如果张量维度大于3，添加左括号
        if (tensor->n_dims > 3)
            oss << "[";
        // 遍历第4维度
        for (int i3 = 0; i3 < tensor->ne[3]; i3++) {
            // 如果张量维度大于2，添加左括号
            if (tensor->n_dims > 2)
                oss << (i3 > 0 ? ",\n\n[" : "[");
            // 遍历第3维度
            for (int i2 = 0; i2 < tensor->ne[2]; i2++) {
                // 如果张量维度大于1，添加左括号
                if (tensor->n_dims > 1)
                    oss << (i2 > 0 ? ",\n\n[" : "[");
                // 遍历第2维度
                for (int i1 = 0; i1 < tensor->ne[1]; i1++) {
                    // 添加左括号
                    oss << (i1 > 0 ? ",\n[" : "[");
                    // 遍历第1维度
                    for (int i0 = 0; i0 < tensor->ne[0]; i0++) {
                        // 计算数据指针位置
                        auto ptr = (char *)tensor->data + i3 * tensor->nb[3] + i2 * tensor->nb[2] + i1 * tensor->nb[1] +
                                   i0 * tensor->nb[0];
                        // 添加逗号
                        oss << (i0 > 0 ? ", " : "");
                        // 根据张量类型输出数据
                        if (tensor->type == GGML_TYPE_I32) {
                            oss << *(int *)ptr;
                        } else {
                            float val;
                            if (tensor->type == GGML_TYPE_F32) {
                                val = *(float *)ptr;
                            } else if (tensor->type == GGML_TYPE_F16) {
                                val = ggml_fp16_to_fp32(*(ggml_fp16_t *)ptr);
                            } else {
                                CHATGLM_THROW << "unimplemented";
                            }
                            // 格式化输出浮点数
                            oss << std::setw(7) << std::fixed << std::setprecision(4) << val;
                        }
                    }
                    oss << "]";
                }
                // 如果张量维度大于1，添加右括号
                if (tensor->n_dims > 1)
                    oss << "]";
            }
            // 如果张量维度大于2，添加右括号
            if (tensor->n_dims > 2)
                oss << "]";
        }
        // 如果张量维度大于3，添加右括号
        if (tensor->n_dims > 3)
            oss << "]";
        // 添加逗号
        oss << ", ";
    }

    // 输出张��形状和步长信息
    oss << "shape=" << shape_to_string(tensor) << ", stride=" << strides_to_string(tensor) << ")";
    // 返回字符串
    return oss.str();
// 为张量分配缓冲区，如果使用了 CUDA，则调用 CUDA 版本的函数
ggml_tensor *tensor_assign_buffers(ggml_tensor *tensor) {
#ifdef GGML_USE_CUBLAS
    ggml_cuda_assign_buffers(tensor);
#endif
    // 返回张量
    return tensor;
}

// 将张量移动到设备上，如果使用了 CUDA，则进行相应的操作
ggml_tensor *tensor_to_device(ggml_tensor *tensor) {
#ifdef GGML_USE_CUBLAS
    // 如果张量在 CPU 上，则将其移动到 GPU 上
    if (tensor->backend == GGML_BACKEND_CPU) {
        tensor->backend = GGML_BACKEND_GPU;
        ggml_cuda_transform_tensor(tensor->data, tensor);
    }
#endif
    // 返回张量
    return tensor;
}

// 将张量移动到 CPU 上，如果使用了 CUDA，则进行相应的操作
ggml_tensor *tensor_to_cpu(ggml_tensor *tensor) {
#ifdef GGML_USE_CUBLAS
    // 如果张量不在 CPU 上，则释放 GPU 数据并将其移动到 CPU 上
    if (tensor->backend != GGML_BACKEND_CPU) {
        ggml_cuda_free_data(tensor);
        tensor->backend = GGML_BACKEND_CPU;
    }
#endif
    // 返回张量
    return tensor;
}

// 初始化静态成员变量
const std::string ToolCallMessage::TYPE_FUNCTION = "function";
const std::string ToolCallMessage::TYPE_CODE = "code";

// 初始化静态成员变量
const std::string ChatMessage::ROLE_USER = "user";
const std::string ChatMessage::ROLE_ASSISTANT = "assistant";
const std::string ChatMessage::ROLE_SYSTEM = "system";
const std::string ChatMessage::ROLE_OBSERVATION = "observation";

// 检查聊天消息的有效性
void BaseTokenizer::check_chat_messages(const std::vector<ChatMessage> &messages) {
    // 确保消息数量为奇数
    CHATGLM_CHECK(messages.size() % 2 == 1) << "invalid chat messages size " << messages.size();
    for (size_t i = 0; i < messages.size(); i++) {
        // 根据索引确定目标角色
        const std::string &target_role = (i % 2 == 0) ? ChatMessage::ROLE_USER : ChatMessage::ROLE_ASSISTANT;
        // 检查消息的角色是否符合预期
        CHATGLM_CHECK(messages[i].role == target_role)
            << "expect messages[" << i << "].role to be " << target_role << ", but got " << messages[i].role;
    }
}

// 辅助函数，用于计算图的计算
// 参考自 https://github.com/ggerganov/llama.cpp/blob/master/llama.cpp
void ggml_graph_compute_helper(std::vector<uninitialized_char> &buf, ggml_cgraph *graph, int n_threads) {
    // 根据图和线程数创建计划
    struct ggml_cplan plan = ggml_graph_plan(graph, n_threads);

    // 如果计划的工作大小大于 0，则调整缓冲区大小并设置工作数据
    if (plan.work_size > 0) {
        buf.resize(plan.work_size);
        plan.work_data = (uint8_t *)buf.data();
    }

    // 执行图的计算
    ggml_graph_compute(graph, &plan);
}

// 用于调试目的的空函数
// 在输入张量上添加一个全零张量，返回结果张量
[[maybe_unused]] static inline ggml_tensor *add_zero(ggml_context *ctx, ggml_tensor *tensor) {
    // 创建一个与输入张量相同类型和维度的全零张量
    ggml_tensor *zeros = ggml_new_tensor(ctx, tensor->type, tensor->n_dims, tensor->ne);
    // 将全零张量的所有元素设置为0
    ggml_set_f32(zeros, 0);
    // 将全零张量传输到设备上
    tensor_to_device(zeros);
    // 将输入张量和全零张量相加，并将结果赋值给输出张量
    ggml_tensor *out = tensor_assign_buffers(ggml_add(ctx, tensor, zeros));
    // 返回输出张量
    return out;
}

// 初始化设备上下文
void ModelContext::init_device_context() {
#ifdef GGML_USE_METAL
    // 创建 Metal 上下文
    ctx_metal = make_unique_ggml_metal_context(1);

    // 获取张量的最大尺寸
    const size_t max_size = ggml_get_max_tensor_size(ctx_w.get());

    // 获取权重数据的指针和大小
    void *weight_data = weight_buffer.empty() ? ggml_get_mem_buffer(ctx_w.get()) : (void *)weight_buffer.data();
    size_t weight_size = weight_buffer.empty() ? ggml_get_mem_size(ctx_w.get()) : weight_buffer.size();
    // 将权重数据添加到 Metal 上下文的缓冲区中
    CHATGLM_CHECK(ggml_metal_add_buffer(ctx_metal.get(), "weights", weight_data, weight_size, max_size));

    // 将键值数据添加到 Metal 上下文的缓冲区中
    CHATGLM_CHECK(ggml_metal_add_buffer(ctx_metal.get(), "kv", ggml_get_mem_buffer(ctx_kv.get()),
                                        ggml_get_mem_size(ctx_kv.get()), 0));

    // 获取计算数据的指针和大小
    void *compute_data = ctx_b ? ggml_get_mem_buffer(ctx_b.get()) : compute_buffer.data();
    size_t compute_size = ctx_b ? ggml_get_mem_size(ctx_b.get()) : compute_buffer.size();
    // 将计算数据添加到 Metal 上下文的缓冲区中
    CHATGLM_CHECK(ggml_metal_add_buffer(ctx_metal.get(), "compute", compute_data, compute_size, 0));

    // 将临时数据添加到 Metal 上下文的缓冲区中
    CHATGLM_CHECK(ggml_metal_add_buffer(ctx_metal.get(), "scratch", scratch.data, scratch.size, 0));
#endif
}

// 将输出标识符放入流处理器组中的每个流处理器
void StreamerGroup::put(const std::vector<int> &output_ids) {
    for (auto &streamer : streamers_) {
        streamer->put(output_ids);
    }
}

// 结束流处理器组中的每个流处理器
void StreamerGroup::end() {
    for (auto &streamer : streamers_) {
        streamer->end();
    }
}

// 修剪字符串开头的空格
static inline void ltrim(std::string &s) {
    s.erase(s.begin(), std::find_if(s.begin(), s.end(), [](unsigned char ch) { return !std::isspace(ch); }));
}

// 修剪字符串末尾的空格
// 去除字符串末尾的空白字符
static inline void rtrim(std::string &s) {
    s.erase(std::find_if(s.rbegin(), s.rend(), [](unsigned char ch) { return !std::isspace(ch); }).base(), s.end());
}

// 修剪字符串两端的空白字符（原地操作）
static inline void trim(std::string &s) {
    rtrim(s);
    ltrim(s);
}

void TextStreamer::put(const std::vector<int> &output_ids) {
    if (is_prompt_) {
        // 跳过提示
        is_prompt_ = false;
        return;
    }

    // 定义标点符号集合
    static const std::vector<char> puncts{',', '!', ':', ';', '?'};

    // 将输出 ID 添加到令牌缓存中
    token_cache_.insert(token_cache_.end(), output_ids.begin(), output_ids.end());
    // 解码令牌缓存生成文本
    std::string text = tokenizer_->decode(token_cache_);
    if (is_first_line_) {
        ltrim(text);
    }
    if (text.empty()) {
        return;
    }

    std::string printable_text;
    if (text.back() == '\n') {
        // 在换行符后刷新缓存
        printable_text = text.substr(print_len_);
        is_first_line_ = false;
        token_cache_.clear();
        print_len_ = 0;
    } else if (std::find(puncts.begin(), puncts.end(), text.back()) != puncts.end()) {
        // 最后一个符号是标点符号，暂时保留
    } else if (text.size() >= 3 && text.compare(text.size() - 3, 3, "�") == 0) {
        // 以不完整令牌结尾，暂时保留
    } else {
        printable_text = text.substr(print_len_);
        print_len_ = text.size();
    }

    os_ << printable_text << std::flush;
}

void TextStreamer::end() {
    std::string text = tokenizer_->decode(token_cache_);
    if (is_first_line_) {
        ltrim(text);
    }
    os_ << text.substr(print_len_) << std::endl;
    is_prompt_ = true;
    is_first_line_ = true;
    token_cache_.clear();
    print_len_ = 0;
}

void PerfStreamer::put(const std::vector<int> &output_ids) {
    CHATGLM_CHECK(!output_ids.empty());
    if (num_prompt_tokens_ == 0) {
        // 在提示评估之前
        start_us_ = ggml_time_us();
        num_prompt_tokens_ = output_ids.size();
}
    } else {
        // 如果不是第一个新的 token
        if (num_output_tokens_ == 0) {
            // 如果输出 token 数为 0，表示第一个新的 token
            // 记录当前时间戳
            prompt_us_ = ggml_time_us();
        }
        // 增加输出 token 的数量
        num_output_tokens_ += output_ids.size();
    }
// 重置性能流对象的各个时间和计数值为0
void PerfStreamer::reset() {
    start_us_ = prompt_us_ = end_us_ = 0;
    num_prompt_tokens_ = num_output_tokens_ = 0;
}

// 将性能流对象转换为字符串表示
std::string PerfStreamer::to_string() const {
    // 创建字符串流对象
    std::ostringstream oss;
    // 添加提示时间、提示令牌数量和平均提示令牌时间的信息
    oss << "prompt time: " << prompt_total_time_us() / 1000.f << " ms / " << num_prompt_tokens() << " tokens ("
        << prompt_token_time_us() / 1000.f << " ms/token)\n";
    // 添加输出时间、输出令牌数量和平均输出令牌时间的信息
    oss << "output time: " << output_total_time_us() / 1000.f << " ms / " << num_output_tokens() << " tokens ("
        << output_token_time_us() / 1000.f << " ms/token)\n";
    // 添加总时间的信息
    oss << "total time: " << (prompt_total_time_us() + output_total_time_us()) / 1000.f << " ms";
    // 返回字符串流对象转换为字符串后的结果
    return oss.str();
}

#ifdef _POSIX_MAPPED_FILES
// POSIX系统下的MappedFile构造函数
MappedFile::MappedFile(const std::string &path) {
    // 打开文件
    int fd = open(path.c_str(), O_RDONLY);
    // 检查文件是否成功打开
    CHATGLM_CHECK(fd > 0) << "cannot open file " << path << ": " << strerror(errno);

    // 获取文件状态信息
    struct stat sb;
    CHATGLM_CHECK(fstat(fd, &sb) == 0) << strerror(errno);
    size = sb.st_size;

    // 映射文件到内存
    data = (char *)mmap(nullptr, size, PROT_READ, MAP_SHARED, fd, 0);
    // 检查映射是否成功
    CHATGLM_CHECK(data != MAP_FAILED) << strerror(errno);

    // 关闭文件
    CHATGLM_CHECK(close(fd) == 0) << strerror(errno);
}

// POSIX系统下的MappedFile析构函数
MappedFile::~MappedFile() { CHATGLM_CHECK(munmap(data, size) == 0) << strerror(errno); }
#elif defined(_WIN32)
// Windows系统下的MappedFile构造函数
MappedFile::MappedFile(const std::string &path) {
    // 打开文件
    int fd = open(path.c_str(), O_RDONLY);
    // 检查文件是否成功打开
    CHATGLM_CHECK(fd > 0) << "cannot open file " << path << ": " << strerror(errno);

    // 获取文件状态信息
    struct _stat64 sb;
    CHATGLM_CHECK(_fstat64(fd, &sb) == 0) << strerror(errno);
    size = sb.st_size;

    // 获取文件句柄
    HANDLE hFile = (HANDLE)_get_osfhandle(fd);

    // 创建文件映射
    HANDLE hMapping = CreateFileMappingA(hFile, NULL, PAGE_READONLY, 0, 0, NULL);
    // 检查文件映射是否成功创建
    CHATGLM_CHECK(hMapping != NULL) << strerror(errno);

    // 映射文件到内存
    data = (char *)MapViewOfFile(hMapping, FILE_MAP_READ, 0, 0, 0);
    // 关闭文件映射句柄
    CloseHandle(hMapping);

    // 检查映射是否成功
    CHATGLM_CHECK(data != NULL) << strerror(errno);

    // 关闭文件
    CHATGLM_CHECK(close(fd) == 0) << strerror(errno);
}
// 在析构函数中释放映射文件的视图
MappedFile::~MappedFile() { CHATGLM_CHECK(UnmapViewOfFile(data)) << strerror(errno); }
#endif

// 根据给定的偏移量和寻址方式移动指针
void ModelLoader::seek(int64_t offset, int whence) {
    if (whence == SEEK_SET) {
        ptr = data + offset;
    } else if (whence == SEEK_CUR) {
        ptr += offset;
    } else if (whence == SEEK_END) {
        ptr = data + size + offset;
    } else {
        CHATGLM_THROW << "invalid seek mode " << whence;
    }
}

// 读取指定长度的字符串并更新指针位置
std::string ModelLoader::read_string(size_t length) {
    std::string s(ptr, ptr + length);
    ptr += length;
    return s;
}

// 读取并检查张量的元数据
void ModelLoader::checked_read_tensor_meta(const std::string &name, int target_ndim, int64_t *target_ne,
                                           ggml_type target_dtype) {
    // 读取并检查张量名称
    {
        int name_size = read_basic<int>();
        CHATGLM_CHECK(name_size == (int)name.size())
            << "tensor " << name << " name size mismatch: expect " << name.size() << " but got " << name_size;
        std::string weight_name = read_string(name_size);
        CHATGLM_CHECK(weight_name == name) << "tensor name mismatch: expect " << name << " but got " << weight_name;
    }

    // 读取并检查张量形状
    {
        int ndim = read_basic<int>();
        CHATGLM_CHECK(ndim == target_ndim)
            << "tensor " << name << " ndim mismatch: expect " << target_ndim << " but got " << ndim;
        for (int i = ndim - 1; i >= 0; i--) {
            int dim_size = read_basic<int>();
            CHATGLM_CHECK(dim_size == target_ne[i]) << "tensor " << name << " shape mismatch at dim " << i
                                                    << ": expect " << target_ne[i] << " but got " << dim_size;
        }
    }

    // 读取并检查张量数据类型
    {
        ggml_type dtype = (ggml_type)read_basic<int>();
        CHATGLM_CHECK(dtype == target_dtype)
            << "tensor " << name << " dtype mismatch: expect " << target_dtype << " but got " << dtype;
    }
}
// 读取指定大小的张量数据，确保内存对齐
void *ModelLoader::read_tensor_data(size_t nbytes) {
    // 内存对齐值
    constexpr int64_t MEM_ALIGNED = 16;
    // 计算数据偏移量，确保内存对齐
    const int64_t data_offset = (tell() + (MEM_ALIGNED - 1)) & ~(MEM_ALIGNED - 1);
    // 获取张量数据的指针
    void *tensor_data = data + data_offset;
    // 移动文件指针到数据结束位置
    seek(data_offset + nbytes, SEEK_SET);
    // 返回张量数据指针
    return tensor_data;
}

// 读取张量数据并填充到张量结构中
void ModelLoader::read_tensor(const std::string &name, ggml_tensor *tensor) {
    // 读取张量元数据
    checked_read_tensor_meta(name, tensor->n_dims, tensor->ne, tensor->type);
    // 读取张量数据并赋值给张量结构
    tensor->data = read_tensor_data(ggml_nbytes(tensor));
}

// Embedding 模块的前向传播
ggml_tensor *Embedding::forward(ModelContext *ctx, ggml_tensor *input) const {
    // 获取输出张量
    ggml_tensor *output = ggml_get_rows(ctx->ctx_b.get(), weight, input);
    // 返回输出张量
    return output;
}

// Linear 模块的前向传播
ggml_tensor *Linear::forward(ModelContext *ctx, ggml_tensor *input) const {
    // 获取上下文
    ggml_context *gctx = ctx->ctx_b.get();
    // 计算输出张量
    ggml_tensor *output = tensor_assign_buffers(ggml_mul_mat(gctx, weight, input)); // [seqlen, out_features]
    // 如果有偏置，则加上偏置
    if (bias) {
        output = tensor_assign_buffers(ggml_add_inplace(gctx, output, bias));
    }
    // 返回输出张量
    return output;
}

// LayerNorm 模块的前向传播
ggml_tensor *LayerNorm::forward(ModelContext *ctx, ggml_tensor *input) const {
    // 获取上下文
    ggml_context *gctx = ctx->ctx_b.get();
    // 根据是否原地操作选择不同的归一化函数
    auto ggml_norm_fn = inplace ? ggml_norm_inplace : ggml_norm;
    // 计算输出张量
    ggml_tensor *output = tensor_assign_buffers(ggml_norm_fn(gctx, input, eps));
    // 加权和偏置
    output = tensor_assign_buffers(ggml_mul_inplace(gctx, output, weight));
    output = tensor_assign_buffers(ggml_add_inplace(gctx, output, bias));
    // 返回输出张量
    return output;
}

// RMSNorm 模块的前向传播
ggml_tensor *RMSNorm::forward(ModelContext *ctx, ggml_tensor *input) const {
    // 获取上下文
    ggml_context *gctx = ctx->ctx_b.get();
    // 根据是否原地操作选择不同的 RMS 归一化函数
    auto ggml_rms_norm_fn = inplace ? ggml_rms_norm_inplace : ggml_rms_norm;
    // 计算输出张量
    ggml_tensor *output = tensor_assign_buffers(ggml_rms_norm_fn(gctx, input, eps));
    // 加权
    output = tensor_assign_buffers(ggml_mul_inplace(gctx, output, weight));
    // 返回输出张量
    return output;
}
// 获取物理核心数目
int get_num_physical_cores() {
    // 获取系统支持的线程数
    unsigned int n_threads = std::thread::hardware_concurrency();
    // 返回线程数，如果大于0，则返回线程数，如果小于等于4，则返回线程数，否则返回线程数的一半
    return n_threads > 0 ? (n_threads <= 4 ? n_threads : n_threads / 2) : 4;
}

// 获取默认线程数
int get_default_num_threads() {
#if defined(GGML_USE_CUBLAS) || defined(GGML_USE_METAL)
    // 如果定义了 GGML_USE_CUBLAS 或 GGML_USE_METAL，则返回1
    return 1;
#else
    // 否则返回物理核心数和16中的较小值
    return std::min(get_num_physical_cores(), 16);
#endif
}

// 将 ModelType 转换为字符串
std::string to_string(ModelType model_type) {
    // 根据不同的 ModelType 返回对应的字符串
    switch (model_type) {
    case ModelType::CHATGLM:
        return "ChatGLM";
    case ModelType::CHATGLM2:
        return "ChatGLM2";
    case ModelType::CHATGLM3:
        return "ChatGLM3";
    case ModelType::BAICHUAN7B:
        return "Baichuan7B";
    case ModelType::BAICHUAN13B:
        return "Baichuan13B";
    case ModelType::INTERNLM:
        return "InternLM";
    default:
        // 如果未知的 ModelType，则抛出异常
        CHATGLM_THROW << "unknown model type " << (int)model_type;
    }
}

// BaseModelForCausalLM 构造函数
BaseModelForCausalLM::BaseModelForCausalLM(ModelConfig config, size_t mem_size, size_t scratch_size, size_t num_weights)
    : config(config) {
    // 设置上下文的数据类型
    ctx_.dtype = config.dtype;
    // 计算上下文权重的大小
    const size_t ctx_w_size = num_weights * ggml_tensor_overhead();
    // 计算上下文键值的大小
    const size_t ctx_kv_size = 2 * config.num_hidden_layers *
                               (config.max_length * config.hidden_size / config.num_attention_heads *
                                    config.num_kv_heads * ggml_type_size(GGML_TYPE_F16) +
                                ggml_tensor_overhead());
    // 创建上下文权重
    ctx_.ctx_w = make_unique_ggml_context(ctx_w_size, nullptr, true);
    // 创建上下文键值
    ctx_.ctx_kv = make_unique_ggml_context(ctx_kv_size + 1 * MB, nullptr, false); // 1MB extra for MPS

    // 调整计算缓冲区大小
    ctx_.compute_buffer.resize(mem_size);
    // 调整临时缓冲区大小
    ctx_.scratch_buffer.resize(scratch_size);
    // 设置临时缓冲区
    ctx_.scratch = {0, ctx_.scratch_buffer.size(), ctx_.scratch_buffer.data()};
#ifdef GGML_USE_CUBLAS
    // 如果定义了 GGML_USE_CUBLAS，则设置 CUDA 的临时缓冲区大小
    ggml_cuda_set_scratch_size(scratch_size);
#endif
}
// 计算前向图的计算，返回 lm_logits
ggml_tensor *BaseModelForCausalLM::forward_graph_compute(const std::vector<int> &input_ids, int n_past, int n_ctx,
                                                         int n_threads, bool is_decoding) {
    // 创建上下文 ctx_b，使用 ctx_.compute_buffer 的大小和数据，不是解码
    ctx_.ctx_b = make_unique_ggml_context(ctx_.compute_buffer.size(), ctx_.compute_buffer.data(), false);
    // 清空 ctx_.gf
    ctx_.gf = {};

    // 如果线程数小于等于 0，则使用默认线程数
    if (n_threads <= 0) {
        n_threads = get_default_num_threads(); // 默认线程数
    }
    // 计算当前输入 ids 的大小
    int curr_input_ids_size = input_ids.size() - n_past;
    // 如果当前输入 ids 大小大于等于 32，且 CPU 支持 BLAS，且 CPU 不支持 GPU BLAS
    if (curr_input_ids_size >= 32 && ggml_cpu_has_blas() && !ggml_cpu_has_gpublas()) {
        n_threads = 1; // 如果启用 BLAS，则使用 1 个线程
    }

    // 创建 curr_input_ids 张量，1 维，类型为 GGML_TYPE_I32
    ggml_tensor *curr_input_ids = ggml_new_tensor_1d(ctx_.ctx_b.get(), GGML_TYPE_I32, curr_input_ids_size);
    // 复制当前输入 ids 数据到 curr_input_ids
    memcpy(curr_input_ids->data, input_ids.data() + n_past, ggml_nbytes(curr_input_ids));

    // 调用 forward 函数计算 lm_logits
    ggml_tensor *lm_logits = forward(&ctx_, curr_input_ids, n_past, n_ctx, is_decoding);
    // 设置 lm_logits 的后端为 GGML_BACKEND_CPU
    lm_logits->backend = GGML_BACKEND_CPU;

    // 构建前向扩展
    ggml_build_forward_expand(&ctx_.gf, lm_logits);
#ifdef GGML_USE_METAL
    // 如果使用 Metal，则调用 ggml_metal_graph_compute
    ggml_metal_graph_compute(ctx_.ctx_metal.get(), &ctx_.gf);
#else
    // 否则调用 ggml_graph_compute_helper
    ggml_graph_compute_helper(ctx_.work_buffer, &ctx_.gf, n_threads);
#endif

#ifdef GGML_PERF
    // 如果定义了 GGML_PERF，则打印图信息
    ggml_graph_print(&ctx_.gf);
#endif

    // 返回 lm_logits
    return lm_logits;
}

// 生成下一个 token
int BaseModelForCausalLM::generate_next_token(const std::vector<int> &input_ids, const GenerationConfig &gen_config,
                                              int n_past, int n_ctx) {
    // 调用 forward_graph_compute 计算 lm_logits
    ggml_tensor *lm_logits = forward_graph_compute(input_ids, n_past, n_ctx, gen_config.num_threads, true);

    // 获取词汇表大小和下一个 token 的 logits
    int vocab_size = lm_logits->ne[0];
    float *next_token_logits = (float *)lm_logits->data;

    // 检查是否有 nan
    for (int i = 0; i < vocab_size; i++) {
        CHATGLM_CHECK(std::isfinite(next_token_logits[i])) << "nan/inf encountered at lm_logits[" << i << "]";
    }

    // logits 预处理
}
    // 如果重复惩罚不等于1，则进行重复惩罚
    if (gen_config.repetition_penalty != 1.f) {
        sampling_repetition_penalty(next_token_logits, next_token_logits + vocab_size, input_ids,
                                    gen_config.repetition_penalty);
    }

    // 定义下一个token的id
    int next_token_id;
    
    // 如果进行采样
    if (gen_config.do_sample) {
        // 温度采样
        if (gen_config.temperature > 0) {
            sampling_temperature(next_token_logits, next_token_logits + vocab_size, gen_config.temperature);
        }

        // 创建包含token id和得分的vector
        std::vector<TokenIdScore> token_scores(vocab_size);
        for (int i = 0; i < vocab_size; i++) {
            token_scores[i] = TokenIdScore(i, next_token_logits[i]);
        }

        // top_k采样
        if (0 < gen_config.top_k && gen_config.top_k < (int)token_scores.size()) {
            sampling_top_k(token_scores.data(), token_scores.data() + gen_config.top_k,
                           token_scores.data() + token_scores.size());
            token_scores.resize(gen_config.top_k);
        }

        // top_p采样
        if (0.f < gen_config.top_p && gen_config.top_p < 1.f) {
            auto pos = sampling_top_p(token_scores.data(), token_scores.data() + token_scores.size(), gen_config.top_p);
            token_scores.resize(pos - token_scores.data());
        }

        // 在token_scores上进行softmax采样
        sampling_softmax_inplace(token_scores.data(), token_scores.data() + token_scores.size());
        for (size_t i = 0; i < token_scores.size(); i++) {
            next_token_logits[i] = token_scores[i].score;
        }

        // 创建随机设备和生成器
        thread_local std::random_device rd;
        thread_local std::mt19937 gen(rd());

        // 创建离散分布
        std::discrete_distribution<> dist(next_token_logits, next_token_logits + token_scores.size());
        next_token_id = token_scores[dist(gen)].id;
    } else {
        // 贪婪搜索
        next_token_id = std::max_element(next_token_logits, next_token_logits + vocab_size) - next_token_logits;
    }

    // 返回下一个token的id
    return next_token_id;
}

// 对输入的序列进行重复惩罚处理
void BaseModelForCausalLM::sampling_repetition_penalty(float *first, float *last, const std::vector<int> &input_ids,
                                                       float penalty) {
    // 检查惩罚值是否为正浮点数
    CHATGLM_CHECK(penalty > 0) << "penalty must be a positive float, but got " << penalty;
    // 计算惩罚值的倒数
    const float inv_penalty = 1.f / penalty;
    // 计算词汇表大小
    const int vocab_size = last - first;
    // 创建一个标记是否出现过的布尔向量
    std::vector<bool> occurrence(vocab_size, false);
    // 遍历输入的 id 序列
    for (const int id : input_ids) {
        // 如果 id 没有出现过
        if (!occurrence[id]) {
            // 根据惩罚值对对应位置的值进行调整
            first[id] *= (first[id] > 0) ? inv_penalty : penalty;
        }
        // 标记 id 已经出现过
        occurrence[id] = true;
    }
}

// 对输入的序列进行温度调整
void BaseModelForCausalLM::sampling_temperature(float *first, float *last, float temp) {
    // 计算温度的倒数
    const float inv_temp = 1.f / temp;
    // 遍历序列，对每个值进行温度调整
    for (float *it = first; it != last; it++) {
        *it *= inv_temp;
    }
}

// 对输入的 TokenIdScore 序列进行 Top-K 采样
void BaseModelForCausalLM::sampling_top_k(TokenIdScore *first, TokenIdScore *kth, TokenIdScore *last) {
    // 使用 std::nth_element 进行 Top-K 采样
    std::nth_element(first, kth, last, std::greater<TokenIdScore>());
}

// 对输入的 TokenIdScore 序列进行 Top-P 采样
TokenIdScore *BaseModelForCausalLM::sampling_top_p(TokenIdScore *first, TokenIdScore *last, float top_p) {
    // 在预期的 O(n) 时间复杂度内进行快速 Top-P 采样
    sampling_softmax_inplace(first, last);

    while (first + 1 < last) {
        // 使用最后一个元素的分数作为枢轴分数
        const float pivot_score = (last - 1)->score; // use mid score?
        // 使用 std::partition 进行 Top-P 采样
        TokenIdScore *mid =
            std::partition(first, last - 1, [pivot_score](const TokenIdScore &x) { return x.score > pivot_score; });
        std::swap(*mid, *(last - 1));

        // 计算前缀和
        const float prefix_sum =
            std::accumulate(first, mid, 0.f, [](float sum, const TokenIdScore &x) { return sum + x.score; });
        if (prefix_sum >= top_p) {
            last = mid;
        } else if (prefix_sum + mid->score < top_p) {
            first = mid + 1;
            top_p -= prefix_sum + mid->score;
        } else {
            return mid + 1;
        }
    }
    return last;
}

// 对输入的 TokenIdScore 序列进行原地 Softmax 处理
void BaseModelForCausalLM::sampling_softmax_inplace(TokenIdScore *first, TokenIdScore *last) {
    # 找到指定范围内最大元素的分数
    float max_score = std::max_element(first, last)->score;
    # 初始化总和为0
    float sum = 0.f;
    # 遍历指定范围内的 TokenIdScore 结构体指针
    for (TokenIdScore *p = first; p != last; p++) {
        # 计算当前元素的指数分数
        float s = std::exp(p->score - max_score);
        # 更新当前元素的分数为指数分数
        p->score = s;
        # 累加指数分数到总和
        sum += s;
    }
    # 计算总和的倒数
    float inv_sum = 1.f / sum;
    # 遍历指定范围内的 TokenIdScore 结构体指针
    for (TokenIdScore *p = first; p != last; p++) {
        # 将当前元素的分数乘以总和的倒数
        p->score *= inv_sum;
    }
// 结束 BaseModelForCausalLM 类的定义

std::vector<int> BaseModelForCausalLM::generate(const std::vector<int> &input_ids, const GenerationConfig &gen_config,
                                                BaseStreamer *streamer) {
    // 检查生成配置中的最大长度是否小于等于模型的最大长度
    CHATGLM_CHECK(gen_config.max_length <= config.max_length)
        << "requested max_length (" << gen_config.max_length << ") is larger than model's max_length ("
        << config.max_length << ")";

    // 初始化输出的 ID 列表，并预留空间
    std::vector<int> output_ids;
    output_ids.reserve(gen_config.max_length);
    // 将输入的 ID 列表赋值给输出的 ID 列表
    output_ids = input_ids;
    // 如果存在流处理器，则将输入的 ID 列表传递给流处理器
    if (streamer) {
        streamer->put(input_ids);
    }

    // 初始化过去的 token 数量为 0，当前 token 数量为输入 ID 列表的大小
    int n_past = 0;
    const int n_ctx = input_ids.size();
    // 计算最大新 token 数量
    const int max_new_tokens = (gen_config.max_new_tokens > 0) ? gen_config.max_new_tokens : gen_config.max_length;

    // 循环生成新的 token，直到达到最大长度或者遇到结束 token
    while ((int)output_ids.size() < std::min(gen_config.max_length, n_ctx + max_new_tokens)) {
        // 生成下一个 token 的 ID
        int next_token_id = generate_next_token(output_ids, gen_config, n_past, n_ctx);

        // 更新过去的 token 数量为输出 ID 列表的大小
        n_past = output_ids.size();
        // 将新生成的 token ID 添加到输出 ID 列表中
        output_ids.emplace_back(next_token_id);

        // 如果存在流处理器，则将新生成的 token ID 传递给流处理器
        if (streamer) {
            streamer->put({next_token_id});
        }

        // 如果生成的 token 是结束 token 或者额外的结束 token，则结束循环
        if (next_token_id == config.eos_token_id ||
            std::find(config.extra_eos_token_ids.begin(), config.extra_eos_token_ids.end(), next_token_id) !=
                config.extra_eos_token_ids.end()) {
            break;
        }
    }

    // 如果存在流处理器，则结束流处理
    if (streamer) {
        streamer->end();
    }

    // 返回生成的输出 ID 列表
    return output_ids;
}

// ChatGLM-6B 模型的定义

ChatGLMTokenizer::ChatGLMTokenizer(std::string_view serialized_model_proto) {
    // 从序列化的模型协议加载模型
    const auto status = sp.LoadFromSerializedProto(serialized_model_proto);
    // 检查加载模型的状态，如果不成功则输出错误信息
    CHATGLM_CHECK(status.ok()) << status.ToString();

    // 初始化特殊 token 的 ID
    bos_token_id = sp.PieceToId("<sop>");
    eos_token_id = sp.PieceToId("<eop>");
    mask_token_id = sp.PieceToId("[MASK]");
    gmask_token_id = sp.PieceToId("[gMASK]");
    pad_token_id = sp.PieceToId("<pad>");
}

// 使用 ChatGLMTokenizer 对象对文本进行编码
std::vector<int> ChatGLMTokenizer::encode(const std::string &text, int max_length) const {
    // 对文本进行预处理，返回预处理后的字符串
    std::string input = preprocess(text);
    // 创建一个整型向量用于存储编码后的文本
    std::vector<int> ids;
    // 使用 sp 对象对预处理后的文本进行编码，结果存储在 ids 中
    sp.Encode(input, &ids);
    // 在 ids 后面添加特殊标记的 token_id
    ids.insert(ids.end(), {gmask_token_id, bos_token_id});
    // 如果 ids 的长度超过了 max_length
    if ((int)ids.size() > max_length) {
        // 滑动窗口：始终保留最后的 max_length 个 token
        ids.erase(ids.begin(), ids.end() - max_length);
    }
    // 返回处理后的 ids
    return ids;
// 对一组聊天消息进行编码，返回编码后的输入 ID 列表
std::vector<int> ChatGLMTokenizer::encode_messages(const std::vector<ChatMessage> &messages, int max_length) const {
    // 构建提示信息
    std::string prompt = build_prompt(messages);
    // 对提示信息进行编码，限制最大长度
    std::vector<int> input_ids = encode(prompt, max_length);
    // 返回编码后的输入 ID 列表
    return input_ids;
}

// 构建提示信息，用于编码
std::string ChatGLMTokenizer::build_prompt(const std::vector<ChatMessage> &messages) {
    // 检查聊天消息是否有效
    check_chat_messages(messages);

    // 创建输出流对象
    std::ostringstream oss_prompt;
    // 如果只有一条消息
    if (messages.size() == 1) {
        oss_prompt << messages.front().content;
    } else {
        // 遍历消息，构建提示信息
        for (size_t i = 0; i < messages.size(); i += 2) {
            oss_prompt << "[Round " << i / 2 << "]\n问：" << messages[i].content << "\n答：";
            if (i + 1 < messages.size()) {
                oss_prompt << messages[i + 1].content << "\n";
            }
        }
    }
    // 返回构建好的提示信息
    return oss_prompt.str();
}

// 解码给定的 ID 列表，返回解码后的文本
std::string ChatGLMTokenizer::decode(const std::vector<int> &ids) const {
    // 创建文本字符串
    std::string text;
    // 使用模型解码 ID 列表，得到文本
    sp.Decode(ids, &text);
    // 对解码后的文本进行后处理
    text = postprocess(text);
    // 返回解码后的文本
    return text;
}

// 使用正则表达式替换文本中的特定内容
static std::string regex_replace(const std::string &input, const std::regex &regex,
                                 std::function<std::string(const std::smatch &)> format) {
    // 创建输出流对象
    std::ostringstream oss;
    // 记录上一个匹配的位置
    int last_index = 0;
    // 遍历匹配正则表达式的结果
    for (auto it = std::sregex_iterator(input.begin(), input.end(), regex); it != std::sregex_iterator(); it++) {
        // 将匹配结果之前的内容和格式化后的内容添加到输出流
        oss << it->prefix() << format(*it);
        // 更新上一个匹配的位置
        last_index = it->position() + it->length();
    }
    // 添加剩余未匹配的内容到输出流
    oss << input.substr(last_index);
    // 返回替换后的文本
    return oss.str();
}

// 对文本进行预处理，替换特定内容为特殊标记
std::string ChatGLMTokenizer::preprocess(const std::string &text) {
    // 创建输出字符串
    std::string output;

    // 替换换行符为特殊��记
    {
        static const std::regex newline_regex("\n");
        output = std::regex_replace(text, newline_regex, "<n>");
    }
    // 替换制表符为特殊标记
    {
        static const std::regex tab_regex("\t");
        output = std::regex_replace(output, tab_regex, "<|tab|>");
    }
    // 替换空格为特殊标记
    {
        // 创建一个静态正则表达式，匹配连续2到80个空格
        static const std::regex pattern(R"([ ]{2,80})");
        // 使用正则表达式替换匹配到的内容，替换规则为将匹配到的空格替换为特定格式的字符串
        output = regex_replace(output, pattern, [](const std::smatch &sm) {
            // 创建一个字符串流对象
            std::ostringstream oss;
            // 将特定格式的字符串写入字符串流
            oss << "<|blank_" << sm.str().size() << "|>";
            // 返回字符串流中的内容
            return oss.str();
        });
    }
    
    // 返回替换后的字符串
    return output;
}

// 替换文本中的标点符号
static inline std::string replace_punctuations(const std::string &text) {
    // 引用：https://stackoverflow.com/questions/37989081/how-to-use-unicode-range-in-c-regex
    // 创建 UTF-8 到 UTF-16 的编码转换器
    static std::wstring_convert<std::codecvt_utf8_utf16<wchar_t>> converter;
    // 定义标点符号替换规则
    static const std::vector<std::pair<std::wregex, std::wstring>> punct_map{
        {std::wregex(converter.from_bytes(R"(([\u4e00-\u9fff]),)")), converter.from_bytes("$1，")},
        {std::wregex(converter.from_bytes(R"(,([\u4e00-\u9fff]))")), converter.from_bytes("，$1")},
        {std::wregex(converter.from_bytes(R"(([\u4e00-\u9fff])!)")), converter.from_bytes("$1！")},
        {std::wregex(converter.from_bytes(R"(!([\u4e00-\u9fff]))")), converter.from_bytes("！$1")},
        {std::wregex(converter.from_bytes(R"(([\u4e00-\u9fff]):)")), converter.from_bytes("$1：")},
        {std::wregex(converter.from_bytes(R"(:([\u4e00-\u9fff]))")), converter.from_bytes("：$1")},
        {std::wregex(converter.from_bytes(R"(([\u4e00-\u9fff]);)")), converter.from_bytes("$1；")},
        {std::wregex(converter.from_bytes(R"(;([\u4e00-\u9fff]))")), converter.from_bytes("；$1")},
        {std::wregex(converter.from_bytes(R"(([\u4e00-\u9fff])\?)")), converter.from_bytes("$1？")},
        {std::wregex(converter.from_bytes(R"(\?([\u4e00-\u9fff]))")), converter.from_bytes("？$1")},
    };
    // 将输入文本转换为宽字符
    std::wstring w_output = converter.from_bytes(text);
    // 遍历标点符号替换规则，逐个替换
    for (const auto &punct_pair : punct_map) {
        w_output = std::regex_replace(w_output, punct_pair.first, punct_pair.second);
    }
    // 将替换后的宽字符转换为字符串
    std::string output = converter.to_bytes(w_output);
    return output;
}

// 后处理文本
std::string ChatGLMTokenizer::postprocess(const std::string &text) {
    std::string output;

    // 换行符号
    {
        static const std::regex pattern(R"(<n>)");
        output = std::regex_replace(text, pattern, "\n");
    }
    // 制表符号
    {
        static const std::regex pattern(R"(<\|tab\|>)");
        output = std::regex_replace(output, pattern, "\t");
    }
    // 匹配空白标记，例如<|blank_3|>，将其替换为对应数量的空格
    {
        static const std::regex pattern(R"(<\|blank_(\d+)\|>)");
        output = regex_replace(output, pattern,
                               [](const std::smatch &sm) { return std::string(std::stoi(sm[1].str()), ' '); });
    }
    // 替换文本中的标点符号
    output = replace_punctuations(output);

    // 返回处理后的文本
    return output;
}

// GLMContextMasker 类的 operator() 方法，用于对注意力分数进行处理
ggml_tensor *GLMContextMasker::operator()(ModelContext *ctx, ggml_tensor *attn_scores, int n_past) const {
    // attn_scores 的形状为 [heads, qlen, klen]
    ggml_context *gctx = ctx->ctx_b.get();
    const int qlen = attn_scores->ne[1];
    const int num_attention_heads = attn_scores->ne[2];
    // 创建一个形状为 [1, qlen - 1, num_attention_heads] 的新张量 inf，填充为负无穷
    ggml_tensor *inf = ggml_new_tensor_3d(gctx, attn_scores->type, 1, qlen - 1, num_attention_heads);
    ggml_set_f32(inf, -INFINITY);
    tensor_to_device(inf); // TODO: optimize
    // 创建一个形状与 attn_scores 相同的张量 masked_attn_scores，并赋值为 inf
    ggml_tensor *masked_attn_scores = tensor_assign_buffers(
        ggml_view_3d(gctx, attn_scores, 1, qlen - 1, num_attention_heads, qlen * ggml_element_size(attn_scores),
                     qlen * qlen * ggml_element_size(attn_scores), (qlen - 1) * ggml_element_size(attn_scores)));
    // 构建前向传播计算图
    ggml_build_forward_expand(&ctx->gf, ggml_cpy(gctx, inf, masked_attn_scores));
    // 返回原始的 attn_scores
    return attn_scores;
}

// GLMBlock 类的 forward 方法，用于前向传播计算
ggml_tensor *GLMBlock::forward(ModelContext *ctx, ggml_tensor *hidden_states, ggml_tensor *position_ids, int n_past,
                               int n_ctx) const {
    ggml_context *gctx = ctx->ctx_b.get();

    // 创建一个形状为 [1] 的新张量 alpha，值为 alpha_value
    ggml_tensor *alpha = ggml_new_f32(gctx, alpha_value);

    // 对隐藏状态进行输入层归一化
    ggml_tensor *attn_input = input_layernorm.forward(ctx, hidden_states);
    // 对归一化后的隐藏状态进行注意力计算
    ggml_tensor *attn_output = attention.forward(ctx, attn_input, position_ids, n_past, n_ctx);
    // 构建前向传播计算图
    ggml_build_forward_expand(&ctx->gf, attn_output);
    // 对 attn_input 进行缩放并赋值给 attn_input
    attn_input = tensor_assign_buffers(ggml_scale_inplace(gctx, attn_input, alpha));
    // 将 attn_input 和 attn_output 相加得到新的隐藏状态
    hidden_states = tensor_assign_buffers(ggml_add_inplace(gctx, attn_input, attn_output));

    // 对经过注意力计算后的隐藏状态进行后续处理
    ggml_tensor *mlp_input = post_attention_layernorm.forward(ctx, hidden_states);
    // 对处理后的隐藏状态进行 MLP 计算
    ggml_tensor *mlp_output = mlp.forward(ctx, mlp_input);
    // 构建前向传播计算图
    ggml_build_forward_expand(&ctx->gf, mlp_output);
    // 对 mlp_input 进行缩放并赋值给 mlp_input
    mlp_input = tensor_assign_buffers(ggml_scale_inplace(gctx, mlp_input, alpha));
    // 将 mlp_input 和 mlp_output 相加得到最终输出
    ggml_tensor *output = tensor_assign_buffers(ggml_add_inplace(gctx, mlp_input, mlp_output));

    // 返回最终输出张量
    return output;
}
// ChatGLMForCausalLM 类的构造函数，继承自 BasicModelForCausalLM 类，初始化状态字典
ChatGLMForCausalLM::ChatGLMForCausalLM(const ModelConfig &config)
    : BasicModelForCausalLM(config, MEM_SIZE, SCRATCH_SIZE, num_weights(config.num_hidden_layers)) {
    // 初始化状态字典
    state_dict_ = state_dict();
}

// ChatGLMForCausalLM 类的加载函数，加载模型参数
void ChatGLMForCausalLM::load(ModelLoader &loader) {
    // 遍历状态字典中的每个项
    for (auto &item : state_dict_) {
        const std::string &name = item.first;
        ggml_tensor *tensor = item.second;
        // 如果不是 "lm_head.weight"，则从 loader 中读取张量数据
        if (name != "lm_head.weight") {
            loader.read_tensor(name, tensor);
        }
    }
    // 将 lm_head.weight 的数据设置为 transformer.word_embeddings.weight 的数据，实现权重共享
    lm_head.weight->data = transformer.word_embeddings.weight->data; // tied weight

    // 将模型参数转移到设备上
    to_device();

    // 将 loader 中的数据设置为上下文的权重缓冲区，初始化设备上下文
    ctx_.weight_buffer = std::string_view(loader.data, loader.size);
    ctx_.init_device_context();
}

// ChatGLMForCausalLM 类的状态字典函数，返回状态字典
StateDict ChatGLMForCausalLM::state_dict() const {
    // 创建状态字典对象
    StateDict sd;
    // 预留状态字典的空间
    sd.reserve(num_weights(config.num_hidden_layers));
    // 将 "transformer.word_embeddings.weight" 和 transformer.word_embeddings.weight 添加到状态字典中
    sd.emplace_back("transformer.word_embeddings.weight", transformer.word_embeddings.weight);
}
    // 遍历隐藏层，构建每个隐藏层的参数键名前缀
    for (int i = 0; i < config.num_hidden_layers; i++) {
        std::string layer_prefix = "transformer.layers." + std::to_string(i) + '.';
        // 将输入层归一化的权重和偏置添加到参数字典中
        sd.emplace_back(layer_prefix + "input_layernorm.weight", transformer.layers[i].input_layernorm.weight);
        sd.emplace_back(layer_prefix + "input_layernorm.bias", transformer.layers[i].input_layernorm.bias);
        // 将注意力机制的权重和偏置添加到参数字典中
        sd.emplace_back(layer_prefix + "attention.query_key_value.weight",
                        transformer.layers[i].attention.query_key_value.weight);
        sd.emplace_back(layer_prefix + "attention.query_key_value.bias",
                        transformer.layers[i].attention.query_key_value.bias);
        // 将注意力机制中的全连接层的权重和偏置添加到参数字典中
        sd.emplace_back(layer_prefix + "attention.dense.weight", transformer.layers[i].attention.dense.weight);
        sd.emplace_back(layer_prefix + "attention.dense.bias", transformer.layers[i].attention.dense.bias);
        // 将注意力机制后的归一化层的权重和偏置添加到参数字典中
        sd.emplace_back(layer_prefix + "post_attention_layernorm.weight",
                        transformer.layers[i].post_attention_layernorm.weight);
        sd.emplace_back(layer_prefix + "post_attention_layernorm.bias",
                        transformer.layers[i].post_attention_layernorm.bias);
        // 将多层感知机中从隐藏层到四倍隐藏层的全连接层的权重和偏置添加到参数字典中
        sd.emplace_back(layer_prefix + "mlp.dense_h_to_4h.weight", transformer.layers[i].mlp.dense_h_to_4h.weight);
        sd.emplace_back(layer_prefix + "mlp.dense_h_to_4h.bias", transformer.layers[i].mlp.dense_h_to_4h.bias);
        // 将多层感知机中从四倍隐藏层到隐藏层的全连接层的权重和偏置添加到参数字典中
        sd.emplace_back(layer_prefix + "mlp.dense_4h_to_h.weight", transformer.layers[i].mlp.dense_4h_to_h.weight);
        sd.emplace_back(layer_prefix + "mlp.dense_4h_to_h.bias", transformer.layers[i].mlp.dense_4h_to_h.bias);
    }
    // 将最终归一化层的权重和偏置添加到参数字典中
    sd.emplace_back("transformer.final_layernorm.weight", transformer.final_layernorm.weight);
    sd.emplace_back("transformer.final_layernorm.bias", transformer.final_layernorm.bias);
    // 将语言模型头部的权重添加到参数字典中
    sd.emplace_back("lm_head.weight", lm_head.weight);
    // 返回参数字典
    return sd;
}

// ChatGLM2-6B 模型的 Tokenizer 类
ChatGLM2Tokenizer::ChatGLM2Tokenizer(std::string_view serialized_model_proto) {
    // 从序列化的模型协议中加载模型
    const auto status = sp.LoadFromSerializedProto(serialized_model_proto);
    // 检查加载状态是否成功
    CHATGLM_CHECK(status.ok()) << status.ToString();

    // 初始化特殊标记的 id
    int special_id = sp.GetPieceSize();
    mask_token_id = special_id++;
    gmask_token_id = special_id++;
    smask_token_id = special_id++;
    sop_token_id = special_id++;
    eop_token_id = special_id++;
}

// 将文本编码为 id 序列
std::vector<int> ChatGLM2Tokenizer::encode(const std::string &text, int max_length) const {
    // 初始化 id 序列
    std::vector<int> ids;
    // 使用 SentencePiece 对文本进行编码
    sp.Encode(text, &ids);
    // 在 id 序列开头插入特殊前缀标记
    ids.insert(ids.begin(), {gmask_token_id, sop_token_id}); // special prefix
    // 如果 id 序列长度超过最大长度，则进行截断
    if ((int)ids.size() > max_length) {
        // 滑动窗口：保留两个特殊前缀标记，删除最不常用的历史记录
        int num_drop = (int)ids.size() - max_length;
        ids.erase(ids.begin() + 2, ids.begin() + 2 + num_drop);
    }
    return ids;
}

// 将 id 序列解码为文本
std::string ChatGLM2Tokenizer::decode(const std::vector<int> &ids) const {
    // 过滤掉特殊标记
    std::vector<int> normal_ids(ids);
    normal_ids.erase(std::remove_if(normal_ids.begin(), normal_ids.end(), [this](int id) { return is_special_id(id); }),
                     normal_ids.end());

    // 解码 id 序列为文本
    std::string text;
    sp.Decode(normal_ids, &text);
    // 替换文本中的标点符号
    text = replace_punctuations(text);
    return text;
}

// 将聊天消息列表编码为 id 序列
std::vector<int> ChatGLM2Tokenizer::encode_messages(const std::vector<ChatMessage> &messages, int max_length) const {
    // 构建提示信息
    std::string prompt = build_prompt(messages);
    // 对提示信息进行编码
    std::vector<int> input_ids = encode(prompt, max_length);
    return input_ids;
}

// 构建提示信息
std::string ChatGLM2Tokenizer::build_prompt(const std::vector<ChatMessage> &messages) {
    // 检查聊天消息列表的有效性
    check_chat_messages(messages);

    // 初始化提示信息流
    std::ostringstream oss_prompt;
    // 遍历消息列表，每次增加2，i表示当前轮数
    for (size_t i = 0; i < messages.size(); i += 2) {
        // 构建提示信息，包括当前轮数和问题内容
        oss_prompt << "[Round " << i / 2 + 1 << "]\n\n问：" << messages[i].content << "\n\n答：";
        // 如果还有下一条消息，添加答案内容
        if (i < messages.size() - 1) {
            oss_prompt << messages[i + 1].content << "\n\n";
        }
    }
    // 返回提示信息的字符串形式
    return oss_prompt.str();
// 检查给定的 id 是否为特殊 id，如果是则返回 true，否则返回 false
bool ChatGLM2Tokenizer::is_special_id(int id) const {
    return id == mask_token_id || id == gmask_token_id || id == smask_token_id || id == sop_token_id ||
           id == eop_token_id;
}

// ChatGLM2ForCausalLM 类的构造函数，初始化基础模型并设置状态字典
ChatGLM2ForCausalLM::ChatGLM2ForCausalLM(const ModelConfig &config)
    : BasicModelForCausalLM(config, MEM_SIZE, SCRATCH_SIZE, num_weights(config.num_hidden_layers)) {
    state_dict_ = state_dict();
}

// 加载模型参数
void ChatGLM2ForCausalLM::load(ModelLoader &loader) {
    // 创建 glu_name_map，用于存储门控线性单元的名称映射关系
    std::unordered_map<std::string, std::string> glu_name_map;
    // 遍历隐藏层，设置门控线性单元的名称映射关系
    for (int i = 0; i < config.num_hidden_layers; i++) {
        std::string layer_prefix = "transformer.encoder.layers." + std::to_string(i) + '.';
        glu_name_map.emplace(layer_prefix + "mlp.gate_proj.weight", layer_prefix + "mlp.dense_h_to_4h.weight");
        glu_name_map.emplace(layer_prefix + "mlp.up_proj.weight", layer_prefix + "mlp.dense_h_to_4h.weight");
    }

    // 遍历状态字典，加载模型参数
    for (auto it = state_dict_.begin(); it != state_dict_.end(); it++) {
        const std::string &name = it->first;
        ggml_tensor *tensor = it->second;

        // 检查是否为门控线性单元的参数，如果是则加载对应的参数
        auto glu_it = glu_name_map.find(name);
        if (glu_it != glu_name_map.end()) {
            // 加载 gate_proj 和 up_proj 参数
            const std::string &dense_h_to_4h_name = glu_it->second;
            ggml_tensor *gate_proj = tensor;
            it++;
            CHATGLM_CHECK(glu_name_map.at(it->first) == dense_h_to_4h_name) << "corrupted glu weights";
            ggml_tensor *up_proj = it->second;

            // 设置目标形状和类型，读取数据并赋值给 gate_proj 和 up_proj
            int64_t target_ne[4]{gate_proj->ne[0], gate_proj->ne[1] + up_proj->ne[1]};
            loader.checked_read_tensor_meta(dense_h_to_4h_name, gate_proj->n_dims, target_ne, gate_proj->type);
            gate_proj->data = loader.read_tensor_data(ggml_nbytes(gate_proj));
            up_proj->data = loader.read_tensor_data(ggml_nbytes(up_proj));
        } else {
            // 加载普通参数
            loader.read_tensor(name, tensor);
        }
    }

    // 将加载的参数传输到设备上
    to_device();
}
    # 将loader中的数据和大小创建为一个std::string_view对象，并赋值给ctx_的weight_buffer
    ctx_.weight_buffer = std::string_view(loader.data, loader.size);
    # 初始化设备上下文
    ctx_.init_device_context();
// 返回模型的状态字典，包含所有权重参数
StateDict ChatGLM2ForCausalLM::state_dict() const {
    // 创建一个空的状态字典
    StateDict sd;
    // 预留空间以容纳所有权重参数
    sd.reserve(num_weights(config.num_hidden_layers));
    // 添加词嵌入权重参数到状态字典
    sd.emplace_back("transformer.embedding.word_embeddings.weight", transformer.word_embeddings.weight);
    // 遍历每个隐藏层，添加对应的权重参数到状态字典
    for (int i = 0; i < config.num_hidden_layers; i++) {
        // 生成当前隐藏层的前缀
        std::string layer_prefix = "transformer.encoder.layers." + std::to_string(i) + '.';
        // 添加输入层归一化权重参数到状态字典
        sd.emplace_back(layer_prefix + "input_layernorm.weight", transformer.layers[i].input_layernorm.weight);
        // 添加自注意力机制的查询、键、值权重参数到状态字典
        sd.emplace_back(layer_prefix + "self_attention.query_key_value.weight",
                        transformer.layers[i].attention.query_key_value.weight);
        // 添加自注意力机制的查询、键、值偏置参数到状态字典
        sd.emplace_back(layer_prefix + "self_attention.query_key_value.bias",
                        transformer.layers[i].attention.query_key_value.bias);
        // 添加自注意力机制的全连接层权重参数到状态字典
        sd.emplace_back(layer_prefix + "self_attention.dense.weight", transformer.layers[i].attention.dense.weight);
        // 添加自注意力机制后的归一化层权重参数到状态字典
        sd.emplace_back(layer_prefix + "post_attention_layernorm.weight",
                        transformer.layers[i].post_attention_layernorm.weight);
        // 添加多层感知机的门控投影权重参数到状态字典
        sd.emplace_back(layer_prefix + "mlp.gate_proj.weight", transformer.layers[i].mlp.gate_proj.weight);
        // 添加多层感知机的上投影权重参数到状态字典
        sd.emplace_back(layer_prefix + "mlp.up_proj.weight", transformer.layers[i].mlp.up_proj.weight);
        // 添加多层感知机的下投影权重参数到状态字典（兼容性）
        sd.emplace_back(layer_prefix + "mlp.dense_4h_to_h.weight", transformer.layers[i].mlp.down_proj.weight);
    }
    // 添加编码器最终归一化层权重参数到状态字典
    sd.emplace_back("transformer.encoder.final_layernorm.weight", transformer.final_layernorm.weight);
    // 添加输出层权重参数到状态字典
    sd.emplace_back("transformer.output_layer.weight", lm_head.weight);
    // 返回状态字典
    return sd;
}

// ChatGLM3-6B

// ChatGLM3Tokenizer类的构造函数，接��序列化的模型协议作为参数
ChatGLM3Tokenizer::ChatGLM3Tokenizer(std::string_view serialized_model_proto) {
    // 从序列化的模型协议加载模型
    const auto status = sp.LoadFromSerializedProto(serialized_model_proto);
    // 检查加载状态是否成功
    CHATGLM_CHECK(status.ok()) << status.ToString();

    // 获取特殊标记的ID，并为掩码标记和全局掩码标记分配ID
    int special_id = sp.GetPieceSize();
    mask_token_id = special_id++;
    gmask_token_id = special_id++;
}
    # 分配特殊标记的 ID，并递增
    smask_token_id = special_id++;
    sop_token_id = special_id++;
    eop_token_id = special_id++;
    system_token_id = special_id++;
    user_token_id = special_id++;
    assistant_token_id = special_id++;
    observation_token_id = special_id++;

    # 初始化特殊标记和对应的 ID
    special_tokens = {
        {"[MASK]", mask_token_id},
        {"[gMASK]", gmask_token_id},
        {"[sMASK]", smask_token_id},
        {"sop", sop_token_id},
        {"eop", eop_token_id},
        {"<|system|>", system_token_id},
        {"<|user|>", user_token_id},
        {"<|assistant|>", assistant_token_id},
        {"<|observation|>", observation_token_id},
    };

    # 将特殊标记和对应的 ID 存入索引字典
    for (const auto &item : special_tokens) {
        index_special_tokens[item.second] = item.first;
    }
// 将文本编码为整数序列，最大长度为max_length
std::vector<int> ChatGLM3Tokenizer::encode(const std::string &text, int max_length) const {
    // 创建一个空的整数向量ids
    std::vector<int> ids;
    // 使用Tokenizer对象sp对文本进行编码，结果存储在ids中
    sp.Encode(text, &ids);
    // 在ids的开头插入特殊标记的token_id，用于表示特殊前缀
    ids.insert(ids.begin(), {gmask_token_id, sop_token_id}); // special prefix
    // 截断ids，使其长度不超过max_length
    truncate(ids, max_length);
    // 返回编码后的整数序列ids
    return ids;
}

// 将整数序列解码为文本
std::string ChatGLM3Tokenizer::decode(const std::vector<int> &ids) const {
    // 使用特殊标记的token_id解码整数序列ids，得到文本
    std::string text = decode_with_special_tokens(ids);
    // 移除文本中的特殊标记
    text = remove_special_tokens(text);
    // 返回解码后的文本
    return text;
}

// 解码整数序列，包含特殊标记的token
std::string ChatGLM3Tokenizer::decode_with_special_tokens(const std::vector<int> &ids) const {
    // 创建一个空的字符串向量pieces
    std::vector<std::string> pieces;
    // 遍历整数序列ids
    for (int id : ids) {
        // 查找id在特殊标记索引中的位置
        auto pos = index_special_tokens.find(id);
        // 如果找到了
        if (pos != index_special_tokens.end()) {
            // 添加特殊标记到pieces中
            pieces.emplace_back(pos->second);
        } else {
            // 添加普通token到pieces中
            pieces.emplace_back(sp.IdToPiece(id));
        }
    }

    // 将pieces中的字符串拼接成文本
    std::string text = sp.DecodePieces(pieces);
    // 返回解码后的文本
    return text;
}

// 移除文本中的特殊标记
std::string ChatGLM3Tokenizer::remove_special_tokens(const std::string &text) {
    // 将文本赋值给output
    std::string output = text;
    // 定义特殊标记的正则表达式
    static const std::vector<std::regex> special_token_regex{
        std::regex(R"(<\|assistant\|>)"),
        std::regex(R"(<\|user\|>)"),
        std::regex(R"(<\|observation\|>"),
    };
    // 遍历特殊标记的正则表达式
    for (const auto &re : special_token_regex) {
        // 使用正则表达式替换output中的特殊标记为空字符串
        output = std::regex_replace(output, re, "");
    }
    // 返回移除特殊标记后的文本
    return output;
}

// 将单条消息编码为整数序列
std::vector<int> ChatGLM3Tokenizer::encode_single_message(const std::string &role, const std::string &content) const {
    // 创建一个空的整数向量input_ids
    std::vector<int> input_ids;
    // 将角色role转换为对应的命令id，并添加到input_ids中
    input_ids.emplace_back(get_command("<|" + role + "|>"));
    // TODO: support metadata
    // 将换行符"\n"编码为整数序列，并存储在newline_ids中
    std::vector<int> newline_ids;
    sp.Encode("\n", &newline_ids);
    // 将newline_ids中的整数序列添加到input_ids的末尾
    input_ids.insert(input_ids.end(), newline_ids.begin(), newline_ids.end());
    // 将消息内容content编码为整数序列，并存储在content_ids中
    std::vector<int> content_ids;
    sp.Encode(content, &content_ids);
    # 将 content_ids 的内容插入到 input_ids 的末尾
    input_ids.insert(input_ids.end(), content_ids.begin(), content_ids.end());
    # 返回合并后的 input_ids
    return input_ids;
// 将多条聊天消息编码成整数向量，限制最大长度
std::vector<int> ChatGLM3Tokenizer::encode_messages(const std::vector<ChatMessage> &messages, int max_length) const {
    // 初始化输入向量，包含特殊标记
    std::vector<int> input_ids{gmask_token_id, sop_token_id};
    // 遍历每条消息
    for (const auto &msg : messages) {
        // 编码单条消息
        auto msg_ids = encode_single_message(msg.role, msg.content);
        // 将单条消息的编码添加到输入向量中
        input_ids.insert(input_ids.end(), msg_ids.begin(), msg_ids.end());

        // 如果消息包含代码块，则将代码块编码添加到输入向量中
        if (!msg.tool_calls.empty() && msg.tool_calls.front().type == ToolCallMessage::TYPE_CODE) {
            auto code_ids = encode_single_message(msg.role, msg.tool_calls.front().code.input);
            input_ids.insert(input_ids.end(), code_ids.begin(), code_ids.end());
        }
    }
    // 添加结束标记
    input_ids.emplace_back(assistant_token_id);
    // 根据最大长度截断输入向量
    truncate(input_ids, max_length);
    // 返回编码后的输入向量
    return input_ids;
}

// 解码整数向量为聊天消息
ChatMessage ChatGLM3Tokenizer::decode_message(const std::vector<int> &ids) const {
    ChatMessage message;
    // 解码消息
    } else {
        // 会话模式下，调用基类解码方法，并修剪内容
        message = BaseTokenizer::decode_message(ids);
        trim(message.content); // 去除会话模式下的开头换行符
    }
    // 返回解码后的消息
    return message;
}

// 获取特殊标记对应的命令
int ChatGLM3Tokenizer::get_command(const std::string &token) const {
    auto pos = special_tokens.find(token);
    // 检查特殊标记是否存在
    CHATGLM_CHECK(pos != special_tokens.end()) << token << " is not a special token";
    // 返回特殊标记对应的命令
    return pos->second;
}

// 判断是否为特殊标记的ID
bool ChatGLM3Tokenizer::is_special_id(int id) const { return index_special_tokens.count(id) > 0; }

// 截断整数向量至指定最大长度
void ChatGLM3Tokenizer::truncate(std::vector<int> &ids, int max_length) {
    if ((int)ids.size() > max_length) {
        // 滑动窗口：保留两个特殊前缀标记，删除最不常用的历史记录
        int num_drop = (int)ids.size() - max_length;
        ids.erase(ids.begin() + 2, ids.begin() + 2 + num_drop);
    }
}

// BaichuanTokenizer类的构造函数，从序列化的模型协议加载模型
BaichuanTokenizer::BaichuanTokenizer(std::string_view serialized_model_proto) {
    // 从序列化的模型协议加载模型
    const auto status = sp.LoadFromSerializedProto(serialized_model_proto);
}
    # 检查状态是否为OK，如果不是则输出状态信息并终止程序
    CHATGLM_CHECK(status.ok()) << status.ToString();
// BaichuanTokenizer 类的 encode 方法，将文本编码为整数序列
std::vector<int> BaichuanTokenizer::encode(const std::string &text, int max_length) const {
    // 创建一个空的整数向量 ids
    std::vector<int> ids;
    // 使用 SentencePiece 对象 sp 对文本进行编码，结果存储在 ids 中
    sp.Encode(text, &ids);
    // 截断 ids，使其长度不超过 max_length
    truncate(ids, max_length);
    // 返回编码后的整数序列
    return ids;
}

// BaichuanTokenizer 类的 decode 方法，将整数序列解码为文本
std::string BaichuanTokenizer::decode(const std::vector<int> &ids) const {
    // 复制整数序列 ids 到 normal_ids
    std::vector<int> normal_ids(ids);
    // 移除特殊 id，即不是 bos_token_id、eos_token_id、pad_token_id 的 id
    normal_ids.erase(std::remove_if(normal_ids.begin(), normal_ids.end(), [this](int id) { return is_special_id(id); }),
                     normal_ids.end());
    
    // 解码 normal_ids，得到文本结果存储在 text 中
    std::string text;
    sp.Decode(normal_ids, &text);
    // 返回解码后的文本
    return text;
}

// BaichuanTokenizer 类的 encode_messages 方法，将多个聊天消息编码为整数序列
std::vector<int> BaichuanTokenizer::encode_messages(const std::vector<ChatMessage> &messages, int max_length) const {
    // 检查聊天消息是否合法
    check_chat_messages(messages);

    // 创建一个空的整数向量 ids，并预留 max_length 的空间
    std::vector<int> ids;
    ids.reserve(max_length);
    // 遍历每条消息，根据角色添加对应的 token_id，并将消息内容编码后添加到 ids 中
    for (const auto &msg : messages) {
        ids.push_back((msg.role == ChatMessage::ROLE_USER) ? USER_TOKEN_ID : ASSISTANT_TOKEN_ID);
        std::vector<int> content_ids = encode(msg.content, max_length);
        ids.insert(ids.end(), content_ids.begin(), content_ids.end());
    }
    // 添加 ASSISTANT_TOKEN_ID 作为结束标志
    ids.push_back(ASSISTANT_TOKEN_ID);

    // 截断 ids，使其长度不超过 max_length
    truncate(ids, max_length);
    // 返回编码后的整数序列
    return ids;
}

// BaichuanTokenizer 类的 is_special_id 方法，判断是否为特殊 id
bool BaichuanTokenizer::is_special_id(int id) const {
    return id == bos_token_id || id == eos_token_id || id == pad_token_id;
}

// BaichuanTokenizer 类的 truncate 方法，截断整数序列使其长度不超过 max_length
void BaichuanTokenizer::truncate(std::vector<int> &ids, int max_length) {
    // 如果 ids 的长度超过 max_length，则截断多余部分
    if ((int)ids.size() > max_length) {
        ids.erase(ids.begin(), ids.end() - max_length);
    }
}

// Baichuan7BForCausalLM 类的构造函数，初始化模型参数
Baichuan7BForCausalLM::Baichuan7BForCausalLM(const ModelConfig &config)
    : BasicModelForCausalLM(config, MEM_SIZE, SCRATCH_SIZE, num_weights(config.num_hidden_layers)) {
    // 初始化状态字典
    state_dict_ = state_dict();
}

// Baichuan7BForCausalLM 类的 load 方法，加载模型参数
void Baichuan7BForCausalLM::load(ModelLoader &loader) {
    // 遍历状态字典，读取对应的张量
    for (auto &item : state_dict_) {
        const std::string &name = item.first;
        ggml_tensor *tensor = item.second;
        loader.read_tensor(name, tensor);
    }

    // 将加载的参数转移到设备上

    // 将加载的数据存储到 ctx_ 的 weight_buffer 中
    ctx_.weight_buffer = std::string_view(loader.data, loader.size);
}
    # 初始化设备上下文
    ctx_.init_device_context();
// 定义 Baichuan7BForCausalLM 类的 state_dict 方法，返回模型的状态字典
StateDict Baichuan7BForCausalLM::state_dict() const {
    // 创建一个空的状态字典
    StateDict sd;
    // 预留状态字典的空间，大小为隐藏层权重的数量
    sd.reserve(num_weights(config.num_hidden_layers));
    // 将词嵌入权重添加到状态字典中
    sd.emplace_back("model.embed_tokens.weight", transformer.word_embeddings.weight);
    // 遍历每个隐藏层，将各层的权重添加到状态字典中
    for (int i = 0; i < config.num_hidden_layers; i++) {
        // 生成当前隐藏层的前缀
        std::string layer_prefix = "model.layers." + std::to_string(i) + '.';
        // 添加输入层归一化权重到状态字典中
        sd.emplace_back(layer_prefix + "input_layernorm.weight", transformer.layers[i].input_layernorm.weight);
        // 添加自注意力权重到状态字典中
        sd.emplace_back(layer_prefix + "self_attn.W_pack.weight", transformer.layers[i].attention.query_key_value.weight);
        // 添加自注意力输出权重到状态字典中
        sd.emplace_back(layer_prefix + "self_attn.o_proj.weight", transformer.layers[i].attention.dense.weight);
        // 添加注意力后归一化权重到状态字典中
        sd.emplace_back(layer_prefix + "post_attention_layernorm.weight", transformer.layers[i].post_attention_layernorm.weight);
        // 添加 MLP 门控投影权重到状态字典中
        sd.emplace_back(layer_prefix + "mlp.gate_proj.weight", transformer.layers[i].mlp.gate_proj.weight);
        // 添加 MLP 下投影权重到状态字典中
        sd.emplace_back(layer_prefix + "mlp.down_proj.weight", transformer.layers[i].mlp.down_proj.weight);
        // 添加 MLP 上投影权重到状态字典中
        sd.emplace_back(layer_prefix + "mlp.up_proj.weight", transformer.layers[i].mlp.up_proj.weight);
    }
    // 添加最终归一化权重到状态字典中
    sd.emplace_back("model.norm.weight", transformer.final_layernorm.weight);
    // 添加语言模型头权重到状态字典中
    sd.emplace_back("lm_head.weight", lm_head.weight);
    // 返回状态字典
    return sd;
}

// ===== Baichuan-13B =====

// Baichuan13BForCausalLM 类的构造函数，初始化模型配置和状态字典
Baichuan13BForCausalLM::Baichuan13BForCausalLM(const ModelConfig &config)
    : BasicModelForCausalLM(config, MEM_SIZE, SCRATCH_SIZE, num_weights(config.num_hidden_layers)) {
    // 生成状态字典
    state_dict_ = state_dict();
}

// Baichuan13BForCausalLM 类的加载方法，从加载器中读取状态字典并将数据传输到设备上
void Baichuan13BForCausalLM::load(ModelLoader &loader) {
    // 遍���状态字典中的每个项
    for (auto &item : state_dict_) {
        // 获取项的名称和张量
        const std::string &name = item.first;
        ggml_tensor *tensor = item.second;
        // 从加载器中读取张量数据
        loader.read_tensor(name, tensor);
    }

    // 将数据传输到设备上
    to_device();

    // 将加载器中的数据作为权重缓冲区
    ctx_.weight_buffer = std::string_view(loader.data, loader.size);
    // 初始化设备上下文
    ctx_.init_device_context();
}
// 返回模型的状态字典，包含所有权重参数
StateDict Baichuan13BForCausalLM::state_dict() const {
    // 创建一个空的状态字典
    StateDict sd;
    // 预留空间以容纳所有权重参数
    sd.reserve(num_weights(config.num_hidden_layers));
    // 添加词嵌入权重参数到状态字典
    sd.emplace_back("model.embed_tokens.weight", transformer.word_embeddings.weight);
    // 遍历每个隐藏层，添加对应的权重参数到状态字典
    for (int i = 0; i < config.num_hidden_layers; i++) {
        // 生成当前层的前缀
        std::string layer_prefix = "model.layers." + std::to_string(i) + '.';
        // 添加输入层归一化权重参数到状态字典
        sd.emplace_back(layer_prefix + "input_layernorm.weight", transformer.layers[i].input_layernorm.weight);
        // 添加自注意力机制的权重参数到状态字典
        sd.emplace_back(layer_prefix + "self_attn.W_pack.weight", transformer.layers[i].attention.query_key_value.weight);
        // 添加自注意力机制输出投影的权重参数到状态字典
        sd.emplace_back(layer_prefix + "self_attn.o_proj.weight", transformer.layers[i].attention.dense.weight);
        // 添加注意力后的归一化权重参数到状态字典
        sd.emplace_back(layer_prefix + "post_attention_layernorm.weight", transformer.layers[i].post_attention_layernorm.weight);
        // 添加多层感知机门控投影的权重参数到状态字典
        sd.emplace_back(layer_prefix + "mlp.gate_proj.weight", transformer.layers[i].mlp.gate_proj.weight);
        // 添加多层感知机下投影的权重参数到状态字典
        sd.emplace_back(layer_prefix + "mlp.down_proj.weight", transformer.layers[i].mlp.down_proj.weight);
        // 添加多层感知机上投影的权重参数到状态字典
        sd.emplace_back(layer_prefix + "mlp.up_proj.weight", transformer.layers[i].mlp.up_proj.weight);
    }
    // 添加最终归一化层的权重参数到状态字典
    sd.emplace_back("model.norm.weight", transformer.final_layernorm.weight);
    // 添加语言模型头部的权重参数到状态字典
    sd.emplace_back("lm_head.weight", lm_head.weight);
    // 返回状态字典
    return sd;
}

// 初始化 InternLMTokenizer 类
InternLMTokenizer::InternLMTokenizer(std::string_view serialized_model_proto) {
    // 从序列化的模型协议加载模型
    const auto status = sp.LoadFromSerializedProto(serialized_model_proto);
    // 检查加载状态是否成功
    CHATGLM_CHECK(status.ok()) << status.ToString();
}

// 将文本编码为整数序列
std::vector<int> InternLMTokenizer::encode(const std::string &text, int max_length) const {
    // 创建一个空的整数序列
    std::vector<int> ids;
    // 使用模型进行文本编码
    sp.Encode(text, &ids);
    // 在序列开头插入特殊前缀标记
    ids.insert(ids.begin(), {bos_token_id}); // special prefix
    // 检查 ids 的大小是否超过最大长度
    if ((int)ids.size() > max_length) {
        // 如果超过最大长度，采用滑动窗口的方式删除最旧的历史记录，但保留特殊前缀
        // 计算需要删除的历史记录数量
        int num_drop = (int)ids.size() - max_length;
        // 从 ids 中删除最旧的历史记录，保留特殊前缀
        ids.erase(ids.begin() + 1, ids.begin() + 1 + num_drop);
    }
    // 返回处理后的 ids
    return ids;
}

// 解码函数，将输入的整数序列转换为文本
std::string InternLMTokenizer::decode(const std::vector<int> &ids) const {
    // 过滤掉特殊标记
    std::vector<int> normal_ids(ids);
    normal_ids.erase(std::remove_if(normal_ids.begin(), normal_ids.end(), [this](int id) { return is_special_id(id); }),
                     normal_ids.end());

    // 解码整数序列为文本
    std::string text;
    sp.Decode(normal_ids, &text);
    // 移除 <eoa> 及其后面的内容
    size_t eoa_pos = text.find("<eoa>");
    if (eoa_pos != std::string::npos) {
        text.erase(eoa_pos);
    }
    return text;
}

// 将聊天消息编码为整数序列
std::vector<int> InternLMTokenizer::encode_messages(const std::vector<ChatMessage> &messages, int max_length) const {
    // 构建提示信息
    std::string prompt = build_prompt(messages);
    // 编码提示信息为整数序列
    std::vector<int> input_ids = encode(prompt, max_length);
    return input_ids;
}

// 构建提示信息字符串
std::string InternLMTokenizer::build_prompt(const std::vector<ChatMessage> &messages) {
    // 检查聊天消息
    check_chat_messages(messages);

    // 构建提示信息字符串
    std::ostringstream oss_prompt;
    for (const auto &msg : messages) {
        if (msg.role == ChatMessage::ROLE_USER) {
            oss_prompt << "<|User|>:" << msg.content << "<eoh>\n<|Bot|>:";
        } else {
            oss_prompt << msg.content << "<eoa>\n";
        }
    }
    return oss_prompt.str();
}

// InternLMForCausalLM 类的构造函数
template <typename InternLMModel>
InternLMForCausalLM<InternLMModel>::InternLMForCausalLM(const ModelConfig &config)
    : BasicModelForCausalLM<InternLMModel>(config, MEM_SIZE, SCRATCH_SIZE, num_weights(config.num_hidden_layers)) {
    this->state_dict_ = state_dict();
}

// 加载模型
template <typename InternLMModel>
void InternLMForCausalLM<InternLMModel>::load(ModelLoader &loader) {
    for (auto &item : this->state_dict_) {
        const std::string &name = item.first;
        ggml_tensor *tensor = item.second;
        loader.read_tensor(name, tensor);
    }

    this->to_device();

    this->ctx_.weight_buffer = std::string_view(loader.data, loader.size);
    this->ctx_.init_device_context();
}

// InternLMModel 模板
// 返回当前模型的状态字典
StateDict InternLMForCausalLM<InternLMModel>::state_dict() const {
    // 创建一个空的状态字典
    StateDict sd;
    // 预留状态字典的空间，根据隐藏层的数量确定
    sd.reserve(num_weights(this->config.num_hidden_layers));
    // 将词嵌入权重添加到状态字典中
    sd.emplace_back("model.embed_tokens.weight", this->transformer.word_embeddings.weight);
    // 遍历隐藏层，将每一层的权重添加到状态字典中
    for (int i = 0; i < this->config.num_hidden_layers; i++) {
        // 生成当前层的前缀
        std::string layer_prefix = "model.layers." + std::to_string(i) + '.';
        // 添加输入层归一化权重到状态字典中
        sd.emplace_back(layer_prefix + "input_layernorm.weight", this->transformer.layers[i].input_layernorm.weight);
        // 添加自注意力机制的查询、键、值权重到状态字典中
        sd.emplace_back(layer_prefix + "self_attn.qkv_proj.weight",
                        this->transformer.layers[i].attention.query_key_value.weight);
        // 如果自注意力机制有偏置，则添加到状态字典中
        if (this->transformer.layers[i].attention.query_key_value.bias) {
            sd.emplace_back(layer_prefix + "self_attn.qkv_proj.bias",
                            this->transformer.layers[i].attention.query_key_value.bias);
        }
        // 添加自注意力机制输出权重到状态字典中
        sd.emplace_back(layer_prefix + "self_attn.o_proj.weight", this->transformer.layers[i].attention.dense.weight);
        // 如果自注意力机制输出有偏置，则添加到状态字典中
        if (this->transformer.layers[i].attention.dense.bias) {
            sd.emplace_back(layer_prefix + "self_attn.o_proj.bias", this->transformer.layers[i].attention.dense.bias);
        }
        // 添加自注意力机制后的归一化权重到状态字典中
        sd.emplace_back(layer_prefix + "post_attention_layernorm.weight",
                        this->transformer.layers[i].post_attention_layernorm.weight);
        // 添加MLP门控投影权重到状态字典中
        sd.emplace_back(layer_prefix + "mlp.gate_proj.weight", this->transformer.layers[i].mlp.gate_proj.weight);
        // 添加MLP上投影权重到状态字典中
        sd.emplace_back(layer_prefix + "mlp.up_proj.weight", this->transformer.layers[i].mlp.up_proj.weight);
        // 添加MLP下投影权重到状态字典中
        sd.emplace_back(layer_prefix + "mlp.down_proj.weight", this->transformer.layers[i].mlp.down_proj.weight);
    }
    // 添加最终归一化权重到状态字典中
    sd.emplace_back("model.norm.weight", this->transformer.final_layernorm.weight);
    // 添加语言模型头部权重到状态字典中
    sd.emplace_back("lm_head.weight", this->lm_head.weight);
    // 返回状态字典
    return sd;
}

// 构造函数，初始化管道对象
Pipeline::Pipeline(const std::string &path) {
    // 使用给定路径创建一个映射文件对象
    mapped_file = std::make_unique<MappedFile>(path);
}
    // 创建一个 ModelLoader 对象，用于加载模型文件的数据
    ModelLoader loader(mapped_file->data, mapped_file->size);

    // 读取文件中的魔数（magic number），应为 "ggml"
    std::string magic = loader.read_string(4);
    // 检查魔数是否正确，如果不正确则输出错误信息
    CHATGLM_CHECK(magic == "ggml") << "model file is broken (bad magic)";

    // 读取模型类型
    ModelType model_type = (ModelType)loader.read_basic<int>();
    // 读取版本号
    int version = loader.read_basic<int>();
    // 如果模型类型为 CHATGLM
    if (model_type == ModelType::CHATGLM) {
        // 检查版本号是否为1，如果不是则输出错误信息
        CHATGLM_CHECK(version == 1) << "only support version 1 for now but got " << version;

        // 读取模型配置信息
        ModelConfig config(model_type, loader.read_basic<ConfigRecordV1>());

        // 读取 tokenizer 的大小
        int proto_size = loader.read_basic<int>();
        // 从文件中读取序列化的模型 proto 数据
        std::string_view serialized_model_proto((char *)mapped_file->data + loader.tell(), proto_size);
        // 移动指针到下一个位置
        loader.seek(proto_size, SEEK_CUR);
        // 创建 ChatGLMTokenizer 对象
        tokenizer = std::make_unique<ChatGLMTokenizer>(serialized_model_proto);

        // 创建 ChatGLMForCausalLM 模型对象
        model = std::make_unique<ChatGLMForCausalLM>(config);
        // 加载模型数据
        model->load(loader);
    // 如果模型类型是CHATGLM2或CHATGLM3，则执行以下操作
    } else if (model_type == ModelType::CHATGLM2 || model_type == ModelType::CHATGLM3) {
        // 检查模型版本是否为1，如果不是则输出错误信息
        CHATGLM_CHECK(version == 1) << "only support version 1 for now but got " << version;

        // 加载配置信息
        ModelConfig config(model_type, loader.read_basic<ConfigRecordV2>());

        // 加载分词器
        int proto_size = loader.read_basic<int>();
        // 从映射文件中读取序列化的模型协议
        std::string_view serialized_model_proto((char *)mapped_file->data + loader.tell(), proto_size);
        loader.seek(proto_size, SEEK_CUR);

        // 如果模型类型是CHATGLM2，则创建ChatGLM2Tokenizer和ChatGLM2ForCausalLM对象
        if (model_type == ModelType::CHATGLM2) {
            tokenizer = std::make_unique<ChatGLM2Tokenizer>(serialized_model_proto);
            model = std::make_unique<ChatGLM2ForCausalLM>(config);
        } else {
            // 如果模型类型是CHATGLM3，则创建ChatGLM3Tokenizer和ChatGLM3ForCausalLM对象
            auto chatglm3_tokenizer = std::make_unique<ChatGLM3Tokenizer>(serialized_model_proto);
            config.extra_eos_token_ids = {chatglm3_tokenizer->observation_token_id, chatglm3_tokenizer->user_token_id};
            tokenizer = std::move(chatglm3_tokenizer);
            model = std::make_unique<ChatGLM3ForCausalLM>(config);
        }

        // 加载模型
        model->load(loader);
    // 如果模型类型是BAICHUAN7B，则执行以下操作
    } else if (model_type == ModelType::BAICHUAN7B) {
        // 检查模型版本是否为1，如果不是则输出错误信息
        CHATGLM_CHECK(version == 1) << "only support version 1 for now but got " << version;

        // 加载配置信息
        ModelConfig config(model_type, loader.read_basic<ConfigRecordV1>());
        // 设置norm_eps为1e-6
        config.norm_eps = 1e-6;

        // 加载分词器
        int proto_size = loader.read_basic<int>();
        // 从映射文件中读取序列化的模型协议
        std::string_view serialized_model_proto((char *)mapped_file->data + loader.tell(), proto_size);
        loader.seek(proto_size, SEEK_CUR);
        // 创建BaichuanTokenizer对象
        tokenizer = std::make_unique<BaichuanTokenizer>(serialized_model_proto);

        // 创建Baichuan7BForCausalLM对象
        model = std::make_unique<Baichuan7BForCausalLM>(config);
        // 加载模型
        model->load(loader);
    // 如果模型类型为 BAICHUAN13B
    } else if (model_type == ModelType::BAICHUAN13B) {
        // 检查版本是否为1，如果不是则抛出异常
        CHATGLM_CHECK(version == 1) << "only support version 1 for now but got " << version;

        // 加载配置信息
        ModelConfig config(model_type, loader.read_basic<ConfigRecordV1>());
        config.norm_eps = 1e-6;

        // 加载分词器
        int proto_size = loader.read_basic<int>();
        // 从内存映射文件中读取序列化的模型协议
        std::string_view serialized_model_proto((char *)mapped_file->data + loader.tell(), proto_size);
        loader.seek(proto_size, SEEK_CUR);
        // 创建 BaichuanTokenizer 对象
        tokenizer = std::make_unique<BaichuanTokenizer>(serialized_model_proto);

        // 加载模型
        model = std::make_unique<Baichuan13BForCausalLM>(config);
        model->load(loader);
    // 如果模型类型为 INTERNLM
    } else if (model_type == ModelType::INTERNLM) {
        // 检查版本是否为1，如果不是则抛出异常
        CHATGLM_CHECK(version == 1) << "only support version 1 for now but got " << version;

        // 加载配置信息
        ModelConfig config(model_type, loader.read_basic<ConfigRecordV1>());
        config.norm_eps = 1e-6;

        // 加载分词器
        int proto_size = loader.read_basic<int>();
        // 从内存映射文件中读取序列化的模型协议
        std::string_view serialized_model_proto((char *)mapped_file->data + loader.tell(), proto_size);
        loader.seek(proto_size, SEEK_CUR);
        // 创建 InternLMTokenizer 对象
        tokenizer = std::make_unique<InternLMTokenizer>(serialized_model_proto);

        // 加载模型
        if (config.hidden_size == 4096) {
            // 根据配置信息中隐藏层大小选择加载 InternLM7BForCausalLM 或 InternLM20BForCausalLM 模型
            model = std::make_unique<InternLM7BForCausalLM>(config);
        } else {
            model = std::make_unique<InternLM20BForCausalLM>(config);
        }
        model->load(loader);
    } else {
        // 如果模型类型无效，则抛出异常
        CHATGLM_THROW << "invalid model type " << (int)model_type;
    }
// 生成函数，根据输入的 ID 列表、生成配置和流处理器生成输出 ID 列表
std::vector<int> Pipeline::generate(const std::vector<int> &input_ids, const GenerationConfig &gen_config,
                                    BaseStreamer *streamer) const {
    // 调用模型对象的 generate 方法生成输出 ID 列表
    std::vector<int> output_ids = model->generate(input_ids, gen_config, streamer);
    // 从输出 ID 列表中截取新的输出 ID 列表
    std::vector<int> new_output_ids(output_ids.begin() + input_ids.size(), output_ids.end());
    // 返回新的输出 ID 列表
    return new_output_ids;
}

// 生成函数，根据输入的提示字符串、生成配置和流处理器生成输出字符串
std::string Pipeline::generate(const std::string &prompt, const GenerationConfig &gen_config,
                               BaseStreamer *streamer) const {
    // 使用分词器对象对提示字符串进行编码，限制最大上下文长度
    std::vector<int> input_ids = tokenizer->encode(prompt, gen_config.max_context_length);
    // 调用上一个 generate 函数生成输出 ID 列表
    std::vector<int> new_output_ids = generate(input_ids, gen_config, streamer);
    // 使用分词器对象对新的输出 ID 列表进行解码，得到输出字符串
    std::string output = tokenizer->decode(new_output_ids);
    // 返回输出字符串
    return output;
}

// 聊天函数，根据输入的聊天消息列表、生成配置和流处理器生成聊天消息
ChatMessage Pipeline::chat(const std::vector<ChatMessage> &messages, const GenerationConfig &gen_config,
                           BaseStreamer *streamer) const {
    // 使用分词器对象对聊天消息列表进行编码，限制最大上下文长度
    std::vector<int> input_ids = tokenizer->encode_messages(messages, gen_config.max_context_length);
    // 调用上一个 generate 函数生成输出 ID 列表
    std::vector<int> new_output_ids = generate(input_ids, gen_config, streamer);
    // 使用分词器对象对新的输出 ID 列表进行解码，得到聊天消息
    ChatMessage output = tokenizer->decode_message(new_output_ids);
    // 返回聊天消息
    return output;
}

} // namespace chatglm
```