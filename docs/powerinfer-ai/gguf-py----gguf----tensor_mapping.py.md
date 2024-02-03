# `PowerInfer\gguf-py\gguf\tensor_mapping.py`

```cpp
# 从 __future__ 模块中导入 annotations 特性
from __future__ import annotations

# 从 typing 模块中导入 Sequence 类型
from typing import Sequence

# 从当前目录下的 constants 模块中导入 MODEL_ARCH, MODEL_TENSOR, MODEL_TENSORS, TENSOR_NAMES 常量
from .constants import MODEL_ARCH, MODEL_TENSOR, MODEL_TENSORS, TENSOR_NAMES


# 定义一个名为 TensorNameMap 的类
class TensorNameMap:
    # 定义一个名为 mapping 的实例变量，类型为字典，键为字符串，值为元组(MODEL_TENSOR, str)
    mapping: dict[str, tuple[MODEL_TENSOR, str]]

    # 定义初始化方法，接受 arch (MODEL_ARCH 类型) 和 n_blocks (整数类型) 两个参数
    def __init__(self, arch: MODEL_ARCH, n_blocks: int):
        # 初始化 mapping 为空字典
        self.mapping = {}
        # 遍历 mappings_cfg 中的每个 tensor 和 keys
        for tensor, keys in self.mappings_cfg.items():
            # 如果 tensor 不在 MODEL_TENSORS[arch] 中，则跳过当前循环
            if tensor not in MODEL_TENSORS[arch]:
                continue
            # 获取 tensor 对应的名称
            tensor_name = TENSOR_NAMES[tensor]
            # 将 tensor_name 和 (tensor, tensor_name) 的元组作为键值对添加到 mapping 中
            self.mapping[tensor_name] = (tensor, tensor_name)
            # 遍历 keys 中的每个 key
            for key in keys:
                # 将 (tensor, tensor_name) 的元组作为值，key 作为键添加到 mapping 中
                self.mapping[key] = (tensor, tensor_name)
        # 遍历 n_blocks 次
        for bid in range(n_blocks):
            # 遍历 block_mappings_cfg 中的每个 tensor 和 keys
            for tensor, keys in self.block_mappings_cfg.items():
                # 如果 tensor 不在 MODEL_TENSORS[arch] 中，则跳过当前循环
                if tensor not in MODEL_TENSORS[arch]:
                    continue
                # 根据当前的 bid 格式化 tensor 对应的名称
                tensor_name = TENSOR_NAMES[tensor].format(bid=bid)
                # 将 tensor_name 和 (tensor, tensor_name) 的元组作为键值对添加到 mapping 中
                self.mapping[tensor_name] = (tensor, tensor_name)
                # 遍历 keys 中的每个 key
                for key in keys:
                    # 根据当前的 bid 格式化 key，并将 (tensor, tensor_name) 的元组作为值，格式化后的 key 作为键添加到 mapping 中
                    key = key.format(bid=bid)
                    self.mapping[key] = (tensor, tensor_name)

    # 定义一个名为 get_type_and_name 的方法，接受 key (字符串类型) 和 try_suffixes (字符串序列类型) 两个参数，返回值为元组(MODEL_TENSOR, str) 或 None
    def get_type_and_name(self, key: str, try_suffixes: Sequence[str] = ()) -> tuple[MODEL_TENSOR, str] | None:
        # 获取 key 对应的值
        result = self.mapping.get(key)
        # 如果值不为 None，则返回该值
        if result is not None:
            return result
        # 遍历 try_suffixes 中的每个后缀
        for suffix in try_suffixes:
            # 如果 key 以当前后缀结尾
            if key.endswith(suffix):
                # 获取去除后缀后的 key 对应的值
                result = self.mapping.get(key[:-len(suffix)])
                # 如果值不为 None，则返回该值的第一个和第二个元素加上后缀的组合
                if result is not None:
                    return result[0], result[1] + suffix
        # 返回 None
        return None

    # 定义一个名为 get_name 的方法，接受 key (字符串类型) 和 try_suffixes (字符串序列类型) 两个参数，返回值为字符串或 None
    def get_name(self, key: str, try_suffixes: Sequence[str] = ()) -> str | None:
        # 调用 get_type_and_name 方法获取结果
        result = self.get_type_and_name(key, try_suffixes=try_suffixes)
        # 如果结果为 None，则返回 None
        if result is None:
            return None
        # 返回结果的第二个元素
        return result[1]
    # 获取指定键对应的类型，如果指定后缀则尝试使用后缀
    def get_type(self, key: str, try_suffixes: Sequence[str] = ()) -> MODEL_TENSOR | None:
        # 调用 get_type_and_name 方法获取类型和名称
        result = self.get_type_and_name(key, try_suffixes = try_suffixes)
        # 如果结果为空，则返回空
        if result is None:
            return None
        # 返回结果的类型
        return result[0]
    
    # 获取指定键对应的名称
    def __getitem__(self, key: str) -> str:
        try:
            # 返回键对应的名称
            return self.mapping[key][1]
        except KeyError:
            # 如果键不存在，则抛出 KeyError 异常
            raise KeyError(key)
    
    # 检查指定键是否存在于映射中
    def __contains__(self, key: str) -> bool:
        return key in self.mapping
    
    # 返回映射的字符串表示形式
    def __repr__(self) -> str:
        return repr(self.mapping)
# 定义一个函数，用于获取模型架构和块数的张量名称映射
def get_tensor_name_map(arch: MODEL_ARCH, n_blocks: int) -> TensorNameMap:
    # 返回一个根据模型架构和块数创建的张量名称映射对象
    return TensorNameMap(arch, n_blocks)
```