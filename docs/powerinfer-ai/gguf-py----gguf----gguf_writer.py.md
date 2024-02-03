# `PowerInfer\gguf-py\gguf\gguf_writer.py`

```cpp
# 导入必要的模块和类
from __future__ import annotations
import os
import shutil
import struct
import tempfile
from enum import Enum, auto
from io import BufferedWriter
from typing import IO, Any, Sequence
import numpy as np
from .constants import (
    GGUF_DEFAULT_ALIGNMENT,
    GGUF_MAGIC,
    GGUF_VERSION,
    GGMLQuantizationType,
    GGUFEndian,
    GGUFValueType,
    Keys,
    RopeScalingType,
    TokenType,
)

# 定义枚举类型WriterState
class WriterState(Enum):
    EMPTY   = auto()
    HEADER  = auto()
    KV_DATA = auto()
    TI_DATA = auto()

# 定义GGUFWriter类
class GGUFWriter:
    fout: BufferedWriter
    temp_file: tempfile.SpooledTemporaryFile[bytes] | None
    tensors: list[np.ndarray[Any, Any]]
    _simple_value_packing = {
        GGUFValueType.UINT8:   "B",
        GGUFValueType.INT8:    "b",
        GGUFValueType.UINT16:  "H",
        GGUFValueType.INT16:   "h",
        GGUFValueType.UINT32:  "I",
        GGUFValueType.INT32:   "i",
        GGUFValueType.FLOAT32: "f",
        GGUFValueType.UINT64:  "Q",
        GGUFValueType.INT64:   "q",
        GGUFValueType.FLOAT64: "d",
        GGUFValueType.BOOL:    "?",
    }

    # 初始化GGUFWriter对象
    def __init__(
        self, path: os.PathLike[str] | str, arch: str, use_temp_file: bool = True,
        endianess: GGUFEndian = GGUFEndian.LITTLE,
    ):
        # 打开文件准备写入
        self.fout = open(path, "wb")
        # 设置架构
        self.arch = arch
        # 设置字节序
        self.endianess = endianess
        self.offset_tensor = 0
        self.data_alignment = GGUF_DEFAULT_ALIGNMENT
        self.kv_data = bytearray()
        self.kv_data_count = 0
        self.ti_data = bytearray()
        self.ti_data_count = 0
        self.use_temp_file = use_temp_file
        self.temp_file = None
        self.tensors = []
        # 打印字节序信息
        print("gguf: This GGUF file is for {0} Endian only".format(
            "Big" if self.endianess == GGUFEndian.BIG else "Little",
        ))
        # 设置初始状态为EMPTY
        self.state = WriterState.EMPTY
        # 添加架构信息
        self.add_architecture()
    # 将头部信息写入文件
    def write_header_to_file(self) -> None:
        # 如果文件状态不是空的，抛出数值错误
        if self.state is not WriterState.EMPTY:
            raise ValueError(f'Expected output file to be empty, got {self.state}')
    
        # 写入魔数
        self._write_packed("<I", GGUF_MAGIC, skip_pack_prefix = True)
        # 写入版本号
        self._write_packed("I", GGUF_VERSION)
        # 写入类型信息数据的数量
        self._write_packed("Q", self.ti_data_count)
        # 写入键值对数据的数量
        self._write_packed("Q", self.kv_data_count)
        # 刷新文件缓冲区
        self.flush()
        # 更新文件状态为头部
        self.state = WriterState.HEADER
    
    # 将键值对数据写入文件
    def write_kv_data_to_file(self) -> None:
        # 如果文件状态不是头部，抛出数值错误
        if self.state is not WriterState.HEADER:
            raise ValueError(f'Expected output file to contain the header, got {self.state}')
    
        # 写入键值对数据
        self.fout.write(self.kv_data)
        # 刷新文件缓冲区
        self.flush()
        # 更新文件状态为键值对数据
        self.state = WriterState.KV_DATA
    
    # 将类型信息数据写入文件
    def write_ti_data_to_file(self) -> None:
        # 如果文件状态不是键值对数据，抛出数值错误
        if self.state is not WriterState.KV_DATA:
            raise ValueError(f'Expected output file to contain KV data, got {self.state}')
    
        # 写入类型信息数据
        self.fout.write(self.ti_data)
        # 刷新文件缓冲区
        self.flush()
        # 更新文件状态为类型信息数据
        self.state = WriterState.TI_DATA
    
    # 添加字符串类型的键值对
    def add_key(self, key: str) -> None:
        self.add_val(key, GGUFValueType.STRING, add_vtype=False)
    
    # 添加无符号8位整数类型的键值对
    def add_uint8(self, key: str, val: int) -> None:
        # 添加键
        self.add_key(key)
        # 添加值
        self.add_val(val, GGUFValueType.UINT8)
    
    # 添加有符号8位整数类型的键值对
    def add_int8(self, key: str, val: int) -> None:
        # 添加键
        self.add_key(key)
        # 添加值
        self.add_val(val, GGUFValueType.INT8)
    
    # 添加无符号16位整数类型的键值对
    def add_uint16(self, key: str, val: int) -> None:
        # 添加键
        self.add_key(key)
        # 添加值
        self.add_val(val, GGUFValueType.UINT16)
    
    # 添加有符号16位整数类型的键值对
    def add_int16(self, key: str, val: int) -> None:
        # 添加键
        self.add_key(key)
        # 添加值
        self.add_val(val, GGUFValueType.INT16)
    
    # 添加无符号32位整数类型的键值对
    def add_uint32(self, key: str, val: int) -> None:
        # 添加键
        self.add_key(key)
        # 添加值
        self.add_val(val, GGUFValueType.UINT32)
    
    # 添加有符号32位整数类型的键值对
    def add_int32(self, key: str, val: int) -> None:
        # 添加键
        self.add_key(key)
        # 添加值
        self.add_val(val, GGUFValueType.INT32)
    # 添加一个浮点数类型的键值对到数据结构中
    def add_float32(self, key: str, val: float) -> None:
        # 调用 add_key 方法添加键
        self.add_key(key)
        # 调用 add_val 方法添加值，并指定值的类型为 FLOAT32
        self.add_val(val, GGUFValueType.FLOAT32)
    
    # 添加一个无符号 64 位整数类型的键值对到数据结构中
    def add_uint64(self, key: str, val: int) -> None:
        # 调用 add_key 方法添加键
        self.add_key(key)
        # 调用 add_val 方法添加值，并指定值的类型为 UINT64
        self.add_val(val, GGUFValueType.UINT64)
    
    # 添加一个有符号 64 位整数类型的键值对到数据结构中
    def add_int64(self, key: str, val: int) -> None:
        # 调用 add_key 方法添加键
        self.add_key(key)
        # 调用 add_val 方法添加值，并指定值的类型为 INT64
        self.add_val(val, GGUFValueType.INT64)
    
    # 添加一个双精度浮点数类型的键值对到数据结构中
    def add_float64(self, key: str, val: float) -> None:
        # 调用 add_key 方法添加键
        self.add_key(key)
        # 调用 add_val 方法添加值，并指定值的类型为 FLOAT64
        self.add_val(val, GGUFValueType.FLOAT64)
    
    # 添加一个布尔类型的键值对到数据结构中
    def add_bool(self, key: str, val: bool) -> None:
        # 调用 add_key 方法添加键
        self.add_key(key)
        # 调用 add_val 方法添加值，并指定值的类型为 BOOL
        self.add_val(val, GGUFValueType.BOOL)
    
    # 添加一个字符串类型的键值对到数据结构中
    def add_string(self, key: str, val: str) -> None:
        # 如果值为空，则直接返回
        if not val:
            return
        # 调用 add_key 方法添加键
        self.add_key(key)
        # 调用 add_val 方法添加值，并指定值的类型为 STRING
        self.add_val(val, GGUFValueType.STRING)
    
    # 添加一个数组类型的键值对到数据结构中
    def add_array(self, key: str, val: Sequence[Any]) -> None:
        # 如果值不是序列类型，则抛出数值错误
        if not isinstance(val, Sequence):
            raise ValueError("Value must be a sequence for array type")
        # 调用 add_key 方法添加键
        self.add_key(key)
        # 调用 add_val 方法添加值，并指定值的类型为 ARRAY
        self.add_val(val, GGUFValueType.ARRAY)
    # 添加值到 GGUF 数据中
    def add_val(self, val: Any, vtype: GGUFValueType | None = None, add_vtype: bool = True) -> None:
        # 如果值类型为空，则获取值的类型
        if vtype is None:
            vtype = GGUFValueType.get_type(val)

        # 如果需要添加值类型，则将值类型打包到 kv_data 中，并增加计数
        if add_vtype:
            self.kv_data += self._pack("I", vtype)
            self.kv_data_count += 1

        # 根据值类型获取打包格式
        pack_fmt = self._simple_value_packing.get(vtype)
        # 如果存在打包格式，则将值打包到 kv_data 中
        if pack_fmt is not None:
            self.kv_data += self._pack(pack_fmt, val, skip_pack_prefix = vtype == GGUFValueType.BOOL)
        # 如果值类型为字符串，则将字符串编码为 utf8，并将长度和编码后的值打包到 kv_data 中
        elif vtype == GGUFValueType.STRING:
            encoded_val = val.encode("utf8") if isinstance(val, str) else val
            self.kv_data += self._pack("Q", len(encoded_val))
            self.kv_data += encoded_val
        # 如果值类型为数组，并且值为序列且不为空，则将数组信息打包到 kv_data 中
        elif vtype == GGUFValueType.ARRAY and isinstance(val, Sequence) and val:
            ltype = GGUFValueType.get_type(val[0])
            # 如果数组中的所有元素类型不一致，则抛出异常
            if not all(GGUFValueType.get_type(i) is ltype for i in val[1:]):
                raise ValueError("All items in a GGUF array should be of the same type")
            self.kv_data += self._pack("I", ltype)
            self.kv_data += self._pack("Q", len(val))
            # 遍历数组中的每个元素，递归调用 add_val 方法添加到 kv_data 中
            for item in val:
                self.add_val(item, add_vtype=False)
        # 如果值类型不在上述情况中，则抛出异常
        else:
            raise ValueError("Invalid GGUF metadata value type or value")

    # 静态方法，用于计算 x 对 n 取整后的值
    @staticmethod
    def ggml_pad(x: int, n: int) -> int:
        return ((x + n - 1) // n) * n

    # 添加张量信息到 GGUF 数据中
    def add_tensor_info(
        self, name: str, tensor_shape: Sequence[int], tensor_dtype: np.dtype[np.float16] | np.dtype[np.float32],
        tensor_nbytes: int, raw_dtype: GGMLQuantizationType | None = None,
    # 定义一个方法，用于向输出文件中添加张量数据
    def add_tensor(
        self, name: str, tensor: np.ndarray[Any, Any], raw_shape: Sequence[int] | None = None,
        raw_dtype: GGMLQuantizationType | None = None,
    ) -> None:
        # 如果字节顺序为大端，则对张量进行字节交换
        if self.endianess == GGUFEndian.BIG:
            tensor.byteswap(inplace=True)
        # 如果使用临时文件，并且临时文件为空，则创建一个临时文件
        if self.use_temp_file and self.temp_file is None:
            fp = tempfile.SpooledTemporaryFile(mode="w+b", max_size=256*1024*1024)
            fp.seek(0)
            self.temp_file = fp

        # 如果原始形状不为空，则使用原始形状，否则使用张量的形状
        shape: Sequence[int] = raw_shape if raw_shape is not None else tensor.shape
        # 调用add_tensor_info方法，添加张量信息到输出文件中
        self.add_tensor_info(name, shape, tensor.dtype, tensor.nbytes, raw_dtype = raw_dtype)

        # 如果临时文件为空，则将张量添加到张量列表中并返回
        if self.temp_file is None:
            self.tensors.append(tensor)
            return

        # 将张量数据写入临时文件
        tensor.tofile(self.temp_file)
        # 写入填充以保证数据对齐
        self.write_padding(self.temp_file, tensor.nbytes)
    # 写入填充数据到文件流中，使得数据长度满足对齐要求
    def write_padding(self, fp: IO[bytes], n: int, align: int | None = None) -> None:
        # 计算需要填充的字节数
        pad = GGUFWriter.ggml_pad(n, align if align is not None else self.data_alignment) - n
        # 如果需要填充，则写入相应数量的零字节
        if pad != 0:
            fp.write(bytes([0] * pad))
    
    # 将张量数据写入文件流中
    def write_tensor_data(self, tensor: np.ndarray[Any, Any]) -> None:
        # 检查状态是否为 TI_DATA
        if self.state is not WriterState.TI_DATA:
            raise ValueError(f'Expected output file to contain tensor info, got {self.state}')
        # 如果字节序为大端，则进行字节交换
        if self.endianess == GGUFEndian.BIG:
            tensor.byteswap(inplace=True)
        # 写入填充数据，使得数据长度满足对齐要求
        self.write_padding(self.fout, self.fout.tell())
        # 将张量数据写入文件流中
        tensor.tofile(self.fout)
        # 写入填充数据，使得数据长度满足对齐要求
        self.write_padding(self.fout, tensor.nbytes)
    
    # 将张量数据写入文件
    def write_tensors_to_file(self) -> None:
        # 将张量信息数据写入文件
        self.write_ti_data_to_file()
        # 写入填充数据，使得数据长度满足对齐要求
        self.write_padding(self.fout, self.fout.tell())
        # 如果临时文件为空，则将张量数据逐个写入文件流中
        if self.temp_file is None:
            while True:
                try:
                    tensor = self.tensors.pop(0)
                except IndexError:
                    break
                tensor.tofile(self.fout)
                self.write_padding(self.fout, tensor.nbytes)
            return
        # 如果临时文件不为空，则将临时文件内容拷贝到文件流中
        self.temp_file.seek(0)
        shutil.copyfileobj(self.temp_file, self.fout)
        self.flush()
        self.temp_file.close()
    
    # 刷新文件流
    def flush(self) -> None:
        self.fout.flush()
    
    # 关闭文件流
    def close(self) -> None:
        self.fout.close()
    
    # 添加架构信息
    def add_architecture(self) -> None:
        self.add_string(Keys.General.ARCHITECTURE, self.arch)
    
    # 添加作者信息
    def add_author(self, author: str) -> None:
        self.add_string(Keys.General.AUTHOR, author)
    
    # 添加张量数据布局信息
    def add_tensor_data_layout(self, layout: str) -> None:
        self.add_string(Keys.LLM.TENSOR_DATA_LAYOUT.format(arch=self.arch), layout)
    
    # 添加 URL 信息
    def add_url(self, url: str) -> None:
        self.add_string(Keys.General.URL, url)
    
    # 添加描述信息
    def add_description(self, description: str) -> None:
        self.add_string(Keys.General.DESCRIPTION, description)
    # 添加源URL到数据中
    def add_source_url(self, url: str) -> None:
        self.add_string(Keys.General.SOURCE_URL, url)
    
    # 添加源HF仓库到数据中
    def add_source_hf_repo(self, repo: str) -> None:
        self.add_string(Keys.General.SOURCE_HF_REPO, repo)
    
    # 添加文件类型到数据中
    def add_file_type(self, ftype: int) -> None:
        self.add_uint32(Keys.General.FILE_TYPE, ftype)
    
    # 添加名称到数据中
    def add_name(self, name: str) -> None:
        self.add_string(Keys.General.NAME, name)
    
    # 添加量化版本到数据中
    def add_quantization_version(self, quantization_version: GGMLQuantizationType) -> None:
        self.add_uint32(Keys.General.QUANTIZATION_VERSION, quantization_version)
    
    # 添加自定义对齐到数据中
    def add_custom_alignment(self, alignment: int) -> None:
        self.data_alignment = alignment
        self.add_uint32(Keys.General.ALIGNMENT, alignment)
    
    # 添加上下文长度到数据中
    def add_context_length(self, length: int) -> None:
        self.add_uint32(Keys.LLM.CONTEXT_LENGTH.format(arch=self.arch), length)
    
    # 添加嵌入长度到数据中
    def add_embedding_length(self, length: int) -> None:
        self.add_uint32(Keys.LLM.EMBEDDING_LENGTH.format(arch=self.arch), length)
    
    # 添加块数量到数据中
    def add_block_count(self, length: int) -> None:
        self.add_uint32(Keys.LLM.BLOCK_COUNT.format(arch=self.arch), length)
    
    # 添加前馈长度到数据中
    def add_feed_forward_length(self, length: int) -> None:
        self.add_uint32(Keys.LLM.FEED_FORWARD_LENGTH.format(arch=self.arch), length)
    
    # 添加并行残差到数据中
    def add_parallel_residual(self, use: bool) -> None:
        self.add_bool(Keys.LLM.USE_PARALLEL_RESIDUAL.format(arch=self.arch), use)
    
    # 添加头数量到数据中
    def add_head_count(self, count: int) -> None:
        self.add_uint32(Keys.Attention.HEAD_COUNT.format(arch=self.arch), count)
    
    # 添加键值头数量到数据中
    def add_head_count_kv(self, count: int) -> None:
        self.add_uint32(Keys.Attention.HEAD_COUNT_KV.format(arch=self.arch), count)
    
    # 添加最大偏差到数据中
    def add_max_alibi_bias(self, bias: float) -> None:
        self.add_float32(Keys.Attention.MAX_ALIBI_BIAS.format(arch=self.arch), bias)
    
    # 添加限制KQV到数据中
    def add_clamp_kqv(self, value: float) -> None:
        self.add_float32(Keys.Attention.CLAMP_KQV.format(arch=self.arch), value)
    # 添加 Layer Normalization 的 epsilon 值
    def add_layer_norm_eps(self, value: float) -> None:
        self.add_float32(Keys.Attention.LAYERNORM_EPS.format(arch=self.arch), value)
    
    # 添加 Layer Normalization 的 RMS epsilon 值
    def add_layer_norm_rms_eps(self, value: float) -> None:
        self.add_float32(Keys.Attention.LAYERNORM_RMS_EPS.format(arch=self.arch), value)
    
    # 添加 Rope 的维度数量
    def add_rope_dimension_count(self, count: int) -> None:
        self.add_uint32(Keys.Rope.DIMENSION_COUNT.format(arch=self.arch), count)
    
    # 添加 Rope 的频率基数
    def add_rope_freq_base(self, value: float) -> None:
        self.add_float32(Keys.Rope.FREQ_BASE.format(arch=self.arch), value)
    
    # 添加 Rope 的缩放类型
    def add_rope_scaling_type(self, value: RopeScalingType) -> None:
        self.add_string(Keys.Rope.SCALING_TYPE.format(arch=self.arch), value.value)
    
    # 添加 Rope 的缩放因子
    def add_rope_scaling_factor(self, value: float) -> None:
        self.add_float32(Keys.Rope.SCALING_FACTOR.format(arch=self.arch), value)
    
    # 添加 Rope 的原始上下文长度
    def add_rope_scaling_orig_ctx_len(self, value: int) -> None:
        self.add_uint32(Keys.Rope.SCALING_ORIG_CTX_LEN.format(arch=self.arch), value)
    
    # 添加 Rope 的微调标志
    def add_rope_scaling_finetuned(self, value: bool) -> None:
        self.add_bool(Keys.Rope.SCALING_FINETUNED.format(arch=self.arch), value)
    
    # 添加 Tokenizer 的模型
    def add_tokenizer_model(self, model: str) -> None:
        self.add_string(Keys.Tokenizer.MODEL, model)
    
    # 添加 Token 列表
    def add_token_list(self, tokens: Sequence[str] | Sequence[bytes] | Sequence[bytearray]) -> None:
        self.add_array(Keys.Tokenizer.LIST, tokens)
    
    # 添加 Token 合并
    def add_token_merges(self, merges: Sequence[str] | Sequence[bytes] | Sequence[bytearray]) -> None:
        self.add_array(Keys.Tokenizer.MERGES, merges)
    
    # 添加 Token 类型
    def add_token_types(self, types: Sequence[TokenType] | Sequence[int]) -> None:
        self.add_array(Keys.Tokenizer.TOKEN_TYPE, types)
    
    # 添加 Token 分数
    def add_token_scores(self, scores: Sequence[float]) -> None:
        self.add_array(Keys.Tokenizer.SCORES, scores)
    
    # 添加 BOS（Beginning of Sentence）Token 的 ID
    def add_bos_token_id(self, id: int) -> None:
        self.add_uint32(Keys.Tokenizer.BOS_ID, id)
    # 添加结束标记的标识符到配置文件中
    def add_eos_token_id(self, id: int) -> None:
        self.add_uint32(Keys.Tokenizer.EOS_ID, id)

    # 添加未知标记的标识符到配置文件中
    def add_unk_token_id(self, id: int) -> None:
        self.add_uint32(Keys.Tokenizer.UNK_ID, id)

    # 添加分隔标记的标识符到配置文件中
    def add_sep_token_id(self, id: int) -> None:
        self.add_uint32(Keys.Tokenizer.SEP_ID, id)

    # 添加填充标记的标识符到配置文件中
    def add_pad_token_id(self, id: int) -> None:
        self.add_uint32(Keys.Tokenizer.PAD_ID, id)

    # 添加是否添加开始标记的布尔值到配置文件中
    def add_add_bos_token(self, value: bool) -> None:
        self.add_bool(Keys.Tokenizer.ADD_BOS, value)

    # 添加是否添加结束标记的布尔值到配置文件中
    def add_add_eos_token(self, value: bool) -> None:
        self.add_bool(Keys.Tokenizer.ADD_EOS, value)

    # 添加稀疏阈值到配置文件中
    def add_sparse_threshold(self, value: float) -> None:
        self.add_float32(Keys.PowerInfer.SPARSE_THRESHOLD, value)

    # 封装数据并返回字节流
    def _pack(self, fmt: str, value: Any, skip_pack_prefix: bool = False) -> bytes:
        pack_prefix = ''
        if not skip_pack_prefix:
            pack_prefix = '<' if self.endianess == GGUFEndian.LITTLE else '>'
        return struct.pack(f'{pack_prefix}{fmt}', value)

    # 将封装好的数据写入文件
    def _write_packed(self, fmt: str, value: Any, skip_pack_prefix: bool = False) -> None:
        self.fout.write(self._pack(fmt, value, skip_pack_prefix))
```