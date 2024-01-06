# `PowerInfer\examples\llava\convert-image-encoder-to-gguf.py`

```
# 导入必要的模块
import argparse  # 用于解析命令行参数
import os  # 用于操作系统相关的功能
import json  # 用于处理 JSON 数据

import torch  # PyTorch 深度学习库
import numpy as np  # 用于数值计算的库
from gguf import *  # 导入 gguf 模块中的所有内容
from transformers import CLIPModel, CLIPProcessor  # 导入 CLIP 模型和处理器

TEXT = "clip.text"  # 文本输入的标识符
VISION = "clip.vision"  # 视觉输入的标识符

# 根据原始键和架构返回格式化后的键
def k(raw_key: str, arch: str) -> str:
    return raw_key.format(arch=arch)

# 判断是否应该跳过张量
def should_skip_tensor(name: str, has_text: bool, has_vision: bool, has_llava: bool) -> bool:
    if name in (
        "logit_scale",
```
（此处代码未完，无法继续解释）
# 检查给定的名称是否属于文本模型的位置 ID 或者视觉模型的位置 ID，如果是则返回 True
if name in (
    "text_model.embeddings.position_ids",
    "vision_model.embeddings.position_ids",
):
    return True

# 如果具有 LLAVA 并且名称属于指定的权重或偏置项，则返回 True
if has_llava and name in ["visual_projection.weight", "vision_model.post_layernorm.weight", "vision_model.post_layernorm.bias"]:
    return True

# 如果名称以 "v" 开头并且没有视觉模型，则返回 True
if name.startswith("v") and not has_vision:
    return True

# 如果名称以 "t" 开头并且没有文本模型，则返回 True
if name.startswith("t") and not has_text:
    return True

# 如果以上条件都不满足，则返回 False
return False


# 根据给定的名称返回相同的名称
def get_tensor_name(name: str) -> str:
    # 如果名称中包含 "projection"，则直接返回该名称
    if "projection" in name:
        return name
# 如果文件名中包含"mm_projector"，则将其替换为"mm"
if "mm_projector" in name:
    return name.replace("model.mm_projector", "mm")

# 将字节流转换为Unicode字符串
def bytes_to_unicode():
    """
    返回utf-8字节的列表和相应的Unicode字符串列表。
    可逆的bpe代码适用于Unicode字符串。
    这意味着如果要避免UNKs，您需要在词汇表中拥有大量的Unicode字符。
    当您处理大约10B个标记的数据集时，您最终需要大约5K个Unicode字符才能获得良好的覆盖率。
    这在正常情况下，比如32K bpe词汇表中占据了相当大的比例。
    为了避免这种情况，我们需要在utf-8字节和Unicode字符串之间建立查找表。
    并避免将bpe代码无法处理的空格/控制字符映射到其中。
    """
    # 创建utf-8字节的列表
    bs = (
        list(range(ord("!"), ord("~") + 1))
        + list(range(ord("¡"), ord("¬") + 1))
        + list(range(ord("®"), ord("ÿ") + 1))
    )
    # 复制 bs 列表
    cs = bs[:]
    # 初始化 n 为 0
    n = 0
    # 遍历 0 到 255 的整数
    for b in range(2**8):
        # 如果 b 不在 bs 列表中
        if b not in bs:
            # 将 b 添加到 bs 列表中
            bs.append(b)
            # 将 2**8 + n 添加到 cs 列表中
            cs.append(2**8 + n)
            # n 自增 1
            n += 1
    # 将 cs 列表中的整数转换为对应的字符
    cs = [chr(n) for n in cs]
    # 返回 bs 和 cs 组成的字典
    return dict(zip(bs, cs))


# 创建解析器对象
ap = argparse.ArgumentParser(prog="convert_hf_to_gguf.py")
# 添加模型目录参数
ap.add_argument("-m", "--model-dir", help="Path to model directory cloned from HF Hub", required=True)
# 添加使用 f32 替代 f16 的参数
ap.add_argument("--use-f32", action="store_true", default=False, help="Use f32 instead of f16")
# 添加仅保存文本模型的参数
ap.add_argument("--text-only", action="store_true", required=False,
                help="Save a text-only model. It can't be used to encode images")
