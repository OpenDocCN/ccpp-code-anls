# `PowerInfer\gguf-py\gguf\gguf_writer.py`

```
# 导入必要的模块和库
from __future__ import annotations  # 导入未来版本的注解特性

import os  # 导入操作系统模块
import shutil  # 导入文件操作模块
import struct  # 导入处理字节数据的模块
import tempfile  # 导入临时文件模块
from enum import Enum, auto  # 导入枚举类型和自动编号功能
from io import BufferedWriter  # 导入缓冲写入流
from typing import IO, Any, Sequence  # 导入类型提示相关的模块

import numpy as np  # 导入数值计算库

from .constants import (  # 从常量模块中导入特定的常量
    GGUF_DEFAULT_ALIGNMENT,  # 导入默认对齐方式
    GGUF_MAGIC,  # 导入魔术数
    GGUF_VERSION,  # 导入版本号
    GGMLQuantizationType,  # 导入量化类型
    GGUFEndian,  # 导入字节序
    GGUFValueType,  # 导入值类型
    Keys,  # 导入键值
# 导入需要的模块和类
from enum import Enum, auto
from typing import Any, Union
import tempfile
from io import BufferedWriter
import numpy as np

# 定义枚举类型
class RopeScalingType(Enum):
    ...

class TokenType(Enum):
    ...

# 定义写入器状态枚举
class WriterState(Enum):
    EMPTY   = auto()  # 空状态
    HEADER  = auto()  # 头部状态
    KV_DATA = auto()  # 键值数据状态
    TI_DATA = auto()  # TI数据状态

# 定义GGUFWriter类
class GGUFWriter:
    fout: BufferedWriter  # 文件输出流
    temp_file: tempfile.SpooledTemporaryFile[bytes] | None  # 临时文件
    tensors: list[np.ndarray[Any, Any]]  # 张量列表
    _simple_value_packing = {  # 简单数值打包方式
        GGUFValueType.UINT8:   "B",  # 无符号8位整数
        GGUFValueType.INT8:    "b",  # 有符号8位整数
        GGUFValueType.UINT16:  "H",  # 无符号16位整数
# 定义一个字典，将 GGUFValueType 枚举类型映射到对应的格式字符串
{
    GGUFValueType.INT16:   "h",      # 16 位有符号整数
    GGUFValueType.UINT32:  "I",      # 32 位无符号整数
    GGUFValueType.INT32:   "i",      # 32 位有符号整数
    GGUFValueType.FLOAT32: "f",      # 32 位浮点数
    GGUFValueType.UINT64:  "Q",      # 64 位无符号整数
    GGUFValueType.INT64:   "q",      # 64 位有符号整数
    GGUFValueType.FLOAT64: "d",      # 64 位浮点数
    GGUFValueType.BOOL:    "?",      # 布尔值
}

# 初始化函数，接受文件路径、架构、是否使用临时文件和字节序参数
def __init__(
    self, path: os.PathLike[str] | str, arch: str, use_temp_file: bool = True,
    endianess: GGUFEndian = GGUFEndian.LITTLE,
):
    # 打开文件准备写入
    self.fout = open(path, "wb")
    # 存储架构信息
    self.arch = arch
    # 存储字节序信息
    self.endianess = endianess
    # 初始化数据偏移量
    self.offset_tensor = 0
    # 数据对齐方式，默认为 GGUF_DEFAULT_ALIGNMENT
    self.data_alignment = GGUF_DEFAULT_ALIGNMENT
    # 键值对数据，初始化为空字节数组
    self.kv_data = bytearray()
# 初始化键值数据计数器
self.kv_data_count = 0
# 初始化存储数据的字节数组
self.ti_data = bytearray()
# 初始化数据计数器
self.ti_data_count = 0
# 设置是否使用临时文件的标志
self.use_temp_file = use_temp_file
# 初始化临时文件
self.temp_file = None
# 初始化张量列表
self.tensors = []
# 打印消息，指示 GGUF 文件的字节顺序
print("gguf: This GGUF file is for {0} Endian only".format(
    "Big" if self.endianess == GGUFEndian.BIG else "Little",
))
# 设置写入状态为EMPTY
self.state = WriterState.EMPTY

# 调用add_architecture方法
self.add_architecture()

# 写入文件头部信息
def write_header_to_file(self) -> None:
    # 检查写入状态是否为空，如果不为空则抛出异常
    if self.state is not WriterState.EMPTY:
        raise ValueError(f'Expected output file to be empty, got {self.state}')

    # 写入GGUF_MAGIC值
    self._write_packed("<I", GGUF_MAGIC, skip_pack_prefix = True)
    # 写入GGUF_VERSION值
    self._write_packed("I", GGUF_VERSION)
    # 写入ti_data_count值
    self._write_packed("Q", self.ti_data_count)
# 使用指定格式将键值数据的数量写入文件
self._write_packed("Q", self.kv_data_count)
# 刷新文件缓冲区
self.flush()
# 设置状态为HEADER，表示已经写入了头部信息
self.state = WriterState.HEADER

# 将键值数据写入文件
def write_kv_data_to_file(self) -> None:
    # 如果状态不是HEADER，抛出数值错误
    if self.state is not WriterState.HEADER:
        raise ValueError(f'Expected output file to contain the header, got {self.state}')

    # 将键值数据写入文件
    self.fout.write(self.kv_data)
    # 刷新文件缓冲区
    self.flush()
    # 设置状态为KV_DATA，表示已经写入了键值数据
    self.state = WriterState.KV_DATA

# 将时间数据写入文件
def write_ti_data_to_file(self) -> None:
    # 如果状态不是KV_DATA，抛出数值错误
    if self.state is not WriterState.KV_DATA:
        raise ValueError(f'Expected output file to contain KV data, got {self.state}')

    # 将时间数据写入文件
    self.fout.write(self.ti_data)
    # 刷新文件缓冲区
    self.flush()
    # 设置状态为TI_DATA，表示已经写入了时间数据
    self.state = WriterState.TI_DATA
# 添加一个字符串类型的键到数据结构中
def add_key(self, key: str) -> None:
    # 调用 add_val 方法，添加一个字符串类型的值到数据结构中
    self.add_val(key, GGUFValueType.STRING, add_vtype=False)

# 添加一个 8 位无符号整数类型的键值对到数据结构中
def add_uint8(self, key: str, val: int) -> None:
    # 调用 add_key 方法，添加一个字符串类型的键到数据结构中
    self.add_key(key)
    # 调用 add_val 方法，添加一个 8 位无符号整数类型的值到数据结构中
    self.add_val(val, GGUFValueType.UINT8)

# 添加一个 8 位有符号整数类型的键值对到数据结构中
def add_int8(self, key: str, val: int) -> None:
    # 调用 add_key 方法，添加一个字符串类型的键到数据结构中
    self.add_key(key)
    # 调用 add_val 方法，添加一个 8 位有符号整数类型的值到数据结构中
    self.add_val(val, GGUFValueType.INT8)

# 添加一个 16 位无符号整数类型的键值对到数据结构中
def add_uint16(self, key: str, val: int) -> None:
    # 调用 add_key 方法，添加一个字符串类型的键到数据结构中
    self.add_key(key)
    # 调用 add_val 方法，添加一个 16 位无符号整数类型的值到数据结构中
    self.add_val(val, GGUFValueType.UINT16)

# 添加一个 16 位有符号整数类型的键值对到数据结构中
def add_int16(self, key: str, val: int) -> None:
    # 调用 add_key 方法，添加一个字符串类型的键到数据结构中
    self.add_key(key)
    # 调用 add_val 方法，添加一个 16 位有符号整数类型的值到数据结构中
    self.add_val(val, GGUFValueType.INT16)

# 添加一个 32 位无符号整数类型的键值对到数据结构中
def add_uint32(self, key: str, val: int) -> None:
# 添加键值对中的键
self.add_key(key)
# 添加键值对中的值，并指定值的类型为无符号32位整数
self.add_val(val, GGUFValueType.UINT32)

# 添加键值对中的键
self.add_key(key)
# 添加键值对中的值，并指定值的类型为32位整数
self.add_val(val, GGUFValueType.INT32)

# 添加键值对中的键
self.add_key(key)
# 添加键值对中的值，并指定值的类型为32位浮点数
self.add_val(val, GGUFValueType.FLOAT32)

# 添加键值对中的键
self.add_key(key)
# 添加键值对中的值，并指定值的类型为无符号64位整数
self.add_val(val, GGUFValueType.UINT64)

# 添加键值对中的键
self.add_key(key)
# 添加键值对中的值，并指定值的类型为64位整数
self.add_val(val, GGUFValueType.INT64)

# 添加键值对中的键
self.add_key(key)
# 添加键值对中的值，并指定值的类型为64位浮点数
# 添加键值对中的键
self.add_key(key)
# 添加键值对中的值，值的类型为浮点数
self.add_val(val, GGUFValueType.FLOAT64)

# 添加键值对中的键，值的类型为布尔型
def add_bool(self, key: str, val: bool) -> None:
    self.add_key(key)
    self.add_val(val, GGUFValueType.BOOL)

# 添加键值对中的键，值的类型为字符串
def add_string(self, key: str, val: str) -> None:
    # 如果值为空，则直接返回
    if not val:
        return
    self.add_key(key)
    self.add_val(val, GGUFValueType.STRING)

# 添加键值对中的键，值的类型为数组
def add_array(self, key: str, val: Sequence[Any]) -> None:
    # 如果值不是序列类型，则抛出数值错误
    if not isinstance(val, Sequence):
        raise ValueError("Value must be a sequence for array type")

    self.add_key(key)
    self.add_val(val, GGUFValueType.ARRAY)
# 向键值对数据中添加值，可以指定值的类型和是否添加值的类型信息
def add_val(self, val: Any, vtype: GGUFValueType | None = None, add_vtype: bool = True) -> None:
    # 如果未指定值的类型，则根据值自动获取类型
    if vtype is None:
        vtype = GGUFValueType.get_type(val)

    # 如果需要添加值的类型信息，则将类型信息打包到键值对数据中
    if add_vtype:
        self.kv_data += self._pack("I", vtype)
        self.kv_data_count += 1

    # 根据值的类型选择合适的打包格式进行打包
    pack_fmt = self._simple_value_packing.get(vtype)
    if pack_fmt is not None:
        self.kv_data += self._pack(pack_fmt, val, skip_pack_prefix = vtype == GGUFValueType.BOOL)
    # 如果值的类型为字符串，则将字符串编码成 utf8 格式并打包长度和内容
    elif vtype == GGUFValueType.STRING:
        encoded_val = val.encode("utf8") if isinstance(val, str) else val
        self.kv_data += self._pack("Q", len(encoded_val))
        self.kv_data += encoded_val
    # 如果值的类型为数组，并且值是序列类型且不为空，则打包数组的元素类型和元素值
    elif vtype == GGUFValueType.ARRAY and isinstance(val, Sequence) and val:
        ltype = GGUFValueType.get_type(val[0])
        # 检查数组中所有元素是否为相同类型，如果不是则抛出异常
        if not all(GGUFValueType.get_type(i) is ltype for i in val[1:]):
            raise ValueError("All items in a GGUF array should be of the same type")
        self.kv_data += self._pack("I", ltype)
    # 将值的长度打包成无符号长整型，并添加到键值数据中
    self.kv_data += self._pack("Q", len(val))
    # 遍历值中的每个元素，并将其添加到键值数据中，不添加值类型
    for item in val:
        self.add_val(item, add_vtype=False)
    # 如果值的类型不符合要求，抛出数值错误异常
    else:
        raise ValueError("Invalid GGUF metadata value type or value")

# 静态方法，用于对给定的整数进行 GGML 填充
@staticmethod
def ggml_pad(x: int, n: int) -> int:
    return ((x + n - 1) // n) * n

# 添加张量信息到 GGUF 文件中
def add_tensor_info(
    self, name: str, tensor_shape: Sequence[int], tensor_dtype: np.dtype[np.float16] | np.dtype[np.float32],
    tensor_nbytes: int, raw_dtype: GGMLQuantizationType | None = None,
) -> None:
    # 如果文件状态不为空，抛出数值错误异常
    if self.state is not WriterState.EMPTY:
        raise ValueError(f'Expected output file to be empty, got {self.state}')
    # 如果原始数据类型为空且张量数据类型不是 np.float32 或 np.float16，抛出数值错误异常
    if raw_dtype is None and tensor_dtype not in (np.float32, np.float16):
        raise ValueError("Only F32 and F16 tensors are supported for now")
# 将文件名编码为 UTF-8 格式
encoded_name = name.encode("utf8")
# 将编码后的文件名长度打包到 ti_data 中
self.ti_data += self._pack("Q", len(encoded_name))
# 将编码后的文件名数据添加到 ti_data 中
self.ti_data += encoded_name
# 获取张量的维度
n_dims = len(tensor_shape)
# 将张量的维度打包到 ti_data 中
self.ti_data += self._pack("I", n_dims)
# 遍历张量的维度，将每个维度的大小打包到 ti_data 中
for i in range(n_dims):
    self.ti_data += self._pack("Q", tensor_shape[n_dims - 1 - i])
# 如果原始数据类型为 None，则根据张量数据类型确定数据类型
if raw_dtype is None:
    dtype = GGMLQuantizationType.F32 if tensor_dtype == np.float32 else GGMLQuantizationType.F16
else:
    dtype = raw_dtype
# 将数据类型打包到 ti_data 中
self.ti_data += self._pack("I", dtype)
# 将张量的偏移量打包到 ti_data 中
self.ti_data += self._pack("Q", self.offset_tensor)
# 更新张量的偏移量
self.offset_tensor += GGUFWriter.ggml_pad(tensor_nbytes, self.data_alignment)
# 更新 ti_data_count
self.ti_data_count += 1
# 如果数据的字节顺序是大端序，就进行字节交换
if self.endianess == GGUFEndian.BIG:
    tensor.byteswap(inplace=True)

# 如果使用临时文件并且临时文件为空，就创建一个临时文件
if self.use_temp_file and self.temp_file is None:
    fp = tempfile.SpooledTemporaryFile(mode="w+b", max_size=256*1024*1024)
    fp.seek(0)
    self.temp_file = fp

# 确定张量的形状，如果原始形状不为空，则使用原始形状，否则使用张量的形状
shape: Sequence[int] = raw_shape if raw_shape is not None else tensor.shape
# 添加张量信息到数据结构中
self.add_tensor_info(name, shape, tensor.dtype, tensor.nbytes, raw_dtype = raw_dtype)

# 如果没有临时文件，就将张量添加到数据结构中并返回
if self.temp_file is None:
    self.tensors.append(tensor)
    return

# 将张量数据写入临时文件
tensor.tofile(self.temp_file)
# 在临时文件中写入填充以对齐数据
self.write_padding(self.temp_file, tensor.nbytes)

# 写入填充数据以对齐数据
def write_padding(self, fp: IO[bytes], n: int, align: int | None = None) -> None:
    # 计算需要填充的字节数
    pad = GGUFWriter.ggml_pad(n, align if align is not None else self.data_alignment) - n
    # 如果需要填充，就进行填充
    if pad != 0:
    # 在文件中写入指定数量的零字节，用于填充数据
    fp.write(bytes([0] * pad))

    # 将张量数据写入文件
    def write_tensor_data(self, tensor: np.ndarray[Any, Any]) -> None:
        # 检查状态是否为写入张量数据
        if self.state is not WriterState.TI_DATA:
            raise ValueError(f'Expected output file to contain tensor info, got {self.state}')

        # 如果字节顺序为大端序，进行字节交换
        if self.endianess == GGUFEndian.BIG:
            tensor.byteswap(inplace=True)
        # 写入填充数据
        self.write_padding(self.fout, self.fout.tell())
        # 将张量数据写入文件
        tensor.tofile(self.fout)
        # 写入填充数据
        self.write_padding(self.fout, tensor.nbytes)

    # 将张量数据写入文件
    def write_tensors_to_file(self) -> None:
        # 将张量信息数据写入文件
        self.write_ti_data_to_file()

        # 写入填充数据
        self.write_padding(self.fout, self.fout.tell())

        # 如果临时文件为空
        if self.temp_file is None:
            # 循环直到条件满足
            while True:
                try:
    # 从 self.tensors 列表中弹出第一个张量对象
    tensor = self.tensors.pop(0)
    # 如果 self.tensors 列表为空，抛出 IndexError 异常并结束循环
    except IndexError:
        break
    # 将张量对象的数据写入 self.fout 文件
    tensor.tofile(self.fout)
    # 写入填充数据，使得数据长度为 tensor.nbytes
    self.write_padding(self.fout, tensor.nbytes)
    # 返回

    # 将临时文件的指针移动到文件开头
    self.temp_file.seek(0)
    # 将临时文件的内容复制到 self.fout 文件
    shutil.copyfileobj(self.temp_file, self.fout)
    # 刷新 self.fout 文件
    self.flush()
    # 关闭临时文件
    self.temp_file.close()

# 刷新 self.fout 文件
def flush(self) -> None:
    self.fout.flush()

# 关闭 self.fout 文件
def close(self) -> None:
    self.fout.close()

# 添加架构信息
def add_architecture(self) -> None:
# 添加架构信息到通用键 ARCHITECTURE
self.add_string(Keys.General.ARCHITECTURE, self.arch)

# 添加作者信息到通用键 AUTHOR
def add_author(self, author: str) -> None:
    self.add_string(Keys.General.AUTHOR, author)

# 添加张量数据布局信息到LLM键下的特定架构
def add_tensor_data_layout(self, layout: str) -> None:
    self.add_string(Keys.LLM.TENSOR_DATA_LAYOUT.format(arch=self.arch), layout)

# 添加 URL 到通用键 URL
def add_url(self, url: str) -> None:
    self.add_string(Keys.General.URL, url)

# 添加描述信息到通用键 DESCRIPTION
def add_description(self, description: str) -> None:
    self.add_string(Keys.General.DESCRIPTION, description)

# 添加源 URL 到通用键 SOURCE_URL
def add_source_url(self, url: str) -> None:
    self.add_string(Keys.General.SOURCE_URL, url)

# 添加源 HF 仓库信息到通用键 SOURCE_HF_REPO
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

# 添加自定义对齐方式到数据中
def add_custom_alignment(self, alignment: int) -> None:
    # 设置数据对齐方式
    self.data_alignment = alignment
    # 添加对齐方式到数据中
    self.add_uint32(Keys.General.ALIGNMENT, alignment)

# 添加上下文长度到数据中
def add_context_length(self, length: int) -> None:
    # 添加上下文长度到数据中，使用特定的架构格式化键
    self.add_uint32(Keys.LLM.CONTEXT_LENGTH.format(arch=self.arch), length)

# 添加嵌入长度到数据中
def add_embedding_length(self, length: int) -> None:
    # 添加嵌入长度到数据中，使用特定的架构格式化键
    self.add_uint32(Keys.LLM.EMBEDDING_LENGTH.format(arch=self.arch), length)
# 添加一个整数类型的数据到指定的键值对中
def add_block_count(self, length: int) -> None:
    self.add_uint32(Keys.LLM.BLOCK_COUNT.format(arch=self.arch), length)

# 添加一个整数类型的数据到指定的键值对中
def add_feed_forward_length(self, length: int) -> None:
    self.add_uint32(Keys.LLM.FEED_FORWARD_LENGTH.format(arch=self.arch), length)

# 添加一个布尔类型的数据到指定的键值对中
def add_parallel_residual(self, use: bool) -> None:
    self.add_bool(Keys.LLM.USE_PARALLEL_RESIDUAL.format(arch=self.arch), use)

# 添加一个整数类型的数据到指定的键值对中
def add_head_count(self, count: int) -> None:
    self.add_uint32(Keys.Attention.HEAD_COUNT.format(arch=self.arch), count)

# 添加一个整数类型的数据到指定的键值对中
def add_head_count_kv(self, count: int) -> None:
    self.add_uint32(Keys.Attention.HEAD_COUNT_KV.format(arch=self.arch), count)

# 添加一个浮点数类型的数据到指定的键值对中
def add_max_alibi_bias(self, bias: float) -> None:
    self.add_float32(Keys.Attention.MAX_ALIBI_BIAS.format(arch=self.arch), bias)

# 添加一个浮点数类型的数据到指定的键值对中
def add_clamp_kqv(self, value: float) -> None:
    self.add_float32(Keys.Attention.CLAMP_KQV.format(arch=self.arch), value)
# 添加层归一化的 epsilon 值
def add_layer_norm_eps(self, value: float) -> None:
    self.add_float32(Keys.Attention.LAYERNORM_EPS.format(arch=self.arch), value)

# 添加层归一化的 RMS epsilon 值
def add_layer_norm_rms_eps(self, value: float) -> None:
    self.add_float32(Keys.Attention.LAYERNORM_RMS_EPS.format(arch=self.arch), value)

# 添加绳索的维度数量
def add_rope_dimension_count(self, count: int) -> None:
    self.add_uint32(Keys.Rope.DIMENSION_COUNT.format(arch=self.arch), count)

# 添加绳索的基础频率
def add_rope_freq_base(self, value: float) -> None:
    self.add_float32(Keys.Rope.FREQ_BASE.format(arch=self.arch), value)

# 添加绳索的缩放类型
def add_rope_scaling_type(self, value: RopeScalingType) -> None:
    self.add_string(Keys.Rope.SCALING_TYPE.format(arch=self.arch), value.value)

# 添加绳索的缩放因子
def add_rope_scaling_factor(self, value: float) -> None:
    self.add_float32(Keys.Rope.SCALING_FACTOR.format(arch=self.arch), value)

# 添加绳索的原始上下文长度
def add_rope_scaling_orig_ctx_len(self, value: int) -> None:
# 添加一个无符号32位整数值到指定的键
def add_uint32(Keys.Rope.SCALING_ORIG_CTX_LEN.format(arch=self.arch), value):
    self.add_uint32(Keys.Rope.SCALING_ORIG_CTX_LEN.format(arch=self.arch), value)

# 添加一个布尔值到指定的键
def add_rope_scaling_finetuned(self, value: bool) -> None:
    self.add_bool(Keys.Rope.SCALING_FINETUNED.format(arch=self.arch), value)

# 添加一个字符串型数值到指定的键
def add_tokenizer_model(self, model: str) -> None:
    self.add_string(Keys.Tokenizer.MODEL, model)

# 添加一个 token 列表到指定的键
def add_token_list(self, tokens: Sequence[str] | Sequence[bytes] | Sequence[bytearray]) -> None:
    self.add_array(Keys.Tokenizer.LIST, tokens)

# 添加一个 token 合并列表到指定的键
def add_token_merges(self, merges: Sequence[str] | Sequence[bytes] | Sequence[bytearray]) -> None:
    self.add_array(Keys.Tokenizer.MERGES, merges)

# 添加一个 token 类型列表到指定的键
def add_token_types(self, types: Sequence[TokenType] | Sequence[int]) -> None:
    self.add_array(Keys.Tokenizer.TOKEN_TYPE, types)

# 添加一个 token 分数列表到指定的键
def add_token_scores(self, scores: Sequence[float]) -> None:
    self.add_array(Keys.Tokenizer.SCORES, scores)
# 添加开始标记的标记ID
def add_bos_token_id(self, id: int) -> None:
    self.add_uint32(Keys.Tokenizer.BOS_ID, id)

# 添加结束标记的标记ID
def add_eos_token_id(self, id: int) -> None:
    self.add_uint32(Keys.Tokenizer.EOS_ID, id)

# 添加未知标记的标记ID
def add_unk_token_id(self, id: int) -> None:
    self.add_uint32(Keys.Tokenizer.UNK_ID, id)

# 添加分隔标记的标记ID
def add_sep_token_id(self, id: int) -> None:
    self.add_uint32(Keys.Tokenizer.SEP_ID, id)

# 添加填充标记的标记ID
def add_pad_token_id(self, id: int) -> None:
    self.add_uint32(Keys.Tokenizer.PAD_ID, id)

# 添加是否在开始处添加标记的布尔值
def add_add_bos_token(self, value: bool) -> None:
    self.add_bool(Keys.Tokenizer.ADD_BOS, value)

# 添加是否在结束处添加标记的布尔值
def add_add_eos_token(self, value: bool) -> None:
    self.add_bool(Keys.Tokenizer.ADD_EOS, value)
# 添加稀疏阈值，将给定的浮点数值添加到指定的键中
def add_sparse_threshold(self, value: float) -> None:
    self.add_float32(Keys.PowerInfer.SPARSE_THRESHOLD, value)

# 将给定的值按照指定的格式进行打包，并返回打包后的字节流
def _pack(self, fmt: str, value: Any, skip_pack_prefix: bool = False) -> bytes:
    pack_prefix = ''
    # 如果不跳过打包前缀，则根据字节顺序设置打包前缀
    if not skip_pack_prefix:
        pack_prefix = '<' if self.endianess == GGUFEndian.LITTLE else '>'
    return struct.pack(f'{pack_prefix}{fmt}', value)

# 将给定的值按照指定的格式进行打包，并将打包后的字节流写入输出文件
def _write_packed(self, fmt: str, value: Any, skip_pack_prefix: bool = False) -> None:
    self.fout.write(self._pack(fmt, value, skip_pack_prefix))
```