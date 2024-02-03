# `ggml\examples\gpt-2\convert-ckpt-to-ggml.py`

```cpp
# Convert a model checkpoint to a ggml compatible file
# 将模型检查点转换为 ggml 兼容文件

# Load the model using TensorFlow.
# 使用 TensorFlow 加载模型
import sys
import json
import struct
import numpy as np
import tensorflow as tf

# ref: https://github.com/openai/gpt-2/blob/master/src/encoder.py
# 引用：https://github.com/openai/gpt-2/blob/master/src/encoder.py
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
    # 返回 utf-8 字节列表和相应的 Unicode 字符串列表
    # 可逆的 bpe 代码适用于 Unicode 字符串
    # 这意味着如果要避免 UNKs，则需要在词汇表中拥有大量的 Unicode 字符
    # 当你处理类似 10B 令牌数据集时，最终需要大约 5K 个字符以获得良好的覆盖率
    # 这在你正常的 32K bpe 词汇表中占据了相当大的比例
    # 为了避免这种情况，我们需要在 utf-8 字节和 Unicode 字符串之间建立查找表
    # 并避免映射到 bpe 代码无法处理的空格/控制字符

# helper method to convert a numpy array to different float types
# 辅助方法，将 numpy 数组转换为不同的浮点类型
def convert_to_ftype(data, ftype):
    # fp16
    if ftype == 1:
        return data.astype(np.float16)
    # 如果 ftype 为 1，则返回转换为 np.float16 类型的数据

    assert False, "Invalid ftype: " + str(ftype)
    # 断言，如果 ftype 无效，则抛出错误信息："Invalid ftype: " + str(ftype)

if len(sys.argv) < 3:
    print("Usage: convert-ckpt-to-ggml.py dir-model ftype\n")
    print("  ftype == 0 -> float32")
    # 如果命令行参数少于 3 个，则打印使用说明和 ftype 的说明
    # 打印消息，表示 ftype 等于 1 时代表 float16 类型
    print("  ftype == 1 -> float16")
    # 退出程序，状态码为1
    sys.exit(1)
# 将模型输出到与模型相同的目录
dir_model = sys.argv[1]
fname_out = sys.argv[1] + "/ggml-model.bin"

# 以 UTF-8 编码打开 encoder.json 文件，并加载其内容
with open(dir_model + "/encoder.json", "r", encoding="utf-8") as f:
    encoder = json.load(f)

# 以 UTF-8 编码打开 hparams.json 文件，并加载其内容
with open(dir_model + "/hparams.json", "r", encoding="utf-8") as f:
    hparams = json.load(f)

# 可能的数据类型
#   ftype == 0 -> float32
#   ftype == 1 -> float16
#
# 将 ftype 映射为字符串
ftype_str = ["f32", "f16"]

# 默认数据类型为 float16
ftype = 1
# 如果命令行参数大于 2 个，则将 ftype 设置为第三个参数的整数值
if len(sys.argv) > 2:
    ftype = int(sys.argv[2])
    # 如果 ftype 不在 0 和 1 之间，则打印错误信息并退出程序
    if ftype < 0 or ftype > 1:
        print("Invalid ftype: " + str(ftype))
        sys.exit(1)
    # 更新输出文件名
    fname_out = sys.argv[1] + "/ggml-model-" + ftype_str[ftype] + ".bin"

# 获取模型变量列表
list_vars = tf.train.list_variables(dir_model)

# 以二进制写模式打开输出文件
fout = open(fname_out, "wb")

# 写入魔数和模型参数到输出文件
fout.write(struct.pack("i", 0x67676d6c)) # magic: ggml in hex
fout.write(struct.pack("i", hparams["n_vocab"]))
fout.write(struct.pack("i", hparams["n_ctx"]))
fout.write(struct.pack("i", hparams["n_embd"]))
fout.write(struct.pack("i", hparams["n_head"]))
fout.write(struct.pack("i", hparams["n_layer"]))
fout.write(struct.pack("i", ftype))

# 将字节编码转换为 Unicode
byte_encoder = bytes_to_unicode()
byte_decoder = {v:k for k, v in byte_encoder.items()}

# 写入编码器的长度和内容到输出文件
fout.write(struct.pack("i", len(encoder)))

for key in encoder:
    text = bytearray([byte_decoder[c] for c in key])
    fout.write(struct.pack("i", len(text)))
    fout.write(text)

# 遍历模型变量列表
for name, shape in list_vars:
    print("Processing variable: " + name + " with shape: ", shape)

    # 加载变量数据并压缩维度
    data = tf.train.load_variable(dir_model, name).squeeze()
    n_dims = len(data.shape);

    # 对于效率 - 转置投影矩阵
    # "model/h.*/attn/c_attn/w"
    # "model/h.*/attn/c_proj/w"
    # "model/h.*/mlp/c_fc/w"
    # "model/h.*/mlp/c_proj/w"
    if name[-14:] == "/attn/c_attn/w" or \
       name[-14:] == "/attn/c_proj/w" or \
       name[-11:] == "/mlp/c_fc/w" or \
       name[-13:] == "/mlp/c_proj/w":
        print("  Transposing")
        data = data.transpose()

    dshape = data.shape

    ftype_cur = 0
    # 如果文件类型不为0，则进行以下操作
    if ftype != 0:
        # 匹配文件名：
        #  "model/wte"
        #  "model/h.*/attn/c_attn/w"
        #  "model/h.*/attn/c_proj/w"
        #  "model/h.*/mlp/c_fc/w"
        #  "model/h.*/mlp/c_proj/w"
        if name == "model/wte" or name[-2:] == "/w":
            # 打印转换为指定文件类型的信息
            print("  Converting to " + ftype_str[ftype])
            # 转换数据为指定文件类型
            data = convert_to_ftype(data, ftype)
            # 更新当前文件类型
            ftype_cur = ftype
        else:
            # 打印转换为float32的信息
            print("  Converting to float32")
            # 将数据转换为float32类型
            data = data.astype(np.float32)
            # 更新当前文件类型为0
            ftype_cur = 0

    # 头部信息
    # 将文件名转换为UTF-8编码
    str = name.encode('utf-8')
    # 将头部信息写入输出文件
    fout.write(struct.pack("iii", n_dims, len(str), ftype_cur))
    # 写入维度信息
    for i in range(n_dims):
        fout.write(struct.pack("i", dshape[n_dims - 1 - i]))
    # 写入文件名
    fout.write(str);

    # 数据
    # 将数据以二进制形式写入输出文件
    data.tofile(fout)
# 关闭输出文件
fout.close()

# 打印完成消息和输出文件名
print("Done. Output file: " + fname_out)
print("")
```