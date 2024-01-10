# `ggml\examples\replit\main.cpp`

```
#include "ggml/ggml.h"  // 引入 ggml 库的头文件

#include "common-ggml.h"  // 引入通用 ggml 头文件
#include "common.h"  // 引入通用头文件

#include <cassert>  // 断言库
#include <cmath>  // 数学函数库
#include <cinttypes>  // 格式化输入输出库
#include <cstddef>  // C 标准库定义的一些宏和类型
#include <cstdio>  // 标准输入输出库
#include <cstring>  // 字符串库
#include <fstream>  // 文件流库
#include <iostream>  // 输入输出流库
#include <map>  // 映射容器库
#include <stdint.h>  // 标准整数类型库
#include <string>  // 字符串库
#include <unordered_map>  // 无序映射容器库
#include <utility>  // 实用程序库
#include <vector>  // 向量容器库

#if defined(_WIN32)
#define NOMINMAX
#include <Windows.h>  // Windows 系统库
bool is_stdin_terminal() {
    auto in = GetStdHandle(STD_INPUT_HANDLE);
    return GetFileType(in) == FILE_TYPE_CHAR;  // 检查标准输入是否为终端
}
#else
#include <unistd.h>  // POSIX 系统库
bool is_stdin_terminal() {
    return isatty(STDIN_FILENO);  // 检查标准输入是否为终端
}
#endif

#if defined(_MSC_VER)
#pragma warning(disable: 4244 4267) // 禁止特定警告
#endif

using piece_t = std::pair<std::size_t, float>;  // 定义 piece_t 类型为包含大小和浮点数的对
using piece_map_t = std::unordered_map<std::string, piece_t>;  // 定义 piece_map_t 类型为字符串到 piece_t 的无序映射

struct replit_tokenizer {
    gpt_vocab raw_vocab;  // 定义 gpt_vocab 类型的 raw_vocab 成员
    piece_map_t piece_map;  // 定义 piece_map_t 类型的 piece_map 成员
    std::vector<std::string> vocab;  // 定义字符串向量类型的 vocab 成员
};

std::pair<std::vector<std::size_t>, float> encode_word(const std::string & word, const piece_map_t & model) {
    std::vector<int> best_segmentations_starts(word.length() + 1, -1);  // 创建长度为单词长度加一的整数向量，初始化为-1
    best_segmentations_starts[0] = 0;  // 设置第一个元素为0

    std::vector<float> best_segmentations_scores(word.length() + 1, -std::numeric_limits<float>::infinity());  // 创建长度为单词长度加一的浮点数向量，初始化为负无穷
    best_segmentations_scores[0] = 1.0;  // 设置第一个元素为1.0
    // 遍历单词的每个可能的起始位置
    for (size_t start_idx = 0; start_idx < word.length(); ++start_idx) {
        // 获取当前起始位置的最佳分词得分
        float best_score_at_start = best_segmentations_scores[start_idx];
        // 遍历从当前起始位置到单词末尾的所有可能的结束位置
        for (size_t end_idx = start_idx + 1; end_idx <= word.length(); ++end_idx) {
            // 获取当前起始位置到当前结束位置的子串
            std::string token = word.substr(start_idx, end_idx - start_idx);
            // 如果模型中存在该子串，并且当前起始位置的最佳分词得分不是负无穷
            if (model.count(token) && best_score_at_start != -std::numeric_limits<float>::infinity()) {
                // 获取该子串在模型中的得分
                float token_score = model.at(token).second;
                // 计算当前子串的得分
                float score = token_score + best_score_at_start;
                // 如果当前结束位置的最佳分词得分是负无穷，或者大于当前计算得分
                if (best_segmentations_scores[end_idx] == -std::numeric_limits<float>::infinity() ||
                    best_segmentations_scores[end_idx] > score) {
                    // 更新当前结束位置的最佳分词起始位置和得分
                    best_segmentations_starts[end_idx] = start_idx;
                    best_segmentations_scores[end_idx] = score;
                }
            }
        }
    }

    // 如果单词的最佳分词得分是负无穷
    if (best_segmentations_scores.back() == -std::numeric_limits<float>::infinity()) {
        // 返回一个包含单个 token 的 vector 和得分为 0 的 pair
        return std::make_pair(std::vector<std::size_t>{0}, 0.0f);
    }

    // 获取单词的最佳分词得分
    float score = best_segmentations_scores.back();
    // 获取最佳分词的起始位置和结束位置
    int start = best_segmentations_starts.back();
    int end = word.length();
    // 存储分词后的 token
    std::vector<std::size_t> tokens;
    // 从最后一个 token 开始，逐步回溯获取所有 token
    while (start != 0) {
        // 获取当前 token 在模型中的 id
        const auto token_id = model.at(word.substr(start, end - start)).first;
        // 将当前 token id 插入到 tokens 的开头
        tokens.insert(tokens.begin(), token_id);
        // 获取下一个 token 的起始位置
        int next_start = best_segmentations_starts[start];
        // 更新结束位置和起始位置
        end = start;
        start = next_start;
    }
    // 获取起始位置到结束位置的 token 在模型中的 id
    const auto token_id = model.at(word.substr(start, end - start)).first;
    // 将该 token id 插入到 tokens 的开头
    tokens.insert(tokens.begin(), token_id);
    // 返回包含分词后的 tokens 和得分的 pair
    return std::make_pair(tokens, score);
// 加载分词器的词汇表
bool replit_tokenizer_load(replit_tokenizer & tokenizer, std::istream & fin, int max_vocab_size) {
    // 定义变量
    std::string word;
    // 初始化缓冲区
    std::vector<char> buf(128);

    // 遍历词汇表
    for (int i = 0; i < max_vocab_size; i++) {
        // 读取词的长度
        uint32_t len;
        fin.read((char *)&len, sizeof(len));

        // 调整缓冲区大小并读取词
        buf.resize(len);
        fin.read((char *)buf.data(), len);
        word.assign(buf.data(), len);

        // 读取词的分数并存入分词器的词汇表
        float score;
        fin.read((char *)&score, sizeof(score));
        tokenizer.piece_map[word] = std::make_pair(i, -score);
        tokenizer.raw_vocab.id_to_token[i] = word;
    }

    return true;
}

// 替换字符串中的所有指定子串
std::string replace_all(const std::string & str,    // where to work
                        const std::string & find,   // substitute 'find'
                        const std::string & replace //      by 'replace'
) {
    // 使用标准命名空间
    using namespace std;
    // 定义变量
    string result;
    size_t find_len = find.size();
    size_t pos, from = 0;
    // 遍历字符串，替换指定子串
    while (string::npos != (pos = str.find(find, from))) {
        result.append(str, from, pos - from);
        result.append(replace);
        from = pos + find_len;
    }
    result.append(str, from, string::npos);
    return result;
}

// 分词器的空白符号
std::string ws_symbol = "\342\226\201";
// 对文本进行分词
std::vector<std::size_t> replit_tokenizer_tokenize(replit_tokenizer & tokenizer, const std::string & text) {
    // 定义变量
    std::vector<std::size_t> tokens;
    // 替换空格为分词器的空白符号
    auto normalized_text = replace_all(text, " ", ws_symbol);
    // 对文本进行编码
    auto tokenized = encode_word(normalized_text, tokenizer.piece_map);

    return tokenized.first;
}

// 对分词进行反解码
std::string replit_tokenizer_detokenize(replit_tokenizer & tokenizer, const std::vector<std::size_t> & tokens) {
    // 定义变量
    std::string text;
    // 遍历分词并进行反解码
    for (auto token : tokens) {
        text += tokenizer.raw_vocab.id_to_token[token];
    }
    // 将分词器的空白符号替换为普通空格
    auto denormalized_text = replace_all(text, ws_symbol, " ");
    return denormalized_text;
}

// 分词器的超参数结构体
// 暂时没有默认值
struct replit_hparams {
    int32_t d_model = 0;
    int32_t max_seq_len = 0;
    int32_t n_heads = 0;
    int32_t n_layers = 0;
    # 定义一个32位整数变量n_vocab，并初始化为0
    int32_t n_vocab = 0;
    # 定义一个32位整数变量ftype，并初始化为0
    int32_t ftype = 0;
};

// 定义一个结构体 replit_layer，包含了模型的不同层的权重信息
struct replit_layer {
    // 预归一化
    struct ggml_tensor * norm_1_weight;

    // 注意力机制
    struct ggml_tensor * c_attn_wqkv_weight;
    struct ggml_tensor * c_attn_out_proj_weight;

    // 后归一化
    struct ggml_tensor * norm_2_weight;

    // 前馈神经网络
    struct ggml_tensor * ffn_up_proj;
    struct ggml_tensor * ffn_down_proj;
};

// 定义一个结构体 replit_model，包含了模型的超参数、权重、层信息等
struct replit_model {
    replit_hparams hparams;

    struct ggml_tensor * wte_weight;    // 位置嵌入
    struct ggml_tensor * norm_f_weight; // 语言模型头部

    std::vector<replit_layer> layers;   // 模型的不同层

    // 键值记忆
    struct ggml_tensor * memory_k;
    struct ggml_tensor * memory_v;

    struct ggml_context * ctx;
    std::map<std::string, struct ggml_tensor *> tensors;  // 张量映射
};

// 从文件中加载模型的权重
bool replit_model_load(const std::string & fname, replit_model & model, replit_tokenizer & vocab) {
    printf("%s: loading model from '%s' - please wait ...\n", __func__, fname.c_str());

    auto fin = std::ifstream(fname, std::ios::binary);
    if (!fin) {
        fprintf(stderr, "%s: failed to open '%s'\n", __func__, fname.c_str());
        return false;
    }

    // 验证魔数
    {
        uint32_t magic;
        fin.read((char *)&magic, sizeof(magic));
        if (magic != GGML_FILE_MAGIC) {
            fprintf(stderr, "%s: invalid model file '%s' (bad magic)\n", __func__, fname.c_str());
            return false;
        }
    }

    // 加载超参数
    {
        // 获取模型的超参数引用
        auto & hparams = model.hparams;

        // 从文件中读取模型的维度，并存入超参数中
        fin.read((char *)&hparams.d_model, sizeof(hparams.d_model));
        fin.read((char *)&hparams.max_seq_len, sizeof(hparams.max_seq_len));
        fin.read((char *)&hparams.n_heads, sizeof(hparams.n_heads));
        fin.read((char *)&hparams.n_layers, sizeof(hparams.n_layers));
        fin.read((char *)&hparams.n_vocab, sizeof(hparams.n_vocab));
        fin.read((char *)&hparams.ftype, sizeof(hparams.ftype));

        // 计算量化版本号
        const int32_t qntvr = hparams.ftype / GGML_QNT_VERSION_FACTOR;

        // 打印超参数信息
        printf("%s: d_model      = %d\n", __func__, hparams.d_model);
        printf("%s: max_seq_len  = %d\n", __func__, hparams.max_seq_len);
        printf("%s: n_heads      = %d\n", __func__, hparams.n_heads);
        printf("%s: n_layers     = %d\n", __func__, hparams.n_layers);
        printf("%s: n_vocab      = %d\n", __func__, hparams.n_vocab);
        printf("%s: ftype        = %d\n", __func__, hparams.ftype);
        printf("%s: qntvr        = %d\n", __func__, qntvr);

        // 计算量化版本号
        hparams.ftype %= GGML_QNT_VERSION_FACTOR;
    }

    // 加载词汇表
    replit_tokenizer_load(vocab, fin, model.hparams.n_vocab);

    // 对于大张量，我们有选项将数据存储为16位浮点数或量化，以节省内存并加快计算速度
    ggml_type wtype = ggml_ftype_to_ggml_type((ggml_ftype)(model.hparams.ftype));
    if (wtype == GGML_TYPE_COUNT) {
        // 如果模型文件的 ftype 值无效，则打印错误信息并返回
        fprintf(stderr, "%s: invalid model file '%s' (bad ftype value %d)\n", __func__, fname.c_str(),
                model.hparams.ftype);
        return false;
    }

    // 获取模型的上下文引用
    auto & ctx = model.ctx;

    // 上下文大小初始化为0
    size_t ctx_size = 0;
    {
        // 获取模型的超参数
        const auto & hparams = model.hparams;

        // 从超参数中获取模型的嵌入维度、层数、最大序列长度和词汇表大小
        const int n_embd = hparams.d_model;
        const int n_layer = hparams.n_layers;
        const int n_ctx = hparams.max_seq_len;
        const int n_vocab = hparams.n_vocab;

        // 计算上下文大小并添加相应的权重矩阵大小
        ctx_size += ggml_row_size(wtype,         n_embd*n_vocab); // wte_weight
        ctx_size += ggml_row_size(GGML_TYPE_F32, n_embd);         // ln_f_weight
        ctx_size += n_layer * (ggml_row_size(GGML_TYPE_F32, n_embd));       // ln_1_weight
        ctx_size += n_layer * (ggml_row_size(wtype, 3 * n_embd * n_embd));  // attn_Wqkv_weight
        ctx_size += n_layer * (ggml_row_size(wtype,     n_embd * n_embd)); // attn_out_proj_weight
        ctx_size += n_layer * (ggml_row_size(GGML_TYPE_F32, n_embd)); // ln_2_weight
        ctx_size += n_layer * (ggml_row_size(wtype, 4 * n_embd * n_embd)); // mlp_mlp_up_weight
        ctx_size += n_layer * (ggml_row_size(wtype, 4 * n_embd * n_embd)); // mlp_mlp_down_weight
        ctx_size += n_ctx * n_layer * ggml_row_size(GGML_TYPE_F16, n_embd); // memory_k
        ctx_size += n_ctx * n_layer * ggml_row_size(GGML_TYPE_F16, n_embd); // memory_v
        ctx_size += (1 + 6 * n_layer) * 512; // object overhead

        // 打印上下文大小
        printf("%s: ggml ctx size = %6.2f MB\n", __func__, ctx_size / (1024.0 * 1024.0));
    }

    // 创建 ggml 上下文
    {
        // 设置 ggml 初始化参数
        struct ggml_init_params params = {
            /*.mem_size   =*/ ctx_size,
            /*.mem_buffer =*/ NULL,
            /*.no_alloc   =*/ false,
        };

        // 初始化 ggml 上下文
        model.ctx = ggml_init(params);
        // 如果初始化失败，则打印错误信息并返回 false
        if (!model.ctx) {
            fprintf(stderr, "%s: ggml_init() failed\n", __func__);
            return false;
        }
    }

    // 为权重准备内存空间
    {
        // 获取模型的超参数
        const auto & hparams = model.hparams;
    
        // 获取嵌入维度、层数和词汇量大小
        const size_t n_embd = hparams.d_model;
        const size_t n_layer = hparams.n_layers;
        const size_t n_vocab = hparams.n_vocab;
    
        // 调整模型的层数
        model.layers.resize(n_layer);
    
        // 创建嵌入权重张量和 LayerNorm 权重张量
        model.wte_weight = ggml_new_tensor_2d(ctx, wtype, n_embd, n_vocab);
        model.norm_f_weight = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_embd);
    
        // 通过名称映射张量
        model.tensors["transformer.wte.weight"] = model.wte_weight;
        model.tensors["transformer.norm_f.weight"] = model.norm_f_weight;
    
        // 遍历每一层，创建对应的张量并通过名称映射
        for (int i = 0; i < (int)n_layer; ++i) {
            auto & layer = model.layers[i];
    
            layer.norm_1_weight = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_embd);
            layer.c_attn_wqkv_weight = ggml_new_tensor_2d(ctx, wtype, n_embd, 3 * n_embd);
            layer.c_attn_out_proj_weight = ggml_new_tensor_2d(ctx, wtype, n_embd, n_embd);
            layer.norm_2_weight = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_embd);
            layer.ffn_up_proj = ggml_new_tensor_2d(ctx, wtype, n_embd, 4 * n_embd);
            layer.ffn_down_proj = ggml_new_tensor_2d(ctx, wtype, 4 * n_embd, n_embd);
    
            // 通过名称映射张量
            model.tensors["transformer.blocks." + std::to_string(i) + ".norm_1.weight"] = layer.norm_1_weight;
            model.tensors["transformer.blocks." + std::to_string(i) + ".attn.Wqkv.weight"] = layer.c_attn_wqkv_weight;
            model.tensors["transformer.blocks." + std::to_string(i) + ".attn.out_proj.weight"] = layer.c_attn_out_proj_weight;
            model.tensors["transformer.blocks." + std::to_string(i) + ".norm_2.weight"] = layer.norm_2_weight;
            model.tensors["transformer.blocks." + std::to_string(i) + ".ffn.up_proj.weight"] = layer.ffn_up_proj;
            model.tensors["transformer.blocks." + std::to_string(i) + ".ffn.down_proj.weight"] = layer.ffn_down_proj;
        }
    }
    
    // key + value memory
    {
        // 获取模型的超参数
        const auto & hparams = model.hparams;
    
        // 获取嵌入维度
        const int n_embd = hparams.d_model;
        // 获取层数
        const int n_layer = hparams.n_layers;
        // 获取上下文长度
        const int n_ctx = hparams.max_seq_len;
    
        // 计算总的记忆单元数
        const int64_t n_mem = n_layer * n_ctx;
        // 计算总的元素数
        const int64_t n_elements = n_embd * n_mem;
    
        // 创建记忆的键和值的张量
        model.memory_k = ggml_new_tensor_1d(ctx, GGML_TYPE_F16, n_elements);
        model.memory_v = ggml_new_tensor_1d(ctx, GGML_TYPE_F16, n_elements);
    
        // 计算内存占用大小
        const size_t memory_size = ggml_nbytes(model.memory_k) + ggml_nbytes(model.memory_v);
    
        // 打印内存占用大小和记忆单元数
        printf("%s: memory_size = %8.2f MB, n_mem = %" PRIu64 "\n", __func__, memory_size / 1024.0 / 1024.0, n_mem);
    }
    
    // 加载权重
    }
    
    fin.close();
    
    return true;
// 评估变换器
//
//   - model:     模型
//   - n_threads: 使用的线程数
//   - n_past:    到目前为止的上下文大小
//   - embd_inp:  上下文中标记的嵌入
//   - embd_w:    下一个标记的预测对数
//
bool replit_eval(const replit_model & model, const int n_threads, const int n_past,
                 const std::vector<gpt_vocab::id> & embd_inp, std::vector<float> & embd_w, bool logits_all,
                 size_t & mem_per_token) {
    const int N = embd_inp.size();

    const auto & hparams = model.hparams;

    const int n_embd = hparams.d_model;
    const int n_layer = hparams.n_layers;
    const int n_head = hparams.n_heads;
    const int n_vocab = hparams.n_vocab;
    const int n_ctx = hparams.max_seq_len;
    const float eps = 1e-5f;

    static size_t buf_size = 256u * 1024 * 1024;
    static void * buf = malloc(buf_size);

    if (mem_per_token > 0 && mem_per_token * N > buf_size) {
        const size_t buf_size_new = 1.1 * (mem_per_token * N); // 增加10%以考虑 ggml 对象的开销
        // printf("\n%s: reallocating buffer from %zu to %zu bytes\n", __func__,
        // buf_size, buf_size_new);

        // 重新分配
        buf_size = buf_size_new;
        buf = realloc(buf, buf_size);
        if (buf == nullptr) {
            fprintf(stderr, "%s: failed to allocate %zu bytes\n", __func__, buf_size);
            return false;
        }
    }

    struct ggml_init_params params = {
        /*.mem_size   =*/ buf_size,
        /*.mem_buffer =*/ buf,
        /*.no_alloc   =*/ false,
    };

    struct ggml_context * ctx0 = ggml_init(params);
    struct ggml_cgraph * gf = ggml_new_graph(ctx0);

    struct ggml_tensor * embd = ggml_new_tensor_1d(ctx0, GGML_TYPE_I32, N);
    memcpy(embd->data, embd_inp.data(), N * ggml_element_size(embd));

    struct ggml_tensor * inpL = ggml_get_rows(ctx0, model.wte_weight, embd);

    }

    // norm
    {
        // 对输入进行规范化处理
        inpL = ggml_norm(ctx0, inpL, eps);
        // inpL = ln_f_g*inpL
        // 使用 ln_f_g 与 inpL 进行矩阵相乘
        inpL = ggml_mul(ctx0, ggml_repeat(ctx0, model.norm_f_weight, inpL), inpL);
    }

    // output embedding weight tied to input embedding
    // 使用输入的结果与 wte_weight 进行矩阵相乘
    inpL = ggml_mul_mat(ctx0, model.wte_weight, inpL);

    // logits -> probs
    // 将 logits 转换为概率
    // inpL = ggml_soft_max(ctx0, inpL);

    // run the computation
    // 运行计算
    ggml_build_forward_expand(gf, inpL);
    ggml_graph_compute_with_ctx(ctx0, gf, n_threads);

    // std::cout << "Qcur" << std::endl;
    // print_tensor(Qcur);

    // if (n_past%100 == 0) {
    // ggml_graph_print(&gf);
    // ggml_graph_dump_dot(&gf, NULL, "mpt-model.dot");
    // }

    if (logits_all) {
        // return result for all tokens
        // 返回所有标记的结果
        embd_w.resize(n_vocab * N);
        memcpy(embd_w.data(), (float *)ggml_get_data(inpL), sizeof(float) * n_vocab * N);
    } else {
        // return result for just the last token
        // 仅返回最后一个标记的结果
        embd_w.resize(n_vocab);
        memcpy(embd_w.data(), (float *)ggml_get_data(inpL) + (n_vocab * (N - 1)), sizeof(float) * n_vocab);
    }

    if (mem_per_token == 0) {
        // 计算每个标记的内存占用
        mem_per_token = ggml_used_mem(ctx0) / N;
    }
    // printf("used_mem = %zu\n", ggml_used_mem(ctx0));

    // 释放上下文
    ggml_free(ctx0);

    // 返回 true
    return true;
}

int main(int argc, char ** argv) {
    // 记录程序开始时间
    const int64_t t_main_start_us = ggml_time_us();

    // 初始化 GPT 参数对象
    gpt_params params;
    params.model = "";

    // 解析命令行参数，如果解析失败则返回 1
    if (gpt_params_parse(argc, argv, params) == false) {
        return 1;
    }

    // 如果参数中的随机种子小于 0，则使用当前时间作为种子
    if (params.seed < 0) {
        params.seed = time(NULL);
    }

    // 打印函数名和种子值
    printf("%s: seed = %d\n", __func__, params.seed);

    // 使用种子值初始化随机数生成器
    std::mt19937 rng(params.seed);

    // 如果提示为空，则根据情况生成随机提示或从标准输入读取
    if (params.prompt.empty()) {
        if (!is_stdin_terminal()) {
            std::string line;
            while (std::getline(std::cin, line)) {
                params.prompt = params.prompt + "\n" + line;
            }
        } else {
            params.prompt = gpt_random_prompt(rng);
        }
    }

    // 初始化加载时间
    int64_t t_load_us = 0;

    // 初始化词汇表和模型对象
    replit_tokenizer vocab;
    replit_model model;

    // 加载模型
    {
        const int64_t t_start_us = ggml_time_us();

        if (!replit_model_load(params.model, model, vocab)) {
            fprintf(stderr, "%s: failed to load model from '%s'\n", __func__, params.model.c_str());
            return 1;
        }

        t_load_us = ggml_time_us() - t_start_us;
    }

    // 初始化过去的标记数
    int n_past = 0;

    // 初始化采样时间和预测时间
    int64_t t_sample_us = 0;
    int64_t t_predict_us = 0;

    // 初始化 logits 数组
    std::vector<float> logits;

    // 对提示进行标记化
    std::vector<std::size_t> embd_inp = replit_tokenizer_tokenize(vocab, params.prompt);

    // 打印提示中的标记数
    printf("%s: number of tokens in prompt = %zu\n", __func__, embd_inp.size());

    // 打印每个标记的索引和值
    for (size_t i = 0; i < embd_inp.size(); i++) {
        printf("%s: token[%zu] = %6zu\n", __func__, i, embd_inp[i]);
        // vocab.id_to_token.at(embd_inp[i]).c_str()
    }
    printf("\n");

    // 确定每个标记所需的推理内存
    size_t mem_per_token = 0;
    replit_eval(model, params.n_threads, 0, {0, 1, 2, 3}, logits, false, mem_per_token);
}
    // 从当前嵌入大小开始循环直到嵌入输入大小加上预测数
    for (size_t i = embd.size(); i < embd_inp.size() + params.n_predict; i++) {
        // 预测
        if (embd.size() > 0) {
            // 记录开始预测的时间
            const int64_t t_start_us = ggml_time_us();

            // 如果无法成功预测，则打印错误信息并返回1
            if (!replit_eval(model, params.n_threads, n_past, embd, logits, false, mem_per_token)) {
                printf("Failed to predict\n");
                return 1;
            }

            // 计算预测时间并累加到总预测时间
            t_predict_us += ggml_time_us() - t_start_us;
        }

        // 更新过去的数量
        n_past += embd.size();
        // 清空嵌入
        embd.clear();

        // 如果当前索引大于等于嵌入输入大小
        if (i >= embd_inp.size()) {
            // 采样下一个标记
            const int top_k = params.top_k;
            const float top_p = params.top_p;
            const float temp = params.temp;

            const int n_vocab = model.hparams.n_vocab;

            gpt_vocab::id id = 0;

            {
                // 记录开始采样的时间
                const int64_t t_start_sample_us = ggml_time_us();

                // 从logits中采样下一个标记
                id = gpt_sample_top_k_top_p(vocab.raw_vocab, logits.data() + (logits.size() - n_vocab), top_k, top_p,
                                            temp, rng);

                // 计算采样时间并累加到总采样时间
                t_sample_us += ggml_time_us() - t_start_sample_us;
            }

            // 将采样的标记添加到上下文中
            embd.push_back(id);
        } else {
            // 如果执行到这里，意味着我们仍在处理输入提示
            for (size_t k = i; k < embd_inp.size(); k++) {
                embd.push_back(embd_inp[k]);
                // 如果嵌入大小超过批处理大小，则跳出循环
                if (int32_t(embd.size()) > params.n_batch) {
                    break;
                }
            }
            // 更新索引
            i += embd.size() - 1;
        }

        // 显示文本
        for (auto id : embd) {
            printf("%s", replit_tokenizer_detokenize(vocab, {static_cast<std::size_t>(id)}).c_str());
        }
        fflush(stdout);

        // 文本结束标记
        if (embd.back() == 0) {
            break;
        }
    }

    // 报告时间
    {
        // 记录主函数结束时间
        const int64_t t_main_end_us = ggml_time_us();

        // 打印内存每个标记的大小
        printf("\n\n");
        printf("%s: mem per token = %8zu bytes\n", __func__, mem_per_token);
        // 打印加载时间
        printf("%s:     load time = %8.2f ms\n", __func__, t_load_us / 1000.0f);
        // 打印采样时间
        printf("%s:   sample time = %8.2f ms\n", __func__, t_sample_us / 1000.0f);
        // 打印预测时间和每个标记的平均预测时间
        printf("%s:  predict time = %8.2f ms / %.2f ms per token\n", __func__, t_predict_us / 1000.0f,
               t_predict_us / 1000.0f / n_past);
        // 打印总时间
        printf("%s:    total time = %8.2f ms\n", __func__, (t_main_end_us - t_main_start_us) / 1000.0f);
    }

    // 释放模型上下文
    ggml_free(model.ctx);

    // 返回成功状态码
    return 0;
# 闭合前面的函数定义
```