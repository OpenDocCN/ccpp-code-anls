# `whisper.cpp\models\ggml_to_pt.py`

```cpp
# 导入所需的库
import struct
import torch
import numpy as np
from collections import OrderedDict
from pathlib import Path
import sys

# 检查命令行参数是否足够
if len(sys.argv) < 3:
    print(
        "Usage: convert-ggml-to-pt.py model.bin dir-output\n")
    sys.exit(1)

# 从命令行参数中获取输入文件名和输出目录
fname_inp = Path(sys.argv[1])
dir_out = Path(sys.argv[2])
fname_out = dir_out / "torch-model.pt"

# 打开 ggml 文件
with open(fname_inp, "rb") as f:
    # 读取魔术数字和超参数
    magic_number, n_vocab, n_audio_ctx, n_audio_state, n_audio_head, n_audio_layer, n_text_ctx, n_text_state, n_text_head, n_text_layer, n_mels, use_f16 = struct.unpack("12i", f.read(48))
    print(f"Magic number: {magic_number}")
    print(f"Vocab size: {n_vocab}")
    print(f"Audio context size: {n_audio_ctx}")
    print(f"Audio state size: {n_audio_state}")
    print(f"Audio head size: {n_audio_head}")
    print(f"Audio layer size: {n_audio_layer}")
    print(f"Text context size: {n_text_ctx}")
    print(f"Text head size: {n_text_head}")
    print(f"Mel size: {n_mels}")
    
    # 读取 mel 滤波器
    filters_shape_0 = struct.unpack("i", f.read(4))[0]
    print(f"Filters shape 0: {filters_shape_0}")
    filters_shape_1 = struct.unpack("i", f.read(4))[0]
    print(f"Filters shape 1: {filters_shape_1}")

    # 读取 tokenizer tokens
    mel_filters = np.zeros((filters_shape_0, filters_shape_1))

    # 逐个读取 mel 滤波器的值
    for i in range(filters_shape_0):
        for j in range(filters_shape_1):
            mel_filters[i][j] = struct.unpack("f", f.read(4))[0]
    
    # 读取 tokens 的数量
    bytes_data = f.read(4) 
    num_tokens = struct.unpack("i", bytes_data)[0]
    tokens = {}
    # 循环读取指定数量的 token
    for _ in range(num_tokens):
        # 读取 token 长度
        token_len = struct.unpack("i", f.read(4))[0]
        # 读取 token 数据
        token = f.read(token_len)
        # 将 token 存入 tokens 字典中
        tokens[token] = {}
    
    # 读取模型变量
    model_state_dict = OrderedDict()
    while True:
        try:
            # 读取变量的维度、名称长度和数据类型
            n_dims, name_length, ftype = struct.unpack("iii", f.read(12))
        except struct.error:
            break  # 文件结束
        # 读取每个维度的大小
        dims = [struct.unpack("i", f.read(4))[0] for _ in range(n_dims)]
        # 将维度反转
        dims = dims[::-1]
        # 读取变量名称并解码为 UTF-8 格式
        name = f.read(name_length).decode("utf-8")
        # 根据数据类型读取数据
        if ftype == 1:  # f16
            data = np.fromfile(f, dtype=np.float16, count=np.prod(dims)).reshape(dims)
        else:  # f32
            data = np.fromfile(f, dtype=np.float32, count=np.prod(dims)).reshape(dims)

        # 如果变量名为 "encoder.conv1.bias" 或 "encoder.conv2.bias"
        if name in  ["encoder.conv1.bias", "encoder.conv2.bias"]:
            # 只保留数据的第一列
            data = data[:, 0]
        
        # 将数据转换为 PyTorch 张量并存入模型状态字典中
        model_state_dict[name] = torch.from_numpy(data)
# 现在你有存储在 model_state_dict 中的模型状态字典
# 你可以将这个 state_dict 加载到具有相同架构的模型中

# 创建模型维度对象 dims，使用 ModelDimensions 类
from whisper import Whisper, ModelDimensions
dims = ModelDimensions(
    n_mels=n_mels,  # 梅尔频谱的数量
    n_audio_ctx=n_audio_ctx,  # 音频上下文的维度
    n_audio_state=n_audio_state,  # 音频状态的维度
    n_audio_head=n_audio_head,  # 音频头的数量
    n_audio_layer=n_audio_layer,  # 音频层的数量
    n_text_ctx=n_text_ctx,  # 文本上下文的维度
    n_text_state=n_text_state,  # 文本状态的维度
    n_text_head=n_text_head,  # 文本头的数量
    n_text_layer=n_text_layer,  # 文本层的数量
    n_vocab=n_vocab,  # 词汇表的大小
)
# 创建 Whisper 模型对象，使用 dims 参数
model = Whisper(dims)  # 用你的模型类替换 Whisper
# 加载模型的状态字典
model.load_state_dict(model_state_dict)

# 将模型以 PyTorch 格式保存
torch.save(model.state_dict(), fname_out)
```