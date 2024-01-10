# `ggml\examples\python\example_test_all_quants.py`

```
# 从 ggml 模块中导入 ffi 和 lib
from ggml import ffi, lib
# 从 ggml.utils 模块中导入 init, numpy, copy
from ggml.utils import init, numpy, copy
# 导入 numpy 库，并重命名为 np
import numpy as np
# 从 math 模块中导入 pi, cos, sin, ceil
from math import pi, cos, sin, ceil
# 导入 matplotlib.pyplot 库，并重命名为 plt
import matplotlib.pyplot as plt

# 初始化上下文，设置内存大小为 100MB，将会自动进行垃圾回收
ctx = init(mem_size=100*1024*1024)
# 设置 n 的值为 256
n = 256

# 创建原始二维数组 orig
orig = np.array([
    [
        cos(j * 2 * pi / n) * (sin(i * 2 * pi / n))
        for j in range(n)
    ]
    for i in range(n)
], np.float32)
# 使用 lib.ggml_new_tensor_2d 函数创建原始张量 orig_tensor，并将 orig 复制到 orig_tensor
orig_tensor = lib.ggml_new_tensor_2d(ctx, lib.GGML_TYPE_F32, n, n)
copy(orig, orig_tensor)

# 创建量化类型列表 quants
quants = [
    type for type in range(lib.GGML_TYPE_COUNT)
    if lib.ggml_is_quantized(type) and
       type not in [lib.GGML_TYPE_Q8_1, lib.GGML_TYPE_Q8_K] # 显然不支持
]
# 将量化类型列表按名称排序
quants.sort(key=get_name)
# 在列表开头插入 None
quants.insert(0, None)
# 打印量化类型列表
print(quants)

# 设置 ncols 和 nrows 的值
ncols=4
nrows = ceil(len(quants) / ncols)

# 创建一个图形窗口，设置大小为 (ncols * 5, nrows * 5)，布局为 'tight'
plt.figure(figsize=(ncols * 5, nrows * 5), layout='tight')

# 遍历量化类型列表 quants
for i, type in enumerate(quants):
    # 在图形窗口中创建子图
    plt.subplot(nrows, ncols, i + 1)
    try:
        # 如果类型为 None
        if type == None:
            # 设置子图标题为 'Original'，并显示原始二维数组 orig
            plt.title('Original')
            plt.imshow(orig)
        else:
            # 使用 lib.ggml_new_tensor_2d 函数创建量化张量 quantized_tensor，并将 orig_tensor 复制到 quantized_tensor
            quantized_tensor = lib.ggml_new_tensor_2d(ctx, type, n, n)
            copy(orig_tensor, quantized_tensor)
            # 将量化张量转换为 numpy 数组 quantized
            quantized = numpy(quantized_tensor, allow_copy=True)
            # 计算差值矩阵 d，并计算 L2 范数、L∞ 范数和压缩比
            d = quantized - orig
            results = {
                "l2": np.linalg.norm(d, 2),
                "linf": np.linalg.norm(d, np.inf),
                "compression":
                    round(lib.ggml_nbytes(orig_tensor) /
                          lib.ggml_nbytes(quantized_tensor), 1)
            }
            # 获取量化类型的名称
            name = get_name(type)
            # 打印量化类型名称和计算结果
            print(f'{name}: {results}')

            # 设置子图标题为量化类型名称和压缩比，显示量化数组 quantized
            plt.title(f'{name} ({results["compression"]}x smaller)')
            plt.imshow(quantized, interpolation='nearest')
        
    except Exception as e:
        # 捕获异常并打印错误信息
        print(f'Error: {e}')

# 显示图形窗口
plt.show()
```