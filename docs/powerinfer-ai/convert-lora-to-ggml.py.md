# `PowerInfer\convert-lora-to-ggml.py`

```
#!/usr/bin/env python3
# 指定使用 Python3 解释器

from __future__ import annotations
# 导入未来版本的特性，用于支持类型注解

import json
import os
import re
import struct
import sys
from typing import Any, BinaryIO, Sequence
# 导入需要使用的模块和类型注解

import numpy as np
import torch
# 导入 numpy 和 torch 模块

NUMPY_TYPE_TO_FTYPE: dict[str, int] = {"float32": 0, "float16": 1}
# 定义一个字典，用于将 numpy 类型映射为特定的整数值

HF_SUBLAYER_TO_GGML = {
    "self_attn.q_proj": "attn_q",
    "self_attn.k_proj": "attn_k",
    "self_attn.v_proj": "attn_v",
# 定义一个字典，用于将特定的字符串映射为另一个字符串
# 定义一个字典，用于将输入的字符串映射为相应的输出字符串
{
    "self_attn.o_proj": "attn_output",
    "mlp.gate_proj": "ffn_gate",
    "mlp.down_proj": "ffn_down",
    "mlp.up_proj": "ffn_up",
    "input_layernorm": "attn_norm",
    "post_attention_layernorm": "ffn_norm",
}

# 定义一个函数，用于将输入的字符串进行转换
def translate_tensor_name(t: str) -> str:
    # 使用正则表达式匹配输入字符串，提取相关信息
    match = re.match(r".*layers\.(\d+)\.(\w+\.\w+)\.lora_(A|B)\.weight", t)
    if match:
        # 提取匹配到的信息中的第一个分组，表示层数
        nn = match.group(1)
        # 提取匹配到的信息中的第二个分组，表示子层名称
        sub_layer = match.group(2)
        # 提取匹配到的信息中的第三个分组，表示lora类型
        lora_type = match.group(3)

        # 将子层名称转换为另一个名称
        sub_layer_renamed = HF_SUBLAYER_TO_GGML.get(sub_layer)
        # 如果转换后的子层名称不存在，则打印错误信息并退出程序
        if sub_layer_renamed is None:
            print(f"Error: unrecognized sub-layer {sub_layer} in tensor {t}")
            sys.exit(1)
# 创建一个字符串，包含特定格式的信息
output_string = (
    f"blk.{nn}.{HF_SUBLAYER_TO_GGML[sub_layer]}.weight.lora{lora_type}"
)
# 返回创建的字符串
return output_string
# 如果条件不满足，打印错误信息并退出程序
else:
    print(f"Error: unrecognized tensor {t}")
    sys.exit(1)

# 写入文件头部信息
def write_file_header(fout: BinaryIO, params: dict[str, Any]) -> None:
    # 写入文件魔数（ggml lora）
    fout.write(b"ggla"[::-1])
    # 写入文件版本号
    fout.write(struct.pack("i", 1))
    # 写入参数中的 r 值
    fout.write(struct.pack("i", params["r"])
    # 检查并转换参数中的 lora_alpha 值为整数，如果无法无损转换则抛出异常
    assert (
        int(params["lora_alpha"]) == params["lora_alpha"]
    ), "cannot convert float to int losslessly"
# 将参数中的lora_alpha转换为整数，使用struct.pack将其打包成二进制数据并写入文件
fout.write(struct.pack("i", int(params["lora_alpha"]))

# 写入张量的头部信息，包括名称、形状、数据类型
# 将名称转换为utf-8编码
sname = name.encode("utf-8")
# 使用struct.pack将形状的长度、名称的长度、数据类型的转换后的值打包成二进制数据并写入文件
fout.write(
    struct.pack(
        "iii",
        len(shape),
        len(sname),
        NUMPY_TYPE_TO_FTYPE[data_type.name],
    )
)
# 使用struct.pack将形状的每个维度的值打包成二进制数据并写入文件
fout.write(struct.pack("i" * len(shape), *shape[::-1]))
# 将名称写入文件
fout.write(sname)
# 将文件指针移动到32的倍数位置
fout.seek((fout.tell() + 31) & -32)
# 检查命令行参数是否为2个，如果不是则打印用法提示并退出程序
if len(sys.argv) != 2:
    print(f"Usage: python {sys.argv[0]} <path>")
    print(
        "Path must contain HuggingFace PEFT LoRA files 'adapter_config.json' and 'adapter_model.bin'"
    )
    sys.exit(1)

# 构建输入文件路径
input_json = os.path.join(sys.argv[1], "adapter_config.json")
input_model = os.path.join(sys.argv[1], "adapter_model.bin")
output_path = os.path.join(sys.argv[1], "ggml-adapter-model.bin")

# 加载模型文件到内存中
model = torch.load(input_model, map_location="cpu")

# 打开并读取输入的 JSON 文件
with open(input_json, "r") as f:
    params = json.load(f)

# 检查 JSON 文件中的 PEFT 类型是否为 LORA，如果不是则打印错误信息并退出程序
if params["peft_type"] != "LORA":
    print(f"Error: unsupported adapter type {params['peft_type']}, expected LORA")
    sys.exit(1)
# 如果参数中的fan_in_fan_out为True，则打印错误信息并退出程序
if params["fan_in_fan_out"] is True:
    print("Error: param fan_in_fan_out is not supported")
    sys.exit(1)

# 如果参数中的bias不为None且不为"none"，则打印错误信息并退出程序
if params["bias"] is not None and params["bias"] != "none":
    print("Error: param bias is not supported")
    sys.exit(1)

# TODO: 这些似乎是已经训练过但没有lora的层。似乎并不广泛使用，但最终应该得到支持
# 如果参数中的modules_to_save不为None且长度大于0，则打印错误信息并退出程序
if params["modules_to_save"] is not None and len(params["modules_to_save"]) > 0:
    print("Error: param modules_to_save is not supported")
    sys.exit(1)

# 以二进制写入模式打开输出路径的文件
with open(output_path, "wb") as fout:
    # 清空文件内容
    fout.truncate()

    # 写入文件头部信息
    write_file_header(fout, params)
    # 遍历模型中的每个键值对
    for k, v in model.items():
        # 如果键以".default.weight"结尾
# 如果 k 中包含 ".default.weight"，则将其替换为 ".weight"
k = k.replace(".default.weight", ".weight")
# 如果 k 是 "llama_proj.weight" 或 "llama_proj.bias"，则跳过本次循环
if k in ["llama_proj.weight", "llama_proj.bias"]:
    continue
# 如果 k 以 "lora_A.weight" 结尾
if k.endswith("lora_A.weight"):
    # 如果 v 的数据类型不是 torch.float16 或 torch.float32，则将其转换为 torch.float()
    if v.dtype != torch.float16 and v.dtype != torch.float32:
        v = v.float()
    # 将 v 进行转置
    v = v.T
# 否则将 v 转换为 torch.float()
else:
    v = v.float()

# 将 v 转换为 numpy 数组
t = v.detach().numpy()
# 将 k 转换为对应的名称
tname = translate_tensor_name(k)
# 打印转换后的信息，包括原始名称、转换后的名称、形状、数据类型和占用内存大小
print(f"{k} => {tname} {t.shape} {t.dtype} {t.nbytes/1024/1024:.2f}MB")
# 将数组的头信息写入输出文件
write_tensor_header(fout, tname, t.shape, t.dtype)
# 将数组数据写入输出文件
t.tofile(fout)

# 打印转换完成的信息
print(f"Converted {input_json} and {input_model} to {output_path}")
```