# 添加仅保存视觉模型的参数
ap.add_argument("--vision-only", action="store_true", required=False,
                help="Save a vision-only model. It can't be used to encode texts")
# 添加 llava.projector 文件路径参数
ap.add_argument("--llava-projector", help="Path to llava.projector file. If specified, save an image encoder for LLaVA models.")
# 添加命令行参数，用于覆盖图像均值的值，nargs=3表示需要输入3个值，类型为浮点数，不是必需的参数，提供帮助信息
ap.add_argument("--image-mean", nargs=3, type=float, required=False, help="Override image mean values")
# 添加命令行参数，用于覆盖图像标准差的值，nargs=3表示需要输入3个值，类型为浮点数，不是必需的参数，提供帮助信息
ap.add_argument("--image-std", nargs=3, type=float, required=False, help="Override image std values")
# 添加命令行参数，用于指定输出目录，如果未指定，则默认为原始模型目录
ap.add_argument("-o", "--output-dir", help="Directory to save GGUF files. Default is the original model directory", default=None)

# 解析命令行参数
args = ap.parse_args()

# 如果同时指定了--text-only和--image-only参数，则打印错误信息并退出程序
if args.text_only and args.vision_only:
    print("--text-only and --image-only arguments cannot be specified at the same time.")
    exit(1)

# 如果使用了--use_f32参数，则打印警告信息
if args.use_f32:
    print("WARNING: Weights for the convolution op is always saved in f16, as the convolution op in GGML does not support 32-bit kernel weights yet.")

# 如果output_dir参数为None，则将输出目录设置为与模型相同的目录
dir_model = args.model_dir

# 以utf-8编码方式打开模型目录下的vocab.json文件，并加载其中的内容到vocab变量中
with open(dir_model + "/vocab.json", "r", encoding="utf-8") as f:
    vocab = json.load(f)
# 从词汇表中获取所有的标记
tokens = [key for key in vocab]

# 打开配置文件，读取其中的内容
with open(dir_model + "/config.json", "r", encoding="utf-8") as f:
    # 加载 JSON 文件中的配置信息
    config = json.load(f)
    # 从配置信息中获取视觉模型的超参数
    v_hparams = config["vision_config"]
    # 从配置信息中获取文本模型的超参数
    t_hparams = config["text_config"]

# 可能的数据类型
#   ftype == 0 -> float32
#   ftype == 1 -> float16
#
# 从数据类型到字符串的映射
ftype_str = ["f32", "f16"]

# 默认数据类型为 float16
ftype = 1
# 如果命令行参数指定使用 float32，则将数据类型设置为 float32
if args.use_f32:
    ftype = 0

# 从预训练模型目录加载 CLIP 模型
model = CLIPModel.from_pretrained(dir_model)
# 从预训练模型目录加载 CLIP 处理器
processor = CLIPProcessor.from_pretrained(dir_model)

# 初始化中间文件名为 None，以及各种编码器和投影器的标志
fname_middle = None
has_text_encoder = True
has_vision_encoder = True
has_llava_projector = False

# 根据命令行参数设置中间文件名和各种编码器和投影器的标志
if args.text_only:
    fname_middle = "text-"
    has_vision_encoder = False
elif args.vision_only:
    fname_middle = "vision-"
    has_text_encoder = False
elif args.llava_projector is not None:
    fname_middle = "mmproj-"
    has_text_encoder = False
    has_llava_projector = True
else:
    fname_middle = ""

