# `PowerInfer\convert-lora-to-ggml.py`

```
#!/usr/bin/env python3
# 指定脚本解释器为 Python3

from __future__ import annotations
# 导入未来的注解特性

import json
import os
import re
import struct
import sys
from typing import Any, BinaryIO, Sequence
# 导入所需的模块和类型注解

import numpy as np
import torch
# 导入 numpy 和 torch 模块

NUMPY_TYPE_TO_FTYPE: dict[str, int] = {"float32": 0, "float16": 1}
# 定义一个字典，将 numpy 类型映射到 FTYPE

HF_SUBLAYER_TO_GGML = {
    "self_attn.q_proj": "attn_q",
    "self_attn.k_proj": "attn_k",
    "self_attn.v_proj": "attn_v",
    "self_attn.o_proj": "attn_output",
    "mlp.gate_proj": "ffn_gate",
    "mlp.down_proj": "ffn_down",
    "mlp.up_proj": "ffn_up",
    "input_layernorm": "attn_norm",
    "post_attention_layernorm": "ffn_norm",
}
# 定义一个字典，将 HF 子层名称映射到 GGML

def translate_tensor_name(t: str) -> str:
    # 定义一个函数，将输入的张量名称翻译成指定格式的名称
    match = re.match(r".*layers\.(\d+)\.(\w+\.\w+)\.lora_(A|B)\.weight", t)
    # 使用正则表达式匹配张量名称
    if match:
        nn = match.group(1)
        sub_layer = match.group(2)
        lora_type = match.group(3)

        sub_layer_renamed = HF_SUBLAYER_TO_GGML.get(sub_layer)
        # 获取重命名后的子层名称
        if sub_layer_renamed is None:
            print(f"Error: unrecognized sub-layer {sub_layer} in tensor {t}")
            sys.exit(1)
            # 如果子层名称未被识别，则输出错误信息并退出程序

        output_string = (
            f"blk.{nn}.{HF_SUBLAYER_TO_GGML[sub_layer]}.weight.lora{lora_type}"
        )
        # 根据规则生成输出字符串
        return output_string
    else:
        print(f"Error: unrecognized tensor {t}")
        sys.exit(1)
        # 如果张量名称未被识别，则输出错误信息并退出程序

def write_file_header(fout: BinaryIO, params: dict[str, Any]) -> None:
    # 定义一个函数，用于写入文件头部信息
    fout.write(b"ggla"[::-1])  # magic (ggml lora)
    # 写入文件魔数
    fout.write(struct.pack("i", 1))  # file version
    # 写入文件版本号
    fout.write(struct.pack("i", params["r"]))
    # 写入参数中的 r 值
    assert (
        int(params["lora_alpha"]) == params["lora_alpha"]
    ), "cannot convert float to int losslessly"
    # 断言参数中的 lora_alpha 是否可以转换为整数
    fout.write(struct.pack("i", int(params["lora_alpha"])))
    # 将 lora_alpha 写入文件

def write_tensor_header(
    self, name: str, shape: Sequence[int], data_type: np.dtype[Any]
) -> None:
    # 定义一个函数，用于写入张量头部信息
    # 将字符串转换为 UTF-8 编码的字节流
    sname = name.encode("utf-8")
    # 将数据按照指定格式打包，并写入文件
    fout.write(
        struct.pack(
            "iii",
            len(shape),  # 写入形状的长度
            len(sname),  # 写入名称的长度
            NUMPY_TYPE_TO_FTYPE[data_type.name],  # 写入数据类型对应的文件类型
        )
    )
    # 将形状数据按照指定格式打包，并写入文件
    fout.write(struct.pack("i" * len(shape), *shape[::-1]))
    # 将名称数据写入文件
    fout.write(sname)
    # 移动文件指针到下一个 32 字节的倍数位置
    fout.seek((fout.tell() + 31) & -32)
# 检查命令行参数是否为2个，如果不是则打印使用说明并退出程序
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

# 加载模型文件
model = torch.load(input_model, map_location="cpu")

# 打开并读取输入的 JSON 文件
with open(input_json, "r") as f:
    params = json.load(f)

# 检查 PEFT 类型是否为 LORA，如果不是则打印错误信息并退出程序
if params["peft_type"] != "LORA":
    print(f"Error: unsupported adapter type {params['peft_type']}, expected LORA")
    sys.exit(1)

# 检查参数 fan_in_fan_out 是否为 True，如果是则打印错误信息并退出程序
if params["fan_in_fan_out"] is True:
    print("Error: param fan_in_fan_out is not supported")
    sys.exit(1)

# 检查参数 bias 是否为 None 或 "none"，如果不是则打印错误信息并退出程序
if params["bias"] is not None and params["bias"] != "none":
    print("Error: param bias is not supported")
    sys.exit(1)

# 检查参数 modules_to_save 是否为非空列表，如果是则打印错误信息并退出程序
if params["modules_to_save"] is not None and len(params["modules_to_save"]) > 0:
    print("Error: param modules_to_save is not supported")
    sys.exit(1)

# 打开输出文件并清空内容
with open(output_path, "wb") as fout:
    fout.truncate()

    # 写入文件头信息
    write_file_header(fout, params)
    # 遍历模型的键值对
    for k, v in model.items():
        # 如果键以 ".default.weight" 结尾，则替换为 ".weight"
        if k.endswith(".default.weight"):
            k = k.replace(".default.weight", ".weight")
        # 如果键为特定值，则跳过当前循环
        if k in ["llama_proj.weight", "llama_proj.bias"]:
            continue
        # 如果键以 "lora_A.weight" 结尾，则进行特定处理
        if k.endswith("lora_A.weight"):
            # 如果数据类型不是 float16 或 float32，则转换为 float 类型
            if v.dtype != torch.float16 and v.dtype != torch.float32:
                v = v.float()
            # 转置矩阵
            v = v.T
        else:
            # 转换为 float 类型
            v = v.float()

        # 转换为 numpy 数组
        t = v.detach().numpy()
        # 转换张量名称
        tname = translate_tensor_name(k)
        # 打印张量信息
        print(f"{k} => {tname} {t.shape} {t.dtype} {t.nbytes/1024/1024:.2f}MB")
        # 写入张量头信息
        write_tensor_header(fout, tname, t.shape, t.dtype)
        # 将张量数据写入文件
        t.tofile(fout)

# 打印转换完成的信息
print(f"Converted {input_json} and {input_model} to {output_path}")
```