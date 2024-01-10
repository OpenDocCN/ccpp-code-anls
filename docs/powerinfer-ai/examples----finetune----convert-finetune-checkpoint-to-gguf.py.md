# `PowerInfer\examples\finetune\convert-finetune-checkpoint-to-gguf.py`

```
#!/usr/bin/env python3
# 设置脚本的解释器为 Python3

# 导入必要的库
import argparse  # 用于解析命令行参数
import gguf  # 导入 gguf 模块
import os  # 提供与操作系统交互的功能
import struct  # 用于处理二进制数据
import sys  # 提供与 Python 解释器交互的功能
import numpy as np  # 导入 NumPy 库，用于处理数组和矩阵
from pathlib import Path  # 用于处理文件路径的类

# gguf 模块中定义的常量
LLM_KV_OPTIMIZER_TYPE = "optimizer.type"
LLM_KV_OPTIMIZER_TYPE_ADAM  = "adam"
LLM_KV_OPTIMIZER_TYPE_LBFGS = "lbfgs"
LLM_KV_OPTIMIZER_FILE_VERSION               = "optimizer.file_version"
LLM_KV_OPTIMIZER_CONVERGENCE_PAST_COUNT     = "optimizer.convergence_past_count"
LLM_KV_OPTIMIZER_PARAMETER_COUNT            = "optimizer.parameter_count"
LLM_KV_OPTIMIZER_ITERATION_COUNT            = "optimizer.iteration_count"
LLM_KV_OPTIMIZER_JUST_INITIALIZED           = "optimizer.just_initialized"
LLM_KV_OPTIMIZER_ADAM_BEST_LOSS             = "optimizer.adam.best_loss"
LLM_KV_OPTIMIZER_ADAM_PREVIOUS_LOSS         = "optimizer.adam.previous_loss"
LLM_KV_OPTIMIZER_ADAM_NO_IMPROVEMENT_COUNT  = "optimizer.adam.no_improvement_count"
LLM_KV_OPTIMIZER_LBFGS_APPROX_HESSIAN_COUNT = "optimizer.lbfgs.approx_hessian_count"
LLM_KV_OPTIMIZER_LBFGS_BEST_LOSS            = "optimizer.lbfgs.best_loss"
LLM_KV_OPTIMIZER_LBFGS_LINE_SEARCH_STEP     = "optimizer.lbfgs.line_search_step"
LLM_KV_OPTIMIZER_LBFGS_LINE_SEARCH_J        = "optimizer.lbfgs.line_search_j"
LLM_KV_OPTIMIZER_LBFGS_LINE_SEARCH_K        = "optimizer.lbfgs.line_search_k"
LLM_KV_OPTIMIZER_LBFGS_LINE_SEARCH_END      = "optimizer.lbfgs.line_search_end"
LLM_KV_OPTIMIZER_LBFGS_NO_IMPROVEMENT_COUNT = "optimizer.lbfgs.no_improvement_count"

LLM_TENSOR_OPTIMIZER_ADAM_FIRST_MOMENTS    = "optimizer.adam.first_moments"
LLM_TENSOR_OPTIMIZER_ADAM_SECOND_MOMENTS   = "optimizer.adam.second_moments"
LLM_TENSOR_OPTIMIZER_ADAM_PAST_LOSS_VALUES = "optimizer.adam.past_loss_values"

LLM_TENSOR_OPTIMIZER_LBFGS_CURRENT_PARAMETERS  = "optimizer.lbfgs.current_parameters"
LLM_TENSOR_OPTIMIZER_LBFGS_PREVIOUS_PARAMETERS = "optimizer.lbfgs.previous_parameters"
LLM_TENSOR_OPTIMIZER_LBFGS_CURRENT_GRADIENTS   = "optimizer.lbfgs.current_gradients"
# 定义一系列字符串常量，用于表示张量优化器的不同属性
LLM_TENSOR_OPTIMIZER_LBFGS_PREVIOUS_GRADIENTS  = "optimizer.lbfgs.previous_gradients"
LLM_TENSOR_OPTIMIZER_LBFGS_SEARCH_DIRECTION    = "optimizer.lbfgs.search_direction"
LLM_TENSOR_OPTIMIZER_LBFGS_PAST_LOSS_VALUES    = "optimizer.lbfgs.past_loss_values"
LLM_TENSOR_OPTIMIZER_LBFGS_MEMORY_ALPHA        = "optimizer.lbfgs.memory_alpha"
LLM_TENSOR_OPTIMIZER_LBFGS_MEMORY_YS           = "optimizer.lbfgs.memory_ys"
LLM_TENSOR_OPTIMIZER_LBFGS_MEMORY_S            = "optimizer.lbfgs.memory_s"
LLM_TENSOR_OPTIMIZER_LBFGS_MEMORY_Y            = "optimizer.lbfgs.memory_y"

# 定义训练类型的字符串常量
LLM_KV_TRAINING_TYPE_TRAIN_MODEL   = "train_model"
LLM_KV_TRAINING_TYPE_FINETUNE_LORA = "finetune_lora"
LLM_KV_TRAINING_TYPE               = "training.type"
LLM_KV_TRAINING_FILE_VERSION       = "training.file_version"
LLM_KV_TRAINING_ITERATION_COUNT    = "training.iteration_count"
LLM_KV_TRAINING_SAMPLE_COUNT       = "training.sample_count"
LLM_KV_TRAINING_TOKEN_COUNT        = "training.token_count"

# 定义 LORA 模型训练过程中的字符串常量
LLM_KV_TRAINING_LORA_RANK_TOKEN_EMBD  = "training.lora.rank.token_embd"
LLM_KV_TRAINING_LORA_RANK_OUTPUT_NORM = "training.lora.rank.output_norm"
LLM_KV_TRAINING_LORA_RANK_OUTPUT      = "training.lora.rank.output"
LLM_KV_TRAINING_LORA_RANK_ATTN_NORM   = "training.lora.rank.attn_norm"
LLM_KV_TRAINING_LORA_RANK_ATTN_Q      = "training.lora.rank.attn_q"
LLM_KV_TRAINING_LORA_RANK_ATTN_K      = "training.lora.rank.attn_k"
LLM_KV_TRAINING_LORA_RANK_ATTN_V      = "training.lora.rank.attn_v"
LLM_KV_TRAINING_LORA_RANK_ATTN_OUT    = "training.lora.rank.attn_output"
LLM_KV_TRAINING_LORA_RANK_FFN_NORM    = "training.lora.rank.ffn_norm"
LLM_KV_TRAINING_LORA_RANK_FFN_GATE    = "training.lora.rank.ffn_gate"
LLM_KV_TRAINING_LORA_RANK_FFN_DOWN    = "training.lora.rank.ffn_down"
LLM_KV_TRAINING_LORA_RANK_FFN_UP      = "training.lora.rank.ffn_up"

# 定义 Tensor 类
class Tensor:
    # 初始化函数，设置数据类型和元素个数，默认为空列表
    def __init__(self, dtype='f', ne=None):
        # 如果元素个数为空，则设置为一个空列表
        if ne is None:
            ne = []
        # 设置数据类型
        self.dtype = dtype
        # 设置元素个数
        self.ne = ne
        # 设置字节数为0
        self.nbytes = 0
        # 如果数据类型为 'f'
        if self.dtype == 'f':
            # 如果元素个数为空
            if len(self.ne) == 0:
                # 字节数为0
                self.nbytes = 0
            else:
                # 计算字节数
                self.nbytes = int(np.product(self.ne)) * 4
        else:
            # 抛出数值错误，处理不了的数据类型
            raise ValueError(f"Unhandled data type '{self.dtype}'")

    # 加载函数，从给定数据和偏移量加载数据
    def load(self, data, offset):
        # 读取数据的维度
        nd = struct.unpack('<I', bytes(data[offset:offset + 4]))[0]; offset += 4
        # 读取数据的名称长度
        namelen = struct.unpack('<I', bytes(data[offset:offset + 4]))[0]; offset += 4
        # 读取数据的类型
        dtype = struct.unpack('<I', bytes(data[offset:offset + 4]))[0]; offset += 4

        # 断言数据的维度与元素个数列表长度相等
        assert(nd == len(self.ne))
        # 初始化一个空列表
        ne = []
        # 遍历数据的维度
        for d in range(nd):
            # 读取每个维度的元素个数
            n = struct.unpack('<I', bytes(data[offset:offset + 4]))[0]; offset += 4
            ne.append(n)

        # 如果读取的元素个数列表与对象的元素个数列表不相等
        if tuple(ne) != tuple(self.ne):
            # 抛出数值错误，期望的元素个数与文件中读取的不匹配
            raise ValueError(f"Tensor.load: Expected number of elements {str(self.ne)} does not match what is read from file {str(ne)}")

        # 如果数据类型为 'f'
        if self.dtype == 'f':
            # 断言数据类型为0
            assert(dtype == 0)
        else:
            # 抛出数值错误，处理不了的数据类型
            raise ValueError(f"Unhandled data type '{self.dtype}'")

        # 读取数据的名称
        self.name = bytes(data[offset:offset+namelen]); offset += namelen
        # 32字节对齐
        offset += (0 - offset) & 31
        # 读取数据
        self.data = data[offset:offset+self.nbytes]
        offset += self.nbytes
        # 返回偏移量
        return offset

    # 计算最大存储空间
    def max_storage_size(self):
        result = 0
        # 加上维度的字节数
        result += 4 # nd
        # 加上名称长度的字节数
        result += 4 # namelen
        # 加上数据类型的字节数
        result += 4 # dtype
        # 加上元素个数列表的字节数
        result += len(self.ne)*8 # ne
        # 加上名称的字节数（根据提交3b5515bbe0e2224425986ba24f1f5d84aa38dce9的最大值）
        result += 48 # name (maximum as of commit 3b5515bbe0e2224425986ba24f1f5d84aa38dce9)
        # 加上32字节对齐的字节数
        result += 31 # 32-byte alignment
        # 加上数据的字节数
        result += self.nbytes
        # 返回结果
        return result
    # 保存 GGUF（通用图形格式）数据，将张量添加到 GGUF 写入器中
    def save_gguf(self, gguf_writer, name):
        # 将张量添加到 GGUF 写入器中，包括名称、数据、反转后的形状和数据类型
        gguf_writer.add_tensor(
            name=name,
            tensor=self.data,
            raw_shape=np.array(list(reversed(self.ne))),
            raw_dtype=gguf.GGMLQuantizationType.F32)
# 定义 OptimizationContext 类
class OptimizationContext:
    # 初始化方法，无需执行任何操作
    def __init__(self):
        pass

# 定义 LoraParams 类
class LoraParams:
    # 初始化方法，无需执行任何操作
    def __init__(self):
        pass

    # 加载数据的方法，接收数据和偏移量作为参数
    def load(self, data, offset):
        # 从数据中按照小端序解包 4 字节数据，赋值给 n_rank_attention_norm，并更新偏移量
        self.n_rank_attention_norm  = struct.unpack('<I', bytes(data[offset:offset + 4]))[0];  offset += 4
        # 从数据中按照小端序解包 4 字节数据，赋值给 n_rank_wq，并更新偏移量
        self.n_rank_wq              = struct.unpack('<I', bytes(data[offset:offset + 4]))[0];  offset += 4
        # 从数据中按照小端序解包 4 字节数据，赋值给 n_rank_wk，并更新偏移量
        self.n_rank_wk              = struct.unpack('<I', bytes(data[offset:offset + 4]))[0];  offset += 4
        # 从数据中按照小端序解包 4 字节数据，赋值给 n_rank_wv，并更新偏移量
        self.n_rank_wv              = struct.unpack('<I', bytes(data[offset:offset + 4]))[0];  offset += 4
        # 从数据中按照小端序解包 4 字节数据，赋值给 n_rank_wo，并更新偏移量
        self.n_rank_wo              = struct.unpack('<I', bytes(data[offset:offset + 4]))[0];  offset += 4
        # 从数据中按照小端序解包 4 字节数据，赋值给 n_rank_ffn_norm，并更新偏移量
        self.n_rank_ffn_norm        = struct.unpack('<I', bytes(data[offset:offset + 4]))[0];  offset += 4
        # 从数据中按照小端序解包 4 字节数据，赋值给 n_rank_w1，并更新偏移量
        self.n_rank_w1              = struct.unpack('<I', bytes(data[offset:offset + 4]))[0];  offset += 4
        # 从数据中按照小端序解包 4 字节数据，赋值给 n_rank_w2，并更新偏移量
        self.n_rank_w2              = struct.unpack('<I', bytes(data[offset:offset + 4]))[0];  offset += 4
        # 从数据中按照小端序解包 4 字节数据，赋值给 n_rank_w3，并更新偏移量
        self.n_rank_w3              = struct.unpack('<I', bytes(data[offset:offset + 4]))[0];  offset += 4
        # 从数据中按照小端序解包 4 字节数据，赋值给 n_rank_tok_embeddings，并更新偏移量
        self.n_rank_tok_embeddings  = struct.unpack('<I', bytes(data[offset:offset + 4]))[0];  offset += 4
        # 从数据中按照小端序解包 4 字节数据，赋值给 n_rank_norm，并更新偏移量
        self.n_rank_norm            = struct.unpack('<I', bytes(data[offset:offset + 4]))[0];  offset += 4
        # 从数据中按照小端序解包 4 字节数据，赋值给 n_rank_output，并更新偏移量
        self.n_rank_output          = struct.unpack('<I', bytes(data[offset:offset + 4]))[0];  offset += 4
        # 返回更新后的偏移量
        return offset
    # 保存 LLM_KV_TRAINING_LORA_RANK_TOKEN_EMBD 的无符号 32 位整数值到 gguf_writer
    gguf_writer.add_uint32(LLM_KV_TRAINING_LORA_RANK_TOKEN_EMBD,  self.n_rank_tok_embeddings)
    # 保存 LLM_KV_TRAINING_LORA_RANK_OUTPUT_NORM 的无符号 32 位整数值到 gguf_writer
    gguf_writer.add_uint32(LLM_KV_TRAINING_LORA_RANK_OUTPUT_NORM, self.n_rank_norm)
    # 保存 LLM_KV_TRAINING_LORA_RANK_OUTPUT 的无符号 32 位整数值到 gguf_writer
    gguf_writer.add_uint32(LLM_KV_TRAINING_LORA_RANK_OUTPUT,      self.n_rank_output)
    # 保存 LLM_KV_TRAINING_LORA_RANK_ATTN_NORM 的无符号 32 位整数值到 gguf_writer
    gguf_writer.add_uint32(LLM_KV_TRAINING_LORA_RANK_ATTN_NORM,   self.n_rank_attention_norm)
    # 保存 LLM_KV_TRAINING_LORA_RANK_ATTN_Q 的无符号 32 位整数值到 gguf_writer
    gguf_writer.add_uint32(LLM_KV_TRAINING_LORA_RANK_ATTN_Q,      self.n_rank_wq)
    # 保存 LLM_KV_TRAINING_LORA_RANK_ATTN_K 的无符号 32 位整数值到 gguf_writer
    gguf_writer.add_uint32(LLM_KV_TRAINING_LORA_RANK_ATTN_K,      self.n_rank_wk)
    # 保存 LLM_KV_TRAINING_LORA_RANK_ATTN_V 的无符号 32 位整数值到 gguf_writer
    gguf_writer.add_uint32(LLM_KV_TRAINING_LORA_RANK_ATTN_V,      self.n_rank_wv)
    # 保存 LLM_KV_TRAINING_LORA_RANK_ATTN_OUT 的无符号 32 位整数值到 gguf_writer
    gguf_writer.add_uint32(LLM_KV_TRAINING_LORA_RANK_ATTN_OUT,    self.n_rank_wo)
    # 保存 LLM_KV_TRAINING_LORA_RANK_FFN_NORM 的无符号 32 位整数值到 gguf_writer
    gguf_writer.add_uint32(LLM_KV_TRAINING_LORA_RANK_FFN_NORM,    self.n_rank_ffn_norm)
    # 保存 LLM_KV_TRAINING_LORA_RANK_FFN_GATE 的无符号 32 位整数值到 gguf_writer
    gguf_writer.add_uint32(LLM_KV_TRAINING_LORA_RANK_FFN_GATE,    self.n_rank_w1)
    # 保存 LLM_KV_TRAINING_LORA_RANK_FFN_DOWN 的无符号 32 位整数值到 gguf_writer
    gguf_writer.add_uint32(LLM_KV_TRAINING_LORA_RANK_FFN_DOWN,    self.n_rank_w2)
    # 保存 LLM_KV_TRAINING_LORA_RANK_FFN_UP 的无符号 32 位整数值到 gguf_writer
    gguf_writer.add_uint32(LLM_KV_TRAINING_LORA_RANK_FFN_UP,      self.n_rank_w3)
# 定义一个名为 ModelParams 的类
class ModelParams:
    # 初始化方法，设置参数 n_ff 的默认值为 None
    def __init__(self, n_ff = None):
        self.n_ff = n_ff

    # 加载方法，从给定数据和偏移量处读取参数并赋值给对象属性
    def load(self, data, offset):
        # 从数据中读取 n_vocab，并更新偏移量
        self.n_vocab = struct.unpack('<I', bytes(data[offset:offset + 4]))[0];  offset += 4
        # 从数据中读取 n_embd，并更新偏移量
        self.n_embd  = struct.unpack('<I', bytes(data[offset:offset + 4]))[0];  offset += 4
        # 从数据中读取 n_mult，并更新偏移量
        self.n_mult  = struct.unpack('<I', bytes(data[offset:offset + 4]))[0];  offset += 4
        # 从数据中读取 n_head，并更新偏移量
        self.n_head  = struct.unpack('<I', bytes(data[offset:offset + 4]))[0];  offset += 4
        # 从数据中读取 n_layer，并更新偏移量
        self.n_layer = struct.unpack('<I', bytes(data[offset:offset + 4]))[0];  offset += 4
        # 从数据中读取 n_rot，并更新偏移量
        self.n_rot   = struct.unpack('<I', bytes(data[offset:offset + 4]))[0];  offset += 4
        # 返回更新后的偏移量
        return offset

    # 获取 n_ff 的方法
    def get_n_ff(self):
        # 如果 n_ff 为 None，则根据公式计算并返回值
        if self.n_ff is None:
            return ((2*(4*self.n_embd)//3 + self.n_mult - 1)//self.n_mult)*self.n_mult
        # 否则直接返回 n_ff 的值
        else:
            return self.n_ff

    # 保存 gguf 的方法
    def save_gguf(self, gguf_writer):
        # 不保存 self.n_vocab
        gguf_writer.add_embedding_length(self.n_embd)
        gguf_writer.add_head_count(self.n_head)
        gguf_writer.add_block_count(self.n_layer)
        gguf_writer.add_rope_dimension_count(self.n_rot)
        gguf_writer.add_feed_forward_length(self.get_n_ff())

# 定义一个名为 tensor_name 的函数，接受 key、bid 和 suffix 三个参数
def tensor_name(key, bid=None, suffix=".weight"):
    # 返回根据 key 和 bid 格式化后的字符串，再加上后缀
    return gguf.TENSOR_NAMES[key].format(bid=bid) + suffix

# 定义一个名为 Layer 的类
class Layer:
    # 初始化函数，接受参数 params, lora_params, bid
    def __init__(self, params, lora_params, bid):
        # 将参数 bid 赋值给对象的属性 bid
        self.bid = bid
        # 初始化属性 att_norm_a 为一个 Tensor 对象，包含 lora_params.n_rank_attention_norm 行，params.n_embd 列的矩阵
        self.att_norm_a = Tensor('f', [lora_params.n_rank_attention_norm, params.n_embd])
        # 初始化属性 att_norm_b 为一个 Tensor 对象，包含 lora_params.n_rank_attention_norm 行，1 列的矩阵
        self.att_norm_b = Tensor('f', [lora_params.n_rank_attention_norm, 1])
        # 初始化属性 wq_a 为一个 Tensor 对象，包含 lora_params.n_rank_wq 行，params.n_embd 列的矩阵
        self.wq_a       = Tensor('f', [lora_params.n_rank_wq, params.n_embd])
        # 初始化属性 wq_b 为一个 Tensor 对象，包含 lora_params.n_rank_wq 行，params.n_embd 列的矩阵
        self.wq_b       = Tensor('f', [lora_params.n_rank_wq, params.n_embd])
        # 初始化属性 wk_a 为一个 Tensor 对象，包含 lora_params.n_rank_wk 行，params.n_embd 列的矩阵
        self.wk_a       = Tensor('f', [lora_params.n_rank_wk, params.n_embd])
        # 初始化属性 wk_b 为一个 Tensor 对象，包含 lora_params.n_rank_wk 行，params.n_embd 列的矩阵
        self.wk_b       = Tensor('f', [lora_params.n_rank_wk, params.n_embd])
        # 初始化属性 wv_a 为一个 Tensor 对象，包含 lora_params.n_rank_wv 行，params.n_embd 列的矩阵
        self.wv_a       = Tensor('f', [lora_params.n_rank_wv, params.n_embd])
        # 初始化属性 wv_b 为一个 Tensor 对象，包含 lora_params.n_rank_wv 行，params.n_embd 列的矩阵
        self.wv_b       = Tensor('f', [lora_params.n_rank_wv, params.n_embd])
        # 初始化属性 wo_a 为一个 Tensor 对象，包含 lora_params.n_rank_wo 行，params.n_embd 列的矩阵
        self.wo_a       = Tensor('f', [lora_params.n_rank_wo, params.n_embd])
        # 初始化属性 wo_b 为一个 Tensor 对象，包含 lora_params.n_rank_wo 行，params.n_embd 列的矩阵
        self.wo_b       = Tensor('f', [lora_params.n_rank_wo, params.n_embd])
        # 初始化属性 ffn_norm_a 为一个 Tensor 对象，包含 lora_params.n_rank_ffn_norm 行，params.n_embd 列的矩阵
        self.ffn_norm_a = Tensor('f', [lora_params.n_rank_ffn_norm, params.n_embd])
        # 初始化属性 ffn_norm_b 为一个 Tensor 对象，包含 lora_params.n_rank_ffn_norm 行，1 列的矩阵
        self.ffn_norm_b = Tensor('f', [lora_params.n_rank_ffn_norm, 1])
        # 初始化属性 w1_a 为一个 Tensor 对象，包含 lora_params.n_rank_w1 行，params.n_embd 列的矩阵
        self.w1_a       = Tensor('f', [lora_params.n_rank_w1, params.n_embd])
        # 初始化属性 w1_b 为一个 Tensor 对象，包含 lora_params.n_rank_w1 行，params.get_n_ff() 列的矩阵
        self.w1_b       = Tensor('f', [lora_params.n_rank_w1, params.get_n_ff()])
        # 初始化属性 w2_a 为一个 Tensor 对象，包含 lora_params.n_rank_w2 行，params.get_n_ff() 列的矩阵
        self.w2_a       = Tensor('f', [lora_params.n_rank_w2, params.get_n_ff()])
        # 初始化属性 w2_b 为一个 Tensor 对象，包含 lora_params.n_rank_w2 行，params.n_embd 列的矩阵
        self.w2_b       = Tensor('f', [lora_params.n_rank_w2, params.n_embd])
        # 初始化属性 w3_a 为一个 Tensor 对象，包含 lora_params.n_rank_w3 行，params.n_embd 列的矩阵
        self.w3_a       = Tensor('f', [lora_params.n_rank_w3, params.n_embd])
        # 初始化属性 w3_b 为一个 Tensor 对象，包含 lora_params.n_rank_w3 行，params.get_n_ff() 列的矩阵
        self.w3_b       = Tensor('f', [lora_params.n_rank_w3, params.get_n_ff()])
    # 从给定数据和偏移量加载att_norm_a，并更新偏移量
    offset = self.att_norm_a.load(data, offset)
    # 从给定数据和偏移量加载att_norm_b，并更新偏移量
    offset = self.att_norm_b.load(data, offset)
    # 从给定数据和偏移量加载wq_a，并更新偏移量
    offset = self.wq_a.load(data, offset)
    # 从给定数据和偏移量加载wq_b，并更新偏移量
    offset = self.wq_b.load(data, offset)
    # 从给定数据和偏移量加载wk_a，并更新偏移量
    offset = self.wk_a.load(data, offset)
    # 从给定数据和偏移量加载wk_b，并更新偏移量
    offset = self.wk_b.load(data, offset)
    # 从给定数据和偏移量加载wv_a，并更新偏移量
    offset = self.wv_a.load(data, offset)
    # 从给定数据和偏移量加载wv_b，并更新偏移量
    offset = self.wv_b.load(data, offset)
    # 从给定数据和偏移量加载wo_a，并更新偏移量
    offset = self.wo_a.load(data, offset)
    # 从给定数据和偏移量加载wo_b，并更新偏移量
    offset = self.wo_b.load(data, offset)
    # 从给定数据和偏移量加载ffn_norm_a，并更新偏移量
    offset = self.ffn_norm_a.load(data, offset)
    # 从给定数据和偏移量加载ffn_norm_b，并更新偏移量
    offset = self.ffn_norm_b.load(data, offset)
    # 从给定数据和偏移量加载w1_a，并更新偏移量
    offset = self.w1_a.load(data, offset)
    # 从给定数据和偏移量加载w1_b，并更新偏移量
    offset = self.w1_b.load(data, offset)
    # 从给定数据和偏移量加载w2_a，并更新偏移量
    offset = self.w2_a.load(data, offset)
    # 从给定数据和偏移量加载w2_b，并更新偏移量
    offset = self.w2_b.load(data, offset)
    # 从给定数据和偏移量加载w3_a，并更新偏移量
    offset = self.w3_a.load(data, offset)
    # 从给定数据和偏移量加载w3_b，并更新偏移量
    offset = self.w3_b.load(data, offset)
    # 返回最终的偏移量
    return offset
class LoraModel:
    # 初始化函数，创建模型参数对象、Lora参数对象和空的层列表
    def __init__(self, n_ff = None):
        self.params = ModelParams(n_ff = n_ff)  # 创建模型参数对象
        self.lora_params = LoraParams()  # 创建Lora参数对象
        self.layers = []  # 创建空的层列表

    # 加载函数，从给定数据和偏移量加载模型参数和层
    def load(self, data, offset):
        offset = self.params.load(data, offset)  # 加载模型参数
        offset = self.lora_params.load(data, offset)  # 加载Lora参数

        # 创建并加载token嵌入矩阵、归一化矩阵和输出矩阵
        self.tok_embd_a = Tensor('f', [self.lora_params.n_rank_tok_embeddings, self.params.n_embd])
        self.tok_embd_b = Tensor('f', [self.lora_params.n_rank_tok_embeddings, self.params.n_vocab])
        self.norm_a     = Tensor('f', [self.lora_params.n_rank_norm, self.params.n_embd])
        self.norm_b     = Tensor('f', [self.lora_params.n_rank_norm, 1])
        self.output_a   = Tensor('f', [self.lora_params.n_rank_output, self.params.n_embd])
        self.output_b   = Tensor('f', [self.lora_params.n_rank_output, self.params.n_vocab])

        offset = self.tok_embd_a.load(data, offset)  # 加载token嵌入矩阵
        offset = self.tok_embd_b.load(data, offset)  # 加载token嵌入矩阵
        offset = self.norm_a.load(data, offset)  # 加载归一化矩阵
        offset = self.norm_b.load(data, offset)  # 加载归一化矩阵
        offset = self.output_a.load(data, offset)  # 加载输出矩阵
        offset = self.output_b.load(data, offset)  # 加载输出矩阵

        self.layers.clear()  # 清空层列表
        # 遍历模型参数中的层数，创建并加载每一层
        for bid in range(self.params.n_layer):
            layer = Layer(self.params, self.lora_params, bid)  # 创建层对象
            offset = layer.load(data, offset)  # 加载层
            self.layers.append(layer)  # 将加载的层添加到层列表中

        return offset  # 返回最终的偏移量
    # 保存 GGUF 文件，将参数保存到 GGUF 写入器中
    def save_gguf(self, gguf_writer):
        self.params.save_gguf(gguf_writer)  # 保存参数到 GGUF 写入器中
        self.lora_params.save_gguf(gguf_writer)  # 保存 LORA 参数到 GGUF 写入器中

        # 保存 token_embd_a 的 GGUF 文件，指定名称为 tensor_name(gguf.MODEL_TENSOR.TOKEN_EMBD, suffix=".weight.lora_a")
        self.tok_embd_a.save_gguf(gguf_writer, name=tensor_name(gguf.MODEL_TENSOR.TOKEN_EMBD,  suffix=".weight.lora_a"))
        # 保存 token_embd_b 的 GGUF 文件，指定名称为 tensor_name(gguf.MODEL_TENSOR.TOKEN_EMBD, suffix=".weight.lora_b")
        self.tok_embd_b.save_gguf(gguf_writer, name=tensor_name(gguf.MODEL_TENSOR.TOKEN_EMBD,  suffix=".weight.lora_b"))
        # 保存 norm_a 的 GGUF 文件，指定名称为 tensor_name(gguf.MODEL_TENSOR.OUTPUT_NORM, suffix=".weight.lora_a")
        self.norm_a.save_gguf    (gguf_writer, name=tensor_name(gguf.MODEL_TENSOR.OUTPUT_NORM, suffix=".weight.lora_a"))
        # 保存 norm_b 的 GGUF 文件，指定名称为 tensor_name(gguf.MODEL_TENSOR.OUTPUT_NORM, suffix=".weight.lora_b")
        self.norm_b.save_gguf    (gguf_writer, name=tensor_name(gguf.MODEL_TENSOR.OUTPUT_NORM, suffix=".weight.lora_b"))
        # 保存 output_a 的 GGUF 文件，指定名称为 tensor_name(gguf.MODEL_TENSOR.OUTPUT, suffix=".weight.lora_a")
        self.output_a.save_gguf  (gguf_writer, name=tensor_name(gguf.MODEL_TENSOR.OUTPUT,      suffix=".weight.lora_a"))
        # 保存 output_b 的 GGUF 文件，指定名称为 tensor_name(gguf.MODEL_TENSOR.OUTPUT, suffix=".weight.lora_b")
        self.output_b.save_gguf  (gguf_writer, name=tensor_name(gguf.MODEL_TENSOR.OUTPUT,      suffix=".weight.lora_b"))

        # 遍历每个层，将每个层的 GGUF 文件保存到 GGUF 写入器中
        for layer in self.layers:
            layer.save_gguf(gguf_writer)
# 定义 LoraCheckpoint 类
class LoraCheckpoint:
    # 初始化方法，接受 n_ff 参数，默认为 None
    def __init__(self, n_ff = None):
        # 创建 LoraModel 对象，并将 n_ff 参数传入
        self.model = LoraModel(n_ff = n_ff)
        # 创建 OptimizationContext 对象
        self.opt_ctx = OptimizationContext()

    # 加载方法，接受 data 和 offset 两个参数
    def load(self, data, offset):
        # 读取文件头的 magic 字段，长度为 4 字节
        magic   = bytes(reversed(data[offset:offset + 4])); offset += 4
        # 检查 magic 字段是否为 'ggcl'，如果不是则抛出数值错误
        if magic != b'ggcl':
            raise ValueError(f"File header magic indicates, that this is no finetune-lora checkpoint file. Expected 'ggcl', Got '{str(magic)}'")

        # 读取版本号字段，长度为 4 字节
        self.version = struct.unpack('<I', bytes(data[offset:offset + 4]))[0]; offset += 4
        # 检查版本号是否为 0，如果不是则抛出数值错误
        if self.version != 0:
            raise ValueError('Invalid version of checkpoint file')

        # 读取训练迭代次数字段，长度为 4 字节
        self.train_its     = struct.unpack('<I', bytes(data[offset:offset + 4]))[0]; offset += 4
        # 读取训练样本数字段，长度为 4 字节
        self.train_samples = struct.unpack('<I', bytes(data[offset:offset + 4]))[0]; offset += 4
        # 读取训练标记数字段，长度为 4 字节
        self.train_tokens  = struct.unpack('<I', bytes(data[offset:offset + 4]))[0]; offset += 4

        # 调用 LoraModel 对象的 load 方法，加载模型数据，更新 offset
        offset = self.model.load(data, offset)
        # 调用 OptimizationContext 对象的 load 方法，加载优化上下文数据，更新 offset
        offset = self.opt_ctx.load(data, offset)

        # 返回更新后的 offset
        return offset

    # 保存为 GGUF 格式的方法，接受 gguf_writer 参数
    def save_gguf(self, gguf_writer):
        # 向 gguf_writer 中添加文件类型为 F32
        gguf_writer.add_file_type(gguf.GGMLQuantizationType.F32)
        # 向 gguf_writer 中添加层归一化的 RMS 误差值
        gguf_writer.add_layer_norm_rms_eps(1e-5)
        # 向 gguf_writer 中添加键值对训练文件版本号
        gguf_writer.add_uint32(LLM_KV_TRAINING_FILE_VERSION,    0)
        # 向 gguf_writer 中添加键值对训练类型
        gguf_writer.add_string(LLM_KV_TRAINING_TYPE,            LLM_KV_TRAINING_TYPE_FINETUNE_LORA)
        # 向 gguf_writer 中添加键值对训练迭代次数
        gguf_writer.add_uint32(LLM_KV_TRAINING_ITERATION_COUNT, self.train_its)
        # 向 gguf_writer 中添加键值对训练样本数
        gguf_writer.add_uint32(LLM_KV_TRAINING_SAMPLE_COUNT,    self.train_samples)
        # 向 gguf_writer 中添加键值对训练标记数
        gguf_writer.add_uint32(LLM_KV_TRAINING_TOKEN_COUNT,     self.train_tokens)
        # 调用 LoraModel 对象的 save_gguf 方法，保存模型数据到 gguf_writer
        self.model.save_gguf(gguf_writer)
        # 调用 OptimizationContext 对象的 save_gguf 方法，保存优化上下文数据到 gguf_writer
        self.opt_ctx.save_gguf(gguf_writer)

# 处理命令行参数的方法
def handle_args():
    # 创建参数解析器对象，设置描述信息
    parser = argparse.ArgumentParser(description = 'Convert finetune checkpoints to GGUF')
    # 添加输入文件参数，类型为 Path，帮助信息为输入 finetune checkpoint 文件名，必填
    parser.add_argument('--input',  '-i', type = Path, help = 'Input finetune checkpoint filename', required=True)
    # 添加一个名为'--output'的命令行参数，缩写为'-o'，类型为Path，帮助信息为'Output GGUF filename'，必须提供
    parser.add_argument('--output', '-o', type = Path, help = 'Output GGUF filename', required=True)
    # 添加一个名为'--ff'的命令行参数，类型为int，帮助信息为'Feedforward size, if not provided compute from n_mult. Provide this if you get 'ValueError: Tensor.load: Expected number of elements does not match what is read from file''，可选参数
    parser.add_argument('--ff', type = int, help = "Feedforward size, if not provided compute from n_mult. Provide this if you get 'ValueError: Tensor.load: Expected number of elements does not match what is read from file'", required=False)
    # 解析命令行参数并返回
    return parser.parse_args()
# 主函数，程序的入口
def main():
    # 处理命令行参数，返回配置对象
    cfg = handle_args()
    # 打印配置信息
    print(cfg)
    # 以只读模式创建一个内存映射文件对象
    data = np.memmap(cfg.input, mode = 'r')
    # 创建一个 LoraCheckpoint 对象
    chk = LoraCheckpoint(n_ff = cfg.ff)
    # 设置偏移量为 0，并加载数据
    offset = 0
    offset = chk.load(data, offset)
    # 断言偏移量等于数据长度，即已经读取了所有可用数据
    assert(offset == len(data))

    # 创建一个 GGUFWriter 对象，指定输出文件和模型架构
    gguf_writer = gguf.GGUFWriter(cfg.output, gguf.MODEL_ARCH_NAMES[gguf.MODEL_ARCH.LLAMA], use_temp_file = False)
    # 保存 GGUF 数据到 GGUFWriter 对象
    chk.save_gguf(gguf_writer)
    # 打印提示信息
    print("    gguf: write header")
    # 将头部信息写入文件
    gguf_writer.write_header_to_file()
    # 打印提示信息
    print("    gguf: write metadata")
    # 将元数据写入文件
    gguf_writer.write_kv_data_to_file()
    # 打印提示信息
    print("    gguf: write tensors")
    # 将张量数据写入文件
    gguf_writer.write_tensors_to_file()
    # 关闭 GGUFWriter 对象
    gguf_writer.close()

# 如果当前脚本为主程序，则执行 main 函数
if __name__ == '__main__':
    main()
```