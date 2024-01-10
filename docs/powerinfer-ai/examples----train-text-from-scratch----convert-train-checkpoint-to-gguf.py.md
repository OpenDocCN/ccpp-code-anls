# `PowerInfer\examples\train-text-from-scratch\convert-train-checkpoint-to-gguf.py`

```
#!/usr/bin/env python3
# 设置脚本的解释器为 Python3

# 导入命令行参数解析模块
import argparse
# 导入操作系统相关的模块
import os
# 导入用于处理二进制数据的模块
import struct
# 导入系统相关的模块
import sys
# 导入处理数组和矩阵的模块
import numpy as np
# 导入处理文件路径的模块
from pathlib import Path

# 如果环境变量中没有设置 NO_LOCAL_GGUF，则将 gguf-py 目录添加到系统路径中
if 'NO_LOCAL_GGUF' not in os.environ:
    sys.path.insert(1, str(Path(__file__).parent / '..' / '..' / 'gguf-py'))
import gguf

# gguf 相关的常量定义
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
# 定义一系列的常量，用于表示优化器的参数和状态
LLM_TENSOR_OPTIMIZER_LBFGS_PREVIOUS_PARAMETERS = "optimizer.lbfgs.previous_parameters"
LLM_TENSOR_OPTIMIZER_LBFGS_CURRENT_GRADIENTS   = "optimizer.lbfgs.current_gradients"
LLM_TENSOR_OPTIMIZER_LBFGS_PREVIOUS_GRADIENTS  = "optimizer.lbfgs.previous_gradients"
LLM_TENSOR_OPTIMIZER_LBFGS_SEARCH_DIRECTION    = "optimizer.lbfgs.search_direction"
LLM_TENSOR_OPTIMIZER_LBFGS_PAST_LOSS_VALUES    = "optimizer.lbfgs.past_loss_values"
LLM_TENSOR_OPTIMIZER_LBFGS_MEMORY_ALPHA        = "optimizer.lbfgs.memory_alpha"
LLM_TENSOR_OPTIMIZER_LBFGS_MEMORY_YS           = "optimizer.lbfgs.memory_ys"
LLM_TENSOR_OPTIMIZER_LBFGS_MEMORY_S            = "optimizer.lbfgs.memory_s"
LLM_TENSOR_OPTIMIZER_LBFGS_MEMORY_Y            = "optimizer.lbfgs.memory_y"

# 定义一系列的常量，用于表示训练类型和相关信息
LLM_KV_TRAINING_TYPE_TRAIN_MODEL   = "train_model"
LLM_KV_TRAINING_TYPE_FINETUNE_LORA = "finetune_lora"
LLM_KV_TRAINING_TYPE               = "training.type"
LLM_KV_TRAINING_FILE_VERSION       = "training.file_version"
LLM_KV_TRAINING_ITERATION_COUNT    = "training.iteration_count"
LLM_KV_TRAINING_SAMPLE_COUNT       = "training.sample_count"
LLM_KV_TRAINING_TOKEN_COUNT        = "training.token_count"

# 定义一个名为Tensor的类
class Tensor:
    # 初始化方法，接受数据类型和ne参数
    def __init__(self, dtype='f', ne=None):
        # 如果ne参数为None，则设置为一个空列表
        if ne is None:
            ne = []
        # 设置类的属性
        self.dtype = dtype
        self.ne = ne
        self.nbytes = 0
        # 如果数据类型为'f'
        if self.dtype == 'f':
            # 如果ne列表为空，则nbytes为0
            if len(self.ne) == 0:
                self.nbytes = 0
            # 否则，计算ne列表中元素的乘积并乘以4，赋值给nbytes
            else:
                self.nbytes = int(np.product(self.ne)) * 4
        # 如果数据类型不是'f'，则抛出数值错误异常
        else:
            raise ValueError(f"Unhandled data type '{self.dtype}'")
    # 从给定数据和偏移量加载数据
    def load(self, data, offset):
        # 从数据中解包出nd（数据数量），并更新偏移量
        nd = struct.unpack('<I', bytes(data[offset:offset + 4]))[0]; offset += 4
        # 从数据中解包出namelen（名称长度），并更新偏移量
        namelen = struct.unpack('<I', bytes(data[offset:offset + 4]))[0]; offset += 4
        # 从数据中解包出dtype（数据类型），并更新偏移量
        dtype = struct.unpack('<I', bytes(data[offset:offset + 4]))[0]; offset += 4

        # 断言数据数量与self.ne的长度相等
        assert(nd == len(self.ne))
        # 创建空列表ne
        ne = []
        # 遍历数据数量次
        for d in range(nd):
            # 从数据中解包出n，并更新偏移量
            n = struct.unpack('<I', bytes(data[offset:offset + 4]))[0]; offset += 4
            # 将n添加到ne列表中
            ne.append(n)

        # 断言ne元组与self.ne元组相等
        assert(tuple(ne) == tuple(self.ne))

        # 如果数据类型为'f'，则断言dtype为0；否则引发异常
        if self.dtype == 'f':
            assert(dtype == 0)
        else:
            raise ValueError(f"Unhandled data type '{self.dtype}'")

        # 将数据中的名称赋值给self.name，并更新偏移量
        self.name = bytes(data[offset:offset+namelen]); offset += namelen
        # 32字节对齐
        offset += (0 - offset) & 31
        # 将数据中的数据赋值给self.data
        self.data = data[offset:offset+self.nbytes]
        # 更新偏移量
        offset += self.nbytes
        # 返回更新后的偏移量
        return offset

    # 计算最大存储空间大小
    def max_storage_size(self):
        result = 0
        # 增加4字节，用于存储nd
        result += 4 # nd
        # 增加4字节，用于存储namelen
        result += 4 # namelen
        # 增加4字节，用于存储dtype
        result += 4 # dtype
        # 增加ne列表长度乘以8字节，用于存储ne
        result += len(self.ne)*8 # ne
        # 增加48字节，用于存储名称（截至提交3b5515bbe0e2224425986ba24f1f5d84aa38dce9时的最大值）
        result += 48 # name (maximum as of commit 3b5515bbe0e2224425986ba24f1f5d84aa38dce9)
        # 增加31字节，用于32字节对齐
        result += 31 # 32-byte alignment
        # 增加self.nbytes字节，用于存储数据
        result += self.nbytes
        # 返回结果
        return result

    # 将数据保存到gguf_writer中
    def save_gguf(self, gguf_writer, name):
        # 向gguf_writer中添加张量
        gguf_writer.add_tensor(
            name=name,
            tensor=self.data,
            raw_shape=np.array(list(reversed(self.ne))),
            raw_dtype=gguf.GGMLQuantizationType.F32)
# 定义 OptimizationParamsV0 类
class OptimizationParamsV0:
    # 初始化函数
    def __init__(self):
        pass

# 定义 OptimizationContext 类
class OptimizationContext:
    # 初始化函数
    def __init__(self):
        pass

# 定义 ModelParams 类
class ModelParams:
    # 初始化函数
    def __init__(self):
        pass

    # 从数据中加载模型参数
    def load(self, data, offset):
        # 从数据中解析出词汇量
        self.n_vocab = struct.unpack('<I', bytes(data[offset:offset + 4]))[0];  offset += 4
        # 从数据中解析出嵌入维度
        self.n_embd  = struct.unpack('<I', bytes(data[offset:offset + 4]))[0];  offset += 4
        # 从数据中解析出多头注意力的数量
        self.n_mult  = struct.unpack('<I', bytes(data[offset:offset + 4]))[0];  offset += 4
        # 从数据中解析出注意力头的数量
        self.n_head  = struct.unpack('<I', bytes(data[offset:offset + 4]))[0];  offset += 4
        # 从数据中解析出层数
        self.n_layer = struct.unpack('<I', bytes(data[offset:offset + 4]))[0];  offset += 4
        # 从数据中解析出旋转数量
        self.n_rot   = struct.unpack('<I', bytes(data[offset:offset + 4]))[0];  offset += 4
        return offset

    # 获取前馈神经网络的数量
    def get_n_ff(self):
        # 根据公式计算前馈神经网络的数量
        return ((2*(4*self.n_embd)//3 + self.n_mult - 1)//self.n_mult)*self.n_mult

    # 保存 GGUF（某种数据格式）到文件中
    def save_gguf(self, gguf_writer):
        # self.n_vocab 不会被保存
        # 添加嵌入长度到 GGUF 中
        gguf_writer.add_embedding_length(self.n_embd)
        # 添加注意力头数量到 GGUF 中
        gguf_writer.add_head_count(self.n_head)
        # 添加块数量到 GGUF 中
        gguf_writer.add_block_count(self.n_layer)
        # 添加绳索维度数量到 GGUF 中
        gguf_writer.add_rope_dimension_count(self.n_rot)
        # 添加前馈神经网络长度到 GGUF 中
        gguf_writer.add_feed_forward_length(self.get_n_ff())

# 定义 tensor_name 函数
def tensor_name(key, bid=None):
    # 返回格式化后的张量名称
    return gguf.TENSOR_NAMES[key].format(bid=bid) + ".weight"

# 定义 Layer 类
class Layer:
    pass
    # 初始化函数，接受参数和标识符
    def __init__(self, params, bid):
        # 将参数中的标识符赋值给对象属性
        self.bid = bid
        # 初始化注意力层的归一化张量
        self.att_norm = Tensor('f', [params.n_embd])
        # 初始化查询权重张量
        self.wq = Tensor('f', [params.n_embd, params.n_embd])
        # 初始化键权重张量
        self.wk = Tensor('f', [params.n_embd, params.n_embd])
        # 初始化值权重张量
        self.wv = Tensor('f', [params.n_embd, params.n_embd])
        # 初始化输出权重张量
        self.wo = Tensor('f', [params.n_embd, params.n_embd])
        # 初始化前馈神经网络层的归一化张量
        self.ffn_norm = Tensor('f', [params.n_embd])
        # 初始化前馈神经网络第一层权重张量
        self.w1 = Tensor('f', [params.n_embd, params.get_n_ff()])
        # 初始化前馈神经网络第二层权重张量
        self.w2 = Tensor('f', [params.get_n_ff(), params.n_embd])
        # 初始化前馈神经网络第三层权重张量
        self.w3 = Tensor('f', [params.n_embd, params.get_n_ff()])
    
    # 加载函数，接受数据和偏移量
    def load(self, data, offset):
        # 调用各张量的加载函数，并更新偏移量
        offset = self.att_norm.load(data, offset)
        offset = self.wq.load(data, offset)
        offset = self.wk.load(data, offset)
        offset = self.wv.load(data, offset)
        offset = self.wo.load(data, offset)
        offset = self.ffn_norm.load(data, offset)
        offset = self.w1.load(data, offset)
        offset = self.w2.load(data, offset)
        offset = self.w3.load(data, offset)
        # 返回更新后的偏移量
        return offset
    # 保存自注意力模型的各个部分到 GGUF 文件中
    def save_gguf(self, gguf_writer):
        # 保存注意力规范化层的 GGUF 数据，指定名称为注意力规范化层对应的张量名
        self.att_norm.save_gguf(gguf_writer, name=tensor_name(gguf.MODEL_TENSOR.ATTN_NORM, self.bid))
        # 保存注意力查询权重的 GGUF 数据，指定名称为注意力查询权重对应的张量名
        self.wq.save_gguf      (gguf_writer, name=tensor_name(gguf.MODEL_TENSOR.ATTN_Q,    self.bid))
        # 保存注意力键权重的 GGUF 数据，指定名称为注意力键权重对应的张量名
        self.wk.save_gguf      (gguf_writer, name=tensor_name(gguf.MODEL_TENSOR.ATTN_K,    self.bid))
        # 保存注意力值权重的 GGUF 数据，指定名称为注意力值权重对应的张量名
        self.wv.save_gguf      (gguf_writer, name=tensor_name(gguf.MODEL_TENSOR.ATTN_V,    self.bid))
        # 保存注意力输出权重的 GGUF 数据，指定名称为注意力输出权重对应的张量名
        self.wo.save_gguf      (gguf_writer, name=tensor_name(gguf.MODEL_TENSOR.ATTN_OUT,  self.bid))
        # 保存前馈神经网络规范化层的 GGUF 数据，指定名称为前馈神经网络规范化层对应的张量名
        self.ffn_norm.save_gguf(gguf_writer, name=tensor_name(gguf.MODEL_TENSOR.FFN_NORM,  self.bid))
        # 保存前馈神经网络第一层权重的 GGUF 数据，指定名称为前馈神经网络第一层权重对应的张量名
        self.w1.save_gguf      (gguf_writer, name=tensor_name(gguf.MODEL_TENSOR.FFN_GATE,  self.bid))
        # 保存前馈神经网络第二层权重的 GGUF 数据，指定名称为前馈神经网络第二层权重对应的张量名
        self.w2.save_gguf      (gguf_writer, name=tensor_name(gguf.MODEL_TENSOR.FFN_DOWN,  self.bid))
        # 保存前馈神经网络第三层权重的 GGUF 数据，指定名称为前馈神经网络第三层权重对应的张量名
        self.w3.save_gguf      (gguf_writer, name=tensor_name(gguf.MODEL_TENSOR.FFN_UP,    self.bid))
# 定义模型类
class Model:
    # 初始化方法
    def __init__(self):
        # 初始化模型参数对象
        self.params = ModelParams()
        # 初始化层列表
        self.layers = []

    # 加载方法，用于加载模型数据
    def load(self, data, offset):
        # 调用模型参数对象的加载方法，返回新的偏移量
        offset = self.params.load(data, offset)

        # 初始化 token embedding 张量
        self.tok_embd = Tensor('f', [self.params.n_embd, self.params.n_vocab])
        # 初始化 norm 张量
        self.norm     = Tensor('f', [self.params.n_embd])
        # 初始化输出张量
        self.output   = Tensor('f', [self.params.n_embd, self.params.n_vocab])

        # 调用各张量的加载方法，返回新的偏移量
        offset = self.tok_embd.load(data, offset)
        offset = self.norm.load(data, offset)
        offset = self.output.load(data, offset)

        # 清空层列表
        self.layers.clear()
        # 遍历层的数量，创建并加载每一层
        for bid in range(self.params.n_layer):
            layer = Layer(self.params, bid)
            offset = layer.load(data, offset)
            self.layers.append(layer)

        # 返回最终偏移量
        return offset

    # 保存 gguf 方法，用于保存模型到 gguf_writer
    def save_gguf(self, gguf_writer):
        # 调用模型参数对象的保存 gguf 方法
        self.params.save_gguf(gguf_writer)

        # 调用各张量的保存 gguf 方法，指定名称
        self.tok_embd.save_gguf(gguf_writer, name=tensor_name(gguf.MODEL_TENSOR.TOKEN_EMBD))
        self.norm.save_gguf    (gguf_writer, name=tensor_name(gguf.MODEL_TENSOR.OUTPUT_NORM))
        self.output.save_gguf  (gguf_writer, name=tensor_name(gguf.MODEL_TENSOR.OUTPUT))

        # 遍历层列表，调用每一层的保存 gguf 方法
        for layer in self.layers:
            layer.save_gguf(gguf_writer)

# 定义检查点类
class Checkpoint:
    # 初始化方法
    def __init__(self):
        # 初始化模型对象
        self.model = Model()
        # 初始化优化上下文对象
        self.opt_ctx = OptimizationContext()
    # 从给定数据中加载检查点文件的内容，并返回偏移量
    def load(self, data, offset):
        # 读取并反转4个字节的数据，得到文件头部的魔数
        magic   = bytes(reversed(data[offset:offset + 4])); offset += 4
        # 如果魔数不是'ggcp'，则抛出数值错误
        if magic != b'ggcp':
            raise ValueError(f"File header magic indicates, that this is no checkpoint file. Expected 'ggcp', Got '{str(magic)}'")

        # 从数据中读取4个字节，解析为小端无符号整数，得到版本号
        self.version = struct.unpack('<I', bytes(data[offset:offset + 4]))[0]; offset += 4
        # 如果版本号不为0，则抛出数值错误
        if self.version != 0:
            raise ValueError('Invalid version of checkpoint file')

        # 从数据中读取4个字节，解析为小端无符号整数，得到训练迭代次数
        self.train_its     = struct.unpack('<I', bytes(data[offset:offset + 4]))[0]; offset += 4
        # 从数据中读取4个字节，解析为小端无符号整数，得到训练样本数
        self.train_samples = struct.unpack('<I', bytes(data[offset:offset + 4]))[0]; offset += 4
        # 从数据中读取4个字节，解析为小端无符号整数，得到训练标记数
        self.train_tokens  = struct.unpack('<I', bytes(data[offset:offset + 4]))[0]; offset += 4

        # 调用模型对象的加载方法，加载模型数据，并更新偏移量
        offset = self.model.load(data, offset)
        # 调用优化上下文对象的加载方法，加载优化上下文数据，并更新偏移量
        offset = self.opt_ctx.load(data, offset)

        # 返回更新后的偏移量
        return offset

    # 将训练信息保存到 GGUF 文件中
    def save_gguf(self, gguf_writer):
        # 向 GGUF 写入文件类型
        gguf_writer.add_file_type(gguf.GGMLQuantizationType.F32)
        # 向 GGUF 写入层归一化的 RMS 误差
        gguf_writer.add_layer_norm_rms_eps(1e-5)
        # 向 GGUF 写入训练文件版本号
        gguf_writer.add_uint32(LLM_KV_TRAINING_FILE_VERSION,    0)
        # 向 GGUF 写入训练类型
        gguf_writer.add_string(LLM_KV_TRAINING_TYPE,            LLM_KV_TRAINING_TYPE_TRAIN_MODEL)
        # 向 GGUF 写入训练迭代次数
        gguf_writer.add_uint32(LLM_KV_TRAINING_ITERATION_COUNT, self.train_its)
        # 向 GGUF 写入训练样本数
        gguf_writer.add_uint32(LLM_KV_TRAINING_SAMPLE_COUNT,    self.train_samples)
        # 向 GGUF 写入训练标记数
        gguf_writer.add_uint32(LLM_KV_TRAINING_TOKEN_COUNT,     self.train_tokens)
        # 调用模型对象的保存 GGUF 方法，将模型数据保存到 GGUF 文件中
        self.model.save_gguf(gguf_writer)
        # 调用优化上下文对象的保存 GGUF 方法，将优化上下文数据保存到 GGUF 文件中
        self.opt_ctx.save_gguf(gguf_writer)
# 处理命令行参数，解析输入和输出文件名
def handle_args():
    # 创建参数解析器，设置描述信息
    parser = argparse.ArgumentParser(description = 'Convert train-text-from-scratch checkpoints to GGUF')
    # 添加输入文件名参数
    parser.add_argument('--input',  '-i', type = Path, help = 'Input train checkpoint filename', required=True)
    # 添加输出文件名参数
    parser.add_argument('--output', '-o', type = Path, help ='Output GGUF filename', required=True)
    # 解析并返回参数
    return parser.parse_args()

# 主函数
def main():
    # 处理命令行参数
    cfg = handle_args()
    # 从文件中创建内存映射
    data = np.memmap(cfg.input, mode = 'r')
    # 创建 Checkpoint 对象
    chk = Checkpoint()
    # 设置偏移量为 0
    offset = 0
    # 从内存映射中加载数据到 Checkpoint 对象，返回新的偏移量
    offset = chk.load(data, offset)
    # 断言偏移量等于数据长度，即已经读取了所有可用数据
    assert(offset == len(data))

    # 创建 GGUFWriter 对象，指定输出文件名和模型架构
    gguf_writer = gguf.GGUFWriter(cfg.output, gguf.MODEL_ARCH_NAMES[gguf.MODEL_ARCH.LLAMA], use_temp_file = False)
    # 将 Checkpoint 对象保存为 GGUF 格式
    chk.save_gguf(gguf_writer)
    # 输出提示信息
    print("    gguf: write header")
    # 将 GGUF 头部信息写入文件
    gguf_writer.write_header_to_file()
    # 输出提示信息
    print("    gguf: write metadata")
    # 将 GGUF 元数据写入文件
    gguf_writer.write_kv_data_to_file()
    # 输出提示信息
    print("    gguf: write tensors")
    # 将 GGUF 张量数据写入文件
    gguf_writer.write_tensors_to_file()
    # 关闭 GGUFWriter 对象
    gguf_writer.close()

# 如果当前脚本被直接执行，则调用主函数
if __name__ == '__main__':
    main()
```