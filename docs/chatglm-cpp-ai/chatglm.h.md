# `chatglm.cpp\chatglm.h`

```cpp
#pragma once

#include <cmath> // 包含数学函数库
#include <ggml.h> // 包含 ggml 库
#include <iomanip> // 控制输出格式库
#include <sentencepiece_processor.h> // 包含 sentencepiece 处理器库
#include <sstream> // 提供了对内存数据流的支持
#include <unordered_map> // 包含无序映射库

#ifdef GGML_USE_METAL
#include <ggml-metal.h> // 如果定义了 GGML_USE_METAL，则包含 ggml-metal 库
#endif

namespace chatglm {

// ===== common =====

static constexpr size_t MB = 1024 * 1024; // 定义常量 MB 为 1024 * 1024

class LogMessageFatal {
  public:
    LogMessageFatal(const char *file, int line) { oss_ << file << ':' << line << ' '; } // 构造函数，记录文件名和行号
    [[noreturn]] ~LogMessageFatal() noexcept(false) { throw std::runtime_error(oss_.str()); } // 析构函数，抛出运行时错误
    std::ostringstream &stream() { return oss_; } // 返回字符串流对象

  private:
    std::ostringstream oss_; // 字符串流对象
};

#define CHATGLM_THROW ::chatglm::LogMessageFatal(__FILE__, __LINE__).stream() // 定义宏，抛出致命错误
#define CHATGLM_CHECK(cond)                                                                                            \
    if (!(cond))                                                                                                       \
    CHATGLM_THROW << "check failed (" #cond ") " // 定义宏，检查条件是否成立，否则抛出错误

#define CHATGLM_CHECK_CUDA(call)                                                                                       \
    do {                                                                                                               \
        cudaError_t error = (call);                                                                                    \
        CHATGLM_CHECK(error == cudaSuccess) << "CUDA error: " << cudaGetErrorString(error);                            \
    } while (0) // 定义宏，检查 CUDA 调用是否成功，否则抛出 CUDA 错误信息

std::string to_string(ggml_tensor *tensor, bool with_data = true); // 将 ggml_tensor 转换为字符串

ggml_tensor *tensor_assign_buffers(ggml_tensor *tensor); // 分配 ggml_tensor 的缓冲区

ggml_tensor *tensor_to_device(ggml_tensor *tensor); // 将 ggml_tensor 移动到设备

ggml_tensor *tensor_to_cpu(ggml_tensor *tensor); // 将 ggml_tensor 移动到 CPU

enum class ModelType {
    CHATGLM = 1,
    CHATGLM2 = 2,
    CHATGLM3 = 3,
    BAICHUAN7B = 1024,
    BAICHUAN13B = 1025,
    INTERNLM = 1280,
};

std::string to_string(ModelType model_type); // 将 ModelType 转换为字符串

// For compatibility
struct ConfigRecordV1 {
    // common attributes
    ggml_type dtype; // ggml 类型
    # 定义词汇表大小
    int vocab_size;
    # 定义隐藏层大小
    int hidden_size;
    # 定义注意力头的数量
    int num_attention_heads;
    # 定义隐藏层的数量
    int num_hidden_layers;
    # 定义中间层大小
    int intermediate_size;
    # 用于序列生成的最大长度
    int max_length;
    # 用于分词器的开始标记的标识符
    int bos_token_id;
    # 用于分词器的结束标记的标识符
    int eos_token_id;
    # 用于分词器的填充标记的标识符
    int pad_token_id;
    # 用于分词器的分隔标记的标识符
    int sep_token_id;
// 结构体 ConfigRecordV2 继承自 ConfigRecordV1，用于兼容性
struct ConfigRecordV2 : public ConfigRecordV1 {
    // 新增属性 num_kv_heads
    int num_kv_heads;
};

// 用于保存 ModelConfig 的键值记录
class ModelConfig {
  public:
    // 默认构造函数
    ModelConfig() = default;

    // 构造函数，初始化 ModelConfig 的各个属性
    ModelConfig(ModelType model_type, ggml_type dtype, int vocab_size, int hidden_size, int num_attention_heads,
                int num_kv_heads, int num_hidden_layers, int intermediate_size, float norm_eps, int max_length,
                int bos_token_id, int eos_token_id, int pad_token_id, int sep_token_id,
                std::vector<int> extra_eos_token_ids)
        : model_type(model_type), dtype(dtype), vocab_size(vocab_size), hidden_size(hidden_size),
          num_attention_heads(num_attention_heads), num_kv_heads(num_kv_heads), num_hidden_layers(num_hidden_layers),
          intermediate_size(intermediate_size), norm_eps(norm_eps), max_length(max_length), bos_token_id(bos_token_id),
          eos_token_id(eos_token_id), pad_token_id(pad_token_id), sep_token_id(sep_token_id),
          extra_eos_token_ids(std::move(extra_eos_token_ids)) {}

    // 构造函数，根据 ConfigRecordV1 初始化 ModelConfig
    ModelConfig(ModelType model_type, const ConfigRecordV1 &rec)
        : ModelConfig(model_type, rec.dtype, rec.vocab_size, rec.hidden_size, rec.num_attention_heads,
                      rec.num_attention_heads, rec.num_hidden_layers, rec.intermediate_size, 1e-5, rec.max_length,
                      rec.bos_token_id, rec.eos_token_id, rec.pad_token_id, rec.sep_token_id, {}) {}

    // 构造函数，根据 ConfigRecordV2 初始化 ModelConfig
    ModelConfig(ModelType model_type, const ConfigRecordV2 &rec)
        : ModelConfig(model_type, rec.dtype, rec.vocab_size, rec.hidden_size, rec.num_attention_heads, rec.num_kv_heads,
                      rec.num_hidden_layers, rec.intermediate_size, 1e-5, rec.max_length, rec.bos_token_id,
                      rec.eos_token_id, rec.pad_token_id, rec.sep_token_id, {}) {}

    // 返回 model_type 的字符串表示
    std::string model_type_name() const { return to_string(model_type); }

  public:
    // ModelConfig 的属性
    ModelType model_type;
    ggml_type dtype;
    int vocab_size;
    # 定义隐藏层的大小
        int hidden_size;
    # 定义注意力头的数量
        int num_attention_heads;
    # 定义键值头的数量
        int num_kv_heads;
    # 定义隐藏层的数量
        int num_hidden_layers;
    # 定义中间层的大小
        int intermediate_size;
    # 定义归一化的 epsilon 值
        float norm_eps;
    # 定义最大长度
        int max_length;
    # 定义起始标记的 token id
        int bos_token_id;
    # 定义结束标记的 token id
        int eos_token_id;
    # 定义填充标记的 token id
        int pad_token_id;
    # 定义分隔标记的 token id
        int sep_token_id;
    # 定义额外的结束标记的 token ids
        std::vector<int> extra_eos_token_ids;
// 结构体 FunctionMessage，用于表示函数消息，包含函数名和参数
struct FunctionMessage {
    std::string name; // 函数名
    std::string arguments; // 参数

    FunctionMessage() = default; // 默认构造函数
    FunctionMessage(std::string name, std::string arguments) : name(std::move(name)), arguments(std::move(arguments)) {} // 带参数的构造函数

    // 重载输出流操作符，用于输出 FunctionMessage 对象的信息
    friend std::ostream &operator<<(std::ostream &os, const FunctionMessage &self) {
        return os << "FunctionMessage(name=" << std::quoted(self.name) << ", arguments=" << std::quoted(self.arguments) << ")";
    }
};

// 结构体 CodeMessage，用于表示代码消息，包含输入的代码
struct CodeMessage {
    std::string input; // 输入的代码

    CodeMessage() = default; // 默认构造函数
    CodeMessage(std::string input) : input(std::move(input)) {} // 带参数的构造函数

    // 重载输出流操作符，用于输出 CodeMessage 对象的信息
    friend std::ostream &operator<<(std::ostream &os, const CodeMessage &self) {
        return os << "CodeMessage(input=" << std::quoted(self.input) << ")";
    }
};

// 结构体 ToolCallMessage，用于表示工具调用消息，包含类型、函数消息和代码消息
struct ToolCallMessage {
    std::string type; // 类型
    FunctionMessage function; // 函数消息
    CodeMessage code; // 代码消息

    static const std::string TYPE_FUNCTION; // 函数类型
    static const std::string TYPE_CODE; // 代码类型

    // 构造函数，接受函数消息作为参数
    ToolCallMessage(FunctionMessage function) : type(TYPE_FUNCTION), function(std::move(function)) {}

    // 构造函数，接受代码消息作为参数
    ToolCallMessage(CodeMessage code) : type(TYPE_CODE), code(std::move(code)) {}

    // 重载输出流操作符，用于输出 ToolCallMessage 对象的信息
    friend std::ostream &operator<<(std::ostream &os, const ToolCallMessage &self) {
        return os << "ToolCallMessage(type=" << std::quoted(self.type) << ", function=" << self.function << ", code=" << self.code << ")";
    }
};

// 结构体 ChatMessage，用于表示聊天消息，包含角色、内容和工具调用消息列表
struct ChatMessage {
    std::string role; // 角色
    std::string content; // 内容
    std::vector<ToolCallMessage> tool_calls; // 工具调用消息列表

    static const std::string ROLE_USER; // 用户角色
    static const std::string ROLE_ASSISTANT; // 助手角色
    static const std::string ROLE_SYSTEM; // 系统角色
    static const std::string ROLE_OBSERVATION; // 观察角色

    ChatMessage() = default; // 默认构造函数
    // 带参数的构造函数，接受角色、内容和工具调用消息列表作为参数
    ChatMessage(std::string role, std::string content, std::vector<ToolCallMessage> tool_calls = {})
        : role(std::move(role)), content(std::move(content)), tool_calls(std::move(tool_calls)) {}
    # 重载输出流操作符，用于将 ChatMessage 对象输出到流中
    friend std::ostream &operator<<(std::ostream &os, const ChatMessage &self) {
        # 输出角色信息，并使用 quoted 函数确保内容被引用
        os << "ChatMessage(role=" << std::quoted(self.role) << ", content=" << std::quoted(self.content)
           << ", tool_calls=[";
        # 遍历工具调用列表，输出每个工具调用
        for (size_t i = 0; i < self.tool_calls.size(); i++) {
            # 如果不是第一个工具调用，则输出逗号和空格
            os << (i > 0 ? ", " : "") << self.tool_calls[i];
        }
        # 输出结束符号并返回输出流
        return os << "])";
    }
// 结束类定义
};

// 基础分词器类
class BaseTokenizer {
  public:
    // 虚析构函数
    virtual ~BaseTokenizer() = default;

    // 编码文本并返回编码结果
    virtual std::vector<int> encode(const std::string &text, int max_length) const = 0;

    // 解码编码结果并返回原始文本
    virtual std::string decode(const std::vector<int> &ids) const = 0;

    // 编码消息并返回编码结果
    virtual std::vector<int> encode_messages(const std::vector<ChatMessage> &messages, int max_length) const = 0;

    // 解码消息并返回解码结果
    virtual ChatMessage decode_message(const std::vector<int> &ids) const {
        return {ChatMessage::ROLE_ASSISTANT, decode(ids)};
    }

  protected:
    // 检查聊天消息
    static void check_chat_messages(const std::vector<ChatMessage> &messages);
};

// 自定义删除器结构体
struct ggml_context_deleter_t {
    void operator()(ggml_context *ctx) const noexcept { ggml_free(ctx); }
};

// 使用自定义删除器的智能指针类型
using unique_ggml_context_t = std::unique_ptr<ggml_context, ggml_context_deleter_t>;

// 创建并返回智能指针
static inline unique_ggml_context_t make_unique_ggml_context(size_t mem_size, void *mem_buffer, bool no_alloc) {
    return unique_ggml_context_t(ggml_init({mem_size, mem_buffer, no_alloc}));
}

#ifdef GGML_USE_METAL
// Metal上下文删除器结构体
struct ggml_metal_context_deleter_t {
    void operator()(ggml_metal_context *ctx) const noexcept { ggml_metal_free(ctx); }
};

// 使用Metal上下文删除器的智能指针类型
using unique_ggml_metal_context_t = std::unique_ptr<ggml_metal_context, ggml_metal_context_deleter_t>;

// 创建并返回Metal上下文智能指针
static inline unique_ggml_metal_context_t make_unique_ggml_metal_context(int n_cb) {
    return unique_ggml_metal_context_t(ggml_metal_init(n_cb));
}
#endif

// 参考链接：https://stackoverflow.com/questions/11149665/c-vector-that-doesnt-initialize-its-members
// 未初始化的字符结构体
struct uninitialized_char {
    char m;
    uninitialized_char() {}
};

// 执行图计算的辅助函数
void ggml_graph_compute_helper(std::vector<uninitialized_char> &buf, ggml_cgraph *graph, int n_threads);

// 模型上下文结构体
struct ModelContext {
    ggml_type dtype;
    unique_ggml_context_t ctx_w;  // 权重
    unique_ggml_context_t ctx_kv; // 键值缓存
    unique_ggml_context_t ctx_b;  // 缓冲区
#ifdef GGML_USE_METAL
    unique_ggml_metal_context_t ctx_metal;
#endif
    ggml_cgraph gf;
    ggml_scratch scratch;
    // 用于存储 BLAS 缓冲区的向量，未初始化
    std::vector<uninitialized_char> compute_buffer; // BLAS buffer
    // 用于存储中间张量缓冲区的向量，未初始化
    std::vector<uninitialized_char> scratch_buffer; // intermediate tensor buffer
    // 用于存储映射权重的字符串视图
    std::string_view weight_buffer;                 // mapped weight
    // 用于存储图计算临时缓冲区的向量，未初始化
    std::vector<uninitialized_char> work_buffer;    // temporary buffer for graph computing

    // 初始化设备上下文
    void init_device_context();
};

// 嵌入层类
class Embedding {
  public:
    Embedding() : weight(nullptr) {} // 初始化权重为空指针
    Embedding(ModelContext *ctx, int num_embeddings, int embedding_dim)
        : weight(ggml_new_tensor_2d(ctx->ctx_w.get(), ctx->dtype, embedding_dim, num_embeddings)) {} // 使用给定的上下文、嵌入数量和维度创建权重张量

    ggml_tensor *forward(ModelContext *ctx, ggml_tensor *input) const; // 前向传播函数声明

  public:
    ggml_tensor *weight; // 权重张量指针
};

// 线性层类
class Linear {
  public:
    Linear() : weight(nullptr), bias(nullptr) {} // 初始化权重和偏置为空指针
    Linear(ModelContext *ctx, int in_features, int out_features, bool use_bias = true)
        : weight(ggml_new_tensor_2d(ctx->ctx_w.get(), ctx->dtype, in_features, out_features)),
          bias(use_bias ? ggml_new_tensor_1d(ctx->ctx_w.get(), GGML_TYPE_F32, out_features) : nullptr) {} // 使用给定的上下文、输入特征数、输出特征数和是否使用偏置创建权重和偏置张量

    int in_features() const { return weight->ne[0]; } // 返回输入特征数
    int out_features() const { return weight->ne[1]; } // 返回输出特征数

    ggml_tensor *forward(ModelContext *ctx, ggml_tensor *input) const; // 前向传播函数声明

  public:
    ggml_tensor *weight; // 权重张量指针 [out_features, in_features]
    ggml_tensor *bias;   // 偏置张量指针 [out_features]
};

// 层归一化类
class LayerNorm {
  public:
    LayerNorm() = default; // 默认构造函数
    LayerNorm(ModelContext *ctx, int normalized_shape, bool inplace = true, float eps = 1e-5f)
        : weight(ggml_new_tensor_1d(ctx->ctx_w.get(), GGML_TYPE_F32, normalized_shape)),
          bias(ggml_new_tensor_1d(ctx->ctx_w.get(), GGML_TYPE_F32, normalized_shape)), inplace(inplace), eps(eps) {} // 使用给定的上下文、归一化形状、是否原地操作和 epsilon 创建权重和偏置张量

    ggml_tensor *forward(ModelContext *ctx, ggml_tensor *input) const; // 前向传播函数声明

  public:
    ggml_tensor *weight; // 权重张量指针 [normalized_shape]
    ggml_tensor *bias;   // 偏置张量指针 [normalized_shape]
    bool inplace; // 是否原地操作
    float eps; // epsilon 值
};

// RMS 归一化类
class RMSNorm {
  public:
    RMSNorm() = default; // 默认构造函数
    RMSNorm(ModelContext *ctx, int normalized_shape, bool inplace = true, float eps = 1e-5f)
        : weight(ggml_new_tensor_1d(ctx->ctx_w.get(), GGML_TYPE_F32, normalized_shape)), inplace(inplace), eps(eps) {} // 使用给定的上下文、归一化形状、是否原地操作和 epsilon 创建权重张量

    ggml_tensor *forward(ModelContext *ctx, ggml_tensor *input) const; // 前向传播函数声明

  public:
    ggml_tensor *weight; // 权重张量指针 [normalized_shape]
    bool inplace; // 是否原地操作
    # 定义一个浮点数变量 eps
};

// 激活函数类型枚举
enum class ActivationType {
    GELU,  // GELU 激活函数
    SILU,  // SILU 激活函数
};

// 应用激活函数到输入张量中
template <ActivationType ACT_TYPE>
static inline ggml_tensor *apply_activation_inplace(ggml_context *ctx, ggml_tensor *hidden_states) {
    // 静态断言，确保激活函数类型为 GELU 或 SILU
    static_assert(ACT_TYPE == ActivationType::GELU || ACT_TYPE == ActivationType::SILU);
    if constexpr (ACT_TYPE == ActivationType::GELU) {
        // 如果激活函数类型为 GELU，则应用 GELU 激活函数到隐藏状态张量中
        hidden_states = tensor_assign_buffers(ggml_gelu_inplace(ctx, hidden_states));
    } else if constexpr (ACT_TYPE == ActivationType::SILU) {
        // 如果激活函数类型为 SILU，则应用 SILU 激活函数到隐藏状态张量中
        hidden_states = tensor_assign_buffers(ggml_silu_inplace(ctx, hidden_states));
    } else {
        // 如果激活函数类型未知，则抛出异常
        CHATGLM_THROW << "Unknown activation type " << (int)ACT_TYPE;
    }
    return hidden_states;
}

// 基础 MLP 类模板
template <ActivationType ACT_TYPE>
class BasicMLP {
  public:
    BasicMLP() = default;
    BasicMLP(ModelContext *ctx, int hidden_size, int intermediate_size)
        : dense_h_to_4h(ctx, hidden_size, intermediate_size), dense_4h_to_h(ctx, intermediate_size, hidden_size) {}

    // 前向传播函数
    ggml_tensor *forward(ModelContext *ctx, ggml_tensor *hidden_states) const {
        ggml_context *gctx = ctx->ctx_b.get();
        // 将隐藏状态张量传递给第一个全连接层
        hidden_states = dense_h_to_4h.forward(ctx, hidden_states);
        // 应用激活函数到隐藏状态张量中
        hidden_states = apply_activation_inplace<ACT_TYPE>(gctx, hidden_states);
        // 将隐藏状态张量传递给第二个全连接层
        hidden_states = dense_4h_to_h.forward(ctx, hidden_states);
        return hidden_states;
    }

  public:
    Linear dense_h_to_4h;  // 第一个全连接层
    Linear dense_4h_to_h;  // 第二个全连接层
};

// 基础 GLU 类模板
template <ActivationType ACT_TYPE, bool USE_BIAS>
class BasicGLU {
  public:
    BasicGLU() = default;
    BasicGLU(ModelContext *ctx, int hidden_size, int intermediate_size)
        : gate_proj(ctx, hidden_size, intermediate_size, USE_BIAS),
          up_proj(ctx, hidden_size, intermediate_size, USE_BIAS),
          down_proj(ctx, intermediate_size, hidden_size, USE_BIAS) {}
    ggml_tensor *forward(ModelContext *ctx, ggml_tensor *hidden_states) const {
        // 获取上下文对象
        ggml_context *gctx = ctx->ctx_b.get();
        // 计算门控投影
        ggml_tensor *gate = gate_proj.forward(ctx, hidden_states);
        // 应用激活函数到门控投影
        gate = apply_activation_inplace<ACT_TYPE>(gctx, gate);
        // 计算上游投影
        hidden_states = up_proj.forward(ctx, hidden_states);
        // 将上游投影与门控投影相乘，并将结果赋值给 hidden_states
        hidden_states = tensor_assign_buffers(ggml_mul_inplace(gctx, hidden_states, gate));
        // 计算下游投影
        hidden_states = down_proj.forward(ctx, hidden_states);
        // 返回最终的隐藏状态
        return hidden_states;
    }

  public:
    // 定义门控投影
    Linear gate_proj;
    // 定义上游投影
    Linear up_proj;
    // 定义下游投影
    Linear down_proj;
};

// 定义一个结构体 CausalContextMasker，包含一个重载的操作符()，用于处理注意力分数
struct CausalContextMasker {
    // 重载的操作符()，接受模型上下文、注意力分数和过去步数作为参数，返回处理后的注意力分数
    ggml_tensor *operator()(ModelContext *ctx, ggml_tensor *attn_scores, int n_past) const {
        // 调用 ggml_diag_mask_inf_inplace 函数对注意力分数进行处理，并返回处理后的结果
        return tensor_assign_buffers(ggml_diag_mask_inf_inplace(ctx->ctx_b.get(), attn_scores, n_past));
    }
};

// 定义一个枚举类型 RopeType，包含三种绳索类型
enum RopeType {
    ROPE_TYPE_DEFAULT = 0,
    ROPE_TYPE_NEOX = 2,
    ROPE_TYPE_CHATGLM = 4,
};

// 定义一个结构体 NoopRoper，包含一个重载的操作符()，用于处理模型上下文、两个张量和上下文长度
struct NoopRoper {
    // 重载的操作符()，接受模型上下文、两个张量和上下文长度作为参数，返回第一个张量
    ggml_tensor *operator()(ModelContext *ctx, ggml_tensor *a, ggml_tensor *b, int n_ctx) const { return a; }
};

// 定义一个模板结构体 BasicRoper，包含一个重载的操作符()，用于处理模型上下文、两个张量和上下文长度
template <RopeType MODE, int DIM_SCALE>
struct BasicRoper {
    // 重载的操作符()，接受模型上下文、两个张量和上下文长度作为参数，返回处理后的第一个张量
    ggml_tensor *operator()(ModelContext *ctx, ggml_tensor *a, ggml_tensor *b, int n_ctx) const {
        // 获取模型上下文指针
        ggml_context *gctx = ctx->ctx_b.get();
#ifdef GGML_USE_CUBLAS
        // 如果张量 a 不是连续的，则调用 ggml_cont 函数对其进行处理
        if (!ggml_is_contiguous(a)) {
            a = tensor_assign_buffers(ggml_cont(gctx, a));
        }
#endif
        // 获取张量头部大小和绳索维度
        const int head_size = a->ne[0];
        const int rope_dim = head_size / DIM_SCALE;
        // 调用 ggml_rope_inplace 函数对张量 a 进行处理，并返回处理后的结果
        a = tensor_assign_buffers(ggml_rope_inplace(gctx, a, b, rope_dim, MODE, n_ctx)); // [qlen, heads, head_size]

        return a;
    }
};

// 定义一个结构体 GLMRoper
struct GLMRoper {
        // 根据给定的上下文、两个张量和上下文长度，执行操作并返回结果张量
        ggml_tensor *operator()(ModelContext *ctx, ggml_tensor *a, ggml_tensor *b, int n_ctx) const {
            // 张量 a（激活）的形状为 [qlen, heads, head_size]
            // 张量 b（位置ID）的形状为 [2 * qlen]
            ggml_context *gctx = ctx->ctx_b.get();

            // 定义头部大小、头部数量、qlen和绳子维度
            const int head_size = a->ne[0];
            const int num_heads = a->ne[1];
            const int qlen = a->ne[2];
            const int rope_dim = head_size / 2;

            // 从张量 b 中创建两个视图张量 b1 和 b2
            ggml_tensor *b1 = ggml_view_1d(gctx, b, qlen, 0);
            ggml_tensor *b2 = ggml_view_1d(gctx, b, qlen, qlen * ggml_element_size(b));

            // 从张量 a 中创建两个三维视图张量 a1 和 a2，分别表示绳子的两部分
            ggml_tensor *a1 = ggml_view_3d(gctx, a, head_size / 2, num_heads, qlen, a->nb[1], a->nb[2], 0);
            ggml_tensor *a2 = ggml_view_3d(gctx, a, head_size / 2, num_heads, qlen, a->nb[1], a->nb[2],
                                           head_size / 2 * ggml_element_size(a));

            // 初始化绳子的两部分的张量
            ggml_tensor *a1_rope = a1;
            ggml_tensor *a2_rope = a2;
#ifdef GGML_USE_CUBLAS
        // 如果定义了 GGML_USE_CUBLAS，则对 a1_rope 和 a2_rope 进行缓冲区分配
        a1_rope = tensor_assign_buffers(ggml_cont(gctx, a1_rope));
        a2_rope = tensor_assign_buffers(ggml_cont(gctx, a2_rope));
#endif

        // 对 a1_rope 进行缓冲区分配，并进行 ROPE 操作
        a1_rope = tensor_assign_buffers(
            ggml_rope_inplace(gctx, a1_rope, b1, rope_dim, ROPE_TYPE_NEOX, n_ctx)); // [qlen, heads, head_size/2]
        // 对 a2_rope 进行缓冲区分配，并进行 ROPE 操作
        a2_rope = tensor_assign_buffers(
            ggml_rope_inplace(gctx, a2_rope, b2, rope_dim, ROPE_TYPE_NEOX, n_ctx)); // [qlen, heads, head_size/2]

#ifdef GGML_USE_CUBLAS
        // 如果定义了 GGML_USE_CUBLAS，则对 a1_rope 和 a2_rope 进行复制操作
        a1_rope = ggml_cpy(gctx, a1_rope, a1);
        a2_rope = ggml_cpy(gctx, a2_rope, a2);
#endif
        // 构建前向扩展操作，对 a1_rope 进行操作
        ggml_build_forward_expand(&ctx->gf, a1_rope);
        // 构建前向扩展操作，对 a2_rope 进行操作
        ggml_build_forward_expand(&ctx->gf, a2_rope);

        // 返回 a
        return a;
    }
};

// 定义模板类 BasicAttention
template <bool USE_QKV_BIAS, bool USE_DENSE_BIAS, bool INTERLEAVED_QKV, typename Roper, bool USE_ALIBI,
          typename ContextMasker>
class BasicAttention {
  public:
    // 默认构造函数
    BasicAttention() = default;
    // 构造函数，初始化参数和成员变量
    BasicAttention(ModelContext *ctx, int hidden_size, int num_attention_heads, int num_kv_heads, int max_length)
        : num_attention_heads(num_attention_heads), num_kv_heads(num_kv_heads),
          // 初始化 query_key_value
          query_key_value(ctx, hidden_size, hidden_size + 2 * (hidden_size / num_attention_heads) * num_kv_heads,
                          USE_QKV_BIAS),
          // 初始化 dense
          dense(ctx, hidden_size, hidden_size, USE_DENSE_BIAS),
          // 初始化 k_cache
          k_cache(ggml_new_tensor_3d(ctx->ctx_kv.get(), GGML_TYPE_F16, hidden_size / num_attention_heads, max_length,
                                     num_kv_heads)),
          // 初始化 v_cache
          v_cache(ggml_new_tensor_3d(ctx->ctx_kv.get(), GGML_TYPE_F16, max_length, hidden_size / num_attention_heads,
                                     num_kv_heads)) {}

    }

  public:
    // 定义成员变量 num_attention_heads 和 num_kv_heads
    int num_attention_heads;
    int num_kv_heads;
    // 定义 Linear 类型的成员变量 query_key_value 和 dense
    Linear query_key_value;
    Linear dense;
    // 定义 ggml_tensor 类型的成员变量 k_cache，表示 [kv_heads, max_len, head_size]
    ggml_tensor *k_cache;
    // 定义 ggml_tensor 类型的成员变量 v_cache，表示 [kv_heads, head_size, max_len]

  private:
    // 定义模板参数 Roper 和 ContextMasker 的成员变量 roper_ 和 context_masker_
    Roper roper_;
    ContextMasker context_masker_;
// 定义一个模板类 BasicBlock，包含 Norm、Attention 和 MLP 三个模板参数
template <typename Norm, typename Attention, typename MLP>
class BasicBlock {
  public:
    // 默认构造函数
    BasicBlock() = default;
    // 构造函数，初始化各个成员变量
    BasicBlock(ModelContext *ctx, int hidden_size, int num_attention_heads, int num_kv_heads, int intermediate_size,
               int max_length, float norm_eps)
        : input_layernorm(ctx, hidden_size, false, norm_eps),
          attention(ctx, hidden_size, num_attention_heads, num_kv_heads, max_length),
          post_attention_layernorm(ctx, hidden_size, false, norm_eps), mlp(ctx, hidden_size, intermediate_size) {}

    // 前向传播函数
    ggml_tensor *forward(ModelContext *ctx, ggml_tensor *hidden_states, ggml_tensor *position_ids, int n_past,
                         int n_ctx) const {
        ggml_context *gctx = ctx->ctx_b.get();

        ggml_tensor *residual = hidden_states;
        // 输入层归一化
        hidden_states = input_layernorm.forward(ctx, hidden_states);
        // 注意力机制
        hidden_states = attention.forward(ctx, hidden_states, position_ids, n_past, n_ctx);
        // 残差连接
        hidden_states = tensor_assign_buffers(ggml_add_inplace(gctx, hidden_states, residual));

        residual = hidden_states;
        // 后注意力层归一化
        hidden_states = post_attention_layernorm.forward(ctx, hidden_states);
        // 多层感知机
        hidden_states = mlp.forward(ctx, hidden_states);
        // 残差连接
        hidden_states = tensor_assign_buffers(ggml_add_inplace(gctx, hidden_states, residual));

        return hidden_states;
    }

  protected:
    // 受保护的构造函数，用于初始化成员变量
    BasicBlock(Norm input_layernorm, Attention attention, Norm post_attention_layernorm, MLP mlp)
        : input_layernorm(input_layernorm), attention(attention), post_attention_layernorm(post_attention_layernorm),
          mlp(mlp) {}

  public:
    // 成员变量：输入层归一化、注意力机制、后注意力层归一化、多层感知机
    Norm input_layernorm;
    Attention attention;
    Norm post_attention_layernorm;
    MLP mlp;
};

// 定义一个结构体 NoopPositionIdsGenerator，重载 () 运算符
struct NoopPositionIdsGenerator {
    // 生成位置编码的函数，返回空指针
    ggml_tensor *operator()(ggml_context *ctx, int qlen, int n_past, int n_ctx) const { return nullptr; }
};

// 定义一个结构体 BasicPositionIdsGenerator
    # 通过给定的上下文、查询长度、过去长度和上下文长度，生成位置编码的张量
    ggml_tensor *operator()(ggml_context *ctx, int qlen, int n_past, int n_ctx) const {
        # 创建一个一维张量用于存储位置编码
        ggml_tensor *position_ids = ggml_new_tensor_1d(ctx, GGML_TYPE_I32, qlen);
        # 遍历查询长度，为每个位置计算位置编码并存储到张量中
        for (int i = 0; i < qlen; i++) {
            ((int *)position_ids->data)[i] = n_past + i;
        }
        # 返回位置编码张量
        return position_ids;
    }
};

// 定义一个结构体，用于生成位置 ID 的张量
struct GLMPositionIdsGenerator {
    ggml_tensor *operator()(ggml_context *ctx, int qlen, int n_past, int n_ctx) const {
        // 创建一个一维整型张量，长度为 qlen * 2
        ggml_tensor *position_ids = ggml_new_tensor_1d(ctx, GGML_TYPE_I32, qlen * 2);
        // 遍历生成位置 ID 的张量
        for (int i = 0; i < qlen; i++) {
            const int p = n_past + i;
            // 将位置 ID 存入张量中
            ((int *)position_ids->data)[i] = std::min(p, n_ctx - 2);
            ((int *)position_ids->data)[qlen + i] = std::max(p - (n_ctx - 2), 0);
        }
        // 返回生成的位置 ID 张量
        return position_ids;
    }
};

// 定义一个模板类 BasicModel，包含 Block、Norm 和 PositionIdsGenerator 三个模板参数
template <typename Block, typename Norm, typename PositionIdsGenerator>
class BasicModel {
  public:
    // 默认构造函数
    BasicModel() = default;

    // 构造函数，接受 Embedding、layers 和 final_layernorm 作为参数
    BasicModel(Embedding word_embeddings, std::vector<Block> layers, Norm final_layernorm)
        : word_embeddings(word_embeddings), layers(std::move(layers)), final_layernorm(final_layernorm) {}

    // 构造函数，接受 ModelContext 指针和 ModelConfig 作为参数
    BasicModel(ModelContext *ctx, const ModelConfig &config)
        : word_embeddings(ctx, config.vocab_size, config.hidden_size), layers(build_layers(ctx, config)),
          final_layernorm(ctx, config.hidden_size) {}

    // 前向传播函数，接受 ModelContext 指针、input_ids、n_past 和 n_ctx 作为参数
    ggml_tensor *forward(ModelContext *ctx, ggml_tensor *input_ids, int n_past, int n_ctx) const {
        ggml_context *gctx = ctx->ctx_b.get();
        // 生成位置 ID 张量
        ggml_tensor *position_ids = pos_ids_gen_(gctx, input_ids->ne[0], n_past, n_ctx);
        if (position_ids) {
            tensor_to_device(position_ids);
        }
        // 计算隐藏状态
        ggml_tensor *hidden_states = word_embeddings.forward(ctx, input_ids);
        // 遍历每个层进行前向传播
        for (const auto &layer : layers) {
            ggml_set_scratch(gctx, ctx->scratch);
            hidden_states = layer.forward(ctx, hidden_states, position_ids, n_past, n_ctx);
        }
        if (position_ids) {
            tensor_to_cpu(position_ids);
        }
        ggml_scratch empty_scratch = {0, 0, nullptr};
        ggml_set_scratch(gctx, empty_scratch);
        // 最终层归一化
        hidden_states = final_layernorm.forward(ctx, hidden_states);
        // 返回隐藏状态
        return hidden_states;
    }

  private:
    // 构建神经网络层，返回一个包含所有层的向量
    std::vector<Block> build_layers(ModelContext *ctx, const ModelConfig &config) {
        // 初始化一个空的层向量，预留足够的空间
        std::vector<Block> layers;
        layers.reserve(config.num_hidden_layers);
        // 遍历每一层，创建并添加到层向量中
        for (int layer_id = 0; layer_id < config.num_hidden_layers; layer_id++) {
            // TODO: 减少最大长度？32k 对于 CPU 推断可能太大
            // 创建一个新的 Block 对象，并添加到层向量中
            layers.emplace_back(ctx, config.hidden_size, config.num_attention_heads, config.num_kv_heads,
                                config.intermediate_size, config.max_length, config.norm_eps);
        }
        // 返回构建好的层向量
        return layers;
    }

  public:
    // 词嵌入层
    Embedding word_embeddings;
    // 神经网络层向量
    std::vector<Block> layers;
    // 最终的层归一化
    Norm final_layernorm;

  private:
    // 位置 ID 生成器
    PositionIdsGenerator pos_ids_gen_;
};

// 基类 BaseStreamer，定义了接口函数
class BaseStreamer {
  public:
    virtual ~BaseStreamer() = default;
    virtual void put(const std::vector<int> &output_ids) = 0;
    virtual void end() = 0;
};

// 派生类 StreamerGroup，继承自 BaseStreamer
class StreamerGroup : public BaseStreamer {
  public:
    // 构造函数，接受一个包含 BaseStreamer 智能指针的 vector
    StreamerGroup(std::vector<std::shared_ptr<BaseStreamer>> streamers) : streamers_(std::move(streamers)) {}
    // 实现基类接口函数 put
    void put(const std::vector<int> &output_ids) override;
    // 实现基类接口函数 end
    void end() override;

  private:
    std::vector<std::shared_ptr<BaseStreamer>> streamers_;
};

// 派生类 TextStreamer，继承自 BaseStreamer
// 参考链接：https://github.com/huggingface/transformers/blob/main/src/transformers/generation/streamers.py
class TextStreamer : public BaseStreamer {
  public:
    // 构造函数，接受一个 ostream 引用和 BaseTokenizer 指针
    TextStreamer(std::ostream &os, BaseTokenizer *tokenizer)
        : os_(os), tokenizer_(tokenizer), is_prompt_(true), is_first_line_(true), print_len_(0) {}
    // 实现基类接口函数 put
    void put(const std::vector<int> &output_ids) override;
    // 实现基类接口函数 end
    void end() override;

  private:
    std::ostream &os_;
    BaseTokenizer *tokenizer_;
    bool is_prompt_;
    bool is_first_line_;
    std::vector<int> token_cache_;
    int print_len_;
};

// 派生类 PerfStreamer，继承自 BaseStreamer
class PerfStreamer : public BaseStreamer {
  public:
    // 构造函数，初始化计时变量
    PerfStreamer() : start_us_(0), prompt_us_(0), end_us_(0), num_prompt_tokens_(0), num_output_tokens_(0) {}

    // 实现基类接口函数 put
    void put(const std::vector<int> &output_ids) override;
    // 实现基类接口函数 end
    void end() override { end_us_ = ggml_time_us(); }

    // 重置计时变量
    void reset();
    // 将性能数据转换为字符串
    std::string to_string() const;

    // 获取提示 token 数量
    int64_t num_prompt_tokens() const { return num_prompt_tokens_; }
    // 获取提示总时间
    int64_t prompt_total_time_us() const { return prompt_us_ - start_us_; }
    // 获取每个提示 token 的平均时间
    int64_t prompt_token_time_us() const {
        return num_prompt_tokens() ? prompt_total_time_us() / num_prompt_tokens() : 0;
    }
    // 获取输出 token 数量
    int64_t num_output_tokens() const { return num_output_tokens_; }
    // 获取输出总时间
    int64_t output_total_time_us() const { return end_us_ - prompt_us_; }
    // 获取每个输出 token 的平均时间
    int64_t output_token_time_us() const {
        return num_output_tokens() ? output_total_time_us() / num_output_tokens() : 0;
    }

  private:
    int64_t start_us_;
    # 定义一个64位整型变量，用于存储提示的时间戳
    int64_t prompt_us_;
    # 定义一个64位整型变量，用于存储结束的时间戳
    int64_t end_us_;
    # 定义一个64位整型变量，用于存储提示的令牌数量
    int64_t num_prompt_tokens_;
    # 定义一个64位整型变量，用于存储输出的令牌数量
    int64_t num_output_tokens_;
};

// 定义 MappedFile 类
class MappedFile {
  public:
    // 构造函数，接受文件路径作为参数
    MappedFile(const std::string &path);
    // 析构函数
    ~MappedFile();

  public:
    // 文件数据指针
    char *data;
    // 文件大小
    size_t size;
};

// 定义 ModelLoader 类
class ModelLoader {
  public:
    // 构造函数，接受数据指针和大小作为参数
    ModelLoader(char *data, size_t size) : data(data), size(size), ptr(data) {}

    // 返回当前指针位置
    int64_t tell() const { return ptr - data; }

    // 移动指针到指定位置
    void seek(int64_t offset, int whence);

    // 读取基本数据类型
    template <typename T>
    T read_basic() {
        T obj = *(T *)ptr;
        ptr += sizeof(T);
        return obj;
    }

    // 读取字符串
    std::string read_string(size_t length);

    // 读取张量元数据
    void checked_read_tensor_meta(const std::string &name, int ndim, int64_t *ne, ggml_type dtype);

    // 读取张量数据
    void *read_tensor_data(size_t nbytes);

    // 读取张量
    void read_tensor(const std::string &name, ggml_tensor *tensor);

  public:
    // 数据指针
    char *data;
    // 数据大小
    size_t size;
    // 当前指针位置
    char *ptr;
};

// 定义 GenerationConfig 结构体
struct GenerationConfig {
    // 生成配置参数
    int max_length;
    int max_new_tokens;
    int max_context_length;
    bool do_sample;
    int top_k;
    float top_p;
    float temperature;
    float repetition_penalty;
    int num_threads;

    // 构造函数，设置默认值
    GenerationConfig(int max_length = 2048, int max_new_tokens = -1, int max_context_length = 512,
                     bool do_sample = true, int top_k = 0, float top_p = 0.7, float temperature = 0.95,
                     float repetition_penalty = 1.f, int num_threads = 0)
        : max_length(max_length), max_new_tokens(max_new_tokens), max_context_length(max_context_length),
          do_sample(do_sample), top_k(top_k), top_p(top_p), temperature(temperature),
          repetition_penalty(repetition_penalty), num_threads(num_threads) {}
};

// 获取物理核心数
int get_num_physical_cores();
// 获取默认线程数
int get_default_num_threads();

// 定义 TokenIdScore 结构体
struct TokenIdScore {
    // token id 和得分
    int id;
    float score;

    // 默认构造函数
    TokenIdScore() = default;
    // 构造函数，接受 id 和得分作为参数
    TokenIdScore(int id, float score) : id(id), score(score) {}

    // 重载小于运算符
    bool operator<(const TokenIdScore &other) const { return score < other.score; }
    // 重载大于运算符
    bool operator>(const TokenIdScore &other) const { return score > other.score; }
    # 重载输出流操作符，用于将 TokenIdScore 对象输出到流中
    friend std::ostream &operator<<(std::ostream &os, const TokenIdScore &self) {
        # 将 TokenIdScore 对象的 id 和 score 输出到流中
        return os << "TokenIdScore(id=" << self.id << ", score=" << self.score << ")";
    }
// 结束类定义
};

// 定义基础的 CausalLM 模型类
class BaseModelForCausalLM {
  public:
    // 构造函数，接受模型配置、内存大小、临时内存大小和权重数量作为参数
    BaseModelForCausalLM(ModelConfig config, size_t mem_size, size_t scratch_size, size_t num_weights);
    // 虚析构函数
    virtual ~BaseModelForCausalLM() = default;

    // 加载模型
    virtual void load(ModelLoader &loader) = 0;
    // 前向推理函数，接受模型上下文、输入张量、过去步数、上下文长度和解码标志作为参数
    virtual ggml_tensor *forward(ModelContext *ctx, ggml_tensor *input_ids, int n_past, int n_ctx,
                                 bool is_decoding) const = 0;

    // 前向图计算函数，接受输入张量、过去步数、上下文长度、线程数和解码标志作为参数
    ggml_tensor *forward_graph_compute(const std::vector<int> &input_ids, int n_past, int n_ctx, int n_threads,
                                       bool is_decoding);

    // 生成函数，接受输入张量、生成配置和流处理器作为参数
    std::vector<int> generate(const std::vector<int> &input_ids, const GenerationConfig &gen_config,
                              BaseStreamer *streamer = nullptr);

    // 生成下一个标记函数，接受输入张量、生成配置、过去步数和上下文长度作为参数
    int generate_next_token(const std::vector<int> &input_ids, const GenerationConfig &gen_config, int n_past,
                            int n_ctx);

    // logits 处理器，接受首尾指针、输入张量和惩罚值作为参数
    static void sampling_repetition_penalty(float *first, float *last, const std::vector<int> &input_ids,
                                            float penalty);
    // logits 调节器，接受首尾指针和温度值作为参数
    static void sampling_temperature(float *first, float *last, float temp);
    // logits 选择器，接受首尾指针和 top-k 值作为参数
    static void sampling_top_k(TokenIdScore *first, TokenIdScore *kth, TokenIdScore *last);
    // logits 选择器，接受首尾指针和 top-p 值作为参数
    static TokenIdScore *sampling_top_p(TokenIdScore *first, TokenIdScore *last, float top_p);

    // softmax 计算函数，接受首尾指针作为参数
    static void sampling_softmax_inplace(TokenIdScore *first, TokenIdScore *last);

  protected:
    // 模型上下文
    ModelContext ctx_;

  public:
    // 模型配置
    ModelConfig config;
};

// 状态字典类型定义
using StateDict = std::vector<std::pair<std::string, ggml_tensor *>>;

// 基础 CausalLM 模型类模板，继承自 BaseModelForCausalLM
template <typename Model>
class BasicModelForCausalLM : public BaseModelForCausalLM {
  protected:
    // 基于给定的配置、内存大小、临时内存大小和权重数量构造 BasicModelForCausalLM 对象
    BasicModelForCausalLM(const ModelConfig &config, size_t mem_size, size_t scratch_size, size_t num_weights)
        : BaseModelForCausalLM(config, mem_size, scratch_size, num_weights), transformer(&ctx_, config),
          lm_head(&ctx_, config.hidden_size, config.vocab_size, false) {
        // 检查权重内存是否被正确使用
        CHATGLM_CHECK(ggml_used_mem(ctx_.ctx_w.get()) == ggml_get_mem_size(ctx_.ctx_w.get()))
            << "corrupted model weights";
        // 检查键值缓存内存是否被正确使用
        CHATGLM_CHECK(ggml_used_mem(ctx_.ctx_kv.get()) + 1 * MB == ggml_get_mem_size(ctx_.ctx_kv.get()))
            << "corrupted kv cache";
    }
    // 析构函数，将模型权重转移到 CPU
    ~BasicModelForCausalLM() { to_cpu(); }

  public:
    // 实现基类的前向传播方法
    ggml_tensor *forward(ModelContext *ctx, ggml_tensor *input_ids, int n_past, int n_ctx,
                         bool is_decoding) const override {
        // 使用 Transformer 进行前向传播
        ggml_tensor *transformer_outputs = transformer.forward(ctx, input_ids, n_past, n_ctx);
        // 注意：仅在解码时计算下一个标记的对数概率
        if (is_decoding && input_ids->ne[0] > 1) {
            // 如果是解码且输入标记数量大于1，则重新分配缓冲区
            transformer_outputs = tensor_assign_buffers(
                ggml_view_1d(ctx->ctx_b.get(), transformer_outputs, config.hidden_size,
                             (input_ids->ne[0] - 1) * config.hidden_size * ggml_element_size(transformer_outputs)));
        }
        // 使用 LM 头部进行前向传播，得到 LM 模型的输出
        ggml_tensor *lm_logits = lm_head.forward(ctx, transformer_outputs);
        return lm_logits;
    }

  protected:
    // 将模型权重转移到 CPU
    void to_cpu() {
        // 将状态字典中的张量转移到 CPU
        for (auto &item : state_dict_) {
            tensor_to_cpu(item.second);
        }
        // 将 Transformer 层中的注意力缓存转移到 CPU
        for (auto &layer : transformer.layers) {
            tensor_to_cpu(layer.attention.k_cache);
            tensor_to_cpu(layer.attention.v_cache);
        }
    }
    // 将模型参数移动到设备上
    void to_device() {
        // 遍历模型参数字典
        for (auto &item : state_dict_) {
            ggml_tensor *tensor = item.second;
            // 不应该将嵌入层放置到设备上
            if (tensor != transformer.word_embeddings.weight) {
                // 将张量移动到设备上
                tensor_to_device(tensor);
            }
        }

        // 遍历变压器模型的每一层
        for (auto &layer : transformer.layers) {
            // 将注意力机制中的 k_cache 和 v_cache 移动到设备上
            tensor_to_device(layer.attention.k_cache);
            tensor_to_device(layer.attention.v_cache);
        }
    }

  public:
    // 变压器模型
    Model transformer;
    // 线性层 lm_head

  protected:
    // 模型参数字典
    StateDict state_dict_;
// 结构体 GLMContextMasker 重载了函数调用操作符，用于对模型上下文进行处理
struct GLMContextMasker {
    ggml_tensor *operator()(ModelContext *ctx, ggml_tensor *attn_scores, int n_past) const;
};

// 使用 GLMBlock 类定义了一个基本块，包含 LayerNorm、GLMAttention 和 GLMMLP 三个组件
class GLMBlock : public BasicBlock<LayerNorm, GLMAttention, GLMMLP> {
  public:
    // 默认构造函数
    GLMBlock() = default;
    // 带参数的构造函数，初始化 GLMBlock 的各个组件
    GLMBlock(ModelContext *ctx, int hidden_size, int num_attention_heads, int num_kv_heads, int intermediate_size,
             int max_length, float norm_eps)
        : BasicBlock(LayerNorm(ctx, hidden_size, false, norm_eps),
                     GLMAttention(ctx, hidden_size, num_attention_heads, num_attention_heads, max_length),
                     LayerNorm(ctx, hidden_size, false, norm_eps), GLMMLP(ctx, hidden_size, intermediate_size)),
          alpha_value(std::sqrt(2.f * 28)) {}

    // 前向传播函数，处理隐藏状态和位置 ID，返回处理后的张量
    ggml_tensor *forward(ModelContext *ctx, ggml_tensor *hidden_states, ggml_tensor *position_ids, int n_past,
                         int n_ctx) const;

  public:
    // alpha_value 参数，用于计算
    float alpha_value;
};

// 使用 BasicAttention 模板定义 GLMAttention 类
using GLMAttention = BasicAttention<true, true, true, GLMRoper, false, GLMContextMasker>;

// 使用 BasicMLP 模板定义 GLMMLP 类
using GLMMLP = BasicMLP<ActivationType::GELU>;

// ChatGLMTokenizer 类，继承自 BaseTokenizer 类
class ChatGLMTokenizer : public BaseTokenizer {
  public:
    // 构造函数，接受序列化的模型协议字符串
    ChatGLMTokenizer(std::string_view serialized_model_proto);

    // 编码函数，将文本编码为整数序列
    std::vector<int> encode(const std::string &text, int max_length) const override;

    // 解码函数，将整数序列解码为文本
    std::string decode(const std::vector<int> &ids) const override;

    // 编码消息函数，将消息列表编码为整数序列
    std::vector<int> encode_messages(const std::vector<ChatMessage> &messages, int max_length) const override;

    // 构建提示函数，根据消息列表构建提示字符串
    static std::string build_prompt(const std::vector<ChatMessage> &messages);

  private:
    // 预处理函数，对文本进行预处理
    static std::string preprocess(const std::string &text);

    // 后处理函数，对文本进行后处理
    static std::string postprocess(const std::string &text);

  public:
    // SentencePieceProcessor 对象
    sentencepiece::SentencePieceProcessor sp;
    // 起始标记 ID
    int bos_token_id;
    // 结束标记 ID
    int eos_token_id;
    // 掩码标记 ID
    int mask_token_id;
    // 全局掩码标记 ID
    int gmask_token_id;
    // 填充标记 ID
    int pad_token_id;
};
// 使用 ChatGLMModel 模板定义 ChatGLMForCausalLM 类
using ChatGLMModel = BasicModel<GLMBlock, LayerNorm, GLMPositionIdsGenerator>;

class ChatGLMForCausalLM : public BasicModelForCausalLM<ChatGLMModel> {
  public:
    // ChatGLMForCausalLM 构造函数，接受 ModelConfig 参数
    ChatGLMForCausalLM(const ModelConfig &config);

    // 重写 load 方法，加载模型
    void load(ModelLoader &loader) override;

    // 静态方法，计算权重数量，根据隐藏层数量计算
    static int num_weights(int num_hidden_layers) { return 4 + num_hidden_layers * 12; }

  private:
    // 返回状态字典
    StateDict state_dict() const;

  public:
    // 定义内存大小常量
    static constexpr size_t MEM_SIZE = 1280 * MB;     // 2k context
    // 定义临时内存大小常量
    static constexpr size_t SCRATCH_SIZE = 1024 * MB; // 2k context
};

// ===== ChatGLM2-6B =====

// 使用 ChatGLM2Tokenizer 类继承 BaseTokenizer 类
class ChatGLM2Tokenizer : public BaseTokenizer {
  public:
    // ChatGLM2Tokenizer 构造函数，接受 serialized_model_proto 参数
    ChatGLM2Tokenizer(std::string_view serialized_model_proto);

    // 重写 encode 方法，将文本编码为整数序列
    std::vector<int> encode(const std::string &text, int max_length) const override;

    // 重写 decode 方法，将整数序列解码为文本
    std::string decode(const std::vector<int> &ids) const override;

    // 编码消息序列的方法
    std::vector<int> encode_messages(const std::vector<ChatMessage> &messages, int max_length) const override;

    // 构建提示信息的静态方法
    static std::string build_prompt(const std::vector<ChatMessage> &messages);

  private:
    // 判断是否为特殊 id 的方法
    bool is_special_id(int id) const;

  public:
    // 定义 SentencePieceProcessor 对象
    sentencepiece::SentencePieceProcessor sp;
    // 定义 mask_token_id
    int mask_token_id;
    // 定义 gmask_token_id
    int gmask_token_id;
    // 定义 smask_token_id
    int smask_token_id;
    // 定义 sop_token_id
    int sop_token_id;
    // 定义 eop_token_id
    int eop_token_id;
};

// 使用 GLM2Attention 模板定义 GLM2Attention 类
using GLM2Attention = BasicAttention<true, false, false, BasicRoper<ROPE_TYPE_DEFAULT, 2>, false, CausalContextMasker>;

// 使用 GLM2MLP 模板定义 GLM2MLP 类
using GLM2MLP = BasicGLU<ActivationType::SILU, false>;

// 使用 GLM2Block 模板定义 GLM2Block 类
using GLM2Block = BasicBlock<RMSNorm, GLM2Attention, GLM2MLP>;

// 使用 ChatGLM2Model 模板定义 ChatGLM2Model 类
using ChatGLM2Model = BasicModel<GLM2Block, RMSNorm, BasicPositionIdsGenerator>;

// 使用 ChatGLM2Model 类继承 BasicModelForCausalLM 类
class ChatGLM2ForCausalLM : public BasicModelForCausalLM<ChatGLM2Model> {
  public:
    // ChatGLM2ForCausalLM 构造函数，接受 ModelConfig 参数
    ChatGLM2ForCausalLM(const ModelConfig &config);

    // 重写 load 方法，加载模型
    void load(ModelLoader &loader) override;

    // 静态方法，计算权重数量，根据隐藏层数量计算
    static int num_weights(int num_hidden_layers) { return 3 + num_hidden_layers * 8; }

  private:
    // 返回状态字典
    StateDict state_dict() const;

  public:
    // 定义内存大小常量
    static constexpr size_t MEM_SIZE = 1280 * MB;     // 2k context
    # 定义一个静态常量，表示缓冲区的大小为1280MB，用于存储上下文信息
    static constexpr size_t SCRATCH_SIZE = 1280 * MB; // 2k context
// 结束了 ChatGLM3Tokenizer 类的定义

// ChatGLM3-6B 模型的 Tokenizer 类，继承自 BaseTokenizer 类
class ChatGLM3Tokenizer : public BaseTokenizer {
  public:
    // 构造函数，接受序列化的模型协议作为参数
    ChatGLM3Tokenizer(std::string_view serialized_model_proto);

    // 对文本进行编码，返回编码后的整数向量
    std::vector<int> encode(const std::string &text, int max_length) const override;

    // 对整数向量进行解码，返回解码后的文本
    std::string decode(const std::vector<int> &ids) const override;

    // 对消息列表进行编码，返回编码后的整数向量
    std::vector<int> encode_messages(const std::vector<ChatMessage> &messages, int max_length) const override;

    // 对整数向量进行解码，返回解码后的消息
    ChatMessage decode_message(const std::vector<int> &ids) const override;

  private:
    // 对单个消息进行编码，返回编码后的整数向量
    std::vector<int> encode_single_message(const std::string &role, const std::string &content) const;

    // 带有特殊标记的解码方法，返回解码后的文本
    std::string decode_with_special_tokens(const std::vector<int> &ids) const;

    // 移除特殊标记的方法，返回移除特殊标记后的文本
    static std::string remove_special_tokens(const std::string &text);

    // 获取命令的方法，返回命令对应的整数
    int get_command(const std::string &token) const;

    // 判断是否为特殊标记的方法，返回是否为特殊标记的布尔值
    bool is_special_id(int id) const;

    // 截断整数向量的方法，修改传入的整数向量使其不超过最大长度
    static void truncate(std::vector<int> &ids, int max_length);

  public:
    // SentencePiece 处理器对象
    sentencepiece::SentencePieceProcessor sp;
    // 各种特殊标记的整数标识
    int mask_token_id;
    int gmask_token_id;
    int smask_token_id;
    int sop_token_id;
    int eop_token_id;
    int system_token_id;
    int user_token_id;
    int assistant_token_id;
    int observation_token_id;
    // 特殊标记到整数的映射
    std::unordered_map<std::string, int> special_tokens;
    // 整数到特殊标记的映射
    std::unordered_map<int, std::string> index_special_tokens;
};

// ChatGLM3Model 类与 ChatGLM2Model 类相同
using ChatGLM3Model = ChatGLM2Model;

// ChatGLM3ForCausalLM 类与 ChatGLM2ForCausalLM 类相同

// 结束了 BaichuanTokenizer 类的定义

// Baichuan 模型的 Tokenizer 类，继承自 BaseTokenizer 类
class BaichuanTokenizer : public BaseTokenizer {
  public:
    // 构造函数，接受序列化的模型协议作为参数
    BaichuanTokenizer(std::string_view serialized_model_proto);

    // 对文本进行编码，返回编码后的整数向量
    std::vector<int> encode(const std::string &text, int max_length) const override;

    // 对整数向量进行解码，返回解码后的文本
    std::string decode(const std::vector<int> &ids) const override;

    // 对消息列表进行编码，返回编码后的整数向量
    std::vector<int> encode_messages(const std::vector<ChatMessage> &messages, int max_length) const override;

  private:
    // 判断是否为特殊标记的方法，返回是否为特殊标记的布尔值
    bool is_special_id(int id) const;

    // 截断整数向量的方法，修改传入的整数向量使其不超过最大长度
    static void truncate(std::vector<int> &ids, int max_length);

  public:
    // 定义用户令牌ID为195
    static constexpr int USER_TOKEN_ID = 195;
    // 定义助手令牌ID为196
    static constexpr int ASSISTANT_TOKEN_ID = 196;

    // 创建SentencePieceProcessor对象
    sentencepiece::SentencePieceProcessor sp;
    // 定义开始标记令牌ID
    int bos_token_id;
    // 定义结束标记令牌ID
    int eos_token_id;
    // 定义填充标记令牌ID
    int pad_token_id;
// 定义 Baichuan-7B 模型的注意力机制类型
using Baichuan7BAttention = BasicAttention<false, false, false, BasicRoper<ROPE_TYPE_NEOX, 1>, false, CausalContextMasker>;

// 定义 Baichuan-7B 模型的多层感知机类型
using Baichuan7BMLP = BasicGLU<ActivationType::SILU, false>;

// 定义 Baichuan-7B 模型的基本块类型
using Baichuan7BBlock = BasicBlock<RMSNorm, Baichuan7BAttention, Baichuan7BMLP>;

// 定义 Baichuan-7B 模型类型
using Baichuan7BModel = BasicModel<Baichuan7BBlock, RMSNorm, BasicPositionIdsGenerator>;

// 定义 Baichuan7BForCausalLM 类，继承自 BasicModelForCausalLM<Baichuan7BModel>
class Baichuan7BForCausalLM : public BasicModelForCausalLM<Baichuan7BModel> {
  public:
    Baichuan7BForCausalLM(const ModelConfig &config);

    void load(ModelLoader &loader) override;

    // 计算权重数量的静态方法
    static int num_weights(int num_hidden_layers) { return 3 + num_hidden_layers * 7; }

  private:
    // 返回状态字典的私有方法
    StateDict state_dict() const;

  public:
    // 定义内存大小和缓冲区大小的常量
    static constexpr size_t MEM_SIZE = 1280 * MB;
    static constexpr size_t SCRATCH_SIZE = 1280 * MB;
};

// 定义 Baichuan-13B 模型的注意力机制类型
using Baichuan13BAttention = BasicAttention<false, false, false, NoopRoper, true, CausalContextMasker>;

// 定义 Baichuan-13B 模型的多层感知机类型
using Baichuan13BMLP = BasicGLU<ActivationType::SILU, false>;

// 定义 Baichuan-13B 模型的基本块类型
using Baichuan13BBlock = BasicBlock<RMSNorm, Baichuan13BAttention, Baichuan13BMLP>;

// 定义 Baichuan-13B 模型类型
using Baichuan13BModel = BasicModel<Baichuan13BBlock, RMSNorm, NoopPositionIdsGenerator>;

// 定义 Baichuan13BForCausalLM 类，继承自 BasicModelForCausalLM<Baichuan13BModel>
class Baichuan13BForCausalLM : public BasicModelForCausalLM<Baichuan13BModel> {
  public:
    Baichuan13BForCausalLM(const ModelConfig &config);

    void load(ModelLoader &loader) override;

    // 计算权重数量的静态方法
    static int num_weights(int num_hidden_layers) { return 3 + num_hidden_layers * 7; }

  private:
    // 返回状态字典的私有方法
    StateDict state_dict() const;

  public:
    // 定义内存大小和缓冲区大小的常量
    static constexpr size_t MEM_SIZE = 1280 * MB;
    static constexpr size_t SCRATCH_SIZE = 1280 * MB;
};

// 定义 InternLMTokenizer 类，继承自 BaseTokenizer
class InternLMTokenizer : public BaseTokenizer {
  public:
    // 构造函数，接受序列化的模型协议字符串
    InternLMTokenizer(std::string_view serialized_model_proto);

    // 编码文本的方法，返回编码后的整数向量
    std::vector<int> encode(const std::string &text, int max_length) const override;

    // 解码整数向量的方法，返回解码后的文本
    std::string decode(const std::vector<int> &ids) const override;
    // 重写函数，将聊天消息编码为整数向量，限制最大长度
    std::vector<int> encode_messages(const std::vector<ChatMessage> &messages, int max_length) const override;

    // 静态函数，构建提示信息字符串，基于给定的聊天消息
    static std::string build_prompt(const std::vector<ChatMessage> &messages);

  private:
    // 检查给定的 ID 是否为特殊标记 ID，返回布尔值
    bool is_special_id(int id) const { return id == unk_token_id || id == bos_token_id || id == eos_token_id; }

  public:
    // SentencePiece 处理器对象
    sentencepiece::SentencePieceProcessor sp;
    // 静态常量，未知标记 ID
    static constexpr int unk_token_id = 0;
    // 静态常量，起始标记 ID
    static constexpr int bos_token_id = 1;
    // 静态常量，结束标记 ID
    static constexpr int eos_token_id = 2;
// 结束 C++ 类定义
};

// 使用 InternLM7BAttention 类型定义 InternLM7BAttention 类
using InternLM7BAttention =
    BasicAttention<true, true, false, BasicRoper<ROPE_TYPE_NEOX, 1>, false, CausalContextMasker>;

// 使用 InternLM7BMLP 类型定义 InternLM7BMLP 类
using InternLM7BMLP = BasicGLU<ActivationType::SILU, false>;

// 使用 InternLM7BBlock 类型定义 InternLM7BBlock 类
using InternLM7BBlock = BasicBlock<RMSNorm, InternLM7BAttention, InternLM7BMLP>;

// 使用 InternLM7BModel 类型定义 InternLM7BModel 类
using InternLM7BModel = BasicModel<InternLM7BBlock, RMSNorm, BasicPositionIdsGenerator>;

// 使用 InternLM20BAttention 类型定义 InternLM20BAttention 类
using InternLM20BAttention =
    BasicAttention<false, false, false, BasicRoper<ROPE_TYPE_NEOX, 1>, false, CausalContextMasker>;

// 使用 InternLM20BMLP 类型定义 InternLM20BMLP 类
using InternLM20BMLP = BasicGLU<ActivationType::SILU, false>;

// 使用 InternLM20BBlock 类型定义 InternLM20BBlock 类
using InternLM20BBlock = BasicBlock<RMSNorm, InternLM20BAttention, InternLM20BMLP>;

// 使用 InternLM20BModel 类型定义 InternLM20BModel 类
using InternLM20BModel = BasicModel<InternLM20BBlock, RMSNorm, BasicPositionIdsGenerator>;

// 模板类定义，继承自 BasicModelForCausalLM 类
template <typename InternLMModel>
class InternLMForCausalLM : public BasicModelForCausalLM<InternLMModel> {
  public:
    // 构造函数
    InternLMForCausalLM(const ModelConfig &config);

    // 重写 load 方法
    void load(ModelLoader &loader) override;

    // 静态方法，返回权重数量
    static int num_weights(int num_hidden_layers) {
        return 3 + num_hidden_layers * (std::is_same_v<InternLMModel, InternLM7BModel> ? 9 : 7);
    }

  private:
    // 返回状态字典
    StateDict state_dict() const;

  public:
    // 静态常量，内存大小
    static constexpr size_t MEM_SIZE = 1280 * MB;
    // 静态常量，缓冲区大小
    static constexpr size_t SCRATCH_SIZE = 1280 * MB;
};

// 使用 InternLM7BModel 类型定义 InternLM7BForCausalLM 类
using InternLM7BForCausalLM = InternLMForCausalLM<InternLM7BModel>;

// 使用 InternLM20BModel 类型定义 InternLM20BForCausalLM 类
using InternLM20BForCausalLM = InternLMForCausalLM<InternLM20BModel>;

// ===== pipeline =====

// Pipeline 类定义
class Pipeline {
  public:
    // 构造函数
    Pipeline(const std::string &path);

    // 生成方法，返回整数向量
    std::vector<int> generate(const std::vector<int> &input_ids, const GenerationConfig &gen_config,
                              BaseStreamer *streamer = nullptr) const;

    // 生成方法，返回字符串
    std::string generate(const std::string &prompt, const GenerationConfig &gen_config,
                         BaseStreamer *streamer = nullptr) const;
    // 定义一个函数chat，接受消息向量、生成配置和流对象作为参数，返回一个ChatMessage对象
    ChatMessage chat(const std::vector<ChatMessage> &messages, const GenerationConfig &gen_config,
                     BaseStreamer *streamer = nullptr) const;

  public:
    // 声明一个独占指针tokenizer，指向BaseTokenizer类型的对象
    std::unique_ptr<BaseTokenizer> tokenizer;
    // 声明一个独占指针model，指向BaseModelForCausalLM类型的对象
    std::unique_ptr<BaseModelForCausalLM> model;
    // 声明一个独占指针mapped_file，指向MappedFile类型的对象
    std::unique_ptr<MappedFile> mapped_file;
};

// 结束 chatglm 命名空间
} // namespace chatglm
```