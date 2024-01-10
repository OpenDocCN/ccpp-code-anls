# `ggml\examples\replit\convert-h5-to-ggml.py`

```
# 导入所需的库
from pathlib import Path
import sys
import struct
import json
import numpy as np
from transformers import AutoModelForCausalLM, AutoTokenizer
import sentencepiece.sentencepiece_model_pb2 as model

# 检查命令行参数是否足够
if len(sys.argv) < 3:
    print("Usage: convert-h5-to-ggml.py dir-model [use-f32]\n")
    print("  ftype == 0 -> float32")
    print("  ftype == 1 -> float16")
    sys.exit(1)

# 设置输出文件的路径
dir_model = sys.argv[1]
fname_out = sys.argv[1] + "/ggml-model.bin"

# 从配置文件中加载超参数
with open(dir_model + "/config.json", "r", encoding="utf-8") as f:
    hparams = json.load(f)

# 解析 sentencepiece 模型
sp_proto = model.ModelProto()
sp_proto.ParseFromString(open(Path(sys.argv[1]) / "spiece.model", "rb").read())

# 定义数据类型的字符串表示
ftype_str = ["f32", "f16"]

# 根据命令行参数确定数据类型
ftype = 1
if len(sys.argv) > 2:
    ftype = int(sys.argv[2])
    if ftype < 0 or ftype > 1:
        print("Invalid ftype: " + str(ftype))
        sys.exit(1)
    fname_out = sys.argv[1] + "/ggml-model-" + ftype_str[ftype] + ".bin"

# 加载模型的 tokenizer 和模型本身
tokenizer = AutoTokenizer.from_pretrained(dir_model, trust_remote_code=True)
model = AutoModelForCausalLM.from_pretrained(
    dir_model, low_cpu_mem_usage=True, trust_remote_code=True
)

# 打印模型的变量名、形状和数据类型
list_vars = model.state_dict()
for name in list_vars.keys():
    print(name, list_vars[name].shape, list_vars[name].dtype)

# 打开输出文件
fout = open(fname_out, "wb")

# 写入模型的一些基本信息到输出文件
fout.write(struct.pack("i", 0x67676D6C))  # magic: ggml in hex
fout.write(struct.pack("i", hparams["d_model"]))
fout.write(struct.pack("i", hparams["max_seq_len"]))
fout.write(struct.pack("i", hparams["n_heads"]))
fout.write(struct.pack("i", hparams["n_layers"]))
fout.write(struct.pack("i", hparams["vocab_size"]))
fout.write(struct.pack("i", ftype))

# TODO: 暂时的 hack，不处理实现 tokenizer 的细节
for piece in sp_proto.pieces:
    encoded_piece = piece.piece.encode("utf-8")
    # 使用 struct.pack 将编码后的片段长度以整数形式写入文件
    fout.write(struct.pack("i", len(encoded_piece)))
    # 将编码后的片段数据写入文件
    fout.write(encoded_piece)
    # 使用 struct.pack 将片段得分以浮点数形式写入文件
    fout.write(struct.pack("f", piece.score))
# 如果指定的词汇表大小大于当前的词汇表大小
if hparams["vocab_size"] > len(sp_proto.pieces):
    # 补充空白数据，使词汇表大小达到指定大小
    for i in range(hparams["vocab_size"] - len(sp_proto.pieces)):
        fout.write(struct.pack("i", 0))  # 写入整数0
        fout.write(struct.pack("f", 0))  # 写入浮点数0.0

# 遍历变量列表中的每个变量名
for name in list_vars.keys():
    # 获取变量的数据并转换为numpy数组
    data = list_vars[name].squeeze().numpy()
    print("Processing variable: " + name + " with shape: ", data.shape)

    # 获取数据的维度
    n_dims = len(data.shape)

    # 根据数据类型进行转换，0表示float32，1表示float16
    ftype_cur = 0
    if ftype != 0:
        # 如果数据类型不是float32，且变量名以".weight"结尾且维度为2，则转换为float16
        if name[-7:] == ".weight" and n_dims == 2:
            print("  Converting to float16")
            data = data.astype(np.float16)
            ftype_cur = 1
        else:
            print("  Converting to float32")
            data = data.astype(np.float32)
            ftype_cur = 0
    else:
        # 如果数据类型不是float32，则转换为float32
        if data.dtype != np.float32:
            print("  Converting to float32")
            data = data.astype(np.float32)
            ftype_cur = 0

    # 写入变量名的编码、维度、数据类型到输出文件
    str = name.encode("utf-8")
    fout.write(struct.pack("iii", n_dims, len(str), ftype_cur))  # 写入维度、编码长度、数据类型
    for i in range(n_dims):
        fout.write(struct.pack("i", data.shape[n_dims - 1 - i]))  # 写入每个维度的大小
    fout.write(str)  # 写入编码

    # 写入数据到输出文件
    data.tofile(fout)

# 关闭输出文件
fout.close()

# 打印处理完成的消息和输出文件名
print("Done. Output file: " + fname_out)
print("")
```