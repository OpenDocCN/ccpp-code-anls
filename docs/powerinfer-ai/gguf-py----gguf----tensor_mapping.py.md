# `PowerInfer\gguf-py\gguf\tensor_mapping.py`

```
# 从 __future__ 模块中导入 annotations 特性
from __future__ import annotations

# 从 typing 模块中导入 Sequence 类型
from typing import Sequence

# 从当前目录下的 constants 模块中导入 MODEL_ARCH, MODEL_TENSOR, MODEL_TENSORS, TENSOR_NAMES 常量
from .constants import MODEL_ARCH, MODEL_TENSOR, MODEL_TENSORS, TENSOR_NAMES

# 定义一个名为 TensorNameMap 的类
class TensorNameMap:
    # 定义一个名为 mappings_cfg 的类属性，其类型为字典，键为 MODEL_TENSOR 枚举类型，值为元组类型
    mappings_cfg: dict[MODEL_TENSOR, tuple[str, ...]] = {
        # Token embeddings
        MODEL_TENSOR.TOKEN_EMBD: (
            "gpt_neox.embed_in",                         # gptneox
            "transformer.wte",                           # gpt2 gpt-j mpt refact
            "transformer.word_embeddings",               # falcon
            "word_embeddings",                           # bloom
            "model.embed_tokens",                        # llama-hf
            "tok_embeddings",                            # llama-pth
            "embeddings.word_embeddings",                # bert
            "language_model.embedding.word_embeddings",  # persimmon
        ),
# Token type embeddings
# 令牌类型嵌入

MODEL_TENSOR.TOKEN_TYPES: (
    "embeddings.token_type_embeddings",  # bert
),

# Normalization of token embeddings
# 令牌嵌入的归一化
MODEL_TENSOR.TOKEN_EMBD_NORM: (
    "word_embeddings_layernorm",  # bloom
),

# Position embeddings
# 位置嵌入
MODEL_TENSOR.POS_EMBD: (
    "transformer.wpe",                 # gpt2
    "embeddings.position_embeddings",  # bert
),

# Output
# 输出
MODEL_TENSOR.OUTPUT: (
    "embed_out",                 # gptneox
```

# 模型头部
MODEL_TENSOR.HEAD: (
    "lm_head",                   # gpt2 mpt falcon llama-hf baichuan
    "output",                    # llama-pth bloom
    "word_embeddings_for_head",  # persimmon
),

# 输出规范化
MODEL_TENSOR.OUTPUT_NORM: (
    "gpt_neox.final_layer_norm",               # gptneox
    "transformer.ln_f",                        # gpt2 gpt-j falcon
    "model.norm",                              # llama-hf baichuan
    "norm",                                    # llama-pth
    "embeddings.LayerNorm",                    # bert
    "transformer.norm_f",                      # mpt
    "ln_f",                                    # refact bloom
    "language_model.encoder.final_layernorm",  # persimmon
),

# 绳索频率
MODEL_TENSOR.ROPE_FREQS: (
    "rope.freqs",  # llama-pth
),
# 定义一个字典，用于存储模型张量和对应的层名称
block_mappings_cfg: dict[MODEL_TENSOR, tuple[str, ...]] = {
    # Attention norm
    MODEL_TENSOR.ATTN_NORM: (
        "gpt_neox.layers.{bid}.input_layernorm",                # gptneox
        "transformer.h.{bid}.ln_1",                             # gpt2 gpt-j refact
        "transformer.blocks.{bid}.norm_1",                      # mpt
        "transformer.h.{bid}.input_layernorm",                  # falcon7b
        "h.{bid}.input_layernorm",                              # bloom
        "transformer.h.{bid}.ln_mlp",                           # falcon40b
        "model.layers.{bid}.input_layernorm",                   # llama-hf
        "layers.{bid}.attention_norm",                          # llama-pth
        "encoder.layer.{bid}.attention.output.LayerNorm",       # bert
        "language_model.encoder.layers.{bid}.input_layernorm",  # persimmon
        "model.layers.{bid}.ln1",                               # yi
    ),
    # Attention norm 2
    # 其他模型张量和层名称的映射
}
# 定义模型张量的注意力规范化层
MODEL_TENSOR.ATTN_NORM_2: (
    "transformer.h.{bid}.ln_attn",  # falcon40b
),

# 定义模型张量的注意力查询-键-值
MODEL_TENSOR.ATTN_QKV: (
    "gpt_neox.layers.{bid}.attention.query_key_value",                     # gptneox
    "transformer.h.{bid}.attn.c_attn",                                     # gpt2
    "transformer.blocks.{bid}.attn.Wqkv",                                  # mpt
    "transformer.h.{bid}.self_attention.query_key_value",                  # falcon
    "h.{bid}.self_attention.query_key_value",                              # bloom
    "language_model.encoder.layers.{bid}.self_attention.query_key_value",  # persimmon
),

