# `ggml\examples\sam\convert-pth-to-ggml.py`

```cpp
# Convert a SAM model checkpoint to a ggml compatible file

# 导入所需的库
import sys
import torch
import struct
import numpy as np

# 检查命令行参数是否足够
if len(sys.argv) < 3:
    print("Usage: convert-pth-to-ggml.py file-model dir-output [ftype]\n")
    print("  ftype == 0 -> float32")
    print("  ftype == 1 -> float16")
    sys.exit(1)

# 获取输入的文件名和输出目录
fname_model = sys.argv[1]
dir_out     = sys.argv[2]
fname_out   = dir_out + "/ggml-model.bin"

# 定义数据类型的字符串表示
ftype_str = ["f32", "f16"]

# 解析命令行参数中的数据类型
ftype = 1
if len(sys.argv) > 3:
    ftype = int(sys.argv[3])

# 检查数据类型是否有效
if ftype < 0 or ftype > 1:
    print("Invalid ftype: " + str(ftype))
    sys.exit(1)

# 根据数据类型修改输出文件名
fname_out = fname_out.replace(".bin", "-" + ftype_str[ftype] + ".bin")

# 默认参数设置为 sam_vit_b checkpoint
n_enc_state = 768
n_enc_layers = 12
n_enc_heads = 12
n_enc_out_chans = 256
n_pt_embd = 4

# 从模型文件中加载模型
model = torch.load(fname_model, map_location="cpu")

# 根据模型参数调整默认参数
for k, v in model.items():
    print(k, v.shape)
    if k == "image_encoder.blocks.0.norm1.weight":
        n_enc_state = v.shape[0]

# 根据 n_enc_state 的值调整其他参数
if n_enc_state == 1024: # sam_vit_l
    n_enc_layers = 24
    n_enc_heads  = 16
elif n_enc_state == 1280: # sam_vit_h
    n_enc_layers = 32
    n_enc_heads  = 16

# 构建模型参数字典
hparams = {
    "n_enc_state":      n_enc_state,
    "n_enc_layers":     n_enc_layers,
    "n_enc_heads":      n_enc_heads,
    "n_enc_out_chans":  n_enc_out_chans,
    "n_pt_embd":        n_pt_embd,
}

# 打印模型参数
print(hparams)

# 打印模型中的所有参数和它们的形状
for k, v in model.items():
    print(k, v.shape)

# 打开输出文件
fout = open(fname_out, "wb")

# 写入文件头部信息
fout.write(struct.pack("i", 0x67676d6c)) # magic: ggml in hex
fout.write(struct.pack("i", hparams["n_enc_state"]))
fout.write(struct.pack("i", hparams["n_enc_layers"]))
fout.write(struct.pack("i", hparams["n_enc_heads"]))
fout.write(struct.pack("i", hparams["n_enc_out_chans"]))
fout.write(struct.pack("i", hparams["n_pt_embd"]))
fout.write(struct.pack("i", ftype))
# 遍历模型中的每个键值对
for k, v in model.items():
    # 获取键值对中的键和值
    name = k
    shape = v.shape

    # 如果键的前19个字符等于"prompt_encoder.mask"，则跳过当前循环，继续下一次循环
    if name[:19] == "prompt_encoder.mask":
        continue

    # 打印当前处理的变量名、形状和数据类型
    print("Processing variable: " + name + " with shape: ", shape, " and type: ", v.dtype)

    # 将张量转换为 numpy 数组
    data = v.numpy()
    n_dims = len(data.shape)

    # 获取数据的形状
    dshape = data.shape

    # 默认类型为 fp16
    ftype_cur = 1
    # 如果 ftype 为 0，或者数据维度为 1，或者变量名为特定值，则将数据类型转换为 float32
    if ftype == 0 or n_dims == 1 or \
            name == "image_encoder.pos_embed" or \
            name.startswith("prompt_encoder") or \
            name.startswith("mask_decoder.iou_token") or \
            name.startswith("mask_decoder.mask_tokens"):
        print("  Converting to float32")
        data = data.astype(np.float32)
        ftype_cur = 0
    # 否则将数据类型转换为 float16
    else:
        print("  Converting to float16")
        data = data.astype(np.float16)

    # 如果变量名为特定值，则将数据重新整形为 4 维张量
    if name == "image_encoder.patch_embed.proj.bias":
        data = data.reshape(1, data.shape[0], 1, 1)
        n_dims = len(data.shape)
        dshape = data.shape

    # 打印新的数据形状
    print("  New shape: ", dshape)

    # 将变量名转换为 utf-8 编码的字节流
    str = name.encode('utf-8')
    # 将数据写入输出文件
    fout.write(struct.pack("iii", n_dims, len(str), ftype_cur))
    for i in range(n_dims):
        fout.write(struct.pack("i", dshape[n_dims - 1 - i]))
    fout.write(str)

    # 将数据写入输出文件
    data.tofile(fout)

# 关闭输出文件
fout.close()

# 打印处理完成的消息和输出文件名
print("Done. Output file: " + fname_out)
print("")
```