# 设置输出目录为命令行参数中指定的目录，如果没有指定则使用模型目录
output_dir = args.output_dir if args.output_dir is not None else dir_model
# 如果输出目录不存在，则创建输出目录
os.makedirs(output_dir, exist_ok=True)
# 从输出目录中获取基本名称，并去掉前缀"ggml_"
output_prefix = os.path.basename(output_dir).replace("ggml_", "")
# 构建输出文件名
fname_out = os.path.join(output_dir, f"{fname_middle}model-{ftype_str[ftype]}.gguf")
# 创建 GGUFWriter 对象，指定输出文件路径和架构
fout = GGUFWriter(path=fname_out, arch="clip")

# 添加布尔类型的属性到输出文件
fout.add_bool("clip.has_text_encoder", has_text_encoder)
fout.add_bool("clip.has_vision_encoder", has_vision_encoder)
fout.add_bool("clip.has_llava_projector", has_llava_projector)
# 添加文件类型到输出文件
fout.add_file_type(ftype)
# 添加模型名称到输出文件
model_name = config["_name_or_path"] if "_name_or_path" in config else os.path.basename(dir_model)
fout.add_name(model_name)

# 根据参数添加描述信息到输出文件
if args.text_only:
    fout.add_description("text-only CLIP model")
elif args.vision_only and not has_llava_projector:
    fout.add_description("vision-only CLIP model")
elif has_llava_projector:
    fout.add_description("image encoder for LLaVA")
else:
    fout.add_description("two-tower CLIP model")
# 如果存在文本编码器，则添加文本模型的超参数
if has_text_encoder:
    # 添加文本模型的上下文长度
    fout.add_uint32(k(KEY_CONTEXT_LENGTH, TEXT), t_hparams["max_position_embeddings"])
    # 添加文本模型的嵌入长度
    fout.add_uint32(k(KEY_EMBEDDING_LENGTH, TEXT), t_hparams["hidden_size"])
    # 添加文本模型的前馈长度
    fout.add_uint32(k(KEY_FEED_FORWARD_LENGTH, TEXT), t_hparams["intermediate_size"])
    # 添加文本模型的投影维度
    fout.add_uint32("clip.text.projection_dim", t_hparams.get("projection_dim", config["projection_dim"]))
    # 添加文本模型的注意力头数
    fout.add_uint32(k(KEY_ATTENTION_HEAD_COUNT, TEXT), t_hparams["num_attention_heads"])
    # 添加文本模型的注意力层标准化参数
    fout.add_float32(k(KEY_ATTENTION_LAYERNORM_EPS, TEXT), t_hparams["layer_norm_eps"])
    # 添加文本模型的块数量
    fout.add_uint32(k(KEY_BLOCK_COUNT, TEXT), t_hparams["num_hidden_layers"])
    # 添加文本模型的标记列表
    fout.add_token_list(tokens)

# 如果存在视觉编码器，则添加视觉模型的超参数
if has_vision_encoder:
    # 添加视觉模型的图像大小
    fout.add_uint32("clip.vision.image_size", v_hparams["image_size"])
    # 添加视觉模型的补丁大小
    fout.add_uint32("clip.vision.patch_size", v_hparams["patch_size"])
    # 添加视觉模型的嵌入长度
    fout.add_uint32(k(KEY_EMBEDDING_LENGTH, VISION), v_hparams["hidden_size"])
    # 添加视觉模型的前馈长度
    fout.add_uint32(k(KEY_FEED_FORWARD_LENGTH, VISION), v_hparams["intermediate_size"])
    # 添加视觉模型的投影维度
    fout.add_uint32("clip.vision.projection_dim", v_hparams.get("projection_dim", config["projection_dim"]))
    # 添加视觉模型的注意力头数
    fout.add_uint32(k(KEY_ATTENTION_HEAD_COUNT, VISION), v_hparams["num_attention_heads"])
    # 添加视觉模型的注意力层标准化参数
    fout.add_float32(k(KEY_ATTENTION_LAYERNORM_EPS, VISION), v_hparams["layer_norm_eps"])
