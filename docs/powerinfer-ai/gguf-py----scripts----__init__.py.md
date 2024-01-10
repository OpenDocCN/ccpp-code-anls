# `PowerInfer\gguf-py\scripts\__init__.py`

```
# 导入操作系统模块
import os

# 从 importlib 模块中导入 import_module 函数
from importlib import import_module

# 设置环境变量 NO_LOCAL_GGUF 为 TRUE
os.environ["NO_LOCAL_GGUF"] = "TRUE"

# 导入 gguf-convert-endian 脚本的主函数，并赋值给 gguf_convert_endian_entrypoint
gguf_convert_endian_entrypoint = import_module("scripts.gguf-convert-endian").main

# 导入 gguf-dump 脚本的主函数，并赋值给 gguf_dump_entrypoint
gguf_dump_entrypoint = import_module("scripts.gguf-dump").main

# 导入 gguf-set-metadata 脚本的主函数，并赋值给 gguf_set_metadata_entrypoint
gguf_set_metadata_entrypoint = import_module("scripts.gguf-set-metadata").main

# 删除 import_module 和 os，释放内存
del import_module, os
```