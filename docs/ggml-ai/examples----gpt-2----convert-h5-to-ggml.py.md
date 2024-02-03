# `ggml\examples\gpt-2\convert-h5-to-ggml.py`

```cpp
# Convert GPT-2 h5 transformer model to ggml format
#
# Load the model using GPT2Model.
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
import numpy as np
import re

from transformers import GPT2Model

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

if len(sys.argv) < 2:
    print("Usage: convert-h5-to-ggml.py dir-model [use-f32]\n")
    sys.exit(1)

# output in the same directory as the model
dir_model = sys.argv[1]
fname_out = sys.argv[1] + "/ggml-model.bin"

with open(dir_model + "/vocab.json", "r", encoding="utf-8") as f:
    encoder = json.load(f)
# 以只读方式打开文件，使用utf-8编码读取文件内容，加载为json对象
with open(dir_model + "/added_tokens.json", "r", encoding="utf-8") as f:
    encoder_added = json.load(f)

# 以只读方式打开文件，使用utf-8编码读取文件内容，加载为json对象
with open(dir_model + "/config.json", "r", encoding="utf-8") as f:
    hparams = json.load(f)

# 使用16位或32位浮点数
use_f16 = True
# 如果命令行参数数量大于2，则使用32位浮点数，设置输出文件名为第一个命令行参数加上后缀"/ggml-model-f32.bin"
if len(sys.argv) > 2:
    use_f16 = False
    fname_out = sys.argv[1] + "/ggml-model-f32.bin"

# 从预训练模型目录加载GPT2模型，设置低CPU内存使用
model = GPT2Model.from_pretrained(dir_model, low_cpu_mem_usage=True)

# 获取模型的状态字典
list_vars = model.state_dict()

# 以二进制写入模式打开文件
fout = open(fname_out, "wb")

# 写入特定格式的数据到文件中
fout.write(struct.pack("i", 0x67676d6c)) # magic: ggml in hex
fout.write(struct.pack("i", hparams["vocab_size"]))
fout.write(struct.pack("i", hparams["n_positions"]))
fout.write(struct.pack("i", hparams["n_embd"]))
fout.write(struct.pack("i", hparams["n_head"]))
fout.write(struct.pack("i", hparams["n_layer"]))
#fout.write(struct.pack("i", hparams["rotary_dim"]))
fout.write(struct.pack("i", use_f16))

# 将字节编码转换为Unicode编码
byte_encoder = bytes_to_unicode()
byte_decoder = {v:k for k, v in byte_encoder.items()}

# 写入特定格式的数据到文件中
fout.write(struct.pack("i", len(encoder) + len(encoder_added)))

# 遍历encoder中的键值对
for key in encoder:
    # 将键转换为字节编码，写入文件
    text = bytearray([byte_decoder[c] for c in key])
    fout.write(struct.pack("i", len(text)))
    fout.write(text)

# 遍历encoder_added中的键值对
for key in encoder_added:
    # 将键转换为字节编码，写入文件
    text = bytearray([byte_decoder[c] for c in key])
    fout.write(struct.pack("i", len(text)))
    fout.write(text)

# 遍历模型状态字典中的变量
for name in list_vars.keys():
    # 获取变量的数据并转换为numpy数组
    data = list_vars[name].squeeze().numpy()
    print("Processing variable: " + name + " with shape: ", data.shape)

    # 如果变量名以"attn.masked_bias"或".attn.bias"结尾，则跳过
    if name.endswith("attn.masked_bias") or name.endswith(".attn.bias"):
        print("  Skipping variable: " + name)
        continue

    # 获取数据的维度数量
    n_dims = len(data.shape);

    # ftype == 0 -> float32, ftype == 1 -> float16
    ftype = 0;
    # 如果使用 float16 数据类型
    if use_f16:
        # 如果文件名以 ".weight" 结尾且维度为 2
        if name[-7:] == ".weight" and n_dims == 2:
            # 打印信息
            print("  Converting to float16")
            # 将数据类型转换为 float16
            data = data.astype(np.float16)
            # 设置数据类型标志为 1
            ftype = 1
        else:
            # 打印信息
            print("  Converting to float32")
            # 将数据类型转换为 float32
            data = data.astype(np.float32)
            # 设置数据类型标志为 0
            ftype = 0

    # 为了提高效率 - 转置这些矩阵：
    #  "transformer.h.*.mlp.c_proj.weight
    if name.endswith(".mlp.c_proj.weight"):
        # 打印信息
        print("  Transposing")
        # 转置数据
        data = data.transpose()

    # 重命名头部以保持兼容性
    if name == "ln_f.weight":
        name = "model/ln_f/g"
    elif name == "ln_f.bias":
        name = "model/ln_f/b"
    elif name == "wte.weight":
        name = "model/wte"
    elif name == "wpe.weight":
        name = "model/wpe"
    elif re.match(r"h\.\d+\.ln_1\.weight", name):
        i = re.findall("\d+", name)[0]
        name = f"model/h{i}/ln_1/g"
    # ...（其他重命名规则）
    # 如果变量名匹配"h.\d+.mlp.c_fc.bias"的正则表达式，则执行以下操作
    elif re.match(r"h.\d+.mlp.c_fc.bias", name):
        # 从变量名中提取数字
        i = re.findall("\d+", name)[0]
        # 根据提取的数字构建新的变量名
        name = f"model/h{i}/mlp/c_fc/b"
    # 如果变量名匹配"h.\d+.mlp.c_proj.weight"的正则表达式，则执行以下操作
    elif re.match(r"h.\d+.mlp.c_proj.weight", name):
        # 从变量名中提取数字
        i = re.findall("\d+", name)[0]
        # 根据提取的数字构建新的变量名
        name = f"model/h{i}/mlp/c_proj/w"
    # 如果变量名匹配"h.\d+.mlp.c_proj.bias"的正则表达式，则执行以下操作
    elif re.match(r"h.\d+.mlp.c_proj.bias", name):
        # 从变量名中提取数字
        i = re.findall("\d+", name)[0]
        # 根据提取的数字构建新的变量名
        name = f"model/h{i}/mlp/c_proj/b"
    # 如果变量名不匹配以上任何正则表达式，则打印未识别的变量名
    else:
        print("Unrecognized variable name. %s", name)

    # 将变量名转换为 UTF-8 编码的字节流
    str = name.encode('utf-8')

    # 将数据的维度、字节流的长度和文件类型写入输出文件
    fout.write(struct.pack("iii", n_dims, len(str), ftype))
    # 遍历数据的维度，将每个维度的大小写入输出文件
    for i in range(n_dims):
        fout.write(struct.pack("i", data.shape[n_dims - 1 - i]))
    # 将变量名的字节流写入输出文件
    fout.write(str);

    # 将数据以二进制形式写入输出文件
    data.tofile(fout)
# 关闭输出文件
fout.close()

# 打印完成消息和输出文件名
print("Done. Output file: " + fname_out)
print("")
```