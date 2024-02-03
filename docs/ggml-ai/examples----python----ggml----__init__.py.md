# `ggml\examples\python\ggml\__init__.py`

```cpp
"""
  Python bindings for the ggml library.

  Usage example:

      from ggml import lib, ffi
      from ggml.utils import init, copy, numpy
      import numpy as np

      ctx = init(mem_size=10*1024*1024)
      n = 1024
      n_threads = 4

      a = lib.ggml_new_tensor_1d(ctx, lib.GGML_TYPE_Q5_K, n)
      b = lib.ggml_new_tensor_1d(ctx, lib.GGML_TYPE_F32, n)
      sum = lib.ggml_add(ctx, a, b)

      gf = ffi.new('struct ggml_cgraph*')
      lib.ggml_build_forward_expand(gf, sum)

      copy(np.array([i for i in range(n)], np.float32), a)
      copy(np.array([i*100 for i in range(n)], np.float32), b)
      lib.ggml_graph_compute_with_ctx(ctx, gf, n_threads)

      print(numpy(sum, allow_copy=True))

  See https://cffi.readthedocs.io/en/latest/cdef.html for more on cffi.
"""

try:
    from ggml.cffi import ffi as ffi  # 导入 ggml.cffi 模块中的 ffi 对象
except ImportError as e:
    raise ImportError(f"Couldn't find ggml bindings ({e}). Run `python regenerate.py` or check your PYTHONPATH.")  # 如果导入失败，抛出 ImportError 异常

import os, platform  # 导入 os 和 platform 模块

__exact_library = os.environ.get("GGML_LIBRARY")  # 获取环境变量 GGML_LIBRARY 的值
if __exact_library:  # 如果环境变量 GGML_LIBRARY 存在
    __candidates = [__exact_library]  # 将其作为候选库
elif platform.system() == "Windows":  # 如果系统是 Windows
    __candidates = ["ggml_shared.dll", "llama.dll"]  # 设置 Windows 下的候选库
else:  # 如果系统不是 Windows
    __candidates = ["libggml_shared.so", "libllama.so"]  # 设置 Linux 下的候选库
    if platform.system() == "Darwin":  # 如果系统是 macOS
        __candidates += ["libggml_shared.dylib", "libllama.dylib"]  # 添加 macOS 下的候选库

for i, name in enumerate(__candidates):  # 遍历候选库列表
    try:
        # This is where all the functions, enums and constants are defined
        lib = ffi.dlopen(name)  # 使用 cffi 打开候选库
    except OSError:  # 如果打开失败
        if i < len(__candidates) - 1:  # 如果不是最后一个候选库
            continue  # 继续尝试下一个候选库
        raise OSError(f"Couldn't find ggml's shared library (tried names: {__candidates}). Add its directory to DYLD_LIBRARY_PATH (on Mac) or LD_LIBRARY_PATH, or define GGML_LIBRARY.")  # 如果所有候选库都尝试失败，抛出 OSError 异常

# This contains the cffi helpers such as new, cast, string, etc.
# https://cffi.readthedocs.io/en/latest/ref.html#ffi-interface
ffi = ffi  # 设置 ffi 对象
```