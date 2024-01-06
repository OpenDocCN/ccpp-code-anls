# `PowerInfer\examples\finetune\convert-finetune-checkpoint-to-gguf.py`

```
#!/usr/bin/env python3
# 设置脚本的解释器为 Python 3

import argparse
# 导入用于解析命令行参数的模块
import gguf
# 导入 gguf 模块
import os
# 导入操作系统相关的模块
import struct
# 导入处理二进制数据的模块
import sys
# 导入与 Python 解释器相关的模块
import numpy as np
# 导入数值计算库 numpy
from pathlib import Path
# 从 pathlib 模块中导入 Path 类

# gguf 常量
LLM_KV_OPTIMIZER_TYPE = "optimizer.type"
# 定义 gguf 中优化器类型的常量
LLM_KV_OPTIMIZER_TYPE_ADAM  = "adam"
# 定义 gguf 中优化器类型为 adam 的常量
LLM_KV_OPTIMIZER_TYPE_LBFGS = "lbfgs"
# 定义 gguf 中优化器类型为 lbfgs 的常量
LLM_KV_OPTIMIZER_FILE_VERSION               = "optimizer.file_version"
# 定义 gguf 中优化器文件版本的常量
LLM_KV_OPTIMIZER_CONVERGENCE_PAST_COUNT     = "optimizer.convergence_past_count"
# 定义 gguf 中优化器收敛过去计数的常量
LLM_KV_OPTIMIZER_PARAMETER_COUNT            = "optimizer.parameter_count"
# 定义 gguf 中优化器参数计数的常量
LLM_KV_OPTIMIZER_ITERATION_COUNT            = "optimizer.iteration_count"
# 定义 gguf 中优化器迭代计数的常量
LLM_KV_OPTIMIZER_JUST_INITIALIZED           = "optimizer.just_initialized"
# 定义 gguf 中优化器刚初始化的常量
# 定义一系列键值对，用于存储优化器 Adam 的最佳损失、先前损失、无改善计数等信息
LLM_KV_OPTIMIZER_ADAM_BEST_LOSS             = "optimizer.adam.best_loss"
LLM_KV_OPTIMIZER_ADAM_PREVIOUS_LOSS         = "optimizer.adam.previous_loss"
LLM_KV_OPTIMIZER_ADAM_NO_IMPROVEMENT_COUNT  = "optimizer.adam.no_improvement_count"

# 定义一系列键值对，用于存储优化器 LBFGS 的近似 Hessian 矩阵计数、最佳损失、线搜索步长等信息
LLM_KV_OPTIMIZER_LBFGS_APPROX_HESSIAN_COUNT = "optimizer.lbfgs.approx_hessian_count"
LLM_KV_OPTIMIZER_LBFGS_BEST_LOSS            = "optimizer.lbfgs.best_loss"
LLM_KV_OPTIMIZER_LBFGS_LINE_SEARCH_STEP     = "optimizer.lbfgs.line_search_step"
LLM_KV_OPTIMIZER_LBFGS_LINE_SEARCH_J        = "optimizer.lbfgs.line_search_j"
LLM_KV_OPTIMIZER_LBFGS_LINE_SEARCH_K        = "optimizer.lbfgs.line_search_k"
LLM_KV_OPTIMIZER_LBFGS_LINE_SEARCH_END      = "optimizer.lbfgs.line_search_end"
LLM_KV_OPTIMIZER_LBFGS_NO_IMPROVEMENT_COUNT = "optimizer.lbfgs.no_improvement_count"

# 定义一系列键值对，用于存储优化器 Adam 的一阶矩、二阶矩、过去损失值等张量信息
LLM_TENSOR_OPTIMIZER_ADAM_FIRST_MOMENTS    = "optimizer.adam.first_moments"
LLM_TENSOR_OPTIMIZER_ADAM_SECOND_MOMENTS   = "optimizer.adam.second_moments"
LLM_TENSOR_OPTIMIZER_ADAM_PAST_LOSS_VALUES = "optimizer.adam.past_loss_values"

# 定义一系列键值对，用于存储优化器 LBFGS 的当前参数、先前参数、当前梯度、先前梯度、搜索方向等张量信息
LLM_TENSOR_OPTIMIZER_LBFGS_CURRENT_PARAMETERS  = "optimizer.lbfgs.current_parameters"
LLM_TENSOR_OPTIMIZER_LBFGS_PREVIOUS_PARAMETERS = "optimizer.lbfgs.previous_parameters"
LLM_TENSOR_OPTIMIZER_LBFGS_CURRENT_GRADIENTS   = "optimizer.lbfgs.current_gradients"
LLM_TENSOR_OPTIMIZER_LBFGS_PREVIOUS_GRADIENTS  = "optimizer.lbfgs.previous_gradients"
LLM_TENSOR_OPTIMIZER_LBFGS_SEARCH_DIRECTION    = "optimizer.lbfgs.search_direction"
# 定义优化器 LBFGS 的过去损失值
LLM_TENSOR_OPTIMIZER_LBFGS_PAST_LOSS_VALUES    = "optimizer.lbfgs.past_loss_values"
# 定义优化器 LBFGS 的内存参数 alpha
LLM_TENSOR_OPTIMIZER_LBFGS_MEMORY_ALPHA        = "optimizer.lbfgs.memory_alpha"
# 定义优化器 LBFGS 的内存参数 ys
LLM_TENSOR_OPTIMIZER_LBFGS_MEMORY_YS           = "optimizer.lbfgs.memory_ys"
# 定义优化器 LBFGS 的内存参数 s
LLM_TENSOR_OPTIMIZER_LBFGS_MEMORY_S            = "optimizer.lbfgs.memory_s"
# 定义优化器 LBFGS 的内存参数 y
LLM_TENSOR_OPTIMIZER_LBFGS_MEMORY_Y            = "optimizer.lbfgs.memory_y"

# 定义训练类型为训练模型
LLM_KV_TRAINING_TYPE_TRAIN_MODEL   = "train_model"
# 定义训练类型为微调 LORA
LLM_KV_TRAINING_TYPE_FINETUNE_LORA = "finetune_lora"
# 定义训练类型
LLM_KV_TRAINING_TYPE               = "training.type"
# 定义训练文件版本
LLM_KV_TRAINING_FILE_VERSION       = "training.file_version"
# 定义训练迭代次数
LLM_KV_TRAINING_ITERATION_COUNT    = "training.iteration_count"
# 定义训练样本数量
LLM_KV_TRAINING_SAMPLE_COUNT       = "training.sample_count"
# 定义训练标记数量
LLM_KV_TRAINING_TOKEN_COUNT        = "training.token_count"

# 定义 LORA 排名的标记嵌入
LLM_KV_TRAINING_LORA_RANK_TOKEN_EMBD  = "training.lora.rank.token_embd"
# 定义 LORA 排名的输出规范化
LLM_KV_TRAINING_LORA_RANK_OUTPUT_NORM = "training.lora.rank.output_norm"
# 定义 LORA 排名的输出
LLM_KV_TRAINING_LORA_RANK_OUTPUT      = "training.lora.rank.output"
# 定义 LORA 排名的注意力规范化
LLM_KV_TRAINING_LORA_RANK_ATTN_NORM   = "training.lora.rank.attn_norm"
# 定义 LORA 排名的注意力查询
LLM_KV_TRAINING_LORA_RANK_ATTN_Q      = "training.lora.rank.attn_q"
# 定义 LORA 排名的注意力键
LLM_KV_TRAINING_LORA_RANK_ATTN_K      = "training.lora.rank.attn_k"
# 定义一系列的常量，用于表示训练中的不同类型的数据
LLM_KV_TRAINING_LORA_RANK_ATTN_V      = "training.lora.rank.attn_v"
LLM_KV_TRAINING_LORA_RANK_ATTN_OUT    = "training.lora.rank.attn_output"
LLM_KV_TRAINING_LORA_RANK_FFN_NORM    = "training.lora.rank.ffn_norm"
LLM_KV_TRAINING_LORA_RANK_FFN_GATE    = "training.lora.rank.ffn_gate"
LLM_KV_TRAINING_LORA_RANK_FFN_DOWN    = "training.lora.rank.ffn_down"
LLM_KV_TRAINING_LORA_RANK_FFN_UP      = "training.lora.rank.ffn_up"

# 定义一个名为Tensor的类
class Tensor:
    # 初始化方法，用于创建Tensor对象
    def __init__(self, dtype='f', ne=None):
        # 如果ne参数为None，则将其设置为空列表
        if ne is None:
            ne = []
        # 设置对象的数据类型和ne属性
        self.dtype = dtype
        self.ne = ne
        self.nbytes = 0
        # 根据数据类型和ne属性计算对象的字节数
        if self.dtype == 'f':
            if len(self.ne) == 0:
                self.nbytes = 0
            else:
                self.nbytes = int(np.product(self.ne)) * 4
        else:
            # 如果数据类型不是'f'，则需要补充代码来处理其他数据类型的情况
# 如果数据类型不被处理，则引发值错误
raise ValueError(f"Unhandled data type '{self.dtype}'")

# 从给定数据和偏移量加载数据
def load(self, data, offset):
    # 从数据中解包出nd（张量的维度）并更新偏移量
    nd = struct.unpack('<I', bytes(data[offset:offset + 4]))[0]; offset += 4
    # 从数据中解包出namelen（张量名称的长度）并更新偏移量
    namelen = struct.unpack('<I', bytes(data[offset:offset + 4]))[0]; offset += 4
    # 从数据中解包出dtype（张量数据类型）并更新偏移量
    dtype = struct.unpack('<I', bytes(data[offset:offset + 4]))[0]; offset += 4

    # 断言nd（张量的维度）与self.ne的长度相等
    assert(nd == len(self.ne))
    # 创建一个空列表ne
    ne = []
    # 遍历nd次，从数据中解包出n（每个维度的大小）并更新偏移量，将n添加到ne列表中
    for d in range(nd):
        n = struct.unpack('<I', bytes(data[offset:offset + 4]))[0]; offset += 4
        ne.append(n)

    # 如果ne与self.ne不相等，则引发值错误
    if tuple(ne) != tuple(self.ne):
        raise ValueError(f"Tensor.load: Expected number of elements {str(self.ne)} does not match what is read from file {str(ne)}")

    # 如果数据类型为'f'，则断言dtype为0
    if self.dtype == 'f':
        assert(dtype == 0)
    # 否则，引发值错误
    else:
        raise ValueError(f"Unhandled data type '{self.dtype}'")
        # 从数据中提取文件名，存储为字节类型，更新偏移量
        self.name = bytes(data[offset:offset+namelen]); offset += namelen
        # 32字节对齐
        offset += (0 - offset) & 31
        # 从数据中提取文件内容，更新偏移量
        self.data = data[offset:offset+self.nbytes]
        offset += self.nbytes
        # 返回更新后的偏移量
        return offset

    # 计算最大存储空间
    def max_storage_size(self):
        result = 0
        # 添加nd的存储空间
        result += 4 # nd
        # 添加namelen的存储空间
        result += 4 # namelen
        # 添加dtype的存储空间
        result += 4 # dtype
        # 添加ne列表的存储空间
        result += len(self.ne)*8 # ne
        # 添加name的存储空间
        result += 48 # name (maximum as of commit 3b5515bbe0e2224425986ba24f1f5d84aa38dce9)
        # 32字节对齐
        result += 31 # 32-byte alignment
        # 添加文件内容的存储空间
        result += self.nbytes
        # 返回总的存储空间
        return result

    # 保存gguf文件
    def save_gguf(self, gguf_writer, name):
# 将张量添加到GGUF写入器中
gguf_writer.add_tensor(
    name=name,  # 张量的名称
    tensor=self.data,  # 张量的数据
    raw_shape=np.array(list(reversed(self.ne))),  # 张量的原始形状
    raw_dtype=gguf.GGMLQuantizationType.F32  # 张量的原始数据类型
)

# 优化上下文类
class OptimizationContext:
    def __init__(self):
        pass

    # 从数据中加载优化上下文
    def load(self, data, offset):
        # 从数据中解析版本号
        self.version = struct.unpack('<I', bytes(data[offset:offset + 4]))[0]
        offset += 4

        # 如果版本号不是1，则抛出数值错误
        if self.version != 1:
            raise ValueError('Invalid version of optimization context in checkpoint file')

        # 解析过去的迭代次数、LBFGS参数m和nx
        self.past    = struct.unpack('<i', bytes(data[offset:offset + 4]))[0];  offset += 4
        self.lbfgs_m = struct.unpack('<i', bytes(data[offset:offset + 4]))[0];  offset += 4
        self.nx      = struct.unpack('N',  bytes(data[offset:offset + 8]))[0];  offset += 8
# 从数据中解析出一个整数，使用小端字节序，更新偏移量
self.iter = struct.unpack('<i', bytes(data[offset:offset + 4]))[0];  offset += 4
# 从数据中解析出一个整数，使用小端字节序，转换成布尔值，更新偏移量
self.just_initialized = bool(struct.unpack('<i', bytes(data[offset:offset + 4]))[0]);  offset += 4

# 创建指定形状的浮点数张量
self.adam_m = Tensor('f', [self.nx])
self.adam_v = Tensor('f', [self.nx])
self.adam_pf = Tensor('f', [self.past] if self.past > 0 else [])

# 创建指定形状的浮点数张量
self.lbfgs_x = Tensor('f', [self.nx])
self.lbfgs_xp = Tensor('f', [self.nx])
self.lbfgs_g = Tensor('f', [self.nx])
self.lbfgs_gp = Tensor('f', [self.nx])
self.lbfgs_d = Tensor('f', [self.nx])
self.lbfgs_pf = Tensor('f', [self.past] if self.past > 0 else [])
self.lbfgs_lmal = Tensor('f', [self.lbfgs_m])
self.lbfgs_lmys = Tensor('f', [self.lbfgs_m])
self.lbfgs_lms = Tensor('f', [self.nx, self.lbfgs_m])
self.lbfgs_lmy = Tensor('f', [self.nx, self.lbfgs_m])

# 在版本1中忘记保存类型，根据剩余字节数猜测类型
# 猜测 self.type 的类型根据剩余字节数
# 计算 type 0 的大小
size_type_0 = 12 + sum([t.max_storage_size() for t in
                        [self.adam_m, self.adam_v]
                        +([self.adam_pf] if (self.past > 0) else [])])

# 计算 type 1 的大小
size_type_1 = 24 + sum([t.max_storage_size() for t in
                        [self.lbfgs_x, self.lbfgs_xp, self.lbfgs_g,
                         self.lbfgs_gp, self.lbfgs_d, self.lbfgs_pf,
                         self.lbfgs_lmal, self.lbfgs_lmys,
                         self.lbfgs_lms, self.lbfgs_lmy]
                         +([self.lbfgs_pf] if (self.past > 0) else [])])

# 由于对齐填充，大小可能不是精确的，但两种类型的大小差异很大，所以可以选择最接近的大小
remaining = len(data) - offset
if abs(remaining - size_type_0) < abs(remaining - size_type_1):
    self.type = 0
else:
    self.type = 1

# 如果类型为 0，则加载数据到 adam_m
if self.type == 0:
    offset = self.adam_m.load(data, offset)
# 从数据中加载 Adam 算法的参数，并更新偏移量
offset = self.adam_v.load(data, offset)
offset = self.adam_pf.load(data,offset)

# 从数据中解析并加载 Adam 算法的最佳函数值、上一次的函数值和无改进次数，并更新偏移量
self.adam_fx_best          = struct.unpack('<f', bytes(data[offset:offset + 4]))[0];  offset += 4
self.adam_fx_prev          = struct.unpack('<f', bytes(data[offset:offset + 4]))[0];  offset += 4
self.adam_n_no_improvement = struct.unpack('<i', bytes(data[offset:offset + 4]))[0];  offset += 4

# 如果类型为 1，则加载 LBFGS 算法的参数，并更新偏移量
elif self.type == 1:
    offset = self.lbfgs_x.load(data, offset)
    offset = self.lbfgs_xp.load(data, offset)
    offset = self.lbfgs_g.load(data, offset)
    offset = self.lbfgs_gp.load(data, offset)
    offset = self.lbfgs_d.load(data, offset)
    offset = self.lbfgs_pf.load(data, offset)
    offset = self.lbfgs_lmal.load(data, offset)
    offset = self.lbfgs_lmys.load(data, offset)
    offset = self.lbfgs_lms.load(data, offset)
    offset = self.lbfgs_lmy.load(data, offset)

# 从数据中解析并加载 LBFGS 算法的最佳函数值，并更新偏移量
self.lbfgs_fx_best          = struct.unpack('<f', bytes(data[offset:offset + 4]))[0];  offset += 4
# 从数据中解析出 lbfgs_step，并将偏移量增加4
self.lbfgs_step = struct.unpack('<f', bytes(data[offset:offset + 4]))[0]; offset += 4
# 从数据中解析出 lbfgs_j，并将偏移量增加4
self.lbfgs_j = struct.unpack('<i', bytes(data[offset:offset + 4]))[0]; offset += 4
# 从数据中解析出 lbfgs_k，并将偏移量增加4
self.lbfgs_k = struct.unpack('<i', bytes(data[offset:offset + 4]))[0]; offset += 4
# 从数据中解析出 lbfgs_end，并将偏移量增加4
self.lbfgs_end = struct.unpack('<i', bytes(data[offset:offset + 4]))[0]; offset += 4
# 从数据中解析出 lbfgs_n_no_improvement，并将偏移量增加4
self.lbfgs_n_no_improvement = struct.unpack('<i', bytes(data[offset:offset + 4]))[0]; offset += 4

# 如果类型不是上述类型，则抛出异常
else:
    raise ValueError(f"Invalid optimizer type '{self.type}'")

# 返回偏移量
return offset

# 将优化器的信息保存到 gguf_writer 中
def save_gguf(self, gguf_writer):
    # 添加文件版本信息
    gguf_writer.add_uint32(LLM_KV_OPTIMIZER_FILE_VERSION, 0)
    # 添加收敛过去次数信息
    gguf_writer.add_uint32(LLM_KV_OPTIMIZER_CONVERGENCE_PAST_COUNT, self.past)
    # 添加参数数量信息
    gguf_writer.add_uint64(LLM_KV_OPTIMIZER_PARAMETER_COUNT, self.nx)
    # 添加迭代次数信息
    gguf_writer.add_uint32(LLM_KV_OPTIMIZER_ITERATION_COUNT, self.iter)
    # 添加刚初始化信息
    gguf_writer.add_bool(LLM_KV_OPTIMIZER_JUST_INITIALIZED, self.just_initialized)

    # 如果类型为0，则添加类型信息为ADAM
    if self.type == 0:
        gguf_writer.add_string(LLM_KV_OPTIMIZER_TYPE, LLM_KV_OPTIMIZER_TYPE_ADAM)
# 将 self.adam_fx_best 添加为 float32 类型的键值对到 gguf_writer 中
gguf_writer.add_float32(LLM_KV_OPTIMIZER_ADAM_BEST_LOSS, self.adam_fx_best)
# 将 self.adam_fx_prev 添加为 float32 类型的键值对到 gguf_writer 中
gguf_writer.add_float32(LLM_KV_OPTIMIZER_ADAM_PREVIOUS_LOSS, self.adam_fx_prev)
# 将 self.adam_n_no_improvement 添加为 uint32 类型的键值对到 gguf_writer 中
gguf_writer.add_uint32(LLM_KV_OPTIMIZER_ADAM_NO_IMPROVEMENT_COUNT, self.adam_n_no_improvement)

# 将 self.adam_m 保存到 gguf_writer 中，键名为 LLM_TENSOR_OPTIMIZER_ADAM_FIRST_MOMENTS
self.adam_m.save_gguf(gguf_writer, name=LLM_TENSOR_OPTIMIZER_ADAM_FIRST_MOMENTS)
# 将 self.adam_v 保存到 gguf_writer 中，键名为 LLM_TENSOR_OPTIMIZER_ADAM_SECOND_MOMENTS
self.adam_v.save_gguf(gguf_writer, name=LLM_TENSOR_OPTIMIZER_ADAM_SECOND_MOMENTS)
# 如果 self.past 大于 0，则将 self.adam_pf 保存到 gguf_writer 中，键名为 LLM_TENSOR_OPTIMIZER_ADAM_PAST_LOSS_VALUES
if self.past > 0:
    self.adam_pf.save_gguf(gguf_writer, name=LLM_TENSOR_OPTIMIZER_ADAM_PAST_LOSS_VALUES)

# 如果 self.type 等于 1，则添加优化器类型为 L-BFGS 到 gguf_writer 中
gguf_writer.add_string(LLM_KV_OPTIMIZER_TYPE, LLM_KV_OPTIMIZER_TYPE_LBFGS)
# 将 self.lbfgs_m 添加为 uint32 类型的键值对到 gguf_writer 中
gguf_writer.add_uint32(LLM_KV_OPTIMIZER_LBFGS_APPROX_HESSIAN_COUNT, self.lbfgs_m)
# 将 self.lbfgs_fx_best 添加为 float32 类型的键值对到 gguf_writer 中
gguf_writer.add_float32(LLM_KV_OPTIMIZER_LBFGS_BEST_LOSS, self.lbfgs_fx_best)
# 将 self.lbfgs_step 添加为 float32 类型的键值对到 gguf_writer 中
gguf_writer.add_float32(LLM_KV_OPTIMIZER_LBFGS_LINE_SEARCH_STEP, self.lbfgs_step)
# 将 self.lbfgs_j 添加为 int32 类型的键值对到 gguf_writer 中
gguf_writer.add_int32(LLM_KV_OPTIMIZER_LBFGS_LINE_SEARCH_J, self.lbfgs_j)
# 将 self.lbfgs_k 添加为 int32 类型的键值对到 gguf_writer 中
gguf_writer.add_int32(LLM_KV_OPTIMIZER_LBFGS_LINE_SEARCH_K, self.lbfgs_k)
# 将 self.lbfgs_end 添加为 int32 类型的键值对到 gguf_writer 中
gguf_writer.add_int32(LLM_KV_OPTIMIZER_LBFGS_LINE_SEARCH_END, self.lbfgs_end)
# 将 self.lbfgs_n_no_improvement 添加为 uint32 类型的键值对到 gguf_writer 中
gguf_writer.add_uint32(LLM_KV_OPTIMIZER_LBFGS_NO_IMPROVEMENT_COUNT, self.lbfgs_n_no_improvement)

# 将 self.lbfgs_x 保存到 gguf_writer 中，键名为 LLM_TENSOR_OPTIMIZER_LBFGS_CURRENT_PARAMETERS
self.lbfgs_x.save_gguf(gguf_writer, name=LLM_TENSOR_OPTIMIZER_LBFGS_CURRENT_PARAMETERS)
# 保存 lbfgs_xp 的数据到 gguf 文件中，使用指定的名称
self.lbfgs_xp.save_gguf(gguf_writer, name=LLM_TENSOR_OPTIMIZER_LBFGS_PREVIOUS_PARAMETERS)
# 保存 lbfgs_g 的数据到 gguf 文件中，使用指定的名称
self.lbfgs_g.save_gguf(gguf_writer, name=LLM_TENSOR_OPTIMIZER_LBFGS_CURRENT_GRADIENTS)
# 保存 lbfgs_gp 的数据到 gguf 文件中，使用指定的名称
self.lbfgs_gp.save_gguf(gguf_writer, name=LLM_TENSOR_OPTIMIZER_LBFGS_PREVIOUS_GRADIENTS)
# 保存 lbfgs_d 的数据到 gguf 文件中，使用指定的名称
self.lbfgs_d.save_gguf(gguf_writer, name=LLM_TENSOR_OPTIMIZER_LBFGS_SEARCH_DIRECTION)
# 如果 past 大于 0，则保存 lbfgs_pf 的数据到 gguf 文件中，使用指定的名称
if self.past > 0:
    self.lbfgs_pf.save_gguf(gguf_writer, name=LLM_TENSOR_OPTIMIZER_LBFGS_PAST_LOSS_VALUES)
# 保存 lbfgs_lmal 的数据到 gguf 文件中，使用指定的名称
self.lbfgs_lmal.save_gguf(gguf_writer, name=LLM_TENSOR_OPTIMIZER_LBFGS_MEMORY_ALPHA)
# 保存 lbfgs_lmys 的数据到 gguf 文件中，使用指定的名称
self.lbfgs_lmys.save_gguf(gguf_writer, name=LLM_TENSOR_OPTIMIZER_LBFGS_MEMORY_YS)
# 保存 lbfgs_lms 的数据到 gguf 文件中，使用指定的名称
self.lbfgs_lms.save_gguf(gguf_writer, name=LLM_TENSOR_OPTIMIZER_LBFGS_MEMORY_S)
# 保存 lbfgs_lmy 的数据到 gguf 文件中，使用指定的名称
self.lbfgs_lmy.save_gguf(gguf_writer, name=LLM_TENSOR_OPTIMIZER_LBFGS_MEMORY_Y)
# 如果不满足条件，则抛出 ValueError 异常
else:
    raise ValueError('Unknown optimizer type')

# 定义 LoraParams 类
class LoraParams:
    # 初始化方法
    def __init__(self):
        pass

    # 加载数据的方法
    def load(self, data, offset):
        # 从指定偏移位置解包数据，并赋值给对应的属性
        self.n_rank_attention_norm  = struct.unpack('<I', bytes(data[offset:offset + 4]))[0];  offset += 4
        self.n_rank_wq              = struct.unpack('<I', bytes(data[offset:offset + 4]))[0];  offset += 4
# 从数据中解包一个无符号整数，使用小端字节顺序，将结果赋值给 self.n_rank_wk，然后增加偏移量4
self.n_rank_wk = struct.unpack('<I', bytes(data[offset:offset + 4]))[0];  offset += 4
# 从数据中解包一个无符号整数，使用小端字节顺序，将结果赋值给 self.n_rank_wv，然后增加偏移量4
self.n_rank_wv = struct.unpack('<I', bytes(data[offset:offset + 4]))[0];  offset += 4
# 从数据中解包一个无符号整数，使用小端字节顺序，将结果赋值给 self.n_rank_wo，然后增加偏移量4
self.n_rank_wo = struct.unpack('<I', bytes(data[offset:offset + 4]))[0];  offset += 4
# 从数据中解包一个无符号整数，使用小端字节顺序，将结果赋值给 self.n_rank_ffn_norm，然后增加偏移量4
self.n_rank_ffn_norm = struct.unpack('<I', bytes(data[offset:offset + 4]))[0];  offset += 4
# 从数据中解包一个无符号整数，使用小端字节顺序，将结果赋值给 self.n_rank_w1，然后增加偏移量4
self.n_rank_w1 = struct.unpack('<I', bytes(data[offset:offset + 4]))[0];  offset += 4
# 从数据中解包一个无符号整数，使用小端字节顺序，将结果赋值给 self.n_rank_w2，然后增加偏移量4
self.n_rank_w2 = struct.unpack('<I', bytes(data[offset:offset + 4]))[0];  offset += 4
# 从数据中解包一个无符号整数，使用小端字节顺序，将结果赋值给 self.n_rank_w3，然后增加偏移量4
self.n_rank_w3 = struct.unpack('<I', bytes(data[offset:offset + 4]))[0];  offset += 4
# 从数据中解包一个无符号整数，使用小端字节顺序，将结果赋值给 self.n_rank_tok_embeddings，然后增加偏移量4
self.n_rank_tok_embeddings = struct.unpack('<I', bytes(data[offset:offset + 4]))[0];  offset += 4
# 从数据中解包一个无符号整数，使用小端字节顺序，将结果赋值给 self.n_rank_norm，然后增加偏移量4
self.n_rank_norm = struct.unpack('<I', bytes(data[offset:offset + 4]))[0];  offset += 4
# 从数据中解包一个无符号整数，使用小端字节顺序，将结果赋值给 self.n_rank_output，然后增加偏移量4
self.n_rank_output = struct.unpack('<I', bytes(data[offset:offset + 4]))[0];  offset += 4
# 返回偏移量
return offset

# 将 self.n_rank_tok_embeddings 写入 gguf_writer
gguf_writer.add_uint32(LLM_KV_TRAINING_LORA_RANK_TOKEN_EMBD,  self.n_rank_tok_embeddings)
# 将 self.n_rank_norm 写入 gguf_writer
gguf_writer.add_uint32(LLM_KV_TRAINING_LORA_RANK_OUTPUT_NORM, self.n_rank_norm)
# 将 self.n_rank_output 写入 gguf_writer
gguf_writer.add_uint32(LLM_KV_TRAINING_LORA_RANK_OUTPUT,      self.n_rank_output)
# 将 self.n_rank_attention_norm 写入 gguf_writer
gguf_writer.add_uint32(LLM_KV_TRAINING_LORA_RANK_ATTN_NORM,   self.n_rank_attention_norm)
# 将 self.n_rank_wq 写入 gguf_writer
gguf_writer.add_uint32(LLM_KV_TRAINING_LORA_RANK_ATTN_Q,      self.n_rank_wq)
# 将 self.n_rank_wk 写入 gguf_writer
gguf_writer.add_uint32(LLM_KV_TRAINING_LORA_RANK_ATTN_K,      self.n_rank_wk)
# 将 self.n_rank_wv 写入 gguf_writer
gguf_writer.add_uint32(LLM_KV_TRAINING_LORA_RANK_ATTN_V,      self.n_rank_wv)
# 将self.n_rank_wo添加到gguf_writer中，使用LLM_KV_TRAINING_LORA_RANK_ATTN_OUT作为键
gguf_writer.add_uint32(LLM_KV_TRAINING_LORA_RANK_ATTN_OUT,    self.n_rank_wo)
# 将self.n_rank_ffn_norm添加到gguf_writer中，使用LLM_KV_TRAINING_LORA_RANK_FFN_NORM作为键
gguf_writer.add_uint32(LLM_KV_TRAINING_LORA_RANK_FFN_NORM,    self.n_rank_ffn_norm)
# 将self.n_rank_w1添加到gguf_writer中，使用LLM_KV_TRAINING_LORA_RANK_FFN_GATE作为键
gguf_writer.add_uint32(LLM_KV_TRAINING_LORA_RANK_FFN_GATE,    self.n_rank_w1)
# 将self.n_rank_w2添加到gguf_writer中，使用LLM_KV_TRAINING_LORA_RANK_FFN_DOWN作为键
gguf_writer.add_uint32(LLM_KV_TRAINING_LORA_RANK_FFN_DOWN,    self.n_rank_w2)
# 将self.n_rank_w3添加到gguf_writer中，使用LLM_KV_TRAINING_LORA_RANK_FFN_UP作为键
gguf_writer.add_uint32(LLM_KV_TRAINING_LORA_RANK_FFN_UP,      self.n_rank_w3)

# 定义ModelParams类
class ModelParams:
    # 初始化方法，设置n_ff属性
    def __init__(self, n_ff = None):
        self.n_ff = n_ff

    # 加载方法，从data中读取参数并设置到对象中
    def load(self, data, offset):
        # 从data中读取n_vocab并设置到对象中，然后更新offset
        self.n_vocab = struct.unpack('<I', bytes(data[offset:offset + 4]))[0];  offset += 4
        # 从data中读取n_embd并设置到对象中，然后更新offset
        self.n_embd  = struct.unpack('<I', bytes(data[offset:offset + 4]))[0];  offset += 4
        # 从data中读取n_mult并设置到对象中，然后更新offset
        self.n_mult  = struct.unpack('<I', bytes(data[offset:offset + 4]))[0];  offset += 4
        # 从data中读取n_head并设置到对象中，然后更新offset
        self.n_head  = struct.unpack('<I', bytes(data[offset:offset + 4]))[0];  offset += 4
        # 从data中读取n_layer并设置到对象中，然后更新offset
        self.n_layer = struct.unpack('<I', bytes(data[offset:offset + 4]))[0];  offset += 4
        # 从data中读取n_rot并设置到对象中，然后更新offset
        self.n_rot   = struct.unpack('<I', bytes(data[offset:offset + 4]))[0];  offset += 4
        # 返回更新后的offset
        return offset

    # 获取n_ff属性的方法
    def get_n_ff(self):
        if self.n_ff is None:
            # 如果 self.n_ff 为空，则根据公式计算并返回一个值
            # 这个值是根据模型参数计算得出的 feed forward 层的大小
            return ((2*(4*self.n_embd)//3 + self.n_mult - 1)//self.n_mult)*self.n_mult
        else:
            # 如果 self.n_ff 不为空，则直接返回其值
            return self.n_ff

    def save_gguf(self, gguf_writer):
        # 保存 gguf 文件时，不保存 self.n_vocab
        # 将模型参数添加到 gguf_writer 中
        gguf_writer.add_embedding_length(self.n_embd)
        gguf_writer.add_head_count(self.n_head)
        gguf_writer.add_block_count(self.n_layer)
        gguf_writer.add_rope_dimension_count(self.n_rot)
        gguf_writer.add_feed_forward_length(self.get_n_ff())

def tensor_name(key, bid=None, suffix=".weight"):
    # 根据 key 和 bid 构建张量的名称，并添加后缀
    return gguf.TENSOR_NAMES[key].format(bid=bid) + suffix

class Layer:
    def __init__(self, params, lora_params, bid):
        # 初始化 Layer 类时，设置 bid 属性
        self.bid = bid
# 初始化各种张量，用于存储模型参数
self.att_norm_a = Tensor('f', [lora_params.n_rank_attention_norm, params.n_embd])  # 存储注意力层的参数
self.att_norm_b = Tensor('f', [lora_params.n_rank_attention_norm, 1])  # 存储注意力层的参数
self.wq_a       = Tensor('f', [lora_params.n_rank_wq, params.n_embd])  # 存储查询权重的参数
self.wq_b       = Tensor('f', [lora_params.n_rank_wq, params.n_embd])  # 存储查询权重的参数
self.wk_a       = Tensor('f', [lora_params.n_rank_wk, params.n_embd])  # 存储键权重的参数
self.wk_b       = Tensor('f', [lora_params.n_rank_wk, params.n_embd])  # 存储键权重的参数
self.wv_a       = Tensor('f', [lora_params.n_rank_wv, params.n_embd])  # 存储值权重的参数
self.wv_b       = Tensor('f', [lora_params.n_rank_wv, params.n_embd])  # 存储值权重的参数
self.wo_a       = Tensor('f', [lora_params.n_rank_wo, params.n_embd])  # 存储输出权重的参数
self.wo_b       = Tensor('f', [lora_params.n_rank_wo, params.n_embd])  # 存储输出权重的参数
self.ffn_norm_a = Tensor('f', [lora_params.n_rank_ffn_norm, params.n_embd])  # 存储前馈神经网络层的参数
self.ffn_norm_b = Tensor('f', [lora_params.n_rank_ffn_norm, 1])  # 存储前馈神经网络层的参数
self.w1_a       = Tensor('f', [lora_params.n_rank_w1, params.n_embd])  # 存储第一个全连接层的参数
self.w1_b       = Tensor('f', [lora_params.n_rank_w1, params.get_n_ff()])  # 存储第一个全连接层的参数
self.w2_a       = Tensor('f', [lora_params.n_rank_w2, params.get_n_ff()])  # 存储第二个全连接层的参数
self.w2_b       = Tensor('f', [lora_params.n_rank_w2, params.n_embd])  # 存储第二个全连接层的参数
self.w3_a       = Tensor('f', [lora_params.n_rank_w3, params.n_embd])  # 存储第三个全连接层的参数
self.w3_b       = Tensor('f', [lora_params.n_rank_w3, params.get_n_ff()])  # 存储第三个全连接层的参数

# 加载模型参数
def load(self, data, offset):
# 从数据中加载 self.att_norm_a 的内容，并更新偏移量
offset = self.att_norm_a.load(data, offset)
# 从数据中加载 self.att_norm_b 的内容，并更新偏移量
offset = self.att_norm_b.load(data, offset)
# 从数据中加载 self.wq_a 的内容，并更新偏移量
offset = self.wq_a.load(data, offset)
# 从数据中加载 self.wq_b 的内容，并更新偏移量
offset = self.wq_b.load(data, offset)
# 从数据中加载 self.wk_a 的内容，并更新偏移量
offset = self.wk_a.load(data, offset)
# 从数据中加载 self.wk_b 的内容，并更新偏移量
offset = self.wk_b.load(data, offset)
# 从数据中加载 self.wv_a 的内容，并更新偏移量
offset = self.wv_a.load(data, offset)
# 从数据中加载 self.wv_b 的内容，并更新偏移量
offset = self.wv_b.load(data, offset)
# 从数据中加载 self.wo_a 的内容，并更新偏移量
offset = self.wo_a.load(data, offset)
# 从数据中加载 self.wo_b 的内容，并更新偏移量
offset = self.wo_b.load(data, offset)
# 从数据中加载 self.ffn_norm_a 的内容，并更新偏移量
offset = self.ffn_norm_a.load(data, offset)
# 从数据中加载 self.ffn_norm_b 的内容，并更新偏移量
offset = self.ffn_norm_b.load(data, offset)
# 从数据中加载 self.w1_a 的内容，并更新偏移量
offset = self.w1_a.load(data, offset)
# 从数据中加载 self.w1_b 的内容，并更新偏移量
offset = self.w1_b.load(data, offset)
# 从数据中加载 self.w2_a 的内容，并更新偏移量
offset = self.w2_a.load(data, offset)
# 从数据中加载 self.w2_b 的内容，并更新偏移量
offset = self.w2_b.load(data, offset)
# 从数据中加载 self.w3_a 的内容，并更新偏移量
offset = self.w3_a.load(data, offset)
# 从数据中加载 self.w3_b 的内容，并更新偏移量
offset = self.w3_b.load(data, offset)
# 返回最终的偏移量
return offset
# 保存模型参数到 GGUF 文件中，每个参数都有对应的名称和路径
def save_gguf(self, gguf_writer):
    # 保存 self.att_norm_a 参数到 GGUF 文件中
    self.att_norm_a.save_gguf(gguf_writer, name=tensor_name(gguf.MODEL_TENSOR.ATTN_NORM, self.bid, ".weight.lora_a"))
    # 保存 self.att_norm_b 参数到 GGUF 文件中
    self.att_norm_b.save_gguf(gguf_writer, name=tensor_name(gguf.MODEL_TENSOR.ATTN_NORM, self.bid, ".weight.lora_b"))
    # 保存 self.wq_a 参数到 GGUF 文件中
    self.wq_a.save_gguf(gguf_writer, name=tensor_name(gguf.MODEL_TENSOR.ATTN_Q, self.bid, ".weight.lora_a"))
    # 保存 self.wq_b 参数到 GGUF 文件中
    self.wq_b.save_gguf(gguf_writer, name=tensor_name(gguf.MODEL_TENSOR.ATTN_Q, self.bid, ".weight.lora_b"))
    # 保存 self.wk_a 参数到 GGUF 文件中
    self.wk_a.save_gguf(gguf_writer, name=tensor_name(gguf.MODEL_TENSOR.ATTN_K, self.bid, ".weight.lora_a"))
    # 保存 self.wk_b 参数到 GGUF 文件中
    self.wk_b.save_gguf(gguf_writer, name=tensor_name(gguf.MODEL_TENSOR.ATTN_K, self.bid, ".weight.lora_b"))
    # 保存 self.wv_a 参数到 GGUF 文件中
    self.wv_a.save_gguf(gguf_writer, name=tensor_name(gguf.MODEL_TENSOR.ATTN_V, self.bid, ".weight.lora_a"))
    # 保存 self.wv_b 参数到 GGUF 文件中
    self.wv_b.save_gguf(gguf_writer, name=tensor_name(gguf.MODEL_TENSOR.ATTN_V, self.bid, ".weight.lora_b"))
    # 保存 self.wo_a 参数到 GGUF 文件中
    self.wo_a.save_gguf(gguf_writer, name=tensor_name(gguf.MODEL_TENSOR.ATTN_OUT, self.bid, ".weight.lora_a"))
    # 保存 self.wo_b 参数到 GGUF 文件中
    self.wo_b.save_gguf(gguf_writer, name=tensor_name(gguf.MODEL_TENSOR.ATTN_OUT, self.bid, ".weight.lora_b"))
    # 保存 self.ffn_norm_a 参数到 GGUF 文件中
    self.ffn_norm_a.save_gguf(gguf_writer, name=tensor_name(gguf.MODEL_TENSOR.FFN_NORM, self.bid, ".weight.lora_a"))
    # 保存 self.ffn_norm_b 参数到 GGUF 文件中
    self.ffn_norm_b.save_gguf(gguf_writer, name=tensor_name(gguf.MODEL_TENSOR.FFN_NORM, self.bid, ".weight.lora_b"))
    # 保存 self.w1_a 参数到 GGUF 文件中
    self.w1_a.save_gguf(gguf_writer, name=tensor_name(gguf.MODEL_TENSOR.FFN_GATE, self.bid, ".weight.lora_a"))
    # 保存 self.w1_b 参数到 GGUF 文件中
    self.w1_b.save_gguf(gguf_writer, name=tensor_name(gguf.MODEL_TENSOR.FFN_GATE, self.bid, ".weight.lora_b"))
    # 保存 self.w2_a 参数到 GGUF 文件中
    self.w2_a.save_gguf(gguf_writer, name=tensor_name(gguf.MODEL_TENSOR.FFN_DOWN, self.bid, ".weight.lora_a"))
    # 保存 self.w2_b 参数到 GGUF 文件中
    self.w2_b.save_gguf(gguf_writer, name=tensor_name(gguf.MODEL_TENSOR.FFN_DOWN, self.bid, ".weight.lora_b"))
    # 保存 self.w3_a 参数到 GGUF 文件中
    self.w3_a.save_gguf(gguf_writer, name=tensor_name(gguf.MODEL_TENSOR.FFN_UP, self.bid, ".weight.lora_a"))
    # 保存 self.w3_b 参数到 GGUF 文件中
    self.w3_b.save_gguf(gguf_writer, name=tensor_name(gguf.MODEL_TENSOR.FFN_UP, self.bid, ".weight.lora_b"))
# 定义一个名为 LoraModel 的类
class LoraModel:
    # 初始化方法，参数 n_ff 默认为 None
    def __init__(self, n_ff = None):
        # 创建 ModelParams 对象，并将其赋值给 self.params
        self.params = ModelParams(n_ff = n_ff)
        # 创建 LoraParams 对象，并将其赋值给 self.lora_params
        self.lora_params = LoraParams()
        # 创建一个空的列表，用于存储层的信息
        self.layers = []

    # 加载方法，接受 data 和 offset 作为参数
    def load(self, data, offset):
        # 调用 self.params 的 load 方法，并更新 offset
        offset = self.params.load(data, offset)
        # 调用 self.lora_params 的 load 方法，并更新 offset
        offset = self.lora_params.load(data, offset)

        # 创建名为 tok_embd_a 的 Tensor 对象，指定数据类型和形状
        self.tok_embd_a = Tensor('f', [self.lora_params.n_rank_tok_embeddings, self.params.n_embd])
        # 创建名为 tok_embd_b 的 Tensor 对象，指定数据类型和形状
        self.tok_embd_b = Tensor('f', [self.lora_params.n_rank_tok_embeddings, self.params.n_vocab])
        # 创建名为 norm_a 的 Tensor 对象，指定数据类型和形状
        self.norm_a     = Tensor('f', [self.lora_params.n_rank_norm, self.params.n_embd])
        # 创建名为 norm_b 的 Tensor 对象，指定数据类型和形状
        self.norm_b     = Tensor('f', [self.lora_params.n_rank_norm, 1])
        # 创建名为 output_a 的 Tensor 对象，指定数据类型和形状
        self.output_a   = Tensor('f', [self.lora_params.n_rank_output, self.params.n_embd])
        # 创建名为 output_b 的 Tensor 对象，指定数据类型和形状
        self.output_b   = Tensor('f', [self.lora_params.n_rank_output, self.params.n_vocab])

        # 调用各个 Tensor 对象的 load 方法，并更新 offset
        offset = self.tok_embd_a.load(data, offset)
        offset = self.tok_embd_b.load(data, offset)
        offset = self.norm_a.load(data, offset)
# 从数据中加载偏移量，并将其存储到self.norm_b中
offset = self.norm_b.load(data, offset)
# 从数据中加载偏移量，并将其存储到self.output_a中
offset = self.output_a.load(data, offset)
# 从数据中加载偏移量，并将其存储到self.output_b中
offset = self.output_b.load(data, offset)

# 清空self.layers列表
self.layers.clear()
# 遍历self.params.n_layer次，创建Layer对象并加载数据，然后将其添加到self.layers列表中
for bid in range(self.params.n_layer):
    layer = Layer(self.params, self.lora_params, bid)
    offset = layer.load(data, offset)
    self.layers.append(layer)

# 返回偏移量
return offset

# 将self.params和self.lora_params保存到gguf_writer中
self.params.save_gguf(gguf_writer)
self.lora_params.save_gguf(gguf_writer)

# 将self.tok_embd_a和self.tok_embd_b保存到gguf_writer中，并指定名称
self.tok_embd_a.save_gguf(gguf_writer, name=tensor_name(gguf.MODEL_TENSOR.TOKEN_EMBD,  suffix=".weight.lora_a"))
self.tok_embd_b.save_gguf(gguf_writer, name=tensor_name(gguf.MODEL_TENSOR.TOKEN_EMBD,  suffix=".weight.lora_b"))
# 将self.norm_a和self.norm_b保存到gguf_writer中，并指定名称
self.norm_a.save_gguf(gguf_writer, name=tensor_name(gguf.MODEL_TENSOR.OUTPUT_NORM, suffix=".weight.lora_a"))
self.norm_b.save_gguf(gguf_writer, name=tensor_name(gguf.MODEL_TENSOR.OUTPUT_NORM, suffix=".weight.lora_b"))
# 调用output_a对象的save_gguf方法，将gguf_writer和文件名作为参数传入
self.output_a.save_gguf(gguf_writer, name=tensor_name(gguf.MODEL_TENSOR.OUTPUT, suffix=".weight.lora_a"))
# 调用output_b对象的save_gguf方法，将gguf_writer和文件名作为参数传入
self.output_b.save_gguf(gguf_writer, name=tensor_name(gguf.MODEL_TENSOR.OUTPUT, suffix=".weight.lora_b"))

# 遍历self.layers列表中的每个元素，调用其save_gguf方法，传入gguf_writer作为参数
for layer in self.layers:
    layer.save_gguf(gguf_writer)

# 定义LoraCheckpoint类
class LoraCheckpoint:
    # 初始化方法，创建LoraModel对象和OptimizationContext对象
    def __init__(self, n_ff = None):
        self.model = LoraModel(n_ff = n_ff)
        self.opt_ctx = OptimizationContext()

    # 加载方法，从给定数据和偏移量读取文件头信息
    def load(self, data, offset):
        # 读取文件头的魔数
        magic = bytes(reversed(data[offset:offset + 4])); offset += 4
        # 检查魔数是否为'ggcl'，如果不是则抛出异常
        if magic != b'ggcl':
            raise ValueError(f"File header magic indicates, that this is no finetune-lora checkpoint file. Expected 'ggcl', Got '{str(magic)}'")

        # 读取文件版本号
        self.version = struct.unpack('<I', bytes(data[offset:offset + 4]))[0]; offset += 4
        # 检查文件版本号是否为0，如果不是则抛出异常
        if self.version != 0:
            raise ValueError('Invalid version of checkpoint file')
# 从数据中解包出训练迭代次数，然后更新偏移量
self.train_its = struct.unpack('<I', bytes(data[offset:offset + 4]))[0]; offset += 4
# 从数据中解包出训练样本数，然后更新偏移量
self.train_samples = struct.unpack('<I', bytes(data[offset:offset + 4]))[0]; offset += 4
# 从数据中解包出训练标记数，然后更新偏移量
self.train_tokens = struct.unpack('<I', bytes(data[offset:offset + 4]))[0]; offset += 4

# 调用模型对象的 load 方法，加载模型数据，更新偏移量
offset = self.model.load(data, offset)
# 调用优化上下文对象的 load 方法，加载优化上下文数据，更新偏移量
offset = self.opt_ctx.load(data, offset)

# 返回更新后的偏移量
return offset

# 保存 GGUF 文件
def save_gguf(self, gguf_writer):
    # 向 GGUF 写入文件类型
    gguf_writer.add_file_type(gguf.GGMLQuantizationType.F32)
    # 向 GGUF 写入层归一化的 RMS 误差
    gguf_writer.add_layer_norm_rms_eps(1e-5)
    # 向 GGUF 写入训练文件版本号
    gguf_writer.add_uint32(LLM_KV_TRAINING_FILE_VERSION, 0)
    # 向 GGUF 写入训练类型
    gguf_writer.add_string(LLM_KV_TRAINING_TYPE, LLM_KV_TRAINING_TYPE_FINETUNE_LORA)
    # 向 GGUF 写入训练迭代次数
    gguf_writer.add_uint32(LLM_KV_TRAINING_ITERATION_COUNT, self.train_its)
    # 向 GGUF 写入训练样本数
    gguf_writer.add_uint32(LLM_KV_TRAINING_SAMPLE_COUNT, self.train_samples)
    # 向 GGUF 写入训练标记数
    gguf_writer.add_uint32(LLM_KV_TRAINING_TOKEN_COUNT, self.train_tokens)
    # 调用模型对象的 save_gguf 方法，保存模型数据到 GGUF
    self.model.save_gguf(gguf_writer)
    # 调用优化上下文对象的 save_gguf 方法，保存优化上下文数据到 GGUF
    self.opt_ctx.save_gguf(gguf_writer)
# 处理命令行参数，返回参数配置
def handle_args():
    # 创建参数解析器，设置描述信息
    parser = argparse.ArgumentParser(description = 'Convert finetune checkpoints to GGUF')
    # 添加输入参数选项，包括参数名、类型、帮助信息和是否必须
    parser.add_argument('--input',  '-i', type = Path, help = 'Input finetune checkpoint filename', required=True)
    # 添加输出参数选项，包括参数名、类型、帮助信息和是否必须
    parser.add_argument('--output', '-o', type = Path, help = 'Output GGUF filename', required=True)
    # 解析命令行参数并返回
    return parser.parse_args()

# 主函数
def main():
    # 处理命令行参数，获取配置
    cfg = handle_args()
    # 打印配置信息
    print(cfg)
    # 从文件中创建内存映射，只读模式
    data = np.memmap(cfg.input, mode = 'r')
    # 创建 LoraCheckpoint 对象
    chk = LoraCheckpoint(n_ff = cfg.ff)
    # 设置偏移量为 0，加载数据到 LoraCheckpoint 对象中
    offset = 0
    offset = chk.load(data, offset)
    # 断言，应该已经读取了所有可用数据
    assert(offset == len(data))

    # 创建 GGUFWriter 对象，设置输出文件名和模型架构
    gguf_writer = gguf.GGUFWriter(cfg.output, gguf.MODEL_ARCH_NAMES[gguf.MODEL_ARCH.LLAMA], use_temp_file = False)
    # 将 LoraCheckpoint 对象保存为 GGUF 格式
    chk.save_gguf(gguf_writer)
    # 打印信息，写入 GGUF 头部
    print("    gguf: write header")
    gguf_writer.write_header_to_file()
# 打印提示信息，表示开始写入元数据
print("    gguf: write metadata")
# 调用 gguf_writer 对象的方法，将键值数据写入文件
gguf_writer.write_kv_data_to_file()
# 打印提示信息，表示开始写入张量数据
print("    gguf: write tensors")
# 调用 gguf_writer 对象的方法，将张量数据写入文件
gguf_writer.write_tensors_to_file()
# 关闭 gguf_writer 对象
gguf_writer.close()

# 如果当前脚本为主程序，则执行 main() 函数
if __name__ == '__main__':
    main()
```