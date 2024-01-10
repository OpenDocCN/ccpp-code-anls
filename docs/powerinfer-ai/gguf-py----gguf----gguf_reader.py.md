# `PowerInfer\gguf-py\gguf\gguf_reader.py`

```
#
# GGUF file reading/modification support. For API usage information,
# please see the files scripts/ for some fairly simple examples.
#

# 导入必要的模块
from __future__ import annotations
import os
from collections import OrderedDict
from typing import Any, Literal, NamedTuple, TypeVar, Union
import numpy as np
import numpy.typing as npt

# 如果作为主程序运行，则导入 sys 和 Path 模块
if __name__ == "__main__":
    import sys
    from pathlib import Path

    # 允许将包中的文件作为脚本运行
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

# 支持的读取版本
READER_SUPPORTED_VERSIONS = [2, GGUF_VERSION]

# 定义一个命名元组 ReaderField
class ReaderField(NamedTuple):
    # 偏移量
    offset: int
    # 字段名
    name: str
    # 数据部分
    parts: list[npt.NDArray[Any]] = []
    # 数据索引
    data: list[int] = [-1]
    # 数据类型
    types: list[GGUFValueType] = []

# 定义一个命名元组 ReaderTensor
class ReaderTensor(NamedTuple):
    name: str
    tensor_type: GGMLQuantizationType
    shape: npt.NDArray[np.uint32]
    n_elements: int
    n_bytes: int
    data_offset: int
    data: npt.NDArray[Any]
    field: ReaderField

# 定义 GGUFReader 类
class GGUFReader:
    # 字节顺序
    byte_order: Literal['I' | 'S'] = 'I'
    # 对齐方式
    alignment: int = GGUF_DEFAULT_ALIGNMENT

    # 注意：内部辅助，API 可能会更改
    # 定义一个字典，将GGUFValueType枚举类型映射到对应的numpy数据类型
    gguf_scalar_to_np: dict[GGUFValueType, type[np.generic]] = {
        GGUFValueType.UINT8:   np.uint8,    # 将UINT8映射到numpy的uint8类型
        GGUFValueType.INT8:    np.int8,     # 将INT8映射到numpy的int8类型
        GGUFValueType.UINT16:  np.uint16,   # 将UINT16映射到numpy的uint16类型
        GGUFValueType.INT16:   np.int16,    # 将INT16映射到numpy的int16类型
        GGUFValueType.UINT32:  np.uint32,   # 将UINT32映射到numpy的uint32类型
        GGUFValueType.INT32:   np.int32,    # 将INT32映射到numpy的int32类型
        GGUFValueType.FLOAT32: np.float32,  # 将FLOAT32映射到numpy的float32类型
        GGUFValueType.UINT64:  np.uint64,   # 将UINT64映射到numpy的uint64类型
        GGUFValueType.INT64:   np.int64,    # 将INT64映射到numpy的int64类型
        GGUFValueType.FLOAT64: np.float64,  # 将FLOAT64映射到numpy的float64类型
        GGUFValueType.BOOL:    np.bool_,    # 将BOOL映射到numpy的bool_类型
    }
    # 初始化方法，接受文件路径和模式参数
    def __init__(self, path: os.PathLike[str] | str, mode: Literal['r' | 'r+' | 'c'] = 'r'):
        # 使用内存映射创建一个数组，将文件内容映射到内存中
        self.data = np.memmap(path, mode = mode)
        # 设置偏移量为0
        offs = 0
        # 检查文件头部的魔术数字是否有效
        if self._get(offs, np.uint32, override_order = '<')[0] != GGUF_MAGIC:
            raise ValueError('GGUF magic invalid')
        offs += 4
        # 读取版本号
        temp_version = self._get(offs, np.uint32)
        # 如果版本号为0，表示文件可能是为相反字节顺序的机器创建的
        if temp_version[0] & 65535 == 0:
            # 设置字节顺序为'S'
            self.byte_order = 'S'
            # 调整版本号的字节顺序
            temp_version = temp_version.newbyteorder(self.byte_order)
        version = temp_version[0]
        # 检查文件版本是否受支持
        if version not in READER_SUPPORTED_VERSIONS:
            raise ValueError(f'Sorry, file appears to be version {version} which we cannot handle')
        # 初始化字段字典
        self.fields: OrderedDict[str, ReaderField] = OrderedDict()
        # 初始化张量列表
        self.tensors: list[ReaderTensor] = []
        offs += self._push_field(ReaderField(offs, 'GGUF.version', [temp_version], [0], [GGUFValueType.UINT32]))
        # 读取张量和键值对的数量
        temp_counts = self._get(offs, np.uint64, 2)
        offs += self._push_field(ReaderField(offs, 'GGUF.tensor_count', [temp_counts[:1]], [0], [GGUFValueType.UINT64]))
        offs += self._push_field(ReaderField(offs, 'GGUF.kv_count', [temp_counts[1:]], [0], [GGUFValueType.UINT64]))
        tensor_count, kv_count = temp_counts
        # 构建字段
        offs = self._build_fields(offs, kv_count)
        # 构建张量字段
        offs, tensors_fields = self._build_tensors_fields(offs, tensor_count)
        # 获取新的对齐值
        new_align = self.fields.get('general.alignment')
        if new_align is not None:
            # 检查对齐值的类型是否正确
            if new_align.types != [GGUFValueType.UINT64]:
                raise ValueError('Bad type for general.alignment field')
            self.alignment = new_align.parts[-1][0]
        # 计算填充值
        padding = offs % self.alignment
        if padding != 0:
            offs += self.alignment - padding
        # 构建张量
        self._build_tensors(offs, tensors_fields)
    # 定义一个类型变量 _DT，它是 npt.DTypeLike 的子类型
    _DT = TypeVar('_DT', bound = npt.DTypeLike)

    # 通过键名获取元数据字段的键值对
    def get_field(self, key: str) -> Union[ReaderField, None]:
        return self.fields.get(key, None)

    # 通过索引从列表中获取张量
    def get_tensor(self, idx: int) -> ReaderTensor:
        return self.tensors[idx]

    # 从数据中获取指定偏移量、数据类型、数量的数据，并可以覆盖字节顺序
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

    # 将字段添加到列表中，如果 skip_sum 为 True，则跳过求和
    def _push_field(self, field: ReaderField, skip_sum: bool = False) -> int:
        if field.name in self.fields:
            raise KeyError(f'Duplicate {field.name} already in list at offset {field.offset}')
        self.fields[field.name] = field
        return 0 if skip_sum else sum(int(part.nbytes) for part in field.parts)

    # 从指定偏移量获取字符串数据
    def _get_str(self, offset: int) -> tuple[npt.NDArray[np.uint64], npt.NDArray[np.uint8]]:
        slen = self._get(offset, np.uint64)
        return slen, self._get(offset + 8, np.uint8, slen[0])

    # 获取字段的各个部分数据
    def _get_field_parts(
        self, orig_offs: int, raw_type: int,
    # 定义一个函数，接受参数并返回一个元组，包含整数、数组和值类型列表
    ) -> tuple[int, list[npt.NDArray[Any]], list[int], list[GGUFValueType]]:
        # 将原始偏移值赋给变量 offs
        offs = orig_offs
        # 创建一个空的值类型列表
        types: list[GGUFValueType] = []
        # 创建一个 GGUFValueType 对象，并添加到值类型列表中
        gtype = GGUFValueType(raw_type)
        types.append(gtype)
        # 处理字符串类型
        if gtype == GGUFValueType.STRING:
            # 调用 _get_str 方法获取字符串的部分，并存储在 sparts 列表中
            sparts: list[npt.NDArray[Any]] = list(self._get_str(offs))
            # 计算字符串部分的总大小
            size = sum(int(part.nbytes) for part in sparts)
            # 返回字符串部分的总大小、字符串部分列表、长度为 1 的列表和值类型列表
            return size, sparts, [1], types
        # 检查是否为简单标量类型
        nptype = self.gguf_scalar_to_np.get(gtype)
        if nptype is not None:
            # 调用 _get 方法获取值，并返回值的大小、值列表、长度为 0 的列表和值类型列表
            val = self._get(offs, nptype)
            return int(val.nbytes), [val], [0], types
        # 处理数组类型
        if gtype == GGUFValueType.ARRAY:
            # 从原始偏移值获取原始数组类型，并更新偏移值
            raw_itype = self._get(offs, np.uint32)
            offs += int(raw_itype.nbytes)
            # 从偏移值获取数组长度，并更新偏移值
            alen = self._get(offs, np.uint64)
            offs += int(alen.nbytes)
            # 创建包含原始数组类型和数组长度的列表
            aparts: list[npt.NDArray[Any]] = [raw_itype, alen]
            # 创建一个数据索引列表
            data_idxs: list[int] = []
            # 遍历数组长度
            for idx in range(alen[0]):
                # 调用 _get_field_parts 方法获取字段部分的大小、部分列表、索引列表和值类型列表
                curr_size, curr_parts, curr_idxs, curr_types = self._get_field_parts(offs, raw_itype[0])
                # 如果是第一个索引，将当前类型列表添加到总类型列表中
                if idx == 0:
                    types += curr_types
                # 计算索引的偏移值
                idxs_offs = len(aparts)
                # 将当前部分添加到 aparts 列表中
                aparts += curr_parts
                # 将当前索引添加到数据索引列表中
                data_idxs += (idx + idxs_offs for idx in curr_idxs)
                # 更新偏移值
                offs += curr_size
            # 返回总偏移值减去原始偏移值、aparts 列表、数据索引列表和值类型列表
            return offs - orig_offs, aparts, data_idxs, types
        # 无法处理未知类型，抛出 ValueError 异常
        raise ValueError('Unknown/unhandled field type {gtype}')
    # 从指定偏移量获取张量数据，并返回一个 ReaderField 对象
    def _get_tensor(self, orig_offs: int) -> ReaderField:
        # 复制原始偏移量
        offs = orig_offs
        # 获取名称长度和名称数据
        name_len, name_data = self._get_str(offs)
        # 更新偏移量
        offs += int(name_len.nbytes + name_data.nbytes)
        # 获取张量维度
        n_dims = self._get(offs, np.uint32)
        # 更新偏移量
        offs += int(n_dims.nbytes)
        # 获取张量维度数据
        dims = self._get(offs, np.uint64, n_dims[0])
        # 更新偏移量
        offs += int(dims.nbytes)
        # 获取原始数据类型
        raw_dtype = self._get(offs, np.uint32)
        # 更新偏移量
        offs += int(raw_dtype.nbytes)
        # 获取张量偏移量
        offset_tensor = self._get(offs, np.uint64)
        # 更新偏移量
        offs += int(offset_tensor.nbytes)
        # 返回 ReaderField 对象
        return ReaderField(
            orig_offs,
            str(bytes(name_data), encoding='utf-8'),
            [name_len, name_data, n_dims, dims, raw_dtype, offset_tensor],
            [1, 3, 4, 5],
        )

    # 根据偏移量和数量构建字段
    def _build_fields(self, offs: int, count: int) -> int:
        # 循环构建字段
        for _ in range(count):
            # 复制原始偏移量
            orig_offs = offs
            # 获取键的长度和数据
            kv_klen, kv_kdata = self._get_str(offs)
            # 更新偏移量
            offs += int(kv_klen.nbytes + kv_kdata.nbytes)
            # 获取原始键值类型
            raw_kv_type = self._get(offs, np.uint32)
            # 更新偏移量
            offs += int(raw_kv_type.nbytes)
            # 构建部分列表
            parts: list[npt.NDArray[Any]] = [kv_klen, kv_kdata, raw_kv_type]
            # 记录索引偏移量
            idxs_offs = len(parts)
            # 获取字段部分的大小、部分列表、索引列表和类型列表
            field_size, field_parts, field_idxs, field_types = self._get_field_parts(offs, raw_kv_type[0])
            # 添加字段部分到部分列表
            parts += field_parts
            # 将字段推送到字段列表中
            self._push_field(ReaderField(
                orig_offs,
                str(bytes(kv_kdata), encoding='utf-8'),
                parts,
                [idx + idxs_offs for idx in field_idxs],
                field_types,
            ), skip_sum=True)
            # 更新偏移量
            offs += field_size
        # 返回最终偏移量
        return offs

    # 根据偏移量和数量构建张量字段
    def _build_tensors_fields(self, offs: int, count: int) -> tuple[int, list[ReaderField]]:
        # 初始化张量字段列表
        tensor_fields = []
        # 循环构建张量字段
        for _ in range(count):
            # 获取张量字段
            field = self._get_tensor(offs)
            # 更新偏移量
            offs += sum(int(part.nbytes) for part in field.parts)
            # 添加张量字段到列表
            tensor_fields.append(field)
        # 返回最终偏移量和张量字段列表
        return offs, tensor_fields
    # 构建张量数据
    def _build_tensors(self, start_offs: int, fields: list[ReaderField]) -> None:
        # 初始化张量列表
        tensors = []
        # 遍历每个字段
        for field in fields:
            # 解析字段的各个部分
            _name_len, name_data, _n_dims, dims, raw_dtype, offset_tensor = field.parts
            # 获取 GGML 类型
            ggml_type = GGMLQuantizationType(raw_dtype[0])
            # 计算元素个数
            n_elems = np.prod(dims)
            # 获取块大小和类型大小
            block_size, type_size = GGML_QUANT_SIZES[ggml_type]
            # 计算字节数
            n_bytes = n_elems * type_size // block_size
            # 计算数据偏移量
            data_offs = int(start_offs + offset_tensor[0])
            # 初始化项类型
            item_type: npt.DTypeLike
            # 根据 GGML 类型确定项的数量和类型
            if ggml_type == GGMLQuantizationType.F32:
                item_count = n_elems
                item_type = np.float32
            elif ggml_type == GGMLQuantizationType.F16:
                item_count = n_elems
                item_type = np.float16
            else:
                item_count = n_bytes
                item_type = np.uint8
            # 构建张量对象并添加到列表中
            tensors.append(ReaderTensor(
                name = str(bytes(name_data), encoding = 'utf-8'),
                tensor_type = ggml_type,
                shape = dims,
                n_elements = n_elems,
                n_bytes = n_bytes,
                data_offset = data_offs,
                data = self._get(data_offs, item_type, item_count),
                field = field,
            ))
        # 将张量列表赋值给对象属性
        self.tensors = tensors
```