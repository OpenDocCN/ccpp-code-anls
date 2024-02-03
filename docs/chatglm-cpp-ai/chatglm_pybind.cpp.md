# `chatglm.cpp\chatglm_pybind.cpp`

```cpp
#include "chatglm.h"
#include <pybind11/pybind11.h>
#include <pybind11/stl.h>

namespace chatglm {

namespace py = pybind11;
using namespace pybind11::literals;

// 定义 PyBaseTokenizer 类，继承自 BaseTokenizer
class PyBaseTokenizer : public BaseTokenizer {
  public:
    using BaseTokenizer::BaseTokenizer;

    // 重写 encode 方法，返回编码后的文本
    std::vector<int> encode(const std::string &text, int max_length) const override {
        PYBIND11_OVERRIDE_PURE(std::vector<int>, BaseTokenizer, encode, text, max_length);
    }
    // 重写 decode 方法，返回解码后的文本
    std::string decode(const std::vector<int> &ids) const override {
        PYBIND11_OVERLOAD_PURE(std::string, BaseTokenizer, decode, ids);
    }
    // 重写 encode_messages 方法，返回编码后的消息
    std::vector<int> encode_messages(const std::vector<ChatMessage> &history, int max_length) const override {
        PYBIND11_OVERLOAD_PURE(std::vector<int>, BaseTokenizer, encode_messages, history, max_length);
    }
};

// 定义 PyBaseModelForCausalLM 类，继承自 BaseModelForCausalLM
class PyBaseModelForCausalLM : public BaseModelForCausalLM {
  public:
    using BaseModelForCausalLM::BaseModelForCausalLM;

    // 重写 load 方法，加载模型
    void load(ModelLoader &loader) override { PYBIND11_OVERLOAD_PURE(void, PyBaseModelForCausalLM, load, loader); }

    // 重写 forward 方法，进行前向传播
    ggml_tensor *forward(ModelContext *ctx, ggml_tensor *input_ids, int n_past, int n_ctx,
                         bool is_decoding) const override {
        PYBIND11_OVERLOAD_PURE(ggml_tensor *, PyBaseModelForCausalLM, forward, ctx, input_ids, n_past, n_ctx,
                               is_decoding)
    }
};

// 将对象转换为字符串
template <typename T>
static inline std::string to_string(const T &obj) {
    std::ostringstream oss;
    oss << obj;
    return oss.str();
}

// 定义 Python 模块 _C
PYBIND11_MODULE(_C, m) {
    // 设置模块文档字符串
    m.doc() = "ChatGLM.cpp python binding";

    // 定义 ModelType 枚举类型
    py::enum_<ModelType>(m, "ModelType")
        .value("CHATGLM", ModelType::CHATGLM)
        .value("CHATGLM2", ModelType::CHATGLM2)
        .value("CHATGLM3", ModelType::CHATGLM3)
        .value("BAICHUAN7B", ModelType::BAICHUAN7B)
        .value("BAICHUAN13B", ModelType::BAICHUAN13B)
        .value("INTERNLM", ModelType::INTERNLM);
}
    # 在 Python 中定义 ModelConfig 类，并将其绑定到模块 m 中，命名为 "ModelConfig"
    py::class_<ModelConfig>(m, "ModelConfig")
        # 定义只读属性 "model_type"，指向 ModelConfig 类中的 model_type 成员变量
        .def_readonly("model_type", &ModelConfig::model_type)
        # 定义只读属性 "vocab_size"，指向 ModelConfig 类中的 vocab_size 成员变量
        .def_readonly("vocab_size", &ModelConfig::vocab_size)
        # 定义只读属性 "hidden_size"，指向 ModelConfig 类中的 hidden_size 成员变量
        .def_readonly("hidden_size", &ModelConfig::hidden_size)
        # 定义只读属性 "num_attention_heads"，指向 ModelConfig 类中的 num_attention_heads 成员变量
        .def_readonly("num_attention_heads", &ModelConfig::num_attention_heads)
        # 定义只读属性 "num_kv_heads"，指向 ModelConfig 类中的 num_kv_heads 成员变量
        .def_readonly("num_kv_heads", &ModelConfig::num_kv_heads)
        # 定义只读属性 "num_hidden_layers"，指向 ModelConfig 类中的 num_hidden_layers 成员变量
        .def_readonly("num_hidden_layers", &ModelConfig::num_hidden_layers)
        # 定义只读属性 "intermediate_size"，指向 ModelConfig 类中的 intermediate_size 成员变量
        .def_readonly("intermediate_size", &ModelConfig::intermediate_size)
        # 定义只读属性 "norm_eps"，指向 ModelConfig 类中的 norm_eps 成员变量
        .def_readonly("norm_eps", &ModelConfig::norm_eps)
        # 定义只读属性 "max_length"，指向 ModelConfig 类中的 max_length 成员变量
        .def_readonly("max_length", &ModelConfig::max_length)
        # 定义只读属性 "bos_token_id"，指向 ModelConfig 类中的 bos_token_id 成员变量
        .def_readonly("bos_token_id", &ModelConfig::bos_token_id)
        # 定义只读属性 "eos_token_id"，指向 ModelConfig 类中的 eos_token_id 成员变量
        .def_readonly("eos_token_id", &ModelConfig::eos_token_id)
        # 定义只读属性 "pad_token_id"，指向 ModelConfig 类中的 pad_token_id 成员变量
        .def_readonly("pad_token_id", &ModelConfig::pad_token_id)
        # 定义只读属性 "sep_token_id"，指向 ModelConfig 类中的 sep_token_id 成员变量
        .def_readonly("sep_token_id", &ModelConfig::sep_token_id)
        # 定义只读属性 "extra_eos_token_ids"，指向 ModelConfig 类中的 extra_eos_token_ids 成员变量
        .def_readonly("extra_eos_token_ids", &ModelConfig::extra_eos_token_ids)
        # 定义只读属性 "model_type_name"，指向 ModelConfig 类中的 model_type_name 成员变量
        .def_property_readonly("model_type_name", &ModelConfig::model_type_name);
    # 定义 GenerationConfig 类，包含一系列参数，并设置默认值
    py::class_<GenerationConfig>(m, "GenerationConfig")
        .def(py::init<int, int, int, bool, int, float, float, float, int>(), "max_length"_a = 2048,
             "max_new_tokens"_a = -1, "max_context_length"_a = 512, "do_sample"_a = true, "top_k"_a = 0,
             "top_p"_a = 0.7, "temperature"_a = 0.95, "repetition_penalty"_a = 1.0, "num_threads"_a = 0)
        .def_readwrite("max_length", &GenerationConfig::max_length)  # 读写 max_length 属性
        .def_readwrite("max_new_tokens", &GenerationConfig::max_new_tokens)  # 读写 max_new_tokens 属性
        .def_readwrite("max_context_length", &GenerationConfig::max_context_length)  # 读写 max_context_length 属性
        .def_readwrite("do_sample", &GenerationConfig::do_sample)  # 读写 do_sample 属性
        .def_readwrite("top_k", &GenerationConfig::top_k)  # 读写 top_k 属性
        .def_readwrite("top_p", &GenerationConfig::top_p)  # 读写 top_p 属性
        .def_readwrite("temperature", &GenerationConfig::temperature)  # 读写 temperature 属性
        .def_readwrite("repetition_penalty", &GenerationConfig::repetition_penalty)  # 读写 repetition_penalty 属性
        .def_readwrite("num_threads", &GenerationConfig::num_threads);  # 读写 num_threads 属性

    # 定义 FunctionMessage 类，包含 __repr__ 和 __str__ 方法，以及 name 和 arguments 属性
    py::class_<FunctionMessage>(m, "FunctionMessage")
        .def("__repr__", &to_string<FunctionMessage>)  # 定义 __repr__ 方法
        .def("__str__", &to_string<FunctionMessage>)  # 定义 __str__ 方法
        .def_readwrite("name", &FunctionMessage::name)  # 读写 name 属性
        .def_readwrite("arguments", &FunctionMessage::arguments);  # 读写 arguments 属性

    # 定义 CodeMessage 类，包含 __repr__ 和 __str__ 方法，以及 input 属性
    py::class_<CodeMessage>(m, "CodeMessage")
        .def("__repr__", &to_string<CodeMessage>)  # 定义 __repr__ 方法
        .def("__str__", &to_string<CodeMessage>)  # 定义 __str__ 方法
        .def_readwrite("input", &CodeMessage::input);  # 读写 input 属性

    # 定义 ToolCallMessage 类，包含 __repr__ 和 __str__ 方法，以及 type、function 和 code 属性
    py::class_<ToolCallMessage>(m, "ToolCallMessage")
        .def("__repr__", &to_string<ToolCallMessage>)  # 定义 __repr__ 方法
        .def("__str__", &to_string<ToolCallMessage>)  # 定义 __str__ 方法
        .def_readwrite("type", &ToolCallMessage::type)  # 读写 type 属性
        .def_readwrite("function", &ToolCallMessage::function)  # 读写 function 属性
        .def_readwrite("code", &ToolCallMessage::code);  # 读写 code 属性
    // 定义 ChatMessage 类，包括构造函数、__repr__、__str__方法以及一些静态常量和成员变量
    py::class_<ChatMessage>(m, "ChatMessage")
        .def(py::init<std::string, std::string, std::vector<ToolCallMessage>>(), "role"_a, "content"_a,
             "tool_calls"_a = std::vector<ToolCallMessage>{})
        .def("__repr__", &to_string<ChatMessage>)
        .def("__str__", &to_string<ChatMessage>)
        .def_readonly_static("ROLE_SYSTEM", &ChatMessage::ROLE_SYSTEM)
        .def_readonly_static("ROLE_USER", &ChatMessage::ROLE_USER)
        .def_readonly_static("ROLE_ASSISTANT", &ChatMessage::ROLE_ASSISTANT)
        .def_readonly_static("ROLE_OBSERVATION", &ChatMessage::ROLE_OBSERVATION)
        .def_readwrite("role", &ChatMessage::role)
        .def_readwrite("content", &ChatMessage::content)
        .def_readwrite("tool_calls", &ChatMessage::tool_calls);

    // 定义 BaseTokenizer 类，包括 encode、decode、encode_messages、decode_message 方法
    py::class_<BaseTokenizer, PyBaseTokenizer>(m, "BaseTokenizer")
        .def("encode", &BaseTokenizer::encode, "text"_a, "max_length"_a)
        .def("decode", &BaseTokenizer::decode, "ids"_a)
        .def("encode_messages", &BaseTokenizer::encode_messages, "messages"_a, "max_length"_a)
        .def("decode_message", &BaseTokenizer::decode_message, "ids"_a);

    // 定义 BaseModelForCausalLM 类，包括 generate_next_token 方法和 config 成员变量
    py::class_<BaseModelForCausalLM, PyBaseModelForCausalLM>(m, "BaseModelForCausalLM")
        .def("generate_next_token", &BaseModelForCausalLM::generate_next_token, "input_ids"_a, "gen_config"_a,
             "n_past"_a, "n_ctx"_a)
        .def_readonly("config", &BaseModelForCausalLM::config);

    // 定义 ChatGLMTokenizer 类，继承自 BaseTokenizer
    py::class_<ChatGLMTokenizer, BaseTokenizer>(m, "ChatGLMTokenizer");

    // 定义 ChatGLMForCausalLM 类，继承自 BaseModelForCausalLM
    py::class_<ChatGLMForCausalLM, BaseModelForCausalLM>(m, "ChatGLMForCausalLM");

    // 定义 ChatGLM2Tokenizer 类，继承自 BaseTokenizer
    py::class_<ChatGLM2Tokenizer, BaseTokenizer>(m, "ChatGLM2Tokenizer");

    // 定义 ChatGLM2ForCausalLM 类，继承自 BaseModelForCausalLM
    py::class_<ChatGLM2ForCausalLM, BaseModelForCausalLM>(m, "ChatGLM2ForCausalLM");

    // 定义 ChatGLM3Tokenizer 类，继承自 BaseTokenizer
    py::class_<ChatGLM3Tokenizer, BaseTokenizer>(m, "ChatGLM3Tokenizer");

    // 定义 Baichuan7B/13B 部分
    // 在 Python 模块中定义 BaichuanTokenizer 类，继承自 BaseTokenizer 类
    py::class_<BaichuanTokenizer, BaseTokenizer>(m, "BaichuanTokenizer");

    // 在 Python 模块中定义 Baichuan7BForCausalLM 类，继承自 BaseModelForCausalLM 类
    py::class_<Baichuan7BForCausalLM, BaseModelForCausalLM>(m, "Baichuan7BForCausalLM");

    // 在 Python 模块中定义 Baichuan13BForCausalLM 类，继承自 BaseModelForCausalLM 类
    py::class_<Baichuan13BForCausalLM, BaseModelForCausalLM>(m, "Baichuan13BForCausalLM");

    // ===== InternLM =====

    // 在 Python 模块中定义 InternLMTokenizer 类，继承自 BaseTokenizer 类
    py::class_<InternLMTokenizer, BaseTokenizer>(m, "InternLMTokenizer");

    // 在 Python 模块中定义 InternLM7BForCausalLM 类，继承自 BaseModelForCausalLM 类
    py::class_<InternLM7BForCausalLM, BaseModelForCausalLM>(m, "InternLM7BForCausalLM");

    // 在 Python 模块中定义 InternLM20BForCausalLM 类，继承自 BaseModelForCausalLM 类
    py::class_<InternLM20BForCausalLM, BaseModelForCausalLM>(m, "InternLM20BForCausalLM");

    // ===== Pipeline ====

    // 在 Python 模块中定义 Pipeline 类
    py::class_<Pipeline>(m, "Pipeline")
        // 定义 Pipeline 类的构造函数，接受一个字符串参数
        .def(py::init<const std::string &>(), "path"_a)
        // 定义 Pipeline 类的 model 属性的只读访问器
        .def_property_readonly("model", [](const Pipeline &self) { return self.model.get(); })
        // 定义 Pipeline 类的 tokenizer 属性的只读访问器
        .def_property_readonly("tokenizer", [](const Pipeline &self) { return self.tokenizer.get(); });
}

} // namespace chatglm
```