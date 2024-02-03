# `ggml\examples\python\example_add_quant.py`

```cpp
# 从 ggml 库中导入 lib 和 ffi 模块
from ggml import lib, ffi
# 从 ggml.utils 模块中导入 init, copy, numpy 函数，并导入 numpy 库的别名 np
from ggml.utils import init, copy, numpy
import numpy as np

# 初始化上下文，设置内存大小为 12MB，当指针被垃圾回收时会自动释放
ctx = init(mem_size=12*1024*1024)
# 设置变量 n 的值为 256
n = 256
# 设置变量 n_threads 的值为 4

# 创建一个一维张量 a，类型为 GGML_TYPE_Q5_K，长度为 n
a = lib.ggml_new_tensor_1d(ctx, lib.GGML_TYPE_Q5_K, n)
# 创建一个一维张量 b，类型为 GGML_TYPE_F32，长度为 n，注意：a 和 b 不能同时被量化
b = lib.ggml_new_tensor_1d(ctx, lib.GGML_TYPE_F32, n)
# 创建一个张量 sum，通过将张量 a 和 b 相加得到，目前全部为零，之后也会被量化

# 参考 cffi 文档如何分配本地内存：非常简单！
# https://cffi.readthedocs.io/en/latest/ref.html#ffi-interface
# 创建一个指向 ggml_cgraph 结构体的指针 gf
gf = ffi.new('struct ggml_cgraph*')
# 使用 gf 构建前向传播图，将 sum 作为参数传入
lib.ggml_build_forward_expand(gf, sum)

# 将一个长度为 n 的浮点数数组（从 0 到 n-1）复制到张量 a 中
copy(np.array([i for i in range(n)], np.float32), a)
# 将一个长度为 n 的浮点数数组（从 0 到 n*100-1）复制到张量 b 中
copy(np.array([i*100 for i in range(n)], np.float32), b)

# 使用上下文 ctx、前向传播图 gf 和线程数 n_threads 进行图的计算
lib.ggml_graph_compute_with_ctx(ctx, gf, n_threads)

# 打印张量 a 的值，允许复制
print(numpy(a, allow_copy=True))
# 打印张量 b 的值
print(numpy(b))
# 打印张量 sum 的值，允许复制
print(numpy(sum, allow_copy=True))
```