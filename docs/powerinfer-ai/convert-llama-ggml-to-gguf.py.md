# `PowerInfer\convert-llama-ggml-to-gguf.py`

```
#!/usr/bin/env python3
# 指定脚本解释器为 Python 3

from __future__ import annotations
# 导入未来版本的注解特性

import argparse
# 导入用于解析命令行参数的模块
import math
# 导入数学函数模块
import struct
# 导入处理字节数据的模块
import sys
# 导入系统相关的模块
from enum import IntEnum
# 从枚举模块中导入 IntEnum 类
from pathlib import Path
# 从路径模块中导入 Path 类

import numpy as np
# 导入数值计算库 numpy

import os
# 导入操作系统相关的模块
if 'NO_LOCAL_GGUF' not in os.environ:
    sys.path.insert(1, str(Path(__file__).parent / 'gguf-py'))
# 如果环境变量中没有 NO_LOCAL_GGUF，则将 gguf-py 目录添加到搜索路径中
import gguf
# 导入 gguf 模块

class GGMLFormat(IntEnum):
    GGML = 0
    GGMF = 1
    GGJT = 2
# 定义 GGMLFormat 枚举类，包含 GGML、GGMF 和 GGJT 三种格式

class GGMLFType(IntEnum):
    # 定义 GGMLFType 枚举类
    ALL_F32              = 0
    MOSTLY_F16           = 1
    MOSTLY_Q4_0          = 2
    MOSTLY_Q4_1          = 3
    MOSTLY_Q4_1_SOME_F16 = 4
    MOSTLY_Q8_0          = 7
    MOSTLY_Q5_0          = 8
    MOSTLY_Q5_1          = 9
    MOSTLY_Q2_K          = 10
    MOSTLY_Q3_K_S        = 11
    MOSTLY_Q3_K_M        = 12
    MOSTLY_Q3_K_L        = 13
    MOSTLY_Q4_K_S        = 14
    MOSTLY_Q4_K_M        = 15
    MOSTLY_Q5_K_S        = 16
    MOSTLY_Q5_K_M        = 17
    MOSTLY_Q6_K          = 18
    # 定义 GGMLFType 枚举类的各个枚举值

class Hyperparameters:
    # 定义超参数类
    def __init__(self):
        # 初始化方法
        self.n_vocab = self.n_embd = self.n_mult = self.n_head = 0
        self.n_layer = self.n_rot = self.n_ff = 0
        self.ftype = GGMLFType.ALL_F32
        # 初始化各个参数和类型

    def set_n_ff(self, model):
        # 设置 n_ff 方法
        ff_tensor_idx = model.tensor_map.get(b'layers.0.feed_forward.w1.weight')
        # 获取指定键对应的值
        assert ff_tensor_idx is not None, 'Missing layer 0 FF tensor'
        # 断言 ff_tensor_idx 不为空，否则抛出异常
        ff_tensor = model.tensors[ff_tensor_idx]
        # 获取指定索引的张量
        self.n_ff = ff_tensor.dims[1]
        # 设置 n_ff 参数为张量的第二个维度大小

    def load(self, data, offset):
        # 加载方法，接受数据和偏移量作为参数
        (
            self.n_vocab,
            self.n_embd,
            self.n_mult,
            self.n_head,
            self.n_layer,
            self.n_rot,
            ftype,
        ) = struct.unpack('<7I', data[offset:offset + (4 * 7)])
        # 使用 struct 模块解包数据
        try:
            self.ftype = GGMLFType(ftype)
            # 尝试将 ftype 转换为 GGMLFType 类型
        except ValueError:
            raise ValueError(f'Invalid ftype {ftype}')
            # 如果转换失败，抛出异常
        return 4 * 7
        # 返回偏移量增加的值
    # 定义一个特殊方法，用于返回对象的字符串表示
    def __str__(self):
        # 返回包含对象各个属性数值的字符串
        return f'<Hyperparameters: n_vocab={self.n_vocab}, n_embd={self.n_embd}, n_mult={self.n_mult}, n_head={self.n_head}, n_layer={self.n_layer}, n_rot={self.n_rot}, n_ff={self.n_ff}, ftype={self.ftype.name}>'
# 定义一个名为 Vocab 的类
class Vocab:
    # 初始化方法，设置是否加载分数
    def __init__(self, load_scores = True):
        # 初始化词汇表列表
        self.items = []
        # 设置是否加载分数
        self.load_scores = load_scores

    # 加载方法，用于加载数据
    def load(self, data, offset, n_vocab):
        # 保存原始偏移量
        orig_offset = offset
        # 遍历 n_vocab 次
        for _ in range(n_vocab):
            # 读取项目长度
            itemlen = struct.unpack('<I', data[offset:offset + 4])[0]
            # 断言项目长度小于 4096，否则抛出异常
            assert itemlen < 4096, 'Absurd vocab item length'
            # 偏移量增加 4
            offset += 4
            # 读取项目文本
            item_text = bytes(data[offset:offset + itemlen])
            # 偏移量增加项目长度
            offset += itemlen
            # 如果设置了加载分数
            if self.load_scores:
                # 读取项目分数
                item_score = struct.unpack('<f', data[offset:offset + 4])[0]
                # 偏移量增加 4
                offset += 4
            # 如果未设置加载分数
            else:
                # 分数默认为 0.0
                item_score = 0.0
            # 将项目文本和分数添加到词汇表列表中
            self.items.append((item_text, item_score))
        # 返回偏移量增加的值
        return offset - orig_offset

# 定义一个名为 Tensor 的类
class Tensor:
    # 初始化方法，设置是否使用填充
    def __init__(self, use_padding = True):
        # 张量名称
        self.name = None
        # 张量维度
        self.dims: tuple[int, ...] = ()
        # 张量数据类型
        self.dtype = None
        # 起始偏移量
        self.start_offset = 0
        # 数据长度
        self.len_bytes = np.int64(0)
        # 是否使用填充
        self.use_padding = use_padding
    # 从给定数据和偏移量加载张量信息
    def load(self, data, offset):
        # 保存原始偏移量
        orig_offset = offset
        # 从数据中解析出张量的维度、名称长度和数据类型
        (n_dims, name_len, dtype) = struct.unpack('<3I', data[offset:offset + 12])
        # 检查张量维度是否在合理范围内
        assert n_dims >= 0 and n_dims <= 4, f'Invalid tensor dimensions {n_dims}'
        # 检查张量名称长度是否合理
        assert name_len < 4096, 'Absurd tensor name length'
        # 获取数据类型对应的量化信息
        quant = gguf.GGML_QUANT_SIZES.get(dtype)
        # 检查数据类型是否已知
        assert quant is not None, 'Unknown tensor type'
        # 获取量化块大小和数据类型大小
        (blksize, tysize) = quant
        offset += 12
        # 设置张量的数据类型
        self.dtype= dtype
        # 解析张量的维度信息
        self.dims = struct.unpack(f'<{n_dims}I', data[offset:offset + (4 * n_dims)])
        offset += 4 * n_dims
        # 解析张量的名称
        self.name = bytes(data[offset:offset + name_len])
        offset += name_len
        # 如果使用填充，计算填充大小
        pad = ((offset + 31) & ~31) - offset if self.use_padding else 0
        offset += pad
        # 计算张量元素个数
        n_elems = np.prod(self.dims)
        # 计算张量占用的字节数
        n_bytes = np.int64(np.int64(n_elems) * np.int64(tysize)) // np.int64(blksize)
        # 设置张量数据的起始偏移量和长度
        self.start_offset = offset
        self.len_bytes = n_bytes
        offset += n_bytes
        # 返回偏移量的增量，即加载的数据长度
        return offset - orig_offset
class GGMLModel:
    # 初始化 GGMLModel 类
    def __init__(self):
        # 初始化超参数
        self.hyperparameters = None
        # 初始化词汇表
        self.vocab = None
        # 初始化张量映射
        self.tensor_map = {}
        # 初始化张量列表
        self.tensors = []

    # 验证文件头部信息
    def validate_header(self, data, offset):
        # 读取文件头部的魔术字节
        magic = bytes(data[offset:offset + 4])
        # 如果魔术字节为'GGUF'，则文件已经是 GGUF 格式，抛出数值错误
        if magic == b'GGUF':
            raise ValueError('File is already in GGUF format.')
        # 如果魔术字节为'lmgg'，则文件格式为 GGML，设置文件格式和版本号，返回偏移量
        if magic == b'lmgg':
            self.file_format = GGMLFormat.GGML
            self.format_version = 1
            return 4
        # 读取版本号
        version = struct.unpack('<I', data[offset + 4:offset + 8])[0]
        # 如果魔术字节为'fmgg'，并且版本号不为1，抛出数值错误
        if magic == b'fmgg':
            if version != 1:
                raise ValueError(f'Cannot handle unexpected GGMF file version {version}')
            self.file_format = GGMLFormat.GGMF
            self.format_version = version
            return 8
        # 如果魔术字节为'tjgg'，并且版本号小于1或大于3，抛出数值错误
        if magic == b'tjgg':
            if version < 1 or version > 3:
                raise ValueError(f'Cannot handle unexpected GGJT file version {version}')
            self.file_format = GGMLFormat.GGJT
            self.format_version = version
            return 8
        # 如果以上条件都不满足，抛出数值错误，提示文件不是 GGML 格式
        raise ValueError(f"Unexpected file magic {magic!r}! This doesn't look like a GGML format file.")

    # 验证转换
    def validate_conversion(self, ftype):
        err = ''
        # 如果文件格式小于 GGJT 或版本号小于2
        if (self.file_format < GGMLFormat.GGJT or self.format_version < 2):
            # 如果文件类型不是全部为F32或大部分为F16，设置错误信息
            if ftype not in (GGMLFType.ALL_F32, GGMLFType.MOSTLY_F16):
                err = 'Quantizations changed in GGJTv2. Can only convert unquantized GGML files older than GGJTv2.'
        # 如果文件格式为 GGJT 并且版本号为2
        elif (self.file_format == GGMLFormat.GGJT and self.format_version == 2):
            # 如果文件类型为Q4_0、Q4_1、Q4_1_SOME_F16或Q8_0，设置错误信息
            if ftype in (GGMLFType.MOSTLY_Q4_0, GGMLFType.MOSTLY_Q4_1,
                         GGMLFType.MOSTLY_Q4_1_SOME_F16, GGMLFType.MOSTLY_Q8_0):
                err = 'Q4 and Q8 quantizations changed in GGJTv3.'
        # 如果错误信息长度大于0，抛出数值错误，提示文件不符合转换条件
        if len(err) > 0:
            raise ValueError(f'{err} Sorry, your {self.file_format.name}v{self.format_version} file of type {ftype.name} is not eligible for conversion.')
    # 从给定数据和偏移量加载数据
    def load(self, data, offset):
        # 调用 validate_header 方法验证数据头部，并更新偏移量
        offset += self.validate_header(data, offset)
        # 创建 Hyperparameters 对象
        hp = Hyperparameters()
        # 调用 hp 对象的 load 方法加载数据，并更新偏移量
        offset += hp.load(data, offset)
        # 打印文件格式和类型信息
        print(f'* File format: {self.file_format.name}v{self.format_version} with ftype {hp.ftype.name}')
        # 调用 validate_conversion 方法验证转换类型
        self.validate_conversion(hp.ftype)
        # 创建 Vocab 对象
        vocab = Vocab(load_scores = self.file_format > GGMLFormat.GGML)
        # 调用 vocab 对象的 load 方法加载数据，并更新偏移量
        offset += vocab.load(data, offset, hp.n_vocab)
        # 创建空列表 tensors 和空字典 tensor_map
        tensors: list[Tensor] = []
        tensor_map = {}
        # 循环直到偏移量小于数据长度
        while offset < len(data):
            # 创建 Tensor 对象
            tensor = Tensor(use_padding = self.file_format > GGMLFormat.GGMF)
            # 调用 tensor 对象的 load 方法加载数据，并更新偏移量
            offset += tensor.load(data, offset)
            # 将 tensor 对象的名称和索引添加到 tensor_map 字典中
            tensor_map[tensor.name] = len(tensors)
            # 将 tensor 对象添加到 tensors 列表中
            tensors.append(tensor)
        # 设置对象的 hyperparameters、vocab、tensors 和 tensor_map 属性
        self.hyperparameters = hp
        self.vocab = vocab
        self.tensors = tensors
        self.tensor_map = tensor_map
        # 调用 hp 对象的 set_n_ff 方法设置对象的 n_ff 属性
        hp.set_n_ff(self)
        # 返回更新后的偏移量
        return offset
# 定义一个名为 GGMLToGGUF 的类
class GGMLToGGUF:
    # 初始化方法，接受 ggml_model、data、cfg 三个参数，还有可选的 params_override、vocab_override、special_vocab 参数
    def __init__(self, ggml_model, data, cfg, params_override = None, vocab_override = None, special_vocab = None):
        # 获取 ggml_model 的超参数
        hp = ggml_model.hyperparameters
        # 将参数赋值给实例变量
        self.model = ggml_model
        self.data = data
        self.cfg = cfg
        self.params_override = params_override
        self.vocab_override = vocab_override
        self.special_vocab = special_vocab
        # 如果 params_override 不为空，则将 n_kv_head 赋值为 params_override.n_head_kv
        if params_override is not None:
            n_kv_head = params_override.n_head_kv
        # 如果 params_override 为空
        else:
            # 如果 cfg.gqa 为 1，则将 n_kv_head 赋值为 hp.n_head
            if cfg.gqa == 1:
                n_kv_head = hp.n_head
            # 如果 cfg.gqa 不为 1
            else:
                # 将 gqa 转换为浮点数
                gqa = float(cfg.gqa)
                n_kv_head = None
                # 遍历范围为 1 到 256 的数字
                for x in range(1, 256):
                    # 如果 hp.n_head 除以 x 等于 gqa
                    if float(hp.n_head) / float(x) == gqa:
                        # 将 n_kv_head 赋值为 x
                        n_kv_head = x
                # 断言 n_kv_head 不为空，如果为空则抛出异常 "Couldn't determine n_kv_head from GQA param"
                assert n_kv_head is not None, "Couldn't determine n_kv_head from GQA param"
                # 打印信息，表示根据 cfg.gqa 猜测的 n_kv_head 值
                print(f'- Guessed n_kv_head = {n_kv_head} based on GQA {cfg.gqa}')
        # 将 n_kv_head 赋值给实例变量
        self.n_kv_head = n_kv_head
        # 调用 gguf 模块的 get_tensor_name_map 方法，获取名称映射
        self.name_map = gguf.get_tensor_name_map(gguf.MODEL_ARCH.LLAMA, ggml_model.hyperparameters.n_layer)

    # 定义一个保存方法
    def save(self):
        # 打印准备保存 GGUF 文件的信息
        print('* Preparing to save GGUF file')
        # 创建 GGUFWriter 对象
        gguf_writer = gguf.GGUFWriter(
            self.cfg.output,
            gguf.MODEL_ARCH_NAMES[gguf.MODEL_ARCH.LLAMA],
            use_temp_file = False )
        # 调用 add_params 方法，将参数添加到 GGUF 文件中
        self.add_params(gguf_writer)
        # 调用 add_vocab 方法，将词汇表添加到 GGUF 文件中
        self.add_vocab(gguf_writer)
        # 如果 special_vocab 不为空，则将其添加到 GGUF 文件中
        if self.special_vocab is not None:
            self.special_vocab.add_to_gguf(gguf_writer)
        # 调用 add_tensors 方法，将张量添加到 GGUF 文件中
        self.add_tensors(gguf_writer)
        # 打印信息，表示正在写入 GGUF 文件头部
        print("    gguf: write header")
        gguf_writer.write_header_to_file()
        # 打印信息，表示正在写入 GGUF 文件元数据
        print("    gguf: write metadata")
        gguf_writer.write_kv_data_to_file()
        # 打印信息，表示正在写入 GGUF 文件张量数据
        print("    gguf: write tensors")
        gguf_writer.write_tensors_to_file()
        # 关闭 GGUFWriter 对象
        gguf_writer.close()
    # 为给定的 gguf_writer 添加模型参数
    def add_params(self, gguf_writer):
        # 获取模型的超参数和配置
        hp = self.model.hyperparameters
        cfg = self.cfg
        # 如果配置中包含描述信息，则使用配置中的描述，否则使用默认描述
        if cfg.desc is not None:
            desc = cfg.desc
        else:
            desc = f'converted from legacy {self.model.file_format.name}v{self.model.format_version} {hp.ftype.name} format'
        try:
            # 尝试获取文件名，如果文件名不是有效的 UTF8 编码，则将其置为 None
            name = cfg.name if cfg.name is not None else cfg.input.name
        except UnicodeDecodeError:
            name = None
        # 打印提示信息
        print('* Adding model parameters and KV items')
        # 如果文件名不为 None，则将其添加到 gguf_writer 中
        if name is not None:
            gguf_writer.add_name(name)
        # 将描述信息添加到 gguf_writer 中
        gguf_writer.add_description(desc)
        # 将文件类型添加到 gguf_writer 中
        gguf_writer.add_file_type(int(hp.ftype))
        # 如果存在参数覆盖，则使用覆盖的参数，否则使用默认参数
        if self.params_override is not None:
            po = self.params_override
            # 检查参数是否匹配，如果不匹配则抛出异常
            assert po.n_embd == hp.n_embd, 'Model hyperparams mismatch'
            assert po.n_layer == hp.n_layer, 'Model hyperparams mismatch'
            assert po.n_head == hp.n_head, 'Model hyperparams mismatch'
            # 添加覆盖参数到 gguf_writer 中
            gguf_writer.add_context_length      (po.n_ctx)
            gguf_writer.add_embedding_length    (po.n_embd)
            gguf_writer.add_block_count         (po.n_layer)
            gguf_writer.add_feed_forward_length (po.n_ff)
            gguf_writer.add_rope_dimension_count(po.n_embd // po.n_head)
            gguf_writer.add_head_count          (po.n_head)
            gguf_writer.add_head_count_kv       (po.n_head_kv)
            gguf_writer.add_layer_norm_rms_eps  (po.f_norm_eps)
            # 返回结果
            return
        # 添加默认参数到 gguf_writer 中
        gguf_writer.add_context_length(cfg.context_length)
        gguf_writer.add_embedding_length(hp.n_embd)
        gguf_writer.add_block_count(hp.n_layer)
        gguf_writer.add_feed_forward_length(hp.n_ff)
        gguf_writer.add_rope_dimension_count(hp.n_embd // hp.n_head)
        gguf_writer.add_head_count(hp.n_head)
        gguf_writer.add_head_count_kv(self.n_kv_head)
        gguf_writer.add_layer_norm_rms_eps(float(cfg.eps))
    # 向 gguf_writer 中添加张量数据
    def add_tensors(self, gguf_writer):
        # 获取张量名称映射表
        tensor_map = self.name_map
        # 获取数据
        data = self.data
        # 打印要添加的张量数量
        print(f'* Adding {len(self.model.tensors)} tensor(s)')
        # 遍历模型中的张量
        for tensor in self.model.tensors:
            # 获取张量名称并转换为 UTF-8 编码
            name = str(tensor.name, 'UTF-8')
            # 获取映射后的张量名称
            mapped_name = tensor_map.get_name(name, try_suffixes = (".weight", ".bias"))
            # 断言映射后的张量名称不为空，否则抛出异常
            assert mapped_name is not None, f'Bad name {name}'
            # 处理张量维度，将第一维和第二维交换
            tempdims = list(tensor.dims[:])
            if len(tempdims) > 1:
                temp = tempdims[1]
                tempdims[1] = tempdims[0]
                tempdims[0] = temp
            # 向 gguf_writer 中添加张量
            gguf_writer.add_tensor(
                mapped_name,
                data[tensor.start_offset:tensor.start_offset + tensor.len_bytes],
                raw_shape = tempdims,
                raw_dtype = tensor.dtype )
# 处理模型元数据，根据配置和超参数
def handle_metadata(cfg, hp):
    # 导入 convert 模块
    import convert
    # 断言模型元数据目录存在且是一个目录
    assert cfg.model_metadata_dir.is_dir(), 'Metadata dir is not a directory'
    # 构建 HF 配置文件路径和原始配置文件路径
    hf_config_path   = cfg.model_metadata_dir / "config.json"
    orig_config_path = cfg.model_metadata_dir / "params.json"
    # 创建一个虚拟模型，用于检查一些张量的形状
    fakemodel = {
        'tok_embeddings.weight': convert.LazyTensor.__new__(convert.LazyTensor),
        'layers.0.feed_forward.w1.weight': convert.LazyTensor.__new__(convert.LazyTensor),
    }
    # 设置虚拟模型的形状信息
    fakemodel['tok_embeddings.weight'].shape = [hp.n_vocab]
    fakemodel['layers.0.feed_forward.w1.weight'].shape = [hp.n_ff]
    # 如果 HF 配置文件存在，则加载 HF 转换器的参数
    if hf_config_path.exists():
        params = convert.Params.loadHFTransformerJson(fakemodel, hf_config_path)
    # 如果原始配置文件存在，则加载原始参数的参数
    elif orig_config_path.exists():
        params = convert.Params.loadOriginalParamsJson(fakemodel, orig_config_path)
    else:
        # 抛出数值错误，无法加载元数据
        raise ValueError('Unable to load metadata')
    # 加载词汇表
    vocab = convert.load_vocab(
        cfg.vocab_dir if cfg.vocab_dir is not None else cfg.model_metadata_dir,
        cfg.vocabtype )
    # FIXME: Respect cfg.vocab_dir?
    # 加载特殊词汇表
    svocab = gguf.SpecialVocab(cfg.model_metadata_dir,
        load_merges = cfg.vocabtype == 'bpe',
        n_vocab = vocab.vocab_size)
    # 检查词汇表大小是否匹配参数
    convert.check_vocab_size(params, vocab)
    # 返回参数、词汇表和特殊词汇表
    return (params, vocab, svocab)

# 处理命令行参数
def handle_args():
    # 创建参数解析器
    parser = argparse.ArgumentParser(description = 'Convert GGML models to GGUF')
    # 添加输入文件参数
    parser.add_argument('--input', '-i', type = Path, required = True,
        help = 'Input GGMLv3 filename')
    # 添加输出文件参数
    parser.add_argument('--output', '-o', type = Path, required = True,
        help ='Output GGUF filename')
    # 添加模型名称参数
    parser.add_argument('--name',
        help = 'Set model name')
    # 添加模型描述参数
    parser.add_argument('--desc',
        help = 'Set model description')
    # 添加一个名为'--gqa'的命令行参数，类型为整数，默认值为1，用于设置grouped-query attention factor
    parser.add_argument('--gqa', type = int, default = 1,
        help = 'grouped-query attention factor (use 8 for LLaMA2 70B)')
    # 添加一个名为'--eps'的命令行参数，默认值为'5.0e-06'，用于设置RMS norm eps
    parser.add_argument('--eps', default = '5.0e-06',
        help = 'RMS norm eps: Use 1e-6 for LLaMA1 and OpenLLaMA, use 1e-5 for LLaMA2')
    # 添加一个名为'--context-length'的命令行参数，类型为整数，默认值为2048，用于设置默认的最大上下文长度
    parser.add_argument('--context-length', '-c', type=int, default = 2048,
        help = 'Default max context length: LLaMA1 is typically 2048, LLaMA2 is typically 4096')
    # 添加一个名为'--model-metadata-dir'的命令行参数，类型为Path，用于指定加载HuggingFace/.pth词汇和元数据的目录
    parser.add_argument('--model-metadata-dir', '-m', type = Path,
        help ='Load HuggingFace/.pth vocab and metadata from the specified directory')
    # 添加一个名为'--vocab-dir'的命令行参数，类型为Path，用于指定包含tokenizer.model的目录，如果与模型文件分开的话
    parser.add_argument("--vocab-dir", type=Path,
        help="directory containing tokenizer.model, if separate from model file - only meaningful with --model-metadata-dir")
    # 添加一个名为'--vocabtype'的命令行参数，选项为["spm", "bpe"]，默认值为"spm"，用于设置词汇格式
    parser.add_argument("--vocabtype", choices=["spm", "bpe"], default="spm",
        help="vocab format - only meaningful with --model-metadata-dir and/or --vocab-dir (default: spm)")
    # 解析命令行参数并返回
    return parser.parse_args()
# 主函数，程序入口
def main():
    # 处理命令行参数，获取配置信息
    cfg = handle_args()
    # 打印配置信息
    print(f'* Using config: {cfg}')
    # 打印警告信息
    print('\n=== WARNING === Be aware that this conversion script is best-effort. Use a native GGUF model if possible. === WARNING ===\n')
    # 如果模型元数据目录为空并且 gqa 为 1 或 eps 为 '5.0e-06'
    if cfg.model_metadata_dir is None and (cfg.gqa == 1 or cfg.eps == '5.0e-06'):
        # 打印提示信息
        print('- Note: If converting LLaMA2, specifying "--eps 1e-5" is required. 70B models also need "--gqa 8".')
    # 从文件中创建内存映射，只读模式
    data = np.memmap(cfg.input, mode = 'r')
    # 创建 GGMLModel 对象
    model = GGMLModel()
    # 打印扫描 GGML 输入文件的信息
    print('* Scanning GGML input file')
    # 加载 GGML 模型数据，获取偏移量
    offset = model.load(data, 0)
    # 打印 GGML 模型的超参数
    print(f'* GGML model hyperparameters: {model.hyperparameters}')
    # 初始化变量
    vocab_override = None
    params_override = None
    special_vocab = None
    # 如果模型元数据目录不为空
    if cfg.model_metadata_dir is not None:
        # 处理模型元数据，获取参数覆盖、词汇表覆盖和特殊词汇
        (params_override, vocab_override, special_vocab) = handle_metadata(cfg, model.hyperparameters)
        # 打印提示信息
        print('!! Note: When overriding params the --gqa, --eps and --context-length options are ignored.')
        # 打印覆盖的参数
        print(f'* Overriding params: {params_override}')
        # 打印覆盖的词汇表
        print(f'* Overriding vocab: {vocab_override}')
        # 打印特殊词汇
        print(f'* Special vocab: {special_vocab}')
    else:
        # 打印警告信息
        print('\n=== WARNING === Special tokens may not be converted correctly. Use --model-metadata-dir if possible === WARNING ===\n')
        # 如果文件格式为 GGML
        if model.file_format == GGMLFormat.GGML:
            # 打印提示信息
            print('! This is a very old GGML file that does not contain vocab scores. Strongly recommend using model metadata!')
    # 创建 GGMLToGGUF 转换器对象
    converter = GGMLToGGUF(model, data, cfg,
        params_override = params_override,
        vocab_override = vocab_override,
        special_vocab = special_vocab )
    # 保存转换结果
    converter.save()
    # 打印成功完成信息，并输出保存路径
    print(f'* Successful completion. Output saved to: {cfg.output}')

# 如果当前脚本为主程序
if __name__ == '__main__':
    # 调用主函数
    main()
```