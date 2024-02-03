# `ggml\examples\gpt-j\convert-h5-to-ggml.py`

```cpp
# Convert GPT-J-6B h5 transformer model to ggml format
#
# Load the model using GPTJForCausalLM.
# Iterate over all variables and write them to a binary file.
#
# For each variable, write the following:
#   - Number of dimensions (int)
#   - Name length (int)
#   - Dimensions (int[n_dims])
#   - Name (char[name_length])
#   - Data (float[n_dims])
#
# By default, the bigger matrices are converted to 16-bit floats.
# This can be disabled by adding the "use-f32" CLI argument.
#
# At the start of the ggml file we write the model parameters
# and vocabulary.
#

import sys
import struct
import json
import torch
import numpy as np

from transformers import GPTJForCausalLM

# ref: https://github.com/openai/gpt-2/blob/master/src/encoder.py
def bytes_to_unicode():
    """
    Returns list of utf-8 byte and a corresponding list of unicode strings.
    The reversible bpe codes work on unicode strings.
    This means you need a large # of unicode characters in your vocab if you want to avoid UNKs.
    When you're at something like a 10B token dataset you end up needing around 5K for decent coverage.
    This is a signficant percentage of your normal, say, 32K bpe vocab.
    To avoid that, we want lookup tables between utf-8 bytes and unicode strings.
    And avoids mapping to whitespace/control characters the bpe code barfs on.
    """
    bs = list(range(ord("!"), ord("~")+1))+list(range(ord("¡"), ord("¬")+1))+list(range(ord("®"), ord("ÿ")+1))
    cs = bs[:]
    n = 0
    for b in range(2**8):
        if b not in bs:
            bs.append(b)
            cs.append(2**8+n)
            n += 1
    cs = [chr(n) for n in cs]
    return dict(zip(bs, cs))

if len(sys.argv) < 3:
    print("Usage: convert-h5-to-ggml.py dir-model [use-f32]\n")
    print("  ftype == 0 -> float32")
    print("  ftype == 1 -> float16")
    sys.exit(1)

# output in the same directory as the model
dir_model = sys.argv[1]
fname_out = sys.argv[1] + "/ggml-model.bin"
# 以只读方式打开文件，使用utf-8编码读取vocab.json文件内容，加载到encoder变量中
with open(dir_model + "/vocab.json", "r", encoding="utf-8") as f:
    encoder = json.load(f)

# 以只读方式打开文件，使用utf-8编码读取added_tokens.json文件内容，加载到encoder_added变量中
with open(dir_model + "/added_tokens.json", "r", encoding="utf-8") as f:
    encoder_added = json.load(f)

# 以只读方式打开文件，使用utf-8编码读取config.json文件内容，加载到hparams变量中
with open(dir_model + "/config.json", "r", encoding="utf-8") as f:
    hparams = json.load(f)

# 定义可能的数据类型，以及对应的字符串表示
# ftype == 0 -> float32
# ftype == 1 -> float16
ftype_str = ["f32", "f16"]

# 初始化ftype为1
ftype = 1
# 如果命令行参数个数大于2
if len(sys.argv) > 2:
    # 将第三个命令行参数转换为整数，赋值给ftype
    ftype = int(sys.argv[2])
    # 如果ftype小于0或者大于1
    if ftype < 0 or ftype > 1:
        # 打印错误信息并退出程序
        print("Invalid ftype: " + str(ftype))
        sys.exit(1)
    # 拼接输出文件名
    fname_out = sys.argv[1] + "/ggml-model-" + ftype_str[ftype] + ".bin"

# 从预训练模型目录加载GPTJForCausalLM模型，设置低CPU内存使用
model = GPTJForCausalLM.from_pretrained(dir_model, low_cpu_mem_usage=True)

# 获取模型的状态字典
list_vars = model.state_dict()

# 以二进制写入模式打开输出文件
fout = open(fname_out, "wb")

# 写入魔数：ggml的十六进制表示
fout.write(struct.pack("i", 0x67676d6c))
# 写入词汇表大小
fout.write(struct.pack("i", hparams["vocab_size"]))
# 写入位置编码的长度
fout.write(struct.pack("i", hparams["n_positions"]))
# 写入嵌入层的维度
fout.write(struct.pack("i", hparams["n_embd"]))
# 写入注意力头的数量
fout.write(struct.pack("i", hparams["n_head"]))
# 写入层数
fout.write(struct.pack("i", hparams["n_layer"]))
# 写入旋转维度
fout.write(struct.pack("i", hparams["rotary_dim"]))
# 写入数据类型
fout.write(struct.pack("i", ftype))

# 将编码器和新增编码器的长度写入输出文件
fout.write(struct.pack("i", len(encoder) + len(encoder_added)))

# 遍历编码器中的键
for key in encoder:
    # 将键转换为字节流
    text = bytearray([byte_decoder[c] for c in key])
    # 写入字节流的长度
    fout.write(struct.pack("i", len(text)))
    # 写入字节流
    fout.write(text)

# 遍历新增编码器中的键
for key in encoder_added:
    # 将键转换为字节流
    text = bytearray([byte_decoder[c] for c in key])
    # 写入字节流的长度
    fout.write(struct.pack("i", len(text)))
    # 写入字节流
    fout.write(text)

# 遍历模型状态字典中的变量名
for name in list_vars.keys():
    # 获取变量的数据，转换为numpy数组
    data = list_vars[name].squeeze().numpy()
    # 打印处理的变量名和其形状
    print("Processing variable: " + name + " with shape: ", data.shape)

    # 如果变量名以"attn.masked_bias"或".attn.bias"结尾
    if name.endswith("attn.masked_bias") or name.endswith(".attn.bias"):
        # 打印跳过的变量名
        print("  Skipping variable: " + name)
        # 继续下一次循环
        continue
    # 获取数据的维度
    n_dims = len(data.shape);

    # 根据 ftype 的值确定数据类型，0 代表 float32，1 代表 float16
    ftype_cur = 0;
    if ftype != 0:
        # 如果文件名以 ".weight" 结尾且数据维度为 2，则将数据类型转换为 float16
        if name[-7:] == ".weight" and n_dims == 2:
            print("  Converting to float16")
            data = data.astype(np.float16)
            ftype_cur = 1
        else:
            # 否则将数据类型转换为 float32
            print("  Converting to float32")
            data = data.astype(np.float32)
            ftype_cur = 0
    else:
        # 如果 ftype 为 0 且数据类型不是 float32，则将数据类型转换为 float32
        if data.dtype != np.float32:
            print("  Converting to float32")
            data = data.astype(np.float32)
            ftype_cur = 0

    # 写入文件头部信息
    str = name.encode('utf-8')
    fout.write(struct.pack("iii", n_dims, len(str), ftype_cur))
    for i in range(n_dims):
        fout.write(struct.pack("i", data.shape[n_dims - 1 - i]))
    fout.write(str);

    # 写入数据
    data.tofile(fout)
# 关闭输出文件
fout.close()

# 打印完成消息和输出文件名
print("Done. Output file: " + fname_out)
print("")
```