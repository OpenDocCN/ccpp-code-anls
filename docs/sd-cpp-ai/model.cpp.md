# `stable-diffusion.cpp\model.cpp`

```
#include <stdarg.h>
#include <fstream>
#include <regex>
#include <set>
#include <string>
#include <unordered_map>
#include <vector>

#include "model.h"
#include "stable-diffusion.h"
#include "util.h"
#include "vocab.hpp"

#include "ggml/ggml-alloc.h"
#include "ggml/ggml-backend.h"
#include "ggml/ggml.h"

#include "stable-diffusion.h"

#ifdef SD_USE_METAL
#include "ggml-metal.h"
#endif

#define ST_HEADER_SIZE_LEN 8

// 读取 64 位无符号整数
uint64_t read_u64(uint8_t* buffer) {
    // little endian
    uint64_t value = 0;
    value |= static_cast<int64_t>(buffer[7]) << 56;
    value |= static_cast<int64_t>(buffer[6]) << 48;
    value |= static_cast<int64_t>(buffer[5]) << 40;
    value |= static_cast<int64_t>(buffer[4]) << 32;
    value |= static_cast<int64_t>(buffer[3]) << 24;
    value |= static_cast<int64_t>(buffer[2]) << 16;
    value |= static_cast<int64_t>(buffer[1]) << 8;
    value |= static_cast<int64_t>(buffer[0]);
    return value;
}

// 读取 32 位整数
int32_t read_int(uint8_t* buffer) {
    // little endian
    int value = 0;
    value |= buffer[3] << 24;
    value |= buffer[2] << 16;
    value |= buffer[1] << 8;
    value |= buffer[0];
    return value;
}

// 读取 16 位无符号整数
uint16_t read_short(uint8_t* buffer) {
    // little endian
    uint16_t value = 0;
    value |= buffer[1] << 8;
    value |= buffer[0];
    return value;
}

/*================================================= Preprocess ==================================================*/

// 自注意力层权重名称
std::string self_attn_names[] = {
    "self_attn.q_proj.weight",
    "self_attn.k_proj.weight",
    "self_attn.v_proj.weight",
    "self_attn.q_proj.bias",
    "self_attn.k_proj.bias",
    "self_attn.v_proj.bias",
};

// 未使用的张量名称
const char* unused_tensors[] = {
    "betas",
    "alphas_cumprod_prev",
    "sqrt_alphas_cumprod",
    "sqrt_one_minus_alphas_cumprod",
    "log_one_minus_alphas_cumprod",
    "sqrt_recip_alphas_cumprod",
    "sqrt_recipm1_alphas_cumprod",
    "posterior_variance",
    "posterior_log_variance_clipped",
    "posterior_mean_coef1",
    "posterior_mean_coef2",
};
    # 定义一系列字符串，表示需要处理的模型参数的路径
    "cond_stage_model.transformer.text_model.embeddings.position_ids",
    "cond_stage_model.model.logit_scale",
    "cond_stage_model.model.text_projection",
    "conditioner.embedders.0.transformer.text_model.embeddings.position_ids",
    "conditioner.embedders.0.model.logit_scale",
    "conditioner.embedders.1.model.logit_scale",
    "model.diffusion_model.time_embedding.cond_proj.weight",
    "unet.time_embedding.cond_proj.weight",
    "model_ema.decay",
    "model_ema.num_updates",
    "model_ema.diffusion_model",
    "embedding_manager",
    "denoiser.sigmas",
// 检查给定名称是否为未使用的张量
bool is_unused_tensor(std::string name) {
    // 遍历未使用张量数组，检查是否以某个未使用张量名称开头
    for (int i = 0; i < sizeof(unused_tensors) / sizeof(const char*); i++) {
        if (starts_with(name, unused_tensors[i])) {
            return true;
        }
    }
    return false;
}

// 将 OpenAI 的模型名称映射到 Hugging Face 的模型名称
std::unordered_map<std::string, std::string> open_clip_to_hf_clip_model = {
    {"model.ln_final.bias", "transformer.text_model.final_layer_norm.bias"},
    {"model.ln_final.weight", "transformer.text_model.final_layer_norm.weight"},
    {"model.positional_embedding", "transformer.text_model.embeddings.position_embedding.weight"},
    {"model.token_embedding.weight", "transformer.text_model.embeddings.token_embedding.weight"},
    {"model.text_projection", "transformer.text_model.text_projection"},
};

// 将 OpenAI 的模型名称映射到 Hugging Face 的残差块名称
std::unordered_map<std::string, std::string> open_clip_to_hk_clip_resblock = {
    {"attn.out_proj.bias", "self_attn.out_proj.bias"},
    {"attn.out_proj.weight", "self_attn.out_proj.weight"},
    {"ln_1.bias", "layer_norm1.bias"},
    {"ln_1.weight", "layer_norm1.weight"},
    {"ln_2.bias", "layer_norm2.bias"},
    {"ln_2.weight", "layer_norm2.weight"},
    {"mlp.c_fc.bias", "mlp.fc1.bias"},
    {"mlp.c_fc.weight", "mlp.fc1.weight"},
    {"mlp.c_proj.bias", "mlp.fc2.bias"},
    {"mlp.c_proj.weight", "mlp.fc2.weight"},
};

// 将 VAE 解码器的名称映射到新的名称
std::unordered_map<std::string, std::string> vae_decoder_name_map = {
    {"first_stage_model.decoder.mid.attn_1.to_k.bias", "first_stage_model.decoder.mid.attn_1.k.bias"},
    {"first_stage_model.decoder.mid.attn_1.to_k.weight", "first_stage_model.decoder.mid.attn_1.k.weight"},
    {"first_stage_model.decoder.mid.attn_1.to_out.0.bias", "first_stage_model.decoder.mid.attn_1.proj_out.bias"},
    {"first_stage_model.decoder.mid.attn_1.to_out.0.weight", "first_stage_model.decoder.mid.attn_1.proj_out.weight"},
    {"first_stage_model.decoder.mid.attn_1.to_q.bias", "first_stage_model.decoder.mid.attn_1.q.bias"},
    {"first_stage_model.decoder.mid.attn_1.to_q.weight", "first_stage_model.decoder.mid.attn_1.q.weight"},
};
    # 创建包含两个字符串集合的字典，用于存储模型参数的权重和偏置
    {"first_stage_model.decoder.mid.attn_1.to_v.bias", "first_stage_model.decoder.mid.attn_1.v.bias"},
    # 创建包含两个字符串集合的字典，用于存储模型参数的权重和偏置
    {"first_stage_model.decoder.mid.attn_1.to_v.weight", "first_stage_model.decoder.mid.attn_1.v.weight"},
// 将 OpenAI 的模型名称转换为 Hugging Face 的模型名称
std::string convert_open_clip_to_hf_clip(const std::string& name) {
    // 复制输入名称
    std::string new_name = name;
    std::string prefix;
    // 检查是否以特定前缀开头，进行相应的处理
    if (starts_with(new_name, "conditioner.embedders.0.")) {
        prefix   = "cond_stage_model.";
        new_name = new_name.substr(strlen("conditioner.embedders.0."));
    } else if (starts_with(new_name, "conditioner.embedders.1.")) {
        prefix   = "cond_stage_model.1.";
        new_name = new_name.substr(strlen("conditioner.embedders.0."));
    } else if (starts_with(new_name, "cond_stage_model.")) {
        prefix   = "cond_stage_model.";
        new_name = new_name.substr(strlen("cond_stage_model."));
    } else {
        return new_name;
    }
    // 定义 OpenAI 模型和 Hugging Face 模型的前缀
    std::string open_clip_resblock_prefix = "model.transformer.resblocks.";
    std::string hf_clip_resblock_prefix   = "transformer.text_model.encoder.layers.";

    // 如果在映射表中找到对应的模型名称，则替换为 Hugging Face 的模型名称
    if (open_clip_to_hf_clip_model.find(new_name) != open_clip_to_hf_clip_model.end()) {
        new_name = open_clip_to_hf_clip_model[new_name];
    }

    // 如果名称以特定前缀开头，则进行相应的处理
    if (new_name.find(open_clip_resblock_prefix) == 0) {
        std::string remain = new_name.substr(open_clip_resblock_prefix.length());
        std::string idx    = remain.substr(0, remain.find("."));
        std::string suffix = remain.substr(idx.length() + 1);

        // 根据后缀进行不同的处理
        if (suffix == "attn.in_proj_weight" || suffix == "attn.in_proj_bias") {
            new_name = hf_clip_resblock_prefix + idx + "." + suffix;
        } else if (open_clip_to_hk_clip_resblock.find(suffix) != open_clip_to_hk_clip_resblock.end()) {
            std::string new_suffix = open_clip_to_hk_clip_resblock[suffix];
            new_name               = hf_clip_resblock_prefix + idx + "." + new_suffix;
        }
    }

    // 返回处理后的模型名称
    return prefix + new_name;
}

// 将 VAE 解码器的名称转换为映射表中的名称
std::string convert_vae_decoder_name(const std::string& name) {
    // 如果在映射表中找到对应的名称，则返回映射后的名称
    if (vae_decoder_name_map.find(name) != vae_decoder_name_map.end()) {
        return vae_decoder_name_map[name];
    }
    // 否则返回原始名称
    return name;
}
// 定义一个嵌套的无序映射表，用于将不同后缀转换为计算机视觉中的命名约定，下划线版本
std::unordered_map<std::string, std::unordered_map<std::string, std::string>> suffix_conversion_underline = {
    {
        "attentions",
        {
            {"to_k", "k"},
            {"to_q", "q"},
            {"to_v", "v"},
            {"to_out_0", "proj_out"},
            {"group_norm", "norm"},
        },
    },
    {
        "resnets",
        {
            {"conv1", "in_layers_2"},
            {"conv2", "out_layers_3"},
            {"norm1", "in_layers_0"},
            {"norm2", "out_layers_0"},
            {"time_emb_proj", "emb_layers_1"},
            {"conv_shortcut", "skip_connection"},
        },
    },
};

// 定义一个嵌套的无序映射表，用于将不同后缀转换为计算机视觉中的命名约定，点号版本
std::unordered_map<std::string, std::unordered_map<std::string, std::string>> suffix_conversion_dot = {
    {
        "attentions",
        {
            {"to_k", "k"},
            {"to_q", "q"},
            {"to_v", "v"},
            {"to_out.0", "proj_out"},
            {"group_norm", "norm"},
        },
    },
    {
        "resnets",
        {
            {"conv1", "in_layers.2"},
            {"conv2", "out_layers.3"},
            {"norm1", "in_layers.0"},
            {"norm2", "out_layers.0"},
            {"time_emb_proj", "emb_layers.1"},
            {"conv_shortcut", "skip_connection"},
        },
    },
};

// 将不同后缀的名称转换为计算机视觉中的命名约定
std::string convert_diffusers_name_to_compvis(const std::string& key, char seq) {
    // 定义一个字符串向量
    std::vector<std::string> m;

    // 定义一个 lambda 函数，用于匹配正则表达式并提取匹配结果
    auto match = [](std::vector<std::string>& match_list, const std::regex& regex, const std::string& key) {
        auto r = std::smatch{};
        // 如果 key 与正则表达式不匹配，则返回 false
        if (!std::regex_match(key, r, regex)) {
            return false;
        }

        // 清空匹配列表
        match_list.clear();
        // 将匹配结果存入匹配列表
        for (size_t i = 1; i < r.size(); ++i) {
            match_list.push_back(r.str(i));
        }
        return true;
    };

    // 定义一个嵌套的无序映射表，用于存储后缀转换规则
    std::unordered_map<std::string, std::unordered_map<std::string, std::string>> suffix_conversion;
    // 根据不同的分隔符选择不同的后缀转换规则
    if (seq == '_') {
        suffix_conversion = suffix_conversion_underline;
    } else {
        suffix_conversion = suffix_conversion_dot;
    // 定义 lambda 函数 get_converted_suffix，用于获取转换后的后缀
    auto get_converted_suffix = [&suffix_conversion](const std::string& outer_key, const std::string& inner_key) {
        // 查找外部键在后缀转换字典中的位置
        auto outer_iter = suffix_conversion.find(outer_key);
        // 如果找到外部键
        if (outer_iter != suffix_conversion.end()) {
            // 查找内部键在外部键对应值的位置
            auto inner_iter = outer_iter->second.find(inner_key);
            // 如果找到内部键
            if (inner_iter != outer_iter->second.end()) {
                // 返回内部键对应的值
                return inner_iter->second;
            }
        }
        // 如果未找到内部键，返回原内部键
        return inner_key;
    };

    // 匹配 unet 模式中的 conv_in
    if (match(m, std::regex(format("unet%cconv_in(.*)", seq)), key)) {
        // 返回对应的模型输入块
        return format("model%cdiffusion_model%cinput_blocks%c0%c0", seq, seq, seq, seq) + m[0];
    }

    // 匹配 unet 模式中的 conv_out
    if (match(m, std::regex(format("unet%cconv%cout(.*)", seq, seq)), key)) {
        // 返回对应的模型输出块
        return format("model%cdiffusion_model%cout%c2", seq, seq, seq) + m[0];
    }

    // 匹配 unet 模式中的 conv_norm_out
    if (match(m, std::regex(format("unet%cconv_norm_out(.*)", seq)), key)) {
        // 返回对应的模型输出块
        return format("model%cdiffusion_model%cout%c0", seq, seq, seq) + m[0];
    }

    // 匹配 unet 模式中的 time_embedding_linear
    if (match(m, std::regex(format("unet%ctime_embedding%clinear_(\\d+)(.*)", seq, seq)), key)) {
        // 返回对应的时间嵌入块
        return format("model%cdiffusion_model%ctime_embed%c", seq, seq, seq) + std::to_string(std::stoi(m[0]) * 2 - 2) + m[1];
    }

    // 匹配 unet 模式中的 down_blocks
    if (match(m, std::regex(format("unet%cdown_blocks%c(\\d+)%c(attentions|resnets)%c(\\d+)%c(.+)", seq, seq, seq, seq, seq)), key)) {
        // 获取转换后的后缀
        std::string suffix = get_converted_suffix(m[1], m[3]);
        // LOG_DEBUG("%s %s %s %s", m[0].c_str(), m[1].c_str(), m[2].c_str(), m[3].c_str());
        // 返回对应的输入块
        return format("model%cdiffusion_model%cinput_blocks%c", seq, seq, seq) + std::to_string(1 + std::stoi(m[0]) * 3 + std::stoi(m[2])) + seq +
               (m[1] == "attentions" ? "1" : "0") + seq + suffix;
    }
    // 如果匹配到特定格式的字符串，提取关键信息并生成新的字符串返回
    if (match(m, std::regex(format("unet%cmid_block%c(attentions|resnets)%c(\\d+)%c(.+)", seq, seq, seq, seq)), key)) {
        // 获取转换后的后缀
        std::string suffix = get_converted_suffix(m[0], m[2]);
        // 根据匹配结果生成新的字符串并返回
        return format("model%cdiffusion_model%cmiddle_block%c", seq, seq, seq) + (m[0] == "attentions" ? "1" : std::to_string(std::stoi(m[1]) * 2)) +
               seq + suffix;
    }

    // 如果匹配到特定格式的字符串，提取关键信息并生成新的字符串返回
    if (match(m, std::regex(format("unet%cup_blocks%c(\\d+)%c(attentions|resnets)%c(\\d+)%c(.+)", seq, seq, seq, seq, seq)), key)) {
        // 获取转换后的后缀
        std::string suffix = get_converted_suffix(m[1], m[3]);
        // 根据匹配结果生成新的字符串并返回
        return format("model%cdiffusion_model%coutput_blocks%c", seq, seq, seq) + std::to_string(std::stoi(m[0]) * 3 + std::stoi(m[2])) + seq +
               (m[1] == "attentions" ? "1" : "0") + seq + suffix;
    }

    // 如果匹配到特定格式的字符串，生成新的字符串返回
    if (match(m, std::regex(format("unet%cdown_blocks%c(\\d+)%cdownsamplers%c0%cconv", seq, seq, seq, seq, seq)), key)) {
        // 根据匹配结果生成新的字符串并返回
        return format("model%cdiffusion_model%cinput_blocks%c", seq, seq, seq) + std::to_string(3 + std::stoi(m[0]) * 3) + seq + "0" + seq + "op";
    }

    // 如果匹配到特定格式的字符串，生成新的字符串返回
    if (match(m, std::regex(format("unet%cup_blocks%c(\\d+)%cupsamplers%c0%cconv", seq, seq, seq, seq, seq)), key)) {
        // 根据匹配结果生成新的字符串并返回
        return format("model%cdiffusion_model%coutput_blocks%c", seq, seq, seq) + std::to_string(2 + std::stoi(m[0]) * 3) + seq +
               (std::stoi(m[0]) > 0 ? "2" : "1") + seq + "conv";
    }

    // clip
    // 如果匹配到特定格式的字符串，生成新的字符串返回
    if (match(m, std::regex(format("te%ctext_model%cencoder%clayers%c(\\d+)%c(.+)", seq, seq, seq, seq, seq)), key)) {
        // 根据匹配结果生成新的字符串并返回
        return format("cond_stage_model%ctransformer%ctext_model%cencoder%clayers%c", seq, seq, seq, seq, seq) + m[0] + seq + m[1];
    }

    // 如果匹配到特定格式的字符串，生成新的字符串返回
    if (match(m, std::regex(format("te%ctext_model(.*)", seq)), key)) {
        // 根据匹配结果生成新的字符串并返回
        return format("cond_stage_model%ctransformer%ctext_model", seq, seq) + m[0];
    }

    // vae
    // 如果匹配到指定格式的字符串，提取关键信息并返回相应的格式化字符串
    if (match(m, std::regex(format("vae%c(.*)%cconv_norm_out(.*)", seq, seq)), key)) {
        return format("first_stage_model%c%s%cnorm_out%s", seq, m[0].c_str(), seq, m[1].c_str());
    }

    // 如果匹配到指定格式的字符串，提取关键信息并返回相应的格式化字符串
    if (match(m, std::regex(format("vae%c(.*)%cmid_block%c(attentions|resnets)%c(\\d+)%c(.+)", seq, seq, seq, seq, seq)), key)) {
        std::string suffix;
        std::string block_name;
        // 根据不同情况设置块名称和后缀
        if (m[1] == "attentions") {
            block_name = "attn";
            suffix     = get_converted_suffix(m[1], m[3]);
        } else {
            block_name = "block";
            suffix     = m[3];
        }
        return format("first_stage_model%c%s%cmid%c%s_%d%c%s",
                      seq, m[0].c_str(), seq, seq, block_name.c_str(), std::stoi(m[2]) + 1, seq, suffix.c_str());
    }

    // 如果匹配到指定格式的字符串，提取关键信息并返回相应的格式化字符串
    if (match(m, std::regex(format("vae%c(.*)%cup_blocks%c(\\d+)%cresnets%c(\\d+)%c(.+)", seq, seq, seq, seq, seq, seq)), key)) {
        std::string suffix = m[3];
        // 如果后缀为"conv_shortcut"，替换为"nin_shortcut"
        if (suffix == "conv_shortcut") {
            suffix = "nin_shortcut";
        }
        return format("first_stage_model%c%s%cup%c%d%cblock%c%s%c%s",
                      seq, m[0].c_str(), seq, seq, 3 - std::stoi(m[1]), seq, seq, m[2].c_str(), seq, suffix.c_str());
    }

    // 如果匹配到指定格式的字符串，提取关键信息并返回相应的格式化字符串
    if (match(m, std::regex(format("vae%c(.*)%cdown_blocks%c(\\d+)%cdownsamplers%c0%cconv", seq, seq, seq, seq, seq, seq)), key)) {
        return format("first_stage_model%c%s%cdown%c%d%cdownsample%cconv",
                      seq, m[0].c_str(), seq, seq, std::stoi(m[1]), seq, seq);
    }

    // 如果匹配到指定格式的字符串，提取关键信息并返回相应的格式化字符串
    if (match(m, std::regex(format("vae%c(.*)%cdown_blocks%c(\\d+)%cresnets%c(\\d+)%c(.+)", seq, seq, seq, seq, seq, seq)), key)) {
        std::string suffix = m[3];
        // 如果后缀为"conv_shortcut"，替换为"nin_shortcut"
        if (suffix == "conv_shortcut") {
            suffix = "nin_shortcut";
        }
        return format("first_stage_model%c%s%cdown%c%d%cblock%c%s%c%s",
                      seq, m[0].c_str(), seq, seq, std::stoi(m[1]), seq, seq, m[2].c_str(), seq, suffix.c_str());
    }
    // 如果匹配到指定格式的字符串，提取关键信息并返回相应的格式化字符串
    if (match(m, std::regex(format("vae%c(.*)%cup_blocks%c(\\d+)%cupsamplers%c0%cconv", seq, seq, seq, seq, seq, seq)), key)) {
        return format("first_stage_model%c%s%cup%c%d%cupsample%cconv",
                      seq, m[0].c_str(), seq, seq, 3 - std::stoi(m[1]), seq, seq);
    }

    // 如果匹配到指定格式的字符串，提取关键信息并返回相应的格式化字符串
    if (match(m, std::regex(format("vae%c(.*)", seq)), key)) {
        return format("first_stage_model%c", seq) + m[0];
    }

    // 如果没有匹配到任何格式的字符串，直接返回原始的关键信息
    return key;
// 将给定的张量名称转换为新的张量名称
std::string convert_tensor_name(const std::string& name) {
    std::string new_name;
    // 如果名称以"cond_stage_model."或"conditioner.embedders."开头，则调用convert_open_clip_to_hf_clip函数进行转换
    if (starts_with(name, "cond_stage_model.") || starts_with(name, "conditioner.embedders.")) {
        new_name = convert_open_clip_to_hf_clip(name);
    } 
    // 如果名称以"first_stage_model.decoder"开头，则调用convert_vae_decoder_name函数进行转换
    else if (starts_with(name, "first_stage_model.decoder")) {
        new_name = convert_vae_decoder_name(name);
    } 
    // 如果名称以"control_model."开头，则处理controlnet pth模型的情况
    else if (starts_with(name, "control_model.")) {
        // 查找第一个"."的位置
        size_t pos = name.find('.');
        // 如果找到了"."，则截取"."后面的部分作为新名称
        if (pos != std::string::npos) {
            new_name = name.substr(pos + 1);
        }
    } 
    // 如果名称以"lora_"开头，则处理lora的情况
    else if (starts_with(name, "lora_")) {
        // 查找第一个"."的位置
        size_t pos = name.find('.');
        // 如果找到了"."，则处理名称中的网络部分和非网络部分
        if (pos != std::string::npos) {
            // 截取非网络部分的名称
            std::string name_without_network_parts = name.substr(5, pos - 5);
            // 截取网络部分的名称
            std::string network_part = name.substr(pos + 1);
            // 调用convert_diffusers_name_to_compvis函数将非网络部分名称转换为新的键
            std::string new_key = convert_diffusers_name_to_compvis(name_without_network_parts, '_');
            // 如果新键为空，则保持原名称，否则构建新名称
            if (new_key.empty()) {
                new_name = name;
            } else {
                new_name = "lora." + new_key + "." + network_part;
            }
        } else {
            new_name = name;
        }
    }
    } else if (starts_with(name, "unet") || starts_with(name, "vae") || starts_with(name, "te")) {  // 检查名称是否以"unet"、"vae"或"te"开头，用于diffuser
        // 查找最后一个"."的位置
        size_t pos = name.find_last_of('.');
        if (pos != std::string::npos) {
            // 提取不包含网络部分的名称
            std::string name_without_network_parts = name.substr(0, pos);
            // 提取网络部分
            std::string network_part = name.substr(pos + 1);
            // LOG_DEBUG("%s %s", name_without_network_parts.c_str(), network_part.c_str());
            // 将diffusers名称转换为compvis名称
            std::string new_key = convert_diffusers_name_to_compvis(name_without_network_parts, '.');
            if (new_key.empty()) {
                new_name = name;
            } else {
                new_name = new_key + "." + network_part;
            }
        } else {
            new_name = name;
        }
    } else {
        new_name = name;
    }
    // 如果新名称不等于原名称，则输出调试信息
    // if (new_name != name) {
    //     LOG_DEBUG("%s => %s", name.c_str(), new_name.c_str());
    // }
    // 返回新名称
    return new_name;
    // 预处理张量，将处理后的张量存储到 processed_tensor_storages 中
    void preprocess_tensor(TensorStorage tensor_storage,
                           std::vector<TensorStorage>& processed_tensor_storages) {
        // 创建一个结果向量
        std::vector<TensorStorage> result;
        // 将张量名称转换为新名称
        std::string new_name = convert_tensor_name(tensor_storage.name);

        // 如果新名称以特定字符串开头并且以特定字符串结尾，则对张量进行unsqueeze操作
        if (starts_with(new_name, "model.diffusion_model.") &&
            (ends_with(new_name, "proj_in.weight") || ends_with(new_name, "proj_out.weight"))) {
            tensor_storage.unsqueeze();
        }

        // 如果新名称以特定字符串开头并且包含特定子字符串，则对张量进行unsqueeze操作
        if (starts_with(new_name, "first_stage_model.") && new_name.find("attn_1") != std::string::npos) {
            tensor_storage.unsqueeze();
        }

        // 更新张量的名称为新名称
        tensor_storage.name = new_name;

        // 如果新名称包含特定子字符串并且以特定字符串结尾，则对张量进行分块处理
        if (new_name.find("transformer.text_model.encoder.layers.") != std::string::npos &&
            ends_with(new_name, "attn.in_proj_weight")) {
            // 获取前缀的大小和内容
            size_t prefix_size = new_name.find("attn.in_proj_weight");
            std::string prefix = new_name.substr(0, prefix_size);

            // 将张量分成三块
            std::vector<TensorStorage> chunks = tensor_storage.chunk(3);
            // 更新每块的名称
            chunks[0].name = prefix + "self_attn.q_proj.weight";
            chunks[1].name = prefix + "self_attn.k_proj.weight";
            chunks[2].name = prefix + "self_attn.v_proj.weight";

            // 将处理后的张量块添加到 processed_tensor_storages 中
            processed_tensor_storages.insert(processed_tensor_storages.end(), chunks.begin(), chunks.end());
    // 如果新名称中包含特定字符串并且以特定字符串结尾
    } else if (new_name.find("transformer.text_model.encoder.layers.") != std::string::npos &&
               ends_with(new_name, "attn.in_proj_bias")) {
        // 获取特定字符串的位置作为前缀大小
        size_t prefix_size = new_name.find("attn.in_proj_bias");
        // 提取前缀
        std::string prefix = new_name.substr(0, prefix_size);

        // 将张量存储分成三个部分
        std::vector<TensorStorage> chunks = tensor_storage.chunk(3);
        // 设置每个部分的名称
        chunks[0].name                    = prefix + "self_attn.q_proj.bias";
        chunks[1].name                    = prefix + "self_attn.k_proj.bias";
        chunks[2].name                    = prefix + "self_attn.v_proj.bias";

        // 将处理后的张量存储添加到已处理的张量存储列表中
        processed_tensor_storages.insert(processed_tensor_storages.end(), chunks.begin(), chunks.end());
    } else {
        // 如果不符合条件，则将原始张量存储添加到已处理的张量存储列表中
        processed_tensor_storages.push_back(tensor_storage);
    }
// 将 bfloat16 类型数据转换为 float 类型数据
float bf16_to_f32(uint16_t bfloat16) {
    // 将 bfloat16 左移 16 位，得到 32 位整数
    uint32_t val_bits = (static_cast<uint32_t>(bfloat16) << 16);
    // 将整数解释为 float 类型数据并返回
    return *reinterpret_cast<float*>(&val_bits);
}

// 将 bfloat16 类型数组转换为 float 类型数组
void bf16_to_f32_vec(uint16_t* src, float* dst, int64_t n) {
    // 支持原地操作
    for (int64_t i = n - 1; i >= 0; i--) {
        // 调用 bf16_to_f32 函数将每个元素转换为 float 类型并存储到目标数组中
        dst[i] = bf16_to_f32(src[i]);
    }
}

// 将源数据转换为目标数据类型
void convert_tensor(void* src, ggml_type src_type, void* dst, ggml_type dst_type, int n) {
    // 如果源数据类型和目标数据类型相同
    if (src_type == dst_type) {
        // 计算需要拷贝的字节数
        size_t nbytes = n * ggml_type_size(src_type) / ggml_blck_size(src_type);
        // 使用 memcpy 函数将源数据拷贝到目标数据
        memcpy(((char*)dst), ((char*)src), nbytes);
    } 
    // 如果源数据类型为 GGML_TYPE_F32
    else if (src_type == GGML_TYPE_F32) {
        // 如果目标数据类型为 GGML_TYPE_F16
        if (dst_type == GGML_TYPE_F16) {
            // 调用 ggml_fp32_to_fp16_row 函数将 float 类型数组转换为 ggml_fp16_t 类型数组
            ggml_fp32_to_fp16_row((float*)src, (ggml_fp16_t*)dst, n);
        } 
        // 如果目标数据类型不为 GGML_TYPE_F16
        else {
            // 创建长度为 16 的整型数组 hist
            int64_t hist[16];
            // 调用 ggml_quantize_chunk 函数进行量化操作
            ggml_quantize_chunk(dst_type, (float*)src, dst, 0, n, hist);
        }
    } 
    // 如果目标数据类型为 GGML_TYPE_F32
    else if (dst_type == GGML_TYPE_F32) {
        // 如果源数据类型为 GGML_TYPE_F16
        if (src_type == GGML_TYPE_F16) {
            // 调用 ggml_fp16_to_fp32_row 函数将 ggml_fp16_t 类型数组转换为 float 类型数组
            ggml_fp16_to_fp32_row((ggml_fp16_t*)src, (float*)dst, n);
        } 
        // 如果源数据类型不为 GGML_TYPE_F16
        else {
            // 获取源数据类型的类型特征
            auto qtype = ggml_internal_get_type_traits(src_type);
            // 如果没有浮点数解量化函数，则抛出异常
            if (qtype.to_float == NULL) {
                throw std::runtime_error(format("type %s unsupported for integer quantization: no dequantization available",
                                                ggml_type_name(src_type)));
            }
            // 调用类型特征中的 to_float 函数进行解量化操作
            qtype.to_float(src, (float*)dst, n);
        }
    }
}
    } else {
        // 如果源类型为 GGML_TYPE_F16，则目标类型为量化类型
        // 如果源类型为量化类型，则目标类型为 GGML_TYPE_F16 或者目标类型为量化类型
        // 获取源类型的量化信息
        auto qtype = ggml_internal_get_type_traits(src_type);
        // 如果没有反量化函数，则抛出运行时错误
        if (qtype.to_float == NULL) {
            throw std::runtime_error(format("type %s unsupported for integer quantization: no dequantization available",
                                            ggml_type_name(src_type)));
        }
        // 创建一个存储数据的缓冲区
        std::vector<char> buf;
        buf.resize(sizeof(float) * n);
        char* src_data_f32 = buf.data();
        // 将源数据转换为 float 类型
        qtype.to_float(src, (float*)src_data_f32, n);
        // 如果目标类型为 GGML_TYPE_F16
        if (dst_type == GGML_TYPE_F16) {
            // 将 float 类型数据转换为 ggml_fp16_t 类型数据
            ggml_fp32_to_fp16_row((float*)src_data_f32, (ggml_fp16_t*)dst, n);
        } else {
            // 创建一个长度为 16 的直方图数组
            int64_t hist[16];
            // 将 float 类型数据量化为目标类型数据
            ggml_quantize_chunk(dst_type, (float*)src_data_f32, dst, 0, n, hist);
        }
    }
// 结束 ModelLoader 命名空间
}

/*================================================= ModelLoader ==================================================*/

// 从 https://github.com/openai/CLIP/blob/main/clip/simple_tokenizer.py#L16 转移而来，将 Unicode 转换为字节
std::map<char, int> unicode_to_byte() {
    std::map<int, char> byte_to_unicode;

    // 列出 utf-8 字节范围
    for (int b = static_cast<int>('!'); b <= static_cast<int>('~'); ++b) {
        byte_to_unicode[b] = static_cast<char>(b);
    }

    for (int b = 49825; b <= 49836; ++b) {
        byte_to_unicode[b] = static_cast<char>(b);
    }

    for (int b = 49838; b <= 50111; ++b) {
        byte_to_unicode[b] = static_cast<char>(b);
    }
    // printf("%d %d %d %d\n", static_cast<int>('¡'), static_cast<int>('¬'), static_cast<int>('®'), static_cast<int>('ÿ'));
    // exit(1);

    int n = 0;
    for (int b = 0; b < 256; ++b) {
        if (byte_to_unicode.find(b) == byte_to_unicode.end()) {
            byte_to_unicode[b] = static_cast<char>(256 + n);
            n++;
        }
    }

    // byte_encoder = bytes_to_unicode()
    // byte_decoder = {v: k for k, v in byte_encoder.items()}
    std::map<char, int> byte_decoder;

    for (const auto& entry : byte_to_unicode) {
        byte_decoder[entry.second] = entry.first;
    }

    byte_to_unicode.clear();

    return byte_decoder;
}

// 检查文件是否为 ZIP 文件
bool is_zip_file(const std::string& file_path) {
    struct zip_t* zip = zip_open(file_path.c_str(), 0, 'r');
    if (zip == NULL) {
        return false;
    }
    zip_close(zip);
    return true;
}

// 检查文件是否为 GGUF 文件
bool is_gguf_file(const std::string& file_path) {
    std::ifstream file(file_path, std::ios::binary);
    if (!file.is_open()) {
        return false;
    }

    char magic[4];

    file.read(magic, sizeof(magic));
    if (!file) {
        return false;
    }
    for (uint32_t i = 0; i < sizeof(magic); i++) {
        if (magic[i] != GGUF_MAGIC[i]) {
            return false;
        }
    }

    return true;
}

// 检查文件是否为 Safetensors 文件
bool is_safetensors_file(const std::string& file_path) {
    // 打开文件流，以二进制模式读取文件内容
    std::ifstream file(file_path, std::ios::binary);
    // 如果文件打开失败，则返回 false
    if (!file.is_open()) {
        return false;
    }

    // 获取文件大小
    file.seekg(0, file.end);
    size_t file_size_ = file.tellg();
    file.seekg(0, file.beg);

    // 读取头部大小
    if (file_size_ <= ST_HEADER_SIZE_LEN) {
        return false;
    }

    // 读取头部大小到缓冲区
    uint8_t header_size_buf[ST_HEADER_SIZE_LEN];
    file.read((char*)header_size_buf, ST_HEADER_SIZE_LEN);
    // 如果读取失败，则返回 false
    if (!file) {
        return false;
    }

    // 将头部大小转换为无符号整数
    size_t header_size_ = read_u64(header_size_buf);
    // 如果头部大小超过文件大小或者小于等于2，则返回 false
    if (header_size_ >= file_size_ || header_size_ <= 2) {
        return false;
    }

    // 读取头部内容到缓冲区
    std::vector<char> header_buf;
    header_buf.resize(header_size_ + 1);
    header_buf[header_size_] = '\0';
    file.read(header_buf.data(), header_size_);
    // 如果读取失败，则返回 false
    if (!file) {
        return false;
    }
    
    // 解析头部内容为 JSON 对象
    nlohmann::json header_ = nlohmann::json::parse(header_buf.data());
    // 如果解析失败，则返回 false
    if (header_.is_discarded()) {
        return false;
    }
    // 返回 true 表示成功
    return true;
}

// 从文件初始化模型加载器
bool ModelLoader::init_from_file(const std::string& file_path, const std::string& prefix) {
    // 如果文件路径是一个目录
    if (is_directory(file_path)) {
        // 使用 diffusers 格式加载文件
        LOG_INFO("load %s using diffusers format", file_path.c_str());
        return init_from_diffusers_file(file_path, prefix);
    } 
    // 如果文件是 gguf 格式
    else if (is_gguf_file(file_path)) {
        // 使用 gguf 格式加载文件
        LOG_INFO("load %s using gguf format", file_path.c_str());
        return init_from_gguf_file(file_path, prefix);
    } 
    // 如果文件是 safetensors 格式
    else if (is_safetensors_file(file_path)) {
        // 使用 safetensors 格式加载文件
        LOG_INFO("load %s using safetensors format", file_path.c_str());
        return init_from_safetensors_file(file_path, prefix);
    } 
    // 如果文件是 zip 格式
    else if (is_zip_file(file_path)) {
        // 使用 checkpoint 格式加载文件
        LOG_INFO("load %s using checkpoint format", file_path.c_str());
        return init_from_ckpt_file(file_path, prefix);
    } 
    // 如果文件格式未知
    else {
        // 输出警告信息
        LOG_WARN("unknown format %s", file_path.c_str());
        return false;
    }
}

/*================================================= GGUFModelLoader ==================================================*/

// 从 gguf 文件初始化模型加载器
bool ModelLoader::init_from_gguf_file(const std::string& file_path, const std::string& prefix) {
    // 输出调试信息
    LOG_DEBUG("init from '%s'", file_path.c_str());
    // 将文件路径添加到文件路径列表中
    file_paths_.push_back(file_path);
    // 获取文件路径在列表中的索引
    size_t file_index = file_paths_.size() - 1;

    // 初始化 gguf 上下文和 meta 上下文
    gguf_context* ctx_gguf_ = NULL;
    ggml_context* ctx_meta_ = NULL;
    // 从文件中初始化 gguf 上下文
    ctx_gguf_ = gguf_init_from_file(file_path.c_str(), {true, &ctx_meta_});
    // 如果初始化失败
    if (!ctx_gguf_) {
        // 输出错误信息
        LOG_ERROR("failed to open '%s'", file_path.c_str());
        return false;
    }

    // 获取 gguf 上下文中张量的数量
    int n_tensors = gguf_get_n_tensors(ctx_gguf_);

    // 初始化总大小和数据偏移量
    size_t total_size = 0;
    size_t data_offset = gguf_get_data_offset(ctx_gguf_);
    // 遍历所有张量，获取张量名称
    for (int i = 0; i < n_tensors; i++) {
        // 获取张量名称
        std::string name = gguf_get_tensor_name(ctx_gguf_, i);
        // 获取张量的元数据
        struct ggml_tensor* dummy = ggml_get_tensor(ctx_meta_, name.c_str());
        // 计算数据在文件中的偏移量
        size_t offset = data_offset + gguf_get_tensor_offset(ctx_gguf_, i);

        // 打印张量名称（注释掉的代码）
        // LOG_DEBUG("%s", name.c_str());

        // 创建张量存储对象
        TensorStorage tensor_storage(prefix + name, dummy->type, dummy->ne, ggml_n_dims(dummy), file_index, offset);

        // 检查张量的字节数是否与存储对象的字节数相等
        GGML_ASSERT(ggml_nbytes(dummy) == tensor_storage.nbytes());

        // 将张量存储对象添加到存储对象列表中
        tensor_storages.push_back(tensor_storage);
    }

    // 释放上下文资源
    gguf_free(ctx_gguf_);
    ggml_free(ctx_meta_);

    // 返回操作成功
    return true;
// 定义一个枚举类型，将字符串类型的数据类型转换为对应的枚举值
ggml_type str_to_ggml_type(const std::string& dtype) {
    ggml_type ttype = GGML_TYPE_COUNT;
    if (dtype == "F16") {
        ttype = GGML_TYPE_F16;
    } else if (dtype == "BF16") {
        ttype = GGML_TYPE_F32;
    } else if (dtype == "F32") {
        ttype = GGML_TYPE_F32;
    }
    return ttype;
}

// 从 SafeTensors 文件初始化模型加载器
bool ModelLoader::init_from_safetensors_file(const std::string& file_path, const std::string& prefix) {
    // 记录日志，初始化模型加载器
    LOG_DEBUG("init from '%s'", file_path.c_str());
    // 将文件路径添加到文件路径列表中
    file_paths_.push_back(file_path);
    // 获取文件路径在列表中的索引
    size_t file_index = file_paths_.size() - 1;
    // 以二进制方式打开文件
    std::ifstream file(file_path, std::ios::binary);
    // 如果文件打开失败，则记录错误日志并返回 false
    if (!file.is_open()) {
        LOG_ERROR("failed to open '%s'", file_path.c_str());
        return false;
    }

    // 获取文件大小
    file.seekg(0, file.end);
    size_t file_size_ = file.tellg();
    file.seekg(0, file.beg);

    // 读取头部大小
    if (file_size_ <= ST_HEADER_SIZE_LEN) {
        LOG_ERROR("invalid safetensor file '%s'", file_path.c_str());
        return false;
    }

    uint8_t header_size_buf[ST_HEADER_SIZE_LEN];
    file.read((char*)header_size_buf, ST_HEADER_SIZE_LEN);
    if (!file) {
        LOG_ERROR("read safetensors header size failed: '%s'", file_path.c_str());
        return false;
    }

    size_t header_size_ = read_u64(header_size_buf);
    if (header_size_ >= file_size_) {
        LOG_ERROR("invalid safetensor file '%s'", file_path.c_str());
        return false;
    }

    // 读取头部信息
    std::vector<char> header_buf;
    header_buf.resize(header_size_ + 1);
    header_buf[header_size_] = '\0';
    file.read(header_buf.data(), header_size_);
    if (!file) {
        LOG_ERROR("read safetensors header failed: '%s'", file_path.c_str());
        return false;
    }

    // 解析头部信息为 JSON 格式
    nlohmann::json header_ = nlohmann::json::parse(header_buf.data());
    // 遍历 header_ 中的每个键值对
    for (auto& item : header_.items()) {
        // 获取键值对中的键名
        std::string name = item.key();
        // 获取键值对中的值，解析为 JSON 格式的 tensor_info
        nlohmann::json tensor_info = item.value();
        // 如果需要输出调试信息，可以使用 LOG_DEBUG 函数
        // LOG_DEBUG("%s %s\n", name.c_str(), tensor_info.dump().c_str());

        // 如果键名为 "__metadata__"，则跳过当前循环
        if (name == "__metadata__") {
            continue;
        }

        // 如果当前 tensor 是未使用的，则跳过当前循环
        if (is_unused_tensor(name)) {
            continue;
        }

        // 获取 tensor_info 中的数据类型 dtype 和形状 shape
        std::string dtype = tensor_info["dtype"];
        nlohmann::json shape = tensor_info["shape"];

        // 获取数据在文件中的起始和结束位置
        size_t begin = tensor_info["data_offsets"][0].get<size_t>();
        size_t end = tensor_info["data_offsets"][1].get<size_t>();

        // 将数据类型 dtype 转换为 ggml_type 类型
        ggml_type type = str_to_ggml_type(dtype);
        // 如果数据类型不支持，则输出错误信息并返回 false
        if (type == GGML_TYPE_COUNT) {
            LOG_ERROR("unsupported dtype '%s'", dtype.c_str());
            return false;
        }

        // 如果数据形状维度大于 4，则输出错误信息并返回 false
        if (shape.size() > 4) {
            LOG_ERROR("invalid tensor '%s'", name.c_str());
            return false;
        }

        // 获取数据形状的维度数和各维度大小
        int n_dims = (int)shape.size();
        int64_t ne[4] = {1, 1, 1, 1};
        for (int i = 0; i < n_dims; i++) {
            ne[i] = shape[i].get<int64_t>();
        }

        // 创建 TensorStorage 对象，设置名称、类型、形状、文件索引和数据起始位置等信息
        TensorStorage tensor_storage(prefix + name, type, ne, n_dims, file_index, ST_HEADER_SIZE_LEN + header_size_ + begin);

        // 反转形状大小数组
        tensor_storage.reverse_ne();

        // 计算 tensor 数据大小
        size_t tensor_data_size = end - begin;

        // 如果数据类型为 "BF16"，则设置 is_bf16 为 true，并校验数据大小是否正确
        if (dtype == "BF16") {
            tensor_storage.is_bf16 = true;
            GGML_ASSERT(tensor_storage.nbytes() == tensor_data_size * 2);
        } else {
            // 校验数据大小是否正确
            GGML_ASSERT(tensor_storage.nbytes() == tensor_data_size);
        }

        // 将当前 TensorStorage 对象添加到 tensor_storages 中
        tensor_storages.push_back(tensor_storage);
    }

    // 返回 true 表示处理成功
    return true;
// 结束 ModelLoader 类定义
}

/*================================================= DiffusersModelLoader ==================================================*/

// 从给定文件路径和前缀初始化 DiffusersModelLoader
bool ModelLoader::init_from_diffusers_file(const std::string& file_path, const std::string& prefix) {
    // 构建 unet_path、vae_path 和 clip_path
    std::string unet_path = path_join(file_path, "unet/diffusion_pytorch_model.safetensors");
    std::string vae_path  = path_join(file_path, "vae/diffusion_pytorch_model.safetensors");
    std::string clip_path = path_join(file_path, "text_encoder/model.safetensors");

    // 如果无法从 unet_path 初始化，则返回 false
    if (!init_from_safetensors_file(unet_path, "unet.")) {
        return false;
    }
    // 如果无法从 vae_path 初始化，则返回 false
    if (!init_from_safetensors_file(vae_path, "vae.")) {
        return false;
    }
    // 如果无法从 clip_path 初始化，则返回 false
    if (!init_from_safetensors_file(clip_path, "te.")) {
        return false;
    }
    // 初始化成功，返回 true
    return true;
}

/*================================================= CkptModelLoader ==================================================*/

// 使用 pickletools 查看数据文件的前 100 行内容
// 用法示例：$ python -m pickletools sd-v1-4/archive/data.pkl | head -n 100
// 以下是示例输出的部分内容
// ...
//   171: c                    GLOBAL     'torch FloatStorage'
//   191: q                    BINPUT     10
//   193: X                    BINUNICODE '0'
//   199: q                    BINPUT     11
//   201: X                    BINUNICODE 'cpu'
//   209: q                    BINPUT     12
//   211: M                    BININT2    1000
//   214: t                    TUPLE      (MARK at 156)
//   215: q                BINPUT     13
//   217: Q                BINPERSID
//   218: K                BININT1    0
//   220: M                BININT2    1000
//  ...............................
//  3201: q            BINPUT     250
//  3203: R            REDUCE
//  3204: q            BINPUT     251
//  3206: X            BINUNICODE 'model.diffusion_model.input_blocks.1.1.proj_in.weight'
//  3264: q            BINPUT     252
//  3266: h            BINGET     8
//  3268: (            MARK
//  3269: (                MARK
//  3270: h                    BINGET     9
//  3272: h                    BINGET     10
//  3274: X                    BINUNICODE '30'
//  3281: q                    BINPUT     253
//  3283: h                    BINGET     12
//  3285: J                    BININT     102400
//  3290: t                    TUPLE      (MARK at 3269)
//  3291: q                BINPUT     254
//  3293: Q                BINPERSID
//  3294: K                BININT1    0
//  3296: (                MARK
//  3297: M                    BININT2    320
//  3300: M                    BININT2    320
//  3303: K                    BININT1    1
//  3305: K                    BININT1    1
//  3307: t                    TUPLE      (MARK at 3296)
//  3308: q                BINPUT     255
//  3310: (                MARK
//  3311: M                    BININT2    320
//  3314: K                    BININT1    1
//  3316: K                    BININT1    1
//  3318: K                    BININT1    1
//  3320: t                    TUPLE      (MARK at 3310)
//  3321: r                LONG_BINPUT 256
// 定义一个结构体 PickleTensorReader，用于读取 pickle 格式的张量数据
struct PickleTensorReader {
    // 定义枚举类型 ReadPhase，表示读取的不同阶段
    enum ReadPhase {
        READ_NAME,   // 读取名称阶段
        READ_DATA,   // 读取数据阶段
        CHECK_SIZE,  // 检查大小阶段
        READ_DIMENS  // 读取维度阶段
    };
    // 初始化读取阶段为 READ_NAME
    ReadPhase phase   = READ_NAME;
    // 初始化条目大小为 0
    size_t entry_size = 0;
    // 初始化元素数量为 0
    int32_t nelements = 0;

    // 定义张量存储结构
    TensorStorage tensor_storage;

    // 定义全局类型 ggml_type，表示所有 pickle_tensors 的数据类型
    static ggml_type global_type;
    // 是否已读取全局类型的标志
    static bool read_global_type;

    // 读取整数值的函数
    bool read_int_value(uint32_t value) {
        // 如果当前阶段为 CHECK_SIZE
        if (phase == CHECK_SIZE) {
            // 检查条目大小是否等于 value 乘以数据类型的大小
            if (entry_size == value * ggml_type_size(tensor_storage.type)) {
                // 更新元素数量和阶段为 READ_DIMENS
                nelements = value;
                phase     = READ_DIMENS;
                return true;
            } else {
                // 否则将阶段设置为 READ_NAME
                phase = READ_NAME;
            }
        } 
        // 如果当前阶段为 READ_DIMENS
        else if (phase == READ_DIMENS) {
            // 如果维度数量超过 4，则将阶段设置为 READ_NAME 并重置维度数量
            if (tensor_storage.n_dims + 1 > 4) {  // too many dimens
                phase                 = READ_NAME;
                tensor_storage.n_dims = 0;
            }
            // 如果元素数量能够整除 value，则更新维度信息
            if (nelements % value == 0) {
                tensor_storage.ne[tensor_storage.n_dims] = value;
                tensor_storage.n_dims++;
            }
        }
        return false;
    }
}
    // 读取全局变量，根据字符串类型设置全局类型和张量存储类型
    void read_global(const std::string& str) {
        // 如果字符串为"FloatStorage"
        if (str == "FloatStorage") {
            // 如果需要读取全局类型
            if (read_global_type) {
                // 设置全局类型为 GGML_TYPE_F32
                global_type      = GGML_TYPE_F32;
                read_global_type = false;
            }
            // 设置张量存储类型为 GGML_TYPE_F32
            tensor_storage.type = GGML_TYPE_F32;
        } 
        // 如果字符串为"HalfStorage"
        else if (str == "HalfStorage") {
            // 如果需要读取全局类型
            if (read_global_type) {
                // 设置全局类型为 GGML_TYPE_F16
                global_type      = GGML_TYPE_F16;
                read_global_type = false;
            }
            // 设置张量存储类型为 GGML_TYPE_F16
            tensor_storage.type = GGML_TYPE_F16;
        }
    }

    // 读取字符串，根据字符串内容设置全局类型、张量存储名称和类型
    void read_string(const std::string& str, struct zip_t* zip, std::string dir) {
        // 如果字符串为"storage"
        if (str == "storage") {
            // 设置需要读取全局类型为 true
            read_global_type = true;
        } 
        // 如果字符串不为"state_dict"
        else if (str != "state_dict") {
            // 如果处于读取数据阶段
            if (phase == READ_DATA) {
                // 构建 ZIP 文件中数据的路径
                std::string entry_name = dir + "data/" + std::string(str);

                size_t i, n = zip_entries_total(zip);
                // 遍历 ZIP 文件中的所有条目
                for (i = 0; i < n; ++i) {
                    zip_entry_openbyindex(zip, i);
                    {
                        // 获取当前 ZIP 条目的名称
                        std::string name = zip_entry_name(zip);
                        // 如果名称与目标路径相符
                        if (name == entry_name) {
                            // 设置张量存储在 ZIP 文件中的索引和大小
                            tensor_storage.index_in_zip = (int)i;
                            entry_size                  = zip_entry_size(zip);
                            zip_entry_close(zip);
                            break;
                        }
                    }
                    zip_entry_close(zip);
                }

                // 根据条目大小判断下一步操作
                phase = entry_size > 0 ? CHECK_SIZE : READ_NAME;
            }
            // 如果未读取全局类型且处于读取名称阶段
            if (!read_global_type && phase == READ_NAME) {
                // 设置张量存储的名称和类型
                tensor_storage.name = str;
                phase               = READ_DATA;
                tensor_storage.type = global_type;
            }
        }
    }
// 定义 PickleTensorReader 类的静态成员变量 global_type，表示所有 pickle_tensors 的数据类型，默认为 GGML_TYPE_F32
ggml_type PickleTensorReader::global_type = GGML_TYPE_F32;
// 定义 PickleTensorReader 类的静态成员变量 read_global_type，表示是否已经读取了全局数据类型，默认为 false
bool PickleTensorReader::read_global_type = false;

// 在 buffer 中查找字符 c，返回第一个匹配字符的位置，如果找不到返回 -1
int find_char(uint8_t* buffer, int len, char c) {
    for (int pos = 0; pos < len; pos++) {
        if (buffer[pos] == c) {
            return pos;
        }
    }
    return -1;
}

// 定义最大字符串缓冲区大小为 512
#define MAX_STRING_BUFFER 512

// 从 buffer 中解析数据，存储在 zip 中，dir 表示目录，file_index 表示文件索引，prefix 表示前缀
bool ModelLoader::parse_data_pkl(uint8_t* buffer,
                                 size_t buffer_size,
                                 zip_t* zip,
                                 std::string dir,
                                 size_t file_index,
                                 const std::string& prefix) {
    // 计算 buffer 的结束位置
    uint8_t* buffer_end = buffer + buffer_size;
    // 返回 true 表示解析成功
    return true;
}

// 从 ckpt 文件中初始化模型加载器，file_path 表示文件路径，prefix 表示前缀
bool ModelLoader::init_from_ckpt_file(const std::string& file_path, const std::string& prefix) {
    // 输出调试信息
    LOG_DEBUG("init from '%s'", file_path.c_str());
    // 将文件路径添加到 file_paths_ 中
    file_paths_.push_back(file_path);
    // 获取文件索引
    size_t file_index = file_paths_.size() - 1;

    // 打开 ZIP 文件
    struct zip_t* zip = zip_open(file_path.c_str(), 0, 'r');
    // 如果打开失败，输出错误信息并返回 false
    if (zip == NULL) {
        LOG_ERROR("failed to open '%s'", file_path.c_str());
        return false;
    }
    // 获取 ZIP 文件中的条目数量
    int n = (int)zip_entries_total(zip);
    // 遍历 ZIP 文件中的每个条目
    for (int i = 0; i < n; ++i) {
        // 打开 ZIP 文件中的第 i 个条目
        zip_entry_openbyindex(zip, i);
        {
            // 获取条目的名称
            std::string name = zip_entry_name(zip);
            // 查找是否包含 "data.pkl" 字符串
            size_t pos       = name.find("data.pkl");
            // 如果找到了
            if (pos != std::string::npos) {
                // 获取目录名
                std::string dir = name.substr(0, pos);
                void* pkl_data  = NULL;
                size_t pkl_size;
                // 读取条目数据
                zip_entry_read(zip, &pkl_data, &pkl_size);

                // 解析数据
                parse_data_pkl((uint8_t*)pkl_data, pkl_size, zip, dir, file_index, prefix);

                // 释放数据内存
                free(pkl_data);
            }
        }
        // 关闭 ZIP 条目
        zip_entry_close(zip);
    }
    // 关闭 ZIP 文件
    zip_close(zip);
    // 返回 true 表示初始化成功
    return true;
}

// 获取 SD 版本
SDVersion ModelLoader::get_sd_version() {
    // 返回版本号 VERSION_1_x
    // return VERSION_1_x;
}
    // 定义一个 TensorStorage 类型的变量 token_embedding_weight
    TensorStorage token_embedding_weight;
    // 遍历 tensor_storages 中的每一个 tensor_storage
    for (auto& tensor_storage : tensor_storages) {
        // 如果 tensor_storage 的名称包含 "conditioner.embedders.1"，返回 VERSION_XL
        if (tensor_storage.name.find("conditioner.embedders.1") != std::string::npos) {
            return VERSION_XL;
        }
        // 如果 tensor_storage 的名称包含 "cond_stage_model.1"，返回 VERSION_XL
        if (tensor_storage.name.find("cond_stage_model.1") != std::string::npos) {
            return VERSION_XL;
        }
        // 如果 tensor_storage 的名称符合以下条件之一，将其赋值给 token_embedding_weight
        if (tensor_storage.name == "cond_stage_model.transformer.text_model.embeddings.token_embedding.weight" ||
            tensor_storage.name == "cond_stage_model.model.token_embedding.weight" ||
            tensor_storage.name == "text_model.embeddings.token_embedding.weight" ||
            tensor_storage.name == "te.text_model.embeddings.token_embedding.weight" ||
            tensor_storage.name == "conditioner.embedders.0.model.token_embedding.weight" ||
            tensor_storage.name == "conditioner.embedders.0.transformer.text_model.embeddings.token_embedding.weight") {
            token_embedding_weight = tensor_storage;
            // break;  // 注释掉的 break 语句，表示找到符合条件的 tensor_storage 后不再继续遍历
        }
    }
    // 如果 token_embedding_weight 的第一个元素等于 768，返回 VERSION_1_x
    if (token_embedding_weight.ne[0] == 768) {
        return VERSION_1_x;
    } else if (token_embedding_weight.ne[0] == 1024) {
        // 如果 token_embedding_weight 的第一个元素等于 1024，返回 VERSION_2_x
        return VERSION_2_x;
    }
    // 其他情况返回 VERSION_COUNT
    return VERSION_COUNT;
}

// 获取模型加载器的 sd_wtype 属性
ggml_type ModelLoader::get_sd_wtype() {
    // 遍历张量存储列表
    for (auto& tensor_storage : tensor_storages) {
        // 如果张量未被使用，则继续下一个循环
        if (is_unused_tensor(tensor_storage.name)) {
            continue;
        }

        // 如果张量名称包含 ".weight" 和 "time_embed" 字符串，则返回该张量的类型
        if (tensor_storage.name.find(".weight") != std::string::npos &&
            tensor_storage.name.find("time_embed") != std::string::npos) {
            return tensor_storage.type;
        }
    }
    // 如果未找到符合条件的张量，则返回 GGML_TYPE_COUNT
    return GGML_TYPE_COUNT;
}

// 加载 merges
std::string ModelLoader::load_merges() {
    // 将 merges_utf8_c_str 转换为 UTF-8 字符串
    std::string merges_utf8_str(reinterpret_cast<const char*>(merges_utf8_c_str), sizeof(merges_utf8_c_str));
    return merges_utf8_str;
}

// 移除重复的张量存储
void remove_duplicates(std::vector<TensorStorage>& vec) {
    // 使用哈希表存储张量名称和索引的映射关系
    std::unordered_map<std::string, size_t> name_to_index_map;

    // 遍历张量存储列表
    for (size_t i = 0; i < vec.size(); ++i) {
        const std::string& current_name = vec[i].name;
        auto it = name_to_index_map.find(current_name);

        // 如果当前张量名称已存在于哈希表中，则替换原有张量存储
        if (it != name_to_index_map.end()) {
            vec[it->second] = vec[i];
        } else {
            // 否则将当前张量名称和索引存入哈希表
            name_to_index_map[current_name] = i;
        }
    }

    // 调整张量存储列表的大小，去除重复的张量存储
    vec.resize(name_to_index_map.size());
}

// 加载张量
bool ModelLoader::load_tensors(on_new_tensor_cb_t on_new_tensor_cb, ggml_backend_t backend) {
    std::vector<TensorStorage> processed_tensor_storages;
    // 遍历张量存储列表
    for (auto& tensor_storage : tensor_storages) {
        // LOG_DEBUG("%s", name.c_str());

        // 如果张量未被使用，则继续下一个循环
        if (is_unused_tensor(tensor_storage.name)) {
            continue;
        }

        // 预处理张量存储
        preprocess_tensor(tensor_storage, processed_tensor_storages);
    }
    // 移除重复的张量存储
    remove_duplicates(processed_tensor_storages);
    bool success = true;
    }
    return success;
}

// 加载张量
bool ModelLoader::load_tensors(std::map<std::string, struct ggml_tensor*>& tensors,
                               ggml_backend_t backend,
                               std::set<std::string> ignore_tensors) {
    // 存储文件中的张量名称
    std::set<std::string> tensor_names_in_file;
    // 定义一个 lambda 函数 on_new_tensor_cb，用于处理新的张量数据
    auto on_new_tensor_cb = [&](const TensorStorage& tensor_storage, ggml_tensor** dst_tensor) -> bool {
        // 获取张量的名称
        const std::string& name = tensor_storage.name;
        // 将张量名称添加到文件中的张量名称集合中
        tensor_names_in_file.insert(name);

        // 声明一个指向 ggml_tensor 结构体的指针 real
        struct ggml_tensor* real;
        // 如果张量在 tensors 中已存在，则将其赋值给 real
        if (tensors.find(name) != tensors.end()) {
            real = tensors[name];
        } else {
            // 如果张量不在 tensors 中，且不在 ignore_tensors 中，则输出警告信息
            if (ignore_tensors.find(name) == ignore_tensors.end()) {
                LOG_WARN("unknown tensor '%s' in model file", name.c_str());
            }
            return true;
        }

        // 检查张量的形状是否与预期形状一致，若不一致则输出错误信息
        if (
            real->ne[0] != tensor_storage.ne[0] ||
            real->ne[1] != tensor_storage.ne[1] ||
            real->ne[2] != tensor_storage.ne[2] ||
            real->ne[3] != tensor_storage.ne[3]) {
            LOG_ERROR(
                "tensor '%s' has wrong shape in model file: "
                "got [%d, %d, %d, %d], expected [%d, %d, %d, %d]",
                name.c_str(),
                (int)tensor_storage.ne[0], (int)tensor_storage.ne[1], (int)tensor_storage.ne[2], (int)tensor_storage.ne[3],
                (int)real->ne[0], (int)real->ne[1], (int)real->ne[2], (int)real->ne[3]);
            return false;
        }

        // 将 real 赋值给目标张量指针 dst_tensor
        *dst_tensor = real;

        return true;
    };

    // 载入张量数据，使用 on_new_tensor_cb 处理新的张量数据，使用 backend
    bool success = load_tensors(on_new_tensor_cb, backend);
    // 若载入张量数据失败，则输出错误信息并返回 false
    if (!success) {
        LOG_ERROR("load tensors from file failed");
        return false;
    }

    // 初始化一个标志，用于标记是否有张量未初始化
    bool some_tensor_not_init = false;
    // 遍历tensors中的每个pair
    for (auto pair : tensors) {
        // 如果pair的first包含特定字符串，则跳过本次循环
        if (pair.first.find("cond_stage_model.transformer.text_model.encoder.layers.23") != std::string::npos) {
            continue;
        }
        // 如果pair的first包含特定字符串，则跳过本次循环
        if (pair.first.find("alphas_cumprod") != std::string::npos) {
            continue;
        }

        // 如果pair的first包含特定字符串，则跳过本次循环
        if (pair.first.find("alphas_cumprod") != std::string::npos) {
            continue;
        }

        // 如果tensor_names_in_file中不包含pair的first，则输出错误信息并设置some_tensor_not_init为true
        if (tensor_names_in_file.find(pair.first) == tensor_names_in_file.end()) {
            LOG_ERROR("tensor '%s' not in model file", pair.first.c_str());
            some_tensor_not_init = true;
        }
    }

    // 如果有未初始化的张量，则返回false
    if (some_tensor_not_init) {
        return false;
    }
    // 所有张量都已初始化，返回true
    return true;
}

// 将模型数据保存到 GGUF 文件
bool ModelLoader::save_to_gguf_file(const std::string& file_path, ggml_type type) {
    // 初始化 CPU 后端
    auto backend    = ggml_backend_cpu_init();
    // 内存大小初始化为 1MB，用于填充
    size_t mem_size = 1 * 1024 * 1024;  // for padding
    // 计算存储张量所需的内存大小
    mem_size += tensor_storages.size() * ggml_tensor_overhead();
    mem_size += cal_mem_size(backend, type);
    // 打印模型张量内存大小信息
    LOG_INFO("model tensors mem size: %.2fMB", mem_size / 1024.f / 1024.f);
    // 初始化 GGML 上下文
    ggml_context* ggml_ctx = ggml_init({mem_size, NULL, false});

    // 初始化空的 GGUF 上下文
    gguf_context* gguf_ctx = gguf_init_empty();

    // 定义新张量回调函数
    auto on_new_tensor_cb = [&](const TensorStorage& tensor_storage, ggml_tensor** dst_tensor) -> bool {
        const std::string& name = tensor_storage.name;

        ggml_type tensor_type = tensor_storage.type;
        // 根据类型设置张量类型
        if (type != GGML_TYPE_COUNT) {
            if (ggml_is_quantized(type) && tensor_storage.ne[0] % 32 != 0) {
                tensor_type = GGML_TYPE_F16;
            } else {
                tensor_type = type;
            }
        }

        // 创建新张量
        ggml_tensor* tensor = ggml_new_tensor(ggml_ctx, tensor_type, tensor_storage.n_dims, tensor_storage.ne);
        if (tensor == NULL) {
            LOG_ERROR("ggml_new_tensor failed");
            return false;
        }
        ggml_set_name(tensor, name.c_str());

        // 将新张量添加到 GGUF 上下文中
        *dst_tensor = tensor;
        gguf_add_tensor(gguf_ctx, tensor);

        return true;
    };

    // 加载张量数据
    bool success = load_tensors(on_new_tensor_cb, backend);
    // 释放后端资源
    ggml_backend_free(backend);
    LOG_INFO("load tensors done");
    LOG_INFO("trying to save tensors to %s", file_path.c_str());
    // 如果加载成功，则将数据保存到文件中
    if (success) {
        gguf_write_to_file(gguf_ctx, file_path.c_str(), false);
    }
    // 释放 GGML 上下文
    ggml_free(ggml_ctx);
    释放 gguf_ctx 占用的内存空间
    gguf_free(gguf_ctx);
    返回 success 变量的值作为函数的返回值
    return success;
}

// 计算模型加载器的内存大小
int64_t ModelLoader::cal_mem_size(ggml_backend_t backend, ggml_type type) {
    // 设置对齐值为128
    size_t alignment = 128;
    // 如果后端不为空，获取后端的对齐值
    if (backend != NULL) {
        alignment = ggml_backend_get_alignment(backend);
    }
    // 初始化内存大小为0
    int64_t mem_size = 0;
    // 存储处理过的张量存储
    std::vector<TensorStorage> processed_tensor_storages;
    // 遍历张量存储
    for (auto& tensor_storage : tensor_storages) {
        // 如果张量未使用，则跳过
        if (is_unused_tensor(tensor_storage.name)) {
            continue;
        }
        // 预处理张量存储
        preprocess_tensor(tensor_storage, processed_tensor_storages);
    }

    // 遍历处理过的张量存储
    for (auto& tensor_storage : processed_tensor_storages) {
        // 获取张量类型
        ggml_type tensor_type = tensor_storage.type;
        // 如果指定了类型
        if (type != GGML_TYPE_COUNT) {
            // 如果是量化类型且张量大小不是32的倍数，则设置类型为GGML_TYPE_F16，否则设置为指定类型
            if (ggml_is_quantized(type) && tensor_storage.ne[0] % 32 != 0) {
                tensor_type = GGML_TYPE_F16;
            } else {
                tensor_type = type;
            }
        }
        // 更新张量类型
        tensor_storage.type = tensor_type;
        // 计算内存大小并加上对齐值
        mem_size += tensor_storage.nbytes() + alignment;
    }

    return mem_size;
}

// 转换函数
bool convert(const char* input_path, const char* vae_path, const char* output_path, sd_type_t output_type) {
    // 创建模型加载器
    ModelLoader model_loader;

    // 从文件初始化模型加载器
    if (!model_loader.init_from_file(input_path)) {
        LOG_ERROR("init model loader from file failed: '%s'", input_path);
        return false;
    }

    // 如果VAE路径不为空，从文件初始化模型加载器
    if (vae_path != NULL && strlen(vae_path) > 0) {
        if (!model_loader.init_from_file(vae_path, "vae.")) {
            LOG_ERROR("init model loader from file failed: '%s'", vae_path);
            return false;
        }
    }
    // 将模型保存为GGUF文件
    bool success = model_loader.save_to_gguf_file(output_path, (ggml_type)output_type);
    return success;
}
```