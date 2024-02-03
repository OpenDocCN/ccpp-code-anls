# `PowerInfer\gguf-py\examples\writer.py`

```cpp
#!/usr/bin/env python3
# 指定使用 Python3 解释器

import sys
from pathlib import Path
# 导入 sys 和 Path 模块

import numpy as np
# 导入 numpy 模块，并使用别名 np

# Necessary to load the local gguf package
sys.path.insert(0, str(Path(__file__).parent.parent))
# 将当前文件的父目录的父目录添加到 sys.path 中，以便加载本地的 gguf 包

from gguf import GGUFWriter  # noqa: E402
# 从 gguf 包中导入 GGUFWriter 类

# Example usage:
def writer_example() -> None:
    # 定义一个示例函数 writer_example，无返回值

    # Example usage with a file
    gguf_writer = GGUFWriter("example.gguf", "llama")
    # 创建 GGUFWriter 对象，指定文件名和密码

    gguf_writer.add_architecture()
    # 向 GGUFWriter 对象中添加架构信息
    gguf_writer.add_block_count(12)
    # 向 GGUFWriter 对象中添加块数量信息
    gguf_writer.add_uint32("answer", 42)  # Write a 32-bit integer
    # 向 GGUFWriter 对象中添加 32 位整数信息
    gguf_writer.add_float32("answer_in_float", 42.0)  # Write a 32-bit float
    # 向 GGUFWriter 对象中添加 32 位浮点数信息
    gguf_writer.add_custom_alignment(64)
    # 向 GGUFWriter 对象中添加自定义对齐信息

    tensor1 = np.ones((32,), dtype=np.float32) * 100.0
    tensor2 = np.ones((64,), dtype=np.float32) * 101.0
    tensor3 = np.ones((96,), dtype=np.float32) * 102.0
    # 创建三个 numpy 数组

    gguf_writer.add_tensor("tensor1", tensor1)
    gguf_writer.add_tensor("tensor2", tensor2)
    gguf_writer.add_tensor("tensor3", tensor3)
    # 向 GGUFWriter 对象中添加三个张量信息

    gguf_writer.write_header_to_file()
    # 将头部信息写入文件
    gguf_writer.write_kv_data_to_file()
    # 将键值对数据写入文件
    gguf_writer.write_tensors_to_file()
    # 将张量数据写入文件

    gguf_writer.close()
    # 关闭 GGUFWriter 对象


if __name__ == '__main__':
    writer_example()
    # 如果当前脚本被直接执行，则调用 writer_example 函数
```