# 定义模型张量的注意力查询
MODEL_TENSOR.ATTN_Q: (
    "model.layers.{bid}.self_attn.q_proj",       # llama-hf
    "layers.{bid}.attention.wq",                 # llama-pth
    "encoder.layer.{bid}.attention.self.query",  # bert
    "transformer.h.{bid}.attn.q_proj",           # gpt-j
```
        ),

        # 注意力机制的键
        MODEL_TENSOR.ATTN_K: (
            "model.layers.{bid}.self_attn.k_proj",     # llama-hf
            "layers.{bid}.attention.wk",               # llama-pth
            "encoder.layer.{bid}.attention.self.key",  # bert
            "transformer.h.{bid}.attn.k_proj",         # gpt-j
        ),

        # 注意力机制的值
        MODEL_TENSOR.ATTN_V: (
            "model.layers.{bid}.self_attn.v_proj",       # llama-hf
            "layers.{bid}.attention.wv",                 # llama-pth
            "encoder.layer.{bid}.attention.self.value",  # bert
            "transformer.h.{bid}.attn.v_proj",           # gpt-j
        ),

        # 注意力机制的输出
        MODEL_TENSOR.ATTN_OUT: (
# 下面是一系列字符串，表示不同模型中的特定层的命名规则
# 例如 "gpt_neox.layers.{bid}.attention.dense" 表示 gptneox 模型中的某一层
# 每个字符串都是一个模型中的特定层的命名规则
# 以下是一些示例：
# "transformer.h.{bid}.attn.c_proj" 表示 gpt2 refact 模型中的某一层
# "encoder.layer.{bid}.attention.output.dense" 表示 bert 模型中的某一层
# "language_model.encoder.layers.{bid}.self_attention.dense" 表示 persimmon 模型中的某一层

# 下面是一些表示模型中特定层的旋转嵌入和前馈正则化的命名规则
# 例如 "model.layers.{bid}.self_attn.rotary_emb.inv_freq" 表示 llama-hf 模型中的某一层的旋转嵌入
# "layers.{bid}.attention.inner_attention.rope.freqs" 表示 llama-pth 模型中的某一层的前馈正则化
# 下面是一系列模型层的命名，用于不同的深度学习模型
# 例如 "gpt_neox.layers.{bid}.post_attention_layernorm" 是用于 gptneox 模型的层
# 每个模型都有不同的命名规则和层结构
# 通过这些命名可以方便地在代码中引用和操作特定的模型层
# 编码器层的中间稠密层，用于BERT模型
"encoder.layer.{bid}.intermediate.dense", 

# GPT-J模型的transformer模块中的MLP输入层
"transformer.h.{bid}.mlp.fc_in", 

# Persimmon模型的语言模型编码器层中的稠密层
"language_model.encoder.layers.{bid}.mlp.dense_h_to_4h", 

# 模型的前馈门
MODEL_TENSOR.FFN_GATE: (
    # Llama-HF重构中的模型层的MLP门投影
    "model.layers.{bid}.mlp.gate_proj",  
    # Llama-PTH中的层的前馈网络w1
    "layers.{bid}.feed_forward.w1",      
),

# 前馈网络下层
MODEL_TENSOR.FFN_DOWN: (
    # GPT-Neox模型中的层的稠密层4h到h
    "gpt_neox.layers.{bid}.mlp.dense_4h_to_h",                
    # GPT2重构中的transformer模块的c_proj
    "transformer.h.{bid}.mlp.c_proj",                         
    # MPT模型中的transformer模块的前馈网络下投影
    "transformer.blocks.{bid}.ffn.down_proj",                 
    # Falcon模型中的transformer模块的稠密层4h到h
    "transformer.h.{bid}.mlp.dense_4h_to_h",                  
    # Bloom模型中的h层的稠密层4h到h
    "h.{bid}.mlp.dense_4h_to_h",                              
    # Llama-HF模型中的模型层的前馈网络下投影
    "model.layers.{bid}.mlp.down_proj",                       
    # Llama-PTH中的层的前馈网络w2
    "layers.{bid}.feed_forward.w2",                           
)
# 模型中的输出层密集连接层，用于BERT模型
"encoder.layer.{bid}.output.dense", 

# GPT-J模型中的变压器模块中的多层感知机的输出层
"transformer.h.{bid}.mlp.fc_out", 

# Persimmon模型中的语言模型编码器层中的多层感知机的输出层
"language_model.encoder.layers.{bid}.mlp.dense_4h_to_h", 

# 模型中的注意力机制中的查询向量的归一化层
"language_model.encoder.layers.{bid}.self_attention.q_layernorm", 

# 模型中的注意力机制中的键向量的归一化层
"language_model.encoder.layers.{bid}.self_attention.k_layernorm", 

# Persimmon模型中的注意力机制中的旋转嵌入的频率逆向
"language_model.encoder.layers.{bid}.self_attention.rotary_emb.inv_freq", 

# 模型中的全连接层1
"model.layers.{bid}.fc1",
# 定义一个字典，用于存储模型张量和其对应的键
mapping: dict[str, tuple[MODEL_TENSOR, str]]

# 初始化方法，接受模型架构和块数作为参数
def __init__(self, arch: MODEL_ARCH, n_blocks: int):
    # 初始化一个空的映射字典
    self.mapping = {}
    # 遍历配置文件中的映射关系
    for tensor, keys in self.mappings_cfg.items():
        # 如果张量不在指定模型架构的张量列表中，则跳过
        if tensor not in MODEL_TENSORS[arch]:
            continue
        # 获取张量的名称
        tensor_name = TENSOR_NAMES[tensor]
        # 将张量名称和对应的键存储到映射字典中
        self.mapping[tensor_name] = (tensor, tensor_name)
        # 遍历每个键，并将其与张量名称和对应的键存储到映射字典中
        for key in keys:
            self.mapping[key] = (tensor, tensor_name)
    # 遍历每个块
    for bid in range(n_blocks):
        # 遍历块配置文件中的映射关系
        for tensor, keys in self.block_mappings_cfg.items():
            # 如果张量不在指定模型架构的张量列表中，则跳过
            if tensor not in MODEL_TENSORS[arch]:
# 继续循环
                    continue
                # 根据索引从 TENSOR_NAMES 中获取张量名称
                tensor_name = TENSOR_NAMES[tensor].format(bid = bid)
                # 将张量名称和张量的映射关系存入 mapping 字典中
                self.mapping[tensor_name] = (tensor, tensor_name)
                # 遍历 keys 列表，根据索引从 TENSOR_NAMES 中获取张量名称，并存入 mapping 字典中
                for key in keys:
                    key = key.format(bid = bid)
                    self.mapping[key] = (tensor, tensor_name)

    # 根据给定的 key 和尝试的后缀获取类型和名称的方法
    def get_type_and_name(self, key: str, try_suffixes: Sequence[str] = ()) -> tuple[MODEL_TENSOR, str] | None:
        # 在 mapping 字典中查找给定 key 对应的值
        result = self.mapping.get(key)
        # 如果找到了对应的值，则返回结果
        if result is not None:
            return result
        # 否则，尝试使用后缀来查找对应的值
        for suffix in try_suffixes:
            if key.endswith(suffix):
                result = self.mapping.get(key[:-len(suffix)])
                if result is not None:
                    return result[0], result[1] + suffix
        # 如果都找不到，则返回 None
        return None

    # 根据给定的 key 和尝试的后缀获取名称的方法
    def get_name(self, key: str, try_suffixes: Sequence[str] = ()) -> str | None:
        # 调用 get_type_and_name 方法获取类型和名称，然后返回名称
        result = self.get_type_and_name(key, try_suffixes = try_suffixes)
        # 如果结果为 None，则返回 None
        if result is None:
            return None
        # 返回结果的第二个元素
        return result[1]

    # 获取给定键的类型
    def get_type(self, key: str, try_suffixes: Sequence[str] = ()) -> MODEL_TENSOR | None:
        # 获取给定键的类型和名称
        result = self.get_type_and_name(key, try_suffixes = try_suffixes)
        # 如果结果为 None，则返回 None
        if result is None:
            return None
        # 返回结果的第一个元素
        return result[0]

    # 获取给定键的值
    def __getitem__(self, key: str) -> str:
        try:
            # 返回给定键对应的值
            return self.mapping[key][1]
        except KeyError:
            # 如果键不存在，则抛出 KeyError 异常
            raise KeyError(key)

    # 检查给定键是否存在于映射中
    def __contains__(self, key: str) -> bool:
        # 返回给定键是否存在于映射中的布尔值
        return key in self.mapping

    # 返回映射的字符串表示形式
    def __repr__(self) -> str:
# 返回一个表示对象的字符串
return repr(self.mapping)

# 根据模型架构和块数获取张量名称映射
def get_tensor_name_map(arch: MODEL_ARCH, n_blocks: int) -> TensorNameMap:
    return TensorNameMap(arch, n_blocks)
```