# `ggml\examples\dolly-v2\convert-h5-to-ggml.py`

```cpp
# 导入必要的库
import sys
import struct
import json
import numpy as np

# 从transformers库中导入AutoModelForCausalLM和AutoTokenizer
from transformers import AutoModelForCausalLM, AutoTokenizer

# 检查命令行参数数量是否足够
if len(sys.argv) < 3:
    print("Usage: convert-h5-to-ggml.py dir-model [use-f32]\n")
    print("  ftype == 0 -> float32")
    print("  ftype == 1 -> float16")
    sys.exit(1)

# 设置模型目录和输出文件名
dir_model = sys.argv[1]
fname_out = sys.argv[1] + "/ggml-model.bin"

# 从文件中读取tokenizer的配置
with open(dir_model + "/tokenizer.json", "r", encoding="utf-8") as f:
    encoder = json.load(f)

# 从文件中读取模型的配置
with open(dir_model + "/config.json", "r", encoding="utf-8") as f:
    hparams = json.load(f)

# 定义数据类型的字符串表示
ftype_str = ["f32", "f16"]

# 设置默认数据类型为float16
ftype = 1
# 如果命令行参数中有第三个参数，则使用该参数作为数据类型
if len(sys.argv) > 2:
    ftype = int(sys.argv[2])
    # 检查数据类型是否合法
    if ftype < 0 or ftype > 1:
        print("Invalid ftype: " + str(ftype))
        sys.exit(1)
    # 根据数据类型设置输出文件名
    fname_out = sys.argv[1] + "/ggml-model-" + ftype_str[ftype] + ".bin"

# 从模型目录中加载tokenizer
tokenizer = AutoTokenizer.from_pretrained(dir_model)
# 从模型目录中加载模型
model = AutoModelForCausalLM.from_pretrained(dir_model, low_cpu_mem_usage=True)

# 打印模型的变量名、形状和数据类型
list_vars = model.state_dict()
for name in list_vars.keys():
    print(name, list_vars[name].shape, list_vars[name].dtype)

# 打开输出文件
fout = open(fname_out, "wb")

# 写入模型配置信息到输出文件
fout.write(struct.pack("i", 0x67676d6c)) # magic: ggml in hex
fout.write(struct.pack("i", hparams["vocab_size"]))
fout.write(struct.pack("i", hparams["max_position_embeddings"]))
fout.write(struct.pack("i", hparams["hidden_size"]))
fout.write(struct.pack("i", hparams["num_attention_heads"]))
fout.write(struct.pack("i", hparams["num_hidden_layers"]))
fout.write(struct.pack("i", int(hparams["rotary_pct"]*(hparams["hidden_size"]//hparams["num_attention_heads"])))
fout.write(struct.pack("i", hparams["use_parallel_residual"]))
fout.write(struct.pack("i", ftype))

# TODO: temporary hack to not deal with implementing the tokenizer
# 获取句点的编码值
dot_token = tokenizer.encode('.')[0]
# 遍历词汇表大小的范围
for i in range(hparams["vocab_size"]):
    # 将句点和当前索引编码为 UTF-8 格式的文本
    text = tokenizer.decode([dot_token, i]).encode('utf-8')
    # 移除第一个字节（通常是句点）
    text = text[1:]
    # 将文本长度打包为 4 字节整数并写入输出文件
    fout.write(struct.pack("i", len(text)))
    # 将文本数据写入输出文件
    fout.write(text)

# 遍历变量列表中的每个变量名
for name in list_vars.keys():
    # 获取变量数据并转换为 NumPy 数组
    data = list_vars[name].squeeze().numpy()
    # 打印正在处理的变量名和其形状
    print("Processing variable: " + name + " with shape: ", data.shape)

    # 如果变量名以指定后缀结尾，则跳过处理
    if name.endswith(".attention.masked_bias") or     \
       name.endswith(".attention.bias") or \
       name.endswith(".attention.rotary_emb.inv_freq"):
        print("  Skipping variable: " + name)
        continue

    # 获取数据的维度数
    n_dims = len(data.shape);

    # 根据 ftype 的值确定当前数据类型（0 表示 float32，1 表示 float16）
    ftype_cur = 0;
    if ftype != 0:
        if name[-7:] == ".weight" and n_dims == 2:
            print("  Converting to float16")
            # 将数据类型转换为 float16
            data = data.astype(np.float16)
            ftype_cur = 1
        else:
            print("  Converting to float32")
            # 将数据类型转换为 float32
            data = data.astype(np.float32)
            ftype_cur = 0
    else:
        if data.dtype != np.float32:
            print("  Converting to float32")
            # 将数据类型转换为 float32
            data = data.astype(np.float32)
            ftype_cur = 0

    # 将变量名编码为 UTF-8 格式的字节流
    str = name.encode('utf-8')
    # 写入变量的维度数、变量名长度和数据类型到输出文件
    fout.write(struct.pack("iii", n_dims, len(str), ftype_cur))
    # 写入每个维度的大小到输出文件
    for i in range(n_dims):
        fout.write(struct.pack("i", data.shape[n_dims - 1 - i]))
    # 写入变量名到输出文件
    fout.write(str);

    # 将变量数据写入输出文件
    data.tofile(fout)

# 关闭输出文件
fout.close()

# 打印处理完成的消息和输出文件名
print("Done. Output file: " + fname_out)
print("")
```