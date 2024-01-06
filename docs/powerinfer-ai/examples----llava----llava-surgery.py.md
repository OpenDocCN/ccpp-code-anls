# `PowerInfer\examples\llava\llava-surgery.py`

```
# 导入必要的库
import argparse  # 用于解析命令行参数
import glob  # 用于查找文件路径模式匹配
import os  # 用于操作文件和目录路径
import torch  # PyTorch深度学习库

# 创建参数解析器
ap = argparse.ArgumentParser()
ap.add_argument("-m", "--model", help="Path to LLaVA v1.5 model")  # 添加命令行参数，指定LLaVA v1.5模型的路径
args = ap.parse_args()  # 解析命令行参数

# 查找包含多模态投影仪权重的模型部分
path = sorted(glob.glob(f"{args.model}/pytorch_model*.bin"))[-1]  # 使用glob模块查找符合模式的文件路径，并选择最新的一个
checkpoint = torch.load(path)  # 加载模型检查点

# 获取多模态张量名称的列表
mm_tensors = [k for k, v in checkpoint.items() if k.startswith("model.mm_projector")]

# 将这些张量存储在一个新的字典中，并将其保存
projector = {name: checkpoint[name].float() for name in mm_tensors}  # 将多模态投影仪权重存储在新的字典中，并转换为浮点数
torch.save(projector, f"{args.model}/llava.projector")  # 保存多模态投影仪权重到指定路径
# 从检查点中删除这些张量，并重新保存检查点
for name in mm_tensors:
    del checkpoint[name]

# BakLLaVA 模型中包含 CLIP 张量
clip_tensors = [k for k, v in checkpoint.items() if k.startswith("model.vision_tower")]
if len(clip_tensors) > 0:
    # 从检查点中提取 CLIP 张量，并保存为新的文件
    clip = {name.replace("vision_tower.vision_tower.", ""): checkpoint[name].float() for name in clip_tensors}
    torch.save(clip, f"{args.model}/llava.clip")

    # 从检查点中删除这些张量
    for name in clip_tensors:
        del checkpoint[name]

    # 为了能够转换 Mistral 模型，需要删除添加的标记
    if os.path.exists(f"{args.model}/added_tokens.json"):
        with open(f"{args.model}/added_tokens.json", "w") as f:
            f.write("{}\n")
# 将checkpoint保存到指定的路径
torch.save(checkpoint, path)

# 打印提示信息
print("Done!")
print(f"Now you can convert {args.model} to a a regular LLaMA GGUF file.")
print(f"Also, use {args.model}/llava.projector to prepare a llava-encoder.gguf file.")
```