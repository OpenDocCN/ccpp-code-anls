# `PowerInfer\examples\llava\llava-surgery.py`

```
# 导入必要的库
import argparse  # 用于解析命令行参数
import glob  # 用于查找文件路径模式匹配
import os  # 用于执行文件操作
import torch  # 用于深度学习框架

# 创建参数解析器
ap = argparse.ArgumentParser()
ap.add_argument("-m", "--model", help="Path to LLaVA v1.5 model")  # 添加命令行参数，指定LLaVA v1.5模型的路径
args = ap.parse_args()  # 解析命令行参数

# 查找包含多模态投影仪权重的模型部分
path = sorted(glob.glob(f"{args.model}/pytorch_model*.bin"))[-1]  # 根据模型路径查找符合条件的文件，并选择最新的一个
checkpoint = torch.load(path)  # 加载模型参数

# 获取多模态张量名称列表
mm_tensors = [k for k, v in checkpoint.items() if k.startswith("model.mm_projector")]

# 将这些张量存储在一个新的字典中，并保存
projector = {name: checkpoint[name].float() for name in mm_tensors}  # 将多模态张量存储在新的字典中，并转换为浮点数
torch.save(projector, f"{args.model}/llava.projector")  # 保存多模态投影仪权重

# 从检查点中删除这些张量并重新保存
for name in mm_tensors:
    del checkpoint[name]  # 从检查点中删除多模态张量

# BakLLaVA模型中包含CLIP张量
clip_tensors = [k for k, v in checkpoint.items() if k.startswith("model.vision_tower")]
if len(clip_tensors) > 0:
    clip = {name.replace("vision_tower.vision_tower.", ""): checkpoint[name].float() for name in clip_tensors}  # 将CLIP张量存储在新的字典中，并转换为浮点数
    torch.save(clip, f"{args.model}/llava.clip")  # 保存CLIP张量

    # 删除这些张量
    for name in clip_tensors:
        del checkpoint[name]  # 从检查点中删除CLIP张量

    # 需要删除添加的标记以便能够转换Mistral模型
    if os.path.exists(f"{args.model}/added_tokens.json"):
        with open(f"{args.model}/added_tokens.json", "w") as f:
            f.write("{}\n")  # 如果存在添加的标记文件，则清空文件内容

# 保存检查点
torch.save(checkpoint, path)  # 保存修改后的检查点

# 打印完成信息
print("Done!")  # 输出完成信息
print(f"Now you can convert {args.model} to a a regular LLaMA GGUF file.")  # 输出转换提示信息
print(f"Also, use {args.model}/llava.projector to prepare a llava-encoder.gguf file.")  # 输出使用提示信息
```