# 根据是否有LLAVA投影仪来确定块数，如果有则减去1，否则使用全部隐藏层
block_count = v_hparams["num_hidden_layers"] - 1 if has_llava_projector else v_hparams["num_hidden_layers"]
# 将块数添加到输出文件中
fout.add_uint32(k(KEY_BLOCK_COUNT, VISION), block_count)

# 根据参数或处理器中的图像均值和标准差来确定图像均值和标准差
image_mean = processor.image_processor.image_mean if args.image_mean is None else args.image_mean
image_std = processor.image_processor.image_std if args.image_std is None else args.image_std
# 将图像均值和标准差添加到输出文件中
fout.add_array("clip.vision.image_mean", image_mean)
fout.add_array("clip.vision.image_std", image_std)

# 根据隐藏层激活函数是否为gelu来确定是否使用gelu激活函数
use_gelu = v_hparams["hidden_act"] == "gelu"
# 将是否使用gelu激活函数的信息添加到输出文件中
fout.add_bool("clip.use_gelu", use_gelu)

# 如果有LLAVA投影仪，则移除视觉模型编码器的最后一层，并加载LLAVA投影仪
if has_llava_projector:
    model.vision_model.encoder.layers.pop(-1)
    projector = torch.load(args.llava_projector)
    # 遍历LLAVA投影仪中的数据，并根据维度和类型进行处理
    for name, data in projector.items():
        name = get_tensor_name(name)
        if data.ndim == 2:
            data = data.squeeze().numpy().astype(np.float16)
        else:
            # 进行其他处理（缺失部分代码）
    # 将数据转换为numpy数组，并转换为32位浮点数
    data = data.squeeze().numpy().astype(np.float32)

    # 将数据添加到输出文件中
    fout.add_tensor(name, data)

    # 打印信息，指示投影器张量已添加
    print("Projector tensors added\n")

# 获取模型的状态字典
state_dict = model.state_dict()

# 遍历状态字典中的每个参数名和数据
for name, data in state_dict.items():
    # 检查是否应该跳过该张量
    if should_skip_tensor(name, has_text_encoder, has_vision_encoder, has_llava_projector):
        # 如果需要跳过，打印跳过的参数名并继续下一个参数
        print(f"skipping parameter: {name}")
        continue

    # 获取张量的名称
    name = get_tensor_name(name)
    # 将数据转换为numpy数组
    data = data.squeeze().numpy()

    # 获取数据的维度
    n_dims = len(data.shape)

    # 设置默认的浮点类型为32位
    ftype_cur = 0
# 如果数据的维度为4，则将数据类型转换为np.float16，并设置ftype_cur为1
if n_dims == 4:
    print(f"tensor {name} is always saved in f16")
    data = data.astype(np.float16)
    ftype_cur = 1
# 如果ftype为1
elif ftype == 1:
    # 如果文件名以".weight"结尾且维度为2，则将数据类型转换为np.float16，并设置ftype_cur为1
    if name[-7:] == ".weight" and n_dims == 2:
        print("  Converting to float16")
        data = data.astype(np.float16)
        ftype_cur = 1
    # 否则将数据类型转换为np.float32，并设置ftype_cur为0
    else:
        print("  Converting to float32")
        data = data.astype(np.float32)
        ftype_cur = 0
# 如果以上条件都不满足
else:
    # 如果数据类型不是np.float32，则将数据类型转换为np.float32，并设置ftype_cur为0
    if data.dtype != np.float32:
        print("  Converting to float32")
        data = data.astype(np.float32)
        ftype_cur = 0

# 打印名称、数据类型和形状信息
print(f"{name} - {ftype_str[ftype_cur]} - shape = {data.shape}")
# 将张量数据添加到输出文件中，使用给定的名称和数据
fout.add_tensor(name, data)

# 将头部信息写入文件
fout.write_header_to_file()

# 将键值数据写入文件
fout.write_kv_data_to_file()

# 将张量数据写入文件
fout.write_tensors_to_file()

# 关闭输出文件
fout.close()

# 打印完成信息，输出文件名
print("Done. Output file: " + fname_out)
```