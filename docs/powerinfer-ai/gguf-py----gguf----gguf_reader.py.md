# `PowerInfer\gguf-py\gguf\gguf_reader.py`

```
# 导入 GGUF 文件读取/修改支持的模块
# 有关 API 使用信息，请参阅 scripts/ 目录中的一些相当简单的示例
from __future__ import annotations

# 导入所需的模块
import os
from collections import OrderedDict
from typing import Any, Literal, NamedTuple, TypeVar, Union

# 导入 numpy 库
import numpy as np
import numpy.typing as npt

# 如果作为主程序运行
if __name__ == "__main__":
    # 导入 sys 和 Path 模块
    import sys
    from pathlib import Path

    # 允许在包中运行文件作为脚本
    sys.path.insert(0, str(Path(__file__).parent.parent))
# 从 gguf.constants 模块中导入常量
from gguf.constants import (
    GGML_QUANT_SIZES,
    GGUF_DEFAULT_ALIGNMENT,
    GGUF_MAGIC,
    GGUF_VERSION,
    GGMLQuantizationType,
    GGUFValueType,
)

# 定义一个列表，包含读取器支持的版本号
READER_SUPPORTED_VERSIONS = [2, GGUF_VERSION]

# 定义一个命名元组，表示读取器的字段
class ReaderField(NamedTuple):
    # 字段的起始偏移量
    offset: int

    # 字段的名称（不一定来自文件数据）
    name: str
# 数据部分。一些类型有多个组件，比如由长度和字符串数据组成的字符串。
parts: list[npt.NDArray[Any]] = []  # 用于存储数据部分的列表

# 指向我们可以调用的实际数据的部分索引。例如，字符串数组将被填充为指向实际字符串数据的索引。
data: list[int] = [-1]  # 用于存储实际数据索引的列表，初始值为-1

types: list[GGUFValueType] = []  # 用于存储数据类型的列表


class ReaderTensor(NamedTuple):
    name: str  # 张量的名称
    tensor_type: GGMLQuantizationType  # 张量的量化类型
    shape: npt.NDArray[np.uint32]  # 张量的形状
    n_elements: int  # 张量的元素数量
    n_bytes: int  # 张量的字节数
    data_offset: int  # 数据的偏移量
    data: npt.NDArray[Any]  # 张量的数据
# 定义一个类 GGUFReader
class GGUFReader:
    # 定义属性 byte_order，表示字节顺序，默认为 'I'，即与主机字节顺序相同
    # 或者 'S'，表示与主机字节顺序相反
    byte_order: Literal['I' | 'S'] = 'I'
    # 定义属性 alignment，表示对齐方式，默认为 GGUF_DEFAULT_ALIGNMENT
    alignment: int = GGUF_DEFAULT_ALIGNMENT

    # 内部辅助函数，API 可能会改变
    # 定义字典 gguf_scalar_to_np，将 GGUFValueType 映射为对应的 numpy 类型
    gguf_scalar_to_np: dict[GGUFValueType, type[np.generic]] = {
        GGUFValueType.UINT8:   np.uint8,
        GGUFValueType.INT8:    np.int8,
        GGUFValueType.UINT16:  np.uint16,
        GGUFValueType.INT16:   np.int16,
        GGUFValueType.UINT32:  np.uint32,
        GGUFValueType.INT32:   np.int32,
        GGUFValueType.FLOAT32: np.float32,
        GGUFValueType.UINT64:  np.uint64,
        GGUFValueType.INT64:   np.int64,
        GGUFValueType.FLOAT64: np.float64,
    }
# 定义一个字典，将 GGUFValueType.BOOL 映射为 np.bool_ 类型
GGUF_VALUE_TYPE_MAP = {
    GGUFValueType.BOOL:    np.bool_,
}

# 初始化方法，接受一个文件路径和模式参数，默认为 'r'
def __init__(self, path: os.PathLike[str] | str, mode: Literal['r' | 'r+' | 'c'] = 'r'):
    # 使用 np.memmap 创建一个内存映射文件对象，并将其赋值给 self.data
    self.data = np.memmap(path, mode = mode)
    offs = 0
    # 检查文件头部的 GGUF_MAGIC 是否有效，如果不是则抛出 ValueError 异常
    if self._get(offs, np.uint32, override_order = '<')[0] != GGUF_MAGIC:
        raise ValueError('GGUF magic invalid')
    offs += 4
    # 读取文件版本信息
    temp_version = self._get(offs, np.uint32)
    # 如果文件版本信息的低 16 位为 0，则表示文件可能是为相反字节顺序的机器创建的
    if temp_version[0] & 65535 == 0:
        # 设置字节顺序为 'S'，并将版本信息转换为当前字节顺序
        self.byte_order = 'S'
        temp_version = temp_version.newbyteorder(self.byte_order)
    version = temp_version[0]
    # 如果文件版本不在支持的版本列表中，则抛出 ValueError 异常
    if version not in READER_SUPPORTED_VERSIONS:
        raise ValueError(f'Sorry, file appears to be version {version} which we cannot handle')
    # 初始化一个有序字典用于存储字段信息
    self.fields: OrderedDict[str, ReaderField] = OrderedDict()
    # 初始化一个列表用于存储张量信息
    self.tensors: list[ReaderTensor] = []
# 增加偏移量并将字段推送到读取器中
offs += self._push_field(ReaderField(offs, 'GGUF.version', [temp_version], [0], [GGUFValueType.UINT32]))
# 从指定偏移量获取两个 np.uint64 类型的临时计数值
temp_counts = self._get(offs, np.uint64, 2)
# 增加偏移量并将字段推送到读取器中，用于存储张量数量
offs += self._push_field(ReaderField(offs, 'GGUF.tensor_count', [temp_counts[:1]], [0], [GGUFValueType.UINT64]))
# 增加偏移量并将字段推送到读取器中，用于存储键值对数量
offs += self._push_field(ReaderField(offs, 'GGUF.kv_count', [temp_counts[1:]], [0], [GGUFValueType.UINT64]))
# 将临时计数值分别赋值给张量数量和键值对数量
tensor_count, kv_count = temp_counts
# 构建键值对字段
offs = self._build_fields(offs, kv_count)
# 构建张量字段
offs, tensors_fields = self._build_tensors_fields(offs, tensor_count)
# 获取新的对齐值
new_align = self.fields.get('general.alignment')
# 如果存在新的对齐值
if new_align is not None:
    # 如果新的对齐值类型不是 UINT64
    if new_align.types != [GGUFValueType.UINT64]:
        # 抛出数值错误异常
        raise ValueError('Bad type for general.alignment field')
    # 将新的对齐值赋给对象的对齐属性
    self.alignment = new_align.parts[-1][0]
# 计算填充值
padding = offs % self.alignment
# 如果填充值不为 0
if padding != 0:
    # 增加偏移量以对齐
    offs += self.alignment - padding
# 构建张量
self._build_tensors(offs, tensors_fields)

# 定义一个类型变量 _DT，其类型为 npt.DTypeLike 的子类
_DT = TypeVar('_DT', bound = npt.DTypeLike)

# 通过键获取键值元数据字段
# 根据给定的键获取字段，如果不存在则返回 None
def get_field(self, key: str) -> Union[ReaderField, None]:
    return self.fields.get(key, None)

# 通过索引从列表中获取张量
def get_tensor(self, idx: int) -> ReaderTensor:
    return self.tensors[idx]

# 从数据中获取指定偏移量的内容，并按照指定的数据类型和数量返回数组
def _get(
    self, offset: int, dtype: npt.DTypeLike, count: int = 1, override_order: None | Literal['I' | 'S' | '<'] = None,
) -> npt.NDArray[Any]:
    count = int(count)
    itemsize = int(np.empty([], dtype = dtype).itemsize)
    end_offs = offset + itemsize * count
    return (
        self.data[offset:end_offs]
        .view(dtype = dtype)[:count]
        .newbyteorder(override_order or self.byte_order)
    )

# 将字段推送到数据中，并返回新的数据长度
def _push_field(self, field: ReaderField, skip_sum: bool = False) -> int:
    # 如果字段名已经存在于字段列表中，则抛出KeyError异常
    if field.name in self.fields:
        raise KeyError(f'Duplicate {field.name} already in list at offset {field.offset}')
    # 将字段名和字段对象添加到字段列表中
    self.fields[field.name] = field
    # 如果skip_sum为False，则返回字段部分的总字节数；否则返回0
    return 0 if skip_sum else sum(int(part.nbytes) for part in field.parts)

# 获取字符串数据
def _get_str(self, offset: int) -> tuple[npt.NDArray[np.uint64], npt.NDArray[np.uint8]]:
    # 获取字符串的长度
    slen = self._get(offset, np.uint64)
    # 返回字符串的长度和字符串的字节数组
    return slen, self._get(offset + 8, np.uint8, slen[0])

# 获取字段的部分数据
def _get_field_parts(
    self, orig_offs: int, raw_type: int,
) -> tuple[int, list[npt.NDArray[Any]], list[int], list[GGUFValueType]]:
    # 初始化偏移量
    offs = orig_offs
    # 初始化类型列表
    types: list[GGUFValueType] = []
    # 获取字段值的类型
    gtype = GGUFValueType(raw_type)
    # 将字段值的类型添加到类型列表中
    types.append(gtype)
    # 处理字符串类型的字段
    if gtype == GGUFValueType.STRING:
        # 获取字符串的各个部分
        sparts: list[npt.NDArray[Any]] = list(self._get_str(offs))
        # 计算字符串部分的总字节数
        size = sum(int(part.nbytes) for part in sparts)
        # 返回大小、部分、类型
        return size, sparts, [1], types
        # 检查是否为简单标量类型
        nptype = self.gguf_scalar_to_np.get(gtype)
        if nptype is not None:
            val = self._get(offs, nptype)
            return int(val.nbytes), [val], [0], types
        # 处理数组
        if gtype == GGUFValueType.ARRAY:
            # 获取数组元素类型
            raw_itype = self._get(offs, np.uint32)
            offs += int(raw_itype.nbytes)
            # 获取数组长度
            alen = self._get(offs, np.uint64)
            offs += int(alen.nbytes)
            # 初始化数组部分和数据索引
            aparts: list[npt.NDArray[Any]] = [raw_itype, alen]
            data_idxs: list[int] = []
            # 遍历数组元素
            for idx in range(alen[0]):
                # 获取当前字段的部分
                curr_size, curr_parts, curr_idxs, curr_types = self._get_field_parts(offs, raw_itype[0])
                if idx == 0:
                    types += curr_types
                # 记录当前部分的索引
                idxs_offs = len(aparts)
                aparts += curr_parts
    # 将当前索引加上偏移量，形成新的索引列表
    data_idxs += (idx + idxs_offs for idx in curr_idxs)
    # 偏移量增加当前大小
    offs += curr_size
    # 返回偏移量减去原始偏移量，aparts，数据索引，类型
    return offs - orig_offs, aparts, data_idxs, types
    # 无法处理未知/未处理的字段类型
    raise ValueError('Unknown/unhandled field type {gtype}')
    
# 获取张量数据
def _get_tensor(self, orig_offs: int) -> ReaderField:
    # 偏移量等于原始偏移量
    offs = orig_offs
    # 获取名称长度和名称数据
    name_len, name_data = self._get_str(offs)
    # 偏移量增加名称长度和名称数据的字节数
    offs += int(name_len.nbytes + name_data.nbytes)
    # 获取维度数
    n_dims = self._get(offs, np.uint32)
    # 偏移量增加维度数的字节数
    offs += int(n_dims.nbytes)
    # 获取维度
    dims = self._get(offs, np.uint64, n_dims[0])
    # 偏移量增加维度的字节数
    offs += int(dims.nbytes)
    # 获取原始数据类型
    raw_dtype = self._get(offs, np.uint32)
    # 偏移量增加原始数据类型的字节数
    offs += int(raw_dtype.nbytes)
    # 获取张量偏移量
    offset_tensor = self._get(offs, np.uint64)
    # 偏移量增加张量偏移量的字节数
    offs += int(offset_tensor.nbytes)
    # 返回读取的字段
    return ReaderField(
        orig_offs,
# 将字节数据转换为字符串，使用utf-8编码
str(bytes(name_data), encoding = 'utf-8'),

# 构建字段，根据偏移量和数量进行循环
def _build_fields(self, offs: int, count: int) -> int:

# 循环count次，每次处理一个字段
for _ in range(count):

# 保存原始偏移量
orig_offs = offs

# 获取键的长度和数据
kv_klen, kv_kdata = self._get_str(offs)

# 更新偏移量
offs += int(kv_klen.nbytes + kv_kdata.nbytes)

# 获取键值对类型
raw_kv_type = self._get(offs, np.uint32)

# 更新偏移量
offs += int(raw_kv_type.nbytes)

# 将键值对的长度、数据和类型存入列表
parts: list[npt.NDArray[Any]] = [kv_klen, kv_kdata, raw_kv_type]

# 记录字段的索引偏移量
idxs_offs = len(parts)

# 获取字段的大小、部分、索引和类型
field_size, field_parts, field_idxs, field_types = self._get_field_parts(offs, raw_kv_type[0])

# 将字段的部分添加到列表中
parts += field_parts

# 将字段信息压入字段列表
self._push_field(ReaderField(
    orig_offs,
    str(bytes(kv_kdata), encoding = 'utf-8'),
    parts,
# 对字段索引进行偏移，并生成新的索引列表
[idx + idxs_offs for idx in field_idxs],
# 生成字段类型列表
field_types,
# 跳过求和操作
), skip_sum = True)
# 偏移量增加字段大小
offs += field_size
# 返回偏移量
return offs

# 构建张量字段
def _build_tensors_fields(self, offs: int, count: int) -> tuple[int, list[ReaderField]]:
    tensor_fields = []
    # 遍历字段数量
    for _ in range(count):
        # 获取张量字段
        field = self._get_tensor(offs)
        # 偏移量增加字段部分的字节数总和
        offs += sum(int(part.nbytes) for part in field.parts)
        # 将字段添加到张量字段列表
        tensor_fields.append(field)
    # 返回偏移量和张量字段列表
    return offs, tensor_fields

# 构建张量
def _build_tensors(self, start_offs: int, fields: list[ReaderField]) -> None:
    tensors = []
    # 遍历字段列表
    for field in fields:
        # 获取字段部分的数据
        _name_len, name_data, _n_dims, dims, raw_dtype, offset_tensor = field.parts
        # 获取 GGML 类型
        ggml_type = GGMLQuantizationType(raw_dtype[0])
        # 计算张量元素数量
        n_elems = np.prod(dims)
# 从 GGML_QUANT_SIZES 字典中获取块大小和类型大小
block_size, type_size = GGML_QUANT_SIZES[ggml_type]
# 计算数据的字节数
n_bytes = n_elems * type_size // block_size
# 计算数据的偏移量
data_offs = int(start_offs + offset_tensor[0])
# 定义变量 item_type
item_type: npt.DTypeLike
# 根据 ggml_type 的值确定数据项的数量和类型
if ggml_type == GGMLQuantizationType.F32:
    item_count = n_elems
    item_type = np.float32
elif ggml_type == GGMLQuantizationType.F16:
    item_count = n_elems
    item_type = np.float16
else:
    item_count = n_bytes
    item_type = np.uint8
# 将读取的张量信息添加到列表中
tensors.append(ReaderTensor(
    name = str(bytes(name_data), encoding = 'utf-8'),
    tensor_type = ggml_type,
    shape = dims,
    n_elements = n_elems,
    n_bytes = n_bytes,
    data_offset = data_offs,
# 从指定偏移量和数据类型中获取数据，并赋值给变量data
data = self._get(data_offs, item_type, item_count),
# 将变量field赋值给字段field
field = field,
# 将字段field和数据data组成元组，并添加到列表中
))
# 将列表赋值给变量self.tensors
self.tensors = tensors
```