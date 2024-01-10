# `ggml\examples\mnist\convert-h5-to-ggml.py`

```
# 将 MNIS h5 转换器模型转换为 ggml 格式
#
# 使用 PyTorch 加载保存的模型（state_dict）
# 遍历所有变量并将它们写入二进制文件
#
# 对于每个变量，写入以下内容：
#   - 维度数量（int）
#   - 名称长度（int）
#   - 维度（int[n_dims]）
#   - 名称（char[name_length]）
#   - 数据（float[n_dims]）
#
# 在 ggml 文件的开头，我们写入模型参数

import sys
import struct
import json
import numpy as np
import re

import torch
import torch.nn as nn
import torchvision.datasets as dsets
import torchvision.transforms as transforms
from torch.autograd import Variable

# 检查命令行参数是否为2个，如果不是则打印用法并退出
if len(sys.argv) != 2:
    print("Usage: convert-h5-to-ggml.py model\n")
    sys.exit(1)

# 从命令行参数中获取模型文件名
state_dict_file = sys.argv[1]
# 输出文件名
fname_out = "models/mnist/ggml-model-f32.bin"

# 使用 PyTorch 加载模型参数
state_dict = torch.load(state_dict_file, map_location=torch.device('cpu'))

# 打开输出文件，以二进制写入模式
fout = open(fname_out, "wb")

# 写入 ggml 文件的魔术数字
fout.write(struct.pack("i", 0x67676d6c)) # magic: ggml in hex

# 遍历模型参数的键
for name in state_dict.keys():
    # 将数据转换为 numpy 数组，并去除维度为1的维度
    data = state_dict[name].squeeze().numpy()
    print("Processing variable: " + name + " with shape: ", data.shape) 
    n_dims = len(data.shape);
   
    # 写入维度数量
    fout.write(struct.pack("i", n_dims))
    
    # 将数据类型转换为 np.float32，并写入每个维度的大小
    data = data.astype(np.float32)
    for i in range(n_dims):
        fout.write(struct.pack("i", data.shape[n_dims - 1 - i]))

    # 写入数据
    data.tofile(fout)

# 关闭输出文件
fout.close()

# 打印完成消息和输出文件名
print("Done. Output file: " + fname_out)
print("")
```