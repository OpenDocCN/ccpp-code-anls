# `PowerInfer\examples\llava\convert-image-encoder-to-gguf.py`

```cpp
# 导入必要的库
import argparse  # 用于解析命令行参数
import os  # 用于操作系统相关的功能
import json  # 用于处理 JSON 数据

import torch  # PyTorch 深度学习库
import numpy as np  # 用于处理数组和矩阵的库
from gguf import *  # 导入自定义的 gguf 模块
from transformers import CLIPModel, CLIPProcessor  # 从 transformers 库中导入 CLIPModel 和 CLIPProcessor 类

TEXT = "clip.text"  # 定义文本标识符
VISION = "clip.vision"  # 定义视觉标识符


def k(raw_key: str, arch: str) -> str:
    # 根据原始键和架构返回格式化后的键
    return raw_key.format(arch=arch)


def should_skip_tensor(name: str, has_text: bool, has_vision: bool, has_llava: bool) -> bool:
    # 判断是否应该跳过特定的张量
    if name in (
        "logit_scale",
        "text_model.embeddings.position_ids",
        "vision_model.embeddings.position_ids",
    ):
        return True  # 如果张量名在指定的列表中，则跳过

    if has_llava and name in ["visual_projection.weight", "vision_model.post_layernorm.weight", "vision_model.post_layernorm.bias"]:
        return True  # 如果具有特定架构并且张量名在指定的列表中，则跳过

    if name.startswith("v") and not has_vision:
        return True  # 如果张量名以 "v" 开头且没有视觉模型，则跳过

    if name.startswith("t") and not has_text:
        return True  # 如果张量名以 "t" 开头且没有文本模型，则跳过

    return False  # 其他情况下不跳过


def get_tensor_name(name: str) -> str:
    # 获取张量的名称
    if "projection" in name:
        return name  # 如果名称中包含 "projection"，则返回原始名称

    if "mm_projector" in name:
        return name.replace("model.mm_projector", "mm")  # 如果名称中包含 "mm_projector"，则替换为 "mm"

    return name.replace("text_model", "t").replace("vision_model", "v").replace("encoder.layers", "blk").replace("embeddings.", "").replace("_proj", "").replace("self_attn.", "attn_").replace("layer_norm", "ln").replace("layernorm", "ln").replace("mlp.fc1", "ffn_down").replace("mlp.fc2", "ffn_up").replace("embedding", "embd").replace("final", "post").replace("layrnorm", "ln")


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
    # 将 utf-8 字节和相应的 Unicode 字符串列表返回
    # 可逆的 bpe 编码适用于 Unicode 字符串
    # 这意味着如果要避免 UNKs，则需要在词汇表中拥有大量的 Unicode 字符
    # 当你处理大约 100 亿个标记的数据集时，你最终需要大约 5K 个字符来获得良好的覆盖率
    # 这在你的常规 32K bpe 词汇表中占据了相当大的比例
    # 为了避免这种情况，我们需要在 utf-8 字节和 Unicode 字符串之间建立查找表
    # 并避免映射到 bpe 代码无法处理的空格/控制字符
    # 创建一个包含标准 ASCII 字符的列表
    bs = (
        list(range(ord("!"), ord("~") + 1))
        # 将扩展的拉丁字符添加到列表中
        + list(range(ord("¡"), ord("¬") + 1))
        # 将扩展的特殊字符添加到列表中
        + list(range(ord("®"), ord("ÿ") + 1))
    )
    # 复制 bs 列表到 cs 列表
    cs = bs[:]
    # 初始化 n 为 0
    n = 0
    # 遍历 0 到 255 的整数
    for b in range(2**8):
        # 如果整数不在 bs 列表中
        if b not in bs:
            # 将整数添加到 bs 列表
            bs.append(b)
            # 将 256 + n 添加到 cs 列表
            cs.append(2**8 + n)
            # n 自增 1
            n += 1
    # 将 cs 列表中的整数转换为对应的字符
    cs = [chr(n) for n in cs]
    # 返回 bs 和 cs 列表对应的字典
    return dict(zip(bs, cs))
# 创建一个参数解析器对象，设置程序名称为"convert_hf_to_gguf.py"
ap = argparse.ArgumentParser(prog="convert_hf_to_gguf.py")
# 添加一个参数，用于指定从 HF Hub 克隆的模型目录的路径，必须提供
ap.add_argument("-m", "--model-dir", help="Path to model directory cloned from HF Hub", required=True)
# 添加一个参数，用于指定是否使用 f32 而不是 f16
ap.add_argument("--use-f32", action="store_true", default=False, help="Use f32 instead of f16")
# 添加一个参数，用于指定是否只保存文本模型，不能用于编码图像
ap.add_argument("--text-only", action="store_true", required=False, help="Save a text-only model. It can't be used to encode images")
# 添加一个参数，用于指定是否只保存图像模型，不能用于编码文本
ap.add_argument("--vision-only", action="store_true", required=False, help="Save a vision-only model. It can't be used to encode texts")
# 添加一个参数，用于指定 llava.projector 文件的路径，如果指定了，则保存用于 LLaVA 模型的图像编码器
ap.add_argument("--llava-projector", help="Path to llava.projector file. If specified, save an image encoder for LLaVA models.")
# 添加一个参数，用于覆盖图像均值值
ap.add_argument("--image-mean", nargs=3, type=float, required=False, help="Override image mean values")
# 添加一个参数，用于覆盖图像标准差值
ap.add_argument("--image-std", nargs=3, type=float, required=False, help="Override image std values")
# 添加一个参数，用于指定保存 GGUF 文件的目录，默认为原始模型目录
ap.add_argument("-o", "--output-dir", help="Directory to save GGUF files. Default is the original model directory", default=None)

# 解析命令行参数
args = ap.parse_args()

# 如果同时指定了 --text-only 和 --image-only 参数，则打印错误信息并退出程序
if args.text_only and args.vision_only:
    print("--text-only and --image-only arguments cannot be specified at the same time.")
    exit(1)

# 如果指定了 --use_f32 参数，则打印警告信息
if args.use_f32:
    print("WARNING: Weights for the convolution op is always saved in f16, as the convolution op in GGML does not support 32-bit kernel weights yet.")

# 如果 output_dir 参数为 None，则将模型目录设置为 dir_model
dir_model = args.model_dir

# 从模型目录中读取并加载 vocab.json 文件，获取词汇表
with open(dir_model + "/vocab.json", "r", encoding="utf-8") as f:
    vocab = json.load(f)
    tokens = [key for key in vocab]

# 从模型目录中读取并加载 config.json 文件，获取配置信息
with open(dir_model + "/config.json", "r", encoding="utf-8") as f:
    config = json.load(f)
    v_hparams = config["vision_config"]
    t_hparams = config["text_config"]

# 可能的数据类型
#   ftype == 0 -> float32
#   ftype == 1 -> float16
#
# 将 ftype 映射为字符串
ftype_str = ["f32", "f16"]

# 默认数据类型为 float16
ftype = 1
# 如果指定了 --use_f32 参数，则数据类型为 float32
if args.use_f32:
    ftype = 0

# 从预训练模型目录中加载 CLIP 模型
model = CLIPModel.from_pretrained(dir_model)
# 从预训练模型目录加载 CLIP 处理器
processor = CLIPProcessor.from_pretrained(dir_model)

# 初始化中间文件名为 None，以及各种标志位
fname_middle = None
has_text_encoder = True
has_vision_encoder = True
has_llava_projector = False

# 根据命令行参数设置中间文件名和各种标志位
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

# 设置输出目录为命令行参数中的输出目录，如果没有则使用模型目录
output_dir = args.output_dir if args.output_dir is not None else dir_model
# 创建输出目录
os.makedirs(output_dir, exist_ok=True)
# 从输出目录中提取模型名称作为前缀
output_prefix = os.path.basename(output_dir).replace("ggml_", "")
# 构建输出文件名
fname_out = os.path.join(output_dir, f"{fname_middle}model-{ftype_str[ftype]}.gguf")
# 创建 GGUFWriter 对象
fout = GGUFWriter(path=fname_out, arch="clip")

# 添加各种标志位和模型信息到 GGUFWriter 对象中
fout.add_bool("clip.has_text_encoder", has_text_encoder)
fout.add_bool("clip.has_vision_encoder", has_vision_encoder)
fout.add_bool("clip.has_llava_projector", has_llava_projector)
fout.add_file_type(ftype)
model_name = config["_name_or_path"] if "_name_or_path" in config else os.path.basename(dir_model)
fout.add_name(model_name)

# 根据命令行参数和标志位添加模型描述信息
if args.text_only:
    fout.add_description("text-only CLIP model")
elif args.vision_only and not has_llava_projector:
    fout.add_description("vision-only CLIP model")
elif has_llava_projector:
    fout.add_description("image encoder for LLaVA")
else:
    fout.add_description("two-tower CLIP model")

# 如果有文本编码器，添加文本模型超参数到 GGUFWriter 对象中
if has_text_encoder:
    # text_model hparams
    fout.add_uint32(k(KEY_CONTEXT_LENGTH, TEXT), t_hparams["max_position_embeddings"])
    fout.add_uint32(k(KEY_EMBEDDING_LENGTH, TEXT), t_hparams["hidden_size"])
    fout.add_uint32(k(KEY_FEED_FORWARD_LENGTH, TEXT), t_hparams["intermediate_size"])
    fout.add_uint32("clip.text.projection_dim", t_hparams.get("projection_dim", config["projection_dim"]))
    fout.add_uint32(k(KEY_ATTENTION_HEAD_COUNT, TEXT), t_hparams["num_attention_heads"])
    fout.add_float32(k(KEY_ATTENTION_LAYERNORM_EPS, TEXT), t_hparams["layer_norm_eps"])
    # 向fout对象中添加一个32位无符号整数，键为(KEY_BLOCK_COUNT, TEXT)，值为t_hparams["num_hidden_layers"]
    fout.add_uint32(k(KEY_BLOCK_COUNT, TEXT), t_hparams["num_hidden_layers"])
    # 向fout对象中添加一个token列表
    fout.add_token_list(tokens)
# 如果存在视觉编码器
if has_vision_encoder:
    # 添加视觉模型超参数
    fout.add_uint32("clip.vision.image_size", v_hparams["image_size"])
    fout.add_uint32("clip.vision.patch_size", v_hparams["patch_size"])
    fout.add_uint32(k(KEY_EMBEDDING_LENGTH, VISION), v_hparams["hidden_size"])
    fout.add_uint32(k(KEY_FEED_FORWARD_LENGTH, VISION), v_hparams["intermediate_size"])
    fout.add_uint32("clip.vision.projection_dim", v_hparams.get("projection_dim", config["projection_dim"]))
    fout.add_uint32(k(KEY_ATTENTION_HEAD_COUNT, VISION), v_hparams["num_attention_heads"])
    fout.add_float32(k(KEY_ATTENTION_LAYERNORM_EPS, VISION), v_hparams["layer_norm_eps"])
    # 根据是否有 llava 投影器确定块数
    block_count = v_hparams["num_hidden_layers"] - 1 if has_llava_projector else v_hparams["num_hidden_layers"]
    fout.add_uint32(k(KEY_BLOCK_COUNT, VISION), block_count)

    # 获取图像均值和标准差
    image_mean = processor.image_processor.image_mean if args.image_mean is None else args.image_mean
    image_std = processor.image_processor.image_std if args.image_std is None else args.image_std
    fout.add_array("clip.vision.image_mean", image_mean)
    fout.add_array("clip.vision.image_std", image_std)

# 根据是否使用 gelu 激活函数添加布尔值
use_gelu = v_hparams["hidden_act"] == "gelu"
fout.add_bool("clip.use_gelu", use_gelu)

# 如果存在 llava 投影器
if has_llava_projector:
    # 移除视觉模型编码器的最后一层
    model.vision_model.encoder.layers.pop(-1)
    # 加载 llava 投影器
    projector = torch.load(args.llava_projector)
    # 遍历投影器的名称和数据
    for name, data in projector.items():
        name = get_tensor_name(name)
        # 如果数据维度为2，则转换为float16类型，否则转换为float32类型
        if data.ndim == 2:
            data = data.squeeze().numpy().astype(np.float16)
        else:
            data = data.squeeze().numpy().astype(np.float32)
        # 添加张量
        fout.add_tensor(name, data)
    # 打印信息
    print("Projector tensors added\n")

# 获取模型的状态字典
state_dict = model.state_dict()
# 遍历状态字典的名称和数据
for name, data in state_dict.items():
    # 如果应该跳过该张量，则打印跳过信息并继续下一次循环
    if should_skip_tensor(name, has_text_encoder, has_vision_encoder, has_llava_projector):
        print(f"skipping parameter: {name}")
        continue
    # 获取张量名称并将数据转换为numpy数组
    name = get_tensor_name(name)
    data = data.squeeze().numpy()
    # 获取数据的维度
    n_dims = len(data.shape)

    # 根据条件设置数据类型，ftype == 0 -> float32, ftype == 1 -> float16
    ftype_cur = 0
    if n_dims == 4:
        # 如果数据维度为4，则将数据类型设置为float16，并打印提示信息
        print(f"tensor {name} is always saved in f16")
        data = data.astype(np.float16)
        ftype_cur = 1
    elif ftype == 1:
        # 如果数据类型为float16，且数据名称以".weight"结尾且维度为2，则将数据类型设置为float16，并打印提示信息
        if name[-7:] == ".weight" and n_dims == 2:
            print("  Converting to float16")
            data = data.astype(np.float16)
            ftype_cur = 1
        else:
            # 否则将数据类型设置为float32，并打印提示信息
            print("  Converting to float32")
            data = data.astype(np.float32)
            ftype_cur = 0
    else:
        # 如果数据类型不是float32，则将数据类型设置为float32，并打印提示信息
        if data.dtype != np.float32:
            print("  Converting to float32")
            data = data.astype(np.float32)
            ftype_cur = 0

    # 打印数据名称、数据类型和形状信息
    print(f"{name} - {ftype_str[ftype_cur]} - shape = {data.shape}")
    # 将数据添加到输出对象中
    fout.add_tensor(name, data)
# 将文件头信息写入文件
fout.write_header_to_file()
# 将键值数据写入文件
fout.write_kv_data_to_file()
# 将张量数据写入文件
fout.write_tensors_to_file()
# 关闭文件
fout.close()

# 打印完成信息，输出文件名
print("Done. Output file: " + fname_out)
```